Extract Data from Documents with GPT-4, PDFVector & PostgreSQL Export

https://n8nworkflows.xyz/workflows/extract-data-from-documents-with-gpt-4--pdfvector---postgresql-export-7357


# Extract Data from Documents with GPT-4, PDFVector & PostgreSQL Export

### 1. Workflow Overview

This workflow automates the extraction of structured data from various document types such as invoices, contracts, reports, and forms. It is designed for scenarios where documents arrive continuously in a monitored folder, and the need is to parse, extract key information with AI assistance, validate the output, route data based on document type, store records in PostgreSQL databases, and finally export collected data into CSV files.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Monitors a local folder for new documents and triggers the workflow.
- **1.2 Document Parsing:** Uses PDFVector’s LLM-powered parser to process document content for better text extraction.
- **1.3 AI-Based Data Extraction:** Employs GPT-4 to extract specific structured fields from the parsed document text.
- **1.4 Data Validation & Cleaning:** Applies custom JavaScript code to validate and sanitize the extracted data.
- **1.5 Conditional Routing:** Routes data based on document type to the appropriate storage node.
- **1.6 Data Storage:** Inserts structured data into PostgreSQL tables — invoices into an invoice-specific table, others into a general documents table.
- **1.7 Data Export:** Exports stored data into a dated CSV file for external use or archiving.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Watches a local folder `/documents/incoming` for newly created files and triggers the workflow when new documents arrive.

- **Nodes Involved:**  
  - Watch Folder

- **Node Details:**  

  - **Watch Folder**  
    - Type: `Local File Trigger`  
    - Role: Trigger node monitoring filesystem events  
    - Configuration: Watches path `/documents/incoming` for `file:created` events  
    - Inputs: None (trigger node)  
    - Outputs: Emits file metadata including file path and name  
    - Edge Cases: Permissions issues on folder, missed events if folder is inaccessible, file locking or partial file write during detection  

---

#### 2.2 Document Parsing

- **Overview:**  
  Parses the newly detected document content using PDFVector’s LLM parser for enhanced text extraction.

- **Nodes Involved:**  
  - PDF Vector - Parse Document

- **Node Details:**  

  - **PDF Vector - Parse Document**  
    - Type: `PDFVector` node (third-party integration)  
    - Role: Parses document content using LLM-enhanced techniques for better text extraction than simple OCR  
    - Configuration:  
      - Resource: Document  
      - Operation: Parse  
      - Document URL: Uses expression `{{$json.filePath}}` from the trigger node’s output to specify the file to parse  
      - LLM usage: always enabled for improved parsing quality  
    - Inputs: Output of Watch Folder node (file metadata)  
    - Outputs: JSON containing extracted document content under `.content` field  
    - Edge Cases: File not found or inaccessible, parsing errors, large file size timeout, API rate limits or errors from PDFVector service  

---

#### 2.3 AI-Based Data Extraction

- **Overview:**  
  Sends the parsed document content to OpenAI GPT-4 model to extract predefined structured data fields as JSON.

- **Nodes Involved:**  
  - Extract Structured Data

- **Node Details:**  

  - **Extract Structured Data**  
    - Type: `OpenAI` node  
    - Role: Uses GPT-4 to extract structured information from the document text  
    - Configuration:  
      - Model: GPT-4  
      - Response Format: JSON Object  
      - Prompt: Requests extraction of specific fields including document type, dates, parties, amounts, terms, references, addresses, contacts  
      - Message Content: Template includes `{{ $json.content }}` from previous node’s output  
    - Inputs: Document content from PDF Vector node  
    - Outputs: JSON with extracted fields (e.g., documentType, date, parties, amounts, emails)  
    - Edge Cases: API authentication errors, rate limits, unexpected or malformed content from the parser, invalid JSON response, network timeouts  

---

#### 2.4 Data Validation & Cleaning

- **Overview:**  
  Processes the raw extracted JSON to validate, clean, and standardize fields such as dates, monetary values, and emails.

- **Nodes Involved:**  
  - Validate & Clean Data

- **Node Details:**  

  - **Validate & Clean Data**  
    - Type: `Code` node (JavaScript)  
    - Role: Parses the raw JSON string, validates fields, cleans monetary values, filters emails, and adds metadata  
    - Key Logic:  
      - Parses raw JSON from `content` field  
      - Validates documentType, defaults to 'unknown' if missing  
      - Parses dates to ISO format or null if invalid  
      - Cleans amounts by stripping non-numeric characters and parsing floats  
      - Validates email formats with regex  
      - Attaches original raw data, file name from Watch Folder node, and timestamp of processing  
    - Inputs: JSON from OpenAI node  
    - Outputs: Cleaned and validated JSON object  
    - Edge Cases: Malformed JSON input causing parse errors, missing expected fields, invalid date formats, empty or invalid amounts, emails failing regex validation  

---

#### 2.5 Conditional Routing

- **Overview:**  
  Routes the validated data into different storage paths based on document type (e.g., invoices vs others).

- **Nodes Involved:**  
  - Route by Document Type

- **Node Details:**  

  - **Route by Document Type**  
    - Type: `Switch` node  
    - Role: Routes data on the value of `documentType` field  
    - Configuration:  
      - Condition: If `documentType` equals `"invoice"` route to invoice storage  
      - Default route for other document types  
    - Inputs: Validated JSON from Code node  
    - Outputs: Two outputs, one for invoices, one for other documents  
    - Edge Cases: Unexpected or missing `documentType` causing fallback to default route, case sensitivity in string matching  

---

#### 2.6 Data Storage

- **Overview:**  
  Inserts structured data into PostgreSQL tables designed for invoices and general document records.

- **Nodes Involved:**  
  - Store Invoice Data  
  - Store Other Documents

- **Node Details:**  

  - **Store Invoice Data**  
    - Type: `PostgreSQL` node  
    - Role: Inserts invoice data into `invoices` table  
    - Configuration:  
      - Operation: Insert  
      - Table: `invoices`  
      - Columns: `invoice_number, vendor, amount, date, raw_data` (assumed mapped from validated fields)  
    - Inputs: Data routed for invoices from Switch node  
    - Outputs: Confirmation of insertion, passes data forward  
    - Credential: Requires PostgreSQL credentials configured in n8n  
    - Edge Cases: Database connection errors, constraint violations, missing required fields, data type mismatches  

  - **Store Other Documents**  
    - Type: `PostgreSQL` node  
    - Role: Inserts generic document data into `documents` table  
    - Configuration:  
      - Operation: Insert  
      - Table: `documents`  
      - Columns: `type, content, metadata, processed_at` (mapped from validated JSON)  
    - Inputs: Data routed for non-invoice documents from Switch node  
    - Outputs: Confirmation of insertion, passes data forward  
    - Credential: Same PostgreSQL credentials as above  
    - Edge Cases: Same as Store Invoice Data node  

---

#### 2.7 Data Export

- **Overview:**  
  Exports the stored data into a CSV file with a timestamped file name for archiving or downstream processing.

- **Nodes Involved:**  
  - Export to CSV

- **Node Details:**  

  - **Export to CSV**  
    - Type: `Write Binary File` node  
    - Role: Writes a CSV file to disk containing extracted data from previous storage nodes  
    - Configuration:  
      - File Name: `extracted_data_{{ $now.format('yyyy-MM-dd') }}.csv` (dynamically dated)  
      - File Content: Converts all incoming items’ JSON to CSV format using `.toCsv()` method  
    - Inputs: Combined output from both Store Invoice Data and Store Other Documents nodes  
    - Outputs: File saved on disk (default location depends on environment)  
    - Edge Cases: File write permission errors, filename collisions, large dataset memory limits  

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                       | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                              |
|---------------------------|------------------------|------------------------------------|-----------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| Pipeline Info             | Sticky Note            | Workflow overview and documentation | None                  | None                   | ## Document Extraction Pipeline Extracts structured data from: - Invoices - Contracts - Reports - Forms Customize extraction rules in the AI node |
| Watch Folder             | Local File Trigger     | Trigger workflow on new documents   | None                  | PDF Vector - Parse Document | Triggers when new documents arrive                                                                     |
| PDF Vector - Parse Document | PDFVector              | Parse document content with LLM     | Watch Folder          | Extract Structured Data | Parse with LLM for better extraction                                                                    |
| Extract Structured Data  | OpenAI                 | Extract structured data via GPT-4   | PDF Vector - Parse Document | Validate & Clean Data      |                                                                                                        |
| Validate & Clean Data    | Code                   | Validate and clean extracted JSON   | Extract Structured Data | Route by Document Type  |                                                                                                        |
| Route by Document Type   | Switch                 | Route data by document type         | Validate & Clean Data  | Store Invoice Data, Store Other Documents |                                                                                                        |
| Store Invoice Data       | PostgreSQL             | Insert invoice data into invoices table | Route by Document Type | Export to CSV           |                                                                                                        |
| Store Other Documents    | PostgreSQL             | Insert generic documents into documents table | Route by Document Type | Export to CSV           |                                                                                                        |
| Export to CSV            | Write Binary File      | Export extracted data to CSV file   | Store Invoice Data, Store Other Documents | None                   |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note node**  
   - Name: `Pipeline Info`  
   - Content:  
     ```
     ## Document Extraction Pipeline

     Extracts structured data from:
     - Invoices
     - Contracts
     - Reports
     - Forms

     Customize extraction rules in the AI node
     ```

2. **Create a Local File Trigger node**  
   - Name: `Watch Folder`  
   - Path: `/documents/incoming` (adjust to your local watch folder)  
   - Events: `file:created`  
   - This node triggers the workflow on new files.

3. **Create a PDFVector node**  
   - Name: `PDF Vector - Parse Document`  
   - Resource: `document`  
   - Operation: `parse`  
   - Document URL: Use expression `{{$json.filePath}}` from Watch Folder output  
   - Use LLM: Always enabled  
   - Connect input from `Watch Folder`.

4. **Create an OpenAI node**  
   - Name: `Extract Structured Data`  
   - Model: `gpt-4`  
   - Response Format: JSON object  
   - Messages: Single message with content:  
     ```
     Extract the following information from this document:

     1. Document Type (invoice, contract, report, etc.)
     2. Date/Dates mentioned
     3. Parties involved (names, companies)
     4. Key amounts/values
     5. Important terms or conditions
     6. Reference numbers
     7. Addresses
     8. Contact information

     Document content:
     {{ $json.content }}

     Return as structured JSON.
     ```  
   - Connect input from `PDF Vector - Parse Document`.  
   - Configure OpenAI credentials in n8n.

5. **Create a Code node**  
   - Name: `Validate & Clean Data`  
   - Function code (JavaScript):  
     ```javascript
     // Validate and clean extracted data
     const extracted = JSON.parse($json.content);
     const validated = {};

     // Validate document type
     validated.documentType = extracted.documentType || 'unknown';

     // Parse and validate dates
     if (extracted.date) {
       const date = new Date(extracted.date);
       validated.date = isNaN(date) ? null : date.toISOString();
     }

     // Clean monetary values
     if (extracted.amounts) {
       validated.amounts = extracted.amounts.map(amt => {
         const cleaned = amt.replace(/[^0-9.-]/g, '');
         return parseFloat(cleaned) || 0;
       });
     }

     // Validate email addresses
     if (extracted.emails) {
       validated.emails = extracted.emails.filter(email => 
         /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
       );
     }

     validated.raw = extracted;
     validated.fileName = $node['Watch Folder'].json.fileName;
     validated.processedAt = new Date().toISOString();

     return validated;
     ```  
   - Connect input from `Extract Structured Data`.

6. **Create a Switch node**  
   - Name: `Route by Document Type`  
   - Add string condition:  
     - Value 1: `{{ $json.documentType }}`  
     - Operation: equals  
     - Value 2: `invoice`  
   - Configure two outputs: one for invoices, one default for other documents  
   - Connect input from `Validate & Clean Data`.

7. **Create a PostgreSQL node for invoices**  
   - Name: `Store Invoice Data`  
   - Operation: Insert  
   - Table: `invoices`  
   - Columns: `invoice_number, vendor, amount, date, raw_data` (map from validated JSON fields accordingly)  
   - Connect input from first output of `Route by Document Type` (invoice path).  
   - Configure PostgreSQL credentials.

8. **Create a PostgreSQL node for other documents**  
   - Name: `Store Other Documents`  
   - Operation: Insert  
   - Table: `documents`  
   - Columns: `type, content, metadata, processed_at` (map from validated JSON fields)  
   - Connect input from second output of `Route by Document Type` (default path).  
   - Use same PostgreSQL credentials.

9. **Create a Write Binary File node**  
   - Name: `Export to CSV`  
   - File Name: `extracted_data_{{ $now.format('yyyy-MM-dd') }}.csv`  
   - File Content: Use expression `{{ $items().map(item => item.json).toCsv() }}` to convert JSON to CSV  
   - Connect inputs from both `Store Invoice Data` and `Store Other Documents`.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow relies on PDFVector’s LLM-powered parser for superior document parsing compared to basic OCR methods.       | PDFVector integration in n8n (third-party node)                                                |
| GPT-4 model is used for versatile, customizable extraction of varied document types, improving accuracy and flexibility. | OpenAI GPT-4 API usage within n8n                                                              |
| Ensure PostgreSQL tables `invoices` and `documents` are pre-created with appropriate schema matching the inserted columns.| PostgreSQL database preparation                                                                |
| The exported CSV file is named with the current date to facilitate daily archiving or batch processing.                   | File naming convention in Write Binary File node                                               |
| Adjust local folder path in `Watch Folder` node as per deployment environment permissions and folder structure.           | Local filesystem access and permissions                                                        |
| This pipeline can be extended by adding more document types and adapting the AI prompt and routing logic accordingly.     | Workflow scalability and customization                                                         |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.