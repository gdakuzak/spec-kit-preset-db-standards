## Database Confirmation (db-standards preset)

Before drafting or updating the constitution, you **MUST** explicitly ask the
user to confirm the project's database. Do not infer it silently from the repo —
ask, even when a value seems obvious, and echo it back for confirmation.

Confirm:

- **Engine** — e.g. PostgreSQL, MySQL / MariaDB, SQL Server, Oracle, SQLite,
  CockroachDB, or another.
- **Major version** — e.g. PostgreSQL 16.
- **Migration tool / mechanism** — e.g. Flyway, Liquibase, Alembic, Ecto
  migrations, Prisma Migrate, Rails, raw SQL.

If the user genuinely hasn't decided yet, write `TODO(DATABASE_ENGINE)` and list
it in the Sync Impact Report as a deferred item.

Record the confirmed answers in the constitution's **Database** section
(placeholders `[DATABASE_ENGINE]`, `[DATABASE_VERSION]`,
`[DATABASE_MIGRATION_TOOL]`). Every later db-standards step — SQL portability,
data types, migration mechanics, query safety, row-level security — is guided by
this value, so it must be settled in the constitution first.
