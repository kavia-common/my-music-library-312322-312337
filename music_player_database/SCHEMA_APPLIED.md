# PostgreSQL schema (applied)

This database schema was applied to the running PostgreSQL instance by executing each statement **one at a time** using the `psql ...` command stored in `db_connection.txt`.

Connection:
- `db_connection.txt` contains a single line like: `psql postgresql://appuser:...@localhost:5000/myapp`

## DDL statements executed (one per call)

1. Enable UUID generation support:
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

2. Users table:
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
- `songs.user_id` is therefore **nullable** (may be NULL). If present, it can still reference `users(id)`.

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
CREATE INDEX IF NOT EXISTS idx_songs_created_at ON songs (created_at);
```

## No-auth migration (applied)

If your database was previously created with `songs.user_id NOT NULL`, migrate it to no-auth mode by executing this **single statement** (one `-c` call):

```sql
ALTER TABLE songs ALTER COLUMN user_id DROP NOT NULL;
```

## Re-applying manually (example)

From `music_player_database/`:

```bash
CONN=$(head -n 1 db_connection.txt | tr -d '\r\n')
$CONN -v ON_ERROR_STOP=1 -c "CREATE EXTENSION IF NOT EXISTS pgcrypto"
# ...repeat with one -c per SQL statement...
# if migrating an existing DB to no-auth mode:
$CONN -v ON_ERROR_STOP=1 -c "ALTER TABLE songs ALTER COLUMN user_id DROP NOT NULL"
```
