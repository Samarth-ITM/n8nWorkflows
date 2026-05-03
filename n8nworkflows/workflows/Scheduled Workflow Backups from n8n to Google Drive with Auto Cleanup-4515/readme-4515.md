Scheduled Workflow Backups from n8n to Google Drive with Auto Cleanup

https://n8nworkflows.xyz/workflows/scheduled-workflow-backups-from-n8n-to-google-drive-with-auto-cleanup-4515


# Scheduled Workflow Backups from n8n to Google Drive with Auto Cleanup

### 1. Workflow Overview

This workflow automates the process of backing up all n8n workflows to a Google Drive folder every hour, with automatic cleanup of old backups beyond a configurable retention period. It is designed for n8n administrators and users who want to safeguard their workflow configurations through regular, systematic backups stored externally. The workflow logically divides into two main blocks:

- **1.1 Scheduled Backup Creation:** Triggered hourly, this block generates a timestamped folder in Google Drive, fetches all existing n8n workflows, converts each to a JSON file, and uploads them individually to the created folder with short waits between uploads to prevent API rate limits.

- **1.2 Old Backup Cleanup:** This block calculates a cutoff date based on a retention period (default 7 days), searches for backup folders older than this date in Google Drive, and deletes them to manage storage space.

Supporting the main blocks are configuration and utility nodes that handle date/time formatting and parameter settings.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Backup Creation

**Overview:**  
This block triggers the workflow every hour, creates a uniquely named backup folder in Google Drive using the current date and hour, retrieves all n8n workflows, converts each workflow's JSON to binary, and uploads them one at a time to the backup folder with a wait time between uploads.

**Nodes Involved:**  
- Schedule Trigger  
- Settings  
- Date Time  
- Date & Time Format  
- Date & Time Hour  
- Google Drive Backup Folder Every Hour  
- n8n (API node for workflows)  
- Move Binary Data  
- Split In Batches  
- Google Drive Upload Workflows  
- Wait  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* scheduleTrigger  
  - *Role:* Initiates workflow every hour automatically.  
  - *Config:* Interval set to every 1 hour.  
  - *Input/Output:* No input; outputs trigger event to Settings node.  
  - *Edge cases:* Workflow will not trigger if n8n instance is down; schedule misconfiguration may cause missed runs.

- **Settings**  
  - *Type:* set  
  - *Role:* Defines workflow parameters including backup retention period ("Coverage Period" default 7 days).  
  - *Config:* Single number assignment `Coverage Period = 7`.  
  - *Input:* Trigger from Schedule Trigger.  
  - *Output:* Passes coverage period to Date Time node.  
  - *Edge cases:* Incorrect values may break cleanup logic.

- **Date Time**  
  - *Type:* dateTime  
  - *Role:* Captures the current timestamp at runtime.  
  - *Config:* Output field named "now".  
  - *Input:* Settings node output.  
  - *Output:* Provides current date/time to formatting nodes.  
  - *Edge cases:* Timezone mismatches or system clock errors affect folder naming.

- **Date & Time Format**  
  - *Type:* dateTime  
  - *Role:* Formats captured timestamp to date string "yyyy-MM-dd".  
  - *Config:* Uses expression `={{ $('Date Time').first().json.now }}` for input date, format set to "yyyy-MM-dd".  
  - *Input:* Date Time node.  
  - *Output:* Formatted date string for folder naming.  
  - *Edge cases:* Expression failure if prior node data missing.

- **Date & Time Hour**  
  - *Type:* dateTime  
  - *Role:* Extracts the hour part from the timestamp.  
  - *Config:* Extracts "hour" part from `={{ $('Date Time').first().json.now }}`; output field named "hour".  
  - *Input:* Date & Time Format node.  
  - *Output:* Hour string for folder naming.  
  - *Edge cases:* Same as above.

- **Google Drive Backup Folder Every Hour**  
  - *Type:* googleDrive (folder resource)  
  - *Role:* Creates a new folder in Google Drive named `n8n_backup_YYYY-MM-DD_HH`.  
  - *Config:*  
    - Folder name uses expression `=n8n_backup_{{ $('Date & Time Format').first().json.formattedDate }}_{{ $('Date & Time Hour').first().json.hour }}`  
    - Drive ID and parent folder ID set to specific Google Drive folders via OAuth2 credentials.  
  - *Input:* Date & Time Hour node.  
  - *Output:* Newly created folder ID for upload target and cleanup calculation.  
  - *Edge cases:* Google Drive API auth failures, quota limits, or folder name collisions.

- **n8n**  
  - *Type:* n8n (API)  
  - *Role:* Fetches all existing workflows from the n8n instance.  
  - *Config:* No filters; retrieves all workflows. Uses n8n API credentials.  
  - *Input:* Google Drive Backup Folder Every Hour node.  
  - *Output:* JSON array of workflows.  
  - *Edge cases:* Authentication errors, API rate limits, or empty workflow list.

- **Move Binary Data**  
  - *Type:* moveBinaryData  
  - *Role:* Converts workflow JSON data to binary file format for upload.  
  - *Config:* Mode "jsonToBinary", sets filename from workflow name, uses raw data.  
  - *Input:* n8n node.  
  - *Output:* Binary data ready for upload.  
  - *Edge cases:* File naming conflicts or invalid characters.

- **Split In Batches**  
  - *Type:* splitInBatches  
  - *Role:* Processes workflows one at a time to manage upload pacing.  
  - *Config:* Batch size set to 1, no reset.  
  - *Input:* Move Binary Data node.  
  - *Output:* Single workflow binary data per execution.  
  - *Edge cases:* Large workflow counts may increase total runtime.

- **Google Drive Upload Workflows**  
  - *Type:* googleDrive (file resource)  
  - *Role:* Uploads individual workflow JSON files to the backup folder.  
  - *Config:*  
    - File name from batch item binary file name with `.json` extension.  
    - Parent folder ID set dynamically from created backup folder.  
    - Authentication via OAuth2 credentials.  
  - *Input:* Split In Batches node.  
  - *Output:* Confirmation of upload.  
  - *Edge cases:* API upload failures, connection timeouts.

- **Wait**  
  - *Type:* wait  
  - *Role:* Adds a 3-second delay between individual workflow uploads to avoid API throttling.  
  - *Config:* Wait amount set to 3 seconds.  
  - *Input:* Google Drive Upload Workflows node.  
  - *Output:* Pauses workflow before continuing.  
  - *Edge cases:* Longer wait times increase total backup duration.

---

#### 2.2 Old Backup Cleanup

**Overview:**  
This block calculates a cutoff date based on the backup retention period, searches Google Drive for backup folders older than that cutoff date, and deletes them to manage disk space.

**Nodes Involved:**  
- Date & Time Subtract Coverage Period  
- Date & Time Format 2  
- Google Drive Search  
- Google Drive Delete  
- Date & Time Hour (shared with creation block)  
- Settings (shared with creation block)  

**Node Details:**

- **Date & Time Subtract Coverage Period**  
  - *Type:* dateTime  
  - *Role:* Calculates the date/time before which backups should be deleted.  
  - *Config:* Subtracts the "Coverage Period" number of days from current date/time.  
  - *Input:* Settings node for coverage period, Date Time node for current time.  
  - *Output:* New date representing cutoff for cleanup.  
  - *Edge cases:* Incorrect coverage period may delete backups prematurely or retain too long.

- **Date & Time Format 2**  
  - *Type:* dateTime  
  - *Role:* Formats the cutoff date as "yyyy-MM-dd" to match backup folder naming.  
  - *Config:* Uses output from Date & Time Subtract Coverage Period.  
  - *Input:* Date & Time Subtract Coverage Period node.  
  - *Output:* Formatted date string for search query.  
  - *Edge cases:* Expression failures.

- **Google Drive Search**  
  - *Type:* googleDrive  
  - *Role:* Searches Google Drive for backup folders matching the naming pattern and cutoff date.  
  - *Config:*  
    - Query string formatted as `=n8n_backup_{{ formattedDate }}_{{ hour }}` to find folders by date and hour.  
    - Limit set to 1 (to find specific folder before deletion).  
  - *Input:* Date & Time Format 2 and Date & Time Hour nodes.  
  - *Output:* Folder ID(s) matching old backup folders.  
  - *Edge cases:* Search may miss folders if naming convention changes; API rate limits.

- **Google Drive Delete**  
  - *Type:* googleDrive (folder resource)  
  - *Role:* Deletes the identified old backup folder by ID.  
  - *Config:* Uses folder ID from Google Drive Search for deletion.  
  - *Input:* Google Drive Search node.  
  - *Output:* Confirmation of deletion.  
  - *Edge cases:* Permission errors, folder not found, API failures.

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                                  | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|-----------------------------------|-----------------------|-------------------------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | scheduleTrigger       | Starts workflow every hour                       | —                               | Settings                       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Settings                         | set                   | Defines backup retention period ("Coverage Period") | Schedule Trigger                | Date Time                     | ## Backup Coverage Period                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Date Time                       | dateTime              | Captures current timestamp                       | Settings                       | Date & Time Format             | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Date & Time Format               | dateTime              | Formats date as yyyy-MM-dd                        | Date Time                     | Date & Time Hour              | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Date & Time Hour                 | dateTime              | Extracts hour from current time                   | Date & Time Format             | Google Drive Backup Folder Every Hour, Date & Time Subtract Coverage Period | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Google Drive Backup Folder Every Hour | googleDrive (folder) | Creates backup folder in Google Drive named by date and hour | Date & Time Hour              | n8n                          | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| n8n                             | n8n (API)             | Fetches all workflows from n8n instance          | Google Drive Backup Folder Every Hour | Move Binary Data              | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Move Binary Data                | moveBinaryData        | Converts workflow JSON data to binary file       | n8n                           | Split In Batches              | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Split In Batches                | splitInBatches        | Processes one workflow at a time                  | Move Binary Data              | Google Drive Upload Workflows | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Google Drive Upload Workflows   | googleDrive (file)    | Uploads workflow JSON files to backup folder     | Split In Batches              | Wait                         | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Wait                           | wait                  | Waits 3 seconds between uploads                   | Google Drive Upload Workflows | Split In Batches              | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Date & Time Subtract Coverage Period | dateTime              | Calculates cutoff date for backup deletion        | Date & Time Hour, Settings    | Date & Time Format 2          | ## Delete N8n Backup Workflows Over The Past X Days                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Date & Time Format 2            | dateTime              | Formats cutoff date for search                     | Date & Time Subtract Coverage Period | Google Drive Search          | ## Delete N8n Backup Workflows Over The Past X Days                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Google Drive Search             | googleDrive           | Searches for old backup folders to delete         | Date & Time Format 2          | Google Drive Delete           | ## Delete N8n Backup Workflows Over The Past X Days                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Google Drive Delete             | googleDrive           | Deletes old backup folders                          | Google Drive Search           | —                            | ## Delete N8n Backup Workflows Over The Past X Days                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note7                   | stickyNote            | Notes on creating hourly backup workflows          | —                           | —                            | ## Create N8n Backup Workflows Every Hour                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note                    | stickyNote            | Notes on deleting old backups                        | —                           | —                            | ## Delete N8n Backup Workflows Over The Past X Days                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note1                   | stickyNote            | Notes on backup coverage period                      | —                           | —                            | ## Backup Coverage Period                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note2                   | stickyNote            | Overview and detailed description of entire workflow | —                           | —                            | ## Hourly n8n Workflow Backup to Google Drive  \n\nThis workflow provides a robust solution for automatically backing up all your n8n workflows to Google Drive on schedule (default to every hour). It creates a uniquely named folder for each backup instance, incorporating the date and hour, and then systematically uploads each workflow as an individual JSON file. To manage storage space, the workflow also includes a cleanup mechanism that deletes backup folders older than a user-defined retention period (defaulting to 7 days).\n\nFor more powerful n8n templates, visit our website or contact us at [**AI Automation Pro**](https://aiautomationpro.org/). We help your business build custom AI workflow automation and apps. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: scheduleTrigger  
   - Configure to run every 1 hour (set interval to hours, value 1).  

2. **Create Settings Node**  
   - Type: set  
   - Add a number parameter named "Coverage Period" with value 7 (default retention days).  
   - Connect Schedule Trigger output to this node.  

3. **Create Date Time Node**  
   - Type: dateTime  
   - Set output field name to "now".  
   - Connect Settings output to this node.  

4. **Create Date & Time Format Node**  
   - Type: dateTime  
   - Operation: formatDate  
   - Date: Use expression `={{ $('Date Time').first().json.now }}`  
   - Format: "yyyy-MM-dd"  
   - Connect Date Time output to this node.  

5. **Create Date & Time Hour Node**  
   - Type: dateTime  
   - Operation: extractDate  
   - Part: hour  
   - Date: Use expression `={{ $('Date Time').first().json.now }}`  
   - Output field name: "hour"  
   - Connect Date & Time Format output to this node.  

6. **Create Google Drive Backup Folder Every Hour Node**  
   - Type: googleDrive (resource: folder)  
   - Name: Use expression `=n8n_backup_{{ $('Date & Time Format').first().json.formattedDate }}_{{ $('Date & Time Hour').first().json.hour }}`  
   - Drive ID: Select the Google Drive or specify its ID where backups will be stored.  
   - Folder ID: Set the parent folder ID in Google Drive (e.g., "n8n_Backups" folder).  
   - Credentials: Select or create Google Drive OAuth2 credentials.  
   - Connect Date & Time Hour output to this node.  

7. **Create n8n Node (API)**  
   - Type: n8n (API)  
   - Configure with n8n API credentials (OAuth or API key) that have permission to list workflows.  
   - No filters to fetch all workflows.  
   - Connect Google Drive Backup Folder Every Hour output to this node.  

8. **Create Move Binary Data Node**  
   - Type: moveBinaryData  
   - Mode: jsonToBinary  
   - File Name: Use expression `={{ $json.name }}` to name files after workflow names.  
   - Use raw data: true  
   - Connect n8n node output to this node.  

9. **Create Split In Batches Node**  
   - Type: splitInBatches  
   - Batch size: 1  
   - Reset: false  
   - Connect Move Binary Data output to this node.  

10. **Create Google Drive Upload Workflows Node**  
    - Type: googleDrive (resource: file)  
    - File name: Use expression `={{ $('Split In Batches').item.binary.data.fileName }}.json`  
    - Parents: Use expression `={{ $('Google Drive Backup Folder Every Hour').first().json.id }}` to upload files into the created folder.  
    - Authentication: OAuth2  
    - Credentials: Use the same Google Drive OAuth2 credentials as folder creation.  
    - Connect Split In Batches output to this node.  

11. **Create Wait Node**  
    - Type: wait  
    - Amount: 3 seconds  
    - Connect Google Drive Upload Workflows output to this node.  
    - Connect Wait node output back to Split In Batches to continue batch processing.  

12. **Create Date & Time Subtract Coverage Period Node**  
    - Type: dateTime  
    - Operation: subtractFromDate  
    - Duration: Use expression `={{ $('Settings').first().json['Coverage Period'] }}` (number of days)  
    - Magnitude: Use expression `={{ $('Date Time').first().json.now }}`  
    - Connect Date & Time Hour output to this node (for current time).  
    - Connect Settings output to this node (for coverage period).  

13. **Create Date & Time Format 2 Node**  
    - Type: dateTime  
    - Operation: formatDate  
    - Date: Use expression `={{ $('Date & Time Subtract Coverage Period').first().json.newDate }}`  
    - Format: "yyyy-MM-dd"  
    - Connect Date & Time Subtract Coverage Period output to this node.  

14. **Create Google Drive Search Node**  
    - Type: googleDrive (resource: fileFolder)  
    - Query String: Use expression `=n8n_backup_{{ $('Date & Time Format 2').first().json.formattedDate }}_{{ $('Date & Time Hour').first().json.hour }}`  
    - Limit: 1  
    - Credentials: Use Google Drive OAuth2 credentials.  
    - Connect Date & Time Format 2 output to this node.  

15. **Create Google Drive Delete Node**  
    - Type: googleDrive (resource: folder)  
    - Operation: deleteFolder  
    - Folder ID: Use expression `={{ $('Google Drive Search').first().json.id }}`  
    - Credentials: Use Google Drive OAuth2 credentials.  
    - Connect Google Drive Search output to this node.  

16. **Add Sticky Notes** (optional)  
    - Create sticky note nodes to document workflow segments and parameters for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow provides a robust solution for automatically backing up all your n8n workflows to Google Drive on schedule (default to every hour), with filename and folder naming conventions incorporating date and hour. It also includes a cleanup mechanism for deleting old backups beyond a configurable retention period. Visit [AI Automation Pro](https://aiautomationpro.org/) for more templates and professional AI workflow automation services. | Main workflow description and branding.                                                                           |
| The workflow requires proper configuration of Google Drive OAuth2 credentials (enable Drive API, correct scopes) and n8n API credentials (with permissions to read workflows). Ensure credentials are kept secure and refreshed as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Credential setup instructions.                                                                                    |
| The Wait node is set to 3 seconds between uploads to avoid Google Drive API rate limits; adjust this if you encounter throttling or if you have many workflows.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Rate limiting and performance tuning.                                                                             |
| An error workflow is set (`KhpM42Ckgy6qgzCz`) for handling errors globally; update or implement it in your n8n instance for error notifications or recovery.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Error handling configuration.                                                                                      |
| To modify backup frequency, adjust the Schedule Trigger node’s interval. To change retention period, update the "Coverage Period" value in the Settings node. Backup folder and file naming conventions can be customized in the respective Google Drive nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Customization instructions.                                                                                        |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.