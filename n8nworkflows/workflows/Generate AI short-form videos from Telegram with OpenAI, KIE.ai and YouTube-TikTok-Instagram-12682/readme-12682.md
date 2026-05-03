Generate AI short-form videos from Telegram with OpenAI, KIE.ai and YouTube/TikTok/Instagram

https://n8nworkflows.xyz/workflows/generate-ai-short-form-videos-from-telegram-with-openai--kie-ai-and-youtube-tiktok-instagram-12682


# Generate AI short-form videos from Telegram with OpenAI, KIE.ai and YouTube/TikTok/Instagram

## 1. Workflow Overview

This n8n workflow turns Telegram messages (text and optionally images) into AI-generated short-form vertical videos. It uses OpenAI to enhance prompts and produce metadata, KIE.ai to generate (and optionally extend) the video, stores outputs in S3, and provides one-tap publishing actions to YouTube Shorts, TikTok, and Instagram Reels (via Late.dev). Session state is stored in Redis so users can interact via Telegram inline buttons (model selection, publish, extend, merge, cancel).

### 1.1 Telegram Input Reception & Routing
Receives Telegram messages and callback queries, classifies the intent (new prompt, choose model, publish, extend, merge, help, cancel), then routes to the appropriate handler branch.

### 1.2 Prompt Generation & Session Creation (OpenAI + Redis)
Extracts user input (text and optional photo), generates a structured JSON output from OpenAI (video prompt + YouTube metadata), formats it into a session object, stores it in Redis, then asks the user to pick a video model via inline buttons.

### 1.3 Video Generation (KIE.ai + Polling Trigger)
When the user selects a model, the workflow checks KIE.ai credit balance, starts a generation job, stores the task ID in Redis, updates the Telegram message, and triggers an external polling webhook (`/webhook/video-poll`) to monitor completion.

### 1.4 Publishing to YouTube Shorts (S3 -> YouTube)
On “YouTube” publish callback, downloads the video from S3 using `sessionId.mp4`, uploads to YouTube with the generated metadata, updates Redis, and notifies the user with the Shorts URL.

### 1.5 Publishing to TikTok / Instagram / All (KIE status -> Late.dev)
On publish callbacks for TikTok/Instagram/All, it retrieves the session, fetches the final video URL from KIE.ai’s status endpoint, generates a platform-ready caption with hashtags, and publishes via Late.dev.

### 1.6 Extend Video (Veo only) + Polling Trigger
If the user taps “Extend”, the workflow validates the original session/task/model, checks credits, uses OpenAI to generate a continuation prompt, starts a KIE.ai “extend” job, stores a new “extension session”, and triggers the polling webhook.

### 1.7 Merge Extended Video (Auto via Transloadit or Manual Links)
For extended videos, the user can:
- Auto-merge: fetch original + extended URLs, create a Transloadit concat assembly, poll until complete, upload merged output to S3, create a merged session, and send the merged video back to Telegram with publish buttons.
- Manual merge: send both URLs plus suggested tools (CapCut/Canva/ClipChamp).

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Input Reception & Routing
**Overview:** Receives all Telegram bot updates (messages and button callbacks), derives a `routeType`, then fans out into dedicated subflows.

**Nodes involved:**  
- Telegram Trigger  
- Determine Route (Code)  
- Route Input (Switch)  
- Telegram Input (Sticky Note)

#### Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point; listens to Telegram webhook updates.
- **Config choices:** Subscribed to update types `message` and `callback_query`.
- **Key variables:** Outputs Telegram update JSON (message or callback_query).
- **Connections:** → Determine Route.
- **Failure modes:** Telegram credential issues; webhook misconfigured; bot not reachable.

#### Determine Route
- **Type / role:** `code` — classifies the input into an internal action.
- **Logic highlights:**
  - For new messages: extracts `text` or `caption`, detects slash command templates (`/general`, `/lost`, `/3d`, `/story`, `/help`), sets:
    - `routeType = 'new_message'` or `'help'`
    - `template` (general/lost/3d/story)
    - `userPrompt` (text after command)
  - For callback queries: parses `callback_query.data` prefixes:
    - `model:` → `generate_with_model`
    - `publish:` → `publish`
    - `tiktok:` → `tiktok`
    - `instagram:` → `instagram`
    - `publishall:` → `publishall`
    - `extend:` → `extend`
    - `cancel:` → `cancel`
    - `am:` → `automerge`
    - `mm:` → `manualmerge`
- **Outputs:** Passes through original update JSON + `routeType/template/userPrompt`.
- **Connections:** → Route Input.
- **Failure modes:** Unexpected Telegram payload shape; callback data missing/invalid.

#### Route Input
- **Type / role:** `switch` — routes by `{{$json.routeType}}`.
- **Outputs (named):** NewMessage, GenerateWithModel, Publish, Cancel, Help, Extend, TikTok, Instagram, PublishAll, AutoMerge, ManualMerge; fallback “none”.
- **Connections:** Sends to the corresponding “Parse …” or “Extract Input” branch.
- **Failure modes:** If `routeType` is “unknown” → goes to fallback and is effectively dropped (no handler).

---

### Block 2 — Prompt Generation & Session Storage (Text/Image -> OpenAI -> Redis)
**Overview:** Builds a `sessionId`, optionally obtains a Telegram photo URL, calls OpenAI for JSON metadata + prompt, stores it in Redis, then asks the user to choose a KIE.ai model.

**Nodes involved:**  
- Extract Input (Code)  
- Send Initial Status (Telegram)  
- Has Photo? (IF)  
- Get Photo URL (Telegram)  
- Build Photo URL (Code)  
- No Photo (Code)  
- Merge (Merge)  
- Send Prompt Status (Telegram)  
- Generate Prompt & Metadata (HTTP Request)  
- Format Session (Code)  
- Store Session (Redis)  
- Send Prompt for Approval (Telegram)  
- Prompt Generation (Sticky Note)

#### Extract Input
- **Type / role:** `code` — normalizes Telegram message into workflow fields.
- **Key fields created:**
  - `sessionId = chatId_messageId_timestamp`
  - `chatId`, `messageId`
  - `textInput` (from message text/caption or `userPrompt`)
  - `photoFileId` (largest photo if any)
  - `hasPhoto` boolean
  - `template` from Determine Route
- **Connections:** → Has Photo? and Send Initial Status (parallel fan-out via two connections).
- **Failure modes:** Missing `message` object (shouldn’t happen on new_message route).

#### Send Initial Status
- **Type / role:** `telegram` — sends “Working on it”.
- **Config:** Markdown parse mode.
- **Failure modes:** Telegram send errors; chat blocked bot.

#### Has Photo?
- **Type / role:** `if` — checks `{{$json.hasPhoto}}`.
- **True:** → Get Photo URL  
- **False:** → No Photo

#### Get Photo URL
- **Type / role:** `telegram` (resource: file) — fetches Telegram file info from `photoFileId`.
- **Output:** Contains `file_path` (Telegram internal path).
- **Failure modes:** file_id invalid/expired; Telegram API errors.

#### Build Photo URL
- **Type / role:** `code` — constructs a public download URL to the Telegram-hosted image.
- **Key implementation detail:** Uses a **hardcoded bot token**:
  - `const botToken = '8209006204:…'`
  - `photoUrl = https://api.telegram.org/file/bot${botToken}/${file_path}`
- **Connections:** → Merge
- **Failure modes / security risk:** Hardcoded token leaks credentials if workflow exported; token rotation required; breaks if token changes.

#### No Photo
- **Type / role:** `code` — passes session data with `photoUrl: null`.
- **Connections:** → Merge

#### Merge
- **Type / role:** `merge` — merges “photo branch” and “no photo branch” back into one stream.
- **Connections:** → Send Prompt Status

#### Send Prompt Status
- **Type / role:** `telegram` — informs user prompt crafting is in progress.
- **Connections:** → Generate Prompt & Metadata

#### Generate Prompt & Metadata
- **Type / role:** `httpRequest` — calls OpenAI Chat Completions.
- **Endpoint:** `POST https://api.openai.com/v1/chat/completions`
- **Auth:** Header Auth credential (“OpenAI - Header Auth”).
- **Model:** `gpt-4o`
- **Response format:** `json_object` enforced.
- **Template logic (important):**
  - Reads `template` and `userPrompt` from `$('Determine Route').first().json`
  - Cleans commands from the user prompt (`/story`, `/lost`, `/3d`)
  - Uses one of 4 system prompts: `lost`, `story`, `3d`, `general`
  - If photo exists: uses “multimodal” content array with `image_url`.
- **Expected output JSON keys:** `videoPrompt`, `title`, `description`, `tags`, `shortSummary`
- **Failure modes:**
  - OpenAI auth/quotas
  - Model refusing content
  - Returned content not valid JSON (even though `response_format` reduces this risk)
  - Image URL inaccessible to OpenAI (Telegram file URL may require no auth; typically ok)

#### Format Session
- **Type / role:** `code` — parses OpenAI JSON and builds canonical session object.
- **Key fields:**
  - `sessionId/chatId/messageId/template/originalInput`
  - `generatedPrompt`
  - `youtubeMetadata {title, description, tags}`
  - `shortSummary`
  - `status: 'prompt_ready'`
- **Failure modes:** JSON.parse error → throws explicit error.

#### Store Session
- **Type / role:** `redis` — persists session under `session:{sessionId}`.
- **TTL:** 86400 seconds (1 day).
- **Failure modes:** Redis connectivity/auth; serialization issues.

#### Send Prompt for Approval
- **Type / role:** `telegram` — displays prompt + metadata and model selection buttons.
- **Inline buttons:** callback_data formats:
  - `model:veo3_fast:{sessionId}`
  - `model:veo3:{sessionId}`
  - `model:sora2:{sessionId}`
  - `model:seedance:{sessionId}`
  - `cancel:{sessionId}`
- **Failure modes:** Message too long (Telegram message length limits); Markdown formatting issues.

---

### Block 3 — Video Generation (Model selection -> Balance -> KIE job -> Polling)
**Overview:** Handles the model selection callback, loads the session, checks KIE credits, starts a generation job on the correct endpoint, stores task details in Redis, edits Telegram status, and triggers an external polling workflow.

**Nodes involved:**  
- Parse Model Selection (Code)  
- Answer Generate (Telegram)  
- Get Session (Redis)  
- Parse Session (Code)  
- Check Balance (HTTP Request)  
- Validate Balance (Code)  
- Balance OK? (IF)  
- Send Insufficient Balance (Telegram)  
- Start Video Generation (HTTP Request)  
- Add Task ID (Code)  
- Store Generating Session (Redis)  
- Send Generating Status (Telegram)  
- Trigger Polling (HTTP Request)  
- Video Generation (Sticky Note)

#### Parse Model Selection
- **Type / role:** `code` — parses callback `model:modelName:sessionId`.
- **Model mapping:**
  - `veo3_fast` → type `veo`, 60 credits
  - `veo3` → type `veo`, 250 credits
  - `sora2` → type `market`, 30 credits
  - `seedance` → type `market`, 84 credits
- **Outputs:** sessionId, selectedModel, modelType, modelName, estimatedCredits, callback chat/message IDs, callbackQueryId.
- **Failure modes:** malformed callback; unknown model falls back to veo3_fast.

#### Answer Generate
- **Type / role:** `telegram` callback answer — acknowledges the button tap.
- **Failure modes:** Telegram callback answer timeout if workflow slow before answering (this node is early, good).

#### Get Session
- **Type / role:** `redis get` — reads `session:{sessionId}`.
- **Failure modes:** missing session (TTL expired or deleted).

#### Parse Session
- **Type / role:** `code` — parses Redis result, merges session with model selection data, sets `status: 'generating'`.
- **Implementation note:** Redis node returns in `propertyName` field.
- **Failure modes:** session JSON invalid; null session.

#### Check Balance
- **Type / role:** `httpRequest` — calls KIE credit endpoint.
- **Endpoint:** `GET https://api.kie.ai/api/v1/chat/credit`
- **Auth:** Header Auth (“KieAI - Header Auth”).
- **Failure modes:** auth, rate limit, API downtime.

#### Validate Balance
- **Type / role:** `code` — compares balance with `estimatedCredits`.
- **Outputs:** `hasEnough`, `shortfall`, `requiredCredits`.
- **Failure modes:** KIE response shape changes (`data` not numeric).

#### Balance OK?
- **Type / role:** `if` — branches on `hasEnough`.
- **False branch:** → Send Insufficient Balance
- **True branch:** → Start Video Generation

#### Send Insufficient Balance
- **Type / role:** `telegram` edit message — updates the original Telegram message with balance error.
- **Failure modes:** editing wrong/old message id; message no longer editable.

#### Start Video Generation
- **Type / role:** `httpRequest` — starts a KIE generation job.
- **Endpoint selection (expression):**
  - Veo: `https://api.kie.ai/api/v1/veo/generate`
  - Market: `https://api.kie.ai/api/v1/jobs/createTask`
- **Body selection:**
  - Veo: `{ prompt, model, generationType: TEXT_2_VIDEO, aspectRatio: 9:16, enableTranslation: true }`
  - Sora2: `{ model: 'sora-2-text-to-video', input: { prompt, aspect_ratio: 'portrait' } }`
  - Seedance: `{ model: 'bytedance/seedance-1.5-pro', input: { prompt: JSON.stringify(prompt), aspect_ratio: '9:16', resolution:'720p', duration:'12', ... } }`
- **Failure modes:** KIE errors; prompt rejected; body mismatch per model.

#### Add Task ID
- **Type / role:** `code` — validates KIE response, sets `taskId`, decides which status endpoint to use later:
  - `veo` → `/api/v1/veo/record-info`
  - `market` → `/api/v1/jobs/recordInfo`
- **Also sets:** `pollAttempt:0`, `balanceAtGeneration`.
- **Failure modes:** KIE returns `code != 200` → throws; missing `data.taskId`.

#### Store Generating Session
- **Type / role:** `redis set` — saves updated session with taskId.
- **TTL:** 1 day.
- **Failure modes:** Redis outage.

#### Send Generating Status
- **Type / role:** `telegram` edit message — updates the original model-selection message to “Generation started”.
- **Failure modes:** message edit fails; Markdown formatting issues.

#### Trigger Polling
- **Type / role:** `httpRequest` — calls an external webhook to start polling.
- **URL:** `POST https://n8n.mosaabyassir.xyz/webhook/video-poll`
- **Body includes:** sessionId, taskId, chatId, messageId, selectedModel, modelType, statusEndpoint, attempt.
- **External dependency:** This workflow expects a separate “polling workflow” at that endpoint to:
  - Poll KIE status
  - Store final video in S3 as `videos/{sessionId}.mp4`
  - Update Redis session with `videoUrl`/`s3Url`/etc.
  - Send final Telegram message with publish/extend/merge actions
- **Failure modes:** webhook unreachable; polling workflow not deployed; schema mismatch.

---

### Block 4 — YouTube Publishing (S3 download -> YouTube upload)
**Overview:** On “YouTube” callback, fetches the stored video from S3 (by sessionId), uploads it to YouTube Shorts, updates Redis, and informs the user.

**Nodes involved:**  
- Parse Publish (Code)  
- Answer Publish (Telegram)  
- Get Session (Publish) (Redis)  
- Parse Session (Publish) (Code)  
- Download Video (S3)  
- Upload to YouTube (YouTube)  
- Format Results (Code)  
- Update Session (Redis)  
- Send Success (Telegram)  
- YouTube Publishing (Sticky Note)

#### Parse Publish
- **Type / role:** `code` — extracts `sessionId` from `publish:{sessionId}`.
- **Failure modes:** malformed callback.

#### Answer Publish
- **Type / role:** `telegram` callback answer.

#### Get Session (Publish) / Parse Session (Publish)
- **Role:** loads session, attaches callback chat/message IDs.

#### Download Video
- **Type / role:** `s3` download — fetches binary file.
- **Bucket:** `shorts`
- **Key:** `videos/{sessionId}.mp4`
- **Failure modes:** object missing (polling didn’t upload); wrong bucket/key; S3 credentials.

#### Upload to YouTube
- **Type / role:** `youTube` upload.
- **Credential:** OAuth2 (“YouTube account”).
- **Metadata:**
  - `title` from session
  - `tags` joined as CSV
  - `description` constructed as: tags + separator + session description
- **Options:** public, language en, notifySubscribers true, categoryId 23, region US.
- **Failure modes:** OAuth expired; channel restrictions; quota limits; upload size/timeouts.

#### Format Results
- **Type / role:** `code` — extracts `uploadId` as videoId, builds Shorts URL `https://youtube.com/shorts/{id}`, sets status published.

#### Update Session
- **Type / role:** `redis set` — stores updated session.
- **TTL:** 604800 seconds (7 days).

#### Send Success
- **Type / role:** `telegram` — sends confirmation + URL.

---

### Block 5 — Help & Cancel
**Overview:** Provides a command help message and allows cancelling by deleting the session and editing the message.

**Nodes involved:**  
- Send Help (Telegram)  
- Parse Cancel (Code)  
- Answer Cancel (Telegram)  
- Delete Session (Redis)  
- Send Cancel (Telegram)

#### Send Help
- **Type / role:** Telegram message with usage examples and supported commands.

#### Cancel flow
- **Parse Cancel:** parses `cancel:{sessionId}`
- **Answer Cancel:** callback answer
- **Delete Session:** Redis delete `session:{sessionId}`
- **Send Cancel:** edits the original message to “Cancelled”

**Failure modes:** session already expired; edit message fails.

---

### Block 6 — Extend Video (Veo only)
**Overview:** Extends Veo-generated videos by ~8 seconds. Validates it’s a Veo session with a taskId, checks credits, generates a continuation prompt with OpenAI, starts KIE extend job, stores a new extension session, and triggers polling.

**Nodes involved:**  
- Parse Extend (Code)  
- Answer Extend (Telegram)  
- Get Session (Extend) (Redis)  
- Parse Session (Extend) (Code)  
- Check Extend Balance (HTTP Request)  
- Validate Extend Balance (Code)  
- Extend Balance OK? (IF)  
- Send Insufficient Extend Balance (Telegram)  
- Generate Extend Prompt (HTTP Request to OpenAI)  
- Parse Extend Prompt (Code)  
- Start Extend Generation (HTTP Request to KIE)  
- Add Extend Task ID (Code)  
- Store Extending Session (Redis)  
- Send Extending Status (Telegram)  
- Trigger Extend Polling (HTTP Request)  
- Video Extension (Sticky Note)

#### Parse Session (Extend) (key rules)
- Requires `session.taskId` present.
- Only allows `selectedModel` in `{veo3, veo3_fast}`.
- Sets `extendCredits = 60`.
- Failure modes: throws explicit errors for unsupported model or missing task.

#### Generate Extend Prompt
- OpenAI endpoint chat completions, model `gpt-4o-mini`.
- Returns plain text continuation (2–3 sentences).
- Failure modes: OpenAI refusal/latency; prompt too long.

#### Start Extend Generation
- KIE endpoint: `POST https://api.kie.ai/api/v1/veo/extend`
- Body: `{ taskId: originalTaskId, prompt: extendPrompt }`
- Failure modes: extend not supported; task not found; KIE errors.

#### Add Extend Task ID
- Creates a new `extendSessionId = {originalSessionId}_ext_{timestamp}`
- Sets `isExtend: true`, `parentSessionId`, `status: 'extending'`
- Stores status endpoint Veo record-info.

#### Trigger Extend Polling
- Calls the same external polling webhook `/webhook/video-poll` with `isExtend: true` and `parentSessionId`.

---

### Block 7 — Social Publishing (TikTok / Instagram / All via Late.dev)
**Overview:** Fetches the rendered video URL from KIE.ai status endpoint, generates a platform caption with hashtags, and publishes via Late.dev.

**Nodes involved:**  
TikTok:
- Parse TikTok (Code)
- Answer TikTok (Telegram)
- Send TikTok Status (Telegram)
- Get Session (TikTok) (Redis)
- Prepare KIE Request (TikTok) (Code)
- Fetch Video URL (TikTok) (HTTP Request to KIE)
- Prepare Publish (TikTok) (Code)
- Publish to TikTok (HTTP Request to Late.dev)
- Send TikTok Success (Telegram)

Instagram:
- Parse Instagram (Code)
- Answer Instagram (Telegram)
- Send Instagram Status (Telegram)
- Get Session (Instagram) (Redis)
- Prepare KIE Request (Instagram) (Code)
- Fetch Video URL (Instagram) (HTTP Request)
- Prepare Publish (Instagram) (Code)
- Publish to Instagram (HTTP Request)
- Send Instagram Success (Telegram)

All:
- Parse PublishAll (Code)
- Answer PublishAll (Telegram)
- Send PublishAll Status (Telegram)
- Get Session (PublishAll) (Redis)
- Prepare KIE Request (PublishAll) (Code)
- Fetch Video URL (PublishAll) (HTTP Request)
- Prepare Publish (PublishAll) (Code)
- Publish to All Platforms (HTTP Request)
- Send PublishAll Success (Telegram)

**Key implementation details:**
- KIE status URL is built dynamically: `apiUrl = https://api.kie.ai + session.statusEndpoint`.
- Video URL extraction:
  - Veo: `data.response.originUrls[0]` fallback `resultUrls[0]`
  - Market: `data.videoUrl` or `data.output.video` or parsed `data.resultJson`
- Captions:
  - Randomized “human-like” templates + hashtags from session tags + common hashtags.
- Late.dev endpoint: `POST https://getlate.dev/api/v1/posts`
  - TikTok includes `tiktokSettings` and `accountId: 695fe13a4207e06f4ca84773`
  - Instagram includes `accountId: 695fe0f44207e06f4ca84770`
  - All includes both.
- Failure modes:
  - Late.dev auth/account not connected
  - KIE status not ready yet (videoUrl null) → current code does not explicitly fail early; could publish with null URL (should be guarded)
  - Platform restrictions, rate limits, content policy rejections

---

### Block 8 — Merge Extended Video (Auto-Merge via Transloadit, or Manual)
**Overview:** Supports combining original + extended videos. Auto-merge uses Transloadit concat, polls status, uploads merged output to S3, creates a merged session, and sends the merged video back to Telegram. Manual merge sends direct links and suggested tools.

**Nodes involved:**  
Auto-merge:
- Parse AutoMerge (Code)
- Fetch Original Session (Redis)
- Fetch Extension Session (Redis)
- Get Merge URLs (Code)
- Send Merging Status (Telegram)
- Create Transloadit Assembly (HTTP Request)
- Parse Assembly Response (Code)
- Wait for Merge (Wait)
- Poll Assembly Status (HTTP Request)
- Check Merge Complete (Code)
- Merge Complete? (IF)
- Check Failed? (IF)
- Still Executing (Code)
- Send Merge Failed (Telegram)
- Download Merged Video (HTTP Request file)
- Prepare Merged Session (Code)
- Upload Merged to S3 (S3 upload)
- Get Original Session (Redis)
- Create Merged Session (Code)
- Store Merged Session (Redis)
- Download Merged for Telegram (S3 download)
- Send Merged Video (Telegram)

Manual merge:
- Parse ManualMerge (Code)
- Fetch Original (Manual) (Redis)
- Fetch Extension (Manual) (Redis)
- Get Manual URLs (Code)
- Send Manual Links (Telegram)
- Video Merge / Video Merge1 (Sticky Notes)

#### Get Merge URLs (Auto-merge)
- Extracts video URLs from session using multiple possible keys (`videoUrl`, `s3Url`, KIE raw response fields).
- Builds Transloadit params including:
  - `/http/import` steps for original and extension
  - `/video/concat` step
  - `/file/export/serve` step
- **Sensitive key:** Transloadit `authKey` is hardcoded: `kCh6E3C...`
- Failure modes:
  - Video URL not found in session (throws)
  - Transloadit key invalid/expired
  - Imported URLs inaccessible (private S3, expired KIE links)

#### Create Transloadit Assembly
- **Type / role:** HTTP multipart POST to `https://api2.transloadit.com/assemblies`
- Timeout: 120s
- Sends `params` as form-data.

#### Wait/Poll loop
- Wait 10 seconds then poll `assembly_ssl_url`.
- If `ASSEMBLY_EXECUTING`/`UPLOADING`, loops back to Wait.
- On completion extracts merged video URL from `results.exported[0].ssl_url`.

#### Upload Merged to S3 + Create merged session
- Uploads merged binary to `shorts/videos/{mergedSessionId}.mp4` where `mergedSessionId = parentSessionId + '_merged'`.
- Creates merged session object copying metadata from original session.
- Sends merged video to Telegram with publish buttons.

#### Manual merge
- Collects original and extension URLs similarly and sends a message with tool links:
  - https://www.capcut.com/
  - https://www.canva.com/video-editor/
  - https://clipchamp.com/

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main | stickyNote | High-level explanation + setup checklist | — | — | ## How it works … Setup steps … |
| Sticky Note | stickyNote | Demo video link | — | — | [![Workflow Demo Video](https://img.youtube.com/vi/OI_oJ_2F1O0/hqdefault.jpg)](https://youtu.be/OI_oJ_2F1O0) |
| Telegram Input | stickyNote | Comment | — | — | **Telegram Trigger** — Receives messages and button callbacks, routes to handlers |
| Prompt Generation | stickyNote | Comment | — | — | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Video Generation | stickyNote | Comment | — | — | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| YouTube Publishing | stickyNote | Comment | — | — | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Social Publishing | stickyNote | Comment | — | — | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Video Extension | stickyNote | Comment | — | — | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Video Merge | stickyNote | Comment | — | — | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Video Merge1 | stickyNote | Comment | — | — | **Manual-Merge** — Send original + extended video for manual merge |
| Telegram Trigger | telegramTrigger | Entry point (message + callback_query) | — | Determine Route | **Telegram Trigger** — Receives messages and button callbacks, routes to handlers |
| Determine Route | code | Classify routeType/template/userPrompt | Telegram Trigger | Route Input | **Telegram Trigger** — Receives messages and button callbacks, routes to handlers |
| Route Input | switch | Fan-out by routeType | Determine Route | Extract Input; Parse Model Selection; Parse Publish; Parse Cancel; Send Help; Parse Extend; Parse TikTok; Parse Instagram; Parse PublishAll; Parse AutoMerge; Parse ManualMerge | **Telegram Trigger** — Receives messages and button callbacks, routes to handlers |
| Extract Input | code | Create sessionId, extract text/photo | Route Input (NewMessage) | Has Photo?; Send Initial Status | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Send Initial Status | telegram | Notify “Working on it” | Extract Input | — | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Has Photo? | if | Branch on photo presence | Extract Input | Get Photo URL; No Photo | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Get Photo URL | telegram | Fetch Telegram file info | Has Photo? (true) | Build Photo URL | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Build Photo URL | code | Build Telegram file URL (uses hardcoded bot token) | Get Photo URL | Merge | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| No Photo | code | Set photoUrl null | Has Photo? (false) | Merge | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Merge | merge | Rejoin photo/no-photo paths | Build Photo URL; No Photo | Send Prompt Status | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Send Prompt Status | telegram | Notify prompt crafting | Merge | Generate Prompt & Metadata | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Generate Prompt & Metadata | httpRequest | OpenAI prompt+metadata JSON generation | Send Prompt Status | Format Session | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Format Session | code | Parse OpenAI JSON; build session object | Generate Prompt & Metadata | Store Session | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Store Session | redis | Save session:sessionId | Format Session | Send Prompt for Approval | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Send Prompt for Approval | telegram | Send prompt + model selection buttons | Store Session | — | **Prompt Generation** — Extracts input, calls OpenAI, stores session in Redis |
| Parse Model Selection | code | Parse callback model:* and map model config | Route Input (GenerateWithModel) | Answer Generate | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Answer Generate | telegram | Answer callback query | Parse Model Selection | Get Session | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Get Session | redis | Load session for generation | Answer Generate | Parse Session | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Parse Session | code | Merge session with model selection; set generating | Get Session | Check Balance | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Check Balance | httpRequest | KIE credits check | Parse Session | Validate Balance | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Validate Balance | code | Compare credits required vs balance | Check Balance | Balance OK? | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Balance OK? | if | Branch on sufficient credits | Validate Balance | Start Video Generation; Send Insufficient Balance | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Send Insufficient Balance | telegram | Edit message with credit shortfall | Balance OK? (false) | — | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Start Video Generation | httpRequest | Start KIE job (endpoint varies by model) | Balance OK? (true) | Add Task ID | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Add Task ID | code | Validate KIE result; set taskId/statusEndpoint | Start Video Generation | Store Generating Session | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Store Generating Session | redis | Save updated generating session | Add Task ID | Send Generating Status | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Send Generating Status | telegram | Edit message: job started | Store Generating Session | Trigger Polling | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Trigger Polling | httpRequest | Trigger external polling webhook | Send Generating Status | — | **Video Generation** — Checks balance, starts KIE.ai job, triggers polling webhook |
| Parse Publish | code | Parse publish:{sessionId} | Route Input (Publish) | Answer Publish | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Answer Publish | telegram | Answer callback query | Parse Publish | Get Session (Publish) | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Get Session (Publish) | redis | Load session | Answer Publish | Parse Session (Publish) | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Parse Session (Publish) | code | Parse session and attach callback IDs | Get Session (Publish) | Download Video | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Download Video | s3 | Download mp4 from S3 | Parse Session (Publish) | Upload to YouTube | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Upload to YouTube | youTube | Upload Shorts | Download Video | Format Results | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Format Results | code | Build youtubeUrl, mark published | Upload to YouTube | Update Session | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Update Session | redis | Save published session | Format Results | Send Success | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Send Success | telegram | Notify YouTube URL | Update Session | — | **YouTube Shorts** — Downloads from S3, uploads to YouTube with metadata |
| Parse TikTok | code | Parse tiktok:{sessionId} | Route Input (TikTok) | Answer TikTok | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Answer TikTok | telegram | Answer callback query | Parse TikTok | Send TikTok Status | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Send TikTok Status | telegram | Notify publishing started | Answer TikTok | Get Session (TikTok) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Get Session (TikTok) | redis | Load session | Send TikTok Status | Prepare KIE Request (TikTok) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Prepare KIE Request (TikTok) | code | Build KIE status URL + pass session | Get Session (TikTok) | Fetch Video URL (TikTok) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Fetch Video URL (TikTok) | httpRequest | Query KIE status endpoint | Prepare KIE Request (TikTok) | Prepare Publish (TikTok) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Prepare Publish (TikTok) | code | Extract videoUrl; craft caption+hashtags | Fetch Video URL (TikTok) | Publish to TikTok | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Publish to TikTok | httpRequest | Late.dev post create | Prepare Publish (TikTok) | Send TikTok Success | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Send TikTok Success | telegram | Notify done | Publish to TikTok | — | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Parse Instagram | code | Parse instagram:{sessionId} | Route Input (Instagram) | Answer Instagram | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Answer Instagram | telegram | Answer callback query | Parse Instagram | Send Instagram Status | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Send Instagram Status | telegram | Notify publishing started | Answer Instagram | Get Session (Instagram) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Get Session (Instagram) | redis | Load session | Send Instagram Status | Prepare KIE Request (Instagram) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Prepare KIE Request (Instagram) | code | Build KIE status URL + pass session | Get Session (Instagram) | Fetch Video URL (Instagram) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Fetch Video URL (Instagram) | httpRequest | Query KIE status endpoint | Prepare KIE Request (Instagram) | Prepare Publish (Instagram) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Prepare Publish (Instagram) | code | Extract videoUrl; craft IG caption | Fetch Video URL (Instagram) | Publish to Instagram | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Publish to Instagram | httpRequest | Late.dev post create | Prepare Publish (Instagram) | Send Instagram Success | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Send Instagram Success | telegram | Notify done | Publish to Instagram | — | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Parse PublishAll | code | Parse publishall:{sessionId} | Route Input (PublishAll) | Answer PublishAll | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Answer PublishAll | telegram | Answer callback query | Parse PublishAll | Send PublishAll Status | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Send PublishAll Status | telegram | Notify publishing started | Answer PublishAll | Get Session (PublishAll) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Get Session (PublishAll) | redis | Load session | Send PublishAll Status | Prepare KIE Request (PublishAll) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Prepare KIE Request (PublishAll) | code | Build KIE status URL + pass session | Get Session (PublishAll) | Fetch Video URL (PublishAll) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Fetch Video URL (PublishAll) | httpRequest | Query KIE status endpoint | Prepare KIE Request (PublishAll) | Prepare Publish (PublishAll) | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Prepare Publish (PublishAll) | code | Extract videoUrl; craft caption | Fetch Video URL (PublishAll) | Publish to All Platforms | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Publish to All Platforms | httpRequest | Late.dev post to TikTok+IG | Prepare Publish (PublishAll) | Send PublishAll Success | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Send PublishAll Success | telegram | Notify done | Publish to All Platforms | — | **TikTok & Instagram** — Publishes via Late.dev API with auto-generated captions |
| Parse Extend | code | Parse extend:{sessionId} | Route Input (Extend) | Answer Extend | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Answer Extend | telegram | Answer callback query | Parse Extend | Get Session (Extend) | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Get Session (Extend) | redis | Load session | Answer Extend | Parse Session (Extend) | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Parse Session (Extend) | code | Validate taskId + Veo-only; set extendCredits | Get Session (Extend) | Check Extend Balance | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Check Extend Balance | httpRequest | KIE credit check | Parse Session (Extend) | Validate Extend Balance | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Validate Extend Balance | code | Compare balance vs extendCredits | Check Extend Balance | Extend Balance OK? | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Extend Balance OK? | if | Branch | Validate Extend Balance | Generate Extend Prompt; Send Insufficient Extend Balance | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Send Insufficient Extend Balance | telegram | Edit message error | Extend Balance OK? (false) | — | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Generate Extend Prompt | httpRequest | OpenAI continuation prompt | Extend Balance OK? (true) | Parse Extend Prompt | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Parse Extend Prompt | code | Extract continuation text | Generate Extend Prompt | Start Extend Generation | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Start Extend Generation | httpRequest | KIE Veo extend job | Parse Extend Prompt | Add Extend Task ID | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Add Extend Task ID | code | Create extension sessionId; set taskId | Start Extend Generation | Store Extending Session | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Store Extending Session | redis | Store extension session | Add Extend Task ID | Send Extending Status | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Send Extending Status | telegram | Notify extending | Store Extending Session | Trigger Extend Polling | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Trigger Extend Polling | httpRequest | Trigger external polling webhook | Send Extending Status | — | **Extend Video** — Adds 8 seconds using OpenAI continuation prompt (Veo only) |
| Parse AutoMerge | code | Parse am:{sessionId}; derive parentSessionId | Route Input (AutoMerge) | Fetch Original Session | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Fetch Original Session | redis | Load original session | Parse AutoMerge | Fetch Extension Session | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Fetch Extension Session | redis | Load extension session | Fetch Original Session | Get Merge URLs | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Get Merge URLs | code | Extract URLs; build Transloadit params | Fetch Extension Session | Send Merging Status | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Send Merging Status | telegram | Notify merge start | Get Merge URLs | Create Transloadit Assembly | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Create Transloadit Assembly | httpRequest | Create concat job | Send Merging Status | Parse Assembly Response | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Parse Assembly Response | code | Store assembly IDs/pollUrl | Create Transloadit Assembly | Wait for Merge | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Wait for Merge | wait | Delay between polls | Parse Assembly Response; Still Executing | Poll Assembly Status | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Poll Assembly Status | httpRequest | Poll Transloadit assembly | Wait for Merge | Check Merge Complete | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Check Merge Complete | code | Interpret status; extract mergedVideoUrl | Poll Assembly Status | Merge Complete? | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Merge Complete? | if | Branch complete vs not | Check Merge Complete | Download Merged Video; Check Failed? | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Check Failed? | if | Branch failed vs still executing | Merge Complete? (false) | Send Merge Failed; Still Executing | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Still Executing | code | Pass-through to loop | Check Failed? (false) | Wait for Merge | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Send Merge Failed | telegram | Notify failure + manual URLs | Check Failed? (true) | — | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Download Merged Video | httpRequest | Download merged file binary | Merge Complete? (true) | Prepare Merged Session | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Prepare Merged Session | code | Create mergedSessionId and merge metadata pointers | Download Merged Video | Upload Merged to S3 | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Upload Merged to S3 | s3 | Upload merged mp4 to S3 | Prepare Merged Session | Get Original Session | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Get Original Session | redis | Load original for metadata copy | Upload Merged to S3 | Create Merged Session | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Create Merged Session | code | Build merged session object | Get Original Session | Store Merged Session | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Store Merged Session | redis | Store merged session | Create Merged Session | Download Merged for Telegram | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Download Merged for Telegram | s3 | Download merged mp4 for sending | Store Merged Session | Send Merged Video | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Send Merged Video | telegram | Send video + publish buttons | Download Merged for Telegram | — | **Auto-Merge** — Concatenates original + extended video via Transloadit |
| Parse ManualMerge | code | Parse mm:{sessionId}; derive parentSessionId | Route Input (ManualMerge) | Fetch Original (Manual) | **Manual-Merge** — Send original + extended video for manual merge |
| Fetch Original (Manual) | redis | Load original | Parse ManualMerge | Fetch Extension (Manual) | **Manual-Merge** — Send original + extended video for manual merge |
| Fetch Extension (Manual) | redis | Load extension | Fetch Original (Manual) | Get Manual URLs | **Manual-Merge** — Send original + extended video for manual merge |
| Get Manual URLs | code | Extract URLs and title | Fetch Extension (Manual) | Send Manual Links | **Manual-Merge** — Send original + extended video for manual merge |
| Send Manual Links | telegram | Send URLs + tool links | Get Manual URLs | — | **Manual-Merge** — Send original + extended video for manual merge |
| Parse Cancel | code | Parse cancel:{sessionId} | Route Input (Cancel) | Answer Cancel |  |
| Delete Session | redis | Delete session key | Answer Cancel | Send Cancel |  |
| Send Cancel | telegram | Edit message to cancelled | Delete Session | — |  |
| Send Help | telegram | Display help text | Route Input (Help) | — |  |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

1. **Create a new workflow** in n8n and name it:  
   “Generate text-to-video content with Telegram, OpenAI and KIE.ai, then publish to YouTube, TikTok and Instagram”.

2. **Add Telegram Trigger**
   - Node: *Telegram Trigger*
   - Updates: `message`, `callback_query`
   - Credentials: Telegram API token (create bot via @BotFather).

3. **Add “Determine Route” (Code)**
   - Node: *Code*
   - Implement routing logic:
     - If `message`: detect slash commands (`/general /lost /3d /story /help`), set `routeType`, `template`, `userPrompt`
     - If `callback_query`: map prefixes `model: publish: tiktok: instagram: publishall: extend: cancel: am: mm:`
   - Connect: Telegram Trigger → Determine Route.

4. **Add “Route Input” (Switch)**
   - Node: *Switch*
   - Create rules (string equals on `{{$json.routeType}}`):
     - `new_message`, `generate_with_model`, `publish`, `cancel`, `help`, `extend`, `tiktok`, `instagram`, `publishall`, `automerge`, `manualmerge`
   - Connect: Determine Route → Route Input.

---

### Prompt Generation branch (`new_message`)

5. **Add “Extract Input” (Code)**
   - Create `sessionId`, `chatId`, `messageId`, `textInput`, `photoFileId`, `hasPhoto`, `template`, timestamp.
   - Connect: Route Input (NewMessage) → Extract Input.

6. **Add “Send Initial Status” (Telegram)**
   - Operation: Send Message
   - Chat ID: `{{$json.chatId}}`
   - Text: “Working on it…”
   - Markdown enabled
   - Connect: Extract Input → Send Initial Status (in parallel with next step).

7. **Add “Has Photo?” (IF)**
   - Condition: `{{$json.hasPhoto}} == true`
   - Connect: Extract Input → Has Photo?

8. **Photo path**
   - Add “Get Photo URL” (Telegram node, resource: file)
     - fileId: `{{$json.photoFileId}}`
   - Add “Build Photo URL” (Code)
     - Build `photoUrl` from Telegram `file_path`
     - **Important:** do *not* hardcode the bot token. Prefer:
       - Store bot token in n8n credentials or environment variable and inject it, or
       - Use Telegram API file URL methods in a safer way.
   - Connect: Has Photo? (true) → Get Photo URL → Build Photo URL.

9. **No-photo path**
   - Add “No Photo” (Code) setting `photoUrl: null` while preserving other fields.
   - Connect: Has Photo? (false) → No Photo.

10. **Merge paths**
   - Add “Merge” (Merge node) to reunify photo and no-photo outputs.
   - Connect: Build Photo URL → Merge, and No Photo → Merge.

11. **Add “Send Prompt Status” (Telegram)**
   - Chat ID: `{{$json.chatId}}`
   - Text: “Crafting your video prompt…”
   - Connect: Merge → Send Prompt Status.

12. **Add “Generate Prompt & Metadata” (HTTP Request to OpenAI)**
   - Method: POST
   - URL: `https://api.openai.com/v1/chat/completions`
   - Auth: Header Auth credential (OpenAI API key as `Authorization: Bearer ...`)
   - Body: JSON, with:
     - model: `gpt-4o`
     - system prompt chosen by `template`
     - user content includes text and optional `image_url`
     - `response_format: { type: "json_object" }`
   - Connect: Send Prompt Status → Generate Prompt & Metadata.

13. **Add “Format Session” (Code)**
   - Parse OpenAI `choices[0].message.content` JSON.
   - Create session object including metadata and `status: 'prompt_ready'`.
   - Connect: Generate Prompt & Metadata → Format Session.

14. **Add “Store Session” (Redis set)**
   - Key: `session:{{$json.sessionId}}`
   - Value: `{{JSON.stringify($json)}}`
   - TTL: 86400, expiry enabled.
   - Connect: Format Session → Store Session.

15. **Add “Send Prompt for Approval” (Telegram)**
   - Send message with generated prompt + title/tags.
   - Add inline keyboard:
     - Model buttons with callback_data `model:<model>:<sessionId>`
     - Cancel button `cancel:<sessionId>`
   - Connect: Store Session → Send Prompt for Approval.

---

### Generation branch (`generate_with_model`)

16. **Add “Parse Model Selection” (Code)**
   - Parse callback format `model:modelName:sessionId`
   - Map to `modelType` and required credits
   - Connect: Route Input (GenerateWithModel) → Parse Model Selection.

17. **Add “Answer Generate” (Telegram callback answer)**
   - queryId: `{{$json.callbackQueryId}}`
   - Connect: Parse Model Selection → Answer Generate.

18. **Add “Get Session” (Redis get)**
   - Key: `session:{{ $('Parse Model Selection').first().json.sessionId }}`
   - Connect: Answer Generate → Get Session.

19. **Add “Parse Session” (Code)**
   - Parse Redis string (often found in `propertyName`)
   - Merge in model selection info
   - Connect: Get Session → Parse Session.

20. **Add “Check Balance” (HTTP Request to KIE)**
   - URL: `https://api.kie.ai/api/v1/chat/credit`
   - Header Auth: KIE API key
   - Connect: Parse Session → Check Balance.

21. **Add “Validate Balance” (Code)**
   - Compare `balance >= estimatedCredits`
   - Connect: Check Balance → Validate Balance.

22. **Add “Balance OK?” (IF)**
   - Condition: `{{$json.hasEnough}} == true`
   - False → Telegram edit message “Insufficient credits”
   - True → start generation

23. **Add “Start Video Generation” (HTTP Request)**
   - URL chosen by selected model (veo vs market endpoints)
   - Body format differs per model as in the workflow
   - Connect: Balance OK? (true) → Start Video Generation.

24. **Add “Add Task ID” (Code)**
   - Validate KIE response (`code == 200`)
   - Add `taskId`, `statusEndpoint` and persist polling fields
   - Connect: Start Video Generation → Add Task ID.

25. **Add “Store Generating Session” (Redis set)**
   - Store updated session under same key
   - Connect: Add Task ID → Store Generating Session.

26. **Add “Send Generating Status” (Telegram edit message)**
   - Edit the model-selection message to show task info
   - Connect: Store Generating Session → Send Generating Status.

27. **Add “Trigger Polling” (HTTP Request)**
   - POST your polling workflow webhook, send session/task info
   - Connect: Send Generating Status → Trigger Polling.
   - **Also build the polling workflow** (separate): it must poll KIE, download the result, upload to S3, update Redis, and message the user with publish/extend/merge buttons.

---

### YouTube publish branch (`publish`)

28. **Add Parse/Answer/Get/Parse nodes** (as above):
   - Parse Publish (Code) → Answer Publish (Telegram callback) → Get Session (Redis) → Parse Session (Code)

29. **Add “Download Video” (S3)**
   - Bucket: `shorts`
   - Key: `videos/{{$json.sessionId}}.mp4`

30. **Add “Upload to YouTube”**
   - OAuth2 credential for YouTube
   - Operation upload video, set title/description/tags from session

31. **Add “Format Results” (Code)** then **Update Session (Redis)** then **Send Success (Telegram)**

---

### TikTok / Instagram / All branches

32. For each branch:
   - Parse callback → Answer callback → status message
   - Redis get session
   - Build KIE status request (apiUrl + taskId)
   - Fetch status, extract video URL
   - Generate caption + hashtags
   - POST to Late.dev `/api/v1/posts` with correct platform accountId(s)
   - Send success message

---

### Extend branch (`extend`) (Veo-only)

33. Parse Extend → Answer Extend → Get Session (Redis) → Parse Session (validate veo + taskId)
34. Check credits (KIE) → validate → IF enough:
35. OpenAI continuation prompt (gpt-4o-mini) → parse
36. KIE Veo extend endpoint → store extension session (new sessionId)
37. Notify user and trigger polling webhook with `isExtend: true` + `parentSessionId`

---

### Merge branches (`am:` and `mm:`)

38. **Manual merge**
   - Load original and extension sessions → extract URLs → send message with both links and tool links.

39. **Auto-merge**
   - Load both sessions → extract URLs → create Transloadit assembly for concat
   - Poll until complete
   - Download merged file → upload to S3 → create merged session copying original metadata
   - Send merged video to Telegram with publish buttons

**Credentials required**
- Telegram Bot API
- OpenAI API (Header Auth)
- KIE.ai API key (Header Auth)
- Redis credentials
- S3 credentials (bucket `shorts`)
- YouTube OAuth2
- Late.dev API key (Header Auth)
- Transloadit key (**do not hardcode; store as credential or env var**)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow demo video thumbnail link | https://youtu.be/OI_oJ_2F1O0 |
| Manual merge tool links included in Telegram message | https://www.capcut.com/ ; https://www.canva.com/video-editor/ ; https://clipchamp.com/ |
| External dependency: polling webhook must exist and match payload | `POST https://n8n.mosaabyassir.xyz/webhook/video-poll` |
| Security concern: Telegram bot token is hardcoded in “Build Photo URL” | Replace with env var/credential-based injection; rotate token if exposed |
| Security concern: Transloadit auth key is hardcoded in “Get Merge URLs” | Move to credential/env var and rotate if exposed |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.