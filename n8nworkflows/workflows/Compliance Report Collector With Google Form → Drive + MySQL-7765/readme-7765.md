Compliance Report Collector With Google Form → Drive + MySQL

https://n8nworkflows.xyz/workflows/compliance-report-collector-with-google-form---drive---mysql-7765


# Compliance Report Collector With Google Form → Drive + MySQL

### 1. Workflow Overview

This workflow automates the collection and logging of compliance reports submitted via a Google Form linked to a Google Sheets spreadsheet. When a new form response is received, it extracts relevant metadata about the submission and the associated uploaded file stored in Google Drive, then consolidates this data and logs it into a MySQL database.

The workflow is logically divided into these blocks:

- **1.1 Google Sheets Form Trigger:** Watches the Google Sheet for new rows added by form submissions, triggering the workflow.
- **1.2 Extract Google Sheets Data:** Parses and extracts key fields from the new row, including extracting the Google Drive file ID from the uploaded file URL.
- **1.3 Fetch File Metadata from Google Drive:** Retrieves metadata (file name and MIME type) for the uploaded file using the Drive API.
- **1.4 Merge Data:** Combines form submission data with the file metadata into a single structured object.
- **1.5 Rename and Normalize Fields:** Adjusts field names to match the MySQL database schema and removes unnecessary fields.
- **1.6 Log to MySQL:** Inserts the cleaned data into a MySQL table for persistent storage and future compliance auditing.

---

### 2. Block-by-Block Analysis

#### 1.1 Google Sheets Form Trigger

- **Overview:** Monitors the linked Google Sheets document for newly added rows, indicating new form submissions.
- **Nodes Involved:** `Google Sheets form Trigger`
- **Node Details:**
  - **Type & Role:** `googleSheetsTrigger` node; event-based trigger to detect new rows.
  - **Configuration:** Polls every minute, watching the sheet named “compliance_report_sheet(Responses)” within the specified Google Sheets document (document ID: `1U4MfFW06VMW0gW3fAkGFDvDnomb1j_0E-0lbEkPlli0`).
  - **Expressions:** None beyond document and sheet selection.
  - **Input/Output:** No input; outputs new row data as JSON.
  - **Credentials:** Uses OAuth2 credentials linked to a Google Sheets account.
  - **Failure Modes:** Possible auth failures, permission issues if the sheet is moved or renamed, or API quota limitations.
  - **Sub-workflows:** None.

#### 1.2 Extract Google Sheets Data

- **Overview:** Parses incoming row data, extracts relevant fields and the Google Drive file ID from the uploaded report URL.
- **Nodes Involved:** `Extract googlesheet data`
- **Node Details:**
  - **Type & Role:** `function` node; custom JavaScript to process input data.
  - **Configuration:** Processes all new rows; extracts fields:
    - `reporter` from “Reporter Name”
    - `category` from “Report Category”
    - `timestamp` parsed and converted to ISO string
    - `email` from “Email Address”
    - `description` from “Description”
    - `folder_id` extracted from URL in “Upload Report File” field using regex to find Google Drive file ID from either query params or path.
  - **Expressions:** Uses `$input.all()`, regex matching, and date parsing.
  - **Input/Output:** Input from Google Sheets Trigger node; outputs JSON objects with extracted fields.
  - **Failure Modes:** Missing or malformed URLs could lead to null `folder_id`; date parsing errors if timestamp format varies.
  - **Sub-workflows:** None.

#### 1.3 Fetch File Metadata from Google Drive

- **Overview:** Retrieves binary file metadata for the uploaded report file from Google Drive using the extracted file ID.
- **Nodes Involved:** `Get file uploaded metadata`
- **Node Details:**
  - **Type & Role:** `googleDrive` node; downloads the file using file ID.
  - **Configuration:** Operation set to “download” with file ID dynamically set from the previous node’s `folder_id`.
  - **Expressions:** File ID expression: `={{ $json.folder_id }}`
  - **Input/Output:** Input from Extract Google Sheets Data node; outputs binary file data plus metadata.
  - **Credentials:** Uses OAuth2 credentials for Google Drive.
  - **Failure Modes:** Invalid or missing file ID; permissions errors if the file is not accessible; API rate limits; file deleted or moved.
  - **Sub-workflows:** None.

#### 1.4 Merge Data

- **Overview:** Combines the JSON data from the form submission and the binary metadata from Google Drive into a single JSON object per submission.
- **Nodes Involved:** `Merge sheet & file data`
- **Node Details:**
  - **Type & Role:** `merge` node; combines data streams.
  - **Configuration:** Mode set to “combine” by position, including unpaired items to ensure no data loss.
  - **Input/Output:** Inputs: data from Google Drive node and Extract Google Sheets Data node; output: merged JSON.
  - **Failure Modes:** Mismatched data arrays could cause incomplete merging; unexpected missing data.
  - **Sub-workflows:** None.

#### 1.5 Rename and Normalize Fields

- **Overview:** Normalizes field names to match the MySQL database schema and filters out unnecessary fields.
- **Nodes Involved:** `Rename Fields`
- **Node Details:**
  - **Type & Role:** `set` node; manually sets and renames fields.
  - **Configuration:** Uses raw JSON mode to rename:
    - `fileName` → `file_name`
    - `mimeType` → `mime_type`
  - Only includes fields: `reporter`, `category`, `timestamp`, `folder_id`, `file_name`, `mime_type`, `email`, `description`.
  - **Expressions:** Uses expressions like `{{$json.reporter}}` and `{{$binary.data.fileName}}`.
  - **Input/Output:** Input from Merge node; output normalized JSON.
  - **Failure Modes:** Missing binary data could cause null `file_name` or `mime_type`; expression evaluation errors.
  - **Sub-workflows:** None.

#### 1.6 Log to MySQL

- **Overview:** Inserts the normalized compliance report data into the MySQL table `report_logs`.
- **Nodes Involved:** `Log to MySQL`
- **Node Details:**
  - **Type & Role:** `mySql` node; executes insert operation.
  - **Configuration:** Target table: `report_logs`; default insert operation.
  - **Input/Output:** Input from Rename Fields node; outputs execution result.
  - **Credentials:** Uses MySQL credentials.
  - **Failure Modes:** Connection errors, auth failures, SQL errors if table schema mismatch or data constraint violations, network timeouts.
  - **Sub-workflows:** None.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                     | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                  |
|---------------------------|-------------------------|-----------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Google Sheets form Trigger | googleSheetsTrigger     | Trigger on new form submission    | —                          | Extract googlesheet data  |                                                                                              |
| Extract googlesheet data   | function                | Parse form data, extract file ID  | Google Sheets form Trigger | Get file uploaded metadata| "Extract metadata for logging"                                                              |
| Get file uploaded metadata | googleDrive             | Download uploaded file metadata   | Extract googlesheet data   | Merge sheet & file data   |                                                                                              |
| Merge sheet & file data    | merge                   | Combine form & file metadata       | Get file uploaded metadata | Rename Fields            |                                                                                              |
| Rename Fields             | set                     | Normalize field names for MySQL    | Merge sheet & file data    | Log to MySQL             |                                                                                              |
| Log to MySQL              | mySql                   | Insert data into MySQL             | Rename Fields             | —                        |                                                                                              |
| Sticky Note               | stickyNote              | Documentation note                 | —                          | —                        | "## Compliance Report Collector (Google Form → Drive + MySQL)"                              |
| Sticky Note1              | stickyNote              | Detailed workflow description      | —                          | —                        | Detailed block explanations and workflow description                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger Node**
   - Node Type: Google Sheets Trigger
   - Purpose: To trigger the workflow when a new row is added.
   - Configuration:
     - Event: `rowAdded`
     - Polling interval: every 1 minute
     - Document ID: `1U4MfFW06VMW0gW3fAkGFDvDnomb1j_0E-0lbEkPlli0`
     - Sheet Name: `compliance_report_sheet(Responses)`
   - Credentials: Assign OAuth2 credentials for Google Sheets with access to the document.

2. **Add a Function Node to Extract Google Sheet Data**
   - Node Type: Function
   - Purpose: Extract relevant fields and parse the Google Drive file ID from the form URL.
   - Code: Use the provided JavaScript function to parse each row and extract:
     - Reporter Name
     - Report Category
     - Timestamp (converted to ISO string)
     - Email Address
     - Description
     - Extract file ID from "Upload Report File" URL using regex.
   - Connect input from Google Sheets Trigger.

3. **Add a Google Drive Node to Fetch File Metadata**
   - Node Type: Google Drive
   - Purpose: Download the file to access metadata (name, MIME type).
   - Configuration:
     - Operation: Download
     - File ID: Expression referencing `folder_id` from previous function node: `={{ $json.folder_id }}`
   - Credentials: Assign OAuth2 credentials for Google Drive with permission to access files.
   - Connect input from the Function node.

4. **Add a Merge Node to Combine Data**
   - Node Type: Merge
   - Purpose: Combine JSON data from the Function node and file metadata from Google Drive node.
   - Configuration:
     - Mode: Combine
     - Combine by: Position
     - Include unpaired items: Enabled
   - Connect inputs from Google Drive node and Function node.

5. **Add a Set Node to Rename and Normalize Fields**
   - Node Type: Set
   - Purpose: Rename fields to match MySQL columns and remove extraneous data.
   - Configuration:
     - Mode: Raw JSON
     - JSON:
       ```json
       {
         "reporter": "={{$json.reporter}}",
         "category": "={{$json.category}}",
         "timestamp": "={{$json.timestamp}}",
         "folder_id": "={{$json.folder_id}}",
         "file_name": "={{$binary.data.fileName}}",
         "mime_type": "={{$binary.data.mimeType}}",
         "email": "={{$json.email}}",
         "description": "={{$json.description}}"
       }
       ```
   - Connect input from Merge node.

6. **Add a MySQL Node to Log Data**
   - Node Type: MySQL
   - Purpose: Insert the normalized data into the `report_logs` table.
   - Configuration:
     - Operation: Insert
     - Table: `report_logs`
   - Credentials: Assign MySQL credentials with write permissions.
   - Connect input from Set node.

7. **Add Sticky Note Nodes (Optional)**
   - To document workflow overview and block explanations.
   - Node Type: Sticky Note
   - Content: Use the provided descriptions for clarity.

8. **Connect Nodes Sequentially**
   - Google Sheets form Trigger → Extract googlesheet data → Get file uploaded metadata → Merge sheet & file data → Rename Fields → Log to MySQL

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow is designed to handle Google Form submissions where users upload compliance reports as files.   | Project context: Compliance reporting automation                                                      |
| The Google Drive file ID extraction supports multiple URL formats: query param `?id=` or path `/d/.../view`. | Reference for Google Drive file URLs: https://developers.google.com/drive/api/v3/file#file_ids         |
| Ensure Google API credentials have sufficient permissions for Sheets and Drive access.                        | Google Cloud Console setup instructions: https://console.cloud.google.com/apis/credentials             |
| MySQL table `report_logs` schema must align with the renamed fields in the workflow.                          | Example schema fields: reporter, category, timestamp, folder_id, file_name, mime_type, email, description |
| Regular monitoring and error handling should be implemented for production use to catch API or DB failures.  | n8n documentation on error workflows: https://docs.n8n.io/nodes/trigger/#error-trigger                  |
| Workflow is inactive by default; activate to start listening for new form submissions.                        |                                                                                                       |

---

**Disclaimer:** The text provided is derived solely from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.