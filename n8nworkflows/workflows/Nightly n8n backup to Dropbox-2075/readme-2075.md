Nightly n8n backup to Dropbox

https://n8nworkflows.xyz/workflows/nightly-n8n-backup-to-dropbox-2075


# Nightly n8n backup to Dropbox

### 1. Workflow Overview

This workflow automates the nightly backup of all n8n workflows to a specified Dropbox folder. Its primary purpose is to maintain an organized backup system by moving previous backups to an "old" folder with date-stamped filenames, uploading current workflow JSON files, and optionally purging backups older than a configurable number of days.

Logical blocks:

- **1.1 Schedule and Initialization:** Triggers the workflow at a scheduled time and sets the destination folder path.
- **1.2 Backup Rotation:** Lists current backups in Dropbox, filters files, and moves them to an "old" folder with a date suffix.
- **1.3 Workflow Export and Upload:** Retrieves all current n8n workflows, converts them to JSON files, and uploads them to Dropbox.
- **1.4 Purging Old Backups (optional):** Lists backups in the "old" folder, checks their age against the purge threshold, and deletes those exceeding it.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule and Initialization

- **Overview:**  
  This block schedules the nightly backup execution and defines the Dropbox destination folder for storing backups.

- **Nodes Involved:**  
  - Schedule Trigger  
  - DESTINATION FOLDER  
  - GET CURRENT DATE

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a defined schedule (default daily).  
    - Configuration: Uses default schedule with interval set to daily (can be customized).  
    - Inputs: None (start node)  
    - Outputs: Connects to DESTINATION FOLDER  
    - Edge Cases: Misconfigured schedule may cause no triggers; time zone considerations may affect timing.

  - **DESTINATION FOLDER**  
    - Type: Set  
    - Role: Defines the Dropbox folder path where backups will be stored.  
    - Configuration: Sets a string field named `folder` with a value like `/n8n_backups/` (must include trailing slash).  
    - Inputs: From Schedule Trigger  
    - Outputs: Connects to GET CURRENT DATE, WAIT FOR MOVE TO FINISH, and PURGE DAYS nodes  
    - Edge Cases: Incorrect folder path or missing trailing slash may cause upload failures.

  - **GET CURRENT DATE**  
    - Type: DateTime  
    - Role: Obtains the current date/time formatted as `yyyy-MM-dd_HHmm` for timestamping backup files.  
    - Configuration: Format set to `=yyyy-MM-dd_HHmm`, uses current time `$now`.  
    - Inputs: From DESTINATION FOLDER  
    - Outputs: Connects to GET CURRENT BACKUPS  
    - Edge Cases: Date formatting issues unlikely but possible if time zone is inconsistent.

---

#### 1.2 Backup Rotation

- **Overview:**  
  This block manages existing backups by listing current backup files, filtering out folders, and moving files to an "old" subfolder with a date suffix appended to the filenames.

- **Nodes Involved:**  
  - GET CURRENT BACKUPS  
  - IGNORE FOLDERS  
  - MOVE INTO OLD FOLDER  
  - WAIT FOR MOVE TO FINISH  
  - Sticky Note ("MOVE CURRENT BACKUPS TO OLD FOLDER")

- **Node Details:**

  - **GET CURRENT BACKUPS**  
    - Type: Dropbox  
    - Role: Lists contents of the destination Dropbox folder to retrieve current backups.  
    - Configuration: List operation on path from DESTINATION FOLDER's `folder` property, limit 250 items.  
    - Authentication: Dropbox OAuth2  
    - Inputs: From GET CURRENT DATE  
    - Outputs: Connects to IGNORE FOLDERS  
    - Edge Cases: Dropbox API rate limits, folder not existing, or auth token expiration.

  - **IGNORE FOLDERS**  
    - Type: Filter  
    - Role: Filters out folder entries, passing only files for processing.  
    - Configuration: Filters where `type` is not "folder".  
    - Inputs: From GET CURRENT BACKUPS  
    - Outputs: Connects to MOVE INTO OLD FOLDER  
    - Edge Cases: Unexpected types or missing `type` field in Dropbox response.

  - **MOVE INTO OLD FOLDER**  
    - Type: Dropbox  
    - Role: Moves each current backup file into the `old` subfolder, renaming with the current date suffix.  
    - Configuration: Move operation using file’s `pathDisplay`, destination path constructed as `<destination_folder>old/<filename>_<current_date>.json`.  
    - Authentication: Dropbox OAuth2  
    - Inputs: From IGNORE FOLDERS  
    - Outputs: Connects to WAIT FOR MOVE TO FINISH  
    - Execution: Runs for each file; onError set to continue to avoid halting on individual failures.  
    - Edge Cases: Move conflicts, permissions issues, file locks.

  - **WAIT FOR MOVE TO FINISH**  
    - Type: Merge  
    - Role: Waits for all move operations to complete before proceeding.  
    - Configuration: Mode set to "chooseBranch", output set to input2 (i.e., waits for MOVE INTO OLD FOLDER to finish).  
    - Inputs: From MOVE INTO OLD FOLDER and DESTINATION FOLDER (parallel branch)  
    - Outputs: Connects to GET WORKFLOWS  
    - Edge Cases: Potential synchronization issues if moves are slow or API calls fail.

  - **Sticky Note ("MOVE CURRENT BACKUPS TO OLD FOLDER")**  
    - Role: Describes the logic of this block for clarity.

---

#### 1.3 Workflow Export and Upload

- **Overview:**  
  This block exports all active n8n workflows as JSON files, converts them into binary format for upload, and uploads them to the specified Dropbox folder.

- **Nodes Involved:**  
  - GET WORKFLOWS  
  - MAKE JSON FILES  
  - UPLOAD WORKFLOWS  
  - Sticky Note1 ("BACKUP ALL CURRENT WORKFLOWS")

- **Node Details:**

  - **GET WORKFLOWS**  
    - Type: n8n  
    - Role: Retrieves all workflows from the n8n instance via internal API.  
    - Configuration: No filters applied, fetches all workflows.  
    - Credentials: n8n API credentials required  
    - Inputs: From WAIT FOR MOVE TO FINISH  
    - Outputs: Connects to MAKE JSON FILES  
    - Edge Cases: API connection failures, permission issues.

  - **MAKE JSON FILES**  
    - Type: Move Binary Data  
    - Role: Converts JSON workflow data into binary data format with a filename matching the workflow’s name for upload.  
    - Configuration: Mode set to `jsonToBinary`, filename set dynamically from workflow name.  
    - Inputs: From GET WORKFLOWS  
    - Outputs: Connects to UPLOAD WORKFLOWS  
    - Edge Cases: Workflow names with invalid filename characters may cause upload errors.

  - **UPLOAD WORKFLOWS**  
    - Type: Dropbox  
    - Role: Uploads the binary workflow JSON files to the Dropbox destination folder.  
    - Configuration: Upload operation with binary data input, path set dynamically as `<destination_folder><workflow_name>.json`.  
    - Authentication: Dropbox OAuth2  
    - Inputs: From MAKE JSON FILES  
    - Outputs: None (end of this branch)  
    - Edge Cases: Dropbox API limits, file overwrite conflicts, invalid paths.

  - **Sticky Note1 ("BACKUP ALL CURRENT WORKFLOWS")**  
    - Role: Describes the export and upload process.

---

#### 1.4 Purging Old Backups (Optional)

- **Overview:**  
  This optional block purges backups older than a specified number of days from the `old` folder to save storage and maintain hygiene.

- **Nodes Involved:**  
  - PURGE DAYS  
  - LIST OLD BACKUPS  
  - CHECK DATES  
  - DELETE OLD BACKUPS  
  - Sticky Note2 ("PURGE BACKUPS OLDER THEN 30 DAYS")

- **Node Details:**

  - **PURGE DAYS**  
    - Type: DateTime  
    - Role: Calculates the cutoff date by subtracting the purge days (default 30) from current date/time.  
    - Configuration: Operation set to subtract from `$now`, magnitude 30 (days).  
    - Inputs: From DESTINATION FOLDER  
    - Outputs: Connects to LIST OLD BACKUPS  
    - Edge Cases: Incorrect purge days value or time zone issues.

  - **LIST OLD BACKUPS**  
    - Type: Dropbox  
    - Role: Lists files inside the `old` subfolder for evaluation.  
    - Configuration: List operation on path `<destination_folder>old`, limit 500.  
    - Authentication: Dropbox OAuth2  
    - Inputs: From PURGE DAYS  
    - Outputs: Connects to CHECK DATES  
    - Edge Cases: Folder may not exist, Dropbox API limits.

  - **CHECK DATES**  
    - Type: If  
    - Role: Checks if each file’s last modified date is older than the purge cutoff date.  
    - Configuration: Condition compares file's `lastModifiedServer` with `PURGE DAYS` calculated date, passes files older than cutoff.  
    - Inputs: From LIST OLD BACKUPS  
    - Outputs: True branch to DELETE OLD BACKUPS (files to delete), False branch ignored.  
    - Edge Cases: Missing or malformed date fields, time zone differences.

  - **DELETE OLD BACKUPS**  
    - Type: Dropbox  
    - Role: Deletes files identified as older than the purge threshold.  
    - Configuration: Delete operation on file path from input JSON.  
    - Authentication: Dropbox OAuth2  
    - Inputs: From CHECK DATES (true branch)  
    - Outputs: None (end of purge branch)  
    - Edge Cases: Deletion failures, permission errors, file lock conflicts.

  - **Sticky Note2 ("PURGE BACKUPS OLDER THEN 30 DAYS")**  
    - Role: Explains this block’s purpose and default disabled status for safety.

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                           | Input Node(s)             | Output Node(s)              | Sticky Note                                                       |
|-----------------------|----------------------|-----------------------------------------|---------------------------|-----------------------------|------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger     | Starts workflow nightly                 | —                         | DESTINATION FOLDER           |                                                                  |
| DESTINATION FOLDER     | Set                  | Defines Dropbox destination folder path | Schedule Trigger           | GET CURRENT DATE, WAIT FOR MOVE TO FINISH, PURGE DAYS |                                                                  |
| GET CURRENT DATE       | DateTime             | Gets current timestamp for filenames    | DESTINATION FOLDER         | GET CURRENT BACKUPS          |                                                                  |
| GET CURRENT BACKUPS    | Dropbox              | Lists current backup files               | GET CURRENT DATE           | IGNORE FOLDERS               |                                                                  |
| IGNORE FOLDERS         | Filter               | Filters out folders from backup listing | GET CURRENT BACKUPS        | MOVE INTO OLD FOLDER         |                                                                  |
| MOVE INTO OLD FOLDER   | Dropbox              | Moves current backups to 'old' folder with date suffix | IGNORE FOLDERS             | WAIT FOR MOVE TO FINISH      |                                                                  |
| WAIT FOR MOVE TO FINISH| Merge                | Waits for move operations to complete   | MOVE INTO OLD FOLDER, DESTINATION FOLDER | GET WORKFLOWS               |                                                                  |
| GET WORKFLOWS          | n8n API              | Retrieves all current workflows          | WAIT FOR MOVE TO FINISH    | MAKE JSON FILES              |                                                                  |
| MAKE JSON FILES        | Move Binary Data     | Converts workflows to binary JSON files  | GET WORKFLOWS              | UPLOAD WORKFLOWS             |                                                                  |
| UPLOAD WORKFLOWS       | Dropbox              | Uploads workflow JSON files to Dropbox  | MAKE JSON FILES            | —                           |                                                                  |
| PURGE DAYS             | DateTime             | Calculates purge cutoff date             | DESTINATION FOLDER         | LIST OLD BACKUPS             | Enabled and configured to set backup retention period           |
| LIST OLD BACKUPS       | Dropbox              | Lists backups in 'old' folder            | PURGE DAYS                 | CHECK DATES                 |                                                                  |
| CHECK DATES            | If                   | Checks if backup is older than purge date | LIST OLD BACKUPS           | DELETE OLD BACKUPS           |                                                                  |
| DELETE OLD BACKUPS     | Dropbox              | Deletes backups older than purge date    | CHECK DATES (true branch)  | —                           |                                                                  |
| Sticky Note            | Sticky Note          | "MOVE CURRENT BACKUPS TO OLD FOLDER"     | —                         | —                           | Covers backup rotation block                                    |
| Sticky Note1           | Sticky Note          | "BACKUP ALL CURRENT WORKFLOWS"            | —                         | —                           | Covers workflow export and upload block                          |
| Sticky Note2           | Sticky Note          | "PURGE BACKUPS OLDER THEN 30 DAYS"        | —                         | —                           | Covers purge old backups block, disabled by default for safety  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Node Type: Schedule Trigger  
   - Configure trigger interval to daily or desired frequency.  

2. **Create Set Node: DESTINATION FOLDER**  
   - Node Type: Set  
   - Add a field `folder` of type string with value `/n8n_backups/` (include trailing slash).  
   - Connect Schedule Trigger → DESTINATION FOLDER.

3. **Create DateTime Node: GET CURRENT DATE**  
   - Node Type: DateTime  
   - Operation: Format Date  
   - Date: `={{ $now }}`  
   - Format: `=yyyy-MM-dd_HHmm`  
   - Connect DESTINATION FOLDER → GET CURRENT DATE.

4. **Create Dropbox Node: GET CURRENT BACKUPS**  
   - Node Type: Dropbox  
   - Operation: List Folder  
   - Path: `={{ $('DESTINATION FOLDER').last().json.folder }}`  
   - Limit: 250  
   - Authentication: Dropbox OAuth2 (set up credentials beforehand)  
   - Connect GET CURRENT DATE → GET CURRENT BACKUPS.

5. **Create Filter Node: IGNORE FOLDERS**  
   - Node Type: Filter  
   - Condition: `$json.type != 'folder'` (string not equals)  
   - Connect GET CURRENT BACKUPS → IGNORE FOLDERS.

6. **Create Dropbox Node: MOVE INTO OLD FOLDER**  
   - Node Type: Dropbox  
   - Operation: Move  
   - Path: `={{ $json.pathDisplay }}`  
   - To Path: `={{ $('DESTINATION FOLDER').last().json.folder }}old/{{ $json.name }}_{{ $('GET CURRENT DATE').last().json.formattedDate }}.json`  
   - Authentication: Dropbox OAuth2  
   - Error Handling: Continue on error  
   - Connect IGNORE FOLDERS → MOVE INTO OLD FOLDER.

7. **Create Merge Node: WAIT FOR MOVE TO FINISH**  
   - Node Type: Merge  
   - Mode: Choose Branch  
   - Output: Input2 (waits for MOVE INTO OLD FOLDER to finish)  
   - Connect MOVE INTO OLD FOLDER → WAIT FOR MOVE TO FINISH (input 1)  
   - Connect DESTINATION FOLDER → WAIT FOR MOVE TO FINISH (input 2).

8. **Create n8n Node: GET WORKFLOWS**  
   - Node Type: n8n  
   - Filters: None (retrieve all workflows)  
   - Credentials: n8n API credentials configured  
   - Connect WAIT FOR MOVE TO FINISH → GET WORKFLOWS.

9. **Create Move Binary Data Node: MAKE JSON FILES**  
   - Node Type: Move Binary Data  
   - Mode: jsonToBinary  
   - File Name: `={{ $json.name }}` (uses workflow name)  
   - Connect GET WORKFLOWS → MAKE JSON FILES.

10. **Create Dropbox Node: UPLOAD WORKFLOWS**  
    - Node Type: Dropbox  
    - Operation: Upload  
    - Path: `={{ $('DESTINATION FOLDER').last().json.folder }}{{ $('GET WORKFLOWS').item.json.name }}.json`  
    - Binary Data: true  
    - Authentication: Dropbox OAuth2  
    - Connect MAKE JSON FILES → UPLOAD WORKFLOWS.

11. **Create DateTime Node: PURGE DAYS** (optional, enable if purging old backups)  
    - Node Type: DateTime  
    - Operation: Subtract From Date  
    - Duration: 30 (or desired purge days)  
    - Magnitude: `={{ $now }}`  
    - Connect DESTINATION FOLDER → PURGE DAYS.

12. **Create Dropbox Node: LIST OLD BACKUPS**  
    - Node Type: Dropbox  
    - Operation: List Folder  
    - Path: `={{ $('DESTINATION FOLDER').last().json.folder }}old`  
    - Limit: 500  
    - Authentication: Dropbox OAuth2  
    - Connect PURGE DAYS → LIST OLD BACKUPS.

13. **Create If Node: CHECK DATES**  
    - Node Type: If  
    - Condition: `$json.lastModifiedServer < $('PURGE DAYS').item.json.newDate` (dateTime before)  
    - Connect LIST OLD BACKUPS → CHECK DATES.

14. **Create Dropbox Node: DELETE OLD BACKUPS**  
    - Node Type: Dropbox  
    - Operation: Delete  
    - Path: `={{ $json.pathDisplay }}`  
    - Authentication: Dropbox OAuth2  
    - Connect CHECK DATES (true branch) → DELETE OLD BACKUPS.

15. **Add Sticky Notes** (optional but recommended for clarity)  
    - Add notes describing each block and their purpose at appropriate positions.

16. **Finalize and Enable Workflow**  
    - Verify all connections and credentials.  
    - Enable the workflow and test manually before relying on scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The purge old backups functionality is disabled by default for safety. Enable only after testing. | Safety note in workflow description                                                                |
| Ensure your Dropbox OAuth2 credentials have sufficient permissions for file operations.        | Dropbox API permissions                                                                              |
| Workflow names with special characters may cause upload errors; sanitize names if necessary.   | Potential filename constraints on Dropbox                                                           |
| You can adjust the schedule trigger to run at any desired time, respecting your time zone.    | n8n Schedule Trigger documentation                                                                 |
| Project credits and template origin: n8n community workflow template #2075                      | https://n8n.io/workflows/2075-nightly-n8n-backup-to-dropbox                                         |

---

This reference document fully describes the workflow structure, nodes, configurations, and provides explicit instructions to reimplement it manually in n8n, ensuring maintainability and extensibility.