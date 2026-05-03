Generate frozen ASMR product videos with Gemini, Veo3, GPT-4o and post to YouTube, TikTok, Instagram and Pinterest

https://n8nworkflows.xyz/workflows/generate-frozen-asmr-product-videos-with-gemini--veo3--gpt-4o-and-post-to-youtube--tiktok--instagram-and-pinterest-11756


# Generate frozen ASMR product videos with Gemini, Veo3, GPT-4o and post to YouTube, TikTok, Instagram and Pinterest

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate frozen ASMR product videos with Gemini, Veo3, GPT-4o and post to YouTube, TikTok, Instagram and Pinterest

**Purpose:**  
This workflow periodically checks a Google Sheet for a new product row (topic + product image URL). For each eligible row, it (1) downloads the product image, (2) uses **Gemini** to create a “frozen” version of that image, (3) uploads that frozen image to **ImgBB** to obtain a public URL, (4) asks **Veo3 (via Kie.ai)** to generate a short vertical ASMR-style video from that frozen image, (5) generates platform-specific captions with **GPT‑4o**, (6) uploads/publishes the resulting video to **YouTube** and to **TikTok/Instagram/Pinterest** via **upload-post.com**, then (7) updates the sheet and notifies via Telegram.

### 1.1 Scheduling & Sheet Intake
- Trigger on a schedule, read the sheet, pick the first row that looks “new”.

### 1.2 Validation & Locking (Mark as Processing)
- Validate required fields; mark the row as `processing` in the sheet to reduce duplicates.

### 1.3 Freeze Image (Gemini)
- Download the original product image, build a strict prompt preserving identity/text, request an IMAGE output from Gemini.

### 1.4 Host Frozen Image + Build Video Prompt
- Extract base64 image from Gemini response, upload to ImgBB, build a Veo3 prompt and payload.

### 1.5 Generate Video (Veo3) + Polling
- Submit Veo3 task, wait, check status, loop until ready.

### 1.6 Write Captions (GPT‑4o) + Prepare Upload Payload
- Ask GPT‑4o for JSON captions/titles; parse safely; combine with Veo3 video URL.

### 1.7 Publish + Sheet Update + Telegram Notification
- Download the video file (for YouTube upload), upload to YouTube, and submit to upload-post.com for TikTok/Instagram/Pinterest; update sheet and send Telegram message.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Sheet Intake
**Overview:** Runs twice daily and loads rows from a Google Sheet.  
**Nodes involved:** `Schedule Trigger2`, `Get Sheet Data`

#### Node: Schedule Trigger2
- **Type / role:** Schedule Trigger; entry point.
- **Configuration:** Runs at **09:00** and **15:00** (hours). Interval-based rule with two trigger hours.
- **Outputs:** To `Get Sheet Data`.
- **Edge cases:**
  - Timezone is n8n instance timezone; may not match your local timezone.
  - If the workflow execution overlaps (slow Veo3 renders), you may have concurrent runs unless concurrency is limited.

#### Node: Get Sheet Data
- **Type / role:** Google Sheets node; reads sheet data.
- **Configuration choices:**
  - Document ID: `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `gid=0` (Sheet1)
  - Operation not explicitly shown in JSON; by context it is reading rows from the sheet.
- **Outputs:** To `Validate Input2`.
- **Failure types:**
  - Google OAuth credential missing/expired.
  - Spreadsheet ID wrong or permissions missing.
  - Data shape issues: missing columns (`topic`, `image_url`, `status`, optional `row_number`).

---

### Block 2 — Validation & Locking (Mark Processing)
**Overview:** Selects one “new” row and normalizes fields, then marks it as processing.  
**Nodes involved:** `Validate Input2`, `Mark Processing2`

#### Node: Validate Input2
- **Type / role:** Code node; selects a valid candidate row.
- **Logic:**
  - Iterates all sheet rows.
  - Finds the first row where:
    - `topic` is non-empty
    - `image_url` is non-empty
    - `status` is not in: `done`, `processing`, `uploaded`, `error` (case-insensitive)
  - Splits `topic` by `—` (em dash) to derive:
    - `scent_name` = left part
    - `brand_name` = right part (optional)
  - Outputs normalized JSON:
    - `topic`, `scent_name`, `brand_name`, `reference_url` (from `image_url`), `row_number` (default 2 if missing)
- **Key expressions/variables:** `item.json.topic`, `item.json.image_url`, `item.json.status`
- **Outputs:** To `Mark Processing2`. Returns `[]` (no items) if nothing valid found.
- **Edge cases / failures:**
  - If the sheet has multiple new rows, it processes only the first match.
  - If you use a hyphen instead of em dash in `topic`, `brand_name` will be empty.
  - `row_number` is not actually used later to target a row; updates are matched by `topic`.

#### Node: Mark Processing2
- **Type / role:** Google Sheets node; updates row status to prevent reprocessing.
- **Configuration:**
  - Operation: `update`
  - Matching columns: `topic`
  - Sets:
    - `topic` = current topic
    - `status` = `processing`
  - `onError: continueRegularOutput` (it will continue even if this update fails).
- **Input:** From `Validate Input2`.
- **Output:** To `Download Product Image`.
- **Failure types / edge cases:**
  - If topic is not unique in the sheet, **multiple rows can be updated** unintentionally.
  - If the update fails but the workflow continues, you can get duplicate processing in later runs.

---

### Block 3 — Freeze Image (Gemini)
**Overview:** Downloads the product image, builds a “frozen effect” request while preserving identity/text, and calls Gemini image generation.  
**Nodes involved:** `Download Product Image`, `Build Image Prompt`, `Gemini (Frozen Image)`

#### Node: Download Product Image
- **Type / role:** HTTP Request; fetches the source image as binary file.
- **Configuration:**
  - URL: `$('Validate Input2').item.json.reference_url`
  - Response format: `file` (binary)
- **Outputs:** To `Build Image Prompt`.
- **Failure types:**
  - 404/403 (image URL not public).
  - Non-image content served (HTML page, hotlink protection).
  - Large file or slow host causing timeouts.

#### Node: Build Image Prompt
- **Type / role:** Code node; converts binary image to base64 and builds Gemini request body.
- **Key steps:**
  - Reads binary from `item.binary.data || file || image`.
  - Validates it exists and length > 50.
  - Removes any `data:*;base64,` prefix.
  - Builds a strict prompt enforcing:
    - **Identity lock** (same geometry)
    - **Text lock** (do not alter text/logos)
    - “Frozen treatment” style transform only
    - 9:16 background/lighting constraints
  - Builds `request_body` with:
    - `contents[].parts[]`: `inline_data` (image) + `text` (prompt)
    - `generationConfig.responseModalities = ["IMAGE"]`
    - `imageConfig.aspectRatio = "9:16"`
- **Outputs:** To `Gemini (Frozen Image)`.
- **Failure types / edge cases:**
  - If upstream binary property name differs, it may not find the image.
  - Some images may be too large for model limits; Gemini can reject with payload size errors.

#### Node: Gemini (Frozen Image)
- **Type / role:** HTTP Request; calls Gemini generateContent endpoint.
- **Configuration:**
  - POST to: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent`
  - Headers:
    - `x-goog-api-key: YOUR_GOOGLE_AI_API_KEY`
    - `Content-Type: application/json`
  - Body: `{{$json.request_body}}`
- **Outputs:** To `Extract Image`.
- **Failure types:**
  - Invalid API key / billing disabled.
  - Model name changes/deprecation (preview model).
  - Safety/format errors; response may not include expected image data structure.

---

### Block 4 — Host Frozen Image + Build Video Prompt
**Overview:** Extracts the generated frozen image as base64, uploads it to ImgBB to get a public URL, then composes a detailed Veo3 prompt.  
**Nodes involved:** `Extract Image`, `Upload to ImgBB`, `Build Video Prompt`

#### Node: Extract Image
- **Type / role:** Code node; parses Gemini response into `image_base64`.
- **Logic:**
  - Tries `response.candidates[0].content.parts[].inlineData.data`
  - Fallback: `response.predictions[0].bytesBase64Encoded`
  - Throws error if not found.
  - Outputs:
    - `image_base64`
    - `topic`, `scent_name`, `brand_name` pulled from `Validate Input2`
- **Outputs:** To `Upload to ImgBB`.
- **Failure types / edge cases:**
  - Gemini response schema differences (e.g., `inline_data` vs `inlineData` casing).
  - Empty candidates due to refusal or transient errors -> hard failure.

#### Node: Upload to ImgBB
- **Type / role:** HTTP Request; hosts base64 image publicly.
- **Configuration:**
  - POST to: `https://api.imgbb.com/1/upload?key=YOUR_TOKEN_HERE`
  - Content-Type: `form-urlencoded`
  - Body param: `image = {{$json.image_base64}}`
- **Outputs:** To `Build Video Prompt`.
- **Failure types:**
  - Invalid ImgBB key, quota exceeded.
  - Response may not contain `data.display_url` if upload fails.

#### Node: Build Video Prompt
- **Type / role:** Code node; constructs an explicit ASMR Veo3 prompt and attaches the hosted image URL.
- **Key outputs:**
  - `video_prompt`: long “restriction-heavy” prompt (no text/logos/knives; blunt scraper tool definition; macro frozen packshot; audio constraints).
  - `image_url`: from ImgBB `data.display_url`
  - Pass-through: `topic`, `scent_name`, `brand_name`
- **Outputs:** To `Generate Video (Veo3)`.
- **Edge cases:**
  - If ImgBB returns a different field name, `imageUrl` becomes undefined and Veo3 request fails.

---

### Block 5 — Generate Video (Veo3) + Polling
**Overview:** Submits a Veo3 video generation task, then polls every 30 seconds until the render is ready.  
**Nodes involved:** `Generate Video (Veo3)`, `Wait 30s`, `Check Video Status`, `Ready?`, `Wait & Retry2`

#### Node: Generate Video (Veo3)
- **Type / role:** HTTP Request; starts Veo generation via Kie.ai.
- **Configuration:**
  - POST `https://api.kie.ai/api/v1/veo/generate`
  - Timeout: 60s
  - JSON body includes:
    - `prompt` = `{{$json.video_prompt}}`
    - `model` = `veo3_fast`
    - `aspectRatio` = `9:16`
    - `imageUrls` = `[{{$json.image_url}}]`
  - Header: `Authorization: Bearer YOUR_TOKEN_HERE`
- **Outputs:** To `Wait 30s`.
- **Failure types:**
  - Invalid token / insufficient credits.
  - 4xx if prompt violates provider policy.
  - 60s timeout if endpoint is slow to accept jobs.

#### Node: Wait 30s
- **Type / role:** Wait node; delay before polling.
- **Configuration:** Wait **30 seconds**.
- **Outputs:** To `Check Video Status`.

#### Node: Check Video Status
- **Type / role:** HTTP Request; polls task status.
- **Configuration:**
  - GET/Request to `https://api.kie.ai/api/v1/veo/record-info` (uses query params)
  - Query: `taskId = $('Generate Video (Veo3)').item.json.data.taskId`
  - Header: `Authorization: Bearer YOUR_TOKEN_HERE`
- **Outputs:** To `Ready?`.
- **Failure types:**
  - If `taskId` path differs in response, polling breaks.
  - Transient 5xx; no retry except via workflow loop.

#### Node: Ready?
- **Type / role:** IF node; checks completion.
- **Condition:** `{{$json.data.successFlag}} == 1`
- **True output:** To `GPT-4o (Captions)`
- **False output:** To `Wait & Retry2`
- **Edge cases:**
  - Provider may use different flags (`status`, `state`) -> condition never becomes true.

#### Node: Wait & Retry2
- **Type / role:** Wait node; backoff.
- **Configuration:** Wait **30 seconds** then loops to `Check Video Status`.
- **Risk:** Infinite loop if task never reaches `successFlag = 1`. Consider adding a max retries counter.

---

### Block 6 — Write Captions (GPT‑4o) + Prepare Upload Payload
**Overview:** Generates platform-specific copy as JSON, parses it defensively, and compiles final upload metadata including the Veo result URL.  
**Nodes involved:** `GPT-4o (Captions)`, `Prepare Upload2`

#### Node: GPT-4o (Captions)
- **Type / role:** OpenAI (LangChain) node; text generation.
- **Configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.9`
  - System prompt enforces:
    - hypnotic micro-text, short, observational, mysterious
    - hard banned words (knife/blade/cut/etc.)
    - **Return JSON only** with specific keys
  - User message references: `TOPIC: '{{ $('Build Video Prompt').item.json.topic }}'`
- **Outputs:** To `Prepare Upload2`.
- **Failure types:**
  - Missing OpenAI credentials.
  - Model refusal or returning non-JSON (handled partially downstream).
  - Content policy issues if topic includes prohibited content.

#### Node: Prepare Upload2
- **Type / role:** Code node; parses GPT output + attaches video URL.
- **Key logic:**
  - Gets Veo video URL from: `$('Check Video Status').item.json.data.response.resultUrls[0]`
  - Attempts to parse JSON from `response.message.content` (or `response.content`)
  - Strips ```json fences if present
  - Fallback to defaults if parsing fails
- **Outputs:** To `Download Video`.
- **Edge cases:**
  - If Veo status response does not contain `resultUrls[0]`, uploads will fail.
  - If GPT returns malformed JSON, captions become empty strings (still uploads).

---

### Block 7 — Publish + Update + Notify
**Overview:** Downloads the rendered video file for YouTube upload, also sends the hosted video URL to upload-post.com for TikTok/Instagram/Pinterest, then updates Google Sheets and sends Telegram confirmation.  
**Nodes involved:** `Download Video`, `YouTube`, `TikTok`, `Instagram`, `Pinterest`, `Update Sheet`, `Telegram2`

#### Node: Download Video
- **Type / role:** HTTP Request; fetches final video as binary.
- **Configuration:**
  - URL: `{{$json.video_url}}`
  - Response format: `file`
- **Outputs:** Fan-out to `YouTube`, `Pinterest`, `TikTok`, `Instagram`.
- **Failure types:**
  - Video URL expired/not public.
  - Large file download timeouts (no custom timeout set here).

#### Node: YouTube
- **Type / role:** YouTube node; uploads video.
- **Configuration:**
  - Resource: `video` / Operation: `upload`
  - Title: `$('Prepare Upload2').item.json.youtube_title`
  - Description: `$('Prepare Upload2').item.json.youtube_description`
  - CategoryId: `24`
  - Region: `US`
- **Input:** Expects binary video from `Download Video`.
- **Output:** To `Update Sheet` (provides `uploadId`).
- **Failure types:**
  - OAuth not connected.
  - YouTube upload limits/processing errors.
  - If binary property name mismatches YouTube node expectations, upload fails.

#### Node: TikTok
- **Type / role:** HTTP Request; submits upload job to upload-post.com for TikTok.
- **Configuration:**
  - POST `https://api.upload-post.com/api/upload`
  - Multipart form-data:
    - `title` = YouTube title
    - `description` + `caption` = TikTok caption
    - `user` = `YOUR_TIKTOK_USERNAME`
    - `platform[]` = `tiktok`
    - `mediaUrl` = hosted video URL (not the binary)
  - Header: `Authorization: Apikey YOUR_UPLOAD_POST_API_KEY`
- **Output:** To `Update Sheet`
- **Failure types / edge cases:**
  - If upload-post expects a file upload rather than `mediaUrl`, it will fail.
  - Missing/incorrect username mapping on upload-post side.

#### Node: Instagram
- **Type / role:** HTTP Request; submits upload job to upload-post.com for Instagram.
- **Configuration:** Same endpoint and auth; `platform[] = instagram`, `user = YOUR_INSTAGRAM_USERNAME`, captions from `instagram_caption`.
- **Output:** To `Update Sheet`
- **Failure types:** Same as TikTok; also Instagram API constraints (reels/format) handled by provider.

#### Node: Pinterest
- **Type / role:** HTTP Request; submits upload job to upload-post.com for Pinterest.
- **Configuration:**
  - `platform[] = pinterest`
  - `user = YOUR_PINTEREST_USERNAME`
  - `boardId = YOUR_PINTEREST_BOARD_ID`
  - Title/description from Pinterest fields
- **onError:** `continueRegularOutput` (Pinterest failures won’t stop sheet update path for this node).
- **Output:** To `Update Sheet`
- **Failure types:**
  - Invalid board ID / permissions.
  - Provider limitations on video pins.

#### Node: Update Sheet
- **Type / role:** Google Sheets update; marks success and stores YouTube info.
- **Configuration:**
  - Matching columns: `topic`
  - Sets:
    - `status = uploaded`
    - `video_id = {{$json.uploadId}}` (expects input from YouTube node)
    - `youtube_url = https://youtube.com/shorts/{{$json.uploadId}}`
- **Inputs:** Can be triggered by **YouTube, TikTok, Instagram, Pinterest** (all connected).
- **Critical edge case (design issue):**
  - When triggered by TikTok/Instagram/Pinterest, `$json.uploadId` likely **does not exist**, so the sheet may be updated with empty/undefined `video_id` and incorrect `youtube_url`.  
  - Because all four platforms converge into this single node, whichever finishes first may update the row prematurely or incorrectly.
- **Output:** To `Telegram2`.

#### Node: Telegram2
- **Type / role:** Telegram node; sends completion message.
- **Configuration:**
  - Chat ID: `YOUR_TELEGRAM_CHAT_ID`
  - Text:
    - `✓ {topic}`
    - `https://youtube.com/shorts/{uploadId}`
- **Input:** From `Update Sheet` (so it inherits the same `uploadId` dependency issue).
- **Failure types:**
  - Telegram bot token not configured in credentials.
  - Chat ID incorrect (bot not allowed in chat).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow description / setup notes | — | — | ## ❄️ Frozen ASMR Video Creator \nCreate "frozen" product videos automatically and post them to social media. No coding required.\n\n### How it works\n1. **Sheet:** Finds a new product image in your Google Sheet.\n2. **Ice Effect:** Uses Gemini AI to freeze the image.\n3. **Video:** Uses Veo3 to animate the image into a 10s ASMR video.\n4. **Captions:** GPT-4o writes the text for you.\n5. **Publish:** Posts to YouTube, TikTok, Instagram, and Pinterest.\n\n### Setup steps\n1. **Get Keys:** You need API keys for Gemini, Kie.ai, ImgBB, OpenAI, and upload-post.com.\n2. **Prepare Sheet:** Make a Google Sheet with columns: `topic`, `image_url`, `status`.\n3. **Connect:** Double-click the YouTube and Telegram nodes to connect your accounts.\n4. **Config:** Press `Ctrl+F`, search for "YOUR_", and paste your real API keys. |
| Sticky Note1 | Sticky Note | Block label: Get Data | — | — | ## 1. Get Data\nChecks your Google Sheet for new rows. If the row is clean, it starts the process. |
| Sticky Note2 | Sticky Note | Block label: Freeze Image | — | — | ## 2. Freeze Image\nSends the photo to Gemini AI to add a realistic ice and frost effect. |
| Sticky Note3 | Sticky Note | Block label: Make Video | — | — | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Sticky Note4 | Sticky Note | Block label: Write & Post | — | — | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Schedule Trigger2 | Schedule Trigger | Start executions at set times | — | Get Sheet Data | ## 1. Get Data\nChecks your Google Sheet for new rows. If the row is clean, it starts the process. |
| Get Sheet Data | Google Sheets | Read rows from sheet | Schedule Trigger2 | Validate Input2 | ## 1. Get Data\nChecks your Google Sheet for new rows. If the row is clean, it starts the process. |
| Validate Input2 | Code | Pick first valid “new” row, normalize fields | Get Sheet Data | Mark Processing2 | ## 1. Get Data\nChecks your Google Sheet for new rows. If the row is clean, it starts the process. |
| Mark Processing2 | Google Sheets | Update row status to `processing` | Validate Input2 | Download Product Image | ## 1. Get Data\nChecks your Google Sheet for new rows. If the row is clean, it starts the process. |
| Download Product Image | HTTP Request | Download source image as binary | Mark Processing2 | Build Image Prompt | ## 2. Freeze Image\nSends the photo to Gemini AI to add a realistic ice and frost effect. |
| Build Image Prompt | Code | Build Gemini image-generation request body | Download Product Image | Gemini (Frozen Image) | ## 2. Freeze Image\nSends the photo to Gemini AI to add a realistic ice and frost effect. |
| Gemini (Frozen Image) | HTTP Request | Generate frozen image with Gemini | Build Image Prompt | Extract Image | ## 2. Freeze Image\nSends the photo to Gemini AI to add a realistic ice and frost effect. |
| Extract Image | Code | Extract base64 image from Gemini response | Gemini (Frozen Image) | Upload to ImgBB | ## 2. Freeze Image\nSends the photo to Gemini AI to add a realistic ice and frost effect. |
| Upload to ImgBB | HTTP Request | Host frozen image, get public URL | Extract Image | Build Video Prompt | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Build Video Prompt | Code | Compose Veo3 prompt + attach image URL | Upload to ImgBB | Generate Video (Veo3) | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Generate Video (Veo3) | HTTP Request | Submit Veo3 generation task (Kie.ai) | Build Video Prompt | Wait 30s | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Wait 30s | Wait | Delay before polling status | Generate Video (Veo3) | Check Video Status | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Check Video Status | HTTP Request | Poll Veo task status | Wait 30s; Wait & Retry2 | Ready? | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Ready? | IF | Branch on `successFlag == 1` | Check Video Status | GPT-4o (Captions); Wait & Retry2 | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| Wait & Retry2 | Wait | Retry delay loop | Ready? (false) | Check Video Status | ## 3. Make Video\nSends the frozen image to Veo3. It waits automatically until the video is fully rendered and ready to download. |
| GPT-4o (Captions) | OpenAI (LangChain) | Generate captions/titles JSON | Ready? (true) | Prepare Upload2 | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Prepare Upload2 | Code | Parse GPT JSON; attach video URL | GPT-4o (Captions) | Download Video | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Download Video | HTTP Request | Download generated video as binary | Prepare Upload2 | YouTube; Pinterest; TikTok; Instagram | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| YouTube | YouTube | Upload video to YouTube Shorts | Download Video | Update Sheet | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| TikTok | HTTP Request | Upload via upload-post.com (TikTok) | Download Video | Update Sheet | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Instagram | HTTP Request | Upload via upload-post.com (Instagram) | Download Video | Update Sheet | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Pinterest | HTTP Request | Upload via upload-post.com (Pinterest) | Download Video | Update Sheet | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Update Sheet | Google Sheets | Mark row uploaded + store YouTube ID/URL | YouTube; TikTok; Instagram; Pinterest | Telegram2 | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |
| Telegram2 | Telegram | Notify completion with YouTube link | Update Sheet | — | ## 4. Write & Post\nGPT-4o writes descriptions for each app. Then, the workflow uploads the video to all 4 platforms at once. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it exactly as the title.

2. **Add a Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure two daily trigger hours: **09:00** and **15:00** (adjust timezone as needed).

3. **Add Google Sheets: Get Sheet Data**
   - Node: **Google Sheets**
   - Credentials: connect Google account with access to the spreadsheet.
   - Document ID: your sheet ID.
   - Sheet: select Sheet1 / `gid=0`.
   - Operation: read/get rows (default “Read” style operation).
   - Connect: `Schedule Trigger -> Get Sheet Data`.

4. **Add Code: Validate Input2**
   - Node: **Code**
   - Paste logic to:
     - select first row where `topic` and `image_url` are non-empty and `status` not in `done/processing/uploaded/error`
     - output `topic, scent_name, brand_name, reference_url, row_number`
   - Connect: `Get Sheet Data -> Validate Input2`.

5. **Add Google Sheets: Mark Processing2**
   - Node: **Google Sheets**
   - Operation: **Update**
   - Matching column: `topic`
   - Set `status = processing`
   - Enable “Continue on Fail” (onError continue) if you want same behavior.
   - Connect: `Validate Input2 -> Mark Processing2`.

6. **Add HTTP Request: Download Product Image**
   - Node: **HTTP Request**
   - URL: expression = `{{$('Validate Input2').item.json.reference_url}}`
   - Response: **File**
   - Connect: `Mark Processing2 -> Download Product Image`.

7. **Add Code: Build Image Prompt**
   - Node: **Code**
   - Convert binary to base64, build Gemini `generateContent` body with `responseModalities: ["IMAGE"]` and `aspectRatio: "9:16"`.
   - Connect: `Download Product Image -> Build Image Prompt`.

8. **Add HTTP Request: Gemini (Frozen Image)**
   - Node: **HTTP Request**
   - Method: POST
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent`
   - Headers:
     - `x-goog-api-key`: your Google AI API key
     - `Content-Type`: `application/json`
   - Body: JSON = `{{$json.request_body}}`
   - Connect: `Build Image Prompt -> Gemini (Frozen Image)`.

9. **Add Code: Extract Image**
   - Node: **Code**
   - Parse Gemini response and output `image_base64` plus topic fields.
   - Connect: `Gemini (Frozen Image) -> Extract Image`.

10. **Add HTTP Request: Upload to ImgBB**
    - Node: **HTTP Request**
    - Method: POST
    - URL: `https://api.imgbb.com/1/upload?key=YOUR_IMGBB_KEY`
    - Content-Type: `form-urlencoded`
    - Body field: `image = {{$json.image_base64}}`
    - Connect: `Extract Image -> Upload to ImgBB`.

11. **Add Code: Build Video Prompt**
    - Node: **Code**
    - Read `data.display_url` from ImgBB output.
    - Compose the Veo3 prompt and output `video_prompt` + `image_url`.
    - Connect: `Upload to ImgBB -> Build Video Prompt`.

12. **Add HTTP Request: Generate Video (Veo3)**
    - Node: **HTTP Request**
    - Method: POST
    - URL: `https://api.kie.ai/api/v1/veo/generate`
    - Timeout: 60000 ms
    - Header: `Authorization: Bearer <KIE_AI_TOKEN>`
    - JSON body:
      - `prompt`, `model: veo3_fast`, `aspectRatio: 9:16`, `imageUrls: [image_url]`
    - Connect: `Build Video Prompt -> Generate Video (Veo3)`.

13. **Add Wait: Wait 30s**
    - Node: **Wait**
    - Amount: 30 seconds
    - Connect: `Generate Video (Veo3) -> Wait 30s`.

14. **Add HTTP Request: Check Video Status**
    - Node: **HTTP Request**
    - URL: `https://api.kie.ai/api/v1/veo/record-info`
    - Query param: `taskId = {{$('Generate Video (Veo3)').item.json.data.taskId}}`
    - Header: `Authorization: Bearer <KIE_AI_TOKEN>`
    - Connect: `Wait 30s -> Check Video Status`.

15. **Add IF: Ready?**
    - Node: **IF**
    - Condition (Number equals): `{{$json.data.successFlag}}` equals `1`
    - True path -> captions; False path -> retry wait
    - Connect: `Check Video Status -> Ready?`.

16. **Add Wait: Wait & Retry2**
    - Node: **Wait**
    - Amount: 30 seconds
    - Connect: `Ready? (false) -> Wait & Retry2 -> Check Video Status` (loop).

17. **Add OpenAI: GPT-4o (Captions)**
    - Node: **OpenAI (LangChain)**
    - Credentials: OpenAI API key in n8n credentials.
    - Model: `gpt-4o`
    - Temperature: 0.9
    - Messages:
      - System: rules + required JSON keys (as in workflow)
      - User: topic reference, plus constraints
    - Connect: `Ready? (true) -> GPT-4o (Captions)`.

18. **Add Code: Prepare Upload2**
    - Node: **Code**
    - Parse GPT JSON from the message content, strip code fences.
    - Pull video URL from `Check Video Status` at `data.response.resultUrls[0]`.
    - Output combined object with captions + `video_url`.
    - Connect: `GPT-4o (Captions) -> Prepare Upload2`.

19. **Add HTTP Request: Download Video**
    - Node: **HTTP Request**
    - URL: `{{$json.video_url}}`
    - Response: **File**
    - Connect: `Prepare Upload2 -> Download Video`.

20. **Add YouTube: Upload**
    - Node: **YouTube**
    - Credentials: connect YouTube account (OAuth2).
    - Operation: Upload video
    - Title/Description from `Prepare Upload2`
    - CategoryId: 24, Region: US
    - Connect: `Download Video -> YouTube`.

21. **Add HTTP Request: TikTok (upload-post.com)**
    - Node: **HTTP Request**
    - POST `https://api.upload-post.com/api/upload`
    - Multipart form-data:
      - `user`, `platform[] = tiktok`, `mediaUrl = {{$('Prepare Upload2').item.json.video_url}}`
      - captions/description fields from `Prepare Upload2`
    - Header: `Authorization: Apikey <UPLOAD_POST_KEY>`
    - Connect: `Download Video -> TikTok`.

22. **Add HTTP Request: Instagram (upload-post.com)**
    - Same as TikTok but `platform[] = instagram`, `user = YOUR_INSTAGRAM_USERNAME`.
    - Connect: `Download Video -> Instagram`.

23. **Add HTTP Request: Pinterest (upload-post.com)**
    - Same endpoint; `platform[] = pinterest`, `user`, `boardId`.
    - Optional: set node to “Continue On Fail” if desired.
    - Connect: `Download Video -> Pinterest`.

24. **Add Google Sheets: Update Sheet**
    - Node: **Google Sheets** (Update)
    - Match column: `topic`
    - Set `status = uploaded`, plus `video_id` and `youtube_url` using YouTube output `uploadId`.
    - Connect outputs:
      - **Recommended**: connect **only from YouTube** to avoid missing `uploadId` issues.
      - If you keep all four connections as in JSON, add logic to prevent overwriting when `uploadId` is absent.

25. **Add Telegram: Telegram2**
    - Node: **Telegram**
    - Credentials: Telegram bot token in n8n.
    - Chat ID: your chat/channel ID.
    - Message includes topic + YouTube Shorts URL with `uploadId`.
    - Connect: `Update Sheet -> Telegram2`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Requires API keys for Gemini, Kie.ai (Veo3), ImgBB, OpenAI, and upload-post.com; connect YouTube + Telegram in-node | Mentioned in the main sticky note instructions |
| Google Sheet must include columns: `topic`, `image_url`, `status` (and optionally `row_number`) | Mentioned in the main sticky note instructions |
| Replace all placeholders by searching for `YOUR_` | Mentioned in the main sticky note instructions |
| Key external endpoints used | `https://generativelanguage.googleapis.com/…`, `https://api.kie.ai/…`, `https://api.imgbb.com/…`, `https://api.upload-post.com/api/upload`, `https://youtube.com/shorts/{uploadId}` |