# MySQL / MariaDB — engine-specific guidance

Read this alongside the `db-standards` plan sections it sharpens. It doesn't
repeat anything already covered generically — only where MySQL/MariaDB's own
behavior is the right answer, or where it quietly breaks a generic assumption.

## Storage engine

- **InnoDB, always.** Never `MyISAM` — no transactions, no foreign-key support,
  table-level locking. If a table doesn't declare `ENGINE=InnoDB` explicitly,
  say why in the plan's Schema changes table.
- InnoDB **clusters the table on the primary key** — rows are physically stored
  in PK order. This makes the PK choice in Naming conventions matter more here
  than on most engines:
  - A UUID (v4, random) primary key causes random-order inserts across the
    whole B-tree, fragmenting it and thrashing the buffer pool on a large table.
  - Prefer **UUID v7 / ULID** (time-ordered) as the naming section already
    recommends, or a plain `AUTO_INCREMENT` bigint, specifically *because* of
    this clustering behavior — not just as a general id-shape preference.
  - If a natural UUID v4 is required for another reason, keep it as a
    secondary unique key and cluster on a surrogate `AUTO_INCREMENT` instead.

## Character set & collation

- **`utf8mb4`**, never `utf8` — MySQL's `utf8` is a legacy 3-byte-max encoding
  that cannot store most emoji or some CJK characters; it silently truncates or
  errors on insert.
- Set it at the database, table, *and* connection level — a mismatched
  connection charset re-introduces mangled data even with the schema correct.
- Pick collation deliberately and record it in the constitution's Database
  section: `utf8mb4_0900_ai_ci` (or `_as_cs` for accent/case-sensitive) on
  MySQL 8; on MariaDB, `utf8mb4_uca1400_ai_ci` or the classic
  `utf8mb4_unicode_ci`. Case-insensitive is the common default — state it
  explicitly if a column (e.g. a slug used as a lookup key) needs
  case-sensitive comparison instead.

## Constraints — a real gap versus the generic rule

- **`CHECK` constraints are parsed but silently unenforced before MySQL 8.0.16**
  and on older MariaDB. Constraints & integrity's "CHECK for every expressible
  single-row rule" needs a version gate: confirm the engine version in the
  constitution before relying on `CHECK` for anything you can't afford to have
  silently ignored. Below 8.0.16, enforce the same rule with a `BEFORE INSERT/
  UPDATE` trigger instead, and record that substitution in Constraints &
  integrity's enforcement table.
- Foreign keys, `NOT NULL`, and `UNIQUE` are enforced normally on InnoDB — only
  `CHECK` has the gap.

## Online DDL

- Before altering a large table in production, check whether the operation
  supports `ALGORITHM=INSTANT` (MySQL 8.0.12+: adding a column, some renames) or
  `ALGORITHM=INPLACE` (most index/column operations, no table rebuild, brief
  metadata lock) versus the default `COPY` (rebuilds the whole table, holds a
  long lock).
- State the algorithm in the plan's Schema changes / Zero-downtime concerns —
  this is the concrete mechanism behind that generic bullet on this engine.
- `pt-online-schema-change` / `gh-ost` are the fallback for an operation MySQL
  itself can't do online (e.g. changing a column's type on older versions).

## Transactions & locking

- Default isolation is **`REPEATABLE READ`**, and InnoDB adds **gap locks** on
  indexed range scans under it — a common source of deadlocks that don't occur
  under plain read-committed engines. When Transactions & concurrency (backlog)
  applies here: prefer narrow, indexed `WHERE` predicates in transactional
  updates to minimize gap-lock scope, or switch to `READ COMMITTED` when gap
  locking isn't needed for the invariant being protected.
- Statement-based vs row-based replication changes what's safe to write:
  under statement-based replication, a write using `NOW()`, `RAND()`, or a
  non-deterministic function can diverge between primary and replica. Confirm
  the replication format in the constitution's Database section (Migration
  tool / notes) before relying on such a function in a write path; row-based
  (the modern default) avoids the issue.

## JSON columns

- Same rule as the generic Normalization section's stance on JSON: fine for
  genuinely schemaless or write-once payloads, not a shortcut around a child
  table you'll need to query.
- Index a specific JSON path with a **generated column + index** on it
  (`col GENERATED ALWAYS AS (JSON_EXTRACT(...))`, then index the generated
  column) rather than scanning the JSON document per query.
