Create AI product images and marketing videos with NanoBanana Pro, Veo 3.1 and Blotato

https://n8nworkflows.xyz/workflows/create-ai-product-images-and-marketing-videos-with-nanobanana-pro--veo-3-1-and-blotato-12462


# Create AI product images and marketing videos with NanoBanana Pro, Veo 3.1 and Blotato

## 1. Workflow Overview

**Workflow name (in JSON):** `Generate product images with NanoBanana Pro to Veo videos and Blotato - vide 2 ok`  
**User-provided title:** *Create AI product images and marketing videos with NanoBanana Pro, Veo 3.1 and Blotato*

This workflow automates a full pipeline to: (1) ingest 3 reference images, (2) describe them with OpenAI Vision, (3) generate an image prompt, (4) create a new “hero” image via **fal.ai NanoBanana Pro**, (5) create a **2×3 contact sheet** (6 keyframes), (6) crop the 6 frames, (7) generate multiple short videos with **fal.ai Veo 3.1** using adjacent frames, (8) merge clips into one final MP4, (9) write all URLs back to **Google Sheets**, and (10) upload/publish the final video with **Blotato** (YouTube in this version).

### 1.1 Block A — Input reception (Form upload) + validation
Receives exactly 3 images from an n8n Form Trigger, validates required files, and normalizes binary field names.

### 1.2 Block B — Reference images storage + vision analysis + prompt engineering
Uploads images to Google Drive, uses OpenAI Vision to describe them, aggregates descriptions/URLs, and uses an LLM agent (with structured output parser) to generate a NanoBanana image prompt.

### 1.3 Block C — NanoBanana Pro image creation + logging to Google Sheets
Calls NanoBanana Pro (edit endpoint) to generate an image from the prompt and reference image URLs, waits, downloads result, then appends a new row to Google Sheets.

### 1.4 Block D — Contact sheet prompting (6 frames in one image) + update sheet
Takes the NanoBanana output image, generates a 2×3 contact sheet (6 keyframes) via NanoBanana Pro, waits, downloads, and updates the Google Sheet row with the contact sheet URL and status.

### 1.5 Block E — Scheduled processing: crop contact sheet into 6 images + store + update sheet
On a schedule, finds new rows in Google Sheets, downloads the contact sheet, computes image dimensions, crops into 6 frames, uploads each crop to Google Drive, and writes each frame URL back to Google Sheets (`new_image_1` … `new_image_6`).

### 1.6 Block F — Veo video generation (adjacent frames) + update sheet
Combines pairs of consecutive frames, generates 5 short videos with Veo 3.1 (1→2, 2→3, 3→4, 4→5, 5→6), waits after each, and updates Google Sheets (`video_1`…`video_5`).

### 1.7 Block G — Merge videos + finalize + publish
Merges the 5 clips into a single MP4 via fal ffmpeg API, waits, updates Google Sheets (status + Final video URL), uploads to Blotato, then creates a YouTube post.

---

## 2. Block-by-Block Analysis

### Block A — Input reception (Form upload) + validation

**Overview:** Collects 3 images via an n8n hosted form, checks presence of all three binary fields, and prepares consistent binary naming for later code and nodes.

**Nodes involved:**
- `Form Trigger (3 images)`
- `Validate inputs`
- `Error Response - Missing Files`
- `Normalize binary names`
- `Split images`

#### Node: Form Trigger (3 images)
- **Type/role:** `n8n-nodes-base.formTrigger` — public form endpoint to receive uploaded files.
- **Config highlights:**
  - Title: “Upload 3 Images for Analysis”
  - Button label: “Analyze Images”
  - Fields: `image1`, `image2`, `image3` (required, single file) + `notes` (optional)
  - Accepted types: `.jpg, .jpeg, .png, .gif, .bmp, .webp`
- **Outputs:** One item with `$binary.image1`, `$binary.image2`, `$binary.image3`.
- **Failure/edge cases:** user uploads fewer than 3 images, wrong file types, large files hitting instance limits.

#### Node: Validate inputs
- **Type/role:** `n8n-nodes-base.if` — gate to enforce presence of three binary fields.
- **Conditions:** checks `exists` for `$binary.image1`, `$binary.image2`, `$binary.image3`.
- **True output:** to `Normalize binary names`
- **False output:** to `Error Response - Missing Files`
- **Failure/edge cases:** binary field names differ (e.g., if form changes), causing false negatives.

#### Node: Error Response - Missing Files
- **Type/role:** `n8n-nodes-base.set` — creates an error message payload.
- **Config:** sets `json.error = "Please upload 3 images (image1, image2, image3)."`
- **Outputs:** not connected further in this workflow (acts as terminal response payload).
- **Edge cases:** if you want the form to display error, you typically add an HTTP Response node in webhook-based flows; Form Trigger behavior may vary by n8n version.

#### Node: Normalize binary names
- **Type/role:** `n8n-nodes-base.set` — maps binary inputs into JSON object fields.
- **Config:** sets:
  - `json.image0 = $binary.image1`
  - `json.image1 = $binary.image2`
  - `json.image2 = $binary.image3`
  - `includeOtherFields = true`
- **Important note:** Despite the name, it does **not** rename `$binary.*` fields; it stores them in JSON. Downstream code expects `$binary.image1..3`, so this node does not actually help the later `Split images` node (see below).
- **Edge cases:** mismatch between expected binary fields and what’s produced next.

#### Node: Split images
- **Type/role:** `n8n-nodes-base.code` — splits a single item containing 3 binaries into 3 separate items.
- **Key logic:** expects `item.binary.image1`, `item.binary.image2`, `item.binary.image3`. It throws if missing.
- **Output:** 3 items:
  - item1: `json.imageNumber=1`, `binary.image = original image1`
  - item2: `json.imageNumber=2`, `binary.image = original image2`
  - item3: `json.imageNumber=3`, `binary.image = original image3`
- **Connections:** feeds both `OpenAI Vision – Image 1` and `Upload file`.
- **Likely bug:** because `Normalize binary names` didn’t rename `$binary.image1..3`, this code works only if the incoming item actually still has `$binary.image1..3` (it does, since `includeOtherFields=true` keeps original fields). However, the code checks `item.binary.image1` but then outputs `item.binary.image1` etc—so it is fine **only if** original binary names are exactly `image1,image2,image3`.
- **Failure/edge cases:** any changes to form field names break this node; memory usage for large binaries.

---

### Block B — Reference images storage + vision analysis + prompt engineering

**Overview:** Uploads each of the 3 images to Google Drive, gets shareable links, generates detailed descriptions using OpenAI Vision, aggregates everything, then builds a structured NanoBanana prompt using an LLM agent and parser.

**Nodes involved:**
- `Upload file`
- `OpenAI Vision – Image 1`
- `Merge`
- `Aggregate descriptions`
- `LLM: OpenAI Chat`
- `LLM: Structured Output Parser`
- `Generate Image Prompt`

#### Node: Upload file
- **Type/role:** `n8n-nodes-base.googleDrive` — uploads each split binary image to Drive.
- **Config highlights:**
  - Input binary property: `image`
  - File name: `{{$binary.image.fileName}}`
  - Drive ID: placeholder
  - Folder ID: placeholder/blank
- **Credentials:** `Google Drive account` (OAuth2)
- **Output:** includes `webContentLink` / `webViewLink` depending on Drive settings.
- **Failure/edge cases:** OAuth expiry, insufficient permissions, missing Drive ID/folder ID, Drive returning links that require permissions (not publicly accessible by fal/OpenAI if not shared properly).

#### Node: OpenAI Vision – Image 1
- **Type/role:** `@n8n/n8n-nodes-langchain.openAi` — image analysis on each item.
- **Config:**
  - Operation: `analyze`
  - Resource: `image`
  - Input type: `base64`
  - Binary property: `image`
  - Prompt text: “Describe the image in detail…”
  - Model: `gpt-4o`
- **Credentials:** `OpenAi account`
- **Output:** a LangChain-style response object; in later aggregation they expect it under `json["0"].content[]`.
- **Edge cases:** OpenAI content policy refusal, rate limits, model access, large images causing request failure.

#### Node: Merge
- **Type/role:** `n8n-nodes-base.merge` — `combineByPosition`
- **Purpose:** merges the two parallel branches:
  - Branch A: Drive upload results
  - Branch B: OpenAI vision results
- **Result structure:** each merged item contains both payloads; the OpenAI result is available in the merged JSON under key `"0"` (as referenced in `Aggregate descriptions`).
- **Edge cases:** if one branch returns fewer items than the other, combine-by-position misaligns.

#### Node: Aggregate descriptions
- **Type/role:** `n8n-nodes-base.code` — consolidates 3 merged items into a single JSON payload.
- **Key outputs:**
  - `image1Description`, `image2Description`, `image3Description`
  - `image1Url`, `image2Url`, `image3Url`
  - `imageUrls` (array of valid links)
  - `allDescriptions` (formatted multi-image text)
- **Notable assumptions:**
  - OpenAI data is at `it.json["0"]` with `content[]` objects containing `.text`
  - Drive URL uses `webContentLink` or `webViewLink`
- **Edge cases:** Drive link absent; OpenAI response shape differs; if Merge order changes, parsing breaks.

#### Node: LLM: OpenAI Chat
- **Type/role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model backend for the agent.
- **Model:** `gpt-4.1-mini`
- **Connection:** provides `ai_languageModel` to `Generate Image Prompt`.
- **Edge cases:** model availability; rate limits.

#### Node: LLM: Structured Output Parser
- **Type/role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema output.
- **Schema example:** `{ "image_prompt": "string" }`
- **Connection:** provides `ai_outputParser` to `Generate Image Prompt`.
- **Edge cases:** if LLM output is not valid JSON, parsing fails; add retry/fallback if needed.

#### Node: Generate Image Prompt
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` — generates a final `image_prompt` string for NanoBanana.
- **Inputs:**
  - Text includes `{{$json.allDescriptions}}` from `Aggregate descriptions`
  - System message: strict template; only vary hand positioning and product wear/placement, keep everything else identical.
- **Output:** `json.output.image_prompt` (used by NanoBanana request)
- **Edge cases:** LLM deviates from constraints; parser failures; prompt too long.

---

### Block C — NanoBanana Pro image creation + logging to Google Sheets

**Overview:** Submits prompt + reference image URLs to NanoBanana Pro edit endpoint, waits for processing, downloads response, then appends a tracking row in Google Sheets with status `nano_done`.

**Nodes involved:**
- `NanoBanana: Create Image`
- `Wait for Image Edit`
- `Download Edited Image`
- `Append row in sheet`

#### Node: NanoBanana: Create Image
- **Type/role:** `n8n-nodes-base.httpRequest` — call fal NanoBanana edit queue endpoint.
- **Endpoint:** `https://queue.fal.run/fal-ai/nano-banana-pro/edit`
- **Body highlights:**
  - `prompt`: sanitized from `($json.output.image_prompt || '')` (newlines removed, quotes escaped)
  - `image_urls`: `$('Aggregate descriptions').item.json.imageUrls` (stringified)
  - `resolution`: `1K`, `aspect_ratio`: `16:9`, `output_format`: `png`
- **Headers:** `Authorization` placeholder (fal key), `Content-Type: application/json`
- **Output:** fal queue response with `response_url` (polled later).
- **Failure/edge cases:** invalid fal key; fal queue delays; `image_urls` not publicly accessible (Drive permissions).

#### Node: Wait for Image Edit
- **Type/role:** `n8n-nodes-base.wait` — simple delay.
- **Config:** 2 minutes.
- **Purpose:** gives fal time to finish before downloading `response_url`.
- **Edge cases:** fixed wait may be too short; best practice is polling until status complete.

#### Node: Download Edited Image
- **Type/role:** `n8n-nodes-base.httpRequest`
- **URL:** `{{$json.response_url}}`
- **Output:** JSON containing `images[0].url` (used by Sheets append).
- **Edge cases:** `response_url` not ready yet, 404/202; transient fal errors.

#### Node: Append row in sheet
- **Type/role:** `n8n-nodes-base.googleSheets` — append new record.
- **Operation:** `append`
- **Writes columns:**
  - `status = "nano_done"`
  - `image_1..3` from aggregated Drive URLs
  - `description_all`
  - `image_nanobanana = $json.images[0].url`
- **Matching columns:** shown as `["id"]` in config, but append doesn’t typically require matching; ensure sheet schema aligns.
- **Failure/edge cases:** sheet tab/document placeholders not set; column names mismatch; permission errors.

---

### Block D — Contact sheet prompting (6 frames) + update sheet

**Overview:** Manually-triggered block to generate a 2×3 contact sheet (6 frames) from the NanoBanana image, then update the existing sheet row with `image_contactsheet` and status.

**Nodes involved:**
- `When clicking ‘Execute workflow’`
- `Get image nanobanana`
- `Edit Fields : contactSheetPrompt`
- `NanoBanana: Contact Sheet`
- `Wait for Image Edit1`
- `Download Edited Image1`
- `Update database`

#### Node: When clicking ‘Execute workflow’
- **Type/role:** `n8n-nodes-base.manualTrigger` — dev/test trigger.
- **Purpose:** starts contact sheet generation manually.
- **Edge cases:** none; not used in scheduled automation.

#### Node: Get image nanobanana
- **Type/role:** `n8n-nodes-base.googleSheets` — reads rows (operation not explicitly set; default is typically “read/getMany”).
- **Config:** only `documentId` set; `sheetName` left blank in JSON (must be set in UI).
- **Output expected:** includes `image_nanobanana` field.
- **Edge cases:** without filters, may read many rows; downstream nodes assume a single “current” item.

#### Node: Edit Fields : contactSheetPrompt
- **Type/role:** `n8n-nodes-base.set`
- **Config:** sets an extensive `contactSheetPrompt` requiring:
  - exact identity preservation
  - 2×3 contact sheet (6 frames)
  - fixed wardrobe/environment/lighting continuity
  - Fuji Velvia + hard flash aesthetic
- **Output:** passes prompt to NanoBanana contact sheet call.

#### Node: NanoBanana: Contact Sheet
- **Type/role:** `n8n-nodes-base.httpRequest`
- **Endpoint:** `https://queue.fal.run/fal-ai/nano-banana-pro/edit`
- **Body highlights:**
  - `prompt`: `contactSheetPrompt`
  - `image_urls`: built from `Get image nanobanana`. Includes logic to accept:
    - array
    - JSON string array
    - single string -> wrapped array
    - else empty array
  - `aspect_ratio`: `3:2` (fits 2×3 sheet nicely)
- **Headers:** includes a **hard-coded fal key** in JSON (`Authorization: key ...`). This is a security risk and should be moved to credentials/variables.
- **Edge cases:** empty `image_urls` leads to fal error; key leakage; response not ready.

#### Node: Wait for Image Edit1
- **Type/role:** wait 2 minutes.

#### Node: Download Edited Image1
- **Type/role:** HTTP GET to `{{$json.response_url}}`
- **Output:** expects `images[0].url` which is the contact sheet image URL.

#### Node: Update database
- **Type/role:** Google Sheets `update`
- **Matching column:** `image_nanobanana`
- **Writes:**
  - `status = "ContactSheet_done"`
  - `image_nanobanana` preserved
  - `image_contactsheet = $json.images[0].url`
  - `row_number = 0` (note: might be unintended; schema lists row_number as read-only; setting it may be ignored)
- **Edge cases:** if multiple rows share same `image_nanobanana`, updates may affect multiple; matching requires exact text match.

---

### Block E — Scheduled: crop contact sheet into 6 frames + store + update sheet

**Overview:** On schedule, reads Google Sheets for “new image”, downloads the contact sheet, computes dimensions, crops 6 frames (2 rows × 3 columns), uploads each crop to Drive, and writes URLs back into the sheet.

**Nodes involved:**
- `Schedule Trigger`
- `Search new image`
- `Set Image URL`
- `Download Image`
- `Edit Image` (information)
- `Crop Top Left`, `Crop Top Center`, `Crop Top Right`
- `Crop Bottom Left`, `Crop Bottom Center`, `Crop Bottom Right`
- `Upload to Google Drive`, `Upload to Google Drive1..5`
- `Update url image_top_left..image_bottom_right`
- `Merge1..Merge5`

#### Node: Schedule Trigger
- **Type/role:** `n8n-nodes-base.scheduleTrigger`
- **Config:** interval rule exists but is empty in JSON; must be configured in UI (cron/interval).
- **Output:** triggers sheet scan.
- **Edge cases:** misconfigured interval = never runs.

#### Node: Search new image
- **Type/role:** Google Sheets read/search node (operation not specified in JSON).
- **Intent (by name):** find rows needing cropping/video generation.
- **Critical requirement:** must output `image_contactsheet` per row.
- **Edge cases:** missing filter may process already-done rows; should filter by `status` (e.g., `ContactSheet_done` but not yet “done”).

#### Node: Set Image URL
- **Type/role:** Set node
- **Config:** `image_url = $json.image_contactsheet`
- **Purpose:** normalizes field used by downloader.

#### Node: Download Image
- **Type/role:** HTTP Request download file
- **URL:** `{{$json.image_url}}`
- **Response format:** `file` (binary)
- **Edge cases:** if `image_contactsheet` is a gated URL; 403; large image; timeouts.

#### Node: Edit Image
- **Type/role:** `n8n-nodes-base.editImage` operation `information`
- **Purpose:** reads binary image metadata, especially `size.width` and `size.height` used by crop expressions.
- **Edge cases:** unsupported file formats; missing binary input.

#### Nodes: Crop Top/Bottom (6 nodes)
- **Type/role:** `n8n-nodes-base.editImage` operation `crop`
- **Cropping math:**
  - `width = floor(size.width / 3)`
  - `height = floor(size.height / 2)`
  - X offsets: `0`, `floor(width)`, `floor(2*width)`
  - Y offsets: `0` or `floor(size.height/2)`
- **Outputs:** each emits cropped binary image.
- **Edge cases:** if image is not exactly divisible by 3×2, last column/row may lose pixels; ensure consistent contact sheet layout.

#### Nodes: Upload to Google Drive (6 nodes)
- **Type/role:** upload each cropped frame to Drive.
- **Naming:** `image_top_left_YYYY-MM-DD_HH-MM-SS.png` etc (timestamp-based).
- **Output:** link fields used for Sheets updates (`webContentLink`).
- **Edge cases:** folderId blank in several nodes; ensure correct Drive/folder; `webContentLink` availability depends on Drive permissions.

#### Nodes: Update url image_… (6 nodes)
- **Type/role:** Google Sheets `update`
- **Matching column:** `image_contactsheet`
- **Writes:**
  - `new_image_1..new_image_6 = $json.webContentLink` depending on frame
  - also writes `image_contactsheet = $('Set Image URL').first().json.image_url` (reinforces matching key)
- **Edge cases:** if Drive returns `webViewLink` instead of `webContentLink`, your Veo step may break; also if multiple rows share same contact sheet URL, updates collide.

#### Nodes: Merge1..Merge5
- **Type/role:** Merge `combineByPosition` to pair consecutive frames for Veo:
  - Merge1: (new_image_1 + new_image_2) → Veo video1
  - Merge2: (new_image_2 + new_image_3) → Veo video2
  - Merge3: (new_image_3 + new_image_4) → Veo video3
  - Merge4: (new_image_4 + new_image_5) → Veo video4
  - Merge5: (new_image_5 + new_image_6) → Veo video5
- **Edge cases:** race conditions if sheet updates are asynchronous; combine-by-position requires both inputs available and aligned.

---

### Block F — Veo 3.1 generation + update Google Sheets

**Overview:** Calls Veo for each consecutive pair of frames to produce short videos, then writes video URLs to Google Sheets.

**Nodes involved:**
- `Veo Generation`, `Veo Generation1`, `Veo Generation2`, `Veo Generation3`, `Veo Generation4`
- `Wait`, `Wait1`, `Wait2`, `Wait3`, `Wait4`
- `Update video 1..5`
- `Merge6`

#### Nodes: Veo Generation (1..4)
- **Type/role:** HTTP Request to fal Veo 3.1 endpoint.
- **Endpoint:** `https://fal.run/fal-ai/veo3.1/first-last-frame-to-video`
- **Common parameters:**
  - `first_frame_url` / `last_frame_url` from merged new images
  - `duration: "4s"`, `resolution: "720p"`, `aspect_ratio: "auto"`
  - `generate_audio: false` (even though one prompt mentions audio)
  - `auto_fix: true`
  - Timeout: `600000` (10 minutes)
- **Headers:** Authorization placeholder (fal key)
- **Edge cases:** URLs not publicly accessible; Veo processing longer than 3 minutes; fal model errors; 429; prompt mismatch.

#### Nodes: Wait (0..4)
- **Type/role:** Wait nodes (3 minutes each) after each Veo request.
- **Purpose:** allow async completion; however Veo endpoint used (`fal.run`) often returns a final JSON already; if it’s queued, this wait may not be sufficient.
- **Edge cases:** fixed wait too short; better polling on `response_url` pattern if returned.

#### Nodes: Update video 1..5
- **Type/role:** Google Sheets update
- **Matching column:** `image_contactsheet`
- **Writes:** `video_n = $json.video.url`, plus `image_contactsheet` for matching.
- **Edge cases:** `video.url` absent (API error shape), updates hitting wrong rows if not unique.

#### Node: Merge6
- **Type/role:** Merge node combine-by-position with `numberInputs: 5`
- **Purpose:** collect outputs of Update video 1..5 into one item for merge step.
- **Important:** Update nodes output sheet operation result, not necessarily the row data. The merge step later expects fields `video_1..video_5`, but those fields are not guaranteed to exist on the merged output unless the update node returns them. This is a common integration pitfall.
- **Edge cases:** You may need an explicit “Read row” after updates to gather all video URLs reliably.

---

### Block G — Merge videos + finalize + publish

**Overview:** Merges 5 video clips into one MP4, updates sheet with final URL + status, uploads to Blotato, and creates a YouTube post.

**Nodes involved:**
- `Merge 3 Videos`
- `Wait: Merge Process`
- `Update URL Final video`
- `Upload Video to BLOTATO`
- `Youtube`

#### Node: Merge 3 Videos
- **Type/role:** HTTP Request to fal ffmpeg merge API.
- **Endpoint:** `https://fal.run/fal-ai/ffmpeg-api/merge-videos`
- **Body:** `video_urls` array referencing:
  - `{{$json.video_1}} ... {{$json.video_5}}`
  - resolution: `landscape_16_9`, output format mp4
- **Risk:** upstream merged item may not contain `video_1..video_5` fields (see Merge6 note).
- **Edge cases:** wrong/missing URLs; ffmpeg API errors; authorization issues.

#### Node: Wait: Merge Process
- **Type/role:** Wait 3 minutes, then proceed.
- **Edge cases:** merge may take longer.

#### Node: Update URL Final video
- **Type/role:** Google Sheets update
- **Matching column:** `image_contactsheet`
- **Writes:**
  - `status = "done"`
  - `Final video = $json.video.url`
  - `image_contactsheet = $('Set Image URL').item.json.image_url`
- **Edge cases:** uses `$('Set Image URL').item` (not `.first()`), so if multiple items are in flight, item mapping must match; otherwise writes wrong row.

#### Node: Upload Video to BLOTATO
- **Type/role:** Blotato node (`@blotato/n8n-nodes-blotato.blotato`) resource `media`.
- **Config:** `mediaUrl = {{$json['Final video']}}`
- **Output:** typically returns `url` for uploaded media.
- **Edge cases:** Blotato auth, unsupported URL, platform upload limits.

#### Node: Youtube
- **Type/role:** Blotato post creation for YouTube.
- **Config:**
  - platform: `youtube`
  - account: id `8047` (“DR FIRASS (Dr. Firas)”)
  - text/title: “My new vidéo”
  - media URLs: `{{$json.url}}` (output from previous Blotato media upload)
  - privacy: `private`
  - notify subscribers: false
- **Edge cases:** account disconnected, YouTube quota/policy, video processing delays.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Documentation / setup info |  |  | # Setup & Configuration Guide / Required services: NanoBanana Pro, Veo 3.1, BLOTATO (includes https://blotato.com/?ref=firas) |
| Sticky Note1 | stickyNote | Section header |  |  | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Sticky Note2 | stickyNote | Section header |  |  | # Step 2 – Contact Sheet Prompting Technique |
| Sticky Note3 | stickyNote | Documentation links |  |  | Notion docs: https://automatisation.notion.site/Generate-product-images-with-NanoBanana-Pro-to-Veo-videos-and-Blotato-2dc3d6550fd980dc81adea10a5df6b28?source=copy_link / Copy Google Sheet: https://docs.google.com/spreadsheets/d/130hio-ntnPCZbGzmp1R3ROHXSpKQBUC0I_iM0uQPPi4/copy |
| Form Trigger (3 images) | formTrigger | Collect 3 uploaded images | — | Validate inputs | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Validate inputs | if | Ensure 3 binaries exist | Form Trigger (3 images) | Normalize binary names; Error Response - Missing Files | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Error Response - Missing Files | set | Error payload when inputs missing | Validate inputs (false) | — | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Normalize binary names | set | Prepare consistent fields (keeps binaries) | Validate inputs (true) | Split images | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Split images | code | Split 3 binaries into 3 items | Normalize binary names | OpenAI Vision – Image 1; Upload file | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Upload file | googleDrive | Store each reference image | Split images | Merge | # Step 1 –  Product Image Creation – NanoBanana Pro |
| OpenAI Vision – Image 1 | OpenAI (LangChain) | Describe each image | Split images | Merge | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Merge | merge | Combine Drive + Vision per image | Upload file; OpenAI Vision – Image 1 | Aggregate descriptions | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Aggregate descriptions | code | Build `allDescriptions` + `imageUrls` | Merge | Generate Image Prompt | # Step 1 –  Product Image Creation – NanoBanana Pro |
| LLM: OpenAI Chat | lmChatOpenAi | LLM backend for agent | — | Generate Image Prompt (ai_languageModel) | # Step 1 –  Product Image Creation – NanoBanana Pro |
| LLM: Structured Output Parser | outputParserStructured | Enforce JSON output schema | — | Generate Image Prompt (ai_outputParser) | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Generate Image Prompt | agent | Produce `image_prompt` for NanoBanana | Aggregate descriptions; LLM nodes | NanoBanana: Create Image | # Step 1 –  Product Image Creation – NanoBanana Pro |
| NanoBanana: Create Image | httpRequest | Generate hero image | Generate Image Prompt | Wait for Image Edit | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Wait for Image Edit | wait | Delay before downloading fal result | NanoBanana: Create Image | Download Edited Image | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Download Edited Image | httpRequest | Fetch NanoBanana result | Wait for Image Edit | Append row in sheet | # Step 1 –  Product Image Creation – NanoBanana Pro |
| Append row in sheet | googleSheets | Append `nano_done` row | Download Edited Image | — | # Step 1 –  Product Image Creation – NanoBanana Pro |
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry for contact sheet step | — | Get image nanobanana | # Step 2 – Contact Sheet Prompting Technique |
| Get image nanobanana | googleSheets | Read row(s) with NanoBanana image | When clicking ‘Execute workflow’ | Edit Fields : contactSheetPrompt | # Step 2 – Contact Sheet Prompting Technique |
| Edit Fields : contactSheetPrompt | set | Provide long prompt for contact sheet | Get image nanobanana | NanoBanana: Contact Sheet | # Step 2 – Contact Sheet Prompting Technique |
| NanoBanana: Contact Sheet | httpRequest | Generate 2×3 keyframe sheet | Edit Fields : contactSheetPrompt | Wait for Image Edit1 | # Step 2 – Contact Sheet Prompting Technique |
| Wait for Image Edit1 | wait | Delay | NanoBanana: Contact Sheet | Download Edited Image1 | # Step 2 – Contact Sheet Prompting Technique |
| Download Edited Image1 | httpRequest | Fetch contact sheet output | Wait for Image Edit1 | Update database | # Step 2 – Contact Sheet Prompting Technique |
| Update database | googleSheets | Update row with contactsheet URL/status | Download Edited Image1 | — | # Step 2 – Contact Sheet Prompting Technique |
| Schedule Trigger | scheduleTrigger | Periodic processing of new rows | — | Search new image | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Search new image | googleSheets | Find rows to process | Schedule Trigger | Set Image URL | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Set Image URL | set | Map `image_contactsheet` → `image_url` | Search new image | Download Image | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Download Image | httpRequest | Download contact sheet as file | Set Image URL | Edit Image | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Edit Image | editImage | Get image dimensions | Download Image | Crop Top Left/Center/Right; Crop Bottom Left/Center/Right | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Crop Top Left | editImage | Crop frame 1 | Edit Image | Upload to Google Drive | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload to Google Drive | googleDrive | Upload cropped frame 1 | Crop Top Left | Update url image_top_left | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update url image_top_left | googleSheets | Write `new_image_1` | Upload to Google Drive | Merge1 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Crop Top Center | editImage | Crop frame 2 | Edit Image | Upload to Google Drive1 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload to Google Drive1 | googleDrive | Upload cropped frame 2 | Crop Top Center | Update url image_top_center | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update url image_top_center | googleSheets | Write `new_image_2` | Upload to Google Drive1 | Merge1; Merge2 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Crop Top Right | editImage | Crop frame 3 | Edit Image | Upload to Google Drive2 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload to Google Drive2 | googleDrive | Upload cropped frame 3 | Crop Top Right | Update url image_top_right | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update url image_top_right | googleSheets | Write `new_image_3` | Upload to Google Drive2 | Merge2; Merge3 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Crop Bottom Left | editImage | Crop frame 4 | Edit Image | Upload to Google Drive3 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload to Google Drive3 | googleDrive | Upload cropped frame 4 | Crop Bottom Left | Update url image_bottom_left | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update url image_bottom_left | googleSheets | Write `new_image_4` | Upload to Google Drive3 | Merge3; Merge4 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Crop Bottom Center | editImage | Crop frame 5 | Edit Image | Upload to Google Drive4 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload to Google Drive4 | googleDrive | Upload cropped frame 5 | Crop Bottom Center | Update url image_bottom_center | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update url image_bottom_center | googleSheets | Write `new_image_5` | Upload to Google Drive4 | Merge4; Merge5 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Crop Bottom Right | editImage | Crop frame 6 | Edit Image | Upload to Google Drive5 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload to Google Drive5 | googleDrive | Upload cropped frame 6 | Crop Bottom Right | Update url image_bottom_right | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update url image_bottom_right | googleSheets | Write `new_image_6` | Upload to Google Drive5 | Merge5 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge1 | merge | Combine new_image_1+2 | Update url image_top_left; Update url image_top_center | Veo Generation | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Veo Generation | httpRequest | Video 1 from frames 1→2 | Merge1 | Wait | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Wait | wait | Delay before sheet update | Veo Generation | Update video 1 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update video 1 | googleSheets | Write `video_1` | Wait | Merge6 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge2 | merge | Combine new_image_2+3 | Update url image_top_center; Update url image_top_right | Veo Generation1 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Veo Generation1 | httpRequest | Video 2 from frames 2→3 | Merge2 | Wait1 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Wait1 | wait | Delay | Veo Generation1 | Update video 2 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update video 2 | googleSheets | Write `video_2` | Wait1 | Merge6 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge3 | merge | Combine new_image_3+4 | Update url image_top_right; Update url image_bottom_left | Veo Generation2 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Veo Generation2 | httpRequest | Video 3 from frames 3→4 | Merge3 | Wait2 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Wait2 | wait | Delay | Veo Generation2 | Update video 3 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update video 3 | googleSheets | Write `video_3` | Wait2 | Merge6 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge4 | merge | Combine new_image_4+5 | Update url image_bottom_left; Update url image_bottom_center | Veo Generation3 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Veo Generation3 | httpRequest | Video 4 from frames 4→5 | Merge4 | Wait3 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Wait3 | wait | Delay | Veo Generation3 | Update video 4 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update video 4 | googleSheets | Write `video_4` | Wait3 | Merge6 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge5 | merge | Combine new_image_5+6 | Update url image_bottom_center; Update url image_bottom_right | Veo Generation4 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Veo Generation4 | httpRequest | Video 5 from frames 5→6 | Merge5 | Wait4 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Wait4 | wait | Delay | Veo Generation4 | Update video 5 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update video 5 | googleSheets | Write `video_5` | Wait4 | Merge6 | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge6 | merge | Collect 5 video update branches | Update video 1..5 | Merge 3 Videos | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Merge 3 Videos | httpRequest | Merge clips into final MP4 | Merge6 | Wait: Merge Process | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Wait: Merge Process | wait | Delay | Merge 3 Videos | Update URL Final video | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Update URL Final video | googleSheets | Write final URL + status done | Wait: Merge Process | Upload Video to BLOTATO | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Upload Video to BLOTATO | blotato | Upload media asset | Update URL Final video | Youtube | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |
| Youtube | blotato | Create YouTube post | Upload Video to BLOTATO | — | # Step 3 – Video Creation & Publishing (Veo 3.1 + Blotato) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Google Sheets OAuth2 credential named **`Google Sheets account`**
   2. Google Drive OAuth2 credential named **`Google Drive account`**
   3. OpenAI API credential named **`OpenAi account`**
   4. Blotato API credential named **`Blotato account`**
   5. Store your **fal.ai API key** in a safe place (preferably as an n8n credential or environment variable). Do **not** hard-code it in nodes.

2) **Step 1: Form intake + validation**
   1. Add **Form Trigger** node:
      - Title: “Upload 3 Images for Analysis”
      - Fields: `image1` (file, required), `image2` (file, required), `image3` (file, required), optional `notes`
   2. Add **IF** node “Validate inputs”:
      - Conditions: binary `image1`, `image2`, `image3` exist
   3. Add **Set** node “Error Response - Missing Files” on false branch:
      - `error`: “Please upload 3 images (image1, image2, image3).”
   4. Add **Set** node “Normalize binary names” on true branch:
      - Keep `includeOtherFields = true` (so binary remains available)
   5. Add **Code** node “Split images”:
      - Input: one item with `$binary.image1..3`
      - Output: 3 items each with `binary.image`

3) **Step 1: Upload to Drive + OpenAI Vision**
   1. Add **Google Drive** node “Upload file”:
      - Upload binary property: `image`
      - File name: `{{$binary.image.fileName}}`
      - Select Drive & Folder
   2. Add **OpenAI (LangChain) Image Analyze** node:
      - Model: `gpt-4o`
      - Input: base64 from binary property `image`
      - Prompt: detailed image description instruction
   3. Add **Merge** node (combine by position) to merge Drive upload result + OpenAI result.

4) **Step 1: Aggregate + prompt generation**
   1. Add **Code** node “Aggregate descriptions”:
      - Extract the text from OpenAI result and link from Drive
      - Output a single item containing `allDescriptions` and `imageUrls[]`
   2. Add **LM Chat OpenAI** node (model `gpt-4.1-mini`).
   3. Add **Structured Output Parser** with schema example: `{ "image_prompt": "string" }`
   4. Add **Agent** node “Generate Image Prompt”:
      - Provide system message template (as in workflow)
      - Input text includes `{{$json.allDescriptions}}`
      - Attach the LM node and output parser via AI connections.

5) **Step 1: NanoBanana hero image + append to sheet**
   1. Add **HTTP Request** “NanoBanana: Create Image”:
      - POST JSON to `https://queue.fal.run/fal-ai/nano-banana-pro/edit`
      - Headers: Authorization (fal key), Content-Type
      - Body includes prompt and `image_urls` from aggregated `imageUrls`
   2. Add **Wait** 2 minutes (or implement polling).
   3. Add **HTTP Request** to GET `response_url`.
   4. Add **Google Sheets** “Append row in sheet”:
      - Operation append
      - Write status `nano_done`, store image URLs, descriptions, and `image_nanobanana`.

6) **Step 2: Contact sheet generation (manual path)**
   1. Add **Manual Trigger**.
   2. Add **Google Sheets** read node “Get image nanobanana” (ideally filter status `nano_done`).
   3. Add **Set** node with `contactSheetPrompt` (the long continuity + 2×3 sheet instruction).
   4. Add **HTTP Request** “NanoBanana: Contact Sheet”:
      - POST to same NanoBanana queue endpoint
      - Body: `prompt=contactSheetPrompt`, `image_urls=[image_nanobanana]`, `aspect_ratio=3:2`
   5. Add **Wait** 2 minutes.
   6. Add **HTTP Request** GET `response_url`.
   7. Add **Google Sheets** update node:
      - Match by `image_nanobanana`
      - Set `status=ContactSheet_done`, `image_contactsheet=<contactsheet url>`

7) **Step 3: Scheduled cropping + frame upload**
   1. Add **Schedule Trigger** (configure interval/cron).
   2. Add **Google Sheets** “Search new image”:
      - Filter rows where `status = ContactSheet_done` and `new_image_1` empty (recommended).
   3. Add **Set** “Set Image URL”: `image_url = image_contactsheet`
   4. Add **HTTP Request** download as **file** from `image_url`.
   5. Add **Edit Image** operation `information` to get `size.width/height`.
   6. Add 6 **Edit Image** crop nodes for 2×3 grid.
   7. Add 6 **Google Drive upload** nodes; ensure sharing makes links usable by fal/Veo.
   8. Add 6 **Google Sheets update** nodes to write `new_image_1..6`, matched by `image_contactsheet`.

8) **Step 3: Veo video generation (5 clips)**
   1. Add 5 **Merge** nodes (combine-by-position) to pair consecutive `new_image_n` values.
   2. Add 5 **HTTP Request** nodes calling:
      - `https://fal.run/fal-ai/veo3.1/first-last-frame-to-video`
      - with `first_frame_url` and `last_frame_url`
   3. Add **Wait** nodes (or polling) after each.
   4. Add 5 **Google Sheets update** nodes to write `video_1..video_5`.

9) **Final merge + publish**
   1. Add **Merge** node with 5 inputs (or re-read the sheet row to ensure you have all `video_n` fields).
   2. Add **HTTP Request** to `https://fal.run/fal-ai/ffmpeg-api/merge-videos` with `video_urls[]`.
   3. Add **Wait**.
   4. Add **Google Sheets update** to set `Final video` and `status=done`.
   5. Add **Blotato** node to upload media by URL.
   6. Add **Blotato** node to create YouTube post:
      - account, title, privacy settings, and attach uploaded media URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| fal NanoBanana Pro endpoint | https://fal.ai/models/fal-ai/nano-banana-pro/edit/api |
| fal Veo 3.1 endpoint | https://fal.ai/models/fal-ai/veo3.1/first-last-frame-to-video |
| Blotato signup | https://blotato.com/?ref=firas |
| Full documentation (Notion) | https://automatisation.notion.site/Generate-product-images-with-NanoBanana-Pro-to-Veo-videos-and-Blotato-2dc3d6550fd980dc81adea10a5df6b28?source=copy_link |
| Copy the Google Sheet template | https://docs.google.com/spreadsheets/d/130hio-ntnPCZbGzmp1R3ROHXSpKQBUC0I_iM0uQPPi4/copy |
| Security note | One node contains a hard-coded fal API key in the Authorization header. Move it to credentials or environment variables before sharing/deploying. |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.