Video Background Removal and Image Composition with Google Drive Integration

https://n8nworkflows.xyz/workflows/video-background-removal-and-image-composition-with-google-drive-integration-9820


# Video Background Removal and Image Composition with Google Drive Integration

### 1. Workflow Overview

This workflow automates the removal of video backgrounds and the composition of the foreground onto a custom image background, followed by saving the final output to Google Drive. It is designed to handle a wide range of video types such as AI-generated actors, product demos, webcam footage, and profile videos. The processed videos are composited using customizable templates and exported in MP4 format.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Accepts input either via webhook or manual trigger with URLs for the foreground video and background image.
- **1.2 Job Creation and Image Composition:** Submits the video to the VideoBGRemover API, initiates background removal and composition with the specified background image and template.
- **1.3 Job Status Polling:** Periodically checks the job status until completion or failure.
- **1.4 Download and Upload:** Downloads the processed video and uploads it to Google Drive.
- **1.5 Response and Error Handling:** Sends back results or error messages either as webhook responses or manual output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block gathers the required inputs — URLs for the foreground video and background image — from two sources: a webhook POST request or a manual test trigger. It normalizes and merges the data for downstream processing.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Manual Trigger  
  - Extract Webhook Data (Set)  
  - Sample URLs (Set)  
  - Merge Triggers  

- **Node Details:**

  - **Webhook Trigger**  
    - Type: Webhook Trigger  
    - Role: Entry point for automated processing via HTTP POST requests.  
    - Configuration: Listens on path `/compose-video-image`, accepts POST, returns all entries as response data.  
    - Inputs: HTTP POST data payload.  
    - Outputs: JSON containing video and image URLs (expected keys: `foreground_video_url`, `background_image_url`).  
    - Edge Cases: Missing or malformed JSON input; fallback handled in Extract Webhook Data node.

  - **Manual Trigger**  
    - Type: Manual Trigger  
    - Role: Allows manual testing of the workflow with preset sample URLs.  
    - Configuration: No parameters.  
    - Inputs: None, triggers manually.  
    - Outputs: Triggers the Sample URLs node.

  - **Extract Webhook Data (Set)**  
    - Type: Set  
    - Role: Extracts and normalizes input URLs from webhook data, providing defaults if missing.  
    - Configuration: Uses expressions to access nested JSON properties safely, defaults to sample video and image URLs if inputs are missing.  
    - Outputs: Variables `foreground_video_url`, `background_image_url`, and source marker `"webhook"`.  
    - Edge Cases: Handles missing or incorrectly named JSON keys gracefully.

  - **Sample URLs (Set)**  
    - Type: Set  
    - Role: Sets static sample URLs for manual testing.  
    - Configuration: Hardcoded URLs for foreground video and background image, source marked as `"manual"`.  
    - Outputs: Same variables as above for uniform downstream processing.

  - **Merge Triggers**  
    - Type: Merge  
    - Role: Combines webhook and manual trigger data streams into a single unified flow.  
    - Configuration: Append mode to allow either input to proceed.  
    - Inputs: From Extract Webhook Data and Sample URLs.  
    - Outputs: Unified normalized input object.

---

#### 2.2 Job Creation and Image Composition

- **Overview:**  
  This block creates a job on the VideoBGRemover API by uploading the foreground video URL, then starts the image composition process specifying background parameters and export settings.

- **Nodes Involved:**  
  - 1. Create Job (Upload Foreground) (HTTP Request)  
  - 2. Start Image Composition (HTTP Request)

- **Node Details:**

  - **1. Create Job (Upload Foreground)**  
    - Type: HTTP Request  
    - Role: Sends POST request to create a new background removal job with the foreground video URL.  
    - Configuration:  
      - URL: `https://api.videobgremover.com/api/v1/jobs`  
      - Method: POST  
      - Headers: `X-API-Key` from environment variable, `Content-Type: application/json`  
      - Body: JSON with `"video_url"` set from input data.  
      - Response: JSON, never errors (to allow workflow to handle API errors explicitly).  
    - Inputs: Foreground video URL from merged input.  
    - Outputs: JSON containing job ID and status.  
    - Edge Cases: API key missing/invalid, network timeout, invalid video URL.

  - **2. Start Image Composition**  
    - Type: HTTP Request  
    - Role: Initiates the composition step on the created job, applying the background image and composition template.  
    - Configuration:  
      - URL: `https://api.videobgremover.com/api/v1/jobs/{{job_id}}/start` (dynamic)  
      - Method: POST  
      - Headers: Same as above  
      - Body: JSON specifying background as image composition with template `centered`, background image URL, and export format (H.264 MP4), preset medium.  
    - Inputs: Job ID from previous node, background image URL from merged input.  
    - Outputs: JSON including export ID and job status.  
    - Edge Cases: Invalid job ID, invalid background URL, API errors.

---

#### 2.3 Job Status Polling

- **Overview:**  
  This block repeatedly checks the status of the job every 20 seconds until it is either completed or failed. It branches accordingly to handle success or failure.

- **Nodes Involved:**  
  - 3. Check Job Status (HTTP Request)  
  - Is Complete? (If)  
  - Has Failed? (If)  
  - Wait 20s (Wait)  
  - Build Error Response (Set)

- **Node Details:**

  - **3. Check Job Status**  
    - Type: HTTP Request  
    - Role: GET request to retrieve current job status and metadata.  
    - Configuration:  
      - URL: `https://api.videobgremover.com/api/v1/jobs/{{job_id}}/status`  
      - Method: GET  
      - Headers: Include API key  
      - Response: JSON, never errors.  
    - Inputs: Job ID from create job node.  
    - Outputs: JSON with status field (`processing`, `completed`, or `failed`), and additional metadata like processed video URL and video length.  
    - Edge Cases: Network issues, API errors.

  - **Is Complete?**  
    - Type: If  
    - Role: Checks if job status equals `completed`.  
    - Branches: Yes (completed), No (others).  
    - Inputs: Status from job status check.

  - **Has Failed?**  
    - Type: If  
    - Role: Checks if job status equals `failed`.  
    - Branches: Yes (failure), No (continue polling).  
    - Inputs: Status from job status check.

  - **Wait 20s**  
    - Type: Wait  
    - Role: Pauses workflow for 20 seconds before retrying status check.  
    - Inputs: From Has Failed? node if job is not failed and not completed.  
    - Outputs: Loops back to Check Job Status node.

  - **Build Error Response**  
    - Type: Set  
    - Role: Constructs a structured error response object with job ID, status, error message, and source.  
    - Inputs: Job status and error details from API.  
    - Outputs: Final error response for webhook or manual output.

---

#### 2.4 Download and Upload

- **Overview:**  
  Upon successful completion, this block downloads the composited video file and uploads it to Google Drive, generating shareable links and metadata.

- **Nodes Involved:**  
  - 4. Download Video (HTTP Request)  
  - 5. Upload to Google Drive (Google Drive node)  
  - Build Success Response (Set)

- **Node Details:**

  - **4. Download Video**  
    - Type: HTTP Request  
    - Role: Downloads the processed video file from the URL provided by the API.  
    - Configuration:  
      - URL: from job status node’s `processed_video_url`  
      - Response: File, binary data expected.  
    - Inputs: Processed video URL.  
    - Outputs: Binary data of the video for upload.  
    - Edge Cases: Download failure, file corruption.

  - **5. Upload to Google Drive**  
    - Type: Google Drive  
    - Role: Uploads downloaded video file to Google Drive.  
    - Configuration:  
      - Operation: Upload  
      - File name: Dynamic including job ID and timestamp with `.mp4` extension  
      - Folder: Root of “My Drive” by default (can be customized)  
      - Credentials: Google Drive OAuth2 credentials must be configured in n8n.  
      - Binary Property: `data` (binary data from download node).  
    - Inputs: Binary video data.  
    - Outputs: Metadata including Google Drive file ID and shareable webViewLink.  
    - Edge Cases: Authentication errors, quota limits, upload failures.

  - **Build Success Response**  
    - Type: Set  
    - Role: Creates a structured success response object including job ID, export ID, Google Drive file info, download URL, video length, and source.  
    - Inputs: Outputs from upload and previous nodes.  
    - Outputs: Final success response for webhook or manual output.

---

#### 2.5 Response and Error Handling

- **Overview:**  
  Sends final success or error responses back to the caller. Distinguishes between webhook-triggered runs and manual tests.

- **Nodes Involved:**  
  - From Webhook? (If)  
  - Respond to Webhook  
  - Manual Test Complete (Set)

- **Node Details:**

  - **From Webhook?**  
    - Type: If  
    - Role: Checks whether the source of the input was webhook or manual (based on the `source` field).  
    - Branches:  
      - Yes: For webhook origin, send HTTP response.  
      - No: For manual origin, prepare output data.  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Returns HTTP response with final JSON payload and status code 200.  
    - Inputs: Final result object from success or error build nodes.

  - **Manual Test Complete**  
    - Type: Set  
    - Role: Prepares final result object as output data for manual execution.  
    - Inputs: Final result object.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                  |
|-------------------------------|---------------------|----------------------------------------|--------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| 📋 Overview                   | Sticky Note         | General workflow information            |                                |                                   | ## 🎬 Video Background Removal & Image Composition ... [📖 Full Documentation](https://docs.videobgremover.com/) |
| 🔑 API Key Setup              | Sticky Note         | Instructions for API key setup          |                                |                                   | ## 🔑 API Key Setup (Required) ... Uses `$vars.VIDEOBGREMOVER_KEY` (secure)                                   |
| 📥 Inputs                    | Sticky Note         | Input requirements description          |                                |                                   | ## 📥 Input Required ... Both must be **publicly accessible URLs**                                           |
| 🎨 Composition               | Sticky Note         | Composition template details             |                                |                                   | ## 🎨 Composition Template ... Customize in the "Start Composition" node                                    |
| 🔄 Polling                   | Sticky Note         | Explanation of processing & polling     |                                |                                   | ## 🔄 Processing & Polling ... Typical: 3-5 min per min of video                                             |
| 💾 Google Drive              | Sticky Note         | Google Drive setup instructions         |                                |                                   | ## 💾 Google Drive Setup ... Setup time: ~2 minutes                                                         |
| 🚀 Usage                     | Sticky Note         | How to use manual and webhook triggers  |                                |                                   | ## 🚀 How to Use ... Batch processing with Google Sheets or Airtable                                         |
| Webhook Trigger              | Webhook             | Receives webhook POST input              |                                | Extract Webhook Data              |                                                                                                              |
| Manual Trigger               | Manual Trigger      | Manual workflow execution trigger        |                                | Sample URLs (Edit Here)           |                                                                                                              |
| Extract Webhook Data         | Set                 | Normalizes webhook input data            | Webhook Trigger                | Merge Triggers                   |                                                                                                              |
| Sample URLs (Edit Here)      | Set                 | Provides sample URLs for manual tests    | Manual Trigger                 | Merge Triggers                   |                                                                                                              |
| Merge Triggers              | Merge               | Combines webhook/manual data             | Extract Webhook Data, Sample URLs | 1. Create Job (Upload Foreground) |                                                                                                              |
| 1. Create Job (Upload Foreground) | HTTP Request       | Creates background removal job            | Merge Triggers                | 2. Start Image Composition        |                                                                                                              |
| 2. Start Image Composition   | HTTP Request       | Starts image composition with background | 1. Create Job (Upload Foreground) | 3. Check Job Status              |                                                                                                              |
| 3. Check Job Status          | HTTP Request       | Polls job status                          | 2. Start Image Composition     | Is Complete?                    |                                                                                                              |
| Is Complete?                | If                  | Checks if job is completed                | 3. Check Job Status            | 4. Download Video, Has Failed?   |                                                                                                              |
| Has Failed?                 | If                  | Checks if job has failed                   | 3. Check Job Status            | Build Error Response, Wait 20s   |                                                                                                              |
| Wait 20s                   | Wait                 | Waits 20 seconds before retrying          | Has Failed?                   | 3. Check Job Status              |                                                                                                              |
| 4. Download Video           | HTTP Request       | Downloads processed video file             | Is Complete?                  | 5. Upload to Google Drive        |                                                                                                              |
| 5. Upload to Google Drive   | Google Drive        | Uploads video to Google Drive              | 4. Download Video             | Build Success Response           |                                                                                                              |
| Build Success Response      | Set                 | Builds success response object             | 5. Upload to Google Drive     | From Webhook?                   |                                                                                                              |
| Build Error Response        | Set                 | Builds error response object               | Has Failed?                   | From Webhook?                   |                                                                                                              |
| From Webhook?               | If                  | Determines if source is webhook or manual | Build Success Response, Build Error Response | Respond to Webhook, Manual Test Complete |                                                                                                              |
| Respond to Webhook          | Respond to Webhook  | Sends HTTP response for webhook calls     | From Webhook?                 |                               |                                                                                                              |
| Manual Test Complete        | Set                 | Outputs final result for manual execution  | From Webhook?                 |                               |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node:**  
   - Type: `Webhook`  
   - Path: `compose-video-image`  
   - Method: POST  
   - Response Data: All entries  
   - Response Mode: Response Node  

2. **Create Manual Trigger node:**  
   - Type: `Manual Trigger`  

3. **Create 'Extract Webhook Data' Set node:**  
   - Assign variables:  
     - `foreground_video_url` = `{{$json.body?.foreground_video_url ?? $json.foreground_video_url ?? 'https://videos.videobgremover.com/public-videos/assets/ai-actor.mp4'}}`  
     - `background_image_url` = `{{$json.body?.background_image_url ?? $json.background_image_url ?? 'https://drive.google.com/uc?id=1DYJxUcuN5n4BT4YZnRKYmXuLNSCs5bPy'}}`  
     - `source` = `"webhook"`  

4. **Create 'Sample URLs (Edit Here)' Set node:**  
   - Assign variables:  
     - `foreground_video_url` = `"https://videos.videobgremover.com/public-videos/assets/ai-actor.mp4"`  
     - `background_image_url` = `"https://drive.google.com/uc?id=1DYJxUcuN5n4BT4YZnRKYmXuLNSCs5bPy"`  
     - `source` = `"manual"`  

5. **Create 'Merge Triggers' Merge node:**  
   - Mode: Append  
   - Connect inputs from `Extract Webhook Data` and `Sample URLs (Edit Here)` nodes.  

6. **Create '1. Create Job (Upload Foreground)' HTTP Request node:**  
   - Method: POST  
   - URL: `https://api.videobgremover.com/api/v1/jobs`  
   - Headers:  
     - `X-API-Key`: `{{$vars.VIDEOBGREMOVER_KEY}}` (ensure environment variable is set)  
     - `Content-Type`: `application/json`  
   - Body (JSON): `{ "video_url": "{{ $json.foreground_video_url }}" }`  
   - Response Format: JSON, never error  
   - Connect input from `Merge Triggers`.  

7. **Create '2. Start Image Composition' HTTP Request node:**  
   - Method: POST  
   - URL: `https://api.videobgremover.com/api/v1/jobs/{{ $json.id }}/start`  
   - Headers: Same as above  
   - Body (JSON):  
     ```
     {
       "background": {
         "type": "composition",
         "composition": {
           "template": "centered",
           "background_type": "image",
           "background_url": "{{ $('Merge Triggers').item.json.background_image_url }}",
           "export_format": "h264",
           "export_preset": "medium"
         }
       }
     }
     ```  
   - Response Format: JSON, never error  
   - Connect input from `1. Create Job (Upload Foreground)` node.  

8. **Create '3. Check Job Status' HTTP Request node:**  
   - Method: GET  
   - URL: `https://api.videobgremover.com/api/v1/jobs/{{ $('1. Create Job (Upload Foreground)').item.json.id }}/status`  
   - Headers:  
     - `X-API-Key`: `{{$vars.VIDEOBGREMOVER_KEY}}`  
   - Response Format: JSON, never error  
   - Connect input from `2. Start Image Composition`.  

9. **Create 'Is Complete?' If node:**  
   - Condition: `$json.status === "completed"`  
   - Connect input from `3. Check Job Status`.  

10. **Create 'Has Failed?' If node:**  
    - Condition: `$json.status === "failed"`  
    - Connect input from `3. Check Job Status` (second branch from `Is Complete?` node).  

11. **Create 'Wait 20s' Wait node:**  
    - Amount: 20 seconds  
    - Connect input from `Has Failed?` node's "false" branch (job not failed and not complete).  
    - Connect output back to `3. Check Job Status` to poll again.  

12. **Create 'Build Error Response' Set node:**  
    - Assign variable `final_result` to an object containing:  
      - `success: false`  
      - `job_id` from `1. Create Job (Upload Foreground)`  
      - `error` from `$json.error` or default message  
      - `status` from current JSON  
      - `message` indicating failure  
      - `source` from merged input  
    - Connect input from `Has Failed?` node's "true" branch.  

13. **Create '4. Download Video' HTTP Request node:**  
    - Method: GET  
    - URL: `{{$json.processed_video_url}}`  
    - Response Type: File (binary)  
    - Connect input from `Is Complete?` node's "true" branch.  

14. **Create '5. Upload to Google Drive' Google Drive node:**  
    - Operation: Upload  
    - File Name: `composed_video_{{$('1. Create Job (Upload Foreground)').item.json.id}}_{{ new Date().getTime() }}.mp4`  
    - Folder: Root of "My Drive" or custom folder as preferred  
    - Binary Property: `data` (from download node)  
    - Credentials: Google Drive OAuth2 must be configured  
    - Connect input from `4. Download Video`.  

15. **Create 'Build Success Response' Set node:**  
    - Assign variable `final_result` to an object containing:  
      - `success: true`  
      - Job and export IDs  
      - Google Drive file ID and URL  
      - Download URL  
      - Video length in seconds  
      - Success message  
      - Source  
    - Connect input from `5. Upload to Google Drive`.  

16. **Create 'From Webhook?' If node:**  
    - Condition: Check if `final_result.source === "webhook"` or fallback to `source`.  
    - Connect inputs from `Build Success Response` and `Build Error Response`.  

17. **Create 'Respond to Webhook' node:**  
    - Response Body: `{{$json.final_result ?? $json}}`  
    - Response Code: 200  
    - Connect input from `From Webhook?` node's "true" branch.  

18. **Create 'Manual Test Complete' Set node:**  
    - Pass through `final_result` for manual output.  
    - Connect input from `From Webhook?` node's "false" branch.  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Full documentation and API details available at https://docs.videobgremover.com/                         | Official workflow documentation link                           |
| Pricing for VideoBGRemover API: $0.50-$2.00 per minute processed                                         | Important for cost estimation                                  |
| Setup requires adding API key as n8n environment variable named `VIDEOBGREMOVER_KEY`                      | Security best practice                                         |
| Google Drive OAuth2 connection must be authorized before use                                            | Google Drive node setup                                        |
| Processing duration: 3-5 minutes per minute of video length                                             | Helps manage user expectations                                |
| Composition templates available: `centered`, `fullscreen`, `picture_in_picture`, `ai_ugc_ad`              | Customization details for output video look                   |
| Input URLs must be publicly accessible to allow API processing                                          | Critical input constraint                                     |
| Supports webhook automation and manual testing, compatible with batch processing via Google Sheets/Airtable | Extensibility notes                                           |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.