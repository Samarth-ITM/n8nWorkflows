Extract video metadata & auto-upload to YouTube with GPT-4o SEO optimization

https://n8nworkflows.xyz/workflows/extract-video-metadata---auto-upload-to-youtube-with-gpt-4o-seo-optimization-11823


# Extract video metadata & auto-upload to YouTube with GPT-4o SEO optimization

## 1. Workflow Overview

**Purpose:** Receive a video file via HTTP webhook, validate and extract basic file metadata, generate YouTube SEO metadata using GPT‑4o (title/description/tags/category), upload the video to YouTube, optionally add it to a playlist, log the upload to Google Sheets, notify a team via Gmail, and respond to the original webhook request with the result.

**Typical use cases**
- Automated publishing pipeline for a content team
- Central “upload endpoint” for apps/services that produce videos
- SEO/metadata standardization via AI prior to publishing

### 1.1 Video Intake & Validation
Receives the binary video file, initializes an upload session (IDs/timestamps/defaults), ensures binary exists, validates format and size.

### 1.2 Metadata Extraction & AI Context Preparation
Builds a lightweight “content context” for the AI prompt from project inputs + file properties.

### 1.3 AI SEO Metadata Generation (GPT‑4o)
Uses a LangChain Agent connected to an OpenAI Chat Model (gpt‑4o) to produce JSON metadata; then parses/sanitizes it and maps category names to YouTube category IDs.

### 1.4 Publish Settings & YouTube Upload
Decides between scheduled vs immediate publish settings, uploads to YouTube with generated metadata.

### 1.5 Playlist, Logging, Notification, Webhook Response
Optionally adds the uploaded video to a playlist, appends a log row to Google Sheets, emails a confirmation, returns a JSON webhook response, and generates an aggregate summary output.

---

## 2. Block-by-Block Analysis

### Block 1 — Video Intake & Validation
**Overview:** Accepts a POST request containing a binary video, creates an upload session context, verifies the binary payload exists, and validates file type/extension and size before continuing.

**Nodes involved:**
- Video Upload Webhook
- Initialize Upload Session
- Validate Binary Data
- Extract Video Metadata
- Error Response

#### Node: Video Upload Webhook
- **Type / role:** `Webhook` (entry point). Receives inbound HTTP POST requests.
- **Key configuration:**
  - **Path:** `video-upload`
  - **Method:** `POST`
  - **Response mode:** `responseNode` (the workflow must end with a Respond to Webhook node to return a response).
- **Inputs / outputs:**
  - **Output:** to **Initialize Upload Session**.
- **Edge cases / failures:**
  - Missing binary field if caller doesn’t send multipart/form-data correctly.
  - Large uploads may hit n8n reverse proxy / instance limits before workflow runs.
  - If no Respond node executes, request may hang until timeout.

#### Node: Initialize Upload Session
- **Type / role:** `Set` (create normalized fields used downstream).
- **Configuration choices (interpreted):**
  - Creates:
    - `uploadId`: `VID-{{yyyyMMddHHmmss}}` using `$now.format(...)`
    - `timestamp`: `$now.toISO()`
    - `projectName`: from `$json.body?.projectName` else `"Untitled Video"`
    - `targetAudience`: from body else `"general"`
    - `language`: from body else `"en"`
    - `playlistId`: from body else `""`
    - `publishTime`: from body else `""`
- **Key expressions / variables:**
  - `$json.body?.<field>` optional chaining: depends on webhook payload structure.
  - `$now` formatting.
- **Inputs / outputs:**
  - Input from **Video Upload Webhook**.
  - Output to **Validate Binary Data**.
- **Edge cases / failures:**
  - If the webhook payload doesn’t have a `body` object (depending on content-type), expressions may resolve to defaults (usually OK).
  - `publishTime` is not validated for ISO format here; later logic assumes a non-empty string is meaningful.

#### Node: Validate Binary Data
- **Type / role:** `IF` (guards workflow execution).
- **Configuration choices:**
  - Condition: `$binary.data !== undefined` must be true.
  - Strict type validation enabled.
- **Routing:**
  - **True branch:** to **Extract Video Metadata**
  - **False branch:** to **Error Response**
- **Edge cases / failures:**
  - If binary property name is not `data` (e.g., `video`), this will fail. Caller and webhook node must align on binary field name.
  - If the webhook is configured to not accept binary/multipart, `$binary` can be empty.

#### Node: Extract Video Metadata
- **Type / role:** `Code` (validation + metadata extraction from binary header info supplied by n8n).
- **Main logic:**
  - Reads `$input.first().binary.data`.
  - Builds `videoMetadata`:
    - `fileName`, `mimeType`, `fileSize`, `fileExtension`
  - Validates:
    - MIME in allowed list: `video/mp4`, `video/quicktime`, `video/x-msvideo`, `video/webm`, `video/x-matroska`
    - OR extension in allowed list: `mp4`, `mov`, `avi`, `webm`, `mkv`
  - Validates YouTube max size: `128GB`
  - Merges in session fields from **Initialize Upload Session** via:
    - `const inputData = $('Initialize Upload Session').first().json;`
  - Returns JSON + preserves original `binary`.
- **Inputs / outputs:**
  - Input from **Validate Binary Data** (true).
  - Output to **Prepare AI Context**.
- **Edge cases / failures:**
  - Throws explicit errors for missing binary, invalid format, or size too large.
  - Binary metadata like `fileSize` may be missing (then defaults to 0), which can weaken validation.
  - Uses `$('Initialize Upload Session')...` which assumes that node executed and has data (true in the current graph).
- **Version-specific notes:**
  - Code node behavior depends on n8n runtime; ensure “Code” node is allowed (some hardened environments restrict it).

#### Node: Error Response
- **Type / role:** `Respond to Webhook` (error path response).
- **Configuration choices:**
  - Responds with JSON string:
    - `{ error: 'No video file provided', uploadId: <from Initialize Upload Session> }`
- **Inputs / outputs:**
  - Input from **Validate Binary Data** (false).
  - Terminates the webhook response.
- **Edge cases / failures:**
  - If **Initialize Upload Session** did not run (not possible in current flow), expression would fail.
  - Returns JSON as a **string** (because it uses `JSON.stringify(...)`). Many clients accept this, but it means response body is a string rather than an object unless n8n sets correct headers and clients parse it.

---

### Block 2 — Metadata Extraction & AI Context Preparation
**Overview:** Builds a prompt-ready context object combining user-provided fields and coarse video properties to guide the AI metadata generation.

**Nodes involved:**
- Prepare AI Context

#### Node: Prepare AI Context
- **Type / role:** `Code` (construct prompt context).
- **Main logic:**
  - Reads `videoMetadata` and session fields.
  - Creates `contentContext` with:
    - `projectName`, `fileName`, `fileType`, `targetAudience`, `language`
    - `estimatedDuration`: a rough estimate derived from file size:
      - `Math.round(fileSize / (1024*1024*5)) + ' minutes (estimated)'`
      - This assumes ~5 MB per minute (very approximate).
  - Outputs merged JSON and keeps binary.
- **Inputs / outputs:**
  - Input from **Extract Video Metadata**.
  - Output to **AI Metadata Generator**.
- **Edge cases / failures:**
  - If `fileSize` is 0 or missing, duration becomes `"unknown"` (by code branch).
  - Duration estimate can be very wrong for high/low bitrate videos; purely informational for AI.

---

### Block 3 — AI SEO Metadata Generation (GPT‑4o)
**Overview:** Calls GPT‑4o through a LangChain Agent to generate YouTube SEO metadata, then parses and normalizes the result (including category ID mapping).

**Nodes involved:**
- AI Metadata Generator
- OpenAI Chat Model
- Parse AI Metadata

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` (language model provider for the agent).
- **Key configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.7` (creative but not extreme)
- **Connections:**
  - Connected to **AI Metadata Generator** via `ai_languageModel`.
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials.
  - Model availability / org restrictions.
  - Rate limiting / token limits if descriptions become too long.

#### Node: AI Metadata Generator
- **Type / role:** `LangChain Agent` (prompt orchestration and execution).
- **Configuration choices (interpreted):**
  - Prompt requests JSON output with keys: `title`, `description`, `tags` (array), `category`, `categoryId`.
  - Includes constraints:
    - Title max 100 chars
    - Description 300–500 words with hook, chapters, CTA, keywords
    - 10–15 tags
    - Category must be chosen from a fixed list
  - System message: “You are a YouTube SEO expert… Always output valid JSON.”
- **Inputs / outputs:**
  - Input from **Prepare AI Context**.
  - Output to **Parse AI Metadata**. (Agent output is expected in `inputData.output` downstream.)
- **Edge cases / failures:**
  - The agent may return text with markdown fences or additional commentary despite instructions.
  - Output may be invalid JSON; downstream node tries to recover.
  - If the agent uses tools/other capabilities by default (depending on n8n node version), ensure it’s configured to only respond with text.

#### Node: Parse AI Metadata
- **Type / role:** `Code` (robust parsing + normalization).
- **Main logic:**
  - Reads `inputData.output` (agent output) or uses `'{}'`.
  - Extracts the first `{ ... }` block via regex and attempts `JSON.parse`.
  - On parse failure, uses fallback metadata:
    - title from `projectName`
    - generic description/tags
    - default category Science & Technology (ID `28`)
  - Maps category names to IDs using `categoryMap`.
  - Enforces:
    - Title truncated to 100 chars
    - Tags limited to 15
  - Restores original context data from **Prepare AI Context** and binary from that node.
- **Inputs / outputs:**
  - Input from **AI Metadata Generator**.
  - Output to **Check Scheduled Publish**.
- **Edge cases / failures:**
  - Regex `{[\s\S]*}` is greedy and may capture too much if the model outputs multiple JSON objects or braces in text.
  - If the agent returns valid JSON but uses different key names/types (e.g., tags as string), tags become empty.
  - Relies on node name references: `$('Prepare AI Context')...` must exist and have an item.

---

### Block 4 — Publish Settings & YouTube Upload
**Overview:** Determines whether to schedule publication, sets privacy accordingly, then uploads the video to YouTube with AI-generated metadata.

**Nodes involved:**
- Check Scheduled Publish
- Set Scheduled Settings
- Set Immediate Publish
- Merge Publish Settings
- YouTube Upload Video

#### Node: Check Scheduled Publish
- **Type / role:** `IF` (branch on scheduling).
- **Condition:**
  - `publishTime` is not empty.
  - Loose type validation.
- **Routing:**
  - **True:** to **Set Scheduled Settings**
  - **False:** to **Set Immediate Publish**
- **Edge cases / failures:**
  - Any non-empty string triggers scheduled mode even if not a valid datetime.
  - No timezone normalization is performed.

#### Node: Set Scheduled Settings
- **Type / role:** `Set` (apply scheduled publish fields).
- **Configuration:**
  - `privacyStatus = "private"`
  - `publishAt = {{$json.publishTime}}`
  - `includeOtherFields = true` (keeps metadata and binary references in the item JSON, but note the binary is separate).
- **Output:** to **Merge Publish Settings** (branch 0).

#### Node: Set Immediate Publish
- **Type / role:** `Set` (apply immediate publish fields).
- **Configuration:**
  - `privacyStatus = "public"`
  - `publishAt = ""`
  - `includeOtherFields = true`
- **Output:** to **Merge Publish Settings** (branch 1).

#### Node: Merge Publish Settings
- **Type / role:** `Merge` (choose one branch result).
- **Mode:** `chooseBranch` (passes through whichever branch produced data).
- **Output:** to **YouTube Upload Video**.
- **Edge cases / failures:**
  - If both branches could theoretically execute (not here), chooseBranch behavior matters. With IF, only one branch will emit items.

#### Node: YouTube Upload Video
- **Type / role:** `YouTube` node (video upload).
- **Operation:** `video.upload`
- **Key configuration (interpreted):**
  - Title: `{{$json.youtubeMetadata.title}}`
  - Category ID: `{{$json.youtubeMetadata.categoryId}}`
  - Options:
    - Tags: `{{$json.youtubeMetadata.tags.join(',')}}` (comma-separated string)
    - Description: `{{$json.youtubeMetadata.description}}`
    - Privacy: `{{$json.privacyStatus}}`
    - Default language: `{{$json.language}}`
    - Notify subscribers: `true`
- **Inputs / outputs:**
  - Input from **Merge Publish Settings**.
  - Output to **Process Upload Result**.
- **Credentials required:**
  - YouTube OAuth2 with permission to upload videos.
- **Edge cases / failures:**
  - Upload requires binary video data to be in the expected binary property for the YouTube node. This workflow preserves binary through earlier nodes, but the YouTube node may require a specific binary field name/config depending on n8n version.
  - YouTube API quota limits and resumable upload failures.
  - `privacyStatus=private` with `publishAt` set may not schedule unless the node/API supports scheduling via `publishedAt` / `publishAt` semantics; this workflow sets `publishAt` in JSON but the YouTube node configuration shown does **not** pass `publishAt` to the node’s options (potential functional gap).

---

### Block 5 — Playlist, Logging, Notification, Webhook Response
**Overview:** Processes upload response into a canonical record, optionally adds the video to a playlist, logs to Google Sheets, emails a confirmation, responds to the webhook, and also emits an aggregated summary.

**Nodes involved:**
- Process Upload Result
- Check Playlist Add
- Add to Playlist
- Merge Playlist Result
- Log Upload to Sheets
- Send Upload Notification
- Success Response
- Generate Upload Summary

#### Node: Process Upload Result
- **Type / role:** `Code` (normalize the YouTube response + enrich with context).
- **Main logic:**
  - Reads upload result JSON.
  - Reads prior context from **Merge Publish Settings**.
  - Computes:
    - `videoId` from `uploadResult.id || uploadResult.videoId || 'unknown'`
    - `videoUrl = https://youtu.be/<videoId>`
  - Outputs a flattened record including metadata, privacy, playlistId, status, timestamps.
- **Output:** to **Check Playlist Add**.
- **Edge cases / failures:**
  - If YouTube node returns a different structure, `videoId` may become `unknown`.
  - No explicit error handling for missing fields beyond fallback.

#### Node: Check Playlist Add
- **Type / role:** `IF` (optional playlist insertion).
- **Condition:** `playlistId` not empty.
- **Routing:**
  - **True:** to **Add to Playlist**
  - **False:** to **Merge Playlist Result** (branch 1)
- **Edge cases / failures:**
  - Invalid playlist ID or insufficient permissions will fail on the true branch.

#### Node: Add to Playlist
- **Type / role:** `YouTube` node (playlist item insertion).
- **Resource:** `playlistItem`
- **Parameters:**
  - `videoId = {{$json.videoId}}`
  - `playlistId = {{$json.playlistId}}`
- **Output:** to **Merge Playlist Result** (branch 0)
- **Edge cases / failures:**
  - Playlist privacy restrictions.
  - OAuth scope missing for playlist modifications.

#### Node: Merge Playlist Result
- **Type / role:** `Merge` (chooseBranch).
- **Purpose:** Continue with either “added to playlist” result or “skipped playlist” path.
- **Outputs:**
  - To **Log Upload to Sheets**
  - To **Generate Upload Summary**
- **Edge cases / failures:**
  - If playlist addition fails hard, the workflow may stop before logging/notification unless error handling is added.

#### Node: Log Upload to Sheets
- **Type / role:** `Google Sheets` (append log row).
- **Operation:** `append`
- **Sheet:** `VideoUploads`
- **Document:** documentId is configured as a selectable list but currently **empty** in JSON (must be set).
- **Columns mapped:**
  - Title, Status, Category, Video ID, Upload ID, Video URL, Playlist ID, Uploaded At, Privacy Status
- **Credentials required:** Google Sheets OAuth2 (Google account with access to the spreadsheet).
- **Edge cases / failures:**
  - Missing `documentId` will cause configuration/runtime error.
  - Sheet/tab name mismatch (`VideoUploads` must exist).
  - Concurrent appends may hit API rate limits.

#### Node: Send Upload Notification
- **Type / role:** `Gmail` (send email).
- **To:** `content-team@company.com`
- **Subject:** `Video Uploaded: {{ $json.title }}`
- **Body:** includes uploadId, URL, category, privacy, publishAt, tags, description, uploadedAt.
- **Credentials required:** Gmail OAuth2.
- **Edge cases / failures:**
  - Gmail sending limits, invalid “from” account permissions, or restricted recipients.
  - If `tags` is not an array (should be), `join` could fail—here tags comes from Process Upload Result, which sets it from earlier parsed array.

#### Node: Success Response
- **Type / role:** `Respond to Webhook` (final API response on success path).
- **Response body:** `JSON.stringify({ success: true, uploadId, videoId, videoUrl, title, category, uploadStatus })`
- **Edge cases / failures:**
  - Like Error Response, returns JSON as a string; some API consumers expect an object.
  - Must always be reached for webhook call to complete; if an upstream node errors (Sheets/Gmail), webhook response may never be sent unless “Continue On Fail” is configured (not shown).

#### Node: Generate Upload Summary
- **Type / role:** `Code` (aggregates multiple items into a summary object).
- **Logic:**
  - Reads all input items and returns:
    - `totalUploads`
    - `successCount` where `uploadStatus === 'completed'`
    - `videos`: array of `{videoId,title,url}`
    - `generatedAt`
- **Inputs / outputs:**
  - Input from **Merge Playlist Result**.
  - No downstream connection (produces data for workflow run results, not for webhook response).
- **Edge cases / failures:**
  - If only one item flows, summary still works.
  - Not used in webhook response; informational unless another node consumes it.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## Video Metadata Extraction & YouTube Auto-Upload with AI; Overview, Key Features, Required Credentials |
| Sticky Note1 | Sticky Note | Documentation (Step 1) | — | — | ### Step 1: Video Intake & Validation (receives via webhook, validates, extracts initial metadata, stores binary) |
| Sticky Note2 | Sticky Note | Documentation (Step 2) | — | — | ### Step 2: Metadata Extraction & Analysis |
| Sticky Note3 | Sticky Note | Documentation (Step 3) | — | — | ### Step 3: AI Content Generation |
| Sticky Note4 | Sticky Note | Documentation (Step 4) | — | — | ### Step 4: YouTube Upload & Post-Processing |
| Video Upload Webhook | Webhook | Entry point receiving video binary | — | Initialize Upload Session | Step 1 note content applies |
| Initialize Upload Session | Set | Create upload/session variables and defaults | Video Upload Webhook | Validate Binary Data | Step 1 note content applies |
| Validate Binary Data | IF | Ensure binary video exists | Initialize Upload Session | Extract Video Metadata; Error Response | Step 1 note content applies |
| Extract Video Metadata | Code | Validate type/size; extract filename/mime/size/ext | Validate Binary Data | Prepare AI Context | Step 2 note content applies |
| Error Response | Respond to Webhook | Return error when no binary provided | Validate Binary Data (false) | — | Step 1 note content applies |
| Prepare AI Context | Code | Build prompt context for AI | Extract Video Metadata | AI Metadata Generator | Step 2 note content applies |
| AI Metadata Generator | LangChain Agent | Generate SEO metadata JSON | Prepare AI Context; OpenAI Chat Model (AI input) | Parse AI Metadata | Step 3 note content applies |
| OpenAI Chat Model | OpenAI Chat Model | LLM provider (gpt‑4o) | — | AI Metadata Generator | Step 3 note content applies |
| Parse AI Metadata | Code | Parse/validate AI JSON; map category IDs | AI Metadata Generator | Check Scheduled Publish | Step 3 note content applies |
| Check Scheduled Publish | IF | Decide scheduled vs immediate publish | Parse AI Metadata | Set Scheduled Settings; Set Immediate Publish | Step 4 note content applies |
| Set Scheduled Settings | Set | Apply privacy private + publishAt | Check Scheduled Publish (true) | Merge Publish Settings | Step 4 note content applies |
| Set Immediate Publish | Set | Apply privacy public | Check Scheduled Publish (false) | Merge Publish Settings | Step 4 note content applies |
| Merge Publish Settings | Merge | Pass through chosen publish configuration | Set Scheduled Settings; Set Immediate Publish | YouTube Upload Video | Step 4 note content applies |
| YouTube Upload Video | YouTube | Upload video with title/description/tags/category | Merge Publish Settings | Process Upload Result | Step 4 note content applies |
| Process Upload Result | Code | Normalize upload result and create canonical record | YouTube Upload Video | Check Playlist Add | Step 4 note content applies |
| Check Playlist Add | IF | Decide whether to add to playlist | Process Upload Result | Add to Playlist; Merge Playlist Result | Step 4 note content applies |
| Add to Playlist | YouTube | Insert video into playlist | Check Playlist Add (true) | Merge Playlist Result | Step 4 note content applies |
| Merge Playlist Result | Merge | Continue regardless of playlist branch | Add to Playlist; Check Playlist Add (false) | Log Upload to Sheets; Generate Upload Summary | Step 4 note content applies |
| Log Upload to Sheets | Google Sheets | Append upload record to a spreadsheet | Merge Playlist Result | Send Upload Notification | Step 4 note content applies |
| Send Upload Notification | Gmail | Email confirmation to content team | Log Upload to Sheets | Success Response | Step 4 note content applies |
| Success Response | Respond to Webhook | Return success JSON to caller | Send Upload Notification | — | Step 4 note content applies |
| Generate Upload Summary | Code | Create aggregate summary of uploaded items | Merge Playlist Result | — | Step 4 note content applies |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add “Webhook” node** named **Video Upload Webhook**  
   - Method: **POST**  
   - Path: **video-upload**  
   - Response mode: **Using “Respond to Webhook” node** (`responseNode`)  
   - Ensure the webhook is able to accept **binary file uploads** (typically multipart/form-data with a file field that n8n stores as `binary.data`).

3. **Add “Set” node** named **Initialize Upload Session**, connect from the Webhook  
   - Add fields:
     - `uploadId` (String): `={{ 'VID-' + $now.format('yyyyMMddHHmmss') }}`
     - `timestamp` (String): `={{ $now.toISO() }}`
     - `projectName`: `={{ $json.body?.projectName || 'Untitled Video' }}`
     - `targetAudience`: `={{ $json.body?.targetAudience || 'general' }}`
     - `language`: `={{ $json.body?.language || 'en' }}`
     - `playlistId`: `={{ $json.body?.playlistId || '' }}`
     - `publishTime`: `={{ $json.body?.publishTime || '' }}`

4. **Add “IF” node** named **Validate Binary Data**, connect from Initialize Upload Session  
   - Condition: Boolean → `={{ $binary.data !== undefined }}` is **true**  
   - True output → next step; False output → error response.

5. **Add “Respond to Webhook” node** named **Error Response**, connect from Validate Binary Data (false)  
   - Respond with: **JSON**  
   - Body expression:  
     - `={{ JSON.stringify({ error: 'No video file provided', uploadId: $('Initialize Upload Session').first().json.uploadId }) }}`

6. **Add “Code” node** named **Extract Video Metadata**, connect from Validate Binary Data (true)  
   - Implement logic to:
     - Read `$binary.data` fields (`fileName`, `mimeType`, `fileSize`)
     - Validate allowed MIME/extensions (mp4/mov/avi/webm/mkv)
     - Validate file size ≤ 128GB
     - Merge in fields from Initialize Upload Session
     - Return `{ ...inputData, videoMetadata: {...}, validationStatus:'passed' }` and preserve binary.

7. **Add “Code” node** named **Prepare AI Context**, connect from Extract Video Metadata  
   - Build `contentContext` with projectName, fileName, fileType, targetAudience, language, and an estimated duration derived from file size.
   - Return merged JSON and preserve binary.

8. **Add “OpenAI Chat Model” node** named **OpenAI Chat Model**  
   - Model: **gpt-4o**  
   - Temperature: **0.7**  
   - Configure **OpenAI credentials** (API key) in n8n.

9. **Add “AI Agent (LangChain)” node** named **AI Metadata Generator**  
   - Prompt type: **Define**  
   - Paste a prompt that requests JSON with: `title`, `description`, `tags` (array), `category`, `categoryId`, using the variables from `contentContext`.  
   - Set System message: “You are a YouTube SEO expert… Always output valid JSON.”  
   - Connect:
     - Main input from **Prepare AI Context**
     - AI language model input from **OpenAI Chat Model**

10. **Add “Code” node** named **Parse AI Metadata**, connect from AI Metadata Generator  
   - Parse agent output (typically in a field like `output`) and extract the JSON object.
   - Add fallbacks if parsing fails.
   - Map category names to YouTube category IDs (1,2,10,15,17,19,20,22,23,24,25,26,27,28,29).
   - Enforce title ≤ 100 chars and tags ≤ 15.
   - Preserve original binary for upload.

11. **Add “IF” node** named **Check Scheduled Publish**, connect from Parse AI Metadata  
   - Condition: String → `={{ $json.publishTime }}` **not empty**.

12. **Add “Set” node** named **Set Scheduled Settings**, connect from Check Scheduled Publish (true)  
   - `privacyStatus = "private"`  
   - `publishAt = {{$json.publishTime}}`  
   - Include other fields: **true**

13. **Add “Set” node** named **Set Immediate Publish**, connect from Check Scheduled Publish (false)  
   - `privacyStatus = "public"`  
   - `publishAt = ""`  
   - Include other fields: **true**

14. **Add “Merge” node** named **Merge Publish Settings**  
   - Mode: **Choose Branch**  
   - Connect both Set nodes into this Merge.
   - Output goes to YouTube upload.

15. **Add “YouTube” node** named **YouTube Upload Video**, connect from Merge Publish Settings  
   - Resource: **Video**
   - Operation: **Upload**
   - Title: `={{ $json.youtubeMetadata.title }}`
   - Category ID: `={{ $json.youtubeMetadata.categoryId }}`
   - Options:
     - Tags: `={{ $json.youtubeMetadata.tags.join(',') }}`
     - Description: `={{ $json.youtubeMetadata.description }}`
     - Privacy Status: `={{ $json.privacyStatus }}`
     - Default language: `={{ $json.language }}`
     - Notify subscribers: **true**
   - Configure **YouTube OAuth2 credentials** with upload permissions.
   - Ensure the node is configured to use the incoming **binary video** (field name must match your instance expectations; this workflow assumes `binary.data`).

16. **Add “Code” node** named **Process Upload Result**, connect from YouTube Upload Video  
   - Create a normalized output object containing: uploadId, videoId, videoUrl, title/description/tags/category/categoryId, privacyStatus, publishAt, playlistId, uploadStatus, uploadedAt.

17. **Add “IF” node** named **Check Playlist Add**, connect from Process Upload Result  
   - Condition: `playlistId` not empty.

18. **Add “YouTube” node** named **Add to Playlist**, connect from Check Playlist Add (true)  
   - Resource: **playlistItem**
   - Set `videoId` and `playlistId` from JSON.
   - Reuse the same YouTube OAuth2 credentials.

19. **Add “Merge” node** named **Merge Playlist Result**  
   - Mode: **Choose Branch**
   - Connect:
     - True branch output (Add to Playlist) to one Merge input
     - False branch output (from Check Playlist Add false) to the other Merge input

20. **Add “Google Sheets” node** named **Log Upload to Sheets**, connect from Merge Playlist Result  
   - Operation: **Append**
   - Choose or paste the **Document ID** (required).
   - Sheet name/tab: **VideoUploads** (must exist).
   - Map columns: Title, Status, Category, Video ID, Upload ID, Video URL, Playlist ID, Uploaded At, Privacy Status.
   - Configure **Google Sheets OAuth2 credentials**.

21. **Add “Gmail” node** named **Send Upload Notification**, connect from Log Upload to Sheets  
   - To: `content-team@company.com`
   - Subject/body using expressions with `uploadId`, `title`, `videoUrl`, etc.
   - Configure **Gmail OAuth2 credentials**.

22. **Add “Respond to Webhook” node** named **Success Response**, connect from Send Upload Notification  
   - Respond with JSON body (as in workflow) containing: success, uploadId, videoId, videoUrl, title, category, uploadStatus.

23. **(Optional) Add “Code” node** named **Generate Upload Summary**, connect from Merge Playlist Result  
   - Aggregate all items and return summary object.
   - This node is not required for webhook response; it’s for run-level output/monitoring.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Required credentials: YouTube OAuth2, OpenAI API, Gmail, Google Sheets | From the workflow’s main sticky note (top-left) |
| The workflow returns webhook responses via dedicated Respond nodes (success/error). If downstream nodes (Sheets/Gmail) fail, the webhook may never receive a success response unless error handling is added. | Operational consideration for production deployments |
| Scheduling gap: `publishAt` is computed but not clearly passed into the YouTube upload node configuration shown; verify if your YouTube node version supports scheduled publishing and which field it expects (often `publishedAt`). | Potential integration mismatch to validate in your n8n version |