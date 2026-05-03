Generate Seedance AI lighting look dev variations with review via Drive, Notion, and Slack

https://n8nworkflows.xyz/workflows/generate-seedance-ai-lighting-look-dev-variations-with-review-via-drive--notion--and-slack-14881


# Generate Seedance AI lighting look dev variations with review via Drive, Notion, and Slack

Let me now carefully create the comprehensive reference document. Seedance AI Lighting Look Dev with Multi-Variation Rendering & Review Workflow

---

## 1. Workflow Overview

This workflow automates the generation of AI-driven lighting look development (look dev) references for VFX production. A lighting TD or VFX supervisor submits a structured lighting brief via a web form—including shot/sequence identification, a reference plate image, a mood description, scene type, and CG subject type. The system fans the brief into **five distinct lighting variation prompts** (Key Light, Soft Diffuse, Dramatic/Noir, Plate Match, and Comp Grade), submits each as an image-to-video job to the Seedance API, polls until every render completes, downloads the resulting videos, and uploads them to Google Drive with structured naming and metadata tagging. Finally, all five variations are aggregated into an HTML lookbook stored in a Notion database, and GPT-4o-mini drafts a contextual Slack notification that is posted to the lighting team channel.

### Logical Blocks

| Block | Name | Nodes | Purpose |
|-------|------|-------|---------|
| 1 | **Form Intake & Validation** | Form Trigger, Validate & Extract | Capture and validate the lighting brief |
| 2 | **Prompt Fan-Out** | Fan-Out: 5 Lighting Variations, Build Lighting Request Body | Expand one brief into five Seedance-ready request payloads |
| 3 | **AI Video Generation** | Seedance: Generate Lighting Ref, Merge Lighting Job ID + Metadata | Submit jobs and track returned job IDs |
| 4 | **Render Polling Loop** | Poll: Check Lighting Job Status, Lighting Render Complete?, Wait 20s | Loop-check each job every 20 s until completion |
| 5 | **Asset Storage** | Build Lighting Asset Metadata, Download Lighting Ref Video, Google Drive: Upload Lighting Ref | Extract video URLs, download files, upload to Drive |
| 6 | **Lookbook & Notifications** | Aggregate + Build Lookbook, Record the log in Notion, AI - Generate the slack message, Slack: Notify Lighting Team | Aggregate results, persist to Notion, notify team via Slack |
| 7 | **Error Handler** | On Workflow Error, Slack: Error Alert | Catch any workflow failure and alert the ops channel |

---

## 2. Block-by-Block Analysis

---

### Block 1 — Form Intake & Validation

**Overview:** A production-facing web form captures the lighting brief. A Code node immediately normalises dropdown labels into short codes and enforces the presence of required fields, preventing malformed data from propagating downstream.

**Nodes Involved:**
- `n8n Form: Lighting Brief`
- `Validate & Extract Lighting Brief`

---

#### Node: `n8n Form: Lighting Brief`

- **Type:** `n8n-nodes-base.formTrigger` (v2.2)
- **Technical Role:** Webhook-based entry point that renders a form at a unique URL and returns submitted fields as the first item of the workflow execution.
- **Configuration:**
  - **Form Title:** "Lighting & Look Dev Reference Request"
  - **Form Description:** "Submit a lighting brief to generate AI look development references for your sequence."
  - **Fields (8):**
    | Label | Type | Required | Placeholder / Options |
    |-------|------|----------|----------------------|
    | Shot Code | text | ✅ | `SQ010_SH020` |
    | Sequence Code | text | ❌ | `SQ010` |
    | Project ID | text | ❌ | `PROJ-001` |
    | Plate Image URL | text | ✅ | `https://your-server.com/onset_still.jpg` |
    | Lighting Mood Description | textarea | ✅ | Overcast soft light with cool blue shadows… |
    | Scene Type | dropdown | ✅ | Exterior Day, Exterior Night, Interior Day, Interior Night, Golden Hour, Overcast, Storm / Dramatic |
    | CG Subject Type | dropdown | ✅ | Character / Human, Vehicle, Creature, Hard Surface Object, Environment / Set Extension |
    | Supervisor Email | email | ❌ | `user@example.com` |
- **Key Expressions / Variables:** None (raw form output).
- **Input:** None (trigger).
- **Output:** Single item containing keys matching each `fieldLabel`.
- **Edge Cases:**
  - If the Plate Image URL is not publicly accessible, the Seedance API call will fail later.
  - Malformed email in Supervisor Email will pass through (not validated here).
  - Empty optional fields default to empty strings.

---

#### Node: `Validate & Extract Lighting Brief`

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Normalises and validates form output; maps human-readable dropdown values to internal short codes.
- **Configuration:**
  - **JS Code** performs the following:
    - Maps `Scene Type` via `sceneTypeMap` (e.g., "Exterior Day" → `"ext_day"`, "Storm / Dramatic" → `"storm"`).
    - Maps `CG Subject Type` via `cgSubjectMap` (e.g., "Character / Human" → `"character"`, "Environment / Set Extension" → `"environment"`).
    - Falls back to `"ext_day"` and `"character"` if unmapped values appear.
    - Derives `sequenceCode` from `Shot Code` split by `_` if Sequence Code is blank.
    - Sets `requestTimestamp` to current ISO string.
    - Throws explicit errors for missing `shotCode`, `plateImageUrl`, or `moodDescription`.
- **Key Expressions / Variables:**
  - `form['Shot Code']`, `form['Scene Type']`, etc.
  - `sceneTypeMap`, `cgSubjectMap` — lookup dictionaries.
- **Input:** Output of Form Trigger (1 item).
- **Output:** 1 item with normalised keys: `shotCode`, `sequenceCode`, `projectId`, `plateImageUrl`, `moodDescription`, `sceneType`, `cgSubjectType`, `supervisorEmail`, `requestTimestamp`.
- **Edge Cases:**
  - Unrecognised dropdown value → default code applied silently; no error thrown.
  - Shot Code containing no underscore and Sequence Code empty → `sequenceCode` becomes `"SEQ-000"`.
  - Any thrown error halts execution (caught by Error Handler if wired).

---

### Block 2 — Prompt Fan-Out

**Overview:** Expands the single validated brief into five distinct Seedance-ready request payloads. Each variation targets a different lighting rig aesthetic (Key Light, Soft Diffuse, Dramatic, Plate Match, Comp Grade) and carries a uniquely constructed prompt.

**Nodes Involved:**
- `Fan-Out: 5 Lighting Variations`
- `Build Lighting Request Body`

---

#### Node: `Fan-Out: 5 Lighting Variations`

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Creates 5 items from 1, each embedding a variation-specific prompt and metadata.
- **Configuration:**
  - **JS Code** constructs an array `variations` with 5 elements:
    | variantId | lightType | label | lightIcon | Prompt Focus |
    |-----------|-----------|-------|-----------|-------------|
    | LLD-KEY | key_light | Key Light Setup | ☀️ | Strong single key light at 45°, deep shadows, rim light |
    | LLD-SOFT | soft_diffuse | Soft Diffuse Setup | 🌥️ | Overcast sky dome, no hard shadows, soft wrap |
    | LLD-DRAM | dramatic_contrast | High Contrast / Dramatic | 🎭 | Noir-inspired single side source, deep blacks |
    | LLD-PLATE | plate_match | Plate Match Reference | 🎬 | Lighting precisely matched to reference plate |
    | LLD-COMP | comp_grade | Comp Grade Preview | 🎨 | Final comp-graded with colour correction, grain, vignette |
  - Each `safePrompt` is built by `JSON.stringify(...).slice(1,-1)` to produce a safely escaped string embedding `moodDescription`, `cgSubjectType`, and `sceneType` variables, plus Seedance flags `--duration 5 --camerafixed true`.
  - Returns `variations.map(v => ({ json: { ...v, ...d } }))` — every variation inherits all parent brief fields.
- **Key Expressions / Variables:**
  - `d.moodDescription`, `d.sceneType`, `d.cgSubjectType` — pulled from input.
  - `safePrompt` — escaped prompt string for each variant.
- **Input:** 1 item from Validate node.
- **Output:** 5 items, each with `variantId`, `lightType`, `label`, `lightIcon`, `safePrompt`, plus all brief fields.
- **Edge Cases:**
  - If `moodDescription` contains characters that break JSON.stringify (e.g., unbalanced quotes), the slice trick may produce a malformed prompt.
  - The `--duration 5 --camerafixed true` flags are Seedance-specific; if the model is changed, they may be ignored or cause errors.

---

#### Node: `Build Lighting Request Body`

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Wraps each variation into the exact JSON body format required by the Seedance image-to-video API.
- **Configuration:**
  - **JS Code** constructs:
    ```
    {
      model: 'seedance-1-5-pro-251215',
      content: [
        { type: 'text', text: input.safePrompt },
        { type: 'image_url', image_url: { url: input.plateImageUrl } }
      ],
      generate_audio: false,
      ratio: '16:9',
      duration: 5,
      watermark: false
    }
    ```
  - The body is stringified and stored as `requestBody`; a `mode: 'image_to_video'` flag is added.
- **Key Expressions / Variables:**
  - `input.safePrompt`, `input.plateImageUrl` — used to construct the multimodal request.
- **Input:** 5 items from Fan-Out.
- **Output:** 5 items, each now carrying `requestBody` (string) and `mode`.
- **Edge Cases:**
  - If `plateImageUrl` is invalid or returns a non-image content type, the Seedance API will reject the job.
  - Model identifier `seedance-1-5-pro-251215` is version-specific; if Seedance retires this model, all requests fail.

---

### Block 3 — AI Video Generation

**Overview:** Submits each lighting variation to the Seedance API and captures the returned job ID, pairing it with the originating variation's metadata for downstream tracking.

**Nodes Involved:**
- `Seedance: Generate Lighting Ref`
- `Merge Lighting Job ID + Metadata`

---

#### Node: `Seedance: Generate Lighting Ref`

- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Makes a POST request to the Seedance API to initiate an image-to-video generation job.
- **Configuration:**
  - **Method:** POST
  - **Send Body:** Yes (JSON)
  - **JSON Body:** `={{ JSON.parse($json.requestBody) }}` — parses the stringified body back to an object.
  - **Send Headers:** Yes
  - **Headers:**
    - `Authorization` — value not specified in JSON (must be set to `Bearer YOUR_SEEDANCE_API_KEY`).
    - `Content-Type: application/json`
  - **URL:** Not specified in JSON (must be configured to the Seedance endpoint, e.g., `https://api.seedance.ai/v1/generations` or equivalent).
- **Key Expressions / Variables:**
  - `$json.requestBody` — parsed into JSON body.
- **Input:** 5 items from Build Lighting Request Body.
- **Output:** 5 items containing API response (expected: `id`, `status`).
- **Edge Cases:**
  - Missing or invalid API key → 401/403 response, workflow fails.
  - Rate limiting → 429 response; no built-in retry logic.
  - URL not set → request fails immediately.
  - Network timeout → node-level timeout may need adjustment for long uploads.

---

#### Node: `Merge Lighting Job ID + Metadata`

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Combines the Seedance API response (job ID and status) with the original variation metadata so downstream nodes can track which job corresponds to which lighting variation.
- **Configuration:**
  - **JS Code:**
    - Reads `httpResult` from `$input.first().json` (API response).
    - Reads `variantData` from `$('Build Lighting Request Body').first().json` (upstream node reference).
    - Returns merged object: `{ ...variantData, id: httpResult.id, jobStatus: httpResult.status }`.
- **Key Expressions / Variables:**
  - Uses n8n node reference syntax `$('Build Lighting Request Body')` — relies on data being available at that node.
- **Input:** 1 item from Seedance HTTP response (per variation, executed in parallel for each of the 5 items).
- **Output:** 5 items, each with `id` (Seedance job ID) and `jobStatus`, plus all original variant and brief fields.
- **Edge Cases:**
  - If the API returns an unexpected schema (e.g., `job_id` instead of `id`), the merge produces `undefined` for `id`.
  - Node reference `$('Build Lighting Request Body')` may return stale data if execution mode changes.

---

### Block 4 — Render Polling Loop

**Overview:** A self-referencing loop that checks each Seedance job every 20 seconds. If the job status is `"succeeded"`, execution proceeds to asset extraction; otherwise, it waits and polls again.

**Nodes Involved:**
- `Poll: Check Lighting Job Status`
- `Lighting Render Complete?`
- `Wait 20s`

---

#### Node: `Poll: Check Lighting Job Status`

- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** GET request to check the current status of a Seedance generation job.
- **Configuration:**
  - **URL:** `=` (dynamic expression — must be set to the Seedance job status endpoint, e.g., `https://api.seedance.ai/v1/generations/{{$json.id}}`).
  - **Send Headers:** Yes
  - **Headers:**
    - `Authorization` — must contain `Bearer YOUR_SEEDANCE_API_KEY`.
- **Key Expressions / Variables:**
  - `$json.id` — Seedance job ID from the merged data.
- **Input:** 5 items from Merge node (first call) or from Wait 20s (subsequent calls).
- **Output:** 5 items with updated `status` and (if complete) `content.video_url` or similar fields.
- **Edge Cases:**
  - Job that never transitions to `succeeded` (e.g., `failed`, `cancelled`) will loop forever.
  - If the API changes the status field name, the IF condition will never match.
  - No maximum retry count — could loop indefinitely for stuck jobs.

---

#### Node: `Lighting Render Complete?`

- **Type:** `n8n-nodes-base.if` (v2)
- **Technical Role:** Conditional branch — true path continues to asset extraction; false path triggers the wait loop.
- **Configuration:**
  - **Combinator:** AND
  - **Condition:** `$json.status` equals `"succeeded"` (string, strict type validation, case-sensitive).
- **Key Expressions / Variables:**
  - `={{ $json.status }}` — compared against `"succeeded"`.
- **Input:** 5 items from Poll node.
- **Output:**
  - **True branch (index 0):** Items whose status is `"succeeded"` → `Build Lighting Asset Metadata`
  - **False branch (index 1):** Items not yet succeeded → `Wait 20s`
- **Edge Cases:**
  - Status values like `"Succeeded"` (capital S) or `"completed"` will not match → endless loop.
  - If the API returns `failed` status, the loop will wait and retry forever since `failed` ≠ `succeeded`.

---

#### Node: `Wait 20s`

- **Type:** `n8n-nodes-base.wait` (v1.1)
- **Technical Role:** Pauses execution for 20 seconds before re-polling the Seedance API.
- **Configuration:**
  - **Amount:** 20 (seconds)
  - **Webhook ID:** `lld-wait-webhook-001` — used internally by n8n to resume execution.
- **Key Expressions / Variables:** None.
- **Input:** Items from the false branch of the IF node.
- **Output:** Same items, forwarded to `Poll: Check Lighting Job Status` (loop back).
- **Edge Cases:**
  - If the n8n instance restarts during a wait, the webhook resume may be lost.
  - Wait is global per item — all 5 variations poll independently and in parallel.

---

### Block 5 — Asset Storage

**Overview:** Once a render completes, its video URL is extracted, the file is downloaded as binary, and uploaded to a designated Google Drive folder with structured naming. Asset metadata is enriched with pipeline tags.

**Nodes Involved:**
- `Build Lighting Asset Metadata`
- `Download Lighting Ref Video`
- `Google Drive: Upload Lighting Ref`

---

#### Node: `Build Lighting Asset Metadata`

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Extracts the video URL from the Seedance response, falls back through multiple possible response schemas, and builds a comprehensive metadata object including pipeline tracking tags.
- **Configuration:**
  - **JS Code:**
    - Reads `input` from `$input.first().json` (poll result).
    - Reads `variantData` from `$('Merge Lighting Job ID + Metadata').first().json`.
    - Extracts `videoUrl` from `input.content.video_url`, falling back to `input.output_url`, `input.video_url`, or a placeholder string containing the job ID.
    - Constructs output with all brief fields, variant fields, `videoUrl`, `jobId`, `resolution`, `duration`, `generatedAt`, and a `tags` object containing: `asset_type: 'lighting_lookdev'`, `light_type`, `scene_type`, `cg_subject`, `shot_code`, `ai_generated: true`, `review_status: 'pending_review'`, `workflow: 'lighting_lookdev_builder'`.
- **Key Expressions / Variables:**
  - Node reference: `$('Merge Lighting Job ID + Metadata')`.
- **Input:** 1 item from the true branch of the IF node (per variation).
- **Output:** 1 item with full metadata and `videoUrl`.
- **Edge Cases:**
  - If Seedance changes the response schema (e.g., nests video URL differently), the fallback chain may not find the URL, resulting in a placeholder string rather than a usable URL.
  - `resolution` and `duration` fields may be undefined if the API does not return them.

---

#### Node: `Download Lighting Ref Video`

- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Downloads the rendered video as a binary file for upload.
- **Configuration:**
  - **URL:** `={{ $json.videoUrl }}`
  - **Response Format:** File (binary)
- **Key Expressions / Variables:**
  - `$json.videoUrl` — the URL extracted by the metadata node.
- **Input:** 1 item from Build Lighting Asset Metadata.
- **Output:** 1 item with binary attachment (the downloaded `.mp4` file).
- **Edge Cases:**
  - If `videoUrl` is the placeholder string (not a real URL), the HTTP request will fail with a DNS/connection error.
  - Large files may require timeout configuration adjustment.
  - No authentication headers — if Seedance requires a signed URL or bearer token for downloads, this will fail.

---

#### Node: `Google Drive: Upload Lighting Ref`

- **Type:** `n8n-nodes-base.googleDrive` (v3)
- **Technical Role:** Uploads the downloaded video to a specific Google Drive folder.
- **Configuration:**
  - **Operation:** Upload file
  - **File Name:** `={{ $json.shotCode }}_lighting_{{ $json.lightType }}_{{ $json.variantId }}.mp4`
    - Example: `SQ010_SH020_lighting_key_light_LLD-KEY.mp4`
  - **Drive:** My Drive
  - **Folder ID:** `1fL57MP1QWsF0O_n7WuqfzJ0hO6I9Csrh` (cached name: "Simulation FX Concept Generator")
  - **Credential:** `googleDriveOAuth2Api` — "Automations.ai"
- **Key Expressions / Variables:**
  - `$json.shotCode`, `$json.lightType`, `$json.variantId` — used in filename.
- **Input:** 1 item with binary attachment from Download node.
- **Output:** 1 item with Google Drive file metadata (id, webViewLink, etc.).
- **Edge Cases:**
  - If the folder ID does not exist or the credential lacks write permission, upload fails.
  - Duplicate filenames will create a new version or a separate file depending on Drive settings.

---

### Block 6 — Lookbook & Notifications

**Overview:** Aggregates all five completed variations into a single HTML lookbook for Notion, and generates a human-readable Slack notification via GPT-4o-mini. Both outputs are delivered in parallel.

**Nodes Involved:**
- `Aggregate + Build Lookbook`
- `Record the log in Notion`
- `AI - Generate the slack message`
- `Slack: Notify Lighting Team`

---

#### Node: `Aggregate + Build Lookbook`

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Collects all variation metadata and constructs two output artefacts: an HTML lookbook table for Notion and a Slack-friendly text summary for AI rewriting.
- **Configuration:**
  - **JS Code:**
    - Reads all items from `$input.all()` (Google Drive upload results — but note: it also references `$('Build Lighting Asset Metadata').all()` to get metadata).
    - Builds `lookbookRows` — an HTML table row per variation with columns: Variation (icon + label), Light Type, Video link, Resolution, Generated timestamp.
    - Builds `lookbookHtml` — a full HTML document with: heading (shot code), sequence/project info, scene/subject type, mood description, plate link, the variations table, and usage notes.
    - Builds `slackLines` — a text block per variation with emoji, label, variant ID, video link, light type, timestamp — formatted with Slack markdown.
    - Returns a single item with: `shotCode`, `sequenceCode`, `projectId`, `moodDescription`, `sceneType`, `cgSubjectType`, `supervisorEmail`, `plateImageUrl`, `lookbookHtml`, `slackLines`, `totalVariations`, `generatedAt`.
- **Key Expressions / Variables:**
  - Node reference: `$('Build Lighting Asset Metadata')` — used to pull all metadata items.
- **Input:** 5 items (one per variation) from Google Drive Upload.
- **Output:** 1 item with aggregated lookbook HTML and Slack summary text.
- **Edge Cases:**
  - If fewer than 5 items arrive (e.g., one job failed silently), `metaItems[0]` may reference incomplete data.
  - HTML is not sanitised — any user-entered mood description containing `<script>` or similar will appear in the Notion page.
  - The `totalVariations` field reflects `metaItems.length`, which may be less than 5 if some variations failed.

---

#### Node: `Record the log in Notion`

- **Type:** `n8n-nodes-base.notion` (v2.2)
- **Technical Role:** Creates a new page in a Notion database to persist the lighting lookbook and associated metadata.
- **Configuration:**
  - **Operation:** Create database page
  - **Database:** "Lighting & Look Dev Reference DB" (`330802b9-1fa0-8027-8e92-f1c712bad8fd`)
  - **Properties:**
    - `shotCode` (title property): `={{ $json.shotCode }}`
    - `lookbookHtml` (rich_text property): `={{ $json.lookbookHtml }}`
  - **Credential:** `notionApi` — "saurabh notion"
- **Key Expressions / Variables:**
  - `$json.shotCode`, `$json.lookbookHtml`.
- **Input:** 1 item from Aggregate node.
- **Output:** Notion page creation result (page ID, URL, etc.).
- **Edge Cases:**
  - If the database ID or property names don't match the actual Notion database schema, the API returns a 400 error.
  - `lookbookHtml` stored as `rich_text` — Notion renders raw HTML tags, not formatted content (rich_text doesn't parse HTML).
  - Notion API has a 2000-character limit per rich_text block; long HTML may be truncated.

---

#### Node: `AI - Generate the slack message`

- **Type:** `@n8n/n8n-nodes-langchain.openAi` (v1.7)
- **Technical Role:** Uses GPT-4o-mini to rewrite the raw variation summary into a polished, professional Slack notification.
- **Configuration:**
  - **Model:** GPT-4O-MINI
  - **Options:** Max Tokens 300, Temperature 0.7
  - **Messages:**
    - **System prompt:** You are a VFX production assistant. Write a clear, professional Slack message. Use Slack markdown (bold with asterisks, code with backticks). Include shot code, mood, scene type, variations ready. Keep scannable with emoji and line breaks. End with a call-to-action. Never include raw HTML, JSON, or technical URLs. Output ONLY the Slack message text.
    - **User prompt:** Template injecting `$json.shotCode`, `$json.sequenceCode`, `$json.projectId`, `$json.sceneType`, `$json.cgSubjectType`, `$json.moodDescription`, `$json.totalVariations`, `$json.generatedAt`, `$json.supervisorEmail`, and `$json.slackLines`.
- **Key Expressions / Variables:**
  - All fields from the Aggregate node are injected into the user prompt.
- **Input:** 1 item from Aggregate node.
- **Output:** 1 item with AI-generated message text (in the LLM output field, typically `message.content` or `text`).
- **Edge Cases:**
  - If the OpenAI API key is missing or quota exhausted, the node fails.
  - 300 max tokens may truncate the message if all 5 variations are listed with links.
  - The system prompt explicitly forbids raw URLs but the user prompt contains video links — GPT may still include them.

---

#### Node: `Slack: Notify Lighting Team`

- **Type:** `n8n-nodes-base.slack` (v2.3)
- **Technical Role:** Posts the AI-generated message to a designated Slack channel.
- **Configuration:**
  - **Operation:** Send message
  - **Channel:** `C0ANFAL4WJ2` (cached name: "social")
  - **Message Text:** `={{ $json.message.content }}`
  - **Authentication:** OAuth2
- **Key Expressions / Variables:**
  - `$json.message.content` — the AI-generated Slack message text.
- **Input:** 1 item from AI - Generate the slack message.
- **Output:** Slack API response (ok, ts, etc.).
- **Edge Cases:**
  - If the AI node outputs the text in a different field name (e.g., `text` instead of `message.content`), the Slack message body will be empty.
  - Channel ID must exist and the bot must have `chat:write` scope.
  - If the message exceeds Slack's 40,000 character limit, it will be rejected.

---

### Block 7 — Error Handler

**Overview:** Catches any unhandled error across the entire workflow and immediately sends a formatted Slack alert to the operations channel, preventing silent failures.

**Nodes Involved:**
- `On Workflow Error`
- `Slack: Error Alert`

---

#### Node: `On Workflow Error`

- **Type:** `n8n-nodes-base.errorTrigger` (v1)
- **Technical Role:** Global error trigger that activates whenever any node in the workflow throws an uncaught error.
- **Configuration:** No parameters required.
- **Key Expressions / Variables:** None.
- **Input:** Automatic — triggered by runtime errors anywhere in the workflow.
- **Output:** 1 item containing `message` (error description), `node` (failing node name), etc.
- **Edge Cases:**
  - Only catches unhandled errors; nodes with built-in error branches won't trigger this.
  - If the Slack error alert node itself fails, the error is not re-caught (potential silent failure).

---

#### Node: `Slack: Error Alert`

- **Type:** `n8n-nodes-base.slack` (v2.3)
- **Technical Role:** Sends a formatted error notification to Slack.
- **Configuration:**
  - **Operation:** Send message
  - **Channel:** `C0ANFAL4WJ2` (cached name: "social")
  - **Message Text:** `=❌ *Lighting Look Dev Generator Failed*\n\nError: {{ $json.message }}\nTime: {{ new Date().toISOString() }}`
  - **Authentication:** OAuth2
- **Key Expressions / Variables:**
  - `$json.message` — the error message from the error trigger.
  - `new Date().toISOString()` — timestamp of the alert.
- **Input:** 1 item from On Workflow Error.
- **Output:** Slack API response.
- **Edge Cases:**
  - If the Slack OAuth token is invalid or the channel ID is wrong, the error alert itself fails silently.
  - No retry logic — a single Slack API failure means the alert is lost.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|------------|
| n8n Form: Lighting Brief | formTrigger | Web form entry point for lighting brief submission | — (trigger) | Validate & Extract Lighting Brief | 📋 Form Intake & Validation: Captures the lighting brief from a web form and validates all required fields — shot code, plate URL, mood description, and scene type — before anything reaches the AI pipeline. Bad data is rejected here, not downstream. |
| Validate & Extract Lighting Brief | code | Normalises dropdowns to codes, validates required fields | n8n Form: Lighting Brief | Fan-Out: 5 Lighting Variations | 📋 Form Intake & Validation: Captures the lighting brief from a web form and validates all required fields — shot code, plate URL, mood description, and scene type — before anything reaches the AI pipeline. Bad data is rejected here, not downstream. |
| Fan-Out: 5 Lighting Variations | code | Expands one brief into 5 lighting variation items with unique prompts | Validate & Extract Lighting Brief | Build Lighting Request Body | 🔀 Prompt Fan-Out: Expands the single brief into five distinct lighting prompts — Key Light, Soft Diffuse, Dramatic, Plate Match, and Comp Grade — and wraps each as a multimodal Seedance request with the plate image attached. |
| Build Lighting Request Body | code | Constructs Seedance API JSON body per variation | Fan-Out: 5 Lighting Variations | Seedance: Generate Lighting Ref | 🔀 Prompt Fan-Out: Expands the single brief into five distinct lighting prompts — Key Light, Soft Diffuse, Dramatic, Plate Match, and Comp Grade — and wraps each as a multimodal Seedance request with the plate image attached. |
| Seedance: Generate Lighting Ref | httpRequest | POSTs image-to-video generation request to Seedance API | Build Lighting Request Body | Merge Lighting Job ID + Metadata | 🎥 AI Video Generation: Submits each lighting prompt to the Seedance image-to-video API. Job IDs returned by the API are merged with original metadata so the polling loop tracks the correct render per variation. |
| Merge Lighting Job ID + Metadata | code | Merges Seedance job ID/status with variant metadata | Seedance: Generate Lighting Ref | Poll: Check Lighting Job Status | 🎥 AI Video Generation: Submits each lighting prompt to the Seedance image-to-video API. Job IDs returned by the API are merged with original metadata so the polling loop tracks the correct render per variation. |
| Poll: Check Lighting Job Status | httpRequest | GETs job status from Seedance API | Merge Lighting Job ID + Metadata, Wait 20s | Lighting Render Complete? | 🔄 Render Polling Loop: Checks each render job's status every 20 seconds. The workflow loops until the API confirms success — only then does the variation proceed to asset extraction and storage. |
| Lighting Render Complete? | if | Branches on status == "succeeded" | Poll: Check Lighting Job Status | Build Lighting Asset Metadata (true), Wait 20s (false) | 🔄 Render Polling Loop: Checks each render job's status every 20 seconds. The workflow loops until the API confirms success — only then does the variation proceed to asset extraction and storage. |
| Wait 20s | wait | Pauses execution 20 seconds before re-polling | Lighting Render Complete? (false) | Poll: Check Lighting Job Status | 🔄 Render Polling Loop: Checks each render job's status every 20 seconds. The workflow loops until the API confirms success — only then does the variation proceed to asset extraction and storage. |
| Build Lighting Asset Metadata | code | Extracts video URL and builds enriched asset metadata | Lighting Render Complete? (true) | Download Lighting Ref Video | 💾 Asset Storage: Downloads each rendered video and uploads it to Google Drive with a structured filename (`SHOT_lighting_TYPE_VARIANT.mp4`). Asset metadata is tagged with scene type, CG subject, shot code, and review status for pipeline tracking. |
| Download Lighting Ref Video | httpRequest | Downloads rendered video as binary file | Build Lighting Asset Metadata | Google Drive: Upload Lighting Ref | 💾 Asset Storage: Downloads each rendered video and uploads it to Google Drive with a structured filename (`SHOT_lighting_TYPE_VARIANT.mp4`). Asset metadata is tagged with scene type, CG subject, shot code, and review status for pipeline tracking. |
| Google Drive: Upload Lighting Ref | googleDrive | Uploads video to Drive folder with structured naming | Download Lighting Ref Video | Aggregate + Build Lookbook | 💾 Asset Storage: Downloads each rendered video and uploads it to Google Drive with a structured filename (`SHOT_lighting_TYPE_VARIANT.mp4`). Asset metadata is tagged with scene type, CG subject, shot code, and review status for pipeline tracking. |
| Aggregate + Build Lookbook | code | Aggregates all variations into HTML lookbook and Slack summary | Google Drive: Upload Lighting Ref | Record the log in Notion, AI - Generate the slack message | 📣 Lookbook & Notifications: Aggregates all five variations into an HTML lookbook page saved to Notion. GPT-4o-mini writes a team-facing Slack message with variation links, lighting mood context, and a clear review call-to-action. |
| Record the log in Notion | notion | Creates a database page with shot code and lookbook HTML | Aggregate + Build Lookbook | — | 📣 Lookbook & Notifications: Aggregates all five variations into an HTML lookbook page saved to Notion. GPT-4o-mini writes a team-facing Slack message with variation links, lighting mood context, and a clear review call-to-action. |
| AI - Generate the slack message | @n8n/n8n-nodes-langchain.openAi | GPT-4o-mini rewrites variation summary into a polished Slack message | Aggregate + Build Lookbook | Slack: Notify Lighting Team | 📣 Lookbook & Notifications: Aggregates all five variations into an HTML lookbook page saved to Notion. GPT-4o-mini writes a team-facing Slack message with variation links, lighting mood context, and a clear review call-to-action. |
| Slack: Notify Lighting Team | slack | Posts AI-generated notification to the lighting team channel | AI - Generate the slack message | — | 📣 Lookbook & Notifications: Aggregates all five variations into an HTML lookbook page saved to Notion. GPT-4o-mini writes a team-facing Slack message with variation links, lighting mood context, and a clear review call-to-action. 🔐 Credentials & Security: Use OAuth2 for Slack, Google Drive, and Notion. Store your Seedance and OpenAI keys as n8n credentials — never hardcode tokens in HTTP Request headers. Replace all `YOUR_*` placeholders before publishing or sharing this template. |
| On Workflow Error | errorTrigger | Global error trigger catching any unhandled workflow error | — (automatic) | Slack: Error Alert | ⚠️ Error Handler: Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |
| Slack: Error Alert | slack | Sends error notification to Slack ops channel | On Workflow Error | — | ⚠️ Error Handler: Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. 🔐 Credentials & Security: Use OAuth2 for Slack, Google Drive, and Notion. Store your Seedance and OpenAI keys as n8n credentials — never hardcode tokens in HTTP Request headers. Replace all `YOUR_*` placeholders before publishing or sharing this template. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Seedance AI Lighting Look Dev with Multi-Variation Rendering & Review Workflow".

2. **Add Form Trigger node:**
   - Type: `n8n Form Trigger` (v2.2)
   - Form Title: "Lighting & Look Dev Reference Request"
   - Form Description: "Submit a lighting brief to generate AI look development references for your sequence."
   - Add 8 fields:
     - "Shot Code" (text, required, placeholder `SQ010_SH020`)
     - "Sequence Code" (text, not required, placeholder `SQ010`)
     - "Project ID" (text, not required, placeholder `PROJ-001`)
     - "Plate Image URL" (text, required, placeholder `https://your-server.com/onset_still.jpg`)
     - "Lighting Mood Description" (textarea, required, placeholder `Overcast soft light with cool blue shadows, golden hour warmth on edges, cinematic film look`)
     - "Scene Type" (dropdown, required, options: Exterior Day, Exterior Night, Interior Day, Interior Night, Golden Hour, Overcast, Storm / Dramatic)
     - "CG Subject Type" (dropdown, required, options: Character / Human, Vehicle, Creature, Hard Surface Object, Environment / Set Extension)
     - "Supervisor Email" (email, not required, placeholder `user@example.com`)

3. **Add Code node: "Validate & Extract Lighting Brief"**
   - Type: Code (v2)
   - Language: JavaScript
   - Paste the validation code that maps dropdown values to short codes (`sceneTypeMap`, `cgSubjectMap`), derives `sequenceCode` from shot code if blank, sets `requestTimestamp`, and throws errors for missing required fields.
   - Connect: Form Trigger → this node.

4. **Add Code node: "Fan-Out: 5 Lighting Variations"**
   - Type: Code (v2)
   - Language: JavaScript
   - Paste the fan-out code that creates the 5 variation objects (`LLD-KEY`, `LLD-SOFT`, `LLD-DRAM`, `LLD-PLATE`, `LLD-COMP`), each with `variantId`, `lightType`, `label`, `lightIcon`, and `safePrompt`.
   - Connect: Validate node → this node.

5. **Add Code node: "Build Lighting Request Body"**
   - Type: Code (v2)
   - Language: JavaScript
   - Paste the code that constructs the Seedance API body with model `seedance-1-5-pro-251215`, content array (text + image_url), `generate_audio: false`, `ratio: '16:9'`, `duration: 5`, `watermark: false`, and stringifies it as `requestBody`.
   - Connect: Fan-Out node → this node.

6. **Add HTTP Request node: "Seedance: Generate Lighting Ref"**
   - Type: HTTP Request (v4.3)
   - Method: POST
   - URL: Set to your Seedance API generations endpoint (e.g., `https://api.seedance.ai/v1/generations` — verify with Seedance docs).
   - Send Body: Yes → Specify Body → JSON
   - JSON Body: `={{ JSON.parse($json.requestBody) }}`
   - Send Headers: Yes
   - Header "Authorization": Value `Bearer YOUR_SEEDANCE_API_KEY` (create an n8n credential of type Header Auth or enter directly for testing — production: use credential).
   - Header "Content-Type": `application/json`
   - Connect: Build Lighting Request Body → this node.

7. **Add Code node: "Merge Lighting Job ID + Metadata"**
   - Type: Code (v2)
   - Language: JavaScript
   - Paste the merge code that reads from `$input.first().json` (API response) and `$('Build Lighting Request Body').first().json` (variant data), combining `id` and `jobStatus` with variant fields.
   - Connect: Seedance HTTP Request → this node.

8. **Add HTTP Request node: "Poll: Check Lighting Job Status"**
   - Type: HTTP Request (v4.3)
   - Method: GET
   - URL: Dynamic expression pointing to the Seedance job status endpoint with the job ID, e.g., `https://api.seedance.ai/v1/generations/{{$json.id}}` (verify exact endpoint with Seedance docs).
   - Send Headers: Yes
   - Header "Authorization": `Bearer YOUR_SEEDANCE_API_KEY` (same as step 6).
   - Connect: Merge node → this node. Also connect Wait 20s → this node (loop).

9. **Add IF node: "Lighting Render Complete?"**
   - Type: IF (v2)
   - Condition: `$json.status` equals `succeeded` (string, case-sensitive, strict type validation).
   - Connect: Poll node → this node.
   - True output → `Build Lighting Asset Metadata` (step 11).
   - False output → `Wait 20s` (step 10).

10. **Add Wait node: "Wait 20s"**
    - Type: Wait (v1.1)
    - Amount: 20 seconds.
    - Connect: IF node (false branch) → this node → Poll node (step 8, completing the loop).

11. **Add Code node: "Build Lighting Asset Metadata"**
    - Type: Code (v2)
    - Language: JavaScript
    - Paste the metadata extraction code that reads the poll result, extracts `videoUrl` (with fallback chain), and builds the full metadata object including `tags`.
    - Connect: IF node (true branch) → this node.

12. **Add HTTP Request node: "Download Lighting Ref Video"**
    - Type: HTTP Request (v4.3)
    - Method: GET
    - URL: `={{ $json.videoUrl }}`
    - Response Format: File
    - Connect: Build Lighting Asset Metadata → this node.

13. **Add Google Drive node: "Google Drive: Upload Lighting Ref"**
    - Type: Google Drive (v3)
    - Operation: Upload
    - File Name: `={{ $json.shotCode }}_lighting_{{ $json.lightType }}_{{ $json.variantId }}.mp4`
    - Drive: My Drive
    - Folder: Select your target folder (or paste folder ID, e.g., `1fL57MP1QWsF0O_n7WuqfzJ0hO6I9Csrh`).
    - Credential: Create a Google Drive OAuth2 credential and connect it.
    - Connect: Download Lighting Ref Video → this node.

14. **Add Code node: "Aggregate + Build Lookbook"**
    - Type: Code (v2)
    - Language: JavaScript
    - Paste the aggregation code that reads all items, builds `lookbookHtml` (full HTML table with variation details), builds `slackLines` (text summary per variation), and returns a single consolidated item.
    - Connect: Google Drive Upload → this node.

15. **Add Notion node: "Record the log in Notion"**
    - Type: Notion (v2.2)
    - Operation: Create Database Page
    - Database: Select your "Lighting & Look Dev Reference DB" (or create a Notion database with a `title` property named `shotCode` and a `rich_text` property named `lookbookHtml`).
    - Property Mappings:
      - `shotCode` (title): `={{ $json.shotCode }}`
      - `lookbookHtml` (rich_text): `={{ $json.lookbookHtml }}`
    - Credential: Create a Notion API credential (internal integration) with write access to the target database.
    - Connect: Aggregate node → this node (parallel with AI node).

16. **Add OpenAI (LangChain) node: "AI - Generate the slack message"**
    - Type: OpenAI (n8n-nodes-langchain) (v1.7)
    - Model: `gpt-4o-mini`
    - Options: Max Tokens 300, Temperature 0.7
    - System Message: "You are a VFX production assistant. Your job is to write a clear, professional Slack message for a lighting team when their AI look development references are ready. Rules: Write in a concise, professional tone. Use Slack markdown formatting (bold with *asterisks*, code with backticks). Include all key details: shot code, mood, scene type, variations ready. Keep it scannable — use line breaks and emoji for visual hierarchy. End with a clear call to action for the team. Never include raw HTML, JSON, or technical URLs in the message. Output ONLY the Slack message text, nothing else."
    - User Message: Template injecting all fields from the Aggregate node (`shotCode`, `sequenceCode`, `projectId`, `sceneType`, `cgSubjectType`, `moodDescription`, `totalVariations`, `generatedAt`, `supervisorEmail`, `slackLines`).
    - Credential: Create an OpenAI API credential.
    - Connect: Aggregate node → this node (parallel with Notion node).

17. **Add Slack node: "Slack: Notify Lighting Team"**
    - Type: Slack (v2.3)
    - Operation: Send message
    - Channel: Select your lighting team channel (or paste channel ID, e.g., `C0ANFAL4WJ2`).
    - Message Text: `={{ $json.message.content }}`
    - Authentication: OAuth2 (create Slack OAuth2 credential with `chat:write` scope and appropriate bot token).
    - Connect: AI - Generate the slack message → this node.

18. **Add Error Trigger node: "On Workflow Error"**
    - Type: Error Trigger (v1)
    - No parameters required.
    - Connect: This node → Slack: Error Alert (step 19).

19. **Add Slack node: "Slack: Error Alert"**
    - Type: Slack (v2.3)
    - Operation: Send message
    - Channel: Same channel ID as step 17 (or a dedicated ops channel).
    - Message Text: `=❌ *Lighting Look Dev Generator Failed*\n\nError: {{ $json.message }}\nTime: {{ new Date().toISOString() }}`
    - Authentication: Same OAuth2 credential as step 17.
    - Connect: On Workflow Error → this node.

20. **Configure credentials (summary):**
    | Credential | Type | Used By |
    |------------|------|---------|
    | Seedance API Key | Header Auth or directly in HTTP Request headers | Seedance: Generate Lighting Ref, Poll: Check Lighting Job Status |
    | Google Drive OAuth2 | googleDriveOAuth2Api | Google Drive: Upload Lighting Ref |
    | Notion API | notionApi (internal integration) | Record the log in Notion |
    | Slack OAuth2 | slackApi (OAuth2) | Slack: Notify Lighting Team, Slack: Error Alert |
    | OpenAI API | openAiApi | AI - Generate the slack message |

21. **Replace all placeholder values before activation:**
    - Set the Seedance API base URL in both HTTP Request nodes.
    - Set `Authorization` header value to `Bearer <your-seedance-key>` in both HTTP Request nodes.
    - Update the Google Drive folder ID to your target folder.
    - Update the Notion database ID and verify property names (`shotCode` as title, `lookbookHtml` as rich_text).
    - Update the Slack channel ID in both Slack nodes.
    - Ensure all OAuth2 credentials are authorised and connected.

22. **Test the workflow:**
    - Activate the form trigger.
    - Submit a test brief with a valid shot code, a publicly accessible plate image URL, and a mood description.
    - Verify that 5 Seedance jobs are submitted.
    - Verify the polling loop detects completion.
    - Verify videos appear in Google Drive with correct naming.
    - Verify a Notion page is created.
    - Verify a Slack message is posted.

23. **Activate the workflow** once all tests pass.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| This workflow was designed for VFX/lighting look development pipelines where a lighting TD needs rapid AI-generated reference variations to guide CG lighting rig setup. | General purpose context |
| The polling loop has **no maximum retry count or timeout cap**. If a Seedance job enters a `failed` or `cancelled` state, the loop will spin indefinitely. Consider adding a retry counter or a "max attempts" check in the IF condition. | Operational risk |
| The `safePrompt` construction uses `JSON.stringify(...).slice(1,-1)` to escape the prompt string. This is a workaround for escaping quotes and special characters but may produce unexpected results with certain Unicode or emoji sequences. | Technical note |
| The Notion node stores `lookbookHtml` as a `rich_text` property. Notion's rich_text does **not** render HTML — it stores plain text. For rendered HTML content, consider using Notion's `bookmark` or `embed` block types, or link to an externally hosted HTML page. | Notion limitation |
| The Seedance model identifier `seedance-1-5-pro-251215` appears version-specific. Verify current model names in the Seedance documentation before deploying. | API versioning |
| The `sceneTypeMap` and `cgSubjectMap` in the Validate node silently default to `ext_day` and `character` for unrecognised values. If the form dropdown is modified, the maps must be updated accordingly. | Data integrity |
| The Google Drive folder ID `1fL57MP1QWsF0O_n7WuqfzJ0hO6I9Csrh` and the cached name "Simulation FX Concept Generator" are template-specific. Update to your production folder before use. | Folder configuration |
| The Slack channel `C0ANFAL4WJ2` (cached as "social") is used for both success and error notifications. In production, consider separating the error alert into a dedicated ops/incident channel. | Best practice |
| The AI-generated Slack message is capped at 300 tokens. With 5 variations each containing a video link, the message may be truncated. Consider raising to 500 tokens or condensing the variation list. | Token limit |
| No authentication is configured on the Download Lighting Ref Video node. If Seedance requires signed URLs or bearer tokens for video downloads, add the appropriate header. | Security gap |
| The HTML lookbook injected into Notion is **not sanitised**. User-provided mood descriptions could contain script tags or other injection payloads. Consider sanitising or escaping the mood description before embedding in HTML. | XSS risk |
| The Error Handler only catches unhandled errors. Nodes that have their own error handling (retry, continue-on-fail) will not trigger the Slack error alert. Review each node's error behaviour settings. | Error handling scope |