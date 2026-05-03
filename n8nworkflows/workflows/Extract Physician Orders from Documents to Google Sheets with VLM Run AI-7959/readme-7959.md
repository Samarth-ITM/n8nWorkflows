Extract Physician Orders from Documents to Google Sheets with VLM Run AI

https://n8nworkflows.xyz/workflows/extract-physician-orders-from-documents-to-google-sheets-with-vlm-run-ai-7959


# Extract Physician Orders from Documents to Google Sheets with VLM Run AI

### 1. Workflow Overview

This n8n workflow is designed to automate the extraction of structured physician order data from documents uploaded to a specific Google Drive folder and append this data into a Google Sheet. It is targeted at healthcare use cases involving physician orders, such as Durable Medical Equipment (DME) requests, prescription forms, and treatment orders.

The core logic is grouped into three functional blocks:

- **1.1 Input Reception and File Download**: Watches a specific Google Drive folder for new file uploads and downloads the detected files.

- **1.2 AI Processing with VLM Run**: Sends the downloaded physician order documents to the VLM Run AI service under the `healthcare.physician-order` domain to extract structured data.

- **1.3 Data Storage in Google Sheets**: Appends the AI-extracted structured physician order details into a configured Google Sheet, organizing key patient, physician, and order information for downstream use.

This workflow supports multiple file formats commonly used in healthcare documentation, including PDFs, images, and scanned receipts. It requires OAuth2 credentials for Google Drive and Google Sheets, as well as API access to VLM Run.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Download

- **Overview:**  
  This block monitors a designated Google Drive folder for any newly uploaded files and downloads these files immediately upon detection to prepare them for AI processing.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Download file  
  - Sticky Note4

- **Node Details:**

  - **Google Drive Trigger**  
    - *Type & Role:* Event trigger node that listens for new file creations in Google Drive.  
    - *Configuration:*  
      - Watches the folder with ID `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7` (a specific folder reserved for test_data or physician order uploads).  
      - Triggers every minute to check for new files.  
      - Filters on all file types.  
    - *Input/Output:* No input; outputs metadata of new files detected.  
    - *Edge Cases:* Folder ID must be correct and accessible by OAuth credentials. Possible failures include expired tokens or permission errors. Network latency could delay trigger firing.  
    - *Sticky Note4:* Explains this node monitors and downloads physician order files with supported formats like PDFs, images, scanned receipts.

  - **Download file**  
    - *Type & Role:* Downloads the detected file from Google Drive to provide the actual document content for processing.  
    - *Configuration:* File ID dynamically set from the trigger node’s output (`{{$json.id}}`).  
    - *Input/Output:* Input is file metadata from Google Drive Trigger; outputs file binary data.  
    - *Edge Cases:* File access permissions, deleted files after trigger, download timeouts.  
    - *Dependencies:* Requires valid Google Drive OAuth2 credentials.

  - **Sticky Note4**  
    - *Purpose:* Describes the input block’s function and supported file types.

---

#### 2.2 AI Processing with VLM Run

- **Overview:**  
  This block sends the downloaded physician order document to the VLM Run AI service to extract structured clinical and order information as JSON.

- **Nodes Involved:**  
  - VLM Run  
  - Sticky Note3

- **Node Details:**

  - **VLM Run**  
    - *Type & Role:* Third-party AI node that processes documents under the healthcare domain to extract physician order data.  
    - *Configuration:*  
      - Domain set to `healthcare.physician-order`.  
      - Uses the downloaded file binary implicitly passed from the previous node.  
    - *Input/Output:* Input is the downloaded file binary; output is JSON containing extracted fields like patient info, exam data, physician details, and additional notes.  
    - *Edge Cases:* Possible API timeouts, authentication errors, format incompatibility, or incomplete extraction if document quality is poor.  
    - *Sticky Note3:* Details the structured data extracted including patient demographics, ordered items, physician details, and signature dates.

---

#### 2.3 Data Storage in Google Sheets

- **Overview:**  
  This block appends the structured data extracted by VLM Run into a specified Google Sheet, organizing key details into columns for record keeping.

- **Nodes Involved:**  
  - Append row in sheet  
  - Sticky Note2

- **Node Details:**

  - **Append row in sheet**  
    - *Type & Role:* Google Sheets node that appends a new row with data mapped from the AI output.  
    - *Configuration:*  
      - Target spreadsheet ID: `1sxwN8f1BTLkb3zvSmyj5HQ9iOLbqNNeDjCt0JuwzsFY`  
      - Target sheet: `gid=0` (Sheet1)  
      - Columns mapped explicitly to extracted JSON fields, e.g., `PATIENT NAME` from `$json.response.patient_info.name.full_name`, `DIAGNOSIS` from `$json.response.physician_info.diagnosis`, etc.  
      - Schema defines column types and visibility; conversion disabled to preserve string types.  
    - *Input/Output:* Input is structured JSON from VLM Run; output is success/failure of append operation.  
    - *Edge Cases:* Google Sheets API quota limits, invalid data types, missing fields in JSON causing null entries, OAuth credential expiry.  
    - *Sticky Note2:* Explains purpose is to maintain a structured, continually updated order database.

---

### 3. Summary Table

| Node Name          | Node Type                       | Functional Role                    | Input Node(s)       | Output Node(s)       | Sticky Note                                                    |
|--------------------|--------------------------------|----------------------------------|---------------------|----------------------|----------------------------------------------------------------|
| Google Drive Trigger| n8n-nodes-base.googleDriveTrigger | Monitors new files in Drive folder | None                | Download file        | # 📁 Input Processing: Watches folder, triggers on new uploads, supports PDF, images, scanned receipts |
| Download file      | n8n-nodes-base.googleDrive      | Downloads uploaded file for processing | Google Drive Trigger | VLM Run              | # 📁 Input Processing (see above)                              |
| VLM Run            | @vlm-run/n8n-nodes-vlmrun.vlmRun | Extracts structured physician order data | Download file        | Append row in sheet   | # VLM Run (Document): Extracts structured patient, order, and physician info from file |
| Append row in sheet| n8n-nodes-base.googleSheets     | Appends extracted data into Google Sheet | VLM Run               | None                 | # Append Row in Sheet: Builds structured database of physician orders |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: `Google Drive Trigger`  
   - Configure to watch for `fileCreated` events.  
   - Set `folderToWatch` to the Drive folder ID where physician order files will be uploaded (e.g., `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7`).  
   - Set polling interval to every minute.  
   - Authenticate with Google OAuth2 credentials that have access to the folder.

2. **Create Download file Node**  
   - Type: `Google Drive` node with `download` operation.  
   - Set `fileId` parameter to the expression `{{$json.id}}` from the Google Drive Trigger output.  
   - Connect input from the Google Drive Trigger node.  
   - Use the same or valid Google OAuth2 credentials.

3. **Create VLM Run Node**  
   - Type: `@vlm-run/n8n-nodes-vlmrun.vlmRun` (requires installation of VLM Run package).  
   - Set domain parameter to `healthcare.physician-order`.  
   - Connect input from the Download file node.  
   - Ensure VLM Run API credentials/configuration is properly set in n8n credentials.

4. **Create Append row in sheet Node**  
   - Type: `Google Sheets` node with `append` operation.  
   - Set `documentId` to the target Google Sheet ID (e.g., `1sxwN8f1BTLkb3zvSmyj5HQ9iOLbqNNeDjCt0JuwzsFY`).  
   - Set `sheetName` to `gid=0` (or appropriate sheet name).  
   - Define columns mapping explicitly according to the extracted JSON from VLM Run, for example:  
     - `PATIENT NAME` → `{{$json.response.patient_info.name.full_name}}`  
     - `DIAGNOSIS` → `{{$json.response.physician_info.diagnosis}}`  
     - Map all relevant fields as defined in the original workflow’s schema.  
   - Connect input from VLM Run node.  
   - Authenticate with Google Sheets OAuth2 credentials.

5. **Link Workflow Nodes**  
   - Connect Google Drive Trigger → Download file → VLM Run → Append row in sheet.

6. **Test Workflow**  
   - Upload a sample physician order document into the designated Google Drive folder.  
   - Observe workflow triggering, downloading, AI processing, and row appending.  
   - Debug any errors such as access permissions or malformed data.

7. **Add Sticky Notes (Optional)**  
   - Add notes describing each block’s purpose and key information for maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow automatically transforms complex medical order forms into structured JSON and logs into Google Sheets | Ideal for Durable Medical Equipment orders, prescriptions, Medicaid submissions, and compliance tracking           |
| Requires active VLM Run API access and Google OAuth2 credentials with Drive and Sheets scopes                   | Ensure credentials are valid and have necessary permissions                                                          |
| Supported input files include PDFs, JPG, PNG, WEBP, and scanned receipts                                         | Facilitates flexible input from multiple healthcare document sources                                                 |
| Google Sheet uses a structured schema with columns like PATIENT NAME, DOB, DIAGNOSIS, PHYSICIAN NAME, EXAM DATE | Enables downstream reporting, auditing, or integrations with other systems                                           |
| Refer to VLM Run documentation for domain-specific extraction details                                           | https://vlmrun.com/docs                                                                                              |
| n8n community and docs for Google Drive and Sheets OAuth2 setup                                                 | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleDrive/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleSheets/ |

---

This completes the comprehensive documentation and reference for the "Extract Physician Orders from Documents to Google Sheets with VLM Run AI" workflow.