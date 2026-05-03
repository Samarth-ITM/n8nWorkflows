Add subtitles to YouTube videos and save them to Google Drive with VideoDB

https://n8nworkflows.xyz/workflows/add-subtitles-to-youtube-videos-and-save-them-to-google-drive-with-videodb-13632


# Add subtitles to YouTube videos and save them to Google Drive with VideoDB

This document provides a technical breakdown of the n8n workflow designed to automate the transcription, subtitling, and storage of YouTube videos using VideoDB and Google Drive.

### 1. Workflow Overview
The workflow automates the process of adding "burned-in" subtitles to YouTube videos. It starts with a user-submitted URL and handles the asynchronous processing lifecycle—uploading, indexing speech, generating subtitles, and rendering a final file—before saving the result to a cloud storage provider.

**Logical Blocks:**
*   **1.1 Input Reception & Upload:** Captures the YouTube URL and initiates the upload to VideoDB.
*   **1.2 Upload Monitoring:** A polling loop that waits for VideoDB to finish importing the video.
*   **1.3 AI Indexing & Subtitling:** Triggers the speech-to-text engine and applies the subtitle overlay.
*   **1.4 Rendering & Download:** Initiates a download job for the processed video and monitors its completion.
*   **1.5 Storage:** Downloads the final binary file and uploads it to a specific Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Video Upload to VideoDB
This block handles the entry point and the initial communication with the VideoDB API.
*   **Nodes Involved:** `YouTube URL Form Trigger`, `Upload Video to VideoDB`.
*   **Node Details:**
    *   **YouTube URL Form Trigger:** (Form Trigger) Presents a public web form with one required field: `Youtube Video URL`.
    *   **Upload Video to VideoDB:** (VideoDB Node) Uses the `upload` operation. It takes the URL from the form and sends it to VideoDB. 
    *   **Edge Cases:** Invalid YouTube URLs or regional restrictions on the video will cause the VideoDB upload to fail.

#### 2.2 Upload Monitoring Loop
Since video processing is asynchronous, this block polls the status until the upload is "complete".
*   **Nodes Involved:** `Wait Before Upload Status Check`, `Poll Upload Status`, `Is Upload Complete`.
*   **Node Details:**
    *   **Wait Node:** Pauses execution (default settings) to prevent aggressive API hitting.
    *   **Poll Upload Status:** (HTTP Request) Calls the `output_url` provided by the initial upload node using VideoDB credentials.
    *   **Is Upload Complete:** (If Node) Checks if `status` equals `complete`. If false, it loops back to the Wait node.

#### 2.3 Indexing and Adding Subtitles
Once the video is available, the spoken content must be transcribed to create the subtitle track.
*   **Nodes Involved:** `Index Spoken Words`, `Wait Before Indexing Status Check`, `Poll Indexing Status`, `Is Indexing Complete`, `Add Subtitle to Video`.
*   **Node Details:**
    *   **Index Spoken Words:** (VideoDB Node) Triggers the `indexSpokenWords` operation. Language is hardcoded to `en_us`.
    *   **Poll/Wait/If Nodes:** A second polling loop that waits for the `status` to reach `done`.
    *   **Add Subtitle to Video:** (VideoDB Node) Applies the generated index as a visual subtitle overlay on the video stream.

#### 2.4 Generating the Download URL
The processed video with subtitles needs to be rendered into a downloadable file.
*   **Nodes Involved:** `Generate Download Job`, `Wait Before Download Job Check`, `Poll Download Job Status`, `Is Download Job Ready`.
*   **Node Details:**
    *   **Generate Download Job:** (VideoDB Node) Uses the `download` operation. It requires the `stream_link` generated in previous steps.
    *   **Poll/Wait/If Nodes:** A third polling loop specifically for the rendering job. It monitors the `data.status` for the value `done`.

#### 2.5 Uploading to Google Drive
The final step moves the processed file from the cloud buffer to the user's permanent storage.
*   **Nodes Involved:** `Download Subtitled Video File`, `Upload Subtitled Video to Google Drive`.
*   **Node Details:**
    *   **Download Subtitled Video File:** (HTTP Request) Performs a GET request on the `download_url`. It outputs a binary file.
    *   **Upload Subtitled Video to Google Drive:** (Google Drive Node) Saves the binary data. Configured to a specific folder ID (`1YW39wRNL4sHlPLQmM2rnKM5mnCmVOT4W`).
    *   **Configuration:** Requires OAuth2 credentials and a valid `folderId`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| YouTube URL Form Trigger | Form Trigger | Entry Point | - | Upload Video to VideoDB | Video Upload to VideoDB |
| Upload Video to VideoDB | VideoDB | API Upload | YouTube URL Form Trigger | Wait Before Upload Status Check | Video Upload to VideoDB |
| Wait Before Upload Status Check | Wait | Delay | Upload Video to VideoDB, Is Upload Complete | Poll Upload Status | Video Upload to VideoDB |
| Poll Upload Status | HTTP Request | Status Polling | Wait Before Upload Status Check | Is Upload Complete | Video Upload to VideoDB |
| Is Upload Complete | If | Logic Gate | Poll Upload Status | Index Spoken Words, Wait Before Upload Status Check | Video Upload to VideoDB |
| Index Spoken Words | VideoDB | Transcription | Is Upload Complete | Wait Before Indexing Status Check | Indexing the spoken content and Adding Subtitles |
| Wait Before Indexing Status Check | Wait | Delay | Index Spoken Words, Is Indexing Complete | Poll Indexing Status | Indexing the spoken content and Adding Subtitles |
| Poll Indexing Status | HTTP Request | Status Polling | Wait Before Indexing Status Check | Is Indexing Complete | Indexing the spoken content and Adding Subtitles |
| Is Indexing Complete | If | Logic Gate | Poll Indexing Status | Add Subtitle to Video, Wait Before Indexing Status Check | Indexing the spoken content and Adding Subtitles |
| Add Subtitle to Video | VideoDB | Add Subtitles | Is Indexing Complete | Generate Download Job | Indexing the spoken content and Adding Subtitles |
| Generate Download Job | VideoDB | Start Rendering | Add Subtitle to Video | Wait Before Download Job Check | Generating the Download URL |
| Wait Before Download Job Check | Wait | Delay | Generate Download Job, Is Download Job Ready | Poll Download Job Status | Generating the Download URL |
| Poll Download Job Status | HTTP Request | Status Polling | Wait Before Download Job Check | Is Download Job Ready | Generating the Download URL |
| Is Download Job Ready | If | Logic Gate | Poll Download Job Status | Download Subtitled Video File, Wait Before Download Job Check | Generating the Download URL |
| Download Subtitled Video File | HTTP Request | Binary Download | Is Download Job Ready | Upload Subtitled Video to Google Drive | Uploading to GDrive |
| Upload Subtitled Video to Google Drive | Google Drive | Storage | Download Subtitled Video File | - | Uploading to GDrive |

---

### 4. Reproducing the Workflow from Scratch

1.  **Form Trigger:** Create a Form Trigger. Add a field "Youtube Video URL" (Required).
2.  **VideoDB Upload:** Add a VideoDB node. Set operation to `Upload`. Map the URL from the Form.
3.  **Polling Loop (Upload):**
    *   Add a Wait node (set to 5-10 seconds).
    *   Add an HTTP Request node using VideoDB credentials. URL: `{{ $node["Upload Video to VideoDB"].json["data"]["output_url"] }}`.
    *   Add an If node checking if `status` equals `complete`. Connect "False" back to the Wait node.
4.  **Indexing:** Add a VideoDB node. Operation: `Index Spoken Words`. Use the Video ID and Collection ID from the previous successful poll. Set language to `en_us`.
5.  **Polling Loop (Indexing):** Repeat the structure of Step 3, but point the HTTP Request to the `output_url` from the Indexing node. Check for `status` = `done`.
6.  **Subtitling:** Add a VideoDB node. Operation: `Add Subtitle`. Use the same Video ID and Collection ID.
7.  **Download Job:** Add a VideoDB node. Operation: `Download`. Pass the `stream_link` from the subtitling step.
8.  **Polling Loop (Download):** Repeat Step 3 structure again. Point HTTP Request to the download job's `output_url`. Check for `data.status` = `done`.
9.  **File Retrieval:** Add an HTTP Request node. Method: `GET`. URL: `{{ $json["data"]["download_url"] }}`. Ensure "Response Format" is set to **File**.
10. **Google Drive:** Add a Google Drive node. Operation: `Upload`. Select your Folder ID and set the file name using the metadata from VideoDB.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Requirements** | Must have VideoDB API Key and Google Drive OAuth2 configured. |
| **Processing Time** | High-resolution or long videos will cause longer wait times in polling loops. |
| **Target Folder** | Current Folder ID: `1YW39wRNL4sHlPLQmM2rnKM5mnCmVOT4W`. This should be updated for different environments. |
| **VideoDB Functionality** | Detailed documentation for VideoDB nodes can be found at [VideoDB Docs](https://docs.videodb.io). |