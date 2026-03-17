# DataFusion Planner Note: `LEFT/RIGHT SEMI JOIN ... USING` wildcard bug

## Context

- Date: 2026-03-17
- Workspace: `postfusion`
- DataFusion version inspected: `52.1.0` (from local cargo registry)

The tables are identical, so swapping them should not change the output schema.
The schema of Q1 is as expected: `x1 int, x2 int, x3 int`.
However, the schema of Q2 is missing the `x1` field.

## Problem statement

For `SELECT *` over a `USING` join, DataFusion performs wildcard expansion and deduplicates join-key columns by name.

This behavior is correct for joins whose output schema contains both sides (for example, `INNER JOIN ... USING`), but incorrect for `SEMI/ANTI` joins where the output schema only contains one side.

Observed bug:

- In some `LEFT SEMI JOIN ... USING(col)` cases, wildcard expansion skips the left key column even though only left columns are in output.
- Which side gets skipped depends on lexical ordering of qualified columns (table names), not SQL semantics.

## Reproduction

### Case that loses column (current bug)

```sql
create table test(studnr int, lecturenr int, persnr int, grade decimal(2,1));
create table assistants(persnr int, name text, area text, boss int);
explain select * from test left semi join assistants using (persnr);
```

Current behavior (logical plan in this repo): projection contains `studnr`, `lecturenr`, `grade` but not `persnr`.

### Case that does not lose column

```sql
create table s(x1 int, x2 int, x3 int);
create table t(x1 int, x2 int, x3 int);
select * from s left semi join t using (x1);
```

Current behavior keeps `x1`.

Reason for difference: deterministic sort order picks which qualified key to retain before dedupe-by-name.

## Planning pipeline and where behavior comes from

`SessionState::statement_to_plan` is not where the bug is introduced; it delegates to SQL planner:

- `datafusion/src/execution/session_state.rs`
  - `statement_to_plan(...)` resolves tables and calls:
  - `SqlToRel::new_with_options(...).statement_to_plan(statement)`

Join planning path:

- `datafusion-sql/src/relation/join.rs`
  - `JoinConstraint::Using(...)` -> `LogicalPlanBuilder::join_using(...)`

Wildcard path:

- `datafusion-sql/src/select.rs`
  - `SelectItem::Wildcard` is preserved as `SelectExpr::Wildcard(...)`
- `datafusion-expr/src/logical_plan/builder.rs`
  - `project_with_validation(...)` calls `expand_wildcard(...)`
- `datafusion-expr/src/utils.rs`
  - `expand_wildcard(...)` calls `exclude_using_columns(plan)` and skips those columns

## Root cause

`exclude_using_columns(plan)` uses `plan.using_columns()` to collect both sides of each `USING` key pair, then:

1. sorts qualified columns,
2. keeps first by unqualified name,
3. marks later duplicates to skip.

However, this logic does **not** filter to columns that actually exist in the plan output schema.

For `LEFT SEMI` and `LEFT ANTI`, output schema is left-only. For `RIGHT SEMI` and `RIGHT ANTI`, output schema is right-only.

Schema construction confirms this:

- `datafusion-expr/src/logical_plan/builder.rs::build_join_schema(...)`
  - `JoinType::LeftSemi | JoinType::LeftAnti` -> only left fields
  - `JoinType::RightSemi | JoinType::RightAnti` -> only right fields

Because the dedupe list still includes the non-output side, lexical ordering can cause the output-side key to be marked skipped.

## Why this is spec-incorrect

`SELECT *` should expand from the resulting `FROM` schema. Columns absent from that resulting schema must not influence which output columns are skipped.

The current behavior allows non-output columns (from the suppressed side of semi/anti join) to affect `*` expansion.

## Proposed fix (minimal and safe)

Update `datafusion-expr/src/utils.rs::exclude_using_columns(plan)`:

1. Build `output_columns` set from `plan.schema().columns()`.
2. For each `USING` group, keep only columns contained in `output_columns`.
3. Perform existing dedupe-by-name on this filtered set.
4. Return resulting skip set.

Effect:

- `INNER/LEFT/RIGHT/FULL` joins: behavior unchanged (both sides in output).
- `SEMI/ANTI` joins: dedupe only considers surviving side, preventing accidental key loss.

## Suggested tests to add

Location:

- `datafusion-sql/tests/sql_integration.rs`
  - Near `test_using_join_wildcard_schema` (already covers wildcard dedupe for `USING/NATURAL` joins)

Add at least:

1. `SELECT * FROM s LEFT SEMI JOIN t USING (x1)` -> schema contains exactly one `x1` from left side and keeps other left cols.
2. `SELECT * FROM s RIGHT SEMI JOIN t USING (x1)` -> schema contains exactly one `x1` from right side and keeps other right cols.
3. (Optional) left/right anti variants with same assertions.

## Quick implementation sketch

Pseudo-diff for `exclude_using_columns`:

```rust
let output_columns: HashSet<Column> = plan.schema().columns().into_iter().collect();
let using_columns = plan.using_columns()?;
let excluded = using_columns
    .into_iter()
    .flat_map(|cols| {
        let mut cols = cols
            .into_iter()
            .filter(|c| output_columns.contains(c))
            .collect::<Vec<_>>();
        cols.sort();
        let mut out_names = HashSet::<String>::new();
        cols.into_iter().filter_map(move |c| {
            if out_names.insert(c.name.clone()) { None } else { Some(c) }
        })
    })
    .collect::<HashSet<_>>();
```

No other planner stages should need modification.

## Notes for tomorrow

- The repo currently has unrelated local modifications:
  - `src/core/testdata/algebra/logical/property/nullability.md`
- If implementing in `postfusion` by patching vendored dependency behavior, decide whether:
  1. this project will carry a local adaptation/workaround, or
  2. it should upstream a DataFusion fix and adjust expected testdata after dependency update.
