Generate Sora videos from Google Sheets and save results to Google Drive

https://n8nworkflows.xyz/workflows/generate-sora-videos-from-google-sheets-and-save-results-to-google-drive-12287


# Generate Sora videos from Google Sheets and save results to Google Drive

## 1. Workflow Overview

**Purpose:** Automatically generate AI videos with OpenAI Sora from prompts stored in Google Sheets, wait/poll until rendering finishes, generate an SEO-friendly title with GPT‑4.1‑mini, download the final video, upload it to Google Drive, then write back the Drive URL, title, and status into Google Sheets.

**Typical use cases**
- Batch video generation pipeline driven by spreadsheet rows.
- Semi-automated content production where Sheets acts as a queue and Drive as storage.

### 1.1 Data Input & Filtering
Reads all rows from a Google Sheet, keeps only rows not yet processed (empty `STATUS`), and normalizes the downstream field names.

### 1.2 Video Generation & Status Check (Polling Loop)
Submits each prompt to the OpenAI Videos API (`sora-2`), waits 60 seconds, checks job status, then routes:
- `failed` → update sheet with error
- `queued` / `in_progress` → wait again (poll)
- `completed` → proceed

### 1.3 Title Generation (LLM)
When the job is completed, uses a LangChain Agent + OpenAI chat model (`gpt-4.1-mini`) to generate an SEO YouTube title, returning structured JSON with `{title, video_id, prompt}`.

### 1.4 Save Results
Downloads the completed video file, uploads it to Google Drive, and updates the original sheet row with Drive link, title, and status.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Data Input & Filtering
**Overview:** Pulls all candidate prompts from Sheets, filters to only unprocessed items, and ensures expected field names/types exist for later HTTP requests.

**Nodes involved**
- Manual Trigger
- Get Video Prompts from Sheet
- Filter Unprocessed Videos
- Normalize Video Parameters

#### Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point.
- **Config:** No parameters; starts workflow on demand.
- **Outputs:** Sends one empty item into the next node.
- **Failure modes:** None (local execution only).

#### Get Video Prompts from Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows from a Google Sheet tab.
- **Config choices (interpreted):**
  - Targets spreadsheet `example` and sheet/tab `Video prompts` (gid `1347410646`).
  - Operation is implied by node name/usage: fetch rows (read).
- **Credentials:** Google Sheets OAuth2.
- **Outputs:** Items per row, including columns like `PROMPT`, `DURATION (In Seconds)`, `VIDEO RESOLUTION`, `STATUS`, etc.
- **Edge cases / failures:**
  - OAuth consent/refresh token issues.
  - Sheet/tab renamed or missing.
  - Column header mismatches (case/spacing changes break later expressions).

#### Filter Unprocessed Videos
- **Type / role:** `n8n-nodes-base.filter` — keeps only rows with empty status.
- **Key condition:** `STATUS` **is empty** (`={{ $json.STATUS }}` with “empty” operator).
- **Outputs:** Only items where `STATUS` is blank.
- **Edge cases:**
  - If `STATUS` contains whitespace (e.g., `" "`), it may not count as empty depending on n8n behavior; consider trimming if needed.
  - If `STATUS` column doesn’t exist, expression may evaluate unexpectedly or fail validation.

#### Normalize Video Parameters
- **Type / role:** `n8n-nodes-base.set` — standardizes fields for the OpenAI request.
- **Assignments created:**
  - `PROMPT` = `{{$json.PROMPT}}`
  - `DURATION (In Seconds)` (number) = `{{$json['DURATION (In Seconds)']}}`
  - `VIDEO RESOLUTION` = `{{$json['VIDEO RESOLUTION']}}`
- **Outputs:** Same item plus normalized fields.
- **Edge cases:**
  - Non-numeric duration values can coerce to `NaN`/string issues later; the HTTP node falls back to `4` seconds only if the field is falsy in its expression.

**Sticky note covering this block**
- **Step 1: Data Input & Filtering**
  - Reads rows from the sheet
  - Filters eligible prompts
  - Normalizes fields used downstream

---

### Block 1.2 — Video Generation & Status Check (Polling Loop)
**Overview:** Creates a Sora generation job, then repeatedly waits and checks status until completion or failure. Failed jobs are written back to the sheet.

**Nodes involved**
- Create Sora Video Job
- Wait 60s for Rendering
- Check Video Status
- Route by Video Status
- Update video status to failed

#### Create Sora Video Job
- **Type / role:** `n8n-nodes-base.httpRequest` — POST to OpenAI Videos API to start a job.
- **Request:**
  - `POST https://api.openai.com/v1/videos`
  - JSON body (expressions):
    - `model`: `"sora-2"`
    - `prompt`: `{{ $json.PROMPT }}`
    - `size`: `{{ $json['VIDEO RESOLUTION'] || '1280x720' }}`
    - `seconds`: `{{ $json['DURATION (In Seconds)'] || 4 }}`
- **Headers:**
  - `Authorization: Bearer YOUR_TOKEN_HERE` (hard-coded placeholder in node)
  - `Content-Type: application/json`
- **Batching:** enabled (batchSize 1, interval 3000ms).
- **Outputs:** Expected OpenAI response containing at least a job `id`.
- **Edge cases / failures:**
  - Token not replaced → 401.
  - Model access not enabled → 403 / 404-like model error.
  - Invalid `seconds` / `size` → 400.
  - Rate limits / transient 5xx.
- **Important integration note:** This node uses a literal token string, not an n8n credential object. For production, replace with a credential or environment variable expression.

#### Wait 60s for Rendering
- **Type / role:** `n8n-nodes-base.wait` — delay between status polls.
- **Config:** Wait `60` seconds.
- **Outputs:** Pass-through after the delay.
- **Edge cases:**
  - Long queues may require many loops; consider max retries to avoid infinite runs.

#### Check Video Status
- **Type / role:** `n8n-nodes-base.httpRequest` — GET job status.
- **Request:** `GET https://api.openai.com/v1/videos/{{ $json.id }}`
- **Headers:**
  - Has an `Authorization` header entry **without a value** in the workflow JSON.
- **Outputs:** Expected JSON containing `status` plus other metadata.
- **Likely failure / misconfiguration:**
  - Missing Authorization value will cause 401 unless n8n injects it some other way (it doesn’t, by default). This should be fixed to match the token approach used in the other HTTP nodes (or use credentials).

#### Route by Video Status
- **Type / role:** `n8n-nodes-base.switch` — routes based on `{{$json.status}}`.
- **Rules / outputs:**
  1. If `status == "failed"` → **Update video status to failed**
  2. If `status == "queued"` → **Wait 60s for Rendering** (loop)
  3. If `status == "in_progress"` → **Wait 60s for Rendering** (loop)
  4. If `status == "completed"` → **Generate SEO Title with AI**
- **Edge cases:**
  - If OpenAI returns new statuses (e.g., `canceled`) they will drop to “no match” and stop unless a default route is added.
  - If `status` path differs (nested), routing fails.

#### Update video status to failed
- **Type / role:** `n8n-nodes-base.googleSheets` — updates the matching row with failure info.
- **Operation:** Update rows in the same sheet by matching column `PROMPT`.
- **Columns written:**
  - `PROMPT` = `{{$json.prompt}}`
  - `STATUS` = `{{$json.status}} - {{$json.error.message}}`
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - If the failure payload doesn’t include `error.message`, expression may be undefined; consider fallback text.
  - Matching by `PROMPT` can update the wrong row if prompts are duplicated. Using a unique row id/row_number would be safer.

**Sticky note covering this block**
- **Step 2: Video Generation & Status Check**
  - Creates a video generation job
  - Checks status
  - Updates a row in the sheet if video generation failed

---

### Block 1.3 — Title Generation (LLM)
**Overview:** After the video job is completed, the workflow generates a YouTube-ready SEO title from the prompt and duration. Output is constrained to a structured JSON shape.

**Nodes involved**
- Generate SEO Title with AI
- GPT-4 Model
- Structured Output Parser

#### Generate SEO Title with AI
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent node that produces structured output.
- **Prompt content (key variables):**
  - Video prompt: `{{ $json.prompt }}`
  - Video duration: `{{ $json.seconds }} seconds`
  - Requires JSON-only return:
    - `title` (String)
    - `video_id` = `{{ $json.id }}`
    - `prompt` = `{{ $json.prompt }}`
- **Connections:**
  - Uses **GPT-4 Model** as its language model input.
  - Uses **Structured Output Parser** as its output parser.
  - Main output goes to **Download Completed Video**.
- **Edge cases:**
  - If upstream `Check Video Status` returns different field names (e.g., prompt not echoed as `$json.prompt`), the prompt template becomes blank.
  - Agent may sometimes include extra text; the structured parser enforces schema, but may throw parsing errors.

#### GPT-4 Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI chat model configuration for LangChain.
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential (configured in n8n).
- **Edge cases:** OpenAI quota/rate limits; model availability by account/region.

#### Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema.
- **Schema example:**
  ```json
  { "title": "String", "video_id": "String", "prompt": "String" }
  ```
- **Edge cases:** Any non-conforming LLM response causes parser failure; you may want error handling (try/catch branch) if used in production.

**Sticky note covering this block**
- **Step 3: Title Generation**

---

### Block 1.4 — Save Results
**Overview:** Downloads the completed video binary from OpenAI, uploads it to Google Drive, then updates the Google Sheet row with Drive URL, title, and status.

**Nodes involved**
- Download Completed Video
- Upload to Google Drive
- Add video URL and update video status

#### Download Completed Video
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches the rendered video as a file/binary.
- **Request:** `GET https://api.openai.com/v1/videos/{{ $json.output.video_id }}/content`
  - Note: relies on `Generate SEO Title with AI` output being in `$json.output.video_id`.
- **Response format:** `file` (binary).
- **Header:** `Authorization: Bearer YOUR_TOKEN_HERE` (hard-coded placeholder).
- **Batching:** batchSize 1, interval 3000ms.
- **Outputs:** Binary video data + JSON including `output.title`, `output.prompt`, etc.
- **Edge cases / failures:**
  - If the completed job’s video id is not the same as the job id, this is correct; otherwise mismatch can 404.
  - Token placeholder not replaced → 401.
  - Large files may hit execution memory limits depending on n8n configuration.

#### Upload to Google Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads binary to Drive.
- **Config:**
  - File name: `={{ $json.output.title }}`
  - Drive: “My Drive”, folder: root
- **Credentials:** Google Drive OAuth2.
- **Outputs:** Drive file metadata including `webViewLink` (used downstream).
- **Edge cases:**
  - If title contains invalid filename characters, Drive typically sanitizes but may produce unexpected names.
  - If binary property name isn’t what the Drive node expects, upload may fail (ensure “Binary Property” default matches the HTTP node output).

#### Add video URL and update video status
- **Type / role:** `n8n-nodes-base.googleSheets` — updates the original row with results.
- **Operation:** Update rows, matching by `PROMPT`.
- **Columns written:**
  - `PROMPT` = `{{ $items('Download Completed Video')[$itemIndex].json.output.prompt }}`
  - `STATUS` = `Video Created`
  - `VIDEO URL` = `{{ $json.webViewLink }}`
  - `VIDEO TITLE` = `{{ $items('Download Completed Video')[$itemIndex].json.output.title }}`
- **Connection / data dependency:**
  - Uses the current item from **Upload to Google Drive** for `webViewLink`.
  - Uses cross-node item lookup to pull `prompt` and `title` from **Download Completed Video** for the same `$itemIndex`.
- **Edge cases:**
  - If item ordering changes (e.g., merges, splits, or different batching), `$itemIndex` alignment can break and write wrong titles/prompts.
  - Duplicate `PROMPT` values can update the wrong row.

**Sticky note covering this block**
- **Step 4: Save Results**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual start | — | Get Video Prompts from Sheet | Generate Sora AI videos, save to Google Drive, and update Google Sheets… (setup/requirements note) |
| Get Video Prompts from Sheet | n8n-nodes-base.googleSheets | Read prompt rows from Sheets | Manual Trigger | Filter Unprocessed Videos | Step 1: Data Input & Filtering — Reads rows, filters eligible prompts, normalizes fields |
| Filter Unprocessed Videos | n8n-nodes-base.filter | Keep rows where STATUS is empty | Get Video Prompts from Sheet | Normalize Video Parameters | Step 1: Data Input & Filtering — Reads rows, filters eligible prompts, normalizes fields |
| Normalize Video Parameters | n8n-nodes-base.set | Standardize field names/types | Filter Unprocessed Videos | Create Sora Video Job | Step 1: Data Input & Filtering — Reads rows, filters eligible prompts, normalizes fields |
| Create Sora Video Job | n8n-nodes-base.httpRequest | Create OpenAI Sora video job | Normalize Video Parameters | Wait 60s for Rendering | Step 2: Video Generation & Status Check — Creates job, checks status, updates sheet on failure |
| Wait 60s for Rendering | n8n-nodes-base.wait | Delay between status polls | Create Sora Video Job; Route by Video Status | Check Video Status | Step 2: Video Generation & Status Check — Creates job, checks status, updates sheet on failure |
| Check Video Status | n8n-nodes-base.httpRequest | Poll OpenAI job status | Wait 60s for Rendering | Route by Video Status | Step 2: Video Generation & Status Check — Creates job, checks status, updates sheet on failure |
| Route by Video Status | n8n-nodes-base.switch | Branch on status (failed/queued/in_progress/completed) | Check Video Status | Update video status to failed; Wait 60s for Rendering; Generate SEO Title with AI | Step 2: Video Generation & Status Check — Creates job, checks status, updates sheet on failure |
| Update video status to failed | n8n-nodes-base.googleSheets | Write error status back to sheet | Route by Video Status | — | Step 2: Video Generation & Status Check — Creates job, checks status, updates sheet on failure |
| Generate SEO Title with AI | @n8n/n8n-nodes-langchain.agent | Generate structured SEO title JSON | Route by Video Status | Download Completed Video | Step 3: Title Generation |
| GPT-4 Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for agent | — (AI connection) | Generate SEO Title with AI | Step 3: Title Generation |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON output schema | — (AI connection) | Generate SEO Title with AI | Step 3: Title Generation |
| Download Completed Video | n8n-nodes-base.httpRequest | Download rendered video binary | Generate SEO Title with AI | Upload to Google Drive | Step 4: Save Results |
| Upload to Google Drive | n8n-nodes-base.googleDrive | Upload video to Drive | Download Completed Video | Add video URL and update video status | Step 4: Save Results |
| Add video URL and update video status | n8n-nodes-base.googleSheets | Update sheet with Drive link/title/status | Upload to Google Drive | — | Step 4: Save Results |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | (contains workflow description/setup/requirements) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | Step 2: Video Generation & Status Check… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | Step 3: Title Generation |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | Step 4: Save Results |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment block | — | — | Generate Sora AI videos, save to Google Drive, and update Google Sheets… |
| Sticky Note (Step 1) | n8n-nodes-base.stickyNote | Comment block | — | — | Step 1: Data Input & Filtering… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheet**
   - Spreadsheet with a tab (e.g., “Video prompts”).
   - Columns: `PROMPT`, `DURATION (In Seconds)`, `VIDEO RESOLUTION`, `VIDEO TITLE`, `VIDEO URL`, `STATUS`.
   - Ensure `PROMPT` values are unique if you plan to match/update by `PROMPT`.

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**.

3. **Add node: Google Sheets (read rows)**
   - Node type: **Google Sheets**
   - Configure to **read/get all rows** from your spreadsheet + tab.
   - Select your **Google Sheets OAuth2** credential.
   - Connect: **Manual Trigger → Get Video Prompts from Sheet**.

4. **Add node: Filter (unprocessed rows)**
   - Node type: **Filter**
   - Condition: `STATUS` **is empty** (left value `{{$json.STATUS}}`).
   - Connect: **Get Video Prompts from Sheet → Filter Unprocessed Videos**.

5. **Add node: Set (normalize parameters)**
   - Node type: **Set**
   - Add fields:
     - `PROMPT` (string) = `{{$json.PROMPT}}`
     - `DURATION (In Seconds)` (number) = `{{$json['DURATION (In Seconds)']}}`
     - `VIDEO RESOLUTION` (string) = `{{$json['VIDEO RESOLUTION']}}`
   - Connect: **Filter Unprocessed Videos → Normalize Video Parameters**.

6. **Add node: HTTP Request (create Sora job)**
   - Node type: **HTTP Request**
   - Method: `POST`
   - URL: `https://api.openai.com/v1/videos`
   - Body: JSON with:
     - `model`: `sora-2`
     - `prompt`: `{{$json.PROMPT}}`
     - `size`: `{{$json['VIDEO RESOLUTION'] || '1280x720'}}`
     - `seconds`: `{{$json['DURATION (In Seconds)'] || 4}}`
   - Headers:
     - `Authorization: Bearer <YOUR_OPENAI_API_KEY>`
     - `Content-Type: application/json`
   - Connect: **Normalize Video Parameters → Create Sora Video Job**.

7. **Add node: Wait**
   - Node type: **Wait**
   - Amount: `60 seconds`
   - Connect: **Create Sora Video Job → Wait 60s for Rendering**.

8. **Add node: HTTP Request (check status)**
   - Node type: **HTTP Request**
   - Method: `GET`
   - URL: `https://api.openai.com/v1/videos/{{$json.id}}`
   - Header:
     - `Authorization: Bearer <YOUR_OPENAI_API_KEY>` (ensure it is set; the provided workflow is missing the value here)
   - Connect: **Wait 60s for Rendering → Check Video Status**.

9. **Add node: Switch (route by status)**
   - Node type: **Switch**
   - Value to evaluate: `{{$json.status}}`
   - Rules:
     - equals `failed` → failure branch
     - equals `queued` → loop branch
     - equals `in_progress` → loop branch
     - equals `completed` → success branch
   - Connect: **Check Video Status → Route by Video Status**.
   - Connect loop branches back to **Wait 60s for Rendering**.

10. **Add node: Google Sheets (update failure)**
   - Node type: **Google Sheets**
   - Operation: **Update**
   - Matching column: `PROMPT`
   - Set:
     - `PROMPT` = `{{$json.prompt}}`
     - `STATUS` = `{{$json.status}} - {{$json.error.message}}`
   - Connect: **Route by Video Status (failed) → Update video status to failed**.

11. **Add LLM nodes for title generation**
   - Add node: **OpenAI Chat Model** (LangChain)  
     - Model: `gpt-4.1-mini`
     - Credential: **OpenAI API** in n8n
   - Add node: **Structured Output Parser**
     - Schema: `{ "title": "String", "video_id": "String", "prompt": "String" }`
   - Add node: **AI Agent**
     - Prompt uses `{{$json.prompt}}`, `{{$json.seconds}}`, and returns JSON with `video_id: {{$json.id}}`.
     - Connect AI Model → Agent (AI connection)
     - Connect Output Parser → Agent (AI connection)
   - Connect: **Route by Video Status (completed) → Generate SEO Title with AI**.

12. **Add node: HTTP Request (download video)**
   - Node type: **HTTP Request**
   - Method: `GET`
   - URL: `https://api.openai.com/v1/videos/{{$json.output.video_id}}/content`
   - Response: **File** (binary)
   - Header: `Authorization: Bearer <YOUR_OPENAI_API_KEY>`
   - Connect: **Generate SEO Title with AI → Download Completed Video**.

13. **Add node: Google Drive (upload)**
   - Node type: **Google Drive**
   - Operation: Upload (from binary)
   - File name: `{{$json.output.title}}`
   - Folder: choose target folder (root in the provided workflow)
   - Credential: **Google Drive OAuth2**
   - Connect: **Download Completed Video → Upload to Google Drive**.

14. **Add node: Google Sheets (update success)**
   - Node type: **Google Sheets**
   - Operation: **Update**
   - Matching column: `PROMPT`
   - Set:
     - `PROMPT` = `{{$items('Download Completed Video')[$itemIndex].json.output.prompt}}`
     - `STATUS` = `Video Created`
     - `VIDEO URL` = `{{$json.webViewLink}}`
     - `VIDEO TITLE` = `{{$items('Download Completed Video')[$itemIndex].json.output.title}}`
   - Connect: **Upload to Google Drive → Add video URL and update video status**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate Sora AI videos, save to Google Drive, and update Google Sheets” + description of how it works, setup steps, and requirements (OpenAI Sora access + Google OAuth2 for Sheets/Drive). | Sticky note content embedded in the workflow canvas |
| Potential configuration issue: **Check Video Status** node’s `Authorization` header has no value in the provided JSON; it must be set (or replaced by a shared credential mechanism). | OpenAI polling step |
| Matching Sheet rows by `PROMPT` can mis-update if prompts repeat; consider matching by a unique row id/row_number column instead. | Reliability / data integrity |