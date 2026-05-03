Automatic PDF Compression with iLovePDF for Google Drive Files

https://n8nworkflows.xyz/workflows/automatic-pdf-compression-with-ilovepdf-for-google-drive-files-10766


# Automatic PDF Compression with iLovePDF for Google Drive Files

### 1. Workflow Overview

This workflow automates the compression of PDF files stored in Google Drive by leveraging the iLovePDF Compress Tool through iLoveAPI. It monitors a specific Google Drive folder for newly uploaded files, compresses these files via iLoveAPI, and stores the compressed versions in a separate Google Drive folder. The original files are then moved to an archive folder to prevent reprocessing.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Reception:** Detects new files added to a monitored Google Drive folder.
- **1.2 Authentication & Task Initialization:** Authenticates with iLoveAPI and starts a compression task.
- **1.3 File Handling:** Downloads the original file from Google Drive and uploads it to iLoveAPI.
- **1.4 Compression Processing:** Sends the compression request and downloads the compressed file.
- **1.5 Upload & Archival:** Uploads the compressed file back to Google Drive and moves the original file to an archive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

**Overview:**  
Detects when a new file is created in a specific Google Drive folder and initiates the workflow.

**Nodes Involved:**

- Upload your file to Google Drive (Google Drive Trigger)
- When clicking ‘Test workflow’ (Manual Trigger)

**Node Details:**

- **Upload your file to Google Drive**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder (`folderToWatch` with ID `1vtY_ATTQOFggVQ_EnzlrpMcY6XrVpeD0`) for new files created. It polls every minute.  
  - Connections: Output triggers the workflow chain (connects to "Download file" and "Send your iLoveAPI public key to their server")  
  - Edge Cases: Folder permission issues, delayed triggers due to polling interval, or no new files.  
  - Version: 1  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing purposes.  
  - Connections: Starts the authentication process directly.  
  - Edge Cases: Manual trigger bypasses folder monitoring; must ensure file context is correctly set for testing.  
  - Version: 1  

#### 2.2 Authentication & Task Initialization

**Overview:**  
Sends the iLoveAPI public key to retrieve a Bearer token, then starts a compression task on the iLoveAPI server.

**Nodes Involved:**

- Send your iLoveAPI public key to their server (HTTP Request)  
- Get task from iLoveAPI server (HTTP Request)  

**Node Details:**

- **Send your iLoveAPI public key to their server**  
  - Type: HTTP Request  
  - Role: Authenticates with iLoveAPI by sending the public key to obtain a Bearer token.  
  - Configuration: POST to `https://api.ilovepdf.com/v1/auth` with body parameter `public_key` set to `"YOUR_ILOVEAPI_PUBLIC_KEY"`.  
  - Key Expressions: Public key is hardcoded; output contains token used in headers downstream.  
  - Connections: Output token feeds into "Get task from iLoveAPI server" node.  
  - Failure Modes: Invalid or expired public key, network errors, API downtime.  
  - Version: 4.2  

- **Get task from iLoveAPI server**  
  - Type: HTTP Request  
  - Role: Starts the compression task on iLoveAPI, returning task ID and server URL.  
  - Configuration: POST to `https://api.ilovepdf.com/v1/start/compress` with body parameters including `public_key` and `tool` set to `"compress"`. Authorization header uses token from previous node.  
  - Key Expressions: Authorization header dynamically set as `Bearer {{ $json.token }}` from prior node's token.  
  - Connections: Output contains task info and server details, flows into data grouping node.  
  - Failure Modes: Token expiration, invalid task parameters, API errors.  
  - Version: 4.2  

#### 2.3 File Handling

**Overview:**  
Downloads the original Google Drive file, groups authentication and task data, then uploads the file to the iLoveAPI server.

**Nodes Involved:**

- Download file (Google Drive)  
- Group public key, task and downloaded file data (Merge)  
- Upload PDF to iLoveAPI server (HTTP Request)  

**Node Details:**

- **Download file**  
  - Type: Google Drive  
  - Role: Downloads the file from Google Drive using the file ID detected by the trigger.  
  - Configuration: Operation set to "download"; fileId dynamically set as `{{$json.id}}` from the trigger.  
  - Connections: Outputs binary file data, connects to merge node.  
  - Failure Modes: File not found, permission denied, network issues.  
  - Version: 3  

- **Group public key, task and downloaded file data**  
  - Type: Merge  
  - Role: Combines data streams from task initialization and downloaded file for unified processing.  
  - Configuration: Mode "combine" by position; includes unpaired data.  
  - Inputs: Receives task data and downloaded file binary data.  
  - Connections: Output flows into "Upload PDF to iLoveAPI server."  
  - Failure Modes: Mismatched inputs causing incomplete data.  
  - Version: 3  

- **Upload PDF to iLoveAPI server**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded PDF file to the iLoveAPI server to prepare for compression.  
  - Configuration: POST to dynamic URL `https://{{ $json.server }}/v1/upload`, multipart form-data with task ID and file binary data. Authorization uses Bearer token from authentication node.  
  - Key Expressions: Uses merged data for server URL, task ID, and file binary.  
  - Failure Modes: Upload failure, file size limits, network errors.  
  - Version: 4.2  

#### 2.4 Compression Processing

**Overview:**  
Sends the compression command to iLoveAPI and downloads the resulting compressed file.

**Nodes Involved:**

- Compress your file (HTTP Request)  
- Download compressed file (HTTP Request)  

**Node Details:**

- **Compress your file**  
  - Type: HTTP Request  
  - Role: Sends a compression request with task ID, tool name, and file references to iLoveAPI's `/v1/process` endpoint.  
  - Configuration: POST with JSON body including task, compress tool, and file info (server file name and original file name). Authorization header uses Bearer token.  
  - Key Expressions: Body uses dynamic expressions referencing task and file names from previous nodes.  
  - Connections: Output triggers downloading of compressed file.  
  - Failure Modes: Invalid task ID, malformed request, API errors, token expiration.  
  - Version: 4.2+ (supports JSON body and advanced headers)  

- **Download compressed file**  
  - Type: HTTP Request  
  - Role: Downloads the compressed PDF file using the task ID from iLoveAPI.  
  - Configuration: GET request to dynamic URL `https://{{ server }}/v1/download/{{ task }}`, response format set to file with output property name set dynamically from the filename. Authorization uses Bearer token.  
  - Key Expressions: Authorization token and URLs are dynamically set.  
  - Connections: Output triggers upload to Google Drive destination folder.  
  - Failure Modes: Download failure, file corruption, token expiration.  
  - Version: 4.3  

#### 2.5 Upload & Archival

**Overview:**  
Uploads the compressed PDF back to Google Drive destination folder and moves the original file to an archive folder.

**Nodes Involved:**

- Move compressed file to new Google Drive folder (Google Drive)  

**Node Details:**

- **Move compressed file to new Google Drive folder**  
  - Type: Google Drive  
  - Role: Moves the original file (not the compressed one) to an archive folder to prevent reprocessing.  
  - Configuration: Operation "move" with `fileId` from the original upload node and destination folder ID `1-_vUMwMdKcEo9uLXX9UBCdlS_wHUPzii`.  
  - Connections: Final step in workflow.  
  - Failure Modes: Permission issues, file lock conflicts, missing destination folder.  
  - Version: 3  

---

### 3. Summary Table

| Node Name                               | Node Type               | Functional Role                          | Input Node(s)                              | Output Node(s)                              | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|----------------------------------------|-------------------------|----------------------------------------|--------------------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’           | Manual Trigger          | Manual start trigger for testing       | -                                          | Send your iLoveAPI public key to their server |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Upload your file to Google Drive        | Google Drive Trigger    | Watches source folder for new files    | -                                          | Download file, Send your iLoveAPI public key to their server | ## Sets up iLovePDF credentials and gets file from folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Download file                          | Google Drive            | Downloads original file from Drive     | Upload your file to Google Drive            | Group public key, task and downloaded file data |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Send your iLoveAPI public key to their server | HTTP Request            | Authenticates with iLoveAPI            | When clicking ‘Test workflow’, Upload your file to Google Drive | Get task from iLoveAPI server                 | ## Sets up iLovePDF credentials and gets file from folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Get task from iLoveAPI server          | HTTP Request            | Starts compression task on iLoveAPI    | Send your iLoveAPI public key to their server | Group public key, task and downloaded file data | ## Sets up iLovePDF credentials and gets file from folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Group public key, task and downloaded file data | Merge                   | Combines authentication, task, and file data | Get task from iLoveAPI server, Download file | Upload PDF to iLoveAPI server                  | ## Unifies data from previous step and does the tool processes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Upload PDF to iLoveAPI server           | HTTP Request            | Uploads file to iLoveAPI server        | Group public key, task and downloaded file data | Compress your file                            | ## Unifies data from previous step and does the tool processes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Compress your file                     | HTTP Request            | Sends compression command to iLoveAPI | Upload PDF to iLoveAPI server               | Download compressed file                       | ## Unifies data from previous step and does the tool processes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Download compressed file               | HTTP Request            | Downloads compressed file from iLoveAPI | Compress your file                           | Move compressed file to new Google Drive folder | ## Upload processed file to drive folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Move compressed file to new Google Drive folder | Google Drive            | Moves original file to archive folder  | Download compressed file                      | -                                             | ## Upload processed file to drive folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Sticky Note9                          | Sticky Note             | Documentation and workflow explanation | -                                          | -                                             | ## Try this iLovePDF Compress Automation! This n8n workflow automates compressing files in Google Drive using iLovePDF. When a file is added to a monitored folder, the workflow compresses it and saves it to another folder, optimizing storage. The process includes authenticating with iLoveAPI, initiating the compression task, downloading the compressed file, and moving the original to an archive folder to prevent reprocessing. This setup is ideal for managing large PDFs and improving storage efficiency. [...] |
| Sticky Note8                          | Sticky Note             | Adaptation advice                      | -                                          | -                                             | ### Adapt it as you need! This is just an example of using it for you to know how the flow should start to work without issues. After the "combine" step, you can change it according your needs but **always maintaining the four main steps of ILoveAPI's request workflow: start, upload, process and download** (e.g., an step for sending an email with the compressed file instead of moving it to another folder)                                                                                                                                              |
| Sticky Note                          | Sticky Note             | Block label for initial setup          | -                                          | -                                             | ## Sets up iLovePDF credentials and gets file from folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note1                         | Sticky Note             | Block label for process unification    | -                                          | -                                             | ## Unifies data from previous step and does the tool processes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Sticky Note2                         | Sticky Note             | Block label for upload and archival    | -                                          | -                                             | ## Upload processed file to drive folder                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node ("Upload your file to Google Drive")**  
   - Type: Google Drive Trigger  
   - Operation: Watch for "fileCreated" event  
   - Poll Interval: Every minute  
   - Folder to Watch: Set folder ID to the source folder (e.g., `1vtY_ATTQOFggVQ_EnzlrpMcY6XrVpeD0`)  
   - Credential: Configure Google Drive OAuth2 credentials with access to this folder  

2. **Create Manual Trigger Node ("When clicking ‘Test workflow’")**  
   - Type: Manual Trigger  
   - Purpose: Enable manual execution for testing  

3. **Create HTTP Request Node ("Send your iLoveAPI public key to their server")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.ilovepdf.com/v1/auth`  
   - Body Parameters: Add parameter `public_key` with your actual iLoveAPI public key  
   - Send Body: Yes  
   - Credential: None (public key is sent directly)  
   - Connect outputs of Manual Trigger and Google Drive Trigger nodes to this node  

4. **Create HTTP Request Node ("Get task from iLoveAPI server")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.ilovepdf.com/v1/start/compress`  
   - Body Parameters:  
     - `public_key`: your iLoveAPI public key  
     - `tool`: "compress"  
   - Headers: Add Authorization header with value `Bearer {{$json.token}}` (token from previous node)  
   - Connect output of "Send your iLoveAPI public key to their server" to this node  

5. **Create Google Drive Node ("Download file")**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: `{{$json.id}}` (from trigger)  
   - Credential: Same Google Drive credentials  
   - Connect output of Google Drive Trigger node to this node  

6. **Create Merge Node ("Group public key, task and downloaded file data")**  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Include Unpaired: True  
   - Connect "Get task from iLoveAPI server" and "Download file" outputs to this node  

7. **Create HTTP Request Node ("Upload PDF to iLoveAPI server")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://{{$json.server}}/v1/upload` (dynamic from merged data)  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `task`: `{{$json.task}}`  
     - `file`: binary data field named "data" from the downloaded file  
   - Headers: Authorization: Bearer token from "Send your iLoveAPI public key to their server" node  
   - Connect output of Merge node to this node  

8. **Create HTTP Request Node ("Compress your file")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://{{$json.server}}/v1/process`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "task": "{{$json.task}}",
       "tool": "compress",
       "files": [
         {
           "server_filename": "{{$json.server_filename}}",
           "filename": "{{$node['Download file'].binary.data.fileName}}"
         }
       ]
     }
     ```  
   - Headers: Authorization with Bearer token as before  
   - Connect output of "Upload PDF to iLoveAPI server" to this node  

9. **Create HTTP Request Node ("Download compressed file")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://{{$json.server}}/v1/download/{{$json.task}}`  
   - Response Format: File  
   - Output Property Name: Use dynamic filename from JSON response e.g., `{{$json.download_filename}}`  
   - Headers: Authorization with Bearer token as before  
   - Connect output of "Compress your file" to this node  

10. **Create Google Drive Node ("Move compressed file to new Google Drive folder")**  
    - Type: Google Drive  
    - Operation: Move  
    - File ID: ID of the original file from "Upload your file to Google Drive" node output (`{{$json.id}}`)  
    - Destination Folder ID: Set to your archive folder ID (e.g., `1-_vUMwMdKcEo9uLXX9UBCdlS_wHUPzii`)  
    - Credential: Same Google Drive credentials  
    - Connect output of "Download compressed file" to this node  

11. **Credential Setup:**  
    - Google Drive OAuth2 Credential with appropriate scopes to read/write/move files.  
    - No specific credential needed for HTTP nodes, but iLoveAPI public key must be set correctly in nodes’ parameters.  

12. **Optional:**  
    - Add Sticky Notes to document each block for clarity, including usage instructions and adaptation tips.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates compressing files in Google Drive using iLovePDF. When a file is added to a monitored folder, the workflow compresses it and saves it to another folder, optimizing storage. The process includes authenticating with iLoveAPI, initiating the compression task, downloading the compressed file, and moving the original to an archive folder to prevent reprocessing. Ideal for managing large PDFs and improving storage efficiency. | See Sticky Note9 content inside the workflow for detailed overview and setup instructions.                                                                                                                                         |
| Adapt the workflow as needed, especially after the data merge step. Always maintain the four main iLoveAPI request workflow steps: start, upload, process, and download. You can replace final steps (e.g., move file) with email sending or other integrations as required.                                                                                                                                                | Sticky Note8 content inside workflow; encourages customization while preserving core API interaction pattern.                                                                                                                    |
| Requires Google Drive account with appropriate API credentials and an iLoveAPI developer account with API key. Ensure the correct folder IDs are configured for source, destination, and archive folders.                                                                                                                                                                                                                              | Credential setup and folder ID configuration are critical for the workflow to function correctly.                                                                                                                                  |
| iLoveAPI documentation: https://developer.ilovepdf.com/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Official API docs for advanced customization and troubleshooting.                                                                                                                                                                  |
| n8n Google Drive node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Useful for understanding Google Drive node operations like download, move, and trigger.                                                                                                                                           |

---

*Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*