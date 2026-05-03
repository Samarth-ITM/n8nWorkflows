Restore Workflows and Credentials from Remote FTP Backup Storage

https://n8nworkflows.xyz/workflows/restore-workflows-and-credentials-from-remote-ftp-backup-storage-9156


# Restore Workflows and Credentials from Remote FTP Backup Storage

### 1. Workflow Overview

This workflow automates the restoration of n8n workflows and credentials from backups stored on a remote FTP server. It is designed primarily for scenarios such as migrating to a new n8n instance, recovering from data loss, or syncing environments by restoring workflow and credential backups. The workflow includes two primary logical blocks:

- **1.1 Restore Credentials from FTP**: Downloads and imports n8n credential files from the latest backup folder on the FTP server.
- **1.2 Restore Workflows from FTP**: Downloads and imports n8n workflow files from the latest backup folder on the FTP server, excluding the currently running workflow to avoid overwrite.

Supporting blocks include initialization and setup, error handling, folder creation on the local server, and notification emails upon success.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Setup

- **Overview**: Establishes environment variables, paths, time zone configurations, and custom workflow parameters. It defines paths for backup folders, temporary restore folders, and admin email. This block sets the foundation for all subsequent operations.
- **Nodes Involved**:  
  - Init
  - Start Restore

- **Node Details**:  
  - **Init**  
    - Type: Code  
    - Role: Defines local dates, timezone, backup paths, project directories, and custom configuration variables. Outputs structured JSON with all config data.  
    - Key expressions: Uses environment variables (e.g., `$env.N8N_ADMIN_EMAIL`, `$env.FTP_BACKUP_FOLDER`), local timezone detection or user-defined timezone, and prepares paths for workflows and credentials restore folders.  
    - Input: Manual trigger node ("Start Restore") initiates it.  
    - Output: JSON object with `timeData`, `workflowConfig`, and `customConfig`.  
    - Edge cases: Environment variables missing or misconfigured may cause wrong paths or emails. Timezone misconfiguration could affect logging or scheduling.  
  - **Start Restore**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow. Accepts JSON input parameters to specify whether to restore credentials, workflows, or both.  
    - Configuration: Default payload example `{ "credentials": true, "worflows": false }` (note the typo "worflows" is intentional in the workflow).  
    - Output: Triggers the Init node.

#### 2.2 Finding the Latest Backup Folder on FTP

- **Overview**: Lists the remote FTP backup directory and identifies the most recent backup based on folder names formatted as dates (YYYY-MM-DD). Extracts paths for credentials and workflows subfolders for that backup date.
- **Nodes Involved**:  
  - List Credentials Folders  
  - Find Last Backup  
  - ERROR: Find Most Recent Credentials Folder

- **Node Details**:  
  - **List Credentials Folders**  
    - Type: FTP  
    - Role: Lists directories/files at the root FTP backup path to retrieve candidate backup folders.  
    - Configuration: Path set from Init node custom config `FTP_BACKUP_FOLDER`.  
    - Credentials: FTP credential required.  
    - Output: FTP listing passed to Find Last Backup.  
    - Edge cases: FTP connectivity issues, permission errors, empty directory.  
  - **Find Last Backup**  
    - Type: Code  
    - Role: Processes FTP folder listing to filter directories matching date format YYYY-MM-DD, sorts them descending, selects the newest backup folder, and builds full paths for credentials and workflows subfolders.  
    - Key expressions: Uses regex for date format, sorts using JavaScript Date objects, constructs paths using template strings.  
    - Input: FTP listing from previous node.  
    - Output: JSON with `latestBackupDate`, `ftpCredentialsPath`, `ftpWorkflowsPath`, and other config.  
    - Error handling: If no suitable folders found, triggers error output leading to the error node.  
    - Sub-workflow: None.  
  - **ERROR: Find Most Recent Credentials Folder**  
    - Type: Stop and Error  
    - Role: Stops workflow execution if no backup folder is found, outputs error message from Find Last Backup.  
    - Input: Error output from Find Last Backup.  
    - Output: None (workflow stopped).  

#### 2.3 Creating Temporary Local Folders for Restore

- **Overview**: Creates local temporary folders on the n8n server to store downloaded credential and workflow files before import. Folder creation is conditional based on user choice to restore credentials, workflows, or both.
- **Nodes Involved**:  
  - Create Temp Folder

- **Node Details**:  
  - **Create Temp Folder**  
    - Type: Execute Command (Bash script)  
    - Role: Creates folders named after the latest backup date with suffixes for credentials and workflows respectively. Checks restore flags from Start Restore node to decide which folders to create. Exits with error if neither folder is created.  
    - Key expressions: Uses environment variables and node JSON parameters to build folder paths and decide actions.  
    - Input: Output from Find Last Backup node.  
    - Output: Logs folder creation status or error exit.  
    - Edge cases: Permissions issues on local server, invalid paths, no restore options selected (causes exit with error).  

#### 2.4 Restore Credentials from FTP

- **Overview**: When enabled, downloads credential files from the latest FTP backup folder, writes them to local disk, and imports them into n8n.
- **Nodes Involved**:  
  - Restore Credentials?  
  - List Credentials Files  
  - Download Credential Files  
  - Write Credential Files To Disk  
  - Restore Credentials  
  - SUCCESS email Credentials

- **Node Details**:  
  - **Restore Credentials?**  
    - Type: If  
    - Role: Checks if the "credentials" flag from Start Restore is true to proceed with credentials restore.  
    - Input: Output from Create Temp Folder.  
    - Output: If true, proceeds to List Credentials Files.  
  - **List Credentials Files**  
    - Type: FTP  
    - Role: Lists files in the credentials subfolder of the latest backup on FTP.  
    - Configuration: Path taken from Find Last Backup node output (`ftpCredentialsPath`).  
    - Credentials: FTP credential required.  
  - **Download Credential Files**  
    - Type: FTP  
    - Role: Downloads each credential file listed from FTP.  
    - Input: List from previous node.  
    - Output: Binary files passed to next node.  
  - **Write Credential Files To Disk**  
    - Type: Read/Write File  
    - Role: Writes downloaded credential JSON files to the local credentials temporary folder, path constructed from Init node config and backup date.  
    - Key expressions: File path composed dynamically using backup date and credentials temp folder.  
    - Input: Files from FTP download node.  
  - **Restore Credentials**  
    - Type: Execute Command  
    - Role: Runs `n8n import:credentials` CLI command to import all credentials JSON files from the local temp folder into n8n.  
    - Command: `n8n import:credentials --separate --input=...` (path dynamically built).  
  - **SUCCESS email Credentials** (disabled by default)  
    - Type: Email Send  
    - Role: Sends notification email on successful credentials restore, including logs and paths used.  
    - Configuration: Uses SMTP credentials and environment admin email.  
    - Disabled by default to prevent unwanted emails unless enabled.

- **Edge cases**:  
  - FTP connection or permission errors  
  - File write permission errors on local disk  
  - CLI import failures (e.g., malformed JSON, duplicate credentials)  
  - Disabled email node requires manual enabling for notifications.

#### 2.5 Restore Workflows from FTP

- **Overview**: When enabled, downloads workflow files from the latest FTP backup, writes them locally, excludes the current workflow file to prevent overwrite, and imports remaining workflows into n8n.
- **Nodes Involved**:  
  - Restore Workflows?  
  - List Most Recent Workflows Folder  
  - Download Workflow Files  
  - Filter out Credentials sub-folder  
  - Write Workflow Files To Disk  
  - Exclude Current Workflow From Selection  
  - Restore Workflows  
  - SUCCESS email Workflows

- **Node Details**:  
  - **Restore Workflows?**  
    - Type: If  
    - Role: Checks if the "worflows" flag (note typo) from Start Restore is true to proceed with workflows restore.  
  - **List Most Recent Workflows Folder**  
    - Type: FTP  
    - Role: Lists files in the workflows backup folder on FTP for the latest backup date.  
    - Configuration: Path from Find Last Backup node output (`ftpWorkflowsPath`).  
  - **Download Workflow Files**  
    - Type: FTP  
    - Role: Downloads workflow JSON files from FTP.  
    - On error: Set to continue to allow partial restore if some files fail.  
  - **Filter out Credentials sub-folder**  
    - Type: Filter  
    - Role: Removes any files that are related to credentials subfolder to avoid mixing.  
    - Condition: Filters out items if binary data exists (ensures non-empty files).  
  - **Write Workflow Files To Disk**  
    - Type: Read/Write File  
    - Role: Writes downloaded workflow JSON files to local workflows temp folder.  
  - **Exclude Current Workflow From Selection**  
    - Type: Execute Command (shell script)  
    - Role: Removes the JSON file corresponding to the currently running workflow from the restore folder to prevent self-overwrite, using shell commands to sanitize workflow name and delete the file.  
  - **Restore Workflows**  
    - Type: Execute Command  
    - Role: Runs `n8n import:workflow` CLI command to import all workflow JSON files from the local temp folder into n8n.  
  - **SUCCESS email Workflows** (disabled by default)  
    - Type: Email Send  
    - Role: Sends notification email on successful workflows restore, including logs and paths used.  
    - Disabled by default.

- **Edge cases**:  
  - FTP download errors (partial continue)  
  - File write permission errors  
  - CLI import failures (duplicate, malformed workflows)  
  - Current workflow file not found in restore folder (safe condition)  
  - Disabled email node requires manual enabling.

#### 2.6 Notification and Error Handling

- **Overview**: Sends email notifications on successful completion of credentials and workflows restore. Handles stopping the workflow with error if no valid backup folder found.
- **Nodes Involved**:  
  - SUCCESS email Credentials (disabled)  
  - SUCCESS email Workflows (disabled)  
  - ERROR: Find Most Recent Credentials Folder

- **Node Details**:  
  - **SUCCESS email Credentials / Workflows**  
    - Type: Email Send  
    - Role: Notify admin of success with detailed logs.  
    - Disabled by default, requires SMTP credential setup and enabling.  
  - **ERROR: Find Most Recent Credentials Folder**  
    - Type: Stop and Error  
    - Role: Stops workflow on failure to find backup folder, outputs error message.

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                           | Input Node(s)                | Output Node(s)                         | Sticky Note                                                                                                    |
|-----------------------------------|------------------------|-----------------------------------------|-----------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Start Restore                     | Manual Trigger         | Entry trigger, start restore             | -                           | Init                                  | CRED 1 WF 0                                                                                                    |
| Init                             | Code                   | Initialization of time, paths, config   | Start Restore               | List Credentials Folders               | Contains timezone and folder configuration code.                                                             |
| List Credentials Folders          | FTP                    | List FTP backup root folders             | Init                       | Find Last Backup                      |                                                                                                                |
| Find Last Backup                  | Code                   | Find latest backup folder by date        | List Credentials Folders    | Create Temp Folder, ERROR node        |                                                                                                                |
| ERROR: Find Most Recent Credentials Folder | Stop and Error         | Stops workflow on backup folder not found | Find Last Backup (error)    | -                                     |                                                                                                                |
| Create Temp Folder                | Execute Command        | Create local temp folders for restore    | Find Last Backup            | Restore Credentials?, Restore Workflows? | For credential files on server                                                                                 |
| Restore Credentials?             | If                      | Check if credentials restore requested   | Create Temp Folder          | List Credentials Files                |                                                                                                                |
| List Credentials Files           | FTP                    | List credential files in backup folder   | Restore Credentials?        | Download Credential Files             |                                                                                                                |
| Download Credential Files        | FTP                    | Download credentials files                | List Credentials Files      | Write Credential Files To Disk        |                                                                                                                |
| Write Credential Files To Disk   | Read/Write File        | Write credentials to local disk           | Download Credential Files   | Restore Credentials                   |                                                                                                                |
| Restore Credentials              | Execute Command        | Import credentials into n8n               | Write Credential Files To Disk | SUCCESS email Credentials (disabled) |                                                                                                                |
| SUCCESS email Credentials        | Email Send             | Notify success of credentials restore    | Restore Credentials         | -                                     | Disabled by default                                                                                             |
| Restore Workflows?               | If                      | Check if workflows restore requested     | Create Temp Folder          | List Most Recent Workflows Folder     |                                                                                                                |
| List Most Recent Workflows Folder | FTP                    | List workflow files in backup folder     | Restore Workflows?          | Download Workflow Files               |                                                                                                                |
| Download Workflow Files          | FTP                    | Download workflow files                   | List Most Recent Workflows Folder | Filter out Credentials sub-folder    | Ignore Credentials sub-workflow error                                                                          |
| Filter out Credentials sub-folder | Filter                  | Filter out credential files from workflows | Download Workflow Files     | Write Workflow Files To Disk          |                                                                                                                |
| Write Workflow Files To Disk     | Read/Write File        | Write workflows to local disk             | Filter out Credentials sub-folder | Exclude Current Workflow From Selection |                                                                                                                |
| Exclude Current Workflow From Selection | Execute Command        | Remove current running workflow file to avoid overwrite | Write Workflow Files To Disk | Restore Workflows                    |                                                                                                                |
| Restore Workflows               | Execute Command        | Import workflows into n8n                  | Exclude Current Workflow From Selection | SUCCESS email Workflows (disabled)     |                                                                                                                |
| SUCCESS email Workflows          | Email Send             | Notify success of workflows restore      | Restore Workflows           | -                                     | Disabled by default                                                                                             |
| Sticky Note                     | Sticky Note            | Section header "## 2. Restore Workflows from FTP" | -                           | -                                     |                                                                                                                |
| Sticky Note1                    | Sticky Note            | Instructions, warnings, and start guide  | -                           | -                                     | Explains warnings about restore order, folder config, and usage instructions                                   |
| Sticky Note2                    | Sticky Note            | Section header "## 1. Restore Credentials from FTP" | -                           | -                                     |                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "Start Restore"  
   - Purpose: Start the workflow manually with JSON input parameters.  
   - Parameters: No default; instruct user to provide JSON input like `{ "credentials": true, "worflows": false }` (note typo `worflows`).  

2. **Create Code Node for Initialization**  
   - Name: "Init"  
   - Purpose: Initialize time zone, paths, environment variables, and custom configuration.  
   - Configuration: Paste the provided JavaScript code configuring timezone, project folders, backup folders, and output a JSON object with `timeData`, `workflowConfig`, and `customConfig`.  
   - Connect "Start Restore" to "Init".  

3. **Create FTP Node to List Backup Root Folder**  
   - Name: "List Credentials Folders"  
   - Operation: List  
   - Path: Use expression to get `customConfig.FTP_BACKUP_FOLDER` from "Init" node output.  
   - Credentials: Configure FTP credentials for your backup server.  
   - Connect "Init" to this node.  

4. **Create Code Node to Find Latest Backup**  
   - Name: "Find Last Backup"  
   - Purpose: From FTP listing, find the newest backup folder with YYYY-MM-DD format.  
   - Code: Use provided JavaScript to filter directories, sort dates, and output paths for workflows and credentials.  
   - Configure On Error to "Continue" using error output.  
   - Connect "List Credentials Folders" to "Find Last Backup".  

5. **Create Stop and Error Node for Failure Case**  
   - Name: "ERROR: Find Most Recent Credentials Folder"  
   - Error Message: Use expression `{{ $json.error }}` from "Find Last Backup" error output.  
   - Connect error output of "Find Last Backup" to this node.  

6. **Create Execute Command Node to Create Temporary Folders**  
   - Name: "Create Temp Folder"  
   - Command: Bash script that reads from "Init" and "Find Last Backup" to create temp folders for credentials and workflows based on user restore flags.  
   - Connect success output of "Find Last Backup" to "Create Temp Folder".  

7. **Create If Node to Check Credentials Restore Flag**  
   - Name: "Restore Credentials?"  
   - Condition: Boolean check if `Start Restore` JSON property `credentials` is true.  
   - Connect "Create Temp Folder" success output to this node.  

8. **Create FTP Node to List Credential Files**  
   - Name: "List Credentials Files"  
   - Path: From "Find Last Backup" output `ftpCredentialsPath`.  
   - Credentials: Use FTP credential.  
   - Connect "Restore Credentials?" true output here.  

9. **Create FTP Node to Download Credential Files**  
   - Name: "Download Credential Files"  
   - Path: Use expression `{{ $json.path }}` from list node output.  
   - Credentials: FTP credential.  
   - Connect "List Credentials Files" to this node.  

10. **Create Read/Write File Node to Write Credential Files**  
    - Name: "Write Credential Files To Disk"  
    - Operation: Write  
    - File Name: Use expression combining project root path, latest backup date, credentials temp folder, and filename.  
    - Connect "Download Credential Files" to this node.  

11. **Create Execute Command Node to Import Credentials**  
    - Name: "Restore Credentials"  
    - Command: `n8n import:credentials --separate --input=<path>` using dynamic path from Init and Find Last Backup.  
    - Connect "Write Credential Files To Disk" to this node.  

12. **Create Disabled Email Node to Notify Credential Restore Success**  
    - Name: "SUCCESS email Credentials"  
    - Configure SMTP credentials and email content using expressions referencing other nodes.  
    - Connect "Restore Credentials" to this node.  
    - Disable node by default, enable as needed.  

13. **Create If Node to Check Workflows Restore Flag**  
    - Name: "Restore Workflows?"  
    - Condition: Boolean check if `Start Restore` JSON property `worflows` (note typo) is true.  
    - Connect "Create Temp Folder" success output to this node.  

14. **Create FTP Node to List Workflow Files**  
    - Name: "List Most Recent Workflows Folder"  
    - Path: From Find Last Backup output `ftpWorkflowsPath`.  
    - Credentials: FTP credential.  
    - Connect "Restore Workflows?" true output here.  

15. **Create FTP Node to Download Workflow Files**  
    - Name: "Download Workflow Files"  
    - Path: Use expression `{{ $json.path }}` from list node output.  
    - Credentials: FTP credential.  
    - On error: Set to continue on error.  
    - Connect "List Most Recent Workflows Folder" to this node.  

16. **Create Filter Node to Exclude Credentials Subfolder**  
    - Name: "Filter out Credentials sub-folder"  
    - Condition: Filter out entries where binary data is missing (to ignore invalid files).  
    - Connect "Download Workflow Files" to this node.  

17. **Create Read/Write File Node to Write Workflow Files**  
    - Name: "Write Workflow Files To Disk"  
    - Operation: Write  
    - File Name: Use project root path, latest backup date, workflows temp folder, and filename.  
    - Connect "Filter out Credentials sub-folder" to this node.  

18. **Create Execute Command Node to Remove Current Workflow File**  
    - Name: "Exclude Current Workflow From Selection"  
    - Script: Shell script to sanitize current workflow name and delete matching JSON file in restore folder to avoid overwrite.  
    - Connect "Write Workflow Files To Disk" to this node.  

19. **Create Execute Command Node to Import Workflows**  
    - Name: "Restore Workflows"  
    - Command: `n8n import:workflow --separate --input=<path>` using dynamic path.  
    - Connect "Exclude Current Workflow From Selection" to this node.  

20. **Create Disabled Email Node to Notify Workflow Restore Success**  
    - Name: "SUCCESS email Workflows"  
    - Configure SMTP credentials and email content.  
    - Connect "Restore Workflows" to this node.  
    - Disable node by default.  

21. **Create Sticky Note Nodes**  
    - Add sticky notes for sections "## 1. Restore Credentials from FTP" and "## 2. Restore Workflows from FTP" and a large instructional sticky note at the start with warnings and usage instructions.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ⚠️ Warning: When doing a full restore, restore credentials before workflows. Existing credentials or workflows will be overwritten.      | Sticky Note1 contains this warning and usage instructions including sample JSON for Start Restore parameters.                                                                  |
| This workflow can be used to migrate workflows and credentials from one n8n instance to another by leveraging FTP backups.                 | Sticky Note1 instructions.                                                                                                                                                      |
| The workflow requires pre-configured FTP credentials for the remote backup server and optionally SMTP credentials for email notifications. | Use your own FTP credential and SMTP credential in the respective nodes.                                                                                                       |
| The workflow assumes backup folders on FTP are named by date in YYYY-MM-DD format and contain subfolders for credentials and workflows.   | The Find Last Backup code enforces this naming convention and filters accordingly.                                                                                              |
| The shell scripts sanitize filenames to avoid overwrite and handle spaces and special characters in workflow names.                       | See the shell script in "Exclude Current Workflow From Selection" node.                                                                                                         |
| Email notification nodes are disabled by default to avoid unintentional emails; enable and configure them as needed.                      | SUCCESS email Credentials and SUCCESS email Workflows nodes.                                                                                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.