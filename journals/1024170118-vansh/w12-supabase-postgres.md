# Week 12 : Moving the Database to Supabase PostgreSQL

## Problem

Until now the backend used a local SQLite file. A deployed backend needs a
database that survives container restarts and can be reached from Render.
Supabase provides a hosted PostgreSQL database (and file storage) on a free
plan.

## Relevant Context

Supabase is reached through its connection pooler. In **session mode** it
allows 15 client sessions in total, shared by every process that connects.
The pooler also closes connections that stay idle for a while.

## Key Observation

Three things change when moving from a local file to a shared, hosted
database:

1. the number of connections is a shared, limited resource;
2. a pooled connection may already be closed when the app tries to use it;
3. creating or altering tables on startup now changes a shared production
   database, so it must not happen automatically.

## Solution

1. **A small connection pool** for PostgreSQL URLs (SQLite gets no pool
   options):

    ``` python
    def _pool_options_for_url(database_url):
        if not database_url.startswith("postgresql"):
            return {}
        return {"pool_size": 2, "max_overflow": 0, "pool_timeout": 15}
    ```

    (Raised to 5 + 3 in Week 17 after a starvation bug; see Aviraj's journal.)

2. **`pool_pre_ping=True`**, so SQLAlchemy tests each connection with a cheap
   query before handing it out and replaces it if the pooler has closed it.

3. **No automatic DDL on hosted Supabase.** Schema creation runs only when
   `AUTO_MIGRATE` is set, and even then it refuses a Supabase host:

    ``` python
    def require_generic_schema_target_allowed(target_engine):
        """Hosted Supabase DDL must use separately reviewed SQL, never initialization."""
        ...
        if hostname.endswith((".supabase.co", ".supabase.com")):
            raise RuntimeError("Generic schema initialization is disabled for hosted Supabase.")
    ```

    Schema changes for Supabase are written as reviewed SQL files under
    `backend/supabase/migrations/` and applied by hand.

**Because**

`create_all` and ad-hoc `ALTER TABLE` calls are convenient on a laptop, but
on a shared database a mistake cannot be undone by deleting a file. Making
the guard check the host name means the protection does not depend on
someone remembering to set an environment variable.
