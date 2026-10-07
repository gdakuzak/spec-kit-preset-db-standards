# PostgreSQL — engine-specific guidance

Read this alongside the `db-standards` plan sections it sharpens. It doesn't
repeat anything already covered generically — only where PostgreSQL's own
behavior is the right answer, or where it quietly breaks a generic assumption.

## Version

- Run a **supported major** (the project supports each for 5 years; check
  postgresql.org/support/versioning) and apply **minor releases** promptly —
  security fixes ship only in minors. Record the exact version in the
  constitution's Database section; the feature gates below depend on it.
- Gate features on the version before the plan relies on them:

  | Feature | Min. version |
  |---------|--------------|
  | `GENERATED ... AS IDENTITY`, declarative partitioning, logical replication | 10 |
  | `INCLUDE` indexes, constant-`DEFAULT` `ADD COLUMN` without rewrite, hash partitioning | 11 |
  | `REINDEX CONCURRENTLY`, stored generated columns, `SET NOT NULL` skipping the scan via a validated `CHECK`, CTEs inlined by default | 12 |
  | `gen_random_uuid()` in core (no `pgcrypto`) | 13 |
  | `scram-sha-256` as default `password_encryption` | 14 |
  | `UNIQUE NULLS NOT DISTINCT`, `MERGE`, `public` schema no longer writable by everyone | 15 |
  | native `uuidv7()`, virtual generated columns | 18 |

  A feature above the project's version → use the older idiom and say so in
  the plan.

## Types

- **`text`** over `varchar(n)` unless the length limit is a real business rule —
  there is no performance difference, and `varchar(n)` makes widening a column
  a needless migration. If a limit matters, prefer `text` + `CHECK (length(col) <= n)`.
- **`timestamptz`**, never `timestamp` (without time zone) — the latter stores
  a wall-clock value with no zone and silently shifts meaning when the session
  `TimeZone` changes. Store UTC instants as `timestamptz`.
- **`numeric(p,s)`** for money and exact quantities, never `real`/`double
  precision`/`money`. Use `bigint` over `integer` for any key that could
  outgrow 2^31.
- **`uuid`** is a native type (16 bytes) — never store UUIDs as `text`/`char(36)`.
- Prefer **`GENERATED ... AS IDENTITY`** over `serial`/`bigserial` — standard
  SQL, no hidden sequence ownership quirks, and `GENERATED ALWAYS` blocks
  accidental manual id inserts.
- **`enum` types** are hard to change (values can be added, not removed, and
  `ALTER TYPE ... ADD VALUE` has transaction caveats). For a set that will
  evolve, use a lookup table with a FK or `text` + `CHECK`.

## Primary keys & identifiers

- Unlike InnoDB, PostgreSQL heaps are **not clustered on the PK**, so a random
  UUID v4 doesn't fragment the table itself — but it still bloats and slows the
  PK **index** (random B-tree inserts, poor cache locality, more WAL via
  full-page writes). The naming section's UUID v7 / ULID recommendation still
  applies for write-heavy tables; use `identity` bigint when ids aren't exposed.
- `CLUSTER` is a one-time physical reorder, not maintained — don't plan around it.

## Schemas & partitioning

- **Dedicated schemas** per application or tenant; don't create objects in
  `public`. **Schema-qualify** objects in migrations (`billing.invoices`) so
  the result doesn't depend on `search_path`.
- Separate the **schema owner** (runs migrations) from the **application
  role** (DML only), and manage grants through **group roles**; add a
  read-only role for analysts.
- **Declarative partitioning** (`PARTITION BY RANGE/LIST/HASH`) for very large
  tables, typically by date: it gives partition pruning and cheap archival
  (`DETACH`/`DROP` a partition instead of `DELETE`). Queries must filter on
  the partition key to benefit; the PK/unique constraints must include it;
  state the key and retention in the plan.

## Constraints

- `CHECK`, `NOT NULL`, `UNIQUE`, `FOREIGN KEY` are all enforced; this engine
  has no gap versus the generic Constraints & integrity rule.
- **Exclusion constraints** (`EXCLUDE USING gist`) express rules `UNIQUE`
  can't — e.g. no two overlapping reservations for the same room. Use them
  instead of app-side overlap checks; list them in the enforcement table.
- `UNIQUE` treats `NULL`s as distinct; use `UNIQUE NULLS NOT DISTINCT` (PG 15+)
  or a partial unique index when "at most one NULL" is the intended rule.
- Add FKs as `NOT VALID` then `VALIDATE CONSTRAINT` on a large table (see
  Online DDL). **Foreign key columns are not auto-indexed** — index the
  referencing column yourself, or deletes/updates on the parent seq-scan the child.

## Indexing

- **B-tree** is the default; reach for others deliberately and record why in
  the indexing plan:
  - **GIN** — `jsonb`, arrays, full-text (`tsvector`), `pg_trgm` for `LIKE '%x%'`.
  - **GiST** — ranges, geometry (PostGIS), exclusion constraints.
  - **BRIN** — huge append-only tables with natural physical ordering (e.g.
    time-series); tiny, but useless on unordered data.
- **Partial indexes** (`WHERE deleted_at IS NULL`) and **expression indexes**
  (`lower(email)`) are first-class — prefer them to indexing everything.
- **`INCLUDE`** columns (PG 11+) give covering/index-only scans without
  widening the key.
- `CREATE INDEX CONCURRENTLY` for any index on a live table (cannot run inside
  a transaction block — check the migration tool supports that).
- Index-only scans depend on the **visibility map**, kept current by autovacuum;
  a table that's never vacuumed won't get them.

## Queries

- Use `EXISTS` for existence checks, not `COUNT(*) > 0`.
- Run `EXPLAIN ANALYZE` on CTEs, window functions, and `LATERAL` joins before
  shipping. Since PG 12 a CTE is inlined unless written `AS MATERIALIZED` (or
  referenced more than once), so older advice about CTEs as "optimization
  fences" no longer holds.
- Keep complex SQL in the repo under migrations/versioned files, not in
  strings hidden in the ORM.

## Online DDL & migrations

- Most DDL takes an `ACCESS EXCLUSIVE` lock; the danger is the **lock queue**:
  a DDL waiting behind a long-running transaction blocks every query behind it.
  Set `lock_timeout` (e.g. `SET lock_timeout = '3s'`) on migrations and retry,
  rather than letting them wait indefinitely.
- Cheap on modern PG: adding a nullable column, or a column with a constant
  `DEFAULT` (PG 11+, metadata-only). Expensive: changing a column type
  (table rewrite), adding a volatile-default column.
- Safe patterns for large tables: `ADD CONSTRAINT ... NOT VALID` →
  `VALIDATE CONSTRAINT`; `CREATE INDEX CONCURRENTLY`; for `SET NOT NULL`, first
  add a validated `CHECK (col IS NOT NULL)` (PG 12+ then skips the scan).
- DDL is **transactional** — a failed migration rolls back fully. Use this:
  wrap multi-step migrations in one transaction (except `CONCURRENTLY` ops).

## Transactions & locking

- Default isolation is **`READ COMMITTED`**. `REPEATABLE READ` is snapshot
  isolation; `SERIALIZABLE` (SSI) can abort with `40001` — callers must retry.
- Row-level locking: `SELECT ... FOR UPDATE`, and **`FOR UPDATE SKIP LOCKED`**
  for queue-style workers. `FOR NO KEY UPDATE` is enough when the key isn't
  changing and blocks fewer FK-checking inserts.
- Use **advisory locks** (`pg_advisory_xact_lock`) for application-level
  mutual exclusion instead of inventing a lock table.
- **Batch large updates/deletes** (e.g. 1–10k rows per transaction, keyed by
  PK range) — one giant transaction holds locks, bloats the table, and delays
  vacuum.
- Keep transactions short: an idle-in-transaction session holds back vacuum
  and bloats tables. Set `idle_in_transaction_session_timeout`.

## MVCC, bloat & vacuum

- Updates and deletes leave dead tuples; **autovacuum** reclaims them. Hot,
  high-churn tables may need per-table `autovacuum_*` tuning — note it in the plan.
- Tune autovacuum **per table** for large or high-churn tables — the defaults
  (`autovacuum_vacuum_scale_factor` 0.2 = 20% of rows dead) are too lax at
  scale: lower `autovacuum_vacuum_scale_factor` / `autovacuum_vacuum_threshold`
  on that table via `ALTER TABLE ... SET (...)`.
- Run **`ANALYZE`** after a bulk load or mass change so the planner has fresh
  statistics.
- Measure index/table bloat with `pgstattuple` before reaching for `pg_repack`.
- Frequent updates of an indexed column defeat **HOT updates**; leave free
  space (`fillfactor` < 100) on update-heavy tables.
- `VACUUM FULL` rewrites the table under an exclusive lock — use `pg_repack`
  for online reclaim.

## JSON

- **`jsonb`**, never `json` (no indexing, reparsed on every access).
- Same stance as the generic Normalization section: fine for schemaless or
  write-once payloads, not a shortcut around a child table you'll query.
- Index with **GIN** (`jsonb_path_ops` for containment `@>` only — smaller and
  faster), or a **B-tree expression index** on one extracted path
  (`((data->>'status'))`) for equality/range on a known key.

## Security

- **Row-Level Security**: `ENABLE ROW LEVEL SECURITY` plus policies; the table
  owner bypasses RLS unless `FORCE ROW LEVEL SECURITY` is set, and superusers /
  `BYPASSRLS` roles always bypass it. The app must connect as a non-owner role.
- Revoke the default: `REVOKE ALL ON SCHEMA public FROM PUBLIC` (PG < 15) and
  grant least-privilege roles per the generic data-protection section.
- `SECURITY DEFINER` functions run as their owner — pin `SET search_path`
  inside them, or a caller can hijack name resolution.
- Parameterize via the driver's bind protocol; never build SQL with `format('%s')`
  from input (use `%I` / `%L` / `quote_ident` only for allowlisted identifiers).

## Connections

- Each connection is a **process** (~MBs); thousands of direct connections
  hurt. Put **PgBouncer** (or the managed equivalent) in front for many
  clients, and note the pooling mode in the plan: transaction pooling breaks
  session state — session-level `SET`, advisory locks (non-xact), `LISTEN`,
  and (older) prepared statements.
