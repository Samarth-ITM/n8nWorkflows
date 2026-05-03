Generate Sora v2 ASMR clips with GPT-5.1, stitch via Cloudinary, and post to Twitter/X

https://n8nworkflows.xyz/workflows/generate-sora-v2-asmr-clips-with-gpt-5-1--stitch-via-cloudinary--and-post-to-twitter-x-12590


# Generate Sora v2 ASMR clips with GPT-5.1, stitch via Cloudinary, and post to Twitter/X

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate Sora v2 ASMR clips with GPT-5.1, stitch via Cloudinary, and post to Twitter/X  
**Workflow name (in JSON):** Generate Sora videos, stitch clips, and post to Twitter  
**Status:** Inactive (`active: false`)

**Purpose:**  
On a schedule, the workflow generates 3 “oddly satisfying” ASMR cutting scene prompts and a Twitter/X caption using GPT-5.1, stores the prompts in Google Sheets, generates 3 short Sora v2 video clips in parallel, polls OpenAI until each clip renders, downloads completed videos, uploads each clip to Cloudinary, stitches the uploaded clips into one combined video using Cloudinary transformations, downloads the stitched result, uploads it to Twitter/X using chunked media upload (INIT → APPEND → FINALIZE), waits for Twitter processing to finish, then posts a tweet with the generated caption and updates Google Sheets status to “Posted”.

### 1.1 Trigger / Orchestration
- Runs on a schedule and starts the pipeline.

### 1.2 Idea Generation (LLM Agent)
- GPT-5.1 agent produces a structured JSON containing:
  - Category
  - Twitter caption text
  - 3 clip prompts (each with sound + duration)

### 1.3 Tracking (Google Sheets)
- Appends a row with category, scenes, and status “In Progress”.

### 1.4 Video Generation (Sora v2 in parallel)
- Creates three video jobs on OpenAI `videos` API (model `sora-2`) in parallel.

### 1.5 Monitoring / Downloading / Upload to Cloudinary (per clip loop)
- Loops through the three clips:
  - Polls status every 30s
  - Downloads completed video
  - Uploads binary to Cloudinary
  - Collects Cloudinary `public_id` for stitching

### 1.6 Stitching (Cloudinary splice URL) + Download
- Builds a Cloudinary transformation URL to splice the clips.
- Downloads the stitched MP4 as a binary file.

### 1.7 Twitter/X Upload + Post
- Prepares byte size and media type.
- Uploads video via Twitter chunked upload (single segment in this implementation).
- Finalizes, polls processing state until succeeded.
- Posts tweet with caption and attached media id.
- Appends “Posted” status to Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduled start
**Overview:** Triggers workflow executions on a configured interval.  
**Nodes involved:** `Schedule Trigger`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration choices:** Uses `rule.interval` with a default/empty interval object (effectively “run on interval” but requires proper configuration in UI).
- **Inputs/Outputs:** No inputs; outputs to `ASMR Cutting Ideas`.
- **Edge cases / failures:**
  - Misconfigured interval may cause unexpected frequency (or not run).
  - Timezone differences depending on n8n instance settings.
- **Version notes:** typeVersion `1.2`.

---

### Block 2.2 — Generate clip ideas (Agent + model + structured parsing + Think tool)
**Overview:** Creates 3 ASMR cutting prompts, sounds, durations, and a Twitter caption in a strict JSON structure.  
**Nodes involved:** `OpenAI Chat Model`, `AI Reasoning Tool`, `Format JSON Output`, `ASMR Cutting Ideas`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides GPT model to the agent.
- **Configuration choices:**
  - Model: `gpt-5.1`
  - No special options set.
- **Connections:**
  - Output (ai_languageModel) → `ASMR Cutting Ideas`.
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials.
  - Model access not enabled for the API key/account.
  - Rate limits.
- **Version notes:** typeVersion `1.2`.

#### Node: AI Reasoning Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolThink` — “Think tool” available to the agent for internal reasoning.
- **Configuration choices:** Default.
- **Connections:**
  - Output (ai_tool) → `ASMR Cutting Ideas`.
- **Edge cases / failures:** Usually none; if removed, the agent system prompt references it (“Review and adjust the output using the Think tool.”).
- **Version notes:** typeVersion `1`.

#### Node: Format JSON Output
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON schema.
- **Configuration choices:**
  - Provides an example JSON schema including `Category`, `Twitter_Text`, and `Clip_1..Clip_3` objects with `Prompt`, `Sound`, `Duration`.
- **Connections:**
  - Output (ai_outputParser) → `ASMR Cutting Ideas`.
- **Edge cases / failures:**
  - If the LLM output deviates from expected structure, parsing fails or yields empty/partial fields.
  - Prompt text may include forbidden quotes; the agent instructions request single quotes for emphasis, but outputs still may include double quotes.
- **Version notes:** typeVersion `1.3`.

#### Node: ASMR Cutting Ideas
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — main LLM agent that generates the final structured output.
- **Configuration choices:**
  - Prompt type: `define`
  - System message contains detailed constraints (static frontal shot, knife cutting toward camera, safe language, durations 8 or 12 only, etc.).
  - Has output parser enabled (connected to `Format JSON Output`).
  - Uses the Think tool (connected via `AI Reasoning Tool`).
- **Key variables produced (used later):**
  - `$('ASMR Cutting Ideas').first().json.output.Clip_1.Prompt` etc.
  - `$('ASMR Cutting Ideas').first().json.output.Clip_#.Sound`
  - `$('ASMR Cutting Ideas').first().json.output.Clip_#.Duration`
  - `$('ASMR Cutting Ideas').first().json.output.Twitter_Text`
- **Connections:**
  - Input: `Schedule Trigger`
  - Dependencies: `OpenAI Chat Model`, `AI Reasoning Tool`, `Format JSON Output`
  - Output → `Save Category and Clip Scripts`
- **Edge cases / failures:**
  - Output parser mismatch (missing fields) breaks downstream expressions that assume fields exist.
  - Long prompts could exceed what downstream APIs accept (Sora prompt length limits may apply).
- **Version notes:** typeVersion `2.1`.

**Sticky note context:** “## Generate Clip Idea” applies to this block.

---

### Block 2.3 — Save prompts / status tracking in Google Sheets
**Overview:** Appends a tracking row to a Google Sheet with status “In Progress” and the three scene prompts.  
**Nodes involved:** `Save Category and Clip Scripts`

#### Node: Save Category and Clip Scripts
- **Type / role:** `n8n-nodes-base.googleSheets` — append row to a spreadsheet.
- **Configuration choices:**
  - Operation: `append`
  - Document: `1qUir1c-_ScMnoYVoQ0W41nsv5IpLW6rjK8HUNqvNnAg` (example)
  - Sheet gid (as list value): `1420239823`
  - Columns mapped:
    - `Status` = `In Progress`
    - `Scene 1` = `{{$json.output.Clip_1.Prompt}}`
    - `Scene 2` = `{{$json.output.Clip_2.Prompt}}`
    - `Scene 3` = `{{$json.output.Clip_3.Prompt}}`
    - `Category` = `{{$json.output.Category}}`
- **Connections:**
  - Input: `ASMR Cutting Ideas`
  - Output (fan-out) → `Create Sora Video Scene - 1`, `Create Sora Video Scene - 2`, `Create Sora Video Scene - 3`
- **Edge cases / failures:**
  - OAuth token expiration / missing permissions.
  - Wrong sheet schema (missing column headers).
  - Appending does not give a stable row key; later “Update Status” is also an append, not an update (see Block 2.8).
- **Version notes:** typeVersion `4.6`.

---

### Block 2.4 — Generate 3 Sora v2 clips in parallel (OpenAI Videos API)
**Overview:** Creates three separate Sora video generation jobs using OpenAI’s `/v1/videos` endpoint.  
**Nodes involved:** `Create Sora Video Scene - 1`, `Create Sora Video Scene - 2`, `Create Sora Video Scene - 3`, `Combine All Scenes`

#### Node: Create Sora Video Scene - 1
- **Type / role:** `n8n-nodes-base.httpRequest` — POST to OpenAI videos API to create a job.
- **Configuration choices:**
  - POST `https://api.openai.com/v1/videos`
  - JSON body includes:
    - `model: "sora-2"`
    - `prompt:` concatenates:
      - Scene 1 prompt from the current item (`$json['Scene 1']`)
      - Sound from `$('ASMR Cutting Ideas').first().json.output.Clip_1.Sound`
      - Sanitization: replaces newlines and escapes double quotes.
    - `size: "720x1280"` (vertical)
    - `seconds:` from `Clip_1.Duration`
  - Headers include Authorization bearer token placeholder (`YOUR_TOKEN_HERE KEY`) and Content-Type.
- **Connections:** Input from `Save Category and Clip Scripts`; output → `Combine All Scenes` (input 0).
- **Edge cases / failures:**
  - Token placeholder must be replaced; otherwise 401.
  - Bad prompt formatting if expressions resolve to `null`.
  - API may return errors for unsupported duration/size.
- **Version notes:** typeVersion `4.2`.

#### Node: Create Sora Video Scene - 2
- Same structure as Scene 1 but uses `Scene 2`, `Clip_2.Sound`, `Clip_2.Duration`.
- Output → `Combine All Scenes` (input 1).
- Version: `4.2`.

#### Node: Create Sora Video Scene - 3
- Same structure as Scene 1 but uses `Scene 3`, `Clip_3.Sound`, `Clip_3.Duration`.
- Output → `Combine All Scenes` (input 2).
- Version: `4.2`.

#### Node: Combine All Scenes
- **Type / role:** `n8n-nodes-base.merge` — waits for all 3 parallel requests and merges into one stream.
- **Configuration choices:** `numberInputs: 3`.
- **Connections:**
  - Inputs: the three “Create Sora Video Scene” nodes
  - Output → `Loop Through Videos`
- **Edge cases / failures:**
  - If one branch errors hard, merge may not receive all inputs. (No per-node error handling is configured here.)
- **Version notes:** typeVersion `3.2`.

**Sticky note context:** “## Generate Clips” applies to this block.

---

### Block 2.5 — Monitor Sora jobs, download finished clips, upload to Cloudinary (iterative loop)
**Overview:** Iterates over generated video jobs, polling status until completed/failed; downloads completed videos and uploads them to Cloudinary, collecting `public_id` values for stitching.  
**Nodes involved:** `Loop Through Videos`, `Wait 30s for Rendering`, `Check Video Status`, `Check If Still Processing`, `Check If Render Complete`, `Download Completed Video`, `Upload to Cloudinary`, `Collect Video Id`

#### Node: Loop Through Videos
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterative loop controller.
- **Configuration choices:** Default (batch size not explicitly set; n8n default is typically 1 item per batch in UI unless configured).
- **Connections:**
  - Input: `Combine All Scenes`
  - Output 1 → `Collect Video Id` (when loop continues / after uploads push items back)
  - Output 2 → `Wait 30s for Rendering` (process current batch/job)
- **Edge cases / failures:**
  - If batch size > 1, status polling logic may not behave as intended.
- **Version notes:** typeVersion `3`.

#### Node: Wait 30s for Rendering
- **Type / role:** `n8n-nodes-base.wait` — delay between polling attempts.
- **Configuration choices:** `amount: 30` seconds.
- **Connections:** Input from `Loop Through Videos` and from `Check If Render Complete` (not complete); output → `Check Video Status`.
- **Edge cases / failures:** Long rendering times cause many executions; consider max retries.
- **Version notes:** typeVersion `1.1`.

#### Node: Check Video Status
- **Type / role:** `n8n-nodes-base.httpRequest` — GET OpenAI video job status.
- **Configuration choices:**
  - URL: `https://api.openai.com/v1/videos/{{ $json.id }}`
  - Authorization bearer token placeholder.
- **Connections:** Output → `Check If Still Processing`.
- **Edge cases / failures:** 401 if token not set; 404 if job expired; transient network failures.
- **Version notes:** `4.2`.

#### Node: Check If Still Processing
- **Type / role:** `n8n-nodes-base.if` — gate out failed jobs.
- **Configuration choices:** Condition checks `status != "failed"`.
  - True branch → `Check If Render Complete`
  - False branch → `Loop Through Videos` (skip this item and continue loop)
- **Edge cases / failures:** If API returns unexpected status values, job might loop forever (e.g., `queued`) but it still passes “not failed”.
- **Version notes:** `2.2`.

#### Node: Check If Render Complete
- **Type / role:** `n8n-nodes-base.if` — decides whether to download.
- **Configuration choices:** Condition checks `status == "completed"`.
  - True branch → `Download Completed Video`
  - False branch → `Wait 30s for Rendering` (poll again)
- **Edge cases / failures:** If status becomes “canceled” or other terminal states, it will keep polling unless captured.
- **Version notes:** `2.2`.

#### Node: Download Completed Video
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads rendered video binary.
- **Configuration choices:**
  - URL: `https://api.openai.com/v1/videos/{{ $json.id }}/content`
  - Response format: `file` (binary)
  - `onError: continueRegularOutput` (important): if download fails, workflow continues.
- **Connections:** Output → `Upload to Cloudinary`.
- **Edge cases / failures:**
  - If download fails and continues, `Upload to Cloudinary` may receive missing binary and fail.
  - Sora content may expire (sticky note mentions ~24 hours).
- **Version notes:** `4.2`.

#### Node: Upload to Cloudinary
- **Type / role:** `n8n-nodes-base.httpRequest` — uploads clip binary to Cloudinary.
- **Configuration choices:**
  - POST `https://api.cloudinary.com/v1_1/{Cloud name here}/video/upload`
  - Multipart form-data:
    - `file` from binary field `data`
    - `upload_preset = n8n_integration`
  - Header sets multipart/form-data (Cloudinary typically sets boundary automatically; forcing Content-Type can sometimes break if boundary missing).
- **Connections:** Output → `Loop Through Videos` (feeds loop continuation).
- **Edge cases / failures:**
  - Cloud name placeholder must be replaced.
  - Upload preset must exist and allow unsigned upload (if no API key/secret used).
  - Binary field name must be exactly `data` (as downloaded).
- **Version notes:** `4.2`.

#### Node: Collect Video Id
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates Cloudinary `public_id` values into an array for stitching.
- **Configuration choices:**
  - Aggregates field `public_id`.
- **Connections:** Output → `Build Stitch URL`.
- **Edge cases / failures:**
  - If some uploads failed/skipped, `public_id` array may have fewer than 3 items; stitching code filters empties.
- **Version notes:** typeVersion `1`.

**Sticky note context:** “## Monitor & Download Videos” applies to this block.

---

### Block 2.6 — Build Cloudinary stitch URL + download stitched video
**Overview:** Builds a Cloudinary splice transformation URL to concatenate available clips, then downloads the resulting MP4.  
**Nodes involved:** `Build Stitch URL`, `Download Stitched Video`

#### Node: Build Stitch URL
- **Type / role:** `n8n-nodes-base.code` — constructs a Cloudinary URL for stitched output.
- **Configuration choices (interpreted):**
  - Requires setting `cloudName = "{replace with cloud name}"`
  - Reads `public_id` array from the aggregated input (`$input.first().json.public_id?.[i]`)
  - Filters invalid IDs
  - Applies Twitter-compatible transformations:
    - `vc_h264:baseline:3.0,ac_aac,ar_44100,q_auto:good,f_mp4`
  - Uses `fl_splice` + `l_video:<public_id>` layers to concatenate.
  - Returns `{ stitched_video_url: base }` or `{ stitched_video_url: null }` if none.
- **Connections:** Output → `Download Stitched Video`.
- **Edge cases / failures:**
  - If `stitched_video_url` is `null`, download will fail unless guarded (no guard is present).
  - If `public_id` contains folders, it replaces `/` with `:` for layer syntax.
- **Version notes:** typeVersion `2`.

#### Node: Download Stitched Video
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the final stitched MP4 as binary.
- **Configuration choices:**
  - URL from `{{$json.stitched_video_url}}`
  - Response: `file` (binary)
- **Connections:**
  - Output → `Prepare Upload Data`
  - Output also → `Combine Media ID + Binary` (input 1), but note: this merge expects INIT output on input 0 later; the actual flow merges after INIT returns.
- **Edge cases / failures:**
  - If stitch URL points to a not-yet-generated derived asset, Cloudinary may generate on-the-fly; initial request may be slow or timeout.
  - If URL is null/empty, request fails.
- **Version notes:** `4.2`.

---

### Block 2.7 — Twitter/X chunked upload (video)
**Overview:** Prepares metadata (bytes + mime type), initializes upload, appends the binary as a single segment, finalizes upload, and waits for processing success.  
**Nodes involved:** `Prepare Upload Data`, `Twitter Upload - INIT`, `Combine Media ID + Binary`, `Twitter Upload - APPEND`, `Finalize Upload`, `Wait for Twitter Processing`, `Check Twitter Processing Status`, `Check If Twitter Ready`

#### Node: Prepare Upload Data
- **Type / role:** `n8n-nodes-base.code` — calculates `total_bytes` and passes through binary.
- **Configuration choices:**
  - Reads `items[0].binary.data`
  - Computes length correctly whether `data` is a Buffer or base64.
  - Outputs:
    - `json.total_bytes`
    - `json.media_type` default `video/mp4`
    - `binary.data` preserved
- **Connections:** Output → `Twitter Upload - INIT`.
- **Edge cases / failures:**
  - If binary is missing (failed download), code node throws.
- **Version notes:** typeVersion `2`.

#### Node: Twitter Upload - INIT
- **Type / role:** `n8n-nodes-base.httpRequest` — starts Twitter chunked upload session.
- **Configuration choices:**
  - POST `https://upload.twitter.com/1.1/media/upload.json`
  - multipart form-data:
    - `command=INIT`
    - `total_bytes={{$json.total_bytes}}`
    - `media_type={{$json.media_type}}`
    - `media_category=tweet_video`
  - Authentication: `genericCredentialType` with `oAuth1Api`
- **Connections:** Output → `Combine Media ID + Binary` (input 0).
- **Edge cases / failures:**
  - OAuth 1.0a required for v1.1 upload endpoints; wrong credentials produce 401/403.
  - If `total_bytes` mismatches actual upload, later steps fail.
- **Version notes:** `4.2`.

#### Node: Combine Media ID + Binary
- **Type / role:** `n8n-nodes-base.merge` — combines INIT response (media_id) with the binary payload.
- **Configuration choices:** `mode=combine`, `combineBy=combineByPosition`.
- **Connections:**
  - Input 0: from `Twitter Upload - INIT` (JSON with `media_id_string`)
  - Input 1: from `Download Stitched Video` (binary)
  - Output → `Twitter Upload - APPEND`
- **Edge cases / failures:**
  - Requires both inputs to arrive; if download fails, APPEND can’t run.
- **Version notes:** `3.2`.

#### Node: Twitter Upload - APPEND
- **Type / role:** `n8n-nodes-base.httpRequest` — uploads the media bytes.
- **Configuration choices:**
  - POST `https://upload.twitter.com/1.1/media/upload.json`
  - multipart form-data:
    - `command=APPEND`
    - `media_id={{$json.media_id_string}}`
    - `segment_index=0`
    - `media` from binary field `data`
  - OAuth1 auth.
- **Connections:** Output → `Finalize Upload`.
- **Edge cases / failures:**
  - This implementation always uses `segment_index=0` and only one APPEND call. Large files may exceed single-request limits; true chunking would require splitting into multiple segments.
- **Version notes:** `4.2`.

#### Node: Finalize Upload
- **Type / role:** `n8n-nodes-base.httpRequest` — completes upload and triggers processing.
- **Configuration choices:**
  - POST `https://upload.twitter.com/1.1/media/upload.json`
  - multipart form-data:
    - `command=FINALIZE`
    - `media_id` from `$('Combine Media ID + Binary').first().json.media_id_string`
- **Connections:** Output → `Wait for Twitter Processing`.
- **Edge cases / failures:**
  - If media is still uploading or corrupted, FINALIZE fails.
- **Version notes:** `4.2`.

#### Node: Wait for Twitter Processing
- **Type / role:** `n8n-nodes-base.wait` — introduces delay between status checks.
- **Configuration choices:** No explicit `amount` set in JSON (uses node defaults; in n8n this typically means “wait until resumed” if configured that way, but here it is used like a delay loop; you should set a specific delay, e.g., 10–30 seconds).
- **Connections:** Output → `Check Twitter Processing Status`.
- **Edge cases / failures:** If no delay is set, polling can be too aggressive or behave unexpectedly.
- **Version notes:** `1.1`.

#### Node: Check Twitter Processing Status
- **Type / role:** `n8n-nodes-base.httpRequest` — polls processing state.
- **Configuration choices:**
  - GET `https://upload.twitter.com/1.1/media/upload.json`
  - Query:
    - `command=STATUS`
    - `media_id={{$json.media_id_string}}`
  - OAuth1 auth.
- **Connections:** Output → `Check If Twitter Ready`.
- **Edge cases / failures:** Twitter can return `processing_info` absent for already-ready media; IF node expects `.processing_info.state`.
- **Version notes:** `4.2`.

#### Node: Check If Twitter Ready
- **Type / role:** `n8n-nodes-base.if` — continues when processing succeeded.
- **Configuration choices:**
  - Condition: `processing_info.state == "succeeded"`
  - True → `Post a Tweet`
  - False → `Wait for Twitter Processing` (poll again)
- **Edge cases / failures:**
  - If state is `failed`, it will keep waiting forever (no terminal failure branch configured).
  - If `processing_info` missing, expression may evaluate to null and branch false, causing looping.
- **Version notes:** `2.2`.

**Sticky note context:** “## Upload to Twitter/X” applies to this block.

---

### Block 2.8 — Post tweet and update tracking sheet
**Overview:** Posts the final tweet containing the stitched video and then appends a “Posted” row in the tracking sheet.  
**Nodes involved:** `Post a Tweet`, `Update Status`

#### Node: Post a Tweet
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Twitter API v2 to post the tweet.
- **Configuration choices:**
  - POST `https://api.twitter.com/2/tweets`
  - JSON body:
    - `text` from `$('ASMR Cutting Ideas').first().json.output.Twitter_Text`
    - `media.media_ids = [ "{{ $json.media_id_string }}" ]`
  - OAuth1 auth used (works for many setups, though Twitter v2 often uses OAuth2; depends on app permissions).
- **Connections:** Output → `Update Status`.
- **Edge cases / failures:**
  - If Twitter app lacks v2 write permissions, returns 403.
  - If caption exceeds 280 chars, request fails.
- **Version notes:** `4.2`.

#### Node: Update Status
- **Type / role:** `n8n-nodes-base.googleSheets` — logs “Posted” status.
- **Configuration choices:**
  - Operation: `append` (despite name “Update Status”)
  - Writes only `Status = Posted` (other fields removed from schema mapping).
- **Connections:** Input from `Post a Tweet`; no further outputs.
- **Important behavior note:** This does **not** update the previously appended “In Progress” row; it appends a new row with only `Status=Posted`.
- **Edge cases / failures:** Same OAuth and schema issues as the first Sheets node.
- **Version notes:** `4.6`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Time-based workflow entry | — | ASMR Cutting Ideas |  |
| OpenAI Chat Model | lmChatOpenAi | Provides GPT-5.1 model to agent | — | ASMR Cutting Ideas (ai_languageModel) | ## Generate Clip Idea |
| AI Reasoning Tool | toolThink | Think tool for agent reasoning | — | ASMR Cutting Ideas (ai_tool) | ## Generate Clip Idea |
| Format JSON Output | outputParserStructured | Enforce structured JSON output | — | ASMR Cutting Ideas (ai_outputParser) | ## Generate Clip Idea |
| ASMR Cutting Ideas | langchain.agent | Generate 3 clip prompts + caption | Schedule Trigger | Save Category and Clip Scripts | ## Generate Clip Idea |
| Save Category and Clip Scripts | googleSheets | Append prompts + “In Progress” to sheet | ASMR Cutting Ideas | Create Sora Video Scene - 1; Create Sora Video Scene - 2; Create Sora Video Scene - 3 |  |
| Create Sora Video Scene - 1 | httpRequest | Create Sora job for clip 1 | Save Category and Clip Scripts | Combine All Scenes | ## Generate Clips |
| Create Sora Video Scene - 2 | httpRequest | Create Sora job for clip 2 | Save Category and Clip Scripts | Combine All Scenes | ## Generate Clips |
| Create Sora Video Scene - 3 | httpRequest | Create Sora job for clip 3 | Save Category and Clip Scripts | Combine All Scenes | ## Generate Clips |
| Combine All Scenes | merge | Merge 3 parallel Sora job responses | Scene - 1; Scene - 2; Scene - 3 | Loop Through Videos | ## Generate Clips |
| Loop Through Videos | splitInBatches | Iterate through video jobs / uploaded clips | Combine All Scenes; Upload to Cloudinary; Check If Still Processing (false) | Collect Video Id; Wait 30s for Rendering | ## Monitor & Download Videos |
| Wait 30s for Rendering | wait | Poll delay for Sora rendering | Loop Through Videos; Check If Render Complete (false) | Check Video Status | ## Monitor & Download Videos |
| Check Video Status | httpRequest | Poll OpenAI job status | Wait 30s for Rendering | Check If Still Processing | ## Monitor & Download Videos |
| Check If Still Processing | if | Skip failed jobs | Check Video Status | Check If Render Complete (true); Loop Through Videos (false) | ## Monitor & Download Videos |
| Check If Render Complete | if | Download when completed, else keep polling | Check If Still Processing | Download Completed Video (true); Wait 30s for Rendering (false) | ## Monitor & Download Videos |
| Download Completed Video | httpRequest | Download Sora clip binary | Check If Render Complete | Upload to Cloudinary | ## Monitor & Download Videos |
| Upload to Cloudinary | httpRequest | Upload clip to Cloudinary (public_id output) | Download Completed Video | Loop Through Videos | ## Monitor & Download Videos |
| Collect Video Id | aggregate | Aggregate Cloudinary public_id values | Loop Through Videos | Build Stitch URL |  |
| Build Stitch URL | code | Construct Cloudinary splice URL | Collect Video Id | Download Stitched Video |  |
| Download Stitched Video | httpRequest | Download stitched MP4 binary | Build Stitch URL | Prepare Upload Data; Combine Media ID + Binary (input 1) |  |
| Prepare Upload Data | code | Compute bytes/mime for Twitter INIT | Download Stitched Video | Twitter Upload - INIT | ## Upload to Twitter/X |
| Twitter Upload - INIT | httpRequest | Start Twitter chunked upload | Prepare Upload Data | Combine Media ID + Binary (input 0) | ## Upload to Twitter/X |
| Combine Media ID + Binary | merge | Merge INIT media_id + stitched binary | Twitter Upload - INIT; Download Stitched Video | Twitter Upload - APPEND | ## Upload to Twitter/X |
| Twitter Upload - APPEND | httpRequest | Upload media segment 0 | Combine Media ID + Binary | Finalize Upload | ## Upload to Twitter/X |
| Finalize Upload | httpRequest | Finalize upload for processing | Twitter Upload - APPEND | Wait for Twitter Processing | ## Upload to Twitter/X |
| Wait for Twitter Processing | wait | Delay between STATUS polls | Finalize Upload; Check If Twitter Ready (false) | Check Twitter Processing Status | ## Upload to Twitter/X |
| Check Twitter Processing Status | httpRequest | Poll processing state | Wait for Twitter Processing | Check If Twitter Ready | ## Upload to Twitter/X |
| Check If Twitter Ready | if | Proceed when processing succeeded | Check Twitter Processing Status | Post a Tweet (true); Wait for Twitter Processing (false) | ## Upload to Twitter/X |
| Post a Tweet | httpRequest | Create tweet with media | Check If Twitter Ready (true) | Update Status | ## Upload to Twitter/X |
| Update Status | googleSheets | Append “Posted” status row | Post a Tweet | — | ## Upload to Twitter/X |
| Sticky Note | stickyNote | Comment container | — | — | ## Generate Clips |
| Sticky Note1 | stickyNote | Comment container | — | — | ## Monitor & Download Videos |
| Sticky Note2 | stickyNote | Comment container | — | — | ## Generate Clip Idea |
| Sticky Note3 | stickyNote | Comment container | — | — | (See section 5 for full content) |
| Sticky Note5 | stickyNote | Comment container | — | — | ## Upload to Twitter/X |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: “Generate Sora videos, stitch clips, and post to Twitter”.

2) **Add Trigger**
   - Add **Schedule Trigger**.
   - Configure an interval (e.g., every day/hour) as desired.

3) **Add AI generation (LangChain agent stack)**
   1. Add **OpenAI Chat Model** (LangChain).
      - Set model to **gpt-5.1**.
      - Configure **OpenAI API** credentials (API key with access to GPT-5.1).
   2. Add **AI Reasoning Tool (Think)**.
   3. Add **Structured Output Parser**.
      - Provide a schema/example that includes:
        - `Category`
        - `Twitter_Text`
        - `Clip_1/2/3` each with `Prompt`, `Sound`, `Duration` (8 or 12).
   4. Add **Agent** node (LangChain).
      - System message: include the constraints (static frontal shot, knife cuts toward camera, 3 prompts, durations 8/12, moderation-safe language, etc.).
      - Enable output parser and connect the parser node.
      - Connect **OpenAI Chat Model** to the agent’s language model input.
      - Connect **Think tool** to the agent’s tools input.
   5. Connect: **Schedule Trigger → Agent**.

4) **Add Google Sheets tracking (In Progress)**
   - Add **Google Sheets** node: “Save Category and Clip Scripts”.
   - Auth: configure **Google Sheets OAuth2** credential.
   - Operation: **Append**.
   - Select Spreadsheet + Sheet.
   - Map columns:
     - `Status` = `In Progress`
     - `Scene 1` = `{{ $json.output.Clip_1.Prompt }}`
     - `Scene 2` = `{{ $json.output.Clip_2.Prompt }}`
     - `Scene 3` = `{{ $json.output.Clip_3.Prompt }}`
     - `Category` = `{{ $json.output.Category }}`
   - Connect: **Agent → Save Category and Clip Scripts**.

5) **Create 3 parallel Sora v2 video jobs**
   - Add 3 **HTTP Request** nodes:
     - “Create Sora Video Scene - 1/2/3”
   - Each node:
     - Method: **POST**
     - URL: `https://api.openai.com/v1/videos`
     - Auth: use a secure credential approach if possible (avoid hardcoding bearer tokens).
     - Body: JSON with:
       - `model`: `sora-2`
       - `prompt`: combine scene prompt + corresponding sound (sanitize newlines/quotes)
       - `size`: `720x1280`
       - `seconds`: corresponding duration
   - Connect: **Save Category and Clip Scripts → (all 3 Create Sora nodes)**.

6) **Merge the 3 Sora job responses**
   - Add **Merge** node “Combine All Scenes”.
   - Mode: merge **3 inputs**.
   - Connect each Create Scene node to one input of Merge.

7) **Iterate and poll each job**
   1. Add **Split In Batches** node “Loop Through Videos”.
      - Set batch size to **1** (recommended for predictable polling).
      - Connect: **Combine All Scenes → Loop Through Videos**.
   2. Add **Wait** node “Wait 30s for Rendering”.
      - Amount: **30 seconds**.
      - Connect: **Loop Through Videos (batch output) → Wait 30s**.
   3. Add **HTTP Request** node “Check Video Status”.
      - GET `https://api.openai.com/v1/videos/{{ $json.id }}`
      - Bearer auth to OpenAI.
      - Connect: **Wait 30s → Check Video Status**.
   4. Add **IF** node “Check If Still Processing”.
      - Condition: `{{$json.status}} != failed`
      - True → proceed; False → skip (connect to loop continue).
      - Connect: **Check Video Status → Check If Still Processing**.
   5. Add **IF** node “Check If Render Complete”.
      - Condition: `{{$json.status}} == completed`
      - True → download; False → wait again (connect back to Wait 30s).
      - Connect: **Check If Still Processing (true) → Check If Render Complete**.
   6. Add **HTTP Request** “Download Completed Video”.
      - GET `https://api.openai.com/v1/videos/{{ $json.id }}/content`
      - Response: **File** (binary).
      - (Optional but matches JSON): set “On Error” to **Continue**.
      - Connect: **Check If Render Complete (true) → Download Completed Video**.

8) **Upload each completed clip to Cloudinary**
   - Add **HTTP Request** “Upload to Cloudinary”.
     - POST `https://api.cloudinary.com/v1_1/<your_cloud_name>/video/upload`
     - Content type: **multipart/form-data**
     - Body:
       - `file` = binary from field `data`
       - `upload_preset` = `n8n_integration` (create this in Cloudinary; ensure it matches your security model)
   - Connect: **Download Completed Video → Upload to Cloudinary**.
   - Connect: **Upload to Cloudinary → Loop Through Videos** (to continue processing next clip).
   - Connect: **Check If Still Processing (false) → Loop Through Videos** (to skip failed jobs).

9) **Aggregate Cloudinary public_ids after loop**
   - Add **Aggregate** node “Collect Video Id”.
     - Aggregate field: `public_id`
   - Connect: **Loop Through Videos (done output) → Collect Video Id**.

10) **Build stitch URL**
   - Add **Code** node “Build Stitch URL”.
   - Implement logic:
     - Read the first three `public_id` values from the aggregate output.
     - Construct a Cloudinary URL using `fl_splice` and `l_video:<public_id>`.
     - Include H.264 baseline + AAC audio transformations for Twitter compatibility.
   - Set your Cloudinary cloud name in code.
   - Connect: **Collect Video Id → Build Stitch URL**.

11) **Download stitched video**
   - Add **HTTP Request** “Download Stitched Video”.
     - URL = `{{$json.stitched_video_url}}`
     - Response: **File**
   - Connect: **Build Stitch URL → Download Stitched Video**.

12) **Prepare Twitter upload metadata**
   - Add **Code** node “Prepare Upload Data”.
     - Compute `total_bytes` from binary.
     - Set `media_type` to `video/mp4`.
     - Pass through binary `data`.
   - Connect: **Download Stitched Video → Prepare Upload Data**.

13) **Twitter chunked upload**
   1. Configure **Twitter OAuth 1.0a** credentials in n8n (required for `upload.twitter.com/1.1` endpoints).
   2. Add **HTTP Request** “Twitter Upload - INIT”.
      - POST `https://upload.twitter.com/1.1/media/upload.json`
      - multipart:
        - `command=INIT`
        - `total_bytes={{$json.total_bytes}}`
        - `media_type={{$json.media_type}}`
        - `media_category=tweet_video`
      - Auth: OAuth1
      - Connect: **Prepare Upload Data → INIT**
   3. Add **Merge** node “Combine Media ID + Binary”
      - Mode: **Combine by position**
      - Input 0 from INIT, input 1 from Download Stitched Video
      - Connect:
        - **INIT → Merge (input 0)**
        - **Download Stitched Video → Merge (input 1)**
   4. Add **HTTP Request** “Twitter Upload - APPEND”.
      - POST same endpoint
      - multipart:
        - `command=APPEND`
        - `media_id={{$json.media_id_string}}`
        - `segment_index=0`
        - `media` = binary `data`
      - Auth: OAuth1
      - Connect: **Merge → APPEND**
   5. Add **HTTP Request** “Finalize Upload”.
      - multipart:
        - `command=FINALIZE`
        - `media_id={{ $('Combine Media ID + Binary').first().json.media_id_string }}`
      - Connect: **APPEND → FINALIZE**

14) **Wait and poll Twitter processing**
   1. Add **Wait** node “Wait for Twitter Processing”.
      - Set delay (recommended: 10–30 seconds).
      - Connect: **FINALIZE → Wait**
   2. Add **HTTP Request** “Check Twitter Processing Status”.
      - GET `https://upload.twitter.com/1.1/media/upload.json`
      - Query:
        - `command=STATUS`
        - `media_id={{$json.media_id_string}}`
      - Connect: **Wait → STATUS**
   3. Add **IF** “Check If Twitter Ready”.
      - Condition: `{{$json.processing_info.state}} == succeeded`
      - True → post tweet
      - False → back to Wait
      - Connect: **STATUS → IF**, **IF(false) → Wait**

15) **Post tweet with attached media**
   - Add **HTTP Request** “Post a Tweet”.
     - POST `https://api.twitter.com/2/tweets`
     - JSON:
       - `text`: `{{ $('ASMR Cutting Ideas').first().json.output.Twitter_Text }}`
       - `media.media_ids`: `[ "{{ $json.media_id_string }}" ]`
     - Auth: OAuth1 (or OAuth2 if you prefer; match your app permissions).
   - Connect: **Check If Twitter Ready (true) → Post a Tweet**

16) **Log Posted status**
   - Add **Google Sheets** node “Update Status”.
     - Operation: Append
     - `Status = Posted`
   - Connect: **Post a Tweet → Update Status**
   - If you want a true update, replace this with “Update row” using a unique key captured from the first append.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Workflow summary + requirements**: Generate ideas (GPT-5.1), create 3 Sora v2 clips, monitor/download, upload to Cloudinary, stitch via transformations, upload to Twitter using chunked upload, post tweet, track status in Sheets. | From sticky note: “# Generate Sora videos, stitch clips, and post to Twitter” |
| Required credentials: OpenAI API key (GPT-5.1 + Sora v2), Google Sheets OAuth, Cloudinary account (upload preset `n8n_integration`), Twitter OAuth 1.0a. | Sticky note requirements section |
| Configure Cloudinary cloud name in both “Build Stitch URL” and “Upload to Cloudinary”. | Sticky note configuration section |
| Google Sheets document: `https://docs.google.com/spreadsheets/d/1qUir1c-_ScMnoYVoQ0W41nsv5IpLW6rjK8HUNqvNnAg/edit?usp=drivesdk` | Present in node cached URLs (example) |
| Notes: failed Sora videos are skipped; Twitter processing takes ~10–30s; Twitter max video size ~512MB; Sora videos expire after ~24 hours. | Sticky note “Important Notes” section |