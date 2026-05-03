Extract Invoice Data from Google Drive to Sheets with Mistral OCR and Gemini

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-google-drive-to-sheets-with-mistral-ocr-and-gemini-4868


# Extract Invoice Data from Google Drive to Sheets with Mistral OCR and Gemini

### 1. Workflow Overview

This workflow automates the extraction of invoice data stored in a specific Google Drive folder and appends the parsed structured data into a Google Sheets spreadsheet. It uses Mistral OCR services to convert invoice documents (PDFs/images) into text, then applies Google Gemini’s large language model (LLM) to extract structured invoice fields from the OCR text.

**Target Use Cases:**  
- Automating invoice data extraction from mixed document types (PDF/image) in Google Drive.  
- Maintaining a central sheet with validated invoice fields for financial or accounting systems.  
- Avoiding reprocessing of already handled files using Google Sheets as a tracking database.

**Logical Blocks:**  
- **1.1 Input Reception & File Tracking:** Monitor Google Drive folder for new invoices, load files, and filter out already processed ones.  
- **1.2 File Download & MIME Type Check:** Download unprocessed files and determine file type (image or document).  
- **1.3 OCR Processing (Mistral):** Upload files to Mistral, obtain signed URLs, and perform OCR using appropriate Mistral OCR endpoints for images or documents.  
- **1.4 OCR Text Consolidation:** Combine OCR results from multiple pages, remove unnecessary metadata fields, and prepare text for language model parsing.  
- **1.5 Field Extraction with Google Gemini:** Use Google Gemini LLM to extract structured data in JSON format from OCR text.  
- **1.6 Post-Processing & Data Saving:** Append extracted, structured invoice data back into Google Sheets, including file IDs for traceability.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & File Tracking

**Overview:**  
This block listens for new files created in a designated Google Drive folder. It loads all files and retrieves existing processed invoice records from a Google Sheet to avoid duplicate processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- On new file in Google Drive (Google Drive Trigger)  
- Load files from Google Drive folder (Google Drive)  
- Get Fields Schema (Google Sheets)  
- Get already processed rows from Sheets (Google Sheets)  
- Filter processed files (Merge)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation for testing.  
  - Connections: Triggers Load files and Get Fields Schema.  
  - Failure types: None typical; manual trigger.

- **On new file in Google Drive**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder ("Invoices_Inbox") for new file creation.  
  - Config: Trigger set for "fileCreated" event within folder ID "15Xpvr0Q4cBwYv1e_jNI8JO4LKAQmgYC2".  
  - Credentials: Google Drive OAuth2.  
  - Output: New file metadata.  
  - Edge cases: Folder permission errors, API rate limits, missing folder, or deleted folder.

- **Load files from Google Drive folder**  
  - Type: Google Drive (List files)  
  - Role: Retrieves all files from the specified Drive folder for batch processing.  
  - Config: Folder filter matching the same folder. Returns all files.  
  - ExecuteOnce: True (runs once per trigger).  
  - Credentials: Same Google Drive OAuth2.  
  - Failure: API errors, empty folder returns empty list.

- **Get Fields Schema**  
  - Type: Google Sheets (Read)  
  - Role: Reads the column headers (schema) from the Google Sheet "Sheet1" to define which invoice fields to extract.  
  - Config: Reads starting row 1, expects columns definition.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Spreadsheet access issues, schema missing or malformed.

- **Get already processed rows from Sheets**  
  - Type: Google Sheets (Read)  
  - Role: Loads existing rows from the Google Sheet to identify already processed Google Drive file IDs.  
  - Config: Reads full data from "Sheet1".  
  - Credentials: Google Sheets OAuth2.  
  - Output: Rows with previously processed file Ids.  
  - Edge cases: Sheet locked, permissions error.

- **Filter processed files**  
  - Type: Merge (Join)  
  - Role: Compares newly loaded Drive files against already processed file IDs to exclude duplicates.  
  - Config: Join mode "keepNonMatches" — keeps files that do NOT match IDs in processed list.  
  - Input1: Files from Drive folder  
  - Input2: Processed rows from Sheets  
  - Output: Only files not processed before.  
  - Edge cases: If processed list empty, all files pass; if mismatch keys, may process duplicates.

---

#### Block 1.2: File Download & MIME Type Check

**Overview:**  
Downloads each unprocessed file from Google Drive, then uses an If node to check if the file is an image or not, routing accordingly for OCR processing.

**Nodes Involved:**  
- Download file for OCR (Google Drive)  
- If (Conditional check)

**Node Details:**

- **Download file for OCR**  
  - Type: Google Drive (Download)  
  - Role: Downloads the actual file binary data for OCR processing.  
  - Config: Uses file ID from filtered files.  
  - Credentials: Google Drive OAuth2.  
  - Output: Binary data and metadata including MIME type.  
  - Failure: File missing, permission denied, large file/timeouts.

- **If**  
  - Type: Conditional  
  - Role: Checks if the downloaded file’s MIME type starts with "image/".  
  - Config: Condition on $binary.data.mimeType startsWith "image/".  
  - Output: Routes to either Mistral IMAGE OCR or Mistral DOC OCR flow.  
  - Edge cases: MIME type missing or unexpected, false negatives.

---

#### Block 1.3: OCR Processing (Mistral)

**Overview:**  
Uploads files to Mistral’s platform, obtains signed URLs, and performs OCR using either the image or document OCR API depending on file type.

**Nodes Involved:**  
- Mistral Upload / Mistral Upload1 (HTTP Request)  
- Mistral Signed URL / Mistral Signed URL1 (HTTP Request)  
- Mistral DOC OCR / Mistral IMAGE OCR (HTTP Request)

**Node Details:**

- **Mistral Upload / Mistral Upload1**  
  - Type: HTTP Request (POST multipart-form-data)  
  - Role: Uploads file binary data to Mistral API with purpose "ocr".  
  - Config: URL "https://api.mistral.ai/v1/files", sends binary data from previous node.  
  - Credentials: Mistral Cloud API key.  
  - Output: JSON with file ID.  
  - Failure: API key invalid, upload timeout, network issues.

- **Mistral Signed URL / Mistral Signed URL1**  
  - Type: HTTP Request (GET)  
  - Role: Requests a signed URL valid for 24 hours to access uploaded file.  
  - Config: URL template with file id from upload, query param expiry=24.  
  - Credentials: Mistral Cloud API.  
  - Output: JSON with temporary file access URL.  
  - Failure: Invalid file ID, expired token, API errors.

- **Mistral DOC OCR**  
  - Type: HTTP Request (POST JSON)  
  - Role: Sends document URL to Mistral OCR endpoint for document OCR.  
  - Config: JSON body specifying "mistral-ocr-latest" model, document URL, requests base64 image inclusion.  
  - Credentials: Mistral Cloud API.  
  - Output: OCR result with pages and markdown text.  
  - Failure: API errors, invalid document, rate limits.

- **Mistral IMAGE OCR**  
  - Type: HTTP Request (POST JSON)  
  - Role: Sends image URL to Mistral OCR endpoint for image OCR.  
  - Config: Similar to DOC OCR but document type is "image_url".  
  - Credentials: Mistral Cloud API.  
  - Output: OCR result.  
  - Failure: Same as above.

---

#### Block 1.4: OCR Text Consolidation

**Overview:**  
Joins the OCR text from multiple pages into a single string, removes unnecessary fields, and prepares the schema for extraction.

**Nodes Involved:**  
- extract schema (Code)  
- Join OCR Pages and remove id field (Code)

**Node Details:**

- **extract schema**  
  - Type: Code  
  - Role: Extracts column headers (keys) from the first row of the schema read from Google Sheets.  
  - Logic: Gets keys from first JSON item, returns as schema array.  
  - Input: Output of Get Fields Schema.  
  - Output: Schema array for later use.  
  - Edge cases: Empty input, malformed data.

- **Join OCR Pages and remove id field**  
  - Type: Code  
  - Role:  
    - For each OCR document, joins all page markdown texts with double newline separation.  
    - Filters out "Id" and "row_number" fields from schema.  
    - Outputs combined OCR text and filtered schema.  
  - Input: OCR JSON from Mistral OCR nodes, plus schema from extract schema node.  
  - Output: Object with `ocr_text` and `schema`.  
  - Edge cases: No pages available, missing fields.

---

#### Block 1.5: Field Extraction with Google Gemini

**Overview:**  
Uses Google Gemini LLM to parse the consolidated OCR text and extract structured invoice fields as JSON.

**Nodes Involved:**  
- Google Gemini Chat Model (LLM)  
- Field Extractor (Langchain Agent)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Provides LLM inference with Google’s Gemini model (version 2.5 flash preview).  
  - Credentials: Google Palm API key.  
  - Input: Text prompt from Field Extractor node.  
  - Output: LLM raw JSON response (invoice data).  
  - Failure: API quota exceeded, auth errors, malformed prompt.

- **Field Extractor**  
  - Type: Langchain Agent  
  - Role: Prepares a prompt to extract specific invoice fields from OCR text using Gemini model.  
  - Configuration:  
    - System message defines the assistant role as invoice parser.  
    - Prompt text dynamically injects schema fields and OCR text.  
    - Enforces strict JSON-only response formatting.  
  - Input: OCR text and schema from previous node.  
  - Output: Raw JSON string with extracted invoice data.  
  - Edge cases: LLM hallucination, invalid JSON output, missing fields.

---

#### Block 1.6: Post-Processing & Data Saving

**Overview:**  
Parses LLM JSON output, injects original Google Drive file IDs for traceability, and appends the structured invoice data to Google Sheets.

**Nodes Involved:**  
- Add google drive Id back (Code)  
- Save data to Google Sheets (Google Sheets)

**Node Details:**

- **Add google drive Id back**  
  - Type: Code  
  - Role:  
    - Parses the JSON string output from Field Extractor node.  
    - Adds corresponding Google Drive file ID from filtered files to each extracted data item under key "Id".  
  - Input: Raw JSON string outputs and filtered files list.  
  - Output: JSON objects ready for Sheets append.  
  - Edge cases: JSON parse errors, index mismatch.

- **Save data to Google Sheets**  
  - Type: Google Sheets (Append)  
  - Role: Appends the structured invoice data to a Google Sheets document.  
  - Config: Appends to "Sheet1" with columns mapped automatically.  
  - Columns: Id, Invoice No, Invoice date, Invoice Period, Gross Amount incl. VAT, IBAN.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: API quota exceeded, sheet locked, data type mismatch.

---

### 3. Summary Table

| Node Name                     | Node Type                            | Functional Role                              | Input Node(s)                          | Output Node(s)                      | Sticky Note                                                                                           |
|-------------------------------|------------------------------------|----------------------------------------------|--------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                     | Manual start trigger for testing              | None                                 | Load files, Get Fields Schema       |                                                                                                     |
| On new file in Google Drive    | Google Drive Trigger               | Watches folder for new files                   | None                                 | Load files, Get Fields Schema       |                                                                                                     |
| Load files from Google Drive folder | Google Drive (List files)          | Lists all files in designated Drive folder   | When clicking..., On new file         | Filter processed files              |                                                                                                     |
| Get Fields Schema              | Google Sheets (Read)               | Reads schema (column headers) from sheet     | When clicking..., On new file         | extract schema                    | ### Select Sheet<br>Add column headers in the sheet to which you save data on row one to specify which fields you are interested in extracting. |
| Get already processed rows from Sheets | Google Sheets (Read)               | Reads already processed file IDs from sheet  | extract schema                      | Filter processed files              | >> ### First column header in cell A1 MUST BE "Id". Id tracks the google drive Id and ensures the workflow is not processing previously processed files |
| Filter processed files         | Merge (Join)                      | Filters out files already processed           | Load files, Get already processed rows | Download file for OCR             |                                                                                                     |
| Download file for OCR          | Google Drive (Download)            | Downloads file binary for OCR                  | Filter processed files              | If                                |                                                                                                     |
| If                           | Conditional                       | Checks if file is image or document           | Download file for OCR               | Mistral Upload1, Mistral Upload    |                                                                                                     |
| Mistral Upload                | HTTP Request                      | Uploads file for OCR (document path)          | If (document branch)                | Mistral Signed URL                 | Requires Mistral API Key for La Platforme<br>https://mistral.ai/products/la-plateforme               |
| Mistral Signed URL            | HTTP Request                      | Retrieves signed URL for uploaded document    | Mistral Upload                    | Mistral DOC OCR                  |                                                                                                     |
| Mistral DOC OCR               | HTTP Request                      | Performs OCR on document URL                    | Mistral Signed URL                 | Join OCR Pages and remove id field |                                                                                                     |
| Mistral Upload1               | HTTP Request                      | Uploads file for OCR (image path)              | If (image branch)                  | Mistral Signed URL1               |                                                                                                     |
| Mistral Signed URL1           | HTTP Request                      | Retrieves signed URL for uploaded image        | Mistral Upload1                   | Mistral IMAGE OCR                |                                                                                                     |
| Mistral IMAGE OCR             | HTTP Request                      | Performs OCR on image URL                       | Mistral Signed URL1               | Join OCR Pages and remove id field |                                                                                                     |
| extract schema                | Code                             | Extracts schema keys from sheet data           | Get Fields Schema                 | Get already processed rows from Sheets |                                                                                                     |
| Join OCR Pages and remove id field | Code                             | Joins OCR page texts and cleans schema         | Mistral DOC OCR, Mistral IMAGE OCR | Field Extractor                  |                                                                                                     |
| Google Gemini Chat Model      | Langchain LLM Model              | Runs Google Gemini LLM for field extraction    | Field Extractor                   | Field Extractor                  |                                                                                                     |
| Field Extractor               | Langchain Agent                  | Prepares prompt and extracts invoice fields    | Join OCR Pages and remove id field, Google Gemini Chat Model | Add google drive Id back         |                                                                                                     |
| Add google drive Id back      | Code                             | Parses JSON output and adds Drive file IDs     | Field Extractor                  | Save data to Google Sheets         |                                                                                                     |
| Save data to Google Sheets    | Google Sheets (Append)            | Saves extracted invoice data into spreadsheet | Add google drive Id back          | None                            | ### Select sheet to save data (same as at the start)                                               |
| Sticky Note                   | Sticky Note                      | Notes on modifying folder source                | None                             | None                           | ### Modify Folder source                                                                            |
| Sticky Note1                  | Sticky Note                      | Notes on viewing files using file IDs           | None                             | None                           | ### File Ids<br>You can view the files using Id from the first column. replace the <Id> below<br>https://drive.google.com/file/d/<Id>/view |
| Sticky Note2                  | Sticky Note                      | Notes on polling interval                        | None                             | None                           | ### Set interval for checking drive folder and pull new files                                      |
| Sticky Note3                  | Sticky Note                      | Notes on sheet setup and column headers          | None                             | None                           | ### Select Sheet<br>Add column headers in the sheet to which you save data on row one to specify which fields you are interested in extracting. |
| Sticky Note4                  | Sticky Note                      | Reminder to select sheet                          | None                             | None                           | ### Select sheet                                                                                   |
| Sticky Note5                  | Sticky Note                      | Reminder about saving data to sheet              | None                             | None                           | ### Select sheet to save data (same as at the start)                                              |
| Sticky Note6                  | Sticky Note                      | Reminder about first sheet column header must be "Id" | None                             | None                           | >> ### First column header in cell A1 MUST BE "Id". Id tracks the google drive Id and ensures the workflow is not processing previously processed files |
| Sticky Note7                  | Sticky Note                      | Notes about using Drive file ID                   | None                             | None                           | Uses drive ID - no need to modify                                                                  |
| Sticky Note8                  | Sticky Note                      | Notes about Mistral API key requirement           | None                             | None                           | Requires Mistral API Key for La Platforme<br>https://mistral.ai/products/la-plateforme              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: For manual test runs.

2. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure:  
     - Event: fileCreated  
     - Folder to watch: Set to your target folder ID (e.g., "15Xpvr0Q4cBwYv1e_jNI8JO4LKAQmgYC2")  
     - Credential: Google Drive OAuth2 with appropriate permissions.

3. **Create Google Drive List Node** ("Load files from Google Drive folder")  
   - Type: Google Drive (List files)  
   - Configure to list all files in the same folder as trigger above.  
   - Set to execute once per trigger.  
   - Credential: Same Google Drive OAuth2.

4. **Create Google Sheets Node** ("Get Fields Schema")  
   - Type: Google Sheets (Read)  
   - Configure to read from your target spreadsheet and sheet (e.g., "Sheet1", document ID as per your sheet).  
   - Read starting at first row to get the header schema.  
   - Credential: Google Sheets OAuth2.

5. **Create Code Node** ("extract schema")  
   - Paste code to extract keys from first row JSON to create schema array.

6. **Create Google Sheets Node** ("Get already processed rows from Sheets")  
   - Type: Google Sheets (Read)  
   - Configure to read full rows from the same sheet to get existing processed file IDs.  
   - Credential: Google Sheets OAuth2.

7. **Create Merge Node** ("Filter processed files")  
   - Configure as Join with mode "keepNonMatches"  
   - Input1: Files from Drive list  
   - Input2: Processed rows from Sheets  
   - Join by matching file ID to exclude already processed files.

8. **Create Google Drive Node** ("Download file for OCR")  
   - Type: Google Drive (Download)  
   - Config: Use file ID from filtered files.  
   - Credential: Google Drive OAuth2.

9. **Create If Node**  
   - Condition: Check if downloaded file’s MIME type starts with "image/".  
   - Output branches: True (image), False (document).

10. **Create Mistral Upload Nodes (2)**  
    - Type: HTTP Request (POST multipart-form-data)  
    - URL: https://api.mistral.ai/v1/files  
    - Body parameters: purpose="ocr", file binary from previous node.  
    - Credential: Mistral Cloud API key.  
    - One for image branch ("Mistral Upload1"), one for document branch ("Mistral Upload").

11. **Create Mistral Signed URL Nodes (2)**  
    - Type: HTTP Request (GET)  
    - URL template: https://api.mistral.ai/v1/files/{{ $json.id }}/url  
    - Query param: expiry=24  
    - Credential: Mistral Cloud API.

12. **Create Mistral OCR Nodes (2)**  
    - Type: HTTP Request (POST JSON)  
    - For documents: JSON body with "document_url" from signed URL, model "mistral-ocr-latest", include_image_base64=true  
    - For images: JSON body with "image_url" from signed URL, model "mistral-ocr-latest"  
    - Credential: Mistral Cloud API.

13. **Create Code Node** ("Join OCR Pages and remove id field")  
    - Paste code to join page markdown texts and remove "Id" and "row_number" from schema.

14. **Create Langchain Google Gemini Chat Model Node**  
    - Configure with model name "models/gemini-2.5-flash-preview-05-20"  
    - Credential: Google Palm API key.

15. **Create Langchain Agent Node** ("Field Extractor")  
    - Text prompt: Inject schema and OCR text with instructions for JSON-only output.  
    - System message: "You are a document parsing assistant designed to extract structured data from invoice PDFs for automated uploading and validation in a financial system."  
    - Connect input to join OCR text output.  
    - Connect AI Language Model input to Gemini Chat Model node.

16. **Create Code Node** ("Add google drive Id back")  
    - Paste code to parse JSON output and add Google Drive file Id for traceability.

17. **Create Google Sheets Node** ("Save data to Google Sheets")  
    - Type: Append operation  
    - Configure to append to the same sheet used for tracking.  
    - Map columns automatically based on schema.  
    - Credential: Google Sheets OAuth2.

18. **Connect nodes following the data flow:**  
    - Trigger nodes → Load files & Get Fields Schema → extract schema → Get already processed rows → Filter processed files → Download file → If → Mistral Upload → Mistral Signed URL → Mistral OCR → Join OCR Pages → Field Extractor (via Gemini LLM) → Add google drive Id back → Save data to Sheets.

19. **Credential Setup:**  
    - Google Drive OAuth2 with read/download permissions.  
    - Google Sheets OAuth2 with read/write access.  
    - Mistral Cloud API key for OCR endpoints.  
    - Google Palm API key for Gemini LLM.

20. **Sheet Setup:**  
    - Ensure the target Google Sheet has headers in the first row.  
    - First column header **must** be "Id" to track processed files.  
    - Other columns correspond to invoice fields to extract (e.g., Invoice No, Invoice date, IBAN).

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                 |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Requires Mistral API Key for La Platforme: https://mistral.ai/products/la-plateforme              | Mistral OCR API authentication                                  |
| File Ids can be viewed via URL: https://drive.google.com/file/d/<Id>/view                         | Useful for manual verification of files                        |
| First column header in sheet **must** be "Id" to track processed files and avoid duplicates       | Critical for workflow correctness                              |
| Set interval for checking Drive folder and pulling new files accordingly                          | Configure Google Drive Trigger polling frequency               |
| Add column headers in Google Sheet on row one to specify invoice fields to extract                | Defines schema for Langchain extraction                         |

---

**Disclaimer:**  
The provided text is exclusively generated from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.