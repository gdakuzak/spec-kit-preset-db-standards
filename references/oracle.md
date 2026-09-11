# Oracle — engine-specific guidance

Read this alongside the `db-standards` plan sections it sharpens. It doesn't
repeat anything already covered generically (normalization, constraints, query
safety) — only where Oracle's own idiom is the *right* answer, not a
portability risk.

## Bind variables (sharpens Query Safety)

Oracle's shared-pool cursor cache makes this a performance rule as much as a
security one: a literal value inline in SQL text forces a hard parse per
distinct statement, which competes for the library-cache latch under load.

- Every value is a bind variable (`:name` or `?`), no exceptions — this is the
  same rule as Query Safety, but on Oracle it also fixes a real bottleneck.
- Don't rely on `CURSOR_SHARING = FORCE` to paper over literal SQL — fix the
  call site. It's a last-resort DBA lever, not a design choice to depend on.
- Dynamic SQL (`EXECUTE IMMEDIATE`) still binds every value; it never
  concatenates input into the string.

## Data types

- **`NUMBER(p,s)`** always declares precision and scale. Bare `NUMBER` has no
  fixed byte size and silently accepts anything — the equivalent of skipping
  the `numeric(19,4)` rule in Naming conventions.
- **`VARCHAR2` length semantics**: state whether the column is `BYTE` or `CHAR`
  semantics explicitly (`VARCHAR2(100 CHAR)`), and confirm it against
  `NLS_LENGTH_SEMANTICS` recorded in the constitution's Database section. Mixed
  byte/char columns in a multi-byte (UTF8/AL32UTF8) database is a recurring
  truncation bug.
- **`DATE` has no sub-second precision**; use `TIMESTAMP` (and
  `TIMESTAMP WITH TIME ZONE` per the UTC rule in SQL portability) for anything
  finer than a day-second.
- Prefer `IDENTITY` columns (12c+) over a manually managed `SEQUENCE` +
  trigger for surrogate keys — fewer moving parts, same guarantee. Keep a plain
  `SEQUENCE` only when the id must be shared across tables or generated before
  insert.

## Indexing

- B-tree is the default (matches the generic Indexing plan section).
- **Bitmap indexes** are for low-cardinality columns on read-mostly / reporting
  tables (star-schema dimension keys, status flags on an OLAP table) — never on
  a table with concurrent DML; bitmap index locking escalates to the whole
  segment of rows sharing a bit, not row-level.
- Function-based indexes when a hot query filters on an expression
  (`UPPER(email)`, `TRUNC(created_at)`) — index the expression rather than
  duplicating the column transformed.

## PL/SQL: when a stored procedure earns its place

SQL portability already says business logic default lives in the app, not the
database, unless justified. On Oracle that justification is common enough to
spell out:

- A **package** (not loose procedures) is the unit — package spec is the
  contract, package body can change without invalidating dependents.
- Earn a PL/SQL package for: enforcing a multi-row/multi-table invariant
  (Constraints & integrity already asks for this to live in a transaction or
  trigger — a package is where that logic goes), a batch operation that must
  run inside the database to avoid round-trips, or logic genuinely shared by
  multiple heterogeneous callers (app, ETL, another team's job).
- Don't reach for it just because "that's how this shop does things" — every
  package is recorded in the SQL portability vendor-feature table like any
  other Oracle-specific feature.
- `WHEN OTHERS THEN NULL` (or `WHEN OTHERS THEN ...` without `RAISE`) is
  forbidden — it swallows errors Constraints & integrity relies on surfacing.
  Catch specific exceptions; if you must catch `OTHERS`, log and re-raise.

## Partitioning

- Partition a table when a query pattern the plan's Indexing section
  identifies always filters on one column with natural buckets — a date
  (range), a tenant/region (list), or neither (hash, for even distribution
  with no natural predicate).
- State the partition key and strategy in the plan's Schema changes table
  alongside the DDL; partitioning is a schema decision, not an ops afterthought.
- OLAP fact tables (see `db-standards`' Normalization → OLAP track) are the
  most common candidate — partition by the load date/grain.

## Materialized views (OLAP track)

- The concrete Oracle answer to the OLAP section's "idempotent, documented
  load": a materialized view with `REFRESH FAST ON COMMIT` or a scheduled
  `REFRESH COMPLETE`/`REFRESH FORCE`, backed by a materialized view log on the
  source table for fast refresh.
- State the refresh mode and cadence in the OLAP table's grain comment — it's
  the same information the generic section already asks for.
