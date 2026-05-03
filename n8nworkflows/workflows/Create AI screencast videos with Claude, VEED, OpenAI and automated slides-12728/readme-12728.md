Create AI screencast videos with Claude, VEED, OpenAI and automated slides

https://n8nworkflows.xyz/workflows/create-ai-screencast-videos-with-claude--veed--openai-and-automated-slides-12728


# Create AI screencast videos with Claude, VEED, OpenAI and automated slides

## 1. Workflow Overview

**Workflow name:** Create AI screencast videos with VEED and automated slides  
**Purpose:** Automatically generate a “screencast-style” video combining:
- A **talking-head avatar** (image + voiceover → lip-synced video via VEED)
- A set of **AI-generated presentation slides** (images generated from Claude-authored prompts via FAL)
- A final **composited landscape video** (slides as main area + avatar as picture-in-picture) rendered by **Creatomate**, then uploaded to **Google Drive** and logged to **Google Sheets**.

**Typical use cases**
- Short thought-leadership or product explainers (25–40 seconds)
- Social content with consistent slide aesthetics and a presenter overlay
- Rapid generation of branded content variants (topic, intention, audience, style)

### 1.1 Input & Configuration
Manual start → central configuration for topic, branding, timing, style, and API keys.

### 1.2 AI Planning (Claude)
Build a structured prompt → call Anthropic → parse JSON response into:
- voiceover script (or use custom script)
- slide prompts (16:9)
- optional avatar prompt (unless custom avatar URL provided)
- caption + hashtags
Also calculates slide count/duration estimates and picks voice ID.

### 1.3 Parallel Production
Splits into two flows:
- **Avatar flow:** avatar image (OpenAI or custom URL) → ElevenLabs TTS → VEED talking head video
- **Slides flow:** expand slides → generate slide images via FAL Flux Pro → aggregate/sort into ordered list

### 1.4 Composition & Render (Creatomate)
Merge avatar + slides → build Creatomate element tree (optionally with background) → start render → poll until succeeded.

### 1.5 Output
Download final MP4 → upload to Google Drive → log metadata to Google Sheets → return final summary object.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Workflow Configuration
**Overview:** Entry point and a single “configuration object” that drives all downstream behavior (topic, style, timing, API keys, voice/avatar/script overrides).  
**Nodes involved:**  
- When clicking 'Execute workflow'
- ⚙️ Workflow Configuration

#### Node: When clicking 'Execute workflow'
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – starts execution manually.
- **Configuration choices:** No parameters.
- **Outputs:** One empty item to the next node.
- **Failure modes:** None (except workflow not executed).

#### Node: ⚙️ Workflow Configuration
- **Type / role:** Code node – returns one item containing all runtime settings.
- **Key settings produced:**
  - Content: `topic`, `intention`, `brand_name`, `target_audience`, `trending_hashtags`
  - Slide style: `slide_style` (predefined styles)
  - Timing: `seconds_per_slide` (used later to estimate slide count)
  - Optional background: `background` (empty / gradient array / image URL)
  - API keys: `anthropic_api_key`, `openai_api_key`, `elevenlabs_api_key`, `creatomate_api_key`, `fal_api_key`
  - Voice: `voice_selection`
  - Avatar overrides: `custom_avatar_description`, `custom_avatar_image_url`
  - Script override: `custom_script`
- **Input/Output:** Trigger → Build Claude Prompt.
- **Edge cases / failures:**
  - Missing/invalid API keys will later surface as 401/403 errors in HTTP nodes.
  - `background` type must match expected patterns:
    - `""` for none
    - URL string for image background
    - array of hex colors for gradient background
- **Version requirements:** Code node v2.

---

### Block 2 — AI Planning with Claude (prompt → content → parsed plan)
**Overview:** Generates a detailed Claude prompt based on config, requests JSON-only output from Anthropic, then parses and normalizes it into a consistent internal structure (including voice ID selection and duration calculations).  
**Nodes involved:**  
- 🧠 Build Claude Prompt  
- 🤖 Claude: Generate Content  
- 📋 Parse Claude Response

#### Node: 🧠 Build Claude Prompt
- **Type / role:** Code node – constructs an instruction prompt for Claude, including a strict JSON schema response format.
- **Configuration choices (interpreted):**
  - Selects one of 5 slide design systems (`slideStyles`) using `config.slide_style`.
  - If a **custom script** is provided, estimates slide count from word count and `seconds_per_slide`, constrained 3–8.
  - If **custom avatar URL** is provided, skips avatar prompt generation requirements.
  - Builds a JSON “response format” template; instructs Claude to output **only valid JSON**.
- **Key variables added to output:**
  - `claude_prompt`, `slide_style_config`, `estimated_slides`
  - flags: `has_custom_avatar`, `has_custom_avatar_url`, `has_custom_script`
- **Input/Output:** Configuration → Claude HTTP request.
- **Failure modes:** Expression/JS errors if config fields are missing or non-strings (rare, since config node controls shape).

#### Node: 🤖 Claude: Generate Content
- **Type / role:** HTTP Request – calls Anthropic Messages API.
- **Configuration choices:**
  - POST `https://api.anthropic.com/v1/messages`
  - Model: `claude-sonnet-4-20250514`
  - `max_tokens: 3000`
  - Sends one user message whose `content` is the serialized prompt text.
  - Headers:
    - `x-api-key: {{$json.anthropic_api_key}}`
    - `anthropic-version: 2023-06-01`
    - `Content-Type: application/json`
- **Input/Output:** Build Claude Prompt → Parse Claude Response.
- **Failure modes:**
  - 401/403 if key invalid
  - 429 rate limits
  - 400 if payload malformed (less likely)
  - Timeout/network issues

#### Node: 📋 Parse Claude Response
- **Type / role:** Code node – extracts Claude’s text, strips possible code fences, JSON-parses, and applies robust fallbacks.
- **Normalization performed:**
  - If JSON parse fails, creates fallback:
    - default `audio_script`
    - default 5 slides with safe prompts
    - default `avatar_prompt`, caption, theme, language
  - Determines ElevenLabs `voice_id` from `voice_selection` map:
    - cristina, enrique, susie, jeff, custom
  - Forces `language` to `es` if Spanish voice chosen.
  - Chooses avatar strategy:
    - If `has_custom_avatar_url` → no avatar prompt needed downstream
    - Else if `has_custom_avatar` → builds a portrait prompt from `custom_avatar_description`
    - Else uses Claude `avatar_prompt` or fallback prompt
  - Determines final script:
    - `custom_script` wins; else Claude `audio_script`; else fallback
  - Calculates:
    - `script_word_count`
    - `estimated_duration` ≈ ceil(words/2.5) + 2 seconds buffer
    - `num_slides` from Claude slides length (or generated defaults)
    - `duration_per_slide = estimated_duration / num_slides`
  - Determines `background_type`:
    - `"none"` / `"url"` / `"gradient"` and sets `background_value`
- **Outputs:** A single normalized item containing:
  - `script_audio`, `slides[]`, `avatar_prompt`, `voice_id`, timing, keys, etc.
- **Failure modes:**
  - If Claude returns unexpected structure, JSON parse fallback is used.
  - If `slides` empty and duration calc yields edge values, default slides are generated.
- **Note:** This node references data from `🧠 Build Claude Prompt` by name using `$('🧠 Build Claude Prompt').item.json`.

---

### Block 3 — Parallelization Router
**Overview:** Duplicates the normalized data into two items and routes each into either the avatar pipeline or the slides pipeline.  
**Nodes involved:**  
- 🔀 Split into Flows  
- 🔀 Avatar Flow?  
- 🔀 Slides Flow?

#### Node: 🔀 Split into Flows
- **Type / role:** Code node – emits two items:
  - same JSON + `flow: "avatar"`
  - same JSON + `flow: "slides"`
- **Input/Output:** Parse Claude Response → both IF routers.
- **Failure modes:** None typical.

#### Node: 🔀 Avatar Flow?
- **Type / role:** IF node – checks `{{$json.flow}} == "avatar"`.
- **True output:** goes to “Has Custom Avatar URL?”
- **False output:** unused (implicit discard).

#### Node: 🔀 Slides Flow?
- **Type / role:** IF node – checks `{{$json.flow}} == "slides"`.
- **True output:** goes to “Expand Slides”
- **False output:** unused.

---

### Block 4A — Avatar + Audio + Talking Head (VEED)
**Overview:** Produces the presenter video: choose avatar source (custom URL or OpenAI image), generate voiceover with ElevenLabs, upload assets to temporary hosting, then create a lip-synced video via VEED.  
**Nodes involved:**  
- 🖼️ Has Custom Avatar URL?  
- 📸 Use Custom Avatar URL  
- 🎨 Generate Avatar (OpenAI)  
- 📸 Extract Avatar Image  
- ☁️ Upload Avatar Image  
- 💾 Store Avatar URL  
- 🔊 Generate Audio (ElevenLabs)  
- 🎵 Convert Audio  
- ☁️ Upload Audio  
- 💾 Store Audio URL  
- 🎬 Generate Talking Head (VEED)  
- 📹 Extract VEED Video URL

#### Node: 🖼️ Has Custom Avatar URL?
- **Type / role:** IF – checks boolean `has_custom_avatar_url == true`.
- **True path:** Use Custom Avatar URL (skip generation/upload)
- **False path:** Generate Avatar (OpenAI)

#### Node: 📸 Use Custom Avatar URL
- **Type / role:** Code – sets `avatar_image_url = custom_avatar_image_url`.
- **Used when:** config provided a direct public URL.
- **Failure modes:** If URL is not publicly accessible, VEED may fail later.

#### Node: 🎨 Generate Avatar (OpenAI)
- **Type / role:** HTTP Request – OpenAI Images generation.
- **Request:**
  - POST `https://api.openai.com/v1/images/generations`
  - model `gpt-image-1`
  - `prompt = $json.avatar_prompt`
  - `size: 1024x1536`, `quality: high`
  - Authorization header: `Bearer {{$json.openai_api_key}}`
- **Output:** base64 image in `data[0].b64_json` (expected).
- **Failure modes:** 401/429, safety refusals, schema changes, no `b64_json` returned.

#### Node: 📸 Extract Avatar Image
- **Type / role:** Code – converts OpenAI base64 result to **binary** field `avatar_image` (PNG).
- **Important detail:** Throws hard error if no base64 data found.
- **Output:** JSON (previous data) + binary `{ avatar_image: ... }`
- **Failure modes:** missing `b64_json` → workflow stops unless error handling added.

#### Node: ☁️ Upload Avatar Image
- **Type / role:** HTTP Request – uploads binary file to `tmpfiles.org`.
- **Request:** multipart/form-data with `file` = `avatar_image`.
- **Output:** JSON containing `data.url`.
- **Failure modes:** tmpfiles downtime, response shape changes.

#### Node: 💾 Store Avatar URL
- **Type / role:** Code – converts tmpfiles “view” URL to a direct download form:
  - `http://tmpfiles.org/{id}/{name}` → `https://tmpfiles.org/dl/{id}/{name}`
- **Output:** sets `avatar_image_url`.
- **Failure modes:** regex mismatch if tmpfiles response format changes.

#### Node: 🔊 Generate Audio (ElevenLabs)
- **Type / role:** HTTP Request – text-to-speech.
- **Request:**
  - POST `https://api.elevenlabs.io/v1/text-to-speech/{{$json.voice_id}}`
  - JSON body includes `text: $json.script_audio`, model `eleven_multilingual_v2`
  - Response format set to **file**, stored in binary property `audio`
  - Headers: `xi-api-key`, `Accept: audio/mpeg`
- **Failure modes:** 401/429, unsupported voice ID, content rejection, long text limits.

#### Node: 🎵 Convert Audio
- **Type / role:** Code – normalizes the binary field to `audio_mp3` and restores JSON from the correct upstream branch.
- **Notable logic:** Tries to fetch prior JSON from nodes in priority:
  1) `💾 Store Avatar URL`
  2) `📸 Use Custom Avatar URL`
  3) `🖼️ Has Custom Avatar URL?`
- **Output:** binary `audio_mp3` with filename `voiceover.mp3`.
- **Failure modes:**
  - If node names change, the `$('node name')` lookups fail.
  - If ElevenLabs node output property name changes, `item.binary.audio` might be missing.

#### Node: ☁️ Upload Audio
- **Type / role:** HTTP Request – uploads `audio_mp3` to tmpfiles.
- **Failure modes:** same as avatar upload.

#### Node: 💾 Store Audio URL
- **Type / role:** Code – converts tmpfiles URL to `https://tmpfiles.org/dl/...` and sets `audio_url`.
- **Failure modes:** same regex dependency.

#### Node: 🎬 Generate Talking Head (VEED)
- **Type / role:** VEED node (`n8n-nodes-veed.veed`) – generates lip-synced avatar video.
- **Key parameters:**
  - `audioUrl: {{$json.audio_url}}`
  - `imageUrl: {{$json.avatar_image_url}}`
  - `resolution: {{$json.video_resolution}}` (note config says VEED supports 720p)
  - `aspectRatio: 9:16` (vertical)
  - Timeout option: 60 seconds
- **Failure modes:**
  - Invalid/expired asset URLs (tmpfiles is temporary)
  - VEED API errors, timeouts (60s may be short for some renders)
  - Resolution constraints (config comment: VEED only supports 720p)

#### Node: 📹 Extract VEED Video URL
- **Type / role:** Code – extracts a video URL from multiple possible response shapes:
  - `video.url`, `output.video_url`, `videoUrl`, or `url`
- **Output:** `avatar_video_url` + `asset_type: "avatar_video"`
- **Failure modes:** VEED response schema not matching any checked fields → `avatar_video_url` becomes empty, breaking downstream composition.

---

### Block 4B — Slide Image Generation (FAL Flux Pro)
**Overview:** Turns Claude slide prompts into actual 1920×1080 slide images, in parallel per slide; then aggregates and sorts them for composition.  
**Nodes involved:**  
- 📑 Expand Slides  
- 🖼️ Generate Slide (FAL)  
- 📸 Extract Slide URL  
- 📚 Aggregate Slides  
- 📊 Format Slides

#### Node: 📑 Expand Slides
- **Type / role:** Code – maps `slides[]` into multiple items so each can be generated independently.
- **Output fields:** `current_slide`, `slide_index`, plus the full base data copied along.
- **Failure modes:** `slides` missing or not an array → generates 0 items; downstream will have no slides.

#### Node: 🖼️ Generate Slide (FAL)
- **Type / role:** HTTP Request – calls FAL Flux Pro.
- **Request:**
  - POST `https://fal.run/fal-ai/flux-pro/v1.1`
  - prompt: `current_slide.image_prompt`
  - image size: 1920×1080
  - `num_images: 1`
  - Auth: `Authorization: Key {{$json.fal_api_key}}`
- **Batching option:** enabled with batchSize 1 (effectively per-item; still allows n8n batching behavior).
- **Failure modes:** invalid key, content safety blocks, rate limits, slow generation.

#### Node: 📸 Extract Slide URL
- **Type / role:** Code – pairs each FAL response to the corresponding expanded slide item, extracts an image URL.
- **Extraction logic:** checks `images[0].url`, `image.url`, `output[0]`.
- **Also sets:** `duration_seconds` = `current_slide.duration_seconds` OR `seconds_per_slide` OR 9.
- **Failure modes:**
  - If batching/concurrency changes item ordering, index-based pairing can mismatch slides.
  - No URL found → throws error and stops execution.

#### Node: 📚 Aggregate Slides
- **Type / role:** Aggregate node – aggregates all incoming slide items into one field `all_slides_data`.
- **Mode:** “aggregate all item data”.
- **Failure modes:** If no slide items arrive, output may be empty/undefined, causing format step issues.

#### Node: 📊 Format Slides
- **Type / role:** Code – sorts aggregated slides by `slide_index` and outputs `slides_with_urls`.
- **Output:** `{ slides_with_urls: [...], asset_type: "all_slides" }`
- **Failure modes:** missing indices → sort fallback uses 0; may reorder incorrectly if indices absent.

---

### Block 5 — Merge + Creatomate Composition + Render Polling
**Overview:** Combines avatar video URL and ordered slides into a Creatomate “elements” timeline, triggers a render, then polls until status is `succeeded`.  
**Nodes involved:**  
- 🔗 Merge Avatar + Slides  
- 📦 Prepare Creatomate Request  
- 🎬 Render Video (Creatomate)  
- 📊 Extract Render Info  
- ⏳ Wait for Render  
- 🔍 Check Render Status  
- 📋 Process Status  
- ✅ Render Done?

#### Node: 🔗 Merge Avatar + Slides
- **Type / role:** Merge node – combines streams.
- **Mode:** `combine` with `combineAll` (creates a combined set containing both branch outputs).
- **Inputs:**  
  - Input 0: avatar branch item (`avatar_video_url`)
  - Input 1: slides branch item (`slides_with_urls`)
- **Failure modes:**
  - If either branch doesn’t produce an item, the merge/combineAll behavior can lead to missing data downstream.

#### Node: 📦 Prepare Creatomate Request
- **Type / role:** Code – builds the full Creatomate RenderScript `elements`.
- **Key behaviors:**
  - Detects which combined items contain `avatar_video_url` and which contain `slides_with_urls`.
  - Uses `estimated_duration` to compute `durationPerSlide`.
  - Builds `slideElements` (track 1 images with optional fade transitions).
  - Optional background support:
    - `background_type: gradient` → adds a `shape` element with `fill_color` gradient stops
    - `background_type: url` → adds a full-canvas background image
  - Layout logic:
    - **With background:** slides in a rounded composition (74% width) + avatar video (20% width) with margins
    - **No background:** full-bleed slides (78%) + avatar (22%) without rounded clipping
- **Output:** `creatomate_elements`, `total_duration`, `num_slides`, etc.
- **Failure modes:**
  - If `avatarVideoUrl` is empty, Creatomate may error or render without presenter.
  - If slide URLs are empty/unreachable, render fails.
  - Gradient stop calculation can divide by zero if only one color is provided (the code uses `(min(colors.length,5)-1)`; with length 1, this becomes 0). This is a real edge case if user sets background gradient array with a single color.

#### Node: 🎬 Render Video (Creatomate)
- **Type / role:** HTTP Request – starts a render job.
- **Request:**
  - POST `https://api.creatomate.com/v2/renders`
  - Output: mp4, 1920×1080, 60 fps, H.264 high profile, CRF 18
  - Body includes `elements` as JSON string of `creatomate_elements`
  - Auth: `Bearer {{$json.creatomate_api_key}}`
- **Failure modes:** invalid key, invalid element schema, rate limits, render queue delays.

#### Node: 📊 Extract Render Info
- **Type / role:** Code – normalizes Creatomate response shape (array or object) into:
  - `render_id`, `render_url`, `render_status`
- **Failure modes:** unexpected response format leads to empty `render_id`, breaking polling.

#### Node: ⏳ Wait for Render
- **Type / role:** Wait node – pauses for 30 seconds before checking status.
- **Failure modes:** None typical; but increases total run time.

#### Node: 🔍 Check Render Status
- **Type / role:** HTTP Request – fetch render by ID.
- **Request:** GET `https://api.creatomate.com/v2/renders/{{$json.render_id}}`
- **Auth header:** reads key from `$('📦 Prepare Creatomate Request').item.json.creatomate_api_key`
- **Failure modes:**
  - If node name changes, credential lookup breaks.
  - Missing/empty `render_id`.
  - 404 if render ID invalid/expired.

#### Node: 📋 Process Status
- **Type / role:** Code – updates:
  - `render_status`
  - `final_video_url` (from `statusResponse.url`)
- **Failure modes:** If `url` not present until succeeded, may remain empty.

#### Node: ✅ Render Done?
- **Type / role:** IF – checks `render_status == "succeeded"`.
- **True:** Download final video  
- **False:** loops back to Wait (poll every 30s)

---

### Block 6 — Output (Download → Drive → Sheets)
**Overview:** Retrieves the final MP4, stores it in Google Drive, then logs a record into Google Sheets and returns a final summary payload.  
**Nodes involved:**  
- ⬇️ Download Final Video  
- 📤 Upload to Drive  
- ✅ Prepare Final Data  
- 📝 Log to Sheets

#### Node: ⬇️ Download Final Video
- **Type / role:** HTTP Request – downloads binary MP4 from `final_video_url`.
- **Response format:** file (binary).
- **Failure modes:** URL expired, 403/404, large file download issues.

#### Node: 📤 Upload to Drive
- **Type / role:** Google Drive node – uploads the downloaded binary to a specified folder.
- **Key configuration:**
  - File name: `topic` sanitized to underscores, truncated to 30 chars, plus timestamp.
  - Drive: “My Drive”
  - Folder: set via `YOUR_GOOGLE_DRIVE_FOLDER_URL`
- **Credentials required:** Google Drive OAuth2 in n8n.
- **Failure modes:** OAuth not configured, folder URL invalid, insufficient permissions, file too large.

#### Node: ✅ Prepare Final Data
- **Type / role:** Code – creates final structured output including Drive link and metadata (caption, script, theme, etc.).
- **Drive URL logic:** uses `webViewLink` or constructs `https://drive.google.com/file/d/{id}/view`.
- **Failure modes:** missing `id` in Drive response (rare).

#### Node: 📝 Log to Sheets
- **Type / role:** Google Sheets node – appends a row to `Sheet1`.
- **Configuration:**
  - Operation: append
  - Mapping: auto-map input data
  - Document: `YOUR_GOOGLE_SHEETS_URL`
- **Credentials required:** Google Sheets OAuth2 in n8n.
- **Failure modes:** permissions, wrong sheet name, auto-mapping mismatch, Sheets API quotas.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Entry point (manual run) | — | ⚙️ Workflow Configuration | ## How it works … (full note content applies to overall workflow) |
| ⚙️ Workflow Configuration | Code | Central config object (topic, keys, style, overrides) | When clicking 'Execute workflow' | 🧠 Build Claude Prompt | **1. Configuration** — Set your topic, API keys, voice, and slide style preferences |
| 🧠 Build Claude Prompt | Code | Builds strict JSON-output prompt for Claude | ⚙️ Workflow Configuration | 🤖 Claude: Generate Content | **2. AI script and slides** — Claude generates voiceover script, slide prompts, avatar description, and social caption |
| 🤖 Claude: Generate Content | HTTP Request | Calls Anthropic to generate script/slide prompts/caption | 🧠 Build Claude Prompt | 📋 Parse Claude Response | **2. AI script and slides** — Claude generates voiceover script, slide prompts, avatar description, and social caption |
| 📋 Parse Claude Response | Code | Parses/validates Claude JSON; computes durations; selects voice | 🤖 Claude: Generate Content | 🔀 Split into Flows | **2. AI script and slides** — Claude generates voiceover script, slide prompts, avatar description, and social caption |
| 🔀 Split into Flows | Code | Duplicates item into avatar + slides flows | 📋 Parse Claude Response | 🔀 Avatar Flow?, 🔀 Slides Flow? | ## How it works … (parallel avatar + slides) |
| 🔀 Avatar Flow? | IF | Route only avatar item | 🔀 Split into Flows | 🖼️ Has Custom Avatar URL? | ### Avatar and audio generation … |
| 🖼️ Has Custom Avatar URL? | IF | Decide custom avatar URL vs OpenAI generation | 🔀 Avatar Flow? | 📸 Use Custom Avatar URL, 🎨 Generate Avatar (OpenAI) | **3. Avatar generation** — Creates or uses custom avatar image via OpenAI, uploads to temporary storage |
| 📸 Use Custom Avatar URL | Code | Set avatar_image_url from provided URL | 🖼️ Has Custom Avatar URL? (true) | 🔊 Generate Audio (ElevenLabs) | **3. Avatar generation** — Creates or uses custom avatar image via OpenAI, uploads to temporary storage |
| 🎨 Generate Avatar (OpenAI) | HTTP Request | Generate avatar portrait image (base64) | 🖼️ Has Custom Avatar URL? (false) | 📸 Extract Avatar Image | **3. Avatar generation** — Creates or uses custom avatar image via OpenAI, uploads to temporary storage |
| 📸 Extract Avatar Image | Code | Convert OpenAI base64 to binary PNG | 🎨 Generate Avatar (OpenAI) | ☁️ Upload Avatar Image | **3. Avatar generation** — Creates or uses custom avatar image via OpenAI, uploads to temporary storage |
| ☁️ Upload Avatar Image | HTTP Request | Upload avatar PNG to tmpfiles | 📸 Extract Avatar Image | 💾 Store Avatar URL | **3. Avatar generation** — Creates or uses custom avatar image via OpenAI, uploads to temporary storage |
| 💾 Store Avatar URL | Code | Convert tmpfiles URL to direct download; store avatar_image_url | ☁️ Upload Avatar Image | 🔊 Generate Audio (ElevenLabs) | **3. Avatar generation** — Creates or uses custom avatar image via OpenAI, uploads to temporary storage |
| 🔊 Generate Audio (ElevenLabs) | HTTP Request | TTS voiceover → binary mp3 | 💾 Store Avatar URL / 📸 Use Custom Avatar URL | 🎵 Convert Audio | ### Avatar and audio generation … |
| 🎵 Convert Audio | Code | Rename binary prop; restore JSON from correct branch | 🔊 Generate Audio (ElevenLabs) | ☁️ Upload Audio | ### Avatar and audio generation … |
| ☁️ Upload Audio | HTTP Request | Upload mp3 to tmpfiles | 🎵 Convert Audio | 💾 Store Audio URL | ### Avatar and audio generation … |
| 💾 Store Audio URL | Code | Convert tmpfiles URL; store audio_url | ☁️ Upload Audio | 🎬 Generate Talking Head (VEED) | ### Avatar and audio generation … |
| 🎬 Generate Talking Head (VEED) | VEED | Lip-sync avatar image + audio into vertical video | 💾 Store Audio URL | 📹 Extract VEED Video URL | **4. Talking head** — VEED creates lip-synced video from avatar and audio |
| 📹 Extract VEED Video URL | Code | Normalize VEED response into avatar_video_url | 🎬 Generate Talking Head (VEED) | 🔗 Merge Avatar + Slides | **4. Talking head** — VEED creates lip-synced video from avatar and audio |
| 🔀 Slides Flow? | IF | Route only slides item | 🔀 Split into Flows | 📑 Expand Slides | ### Slide image generation … |
| 📑 Expand Slides | Code | Fan-out slides[] into per-slide items | 🔀 Slides Flow? | 🖼️ Generate Slide (FAL) | ### Slide image generation … |
| 🖼️ Generate Slide (FAL) | HTTP Request | Generate each slide image via FAL Flux Pro | 📑 Expand Slides | 📸 Extract Slide URL | ### Slide image generation … |
| 📸 Extract Slide URL | Code | Extract URL from FAL response; keep slide_index | 🖼️ Generate Slide (FAL) | 📚 Aggregate Slides | ### Slide image generation … |
| 📚 Aggregate Slides | Aggregate | Combine all slide items into one payload | 📸 Extract Slide URL | 📊 Format Slides | ### Slide image generation … |
| 📊 Format Slides | Code | Sort slides; output slides_with_urls | 📚 Aggregate Slides | 🔗 Merge Avatar + Slides | ### Slide image generation … |
| 🔗 Merge Avatar + Slides | Merge | Combine avatar_video_url + slides_with_urls | 📹 Extract VEED Video URL, 📊 Format Slides | 📦 Prepare Creatomate Request | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| 📦 Prepare Creatomate Request | Code | Build Creatomate elements timeline and layout | 🔗 Merge Avatar + Slides | 🎬 Render Video (Creatomate) | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| 🎬 Render Video (Creatomate) | HTTP Request | Start Creatomate render | 📦 Prepare Creatomate Request | 📊 Extract Render Info | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| 📊 Extract Render Info | Code | Capture render_id/status/url | 🎬 Render Video (Creatomate) | ⏳ Wait for Render | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| ⏳ Wait for Render | Wait | Delay between status polls (30s) | 📊 Extract Render Info, ✅ Render Done? (false) | 🔍 Check Render Status | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| 🔍 Check Render Status | HTTP Request | Query Creatomate render by ID | ⏳ Wait for Render | 📋 Process Status | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| 📋 Process Status | Code | Update render_status and final_video_url | 🔍 Check Render Status | ✅ Render Done? | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| ✅ Render Done? | IF | Loop until `succeeded` | 📋 Process Status | ⬇️ Download Final Video (true), ⏳ Wait for Render (false) | **5. Video composition** — Creatomate merges slides as background with talking head overlay, then polls until render completes |
| ⬇️ Download Final Video | HTTP Request | Download final MP4 binary | ✅ Render Done? (true) | 📤 Upload to Drive | **6. Output** — Downloads final video, uploads to Google Drive, and logs results to Google Sheets |
| 📤 Upload to Drive | Google Drive | Store MP4 in Drive folder | ⬇️ Download Final Video | ✅ Prepare Final Data | **6. Output** — Downloads final video, uploads to Google Drive, and logs results to Google Sheets |
| ✅ Prepare Final Data | Code | Assemble final response object (Drive URL + metadata) | 📤 Upload to Drive | 📝 Log to Sheets | **6. Output** — Downloads final video, uploads to Google Drive, and logs results to Google Sheets |
| 📝 Log to Sheets | Google Sheets | Append run metadata to a spreadsheet | ✅ Prepare Final Data | — | **6. Output** — Downloads final video, uploads to Google Drive, and logs results to Google Sheets |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: “Create AI screencast videos with VEED and automated slides”
- Keep it inactive until credentials are set.

2) **Add Trigger**
- Add **Manual Trigger** node: `When clicking 'Execute workflow'`.

3) **Add Configuration (Code)**
- Add **Code** node: `⚙️ Workflow Configuration`
- Return a single JSON object containing:
  - content fields (`topic`, `intention`, `brand_name`, `target_audience`, `trending_hashtags`)
  - `slide_style`, `seconds_per_slide`, `background`
  - API keys: `anthropic_api_key`, `openai_api_key`, `elevenlabs_api_key`, `creatomate_api_key`, `fal_api_key`
  - `voice_selection`
  - optional avatar/script overrides (`custom_avatar_description`, `custom_avatar_image_url`, `custom_script`)
- Connect: Manual Trigger → Configuration.

4) **Build Claude Prompt (Code)**
- Add **Code** node: `🧠 Build Claude Prompt`
- Implement:
  - slide style lookup table
  - dynamic instructions based on presence of custom script/avatar
  - strict JSON-only output requirement
- Output fields must include at least: `claude_prompt` plus config passthrough.
- Connect: Configuration → Build Claude Prompt.

5) **Call Anthropic (HTTP Request)**
- Add **HTTP Request** node: `🤖 Claude: Generate Content`
- Method: POST
- URL: `https://api.anthropic.com/v1/messages`
- Headers:
  - `x-api-key` = `{{$json.anthropic_api_key}}`
  - `anthropic-version` = `2023-06-01`
  - `Content-Type` = `application/json`
- Body (JSON):
  - model: `claude-sonnet-4-20250514`
  - max_tokens: 3000
  - messages: user content set to the generated prompt
- Connect: Build Claude Prompt → Claude node.

6) **Parse/Normalize Claude Output (Code)**
- Add **Code** node: `📋 Parse Claude Response`
- Responsibilities:
  - extract `content[0].text`
  - strip ``` fences if present
  - JSON.parse with fallback defaults
  - select ElevenLabs `voice_id`
  - compute `estimated_duration`, `num_slides`, `duration_per_slide`
  - normalize `slides[]`, `script_audio`, `avatar_prompt`
  - interpret `background` into `background_type/background_value`
- Connect: Claude → Parse.

7) **Split into parallel flows (Code + IF routers)**
- Add **Code** node: `🔀 Split into Flows` → output two items with `flow: avatar` and `flow: slides`.
- Add **IF** node: `🔀 Avatar Flow?` condition `{{$json.flow}} equals "avatar"`.
- Add **IF** node: `🔀 Slides Flow?` condition `{{$json.flow}} equals "slides"`.
- Connect: Parse → Split.
- Connect: Split → Avatar Flow? and Split → Slides Flow? (same output to both routers).

---

### Avatar branch (talking head)

8) **Custom avatar URL decision**
- Add **IF** node: `🖼️ Has Custom Avatar URL?` condition `{{$json.has_custom_avatar_url}} equals true`.
- Connect: Avatar Flow? (true) → Has Custom Avatar URL?

9) **If TRUE: use provided URL**
- Add **Code** node: `📸 Use Custom Avatar URL` setting `avatar_image_url = custom_avatar_image_url`.
- Connect: Has Custom Avatar URL? (true) → Use Custom Avatar URL.

10) **If FALSE: generate avatar via OpenAI**
- Add **HTTP Request** node: `🎨 Generate Avatar (OpenAI)`
  - POST `https://api.openai.com/v1/images/generations`
  - Header `Authorization: Bearer {{$json.openai_api_key}}`
  - JSON: model `gpt-image-1`, prompt `{{$json.avatar_prompt}}`, size `1024x1536`, quality `high`
- Add **Code** node: `📸 Extract Avatar Image` → convert `b64_json` to binary PNG field `avatar_image`.
- Add **HTTP Request** node: `☁️ Upload Avatar Image` (multipart upload to `https://tmpfiles.org/api/v1/upload`, file field from `avatar_image`).
- Add **Code** node: `💾 Store Avatar URL` → convert tmpfiles URL to `/dl/` and set `avatar_image_url`.
- Connect: Has Custom Avatar URL? (false) → Generate Avatar → Extract Avatar Image → Upload Avatar Image → Store Avatar URL.

11) **Generate voiceover (ElevenLabs)**
- Add **HTTP Request** node: `🔊 Generate Audio (ElevenLabs)`
  - POST `https://api.elevenlabs.io/v1/text-to-speech/{{$json.voice_id}}`
  - Headers: `xi-api-key: {{$json.elevenlabs_api_key}}`, `Accept: audio/mpeg`, `Content-Type: application/json`
  - Body includes `text: {{$json.script_audio}}` and voice settings
  - Response: **File**; output binary property name `audio`
- Connect:
  - Use Custom Avatar URL → Generate Audio
  - Store Avatar URL → Generate Audio

12) **Normalize and upload audio**
- Add **Code** node: `🎵 Convert Audio`:
  - move binary `audio` to `audio_mp3` with filename `voiceover.mp3`
  - restore JSON from the appropriate upstream node (be careful if you rename nodes)
- Add **HTTP Request** node: `☁️ Upload Audio` to tmpfiles (multipart; file from `audio_mp3`)
- Add **Code** node: `💾 Store Audio URL` convert to `/dl/`, store `audio_url`
- Connect: Generate Audio → Convert Audio → Upload Audio → Store Audio URL.

13) **Generate VEED talking head**
- Add **VEED** node: `🎬 Generate Talking Head (VEED)`
  - audioUrl: `{{$json.audio_url}}`
  - imageUrl: `{{$json.avatar_image_url}}`
  - resolution: `{{$json.video_resolution}}` (set config to `720p`)
  - aspect ratio: `9:16`
  - timeout: 60s (increase if needed)
- Add **Code** node: `📹 Extract VEED Video URL` to set `avatar_video_url`.
- Connect: Store Audio URL → VEED → Extract VEED Video URL.

---

### Slides branch

14) **Expand slides into items**
- Add **Code** node: `📑 Expand Slides` mapping `slides[]` to multiple items with `current_slide` and `slide_index`.
- Connect: Slides Flow? (true) → Expand Slides.

15) **Generate slide images via FAL**
- Add **HTTP Request** node: `🖼️ Generate Slide (FAL)`
  - POST `https://fal.run/fal-ai/flux-pro/v1.1`
  - Header: `Authorization: Key {{$json.fal_api_key}}`
  - JSON body: prompt `{{$json.current_slide.image_prompt}}`, size 1920×1080, num_images 1
- Add **Code** node: `📸 Extract Slide URL` to extract the resulting image URL and keep ordering fields.
- Connect: Expand Slides → Generate Slide → Extract Slide URL.

16) **Aggregate and sort**
- Add **Aggregate** node: `📚 Aggregate Slides` to aggregate all item data into `all_slides_data`.
- Add **Code** node: `📊 Format Slides` to sort by `slide_index` and output `slides_with_urls`.
- Connect: Extract Slide URL → Aggregate Slides → Format Slides.

---

### Merge + Creatomate render + polling

17) **Merge branches**
- Add **Merge** node: `🔗 Merge Avatar + Slides`
  - Mode: Combine
  - Combine by: Combine All
- Connect:
  - Extract VEED Video URL → Merge (input 0)
  - Format Slides → Merge (input 1)

18) **Prepare Creatomate elements**
- Add **Code** node: `📦 Prepare Creatomate Request`
  - Build `creatomate_elements` using slide images and avatar video URL
  - Respect `background_type/background_value` for optional background
  - Use `estimated_duration` to set durations
- Connect: Merge → Prepare Creatomate Request.

19) **Start Creatomate render**
- Add **HTTP Request** node: `🎬 Render Video (Creatomate)`
  - POST `https://api.creatomate.com/v2/renders`
  - Header: `Authorization: Bearer {{$json.creatomate_api_key}}`
  - Body includes encoding settings and `elements: {{$json.creatomate_elements}}`
- Add **Code** node: `📊 Extract Render Info` to set `render_id`, `render_status`, `render_url`.
- Connect: Prepare Creatomate Request → Render Video → Extract Render Info.

20) **Polling loop**
- Add **Wait** node: `⏳ Wait for Render` amount 30 seconds.
- Add **HTTP Request** node: `🔍 Check Render Status`
  - GET `https://api.creatomate.com/v2/renders/{{$json.render_id}}`
  - Auth header must use your API key (either from current item or referenced node)
- Add **Code** node: `📋 Process Status` to set `render_status` and `final_video_url`.
- Add **IF** node: `✅ Render Done?` condition `{{$json.render_status}} equals "succeeded"`.
- Connect:
  - Extract Render Info → Wait → Check Render Status → Process Status → Render Done?
  - Render Done? (false) → Wait (loop)

---

### Output

21) **Download final MP4**
- Add **HTTP Request** node: `⬇️ Download Final Video`
  - URL: `{{$json.final_video_url}}`
  - Response: File
- Connect: Render Done? (true) → Download Final Video.

22) **Upload to Google Drive**
- Add **Google Drive** node: `📤 Upload to Drive`
  - Operation: Upload
  - Binary property: the downloaded file
  - Folder: set your target folder
  - Filename expression based on topic + timestamp
- **Credentials:** configure Google Drive OAuth2 in n8n and select it in node.
- Connect: Download Final Video → Upload to Drive.

23) **Prepare final metadata**
- Add **Code** node: `✅ Prepare Final Data` to output a summary JSON including Drive link, script, caption, etc.
- Connect: Upload to Drive → Prepare Final Data.

24) **Log to Google Sheets**
- Add **Google Sheets** node: `📝 Log to Sheets`
  - Operation: Append
  - Spreadsheet: your document
  - Sheet name: `Sheet1` (or adjust)
  - Mapping: auto-map from `✅ Prepare Final Data`
- **Credentials:** Google Sheets OAuth2 in n8n.
- Connect: Prepare Final Data → Log to Sheets.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow generates professional screencast-style videos with a talking head avatar and AI-generated slides; it runs two parallel processes (avatar path + slides path), then composites in Creatomate, uploads to Drive, and logs to Sheets. | Sticky note: “How it works” |
| Setup: add API keys in the Configuration node (Anthropic, OpenAI, ElevenLabs, FAL.ai, Creatomate). | Sticky note: “How it works” |
| Setup: configure n8n credentials for Google Drive OAuth2 and Google Sheets OAuth2; update folder URL and Sheets URL in output nodes. | Sticky note: “How it works” |
| Avatar/audio branch: ElevenLabs generates speech, VEED lip-syncs to produce a vertical presenter video. | Sticky note: “Avatar and audio generation” |
| Slides branch: FAL Flux Pro generates 5–7 high-quality 16:9 slide images from Claude prompts. | Sticky note: “Slide image generation” |
| VEED resolution constraint mentioned in config: “VEED only supports 720p”. | Configuration code comments |
| Temporary hosting uses tmpfiles.org; links may expire, affecting VEED/Creatomate steps. Consider replacing with durable storage (e.g., S3/GCS) for production. | Workflow design implication |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.