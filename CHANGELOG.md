# Changelog

All notable changes to this preset. Versions are tagged `vX.Y.Z` and released on
GitHub. While on `0.x`, a feature bumps the minor and a fix bumps the patch.

## 0.6.1 — 2026-09-09

### Changed

- `README.md`: install URL now points at `main` instead of a version tag, and a
  new "Update" section (remove + re-add) for refreshing an existing install.

## 0.6.0 — 2026-09-09

### Changed

- Column data classification is now recommended, not mandatory. The
  `constitution-template`, `plan-template` "Data protection" section, and the
  schema-review checklist all phrase it as a recommendation.

## 0.5.0 — 2026-09-09

### Changed

- Encryption at rest is no longer a confirmed field in the constitution. The
  `speckit.constitution` command no longer requires an answer and the
  `constitution-template` drops the `[DATABASE_ENCRYPTION_POLICY]` placeholder —
  it's now a usage recommendation (record which data classes are encrypted and
  how, full-disk / tablespace / column, if compliance calls for it).

## 0.4.0 — 2026-09-09

### Added

- `constitution-template` → "Data protection" section: project-wide posture —
  app connects as a non-owner role, reporting reads split off, row-level security
  policy, encryption-at-rest policy, mandatory column data classification.
- `speckit.constitution` (command): also confirms the Data protection posture
  with the user.
- `plan-template` → "Data protection" section: class every new column
  (public / internal / PII / regulated / secret) and state handling; secrets
  never plaintext; multi-tenant RLS; least-privilege grants.
- `plan-template` → "Identifiers exposed externally" section: no sequential ID
  on an external surface — expose UUID/ULID or an opaque token; authorization is
  still per-object.
- `tasks-template` → "Data handling" checklist expanded; new "Identifiers
  exposed externally" checklist.

## 0.3.0 — 2026-09-09

### Added

- `plan-template` → "Constraints & integrity" section: the database is where an
  invariant is guaranteed (app validation is UX only); per-table invariant →
  enforcement table; `NOT NULL` by default, explicit FK `ON DELETE` actions,
  `UNIQUE` as a constraint, `CHECK` for every expressible single-row rule,
  multi-row/multi-table invariants in a transaction or trigger, DB-side defaults.
- `tasks-template` → "Integrity" checklist expanded to match.

## 0.2.0 — 2026-09-09

### Added

- `speckit.constitution` (command, `append`): requires the user to confirm the
  target database — engine, major version, migration tool — before the
  constitution is drafted; the value is not inferred silently.
- `constitution-template` (`append`): a "Database" section holding engine /
  version / migration tool. Changing the engine after ratification is a MAJOR
  constitution amendment.
- `plan-template`: a "Target database" pointer at the top of "Database Design" —
  read the engine/version from the constitution, stop and resolve it there if
  unset.

## 0.1.0 — 2026-09-08

Initial preset. All overrides use the `append` strategy.

### Added

- `spec-template` → "Data Requirements": entities, reads/writes, retention,
  consistency, non-functional expectations, and the external input that reaches
  the database.
- `plan-template` → "Database Design":
  - Naming conventions (tables, columns, PK/FK, indexes, timestamps, enums, money).
  - Normalization: classify each table OLTP vs OLAP. OLTP targets 3NF with a
    denormalization "debt" alert; OLAP is denormalized by principle only (no
    prescribed modeling style).
  - Schema changes (reversibility, backfill, zero-downtime).
  - SQL portability: ANSI-first; flag vendor functions/procs/triggers for
    migration risk; propose the standard alternative.
  - Query safety (SQL injection): bound parameters only, `LIKE` / `IN` handling,
    allowlisted dynamic identifiers, alert table for raw dynamic SQL.
  - Indexing plan (query → columns → index).
  - N+1 and query volume (eager/batch load, query-count budget with a test,
    keyset pagination).
- `tasks-template` → "Schema Review Checklist": naming, integrity, normalization
  (OLTP + OLAP), SQL portability, query safety, performance, migration safety,
  data handling.
