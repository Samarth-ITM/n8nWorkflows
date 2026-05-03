Video Transcription Automation with VLM Run, Google Drive and Docs

https://n8nworkflows.xyz/workflows/video-transcription-automation-with-vlm-run--google-drive-and-docs-6703


# Video Transcription Automation with VLM Run, Google Drive and Docs

### 1. Workflow Overview

This workflow automates the transcription of video files uploaded to a specified Google Drive folder using the AI-powered VLM Run service. It is designed to facilitate accurate, timestamped speech recognition and store the results in a structured Google Docs document for easy sharing and collaboration.

**Target Use Cases:**  
- Transcribing meeting recordings  
- Creating accessible content from videos  
- Automating video documentation  
- Content analysis and review  

**Logical Blocks:**

- **1.1 Input Reception and Video Download:** Watches a Google Drive folder for new video uploads and downloads the video file for processing.  
- **1.2 AI Processing with VLM Run:** Sends the video asynchronously to VLM Run for transcription and receives the transcription results via webhook callback.  
- **1.3 Data Storage:** Saves the transcription output, including timestamps and metadata, into a formatted Google Docs document.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Video Download

**Overview:**  
Monitors a specific Google Drive folder for new video file uploads, automatically downloads the detected video files to prepare them for transcription.

**Nodes Involved:**  
- Monitor Video Uploads (Google Drive Trigger)  
- Download Video (Google Drive)  

**Node Details:**  

- **Monitor Video Uploads**  
  - Type: Google Drive Trigger  
  - Role: Watches a designated Google Drive folder for new files created (uploaded).  
  - Configuration: Trigger set to watch "fileCreated" events every minute on a specific folder (URL provided).  
  - Inputs: None (trigger node)  
  - Outputs: File metadata including ID, passed to Download Video node  
  - Edge Cases: Folder access permission errors, API rate limits, delayed triggers if polling interval is too long  
  - Credentials: Google Drive OAuth2 with appropriate scopes for folder monitoring  

- **Download Video**  
  - Type: Google Drive  
  - Role: Downloads the video file from Google Drive using the file ID from the trigger node.  
  - Configuration: Operation set to "download" with dynamic file ID from previous node.  
  - Inputs: File metadata JSON with file ID  
  - Outputs: Binary video file data for AI processing  
  - Edge Cases: File not found, permission denied, network timeouts, large file download delays  
  - Credentials: Same Google Drive OAuth2 credentials as trigger node  

---

#### 1.2 AI Processing with VLM Run

**Overview:**  
Processes the downloaded video asynchronously using VLM Run’s AI transcription service, then awaits and handles the transcription result via a webhook callback.

**Nodes Involved:**  
- VLM Run Video Transcriber (@vlm-run/n8n-nodes-vlmrun.vlmRun)  
- Receive Transcription Results (Webhook)  

**Node Details:**  

- **VLM Run Video Transcriber**  
  - Type: VLM Run custom node for video transcription  
  - Role: Sends the video file to VLM Run for speech-to-text transcription with timestamps and metadata.  
  - Configuration:  
    - Domain: "video.transcription"  
    - Operation: "video"  
    - Callback URL: webhook endpoint URL for asynchronous result receipt  
    - Async Processing: Enabled (to handle longer videos without timeout)  
  - Inputs: Binary video file from Download Video node  
  - Outputs: None directly; result is sent asynchronously via webhook  
  - Edge Cases: API authentication errors, video format incompatibility, network issues, callback URL unreachable, transcription failures  
  - Credentials: VLM Run API key configured  

- **Receive Transcription Results**  
  - Type: Webhook  
  - Role: Receives POST requests containing transcription results from VLM Run asynchronously.  
  - Configuration:  
    - Path: "video-transcription-vlm-run" (matches callback URL in VLM Run node)  
    - HTTP Method: POST  
  - Inputs: HTTP POST data (transcription JSON payload)  
  - Outputs: JSON with transcription data forwarded to storage node  
  - Edge Cases: Webhook endpoint downtime, malformed data, security concerns (validate source), concurrency issues  

---

#### 1.3 Data Storage

**Overview:**  
Formats and saves the transcription results including segments, timestamps, and metadata into a Google Docs document for easy access and collaboration.

**Nodes Involved:**  
- Save Transcription to Docs (Google Docs)  

**Node Details:**  

- **Save Transcription to Docs**  
  - Type: Google Docs  
  - Role: Inserts formatted transcription data into a designated Google Docs document.  
  - Configuration:  
    - Operation: Update existing document by URL  
    - Text content uses expressions to format:  
      - Current date/time  
      - Total video duration from metadata  
      - Iterates through transcription segments to include start/end times, transcript text, video description  
      - Separators for readability  
  - Inputs: Transcription JSON from webhook node  
  - Outputs: Updated Google Docs document  
  - Edge Cases: Document access permissions, rate limits, malformed transcription data, document version conflicts  
  - Credentials: Google Docs OAuth2 with editing rights  

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                       | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                                                            |
|-----------------------------|----------------------------------|-------------------------------------|------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| 🎥 Workflow Overview        | Sticky Note                      | Provides high-level workflow summary| -                      | -                          | Overview of AI video transcription workflow, use cases, and requirements                                                               |
| 📁 Input Documentation1       | Sticky Note                      | Describes video input block          | -                      | -                          | Details about monitoring Google Drive folder and downloading videos                                                                    |
| Monitor Video Uploads        | Google Drive Trigger             | Watches folder for new video files   | -                      | Download Video             | Watches designated Google Drive folder for new video files and triggers transcription processing automatically                          |
| Download Video              | Google Drive                    | Downloads video files for processing | Monitor Video Uploads   | VLM Run Video Transcriber  | Downloads the uploaded video file from Google Drive and prepares it for AI-powered speech transcription                                 |
| 🤖 AI Processing Documentation1 | Sticky Note                      | Explains AI transcription processing | -                      | -                          | Describes VLM Run’s transcription features and output                                                                                   |
| VLM Run Video Transcriber    | VLM Run Node                   | Sends video for asynchronous AI transcription | Download Video          | (async webhook)            | Processes video files using VLM AI to generate accurate speech transcriptions with timestamps and metadata                             |
| 🔗 Async Processing Documentation1 | Sticky Note                      | Explains async transcription workflow | -                      | -                          | Details asynchronous processing flow and benefits                                                                                       |
| Receive Transcription Results | Webhook                        | Receives transcription results       | (async from VLM Run)   | Save Transcription to Docs | Receives the completed video transcription data from VLM AI when asynchronous processing is finished                                    |
| Save Transcription to Docs   | Google Docs                    | Saves transcription output to document| Receive Transcription Results | -                      | Stores transcription data in Google Docs for easy access and sharing                                                                    |
| 📄 Output Documentation      | Sticky Note                      | Describes data storage benefits      | -                      | -                          | Details about storing transcription results in Google Docs and associated benefits                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Monitor Video Uploads" Node**  
   - Type: Google Drive Trigger  
   - Configure event: "fileCreated"  
   - Set polling interval: every minute  
   - Specify folder to watch via URL (Google Drive folder containing videos)  
   - Connect Google Drive OAuth2 credentials with access to watch the folder  

2. **Create "Download Video" Node**  
   - Type: Google Drive  
   - Operation: "download"  
   - Set fileId parameter to: `{{$json["id"]}}` (dynamic from trigger)  
   - Use same Google Drive OAuth2 credentials as trigger  
   - Connect output of "Monitor Video Uploads" to input of this node  

3. **Create "VLM Run Video Transcriber" Node**  
   - Type: VLM Run custom node (install `@vlm-run/n8n-nodes-vlmrun` if needed)  
   - Set Domain: "video.transcription"  
   - Operation: "video"  
   - Enable asynchronous processing (check "processAsynchronously")  
   - Set Callback URL: Use your n8n webhook endpoint URL appended with path `/video-transcription-vlm-run`  
   - Connect VLM Run API credentials  
   - Connect input to output of "Download Video" node  

4. **Create "Receive Transcription Results" Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `video-transcription-vlm-run` (must match callback URL in VLM Run node)  
   - No authentication configured (optional: add security if needed)  
   - This is the async entry point for transcription results  
   - Connect output to next node  

5. **Create "Save Transcription to Docs" Node**  
   - Type: Google Docs  
   - Operation: "update"  
   - Document URL: Set the Google Docs document where transcripts will be saved  
   - Configure Insert Text action with the following template:  
     ```
     🎥 Video Transcription Report

     🗓️ Date: {{ new Date().toLocaleString('en-US', { dateStyle: 'medium', timeStyle: 'short' }) }}
     ⏱️ Total Duration: {{ $json.body.response.metadata.duration }} seconds

     {{ $json.body.response.segments.map((segment, index) => `
     🔹 Segment ${index + 1}
     ⏰ Time: ${segment.start_time.toFixed(2)}s → ${segment.end_time.toFixed(2)}s

     🎤 Audio Transcript:
     "${segment.audio.content.trim()}"

     📹 Video Description:
     "${segment.video.content.trim()}"

     ${'─'.repeat(50)}` ).join('\n') }}
     ```
   - Connect Google Docs OAuth2 credentials with edit rights to the document  
   - Connect input from "Receive Transcription Results" node  

6. **Add Sticky Notes**  
   - Add sticky notes for workflow overview, input processing, AI processing, async flow, and data storage to document the workflow clearly (optional but recommended).  

7. **Test the Workflow**  
   - Upload a supported video file (e.g., MP4) to the monitored Google Drive folder  
   - Confirm the workflow triggers, downloads the file, sends it to VLM Run, receives transcription, and updates Google Docs correctly  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow requires VLM Run API access for video transcription                                        | https://playground.vlm.run                                                                         |
| Google Drive OAuth2 and Google Docs OAuth2 credentials must have appropriate scopes and permissions | Google Drive API & Google Docs API                                                                |
| Async processing avoids timeouts and handles large video files reliably                              | See sticky note "🔗 Async Processing Flow"                                                        |
| Document formatting uses JavaScript expressions for dynamic content injection                        | Google Docs node "Insert Text" action                                                              |
| Supported video formats include MP4, AVI, MOV, WebM, MKV                                            | Mentioned in sticky note "📁 Video Input Processing"                                               |

---

**Disclaimer:** The provided text is extracted solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.