Generate YouTube shorts using WayinVideo AI and upload clips to Google Drive

https://n8nworkflows.xyz/workflows/generate-youtube-shorts-using-wayinvideo-ai-and-upload-clips-to-google-drive-14407


# Generate YouTube shorts using WayinVideo AI and upload clips to Google Drive

# 1. Workflow Overview

This workflow takes a video URL supplied through chat input, submits it to the WayinVideo AI Clips API, waits for processing, repeatedly polls until clips are available, downloads each generated short clip, and uploads the resulting files to a Google Drive folder.

Its main use case is automated repurposing of long-form video content into vertical short-form clips suitable for social media or archival workflows. It is especially useful for podcasts, interviews, webinars, or YouTube videos where AI-selected highlights should be exported and stored automatically.

A notable implementation detail: the JSON provided does **not include an explicit trigger node**, but the first API node expects `{{$json.chatInput}}`. This means the workflow was designed to be started by an upstream chat-based trigger or another node/workflow that passes a `chatInput` field containing the source video URL.

## 1.1 Input Reception

The workflow expects an incoming item containing a `chatInput` property with a public video URL. No validation is performed before submission.

## 1.2 Video Submission to WayinVideo

The video URL is sent to the WayinVideo Clips API with generation settings such as clip duration, number of clips, captions, reframing, resolution, and aspect ratio.

## 1.3 Delayed Polling Loop

After submission, the workflow waits 30 seconds, checks the processing result endpoint, and uses an IF condition to determine whether clips are available. If not, it loops back into another 30-second wait and polls again.

## 1.4 Clip Extraction and Normalization

Once clips are ready, a Code node transforms the API response into one item per clip, extracting the clip title, export link, score, tags, description, and timing metadata.

## 1.5 File Download and Cloud Storage

Each generated clip file is downloaded from its export URL and uploaded into a specified Google Drive folder using the clip title as the file name.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This logical block is implicit rather than fully implemented in the JSON. The workflow assumes another node, trigger, or parent workflow provides a `chatInput` field containing the source video URL.

### Nodes Involved
- No explicit trigger node is present in the JSON
- Indirect dependency: `🎬 Submit Video to WayinVideo API`

### Node Details

#### Expected upstream input
- **Type and technical role:** External workflow input / missing trigger dependency
- **Configuration choices:** The workflow expects one incoming JSON item with a field named `chatInput`
- **Key expressions or variables used:** `{{$json.chatInput}}`
- **Input and output connections:** Not represented in the JSON
- **Version-specific requirements:** None specific, but any trigger or parent workflow must pass the field in a way compatible with n8n expressions
- **Edge cases or potential failure types:**
  - `chatInput` missing or empty
  - invalid or private video URL
  - unsupported video source by WayinVideo
- **Sub-workflow reference:** If this workflow is invoked by another workflow, the caller must provide `chatInput`

---

## 2.2 Video Submission to WayinVideo

### Overview
This block starts the AI clipping job by sending the submitted video URL and generation parameters to WayinVideo. It defines how many clips should be created and how they should be formatted.

### Nodes Involved
- `🎬 Submit Video to WayinVideo API`

### Node Details

#### 🎬 Submit Video to WayinVideo API
- **Type and technical role:** `HTTP Request` node; starts a clip-generation job via WayinVideo REST API
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
  - Sends JSON body
  - Sends explicit headers for API versioning and Bearer auth
  - Uses the incoming `chatInput` as `video_url`
  - Sets project metadata and clip generation options
- **Key expressions or variables used:**
  - `video_url = {{$json.chatInput}}`
  - `project_name = Podcast Clips - {{ new Date().toISOString().split('T')[0] }}`
- **Body parameters interpreted:**
  - `video_url`: source video URL
  - `project_name`: dynamic name with current date
  - `target_duration`: `DURATION_30_60`
  - `limit`: `3`
  - `enable_export`: `true`
  - `resolution`: `HD_720`
  - `enable_caption`: `true`
  - `caption_display`: `original`
  - `cc_style_tpl`: `temp-7`
  - `enable_ai_reframe`: `true`
  - `ratio`: `RATIO_9_16`
- **Headers:**
  - `x-wayinvideo-api-version: v2`
  - `Authorization: Bearer YOUR_TOKEN_HERE`
  - `Content-Type: application/json`
- **Input and output connections:**
  - Input: expected external item with `chatInput`
  - Output: `⏳ Wait 30 Seconds`
- **Version-specific requirements:**
  - Node type version `4.2`
  - Expression syntax assumes modern n8n expression engine
- **Edge cases or potential failure types:**
  - missing or invalid Bearer token
  - invalid video URL
  - API quota/rate limits
  - unsupported source provider
  - malformed body if expressions resolve incorrectly
  - API schema changes in WayinVideo v2
- **Sub-workflow reference:** None

---

## 2.3 Delayed Polling Loop

### Overview
This block prevents immediate result checks and implements a retry loop until clip data becomes available. It uses the job ID returned by the submission step to poll the results endpoint.

### Nodes Involved
- `⏳ Wait 30 Seconds`
- `🔄 Poll for Clip Results`
- `✅ Clips Ready?`

### Node Details

#### ⏳ Wait 30 Seconds
- **Type and technical role:** `Wait` node; delays execution between submission and poll attempts
- **Configuration choices:**
  - Wait duration: 30 seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from:
    - `🎬 Submit Video to WayinVideo API`
    - `✅ Clips Ready?` false branch
  - Output to:
    - `🔄 Poll for Clip Results`
- **Version-specific requirements:**
  - Node type version `1`
  - Wait node uses a generated webhook ID internally for resume handling
- **Edge cases or potential failure types:**
  - workflow resume issues if execution storage is misconfigured
  - long-running workflow retention constraints
  - instance restart before resume, depending on n8n deployment mode
- **Sub-workflow reference:** None

#### 🔄 Poll for Clip Results
- **Type and technical role:** `HTTP Request` node; fetches processing results for the previously submitted job
- **Configuration choices:**
  - Method defaults to `GET`
  - URL is dynamically built from the original submission response job ID
  - Sends the same API version and auth headers
- **Key expressions or variables used:**
  - `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $('🎬 Submit Video to WayinVideo API').item.json.data.id }}`
- **Headers:**
  - `Authorization: Bearer YOUR_TOKEN_HERE`
  - `Content-Type: application/json`
  - `x-wayinvideo-api-version: v2`
- **Input and output connections:**
  - Input: `⏳ Wait 30 Seconds`
  - Output: `✅ Clips Ready?`
- **Version-specific requirements:**
  - Node type version `4.2`
  - Cross-node expression using `$('Node Name').item.json...` requires compatible n8n expression support
- **Edge cases or potential failure types:**
  - original submission response missing `data.id`
  - auth failure
  - API returns processing state without `data.clips`
  - intermittent network errors
  - job expired or deleted remotely
- **Sub-workflow reference:** None

#### ✅ Clips Ready?
- **Type and technical role:** `If` node; branches based on whether the API response contains a non-empty clips array
- **Configuration choices:**
  - Condition checks that `{{$json.data.clips}}` is an array and is not empty
  - True branch continues processing
  - False branch loops back to wait again
- **Key expressions or variables used:**
  - `{{$json.data.clips}}`
- **Input and output connections:**
  - Input: `🔄 Poll for Clip Results`
  - True output: `⚙️ Extract Clip Details`
  - False output: `⏳ Wait 30 Seconds`
- **Version-specific requirements:**
  - Node type version `2.3`
  - Uses condition engine version 3
- **Edge cases or potential failure types:**
  - if API returns a status object without `data.clips`, branch resolves false and loops indefinitely
  - if API returns malformed structure, condition may not behave as expected
  - no explicit maximum retry count, so failed jobs may cause endless polling
- **Sub-workflow reference:** None

---

## 2.4 Clip Extraction and Normalization

### Overview
This block converts the WayinVideo results payload into a clean list of clip items. Each output item represents one generated clip and contains the metadata needed for download and naming.

### Nodes Involved
- `⚙️ Extract Clip Details`

### Node Details

#### ⚙️ Extract Clip Details
- **Type and technical role:** `Code` node; maps the clips array into individual n8n items
- **Configuration choices:**
  - JavaScript reads `const clips = $json.data.clips;`
  - Returns one item per clip with selected fields only
- **Key expressions or variables used:**
  - Reads: `$json.data.clips`
  - Outputs:
    - `title`
    - `export_link`
    - `score`
    - `tags`
    - `desc`
    - `begin_ms`
    - `end_ms`
- **Input and output connections:**
  - Input: `✅ Clips Ready?` true branch
  - Output: `⬇️ Download Clip File`
- **Version-specific requirements:**
  - Node type version `2`
  - Requires JavaScript code execution support enabled in n8n
- **Edge cases or potential failure types:**
  - `data.clips` undefined or not an array
  - individual clip missing `export_link` or `title`
  - null values causing downstream naming or download issues
- **Sub-workflow reference:** None

---

## 2.5 File Download and Google Drive Upload

### Overview
This block downloads the generated clip binary from each export URL and uploads it to a target Google Drive folder. It is the final storage stage of the workflow.

### Nodes Involved
- `⬇️ Download Clip File`
- `☁️ Upload Clip to Google Drive`

### Node Details

#### ⬇️ Download Clip File
- **Type and technical role:** `HTTP Request` node; downloads the clip file as binary data
- **Configuration choices:**
  - URL comes from the extracted `export_link`
  - Response format is configured as `file`
  - Binary output defaults to field name `data`
- **Key expressions or variables used:**
  - `{{$json.export_link}}`
- **Input and output connections:**
  - Input: `⚙️ Extract Clip Details`
  - Output: `☁️ Upload Clip to Google Drive`
- **Version-specific requirements:**
  - Node type version `4.4`
  - Binary response handling must be supported by the installed n8n version
- **Edge cases or potential failure types:**
  - expired or inaccessible export URL
  - slow file download / timeout
  - very large file sizes affecting memory or execution limits
  - missing binary output if the response is not a file
- **Sub-workflow reference:** None

#### ☁️ Upload Clip to Google Drive
- **Type and technical role:** `Google Drive` node; uploads the downloaded binary file to Drive
- **Configuration choices:**
  - Uses Google Drive OAuth2 credentials
  - Upload target is `My Drive`
  - Folder is set via a placeholder ID
  - File name comes from the clip title
  - Binary input field is `data`
- **Key expressions or variables used:**
  - File name: `{{ $('⚙️ Extract Clip Details').item.json.title }}`
  - Folder ID placeholder: `YOUR_GOOGLE_DRIVE_FOLDER_ID`
  - Binary property: `data`
- **Input and output connections:**
  - Input: `⬇️ Download Clip File`
  - Output: none; terminal node
- **Version-specific requirements:**
  - Node type version `3`
  - Requires valid Google Drive OAuth2 credential in n8n
- **Edge cases or potential failure types:**
  - invalid or inaccessible folder ID
  - insufficient Google Drive permissions
  - OAuth token expiry or revoked consent
  - duplicate file names depending on Drive behavior
  - empty title causing poor file naming
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Setup Guide | Sticky Note | General workflow guidance and setup instructions |  |  | ## 🎬 Video → AI Clips → Google Drive<br>**What it does:** User pastes a video URL in chat (YouTube, etc). WayinVideo AI automatically finds the best short clips, adds captions, reframes to 9:16, and uploads them to Google Drive.<br>**How it works:** 1. User sends a video URL via chat 2. Video is submitted to WayinVideo API for clip generation 3. Waits 30s, then polls until clips are ready 4. If not ready → waits 30s again (loop) 5. Extracts clip title, download link, score & tags 6. Downloads each clip file 7. Uploads each clip to your Google Drive folder<br>**Credentials to Add:** Google Drive OAuth2<br>**Placeholders to Fill:** `YOUR_WAYINVIDEO_API_KEY`, `YOUR_GOOGLE_DRIVE_FOLDER_ID`<br>WayinVideo API: `wayinvideo.com/dashboard/api` |
| 🎬 Submit Video to WayinVideo API | HTTP Request | Starts the AI clip generation job | External input not included in JSON | ⏳ Wait 30 Seconds | ## 🎬 Submit Video for Processing<br>• Sends video URL to WayinVideo API<br>• Starts AI clip generation process<br>• Applies settings (duration, captions, format)<br>👉 This is where clip creation begins |
| ⏳ Wait 30 Seconds | Wait | Delays before each poll attempt | 🎬 Submit Video to WayinVideo API; ✅ Clips Ready? (false) | 🔄 Poll for Clip Results | ## ⏳ Wait & Check Processing Status<br>• Waits 30 seconds before checking<br>• Requests API to check if clips are ready<br>• Uses job ID to fetch results<br>👉 Prevents instant API calls & allows processing time |
| 🔄 Poll for Clip Results | HTTP Request | Checks WayinVideo processing results | ⏳ Wait 30 Seconds | ✅ Clips Ready? | ## ⏳ Wait & Check Processing Status<br>• Waits 30 seconds before checking<br>• Requests API to check if clips are ready<br>• Uses job ID to fetch results<br>👉 Prevents instant API calls & allows processing time |
| ✅ Clips Ready? | If | Tests whether clips are available yet | 🔄 Poll for Clip Results | ⚙️ Extract Clip Details; ⏳ Wait 30 Seconds | ## 🔁 Check if Clips are Ready<br>• Checks if clips data is available<br>• If NOT ready → waits again (loop)<br>• If ready → moves to next step<br>👉 Smart loop until clips are generated |
| ⚙️ Extract Clip Details | Code | Converts clips array into one item per clip | ✅ Clips Ready? | ⬇️ Download Clip File | ## ⚙️ Extract Clip Information<br>• Loops through all generated clips<br>• Extracts title, link, score, tags<br>• Prepares clean structured output<br>👉 Converts API response into usable data |
| ⬇️ Download Clip File | HTTP Request | Downloads each generated clip as binary | ⚙️ Extract Clip Details | ☁️ Upload Clip to Google Drive | ## 📥 Download & Save Clips<br>• Downloads clip using export link<br>• Uploads file to Google Drive<br>• Saves with clip title<br>👉 Final storage of generated clips |
| ☁️ Upload Clip to Google Drive | Google Drive | Uploads clip files to a Google Drive folder | ⬇️ Download Clip File |  | ## 📥 Download & Save Clips<br>• Downloads clip using export link<br>• Uploads file to Google Drive<br>• Saves with clip title<br>👉 Final storage of generated clips |
| Sticky Note | Sticky Note | Context note for submission block |  |  | ## 🎬 Submit Video for Processing<br>• Sends video URL to WayinVideo API<br>• Starts AI clip generation process<br>• Applies settings (duration, captions, format)<br>👉 This is where clip creation begins |
| Sticky Note1 | Sticky Note | Context note for wait/poll block |  |  | ## ⏳ Wait & Check Processing Status<br>• Waits 30 seconds before checking<br>• Requests API to check if clips are ready<br>• Uses job ID to fetch results<br>👉 Prevents instant API calls & allows processing time |
| Sticky Note2 | Sticky Note | Context note for readiness check block |  |  | ## 🔁 Check if Clips are Ready<br>• Checks if clips data is available<br>• If NOT ready → waits again (loop)<br>• If ready → moves to next step<br>👉 Smart loop until clips are generated |
| Sticky Note3 | Sticky Note | Context note for clip extraction block |  |  | ## ⚙️ Extract Clip Information<br>• Loops through all generated clips<br>• Extracts title, link, score, tags<br>• Prepares clean structured output<br>👉 Converts API response into usable data |
| Sticky Note4 | Sticky Note | Context note for download/upload block |  |  | ## 📥 Download & Save Clips<br>• Downloads clip using export link<br>• Uploads file to Google Drive<br>• Saves with clip title<br>👉 Final storage of generated clips |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add an entry node or upstream source that provides `chatInput`.**
   - The provided JSON does not include this node.
   - To reproduce the intended behavior, add one of these:
     - a Chat Trigger node,
     - a Webhook node,
     - an Execute Workflow caller,
     - or a Manual Trigger plus Set node for testing.
   - Ensure the incoming JSON contains:
     - `chatInput`: a valid public video URL

3. **Add an HTTP Request node named `🎬 Submit Video to WayinVideo API`.**
   - Method: `POST`
   - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
   - Enable sending headers
   - Enable sending body
   - Set body content as JSON-style parameters
   - Add body parameters:
     - `video_url` → `{{$json.chatInput}}`
     - `project_name` → `Podcast Clips - {{ new Date().toISOString().split('T')[0] }}`
     - `target_duration` → `DURATION_30_60`
     - `limit` → `3`
     - `enable_export` → `true`
     - `resolution` → `HD_720`
     - `enable_caption` → `true`
     - `caption_display` → `original`
     - `cc_style_tpl` → `temp-7`
     - `enable_ai_reframe` → `true`
     - `ratio` → `RATIO_9_16`
   - Add headers:
     - `x-wayinvideo-api-version` → `v2`
     - `Authorization` → `Bearer YOUR_TOKEN_HERE`
     - `Content-Type` → `application/json`
   - Replace `YOUR_TOKEN_HERE` with your real WayinVideo API Bearer token.

4. **Connect the trigger/input node to `🎬 Submit Video to WayinVideo API`.**

5. **Add a Wait node named `⏳ Wait 30 Seconds`.**
   - Unit: `Seconds`
   - Amount: `30`

6. **Connect `🎬 Submit Video to WayinVideo API` to `⏳ Wait 30 Seconds`.**

7. **Add an HTTP Request node named `🔄 Poll for Clip Results`.**
   - Method: `GET`
   - URL:
     - `=https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $('🎬 Submit Video to WayinVideo API').item.json.data.id }}`
   - Add headers:
     - `Authorization` → `Bearer YOUR_TOKEN_HERE`
     - `Content-Type` → `application/json`
     - `x-wayinvideo-api-version` → `v2`
   - Use the same WayinVideo token as above.

8. **Connect `⏳ Wait 30 Seconds` to `🔄 Poll for Clip Results`.**

9. **Add an If node named `✅ Clips Ready?`.**
   - Configure a condition that checks whether `data.clips` is not empty.
   - Use:
     - Left value: `{{$json.data.clips}}`
     - Operator: `is not empty` for array data
   - This creates:
     - **true** branch when clips are available
     - **false** branch when clips are still not ready

10. **Connect `🔄 Poll for Clip Results` to `✅ Clips Ready?`.**

11. **Create the loop for retry behavior.**
   - Connect the **false** output of `✅ Clips Ready?` back to `⏳ Wait 30 Seconds`.

12. **Add a Code node named `⚙️ Extract Clip Details`.**
   - Paste this JavaScript logic conceptually:
     - read the array from `data.clips`
     - output one item per clip
     - keep only the relevant metadata fields
   - Equivalent code:
     ```javascript
     const clips = $json.data.clips;

     return clips.map(clip => ({
       json: {
         title: clip.title,
         export_link: clip.export_link,
         score: clip.score,
         tags: clip.tags,
         desc: clip.desc,
         begin_ms: clip.begin_ms,
         end_ms: clip.end_ms
       }
     }));
     ```

13. **Connect the true output of `✅ Clips Ready?` to `⚙️ Extract Clip Details`.**

14. **Add an HTTP Request node named `⬇️ Download Clip File`.**
   - Method: `GET`
   - URL: `{{$json.export_link}}`
   - Response format: `File`
   - This will store binary data, typically in the binary field `data`

15. **Connect `⚙️ Extract Clip Details` to `⬇️ Download Clip File`.**

16. **Add a Google Drive node named `☁️ Upload Clip to Google Drive`.**
   - Operation: upload a file
   - Credential: connect a Google Drive OAuth2 credential
   - Drive: `My Drive`
   - Folder: choose or paste your target folder ID
   - Binary property / input field name: `data`
   - File name:
     - `{{ $('⚙️ Extract Clip Details').item.json.title }}`
   - Replace the placeholder folder with your actual destination folder.

17. **Connect `⬇️ Download Clip File` to `☁️ Upload Clip to Google Drive`.**

18. **Configure Google Drive credentials.**
   - In n8n, create or select a Google Drive OAuth2 credential.
   - Grant access to the Google account that owns or can write to the target folder.
   - If using a shared drive instead of My Drive, adjust drive/folder settings accordingly.

19. **Test with a known public video URL.**
   - Example test input structure:
     - `chatInput: https://www.youtube.com/watch?v=...`
   - Confirm:
     - submission returns a `data.id`
     - polling eventually returns a non-empty `data.clips`
     - each clip has an `export_link`
     - files appear in Google Drive

20. **Optionally harden the workflow before production use.**
   - Add a validation step before submission to ensure `chatInput` is a URL
   - Add a retry counter to prevent infinite polling
   - Add error branches for:
     - failed job submission
     - empty clips after many retries
     - broken export links
     - upload failures
   - Optionally sanitize file names before uploading to Drive

## Sub-workflow setup
No Sub-workflow node is present in this JSON.

However, if you plan to use this workflow as a called workflow:
- **Required input:** one item containing `chatInput`
- **Expected behavior:** creates clips and uploads them to Google Drive
- **Potential output:** final Google Drive upload result items from the last node

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| WayinVideo API token must replace the Authorization header placeholder in both API request nodes. | `wayinvideo.com/dashboard/api` |
| Google Drive folder placeholder must be replaced with a real folder ID. | Example format: `https://drive.google.com/drive/folders/FOLDER_ID` |
| The workflow is intended for chat-based URL submission, but no trigger is included in the JSON. You must add one manually. | Architecture note |
| The polling loop has no maximum retry limit, so a failed or stuck processing job may keep the workflow running indefinitely. | Reliability note |
| Default clip generation settings are vertical shorts with captions and AI reframing. | `target_duration=DURATION_30_60`, `ratio=RATIO_9_16`, `resolution=HD_720` |
| The setup note describes the end-to-end purpose: submit a video URL, generate AI clips, poll until ready, download each clip, then upload them to Google Drive. | Workflow-level context |