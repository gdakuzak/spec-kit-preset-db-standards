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

Also confirm the **Data protection** posture with the user (placeholders
`[DATABASE_APP_ROLE]`, `[DATABASE_READ_SPLIT]`, `[DATABASE_RLS_POLICY]`): how the
app connects, whether reporting reads are split off, and whether row-level
security is required. Offer the recommended stance shown in the template; record
what the user decides. Encryption at rest is a recommendation only — mention it
if the project's compliance context calls for it, but don't require an answer.

Record the confirmed answers in the constitution's **Database** and **Data
protection** sections. Every later db-standards step — SQL portability, data
types, migration mechanics, query safety, row-level security — is guided by these
values, so they must be settled in the constitution first.
