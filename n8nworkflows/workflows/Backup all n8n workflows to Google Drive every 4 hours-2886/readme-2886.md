Backup all n8n workflows to Google Drive every 4 hours

https://n8nworkflows.xyz/workflows/backup-all-n8n-workflows-to-google-drive-every-4-hours-2886


# Backup all n8n workflows to Google Drive every 4 hours

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows to Google Drive every 4 hours, replacing the need for GitHub backups. It ensures that workflow JSON files are safely stored in a timestamped folder on Google Drive and manages storage by deleting previous backup folders. The workflow is structured into three main logical blocks:

- **1.1 Trigger and Folder Creation:** Initiates the backup process on a schedule or manual trigger and creates a new timestamped folder in a specified Google Drive parent folder.
- **1.2 Workflow Export and Upload:** Retrieves all n8n workflows, converts each to a JSON file, and uploads them into the newly created Google Drive folder.
- **1.3 Cleanup of Old Backups:** Lists existing backup folders, filters out the current backup folder, and deletes all previous backup folders to avoid clutter.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Folder Creation

- **Overview:**  
  This block starts the workflow either manually or on a schedule (every 4 hours) and creates a new Google Drive folder named with the current date and time inside a specified parent folder.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger  
  - create new folder (Google Drive Folder Creation)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow for testing or immediate backup.  
    - Configuration: No parameters; triggers workflow on user click.  
    - Inputs: None  
    - Outputs: Triggers "create new folder" node.  
    - Edge cases: None significant; manual trigger depends on user action.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every 4 hours.  
    - Configuration: Interval set to 4 hours.  
    - Inputs: None  
    - Outputs: Triggers "create new folder" node.  
    - Edge cases: Potential timing drift or missed triggers if n8n instance is down.

  - **create new folder**  
    - Type: Google Drive node (folder creation)  
    - Role: Creates a new folder in Google Drive inside a specified parent folder to store the backup files.  
    - Configuration:  
      - Folder name dynamically generated using the current date and time formatted as "Workflow Backups [weekday] [time] [dd-MM-yyyy]" (e.g., "Workflow Backups Monday 10:00 AM 01-01-2024").  
      - Parent folder ID is fixed (configured to a specific Google Drive folder).  
      - Drive ID set to "My Drive".  
    - Inputs: Triggered by either manual or schedule trigger.  
    - Outputs: Passes newly created folder metadata (including folder ID) downstream.  
    - Edge cases:  
      - Google Drive API quota limits or permission errors if the service account lacks access.  
      - Folder creation failure if the parent folder ID is invalid or deleted.  
    - Credentials: Google Drive OAuth2 service account with access to the parent folder.

---

#### 1.2 Workflow Export and Upload

- **Overview:**  
  This block retrieves all existing n8n workflows, iterates over them, converts each workflow to a JSON file, and uploads each file into the newly created Google Drive folder.

- **Nodes Involved:**  
  - n8n (n8n API node)  
  - Loop Over Items (Split in Batches)  
  - Convert to File  
  - Google Drive (File Upload)

- **Node Details:**

  - **n8n**  
    - Type: n8n API node  
    - Role: Fetches all workflows from the n8n instance via API.  
    - Configuration: No filters applied; retrieves all workflows.  
    - Inputs: Receives trigger from "create new folder" node.  
    - Outputs: List of workflows as JSON objects.  
    - Credentials: Uses n8n API credentials with appropriate permissions.  
    - Edge cases:  
      - API authentication failure.  
      - Network or timeout errors.  
      - Empty workflow list if no workflows exist.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes workflows one by one to avoid overloading downstream nodes.  
    - Configuration: Default batch size (not explicitly set, defaults to 1).  
    - Inputs: Receives workflows array from the n8n node.  
    - Outputs: Sends each workflow individually to downstream nodes.  
    - Edge cases: Large number of workflows may increase execution time.

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts each workflow JSON object into a JSON file format suitable for upload.  
    - Configuration:  
      - Format: JSON  
      - Filename: Uses the workflow's name property with ".json" extension (e.g., "MyWorkflow.json").  
    - Inputs: Single workflow JSON from "Loop Over Items".  
    - Outputs: File data object for upload.  
    - Edge cases:  
      - Workflow name missing or containing invalid filename characters (may cause upload issues).  
      - Conversion errors if workflow JSON is malformed.

  - **Google Drive (File Upload)**  
    - Type: Google Drive node (file upload)  
    - Role: Uploads the JSON file to the newly created backup folder on Google Drive.  
    - Configuration:  
      - File name: Derived from the workflow name with ".json" extension.  
      - Folder ID: Dynamically set to the ID of the folder created in "create new folder".  
      - Drive ID: "My Drive".  
    - Inputs: Receives file from "Convert to File".  
    - Outputs: Passes upload result downstream (connected back to "Loop Over Items" to continue batch processing).  
    - Credentials: Google Drive OAuth2 service account.  
    - Edge cases:  
      - Upload failure due to permission issues or quota limits.  
      - Filename conflicts (Google Drive allows duplicates, but may cause confusion).  
      - Network interruptions.

---

#### 1.3 Cleanup of Old Backups

- **Overview:**  
  This block lists all existing backup folders in the parent Google Drive folder, filters out the current backup folder, and deletes all other folders to maintain only the latest backup.

- **Nodes Involved:**  
  - Loop Over Items (second output)  
  - Get folders (Google Drive list)  
  - Filter  
  - delete folder (Google Drive delete)

- **Node Details:**

  - **Loop Over Items (second output)**  
    - Role: After processing all workflows, triggers the cleanup process.  
    - Outputs: Triggers "Get folders" node.

  - **Get folders**  
    - Type: Google Drive node (list files/folders)  
    - Role: Retrieves all folders inside the specified parent backup folder on Google Drive.  
    - Configuration:  
      - Filter: Folder ID set to the parent backup folder ID.  
      - Resource: fileFolder (to get both files and folders, but filtered to folders).  
    - Inputs: Triggered by "Loop Over Items".  
    - Outputs: List of backup folders.  
    - Credentials: Google Drive OAuth2 service account.  
    - Edge cases:  
      - API quota limits.  
      - Permission errors if service account access revoked.

  - **Filter**  
    - Type: Filter node  
    - Role: Filters out the current backup folder from the list to avoid deleting it.  
    - Configuration:  
      - Condition: Folder ID not equal to the current backup folder ID (from "create new folder").  
      - Case insensitive.  
    - Inputs: Receives folder list from "Get folders".  
    - Outputs: Passes folders to delete downstream.  
    - Edge cases:  
      - If current folder ID is missing or incorrect, may delete all folders including current backup.

  - **delete folder**  
    - Type: Google Drive node (delete folder)  
    - Role: Permanently deletes each folder filtered for removal.  
    - Configuration:  
      - Operation: Delete folder permanently.  
      - Folder ID: Dynamically set from filtered folder IDs.  
    - Inputs: Receives folders to delete from "Filter".  
    - Outputs: None (end of workflow).  
    - Credentials: Google Drive OAuth2 service account.  
    - Edge cases:  
      - Deletion failure due to permission issues or folder being in use.  
      - API quota limits.  
      - Irreversible deletion (no recycle bin).

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                   |
|---------------------|-------------------------|-------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger          | Manual start of backup workflow      | None                   | create new folder       |                                                                                              |
| Schedule Trigger     | Schedule Trigger        | Automatic start every 4 hours        | None                   | create new folder       |                                                                                              |
| create new folder    | Google Drive (folder)   | Creates timestamped backup folder    | On clicking 'execute', Schedule Trigger | n8n                    |                                                                                              |
| n8n                 | n8n API                 | Retrieves all workflows               | create new folder       | Loop Over Items         |                                                                                              |
| Loop Over Items      | SplitInBatches          | Processes workflows one by one        | n8n                     | Get folders (2nd output), Convert to File (1st output) |                                                                                              |
| Convert to File      | ConvertToFile           | Converts workflow JSON to file        | Loop Over Items         | Google Drive (file upload) |                                                                                              |
| Google Drive        | Google Drive (file)      | Uploads workflow JSON file            | Convert to File         | Loop Over Items         |                                                                                              |
| Get folders          | Google Drive (list)     | Lists all backup folders              | Loop Over Items (2nd output) | Filter                  |                                                                                              |
| Filter               | Filter                  | Filters out current backup folder    | Get folders             | delete folder           |                                                                                              |
| delete folder        | Google Drive (delete)   | Deletes old backup folders            | Filter                  | None                    |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named "On clicking 'execute'".
   - Add a **Schedule Trigger** node named "Schedule Trigger" with interval set to every 4 hours.

2. **Create Folder Node:**
   - Add a **Google Drive** node named "create new folder".
   - Set operation to create a folder.
   - Set folder name to expression:  
     `Workflow Backups {{ $now.format('cccc t dd-MM-yyyy') }}`  
   - Set parent folder ID to your designated Google Drive folder ID (e.g., "1hnHubRgcstU8OgV8BPwPNivfTZT5g2Wf").
   - Set Drive ID to "My Drive".
   - Connect both trigger nodes ("On clicking 'execute'" and "Schedule Trigger") to this node.
   - Configure Google Drive OAuth2 credentials with a service account that has access to the parent folder.

3. **Retrieve Workflows:**
   - Add an **n8n API** node named "n8n".
   - Configure it to list all workflows (no filters).
   - Connect "create new folder" node output to this node.
   - Use n8n API credentials with sufficient permissions.

4. **Split Workflows for Processing:**
   - Add a **SplitInBatches** node named "Loop Over Items".
   - Connect "n8n" node output to this node.
   - Default batch size is fine (process one workflow at a time).

5. **Convert Workflow to JSON File:**
   - Add a **ConvertToFile** node named "Convert to File".
   - Set operation to "toJson".
   - Set file name to expression: `={{ $json.name + ".json" }}`.
   - Connect "Loop Over Items" first output to this node.

6. **Upload JSON File to Google Drive:**
   - Add a **Google Drive** node named "Google Drive".
   - Set operation to upload a file.
   - Set file name to expression: `={{ $('Loop Over Items').item.json.name + ".json" }}`.
   - Set folder ID to expression: `={{ $('create new folder').item.json.id }}` (the newly created folder).
   - Set Drive ID to "My Drive".
   - Connect "Convert to File" node to this node.
   - Connect this node back to "Loop Over Items" to continue batch processing.
   - Use the same Google Drive OAuth2 credentials.

7. **Cleanup Old Backups:**
   - Connect the second output of "Loop Over Items" (trigger after all workflows processed) to a new **Google Drive** node named "Get folders".
   - Configure "Get folders" to list files/folders.
   - Set filter to list folders inside the parent backup folder ID (same as in step 2).
   - Use the same Google Drive credentials.

8. **Filter Current Backup Folder:**
   - Add a **Filter** node named "Filter".
   - Connect "Get folders" output to "Filter".
   - Set condition: Folder ID not equal to `={{ $('create new folder').item.json.id }}`.
   - Case insensitive.

9. **Delete Old Backup Folders:**
   - Add a **Google Drive** node named "delete folder".
   - Set operation to delete folder permanently.
   - Set folder ID to `={{ $json.id }}` (from filtered folders).
   - Connect "Filter" output to this node.
   - Use the same Google Drive credentials.

10. **Activate Workflow:**
    - Save and activate the workflow.
    - Ensure the Google Drive service account has been shared access to the parent folder.
    - Test manually using "On clicking 'execute'" or wait for scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow replaces GitHub backups with Google Drive backups for n8n workflows.                  | Workflow description and purpose.                                                               |
| Share your Google Drive backup folder with the service account email to grant access.               | Setup step 3 in the workflow description.                                                       |
| Author: Zacharia Kimotho (LinkedIn: https://www.linkedin.com/in/zacharia-kimotho/)                  | Credits and contact.                                                                             |
| The workflow deletes old backup folders to keep only the latest backup, ensuring clean storage.    | Important operational detail to avoid storage clutter.                                          |
| Google Drive API quotas and permissions are critical; ensure service account has proper access.     | Potential failure points in folder creation, file upload, and deletion.                          |
| Workflow uses n8n API node with retry enabled to improve reliability when fetching workflows.       | Helps mitigate transient API failures.                                                          |
| Folder naming uses current date and time formatted as "Workflow Backups [weekday] [time] [date]".  | Helps identify backups easily by timestamp.                                                     |

---

This documentation enables full understanding, reproduction, and maintenance of the "Backup all n8n workflows to Google Drive every 4 hours" workflow.