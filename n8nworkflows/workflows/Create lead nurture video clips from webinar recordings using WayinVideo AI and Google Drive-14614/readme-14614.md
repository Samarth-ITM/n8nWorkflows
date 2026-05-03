Create lead nurture video clips from webinar recordings using WayinVideo AI and Google Drive

https://n8nworkflows.xyz/workflows/create-lead-nurture-video-clips-from-webinar-recordings-using-wayinvideo-ai-and-google-drive-14614


# Create lead nurture video clips from webinar recordings using WayinVideo AI and Google Drive

# 1. Workflow Overview

This workflow turns a webinar recording URL into short AI-generated video clips for lead nurturing. A user submits webinar details through an n8n hosted form, the workflow sends the source video to the WayinVideo API for clip generation, polls until processing completes, then downloads each exported clip and uploads the files into a Google Drive folder.

Typical use cases:
- Repurposing webinars into short follow-up clips for email sequences
- Creating post-event nurture assets without manual editing
- Building a simple internal content pipeline from recorded webinars

The workflow is organized into four functional blocks.

## 1.1 Input Reception

A hosted n8n form collects the webinar recording URL and related metadata such as title and requested clip count.

## 1.2 Job Submission and Initial Delay

The workflow submits a clip-generation job to the WayinVideo API, then pauses for 45 seconds before checking results for the first time.

## 1.3 Result Polling

The workflow repeatedly queries WayinVideo for the clipping job result. If clips are not ready yet, it loops back to the Wait node and retries after another 45 seconds.

## 1.4 Clip Extraction, Download, and Cloud Storage

Once clips are available, the workflow extracts the returned clips array, creates one item per clip, downloads each clip file, and uploads each file to a target Google Drive folder.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block provides the public entry point to the workflow. It gathers all user inputs required to create a WayinVideo clipping job.

### Nodes Involved
- `1. Form — Webinar URL + Details`

### Node Details

#### 1. Form — Webinar URL + Details
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes a hosted web form and starts the workflow on submission.
- **Configuration choices:**
  - Form title: `🎯 Webinar Recording to Lead Nurture Clips`
  - Description explains that AI will extract engaging clips from a webinar recording URL
  - Four required fields:
    - `Webinar Recording URL`
    - `Webinar Topic / Title`
    - `Company / Brand Name`
    - `Max Clips to Generate`
- **Key expressions or variables used:**
  - Later nodes reference submitted values using:
    - `$json['Webinar Recording URL']`
    - `$json['Webinar Topic / Title']`
    - `$json['Max Clips to Generate']`
  - `Company / Brand Name` is collected but not used by downstream nodes in the current version
- **Input and output connections:**
  - Input: none, this is the workflow trigger
  - Output: `2. WayinVideo — Submit Clipping Task1`
- **Version-specific requirements:**
  - Uses Form Trigger version `2.2`
  - Requires n8n instance support for hosted forms
- **Edge cases or potential failure types:**
  - Invalid or inaccessible webinar URL may only fail later at the API stage
  - `Max Clips to Generate` is a text form field, so invalid numeric input could break the JSON body or trigger API validation errors
  - Public form access may allow malformed user submissions if not additionally validated
- **Sub-workflow reference:** none

---

## Block 2 — Job Submission and Initial Delay

### Overview
This block sends the webinar recording to WayinVideo for AI clip extraction and then introduces a fixed wait before polling begins. The delay reduces the chance of checking the job too early.

### Nodes Involved
- `2. WayinVideo — Submit Clipping Task1`
- `3. Wait — 45 Seconds`

### Node Details

#### 2. WayinVideo — Submit Clipping Task1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to create a clip-generation task in the WayinVideo API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
  - Sends JSON body
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
  - Body fields:
    - `video_url`: from form submission
    - `project_name`: from webinar title
    - `target_duration`: `DURATION_30_60`
    - `limit`: from max clips field
    - `enable_export`: `true`
    - `resolution`: `HD_720`
    - `enable_caption`: `true`
    - `enable_ai_reframe`: `false`
    - `target_lang`: `en`
- **Key expressions or variables used:**
  - `{{ $json['Webinar Recording URL'] }}`
  - `{{ $json['Webinar Topic / Title'] }}`
  - `{{ $json['Max Clips to Generate'] }}`
- **Input and output connections:**
  - Input: `1. Form — Webinar URL + Details`
  - Output: `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses HTTP Request version `4.2`
  - Assumes the API accepts bearer-token authentication through manual headers
- **Edge cases or potential failure types:**
  - Invalid bearer token
  - API version mismatch
  - Unsupported recording URL or inaccessible media source
  - JSON body failure if `Max Clips to Generate` is non-numeric
  - API-side quota/rate-limit or content-processing errors
  - Timeout on long API response
- **Sub-workflow reference:** none

#### 3. Wait — 45 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses the workflow execution before polling starts, and also serves as the retry delay in the polling loop.
- **Configuration choices:**
  - Wait amount: `45` seconds
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Inputs:
    - `2. WayinVideo — Submit Clipping Task1`
    - `5. If — Clips Ready Check` false branch
  - Output: `4. WayinVideo — Poll Clip Results`
- **Version-specific requirements:**
  - Uses Wait node version `1.1`
- **Edge cases or potential failure types:**
  - Requires proper n8n execution resumption behavior
  - If the workflow is disabled or execution retention is affected, resume behavior may differ by deployment
  - Repeated loop usage can consume execution resources over time
- **Sub-workflow reference:** none

---

## Block 3 — Result Polling

### Overview
This block checks whether WayinVideo has completed clip generation. If the expected data is present, the workflow continues; otherwise, it re-enters the wait loop.

### Nodes Involved
- `4. WayinVideo — Poll Clip Results`
- `5. If — Clips Ready Check`

### Node Details

#### 4. WayinVideo — Poll Clip Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a GET-style request to fetch processing results for the previously created clipping task.
- **Configuration choices:**
  - URL is dynamically built using the task ID returned by the submit node
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `x-wayinvideo-api-version: v2`
    - `Content-Type: application/json`
- **Key expressions or variables used:**
  - `https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $('2. WayinVideo — Submit Clipping Task1').item.json.data.id }}`
  - This explicitly references the submit node’s returned job ID
- **Input and output connections:**
  - Input: `3. Wait — 45 Seconds`
  - Output: `5. If — Clips Ready Check`
- **Version-specific requirements:**
  - Uses HTTP Request version `4.2`
- **Edge cases or potential failure types:**
  - Missing `data.id` in the original submission response
  - Authentication failure
  - API returns unexpected schema or a transient processing status
  - Network timeout or rate limiting during repeated polling
- **Sub-workflow reference:** none

#### 5. If — Clips Ready Check
- **Type and technical role:** `n8n-nodes-base.if`  
  Tests whether the polling response includes a non-empty `data` object.
- **Configuration choices:**
  - Condition type uses object operation `notEmpty`
  - Evaluates `={{ $json.data }}`
  - True branch means clips are considered ready
  - False branch loops back to the Wait node
- **Key expressions or variables used:**
  - `={{ $json.data }}`
- **Input and output connections:**
  - Input: `4. WayinVideo — Poll Clip Results`
  - True output: `6. Code — Extract Clips Array1`
  - False output: `3. Wait — 45 Seconds`
- **Version-specific requirements:**
  - Uses If node version `2.3`
- **Edge cases or potential failure types:**
  - The condition checks whether `data` is non-empty, not specifically whether `data.clips` exists and is complete
  - If the API returns partial or error-shaped data, the workflow may proceed too early
  - If the API never returns usable data, this creates an infinite loop
- **Sub-workflow reference:** none

**Important risk:**  
The workflow includes a documented infinite polling risk. If WayinVideo never completes successfully, the loop continues indefinitely. A retry counter is strongly recommended.

---

## Block 4 — Clip Extraction, Download, and Cloud Storage

### Overview
After clips become available, this block transforms the WayinVideo response into one n8n item per clip, downloads each file as binary data, and uploads it into Google Drive.

### Nodes Involved
- `6. Code — Extract Clips Array1`
- `7. HTTP — Download Clip File`
- `8. Google Drive — Upload Clip1`

### Node Details

#### 6. Code — Extract Clips Array1
- **Type and technical role:** `n8n-nodes-base.code`  
  Reads the returned clips array and emits one output item per clip.
- **Configuration choices:**
  - JavaScript code:
    - Reads `const clips = $json.data.clips;`
    - Maps each clip to a simplified JSON object containing:
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
  - Input: `5. If — Clips Ready Check` true branch
  - Output: `7. HTTP — Download Clip File`
- **Version-specific requirements:**
  - Uses Code node version `2`
  - Requires JavaScript execution enabled in the environment
- **Edge cases or potential failure types:**
  - If `data.clips` is undefined or not an array, the node will throw an error
  - Empty clip arrays will produce no output items
  - Missing `export_link` on any clip will break downstream downloading
- **Sub-workflow reference:** none

#### 7. HTTP — Download Clip File
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads each clip file from its exported file URL and stores the result as binary data.
- **Configuration choices:**
  - URL comes from the clip item’s `export_link`
  - Response format is configured as `file`
- **Key expressions or variables used:**
  - `={{ $json.export_link }}`
- **Input and output connections:**
  - Input: `6. Code — Extract Clips Array1`
  - Output: `8. Google Drive — Upload Clip1`
- **Version-specific requirements:**
  - Uses HTTP Request version `4.4`
- **Edge cases or potential failure types:**
  - Expired or invalid export link
  - Large file download issues
  - Timeout or remote CDN failures
  - Missing binary output if the response is not an actual file
- **Sub-workflow reference:** none

#### 8. Google Drive — Upload Clip1
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the downloaded binary clip file into a specified Google Drive folder.
- **Configuration choices:**
  - File name comes from the extracted clip title
  - Drive: `My Drive`
  - Folder ID must be replaced with the target folder
  - Uses Google Drive OAuth2 credentials
- **Key expressions or variables used:**
  - File name: `={{ $('6. Code — Extract Clips Array1').item.json.title }}`
- **Input and output connections:**
  - Input: `7. HTTP — Download Clip File`
  - Output: none
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
  - Requires a valid Google Drive OAuth2 credential in n8n
  - Assumes the node is configured to upload the incoming binary file
- **Edge cases or potential failure types:**
  - Missing or invalid Google OAuth2 credentials
  - Folder ID not found or inaccessible
  - File name collisions depending on Drive behavior
  - Upload failure if binary property mapping is incorrect or absent
- **Sub-workflow reference:** none

**Implementation note:**  
In a real rebuild, confirm the Google Drive node’s upload operation and binary property configuration explicitly. The JSON excerpt shows name and folder settings, but the upload action depends on proper binary handling in the node UI.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main — Overview & Setup | Sticky Note | Workspace documentation and setup guidance |  |  | ## Webinar Recording → Lead Nurture Clips<br>### How it works<br>This workflow accepts a webinar recording URL via a hosted form and submits it to the WayinVideo AI API to extract the most engaging clip segments. It polls every 45 seconds until the clips are ready, then downloads each clip file and uploads it directly to a Google Drive folder — ready to use in email lead nurture sequences.<br>### Setup<br>1. Replace `YOUR_WAYINVIDEO_API_KEY` in nodes **2** and **4** with your WayinVideo bearer token<br>2. Create a **Google Drive OAuth2** credential in n8n and connect it to node **8**<br>3. Replace `YOUR_GOOGLE_DRIVE_FOLDER_ID` in node **8** with your target Drive folder ID<br>4. Activate the workflow and open the form URL to submit your first webinar recording<br>### Customization Tips<br>- Change `target_duration` to `DURATION_15_30` for shorter social media clips<br>- Set `enable_ai_reframe` to `true` for vertical/mobile-optimized output<br>- Add a Gmail or Mailchimp node after node 8 to automatically email clip links to leads |
| Section — Input | Sticky Note | Section label for input collection |  |  | ## 📥 Section 1 — Input<br>Collects the webinar recording URL, topic, brand name, and max clip count via a hosted n8n form. |
| Section — Submit & Wait | Sticky Note | Section label for submission and first delay |  |  | ## 🚀 Section 2 — Submit & Wait<br>Sends the clipping job to the WayinVideo API, then waits 45 seconds before the first result check. |
| Section — Poll Results | Sticky Note | Section label for polling loop |  |  | ## 🔄 Section 3 — Poll for Results<br>Checks whether clips are ready. If not, loops back to the Wait node and retries every 45 seconds. |
| Section — Extract & Upload | Sticky Note | Section label for clip extraction and storage |  |  | ## 📤 Section 4 — Extract, Download & Upload<br>Splits the clips array, downloads each video file, and uploads it to the configured Google Drive folder. |
| Warning — Infinite Loop Risk | Sticky Note | Operational warning about unbounded polling |  |  | ## ⚠️ WARNING — Infinite Polling Loop Risk<br>If WayinVideo never returns a completed result (e.g. invalid URL, unsupported format, API error), this workflow will loop indefinitely and consume execution credits. Add a retry counter using a Set node + a second If node to cap retries at 10–15 attempts maximum. |
| 1. Form — Webinar URL + Details | Form Trigger | Collects webinar input from a hosted form and starts the workflow |  | 2. WayinVideo — Submit Clipping Task1 | ## 📥 Section 1 — Input<br>Collects the webinar recording URL, topic, brand name, and max clip count via a hosted n8n form. |
| 2. WayinVideo — Submit Clipping Task1 | HTTP Request | Creates the WayinVideo clipping task | 1. Form — Webinar URL + Details | 3. Wait — 45 Seconds | ## 🚀 Section 2 — Submit & Wait<br>Sends the clipping job to the WayinVideo API, then waits 45 seconds before the first result check. |
| 3. Wait — 45 Seconds | Wait | Delays first poll and acts as retry delay in loop | 2. WayinVideo — Submit Clipping Task1; 5. If — Clips Ready Check | 4. WayinVideo — Poll Clip Results | ## 🚀 Section 2 — Submit & Wait<br>Sends the clipping job to the WayinVideo API, then waits 45 seconds before the first result check.<br>## ⚠️ WARNING — Infinite Polling Loop Risk<br>If WayinVideo never returns a completed result (e.g. invalid URL, unsupported format, API error), this workflow will loop indefinitely and consume execution credits. Add a retry counter using a Set node + a second If node to cap retries at 10–15 attempts maximum. |
| 4. WayinVideo — Poll Clip Results | HTTP Request | Polls WayinVideo for task completion and clip results | 3. Wait — 45 Seconds | 5. If — Clips Ready Check | ## 🔄 Section 3 — Poll for Results<br>Checks whether clips are ready. If not, loops back to the Wait node and retries every 45 seconds.<br>## ⚠️ WARNING — Infinite Polling Loop Risk<br>If WayinVideo never returns a completed result (e.g. invalid URL, unsupported format, API error), this workflow will loop indefinitely and consume execution credits. Add a retry counter using a Set node + a second If node to cap retries at 10–15 attempts maximum. |
| 5. If — Clips Ready Check | If | Determines whether returned result data is present or whether polling should continue | 4. WayinVideo — Poll Clip Results | 6. Code — Extract Clips Array1; 3. Wait — 45 Seconds | ## 🔄 Section 3 — Poll for Results<br>Checks whether clips are ready. If not, loops back to the Wait node and retries every 45 seconds.<br>## ⚠️ WARNING — Infinite Polling Loop Risk<br>If WayinVideo never returns a completed result (e.g. invalid URL, unsupported format, API error), this workflow will loop indefinitely and consume execution credits. Add a retry counter using a Set node + a second If node to cap retries at 10–15 attempts maximum. |
| 6. Code — Extract Clips Array1 | Code | Splits the returned clips array into one item per clip | 5. If — Clips Ready Check | 7. HTTP — Download Clip File | ## 📤 Section 4 — Extract, Download & Upload<br>Splits the clips array, downloads each video file, and uploads it to the configured Google Drive folder. |
| 7. HTTP — Download Clip File | HTTP Request | Downloads each exported clip as a file | 6. Code — Extract Clips Array1 | 8. Google Drive — Upload Clip1 | ## 📤 Section 4 — Extract, Download & Upload<br>Splits the clips array, downloads each video file, and uploads it to the configured Google Drive folder. |
| 8. Google Drive — Upload Clip1 | Google Drive | Uploads each downloaded clip to Google Drive | 7. HTTP — Download Clip File |  | ## 📤 Section 4 — Extract, Download & Upload<br>Splits the clips array, downloads each video file, and uploads it to the configured Google Drive folder. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it: `Create lead nurture video clips from webinar recordings using WayinVideo AI and Google Drive`.

2. **Add a Form Trigger node**.
   - Node type: `Form Trigger`
   - Name it: `1. Form — Webinar URL + Details`
   - Set the form title to: `🎯 Webinar Recording to Lead Nurture Clips`
   - Set the description to: `Paste your webinar recording URL — AI will extract the top engaging clips ready for your email lead nurture sequence.`
   - Add these required fields:
     1. `Webinar Recording URL`
     2. `Webinar Topic / Title`
     3. `Company / Brand Name`
     4. `Max Clips to Generate`
   - Keep all fields required.
   - Save the node.

3. **Add the first HTTP Request node** to submit the clipping task.
   - Node type: `HTTP Request`
   - Name it: `2. WayinVideo — Submit Clipping Task1`
   - Connect `1. Form — Webinar URL + Details` → this node.
   - Configure:
     - Method: `POST`
     - URL: `https://wayinvideo-api.wayin.ai/api/v2/clips`
     - Send body: enabled
     - Body content type: JSON
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Set the JSON body to:
     - `video_url` from `Webinar Recording URL`
     - `project_name` from `Webinar Topic / Title`
     - `target_duration` = `DURATION_30_60`
     - `limit` from `Max Clips to Generate`
     - `enable_export` = `true`
     - `resolution` = `HD_720`
     - `enable_caption` = `true`
     - `enable_ai_reframe` = `false`
     - `target_lang` = `en`
   - Use expressions:
     - `{{$json['Webinar Recording URL']}}`
     - `{{$json['Webinar Topic / Title']}}`
     - `{{$json['Max Clips to Generate']}}`

4. **Add a Wait node**.
   - Node type: `Wait`
   - Name it: `3. Wait — 45 Seconds`
   - Connect `2. WayinVideo — Submit Clipping Task1` → this node.
   - Configure it to wait for `45 seconds`.

5. **Add a second HTTP Request node** for polling.
   - Node type: `HTTP Request`
   - Name it: `4. WayinVideo — Poll Clip Results`
   - Connect `3. Wait — 45 Seconds` → this node.
   - Configure:
     - URL as expression:  
       `=https://wayinvideo-api.wayin.ai/api/v2/clips/results/{{ $('2. WayinVideo — Submit Clipping Task1').item.json.data.id }}`
   - Add headers:
     - `Authorization` = `Bearer YOUR_TOKEN_HERE`
     - `x-wayinvideo-api-version` = `v2`
     - `Content-Type` = `application/json`
   - Leave method as default GET unless your API account documentation requires otherwise.

6. **Add an If node** to check readiness.
   - Node type: `If`
   - Name it: `5. If — Clips Ready Check`
   - Connect `4. WayinVideo — Poll Clip Results` → this node.
   - Add a condition:
     - Left value: `{{$json.data}}`
     - Operator: `is not empty` / object `notEmpty`
   - True branch = clips ready
   - False branch = not ready

7. **Create the polling loop**.
   - Connect the **false** output of `5. If — Clips Ready Check` back to `3. Wait — 45 Seconds`.
   - This creates the retry cycle:
     - Wait
     - Poll
     - Check
     - Loop if still not ready

8. **Add a Code node** to flatten the clips list.
   - Node type: `Code`
   - Name it: `6. Code — Extract Clips Array1`
   - Connect the **true** output of `5. If — Clips Ready Check` → this node.
   - Paste this JavaScript logic:
     - Read `data.clips`
     - Return one item per clip
     - Include fields:
       - `title`
       - `export_link`
       - `score`
       - `tags`
       - `desc`
       - `begin_ms`
       - `end_ms`
   - Equivalent logic:
     - `const clips = $json.data.clips;`
     - `return clips.map(clip => ({ json: { ... } }))`

9. **Add an HTTP Request node to download the clip file**.
   - Node type: `HTTP Request`
   - Name it: `7. HTTP — Download Clip File`
   - Connect `6. Code — Extract Clips Array1` → this node.
   - Configure:
     - URL: `{{$json.export_link}}`
     - Response format: `File`
   - This ensures each clip download becomes binary data for the next node.

10. **Create Google Drive credentials** in n8n.
    - Credential type: `Google Drive OAuth2`
    - Authenticate with the Google account that owns or can edit the target folder.
    - Ensure the scope allows file upload.

11. **Add a Google Drive node**.
    - Node type: `Google Drive`
    - Name it: `8. Google Drive — Upload Clip1`
    - Connect `7. HTTP — Download Clip File` → this node.
    - Attach the Google Drive OAuth2 credential created above.
    - Configure the node to upload a file from binary input.
    - Set:
      - Drive: `My Drive`
      - Folder: your target folder
      - File name: use the clip title with expression  
        `{{$('6. Code — Extract Clips Array1').item.json.title}}`
    - Replace the placeholder folder ID with your actual Google Drive folder ID.

12. **Verify binary upload settings** in the Google Drive node.
    - Confirm the node is using the binary property produced by the download node.
    - If needed, explicitly set the binary property name used by the HTTP download node.
    - If your environment defaults to `data`, confirm that property exists in execution output.

13. **Replace placeholders**.
    - In node `2. WayinVideo — Submit Clipping Task1`, replace `YOUR_TOKEN_HERE` with the real WayinVideo bearer token.
    - In node `4. WayinVideo — Poll Clip Results`, replace `YOUR_TOKEN_HERE` with the same token.
    - In node `8. Google Drive — Upload Clip1`, replace the placeholder folder selection with your real folder.

14. **Test the workflow**.
    - Submit the form with:
      - A publicly accessible webinar recording URL
      - A title
      - A reasonable clip count such as `3` or `5`
    - Confirm:
      - The submit node returns a job ID
      - The polling node eventually returns `data.clips`
      - The download node produces binary files
      - Google Drive receives the uploaded clip files

15. **Activate the workflow**.
    - Once validated, activate it so the hosted form endpoint is publicly usable.

16. **Strongly recommended hardening step: add retry limits**.
    - Add a `Set` node before or after the Wait node to maintain a retry counter.
    - Add another `If` node to stop after 10 to 15 attempts.
    - Route the exceeded-retry branch to an error notification node or a graceful end state.
    - This prevents infinite looping if WayinVideo never finishes successfully.

17. **Optional improvements**.
    - Use `Company / Brand Name` in:
      - Drive file naming
      - Folder routing
      - Email metadata
    - Change `target_duration` to `DURATION_15_30` for shorter clips
    - Set `enable_ai_reframe` to `true` for mobile/vertical content
    - Add Gmail, Outlook, Mailchimp, or Slack after Google Drive to distribute uploaded clip links automatically

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not require a separate callable workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace the WayinVideo bearer token in both API nodes before activation. | WayinVideo API authentication |
| Connect a Google Drive OAuth2 credential to the upload node and verify folder access. | Google Drive integration |
| The collected field `Company / Brand Name` is currently unused in downstream logic and can be leveraged for naming, routing, or segmentation. | Internal workflow design note |
| The polling loop has no built-in retry cap and may run indefinitely if processing never completes. | Operational risk |
| Customization tip: set `target_duration` to `DURATION_15_30` for shorter clips. | WayinVideo request body |
| Customization tip: set `enable_ai_reframe` to `true` for vertical/mobile-optimized output. | WayinVideo request body |
| You can add Gmail or Mailchimp after the Google Drive node to automatically distribute generated clip links to leads. | Post-processing extension |