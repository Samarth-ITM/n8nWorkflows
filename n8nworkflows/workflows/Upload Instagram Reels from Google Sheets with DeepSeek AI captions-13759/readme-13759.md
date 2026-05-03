Upload Instagram Reels from Google Sheets with DeepSeek AI captions

https://n8nworkflows.xyz/workflows/upload-instagram-reels-from-google-sheets-with-deepseek-ai-captions-13759


# Upload Instagram Reels from Google Sheets with DeepSeek AI captions

# 1. Workflow Overview

This workflow automates the publication of Instagram Reels using content tracked in Google Sheets, media stored in Google Drive, AI-generated captions from DeepSeek, optional caption archiving in Airtable, and final publishing through the Facebook/Instagram Graph API.

Its main use case is batch or scheduled social media publishing for creators, agencies, or internal marketing teams who manage a spreadsheet-based content calendar. The workflow downloads a source video, reformats it with FFmpeg, uploads the processed file to a public server, generates a short caption, creates an Instagram Reel media container, waits for Instagram processing to complete, publishes the Reel, then marks the spreadsheet row as posted and performs server cleanup.

## 1.1 Trigger Phase

The workflow can start from two entry points:

- a **Schedule Trigger** running every 12 hours
- a **Google Sheets Trigger** for new rows added to a sheet

In the provided workflow, the Google Sheets trigger is disabled, so the active intended entry point is the scheduled run.

## 1.2 Content Preparation

After triggering, the workflow fetches rows from Google Sheets, downloads the video from Google Drive using the file URL or ID stored in the row, saves the file locally, processes it with FFmpeg into a 1080x1920 Reel-friendly format, reloads the processed file, and uploads it to a remote web server via SSH.

## 1.3 Caption Generation

The workflow then calls a DeepSeek chat model through an AI Agent node to generate an Instagram caption based on the spreadsheet title field. There is also an Airtable storage node intended to archive the caption, but that node is disabled.

## 1.4 Instagram Publishing

The processed video is published in two phases required by the Graph API:

1. create the Instagram media container for a Reel
2. poll until processing is finished
3. publish the created media container

## 1.5 Post-Publish Cleanup

Once the Reel is published, the workflow updates the original Google Sheets row to mark it as posted, then removes uploaded files from the remote server.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger Phase

### Overview

This block defines how the workflow starts. It supports both periodic execution and event-based execution from a spreadsheet, though only the scheduled trigger is currently active.

### Nodes Involved

- Trigger: Every 12 Hours
- Trigger: New Row in Sheet

### Node Details

#### Trigger: Every 12 Hours

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; time-based entry point.
- **Configuration choices:** Configured to run every 12 hours.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch Unposted Videos**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Usually low-risk. Possible missed runs if the n8n instance is down during scheduled time windows.
- **Sub-workflow reference:** None.

#### Trigger: New Row in Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheetsTrigger`; event-style polling trigger for new rows.
- **Configuration choices:** Polling interval set to every minute; document ID and sheet name are not filled in the JSON.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch Unposted Videos**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Disabled in the current workflow
  - Missing sheet configuration would prevent operation if enabled
  - OAuth permission errors
  - Duplicate handling depends on trigger behavior and spreadsheet state
- **Sub-workflow reference:** None.

---

## 2.2 Content Preparation

### Overview

This block retrieves candidate rows from Google Sheets, downloads the related video from Google Drive, writes it to local storage, transforms it with FFmpeg, reloads the processed file, and uploads it to a remote server over SSH so the Instagram API can access it publicly.

### Nodes Involved

- Fetch Unposted Videos
- Download Video from Drive
- Save Downloaded File
- Process Video with FFmpeg
- Load Processed Video
- Upload to Server via SSH

### Node Details

#### Fetch Unposted Videos

- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads spreadsheet rows.
- **Configuration choices:** Document ID and sheet name are expected but blank in the JSON. No explicit operation is shown, so this is functioning as a read/get rows step.
- **Key expressions or variables used:** Downstream nodes expect fields such as:
  - `LINK` for Google Drive file reference
  - `NAME` for title/caption prompt
- **Input and output connections:** Input from both triggers; output to **Download Video from Drive**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Spreadsheet credentials missing or expired
  - Empty sheet or no matching rows
  - If the sheet contains already-posted rows and no filtering is configured, duplicates may be processed
  - Schema drift if column names change
- **Sub-workflow reference:** None.

#### Download Video from Drive

- **Type and technical role:** `n8n-nodes-base.googleDrive`; downloads the source video as binary data.
- **Configuration choices:** `operation: download`; file ID is derived from `{{$json.LINK}}` using URL mode, meaning the source row likely contains a Drive sharing URL.
- **Key expressions or variables used:** `={{ $json.LINK }}`
- **Input and output connections:** Input from **Fetch Unposted Videos**; output to **Save Downloaded File**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Invalid or non-public Drive link
  - Wrong file type
  - Large file download timeouts
  - Missing Drive permissions
- **Sub-workflow reference:** None.

#### Save Downloaded File

- **Type and technical role:** `n8n-nodes-base.readWriteFile`; writes binary content to the local filesystem.
- **Configuration choices:** Writes the downloaded binary to `/tmp/{{ $execution.id }}_input.mp4`.
- **Key expressions or variables used:** `=/tmp/{{ $execution.id }}_input.mp4`
- **Input and output connections:** Input from **Download Video from Drive**; output to **Process Video with FFmpeg**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Container or host filesystem permissions
  - Binary input missing
  - Low disk space
  - Path assumptions may differ between n8n deployment modes
- **Sub-workflow reference:** None.

#### Process Video with FFmpeg

- **Type and technical role:** `n8n-nodes-base.executeCommand`; executes an OS-level shell command for video transformation.
- **Configuration choices:** Runs FFmpeg with:
  - input: `/data/{{ $execution.id }}_input.mp4`
  - video filter: `scale=1080:1920`
  - codec: `libx264`
  - pixel format: `yuv420p`
  - output: `/data/{{ $execution.id }}_output.mp4`
- **Key expressions or variables used:** `{{ $execution.id }}`
- **Input and output connections:** Input from **Save Downloaded File**; output to **Load Processed Video**.
- **Version-specific requirements:** Type version `1`.
- **Important implementation note:** This node uses `/data/...` paths, while the previous file-write node uses `/tmp/...`. As configured, this is inconsistent and will fail unless `/tmp` and `/data` are mapped or synchronized externally.
- **Edge cases or potential failure types:**
  - FFmpeg not installed in the n8n runtime
  - Filesystem path mismatch (`/tmp` vs `/data`)
  - Unsupported source codec
  - Corrupted video file
  - Execution environment forbids shell commands
- **Sub-workflow reference:** None.

#### Load Processed Video

- **Type and technical role:** `n8n-nodes-base.readWriteFile`; reads the processed video back into n8n as binary.
- **Configuration choices:** Reads from `/tmp/{{ $execution.id }}_output.mp4`.
- **Key expressions or variables used:** `=/tmp/{{ $execution.id }}_output.mp4`
- **Input and output connections:** Input from **Process Video with FFmpeg**; output to **Upload to Server via SSH**.
- **Version-specific requirements:** Type version `1`.
- **Important implementation note:** This path also conflicts with the FFmpeg output path, which writes to `/data/...`. Without correction, this node will likely fail to find the processed file.
- **Edge cases or potential failure types:**
  - File not found due to path mismatch
  - Permissions issue
  - Output file generation failure upstream
- **Sub-workflow reference:** None.

#### Upload to Server via SSH

- **Type and technical role:** `n8n-nodes-base.ssh`; transfers the processed file to a remote server.
- **Configuration choices:**
  - Resource: file
  - Authentication: private key
  - Remote path: `/var/www/html/uploads/`
- **Key expressions or variables used:** None visible in parameters.
- **Input and output connections:** Input from **Load Processed Video**; output to **Generate Caption with DeepSeek**.
- **Version-specific requirements:** Type version `1`.
- **Credential configuration:** Uses SSH private key credential named `SSH Password account`.
- **Edge cases or potential failure types:**
  - Private key authentication failure
  - Incorrect remote path
  - Web server may not expose the uploaded file publicly
  - If the filename is not deterministic or not captured, the later Instagram media creation step may lack a valid `video_url`
- **Sub-workflow reference:** None.

**Critical structural issue in this block:**  
The workflow uploads a processed video to a server, but the subsequent Instagram media container node does not include a `video_url` parameter. For Reel publishing via the Graph API, a publicly accessible media URL is usually required. As provided, the workflow appears incomplete or incorrectly configured at that point.

---

## 2.3 Caption Generation

### Overview

This block generates an Instagram caption from the video title using DeepSeek. The generated text is intended to be saved optionally in Airtable, but the Airtable node is disabled.

### Nodes Involved

- Generate Caption with DeepSeek
- DeepSeek Chat Model
- Store Caption in Airtable

### Node Details

#### Generate Caption with DeepSeek

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI Agent node that orchestrates prompt execution using a connected language model.
- **Configuration choices:**
  - Prompt type: defined directly in the node
  - Prompt asks for:
    - 2–4 engaging sentences
    - 3–5 hashtags
    - a CTA to check link in bio
    - total under 150 characters
  - It uses the spreadsheet `NAME` field as the video title basis
- **Key expressions or variables used:**
  - `{{ $('Get Rows').item.json.NAME}}`
- **Input and output connections:**
  - Main input from **Upload to Server via SSH**
  - AI language model input from **DeepSeek Chat Model**
  - Main output to **Store Caption in Airtable**
- **Version-specific requirements:** Type version `2`.
- **Critical implementation issue:** The expression references a node named **Get Rows**, but no such node exists. The actual spreadsheet node is named **Fetch Unposted Videos**. This will cause expression resolution failure unless manually corrected.
- **Edge cases or potential failure types:**
  - Missing or invalid DeepSeek credentials
  - Prompt output may exceed 150 characters despite the instruction
  - Expression failure due to bad node reference
  - AI Agent output format may vary and affect downstream caption mapping
- **Sub-workflow reference:** None.

#### DeepSeek Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatDeepSeek`; provides the language model backend for the AI Agent.
- **Configuration choices:** No special options configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connects to **Generate Caption with DeepSeek** via the `ai_languageModel` port.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Credential/authentication issues
  - Model quota or rate limits
  - Regional/API availability issues
- **Sub-workflow reference:** None.

#### Store Caption in Airtable

- **Type and technical role:** `n8n-nodes-base.airtable`; creates a record in Airtable.
- **Configuration choices:**
  - Operation: create
  - Base and table IDs are empty
  - No field mapping defined
- **Input and output connections:** Input from **Generate Caption with DeepSeek**; output to **Create Instagram Media Container**.
- **Version-specific requirements:** Type version `2.1`.
- **Current state:** Disabled.
- **Critical structural consequence:** Because this node is disabled and sits inline between caption generation and Instagram container creation, workflow behavior depends on n8n disabled-node execution semantics in this version. In practice, this should be reviewed carefully after import.
- **Edge cases or potential failure types:**
  - Missing Airtable configuration if enabled
  - No mapped caption field
  - Credential errors
- **Sub-workflow reference:** None.

---

## 2.4 Instagram Publishing

### Overview

This block performs the actual Instagram Reel publication lifecycle: create media container, wait, poll processing state, and publish once processing is complete.

### Nodes Involved

- Create Instagram Media Container
- Wait for Video Processing
- Check Video Processing Status
- Is Video Processed?
- Publish Reel to Instagram

### Node Details

#### Create Instagram Media Container

- **Type and technical role:** `n8n-nodes-base.facebookGraphApi`; creates an Instagram media object before final publication.
- **Configuration choices:**
  - Edge: `media`
  - Host URL: `graph-video.facebook.com`
  - Method: `POST`
  - Graph API version: `v22.0`
  - Query parameters:
    - `media_type=REELS`
    - `caption={{ $('AI Agent').item.json.output }}`
- **Key expressions or variables used:**
  - `={{ $('AI Agent').item.json.output }}`
- **Input and output connections:** Input from **Store Caption in Airtable**; output to **Wait for Video Processing**.
- **Version-specific requirements:** Type version `1`.
- **Critical implementation issues:**
  - References a node named **AI Agent**, but the actual node is **Generate Caption with DeepSeek**
  - No `video_url` query parameter is present, which is normally required for Reel upload
  - `node` is set to a blank space rather than the Instagram Business Account ID
- **Edge cases or potential failure types:**
  - Missing Facebook Graph credentials
  - Invalid account ID
  - Missing required `video_url`
  - Caption expression failure
  - Graph API permission errors such as missing `instagram_content_publish`
- **Sub-workflow reference:** None.

#### Wait for Video Processing

- **Type and technical role:** `n8n-nodes-base.wait`; delays the flow before checking media processing status.
- **Configuration choices:** Wait amount set to 90. In the exported JSON, no explicit unit is shown; in n8n this usually corresponds to a wait duration configured in the UI and should be verified after import.
- **Input and output connections:** Input from **Create Instagram Media Container** and from the false branch of **Is Video Processed?**; output to **Check Video Processing Status**.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If the wait is too short, repeated polling loops may occur
  - If the wait is too long, publication latency increases
- **Sub-workflow reference:** None.

#### Check Video Processing Status

- **Type and technical role:** `n8n-nodes-base.httpRequest`; polls the Graph API for media processing state.
- **Configuration choices:**
  - URL: `https://graph.facebook.com/v21.0/{{ $('Create container').item.json.id }}?fields=status_code,status`
  - Authentication: predefined credential type
  - Credential type: `facebookGraphApi`
- **Key expressions or variables used:**
  - `{{ $('Create container').item.json.id }}`
- **Input and output connections:** Input from **Wait for Video Processing**; output to **Is Video Processed?**
- **Version-specific requirements:** Type version `4.3`.
- **Critical implementation issue:** It references a node named **Create container**, but the actual node is **Create Instagram Media Container**. This breaks the polling URL expression.
- **Edge cases or potential failure types:**
  - Authentication failure
  - Wrong media creation ID due to bad expression
  - API version mismatch (`v21.0` here vs `v22.0` elsewhere)
  - Temporary Graph API delays
- **Sub-workflow reference:** None.

#### Is Video Processed?

- **Type and technical role:** `n8n-nodes-base.if`; checks whether Instagram has finished processing the uploaded Reel.
- **Configuration choices:**
  - Condition: `status_code == FINISHED`
  - Strict type validation enabled
- **Key expressions or variables used:** `={{ $json.status_code }}`
- **Input and output connections:**
  - Input from **Check Video Processing Status**
  - True output to **Publish Reel to Instagram**
  - False output to **Wait for Video Processing**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - If the API returns `ERROR`, `IN_PROGRESS`, or another state, the false branch loops
  - No explicit max retry count, so the loop could continue indefinitely
- **Sub-workflow reference:** None.

#### Publish Reel to Instagram

- **Type and technical role:** `n8n-nodes-base.facebookGraphApi`; final publish step for the prepared media container.
- **Configuration choices:**
  - Edge: `media_publish`
  - Method: `POST`
  - Graph API version: `v22.0`
  - Query parameter: `creation_id={{ $json.id }}`
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:** Input from **Is Video Processed?** true branch; output to **Mark as Posted**.
- **Version-specific requirements:** Type version `1`.
- **Potential issue:** `node` is set to a blank space and should contain the Instagram Business Account ID.
- **Edge cases or potential failure types:**
  - Missing permissions
  - Invalid or expired creation ID
  - Wrong account context
- **Sub-workflow reference:** None.

---

## 2.5 Post-Publish Cleanup

### Overview

This block updates the Google Sheets row after successful publication and cleans up uploaded files on the remote server.

### Nodes Involved

- Mark as Posted
- Execute a command

### Node Details

#### Mark as Posted

- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates a spreadsheet row.
- **Configuration choices:**
  - Operation: `update`
  - Document ID and sheet name are blank in the JSON
  - No update key or mapped fields are visible in the export
- **Key expressions or variables used:** None visible.
- **Input and output connections:** Input from **Publish Reel to Instagram**; output to **Execute a command**.
- **Version-specific requirements:** Type version `4.7`.
- **Critical implementation issue:** The update operation is incomplete in the JSON. For Google Sheets update, row identification and updated columns must be configured, otherwise this node cannot reliably mark the correct row as posted.
- **Edge cases or potential failure types:**
  - Incorrect row updated
  - Missing key column mapping
  - Permission or sheet schema errors
- **Sub-workflow reference:** None.

#### Execute a command

- **Type and technical role:** `n8n-nodes-base.ssh`; runs a remote shell command for cleanup.
- **Configuration choices:**
  - Command: `rm -f /var/www/html/uploads/*`
  - Authentication: private key
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Mark as Posted**; no downstream node.
- **Version-specific requirements:** Type version `1`.
- **Credential configuration:** Uses SSH private key credential named `SSH Password account`.
- **Edge cases or potential failure types:**
  - Deletes all files in the uploads directory, not just the current execution’s file
  - Dangerous in multi-run or multi-tenant environments
  - SSH auth and permissions issues
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: New Row in Sheet | Google Sheets Trigger | Optional event trigger when a new spreadsheet row is added |  | Fetch Unposted Videos | ## Runs every 12 hours or when new rows added to sheet |
| Trigger: Every 12 Hours | Schedule Trigger | Main scheduled entry point |  | Fetch Unposted Videos | ## Runs every 12 hours or when new rows added to sheet |
| Fetch Unposted Videos | Google Sheets | Reads rows containing video metadata and links | Trigger: New Row in Sheet; Trigger: Every 12 Hours | Download Video from Drive | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| Download Video from Drive | Google Drive | Downloads source video from Google Drive | Fetch Unposted Videos | Save Downloaded File | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| Save Downloaded File | Read/Write Files from Disk | Saves downloaded binary to local storage | Download Video from Drive | Process Video with FFmpeg | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| Process Video with FFmpeg | Execute Command | Transcodes/scales video for Instagram Reels | Save Downloaded File | Load Processed Video | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| Load Processed Video | Read/Write Files from Disk | Loads processed video file back into n8n | Process Video with FFmpeg | Upload to Server via SSH | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| Upload to Server via SSH | SSH | Uploads processed video to a remote server | Load Processed Video | Generate Caption with DeepSeek | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| DeepSeek Chat Model | DeepSeek Chat Model | Provides LLM backend for caption generation |  | Generate Caption with DeepSeek | ## Generate AI caption with DeepSeek → Store in Airtable |
| Generate Caption with DeepSeek | LangChain Agent | Generates short Instagram caption from video title | Upload to Server via SSH; DeepSeek Chat Model | Store Caption in Airtable | ## Generate AI caption with DeepSeek → Store in Airtable |
| Store Caption in Airtable | Airtable | Optionally stores generated caption in Airtable | Generate Caption with DeepSeek | Create Instagram Media Container | ## Generate AI caption with DeepSeek → Store in Airtable |
| Create Instagram Media Container | Facebook Graph API | Creates Instagram Reel media container | Store Caption in Airtable | Wait for Video Processing | ## Create media container → Wait → Check status → Publish to Instagram |
| Wait for Video Processing | Wait | Delays before polling media processing status | Create Instagram Media Container; Is Video Processed? | Check Video Processing Status | ## Create media container → Wait → Check status → Publish to Instagram |
| Check Video Processing Status | HTTP Request | Polls Graph API for processing state | Wait for Video Processing | Is Video Processed? | ## Create media container → Wait → Check status → Publish to Instagram |
| Is Video Processed? | IF | Routes flow based on processing completion | Check Video Processing Status | Publish Reel to Instagram; Wait for Video Processing | ## Create media container → Wait → Check status → Publish to Instagram |
| Publish Reel to Instagram | Facebook Graph API | Publishes the processed Reel | Is Video Processed? | Mark as Posted | ## Create media container → Wait → Check status → Publish to Instagram |
| Mark as Posted | Google Sheets | Updates spreadsheet to mark content as published | Publish Reel to Instagram | Execute a command | ## Mark as posted and clean videos |
| Execute a command | SSH | Removes uploaded files from remote server | Mark as Posted |  | ## Mark as posted and clean videos |
| Sticky Note | Sticky Note | Documentation note for overall workflow |  |  | # Upload reels with AI Caption from Google Sheets  This n8n workflow template automates the entire process of publishing Instagram Reels from content stored in Google Sheets and Google Drive. It's designed for content creators, social media managers, and businesses who maintain a content calendar in spreadsheets and need automated publishing with AI-generated captions.  ## How It Works  1. **Trigger**: Runs every 12 hours (or when new rows are added) 2. **Fetch Content**: Retrieves unposted videos from spreadsheet 3. **Process Video**: Downloads from Drive and adds custom overlays with FFmpeg 4. **Generate Caption**: Uses AI to create engaging Instagram captions 5. **Publish**: Uploads to Instagram and marks as posted  ## Setup Steps  - Configure Google Sheets connection and document ID - Set up Google Drive access for video files - Add Instagram/Facebook Graph API credentials - Optional: Configure SSH and Airtable for advanced features |
| ## 1. Trigger Phase | Sticky Note | Visual section label |  |  | ## Runs every 12 hours or when new rows added to sheet |
| ## 2. Content Preparation | Sticky Note | Visual section label |  |  | ## Download video → Save file → Process with FFmpeg → Load processed video → Upload via SSH |
| ## 3. Caption Generation | Sticky Note | Visual section label |  |  | ## Generate AI caption with DeepSeek → Store in Airtable |
| ## 4. Publishing | Sticky Note | Visual section label |  |  | ## Create media container → Wait → Check status → Publish to Instagram |
| ## 5. Cleanup | Sticky Note | Visual section label |  |  | ## Mark as posted and clean videos |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence, including corrections required to make the workflow operational.

## 4.1 Prepare prerequisites

1. **Set up Google Sheets data structure**
   - Create a sheet with at least:
     - `NAME` — video title
     - `LINK` — Google Drive file URL or file ID
     - `POSTED` — publication status
     - optionally a unique ID column such as `ROW_ID`
   - Ensure the n8n Google credentials can access the spreadsheet.

2. **Set up Google Drive**
   - Store source video files in Google Drive.
   - Ensure the Drive credential used by n8n has permission to download them.

3. **Prepare the n8n runtime**
   - Install FFmpeg in the environment where n8n runs.
   - Confirm the filesystem path you will use is writable, for example `/tmp/`.
   - If using Docker, mount a writable directory if needed.

4. **Prepare a public hosting location for processed videos**
   - Set up a remote server with SSH access.
   - Ensure uploaded files are publicly accessible via HTTPS, for example:
     - local path: `/var/www/html/uploads/`
     - public base URL: `https://yourdomain.com/uploads/`
   - This is required because Instagram Reel creation typically needs a public `video_url`.

5. **Prepare Meta credentials**
   - Create a Facebook Graph API app with the required permissions.
   - Use an Instagram Business or Creator account linked to a Facebook Page.
   - Obtain or configure:
     - Instagram Business Account ID
     - credentials with `instagram_content_publish` and related permissions

6. **Optional: Airtable**
   - Create a base/table if you want to archive generated captions.
   - This is optional and was disabled in the provided workflow.

7. **Prepare DeepSeek credentials**
   - Add the DeepSeek API credentials in n8n for the chat model node.

---

## 4.2 Create the trigger nodes

1. Add a **Schedule Trigger** node named **Trigger: Every 12 Hours**
   - Set interval to every `12` hours.

2. Add a **Google Sheets Trigger** node named **Trigger: New Row in Sheet**
   - Configure Google Sheets credentials.
   - Select the spreadsheet and sheet.
   - Set poll frequency to every minute.
   - Leave it disabled if you want scheduled-only behavior.

---

## 4.3 Create the spreadsheet reader

3. Add a **Google Sheets** node named **Fetch Unposted Videos**
   - Connect both triggers to this node.
   - Configure:
     - credentials
     - document ID
     - sheet name
   - Set it to read rows.
   - Ideally configure filtering so only rows where `POSTED` is blank or `false` are returned.
   - Confirm downstream fields include `NAME` and `LINK`.

---

## 4.4 Create the media download and local file handling steps

4. Add a **Google Drive** node named **Download Video from Drive**
   - Connect **Fetch Unposted Videos** → **Download Video from Drive**
   - Operation: `Download`
   - File ID:
     - use the `LINK` column
     - if the sheet stores a full URL, use URL mode
     - expression: `{{$json.LINK}}`

5. Add a **Read/Write Files from Disk** node named **Save Downloaded File**
   - Connect **Download Video from Drive** → **Save Downloaded File**
   - Operation: `Write File`
   - File name:
     - `/tmp/{{ $execution.id }}_input.mp4`

6. Add an **Execute Command** node named **Process Video with FFmpeg**
   - Connect **Save Downloaded File** → **Process Video with FFmpeg**
   - Use a corrected command so paths match:
     - `ffmpeg -y -i /tmp/{{ $execution.id }}_input.mp4 -vf "scale=1080:1920" -c:v libx264 -pix_fmt yuv420p /tmp/{{ $execution.id }}_output.mp4`
   - This produces a portrait 1080x1920 output suitable for Reels.

7. Add a **Read/Write Files from Disk** node named **Load Processed Video**
   - Connect **Process Video with FFmpeg** → **Load Processed Video**
   - Operation: `Read File`
   - File selector:
     - `/tmp/{{ $execution.id }}_output.mp4`

---

## 4.5 Upload the processed file to a public server

8. Add an **SSH** node named **Upload to Server via SSH**
   - Connect **Load Processed Video** → **Upload to Server via SSH**
   - Resource: `File`
   - Authentication: `Private Key`
   - Configure SSH credentials
   - Remote path:
     - `/var/www/html/uploads/`
   - Ensure the uploaded filename is preserved or set explicitly if the node allows it.

9. Add a way to build the public video URL
   - The provided JSON is missing this step, but it is necessary.
   - Add a **Set** node after SSH, for example named **Build Public Video URL**
   - Create a field such as `video_url`
   - Example value:
     - `https://yourdomain.com/uploads/{{ $execution.id }}_output.mp4`
   - Also carry forward `NAME`, row identifiers, and any fields needed later.

This additional node is essential for a working Instagram Reel upload flow.

---

## 4.6 Create the DeepSeek caption generation block

10. Add a **DeepSeek Chat Model** node named **DeepSeek Chat Model**
   - Configure DeepSeek credentials.
   - Leave default model options unless you have specific needs.

11. Add an **AI Agent** node named **Generate Caption with DeepSeek**
   - Connect **Build Public Video URL** (or **Upload to Server via SSH** if you skip the Set node) → **Generate Caption with DeepSeek**
   - Connect **DeepSeek Chat Model** to the AI Agent’s language model input
   - Use a direct prompt similar to:
     - Generate an engaging Instagram caption for a video titled `"{{ $('Fetch Unposted Videos').item.json.NAME }}"`
     - Include:
       - 2–4 engaging sentences
       - 3–5 hashtags
       - CTA to check link in bio
       - keep under 150 characters
   - Correct the broken reference:
     - replace `$('Get Rows')` with `$('Fetch Unposted Videos')`

12. Optional: Add an **Airtable** node named **Store Caption in Airtable**
   - Connect **Generate Caption with DeepSeek** → **Store Caption in Airtable**
   - Operation: `Create`
   - Configure Airtable credentials, base, and table
   - Map fields such as:
     - title → `NAME`
     - caption → AI Agent output
     - timestamp → current execution time
   - If you do not want Airtable, remove this node and connect the caption node directly to the Instagram media node.

---

## 4.7 Create the Instagram Reel publication block

13. Add a **Facebook Graph API** node named **Create Instagram Media Container**
   - Input should come from:
     - **Store Caption in Airtable** if Airtable is enabled
     - otherwise directly from **Generate Caption with DeepSeek** or your intermediate Set node
   - Configure:
     - HTTP method: `POST`
     - Host URL: `graph-video.facebook.com`
     - Graph API version: `v22.0`
     - Node: your Instagram Business Account ID
     - Edge: `media`
   - Add query parameters:
     - `media_type` = `REELS`
     - `video_url` = `{{$json.video_url}}`
     - `caption` = expression pointing to the AI output, for example:
       - `{{ $('Generate Caption with DeepSeek').item.json.output }}`
   - Correct the broken reference:
     - replace `$('AI Agent')` with `$('Generate Caption with DeepSeek')`

14. Add a **Wait** node named **Wait for Video Processing**
   - Connect **Create Instagram Media Container** → **Wait for Video Processing**
   - Set wait duration, for example 90 seconds.

15. Add an **HTTP Request** node named **Check Video Processing Status**
   - Connect **Wait for Video Processing** → **Check Video Processing Status**
   - Method: `GET`
   - Authentication: predefined credential type using your Facebook Graph API credential
   - URL:
     - `https://graph.facebook.com/v22.0/{{ $('Create Instagram Media Container').item.json.id }}?fields=status_code,status`
   - Correct the broken reference:
     - replace `$('Create container')` with `$('Create Instagram Media Container')`
   - Also align API version to `v22.0` for consistency.

16. Add an **IF** node named **Is Video Processed?**
   - Connect **Check Video Processing Status** → **Is Video Processed?**
   - Condition:
     - left: `{{$json.status_code}}`
     - operator: equals
     - right: `FINISHED`

17. Connect the false branch back to **Wait for Video Processing**
   - This creates the polling loop.

18. Add a **Facebook Graph API** node named **Publish Reel to Instagram**
   - Connect the true branch of **Is Video Processed?** → **Publish Reel to Instagram**
   - Configure:
     - method: `POST`
     - Graph API version: `v22.0`
     - Node: your Instagram Business Account ID
     - Edge: `media_publish`
     - query parameter:
       - `creation_id` = `{{$json.id}}`

19. Recommended improvement: add a retry limit
   - The original workflow has no loop breaker.
   - Add a counter or maximum polling attempts using a Set/Code/IF combination to avoid infinite looping when processing fails.

---

## 4.8 Mark the row as posted

20. Add a **Google Sheets** node named **Mark as Posted**
   - Connect **Publish Reel to Instagram** → **Mark as Posted**
   - Operation: `Update`
   - Configure:
     - same spreadsheet and sheet
     - a unique matching key such as row number, `ROW_ID`, or another unique column
   - Update fields such as:
     - `POSTED = TRUE`
     - `POSTED_AT = {{ $now }}`
     - optionally `INSTAGRAM_MEDIA_ID`

This part is incomplete in the provided JSON and must be explicitly configured.

---

## 4.9 Add cleanup

21. Add an **SSH** node named **Execute a command**
   - Connect **Mark as Posted** → **Execute a command**
   - Authentication: same SSH private key
   - Command in the provided workflow:
     - `rm -f /var/www/html/uploads/*`

22. Recommended safer cleanup
   - Instead of deleting every uploaded file, delete only the current file:
     - `rm -f /var/www/html/uploads/{{ $execution.id }}_output.mp4`
   - This avoids race conditions and unintended deletion across runs.

23. Optional local cleanup
   - If needed, add an **Execute Command** node locally to remove:
     - `/tmp/{{ $execution.id }}_input.mp4`
     - `/tmp/{{ $execution.id }}_output.mp4`

---

## 4.10 Add visual documentation notes

24. Add sticky notes corresponding to the blocks:
   - Trigger phase
   - Content preparation
   - Caption generation
   - Publishing
   - Cleanup
   - Optional overall note describing the architecture and prerequisites

---

## 4.11 Credentials required

25. Configure the following credentials in n8n:
   - **Google Sheets OAuth2**
   - **Google Drive OAuth2**
   - **DeepSeek API**
   - **Facebook Graph API**
   - **SSH Private Key**
   - **Airtable API** if using Airtable

---

## 4.12 Corrections required for this specific exported workflow

To reproduce the exported design accurately while making it functional, correct these items:

1. `Generate Caption with DeepSeek`  
   - change `$('Get Rows')` to `$('Fetch Unposted Videos')`

2. `Create Instagram Media Container`  
   - change `$('AI Agent')` to `$('Generate Caption with DeepSeek')`
   - add `video_url`
   - replace blank `node` value with Instagram Business Account ID

3. `Check Video Processing Status`  
   - change `$('Create container')` to `$('Create Instagram Media Container')`
   - ideally use the same Graph API version as the other Facebook nodes

4. File paths  
   - make all file nodes and FFmpeg command use the same directory, preferably `/tmp/` or `/data/`, but not both inconsistently

5. `Mark as Posted`  
   - define exact row matching and updated columns

6. Cleanup command  
   - scope deletion to the current file rather than all files in the upload directory

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow template automates Instagram Reel publishing from Google Sheets and Google Drive with AI-generated captions. | Overall workflow concept |
| Intended audience includes content creators, social media managers, and businesses managing a spreadsheet-based content calendar. | Overall workflow concept |
| Original setup notes mention optional SSH and Airtable configuration for advanced features. | Overall workflow concept |
| The overall note says FFmpeg “adds custom overlays,” but the actual FFmpeg command only scales/transcodes the video to 1080x1920 and does not add overlays. | Accuracy note about current implementation |
| The workflow uses Facebook Graph API versions `v22.0` and `v21.0` in different nodes; aligning versions is recommended. | Consistency note |
| Instagram Reel media creation usually requires a publicly reachable `video_url`. The exported workflow uploads to a server but does not pass that URL into the media creation step. | Critical implementation note |
| The workflow contains several broken node references in expressions due to renamed nodes. These must be fixed before execution. | Critical implementation note |

If you want, I can also produce a second version of this document as a **cleaned operational specification**, reflecting the fixes needed to make the workflow executable end-to-end.