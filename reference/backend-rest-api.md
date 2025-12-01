# Flashcard Manager REST API

The Flashcard Manager project now exposes a small REST API for managing flashcard collections. All endpoints live under `/api/` and require token authentication.

## Authentication

Obtain a token by posting email/password credentials to:

- `POST /api/token-auth/`

Request body:

```json
{
  "email": "user@example.com",
  "password": "your-password"
}
```

Successful responses return:

```json
{
  "token": "abcdef123456..."
}
```

Include the token in the `Authorization` header for every subsequent request:

```
Authorization: Token abcdef123456...
```

Tokens are persistent and tied to the user; rotate them by deleting/recreating via the Django admin if needed.

## Endpoints

### Collections

`/api/collections/`

| Method | Description                 | Notes                                    |
|--------|-----------------------------|------------------------------------------|
| GET    | List collections            | Returns an array of collection objects   |
| POST   | Create a collection         | `title` (required), `description` (optional), styling fields (optional) |

`/api/collections/{id}/`

| Method | Description                 | Notes                                    |
|--------|-----------------------------|------------------------------------------|
| GET    | Retrieve one collection     |                                          |
| PATCH  | Partial update              | Supply any subset of writable fields     |
| PUT    | Replace the collection      | Provide all writable fields              |
| DELETE | Delete the collection       | Also deletes its flashcards              |

Collection representation:

```json
{
  "id": 1,
  "title": "Biology 101",
  "description": "Introductory course",
  "header_color": "#100e75",
  "background_color": "#f0f0f0",
  "font_color": "#171717",
  "header_font_color": "#fff",
  "header_text_left": "Chapter 1",
  "created_at": "2025-01-01T12:00:00Z",
  "updated_at": "2025-01-02T09:30:00Z"
}
```

**Styling Fields:**
- `header_color` - Hex color for collection header background (default: `#100e75`)
- `background_color` - Hex color for collection card background (default: `#f0f0f0`)
- `font_color` - Hex color for general text (default: `#171717`)
- `header_font_color` - Hex color for header text (default: `#fff`)
- `header_text_left` - Text displayed on left side of header (default: empty string, optional)

### Flashcards

`/api/flashcards/`

| Method | Description                 | Notes                                                                          |
|--------|-----------------------------|--------------------------------------------------------------------------------|
| GET    | List flashcards             | Optional query param `collection=<id>` filters by collection                   |
| POST   | Create a flashcard          | Requires `collection`, `front`, `back`; `header_right` is optional            |

`/api/flashcards/{id}/`

| Method | Description                 | Notes                                    |
|--------|-----------------------------|------------------------------------------|
| GET    | Retrieve one flashcard      |                                          |
| PATCH  | Partial update              | Update any subset of writable fields     |
| PUT    | Replace the flashcard       | Provide all writable fields              |
| DELETE | Delete the flashcard        |                                          |

Flashcard representation:

```json
{
  "id": 10,
  "collection": 1,
  "front": "What is photosynthesis?",
  "back": "The process plants use to convert light energy into chemical energy.",
  "header_right": "Page 42",
  "created_at": "2025-01-02T13:00:00Z",
  "updated_at": "2025-01-02T13:05:00Z"
}
```

**Optional Fields:**
- `header_right` - Text displayed on right side of flashcard header (max 200 characters, default: empty string)

## Errors

All errors use standard DRF responses:

- `400 Bad Request` for validation failures (response includes `field: ["message"]`)
- `401 Unauthorized` when no/invalid token is provided
- `404 Not Found` when accessing missing records
- `405 Method Not Allowed` when using an unsupported verb

## Development Notes

- The API enforces authentication for every endpoint via `TokenAuthentication`.
- Run migrations after pulling these changes (`python manage.py migrate`) to drop legacy tables and fields.
- The `flashcards/tests.py` suite demonstrates authenticated CRUD flows and can be adapted for integration tests.
