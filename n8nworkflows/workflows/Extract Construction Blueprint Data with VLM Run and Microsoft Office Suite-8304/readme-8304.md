Extract Construction Blueprint Data with VLM Run and Microsoft Office Suite

https://n8nworkflows.xyz/workflows/extract-construction-blueprint-data-with-vlm-run-and-microsoft-office-suite-8304


# Extract Construction Blueprint Data with VLM Run and Microsoft Office Suite

### 1. Workflow Overview

This workflow automates the extraction and structured processing of construction blueprint documents uploaded to a OneDrive folder. Its primary goal is to convert complex construction blueprints into machine-readable structured data using the VLM Run AI service, then append this data into an Excel spreadsheet for tracking, compliance, or reporting purposes.  

The workflow is logically divided into the following blocks:  

- **1.1 Input Reception & File Acquisition**  
  Watches a specified OneDrive folder for new file uploads and downloads the files automatically for processing.  

- **1.2 AI Processing with VLM Run**  
  Sends the downloaded blueprint documents to the VLM Run API configured for the `construction.blueprint` domain to extract structured JSON data including project, drawing, architect, and compliance details.  

- **1.3 Data Append to Excel Sheet**  
  Takes the extracted structured data and appends it as a new row into a dedicated Excel worksheet on OneDrive, storing key metadata fields for ongoing blueprint management.  

This design targets use cases in construction project management, architectural plan review, regulatory submissions, and automated compliance documentation workflows. It requires integration credentials for VLM Run API and Microsoft OneDrive/Excel OAuth2.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Acquisition

**Overview:**  
This block monitors a designated OneDrive folder, automatically triggering when a new file is uploaded. It then downloads the file to be processed downstream.  

**Nodes Involved:**  
- Microsoft OneDrive Trigger  
- Download a file  

**Node Details:**  

- **Microsoft OneDrive Trigger**  
  - *Type:* Trigger node specialized for OneDrive  
  - *Role:* Watches a specific OneDrive folder for new files uploaded in near real-time (every minute polling).  
  - *Configuration:*  
    - FolderId set to a specific OneDrive folder ID where blueprint files are uploaded.  
    - Poll interval every minute.  
    - Watch folder enabled for continuous monitoring.  
  - *Credentials:* Microsoft OneDrive OAuth2 account.  
  - *Input:* None (trigger)  
  - *Output:* Emits metadata about the uploaded file including file ID.  
  - *Failure modes:* Connectivity issues, OAuth token expiration, folder ID misconfiguration.  
  - *Version:* Standard n8n node, no special version required.  

- **Download a file**  
  - *Type:* Microsoft OneDrive node (file operation)  
  - *Role:* Downloads the actual file binary from OneDrive using the file ID from the trigger.  
  - *Configuration:*  
    - Operation: Download  
    - File ID dynamically set to the `$json.id` from the trigger output.  
  - *Credentials:* Same Microsoft OneDrive OAuth2 account.  
  - *Input:* Receives file metadata from OneDrive Trigger.  
  - *Output:* Binary file data of the blueprint document.  
  - *Failure modes:* Download failures, file not found, access denied, token expiration.  
  - *Version:* Standard n8n node.  

---

#### 2.2 AI Processing with VLM Run  

**Overview:**  
This block sends the downloaded file to the VLM Run AI platform for parsing under the `construction.blueprint` domain. The AI extracts structured JSON data including project info, drawing metadata, compliance marks, and annotations.  

**Nodes Involved:**  
- VLM Run Parsing  

**Node Details:**  

- **VLM Run Parsing**  
  - *Type:* VLM Run API integration node (third-party AI document processor)  
  - *Role:* Sends the binary file to VLM Run for AI-powered blueprint data extraction.  
  - *Configuration:*  
    - Domain set as `construction.blueprint` to specify blueprint-specific extraction model.  
  - *Credentials:* VLM Run API key configured with the correct account.  
  - *Input:* Receives binary file from the Download a file node.  
  - *Output:* JSON object with structured fields such as project details, document metadata, drawing titles and numbers, revision history, annotations, compliance info, and title block data.  
  - *Failure modes:* API authentication failure, network errors, unsupported file format rejection, AI parsing errors, timeouts.  
  - *Version:* Uses version 1 of the VLM Run node.  

---

#### 2.3 Data Append to Excel Sheet  

**Overview:**  
This block appends the extracted blueprint data as a new row into a specified Excel worksheet stored on OneDrive. It organizes various metadata fields into columns for easy tracking and reporting.  

**Nodes Involved:**  
- Append data to sheet (Microsoft Excel node)  

**Node Details:**  

- **Append data to sheet**  
  - *Type:* Microsoft Excel node (worksheet operation)  
  - *Role:* Appends a row with multiple columns representing structured blueprint data extracted from VLM Run.  
  - *Configuration:*  
    - Operation: Append row  
    - Workbook and worksheet specified by ID and name, pointing to a "Construction Blueprint" Excel file on OneDrive.  
    - Columns include:  
      - Project Details (complex nested object stringified with filtering)  
      - Document Type, Number, Issue Date  
      - Author's Name (complex nested field)  
      - Drawing Title Numbers, Scale Legends  
      - Revision History, Annotations Markups  
      - Job Name, Address, Drawing Number, Revision  
      - Drawn By, Checked By, Scale, Agency Name, Document Title  
      - Other Metadata, Drawing Type, Legal Compliance, Scale Information  
    - Many columns use JavaScript expressions to recursively stringify nested JSON fields into readable strings, skipping empty or null values.  
  - *Credentials:* Microsoft Excel OAuth2 account configured separately from OneDrive credentials but using the same OAuth2 mechanism.  
  - *Input:* Receives the structured JSON output from the VLM Run Parsing node.  
  - *Output:* Confirmation of row append operation.  
  - *Failure modes:* OAuth token expiration, Excel file/worksheet not found, column mismatch errors, rate limiting by Microsoft API.  
  - *Version:* Uses version 2.1 of the Excel node for better compatibility with append operations.  

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                           |
|-------------------------|--------------------------------------|----------------------------------------|------------------------|-------------------------|-----------------------------------------------------------------------------------------------------|
| Microsoft OneDrive Trigger | n8n-nodes-base.microsoftOneDriveTrigger | Watches OneDrive folder for new files  | None                   | Download a file         | # 📁 Input Processing Monitors & downloads blueprint files from OneDrive. Supported formats: Images, PDFs, scans. |
| Download a file         | n8n-nodes-base.microsoftOneDrive     | Downloads uploaded file from OneDrive  | Microsoft OneDrive Trigger | VLM Run Parsing         | # 📁 Input Processing Monitors & downloads blueprint files from OneDrive. Supported formats: Images, PDFs, scans. |
| VLM Run Parsing          | @vlm-run/n8n-nodes-vlmrun.vlmRun     | Sends file to VLM Run API for parsing  | Download a file         | Append data to sheet    | # VLM Run (Document) Sends the blueprint file to VLM Run under `construction.blueprint`. Extracts detailed structured data. |
| Append data to sheet     | n8n-nodes-base.microsoftExcel         | Appends extracted data as a row in Excel | VLM Run Parsing         | None                    | # Append Row in Excel Sheet Appends structured blueprint metadata to Excel for tracking and compliance. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Microsoft OneDrive Trigger node:**  
   - Type: Microsoft OneDrive Trigger  
   - Configure credentials with valid Microsoft OneDrive OAuth2 account.  
   - Set `FolderId` to the target folder where blueprint files will be uploaded (use folder picker or enter ID).  
   - Enable `Watch Folder` to true.  
   - Set polling interval to every minute for near real-time detection.  

2. **Create Download a file node:**  
   - Type: Microsoft OneDrive (operation: download)  
   - Credentials: Same Microsoft OneDrive OAuth2 account as trigger.  
   - Set `File ID` expression to `={{ $json.id }}` to dynamically get the file ID from the trigger output.  
   - Connect output of Microsoft OneDrive Trigger node to this node input.  

3. **Create VLM Run Parsing node:**  
   - Type: VLM Run node from the `@vlm-run/n8n-nodes-vlmrun` package.  
   - Credentials: Configure VLM Run API credentials with your API key for the account.  
   - Set `Domain` parameter to `"construction.blueprint"` to activate blueprint-specific extraction.  
   - Connect output of Download a file node to this node input.  

4. **Create Append data to sheet node:**  
   - Type: Microsoft Excel node  
   - Credentials: Configure Microsoft Excel OAuth2 credentials (can be the same or separate from OneDrive credentials).  
   - Set operation to `append`.  
   - Select the workbook by its OneDrive ID or browse to find the Excel file named "Construction Blueprint".  
   - Select the worksheet to append rows (e.g., Sheet1).  
   - Define columns with field mappings as follows (each field uses expressions to handle nested objects):  

     | Column Name           | Expression Example (simplified)                                                          |
     |-----------------------|------------------------------------------------------------------------------------------|
     | PROJECT DETAILS       | Recursive stringify of `$json.response.project_details`                                  |
     | DOCUMENT TYPE         | `$json.response.document_metadata.document_type`                                        |
     | DOCUMENT NUMBER       | `$json.response.document_metadata.document_number`                                      |
     | ISSUE DATE            | `$json.response.document_metadata.issue_date`                                           |
     | AUTHOR'S NAME         | Recursive stringify of `$json.response.document_metadata.author`                         |
     | DRAWING TITLE NUMBERS | Recursive stringify of `$json.response.drawings_blueprints.drawing_titles_numbers`      |
     | SCALE LEGENDS         | `$json.response.drawings_blueprints.scale_legends`                                      |
     | REVISION HISTORY      | Recursive stringify of `$json.response.drawings_blueprints.revision_history`             |
     | ANNOTATIONS MARKUPS   | Recursive stringify of `$json.response.drawings_blueprints.annotations_markups`          |
     | JOB NAME              | `$json.response.title_block.job_name`                                                   |
     | ADDRESS               | Recursive stringify of `$json.response.title_block.address`                             |
     | DRAWING NUMBER        | `$json.response.title_block.drawing_number`                                             |
     | REVISION              | `$json.response.title_block.revision`                                                   |
     | DRAWN BY              | `$json.response.title_block.drawn_by`                                                   |
     | CHECKED BY            | `$json.response.title_block.checked_by`                                                 |
     | SCALE                 | `$json.response.title_block.scale`                                                      |
     | AGENCY NAME           | `$json.response.title_block.agency_name`                                                |
     | DOCUMENT TITLE        | `$json.response.title_block.document_title`                                            |
     | OTHER METADATA        | Recursive stringify of `$json.response.title_block.other_metadata`                      |
     | DRAWING TYPE          | `$json.response.drawing_type`                                                           |
     | LEGAL COMPLIANCE      | Recursive stringify of `$json.response.compliance_legal`                                |
     | SCALE INFORMATION     | `$json.response.scale_information`                                                      |

   - Connect output of VLM Run Parsing node to this node input.  

5. **Activate the workflow:**  
   - Ensure all nodes have valid credentials.  
   - Save and activate the workflow.  
   - Test by uploading a supported blueprint file (JPG, PNG, PDF, WebP, scanned receipt) into the monitored OneDrive folder.  
   - Observe the file being downloaded, processed by VLM Run, and the resulting data appended to the Excel file.  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Construction Blueprint Processing with VLM Run: Automatically extracts structured construction blueprint details from uploaded OneDrive documents, saving them into Excel for tracking, compliance, or reporting. | Sticky Note on workflow describing overall purpose and use cases. |
| Supported blueprint input formats include images (JPG, PNG, WEBP), PDFs, mobile camera uploads, and scanned receipts. | Sticky Note describing input processing details. |
| VLM Run domain `construction.blueprint` extracts detailed project data: project name, address, permit ID, drawing elements, architect details, and compliance flags. | Sticky Note describing VLM Run node purpose. |
| Excel sheet columns capture comprehensive blueprint metadata, supporting structured, updated tracking databases. | Sticky Note on Excel append node. |
| Requires active VLM Run API account and Microsoft OAuth2 credentials for OneDrive and Excel. | Workflow metadata and sticky notes. |

---

*Disclaimer:* The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.