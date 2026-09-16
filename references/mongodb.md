# MongoDB — engine-specific guidance

MongoDB is **NoSQL** (document model). Several `db-standards` plan sections
assume a SQL/relational engine and **do not apply** here — this file replaces
them rather than sharpening them. Use the table below to know what to skip and
what to read instead; the rest of this file is the MongoDB-native design guide.

| `db-standards` section | On MongoDB |
|-------------------------|--------------|
| Normalization (3NF target) | **Replaced.** Embed vs. reference is the per-relationship decision — see Schema design, below. Not a blanket inversion like a key-value store: some relationships still belong split across collections. |
| Schema changes (DDL, backfill) | No fixed schema per document. Use **schema validation** (`$jsonSchema`) to declare and enforce the shape you *do* expect; roll out a shape change with a validation-level strategy, not a migration that locks the collection. |
| Constraints & integrity (`CHECK`, FK) | No native FK. `$jsonSchema` validation covers single-document rules; multi-document invariants need a **multi-document transaction**, below. |
| SQL portability | N/A — there is no SQL. |
| Query safety (SQL injection) | N/A as stated, but the *principle* still applies as **operator injection** — never pass a raw user-supplied object as a query filter; see Operator injection, below. |
| Indexing plan | Replaced by MongoDB index design, below — same goal (one index per real query shape), different mechanism (single/compound/multikey/text/wildcard). |
| N+1 / query volume | Replaced by embedding choices and `$lookup` discipline, below. |
| Data protection (column classification) | Still applies — class every field, same as a column. Consider **Queryable Encryption** / CSFLE for `secret`/`regulated` fields that must stay encrypted even from the server process. |
| Identifiers exposed externally | Still applies, sharpened: a default `ObjectId` encodes its **creation timestamp** and is enumerable in practice — don't expose it as the only identifier where creation time or existence-guessing is a concern; see Identifiers, below. |

## Schema design: embed vs. reference

This is the core per-relationship decision that replaces normalization.

- **Embed** when the child is read together with the parent on the hot path,
  is bounded in count, and doesn't need to be queried or updated independently.
  Embedding is the MongoDB equivalent of a covering index / denormalization
  taken deliberately, not debt — it's the default for one-to-few (an order's
  line items, a user's addresses).
- **Reference** (store the related `_id` and query separately, or `$lookup`)
  when the child is unbounded in count, is queried/updated on its own, or is
  shared by many parents. This is the equivalent of keeping a table split in
  the relational model.
- **Don't let an embedded array grow unbounded.** An ever-growing array
  (comments, log entries, events) on one document is both a hot-document risk
  and, eventually, a document-size problem — reference instead, or bucket
  (below).
- **Bucketing pattern**: for high-volume time-series-like embedded data (IoT
  readings, event logs) that doesn't fit as one unbounded array, group into
  time-windowed "bucket" documents (e.g. one document per sensor per hour)
  instead of one document per reading or one giant array per parent.
- Record the choice per relationship in the plan, the same way the generic
  Normalization section asks OLTP features to record a denormalization:

  | Relationship | Embed or reference | Why |
  |---------------|--------------------|-----|
  | [`order` → `line_items`] | embed | bounded, always read with the order |
  | [`user` → `orders`] | reference | unbounded, queried independently |

## Document validation (replaces DDL)

- Attach a **`$jsonSchema` validator** to every collection that isn't a pure
  scratch/log sink — it's the equivalent of `NOT NULL` / column types /
  `CHECK` from the generic Constraints & integrity section, enforced by the
  server on every write regardless of which app code path wrote it.
- Choose **validation level** deliberately: `strict` (validate all writes,
  including updates to already-invalid legacy documents) vs. `moderate`
  (only validate documents that already pass, letting legacy ones be edited
  without fixing them yet) — state which, and why, for a collection with
  pre-existing data.
- Choose **validation action**: `error` (reject) vs `warn` (log only, write
  still succeeds) — `warn` is a temporary rollout tool, not a steady state for
  anything protecting a real invariant.
- A validator is the single-document equivalent of `CHECK`; it cannot express
  a cross-document rule — that's the transaction section below.

## Identifiers

- Default `_id` is an `ObjectId`: 4-byte timestamp + 5-byte random/machine
  value + 3-byte counter. It is **not cryptographically random** — the
  timestamp is plaintext and the remaining bytes are practically guessable in
  bulk. Follow the generic "Identifiers exposed externally" rule the same way
  as any sequential-ish key: don't expose it where creation-time disclosure or
  enumeration matters; expose a separate UUID v7/ULID or opaque token instead.
- If `_id` itself needs to be something other than the default (a natural
  key, a UUID), set it explicitly on insert — but a client-chosen `_id` loses
  the free time-ordering an `ObjectId`/UUID v7 gives the index, so prefer
  keeping the default unless there's a real reason not to.

## Indexing plan (replaces the generic Indexing plan)

Same discipline as the generic section — one index per real query shape —
different index kinds:

| Query / access pattern | Filter & sort fields | Index |
|------------------------|-----------------------|-------|
| [e.g. "list a user's open orders, newest first"] | `user_id`, `status`, `created_at desc` | compound `{ user_id: 1, status: 1, created_at: -1 }` |

- **Compound index field order**: equality fields first, then sort, same as a
  relational composite index — the **ESR rule** (Equality, Sort, Range).
- **Multikey index** (indexing an array field) works, but a compound index
  with more than one array field is rejected — design around a single
  indexed array per document.
- **Text index** for free-text search on a field; one per collection. For
  real search-product needs (relevance ranking, faceting), that's a search
  engine, not a bigger text index — flag it rather than stretch this one.
- **Partial index** where the query always filters a small subset (mirrors
  the generic section's partial-index guidance) — e.g. index only
  `{ status: "active" }` documents when most are eventually inactive.
- **Wildcard index** (`{ "$**": 1 }`) is a bootstrap/exploration tool for
  genuinely unpredictable field sets, not a substitute for naming the real
  query shapes above.
- Every field used in a `$lookup`'s `foreignField` needs an index on the
  target collection — the `$lookup` equivalent of "every FK used for joins
  has an index."

## Operator injection (replaces SQL injection)

The principle from Query safety carries over even without SQL:

- **Never pass a raw client-supplied object as a query filter or update
  document.** A body like `{"email": {"$ne": null}}` submitted where a plain
  string was expected changes the query's meaning entirely (a classic
  auth-bypass shape: `{"password": {"$gt": ""}}`).
- Build filters/updates from **named, typed fields you extract yourself** —
  never `db.collection.find(req.body)` or spreading user input into a filter.
- If a library/driver allows raw JS (`$where`, `mapReduce` with a JS
  function), treat it exactly like a raw-SQL escape hatch: don't build it
  from input, and prefer the aggregation pipeline's regular operators.
- Same allowlist rule for anything acting as a dynamic "sort field" or
  "group by field" from user input as the generic section's identifier rule.

## Multi-document transactions (replaces multi-row invariants)

- A **single-document** write is already atomic (including nested arrays/
  sub-documents) — many invariants that would need a transaction in a
  relational schema don't need one here *if* the data is embedded together.
  This is a real payoff of the embed choice above, not just a read-time one.
- An invariant spanning **multiple documents or collections** needs a
  **multi-document transaction** (`session.withTransaction`) — the equivalent
  of the generic section's "enforce in the app inside a transaction."
  Transactions have a default 60-second limit and add real overhead; if an
  invariant would need one on a hot path, that's a signal to reconsider the
  embed/reference split rather than reach for a bigger transaction, mirroring
  DynamoDB's guidance not to design around an ever-growing transaction.
- There is no cross-document foreign key. An orphaned reference (a child
  document whose parent was deleted) is prevented by the write path (delete
  children in the same transaction, or an async cleanup keyed off a change
  stream) — never assumed away.

## Read/write concern & consistency

- State the **write concern** (`w: "majority"` for anything that must survive
  a primary failover) and **read concern**/**read preference** per access
  pattern in the plan — this is the MongoDB equivalent of choosing an
  isolation level in the Transactions & concurrency backlog item.
- Reading from a **secondary** trades recency for load distribution — never
  the default for a read-your-own-write path (e.g. render-after-create);
  reserve it for reporting-style reads that tolerate replication lag.

## TTL indexes (replaces retention jobs)

- For data with a defined expiry (the spec's Retention & lifecycle section),
  add a **TTL index** on a date field instead of a scheduled delete job —
  MongoDB's background process removes expired documents automatically
  (within ~60s of expiry, best-effort, not exact — same caveat as DynamoDB
  TTL: don't rely on it for a request-time validity check).

## Change streams (the OLAP / derived-data answer)

- **Change streams** are the mechanism for `db-standards`' OLAP-track
  guidance (derived, rebuildable tables loaded from the source) — a consumer
  watches inserts/updates/deletes and keeps a search index, a materialized
  aggregate, or an OLAP/export copy up to date.
- Treat the consumer the same as any other ETL load in the OLAP section:
  idempotent (safe to reprocess an event, e.g. keyed by `_id` + resume
  token), and the derived store is never treated as authoritative.

## Aggregation pipeline as the OLAP tool

- For in-place analytical queries (rather than an exported OLAP copy), the
  **aggregation pipeline** is the equivalent of a relational analytical
  query — `$match` early (use an index), `$group`/`$bucket` for rollups,
  `$lookup` sparingly (it doesn't use the target's indexes as efficiently as
  a relational join planner and can't be paginated mid-pipeline the way a
  join can).
- `$merge`/`$out` to materialize a pipeline's result into a collection is the
  MongoDB equivalent of a materialized view — state the refresh cadence, same
  as the generic Normalization section asks for a materialized view's
  refresh plan.

## Document size & sharding

- **16 MB document size limit** (hard) — same category of ceiling as
  DynamoDB's item limit. A feature storing large blobs or an unbounded
  embedded array needs a plan before it ships: store the blob in object
  storage and keep a reference, or bound/bucket the array (see Bucketing,
  above).
- If the collection is or will be **sharded**, the **shard key** is the same
  kind of decision as DynamoDB's partition key: high cardinality, even
  access, and — because it's very costly to change later — chosen from the
  feature's actual query/write patterns, not the default `_id`, unless `_id`
  genuinely is that pattern.
