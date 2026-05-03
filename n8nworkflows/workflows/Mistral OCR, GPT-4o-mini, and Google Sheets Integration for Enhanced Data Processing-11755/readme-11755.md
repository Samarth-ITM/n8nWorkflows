Mistral OCR, GPT-4o-mini, and Google Sheets Integration for Enhanced Data Processing

https://n8nworkflows.xyz/workflows/mistral-ocr--gpt-4o-mini--and-google-sheets-integration-for-enhanced-data-processing-11755


# Mistral OCR, GPT-4o-mini, and Google Sheets Integration for Enhanced Data Processing

### 1. Workflow Overview

This workflow automates the extraction and processing of invoice data from uploaded documents using a combination of Mistral OCR, GPT-4o-mini for AI extraction, and Google Sheets for storage and tracking. It is designed to handle multiple invoices in PDF, PNG, or JPG formats, extract structured financial and vendor data fields, validate extraction confidence, and save the results with status annotations into a Google Sheet.

**Target Use Cases:**  
- Automated invoice data entry and digitization  
- Batch processing of scanned or photographed invoices  
- Real-time invoice data validation and error detection  
- Centralized tracking of processed invoices with confidence scoring

**Logical Blocks:**  
- **1.1 Input Reception:** Receives invoice files uploaded by users via a form and splits them into individual items.  
- **2.1 OCR Processing:** Converts files to base64, prepares OCR requests, sends to Mistral OCR API, and formats raw OCR text.  
- **3.1 AI Extraction:** Uses GPT-4o-mini to extract structured invoice fields from OCR text.  
- **3.2 Validation:** Validates extracted data fields, calculates confidence scores, and assigns processing status.  
- **4.1 Output & Storage:** Appends processed and validated invoice data to a Google Sheet and manages rate limiting for batch processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives invoice uploads from users via a web form, supporting multiple files, then splits these into individual invoice items for processing.

**Nodes Involved:**  
- Invoice Upload Form  
- Split Files  
- Loop

**Node Details:**

- **Invoice Upload Form**  
  - Type: Form Trigger  
  - Purpose: Presents an upload form for PDF, PNG, JPG invoice files with multi-file support.  
  - Key Configurations: Accepts multiple files (.pdf, .png, .jpg, .jpeg), required field.  
  - Inputs: HTTP form submission (webhook)  
  - Outputs: Single item with multiple binary files  
  - Edge Cases: No files uploaded triggers error downstream; file type restrictions apply.

- **Split Files**  
  - Type: Code  
  - Purpose: Splits uploaded binary files into separate workflow items, each representing one invoice file.  
  - Key Configurations: Iterates over binary data keys, creates one item per file with its metadata.  
  - Inputs: Single item from the form trigger  
  - Outputs: Multiple items, each with one binary file and JSON metadata (filename, mimeType).  
  - Edge Cases: Throws error if no files present; handles missing fileName or mimeType with defaults.

- **Loop**  
  - Type: SplitInBatches  
  - Purpose: Controls batch processing of individual invoice files, enabling rate limits and sequential execution.  
  - Inputs: Multiple items from Split Files  
  - Outputs: Items one by one to OCR preparation  
  - Edge Cases: Should handle empty inputs gracefully; batch size defaults unless configured.

---

#### 2.1 OCR Processing

**Overview:**  
Prepares the invoice file data for OCR by converting binaries to base64, constructs Mistral OCR API requests, sends files for OCR, and processes the returned OCR text.

**Nodes Involved:**  
- Prepare OCR Request  
- Mistral OCR  
- Format OCR Text  

**Node Details:**

- **Prepare OCR Request**  
  - Type: Code  
  - Purpose: Converts each invoice file binary to a base64 data URL and builds the Mistral OCR request body based on file type (document or image).  
  - Key Configurations: Differentiates PDFs/docs vs image files; uses model "mistral-ocr-latest".  
  - Inputs: Single item from Loop (invoice file binary + metadata)  
  - Outputs: JSON with filename, mimeType, and requestBody formatted for Mistral OCR.  
  - Edge Cases: Correctly handles mime types; assumes binary data is present; no fallback if unknown type.

- **Mistral OCR**  
  - Type: HTTP Request  
  - Purpose: Sends OCR request to Mistral API, receives parsed text data.  
  - Key Configurations: POST to https://api.mistral.ai/v1/ocr, 120s timeout, uses HTTP Header Auth credential with Bearer token.  
  - Inputs: OCR request JSON from Prepare OCR Request  
  - Outputs: OCR response JSON including pages and text.  
  - Edge Cases: API rate limits, timeouts, authentication errors possible.

- **Format OCR Text**  
  - Type: Code  
  - Purpose: Extracts human-readable text from the OCR response, concatenates per-page markdown text or fallback text field, counts pages.  
  - Key Configurations: Produces fallback message if no text found.  
  - Inputs: Mistral OCR JSON response and original filename/mimeType from Prepare OCR Request (via expression).  
  - Outputs: JSON with full OCR text, filename, and page count.  
  - Edge Cases: Missing or malformed OCR response; empty text handling.

---

#### 3.1 AI Extraction

**Overview:**  
Uses GPT-4o-mini to extract structured invoice data fields from OCR text by applying an information extraction model with a system prompt.

**Nodes Involved:**  
- OpenAI Model  
- Extract Invoice Fields  

**Node Details:**

- **OpenAI Model**  
  - Type: Langchain LM Chat OpenAI  
  - Purpose: Hosts GPT-4o-mini language model for prompt-based extraction.  
  - Key Configurations: Model set to “gpt-4o-mini”, standard OpenAI API credentials used.  
  - Inputs: OCR text from Format OCR Text  
  - Outputs: Chat completion response (passed internally to extraction node)  
  - Edge Cases: OpenAI API rate limits, auth errors, model availability.

- **Extract Invoice Fields**  
  - Type: Langchain Information Extractor  
  - Purpose: Parses GPT-4o-mini output to extract specified invoice fields precisely.  
  - Key Configurations:  
    - System prompt instructs extraction with decimal and date formatting rules.  
    - Extracts fields: invoice_number, invoice_date (YYYY-MM-DD), vendor_name, vendor_tax_id, subtotal (number), tax_rate (number), tax_amount (number), total_amount (number), currency.  
  - Inputs: OCR text string  
  - Outputs: JSON with extracted fields, null for missing fields.  
  - Edge Cases: Ambiguous OCR input, model misunderstanding, missing fields, formatting errors.

---

#### 3.2 Validation

**Overview:**  
Validates the extracted invoice data, assigns a confidence score (high, medium, low) based on presence of key fields, detects issues, and sets processing status.

**Nodes Involved:**  
- Validate & Score  

**Node Details:**

- **Validate & Score**  
  - Type: Code  
  - Purpose: Checks presence of critical fields (invoice_number, invoice_date, vendor_name, total_amount), counts filled fields, compiles issues list, assigns confidence level and status ("Processed", "Review", "Error").  
  - Key Configurations:  
    - Required fields count thresholds: 4 for high confidence, 2 for medium, else low.  
    - Issues concatenated as semicolon-separated string.  
    - Adds processing timestamp and page count metadata.  
  - Inputs: Extracted fields from Extract Invoice Fields and metadata from Format OCR Text  
  - Outputs: JSON with all invoice data, confidence, status, issues, processed timestamp, page count  
  - Edge Cases: Missing fields, zero values, inconsistent data, date format errors.

---

#### 4.1 Output & Storage

**Overview:**  
Appends the validated invoice data into a designated Google Sheet and manages rate limiting to avoid API throttling.

**Nodes Involved:**  
- Save to Sheets  
- Rate Limit  

**Node Details:**

- **Save to Sheets**  
  - Type: Google Sheets  
  - Purpose: Appends processed invoice data to a specific Google Sheet and sheet tab.  
  - Key Configurations:  
    - Maps workflow fields to Sheet columns exactly as required (Invoice Number, Date, Vendor Name, etc.)  
    - Append operation on sheet named "Sheet1" within specified Google Sheet ID.  
    - Uses OAuth2 credentials for Google Sheets access.  
  - Inputs: Validated invoice data from Validate & Score  
  - Outputs: Success status, passes to Rate Limit node  
  - Edge Cases: Google API quota errors, invalid Sheet ID, credential expiration.

- **Rate Limit**  
  - Type: Wait  
  - Purpose: Pauses workflow for 1 second to avoid hitting Google Sheets API limits during batch append operations.  
  - Key Configurations: Wait time set to 1 second.  
  - Inputs: Completion of Save to Sheets  
  - Outputs: Loops back to Loop node to process next batch item  
  - Edge Cases: Delay too short to prevent throttling, workflow backlog during large batches.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                                           |
|-----------------------|----------------------------------|-------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Main Overview         | Sticky Note                      | Workflow high-level description | -                        | -                        | ## Invoice OCR Processor with Mistral AI... Lists setup steps, fields extracted, and general workflow purpose                        |
| Warning - Credentials | Sticky Note                      | Credential and setup warnings  | -                        | -                        | ## ⚠️ Required Credentials... Mistral, OpenAI, Google Sheets API setup instructions and required sheet columns                      |
| Section 1             | Sticky Note                      | Input block label              | -                        | -                        | ## 1. Input                                                                                                                           |
| Section 2             | Sticky Note                      | OCR Processing block label     | -                        | -                        | ## 2. OCR Processing                                                                                                                  |
| Section 3             | Sticky Note                      | AI Extraction block label      | -                        | -                        | ## 3. AI Extraction                                                                                                                   |
| Section 4             | Sticky Note                      | Output block label             | -                        | -                        | ## 4. Output                                                                                                                          |
| Invoice Upload Form   | Form Trigger                    | User file upload input         | -                        | Split Files              |                                                                                                                                       |
| Split Files           | Code                            | Splits multi-file input        | Invoice Upload Form       | Loop                     | Throws error if no files uploaded                                                                                                    |
| Loop                  | SplitInBatches                  | Controls batch processing      | Split Files               | Prepare OCR Request       |                                                                                                                                       |
| Prepare OCR Request   | Code                            | Base64 conversion & request build | Loop                      | Mistral OCR              |                                                                                                                                       |
| Mistral OCR           | HTTP Request                   | Calls Mistral OCR API          | Prepare OCR Request       | Format OCR Text           | Requires Mistral API Key; possible timeouts                                                                                          |
| Format OCR Text       | Code                            | Extracts and concatenates OCR text | Mistral OCR              | Extract Invoice Fields    |                                                                                                                                       |
| OpenAI Model          | Langchain LM Chat OpenAI        | GPT-4o-mini language model     | Format OCR Text           | Extract Invoice Fields    | Requires OpenAI API Key                                                                                                               |
| Extract Invoice Fields | Langchain Information Extractor | Extracts structured invoice data | OpenAI Model             | Validate & Score          |                                                                                                                                       |
| Validate & Score      | Code                            | Validates fields, assigns confidence | Extract Invoice Fields   | Save to Sheets            |                                                                                                                                       |
| Save to Sheets        | Google Sheets                   | Appends data to Google Sheet   | Validate & Score          | Rate Limit                | Requires Google Sheets OAuth2 credentials; update Sheet ID and columns                                                               |
| Rate Limit            | Wait                           | Prevent API rate limits        | Save to Sheets            | Loop                     | 1-second wait to prevent API throttling                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Invoice Upload Form")**  
   - Type: Form Trigger  
   - Configure form title as "Invoice Upload"  
   - Add a single file upload field labeled "Invoices"  
   - Enable multiple file uploads  
   - Restrict accepted file types to `.pdf, .png, .jpg, .jpeg`  
   - Mark as required  
   - Assign a webhook ID (e.g., "invoice-upload").

2. **Add Code Node ("Split Files")**  
   - Purpose: Split uploaded multiple files into individual workflow items.  
   - JavaScript code: Extract binary files, create one item each with file metadata.  
   - Input: Connect from Invoice Upload Form node.  
   - Output: Multiple items, each with one file's binary and metadata.  
   - Add error handling for no files uploaded.

3. **Add SplitInBatches Node ("Loop")**  
   - Purpose: Batch processing control for invoices.  
   - Connect input from Split Files.  
   - Configure batch size as per expected load or leave default.  
   - Output to next node for OCR preparation.

4. **Add Code Node ("Prepare OCR Request")**  
   - Purpose: Convert binary to base64, build Mistral OCR request body.  
   - JavaScript logic:  
     - Extract binary data and mimeType.  
     - Convert to base64 data URL.  
     - For PDFs or office docs, set document type to 'document_url'; else 'image_url'.  
     - Compose JSON body with model "mistral-ocr-latest".  
   - Input: Connect from Loop node.  
   - Output: JSON ready for OCR request.

5. **Add HTTP Request Node ("Mistral OCR")**  
   - Purpose: Call Mistral OCR API.  
   - URL: `https://api.mistral.ai/v1/ocr`  
   - Method: POST  
   - Body content: JSON from previous node (use expression)  
   - Timeout: 120 seconds  
   - Authentication: HTTP Header Auth with header `Authorization: Bearer YOUR_MISTRAL_API_KEY`  
   - Credentials: Configure Mistral API key credential in n8n.

6. **Add Code Node ("Format OCR Text")**  
   - Purpose: Extract and concatenate OCR text from Mistral response.  
   - Logic:  
     - If pages array present, concatenate markdown text per page.  
     - Count pages.  
     - Fallback to general text or "No text extracted from document".  
   - Input: Connect from Mistral OCR node.  
   - Output: JSON with full OCR text, filename, and page count.

7. **Add Langchain LM Chat OpenAI Node ("OpenAI Model")**  
   - Purpose: Provide GPT-4o-mini language model for extraction.  
   - Model: Set to "gpt-4o-mini"  
   - Credentials: Configure OpenAI API key credential in n8n.  
   - Input: OCR text from Format OCR Text node.

8. **Add Langchain Information Extractor Node ("Extract Invoice Fields")**  
   - Purpose: Extract structured invoice data fields using GPT output.  
   - Configure system prompt with instructions for precise extraction, date and decimal formats.  
   - Define attributes: invoice_number, invoice_date (YYYY-MM-DD), vendor_name, vendor_tax_id, subtotal (number), tax_rate (number), tax_amount (number), total_amount (number), currency.  
   - Input: Connect from OpenAI Model node.

9. **Add Code Node ("Validate & Score")**  
   - Purpose: Validate extracted fields, calculate confidence, assign status and issues.  
   - Logic:  
     - Check required fields (invoice_number, invoice_date, vendor_name, total_amount).  
     - Count how many are filled.  
     - Assign confidence: high (4+), medium (2-3), low (<2).  
     - Compile missing fields as issues.  
     - Attach processed timestamp and page count.  
   - Input: Connect from Extract Invoice Fields node.

10. **Add Google Sheets Node ("Save to Sheets")**  
    - Purpose: Append extracted and validated data into Google Sheet.  
    - Operation: Append  
    - Sheet Name: "Sheet1" (or your target sheet tab)  
    - Document ID: Set your Google Sheet ID (replace placeholder)  
    - Map workflow fields to columns exactly as per required columns: Invoice Number, Invoice Date, Vendor Name, Vendor Tax ID, Subtotal, Tax Rate (%), Tax Amount, Total Amount, Currency, Filename, Confidence, Status, Issues, Processed At, Pages.  
    - Credentials: Use Google Sheets OAuth2 credentials configured in n8n.  
    - Input: Connect from Validate & Score node.

11. **Add Wait Node ("Rate Limit")**  
    - Purpose: Pause 1 second to mitigate Google Sheets API rate limiting during batch writes.  
    - Amount: 1 second  
    - Input: Connect from Save to Sheets node.  
    - Output: Connect back to Loop node to process next batch item.

12. **Add Sticky Notes**  
    - Add descriptive sticky notes for overview, credential warnings, and block labels per the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Create Mistral API Key at https://console.mistral.ai and configure HTTP Header Auth credential in n8n with header `Authorization: Bearer YOUR_TOKEN_HERE`.                                                                       | Critical for OCR API access                                                                     |
| Create OpenAI API Key at https://platform.openai.com and configure OpenAI credentials in n8n.                                                                                                                                     | Needed for GPT-4o-mini AI extraction                                                           |
| Create Google Sheets OAuth2 credentials and prepare a Google Sheet with columns: Invoice Number, Invoice Date, Vendor Name, Vendor Tax ID, Subtotal, Tax Rate (%), Tax Amount, Total Amount, Currency, Filename, Confidence, Status, Issues, Processed At, Pages. | For storing processed invoice data and tracking status                                         |
| The workflow supports multi-page PDFs and multi-file uploads with batch processing and rate limiting to avoid API throttling.                                                                                                   | Best practice for stable operation                                                             |
| Dates are strictly formatted as YYYY-MM-DD and decimal numbers use dots, per AI prompt instructions.                                                                                                                             | Ensures consistency for downstream systems                                                    |
| Confidence scoring logic is customizable and can be extended for additional validation rules or fields.                                                                                                                          | Allows flexibility for quality control                                                         |

---

**Disclaimer:** The text provided derives exclusively from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.