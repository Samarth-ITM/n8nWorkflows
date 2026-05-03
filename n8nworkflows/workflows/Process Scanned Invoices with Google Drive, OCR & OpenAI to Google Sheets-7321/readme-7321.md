Process Scanned Invoices with Google Drive, OCR & OpenAI to Google Sheets

https://n8nworkflows.xyz/workflows/process-scanned-invoices-with-google-drive--ocr---openai-to-google-sheets-7321


# Process Scanned Invoices with Google Drive, OCR & OpenAI to Google Sheets

### 1. Workflow Overview

This workflow automates the processing of scanned invoices stored in a designated Google Drive folder. It detects new invoice files, downloads and processes them using OCR (Optical Character Recognition) and AI-based information extraction, then logs the structured invoice data into a Google Sheets document for record-keeping and further analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Triggering**: Detect new scanned invoice files in Google Drive and initiate processing.
- **1.2 File Download & Preparation**: Download files from Google Drive, batch process them, and prepare file data for OCR or direct text extraction.
- **1.3 Mime Type Handling & OCR Routing**: Differentiate files by type (image or PDF), and route for appropriate text extraction (OCR or direct PDF text extraction).
- **1.4 OCR & Text Extraction**: Use OCR.Space API for images or poorly extracted PDFs to obtain text from files.
- **1.5 AI-Based Information Extraction**: Use OpenAI GPT-4 via LangChain nodes to extract structured invoice information.
- **1.6 Data Logging and Notification**: Append extracted invoice data to Google Sheets and send email notification with a summary table.
- **1.7 Manual Trigger for Initial Folder Scan**: Allows manual initialization of the workflow to process existing invoices.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Triggering

- **Overview:** This block monitors the Google Drive folder for new scanned invoice files and triggers the workflow automatically. It also provides a manual trigger for initial folder scan.
- **Nodes Involved:**  
  - When clicking 'Execute workflow' (Manual Trigger)  
  - Check for new invoices (Google Drive Trigger)  
  - Fetch 50 docs (Google Drive)  

##### Node Details

- **When clicking 'Execute workflow'**  
  - Type: Manual Trigger  
  - Role: Allows manual workflow start, typically to initialize processing for existing files.  
  - Inputs: None  
  - Outputs: To "Fetch 50 docs" node  
  - Failure cases: None specific; manual trigger depends on user.  

- **Check for new invoices**  
  - Type: Google Drive Trigger  
  - Role: Watches a specified Google Drive folder (ID: GOOGLE_DRIVE_FOLDER_ID_001) for newly created files every minute.  
  - Configuration: Trigger on fileCreated event, folder specified by ID, using Google Drive OAuth2 credentials.  
  - Inputs: None  
  - Outputs: Triggers "Download file" node with new file info.  
  - Failure cases: Auth errors with Google Drive OAuth2, API rate limits, folder access issues.  

- **Fetch 50 docs**  
  - Type: Google Drive (List Files)  
  - Role: Lists up to 50 files in the targeted Google Drive folder to process existing invoices on manual execution.  
  - Inputs: From manual trigger  
  - Outputs: To "Download file" node for each file  
  - Failure cases: Same as above for Drive API, quota limits.

---

#### 1.2 File Download & Preparation

- **Overview:** Downloads each detected file from Google Drive and prepares it for processing by splitting into batches and setting metadata.
- **Nodes Involved:**  
  - Download file (Google Drive)  
  - Loop Over Items (Split in Batches)  
  - Set file identity (Set)  
  - Save base64 (Extract From File)  

##### Node Details

- **Download file**  
  - Type: Google Drive (Download)  
  - Role: Downloads the actual binary content of each invoice file by fileId.  
  - Inputs: File metadata from Drive Trigger or Fetch nodes  
  - Outputs: To "Loop Over Items"  
  - Failure cases: Download failures, network errors, invalid file IDs.  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes files one by one to manage resource usage and orderly flow.  
  - Inputs: Binary files from Download node  
  - Outputs: Parallel to "Prepare a table" and "Set file identity" nodes  
  - Failure cases: Logic errors in batch splitting, empty batches.  

- **Set file identity**  
  - Type: Set  
  - Role: Stores file metadata fields `id` and `name` for use downstream (e.g., linking back to Google Drive).  
  - Inputs: Single file's JSON metadata  
  - Outputs: To "Save base64" node  
  - Failure cases: Missing metadata fields, expression evaluation errors.  

- **Save base64**  
  - Type: Extract From File  
  - Role: Converts binary file data to base64 format and retains original binary data.  
  - Inputs: From "Set file identity"  
  - Outputs: To "Switch Mime Type"  
  - Failure cases: Binary data corruption, conversion failures.

---

#### 1.3 Mime Type Handling & OCR Routing

- **Overview:** Determines the file MIME type to decide processing path: image files go to OCR, PDFs to direct text extraction or fallback OCR if needed.  
- **Nodes Involved:**  
  - Switch Mime Type (Switch)  
  - Extract from File (Extract From File)  
  - Identify CamScanner (If)  
  - Re-set base64 (Set)  

##### Node Details

- **Switch Mime Type**  
  - Type: Switch  
  - Role: Routes files based on MIME type from binary metadata: images (starts with "image/"), PDFs ("application/pdf"), or others.  
  - Inputs: base64 data from "Save base64"  
  - Outputs:  
    - image → "Analyze Image" (OCR)  
    - pdf → "Extract from File" (direct text extraction)  
    - extra/fallback → "Loop Over Items" (skip or ignore)  
  - Failure cases: MIME type missing or incorrect, fallback output undefined.  

- **Extract from File**  
  - Type: Extract From File (PDF Text Extraction)  
  - Role: Attempts to extract text directly from PDF files without OCR.  
  - Inputs: PDF binary from Switch node  
  - Outputs: To "Identify CamScanner"  
  - Failure cases: PDF corrupted, encrypted, or text not embedded (leading to fallback).  

- **Identify CamScanner**  
  - Type: If  
  - Role: Checks if extracted PDF text or metadata indicate the file was created by CamScanner app, which often produces non-text PDFs.  
  - Condition: Matches if author metadata or text starts with "CamScanner"  
  - Inputs: Text extracted from PDF  
  - Outputs:  
    - True → "Re-set base64" (fallback for OCR)  
    - False → "Information Extractor" (proceed with AI extraction)  
  - Failure cases: Missing metadata, false negatives.  

- **Re-set base64**  
  - Type: Set  
  - Role: Prepares base64 data again to send to OCR for CamScanner PDFs.  
  - Inputs: From Identify CamScanner true path  
  - Outputs: To "Convert to File" for OCR preparation  
  - Failure cases: Data missing, expression errors.

---

#### 1.4 OCR & Text Extraction

- **Overview:** Uses OCR.Space API to extract text from images or difficult PDF scans, then formats extracted text for AI processing.  
- **Nodes Involved:**  
  - Convert to File (Convert To File)  
  - Analyze Image (HTTP Request)  
  - Edit Fields (Set)  

##### Node Details

- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts base64 string back to binary file for sending to OCR API.  
  - Inputs: base64 data from "Re-set base64"  
  - Outputs: To "Analyze Image"  
  - Failure cases: Conversion errors, data corruption.  

- **Analyze Image**  
  - Type: HTTP Request  
  - Role: Sends image or PDF file to OCR.Space API for text recognition using OCR Engine 2 with orientation detection enabled.  
  - Configuration: POST multipart/form-data, includes binary file as "file" parameter  
  - Auth: HTTP Header Auth with OCR API key  
  - Inputs: Binary file from Convert to File  
  - Outputs: JSON with OCR results to "Edit Fields"  
  - Failure cases: API rate limits, HTTP errors, invalid API key, network timeouts.  

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts recognized text (`ParsedResults[0].ParsedText`) from OCR JSON response into a simpler `text` field.  
  - Inputs: JSON from OCR response  
  - Outputs: To "Information Extractor"  
  - Failure cases: Missing or malformed OCR response, parsing errors.

---

#### 1.5 AI-Based Information Extraction

- **Overview:** Uses an OpenAI GPT-4 based LangChain node to extract structured invoice fields from text: company name, total amount, currency, date, and invoice number.  
- **Nodes Involved:**  
  - OpenAI Chat Model (LangChain LM Chat OpenAI)  
  - Information Extractor (LangChain Information Extractor)  

##### Node Details

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides GPT-4.1-mini model to assist with information extraction tasks.  
  - Credentials: OpenAI API credentials required  
  - Inputs: Connected via AI language model channel to "Information Extractor"  
  - Outputs: Language model responses back to "Information Extractor"  
  - Failure cases: API quota limits, auth errors, model unavailability.  

- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Extracts specified attributes from text input using AI model: company name, total (number), currency, date, invoice number.  
  - Inputs: Text field from "Edit Fields" or directly from "Identify CamScanner" false path  
  - Outputs: Structured JSON with extracted fields to "Append row in sheet"  
  - Failure cases: Inaccurate or incomplete extraction, empty text input.

---

#### 1.6 Data Logging and Notification

- **Overview:** Appends the extracted invoice data into a predefined Google Sheets document and sends an email notification with a summary table of processed invoices.  
- **Nodes Involved:**  
  - Append row in sheet (Google Sheets)  
  - Prepare a table (HTML Table)  
  - Mailgun (Email)  
  - Loop Over Items (Split in Batches) [used again]  

##### Node Details

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Inserts a new row with invoice data (id, date, company name, total, currency, invoice number, scan link, scan name) into a specified Google Sheets document (ID: GOOGLE_SHEETS_DOCUMENT_ID_001).  
  - Inputs: Structured output from "Information Extractor" and file metadata  
  - Credentials: Google Sheets OAuth2  
  - Failure cases: Sheet access errors, quota exceeded, incorrect schema mapping.  

- **Prepare a table**  
  - Type: HTML  
  - Role: Converts batch of invoice data into an HTML table for email content.  
  - Inputs: Batched invoice data from "Loop Over Items"  
  - Outputs: To "Mailgun"  
  - Failure cases: Formatting errors, empty input.  

- **Mailgun**  
  - Type: Mailgun  
  - Role: Sends email notification with subject "New scanned invoices" and HTML table body to user@example.com.  
  - Credentials: Mailgun API key  
  - Inputs: HTML content from "Prepare a table"  
  - Failure cases: Email sending failures, auth errors, rate limits.

---

#### 1.7 Manual Trigger for Initial Folder Scan

- **Overview:** Manual trigger node allows the user to start the workflow on demand to scan and process all existing invoices in the Google Drive folder, not just new ones.  
- **Nodes Involved:**  
  - When clicking 'Execute workflow' (Manual Trigger)  
  - Fetch 50 docs (Google Drive)  

---

### 3. Summary Table

| Node Name              | Node Type                                  | Functional Role                       | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                 |
|------------------------|--------------------------------------------|------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger                         | Manual start for initial scan       |                             | Fetch 50 docs               | # Start here - manual trigger to initialize Google Drive folder scan                                                        |
| Check for new invoices  | Google Drive Trigger                       | Automatic trigger on new invoice files |                             | Download file               | # Start here - scheduled trigger to identify new invoices                                                                    |
| Fetch 50 docs          | Google Drive                               | Lists existing files in folder      | When clicking 'Execute workflow' | Download file           |                                                                                                                             |
| Download file          | Google Drive                               | Downloads invoice files             | Check for new invoices, Fetch 50 docs | Loop Over Items         |                                                                                                                             |
| Loop Over Items        | Split in Batches                           | Batch processing of files           | Download file               | Prepare a table, Set file identity |                                                                                                                             |
| Set file identity      | Set                                        | Sets metadata fields id and name   | Loop Over Items             | Save base64                | ### Setting some handy values for later down the road                                                                        |
| Save base64            | Extract From File                          | Converts binary to base64           | Set file identity           | Switch Mime Type           |                                                                                                                             |
| Switch Mime Type       | Switch                                    | Routes files by MIME type           | Save base64                 | Analyze Image, Extract from File, Loop Over Items |                                                                                                                             |
| Extract from File      | Extract From File                          | Attempts PDF text extraction        | Switch Mime Type            | Identify CamScanner        | ### Reroute to OCR if needed - fallback for CamScanner PDFs                                                                  |
| Identify CamScanner    | If                                        | Checks if PDF is from CamScanner app | Extract from File           | Re-set base64 (true), Information Extractor (false) |                                                                                                                             |
| Re-set base64          | Set                                        | Prepares base64 data for OCR        | Identify CamScanner         | Convert to File            |                                                                                                                             |
| Convert to File        | Convert To File                           | Converts base64 string back to binary | Re-set base64               | Analyze Image              |                                                                                                                             |
| Analyze Image          | HTTP Request                              | Sends file to OCR.Space API         | Convert to File             | Edit Fields                | OCR.Space API registration: https://ocr.space/OCRAPI                                                                         |
| Edit Fields            | Set                                        | Extracts OCR text from API response | Analyze Image               | Information Extractor      |                                                                                                                             |
| OpenAI Chat Model      | LangChain LM Chat OpenAI                   | Provides GPT-4 language model       | - (AI language model)       | Information Extractor      |                                                                                                                             |
| Information Extractor  | LangChain Information Extractor            | Extracts structured invoice data    | Edit Fields, Identify CamScanner (false) | Append row in sheet      |                                                                                                                             |
| Append row in sheet    | Google Sheets                             | Logs invoice data to Google Sheets  | Information Extractor       | Loop Over Items            |                                                                                                                             |
| Prepare a table        | HTML                                      | Creates HTML table for email        | Loop Over Items             | Mailgun                   |                                                                                                                             |
| Mailgun                | Mailgun                                   | Sends email notification            | Prepare a table             |                            |                                                                                                                             |
| Sticky Note            | Sticky Note                               | Explanation about OCR fallback logic | -                         |                            | ### Reroute to OCR if needed - Try to extract text directly from PDF. CamScanner fallback. OCR.Space API registration link. |
| Sticky Note1           | Sticky Note                               | Comment on setting handy values     | -                         |                            | ### Setting some handy values for later down the road                                                                        |
| Sticky Note2           | Sticky Note                               | Notes on OCR approach for images/PDFs | -                       |                            | ### OCR for images and badly extracted PDFs                                                                                  |
| Sticky Note3           | Sticky Note                               | Workflow start instructions         | -                         |                            | # Start here - manual and scheduled trigger explanation                                                                      |
| Sticky Note4           | Sticky Note                               | Workflow end point comment          | -                         |                            | # End here                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node named "When clicking 'Execute workflow'"**  
   - Purpose: manual start point.

2. **Create a Google Drive Trigger node named "Check for new invoices"**  
   - Trigger on "fileCreated" event in a specific folder with ID `GOOGLE_DRIVE_FOLDER_ID_001`.  
   - Set poll interval to every minute.  
   - Connect credentials for Google Drive OAuth2.

3. **Create a Google Drive node named "Fetch 50 docs"**  
   - Operation: List files in folder with ID `GOOGLE_DRIVE_FOLDER_ID_001`.  
   - Connect to "When clicking 'Execute workflow'".

4. **Create a Google Drive node named "Download file"**  
   - Operation: Download file by `fileId`.  
   - Credentials: Google Drive OAuth2.  
   - Connect inputs from both "Check for new invoices" and "Fetch 50 docs".

5. **Create a SplitInBatches node named "Loop Over Items"**  
   - No specific batch size configured (default).  
   - Connect input from "Download file".

6. **Create a Set node named "Set file identity"**  
   - Assign the JSON fields:  
     - `id` = `{{$json["id"]}}`  
     - `name` = `{{$json["name"]}}`  
   - Connect output from "Loop Over Items".

7. **Create an Extract From File node named "Save base64"**  
   - Operation: "binaryToProperty" (convert binary file to base64 string in property).  
   - Option to keep both binary and base64.  
   - Connect output from "Set file identity".

8. **Create a Switch node named "Switch Mime Type"**  
   - Conditions based on `$binary.data.mimeType`:  
     - If starts with "image/" route to "Analyze Image"  
     - If equals "application/pdf" route to "Extract from File"  
     - Else fallback to "Loop Over Items"  
   - Connect output from "Save base64".

9. **Create an Extract From File node named "Extract from File"**  
   - Operation: "pdf" text extraction.  
   - Connect from "Switch Mime Type" (pdf output).

10. **Create an If node named "Identify CamScanner"**  
    - Condition: Check if `info.Author` equals "CamScanner" or text starts with "CamScanner".  
    - Connect from "Extract from File".

11. **Create a Set node named "Re-set base64"**  
    - Assign `data` property to `{{$node["Save base64"].item.json.data}}` (reuse base64 data).  
    - Connect from "Identify CamScanner" true branch.

12. **Create a Convert To File node named "Convert to File"**  
    - Operation: convert base64 string `data` back to binary, filename from `Save base64` node.  
    - Connect from "Re-set base64".

13. **Create an HTTP Request node named "Analyze Image"**  
    - Method: POST to `https://api.ocr.space/parse/image`.  
    - Content-Type: multipart/form-data, with binary file as "file" parameter.  
    - Include body parameters: `OCREngine=2`, `detectOrientation=true`.  
    - Authentication: HTTP Header with OCR API key credential.  
    - Retry on failure up to 2 times.  
    - Connect from "Convert to File" (image input) and from "Switch Mime Type" (image output).

14. **Create a Set node named "Edit Fields"**  
    - Assign `text` property to `{{$json["ParsedResults"][0]["ParsedText"]}}`.  
    - Connect from "Analyze Image".

15. **Create a LangChain LM Chat OpenAI node named "OpenAI Chat Model"**  
    - Model: GPT-4.1-mini.  
    - Credentials: OpenAI API key.  
    - Connect AI model channel to "Information Extractor".

16. **Create a LangChain Information Extractor node named "Information Extractor"**  
    - Input text from `text` property.  
    - Define attributes to extract:  
      - company name (string)  
      - total (number)  
      - currency (string)  
      - date (string)  
      - invoice number (string)  
    - Connect from "Edit Fields" and from "Identify CamScanner" false branch (direct extraction).  
    - Connect AI language model input from "OpenAI Chat Model".

17. **Create a Google Sheets node named "Append row in sheet"**  
    - Operation: Append row.  
    - Document ID: `GOOGLE_SHEETS_DOCUMENT_ID_001`.  
    - Sheet Name: `gid=0` (Sheet1).  
    - Map columns: id, Scan (Google Drive link), date, total, currency, scan name, company name, invoice number.  
    - Credentials: Google Sheets OAuth2.  
    - Connect from "Information Extractor".

18. **Connect "Append row in sheet" output back to "Loop Over Items"**  
    - For processing next batch item.

19. **Create an HTML node named "Prepare a table"**  
    - Operation: Convert to HTML table with capitalization and custom styling enabled.  
    - Connect from "Loop Over Items" (parallel output).

20. **Create a Mailgun node named "Mailgun"**  
    - Email subject: "New scanned invoices".  
    - To: user@example.com, From: n8n@example.com.  
    - HTML content from "Prepare a table".  
    - Credentials: Mailgun API key.  
    - Connect from "Prepare a table".

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| OCR.Space API registration: https://ocr.space/OCRAPI                                                           | Used for OCR processing of images and PDFs    |
| CamScanner PDF files often lack readable embedded text, necessitating OCR fallback approach                      | Explained in Sticky Note near Identify CamScanner node |
| Manual trigger allows initializing the workflow on existing files before scheduled triggers take over           | Workflow start instructions                    |
| The workflow uses OpenAI GPT-4.1-mini model via LangChain for structured information extraction                   | OpenAI API credentials needed                   |
| Email notifications are sent via Mailgun with a summary HTML table of processed invoices                         | Mailgun credential and usage                    |

---

This documentation should provide complete clarity on the workflow’s structure, logic, configuration, and execution, enabling both human operators and automation agents to understand, reproduce, and extend it confidently.