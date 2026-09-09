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
