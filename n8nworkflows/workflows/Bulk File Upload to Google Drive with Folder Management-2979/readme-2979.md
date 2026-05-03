Bulk File Upload to Google Drive with Folder Management

https://n8nworkflows.xyz/workflows/bulk-file-upload-to-google-drive-with-folder-management-2979


# Bulk File Upload to Google Drive with Folder Management

### 1. Workflow Overview

This workflow automates bulk file uploads to Google Drive with dynamic folder management. It is designed for users submitting multiple files along with a target folder name via a form. The workflow checks if the specified folder exists in Google Drive under a predefined parent folder; if not, it creates the folder. Then, it processes and uploads all submitted files into the appropriate folder, preserving original file names and structure.

**Target Use Cases:**
- Bulk file organization and upload automation
- Automated Google Drive folder creation and management
- Maintaining consistent file structures during batch uploads

**Logical Blocks:**

- **1.1 Input Reception:** Receives user input via a form with multiple file uploads and a folder name.
- **1.2 Folder Existence Check:** Searches Google Drive for the folder under a specified parent folder.
- **1.3 Folder Decision Branch:** Determines whether to use an existing folder or create a new one.
- **1.4 File Preparation and Upload (Existing Folder):** Prepares files and uploads them to the existing folder.
- **1.5 Folder Creation and File Upload (New Folder):** Creates the folder, prepares files, and uploads them to the new folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user input from a web form, including multiple files and a target folder name.

- **Nodes Involved:**  
  - On form submission  
  - Get Folder Name

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point; triggers workflow on form submission  
    - Configuration:  
      - Form titled "Batch File Upload to Google Drive"  
      - Fields:  
        - Multiple file upload (field label: "file", required)  
        - Text input for folder name (field label: "folderName", required)  
      - Webhook ID configured for external form integration  
    - Inputs: None (trigger node)  
    - Outputs: Passes form data (binary files and folderName)  
    - Edge Cases: Missing required fields; large file uploads may cause timeout or memory issues

  - **Get Folder Name**  
    - Type: Set  
    - Role: Extracts and sets the folderName from the form submission JSON for downstream use  
    - Configuration: Assigns `folderName` = `{{$json.folderName}}`  
    - Inputs: From "On form submission"  
    - Outputs: JSON with folderName property  
    - Edge Cases: folderName missing or empty string could cause search query issues

---

#### 1.2 Folder Existence Check

- **Overview:**  
  Searches Google Drive for a folder matching the submitted folder name under a specific parent folder.

- **Nodes Involved:**  
  - Search specific folder

- **Node Details:**

  - **Search specific folder**  
    - Type: Google Drive (File/Folder Search)  
    - Role: Queries Google Drive to find if the folder exists  
    - Configuration:  
      - Resource: fileFolder  
      - Search method: Query  
      - Query string:  
        ```
        mimeType='application/vnd.google-apps.folder' and name = '{{ $json.folderName }}' and '<folderId>' in parents
        ```  
        (Replace `<folderId>` with your Google Drive parent folder ID)  
      - Always output data enabled to handle empty results gracefully  
    - Credentials: Google Drive OAuth2 (configured)  
    - Inputs: From "Get Folder Name"  
    - Outputs: Search results (empty if folder not found)  
    - Edge Cases:  
      - Invalid or missing parent folder ID causes no results  
      - API rate limits or auth errors  
      - Folder name collisions (multiple folders with same name) — only first result used downstream

---

#### 1.3 Folder Decision Branch

- **Overview:**  
  Determines workflow path based on whether the folder was found.

- **Nodes Involved:**  
  - Folder found ?

- **Node Details:**

  - **Folder found ?**  
    - Type: If  
    - Role: Checks if the search result from "Search specific folder" is non-empty  
    - Configuration:  
      - Condition: JSON object from previous node is not empty (`notEmpty` operator on `$json`)  
    - Inputs: From "Search specific folder"  
    - Outputs:  
      - True branch: Folder exists  
      - False branch: Folder does not exist  
    - Edge Cases:  
      - Empty or malformed search results  
      - Expression evaluation errors if input JSON missing

---

#### 1.4 File Preparation and Upload (Existing Folder)

- **Overview:**  
  Prepares each uploaded file individually and uploads them to the existing folder found in Google Drive.

- **Nodes Involved:**  
  - Prepare Files for Upload  
  - Upload Files

- **Node Details:**

  - **Prepare Files for Upload**  
    - Type: Code (JavaScript)  
    - Role: Splits multiple binary files from the form submission into individual items for upload  
    - Configuration:  
      - Iterates over all form submission items and their binary properties  
      - For each binary file, creates a new item with:  
        - JSON: fileName from binary metadata  
        - Binary: file data  
    - Inputs: From "Folder found ?" (true branch)  
    - Outputs: Array of individual file items  
    - Edge Cases:  
      - No binary files present  
      - Binary data corruption or missing fileName

  - **Upload Files**  
    - Type: Google Drive (File Upload)  
    - Role: Uploads each prepared file to the existing folder  
    - Configuration:  
      - Name: `{{$json.fileName}}` (preserves original file name)  
      - Drive ID: "My Drive"  
      - Folder ID: dynamically set to the existing folder ID from "Search specific folder" node (`{{$node["Search specific folder"].item.json.id}}`)  
      - Input data field: "data" (binary data field)  
    - Credentials: Google Drive OAuth2  
    - Inputs: From "Prepare Files for Upload"  
    - Outputs: Upload results  
    - Edge Cases:  
      - Upload failures due to file size limits or API errors  
      - Folder ID missing or invalid  
      - Duplicate file names overwrite or conflict (Google Drive default behavior)

---

#### 1.5 Folder Creation and File Upload (New Folder)

- **Overview:**  
  Creates a new folder in Google Drive, prepares files, and uploads them to this new folder.

- **Nodes Involved:**  
  - Create Folder  
  - Prepare Files for New Folder  
  - Upload to New Folder

- **Node Details:**

  - **Create Folder**  
    - Type: Google Drive (Folder Create)  
    - Role: Creates a new folder with the submitted folder name under a specified parent folder  
    - Configuration:  
      - Name: `{{$node["On form submission"].item.json.folderName}}`  
      - Drive ID: "My Drive"  
      - Parent Folder ID: hardcoded to `"17sGS9HdmAtgpd5rC1sVuiIUGyw2hq9IY"` (replace with your own parent folder ID)  
    - Credentials: Google Drive OAuth2  
    - Inputs: From "Folder found ?" (false branch)  
    - Outputs: New folder metadata including folder ID  
    - Edge Cases:  
      - Folder creation failure due to permissions or API errors  
      - Duplicate folder names allowed by Drive, may cause confusion downstream

  - **Prepare Files for New Folder**  
    - Type: Code (JavaScript)  
    - Role: Same logic as "Prepare Files for Upload" — splits binary files for upload  
    - Configuration: Identical to "Prepare Files for Upload" node  
    - Inputs: From "Create Folder"  
    - Outputs: Individual file items  
    - Edge Cases: Same as above

  - **Upload to New Folder**  
    - Type: Google Drive (File Upload)  
    - Role: Uploads each prepared file to the newly created folder  
    - Configuration:  
      - Name: `{{$json.fileName}}`  
      - Drive ID: "My Drive"  
      - Folder ID: dynamically set to new folder ID from "Create Folder" (`{{$node["Create Folder"].item.json.id}}`)  
      - Input data field: "data"  
    - Credentials: Google Drive OAuth2  
    - Inputs: From "Prepare Files for New Folder"  
    - Outputs: Upload results  
    - Edge Cases: Same as "Upload Files" node

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                          | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                              |
|-------------------------|-----------------------|----------------------------------------|------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger          | Entry point; receives files and folder name | None                   | Get Folder Name                | # 🗂️ Bulk File Upload to Google Drive with Folder Management (overview note)                           |
| Get Folder Name         | Set                   | Extracts folderName from form data     | On form submission      | Search specific folder         | # 🗂️ Bulk File Upload to Google Drive with Folder Management (overview note)                           |
| Search specific folder  | Google Drive (Search) | Searches for folder by name under parent folder | Get Folder Name         | Folder found ?                 | ## 🔍 Search Query Pattern (replace <folderId> in query)                                               |
| Folder found ?          | If                    | Branches workflow based on folder existence | Search specific folder  | Prepare Files for Upload (true), Create Folder (false) | ## 🔄 Decision Point: Folder Check (explains branching logic)                                          |
| Prepare Files for Upload| Code (JavaScript)     | Splits multiple files for upload       | Folder found ? (true)   | Upload Files                  | ## ⚙️ File Processing Notes (file splitting and naming)                                                |
| Upload Files            | Google Drive (Upload) | Uploads files to existing folder       | Prepare Files for Upload| None                         | ## ⚙️ File Processing Notes (file splitting and naming)                                                |
| Create Folder           | Google Drive (Folder) | Creates new folder in Drive             | Folder found ? (false)  | Prepare Files for New Folder   | ## 🔄 Decision Point: Folder Check (explains branching logic)                                          |
| Prepare Files for New Folder | Code (JavaScript) | Splits files for upload to new folder  | Create Folder           | Upload to New Folder           | ## ⚙️ File Processing Notes (file splitting and naming)                                                |
| Upload to New Folder    | Google Drive (Upload) | Uploads files to newly created folder  | Prepare Files for New Folder | None                      | ## ⚙️ File Processing Notes (file splitting and naming)                                                |
| Sticky Note             | Sticky Note           | Overview and workflow description      | None                   | None                         | # 🗂️ Bulk File Upload to Google Drive with Folder Management (overview note)                           |
| Sticky Note1            | Sticky Note           | Explains decision branching and paths  | None                   | None                         | ## 🔄 Decision Point: Folder Check (explains branching logic)                                          |
| Sticky Note2            | Sticky Note           | File processing notes and external link | None                   | None                         | ## ⚙️ File Processing Notes (file splitting and naming), see https://n8n.io/workflows/1621-split-out-binary-data/ |
| Sticky Note4            | Sticky Note           | Search query pattern and instructions  | None                   | None                         | ## 🔍 Search Query Pattern (replace <folderId> in query)                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure form title: "Batch File Upload to Google Drive"  
   - Add form fields:  
     - File upload field (label: "file", required, allow multiple files)  
     - Text field (label: "folderName", required)  
   - Save and note webhook URL for form integration

2. **Create "Get Folder Name" node**  
   - Type: Set  
   - Add assignment:  
     - Variable name: `folderName`  
     - Value: Expression `{{$json.folderName}}`  
   - Connect "On form submission" → "Get Folder Name"

3. **Create "Search specific folder" node**  
   - Type: Google Drive (File/Folder Search)  
   - Set resource: `fileFolder`  
   - Search method: `query`  
   - Query string:  
     ```
     mimeType='application/vnd.google-apps.folder' and name = '{{ $json.folderName }}' and '<folderId>' in parents
     ```  
     Replace `<folderId>` with your Google Drive parent folder ID  
   - Enable "Always Output Data"  
   - Set Google Drive OAuth2 credentials  
   - Connect "Get Folder Name" → "Search specific folder"

4. **Create "Folder found ?" node**  
   - Type: If  
   - Condition: Check if `$json` is not empty (operator: notEmpty)  
   - Connect "Search specific folder" → "Folder found ?"

5. **Create "Prepare Files for Upload" node**  
   - Type: Code (JavaScript)  
   - Paste code:  
     ```javascript
     let results = [];
     const items = $("On form submission").all();

     for (const item of items) {
       for (const key of Object.keys(item.binary)) {
         results.push({
           json: {
             fileName: item.binary[key].fileName,
           },
           binary: {
             data: item.binary[key],
           },
         });
       }
     }

     return results;
     ```  
   - Connect "Folder found ?" (true output) → "Prepare Files for Upload"

6. **Create "Upload Files" node**  
   - Type: Google Drive (File Upload)  
   - Name: Expression `{{$json.fileName}}`  
   - Drive ID: "My Drive"  
   - Folder ID: Expression `{{$node["Search specific folder"].item.json.id}}`  
   - Input Data Field Name: `data`  
   - Set Google Drive OAuth2 credentials  
   - Connect "Prepare Files for Upload" → "Upload Files"

7. **Create "Create Folder" node**  
   - Type: Google Drive (Folder Create)  
   - Name: Expression `{{$node["On form submission"].item.json.folderName}}`  
   - Drive ID: "My Drive"  
   - Folder ID (parent): Set to your Google Drive parent folder ID (e.g., `"17sGS9HdmAtgpd5rC1sVuiIUGyw2hq9IY"`)  
   - Set Google Drive OAuth2 credentials  
   - Connect "Folder found ?" (false output) → "Create Folder"

8. **Create "Prepare Files for New Folder" node**  
   - Type: Code (JavaScript)  
   - Use the same code as "Prepare Files for Upload" node  
   - Connect "Create Folder" → "Prepare Files for New Folder"

9. **Create "Upload to New Folder" node**  
   - Type: Google Drive (File Upload)  
   - Name: Expression `{{$json.fileName}}`  
   - Drive ID: "My Drive"  
   - Folder ID: Expression `{{$node["Create Folder"].item.json.id}}`  
   - Input Data Field Name: `data`  
   - Set Google Drive OAuth2 credentials  
   - Connect "Prepare Files for New Folder" → "Upload to New Folder"

10. **Optional: Add Sticky Notes**  
    - Add sticky notes with workflow overview, decision branching, file processing notes, and search query pattern for documentation and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates bulk file uploads to Google Drive with folder existence check and creation.                 | Overview sticky note in workflow                                                                    |
| Decision branching splits workflow into existing folder path and new folder creation path.                     | Sticky note explaining decision points and paths                                                    |
| File processing involves splitting binary files individually to preserve file names and structure.            | Sticky note with file processing notes and link to example workflow: https://n8n.io/workflows/1621-split-out-binary-data/ |
| Google Drive search query requires replacing `<folderId>` with your actual parent folder ID in Drive.          | Sticky note with search query pattern and instructions                                              |
| Google Drive OAuth2 credentials must be configured in n8n before running this workflow.                         | Credential setup reminder                                                                           |
| Parent folder ID used in search and folder creation must be replaced with your own Drive folder ID.             | Configuration step                                                                                  |

---

This documentation enables advanced users and automation agents to fully understand, reproduce, and modify the workflow, anticipate potential errors, and maintain the workflow effectively.