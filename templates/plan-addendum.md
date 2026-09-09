## Database Design

<!--
  Added by the db-standards preset. Fill this in during /speckit.plan.
  This is the HOW for the "Data Requirements" section of the spec.
-->

**Target database**: [read the engine + version from the constitution's Database
section]. If it is unset or `TODO` there, stop and resolve it via
`/speckit.constitution` before planning schema work — portability, type, and
migration decisions below all depend on it.

### Naming conventions

These are the rules a schema in this project must follow. Call out any
deliberate exception and why.

- **Tables**: `snake_case`, plural (`invoices`, `line_items`).
- **Columns**: `snake_case`, singular. Booleans read as a predicate (`is_active`, `has_shipped`).
- **Primary key**: `id`. Prefer a non-sequential id (UUID v7 / ULID) unless there is a reason for a bigint sequence.
- **Foreign keys**: `<referenced_table_singular>_id` (`invoice_id`). Always add a real FK constraint.
- **Timestamps**: `created_at`, `updated_at` (timezone-aware). Soft delete is `deleted_at` (nullable), never a boolean.
- **Indexes**: `ix_<table>__<col>[_<col>]`. Unique: `ux_<table>__<cols>`. Constraints: `ck_<table>__<rule>`.
- **Join tables**: both singulars, alphabetical (`project_user`).
- **Enums**: store as text with a `CHECK` or a native enum type — not integer codes.
- **Money**: `numeric(19,4)` (or integer minor units), never `float`.

### Normalization

**First, classify each table.** The rule is different for the two workloads.

| Class | Role | Source of truth? | Target |
|-------|------|------------------|--------|
| **Transactional (OLTP)** | the app writes here and enforces business rules | yes | **3NF** |
| **Analytical (OLAP)** | reporting, dashboards, exports, ML features; loaded from OLTP by ETL/ELT | no — derived and rebuildable | denormalized on purpose (see below) |

List each table added by this feature and its class:

| Table | Class | Note |
|-------|-------|------|
| [table] | OLTP / OLAP | [...] |

#### OLTP tables → 3NF

**Every non-key column depends on the key, the whole key, and nothing but the
key.** In plain terms: each non-key column describes *the entity identified by
the PK* — nothing else. If a column describes another non-key column, it belongs
in another table. No repeating groups, no derived column computable from others,
no multi-value column (`tags = "a,b,c"`). Reach for BCNF only when candidate keys
overlap.

> ⚠️ **Denormalizing an OLTP table is debt you take on, not free performance.**
> Before duplicating any data, check for a cheaper way out:
>
> | Symptom pushing you to denormalize | Try first |
> |------------------------------------|-----------|
> | Expensive join on a hot list | Covering index; or a `VIEW` |
> | Repeated `COUNT` / `SUM` | Materialized view with a refresh plan; or a counter column with a recompute path |
> | Many joins to build one screen | `VIEW`; or assemble the projection in the app |
> | Read across services / domains | An explicit read model / cache — not a duplicated column on the transactional table |
> | Reporting / history | A separate OLAP table (below) — don't bolt reporting columns onto OLTP |
>
> What you start paying once you denormalize:
> - **Update anomaly** — the same fact in N places; a write can update only some
> - **Silent divergence** when the sync mechanism has a bug or a race
> - Every **write** gets more expensive and more coupled
> - **Migrations and backfills** get riskier
> - **Ambiguous truth** — which copy is right when they disagree?

If you denormalize an OLTP table anyway, record each case:

| Denormalization | What is duplicated / precomputed | Query problem it solves (with numbers) | How it stays correct | Staleness tolerated |
|-----------------|----------------------------------|----------------------------------------|----------------------|---------------------|
| [e.g. `orders.customer_name`] | copy of `customers.name` | [e.g. order-list join measured at Xms, Y rows] | [trigger / app write / scheduled rebuild] | [none / N minutes] |

- The normalized form stays the source of truth; the denormalized copy is a cache.
- Name the sync mechanism and where it lives. "The app will remember to update it" is not a mechanism.
- A counter/aggregate column (`comments_count`) needs a defined path to recompute it from the source when it drifts.
- JSON/array columns are denormalization: fine for genuinely schemaless or write-once payloads, not a shortcut around a child table you will need to query or join.

#### OLAP tables → denormalized, by principle

Don't apply 3NF here — analytical models are meant to be wide and redundant so
reads are cheap. This preset doesn't prescribe a modeling style (star schema,
one-big-table, etc.); pick one and note it. Whatever the style, hold to:

- **Derived, never authoritative.** The OLTP schema is the source of truth. An
  OLAP table can be dropped and rebuilt from it.
- **Idempotent, documented load.** State the source, the cadence, and whether
  each run is full or incremental. Re-running a load must not double rows or
  corrupt totals.
- **Declared grain.** Each table says, in a comment, what one row represents
  ("one order line per day"). Every measure and dimension must fit that grain.
- **Stable keys.** Carry the OLTP business/natural key so a row can be traced
  back to its source; a surrogate key is fine on top.
- **History is a decision, not an accident.** For each entity say whether the
  table overwrites on change (keep only the current value) or keeps versioned
  rows with a validity range. Don't leave it implicit.
- **Separated from OLTP.** Analytics does not query OLTP tables directly on a
  hot path; OLTP tables do not carry reporting-only columns.

### Schema changes

| Table | Change | New columns / indexes / constraints |
|-------|--------|-------------------------------------|
| [table] | create / alter | [...] |

- Migration is reversible: [yes / no — if no, say why]
- Backfill needed: [none / describe, and whether it runs in the migration or a separate job]
- Zero-downtime concerns: [adding a NOT NULL column, renaming, changing a type, large index build — how each is handled]

### Constraints & integrity

**The database is where a data invariant is guaranteed.** App-side validation is
for UX — early, friendly errors — never the thing that keeps the data correct.
If an invariant can be a constraint, it is a constraint.

List the invariants of each new/changed table and how each is enforced:

| Invariant | Enforcement |
|-----------|-------------|
| [an invoice always has a customer] | `customer_id NOT NULL` + FK |
| [`end_date` is after `start_date`] | `CHECK (end_date > start_date)` |
| [one active membership per user] | partial `UNIQUE (user_id) WHERE status = 'active'` |
| [order total equals the sum of its lines] | application, inside a transaction — crosses rows (see below) |

Rules:

- **`NOT NULL` by default.** A column is nullable only when NULL is a real,
  defined state, and the plan says what NULL means there. No placeholder `''` /
  `0` / `1970-01-01` standing in for "unknown".
- **Every FK is a real constraint** with an explicit action: `ON DELETE RESTRICT`
  (default), `CASCADE` (only for genuinely owned children), or `SET NULL` (only
  if the column is nullable and that state means something). State the choice per FK.
- **`UNIQUE` is a constraint**, not just a unique index by convention. Natural
  keys and "only one of X" rules are `UNIQUE`, partial when the rule is conditional.
- **`CHECK` for every single-row domain rule that is expressible** — ranges,
  allowed values, "at least one of these columns is set", non-negative amounts.
  Don't skip it because the app "already checks".
- **Multi-row / multi-table invariants** (totals match, ranges don't overlap,
  count limits) can't be a simple `CHECK`. Enforce them in the app inside a
  transaction, or in a trigger — and record which and where. This is the one
  place app-side enforcement *is* the guarantee.
- **Defaults belong in the database**, not only in the ORM, so a raw insert
  stays valid.

### Data protection

Follows the constitution's **Data protection** posture. Recommended for this
feature: class every column it adds and state the handling for anything above
`internal`:

| Column / table | Class (public / internal / PII / regulated / secret) | Handling |
|----------------|------------------------------------------------------|----------|
| [`users.email`] | PII | [access via app only; excluded from logs] |
| [`users.password`] | secret | [argon2 hash — never reversible] |
| [`payments.pan`] | regulated | [not stored — tokenized via provider] |

- Ideally every new column has a class. Anything above `internal` names its
  handling: encryption, hashing, tokenization, masking in logs, or access
  restriction.
- **Required:** secrets (passwords, API keys, tokens) are hashed or encrypted — never plaintext.
- Multi-tenant tables carry the tenant key and are covered by the RLS policy, or
  the plan states why app-level scoping is enough here.
- The migration grants the app role only the privileges it needs on new objects
  (no blanket `ALL`).

### Identifiers exposed externally

- Internal primary keys may be sequential (`bigint`). **No sequential ID appears
  on an external surface** — URL, API response, export file, email link.
- What leaves the system is a UUID v7 / ULID, or a separate opaque public token
  column. Map it here:

  | Resource | Internal key | External identifier |
  |----------|--------------|---------------------|
  | [`invoices`] | `id bigint` | `public_id uuid` |

- Exposing an identifier is never a substitute for authorization — every request
  is still checked against the caller's access to that specific object.

### SQL portability

Default to **standard (ANSI) SQL**. It survives a database migration; vendor
extensions do not.

- **Any vendor-specific function, operator, type, or stored procedure/trigger
  must be flagged here**, with: what it is, which database it belongs to, and
  why the standard alternative was not enough.

  | Vendor feature used | Database | Standard alternative | Why the vendor feature anyway |
  |---------------------|----------|----------------------|-------------------------------|
  | [e.g. `jsonb_path_query`] | Postgres | [e.g. none — feature needs it] | [reason] |

- Before reaching for a vendor feature, check whether ANSI SQL already covers
  it and **propose that first**:
  - `COALESCE` / `NULLIF` / `CASE` instead of `IFNULL`, `ISNULL`, `NVL`, `DECODE`
  - `||` (or `CONCAT`) instead of `+` for string concat
  - `CROSS JOIN` / `INNER JOIN ... ON` instead of proprietary join syntax
  - `FETCH FIRST n ROWS ONLY` instead of `LIMIT` / `TOP` where portability matters
  - `EXTRACT(field FROM ts)` instead of `DATE_PART`, `DATEPART`, `strftime`
  - `substring`, `trim`, `upper`, `lower` — standard forms
  - Window functions (`ROW_NUMBER() OVER (...)`) instead of vendor ranking hacks
  - Standard `information_schema` views instead of `pg_catalog` / `sys.*` for introspection
- Business logic in application code or plain SQL, not in stored procedures /
  triggers, unless this section states why the DB is the only correct place.
- If a vendor feature is unavoidable, isolate it (one repository/query module,
  one migration) so a future port has a single place to change.

### Query safety (SQL injection)

Every SQL statement this feature runs is either fully static or built only from
**bound parameters**. User input never reaches a query as concatenated or
interpolated string — not in the app, not inside a stored procedure, not through
an ORM `raw` / `execute` escape hatch.

List every query that takes external input:

| Query / call site | External input it uses | How the input enters the SQL |
|-------------------|------------------------|------------------------------|
| [e.g. `search_invoices` repo method] | `q`, `status`, `sort`, `page` | `q`/`status` → bound params; `sort` → allowlisted column; `page` → cast to int |

Rules:

- **Values** → always bound parameters (`$1`, `?`, `:name`). Never format them
  into the string.
- **`LIKE` patterns** → bind the value and escape `%`, `_`, `\` in it; don't
  build the pattern by concatenation.
- **`IN (...)` lists** → one bound parameter per element (or an array parameter),
  never a joined string.
- **Identifiers** (table / column / schema names) and **`ORDER BY` direction**
  can't be bound. Resolve them through a fixed **allowlist** in code that maps a
  request value to a known-safe identifier. Never pass the request value
  through, even quoted.
- **`LIMIT` / `OFFSET`** from input → parse to a bounded integer.
- ORM query builders are the default. Any raw-SQL escape hatch still uses the
  driver's parameter binding.
- The DB role the app connects as has only the privileges it needs (no `DROP`,
  no `CREATE`, not the table owner) — this caps the blast radius if a hole slips
  through. Full treatment in the Security section.

> ⚠️ **If any query builds SQL from user input without an allowlist**, record it
> here with the reason and the mitigation, and get it reviewed. This is a hole
> by default, not a style choice.
>
> | Query building dynamic SQL from input | Why unavoidable | Mitigation (validation, escaping, isolation) |
> |--------------------------------------|-----------------|---------------------------------------------|
> | [call site] | [reason] | [how the input is constrained] |

### Indexing plan

For every query this feature adds to a hot path, name the index that serves it.

| Query / access pattern | Filter & sort columns | Index |
|------------------------|-----------------------|-------|
| [e.g. "list open invoices for a customer, newest first"] | `customer_id`, `status`, `created_at desc` | `ix_invoices__customer_id_status_created_at` |

- Every foreign key used for lookups or joins has an index.
- Composite index column order = equality columns first, then range/sort.
- No redundant index (one already covered by the leftmost prefix of another).
- Partial index where the query always filters a small subset (`WHERE deleted_at IS NULL`).

### N+1 and query volume

- List every place this feature loads a collection and then touches a related record per item.
- For each: state how the related data is loaded in one query (eager load / join / batched `IN (...)` / dataloader).
- Endpoints returning a list declare a **max query count** budget and are covered by a test that asserts it (query counter / `assert_queries` / echo log).
- No query inside a loop. No ORM lazy-load left implicit on a serialized path.
- Pagination is keyset (seek) for large or unbounded lists; `OFFSET` only for small bounded sets.
