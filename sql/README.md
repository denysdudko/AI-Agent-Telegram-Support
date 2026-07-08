# SQL

## Purpose

The `sql` directory stores database scripts for PostgreSQL, including the current schema reference, migrations, views, and development seed data. It keeps database changes reviewable and reproducible through Git.

## Directory Layout

- `schema/` stores the latest database structure as reference SQL.
- `migrations/` stores ordered database changes that move an environment from one schema state to the next.
- `views/` stores SQL definitions for database views.
- `seed/` stores development and demo data scripts.

## Maintenance Rules

- Every database change must be implemented as a migration.
- The latest database structure must always be reflected in `schema/`.
- Migration files are immutable after they have been committed.
- Seed scripts must contain only development or demo data and must never contain production data.
- SQL scripts must be idempotent whenever practical.

## Versioning

Git history is the primary record for database script changes. Use commits and diffs to review how schema references, migrations, views, and seed data evolve over time.
