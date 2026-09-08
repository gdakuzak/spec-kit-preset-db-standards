## Data Requirements

<!--
  Added by the db-standards preset. Fill this in during /speckit.specify.
  Keep it about WHAT data the feature needs, not HOW it is stored — schema
  design belongs in the plan.
-->

### Entities & relationships

| Entity | Purpose | Owns / references |
|--------|---------|-------------------|
| [Entity] | [why it exists] | [related entities] |

### Data this feature reads

- [source, shape, expected volume]

### Data this feature writes or changes

- [what is created / updated / deleted, and by which action]

### Retention & lifecycle

- [How long is this data kept? Soft-delete or hard-delete? Any archival?]
- [Any PII / regulated data? If yes, name it and the handling rule.]

### External input reaching the database

- [Which fields of this feature's input are used to filter, sort, or search data? List them — the plan must show each one reaching SQL as a bound parameter or an allowlisted identifier.]

### Consistency & concurrency

- [Does anything need a transaction spanning multiple writes?]
- [Any operation that two users could race on? (e.g. claiming a slot)]

### Non-functional expectations

- Expected row counts at 1x and 12-month growth: [...]
- Read/write ratio and latency budget for the hot path: [...]
