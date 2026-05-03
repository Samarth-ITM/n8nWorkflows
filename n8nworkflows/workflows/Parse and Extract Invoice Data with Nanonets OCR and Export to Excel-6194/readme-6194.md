Parse and Extract Invoice Data with Nanonets OCR and Export to Excel

https://n8nworkflows.xyz/workflows/parse-and-extract-invoice-data-with-nanonets-ocr-and-export-to-excel-6194


# Parse and Extract Invoice Data with Nanonets OCR and Export to Excel

### 1. Workflow Overview

This workflow automates the extraction of structured data from invoice documents using the Nanonets OCR API and exports the processed data to an Excel file. It is designed for users who need to parse invoices or similar documents programmatically, extract key fields including line items, and receive the results as a downloadable Excel spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts invoice files via an HTTP webhook for programmatic uploads.
- **1.2 Nanonets Document Submission:** Sends the uploaded file to the Nanonets API for OCR processing.
- **1.3 Processing Delay:** Waits for the asynchronous Nanonets OCR processing to complete.
- **1.4 Fetching Processed Data:** Retrieves the processed document data from Nanonets.
- **1.5 Data Parsing and Structuring:** Parses the raw JSON response, extracting main document fields and detailed line items.
- **1.6 Export to Excel:** Converts the structured JSON data into an Excel file.
- **1.7 Response Delivery:** Sends back the Excel file as the HTTP response to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives PDF or image files uploaded via a POST HTTP webhook, triggering the workflow and providing the raw document data for OCR processing.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Configuration: Listens for POST requests at a unique path; expects multipart form data containing the document binary in the `data` field.  
    - Expressions: None.  
    - Input: External HTTP POST request with file data.  
    - Output: Passes binary file data to next node.  
    - Edge Cases: Missing or malformed file upload; HTTP errors; large file uploads exceeding limits.  
    - Version Requirements: n8n version supporting webhook node v2 (standard).

#### 1.2 Nanonets Document Submission

- **Overview:**  
  Sends the uploaded file to the Nanonets API for OCR processing by creating a new document in the specified workflow.

- **Nodes Involved:**  
  - HTTP Request (named "HTTP Request")  
  - Code (named "Code")

- **Node Details:**

  - **HTTP Request ("HTTP Request")**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: `https://app.nanonets.com/api/v4/workflows/28f99e34-2bfb-4d50-8ac3-17eaa5ee2ecf/documents/`  
      - Authentication: HTTP Basic Auth using stored Nanonets API credentials.  
      - Content-Type: multipart/form-data  
      - Body Parameters:  
        - `file`: binary data from webhook `data` field  
        - `async`: false (to request synchronous processing)  
    - Expressions: URL and body parameter `file` uses binary input.  
    - Input: Binary file from webhook.  
    - Output: JSON response containing processing metadata as string in `data`.  
    - Edge Cases: Authentication failure, network errors, invalid file formats, API limits.  
    - Version Requirements: HTTP Request node v4.2 for multipart-form-data support.

  - **Code ("Code")**  
    - Type: Code (JavaScript)  
    - Configuration: Parses the stringified JSON in `data` field from previous node and extracts the `document_id` for subsequent API calls.  
    - Key Variables: `documentId` extracted from parsed response.  
    - Input: Raw JSON string from HTTP Request response.  
    - Output: JSON object with `documentId` for downstream use.  
    - Edge Cases: JSON parse errors if API response format changes or is invalid.

#### 1.3 Processing Delay

- **Overview:**  
  Allows time for Nanonets to complete document processing before fetching results.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Configuration: Pauses workflow for 15 seconds.  
    - Input: `documentId` from previous node.  
    - Output: Passes unchanged data after delay.  
    - Edge Cases: Insufficient wait time if processing is slow; excessive delay increases latency.

#### 1.4 Fetching Processed Data

- **Overview:**  
  Retrieves the full processed OCR results from Nanonets using the `documentId`.

- **Nodes Involved:**  
  - HTTP Request (named "HTTP Request2")  
  - Code (named "Code1")

- **Node Details:**

  - **HTTP Request ("HTTP Request2")**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET (default)  
      - URL: `https://app.nanonets.com/api/v4/workflows/28f99e34-2bfb-4d50-8ac3-17eaa5ee2ecf/documents/{{ $json.documentId }}`  
      - Authentication: HTTP Basic Auth with Nanonets credentials.  
    - Expressions: Uses `documentId` from previous code node for dynamic URL.  
    - Input: JSON with `documentId`.  
    - Output: JSON response string in `data` containing document OCR results.  
    - Edge Cases: Authentication errors, document not ready or not found, network issues.

  - **Code ("Code1")**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Parses the JSON string in `data` field.  
      - Extracts main header fields: `document_id`, `original_document_name`, and `status`.  
      - Iterates over recognized fields on the first page, adding their first values to the output.  
      - Processes the first table of line items by grouping cells by rows and converting these into objects with keys prefixed by `line_item_`.  
      - Merges header data with each line item to form a list of JSON objects.  
      - Returns array of JSON objects (one per line item) or a single header object if no line items found.  
    - Key Expressions: Dynamic field extraction, table cell grouping by `row`, key normalization with underscores and lowercasing.  
    - Input: Raw Nanonets OCR JSON response.  
    - Output: Array of structured JSON objects representing parsed invoice data.  
    - Edge Cases: Missing or empty tables, unexpected JSON schema, empty or malformed OCR data.

#### 1.5 Export to Excel

- **Overview:**  
  Converts the structured JSON invoice data (including line items) into an Excel file for easy download and further analysis.

- **Nodes Involved:**  
  - Convert to File

- **Node Details:**

  - **Convert to File**  
    - Type: Convert To File  
    - Configuration: Converts incoming JSON data to an Excel (XLS) file.  
    - Input: JSON array of parsed invoice objects from Code1.  
    - Output: Binary Excel file.  
    - Edge Cases: Large datasets might affect performance; unsupported data types in JSON.

#### 1.6 Response Delivery

- **Overview:**  
  Sends the generated Excel file back as the HTTP response to the original webhook request, completing the request-response cycle.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration: Responds with binary data (Excel file) received from Convert to File node.  
    - Input: Binary Excel file.  
    - Output: HTTP response to the webhook caller with the Excel file attached.  
    - Edge Cases: Response size limits, connection timeouts.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                      | Input Node(s)     | Output Node(s)    | Sticky Note                                                                                                  |
|---------------------|-------------------------|------------------------------------|-------------------|-------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook                 | Receives uploaded invoice documents| -                 | HTTP Request      | Workflow Overview: This workflow automates PDF document parsing with Nanonets. Upload a document (via form or webhook), and get structured data—including line items—returned as an Excel file. Use the Form Trigger node for user-uploaded files, or Use the Webhook node for programmatic POST/file uploads. 3. Before The Nanonets API Call Setup Required Add your Nanonets API credentials in n8n’s credentials manager (HTTP Basic Auth). Insert your Nanonets Workflow ID in the HTTP Request nodes as needed. |
| HTTP Request        | HTTP Request            | Sends document to Nanonets API     | Webhook           | Code              |                                                                                                              |
| Code                | Code                    | Extracts document ID from API response| HTTP Request    | Wait              |                                                                                                              |
| Wait                | Wait                    | Waits for OCR processing completion| Code              | HTTP Request2      |                                                                                                              |
| HTTP Request2       | HTTP Request            | Fetches processed OCR data from Nanonets| Wait           | Code1             |                                                                                                              |
| Code1               | Code                    | Parses OCR data; extracts fields and line items | HTTP Request2    | Convert to File    |                                                                                                              |
| Convert to File     | Convert To File          | Converts JSON data to Excel file   | Code1              | Respond to Webhook |                                                                                                              |
| Respond to Webhook  | Respond to Webhook       | Sends Excel file as HTTP response  | Convert to File    | -                 |                                                                                                              |
| Sticky Note         | Sticky Note             | Documentation and overview         | -                 | -                 | Workflow Overview: This workflow automates PDF document parsing with Nanonets. Upload a document (via form or webhook), and get structured data—including line items—returned as an Excel file. Use the Form Trigger node for user-uploaded files, or Use the Webhook node for programmatic POST/file uploads. 3. Before The Nanonets API Call Setup Required Add your Nanonets API credentials in n8n’s credentials manager (HTTP Basic Auth). Insert your Nanonets Workflow ID in the HTTP Request nodes as needed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier string (e.g., "1a03d800-3e91-4284-9323-0609c0974f18")  
   - Expect form data with binary file field named `data`  
   - Purpose: Receive uploaded invoice files to be processed.

2. **Create HTTP Request Node ("HTTP Request")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.nanonets.com/api/v4/workflows/{YOUR_WORKFLOW_ID}/documents/`  
   - Authentication: HTTP Basic Auth (create and select Nanonets API credentials)  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - `file`: binary from Webhook node, field `data`  
     - `async`: false  
   - Connect input from Webhook node.

3. **Create Code Node ("Code")**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code to parse the document ID:  
     ```javascript
     const inputDataString = $('HTTP Request').item.json.data;
     const parsedData = JSON.parse(inputDataString);
     const docId = parsedData.document_id;
     return { json: { documentId: docId } };
     ```
   - Connect input from "HTTP Request" node.

4. **Create Wait Node**  
   - Type: Wait  
   - Amount: 15 seconds (to allow Nanonets to finish processing)  
   - Connect input from "Code" node.

5. **Create HTTP Request Node ("HTTP Request2")**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://app.nanonets.com/api/v4/workflows/{YOUR_WORKFLOW_ID}/documents/{{ $json.documentId }}`  
   - Authentication: HTTP Basic Auth (use same Nanonets credentials)  
   - Connect input from "Wait" node.

6. **Create Code Node ("Code1")**  
   - Type: Code  
   - Language: JavaScript  
   - Paste the following code to parse OCR JSON data and extract header + line items:  
     ```javascript
     const inputDataString = $('HTTP Request2').item.json.data;
     const parsedData = JSON.parse(inputDataString);

     const headerData = {
       document_id: parsedData.document_id,
       original_document_name: parsedData.original_document_name,
       status: parsedData.status
     };

     const fields = parsedData.pages[0]?.data?.fields;
     if (fields) {
       for (const key in fields) {
         if (Array.isArray(fields[key]) && fields[key].length > 0 && typeof fields[key][0].value !== 'undefined') {
           headerData[key] = fields[key][0].value;
         }
       }
     }

     const finalOutputItems = [];
     const firstTable = parsedData.pages[0]?.data?.tables[0];

     if (firstTable && firstTable.cells && Array.isArray(firstTable.cells)) {
       const rows = {};
       firstTable.cells.forEach(cell => {
         if (!rows[cell.row]) {
           rows[cell.row] = {};
         }
         const headerKey = "line_item_" + cell.header.replace(/ /g, '_').toLowerCase();
         rows[cell.row][headerKey] = cell.text;
       });

       for (const rowNum in rows) {
         const lineItemData = rows[rowNum];
         const fullItem = { ...headerData, ...lineItemData };
         finalOutputItems.push({ json: fullItem });
       }
     }

     if (finalOutputItems.length === 0) {
       return [{ json: headerData }];
     }

     return finalOutputItems;
     ```
   - Connect input from "HTTP Request2" node.

7. **Create Convert To File Node**  
   - Type: Convert To File  
   - Operation: XLS (Excel)  
   - Connect input from "Code1" node.

8. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Respond With: Binary (Excel file)  
   - Connect input from "Convert To File" node.

9. **Connect Node Sequence:**  
   Webhook → HTTP Request → Code → Wait → HTTP Request2 → Code1 → Convert To File → Respond to Webhook

10. **Credentials Setup:**  
    - Create HTTP Basic Auth credentials using your Nanonets API key in n8n credentials manager.  
    - Assign these credentials to both HTTP Request nodes.

11. **Replace Nanonets Workflow ID:**  
    - In both HTTP Request nodes, replace `{YOUR_WORKFLOW_ID}` with your actual Nanonets workflow ID.

12. **Test:**  
    - Send a POST request to the webhook URL with a file in the `data` form field.  
    - Expect an Excel file response containing extracted invoice data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow supports programmatic invoice data extraction via webhook and returns detailed structured data including line items as an Excel file.                            | Workflow purpose                                                                                                   |
| Credentials: Nanonets API credentials are required as HTTP Basic Auth; ensure they are correctly configured in n8n credential manager before running.                         | Nanonets API integration                                                                                           |
| For alternative user uploads, a Form Trigger node could replace the webhook to allow manual uploads via n8n UI or forms.                                                      | Possible extension                                                                                                 |
| Nanonets API documentation: https://nanonets.com/blog/ocr-api/ for details on workflow IDs and API usage.                                                                      | External API documentation                                                                                         |
| The 15-second wait is a balance between latency and processing time; adjust based on average processing speed and use asynchronous calls if needed.                            | Performance tuning                                                                                                |
| Excel export uses n8n's Convert To File node, which supports XLS conversion from JSON arrays; large datasets may require chunking or pagination.                               | Data export considerations                                                                                        |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and does not contain any illegal, offensive, or protected elements. All data handled is legal and public.