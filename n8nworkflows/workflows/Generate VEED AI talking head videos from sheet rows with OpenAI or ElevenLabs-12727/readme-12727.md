Generate VEED AI talking head videos from sheet rows with OpenAI or ElevenLabs

https://n8nworkflows.xyz/workflows/generate-veed-ai-talking-head-videos-from-sheet-rows-with-openai-or-elevenlabs-12727


# Generate VEED AI talking head videos from sheet rows with OpenAI or ElevenLabs

## 1. Workflow Overview

**Workflow name:** VEED AI Talking Head Video Generator with Multi-Platform Publishing  
**Purpose:** Generate AI “talking head” avatar videos from Google Sheet rows using VEED Fabric 1.0 (via Fal.ai), with per-row configurable TTS (OpenAI or ElevenLabs), an email approval step, and optional sequential publishing to multiple social platforms.  
**Primary use case:** Batch production and controlled publishing of short-form/vertical avatar videos (Reels/Shorts/Threads, etc.) without mixing context between rows.

### 1.1 Trigger & Input Reception (Sheet-driven queue)
The workflow can run manually or every 15 minutes. It loads Google Sheet rows, filters rows where `STATUS = new`, then processes rows **sequentially** via a Split-in-Batches loop.

### 1.2 Per-row Preparation & Status Lock
For each row, it marks the row as `processing` to prevent duplicate processing, and normalizes parameters (defaults for provider/voice/aspect/resolution).

### 1.3 Text-to-Speech (OpenAI or ElevenLabs) + Drive hosting
Depending on `TTS_PROVIDER`, it generates narration audio, uploads it to Google Drive, makes it public, and builds a direct download URL for VEED.

### 1.4 VEED Video Generation + Error Handling
VEED Fabric 1.0 generates the talking-head video using the portrait image URL + public audio URL. If generation fails or video URL is missing, the row is marked `error` and an error email is sent.

### 1.5 Approval Gate (Email “approve/reject”)
When video generation succeeds, an approval email is sent using Gmail “sendAndWait” with a 24-hour wait. If rejected, the row is marked `rejected`. If approved, it proceeds to publishing.

### 1.6 Multi-platform Publishing (Optional, sequential per platform)
If the `PLATFORMS` column is empty, the workflow skips publishing (still updates status accordingly). Otherwise it splits platforms into one item per platform, loops **one platform at a time**, routes to the correct platform node, formats the outcome into a standard structure, aggregates results, and computes overall status (`published`, `partial`, `failed`, etc.).

### 1.7 Reporting & Final Status Update
Updates the Google Sheet with final status, video URL, and summary. Sends a detailed report email, then loops back to the next row.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Input Reception
**Overview:** Starts on manual run or schedule, reads Google Sheet, filters `STATUS=new`, and enters the row loop.  
**Nodes involved:**  
- When clicking 'Test workflow'  
- Check for New Videos  
- Get New Video Requests  
- Filter Status New  
- Loop Over Rows

#### Node: When clicking 'Test workflow'
- **Type / role:** Manual Trigger; entry point for testing.
- **Configuration:** No parameters.
- **Outputs:** To **Get New Video Requests**.
- **Failure modes:** None.

#### Node: Check for New Videos
- **Type / role:** Schedule Trigger; periodic polling.
- **Configuration:** Every **15 minutes**.
- **Outputs:** To **Get New Video Requests**.
- **Failure modes:** None (but workflow must be **active** to run on schedule).

#### Node: Get New Video Requests
- **Type / role:** Google Sheets node; reads rows from a sheet tab.
- **Configuration choices:**
  - Document ID placeholder: `YOUR_GOOGLE_SHEET_ID`
  - Sheet name selector uses `gid=0` (first tab).
  - `onError: continueRegularOutput` → downstream logic may run with partial/empty data.
- **Inputs:** From either trigger.
- **Outputs:** To **Filter Status New**.
- **Potential failures / edge cases:**
  - Google OAuth not configured / expired token.
  - Wrong spreadsheet ID or permission denied.
  - If the sheet lacks `STATUS` or `row_number`, later expressions break.

#### Node: Filter Status New
- **Type / role:** Filter; retains only items where row is queued for processing.
- **Condition:** `{{ $json.STATUS }} equals "new"` (case-insensitive).
- **Inputs:** Get New Video Requests.
- **Outputs:** Loop Over Rows.
- **Edge cases:**
  - Missing `STATUS` field → filter evaluates false and drops row.
  - If `STATUS` contains whitespace/case variants, filter is case-insensitive but does not trim (e.g., `"new "` would fail).

#### Node: Loop Over Rows
- **Type / role:** Split in Batches; ensures sequential processing of sheet rows.
- **Configuration:** Default options; uses the standard SplitInBatches pattern.
- **Connections:** Uses **output index 1** to proceed to processing nodes, and cycles back to itself after completion.
- **Inputs:** Filter Status New, and later loop-back from reporting/error/rejection nodes.
- **Outputs:**
  - Output 1 (index 1) → **Mark as Processing**
  - Output 0 (index 0) is unused in connections (common pattern: empty means “done”).
- **Edge cases:**
  - If upstream provides 0 items, nothing processes.
  - Correct looping depends on the various “end-of-row” nodes returning to Loop Over Rows.

---

### Block 2 — Row Lock + Parameter Normalization
**Overview:** Marks the current row as `processing`, then maps sheet columns into stable internal fields with defaults.  
**Nodes involved:**  
- Mark as Processing  
- Prepare Video Parameters

#### Node: Mark as Processing
- **Type / role:** Google Sheets update; sets a lock/status for current row.
- **Configuration choices:**
  - Operation: **update**
  - Matching column: `row_number`
  - Sets `STATUS="processing"` for `row_number={{ $('Loop Over Rows').item.json.row_number }}`
  - `onError: continueRegularOutput`
- **Input:** Loop Over Rows (batch item).
- **Output:** Prepare Video Parameters.
- **Edge cases / failures:**
  - If `row_number` is missing from sheet data, update won’t match any row.
  - If multiple rows share the same `row_number` (should not happen), updates may be incorrect.
  - “Continue on error” may allow processing without a proper lock.

#### Node: Prepare Video Parameters
- **Type / role:** Set node; normalizes all needed per-row parameters.
- **Key fields produced:**
  - `imageUrl` ← `AVATAR_IMAGE_URL`
  - `narrationText` ← `NARRATION_TEXT`
  - `ttsProvider` ← `TTS_PROVIDER` default `"openai"`
  - `voiceId` ← `VOICE_ID` default `"alloy"`
  - `videoTitle`, `videoDescription`
  - `platforms` ← `PLATFORMS` default `""`
  - `aspectRatio` default `"9:16"`
  - `resolution` default `"720p"`
  - `rowNumber` number ← `row_number`
- **Input:** Mark as Processing.
- **Output:** TTS Provider?
- **Edge cases:**
  - Invalid aspect ratio/resolution values are not validated here; downstream nodes may fail.
  - If `PLATFORMS` contains unexpected separators, later split logic may produce wrong platform names.

---

### Block 3 — TTS Generation + Drive Hosting
**Overview:** Generates narration audio using selected provider, merges the two possible provider outputs into a single path, uploads MP3 to Drive, makes it public, and produces a direct audio URL.  
**Nodes involved:**  
- TTS Provider?  
- ElevenLabs TTS  
- OpenAI TTS  
- Merge TTS  
- Upload to Drive  
- Make Public  
- Set Audio URL

#### Node: TTS Provider?
- **Type / role:** IF router; selects TTS provider path.
- **Condition:** `{{ $json.ttsProvider === 'elevenlabs' }}` must be true to go to ElevenLabs.
- **Outputs:**
  - True → ElevenLabs TTS
  - False → OpenAI TTS
- **Edge cases:**
  - Provider values must match exactly `'elevenlabs'` (lowercase); `'ElevenLabs'` will fall back to OpenAI.

#### Node: ElevenLabs TTS
- **Type / role:** HTTP Request; calls ElevenLabs text-to-speech and returns a binary audio file.
- **Configuration choices:**
  - POST `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.voiceId }}`
  - JSON body:
    - `text`: stringified narration
    - `model_id`: `eleven_multilingual_v2`
    - `voice_settings`: stability/similarity_boost
  - Response format: **file** (binary)
  - Auth: **HTTP Header Auth** via “genericCredentialType”
  - `onError: continueRegularOutput`
- **Input:** TTS Provider?
- **Output:** Merge TTS (index 0)
- **Potential failures:**
  - 401/403 invalid API key
  - 404 invalid voiceId
  - Rate limit (429), payload too large, or timeouts
  - If audio returned is not MP3-compatible, Drive upload may fail depending on metadata.

#### Node: OpenAI TTS
- **Type / role:** OpenAI (LangChain) node; generates audio from text.
- **Configuration choices:**
  - Resource: **audio**
  - Input: `narrationText`
  - Voice: `voiceId`
  - (Sticky note says model `tts-1-hd`, but the node config does not explicitly show model selection; model may be node default or credential default depending on n8n version.)
  - `onError: continueRegularOutput`
- **Input:** TTS Provider? (false branch)
- **Output:** Merge TTS (index 1)
- **Potential failures:**
  - OpenAI credential missing/invalid
  - Voice name invalid for the selected model
  - Output binary property expectations differ across n8n versions of the OpenAI node.

#### Node: Merge TTS
- **Type / role:** Merge; combines either TTS output into a unified stream.
- **Configuration:** Mode **combine**, include unpaired items = true, combine by position.
- **Inputs:** ElevenLabs TTS (input 0) and OpenAI TTS (input 1).
- **Output:** Upload to Drive.
- **Edge cases:**
  - If one branch errors but still outputs an item without binary data (due to continue-on-error), upload may fail.

#### Node: Upload to Drive
- **Type / role:** Google Drive; uploads the generated audio file.
- **Configuration choices:**
  - File name: timestamp `yyyyMMdd-HHmmss-audio.mp3`
  - Folder ID placeholder: `YOUR_AUDIO_FOLDER_ID`
  - `onError: continueRegularOutput`
- **Input:** Merge TTS (expects binary file data).
- **Output:** Make Public.
- **Failures / edge cases:**
  - Missing binary property (common if upstream failed).
  - Wrong folder ID or insufficient permissions.
  - Drive quota exceeded.

#### Node: Make Public
- **Type / role:** Google Drive share operation; makes uploaded audio publicly readable.
- **Configuration:**
  - Operation: share
  - Permissions: `role=reader`, `type=anyone`
  - File ID: `{{ $json.id }}`
  - `onError: continueRegularOutput`
- **Input:** Upload to Drive.
- **Output:** Set Audio URL.
- **Edge cases:**
  - Org policies may block public sharing.
  - If file ID is missing due to upload failure, expression fails or returns invalid.

#### Node: Set Audio URL
- **Type / role:** Set node; constructs direct download URL for VEED.
- **Key expression:**
  - `audioUrl = https://drive.google.com/uc?export=download&id={{ $('Upload to Drive').item.json.id }}`
- **Input:** Make Public.
- **Output:** VEED Generate Video.
- **Edge cases:**
  - Uses `Upload to Drive` node reference, not `$json.id`; if items desync, could reference wrong ID (usually fine due to sequential processing).
  - Public access might still not work immediately due to Drive propagation delays.

---

### Block 4 — VEED Video Generation + Error Branch
**Overview:** Calls VEED Fabric 1.0 to create a talking-head video; detects error/missing URL and routes to sheet update + email notification.  
**Nodes involved:**  
- VEED Generate Video  
- VEED Error?  
- Update Sheet - Error  
- Send Error Email

#### Node: VEED Generate Video
- **Type / role:** `n8n-nodes-veed.veed`; generates video via VEED Fabric 1.0 (Fal.ai-backed).
- **Configuration:**
  - `audioUrl` from Set Audio URL
  - `imageUrl`, `resolution`, `aspectRatio` from Prepare Video Parameters
  - Timeout option set to **15** (minutes implied by node; depends on node implementation)
  - `onError: continueRegularOutput`
- **Input:** Set Audio URL.
- **Output:** VEED Error?
- **Failures / edge cases:**
  - Fal.ai key missing/invalid.
  - Unsupported image/audio URL (not publicly accessible, redirects, 403).
  - Long generation times may exceed timeout; node may return an error.
  - Output schema assumptions: expects `$json.video.url`.

#### Node: VEED Error?
- **Type / role:** IF; detects generation failure.
- **Condition (OR):**
  - `$json.error` exists, OR
  - `$json.video?.url` does not exist
- **Outputs:**
  - True (error) → Update Sheet - Error
  - False (ok) → Format Email
- **Edge cases:**
  - If VEED returns errors in a different field name, this may miss it.
  - If VEED returns a URL under a different path, it will be treated as failure.

#### Node: Update Sheet - Error
- **Type / role:** Google Sheets update; marks row status.
- **Configuration:**
  - Operation: update by `row_number`
  - Sets `STATUS="error"`
  - Row number from Prepare Video Parameters `rowNumber`
  - `onError: continueRegularOutput`
- **Input:** VEED Error? (true branch)
- **Output:** Send Error Email
- **Failure modes:** OAuth issues, mismatched `row_number`.

#### Node: Send Error Email
- **Type / role:** Gmail; notifies about generation failure.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - Subject: `[ERROR] Video Generation Failed: {{ videoTitle }}`
  - HTML body includes VEED error or missing URL message and row details.
- **Input:** Update Sheet - Error
- **Output:** Loop Over Rows (continues with next row)
- **Edge cases:**
  - Gmail credentials missing, “From” restrictions, daily sending limits.
  - If error object is huge/unstructured, email may be large.

---

### Block 5 — Approval Workflow
**Overview:** Formats approval email with preview link, waits up to 24 hours for approve/reject, routes either to publishing or to rejection logging and sheet update.  
**Nodes involved:**  
- Format Email  
- Send Approval Email  
- Approved?  
- Log Rejection  
- Update Sheet - Rejected

#### Node: Format Email
- **Type / role:** Set node; builds HTML email body and stores generated video URL.
- **Outputs fields:**
  - `emailBody`: HTML with title/description/platforms/preview link
  - `generatedVideoUrl`: `{{ $('VEED Generate Video').item.json.video.url }}`
- **Input:** VEED Error? (success branch)
- **Output:** Send Approval Email
- **Edge cases:** If VEED output lacks `duration` or `requestId`, email still renders but those values may be `undefined`.

#### Node: Send Approval Email
- **Type / role:** Gmail “sendAndWait”; pauses execution until approval action or timeout.
- **Configuration choices:**
  - Operation: **sendAndWait**
  - Wait time: **24 hours** (`resumeAmount: 24`)
  - Approval type: **double** (two-button approve/reject)
  - To: `user@example.com` placeholder
- **Input:** Format Email
- **Output:** Approved?
- **Failures / edge cases:**
  - If the n8n instance cannot resume waiting executions (misconfigured webhooks, queue mode issues), approvals may not work.
  - If user never responds, execution resumes after timeout with a non-approved payload (behavior depends on n8n Gmail node implementation).

#### Node: Approved?
- **Type / role:** IF; checks user response.
- **Condition:** `{{ $json.data.approved }}` is true.
- **Outputs:**
  - True → Prepare Publish Data
  - False → Log Rejection
- **Edge cases:**
  - If timeout returns different structure than `data.approved`, this may evaluate false and mark as rejected unintentionally (currently it routes to rejection branch).

#### Node: Log Rejection
- **Type / role:** Set; adds rejection metadata.
- **Sets:**
  - `status="rejected"`
  - `rejectedAt=now`
- **Input:** Approved? (false branch)
- **Output:** Update Sheet - Rejected

#### Node: Update Sheet - Rejected
- **Type / role:** Google Sheets update; marks row as rejected.
- **Configuration:** update by `row_number`, set `STATUS="rejected"`.
- **Input:** Log Rejection
- **Output:** Loop Over Rows
- **Edge cases:** Same as other sheet updates.

---

### Block 6 — Publishing Preparation + Download
**Overview:** Prepares a normalized publish payload (video URL, title, description, platform list), downloads the generated video as binary for APIs that require upload, and fans out platforms.  
**Nodes involved:**  
- Prepare Publish Data  
- Download Video  
- Split Out Platforms  
- Loop Over Platforms  
- Platform Router

#### Node: Prepare Publish Data
- **Type / role:** Set; creates publish-ready fields.
- **Key fields:**
  - `videoUrl` from `Format Email.generatedVideoUrl`
  - `title`, `description`
  - `platformList` (array): `platforms.toLowerCase().split(',').map(trim)`
- **Input:** Approved? (true branch)
- **Output:** Download Video
- **Edge cases:**
  - Empty platforms string becomes `[""]` (because `''.split(',')` yields `['']`). This can cause an invalid platform item to be routed; however downstream logic may still proceed and produce errors. A safer approach would filter empty strings.
  - If PLATFORMS has spaces/newlines, trimming helps but not for unexpected delimiters like `;`.

#### Node: Download Video
- **Type / role:** HTTP Request; downloads the video binary.
- **Configuration:** URL = `{{ $json.videoUrl }}`, responseFormat = file, continue on error.
- **Input:** Prepare Publish Data
- **Output:** Split Out Platforms (with binary included)
- **Failure modes:**
  - VEED URL may expire, require auth, or block hotlinking.
  - Large files may exceed memory limits; consider streaming or external hosting.

#### Node: Split Out Platforms
- **Type / role:** Split Out; turns `platformList` array into one item per platform while keeping other fields and binary.
- **Configuration:** include all other fields; includeBinary = true.
- **Input:** Download Video
- **Output:** Loop Over Platforms
- **Edge cases:** If `platformList` contains empty string, it will create an item with `platformList=""`.

#### Node: Loop Over Platforms
- **Type / role:** Split in Batches; publishes platforms sequentially.
- **Configuration:**
  - `batchSize=1`
  - `reset` expression: `{{ $node['Loop Over Rows'].context['noItemsLeft'] }}` (attempts to reset the platform loop based on outer loop state)
- **Outputs:**
  - Output 1 → Platform Router (process one platform)
  - Output 0 → Aggregate Platform Results (when done)
- **Potential issues:**
  - The `reset` expression referencing `Loop Over Rows` context is fragile; if context key differs by n8n version, the platform loop might not reset correctly between rows.
  - If the platform loop doesn’t reset, later rows may skip publishing or aggregate wrong results.

#### Node: Platform Router
- **Type / role:** Switch; routes by platform string.
- **Rules:** instagram / youtube / facebook / telegram / threads
- **Value:** `{{ $json.platformList }}`
- **Outputs:** To each platform publish node.
- **Edge cases:** Unrecognized platform values go to no output → platform item would be dropped (and never formatted), affecting final aggregation counts.

---

### Block 7 — Platform Publishing (Instagram / YouTube / Facebook / Telegram / Threads)
**Overview:** Uploads/posts video to selected platform; each path ends by formatting a standard result object `{platform, success, postId, postUrl, error}` then returns to the platform loop.  
**Nodes involved:**  
- Instagram Create → Wait 90s → Instagram Publish → Format Instagram Result  
- YouTube Upload → Format YouTube Result  
- Facebook Post → Format Facebook Result  
- Telegram Post → Format Telegram Result  
- Threads Create Container → Wait 30s Threads → Threads Publish → Format Threads Result  
- (All formatter nodes loop back to Loop Over Platforms)

#### Instagram path
##### Node: Instagram Create
- **Type / role:** HTTP Request; creates a Reels media container.
- **Configuration:**
  - POST `https://graph.facebook.com/v23.0/YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID/media`
  - Query params: `media_type=REELS`, `video_url={{videoUrl}}`, `caption={{title + "\n\n" + description}}`
  - Auth: predefined credential type `facebookGraphApi`
  - continue on error
- **Input:** Platform Router (instagram)
- **Output:** Wait 90s
- **Failures:** Missing permissions (instagram_basic, instagram_content_publish), invalid business account ID, video URL inaccessible to Meta, rate limits.

##### Node: Wait 90s
- **Type / role:** Wait; allows Meta to process container.
- **Configuration:** 90 seconds.
- **Output:** Instagram Publish

##### Node: Instagram Publish
- **Type / role:** HTTP Request; publishes the created container.
- **Configuration:**
  - POST `.../media_publish` with `creation_id={{Instagram Create.id}}`
- **Output:** Format Instagram Result
- **Failures:** Container not ready, container error state, invalid creation_id.

##### Node: Format Instagram Result
- **Type / role:** Set; standardizes output.
- **Logic:** success if `id` exists and no error; postUrl uses `https://www.instagram.com/reel/{id}/`
- **Output:** Loop Over Platforms
- **Edge cases:** Instagram Reel URL format may not match container/publish ID; some APIs return media ID not shortcode.

#### YouTube path
##### Node: YouTube Upload
- **Type / role:** YouTube node; uploads video.
- **Configuration:**
  - Resource: video; Operation: upload
  - Title/Description from fields; privacyStatus public; categoryId 22
  - continue on error
- **Input:** Platform Router (youtube)
- **Output:** Format YouTube Result
- **Edge cases / failures:**
  - Requires binary video property; this node must receive the binary from Download Video → Split Out Platforms.
  - OAuth scope / channel restrictions; quota limits; “made for kids” setting not handled.

##### Node: Format YouTube Result
- **Type / role:** Set; success if `uploadId` exists; URL `https://youtube.com/watch?v={uploadId}`
- **Output:** Loop Over Platforms

#### Facebook path
##### Node: Facebook Post
- **Type / role:** Facebook Graph API node; uploads a video to a Page.
- **Configuration:**
  - Edge: `videos`
  - Node/Page ID: `YOUR_FACEBOOK_PAGE_ID`
  - POST, graphApiVersion v23.0
  - sendBinaryData = true; binaryPropertyName = `data`
  - Query: title + description
- **Input:** Platform Router (facebook)
- **Output:** Format Facebook Result
- **Failures:** Page permissions missing, video size/format limitations, binary property mismatch.

##### Node: Format Facebook Result
- **Type / role:** Set; success if `id` exists; URL `https://www.facebook.com/{id}`
- **Output:** Loop Over Platforms

#### Telegram path
##### Node: Telegram Post
- **Type / role:** Telegram node; sends video to chat/channel.
- **Configuration:**
  - Operation: sendVideo
  - chatId placeholder `YOUR_TELEGRAM_CHAT_ID`
  - binaryData=true, caption = title + description
- **Input:** Platform Router (telegram)
- **Output:** Format Telegram Result
- **Failures:** Bot not admin in channel, wrong chat ID, file too large for Telegram limits, rate limits.

##### Node: Format Telegram Result
- **Type / role:** Set; success if `ok===true` and `result.video` exists; builds t.me URL if username exists.
- **Output:** Loop Over Platforms
- **Edge cases:** For private channels (no username), postUrl will be blank.

#### Threads path
##### Node: Threads Create Container
- **Type / role:** HTTP Request; creates a Threads media container.
- **Configuration:**
  - POST `https://graph.threads.net/v1.0/YOUR_THREADS_USER_ID/threads`
  - Query: `media_type=VIDEO`, `video_url`, `text`
  - Auth: HTTP Query Auth (credential) with access_token
  - continue on error
- **Input:** Platform Router (threads)
- **Output:** Wait 30s Threads
- **Failures:** Token invalid/expired, unsupported video URL, API availability/permission issues.

##### Node: Wait 30s Threads
- **Type / role:** Wait; processing delay.
- **Configuration:** 30 seconds.
- **Output:** Threads Publish

##### Node: Threads Publish
- **Type / role:** HTTP Request; publishes container.
- **Configuration:** POST `/threads_publish` with `creation_id={{Threads Create Container.id}}`
- **Output:** Format Threads Result

##### Node: Format Threads Result
- **Type / role:** Set; success if `id` exists and no error. postUrl left blank.
- **Output:** Loop Over Platforms
- **Edge cases:** Threads public URL derivation is not implemented; you may want to build it once API provides permalink.

---

### Block 8 — Aggregate Results, Update Sheet, Send Report, Continue Outer Loop
**Overview:** Collects all formatted per-platform results, computes final status/summary, updates the sheet, emails the report, then continues to the next sheet row.  
**Nodes involved:**  
- Aggregate Platform Results  
- Format Results  
- Update Sheet - Published  
- Send Report Email

#### Node: Aggregate Platform Results
- **Type / role:** Aggregate; collects all platform result items into a single array.
- **Configuration:** `aggregateAllItemData`
- **Input:** Loop Over Platforms (done output)
- **Output:** Format Results
- **Edge cases:** If some platform items never return (unknown platform routed nowhere), aggregation will exclude them.

#### Node: Format Results
- **Type / role:** Set; computes overall status and summary text.
- **Key logic:**
  - `status`:
    - `approved-no-platforms` if `data.length===0`
    - `published` if all successes
    - `failed` if zero successes
    - else `partial`
  - `detailedResults` is a newline-joined list (uses checkmarks in text).
  - Carries forward `rowNumber`, `videoTitle`, `platforms`, `videoUrl`.
- **Input:** Aggregate Platform Results
- **Output:** Update Sheet - Published
- **Edge cases:**
  - Uses glyphs (✅/❌) which are fine in email/text fields but may not be desired in Sheets.
  - If `data` missing, expressions fail (aggregate should always output `data`).

#### Node: Update Sheet - Published
- **Type / role:** Google Sheets update; finalizes row with results.
- **Configuration:**
  - Match on `row_number`
  - Writes:
    - `STATUS={{status}}`
    - `VIDEO_URL={{videoUrl}}`
    - `PUBLISHED_AT={{ total===0 ? '' : now }}`
    - `PUBLISHING_RESULTS={{summary}}`
  - continue on error
- **Input:** Format Results
- **Output:** Send Report Email
- **Edge cases:** Sheet must contain columns `VIDEO_URL`, `PUBLISHED_AT`, `PUBLISHING_RESULTS` or the node mapping may fail depending on n8n Sheets node behavior.

#### Node: Send Report Email
- **Type / role:** Gmail; sends a final report per video.
- **Configuration:**
  - To: `user@example.com` placeholder
  - Subject varies: `[No Platforms]`, `[Published]`, `[Partial]`
  - HTML body includes summary + detailed per-platform results + link to generated video
- **Input:** Update Sheet - Published
- **Output:** Loop Over Rows
- **Failures:** Gmail quota/sending restrictions; HTML rendering differences.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Documentation / overview |  |  | # VEED AI Talking Head Video Generator … **Node:** n8n-nodes-veed **Author:** VEED.io |
| Sticky Note - Setup | Sticky Note | Setup instructions |  |  | Setup Instructions (Sheet columns, credentials, placeholders, social setup) |
| Sticky Note - Trigger | Sticky Note | Explains trigger/loop design |  |  | 1. Trigger & Input (STATUS=new, loop one at a time) |
| Sticky Note - TTS | Sticky Note | Explains TTS block |  |  | 2. Text-to-Speech Generation (OpenAI/ElevenLabs, upload to Drive) |
| Sticky Note - VEED | Sticky Note | Explains VEED generation |  |  | 3. VEED Video Generation (Fabric 1.0, errors → sheet+email) |
| Sticky Note - Approval | Sticky Note | Explains approval gate |  |  | 4. Approval Workflow (24h timeout, rejected → sheet) |
| Sticky Note - Publishing | Sticky Note | Explains publishing design |  |  | 5. Multi-Platform Publishing (sequential per platform, routing, aggregation) |
| Sticky Note - Report | Sticky Note | Explains final status/report |  |  | 6. Status Update & Report (status values list) |
| When clicking 'Test workflow' | Manual Trigger | Manual entry point | — | Get New Video Requests | 1. Trigger & Input… |
| Check for New Videos | Schedule Trigger | Scheduled entry point | — | Get New Video Requests | 1. Trigger & Input… |
| Get New Video Requests | Google Sheets | Read sheet rows | When clicking 'Test workflow', Check for New Videos | Filter Status New | 1. Trigger & Input… |
| Filter Status New | Filter | Keep STATUS=new | Get New Video Requests | Loop Over Rows | 1. Trigger & Input… |
| Loop Over Rows | Split In Batches | Sequential row processing | Filter Status New; (loop-backs) | Mark as Processing (index 1) | 1. Trigger & Input… |
| Mark as Processing | Google Sheets | Lock row: STATUS=processing | Loop Over Rows | Prepare Video Parameters | 1. Trigger & Input… |
| Prepare Video Parameters | Set | Normalize per-row settings | Mark as Processing | TTS Provider? | 1. Trigger & Input… |
| TTS Provider? | IF | Route to OpenAI vs ElevenLabs | Prepare Video Parameters | ElevenLabs TTS, OpenAI TTS | 2. Text-to-Speech Generation… |
| ElevenLabs TTS | HTTP Request | ElevenLabs TTS (binary audio) | TTS Provider? (true) | Merge TTS | 2. Text-to-Speech Generation… |
| OpenAI TTS | OpenAI (LangChain) | OpenAI TTS (audio) | TTS Provider? (false) | Merge TTS | 2. Text-to-Speech Generation… |
| Merge TTS | Merge | Unify TTS outputs | ElevenLabs TTS, OpenAI TTS | Upload to Drive | 2. Text-to-Speech Generation… |
| Upload to Drive | Google Drive | Upload audio file | Merge TTS | Make Public | 2. Text-to-Speech Generation… |
| Make Public | Google Drive | Share audio publicly | Upload to Drive | Set Audio URL | 2. Text-to-Speech Generation… |
| Set Audio URL | Set | Build public audio download URL | Make Public | VEED Generate Video | 2. Text-to-Speech Generation… |
| VEED Generate Video | VEED | Generate talking-head video | Set Audio URL | VEED Error? | 3. VEED Video Generation… |
| VEED Error? | IF | Detect VEED failure | VEED Generate Video | Update Sheet - Error, Format Email | 3. VEED Video Generation… |
| Update Sheet - Error | Google Sheets | Mark STATUS=error | VEED Error? (true) | Send Error Email | 3. VEED Video Generation… |
| Send Error Email | Gmail | Notify generation failure | Update Sheet - Error | Loop Over Rows | 3. VEED Video Generation… |
| Format Email | Set | Create approval email HTML + store video URL | VEED Error? (false) | Send Approval Email | 4. Approval Workflow… |
| Send Approval Email | Gmail (sendAndWait) | Approval gate | Format Email | Approved? | 4. Approval Workflow… |
| Approved? | IF | Branch approved vs rejected | Send Approval Email | Prepare Publish Data, Log Rejection | 4. Approval Workflow… |
| Log Rejection | Set | Mark rejection metadata | Approved? (false) | Update Sheet - Rejected | 4. Approval Workflow… |
| Update Sheet - Rejected | Google Sheets | Mark STATUS=rejected | Log Rejection | Loop Over Rows | 4. Approval Workflow… |
| Prepare Publish Data | Set | Prepare publish payload + parse platforms | Approved? (true) | Download Video | 5. Multi-Platform Publishing… |
| Download Video | HTTP Request | Download generated video as binary | Prepare Publish Data | Split Out Platforms | 5. Multi-Platform Publishing… |
| Split Out Platforms | Split Out | One item per platform (keep binary) | Download Video | Loop Over Platforms | 5. Multi-Platform Publishing… |
| Loop Over Platforms | Split In Batches | Sequential platform loop | Split Out Platforms; (formatters) | Platform Router (index 1), Aggregate Platform Results (done) | 5. Multi-Platform Publishing… |
| Platform Router | Switch | Route to platform publisher | Loop Over Platforms | Instagram Create / YouTube Upload / Facebook Post / Telegram Post / Threads Create Container | 5. Multi-Platform Publishing… |
| Instagram Create | HTTP Request | Create IG Reels container | Platform Router (instagram) | Wait 90s | 5. Multi-Platform Publishing… |
| Wait 90s | Wait | Delay for IG processing | Instagram Create | Instagram Publish | 5. Multi-Platform Publishing… |
| Instagram Publish | HTTP Request | Publish IG container | Wait 90s | Format Instagram Result | 5. Multi-Platform Publishing… |
| YouTube Upload | YouTube | Upload to YouTube | Platform Router (youtube) | Format YouTube Result | 5. Multi-Platform Publishing… |
| Facebook Post | Facebook Graph API | Upload to FB Page | Platform Router (facebook) | Format Facebook Result | 5. Multi-Platform Publishing… |
| Telegram Post | Telegram | Post video to Telegram | Platform Router (telegram) | Format Telegram Result | 5. Multi-Platform Publishing… |
| Threads Create Container | HTTP Request | Create Threads container | Platform Router (threads) | Wait 30s Threads | 5. Multi-Platform Publishing… |
| Wait 30s Threads | Wait | Delay for Threads processing | Threads Create Container | Threads Publish | 5. Multi-Platform Publishing… |
| Threads Publish | HTTP Request | Publish Threads container | Wait 30s Threads | Format Threads Result | 5. Multi-Platform Publishing… |
| Format Instagram Result | Set | Standardize IG result | Instagram Publish | Loop Over Platforms | 5. Multi-Platform Publishing… |
| Format YouTube Result | Set | Standardize YT result | YouTube Upload | Loop Over Platforms | 5. Multi-Platform Publishing… |
| Format Facebook Result | Set | Standardize FB result | Facebook Post | Loop Over Platforms | 5. Multi-Platform Publishing… |
| Format Telegram Result | Set | Standardize Telegram result | Telegram Post | Loop Over Platforms | 5. Multi-Platform Publishing… |
| Format Threads Result | Set | Standardize Threads result | Threads Publish | Loop Over Platforms | 5. Multi-Platform Publishing… |
| Aggregate Platform Results | Aggregate | Collect all platform results | Loop Over Platforms (done) | Format Results | 6. Status Update & Report… |
| Format Results | Set | Compute overall status/summary | Aggregate Platform Results | Update Sheet - Published | 6. Status Update & Report… |
| Update Sheet - Published | Google Sheets | Write final status + URLs | Format Results | Send Report Email | 6. Status Update & Report… |
| Send Report Email | Gmail | Send publishing report | Update Sheet - Published | Loop Over Rows | 6. Status Update & Report… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet (queue)**
   1. Create a spreadsheet with a tab (e.g., first tab `gid=0`).
   2. Add columns (minimum required by workflow):
      - `AVATAR_IMAGE_URL`, `NARRATION_TEXT`, `TTS_PROVIDER`, `VOICE_ID`
      - `VIDEO_TITLE`, `VIDEO_DESCRIPTION`, `PLATFORMS`
      - `ASPECT_RATIO`, `RESOLUTION`
      - `STATUS`
      - Ensure n8n can access a `row_number` field (n8n Google Sheets node often provides it automatically; if not, add your own unique row id column and adapt matching logic).

2. **Create credentials in n8n**
   1. **Google OAuth2** credential (used by Google Sheets, Drive, Gmail nodes).
   2. **OpenAI** credential for the OpenAI node (TTS).
   3. **ElevenLabs** credential:
      - Create an **HTTP Header Auth** (generic) credential with header `xi-api-key: <YOUR_KEY>` (ElevenLabs uses `xi-api-key`).
   4. **Fal.ai / VEED** credential:
      - Install/enable `n8n-nodes-veed` and configure its credential per the node’s requirements (Fal.ai key).
   5. Optional publishing credentials:
      - **Facebook Graph API** credential (for Instagram + Facebook publishing).
      - **YouTube OAuth2** (YouTube node).
      - **Telegram** credential (Bot token).
      - **HTTP Query Auth** credential for Threads: `access_token=<THREADS_TOKEN>`.

3. **Add triggers**
   1. Add **Manual Trigger** node named `When clicking 'Test workflow'`.
   2. Add **Schedule Trigger** node named `Check for New Videos`, set interval to **every 15 minutes**.

4. **Read and filter rows**
   1. Add **Google Sheets** node `Get New Video Requests`:
      - Operation: read/get all rows (default “Read” behavior).
      - Document ID: your spreadsheet ID.
      - Sheet: select the correct tab (e.g., `gid=0`).
      - Turn on **Continue On Fail** if you want to match the workflow behavior.
   2. Connect both triggers → `Get New Video Requests`.
   3. Add **Filter** node `Filter Status New`:
      - Condition: `STATUS` equals `new` (string).
   4. Connect `Get New Video Requests` → `Filter Status New`.

5. **Sequential row loop**
   1. Add **Split In Batches** node `Loop Over Rows` (default settings).
   2. Connect `Filter Status New` → `Loop Over Rows`.
   3. You will later connect end-of-row nodes back into `Loop Over Rows` to continue.

6. **Mark row processing**
   1. Add **Google Sheets** node `Mark as Processing`:
      - Operation: Update
      - Matching column: `row_number`
      - Values: `STATUS = "processing"`, `row_number = {{$('Loop Over Rows').item.json.row_number}}`
   2. Connect `Loop Over Rows` **output 1** (the “batch item” output) → `Mark as Processing`.

7. **Prepare normalized parameters**
   1. Add **Set** node `Prepare Video Parameters` with fields:
      - `imageUrl`, `narrationText`, `ttsProvider` (default openai), `voiceId` (default alloy)
      - `videoTitle`, `videoDescription`
      - `platforms` default empty string
      - `aspectRatio` default 9:16
      - `resolution` default 720p
      - `rowNumber` from `row_number`
   2. Connect `Mark as Processing` → `Prepare Video Parameters`.

8. **TTS routing**
   1. Add **IF** node `TTS Provider?`:
      - Condition: expression `{{$json.ttsProvider === 'elevenlabs'}}` is true.
   2. Connect `Prepare Video Parameters` → `TTS Provider?`.

9. **ElevenLabs TTS branch**
   1. Add **HTTP Request** node `ElevenLabs TTS`:
      - Method: POST
      - URL: `https://api.elevenlabs.io/v1/text-to-speech/{{$json.voiceId}}`
      - Body: JSON with `text`, `model_id=eleven_multilingual_v2`, voice_settings
      - Response: File (binary)
      - Auth: Generic → HTTP Header Auth (xi-api-key)
      - Continue On Fail enabled (to match)
   2. Connect `TTS Provider?` true output → `ElevenLabs TTS`.

10. **OpenAI TTS branch**
   1. Add **OpenAI** node `OpenAI TTS` (the `@n8n/n8n-nodes-langchain.openAi` audio resource):
      - Resource: Audio / TTS
      - Input: `{{$json.narrationText}}`
      - Voice: `{{$json.voiceId}}`
      - Continue On Fail enabled
   2. Connect `TTS Provider?` false output → `OpenAI TTS`.

11. **Merge TTS outputs**
   1. Add **Merge** node `Merge TTS`:
      - Mode: Combine
      - Include unpaired: true
      - Combine by position
   2. Connect `ElevenLabs TTS` → `Merge TTS` input 1 and `OpenAI TTS` → `Merge TTS` input 2 (either ordering works if you replicate the same combine-by-position pattern).

12. **Upload audio to Drive + public link**
   1. Add **Google Drive** node `Upload to Drive`:
      - Upload binary data as a file
      - File name: `{{$now.format('yyyyMMdd-HHmmss')}}-audio.mp3`
      - Folder ID: your audio folder
      - Continue On Fail enabled
   2. Add **Google Drive** node `Make Public`:
      - Operation: Share
      - File ID: `{{$json.id}}`
      - Permission: anyone / reader
   3. Add **Set** node `Set Audio URL`:
      - `audioUrl = https://drive.google.com/uc?export=download&id={{$('Upload to Drive').item.json.id}}`
   4. Connect: `Merge TTS` → `Upload to Drive` → `Make Public` → `Set Audio URL`.

13. **Generate VEED video**
   1. Add **VEED** node `VEED Generate Video`:
      - audioUrl: `{{$json.audioUrl}}`
      - imageUrl: `{{$('Prepare Video Parameters').item.json.imageUrl}}`
      - resolution/aspectRatio from Prepare Video Parameters
      - Timeout: 15 (as configured in the workflow)
      - Continue On Fail enabled
   2. Connect `Set Audio URL` → `VEED Generate Video`.

14. **VEED error check**
   1. Add **IF** node `VEED Error?`:
      - OR conditions:
        - `{{$json.error}} exists`
        - `{{$json.video?.url}} not exists`
   2. Connect `VEED Generate Video` → `VEED Error?`.

15. **Error branch: update sheet + email + continue**
   1. Add **Google Sheets** node `Update Sheet - Error`:
      - Update by `row_number`
      - Set `STATUS="error"`
   2. Add **Gmail** node `Send Error Email`:
      - To: your email
      - Subject/body similar to the provided HTML (include VEED error and row details)
   3. Connect: `VEED Error?` true → `Update Sheet - Error` → `Send Error Email` → `Loop Over Rows`.

16. **Success branch: approval email**
   1. Add **Set** node `Format Email`:
      - Create `emailBody` HTML including preview link `{{$('VEED Generate Video').item.json.video.url}}`
      - Set `generatedVideoUrl` to that URL
   2. Add **Gmail** node `Send Approval Email`:
      - Operation: sendAndWait
      - Wait time: 24 hours
      - Two-button approval (double)
   3. Add **IF** node `Approved?` checking `{{$json.data.approved}}` is true.
   4. Connect: `VEED Error?` false → `Format Email` → `Send Approval Email` → `Approved?`.

17. **Rejection branch**
   1. Add **Set** node `Log Rejection` (optional metadata).
   2. Add **Google Sheets** node `Update Sheet - Rejected` (STATUS=rejected by row_number).
   3. Connect: `Approved?` false → `Log Rejection` → `Update Sheet - Rejected` → `Loop Over Rows`.

18. **Publishing preparation**
   1. Add **Set** node `Prepare Publish Data`:
      - `videoUrl`, `title`, `description`
      - `platformList = (platforms||'').toLowerCase().split(',').map(trim)`
   2. Add **HTTP Request** node `Download Video` (response: file/binary) using `videoUrl`.
   3. Add **Split Out** node `Split Out Platforms` splitting `platformList`, includeBinary=true.
   4. Add **Split In Batches** node `Loop Over Platforms`:
      - batchSize=1
      - reset expression as in workflow (or implement a safer reset strategy per row).
   5. Add **Switch** node `Platform Router` with rules for instagram/youtube/facebook/telegram/threads; value `{{$json.platformList}}`.
   6. Connect: `Approved?` true → `Prepare Publish Data` → `Download Video` → `Split Out Platforms` → `Loop Over Platforms` (item output) → `Platform Router`.

19. **Add publishing nodes per platform**
   - **Instagram**: HTTP Request `Instagram Create` → Wait `Wait 90s` → HTTP Request `Instagram Publish` → Set `Format Instagram Result` → back to `Loop Over Platforms`.
   - **YouTube**: YouTube node `YouTube Upload` → Set `Format YouTube Result` → back to `Loop Over Platforms`.
   - **Facebook**: Facebook Graph API node `Facebook Post` → Set `Format Facebook Result` → back to `Loop Over Platforms`.
   - **Telegram**: Telegram node `Telegram Post` → Set `Format Telegram Result` → back to `Loop Over Platforms`.
   - **Threads**: HTTP Request `Threads Create Container` → Wait `Wait 30s Threads` → HTTP Request `Threads Publish` → Set `Format Threads Result` → back to `Loop Over Platforms`.

20. **Aggregate platform results and finalize**
   1. Add **Aggregate** node `Aggregate Platform Results` connected from `Loop Over Platforms` “done” output.
   2. Add **Set** node `Format Results` to compute final status + summary + detailedResults.
   3. Add **Google Sheets** node `Update Sheet - Published` updating status/video URL/timestamps.
   4. Add **Gmail** node `Send Report Email` and connect back to `Loop Over Rows`.
   5. Connect: `Aggregate Platform Results` → `Format Results` → `Update Sheet - Published` → `Send Report Email` → `Loop Over Rows`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Node: n8n-nodes-veed” | Workflow overview sticky note (requires VEED community node installed) |
| Fal.ai API Key required | Mentioned in overview/setup notes: https://fal.ai/dashboard/keys |
| OpenAI API keys | https://platform.openai.com/api-keys |
| ElevenLabs API keys | https://elevenlabs.io/app/settings/api-keys |
| Google Sheet ID / Folder ID placeholders must be replaced | Replace `YOUR_GOOGLE_SHEET_ID`, `YOUR_AUDIO_FOLDER_ID` in nodes |
| Social placeholders must be replaced | `YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID`, `YOUR_FACEBOOK_PAGE_ID`, `YOUR_TELEGRAM_CHAT_ID`, `YOUR_THREADS_USER_ID` |
| Threads auth uses HTTP Query Auth | Credential should pass `access_token` as query parameter (per sticky note) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.