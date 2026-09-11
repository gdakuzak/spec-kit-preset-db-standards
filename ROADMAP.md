# Roadmap

Working notes for the `db-standards` preset. Not shipped documentation — this
tracks what's done and what's next so we can pick up between sessions.

## Shipped

See `CHANGELOG.md` for the per-version breakdown. Current: **v0.7.0**.

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
- **references/{oracle,mysql,sqlserver,dynamodb}.md** → engine-specific best
  practices, not appended to any template; `plan-template`'s "Target database"
  line points at the matching one. *(v0.7.0)* — see "Engine-specific reference
  files" below for what each covers.

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

## Engine-specific reference files

~~Considered separate satellite presets (one repo per engine) — reverted~~: too
much ceremony for four repos that all just sharpen the same generic sections.
Instead, engine-specific best practices live **as files inside this preset**,
under `references/<engine>.md` — one file per engine, shipped with this repo,
not appended into any template. The "Target database" line at the top of the
plan's Database Design section points at the matching file by the constitution's
`[DATABASE_ENGINE]`.

Why this works without changing `preset.yml`'s schema: `specify preset add`
copies the whole preset directory to `.specify/presets/db-standards/`, not just
the files declared in `provides` — so `references/*.md` ships and is on disk
for the agent to read, it's just never composed into a core template.

| Engine | File | Relational, fits the `db-standards` model? |
|--------|------|-----------------------------------------------|
| Oracle | `references/oracle.md` | Yes — extends the same 3NF/OLTP/OLAP model |
| MySQL / MariaDB | `references/mysql.md` | Yes |
| SQL Server | `references/sqlserver.md` | Yes |
| DynamoDB | `references/dynamodb.md` | **No** — NoSQL, access-pattern-first modeling. Inverts several `db-standards` defaults (denormalization is the *starting point*, not debt; no FK/JOIN/SQL-injection sections apply). Written as its own model, not "Dynamo notes bolted onto the SQL template." |

**Status: shipped** *(v0.7.0)* — full content is in the four `references/*.md`
files themselves; don't duplicate it here. Future engine-guidance work (a new
engine, or expanding an existing file) is tracked as a normal backlog item
below, not in this section.

## Publishing (not started)

- Each feature/fix: bump `preset.yml` + CHANGELOG, commit to `main`, `git tag
  vX.Y.Z`, GitHub release.
- PR to `github/spec-kit`: add to `presets/catalog.community.json` (sorted by id)
  and a row in `docs/community/presets.md`.
- Maintainer applies the `preset-submission` label → catalog-validation workflow.
