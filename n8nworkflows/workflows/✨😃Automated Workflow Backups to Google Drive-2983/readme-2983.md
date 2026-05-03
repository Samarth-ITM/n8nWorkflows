✨😃Automated Workflow Backups to Google Drive

https://n8nworkflows.xyz/workflows/---automated-workflow-backups-to-google-drive-2983


# ✨😃Automated Workflow Backups to Google Drive

### 1. Workflow Overview

This workflow automates daily backups of all n8n workflows to Google Drive, organizing them into timestamped folders, managing retention by deleting backups older than seven days, and notifying the user via Telegram upon completion. It is designed for users who want reliable, automated backup and retention management of their n8n workflows with real-time status updates.

The workflow is logically divided into three main blocks:

- **1.1 Backup Trigger and Folder Creation:** Handles manual or scheduled triggering, generates a timestamp, and creates a new Google Drive folder for the backup.

- **1.2 Workflow Export and Upload:** Retrieves all workflows from n8n, converts each to a JSON file, and uploads them into the created Google Drive folder.

- **1.3 Backup Retention and Notification:** Searches existing backup folders, identifies those older than seven days, deletes them permanently, and sends a Telegram notification with backup details.

---

### 2. Block-by-Block Analysis

#### 1.1 Backup Trigger and Folder Creation

**Overview:**  
This block initiates the backup process either manually or on a daily schedule. It generates a current timestamp and creates a new Google Drive folder named with this timestamp to store the backup files.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Every Day (Scheduled Trigger)  
- Get DateTIme (Set Node)  
- Create Folder with DateTime Stamp (Google Drive Folder Creation)

**Node Details:**

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow.  
  - Configuration: Default manual trigger with no parameters.  
  - Input: None  
  - Output: Triggers "Get DateTIme" node.  
  - Edge Cases: User must manually trigger; no automatic scheduling here.

- **Every Day**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow daily (default interval).  
  - Configuration: Interval set to daily (default empty object in interval array).  
  - Input: None  
  - Output: Triggers "Get DateTIme" node.  
  - Edge Cases: Timezone considerations; configured timezone is America/Vancouver.

- **Get DateTIme**  
  - Type: Set  
  - Role: Assigns the current timestamp to a variable `datetime`.  
  - Configuration: Uses expression `{{$now}}` to get current ISO datetime string.  
  - Input: Trigger node output.  
  - Output: Passes `datetime` to "Create Folder with DateTime Stamp".  
  - Edge Cases: Time format consistency; relies on n8n system time.

- **Create Folder with DateTime Stamp**  
  - Type: Google Drive (Folder Creation)  
  - Role: Creates a new folder in Google Drive named `n8n-Workflow-Backups-{{datetime}}`.  
  - Configuration:  
    - Folder name uses expression to include timestamp.  
    - Drive ID set to "My Drive".  
    - Parent folder is root.  
    - OAuth2 credentials for Google Drive required.  
  - Input: Receives `datetime` from "Get DateTIme".  
  - Output: Passes folder metadata (including folder ID) to "Get Workflows".  
  - Edge Cases: Google Drive API errors (quota, permissions), folder name collisions unlikely due to timestamp.

---

#### 1.2 Workflow Export and Upload

**Overview:**  
This block fetches all workflows from the n8n instance, limits the number to 200 for performance, processes each workflow individually by converting it to a JSON file, and uploads each file to the newly created Google Drive folder.

**Nodes Involved:**  
- Get Workflows (n8n API)  
- Limit to 200 (Limit node)  
- Loop Over Items (SplitInBatches)  
- Execute Once (NoOp)  
- Convert Workflow to JSON File (ConvertToFile)  
- Save JSON File to Google Drive Folder (Google Drive File Upload)

**Node Details:**

- **Get Workflows**  
  - Type: n8n API  
  - Role: Retrieves all workflows from the n8n instance.  
  - Configuration: No filters; fetches all workflows.  
  - Credentials: n8n API credentials required.  
  - Input: Receives folder creation output.  
  - Output: Passes workflows array to "Limit to 200".  
  - Edge Cases: API authentication errors, large number of workflows.

- **Limit to 200**  
  - Type: Limit  
  - Role: Restricts the number of workflows processed to 200 to avoid overload.  
  - Configuration: Max items set to 200.  
  - Input: Workflows from "Get Workflows".  
  - Output: Passes limited workflows to "Loop Over Items".  
  - Edge Cases: If more than 200 workflows exist, only first 200 are backed up.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes workflows one by one in batches (default batch size 1).  
  - Configuration: Reset option disabled to maintain state.  
  - Input: Limited workflows.  
  - Output: Two outputs:  
    - First output triggers "Execute Once" (NoOp) once per batch.  
    - Second output sends each workflow to "Convert Workflow to JSON File".  
  - Edge Cases: Batch processing errors, potential delays with many workflows.

- **Execute Once**  
  - Type: NoOp  
  - Role: Ensures certain downstream nodes execute only once per batch cycle.  
  - Configuration: Execute once enabled.  
  - Input: From "Loop Over Items".  
  - Output: Triggers "Search Folder Names" in retention block.  
  - Edge Cases: None significant; used for flow control.

- **Convert Workflow to JSON File**  
  - Type: ConvertToFile  
  - Role: Converts each workflow JSON data into a downloadable JSON file.  
  - Configuration:  
    - Operation: toJson  
    - File name: Uses workflow name from JSON (`{{$json.name}}`).  
  - Input: Single workflow item.  
  - Output: Binary file data passed to "Save JSON File to Google Drive Folder".  
  - Edge Cases: Workflow JSON structure issues, file naming conflicts.

- **Save JSON File to Google Drive Folder**  
  - Type: Google Drive (File Upload)  
  - Role: Uploads the JSON file to the created Google Drive folder.  
  - Configuration:  
    - File name: Uses binary file name with `.json` extension.  
    - Folder ID: Dynamically set to the folder created earlier (`{{$node["Create Folder with DateTime Stamp"].json.id}}`).  
    - Drive ID: "My Drive".  
    - OAuth2 credentials required.  
  - Input: Binary file from "Convert Workflow to JSON File".  
  - Output: Loops back to "Loop Over Items" for next workflow.  
  - Edge Cases: Google Drive API upload errors, file overwrite conflicts.

---

#### 1.3 Backup Retention and Notification

**Overview:**  
This block manages backup retention by searching for existing backup folders, identifying those older than seven days, deleting them permanently, and finally sending a Telegram notification with backup completion details and a link to the new backup folder.

**Nodes Involved:**  
- Search Folder Names (Google Drive Search)  
- Find Folders to Delete (Code)  
- Delete Folders (Google Drive Delete)  
- Complete Message (Telegram Notification)

**Node Details:**

- **Search Folder Names**  
  - Type: Google Drive (Search)  
  - Role: Searches Google Drive for folders with names containing `n8n-Workflow-Backups`.  
  - Configuration:  
    - Search limited to 10 folders.  
    - Filter set to folders only.  
    - Query string: `n8n-Workflow-Backups`.  
    - OAuth2 credentials required.  
  - Input: Triggered once per execution (via "Execute Once" NoOp).  
  - Output: Passes found folders to "Find Folders to Delete".  
  - Edge Cases: API rate limits, incomplete search results if more than 10 folders exist.

- **Find Folders to Delete**  
  - Type: Code (JavaScript)  
  - Role: Sorts found folders by name descending, slices to keep only the most recent 7, and returns older folders for deletion.  
  - Configuration:  
    - Sorts by folder name (which includes date).  
    - Keeps first 7 folders, deletes the rest.  
  - Input: Folder list from "Search Folder Names".  
  - Output: List of folders to delete passed to "Delete Folders".  
  - Edge Cases: Folder name format must be consistent; if naming changes, sorting may fail.

- **Delete Folders**  
  - Type: Google Drive (Delete Folder)  
  - Role: Permanently deletes folders identified as older than 7 days.  
  - Configuration:  
    - Deletes permanently (`deletePermanently: true`).  
    - Folder ID dynamically set from input JSON.  
    - OAuth2 credentials required.  
  - Input: Folders to delete from "Find Folders to Delete".  
  - Output: Continues workflow; errors are caught and do not stop execution (`onError: continueRegularOutput`).  
  - Edge Cases: Permissions errors, API quota limits, non-existent folders.

- **Complete Message**  
  - Type: Telegram  
  - Role: Sends a notification message to the user indicating backup completion.  
  - Configuration:  
    - Message text includes current timestamp, backup folder name, and a clickable Google Drive folder link.  
    - Chat ID is read from environment variable `TELEGRAM_CHAT_ID`.  
    - Parse mode set to HTML for formatting.  
    - Attribution disabled.  
    - Telegram bot credentials required.  
  - Input: Triggered after "Search Folder Names" (via "Execute Once").  
  - Output: Final node; no further output.  
  - Edge Cases: Telegram API errors, invalid chat ID, network issues.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                          | Input Node(s)                      | Output Node(s)                                  | Sticky Note                                                                                      |
|--------------------------------|-----------------------|----------------------------------------|----------------------------------|------------------------------------------------|-------------------------------------------------------------------------------------------------|
| On clicking 'execute'           | Manual Trigger        | Manual workflow start                   | None                             | Get DateTIme                                   |                                                                                                 |
| Every Day                      | Schedule Trigger      | Scheduled daily workflow start          | None                             | Get DateTIme                                   |                                                                                                 |
| Get DateTIme                  | Set                   | Assign current timestamp                | On clicking 'execute', Every Day | Create Folder with DateTime Stamp               | ## Get DateTime Stamp                                                                            |
| Create Folder with DateTime Stamp | Google Drive (Folder) | Create timestamped backup folder        | Get DateTIme                    | Get Workflows                                  | ## Create NEW Google Folder                                                                      |
| Get Workflows                 | n8n API                | Retrieve all workflows                   | Create Folder with DateTime Stamp | Limit to 200                                   | ## Get All Workflows                                                                            |
| Limit to 200                 | Limit                  | Limit workflows to 200                   | Get Workflows                   | Loop Over Items                                | ## Limit for Debugging Remove this once you have it up and running                              |
| Loop Over Items              | SplitInBatches         | Process workflows one by one             | Limit to 200                   | Execute Once, Convert Workflow to JSON File    | ## Save Workflows to Google Drive                                                               |
| Execute Once                 | NoOp                   | Control node to execute downstream once | Loop Over Items                | Search Folder Names, Complete Message          | ## Keep Most Recent 7 Folders (Days) and Delete Others                                          |
| Convert Workflow to JSON File | ConvertToFile          | Convert workflow JSON to file            | Loop Over Items                | Save JSON File to Google Drive Folder          |                                                                                                 |
| Save JSON File to Google Drive Folder | Google Drive (File) | Upload JSON file to backup folder        | Convert Workflow to JSON File  | Loop Over Items                                |                                                                                                 |
| Search Folder Names          | Google Drive (Search)  | Find existing backup folders             | Execute Once                   | Find Folders to Delete                          | ## Keep Most Recent 7 Folders (Days) and Delete Others                                          |
| Find Folders to Delete       | Code                   | Identify folders older than 7 days       | Search Folder Names            | Delete Folders                                 | ## Keep Most Recent 7 Folders (Days) and Delete Others                                          |
| Delete Folders              | Google Drive (Delete)   | Permanently delete old backup folders    | Find Folders to Delete         | None                                           | ## Keep Most Recent 7 Folders (Days) and Delete Others                                          |
| Complete Message            | Telegram                | Notify user of backup completion         | Execute Once                   | None                                           | ## Notify User via Telegram                                                                      |
| Sticky Note                 | Sticky Note             | Visual comment                          | None                          | None                                           | ## Save Workflows to Google Drive                                                               |
| Sticky Note1                | Sticky Note             | Visual comment                          | None                          | None                                           | ## Keep Most Recent 7 Folders (Days) and Delete Others                                          |
| Sticky Note2                | Sticky Note             | Visual comment                          | None                          | None                                           | ## Notify User via Telegram                                                                      |
| Sticky Note3                | Sticky Note             | Visual comment                          | None                          | None                                           | ## Limit for Debugging Remove this once you have it up and running                              |
| Sticky Note4                | Sticky Note             | Visual comment                          | None                          | None                                           | ## Get All Workflows                                                                            |
| Sticky Note5                | Sticky Note             | Visual comment                          | None                          | None                                           | ## Create NEW Google Folder                                                                      |
| Sticky Note6                | Sticky Note             | Visual comment                          | None                          | None                                           | ## Get DateTime Stamp                                                                            |
| Sticky Note7                | Sticky Note             | Visual comment                          | None                          | None                                           | # ✨😃 Automated Workflow Backups to Google Drive (Full description and instructions)            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "On clicking 'execute'"  
   - No parameters.

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Every Day"  
   - Set interval to daily (default).  
   - Set timezone to America/Vancouver (workflow setting).

3. **Create Set Node for Timestamp**  
   - Type: Set  
   - Name: "Get DateTIme"  
   - Add field: `datetime` (string) with value `={{ $now }}`.

4. **Connect Manual Trigger and Schedule Trigger to "Get DateTIme"**  
   - Both trigger nodes connect to "Get DateTIme".

5. **Create Google Drive Folder Node**  
   - Type: Google Drive  
   - Name: "Create Folder with DateTime Stamp"  
   - Operation: Create Folder  
   - Folder Name: `n8n-Workflow-Backups-{{ $json.datetime }}`  
   - Drive ID: "My Drive"  
   - Parent Folder: root  
   - Credentials: Google Drive OAuth2 (configure with your Google account).

6. **Connect "Get DateTIme" to "Create Folder with DateTime Stamp"**

7. **Create n8n API Node to Get Workflows**  
   - Type: n8n API  
   - Name: "Get Workflows"  
   - No filters.  
   - Credentials: n8n API (configure with your n8n instance credentials).

8. **Connect "Create Folder with DateTime Stamp" to "Get Workflows"**

9. **Create Limit Node**  
   - Type: Limit  
   - Name: "Limit to 200"  
   - Max Items: 200

10. **Connect "Get Workflows" to "Limit to 200"**

11. **Create SplitInBatches Node**  
    - Type: SplitInBatches  
    - Name: "Loop Over Items"  
    - Options: Reset disabled (default).

12. **Connect "Limit to 200" to "Loop Over Items"**

13. **Create NoOp Node**  
    - Type: NoOp  
    - Name: "Execute Once"  
    - Enable "Execute Once" option.

14. **Connect first output of "Loop Over Items" to "Execute Once"**

15. **Create ConvertToFile Node**  
    - Type: ConvertToFile  
    - Name: "Convert Workflow to JSON File"  
    - Operation: toJson  
    - File Name: `={{ $json.name }}`

16. **Connect second output of "Loop Over Items" to "Convert Workflow to JSON File"**

17. **Create Google Drive File Upload Node**  
    - Type: Google Drive  
    - Name: "Save JSON File to Google Drive Folder"  
    - Operation: Upload File  
    - File Name: `={{ $binary.data.fileName }}.json`  
    - Folder ID: `={{ $node["Create Folder with DateTime Stamp"].json.id }}` (expression)  
    - Drive ID: "My Drive"  
    - Credentials: Google Drive OAuth2 (same as folder creation).

18. **Connect "Convert Workflow to JSON File" to "Save JSON File to Google Drive Folder"**

19. **Connect "Save JSON File to Google Drive Folder" back to "Loop Over Items"**  
    - This loops the batch processing.

20. **Create Google Drive Search Node**  
    - Type: Google Drive  
    - Name: "Search Folder Names"  
    - Resource: fileFolder  
    - Operation: Search  
    - Query String: `n8n-Workflow-Backups`  
    - Limit: 10  
    - Filter: folders only  
    - Credentials: Google Drive OAuth2.

21. **Connect "Execute Once" to "Search Folder Names"**

22. **Create Code Node**  
    - Type: Code  
    - Name: "Find Folders to Delete"  
    - Language: JavaScript  
    - Code:  
      ```javascript
      // Sort folders by name descending
      const sortedItems = $input.all().sort((a, b) => {
        if (!a.name || !b.name) return 0;
        return b.name.localeCompare(a.name);
      });
      // Keep only folders older than 7 days
      const olderItems = sortedItems.slice(7);
      return olderItems;
      ```

23. **Connect "Search Folder Names" to "Find Folders to Delete"**

24. **Create Google Drive Delete Folder Node**  
    - Type: Google Drive  
    - Name: "Delete Folders"  
    - Operation: Delete Folder  
    - Folder ID: `={{ $json.id }}` (from input)  
    - Delete Permanently: true  
    - Credentials: Google Drive OAuth2  
    - On Error: Continue Regular Output (to avoid workflow stop on errors).

25. **Connect "Find Folders to Delete" to "Delete Folders"**

26. **Create Telegram Node**  
    - Type: Telegram  
    - Name: "Complete Message"  
    - Text:  
      ```
      {{$now}}
      Workflows Backup Complete
      {{$node["Create Folder with DateTime Stamp"].json.name}}
      https://drive.google.com/drive/folders/{{$node["Create Folder with DateTime Stamp"].json.id}}
      ```  
    - Chat ID: `={{ $env.TELEGRAM_CHAT_ID }}` (environment variable)  
    - Additional Fields:  
      - Parse Mode: HTML  
      - Append Attribution: false  
    - Credentials: Telegram API (configure with your bot token).

27. **Connect "Execute Once" to "Complete Message"**

---

This completes the full workflow setup. Test by manually triggering or waiting for the scheduled run. Verify folder creation, file uploads, deletion of old folders, and Telegram notifications.

---

**Note:**  
- Ensure all credentials (Google Drive OAuth2, n8n API, Telegram API) are properly configured in n8n.  
- Adjust retention period by modifying the slice index in the "Find Folders to Delete" code node.  
- Modify schedule trigger as needed for different backup frequencies.  
- Telegram chat ID must be set as environment variable `TELEGRAM_CHAT_ID`.