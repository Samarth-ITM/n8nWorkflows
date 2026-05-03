Copy Folder Structure Without Files in Google Drive

https://n8nworkflows.xyz/workflows/copy-folder-structure-without-files-in-google-drive-4240


# Copy Folder Structure Without Files in Google Drive

---

### 1. Workflow Overview

This workflow automates the process of copying the **folder structure only** (without any files) from a source Google Drive folder to a destination Google Drive folder. It is designed to replicate directory hierarchies, which is useful for backup purposes, migrations, or re-organizing Google Drive folders while preserving folder nesting.

**Key Use Cases:**  
- Clone folder structures across Google Drive accounts or locations.  
- Prepare empty folder templates matching an existing structure.  
- Automate folder replication for workflows depending on folder layouts without copying content.

**Logical Blocks:**

- **1.1 Input Initialization**: Manual trigger to start the workflow and initial setting of parameters for source and destination folders.  
- **1.2 Fetch Destination Folder Context**: Retrieve existing folders in the destination to avoid duplicates.  
- **1.3 Prepare Folder List from Source**: Get all folders from the source path to be copied.  
- **1.4 Processing Loop & Folder Creation**: Iterate over each source folder, check if it exists at the destination, and create it if missing.  
- **1.5 Folder Context Merging and Naming**: Maintain context between source and destination folder structures for accurate nested replication.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Initialization

**Overview:**  
Starts the workflow manually and sets up initial variables including source and destination folder IDs or identifiers for the folder structures.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- EDIT_THIS_NODE

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution.  
  - Config: Default (no parameters).  
  - Inputs: None  
  - Outputs: Triggers EDIT_THIS_NODE.  
  - Edge cases: None typical; ensures user-triggered only.

- **EDIT_THIS_NODE**  
  - Type: Set  
  - Role: Define key parameters such as source and destination folder IDs or names.  
  - Config: User must enter the source folder ID and destination folder ID here to specify which folders to work on.  
  - Expressions: Likely sets static values or environment variables for source/destination.  
  - Inputs: Trigger from manual node  
  - Outputs: Connects to Get_Folders_DESTINATION and Merge_Folder_Context  
  - Edge cases: Misconfiguration or invalid folder IDs will cause downstream errors (e.g., folder not found).  

---

#### 1.2 Fetch Destination Folder Context

**Overview:**  
Retrieves all folders currently existing in the destination Google Drive folder to avoid recreating folders that already exist.

**Nodes Involved:**  
- Get_Folders_DESTINATION  
- Merge_Folder_Context

**Node Details:**

- **Get_Folders_DESTINATION**  
  - Type: Google Drive (Folder Retrieval)  
  - Role: List all folders under the destination folder.  
  - Config: Uses folder ID from EDIT_THIS_NODE or previous steps to list contents recursively.  
  - Inputs: From EDIT_THIS_NODE  
  - Outputs: To Merge_Folder_Context  
  - Edge cases: API rate limits, permission errors if destination folder is inaccessible or credential issues.

- **Merge_Folder_Context**  
  - Type: Merge  
  - Role: Combines destination folder data with source folder context to allow comparison and mapping.  
  - Config: Defaults (likely 'Merge by index' or 'Merge by fields')  
  - Inputs: From Get_Folders_DESTINATION and EDIT_THIS_NODE (or initial data)  
  - Outputs: To Set Destination Names  
  - Edge cases: Mismatch in data length or format could cause merge issues.

---

#### 1.3 Prepare Folder List from Source

**Overview:**  
Fetches all folders present in the source Google Drive folder to form the list of folders to be copied (structure only).

**Nodes Involved:**  
- Set Destination Names  
- Get_Folders_SOURCE  
- Loop Over Items

**Node Details:**

- **Set Destination Names**  
  - Type: Set  
  - Role: Prepares or modifies folder names or properties based on the merged context to align source folders with destination naming conventions or paths.  
  - Config: May set custom expressions for destination folder names or paths.  
  - Inputs: From Merge_Folder_Context  
  - Outputs: To Get_Folders_SOURCE  
  - Edge cases: Incorrect expressions or missing data could lead to wrong folder naming.

- **Get_Folders_SOURCE**  
  - Type: Google Drive (Folder Retrieval)  
  - Role: List all folders under the source folder, recursively if needed.  
  - Config: Uses source folder ID from Set Destination Names or EDIT_THIS_NODE.  
  - Inputs: From Set Destination Names  
  - Outputs: To Loop Over Items  
  - Edge cases: Permissions issues or API limits if source folder is large or inaccessible.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each source folder individually in batches to avoid API request limits or timeouts.  
  - Config: Default batch size or user-defined.  
  - Inputs: From Get_Folders_SOURCE  
  - Outputs: To Check if folder exists  
  - Edge cases: Large number of folders could slow processing. Batch size misconfiguration may cause errors.

---

#### 1.4 Processing Loop & Folder Creation

**Overview:**  
For each folder from the source, checks if it already exists in the destination; if not, creates the folder in the destination Google Drive.

**Nodes Involved:**  
- Check if folder exists  
- If exists  
- Create Folder  
- Loop Over Items (continuation)

**Node Details:**

- **Check if folder exists**  
  - Type: Code (JavaScript)  
  - Role: Custom logic to determine if the current source folder already exists in the destination folder context.  
  - Config: Compares folder names, IDs or paths to find duplicates.  
  - Inputs: From Loop Over Items (current folder data)  
  - Outputs: To If exists  
  - Edge cases: Logic errors can cause false positives/negatives; errors in code cause workflow failure.

- **If exists**  
  - Type: If  
  - Role: Conditional branching based on existence check.  
  - Config: Checks boolean or condition set by previous node.  
  - Inputs: From Check if folder exists  
  - Outputs:  
    - True branch (folder exists): ends or skips creation.  
    - False branch: proceeds to Create Folder.  
  - Edge cases: Condition misconfiguration may lead to unintended folder creation or skipping.

- **Create Folder**  
  - Type: Google Drive (Folder Creation)  
  - Role: Creates a new folder in the destination Google Drive under the correct parent directory.  
  - Config: Uses folder name and parent folder ID mapped from source folder data.  
  - Inputs: From If exists (false branch)  
  - Outputs: Back to Loop Over Items to continue processing next folder  
  - Edge cases: API quota limits, folder name conflicts, permission errors.

- **Loop Over Items**  
  - Continues to process all source folders in batches.

---

#### 1.5 Folder Context Merging and Naming

**Overview:**  
Maintains mapping and context between source and destination folder structures to ensure correct hierarchical replication.

**Nodes Involved:**  
- Merge_Folder_Context (also part of 1.2)  
- Set Destination Names (also part of 1.3)

**Node Details:**  
Already described above; these nodes help align source folder metadata with destination folder metadata to support correct folder creation paths.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                          | Input Node(s)                  | Output Node(s)                       | Sticky Note |
|----------------------------|---------------------|----------------------------------------|-------------------------------|------------------------------------|-------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Entry point to start the workflow      | None                          | EDIT_THIS_NODE                     |             |
| EDIT_THIS_NODE             | Set                 | Set source and destination folder IDs  | When clicking ‘Test workflow’ | Get_Folders_DESTINATION, Merge_Folder_Context |             |
| Get_Folders_DESTINATION    | Google Drive        | Fetch existing folders at destination   | EDIT_THIS_NODE                | Merge_Folder_Context               |             |
| Merge_Folder_Context       | Merge               | Combine source and destination folder data | EDIT_THIS_NODE, Get_Folders_DESTINATION | Set Destination Names              |             |
| Set Destination Names      | Set                 | Prepare destination folder naming       | Merge_Folder_Context          | Get_Folders_SOURCE                 |             |
| Get_Folders_SOURCE         | Google Drive        | Fetch folders from source folder         | Set Destination Names         | Loop Over Items                   |             |
| Loop Over Items            | SplitInBatches      | Iterate over source folders in batches   | Get_Folders_SOURCE            | Check if folder exists             |             |
| Check if folder exists     | Code                | Check if folder exists in destination    | Loop Over Items               | If exists                        |             |
| If exists                  | If                  | Conditional: folder exists or not         | Check if folder exists        | Create Folder (if false branch)    |             |
| Create Folder              | Google Drive        | Create folder in destination if missing | If exists                    | Loop Over Items                   |             |
| Sticky Note                | Sticky Note         | (Empty)                                 |                               |                                    |             |
| Sticky Note1               | Sticky Note         | (Empty)                                 |                               |                                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: *When clicking ‘Test workflow’*  
   - Purpose: Start workflow manually.

2. **Create Set Node to Define Parameters**  
   - Name: *EDIT_THIS_NODE*  
   - Configure parameters:  
     - Set source folder ID (Google Drive folder ID)  
     - Set destination folder ID  
   - Connect: Manual Trigger → EDIT_THIS_NODE

3. **Create Google Drive Node to Get Destination Folders**  
   - Name: *Get_Folders_DESTINATION*  
   - Operation: List folders under destination folder ID from EDIT_THIS_NODE  
   - Connect: EDIT_THIS_NODE → Get_Folders_DESTINATION

4. **Create Merge Node**  
   - Name: *Merge_Folder_Context*  
   - Purpose: Merge destination folder data with initial context  
   - Connect: EDIT_THIS_NODE (input 2), Get_Folders_DESTINATION (input 1) → Merge_Folder_Context

5. **Create Set Node to Prepare Destination Names**  
   - Name: *Set Destination Names*  
   - Configure: Adjust folder naming or paths as needed based on merged data  
   - Connect: Merge_Folder_Context → Set Destination Names

6. **Create Google Drive Node to Get Source Folders**  
   - Name: *Get_Folders_SOURCE*  
   - Operation: List folders under source folder ID from Set Destination Names  
   - Connect: Set Destination Names → Get_Folders_SOURCE

7. **Create SplitInBatches Node**  
   - Name: *Loop Over Items*  
   - Purpose: Process source folders one by one or in batches  
   - Connect: Get_Folders_SOURCE → Loop Over Items  
   - Configure batch size as required (default is fine)

8. **Create Code Node to Check Folder Existence**  
   - Name: *Check if folder exists*  
   - Implement JavaScript logic to verify if current folder exists in destination data  
   - Input: Current batch item from Loop Over Items  
   - Output: Boolean or condition for If node  
   - Connect: Loop Over Items → Check if folder exists

9. **Create If Node**  
   - Name: *If exists*  
   - Condition: Check boolean from Code node  
   - True branch: Do nothing or continue loop  
   - False branch: Proceed to create folder  
   - Connect: Check if folder exists → If exists

10. **Create Google Drive Node to Create Folder**  
    - Name: *Create Folder*  
    - Operation: Create folder with name and parent ID mapped from source to destination  
    - Connect: If exists (false branch) → Create Folder

11. **Connect Create Folder back to Loop Over Items**  
    - To continue processing next folders after creation.

12. **Add Sticky Notes as Needed**  
    - Optional for comments or instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                |
|-----------------------------------------------------------------------------------------------|-------------------------------|
| Workflow copies folder structures **without files** to replicate Google Drive hierarchies.    | Workflow Description           |
| Ensure Google Drive credentials have appropriate scopes (read and write access to Drive).     | Credential Setup               |
| For large folder trees, consider adjusting batch sizes to avoid API rate limits or timeouts.  | Performance Considerations     |
| Code node uses JavaScript to compare folder existence; test logic carefully to avoid errors.  | Node-specific Implementation  |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---