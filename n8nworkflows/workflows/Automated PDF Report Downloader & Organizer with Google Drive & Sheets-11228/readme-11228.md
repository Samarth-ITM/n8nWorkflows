Automated PDF Report Downloader & Organizer with Google Drive & Sheets

https://n8nworkflows.xyz/workflows/automated-pdf-report-downloader---organizer-with-google-drive---sheets-11228


# Automated PDF Report Downloader & Organizer with Google Drive & Sheets

### 1. Workflow Overview

This workflow, named **"PDF Research Report Collector & Processor"**, is designed to automatically download research PDF reports from URLs listed in a Google Sheet, upload these PDFs to Google Drive for organized storage, and maintain a metadata log of these files in another Google Sheet. It also tracks processing status and logs errors for failed downloads.

**Target Use Cases:**  
- Automating the collection and archival of PDF reports from distributed URLs.  
- Centralizing PDFs in Google Drive with corresponding metadata in Google Sheets.  
- Maintaining data integrity and traceability with status updates and error logs.  
- Scheduled execution to keep the PDF library current without manual intervention.

**Logical Blocks:**  
- **1.1 Input Reception:** Triggering the workflow manually, on schedule, or via another workflow to read pending PDF URLs from Google Sheets.  
- **1.2 PDF URL Processing & Validation:** Looping through each URL, extracting filename information, validating URLs, and preparing download parameters.  
- **1.3 PDF Downloading:** Downloading PDF content with HTTP requests and handling success or failure.  
- **1.4 Upload & Metadata Extraction:** Uploading downloaded PDFs to Google Drive, extracting and formatting metadata.  
- **1.5 Logging & Status Update:** Saving metadata to Google Sheets, updating URL statuses, and logging any errors encountered.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow either manually, on a 12-hour schedule, or when triggered by another workflow. Its main role is to query the Google Sheet named "PDF URLs" to retrieve pending PDF URLs for processing.

**Nodes Involved:**  
- Manual Trigger  
- Schedule (Every 12 Hours)  
- Called by Another Workflow  
- Read Pending PDF URLs

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node for manual execution.  
  - *Config:* No parameters; used for manual start.  
  - *Connections:* Outputs to "Read Pending PDF URLs".  
  - *Failures:* None expected.

- **Schedule (Every 12 Hours)**  
  - *Type:* Time-based trigger.  
  - *Config:* Runs every 12 hours automatically.  
  - *Connections:* Outputs to "Read Pending PDF URLs".  
  - *Failures:* None expected unless n8n scheduler fails.

- **Called by Another Workflow**  
  - *Type:* Trigger node for external workflow invocation.  
  - *Config:* Passes input data through unchanged.  
  - *Connections:* Outputs to "Read Pending PDF URLs".  
  - *Failures:* Dependency on external workflow availability.

- **Read Pending PDF URLs**  
  - *Type:* Google Sheets Read node.  
  - *Config:* Reads rows from the "PDF URLs" sheet in a specified spreadsheet (credentials required).  
  - *Key Expressions:* None; reads all pending URLs.  
  - *Connections:* Outputs to "Loop Over PDFs".  
  - *Failures:* Google API authentication errors, empty or malformed sheet data.

---

#### 1.2 PDF URL Processing & Validation

**Overview:**  
Processes each PDF URL in batches, extracting filenames, checking validity, and filtering URLs before download.

**Nodes Involved:**  
- Loop Over PDFs  
- Completion Summary  
- Prepare Download Info  
- Is Valid URL?  
- Mark as Invalid

**Node Details:**

- **Loop Over PDFs**  
  - *Type:* SplitInBatches node.  
  - *Config:* Processes items one by one without resetting batch.  
  - *Connections:* Receives from "Read Pending PDF URLs", outputs to "Completion Summary" and "Prepare Download Info".  
  - *Failures:* None expected.

- **Completion Summary**  
  - *Type:* Set node.  
  - *Config:* Sets two variables: number of processed PDFs and timestamp of completion.  
  - *Connections:* Outputs to no further nodes.  
  - *Failures:* None.

- **Prepare Download Info**  
  - *Type:* Code node (JavaScript).  
  - *Config:*  
    - Extracts filename from URL, decoding URI components.  
    - If filename missing or invalid, generates a timestamped default name.  
    - Validates if URL likely points to a PDF by checking for ".pdf", "pdf", or "download" in the URL string.  
    - Marks URL as valid or invalid with reasons.  
  - *Inputs:* Items with PDF URLs.  
  - *Outputs:* Items enriched with `pdfUrl`, `fileName`, `isValid`, and `isPdfUrl`.  
  - *Connections:* Outputs to "Is Valid URL?".  
  - *Failures:* Expression errors if URL is malformed; handled by try-catch.

- **Is Valid URL?**  
  - *Type:* If node.  
  - *Config:* Checks if `isValid` is true.  
  - *Connections:*  
    - True branch → "Download PDF"  
    - False branch → "Mark as Invalid".  
  - *Failures:* None.

- **Mark as Invalid**  
  - *Type:* Set node.  
  - *Config:* Sets status to "Invalid", records skip reason, and includes original URL.  
  - *Connections:* Outputs to "Log Error".  
  - *Failures:* None.

---

#### 1.3 PDF Downloading

**Overview:**  
Downloads the validated PDF via HTTP request and checks for download success.

**Nodes Involved:**  
- Download PDF  
- Download Success?  
- Mark as Failed

**Node Details:**

- **Download PDF**  
  - *Type:* HTTP Request node.  
  - *Config:*  
    - Downloads file from `pdfUrl`.  
    - Sets "User-Agent" and "Accept" headers for PDF content.  
    - Timeout set to 60 seconds.  
    - Response expected as binary file.  
    - On error: continues regular output (does not stop workflow).  
  - *Connections:* Outputs to "Download Success?".  
  - *Failures:* Network errors, 404/403 responses, timeout, invalid URLs.

- **Download Success?**  
  - *Type:* If node.  
  - *Config:* Checks if binary data exists in response.  
  - *Connections:*  
    - True → "Upload to Google Drive"  
    - False → "Mark as Failed".  
  - *Failures:* None.

- **Mark as Failed**  
  - *Type:* Set node.  
  - *Config:* Sets status to "Failed", error message, and related URL/title.  
  - *Connections:* Outputs to "Log Error".  
  - *Failures:* None.

---

#### 1.4 Upload & Metadata Extraction

**Overview:**  
Uploads the downloaded PDF file to Google Drive and extracts key metadata for logging.

**Nodes Involved:**  
- Upload to Google Drive  
- Extract File Metadata

**Node Details:**

- **Upload to Google Drive**  
  - *Type:* Google Drive node.  
  - *Config:*  
    - Uploads file with name taken from "Prepare Download Info".  
    - Uploads to a specified folder ID (e.g., "PDF_Reports").  
    - Credentials required for Google Drive OAuth2.  
    - On error: continue workflow without stopping.  
  - *Connections:* Outputs to "Extract File Metadata".  
  - *Failures:* Google Drive API authentication errors, quota limits, invalid folder ID.

- **Extract File Metadata**  
  - *Type:* Code node (JavaScript).  
  - *Config:*  
    - Extracts metadata such as file ID, name, MIME type, size, and Drive URLs from upload response.  
    - Combines with original PDF URL, title, source, and sets download timestamp and status.  
  - *Connections:* Outputs to "Save to PDF Library".  
  - *Failures:* None expected unless upload response malformed.

---

#### 1.5 Logging & Status Update

**Overview:**  
Logs metadata into the "PDF Library" Google Sheet, updates the processing status in the "PDF URLs" sheet, and logs errors in an error sheet.

**Nodes Involved:**  
- Save to PDF Library  
- Update Source Status  
- Log Error

**Node Details:**

- **Save to PDF Library**  
  - *Type:* Google Sheets AppendOrUpdate node.  
  - *Config:*  
    - Appends or updates metadata entries in "PDF Library" sheet.  
    - Uses same spreadsheet as input.  
  - *Connections:* Outputs to "Update Source Status".  
  - *Failures:* Google API errors, data format mismatches.

- **Update Source Status**  
  - *Type:* Google Sheets Update node.  
  - *Config:*  
    - Updates the status of the processed URL in "PDF URLs" sheet (e.g., "Downloaded", "Failed", "Invalid").  
  - *Connections:* Outputs back to "Loop Over PDFs" for next item batch processing.  
  - *Failures:* Google API errors, concurrency issues.

- **Log Error**  
  - *Type:* Google Sheets AppendOrUpdate node.  
  - *Config:*  
    - Logs error details (URL, status, message) into the "Error Log" sheet.  
  - *Connections:* Outputs to "Update Source Status" to maintain status consistency.  
  - *Failures:* Google API errors.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                         | Input Node(s)                | Output Node(s)              | Sticky Note                                                |
|--------------------------|-------------------------|---------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------|
| Sticky Note - Introduction | Sticky Note             | Documentation introduction             |                             |                             | ## PDF Research Report Collector & Processor ...           |
| Manual Trigger           | Manual Trigger          | Manual workflow start                  |                             | Read Pending PDF URLs       |                                                            |
| Schedule (Every 12 Hours) | Schedule Trigger        | Scheduled workflow start every 12 hrs |                             | Read Pending PDF URLs       |                                                            |
| Called by Another Workflow | Execute Workflow Trigger | External workflow trigger              |                             | Read Pending PDF URLs       |                                                            |
| Read Pending PDF URLs    | Google Sheets           | Read PDF URLs to process               | Manual Trigger, Schedule, Called by Another Workflow | Loop Over PDFs             |                                                            |
| Loop Over PDFs           | SplitInBatches          | Process PDFs one by one                | Read Pending PDF URLs        | Completion Summary, Prepare Download Info |                                                            |
| Completion Summary       | Set                     | Summarize batch processing             | Loop Over PDFs               |                             |                                                            |
| Prepare Download Info    | Code                    | Extract filename and validate URLs     | Loop Over PDFs               | Is Valid URL?               |                                                            |
| Is Valid URL?            | If                      | Validate URL presence                   | Prepare Download Info        | Download PDF, Mark as Invalid | ## Validate URL and Downloaded PDF                          |
| Mark as Invalid          | Set                     | Mark URL invalid and set skip reason   | Is Valid URL? (false branch) | Log Error                   |                                                            |
| Download PDF             | HTTP Request            | Download PDF file                      | Is Valid URL? (true branch)  | Download Success?           | ## Validate the Downloaded PDF                              |
| Download Success?        | If                      | Check if PDF downloaded successfully   | Download PDF                | Upload to Google Drive, Mark as Failed |                                                            |
| Mark as Failed           | Set                     | Mark failed downloads and set error   | Download Success? (false branch) | Log Error                   |                                                            |
| Upload to Google Drive   | Google Drive            | Upload PDF to Drive                    | Download Success? (true branch) | Extract File Metadata       | ## Save PDF to Drive and Log in Google Sheet                |
| Extract File Metadata    | Code                    | Extract metadata from Drive upload     | Upload to Google Drive       | Save to PDF Library         |                                                            |
| Save to PDF Library      | Google Sheets           | Save metadata to library sheet         | Extract File Metadata        | Update Source Status        |                                                            |
| Update Source Status     | Google Sheets           | Update PDF URL processing status       | Save to PDF Library, Log Error | Loop Over PDFs             |                                                            |
| Log Error                | Google Sheets           | Log errors with download or validation | Mark as Invalid, Mark as Failed | Update Source Status        |                                                            |
| Sticky Note              | Sticky Note             | Visual note near upload & logging nodes |                             |                             | ## Save PDF to Drive and Log in Google Sheet                |
| Sticky Note1             | Sticky Note             | Visual note near download validation  |                             |                             | ## Validate the Downloaded PDF                              |
| Sticky Note2             | Sticky Note             | Visual note near URL validation        |                             |                             | ## Validate URL and Downloaded PDF                          |
| Sticky Note3             | Sticky Note             | Visual note near reading URLs          |                             |                             | ## Read PDF URLs                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node for manual start.  
   - Add a **Schedule Trigger** node configured to run every 12 hours.  
   - Add an **Execute Workflow Trigger** node to allow external invocation.

2. **Add Google Sheets Node to Read Pending URLs**  
   - Create a **Google Sheets** node.  
   - Set operation to "Read Rows" from the sheet named "PDF URLs".  
   - Connect all three triggers' outputs to this node.  
   - Use OAuth2 credentials for Google Sheets.  
   - Specify the spreadsheet ID where URLs are stored.

3. **Add SplitInBatches Node to Process URLs One by One**  
   - Create a **SplitInBatches** node.  
   - Connect output of "Read Pending PDF URLs" node here.  
   - Configure batch size to 1 (default).  
   - No reset on batch.

4. **Add a Set Node for Completion Summary**  
   - Create a **Set** node.  
   - Define two fields:  
     - `pdfsProcessed`: Number of items processed (`{{$items().length}}`).  
     - `completedAt`: Timestamp using expression (`{{$now.toISO()}}`).  
   - Connect this node as the first output of "Loop Over PDFs".

5. **Add Code Node to Prepare Download Info**  
   - Create a **Code** node with JavaScript.  
   - Copy logic: extract filename from URL, decode URI, generate fallback filename if needed, check if URL likely PDF.  
   - Output fields: `pdfUrl`, `fileName`, `isValid`, `isPdfUrl`.  
   - Connect second output of "Loop Over PDFs" to this node.

6. **Add If Node to Validate URL**  
   - Create an **If** node checking if `isValid` field equals `true`.  
   - True branch connects to "Download PDF".  
   - False branch connects to "Mark as Invalid".

7. **Add Set Node to Mark Invalid URLs**  
   - Create a **Set** node.  
   - Assign `status` = "Invalid", `errorMessage` = skip reason from previous node, `pdfUrl` from input.  
   - Connect false branch of "Is Valid URL?" here.

8. **Add HTTP Request Node to Download PDF**  
   - Create HTTP Request node.  
   - Set URL to `{{$json.pdfUrl}}`.  
   - Set headers:  
     - User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36  
     - Accept: application/pdf,application/octet-stream,*/*  
   - Set response to "File" type (binary).  
   - Timeout: 60,000 ms.  
   - On error: continue workflow.  
   - Connect true branch of "Is Valid URL?" to this node.

9. **Add If Node to Check Download Success**  
   - Create an **If** node checking if binary data exists (`Object.keys($binary).length > 0`).  
   - True branch connects to "Upload to Google Drive".  
   - False branch connects to "Mark as Failed".

10. **Add Set Node to Mark Failed Downloads**  
    - Create a **Set** node.  
    - Assign `status` = "Failed", `errorMessage` = "Download failed - file may not exist or access denied", `pdfUrl` and `title` from "Prepare Download Info".  
    - Connect false branch of "Download Success?" to this node.

11. **Add Google Drive Node to Upload PDFs**  
    - Create Google Drive node.  
    - Set operation to "Upload File".  
    - File Name: from `fileName` field in "Prepare Download Info".  
    - Folder ID: your designated folder (e.g., "PDF_Reports").  
    - Use Google Drive OAuth2 credentials.  
    - On error: continue workflow.  
    - Connect true branch of "Download Success?" to this node.

12. **Add Code Node to Extract File Metadata**  
    - Create a **Code** node.  
    - Extract file metadata from Google Drive response: file ID, name, MIME type, size, webViewLink, webContentLink.  
    - Add original PDF URL, title, source, download timestamp, and status "Downloaded".  
    - Connect output of Google Drive node to this node.

13. **Add Google Sheets Node to Save Metadata**  
    - Create Google Sheets node with operation "Append or Update".  
    - Target sheet: "PDF Library".  
    - Use same spreadsheet ID and OAuth2 credentials.  
    - Connect output of metadata extraction node here.

14. **Add Google Sheets Node to Update Source Status**  
    - Create Google Sheets node with operation "Update".  
    - Target sheet: "PDF URLs".  
    - Update status field per PDF processed (e.g., "Downloaded", "Failed", "Invalid").  
    - Connect output of "Save to PDF Library" and "Log Error" nodes here.

15. **Add Google Sheets Node to Log Errors**  
    - Create Google Sheets node with operation "Append or Update".  
    - Target sheet: "Error Log".  
    - Connect outputs of "Mark as Invalid" and "Mark as Failed" nodes to this node.

16. **Connect Final Loop Back to Continue Processing**  
    - Connect output of "Update Source Status" back to "Loop Over PDFs" node to continue processing remaining PDFs.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow reads PDF URLs from a Google Sheet column named "PDF_URL" in sheet "PDF URLs".                        | Setup instruction in sticky note introduction.                                                                   |
| Downloads PDFs with custom User-Agent and Accept headers to mimic browser behavior for better server acceptance. | HTTP Request node configuration.                                                                                  |
| Uses timestamped default filenames for URLs that do not contain valid PDF filenames.                            | Code node "Prepare Download Info".                                                                                |
| Google Drive uploads are organized into a specified folder ID, ensure folder exists and folder ID is correct.  | Google Drive node settings.                                                                                        |
| Metadata logged includes Drive URLs for easy access and audit trail.                                           | Code node "Extract File Metadata".                                                                                 |
| Errors and invalid URLs are logged in a separate Google Sheet "Error Log" for review and troubleshooting.      | Google Sheets node "Log Error".                                                                                    |
| Workflow can be triggered manually, scheduled, or called by other workflows for flexibility.                    | Trigger nodes at start.                                                                                            |
| Recommended to run the workflow every 12 hours to keep PDF library updated automatically.                       | Schedule trigger configuration.                                                                                   |
| Workflow uses OAuth2 credentials for Google Sheets and Google Drive; ensure credentials have necessary scopes. | Credential setup requirement.                                                                                      |
| This workflow gracefully handles failures by marking status and logging errors rather than terminating.        | Error handling in HTTP Request and Google Drive nodes with "continue on error" enabled.                           |

---

**Disclaimer:** The provided content is derived solely from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected elements. All data handled is lawful and publicly accessible.