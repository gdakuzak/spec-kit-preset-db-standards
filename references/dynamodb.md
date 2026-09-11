# DynamoDB — engine-specific guidance

DynamoDB is not relational. Most of `db-standards`' plan sections assume a
SQL/relational engine and **do not apply** here — this file replaces them
rather than sharpening them. Use the table below to know what to skip and what
to read instead; the rest of this file is the DynamoDB-native design guide.

| `db-standards` section | On DynamoDB |
|-------------------------|--------------|
| Normalization (3NF target) | **Inverted.** Denormalize by default — see Access patterns first, below. |
| Schema changes (DDL, backfill) | N/A — no fixed schema per item; table/index changes are capacity and key-design decisions, not DDL. |
| Constraints & integrity (`CHECK`, FK) | No `CHECK`/FK. Use `ConditionExpression` on writes for single-item invariants; see Multi-item writes below for the rest. |
| SQL portability | N/A — there is no SQL. |
| Query safety (SQL injection) | N/A as stated, but the *principle* still applies: never build a `KeyConditionExpression` / `FilterExpression` string by concatenating input — use `ExpressionAttributeValues` binding, which the SDK requires by default anyway. |
| Indexing plan | Replaced by GSI/LSI design, below — same goal (one index per real access pattern), different mechanism. |
| N+1 / query volume | Replaced by per-item read/write cost accounting, below. |
| Data protection (column classification) | Still applies — class every attribute, same as a column. Encryption at rest is on by default (AWS owned or KMS key); confirm which in the constitution. |
| Identifiers exposed externally | Still applies unchanged — don't expose a sequential/guessable key externally. |

## Access patterns first (replaces "normalize, then design queries")

This is the core inversion. A relational design starts from the data's shape
(normalize) and lets queries follow. DynamoDB starts from the queries:

1. **List every access pattern this feature needs**, before drafting any table
   or index — this replaces the "Data Requirements" spec step's entity list as
   the thing that drives the design. For each: the operation (get single item,
   query a range, get related items together), the input available at call
   time, and the expected item count / frequency.
2. Design the table (partition key, sort key, and any GSIs) to serve every
   listed access pattern with a `GetItem` or a single `Query` — not a `Scan`,
   not an app-side join across tables.
3. If a new access pattern shows up later that the current key design can't
   serve, that's a new GSI or a denormalized copy — not a `Scan`. Record it as
   a deliberate addition, the same way `db-standards`' denormalization table
   asks OLTP features to record a duplicated column.

## Single-table design

- Default to one table per bounded context (not one table per entity type).
  Store heterogeneous item types together, disambiguated by an item-type
  attribute and key prefix (`PK = "USER#123"`, `SK = "ORDER#456"`).
- This is denormalization by design, not debt: fetching an entity and its
  closely-related children in one `Query` (same partition key, sort-key range)
  is the entire point — it replaces the JOIN a relational engine would do.
- Multi-table is still fine for genuinely independent, differently-accessed
  entities (e.g. an audit-log table with its own access pattern and retention).
  Don't force everything into one table if the access patterns don't share a
  partition key.

## Partition key design

- Choose a partition key with **high cardinality and even access** across its
  values — a hot key (one partition key hit far more than others) throttles
  that partition regardless of the table's overall provisioned/on-demand
  capacity.
- A low-cardinality attribute (status, tenant with one dominant tenant, a
  boolean) is a bad partition key alone — combine it with something
  high-cardinality, or shard it (`STATUS#active#<hash_suffix>`).
- Sequential/monotonic partition keys (a plain incrementing id, a timestamp
  alone) concentrate writes on whichever partition currently owns the newest
  range — prefer a composite or hashed key for high-write-rate tables.

## Sort key design

- Use the sort key to make a `Query` (not `Scan`) serve a range or hierarchy:
  `SK` prefixes (`ORDER#2026-01#`) enable `begins_with`, and a comparable value
  (ISO timestamp, zero-padded sequence) enables `between`/`>`/`<`.
- Composite sort keys (`TYPE#SUBTYPE#ID`) let one partition serve several
  related access patterns via different `begins_with` prefixes — this is the
  DynamoDB equivalent of a composite index's column order in the generic
  Indexing plan section.

## Secondary indexes (GSI / LSI)

- One GSI per access pattern the base table's key can't serve — same
  discipline as "one index per query" in the generic Indexing plan section,
  different unit.
- **Projection type**: `KEYS_ONLY` or `INCLUDE` (specific attributes) by
  default; `ALL` only when the index is genuinely read as often as the base
  table — a full projection doubles storage and write cost for every item.
- **GSIs are eventually consistent** — a write is not guaranteed visible on a
  GSI query immediately after. If a feature needs read-your-write consistency
  on that access pattern, it can't be served by a GSI alone; serve it from the
  base table (strongly consistent `GetItem`/`Query`) or design around the lag.
- LSIs share the base table's partition key and give strong consistency, but
  they're fixed at table-creation time and share the base table's 10 GB/
  partition-key limit — prefer a GSI unless strong consistency on an
  alternate sort key is a hard requirement.

## Item design

- **400 KB item size limit** (hard). A feature storing large blobs, long text,
  or an unbounded list attribute needs a plan for that before it ships —
  typically: store the blob in S3 and keep a reference, or cap/paginate the
  list attribute itself (a growing list on one item is also a hot-item risk).
- Attribute names count toward item size — short, consistent attribute names
  matter more here than in a relational schema.

## Multi-item writes (replaces Constraints & integrity's DB-level guarantee)

- A single item's invariants are enforced with a **`ConditionExpression`** on
  the write (e.g. `attribute_not_exists(PK)` for a create-only insert,
  `version = :expected` for optimistic locking) — this is the DynamoDB
  equivalent of a `CHECK` constraint, and it's mandatory the same way: no
  read-then-write from the app without a condition guarding the write.
- Invariants spanning multiple items use **`TransactWriteItems`** — capped at
  **100 items / 4 MB per transaction**. This cap is the concrete ceiling for
  `db-standards`' generic "multi-row invariant, enforced in a transaction"
  rule: if an invariant could ever span more items than that, it needs a
  different design (a single item holding the aggregate, or an async
  reconciliation process), not a bigger transaction.
- There is no cross-item foreign key. An orphaned reference (a child item
  whose parent was deleted) is prevented by the write path (delete children in
  the same transaction, or a stream-triggered cleanup), not by the database.

## TTL (replaces retention jobs)

- For data with a defined expiry (the spec's Retention & lifecycle section),
  set a **TTL attribute** (epoch seconds) instead of a scheduled delete job —
  DynamoDB expires and removes the item automatically (within ~48h of
  expiry, not exactly on time — don't rely on TTL for a request-time
  "is this still valid" check; check the value directly for that).

## Streams (the OLAP / derived-data answer)

- DynamoDB Streams is the mechanism for `db-standards`' OLAP-track guidance
  ("derived, rebuildable" tables loaded from the source): a Lambda or
  consumer on the stream keeps a search index, an aggregate, or an OLAP/export
  copy up to date.
- Treat the stream consumer the same as any other ETL load in the OLAP
  section: idempotent (safe to reprocess a record), and the derived store is
  never treated as authoritative.

## Cost & capacity as a design input

- On-demand capacity avoids provisioning but doesn't avoid hot-partition
  throttling — partition key design still matters under on-demand.
- Every access pattern's expected read/write volume (from the access-pattern
  list above) is the DynamoDB equivalent of the generic N+1 section's
  query-count budget: state the expected RCU/WCU shape for a new hot access
  pattern in the plan, the same way a relational plan states an index for a
  hot query.
