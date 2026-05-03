Convert Typeform data into Spreadsheet

https://n8nworkflows.xyz/workflows/convert-typeform-data-into-spreadsheet-179


# Convert Typeform data into Spreadsheet

### 1. Workflow Overview

This workflow automates the process of capturing new submissions from a Typeform form and appending them as new rows into an existing spreadsheet file stored on NextCloud. It is designed for scenarios where form data must be continuously logged and updated in a centralized Excel file. The logical flow consists of the following functional blocks:

- **1.1 Input Reception:** Triggering the workflow upon a new Typeform submission.
- **1.2 File Acquisition:** Downloading the existing spreadsheet file from NextCloud.
- **1.3 Data Reading:** Parsing the downloaded spreadsheet into structured flow data.
- **1.4 Data Merging:** Combining existing spreadsheet data with the new form submission.
- **1.5 File Conversion:** Converting the updated data back into spreadsheet file format.
- **1.6 File Upload:** Saving the updated spreadsheet file back to NextCloud, overwriting the original.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new submissions from a specific Typeform form and outputs the submitted data to the workflow.

- **Nodes Involved:**  
  - Typeform Trigger

- **Node Details:**  

  - **Typeform Trigger**  
    - *Type & Role:* Trigger node; initiates the workflow on new Typeform form submissions.  
    - *Configuration:*  
      - `formId`: ID of the Typeform form to monitor (must be set).  
      - Credentials: Configured with Typeform API credentials using OAuth or API key.  
    - *Key Variables:* Outputs form submission data with fields named exactly as the form questions (matching spreadsheet columns: Name, Email, Severity, Problem).  
    - *Input/Output:* No input. Outputs JSON object representing one form submission.  
    - *Edge Cases:*  
      - Missing or invalid formId leads to trigger failure.  
      - Authentication errors due to invalid credentials.  
      - Delay or missed triggers if webhook registration fails.  
    - *Version:* n8n Typeform Trigger node, version 1.

---

#### 2.2 File Acquisition

- **Overview:**  
  Downloads the current spreadsheet file from NextCloud storage to provide existing data for appending.

- **Nodes Involved:**  
  - NextCloud (Download)

- **Node Details:**  

  - **NextCloud**  
    - *Type & Role:* File operation node; downloads a file from NextCloud.  
    - *Configuration:*  
      - `operation`: Set to `download`.  
      - `path`: Specifies the file path `"examples/Problems.xls"`.  
      - Credentials: NextCloud API credentials configured with OAuth or basic authentication.  
    - *Input/Output:* No input connections; outputs binary data representing the downloaded file.  
    - *Edge Cases:*  
      - File not found or incorrect path leads to download failure.  
      - Authentication or permission issues.  
      - Network timeouts.  
    - *Version:* n8n NextCloud node, version 1.

---

#### 2.3 Data Reading

- **Overview:**  
  Converts the downloaded spreadsheet binary into structured JSON data usable in the workflow.

- **Nodes Involved:**  
  - Spreadsheet File (Read)

- **Node Details:**  

  - **Spreadsheet File**  
    - *Type & Role:* Data conversion node; reads spreadsheet binary and outputs JSON.  
    - *Configuration:* Defaults for reading (no specific parameters set).  
    - *Input/Output:* Receives binary file data from NextCloud node; outputs JSON array of rows with columns as keys (`Name`, `Email`, `Severity`, `Problem`).  
    - *Edge Cases:*  
      - Corrupted or unsupported file format.  
      - Empty or malformed spreadsheet content.  
    - *Version:* n8n Spreadsheet File node, version 1.

---

#### 2.4 Data Merging

- **Overview:**  
  Appends the new Typeform submission data to the existing spreadsheet data.

- **Nodes Involved:**  
  - Merge

- **Node Details:**  

  - **Merge**  
    - *Type & Role:* Data merging node; combines two data streams.  
    - *Configuration:* Default merge mode (assumed to be `append` or `merge by position`).  
    - *Input/Output:*  
      - Input 1: JSON data from Spreadsheet File (existing rows).  
      - Input 2: JSON data from Typeform Trigger (new submission).  
      - Output: Combined JSON data array including new row.  
    - *Edge Cases:*  
      - Mismatched column names or data structures could cause inconsistent rows.  
      - Large data volumes may cause performance issues.  
    - *Version:* n8n Merge node, version 1.

---

#### 2.5 File Conversion

- **Overview:**  
  Converts the merged JSON dataset back into an Excel spreadsheet file in binary form.

- **Nodes Involved:**  
  - Spreadsheet File1 (Write)

- **Node Details:**  

  - **Spreadsheet File1**  
    - *Type & Role:* Data conversion node; converts JSON data into a spreadsheet file.  
    - *Configuration:*  
      - `operation`: Set to `toFile` to generate a file.  
    - *Input/Output:* Receives merged JSON data; outputs binary (Excel file) data.  
    - *Edge Cases:*  
      - Data containing unsupported formats or types may cause conversion errors.  
      - Column order and naming must be consistent with expectations.  
    - *Version:* n8n Spreadsheet File node, version 1.

---

#### 2.6 File Upload

- **Overview:**  
  Uploads the updated spreadsheet file back to the same location on NextCloud, overwriting the original.

- **Nodes Involved:**  
  - NextCloud1 (Upload)

- **Node Details:**  

  - **NextCloud1**  
    - *Type & Role:* File operation node; uploads binary data to NextCloud.  
    - *Configuration:*  
      - `path`: Dynamically set to the same path as the originally downloaded file (`={{$node["NextCloud"].parameter["path"]}}`).  
      - `binaryDataUpload`: Enabled to upload file binary.  
      - Credentials: Same NextCloud API credentials as the download node.  
    - *Input/Output:* Receives binary file from Spreadsheet File1; no further output.  
    - *Edge Cases:*  
      - Permission issues preventing overwrite.  
      - Network failures during upload.  
      - File locking issues if the file is in use on NextCloud.  
    - *Version:* n8n NextCloud node, version 1.

---

### 3. Summary Table

| Node Name          | Node Type                 | Functional Role               | Input Node(s)  | Output Node(s)   | Sticky Note                                                        |
|--------------------|---------------------------|------------------------------|----------------|------------------|-------------------------------------------------------------------|
| Typeform Trigger   | Typeform Trigger          | Trigger on new form submission | -              | Merge            |                                                                   |
| NextCloud          | NextCloud                 | Download spreadsheet file from NextCloud | -              | Spreadsheet File |                                                                   |
| Spreadsheet File   | Spreadsheet File          | Read spreadsheet binary into JSON | NextCloud      | Merge            |                                                                   |
| Merge              | Merge                     | Combine existing data with new form submission | Spreadsheet File, Typeform Trigger | Spreadsheet File1 |                                                                   |
| Spreadsheet File1  | Spreadsheet File          | Convert JSON data to spreadsheet file | Merge          | NextCloud1       |                                                                   |
| NextCloud1         | NextCloud                 | Upload updated spreadsheet file to NextCloud | Spreadsheet File1 | -                |                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger node:**  
   - Type: Typeform Trigger  
   - Configure `formId` with your Typeform form ID.  
   - Set up Typeform API credentials (OAuth or API key).  
   - No input connections; output is new form submission data.

2. **Create NextCloud node for download:**  
   - Type: NextCloud  
   - Operation: `download`  
   - Path: `examples/Problems.xls` (adjust path to your file location).  
   - Configure NextCloud API credentials (OAuth or basic auth).  
   - No input connections; outputs binary file data.

3. **Create Spreadsheet File node to read:**  
   - Type: Spreadsheet File  
   - Default settings for reading spreadsheet.  
   - Connect input from NextCloud download node.  
   - Outputs JSON array of rows.

4. **Create Merge node:**  
   - Type: Merge  
   - Use default merge mode (append).  
   - Connect input 1 to Spreadsheet File node (existing data).  
   - Connect input 2 to Typeform Trigger node (new submission).  
   - Output is combined JSON data.

5. **Create Spreadsheet File node to write:**  
   - Type: Spreadsheet File  
   - Operation: `toFile` (convert JSON to spreadsheet file).  
   - Connect input from Merge node.  
   - Outputs binary spreadsheet file.

6. **Create NextCloud node for upload:**  
   - Type: NextCloud  
   - Operation: upload (default when binaryDataUpload is true).  
   - Path: Set expression to `={{$node["NextCloud"].parameter["path"]}}` to reuse original file path.  
   - Enable `binaryDataUpload`.  
   - Configure NextCloud API credentials (same as download).  
   - Connect input from Spreadsheet File node (toFile).

7. **Connect nodes in sequence:**  
   - Typeform Trigger → Merge (input 2)  
   - NextCloud (download) → Spreadsheet File (read) → Merge (input 1)  
   - Merge → Spreadsheet File1 (toFile) → NextCloud1 (upload)

8. **Verify column names:**  
   Ensure Typeform question names exactly match spreadsheet columns: `Name`, `Email`, `Severity`, `Problem`.

9. **Test workflow:**  
   Submit a new form entry and confirm the spreadsheet updates with appended data.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                 |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| The spreadsheet file "Problems.xls" is stored under the "examples" folder on NextCloud.            | Workflow file path configuration.                              |
| Typeform form questions must be named exactly as the spreadsheet columns to ensure proper mapping. | Data consistency requirement.                                  |
| For NextCloud credentials, OAuth or basic authentication can be used depending on your setup.      | Credential setup for NextCloud nodes.                          |
| The merge node appends new form data to existing spreadsheet data; verify column alignment.         | Important for data integrity.                                  |
| This workflow can be adapted to other form/spreadsheet types with similar data structure.          | Extensibility note.                                            |

---

This documentation provides a complete and clear reference to understand, reproduce, and modify the "Convert Typeform data into Spreadsheet" workflow with n8n.