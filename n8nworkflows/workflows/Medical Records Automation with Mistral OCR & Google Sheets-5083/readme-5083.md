Medical Records Automation with Mistral OCR & Google Sheets

https://n8nworkflows.xyz/workflows/medical-records-automation-with-mistral-ocr---google-sheets-5083


# Medical Records Automation with Mistral OCR & Google Sheets

### 1. Workflow Overview

This workflow automates the extraction and processing of medical records from uploaded document images or PDFs using Mistral OCR technology and stores the structured data into a Google Sheets spreadsheet. It is designed to handle multi-page documents potentially containing multiple patient records, extracting key medical fields such as patient name, diagnosis, medications, lab results, and visit details.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user-uploaded medical documents via a web form.
- **1.2 Document Upload & Access:** Uploads the document to Mistral’s file hosting service and retrieves a secure, temporary download URL.
- **1.3 OCR Processing:** Sends the document URL to Mistral’s OCR API to extract text and images.
- **1.4 Data Cleaning & Structuring:** Parses the raw OCR results using JavaScript to extract structured patient medical data.
- **1.5 Data Storage:** Appends the cleaned and structured patient data records into a designated Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving medical document uploads through a user-facing web form. It triggers the workflow upon form submission containing a single file.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Name:** On form submission  
  - **Type:** Form Trigger  
  - **Role:** Captures uploaded document files from users via a form titled "Document OCR" with one required file field labeled "Document".  
  - **Configuration:**  
    - Form title: "Document OCR"  
    - Form description: "Please upload your document for processing."  
    - Single file upload allowed, required field  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Passes uploaded file data to the next node  
  - **Version-specific notes:** Uses typeVersion 2.2 which supports file upload fields  
  - **Potential failure points:** Form not submitted, missing file, invalid file type, webhook failure  
  - **Sticky note content:** "Receives uploaded document from user to begin OCR processing."

---

#### 2.2 Document Upload & Access

- **Overview:**  
  This block uploads the user-submitted document file to Mistral’s file hosting API and fetches a temporary signed URL to enable secure access for OCR processing.

- **Nodes Involved:**  
  - Upload to Mistral  
  - Get Signed URL

- **Node Details:**  

  1. **Upload to Mistral**  
     - Type: HTTP Request (POST)  
     - Role: Uploads the binary document file to Mistral’s `/v1/files` endpoint with purpose "ocr".  
     - Configuration:  
       - URL: `https://api.mistral.ai/v1/files`  
       - Method: POST  
       - Content-Type: multipart/form-data  
       - Body Parameters:  
         - purpose = "ocr"  
         - file = binary file from form field "Document"  
       - Authentication: HTTP Header Auth via saved credentials  
     - Inputs: Receives binary file from form submission node  
     - Outputs: Receives JSON response containing file metadata including file ID  
     - Failure modes: Auth errors, file upload errors, network timeout, invalid file  
     - Sticky note: "Sends uploaded document to Mistral for OCR file hosting."

  2. **Get Signed URL**  
     - Type: HTTP Request (GET)  
     - Role: Retrieves a signed URL with 24-hour expiry to access the uploaded document file.  
     - Configuration:  
       - URL: Template `https://api.mistral.ai/v1/files/{{ $json.id }}/url` where `id` is from previous node  
       - Query parameter: expiry=24 (hours)  
       - Authentication: HTTP Header Auth same as above  
     - Inputs: Receives file ID from upload node output  
     - Outputs: JSON containing signed URL for document access  
     - Failure modes: Auth failure, invalid file ID, network issues  
     - Sticky note: "Gets secure download URL from Mistral for the uploaded document."

---

#### 2.3 OCR Processing

- **Overview:**  
  Using the signed URL, this block calls Mistral’s OCR API to extract textual content and embedded images from the document.

- **Nodes Involved:**  
  - Get OCR Results

- **Node Details:**  
  - Type: HTTP Request (POST)  
  - Role: Sends a JSON payload to Mistral OCR endpoint to process the document located at the signed URL.  
  - Configuration:  
    - URL: `https://api.mistral.ai/v1/ocr`  
    - JSON body includes:  
      - `"model": "mistral-ocr-latest"`  
      - `"document": {"type": "document_url", "document_url": "{{ $json.url }}"}`  
      - `"include_image_base64": true` (to include base64 images)  
    - Authentication: HTTP Header Auth same as previous  
  - Inputs: Takes signed URL from previous node  
  - Outputs: OCR results JSON including pages, markdown text per page, and detected fields  
  - Failure modes: Auth errors, invalid URL, OCR service downtime, JSON parsing errors  
  - Sticky note: "Calls Mistral OCR with signed document URL to extract data."

---

#### 2.4 Data Cleaning & Structuring

- **Overview:**  
  Parses the raw OCR JSON output to extract structured patient record fields. It can handle documents containing multiple patient records, extracting fields such as Name, Date of Birth, Diagnosis, Medications, and Lab Results. Special parsing logic addresses multiline Lab Results.

- **Nodes Involved:**  
  - Data cleaning (Code node)

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role:  
    - Aggregates markdown text from all pages of OCR output  
    - Splits text into patient record sections based on patterns like "Patient Record 1" or "🧾 Patient Record 2"  
    - Extracts fields using regex such as "Patient Name", "Diagnosis", and handles multiline "Lab Results" by capturing text until next field  
    - Filters to include only records with at least a Name or Patient ID  
    - Returns array of structured patient records as JSON objects  
  - Key expressions: regex patterns for each field, text splitting, string trimming, multiline field handling  
  - Inputs: OCR result JSON  
  - Outputs: Array of cleaned patient record objects  
  - Version-specific notes: Uses JavaScript ES6 features supported by n8n code node v2  
  - Failure modes: Invalid/malformed OCR data, missing pages, regex mismatches, empty results  
  - Sticky note: "Processes raw OCR text into structured patient records for sheet entry."

---

#### 2.5 Data Storage

- **Overview:**  
  Appends the structured patient records into a Google Sheets spreadsheet under the "Patients" sheet tab. Fields such as Name, Notes, Diagnosis, and others are mapped automatically.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - Type: Google Sheets (Append operation)  
  - Role: Writes each patient record as a new row into a Google Sheet named "Medical Records - Extracted", specifically into the "Patients" sheet/tab.  
  - Configuration:  
    - Spreadsheet ID: `1jRNGNrHAFnvNAAnCHW0vM2784GxIPolzT4x_rFWZRvU`  
    - Sheet Name: "Patients" (gid: 1417843853)  
    - Columns are auto-mapped from input data fields such as Name, Notes, Date of Birth, Diagnosis, etc.  
    - Mapping Mode: AutoMapInputData  
    - Operation: Append (adds new rows)  
  - Inputs: Receives structured patient records from Data cleaning node  
  - Outputs: None (terminal data storage node)  
  - Credentials: Google Sheets OAuth2 account with write access  
  - Failure modes: OAuth token expired, permission denied, sheet not found, data mapping errors  
  - Sticky note:  
    - Link to Google Sheet used: [Google sheet](https://docs.google.com/spreadsheets/d/1jRNGNrHAFnvNAAnCHW0vM2784GxIPolzT4x_rFWZRvU/edit?usp=sharing)  
    - "Appends cleaned patient data to the Google Sheet."

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                            | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                              |
|-------------------|---------------------|------------------------------------------|------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------|
| On form submission| Form Trigger        | Receives uploaded document from user     | None                   | Upload to Mistral       | Receives uploaded document from user to begin OCR processing.                                                            |
| Upload to Mistral | HTTP Request        | Uploads document to Mistral file hosting | On form submission     | Get Signed URL          | Sends uploaded document to Mistral for OCR file hosting.                                                                 |
| Get Signed URL    | HTTP Request        | Retrieves temporary signed URL from Mistral | Upload to Mistral      | Get OCR Results         | Gets secure download URL from Mistral for the uploaded document.                                                         |
| Get OCR Results   | HTTP Request        | Sends signed URL to OCR API for text extraction | Get Signed URL         | Data cleaning           | Calls Mistral OCR with signed document URL to extract data.                                                               |
| Data cleaning     | Code                | Parses OCR text and extracts structured patient records | Get OCR Results        | Google Sheets           | Processes raw OCR text into structured patient records for sheet entry.                                                  |
| Google Sheets     | Google Sheets Append| Appends extracted patient data into spreadsheet | Data cleaning          | None                    | Appends cleaned patient data to the Google Sheet. See [Google sheet](https://docs.google.com/spreadsheets/d/1jRNGNrHAFnvNAAnCHW0vM2784GxIPolzT4x_rFWZRvU/edit?usp=sharing) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger  
   - Configure Form:  
     - Title: "Document OCR"  
     - Description: "Please upload your document for processing."  
     - Add one Form Field:  
       - Type: File  
       - Label: "Document"  
       - Required: Yes  
       - Multiple Files: No  
   - This node triggers the workflow when a user uploads a document.

2. **Add HTTP Request Node to Upload Document to Mistral**  
   - Name: `Upload to Mistral`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/files`  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `purpose`: "ocr"  
     - `file`: Set as binary from the form field "Document" (use parameter type "formBinaryData")  
   - Authentication: HTTP Header Auth  
     - Setup credentials with Mistral API key in header (e.g., `Authorization: Bearer <token>`)  
   - Connect output of `On form submission` node to input here.

3. **Add HTTP Request Node to Get Signed URL from Mistral**  
   - Name: `Get Signed URL`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression `https://api.mistral.ai/v1/files/{{ $json.id }}/url` where `$json.id` is file ID from previous node  
   - Query Parameters:  
     - `expiry`: 24 (hours)  
   - Authentication: Same HTTP Header Auth credentials as above  
   - Connect output of `Upload to Mistral` node to input here.

4. **Add HTTP Request Node to Call Mistral OCR API**  
   - Name: `Get OCR Results`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/ocr`  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "model": "mistral-ocr-latest",
       "document": {
         "type": "document_url",
         "document_url": "{{ $json.url }}"
       },
       "include_image_base64": true
     }
     ```  
     Use expression to fill `"document_url"` dynamically from `Get Signed URL` node output.  
   - Authentication: Same HTTP Header Auth credentials  
   - Connect output of `Get Signed URL` node to input here.

5. **Add Code Node for Data Cleaning & Structuring**  
   - Name: `Data cleaning`  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that:  
     - Aggregates markdown text from OCR pages  
     - Splits into patient record sections  
     - Extracts fields using regex  
     - Handles multi-line lab results  
     - Returns array of patient records  
   - Connect output of `Get OCR Results` node here.

6. **Add Google Sheets Node to Append Data**  
   - Name: `Google Sheets`  
   - Type: Google Sheets (Append)  
   - Operation: Append  
   - Spreadsheet ID: `1jRNGNrHAFnvNAAnCHW0vM2784GxIPolzT4x_rFWZRvU` (or your own copy)  
   - Sheet Name: "Patients"  
   - Mapping Mode: Auto map input data fields to sheet columns  
   - Configure OAuth2 credentials with Google account having access to the spreadsheet  
   - Connect output of `Data cleaning` node here.

7. **Validate Connections and Test**  
   - Ensure `On form submission` → `Upload to Mistral` → `Get Signed URL` → `Get OCR Results` → `Data cleaning` → `Google Sheets` are connected sequentially.  
   - Test by submitting a sample document via form trigger webhook URL.  
   - Monitor logs for errors or data extraction issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow author: David Olusola ([daexai.com](https://www.daexai.com))                                                                                                             | Project credit and contact                                                                                                                                        |
| Webhook ID for form trigger: f9d60b5f-0a09-4654-a840-84a0f745321e                                                                                                                | Useful for testing or routing webhook calls                                                                                                                     |
| Google Sheets template linked here (make a copy before use): [Google sheet](https://docs.google.com/spreadsheets/d/1jRNGNrHAFnvNAAnCHW0vM2784GxIPolzT4x_rFWZRvU/edit?usp=sharing) | Spreadsheet contains columns matching extracted fields for easy data entry                                                                                      |
| Important extracted fields include: Name, Date of Birth, Patient ID, Diagnosis, Medications, Lab Results, Notes, etc.                                                             | Ensures critical medical data is captured                                                                                                                       |
| Mistral OCR API supports base64 image inclusion in results, enabling advanced processing if needed                                                                                | Can be extended for image-based processing or validation                                                                                                        |
| Handle potential failures such as expired signed URLs by adjusting expiry parameter or implementing retry logic                                                                   | Mistral signed URL expiry default is 24 hours; consider your use case                                                                                           |
| JavaScript parsing relies on consistent text patterns like "Patient Record", "Patient Name:", and field labels — documents with very different formatting may require adjustments | Customize regex or add additional parsing logic for other document formats                                                                                       |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.