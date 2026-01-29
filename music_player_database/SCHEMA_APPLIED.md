# PostgreSQL schema (applied)

This database schema was applied to the running PostgreSQL instance by executing each statement **one at a time** using the `psql ...` command stored in `db_connection.txt`.

Connection:
- `db_connection.txt` contains a single line like: `psql postgresql://appuser:...@localhost:5000/myapp`

## No-auth mode alignment (global songs)

Assumptions for the current backend (no-auth mode):
- All endpoints are public (no auth/user context).
- Songs must be **globally visible** (library listing is not filtered by user).
- Therefore, `songs.user_id` **must be nullable** and **must not be required** for inserts.
- Optional: `songs.user_id` may still reference `users(id)` if you keep a `users` table for future use; however, the backend must not depend on it.

> If you are running in a strict no-auth setup and do not want any ownership constraints at all, you may optionally drop the `songs.user_id` foreign key constraint (see “Optional: fully relax ownership constraints” below).

## DDL statements executed (one per call)

1. Enable UUID generation support:
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

2. Users table (kept for compatibility/future use; not required by no-auth song listing):
```sql
CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

3. Index on users.email:
```sql
CREATE INDEX IF NOT EXISTS idx_users_email ON users (email);
```

4. Songs table:

Note (no-auth mode):
- Songs are **global/public**; they are not required to be owned by a user.
- `songs.user_id` is therefore **nullable** (may be NULL).
- If you keep the FK, rows with a non-NULL `user_id` must reference an existing `users(id)`.

```sql
CREATE TABLE IF NOT EXISTS songs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NULL REFERENCES users(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  artist TEXT NOT NULL,
  filename TEXT NOT NULL,
  content_type TEXT NOT NULL,
  size_bytes BIGINT NOT NULL CHECK (size_bytes >= 0),
  duration_seconds INTEGER NULL CHECK (duration_seconds IS NULL OR duration_seconds >= 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

5. Indexes on songs.user_id and songs.created_at:
```sql
CREATE INDEX IF NOT EXISTS idx_songs_user_id ON songs (user_id);
```

```sql
CREATE INDEX IF NOT EXISTS idx_songs_created_at ON songs (created_at);
```

## Migration: align an existing DB to no-auth mode (one statement per call)

If your database was previously created with **ownership enforced**, apply the following as needed.

### A) Make user_id nullable (required for global/no-auth mode)

If `songs.user_id` was `NOT NULL`, run:
```sql
ALTER TABLE songs ALTER COLUMN user_id DROP NOT NULL;
```

### B) Optional: fully relax ownership constraints (drop FK)

If you want `songs.user_id` to be a purely informational field (or you might insert arbitrary UUIDs without creating users), drop the foreign key constraint.

1) Find the FK constraint name (inspect via psql; this is a read-only query):
```sql
SELECT conname
FROM pg_constraint
WHERE conrelid = 'songs'::regclass
  AND contype = 'f';
```

2) Drop the FK using the name you found (example name shown; replace it):
```sql
ALTER TABLE songs DROP CONSTRAINT songs_user_id_fkey;
```

> Note: PostgreSQL auto-names FKs commonly as `songs_user_id_fkey`, but do not assume—confirm via the query above.

## Re-applying manually (example)

From `music_player_database/`:

```bash
CONN=$(head -n 1 db_connection.txt | tr -d '\r\n')
$CONN -v ON_ERROR_STOP=1 -c "CREATE EXTENSION IF NOT EXISTS pgcrypto"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS users (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), email TEXT NOT NULL UNIQUE, password_hash TEXT NOT NULL, created_at TIMESTAMPTZ NOT NULL DEFAULT now())"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_users_email ON users (email)"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS songs (id UUID PRIMARY KEY DEFAULT gen_random_uuid(), user_id UUID NULL REFERENCES users(id) ON DELETE CASCADE, title TEXT NOT NULL, artist TEXT NOT NULL, filename TEXT NOT NULL, content_type TEXT NOT NULL, size_bytes BIGINT NOT NULL CHECK (size_bytes >= 0), duration_seconds INTEGER NULL CHECK (duration_seconds IS NULL OR duration_seconds >= 0), created_at TIMESTAMPTZ NOT NULL DEFAULT now())"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_songs_user_id ON songs (user_id)"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS idx_songs_created_at ON songs (created_at)"

# If migrating an existing DB to no-auth mode:
$CONN -v ON_ERROR_STOP=1 -c "ALTER TABLE songs ALTER COLUMN user_id DROP NOT NULL"
```
