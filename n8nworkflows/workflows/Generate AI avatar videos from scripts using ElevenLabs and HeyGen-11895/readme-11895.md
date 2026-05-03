Generate AI avatar videos from scripts using ElevenLabs and HeyGen

https://n8nworkflows.xyz/workflows/generate-ai-avatar-videos-from-scripts-using-elevenlabs-and-heygen-11895


# Generate AI avatar videos from scripts using ElevenLabs and HeyGen

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow exposes a webhook endpoint that receives a script (text) and parameters, generates voice audio using **ElevenLabs**, uploads that audio to **HeyGen**, creates an **AI avatar video** in HeyGen, polls HeyGen until the video is complete, downloads the resulting video, uploads it to **Google Drive**, then returns a final response to the webhook caller (with the deliverable fetched back from Drive).

**Target use cases**
- Turn short scripts into narrated avatar videos for marketing, internal comms, onboarding, social content.
- Provide an API-like endpoint for other systems (forms, chatbots, CMS) to request video generation and receive a result.

### Logical blocks
**1.1 Request Intake & Normalization**
- Webhook receives request, Set node extracts/normalizes parameters.

**1.2 Audio Generation (ElevenLabs)**
- HTTP request produces TTS audio from script.

**1.3 HeyGen Ingestion & Video Creation**
- Upload audio to HeyGen, then create avatar video referencing that uploaded audio.

**1.4 Polling / Status Tracking**
- Extract video ID, wait, poll status, loop until completed.

**1.5 Retrieval, Storage, and Response**
- Download finished video, upload to Google Drive, optionally download from Drive, respond to webhook.

**1.6 Error Handling / Alerting**
- Global error trigger sends Slack message.

---

## 2. Block-by-Block Analysis

### 2.1 Request Intake & Normalization
**Overview:** Accepts incoming HTTP requests and shapes them into the fields needed for subsequent API calls (script text, voice/avatar IDs, output naming, etc.).

**Nodes involved**
- **Webhook - Receive Request**
- **Parse Request Parameters**
- (Sticky notes: **Sticky Note**, **Sticky Note1** — content empty)

#### Node: Webhook - Receive Request
- **Type / role:** `Webhook` — workflow entry point (HTTP endpoint).
- **Config (interpreted):**
  - Uses an internal `webhookId` (`generate-video-webhook`).
  - HTTP method/path are not present in the JSON excerpt, so they must be verified in the node UI.
- **Inputs/outputs:** No input; outputs to **Parse Request Parameters**.
- **Failure modes / edge cases:**
  - Missing required fields in request body/query (downstream expression failures).
  - Large payloads/timeouts if caller waits synchronously while HeyGen renders (this workflow *does* poll and only responds at the end).
  - Authentication not enforced unless configured in webhook settings.

#### Node: Parse Request Parameters
- **Type / role:** `Set` — maps incoming request data into canonical fields.
- **Config (interpreted):**
  - The JSON does not show which fields are set; typically this node:
    - extracts `script` / `text`
    - selects `voiceId` (ElevenLabs) and avatar/template params (HeyGen)
    - sets filenames and metadata for Drive upload/response
- **Inputs/outputs:** Input from **Webhook - Receive Request**, output to **Generate Audio with ElevenLabs**.
- **Failure modes / edge cases:**
  - If using expressions like `{{$json.body.script}}`, missing keys will result in empty/invalid API calls.
  - Ensure content-type parsing matches caller (JSON body vs query params vs form-data).

---

### 2.2 Audio Generation (ElevenLabs)
**Overview:** Calls ElevenLabs API to generate speech audio from the provided script.

**Nodes involved**
- **Generate Audio with ElevenLabs**
- (Sticky note: **Sticky Note2** — content empty)

#### Node: Generate Audio with ElevenLabs
- **Type / role:** `HTTP Request` — ElevenLabs Text-to-Speech API call.
- **Config (interpreted):**
  - Likely a `POST` request to an ElevenLabs TTS endpoint with:
    - text/script from prior Set node
    - voice ID/model settings
  - Response likely returns audio bytes (binary) or a URL depending on API settings.
- **Inputs/outputs:** Input from **Parse Request Parameters**, output to **Upload Audio to HeyGen**.
- **Failure modes / edge cases:**
  - 401/403 if API key missing/invalid.
  - 429 rate limiting (especially on batch usage).
  - If audio is returned as binary, downstream node must handle binary property name consistently.
  - Script length limits (ElevenLabs may reject overly long text).

---

### 2.3 HeyGen Ingestion & Video Creation
**Overview:** Uploads the generated audio to HeyGen and requests HeyGen to render an avatar video using that audio.

**Nodes involved**
- **Upload Audio to HeyGen**
- **Create Avatar Video with Audio**
- **Extract Video ID**
- (Sticky note: **Sticky Note3** — content empty)

#### Node: Upload Audio to HeyGen
- **Type / role:** `HTTP Request` — sends audio to HeyGen (typically multipart/form-data).
- **Config (interpreted):**
  - Likely `POST` to a HeyGen endpoint that accepts audio uploads and returns an `audio_id` or asset URL.
  - Must include HeyGen API key/authorization header.
- **Inputs/outputs:** Input from **Generate Audio with ElevenLabs**, output to **Create Avatar Video with Audio**.
- **Failure modes / edge cases:**
  - Incorrect multipart configuration (audio not actually attached).
  - Binary property mismatch (node expects `data` but previous node outputs `audio`, etc.).
  - 413 if audio too large.

#### Node: Create Avatar Video with Audio
- **Type / role:** `HTTP Request` — requests HeyGen video generation.
- **Config (interpreted):**
  - Likely `POST` with payload referencing:
    - uploaded audio reference (`audio_id` / URL)
    - avatar selection / template / background
    - resolution/aspect ratio
- **Inputs/outputs:** Input from **Upload Audio to HeyGen**, output to **Extract Video ID**.
- **Failure modes / edge cases:**
  - Missing required HeyGen parameters (avatar/template not set).
  - 401/403 if HeyGen credentials invalid.
  - 400 if audio reference invalid or not yet processed.

#### Node: Extract Video ID
- **Type / role:** `Set` — extracts the created `video_id` (or equivalent) from HeyGen response.
- **Config (interpreted):**
  - Likely sets a field like `videoId = $json.data.video_id` (exact expression not shown).
- **Inputs/outputs:** Input from **Create Avatar Video with Audio**, output to **Wait 5 Seconds**.
- **Failure modes / edge cases:**
  - If HeyGen response shape changes, extraction returns null and polling fails.

---

### 2.4 Polling / Status Tracking
**Overview:** Waits briefly, polls HeyGen for video render status, loops until the status indicates completion.

**Nodes involved**
- **Wait 5 Seconds**
- **Poll Video Status**
- **Check If Completed**
- (Sticky note: **Sticky Note4** — content empty)

#### Node: Wait 5 Seconds
- **Type / role:** `Wait` — delays before status polling.
- **Config (interpreted):**
  - Default wait configuration (not shown). Node name indicates 5 seconds.
  - Contains a `webhookId` named `video-status-poll` (used by Wait node’s resume mechanism).
- **Inputs/outputs:** Input from **Extract Video ID** and from IF “not completed” loop; output to **Poll Video Status**.
- **Failure modes / edge cases:**
  - Very long render times can cause many loops; consider maximum attempts.
  - Wait/resume requires n8n to be able to persist executions (important on ephemeral setups).

#### Node: Poll Video Status
- **Type / role:** `HTTP Request` — queries HeyGen for render status.
- **Config (interpreted):**
  - Likely `GET`/`POST` to status endpoint using `videoId`.
- **Inputs/outputs:** Input from **Wait 5 Seconds**, output to **Check If Completed**.
- **Failure modes / edge cases:**
  - Intermittent HeyGen errors; should consider retry/backoff.
  - If `videoId` missing/invalid -> 4xx responses.

#### Node: Check If Completed
- **Type / role:** `IF` — branching logic.
- **Config (interpreted):**
  - Condition checks HeyGen status field (e.g., `completed`, `done`, `success`) and routes:
    - **True (completed)** → **Extract Video Details**
    - **False (not completed)** → **Wait 5 Seconds** (loop)
  - Exact condition not visible in JSON (node parameters empty in excerpt).
- **Inputs/outputs:** Input from **Poll Video Status**, outputs to both branches as above.
- **Failure modes / edge cases:**
  - If HeyGen returns unexpected status strings, condition may never become true (infinite loop risk).
  - If error status exists (failed/canceled), workflow should ideally exit with an error message; not shown here.

---

### 2.5 Retrieval, Storage, and Response
**Overview:** Once complete, extracts the final video URL/details, downloads the video, uploads it to Google Drive, then returns a response to the webhook caller.

**Nodes involved**
- **Extract Video Details**
- **Download Video**
- **Upload to Google Drive**
- **Download from Drive**
- **Respond to Webhook**
- (Sticky note: **Sticky Note5** — content empty)

#### Node: Extract Video Details
- **Type / role:** `Set` — captures download URL, filename, metadata from poll response.
- **Config (interpreted):**
  - Likely sets:
    - `videoUrl` / `download_url`
    - output filename (maybe includes timestamp/request id)
- **Inputs/outputs:** Input from **Check If Completed** (true branch), output to **Download Video**.
- **Failure modes / edge cases:**
  - Missing `video_url` if HeyGen returns nested structures or requires another endpoint.

#### Node: Download Video
- **Type / role:** `HTTP Request` — downloads the finished video binary.
- **Config (interpreted):**
  - Uses `videoUrl` from previous node.
  - Must be configured to return binary data.
- **Inputs/outputs:** Input from **Extract Video Details**, output to **Upload to Google Drive**.
- **Failure modes / edge cases:**
  - Large video downloads can hit memory/time limits.
  - 403 if the video URL is signed/expired by the time it is fetched.

#### Node: Upload to Google Drive
- **Type / role:** `Google Drive` — uploads binary video to Drive.
- **Config (interpreted):**
  - Likely “Upload” operation with:
    - target folder
    - file name from Set node
    - binary property containing the video
- **Credentials:** Google Drive OAuth2 (or service account if supported by your n8n setup).
- **Inputs/outputs:** Input from **Download Video**, output to **Download from Drive**.
- **Failure modes / edge cases:**
  - OAuth token expired/insufficient scopes.
  - Folder ID wrong / no permissions.
  - File size limits (depending on account/API constraints).

#### Node: Download from Drive
- **Type / role:** `Google Drive` — downloads the uploaded file (often to produce a stable binary/preview link for response).
- **Config (interpreted):**
  - Likely “Download” operation using the file ID returned by upload.
- **Inputs/outputs:** Input from **Upload to Google Drive**, output to **Respond to Webhook**.
- **Failure modes / edge cases:**
  - If upload returns unexpected fields, download may not find file ID.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` — final HTTP response to caller.
- **Config (interpreted):**
  - Sends back success payload (possibly includes Drive link, file id, and/or binary).
  - Parameters not visible in excerpt; verify if it returns JSON vs binary file.
- **Inputs/outputs:** Input from **Download from Drive**; ends workflow.
- **Failure modes / edge cases:**
  - If response is binary, must set correct headers.
  - If caller expects quick response, this synchronous polling design may time out at the caller/proxy.

---

### 2.6 Error Handling / Alerting
**Overview:** Catches workflow errors globally and sends a Slack alert.

**Nodes involved**
- **Error Handler Trigger**
- **Send a message**
- (Sticky note: **Sticky Note7** — content empty)

#### Node: Error Handler Trigger
- **Type / role:** `Error Trigger` — runs when the workflow execution errors.
- **Config (interpreted):**
  - Triggers on workflow errors; passes error context to downstream nodes.
- **Inputs/outputs:** No input; output to **Send a message**.
- **Failure modes / edge cases:**
  - If the Slack node fails too, you may lose visibility; consider adding alternative logging.

#### Node: Send a message
- **Type / role:** `Slack` — posts error notification.
- **Config (interpreted):**
  - Uses Slack credentials (or webhook-based auth depending on configuration).
  - Node JSON includes a `webhookId` (internal), not Slack’s incoming webhook URL.
- **Inputs/outputs:** Input from **Error Handler Trigger**, ends branch.
- **Failure modes / edge cases:**
  - Slack auth errors, missing channel, insufficient scopes.
  - Message formatting issues if referencing missing error fields.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Comment container | — | — |  |
| Sticky Note1 | Sticky Note | Comment container | — | — |  |
| Sticky Note2 | Sticky Note | Comment container | — | — |  |
| Sticky Note3 | Sticky Note | Comment container | — | — |  |
| Sticky Note4 | Sticky Note | Comment container | — | — |  |
| Sticky Note5 | Sticky Note | Comment container | — | — |  |
| Webhook - Receive Request | Webhook | Entry point (HTTP request) | — | Parse Request Parameters |  |
| Parse Request Parameters | Set | Normalize request fields | Webhook - Receive Request | Generate Audio with ElevenLabs |  |
| Generate Audio with ElevenLabs | HTTP Request | TTS audio generation | Parse Request Parameters | Upload Audio to HeyGen |  |
| Upload Audio to HeyGen | HTTP Request | Upload audio asset to HeyGen | Generate Audio with ElevenLabs | Create Avatar Video with Audio |  |
| Create Avatar Video with Audio | HTTP Request | Start HeyGen video render | Upload Audio to HeyGen | Extract Video ID |  |
| Extract Video ID | Set | Extract `videoId` from render response | Create Avatar Video with Audio | Wait 5 Seconds |  |
| Wait 5 Seconds | Wait | Delay between polls | Extract Video ID; Check If Completed (false) | Poll Video Status |  |
| Poll Video Status | HTTP Request | Query HeyGen render status | Wait 5 Seconds | Check If Completed |  |
| Check If Completed | IF | Branch: completed vs keep polling | Poll Video Status | Extract Video Details (true); Wait 5 Seconds (false) |  |
| Extract Video Details | Set | Extract download URL/metadata | Check If Completed (true) | Download Video |  |
| Download Video | HTTP Request | Download final video binary | Extract Video Details | Upload to Google Drive |  |
| Upload to Google Drive | Google Drive | Store video file | Download Video | Download from Drive |  |
| Download from Drive | Google Drive | Retrieve file (binary/link) for response | Upload to Google Drive | Respond to Webhook |  |
| Respond to Webhook | Respond to Webhook | Return final result to caller | Download from Drive | — |  |
| Sticky Note7 | Sticky Note | Comment container | — | — |  |
| Error Handler Trigger | Error Trigger | Global error entry point | — | Send a message |  |
| Send a message | Slack | Notify on failure | Error Handler Trigger | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: *Generate AI avatar videos from scripts using ElevenLabs and HeyGen to Google Drive* (or your preferred name).
- Ensure **Execution Order** is `v1` (Workflow settings).

2) **Add Webhook node**
- Node: **Webhook**
- Name: `Webhook - Receive Request`
- Configure:
  - Path (e.g. `/generate-video`) and method (typically `POST`)
  - Response mode: *Using “Respond to Webhook” node* (so it waits for final output)
  - (Optional) Authentication (recommended)
- Connect to next node.

3) **Add Set node to parse/normalize input**
- Node: **Set**
- Name: `Parse Request Parameters`
- Configure fields you need, typically:
  - `scriptText` from request body (e.g. `{{$json.body.script}}`)
  - `elevenlabsVoiceId` (from request or constant)
  - HeyGen avatar/template identifiers (constant or request-provided)
  - `outputFileName` (e.g. `avatar-video-{{$now}}.mp4`)
- Connect: `Webhook - Receive Request` → `Parse Request Parameters`.

4) **Add HTTP Request to ElevenLabs (TTS)**
- Node: **HTTP Request**
- Name: `Generate Audio with ElevenLabs`
- Configure (typical setup):
  - Method: `POST`
  - URL: ElevenLabs TTS endpoint for your plan/region
  - Authentication: Header `xi-api-key: <YOUR_ELEVENLABS_API_KEY>` (or n8n credential if you encapsulate it)
  - Body: JSON containing the text and voice settings
  - Response: **File/Binary** (audio)
- Connect: `Parse Request Parameters` → `Generate Audio with ElevenLabs`.

5) **Add HTTP Request to upload audio to HeyGen**
- Node: **HTTP Request**
- Name: `Upload Audio to HeyGen`
- Configure:
  - Method: `POST`
  - URL: HeyGen audio upload endpoint
  - Auth header: `Authorization: Bearer <HEYGEN_API_KEY>` (or per HeyGen’s required header)
  - Send **multipart/form-data** with the audio binary from ElevenLabs (select the correct binary property)
- Connect: `Generate Audio with ElevenLabs` → `Upload Audio to HeyGen`.

6) **Add HTTP Request to create avatar video**
- Node: **HTTP Request**
- Name: `Create Avatar Video with Audio`
- Configure:
  - Method: `POST`
  - URL: HeyGen “create video” endpoint
  - JSON body referencing the uploaded audio (`audio_id` / URL) plus avatar/template options
- Connect: `Upload Audio to HeyGen` → `Create Avatar Video with Audio`.

7) **Extract the HeyGen video ID**
- Node: **Set**
- Name: `Extract Video ID`
- Configure:
  - Add field `videoId` from HeyGen response (e.g. `{{$json.data.video_id}}` depending on actual response)
- Connect: `Create Avatar Video with Audio` → `Extract Video ID`.

8) **Add Wait node**
- Node: **Wait**
- Name: `Wait 5 Seconds`
- Configure:
  - Wait time: 5 seconds (or equivalent)
- Connect: `Extract Video ID` → `Wait 5 Seconds`.

9) **Add HTTP Request to poll video status**
- Node: **HTTP Request**
- Name: `Poll Video Status`
- Configure:
  - Method: `GET` or `POST` per HeyGen API
  - Include `videoId` in URL params/body
- Connect: `Wait 5 Seconds` → `Poll Video Status`.

10) **Add IF node to check completion**
- Node: **IF**
- Name: `Check If Completed`
- Configure condition based on HeyGen response, e.g.:
  - `{{$json.data.status}}` equals `completed` (adjust to HeyGen’s actual status)
- Connect: `Poll Video Status` → `Check If Completed`.
- Wire **False** branch back to **Wait 5 Seconds** to loop.

11) **Extract final download details**
- Node: **Set**
- Name: `Extract Video Details`
- Configure fields like:
  - `videoUrl` from status response (e.g. `{{$json.data.video_url}}`)
  - `outputFileName` if not already set
- Connect: `Check If Completed (true)` → `Extract Video Details`.

12) **Download the video**
- Node: **HTTP Request**
- Name: `Download Video`
- Configure:
  - Method: `GET`
  - URL: `{{$json.videoUrl}}`
  - Response: **File/Binary**
- Connect: `Extract Video Details` → `Download Video`.

13) **Upload to Google Drive**
- Node: **Google Drive**
- Name: `Upload to Google Drive`
- Credentials: Configure Google OAuth2 for Drive with required scopes.
- Operation: Upload
- Configure:
  - Folder: choose target folder
  - File name: `{{$json.outputFileName}}`
  - Binary property: the binary from `Download Video`
- Connect: `Download Video` → `Upload to Google Drive`.

14) **(Optional) Download from Drive**
- Node: **Google Drive**
- Name: `Download from Drive`
- Operation: Download
- Configure:
  - File ID from upload output
- Connect: `Upload to Google Drive` → `Download from Drive`.

15) **Respond to the webhook**
- Node: **Respond to Webhook**
- Name: `Respond to Webhook`
- Configure response:
  - JSON including Drive file ID/link, or return binary video (depending on your need)
- Connect: `Download from Drive` → `Respond to Webhook`.

16) **Add error alerting**
- Node: **Error Trigger**
- Name: `Error Handler Trigger`
- Add Slack node:
  - Node: **Slack**
  - Name: `Send a message`
  - Configure Slack credentials and channel
  - Message content referencing error fields (e.g. execution ID, node name, error message)
- Connect: `Error Handler Trigger` → `Send a message`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but have empty content in the provided workflow JSON. | If you expected comments/links, they were not included in the export or were left blank. |
| The workflow is synchronous (webhook waits until rendering completes). Consider caller/proxy timeouts; for production, you may prefer an async pattern (respond immediately with job ID, then callback/poll externally). | Architectural consideration for reliability/scalability. |
| Add a maximum polling attempt counter to avoid infinite loops if HeyGen returns unexpected statuses or fails silently. | Operational hardening. |