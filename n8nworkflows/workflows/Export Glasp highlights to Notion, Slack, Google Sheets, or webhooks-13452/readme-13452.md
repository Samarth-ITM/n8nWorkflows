Export Glasp highlights to Notion, Slack, Google Sheets, or webhooks

https://n8nworkflows.xyz/workflows/export-glasp-highlights-to-notion--slack--google-sheets--or-webhooks-13452


# Export Glasp highlights to Notion, Slack, Google Sheets, or webhooks

## 1. Workflow Overview

**Purpose:** Automatically export new Glasp highlights on a schedule by calling the Glasp Highlights Export API, deduplicating already-exported documents, and formatting each document’s highlights into structured fields (including Markdown/text) ready to send to a destination such as Notion, Slack, Google Sheets, email, or a webhook.

**Target use cases:**
- Keep a knowledge base (Notion/Sheets) in sync with newly highlighted content from Glasp
- Push reading/highlight digests to Slack or email
- Trigger downstream automations via webhook using structured highlight data

### 1.1 Scheduling & Run State Initialization
Runs every 6 hours and computes the `updatedAfter` timestamp to query Glasp. Uses workflow static data to persist `lastRunAt` and dedupe exported documents.

### 1.2 Glasp API Retrieval (Paginated)
Fetches highlights from Glasp using `updatedAfter`, following pagination via `nextPageCursor`.

### 1.3 Deduplication, Tracking, and Output Formatting
Aggregates all pages, filters out documents already exported, updates tracking state, and outputs one item per new document with a rich payload (markdown, text, highlight array, metadata). If nothing new is found, outputs a single informational item.

### 1.4 Destination Connection (User-Added)
A sticky note indicates where to add your preferred export nodes after the formatter node.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Run State Initialization

**Overview:** Triggers the workflow periodically and prepares API query parameters based on the last successful run, including a buffer to avoid missing late-arriving updates.

**Nodes Involved:**
- Schedule Trigger
- Prepare Parameters
- Sticky Note - Setup
- Sticky Note - Security

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — entry point; periodic execution.
- **Configuration (interpreted):**
  - Runs every **6 hours** (`hoursInterval: 6`).
- **Inputs / outputs:**
  - **Output:** Connects to **Prepare Parameters**.
- **Version-specific notes:** Type version `1.2` (standard schedule trigger behavior).
- **Edge cases / failure types:**
  - If n8n instance is down, scheduled runs are missed unless you use external scheduling/queueing patterns.
  - Timezone behavior depends on n8n instance settings for schedule triggers.

#### Node: Prepare Parameters
- **Type / role:** `Code` (`n8n-nodes-base.code`) — computes `updatedAfter` and loads persisted dedupe state.
- **Configuration choices:**
  - Uses `global` workflow static data: `$getWorkflowStaticData('global')`.
  - Determines **first run** by absence of `sd.lastRunAt`.
  - Sets `baseDate`:
    - First run: `now - 24h`
    - Otherwise: `sd.lastRunAt`
  - Applies **5-minute buffer**: `baseDate - 5 minutes` to avoid missing highlights updated near the boundary.
  - Ensures `sd.exportedDocs` exists (object map of `docId -> exportedAt`).
- **Key variables / outputs:**
  - Outputs:
    - `updatedAfter` (ISO string)
    - `exportedDocs` (current tracking map; mainly informational for later logic)
- **Inputs / outputs:**
  - **Input:** From Schedule Trigger.
  - **Output:** To Glasp API.
- **Version-specific notes:** Type version `2` (Code node).
- **Edge cases / failure types:**
  - If workflow static data is cleared/reset, the workflow will re-export up to the last 24h (minus buffer).
  - Date parsing of `sd.lastRunAt` assumes it is a valid ISO string.

#### Sticky Note - Setup (documentation node)
- **Type / role:** `Sticky Note` — provides setup instructions.
- **Content (preserved):**
  - Get token: https://glasp.co/settings/access_token  
  - Create Header Auth credential:
    - Name: `Authorization`
    - Value: `Bearer YOUR_TOKEN`
  - Assign it to the **Glasp API** node.
  - Optional schedule adjustments (default: every 6 hours).
- **Connections:** None (visual/documentation only).

#### Sticky Note - Security (documentation node)
- **Type / role:** `Sticky Note` — security notes.
- **Content (preserved):**
  - Token stored in encrypted credentials, not in workflow JSON
  - Export tracking cleans after 30 days
  - No secrets in code
- **Connections:** None.

---

### Block 2 — Glasp API Retrieval (Paginated)

**Overview:** Calls Glasp’s highlights export endpoint, passing the computed `updatedAfter` value, and paginates through results using `nextPageCursor`.

**Nodes Involved:**
- Glasp API

#### Node: Glasp API
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — fetch highlight export data from Glasp.
- **Configuration choices (interpreted):**
  - **Method:** GET
  - **URL:** `https://api.glasp.co/v1/highlights/export`
  - **Authentication:** Generic credential type → `httpHeaderAuth`
    - Credential name in workflow: **Glasp Token** (must exist in n8n)
    - Intended header per sticky note: `Authorization: Bearer <token>`
  - **Query parameters:**
    - `updatedAfter={{ $json.updatedAfter }}`
  - **Response format:** JSON
  - **Pagination enabled:**
    - Uses a computed “next URL” expression:
      - `={{ $response.body.nextPageCursor ? 'https://api.glasp.co/v1/highlights/export?updatedAfter=' + $json.updatedAfter + '&pageCursor=' + $response.body.nextPageCursor : '' }}`
    - `maxRequests: 100`
    - Mode: `responseContainsNextURL` (n8n will continue while the expression returns a non-empty URL).
- **Inputs / outputs:**
  - **Input:** From Prepare Parameters.
  - **Output:** To Filter & Format.
- **Version-specific notes:** Type version `4.2` (HTTP node; pagination/options vary by version).
- **Edge cases / failure types:**
  - **401/403** if token missing/invalid, header misconfigured, or revoked.
  - **429** rate limiting if Glasp throttles; may need retry/backoff (not implemented here).
  - Pagination expression assumes `nextPageCursor` is in `$response.body`; if API shape changes, pagination may stop early or error.
  - If `updatedAfter` is malformed, API may return error or empty results.

---

### Block 3 — Deduplication, Tracking, and Output Formatting

**Overview:** Collects results across all paginated pages, filters documents already exported, updates persistent tracking (`exportedDocs`, `lastRunAt`), cleans old tracking entries, and outputs a formatted export payload per document.

**Nodes Involved:**
- Filter & Format
- Sticky Note - Destination Guide

#### Node: Filter & Format
- **Type / role:** `Code` — aggregates paginated items, deduplicates, persists run state, formats output.
- **Configuration choices (interpreted):**
  - Loads/updates workflow static data (`global`):
    - `sd.exportedDocs` map
    - `sd.lastRunAt`
  - Aggregation approach:
    - Uses `$input.all()` to collect all incoming items (each item is a page from Glasp API).
    - Concatenates `data.results` arrays.
  - Filtering rules:
    - Keep docs where:
      - `!sd.exportedDocs[doc.id]` (not previously exported)
      - `doc.highlights` exists and `doc.highlights.length > 0`
  - Tracking updates:
    - Marks each exported doc id with current timestamp
    - Sets `sd.lastRunAt = now`
    - Removes entries older than **30 days**
  - Output behavior:
    - If no new docs: returns one item `{ message: 'No new highlights found.', count: 0 }`
    - Otherwise: returns **one item per document** with:
      - Metadata: `documentId`, `title`, `url`, `glasp_url`, `domain`, `category`, `tags`, `author`, `thumbnail_url`, `document_note`, `is_favorite`, `createdAt`, `updatedAt`
      - Counts: `highlightCount`
      - Renderings:
        - `highlightsText` (plain text concatenation; includes note inline)
        - `highlightsMarkdown` (markdown with title, links, tags, doc note, then per-highlight quote block + note + color/date line)
      - `highlights[]` array of objects: `id`, `text`, `note`, `color`, `highlighted_at`
- **Key expressions/variables used:**
  - `$getWorkflowStaticData('global')`
  - `$input.all()`
- **Inputs / outputs:**
  - **Input:** From Glasp API (potentially many items due to pagination).
  - **Output:** Not connected in provided workflow (intentionally; user should attach destination nodes).
- **Version-specific notes:** Type version `2`.
- **Edge cases / failure types:**
  - If Glasp API returns a different structure (e.g., `results` not present), `allResults` may remain empty and you’ll always get “No new highlights found.”
  - Dedupe is document-level (`doc.id`), not highlight-level. If an existing document receives *new highlights* later, it will **not** export again because the doc id is already tracked. (This may be desired or may need adjusting to track `updated_at` or per-highlight IDs.)
  - `new Date(h.highlighted_at)` assumes valid date strings; invalid values may produce “Invalid Date” formatting in markdown line.
  - Large highlight payloads could exceed downstream limits (Slack message size, Notion property limits, webhook payload limits).

#### Sticky Note - Destination Guide (documentation node)
- **Type / role:** `Sticky Note` — explains how to attach destination nodes.
- **Content (preserved):**
  - Add export node after “Filter & Format”
  - Available fields per item:
    - `title`, `url`, `glasp_url`, `domain`, `category`, `tags`, `highlightCount`, `highlightsText`, `highlightsMarkdown`, `highlights[]`, `createdAt`, `updatedAt`
  - Examples:
    - Notion: Create Database Item
    - Slack: Send Message
    - Google Sheets: Append Row
    - Email: Send Email
    - Webhook: HTTP Request POST
- **Connections:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Periodic workflow entry (every 6 hours) | — | Prepare Parameters | ## [Setup - 2 steps]\n\n**Step 1:** Get your Glasp Access Token\nhttps://glasp.co/settings/access_token\n\n**Step 2:** Create a \"Header Auth\" credential\n- Name: Authorization\n- Value: Bearer YOUR_TOKEN\nThen assign it to the \"Glasp API\" node.\n\n**Optional:** Adjust schedule frequency\nby clicking the Schedule Trigger node.\nDefault: every 6 hours. |
| Prepare Parameters | Code | Compute `updatedAfter`, initialize/load tracking state | Schedule Trigger | Glasp API | ## [Setup - 2 steps]\n\n**Step 1:** Get your Glasp Access Token\nhttps://glasp.co/settings/access_token\n\n**Step 2:** Create a \"Header Auth\" credential\n- Name: Authorization\n- Value: Bearer YOUR_TOKEN\nThen assign it to the \"Glasp API\" node.\n\n**Optional:** Adjust schedule frequency\nby clicking the Schedule Trigger node.\nDefault: every 6 hours. |
| Glasp API | HTTP Request | Call Glasp export endpoint with pagination | Prepare Parameters | Filter & Format | ## [Security]\n\n- Access token is stored in n8n's\n  encrypted Credentials, never in\n  the workflow JSON.\n- Exported doc tracking auto-cleans\n  after 30 days.\n- No secrets in code. |
| Filter & Format | Code | Aggregate pages, dedupe, persist state, format output per doc | Glasp API | — (destination to be added) | ## [Connect Your Destination Here]\n\nAdd your preferred export node after \"Filter & Format\".\n\n**Available fields per item:**\n- `title` -- Article/page title\n- `url` -- Original URL\n- `glasp_url` -- Glasp page URL\n- `domain`, `category`, `tags`\n- `highlightCount`\n- `highlightsText` -- Plain text\n- `highlightsMarkdown` -- Markdown\n- `highlights[]` -- Array of objects\n- `createdAt`, `updatedAt`\n\n**Examples:**\n- Notion: Create Database Item\n- Slack: Send Message\n- Google Sheets: Append Row\n- Email: Send Email\n- Webhook: HTTP Request POST |
| Sticky Note - Destination Guide | Sticky Note | Documentation: where/how to connect destination | — | — |  |
| Sticky Note - Setup | Sticky Note | Documentation: token + credential setup + schedule | — | — |  |
| Sticky Note - Security | Sticky Note | Documentation: credential storage and retention | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Glasp Highlights Auto Export**
   - (Optional) Add tags: Glasp, Highlights, Export, Automation

2. **Add “Schedule Trigger” node**
   - Node type: **Schedule Trigger**
   - Configure:
     - Interval: **Every 6 hours**
   - This is the workflow entry point.

3. **Add “Prepare Parameters” node**
   - Node type: **Code**
   - Paste code that:
     - Loads `$getWorkflowStaticData('global')`
     - Computes `updatedAfter`:
       - First run: `now - 24h`
       - Otherwise: `sd.lastRunAt`
       - Subtract 5 minutes buffer
     - Ensures `sd.exportedDocs` exists
     - Returns one item with `updatedAfter` and `exportedDocs`
   - Connect: **Schedule Trigger → Prepare Parameters**

4. **Create Glasp credential (required)**
   - In n8n **Credentials**, create: **Header Auth** (HTTP Header Auth)
   - Configure:
     - Header Name: `Authorization`
     - Header Value: `Bearer <YOUR_GLASP_ACCESS_TOKEN>`
   - Token source: https://glasp.co/settings/access_token  
   - Name the credential something like: **Glasp Token**

5. **Add “Glasp API” node**
   - Node type: **HTTP Request**
   - Configure:
     - Method: **GET**
     - URL: `https://api.glasp.co/v1/highlights/export`
     - Authentication: **Header Auth** (HTTP Header Auth) using credential **Glasp Token**
     - Query parameter:
       - `updatedAfter` = expression `{{$json.updatedAfter}}`
     - Response: JSON
     - Pagination:
       - Enable pagination
       - Next URL expression:
         - `{{$response.body.nextPageCursor ? 'https://api.glasp.co/v1/highlights/export?updatedAfter=' + $json.updatedAfter + '&pageCursor=' + $response.body.nextPageCursor : '' }}`
       - Max requests: `100`
   - Connect: **Prepare Parameters → Glasp API**

6. **Add “Filter & Format” node**
   - Node type: **Code**
   - Paste code that:
     - Aggregates all input pages via `$input.all()`
     - Concatenates `item.json.results` arrays
     - Filters docs not in `sd.exportedDocs` and with `highlights.length > 0`
     - Updates `sd.exportedDocs[doc.id] = now`
     - Updates `sd.lastRunAt = now`
     - Deletes tracking entries older than 30 days
     - Outputs either:
       - A single “No new highlights found” item, or
       - One item per new document with fields including `highlightsMarkdown` and `highlightsText`
   - Connect: **Glasp API → Filter & Format**

7. **Add your destination export node (choose one)**
   - Connect it after **Filter & Format**.
   - Examples:
     - **Notion**: “Create Database Page” mapping fields like:
       - Title ← `{{$json.title}}`
       - URL ← `{{$json.url}}`
       - Content/Notes ← `{{$json.highlightsMarkdown}}`
       - Tags ← `{{$json.tags}}` (may require multi-select mapping)
     - **Slack**: “Send Message”
       - Text ← `{{$json.highlightsMarkdown}}` (may need trimming if too long)
     - **Google Sheets**: “Append Row”
       - Columns ← `title`, `url`, `highlightCount`, `highlightsText`, `updatedAt`, etc.
     - **Webhook/HTTP Request**: POST JSON body = `{{$json}}`

8. **(Optional) Add sticky notes for operators**
   - Add sticky notes mirroring:
     - Setup steps (token + credential)
     - Security notes
     - Destination guide / available fields

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get your Glasp Access Token | https://glasp.co/settings/access_token |
| Header Auth credential should be `Authorization: Bearer YOUR_TOKEN` | Used by the “Glasp API” HTTP Request node |
| Exported-doc tracking is stored in workflow static data and auto-cleans after 30 days | Avoids re-exporting the same document repeatedly |
| Add your destination node after “Filter & Format”; payload includes `highlightsMarkdown`, `highlightsText`, and `highlights[]` | See “Sticky Note - Destination Guide” content in workflow |