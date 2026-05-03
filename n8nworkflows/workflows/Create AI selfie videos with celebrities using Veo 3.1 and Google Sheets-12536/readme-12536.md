Create AI selfie videos with celebrities using Veo 3.1 and Google Sheets

https://n8nworkflows.xyz/workflows/create-ai-selfie-videos-with-celebrities-using-veo-3-1-and-google-sheets-12536


# Create AI selfie videos with celebrities using Veo 3.1 and Google Sheets

## 1. Workflow Overview

**Purpose:** Generate a sequence of AI “selfie-style” video clips (e.g., with celebrities/public figures) using **fal.ai Veo 3.1** from prompts and images stored in **Google Sheets**, write each resulting clip URL back to the sheet, then **merge** all completed clips into one final video using **fal.ai FFmpeg API**, and optionally upload/distribute the final video to **Google Drive**, **YouTube (via upload-post.com)**, and **Postiz**.

**Typical use cases**
- Batch creation of short-form “viral” clips from structured prompt rows in a spreadsheet.
- Automated clip stitching (linear compilation) and distribution to multiple platforms.

### Logical blocks
1. **1.1 Manual Start & Sheet Input**
2. **1.2 Clip Generation (row-by-row) + Polling**
3. **1.3 Writeback to Sheet**
4. **1.4 Collect Clip URLs + Merge Video + Polling**
5. **1.5 Fetch Final File + Upload/Distribution**
6. **1.6 Documentation / Operator Notes (Sticky Notes)**

---

## 2. Block-by-Block Analysis

### 2.1 Manual Start & Sheet Input

**Overview:** Starts the workflow manually, reads all candidate rows from Google Sheets that are intended to be processed, and initiates a batch loop.

**Nodes involved**
- When clicking ‘Execute workflow’
- Get prompts
- Loop Over Items

#### Node: **When clicking ‘Execute workflow’**
- **Type / role:** Manual Trigger (entry point for testing or manual runs).
- **Config:** No parameters.
- **Outputs:** Triggers `Get prompts`.
- **Failure/edge cases:** None (only runs when executed manually).

#### Node: **Get prompts**
- **Type / role:** Google Sheets node; fetches rows to process.
- **Config choices (interpreted):**
  - Reads from Spreadsheet: **“Create, Extend and merge video”** (documentId `1QLZ...`), sheet **Foglio1** (`gid=0`).
  - Uses a filter UI with lookup column **MERGE** (value not explicitly set here—likely intended to select rows needing generation/merge).
- **Key fields produced:** Row JSON includes columns such as `START`, `LAST`, `PROMPT`, `DURATION`, `VIDEO URL`, `MERGE`, plus `row_number`.
- **Connections:** `When clicking…` → `Get prompts` → `Loop Over Items`.
- **Failure/edge cases:**
  - OAuth expiry/permission errors.
  - Filter misconfiguration could return zero rows or unintended rows.

#### Node: **Loop Over Items**
- **Type / role:** Split In Batches; iterates rows and also gates transitions to merging stage.
- **Config:** Default batching behavior (batch size not specified; relies on node defaults).
- **Connections (important):**
  - Output 1 goes to `Get prompt` (to generate each clip).
  - Output 2 goes to `Get Video Url to merge` (merge stage kickoff).
- **Edge cases:**
  - If batch size/default behavior is not what you expect, you may process too many items at once or not loop properly.
  - If `Update video url` doesn’t route back correctly, loop can stall.

---

### 2.2 Clip Generation (row-by-row) + Polling (fal.ai Veo 3.1)

**Overview:** For each sheet row, retrieves row data, formats parameters, requests Veo generation (async), then polls until completion.

**Nodes involved**
- Get prompt
- Set params
- Generate clip
- Wait 60 sec.
- Get status clip
- Completed?
- Get Clip Url

#### Node: **Get prompt**
- **Type / role:** Google Sheets node; fetches row(s) to generate from.
- **Config choices:**
  - Same spreadsheet and sheet (`gid=0`).
  - Filter UI lookup column **VIDEO URL** (value not provided here—likely intended to fetch rows missing URL or matching current row context).
- **Connections:** `Loop Over Items` → `Get prompt` → `Set params`.
- **Edge cases:**
  - If filter doesn’t bind to current loop item, it may return wrong rows or none.
  - If multiple rows returned, downstream nodes generate multiple clips unexpectedly.

#### Node: **Set params**
- **Type / role:** Set node; normalizes fields required by the Veo API request.
- **Config choices:**
  - Creates fields:
    - `prompt` = `{{$json.PROMPT}}`
    - `duration` = `{{$json.DURATION}}` (later appended with `"s"`)
    - `first_image` = `{{$json.START}}`
    - `last_image` = `{{$json.LAST}}`
- **Connections:** `Get prompt` → `Set params` → `Generate clip`.
- **Edge cases:**
  - Missing/blank sheet cells produce invalid API payload.
  - Non-numeric duration may cause API validation failures.

#### Node: **Generate clip**
- **Type / role:** HTTP Request; submits Veo 3.1 “first-last-frame-to-video” generation job (async queue).
- **Endpoint:** `POST https://queue.fal.run/fal-ai/veo3.1/fast/first-last-frame-to-video`
- **Body (interpreted):**
  - `prompt`: from `Set params`
  - `aspect_ratio`: auto
  - `duration`: `"{{duration}}s"`
  - `resolution`: 720p
  - `generate_audio`: false
  - `first_frame_url`: start image URL
  - `last_frame_url`: last image URL
- **Auth:** Generic credential type → **HTTP Header Auth** (Fal.run API). Header includes `Content-Type: application/json`.
  - Note: The node also references an **HTTP Bearer Auth** credential (“Runpods”), but the active config is header auth via Fal.run.
- **Outputs:** Expected to return `request_id`.
- **Connections:** `Generate clip` → `Wait 60 sec.`.
- **Failure/edge cases:**
  - 401/403 invalid API key.
  - 400 payload issues (bad URLs, duration, unsupported image format).
  - Queue timeouts or rate limits.

#### Node: **Wait 60 sec.**
- **Type / role:** Wait; delays before polling.
- **Config:** Wait 60 seconds.
- **Connections:** `Generate clip` → `Wait 60 sec.` → `Get status clip`.
- **Edge cases:** If jobs take longer, this forces repeated polling cycles.

#### Node: **Get status clip**
- **Type / role:** HTTP Request; polls job status.
- **Endpoint:** `GET https://queue.fal.run/fal-ai/veo3.1/requests/{{ request_id }}/status`
  - `request_id` is taken from `$('Generate clip').item.json.request_id`.
- **Auth:** Fal.run header auth (and also references Runpods bearer credential similarly).
- **Connections:** `Get status clip` → `Completed?`.
- **Failure/edge cases:** If `request_id` is missing (Generate failed), expression will error.

#### Node: **Completed?**
- **Type / role:** IF; checks if generation finished.
- **Condition:** `{{$json.status}} == "COMPLETED"`
- **Connections:**
  - **True** → `Get Clip Url`
  - **False** → `Wait 60 sec.` (poll again)
- **Edge cases:**
  - If status is `"FAILED"` or `"CANCELED"`, it will loop forever unless handled. Consider adding additional branches for failure statuses.

#### Node: **Get Clip Url**
- **Type / role:** HTTP Request; fetches completed job payload (including final video URL and metadata).
- **Endpoint:** `GET https://queue.fal.run/fal-ai/veo3.1/requests/{{ request_id }}`
- **Outputs:** Expected structure includes `video.url` and `video.file_name`.
- **Connections:** `Get Clip Url` → `Update video url`.
- **Failure/edge cases:** Same as polling; if request is not ready, may return incomplete payload.

---

### 2.3 Writeback to Sheet

**Overview:** Writes the generated clip URL back into the Google Sheet and marks the row as ready for merge, then returns to the loop.

**Nodes involved**
- Update video url

#### Node: **Update video url**
- **Type / role:** Google Sheets node; updates the same row with output video URL.
- **Config choices (interpreted):**
  - Operation: **Update**
  - Matching column: `row_number` (read-only row identifier from Sheets node)
  - Sets:
    - `MERGE` = `"x"`
    - `VIDEO URL` = `{{ $('Completed?').item.json.video.url }}`
    - `row_number` = `{{ $('Loop Over Items').item.json.row_number }}`
  - Note: It references sheetName cached as another doc URL (`1MisBkHc...`) but documentId is `1QLZ...`. In n8n this can happen due to cached UI metadata; ensure the actual documentId+sheet is correct.
- **Connections:** `Get Clip Url` → `Update video url` → back to `Loop Over Items`.
- **Failure/edge cases:**
  - If `Completed?` branch item context doesn’t contain `video.url`, expression fails.
  - If `row_number` is missing or mismatched, the update targets the wrong row.
  - Sheet protection / permissions can block updates.

---

### 2.4 Collect Clip URLs + Merge Video + Polling (fal.ai FFmpeg)

**Overview:** After clip generation loop ends, the workflow reads all rows marked for merge, builds a JSON array of video URLs, submits a merge job, then polls until completed.

**Nodes involved**
- Get Video Url to merge
- Set VideoUrls Json
- Merge Videos
- Wait 30 sec.
- Get status2
- Completed?2
- Get final video url

#### Node: **Get Video Url to merge**
- **Type / role:** Google Sheets node; selects rows to merge.
- **Config:**
  - Filter: `MERGE` equals `"x"`.
- **Connections:** `Loop Over Items` (second output) → `Get Video Url to merge` → `Set VideoUrls Json`.
- **Edge cases:** If MERGE markers aren’t set (or were set erroneously), merge input will be empty or wrong.

#### Node: **Set VideoUrls Json**
- **Type / role:** Code node; collects all `VIDEO URL` values into one array.
- **Logic (interpreted):**
  - `videoUrls = $input.all().map(item => item.json["VIDEO URL"]);`
  - Returns `{ videos: videoUrls }`
- **Connections:** `Get Video Url to merge` → `Set VideoUrls Json` → `Merge Videos`.
- **Edge cases:**
  - Blank/invalid URLs will be included; FFmpeg merge may fail.
  - If Sheets returns headers/empty rows, you may get `null` entries.

#### Node: **Merge Videos**
- **Type / role:** HTTP Request; submits merge job to fal.ai ffmpeg-api (async).
- **Endpoint:** `POST https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
- **Body:**
  - `video_urls`: `JSON.stringify($json.videos)` (array serialized)
  - `target_fps`: 24
- **Auth:** Fal.run header auth; Content-Type JSON.
- **Outputs:** `request_id` for merge job.
- **Connections:** `Merge Videos` → `Wait 30 sec.`
- **Failure/edge cases:**
  - If `video_urls` is empty, API may reject or produce empty output.
  - Large merges can take longer; polling interval may need adjustment.

#### Node: **Wait 30 sec.**
- **Type / role:** Wait; delays before polling merge status.
- **Config:** 30 seconds.
- **Connections:** `Wait 30 sec.` → `Get status2`.

#### Node: **Get status2**
- **Type / role:** HTTP Request; polls merge job status.
- **Endpoint:** `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Merge Videos').item.json.request_id }}/status`
- **Auth:** Fal.run header auth; query param contains `Content-Type: application/json` (unusual as query parameter; typically should be a header).
- **Connections:** `Get status2` → `Completed?2`.
- **Edge cases:**
  - If `request_id` is missing, expression fails.
  - Misplaced Content-Type may not break but is not standard.

#### Node: **Completed?2**
- **Type / role:** IF; checks merge completion.
- **Condition:** `{{$json.status}} == "COMPLETED"`
- **Connections:**
  - **True** → `Get final video url`
  - **False** → `Wait 30 sec.` (poll again)
- **Edge cases:**
  - Same infinite-loop risk if status becomes FAILED; consider handling failure statuses.

#### Node: **Get final video url**
- **Type / role:** HTTP Request; fetches final merged video metadata (including final URL).
- **Endpoint:** `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
  - Uses `request_id` from prior node output (status or merge response).
- **Auth:** Fal.run header auth.
- **Connections:** `Get final video url` → `Get final video file`.
- **Edge cases:** If the returned structure differs, downstream `$json.video.url` may be missing.

---

### 2.5 Fetch Final File + Upload/Distribution

**Overview:** Downloads the merged video binary and sends it to storage/distribution services.

**Nodes involved**
- Get final video file
- Upload Video
- Upload to Youtube
- Upload to Postiz
- Upload to Social

#### Node: **Get final video file**
- **Type / role:** HTTP Request; downloads the final video.
- **Endpoint:** `GET {{$json.video.url}}`
- **Output:** Expected to be binary data in field `data` (n8n HTTP request typically stores binary when configured; here options are empty—ensure “Response Format”/binary settings are correct in your n8n version).
- **Connections:** Sends output to:
  - `Upload Video` (Google Drive)
  - `Upload to Youtube` (upload-post.com)
  - `Upload to Postiz`
- **Failure/edge cases:**
  - Large files can time out; adjust node timeout / download settings.
  - If response is not treated as binary, upload nodes expecting binary will fail.

#### Node: **Upload Video**
- **Type / role:** Google Drive; uploads final video to a folder.
- **Config:**
  - File name: `{{$now.format('yyyyLLddHHmmss')}}-{{ $('Get Clip Url').item.json.video.file_name }}`
  - Drive: My Drive
  - Folder: “Fal.run” (`1aHRwLW...`)
- **Credentials:** Google Drive OAuth2.
- **Connections:** Receives from `Get final video file`. No outgoing connections.
- **Edge cases:**
  - Uses `Get Clip Url`’s `video.file_name` (a clip file name) to name the final merged file; this may be misleading or fail if item context is not available at this stage. Prefer using `Get final video url` metadata if available.

#### Node: **Upload to Youtube**
- **Type / role:** HTTP Request; sends multipart upload to upload-post.com API for YouTube publishing.
- **Endpoint:** `POST https://api.upload-post.com/api/upload`
- **Multipart body:**
  - `title` = `XXX` (placeholder)
  - `user` = `YOUR_USERNAME` (placeholder)
  - `platform[]` = `youtube`
  - `video` = binary form field from input binary `data`
- **Auth:** Generic credential type → HTTP Header Auth (credential name shown as “Youtube Transcript Extractor API 1”, but used here for upload-post).
- **Connections:** Receives from `Get final video file`. No outgoing.
- **Failure/edge cases:**
  - Wrong API key/header, invalid user, or missing YouTube linkage in upload-post account.
  - If binary field name is not `data`, upload will be empty.

#### Node: **Upload to Postiz**
- **Type / role:** HTTP Request; uploads the file to Postiz storage first.
- **Endpoint:** `POST https://api.postiz.com/public/v1/upload`
- **Multipart body:**
  - `file` = binary from `data`
- **Auth:** HTTP Header Auth credential “Postiz”.
- **Connections:** `Upload to Postiz` → `Upload to Social`.
- **Edge cases:** Same binary concerns; auth/plan limitations on Postiz.

#### Node: **Upload to Social**
- **Type / role:** Postiz node; creates/schedules a social post using uploaded media.
- **Config (interpreted):**
  - `date`: now in `yyyy-LL-ddTHH:ii:ss`
  - `posts.post[0].value.contentItem[0]`:
    - Uses uploaded file `id` and `path` from previous node output
    - `content` = `XXX` (placeholder caption)
  - `integrationId` = `XXX` (placeholder; should be your channel/account integration ID)
  - `shortLink`: true
- **Credentials:** Postiz API credential.
- **Connections:** Receives from `Upload to Postiz`.
- **Failure/edge cases:**
  - Wrong/missing integrationId.
  - Post formatting schema mismatches if Postiz API changes.

---

### 2.6 Documentation / Operator Notes (Sticky Notes)

**Overview:** Sticky notes provide required manual configuration guidance and external links.

**Nodes involved**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9

These nodes do not affect execution but contain critical setup instructions and links.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get prompts |  |
| Get prompts | Google Sheets | Read rows to process | When clicking ‘Execute workflow’ | Loop Over Items | ## STEP 1 - Prepare the Sheet<br>- Clone [this sheet](https://docs.google.com/spreadsheets/d/1QLZmenUJ-vQAw2UzCe-zKTeWU9ybC1aVfi4ThVq3aUg/edit?usp=sharing) and fill the columns START, LAST, PROMPT and DURATION as in the current sheet<br><br>**VERY IMPORTANT:** the image in the **LAST** column must be the same as the image in the **START** column of the following row! |
| Loop Over Items | Split In Batches | Iterate rows; then proceed to merge stage | Get prompts; Update video url | Get prompt; Get Video Url to merge |  |
| Get prompt | Google Sheets | Fetch row data for generation | Loop Over Items | Set params | ## STEP 2 - Generate videos based on the provided prompt<br>For each row in the sheet, the workflow calls the fal.ai VEO 3.1 API to generate a video clip based on the provided prompt, start image, end image, and duration. The clip is created asynchronously, so the workflow polls the API for status until completion.<br><br>Create an account [here](https://fal.ai/) and obtain API KEY.<br>In the HTTP's nodes set "Header Auth" and set:<br>- Name: "Authorization"<br>- Value: "Key YOURAPIKEY" |
| Set params | Set | Map sheet columns to API fields | Get prompt | Generate clip | ## STEP 2 - Generate videos based on the provided prompt<br>…(fal.ai key instructions as above)… |
| Generate clip | HTTP Request | Submit Veo 3.1 generation job | Set params | Wait 60 sec. | ## STEP 2 - Generate videos based on the provided prompt<br>…(fal.ai key instructions as above)… |
| Wait 60 sec. | Wait | Delay before polling Veo job | Generate clip; Completed? (false) | Get status clip | ## STEP 2 - Generate videos based on the provided prompt<br>…(fal.ai key instructions as above)… |
| Get status clip | HTTP Request | Poll Veo job status | Wait 60 sec. | Completed? | ## STEP 2 - Generate videos based on the provided prompt<br>…(fal.ai key instructions as above)… |
| Completed? | IF | Branch on Veo completion | Get status clip | Get Clip Url (true); Wait 60 sec. (false) | ## STEP 2 - Generate videos based on the provided prompt<br>…(fal.ai key instructions as above)… |
| Get Clip Url | HTTP Request | Fetch Veo result incl. video URL | Completed? (true) | Update video url | ## STEP 2 - Generate videos based on the provided prompt<br>…(fal.ai key instructions as above)… |
| Update video url | Google Sheets | Write clip URL + mark MERGE=x | Get Clip Url | Loop Over Items | ## STEP 3 - Update Sheet<br>Update sheet with the generated video url |
| Get Video Url to merge | Google Sheets | Read all MERGE=x rows for merging | Loop Over Items (second output) | Set VideoUrls Json | ## STEP 4 - Merge videos<br>Merge the individual clips into a single, linear video using Fal AI’s ffmpeg API.<br><br>In the HTTP's nodes set "Header Auth" and set:<br>- Name: "Authorization"<br>- Value: "Key YOURAPIKEY" |
| Set VideoUrls Json | Code | Build `{videos:[...]}` array from rows | Get Video Url to merge | Merge Videos | ## STEP 4 - Merge videos<br>…(fal.ai key instructions as above)… |
| Merge Videos | HTTP Request | Submit FFmpeg merge job | Set VideoUrls Json | Wait 30 sec. | ## STEP 4 - Merge videos<br>…(fal.ai key instructions as above)… |
| Wait 30 sec. | Wait | Delay before polling merge status | Merge Videos; Completed?2 (false) | Get status2 | ## STEP 4 - Merge videos<br>…(fal.ai key instructions as above)… |
| Get status2 | HTTP Request | Poll merge status | Wait 30 sec. | Completed?2 | ## STEP 4 - Merge videos<br>…(fal.ai key instructions as above)… |
| Completed?2 | IF | Branch on merge completion | Get status2 | Get final video url (true); Wait 30 sec. (false) | ## STEP 4 - Merge videos<br>…(fal.ai key instructions as above)… |
| Get final video url | HTTP Request | Fetch final merged video metadata | Completed?2 (true) | Get final video file | ## STEP 4 - Merge videos<br>…(fal.ai key instructions as above)… |
| Get final video file | HTTP Request | Download final video file | Get final video url | Upload Video; Upload to Youtube; Upload to Postiz | ## STEP 5 - Upload video<br>Upload video to Google Drive and social media platforms |
| Upload Video | Google Drive | Upload final video to Drive folder | Get final video file | — | Upload final video to Google Drive |
| Upload to Youtube | HTTP Request | Upload to YouTube via upload-post.com | Get final video file | — | Set YOUR_USERNAME and TITLE for [Upload-Post]((https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)) |
| Upload to Postiz | HTTP Request | Upload media file to Postiz | Get final video file | Upload to Social | Set Channel_ID and TITLE for [Postiz](https://affiliate.postiz.com/n3witalia) (TikTok, Instagram, Facebook, X, Youtube) |
| Upload to Social | Postiz | Create/schedule social post with uploaded media | Upload to Postiz | — | Set Channel_ID and TITLE for [Postiz](https://affiliate.postiz.com/n3witalia) (TikTok, Instagram, Facebook, X, Youtube) |
| Sticky Note | Sticky Note | Comment node | — | — | Set YOUR_USERNAME and TITLE for [Upload-Post]((https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)) |
| Sticky Note1 | Sticky Note | Comment node | — | — | Set Channel_ID and TITLE for [Postiz](https://affiliate.postiz.com/n3witalia) (TikTok, Instagram, Facebook, X, Youtube) |
| Sticky Note2 | Sticky Note | Comment node | — | — | Upload final video to Google Drive |
| Sticky Note3 | Sticky Note | Comment node | — | — | ## Create Viral Selfie Videos With Celebrities Using Veo 3.1 & Share Everywhere<br>…(full note content includes workflow explanation and setup steps)… |
| Sticky Note4 | Sticky Note | Comment node | — | — | ## STEP 1 - Prepare the Sheet<br>- Clone [this sheet](https://docs.google.com/spreadsheets/d/1QLZmenUJ-vQAw2UzCe-zKTeWU9ybC1aVfi4ThVq3aUg/edit?usp=sharing)… |
| Sticky Note5 | Sticky Note | Comment node | — | — | ## STEP 2 - Generate videos…<br>Create an account [here](https://fal.ai/)… |
| Sticky Note6 | Sticky Note | Comment node | — | — | ## STEP 3 - Update Sheet<br>Update sheet with the generated video url |
| Sticky Note7 | Sticky Note | Comment node | — | — | ## STEP 4 - Merge videos<br>…(fal.ai key instructions)… |
| Sticky Note8 | Sticky Note | Comment node | — | — | ## STEP 5 - Upload video<br>Upload video to Google Drive and social media platforms |
| Sticky Note9 | Sticky Note | Comment node | — | — | ## MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia)… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Create Viral Selfie Videos With Celebrities Using Veo 3.1*.
2. Add **Manual Trigger** node: “When clicking ‘Execute workflow’”.
3. Add **Google Sheets** node: “Get prompts”
   - Credential: Google Sheets OAuth2 (authorize).
   - Select your cloned spreadsheet and sheet (Foglio1).
   - Configure a filter appropriate to your logic (commonly: rows where `VIDEO URL` is empty, or `MERGE` not set).
   - Ensure the output includes `row_number`.
4. Add **Split In Batches** node: “Loop Over Items”
   - Connect: Manual Trigger → Get prompts → Loop Over Items.
   - Keep default, or set batch size (e.g., 1) if you want strict sequential generation.
5. Add **Google Sheets** node: “Get prompt”
   - Use the same sheet.
   - Configure filter so it returns the *current* row (recommended approach: don’t re-query; instead, remove this node and use the loop item directly. If you keep it, bind the filter to `row_number` from `Loop Over Items`).
6. Add **Set** node: “Set params”
   - Create fields: `prompt`, `duration`, `first_image`, `last_image` from the sheet columns.
7. Create **Fal.run API credential**
   - Use **HTTP Header Auth** credential in n8n.
   - Header name: `Authorization`
   - Header value: `Key YOUR_FAL_API_KEY`
8. Add **HTTP Request** node: “Generate clip”
   - Method: POST
   - URL: `https://queue.fal.run/fal-ai/veo3.1/fast/first-last-frame-to-video`
   - Send JSON body with `prompt`, `duration` + “s”, `first_frame_url`, `last_frame_url`, plus your chosen settings.
   - Header: `Content-Type: application/json`
   - Auth: use Fal.run header auth credential.
9. Add **Wait** node: “Wait 60 sec.” and connect from “Generate clip”.
10. Add **HTTP Request** node: “Get status clip”
    - URL: `https://queue.fal.run/fal-ai/veo3.1/requests/{{request_id}}/status`
    - Use expression referencing “Generate clip” output `request_id`.
11. Add **IF** node: “Completed?”
    - Condition: `status` equals `COMPLETED`.
    - True → next step; False → back to “Wait 60 sec.” (poll loop).
12. Add **HTTP Request** node: “Get Clip Url”
    - URL: `https://queue.fal.run/fal-ai/veo3.1/requests/{{request_id}}`
13. Add **Google Sheets (Update)** node: “Update video url”
    - Match on `row_number` from the loop item.
    - Set:
      - `VIDEO URL` to the returned `video.url`
      - `MERGE` to `x`
    - Connect: Get Clip Url → Update video url → Loop Over Items (to continue batches).
14. **Merge stage input**
    - Add Google Sheets node: “Get Video Url to merge”
      - Filter: `MERGE` equals `x`.
      - Connect from the **second output** of Split In Batches (“no items left / done” path).
15. Add **Code** node: “Set VideoUrls Json”
    - Build `videos` array from all incoming items’ `VIDEO URL`.
16. Add **HTTP Request** node: “Merge Videos”
    - POST `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
    - JSON body: `video_urls` (array), `target_fps`.
    - Auth: Fal.run header auth.
17. Add **Wait** node: “Wait 30 sec.”
18. Add **HTTP Request** node: “Get status2”
    - GET `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{request_id}}/status`
19. Add **IF** node: “Completed?2”
    - `status == COMPLETED`
    - False → back to “Wait 30 sec.”
20. Add **HTTP Request** node: “Get final video url”
    - GET `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{request_id}}`
21. Add **HTTP Request** node: “Get final video file”
    - GET `{{$json.video.url}}`
    - Configure response as **binary** (so downstream multipart uploads can use the binary property, typically named `data`).
22. Add **Google Drive** node: “Upload Video”
    - Operation: Upload
    - Binary property: `data`
    - Choose destination folder.
23. (Optional) Add **HTTP Request** node: “Upload to Youtube” (upload-post.com)
    - Multipart form-data: `video` from binary `data`, plus `title`, `user`, `platform[]`.
    - Configure upload-post API key via HTTP Header Auth.
24. (Optional) Add **HTTP Request** node: “Upload to Postiz”
    - Multipart: `file` from binary `data`
    - Auth: Postiz header auth.
25. (Optional) Add **Postiz** node: “Upload to Social”
    - Use uploaded `id` and `path`.
    - Set `integrationId` to your connected channel ID and set the post content.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Clone the starter sheet and ensure column continuity: LAST image of row N must equal START image of row N+1 | https://docs.google.com/spreadsheets/d/1QLZmenUJ-vQAw2UzCe-zKTeWU9ybC1aVfi4ThVq3aUg/edit?usp=sharing |
| fal.ai account + API key required; set Header Auth `Authorization: Key YOURAPIKEY` | https://fal.ai/ |
| Upload-post configuration requires setting YOUR_USERNAME and TITLE | [Upload-Post](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) |
| Postiz publishing requires Channel/Integration ID and content | https://affiliate.postiz.com/n3witalia |
| YouTube channel link included in workflow notes | https://youtube.com/@n3witalia |

Disclaimer (as provided): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…*