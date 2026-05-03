Monitor Dropbox Folders for New Files with DB Comparison

https://n8nworkflows.xyz/workflows/monitor-dropbox-folders-for-new-files-with-db-comparison-3297


# Monitor Dropbox Folders for New Files with DB Comparison

### 1. Workflow Overview

This workflow is designed to monitor multiple Dropbox folders for new or updated files using Dropbox webhooks and n8n automation. Since n8n lacks a native "watch folder" node for Dropbox, this workflow implements two distinct logical approaches to detect and process files in monitored folders:

- **Way #1:** For each file in a monitored folder (both new and existing), the workflow triggers a specialized sub-workflow to process that file individually.
- **Way #2:** The workflow lists all files in a monitored folder, compares them against a database of known files, filters out files already processed, and triggers a sub-workflow only for new files.

The workflow is structured into these main logical blocks:

- **1.1 Webhook Reception and Immediate Response:** Receives Dropbox webhook events and promptly responds to Dropbox to avoid webhook disabling.
- **1.2 Folder Monitoring Setup:** Defines which folders to monitor by setting folder path variables for each branch.
- **1.3 Way #1 - Per File Processing:** Lists all files in a folder, filters out folders, and calls a sub-workflow for each file.
- **1.4 Way #2 - New File Detection via DB Comparison:** Lists all files, queries a database for known files, merges to find new files, adds new files to the DB, and calls a sub-workflow for new files only.

Each folder to monitor is represented as a separate branch in the workflow, allowing parallel monitoring of multiple folders with customized sub-workflows.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Reception and Immediate Response

**Overview:**  
This block receives incoming Dropbox webhook notifications and immediately responds to Dropbox to confirm receipt, preventing webhook disabling due to timeout.

**Nodes Involved:**  
- Webhook  
- Respond to Dropbox in less than 10sec  
- Just a quick answer to Dropbox - webhook validation

**Node Details:**

- **Webhook**  
  - *Type:* Webhook node  
  - *Role:* Entry point for Dropbox webhook POST/GET requests  
  - *Configuration:* Path set to a unique webhook ID; accepts both POST and GET methods; response mode set to "responseNode" for asynchronous response handling.  
  - *Input/Output:* No input; outputs webhook data to two parallel branches.  
  - *Failure Modes:* Network issues, invalid webhook calls, malformed payloads.  
  - *Notes:* Critical for webhook reliability.

- **Respond to Dropbox in less than 10sec**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends immediate HTTP response to Dropbox webhook requests  
  - *Configuration:* Responds with the `challenge` query parameter from the webhook request, as required by Dropbox webhook validation.  
  - *Input:* Receives webhook data.  
  - *Output:* Ends this branch.  
  - *Failure Modes:* Missing or malformed `challenge` parameter.

- **Just a quick answer to Dropbox - webhook validation**  
  - *Type:* Respond to Webhook  
  - *Role:* Handles webhook validation requests from Dropbox (GET requests)  
  - *Configuration:* Same as above, responds with the `challenge` query parameter.  
  - *Input:* Receives webhook data.  
  - *Output:* Ends this branch.  
  - *Failure Modes:* Same as above.

---

#### 1.2 Folder Monitoring Setup

**Overview:**  
Sets variables defining the Dropbox folder paths to monitor. This allows reuse of folder path variables in subsequent nodes and branches.

**Nodes Involved:**  
- set_folder A  
- set_folder to watch B  
- Sticky Note4  
- Sticky Note5

**Node Details:**

- **set_folder A**  
  - *Type:* Set node  
  - *Role:* Defines variable `folder_to_watch` for folder A (e.g., "/z_Apps/a_iphone/RecUp Memos/")  
  - *Configuration:* Assigns a string value to `folder_to_watch`.  
  - *Input:* Receives trigger from Respond to Dropbox node.  
  - *Output:* Passes data downstream to Dropbox get files node.  
  - *Failure Modes:* Incorrect folder path string.

- **set_folder to watch B**  
  - *Type:* Set node  
  - *Role:* Defines variable `folder_to_watch` for folder B (e.g., "/z_Apps/auphonic/whisper")  
  - *Configuration:* Similar to set_folder A.  
  - *Input:* Receives trigger from Respond to Dropbox node.  
  - *Output:* Passes data downstream to Dropbox - List watched folder and NocoDB query nodes.  
  - *Failure Modes:* Same as above.

- **Sticky Note4 & Sticky Note5**  
  - *Type:* Sticky Note  
  - *Role:* Documentation nodes explaining duplication of branches per folder and rationale for using variables.  
  - *No inputs or outputs.*

---

#### 1.3 Way #1 - Per File Processing (Process All Files)

**Overview:**  
This branch processes every file in a specified folder by listing all files, filtering out folders, and executing a sub-workflow for each file.

**Nodes Involved:**  
- Dropbox get files  
- Switch File vs Folder1  
- Execute Workflow - what i want to do for this folder/file A  
- set_folder A  
- Sticky Note  
- Sticky Note2

**Node Details:**

- **Dropbox get files**  
  - *Type:* Dropbox node (folder list)  
  - *Role:* Lists all entries (files and folders) in the folder specified by `folder_to_watch`  
  - *Configuration:* Uses OAuth2 credentials; path set dynamically from `folder_to_watch`; excludes deleted and mounted folders; returns all entries.  
  - *Input:* Receives folder path from set_folder A.  
  - *Output:* Outputs list of files and folders.  
  - *Failure Modes:* Authentication errors, invalid folder path, API rate limits.

- **Switch File vs Folder1**  
  - *Type:* Switch node  
  - *Role:* Filters entries to separate files from folders  
  - *Configuration:* Checks if `type` equals "file" or "folder"; outputs only files to next node.  
  - *Input:* Receives list from Dropbox get files.  
  - *Output:* Files branch connected to Execute Workflow node; folders branch unused here.  
  - *Failure Modes:* Unexpected or missing `type` field.

- **Execute Workflow - what i want to do for this folder/file A**  
  - *Type:* Execute Workflow node  
  - *Role:* For each file, triggers a specialized sub-workflow to process it  
  - *Configuration:* Mode set to "each" to process each file individually; does not wait for sub-workflow completion; workflow ID references a specific sub-workflow for folder A.  
  - *Input:* Receives filtered file entries.  
  - *Output:* No further output in this workflow.  
  - *Failure Modes:* Sub-workflow errors, invalid workflow ID, data mapping issues.

- **Sticky Note & Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Documentation describing this approach as "Way 1" and its purpose.

---

#### 1.4 Way #2 - New File Detection via DB Comparison

**Overview:**  
This branch lists all files in a folder, queries a database (NocoDB) for known files, filters out files already processed, adds new files to the database, and triggers a sub-workflow only for new files.

**Nodes Involved:**  
- set_folder to watch B  
- Dropbox - List watched folder  
- NocoDB - Get know files to exclude  
- Switch File vs Folder  
- Merge - Keep only new items  
- NocoDB - Add this file in the table  
- Execute Workflow - Something to do for new files  
- Sticky Note3  
- Sticky Note5

**Node Details:**

- **set_folder to watch B**  
  - *Type:* Set node  
  - *Role:* Defines `folder_to_watch` variable for this branch.  
  - *Input:* Triggered from Respond to Dropbox node.  
  - *Output:* Passes data to Dropbox list and NocoDB query nodes.

- **Dropbox - List watched folder**  
  - *Type:* Dropbox node (folder list)  
  - *Role:* Lists all entries (files and folders) in the monitored folder  
  - *Configuration:* Uses OAuth2 credentials; path set dynamically from `folder_to_watch`; excludes deleted and mounted folders; returns all entries.  
  - *Input:* Receives folder path from set_folder to watch B.  
  - *Output:* Outputs list of files and folders.  
  - *Failure Modes:* Same as Dropbox get files node.

- **NocoDB - Get know files to exclude**  
  - *Type:* NocoDB node  
  - *Role:* Queries the database table for files already known in the monitored folder  
  - *Configuration:* Uses API token authentication; queries table with filter `folder_to_watch eq {{ $json.folder_to_watch }}`; returns all matching records.  
  - *Input:* Receives folder path to query known files.  
  - *Output:* Outputs known files data.  
  - *Failure Modes:* Authentication errors, query syntax errors, network issues.

- **Switch File vs Folder**  
  - *Type:* Switch node  
  - *Role:* Filters Dropbox entries to separate files from folders  
  - *Configuration:* Checks if `type` equals "file" or "folder"; only files proceed to merge.  
  - *Input:* Receives Dropbox folder listing.  
  - *Output:* Files branch connected to Merge node.  
  - *Failure Modes:* Same as other Switch node.

- **Merge - Keep only new items**  
  - *Type:* Merge node  
  - *Role:* Compares Dropbox files (input1) against known files from DB (input2) by matching Dropbox file `id` with DB `data.id`  
  - *Configuration:* Join mode "keepNonMatches" to keep only files not found in DB (new files); output data from input1 (Dropbox files).  
  - *Input:* Input1 from Dropbox files; Input2 from NocoDB known files.  
  - *Output:* Outputs only new files.  
  - *Failure Modes:* Data format mismatches, missing IDs, merge failures.

- **NocoDB - Add this file in the table**  
  - *Type:* NocoDB node  
  - *Role:* Adds new Dropbox file metadata to the database to mark it as processed  
  - *Configuration:* Uses API token authentication; creates new record with fields: `folder_to_watch`, `data` (JSON string of Dropbox file metadata), and `file_id` (Dropbox file ID); uses expressions to map data from the current file item and folder variable.  
  - *Input:* Receives new files from Merge node.  
  - *Output:* Passes data to Execute Workflow node.  
  - *Failure Modes:* Authentication errors, data mapping errors, DB write failures.

- **Execute Workflow - Something to do for new files**  
  - *Type:* Execute Workflow node  
  - *Role:* Triggers a specialized sub-workflow to process new files only  
  - *Configuration:* Standard execution mode; references a specific workflow ID for processing new files in this folder.  
  - *Input:* Receives newly added files from NocoDB add node.  
  - *Output:* No further output in this workflow.  
  - *Failure Modes:* Sub-workflow errors, invalid workflow ID.

- **Sticky Note3 & Sticky Note5**  
  - *Type:* Sticky Note  
  - *Role:* Documentation describing this approach as "Way 2" and the use of a variable for folder path.

---

### 3. Summary Table

| Node Name                                  | Node Type               | Functional Role                                      | Input Node(s)                        | Output Node(s)                                | Sticky Note                                                                                          |
|--------------------------------------------|-------------------------|-----------------------------------------------------|------------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                                    | Webhook                 | Receives Dropbox webhook events                      | -                                  | Respond to Dropbox in less than 10sec, Just a quick answer to Dropbox - webhook validation | ## Dropbox<br>Dropbox call me each time a modification is done somewhere in my dropbox.             |
| Respond to Dropbox in less than 10sec      | Respond to Webhook      | Sends immediate response to Dropbox webhook          | Webhook                            | set_folder A, set_folder to watch B             |                                                                                                    |
| Just a quick answer to Dropbox - webhook validation | Respond to Webhook      | Handles webhook validation GET requests              | Webhook                            | -                                             |                                                                                                    |
| set_folder A                               | Set                     | Defines folder path variable for folder A            | Respond to Dropbox in less than 10sec | Dropbox get files                              | I duplicate those processes for each folder i want to watch                                         |
| Dropbox get files                          | Dropbox                 | Lists all entries in folder A                         | set_folder A                      | Switch File vs Folder1                         |                                                                                                    |
| Switch File vs Folder1                     | Switch                  | Filters files vs folders in folder A                  | Dropbox get files                 | Execute Workflow - what i want to do for this folder/file A |                                                                                                    |
| Execute Workflow - what i want to do for this folder/file A | Execute Workflow        | Processes each file individually via sub-workflow    | Switch File vs Folder1            | -                                             |                                                                                                    |
| set_folder to watch B                      | Set                     | Defines folder path variable for folder B            | Respond to Dropbox in less than 10sec | Dropbox - List watched folder, NocoDB - Get know files to exclude | I define in a "variable" the folder to watch to ease the next steps                                |
| Dropbox - List watched folder              | Dropbox                 | Lists all entries in folder B                         | set_folder to watch B             | Switch File vs Folder, NocoDB - Get know files to exclude |                                                                                                    |
| NocoDB - Get know files to exclude         | NocoDB                  | Queries DB for known files in folder B                | set_folder to watch B             | Merge - Keep only new items                    |                                                                                                    |
| Switch File vs Folder                      | Switch                  | Filters files vs folders in folder B                  | Dropbox - List watched folder     | Merge - Keep only new items                     |                                                                                                    |
| Merge - Keep only new items                | Merge                   | Keeps only files not found in DB (new files)          | Switch File vs Folder, NocoDB - Get know files to exclude | NocoDB - Add this file in the table            |                                                                                                    |
| NocoDB - Add this file in the table         | NocoDB                  | Adds new file metadata to DB                           | Merge - Keep only new items       | Execute Workflow - Something to do for new files |                                                                                                    |
| Execute Workflow - Something to do for new files | Execute Workflow        | Processes new files via sub-workflow                   | NocoDB - Add this file in the table | -                                             |                                                                                                    |
| Sticky Note                                | Sticky Note             | Describes Way #1 approach                              | -                                | -                                             | ## Watch Files, 2 ways :<br>1. We explore each file in a folder (new and old ones)<br>2. We want to filter new files only |
| Sticky Note1                               | Sticky Note             | Describes Dropbox webhook trigger                      | -                                | -                                             | ## Dropbox<br>Dropbox call me each time a modification is done somewhere in my dropbox.             |
| Sticky Note2                               | Sticky Note             | Describes Way #1 approach                              | -                                | -                                             | ### Way 1 - We call the subworklow for each file in the specified folder                           |
| Sticky Note3                               | Sticky Note             | Describes Way #2 approach                              | -                                | -                                             | ### Way 2- We filter new/old files then we call the subworkflow only for new files                 |
| Sticky Note4                               | Sticky Note             | Notes duplication of folder monitoring branches       | -                                | -                                             | I duplicate those processes for each folder i want to watch                                        |
| Sticky Note5                               | Sticky Note             | Notes use of variable for folder path                  | -                                | -                                             | I define in a "variable" the folder to watch to ease the next steps                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: Unique identifier (e.g., "29a6482f-36ac-4c15-8792-450aa32cf5f4")  
   - HTTP Methods: POST and GET  
   - Response Mode: responseNode  
   - Purpose: Receive Dropbox webhook notifications.

2. **Create Respond to Webhook Nodes**  
   - Two nodes:  
     - "Respond to Dropbox in less than 10sec"  
     - "Just a quick answer to Dropbox - webhook validation"  
   - Both respond with the expression `{{$json.query.challenge}}` as plain text.  
   - Connect both nodes as parallel outputs from the Webhook node.

3. **Create Set Nodes for Folder Variables**  
   - "set_folder A": Set variable `folder_to_watch` to folder path (e.g., "/z_Apps/a_iphone/RecUp Memos/")  
   - "set_folder to watch B": Set variable `folder_to_watch` to another folder path (e.g., "/z_Apps/auphonic/whisper")  
   - Connect both Set nodes from "Respond to Dropbox in less than 10sec" node.

4. **Way #1 Branch (Process All Files)**  
   - Create Dropbox node "Dropbox get files"  
     - Operation: List folder contents  
     - Path: `={{ $json.folder_to_watch }}`  
     - Filters: exclude deleted and mounted folders  
     - Authentication: OAuth2 with Dropbox credentials  
   - Connect "set_folder A" to "Dropbox get files".  
   - Create Switch node "Switch File vs Folder1"  
     - Condition: `$json.type` equals "file" outputs to "file" branch  
   - Connect "Dropbox get files" to "Switch File vs Folder1".  
   - Create Execute Workflow node "Execute Workflow - what i want to do for this folder/file A"  
     - Mode: Each (process each item individually)  
     - Workflow ID: Select the sub-workflow for folder A processing  
     - Wait for sub-workflow: false  
   - Connect "file" output of Switch node to Execute Workflow node.

5. **Way #2 Branch (Process New Files Only)**  
   - Create Dropbox node "Dropbox - List watched folder"  
     - Same configuration as Dropbox get files but connected to "set_folder to watch B".  
   - Create NocoDB node "NocoDB - Get know files to exclude"  
     - Operation: getAll  
     - Table: Your NocoDB table name  
     - Filter: `=(folder_to_watch,eq,{{ $json.folder_to_watch }})`  
     - Authentication: NocoDB API token  
   - Connect "set_folder to watch B" to both Dropbox and NocoDB nodes.  
   - Create Switch node "Switch File vs Folder"  
     - Condition: `$json.type` equals "file" outputs to "file" branch  
   - Connect Dropbox node to Switch node.  
   - Create Merge node "Merge - Keep only new items"  
     - Mode: Combine  
     - Join Mode: Keep Non Matches  
     - Merge By Fields: Dropbox file `id` and NocoDB `data.id`  
     - Output data from input1 (Dropbox files)  
   - Connect Switch node "file" output to Merge input1.  
   - Connect NocoDB get node to Merge input2.  
   - Create NocoDB node "NocoDB - Add this file in the table"  
     - Operation: Create  
     - Table: Same as above  
     - Fields:  
       - `folder_to_watch`: `={{ $('set_folder to watch B').item.json.folder_to_watch }}`  
       - `data`: JSON string of Dropbox file metadata using expressions for each field (id, name, lastModifiedClient, lastModifiedServer, rev, contentSize, type, contentHash, pathLower, pathDisplay, isDownloadable)  
       - `file_id`: `={{ $json.id }}`  
     - Authentication: NocoDB API token  
   - Connect Merge node output to NocoDB add node.  
   - Create Execute Workflow node "Execute Workflow - Something to do for new files"  
     - Workflow ID: Select sub-workflow for processing new files in folder B  
   - Connect NocoDB add node to Execute Workflow node.

6. **Add Sticky Notes**  
   - Add sticky notes for documentation as per the original workflow to explain Dropbox webhook usage, the two ways of watching files, and variable usage.

7. **Credentials Setup**  
   - Dropbox OAuth2 credentials configured with appropriate app keys and tokens.  
   - NocoDB API token credentials configured with access to the project and table.

8. **Testing and Validation**  
   - Test webhook reception with Dropbox webhook test tools.  
   - Validate folder paths and database connectivity.  
   - Confirm sub-workflows execute as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed to centralize Dropbox folder monitoring in a single n8n workflow, scalable by adding branches per folder. | Workflow design philosophy from the user’s description.                                         |
| Dropbox webhook requires immediate response to avoid disabling; hence the early Respond to Webhook nodes.        | Dropbox webhook API documentation.                                                              |
| Two monitoring approaches: process all files vs. process only new files via DB comparison, allowing flexibility.| User’s workflow description section “How it works”.                                            |
| Database schema includes columns: folder_to_watch, data (JSON), timestamp, file_id for efficient file tracking.  | User’s description of DB structure.                                                             |
| Sub-workflows are referenced by ID and must be created separately with expected input/output schema.             | Workflow modularity and scalability.                                                            |
| Dropbox OAuth2 and NocoDB API token credentials must be set up securely in n8n credentials manager.              | n8n credentials management best practices.                                                      |
| Sticky notes in the workflow provide inline documentation for maintainers and users.                             | Enhances workflow readability and maintainability.                                             |

---

This document fully describes the workflow’s structure, logic, and configuration, enabling advanced users and automation agents to understand, reproduce, and extend the Dropbox folder monitoring system implemented in n8n.