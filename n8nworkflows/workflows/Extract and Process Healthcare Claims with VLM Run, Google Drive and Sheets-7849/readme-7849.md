Extract and Process Healthcare Claims with VLM Run, Google Drive and Sheets

https://n8nworkflows.xyz/workflows/extract-and-process-healthcare-claims-with-vlm-run--google-drive-and-sheets-7849


# Extract and Process Healthcare Claims with VLM Run, Google Drive and Sheets

### 1. Workflow Overview

This workflow automates the extraction and processing of healthcare claim documents uploaded to a designated Google Drive folder. It is designed to monitor for new healthcare claim files, download them, extract structured claim data using VLM Run’s specialized healthcare claims-processing domain, and log the extracted information into a Google Sheet for tracking, compliance, or reporting purposes.

Logical blocks included are:

- **1.1 Input Reception:** Detects new file uploads in a specific Google Drive folder.
- **1.2 File Acquisition:** Downloads the actual file (PDF/image) to obtain binary content.
- **1.3 AI-Powered Data Extraction:** Sends the downloaded file to VLM Run for structured data extraction using the healthcare claims-processing model.
- **1.4 Data Recording:** Appends the extracted structured data as a new row in a Google Sheet for record keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Watches a specific Google Drive folder for new file uploads (healthcare claim forms) and passes the file metadata downstream.

- **Nodes Involved:**  
  - Google Drive Trigger

- **Node Details:**

  - **Google Drive Trigger**  
    - Type: Trigger  
    - Role: Watches a designated Google Drive folder for newly created files (fileCreated event).  
    - Configuration:  
      - Event type: `fileCreated`  
      - Poll frequency: Every minute  
      - Watches specific folder: Folder with ID `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7`  
    - Inputs: None (trigger node)  
    - Outputs: Metadata of detected file including file ID, name, and MIME type  
    - Credential: Google Drive OAuth2 account  
    - Edge cases / Failure modes:  
      - Authentication/authorization errors if token expires or is invalid  
      - Network timeouts or API rate limits  
      - Folder ID changes or permission revocation  
    - Notes:  
      - Automates intake of claim files without manual intervention  
      - Sticky note describes this function and benefits

#### 2.2 File Acquisition

- **Overview:**  
  Downloads the uploaded file from Google Drive to provide the actual document content (binary data) for processing.

- **Nodes Involved:**  
  - Download file

- **Node Details:**

  - **Download file**  
    - Type: Google Drive node  
    - Role: Downloads the file identified by the file ID from the trigger node.  
    - Configuration:  
      - Operation: `download`  
      - File ID: Uses expression `={{ $json.id }}` from the Google Drive Trigger output  
    - Inputs: Receives file metadata from Google Drive Trigger  
    - Outputs: Binary data of the downloaded file (e.g., PDF/image)  
    - Credential: Google Drive OAuth2 account  
    - Edge cases / Failure modes:  
      - File not found or deleted between detection and download  
      - Permission errors for the file  
      - Network errors or API throttling  
      - Large files causing memory or timeout issues  
    - Notes:  
      - Provides clean binary input for the VLM Run node  
      - Sticky note explains function and benefit  

#### 2.3 AI-Powered Data Extraction

- **Overview:**  
  Sends the downloaded healthcare claim document to VLM Run’s AI service specialized for healthcare claims processing to extract structured data fields.

- **Nodes Involved:**  
  - VLM Run

- **Node Details:**

  - **VLM Run**  
    - Type: VLM Run Document Processing node  
    - Role: Processes the binary healthcare claim document to extract structured JSON data such as patient, claim, provider, and payer details.  
    - Configuration:  
      - Domain: `healthcare.claims-processing` (specialized model for healthcare claims)  
    - Inputs: Receives binary file data from Download file node  
    - Outputs: JSON object containing extracted fields such as form type, amounts, insured/patient demographics, accident info, dates, insurance details, physician info, and processing notes.  
    - Credential: VLM Run API credentials  
    - Edge cases / Failure modes:  
      - API authentication failure or quota exhaustion  
      - Parsing errors if document format unsupported or corrupted  
      - Unexpected or incomplete document content causing missing fields  
      - Network or timeout issues with the API  
    - Notes:  
      - Enables conversion of complex medical claim forms into machine-readable data  
      - Sticky note details extraction fields and benefits  

#### 2.4 Data Recording

- **Overview:**  
  Appends the structured data extracted from the claims document as a new row in a designated Google Sheet for ongoing tracking and reporting.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets node  
    - Role: Inserts a new row into a specific Google Sheet with all extracted claim fields mapped to columns.  
    - Configuration:  
      - Operation: `append`  
      - Document ID: Spreadsheet ID `1wZQDu7Hd0iV8dqiLbfUzAwUqKnn8J0ZQFO1weZ_ShxQ`  
      - Sheet: Sheet1 (gid=0)  
      - Columns: Mapped explicitly from the VLM Run JSON response, covering over 50 fields (e.g., FORM TYPE, AMOUNT DUE, INSURED NAME, PATIENT ADDRESS, REFERRING PHYSICIAN NAME, etc.)  
      - Mapping Mode: Explicitly defined columns, no type conversion enabled  
    - Inputs: JSON data from VLM Run node  
    - Outputs: Confirmation of row append operation  
    - Credential: Google Sheets OAuth2 account  
    - Edge cases / Failure modes:  
      - Authentication or permission errors on Google Sheets  
      - Schema mismatch or missing fields causing append failures  
      - API rate limits or network issues  
      - Large payloads exceeding cell limits  
    - Notes:  
      - Provides a structured, continuously updated database for healthcare claim tracking  
      - Sticky note summarizes function and benefits  

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                         | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                                               |
|---------------------|--------------------------------|---------------------------------------|---------------------|------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger | n8n-nodes-base.googleDriveTrigger | Watches Google Drive folder for new files | None                | Download file          | # Google Drive Trigger: Watches folder for new uploads; automates intake; passes metadata downstream                      |
| Download file       | n8n-nodes-base.googleDrive     | Downloads the actual uploaded file     | Google Drive Trigger | VLM Run                | # Download File: Downloads binary file for processing; ensures clean input for VLM Run                                     |
| VLM Run             | @vlm-run/n8n-nodes-vlmrun.vlmRun | Extracts structured claim data via AI | Download file       | Append row in sheet    | # VLM Run (Document): Converts medical form to JSON; extracts patient, claim, provider, payer info                        |
| Append row in sheet | n8n-nodes-base.googleSheets    | Appends extracted data to Google Sheet | VLM Run             | None                   | # Append Row in Sheet: Records structured data in Google Sheet for tracking and reporting                                 |
| Sticky Note         | n8n-nodes-base.stickyNote      | Documentation / explanation            | None                | None                   | # Health Care Claim Processing overview and usage description                                                             |
| Sticky Note1        | n8n-nodes-base.stickyNote      | Documentation for Google Drive Trigger | None                | None                   | # Google Drive Trigger function and benefits                                                                              |
| Sticky Note4        | n8n-nodes-base.stickyNote      | Documentation for Download file node  | None                | None                   | # Download File function and benefits                                                                                      |
| Sticky Note3        | n8n-nodes-base.stickyNote      | Documentation for VLM Run node        | None                | None                   | # VLM Run function and benefit                                                                                            |
| Sticky Note2        | n8n-nodes-base.stickyNote      | Documentation for Append row in sheet | None                | None                   | # Append Row in Sheet function and benefit                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: `Google Drive Trigger`  
   - Set event to `fileCreated`  
   - Set polling interval to every minute  
   - Enable trigger on specific folder with folder ID: `1E8rvLEWKguorMT36yCD1jY78G0u8g6g7`  
   - Attach Google Drive OAuth2 credentials  
   - This node will output metadata of newly uploaded files

2. **Create Download file node**  
   - Type: `Google Drive`  
   - Set operation to `download`  
   - For file ID parameter, use expression: `={{ $json.id }}` (from Google Drive Trigger output)  
   - Attach Google Drive OAuth2 credentials (can be same as trigger)  
   - Connect Google Drive Trigger output to Download file input  
   - This node downloads the actual file content (binary) for processing

3. **Create VLM Run node**  
   - Type: `VLM Run` (custom node from `@vlm-run/n8n-nodes-vlmrun`)  
   - Set domain to `healthcare.claims-processing` to use the healthcare claim extraction model  
   - Attach VLM Run API credentials  
   - Connect Download file node output to VLM Run node input  
   - This node sends the binary file to VLM Run for AI-powered extraction of structured claim data

4. **Create Append row in sheet node**  
   - Type: `Google Sheets`  
   - Set operation to `append`  
   - Specify the Google Sheet document ID: `1wZQDu7Hd0iV8dqiLbfUzAwUqKnn8J0ZQFO1weZ_ShxQ`  
   - Set sheet name to “Sheet1” (gid=0)  
   - Define columns explicitly, mapping each column to corresponding fields from VLM Run output JSON, e.g.:  
     - FORM TYPE → `={{ $json.response.form_type }}`  
     - AMOUNT DUE → `={{ $json.response.balance_due }}`  
     - INSURED NAME → `={{ $json.response.insured_name }}`  
     - … and so on for all fields (50+ fields as per mapping in original workflow)  
   - Disable attempt to convert types and convert fields to string options  
   - Attach Google Sheets OAuth2 credentials  
   - Connect VLM Run node output to Append row in sheet input

5. **(Optional) Add sticky notes for documentation**  
   - Add sticky notes describing each major block for clarity and maintainability

6. **Activate the workflow**  
   - Once all nodes are configured and connected, activate the workflow to enable continuous monitoring and processing

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Health Care Claim Processing with VLM Run automates extraction from healthcare claim forms uploaded to Google Drive.            | Overview sticky note at workflow start                                                                             |
| Google Drive Trigger watches a specific folder for new uploads, enabling automated intake of claim forms.                      | Sticky note near Google Drive Trigger node                                                                         |
| Download file node ensures actual document binary data is available for AI processing.                                          | Sticky note near Download file node                                                                                 |
| VLM Run’s `healthcare.claims-processing` domain extracts detailed structured medical claim data from complex forms.            | Sticky note near VLM Run node                                                                                       |
| Append row in sheet node continuously updates a Google Sheet, creating a structured database for claims tracking and reporting.| Sticky note near Append row in sheet node                                                                           |
| Requirements: VLM Run API access, Google Drive + Sheets OAuth2 credentials, and an n8n server with activated workflow.          | General prerequisites from overview sticky note                                                                    |
| For more information on VLM Run and healthcare claims processing, consult VLM Run documentation and Google Sheets API guides. | External documentation resources (not included here)                                                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.