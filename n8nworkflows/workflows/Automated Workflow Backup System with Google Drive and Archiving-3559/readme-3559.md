Automated Workflow Backup System with Google Drive and Archiving

https://n8nworkflows.xyz/workflows/automated-workflow-backup-system-with-google-drive-and-archiving-3559


# Automated Workflow Backup System with Google Drive and Archiving

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows to Google Drive on a scheduled basis. It is designed to:

- Regularly export all active workflows as JSON files into a specified Google Drive folder.
- Detect workflows tagged with the case-sensitive label `ARCHIVE` and back them up into a dedicated `Archive` subfolder.
- Automatically delete archived workflows from the n8n instance after backup, keeping the workspace clean.
- Allow customization of the backup schedule and Google Drive folder structure.

The workflow logic is organized into the following blocks:

- **1.1 Scheduled Trigger & Folder Preparation:** Initiates the backup process on a defined schedule and ensures the target Google Drive folder exists.
- **1.2 Workflow Retrieval & Branching:** Retrieves all workflows from n8n and routes them based on the presence of the `ARCHIVE` tag.
- **1.3 Backup Conversion & Upload:** Converts workflows to JSON files and uploads them to the appropriate Google Drive folder (main or Archive).
- **1.4 Cleanup of Archived Workflows:** Deletes workflows tagged `ARCHIVE` from the n8n instance after successful backup.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Folder Preparation

- **Overview:**  
  This block triggers the workflow on a schedule and prepares the Google Drive folder where backups will be stored.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Create to date folder

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow automatically based on a time schedule.  
    - Configuration: Default cron schedule (can be customized).  
    - Inputs: None (trigger node)  
    - Outputs: Triggers "Create to date folder" node.  
    - Edge Cases: Misconfiguration of schedule may cause no runs or too frequent runs.

  - **Create to date folder**  
    - Type: Google Drive  
    - Role: Ensures the target Google Drive folder exists or creates it for storing backups.  
    - Configuration: Uses Google Drive OAuth2 credentials; configured to create or confirm the main backup folder.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Passes control to "GET Workflows" node.  
    - Key Variables: Folder ID for main backup folder (must be set manually).  
    - Edge Cases: Google Drive API errors (auth failure, quota limits, invalid folder ID).

---

#### 2.2 Workflow Retrieval & Branching

- **Overview:**  
  Retrieves all workflows from the n8n instance and routes them based on whether they contain the `ARCHIVE` tag.

- **Nodes Involved:**  
  - GET Workflows  
  - If

- **Node Details:**

  - **GET Workflows**  
    - Type: n8n (internal)  
    - Role: Fetches all workflows currently present in the n8n instance.  
    - Configuration: Default, no parameters needed.  
    - Inputs: From "Create to date folder".  
    - Outputs: Passes workflows to "If" node.  
    - Edge Cases: API errors, empty workflow list.

  - **If**  
    - Type: If  
    - Role: Checks each workflow for the presence of the `ARCHIVE` tag (case-sensitive).  
    - Configuration: Conditional logic to separate workflows tagged `ARCHIVE` from others.  
    - Inputs: Workflows from "GET Workflows".  
    - Outputs: Two branches:  
      - True branch: workflows tagged `ARCHIVE` → "Convert to JSON'"  
      - False branch: other workflows → "Convert to JSON"  
    - Key Expressions: Checks workflow tags array for exact match to `ARCHIVE`.  
    - Edge Cases: Missing or malformed tags, case sensitivity issues.

---

#### 2.3 Backup Conversion & Upload

- **Overview:**  
  Converts workflows into JSON files and uploads them to Google Drive in the appropriate folder.

- **Nodes Involved:**  
  - Convert to JSON  
  - Convert to JSON'  
  - Save all other Workflows  
  - Save 'ARCHIVE' Workflows

- **Node Details:**

  - **Convert to JSON**  
    - Type: Convert to File  
    - Role: Converts workflows (non-archived) into `.json` files for backup.  
    - Configuration: Outputs JSON file with workflow data.  
    - Inputs: From "If" node (false branch).  
    - Outputs: Passes files to "Save all other Workflows".  
    - Edge Cases: Conversion errors if workflow data is malformed.

  - **Convert to JSON'**  
    - Type: Convert to File  
    - Role: Converts workflows tagged `ARCHIVE` into `.json` files.  
    - Configuration: Same as above, but for archived workflows.  
    - Inputs: From "If" node (true branch).  
    - Outputs: Passes files to "Save 'ARCHIVE' Workflows".  
    - Edge Cases: Same as above.

  - **Save all other Workflows**  
    - Type: Google Drive  
    - Role: Uploads JSON files of non-archived workflows to the main backup folder in Google Drive.  
    - Configuration: Uses Google Drive OAuth2 credentials; target folder ID set to main backup folder.  
    - Inputs: From "Convert to JSON".  
    - Outputs: End of branch.  
    - Edge Cases: Google Drive upload errors, insufficient permissions.

  - **Save 'ARCHIVE' Workflows**  
    - Type: Google Drive  
    - Role: Uploads JSON files of archived workflows to the `Archive` subfolder in Google Drive.  
    - Configuration: Uses Google Drive OAuth2 credentials; target folder ID set to Archive folder.  
    - Inputs: From "Convert to JSON'".  
    - Outputs: Passes to "Delete 'ARCHIVE' Workflows".  
    - Edge Cases: Same as above.

---

#### 2.4 Cleanup of Archived Workflows

- **Overview:**  
  Deletes workflows tagged `ARCHIVE` from the n8n instance after successful backup to the Archive folder.

- **Nodes Involved:**  
  - Delete 'ARCHIVE' Workflows

- **Node Details:**

  - **Delete 'ARCHIVE' Workflows**  
    - Type: n8n (internal)  
    - Role: Deletes archived workflows from the n8n instance to reduce clutter.  
    - Configuration: Uses workflow IDs from previous nodes to delete workflows.  
    - Inputs: From "Save 'ARCHIVE' Workflows".  
    - Outputs: End of workflow.  
    - Edge Cases: API errors, permission issues, accidental deletion if tagging is incorrect.

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                              | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                      |
|--------------------------|-----------------------|----------------------------------------------|-------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger      | Initiates workflow on schedule                | None                    | Create to date folder        |                                                                                                 |
| Create to date folder    | Google Drive          | Ensures main backup folder exists             | Schedule Trigger        | GET Workflows                |                                                                                                 |
| GET Workflows           | n8n                   | Retrieves all workflows from n8n instance     | Create to date folder   | If                          |                                                                                                 |
| If                      | If                    | Routes workflows based on `ARCHIVE` tag       | GET Workflows           | Convert to JSON', Convert to JSON |                                                                                                 |
| Convert to JSON'         | Convert to File       | Converts archived workflows to JSON files     | If (true branch)        | Save 'ARCHIVE' Workflows     |                                                                                                 |
| Convert to JSON          | Convert to File       | Converts non-archived workflows to JSON files | If (false branch)       | Save all other Workflows     |                                                                                                 |
| Save 'ARCHIVE' Workflows | Google Drive          | Uploads archived workflows to Archive folder  | Convert to JSON'        | Delete 'ARCHIVE' Workflows   |                                                                                                 |
| Save all other Workflows | Google Drive          | Uploads non-archived workflows to main folder | Convert to JSON         | None                        |                                                                                                 |
| Delete 'ARCHIVE' Workflows| n8n                   | Deletes archived workflows from n8n instance  | Save 'ARCHIVE' Workflows| None                        |                                                                                                 |
| Sticky Note9             | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note2             | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note              | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note3             | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note1             | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note4             | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note5             | Sticky Note           |                                               |                         |                             |                                                                                                 |
| Sticky Note6             | Sticky Note           |                                               |                         |                             |                                                                                                 |

(Note: Sticky notes have no content in the provided JSON, so their cells are left blank.)

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it**: "Automated Workflow Backup System with Google Drive and Archiving".

2. **Add a Schedule Trigger node**:  
   - Type: Schedule Trigger  
   - Configure the schedule as desired (default runs periodically, e.g., daily).  
   - No credentials needed.

3. **Add a Google Drive node named "Create to date folder"**:  
   - Operation: Create or ensure folder exists (use "Create Folder" or "Search Folder" operation).  
   - Credentials: Connect your Google Drive OAuth2 credentials.  
   - Parameters: Set the folder name to your main backup folder name or use the folder ID directly.  
   - Connect the Schedule Trigger output to this node.

4. **Add an n8n node named "GET Workflows"**:  
   - Operation: Use the internal n8n API to fetch all workflows.  
   - Connect the output of "Create to date folder" to this node.

5. **Add an If node named "If"**:  
   - Condition: Check if the workflow's tags array contains the exact string `ARCHIVE` (case-sensitive).  
   - Connect the output of "GET Workflows" to this node.

6. **Add two Convert to File nodes**:  
   - One named "Convert to JSON" for non-archived workflows (false branch).  
   - One named "Convert to JSON'" for archived workflows (true branch).  
   - Configure both to convert workflow data to JSON files.  
   - Connect the respective outputs of the If node to these nodes.

7. **Add two Google Drive nodes**:  
   - "Save all other Workflows": Uploads JSON files to the main backup folder.  
     - Credentials: Google Drive OAuth2.  
     - Folder ID: Set to main backup folder ID.  
     - Connect from "Convert to JSON".  
   - "Save 'ARCHIVE' Workflows": Uploads JSON files to the Archive subfolder.  
     - Credentials: Google Drive OAuth2.  
     - Folder ID: Set to Archive subfolder ID inside the main folder.  
     - Connect from "Convert to JSON'".

8. **Add an n8n node named "Delete 'ARCHIVE' Workflows"**:  
   - Operation: Delete workflows by ID from the n8n instance.  
   - Connect from "Save 'ARCHIVE' Workflows".  
   - Ensure it uses the workflow IDs from the archived workflows to delete them.

9. **Set up credentials**:  
   - Google Drive OAuth2 credentials must be created and connected in n8n.  
   - Ensure n8n has API access to manage workflows.

10. **Set folder IDs**:  
    - Retrieve folder IDs from Google Drive URLs for both main and Archive folders.  
    - Insert these IDs in the respective Google Drive nodes.

11. **Test the workflow**:  
    - Run manually to verify workflows are backed up correctly.  
    - Check Google Drive folders for JSON files.  
    - Verify that workflows tagged `ARCHIVE` are deleted after backup.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| If you have any question, or difficulty, feel free to come discuss about it on my Telegram (you might find something there 🎁) | https://www.bonzai.pro/niten-musa/lp/7018/niten-musa                                                |
| Folder IDs can be found in the Google Drive folder URL, e.g., https://drive.google.com/drive/u/0/folders/**folderID** | Important for configuring Google Drive nodes                                                       |
| Workflows are archived only if tagged exactly with `ARCHIVE` (case-sensitive)                         | Tagging convention critical for workflow branching logic                                           |
| Customize the schedule trigger to run backups daily, weekly, or at any interval                      | Use the built-in Cron node options in n8n                                                          |
| Ensure Google Drive OAuth2 credentials have sufficient permissions to create folders and upload files | Credential setup is essential for workflow operation                                               |

---

This document fully describes the workflow’s structure, logic, and setup, enabling both human users and automation agents to understand, reproduce, and maintain the backup system effectively.