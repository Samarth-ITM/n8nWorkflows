Construction Blueprint to Google Sheets Automation with VLM Run and Google Drive

https://n8nworkflows.xyz/workflows/construction-blueprint-to-google-sheets-automation-with-vlm-run-and-google-drive-7956


# Construction Blueprint to Google Sheets Automation with VLM Run and Google Drive

### 1. Workflow Overview

This workflow automates the extraction and logging of structured data from construction blueprint files uploaded to a specific Google Drive folder. It is designed for construction, architectural, and engineering professionals who need to systematically process blueprint documents and maintain an up-to-date, searchable record in Google Sheets.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Download**  
  Watches a designated Google Drive folder for new files, triggering the workflow when a file is uploaded, then downloads the file for processing.

- **1.2 AI Processing with VLM Run**  
  Sends the downloaded blueprint file to the VLM Run node, configured with the `construction.blueprint` domain, to extract detailed, structured data from complex construction documents.

- **1.3 Data Logging to Google Sheets**  
  Takes the structured JSON response from VLM Run and appends it as a new row in a preconfigured Google Sheet, mapping detailed blueprint metadata into corresponding columns for tracking and reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Download

- **Overview:**  
  This block monitors a specific Google Drive folder for newly uploaded files and downloads them automatically for subsequent processing.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Download file

- **Node Details:**

  - **Google Drive Trigger**  
    - *Type & Role:* Trigger node; listens for new file creation events in Google Drive.  
    - *Configuration:*  
      - Watches a specific folder (ID: `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7`).  
      - Triggers every minute (`pollTimes` set to everyMinute).  
      - Monitors all file types.  
    - *Input/Output:* No input; outputs metadata of newly created files.  
    - *Potential Failures:* Authentication errors with Google Drive OAuth2; folder access or permission issues; API rate limits.  
    - *Version:* n8n Google Drive Trigger node v1.

  - **Download file**  
    - *Type & Role:* Google Drive node; downloads the file triggered by the previous node.  
    - *Configuration:*  
      - Downloads file by ID taken from trigger output (`{{$json.id}}`).  
      - Operation: download.  
    - *Input:* Receives file metadata from Google Drive Trigger.  
    - *Output:* Binary content of the downloaded file, plus metadata.  
    - *Potential Failures:* File not found; permission denied; file size limits; download timeouts.  
    - *Version:* n8n Google Drive node v3.

- **Notes:**  
  Supported file formats include images (JPG, PNG, WEBP), PDFs, scanned documents, and mobile uploads, ensuring flexibility in source document types.

#### 2.2 AI Processing with VLM Run

- **Overview:**  
  Processes the downloaded blueprint file using the VLM Run AI node to extract rich, structured construction blueprint data in JSON format.

- **Nodes Involved:**  
  - VLM Run

- **Node Details:**

  - **VLM Run**  
    - *Type & Role:* AI processing node from the `@vlm-run/n8n-nodes-vlmrun` package.  
    - *Configuration:*  
      - Domain set to `construction.blueprint` to specify blueprint document processing.  
      - Input expected: binary file from "Download file".  
      - Output: JSON with detailed blueprint metadata fields (e.g., title block, document metadata, drawings).  
    - *Input:* Binary blueprint file.  
    - *Output:* Parsed JSON with structured blueprint details.  
    - *Potential Failures:* API authentication issues; invalid or corrupt file input; processing timeouts; domain-specific parsing errors.  
    - *Version:* v1 of the VLM Run node.

- **Notes:**  
  The node extracts comprehensive information such as project details, drawing titles, revision history, legal compliance, annotations, and author information, converting complex blueprint visuals into machine-readable data.

#### 2.3 Data Logging to Google Sheets

- **Overview:**  
  Maps the structured data from VLM Run into a Google Sheet by appending a new row with multiple blueprint metadata fields for each processed document.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**

  - **Append row in sheet**  
    - *Type & Role:* Google Sheets node; performs append operation to add data rows.  
    - *Configuration:*  
      - Document ID: Google Sheet with ID `1NyNXDvHHJwedBSXa8qENv_4nBlqyaU-l3wgtAK-9Om0`.  
      - Sheet name: `gid=0` (Sheet1).  
      - Operation: Append row.  
      - Columns mapped to JSON paths from VLM Run output, including complex nested fields with inline JavaScript expressions to flatten objects and arrays into formatted strings.  
      - Examples of mapped fields: SCALE, ADDRESS, JOB NAME, REVISION, CHECKED BY, ISSUE DATE, AGENCY NAME, DRAWING TYPE, AUTHOR'S NAME, DOCUMENT TYPE, and many others capturing title block, metadata, and blueprint details.  
    - *Input:* JSON output from VLM Run node.  
    - *Output:* Confirmation of row appended.  
    - *Potential Failures:* Google Sheets API permission errors; quota limits; malformed data causing expression failures; sheet not found or access restricted; data type mismatches.  
    - *Version:* Google Sheets node v4.7.

- **Notes:**  
  The use of complex JavaScript expressions ensures nested and optional data are properly formatted into readable strings, handling nulls, arrays, and objects gracefully to maintain data integrity.

---

### 3. Summary Table

| Node Name          | Node Type                      | Functional Role                          | Input Node(s)     | Output Node(s)         | Sticky Note                                                                                                                   |
|--------------------|--------------------------------|----------------------------------------|-------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger| n8n-nodes-base.googleDriveTrigger | Watches Google Drive folder for new files | -                 | Download file          | # 📁 Input Processing<br>Monitors & downloads blueprint files from Google Drive.<br>Supported formats: images, PDFs, scans.    |
| Download file      | n8n-nodes-base.googleDrive     | Downloads file from Google Drive       | Google Drive Trigger | VLM Run                | # 📁 Input Processing<br>Monitors & downloads blueprint files from Google Drive.<br>Supported formats: images, PDFs, scans.    |
| VLM Run            | @vlm-run/n8n-nodes-vlmrun.vlmRun | Extracts structured blueprint data via AI | Download file       | Append row in sheet    | # VLM Run (Document)<br>Extracts detailed structured data from construction blueprints.<br>Transforms complex documents to JSON.|
| Append row in sheet| n8n-nodes-base.googleSheets    | Appends extracted data into Google Sheets | VLM Run             | -                      | # Append Row in Sheet<br>Appends structured data to Google Sheet columns for tracking blueprints.                              |
| Sticky Note1       | n8n-nodes-base.stickyNote      | Documentation note                     | -                 | -                      | # 📁 Input Processing<br>Monitors & downloads blueprint files from Google Drive.<br>Supported formats: images, PDFs, scans.    |
| Sticky Note2       | n8n-nodes-base.stickyNote      | Documentation note                     | -                 | -                      | # Append Row in Sheet<br>Appends extracted structured data into a Google Sheet.<br>Provides a structured, updated database.     |
| Sticky Note3       | n8n-nodes-base.stickyNote      | Documentation note                     | -                 | -                      | # VLM Run (Document)<br>Extracts structured blueprint details.<br>Turns complex blueprints into machine-readable JSON.          |
| Sticky Note        | n8n-nodes-base.stickyNote      | Documentation note                     | -                 | -                      | # Construction Blueprint Processing with VLM Run<br>Overview and requirements of the workflow.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**  
   - Type: `Google Drive Trigger`  
   - Configure:  
     - Event: `fileCreated`  
     - Polling: every minute  
     - Folder to watch: Set folder ID to `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7` (replace with your target folder)  
     - File types: All  
   - Connect output to "Download file" node.

2. **Create Download file node:**  
   - Type: `Google Drive`  
   - Operation: `download`  
   - File ID: Expression `{{$json.id}}` from Google Drive Trigger output  
   - Connect output to "VLM Run" node.

3. **Create VLM Run node:**  
   - Type: `@vlm-run/n8n-nodes-vlmrun.vlmRun`  
   - Domain: Set to `construction.blueprint`  
   - Input: Binary data from "Download file" node  
   - Connect output to "Append row in sheet" node.

4. **Create Append row in sheet node:**  
   - Type: `Google Sheets`  
   - Operation: `append`  
   - Document ID: `1NyNXDvHHJwedBSXa8qENv_4nBlqyaU-l3wgtAK-9Om0` (set your Google Sheet ID)  
   - Sheet Name: `gid=0` or your target sheet name  
   - Columns: Define column mappings with expressions extracting fields from VLM Run JSON output. Use provided JavaScript expressions to handle nested objects and arrays, for example:  
     - SCALE: `{{$json.response.title_block.scale}}`  
     - ADDRESS: Custom JS to flatten nested address object arrays into strings  
     - JOB NAME, REVISION, CHECKED BY, ISSUE DATE, and other detailed fields similarly mapped  
   - Ensure OAuth2 credentials for Google Sheets are configured.  
   - No output connection needed (end of workflow).

5. **Set Credentials:**  
   - Google Drive Trigger and Download file nodes: Configure Google Drive OAuth2 credentials with necessary folder access.  
   - Append row in sheet node: Configure Google Sheets OAuth2 credentials with write access to the target spreadsheet.  
   - VLM Run node: Configure API credentials or authentication as required by VLM Run service.

6. **Optional:**  
   - Add Sticky Note nodes to document workflow blocks for clarity.  
   - Test workflow by uploading a supported blueprint file (e.g., PDF or image) to the watched Google Drive folder and verify data appends correctly in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow automates extraction of blueprint data for compliance, reporting, and project tracking.                                                                              | Workflow purpose and use cases.                                                                               |
| Requires VLM Run API access and Google OAuth2 credentials for Drive and Sheets.                                                                                               | Prerequisites for running the workflow.                                                                      |
| Inline JavaScript expressions carefully flatten complex nested JSON into readable string fields for Google Sheets compatibility.                                            | Important for maintaining data integrity in sheet columns.                                                   |
| Supported file formats include images (JPG, PNG, WEBP), PDFs, scanned documents, and mobile uploads.                                                                         | Broad input format support.                                                                                    |
| [Google Sheets API documentation](https://developers.google.com/sheets/api)                                                                                                | For advanced sheet operations or troubleshooting.                                                            |
| [VLM Run Documentation](https://vlm.run/docs) (Assumed link for VLM Run node setup and domain usage; replace with actual URL)                                               | For understanding node domain configurations and API capabilities.                                          |
| Workflow tested with n8n version supporting Google Drive nodes v3, Google Sheets nodes v4.7, and VLM Run node v1.                                                             | Version compatibility notes.                                                                                   |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.