Recursive Google Drive Folder Duplicator with Permission Preservation

https://n8nworkflows.xyz/workflows/recursive-google-drive-folder-duplicator-with-permission-preservation-7402


# Recursive Google Drive Folder Duplicator with Permission Preservation

### 1. Workflow Overview

This workflow provides a fully recursive solution to duplicate a Google Drive folder, including all nested subfolders and files, while preserving sharing permissions and folder structure. It is designed for use cases such as backing up folder hierarchies, duplicating templates, archiving projects, or migrating data with maintained access controls.

The workflow is logically divided into the following blocks:

- **1.1 Configuration Setup:** Defines input parameters such as the source folder ID, target parent folder ID, and the new main folder name to create.
- **1.2 Main Folder Creation:** Creates the top-level duplicate folder in the target location.
- **1.3 Recursive Processing Engine:** Recursively traverses the source folder hierarchy by:
  - Scanning folder contents (files and subfolders)
  - Separating files from folders
  - Copying files and applying original permissions
  - Creating subfolders and applying their permissions
  - Recursively invoking itself for each subfolder found until all nested levels are processed
- **1.4 Completion Notification:** Marks the successful duplication of the main folder and reports key IDs.

### 2. Block-by-Block Analysis

---

#### Block 1.1: Configuration Setup

**Overview:**  
This block initializes the workflow by setting essential parameters needed for the duplication process: the source folder to copy, the target parent folder where the copy will reside, and the name for the new duplicated folder.

**Nodes Involved:**  
- Set Configuration

**Node Details:**  

- **Set Configuration**  
  - Type: Set node (parameter initialization)  
  - Configuration: Defines three string parameters:
    - `sourceFolderId`: ID of the folder to duplicate (default: `"1RdenWIrighEr6S53e25CkwYmUa7jAFjq"`)
    - `targetFolderId`: ID of the parent folder where the duplicated folder will be created (default: `"13BQRVwg2QFGFRJstRzUt0CoO5w8bpS1I"`)
    - `newFolderName`: Name for the duplicated folder (default: `"Duplicated Folder"`)  
  - Inputs: None (start node)  
  - Outputs: Connects to "Create Main Folder"  
  - Edge Cases: Misconfigured folder IDs or insufficient permissions will cause failure downstream.

---

#### Block 1.2: Main Folder Creation

**Overview:**  
Creates the top-level duplicated folder under the target parent folder using the configured name.

**Nodes Involved:**  
- Create Main Folder

**Node Details:**  

- **Create Main Folder**  
  - Type: Google Drive node (Folder Create operation)  
  - Configuration:
    - `name`: Uses `newFolderName` set in previous node.
    - `driveId`: Set to `targetFolderId` from configuration.
    - `folderId`: Set to `"root"` of the target drive.
    - Resource: Folder  
  - Credentials: Uses configured Google Drive OAuth2 credentials with full drive access.  
  - Inputs: From "Set Configuration"  
  - Outputs: Connects to "Start Recursive Processing"  
  - Edge Cases: API quota limits, invalid drive or folder ID, or permission issues may cause failures.

---

#### Block 1.3: Recursive Processing Engine

**Overview:**  
This block contains the core logic for recursively duplicating folder contents. It repeatedly scans the contents of a source folder, separates files and folders, copies each while preserving permissions, and recursively processes subfolders.

**Nodes Involved:**  
- Start Recursive Processing  
- Search Folder Contents1  
- Filter Files Only1  
- Filter Folders Only1  
- Copy File1  
- Prepare File Data  
- Check File Has Permissions1  
- Prepare File Permissions1  
- Apply File Permission1  
- Create Subfolder1  
- Prepare Folder Data1  
- Check Folder Has Permissions1  
- Prepare Folder Permissions1  
- Apply Folder Permission1  
- Recursive Call to Self  
- Level Completion  

**Node Details:**  

- **Start Recursive Processing**  
  - Type: Execute Workflow (calls self)  
  - Configuration: Calls the current workflow with inputs:
    - `sourceFolderId`: From initial configuration or recursive parameters
    - `targetFolderId`: Newly created folder ID where contents will be copied  
  - Inputs: From "Create Main Folder" (initial call) or from "Apply Folder Permission1" (recursive calls)  
  - Outputs: Connects to "Main Completion" or "Level Completion"  
  - Edge Cases: Recursive depth limits, infinite loops if parameters misconfigured.

- **Search Folder Contents1**  
  - Type: Google Drive node (Search files and folders)  
  - Configuration:
    - Query: `'{{ $json.sourceFolderId }}' in parents and trashed = false`
    - Return all results, requesting fields: `id`, `name`, `mimeType`, `permissions`
  - Inputs: From recursive parameters  
  - Outputs: Connects to "Filter Files Only1" and "Filter Folders Only1"  
  - Edge Cases: Large folder contents may hit API or performance limits.

- **Filter Files Only1**  
  - Type: Filter node  
  - Configuration: Filters out only files (mimeType ≠ "application/vnd.google-apps.folder")  
  - Inputs: From "Search Folder Contents1"  
  - Outputs: Connects to "Copy File1"  
  - Edge Cases: Correct mimeType filtering is critical to avoid errors.

- **Filter Folders Only1**  
  - Type: Filter node  
  - Configuration: Filters only folders (mimeType = "application/vnd.google-apps.folder")  
  - Inputs: From "Search Folder Contents1"  
  - Outputs: Connects to "Create Subfolder1"  
  - Edge Cases: Same as above.

- **Copy File1**  
  - Type: Google Drive node (Copy file operation)  
  - Configuration:
    - Copies file with same name to the specified `driveId` and `folderId` (target folder)
  - Inputs: From "Filter Files Only1"  
  - Outputs: Connects to "Prepare File Data"  
  - Edge Cases: Copy failures if file is restricted or quota exceeded.

- **Prepare File Data**  
  - Type: Set node  
  - Configuration: Stores original file ID, permissions (stringified), and new file ID for permission application  
  - Inputs: From "Copy File1"  
  - Outputs: Connects to "Check File Has Permissions1"

- **Check File Has Permissions1**  
  - Type: If node  
  - Configuration: Checks if original permissions array length > 0 to decide if permissions should be applied  
  - Inputs: From "Prepare File Data"  
  - Outputs: 
    - True branch: to "Prepare File Permissions1"
    - False branch: to "Level Completion" (skip permissions)  
  - Edge Cases: Permissions array missing or malformed JSON.

- **Prepare File Permissions1**  
  - Type: Code node (JavaScript)  
  - Configuration: Parses permissions and builds permission objects excluding owners; handles user, group, domain types  
  - Inputs: From "Check File Has Permissions1"  
  - Outputs: Connects to "Apply File Permission1"  
  - Edge Cases: Permissions with unexpected properties; malformed JSON.

- **Apply File Permission1**  
  - Type: Google Drive node (Share operation)  
  - Configuration:
    - Applies permissions to new file ID without sending notification emails  
  - Inputs: From "Prepare File Permissions1"  
  - Outputs: Connects to "Level Completion"  
  - Edge Cases: Permission application can fail if lacking admin rights or due to API limits.

- **Create Subfolder1**  
  - Type: Google Drive node (Folder create)  
  - Configuration:
    - Creates a folder with the same name in the target folder  
  - Inputs: From "Filter Folders Only1"  
  - Outputs: Connects to "Prepare Folder Data1"  
  - Edge Cases: Duplicate folder names may cause confusion but allowed in Drive.

- **Prepare Folder Data1**  
  - Type: Set node  
  - Configuration: Stores original folder ID, permissions (stringified), and new folder ID for permission application  
  - Inputs: From "Create Subfolder1"  
  - Outputs: Connects to "Check Folder Has Permissions1"

- **Check Folder Has Permissions1**  
  - Type: If node  
  - Configuration: Checks if original folder permissions length > 0  
  - Inputs: From "Prepare Folder Data1"  
  - Outputs:
    - True branch: to "Prepare Folder Permissions1"
    - False branch: directly to "Recursive Call to Self" (skip permissions)  
  - Edge Cases: Same as file permissions check.

- **Prepare Folder Permissions1**  
  - Type: Code node (JavaScript)  
  - Configuration: Similar to file permissions preparation but applied to folders  
  - Inputs: From "Check Folder Has Permissions1"  
  - Outputs: Connects to "Apply Folder Permission1"  
  - Edge Cases: Same as file permissions preparation.

- **Apply Folder Permission1**  
  - Type: Google Drive node (Share operation)  
  - Configuration:
    - Applies permissions to the new folder ID without notification  
  - Inputs: From "Prepare Folder Permissions1"  
  - Outputs: Connects to "Recursive Call to Self"  
  - Edge Cases: Same as file permission application.

- **Recursive Call to Self**  
  - Type: Execute Workflow (self-call)  
  - Configuration:
    - Calls the same workflow with:
      - `sourceFolderId`: original subfolder ID
      - `targetFolderId`: new subfolder ID  
  - Inputs: From "Apply Folder Permission1" or "Check Folder Has Permissions1" (false branch)  
  - Outputs: Connects to "Level Completion"  
  - Edge Cases: Deep recursion can cause stack or runtime limits.

- **Level Completion**  
  - Type: Set node  
  - Configuration: Sets status message `"Level processing completed"`  
  - Inputs: From "Apply File Permission1", "Recursive Call to Self", or "Check File Has Permissions1" (false branch)  
  - Outputs: None (end of recursive loop step)

---

#### Block 1.4: Completion Notification

**Overview:**  
Marks end of the main workflow run, confirming that the top-level folder duplication succeeded.

**Nodes Involved:**  
- Main Completion

**Node Details:**  

- **Main Completion**  
  - Type: Set node  
  - Configuration: Sets status message `"Main folder duplication completed successfully"` and outputs key IDs for reference:
    - `sourceFolder`: original source folder ID
    - `newMainFolder`: ID of newly created main folder  
  - Inputs: From "Start Recursive Processing"  
  - Outputs: None (workflow completion)  

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                             | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                            |
|---------------------------|-------------------------|---------------------------------------------|---------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Set Configuration         | Set                     | Initialize source/target folder IDs and name | None                            | Create Main Folder              | Configuration Hub: sets sourceFolderId, targetFolderId, newFolderName                                                  |
| Create Main Folder        | Google Drive            | Create top-level duplicated folder           | Set Configuration               | Start Recursive Processing      |                                                                                                                        |
| Start Recursive Processing | Execute Workflow        | Recursive engine entry point                   | Create Main Folder              | Main Completion                | Core recursive engine: scans, splits, copies, creates, recurses                                                       |
| Main Completion           | Set                     | Final status and output                        | Start Recursive Processing      | None                          |                                                                                                                        |
| Search Folder Contents1    | Google Drive            | List contents of the current source folder   | Start Recursive Processing (recursive) | Filter Files Only1, Filter Folders Only1 |                                                                                                                        |
| Filter Files Only1         | Filter                  | Filter files from folder contents             | Search Folder Contents1          | Copy File1                    |                                                                                                                        |
| Filter Folders Only1       | Filter                  | Filter folders from folder contents           | Search Folder Contents1          | Create Subfolder1             |                                                                                                                        |
| Copy File1                 | Google Drive            | Copy individual files                          | Filter Files Only1               | Prepare File Data             |                                                                                                                        |
| Prepare File Data          | Set                     | Prepare data for copying permissions           | Copy File1                      | Check File Has Permissions1    |                                                                                                                        |
| Check File Has Permissions1| If                      | Check if file permissions exist                | Prepare File Data               | Prepare File Permissions1 (T), Level Completion (F) |                                                                                                                        |
| Prepare File Permissions1  | Code                    | Parse and prepare file permissions objects     | Check File Has Permissions1     | Apply File Permission1         |                                                                                                                        |
| Apply File Permission1     | Google Drive            | Apply sharing permissions to copied file       | Prepare File Permissions1       | Level Completion              |                                                                                                                        |
| Create Subfolder1          | Google Drive            | Create subfolder in target folder              | Filter Folders Only1             | Prepare Folder Data1           |                                                                                                                        |
| Prepare Folder Data1       | Set                     | Prepare data for folder permissions             | Create Subfolder1               | Check Folder Has Permissions1  |                                                                                                                        |
| Check Folder Has Permissions1| If                    | Check if folder permissions exist               | Prepare Folder Data1            | Prepare Folder Permissions1 (T), Recursive Call to Self (F) |                                                                                                                        |
| Prepare Folder Permissions1| Code                    | Parse and prepare folder permissions objects    | Check Folder Has Permissions1   | Apply Folder Permission1       |                                                                                                                        |
| Apply Folder Permission1   | Google Drive            | Apply sharing permissions to new folder         | Prepare Folder Permissions1     | Recursive Call to Self         |                                                                                                                        |
| Recursive Call to Self     | Execute Workflow        | Recursively process subfolders                   | Apply Folder Permission1, Check Folder Has Permissions1 (F) | Level Completion              |                                                                                                                        |
| Level Completion          | Set                     | Mark completion of one recursion level          | Apply File Permission1, Recursive Call to Self, Check File Has Permissions1 (F) | None                          |                                                                                                                        |
| Sticky Note               | Sticky Note             | Overview and instructions                       | None                          | None                          | Google Drive Folder Duplicator with Permissions: explains purpose, use cases, and setup requirements                  |
| Sticky Note2              | Sticky Note             | Configuration guidance                          | None                          | None                          | START HERE: explains configuration hub and parameters                                                                |
| Sticky Note3              | Sticky Note             | Recursive engine explanation                     | None                          | None                          | CORE RECURSIVE ENGINE: explains scanning, splitting, copying, creating, recursion steps with unlimited depth recursion |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Set Configuration" Node (Set)**  
   - Add a Set node to define workflow parameters:  
     - `sourceFolderId` (string): ID of the folder to duplicate.  
     - `targetFolderId` (string): ID of the parent folder where the new folder will be created.  
     - `newFolderName` (string): Desired name for the duplicated folder.  
   - No inputs; outputs connect to Create Main Folder node.

2. **Create "Create Main Folder" Node (Google Drive)**  
   - Operation: Create folder.  
   - Name: Use expression `{{$json.newFolderName}}`.  
   - Drive ID: Use expression `{{$json.targetFolderId}}`.  
   - Folder ID: Set to `"root"`.  
   - Credentials: Attach Google Drive OAuth2 credentials with full drive access.  
   - Connect input from "Set Configuration".  
   - Output connects to "Start Recursive Processing".

3. **Create "Start Recursive Processing" Node (Execute Workflow)**  
   - Operation: Execute the current workflow (self-call).  
   - Workflow ID: Set to current workflow ID expression: `{{$workflow.id}}`.  
   - Inputs:  
     - `sourceFolderId`: `{{$node["Set Configuration"].json.sourceFolderId}}` on initial call or passed recursively.  
     - `targetFolderId`: `{{$json.id}}` (ID of newly created folder).  
   - Connect input from "Create Main Folder".  
   - Output connects to "Main Completion".

4. **Create "Main Completion" Node (Set)**  
   - Set strings:  
     - `status`: `"Main folder duplication completed successfully"`.  
     - `sourceFolder`: `{{$node["Set Configuration"].json.sourceFolderId}}`.  
     - `newMainFolder`: `{{$node["Create Main Folder"].json.id}}`.  
   - Connect input from "Start Recursive Processing".

5. **Create "Search Folder Contents1" Node (Google Drive)**  
   - Operation: Search files and folders.  
   - Query: `'{{ $json.sourceFolderId }}' in parents and trashed = false`.  
   - Options: Return all results, fields: `id`, `name`, `mimeType`, `permissions`.  
   - Credentials: Google Drive OAuth2.  
   - Connect input from "Start Recursive Processing" (recursive calls).

6. **Create "Filter Files Only1" Node (Filter)**  
   - Condition: Filter out items where `mimeType` ≠ `"application/vnd.google-apps.folder"`.  
   - Connect input from "Search Folder Contents1".  
   - Output connects to "Copy File1".

7. **Create "Filter Folders Only1" Node (Filter)**  
   - Condition: Filter items where `mimeType` = `"application/vnd.google-apps.folder"`.  
   - Connect input from "Search Folder Contents1".  
   - Output connects to "Create Subfolder1".

8. **Create "Copy File1" Node (Google Drive)**  
   - Operation: Copy file.  
   - Name: `{{$json.name}}`.  
   - File ID: `{{$json.id}}`.  
   - Drive ID: Use expression `{{$node["Start Recursive Processing"].json.targetFolderId}}`.  
   - Folder ID: `"root"`.  
   - Credentials: Google Drive OAuth2.  
   - Connect input from "Filter Files Only1".  
   - Output connects to "Prepare File Data".

9. **Create "Prepare File Data" Node (Set)**  
   - Set strings:  
     - `originalFileId`: `{{$node["Filter Files Only1"].json.id}}`.  
     - `originalPermissions`: `{{ JSON.stringify($node["Filter Files Only1"].json.permissions || []) }}`.  
     - `newFileId`: `{{$json.id}}` (output from Copy File).  
   - Connect input from "Copy File1".  
   - Output connects to "Check File Has Permissions1".

10. **Create "Check File Has Permissions1" Node (If)**  
    - Condition: Check if `JSON.parse($json.originalPermissions || '[]').length > 0`.  
    - Connect input from "Prepare File Data".  
    - True branch connects to "Prepare File Permissions1".  
    - False branch connects to "Level Completion".

11. **Create "Prepare File Permissions1" Node (Code)**  
    - JavaScript code parses permissions excluding owners and prepares permission objects for users, groups, domains.  
    - Connect input from "Check File Has Permissions1" (true).  
    - Output connects to "Apply File Permission1".

12. **Create "Apply File Permission1" Node (Google Drive)**  
    - Operation: Share.  
    - File ID: `{{$json.fileId}}`.  
    - Options: `sendNotificationEmail` set to false.  
    - Credentials: Google Drive OAuth2.  
    - Connect input from "Prepare File Permissions1".  
    - Output connects to "Level Completion".

13. **Create "Create Subfolder1" Node (Google Drive)**  
    - Operation: Create folder.  
    - Name: `{{$json.name}}`.  
    - Drive ID: `{{$node["Start Recursive Processing"].json.targetFolderId}}`.  
    - Folder ID: `"root"`.  
    - Credentials: Google Drive OAuth2.  
    - Connect input from "Filter Folders Only1".  
    - Output connects to "Prepare Folder Data1".

14. **Create "Prepare Folder Data1" Node (Set)**  
    - Set strings:  
      - `originalFolderId`: `{{$node["Filter Folders Only1"].json.id}}`.  
      - `originalPermissions`: `{{ JSON.stringify($node["Filter Folders Only1"].json.permissions || []) }}`.  
      - `newFolderId`: `{{$json.id}}`.  
    - Connect input from "Create Subfolder1".  
    - Output connects to "Check Folder Has Permissions1".

15. **Create "Check Folder Has Permissions1" Node (If)**  
    - Condition: Check if `JSON.parse($json.originalPermissions || '[]').length > 0`.  
    - Connect input from "Prepare Folder Data1".  
    - True branch connects to "Prepare Folder Permissions1".  
    - False branch connects to "Recursive Call to Self".

16. **Create "Prepare Folder Permissions1" Node (Code)**  
    - JavaScript code parses and prepares folder permission objects similarly to files.  
    - Connect input from "Check Folder Has Permissions1" (true).  
    - Output connects to "Apply Folder Permission1".

17. **Create "Apply Folder Permission1" Node (Google Drive)**  
    - Operation: Share.  
    - File ID: `{{$json.fileId}}`.  
    - Options: `sendNotificationEmail` false.  
    - Credentials: Google Drive OAuth2.  
    - Connect input from "Prepare Folder Permissions1".  
    - Output connects to "Recursive Call to Self".

18. **Create "Recursive Call to Self" Node (Execute Workflow)**  
    - Calls the current workflow recursively with parameters:  
      - `sourceFolderId`: `{{$json.originalFolderId}}`.  
      - `targetFolderId`: `{{$json.newFolderId}}`.  
    - Connect input from "Apply Folder Permission1" and from "Check Folder Has Permissions1" (false).  
    - Output connects to "Level Completion".

19. **Create "Level Completion" Node (Set)**  
    - Sets the string `status` to `"Level processing completed"`.  
    - Connect inputs from "Apply File Permission1", "Recursive Call to Self", and "Check File Has Permissions1" (false).  
    - No outputs (end of recursion step).

20. **Add Sticky Notes** (Optional for documentation clarity)  
    - Add the provided sticky notes with workflow overview, configuration instructions, and recursive engine explanation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Google Drive Folder Duplicator with Permissions. This workflow duplicates any Google Drive folder recursively, preserving files, folders, and permissions. | Workflow overview and use cases.                                                                                            |
| OAuth2 Google Drive API with scope https://www.googleapis.com/auth/drive is required.                                                                    | Credential setup for Google Drive access.                                                                                   |
| Use cases include backups, template duplication, archiving, and migrations with permission preservation.                                                | Intended applications for the workflow.                                                                                     |
| Recursive logic allows unlimited depth traversal, copying all nested subfolders and files.                                                              | Core recursive engine explanation.                                                                                          |
| Ensure the executing user has read access to source and write/admin access to target folders for permission copying.                                    | Important for preventing permission errors during execution.                                                                |
| Workflow calls itself recursively using Execute Workflow node, passing updated folder IDs.                                                               | Recursive design detail.                                                                                                    |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow designed with n8n, adhering strictly to content policies, containing no illegal or protected elements, and only handling legal, public data.