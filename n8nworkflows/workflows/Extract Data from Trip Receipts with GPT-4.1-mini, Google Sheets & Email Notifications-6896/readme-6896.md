Extract Data from Trip Receipts with GPT-4.1-mini, Google Sheets & Email Notifications

https://n8nworkflows.xyz/workflows/extract-data-from-trip-receipts-with-gpt-4-1-mini--google-sheets---email-notifications-6896


# Extract Data from Trip Receipts with GPT-4.1-mini, Google Sheets & Email Notifications

---

## 1. Workflow Overview

This workflow automates the process of submitting, extracting, and reporting business trip expenses from uploaded receipts/invoices. It is designed to streamline expense claims for employees, enabling automatic data extraction from PDFs, centralized record-keeping in Google Sheets, and notifying the finance team via email.

### Logical Blocks:

- **1.1 Input Reception & File Handling**:  
  Captures employee trip details and multiple PDF receipts via a form submission, then separates and processes each uploaded file individually.

- **1.2 File Storage & Data Extraction**:  
  Uploads each PDF to Google Drive and extracts structured invoice data using an AI agent powered by GPT-4.1-mini.

- **1.3 Data Transformation & Logging**:  
  Transforms the extracted data and employee info into structured records, then appends these records to a Google Sheet for tracking.

- **1.4 Email Generation & Notification**:  
  Generates a detailed HTML email summarizing the trip and all expenses, and sends it to the finance department.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & File Handling

**Overview:**  
Receives form submissions with employee and trip details along with multiple PDF receipts. Parses the multipart file upload into individual files for processing.

**Nodes Involved:**  
- `On form submission`  
- `Handle multiple files`

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing employee name, department (dropdown), trip purpose, trip dates, and multiple PDF receipt uploads.  
  - *Configuration:* Form fields require all inputs; file upload only accepts PDFs; form description guides employees on submission.  
  - *Input/Output:* Outputs JSON form data with binary PDFs.  
  - *Edge Cases:* Missing required fields cause form validation failure; non-PDF files rejected.  
  - *Notes:* Webhook ID enabled for external form triggers.

- **Handle multiple files**  
  - *Type:* Code Node  
  - *Role:* Extracts multiple uploaded PDF files from the form submission’s binary data into separate workflow items for parallel processing.  
  - *Configuration:* Iterates over binary keys starting with "Receipts___Invoices_" and outputs one item per file with JSON metadata and binary data.  
  - *Input/Output:* Receives one item with multiple files; outputs multiple items each with one file.  
  - *Edge Cases:* If no files match naming pattern, outputs empty array; expression errors if binary data missing.

---

### 2.2 File Storage & Data Extraction

**Overview:**  
Saves each uploaded PDF to a designated Google Drive folder and extracts structured expense data using a GPT-4.1-mini-powered AI agent that parses invoice content.

**Nodes Involved:**  
- `Upload file`  
- `Extract from File`  
- `DocClaim Assistant Agent`  
- `Structured Output Parser`

**Node Details:**

- **Upload file**  
  - *Type:* Google Drive Node  
  - *Role:* Uploads each PDF receipt to a specific Google Drive folder for archival and reference.  
  - *Configuration:* Filename includes timestamp and original filename for uniqueness. Folder is preselected (ID: `1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI`).  
  - *Input/Output:* Receives one binary PDF; outputs file metadata including webContentLink for download.  
  - *Credentials:* Requires Google Drive OAuth2 credentials with upload permissions.  
  - *Edge Cases:* Upload failures due to permission issues or rate limits.

- **Extract from File**  
  - *Type:* Extract From File Node  
  - *Role:* Processes uploaded PDF to extract raw text content for AI processing.  
  - *Configuration:* Operation set to "pdf" for text extraction.  
  - *Input/Output:* Receives binary PDF; outputs extracted text as JSON.  
  - *Edge Cases:* PDFs with scanned images only (no text layer) may yield poor extraction; extraction timeout possible on large files.

- **DocClaim Assistant Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Uses GPT-4.1-mini with a specialized prompt to parse raw invoice text into structured JSON with fields like vendor name, invoice number, dates, amounts, currency, payment method, and itemized lines.  
  - *Configuration:* System message defines role as document extraction assistant specialized in receipts/invoices; prompt includes raw text with instruction to output structured data in JSON format.  
  - *Input/Output:* Receives JSON text from extraction; outputs structured JSON under `.output`.  
  - *Credentials:* OpenAI API with GPT-4.1-mini model.  
  - *Edge Cases:* Incomplete or poorly formatted invoices may result in partial or blank fields; API rate limits or timeouts; parsing errors.  

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Validates and enforces the structured JSON schema for the AI output, ensuring consistent field names and types.  
  - *Configuration:* JSON schema example provided to define expected fields and nested structure.  
  - *Input/Output:* Connected to DocClaim Assistant Agent’s output parser interface; ensures structured data correctness.  
  - *Edge Cases:* Parsing failure if AI output deviates from schema; missing fields accepted as blank.

---

### 2.3 Data Transformation & Logging

**Overview:**  
Transforms the AI-extracted invoice data together with employee and trip details into a unified JSON object. Then logs each invoice record with metadata to a Google Sheet for centralized tracking.

**Nodes Involved:**  
- `Transform Output`  
- `Transform invoice record`  
- `Append row in sheet`

**Node Details:**

- **Transform Output**  
  - *Type:* Code Node  
  - *Role:* Aggregates employee data from form submission and all AI-extracted expense records into a structured JSON object for downstream use.  
  - *Configuration:* Reads employee info from form submission; maps over all AI output items to extract key invoice fields, applying defaults for missing data; packages results as `{ employee, expenses }`.  
  - *Input/Output:* Inputs multiple AI extraction items; outputs one combined JSON item.  
  - *Edge Cases:* Missing or malformed AI output fields handled by defaulting to empty or zero values.

- **Transform invoice record**  
  - *Type:* Code Node (run once per item)  
  - *Role:* Enriches each invoice record with employee and trip metadata plus Google Drive file info, preparing it for sheet insertion.  
  - *Configuration:* Maps fields like employee name, department, trip details, file name, download URL, file size, and submission timestamp from form and upload nodes.  
  - *Input/Output:* Receives uploaded file metadata; outputs mapped record for Google Sheets.  
  - *Edge Cases:* Missing metadata fields could cause blank cells.

- **Append row in sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends each transformed invoice record as a new row into a designated Google Sheet for expense tracking.  
  - *Configuration:* Auto-maps input JSON fields to predefined columns in sheet named "Sheet1" in document with ID `1qk6OebcuZkIRorf1k235ew88oZ-UlUJSyiHFqZYDbaU`.  
  - *Input/Output:* Takes single transformed record; outputs Google Sheets append response.  
  - *Credentials:* Google Sheets OAuth2 credentials with write access.  
  - *Edge Cases:* Sheet API rate limits, permission errors, or schema mismatches.

---

### 2.4 Email Generation & Notification

**Overview:**  
Creates a formatted, professional HTML email summarizing the employee’s trip and all extracted expenses, then sends this summary to the finance team for claim processing.

**Nodes Involved:**  
- `Create HTML Email Template`  
- `Send trip expense request to finance team`

**Node Details:**

- **Create HTML Email Template**  
  - *Type:* Code Node  
  - *Role:* Constructs an HTML email body combining employee info and detailed expense tables including summary rows and itemized breakdowns per expense.  
  - *Configuration:* Uses mapped JSON from `Transform Output`; generates HTML with inline styles for readability, including tables for expense summary and detailed line items.  
  - *Input/Output:* Receives combined `{ employee, expenses }`; outputs `{ html }` JSON.  
  - *Edge Cases:* Missing or empty expense items gracefully handled; formatting errors possible if data contains unexpected values.

- **Send trip expense request to finance team**  
  - *Type:* Email Send Node  
  - *Role:* Sends the generated HTML email to the finance department’s email address for reimbursement processing.  
  - *Configuration:* Subject dynamically composed with employee name, department, and trip purpose. Uses SMTP credentials configured.  
  - *Input/Output:* Receives HTML content from previous node; sends email with no output.  
  - *Credentials:* SMTP account with sending permissions.  
  - *Edge Cases:* Email delivery failures due to SMTP issues; invalid recipient address; large email size if many receipts included.

---

## 3. Summary Table

| Node Name                         | Node Type                        | Functional Role                                    | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                                                                                                                                                                                             |
|----------------------------------|---------------------------------|---------------------------------------------------|------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note7                     | Sticky Note                     | Overview and detailed description of workflow     |                              |                                     | # 🧾 Automated Trip Expense Reporting Workflow With gpt-4.1-mini ... (full content describing purpose, users, setup, customization, and requirements)                                                                                                                    |
| Sticky Note3                     | Sticky Note                     | Visual screenshot illustration                      |                              |                                     | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-02+at+10.06.31%E2%80%AFPM.png)                                                                                                                                                    |
| Sticky Note5                     | Sticky Note                     | Explains AI invoice extraction step                 |                              |                                     | ## 1.1 – Extract and Parse Invoices with AI Agent ...                                                                                                                                                                                                                   |
| Sticky Note                      | Sticky Note                     | Explains email notification step                     |                              |                                     | ## 3. Send Claim Summary to Finance Team ...                                                                                                                                                                                                                            |
| Sticky Note6                     | Sticky Note                     | Screenshot related to email generation               |                              |                                     | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-02+at+10.01.32%E2%80%AFPM.png)                                                                                                                                                    |
| Sticky Note9                     | Sticky Note                     | Explains Google Sheet storage step                   |                              |                                     | ## 1.2. Store Invoices in Google Sheet ...                                                                                                                                                                                                                              |
| Sticky Note10                    | Sticky Note                     | Explains email HTML generation                        |                              |                                     | ## 2. Transform and Generate HTML Email ...                                                                                                                                                                                                                            |
| Sticky Note8                     | Sticky Note                     | Screenshot related to email sending                   |                              |                                     | ![Alt text](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-02+at+10.00.53%E2%80%AFPM.png)                                                                                                                                                    |
| On form submission              | Form Trigger                   | Captures employee/trip details and PDF uploads      |                              | Handle multiple files                |                                                                                                                                                                                                                                                                         |
| Handle multiple files           | Code                          | Splits multiple uploaded PDFs into individual items | On form submission           | Upload file, Extract from File       |                                                                                                                                                                                                                                                                         |
| Upload file                    | Google Drive                  | Saves PDF receipts to Google Drive                   | Handle multiple files         | Transform invoice record             |                                                                                                                                                                                                                                                                         |
| Extract from File              | Extract From File             | Extracts raw text from PDFs                           | Handle multiple files         | DocClaim Assistant Agent             |                                                                                                                                                                                                                                                                         |
| DocClaim Assistant Agent       | LangChain Agent              | AI parses invoice text into structured JSON          | Extract from File            | Transform Output                     |                                                                                                                                                                                                                                                                         |
| Structured Output Parser       | LangChain Output Parser      | Validates and formats AI output JSON                  |                             | DocClaim Assistant Agent             |                                                                                                                                                                                                                                                                         |
| Transform Output              | Code                          | Aggregates employee and expense data                  | DocClaim Assistant Agent      | Create HTML Email Template           |                                                                                                                                                                                                                                                                         |
| Transform invoice record      | Code                          | Enriches invoice data with metadata for sheet logging | Upload file                  | Append row in sheet                  |                                                                                                                                                                                                                                                                         |
| Append row in sheet           | Google Sheets                | Logs expense records in Google Sheet                   | Transform invoice record      |                                     |                                                                                                                                                                                                                                                                         |
| Create HTML Email Template    | Code                          | Builds detailed HTML email from expense data          | Transform Output             | Send trip expense request to finance team |                                                                                                                                                                                                                                                                         |
| Send trip expense request to finance team | Email Send                   | Sends claim summary email to finance                    | Create HTML Email Template   |                                     |                                                                                                                                                                                                                                                                         |
| GPT                          | LangChain LM Chat OpenAI     | GPT-4.1-mini model placeholder (not connected)        |                             | DocClaim Assistant Agent (ai_languageModel) |                                                                                                                                                                                                                                                                         |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission")**  
   - Use `Form Trigger` type.  
   - Form title: "Business Trip Claim Submission".  
   - Fields:  
     - Employee Name (text, required)  
     - Department (dropdown: Sales, Engineering, HR, Executive, required)  
     - Trip Purpose (text, required)  
     - From Date (date, required)  
     - To Date (date, required)  
     - Receipts / Invoices (file upload, multiple, accept only `.pdf`, required)  
   - Enable webhook and save.

2. **Add a Code Node ("Handle multiple files")**  
   - Purpose: Split multiple uploaded PDFs into separate items.  
   - JS Code:  
     ```javascript
     const data = $input.item.json;
     const binaryData = $input.item.binary;
     let output = [];
     Object.keys(binaryData)
       .filter(label => label.startsWith("Receipts___Invoices_"))
       .forEach(label => {
         output.push({
           json: data,
           binary: { data: binaryData[label] }
         });
       });
     return output;
     ```  
   - Connect `On form submission` → `Handle multiple files`.

3. **Add a Google Drive node ("Upload file")**  
   - Operation: Upload file.  
   - Set file name: `invoice-{{ $now.toFormat("yyyyLLdd-HHmmss") }}-{{$binary.data.fileName}}`.  
   - Select specific Drive folder by folder ID (e.g., "1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI").  
   - Attach Google Drive OAuth2 credentials with upload permission.  
   - Connect `Handle multiple files` → `Upload file`.

4. **Add an Extract From File node ("Extract from File")**  
   - Operation: PDF text extraction.  
   - Connect `Handle multiple files` → `Extract from File`.  
   - No special credentials needed.

5. **Add a LangChain Agent node ("DocClaim Assistant Agent")**  
   - Model: Use the GPT-4.1-mini model or equivalent.  
   - System Message: Define assistant as an invoice/receipt parser extracting fields such as vendor, invoice/receipt numbers, dates, amounts, tax, currency, payment method, and itemized description; output structured JSON only.  
   - Prompt: Include extracted text with instructions to output structured data.  
   - Enable output parser.  
   - Connect `Extract from File` → `DocClaim Assistant Agent`.  
   - Attach OpenAI API credentials with access to GPT-4.1-mini.

6. **Add a LangChain Structured Output Parser node ("Structured Output Parser")**  
   - Paste the JSON schema example defining all expected fields (expense_type, vendor_name, invoice_number, etc.).  
   - Connect the output parser interface of `DocClaim Assistant Agent` → `Structured Output Parser`.

7. **Add a Code node ("Transform Output")**  
   - Purpose: Aggregate employee info and extract all expense JSON outputs into a single combined JSON object.  
   - Code reads form submission data, maps over AI outputs, and applies default values where fields are missing.  
   - Connect `DocClaim Assistant Agent` → `Transform Output`.

8. **Add a Code node ("Transform invoice record")**  
   - Purpose: For each uploaded file, enrich with employee/trip metadata and upload file info for sheet logging.  
   - Runs once per item.  
   - Reads from `On form submission` for employee details, and from `Upload file` for file metadata.  
   - Connect `Upload file` → `Transform invoice record`.

9. **Add a Google Sheets node ("Append row in sheet")**  
   - Operation: Append row to Google Sheet.  
   - Document: Use a shared Google Sheet ID for expense claims.  
   - Sheet name: "Sheet1" or equivalent.  
   - Field mapping: Map employee fields, trip details, file name, download link, file size, and submission timestamp.  
   - Attach Google Sheets OAuth2 credentials with write permission.  
   - Connect `Transform invoice record` → `Append row in sheet`.

10. **Add a Code node ("Create HTML Email Template")**  
    - Purpose: Generate a formatted HTML email combining employee info and detailed expense summaries, including tables with itemized lines.  
    - Connect `Transform Output` → `Create HTML Email Template`.

11. **Add an Email Send node ("Send trip expense request to finance team")**  
    - Use SMTP or Gmail credentials configured.  
    - Set subject dynamically: `"Expense Claim Request - {{ employee.name }} – {{ employee.department }} - {{ employee.tripPurpose }}"`.  
    - Use HTML content from previous node as email body.  
    - Set recipient as the finance department’s email address.  
    - Connect `Create HTML Email Template` → `Send trip expense request to finance team`.

12. **Optional:** Add Sticky Notes for documentation and screenshots at appropriate positions for clarity.

13. **Activate and test the workflow.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed for employees submitting business trip expense claims, with automated extraction, logging, and notification. It requires proper Google Drive and Sheets API credentials and OpenAI API access to GPT-4.1-mini. | Workflow overview sticky note at start of workflow.                                                                       |
| Screenshots linked in sticky notes show UI and configuration examples to help visualize the flow.                                                                                                                                                                                                   | Sticky Note3, Sticky Note6, Sticky Note8 with image URLs.                                                                 |
| To customize, consider adding approval steps, attaching original PDFs to emails, localization, integration with ERP/accounting, or enforcing validation rules.                                                                                                                                      | Overview sticky note section "How to customize the workflow".                                                              |
| For more info on n8n form triggers and Google integration, visit official n8n docs: https://docs.n8n.io/nodes/n8n-nodes-base.formtrigger/ and https://docs.n8n.io/nodes/n8n-nodes-base.googledrive/                                                                                               | General n8n node documentation.                                                                                            |
| The DocClaim Assistant Agent leverages GPT-4.1-mini specialized prompt engineering for document parsing; the structured output parser ensures reliable data schema conformance to avoid downstream errors.                                                                                          | Node descriptions and prompt examples in Block 2.2.                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.