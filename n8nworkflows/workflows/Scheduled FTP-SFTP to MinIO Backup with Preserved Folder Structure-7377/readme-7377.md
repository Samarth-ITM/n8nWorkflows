Scheduled FTP/SFTP to MinIO Backup with Preserved Folder Structure

https://n8nworkflows.xyz/workflows/scheduled-ftp-sftp-to-minio-backup-with-preserved-folder-structure-7377


# Scheduled FTP/SFTP to MinIO Backup with Preserved Folder Structure

### 1. Workflow Overview

This workflow automates the scheduled backup of files and folders from an FTP/SFTP server to a MinIO object storage bucket, preserving the original folder structure. It is designed for use cases such as backing up website files (e.g., WordPress uploads) or any other data residing on an FTP/SFTP server, ensuring data integrity and easy retrieval. The workflow logically divides into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the backup process on a predefined schedule.
- **1.2 FTP/SFTP Folder Listing:** Recursively lists all files and folders at the specified remote path.
- **1.3 Path Preparation:** Extracts and transforms the file path to maintain folder structure in MinIO.
- **1.4 File Download:** Downloads each file from FTP/SFTP based on the prepared paths.
- **1.5 File Upload to MinIO:** Uploads the downloaded files to the appropriate path on MinIO, preserving folder hierarchy.
- **1.6 Configuration Notes:** Sticky notes providing configuration instructions and best practices.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution daily at 2 AM, automating the backup without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Role: Initiates the workflow on a schedule.
    - Configuration: Set to trigger daily at 2:00 AM (triggerAtHour: 2).
    - Inputs: None (trigger node).
    - Outputs: Connects to "List FTP Folder contents".
    - Edge Cases: If the n8n instance is down at trigger time, the execution may be missed unless "catch-up" is enabled in n8n settings.
    - Version: 1.2
    - Notes: This node must be configured according to desired backup frequency.

---

#### 2.2 FTP/SFTP Folder Listing

- **Overview:**  
  Lists all files and folders recursively in the specified FTP/SFTP directory to prepare for downloading.

- **Nodes Involved:**  
  - List FTP Folder contents

- **Node Details:**

  - **List FTP Folder contents**
    - Type: `n8n-nodes-base.ftp`
    - Role: Retrieves file and folder listings recursively from the remote SFTP server.
    - Configuration:
      - Protocol: SFTP
      - Operation: List
      - Path: `/bitnami/wordpress/wp-content/uploads/2025/04` (example folder)
      - Recursive: true (ensures subfolders and all files are included)
    - Inputs: Connected from Schedule Trigger.
    - Outputs: Connected to "Create Path for MinIO".
    - Credentials: Uses stored SFTP credentials.
    - Edge Cases:
      - Connection/authentication failures if credentials are incorrect or server is unreachable.
      - Large directories may cause timeouts or performance bottlenecks.
      - Permission issues on remote server could prevent listing.
    - Version: 1

---

#### 2.3 Path Preparation

- **Overview:**  
  Processes each file path from the FTP list to generate a corresponding folder path for MinIO upload, maintaining directory hierarchy.

- **Nodes Involved:**  
  - Create Path for MinIO

- **Node Details:**

  - **Create Path for MinIO**
    - Type: `n8n-nodes-base.set`
    - Role: Extracts the folder path by removing the filename from the full FTP path.
    - Configuration:
      - Assignment of a new field `path` using a JavaScript expression:
        - `{{$json.path.replace(/\\/[^\\/]+$/, '')}}`
        - This regex strips the last path segment (file name) to get the parent folder.
    - Inputs: Connected from "List FTP Folder contents".
    - Outputs: Connected to "Download content".
    - Edge Cases:
      - If path is missing or malformed, expression may fail or produce incorrect folder paths.
    - Version: 3.4

---

#### 2.4 File Download

- **Overview:**  
  Downloads individual files from the FTP/SFTP server using the paths prepared to feed the upload step.

- **Nodes Involved:**  
  - Download content

- **Node Details:**

  - **Download content**
    - Type: `n8n-nodes-base.ftp`
    - Role: Downloads each file from the FTP server for upload.
    - Configuration:
      - Protocol: SFTP
      - Operation: Download (default implied by FTP node settings when path points to a file)
      - Path: Set dynamically from the previous node using expression `={{ $('List FTP Folder contents').item.json.path }}`
    - Inputs: Connected from "Create Path for MinIO".
    - Outputs: Connected to "Upload on MinIO with correct Path".
    - Credentials: Uses the same SFTP credentials as listing node.
    - Edge Cases:
      - Download failure if file is deleted/moved between listing and download.
      - Connection drops or timeouts.
      - Large file sizes may slow or interrupt download.
    - Version: 1

---

#### 2.5 File Upload to MinIO

- **Overview:**  
  Uploads each downloaded file to a MinIO bucket, placing it inside the corresponding folder to preserve the original FTP folder structure.

- **Nodes Involved:**  
  - Upload on MinIO with correct Path

- **Node Details:**

  - **Upload on MinIO with correct Path**
    - Type: `n8n-nodes-base.s3`
    - Role: Uploads files to an S3-compatible storage (MinIO).
    - Configuration:
      - Operation: Upload
      - Bucket Name: `myMinIObucket` (example bucket)
      - FileName: Empty string (defaults to original filename from binary data)
      - Additional Fields:
        - `parentFolderKey` set dynamically as `ftpFilesFolder{{ $('Create Path for MinIO').item.json.path }}`
        - This ensures files are uploaded under a folder named `ftpFilesFolder` followed by the relative path derived from the FTP server.
    - Inputs: Receives binary file data from "Download content".
    - Outputs: None (end of flow).
    - Credentials: Uses configured MinIO S3 credentials.
    - Edge Cases:
      - Upload failures due to network issues or MinIO permissions.
      - Bucket or folder path not existing—MinIO will usually create folders implicitly but misconfiguration can cause errors.
    - Version: 1

---

#### 2.6 Configuration Notes (Sticky Notes)

- **Overview:**  
  Provides user guidance on configuration and deployment context.

- **Nodes Involved:**  
  - Sticky Note32 (Configuring Trigger)
  - Sticky Note (MinIO Bucket Setup)
  - Sticky Note33 (Workflow Introduction and Tips)

- **Node Details:**

  - **Sticky Note32**
    - Content: "## Configure this trigger"
    - Position: Near Schedule Trigger node to remind user to customize trigger.
  
  - **Sticky Note**
    - Content: "## Create and configure your MinIO Bucket"
    - Position: Near MinIO upload node, reminding users to prepare MinIO bucket.

  - **Sticky Note33**
    - Content:  
      ```
      ## Automated FTP-SFTP to MinIO Object Backup with Scheduling 

      Want to automatically backup Files & Folder from any FTP-SFTP Server (Like your Wordpress Website !) ? You're at the right place 

      Configure the Schedule Trigger to best suit your needs, create a SFTP account on N8N and write the path to the files you want to backup (could be a simple folder or an entire server) and don't forget to check the Recursive Option (to get all files !)

      Then, simply deploy a MinIO service on your network (Using Proxmox VE ? create a LXC using https://community-scripts.github.io/ProxmoxVE/scripts?id=minio) and configure the path to the bucket where your FTP Files will be (like 192.168.1.123:9000 and your bucket)

      ## -> Pro Tip, you can change the MinIO Node for an AWS S3, Azure Blob, GCP or any S3 Compatible Storage that you want !
      ```
    - Position: Top-left corner, providing a comprehensive overview and tips.
    - Includes a helpful external link for deploying MinIO on Proxmox VE.

---

### 3. Summary Table

| Node Name                       | Node Type                  | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                            |
|--------------------------------|----------------------------|----------------------------------------|------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | scheduleTrigger            | Initiates workflow on schedule (2 AM) | None                         | List FTP Folder contents       | Configure this trigger                                                                                                                 |
| List FTP Folder contents       | ftp                       | Recursively lists files/folders on FTP | Schedule Trigger             | Create Path for MinIO          |                                                                                                                                        |
| Create Path for MinIO          | set                        | Prepares folder path for MinIO upload  | List FTP Folder contents     | Download content               |                                                                                                                                        |
| Download content               | ftp                       | Downloads each file from FTP            | Create Path for MinIO        | Upload on MinIO with correct Path |                                                                                                                                        |
| Upload on MinIO with correct Path | s3                      | Uploads files preserving folder structure | Download content             | None                          | Create and configure your MinIO Bucket                                                                                                |
| Sticky Note32                 | stickyNote                 | Instruction to configure trigger       | None                         | None                          | Configure this trigger                                                                                                                 |
| Sticky Note                   | stickyNote                 | Instruction to configure MinIO bucket  | None                         | None                          | Create and configure your MinIO Bucket                                                                                                |
| Sticky Note33                 | stickyNote                 | Overview, tips, and external resource  | None                         | None                          | Automated FTP-SFTP to MinIO Object Backup with Scheduling. Includes link: https://community-scripts.github.io/ProxmoxVE/scripts?id=minio |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** and name it appropriately (e.g., "Automatic FTP to MinIO Backup").

2. **Add a Schedule Trigger node:**
   - Node Type: `Schedule Trigger`
   - Name: "Schedule Trigger"
   - Configure the trigger rule to run daily at 2:00 AM (set `triggerAtHour` to 2).
   - This node will be the entry point.

3. **Add an FTP node for folder listing:**
   - Node Type: `FTP`
   - Name: "List FTP Folder contents"
   - Set Protocol to `SFTP`.
   - Set Operation to `List`.
   - Set Path to the root directory you want to back up, e.g., `/bitnami/wordpress/wp-content/uploads/2025/04`.
   - Enable Recursive option to true.
   - Use or create SFTP credentials with appropriate username, password/private key for your FTP server.
   - Connect "Schedule Trigger" output to this node's input.

4. **Add a Set node for path preparation:**
   - Node Type: `Set`
   - Name: "Create Path for MinIO"
   - Add a field assignment:
     - Field Name: `path`
     - Type: String
     - Value: Use expression: `{{$json.path.replace(/\\/[^\\/]+$/, '')}}`
   - Connect "List FTP Folder contents" output to this node's input.

5. **Add an FTP node for file download:**
   - Node Type: `FTP`
   - Name: "Download content"
   - Protocol: `SFTP`
   - Operation: Download (default when path points to file)
   - Path: Set dynamically using expression: `={{ $('List FTP Folder contents').item.json.path }}`
   - Use the same SFTP credentials as the listing node.
   - Connect "Create Path for MinIO" output to this node's input.

6. **Add an S3 node for MinIO upload:**
   - Node Type: `S3`
   - Name: "Upload on MinIO with correct Path"
   - Operation: Upload
   - Bucket Name: Set to your MinIO bucket, e.g., `myMinIObucket`.
   - FileName: Leave blank to use the original filename from binary data.
   - In Additional Fields, set `parentFolderKey` to expression: `ftpFilesFolder{{ $('Create Path for MinIO').item.json.path }}`
   - Use or create MinIO S3 credentials with endpoint, access key, and secret key.
   - Connect "Download content" output to this node's input.

7. **Add Sticky Notes for documentation (optional but recommended):**
   - Near "Schedule Trigger": Add a note "## Configure this trigger" to remind about schedule customization.
   - Near "Upload on MinIO with correct Path": Add a note "## Create and configure your MinIO Bucket" to remind bucket setup.
   - At the top-left corner: Add a large note describing the workflow purpose, usage tips, and link to MinIO deployment instructions (https://community-scripts.github.io/ProxmoxVE/scripts?id=minio).

8. **Test the workflow:**
   - Run manually or wait for the scheduled time.
   - Verify files are listed, downloaded, and uploaded properly to MinIO with folder structure preserved.
   - Monitor for errors or performance issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Automated FTP-SFTP to MinIO Object Backup with Scheduling. Can backup entire folder structures recursively.                               | Workflow general description                                                                       |
| Pro Tip: Replace MinIO node with AWS S3, Azure Blob, GCP, or any S3-compatible storage node for alternative cloud storage.                | General workflow flexibility note                                                                  |
| Deploy MinIO on your local network using Proxmox VE LXC container. Setup script available here: https://community-scripts.github.io/ProxmoxVE/scripts?id=minio | MinIO deployment instructions                                                                       |

---

**Disclaimer:**  
The provided workflow content is derived exclusively from an automated n8n workflow and complies fully with content policies. All processed data is legal and public.