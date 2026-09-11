# SQL Server — engine-specific guidance

Read this alongside the `db-standards` plan sections it sharpens. It doesn't
repeat anything already covered generically — only where SQL Server's own
idiom is the right answer.

## Clustered index is a separate decision from the primary key

SQL Server defaults to clustering the table on the primary key, but the PK
constraint and the clustered index are independent choices (`PRIMARY KEY
NONCLUSTERED` is valid). Naming conventions prefers a non-sequential id
(UUID v7 / ULID); on SQL Server that preference needs a caveat:

- If the PK is a random-order key (UUID v4, or a non-time-ordered GUID) and it
  is also the clustered index, every insert lands at a random point in the
  B-tree — page splits and fragmentation, the same failure mode called out for
  MySQL/InnoDB.
- Default choice: cluster on a narrow, ever-increasing key (an `IDENTITY`
  bigint, or `NEWSEQUENTIALID()` if a GUID is required) and keep the naming
  section's preferred id as a `UNIQUE NONCLUSTERED` key if it needs to be the
  public-facing identifier.
- State the clustered-index column explicitly in the plan's Schema changes
  table — don't leave it to the PK default.

## Indexing

- **Covering indexes**: add non-key columns via `INCLUDE (...)` instead of
  widening the index key — keeps the key narrow (cheaper for other lookups)
  while still avoiding a bookmark lookup for the hot query in the plan's
  Indexing plan table.
- Filtered indexes (`WHERE deleted_at IS NULL`, `WHERE status = 'active'`) are
  this engine's version of the generic partial-index rule in Indexing plan.

## Isolation & concurrency

- Default isolation (`READ COMMITTED`) uses locking by default, so readers
  block writers and vice versa. Turn on **`READ_COMMITTED_SNAPSHOT`**
  (database-level, no application change) so readers see a transactionally
  consistent snapshot instead of blocking — this is the concrete SQL Server
  answer to the generic Transactions & concurrency backlog item's "choose
  isolation deliberately."
- `SNAPSHOT` isolation (opt-in per transaction) is stronger still, at the cost
  of update-conflict errors the app must retry on. Use it only where that
  retry path already exists (ties into the optimistic-locking pattern from
  the same backlog item).
- Record the chosen isolation setting in the constitution's Database section
  once decided — it's a project-wide default, not a per-feature choice.

## Temporal tables (the OLTP answer to "history is a decision")

- `db-standards`' Normalization section asks every entity to decide overwrite
  vs. keep-history explicitly. On SQL Server, **system-versioned temporal
  tables** (`WITH (SYSTEM_VERSIONING = ON)`) are usually the right mechanism
  for "keep history" on an OLTP table — the engine maintains the history table
  and point-in-time queries (`FOR SYSTEM_TIME AS OF`) automatically, instead of
  a hand-rolled audit table with triggers.
- Prefer it over a manual audit table unless the history needs a shape the
  automatic history table can't give you (selective columns, custom retention
  per row).

## Identifiers & naming

- `IDENTITY(1,1)` for a plain auto-incrementing surrogate key; a `SEQUENCE`
  object only when the value must be generated before insert or shared across
  tables — same rule as the Oracle reference, different syntax.
- Every object reference is schema-qualified (`dbo.orders`, not `orders`) —
  an unqualified name resolves against the caller's default schema, which is a
  live footgun once more than one schema exists in the database.

## Query plans & parameters

- **Parameter sniffing**: a query plan compiled for one parameter's data
  distribution gets reused for a very different one, producing a plan that's
  fast for the first caller and terrible for the next. When a plan's
  performance varies wildly by input (visible in `sys.dm_exec_query_stats`),
  reach for `OPTION (RECOMPILE)` on that one statement, or restructure the
  query, before reaching for a server-wide setting.
- This is a refinement of the generic N+1 / query-volume section's query-count
  budget: a query can pass the count budget and still regress badly here on
  plan reuse — worth a note in the plan when a query's input distribution is
  genuinely skewed.
