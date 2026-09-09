## Database

<!--
  Added by the db-standards preset. The database engine and version are
  project-wide constraints — downstream db-standards guidance (SQL portability,
  data types, migration mechanics, query safety, RLS) depends on them.

  The constitution command MUST have the user confirm these values before
  drafting; do not infer them silently. Keep this section in the constitution
  even if only partly filled.
-->

- **Engine**: [DATABASE_ENGINE]
- **Version**: [DATABASE_VERSION]
- **Migration tool**: [DATABASE_MIGRATION_TOOL]

Changing the engine after ratification is a MAJOR constitution amendment: it
invalidates portability decisions, type choices, and migration mechanics across
existing plans.

### Data protection

<!--
  Project-wide database security posture. The constitution command MUST confirm
  these with the user. The parenthetical is the recommended stance — change it
  only deliberately.
-->

- **Application connects as**: [DATABASE_APP_ROLE] (recommended: a non-owner role
  with no DDL rights)
- **Reporting / analytics reads**: [DATABASE_READ_SPLIT] (recommended: a separate
  read-only role or replica, not the app's write role)
- **Row-level security**: [DATABASE_RLS_POLICY] (recommended: required on every
  multi-tenant table)
- **Encryption at rest**: [DATABASE_ENCRYPTION_POLICY] (which data classes, and
  how — disk / tablespace / column)
- **Data classification is mandatory**: every column is one of public, internal,
  PII, regulated, or secret. Secrets are never stored in plaintext.
