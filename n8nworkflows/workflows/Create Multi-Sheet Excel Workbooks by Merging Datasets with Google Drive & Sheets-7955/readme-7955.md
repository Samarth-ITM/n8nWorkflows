Create Multi-Sheet Excel Workbooks by Merging Datasets with Google Drive & Sheets

https://n8nworkflows.xyz/workflows/create-multi-sheet-excel-workbooks-by-merging-datasets-with-google-drive---sheets-7955


# Create Multi-Sheet Excel Workbooks by Merging Datasets with Google Drive & Sheets

### 1. Workflow Overview

This workflow automates the creation of multi-sheet Excel workbooks by merging multiple datasets into a single `.xlsx` file with multiple tabs, integrating tightly with Google Drive and Google Sheets. It is designed for users who want to automate reporting and data consolidation tasks without manual copy-pasting, especially when working with Google Workspace.

The workflow logic can be grouped into these blocks:

- **1.1 Input Trigger and Data Generation**: Manual trigger initiates the workflow; two separate Code nodes generate sample datasets.
- **1.2 Data Conversion to Excel Worksheets**: Each dataset is converted into an individual Excel worksheet (one per dataset).
- **1.3 Workbook Assembly and Google Sheets Update**: The individual worksheets are merged into a single multi-sheet Excel file; data from one set is appended to a Google Sheet.
- **1.4 Google Drive Export**: The final Excel file is downloaded/exported from Google Drive for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Data Generation

- **Overview**: This block initiates the workflow manually and generates two separate datasets via Code nodes simulating data retrieval or generation.
- **Nodes Involved**:
  - When clicking ‘Execute workflow’
  - Data Set 1
  - Data Set 2

- **Node Details**:

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Starts the workflow execution on user command.
    - Configuration: Default manual trigger without parameters.
    - Inputs: None (trigger node)
    - Outputs: Triggers downstream nodes on execution.
    - Potential Failures: None under normal use; user must trigger the workflow.
  
  - **Data Set 1**
    - Type: Code (JavaScript)
    - Role: Generates first dataset with columns "Name" and "Age".
    - Configuration: Hardcoded array of JSON objects with sample records.
    - Key Code: Returns array with entries for Alice, Bob, Charlie.
    - Inputs: Trigger from manual node.
    - Outputs: JSON data array.
    - Edge Cases: Static data; no dynamic error handling needed.
  
  - **Data Set 2**
    - Type: Code (JavaScript)
    - Role: Generates second dataset with columns "City" and "Country".
    - Configuration: Hardcoded array of JSON objects with sample records.
    - Key Code: Returns array with entries for New York, London, Tokyo.
    - Inputs: Trigger from manual node.
    - Outputs: JSON data array.
    - Edge Cases: Static data; no dynamic error handling needed.

#### 2.2 Data Conversion to Excel Worksheets

- **Overview**: Converts each dataset into an individual Excel worksheet binary file format, preparing for merge.
- **Nodes Involved**:
  - Convert to File
  - Convert to File1

- **Node Details**:

  - **Convert to File**
    - Type: Convert To File
    - Role: Converts Data Set 1 JSON to Excel worksheet named "Sheet1".
    - Configuration: Operation set to "xlsx"; sheetName set to "Sheet1".
    - Inputs: Data Set 1 JSON.
    - Outputs: Binary Excel file representing first worksheet.
    - Edge Cases: Conversion failure if input JSON is malformed or empty.
  
  - **Convert to File1**
    - Type: Convert To File
    - Role: Converts Data Set 2 JSON to Excel worksheet named "Sheet2".
    - Configuration: Operation "xlsx"; sheetName "Sheet2"; binaryPropertyName set to "data2" to differentiate output.
    - Inputs: Data Set 2 JSON.
    - Outputs: Binary Excel file representing second worksheet.
    - Edge Cases: Same as above; ensure input data integrity.

#### 2.3 Workbook Assembly and Google Sheets Update

- **Overview**: Merges the two Excel worksheet binaries into a single multi-sheet Excel file. Additionally, it appends the second dataset to a Google Sheet for collaborative sharing or further processing.
- **Nodes Involved**:
  - Merge1
  - Save to google sheets

- **Node Details**:

  - **Merge1**
    - Type: Merge
    - Role: Combines the two Excel worksheet binaries into one multi-sheet workbook.
    - Configuration: Mode "combine" with "combineAll" option (merges all inputs).
    - Inputs: Outputs from Convert to File and Convert to File1 nodes.
    - Outputs: Combined Excel workbook binary.
    - Edge Cases: Failures if input binaries are incompatible or missing.
  
  - **Save to google sheets**
    - Type: Google Sheets
    - Role: Appends data from Data Set 2 (City, Country) to a designated worksheet in a Google Sheet.
    - Configuration:
      - Operation: "append"
      - SheetName: Uses sheet with ID 1978181834 (linked to "two" worksheet)
      - DocumentId: Specific Google Spreadsheet ID (example sheet)
      - Columns: Auto-mapping on City and Country fields.
      - Credential: Configured Google Sheets OAuth2 credential.
    - Inputs: Output from Merge1 node (note: connection logic sends merged output here, but effectively uses Data Set 2 for append).
    - Outputs: Append operation result.
    - Edge Cases:
      - Auth errors if OAuth token expired or revoked.
      - Schema mismatch if input data columns differ.
      - Quota limits on Google Sheets API.
  
#### 2.4 Google Drive Export

- **Overview**: Downloads the Google Sheet as an Excel file from Google Drive, making the final multi-sheet workbook available for local use or further processing.
- **Nodes Involved**:
  - Export Excel file

- **Node Details**:

  - **Export Excel file**
    - Type: Google Drive
    - Role: Downloads the Google Sheet document as an Excel file.
    - Configuration:
      - Operation: "download"
      - FileId: Specific Google Sheet document ID
      - Credential: Configured Google Drive OAuth2 credential.
    - Inputs: Triggered directly from manual trigger.
    - Outputs: Binary Excel file downloadable or usable in subsequent workflow steps.
    - Edge Cases:
      - Authentication failures.
      - File not found or permission errors.
      - Download timeouts or rate limiting.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                         | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                          |
|-------------------------|--------------------|---------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Starts workflow execution              | None                         | Export Excel file, Data Set 1, Data Set 2 |                                                                                                                                      |
| Data Set 1              | Code               | Generates first dataset (Name, Age)    | When clicking ‘Execute workflow’ | Convert to File              |                                                                                                                                      |
| Data Set 2              | Code               | Generates second dataset (City, Country) | When clicking ‘Execute workflow’ | Convert to File1             |                                                                                                                                      |
| Convert to File         | Convert To File    | Converts first dataset to Excel worksheet "Sheet1" | Data Set 1                   | Merge1                       |                                                                                                                                      |
| Convert to File1        | Convert To File    | Converts second dataset to Excel worksheet "Sheet2" | Data Set 2                   | Merge1                       |                                                                                                                                      |
| Merge1                  | Merge              | Combines two Excel worksheets into one workbook | Convert to File, Convert to File1 | Save to google sheets        |                                                                                                                                      |
| Save to google sheets   | Google Sheets      | Appends Data Set 2 to Google Sheets   | Merge1                       | None                         |                                                                                                                                      |
| Export Excel file       | Google Drive       | Downloads Google Sheet as Excel file   | When clicking ‘Execute workflow’ | Data Set 1, Data Set 2       |                                                                                                                                      |
| Sticky Note60           | Sticky Note        | Instructions for Google Sheets OAuth2 | None                         | None                         | 1️⃣ Connect Google Sheets (OAuth2) Setup steps + example sheet link: https://docs.google.com/spreadsheets/d/1G6FSm3VdMZt6VubM6g8j0mFw59iEw9npJE0upxj3Y6k/edit?gid=1978181834#gid=1978181834 |
| Sticky Note62           | Sticky Note        | Instructions for Google Drive OAuth2  | None                         | None                         | 2️⃣ Connect Google Drive (OAuth2) Setup steps + folder pointing instructions                                                          |
| Sticky Note54           | Sticky Note        | Workflow summary and automation purpose | None                         | None                         | Create multi-sheet Excel workbooks in n8n to automate reporting using Google Drive + Google Sheets                                   |
| Sticky Note2            | Sticky Note        | Setup instructions and contact info   | None                         | None                         | Setup instructions for Google Sheets & Drive OAuth2, contact email and LinkedIn info                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   - No parameters required.

2. **Create Data Set 1 Code Node**
   - Add a **Code** node named `Data Set 1`.
   - Set language to JavaScript.
   - Insert code:
     ```js
     return [
       { json: { Name: "Alice", Age: 30 } },
       { json: { Name: "Bob", Age: 25 } },
       { json: { Name: "Charlie", Age: 35 } },
     ];
     ```
   - Connect output from `When clicking ‘Execute workflow’` to this node.

3. **Create Data Set 2 Code Node**
   - Add a **Code** node named `Data Set 2`.
   - Set language to JavaScript.
   - Insert code:
     ```js
     return [
       { json: { City: "New York", Country: "USA" } },
       { json: { City: "London", Country: "UK" } },
       { json: { City: "Tokyo", Country: "Japan" } },
     ];
     ```
   - Connect output from `When clicking ‘Execute workflow’` to this node.

4. **Create Convert to File Node for Data Set 1**
   - Add a **Convert To File** node named `Convert to File`.
   - Set Operation to `xlsx`.
   - Under options, set `sheetName` to `"Sheet1"`.
   - Connect output of `Data Set 1` to this node.

5. **Create Convert to File Node for Data Set 2**
   - Add a **Convert To File** node named `Convert to File1`.
   - Set Operation to `xlsx`.
   - Under options, set `sheetName` to `"Sheet2"`.
   - Set the binary property name to `data2`.
   - Connect output of `Data Set 2` to this node.

6. **Create Merge Node**
   - Add a **Merge** node named `Merge1`.
   - Set Mode to `combine`.
   - Set Combine By to `combineAll`.
   - Connect outputs of `Convert to File` (main output 0) and `Convert to File1` (main output 0) to inputs 0 and 1 of this node respectively.

7. **Create Google Sheets Node to Append Data**
   - Add a **Google Sheets** node named `Save to google sheets`.
   - Set Operation to `append`.
   - Set Spreadsheet ID to the desired Google Sheet (e.g., `1G6FSm3VdMZt6VubM6g8j0mFw59iEw9npJE0upxj3Y6k`).
   - Set Sheet Name or Sheet ID to the appropriate worksheet (e.g., `1978181834`).
   - Configure Columns with auto-mapping for `City` and `Country`.
   - Connect output of `Merge1` node to this Google Sheets node.
   - Set credentials to a valid **Google Sheets OAuth2** credential.

8. **Create Google Drive Download Node**
   - Add a **Google Drive** node named `Export Excel file`.
   - Set Operation to `download`.
   - Set File ID to the same Google Sheet document ID used above.
   - Connect output of `When clicking ‘Execute workflow’` to this node.
   - Set credentials to a valid **Google Drive OAuth2** credential.

9. **Credential Setup**
   - Create or verify **Google Sheets (OAuth2)** credential with appropriate permissions.
   - Create or verify **Google Drive (OAuth2)** credential with appropriate permissions.
   - Assign these credentials to the respective nodes.

10. **Test Workflow**
    - Save and execute the workflow manually.
    - Confirm that Excel files are generated with two sheets.
    - Confirm the Google Sheet is appended with data.
    - Confirm the Excel file is downloadable from Google Drive.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow automates building Excel files with multiple tabs using n8n, Google Drive, and Google Sheets, eliminating manual reporting tasks. | Sticky Note54 content |
| Setup instructions for Google Sheets OAuth2, including copying example sheet; linked example sheet provided. | https://docs.google.com/spreadsheets/d/1G6FSm3VdMZt6VubM6g8j0mFw59iEw9npJE0upxj3Y6k/edit?gid=1978181834#gid=1978181834 |
| Setup instructions for Google Drive OAuth2, emphasizing folder selection for `.xlsx` files. | Sticky Note62 content |
| Contact for customization support: Robert Breen (email: rbreen@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, Website: https://ynteractive.com) | Sticky Note2 content |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. The process strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.