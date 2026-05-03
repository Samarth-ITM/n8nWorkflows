Turn an image into a cinematic video with NanoBanana Ultra, Kling, Blotato

https://n8nworkflows.xyz/workflows/turn-an-image-into-a-cinematic-video-with-nanobanana-ultra--kling--blotato-12769


# Turn an image into a cinematic video with NanoBanana Ultra, Kling, Blotato

## 1. Workflow Overview

**Purpose:** Convert a single source image stored in Google Drive into a **professional, multi-shot cinematic video** by:
1) generating a **mandatory 2×3 contact sheet (6 keyframes)** with **NanoBanana Ultra (AtlasCloud)**, then  
2) cropping that contact sheet into frames, generating **3 short start/end-frame clips** with **Kling (AtlasCloud)**, merging them into one MP4 via **fal.ai FFmpeg merge**, and publishing to **YouTube via Blotato**.

**Typical use cases**
- Fashion/editorial “cinematic camera move” clips built from one photo
- Batch production driven by a Google Sheet “status” pipeline
- Semi-automated publishing to social platforms (YouTube in this workflow)

### 1.1 Logical Blocks
1. **Block A — Step 1: Contact sheet generation (Manual entry point)**
   - Reads an image link from Google Sheets → downloads image from Google Drive → uploads to tmpfiles (public URL) → calls AtlasCloud NanoBanana Ultra → waits/polls → downloads PNG → uploads to Drive → updates Google Sheet.
2. **Block B — Step 2: Video creation & publishing (Scheduled entry point)**
   - On schedule reads rows → downloads contact sheet → crops into frames → uploads frames to tmpfiles → creates 3 Kling clips (frame-to-frame) → polls results → merges videos → updates sheet → uploads to Blotato → creates YouTube post.

**Entry points**
- Manual: **When clicking ‘Execute workflow’** (Step 1)
- Scheduled: **Schedule Trigger** (Step 2)

---

## 2. Block-by-Block Analysis

### Block A — Step 1: Create 2×3 Contact Sheet Image (Mandatory)

**Overview:** Pulls an input image from Drive (based on a Drive link stored in Sheets), makes it publicly accessible via tmpfiles, requests a 6-frame **2×3 contact sheet** from AtlasCloud NanoBanana Ultra, then stores the output back to Drive and updates the Sheet status.

**Nodes involved**
- When clicking ‘Execute workflow’
- Get url image
- Download image
- Build Public Image URL nano
- Edit Fields : contactSheetPrompt
- NanoBanana ULTRA: Contact Sheet
- Wait - nanobanana
- download image nano
- Download image PNG (binary)
- Upload file to google drive
- Update with new image
- Sticky Note
- Sticky Note1

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; starts Step 1 on demand.
- **Config:** No parameters.
- **Outputs:** Sends a single empty item into “Get url image”.
- **Failure modes:** None (except workflow not executed).

#### Node: Get url image
- **Type / role:** Google Sheets node; fetches the row(s) that contain the source image URL (`image_nanobanana`).
- **Config choices (interpreted):**
  - Document and sheet tab are placeholders (`Google Sheets Document ID`, `Sheet Tab Name`).
  - Operation is not explicitly shown in parameters; in this node type/version it commonly defaults to **Read/Get** rows.
- **Key fields used downstream:** `$json.image_nanobanana`
- **Connections:** Input from Manual Trigger → output to “Download image”.
- **Failure modes / edge cases:**
  - OAuth failure / expired token.
  - Wrong documentId/sheetName placeholders not replaced.
  - If multiple rows are returned, Step 1 will process each item unless constrained.
  - If `image_nanobanana` is missing, downstream regex extraction will fail.

#### Node: Download image
- **Type / role:** Google Drive (Download file); retrieves the image binary from Drive.
- **Config choices:**
  - **fileId expression:**  
    `{{ $json.image_nanobanana.match(/[?&]id=([a-zA-Z0-9_-]+)/)[1] }}`
  - Saves to binary property **`data`**, with filename `image.png`.
- **Connections:** From “Get url image” → to “Build Public Image URL nano”.
- **Failure modes / edge cases:**
  - If `image_nanobanana` is not a URL containing `?id=...` or `&id=...`, `.match(...)[1]` throws.
  - Drive permission issues (file not shared with the Drive credential).
  - Download size/timeouts.

#### Node: Build Public Image URL nano
- **Type / role:** HTTP Request; uploads the binary file to tmpfiles.org to obtain a public URL.
- **Config choices:**
  - POST `https://tmpfiles.org/api/v1/upload`
  - Multipart form-data; form field `file` uses **binary** field `data`.
  - Response format: JSON.
- **Output:** JSON like `{ data: { url: ... } }` (tmpfiles format).
- **Connections:** From “Download image” → to “Edit Fields : contactSheetPrompt”.
- **Failure modes / edge cases:**
  - tmpfiles rate limits or downtime.
  - If binary field name differs (must be `data`).
  - Large files might be rejected.

#### Node: Edit Fields : contactSheetPrompt
- **Type / role:** Set node; injects a long, strict prompt into `contactSheetPrompt`.
- **Config choices:**
  - Sets **one string field**: `contactSheetPrompt` containing detailed continuity rules + required 6-frame shot list + output format requirements.
- **Connections:** From tmpfiles upload → to “NanoBanana ULTRA: Contact Sheet”.
- **Edge cases:**
  - Prompt length can affect model cost/limits; if AtlasCloud enforces max prompt size, request may fail.

#### Node: NanoBanana ULTRA: Contact Sheet
- **Type / role:** HTTP Request; calls AtlasCloud image generation (NanoBanana Ultra edit-ultra) using the public tmpfiles URL.
- **Config choices:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
  - Header: `Authorization: Bearer <atlascloud Key>`
  - JSON body includes:
    - `model: "google/nano-banana-pro/edit-ultra"`
    - `aspect_ratio: "3:2"`
    - `resolution: "4k"`
    - `output_format: "png"`
    - `enable_sync_mode: false` (async)
    - `images: ["{{ $('Build Public Image URL nano').item.json.data.url.replace(/tmpfiles\\.org\\//, 'tmpfiles.org/dl/') }}"]`
      - Important: converts tmpfiles URL into direct-download `/dl/` form.
    - `prompt: "{{ $json.contactSheetPrompt }}"`
- **Connections:** From Set → to Wait.
- **Failure modes / edge cases:**
  - Invalid/expired AtlasCloud key (401).
  - AtlasCloud async job may take longer than 4 minutes; polling may return “processing”.
  - tmpfiles URL not accessible publicly (AtlasCloud can’t fetch it).
  - URL replacement assumes returned URL contains `tmpfiles.org/` exactly.

#### Node: Wait - nanobanana
- **Type / role:** Wait node; pauses 4 minutes for async image generation.
- **Config:** 4 minutes.
- **Connections:** From NanoBanana → to “download image nano”.
- **Edge cases:** If generation is slower, the next poll may not have outputs yet.

#### Node: download image nano
- **Type / role:** HTTP Request; polls AtlasCloud prediction endpoint to retrieve job status and outputs.
- **Config choices:**
  - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{ $json.data.id }}`
  - Headers: Authorization Bearer; Content-Type application/json
  - Response: JSON
- **Connections:** From Wait → to “Download image PNG (binary)”.
- **Failure modes / edge cases:**
  - If previous node didn’t return `data.id`, expression fails.
  - Job still running: `data.outputs` may be empty.

#### Node: Download image PNG (binary)
- **Type / role:** HTTP Request; downloads the resulting PNG as a binary file.
- **Config:** URL `{{ $json.data.outputs[0] }}`, response format **file**.
- **Connections:** → “Upload file to google drive”.
- **Failure modes / edge cases:**
  - `outputs[0]` missing.
  - Remote URL expired or requires auth.
  - Large binary download timeouts.

#### Node: Upload file to google drive
- **Type / role:** Google Drive; uploads the contact sheet PNG.
- **Config choices:**
  - File name: `{{ $json.data.id }}` (note: at this point `$json` is the binary download item; depending on n8n behavior you may not have `data.id` anymore—see edge case below)
  - `driveId` and `folderId`: placeholders.
- **Connections:** → “Update with new image”.
- **Failure modes / edge cases:**
  - **Potential bug:** after “Download image PNG (binary)”, the JSON may not contain the earlier `data.id` unless n8n keeps it. Safer is referencing `$('download image nano').item.json.data.id`.
  - Drive permission issues; folderId invalid.

#### Node: Update with new image
- **Type / role:** Google Sheets Update; writes outputs back and transitions status.
- **Config choices:**
  - Matching column: `image_nanobanana` (updates row(s) where that matches)
  - Writes:
    - `status = "contact_done"`
    - `image_atlas = {{ $('download image nano').item.json.data.outputs[0] }}`
    - `image_nanobanana = {{ $('Get url image').item.json.image_nanobanana }}`
    - `image_contactsheet = {{ $json.webContentLink }}` (Drive share link from upload node)
- **Connections:** End of Step 1.
- **Failure modes / edge cases:**
  - If multiple rows have same `image_nanobanana`, multiple updates can occur.
  - If Drive returns a different link field (varies by node version/settings), `webContentLink` may be missing.
  - Sheet schema mismatches or missing columns.

---

### Block B — Step 2: Video Creation & Publishing (Kling + Blotato)

**Overview:** On a schedule, reads the contact sheet URL from the sheet, downloads it, crops into frames, uploads frames to tmpfiles, generates 3 Kling videos (start/end frames), merges them, updates the sheet, then uploads/publishes via Blotato to YouTube.

**Nodes involved**
- Schedule Trigger
- Get row(s) in sheet
- Set Image URL
- Download Image
- Edit Image
- Crop Top Left / Crop Top Center / Crop Top Right / Crop Bottom Left
- Upload top left / Upload top center / Upload top right / Upload Bottom left
- Merge / Merge1 / Merge2
- Kling Generation / Kling Generation1 / Kling Generation2
- Wait video / Wait video1 / Wait video2
- download video kling / download video kling1 / download video kling2
- Merge3
- Merge 3 Videos
- Wait
- Update row in sheet
- Upload media (Blotato)
- Create post (Blotato)
- Sticky Note3
- Sticky Note1

#### Node: Schedule Trigger
- **Type / role:** Scheduled entry point for Step 2.
- **Config:** Interval rule is present but unspecified (`interval: [{}]`), meaning it may be incomplete or defaults in UI.
- **Connections:** → “Get row(s) in sheet”.
- **Edge cases:** Misconfigured schedule could run too often or never.

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets read; fetches row(s) that contain `image_contactsheet` and `image_atlas` for processing.
- **Config:** Document and tab placeholders.
- **Connections:** → “Set Image URL”.
- **Edge cases:**
  - No filtering is shown; without a filter (e.g., `status=contact_done`) it may process rows already completed.
  - If multiple rows returned, pipeline repeats per item.

#### Node: Set Image URL
- **Type / role:** Set node; normalizes the contact sheet link into `image_url`.
- **Config:** `image_url = {{ $json.image_contactsheet }}`
- **Connections:** → Download Image.
- **Edge cases:** `image_contactsheet` missing/empty.

#### Node: Download Image
- **Type / role:** HTTP Request; downloads the contact sheet image as binary.
- **Config:** `url = {{ $json.image_url }}`; response format **file**.
- **Connections:** → Edit Image (information).
- **Failure modes:** Link permission issues (Drive link not publicly accessible), timeouts.

#### Node: Edit Image
- **Type / role:** Edit Image; reads image metadata (width/height).
- **Config:** Operation `information` to output size info.
- **Connections:** Fans out to 4 crop nodes.
- **Edge cases:** Unsupported image format; missing binary property (expects default `data` unless configured).

#### Node: Crop Top Left
- **Type / role:** Edit Image crop; creates frame #1.
- **Config:**
  - width `floor(size.width / 3)`
  - height `floor(size.height / 2)`
  - position defaults to top-left (0,0)
- **Connections:** → Upload top left.
- **Edge cases:** If dimensions not divisible cleanly, last column/row might truncate slightly.

#### Node: Upload top left
- **Type / role:** HTTP Request; uploads cropped frame to tmpfiles.
- **Config:** POST tmpfiles upload; multipart from binary `data`; response JSON.
- **Connections:** → Merge (input 0).
- **Edge cases:** Same tmpfiles issues as above.

#### Node: Crop Top Center
- **Type / role:** Crop frame #2.
- **Config:** same width/height; `positionX = floor(size.width / 3)`
- **Connections:** → Upload top center → Merge (input 1) and Merge1 (input 0).
- **Edge cases:** Same cropping rounding issues.

#### Node: Upload top center
- **Type / role:** tmpfiles upload.
- **Output usage:** Provides `data.url` used as Kling end/start frames.
- **Connections:** To Merge and Merge1.

#### Node: Crop Top Right
- **Type / role:** Crop frame #3.
- **Config:** `positionX = floor(size.width * 2 / 3)`
- **Connections:** → Upload top right → Merge1 (input 1) and Merge2 (input 0).

#### Node: Upload top right
- **Type / role:** tmpfiles upload.
- **Connections:** To Merge1 and Merge2.

#### Node: Crop Bottom Left
- **Type / role:** Crop frame #4 (bottom-left).
- **Config:** `positionY = floor(size.height / 2)`
- **Connections:** → Upload Bottom left → Merge2 (input 1).

#### Node: Upload Bottom left
- **Type / role:** tmpfiles upload.
- **Connections:** To Merge2.

> Note: Only 4 frames are cropped/uploaded (top-left, top-center, top-right, bottom-left). The other 2 frames (bottom-center, bottom-right) are not used in this Step 2 implementation.

#### Node: Merge / Merge1 / Merge2
- **Type / role:** Merge nodes combining two items by position for each Kling job.
- **Config:** Mode `combine`, `combineByPosition`.
- **Connections:**
  - Merge → Kling Generation (uses top-left + top-center)
  - Merge1 → Kling Generation1 (top-center + top-right)
  - Merge2 → Kling Generation2 (top-right + bottom-left)
- **Edge cases:** If one upload fails, merge will not produce a combined item.

#### Node: Kling Generation (and 1, 2)
- **Type / role:** HTTP Request; starts Kling i2v start/end frame video generation (async).
- **Config choices:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateVideo`
  - Header: `Authorization: Bearer <atlascloud Key>`
  - Body params:
    - `model = "kwaivgi/kling-v2.1-i2v-pro/start-end-frame"`
    - `duration = 5` seconds
    - `guidance_scale = 0.5`
    - `image` = start frame tmpfiles URL converted to `/dl/`
    - `end_image` = end frame tmpfiles URL converted to `/dl/`
    - `prompt` constant: “The camera very slowly and smoothly lowers on a boom...”
    - `negative_prompt = example_value` (placeholder; likely should be replaced)
- **Connections:** Each → corresponding Wait video node.
- **Failure modes / edge cases:**
  - AtlasCloud auth/limits.
  - `negative_prompt` being meaningless; may reduce quality.
  - tmpfiles URL conversion assumption.
  - Async durations may exceed wait time.

#### Node: Wait video / Wait video1 / Wait video2
- **Type / role:** Wait 4 minutes for each Kling generation.
- **Connections:** → corresponding “download video klingX”.

#### Node: download video kling / kling1 / kling2
- **Type / role:** HTTP Request polling AtlasCloud prediction endpoint for video outputs.
- **Config:** GET prediction by `{{ $json.data.id }}`, returns JSON with `data.outputs[0]` expected to be a video URL.
- **Connections:** Each feeds into Merge3 (3-input combine).
- **Edge cases:** Outputs not ready; empty outputs.

#### Node: Merge3
- **Type / role:** Merge (3 inputs) combining the 3 video prediction results.
- **Config:** combineByPosition, `numberInputs: 3`.
- **Connections:** → Merge 3 Videos.

#### Node: Merge 3 Videos
- **Type / role:** HTTP Request; calls fal.ai ffmpeg merge to concatenate/merge the 3 video URLs.
- **Config choices:**
  - POST `https://fal.run/fal-ai/ffmpeg-api/merge-videos`
  - Header: `Authorization: key <fal Key>`, content-type json
  - JSON body:
    - `video_urls`: three AtlasCloud outputs
    - `output.format = "mp4"`
- **Connections:** → Wait (1 minute).
- **Failure modes / edge cases:**
  - fal key invalid, quotas.
  - If any video URL is not publicly accessible, merge fails.
  - Video codecs/resolution mismatch may cause ffmpeg merge errors.

#### Node: Wait
- **Type / role:** Wait 1 minute (likely to ensure fal output is ready/propagated).
- **Connections:** → Update row in sheet.

#### Node: Update row in sheet
- **Type / role:** Google Sheets Update; writes final video URL, marks done.
- **Config choices:**
  - Matching column: `image_atlas`
  - Writes:
    - `status = "video_done"`
    - `Final video = {{ $('Merge 3 Videos').item.json.video.url }}`
    - `image_atlas = {{ $('Get row(s) in sheet').first().json.image_atlas }}`
- **Connections:** → Upload media (Blotato)
- **Edge cases / potential bug:**
  - Using `first().json.image_atlas` can mismatch when processing multiple rows; it may always write using the first row’s atlas link rather than the current item’s.
  - If fal response schema differs, `video.url` may not exist.

#### Node: Upload media (Blotato)
- **Type / role:** Blotato node; uploads the final MP4 as a media resource.
- **Config:** `mediaUrl = {{ $json['Final video'] }}`, resource = `media`.
- **Connections:** → Create post.
- **Failure modes:** Blotato auth, unsupported URL, download failures.

#### Node: Create post (Blotato)
- **Type / role:** Blotato node; creates a YouTube post (upload) using the uploaded media URL.
- **Config choices:**
  - platform: `youtube`
  - accountId: fixed selection (`DR FIRASS (Dr. Firas)`, id 8047)
  - post text: `test`
  - title: `test`
  - privacyStatus: `private`
  - notifySubscribers: false
  - media URLs: `{{ $json.url }}` (output from Upload media)
- **Failure modes / edge cases:**
  - YouTube account not connected in Blotato.
  - Title/content placeholders not updated.
  - Rate limits / upload size constraints.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual entry point for Step 1 | — | Get url image | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Get url image | googleSheets | Fetch row(s) containing source image URL | When clicking ‘Execute workflow’ | Download image | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Download image | googleDrive | Download source image binary from Drive | Get url image | Build Public Image URL nano | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Build Public Image URL nano | httpRequest | Upload image to tmpfiles to get public URL | Download image | Edit Fields : contactSheetPrompt | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Edit Fields : contactSheetPrompt | set | Define NanoBanana prompt for contact sheet | Build Public Image URL nano | NanoBanana ULTRA: Contact Sheet | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| NanoBanana ULTRA: Contact Sheet | httpRequest | Start async NanoBanana Ultra contact sheet generation | Edit Fields : contactSheetPrompt | Wait - nanobanana | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Wait - nanobanana | wait | Pause before polling image result | NanoBanana ULTRA: Contact Sheet | download image nano | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| download image nano | httpRequest | Poll AtlasCloud prediction for image outputs | Wait - nanobanana | Download image PNG (binary) | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Download image PNG (binary) | httpRequest | Download generated PNG as binary | download image nano | Upload file to google drive | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Upload file to google drive | googleDrive | Upload contact sheet PNG to Drive | Download image PNG (binary) | Update with new image | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Update with new image | googleSheets | Update row: status/contactsheet/atlas links | Upload file to google drive | — | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Schedule Trigger | scheduleTrigger | Scheduled entry point for Step 2 | — | Get row(s) in sheet | ## Step 2 – Video Creation & Publishing (Kling + Blotato)\n\n#  📘 Documentation  \n- Access detailed setup instructions, API config, platform connection guides, and workflow customization tips:\n📎 https://automatisation.notion.site/Turn-Any-Image-into-a-Cinematic-Video-with-NanoBanana-Ultra-Kling-AI-2ea3d6550fd9809c9321e897b9763a28?pvs=73\n-  Credential name: `Google Sheets account`\n📎 https://docs.google.com/spreadsheets/d/130hio-ntnPCZbGzmp1R3ROHXSpKQBUC0I_iM0uQPPi4/copy |
| Get row(s) in sheet | googleSheets | Read rows for video processing | Schedule Trigger | Set Image URL | (same as above) |
| Set Image URL | set | Map sheet field to `image_url` | Get row(s) in sheet | Download Image | (same as above) |
| Download Image | httpRequest | Download contact sheet image as binary | Set Image URL | Edit Image | (same as above) |
| Edit Image | editImage | Extract image dimensions | Download Image | Crop Top Left; Crop Top Center; Crop Top Right; Crop Bottom Left | (same as above) |
| Crop Top Left | editImage | Crop frame 1 | Edit Image | Upload top left | (same as above) |
| Upload top left | httpRequest | Upload frame 1 to tmpfiles | Crop Top Left | Merge | (same as above) |
| Crop Top Center | editImage | Crop frame 2 | Edit Image | Upload top center | (same as above) |
| Upload top center | httpRequest | Upload frame 2 to tmpfiles | Crop Top Center | Merge; Merge1 | (same as above) |
| Crop Top Right | editImage | Crop frame 3 | Edit Image | Upload top right | (same as above) |
| Upload top right | httpRequest | Upload frame 3 to tmpfiles | Crop Top Right | Merge1; Merge2 | (same as above) |
| Crop Bottom Left | editImage | Crop frame 4 | Edit Image | Upload Bottom left | (same as above) |
| Upload Bottom left | httpRequest | Upload frame 4 to tmpfiles | Crop Bottom Left | Merge2 | (same as above) |
| Merge | merge | Combine frame1+frame2 inputs | Upload top left; Upload top center | Kling Generation | (same as above) |
| Kling Generation | httpRequest | Start Kling clip 1 (start/end frames) | Merge | Wait video | (same as above) |
| Wait video | wait | Pause before polling clip 1 | Kling Generation | download video kling | (same as above) |
| download video kling | httpRequest | Poll AtlasCloud for clip 1 URL | Wait video | Merge3 | (same as above) |
| Merge1 | merge | Combine frame2+frame3 inputs | Upload top center; Upload top right | Kling Generation1 | (same as above) |
| Kling Generation1 | httpRequest | Start Kling clip 2 | Merge1 | Wait video1 | (same as above) |
| Wait video1 | wait | Pause before polling clip 2 | Kling Generation1 | download video kling1 | (same as above) |
| download video kling1 | httpRequest | Poll AtlasCloud for clip 2 URL | Wait video1 | Merge3 | (same as above) |
| Merge2 | merge | Combine frame3+frame4 inputs | Upload top right; Upload Bottom left | Kling Generation2 | (same as above) |
| Kling Generation2 | httpRequest | Start Kling clip 3 | Merge2 | Wait video2 | (same as above) |
| Wait video2 | wait | Pause before polling clip 3 | Kling Generation2 | download video kling2 | (same as above) |
| download video kling2 | httpRequest | Poll AtlasCloud for clip 3 URL | Wait video2 | Merge3 | (same as above) |
| Merge3 | merge | Combine 3 polled video outputs | download video kling; download video kling1; download video kling2 | Merge 3 Videos | (same as above) |
| Merge 3 Videos | httpRequest | Merge/concatenate 3 clips via fal ffmpeg API | Merge3 | Wait | (same as above) |
| Wait | wait | Pause before sheet update | Merge 3 Videos | Update row in sheet | (same as above) |
| Update row in sheet | googleSheets | Update row: status + final video URL | Wait | Upload media | (same as above) |
| Upload media | blotato | Upload final MP4 to Blotato | Update row in sheet | Create post | (same as above) |
| Create post | blotato | Create YouTube post (private) | Upload media | — | (same as above) |
| Sticky Note | stickyNote | Comment: Step 1 header | — | — | ## Step 1 - Create 2×3 Contact Sheet Image (Mandatory) |
| Sticky Note3 | stickyNote | Comment: Step 2 header + links | — | — | ## Step 2 – Video Creation & Publishing (Kling + Blotato)\n\n#  📘 Documentation  \n📎 https://automatisation.notion.site/Turn-Any-Image-into-a-Cinematic-Video-with-NanoBanana-Ultra-Kling-AI-2ea3d6550fd9809c9321e897b9763a28?pvs=73\n\nCredential name: `Google Sheets account`\n📎 https://docs.google.com/spreadsheets/d/130hio-ntnPCZbGzmp1R3ROHXSpKQBUC0I_iM0uQPPi4/copy |
| Sticky Note1 | stickyNote | Long embedded description/setup notes | — | — | (Content describes whole workflow; includes AtlasCloud link and Blotato link) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
1. Create **Google Sheets OAuth2** credential named `Google Sheets account`.
2. Create **Google Drive OAuth2** credential named `Google Drive account`.
3. Create **Blotato API** credential named `Blotato account`.
4. Prepare secrets (as environment variables or placeholders):
   - **AtlasCloud API key** (used in HTTP headers as `Authorization: Bearer ...`)
   - **fal.ai key** (used as `Authorization: key ...`)

2) **Prepare Google Sheet**
5. Create/copy a sheet (link from sticky note):  
   https://docs.google.com/spreadsheets/d/130hio-ntnPCZbGzmp1R3ROHXSpKQBUC0I_iM0uQPPi4/copy
6. Ensure columns exist at least:
   - `status`, `image_nanobanana`, `image_contactsheet`, `image_atlas`, `Final video`
7. Ensure `image_nanobanana` contains a Drive URL with an `id` parameter (e.g. `...uc?id=FILE_ID&export=download`).

### Step 1 nodes (Manual)

8. Add **Manual Trigger** node: “When clicking ‘Execute workflow’”.
9. Add **Google Sheets** node “Get url image”
   - Set **Document ID** + **Sheet Tab Name**
   - Configure to read rows containing `image_nanobanana` (optionally filter by `status = nanobanana_done`).
   - Connect Manual Trigger → Get url image.
10. Add **Google Drive** node “Download image”
   - Operation: **Download**
   - File ID expression: `{{$json.image_nanobanana.match(/[?&]id=([a-zA-Z0-9_-]+)/)[1]}}`
   - Binary Property Name: `data`
   - Connect Get url image → Download image.
11. Add **HTTP Request** node “Build Public Image URL nano”
   - POST `https://tmpfiles.org/api/v1/upload`
   - Body: multipart-form-data; field `file` = Binary `data`
   - Response: JSON
   - Connect Download image → Build Public Image URL nano.
12. Add **Set** node “Edit Fields : contactSheetPrompt”
   - Add string field `contactSheetPrompt` with the provided long prompt.
   - Connect Build Public Image URL nano → Set node.
13. Add **HTTP Request** node “NanoBanana ULTRA: Contact Sheet”
   - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
   - Headers: `Authorization: Bearer <ATLAS_KEY>`
   - Body: JSON with:
     - model `google/nano-banana-pro/edit-ultra`
     - aspect_ratio `3:2`
     - resolution `4k`
     - output_format `png`
     - enable_sync_mode `false`
     - images array containing:  
       `{{$('Build Public Image URL nano').item.json.data.url.replace(/tmpfiles\.org\//,'tmpfiles.org/dl/')}}`
     - prompt `{{$json.contactSheetPrompt}}`
   - Connect Set node → NanoBanana.
14. Add **Wait** node “Wait - nanobanana” (4 minutes). Connect NanoBanana → Wait.
15. Add **HTTP Request** node “download image nano”
   - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{$json.data.id}}`
   - Headers: Authorization Bearer; Content-Type application/json
   - Response: JSON
   - Connect Wait → download image nano.
16. Add **HTTP Request** node “Download image PNG (binary)”
   - GET `{{$json.data.outputs[0]}}`
   - Response format: file
   - Connect download image nano → Download image PNG.
17. Add **Google Drive** node “Upload file to google drive”
   - Operation: Upload
   - Destination Drive ID + Folder ID
   - (Recommended) Name: `{{$('download image nano').item.json.data.id}}`
   - Connect Download image PNG → Upload.
18. Add **Google Sheets** node “Update with new image”
   - Operation: Update
   - Match column: `image_nanobanana`
   - Set:
     - `status = contact_done`
     - `image_atlas = {{$('download image nano').item.json.data.outputs[0]}}`
     - `image_contactsheet = {{$json.webContentLink}}` (from Drive upload output)
   - Connect Upload → Update.

### Step 2 nodes (Scheduled)

19. Add **Schedule Trigger** and configure interval (e.g., every 5 minutes).
20. Add **Google Sheets** node “Get row(s) in sheet”
   - Configure read rows; **recommended**: filter where `status = contact_done`.
   - Connect Schedule Trigger → Get row(s).
21. Add **Set** node “Set Image URL”: `image_url = {{$json.image_contactsheet}}`. Connect.
22. Add **HTTP Request** “Download Image”
   - URL `{{$json.image_url}}`
   - Response format: file
23. Add **Edit Image** “Edit Image” operation `information`.
24. Add 4 **Edit Image** crop nodes:
   - “Crop Top Left”: width `floor(width/3)`, height `floor(height/2)`
   - “Crop Top Center”: same + positionX `floor(width/3)`
   - “Crop Top Right”: same + positionX `floor(width*2/3)`
   - “Crop Bottom Left”: same + positionY `floor(height/2)`
   - Connect “Edit Image” → all crop nodes.
25. After each crop, add an **HTTP Request** tmpfiles upload node (top left/center/right/bottom left), same config as Step 1 tmpfiles upload.
26. Add three **Merge** nodes (combineByPosition):
   - Merge: (top left + top center)
   - Merge1: (top center + top right)
   - Merge2: (top right + bottom left)
27. Add three **HTTP Request** nodes for Kling generation (Kling Generation / 1 / 2)
   - POST `https://api.atlascloud.ai/api/v1/model/generateVideo`
   - Header Authorization Bearer
   - Body parameters: model, duration=5, guidance_scale=0.5, prompt, negative_prompt
   - Use `/dl/` URL conversion for `image` and `end_image`.
28. Add three **Wait** nodes (4 minutes each), then three **HTTP Request** polling nodes to `/model/prediction/{id}`.
29. Add **Merge3** with 3 inputs (combineByPosition).
30. Add **HTTP Request** “Merge 3 Videos”
   - POST `https://fal.run/fal-ai/ffmpeg-api/merge-videos`
   - Header: `Authorization: key <FAL_KEY>`
   - JSON: `video_urls` = the three `data.outputs[0]`, output mp4.
31. Add **Wait** (1 minute).
32. Add **Google Sheets** “Update row in sheet”
   - Match column: `image_atlas`
   - Set `status=video_done`, `Final video={{$('Merge 3 Videos').item.json.video.url}}`
33. Add **Blotato** “Upload media”
   - resource: media
   - mediaUrl: `{{$json['Final video']}}`
34. Add **Blotato** “Create post”
   - platform youtube, choose accountId
   - postContentMediaUrls: `{{$json.url}}`
   - set title/text/privacy as needed

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Full documentation (setup instructions, API config, platform connection guides, customization tips) | https://automatisation.notion.site/Turn-Any-Image-into-a-Cinematic-Video-with-NanoBanana-Ultra-Kling-AI-2ea3d6550fd9809c9321e897b9763a28?pvs=73 |
| Copy the Google Sheet used by the workflow | https://docs.google.com/spreadsheets/d/130hio-ntnPCZbGzmp1R3ROHXSpKQBUC0I_iM0uQPPi4/copy |
| AtlasCloud reference link mentioned in notes | https://www.atlascloud.ai?ref=8QKPJE |
| Blotato reference link mentioned in notes | https://blotato.com/?ref=firas |

_Disclaimer (from user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques._