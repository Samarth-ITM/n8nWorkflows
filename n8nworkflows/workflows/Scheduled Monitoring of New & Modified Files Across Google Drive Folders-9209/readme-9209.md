Scheduled Monitoring of New & Modified Files Across Google Drive Folders

https://n8nworkflows.xyz/workflows/scheduled-monitoring-of-new---modified-files-across-google-drive-folders-9209


# Scheduled Monitoring of New & Modified Files Across Google Drive Folders

### 1. Workflow Overview

This workflow monitors a specified root folder in Google Drive and recursively traverses all its nested subfolders to detect new or modified files since the last scheduled execution. It is designed for scheduled monitoring of file changes across a complex folder hierarchy, outputting batches of recently created or updated files.

The workflow consists of the following logical blocks:

- **1.1 Scheduling & Initialization**: Triggers the workflow periodically and initializes timing data for change detection.
- **1.2 Root Folder Listing & First-Level Recursion Trigger**: Lists subfolders and files in the root folder and triggers recursive processing of each subfolder.
- **1.3 Recursive Folder Traversal**: Iteratively lists subfolders and files for each nested folder, invoking itself recursively to traverse multiple levels.
- **1.4 File Filtering & Output**: Filters files created or modified since the last execution and outputs these as the workflow’s result.
- **1.5 Control Flow & Edge Case Handling**: Manages conditional logic for folder existence, recursion termination, and workflow exit paths.
- **1.6 Metadata & Documentation**: Sticky notes provide instructions, configuration guidance, and project credits.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Initialization

**Overview:**  
This block triggers the workflow on a fixed interval (hourly) and sets a timestamp marking the start of the current execution to compare file modification dates later.

**Nodes Involved:**  
- Schedule Trigger  
- SetStartTime

**Node Details:**

- **Schedule Trigger**  
  - *Type*: Schedule Trigger  
  - *Configuration*: Runs every 60 minutes (hourly)  
  - *Inputs*: None (trigger node)  
  - *Outputs*: Initiates folder listing  
  - *Edge Cases*: Trigger misfire, time drift, or disabled node would halt automation.

- **SetStartTime**  
  - *Type*: Code  
  - *Role*: Stores the timestamp of the current execution start into global static data for later reference  
  - *Key Logic*: Retrieves timestamp from trigger event, stores as `lastScheduleExecutionStarted` in global static data  
  - *Inputs*: None (triggered after file comparison wait)  
  - *Outputs*: Timestamp object for downstream use if needed  
  - *Edge Cases*: Failure to write static data could cause inaccurate filtering of changed files.

---

#### 2.2 Root Folder Listing & First-Level Recursion Trigger

**Overview:**  
Lists all subfolders and files within the designated root folder, then triggers recursion into each subfolder.

**Nodes Involved:**  
- List folders in root  
- List all files in root  
- Trigger recursion of first level  
- If folders exist  
- Exit sub workflow (fallback)

**Node Details:**

- **List folders in root**  
  - *Type*: Google Drive (fileFolder resource)  
  - *Role*: Lists all folders at the root folder level  
  - *Configuration*: Folder ID set as the root folder (user must configure)  
  - *Outputs*: Array of folder metadata to evaluate existence and recurse into  
  - *Edge Cases*: Folder ID invalid, auth failures, API rate limits.

- **If folders exist**  
  - *Type*: If  
  - *Role*: Checks if the folder list is not empty (folders exist)  
  - *Input*: Output from "List folders in root"  
  - *Outputs*:  
    - True: Proceed to recursion trigger  
    - False: Exit sub workflow  
  - *Edge Cases*: Empty root folder, malformed data.

- **Exit sub workflow**  
  - *Type*: Set  
  - *Role*: Sets a status message "No folders found" and ends recursion if no subfolders exist  
  - *Input*: False branch of "If folders exist"  
  - *Output*: Status object for logging or downstream checks

- **Trigger recursion of first level**  
  - *Type*: Execute Workflow (self-reference)  
  - *Role*: Calls this same workflow recursively for each root subfolder  
  - *Input*: Folder metadata from "List folders in root"  
  - *Parameters*: Passes folder info for processing  
  - *Edge Cases*: Infinite recursion if misconfigured, performance degradation on large folder trees

- **List all files in root**  
  - *Type*: Google Drive (fileFolder resource)  
  - *Role*: Lists all files directly under the root folder  
  - *Configuration*: Folder ID set as root folder (user-configured)  
  - *Output*: Files metadata for root folder  
  - *Edge Cases*: API limits, invalid folder ID

---

#### 2.3 Recursive Folder Traversal

**Overview:**  
This block handles recursive traversal of nested subfolders. For each folder, it lists subfolders and files, and if subfolders exist, triggers recursion again.

**Nodes Involved:**  
- Loop all subfolders  
- List folders  
- List all files  
- If subfolders exist  
- Trigger recursion to next level

**Node Details:**

- **Loop all subfolders**  
  - *Type*: SplitInBatches  
  - *Role*: Processes subfolders one batch at a time to avoid memory or API overload  
  - *Input*: Folder list (from recursion triggers)  
  - *Output*: Single subfolder per batch for processing  
  - *Edge Cases*: Large folder counts could cause API throttling or delays

- **List folders**  
  - *Type*: Google Drive (fileFolder resource)  
  - *Role*: Lists nested subfolders inside the current folder batch  
  - *Input*: Folder ID from batch item  
  - *Output*: Subfolder metadata array  
  - *Edge Cases*: Invalid folder ID, empty subfolder list

- **List all files**  
  - *Type*: Google Drive (fileFolder resource)  
  - *Role*: Lists all files inside the current folder batch  
  - *Input*: Folder ID from batch item  
  - *Output*: Files metadata array  
  - *Edge Cases*: API rate limiting, empty folder

- **If subfolders exist**  
  - *Type*: If  
  - *Role*: Checks if subfolders were found in the current folder  
  - *Input*: Output of "List folders"  
  - *Outputs*:  
    - True: Trigger recursion again  
    - False: Continue processing files only  
  - *Edge Cases*: False negatives due to API errors

- **Trigger recursion to next level**  
  - *Type*: Execute Workflow (self-reference)  
  - *Role*: Calls the workflow recursively for each nested subfolder detected  
  - *Input*: Subfolder metadata  
  - *Edge Cases*: Deep recursion depth may cause performance issues or stack limits

---

#### 2.4 File Filtering & Output

**Overview:**  
After collecting files recursively, this block filters files created or modified since the last scheduled execution and outputs the filtered list.

**Nodes Involved:**  
- Outputs new or updated files  
- Wait for file comparison to complete

**Node Details:**

- **Outputs new or updated files**  
  - *Type*: Code  
  - *Role*: Filters files by comparing their `modifiedTime` and `createdTime` against the last execution timestamp  
  - *Logic*:  
    - Retrieves `lastScheduleExecutionStarted` from global static data  
    - If not present (first run), sets fallback date far in the past to include all files  
    - Returns files with modification or creation timestamps later than this date  
  - *Input*: Files metadata from folder listing nodes  
  - *Output*: Filtered list of recently changed files  
  - *Edge Cases*: Incorrect timestamps, missing metadata fields, timezone issues

- **Wait for file comparison to complete**  
  - *Type*: Wait  
  - *Role*: Provides a webhook wait step before setting start time for next run, ensuring asynchronous completion  
  - *Input*: After files are listed and filtered  
  - *Output*: Triggers "SetStartTime" node  
  - *Edge Cases*: Timeout, webhook communication failure

---

#### 2.5 Control Flow & Edge Case Handling

**Overview:**  
Manages decision-making to prevent recursion when no folders exist and provides clear exit paths.

**Nodes Involved:**  
- If folders exist  
- Exit sub workflow

**Node Details:**  
Described above in 2.2.

---

#### 2.6 Metadata & Documentation

**Overview:**  
Sticky Note nodes provide user instructions, configuration points, and project credits.

**Nodes Involved:**  
- Sticky Note (root folder instruction)  
- Sticky Note1 (recursive iteration explanation)  
- Sticky Note2 (root folder setting reminder)  
- Sticky Note3 (output behavior explanation)  
- Sticky Note4 (credits and usage instructions)

**Node Details:**

- Each sticky note conveys essential configuration instructions or background information for users setting up or modifying the workflow.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                                | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------------|------------------------|-----------------------------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger       | Initiate workflow on fixed interval           | None                          | List folders in root            |                                                                                                |
| List folders in root          | Google Drive           | List subfolders at root folder                 | Schedule Trigger               | If folders exist, Trigger recursion of first level |                                                                                                |
| If folders exist              | If                     | Check if root subfolders exist                 | List folders in root           | Loop all subfolders / Exit sub workflow |                                                                                                |
| Exit sub workflow             | Set                    | Set status message if no folders found         | If folders exist (False)       | None                           |                                                                                                |
| Trigger recursion of first level | Execute Workflow       | Recursively process each root subfolder        | List folders in root           | List all files in root          |                                                                                                |
| List all files in root        | Google Drive           | List files in root folder                       | Trigger recursion of first level | Outputs new or updated files, Wait for file comparison to complete |                                                                                                |
| Outputs new or updated files  | Code                   | Filter files changed since last run            | List all files in root         | None                           | Outputs batches of files created or updated since the last PRODUCTION execution                 |
| Wait for file comparison to complete | Wait                   | Wait before setting new start time             | Outputs new or updated files   | SetStartTime                   |                                                                                                |
| SetStartTime                 | Code                   | Store current execution start timestamp        | Wait for file comparison to complete | None                           |                                                                                                |
| Loop all subfolders           | SplitInBatches         | Process subfolders one at a time                | If folders exist, If subfolders exist | List folders, List all files   |                                                                                                |
| List folders                 | Google Drive           | List nested subfolders of current folder        | Loop all subfolders            | If subfolders exist             |                                                                                                |
| List all files               | Google Drive           | List files of current folder                     | Loop all subfolders            | Outputs new or updated files    |                                                                                                |
| If subfolders exist          | If                     | Check if nested subfolders exist                 | List folders                  | Trigger recursion to next level / Loop all subfolders |                                                                                                |
| Trigger recursion to next level | Execute Workflow       | Recursive call for each nested subfolder         | If subfolders exist            | Loop all subfolders             |                                                                                                |
| Start recursion             | Execute Workflow Trigger | Entry point for manual or initial start          | None                          | If folders exist               |                                                                                                |
| Sticky Note                 | Sticky Note            | Instruction: Sets root folder to monitor and trigger recursive workflow | None                          | None                           | Sets root folder to monitor and trigger recursive workflow                                      |
| Sticky Note1                | Sticky Note            | Instruction: Recursively iterates through nested folders | None                          | None                           | Recursively iterates through all nested folders and list their files                            |
| Sticky Note2                | Sticky Note            | Instruction: Define root folder in two Google Drive nodes | None                          | None                           | Define root folder in both of these                                                           |
| Sticky Note3                | Sticky Note            | Explanation: Outputs batches of changed files    | None                          | None                           | Outputs batches of files created or updated since the last PRODUCTION execution                 |
| Sticky Note4                | Sticky Note            | Credits and instructions                          | None                          | None                           | Template created by Smultron Studio (https://smultronstudio.com/en) - email hello@smultronstudio.com |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run every 60 minutes (hourly interval)  
   - No credentials required

2. **Add Google Drive Node: List folders in root**  
   - Resource: fileFolder  
   - Operation: List  
   - Filter: `folderId` set to your root folder’s ID (must be user-defined)  
   - `whatToSearch`: folders  
   - Return all results  
   - Connect Schedule Trigger output to this node  
   - Connect your Google Drive credentials  

3. **Add an If Node: If folders exist**  
   - Condition: Check if input data is not empty (use expression to test array length or object non-empty)  
   - Connect output of "List folders in root" to this node

4. **Add a Set Node: Exit sub workflow**  
   - Assign a string field `status` with value "No folders found"  
   - Connect the False branch of "If folders exist" to this node

5. **Add Execute Workflow Node: Trigger recursion of first level**  
   - Set to call the current workflow by ID  
   - Map the input folder data (e.g., folder ID, name) as required by the recursion input  
   - Set to wait for subworkflow to complete (optional)  
   - Connect the True branch of "If folders exist" to this node

6. **Add Google Drive Node: List all files in root**  
   - Resource: fileFolder  
   - Operation: List  
   - Filter: `folderId` set to root folder ID (same as folders node)  
   - `whatToSearch`: files  
   - Return all results  
   - Connect output of "Trigger recursion of first level" to this node

7. **Add Code Node: Outputs new or updated files**  
   - Use JavaScript to filter files based on creation and modification timestamps compared to last execution timestamp stored in static data.  
   - If no timestamp exists (first run), fallback to very old date to output all files.  
   - Connect output of "List all files in root" to this node

8. **Add Wait Node: Wait for file comparison to complete**  
   - No parameters needed (default wait)  
   - Connect output of "Outputs new or updated files" to this node

9. **Add Code Node: SetStartTime**  
   - Store current execution start timestamp from the Schedule Trigger into global static data (e.g., `lastScheduleExecutionStarted`)  
   - Connect output of "Wait for file comparison to complete" to this node

10. **Recursive Traversal Setup: Loop all subfolders**  
    - Add SplitInBatches node to process subfolders one by one  
    - Connect output of "If folders exist" (True branch) or "Trigger recursion to next level" node to this node

11. **Add Google Drive Node: List folders**  
    - For the current folder batch, list its subfolders  
    - Connect output of "Loop all subfolders" to this node

12. **Add Google Drive Node: List all files**  
    - For the current folder batch, list its files  
    - Connect output of "Loop all subfolders" to this node

13. **Add If Node: If subfolders exist**  
    - Check if the "List folders" output is non-empty  
    - Connect output of "List folders" to this node

14. **Add Execute Workflow Node: Trigger recursion to next level**  
    - Call the current workflow recursively for each nested subfolder  
    - Connect True branch of "If subfolders exist" to this node  
    - Connect False branch to "Loop all subfolders" to continue processing files

15. **Connect outputs of "List all files" (recursive) to "Outputs new or updated files" node**  
    - This consolidates files across all folders

16. **Add Sticky Notes at appropriate positions** for user instructions:  
    - Indicate where to set root folder IDs in Google Drive nodes  
    - Explain recursion and output behavior  
    - Add credits and configuration tips

17. **Configure Credentials**  
    - Add Google Drive OAuth2 credentials for all Google Drive nodes  
    - Test connections to ensure access to specified folders

18. **Test workflow manually first** to verify recursive listing and file filtering works correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Template created by Smultron Studio - contact: hello@smultronstudio.com                                                            | https://smultronstudio.com/en                    |
| Upon first execution (or manual runs before first production run), all files in all folders will be output as changed               | Important usage note for expected initial output|
| Workflow may have performance issues with very large folder structures due to recursive API calls to Google Drive                  | Consider schedule interval and folder scope     |
| Set your root folder ID in both "List folders in root" and "List all files in root" Google Drive nodes before activating the workflow | Critical configuration step                       |

---

**Disclaimer:**  
The provided text is derived solely from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.