# Database Standards preset

A [Spec Kit](https://github.com/github/spec-kit) preset that bakes database
conventions into the spec-driven workflow. ORM-agnostic — the rules are about
SQL schema design, not a particular framework.

## What it provides

It **appends** to core templates and one command (your existing spec-kit
templates and other presets keep working):

| Target | What is added | Purpose |
|--------|---------------|---------|
| `speckit.constitution` (command) | Database + Data protection confirmation step | Forces confirming the target DB (engine, version, migration tool) and the security posture (roles, RLS) before the constitution is drafted |
| `constitution-template` | Database + Data protection sections | Records engine / version / migration tool and the project-wide security posture |
| `spec-template` | Data Requirements | What data the feature needs — entities, reads/writes, retention, consistency |
| `plan-template` | Database Design | Naming conventions, OLTP/OLAP classification, normalization (3NF for OLTP) & denormalization trade-offs, schema changes, constraints & integrity (DB-enforced invariants, NOT NULL, FK actions, CHECK), data protection (column classification, secrets, RLS, least-privilege), external identifier exposure, SQL portability (ANSI-first, vendor-feature flagging), query safety (SQL injection — bound params, allowlisted identifiers), indexing plan, N+1 / query-volume plan |
| `tasks-template` | Schema Review Checklist | Run before merging any migration |

## Install

```bash
specify preset add --from https://github.com/gdakuzak/spec-kit-preset-db-standards/archive/refs/heads/main.zip
```

Update an already-installed copy — `add` only appends, so remove first (one line):

```bash
specify preset remove db-standards && specify preset add --from https://github.com/gdakuzak/spec-kit-preset-db-standards/archive/refs/heads/main.zip
```

Local development:

```bash
specify preset add --dev ./spec-kit-preset-db-standards
specify preset resolve constitution-template   # verify the section appears
specify preset remove db-standards     # when done
```

## When to use it

- Any project with a relational database where schema quality matters.
- Teams that want naming, indexing, and N+1 checks to be part of planning and review rather than caught in production.

## When not to use it

- No database, or a schemaless store where these conventions don't map.
- You already have a house preset covering the same templates with `replace` — stacking `append` on top still works, but check the combined output.

## License

MIT
