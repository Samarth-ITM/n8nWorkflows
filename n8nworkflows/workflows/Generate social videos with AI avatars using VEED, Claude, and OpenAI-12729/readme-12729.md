Generate social videos with AI avatars using VEED, Claude, and OpenAI

https://n8nworkflows.xyz/workflows/generate-social-videos-with-ai-avatars-using-veed--claude--and-openai-12729


# Generate social videos with AI avatars using VEED, Claude, and OpenAI

## 1. Workflow Overview

**Purpose:** Automatically generate short-form vertical social videos (TikTok/Reels style) featuring an AI “talking-head” avatar. The workflow uses **Claude (Anthropic)** to write the script/caption (and optionally an avatar image prompt), **OpenAI** to generate the avatar image + TTS voiceover, **tmpfiles.org** as temporary hosting for media files, **VEED** to render a lip-synced video, then **Google Drive** to store the final MP4 and **Google Sheets** to log metadata and links.

**Target use cases:**
- Rapid content production for content creators/marketers (1+ videos per run).
- A/B testing “angles” and hook styles across multiple videos.
- Semi-custom generation: optionally supply a custom avatar description and/or a custom script.

### 1.1 Entry & Configuration
Manual trigger → set a JSON configuration object → generate N “video task” items.

### 1.2 Iteration / Batch Control
Split tasks into batches and loop until all videos are produced.

### 1.3 AI Content Generation (Claude)
Build a dynamic prompt depending on whether the user provided custom avatar/script → call Anthropic Messages API → parse JSON response robustly.

### 1.4 Avatar + Audio Generation (OpenAI)
Generate a portrait avatar image (base64) → convert to binary → upload image to tmpfiles → generate TTS MP3 → normalize binary metadata → upload audio to tmpfiles.

### 1.5 Video Rendering (VEED)
Assemble VEED request payload (URLs/resolution/aspect ratio) → VEED node renders lip-synced video from image+audio.

### 1.6 Storage & Logging (Google)
Prepare filename and extract VEED video URL → download MP4 → upload to Google Drive → prepare final metadata → append a row in Google Sheets → continue loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Workflow Configuration
**Overview:** Starts the workflow manually and defines all user-editable settings (topic, keys, output settings).  
**Nodes involved:**  
- When clicking 'Execute workflow'  
- ⚙️ Workflow Configuration  
- 📋 Generate Video Tasks  

#### Node: When clicking 'Execute workflow'
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – workflow entrypoint.
- **Configuration choices:** No parameters; run is started by a human in the editor.
- **I/O connections:** Outputs to **⚙️ Workflow Configuration**.
- **Edge cases:** None (except user not executing).

#### Node: ⚙️ Workflow Configuration
- **Type / role:** Set node (`n8n-nodes-base.set`) – emits a single JSON config object.
- **Configuration choices (interpreted):**
  - Uses **raw JSON output** to define:
    - `topic`, `intention` (informative/lead_generation/disruption), `brand_name`, `target_audience`
    - `trending_hashtags`
    - `num_videos`
    - `anthropic_api_key`, `openai_api_key` (stored directly in data; not n8n credentials)
    - `video_resolution` (e.g., `720p`), `video_aspect_ratio` (`9:16`)
    - `custom_avatar_description`, `custom_script` (optional)
- **Key variables:** Entire config becomes `$json` downstream.
- **I/O connections:** Input from trigger; output to **📋 Generate Video Tasks**.
- **Edge cases / failures:**
  - Missing/invalid API keys will cause downstream API auth failures.
  - Non-numeric `num_videos` can produce unexpected looping behavior (Code node defaults to 1 if falsy).

#### Node: 📋 Generate Video Tasks
- **Type / role:** Code node (`n8n-nodes-base.code`) – fan-out: creates N items for N videos.
- **Configuration choices (interpreted):**
  - Reads config via `$input.first().json`.
  - Builds a task list with `video_index` plus rotating:
    - `content_angle` (problem-solution, myth-busting, quick-tip, before-after, trend-commentary)
    - `hook_style` (question, controversial, number, transformation, news)
- **Outputs:** `return tasks;` where each item includes original config + per-video fields.
- **I/O connections:** Output to **🔄 Loop Through Videos**.
- **Edge cases / failures:**
  - `num_videos` very large may cause long runs and API rate limiting downstream.
  - If config missing expected fields, prompt generation still works but content quality may degrade.

---

### Block 2 — Batch Iteration / Loop Control
**Overview:** Ensures videos are processed one at a time and loops back until all tasks are complete.  
**Nodes involved:**  
- 🔄 Loop Through Videos  
- 🔄 Continue Loop1  

#### Node: 🔄 Loop Through Videos
- **Type / role:** Split in Batches (`n8n-nodes-base.splitInBatches`) – iterator.
- **Configuration choices:**
  - Batch size not explicitly set in parameters (defaults apply; typically 1 item per batch in many templates unless configured).
  - Uses **two outputs**:
    - Output 0: “No items left” path (unused here)
    - Output 1: “Items” path → main processing
- **I/O connections:**
  - Receives items from **📋 Generate Video Tasks**
  - Output (items path) → **🧠 Build Claude Prompt**
  - Loop continuation: **🔄 Continue Loop1** feeds back into this node
- **Edge cases / failures:**
  - If batch size > 1, downstream nodes expecting single-item context may still work but could mis-associate cross-node references (several Code nodes use `$('Node Name').item.json`, which is safest when processing one item at a time).

#### Node: 🔄 Continue Loop1
- **Type / role:** NoOp (`n8n-nodes-base.noOp`) – purely structural to reconnect flow back to batch node.
- **I/O connections:** From **📝 Log to Sheets** → back to **🔄 Loop Through Videos**.
- **Edge cases:** None.

---

### Block 3 — AI Content Generation (Claude)
**Overview:** Builds a structured prompt to generate JSON output for script/caption/avatar prompt, calls Anthropic Claude, then parses/normalizes the result with fallbacks.  
**Nodes involved:**  
- 🧠 Build Claude Prompt  
- 🤖 Claude: Generate Content  
- 📋 Parse Claude Response  

#### Node: 🧠 Build Claude Prompt
- **Type / role:** Code node – constructs the Claude instruction prompt dynamically.
- **Configuration choices:**
  - Detects custom inputs:
    - `hasCustomAvatar` if `custom_avatar_description` is non-empty
    - `hasCustomScript` if `custom_script` is non-empty
  - Chooses task instructions and required JSON schema depending on cases:
    1) No custom avatar + no custom script: generate image prompt + audio script + caption
    2) Custom avatar only: generate audio script + caption
    3) Custom script only: generate image prompt + caption
    4) Both custom: generate caption only
  - Injects:
    - intention guide (`informative`, `lead_generation`, `disruption`)
    - hook style guide (question/controversial/number/transformation/news)
  - Forces **“Respond ONLY with valid JSON”** in a specified format.
- **Key outputs:**
  - `claude_prompt` (string)
  - `has_custom_avatar`, `has_custom_script`
- **I/O connections:** Output to **🤖 Claude: Generate Content**.
- **Edge cases / failures:**
  - If `intention` or `hook_style` has unexpected values, defaults are used.
  - Prompt size is small; unlikely to exceed limits, but very long custom script/description could.

#### Node: 🤖 Claude: Generate Content
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – calls Anthropic Messages API.
- **Configuration choices:**
  - **POST** `https://api.anthropic.com/v1/messages`
  - Headers:
    - `x-api-key: {{$json.anthropic_api_key}}`
    - `anthropic-version: 2023-06-01`
    - `Content-Type: application/json`
  - JSON body:
    - `model: "claude-sonnet-4-20250514"`
    - `max_tokens: 2000`
    - `messages[0].content`: `{{ JSON.stringify($json.claude_prompt) }}`
      - Note: this wraps the prompt in JSON quotes, resulting in Claude receiving a quoted string (it usually still works, but it’s slightly unusual).
- **I/O connections:** Output to **📋 Parse Claude Response**.
- **Failure modes:**
  - 401/403 invalid key; 429 rate limit; 5xx Anthropic outages.
  - Model name availability (future changes/deprecations).
  - If Anthropic returns content in an unexpected shape, parse step may fall back.

#### Node: 📋 Parse Claude Response
- **Type / role:** Code node – extracts text, parses JSON, applies fallbacks, merges with config.
- **Configuration choices / logic:**
  - Reads Claude response from `$input.first().json`.
  - Reads the config context from `$('🧠 Build Claude Prompt').item.json` (cross-node reference).
  - Extracts `responseText` from `claudeResponse.content[0].text`.
  - Attempts to `JSON.parse()` after stripping ``` code fences if present.
  - On parse error: uses fallback defaults (`caption`, `avatar_gender`, etc.).
  - Produces normalized fields:
    - `script_image` = either enhanced `custom_avatar_description`, or parsed `image_prompt`, or fallback
    - `script_audio` = either `custom_script`, or parsed `audio_script`, or fallback
- **Output fields include:** `caption`, `avatar_gender`, `content_theme`, video settings, keys, plus `used_custom_avatar/script`.
- **I/O connections:** Output to **🎨 Generate Avatar (OpenAI)**.
- **Edge cases / failures:**
  - If Claude returns valid JSON but with different keys, some fields may be empty and fallbacks apply.
  - Cross-node references can misbehave under multi-item concurrency; safest with batch size = 1.

---

### Block 4 — Avatar & Audio Generation + Temporary Hosting
**Overview:** Generates an avatar image and voiceover audio via OpenAI, then uploads both to tmpfiles.org to obtain public URLs for VEED.  
**Nodes involved:**  
- 🎨 Generate Avatar (OpenAI)  
- 📸 Extract Image Data  
- ☁️ Upload Image  
- 💾 Store Image URL  
- 🔊 Generate Audio (TTS)  
- 🎵 Convert Audio  
- ☁️ Upload Audio  
- 📦 Prepare VEED Request  

#### Node: 🎨 Generate Avatar (OpenAI)
- **Type / role:** HTTP Request – OpenAI Images API.
- **Configuration choices:**
  - **POST** `https://api.openai.com/v1/images/generations`
  - Headers: `Authorization: Bearer {{$json.openai_api_key}}`
  - Body:
    - `model: "gpt-image-1"`
    - `prompt: JSON.stringify($json.script_image)`
    - `size: 1024x1536` (portrait)
    - `quality: high`, `n: 1`
- **I/O connections:** Output to **📸 Extract Image Data**.
- **Failure modes:**
  - 401 invalid key; 429 rate limit; content policy rejections; model changes.
  - If API returns URL instead of base64 (or different shape), next node throws error.

#### Node: 📸 Extract Image Data
- **Type / role:** Code node – converts base64 image into n8n binary.
- **Configuration choices / logic:**
  - Expects `imageResponse.data[0].b64_json`.
  - Throws explicit error if missing base64.
  - Outputs:
    - JSON: previous data copied from `$('📋 Parse Claude Response').item.json`
    - Binary: `image_data` with `mimeType: image/png`, `fileName: avatar.png`
- **I/O connections:** Output to **☁️ Upload Image**.
- **Edge cases:** OpenAI response shape mismatch; base64 too large; memory constraints.

#### Node: ☁️ Upload Image
- **Type / role:** HTTP Request – uploads binary to tmpfiles.org.
- **Configuration choices:**
  - **POST** `https://tmpfiles.org/api/v1/upload`
  - `multipart-form-data`, form field `file` from binary `image_data`
  - Response format: JSON
- **I/O connections:** Output to **💾 Store Image URL**.
- **Failure modes:** tmpfiles downtime/rate limits; file size limits; response format changes.

#### Node: 💾 Store Image URL
- **Type / role:** Code node – converts tmpfiles “page URL” to a direct download URL.
- **Configuration choices:**
  - Uses regex replace:
    - `http://tmpfiles.org/<id>/<name>` → `https://tmpfiles.org/dl/<id>/<name>`
  - Stores as `public_image_url`
- **I/O connections:** Output to **🔊 Generate Audio (TTS)**.
- **Edge cases:** If tmpfiles returns `https://` already or a different structure, regex may not match, producing an invalid URL.

#### Node: 🔊 Generate Audio (TTS)
- **Type / role:** HTTP Request – OpenAI TTS.
- **Configuration choices:**
  - **POST** `https://api.openai.com/v1/audio/speech`
  - Headers: `Authorization: Bearer {{$json.openai_api_key}}`
  - Body:
    - `model: "tts-1-hd"`
    - `input: JSON.stringify($json.script_audio)`
    - `voice: "nova"`
    - `response_format: "mp3"`
  - Response: **file** stored in binary property `audio`
- **I/O connections:** Output to **🎵 Convert Audio**.
- **Failure modes:** 401/429; text too long; voice/model availability; binary response issues.

#### Node: 🎵 Convert Audio
- **Type / role:** Code node – renames/normalizes binary field.
- **Configuration choices:**
  - Copies `item.binary.audio` → `item.binary.audio_mp3` with:
    - `fileName: voiceover.mp3`
    - `mimeType: audio/mpeg`
- **I/O connections:** Output to **☁️ Upload Audio**.
- **Edge cases:** If TTS node didn’t output `binary.audio`, upload will fail or upload empty.

#### Node: ☁️ Upload Audio
- **Type / role:** HTTP Request – uploads MP3 to tmpfiles.org.
- **Configuration choices:**
  - **POST** `https://tmpfiles.org/api/v1/upload`
  - multipart `file` from binary field `audio_mp3`
  - Response JSON
- **I/O connections:** Output to **📦 Prepare VEED Request**.
- **Failure modes:** tmpfiles limits/outages; large file.

#### Node: 📦 Prepare VEED Request
- **Type / role:** Code node – assembles VEED-ready payload.
- **Configuration choices:**
  - Converts tmpfiles URL to direct download URL using same regex.
  - Pulls image URL and other metadata from `$('💾 Store Image URL').item.json`.
  - Outputs: `image_url`, `audio_url`, `resolution`, `aspect_ratio`, and metadata fields.
- **I/O connections:** Output to **🎬 Generate Video (VEED)**.
- **Edge cases:** Cross-node item reference issues if multiple items processed concurrently.

---

### Block 5 — Video Rendering (VEED)
**Overview:** Sends the public image/audio URLs to VEED to render a lip-synced talking-head video.  
**Nodes involved:**  
- 🎬 Generate Video (VEED)

#### Node: 🎬 Generate Video (VEED)
- **Type / role:** VEED node (`n8n-nodes-veed.veed`) – external rendering integration.
- **Configuration choices:**
  - `audioUrl`: `{{$json.audio_url}}`
  - `imageUrl`: `{{$json.image_url}}`
  - `resolution`: `{{$json.resolution}}` (defaulted earlier to 720p)
  - `aspectRatio`: `{{$json.aspect_ratio}}` (defaulted earlier to 9:16)
- **Credentials:** Requires VEED/FAL.ai credential connection (as noted in sticky note).
- **I/O connections:** Output to **📁 Prepare Upload**.
- **Failure modes:**
  - Credential/auth failures with VEED.
  - Rendering queue delays/timeouts.
  - Invalid or inaccessible `image_url`/`audio_url` (tmpfiles links expired/blocked).
  - If VEED response doesn’t contain a recognized URL, later step marks status error.

---

### Block 6 — Download, Drive Upload, Sheets Logging, Loop Continuation
**Overview:** Extracts the final video URL, downloads it, uploads to Google Drive, logs metadata to Google Sheets, then loops to process the next video.  
**Nodes involved:**  
- 📁 Prepare Upload  
- ⬇️ Download Video  
- 📤 Upload to Drive  
- ✅ Prepare Final Data  
- 📝 Log to Sheets  
- 🔄 Continue Loop1  

#### Node: 📁 Prepare Upload
- **Type / role:** Code node – builds filename/folder label and extracts a usable VEED video URL.
- **Configuration choices / logic:**
  - Builds:
    - `month_folder` like “January 2026” (note: only computed; not used to choose Drive folder in this version)
    - `file_name` like `YYYY-MM-DD-<topic>_<index>_<timestamp>.mp4`
  - Extracts `videoUrl` from various possible response shapes:
    - `veedResult.video.url`
    - `veedResult.output.video_url`
    - `veedResult.videoUrl`
    - `veedResult.url`
  - Sets `status: 'done'` if URL exists else `'error'`
  - Stores `veed_raw_response` for debugging
- **I/O connections:** Output to **⬇️ Download Video**.
- **Edge cases:**
  - If VEED returns a temporary URL requiring auth/cookies, download will fail.
  - If URL is empty, download node will error (no guard node exists).

#### Node: ⬇️ Download Video
- **Type / role:** HTTP Request – downloads the MP4 as binary.
- **Configuration choices:**
  - URL: `{{$json.video_url}}`
  - Response format: file (binary)
- **I/O connections:** Output to **📤 Upload to Drive**.
- **Failure modes:** 404/403 expired link; large file timeouts; VEED not finished yet.

#### Node: 📤 Upload to Drive
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) – uploads binary to Drive.
- **Configuration choices:**
  - File name: from `$('📁 Prepare Upload').item.json.file_name`
  - Drive: “My Drive”
  - Folder: configured via URL placeholder `YOUR_GOOGLE_DRIVE_FOLDER_URL`
  - (Binary input is taken from previous node’s file download output; in n8n this is typically automatic if binary exists.)
- **Credentials:** Google Drive OAuth2 must be connected.
- **I/O connections:** Output to **✅ Prepare Final Data**.
- **Failure modes:** OAuth expired; insufficient permissions; folder URL invalid; file too large.

#### Node: ✅ Prepare Final Data
- **Type / role:** Code node – creates a clean record for logging.
- **Configuration choices:**
  - Builds a Drive file URL:
    - uses `driveResult.webViewLink` or fallback `https://drive.google.com/file/d/<id>/view`
  - Outputs final metadata: topic, intention, brand, theme, scripts, caption, URLs, status, timestamp.
- **I/O connections:** Output to **📝 Log to Sheets**.
- **Edge cases:** Drive node response lacking `id`/`webViewLink` (rare, but possible with permission issues).

#### Node: 📝 Log to Sheets
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) – append a row for tracking.
- **Configuration choices:**
  - Operation: `append`
  - Document: configured via URL placeholder `YOUR_GOOGLE_SHEETS_URL`
  - Sheet name: `Sheet1`
  - Columns: auto-map input data
- **Credentials:** Google Sheets OAuth2.
- **I/O connections:** Output to **🔄 Continue Loop1**.
- **Failure modes:** OAuth expired; sheet not found; header mismatch with auto-mapping; API quota.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual entrypoint | — | ⚙️ Workflow Configuration | # Generate social videos with AI avatars using VEED and Claude… (How it works + Setup steps) |
| ⚙️ Workflow Configuration | Set | Defines topic, keys, settings, optional custom inputs | When clicking 'Execute workflow' | 📋 Generate Video Tasks | ### Configuration: Set your topic, API keys, and video settings. Supports custom avatar descriptions and scripts. |
| 📋 Generate Video Tasks | Code | Creates N per-video task items with angles/hooks | ⚙️ Workflow Configuration | 🔄 Loop Through Videos | ### AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention. |
| 🔄 Loop Through Videos | Split in Batches | Iterates tasks and enables looping | 📋 Generate Video Tasks; 🔄 Continue Loop1 | 🧠 Build Claude Prompt | ### AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention. |
| 🧠 Build Claude Prompt | Code | Builds dynamic Claude prompt + expected JSON schema | 🔄 Loop Through Videos | 🤖 Claude: Generate Content | ### AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention. |
| 🤖 Claude: Generate Content | HTTP Request | Calls Anthropic Messages API | 🧠 Build Claude Prompt | 📋 Parse Claude Response | ### AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention. |
| 📋 Parse Claude Response | Code | Parses Claude JSON; applies fallbacks; produces normalized fields | 🤖 Claude: Generate Content | 🎨 Generate Avatar (OpenAI) | ### AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention. |
| 🎨 Generate Avatar (OpenAI) | HTTP Request | Generates portrait avatar image (base64) | 📋 Parse Claude Response | 📸 Extract Image Data | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| 📸 Extract Image Data | Code | Converts base64 image to n8n binary | 🎨 Generate Avatar (OpenAI) | ☁️ Upload Image | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| ☁️ Upload Image | HTTP Request | Uploads avatar image to tmpfiles | 📸 Extract Image Data | 💾 Store Image URL | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| 💾 Store Image URL | Code | Converts tmpfiles URL to direct download URL | ☁️ Upload Image | 🔊 Generate Audio (TTS) | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| 🔊 Generate Audio (TTS) | HTTP Request | OpenAI TTS → MP3 binary | 💾 Store Image URL | 🎵 Convert Audio | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| 🎵 Convert Audio | Code | Normalizes audio binary field name/type | 🔊 Generate Audio (TTS) | ☁️ Upload Audio | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| ☁️ Upload Audio | HTTP Request | Uploads MP3 to tmpfiles | 🎵 Convert Audio | 📦 Prepare VEED Request | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| 📦 Prepare VEED Request | Code | Assembles VEED payload (public URLs + settings) | ☁️ Upload Audio | 🎬 Generate Video (VEED) | ### Video Rendering: VEED creates the final lip-synced talking-head video from your avatar and audio. |
| 🎬 Generate Video (VEED) | VEED | Renders lip-synced video | 📦 Prepare VEED Request | 📁 Prepare Upload | ### Video Rendering: VEED creates the final lip-synced talking-head video from your avatar and audio. |
| 📁 Prepare Upload | Code | Extracts video URL; builds filename; stores debug payload | 🎬 Generate Video (VEED) | ⬇️ Download Video | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |
| ⬇️ Download Video | HTTP Request | Downloads rendered MP4 as binary | 📁 Prepare Upload | 📤 Upload to Drive | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |
| 📤 Upload to Drive | Google Drive | Uploads MP4 to Drive folder | ⬇️ Download Video | ✅ Prepare Final Data | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |
| ✅ Prepare Final Data | Code | Creates final log record including Drive URL | 📤 Upload to Drive | 📝 Log to Sheets | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |
| 📝 Log to Sheets | Google Sheets | Appends metadata row to Sheets | ✅ Prepare Final Data | 🔄 Continue Loop1 | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |
| 🔄 Continue Loop1 | NoOp | Loops back to SplitInBatches | 📝 Log to Sheets | 🔄 Loop Through Videos | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |
| Sticky Note | Sticky Note | Documentation / context | — | — | # Generate social videos with AI avatars using VEED and Claude… (How it works + Setup steps) |
| Sticky Note Configuration | Sticky Note | Block label | — | — | ### Configuration: Set your topic, API keys, and video settings. Supports custom avatar descriptions and scripts. |
| Sticky Note AI Content | Sticky Note | Block label | — | — | ### AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention. |
| Sticky Note Avatar Audio | Sticky Note | Block label | — | — | ### Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech. |
| Sticky Note Video Rendering | Sticky Note | Block label | — | — | ### Video Rendering: VEED creates the final lip-synced talking-head video from your avatar and audio. |
| Sticky Note Storage Logging | Sticky Note | Block label | — | — | ### Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n (keep it inactive until credentials are set).

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking 'Execute workflow'`

3) **Add configuration Set node**
   - Node: **Set**
   - Name: `⚙️ Workflow Configuration`
   - Mode: **Raw JSON**
   - Paste a config object with at least:
     - `topic`, `intention`, `brand_name`, `target_audience`, `trending_hashtags`
     - `num_videos`
     - `anthropic_api_key`, `openai_api_key`
     - `video_resolution`, `video_aspect_ratio`
     - `custom_avatar_description`, `custom_script` (empty strings allowed)
   - Connect: Manual Trigger → Configuration

4) **Add “Generate Video Tasks” Code node**
   - Node: **Code**
   - Name: `📋 Generate Video Tasks`
   - Logic: create `num_videos` items; add `video_index`, rotate `content_angle` and `hook_style`.
   - Connect: Configuration → Generate Video Tasks

5) **Add Split In Batches loop controller**
   - Node: **Split In Batches**
   - Name: `🔄 Loop Through Videos`
   - Set batch size to **1** (recommended to avoid cross-item reference issues).
   - Connect: Generate Video Tasks → Split In Batches
   - Use the **“Items” output** (second output) for processing.

6) **Add “Build Claude Prompt” Code node**
   - Node: **Code**
   - Name: `🧠 Build Claude Prompt`
   - Implement:
     - Detect non-empty `custom_avatar_description` / `custom_script`
     - Create instruction prompt with intention + hook style
     - Require JSON-only output with a strict schema
     - Output `claude_prompt`, `has_custom_avatar`, `has_custom_script`
   - Connect: Split In Batches (Items output) → Build Claude Prompt

7) **Add Anthropic HTTP Request**
   - Node: **HTTP Request**
   - Name: `🤖 Claude: Generate Content`
   - Method: **POST**
   - URL: `https://api.anthropic.com/v1/messages`
   - Send headers:
     - `x-api-key: {{$json.anthropic_api_key}}`
     - `anthropic-version: 2023-06-01`
     - `Content-Type: application/json`
   - Send JSON body:
     - model `claude-sonnet-4-20250514`
     - max_tokens `2000`
     - messages: user content = prompt string
   - Connect: Build Claude Prompt → Claude Request

   **Credential note:** This workflow stores API keys in data, not in n8n credentials. You can improve security by using n8n credentials or environment variables and referencing them in expressions.

8) **Add “Parse Claude Response” Code node**
   - Node: **Code**
   - Name: `📋 Parse Claude Response`
   - Implement:
     - Extract `content[0].text`
     - Strip ``` fences
     - Parse JSON with try/catch fallback
     - Produce:
       - `script_image`, `script_audio`, `caption`, `content_theme`, `avatar_gender`
       - carry over keys/settings needed downstream
   - Connect: Claude Request → Parse Claude Response

9) **Add OpenAI Image generation HTTP Request**
   - Node: **HTTP Request**
   - Name: `🎨 Generate Avatar (OpenAI)`
   - Method: POST
   - URL: `https://api.openai.com/v1/images/generations`
   - Headers:
     - `Authorization: Bearer {{$json.openai_api_key}}`
     - `Content-Type: application/json`
   - JSON body:
     - `model: gpt-image-1`
     - `prompt: {{$json.script_image}}`
     - `size: 1024x1536`, `quality: high`, `n: 1`
   - Connect: Parse Claude Response → Generate Avatar

10) **Add image base64 → binary converter Code node**
   - Node: **Code**
   - Name: `📸 Extract Image Data`
   - Implement:
     - Read `data[0].b64_json`
     - Throw if missing
     - Output binary field `image_data` (png)
   - Connect: Generate Avatar → Extract Image Data

11) **Upload image to tmpfiles**
   - Node: **HTTP Request**
   - Name: `☁️ Upload Image`
   - Method: POST
   - URL: `https://tmpfiles.org/api/v1/upload`
   - Body content type: **multipart/form-data**
   - Add form field `file` = binary property `image_data`
   - Connect: Extract Image Data → Upload Image

12) **Store public image URL Code node**
   - Node: **Code**
   - Name: `💾 Store Image URL`
   - Implement:
     - Convert tmpfiles `data.url` into direct `/dl/...` URL
     - Output `public_image_url`
   - Connect: Upload Image → Store Image URL

13) **Generate TTS audio (OpenAI)**
   - Node: **HTTP Request**
   - Name: `🔊 Generate Audio (TTS)`
   - Method: POST
   - URL: `https://api.openai.com/v1/audio/speech`
   - Headers:
     - `Authorization: Bearer {{$json.openai_api_key}}`
     - `Content-Type: application/json`
   - Body:
     - `model: tts-1-hd`, `voice: nova`, `response_format: mp3`
     - `input: {{$json.script_audio}}`
   - Response: **File**, output binary property name `audio`
   - Connect: Store Image URL → Generate Audio (TTS)

14) **Normalize audio binary**
   - Node: **Code**
   - Name: `🎵 Convert Audio`
   - Implement:
     - Copy `binary.audio` → `binary.audio_mp3` with `audio/mpeg` + filename
   - Connect: Generate Audio → Convert Audio

15) **Upload audio to tmpfiles**
   - Node: **HTTP Request**
   - Name: `☁️ Upload Audio`
   - Method: POST
   - URL: `https://tmpfiles.org/api/v1/upload`
   - multipart/form-data, field `file` from binary `audio_mp3`
   - Connect: Convert Audio → Upload Audio

16) **Prepare VEED request Code node**
   - Node: **Code**
   - Name: `📦 Prepare VEED Request`
   - Implement:
     - Convert tmpfiles audio URL to `/dl/...`
     - Set:
       - `image_url` from stored image URL
       - `audio_url` from audio upload
       - `resolution`, `aspect_ratio`
       - keep metadata (topic, caption, scripts, etc.)
   - Connect: Upload Audio → Prepare VEED Request

17) **Add VEED rendering node**
   - Node: **VEED** (community/integration node: `n8n-nodes-veed.veed`)
   - Name: `🎬 Generate Video (VEED)`
   - Set:
     - Audio URL = `{{$json.audio_url}}`
     - Image URL = `{{$json.image_url}}`
     - Resolution = `{{$json.resolution}}`
     - Aspect Ratio = `{{$json.aspect_ratio}}`
   - Configure **VEED/FAL.ai credential** as required by the node.
   - Connect: Prepare VEED Request → VEED node

18) **Prepare filename + extract output URL**
   - Node: **Code**
   - Name: `📁 Prepare Upload`
   - Implement:
     - Detect final `video_url` from possible VEED response paths
     - Build `file_name` including topic/index/date
   - Connect: VEED node → Prepare Upload

19) **Download the rendered video**
   - Node: **HTTP Request**
   - Name: `⬇️ Download Video`
   - Method: GET
   - URL: `{{$json.video_url}}`
   - Response: **File**
   - Connect: Prepare Upload → Download Video

20) **Upload to Google Drive**
   - Node: **Google Drive**
   - Name: `📤 Upload to Drive`
   - Operation: Upload
   - File name expression: `{{$('📁 Prepare Upload').item.json.file_name}}`
   - Folder: set to your target folder (URL or ID)
   - Credentials: Google Drive OAuth2
   - Connect: Download Video → Upload to Drive

21) **Prepare final record**
   - Node: **Code**
   - Name: `✅ Prepare Final Data`
   - Implement:
     - Create `video_url` as Drive webViewLink (or fallback)
     - Include caption, scripts, asset URLs, created timestamp
   - Connect: Drive → Prepare Final Data

22) **Log to Google Sheets**
   - Node: **Google Sheets**
   - Name: `📝 Log to Sheets`
   - Operation: Append
   - Document: your Google Sheet
   - Sheet: `Sheet1` (or your sheet)
   - Mapping: auto-map (or explicitly map columns)
   - Credentials: Google Sheets OAuth2
   - Connect: Prepare Final Data → Log to Sheets

23) **Loop continuation**
   - Node: **NoOp**
   - Name: `🔄 Continue Loop1`
   - Connect: Log to Sheets → Continue Loop → back into **🔄 Loop Through Videos** (the “continue” input).

24) **(Optional) Add sticky notes** to label sections, mirroring the provided notes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate social videos with AI avatars using VEED and Claude” + overview of steps 1–8 and setup checklist (API keys, VEED credential, Google OAuth2, update Drive/Sheets URLs, customize topic/settings). | Sticky note in workflow canvas (applies to entire workflow). |
| “Configuration: Set your topic, API keys, and video settings. Supports custom avatar descriptions and scripts.” | Sticky note over configuration block. |
| “AI Content Generation: Claude creates the script, image prompt, and social caption based on your topic and intention.” | Sticky note over Claude block. |
| “Avatar & Audio: OpenAI generates a photorealistic avatar image and converts the script to natural speech.” | Sticky note over OpenAI image + TTS block. |
| “Video Rendering: VEED creates the final lip-synced talking-head video from your avatar and audio.” | Sticky note over VEED node. |
| “Storage & Logging: Downloads the video, uploads to Google Drive, and logs all metadata to Google Sheets.” | Sticky note over Drive/Sheets block. |