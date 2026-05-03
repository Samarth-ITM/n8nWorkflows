Poll KIE.ai video generation status and send results via Telegram

https://n8nworkflows.xyz/workflows/poll-kie-ai-video-generation-status-and-send-results-via-telegram-12684


# Poll KIE.ai video generation status and send results via Telegram

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow polls **KIE.ai** video-generation tasks until completion (or timeout), then **downloads the video**, **uploads it to S3**, **retrieves session context from Redis**, and **delivers a preview to the user via Telegram** with inline action buttons for publishing and/or extending.

**Primary use case:** As an async “poller” invoked by a main workflow that starts a video generation job on KIE.ai and needs a reliable status-check loop + delivery mechanism.

**Supported models & APIs:**
- **Veo (Veo 3.1 / variants):** via `statusEndpoint` default `/api/v1/veo/record-info` using `successFlag` semantics.
- **Market-style APIs (Sora 2, Seedance):** via `modelType: "market"` using `state/status` semantics and multiple possible output URL locations.

### Logical Blocks
1. **1.1 Webhook Intake & Immediate ACK**  
   Receives polling requests (task/session identifiers), normalizes input, responds immediately.
2. **1.2 Status Polling**  
   Waits 60 seconds, calls KIE.ai status endpoint, parses completion/failure and extracts final video URL.
3. **1.3 Retry / Timeout**  
   If not completed: re-invokes the webhook (up to 15 attempts). If exhausted: notifies user via Telegram.
4. **1.4 Video Download & S3 Upload**  
   Downloads the resulting video file from the extracted URL and uploads to S3.
5. **1.5 Extend Flow Decision + Merge Options UI**  
   If the request is an “extend” flow, sends Telegram merge options; otherwise proceeds normally.
6. **1.6 Session Retrieval, Metadata Build, Session Update**  
   Loads session JSON from Redis, merges it with video URLs, creates metadata text file, uploads metadata, updates Redis session.
7. **1.7 Telegram Delivery**  
   Downloads the S3 video for Telegram, then sends a preview with model-dependent button layout (Veo vs other).

---

## 2. Block-by-Block Analysis

### 2.1 Webhook Intake & Immediate ACK

**Overview:** Accepts POST requests that contain the polling context (sessionId, taskId, chatId, attempt, model info). It normalizes payload differences and returns an immediate JSON response so callers don’t wait for polling to complete.

**Nodes Involved:**
- Poll Trigger
- Parse Input
- Respond OK

#### Node: Poll Trigger
- **Type / role:** `Webhook` (entrypoint). Receives polling requests.
- **Key config:**
  - **Path:** `video-poll`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (response is produced by “Respond OK”)
  - **Raw body:** enabled (`options.rawBody = true`), so `body` may arrive as a JSON string.
- **Outputs:** to **Parse Input**
- **Edge cases / failures:**
  - Invalid JSON body (when rawBody string cannot be parsed later).
  - Missing required fields (taskId/sessionId/chatId) causing downstream failures or wrong notifications.
- **Version notes:** Webhook node `typeVersion: 2`.

#### Node: Parse Input
- **Type / role:** `Code` node to normalize incoming webhook shape.
- **What it does:**
  - Detects whether `input.body` is a string (rawBody) or object.
  - Returns normalized JSON with defaults:
    - `selectedModel` default `veo3_fast`
    - `modelType` default `veo`
    - `statusEndpoint` default `/api/v1/veo/record-info`
    - `attempt` default `1`
    - `isExtend` default `false`
    - `parentSessionId` default `null`
- **Key variables produced:** `sessionId, taskId, chatId, messageId, selectedModel, modelType, statusEndpoint, attempt, isExtend, parentSessionId`
- **Outputs:** to **Respond OK**
- **Edge cases / failures:**
  - `JSON.parse` fails when `input.body` is not valid JSON string.
  - Required fields missing → later HTTP/Telegram/Redis steps may fail.
- **Version notes:** Code node `typeVersion: 2`.

#### Node: Respond OK
- **Type / role:** `Respond to Webhook` to immediately acknowledge request.
- **Key config:**
  - **On error:** `continueRegularOutput` (does not stop workflow if respond fails)
  - Response JSON: `{ success: true, attempt: $json.attempt }`
  - Uses expression: `={{ JSON.stringify({ success: true, attempt: $json.attempt }) }}`
- **Inputs:** from Parse Input
- **Outputs:** to **Wait 1 Minute**
- **Edge cases / failures:**
  - Webhook response already sent / invalid response formatting (rare).
- **Version notes:** `typeVersion: 1.1`.

---

### 2.2 Status Polling

**Overview:** Waits one minute between polls, then calls KIE.ai status endpoint. Parses response into a normalized status (`processing|completed|failed`) and extracts the best candidate `videoUrl`.

**Nodes Involved:**
- Wait 1 Minute
- Check KIE Status
- Parse Status
- Completed?

#### Node: Wait 1 Minute
- **Type / role:** `Wait` to delay polling (rate limiting / avoid hammering).
- **Key config:** `amount: 60` (seconds)
- **Inputs:** from Respond OK
- **Outputs:** to Check KIE Status
- **Edge cases / failures:** execution resumption issues if n8n instance restarts; long waits depend on n8n persistence.
- **Version notes:** `typeVersion: 1.1`.

#### Node: Check KIE Status
- **Type / role:** `HTTP Request` to KIE.ai.
- **Key config:**
  - **URL:** `https://api.kie.ai{{ $('Parse Input').first().json.statusEndpoint }}`
  - **Query param:** `taskId={{ $('Parse Input').first().json.taskId }}`
  - **Auth:** Generic credential type → `httpHeaderAuth` (KIE.ai API key in headers)
  - **On error:** `continueRegularOutput` (workflow continues even if HTTP fails)
- **Inputs:** from Wait 1 Minute
- **Outputs:** to Parse Status
- **Edge cases / failures:**
  - Auth error (401/403) if API key missing/invalid.
  - Non-2xx responses, timeouts, rate limits.
  - If it errors and continues, Parse Status may receive an error-shaped payload and mis-detect status.
- **Version notes:** `typeVersion: 4.2`.

#### Node: Parse Status
- **Type / role:** `Code` node to unify status semantics across model types.
- **Key logic:**
  - Reads `modelType` from Parse Input (`veo` default).
  - For `modelType === 'veo'`:
    - `successFlag`: `0=Generating, 1=Success, 2=Failed, 3=Generation Failed`
    - Extracts URL from:
      - `originUrls[0]` (preferred for 9:16) or `resultUrls[0]`
  - For `modelType === 'market'`:
    - `state/status` → completed if `success|completed|COMPLETED`
    - failed if `failed|FAILED|error`
    - Extracts `videoUrl` from multiple possible locations:
      - `data.videoUrl`, `data.output.video`, `data.result.video_url`
      - `data.resultJson` parsed JSON
      - `data.response` (string or object) parsed JSON
  - Outputs normalized fields:
    - `status`, `isCompleted`, `isFailed`, `videoUrl`, `rawResponse`
- **Inputs:** from Check KIE Status, plus references Parse Input.
- **Outputs:** to Completed?
- **Edge cases / failures:**
  - Unexpected response shape → `videoUrl` remains null even when completed.
  - `JSON.parse` on `resultJson` or `response` may fail; code catches in some places but not everywhere (notably parsing `data.response` assumes JSON if string).
  - If Check KIE Status continued after an error, statusResponse might not have `data` at all.
- **Version notes:** Code node `typeVersion: 2`.

#### Node: Completed?
- **Type / role:** `IF` branching on completion.
- **Condition:** `{{ $json.isCompleted }} == true`
- **True branch:** Get 1080p Video (continue pipeline)
- **False branch:** Can Retry? (retry pipeline)
- **Edge cases:** If `isCompleted` is undefined (bad parse), condition evaluates false and triggers retry/timeout path.
- **Version notes:** `typeVersion: 1`.

---

### 2.3 Retry / Timeout

**Overview:** If the task is still processing, the workflow re-triggers itself by calling its own webhook with `attempt + 1` until attempt 15. If it hits the limit, it informs the user on Telegram.

**Nodes Involved:**
- Can Retry?
- Retry Poll
- Send Timeout

#### Node: Can Retry?
- **Type / role:** `IF` gating retries.
- **Condition:** `{{ $json.attempt < 15 }} == true`
- **True branch:** Retry Poll
- **False branch:** Send Timeout
- **Edge cases:** attempt missing/non-numeric → expression may evaluate unexpectedly.
- **Version notes:** `typeVersion: 1`.

#### Node: Retry Poll
- **Type / role:** `HTTP Request` that re-invokes this workflow’s webhook.
- **Key config:**
  - **POST URL:** `https://n8n.mosaabyassir.xyz/webhook/video-poll` (must match your instance)
  - **Body:** JSON string created by expression:
    - includes `sessionId, taskId, chatId, messageId, selectedModel, modelType, statusEndpoint`
    - sets `attempt: $json.attempt + 1`
  - **specifyBody:** `json`
- **Inputs:** from Can Retry? (true path)
- **Outputs:** none further (this execution ends after enqueueing next poll)
- **Edge cases / failures:**
  - If the URL is wrong/unreachable, polling stops silently.
  - If n8n is behind auth or needs a production webhook URL vs test URL mismatch.
- **Version notes:** `typeVersion: 4.2`.

#### Node: Send Timeout
- **Type / role:** `Telegram` sendMessage to warn user.
- **Message includes:** attempt count and taskId; Markdown parse mode.
- **chatId:** `{{ $json.chatId }}`
- **Inputs:** from Can Retry? (false path)
- **Edge cases / failures:**
  - Telegram auth invalid.
  - chatId missing.
  - Markdown formatting issues (rare).
- **Version notes:** `typeVersion: 1.2`.

---

### 2.4 Video Download & S3 Upload

**Overview:** Once completed, the workflow finalizes the usable `videoUrl`, downloads the video binary, and uploads it to an S3 bucket as `videos/<sessionId>.mp4`.

**Nodes Involved:**
- Get 1080p Video
- Get Final URL
- Download Video
- Upload a file

#### Node: Get 1080p Video
- **Type / role:** `Code` node; currently pass-through placeholder.
- **Behavior:** returns `$input.all()` unchanged.
- **Inputs:** Completed? (true)
- **Outputs:** Get Final URL
- **Edge cases:** none significant.
- **Version notes:** `typeVersion: 2`.

#### Node: Get Final URL
- **Type / role:** `Code` node; ensures `isExtend` and `parentSessionId` are present.
- **Key behavior:**
  - Takes normalized status result from Parse Status.
  - Overrides/ensures:
    - `isExtend` from Parse Input
    - `parentSessionId` from Parse Input
- **Outputs:** Download Video
- **Edge cases:** If Parse Status didn’t produce `videoUrl`, download will fail.
- **Version notes:** `typeVersion: 2`.

#### Node: Download Video
- **Type / role:** `HTTP Request` to download the final video file.
- **Key config:**
  - **URL:** `{{ $json.videoUrl }}`
  - **Response format:** `file` (binary)
- **Outputs:** Upload a file
- **Edge cases / failures:**
  - `videoUrl` null/empty → request fails.
  - Large file timeouts/memory constraints.
  - Remote server denies hotlinking / signed URL expired.
- **Version notes:** `typeVersion: 4.2`.

#### Node: Upload a file
- **Type / role:** `S3` upload of the downloaded binary.
- **Key config:**
  - **Operation:** upload
  - **Bucket:** `shorts`
  - **Key:** `videos/{{ $('Get Final URL').first().json.sessionId }}.mp4`
- **Inputs:** from Download Video (binary)
- **Outputs:** Is Extend?
- **Edge cases / failures:**
  - S3 credentials/permissions (PutObject) missing.
  - Bucket/key region mismatch.
  - Large file multipart requirements (n8n S3 node handles this depending on backend).
- **Version notes:** `typeVersion: 1`.

---

### 2.5 Extend Flow Decision + Merge Options UI

**Overview:** If this polling run corresponds to an “extend video” request, the user is prompted in Telegram to choose how to merge. Regardless, execution converges via a Merge node to continue session enrichment.

**Nodes Involved:**
- Is Extend?
- Send Merge Options
- Skip Concat
- Merge After Concat

#### Node: Is Extend?
- **Type / role:** `IF` checks if this is an extension flow.
- **Condition:** `{{ $('Get Final URL').first().json.isExtend }} == true`
- **True branch:** Send Merge Options
- **False branch:** Skip Concat
- **Edge cases:** If Get Final URL not available (execution path issues), expression fails.
- **Version notes:** `typeVersion: 1`.

#### Node: Send Merge Options
- **Type / role:** `Telegram` sendMessage with inline keyboard (merge choice).
- **chatId:** `{{ $('Parse Input').first().json.chatId }}`
- **Buttons / callback_data:**
  - `am:<sessionId>` for “Auto Merge (Transloadit)”
  - `mm:<sessionId>` for “Get Video Links (Manual)”
- **Outputs:** to Merge After Concat (input 0)
- **Edge cases / failures:**
  - Depends on Parse Input being accessible in this execution.
  - If Telegram message fails, merge path may not execute as intended.
- **Version notes:** `typeVersion: 1.2`.

#### Node: Skip Concat
- **Type / role:** `Code` node; prepares minimal payload for non-extend case.
- **Output JSON:**
  - `sessionId`, `isExtend:false`, `videoUrl`, `rawResponse` (from Get Final URL)
- **Outputs:** Merge After Concat (input 1)
- **Version notes:** `typeVersion: 2`.

#### Node: Merge After Concat
- **Type / role:** `Merge` node to converge extend and non-extend paths.
- **Inputs:**
  - Input 0: from Send Merge Options
  - Input 1: from Skip Concat
- **Outputs:** Get Session
- **Edge cases:** If one side doesn’t emit items, merge behavior depends on merge mode (here defaults; in n8n Merge v2 default is typically “Append”/“Combine” depending on UI setting—verify in editor).
- **Version notes:** `typeVersion: 2`.

---

### 2.6 Session Retrieval, Metadata Build, Session Update

**Overview:** Reads the original session from Redis, merges it with the new video URLs and S3 URL, creates a metadata text file (as binary), uploads metadata to S3, then updates Redis session with `status: video_ready`.

**Nodes Involved:**
- Get Session
- Merge Session Data
- Create Metadata
- Upload Metadata
- Update Session

#### Node: Get Session
- **Type / role:** `Redis` GET.
- **Key:** `session:{{ $('Get Final URL').first().json.sessionId }}`
- **Credentials:** Redis connection (“Apello Version”)
- **Outputs:** Merge Session Data
- **Edge cases / failures:**
  - Key not found → Merge Session Data may fail parsing.
  - Redis connection/auth errors.
- **Version notes:** `typeVersion: 1`.

#### Node: Merge Session Data
- **Type / role:** `Code` node combining Redis session with current video info.
- **Key behaviors:**
  - Takes `mergeData` from `Merge After Concat`.
  - Parses Redis returned payload (`propertyName` fallback).
  - Constructs fixed S3 URL: `https://s3-api.apello.org/shorts/videos/<sessionId>.mp4`
  - Chooses `videoUrl` with fallbacks:
    1. `mergeData.videoUrl`
    2. `mergeData.rawResponse.data.response.originUrls[0]` or `resultUrls[0]`
    3. From `Get Final URL` and its rawResponse (try/catch)
  - Outputs enriched session with:
    - `videoUrl`, `rawResponse`, `s3Url`, `status:'video_ready'`, `updatedAt:<ISO>`
- **Outputs:** Create Metadata
- **Edge cases / failures:**
  - Redis value not JSON → `JSON.parse` fails.
  - Session missing expected fields used later (youtubeMetadata.title, chatId, etc.).
  - Hard-coded S3 public URL must match your actual S3 gateway/domain and bucket policy.
- **Version notes:** `typeVersion: 2`.

#### Node: Create Metadata
- **Type / role:** `Code` node generating a text metadata file and exposing it as binary.
- **Creates:**
  - A formatted metadata text block with:
    - session IDs/timestamps
    - original input summary
    - generation details (model, taskId, template, balance)
    - YouTube metadata (title/description/tags)
    - generated prompt
    - URLs (S3 + KIE)
  - Binary property `metadata` (base64-encoded) with mime `text/plain`
  - `metadataKey: videos/<sessionId>_metadata.txt`
- **Outputs:** Upload Metadata
- **Edge cases / failures:**
  - If fields are missing, it prints `N/A` (safe).
  - Very large descriptions/prompts could produce larger metadata objects (usually fine).
- **Version notes:** `typeVersion: 2`.

#### Node: Upload Metadata
- **Type / role:** `S3` upload of the metadata text file.
- **Key config:**
  - Bucket: `shorts`
  - File name: `{{ $json.metadataKey }}`
  - Binary property: `metadata`
- **Outputs:** Update Session
- **Edge cases:** S3 permission issues.
- **Version notes:** `typeVersion: 1`.

#### Node: Update Session
- **Type / role:** `Redis` SET updated session JSON with TTL.
- **Key config:**
  - Key: `session:{{ $json.sessionId }}`
  - Value: `JSON.stringify($json)`
  - TTL: 86400 seconds (1 day), `expire: true`
- **Outputs:** Download for Telegram
- **Edge cases:** Redis set fails; downstream will still try to send Telegram preview but session state won’t be updated.
- **Version notes:** `typeVersion: 1`.

---

### 2.7 Telegram Delivery

**Overview:** Fetches the uploaded MP4 from S3 as binary and sends it to Telegram. Chooses a different inline keyboard if the selected model is a Veo variant (adds “Extend Video (+8s)”).

**Nodes Involved:**
- Download for Telegram
- Is Veo Model?
- Send Video Preview (Veo)
- Send Video Preview (Other)

#### Node: Download for Telegram
- **Type / role:** `S3` download/get object to obtain binary for Telegram.
- **Key config:**
  - Bucket: `shorts`
  - Key: `videos/{{ $('Create Metadata').first().json.sessionId }}.mp4`
- **Outputs:** Is Veo Model?
- **Edge cases:** object missing (upload failed), permission denied.
- **Version notes:** `typeVersion: 1`.

#### Node: Is Veo Model?
- **Type / role:** `IF` decides which Telegram message template to use.
- **Condition:** `{{ $('Create Metadata').first().json.selectedModel?.startsWith('veo') }} == true`
- **True:** Send Video Preview (Veo)
- **False:** Send Video Preview (Other)
- **Edge cases:** selectedModel missing → condition false → “Other” path.
- **Version notes:** `typeVersion: 1`.

#### Node: Send Video Preview (Veo)
- **Type / role:** `Telegram` sendVideo with inline keyboard and caption.
- **Key config:**
  - chatId: `{{ $('Create Metadata').first().json.chatId }}`
  - binaryData: true (uses binary from previous node)
  - caption (Markdown): includes `youtubeMetadata.title`
  - Buttons (callback_data):
    - `publish:<sessionId>` (YouTube)
    - `tiktok:<sessionId>`
    - `instagram:<sessionId>`
    - `publishall:<sessionId>`
    - `extend:<sessionId>` (only on Veo path)
    - `cancel:<sessionId>`
- **Edge cases / failures:**
  - Telegram file size limits (videos may exceed allowed size for bots).
  - Missing chatId or missing youtubeMetadata title.
- **Version notes:** `typeVersion: 1.2`.

#### Node: Send Video Preview (Other)
- **Type / role:** `Telegram` sendVideo for non-Veo models (no extend button).
- **Same as Veo** but without “Extend Video (+8s)”.
- **Version notes:** `typeVersion: 1.2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main | Sticky Note | Overall workflow explanation + setup steps | — | — | ## How it works … (includes setup steps and requirement to update “Retry Poll” webhook URL) |
| Webhook Trigger | Sticky Note | Block label: Webhook & Parse | — | — | **Webhook & Parse** — Receives polling request, parses input data, responds OK immediately |
| Status Polling | Sticky Note | Block label: Status Polling | — | — | **Status Polling** — Waits 1 min, checks KIE.ai status, parses response for completion |
| Retry Logic | Sticky Note | Block label: Retry Logic | — | — | **Retry Logic** — Retries up to 15x or sends timeout message |
| Video Download | Sticky Note | Block label: Video Processing | — | — | **Video Processing** — Downloads from KIE.ai, uploads to S3, checks if extend flow |
| Session Storage | Sticky Note | Block label: Session & Metadata | — | — | **Session & Metadata** — Retrieves session from Redis, creates metadata, updates status |
| Telegram Delivery | Sticky Note | Block label: Telegram Delivery | — | — | **Telegram Delivery** — Sends video preview with publish action buttons |
| Extend Flow | Sticky Note | Block label: Extend Flow | — | — | **Extend Flow** — Add Merge Option if its an extend request, if initial request skip. |
| Poll Trigger | Webhook | Entry point webhook | — | Parse Input | **Webhook & Parse** — Receives polling request, parses input data, responds OK immediately |
| Parse Input | Code | Normalize webhook payload & defaults | Poll Trigger | Respond OK | **Webhook & Parse** — Receives polling request, parses input data, responds OK immediately |
| Respond OK | Respond to Webhook | Immediate ACK to caller | Parse Input | Wait 1 Minute | **Webhook & Parse** — Receives polling request, parses input data, responds OK immediately |
| Wait 1 Minute | Wait | Delay before polling | Respond OK | Check KIE Status | **Status Polling** — Waits 1 min, checks KIE.ai status, parses response for completion |
| Check KIE Status | HTTP Request | Call KIE.ai status endpoint | Wait 1 Minute | Parse Status | **Status Polling** — Waits 1 min, checks KIE.ai status, parses response for completion |
| Parse Status | Code | Interpret API response, extract `videoUrl` | Check KIE Status | Completed? | **Status Polling** — Waits 1 min, checks KIE.ai status, parses response for completion |
| Completed? | IF | Branch on completion | Parse Status | Get 1080p Video; Can Retry? | **Status Polling** — Waits 1 min, checks KIE.ai status, parses response for completion |
| Can Retry? | IF | Retry gating (<15) | Completed? (false) | Retry Poll; Send Timeout | **Retry Logic** — Retries up to 15x or sends timeout message |
| Retry Poll | HTTP Request | Reinvoke webhook with attempt+1 | Can Retry? (true) | — | **Retry Logic** — Retries up to 15x or sends timeout message |
| Send Timeout | Telegram | Notify user of timeout | Can Retry? (false) | — | **Retry Logic** — Retries up to 15x or sends timeout message |
| Get 1080p Video | Code | Pass-through placeholder | Completed? (true) | Get Final URL | **Video Processing** — Downloads from KIE.ai, uploads to S3, checks if extend flow |
| Get Final URL | Code | Ensure final url + extend flags | Get 1080p Video | Download Video | **Video Processing** — Downloads from KIE.ai, uploads to S3, checks if extend flow |
| Download Video | HTTP Request | Download mp4 as binary | Get Final URL | Upload a file | **Video Processing** — Downloads from KIE.ai, uploads to S3, checks if extend flow |
| Upload a file | S3 | Upload mp4 to S3 | Download Video | Is Extend? | **Video Processing** — Downloads from KIE.ai, uploads to S3, checks if extend flow |
| Is Extend? | IF | Extend vs normal flow | Upload a file | Send Merge Options; Skip Concat | **Extend Flow** — Add Merge Option if its an extend request, if initial request skip. |
| Send Merge Options | Telegram | Ask user how to merge extended video | Is Extend? (true) | Merge After Concat | **Extend Flow** — Add Merge Option if its an extend request, if initial request skip. |
| Skip Concat | Code | Minimal data for non-extend flow | Is Extend? (false) | Merge After Concat | **Extend Flow** — Add Merge Option if its an extend request, if initial request skip. |
| Merge After Concat | Merge | Converge extend/non-extend | Send Merge Options; Skip Concat | Get Session | **Extend Flow** — Add Merge Option if its an extend request, if initial request skip. |
| Get Session | Redis | Load session JSON from Redis | Merge After Concat | Merge Session Data | **Session & Metadata** — Retrieves session from Redis, creates metadata, updates status |
| Merge Session Data | Code | Combine session + URLs + S3 URL | Get Session | Create Metadata | **Session & Metadata** — Retrieves session from Redis, creates metadata, updates status |
| Create Metadata | Code | Build text metadata + binary | Merge Session Data | Upload Metadata | **Session & Metadata** — Retrieves session from Redis, creates metadata, updates status |
| Upload Metadata | S3 | Upload metadata txt to S3 | Create Metadata | Update Session | **Session & Metadata** — Retrieves session from Redis, creates metadata, updates status |
| Update Session | Redis | Persist updated session (TTL 1 day) | Upload Metadata | Download for Telegram | **Session & Metadata** — Retrieves session from Redis, creates metadata, updates status |
| Download for Telegram | S3 | Fetch mp4 from S3 as binary | Update Session | Is Veo Model? | **Telegram Delivery** — Sends video preview with publish action buttons |
| Is Veo Model? | IF | Choose Veo vs Other UI | Download for Telegram | Send Video Preview (Veo); Send Video Preview (Other) | **Telegram Delivery** — Sends video preview with publish action buttons |
| Send Video Preview (Veo) | Telegram | Send video + buttons incl extend | Is Veo Model? (true) | — | **Telegram Delivery** — Sends video preview with publish action buttons |
| Send Video Preview (Other) | Telegram | Send video + buttons (no extend) | Is Veo Model? (false) | — | **Telegram Delivery** — Sends video preview with publish action buttons |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Poll video generation status from KIE.ai and deliver via Telegram*
   - Ensure it is **Active** when ready (for production webhooks).

2. **Add Webhook entrypoint**
   - Node: **Webhook**
   - Name: `Poll Trigger`
   - Method: `POST`
   - Path: `video-poll`
   - Options: enable **Raw Body**
   - Response mode: **Using ‘Respond to Webhook’ node**

3. **Add input normalization**
   - Node: **Code** named `Parse Input`
   - Paste logic to:
     - parse `input.body` if string
     - return JSON with: `sessionId, taskId, chatId, messageId, selectedModel (default veo3_fast), modelType (default veo), statusEndpoint (default /api/v1/veo/record-info), attempt (default 1), isExtend (default false), parentSessionId (default null)`
   - Connect: `Poll Trigger` → `Parse Input`

4. **Add immediate webhook response**
   - Node: **Respond to Webhook** named `Respond OK`
   - Respond with: JSON
   - Body expression: `{{ JSON.stringify({ success: true, attempt: $json.attempt }) }}`
   - Set **On Error** to *Continue Regular Output*
   - Connect: `Parse Input` → `Respond OK`

5. **Add wait**
   - Node: **Wait** named `Wait 1 Minute`
   - Amount: `60` seconds
   - Connect: `Respond OK` → `Wait 1 Minute`

6. **Add KIE.ai status request**
   - Node: **HTTP Request** named `Check KIE Status`
   - Method: GET (default)
   - URL expression: `https://api.kie.ai{{ $('Parse Input').first().json.statusEndpoint }}`
   - Add query param:
     - `taskId` = `{{ $('Parse Input').first().json.taskId }}`
   - Authentication: **Header Auth** (Generic Credential → HTTP Header Auth)
     - Create credential (example): `KieAI - Header Auth`
     - Set header key/value per KIE.ai requirements (e.g., `Authorization: Bearer <API_KEY>` or the exact header name KIE.ai expects).
   - Set **On Error** to *Continue Regular Output*
   - Connect: `Wait 1 Minute` → `Check KIE Status`

7. **Parse status response**
   - Node: **Code** named `Parse Status`
   - Implement logic:
     - if `modelType` is `veo`: use `data.successFlag`, extract `originUrls[0]` or `resultUrls[0]`
     - if `modelType` is `market`: use `data.state/status`, extract `videoUrl` from multiple possible fields, parse `resultJson`/`response` when needed
     - output: `status, isCompleted, isFailed, videoUrl, rawResponse` plus original input data
   - Connect: `Check KIE Status` → `Parse Status`

8. **Branch on completion**
   - Node: **IF** named `Completed?`
   - Condition: Boolean → `{{ $json.isCompleted }}` equals `true`
   - Connect: `Parse Status` → `Completed?`

9. **Completion path: finalize URL + download**
   - Add **Code** `Get 1080p Video` (pass-through)
     - Connect: `Completed? (true)` → `Get 1080p Video`
   - Add **Code** `Get Final URL`
     - Merge/ensure `isExtend` and `parentSessionId` from `Parse Input`
     - Connect: `Get 1080p Video` → `Get Final URL`
   - Add **HTTP Request** `Download Video`
     - URL: `{{ $json.videoUrl }}`
     - Response: **File**
     - Connect: `Get Final URL` → `Download Video`

10. **Upload video to S3**
    - Node: **S3** named `Upload a file`
    - Operation: **Upload**
    - Bucket: `shorts`
    - Key: `videos/{{ $('Get Final URL').first().json.sessionId }}.mp4`
    - Configure S3 credential (example): `S3 account Apello`
      - Must allow PutObject/GetObject on bucket.
    - Connect: `Download Video` → `Upload a file`

11. **Extend decision & merge**
    - Add **IF** `Is Extend?`
      - Condition: `{{ $('Get Final URL').first().json.isExtend }}` equals `true`
      - Connect: `Upload a file` → `Is Extend?`
    - True branch: **Telegram** node `Send Merge Options`
      - chatId: `{{ $('Parse Input').first().json.chatId }}`
      - Text: “Extended Video Ready!... Choose how to merge:”
      - Inline keyboard:
        - Auto merge callback: `am:<sessionId>`
        - Manual links callback: `mm:<sessionId>`
      - Telegram credential: create/select your bot credential.
    - False branch: **Code** node `Skip Concat`
      - Output: `{ sessionId, isExtend:false, videoUrl, rawResponse }` sourced from `Get Final URL`
    - Add **Merge** node `Merge After Concat` to converge:
      - Connect `Send Merge Options` → `Merge After Concat` (input 0)
      - Connect `Skip Concat` → `Merge After Concat` (input 1)

12. **Retrieve session from Redis**
    - Node: **Redis** named `Get Session`
    - Operation: **Get**
    - Key: `session:{{ $('Get Final URL').first().json.sessionId }}`
    - Redis credential: configure host/port/password/TLS as needed.
    - Connect: `Merge After Concat` → `Get Session`

13. **Merge session + new video info**
    - Node: **Code** named `Merge Session Data`
    - Parse Redis session JSON and enrich with:
      - `videoUrl`, `rawResponse`, `s3Url` (your public S3 URL pattern), `status:'video_ready'`, `updatedAt`
    - Connect: `Get Session` → `Merge Session Data`

14. **Create and upload metadata file**
    - Node: **Code** named `Create Metadata`
      - Build text content and emit binary property `metadata`
      - Set `metadataKey` to `videos/<sessionId>_metadata.txt`
    - Node: **S3** named `Upload Metadata`
      - Operation: Upload
      - Bucket: `shorts`
      - Key: `{{ $json.metadataKey }}`
      - Binary property name: `metadata`
    - Connect: `Merge Session Data` → `Create Metadata` → `Upload Metadata`

15. **Update session in Redis**
    - Node: **Redis** named `Update Session`
    - Operation: **Set**
    - Key: `session:{{ $json.sessionId }}`
    - Value: `{{ JSON.stringify($json) }}`
    - TTL: 86400; enable expire
    - Connect: `Upload Metadata` → `Update Session`

16. **Download video back from S3 for Telegram**
    - Node: **S3** named `Download for Telegram`
    - Bucket: `shorts`
    - Key: `videos/{{ $('Create Metadata').first().json.sessionId }}.mp4`
    - Connect: `Update Session` → `Download for Telegram`

17. **Send Telegram preview with correct button set**
    - Node: **IF** named `Is Veo Model?`
      - Condition: `{{ $('Create Metadata').first().json.selectedModel?.startsWith('veo') }}` equals `true`
      - Connect: `Download for Telegram` → `Is Veo Model?`
    - True: **Telegram** `Send Video Preview (Veo)`
      - Operation: sendVideo
      - chatId: `{{ $('Create Metadata').first().json.chatId }}`
      - Binary data: true
      - Caption (Markdown) includes title
      - Inline keyboard includes publish buttons + `extend:<sessionId>` + cancel
    - False: **Telegram** `Send Video Preview (Other)`
      - Same but without the extend button

18. **Retry path (from Completed? false)**
    - Add **IF** `Can Retry?` with condition `{{ $json.attempt < 15 }}`
    - True: **HTTP Request** `Retry Poll`
      - POST to your webhook production URL (must match your instance), path `/webhook/video-poll`
      - JSON body containing same fields but `attempt + 1`
    - False: **Telegram** `Send Timeout`
      - chatId: `{{ $json.chatId }}`
      - Markdown message including attempt and taskId
    - Connect: `Completed? (false)` → `Can Retry?` → (true/false branches)

**Credential checklist**
- **KIE.ai:** HTTP Header Auth credential (API key header as required by KIE.ai).
- **Redis:** host/port/password (and TLS if used).
- **S3:** access key/secret + region/endpoint; permissions for upload/download.
- **Telegram Bot:** bot token credential in n8n.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Update the webhook URL in “Retry Poll” to match your n8n instance | The workflow currently calls `https://n8n.mosaabyassir.xyz/webhook/video-poll`; this must be replaced in your environment. |
| Polling strategy | Waits 1 minute between checks, retries up to 15 times (~15 minutes total). |
| Model support | Logic explicitly supports Veo-style `successFlag` responses and “market” APIs for Sora2/Seedance with flexible URL extraction. |
| Delivery UX | Telegram inline buttons return callback_data like `publish:<sessionId>`, `extend:<sessionId>`, etc., implying another workflow handles Telegram callback queries. |