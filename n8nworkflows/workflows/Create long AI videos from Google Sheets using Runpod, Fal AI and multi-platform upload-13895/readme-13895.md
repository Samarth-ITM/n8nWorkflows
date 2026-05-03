Create long AI videos from Google Sheets using Runpod, Fal AI and multi-platform upload

https://n8nworkflows.xyz/workflows/create-long-ai-videos-from-google-sheets-using-runpod--fal-ai-and-multi-platform-upload-13895


# Create long AI videos from Google Sheets using Runpod, Fal AI and multi-platform upload

# 1. Workflow Overview

This workflow, titled **“Automated Long Video Creator”**, builds a long AI-generated video from scene definitions stored in **Google Sheets**, then distributes the merged result to **Google Drive**, **YouTube**, and **Postiz/social platforms**.

Its main use case is **batch creation of multi-scene long-form videos** where each row in a spreadsheet defines:
- a starting image,
- a prompt,
- a clip duration,
- and whether that clip should be included in the final merge.

The workflow also maintains **visual continuity** between scenes by extracting the **last frame** from each generated clip and writing it into the next row’s `START` field.

## 1.1 Input Reception and Scene Selection
The workflow starts manually, reads candidate rows from Google Sheets, and iterates through them one by one.

## 1.2 Per-Scene Video Generation
For each row, it reads the current scene data, formats the prompt/image/duration payload, sends it to **RunPod WAN 2.5**, and polls until the clip is completed.

## 1.3 Clip Persistence and Continuity Frame Extraction
Once a clip is ready, the workflow writes the generated video URL back to Google Sheets, then calls **Fal AI FFmpeg** to extract the last frame of that clip. That frame is stored into the next row’s `START` column so the next generated scene begins from the previous ending frame.

## 1.4 Final Clip Collection and Video Merge
In parallel with scene iteration, the workflow also gathers all rows marked with `MERGE = x`, collects their `VIDEO URL` values, sends them to **Fal AI FFmpeg merge-videos**, and polls until the final merged video is ready.

## 1.5 Final Distribution
After the merged video is available, the workflow:
- uploads it to **Google Drive**,
- sends it to **Upload-Post** for YouTube upload,
- uploads it to **Postiz**,
- then posts it through a configured social integration.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Batch Iteration

### Overview
This block starts the workflow manually, fetches rows from the Google Sheet that still need processing, and loops through them one item at a time. It is the orchestration entry point for both clip generation and final merge preparation.

### Nodes Involved
- **When clicking ‘Execute workflow’**
- **Get new video**
- **Loop Over Items1**

### Node Details

#### 2.1.1 When clicking ‘Execute workflow’
- **Type / role:** `Manual Trigger` — manual entry point for the workflow.
- **Configuration choices:** No custom parameters; execution begins when launched from the n8n editor.
- **Key expressions / variables:** None.
- **Input / output connections:**
  - Input: none
  - Output: **Get new video**
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:**
  - No automatic scheduling; if recurring execution is needed, this must be replaced or complemented by a Cron trigger.
- **Sub-workflow reference:** None.

#### 2.1.2 Get new video
- **Type / role:** `Google Sheets` — retrieves rows to process.
- **Configuration choices:** Reads from spreadsheet **“Long video”**, sheet **gid=0 / Foglio1**. Filters are configured on:
  - `VIDEO URL`
  - `MERGE`
  
  Because lookup values are not explicitly shown, this node appears intended to fetch rows where output columns are still empty or otherwise match the configured filtering behavior in n8n.
- **Key expressions / variables:** None in the visible config.
- **Input / output connections:**
  - Input: **When clicking ‘Execute workflow’**
  - Output: **Loop Over Items1**
- **Version-specific requirements:** `typeVersion: 4.5`
- **Edge cases / failures:**
  - Wrong spreadsheet ID or sheet selection will cause lookup failure.
  - OAuth permission issues will block access.
  - Ambiguous filters may return unexpected rows if sheet data is inconsistent.
- **Sub-workflow reference:** None.

#### 2.1.3 Loop Over Items1
- **Type / role:** `Split In Batches` — processes sheet rows sequentially.
- **Configuration choices:** Default batching behavior; no custom batch size shown.
- **Key expressions / variables:** Uses each incoming row item, including `row_number`, for later updates.
- **Input / output connections:**
  - Input: **Get new video**, and loop-back from **Update last frame**
  - Outputs:
    - **Get videos**
    - **Get frame**
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases / failures:**
  - If no rows are returned, downstream nodes do not execute.
  - Sequential looping assumes row ordering in Google Sheets is meaningful.
- **Sub-workflow reference:** None.

---

## 2.2 Per-Scene Data Preparation and Video Generation

### Overview
This block takes the current sheet row, fetches the current frame/source row, maps the row fields into a clean structure, submits a generation job to RunPod, and polls until the video is complete.

### Nodes Involved
- **Get frame**
- **Set data**
- **Generate video**
- **Wait 60 sec.3**
- **Generate video status**
- **Completed?3**
- **Get Url Video**

### Node Details

#### 2.2.1 Get frame
- **Type / role:** `Google Sheets` — fetches a row representing the current frame source.
- **Configuration choices:** Reads the same spreadsheet and sheet, with `returnFirstMatch: true`, filtered on `VIDEO URL`.
- **Key expressions / variables:** No explicit lookup value shown in the exported config.
- **Input / output connections:**
  - Input: **Loop Over Items1**
  - Output: **Set data**
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases / failures:**
  - If the lookup is not fully configured, it may return an unintended row or no row.
  - The workflow depends on this node returning `PROMPT`, `DURATION`, and `START`.
- **Sub-workflow reference:** None.

#### 2.2.2 Set data
- **Type / role:** `Set` — normalizes the current row into the fields needed by RunPod.
- **Configuration choices:** Creates:
  - `prompt = {{$json.PROMPT}}`
  - `duration = {{$json.DURATION}}`
  - `image = {{$json.START}}`
- **Key expressions / variables:**
  - `prompt`
  - `duration`
  - `image`
- **Input / output connections:**
  - Input: **Get frame**
  - Output: **Generate video**
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases / failures:**
  - Missing `PROMPT`, `DURATION`, or `START` will create invalid generation payloads.
  - `duration` is stored as string here, but later inserted numerically in JSON; malformed values may break the request body.
- **Sub-workflow reference:** None.

#### 2.2.3 Generate video
- **Type / role:** `HTTP Request` — submits a video generation job to RunPod.
- **Configuration choices:**
  - `POST https://api.runpod.ai/v2/wan-2-5/run`
  - Bearer authentication via **Runpods**
  - JSON body includes:
    - `prompt`
    - `image`
    - empty `negative_prompt`
    - `size = 1280*720`
    - `duration`
    - `seed = -1`
    - `enable_prompt_expansion = false`
    - `enable_safety_checker = true`
- **Key expressions / variables:**
  - `{{ $json.prompt }}`
  - `{{ $json.image }}`
  - `{{ $json.duration }}`
- **Input / output connections:**
  - Input: **Set data**
  - Output: **Wait 60 sec.3**
- **Version-specific requirements:** `typeVersion: 4.3`
- **Edge cases / failures:**
  - Invalid or expired RunPod bearer token.
  - Unsupported image URL, duration, or prompt content.
  - Model endpoint availability or job queue delays.
  - JSON rendering failure if `duration` is not numeric.
- **Sub-workflow reference:** None.

#### 2.2.4 Wait 60 sec.3
- **Type / role:** `Wait` — pauses before polling job status.
- **Configuration choices:** Default wait/resume behavior; no explicit delay field visible, but naming implies ~60 seconds.
- **Key expressions / variables:** None.
- **Input / output connections:**
  - Inputs: **Generate video**, false branch loop from **Completed?3**
  - Output: **Generate video status**
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases / failures:**
  - Wait nodes require webhook-based resume support in n8n.
  - If instance restarts improperly or wait execution storage is misconfigured, polling may fail to resume.
- **Sub-workflow reference:** None.

#### 2.2.5 Generate video status
- **Type / role:** `HTTP Request` — checks RunPod job status.
- **Configuration choices:**
  - `GET https://api.runpod.ai/v2/wan-2-5/status/{{ $json.id }}`
  - Bearer auth via **Runpods**
- **Key expressions / variables:**
  - `{{ $json.id }}`
- **Input / output connections:**
  - Input: **Wait 60 sec.3**
  - Output: **Completed?3**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - Missing `id` from prior response.
  - Non-terminal statuses can cause repeated loops.
  - API rate limits or transient RunPod errors.
- **Sub-workflow reference:** None.

#### 2.2.6 Completed?3
- **Type / role:** `If` — checks whether RunPod reports `COMPLETED`.
- **Configuration choices:** Condition:
  - `{{$json.status}} == "COMPLETED"`
- **Key expressions / variables:**
  - `{{ $json.status }}`
- **Input / output connections:**
  - Input: **Generate video status**
  - True output: **Get Url Video**
  - False output: **Wait 60 sec.3**
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases / failures:**
  - If RunPod returns `FAILED`, this logic still loops because only `COMPLETED` is explicitly handled.
  - A more robust implementation should branch on failure states separately.
- **Sub-workflow reference:** None.

#### 2.2.7 Get Url Video
- **Type / role:** `HTTP Request` — retrieves the final RunPod result payload from the output URL.
- **Configuration choices:**
  - URL comes from `{{ $json.output.result }}`
  - Uses generic credential type with bearer auth configured
- **Key expressions / variables:**
  - `{{ $json.output.result }}`
- **Input / output connections:**
  - Input: **Completed?3** (true branch)
  - Output: **Update video**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - If `output.result` is absent or changed in RunPod response format, the request fails.
  - Signed URLs may expire.
  - Mixed credentials listed in the export suggest possible config leftovers; only the intended auth method should remain.
- **Sub-workflow reference:** None.

---

## 2.3 Write Clip URL Back to Sheet

### Overview
This block stores the generated video URL in the source spreadsheet and marks the row for inclusion in merging.

### Nodes Involved
- **Update video**

### Node Details

#### 2.3.1 Update video
- **Type / role:** `Google Sheets` — updates the current row after clip generation.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Writes:
    - `MERGE = "x"`
    - `VIDEO URL = {{ $('Completed?3').item.json.output.result }}`
    - `row_number = {{ $('Loop Over Items1').item.json.row_number }}`
- **Key expressions / variables:**
  - `{{ $('Completed?3').item.json.output.result }}`
  - `{{ $('Loop Over Items1').item.json.row_number }}`
- **Input / output connections:**
  - Input: **Get Url Video**
  - Output: **Extract last frame1**
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases / failures:**
  - The node writes the URL from **Completed?3** rather than the possibly transformed response from **Get Url Video**. This is important: it assumes `Completed?3.item.json.output.result` is already the desired video URL.
  - If the spreadsheet schema differs from the configured columns, update fails.
  - `row_number` must be present and stable.
- **Sub-workflow reference:** None.

---

## 2.4 Last Frame Extraction and Propagation to the Next Scene

### Overview
This block extracts the last frame from the newly generated clip using Fal AI, polls until the extraction is complete, retrieves the image URL, and writes that image into the next row’s `START` field. It also clears the next row’s `VIDEO URL`, ensuring the next scene can be regenerated using the new continuity image.

### Nodes Involved
- **Extract last frame1**
- **Wait 60 sec.4**
- **Extract Frame Status**
- **Completed?4**
- **Get Last frame**
- **Update last frame**

### Node Details

#### 2.4.1 Extract last frame1
- **Type / role:** `HTTP Request` — submits a Fal AI frame extraction job.
- **Configuration choices:**
  - `POST https://queue.fal.run/fal-ai/ffmpeg-api/extract-frame`
  - JSON body:
    - `video_url = {{ $json["VIDEO URL"] }}`
    - `frame_type = "last"`
  - Header auth via **Fal.run API**
- **Key expressions / variables:**
  - `{{ $json["VIDEO URL"] }}`
- **Input / output connections:**
  - Input: **Update video**
  - Output: **Wait 60 sec.4**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - If incoming item does not include `VIDEO URL`, extraction fails.
  - Fal AI may reject inaccessible URLs or unsupported media.
- **Sub-workflow reference:** None.

#### 2.4.2 Wait 60 sec.4
- **Type / role:** `Wait` — delays before polling extraction status.
- **Configuration choices:** Default wait configuration; named as 60 seconds.
- **Key expressions / variables:** None.
- **Input / output connections:**
  - Inputs: **Extract last frame1**, false branch loop from **Completed?4**
  - Output: **Extract Frame Status**
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases / failures:** Same wait-resume considerations as the other wait nodes.
- **Sub-workflow reference:** None.

#### 2.4.3 Extract Frame Status
- **Type / role:** `HTTP Request` — checks Fal AI extraction job status.
- **Configuration choices:**
  - `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Extract last frame1').item.json.request_id }}/status`
  - Header auth via **Fal.run API**
- **Key expressions / variables:**
  - `{{ $('Extract last frame1').item.json.request_id }}`
- **Input / output connections:**
  - Input: **Wait 60 sec.4**
  - Output: **Completed?4**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - Missing `request_id`.
  - Fal queue delays or rate limits.
  - Trailing space in the configured URL could potentially cause issues depending on server tolerance.
- **Sub-workflow reference:** None.

#### 2.4.4 Completed?4
- **Type / role:** `If` — checks whether frame extraction finished.
- **Configuration choices:** Condition:
  - `{{$json.status}} == "COMPLETED"`
- **Key expressions / variables:**
  - `{{ $json.status }}`
- **Input / output connections:**
  - Input: **Extract Frame Status**
  - True output: **Get Last frame**
  - False output: **Wait 60 sec.4**
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases / failures:**
  - `FAILED` state is not explicitly handled and will loop forever.
- **Sub-workflow reference:** None.

#### 2.4.5 Get Last frame
- **Type / role:** `HTTP Request` — retrieves final extraction result.
- **Configuration choices:**
  - `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
  - Header auth via **Fal.run API**
- **Key expressions / variables:**
  - `{{ $json.request_id }}`
- **Input / output connections:**
  - Input: **Completed?4** (true branch)
  - Output: **Update last frame**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - If `request_id` is not present in the status response, retrieval fails.
  - Output schema changes may break the downstream `images[0].url` expression.
- **Sub-workflow reference:** None.

#### 2.4.6 Update last frame
- **Type / role:** `Google Sheets` — writes continuity image into the next row.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Writes:
    - `START = {{ $json.images[0].url }}`
    - `VIDEO URL = ""`
    - `row_number = {{ $('Loop Over Items1').item.json.row_number + 1 }}`
- **Key expressions / variables:**
  - `{{ $json.images[0].url }}`
  - `{{ $('Loop Over Items1').item.json.row_number + 1 }}`
- **Input / output connections:**
  - Input: **Get Last frame**
  - Output: **Loop Over Items1**
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases / failures:**
  - If there is no next row, the update may fail or affect a nonexistent row.
  - The schema lists a different duration column label (`DURATION (4, 6 or 8seconds)`) than other nodes (`DURATION`), which may indicate sheet schema drift.
  - Missing `images[0].url` will break the update.
- **Sub-workflow reference:** None.

---

## 2.5 Final Video Collection and Merge

### Overview
This block independently gathers all rows marked for merge, converts their `VIDEO URL` values into an array, sends them to Fal AI’s FFmpeg merge endpoint, and polls until the final merged asset is ready.

### Nodes Involved
- **Get videos**
- **videoUrls**
- **Merge videos**
- **Wait 60 sec.2**
- **Merge videos status**
- **Completed?5**
- **Get final video**

### Node Details

#### 2.5.1 Get videos
- **Type / role:** `Google Sheets` — fetches all rows where `MERGE = x`.
- **Configuration choices:**
  - Spreadsheet: same Google Sheet
  - Filter:
    - `MERGE` equals `x`
- **Key expressions / variables:** None.
- **Input / output connections:**
  - Input: **Loop Over Items1**
  - Output: **videoUrls**
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases / failures:**
  - If executed before all rows are generated, merge may include only already completed clips.
  - Because this branch is triggered during iteration, merge timing depends on data state at that moment.
- **Sub-workflow reference:** None.

#### 2.5.2 videoUrls
- **Type / role:** `Code` — transforms rows into a single array of video URLs.
- **Configuration choices:** JavaScript code:
  - reads all input items,
  - extracts `item.json["VIDEO URL"]`,
  - returns one item with `json.videos = [...]`
- **Key expressions / variables:**
  - `VIDEO URL`
  - output field `videos`
- **Input / output connections:**
  - Input: **Get videos**
  - Output: **Merge videos**
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases / failures:**
  - Empty or null URLs are included unless filtered in code.
  - If Google Sheets returns blank rows marked `x`, merge request may fail.
- **Sub-workflow reference:** None.

#### 2.5.3 Merge videos
- **Type / role:** `HTTP Request` — submits merge job to Fal AI FFmpeg API.
- **Configuration choices:**
  - `POST https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
  - JSON body:
    - `video_urls = {{ JSON.stringify($json.videos) }}`
    - `target_fps = 24`
  - Header auth via **Fal.run API**
- **Key expressions / variables:**
  - `{{ JSON.stringify($json.videos) }}`
- **Input / output connections:**
  - Input: **videoUrls**
  - Output: **Wait 60 sec.2**
- **Version-specific requirements:** `typeVersion: 4.3`
- **Edge cases / failures:**
  - Invalid or empty URL list.
  - Clips with incompatible encoding, dimensions, or accessibility.
  - Fal API queue delays.
- **Sub-workflow reference:** None.

#### 2.5.4 Wait 60 sec.2
- **Type / role:** `Wait` — delays before polling merge status.
- **Configuration choices:** Default wait node, named as 60 seconds.
- **Key expressions / variables:** None.
- **Input / output connections:**
  - Inputs: **Merge videos**, false branch loop from **Completed?5**
  - Output: **Merge videos status**
- **Version-specific requirements:** `typeVersion: 1.1`
- **Edge cases / failures:** Same wait-resume caveats as other wait nodes.
- **Sub-workflow reference:** None.

#### 2.5.5 Merge videos status
- **Type / role:** `HTTP Request` — checks merge job status.
- **Configuration choices:**
  - `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Merge videos').item.json.request_id }}/status`
  - Header auth via **Fal.run API**
- **Key expressions / variables:**
  - `{{ $('Merge videos').item.json.request_id }}`
- **Input / output connections:**
  - Input: **Wait 60 sec.2**
  - Output: **Completed?5**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - Missing `request_id`
  - Trailing space in URL
  - Infinite loop on `FAILED` or unexpected statuses
- **Sub-workflow reference:** None.

#### 2.5.6 Completed?5
- **Type / role:** `If` — checks whether final merge is complete.
- **Configuration choices:** Condition:
  - `{{$json.status}} == "COMPLETED"`
- **Key expressions / variables:**
  - `{{ $json.status }}`
- **Input / output connections:**
  - Input: **Merge videos status**
  - True output: **Get final video**
  - False output: **Wait 60 sec.2**
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases / failures:**
  - `FAILED` is not handled explicitly.
- **Sub-workflow reference:** None.

#### 2.5.7 Get final video
- **Type / role:** `HTTP Request` — retrieves merged video result metadata/content endpoint.
- **Configuration choices:**
  - `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
  - Header auth via **Fal.run API**
- **Key expressions / variables:**
  - `{{ $json.request_id }}`
- **Input / output connections:**
  - Input: **Completed?5** (true branch)
  - Outputs:
    - **Upload Video**
    - **Upload to Youtube**
    - **Upload to Postiz**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - The downstream upload nodes expect either a file URL and/or binary payload; if this node returns metadata only, additional download handling may be required.
- **Sub-workflow reference:** None.

---

## 2.6 Final Upload and Distribution

### Overview
This block distributes the merged video to storage and publication channels. It uploads to Google Drive, sends the media to Upload-Post for YouTube, uploads to Postiz, and finally creates a social post through the Postiz node.

### Nodes Involved
- **Upload Video**
- **Upload to Youtube**
- **Upload to Postiz**
- **Upload to Social**

### Node Details

#### 2.6.1 Upload Video
- **Type / role:** `Google Drive` — uploads the merged video to a Drive folder.
- **Configuration choices:**
  - File name:
    - `{{$now.format('yyyyLLddHHmmss')}}-{{ $('Get Clip Url').item.json.video.file_name }}`
  - Drive: `My Drive`
  - Folder: `Fal.run`
- **Key expressions / variables:**
  - `{{ $now.format('yyyyLLddHHmmss') }}`
  - `{{ $('Get Clip Url').item.json.video.file_name }}`
- **Input / output connections:**
  - Input: **Get final video**
  - Output: none
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases / failures:**
  - The expression references **Get Clip Url**, but no node with that exact name exists in this workflow. This is likely a broken leftover reference and will fail unless corrected.
  - A binary file input may be required for Google Drive upload; if only metadata is passed, upload will fail.
  - Folder permissions must allow write access.
- **Sub-workflow reference:** None.

#### 2.6.2 Upload to Youtube
- **Type / role:** `HTTP Request` — uploads the final video to Upload-Post targeting YouTube.
- **Configuration choices:**
  - `POST https://api.upload-post.com/api/upload`
  - `multipart/form-data`
  - Header auth
  - Body fields:
    - `title = XXX`
    - `user = YOUR_USERNAME`
    - `platform[] = youtube`
    - `video` from binary field `data`
- **Key expressions / variables:**
  - Static placeholders need replacement.
- **Input / output connections:**
  - Input: **Get final video**
  - Output: none
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - Requires valid Upload-Post API key.
  - Requires binary data field named `data`.
  - Placeholder values (`XXX`, `YOUR_USERNAME`) must be replaced.
  - Platform-side title/format restrictions may apply.
- **Sub-workflow reference:** None.

#### 2.6.3 Upload to Postiz
- **Type / role:** `HTTP Request` — uploads the video asset to Postiz.
- **Configuration choices:**
  - `POST https://api.postiz.com/public/v1/upload`
  - `multipart/form-data`
  - Header auth
  - `file` uses binary field `data`
- **Key expressions / variables:** Binary field `data`
- **Input / output connections:**
  - Input: **Get final video**
  - Output: **Upload to Social**
- **Version-specific requirements:** `typeVersion: 4.2`
- **Edge cases / failures:**
  - Requires valid Postiz API key.
  - Requires actual binary video input.
  - If Postiz returns unexpected schema, next node may fail.
- **Sub-workflow reference:** None.

#### 2.6.4 Upload to Social
- **Type / role:** `Postiz` — creates/post schedules content using uploaded asset.
- **Configuration choices:**
  - `date = {{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
  - `shortLink = true`
  - Post content includes:
    - media item id/path from previous Postiz upload response
    - content text placeholder `XXX`
    - `integrationId = XXX`
- **Key expressions / variables:**
  - `{{ $json.id }}`
  - `{{ $json.path }}`
  - `{{ $now.format('yyyy-LL-dd') }}`
  - `{{ $now.format('HH:ii:ss') }}`
- **Input / output connections:**
  - Input: **Upload to Postiz**
  - Output: none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:**
  - Placeholder content and integration ID must be replaced.
  - The date formatting uses `ii` for minutes; verify Luxon formatting in your n8n version.
  - Posting may fail if asset upload is incomplete or integration is misconfigured.
- **Sub-workflow reference:** None.

---

## 2.7 Documentation / Visual Annotation Nodes

### Overview
These nodes are non-executable sticky notes used to explain setup, process flow, resources, and external links inside the n8n canvas.

### Nodes Involved
- **Sticky Note3**
- **Sticky Note4**
- **Sticky Note5**
- **Sticky Note6**
- **Sticky Note7**
- **Sticky Note8**
- **Sticky Note9**

### Node Details

#### 2.7.1 Sticky Note3
- **Type / role:** `Sticky Note` — setup instruction for Google Sheets.
- **Configuration choices:** Instructs user to clone the sheet template and fill basic info.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None; documentation only.
- **Sub-workflow reference:** None.

#### 2.7.2 Sticky Note4
- **Type / role:** `Sticky Note` — overall workflow description and setup requirements.
- **Configuration choices:** Documents end-to-end process and required credentials.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 2.7.3 Sticky Note5
- **Type / role:** `Sticky Note` — explains RunPod generation step.
- **Configuration choices:** Links to RunPod signup.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 2.7.4 Sticky Note6
- **Type / role:** `Sticky Note` — explains frame extraction step.
- **Configuration choices:** Describes last frame extraction after each video.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 2.7.5 Sticky Note7
- **Type / role:** `Sticky Note` — explains merge step.
- **Configuration choices:** Notes that individual clips are merged into a final video.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 2.7.6 Sticky Note8
- **Type / role:** `Sticky Note` — explains final upload/distribution step.
- **Configuration choices:** Includes Upload-Post and Postiz API links.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 2.7.7 Sticky Note9
- **Type / role:** `Sticky Note` — branding/promo note.
- **Configuration choices:** Contains YouTube channel subscription link and image.
- **Input / output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start |  | Get new video |  |
| Get new video | Google Sheets | Fetch rows to process from the sheet | When clicking ‘Execute workflow’ | Loop Over Items1 | ## STEP 1 - Set Sheet<br>Clone [this sheet](https://docs.google.com/spreadsheets/d/1MisBkHc1RmsYit1ndaPS7oOvSQV1VBMW7nyehTuiRQs/edit?usp=sharing) and fill basic info |
| Loop Over Items1 | Split In Batches | Sequential row iteration | Get new video; Update last frame | Get videos; Get frame | ## STEP 1 - Set Sheet<br>Clone [this sheet](https://docs.google.com/spreadsheets/d/1MisBkHc1RmsYit1ndaPS7oOvSQV1VBMW7nyehTuiRQs/edit?usp=sharing) and fill basic info |
| Get frame | Google Sheets | Retrieve current row/frame data | Loop Over Items1 | Set data | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Set data | Set | Map sheet fields into API payload fields | Get frame | Generate video | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Generate video | HTTP Request | Submit clip generation job to RunPod | Set data | Wait 60 sec.3 | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Wait 60 sec.3 | Wait | Delay before polling RunPod status | Generate video; Completed?3 | Generate video status | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Generate video status | HTTP Request | Poll RunPod job status | Wait 60 sec.3 | Completed?3 | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Completed?3 | If | Check whether clip generation is complete | Generate video status | Get Url Video; Wait 60 sec.3 | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Get Url Video | HTTP Request | Retrieve generated clip result | Completed?3 | Update video |  |
| Update video | Google Sheets | Write clip URL back to current row | Get Url Video | Extract last frame1 | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Extract last frame1 | HTTP Request | Request extraction of last frame from clip | Update video | Wait 60 sec.4 | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Wait 60 sec.4 | Wait | Delay before polling frame extraction status | Extract last frame1; Completed?4 | Extract Frame Status | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Extract Frame Status | HTTP Request | Poll Fal AI frame extraction status | Wait 60 sec.4 | Completed?4 | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Completed?4 | If | Check whether frame extraction is complete | Extract Frame Status | Get Last frame; Wait 60 sec.4 | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Get Last frame | HTTP Request | Retrieve extracted frame result | Completed?4 | Update last frame | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Update last frame | Google Sheets | Write extracted frame into next row START field | Get Last frame | Loop Over Items1 | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Get videos | Google Sheets | Fetch rows marked for merge | Loop Over Items1 | videoUrls | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| videoUrls | Code | Build array of video URLs | Get videos | Merge videos | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Merge videos | HTTP Request | Submit final merge job to Fal AI | videoUrls | Wait 60 sec.2 | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Wait 60 sec.2 | Wait | Delay before polling merge status | Merge videos; Completed?5 | Merge videos status | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Merge videos status | HTTP Request | Poll final merge job status | Wait 60 sec.2 | Completed?5 | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Completed?5 | If | Check whether merge is complete | Merge videos status | Get final video; Wait 60 sec.2 | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Get final video | HTTP Request | Retrieve merged video result | Completed?5 | Upload Video; Upload to Youtube; Upload to Postiz | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Upload Video | Google Drive | Upload final merged video to Google Drive | Get final video |  | ## STEP 5 - Upload to Social and Google Drive<br>Posted to multiple social platforms and upload to Google Drive<br>- Get [Upload-Post API Key](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)<br>- Get [Postiz API Key](https://affiliate.postiz.com/n3witalia) |
| Upload to Youtube | HTTP Request | Upload final video to Upload-Post/YouTube | Get final video |  | ## STEP 5 - Upload to Social and Google Drive<br>Posted to multiple social platforms and upload to Google Drive<br>- Get [Upload-Post API Key](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)<br>- Get [Postiz API Key](https://affiliate.postiz.com/n3witalia) |
| Upload to Postiz | HTTP Request | Upload final video asset to Postiz | Get final video | Upload to Social | ## STEP 5 - Upload to Social and Google Drive<br>Posted to multiple social platforms and upload to Google Drive<br>- Get [Upload-Post API Key](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)<br>- Get [Postiz API Key](https://affiliate.postiz.com/n3witalia) |
| Upload to Social | Postiz | Publish/upload media to connected social platforms | Upload to Postiz |  | ## STEP 5 - Upload to Social and Google Drive<br>Posted to multiple social platforms and upload to Google Drive<br>- Get [Upload-Post API Key](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)<br>- Get [Postiz API Key](https://affiliate.postiz.com/n3witalia) |
| Sticky Note3 | Sticky Note | Canvas documentation for sheet setup |  |  | ## STEP 1 - Set Sheet<br>Clone [this sheet](https://docs.google.com/spreadsheets/d/1MisBkHc1RmsYit1ndaPS7oOvSQV1VBMW7nyehTuiRQs/edit?usp=sharing) and fill basic info |
| Sticky Note4 | Sticky Note | Canvas documentation for full workflow and setup |  |  | ## Automated Long Video Creator & Multi-Platform Upload<br>This workflow automates the **creation of long AI-generated videos from prompts**, merges the generated clips into a single video, and automatically distributes the final content across multiple platforms.<br><br>### How it works<br>This workflow automates long-form AI video creation by generating multiple clips from prompts stored in a Google Sheet and assembling them into a single video. When triggered, it reads rows containing prompts, durations, and starting images, then sequentially sends these parameters to the RunPod WAN 2.5 video generation API. The workflow monitors job status with periodic polling until each clip is complete, retrieves the resulting video URL, and records it back in the sheet for tracking.<br><br>After each clip is produced, the workflow extracts the final frame using the Fal AI FFmpeg API and assigns it as the starting image for the next scene, ensuring visual continuity. Once all clips marked for merging are ready, their URLs are collected and sent to the Fal AI FFmpeg merge API to produce a single long video. The merged file is then retrieved and automatically distributed by uploading it to Google Drive, publishing it to YouTube, and sending it to Postiz for cross-posting across social platforms such as TikTok, Instagram, Facebook, and X.<br><br>### Setup steps<br>Begin by cloning the provided Google Sheet template and updating the Sheet ID in all Google Sheets nodes within the workflow. Populate the sheet with scene data including the initial image URL (START), the generation prompt (PROMPT), and clip duration (DURATION). Mark any rows that should be included in the final merged video with an “x” in the MERGE column so the workflow knows which clips to assemble.<br><br>Next, configure the required API credentials in n8n: Google Sheets OAuth2 for spreadsheet access, Google Drive OAuth2 for storing the final video, RunPod API for WAN 2.5 video generation, Fal AI API for frame extraction and merging, Upload-Post API for YouTube publishing, and Postiz API for distributing the video to social media platforms. Update workflow nodes with your specific details, including the YouTube username, social integration IDs, titles, and the correct Google Drive folder IDs. Finally, run the workflow manually to test the full pipeline and confirm that clips generate, merge correctly, and publish successfully. |
| Sticky Note5 | Sticky Note | Canvas documentation for generation step |  |  | ## STEP 2  - Generate short video<br>Sign up to [Runpod](https://get.runpod.io/n3witalia)<br>Sends the prompt and parameters to RunPod's WAN 2.5 video generation API |
| Sticky Note6 | Sticky Note | Canvas documentation for frame extraction step |  |  | ## STEP 3 - Extract Last Frame<br>After each video is generated, it extracts the last frame and save to Google Drive |
| Sticky Note7 | Sticky Note | Canvas documentation for merge step |  |  | ## STEP 4 - Merge videos<br>All individual clips are merged into long final video |
| Sticky Note8 | Sticky Note | Canvas documentation for upload/distribution step |  |  | ## STEP 5 - Upload to Social and Google Drive<br>Posted to multiple social platforms and upload to Google Drive<br>- Get [Upload-Post API Key](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)<br>- Get [Postiz API Key](https://affiliate.postiz.com/n3witalia) |
| Sticky Note9 | Sticky Note | Branding / external channel note |  |  | ## MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia). Here I’ll share videos and Shorts with practical tutorials and **FREE templates for n8n**.<br><br>[![image](https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg)](https://youtube.com/@n3witalia) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like **Automated Long Video Creator**.

2. **Add a Manual Trigger node**.
   - Use node type: **Manual Trigger**
   - Keep default settings.
   - This is the workflow entry point.

3. **Prepare a Google Sheet** with at least these columns:
   - `START`
   - `PROMPT`
   - `DURATION`
   - `VIDEO URL`
   - `MERGE`
   - `row_number` must be available to n8n when reading/updating rows.
   - Populate each row with scene data.
   - Put `x` in `MERGE` for rows that should be part of the final video.

4. **Add a Google Sheets node named `Get new video`**.
   - Operation: read/filter rows
   - Select your spreadsheet and target sheet.
   - Configure Google Sheets OAuth2 credentials.
   - Set filters to identify rows needing processing, based on `VIDEO URL` and optionally `MERGE`.
   - Connect **Manual Trigger → Get new video**.

5. **Add a Split In Batches node named `Loop Over Items1`**.
   - Keep default options unless you want a custom batch size.
   - Connect **Get new video → Loop Over Items1**.

6. **Add a Google Sheets node named `Get frame`**.
   - Use the same spreadsheet and sheet.
   - Enable **Return First Match**.
   - Configure filtering so the correct scene row is returned.
   - This node must expose `PROMPT`, `DURATION`, and `START`.
   - Connect **Loop Over Items1 → Get frame**.

7. **Add a Set node named `Set data`**.
   - Create three fields:
     - `prompt` = `{{$json.PROMPT}}`
     - `duration` = `{{$json.DURATION}}`
     - `image` = `{{$json.START}}`
   - Connect **Get frame → Set data**.

8. **Create RunPod credentials**.
   - Add an **HTTP Bearer Auth** credential in n8n.
   - Paste your RunPod API key.
   - Use it for all RunPod nodes.

9. **Add an HTTP Request node named `Generate video`**.
   - Method: `POST`
   - URL: `https://api.runpod.ai/v2/wan-2-5/run`
   - Authentication: **Generic Credential Type**
   - Generic auth type: **HTTP Bearer Auth**
   - Credential: your RunPod credential
   - Body type: JSON
   - JSON body should include:
     - `prompt`
     - `image`
     - `negative_prompt` as empty string
     - `size` as `1280*720`
     - `duration`
     - `seed` as `-1`
     - `enable_prompt_expansion` as `false`
     - `enable_safety_checker` as `true`
   - Connect **Set data → Generate video**.

10. **Add a Wait node named `Wait 60 sec.3`**.
    - Configure an appropriate delay, e.g. 60 seconds.
    - Connect **Generate video → Wait 60 sec.3**.

11. **Add an HTTP Request node named `Generate video status`**.
    - Method: `GET`
    - URL: `https://api.runpod.ai/v2/wan-2-5/status/{{ $json.id }}`
    - Authentication: RunPod bearer auth
    - Connect **Wait 60 sec.3 → Generate video status**.

12. **Add an If node named `Completed?3`**.
    - Condition:
      - Left value: `{{$json.status}}`
      - Operator: equals
      - Right value: `COMPLETED`
    - Connect **Generate video status → Completed?3**.

13. **Create the polling loop for generation**.
    - Connect the **false** output of `Completed?3` back to **Wait 60 sec.3**.
    - Connect the **true** output to the next retrieval node.

14. **Add an HTTP Request node named `Get Url Video`**.
    - Method: `GET`
    - URL: `{{$json.output.result}}`
    - Use the same auth only if required by the returned URL.
    - Connect **Completed?3 (true) → Get Url Video**.

15. **Add a Google Sheets node named `Update video`**.
    - Operation: `Update`
    - Matching column: `row_number`
    - Write:
      - `MERGE = x`
      - `VIDEO URL = {{ $('Completed?3').item.json.output.result }}`
      - `row_number = {{ $('Loop Over Items1').item.json.row_number }}`
    - Connect **Get Url Video → Update video**.

16. **Create Fal AI credentials**.
    - Add an **HTTP Header Auth** credential in n8n.
    - Configure the required authorization header for Fal AI.
    - Use this credential for all Fal AI nodes.

17. **Add an HTTP Request node named `Extract last frame1`**.
    - Method: `POST`
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/extract-frame`
    - Authentication: generic credential type with **HTTP Header Auth**
    - JSON body:
      - `video_url = {{$json["VIDEO URL"]}}`
      - `frame_type = last`
    - Connect **Update video → Extract last frame1**.

18. **Add a Wait node named `Wait 60 sec.4`**.
    - Delay around 60 seconds.
    - Connect **Extract last frame1 → Wait 60 sec.4**.

19. **Add an HTTP Request node named `Extract Frame Status`**.
    - Method: `GET`
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Extract last frame1').item.json.request_id }}/status`
    - Use Fal AI header auth.
    - Connect **Wait 60 sec.4 → Extract Frame Status**.

20. **Add an If node named `Completed?4`**.
    - Condition:
      - `{{$json.status}} == "COMPLETED"`
    - Connect **Extract Frame Status → Completed?4**.

21. **Create the polling loop for frame extraction**.
    - False output of **Completed?4** → **Wait 60 sec.4**
    - True output → next retrieval node

22. **Add an HTTP Request node named `Get Last frame`**.
    - Method: `GET`
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
    - Use Fal AI header auth.
    - Connect **Completed?4 (true) → Get Last frame**.

23. **Add a Google Sheets node named `Update last frame`**.
    - Operation: `Update`
    - Matching column: `row_number`
    - Write:
      - `START = {{ $json.images[0].url }}`
      - `VIDEO URL = ""`
      - `row_number = {{ $('Loop Over Items1').item.json.row_number + 1 }}`
    - Connect **Get Last frame → Update last frame**.

24. **Close the scene-processing loop**.
    - Connect **Update last frame → Loop Over Items1**.
    - This causes the next item/batch iteration to continue.

25. **Add a Google Sheets node named `Get videos`**.
    - Operation: read/filter rows
    - Filter where `MERGE = x`
    - Connect **Loop Over Items1 → Get videos**.

26. **Add a Code node named `videoUrls`**.
    - Paste logic that:
      - reads all incoming items,
      - extracts `VIDEO URL`,
      - returns one item containing `videos: [...]`
    - Connect **Get videos → videoUrls**.

27. **Add an HTTP Request node named `Merge videos`**.
    - Method: `POST`
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
    - Use Fal AI header auth
    - JSON body:
      - `video_urls = {{ JSON.stringify($json.videos) }}`
      - `target_fps = 24`
    - Connect **videoUrls → Merge videos**.

28. **Add a Wait node named `Wait 60 sec.2`**.
    - Delay about 60 seconds.
    - Connect **Merge videos → Wait 60 sec.2**.

29. **Add an HTTP Request node named `Merge videos status`**.
    - Method: `GET`
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Merge videos').item.json.request_id }}/status`
    - Use Fal AI auth.
    - Connect **Wait 60 sec.2 → Merge videos status**.

30. **Add an If node named `Completed?5`**.
    - Condition:
      - `{{$json.status}} == "COMPLETED"`
    - Connect **Merge videos status → Completed?5**.

31. **Create the merge polling loop**.
    - False output of **Completed?5** → **Wait 60 sec.2**
    - True output → next retrieval node

32. **Add an HTTP Request node named `Get final video`**.
    - Method: `GET`
    - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
    - Use Fal AI auth.
    - Connect **Completed?5 (true) → Get final video**.

33. **Configure Google Drive credentials**.
    - Add Google Drive OAuth2 credentials.
    - Ensure write access to the target folder.

34. **Add a Google Drive node named `Upload Video`**.
    - Operation: upload file
    - Choose `My Drive`
    - Select a destination folder
    - Configure a dynamic file name, for example using timestamp plus returned file name
    - Important: make sure the node receives actual binary video content. If `Get final video` only returns metadata, insert a separate download node first.
    - Connect **Get final video → Upload Video**.

35. **Configure Upload-Post auth**.
    - Add an **HTTP Header Auth** credential with your Upload-Post API key.

36. **Add an HTTP Request node named `Upload to Youtube`**.
    - Method: `POST`
    - URL: `https://api.upload-post.com/api/upload`
    - Content type: `multipart/form-data`
    - Authentication: your header auth credential
    - Body fields:
      - `title` = your final title
      - `user` = your Upload-Post username
      - `platform[]` = `youtube`
      - `video` = binary field, usually `data`
    - Connect **Get final video → Upload to Youtube**.

37. **Configure Postiz auth**.
    - Add an **HTTP Header Auth** credential for the Postiz upload endpoint.
    - Also add a **Postiz API** credential if using the native Postiz node.

38. **Add an HTTP Request node named `Upload to Postiz`**.
    - Method: `POST`
    - URL: `https://api.postiz.com/public/v1/upload`
    - Content type: `multipart/form-data`
    - Field:
      - `file` = binary field `data`
    - Connect **Get final video → Upload to Postiz**.

39. **Add a Postiz node named `Upload to Social`**.
    - Set publication datetime using current timestamp.
    - Add post content text.
    - Use uploaded media asset fields from previous node:
      - `id = {{$json.id}}`
      - `path = {{$json.path}}`
    - Set your target `integrationId`
    - Enable `shortLink` if desired.
    - Connect **Upload to Postiz → Upload to Social**.

40. **Replace all placeholders** in the workflow.
    - Replace every `XXX`
    - Replace `YOUR_USERNAME`
    - Replace Google Sheet ID and folder IDs
    - Replace integration IDs
    - Replace any broken cross-node references

41. **Strongly recommended hardening changes before production**
    - Add separate branches for `FAILED`, `CANCELLED`, or timeout states in all three polling loops.
    - Add a binary download step before upload nodes if the final video endpoint does not already provide `data`.
    - Fix the broken expression in `Upload Video` referencing `Get Clip Url`.
    - Filter out blank URLs in the `videoUrls` code node.
    - Ensure merge starts only after all expected clips are finished.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Clone the Google Sheet template and fill in the scene data before running the workflow. | https://docs.google.com/spreadsheets/d/1MisBkHc1RmsYit1ndaPS7oOvSQV1VBMW7nyehTuiRQs/edit?usp=sharing |
| RunPod signup link referenced in the canvas notes. | https://get.runpod.io/n3witalia |
| Upload-Post API key acquisition link. | https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app |
| Postiz API key acquisition link. | https://affiliate.postiz.com/n3witalia |
| Creator’s YouTube channel link shown in the workflow notes. | https://youtube.com/@n3witalia |
| The workflow description note states that the process generates multiple clips from Google Sheets, merges them into one long video, and publishes to several platforms. | Internal canvas documentation |
| The canvas note says to update all Google Sheets nodes with your own Sheet ID and configure Google Drive, RunPod, Fal AI, Upload-Post, and Postiz credentials before testing. | Internal canvas documentation |
| A visible inconsistency exists between some sheet schemas: one node uses `DURATION`, another shows `DURATION (4, 6 or 8seconds)`. Keep your spreadsheet schema consistent across all Google Sheets nodes. | Implementation note |
| Several nodes poll only for `COMPLETED` and do not explicitly handle `FAILED`. Add failure branches to avoid infinite loops. | Reliability note |
| `Upload Video` contains a broken expression referencing a nonexistent node name `Get Clip Url`. This must be corrected before the Google Drive upload can work. | Implementation note |