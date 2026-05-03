Extract invoice data from scanned PDFs to Google Sheets with Sarvam and Gemini

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-scanned-pdfs-to-google-sheets-with-sarvam-and-gemini-13779


# Extract invoice data from scanned PDFs to Google Sheets with Sarvam and Gemini

# Workflow Reference: Extract Invoice Data from Scanned PDFs to Google Sheets

This document provides a technical analysis of the n8n workflow designed to automate document digitization using Sarvam AI for OCR and Google Gemini for structured data extraction.

---

### 1. Workflow Overview
This workflow automates the transition from unstructured scanned PDF invoices to structured data in a spreadsheet. It is designed for finance and operations teams to eliminate manual data entry.

**Logical Phases:**
*   **1.1 Input & Job Initialization:** Receives a PDF via a form and creates a processing job in Sarvam AI.
*   **1.2 Document Upload:** Generates a secure presigned URL and uploads the binary PDF data to Sarvam’s servers.
*   **1.3 OCR Processing & Monitoring:** Commands Sarvam to start the OCR process and waits for completion.
*   **1.4 Result Retrieval & Parsing:** Downloads the OCR output (ZIP), decompresses it, and extracts the raw JSON text content.
*   **1.5 AI Information Extraction:** Uses Google Gemini via a LangChain Information Extractor to identify specific fields (Invoice No, GSTIN, etc.) from the raw text.
*   **1.6 Data Storage:** Appends the final structured attributes into a designated Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Step 1 – Upload Invoice to Sarvam
This block handles the entry point and the initial handshake with the Sarvam AI API.
*   **Nodes Involved:** `Trigger – Invoice upload (PDF)`, `Create Sarvam invoice OCR job`, `Generate Sarvam presigned upload URL`, `Merge job details with upload URL`.
*   **Details:**
    *   **Trigger:** An n8n Form Trigger accepting a file field named `Form`.
    *   **Job Creation:** An HTTP Request (POST) to Sarvam to initialize a "generic" document type job.
    *   **Presigned URL:** Requests a specific URL for the uploaded filename to ensure secure data transfer.
    *   **Merge:** Combines the `job_id` from the first request with the `upload_url` from the second to prepare for the binary upload.

#### 2.2 Step 2 – Run OCR and Monitor Status
Handles the actual processing of the document.
*   **Nodes Involved:** `Upload invoice PDF to Sarvam`, `Start Sarvam invoice OCR`, `Check Sarvam OCR status`.
*   **Details:**
    *   **Upload:** Uses a `PUT` method to send the binary PDF to the presigned URL. Requires specific headers: `x-ms-blob-type: BlockBlob`.
    *   **Start OCR:** A POST request that triggers the Sarvam engine to begin analysis on the uploaded file.
    *   **Monitoring:** Retrieves the status of the job.

#### 2.3 Step 3 – Retrieve OCR Output
Retrieves the processed data once the AI has finished reading the document.
*   **Nodes Involved:** `Wait`, `Get OCR Result`, `Download Sarvam OCR output ZIP`, `Decompress OCR result file`, `Extract invoice OCR JSON`.
*   **Details:**
    *   **Wait:** A 2-minute delay to allow the Sarvam engine sufficient time to process complex PDFs.
    *   **Extraction:** The workflow downloads a ZIP file, decompresses it to find `file_1` (the JSON result), and parses it.
    *   **Failure points:** If the document is large and takes >2 mins, the "Get OCR Result" might fail if the job is not yet finished.

#### 2.4 Step 4 – Convert OCR Text to Structured Fields
Cleans the raw machine-read text and uses LLM intelligence to find specific values.
*   **Nodes Involved:** `Prepare invoice text for LLM`, `Information Extractor`, `Google Gemini Chat Model`.
*   **Details:**
    *   **Prepare Text (Code):** A JavaScript snippet that iterates through the OCR JSON blocks and concatenates all `block.text` values into a single string.
    *   **Information Extractor:** A LangChain tool using a System Prompt to strictly extract: Invoice No, Patient Name, GSTIN, Product Names, and Date.
    *   **Gemini Model:** Provides the reasoning capabilities to distinguish between different dates or numbers on a page.

#### 2.5 Step 5 – Store Invoice Data
The final destination for the processed information.
*   **Nodes Involved:** `Append extracted invoice data to Google Sheets`.
*   **Details:**
    *   **Configuration:** Maps the AI-extracted JSON keys to Google Sheet columns: `Date`, `GSTIN/UIN`, `Invoice No.`, `Patient Name`, and `Description of Good`.
    *   **Operation:** `Append or Update` based on the GSTIN/UIN column to prevent duplicate entries for the same entity if applicable.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger – Invoice upload (PDF) | Form Trigger | Workflow entry point | None | Create Sarvam job, Merge | Step 1 – Upload invoice to Sarvam |
| Create Sarvam invoice OCR job | HTTP Request | Init API Job | Trigger | Generate URL | Step 1 – Upload invoice to Sarvam |
| Generate Sarvam presigned URL | HTTP Request | Auth for upload | Create Sarvam job | Merge | Step 1 – Upload invoice to Sarvam |
| Merge job details... | Merge | Data prep | Trigger, Generate URL | Upload PDF | Step 1 – Upload invoice to Sarvam |
| Upload invoice PDF to Sarvam | HTTP Request | Binary transfer | Merge | Start OCR | Step 2 – Run OCR and monitor status |
| Start Sarvam invoice OCR | HTTP Request | Start processing | Upload PDF | Check status | Step 2 – Run OCR and monitor status |
| Check Sarvam OCR status | HTTP Request | Status check | Start OCR | Wait | Step 2 – Run OCR and monitor status |
| Wait | Wait | Delay | Check status | Get OCR Result | Step 3 – Retrieve OCR output |
| Get OCR Result | HTTP Request | Finalize metadata | Wait | Download ZIP | Step 3 – Retrieve OCR output |
| Download Sarvam output ZIP | HTTP Request | File download | Get OCR Result | Decompress | Step 3 – Retrieve OCR output |
| Decompress OCR result file | Compression | Unzip output | Download ZIP | Extract JSON | Step 3 – Retrieve OCR output |
| Extract invoice OCR JSON | Extract From File | JSON Parsing | Decompress | Prepare text | Step 3 – Retrieve OCR output |
| Prepare invoice text for LLM | Code | Text cleaning | Extract JSON | Information Extractor | Step 4 – Convert OCR text to structured... |
| Information Extractor | LangChain | AI Field Mapping | Prepare text | Google Sheets | Step 4 – Convert OCR text to structured... |
| Google Gemini Chat Model | Gemini Chat | AI Reasoning | Information Extractor | Information Extractor | Step 4 – Convert OCR text to structured... |
| Append to Google Sheets | Google Sheets | Data Storage | Information Extractor | None | Step 5 – Store invoice data |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Form Trigger** node. Add a field named `Form` with the type `File`.
2.  **Sarvam Job Initialization:**
    *   Add an **HTTP Request** node (POST) to `https://api.sarvam.ai/doc-digitization/job/v1`.
    *   Body: `{"job_parameters": {"document_type": "generic"}}`.
    *   Auth: Header Auth with your Sarvam API Key.
3.  **Presigned URL:**
    *   Add an **HTTP Request** node (POST) to `.../upload-files`.
    *   Body: Include the `job_id` from step 2 and the filename from the Form Trigger.
4.  **Data Merging:** Use a **Merge** node (Combine by Position) to bring the binary data from the Trigger and the `upload_url` from the previous step together.
5.  **Binary Upload:**
    *   Add an **HTTP Request** node (PUT). 
    *   URL: Expression pointing to `upload_url`.
    *   Send Binary Data: `True`. Input Data Field Name: `Form`.
    *   Headers: `Content-Type: application/pdf`, `x-ms-blob-type: BlockBlob`.
6.  **OCR Execution:**
    *   Add an **HTTP Request** (POST) to `.../job/v1/{{job_id}}/start`.
    *   Add a **Wait** node set to 2 minutes.
7.  **File Retrieval:**
    *   Add **HTTP Request** to download the ZIP, followed by a **Compression** node (Decompress).
    *   Add **Extract From File** node using the `fromJson` operation on the decompressed binary.
8.  **AI Configuration:**
    *   Add a **Code** node to loop through `$json.data.blocks` and join the `text` fields.
    *   Add the **Information Extractor** node. Define the schema for "Invoice No.", "Patient Name", "GSTIN", "PRODUCT NAME", and "Date".
    *   Connect the **Google Gemini Chat Model** to the Information Extractor.
9.  **Google Sheets:**
    *   Add the **Google Sheets** node. Select the `Append or Update` operation.
    *   Map the output of the Information Extractor to your specific column headers.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Sarvam AI Key Management | [https://dashboard.sarvam.ai/key-management](https://dashboard.sarvam.ai/key-management) |
| n8n Community Support | [https://community.n8n.io/](https://community.n8n.io/) |
| Sarvam Official Logo | [Image Link](https://i.ibb.co/cqhn57z/Image-25-02-26-at-4-57-PM.png) |
| Target Use Cases | Operations, Finance, and Accounting teams for vendor invoice processing. |