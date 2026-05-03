Create long audiobooks with custom voices using Qwen3-TTS voice design

https://n8nworkflows.xyz/workflows/create-long-audiobooks-with-custom-voices-using-qwen3-tts-voice-design-13030


# Create long audiobooks with custom voices using Qwen3-TTS voice design

## 1. Workflow Overview

**Workflow name:** Audiobook with Qwen3-TTS Voice Design  
**Purpose:** Generate many short TTS audio segments from a Google Sheets “script” (text + voice parameters), store the resulting segment URLs back into the sheet, then merge the segments into a single long audiobook using Fal.run’s FFmpeg merge API, and finally upload the finished file to Google Drive.

**Primary use cases**
- Producing long-form audiobooks/podcasts from structured scripts
- Creating multi-speaker or style-varied narration using **Qwen3-TTS (Replicate)** voice design inputs
- Automating export and storage of the final merged audio in Drive

### Logical blocks
**1.1 Manual start & script ingestion (Sheets)**
- Manual trigger, read script rows from Google Sheets.

**1.2 Batch processing loop (rate-limit friendly)**
- Split rows into batches; for each row call Replicate Qwen3-TTS and write the returned audio URL into the sheet.

**1.3 Merge audio segments (Fal.run FFmpeg)**
- Collect all “Temp URL” values and request an async merge job.

**1.4 Polling for merge completion**
- Wait/poll until the merge request becomes `COMPLETED`.

**1.5 Download & upload final audiobook**
- Fetch final audio URL, download the file, upload to Google Drive with timestamped name.

---

## 2. Block-by-Block Analysis

### 2.1 Manual start & script ingestion (Sheets)

**Overview:** Starts the workflow manually and pulls script rows from a Google Sheet that contains the TTS inputs and tracking columns.

**Nodes involved**
- When clicking ‘Execute workflow’
- Get scripts

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`manualTrigger`) entry point.
- **Configuration choices:** No parameters; user runs the workflow manually.
- **Connections:**
  - Output → **Get scripts**
- **Failure / edge cases:** None (only requires manual execution).

#### Node: Get scripts
- **Type / role:** Google Sheets node; reads rows from the “Audiobooks” spreadsheet.
- **Configuration choices (interpreted):**
  - Reads from document **“Audiobooks”** and sheet tab **“Foglio1”** (`gid=0`).
  - Uses a filter UI with lookup column **“Temp URL”** (exact filter behavior depends on node mode; in many templates this is used to fetch rows missing Temp URL or matching a condition).
- **Key fields expected in the sheet:**
  - `Text`, `Speaker`, `Voice Description`, `Style Instruction`, `Temp URL`, `To Merge`, plus `row_number`.
- **Connections:**
  - Input ← Manual Trigger
  - Output → **Loop Over Items**
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Failure / edge cases:**
  - OAuth permission errors
  - Missing columns / renamed headers breaks later expressions
  - If the filter is misconfigured, it may return no rows or the wrong rows (e.g., already processed ones)

---

### 2.2 Batch processing loop (TTS generation + sheet update)

**Overview:** Iterates through script rows in batches, generates an audio segment for each row using Replicate Qwen3-TTS voice design mode, then stores the returned audio URL into the originating row.

**Nodes involved**
- Loop Over Items
- Voice Design
- Update Temp URL
- Wait 10 sec.

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`splitInBatches`) to iterate items gradually.
- **Configuration choices:** Default batch options (batch size not explicitly shown; in n8n it defaults to 1 unless set).
- **Connections:**
  - Input ← **Get scripts**
  - Output(0) → **Set AudioUrls Json** (runs in parallel path)
  - Output(0) → **Voice Design**
  - Input (continue) ← **Wait 10 sec.** (loop-back)
- **Important behavior / edge cases:**
  - Because this node fans out to **Set AudioUrls Json** and **Voice Design** simultaneously, the merge path may start before all rows have been fully generated, depending on timing and how many items are in scope. In practice, this template relies on the batch loop to keep executing, but the “merge all URLs” strategy is sensitive to partial data unless the sheet filter guarantees only “ready-to-merge” rows.
  - If batch size is >1, Replicate calls may spike and hit rate limits.

#### Node: Voice Design
- **Type / role:** HTTP Request; calls Replicate API to run **qwen/qwen3-tts** in `voice_design` mode for one row.
- **Configuration choices:**
  - `POST https://api.replicate.com/v1/models/qwen/qwen3-tts/predictions`
  - JSON body contains:
    - `mode: "voice_design"`
    - `text: {{$json.Text}}`
    - `language: "auto"`
    - `speaker: {{$json.Speaker}}`
    - `voice_description: {{$json["Voice Description"]}}`
    - `style_instruction: {{$json["Style Instruction"]}}`
  - Header: `Prefer: wait` (asks Replicate to wait for completion and return a completed prediction when possible).
  - Auth: Bearer token credential (“Replicate”).
- **Connections:**
  - Input ← **Loop Over Items**
  - Output → **Update Temp URL**
- **Edge cases / failures:**
  - Replicate auth failures (invalid token)
  - Model/endpoint changes or quota exhaustion
  - Long text may exceed model limits; request may fail or truncate
  - If Replicate returns async responses despite `Prefer: wait`, `output` may not be immediately available and `Update Temp URL` will write an empty/incorrect URL unless handled

#### Node: Update Temp URL
- **Type / role:** Google Sheets node; updates the originating row with generated audio URL and marks it for merging.
- **Configuration choices:**
  - Operation: **Update**
  - Matching column: `row_number`
  - Writes:
    - `Temp URL` = `{{$json.output}}`
    - `To Merge` = `x`
    - `row_number` = `{{$('Loop Over Items').item.json.row_number}}`
  - Uses explicit schema mapping.
- **Connections:**
  - Input ← **Voice Design**
  - Output → **Wait 10 sec.**
- **Credentials:** Google Sheets OAuth2
- **Edge cases / failures:**
  - If Replicate response structure differs (e.g., `output` is an array or nested), this may write the wrong value
  - If `row_number` isn’t present, update matching fails
  - Concurrent edits to the sheet can cause updates to target wrong rows if row numbering shifts (depending on how the connector defines `row_number`)

#### Node: Wait 10 sec.
- **Type / role:** Wait node; throttles loop iteration.
- **Configuration choices:** Wait 10 seconds.
- **Connections:**
  - Input ← **Update Temp URL**
  - Output → **Loop Over Items** (continue loop)
- **Edge cases:** Workflow execution time may become long for large books; consider increasing n8n execution timeout or using queue mode.

---

### 2.3 Merge audio segments (Fal.run FFmpeg)

**Overview:** Builds a JSON array of audio segment URLs and submits an asynchronous merge request to Fal.run’s FFmpeg API.

**Nodes involved**
- Set AudioUrls Json
- Merge Audios

#### Node: Set AudioUrls Json
- **Type / role:** Code node; transforms incoming items into `{ urls: [...] }`.
- **Configuration choices / code logic:**
  - Creates one output item:
    - `urls: items.map(item => item.json["Temp URL"])`
- **Connections:**
  - Input ← **Loop Over Items**
  - Output → **Merge Audios**
- **Edge cases / failures:**
  - If any item lacks `Temp URL`, the URL list includes `null/undefined`, potentially breaking the merge call
  - Depending on when this path runs, it may capture incomplete URLs if not all segments are generated yet

#### Node: Merge Audios
- **Type / role:** HTTP Request; submits merge job to Fal.run queue endpoint.
- **Configuration choices:**
  - `POST https://queue.fal.run/fal-ai/ffmpeg-api/merge-audios`
  - JSON body: `{ "audio_urls": <array> }` from `{{$json.urls}}`
  - Auth: header auth credential (“Fal.run API”)
  - Sends `Content-Type: application/json`
- **Connections:**
  - Input ← **Set AudioUrls Json**
  - Output → **Wait 30 sec.**
- **Edge cases / failures:**
  - **API limit noted in sticky note:** merge API limit is **5 audios** per request. Sending more than 5 URLs will fail unless you implement chunked merges.
  - Invalid URLs or unreachable audio files cause merge job failure
  - Fal.run auth failure / quota limits

---

### 2.4 Polling for merge completion

**Overview:** Waits and polls the Fal.run queue status endpoint until the merge request is completed.

**Nodes involved**
- Wait 30 sec.
- Get status
- Completed?

#### Node: Wait 30 sec.
- **Type / role:** Wait node; delay between polling attempts.
- **Configuration choices:** Wait 30 seconds.
- **Connections:**
  - Input ← **Merge Audios** OR (loop) ← **Completed?** (false branch)
  - Output → **Get status**
- **Edge cases:** Excessive polling can slow executions; too infrequent polling delays completion.

#### Node: Get status
- **Type / role:** HTTP Request; checks job status for a merge request id.
- **Configuration choices:**
  - `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Merge Audios').item.json.request_id }}/status`
  - Auth: Fal.run header auth
  - Query parameter includes `Content-Type=application/json` (unusual placement; typically should be a header—yet may still work if ignored).
- **Connections:**
  - Input ← **Wait 30 sec.**
  - Output → **Completed?**
- **Edge cases / failures:**
  - If `request_id` is missing (merge call failed), expression resolution fails
  - Network timeouts or transient 5xx from Fal.run

#### Node: Completed?
- **Type / role:** IF node; checks whether `$json.status === "COMPLETED"`.
- **Configuration choices:**
  - Strict equals condition on `status`.
- **Connections:**
  - True → **Get final audio url**
  - False → **Wait 30 sec.** (poll loop)
- **Edge cases:**
  - Fal.run may use other terminal statuses (e.g., `FAILED`). This workflow doesn’t explicitly handle failure states; it will keep waiting unless status changes to `COMPLETED`.

---

### 2.5 Download & upload final audiobook

**Overview:** Fetches merged audio metadata, downloads the resulting file, then uploads to a Google Drive folder using a timestamped name.

**Nodes involved**
- Get final audio url
- Get File
- Upload Audiobook

#### Node: Get final audio url
- **Type / role:** HTTP Request; retrieves final merge result data.
- **Configuration choices:**
  - `GET https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
  - Auth: Fal.run header auth
- **Connections:**
  - Input ← **Completed?** (true)
  - Output → **Get File**
- **Edge cases:** If request finished but output is not yet persisted, `audio.url` may be missing briefly.

#### Node: Get File
- **Type / role:** HTTP Request; downloads the merged audio file from `{{$json.audio.url}}`.
- **Configuration choices:** Simple GET; no explicit “download as binary” option shown (in some n8n versions you must enable binary download for Drive upload).
- **Connections:**
  - Input ← **Get final audio url**
  - Output → **Upload Audiobook**
- **Edge cases / failures:**
  - If the node does not output **binary data**, the Google Drive upload may fail or upload JSON instead of audio
  - Signed URLs can expire if polling is slow

#### Node: Upload Audiobook
- **Type / role:** Google Drive; uploads final file to a target folder.
- **Configuration choices:**
  - Filename: `{{$now.format('yyyyLLddHHmmss')}}-{{$json.audio.file_name}}`
  - Drive: “My Drive”
  - Folder ID: `1aHRwLWyrqfzoVC8HoB-YMrBvQ4tLC-NZ` (named “Fal.run” in cached result)
- **Connections:**
  - Input ← **Get File**
- **Credentials:** Google Drive OAuth2 (`Google Drive account (n3w.it)`)
- **Edge cases / failures:**
  - OAuth permission errors (no write access to folder)
  - If incoming data isn’t binary, upload fails or produces corrupted file
  - Large files may hit upload limits/timeouts

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get scripts |  |
| Get scripts | Google Sheets | Read script rows from spreadsheet | When clicking ‘Execute workflow’ | Loop Over Items | ## STEP 1 - Clone the Sheet; **Double click** to edit me. [Clone this Sheet](https://docs.google.com/spreadsheets/d/1f4rB-i3cVDzKLi6nv8EMvjjyg9n4rgkOVx28_EI1NBI/edit?usp=sharing) and fill "Text", "Speaker", "Voice Description" and "Style Instruction" |
| Loop Over Items | Split In Batches | Iterate rows in controlled batches | Get scripts; Wait 10 sec. | Set AudioUrls Json; Voice Design |  |
| Voice Design | HTTP Request | Generate TTS segment via Replicate Qwen3-TTS | Loop Over Items | Update Temp URL | ## STEP 2 - Clone the Sheet; Configure [Replicate API](https://replicate.com) credentials for Qwen3-TTS voice synthesis. Ensure your spreadsheet contains appropriate voice descriptions and style instructions compatible with the Qwen3-TTS model's requirements |
| Update Temp URL | Google Sheets | Write generated audio URL back to the row | Voice Design | Wait 10 sec. |  |
| Wait 10 sec. | Wait | Throttle between per-row generations | Update Temp URL | Loop Over Items |  |
| Set AudioUrls Json | Code | Build array of Temp URL values | Loop Over Items | Merge Audios | ## STEP 3 - Merge audios; Set up Fal.run API credentials for FFmpeg audio merging operations; IMPORTANT: Merge API limit: 5 audios. To overcome this limit, you can use a new Loop and a Code node to merge 5 audios at once and then merge them together. This allows you to potentially create very long audiobooks. |
| Merge Audios | HTTP Request | Submit async merge request to Fal.run | Set AudioUrls Json | Wait 30 sec. | ## STEP 3 - Merge audios; Set up Fal.run API credentials for FFmpeg audio merging operations; IMPORTANT: Merge API limit: 5 audios. To overcome this limit, you can use a new Loop and a Code node to merge 5 audios at once and then merge them together. This allows you to potentially create very long audiobooks. |
| Wait 30 sec. | Wait | Delay between merge status checks | Merge Audios; Completed? (false) | Get status | ## STEP 3 - Merge audios; Set up Fal.run API credentials for FFmpeg audio merging operations; IMPORTANT: Merge API limit: 5 audios. To overcome this limit, you can use a new Loop and a Code node to merge 5 audios at once and then merge them together. This allows you to potentially create very long audiobooks. |
| Get status | HTTP Request | Poll Fal.run merge request status | Wait 30 sec. | Completed? | ## STEP 3 - Merge audios; Set up Fal.run API credentials for FFmpeg audio merging operations; IMPORTANT: Merge API limit: 5 audios. To overcome this limit, you can use a new Loop and a Code node to merge 5 audios at once and then merge them together. This allows you to potentially create very long audiobooks. |
| Completed? | IF | Check if merge job is COMPLETED | Get status | Get final audio url (true); Wait 30 sec. (false) | ## STEP 3 - Merge audios; Set up Fal.run API credentials for FFmpeg audio merging operations; IMPORTANT: Merge API limit: 5 audios. To overcome this limit, you can use a new Loop and a Code node to merge 5 audios at once and then merge them together. This allows you to potentially create very long audiobooks. |
| Get final audio url | HTTP Request | Retrieve final merged audio metadata | Completed? (true) | Get File | ## STEP 3 - Merge audios; Set up Fal.run API credentials for FFmpeg audio merging operations; IMPORTANT: Merge API limit: 5 audios. To overcome this limit, you can use a new Loop and a Code node to merge 5 audios at once and then merge them together. This allows you to potentially create very long audiobooks. |
| Get File | HTTP Request | Download merged audio file | Get final audio url | Upload Audiobook | ## STEP 4 - Google Drive; Upload the final file on Google Drive |
| Upload Audiobook | Google Drive | Upload final audiobook to Drive | Get File | — | ## STEP 4 - Google Drive; Upload the final file on Google Drive |
| Sticky Note1 | Sticky Note | Workflow description and setup notes | — | — | ## Create Long Audiobooks with Custom Voices using Qwen3-TTS Voice Design; [Click here to listen](https://iframe.mediadelivery.net/play/580928/5e137292-5d19-4745-9d79-aeb3cc5f5a23) the result of my example. |
| Sticky Note2 | Sticky Note | Step 1 notes (sheet clone) | — | — | ## STEP 1 - Clone the Sheet; [Clone this Sheet](https://docs.google.com/spreadsheets/d/1f4rB-i3cVDzKLi6nv8EMvjjyg9n4rgkOVx28_EI1NBI/edit?usp=sharing) |
| Sticky Note3 | Sticky Note | Step 2 notes (Replicate) | — | — | ## STEP 2 - Clone the Sheet; Configure [Replicate API](https://replicate.com) credentials |
| Sticky Note4 | Sticky Note | Step 3 notes (merge limit) | — | — | ## STEP 3 - Merge audios; IMPORTANT: Merge API limit: 5 audios... |
| Sticky Note5 | Sticky Note | Step 4 notes (Drive) | — | — | ## STEP 4 - Google Drive |
| Sticky Note9 | Sticky Note | Channel / resource link | — | — | ## MY NEW YOUTUBE CHANNEL; [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Audiobook with Qwen3-TTS Voice Design*.

2. **Add Trigger**
   1) Add node **Manual Trigger**  
   2) Name it: *When clicking ‘Execute workflow’*

3. **Google Sheets: read scripts**
   1) Add node **Google Sheets** named *Get scripts*  
   2) Connect: Manual Trigger → Get scripts  
   3) Set credentials: **Google Sheets OAuth2** (authorize access)  
   4) Select the spreadsheet (clone if desired):  
      - Link: https://docs.google.com/spreadsheets/d/1f4rB-i3cVDzKLi6nv8EMvjjyg9n4rgkOVx28_EI1NBI/edit?usp=sharing  
   5) Select sheet/tab: `gid=0` (“Foglio1”)  
   6) Ensure columns exist exactly as used by expressions:  
      - `Text`, `Speaker`, `Voice Description`, `Style Instruction`, `Temp URL`, `To Merge`, and a `row_number` field (provided by n8n’s Sheets connector when reading rows).
   7) (Optional but recommended) Configure filtering to fetch only rows that need processing (e.g., `Temp URL` is empty).

4. **Add loop control**
   1) Add node **Split In Batches** named *Loop Over Items*  
   2) Connect: Get scripts → Loop Over Items  
   3) Set batch size (recommended): **1** to reduce rate-limit risk.

5. **Replicate TTS request**
   1) Add node **HTTP Request** named *Voice Design*  
   2) Connect: Loop Over Items → Voice Design  
   3) Configure:
      - Method: **POST**
      - URL: `https://api.replicate.com/v1/models/qwen/qwen3-tts/predictions`
      - Authentication: **Bearer Auth** (create credential “Replicate” with your token)
      - Headers: `Prefer: wait`
      - Body type: JSON
      - Body:
        - `input.mode` = `voice_design`
        - `input.text` = `{{$json.Text}}`
        - `input.language` = `auto`
        - `input.speaker` = `{{$json.Speaker}}`
        - `input.voice_description` = `{{$json["Voice Description"]}}`
        - `input.style_instruction` = `{{$json["Style Instruction"]}}`

6. **Write back Temp URL to the sheet**
   1) Add node **Google Sheets** named *Update Temp URL*  
   2) Connect: Voice Design → Update Temp URL  
   3) Operation: **Update**  
   4) Matching column: `row_number`  
   5) Map fields:
      - `Temp URL` = `{{$json.output}}`
      - `To Merge` = `x`
      - `row_number` = `{{$('Loop Over Items').item.json.row_number}}`
   6) Use the same Sheet document + tab as in “Get scripts”.

7. **Throttle and continue the loop**
   1) Add node **Wait** named *Wait 10 sec.* with Amount = **10 seconds**  
   2) Connect: Update Temp URL → Wait 10 sec.  
   3) Connect: Wait 10 sec. → Loop Over Items (this is the “continue” loop connection)

8. **Prepare URL list for merging**
   1) Add node **Code** named *Set AudioUrls Json*  
   2) Connect: Loop Over Items → Set AudioUrls Json  
   3) Code:
      - Build one item with `urls` array from `Temp URL`:
        - `urls: items.map(item => item.json["Temp URL"])`

9. **Fal.run merge request**
   1) Add node **HTTP Request** named *Merge Audios*  
   2) Connect: Set AudioUrls Json → Merge Audios  
   3) Configure:
      - Method: **POST**
      - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/merge-audios`
      - Auth: **Header Auth** (create credential “Fal.run API”, typically `Authorization: Key <...>` depending on Fal.run requirements)
      - Header: `Content-Type: application/json`
      - JSON Body: `{ "audio_urls": <urls array> }` using the node’s expression for `{{$json.urls}}`
   4) Constraint: Fal.run merge limit is **5 audios** per request; implement chunking if you exceed this.

10. **Polling loop for merge completion**
   1) Add node **Wait** named *Wait 30 sec.* with Amount = **30 seconds**  
   2) Connect: Merge Audios → Wait 30 sec.
   3) Add node **HTTP Request** named *Get status*  
      - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $('Merge Audios').item.json.request_id }}/status`
      - Auth: Fal.run header auth
   4) Connect: Wait 30 sec. → Get status
   5) Add node **IF** named *Completed?*  
      - Condition: `{{$json.status}}` equals `COMPLETED`
   6) Connect: Get status → Completed?
   7) Connect false branch: Completed? (false) → Wait 30 sec. (to keep polling)

11. **Fetch final result, download file, upload to Drive**
   1) Add node **HTTP Request** named *Get final audio url*  
      - URL: `https://queue.fal.run/fal-ai/ffmpeg-api/requests/{{ $json.request_id }}`
      - Auth: Fal.run header auth
   2) Connect: Completed? (true) → Get final audio url
   3) Add node **HTTP Request** named *Get File*  
      - URL: `{{$json.audio.url}}`
      - Configure to download as **binary** if your n8n version requires it for Drive uploads.
   4) Connect: Get final audio url → Get File
   5) Add node **Google Drive** named *Upload Audiobook*  
      - Credentials: Google Drive OAuth2
      - Folder: set the target folder ID (template uses `1aHRwLWyrqfzoVC8HoB-YMrBvQ4tLC-NZ`)
      - File name: `{{$now.format('yyyyLLddHHmmss')}}-{{$json.audio.file_name}}`
   6) Connect: Get File → Upload Audiobook

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Example output: “Click here to listen” | https://iframe.mediadelivery.net/play/580928/5e137292-5d19-4745-9d79-aeb3cc5f5a23 |
| Clone the spreadsheet template and fill required columns | https://docs.google.com/spreadsheets/d/1f4rB-i3cVDzKLi6nv8EMvjjyg9n4rgkOVx28_EI1NBI/edit?usp=sharing |
| Replicate platform (Qwen3-TTS) | https://replicate.com |
| YouTube channel link from sticky note | https://youtube.com/@n3witalia |
| Important constraint: Fal.run merge API limit is 5 audios per merge request; use chunked merging (merge groups of 5, then merge results). | Mentioned in “STEP 3 - Merge audios” sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.