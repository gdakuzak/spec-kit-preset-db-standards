# Roadmap

Working notes for the `db-standards` preset. Not shipped documentation — this
tracks what's done and what's next so we can pick up between sessions.

## Shipped

See `CHANGELOG.md` for the per-version breakdown. Current: **v0.2.0**.

- **speckit.constitution** (command, append) → requires confirming the target
  database (engine, version, migration tool) with the user before drafting;
  don't infer silently. *(v0.2.0)*
- **constitution-template** (append) → *Database* section (engine / version /
  migration tool) as a project-wide constraint; engine change = MAJOR amendment.
  *(v0.2.0)*
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
- **tasks-template** → *Schema Review Checklist*: naming, integrity,
  normalization (OLTP + OLAP), SQL portability, query safety (SQL injection),
  performance, migration safety, data handling.

## Next up

### Backlog (from the menu, roughly priority order)

1. **Migration discipline** — expand-contract, one logical change per migration,
   no DDL + large backfill in one transaction, `lock_timeout` / `statement_timeout`
   on DDL, tested rollback, forward-only in prod.
2. **Constraints & integrity** — `NOT NULL` by default, `CHECK` for domain rules
   (not only in app code), explicit FK actions (`ON DELETE`), `UNIQUE` as a
   constraint.
3. **Data types & precision** — `timestamptz` in UTC, `text` over arbitrary
   `varchar(n)`, declared `numeric` precision, no nullable boolean, encoding /
   collation.
4. **Transactions & concurrency** — short transactions, deliberate isolation
   level, optimistic locking (`version` column) vs `SELECT FOR UPDATE`,
   idempotency keys for retryable operations.
5. **Security & data protection** — app role is not owner, separate read/write
   credentials, PII classification + handling, sensitive-column encryption, no
   plaintext secrets, RLS for multi-tenant.

### Situational (later, if wanted)

6. Auditing & history (`created_by` / `updated_by`, history / audit-log table).
7. Safe column/table removal (stop writing → stop reading → drop) — pairs with
   expand-contract.
8. Query observability (`EXPLAIN` for hot queries in the plan, slow-query log,
   query comments for tracing).
9. JSON / semi-structured columns (when acceptable, schema validation, GIN
   index, no deep-path access on hot paths).
10. ID exposure (surrogate vs natural, don't expose sequential IDs externally —
    enumeration risk).

### Out of scope

Backup / DR, sharding, connection pooling, server tuning — operational concerns,
not spec-driven design.

## Publishing (not started)

- Each feature/fix: bump `preset.yml` + CHANGELOG, commit to `main`, `git tag
  vX.Y.Z`, GitHub release.
- PR to `github/spec-kit`: add to `presets/catalog.community.json` (sorted by id)
  and a row in `docs/community/presets.md`.
- Maintainer applies the `preset-submission` label → catalog-validation workflow.
