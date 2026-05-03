Monitor and Download Changed Files from Google Drive Automatically

https://n8nworkflows.xyz/workflows/monitor-and-download-changed-files-from-google-drive-automatically-6193


# Monitor and Download Changed Files from Google Drive Automatically

### 1. Workflow Overview

This workflow automates monitoring and downloading files from a specific Google Drive folder that are new or have changed since the last execution. It uses a timestamp file (`n8n_last_run.txt`) stored on Google Drive to keep track of the last run time, enabling incremental synchronization. The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger & Timestamp Retrieval**: Periodically triggers the workflow and attempts to retrieve the last run timestamp from Google Drive.
- **1.2 Timestamp Extraction & Default Handling**: Extracts the timestamp from the retrieved file or defaults to 24 hours prior if the timestamp file is missing.
- **1.3 File Search & Conditional Download**: Searches for files created or modified after the timestamp, and downloads them if found.
- **1.4 Timestamp Update**: Creates and uploads a new timestamp file reflecting the current run time.
- **1.5 Cleanup and Merge**: Deletes the old timestamp file and merges downloaded files for further processing or downstream use.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Timestamp Retrieval

**Overview:**  
This block starts the workflow on a schedule (every minute) and searches for the timestamp file on Google Drive that records the last run time.

**Nodes Involved:**  
- Schedule Trigger1  
- Search for Timestamp File1  
- Download Timestamp File1

**Node Details:**

- **Schedule Trigger1**  
  - *Type*: Schedule Trigger  
  - *Role*: Initiates workflow execution every minute.  
  - *Configuration*: Interval set to trigger every 1 minute.  
  - *Inputs/Outputs*: No inputs; outputs trigger signal to the next node.  
  - *Edge Cases*: If the workflow is disabled or n8n instance is down, no trigger occurs.

- **Search for Timestamp File1**  
  - *Type*: Google Drive  
  - *Role*: Searches for `n8n_last_run.txt` in Google Drive (non-trashed).  
  - *Configuration*: Uses Google Drive OAuth2 credentials; query filters by name and trashed status; returns all matching files.  
  - *Expression Used*: Query string = `"=name = 'n8n_last_run.txt' and trashed = false"`.  
  - *Inputs/Outputs*: Receives trigger from Schedule Trigger1; outputs file metadata if found.  
  - *Edge Cases*: If file is missing, continues execution without error (`onError: continueRegularOutput`).

- **Download Timestamp File1**  
  - *Type*: Google Drive  
  - *Role*: Downloads the timestamp file content based on file ID from previous node.  
  - *Configuration*: Downloads file using file ID expression `{{$json.id}}` from Search for Timestamp File1 output.  
  - *Credentials*: Same Google Drive OAuth2 credentials.  
  - *Edge Cases*: If file ID is invalid or download fails, workflow continues due to error handling in previous node.

---

#### 2.2 Timestamp Extraction & Default Handling

**Overview:**  
Extracts the timestamp string from the downloaded file and prepares a fallback timestamp (24 hours ago) if the file does not exist or is empty.

**Nodes Involved:**  
- Extract Timestamp Text1  
- Edit Fields2

**Node Details:**

- **Extract Timestamp Text1**  
  - *Type*: Extract From File  
  - *Role*: Extracts plain text from the downloaded timestamp file.  
  - *Configuration*: Operation set to extract text content.  
  - *Inputs/Outputs*: Input is the downloaded file binary content; output is extracted text in JSON.  
  - *Edge Cases*: If file is empty or corrupted, extraction may return empty or error.

- **Edit Fields2**  
  - *Type*: Set  
  - *Role*: Sets a variable `searchFromTimestamp` to the extracted timestamp or defaults to ISO string of 24 hours ago.  
  - *Configuration*: Assigns `searchFromTimestamp` using expression:  
    ```javascript
    {{$json.data || new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString()}}
    ```  
  - *Inputs/Outputs*: Takes extracted text JSON; outputs JSON with `searchFromTimestamp` field.  
  - *Edge Cases*: If extracted text is invalid or empty, default timestamp ensures continuity.

---

#### 2.3 File Search & Conditional Download

**Overview:**  
Searches the target Google Drive folder for files created or modified after the timestamp, then downloads each new or changed file.

**Nodes Involved:**  
- Search for New Files2  
- If  
- Download New Files2

**Node Details:**

- **Search for New Files2**  
  - *Type*: Google Drive  
  - *Role*: Searches for files in a specific folder modified or created after `searchFromTimestamp`.  
  - *Configuration*:  
    - Folder ID: `1ycXdgyWRiQ_Ce_IgeZN1au6QKQNYOMJn` (target folder)  
    - Query includes:  
      ```
      '1ycXdgyWRiQ_Ce_IgeZN1au6QKQNYOMJn' in parents and (createdTime > '{{ $json.searchFromTimestamp }}' OR modifiedTime > '{{ $json.searchFromTimestamp }}') and trashed = false
      ```  
    - Returns all matching files.  
  - *Credentials*: Uses Google Drive OAuth2.  
  - *Edge Cases*: If no files match, output is empty but workflow continues.

- **If**  
  - *Type*: If Condition  
  - *Role*: Checks if files were found (non-empty id field).  
  - *Configuration*: Condition checks `{{$json.id}}` is not empty.  
  - *Outputs*:  
    - True branch: files found → triggers download.  
    - False branch: no files → triggers No Operation node.  
  - *Edge Cases*: Expression failures if `id` field missing.

- **Download New Files2**  
  - *Type*: Google Drive  
  - *Role*: Downloads each new or changed file by file ID.  
  - *Configuration*: File ID comes from search results (`{{$json.id}}`).  
  - *Credentials*: Same Google Drive OAuth2 account.  
  - *Edge Cases*: File deleted before download, permission issues, or download failure.

---

#### 2.4 Timestamp Update

**Overview:**  
Creates a new timestamp file representing the current workflow run time and uploads it to Google Drive, replacing the old timestamp file.

**Nodes Involved:**  
- Get Current Time1  
- Create Timestamp File  
- Upload New Timestamp2  
- Delete Old Timestamp File1

**Node Details:**

- **Get Current Time1**  
  - *Type*: DateTime  
  - *Role*: Generates current UTC time in ISO format.  
  - *Configuration*: Timezone set to UTC; no input fields included.  
  - *Outputs*: Current timestamp string.  
  - *Edge Cases*: None significant.

- **Create Timestamp File**  
  - *Type*: Code (JavaScript)  
  - *Role*: Converts the ISO timestamp string to a downloadable text file in base64 format.  
  - *Code Summary*:  
    - Reads `date` field or defaults to current time.  
    - Creates binary data with base64-encoded timestamp string and mimeType `text/plain`.  
  - *Outputs*: Binary file to be uploaded.  
  - *Edge Cases*: Code errors if input missing or malformed.

- **Upload New Timestamp2**  
  - *Type*: Google Drive  
  - *Role*: Uploads the newly created timestamp file to a specific folder, overwriting as needed.  
  - *Configuration*:  
    - Filename: `n8n_last_run.txt`  
    - Drive: "My Drive"  
    - Folder ID: `1ycXdgyWRiQ_Ce_IgeZN1au6QKQNYOMJn` (same as target folder)  
  - *Credentials*: Google Drive OAuth2 account.  
  - *Edge Cases*: Upload failure due to permissions or API limits.

- **Delete Old Timestamp File1**  
  - *Type*: Google Drive  
  - *Role*: Deletes the old timestamp file from Google Drive to prevent duplicates.  
  - *Configuration*: Deletes file by ID from first search node output.  
  - *Credentials*: Google Drive OAuth2.  
  - *Edge Cases*: File may have been already deleted or inaccessible.

---

#### 2.5 Cleanup and Merge

**Overview:**  
Merges downloaded new files with the workflow for further processing and performs a no-operation step if no new files exist.

**Nodes Involved:**  
- No Operation  
- Merge  
- Edit Fields

**Node Details:**

- **No Operation**  
  - *Type*: NoOp (No Operation)  
  - *Role*: Acts as a placeholder for empty branches (no new files).  
  - *Outputs*: Passes empty data to merge node for consistent flow.  
  - *Edge Cases*: None.

- **Merge**  
  - *Type*: Merge  
  - *Role*: Combines output of file downloads and no-operation into a single stream.  
  - *Configuration*: Default merge settings.  
  - *Outputs*: Merged data forwarded downstream.  
  - *Edge Cases*: Merge conflicts if inputs are empty or inconsistent.

- **Edit Fields**  
  - *Type*: Set  
  - *Role*: Placeholder for potential field edits post-merge; configured to execute once.  
  - *Outputs*: Passes data to Delete Old Timestamp File1 node.  
  - *Edge Cases*: None specific.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                  |
|---------------------------|---------------------|-----------------------------------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1          | Schedule Trigger    | Periodically triggers the workflow every minute           |                                 | Search for Timestamp File1      | Workflow Purpose: This workflow automatically downloads new or updated files from a Google Drive folder...  |
| Search for Timestamp File1 | Google Drive        | Searches for last run timestamp file on Google Drive      | Schedule Trigger1               | Download Timestamp File1        | Setup Required: You need to set up your Google Drive credentials in n8n for this workflow to function...     |
| Download Timestamp File1   | Google Drive        | Downloads the timestamp file content                       | Search for Timestamp File1      | Extract Timestamp Text1         | Setup Required (same as above)                                                                               |
| Extract Timestamp Text1    | Extract From File   | Extracts timestamp text from downloaded file               | Download Timestamp File1        | Edit Fields2                   | Timestamp File: The workflow looks for a n8n_last_run.txt file in Google Drive to know when it last ran...   |
| Edit Fields2              | Set                 | Sets timestamp variable or defaults to 24h ago            | Extract Timestamp Text1         | Search for New Files2          | Timestamp File (same as above)                                                                               |
| Search for New Files2      | Google Drive        | Searches for new or modified files since last timestamp   | Edit Fields2                   | If                            | Setup Required (same as above)                                                                               |
| If                        | If Condition        | Checks if new files were found                             | Search for New Files2           | Download New Files2, No Operation |                                                                                                              |
| Download New Files2        | Google Drive        | Downloads each new or changed file                         | If (true branch)                | Merge                         | Setup Required (same as above)                                                                               |
| No Operation              | NoOp                | Placeholder when no new files to download                  | If (false branch)               | Merge                         |                                                                                                              |
| Merge                     | Merge               | Merges downloaded files or no-operation output            | Download New Files2, No Operation | Edit Fields                   |                                                                                                              |
| Edit Fields               | Set                 | Alters fields before deleting old timestamp file          | Merge                         | Delete Old Timestamp File1     |                                                                                                              |
| Delete Old Timestamp File1 | Google Drive        | Deletes old timestamp file from Google Drive               | Edit Fields                   | Get Current Time1             | Setup Required (same as above)                                                                               |
| Get Current Time1          | DateTime            | Gets current UTC time for new timestamp                    | Delete Old Timestamp File1     | Create Timestamp File          | Timestamp Update: After all files are processed, the timestamp file is updated for future runs.              |
| Create Timestamp File      | Code                | Creates new timestamp file content (base64 text)           | Get Current Time1              | Upload New Timestamp2          | Timestamp Update (same as above)                                                                             |
| Upload New Timestamp2      | Google Drive        | Uploads the new timestamp file to Google Drive             | Create Timestamp File          |                                | Timestamp Update (same as above)                                                                             |
| Sticky Note               | Sticky Note          | Workflow purpose description                               |                                 |                                | Workflow Purpose (see above)                                                                                  |
| Sticky Note1              | Sticky Note          | Setup requirements for Google Drive credentials            |                                 |                                | Setup Required (see above)                                                                                    |
| Sticky Note2              | Sticky Note          | Explanation of timestamp file usage                         |                                 |                                | Timestamp File (see above)                                                                                    |
| Sticky Note3              | Sticky Note          | Explanation of timestamp update mechanism                   |                                 |                                | Timestamp Update (see above)                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 1 minute.

2. **Add Google Drive Node to Search for Timestamp File**  
   - Operation: Search files (resource: fileFolder)  
   - Query: `name = 'n8n_last_run.txt' and trashed = false`  
   - Return all results.  
   - Set Google Drive OAuth2 credentials.  
   - Connect Schedule Trigger → this node.

3. **Add Google Drive Node to Download Timestamp File**  
   - Operation: Download file  
   - File ID: Use expression referencing the ID from previous node: `{{$json.id}}`  
   - Use same Google Drive credentials.  
   - Connect Search for Timestamp File → this node.

4. **Add Extract From File Node**  
   - Operation: Extract text from file  
   - Connect Download Timestamp File → this node.

5. **Add Set Node (Edit Fields2)**  
   - Add field: `searchFromTimestamp` (string)  
   - Expression:  
     ```
     {{$json.data || new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString()}}
     ```  
   - Connect Extract Timestamp Text → this node.

6. **Add Google Drive Node to Search for New Files**  
   - Operation: Search files (resource: fileFolder)  
   - Query:  
     ```
     'FOLDER_ID' in parents and (createdTime > '{{ $json.searchFromTimestamp }}' OR modifiedTime > '{{ $json.searchFromTimestamp }}') and trashed = false
     ```  
     Replace `'FOLDER_ID'` with your target folder's ID.  
   - Return all results.  
   - Set Google Drive OAuth2 credentials.  
   - Connect Edit Fields2 → this node.

7. **Add If Node to Check for New Files**  
   - Condition: Check if `{{$json.id}}` is not empty (string not empty).  
   - Connect Search for New Files → this node.

8. **Add Google Drive Node to Download New Files**  
   - Operation: Download file  
   - File ID: `{{$json.id}}` from If node true branch.  
   - Use Google Drive credentials.  
   - Connect If (true) → this node.

9. **Add No Operation Node**  
   - No configuration needed.  
   - Connect If (false) → this node.

10. **Add Merge Node**  
    - Default merge settings.  
    - Connect Download New Files and No Operation → Merge node.

11. **Add Set Node (Edit Fields)**  
    - Configure to execute once.  
    - No specific fields needed (can be used for future expansions).  
    - Connect Merge → Edit Fields node.

12. **Add Google Drive Node to Delete Old Timestamp File**  
    - Operation: Delete file  
    - File ID: Reference file ID from Search for Timestamp File node:  
      ```
      {{$node["Search for Timestamp File1"].json.id}}
      ```  
    - Use Google Drive credentials.  
    - Connect Edit Fields → this node.

13. **Add DateTime Node to Get Current Time**  
    - Timezone: UTC  
    - No input fields selected.  
    - Connect Delete Old Timestamp File → this node.

14. **Add Code Node to Create Timestamp File**  
    - JavaScript:  
      ```javascript
      const dateString = $json['date'] || new Date().toISOString();

      return [{
        json: {},
        binary: {
          data: {
            data: Buffer.from(dateString).toString('base64'),
            mimeType: 'text/plain',
          }
        }
      }];
      ```  
    - Connect Get Current Time → this node.

15. **Add Google Drive Node to Upload New Timestamp File**  
    - Operation: Upload file  
    - Name: `n8n_last_run.txt`  
    - Drive: Select "My Drive" or your target drive.  
    - Folder ID: Your target folder ID (same as search folder).  
    - Use Google Drive credentials.  
    - Connect Create Timestamp File → this node.

16. **Verify All Connections and Credentials**  
    - Ensure all Google Drive nodes use the same OAuth2 credentials.  
    - Replace folder IDs with your actual Google Drive folder IDs.  
    - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow Purpose: This workflow automatically downloads new or updated files from Google Drive.    | Sticky Note node describing overall workflow goal.                                             |
| Setup Required: Google Drive OAuth2 credentials must be configured in n8n and selected in nodes.   | Sticky Note node with setup instructions.                                                      |
| Timestamp File: Uses a text file `n8n_last_run.txt` on Google Drive to track last execution time.  | Sticky Note describing timestamp file usage.                                                  |
| Timestamp Update: Workflow updates the timestamp file after processing new files to track progress.| Sticky Note explaining timestamp update step.                                                 |

---

**Disclaimer:** The provided text is an automated extraction from an n8n workflow and respects all current content policies. It contains no illegal, offensive, or protected content. All data manipulated is legal and publicly accessible.