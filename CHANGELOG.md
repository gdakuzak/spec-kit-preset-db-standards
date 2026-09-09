# Changelog

All notable changes to this preset. Versions are tagged `vX.Y.Z` and released on
GitHub. While on `0.x`, a feature bumps the minor and a fix bumps the patch.

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
