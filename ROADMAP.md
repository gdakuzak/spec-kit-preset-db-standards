# Roadmap

Working notes for the `db-standards` preset. Not shipped documentation — this
tracks what's done and what's next so we can pick up between sessions.

## Shipped

See `CHANGELOG.md` for the per-version breakdown. Current: **v0.6.1**.

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

## Publishing (not started)

- Each feature/fix: bump `preset.yml` + CHANGELOG, commit to `main`, `git tag
  vX.Y.Z`, GitHub release.
- PR to `github/spec-kit`: add to `presets/catalog.community.json` (sorted by id)
  and a row in `docs/community/presets.md`.
- Maintainer applies the `preset-submission` label → catalog-validation workflow.
