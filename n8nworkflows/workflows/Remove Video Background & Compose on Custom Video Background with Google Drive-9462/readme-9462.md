Remove Video Background & Compose on Custom Video Background with Google Drive

https://n8nworkflows.xyz/workflows/remove-video-background---compose-on-custom-video-background-with-google-drive-9462


# Remove Video Background & Compose on Custom Video Background with Google Drive

### 1. Workflow Overview

This workflow automates the process of removing the background from a foreground video and compositing it onto a custom background video, then mixing audio tracks and saving the final composed video to Google Drive. It is designed for creators who want to enhance video content by replacing backgrounds without complex manual editing, ideal for AI-generated content, e-commerce product demos, social media videos, and webcam footage.

Logical blocks:

- **1.1 Input Reception**: Accepts video URLs either via webhook POST requests or manual entry.
- **1.2 Job Creation & Composition Start**: Submits the foreground video URL to the background removal API, then initiates the composition with a specified template and background video.
- **1.3 Job Status Polling**: Periodically checks the job’s processing status until completion or failure.
- **1.4 Download & Upload**: Downloads the final processed video and uploads it to Google Drive.
- **1.5 Response Handling**: Builds structured success or error responses and returns them to the caller (webhook or manual).
- **1.6 Setup & Documentation**: Sticky notes provide contextual setup instructions, API key management, usage guidelines, and best practices.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Handles incoming video URLs either from a webhook POST or manual trigger, extracting and setting necessary input data variables.

**Nodes Involved:**  
- Webhook Trigger  
- Manual Trigger  
- Extract Webhook Data  
- Sample Video URLs (Edit Here)  
- Merge Triggers  

**Node Details:**

- **Webhook Trigger**  
  - Type: Webhook  
  - Role: Listens for POST requests at `/compose-video`, accepts JSON containing foreground and background video URLs.  
  - Config: HTTP method POST, returns all entries in response mode.  
  - Inputs: External HTTP request  
  - Outputs: Passes webhook data downstream  
  - Edge cases: Missing or malformed JSON, invalid URLs.

- **Manual Trigger**  
  - Type: Manual trigger  
  - Role: Allows manual execution for testing with preset URLs.  
  - Inputs: User-triggered  
  - Outputs: Triggers downstream with sample data.

- **Extract Webhook Data**  
  - Type: Set  
  - Role: Extracts `foreground_video_url` and `background_video_url` from webhook body or falls back to defaults if absent. Also sets a `source` field to `"webhook"`.  
  - Key expressions:  
    - `foreground_video_url = $json.body?.foreground_video_url ?? $json.foreground_video_url ?? default`  
    - `background_video_url = $json.body?.background_video_url ?? $json.background_video_url ?? default`  
  - Inputs: Webhook Trigger  
  - Outputs: Structured JSON with URLs and source tag.

- **Sample Video URLs (Edit Here)**  
  - Type: Set  
  - Role: Provides default video URLs and sets `source` to `"manual"` for manual runs.  
  - Inputs: Manual Trigger  
  - Outputs: Structured JSON with preset URLs and source tag.

- **Merge Triggers**  
  - Type: Merge  
  - Role: Combines outputs from webhook and manual source into one unified stream for downstream processing.  
  - Inputs: Extract Webhook Data, Sample Video URLs  
  - Outputs: Unified JSON with selected video URLs and source.

---

#### 2.2 Job Creation & Composition Start

**Overview:**  
Submits the foreground video URL to the Video Background Remover API to create a job, then starts the composition process with the custom background video and preset parameters.

**Nodes Involved:**  
- 1. Create Job (Upload Foreground)  
- 2. Start Composition  

**Node Details:**

- **1. Create Job (Upload Foreground)**  
  - Type: HTTP Request  
  - Role: Posts the foreground video URL to create a background removal job.  
  - Config: POST to `https://api.videobgremover.com/api/v1/jobs` with JSON body containing `video_url`. Uses API key from environment variable `VIDEOBGREMOVER_KEY` in header.  
  - Inputs: Unified video URLs from Merge Triggers  
  - Outputs: JSON containing job `id`, which is used downstream.  
  - Edge cases: Network errors, invalid API key, invalid video URL, API quota limits.

- **2. Start Composition**  
  - Type: HTTP Request  
  - Role: Starts the video composition on the created job using the `ai_ugc_ad` template, custom background video URL, and audio mixing parameters.  
  - Config: POST to `https://api.videobgremover.com/api/v1/jobs/{job_id}/start` with JSON body specifying:  
    - Template: `ai_ugc_ad`  
    - Background type: Video  
    - Background URL: from input  
    - Background audio enabled, volume 0.3  
    - Export format: H.264 (MP4), preset medium  
  - Inputs: Output of Create Job node  
  - Outputs: JSON with export_id and updated job status.  
  - Edge cases: Invalid job ID, API errors, malformed composition parameters.

---

#### 2.3 Job Status Polling

**Overview:**  
Polls the API for job status every 20 seconds until the job completes successfully or fails.

**Nodes Involved:**  
- 3. Check Job Status  
- Is Complete? (If)  
- Has Failed? (If)  
- Wait 20s  

**Node Details:**

- **3. Check Job Status**  
  - Type: HTTP Request  
  - Role: GET request to check the status of the job using job ID.  
  - Config: GET `https://api.videobgremover.com/api/v1/jobs/{job_id}/status` with API key header.  
  - Inputs: After Start Composition, and from Wait 20s node for retries.  
  - Outputs: JSON including `status` (e.g., `processing`, `completed`, `failed`), processed_video_url, length_seconds.  
  - Edge cases: API rate limits, transient network issues.

- **Is Complete?**  
  - Type: If node  
  - Role: Checks if `status === 'completed'`.  
  - Inputs: From Check Job Status  
  - Outputs:  
    - True: Proceed to download video.  
    - False: Check if failed or continue waiting.

- **Has Failed?**  
  - Type: If node  
  - Role: Checks if `status === 'failed'`.  
  - Inputs: From Is Complete? (false)  
  - Outputs:  
    - True: Build error response.  
    - False: Wait 20 seconds and poll again.

- **Wait 20s**  
  - Type: Wait  
  - Role: Delays workflow for 20 seconds before re-polling job status.  
  - Inputs: Has Failed? (false)  
  - Outputs: Loops back to Check Job Status.

---

#### 2.4 Download & Upload

**Overview:**  
When the job is completed, downloads the processed video file and uploads it to Google Drive.

**Nodes Involved:**  
- 4. Download Video  
- 5. Upload to Google Drive  

**Node Details:**

- **4. Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the processed video file from the URL provided by the job status.  
  - Config: Uses `processed_video_url` from job status response, expects file response type.  
  - Inputs: Is Complete? (true)  
  - Outputs: Binary video data for upload.  
  - Edge cases: Download failures, large file handling, slow network.

- **5. Upload to Google Drive**  
  - Type: Google Drive  
  - Role: Uploads the downloaded video file to Google Drive, with a dynamic filename including job ID and timestamp.  
  - Config: Upload to "My Drive" root folder by default, no file conversion, simplified output enabled.  
  - Inputs: Binary data from Download Video  
  - Outputs: Metadata including file ID and shareable webViewLink.  
  - Edge cases: Google Drive auth errors, quota limits, upload failures.

---

#### 2.5 Response Handling

**Overview:**  
Builds structured success or error responses and returns them either to the webhook caller or stores for manual consumption.

**Nodes Involved:**  
- Build Success Response  
- Build Error Response  
- From Webhook? (If)  
- Respond to Webhook  
- Manual Test Complete  

**Node Details:**

- **Build Success Response**  
  - Type: Set  
  - Role: Constructs an object including success status, job ID, export ID, Google Drive file info, download URL, video length, and source (manual or webhook).  
  - Inputs: Upload to Google Drive output  
  - Outputs: JSON response object.

- **Build Error Response**  
  - Type: Set  
  - Role: Constructs an error response object with failure status, job ID, error message, status, and source.  
  - Inputs: Has Failed? (true)  
  - Outputs: JSON error response.

- **From Webhook?**  
  - Type: If  
  - Role: Checks if the source is `"webhook"` to determine response method.  
  - Inputs: From either success or error response nodes  
  - Outputs:  
    - True: Send response to webhook.  
    - False: Treat as manual execution.

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response back to the webhook caller with HTTP 200.  
  - Inputs: From From Webhook? (true)  
  - Outputs: Ends workflow.

- **Manual Test Complete**  
  - Type: Set  
  - Role: Prepares final result object for manual runs, no HTTP response.  
  - Inputs: From From Webhook? (false)  
  - Outputs: Displays final result in execution logs.

---

#### 2.6 Setup & Documentation (Sticky Notes)

**Overview:**  
Provides detailed instructions and contextual information for users to set up API keys, Google Drive credentials, input requirements, composition parameters, polling logic, and usage scenarios.

**Nodes Involved:**  
- 📋 Overview  
- 🔑 API Key Setup  
- 📥 Inputs  
- 🎨 Composition  
- 🔄 Polling  
- 💾 Google Drive  
- 🚀 Usage  

**Node Details:**  
Each sticky note contains markdown-formatted text explaining setup steps, parameter choices, best practices, and links to official documentation and API management portals. This metadata is crucial for admins and users to correctly operate the workflow.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                        | Input Node(s)                         | Output Node(s)                              | Sticky Note                                               |
|-------------------------------|---------------------|-------------------------------------|-------------------------------------|---------------------------------------------|-----------------------------------------------------------|
| 📋 Overview                   | Sticky Note         | Workflow purpose and use cases      | —                                   | —                                           | 🎬 Video Background Removal & Composition overview        |
| 🔑 API Key Setup              | Sticky Note         | API key acquisition and environment | —                                   | —                                           | Details on obtaining & setting API key                    |
| 📥 Inputs                    | Sticky Note         | Input video requirements            | —                                   | —                                           | Foreground and background video requirements              |
| 🎨 Composition               | Sticky Note         | Composition template and audio mix | —                                   | —                                           | Template `ai_ugc_ad` parameters and audio settings        |
| 🔄 Polling                   | Sticky Note         | Job processing and polling logic   | —                                   | —                                           | Explains job status flow and polling intervals            |
| 💾 Google Drive              | Sticky Note         | Google Drive connection instructions | —                                   | —                                           | Steps to authorize and upload to Google Drive             |
| 🚀 Usage                     | Sticky Note         | Using the workflow                 | —                                   | —                                           | Manual and webhook usage instructions                      |
| Webhook Trigger              | Webhook             | Entry point for webhook POST       | —                                   | Extract Webhook Data                         |                                                           |
| Manual Trigger               | Manual Trigger      | Entry point for manual test         | —                                   | Sample Video URLs (Edit Here)                 |                                                           |
| Extract Webhook Data         | Set                 | Extracts input URLs from webhook    | Webhook Trigger                    | Merge Triggers                              |                                                           |
| Sample Video URLs (Edit Here)| Set                 | Provides default test URLs          | Manual Trigger                     | Merge Triggers                              |                                                           |
| Merge Triggers              | Merge               | Combines webhook/manual inputs      | Extract Webhook Data, Sample Video URLs | 1. Create Job (Upload Foreground)             |                                                           |
| 1. Create Job (Upload Foreground) | HTTP Request        | Creates background removal job      | Merge Triggers                    | 2. Start Composition                        |                                                           |
| 2. Start Composition         | HTTP Request        | Starts video composition job        | 1. Create Job                     | 3. Check Job Status                         |                                                           |
| 3. Check Job Status          | HTTP Request        | Polls for job completion status     | 2. Start Composition, Wait 20s     | Is Complete?, loops to itself via Wait 20s |                                                           |
| Is Complete?                 | If                  | Checks if job status is completed   | 3. Check Job Status                | 4. Download Video, Has Failed?              |                                                           |
| Has Failed?                  | If                  | Checks if job status is failed      | Is Complete?                      | Build Error Response, Wait 20s              |                                                           |
| Wait 20s                    | Wait                | Delays polling retries              | Has Failed? (false)                | 3. Check Job Status                         |                                                           |
| 4. Download Video            | HTTP Request        | Downloads processed video file      | Is Complete?                      | 5. Upload to Google Drive                    |                                                           |
| 5. Upload to Google Drive    | Google Drive        | Uploads video to Google Drive       | 4. Download Video                 | Build Success Response                      |                                                           |
| Build Success Response       | Set                 | Builds success JSON response        | 5. Upload to Google Drive         | From Webhook?                               |                                                           |
| Build Error Response         | Set                 | Builds error JSON response          | Has Failed?                      | From Webhook?                               |                                                           |
| From Webhook?                | If                  | Determines response method          | Build Success Response, Build Error Response | Respond to Webhook, Manual Test Complete     |                                                           |
| Respond to Webhook           | Respond to Webhook  | Sends HTTP response to webhook      | From Webhook? (true)              | —                                           |                                                           |
| Manual Test Complete         | Set                 | Finalizes manual run output         | From Webhook? (false)             | —                                           |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes** for documentation and instructions:  
   - Overview, API Key Setup, Inputs, Composition, Polling, Google Drive, Usage.  
   - Use markdown content as per original notes for user guidance.

2. **Add Entry Points:**  
   - Create a **Webhook Trigger** node:  
     - Path: `compose-video`  
     - Method: POST  
     - Response Data: All entries  
     - Response Mode: Response Node  
   - Create a **Manual Trigger** node for testing.

3. **Extract Input Data:**  
   - Add a **Set** node (named `Extract Webhook Data`) connected to Webhook Trigger.  
   - Configure to assign:  
     - `foreground_video_url` from webhook body or defaults.  
     - `background_video_url` from webhook body or defaults.  
     - `source` = `"webhook"`.  
   - Add a **Set** node (`Sample Video URLs (Edit Here)`) connected to Manual Trigger with hardcoded test URLs and `source` = `"manual"`.

4. **Merge Inputs:**  
   - Add a **Merge** node (`Merge Triggers`) in **Append** mode.  
   - Connect outputs of both Set nodes (webhook and manual) to this Merge node.

5. **Create Job:**  
   - Add **HTTP Request** node (`1. Create Job (Upload Foreground)`) connected to Merge Triggers.  
   - Configure:  
     - URL: `https://api.videobgremover.com/api/v1/jobs`  
     - Method: POST  
     - Body (JSON): `{ "video_url": "{{ $json.foreground_video_url }}" }`  
     - Headers:  
       - `X-API-Key`: use environment variable `$vars.VIDEOBGREMOVER_KEY`  
       - `Content-Type`: `application/json`  
     - Set to never error and expect JSON response.

6. **Start Composition:**  
   - Add **HTTP Request** node (`2. Start Composition`) connected to Create Job.  
   - Configure:  
     - URL: `https://api.videobgremover.com/api/v1/jobs/{{ $json.id }}/start`  
     - Method: POST  
     - Body (JSON):  
       ```json
       {
         "background": {
           "type": "composition",
           "composition": {
             "template": "ai_ugc_ad",
             "background_type": "video",
             "background_url": "{{ $json.background_video_url }}",
             "background_audio_enabled": true,
             "background_audio_volume": 0.3,
             "export_format": "h264",
             "export_preset": "medium"
           }
         }
       }
       ```  
     - Headers: Same as Create Job node.

7. **Check Job Status:**  
   - Add **HTTP Request** node (`3. Check Job Status`) connected to Start Composition and also from Wait node (see below).  
   - Configure:  
     - URL: `https://api.videobgremover.com/api/v1/jobs/{{ $('1. Create Job (Upload Foreground)').item.json.id }}/status`  
     - Method: GET  
     - Headers: API key header  
     - Set never error, JSON response.

8. **Add Conditional Logic:**  
   - **If node (`Is Complete?`)** connected to Check Job Status:  
     - Condition: `$json.status === 'completed'`  
     - True path to Download Video node.  
     - False path to Has Failed? node.  
   - **If node (`Has Failed?`)** connected to Is Complete? false output:  
     - Condition: `$json.status === 'failed'`  
     - True path to Build Error Response.  
     - False path to Wait 20s node.

9. **Wait Node:**  
   - Add **Wait** node (`Wait 20s`) connected from Has Failed? false output.  
   - Configure wait time: 20 seconds.  
   - Connect Wait node back to Check Job Status to poll again.

10. **Download Video:**  
    - Add **HTTP Request** node (`4. Download Video`) connected from Is Complete? true output.  
    - Configure:  
      - URL: `{{$json.processed_video_url}}`  
      - Response format: File download (binary).  

11. **Upload to Google Drive:**  
    - Add **Google Drive** node (`5. Upload to Google Drive`) connected from Download Video.  
    - Configure:  
      - Operation: Upload  
      - Binary property: `data` (from downloaded file)  
      - Filename: `composed_video_{{ $('1. Create Job (Upload Foreground)').item.json.id }}_{{ new Date().getTime() }}.mp4`  
      - Drive and Folder: Default "My Drive" root (or specify folder if desired).  
      - Auth: Connect Google Drive OAuth2 credentials.

12. **Build Responses:**  
    - Add **Set** node (`Build Success Response`) connected from Google Drive upload.  
    - Assign JSON with success info: job ID, export ID, Google Drive file ID and URL, download URL, video length, and source.  
    - Add **Set** node (`Build Error Response`) connected from Has Failed? true output.  
    - Assign JSON with failure info: job ID, error details, status, message, and source.

13. **Response Routing:**  
    - Add **If** node (`From Webhook?`) connected from both success and error response nodes.  
    - Condition: `source === 'webhook'`  
    - True path to **Respond to Webhook** node: Return JSON with HTTP 200.  
    - False path to **Set** node (`Manual Test Complete`): Outputs final JSON for manual viewing.

14. **Connect all flows accordingly** to allow webhook and manual execution paths to merge, process, poll, and respond.

15. **Environment Variables & Credentials:**  
    - Add environment variable `VIDEOBGREMOVER_KEY` with your API key.  
    - Configure Google Drive OAuth2 credentials with proper permissions (file upload).  
    - Test connections before running workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                 |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Full documentation and API details available at: https://docs.videobgremover.com/             | Official product documentation                                  |
| API key management portal: https://videobgremover.com/api-management                          | To obtain and manage your API key                               |
| Workflow processing time is approximately 3-5 minutes per minute of video length               | Performance guideline                                           |
| Pricing details: $0.50-$2.00 per minute processed                                             | Cost considerations                                            |
| Workflow supports both manual test mode and webhook automation for integration flexibility    | Usage modes                                                    |
| Google Drive upload returns permanent shareable links and metadata                            | Output details                                                 |
| Composition template `ai_ugc_ad` places foreground video at bottom-right with 95% opacity     | Template customization details                                 |
| Audio mixing balances background at 30% volume and foreground at full volume                   | Audio configuration                                            |

---

**Disclaimer:** The content herein is derived exclusively from an n8n automated workflow and adheres strictly to content policies. All data processed are public and legally compliant.