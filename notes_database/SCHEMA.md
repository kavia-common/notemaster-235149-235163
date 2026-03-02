# NoteMaster database schema (PostgreSQL)

This container uses PostgreSQL, with connection info stored in `db_connection.txt`.

## Overview

The app schema includes:

- `notes`: main notes table, with `is_pinned` and `is_favorite` flags
- `tags`: reusable tags (unique by normalized/lowercased name)
- `note_tags`: many-to-many join table between notes and tags (with safe cascade deletes)

The schema is designed for:
- safe deletes (FKs with `ON DELETE CASCADE` from join table)
- basic search performance (GIN index on `to_tsvector(title, content)`)
- basic tag filtering performance (indexes on join table)

## Tables

### `notes`

- `id` UUID PK (`gen_random_uuid()`)
- `title` text (<= 200 chars)
- `content` text
- `is_pinned` boolean
- `is_favorite` boolean
- `created_at`, `updated_at` timestamps
- trigger automatically updates `updated_at` on update

### `tags`

- `id` UUID PK (`gen_random_uuid()`)
- `name` text
- `normalized_name` generated stored column = `lower(trim(name))`
- `color` text (hex, optional)
- unique constraint on `normalized_name`

### `note_tags`

- composite primary key (`note_id`, `tag_id`)
- `note_id` FK -> `notes(id)` ON DELETE CASCADE
- `tag_id` FK -> `tags(id)` ON DELETE CASCADE

This ensures:
- deleting a note deletes its note_tags rows automatically
- deleting a tag deletes its note_tags rows automatically

## Indexes

- `notes_created_at_idx` on `notes(created_at DESC)` for newest-first lists
- `notes_pinned_created_at_idx` on `notes(is_pinned DESC, created_at DESC)` for pinned-first sorting
- `notes_favorite_created_at_idx` on `notes(is_favorite DESC, created_at DESC)` for favorites-first sorting
- `notes_search_idx` GIN index on `to_tsvector('simple', title || content)` for basic full-text search
- `tags_normalized_name_idx` on `tags(normalized_name)` for tag lookup/autocomplete
- `note_tags_tag_id_note_id_idx` on `note_tags(tag_id, note_id)` for tag->notes filtering

## Applying / Re-applying (one statement at a time)

All commands below use the connection in `db_connection.txt`:

```bash
cd notemaster-235149-235163/notes_database
CONN=$(cat db_connection.txt)
```

### Extensions

```bash
$CONN -v ON_ERROR_STOP=1 -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
```

### DDL

```bash
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS tags (id uuid PRIMARY KEY DEFAULT gen_random_uuid(), name text NOT NULL, normalized_name text GENERATED ALWAYS AS (lower(trim(name))) STORED, color text, created_at timestamptz NOT NULL DEFAULT now(), CONSTRAINT tags_normalized_name_uniq UNIQUE (normalized_name), CONSTRAINT tags_name_len CHECK (char_length(name) BETWEEN 1 AND 50));"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS notes (id uuid PRIMARY KEY DEFAULT gen_random_uuid(), title text NOT NULL DEFAULT '', content text NOT NULL DEFAULT '', is_pinned boolean NOT NULL DEFAULT false, is_favorite boolean NOT NULL DEFAULT false, created_at timestamptz NOT NULL DEFAULT now(), updated_at timestamptz NOT NULL DEFAULT now(), CONSTRAINT notes_title_len CHECK (char_length(title) <= 200));"
$CONN -v ON_ERROR_STOP=1 -c "CREATE TABLE IF NOT EXISTS note_tags (note_id uuid NOT NULL REFERENCES notes(id) ON DELETE CASCADE, tag_id uuid NOT NULL REFERENCES tags(id) ON DELETE CASCADE, created_at timestamptz NOT NULL DEFAULT now(), PRIMARY KEY (note_id, tag_id));"
```

### Indexes

```bash
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS notes_created_at_idx ON notes(created_at DESC);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS notes_pinned_created_at_idx ON notes(is_pinned DESC, created_at DESC);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS notes_favorite_created_at_idx ON notes(is_favorite DESC, created_at DESC);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS note_tags_tag_id_note_id_idx ON note_tags(tag_id, note_id);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS tags_normalized_name_idx ON tags(normalized_name);"
$CONN -v ON_ERROR_STOP=1 -c "CREATE INDEX IF NOT EXISTS notes_search_idx ON notes USING GIN (to_tsvector('simple', coalesce(title,'') || ' ' || coalesce(content,'')));"
```

### updated_at trigger

NOTE: in shell, `$$` expands to PID, so we must escape it as `\$\$`.

```bash
$CONN -v ON_ERROR_STOP=1 -c "CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger LANGUAGE plpgsql AS \$\$ BEGIN NEW.updated_at = now(); RETURN NEW; END; \$\$;"
$CONN -v ON_ERROR_STOP=1 -c "DO \$\$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_trigger WHERE tgname = 'notes_set_updated_at_trg') THEN CREATE TRIGGER notes_set_updated_at_trg BEFORE UPDATE ON notes FOR EACH ROW EXECUTE FUNCTION set_updated_at(); END IF; END \$\$;"
```

## Seed data

Tags:

```bash
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO tags (name, color) VALUES ('work', '#3b82f6') ON CONFLICT (normalized_name) DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO tags (name, color) VALUES ('personal', '#06b6d4') ON CONFLICT (normalized_name) DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO tags (name, color) VALUES ('ideas', '#a855f7') ON CONFLICT (normalized_name) DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO tags (name, color) VALUES ('recipes', '#f59e0b') ON CONFLICT (normalized_name) DO NOTHING;"
```

Notes (simple inserts; may create duplicates if re-run):

```bash
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO notes (title, content, is_pinned, is_favorite) VALUES ('Welcome to NoteMaster', 'This is your first note. Use tags, pin notes, and mark favorites.\n\nTip: try searching for keywords like \"retro\".', true, true);"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO notes (title, content, is_pinned, is_favorite) VALUES ('Retro UI ideas', 'Neon accents, pixel borders, and chunky buttons.\n\nChecklist:\n- Sidebar tags\n- Search bar\n- Note cards', false, false);"
```

Join rows (idempotent because of composite PK + ON CONFLICT DO NOTHING):

```bash
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO note_tags (note_id, tag_id) SELECT n.id, t.id FROM notes n JOIN tags t ON t.normalized_name='work' WHERE n.title='Welcome to NoteMaster' ON CONFLICT DO NOTHING;"
$CONN -v ON_ERROR_STOP=1 -c "INSERT INTO note_tags (note_id, tag_id) SELECT n.id, t.id FROM notes n JOIN tags t ON t.normalized_name='ideas' WHERE n.title='Retro UI ideas' ON CONFLICT DO NOTHING;"
```
