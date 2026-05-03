Repurpose YouTube Shorts to TikTok and Instagram using Blotato

https://n8nworkflows.xyz/workflows/repurpose-youtube-shorts-to-tiktok-and-instagram-using-blotato-13954


# Repurpose YouTube Shorts to TikTok and Instagram using Blotato

# 1. Workflow Overview

This workflow automates the repurposing of YouTube Shorts to TikTok and Instagram using Blotato, while using Google Sheets as a lightweight tracking database and `yt-dlp` as the local download engine.

Its main use case is for creators or teams who want to:
- periodically detect recent videos from a YouTube channel,
- register them in Google Sheets,
- identify unprocessed items,
- download the matching Shorts locally,
- upload the media to Blotato,
- publish to TikTok and Instagram,
- mark success or failure in the spreadsheet,
- and clean up local files afterward.

The workflow is split into two scheduled branches:

## 1.1 Video Discovery and Registration
This branch runs on a schedule, fetches the latest YouTube videos from a configured channel, transforms them into a normalized structure, and inserts or updates them in Google Sheets.

## 1.2 Pending Item Selection
This branch runs on a separate schedule, reads the spreadsheet, and selects rows that are eligible for processing based on their status.

## 1.3 Local Download and File Resolution
For each selected item, the workflow downloads the YouTube Short with `yt-dlp`, locates the actual downloaded file on disk, and confirms that a file exists before continuing.

## 1.4 Blotato Upload and Cross-Platform Posting
The local video is read as binary, uploaded to Blotato as a media asset, then used to create posts for TikTok and Instagram using the video title as post text.

## 1.5 Cleanup and Status Logging
After posting, the workflow deletes the local file and updates Google Sheets with either a success or error status. Error handling is attached to the posting nodes so failed post attempts are still recorded.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Video Discovery and Registration

### Overview
This block fetches recent YouTube videos from a channel through the YouTube Data API, splits the response into individual items, converts each item to a spreadsheet-friendly structure, and upserts the result into Google Sheets.

### Nodes Involved
- Schedule Trigger Get Youtube Video List
- Get Youtube Video List
- Split Out
- Loop List
- Data for Store Video List to Sheets
- Insert List to Sheets

### Node Details

#### 2.1.1 Schedule Trigger Get Youtube Video List
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; starts the discovery branch automatically.
- **Configuration choices:** Runs every 8 hours.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point; outputs to **Get Youtube Video List**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Workflow will not run unless activated.
  - Schedule timing depends on server timezone/configuration.

#### 2.1.2 Get Youtube Video List
- **Type and role:** `n8n-nodes-base.httpRequest`; queries the YouTube Data API search endpoint.
- **Configuration choices:**
  - Uses a GET request against:
    `https://www.googleapis.com/youtube/v3/search`
  - Query parameters are embedded directly in the URL:
    - `key={{$env.YOUTUBE_API_KEY}}`
    - `part=snippet`
    - `channelId={{$env.YOUTUBE_CHANNEL_ID}}`
    - `type=video`
    - `order=date`
    - `maxResults=10`
- **Key expressions or variables used:**
  - `$env.YOUTUBE_API_KEY`
  - `$env.YOUTUBE_CHANNEL_ID`
- **Input and output connections:** Input from **Schedule Trigger Get Youtube Video List**; output to **Split Out**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases / failures:**
  - Missing or invalid environment variables.
  - YouTube API quota exhaustion.
  - Invalid API key or unauthorized API access.
  - Returned items may include non-Shorts videos; this workflow assumes links will be usable as Shorts URLs.

#### 2.1.3 Split Out
- **Type and role:** `n8n-nodes-base.splitOut`; expands the `items` array from the YouTube API response into one item per execution item.
- **Configuration choices:** `fieldToSplitOut = items`.
- **Key expressions or variables used:** `items`.
- **Input and output connections:** Input from **Get Youtube Video List**; output to **Loop List**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - If the API response does not contain `items`, this node may output nothing or fail depending on runtime data shape.

#### 2.1.4 Loop List
- **Type and role:** `n8n-nodes-base.splitInBatches`; iterates through discovered videos one by one.
- **Configuration choices:** Default batch behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Split Out** and feedback from **Insert List to Sheets**; main processing output goes to **Data for Store Video List to Sheets**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - With no incoming items, nothing is processed.
  - Looping behavior depends on proper reconnection from downstream nodes.

#### 2.1.5 Data for Store Video List to Sheets
- **Type and role:** `n8n-nodes-base.set`; normalizes YouTube response fields into the sheet schema.
- **Configuration choices:** Creates these fields:
  - `id = {{ $json.id.videoId }}`
  - `url = https://youtube.com/shorts/{{ $json.id.videoId }}`
  - `title = {{ $json.snippet.title }}`
  - `caption = {{ $json.snippet.description }}`
  - `status = not-processed`
  - `lastupdate = {{$now}}`
- **Key expressions or variables used:**
  - `$json.id.videoId`
  - `$json.snippet.title`
  - `$json.snippet.description`
  - `$now`
- **Input and output connections:** Input from **Loop List**; output to **Insert List to Sheets**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - Missing `id.videoId` causes malformed records.
  - Missing `snippet` fields can produce blank title/caption values.

#### 2.1.6 Insert List to Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`; writes or updates video records in the spreadsheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `id`
  - Auto-maps input into a schema with:
    - `id`
    - `url`
    - `title`
    - `caption`
    - `status`
    - `lastupdate`
- **Key expressions or variables used:** Uses incoming fields from the Set node.
- **Input and output connections:** Input from **Data for Store Video List to Sheets**; output loops back to **Loop List**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Missing Google Sheets credentials.
  - Invalid document or sheet selection.
  - Duplicate handling depends on the `id` match column.
  - If the target sheet columns do not align, writes may fail or map incorrectly.
- **Sub-workflow reference:** None.

---

## 2.2 Block: Pending Item Selection

### Overview
This block reads the tracking spreadsheet on a schedule and passes rows into the processing loop. It then filters items according to status before downloading.

### Nodes Involved
- Schedule Trigger Upload
- Read from Sheets
- Filter by Status Not Processed
- Loop

### Node Details

#### 2.2.1 Schedule Trigger Upload
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; starts the upload branch.
- **Configuration choices:** Runs every 8 hours.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry point; outputs to **Read from Sheets**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Same schedule timing caveats as the other trigger.
  - Both scheduled branches may run independently and can overlap.

#### 2.2.2 Read from Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`; reads rows from the configured spreadsheet.
- **Configuration choices:** Uses the specified document and sheet, with default options.
- **Key expressions or variables used:** None visible in the JSON.
- **Input and output connections:** Input from **Schedule Trigger Upload**; output to **Filter by Status Not Processed**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Credential or permission errors.
  - Empty sheet returns no items.
  - Header mismatch may affect field names.

#### 2.2.3 Filter by Status Not Processed
- **Type and role:** `n8n-nodes-base.if`; status gate before processing.
- **Configuration choices:**
  - Condition: `{{ $json.status }} notEquals "not-processed"`
- **Key expressions or variables used:**
  - `$json.status`
- **Input and output connections:** Input from **Read from Sheets**; true output goes to **Loop**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - The node name suggests it should keep only `not-processed` rows, but the actual condition is the opposite: `status != not-processed`.
  - This is likely a logic mismatch and may cause already-processed or errored rows to be selected instead of new ones.
- **Important implementation note:** If the intended behavior is to process only fresh videos, this condition should probably be changed to `equals "not-processed"`.

#### 2.2.4 Loop
- **Type and role:** `n8n-nodes-base.splitInBatches`; iterates over selected sheet rows one by one.
- **Configuration choices:** Default batch settings.
- **Key expressions or variables used:** Referenced later by other nodes as `$('Loop').item.json...`.
- **Input and output connections:** Input from **Filter by Status Not Processed** and feedback from **Check File Video Exist** false path; main processing output goes to **Download YouTube Shorts via yt-dlp**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - If upstream filtering is wrong, this loop will process the wrong records.
  - Batch size defaults may affect throughput.

---

## 2.3 Block: Local Download and File Resolution

### Overview
This block downloads each selected Short to the local server, locates the file generated by `yt-dlp`, and verifies that a path was returned before trying to read the file.

### Nodes Involved
- Download YouTube Shorts via yt-dlp
- Get File Path Location
- Check File Video Exist
- Read Binary Video

### Node Details

#### 2.3.1 Download YouTube Shorts via yt-dlp
- **Type and role:** `n8n-nodes-base.executeCommand`; invokes `yt-dlp` on the host machine.
- **Configuration choices:**
  - Command:
    `yt-dlp -f best -o "youtube/{{ $json.id }}.%(ext)s" "https://youtube.com/shorts/{{ $json.id }}"`
- **Key expressions or variables used:**
  - `$json.id`
- **Input and output connections:** Input from **Loop**; output to **Get File Path Location**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - Requires self-hosted n8n.
  - `yt-dlp` must be installed and available in the shell PATH.
  - The command writes to `youtube/`, but one sticky note mentions creating a `youtube` directory while another command later searches in `yt`; that mismatch is important.
  - Download can fail for geo restrictions, removed content, age restrictions, rate limits, or ffmpeg-related format issues.

#### 2.3.2 Get File Path Location
- **Type and role:** `n8n-nodes-base.executeCommand`; searches the filesystem for the downloaded file.
- **Configuration choices:**
  - Command:
    `find yt -type f -name "<videoId>*"`
  - Uses the current item from **Loop**:
    `{{$node['Loop'].json.id}}`
- **Key expressions or variables used:**
  - `$node['Loop'].json.id`
- **Input and output connections:** Input from **Download YouTube Shorts via yt-dlp**; output to **Check File Video Exist**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - There is a likely directory mismatch:
    - download path uses `youtube/`
    - search path uses `yt`
  - If not corrected, the node may fail to find downloaded files even when the download succeeded.
  - Multiple matching files could return multiple lines in stdout.

#### 2.3.3 Check File Video Exist
- **Type and role:** `n8n-nodes-base.if`; checks whether a file path exists in command output.
- **Configuration choices:**
  - Condition: `{{ $json.stdout }}` exists
- **Key expressions or variables used:**
  - `$json.stdout`
- **Input and output connections:**
  - True output to **Read Binary Video**
  - False output back to **Loop**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - If `stdout` contains whitespace only, behavior may still count as existing depending on execution result.
  - False path silently continues to the next item without marking an error in Sheets.

#### 2.3.4 Read Binary Video
- **Type and role:** `n8n-nodes-base.readWriteFile`; reads the located file into binary data for upload.
- **Configuration choices:**
  - File selector:
    `{{ $json.stdout }}`
- **Key expressions or variables used:**
  - `$json.stdout`
- **Input and output connections:** Input from **Check File Video Exist**; output to **Upload Media to Blotato**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - If `stdout` contains a newline or multiple paths, file read may fail.
  - Permissions on the file or directory may block access.

---

## 2.4 Block: Blotato Upload and Cross-Platform Posting

### Overview
This block uploads the binary media file to Blotato, receives a hosted media URL, and then uses that URL to create one TikTok post and one Instagram post.

### Nodes Involved
- Upload Media to Blotato
- Blotato Post to Tiktok
- Blotato Post to Instagram

### Node Details

#### 2.4.1 Upload Media to Blotato
- **Type and role:** `@blotato/n8n-nodes-blotato.blotato`; uploads local binary media to Blotato.
- **Configuration choices:**
  - Resource: `media`
  - `useBinaryData = true`
- **Key expressions or variables used:** Consumes binary input from previous node.
- **Input and output connections:** Input from **Read Binary Video**; outputs to both **Blotato Post to Tiktok** and **Blotato Post to Instagram**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Requires Blotato credentials.
  - Upload can fail due to file size, unsupported format, auth errors, or API downtime.
  - Downstream nodes expect the upload response to contain `url`.

#### 2.4.2 Blotato Post to Tiktok
- **Type and role:** `@blotato/n8n-nodes-blotato.blotato`; creates a TikTok post from the uploaded media URL.
- **Configuration choices:**
  - `platform = "tiktok"`
  - Post text:
    `{{ $('Loop').item.json.title }}`
  - Media URL:
    `{{ $json.url }}`
  - Retry on fail enabled
  - `maxTries = 2`
  - `onError = continueErrorOutput`
- **Key expressions or variables used:**
  - `$('Loop').item.json.title`
  - `$json.url`
- **Input and output connections:**
  - Main success output to **Remove Local File**
  - Error output to **Data Status for Update Sheets Error**
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Missing or invalid account selection in Blotato.
  - TikTok platform restrictions, media validation, caption length, or temporary API errors.
  - Because `continueErrorOutput` is used, failures do not stop the workflow; they are routed for logging instead.

#### 2.4.3 Blotato Post to Instagram
- **Type and role:** `@blotato/n8n-nodes-blotato.blotato`; creates an Instagram post from the uploaded media URL.
- **Configuration choices:**
  - Post text:
    `{{ $('Loop').item.json.title }}`
  - Media URL:
    `{{ $json.url }}`
  - Retry on fail enabled
  - `maxTries = 2`
  - `onError = continueErrorOutput`
- **Key expressions or variables used:**
  - `$('Loop').item.json.title`
  - `$json.url`
- **Input and output connections:**
  - Main success output to **Remove Local File**
  - Error output to **Data Status for Update Sheets Error**
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Same category of auth/account/API/media issues as TikTok.
  - If one platform succeeds and the other fails, the workflow may both remove the file and also mark an error depending on execution timing.
- **Important design note:** Both posting nodes run in parallel from the media upload result and both connect into the same cleanup/error paths. This can create duplicate downstream executions.

---

## 2.5 Block: Cleanup and Status Logging

### Overview
This block removes the local file after posting and updates Google Sheets with either `done-uploaded` or `error`. It relies on data carried from prior nodes and uses the original loop item to identify the row by video ID.

### Nodes Involved
- Remove Local File
- Data Status for Update Sheets Success
- Update Sheets Success
- Data Status for Update Sheets Error
- Update Sheets Error

### Node Details

#### 2.5.1 Remove Local File
- **Type and role:** `n8n-nodes-base.executeCommand`; deletes the downloaded video file from local storage.
- **Configuration choices:**
  - Command:
    `rm {{ $('Get File Path Location').item.json.stdout }}`
- **Key expressions or variables used:**
  - `$('Get File Path Location').item.json.stdout`
- **Input and output connections:** Inputs from both **Blotato Post to Tiktok** and **Blotato Post to Instagram** success outputs; output to **Data Status for Update Sheets Success**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:**
  - If both social posts succeed, this node may run twice for the same file.
  - The second `rm` may fail because the file is already deleted.
  - If the path contains whitespace or newline characters, shell parsing can fail.
  - Using shell commands with interpolated paths requires careful sanitization.

#### 2.5.2 Data Status for Update Sheets Success
- **Type and role:** `n8n-nodes-base.set`; prepares a simple payload for success logging.
- **Configuration choices:**
  - `status = "done-uploaded"`
  - `id = {{ $('Loop').item.json.id }}`
- **Key expressions or variables used:**
  - `$('Loop').item.json.id`
- **Input and output connections:** Input from **Remove Local File**; output to **Update Sheets Success**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If item linking breaks in parallelized branches, it may reference the wrong loop item in unusual execution patterns.

#### 2.5.3 Update Sheets Success
- **Type and role:** `n8n-nodes-base.googleSheets`; writes success status back to the tracking sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `id`
  - Explicit values:
    - `id = {{ $json.id }}`
    - `status = {{ $json.status }}`
    - `lastupdate = {{$now}}`
- **Key expressions or variables used:**
  - `$json.id`
  - `$json.status`
  - `$now`
- **Input and output connections:** Input from **Data Status for Update Sheets Success**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Google Sheets credential or sheet config issues.
  - Duplicate success writes possible if both posting nodes trigger cleanup independently.

#### 2.5.4 Data Status for Update Sheets Error
- **Type and role:** `n8n-nodes-base.set`; prepares a simple payload for error logging.
- **Configuration choices:**
  - `status = "error"`
  - `id = {{ $('Loop').item.json.id }}`
- **Key expressions or variables used:**
  - `$('Loop').item.json.id`
- **Input and output connections:** Inputs from error outputs of **Blotato Post to Tiktok** and **Blotato Post to Instagram**; output to **Update Sheets Error**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases / failures:**
  - If one platform fails and the other succeeds, the sheet may receive both success and error updates for the same ID, depending on timing and final write order.

#### 2.5.5 Update Sheets Error
- **Type and role:** `n8n-nodes-base.googleSheets`; writes error status back to the sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Matching column: `id`
  - Explicit values:
    - `id = {{ $json.id }}`
    - `status = {{ $json.status }}`
    - `lastupdate = {{$now}}`
- **Key expressions or variables used:**
  - `$json.id`
  - `$json.status`
  - `$now`
- **Input and output connections:** Input from **Data Status for Update Sheets Error**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases / failures:**
  - Same Sheet issues as other Google Sheets nodes.
  - Final state may be nondeterministic if multiple updates occur for the same row close together.

---

## 2.6 Informational / Annotation Nodes

These nodes do not affect execution logic but provide documentation in the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Notes
- They contain usage explanations, grouping labels, and infrastructure requirements.
- They should be preserved when rebuilding the workflow if visual maintainability matters.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger Get Youtube Video List | n8n-nodes-base.scheduleTrigger | Scheduled entry point for YouTube discovery |  | Get Youtube Video List | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Get Youtube Video List | n8n-nodes-base.httpRequest | Calls YouTube Data API to fetch latest channel videos | Schedule Trigger Get Youtube Video List | Split Out | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Split Out | n8n-nodes-base.splitOut | Splits YouTube API `items` array into individual records | Get Youtube Video List | Loop List | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Loop List | n8n-nodes-base.splitInBatches | Iterates through discovered video records | Split Out, Insert List to Sheets | Data for Store Video List to Sheets | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Data for Store Video List to Sheets | n8n-nodes-base.set | Maps YouTube response fields into sheet schema | Loop List | Insert List to Sheets | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Insert List to Sheets | n8n-nodes-base.googleSheets | Upserts discovered videos into Google Sheets by ID | Data for Store Video List to Sheets | Loop List | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Schedule Trigger Upload | n8n-nodes-base.scheduleTrigger | Scheduled entry point for processing/upload branch |  | Read from Sheets | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Read from Sheets | n8n-nodes-base.googleSheets | Reads tracked video rows from Google Sheets | Schedule Trigger Upload | Filter by Status Not Processed | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Filter by Status Not Processed | n8n-nodes-base.if | Filters rows by status before processing | Read from Sheets | Loop | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Loop | n8n-nodes-base.splitInBatches | Iterates through selected rows for download/upload | Filter by Status Not Processed, Check File Video Exist | Download YouTube Shorts via yt-dlp | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Download YouTube Shorts via yt-dlp | n8n-nodes-base.executeCommand | Downloads the YouTube Short locally with `yt-dlp` | Loop | Get File Path Location | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Get File Path Location | n8n-nodes-base.executeCommand | Locates the downloaded file path on disk | Download YouTube Shorts via yt-dlp | Check File Video Exist | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Check File Video Exist | n8n-nodes-base.if | Verifies that a file path was found | Get File Path Location | Read Binary Video, Loop | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Read Binary Video | n8n-nodes-base.readWriteFile | Reads local file into binary for upload | Check File Video Exist | Upload Media to Blotato | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Upload Media to Blotato | @blotato/n8n-nodes-blotato.blotato | Uploads binary media to Blotato | Read Binary Video | Blotato Post to Tiktok, Blotato Post to Instagram | ## Upload File to Blotato then Post to Tiktok & Instagram |
| Blotato Post to Tiktok | @blotato/n8n-nodes-blotato.blotato | Publishes uploaded media to TikTok | Upload Media to Blotato | Remove Local File, Data Status for Update Sheets Error | ## Upload File to Blotato then Post to Tiktok & Instagram |
| Blotato Post to Instagram | @blotato/n8n-nodes-blotato.blotato | Publishes uploaded media to Instagram | Upload Media to Blotato | Remove Local File, Data Status for Update Sheets Error | ## Upload File to Blotato then Post to Tiktok & Instagram |
| Remove Local File | n8n-nodes-base.executeCommand | Deletes local downloaded video after posting | Blotato Post to Tiktok, Blotato Post to Instagram | Data Status for Update Sheets Success | ## Cleanup Local File & Update Status in Sheets |
| Data Status for Update Sheets Success | n8n-nodes-base.set | Builds success status payload | Remove Local File | Update Sheets Success | ## Cleanup Local File & Update Status in Sheets |
| Update Sheets Success | n8n-nodes-base.googleSheets | Writes success status and timestamp to sheet | Data Status for Update Sheets Success |  | ## Cleanup Local File & Update Status in Sheets |
| Data Status for Update Sheets Error | n8n-nodes-base.set | Builds error status payload | Blotato Post to Tiktok, Blotato Post to Instagram | Update Sheets Error | ## Cleanup Local File & Update Status in Sheets |
| Update Sheets Error | n8n-nodes-base.googleSheets | Writes error status and timestamp to sheet | Data Status for Update Sheets Error |  | ## Cleanup Local File & Update Status in Sheets |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation for discovery block |  |  | ## Grab Youtube Video List, Store to Google Spreadsheet |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation for local download block |  |  | ## Download Video from Youtube with yt-dlp |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas annotation for Blotato posting block |  |  | ## Upload File to Blotato then Post to Tiktok & Instagram |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas annotation for cleanup/status block |  |  | ## Cleanup Local File & Update Status in Sheets |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | It fetches new videos from your YouTube channel, checks whether they have already been processed, downloads them locally using yt-dlp, and then uploads or schedules them via Blotato. Once successfully published, the workflow logs everything into Google Sheets and removes the local file to keep your storage clean. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | This template is designed for creators, agencies, and social media managers who want a simple and reliable way to repurpose short-form content across multiple platforms. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | ### ⚙️ How It Works |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 1. The workflow runs on a schedule (e.g., every 8 hours). |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 2. It retrieves the latest videos from your YouTube channel. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 3. It checks Google Sheets to avoid duplicate uploads by Video ID. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 4. New Shorts are downloaded using yt-dlp. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 5. The video is uploaded (or scheduled) via Blotato to TikTok and Instagram. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 6. The workflow logs the upload results. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | 7. The local file is deleted after a successful upload. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | ### 🛠 Requirements |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | - A YouTube Data API key |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | - A Google Sheets connection |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | - Blotato API credentials |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | - Self-hosted n8n instance (required for yt-dlp execution) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | - yt-dlp installed on your server |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation with workflow description and requirements |  |  | - Make 'youtube' directory for Download location |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## Prerequisites
1. Use a **self-hosted n8n instance** because the workflow relies on `Execute Command`.
2. Install **yt-dlp** on the n8n host and ensure it is available from the shell.
3. Create a local folder for downloads, preferably:
   - `youtube/`
4. Prepare these credentials/integrations:
   - **Google Sheets** credential in n8n
   - **Blotato** credential in n8n
5. Set these environment variables on the n8n server:
   - `YOUTUBE_API_KEY`
   - `YOUTUBE_CHANNEL_ID`
6. Create a Google Sheet with columns:
   - `id`
   - `url`
   - `title`
   - `caption`
   - `status`
   - `lastupdate`

## Build the discovery branch

1. **Create a Schedule Trigger** named **Schedule Trigger Get Youtube Video List**.
   - Set interval to every **8 hours**.

2. **Add an HTTP Request** node named **Get Youtube Video List**.
   - Method: GET
   - URL:
     `https://www.googleapis.com/youtube/v3/search?key={{$env.YOUTUBE_API_KEY}}&part=snippet&channelId={{$env.YOUTUBE_CHANNEL_ID}}&type=video&order=date&maxResults=10`
   - Connect **Schedule Trigger Get Youtube Video List → Get Youtube Video List**.

3. **Add a Split Out** node named **Split Out**.
   - Field to split out: `items`
   - Connect **Get Youtube Video List → Split Out**.

4. **Add a Split In Batches** node named **Loop List**.
   - Keep default options.
   - Connect **Split Out → Loop List**.

5. **Add a Set** node named **Data for Store Video List to Sheets**.
   - Create fields:
     - `id` = `{{ $json.id.videoId }}`
     - `url` = `https://youtube.com/shorts/{{ $json.id.videoId }}`
     - `title` = `{{ $json.snippet.title }}`
     - `caption` = `{{ $json.snippet.description }}`
     - `status` = `not-processed`
     - `lastupdate` = `{{ $now }}`
   - Connect **Loop List → Data for Store Video List to Sheets**.

6. **Add a Google Sheets** node named **Insert List to Sheets**.
   - Credential: your Google Sheets credential
   - Operation: **Append or Update**
   - Select the spreadsheet document and target sheet
   - Match by column: `id`
   - Schema columns:
     - `id`
     - `url`
     - `title`
     - `caption`
     - `status`
     - `lastupdate`
   - Use auto-mapping or explicit mapping aligned with incoming fields.
   - Connect **Data for Store Video List to Sheets → Insert List to Sheets**.

7. **Close the loop** for discovery.
   - Connect **Insert List to Sheets → Loop List** on the loop continuation input.

## Build the processing branch

8. **Create another Schedule Trigger** named **Schedule Trigger Upload**.
   - Set interval to every **8 hours**.

9. **Add a Google Sheets** node named **Read from Sheets**.
   - Credential: same Google Sheets credential
   - Select the same spreadsheet and sheet
   - Use default read behavior.
   - Connect **Schedule Trigger Upload → Read from Sheets**.

10. **Add an IF** node named **Filter by Status Not Processed**.
    - Configure condition on `{{ $json.status }}`
    - The imported workflow uses:
      - `notEquals` → `not-processed`
    - If you want the workflow to process only new items, change it to:
      - `equals` → `not-processed`
    - Connect **Read from Sheets → Filter by Status Not Processed**.

11. **Add a Split In Batches** node named **Loop**.
    - Keep default settings.
    - Connect the **true** output of **Filter by Status Not Processed** to **Loop**.

## Build the local download section

12. **Add an Execute Command** node named **Download YouTube Shorts via yt-dlp**.
    - Command:
      `yt-dlp -f best -o "youtube/{{ $json.id }}.%(ext)s" "https://youtube.com/shorts/{{ $json.id }}"`
    - Connect **Loop → Download YouTube Shorts via yt-dlp**.

13. **Add an Execute Command** node named **Get File Path Location**.
    - Important: fix the folder mismatch.
    - Recommended command:
      `find youtube -type f -name "{{$node['Loop'].json.id}}*"`
    - The imported workflow uses `find yt ...`, which likely does not match the actual download folder.
    - Connect **Download YouTube Shorts via yt-dlp → Get File Path Location**.

14. **Add an IF** node named **Check File Video Exist**.
    - Condition: field `{{ $json.stdout }}` **exists**
    - Connect **Get File Path Location → Check File Video Exist**.

15. **Add a Read/Write Files from Disk** node named **Read Binary Video**.
    - Operation: read file
    - File path:
      `{{ $json.stdout }}`
    - Connect the **true** output of **Check File Video Exist** to **Read Binary Video**.

16. **Connect the false branch** of **Check File Video Exist** back to **Loop**.
    - This skips files that were not found and moves to the next item.
    - Optional improvement: add an error status update instead of silent skip.

## Build the Blotato section

17. **Add a Blotato node** named **Upload Media to Blotato**.
    - Credential: your Blotato credential
    - Resource: **media**
    - Enable **Use Binary Data**
    - Connect **Read Binary Video → Upload Media to Blotato**.

18. **Add a Blotato node** named **Blotato Post to Tiktok**.
    - Credential: same Blotato credential
    - Platform: **tiktok**
    - Set account if required by your Blotato workspace
    - Post text:
      `{{ $('Loop').item.json.title }}`
    - Media URL:
      `{{ $json.url }}`
    - Enable retry on fail
    - Set max tries to **2**
    - Set error handling to **Continue Error Output**
    - Connect **Upload Media to Blotato → Blotato Post to Tiktok**.

19. **Add another Blotato node** named **Blotato Post to Instagram**.
    - Credential: same Blotato credential
    - Set Instagram account if required
    - Post text:
      `{{ $('Loop').item.json.title }}`
    - Media URL:
      `{{ $json.url }}`
    - Enable retry on fail
    - Set max tries to **2**
    - Set error handling to **Continue Error Output**
    - Connect **Upload Media to Blotato → Blotato Post to Instagram**.

## Build the success path

20. **Add an Execute Command** node named **Remove Local File**.
    - Recommended safer command:
      `rm "{{ $('Get File Path Location').item.json.stdout }}"`
    - The imported workflow does not quote the path; quoting is safer.
    - Connect success outputs from:
      - **Blotato Post to Tiktok**
      - **Blotato Post to Instagram**
      to **Remove Local File**.

21. **Add a Set** node named **Data Status for Update Sheets Success**.
    - Fields:
      - `status` = `done-uploaded`
      - `id` = `{{ $('Loop').item.json.id }}`
    - Connect **Remove Local File → Data Status for Update Sheets Success**.

22. **Add a Google Sheets** node named **Update Sheets Success**.
    - Credential: Google Sheets
    - Operation: **Append or Update**
    - Match by: `id`
    - Map:
      - `id` = `{{ $json.id }}`
      - `status` = `{{ $json.status }}`
      - `lastupdate` = `{{ $now }}`
    - Connect **Data Status for Update Sheets Success → Update Sheets Success**.

## Build the error path

23. **Add a Set** node named **Data Status for Update Sheets Error**.
    - Fields:
      - `status` = `error`
      - `id` = `{{ $('Loop').item.json.id }}`
    - Connect error outputs from:
      - **Blotato Post to Tiktok**
      - **Blotato Post to Instagram**
      to **Data Status for Update Sheets Error**.

24. **Add a Google Sheets** node named **Update Sheets Error**.
    - Credential: Google Sheets
    - Operation: **Append or Update**
    - Match by: `id`
    - Map:
      - `id` = `{{ $json.id }}`
      - `status` = `{{ $json.status }}`
      - `lastupdate` = `{{ $now }}`
    - Connect **Data Status for Update Sheets Error → Update Sheets Error**.

## Add documentation notes

25. **Add Sticky Notes** if you want the same visual organization:
   - `## Grab Youtube Video List, Store to Google Spreadsheet`
   - `## Download Video from Youtube with yt-dlp`
   - `## Upload File to Blotato then Post to Tiktok & Instagram`
   - `## Cleanup Local File & Update Status in Sheets`
   - The long explanatory note describing purpose, flow, and requirements

## Recommended corrections before activation

26. Fix these likely issues before turning the workflow on:
   - Change **Filter by Status Not Processed** from `notEquals not-processed` to `equals not-processed` if you only want unprocessed items.
   - Change **Get File Path Location** from `find yt ...` to `find youtube ...` unless your actual directory is `yt`.
   - Quote file paths in shell commands.
   - Consider adding a merge/wait strategy so cleanup and success logging happen only after both TikTok and Instagram posts succeed.
   - Consider separate status fields per platform if cross-platform partial success matters.

27. Save and activate the workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically mirrors your YouTube to TikTok and Instagram, so you don’t have to manually download and re-upload your content across platforms. | Sticky note context |
| It fetches new videos from your YouTube channel, checks whether they have already been processed, downloads them locally using yt-dlp, and then uploads or schedules them via Blotato. Once successfully published, the workflow logs everything into Google Sheets and removes the local file to keep your storage clean. | Sticky note context |
| This template is designed for creators, agencies, and social media managers who want a simple and reliable way to repurpose short-form content across multiple platforms. | Sticky note context |
| Requirements: YouTube Data API key, Google Sheets connection, Blotato API credentials, self-hosted n8n instance, yt-dlp installed, and a local `youtube` download directory. | Operational requirement |
| The workflow contains two independent schedule triggers, both set to run every 8 hours. | Execution design note |
| There is no sub-workflow invocation in this workflow. | Architecture note |
| The current logic likely contains a status filter mismatch (`notEquals not-processed`) and a filesystem path mismatch (`youtube/` vs `yt`). | Important implementation warning |