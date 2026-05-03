Automated Workflow Backups with Google Drive and Slack Notifications

https://n8nworkflows.xyz/workflows/automated-workflow-backups-with-google-drive-and-slack-notifications-7850


# Automated Workflow Backups with Google Drive and Slack Notifications

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows by exporting them as JSON files, compressing them into a ZIP archive, and uploading the archive to a dedicated folder in Google Drive every four hours or upon manual trigger. After each backup, it sends notifications on Slack indicating success or failure. The workflow consists of the following logical blocks:

- **1.1 Trigger Block:** Initiates the workflow either manually or on a scheduled 4-hour interval.  
- **1.2 Google Drive Folder Management:** Creates a timestamped backup folder on Google Drive and sets the folder ID for file uploads.  
- **1.3 Workflow Data Retrieval & Conversion:** Retrieves all existing n8n workflows and converts each workflow’s JSON data into individual files.  
- **1.4 Compression:** Combines all workflow JSON files into a single ZIP archive.  
- **1.5 Upload & Notification:** Uploads the ZIP backup file to the created Google Drive folder and sends Slack notifications for success or failure.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** This block triggers the backup process either manually via user execution or automatically every 4 hours.  
- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger  

- **Node Details:**  
  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connects to “create new folder” node.  
    - Failures: None typical; user must manually trigger.  
    - Version: 1  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow on a repeating 4-hour interval.  
    - Configuration: Interval set to every 4 hours.  
    - Inputs: None  
    - Outputs: Connects to “create new folder” node.  
    - Failures: Possible errors if n8n scheduler fails or is paused.  
    - Version: 1.2  

#### 2.2 Google Drive Folder Management

- **Overview:** Creates a new Google Drive folder named with the current weekday and date, then sets the folder ID for subsequent uploads.  
- **Nodes Involved:**  
  - create new folder (Google Drive)  
  - Set New Folder Id (Set)  

- **Node Details:**  
  - **create new folder**  
    - Type: Google Drive node (Folder resource)  
    - Role: Creates a new backup folder inside a specified parent folder (ID: “14Mui2em5keVRWaoq3dlrvaX8KNRxw2h2” named “N8N Backups”).  
    - Configuration: Folder name dynamically set to “Workflow Backups <Weekday> <Time> <Date>” using `$now.format('cccc t dd-MM-yyyy')`.  
    - Inputs: From triggers (manual or schedule)  
    - Outputs: On success, connects to “Set New Folder Id” and “Get many workflows”; on error, connects to “Failed Notification” but continues workflow.  
    - Credentials: Uses Google Drive OAuth2 credentials (“Google Drive account”).  
    - Failure Modes: API errors, permission issues, rate limiting. Configured to continue on error.  
    - Version: 3  
  - **Set New Folder Id**  
    - Type: Set node  
    - Role: Extracts and stores the created folder’s ID into a JSON property `folderId` for use downstream.  
    - Configuration: Assigns `folderId` = `{{$json.id}}` from the folder creation output.  
    - Inputs: Output from “create new folder”  
    - Outputs: Connects to “Merge” node for combining file upload data.  
    - Failures: None expected.  
    - Version: 3.4  

#### 2.3 Workflow Data Retrieval & Conversion

- **Overview:** Fetches all workflows from the local n8n instance and converts each workflow JSON into individual file objects.  
- **Nodes Involved:**  
  - Get many workflows (n8n API)  
  - Convert to File1 (Convert to File)  

- **Node Details:**  
  - **Get many workflows**  
    - Type: n8n node (n8n API)  
    - Role: Retrieves all workflows available on the current n8n instance.  
    - Configuration: No filters, returns all workflows (`returnAll=1`), retries on failure.  
    - Inputs: From “create new folder” node (runs in parallel)  
    - Outputs: Connects to “Convert to File1”  
    - Credentials: Uses n8n API credentials (“n8n account”).  
    - Failures: Possible API or network errors; retried automatically.  
    - Version: 1  
  - **Convert to File1**  
    - Type: Convert to File node  
    - Role: Converts each workflow JSON item to a separate JSON file; filename is set dynamically with the workflow name.  
    - Configuration: Mode “each”, operation “toJson”, filename set as `{{$json.name}}.json`.  
    - Inputs: From “Get many workflows”  
    - Outputs: Connects to “Code” node.  
    - Failures: Expression errors if `name` is missing.  
    - Version: 1.1  

#### 2.4 Compression

- **Overview:** Aggregates all generated JSON files into a single ZIP archive for easier storage and transfer.  
- **Nodes Involved:**  
  - Code  
  - Compression  
  - Merge  

- **Node Details:**  
  - **Code**  
    - Type: Code (JavaScript) node  
    - Role: Prepares binary data from multiple incoming files into a single object and lists binary keys for the compression node.  
    - Configuration: Custom JavaScript code that iterates all input items, collects their binary data under keys `data_0`, `data_1`, etc., and returns a single item with all binaries and a JSON listing the keys.  
    - Inputs: From “Convert to File1” (multiple files)  
    - Outputs: Connects to “Compression” node  
    - Failures: Code errors if input binary data is missing or malformed.  
    - Version: 2  
  - **Compression**  
    - Type: Compression node  
    - Role: Compresses all files provided in binary form into a ZIP archive named dynamically with the current date.  
    - Configuration: Operation “compress”, output format “zip”, filename `workflows_backup_<YYYY-MM-DD>.zip`, processes binaries from all keys returned by the Code node.  
    - Inputs: From “Code” node  
    - Outputs: Connects to “Merge” node  
    - Failures: File processing errors, memory issues on large data.  
    - Version: 1.1  
  - **Merge**  
    - Type: Merge node  
    - Role: Combines two data streams: compressed archive binary and folder ID metadata for the upload step.  
    - Configuration: Mode “combine”, combine by position (aligns inputs by index).  
    - Inputs:  
      - Main input 0: From “Compression” node (compressed file)  
      - Main input 1: From “Set New Folder Id” node (folder metadata)  
    - Outputs: Connects to “Upload file” node.  
    - Failures: Misalignment if input streams differ in length or timing.  
    - Version: 3.2  

#### 2.5 Upload & Notification

- **Overview:** Uploads the ZIP backup file to the newly created Google Drive folder and sends Slack notifications indicating success or failure of the backup operation.  
- **Nodes Involved:**  
  - Upload file (Google Drive)  
  - Completed Notification (Slack)  
  - Failed Notification (Slack)  

- **Node Details:**  
  - **Upload file**  
    - Type: Google Drive node (File resource)  
    - Role: Uploads the compressed ZIP file into the folder specified by `folderId`.  
    - Configuration: File name set dynamically as `workflows_backup_<YYYY-MM-DD>.zip`, folder ID from merged data, input data field name is `data` (the binary ZIP file).  
    - Inputs: From “Merge” node  
    - Outputs:  
      - On success: Connects to “Completed Notification” node  
      - On error: Connects to “Failed Notification” node (with continue error output enabled upstream)  
    - Credentials: Google Drive OAuth2 credentials (“Google Drive account”)  
    - Failures: API errors, permission issues, rate limits, large file upload errors.  
    - Version: 3  
  - **Completed Notification**  
    - Type: Slack node  
    - Role: Sends a success message to a Slack channel after backup completion.  
    - Configuration: Message text includes number of workflows backed up (`{{ $('Get many workflows').all().length }}`), posts to channel with ID `C099YS0V3M2`.  
    - Inputs: From “Upload file” (success output)  
    - Credentials: Slack API credentials (“Slack account”)  
    - Failures: Slack API errors, invalid webhook or channel ID.  
    - Version: 2.2  
  - **Failed Notification**  
    - Type: Slack node  
    - Role: Sends an error message to Slack if any failure occurs during folder creation or file upload.  
    - Configuration: Static error message “❌ Error in the N8N Backup Workflow”, posts to channel `C099YS0V3M2`.  
    - Inputs: From “create new folder” (on error) and “Upload file” (on error)  
    - Credentials: Slack API credentials (“Slack account”)  
    - Failures: Slack API errors, invalid webhook or channel ID.  
    - Version: 2.2  

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                        | Input Node(s)              | Output Node(s)                             | Sticky Note                                  |
|---------------------|-----------------------|-------------------------------------|----------------------------|--------------------------------------------|----------------------------------------------|
| On clicking 'execute'| Manual Trigger        | Manual start of backup workflow      | -                          | create new folder                         |                                              |
| Schedule Trigger     | Schedule Trigger      | Automatic 4-hour interval trigger    | -                          | create new folder                         |                                              |
| create new folder    | Google Drive          | Creates backup folder in Drive       | On clicking 'execute', Schedule Trigger | Set New Folder Id, Get many workflows, Failed Notification (on error) |                                              |
| Set New Folder Id    | Set                   | Saves created folder ID for uploads  | create new folder           | Merge                                     |                                              |
| Get many workflows   | n8n API               | Retrieves all workflows from n8n     | create new folder           | Convert to File1                          |                                              |
| Convert to File1     | Convert to File       | Converts workflows JSON to files     | Get many workflows          | Code                                      |                                              |
| Code                | Code (JavaScript)     | Prepares binaries for compression    | Convert to File1            | Compression                               |                                              |
| Compression         | Compression           | Compresses files into ZIP archive     | Code                       | Merge                                     |                                              |
| Merge               | Merge                 | Combines folder ID and ZIP binary    | Compression, Set New Folder Id | Upload file                               |                                              |
| Upload file          | Google Drive          | Uploads ZIP archive to Drive folder  | Merge                      | Completed Notification, Failed Notification (on error) |                                              |
| Completed Notification| Slack                 | Sends success notification to Slack | Upload file (success)       | -                                         |                                              |
| Failed Notification  | Slack                 | Sends failure notification to Slack | create new folder (error), Upload file (error) | -                                         |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named “On clicking 'execute'” with default settings.  
   - Add a **Schedule Trigger** node named “Schedule Trigger” and set it to run every 4 hours.  

2. **Create Google Drive Folder:**  
   - Add a **Google Drive** node named “create new folder”.  
   - Configure it to create a folder resource inside the parent folder with ID `14Mui2em5keVRWaoq3dlrvaX8KNRxw2h2`.  
   - Set folder name to expression: `Workflow Backups {{$now.format('cccc t dd-MM-yyyy')}}`.  
   - Use Google Drive OAuth2 credentials.  
   - Set "On Error" to continue error output.  
   - Connect outputs from both trigger nodes to this node.  

3. **Set Folder ID:**  
   - Add a **Set** node named “Set New Folder Id”.  
   - Assign a new field `folderId` with expression `{{$json["id"]}}`.  
   - Connect “create new folder” success output to this node.  

4. **Retrieve All Workflows:**  
   - Add an **n8n** node named “Get many workflows”.  
   - Configure to request all workflows without filters (`returnAll=1`).  
   - Use n8n API credentials.  
   - Enable retry on fail.  
   - Connect “create new folder” success output (parallel) to this node.  

5. **Convert Workflows to Files:**  
   - Add a **Convert to File** node named “Convert to File1”.  
   - Set operation to “toJson”, mode “each”.  
   - Filename expression: `{{$json.name}}.json`.  
   - Connect “Get many workflows” output to this node.  

6. **Prepare Binaries for Compression:**  
   - Add a **Code** node named “Code”.  
   - Paste the following JavaScript code:  
     ```javascript
     let binaries = {}, binary_keys = [];

     for (const [index, inputItem] of Object.entries($input.all())) {
       binaries[`data_${index}`] = inputItem.binary.data;
       binary_keys.push(`data_${index}`);
     }

     return [{
         json: {
             binary_keys: binary_keys.join(',')
         },
         binary: binaries
     }];
     ```  
   - Connect “Convert to File1” output to this node.  

7. **Compress Files into ZIP:**  
   - Add a **Compression** node named “Compression”.  
   - Set operation to “compress”, output format “zip”.  
   - Filename expression: `'workflows_backup_' + new Date().toISOString().slice(0,10) + '.zip'`.  
   - For binary property name, set expression: `Object.keys($binary).join(',')`.  
   - Connect “Code” output to this node.  

8. **Merge Folder ID and ZIP Data:**  
   - Add a **Merge** node named “Merge”.  
   - Set mode to “combine”, combine by position.  
   - Connect “Compression” output to main input 0.  
   - Connect “Set New Folder Id” output to main input 1.  

9. **Upload ZIP to Google Drive:**  
   - Add a **Google Drive** node named “Upload file”.  
   - Configure to upload a file resource.  
   - File name expression: `'workflows_backup_' + new Date().toISOString().slice(0,10) + '.zip'`.  
   - Drive ID: “My Drive”.  
   - Folder ID expression: `{{$json.folderId}}`.  
   - Input data field name: `data`.  
   - Use Google Drive OAuth2 credentials.  
   - Connect “Merge” output to this node.  
   - Set "On Error" to continue error output.  

10. **Add Slack Notification Nodes:**  
    - Add a **Slack** node named “Completed Notification”.  
    - Configure the message text: `✅ Backup has completed - {{ $('Get many workflows').all().length }} workflows have been processed.`  
    - Select the Slack channel by ID: `C099YS0V3M2`.  
    - Use Slack API credentials.  
    - Connect “Upload file” success output to this node.  
    - Enable execute once.  

    - Add another **Slack** node named “Failed Notification”.  
    - Configure the message text: `❌ Error in the N8N Backup Workflow`.  
    - Select the same Slack channel by ID.  
    - Use Slack API credentials.  
    - Connect “create new folder” error output and “Upload file” error output to this node.  
    - Enable execute once.  

11. **Activate Workflow:**  
    - Save and activate the workflow to run every 4 hours or manually trigger via the manual trigger node.  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                              |
|-----------------------------------------------------------------------------------------------|----------------------------------------------|
| Workflow backs up all n8n workflows as JSON files and stores them compressed on Google Drive.  | Main workflow purpose.                        |
| Slack notifications inform about backup success or failure.                                   | Slack channel ID: C099YS0V3M2                 |
| Google Drive parent folder for backups: ID 14Mui2em5keVRWaoq3dlrvaX8KNRxw2h2 ("N8N Backups")   | Folder where backups are organized.           |
| Uses OAuth2 credentials for Google Drive and Slack API credentials for Slack API access.       | Credential setup required.                     |
| Retry enabled on n8n API node to handle transient issues fetching workflows.                    | Improves reliability.                          |
| Compression node output filename dynamically reflects current date to avoid overwriting.       | Naming convention ensures unique backups.     |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.