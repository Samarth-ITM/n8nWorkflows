Generate AI video captions with ZapCap and Google Drive

https://n8nworkflows.xyz/workflows/generate-ai-video-captions-with-zapcap-and-google-drive-8938


# Generate AI video captions with ZapCap and Google Drive

### 1. Workflow Overview

This workflow automates the process of generating AI-powered captions for videos uploaded to a specific Google Drive folder by leveraging the ZapCap API. It continuously monitors a designated Google Drive folder for new video files, uploads them to ZapCap for subtitle generation, tracks the processing status, downloads the captioned video upon completion, and uploads the finalized video back to a specified Google Drive folder.

**Target Use Cases:**  
- Content creators who want to automate subtitle generation for videos  
- Marketing teams needing quick turnarounds on captioned videos  
- Anyone using Google Drive as a central video repository and wanting seamless AI captioning  

**Logical Blocks:**  
- **1.1 Input Reception via Google Drive Trigger:** Watches a Google Drive folder for new video files.  
- **1.2 Video Upload to ZapCap:** Sends the new video URL to ZapCap for processing.  
- **1.3 Processing Trigger and Status Checking:** Initiates captioning tasks and polls for completion or failure.  
- **1.4 Download and Upload of Captioned Video:** Downloads the finished video and uploads it into a target Google Drive folder.  
- **1.5 Wait Loop:** If processing is not complete, waits before re-checking status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception via Google Drive Trigger
- **Overview:**  
  This block continuously monitors a specific Google Drive folder for newly created video files to trigger the workflow.

- **Nodes Involved:**  
  - Google Drive Trigger

- **Node Details:**  
  - **Google Drive Trigger**  
    - Type: Trigger node for Google Drive events  
    - Configured to watch a specific folder (by folder ID) in Google Drive for the event "fileCreated"  
    - Polls every minute to detect new files  
    - Input: None (event trigger)  
    - Output: Metadata of the newly created file, including `webViewLink` for accessing the file URL  
    - Version: 1  
    - Edge Cases:  
      - Folder ID misconfiguration leads to no triggers  
      - Permissions errors if OAuth credentials lack read access  
      - Delay in polling interval might cause latency in reaction time  
      - Large files might trigger multiple events if partial uploads occur  
    - Sticky Note: "Upload a video to your selected Google Drive to start this workflow"

#### 1.2 Video Upload to ZapCap
- **Overview:**  
  Uploads the detected video URL from Google Drive to the ZapCap API to initiate caption generation.

- **Nodes Involved:**  
  - Upload video to ZapCap

- **Node Details:**  
  - **Upload video to ZapCap**  
    - Type: HTTP Request node  
    - Sends a POST request to `https://api.zapcap.ai/videos/url`  
    - Body contains the video URL extracted dynamically from the Google Drive Trigger node: `{{$json.webViewLink}}`  
    - Uses HTTP Header Authentication with API key under header name `x-api-key` (configured in credentials)  
    - Input: Video metadata from Google Drive trigger  
    - Output: JSON response including a unique video ID for ZapCap processing  
    - Version: 4.2  
    - Edge Cases:  
      - Invalid or expired API key leads to authentication failures  
      - If Google Drive sharing permissions are not set to "Anyone with the link - Viewer," ZapCap cannot access the video URL  
      - Network timeouts or API downtime cause failures  
    - Sticky Note: "Make sure the folder has given 'Viewer' access to 'Anyone with the link'!"

#### 1.3 Processing Trigger and Status Checking
- **Overview:**  
  This block initiates the captioning task on ZapCap and continuously polls the task status until completion or failure.

- **Nodes Involved:**  
  - Trigger video processing  
  - Get processing task  
  - If processing completed  
  - Wait

- **Node Details:**  
  - **Trigger video processing**  
    - Type: HTTP Request node  
    - Sends POST to `https://api.zapcap.ai/videos/{{videoId}}/task` to start caption generation  
    - Dynamic URL extracts `videoId` from previous node response  
    - Sends body parameters including `templateId` (preset ID for captioning template) and `autoApprove` set to true  
    - Query parameter `ttl=1d` sets task time-to-live  
    - Input: Video ID from Upload video to ZapCap  
    - Output: JSON including `taskId`  
    - Version: 4.2  
    - Edge Cases: API rate limits, invalid template ID, or API errors  
    - Sticky Note: "ZapCap begins processing your video. This can take between 30 seconds to 2 minutes depending on the length of your video"

  - **Get processing task**  
    - Type: HTTP Request node  
    - GET request to `https://api.zapcap.ai/videos/{{videoId}}/task/{{taskId}}` to fetch current task status  
    - Inputs dynamically taken from previous nodes  
    - Output: Task status JSON with fields like `status` and `downloadUrl` (if completed)  
    - Version: 4.2  
    - Edge Cases: Task ID invalid or missing, API downtime

  - **If processing completed**  
    - Type: If node  
    - Checks if the task status is `"completed"` or `"failed"`  
    - If true: Proceeds to download the captioned video  
    - If false: Loops back to wait node for rechecking later  
    - Version: 2.2  
    - Edge Cases: Status field missing or unexpected values

  - **Wait**  
    - Type: Wait node  
    - Pauses execution before the next status check to avoid rapid polling  
    - Configured with a webhook ID (suggesting it can be externally triggered or resumed)  
    - Version: 1.1  
    - Edge Cases: Wait duration too short causing excessive API calls, too long causing latency

#### 1.4 Download and Upload of Captioned Video
- **Overview:**  
  On task completion, downloads the captioned video from ZapCap and uploads it back to a designated Google Drive folder.

- **Nodes Involved:**  
  - Download completed video  
  - Upload file

- **Node Details:**  
  - **Download completed video**  
    - Type: HTTP Request node  
    - GET request to the download URL provided by ZapCap in the task status response  
    - Input: `downloadUrl` dynamically extracted from task status  
    - Output: Binary video content or file data  
    - Version: 4.2  
    - Edge Cases: Download URL expired, network errors

  - **Upload file**  
    - Type: Google Drive node  
    - Uploads the downloaded video file to a specified Google Drive folder (different from the input folder to avoid overwriting)  
    - Uses OAuth credentials for Google Drive  
    - Dynamic file name taken from original uploaded file name  
    - Version: 3  
    - Edge Cases: Permission errors, quota limits, filename conflicts  
    - Sticky Note: "Captioned video is uploaded to your selected Google Drive folder. Make sure to use different source and destination folders for your videos to prevent possible clashes!"

---

### 3. Summary Table

| Node Name              | Node Type                | Functional Role                    | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                           |
|------------------------|--------------------------|----------------------------------|--------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger    | Google Drive Trigger     | Watches folder for new videos    | None                           | Upload video to ZapCap             | Upload a video to your selected Google Drive to start this workflow                                                   |
| Upload video to ZapCap  | HTTP Request             | Sends video URL to ZapCap API    | Google Drive Trigger           | Trigger video processing           | Make sure the folder has given "Viewer" access to "Anyone with the link"!                                            |
| Trigger video processing| HTTP Request             | Initiates caption task           | Upload video to ZapCap         | Get processing task                | ZapCap begins processing your video. This can take between 30 seconds to 2 minutes depending on the length of your video |
| Get processing task     | HTTP Request             | Checks captioning status         | Trigger video processing, Wait | If processing completed            |                                                                                                                       |
| If processing completed | If                       | Decides next step on status      | Get processing task            | Download completed video, Wait    |                                                                                                                       |
| Wait                   | Wait                     | Pauses before re-checking status | If processing completed        | Get processing task               |                                                                                                                       |
| Download completed video| HTTP Request             | Downloads finished video         | If processing completed (true) | Upload file                      |                                                                                                                       |
| Upload file            | Google Drive             | Uploads captioned video to Drive | Download completed video        | None                            | Captioned video is uploaded to your selected Google Drive folder. Make sure to use different source and destination folders for your videos to prevent possible clashes! |
| Sticky Note            | Sticky Note              | Documentation / Comments         | None                           | None                             | Multiple notes with detailed workflow description and instructions                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**  
   - Type: Google Drive Trigger  
   - Configure OAuth credentials with Google Drive access  
   - Set event to "fileCreated"  
   - Set folder to watch by specifying the target Google Drive folder ID  
   - Set polling interval to every 1 minute  

2. **Create HTTP Request node "Upload video to ZapCap":**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.zapcap.ai/videos/url`  
   - Authentication: HTTP Header Auth with header name `x-api-key` and your ZapCap API key  
   - Body Parameters:  
     - Name: `url`  
     - Value: Expression referencing `{{$json.webViewLink}}` from Google Drive Trigger node  
   - Connect input from Google Drive Trigger output  

3. **Create HTTP Request node "Trigger video processing":**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Expression: `https://api.zapcap.ai/videos/{{$json.id}}/task`  
   - Authentication: HTTP Header Auth with `x-api-key`  
   - Body Parameters:  
     - `templateId`: set to a predefined template ID (e.g., `ca050348-e2d0-49a7-9c75-7a5e8335c67d`)  
     - `autoApprove`: set to boolean true  
   - Query Parameters:  
     - `ttl`: "1d"  
   - Connect input from "Upload video to ZapCap" output  

4. **Create HTTP Request node "Get processing task":**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Expression: `https://api.zapcap.ai/videos/{{$('Upload video to ZapCap').item.json.id}}/task/{{$('Trigger video processing').item.json.taskId}}`  
   - Authentication: HTTP Header Auth with `x-api-key`  
   - Connect input from "Trigger video processing" output and from "Wait" node (see below)  

5. **Create If node "If processing completed":**  
   - Type: If  
   - Condition: Check if `status` field in JSON equals `"completed"` OR `"failed"`  
   - Connect input from "Get processing task" output  
   - True branch: proceed to download video  
   - False branch: proceed to wait  

6. **Create Wait node:**  
   - Type: Wait  
   - Configure wait time or use webhook ID as needed for pacing polling frequency  
   - Connect input from "If processing completed" (false output)  
   - Output connected back to "Get processing task" (creating a loop)  

7. **Create HTTP Request node "Download completed video":**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Expression: `{{$json.downloadUrl}}` (from the completed task status)  
   - Connect input from "If processing completed" (true output)  

8. **Create Google Drive node "Upload file":**  
   - Type: Google Drive  
   - Configure OAuth credentials for Google Drive  
   - Set destination folder ID (different from the input folder to avoid overwriting)  
   - File name: Expression from original file name (`{{$('Google Drive Trigger').item.json.name}}`)  
   - Set binary property to upload file content from "Download completed video" node  
   - Connect input from "Download completed video" output  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Stop wasting hours on video captioning. Upload your videos to a Google Drive folder, and ZapCap automatically generates professional subtitles for you.                                 | Workflow description in Sticky Note                                                            |
| Remember to get your API key from ZapCap dashboard. The header auth must use the name `x-api-key`.                                                                                       | [ZapCap API Key](https://platform.zapcap.ai/dashboard/api-key)                                 |
| Make sure Google Drive videos have "Viewer" access to "Anyone with the link" so ZapCap can access the video URL.                                                                         | Sticky note on "Upload video to ZapCap" node                                                   |
| Captioned video is uploaded to your Google Drive folder. Use different folders for input and output to prevent filename clashes or overwrites.                                          | Sticky note on "Upload file" node                                                              |
| ZapCap Discord support and contact email for help: [ZapCap Discord](https://discord.gg/26fYtvjWBx), email: hi@zapcap.ai                                                                 | Included in the main workflow sticky note                                                      |

---

This structured documentation enables users and automation agents to understand, reproduce, and extend the workflow reliably.