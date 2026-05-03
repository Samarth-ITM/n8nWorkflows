Export JSON file to Google Sheets

https://n8nworkflows.xyz/workflows/export-json-file-to-google-sheets-1736


# Export JSON file to Google Sheets

### 1. Workflow Overview

This workflow automates the export of data from a local JSON file into a Google Sheets spreadsheet. It is designed for use cases where data stored in JSON format needs to be appended regularly or periodically to a Google Sheet for reporting, collaboration, or further processing.

The workflow is divided into three logical blocks:  
- **1.1 JSON File Reading:** Reads the JSON file from the local filesystem.  
- **1.2 Binary Data Handling:** Converts or prepares the binary JSON data for further processing.  
- **1.3 Google Sheets Append:** Appends the processed JSON data rows into a specified Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 JSON File Reading

- **Overview:**  
  Reads a JSON file from the specified local path and outputs the file content as binary data.

- **Nodes Involved:**  
  - `read json file`

- **Node Details:**  

  - **Node Name:** read json file  
  - **Type:** Read Binary File  
  - **Technical Role:** Reads a binary file (here a JSON file) from the host filesystem into the workflow.  
  - **Configuration:**  
    - File path set to `/username/users_spreadsheet.json`.  
  - **Key expressions/variables:** None used.  
  - **Input:** None (start node).  
  - **Output:** Binary data representing the JSON file content.  
  - **Version Requirements:** None specific, standard for n8n.  
  - **Potential Failure Modes:**  
    - File not found or inaccessible path leading to read errors.  
    - Permission errors on file system access.  
  - **Sub-workflow Reference:** None.

#### 1.2 Binary Data Handling

- **Overview:**  
  Processes the binary data output from the previous node, preparing it for Google Sheets input. This node typically converts binary data to JSON or extracts the data parts required.

- **Nodes Involved:**  
  - `move binary data 2`

- **Node Details:**  

  - **Node Name:** move binary data 2  
  - **Type:** Move Binary Data  
  - **Technical Role:** Converts or manipulates binary data within the workflow, commonly to expose the JSON content as workflow JSON data.  
  - **Configuration:**  
    - No additional options configured (default).  
  - **Key expressions/variables:** None.  
  - **Input:** Receives binary data from `read json file`.  
  - **Output:** JSON data ready for downstream usage.  
  - **Version Requirements:** None specific.  
  - **Potential Failure Modes:**  
    - If binary data is not valid JSON, conversion may fail.  
  - **Sub-workflow Reference:** None.

#### 1.3 Google Sheets Append

- **Overview:**  
  Appends the processed JSON data as rows into a specific Google Sheets spreadsheet and range.

- **Nodes Involved:**  
  - `Google Sheets1`

- **Node Details:**  

  - **Node Name:** Google Sheets1  
  - **Type:** Google Sheets  
  - **Technical Role:** Appends rows into a Google Sheets spreadsheet.  
  - **Configuration:**  
    - Operation: Append  
    - Target Sheet ID: `qwertz` (represents the spreadsheet identifier).  
    - Range: `A:C` (columns A to C).  
    - Option "Use Path For Key Row": enabled, which maps JSON keys to sheet columns based on the JSON structure.  
    - Authentication: OAuth2 via configured Google Sheets OAuth2 credentials.  
  - **Key expressions/variables:** None specified beyond sheet range and ID.  
  - **Input:** JSON data from `move binary data 2`.  
  - **Output:** Confirmation or result of append operation.  
  - **Version Requirements:** OAuth2 authentication requires correct credential setup with Google API access.  
  - **Potential Failure Modes:**  
    - Authorization errors if OAuth token is invalid or expired.  
    - Network or API quota errors.  
    - Mismatch between JSON keys and sheet columns causing data misalignment.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role              | Input Node(s)        | Output Node(s)       | Sticky Note                    |
|------------------|---------------------|-----------------------------|----------------------|----------------------|-------------------------------|
| read json file   | Read Binary File    | Reads JSON binary data from file | None                 | move binary data 2   |                               |
| move binary data 2| Move Binary Data     | Converts binary JSON to usable JSON | read json file       | Google Sheets1       |                               |
| Google Sheets1   | Google Sheets        | Appends JSON data rows to Google Sheets | move binary data 2   | None                 | Append data to sheet           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Read Binary File" node:**  
   - Name: `read json file`  
   - Set `File Path` parameter to `/username/users_spreadsheet.json` (adjust path as needed).  
   - No credentials needed.  
   - This node will read the JSON file as binary data.

3. **Add a "Move Binary Data" node:**  
   - Name: `move binary data 2`  
   - Leave default settings (no additional options).  
   - Connect the output of `read json file` to the input of `move binary data 2`.  
   - This node prepares the binary JSON data for further use.

4. **Add a "Google Sheets" node:**  
   - Name: `Google Sheets1`  
   - Set operation to `Append`.  
   - Specify the `Sheet ID` as `qwertz` (replace with the actual Google Sheets ID).  
   - Set `Range` to `A:C` to target columns A to C.  
   - Enable the option `Use Path For Key Row` to map JSON keys to columns automatically.  
   - Configure authentication to use OAuth2 credentials for Google Sheets:  
     - Create or select existing Google Sheets OAuth2 credentials with appropriate scopes (e.g., `https://www.googleapis.com/auth/spreadsheets`).  
   - Connect the output of `move binary data 2` to the input of `Google Sheets1`.

5. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                      | Context or Link                               |
|-------------------------------------------------|-----------------------------------------------|
| OAuth2 credentials must be configured in n8n to access Google Sheets API successfully. | Google Cloud Console: https://console.cloud.google.com/apis/credentials |
| The `Use Path For Key Row` option maps JSON keys to columns, so ensure JSON keys match expected sheet columns. | n8n Google Sheets node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/ |
| File path must be accessible by the n8n instance; use absolute paths or ensure file permissions allow reading. | n8n documentation on File System nodes: https://docs.n8n.io/nodes/nodes-library/nodes/ReadBinaryFile/ |

---

This document fully describes the workflow "Export JSON file to Google Sheets," enabling advanced users and automation agents to understand, reproduce, and maintain it effectively.