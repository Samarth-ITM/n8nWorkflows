Upload & Rename Videos to Google Drive via Apps Script from URL

https://n8nworkflows.xyz/workflows/upload---rename-videos-to-google-drive-via-apps-script-from-url-3815


# Upload & Rename Videos to Google Drive via Apps Script from URL

### 1. Workflow Overview

This workflow facilitates uploading large video or audio files to Google Drive by leveraging a custom Google Apps Script as an intermediary. Since n8n’s native HTTP Download nodes may struggle with large files, the workflow sends a URL to the script, which downloads the file directly to a specified Google Drive folder and returns the file metadata (notably the file URL). The workflow then renames the uploaded file in Google Drive via the Google Drive node, enabling streamlined file management.

**Target Use Cases:**  
- Handling large media file uploads to Google Drive from URLs without downloading the file within n8n.  
- Automating file renaming after upload based on custom naming conventions.  
- Securely uploading files via a secret-authenticated Google Apps Script endpoint.  

**Logical Blocks:**  
- **1.1 Input Trigger:** Manual initiation of the workflow.  
- **1.2 Upload via Google Apps Script:** Sending the video URL and secret key to the Apps Script endpoint to upload the file to Google Drive.  
- **1.3 Rename Uploaded File:** Updating the uploaded file’s name in Google Drive using the returned file URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block provides a manual trigger to start the workflow execution. It allows users to test or run the workflow on demand.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)

- **Node Details:**  

| Node Name                   | Type              | Configuration & Role                                                                                   | Inputs                   | Outputs                | Edge Cases / Failure Modes                        |
|-----------------------------|-------------------|-------------------------------------------------------------------------------------------------------|--------------------------|------------------------|--------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger    | Triggers the workflow manually; no parameters needed.                                                 | None (start node)        | Connected to HTTP Request node | None intrinsic; user must initiate workflow.      |

---

#### 1.2 Upload via Google Apps Script

- **Overview:**  
  This block sends a POST request to the deployed Google Apps Script Web App. It passes the video file URL and a secret key for authentication. The script downloads the file directly to a specified Google Drive folder and returns the file’s URL.

- **Nodes Involved:**  
  - `Send URL to GDrive Script and Upload` (HTTP Request)

- **Node Details:**  

| Node Name                            | Type           | Configuration & Role                                                                                                                                                 | Inputs                               | Outputs                             | Edge Cases / Failure Modes                                                                                                                  |
|------------------------------------|----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Send URL to GDrive Script and Upload| HTTP Request   | - Method: POST<br>- URL: Custom Google Apps Script Web App URL<br>- Body Content Type: JSON<br>- JSON Body includes:<br>  - `videoUrl`: direct download URL<br>  - `secret`: matching secret key<br>- Sends JSON body with file URL and secret for authentication.<br>- Expects response with uploaded file URL from script. | Triggered by Manual Trigger node  | Passes response JSON (including drive file URL) to next node | - HTTP errors (timeout, 403 unauthorized if secret incorrect)<br>- Script errors returning error message<br>- Invalid URLs or inaccessible files |

---

#### 1.3 Rename Uploaded File

- **Overview:**  
  Once the file is uploaded and the URL is received, this block uses the Google Drive node to update the file’s name in the specified Google Drive account. It extracts the file ID from the URL returned by the Apps Script and renames the file to a user-defined name.

- **Nodes Involved:**  
  - `Rename Uploaded Video` (Google Drive)

- **Node Details:**  

| Node Name               | Type         | Configuration & Role                                                                                                                                                     | Inputs                                     | Outputs           | Edge Cases / Failure Modes                                                                                       |
|-------------------------|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|-------------------|-----------------------------------------------------------------------------------------------------------------|
| Rename Uploaded Video    | Google Drive | - Operation: Update file metadata<br>- File ID: Extracted dynamically from the `driveUrl` returned by previous node<br>- New file name: `"Music Video 1"` (hardcoded)<br>- Requires Google Drive OAuth2 credentials with write access<br>- Updates the uploaded file’s name | Receives `driveUrl` from HTTP Request node | None (end node)   | - Invalid or missing file ID (parsing errors)<br>- OAuth token expiration or permission errors<br>- API limits/errors |

---

### 3. Summary Table

| Node Name                   | Node Type        | Functional Role                | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                              |
|-----------------------------|------------------|-------------------------------|-----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger   | Start workflow manually        | None                        | Send URL to GDrive Script and Upload |                                                                                                         |
| Send URL to GDrive Script and Upload | HTTP Request  | Uploads file via Google Apps Script | When clicking ‘Test workflow’ | Rename Uploaded Video          | Use your Web App URL and matching secret key to send file URL for server-side upload in Drive.          |
| Rename Uploaded Video        | Google Drive     | Rename uploaded file in Drive  | Send URL to GDrive Script and Upload | None                         | Use the file URL returned from script to update file name. Requires valid OAuth2 credentials.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.  
   - No parameters needed. This node initiates the workflow.

2. **Create HTTP Request Node**  
   - Add an **HTTP Request** node named `Send URL to GDrive Script and Upload`.  
   - Configure:  
     - **HTTP Method:** POST  
     - **URL:** Your deployed Google Apps Script Web App URL (from script deployment step).  
     - **Body Content Type:** JSON  
     - **JSON Body:**  
       ```json
       {
         "videoUrl": "https://example.com/path/to/your.mp4",
         "secret": "your-strong-secret-here"
       }
       ```  
     - Enable **Send Body** as JSON.  
   - Connect the output of the Manual Trigger node to this HTTP Request node.

3. **Create Google Drive Node**  
   - Add a **Google Drive** node named `Rename Uploaded Video`.  
   - Set **Operation** to `Update`.  
   - For **File ID**, use the expression mode and set it to extract from the previous node's JSON:  
     ```
     {{$json["driveUrl"]}}
     ```  
     (Note: Since the script returns a file URL, you may need to parse the file ID from the URL, or if the script returns just the URL, ensure the Drive node accepts URL input in fileId field with `"mode": "url"`.)  
   - Set **New Updated File Name** to your preferred name, e.g., `Music Video 1`.  
   - In **Credentials**, select or create your Google Drive OAuth2 credentials with write access.  
   - Connect the output of the HTTP Request node to this Google Drive node.

4. **Credential Setup**  
   - For Google Drive node, create OAuth2 credentials:  
     - Client ID and Secret from Google Cloud Console  
     - Scopes: `https://www.googleapis.com/auth/drive` or appropriate scope for file update.  
   - For HTTP Request node, no special credentials needed if your Apps Script endpoint is set to allow **Anyone** access with secret key for authentication.

5. **Deploy Google Apps Script** (outside n8n)  
   - Create a new Google Apps Script project at https://script.google.com.  
   - Replace default code with provided script, inserting:  
     - Your secret key (a strong random string).  
     - Your Google Drive target folder ID.  
   - Deploy as Web App:  
     - Execute as: Me  
     - Access: Anyone  
   - Copy generated Web App URL and use it in the HTTP Request node.

6. **Testing**  
   - Click **Execute Workflow** manually.  
   - Monitor nodes for success. Uploaded file should appear in your specified Google Drive folder with the new name set.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The Google Apps Script endpoint securely authenticates requests via a secret key to prevent unauthorized uploads.                  | Script code section and secret key setup instructions.                                            |
| Folder ID extraction: the folder ID is part of the Google Drive folder URL after `/folders/`.                                       | Example: `https://drive.google.com/drive/u/0/folders/1Xabc12345678defGHIJklmn`                     |
| For generating a strong secret key, you can use: https://acte.ltd/utils/randomkeygen (Encryption key 256).                        | Mentioned in setup steps for secret key generation.                                               |
| Google Apps Script deployment must be set to “Anyone” access to allow n8n to POST without user login but secured by the secret key.| Deployment instructions in the script section.                                                    |
| The workflow is designed to handle large files that n8n cannot download directly, avoiding memory/timeouts.                       | Purpose explained in workflow overview.                                                           |

---

This document provides an exhaustive reference to understand, reproduce, and troubleshoot the workflow for uploading and renaming videos on Google Drive via a Google Apps Script intermediary.