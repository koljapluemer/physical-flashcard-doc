### MVP SPEC — PDF Rendering Backend

#### 1. Core Purpose

Convert user-submitted HTML (containing text, images, and math markup) into a high-quality PDF using headless Chromium (Puppeteer/Playwright). Math rendering is handled via MathJax inside the HTML before PDF export.

---

#### 2. Stack

* **Runtime:** Node.js ≥ 18
* **Language:** TypeScript
* **Framework:** Express
* **Renderer:** Puppeteer (headless Chromium)
* **PDF Output:** A4 format, print backgrounds enabled
* **Transport:** HTTP REST (JSON input, binary PDF output)
* **Hosting Target:** Render.com or Fly.io

---

#### 3. API Endpoints

##### `POST /render`

**Purpose:** Generate a PDF from provided HTML.

**Request Body (JSON):**

```json
{
  "html": "<html>...</html>",
  "options": {
    "format": "A4",
    "margin": { "top": "1cm", "bottom": "1cm", "left": "1cm", "right": "1cm" }
  }
}
```

**Behavior:**

1. Launch headless browser (`--no-sandbox`).
2. Load provided HTML via `page.setContent()`.
3. Wait until network idle and `window.MathJax.typesetPromise()` completes (if present).
4. Render to PDF with specified options.
5. Return PDF as binary stream.

**Response:**

* `Content-Type: application/pdf`
* Binary body containing PDF file

**Error Responses:**

* `400` if `html` missing
* `500` on rendering failure

---

#### 4. Internal Modules

| Module      | Purpose                              |
| ----------- | ------------------------------------ |
| `server.ts` | Express app entrypoint               |
| `render.ts` | Encapsulates Puppeteer logic         |
| `types.ts`  | Defines input/output types           |
| `config.ts` | PDF defaults (margins, format, etc.) |

---

#### 5. Example Flow

1. Frontend (CKEditor + MathType) sends POST `/render` with HTML.
2. Backend injects HTML into headless browser.
3. MathJax runs, HTML fully rendered.
4. Puppeteer saves PDF.
5. PDF returned to browser for download.

---

#### 6. Deployment

* Native Node.js service (Render.com / Fly.io buildpacks or similar).
* Build step: `npm install && npm run build`.
* Start command: `npm start` (runs `node dist/server.js`).
* Environment variables:

  * `PORT`
  * `PUPPETEER_SKIP_DOWNLOAD` (optional if using system Chrome)

---

#### 7. Non-Goals

* No user authentication
* No file persistence
* No frontend hosting
* No queueing or multi-user concurrency handling (MVP only)
