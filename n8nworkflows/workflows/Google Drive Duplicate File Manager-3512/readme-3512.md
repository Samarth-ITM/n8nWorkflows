Google Drive Duplicate File Manager

https://n8nworkflows.xyz/workflows/google-drive-duplicate-file-manager-3512


# Google Drive Duplicate File Manager

### 1. Workflow Overview

This workflow, **Google Drive Duplicate File Manager**, automates the detection and management of duplicate files within a specified Google Drive folder. It is designed for individuals or teams seeking to optimize their Google Drive storage by removing or flagging duplicate files automatically, saving space and reducing confusion.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Configuration:** Watches a Google Drive folder for new files and sets parameters for duplicate handling.
- **1.2 File Retrieval and Filtering:** Retrieves files from the target folder and filters out Google Apps files which cannot be deduplicated by content.
- **1.3 Duplicate Detection:** Identifies duplicates based on file content hashes (MD5Checksum), with options to keep either the first or last uploaded file.
- **1.4 Duplicate Handling:** Depending on configuration, either moves duplicates to trash or renames them with a "DUPLICATE-" prefix.
- **1.5 Final Filtering and No-Op:** Ensures files already flagged as duplicates are not renamed again and handles no-operation cases.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration

**Overview:**  
This block initiates the workflow by monitoring a specified Google Drive folder for newly created files. It then sets configuration parameters that control how duplicates are handled (which file to keep and what action to take on duplicates).

**Nodes Involved:**  
- Google Drive Trigger  
- Config  
- Sticky Note5 (configuration explanation)  
- Sticky Note6 (working folder explanation)  
- Sticky Note7 (trigger settings explanation)

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node for Google Drive events  
  - Configuration: Watches for `fileCreated` events in a specific folder (default folder ID provided). Polling interval set to every 15 minutes.  
  - Credentials: Google Drive OAuth2  
  - Inputs: None (trigger)  
  - Outputs: Emits new file metadata when detected  
  - Edge cases: API quota limits, folder permission issues, large folder polling delays  
  - Notes: Using root folder as watch target is risky due to broad scope.

- **Config**  
  - Type: Set node to define workflow parameters  
  - Configuration: Sets four variables:  
    - `keep`: "last" (default) — whether to keep the last or first file among duplicates  
    - `action`: "flag" (default) — action on duplicates: "trash" or "flag" (rename)  
    - `owner`: email of file owner, derived from trigger data  
    - `folder`: folder ID to work with, derived from trigger data  
  - Inputs: From Google Drive Trigger  
  - Outputs: Configuration data for downstream nodes  
  - Edge cases: Missing or malformed trigger data could cause incorrect config values.

- **Sticky Notes 5, 6, 7**  
  - Provide detailed explanations on configuration parameters, working folder scope, and trigger setup.  
  - Important warnings about root folder usage and Google Apps file exclusion.

---

#### 2.2 File Retrieval and Filtering

**Overview:**  
This block retrieves all files owned by the specified user within the configured folder and filters out Google Apps files (Docs, Sheets, Slides, etc.) since their content cannot be hashed for duplication checks.

**Nodes Involved:**  
- Working Folder  
- Drop Google Apps files  
- Sticky Note (discard Google Apps files explanation)

**Node Details:**

- **Working Folder**  
  - Type: Google Drive node (fileFolder resource)  
  - Configuration: Queries files owned by the configured owner within the specified folder ID. Returns all matching files with all fields.  
  - Inputs: From Config node  
  - Outputs: List of files in the folder owned by the user  
  - Edge cases: API limits, permission errors, empty folder results

- **Drop Google Apps files**  
  - Type: Filter node  
  - Configuration: Filters out files whose MIME type starts with `"application/vnd.google-apps"`, effectively removing Google Docs, Sheets, Slides, etc.  
  - Inputs: From Working Folder  
  - Outputs: Only binary files suitable for MD5 checksum comparison  
  - Edge cases: Files without MIME type or unexpected MIME types may cause false negatives or positives.

- **Sticky Note (Discard Google Apps files)**  
  - Explains rationale for filtering out Google Apps files.

---

#### 2.3 Duplicate Detection

**Overview:**  
This block identifies duplicate files by comparing their MD5Checksum values. It supports two modes: keeping the first uploaded file or the last uploaded file, marking the others as duplicates.

**Nodes Involved:**  
- Keep First/Last (Switch)  
- Deduplicate Keep Last (Code)  
- Deduplicate Keep First (Code)

**Node Details:**

- **Keep First/Last**  
  - Type: Switch node  
  - Configuration: Routes flow based on the `keep` parameter from Config node (`last` or `first`).  
  - Inputs: From Drop Google Apps files  
  - Outputs: To either Deduplicate Keep Last or Deduplicate Keep First  
  - Edge cases: Unexpected `keep` values cause no routing; ensure parameter validation.

- **Deduplicate Keep Last**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Sorts files by creation time descending (latest first)  
    - Iterates files, marking duplicates if MD5Checksum was seen before  
    - Files missing MD5Checksum are marked as not duplicates (failsafe)  
  - Inputs: From Keep First/Last (last output)  
  - Outputs: Files with `isDuplicate` boolean flag  
  - Edge cases: Missing or invalid MD5Checksum, date parsing errors

- **Deduplicate Keep First**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Sorts files by creation time ascending (oldest first)  
    - Marks duplicates similarly to Deduplicate Keep Last but reversed logic  
    - Same failsafes as above  
  - Inputs: From Keep First/Last (first output)  
  - Outputs: Files with `isDuplicate` flag  
  - Edge cases: Same as Deduplicate Keep Last

---

#### 2.4 Duplicate Handling

**Overview:**  
This block processes files marked as duplicates according to the configured action: either sending duplicates to trash or flagging them by renaming with a "DUPLICATE-" prefix.

**Nodes Involved:**  
- Edit Fields  
- Filter  
- Trash/Flag Duplicates (Switch)  
- Send Duplicates to Trash  
- Is Flagged (If)  
- Google Drive (Rename)  
- No Operation, do nothing  
- Sticky Note1 (flagging note)  
- Sticky Note2 (trash retention note)

**Node Details:**

- **Edit Fields**  
  - Type: Set node  
  - Configuration: Passes through relevant fields (`isDuplicate`, `id`, `name`) for clarity and downstream use  
  - Inputs: From Deduplicate nodes  
  - Outputs: To Filter node  
  - Edge cases: None significant

- **Filter**  
  - Type: Filter node  
  - Configuration: Filters only files where `isDuplicate` is true  
  - Inputs: From Edit Fields  
  - Outputs: To Trash/Flag Duplicates  
  - Edge cases: Files missing `isDuplicate` field may be excluded

- **Trash/Flag Duplicates**  
  - Type: Switch node  
  - Configuration: Routes based on `action` parameter (`trash` or `flag`) from Config node  
  - Inputs: From Filter  
  - Outputs: To Send Duplicates to Trash or Is Flagged  
  - Edge cases: Unexpected `action` values cause no routing

- **Send Duplicates to Trash**  
  - Type: Google Drive node  
  - Configuration: Deletes file by ID (moves to trash)  
  - Inputs: From Trash/Flag Duplicates (trash output)  
  - Credentials: Google Drive OAuth2  
  - Edge cases: API quota, permission errors, file already deleted

- **Is Flagged**  
  - Type: If node  
  - Configuration: Checks if file name already starts with "DUPLICATE-" to avoid double flagging  
  - Inputs: From Trash/Flag Duplicates (flag output)  
  - Outputs:  
    - True: To No Operation (skip renaming)  
    - False: To Google Drive rename node  
  - Edge cases: Case sensitivity, files with similar prefixes

- **Google Drive (Rename)**  
  - Type: Google Drive node  
  - Configuration: Updates file name by prefixing "DUPLICATE-"  
  - Inputs: From Is Flagged (false output)  
  - Credentials: Google Drive OAuth2  
  - Edge cases: Naming conflicts, API errors

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Configuration: Does nothing; used to terminate flow for already flagged files  
  - Inputs: From Is Flagged (true output)  
  - Outputs: None

- **Sticky Note1**  
  - Notes that files already starting with "DUPLICATE-" are not flagged again.

- **Sticky Note2**  
  - Explains that trashed files remain recoverable for 30 days before permanent deletion.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                              | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                              |
|-------------------------|-------------------------|----------------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Google Drive Trigger     | Google Drive Trigger    | Triggers workflow on new file creation       | None                       | Config                      | Sticky Note7: Explains trigger settings and folder scope warnings.                                     |
| Config                  | Set                     | Sets workflow parameters (keep, action, owner, folder) | Google Drive Trigger       | Working Folder              | Sticky Note5: Details configuration parameters and their usage.                                        |
| Working Folder          | Google Drive            | Retrieves files owned by user in target folder | Config                     | Drop Google Apps files      | Sticky Note6: Explains working folder and folder depth limitations.                                    |
| Drop Google Apps files   | Filter                  | Filters out Google Apps files (Docs, Sheets, etc.) | Working Folder             | Keep First/Last             | Sticky Note: Explains why Google Apps files are discarded.                                             |
| Keep First/Last          | Switch                  | Routes based on keep parameter ("first" or "last") | Drop Google Apps files      | Deduplicate Keep Last, Deduplicate Keep First |                                                                                                        |
| Deduplicate Keep Last    | Code                    | Marks duplicates keeping last uploaded file  | Keep First/Last (last)      | Edit Fields                 |                                                                                                        |
| Deduplicate Keep First   | Code                    | Marks duplicates keeping first uploaded file | Keep First/Last (first)     | Edit Fields                 |                                                                                                        |
| Edit Fields             | Set                     | Passes relevant fields for filtering          | Deduplicate Keep Last, Deduplicate Keep First | Filter                      |                                                                                                        |
| Filter                  | Filter                  | Filters only files marked as duplicates       | Edit Fields                 | Trash/Flag Duplicates       |                                                                                                        |
| Trash/Flag Duplicates    | Switch                  | Routes based on action parameter ("trash" or "flag") | Filter                     | Send Duplicates to Trash, Is Flagged |                                                                                                        |
| Send Duplicates to Trash | Google Drive            | Moves duplicate files to trash                 | Trash/Flag Duplicates (trash) | None                       | Sticky Note2: Explains 30-day retention of trashed files.                                              |
| Is Flagged               | If                      | Checks if file is already flagged as duplicate | Trash/Flag Duplicates (flag) | No Operation, Google Drive (Rename) | Sticky Note1: Notes files starting with "DUPLICATE-" are not flagged again.                            |
| Google Drive (Rename)    | Google Drive            | Renames duplicate files by prefixing "DUPLICATE-" | Is Flagged (false)          | None                       |                                                                                                        |
| No Operation, do nothing | NoOp                    | Terminates flow for already flagged files     | Is Flagged (true)           | None                       |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Polling interval: Every 15 minutes (default)  
   - Folder to watch: Select specific folder by ID or root folder (use caution)  
   - Credentials: Configure Google Drive OAuth2 credentials

2. **Create Config Node (Set)**  
   - Type: Set  
   - Assign variables:  
     - `keep`: `"last"` (string) — choose `"first"` or `"last"` to keep first or last file  
     - `action`: `"flag"` (string) — choose `"trash"` or `"flag"` for duplicate handling  
     - `owner`: Expression `{{$json.owner || $json.owners[0].emailAddress}}` (email of file owner)  
     - `folder`: Expression `{{$json.folder || $json.parents[0]}}` (folder ID)  
   - Connect input from Google Drive Trigger

3. **Create Working Folder Node (Google Drive)**  
   - Type: Google Drive  
   - Resource: `fileFolder`  
   - Operation: `list` or `search` files  
   - Query: Files owned by `owner` in folder `folder` (use expression referencing Config node)  
   - Return all results, include all fields  
   - Credentials: Google Drive OAuth2  
   - Connect input from Config node

4. **Create Drop Google Apps files Node (Filter)**  
   - Type: Filter  
   - Condition: Exclude files where `mimeType` starts with `"application/vnd.google-apps"`  
   - Connect input from Working Folder node

5. **Create Keep First/Last Node (Switch)**  
   - Type: Switch  
   - Rules:  
     - Output "last" if `keep` parameter equals `"last"`  
     - Output "first" if `keep` parameter equals `"first"`  
   - Connect input from Drop Google Apps files node

6. **Create Deduplicate Keep Last Node (Code)**  
   - Type: Code  
   - JavaScript logic:  
     - Sort files descending by `createdTime` (latest first)  
     - Iterate files, mark `isDuplicate` true if MD5Checksum seen before  
     - Skip files missing MD5Checksum, mark as not duplicate  
   - Connect input from Keep First/Last node's "last" output

7. **Create Deduplicate Keep First Node (Code)**  
   - Type: Code  
   - JavaScript logic:  
     - Sort files ascending by `createdTime` (oldest first)  
     - Mark duplicates similarly but reversed logic  
     - Same failsafe for missing MD5Checksum  
   - Connect input from Keep First/Last node's "first" output

8. **Create Edit Fields Node (Set)**  
   - Type: Set  
   - Assign fields: `isDuplicate`, `id`, `name` from input JSON  
   - Connect inputs from both Deduplicate nodes

9. **Create Filter Node**  
   - Type: Filter  
   - Condition: Only pass items where `isDuplicate` is true  
   - Connect input from Edit Fields node

10. **Create Trash/Flag Duplicates Node (Switch)**  
    - Type: Switch  
    - Rules:  
      - Output "send to trash" if `action` equals `"trash"`  
      - Output "flag as duplicate" if `action` equals `"flag"`  
    - Connect input from Filter node

11. **Create Send Duplicates to Trash Node (Google Drive)**  
    - Type: Google Drive  
    - Operation: `deleteFile`  
    - File ID: Use expression `{{$json.id}}`  
    - Credentials: Google Drive OAuth2  
    - Connect input from Trash/Flag Duplicates node's "send to trash" output

12. **Create Is Flagged Node (If)**  
    - Type: If  
    - Condition: Check if file name starts with `"DUPLICATE-"`  
    - Expression: `{{$json.name.startsWith("DUPLICATE-")}}`  
    - Connect input from Trash/Flag Duplicates node's "flag as duplicate" output

13. **Create No Operation Node (NoOp)**  
    - Type: NoOp  
    - Connect input from Is Flagged node's true output (already flagged files)

14. **Create Google Drive Rename Node**  
    - Type: Google Drive  
    - Operation: `update`  
    - File ID: `{{$json.id}}`  
    - New file name: `"DUPLICATE-" + {{$json.name}}`  
    - Credentials: Google Drive OAuth2  
    - Connect input from Is Flagged node's false output (files not yet flagged)

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Google Drive API limits apply; be mindful of quota and rate limits when running this workflow frequently.      | General API usage considerations                                                                |
| When configured to watch the root folder, the workflow processes all files in all subfolders—use with caution. | Sticky Note7 warning about root folder usage                                                    |
| Google Apps files (Docs, Sheets, Slides, Forms, Drawings) are excluded because their content cannot be hashed. | Sticky Note explaining filtering rationale                                                      |
| Files already prefixed with "DUPLICATE-" are not flagged again to avoid repeated renaming.                     | Sticky Note1                                                                                     |
| Trashed files remain recoverable for 30 days before permanent deletion, allowing review and restoration.       | Sticky Note2                                                                                     |
| Workflow designed to handle one folder depth only; nested subfolder files are not processed by default.        | Sticky Note6                                                                                     |
| For more info on Google Drive API and file metadata, see: https://developers.google.com/drive/api/v3/reference/files | External resource for Google Drive API documentation                                            |

---

This structured documentation enables users and automation agents to fully understand, reproduce, and safely modify the Google Drive Duplicate File Manager workflow.