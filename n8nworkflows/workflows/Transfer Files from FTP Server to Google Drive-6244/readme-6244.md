Transfer Files from FTP Server to Google Drive

https://n8nworkflows.xyz/workflows/transfer-files-from-ftp-server-to-google-drive-6244


# Transfer Files from FTP Server to Google Drive

### 1. Workflow Overview

This workflow automates the transfer of files from an FTP server to a Google Drive folder. It is designed for scenarios where files stored on an FTP server need to be backed up, shared, or processed via Google Drive.

The workflow consists of the following logical blocks:

- **1.1 Trigger and Initialization**  
  Starts the workflow manually and initiates FTP directory listing.

- **1.2 FTP Directory Listing and Filtering**  
  Retrieves the list of files and filters out directories, retaining only actual files.

- **1.3 File Download from FTP**  
  Downloads each filtered file as binary data from the FTP server.

- **1.4 Upload to Google Drive**  
  Uploads the downloaded files to a specified folder in Google Drive.

- **1.5 Workflow Termination**  
  Final no-operation nodes to end branches cleanly.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block sets off the workflow manually and starts the process by listing the FTP directory contents.

- **Nodes Involved:**  
  - Manual Trigger  
  - List FTP Directory  
  - Workflow Documentation (sticky note)

- **Node Details:**

  - **Manual Trigger**  
    - *Type:* Trigger node  
    - *Role:* Starts the workflow on user command  
    - *Configuration:* No parameters needed; manual activation  
    - *Connections:* Outputs to "List FTP Directory"  
    - *Failures:* None expected; user interaction required

  - **List FTP Directory**  
    - *Type:* FTP node (operation: list)  
    - *Role:* Lists all files and directories in the root FTP folder (`/`)  
    - *Configuration:*  
      - Path: `/` (root folder of FTP)  
      - Operation: `list` to get directory entries  
      - Credentials: FTP account with server, username, password  
    - *Connections:* Takes input from "Manual Trigger", outputs to "Filter Files Only"  
    - *Failures:* Authentication errors, connection timeouts, directory access errors possible

  - **Workflow Documentation** (sticky note)  
    - Describes the high-level workflow steps and required configurations.

#### 2.2 FTP Directory Listing and Filtering

- **Overview:**  
  Filters the FTP directory listing to retain only files (excluding directories) to prepare for download.

- **Nodes Involved:**  
  - Filter Files Only  
  - Processing Notes (sticky note)

- **Node Details:**

  - **Filter Files Only**  
    - *Type:* IF node  
    - *Role:* Filters items where the `type` field equals `-` (indicating a file in FTP listing)  
    - *Configuration:*  
      - Condition: `$json.type === "-"`  
      - If true: proceed to download  
      - If false: go to no-op  
    - *Connections:* Input from "List FTP Directory"; outputs to "Download File from FTP" (true) or "No Operation, do nothing" (false)  
    - *Failures:* Expression errors if `type` field is missing or malformed

  - **Processing Notes** (sticky note)  
    - Explains filtering logic and notes that filtering can be adjusted.

#### 2.3 File Download from FTP

- **Overview:**  
  Downloads each filtered file from FTP as binary data, preparing it for upload.

- **Nodes Involved:**  
  - Download File from FTP

- **Node Details:**

  - **Download File from FTP**  
    - *Type:* FTP node (operation: download)  
    - *Role:* Downloads files from FTP server using the file path from the filtered listing  
    - *Configuration:*  
      - Path: Set dynamically from `$json.path` of each file  
      - Binary property set to file name (`$json.name`)  
      - Credentials: Same FTP account as before  
    - *Connections:* Input from "Filter Files Only" (true branch), outputs to "Upload to Google Drive"  
    - *Failures:* File not found, permission denied, timeout, connection errors possible

#### 2.4 Upload to Google Drive

- **Overview:**  
  Uploads each downloaded file binary to a specified Google Drive folder.

- **Nodes Involved:**  
  - Upload to Google Drive

- **Node Details:**

  - **Upload to Google Drive**  
    - *Type:* Google Drive node (upload)  
    - *Role:* Uploads files into Google Drive folder  
    - *Configuration:*  
      - Drive: "My Drive" selected dynamically  
      - Folder ID: `1ABCdefGHIjklMNOpqrSTUvwxyz` (target folder in Google Drive)  
      - Uses binary data from previous node  
      - Credentials: Google Drive OAuth2  
    - *Connections:* Input from "Download File from FTP", outputs to "No Operation, do nothing1"  
    - *Failures:* Authentication errors, quota limits, API errors, file size limits

#### 2.5 Workflow Termination

- **Overview:**  
  Ends the workflow branches cleanly with no-operation nodes.

- **Nodes Involved:**  
  - No Operation, do nothing  
  - No Operation, do nothing1

- **Node Details:**

  - **No Operation, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Ends the false branch of filtering (non-file entries)  
    - *Configuration:* No parameters  
    - *Connections:* Input from "Filter Files Only" (false branch)

  - **No Operation, do nothing1**  
    - *Type:* NoOp node  
    - *Role:* Ends the successful upload branch  
    - *Configuration:* No parameters  
    - *Connections:* Input from "Upload to Google Drive"

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                         | Input Node(s)         | Output Node(s)             | Sticky Note                                                                                                         |
|------------------------|---------------------|---------------------------------------|-----------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger      | Starts workflow manually               | —                     | List FTP Directory          | ## FTP to Google Drive Transfer Workflow This workflow: 1. **Manually triggered** - Click Execute to start ...      |
| List FTP Directory     | FTP                 | Lists all files and directories in FTP root | Manual Trigger        | Filter Files Only           | See above                                                                                                           |
| Filter Files Only      | IF                  | Filters for files only (type == "-")  | List FTP Directory     | Download File from FTP (T), No Operation, do nothing (F) | ## File Processing This section processes each file found in the FTP directory: Filters for actual files...          |
| Download File from FTP | FTP                 | Downloads filtered files from FTP      | Filter Files Only (T)  | Upload to Google Drive      | See above                                                                                                           |
| Upload to Google Drive | Google Drive        | Uploads downloaded files to Drive      | Download File from FTP | No Operation, do nothing1   | See above                                                                                                           |
| No Operation, do nothing | No Operation       | Ends false branch after filtering      | Filter Files Only (F)  | —                          |                                                                                                                     |
| No Operation, do nothing1 | No Operation       | Ends successful upload branch           | Upload to Google Drive | —                          |                                                                                                                     |
| Workflow Documentation | Sticky Note         | Describes workflow purpose and config  | —                     | —                          | ## FTP to Google Drive Transfer Workflow This workflow: 1. **Manually triggered** - Click Execute to start ...      |
| Processing Notes       | Sticky Note         | Notes on file processing and filtering | —                     | —                          | ## File Processing This section processes each file found in the FTP directory: Filters for actual files...          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Trigger" node**  
   - Type: Manual Trigger  
   - No parameters  
   - Position: (optional) top-left for clarity

2. **Create "List FTP Directory" node**  
   - Type: FTP  
   - Operation: List  
   - Path: `/` (root directory)  
   - Connect input from "Manual Trigger" (main output)  
   - Set FTP credentials (server, username, password)

3. **Create "Filter Files Only" node (IF node)**  
   - Type: IF  
   - Condition: Check if `$json.type === "-"` (indicates a file entry)  
   - Connect input from "List FTP Directory"  
   - True branch: continue to download  
   - False branch: no-op end

4. **Create "Download File from FTP" node**  
   - Type: FTP  
   - Operation: Download  
   - Path: `={{ $json.path }}` (dynamic path from filtered file)  
   - Binary Property Name: `={{ $json.name }}` (filename used as binary property)  
   - Connect input from "Filter Files Only" (true output)  
   - Use same FTP credentials as before

5. **Create "Upload to Google Drive" node**  
   - Type: Google Drive  
   - Operation: Upload (default)  
   - Drive: Select "My Drive" or appropriate drive  
   - Folder ID: Enter the target Google Drive folder ID (e.g., `1ABCdefGHIjklMNOpqrSTUvwxyz`)  
   - Connect input from "Download File from FTP"  
   - Use Google Drive OAuth2 credentials

6. **Create "No Operation, do nothing" node**  
   - Type: No Operation  
   - Connect input from "Filter Files Only" (false output)  
   - Purpose: End non-file branch

7. **Create "No Operation, do nothing1" node**  
   - Type: No Operation  
   - Connect input from "Upload to Google Drive"  
   - Purpose: End successful upload branch

8. **Add sticky notes (optional but recommended)**  
   - "Workflow Documentation" near the start, summarizing the workflow and credentials required  
   - "Processing Notes" near filtering nodes, explaining filtering logic and customization tips

9. **Configure credentials**  
   - FTP credentials with correct server, username, password  
   - Google Drive OAuth2 credentials with sufficient scopes to upload files to the target folder

10. **Save and activate the workflow**  
    - Run manual trigger to test  
    - Monitor for errors such as FTP connection, file access, or Google Drive API limits

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow requires valid FTP credentials and Google Drive OAuth2 credentials with file upload permission. | Credential setup in n8n settings                        |
| Adjust the file filtering logic in the "Filter Files Only" node if your FTP server uses different type indicators. | Processing Notes sticky note                            |
| Google Drive folder ID must be valid and accessible by the OAuth2 user.                             | Replace placeholder folder ID with your actual folder ID |
| For large files or many files, consider adding error handling and retries for FTP and Google Drive nodes. | Best practices for robust workflows                     |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. It fully complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.