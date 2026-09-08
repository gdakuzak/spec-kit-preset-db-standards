# Database Standards preset

A [Spec Kit](https://github.com/github/spec-kit) preset that bakes database
conventions into the spec-driven workflow. ORM-agnostic — the rules are about
SQL schema design, not a particular framework.

## What it provides

It **appends** sections to three core templates (your existing spec-kit
templates and other presets keep working):

| Template | Section added | Purpose |
|----------|---------------|---------|
| `spec-template` | Data Requirements | What data the feature needs — entities, reads/writes, retention, consistency |
| `plan-template` | Database Design | Naming conventions, OLTP/OLAP classification, normalization (3NF for OLTP) & denormalization trade-offs, schema changes, SQL portability (ANSI-first, vendor-feature flagging), query safety (SQL injection — bound params, allowlisted identifiers), indexing plan, N+1 / query-volume plan |
| `tasks-template` | Schema Review Checklist | Run before merging any migration |

## Install

```bash
specify preset add --from https://github.com/gdakuzak/spec-kit-preset-db-standards/archive/refs/tags/v0.1.0.zip
```

Local development:

```bash
specify preset add --dev ./spec-kit-preset-db-standards
specify preset resolve plan-template   # verify the section appears
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
