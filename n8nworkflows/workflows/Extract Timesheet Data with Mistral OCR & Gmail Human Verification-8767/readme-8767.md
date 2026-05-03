Extract Timesheet Data with Mistral OCR & Gmail Human Verification

https://n8nworkflows.xyz/workflows/extract-timesheet-data-with-mistral-ocr---gmail-human-verification-8767


# Extract Timesheet Data with Mistral OCR & Gmail Human Verification

### 1. Workflow Overview

This workflow automates the extraction of timesheet data stored in Google Drive files, using Mistral OCR for document text extraction, and then sends the extracted data via Gmail for human verification before saving the verified data back to Google Drive. It is designed for organizations needing to process varied timesheet file formats (PDF, DOC, Excel, images), extract structured timesheet information, and involve manual validation to ensure accuracy.

The workflow is logically divided into the following blocks:

- **1.1 File Retrieval from Google Drive:** Search and loop over timesheet files stored in Google Drive.
- **1.2 File Type Routing and Download:** Identify file type, download the file, and route it to appropriate extraction methods.
- **1.3 OCR Extraction via Mistral Cloud:** Upload files for OCR, obtain signed URLs, perform OCR extraction, or convert Excel files to JSON.
- **1.4 AI Data Cleaning and Formatting:** Use a LangChain AI agent with Mistral Cloud Chat Model to clean and convert extracted raw data into structured JSON.
- **1.5 Formatting Output for Human Verification:** Convert the cleaned JSON data into an HTML email body.
- **1.6 Sending Verification Request and Handling Response:** Send the formatted data via Gmail with a custom form for verification and comments.
- **1.7 Saving Verified Data:** Upon approval, save the verified timesheet data back to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 File Retrieval from Google Drive

- **Overview:**  
  This block searches the Google Drive folder for timesheet files and initiates processing for each file found.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Search files and folders (Google Drive)  
  - Loop Over Items2 (SplitInBatches)  
  - Set File ID1 (Set)  
  - Download File (Google Drive)

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually.  
    - Inputs: None  
    - Outputs: Triggers next node to search files.  
    - Potential failures: None, manual start only.

  - **Search files and folders**  
    - Type: Google Drive node  
    - Role: Searches Google Drive root folder for all files.  
    - Config: No folder ID filter, fetches all files with metadata (id, name, webViewLink, mimeType).  
    - Credentials: Google Drive OAuth2  
    - Inputs: Trigger from manual node.  
    - Outputs: List of files found.  
    - Failures: Auth errors, Google API limits.

  - **Loop Over Items2**  
    - Type: SplitInBatches  
    - Role: Iterates over each found file one at a time (batch size default).  
    - Inputs: List of files from previous node.  
    - Outputs: One file per iteration to downstream nodes.  
    - Failures: None typical; ensure batch size suitable.

  - **Set File ID1**  
    - Type: Set  
    - Role: Extracts and sets file metadata fields (file_id, file_type, file_title, file_url) from current item.  
    - Inputs: Single file item from loop.  
    - Outputs: JSON with file metadata for next download.  
    - Failures: Expression failures if input missing fields.

  - **Download File**  
    - Type: Google Drive (download operation)  
    - Role: Downloads the current file by file_id.  
    - Config: Converts Google Docs to plain text during download if applicable.  
    - Credentials: Google Drive OAuth2  
    - Inputs: file_id from Set File ID1.  
    - Outputs: Binary file content for extraction.  
    - Failures: File not found, permission denied, conversion errors.

---

#### 2.2 File Type Routing and Different Extraction Methods

- **Overview:**  
  Determines the file type and routes the file for appropriate processing: PDF/DOC files are sent for Mistral OCR, Excel files are parsed into JSON, and images are extracted differently.

- **Nodes Involved:**  
  - Switch File Type (Switch)  
  - Edit Fields3 (Set)  
  - Read Spreadsheet File (Spreadsheet File)  
  - HTTP Request1 (HTTP Request)

- **Node Details:**  

  - **Switch File Type**  
    - Type: Switch  
    - Role: Routes according to MIME type of file (`file_type` field).  
    - Conditions:  
      - pdf/doc group: pdf, msword, docx, google doc mime types  
      - excel group: excel mime types and google sheets  
      - image group: jpeg, png, tiff, gif, bmp  
    - Outputs: 3 outputs corresponding to these groups  
    - Inputs: file metadata and binary content from Download File  
    - Failures: Missing or unknown MIME types.

  - **Edit Fields3**  
    - Type: Set  
    - Role: Passes data forward for pdf/doc files, clears fields as preparation for upload.  
    - Inputs: PDF/DOC binary data.  
    - Outputs: Prepared data for Mistral Upload.  
    - Failures: None expected.

  - **Read Spreadsheet File**  
    - Type: Spreadsheet File (Excel parser)  
    - Role: Parses Excel or Google Sheets files into JSON rows.  
    - Config: Header row enabled for column keys.  
    - Inputs: Excel binary data.  
    - Outputs: JSON rows of spreadsheet data.  
    - Failures: Malformed or encrypted Excel files.

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: For image files, sends multipart form-data POST to an external universal file-to-text extractor API.  
    - Config: POST to "https://universal-file-to-text-extractor.vercel.app/extract" with mode=single, output JSONL, including images.  
    - Inputs: Image binary data.  
    - Outputs: JSON text extraction results.  
    - Failures: Network, API limits, invalid file formats.

---

#### 2.3 OCR Extraction via Mistral Cloud

- **Overview:**  
  Uploads documents to Mistral cloud for OCR extraction, retrieves signed URLs, and processes OCR results.

- **Nodes Involved:**  
  - Mistral Upload (HTTP Request)  
  - Mistral Signed URL (HTTP Request)  
  - Mistral DOC OCR (HTTP Request)  
  - normalize output, normalize output1, normalize output2 (Set)  

- **Node Details:**  

  - **Mistral Upload**  
    - Type: HTTP Request (POST multipart-form-data)  
    - Role: Upload file to Mistral for OCR purpose.  
    - Config: Sends file binary under "file" parameter, purpose = "ocr".  
    - Credentials: Mistral Cloud API  
    - Outputs: File metadata including file ID.  
    - Failures: Auth errors, file size limits.

  - **Mistral Signed URL**  
    - Type: HTTP Request (GET)  
    - Role: Obtains a signed URL for the uploaded file valid for 1 minute.  
    - Config: GET request to Mistral API with file ID, expiry=1 (minute).  
    - Credentials: Mistral Cloud API  
    - Outputs: JSON containing a temporary accessible URL.  
    - Failures: Expired URL, invalid file ID.

  - **Mistral DOC OCR**  
    - Type: HTTP Request (POST JSON)  
    - Role: Calls Mistral OCR model on the document URL to extract bounding box annotations.  
    - Config: JSON body specifies "mistral-ocr-latest" model, document_url from signed URL, strict json schema for annotations.  
    - Credentials: Mistral Cloud API  
    - Outputs: OCR JSON data with extracted text and annotations.  
    - Failures: API response errors, schema validation errors.

  - **normalize output / normalize output1 / normalize output2**  
    - Type: Set  
    - Role: Extracts relevant text or markdown content from different OCR or extractor JSON output structures to normalize for AI processing.  
    - Inputs: OCR/extractor raw JSON data.  
    - Outputs: Sets a "content" string field with extracted text for AI agent.  
    - Failures: JSON parsing errors, missing expected fields.

---

#### 2.4 AI Data Cleaning and Formatting

- **Overview:**  
  Uses a LangChain AI agent powered by Mistral Cloud Chat Model to parse the extracted raw text into a strictly formatted JSON representing timesheet data.

- **Nodes Involved:**  
  - AI Agent to clean and format extracted data (LangChain Agent)  
  - Mistral Cloud Chat Model2 (AI Language Model)  
  - clean ai output (Code)  

- **Node Details:**  

  - **AI Agent to clean and format extracted data**  
    - Type: LangChain Agent  
    - Role: Processes the extracted text content to produce a precise JSON array of timesheet entries.  
    - Config: Custom prompt instructs to respond ONLY with valid raw JSON, no markdown or wrapping keys, fully extracting all time entries until totals match.  
    - Inputs: Normalized text content from OCR or extractor outputs.  
    - Outputs: Raw JSON string with structured timesheet data.  
    - Failures: AI parsing errors, incomplete JSON output, API limits.

  - **Mistral Cloud Chat Model2**  
    - Type: AI Language Model (Mistral Cloud)  
    - Role: Underlying model providing language understanding for the LangChain agent.  
    - Credentials: Mistral Cloud API  
    - Inputs: Text from normalize output nodes.  
    - Outputs: AI generated JSON string.  
    - Failures: API or auth errors.

  - **clean ai output**  
    - Type: Code (JavaScript)  
    - Role: Parses the AI agent’s JSON string response into n8n items (JSON objects).  
    - Inputs: Raw JSON string from AI Agent node.  
    - Outputs: Array of JSON objects representing timesheet entries.  
    - Failures: JSON parsing errors if AI output is malformed.

---

#### 2.5 Formatting Output for Human Verification

- **Overview:**  
  Converts the validated JSON timesheet data into an HTML formatted email body for review.

- **Nodes Involved:**  
  - json to html (Code)

- **Node Details:**  

  - **json to html**  
    - Type: Code (JavaScript)  
    - Role: Builds an HTML table displaying timesheet summary and daily entries for email.  
    - Inputs: Parsed JSON timesheet data array from clean ai output.  
    - Outputs: JSON with `email_body` string field containing HTML.  
    - Failures: Missing fields in input JSON, script errors.

---

#### 2.6 Sending Verification Request and Handling Response

- **Overview:**  
  Sends the formatted timesheet data via Gmail with a custom form for user verification, and waits for response to decide further processing.

- **Nodes Involved:**  
  - Send a message (Gmail)  
  - Loop Over Items2 (SplitInBatches) [loop restarted after verification]  
  - Upload file (Google Drive)

- **Node Details:**  

  - **Send a message**  
    - Type: Gmail node (sendAndWait operation)  
    - Role: Sends an email to a specified address with the timesheet HTML, including a form field for verification (Yes/No) and comments.  
    - Config:  
      - Recipient email placeholder "addemail" (to be replaced with actual address).  
      - Subject: "timesheet approval required"  
      - Message body: HTML timesheet data.  
      - Custom form fields: original file URL (readonly), accuracy dropdown (yes/no), and comments.  
    - Credentials: Gmail OAuth2  
    - Outputs: Waits for user reply and form submission.  
    - Failures: Email sending errors, OAuth issues, form response timeout.

  - **Loop Over Items2 (second output)**  
    - Role: If the verification is positive, triggers next iteration or upload node.  
    - Outputs: Moves to uploading verified files.

  - **Upload file**  
    - Type: Google Drive  
    - Role: Uploads the verified timesheet data (or possibly the original file) to Google Drive root folder.  
    - Credentials: Google Drive OAuth2  
    - Inputs: Triggered after verification.  
    - Failures: Upload permission errors, network issues.

---

#### 2.7 Excel Data Cleaning Sub-block

- **Overview:**  
  Processes Excel data extracted from spreadsheet files by converting serial date formats to ISO date strings.

- **Nodes Involved:**  
  - Clean excel data format (Code)  
  - Aggregate (Aggregate)

- **Node Details:**  

  - **Clean excel data format**  
    - Type: Code (JavaScript)  
    - Role: Converts Excel serial dates in specific fields to ISO date strings, only when applicable.  
    - Inputs: JSON rows from Read Spreadsheet File.  
    - Outputs: JSON rows with cleaned date fields.  
    - Failures: Invalid date serials, unexpected data types.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates all cleaned JSON row items into a single dataset for further processing.  
    - Inputs: Cleaned JSON rows.  
    - Outputs: Single aggregated JSON object.  
    - Failures: None typical.

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                               | Input Node(s)               | Output Node(s)                              | Sticky Note                                                                                   |
|----------------------------------|--------------------------------|-----------------------------------------------|-----------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                 | Starts the workflow manually                   | None                        | Search files and folders                     |                                                                                              |
| Search files and folders          | Google Drive (fileFolder)      | Search Google Drive for all files              | When clicking ‘Test workflow’ | Loop Over Items2                             | ## GET ALL TIMESHEET FROM GOOGLE DRIVE                                                      |
| Loop Over Items2                  | SplitInBatches                 | Iterate over each file individually            | Search files and folders      | Set File ID1, Send a message (second output) | ## loop over each file                                                                       |
| Set File ID1                     | Set                           | Extract and set file metadata                   | Loop Over Items2             | Download File                                | ## DOWNLOAD TIMESHEETS                                                                      |
| Download File                    | Google Drive (download)        | Download the current file by ID                 | Set File ID1                 | Switch File Type                             |                                                                                              |
| Switch File Type                 | Switch                        | Route file based on MIME type                    | Download File                | Edit Fields3, Read Spreadsheet File, HTTP Request1 |                                                                                              |
| Edit Fields3                    | Set                           | Prepare PDF/DOC files for Mistral upload        | Switch File Type (pdf/doc)   | Mistral Upload                               | ## MISTRAL OCR for extraction of data from pdf or doc                                      |
| Read Spreadsheet File            | Spreadsheet File               | Parse Excel or Sheets file to JSON               | Switch File Type (excel)     | Clean excel data format                       | ## EXCEL EXCTRATOR convert excel to json                                                   |
| HTTP Request1                   | HTTP Request                  | Send image files to external universal extractor | Switch File Type (image)     | normalize output2                            | ## image ocr extraction                                                                     |
| Mistral Upload                  | HTTP Request                  | Upload PDF/DOC to Mistral OCR                     | Edit Fields3                 | Mistral Signed URL                           |                                                                                              |
| Mistral Signed URL              | HTTP Request                  | Get signed URL for uploaded file                   | Mistral Upload              | Mistral DOC OCR                              |                                                                                              |
| Mistral DOC OCR                | HTTP Request                  | Perform OCR on document via Mistral                | Mistral Signed URL          | normalize output1                            |                                                                                              |
| normalize output               | Set                           | Extract text content from OCR JSON                  | Aggregate                   | AI Agent to clean and format extracted data |                                                                                              |
| normalize output1              | Set                           | Extract text content from OCR JSON                  | Mistral DOC OCR             | AI Agent to clean and format extracted data |                                                                                              |
| normalize output2              | Set                           | Extract text content from image extractor JSON      | HTTP Request1               | AI Agent to clean and format extracted data |                                                                                              |
| AI Agent to clean and format extracted data | LangChain Agent             | Clean raw extracted text into structured JSON       | normalize output*, normalize output1, normalize output2 | clean ai output                              | ## AI Agent to clean and format exctracred data                                           |
| Mistral Cloud Chat Model2       | AI Language Model             | Underlying AI model for LangChain agent            | AI Agent to clean and format extracted data | AI Agent to clean and format extracted data (lmChatMistral) | ## AI Agent to clean and format exctracred data                                           |
| clean ai output                | Code                          | Parse AI JSON string output to n8n JSON items        | AI Agent to clean and format extracted data | json to html                                |                                                                                              |
| json to html                  | Code                          | Format timesheet JSON data to HTML email body        | clean ai output             | Send a message                               |                                                                                              |
| Send a message                | Gmail                        | Send verification email with timesheet data and form | json to html                | Loop Over Items2 (second output), Upload file | ## send for verification and wait for response if accepted save to drive                   |
| Upload file                  | Google Drive (upload)          | Upload verified timesheet file to Google Drive       | Send a message              | None                                         |                                                                                              |
| Clean excel data format       | Code                          | Convert Excel serial dates to ISO date strings        | Read Spreadsheet File       | Aggregate                                    |                                                                                              |
| Aggregate                   | Aggregate                     | Aggregate cleaned Excel rows into one dataset          | Clean excel data format     | normalize output                             |                                                                                              |
| Sticky Note (multiple)         | Sticky Note                   | Informational comments and workflow organization       | None                        | None                                         | See detailed notes below                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - Purpose: Manual start of workflow.

2. **Create Google Drive Node (Search files and folders)**  
   - Operation: Search files/folders in root (no folderId specified).  
   - Fields: id, name, webViewLink, mimeType, and all metadata.  
   - Credentials: Setup Google Drive OAuth2 with access to your Drive.

3. **Add SplitInBatches Node (Loop Over Items2)**  
   - Purpose: Process each file individually.  
   - Connect: From "Search files and folders" output.

4. **Add Set Node (Set File ID1)**  
   - Extract and assign: file_id = `{{$json.id}}`, file_type = `{{$json.mimeType}}`, file_title = `{{$json.name}}`, file_url = `{{$json.webViewLink}}`.  
   - Connect: From "Loop Over Items2" output.

5. **Add Google Drive Node (Download File)**  
   - Operation: Download file by file_id from Set File ID1.  
   - Options: Convert Google Docs to plain text.  
   - Credentials: Use same Google Drive OAuth2.  
   - Connect: From "Set File ID1".

6. **Add Switch Node (Switch File Type)**  
   - Add rules based on MIME type:  
     - pdf/doc: pdf, msword, docx, google doc mime types  
     - excel: xls, xlsx, google sheets mime types  
     - image: jpeg, png, tiff, gif, bmp  
   - Connect: From "Download File".

7. **For pdf/doc output:**  
   - Add Set Node (Edit Fields3) to prepare data for upload (clear fields or keep binary).  
   - Connect: From Switch pdf/doc output.

   - Add HTTP Request Node (Mistral Upload)  
     - URL: https://api.mistral.ai/v1/files  
     - Method: POST multipart/form-data  
     - Body: purpose="ocr", file (binary from Edit Fields3)  
     - Credential: Mistral Cloud API credentials.  
     - Connect: From Edit Fields3.

   - Add HTTP Request Node (Mistral Signed URL)  
     - URL: https://api.mistral.ai/v1/files/{{ $json.id }}/url?expiry=1  
     - Method: GET  
     - Headers: Accept: application/json  
     - Credential: Mistral Cloud API.  
     - Connect: From Mistral Upload.

   - Add HTTP Request Node (Mistral DOC OCR)  
     - URL: https://api.mistral.ai/v1/ocr  
     - Method: POST JSON  
     - Body: JSON specifying `model: "mistral-ocr-latest"`, document_url from signed URL output, bbox annotation format schema.  
     - Credential: Mistral Cloud API.  
     - Connect: From Mistral Signed URL.

   - Add Set Node (normalize output1)  
     - Assign content = `{{$json.pages[0].markdown}}` extracted from OCR response.  
     - Connect: From Mistral DOC OCR.

8. **For Excel output:**  
   - Add Spreadsheet File Node (Read Spreadsheet File)  
     - Parse Excel binary from Switch output.  
     - Header row enabled.  
     - Connect: From Switch Excel output.

   - Add Code Node (Clean excel data format)  
     - JavaScript to convert Excel serial dates to ISO string for date fields.  
     - Connect: From Read Spreadsheet File.

   - Add Aggregate Node  
     - Aggregate all rows to single item.  
     - Connect: From Clean excel data format.

   - Add Set Node (normalize output)  
     - Assign content = `{{$json.data}}` or equivalent data field.  
     - Connect: From Aggregate.

9. **For image files:**  
   - Add HTTP Request Node (HTTP Request1)  
     - POST multipart-form-data to https://universal-file-to-text-extractor.vercel.app/extract  
     - Parameters: mode=single, output_type=jsonl, include_images=true, files = binary data.  
     - Connect: From Switch Image output.

   - Add Set Node (normalize output2)  
     - Assign content = `{{JSON.parse($json["data"][0]).blocks[0].content}}`.  
     - Connect: From HTTP Request1.

10. **AI Processing:**  
    - Add LangChain Agent Node (AI Agent to clean and format extracted data)  
      - Input: content field from normalize output nodes.  
      - Prompt: Custom prompt with strict JSON output instructions.  
      - Connect: From normalize output, normalize output1, normalize output2.

    - Add Mistral Cloud Chat Model Node  
      - Credentials: Mistral Cloud API.  
      - Connect: As language model for LangChain Agent.

    - Add Code Node (clean ai output)  
      - Parse AI JSON string to n8n items.  
      - Connect: From AI Agent.

11. **Format HTML Email:**  
    - Add Code Node (json to html)  
      - Convert parsed JSON timesheet data to HTML table for email body.  
      - Connect: From clean ai output.

12. **Send Email for Verification:**  
    - Add Gmail Node (Send a message)  
      - Operation: sendAndWait  
      - Recipient: Replace "addemail" with actual verification email.  
      - Subject: "timesheet approval required"  
      - Message: Use HTML from json to html node.  
      - Custom Form Fields: original file url (readonly), accuracy dropdown (yes/no), comments field.  
      - Credentials: Gmail OAuth2.  
      - Connect: From json to html.

13. **Handle Verification Response:**  
    - From Gmail node’s success output:  
      - Loop back to Loop Over Items2 (second output) for further processing or upload.  
      - Add Google Drive Upload Node (Upload file)  
        - Upload to root or specified folder in Google Drive.  
        - Credentials: Google Drive OAuth2.  
        - Connect: From Gmail node verification success.

14. **Add Sticky Notes** (optional) for documentation and clarity on each major block and node group.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                       |
|----------------------------------------------------------------------------------------------|------------------------------------------------------|
| Workflow uses Mistral Cloud API for OCR and AI language model processing.                     | Requires Mistral Cloud API credentials and setup.    |
| Gmail OAuth2 credentials must be configured to enable sendAndWait email with custom forms.    | Make sure Gmail API and OAuth consent are configured.|
| Google Drive OAuth2 credentials require full access to Drive files and folders.               | Used for search, download, and upload operations.    |
| The universal file-to-text extractor API endpoint handles image OCR fallback.                 | https://universal-file-to-text-extractor.vercel.app/extract |
| AI agent prompt enforces strict JSON output formatting to ensure parsability.                 | Avoids markdown or extra text in AI responses.       |
| For Excel files, serial date conversion logic is critical to ensure date fields are usable.   | Based on Excel date origin 1899-12-30.                |
| The workflow includes multiple sticky notes for documentation and logical grouping.           | Refer to sticky notes within n8n editor for guidance.|

---

This structured and detailed documentation enables full understanding, reproduction, and modification of the "Extract Timesheet Data with Mistral OCR & Gmail Human Verification" workflow, including error anticipation and integration points.