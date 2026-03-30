# Supabase Database Reference

This document describes the schema, RLS policies, storage buckets, and access patterns for the physical flashcard manager Supabase project.

## Authentication

All tables use Supabase Auth (`auth.uid()`). Every request must include a valid JWT (obtained via Supabase Auth). Row-level security is enforced on every table — unauthenticated reads/writes are rejected.

Environment variables required by clients:

| Variable | Description |
|---|---|
| `SUPABASE_URL` | Project URL |
| `SUPABASE_ANON_KEY` | Public anon key (used with user JWT) |

---

## Tables

### `public.collections`

A flashcard collection. Defines visual styling applied to all cards in the collection.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `bigint` | NO | identity | Primary key |
| `user_id` | `uuid` | NO | `auth.uid()` | Owner (references `auth.users.id`, cascades on delete) |
| `title` | `text` | NO | — | Display name |
| `description` | `text` | YES | — | Optional description |
| `width_mm` | `numeric(10,2)` | NO | `148.50` | Card width in mm (default: A6 landscape) |
| `height_mm` | `numeric(10,2)` | NO | `105.00` | Card height in mm (default: A6 landscape) |
| `font_family` | `text` | NO | `'Arial'` | Font used for card body text |
| `font_size` | `text` | YES | — | Font size override (e.g. `'14px'`) |
| `header_color` | `text` | NO | `'#100e75'` | Background color of the card header |
| `background_color` | `text` | NO | `'#f0f0f0'` | Background color of the card body |
| `font_color` | `text` | NO | `'#171717'` | Body text color |
| `header_font_color` | `text` | NO | `'#ffffff'` | Header text color |
| `header_text_left` | `text` | YES | — | Static text shown on the left side of the header |
| `created_at` | `timestamptz` | NO | `now()` UTC | |
| `updated_at` | `timestamptz` | NO | `now()` UTC | Auto-updated by trigger on every UPDATE |

**Indexes:** `collections_user_id_idx` on `(user_id)`

---

### `public.flashcards`

Individual flashcards belonging to a collection.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `bigint` | NO | identity | Primary key |
| `user_id` | `uuid` | NO | `auth.uid()` | Owner (references `auth.users.id`, cascades on delete) |
| `collection_id` | `bigint` | NO | — | FK → `collections.id` (cascades on delete) |
| `front` | `text` | NO | — | Front face content |
| `back` | `text` | NO | — | Back face content |
| `header_right` | `text` | YES | — | Text shown on the right side of the card header |
| `is_info_card` | `boolean` | NO | `false` | If true, card is displayed as an info/title card |
| `is_favorite` | `boolean` | NO | `false` | User-marked favorite |
| `sort_order` | `integer` | NO | — | Display order within the collection |
| `created_at` | `timestamptz` | NO | `now()` UTC | |
| `updated_at` | `timestamptz` | NO | `now()` UTC | Auto-updated by trigger on every UPDATE |

**Indexes:** `flashcards_user_id_idx` on `(user_id)`, `flashcards_collection_id_idx` on `(collection_id)`

**Constraint:** A trigger (`flashcards_ensure_owned_collection_reference`) verifies that `collection_id` points to a collection owned by `user_id`. Inserting or updating a flashcard with a collection that belongs to a different user raises an exception.

---

### `public.materials`

PDF source files uploaded by users, associated with a collection.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `uuid` | NO | `gen_random_uuid()` | Primary key |
| `user_id` | `uuid` | NO | `auth.uid()` | Owner (references `auth.users.id`, cascades on delete) |
| `collection_id` | `bigint` | NO | — | FK → `collections.id` (cascades on delete) |
| `internal_name` | `text` | NO | — | Human-readable label used internally |
| `original_filename` | `text` | NO | — | Filename as uploaded by the user |
| `storage_path` | `text` | NO | — | Path in the `materials` storage bucket (unique) |
| `created_at` | `timestamptz` | NO | `now()` UTC | |
| `updated_at` | `timestamptz` | NO | `now()` UTC | Auto-updated by trigger on every UPDATE |

**Indexes:** `materials_user_id_idx` on `(user_id)`, `materials_collection_id_idx` on `(collection_id)`

**Constraint:** Same owned-collection trigger as flashcards (`materials_ensure_owned_collection_reference`).

---

## Row-Level Security

RLS is enabled on all three tables. The policy is uniform: **users can only select/insert/update/delete their own rows** (`auth.uid() = user_id`).

| Table | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| `collections` | own rows | authenticated, own | authenticated, own | authenticated, own |
| `flashcards` | authenticated, own | authenticated, own | authenticated, own | authenticated, own |
| `materials` | authenticated, own | authenticated, own | authenticated, own | authenticated, own |

> Note: `collections` SELECT does not require the `authenticated` role explicitly (the policy uses only `auth.uid() = user_id`), but in practice unauthenticated users have no `uid` and will see no rows.

---

## Storage Buckets

### `materials` (private)

Stores PDF files uploaded by users.

| Property | Value |
|---|---|
| Public | No |
| Max file size | 15 MB (15,728,640 bytes) |
| Allowed MIME types | `application/pdf` |
| Path convention | `{user_id}/{filename}` |

**Policies:** Authenticated users can SELECT, INSERT, and DELETE objects where the first path segment equals their `auth.uid()`.

### `images` (public)

Stores images embedded in flashcard content.

| Property | Value |
|---|---|
| Public | Yes (CDN, no auth required for reads) |
| Max file size | 10 MB (10,485,760 bytes) |
| Allowed MIME types | `image/jpeg`, `image/png`, `image/gif`, `image/webp` |
| Path convention | `{user_id}/{filename}` |

**Policies:** Authenticated users can INSERT and DELETE objects where the first path segment equals their `auth.uid()`. No SELECT policy is needed because the bucket is public — objects are served via CDN URL.

---

## Triggers & Functions

| Name | Fires on | Purpose |
|---|---|---|
| `set_updated_at()` | BEFORE UPDATE on all three tables | Sets `updated_at` to current UTC time |
| `ensure_owned_collection_reference()` | BEFORE INSERT OR UPDATE on `flashcards`, `materials` | Prevents cross-user collection references |

---

## Connecting from a New Frontend

1. Initialize the Supabase client with `SUPABASE_URL` and `SUPABASE_ANON_KEY`.
2. Sign in via Supabase Auth (email/password, OAuth, magic link, etc.) to obtain a session JWT.
3. All queries are automatically scoped to the authenticated user by RLS — no manual `where user_id = ...` needed.
4. For the `materials` bucket, upload files to `{user_id}/{filename}`. For the `images` bucket, upload to `{user_id}/{filename}` and use the public CDN URL for display.

Example (TypeScript / `@supabase/supabase-js`):

```ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// Fetch all collections for the logged-in user
const { data, error } = await supabase
  .from('collections')
  .select('*')
  .order('created_at', { ascending: false });
```
