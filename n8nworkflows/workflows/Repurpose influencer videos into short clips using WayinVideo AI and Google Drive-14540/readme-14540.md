Repurpose influencer videos into short clips using WayinVideo AI and Google Drive

https://n8nworkflows.xyz/workflows/repurpose-influencer-videos-into-short-clips-using-wayinvideo-ai-and-google-drive-14540


# Repurpose influencer videos into short clips using WayinVideo AI and Google Drive

# 1. Workflow Overview

This workflow automates the conversion of a long-form influencer video into multiple short-form clips using the WayinVideo AI API, then stores each generated clip in Google Drive.

Typical use cases include:
- Repurposing YouTube interviews, podcasts, or creator videos into Shorts, Reels, or TikTok-style clips
- Building a lightweight internal content pipeline for marketing teams
- Generating AI-selected highlight clips with minimal manual editing

The workflow is organized into four logical blocks:

## 1.1 Input Reception
A web form collects:
- the source video URL,
- the influencer or brand name,
- and the maximum number of clips to generate.

## 1.2 WayinVideo Task Submission
The workflow sends the submitted information to WayinVideo’s clipping API to create an asynchronous clip-generation task. The API returns a task ID.

## 1.3 Polling Loop
Because clip generation is asynchronous, the workflow waits 45 seconds, then polls WayinVideo for results. If clips are not ready, it loops back and waits again.

## 1.4 Clip Extraction, Download, and Google Drive Upload
Once the clips are available, the workflow extracts each clip’s metadata, downloads the exported video file, and uploads each file to a specified Google Drive folder.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block is the workflow entry point. It exposes an n8n form so a user can manually submit the source video and generation parameters that will drive the rest of the process.

### Nodes Involved
- `1. Form — Video URL + Brand + Clip Count`

### Node Details

#### 1. Form — Video URL + Brand + Clip Count
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that starts the workflow from a hosted form submission.
- **Configuration choices:**
  - Form title: `🎬 Influencer Content Repurposing Pipeline`
  - Description explains that the user should paste a long influencer video URL and that AI will extract top clips
  - Three required fields:
    1. `Influencer Video URL`
    2. `Influencer / Brand Name`
    3. `Max Clips to Generate`
- **Key expressions or variables used:**
  - Downstream nodes access submitted values as:
    - `{{$json['Influencer Video URL']}}`
    - `{{$json['Influencer / Brand Name']}}`
    - `{{$json['Max Clips to Generate']}}`
- **Input and output connections:**
  - No incoming connection; this is the workflow trigger
  - Outgoing connection to `2. WayinVideo — Submit Clipping Task`
- **Version-specific requirements:**
  - Uses node type version `2.2`
  - Requires n8n instance support for Form Trigger nodes
- **Edge cases or potential failure types:**
  - User enters a non-video URL or unsupported source URL
  - `Max Clips to Generate` is submitted as non-numeric text; this may break the JSON request body or cause API validation errors
  - If the form webhook is not active/published, submissions will not start the workflow
- **Sub-workflow reference:** None

---

## Block 2 — WayinVideo Task Submission

### Overview
This block sends the user-supplied video and clipping preferences to WayinVideo. It starts an asynchronous clipping job and expects a task identifier in the response.

### Nodes Involved
- `2. WayinVideo — Submit Clipping Task`

### Node Details

#### 2. WayinVideo — Submit Clipping Task
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the WayinVideo API to create a clipping task.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
  - Sends JSON body
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
  - Request body includes:
    - `video_url`: from form input
    - `project_name`: from form input
    - `target_duration`: `DURATION_30_60`
    - `limit`: from form input
    - `enable_export`: `true`
    - `resolution`: `HD_720`
    - `enable_caption`: `true`
    - `enable_ai_reframe`: `true`
    - `ratio`: `RATIO_9_16`
    - `target_lang`: `en`
- **Key expressions or variables used:**
  - `{{$json['Influencer Video URL']}}`
  - `{{$json['Influencer / Brand Name']}}`
  - `{{$json['Max Clips to Generate']}}`
- **Input and output connections:**
  - Input from `1. Form — Video URL + Brand + Clip Count`
  - Output to `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
  - JSON body expression syntax must be supported by this version
- **Edge cases or potential failure types:**
  - Invalid or expired API key
  - Unsupported video URL format
  - WayinVideo API validation error if `limit` is not numeric
  - Rate limiting or API downtime
  - Response shape may differ from expectations; downstream polling assumes `data.id` exists
- **Sub-workflow reference:** None

---

## Block 3 — Polling Loop

### Overview
This block manages the asynchronous nature of clip generation. It waits, checks task status, and repeats until clip data becomes available.

### Nodes Involved
- `3. Wait — 45 Seconds`
- `4. WayinVideo — Poll Clips Result`
- `5. If — Clips Ready?`

### Node Details

#### 3. Wait — 45 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses the workflow between polling attempts.
- **Configuration choices:**
  - Wait amount: `45` seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from:
    - `2. WayinVideo — Submit Clipping Task`
    - `5. If — Clips Ready?` false branch
  - Output to `4. WayinVideo — Poll Clips Result`
- **Version-specific requirements:**
  - Uses node version `1.1`
  - Wait node behavior depends on n8n execution mode and resume support
- **Edge cases or potential failure types:**
  - If the instance loses execution state or wait resumption is misconfigured, polling may not continue
  - Long-running repeated waits can consume execution storage/history
- **Sub-workflow reference:** None

#### 4. WayinVideo — Poll Clips Result
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves the clipping task result from WayinVideo using the task ID returned by the submission step.
- **Configuration choices:**
  - Default method: GET
  - URL dynamically built as:  
    `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{taskId}`
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `{{$('2. WayinVideo — Submit Clipping Task').item.json.data.id}}`
- **Input and output connections:**
  - Input from `3. Wait — 45 Seconds`
  - Output to `5. If — Clips Ready?`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.2`
  - Cross-node item reference via `$('node name').item.json...` must be supported
- **Edge cases or potential failure types:**
  - If the submit node did not return `data.id`, this URL expression fails
  - Invalid credentials or expired token
  - Polling endpoint may return processing state without clips
  - API may return unexpected shape, causing downstream condition issues
- **Sub-workflow reference:** None

#### 5. If — Clips Ready?
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether the polling response contains non-empty `data`, and decides whether to continue or retry.
- **Configuration choices:**
  - Condition: `{{$json.data}}` is `notEmpty`
  - True branch: proceed to clip extraction
  - False branch: return to wait node
- **Key expressions or variables used:**
  - `{{$json.data}}`
- **Input and output connections:**
  - Input from `4. WayinVideo — Poll Clips Result`
  - True output to `6. Code — Extract Clips Array`
  - False output back to `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses IF node version `2.3`
  - Condition engine is configured with strict type validation and version 3 options
- **Edge cases or potential failure types:**
  - `data` may exist even when clips are not actually ready, depending on API response structure
  - If `data` is present but `data.clips` is missing or empty, the next code node may fail or emit no items
  - No retry cap is implemented, so the workflow can loop indefinitely
- **Sub-workflow reference:** None

---

## Block 4 — Clip Extraction, Download, and Google Drive Upload

### Overview
Once clips are ready, this block normalizes the clip list into one item per clip, downloads each file, and uploads it to Google Drive.

### Nodes Involved
- `6. Code — Extract Clips Array`
- `7. HTTP Request — Download Clip File`
- `8. Google Drive — Upload Clip`

### Node Details

#### 6. Code — Extract Clips Array
- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms the WayinVideo result payload into individual n8n items, one per clip.
- **Configuration choices:**
  - JavaScript extracts `const clips = $json.data.clips;`
  - Returns one item per clip with:
    - `title`
    - `export_link`
    - `score`
    - `tags`
    - `desc`
    - `begin_ms`
    - `end_ms`
- **Key expressions or variables used:**
  - `$json.data.clips`
- **Input and output connections:**
  - Input from `5. If — Clips Ready?` true branch
  - Output to `7. HTTP Request — Download Clip File`
- **Version-specific requirements:**
  - Uses Code node version `2`
  - JavaScript runtime must be enabled in n8n
- **Edge cases or potential failure types:**
  - If `$json.data.clips` is undefined or not an array, `.map()` will fail
  - A clip may be missing `export_link` or `title`
  - Empty clips array results in no downstream uploads
- **Sub-workflow reference:** None

#### 7. HTTP Request — Download Clip File
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads each generated clip file as binary data from the provided export link.
- **Configuration choices:**
  - URL: `{{$json.export_link}}`
  - Response format: file/binary
- **Key expressions or variables used:**
  - `{{$json.export_link}}`
- **Input and output connections:**
  - Input from `6. Code — Extract Clips Array`
  - Output to `8. Google Drive — Upload Clip`
- **Version-specific requirements:**
  - Uses HTTP Request node version `4.4`
  - Binary response handling must be supported in this node version
- **Edge cases or potential failure types:**
  - Export link may be expired or inaccessible
  - File download may timeout for large clips
  - If the response is not actually a downloadable file, upload may fail later
  - Some APIs require extra headers for file download; this workflow assumes public/direct access
- **Sub-workflow reference:** None

#### 8. Google Drive — Upload Clip
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the downloaded binary clip file to a target Google Drive folder.
- **Configuration choices:**
  - File name: `{{$('6. Code — Extract Clips Array').item.json.title}}`
  - Drive: `My Drive`
  - Folder: `YOUR_GOOGLE_DRIVE_FOLDER_ID`
  - Uses Google Drive OAuth2 credentials
- **Key expressions or variables used:**
  - `{{$('6. Code — Extract Clips Array').item.json.title}}`
- **Input and output connections:**
  - Input from `7. HTTP Request — Download Clip File`
  - No downstream node
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
  - Requires valid Google Drive OAuth2 credentials configured in n8n
- **Edge cases or potential failure types:**
  - Missing or invalid Google OAuth2 credentials
  - Folder ID not found or insufficient permissions
  - File name collisions may create duplicates or overwrite behavior depending on node defaults
  - If the binary property expected by Google Drive is not present, upload will fail
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main — Overview & Setup | Sticky Note | Documentation and setup guidance embedded in canvas |  |  | ## Influencer Content Repurposing Pipeline<br>Automatically extract viral short clips from any long influencer video using WayinVideo AI, then save each clip to Google Drive.<br><br>### How it works<br>1. Submit a video URL, brand name, and clip count via the web form.<br>2. The workflow calls WayinVideo API to start an AI clipping task.<br>3. It waits 45 seconds, then polls for results.<br>4. If clips are ready, each is downloaded and uploaded to Google Drive. If not ready, it waits and retries.<br><br>### Setup<br>1. Replace **YOUR_WAYINVIDEO_API_KEY** in nodes "2. WayinVideo — Submit Clipping Task" and "4. WayinVideo — Poll Clips Result".<br>2. Connect your Google account in node "8. Google Drive — Upload Clip" via OAuth2.<br>3. Set your Google Drive folder ID in node "8. Google Drive — Upload Clip".<br>4. Optionally adjust clip duration, resolution or ratio in the Submit node.<br><br>### Customization tips<br>- Change `target_duration` to `DURATION_15_30` for shorter TikTok clips.<br>- Add a Slack or email node after upload to notify your team.<br>- Log clip metadata (title, score, tags) to Google Sheets after upload. |
| Section — Input | Sticky Note | Canvas section label for input block |  |  | ## 1. Input<br>User submits a video URL, brand name, and how many clips to generate via a web form. |
| Section — API Submission | Sticky Note | Canvas section label for API submission block |  |  | ## 2. WayinVideo API Submission<br>Sends the video to WayinVideo for AI clipping. Returns a task ID used for polling. |
| Section — Polling Loop | Sticky Note | Canvas section label for polling block |  |  | ## 3. Polling Loop<br>Waits 45 seconds then checks if clips are ready. Loops back if still processing. |
| Section — Download & Upload | Sticky Note | Canvas section label for download/upload block |  |  | ## 4. Extract, Download & Upload<br>Extracts each clip metadata, downloads the video file, then uploads to Google Drive. |
| Warning — Infinite Loop Risk | Sticky Note | Warning note about retry design |  |  | ## ⚠️ WARNING — Infinite Loop Risk<br>If WayinVideo never returns clips (invalid URL or API error), this workflow loops forever. Add a retry counter or max-attempts check in the If node to prevent runaway executions. |
| Warning — Hardcoded API Key | Sticky Note | Security warning about credential handling |  |  | ## ⚠️ WARNING — Hardcoded API Key<br>Do not store the WayinVideo API key in plain text inside node parameters. Move it to an n8n Credential (Header Auth) for security. |
| 1. Form — Video URL + Brand + Clip Count | Form Trigger | Collects user input and starts the workflow |  | 2. WayinVideo — Submit Clipping Task | ## 1. Input<br>User submits a video URL, brand name, and how many clips to generate via a web form. |
| 2. WayinVideo — Submit Clipping Task | HTTP Request | Starts asynchronous WayinVideo clipping job | 1. Form — Video URL + Brand + Clip Count | 3. Wait — 45 Seconds | ## 2. WayinVideo API Submission<br>Sends the video to WayinVideo for AI clipping. Returns a task ID used for polling.<br>## ⚠️ WARNING — Hardcoded API Key<br>Do not store the WayinVideo API key in plain text inside node parameters. Move it to an n8n Credential (Header Auth) for security. |
| 3. Wait — 45 Seconds | Wait | Delays before each poll attempt | 2. WayinVideo — Submit Clipping Task; 5. If — Clips Ready? (false) | 4. WayinVideo — Poll Clips Result | ## 3. Polling Loop<br>Waits 45 seconds then checks if clips are ready. Loops back if still processing. |
| 4. WayinVideo — Poll Clips Result | HTTP Request | Polls WayinVideo for clip-generation result | 3. Wait — 45 Seconds | 5. If — Clips Ready? | ## 3. Polling Loop<br>Waits 45 seconds then checks if clips are ready. Loops back if still processing.<br>## ⚠️ WARNING — Hardcoded API Key<br>Do not store the WayinVideo API key in plain text inside node parameters. Move it to an n8n Credential (Header Auth) for security. |
| 5. If — Clips Ready? | If | Routes execution to extraction or retry loop | 4. WayinVideo — Poll Clips Result | 6. Code — Extract Clips Array; 3. Wait — 45 Seconds | ## 3. Polling Loop<br>Waits 45 seconds then checks if clips are ready. Loops back if still processing.<br>## ⚠️ WARNING — Infinite Loop Risk<br>If WayinVideo never returns clips (invalid URL or API error), this workflow loops forever. Add a retry counter or max-attempts check in the If node to prevent runaway executions. |
| 6. Code — Extract Clips Array | Code | Splits returned clips into one item per clip with selected metadata | 5. If — Clips Ready? (true) | 7. HTTP Request — Download Clip File | ## 4. Extract, Download & Upload<br>Extracts each clip metadata, downloads the video file, then uploads to Google Drive. |
| 7. HTTP Request — Download Clip File | HTTP Request | Downloads each clip as a binary file | 6. Code — Extract Clips Array | 8. Google Drive — Upload Clip | ## 4. Extract, Download & Upload<br>Extracts each clip metadata, downloads the video file, then uploads to Google Drive. |
| 8. Google Drive — Upload Clip | Google Drive | Uploads each downloaded clip to Google Drive | 7. HTTP Request — Download Clip File |  | ## 4. Extract, Download & Upload<br>Extracts each clip metadata, downloads the video file, then uploads to Google Drive. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Repurpose influencer videos into short clips using WayinVideo AI and Google Drive`.

2. **Add a Form Trigger node** named:  
   `1. Form — Video URL + Brand + Clip Count`
   - Set the form title to: `🎬 Influencer Content Repurposing Pipeline`
   - Set the description to:  
     `Paste the influencer's long video URL — AI will extract top viral clips with titles, descriptions, hashtags, and download links.`
   - Add three required form fields:
     1. `Influencer Video URL`
        - Placeholder: `https://www.youtube.com/watch?v=xxxxxxx`
     2. `Influencer / Brand Name`
        - Placeholder: `e.g. top creators, business influencers, podcast hosts`
     3. `Max Clips to Generate`
        - Placeholder: `e.g. 5`

3. **Add an HTTP Request node** named:  
   `2. WayinVideo — Submit Clipping Task`
   - Connect it after the form node.
   - Configure:
     - Method: `POST`
     - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
     - Body content type: JSON
     - Enable sending body
     - Enable sending headers
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Set JSON body to an expression-based object equivalent to:
     - `video_url` = form field `Influencer Video URL`
     - `project_name` = form field `Influencer / Brand Name`
     - `target_duration` = `DURATION_30_60`
     - `limit` = form field `Max Clips to Generate`
     - `enable_export` = `true`
     - `resolution` = `HD_720`
     - `enable_caption` = `true`
     - `enable_ai_reframe` = `true`
     - `ratio` = `RATIO_9_16`
     - `target_lang` = `en`
   - Recommended improvement:
     - Do not hardcode the token.
     - Instead create an **HTTP Header Auth credential** or equivalent secure credential strategy and inject the bearer token there.

4. **Add a Wait node** named:  
   `3. Wait — 45 Seconds`
   - Connect it after the submit node.
   - Set wait amount to `45 seconds`.

5. **Add another HTTP Request node** named:  
   `4. WayinVideo — Poll Clips Result`
   - Connect it after the wait node.
   - Configure:
     - Method: `GET`
     - URL as an expression using the task ID returned by node 2:  
       `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ taskId }}`
   - In n8n expression terms, reference the submission node’s response field `data.id`.
   - Add the same headers as in the submit node:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`

6. **Add an IF node** named:  
   `5. If — Clips Ready?`
   - Connect it after the polling node.
   - Configure the condition to check that `data` is not empty.
   - True branch = clips ready  
   - False branch = not ready yet

7. **Create the retry loop**
   - Connect the **false output** of `5. If — Clips Ready?` back to `3. Wait — 45 Seconds`.
   - This creates the polling loop.

8. **Add a Code node** named:  
   `6. Code — Extract Clips Array`
   - Connect it to the **true output** of the IF node.
   - Use JavaScript that:
     - reads `data.clips`
     - returns one output item per clip
   - Include these fields in each returned item:
     - `title`
     - `export_link`
     - `score`
     - `tags`
     - `desc`
     - `begin_ms`
     - `end_ms`
   - Functional behavior:
     - This converts the API result array into n8n items that can be processed one by one.

9. **Add an HTTP Request node** named:  
   `7. HTTP Request — Download Clip File`
   - Connect it after the code node.
   - Configure:
     - URL = clip `export_link`
     - Response format = `File`
   - This node must download the clip into binary data for the next step.

10. **Add a Google Drive node** named:  
    `8. Google Drive — Upload Clip`
    - Connect it after the download node.
    - Configure Google Drive credentials using **Google OAuth2**.
    - Select:
      - Drive = `My Drive`
      - Folder = your target Google Drive folder
    - Set file name from the clip title generated in the code node.
    - Ensure the node uploads the binary data from the previous HTTP Request node.

11. **Configure Google credentials**
    - In n8n, create or connect a Google Drive OAuth2 credential.
    - Authorize access to the Google account that owns or can write to the destination folder.
    - Confirm the selected folder ID is valid and writable.

12. **Configure WayinVideo authentication securely**
    - Replace `YOUR_TOKEN_HERE` in both HTTP Request nodes.
    - Prefer storing the API key in an n8n credential rather than directly inside the node.
    - If using Header Auth credentials:
      - Header name: `Authorization`
      - Header value: `Bearer <your-token>`

13. **Validate the expected API response structure**
    - Submission node should return something containing `data.id`
    - Poll node should eventually return something containing `data.clips`
    - Each clip should include an `export_link`

14. **Test with a known valid video URL**
    - Submit the form manually.
    - Confirm:
      - the submit call succeeds,
      - the wait resumes,
      - the poll endpoint eventually returns clips,
      - each clip downloads successfully,
      - each file appears in Google Drive.

15. **Add a safety limit to the loop** (strongly recommended)
    - Introduce a retry counter with a Set or Code node before or after polling
    - Update the IF logic to stop after a maximum number of attempts
    - Optionally route failures to Slack, email, or an error-handling branch

16. **Optional enhancements**
    - Change `target_duration` to `DURATION_15_30` for shorter clips
    - Change resolution or ratio to fit another platform
    - Add a Google Sheets node to log:
      - title,
      - score,
      - tags,
      - timestamps,
      - export URL
    - Add Slack/email notification after successful upload

### Credential and dependency expectations

- **WayinVideo**
  - Requires API access and a valid bearer token
  - Both API nodes must use the same authentication method
- **Google Drive**
  - Requires OAuth2 connection
  - Requires target folder selection or folder ID
- **n8n runtime**
  - Must support Form Trigger, Wait, HTTP Request, Code, and Google Drive nodes
  - Wait node resume behavior must be functional in the deployment mode used

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or sub-workflow node. No separate workflow needs to be created.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically extract viral short clips from long influencer videos using WayinVideo AI, then save each clip to Google Drive. | Workflow purpose |
| Replace the placeholder WayinVideo API token in both API nodes before production use. | Security / setup |
| Do not store the WayinVideo API key directly in node parameters; use an n8n credential instead. | Security best practice |
| The workflow currently risks looping forever if WayinVideo never returns clips. Add a retry counter or maximum-attempt threshold. | Reliability / execution safety |
| You can shorten generated clips by changing `target_duration` from `DURATION_30_60` to `DURATION_15_30`. | Customization |
| You can add Slack, email, or Google Sheets steps after upload for notifications or metadata logging. | Extension ideas |