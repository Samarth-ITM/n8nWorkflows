Parse Invoices & Documents with Gemini AI, OCR, and Google Sheets Integration

https://n8nworkflows.xyz/workflows/parse-invoices---documents-with-gemini-ai--ocr--and-google-sheets-integration-7239


# Parse Invoices & Documents with Gemini AI, OCR, and Google Sheets Integration

### 1. Workflow Overview

This workflow automates the extraction of structured invoice or document data from various file formats (PDF, image, CSV) uploaded via a webhook. It uses Optical Character Recognition (OCR) and Google Gemini AI to parse raw text into structured JSON data, which is then appended or updated into a Google Sheets spreadsheet. The workflow is designed to handle invoices, logs, sensor reports, or similar documents, enabling AI-powered automation of manual data entry.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception**: Receives uploaded documents via webhook.
- **1.2 File Type Detection**: Determines the format of the uploaded file (image, PDF, or CSV).
- **1.3 Text Extraction**: Extracts raw text using OCR for images, PDF parsing for PDFs, or direct JSON conversion for CSV files.
- **1.4 AI Processing**: Uses Google Gemini AI to extract key invoice fields from the raw text.
- **1.5 JSON Transformation**: Cleans and transforms AI output into a valid JSON object.
- **1.6 Data Storage**: Appends or updates the parsed invoice data into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow upon receiving a file upload via an HTTP POST webhook.

**Nodes Involved:**  
- Webhook Invoice upload

**Node Details:**

- **Webhook Invoice upload**
  - Type: Webhook (Trigger)
  - Configuration: Listens to POST requests on path `/uploadDoc`.
  - Input: Receives uploaded files in binary form with mimeType metadata.
  - Output: Passes the received file data to the next node.
  - Edge Cases: Invalid HTTP methods, missing files in request, large file uploads causing timeouts.
  - Version: n8n v2+ recommended for webhook stability.

---

#### 2.2 File Type Detection

**Overview:**  
Detects the MIME type of the uploaded file and routes the workflow accordingly for specialized parsing.

**Nodes Involved:**  
- Check file type (Switch node)

**Node Details:**

- **Check file type**
  - Type: Switch node
  - Configuration: Checks the MIME type of the uploaded file from the webhook.
    - Routes to "Image to Text" if MIME type contains "image".
    - Routes to "PDF to Text" if MIME type equals "application/pdf".
    - Routes to "CSV to JSON" if MIME type contains "csv".
  - Input: Binary file and its MIME type from webhook.
  - Output: Routes to one of three nodes based on detected file type.
  - Edge Cases: Unsupported MIME types, incorrect MIME detection, empty or corrupted files.

---

#### 2.3 Text Extraction

**Overview:**  
Extracts raw text content from the file depending on its type: OCR for images, PDF text extraction, or CSV parsing.

**Nodes Involved:**  
- Image to Text  
- PDF to Text  
- CSV to JSON  

**Node Details:**

- **Image to Text**
  - Type: Tesseract OCR node
  - Configuration: Uses Tesseract.js to extract text from image binary data.
  - Input: Binary image from "Check file type".
  - Output: Extracted raw text.
  - Edge Cases: Poor image quality causing OCR inaccuracies, unsupported image formats.

- **PDF to Text**
  - Type: Extract from File (PDF)
  - Configuration: Extracts text content from PDF binary.
  - Input: PDF binary from "Check file type".
  - Output: Extracted raw text.
  - Edge Cases: Encrypted or scanned PDFs without embedded text, large PDFs causing timeouts.

- **CSV to JSON**
  - Type: Extract from File (CSV)
  - Configuration: Converts CSV binary to JSON.
  - Input: CSV file from "Check file type".
  - Output: JSON array representing CSV rows.
  - Edge Cases: Malformed CSV, encoding issues.

---

#### 2.4 AI Processing

**Overview:**  
Sends extracted raw text to Google Gemini AI model for parsing out structured invoice fields.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Format data from text

**Node Details:**

- **Google Gemini Chat Model**
  - Type: Langchain Google Gemini Chat model node
  - Configuration: Uses model "models/gemini-1.5-flash" to process input text.
  - Input: Raw text from OCR or PDF extraction nodes.
  - Output: AI-generated response containing extracted invoice data in JSON-like text.
  - Credentials: Requires Google Palm API credentials.
  - Edge Cases: API authentication failure, rate limits, malformed or ambiguous input text.

- **Format data from text**
  - Type: Langchain Chain LLM node
  - Configuration: Applies a prompt to extract specific invoice fields (invoice_id, dates, names, totals, currency) from input text and return a strictly formatted JSON object.
  - Input: Text from OCR/PDF extraction or Google Gemini output.
  - Output: Text formatted as JSON string.
  - Edge Cases: Missing fields, AI returning invalid JSON or extra text, language model errors.

---

#### 2.5 JSON Transformation

**Overview:**  
Cleans AI output text and parses it into valid JSON for further processing.

**Nodes Involved:**  
- Transfrom data (Code node)

**Node Details:**

- **Transfrom data**
  - Type: Code node (JavaScript)
  - Configuration: Strips markdown code block syntax (```json ... ```) from AI output, then parses cleaned string into JSON.
  - Input: Text from "Format data from text".
  - Output: JSON object with invoice fields.
  - Edge Cases: JSON.parse failures due to malformed AI output, missing data fields.
  - Version: Requires JavaScript environment supporting ES6+.

---

#### 2.6 Data Storage

**Overview:**  
Appends or updates the parsed invoice data into Google Sheets.

**Nodes Involved:**  
- Invoice data  
- Invoice Data

**Node Details:**

- **Invoice data**
  - Type: Google Sheets node
  - Configuration: Appends or updates rows in the "Invoice" sheet using the parsed invoice JSON fields.
  - Mapping: Maps invoice fields like invoice_id, invoice_date, total, vendor_name, customer_name, subtotal, tax_total, currency.
  - Matching: Uses "invoice_id" column to detect existing rows for update.
  - Credentials: Requires Google Sheets OAuth2 credentials.
  - Edge Cases: Credential expiry, sheet access permission errors, API quota limits.

- **Invoice Data**
  - Type: Google Sheets node (duplicate of above)
  - Configuration: Same as "Invoice data", appends or updates data in the same Google Sheet.
  - Input: JSON from "Transfrom data" node.
  - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name             | Node Type                       | Functional Role              | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                  |
|-----------------------|--------------------------------|-----------------------------|----------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Webhook Invoice upload | Webhook                        | Input reception             | —                          | Check file type          |                                                                                              |
| Check file type        | Switch                        | File type detection          | Webhook Invoice upload      | Image to Text, PDF to Text, CSV to JSON |                                                                                              |
| Image to Text          | Tesseract OCR                 | Text extraction (image OCR)  | Check file type             | Format data from text    |                                                                                              |
| PDF to Text            | Extract from File (PDF)       | Text extraction (PDF)        | Check file type             | Format data from text    |                                                                                              |
| CSV to JSON            | Extract from File (CSV)       | Text extraction (CSV to JSON)| Check file type             | Invoice data             |                                                                                              |
| Google Gemini Chat Model | Langchain Google Gemini AI   | AI processing of text        | Format data from text (ai_languageModel) | Format data from text  |                                                                                              |
| Format data from text  | Langchain Chain LLM           | Format AI output text        | Image to Text, PDF to Text, Google Gemini Chat Model | Transfrom data          |                                                                                              |
| Transfrom data         | Code                         | Clean and parse JSON         | Format data from text       | Invoice Data             |                                                                                              |
| Invoice data           | Google Sheets                 | Append/update sheet (CSV path) | CSV to JSON                | —                       |                                                                                              |
| Invoice Data           | Google Sheets                 | Append/update sheet (AI JSON) | Transfrom data             | —                       |                                                                                              |
| Sticky Note            | Sticky Note                  | Documentation                | —                          | —                       | Purpose and core logic overview                                                             |
| Sticky Note1           | Sticky Note                  | Documentation                | —                          | —                       | Workflow title: Smart Document Parser for Invoices, Logs or Sensor Reports (PDF/Image/csv to Sheet) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `uploadDoc`  
   - Method: POST  
   - Purpose: Receive uploaded files.

2. **Create Switch Node for File Type Detection**  
   - Type: Switch  
   - Input: MIME type from webhook binary file  
   - Conditions:  
     - If MIME type contains "image" → route to "Image to Text"  
     - If MIME type equals "application/pdf" → route to "PDF to Text"  
     - If MIME type contains "csv" → route to "CSV to JSON"

3. **Create Extract Text Nodes**  
   - **Image to Text**: Tesseract OCR  
     - Input: binary file from switch  
     - Output: extracted text  
   - **PDF to Text**: Extract from File (PDF)  
     - Input: binary PDF from switch  
     - Output: extracted text  
   - **CSV to JSON**: Extract from File (CSV)  
     - Input: binary CSV from switch  
     - Output: JSON array

4. **Create Google Gemini Chat Model Node**  
   - Use Langchain Google Gemini AI integration  
   - Model: `models/gemini-1.5-flash`  
   - Credentials: Google Palm API OAuth2  
   - Input: extracted text (from Image to Text or PDF to Text)  
   - Output: AI response text

5. **Create Format Data from Text Node (Langchain Chain LLM)**  
   - Prompt: Extract invoice fields from input text and return JSON with keys: invoice_id, invoice_date, due_date, customer_name, vendor_name, subtotal, tax_total, total, currency  
   - Input: raw extracted text or AI output text  
   - Output: JSON string (may include markdown code block formatting)

6. **Create Code Node to Transform Data**  
   - JavaScript code:  
     ```javascript
     const raw = $input.first().json.text || '';
     const cleaned = raw.replace(/```json|```/g, '').trim();
     const parsed = JSON.parse(cleaned);
     return [{ json: parsed }];
     ```  
   - Input: output from "Format data from text"  
   - Output: JSON object with invoice fields

7. **Create Google Sheets Append/Update Nodes**  
   - Two nodes (or one, depending on routing):  
     - For CSV input: connect "CSV to JSON" → "Invoice data"  
     - For AI JSON output: connect "Transfrom data" → "Invoice Data"  
   - Configure both to append or update rows in the "Invoice" sheet  
   - Map columns: invoice_id, invoice_date, due_date, customer_name, vendor_name, subtotal, tax_total, total, currency  
   - Use "invoice_id" as matching column for updates  
   - Credentials: Google Sheets OAuth2

8. **Connect nodes according to file type routing:**  
   - Webhook → Check file type → Image to Text → Google Gemini Chat Model → Format data from text → Transfrom data → Invoice Data  
   - Webhook → Check file type → PDF to Text → Google Gemini Chat Model → Format data from text → Transfrom data → Invoice Data  
   - Webhook → Check file type → CSV to JSON → Invoice data

9. **Add Sticky Notes for documentation as desired.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Purpose: To automatically extract structured data from uploaded documents (PDFs, images, or CSVs) such as invoices and store them in Google Sheets using AI and OCR.                                                                                                                                                                                               | Sticky Note node content                          |
| Core Logic: Webhook trigger → File type detection → Text extraction (OCR/PDF/CSV) → Google Gemini AI for data extraction → JSON formatting → Append/update Google Sheets.                                                                                                                                                                                            | Sticky Note node content                          |
| Outcome: Enables automation of manual data entry workflows across multiple document formats with AI-powered parsing.                                                                                                                                                                                                                                                | Sticky Note node content                          |
| Workflow title: Smart Document Parser for Invoices, Logs or Sensor Reports (PDF/Image/csv to Sheet)                                                                                                                                                                                                                                                                  | Sticky Note1 node content                         |
| Google Gemini AI model used: `models/gemini-1.5-flash`. Requires valid Google Palm API credentials.                                                                                                                                                                                                                                                                 | Node details / Credentials                        |
| Google Sheets appendOrUpdate operation uses invoice_id to avoid duplicate rows.                                                                                                                                                                                                                                                                                      | Node details                                     |
| Tesseract OCR node may require additional setup or language data files depending on document language.                                                                                                                                                                                                                                                             | Node details                                     |
| For robust production, consider adding error handling for API failures, empty or corrupted files, and invalid AI output JSON parsing.                                                                                                                                                                                                                               | Recommended best practice                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.