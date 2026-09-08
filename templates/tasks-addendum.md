## Schema Review Checklist

<!--
  Added by the db-standards preset. Run this before merging any task that
  creates or alters a migration. Every "no" needs a written justification
  in the PR.
-->

### Naming

- [ ] Tables, columns, FKs, indexes, and constraints follow the naming conventions in the plan.
- [ ] Booleans are predicates; enums are text/native (not int codes); money is not a float.
- [ ] `created_at` / `updated_at` present; soft delete uses `deleted_at`, not a boolean.

### Integrity

- [ ] Every foreign key has a real FK constraint with an explicit `ON DELETE` rule.
- [ ] `NOT NULL` on every column that is logically required.
- [ ] `UNIQUE` / `CHECK` constraints express the invariants from the spec.
- [ ] Default values are set in the database, not only in application code.

### Normalization

- [ ] Each new table is classified OLTP or OLAP in the plan.

OLTP tables:

- [ ] Schema is 3NF unless a denormalization is recorded in the plan's table.
- [ ] No multi-value column (comma list, ad-hoc JSON) standing in for a child table that gets queried.
- [ ] Every denormalized / counter / aggregate column names its sync mechanism and a way to recompute it from the source.
- [ ] A view was considered before adding a duplicated column for read convenience.
- [ ] No reporting-only column bolted onto an OLTP table.

OLAP tables:

- [ ] Table is rebuildable from the OLTP source; nothing treats it as authoritative.
- [ ] Load is idempotent and its source + cadence + full/incremental is documented.
- [ ] Grain is stated in a comment and every column fits it.
- [ ] The OLTP business/natural key is carried through.
- [ ] Overwrite-vs-keep-history is decided explicitly per entity.

### Performance

- [ ] Every FK used for lookup/join has an index.
- [ ] Each new hot-path query maps to a named index (see the plan's indexing table).
- [ ] Composite index column order is correct (equality then range/sort); no redundant indexes.
- [ ] List endpoints have a query-count budget with a test asserting it.
- [ ] No query in a loop; related data is eager/batch loaded.
- [ ] Large-list pagination is keyset, not `OFFSET`.

### SQL portability

- [ ] Query uses standard (ANSI) SQL where a standard form exists.
- [ ] Every vendor-specific function / operator / type / proc / trigger is listed in the plan's "SQL portability" table, with its standard alternative and the reason it was still used.
- [ ] No business logic hidden in a stored procedure or trigger unless the plan justifies it.
- [ ] Unavoidable vendor features are isolated to one module/migration, not scattered.

### Query safety (SQL injection)

- [ ] Every query taking external input is in the plan's "Query safety" table.
- [ ] Values reach SQL only as bound parameters — no string concatenation or interpolation (app code, stored procedures, ORM raw/execute).
- [ ] `LIKE` patterns bind the value and escape `%` / `_` / `\`.
- [ ] `IN (...)` lists bind one parameter per element (or an array param).
- [ ] Dynamic identifiers and `ORDER BY` direction come from a code allowlist, not the request value.
- [ ] `LIMIT` / `OFFSET` from input are parsed to a bounded integer.
- [ ] Any query building SQL from input without an allowlist is recorded in the plan's alert table and was reviewed.
- [ ] The app's DB role is not the table owner and lacks DDL privileges.
- [ ] A test covers at least one injection attempt on a user-facing query (quote, `;`, `--`, `' OR '1'='1`).

### Migration safety

- [ ] Migration is reversible, or the PR explains why not.
- [ ] Adding a `NOT NULL` column: has a default or is a two-step (add nullable → backfill → set not null).
- [ ] Index on a large table is built concurrently / without a long lock.
- [ ] Backfill of a large table runs outside the transaction / in batches.
- [ ] Rename or type change is done expand-contract, not in place.

### Data handling

- [ ] PII / regulated columns identified and handled per the spec's retention rule.
- [ ] No secrets or tokens stored in plaintext.
