Automated Workflow & Credential Restoration System for Self-Hosted Environments

https://n8nworkflows.xyz/workflows/automated-workflow---credential-restoration-system-for-self-hosted-environments-9154


# Automated Workflow & Credential Restoration System for Self-Hosted Environments

### 1. Workflow Overview

This workflow is designed to automate the restoration of n8n credentials and workflows from disk backups in self-hosted environments. It targets use cases such as migrating n8n instances, disaster recovery, or routine restoration after data loss. The workflow is structured into two main logical blocks:

- **1.1 Initialization and Backup Discovery**: Prepares environment variables, paths, and searches for the most recent backup folder containing credentials and workflow files.
- **1.2 Restoration Execution**: Conditionally restores credentials and workflows from the identified backup folder, excluding the currently running workflow to avoid conflicts, and sends success notification emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Backup Discovery

**Overview:**  
This block initializes configuration parameters including timezones, folder paths, and backup folder locations. It then lists available backup folders and determines the most recent backup date to target for restoration.

**Nodes Involved:**  
- Start Restore (Manual Trigger)  
- Init (Code)  
- List Bkp Folders (Execute Command)  
- Find Last Backup (Code)  
- ERROR: Find Most Recent Bkp Folder (Stop and Error)

**Node Details:**

- **Start Restore**  
  - Type: Manual Trigger  
  - Role: Entry point to start the restoration process manually.  
  - Configuration: No parameters; has a note reminding to configure JSON input for what to restore (credentials/workflows).  
  - Connections: Outputs to `Init`.  
  - Edge Cases: User may forget to configure JSON indicating what to restore.  
  - Notes: Labeled with "CRED 1 WF 0" indicating default to restore credentials only.

- **Init**  
  - Type: Code  
  - Role: Initializes environment variables, time and date formatting, directory paths, and custom configurations for backup locations.  
  - Configuration:  
    - Detects or sets the local timezone (default 'Europe/Paris').  
    - Defines project folders and log/report file paths.  
    - Defines backup folder from environment variable `N8N_BACKUP_FOLDER` or defaults to `/files/n8n-backups`.  
    - Sets temporary folder name for workflow restoration as `_restore_temp`.  
  - Key expressions: Uses environment variables and workflow name.  
  - Connections: Outputs config JSON to `List Bkp Folders`.  
  - Edge Cases: Environment variables missing or misconfigured could cause path errors.

- **List Bkp Folders**  
  - Type: Execute Command  
  - Role: Runs a shell command listing directories inside the backup folder.  
  - Configuration: Command `ls -1 {{ $json.customConfig.backupFolder }}`.  
  - Input: Receives config JSON from `Init`.  
  - Output: stdout with directory names fed to `Find Last Backup`.  
  - Edge Cases: If backup folder does not exist or is empty, it will cause an error downstream.

- **Find Last Backup**  
  - Type: Code  
  - Role: Parses directory list to find the latest backup directory by date, constructs full paths for credentials and workflows backups.  
  - Configuration: Expects directory names in `stdout` from previous node; sorts by date descending.  
  - Outputs: JSON including backup folder, latest backup date, full paths to credentials and workflows backup locations.  
  - On Error: Continues but triggers `ERROR: Find Most Recent Bkp Folder` node if fails.  
  - Edge Cases: No directories found, invalid date formats, missing or malformed input data.

- **ERROR: Find Most Recent Bkp Folder**  
  - Type: Stop and Error  
  - Role: Stops workflow and reports error if no valid backup folder found or if an error occurs in `Find Last Backup`.  
  - Configuration: Displays error message from input JSON.  
  - Edge Cases: Captures and fails gracefully on backup discovery errors.

---

#### 2.2 Restoration Execution

**Overview:**  
This block handles conditional restoration of credentials and workflows based on user input, excluding the currently running workflow from restoration to prevent conflicts, and cleans up temporary files. Success emails are sent upon completion.

**Nodes Involved:**  
- Restore Credentials? (If)  
- Restore Credentials (Execute Command)  
- SUCCESS email (Email Send)  
- Restore Workflows? (If)  
- Exclude Current Workflow From Selection (Execute Command)  
- Restore Workflows (Execute Command)  
- Delete TEMP Folder (Execute Command)  
- SUCCESS email Workflows (Email Send)

**Node Details:**

- **Restore Credentials?**  
  - Type: If  
  - Role: Checks if the user wants to restore credentials (`true` value in `Start Restore` JSON input).  
  - Condition: Boolean check on `$('Start Restore').item.json.credentials`.  
  - Connections: If true, flows to `Restore Credentials`; if false, skips.

- **Restore Credentials**  
  - Type: Execute Command  
  - Role: Runs n8n CLI command to import credentials from backup folder.  
  - Command: `n8n import:credentials --separate --input={{ $('Find Last Backup').item.json.credentialsFullBackupPath }}`  
  - Input: Uses path from `Find Last Backup` node.  
  - Connections: On success, triggers `SUCCESS email`.  
  - Edge Cases: CLI import failures, permission issues, corrupted backup files.

- **SUCCESS email**  
  - Type: Email Send (disabled by default)  
  - Role: Sends notification email after successful credentials restoration.  
  - Configuration:  
    - To: Admin email from environment variable `N8N_ADMIN_EMAIL`.  
    - Subject and body include exit codes, errors, and backup paths.  
  - Edge Cases: Email sending failures if SMTP not configured.

- **Restore Workflows?**  
  - Type: If  
  - Role: Checks if workflows restoration is enabled (`true` in `Start Restore` JSON).  
  - Connections: If true, flows to `Exclude Current Workflow From Selection`; else skips workflows restoration.

- **Exclude Current Workflow From Selection**  
  - Type: Execute Command  
  - Role: Runs a bash script to copy all workflow files except the current workflow into a temporary folder for restoration.  
  - Configuration:  
    - Uses environment variables and node outputs to determine source and destination paths.  
    - Cleans workflow filename to match backup file naming conventions.  
    - Skips copying the currently running workflow to avoid overwrite.  
  - Output: Logs summary of copied workflows.  
  - Connections: On success, flows to `Restore Workflows`.  
  - Edge Cases: No workflows to restore (only current workflow found), permission problems, path errors.

- **Restore Workflows**  
  - Type: Execute Command  
  - Role: Runs n8n CLI command to import workflows from the temporary folder created above.  
  - Command: `n8n import:workflow --separate --input={{workflowConfig.PROJECT_ROOT_PATH}}/{{latestBackupDate}}{{workflows_temp_folder}}`  
  - Connections: On success, flows to `Delete TEMP Folder`.  
  - Edge Cases: CLI import failures, corrupted workflow files.

- **Delete TEMP Folder**  
  - Type: Execute Command  
  - Role: Cleans up the temporary folder used for workflow restoration.  
  - Command: Bash script to remove temporary folder if it exists.  
  - Connections: On success, triggers `SUCCESS email Workflows`.  
  - Edge Cases: Folder already deleted, permission denied.

- **SUCCESS email Workflows**  
  - Type: Email Send (disabled by default)  
  - Role: Sends notification email after successful workflows restoration.  
  - Configuration:  
    - Similar to credentials' success email but includes detailed steps and logs related to workflows.  
  - Edge Cases: Email sending failures.

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                                  | Input Node(s)                          | Output Node(s)                             | Sticky Note                                         |
|----------------------------------|-----------------------|-------------------------------------------------|--------------------------------------|-------------------------------------------|-----------------------------------------------------|
| Start Restore                    | Manual Trigger        | Entry point to start restoration manually        | -                                    | Init                                      | CRED 1 WF 0; Configure JSON input to select restore options |
| Init                            | Code                  | Initializes env variables, time, paths            | Start Restore                       | List Bkp Folders                          | Instructions on timezone, folder paths, backup folder setup |
| List Bkp Folders                | Execute Command       | Lists backup directories in backup folder         | Init                               | Find Last Backup                          |                                                     |
| Find Last Backup                | Code                  | Finds the latest backup folder by date             | List Bkp Folders                   | Restore Credentials?, Restore Workflows?, ERROR node |                                                     |
| ERROR: Find Most Recent Bkp Folder | Stop and Error        | Stops workflow on backup discovery failure         | Find Last Backup                   | -                                         |                                                     |
| Restore Credentials?            | If                    | Condition to restore credentials                    | Find Last Backup                   | Restore Credentials                       |                                                     |
| Restore Credentials             | Execute Command       | Runs n8n CLI to import credentials                  | Restore Credentials?               | SUCCESS email                            |                                                     |
| SUCCESS email                  | Email Send             | Sends success email after credentials restore       | Restore Credentials               | -                                         | Disabled by default                                  |
| Restore Workflows?             | If                    | Condition to restore workflows                      | Find Last Backup                   | Exclude Current Workflow From Selection  |                                                     |
| Exclude Current Workflow From Selection | Execute Command       | Copies all workflows except current to temp folder | Restore Workflows?                | Restore Workflows                        |                                                     |
| Restore Workflows              | Execute Command       | Runs n8n CLI to import workflows                     | Exclude Current Workflow From Selection | Delete TEMP Folder                      |                                                     |
| Delete TEMP Folder             | Execute Command       | Deletes temporary folder after restore                | Restore Workflows                 | SUCCESS email Workflows                   |                                                     |
| SUCCESS email Workflows        | Email Send             | Sends success email after workflows restore          | Delete TEMP Folder                | -                                         | Disabled by default                                  |
| Sticky Note1                  | Sticky Note            | Label for "1. Restore Credentials from Disk"         | -                                | -                                         |                                                     |
| Sticky Note                   | Sticky Note            | Label for "2. Restore Workflows from Disk"            | -                                | -                                         |                                                     |
| Sticky Note2                  | Sticky Note            | Start here instructions and warnings for full restore | -                                | -                                         | Contains detailed warnings, instructions, and config code |
| Sticky Note3                  | Sticky Note            | Empty note                                            | -                                | -                                         |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `Start Restore`  
   - Purpose: Manual start of the workflow.  
   - Add note: "CRED 1 WF 0" to indicate default restore settings.  
   - Set pinData JSON to: `[{"credentials": true, "worflows": false}]` (adjust as needed).

2. **Create Code node**  
   - Name: `Init`  
   - Paste the provided JavaScript code that initializes timezones, paths, and backup folder variables.  
   - Use environment variables `N8N_BACKUP_FOLDER`, `N8N_ADMIN_EMAIL`, and `N8N_PROJECTS_DIR` as needed or defaults.  
   - Connect `Start Restore` to `Init`.

3. **Create Execute Command node**  
   - Name: `List Bkp Folders`  
   - Command: `ls -1 {{ $json.customConfig.backupFolder }}`  
   - Connect `Init` to `List Bkp Folders`.

4. **Create Code node**  
   - Name: `Find Last Backup`  
   - Paste provided JS code that parses directory list from `List Bkp Folders` and selects the latest backup date folder.  
   - Set error handling to continue on error (so workflow can branch).  
   - Connect `List Bkp Folders` to `Find Last Backup`.

5. **Create Stop and Error node**  
   - Name: `ERROR: Find Most Recent Bkp Folder`  
   - Configuration: Use error message from previous node's JSON.  
   - Connect error output of `Find Last Backup` to this node.

6. **Create If node**  
   - Name: `Restore Credentials?`  
   - Condition: Check if `$('Start Restore').item.json.credentials` is true (boolean).  
   - Connect success output of `Find Last Backup` to this node.

7. **Create Execute Command node**  
   - Name: `Restore Credentials`  
   - Command: `n8n import:credentials --separate --input={{ $('Find Last Backup').item.json.credentialsFullBackupPath }}`  
   - Connect true output of `Restore Credentials?` to this node.

8. **Create Email Send node (optional)**  
   - Name: `SUCCESS email`  
   - Disabled by default.  
   - Configure SMTP credentials, set "To" to `{{$env.N8N_ADMIN_EMAIL}}`.  
   - Subject and text include exit codes and backup paths from previous node.  
   - Connect `Restore Credentials` success output to this node.

9. **Create If node**  
   - Name: `Restore Workflows?`  
   - Condition: Check if `$('Start Restore').item.json.workflows` is true (boolean).  
   - Connect success output of `Find Last Backup` to this node.

10. **Create Execute Command node**  
    - Name: `Exclude Current Workflow From Selection`  
    - Paste provided bash script that copies all workflow JSON files except current workflow to a temporary folder.  
    - Connect true output of `Restore Workflows?` to this node.

11. **Create Execute Command node**  
    - Name: `Restore Workflows`  
    - Command: `n8n import:workflow --separate --input={{ $('Init').first().json.workflowConfig.PROJECT_ROOT_PATH }}/{{ $('Find Last Backup').first().json.latestBackupDate }}{{ $('Init').first().json.customConfig.workflows_temp_folder }}`  
    - Connect success output of `Exclude Current Workflow From Selection` to this node.

12. **Create Execute Command node**  
    - Name: `Delete TEMP Folder`  
    - Paste bash script to delete the temporary folder if it exists.  
    - Connect success output of `Restore Workflows` to this node.

13. **Create Email Send node (optional)**  
    - Name: `SUCCESS email Workflows`  
    - Disabled by default.  
    - Configure SMTP credentials, send to admin email with detailed logs.  
    - Connect success output of `Delete TEMP Folder` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                             | Context or Link                                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| ⚠ Warnings about restoring credentials first, that existing credentials or workflows will be overwritten, and instructions to manually copy workflow to new n8n instance before launching restoration.    | Sticky Note2 content at the start of the workflow                                                                                 |
| Can be used to move workflows between instances.                                                                                                                                                        | Sticky Note2                                                                                                                     |
| Configure the "Start Restore" JSON to choose whether to restore credentials and/or workflows.                                                                                                           | Sticky Note2                                                                                                                     |
| Sample code for timezone configuration and environment variable usage, emphasizing server timezone versus user-defined timezone.                                                                        | Init node code comments                                                                                                          |
| Disabled email notifications for success steps require SMTP credential configuration to enable.                                                                                                          | SUCCESS email and SUCCESS email Workflows nodes                                                                                   |
| The bash script to exclude the current workflow uses filename cleaning to avoid issues with special characters.                                                                                          | Exclude Current Workflow From Selection node                                                                                      |

---

**Disclaimer:**  
The provided text is extracted exclusively from an n8n automation workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.