Extract live stream highlights using WayinVideo AI Clipping API and Google Drive

https://n8nworkflows.xyz/workflows/extract-live-stream-highlights-using-wayinvideo-ai-clipping-api-and-google-drive-14672


# Extract live stream highlights using WayinVideo AI Clipping API and Google Drive

## 1. Workflow Overview

This workflow turns a submitted live stream recording into multiple short highlight clips using the WayinVideo AI Clipping API, then uploads those clips to Google Drive. It is designed for streamers, content teams, and social media managers who want to repurpose long-form stream recordings into short vertical videos without manually reviewing the full video.

The workflow has a single entry point: an n8n Form Trigger. After the user submits a stream URL and metadata, the workflow sends the request to WayinVideo, waits for processing, retrieves the resulting clips, breaks the returned array into one item per clip, downloads each clip file, and uploads each file into a Google Drive folder using the AI-generated clip title as the filename.

### 1.1 Input Reception
This block collects the stream URL and supporting metadata from a public form.

### 1.2 AI Clipping Job Submission and Result Retrieval
This block sends the stream to WayinVideo, waits for processing time, and requests the clipping results using the returned job ID.

### 1.3 Clip Extraction, Download, and Storage
This block transforms the returned clips array into individual n8n items, downloads each exported clip file, and uploads them to Google Drive.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception

**Overview:**  
This block is the workflow entry point. It exposes a web form where users submit the live stream URL, channel name, category, and desired number of clips.

**Nodes Involved:**  
- 1. Form — Stream URL + Details1

### Node Details

#### 1. Form — Stream URL + Details1
- **Type and technical role:**  
  `n8n-nodes-base.formTrigger`  
  Public form-based trigger that starts the workflow when a user submits the form.

- **Configuration choices:**  
  - Form title: `🎮 Live Stream Highlight Extractor`
  - Form description explains that AI will extract viral and exciting moments
  - Four required fields are configured:
    1. `Live Stream Recording URL`
    2. `Streamer / Channel Name`
    3. `Stream Category`
    4. `Number of Highlight Clips`

- **Key expressions or variables used:**  
  This node produces the field values later used by downstream expressions such as:
  - `$json['Live Stream Recording URL']`
  - `$json['Streamer / Channel Name']`
  - `$json['Number of Highlight Clips']`

- **Input and output connections:**  
  - **Input:** none; this is the entry point
  - **Output:** to `2. WayinVideo — Submit Clipping Task1`

- **Version-specific requirements:**  
  - Uses Form Trigger `typeVersion: 2.2`
  - Requires the workflow to be active for the production form URL to trigger properly

- **Edge cases or potential failure types:**  
  - Workflow not activated, so the production form does not trigger
  - User submits a malformed or inaccessible video URL
  - User enters a non-numeric value for `Number of Highlight Clips`; depending on API handling, this may cause the downstream JSON body to become invalid or rejected
  - No validation is implemented for supported platforms or URL reachability

- **Sub-workflow reference:**  
  None

---

## 2.2 AI Clipping Job Submission and Result Retrieval

**Overview:**  
This block creates a WayinVideo clipping job, waits 90 seconds for processing, then calls the result endpoint using the job ID returned by the submission step. It is the core API orchestration layer of the workflow.

**Nodes Involved:**  
- 2. WayinVideo — Submit Clipping Task1
- 3. Wait — 90 Seconds1
- 4. WayinVideo — Get Clips Result1

### Node Details

#### 2. WayinVideo — Submit Clipping Task1
- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Sends a POST request to the WayinVideo API to create a clipping task.

- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
  - Request body is sent as JSON
  - Headers include:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`

- **Key expressions or variables used:**  
  JSON body is dynamically built from form values:
  - `video_url`: `{{ $json['Live Stream Recording URL'] }}`
  - `project_name`: `Highlights - {{ $json['Streamer / Channel Name'] }}`
  - `limit`: `{{ $json['Number of Highlight Clips'] }}`

  Static processing settings:
  - `target_duration`: `DURATION_30_60`
  - `enable_export`: `true`
  - `resolution`: `HD_720`
  - `enable_caption`: `true`
  - `caption_display`: `original`
  - `cc_style_tpl`: `temp-7`
  - `enable_ai_hook`: `true`
  - `ai_hook_script_style`: `excited`
  - `ai_hook_position`: `beginning`
  - `enable_ai_reframe`: `true`
  - `ratio`: `RATIO_9_16`
  - `target_lang`: `en`

- **Input and output connections:**  
  - **Input:** from `1. Form — Stream URL + Details1`
  - **Output:** to `3. Wait — 90 Seconds1`

- **Version-specific requirements:**  
  - Uses HTTP Request `typeVersion: 4.2`
  - The node is configured with manual header parameters rather than reusable credentials

- **Edge cases or potential failure types:**  
  - Missing or invalid API token in the `Authorization` header
  - Invalid `video_url` or unsupported source platform
  - Private, age-restricted, or inaccessible video URLs
  - Invalid JSON if `Number of Highlight Clips` is not numeric
  - WayinVideo quota, billing, or subscription limitations
  - API schema changes that invalidate enum values like `DURATION_30_60` or `RATIO_9_16`

- **Sub-workflow reference:**  
  None

---

#### 3. Wait — 90 Seconds1
- **Type and technical role:**  
  `n8n-nodes-base.wait`  
  Suspends execution for a fixed duration to allow the clipping job time to process before results are requested.

- **Configuration choices:**  
  - Wait amount: `90` seconds

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  - **Input:** from `2. WayinVideo — Submit Clipping Task1`
  - **Output:** to `4. WayinVideo — Get Clips Result1`

- **Version-specific requirements:**  
  - Uses Wait `typeVersion: 1.1`
  - Wait nodes rely on n8n’s resume mechanism; proper instance persistence is required in production environments

- **Edge cases or potential failure types:**  
  - 90 seconds may be too short for long videos, causing empty or incomplete results
  - If the n8n instance is misconfigured for wait/resume behavior, executions may not resume correctly
  - Long-running jobs are not polled repeatedly; the workflow checks only once after waiting

- **Sub-workflow reference:**  
  None

---

#### 4. WayinVideo — Get Clips Result1
- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Retrieves clipping results from WayinVideo using the job ID returned earlier.

- **Configuration choices:**  
  - Method defaults to `GET`
  - URL is dynamically constructed:
    - `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $('2. WayinVideo — Submit Clipping Task1').item.json.data.id }}`
  - Headers include:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`

- **Key expressions or variables used:**  
  - Job ID reference:
    - `$('2. WayinVideo — Submit Clipping Task1').item.json.data.id`

- **Input and output connections:**  
  - **Input:** from `3. Wait — 90 Seconds1`
  - **Output:** to `5. Code — Extract Clip Data`

- **Version-specific requirements:**  
  - Uses HTTP Request `typeVersion: 4.2`
  - Relies on the submission node’s response containing `data.id`

- **Edge cases or potential failure types:**  
  - Invalid or expired API token
  - Submission response does not contain `data.id`
  - Job still processing after 90 seconds, leading to incomplete or empty `data.clips`
  - API returns an error object instead of the expected result structure
  - Expression failure if the referenced node output is missing or malformed

- **Sub-workflow reference:**  
  None

---

## 2.3 Clip Extraction, Download, and Storage

**Overview:**  
This block converts the API response array into one n8n item per clip, downloads each clip file as binary data, and uploads each file to Google Drive. It handles the fan-out from one API result into multiple files.

**Nodes Involved:**  
- 5. Code — Extract Clip Data
- 6. HTTP — Download Clip
- 7. Google Drive — Upload Clip

### Node Details

#### 5. Code — Extract Clip Data
- **Type and technical role:**  
  `n8n-nodes-base.code`  
  Runs JavaScript to extract each clip from the WayinVideo response and emit one item per clip.

- **Configuration choices:**  
  - JavaScript code reads:
    - `const clips = $json.data.clips;`
  - It returns:
    - `title`
    - `export_link`
    - `score`
    - `tags`
    - `desc`
    - `begin_ms`
    - `end_ms`

- **Key expressions or variables used:**  
  - `$json.data.clips`
  - `clips.map(...)`

- **Input and output connections:**  
  - **Input:** from `4. WayinVideo — Get Clips Result1`
  - **Output:** to `6. HTTP — Download Clip`

- **Version-specific requirements:**  
  - Uses Code node `typeVersion: 2`
  - JavaScript execution must be enabled in the n8n environment as standard for this node

- **Edge cases or potential failure types:**  
  - `data.clips` missing or not an array causes runtime failure
  - Empty clips array results in no downstream executions
  - Clip objects missing `export_link` or `title` may break later download or naming steps
  - No sanitization is applied to file names or metadata

- **Sub-workflow reference:**  
  None

---

#### 6. HTTP — Download Clip
- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Downloads the generated clip file from the WayinVideo `export_link`.

- **Configuration choices:**  
  - URL: `={{ $json.export_link }}`
  - Response format is configured as `file`
  - The downloaded content is stored as binary data, typically under the default binary property name `data`

- **Key expressions or variables used:**  
  - `$json.export_link`

- **Input and output connections:**  
  - **Input:** from `5. Code — Extract Clip Data`
  - **Output:** to `7. Google Drive — Upload Clip`

- **Version-specific requirements:**  
  - Uses HTTP Request `typeVersion: 4.4`
  - Compatible with binary response handling expected by the Google Drive upload node

- **Edge cases or potential failure types:**  
  - Missing or invalid `export_link`
  - Temporary CDN or file-hosting expiration
  - Download timeout for large files
  - Non-file or HTML error response returned instead of media
  - Large memory usage if many clips are processed with large file sizes

- **Sub-workflow reference:**  
  None

---

#### 7. Google Drive — Upload Clip
- **Type and technical role:**  
  `n8n-nodes-base.googleDrive`  
  Uploads each downloaded clip file to Google Drive.

- **Configuration choices:**  
  - File name:
    - `={{ $('5. Code — Extract Clip Data').item.json.title }}`
  - Drive:
    - `My Drive`
  - Folder ID:
    - `YOUR_GOOGLE_DRIVE_FOLDER_ID_HERE`
  - Binary property for upload:
    - `=data`

- **Key expressions or variables used:**  
  - File name from the Code node:
    - `$('5. Code — Extract Clip Data').item.json.title`
  - Binary input property:
    - `data`

- **Input and output connections:**  
  - **Input:** from `6. HTTP — Download Clip`
  - **Output:** none

- **Version-specific requirements:**  
  - Uses Google Drive node `typeVersion: 3`
  - Requires a configured Google Drive OAuth2 credential in n8n
  - The folder ID must exist and be accessible by the connected Google account

- **Edge cases or potential failure types:**  
  - Missing or invalid Google Drive OAuth2 credential
  - Incorrect folder ID
  - Insufficient Drive permissions to write to the target folder
  - Title may contain characters problematic for file naming or cause duplicate names
  - If the HTTP node stores binary in a property other than `data`, upload fails because `inputDataFieldName` is set to `data`

- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main — Overview & Setup1 | Sticky Note | Canvas documentation and setup guidance |  |  | ## Live Stream Highlight Extractor — WayinVideo AI<br>### How it works<br>User submits a live stream recording URL via a form. The workflow sends it to the WayinVideo AI Clipping API, waits 90 seconds for processing, then fetches the generated highlight clips. A Code node extracts each clip's title, export link, score, tags, and timestamps. Each clip is then downloaded and uploaded to Google Drive using the AI-generated clip title as the filename.<br>### Setup<br>1. Replace `YOUR_WAYIN_API_KEY_HERE` in nodes 2 and 4 with your WayinVideo API key<br>2. Replace `YOUR_GOOGLE_DRIVE_FOLDER_ID_HERE` in node 7 with your target Drive folder ID<br>3. Connect your Google Drive OAuth2 credentials in node 7<br>4. Activate the workflow and share the Form URL with streamers<br>### Customization Tips<br>- Change `target_duration` to `DURATION_15_30` for shorter Reels/Shorts clips<br>- Swap `ratio` to `RATIO_16_9` for YouTube horizontal highlights<br>- Increase the Wait node beyond 90s for streams longer than 2 hours<br>- Add a Google Sheets node after node 7 to log clip titles, scores, and Drive links |
| Section — Input1 | Sticky Note | Section label for the input block |  |  | ## 📥 Step 1 — User Input<br>Collects the stream recording URL, channel name, category, and number of clips to extract from the user. |
| 1. Form — Stream URL + Details1 | Form Trigger | Collect form input and start workflow |  | 2. WayinVideo — Submit Clipping Task1 | ## 📥 Step 1 — User Input<br>Collects the stream recording URL, channel name, category, and number of clips to extract from the user. |
| Section — AI Clipping1 | Sticky Note | Section label for the WayinVideo processing block |  |  | ## 🤖 Step 2 — AI Clipping via WayinVideo<br>Submits the video to WayinVideo, waits 90 seconds for processing, then fetches the generated highlight clips with export links. |
| 2. WayinVideo — Submit Clipping Task1 | HTTP Request | Create WayinVideo clipping job | 1. Form — Stream URL + Details1 | 3. Wait — 90 Seconds1 | ## 🤖 Step 2 — AI Clipping via WayinVideo<br>Submits the video to WayinVideo, waits 90 seconds for processing, then fetches the generated highlight clips with export links. |
| 3. Wait — 90 Seconds1 | Wait | Pause before retrieving clip results | 2. WayinVideo — Submit Clipping Task1 | 4. WayinVideo — Get Clips Result1 | ## 🤖 Step 2 — AI Clipping via WayinVideo<br>Submits the video to WayinVideo, waits 90 seconds for processing, then fetches the generated highlight clips with export links.<br>## ⚠️ WARNING — Wait Time May Be Too Short<br>For live streams longer than 90 minutes, the 90-second wait may not be enough for WayinVideo to finish processing. If clips are missing or the result is empty, increase the Wait node to 180–300 seconds. |
| 4. WayinVideo — Get Clips Result1 | HTTP Request | Fetch finished clips using WayinVideo job ID | 3. Wait — 90 Seconds1 | 5. Code — Extract Clip Data | ## 🤖 Step 2 — AI Clipping via WayinVideo<br>Submits the video to WayinVideo, waits 90 seconds for processing, then fetches the generated highlight clips with export links. |
| Warning — Wait Duration1 | Sticky Note | Warning about insufficient wait time for long videos |  |  | ## ⚠️ WARNING — Wait Time May Be Too Short<br>For live streams longer than 90 minutes, the 90-second wait may not be enough for WayinVideo to finish processing. If clips are missing or the result is empty, increase the Wait node to 180–300 seconds. |
| Section — Extract & Upload | Sticky Note | Section label for clip extraction, download, and upload |  |  | ## ☁️ Step 3 — Extract, Download & Save Clips<br>Parses each clip from the API response, downloads the video file, and uploads to Google Drive using the AI-generated clip title as the filename. |
| 5. Code — Extract Clip Data | Code | Split returned clips array into individual clip items | 4. WayinVideo — Get Clips Result1 | 6. HTTP — Download Clip | ## ☁️ Step 3 — Extract, Download & Save Clips<br>Parses each clip from the API response, downloads the video file, and uploads to Google Drive using the AI-generated clip title as the filename. |
| 6. HTTP — Download Clip | HTTP Request | Download each clip file as binary data | 5. Code — Extract Clip Data | 7. Google Drive — Upload Clip | ## ☁️ Step 3 — Extract, Download & Save Clips<br>Parses each clip from the API response, downloads the video file, and uploads to Google Drive using the AI-generated clip title as the filename. |
| 7. Google Drive — Upload Clip | Google Drive | Upload downloaded clip to Google Drive folder | 6. HTTP — Download Clip |  | ## ☁️ Step 3 — Extract, Download & Save Clips<br>Parses each clip from the API response, downloads the video file, and uploads to Google Drive using the AI-generated clip title as the filename. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node** named `1. Form — Stream URL + Details1`.
   - Set the form title to: `🎮 Live Stream Highlight Extractor`
   - Set the description to something like:  
     `Paste your live stream recording URL — AI will automatically extract the most viral and exciting highlight moments as ready-to-upload clips.`
   - Add four required fields:
     1. `Live Stream Recording URL`
     2. `Streamer / Channel Name`
     3. `Stream Category`
     4. `Number of Highlight Clips`
   - Keep placeholders similar to the original workflow for user clarity.
   - This node is the workflow entry point.

3. **Add an HTTP Request node** named `2. WayinVideo — Submit Clipping Task1`.
   - Connect it after the Form Trigger.
   - Configure:
     - Method: `POST`
     - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
     - Send Headers: enabled
     - Specify Body: `JSON`
     - Send Body: enabled
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Set the JSON body to use expressions from the form:
     - `video_url` from `Live Stream Recording URL`
     - `project_name` as `Highlights - [channel name]`
     - `limit` from `Number of Highlight Clips`
   - Include the remaining fixed options:
     - `target_duration`: `DURATION_30_60`
     - `enable_export`: `true`
     - `resolution`: `HD_720`
     - `enable_caption`: `true`
     - `caption_display`: `original`
     - `cc_style_tpl`: `temp-7`
     - `enable_ai_hook`: `true`
     - `ai_hook_script_style`: `excited`
     - `ai_hook_position`: `beginning`
     - `enable_ai_reframe`: `true`
     - `ratio`: `RATIO_9_16`
     - `target_lang`: `en`

4. **Insert your WayinVideo API token** in the submit node.
   - Replace `YOUR_TOKEN_HERE` with your real bearer token.
   - This workflow does not use a dedicated credential object for WayinVideo; the token is hardcoded in headers.

5. **Add a Wait node** named `3. Wait — 90 Seconds1`.
   - Connect it after the submit node.
   - Set the wait amount to `90` seconds.
   - If you expect long recordings, use `180` or `300` seconds instead.

6. **Add a second HTTP Request node** named `4. WayinVideo — Get Clips Result1`.
   - Connect it after the Wait node.
   - Configure:
     - Method: `GET`
     - URL as an expression using the job ID from node 2:  
       `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $('2. WayinVideo — Submit Clipping Task1').item.json.data.id }}`
     - Send Headers: enabled
   - Add the same headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`

7. **Insert your WayinVideo API token again** in the results node.
   - The token must be present in both HTTP nodes.
   - If one node is updated and the other is not, the workflow will partially fail.

8. **Add a Code node** named `5. Code — Extract Clip Data`.
   - Connect it after the results node.
   - Use JavaScript that reads the clips array and returns one item per clip:
     - Read `data.clips`
     - Map each clip into an output item containing:
       - `title`
       - `export_link`
       - `score`
       - `tags`
       - `desc`
       - `begin_ms`
       - `end_ms`

9. **Add an HTTP Request node** named `6. HTTP — Download Clip`.
   - Connect it after the Code node.
   - Set the URL to an expression using the current item’s export link:
     - `{{ $json.export_link }}`
   - Configure the response format as `File`.
   - This ensures the clip is stored as binary data, usually under the binary property `data`.

10. **Add a Google Drive node** named `7. Google Drive — Upload Clip`.
    - Connect it after the clip download node.
    - Configure it to upload a file from binary data.
    - Set the binary property/input field to `data`.
    - Set the file name to an expression using the title from the Code node:
      - `{{ $('5. Code — Extract Clip Data').item.json.title }}`
    - Set the Drive to `My Drive`.
    - Set the target folder ID to your destination folder.

11. **Create and attach Google Drive OAuth2 credentials** to the Google Drive node.
    - In n8n, create a `Google Drive OAuth2` credential.
    - Authenticate with the Google account that owns or can edit the destination folder.
    - Select that credential in node 7.

12. **Replace the folder placeholder** in the Google Drive node.
    - Set `folderId` to your actual Google Drive folder ID.
    - You can obtain it from the Drive folder URL after `/folders/`.

13. **Connect the nodes in this exact order:**
    1. Form Trigger  
    2. Submit Clipping Task  
    3. Wait  
    4. Get Clips Result  
    5. Extract Clip Data  
    6. Download Clip  
    7. Upload Clip

14. **Optionally add sticky notes** to document the workflow visually.
    - Input section note
    - AI clipping section note
    - Extract/upload section note
    - Warning note about increasing wait time for long videos
    - General setup note

15. **Test the workflow manually.**
    - Submit a publicly accessible video URL through the form.
    - Confirm node 2 returns a job ID at `data.id`.
    - Confirm node 4 returns `data.clips`.
    - Confirm node 5 emits one item per clip.
    - Confirm node 6 downloads binary content.
    - Confirm node 7 uploads files to the expected Drive folder.

16. **Activate the workflow** once validation is complete.
    - Copy and share the production form URL from the Form Trigger node.

### Credential and Parameter Expectations
- **WayinVideo**
  - No native credential object is used in this workflow
  - Bearer token must be pasted into both HTTP nodes
- **Google Drive**
  - Requires Google Drive OAuth2 credential
  - Connected account must have write access to the target folder

### Input/Output Expectations
- **Form input:** stream URL, channel name, category, clip count
- **WayinVideo submit output:** expected to include `data.id`
- **WayinVideo results output:** expected to include `data.clips`
- **Code output:** one item per clip
- **Download output:** binary clip file in `data`
- **Drive upload output:** uploaded file metadata from Google Drive

### Important Constraints
- The stream URL should be publicly accessible
- The clip count should be numeric
- The wait duration may need to be increased for long streams
- The clip title may need sanitization in stricter file-naming environments

### Sub-workflow Setup
- No sub-workflows are used in this workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| WayinVideo API homepage | https://wayin.ai/api/ |
| Support email | info@incrementors.com |
| Contact page | https://www.incrementors.com/contact-us/ |
| Setup note: the WayinVideo bearer token must be updated in both HTTP Request nodes | Applies to `2. WayinVideo — Submit Clipping Task1` and `4. WayinVideo — Get Clips Result1` |
| Setup note: replace the Google Drive folder placeholder with a real folder ID | Applies to `7. Google Drive — Upload Clip` |
| Operational note: for long streams, increase the Wait node from 90 seconds to 180–300 seconds if result retrieval returns no clips | Applies to `3. Wait — 90 Seconds1` |