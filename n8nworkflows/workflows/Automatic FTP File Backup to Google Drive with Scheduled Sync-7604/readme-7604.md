Automatic FTP File Backup to Google Drive with Scheduled Sync

https://n8nworkflows.xyz/workflows/automatic-ftp-file-backup-to-google-drive-with-scheduled-sync-7604


# Automatic FTP File Backup to Google Drive with Scheduled Sync

### 1. Workflow Overview

This workflow automates the process of backing up files from an FTP server to Google Drive on a scheduled basis. It is designed to facilitate reliable file synchronization, backup, and data transfer between an FTP folder and a Google Drive folder. The workflow runs periodically (default: every hour) and performs the following logical steps:

- **1.1 Scheduled Trigger:** Initiates the workflow on a fixed time interval using a cron job.
- **1.2 FTP Interaction:** Lists files in a specified FTP folder and downloads each file found.
- **1.3 Google Drive Upload:** Uploads the downloaded files to a designated Google Drive folder.

This structure ensures that files from the FTP server are regularly backed up to Google Drive without manual intervention.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger Block

- **Overview:**  
  This block triggers the workflow execution automatically on a schedule, ensuring periodic syncing without manual start.

- **Nodes Involved:**  
  - Start: Every Hour

- **Node Details:**

  - **Start: Every Hour**  
    - Type: Cron Trigger  
    - Technical Role: Initiates the workflow at defined intervals.  
    - Configuration: Set to trigger every hour (mode: everyHour).  
    - Expressions/Variables: None.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "FTP: List Files".  
    - Version Requirements: None specific.  
    - Edge Cases/Potential Failures: Misconfiguration of cron timing could cause unintended trigger frequency. Cron node failures are rare but possible due to system clock issues.  
    - Sub-workflow: None.

#### 2.2 FTP Interaction Block

- **Overview:**  
  This block connects to the FTP server, lists files in the target folder, and downloads each file sequentially.

- **Nodes Involved:**  
  - FTP: List Files  
  - FTP: Download File

- **Node Details:**

  - **FTP: List Files**  
    - Type: FTP Node (List operation)  
    - Technical Role: Retrieves a list of files from the specified FTP folder.  
    - Configuration:  
      - Operation: list  
      - Path: Provided dynamically via expression `/{{FTP_FOLDER}}`, where `FTP_FOLDER` is a user-defined variable representing the FTP directory to sync.  
    - Expressions/Variables: Uses expression to inject folder path from environment or workflow variables.  
    - Inputs: Receives trigger from "Start: Every Hour".  
    - Outputs: Sends each file’s metadata to "FTP: Download File".  
    - Version Requirements: None specific.  
    - Edge Cases/Potential Failures:  
      - Authentication errors due to invalid FTP credentials.  
      - Empty folder returns no files; workflow gracefully skips download step.  
      - Network timeouts or FTP server unavailability.  
      - Permissions errors on FTP folder.  
    - Sub-workflow: None.

  - **FTP: Download File**  
    - Type: FTP Node (Download operation)  
    - Technical Role: Downloads each individual file listed by the previous node.  
    - Configuration:  
      - Operation: download (implicit by node type)  
      - Path: Dynamically set using expression `{{$json["path"]}}` to specify the file path of each file listed.  
      - Options: Default (empty).  
    - Expressions/Variables: Uses the path property from each file item output by the "FTP: List Files" node.  
    - Inputs: Receives file metadata from "FTP: List Files".  
    - Outputs: Passes binary file data to "Google Drive: Upload".  
    - Version Requirements: None specific.  
    - Edge Cases/Potential Failures:  
      - Download failures due to file locks, permission issues, or file disappearing between listing and download.  
      - Large file sizes may cause timeouts or memory issues depending on n8n environment.  
      - Connection drops during download.  
    - Sub-workflow: None.

#### 2.3 Google Drive Upload Block

- **Overview:**  
  This block receives the downloaded files and uploads them into a specified folder in Google Drive using OAuth2 authentication.

- **Nodes Involved:**  
  - Google Drive: Upload

- **Node Details:**

  - **Google Drive: Upload**  
    - Type: Google Drive Node (Upload operation)  
    - Technical Role: Uploads binary files received from the FTP download step to Google Drive.  
    - Configuration:  
      - Operation: Upload with binary data enabled.  
      - Authentication: OAuth2 with configured Google Drive credentials named "My Drive".  
      - Folder ID: Expected (though not explicitly shown in parameters) to be set via credentials or parameters (recommended to configure the target folder ID).  
    - Expressions/Variables: None explicit in node parameters; relies on credential configuration and binary input data.  
    - Inputs: Receives binary file data from "FTP: Download File".  
    - Outputs: None further connected.  
    - Version Requirements: OAuth2 credential support required (n8n version supporting Google Drive OAuth2).  
    - Edge Cases/Potential Failures:  
      - OAuth2 token expiration or revoked permission.  
      - Upload quota exceeded on Google Drive.  
      - File name conflicts or forbidden characters in filenames.  
      - Network interruptions during upload.  
    - Sub-workflow: None.

#### 2.4 Documentation Block

- **Overview:**  
  Provides an embedded sticky note with detailed information and instructions about the workflow's purpose, operation, and setup.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Documentation Node  
    - Technical Role: Provides human-readable instructions and overview directly in the workflow canvas.  
    - Content Summary:  
      - Workflow purpose: FTP to Google Drive sync.  
      - Step-by-step explanation of operation.  
      - Setup instructions for FTP credentials, folder variables, and Google Drive OAuth2.  
      - Scheduling customization advice.  
    - Position: Not connected in data flow; purely informational.  
    - Edge Cases: None applicable.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                      |
|---------------------|---------------------|-------------------------|-----------------------|------------------------|-------------------------------------------------------------------------------------------------|
| Start: Every Hour    | Cron Trigger        | Scheduled workflow start| None                  | FTP: List Files        |                                                                                                 |
| FTP: List Files      | FTP Node (list)     | List files in FTP folder| Start: Every Hour      | FTP: Download File     |                                                                                                 |
| FTP: Download File   | FTP Node (download) | Download listed files   | FTP: List Files        | Google Drive: Upload   |                                                                                                 |
| Google Drive: Upload | Google Drive Node   | Upload files to Drive   | FTP: Download File     | None                   |                                                                                                 |
| Sticky Note         | Sticky Note          | Documentation           | None                  | None                   | # FTP to Google Drive File Sync\n\n## What this workflow does\nThis workflow automatically **downloads files from an FTP folder** and **uploads them to Google Drive**.  \nIt’s useful for backup, reporting, or syncing data between systems.\n\n## How it works\n1. **Cron Trigger** runs on a schedule (default: every hour).  \n2. **FTP List node** checks the specified folder on your FTP server.  \n3. **FTP Download node** fetches each file found.  \n4. **Google Drive Upload node** saves the file into your chosen Google Drive folder.\n\n## Setup\n- **FTP:** Add your FTP credentials and replace `{{FTP_FOLDER}}` with the path of the folder you want to sync.  \n- **Google Drive:** Connect your Google Drive account and replace `{{GDRIVE_FOLDER_ID}}` with the folder ID where files should be stored.  \n- Adjust the Cron schedule (e.g. daily, weekly) depending on how often you need the sync. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node**  
   - Node Name: `Start: Every Hour`  
   - Type: Cron Trigger  
   - Parameters: Set trigger mode to "Every Hour" to run the workflow once per hour.

2. **Create an FTP node for listing files**  
   - Node Name: `FTP: List Files`  
   - Type: FTP  
   - Parameters:  
     - Operation: `list`  
     - Path: `/{{FTP_FOLDER}}` (replace `{{FTP_FOLDER}}` with the target FTP directory path or use an environment variable).  
   - Credentials: Configure FTP credentials with server hostname, username, password, and port as required.

3. **Create an FTP node for downloading files**  
   - Node Name: `FTP: Download File`  
   - Type: FTP  
   - Parameters:  
     - Path: Use expression `{{$json["path"]}}` to dynamically set path from each listed file.  
     - Options: Leave default unless specific FTP client options are needed.  
   - Credentials: Use the same FTP credentials as the listing node.

4. **Create a Google Drive node for uploading files**  
   - Node Name: `Google Drive: Upload`  
   - Type: Google Drive  
   - Parameters:  
     - Binary Data: Enabled (to upload actual file content).  
     - Authentication: Use OAuth2.  
     - Set target folder ID (usually under options or directly in credentials) to the Google Drive folder where files will be uploaded.  
   - Credentials: Configure Google Drive OAuth2 credentials with appropriate scopes for file upload.

5. **Connect nodes in this order:**  
   - `Start: Every Hour` → `FTP: List Files`  
   - `FTP: List Files` → `FTP: Download File`  
   - `FTP: Download File` → `Google Drive: Upload`

6. **(Optional) Add a Sticky Note node**  
   - To document the workflow steps, create a Sticky Note node with the provided explanation text for easier maintenance.

7. **Variables and Environment Setup:**  
   - Define `FTP_FOLDER` as either a workflow variable or environment variable containing the FTP directory path.  
   - Define Google Drive folder ID (`GDRIVE_FOLDER_ID`) as part of OAuth2 credential or in node parameters if supported.

8. **Testing:**  
   - Run the workflow manually to verify connectivity to FTP and Google Drive.  
   - Confirm files are listed, downloaded, and uploaded correctly.  
   - Monitor logs for errors such as authentication failures or file permission issues.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses OAuth2 credentials for Google Drive; ensure scopes include file upload permissions.          | Google Drive OAuth2 credential setup in n8n documentation.                                      |
| Adjust Cron schedule as needed for different backup frequencies (e.g., daily at midnight).                  | Cron expressions reference: https://crontab.guru/                                               |
| FTP credentials must have read permissions on the target folder for listing and downloading files.         | Ensure FTP server access and firewall rules allow connections from n8n host.                     |
| Large files or slow networks may require increasing workflow timeout and memory limits in n8n configuration.| n8n runtime environment settings for resource limits.                                           |
| Google Drive folder ID can be obtained from the folder’s URL in Google Drive (part after `folders/`).      | https://support.google.com/drive/answer/2424384?hl=en                                           |

---

**Disclaimer:** The text above is generated exclusively from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.