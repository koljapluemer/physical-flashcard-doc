# Supabase Database Reference

This document describes the schema, RLS policies, storage buckets, and — critically — **how to interpret and render the data** so other frontends can display flashcards correctly.

## Authentication

All tables use Supabase Auth (`auth.uid()`). Every request must include a valid JWT. Row-level security is enforced — unauthenticated reads/writes are rejected.

| Env var | Description |
|---|---|
| `SUPABASE_URL` | Project URL |
| `SUPABASE_ANON_KEY` | Public anon key (used with user JWT) |

---

## Tables

### `public.collections`

Defines a flashcard collection and the visual styling applied to every card in it.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `bigint` | NO | identity | Primary key |
| `user_id` | `uuid` | NO | `auth.uid()` | Owner — FK → `auth.users.id`, cascade delete |
| `title` | `text` | NO | — | Display name |
| `description` | `text` | YES | — | Optional description |
| `width_mm` | `numeric(10,2)` | NO | `148.50` | Card width in mm |
| `height_mm` | `numeric(10,2)` | NO | `105.00` | Card height in mm |
| `font_family` | `text` | NO | `'Arial'` | Font used for card body text |
| `font_size` | `text` | YES | — | Base font size override (e.g. `'14'` meaning 14 px) |
| `header_color` | `text` | NO | `'#100e75'` | Header background color (hex) |
| `background_color` | `text` | NO | `'#f0f0f0'` | Card body background (hex) |
| `font_color` | `text` | NO | `'#171717'` | Body text color (hex) |
| `header_font_color` | `text` | NO | `'#ffffff'` | Header text color (hex) |
| `header_text_left` | `text` | YES | — | Static text on the left of every card's header |
| `created_at` | `timestamptz` | NO | `now()` UTC | |
| `updated_at` | `timestamptz` | NO | `now()` UTC | Auto-updated on every UPDATE |

---

### `public.flashcards`

Individual flashcards. The `front` and `back` fields are **JSON-encoded `CardSideData` objects** — see [Card Content Format](#card-content-format) below.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `bigint` | NO | identity | Primary key |
| `user_id` | `uuid` | NO | `auth.uid()` | Owner — FK → `auth.users.id`, cascade delete |
| `collection_id` | `bigint` | NO | — | FK → `collections.id`, cascade delete |
| `front` | `text` | NO | — | Front face — JSON-encoded `CardSideData` |
| `back` | `text` | NO | — | Back face — JSON-encoded `CardSideData` |
| `header_right` | `text` | YES | — | Text on the right of the card header (front side only) |
| `is_info_card` | `boolean` | NO | `false` | Marks the card as an info/title card (metadata flag) |
| `is_favorite` | `boolean` | NO | `false` | User-marked favorite |
| `sort_order` | `integer` | NO | — | Display order within the collection |
| `created_at` | `timestamptz` | NO | `now()` UTC | |
| `updated_at` | `timestamptz` | NO | `now()` UTC | Auto-updated on every UPDATE |

**Constraint:** A DB trigger (`flashcards_ensure_owned_collection_reference`) prevents inserting/updating a flashcard with a `collection_id` that belongs to a different user.

---

### `public.materials`

PDF source files associated with a collection, stored in the `materials` storage bucket.

| Column | Type | Nullable | Default | Description |
|---|---|---|---|---|
| `id` | `uuid` | NO | `gen_random_uuid()` | Primary key |
| `user_id` | `uuid` | NO | `auth.uid()` | Owner — FK → `auth.users.id`, cascade delete |
| `collection_id` | `bigint` | NO | — | FK → `collections.id`, cascade delete |
| `internal_name` | `text` | NO | — | Human-readable label |
| `original_filename` | `text` | NO | — | Filename as uploaded |
| `storage_path` | `text` | NO | — | Path in the `materials` bucket (unique) |
| `created_at` | `timestamptz` | NO | `now()` UTC | |
| `updated_at` | `timestamptz` | NO | `now()` UTC | Auto-updated on every UPDATE |

---

## Row-Level Security

RLS is enabled on all three tables. Users can only access their own rows.

| Table | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| `collections` | own rows | own rows | own rows | own rows |
| `flashcards` | own rows | own rows | own rows | own rows |
| `materials` | own rows | own rows | own rows | own rows |

---

## Storage Buckets

### `materials` (private)

| Property | Value |
|---|---|
| Public | No |
| Max file size | 15 MB |
| Allowed types | `application/pdf` |
| Path convention | `{user_id}/{filename}` |

Authenticated users can SELECT/INSERT/DELETE objects where the first path segment matches their `auth.uid()`.

### `images` (public)

| Property | Value |
|---|---|
| Public | Yes (CDN, no auth needed to read) |
| Max file size | 10 MB |
| Allowed types | `image/jpeg`, `image/png`, `image/gif`, `image/webp` |
| Path convention | `{user_id}/{filename}` |

Authenticated users can INSERT/DELETE where the first path segment matches their `auth.uid()`. No SELECT policy — the bucket is public and images are served via CDN URL directly.

---

## Card Content Format

This is the most important section for any frontend that wants to display flashcards.

### `CardSideData` — the JSON structure in `front` / `back`

Every `front` and `back` field is a JSON string with this shape:

```json
{
  "layout": "2-columns",
  "sections": {
    "left": "Markdown content for the left column",
    "right": "Markdown content for the right column"
  }
}
```

If the stored value is **plain text** (not valid JSON), treat it as:

```json
{ "layout": "default", "sections": { "main": "<the plain text>" } }
```

### Layouts

Five layout types are supported. Each layout has a fixed set of section keys:

| `layout` value | Section keys | Visual structure |
|---|---|---|
| `default` | `main` | Single full-width column |
| `2-columns` | `left`, `right` | Two equal columns side by side |
| `3-columns` | `left`, `center`, `right` | Three equal columns |
| `top-row-2-columns` | `top`, `left`, `right` | Full-width row on top, two columns below |
| `bottom-row-2-columns` | `left`, `right`, `bottom` | Two columns on top, full-width row below |

Each section value is a **markdown string** — render it with the markdown pipeline described below.

### Card Header

Every card has a header bar rendered above the content:

```
┌─────────────────────────────────────────┐
│  header_text_left        header_right   │  ← header (collection bg color)
├─────────────────────────────────────────┤
│  [card content sections]                │  ← body (collection bg color)
└─────────────────────────────────────────┘
```

- **Left text:** `collection.header_text_left` — same for every card in the collection
- **Right text:** `flashcard.header_right` — per-card; **only shown on the front side**, not the back
- **Background color:** `collection.header_color`
- **Text color:** `collection.header_font_color`
- Hide the header bar entirely if both left and right text are empty

### `is_info_card`

A metadata flag — `true` means the card is a title/info card rather than a learn card. The rendering is identical. Use it for filtering (e.g. skip info cards in a study session, or display them differently in a deck overview).

---

## Markdown Rendering

Each section's string content is Markdown. The following features must be supported to render cards correctly.

### Standard Markdown (GFM)

Full GitHub Flavored Markdown: bold, italic, headings, bullet/numbered lists, tables, code spans and blocks, blockquotes, links.

### `:::box` — Highlighted box

A custom container directive for callout/highlight boxes:

```
::: box
Content inside the box
:::
```

Renders as a rounded container styled with the collection's `header_color`:

- Background: `header_color` at 8% opacity
- Border: `header_color` at 16% opacity
- Padding + border-radius

Example HTML output:

```html
<aside class="flashcard-box">
  <p>Content inside the box</p>
</aside>
```

### Math — KaTeX

Inline and display math using `$` delimiters (LaTeX):

```
Inline: $E = mc^2$
Display: $$\int_0^1 x\, dx$$
```

Rendered with KaTeX. `\(...\)` and `\[...\]` are **not** supported — use `$` and `$$` only.

### Images with size hints (Obsidian-style)

Standard Markdown images with an optional size suffix in the alt text:

```
![alt](url)           → natural size
![alt|300](url)       → 300 px wide
![alt|x150](url)      → 150 px tall
![alt|300x150](url)   → 300 × 150 px
```

The `|WxH` part is stripped from the rendered alt text and applied as inline CSS (`width`, `height`).

---

## Applying Collection Styles

The collection object provides all visual parameters. Apply them to the card container:

| Collection field | CSS property / usage |
|---|---|
| `background_color` | Card body background |
| `font_color` | Body text color |
| `font_family` | Font face (Google Fonts or system font) |
| `font_size` | Base font size in px |
| `header_color` | Header background; also base color for `:::box` tinting |
| `header_font_color` | Header text color |
| `width_mm` / `height_mm` | Physical card dimensions (for print/PDF) |

---

## Quick-start Example (TypeScript)

```ts
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// 1. Fetch a collection and its cards
const { data: collection } = await supabase
  .from('collections')
  .select('*')
  .eq('id', collectionId)
  .single();

const { data: flashcards } = await supabase
  .from('flashcards')
  .select('*')
  .eq('collection_id', collectionId)
  .order('sort_order');

// 2. Parse the card content
function parseCardSide(raw: string) {
  try {
    return JSON.parse(raw); // { layout, sections }
  } catch {
    return { layout: 'default', sections: { main: raw } };
  }
}

for (const card of flashcards) {
  const front = parseCardSide(card.front);
  const back  = parseCardSide(card.back);
  // front.layout -> e.g. '2-columns'
  // front.sections -> e.g. { left: '...markdown...', right: '...markdown...' }
}
```
