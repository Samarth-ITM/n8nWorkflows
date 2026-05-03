Generate AI motion videos with Kling v2.6 (Kie.ai) and auto-post to TikTok

https://n8nworkflows.xyz/workflows/generate-ai-motion-videos-with-kling-v2-6--kie-ai--and-auto-post-to-tiktok-13888


# Generate AI motion videos with Kling v2.6 (Kie.ai) and auto-post to TikTok

# 1. Workflow Overview

This workflow generates an AI motion video with **Kling v2.6 Motion Control via Kie.ai**, then distributes the generated asset by:

- uploading it to **Postiz**
- creating a **TikTok post** through Postiz
- uploading the same file to **Google Drive**

Its main use case is semi-automated short-form content production: starting from a **static character image** and a **reference motion video**, it creates a new motion clip where the image follows the movement pattern of the reference video, then posts it with a predefined TikTok caption.

## 1.1 Input Reception and Parameter Setup

The workflow starts manually and defines three core inputs:

- `image_url`
- `video_url`
- `tiktok_desc`

These act as the source media and social caption for the rest of the workflow.

## 1.2 AI Video Generation Request

The workflow sends a job creation request to **Kie.ai** using the model `kling-2.6/motion-control`.  
It passes n8n’s **Wait node resume URL** as `callBackUrl`, so Kie.ai can notify the workflow when rendering is complete.

## 1.3 Asynchronous Wait and Result Retrieval

After submission, execution pauses in a **Wait** node configured for **webhook resume**.  
When Kie.ai calls the callback URL, the workflow resumes and fetches the final task details from the Kie.ai job record endpoint.

## 1.4 Result Parsing and File Download

The workflow extracts the generated video URL from the returned `resultJson`, parses it, and downloads the video as binary data.

## 1.5 Publishing and Archival

The downloaded binary file is sent to two destinations:

- **Postiz upload API**, then used to create a TikTok post
- **Google Drive**, as a secondary storage/archive branch

## 1.6 Documentation and Visual Guidance Nodes

The workflow contains several **Sticky Note** nodes that explain setup, provide links, and describe the intended sequence.  
These notes are not executable but are important for understanding how to configure credentials and placeholders.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Manual Start and Input Setup

### Overview
This block manually starts the workflow and prepares the three input values used downstream: source image URL, source motion video URL, and TikTok caption. It is the only entry point in this workflow.

### Nodes Involved
- `When clicking ‘Execute workflow’`
- `Set params`

### Node Details

#### 1) When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry node used for interactive execution from the editor.
- **Configuration choices:**  
  No custom parameters; it simply starts the workflow when executed manually.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Set params`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - No runtime failure in normal use
  - Not suitable for unattended production scheduling unless replaced by another trigger
- **Sub-workflow reference:**  
  None.

#### 2) Set params
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the initial JSON payload consumed by the AI generation request.
- **Configuration choices:**  
  Defines three string fields:
  - `image_url`: direct URL to the character image
  - `video_url`: direct URL to the reference motion video
  - `tiktok_desc`: text caption for TikTok
- **Key expressions or variables used:**  
  Static values in the template; intended to be replaced by the user.
- **Input and output connections:**  
  - Input: `When clicking ‘Execute workflow’`
  - Output: `Run Kling v2.6 Motion Control`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Invalid or inaccessible URLs will later break the Kie.ai generation step
  - Non-direct media URLs may fail if the provider blocks hotlinking or requires cookies/authentication
  - Empty `tiktok_desc` may still run but result in an undesirable social post
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block: AI Motion Generation Submission

### Overview
This block submits the media generation request to Kie.ai using Kling 2.6 Motion Control. It tells Kie.ai where to send the completion callback so n8n can resume execution asynchronously.

### Nodes Involved
- `Run Kling v2.6 Motion Control`

### Node Details

#### 3) Run Kling v2.6 Motion Control
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to Kie.ai to create a rendering task.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.kie.ai/api/v1/jobs/createTask`
  - Authentication: generic credential type using **HTTP Bearer Auth**
  - Request body is JSON
  - Model: `kling-2.6/motion-control`
  - Callback URL: `{{ $execution.resumeUrl }}`
  - Input settings include:
    - prompt: `"Make the character in the image follow the movements of the character in the video."`
    - `input_urls`: image URL array
    - `video_urls`: video URL array
    - `character_orientation`: `"video"`
    - `mode`: `"720p"`
- **Key expressions or variables used:**  
  - `{{ $execution.resumeUrl }}`: generated by n8n for the Wait node resume endpoint
  - `{{ $json.image_url }}`
  - `{{ $json.video_url }}`
- **Input and output connections:**  
  - Input: `Set params`
  - Output: `Wait`
- **Version-specific requirements:**  
  Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Bearer token missing or invalid
  - Kie.ai rejects unsupported model name or payload structure
  - Source media URLs not publicly reachable
  - Callback delivery may fail if the n8n instance is not publicly accessible
  - If Kie.ai never calls back, the execution will remain waiting
  - If response schema changes, downstream task ID extraction may fail
- **Sub-workflow reference:**  
  None.

---

## 2.3 Block: Async Pause and Final Job Lookup

### Overview
This block waits for Kie.ai to notify the workflow that processing is complete, then retrieves the final job metadata using the task ID returned by the creation step.

### Nodes Involved
- `Wait`
- `Result`

### Node Details

#### 4) Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Suspends execution until resumed via webhook callback.
- **Configuration choices:**  
  - Resume mode: `webhook`
  - HTTP method: `POST`
- **Key expressions or variables used:**  
  Indirectly depends on `{{ $execution.resumeUrl }}` passed earlier to Kie.ai.
- **Input and output connections:**  
  - Input: `Run Kling v2.6 Motion Control`
  - Output: `Result`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Callback not received
  - Callback uses wrong HTTP method
  - n8n instance inaccessible from the public internet
  - Workflow execution may wait indefinitely unless timeout/governance is handled outside this workflow
- **Sub-workflow reference:**  
  None.

#### 5) Result
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches final record details for the previously created Kie.ai task.
- **Configuration choices:**  
  - Method: default GET
  - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
  - Authentication: generic credential type using **HTTP Bearer Auth**
  - Query parameter:
    - `taskId = {{ $('Run Kling v2.6 Motion Control').item.json.data.taskId }}`
- **Key expressions or variables used:**  
  - `{{ $('Run Kling v2.6 Motion Control').item.json.data.taskId }}`
- **Input and output connections:**  
  - Input: `Wait`
  - Output: `Parsing`
- **Version-specific requirements:**  
  Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Invalid bearer token
  - `taskId` missing because the createTask response structure changed
  - Record exists but state is failed, incomplete, or malformed
  - API success response may still include a job-level failure state
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block: Result Extraction and Video Download

### Overview
This block pulls the `resultJson` string from the Kie.ai response, parses it into structured data, extracts the first generated video URL, and downloads the file as binary.

### Nodes Involved
- `Parsing`
- `Get ResulUrl`
- `Get File Video`

### Node Details

#### 6) Parsing
- **Type and technical role:** `n8n-nodes-base.set`  
  Simplifies the Kie.ai response by extracting the `resultJson` string into a field called `result`.
- **Configuration choices:**  
  Creates one string field:
  - `result = {{ $json.data.resultJson }}`
- **Key expressions or variables used:**  
  - `{{ $json.data.resultJson }}`
- **Input and output connections:**  
  - Input: `Result`
  - Output: `Get ResulUrl`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - `data.resultJson` absent or null
  - If Kie.ai changes response structure, the expression returns undefined
- **Sub-workflow reference:**  
  None.

#### 7) Get ResulUrl
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the JSON string returned by Kie.ai and extracts the first result URL.
- **Configuration choices:**  
  JavaScript code:
  - gets all input items
  - `JSON.parse(item.json.result)`
  - returns `{ resultUrl: parsed.resultUrls[0] }`
- **Key expressions or variables used:**  
  Uses `item.json.result` inside custom JS.
- **Input and output connections:**  
  - Input: `Parsing`
  - Output: `Get File Video`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Invalid JSON in `result`
  - `resultUrls` missing or empty
  - Typographical issue in node name ("ResulUrl") is harmless but may affect maintainability
- **Sub-workflow reference:**  
  None.

#### 8) Get File Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the generated video file from the temporary result URL.
- **Configuration choices:**  
  - URL: `{{ $json.resultUrl }}`
  - No extra options configured
- **Key expressions or variables used:**  
  - `{{ $json.resultUrl }}`
- **Input and output connections:**  
  - Input: `Get ResulUrl`
  - Outputs:
    - `Upload Video to Postiz`
    - `Upload video`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Temporary URL expired before download
  - Remote host blocks request or has bandwidth/timeouts
  - Depending on node defaults, if response is not handled as file/binary correctly, downstream upload nodes may not receive usable binary data
  - Large video files may hit memory or execution limits
- **Sub-workflow reference:**  
  None.

---

## 2.5 Block: Social Upload and TikTok Publishing

### Overview
This block uploads the generated binary file to Postiz, then creates a TikTok post using the returned uploaded asset metadata and the caption defined earlier.

### Nodes Involved
- `Upload Video to Postiz`
- `TikTok`

### Node Details

#### 9) Upload Video to Postiz
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends the generated video binary to Postiz as multipart form data.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.postiz.com/public/v1/upload`
  - Content type: `multipart-form-data`
  - Authentication: generic credential type using **HTTP Header Auth**
  - Body parameter:
    - `file` from binary property `data`
- **Key expressions or variables used:**  
  Binary field reference:
  - input data field name: `data`
- **Input and output connections:**  
  - Input: `Get File Video`
  - Output: `TikTok`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Postiz API key missing or invalid
  - Binary field name mismatch if `Get File Video` does not output binary under `data`
  - Upload size restrictions
  - Unsupported media format
  - API response schema changes could break the next node’s use of `id` and `path`
- **Sub-workflow reference:**  
  None.

#### 10) TikTok
- **Type and technical role:** `n8n-nodes-postiz.postiz`  
  Creates or schedules a social post in Postiz for TikTok.
- **Configuration choices:**  
  - Posting date built from current timestamp:
    - `{{ $now.format('yyyy-LL-dd') }}T{{ $now.format('HH:ii:ss') }}`
  - `shortLink`: enabled
  - Post content includes:
    - media item ID from Postiz upload response: `{{ $json.id }}`
    - media path from Postiz upload response: `{{ $json.path }}`
    - caption from original parameter node: `{{ $('Set params').item.json.tiktok_desc }}`
  - `integrationId`: currently placeholder `XXX`
- **Key expressions or variables used:**  
  - `{{ $now.format('yyyy-LL-dd') }}`
  - `{{ $now.format('HH:ii:ss') }}`
  - `{{ $json.id }}`
  - `{{ $json.path }}`
  - `{{ $('Set params').item.json.tiktok_desc }}`
- **Input and output connections:**  
  - Input: `Upload Video to Postiz`
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
  - Requires the **Postiz community node/package** to be installed in the n8n instance.
- **Edge cases or potential failure types:**  
  - Invalid or missing Postiz credentials
  - `integrationId` left as `XXX`
  - Scheduled date formatting incompatibility if Postiz expects timezone or different format
  - Returned upload object may not match expected media type semantics
  - The payload internally references `imageItem` despite the workflow handling video; if Postiz tightens validation, this may require adjustment
- **Sub-workflow reference:**  
  None.

---

## 2.6 Block: Google Drive Archival

### Overview
This branch stores the downloaded video in Google Drive. It runs in parallel with the Postiz upload branch after the file is downloaded.

### Nodes Involved
- `Upload video`

### Node Details

#### 11) Upload video
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated file to Google Drive.
- **Configuration choices:**  
  - Drive: `My Drive`
  - Folder: root folder `/`
  - OAuth2 credentials configured
  - No advanced options shown
- **Key expressions or variables used:**  
  None explicitly shown in the exported parameters.
- **Input and output connections:**  
  - Input: `Get File Video`
  - Output: none
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Google OAuth token expired or missing scopes
  - If the node expects a binary field name and defaults do not match incoming data, upload may fail
  - Root-folder uploads may become difficult to organize at scale
  - Duplicate file naming behavior depends on node defaults and incoming metadata
- **Sub-workflow reference:**  
  None.

---

## 2.7 Block: Embedded Documentation and Notes

### Overview
These nodes are non-executable annotations. They describe purpose, setup, credential expectations, and external links.

### Nodes Involved
- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`
- `Sticky Note9`

### Node Details

#### 12) Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note containing the overall workflow description, setup instructions, and example links.
- **Configuration choices:**  
  Large descriptive note with:
  - workflow purpose
  - “Start” video link
  - “Result” video link
  - setup instructions for Kie.ai and Postiz
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None at execution time.
- **Sub-workflow reference:**  
  None.

#### 13) Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Setup note for parameter replacement.
- **Configuration choices:**  
  Instructs users to replace example `image_url`, `video_url`, and `tiktok_desc`.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 14) Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Setup note for AI generation and Kie.ai credential configuration.
- **Configuration choices:**  
  Includes a Kie.ai signup/reference link and notes that `httpBearerAuth` credentials are required.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 15) Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Notes the media extraction/download phase.
- **Configuration choices:**  
  Mentions getting video URL and binary.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 16) Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Setup note for Postiz and social channel linking.
- **Configuration choices:**  
  Explains:
  - creating a Postiz account
  - obtaining API key
  - adding social channels
  - replacing `XXX` with the real `ChannelId`
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 17) Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Promotional note linking to the creator’s YouTube channel.
- **Configuration choices:**  
  Contains subscribe text and a linked image.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start |  | Set params |  |
| Set params | Set | Defines source image URL, reference video URL, and TikTok caption | When clicking ‘Execute workflow’ | Run Kling v2.6 Motion Control | ## STEP 1 - Set params<br>replace the example image_url, video_url, and tiktok_desc with your own values. |
| Run Kling v2.6 Motion Control | HTTP Request | Submits Kling 2.6 motion-control generation job to Kie.ai | Set params | Wait | ## STEP 2 - Generate Video<br>Set your [Kie AI API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) You will need to provide credentials for httpBearerAuth |
| Wait | Wait | Pauses execution until Kie.ai callback resumes the run | Run Kling v2.6 Motion Control | Result | ## STEP 2 - Generate Video<br>Set your [Kie AI API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) You will need to provide credentials for httpBearerAuth |
| Result | HTTP Request | Retrieves final Kie.ai job record using task ID | Wait | Parsing | ## STEP 2 - Generate Video<br>Set your [Kie AI API Key](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) You will need to provide credentials for httpBearerAuth |
| Parsing | Set | Extracts `resultJson` into a simpler field for parsing | Result | Get ResulUrl |  |
| Get ResulUrl | Code | Parses `resultJson` and extracts the generated video URL | Parsing | Get File Video | ## STEP 3 - Get Video<br>Get video url and Binary |
| Get File Video | HTTP Request | Downloads the generated video from the result URL | Get ResulUrl | Upload Video to Postiz; Upload video | ## STEP 3 - Get Video<br>Get video url and Binary |
| Upload Video to Postiz | HTTP Request | Uploads the binary video file to Postiz | Get File Video | TikTok | ## STEP 4 - Publish on TikTok & Google Drive<br>- Create an account on [Postiz](https://postiz.com/?ref=n3witalia) FREE 7 days-trial<br>- Get your API Key and set it in Postiz node and Upload Image node<br>- In Calendar tab on [Postiz](https://postiz.com/?ref=n3witalia) click on "Add channel" and connect your social networks (TikTok)<br>- Once connected, copy the "ChannelId" for each social network and insert the appropriate one into the Postiz nodes, replacing the "XXX" string. |
| TikTok | Postiz | Creates/schedules the TikTok post in Postiz | Upload Video to Postiz |  | ## STEP 4 - Publish on TikTok & Google Drive<br>- Create an account on [Postiz](https://postiz.com/?ref=n3witalia) FREE 7 days-trial<br>- Get your API Key and set it in Postiz node and Upload Image node<br>- In Calendar tab on [Postiz](https://postiz.com/?ref=n3witalia) click on "Add channel" and connect your social networks (TikTok)<br>- Once connected, copy the "ChannelId" for each social network and insert the appropriate one into the Postiz nodes, replacing the "XXX" string. |
| Upload video | Google Drive | Uploads the generated video to Google Drive | Get File Video |  | ## STEP 4 - Publish on TikTok & Google Drive<br>- Create an account on [Postiz](https://postiz.com/?ref=n3witalia) FREE 7 days-trial<br>- Get your API Key and set it in Postiz node and Upload Image node<br>- In Calendar tab on [Postiz](https://postiz.com/?ref=n3witalia) click on "Add channel" and connect your social networks (TikTok)<br>- Once connected, copy the "ChannelId" for each social network and insert the appropriate one into the Postiz nodes, replacing the "XXX" string. |
| Sticky Note | Sticky Note | General workflow description and setup guidance |  |  |  |
| Sticky Note1 | Sticky Note | Input parameter guidance |  |  |  |
| Sticky Note2 | Sticky Note | Kie.ai setup guidance |  |  |  |
| Sticky Note3 | Sticky Note | Result retrieval guidance |  |  |  |
| Sticky Note4 | Sticky Note | Postiz/TikTok/Google Drive publication guidance |  |  |  |
| Sticky Note9 | Sticky Note | Promotional note with YouTube link |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like: **Generate Viral AI Motion Video**.

2. **Add a Manual Trigger node**.
   - Node type: **Manual Trigger**
   - Keep default settings.
   - This will be the workflow’s entry point.

3. **Add a Set node** named **Set params**.
   - Connect it after the Manual Trigger.
   - Add three string fields:
     - `image_url`
     - `video_url`
     - `tiktok_desc`
   - Use direct public URLs for the image and reference motion video.
   - Example intent:
     - `image_url`: a PNG/JPG character image
     - `video_url`: a downloadable MP4 motion reference
     - `tiktok_desc`: your TikTok caption text

4. **Add an HTTP Request node** named **Run Kling v2.6 Motion Control**.
   - Connect it after **Set params**.
   - Configure:
     - Method: `POST`
     - URL: `https://api.kie.ai/api/v1/jobs/createTask`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Bearer Auth**
     - Send Body: enabled
     - Specify Body: `JSON`
   - Use this JSON body logic:
     - `model`: `kling-2.6/motion-control`
     - `callBackUrl`: `{{ $execution.resumeUrl }}`
     - `input.prompt`: `"Make the character in the image follow the movements of the character in the video."`
     - `input.input_urls`: array containing `{{ $json.image_url }}`
     - `input.video_urls`: array containing `{{ $json.video_url }}`
     - `input.character_orientation`: `"video"`
     - `input.mode`: `"720p"`
   - **Credentials setup:**
     - Create an **HTTP Bearer Auth** credential
     - Put your **Kie.ai API key/token** in it
   - Important: your n8n instance must be reachable publicly so Kie.ai can call the callback URL.

5. **Add a Wait node** named **Wait**.
   - Connect it after **Run Kling v2.6 Motion Control**.
   - Configure:
     - Resume: `On Webhook Call`
     - HTTP Method: `POST`
   - This node will generate the resume URL used by `$execution.resumeUrl`.

6. **Add another HTTP Request node** named **Result**.
   - Connect it after **Wait**.
   - Configure:
     - Method: `GET`
     - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
     - Authentication: **Generic Credential Type**
     - Generic Auth Type: **HTTP Bearer Auth**
     - Query Parameters:
       - `taskId` = `{{ $('Run Kling v2.6 Motion Control').item.json.data.taskId }}`
   - Reuse the same **Kie.ai Bearer Auth** credential.

7. **Add a Set node** named **Parsing**.
   - Connect it after **Result**.
   - Create one string field:
     - `result` = `{{ $json.data.resultJson }}`

8. **Add a Code node** named **Get ResulUrl**.
   - Connect it after **Parsing**.
   - Use JavaScript that:
     - parses `item.json.result`
     - returns the first URL from `parsed.resultUrls`
   - Equivalent code:
     ```javascript
     const items = $input.all();

     return items.map(item => {
       const parsed = JSON.parse(item.json.result);

       return {
         json: {
           resultUrl: parsed.resultUrls[0]
         }
       };
     });
     ```
   - This node expects `resultJson` to be a JSON string like:
     - `{"resultUrls":["https://...mp4"]}`

9. **Add an HTTP Request node** named **Get File Video**.
   - Connect it after **Get ResulUrl**.
   - Configure:
     - URL: `{{ $json.resultUrl }}`
   - To make the rest of the workflow reliable, ensure the node downloads the remote file as **binary data**.  
     In many n8n setups this means enabling file download/response-as-file behavior so the binary property becomes available, typically as `data`.
   - This is important because both Postiz upload and Google Drive upload depend on binary input.

10. **Add an HTTP Request node** named **Upload Video to Postiz**.
    - Connect it after **Get File Video**.
    - Configure:
      - Method: `POST`
      - URL: `https://api.postiz.com/public/v1/upload`
      - Authentication: **Generic Credential Type**
      - Generic Auth Type: **HTTP Header Auth**
      - Content Type: `Multipart Form-Data`
      - Send Body: enabled
      - Add body parameter:
        - Name: `file`
        - Parameter Type: `Form Binary Data`
        - Input Data Field Name: `data`
    - **Credentials setup:**
      - Create an **HTTP Header Auth** credential for Postiz
      - Add the header expected by Postiz API, using your Postiz API key
    - Make sure the incoming binary property is actually named `data`; if not, adapt this field.

11. **Install and configure the Postiz node package** if not already present.
    - The workflow uses `n8n-nodes-postiz.postiz`.
    - Your n8n environment must support this node.

12. **Add a Postiz node** named **TikTok**.
    - Connect it after **Upload Video to Postiz**.
    - Configure the posting date:
      - `{{ $now.format('yyyy-LL-dd') }}T{{ $now.format('HH:ii:ss') }}`
    - Enable `shortLink` if desired.
    - Build one post item using:
      - uploaded asset `id` = `{{ $json.id }}`
      - uploaded asset `path` = `{{ $json.path }}`
      - caption/content = `{{ $('Set params').item.json.tiktok_desc }}`
    - Replace the placeholder `integrationId` value `XXX` with your actual **Postiz TikTok Channel/Integration ID**.
    - **Credentials setup:**
      - Add your **Postiz account** credential for the Postiz node.

13. **Add a Google Drive node** named **Upload video**.
    - Connect it directly after **Get File Video** so it runs in parallel with Postiz upload.
    - Configure:
      - Operation: upload file
      - Drive: `My Drive`
      - Folder: your chosen destination, or root
    - **Credentials setup:**
      - Create or reuse a **Google Drive OAuth2** credential
      - Grant file upload permissions
    - Ensure the node is configured to use the incoming binary file property, commonly `data`.

14. **Create the connections in this exact order**:
    1. `When clicking ‘Execute workflow’` → `Set params`
    2. `Set params` → `Run Kling v2.6 Motion Control`
    3. `Run Kling v2.6 Motion Control` → `Wait`
    4. `Wait` → `Result`
    5. `Result` → `Parsing`
    6. `Parsing` → `Get ResulUrl`
    7. `Get ResulUrl` → `Get File Video`
    8. `Get File Video` → `Upload Video to Postiz`
    9. `Upload Video to Postiz` → `TikTok`
    10. `Get File Video` → `Upload video`

15. **Add sticky notes if you want to preserve the original canvas guidance**.
    Suggested notes:
    - Step 1: replace `image_url`, `video_url`, `tiktok_desc`
    - Step 2: add Kie.ai bearer auth
    - Step 3: extract video URL and download binary
    - Step 4: configure Postiz, TikTok integration ID, and Google Drive

16. **Test the workflow manually**.
    - Start with valid direct URLs
    - Run the workflow
    - Confirm:
      - Kie.ai accepts the job
      - Wait node enters waiting state
      - callback reaches n8n
      - result retrieval returns `state: success`
      - `resultJson` contains a valid `resultUrls[0]`
      - video downloads as binary
      - Postiz upload returns media `id` and `path`
      - TikTok post is created
      - Google Drive upload succeeds

17. **Validate production readiness**.
    Before real usage:
    - replace sample URLs
    - replace `XXX` integration ID
    - confirm your n8n public domain and webhook reachability
    - verify Kie.ai quotas/credits
    - verify Postiz account permissions
    - confirm TikTok channel is connected inside Postiz
    - consider adding error handling for failed Kie.ai jobs or missing callback events

### Credential Summary

- **Kie.ai**
  - Used in:
    - `Run Kling v2.6 Motion Control`
    - `Result`
  - Type: **HTTP Bearer Auth**
  - Requirement: valid API token

- **Postiz upload API**
  - Used in:
    - `Upload Video to Postiz`
  - Type: **HTTP Header Auth**
  - Requirement: valid Postiz API key in header form

- **Postiz node account**
  - Used in:
    - `TikTok`
  - Type: **Postiz account credential**
  - Requirement: authorized Postiz account and existing TikTok integration/channel

- **Google Drive**
  - Used in:
    - `Upload video`
  - Type: **Google Drive OAuth2**
  - Requirement: permissions to upload files to chosen drive/folder

### Input/Output Expectations

- **Set params output**
  - JSON fields:
    - `image_url`
    - `video_url`
    - `tiktok_desc`

- **Kie.ai createTask output**
  - Expected field:
    - `data.taskId`

- **Kie.ai recordInfo output**
  - Expected field:
    - `data.resultJson`

- **Code node output**
  - Expected field:
    - `resultUrl`

- **Get File Video output**
  - Expected:
    - binary file, usually under `data`

- **Postiz upload output**
  - Expected fields:
    - `id`
    - `path`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview note includes a start example video | https://iframe.mediadelivery.net/play/580928/707dae0c-0eac-42be-88ea-0210a5c3c906 |
| Workflow overview note includes a final result example video | https://iframe.mediadelivery.net/play/580928/7019158a-e731-4d10-9fea-e5b81c3c634b |
| Kie.ai setup note says to provide `httpBearerAuth` credentials | https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5 |
| Postiz setup note recommends creating an account, getting an API key, and connecting TikTok channels | https://postiz.com/?ref=n3witalia |
| Creator note promotes a YouTube channel with practical n8n videos and free templates | https://youtube.com/@n3witalia |
| Important implementation note: the workflow relies on Kie.ai calling the n8n Wait resume URL, so the n8n instance must be publicly reachable | Infrastructure/network requirement |
| Important implementation note: `integrationId` in the TikTok/Postiz node is still `XXX` and must be replaced before posting will work | Postiz channel configuration |
| Important implementation note: the downloaded file must be available as binary data, likely under the `data` property, for both Postiz and Google Drive upload nodes | Binary handling requirement |
| Important implementation note: the Google Drive branch is archival only and does not feed back into the TikTok posting path | Branching behavior |