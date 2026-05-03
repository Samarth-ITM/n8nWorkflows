Extract Structured Invoice Data from JotForm PDFs with GPT-4.1-mini & Sheets

https://n8nworkflows.xyz/workflows/extract-structured-invoice-data-from-jotform-pdfs-with-gpt-4-1-mini---sheets-9963


# Extract Structured Invoice Data from JotForm PDFs with GPT-4.1-mini & Sheets

### 1. Workflow Overview

This n8n workflow automates the extraction of structured invoice data from PDF invoices submitted via JotForm. It leverages OpenAI’s GPT-4.1-mini model to parse and transform raw PDF content into detailed, validated JSON data representing invoice metadata, line items, company, client details, and totals. The extracted data is then saved locally as JSON files and appended to a Google Sheet for further reporting and analysis.

Logical blocks in the workflow are:

- **1.1 Webhook Reception & Initial Data Formatting**: Receives incoming JotForm webhook POST requests with invoice submission data and formats raw request data for further processing.
- **1.2 Invoice PDF Download & Local Storage**: Fetches the invoice PDF from the JotForm URL, writes it to disk for downstream processing.
- **1.3 PDF Content Extraction & Structured Data Parsing**: Extracts raw text from the PDF, then uses GPT-4.1-mini models to extract structured invoice data in JSON format, applying schema validation for accuracy.
- **1.4 Data Output & Storage**: Saves the structured invoice JSON locally, appends a row to a Google Sheet, and prepares a binary response payload for downstream consumption or APIs.
- **1.5 Supporting Elements**: Includes sticky notes for documentation, references to credentials, and error handling configurations.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Reception & Initial Data Formatting

**Overview:**  
This block triggers the workflow upon receipt of a JotForm invoice submission webhook. It extracts and formats the raw request body for structured extraction by AI.

**Nodes Involved:**  
- Webhook  
- Formatted Structured Data Extract  
- OpenAI Chat Model for Structured Data Formatted Content  
- Sticky Note4 (documentation)

**Node Details:**

- **Webhook**  
  - Type: `n8n-nodes-base.webhook`  
  - Role: HTTP POST endpoint to receive incoming JotForm submission data.  
  - Configuration: Path set uniquely (`b3a65dd9-8203-4aff-8005-80d98fb4c030`), HTTP method POST.  
  - Input: Incoming HTTP POST JSON from JotForm submission.  
  - Output: Passes JSON body to next node.  
  - Edge cases: Network errors, payload formatting issues, webhook authentication (not explicitly configured).  

- **Formatted Structured Data Extract**  
  - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
  - Role: Uses AI to format and extract structured information from the raw request JSON string.  
  - Configuration: Uses expression to feed raw request JSON (`$('Webhook').item.json.body.rawRequest`) as text input. Schema type is `fromJson` with a detailed JSON schema example.  
  - Input: Raw JSON string from webhook.  
  - Output: Parsed structured data object containing form submission and invoice metadata.  
  - Edge cases: AI extraction failures, schema non-conformance, malformed JSON input.  

- **OpenAI Chat Model for Structured Data Formatted Content**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: AI model (GPT-4.1-mini) used by the previous node for structured extraction.  
  - Configuration: Model explicitly set to gpt-4.1-mini with cached result naming for performance.  
  - Credentials: OpenAI API credentials required.  
  - Input/Output: Linked internally to the information extractor node.  
  - Edge cases: API rate limits, network timeouts, invalid API key.  

- **Sticky Note4**  
  - Purpose: Documentation describing the webhook receiver and data formatting block.  
  - Content: "Webhook Receiver & Data Formatting"  

---

#### 1.2 Invoice PDF Download & Local Storage

**Overview:**  
Downloads the invoice PDF from the URL provided in the formatted data and writes it to local disk for content extraction.

**Nodes Involved:**  
- Download Invoice  
- Write File from Disk for Inbound Invoice Processing  
- Sticky Note5 (documentation)

**Node Details:**

- **Download Invoice**  
  - Type: `n8n-nodes-base.httpRequest`  
  - Role: Performs HTTP GET to download the PDF file from the invoice attachment URL.  
  - Configuration: URL set dynamically from extracted JSON (`{{$json.output.invoice.attachments[0].url}}`), response format set to file binary. Uses bearer token and header authentication credentials for JotForm API access.  
  - Input: URL from prior structured extraction node.  
  - Output: Binary PDF file data.  
  - Edge cases: HTTP errors (404, 403), expired URLs, auth failures, network timeouts.  

- **Write File from Disk for Inbound Invoice Processing**  
  - Type: `n8n-nodes-base.readWriteFile`  
  - Role: Writes the downloaded PDF binary data to a local file (`c:\PDF-Invoice-Sample.pdf`) for subsequent extraction.  
  - Configuration: Write operation, filename statically set, data property mapped from prior node’s binary output.  
  - Input: Binary PDF file.  
  - Output: Passes file binary data for extraction.  
  - Edge cases: Filesystem permission errors, disk full, path not found.  

- **Sticky Note5**  
  - Purpose: Documentation describing the invoice download and write to disk process.  
  - Content: "Invoice Download & Write to Disk"  

---

#### 1.3 PDF Content Extraction & Structured Data Parsing

**Overview:**  
Extracts text from the saved PDF and uses GPT-4.1-mini powered Langchain nodes to parse the invoice text into a detailed structured JSON matching a predefined invoice schema.

**Nodes Involved:**  
- Extract from File  
- Structured Data Extract  
- Structured Output Parser  
- OpenAI Chat Model for Structured Data  
- OpenAI Chat Model for Output Parser  
- Sticky Note2 (Structured Data Extraction Using OpenAI)

**Node Details:**

- **Extract from File**  
  - Type: `n8n-nodes-base.extractFromFile`  
  - Role: Extracts raw text content from the PDF binary file.  
  - Configuration: Operation set to PDF extraction, binary property name dynamically sourced from previous node.  
  - Input: Local PDF binary file.  
  - Output: Extracted plain text of the invoice.  
  - Edge cases: Corrupt PDF, unsupported PDF versions, large file size causing timeout.  

- **Structured Data Extract**  
  - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
  - Role: Uses an LLM chain with GPT-4.1-mini to parse extracted text into structured invoice fields.  
  - Configuration: Prompt prepends text with instruction "Parse the below invoice info into a structured data" and passes extracted text. Output parser enabled. Retry on failure enabled.  
  - Input: Extracted invoice text.  
  - Output: Structured JSON invoice data.  
  - Edge cases: AI parsing errors, partial extraction, model API limits.  

- **Structured Output Parser**  
  - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - Role: Validates and auto-fixes the structured JSON output against a manual JSON schema representing invoice structure.  
  - Configuration: JSON Schema defines invoice fields: invoiceId, invoiceNumber, dates, company/client info, items array, totals, currency, payment info, and attachments. Auto-fix enabled to correct minor schema violations.  
  - Input: Raw structured JSON from previous node.  
  - Output: Clean, validated invoice JSON.  
  - Edge cases: Schema mismatch, auto-fix failures, missing required fields.  

- **OpenAI Chat Model for Structured Data**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: GPT-4.1-mini model used to generate structured data in the chain LLM node.  
  - Configuration: Model set explicitly to gpt-4.1-mini.  
  - Input/Output: Linked internally to the chain LLM node.  
  - Edge cases: API errors, network delays.  

- **OpenAI Chat Model for Output Parser**  
  - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  - Role: GPT-4.1-mini model used by the output parser for schema validation and correction.  
  - Configuration: Model set to gpt-4.1-mini.  
  - Input/Output: Linked internally to the output parser node.  
  - Edge cases: API rate limits, parsing delays.  

- **Sticky Note2**  
  - Purpose: Labeling block as "Structured Data Extraction Using OpenAI" for clarity.  

---

#### 1.4 Data Output & Storage

**Overview:**  
Finalizes the workflow by saving validated structured invoice JSON locally, appending the data to a Google Sheet, and preparing a binary response for integration or storage.

**Nodes Involved:**  
- Append or update row in sheet  
- Create a Binary Response  
- Write the Structured Invoice to Disk  
- Sticky Note3 (Export Data Handling)

**Node Details:**

- **Append or update row in sheet**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Appends or updates a row in a designated Google Sheet with the structured invoice data for reporting.  
  - Configuration: Uses automatic column mapping to map output JSON fields, targets a specific sheet (`gid=0`) and document ID (Google Sheet URL provided in cached result).  
  - Input: Structured invoice JSON from previous parsing node.  
  - Output: Confirmation of row append/update.  
  - Credentials: Google Sheets OAuth2 required.  
  - Edge cases: OAuth token expiry, API quota exceeded, sheet access denied.  

- **Create a Binary Response**  
  - Type: `n8n-nodes-base.function`  
  - Role: Converts the structured invoice JSON output into a base64 encoded binary payload for downstream API consumption or storage.  
  - Configuration: Custom JS code creates a binary buffer from JSON string.  
  - Input: Structured JSON invoice.  
  - Output: Binary data with base64 encoded JSON.  
  - Edge cases: Code errors, large payloads causing memory issues.  

- **Write the Structured Invoice to Disk**  
  - Type: `n8n-nodes-base.readWriteFile`  
  - Role: Writes the final structured invoice JSON to local disk with a filename based on invoice ID and date.  
  - Configuration: Dynamic filename pattern (`C:\{invoiceId}-{invoiceDate}.json`), write operation.  
  - Input: Binary JSON data.  
  - Output: File saved locally.  
  - Edge cases: Disk permission errors, invalid filename characters, disk space.  

- **Sticky Note3**  
  - Purpose: Documents this block as “Export Data Handling” to clarify data saving and export operations.  

---

#### 1.5 Supporting Elements

**Overview:**  
Additional nodes support workflow documentation and branding.

**Nodes Involved:**  
- Sticky Note (logo and branding)  
- Sticky Note1 (workflow purpose and flow summary)

**Node Details:**

- **Sticky Note (a21275f7-fc1b-47ab-ba6a-06bcde58875e)**  
  - Displays JotForm logo and highlights the use of OpenAI GPT-4.1-mini for invoice PDF content extraction.  

- **Sticky Note1**  
  - Provides a detailed purpose and flow summary, outlining the workflow steps and integrations used: JotForm API, OpenAI GPT-4.1-mini, Google Sheets, and local file storage.  

---

### 3. Summary Table

| Node Name                                | Node Type                                      | Functional Role                              | Input Node(s)                          | Output Node(s)                                                   | Sticky Note                                                 |
|------------------------------------------|------------------------------------------------|----------------------------------------------|---------------------------------------|-----------------------------------------------------------------|-------------------------------------------------------------|
| Webhook                                  | n8n-nodes-base.webhook                        | Entry point: Receive JotForm invoice webhook | None                                  | Formatted Structured Data Extract                               | Webhook Receiver & Data Formatting                           |
| Formatted Structured Data Extract        | @n8n/n8n-nodes-langchain.informationExtractor | Extract and format raw JSON submission data  | Webhook                               | Download Invoice                                                | Webhook Receiver & Data Formatting                           |
| OpenAI Chat Model for Structured Data Formatted Content | @n8n/n8n-nodes-langchain.lmChatOpenAi           | AI model for initial structured data extraction | Formatted Structured Data Extract (AI model) | Formatted Structured Data Extract                               |                                                             |
| Download Invoice                         | n8n-nodes-base.httpRequest                     | Download invoice PDF from JotForm            | Formatted Structured Data Extract     | Write File from Disk for Inbound Invoice Processing             | Invoice Download & Write to Disk                             |
| Write File from Disk for Inbound Invoice Processing | n8n-nodes-base.readWriteFile                     | Save downloaded PDF to local disk             | Download Invoice                     | Extract from File                                              | Invoice Download & Write to Disk                             |
| Extract from File                       | n8n-nodes-base.extractFromFile                  | Extract text content from saved PDF           | Write File from Disk for Inbound Invoice Processing | Structured Data Extract                                        | Structured Data Extraction Using OpenAI                      |
| Structured Data Extract                 | @n8n/n8n-nodes-langchain.chainLlm               | Parse extracted text into structured JSON data | Extract from File                   | Append or update row in sheet, Create a Binary Response         | Structured Data Extraction Using OpenAI                      |
| OpenAI Chat Model for Structured Data  | @n8n/n8n-nodes-langchain.lmChatOpenAi            | AI model for structured data extraction        | Structured Data Extract (AI model)   | Structured Data Extract                                        |                                                             |
| Structured Output Parser                | @n8n/n8n-nodes-langchain.outputParserStructured | Validate and auto-fix structured JSON output | OpenAI Chat Model for Output Parser   | Structured Data Extract (ai_outputParser)                      |                                                             |
| OpenAI Chat Model for Output Parser     | @n8n/n8n-nodes-langchain.lmChatOpenAi            | AI model for output parsing and validation     | Structured Output Parser (AI model)  | Structured Output Parser                                       |                                                             |
| Append or update row in sheet           | n8n-nodes-base.googleSheets                      | Append structured data to Google Sheet         | Structured Data Extract              | Create a Binary Response                                       | Export Data Handling                                        |
| Create a Binary Response                 | n8n-nodes-base.function                          | Encode JSON output as base64 binary             | Append or update row in sheet        | Write the Structured Invoice to Disk                           | Export Data Handling                                        |
| Write the Structured Invoice to Disk    | n8n-nodes-base.readWriteFile                     | Save final invoice JSON file to disk            | Create a Binary Response             | None                                                          | Export Data Handling                                        |
| Sticky Note                            | n8n-nodes-base.stickyNote                        | Branding and logo                              | None                                  | None                                                          | Uses OpenAI gpt-4.1-mini for structured data extraction    |
| Sticky Note1                           | n8n-nodes-base.stickyNote                        | Workflow purpose and summary                   | None                                  | None                                                          | Automates invoice data extraction from JotForm PDFs        |
| Sticky Note2                           | n8n-nodes-base.stickyNote                        | Block label for structured data extraction    | None                                  | None                                                          | Structured Data Extraction Using OpenAI                     |
| Sticky Note3                           | n8n-nodes-base.stickyNote                        | Block label for data export handling           | None                                  | None                                                          | Export Data Handling                                        |
| Sticky Note4                           | n8n-nodes-base.stickyNote                        | Block label for webhook and formatting         | None                                  | None                                                          | Webhook Receiver & Data Formatting                           |
| Sticky Note5                           | n8n-nodes-base.stickyNote                        | Block label for invoice download and save      | None                                  | None                                                          | Invoice Download & Write to Disk                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - HTTP Method: POST  
   - Path: Use a unique identifier (e.g., `b3a65dd9-8203-4aff-8005-80d98fb4c030`)  
   - No authentication configured (adjust as needed)  

2. **Create Formatted Structured Data Extract Node**  
   - Type: `Information Extractor` (Langchain)  
   - Input Text: Use expression `= Extract Structured Information in JSON from the provided invoice raw request {{ $('Webhook').item.json.body.rawRequest }}`  
   - Schema Type: `fromJson`  
   - JSON Schema Example: Use the detailed example provided in the workflow JSON  
   - Connect input from Webhook main output  
   - Configure AI model node next (see step 3)  

3. **Create OpenAI Chat Model Node for Structured Data Formatted Content**  
   - Type: `lmChatOpenAi`  
   - Model: Select `gpt-4.1-mini`  
   - Credentials: Set your OpenAI API credentials  
   - Connect this node as AI language model input to the Information Extractor node  

4. **Create Download Invoice Node**  
   - Type: `HTTP Request`  
   - Method: GET  
   - URL: Expression `={{ $json.output.invoice.attachments[0].url }}` (from prior extraction)  
   - Response Format: File (binary)  
   - Authentication: Use HTTP Header Auth with JotForm API key or Bearer token as per your setup  
   - Connect input from Formatted Structured Data Extract node  

5. **Create Write File from Disk for Inbound Invoice Processing Node**  
   - Type: `Read/Write File`  
   - Operation: Write  
   - File Name: `"c:\\PDF-Invoice-Sample.pdf"` (or your chosen path)  
   - Data Property Name: Use expression referencing the downloaded file binary name  
   - Connect input from Download Invoice node  

6. **Create Extract from File Node**  
   - Type: `Extract From File`  
   - Operation: PDF  
   - Binary Property Name: Expression pointing to the saved PDF binary property  
   - Connect input from Write File from Disk node  

7. **Create Structured Data Extract Node**  
   - Type: `Chain LLM` (Langchain)  
   - Text Input: `=Parse the below invoice info into a structured data\n\n{{ $json.text }}`  
   - Enable output parser  
   - Enable retry on fail for robustness  
   - Connect input from Extract from File node  

8. **Create OpenAI Chat Model Node for Structured Data**  
   - Type: `lmChatOpenAi`  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API  
   - Connect as AI language model input to Structured Data Extract node  

9. **Create Structured Output Parser Node**  
   - Type: `Output Parser Structured` (Langchain)  
   - Auto Fix: Enabled  
   - Input Schema: Paste the detailed invoice JSON schema provided in the workflow JSON  
   - Connect AI output parser input from OpenAI Chat Model for Output Parser node (see next step)  

10. **Create OpenAI Chat Model Node for Output Parser**  
    - Type: `lmChatOpenAi`  
    - Model: `gpt-4.1-mini`  
    - Credentials: OpenAI API  
    - Connect as AI language model input to Structured Output Parser node  

11. **Connect Structured Output Parser’s AI output parser output to Structured Data Extract node**  
    - This completes the validation loop for structured data extraction  

12. **Create Append or update row in sheet Node**  
    - Type: `Google Sheets`  
    - Operation: Append or update row  
    - Sheet Name: `gid=0` (or your target sheet)  
    - Document ID: Your Google Sheet document ID  
    - Column Mapping Mode: Auto map input data  
    - Credentials: Google Sheets OAuth2 account  
    - Connect input from Structured Data Extract node  

13. **Create Create a Binary Response Node**  
    - Type: `Function`  
    - Code:  
      ```js
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64'),
        },
      };
      return items;
      ```  
    - Connect input from Append or update row in sheet node  

14. **Create Write the Structured Invoice to Disk Node**  
    - Type: `Read/Write File`  
    - Operation: Write  
    - File Name: Expression `=C:\\{{ $json.output.invoiceId }}-{{ $json.output.invoiceDate }}.json`  
    - Connect input from Create a Binary Response node  

15. **Add Sticky Notes for Documentation**  
    - Add notes for each block as per the workflow for clarity and maintenance  

16. **Credential Setup**  
    - OpenAI API key with GPT-4.1-mini access  
    - JotForm API key or Bearer Token for PDF download authentication  
    - Google Sheets OAuth2 credentials with write access  

17. **Testing & Validation**  
    - Test with sample JotForm webhook payloads containing invoice PDF URLs  
    - Verify local PDF download, JSON extraction, sheet updates, and saved files  
    - Monitor for errors such as API limits, malformed PDFs, or schema mismatches  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Uses OpenAI GPT-4.1-mini model for accurate structured data extraction from invoice PDFs submitted via JotForm.                                                                                                                                  | See Sticky Note with JotForm logo and AI usage                                                                 |
| Workflow integrates JotForm API for PDF downloads, Google Sheets for data storage, and local disk for JSON persistence.                                                                                                                          | Explained in workflow description and Sticky Note1                                                             |
| Invoice JSON schema includes comprehensive fields: invoiceId, dates, company, client, line items with quantities and prices, taxes, discounts, payment info, and attachments.                                                                      | Defined in Structured Output Parser node configuration                                                          |
| Retry on fail enabled on key AI parsing chain to improve robustness against transient API or network failures.                                                                                                                                   | Structured Data Extract node configuration                                                                       |
| Use Google Sheets OAuth2 credentials with sufficient permission to append and update rows in the target spreadsheet.                                                                                                                              | Google Sheets node credential requirement                                                                        |
| Ensure local disk paths for PDF and JSON file writes are accessible and permissions are correctly set, especially on Windows environments.                                                                                                       | Write File nodes configuration                                                                                   |
| The workflow does not implement webhook authentication or validation—consider adding security layers for production environments.                                                                                                               | Security best practice note                                                                                       |
| GPT-4.1-mini is a cost-effective variant of GPT-4, providing strong capabilities with reduced latency and cost.                                                                                                                                  | Model choice rationale                                                                                           |
| For extended automation, consider adding error handling nodes or alerting on failures (not present in the current workflow).                                                                                                                     | Suggested enhancement                                                                                           |

---

**Disclaimer:** The provided text is solely derived from an n8n automated workflow. It complies with all applicable content policies, containing no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.