Create celebrity selfie images and transition videos with GPT-4, SeedDream, and Kling

https://n8nworkflows.xyz/workflows/create-celebrity-selfie-images-and-transition-videos-with-gpt-4--seeddream--and-kling-12119


# Create celebrity selfie images and transition videos with GPT-4, SeedDream, and Kling

## 1. Workflow Overview

**Purpose:**  
This n8n workflow generates (1) celebrity “selfie” images and (2) short transition videos using those images. It uses GPT‑4 to generate structured prompts, SeedDream (via HTTP) to generate images asynchronously, and Kling (via HTTP) to generate videos asynchronously. Results are saved and updated in Google Sheets.

**Primary use cases:**
- Batch-generate consistent “celebrity selfie” images from a list of names and options submitted via an n8n Form.
- Poll asynchronous image/video generation APIs until completion.
- Persist outputs (image URLs, statuses, request IDs, video URLs) in Google Sheets for later processing.

### 1.1 Image Generation (Form-driven)
- Collect inputs (celebrity list + options)
- Split into one item per celebrity
- Use GPT‑4 to produce a structured image prompt
- Submit prompt to SeedDream
- Poll SeedDream job status until ready
- Fetch final image result
- Append row(s) to Google Sheets

### 1.2 Video Generation (Manual + Sheets-driven)
- Manually start video pass
- Read “ready” image rows from Google Sheets
- Build Kling prompt(s) using stored image data
- Submit video job to Kling
- Poll Kling job status until ready
- Fetch final video result
- Update rows in a (separate) “CelebrityVideos” sheet

### 1.3 Asynchronous Status Polling Loops (Image + Video)
Both pipelines implement:
- Store request metadata
- Wait fixed time
- Check status endpoint
- If not ready: wait again and re-check
- If ready: fetch result and persist

---

## 2. Block-by-Block Analysis

### Block A — Image pipeline entry & normalization
**Overview:** Receives user input from a Form Trigger, sets defaults/config, and transforms the input into individual items (one per celebrity) for batch processing.  
**Nodes involved:** `📝 Form Input`, `⚙️ Config`, `📥 Process & Split`, `🔄 Loop Each Celebrity`

#### Node: 📝 Form Input
- **Type / role:** Form Trigger (entry point) — starts workflow on form submission.
- **Config (interpreted):** Uses an n8n hosted form/webhook (`webhookId` present). Actual form fields are not included in the JSON.
- **I/O:**  
  - Output: form submission payload → `⚙️ Config`
- **Edge cases / failures:** missing expected form fields; unexpected data types (e.g., celebrity list not a string/array).

#### Node: ⚙️ Config
- **Type / role:** Set — intended to define workflow constants (API keys, model params, sheet IDs, timeouts, etc.).
- **Config:** Parameters are empty in the provided JSON; in a functioning workflow this typically sets:
  - SeedDream base URL / auth header
  - Kling base URL / auth header
  - Google Sheet IDs/tab names
  - Prompt settings (style, aspect ratio, quality)
- **I/O:** `📝 Form Input` → `⚙️ Config` → `📥 Process & Split`
- **Edge cases:** if not actually setting required fields, downstream HTTP/Sheets nodes will fail due to missing expressions/headers.

#### Node: 📥 Process & Split
- **Type / role:** Code — parses the incoming form payload and produces an array of items.
- **Config:** Empty in JSON; expected behavior:
  - Normalize celebrity list (comma/newline-separated → array)
  - Attach config values to each item
  - Possibly generate IDs, timestamps, or “status=queued”
- **I/O:** `⚙️ Config` → `📥 Process & Split` → `🔄 Loop Each Celebrity`
- **Edge cases:** parsing errors; empty list; trimming/duplicate names; non-UTF8 characters; exceeding API limits due to very long lists.

#### Node: 🔄 Loop Each Celebrity
- **Type / role:** Split In Batches — iterates through celebrities sequentially (or in controlled batch size).
- **Config:** Empty in JSON; default batch size in n8n is commonly 1 unless set.
- **Connections:**  
  - Input: `📥 Process & Split`
  - Output (loop branch index 1 per connections JSON): to `🤖 AI Generate Prompt`
  - After completion: continues when downstream node returns to it (here via `📊 Save to Sheets`)
- **Edge cases:** if batch size > 1, prompt generation and polling may overlap and hit rate limits; if the loop isn’t “fed back” correctly, it may stop after one item.

---

### Block B — GPT‑4 structured prompt generation (images)
**Overview:** Uses a GPT‑4 Chat model via LangChain nodes to produce a structured prompt, then merges it into a single prompt payload for SeedDream.  
**Nodes involved:** `GPT-4 Language Model`, `Parse Prompt Response`, `🤖 AI Generate Prompt`, `🔗 Merge Prompt`

#### Node: GPT-4 Language Model
- **Type / role:** LangChain ChatOpenAI LLM provider — supplies the model to the chain.
- **Config:** Empty in JSON; typically includes:
  - Model name (e.g., `gpt-4o`, `gpt-4.1`, etc.)
  - Temperature, max tokens
  - OpenAI credential selection
- **Connections:** outputs through `ai_languageModel` to `🤖 AI Generate Prompt`.
- **Edge cases:** missing OpenAI credentials; model access restrictions; token limit exceeded if celebrity context is too verbose.

#### Node: Parse Prompt Response
- **Type / role:** Structured Output Parser — enforces JSON/schema output from the model.
- **Config:** Empty in JSON; usually defines a schema like:
  - `prompt` (string)
  - `negativePrompt` (string)
  - `style` / `camera` / `lighting` fields
- **Connections:** outputs through `ai_outputParser` to `🤖 AI Generate Prompt`.
- **Edge cases:** LLM produces invalid JSON; schema mismatch; missing required keys → chain fails.

#### Node: 🤖 AI Generate Prompt
- **Type / role:** LangChain LLM Chain — runs prompt template against GPT‑4 and parses into structured output.
- **Config:** Empty in JSON; typically includes:
  - System/user instructions for “celebrity selfie”
  - Constraints for realism, framing, lighting, background, etc.
  - Safety constraints (avoid disallowed content)
- **I/O:**  
  - Input: item per celebrity from `🔄 Loop Each Celebrity`
  - Output: structured prompt fields → `🔗 Merge Prompt`
- **Edge cases:** chain misconfiguration (no prompt template); parser not attached; long celebrity names or extra attributes causing prompt bloat.

#### Node: 🔗 Merge Prompt
- **Type / role:** Code — composes the final request body for SeedDream from chain output + config.
- **Config:** Empty in JSON; expected actions:
  - Choose the final `prompt` string
  - Add optional negative prompt, aspect ratio, steps, seed, etc.
  - Attach celebrity name and metadata for storage in Sheets
- **Connections:** `🤖 AI Generate Prompt` → `🔗 Merge Prompt` → `🎨 SeedDream Generate`
- **Edge cases:** missing fields from parser; invalid parameter mapping to SeedDream API.

---

### Block C — SeedDream image generation + polling loop
**Overview:** Submits image generation to SeedDream, stores request metadata, waits, checks status repeatedly, and fetches the final result when ready.  
**Nodes involved:** `🎨 SeedDream Generate`, `💾 Store Request`, `⏳ Wait 45s`, `📊 Check Status`, `🔗 Merge Status`, `✅ Ready?`, `⏳ Retry 20s`, `🔄 Retry`, `📥 Get Result`

#### Node: 🎨 SeedDream Generate
- **Type / role:** HTTP Request — calls SeedDream “create/generate” endpoint.
- **Config:** Empty in JSON; typically:
  - Method: POST
  - URL: SeedDream endpoint
  - Auth: API key header or bearer token
  - Body: prompt payload from `🔗 Merge Prompt`
- **I/O:** `🔗 Merge Prompt` → `🎨 SeedDream Generate` → `💾 Store Request`
- **Edge cases:** 401/403 auth; 429 rate limit; 5xx; request body mismatch; timeouts for large payloads.

#### Node: 💾 Store Request
- **Type / role:** Code — extracts and stores job/request identifiers (e.g., `requestId`, `taskId`) needed for polling.
- **Connections:** → `⏳ Wait 45s`
- **Edge cases:** SeedDream response missing expected ID field; multiple IDs depending on API version.

#### Node: ⏳ Wait 45s
- **Type / role:** Wait — delays before first status check.
- **Config:** Empty in JSON but node name implies 45 seconds.
- **Connections:** → `📊 Check Status`
- **Edge cases:** long-running jobs may require longer waits; too short wait increases status-check calls and rate limits.

#### Node: 📊 Check Status
- **Type / role:** HTTP Request — calls SeedDream “status” endpoint with stored request ID.
- **Connections:** → `🔗 Merge Status`
- **Edge cases:** status endpoint differs from generate endpoint; transient 5xx; request ID not found; throttling.

#### Node: 🔗 Merge Status
- **Type / role:** Code — combines original request context with returned status (e.g., `state=RUNNING|SUCCEEDED|FAILED`).
- **Connections:** → `✅ Ready?`
- **Edge cases:** inconsistent status strings; nested fields.

#### Node: ✅ Ready?
- **Type / role:** IF — branches on “ready/succeeded?” condition.
- **Connections:**  
  - True → `📥 Get Result`
  - False → `⏳ Retry 20s`
- **Edge cases:** if condition checks wrong field/path, it may loop forever or exit early; “FAILED” should ideally break and log.

#### Node: ⏳ Retry 20s
- **Type / role:** Wait — delay between repeated status checks.
- **Connections:** → `🔄 Retry`
- **Edge cases:** same as above; high frequency may rate-limit.

#### Node: 🔄 Retry
- **Type / role:** Code — prepares next poll attempt (possibly increments retry count, rebuilds request).
- **Connections:** → `📊 Check Status`
- **Edge cases:** missing retry cap → infinite loop; retry state not persisted.

#### Node: 📥 Get Result
- **Type / role:** HTTP Request — fetches the final generated image result (image URL(s), metadata).
- **Connections:** → `📋 Prepare Output`
- **Edge cases:** result not immediately available even when status says ready; signed URLs expiring; response large.

---

### Block D — Persisting image results to Google Sheets
**Overview:** Formats the final image result into a row schema and appends to Google Sheets, then continues the batch loop for the next celebrity.  
**Nodes involved:** `📋 Prepare Output`, `📊 Save to Sheets`

#### Node: 📋 Prepare Output
- **Type / role:** Code — maps API output into a consistent structure for Sheets.
- **Expected fields:** celebrity name, prompt used, requestId, status, image URL, timestamps, etc.
- **Connections:** → `📊 Save to Sheets`
- **Edge cases:** URL missing; multiple images; need to pick first; special characters.

#### Node: 📊 Save to Sheets
- **Type / role:** Google Sheets — append (or update) rows for generated images.
- **Config:** Empty in JSON; typically:
  - Credential: Google OAuth2/service account
  - Spreadsheet ID
  - Sheet/tab name (e.g., “Celebrities”)
  - Operation: Append row
- **Connections:** → feeds back into `🔄 Loop Each Celebrity` (to process next item)
- **Edge cases:** permission errors; wrong sheet/tab; column mismatch; API quota limits; partial writes if batching.

---

### Block E — Video pipeline entry & selection of eligible rows
**Overview:** Manually triggers a second pipeline that reads rows from Sheets (generated images), filters for those ready for video generation, and loops through them.  
**Nodes involved:** `▶️ Start Video Generation`, `⚙️ Video Config`, `📊 Read from Sheets`, `🔍 Filter Ready Rows`, `🔄 Loop Each Video`

#### Node: ▶️ Start Video Generation
- **Type / role:** Manual Trigger — used to start the video stage on demand.
- **Connections:** → `⚙️ Video Config`
- **Edge cases:** none (manual only).

#### Node: ⚙️ Video Config
- **Type / role:** Set — defines Kling-related settings and sheet references.
- **Config:** Empty in JSON; typically includes:
  - Kling endpoints and auth
  - Target output sheet “CelebrityVideos”
  - Video parameters (duration, fps, aspect ratio)
- **Connections:** → `📊 Read from Sheets`

#### Node: 📊 Read from Sheets
- **Type / role:** Google Sheets — reads rows containing generated images.
- **Config:** Empty in JSON; typically operation “Read” from a specific tab/range.
- **Connections:** → `🔍 Filter Ready Rows`
- **Edge cases:** empty sheet; large sheet causing pagination needs; inconsistent headers.

#### Node: 🔍 Filter Ready Rows
- **Type / role:** Code — filters rows to those eligible for video generation (e.g., image status “ready” and no video yet).
- **Connections:** → `🔄 Loop Each Video`
- **Edge cases:** status naming mismatch; duplicate processing if no “processed” marker.

#### Node: 🔄 Loop Each Video
- **Type / role:** Split In Batches — iterates through filtered rows.
- **Connections:** (per connections JSON, loop branch index 1) → `📝 Build Video Prompt`; feedback occurs from `📊 Save to CelebrityVideos`.
- **Edge cases:** same as image loop; ensure batch size and feedback path are correct.

---

### Block F — Kling video generation + polling loop
**Overview:** Builds a Kling prompt from the image row, submits video generation, stores the job ID, waits, polls status, then fetches the final video and updates Sheets.  
**Nodes involved:** `📝 Build Video Prompt`, `🎥 Kling Generate Video`, `💾 Store Video Request`, `⏳ Wait 120s`, `📊 Check Video Status`, `🔗 Merge Video Status`, `✅ Video Ready?`, `⏳ Retry 60s`, `🔄 Retry Video Check`, `📥 Get Video Result`, `📋 Prepare Update`, `📊 Save to CelebrityVideos`

#### Node: 📝 Build Video Prompt
- **Type / role:** Code — constructs Kling request payload.
- **Expected behavior:**
  - Reference the generated image URL as start frame / conditioning input (if Kling supports)
  - Add “transition” instructions (camera movement, scene morph, etc.)
  - Include duration and style controls from `⚙️ Video Config`
- **Connections:** → `🎥 Kling Generate Video`
- **Edge cases:** Kling API expects different field names; image URL must be publicly accessible; signed URLs may expire.

#### Node: 🎥 Kling Generate Video
- **Type / role:** HTTP Request — submits Kling generation job.
- **Connections:** → `💾 Store Video Request`
- **Edge cases:** auth failures; payload mismatch; 429; long job queue.

#### Node: 💾 Store Video Request
- **Type / role:** Code — extracts Kling job ID for polling.
- **Connections:** → `⏳ Wait 120s`
- **Edge cases:** job ID missing; multiple IDs.

#### Node: ⏳ Wait 120s
- **Type / role:** Wait — delay before status check.
- **Connections:** → `📊 Check Video Status`

#### Node: 📊 Check Video Status
- **Type / role:** HTTP Request — checks Kling job status.
- **Connections:** → `🔗 Merge Video Status`

#### Node: 🔗 Merge Video Status
- **Type / role:** Code — merges context + status response.
- **Connections:** → `✅ Video Ready?`

#### Node: ✅ Video Ready?
- **Type / role:** IF — decides whether to fetch result or retry.
- **Connections:**  
  - True → `📥 Get Video Result`  
  - False → `⏳ Retry 60s`
- **Edge cases:** failed jobs should be handled separately; incorrect field check can cause infinite loop.

#### Node: ⏳ Retry 60s
- **Type / role:** Wait — delay between video status polls.
- **Connections:** → `🔄 Retry Video Check`

#### Node: 🔄 Retry Video Check
- **Type / role:** Code — prepares next polling request.
- **Connections:** → `📊 Check Video Status`
- **Edge cases:** missing retry cap; no backoff strategy.

#### Node: 📥 Get Video Result
- **Type / role:** HTTP Request — fetches final video URL(s)/assets.
- **Connections:** → `📋 Prepare Update`
- **Edge cases:** result not ready despite status; large response; expiring URLs.

#### Node: 📋 Prepare Update
- **Type / role:** Code — builds row update payload for output sheet.
- **Expected fields:** celebrity name, image URL, video URL, job ID, status, timestamps, prompt.
- **Connections:** → `📊 Save to CelebrityVideos`

#### Node: 📊 Save to CelebrityVideos
- **Type / role:** Google Sheets — writes video results (append or update).
- **Connections:** → `🔄 Loop Each Video`
- **Edge cases:** same as Sheets write issues; ensure correct key if doing updates (row number or unique ID).

---

### Block G — Sticky notes / annotations
**Overview:** The workflow contains several sticky note nodes, but their `content` is empty in the provided JSON, so there are no actual comments to propagate.  
**Nodes involved:** `Workflow Overview`, `Image Steps`, `Video Steps`, `Section: Image Generation`, `Section: Video Generation`, `Status Check`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Canvas annotation | — | — |  |
| Image Steps | Sticky Note | Canvas annotation | — | — |  |
| Video Steps | Sticky Note | Canvas annotation | — | — |  |
| Section: Image Generation | Sticky Note | Canvas annotation | — | — |  |
| Section: Video Generation | Sticky Note | Canvas annotation | — | — |  |
| Status Check | Sticky Note | Canvas annotation | — | — |  |
| 📝 Form Input | Form Trigger | Entry point for image pipeline | — | ⚙️ Config |  |
| ⚙️ Config | Set | Define constants for image stage | 📝 Form Input | 📥 Process & Split |  |
| 📥 Process & Split | Code | Parse form input into per-celebrity items | ⚙️ Config | 🔄 Loop Each Celebrity |  |
| 🔄 Loop Each Celebrity | Split In Batches | Iterate celebrities | 📥 Process & Split; 📊 Save to Sheets | 🤖 AI Generate Prompt |  |
| GPT-4 Language Model | LangChain ChatOpenAI | LLM provider for prompt generation | — (provider) | 🤖 AI Generate Prompt (ai) |  |
| Parse Prompt Response | LangChain Structured Output Parser | Enforce structured LLM output | — (provider) | 🤖 AI Generate Prompt (ai) |  |
| 🤖 AI Generate Prompt | LangChain LLM Chain | Generate structured image prompt | 🔄 Loop Each Celebrity; GPT-4 Language Model; Parse Prompt Response | 🔗 Merge Prompt |  |
| 🔗 Merge Prompt | Code | Build SeedDream request payload | 🤖 AI Generate Prompt | 🎨 SeedDream Generate |  |
| 🎨 SeedDream Generate | HTTP Request | Submit SeedDream image job | 🔗 Merge Prompt | 💾 Store Request |  |
| 💾 Store Request | Code | Store job/request ID | 🎨 SeedDream Generate | ⏳ Wait 45s |  |
| ⏳ Wait 45s | Wait | Delay before polling | 💾 Store Request | 📊 Check Status |  |
| 📊 Check Status | HTTP Request | Poll SeedDream job status | ⏳ Wait 45s; 🔄 Retry | 🔗 Merge Status |  |
| 🔗 Merge Status | Code | Merge status with context | 📊 Check Status | ✅ Ready? |  |
| ✅ Ready? | IF | Branch on ready vs retry | 🔗 Merge Status | 📥 Get Result; ⏳ Retry 20s |  |
| ⏳ Retry 20s | Wait | Delay between polls | ✅ Ready? (false) | 🔄 Retry |  |
| 🔄 Retry | Code | Prepare next poll iteration | ⏳ Retry 20s | 📊 Check Status |  |
| 📥 Get Result | HTTP Request | Fetch final image result | ✅ Ready? (true) | 📋 Prepare Output |  |
| 📋 Prepare Output | Code | Map result to Sheets row | 📥 Get Result | 📊 Save to Sheets |  |
| 📊 Save to Sheets | Google Sheets | Persist image rows | 📋 Prepare Output | 🔄 Loop Each Celebrity |  |
| ▶️ Start Video Generation | Manual Trigger | Entry point for video pipeline | — | ⚙️ Video Config |  |
| ⚙️ Video Config | Set | Define constants for video stage | ▶️ Start Video Generation | 📊 Read from Sheets |  |
| 📊 Read from Sheets | Google Sheets | Read eligible image rows | ⚙️ Video Config | 🔍 Filter Ready Rows |  |
| 🔍 Filter Ready Rows | Code | Filter rows for video generation | 📊 Read from Sheets | 🔄 Loop Each Video |  |
| 🔄 Loop Each Video | Split In Batches | Iterate videos to generate | 🔍 Filter Ready Rows; 📊 Save to CelebrityVideos | 📝 Build Video Prompt |  |
| 📝 Build Video Prompt | Code | Build Kling request payload | 🔄 Loop Each Video | 🎥 Kling Generate Video |  |
| 🎥 Kling Generate Video | HTTP Request | Submit Kling video job | 📝 Build Video Prompt | 💾 Store Video Request |  |
| 💾 Store Video Request | Code | Store Kling job ID | 🎥 Kling Generate Video | ⏳ Wait 120s |  |
| ⏳ Wait 120s | Wait | Delay before polling | 💾 Store Video Request | 📊 Check Video Status |  |
| 📊 Check Video Status | HTTP Request | Poll Kling job status | ⏳ Wait 120s; 🔄 Retry Video Check | 🔗 Merge Video Status |  |
| 🔗 Merge Video Status | Code | Merge status with context | 📊 Check Video Status | ✅ Video Ready? |  |
| ✅ Video Ready? | IF | Branch on ready vs retry | 🔗 Merge Video Status | 📥 Get Video Result; ⏳ Retry 60s |  |
| ⏳ Retry 60s | Wait | Delay between polls | ✅ Video Ready? (false) | 🔄 Retry Video Check |  |
| 🔄 Retry Video Check | Code | Prepare next poll iteration | ⏳ Retry 60s | 📊 Check Video Status |  |
| 📥 Get Video Result | HTTP Request | Fetch final video result | ✅ Video Ready? (true) | 📋 Prepare Update |  |
| 📋 Prepare Update | Code | Map result to Sheets update | 📥 Get Video Result | 📊 Save to CelebrityVideos |  |
| 📊 Save to CelebrityVideos | Google Sheets | Persist video rows/updates | 📋 Prepare Update | 🔄 Loop Each Video |  |

---

## 4. Reproducing the Workflow from Scratch

> Note: The provided JSON omits nearly all node parameters (prompts, URLs, headers, sheet IDs, schemas). The steps below describe the required structure and the configurations you must supply.

### A) Image Generation workflow (Form → SeedDream → Sheets)

1. **Create `📝 Form Input` (Form Trigger)**
   - Add the form fields you need, typically:
     - `celebrity_names` (text area; comma/newline separated)
     - Optional: `style`, `background`, `aspect_ratio`, `count`, etc.
   - Save and copy the production URL if needed.

2. **Create `⚙️ Config` (Set)**
   - Add fields such as:
     - `seeddreamBaseUrl`
     - `seeddreamApiKey` (prefer referencing n8n credentials or environment variables)
     - `googleSpreadsheetId`
     - `googleSheetTabImages` (e.g., `CelebrityImages`)
     - Poll timings (45s first wait, 20s retry wait)
   - Connect: `📝 Form Input` → `⚙️ Config`

3. **Create `📥 Process & Split` (Code)**
   - Parse `celebrity_names` into an array.
   - Output one item per celebrity, including config fields.
   - Connect: `⚙️ Config` → `📥 Process & Split`

4. **Create `🔄 Loop Each Celebrity` (Split In Batches)**
   - Set **Batch Size = 1** (recommended to avoid rate-limit bursts).
   - Connect: `📥 Process & Split` → `🔄 Loop Each Celebrity`

5. **Create LangChain nodes for prompt generation**
   - **`GPT-4 Language Model`** (ChatOpenAI):
     - Choose OpenAI credential
     - Select model (your choice)
     - Set temperature as desired
   - **`Parse Prompt Response`** (Structured Output Parser):
     - Define a schema, e.g. fields: `prompt`, `negativePrompt`, maybe `camera`, `lighting`
   - **`🤖 AI Generate Prompt`** (LLM Chain):
     - Create a prompt template using celebrity name and style constraints.
     - Attach the Chat model and parser via the dedicated AI connections.
   - Connect main path: `🔄 Loop Each Celebrity` → `🤖 AI Generate Prompt`

6. **Create `🔗 Merge Prompt` (Code)**
   - Combine chain output into SeedDream request body, e.g.:
     - `prompt`
     - `negative_prompt`
     - `width/height` or `aspect_ratio`
     - `num_images`
   - Connect: `🤖 AI Generate Prompt` → `🔗 Merge Prompt`

7. **Create `🎨 SeedDream Generate` (HTTP Request)**
   - Method: POST
   - URL: `{{$json.seeddreamBaseUrl}}/.../generate`
   - Headers: Authorization (Bearer/API key)
   - Body: from `🔗 Merge Prompt`
   - Connect: `🔗 Merge Prompt` → `🎨 SeedDream Generate`

8. **Create `💾 Store Request` (Code)**
   - Extract job ID from SeedDream response into e.g. `jobId`.
   - Keep celebrity + prompt context in the item.
   - Connect: `🎨 SeedDream Generate` → `💾 Store Request`

9. **Create `⏳ Wait 45s` (Wait)**
   - Wait time: 45 seconds
   - Connect: `💾 Store Request` → `⏳ Wait 45s`

10. **Create polling nodes**
   - **`📊 Check Status`** (HTTP Request)
     - Method: GET/POST depending on API
     - URL includes `jobId`
   - **`🔗 Merge Status`** (Code): merge status into item
   - **`✅ Ready?`** (IF): condition `status == "succeeded"` (or API-specific)
   - **`⏳ Retry 20s`** (Wait): 20 seconds
   - **`🔄 Retry`** (Code): preserve jobId and return to status check
   - Connect in this order:
     - `⏳ Wait 45s` → `📊 Check Status` → `🔗 Merge Status` → `✅ Ready?`
     - IF true → `📥 Get Result`
     - IF false → `⏳ Retry 20s` → `🔄 Retry` → `📊 Check Status`

11. **Create `📥 Get Result` (HTTP Request)**
   - Fetch final asset(s) using `jobId`.
   - Connect: `✅ Ready? (true)` → `📥 Get Result`

12. **Create `📋 Prepare Output` (Code)**
   - Build a flat object for Google Sheets columns:
     - celebrity, prompt, jobId, status, imageUrl, createdAt, etc.
   - Connect: `📥 Get Result` → `📋 Prepare Output`

13. **Create `📊 Save to Sheets` (Google Sheets)**
   - Credential: Google (OAuth2 or Service Account)
   - Spreadsheet: your ID
   - Sheet/tab: images tab
   - Operation: Append row
   - Connect: `📋 Prepare Output` → `📊 Save to Sheets`

14. **Close the loop**
   - Connect: `📊 Save to Sheets` → `🔄 Loop Each Celebrity` to process next celebrity in batch.

---

### B) Video Generation workflow (Manual → Kling → Sheets)

15. **Create `▶️ Start Video Generation` (Manual Trigger)**
   - Entry point for the video phase.

16. **Create `⚙️ Video Config` (Set)**
   - Store:
     - `klingBaseUrl`
     - `klingApiKey`
     - `googleSpreadsheetId`
     - `googleSheetTabImages` (source)
     - `googleSheetTabVideos` (destination, e.g. `CelebrityVideos`)
     - Poll timings (120s initial, 60s retry)
   - Connect: `▶️ Start Video Generation` → `⚙️ Video Config`

17. **Create `📊 Read from Sheets` (Google Sheets)**
   - Read rows from the images tab (all rows or a defined range).
   - Connect: `⚙️ Video Config` → `📊 Read from Sheets`

18. **Create `🔍 Filter Ready Rows` (Code)**
   - Filter rows where:
     - image status is ready/succeeded
     - and video not generated yet (blank `videoUrl` or a flag)
   - Connect: `📊 Read from Sheets` → `🔍 Filter Ready Rows`

19. **Create `🔄 Loop Each Video` (Split In Batches)**
   - Batch size 1 recommended.
   - Connect: `🔍 Filter Ready Rows` → `🔄 Loop Each Video`

20. **Create `📝 Build Video Prompt` (Code)**
   - Construct Kling request payload using:
     - imageUrl (as reference input if supported)
     - transition instructions
     - duration, fps, aspect ratio
   - Connect: `🔄 Loop Each Video` → `📝 Build Video Prompt`

21. **Create `🎥 Kling Generate Video` (HTTP Request)**
   - Method: POST
   - URL: `{{$json.klingBaseUrl}}/.../generate`
   - Headers: Authorization
   - Body: from `📝 Build Video Prompt`
   - Connect: `📝 Build Video Prompt` → `🎥 Kling Generate Video`

22. **Create `💾 Store Video Request` (Code)**
   - Extract Kling `jobId`.
   - Connect: `🎥 Kling Generate Video` → `💾 Store Video Request`

23. **Create wait + polling loop**
   - `⏳ Wait 120s` → `📊 Check Video Status` → `🔗 Merge Video Status` → `✅ Video Ready?`
   - If false: `⏳ Retry 60s` → `🔄 Retry Video Check` → `📊 Check Video Status`
   - If true: `📥 Get Video Result`

24. **Create `📥 Get Video Result` (HTTP Request)**
   - Fetch final video URL(s) using `jobId`.
   - Connect: `✅ Video Ready? (true)` → `📥 Get Video Result`

25. **Create `📋 Prepare Update` (Code)**
   - Build row(s) for the output sheet or update payload.
   - Connect: `📥 Get Video Result` → `📋 Prepare Update`

26. **Create `📊 Save to CelebrityVideos` (Google Sheets)**
   - Operation: Append or Update (your choice)
   - If Update, you need a stable key (row number or unique ID).
   - Connect: `📋 Prepare Update` → `📊 Save to CelebrityVideos`

27. **Close the loop**
   - Connect: `📊 Save to CelebrityVideos` → `🔄 Loop Each Video`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| All sticky note nodes are present but have empty content in the provided workflow JSON. | No additional embedded documentation was supplied. |
| The workflow relies heavily on HTTP APIs (SeedDream, Kling) and LangChain nodes, but the JSON omits URLs, headers, schemas, and prompt templates. You must supply these for a working rebuild. | Configuration required in HTTP Request and LangChain nodes. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.