# Roadmap

Working notes for the `db-standards` preset. Not shipped documentation — this
tracks what's done and what's next so we can pick up between sessions.

## Shipped

See `CHANGELOG.md` for the per-version breakdown. Current: **v0.6.2**.

- **speckit.constitution** (command, append) → requires confirming the target
  database (engine, version, migration tool) *(v0.2.0)* and the Data protection
  posture (roles, RLS) *(v0.4.0)* with the user before drafting; don't infer
  silently. Encryption at rest is a usage recommendation, not a confirmed field
  *(v0.5.0)*.
- **constitution-template** (append) → *Database* section (engine / version /
  migration tool; engine change = MAJOR amendment) *(v0.2.0)* and *Data
  protection* section (app role, read split, RLS policy, column classification)
  *(v0.4.0)*; encryption at rest is a recommendation comment *(v0.5.0)*; column
  classification is a recommendation, not mandatory *(v0.6.0)*.
- **spec-template** → *Data Requirements*: entities, reads/writes, retention,
  consistency, non-functional expectations.
- **plan-template** → *Database Design*:
  - Naming conventions (tables, columns, PK/FK, indexes, timestamps, enums, money).
  - Normalization: classify each table OLTP vs OLAP.
    - OLTP → 3NF, with a denormalization "debt" alert (causes + costs) and a
      recording table.
    - OLAP → denormalized by principle only, no prescribed modeling style
      (derived/rebuildable, idempotent load, declared grain, carried business
      key, explicit history decision).
  - Schema changes (reversibility, backfill, zero-downtime).
  - Constraints & integrity: DB is where an invariant is guaranteed (app
    validation is UX only); invariant → enforcement table; `NOT NULL` default,
    explicit FK `ON DELETE`, `UNIQUE` as a constraint, `CHECK` for expressible
    single-row rules, multi-row/table invariants in a transaction or trigger,
    DB-side defaults. *(v0.3.0)*
  - Data protection: class every new column, state handling above `internal`,
    secrets never plaintext, multi-tenant RLS, least-privilege grants. *(v0.4.0)*
  - Identifiers exposed externally: no sequential ID on an external surface;
    expose UUID/ULID or an opaque token; per-object authorization. *(v0.4.0)*
  - SQL portability: ANSI-first; flag vendor functions/procs/triggers for
    migration risk; propose the standard alternative.
  - Query safety (SQL injection): bound parameters only; `LIKE` / `IN`
    handling; dynamic identifiers and `ORDER BY` from a code allowlist;
    `LIMIT`/`OFFSET` cast to int; alert table for raw dynamic SQL from input;
    least-privilege DB role pointer.
  - Indexing plan (query → columns → index).
  - N+1 and query volume (eager/batch load, query-count budget with a test,
    keyset pagination).
  - "Target database" pointer at the top → read engine/version from the
    constitution; stop and resolve it there if unset.
- **tasks-template** → *Schema Review Checklist*: naming, integrity (expanded in
  v0.3.0), normalization (OLTP + OLAP), SQL portability, query safety (SQL
  injection), performance, migration safety, data handling + external-identifier
  exposure (v0.4.0).

## Next up

### Backlog (from the menu, roughly priority order)

1. **Migration discipline** — expand-contract, one logical change per migration,
   no DDL + large backfill in one transaction, `lock_timeout` / `statement_timeout`
   on DDL, tested rollback, forward-only in prod.
2. **Data types & precision** — `timestamptz` in UTC, `text` over arbitrary
   `varchar(n)`, declared `numeric` precision, no nullable boolean, encoding /
   collation.
3. **Transactions & concurrency** — short transactions, deliberate isolation
   level, optimistic locking (`version` column) vs `SELECT FOR UPDATE`,
   idempotency keys for retryable operations.

### Situational (later, if wanted)

4. Auditing & history (`created_by` / `updated_by`, history / audit-log table).
5. Safe column/table removal (stop writing → stop reading → drop) — pairs with
   expand-contract.
6. Query observability (`EXPLAIN` for hot queries in the plan, slow-query log,
   query comments for tracing).
7. JSON / semi-structured columns (when acceptable, schema validation, GIN
   index, no deep-path access on hot paths).

### Done

- ~~Security & data protection~~ → v0.4.0 (constitution posture + per-feature
  column classification; classification softened to a recommendation in v0.6.0).
- ~~ID exposure~~ → v0.4.0 (no sequential ID on external surfaces).

### Out of scope

Backup / DR, sharding, connection pooling, server tuning — operational concerns,
not spec-driven design.

## Satellite presets (per engine)

`db-standards` stays ORM-agnostic/ANSI-first. Engine-specific best practices
(where a vendor's own idiom is the *right* answer, not a portability risk to
flag) go in **separate, stackable presets** — one repo per engine, each a
sibling of this one, each declaring `db-standards` as a soft dependency
(`requires.extensions`-style, or just documented as "install this after
`db-standards`"). A project installs the generic preset plus the one satellite
matching its `[DATABASE_ENGINE]`.

Why satellites and not sections inside this preset: a single project targets
one engine, so Oracle-specific guidance is dead weight (or actively wrong) in a
MySQL project's `plan-template`, and `append` composition can't branch on a
constitution value at install time. Splitting also lets each engine track its
own version independently.

| Engine | Repo (planned) | Preset id | Relational, fits `db-standards` model? |
|--------|-----------------|-----------|------------------------------------------|
| Oracle | `spec-kit-preset-db-oracle` | `db-oracle` | Yes — extends the same 3NF/OLTP/OLAP model |
| MySQL / MariaDB | `spec-kit-preset-db-mysql` | `db-mysql` | Yes |
| SQL Server | `spec-kit-preset-db-sqlserver` | `db-sqlserver` | Yes |
| DynamoDB | `spec-kit-preset-db-dynamodb` | `db-dynamodb` | **No** — NoSQL, access-pattern-first modeling. Inverts several `db-standards` defaults (denormalization is the *starting point*, not debt; no FK/JOIN/SQL-injection sections apply at all). Scope it as its own model, not "Dynamo notes bolted onto the SQL template." |

### Suggested build order

1. **DynamoDB first** — most different from what exists, most likely to surface
   a stacking/composition problem early (e.g. does its `plan-template` addendum
   need `strategy: replace` on the "Database Design" section instead of
   `append`, since most of the SQL-shaped content doesn't apply?).
2. **SQL Server** — largest likely user base after Postgres/MySQL for this kind
   of preset.
3. **MySQL/MariaDB**.
4. **Oracle**.

Reorder freely — this is a guess, not a commitment.

### Planned scope per engine (draft — refine when each satellite starts)

**Oracle**
- Bind variables mandatory (`cursor_sharing`, avoid hard-parse storms) — sharpens this preset's Query Safety section for Oracle specifically.
- `NUMBER` precision/scale discipline (no fixed byte size like other engines' numeric types).
- `VARCHAR2` byte vs char semantics (`NLS_LENGTH_SEMANTICS`) — a real portability gotcha this preset's SQL-portability section can only gesture at generically.
- Sequences vs `IDENTITY` columns (12c+) for PK generation.
- Partitioning strategy (range/hash/list) for large tables.
- Bitmap indexes for low-cardinality reporting columns vs B-tree default.
- PL/SQL package boundaries: when a stored procedure is the right call vs the portability cost flagged upstream — Oracle culture leans on PL/SQL more than most; this preset should say when that's earned.
- Materialized views with fast refresh for the OLAP track.

**MySQL / MariaDB**
- InnoDB only — never MyISAM (transactions, FK support, crash recovery).
- `utf8mb4` always, never `utf8` (which is a 3-byte subset); collation choice and its case-sensitivity implications.
- PK choice affects physical layout: InnoDB clusters the table on the PK. Ever-increasing key vs UUID trade-off is sharper here than on other engines.
- `CHECK` constraints are parsed-but-unenforced before 8.0.16 — a concrete case where this preset's generic "CHECK for every single-row rule" needs an MySQL-version caveat.
- Online DDL: which `ALTER TABLE` operations get `ALGORITHM=INSTANT`/`INPLACE` vs a full table rebuild/lock.
- Default isolation (`REPEATABLE READ`) and gap locks — deadlock patterns to design around.
- Replication mode (statement vs row-based) implications for non-deterministic expressions in writes.

**SQL Server**
- Clustered index is a separate decision from the PK — default (PK = clustered) is often wrong for an ever-increasing key under write concurrency.
- Covering indexes via `INCLUDE` columns vs widening the key.
- `READ_COMMITTED_SNAPSHOT` / snapshot isolation to avoid reader/writer blocking — the concrete answer to this preset's generic "choose isolation deliberately."
- System-versioned temporal tables as the built-in answer to this preset's "history is a decision" (OLAP section) — often better than a hand-rolled audit table.
- `IDENTITY` vs `SEQUENCE`; schema-qualified object names always (`dbo.table`).
- Parameter sniffing — when to `OPTION (RECOMPILE)` vs redesign the query.

**DynamoDB** (own model — see note above)
- Access patterns are enumerated *before* any table design — inverts this preset's "normalize first" spec step entirely for this engine.
- Single-table design as the default; partition-key design for even distribution (hot-partition avoidance).
- Sort-key design for hierarchical/range access.
- GSI/LSI: projection type (`ALL`/`KEYS_ONLY`/`INCLUDE`), GSI eventual consistency, sparse indexes.
- Item size limit (400 KB) and attribute design.
- Idempotent writes via `ConditionExpression`, not app-level locking.
- TTL attribute for expiry instead of a delete job.
- Streams as the equivalent of this preset's OLAP "derived, rebuildable" section.
- `TransactWriteItems` limits (100 items / 4 MB) as the ceiling for this preset's "multi-row invariant" guidance.

## Publishing (not started)

- Each feature/fix: bump `preset.yml` + CHANGELOG, commit to `main`, `git tag
  vX.Y.Z`, GitHub release.
- PR to `github/spec-kit`: add to `presets/catalog.community.json` (sorted by id)
  and a row in `docs/community/presets.md`.
- Maintainer applies the `preset-submission` label → catalog-validation workflow.
