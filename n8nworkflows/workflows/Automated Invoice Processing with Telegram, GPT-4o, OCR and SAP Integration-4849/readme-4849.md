Automated Invoice Processing with Telegram, GPT-4o, OCR and SAP Integration

https://n8nworkflows.xyz/workflows/automated-invoice-processing-with-telegram--gpt-4o--ocr-and-sap-integration-4849


# Automated Invoice Processing with Telegram, GPT-4o, OCR and SAP Integration

### 1. Workflow Overview

This workflow automates the processing of invoices received via Telegram by leveraging OCR, AI-based document parsing, and SAP integration. It is designed for businesses that want to digitize invoice handling, extract structured data with high accuracy from uploaded documents, confirm user actions via Telegram, and finally create purchase invoices in an SAP system.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Acknowledgement:** Receive invoice documents via Telegram, acknowledge receipt, and download the file.
- **1.2 OCR Processing and Parsing:** Upload the invoice to LlamaIndex OCR service, poll for processing completion, retrieve results in markdown format.
- **1.3 AI Extraction of Invoice Data:** Use OpenAI GPT-4o (via LangChain nodes) to extract structured invoice data from OCR output, parsing it into a precise JSON schema.
- **1.4 Data Preparation for SAP:** Process product line details, organize data according to SAP API requirements, and prepare payloads.
- **1.5 User Confirmation via Telegram:** Ask the user via Telegram whether to proceed with uploading data to SAP.
- **1.6 SAP Integration:** Authenticate with SAP, retrieve necessary header and row details from Google Sheets, and post the purchase invoice data.
- **1.7 Final Notification:** Notify the user via Telegram of successful creation of the invoice in SAP.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Acknowledgement

- **Overview:** This block listens for incoming Telegram messages containing invoice documents, extracts file IDs and metadata, acknowledges the receipt to the user, and downloads the invoice file.
- **Nodes Involved:** Trigger Receive Message, Capture Telegram Data, File Received, Download File

**Node Details:**

- **Trigger Receive Message**
  - Type: Telegram Trigger
  - Role: Entry point for the workflow; listens for new messages on Telegram.
  - Config: Listens for "message" updates.
  - Input: Webhook from Telegram.
  - Output: Telegram message JSON.
  - Failures: Connectivity issues with Telegram API.

- **Capture Telegram Data**
  - Type: Set
  - Role: Extracts and stores necessary metadata from the Telegram message (chat ID, file ID, caption).
  - Key expressions: Extracts `message.chat.id`, `message.document.file_id`, and caption.
  - Input: Telegram message JSON from trigger.
  - Output: Structured JSON with extracted fields.
  - Failures: Missing document or malformed message.

- **File Received**
  - Type: Telegram
  - Role: Sends acknowledgment text to user confirming document receipt.
  - Config: Sends fixed message "Hemos recibido tu documento y lo estamos procesando...".
  - Input: Chat ID from previous node.
  - Output: Confirmation message sent.
  - Failures: Telegram API errors.

- **Download File**
  - Type: Telegram
  - Role: Downloads the actual invoice document file from Telegram servers using the file ID.
  - Config: Uses the file ID extracted to fetch the file.
  - Input: File ID from Capture Telegram Data.
  - Output: Binary file data for subsequent processing.
  - Failures: File not found, Telegram server unavailability.

---

#### 1.2 OCR Processing and Parsing

- **Overview:** Uploads the downloaded invoice file to LlamaIndex OCR API, then polls the job status until completion, and finally fetches the parsed markdown result.
- **Nodes Involved:** Upload File LlamaIndex, Validate Status LlamaIndex, Status?, Wait, Get Results LlamaIndex

**Node Details:**

- **Upload File LlamaIndex**
  - Type: HTTP Request
  - Role: Sends the invoice file as multipart form data to LlamaIndex OCR API for parsing.
  - Config: POST to `https://api.cloud.llamaindex.ai/api/v1/parsing/upload` with Authorization header using API key stored in workflow variables.
  - Input: Binary file data from Download File node.
  - Output: Job ID and initial job info.
  - Failures: API auth errors, file upload issues.

- **Validate Status LlamaIndex**
  - Type: HTTP Request
  - Role: Polls the LlamaIndex API for job status using the returned job ID.
  - Config: GET request to the job status endpoint with Authorization header.
  - Input: Job ID from Upload File LlamaIndex.
  - Output: JSON status response.
  - Failures: API errors, network timeouts.

- **Status?**
  - Type: Switch
  - Role: Routes workflow depending on job status (SUCCESS or PENDING).
  - Logic: If status is "SUCCESS", proceed to get results; if "PENDING", wait and poll again.
  - Input: Status JSON.
  - Output: Two branches for status.
  - Failures: Unexpected status values.

- **Wait**
  - Type: Wait
  - Role: Pauses execution for 3 seconds before polling again.
  - Config: Fixed 3 seconds delay.
  - Input: PENDING status branch.
  - Output: Triggers Validate Status LlamaIndex again.
  - Failures: None significant.

- **Get Results LlamaIndex**
  - Type: HTTP Request
  - Role: Retrieves the OCR parsed result in markdown format once job is complete.
  - Config: GET request to job result endpoint with Authorization.
  - Input: Job ID from Validate Status LlamaIndex.
  - Output: Markdown text representing extracted invoice text.
  - Failures: API errors, data format issues.

---

#### 1.3 AI Extraction of Invoice Data

- **Overview:** Uses OpenAI GPT-4o model via LangChain nodes to analyze the markdown invoice text and extract structured invoice data, including supplier info, invoice numbers, dates, product details, and totals.
- **Nodes Involved:** OpenAI Chat Model, Basic LLM Chain, Structured Output Parser (Example), Split Out

**Node Details:**

- **OpenAI Chat Model**
  - Type: LangChain LM Chat OpenAI
  - Role: Provides the GPT-4o model interface to process the invoice text.
  - Config: Model set to "gpt-4o-mini".
  - Input: Markdown text from Get Results LlamaIndex.
  - Output: Text output for further processing.
  - Failures: API quota, auth errors, timeout.

- **Basic LLM Chain**
  - Type: LangChain Chain LLM
  - Role: Defines prompt template instructing GPT-4o to extract invoice fields into JSON.
  - Config: Contains detailed instructions and output format example; expects JSON output.
  - Input: Markdown text from OpenAI Chat Model.
  - Output: JSON with extracted invoice fields.
  - Failures: Parsing errors, incomplete extraction.

- **Structured Output Parser (Example)**
  - Type: LangChain Output Parser Structured
  - Role: Validates and parses the JSON output from GPT-4o against example schema.
  - Config: JSON schema example for invoice data fields.
  - Input: Text output from Basic LLM Chain.
  - Output: Structured JSON.
  - Failures: Schema mismatches.

- **Split Out**
  - Type: Split Out
  - Role: Splits the `detalle_productos` array into individual product lines to process separately.
  - Input: JSON from Basic LLM Chain.
  - Output: Multiple items, each representing one product line.
  - Failures: Empty or missing product details array.

---

#### 1.4 Data Preparation for SAP

- **Overview:** Processes the product line items and organizes the invoice data into the format required by the SAP Purchase Invoices API.
- **Nodes Involved:** Detail, Get Header, Get Row Details, Generate DocumentLines, Merge, Create JSON

**Node Details:**

- **Detail**
  - Type: Google Sheets
  - Role: Appends each product line detail to a Google Sheet for record-keeping.
  - Config: Maps product fields (code, description, quantity, price, subtotal) plus invoice number.
  - Input: Split Out product lines.
  - Output: Appended rows in Google Sheets.
  - Failures: Google Sheets API limits, auth errors.

- **Get Header**
  - Type: Google Sheets
  - Role: Retrieves header data from Google Sheets, possibly to validate or enrich data.
  - Config: Reads specific sheet name and document ID (configured via variables).
  - Input: Triggered after SAP connection.
  - Output: Header rows data.
  - Failures: Sheet not found, API errors.

- **Get Row Details**
  - Type: Google Sheets
  - Role: Retrieves detailed rows of invoice data from Google Sheets.
  - Config: Similar to Get Header with sheet and document ID.
  - Input: After Get Header.
  - Output: Rows data.
  - Failures: API issues.

- **Generate DocumentLines**
  - Type: Code (JavaScript)
  - Role: Converts product line items into SAP API-compatible document line objects.
  - Logic: Maps each item to an object with ItemCode, Quantity, UnitPrice based on product code, quantity, and price.
  - Input: Data from Get Row Details.
  - Output: Object with `DocumentLines` array.
  - Failures: Missing fields, datatype mismatches.

- **Merge**
  - Type: Merge
  - Role: Combines generated document lines with other invoice data.
  - Input: From Generate DocumentLines and Get Row Details.
  - Output: Merged JSON data.
  - Failures: Data alignment issues.

- **Create JSON**
  - Type: Code (JavaScript)
  - Role: Builds the final JSON payload for SAP PurchaseInvoices endpoint.
  - Logic: Includes invoice date, supplier code (CardCode), and DocumentLines.
  - Input: Merged data.
  - Output: JSON payload.
  - Failures: Missing required fields.

---

#### 1.5 User Confirmation via Telegram

- **Overview:** Sends an inline keyboard message to the user via Telegram asking whether to upload the extracted data to SAP. Processes the callback response.
- **Nodes Involved:** ¿Upload to SAP?, Callback Waiting Answer, Answer?

**Node Details:**

- **¿Upload to SAP?**
  - Type: Telegram
  - Role: Sends a message with inline buttons "Si" and "No" for user confirmation.
  - Config: Chat ID from Telegram message capture; buttons return callback data "respuesta_si" or "respuesta_no".
  - Output: Inline keyboard message sent.
  - Failures: Telegram API errors.

- **Callback Waiting Answer**
  - Type: Telegram Trigger
  - Role: Listens for callback_query updates from Telegram inline buttons.
  - Config: Webhook listens for "callback_query" events.
  - Output: Callback JSON with user response.
  - Failures: Missing or delayed callback events.

- **Answer?**
  - Type: Switch
  - Role: Routes workflow based on user choice ("SI" or "NO").
  - If "SI", proceeds to SAP connection; if "NO", terminates workflow with No Operation node.
  - Failures: Unexpected callback data.

- **No Operation, do nothing**
  - Type: NoOp
  - Role: Ends the workflow branch when user declines.
  - Failures: None.

---

#### 1.6 SAP Integration

- **Overview:** Authenticates to SAP, fetches necessary data from Google Sheets, and posts the purchase invoice JSON to SAP API.
- **Nodes Involved:** Connect to SAP, Get Header, Get Row Details, Generate DocumentLines, Merge, Create JSON, POST PurchaseInvoices

**Node Details:**

- **Connect to SAP**
  - Type: HTTP Request
  - Role: Logs into SAP API with credentials stored in variables and obtains session cookie.
  - Config: POST with JSON body containing username, password, and company DB.
  - Output: Session ID in cookie for subsequent requests.
  - Failures: Auth failure, network errors.

- **POST PurchaseInvoices**
  - Type: HTTP Request
  - Role: Sends the finalized purchase invoice JSON to SAP PurchaseInvoices endpoint.
  - Config: POST with JSON body, includes SAP session cookie for auth.
  - Input: JSON from Create JSON node.
  - Output: SAP response with DocEntry.
  - Failures: HTTP errors, invalid payloads.

---

#### 1.7 Final Notification

- **Overview:** Sends a confirmation message to the user via Telegram indicating successful invoice creation in SAP.
- **Nodes Involved:** PurchaseInvoices created

**Node Details:**

- **PurchaseInvoices created**
  - Type: Telegram
  - Role: Sends confirmation text with SAP DocEntry number to Telegram user.
  - Config: Uses chat ID from original Telegram message callback.
  - Failures: Telegram API issues.

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                             | Input Node(s)                 | Output Node(s)             | Sticky Note                              |
|---------------------------|--------------------------------|--------------------------------------------|------------------------------|----------------------------|-----------------------------------------|
| Trigger Receive Message    | Telegram Trigger                | Entry point for receiving invoice documents | -                            | Capture Telegram Data       |                                         |
| Capture Telegram Data      | Set                            | Extracts metadata and file ID from Telegram | Trigger Receive Message       | File Received              |                                         |
| File Received             | Telegram                       | Acknowledges receipt of invoice document   | Capture Telegram Data         | Download File              |                                         |
| Download File              | Telegram                       | Downloads the invoice file from Telegram   | File Received                | Upload File LlamaIndex     |                                         |
| Upload File LlamaIndex     | HTTP Request                   | Uploads file to LlamaIndex OCR API          | Download File                | Validate Status LlamaIndex |                                         |
| Validate Status LlamaIndex | HTTP Request                   | Polls OCR job status                        | Upload File LlamaIndex        | Status?                    |                                         |
| Status?                   | Switch                        | Routes based on OCR job status              | Validate Status LlamaIndex    | Get Results LlamaIndex, Wait |                                         |
| Wait                      | Wait                           | Waits 3 seconds before polling again       | Status? (PENDING)             | Validate Status LlamaIndex |                                         |
| Get Results LlamaIndex     | HTTP Request                   | Retrieves OCR parsed markdown result       | Status? (SUCCESS)             | OpenAI Chat Model          |                                         |
| OpenAI Chat Model          | LangChain LM Chat OpenAI       | Processes OCR text with GPT-4o              | Get Results LlamaIndex        | Basic LLM Chain            |                                         |
| Basic LLM Chain            | LangChain Chain LLM            | Extracts structured invoice data            | OpenAI Chat Model             | Split Out                  |                                         |
| Structured Output Parser (Example) | LangChain Output Parser Structured | Validates JSON extracted by LLM             | Basic LLM Chain               | Basic LLM Chain (output)   |                                         |
| Split Out                 | Split Out                      | Splits product lines array into separate items | Basic LLM Chain             | Detail                     |                                         |
| Detail                    | Google Sheets                  | Appends product line details to Google Sheets | Split Out                   | Header                     |                                         |
| Header                    | Google Sheets                  | Appends header data to Google Sheets       | Detail                       | ¿Upload to SAP?            |                                         |
| ¿Upload to SAP?            | Telegram                       | Asks user to confirm SAP upload             | Header                       | Callback Waiting Answer    |                                         |
| Callback Waiting Answer    | Telegram Trigger               | Waits for user callback response            | ¿Upload to SAP?              | Answer?                    |                                         |
| Answer?                   | Switch                        | Routes workflow on user decision             | Callback Waiting Answer       | Connect to SAP, No Operation |                                         |
| No Operation, do nothing   | NoOp                          | Ends workflow branch if user declines       | Answer? (NO)                 | -                          |                                         |
| Connect to SAP             | HTTP Request                  | Logs into SAP and obtains session cookie    | Answer? (SI)                 | Get Header                 |                                         |
| Get Header                | Google Sheets                  | Retrieves header data from Google Sheets     | Connect to SAP               | Get Row Details, Merge     |                                         |
| Get Row Details           | Google Sheets                  | Retrieves detailed rows from Google Sheets   | Get Header                  | Generate DocumentLines      |                                         |
| Generate DocumentLines     | Code                          | Converts product lines for SAP API payload   | Get Row Details             | Merge                      |                                         |
| Merge                     | Merge                         | Combines document lines with other data      | Generate DocumentLines, Get Row Details | Create JSON         |                                         |
| Create JSON               | Code                          | Builds final JSON payload for SAP upload     | Merge                       | POST PurchaseInvoices      |                                         |
| POST PurchaseInvoices      | HTTP Request                  | Posts invoice data to SAP PurchaseInvoices   | Create JSON                 | PurchaseInvoices created   |                                         |
| PurchaseInvoices created   | Telegram                      | Notifies user of successful SAP invoice creation | POST PurchaseInvoices    | -                          |                                         |
| Wait                      | Wait                          | Waits 3 seconds before polling               | Status? (PENDING)            | Validate Status LlamaIndex |                                         |
| Merge                     | Merge                         | Combines product lines to JSON                | Generate DocumentLines       | Create JSON                |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node ("Trigger Receive Message")**
   - Event: "message"
   - Credential: Connect with Telegram API OAuth2 credentials.
   - Position: Starting node.

2. **Add a Set node ("Capture Telegram Data")**
   - Extract from incoming message:
     - `message.chat.id` (number)
     - `message.chat` (object)
     - `telegram.file_id` from `message.document.file_id` (string)
     - `message.caption` (string)
   - Connect from "Trigger Receive Message".

3. **Add a Telegram node ("File Received")**
   - Send message "Hemos recibido tu documento y lo estamos procesando..."
   - Chat ID: `={{ $json.message.chat.id }}`
   - Credential: Telegram API
   - Connect from "Capture Telegram Data".

4. **Add a Telegram node ("Download File")**
   - Resource: "file"
   - File ID: `={{ $('Capture Telegram Data').item.json.telegram.file_id }}`
   - Credential: Telegram API
   - Connect from "File Received".

5. **Add HTTP Request node ("Upload File LlamaIndex")**
   - Method: POST
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
   - Content-Type: multipart-form-data
   - Body: Upload binary data from "Download File" node as form data field "file"
   - Headers:
     - Accept: application/json
     - Authorization: `Bearer {{ $vars.llamaindex_apikey }}`
   - Connect from "Download File".

6. **Add HTTP Request node ("Validate Status LlamaIndex")**
   - Method: GET
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`
   - Headers: Same as above
   - Connect from "Upload File LlamaIndex" and from "Wait" node (loop).

7. **Add Switch node ("Status?")**
   - Condition on `$json.status`
   - If equals "SUCCESS", route to "Get Results LlamaIndex"
   - If equals "PENDING", route to "Wait"
   - Connect from "Validate Status LlamaIndex".

8. **Add Wait node ("Wait")**
   - Duration: 3 seconds
   - Connect from "Status?" (PENDING branch) to "Validate Status LlamaIndex".

9. **Add HTTP Request node ("Get Results LlamaIndex")**
   - Method: GET
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`
   - Headers: Same as above
   - Connect from "Status?" (SUCCESS branch).

10. **Add LangChain LM Chat OpenAI node ("OpenAI Chat Model")**
    - Model: "gpt-4o-mini"
    - Connect from "Get Results LlamaIndex".

11. **Add LangChain Chain LLM node ("Basic LLM Chain")**
    - Prompt: Instruct GPT to extract invoice fields into JSON with exact field names and formats.
    - Input: `={{ $json.markdown }}`
    - Connect AI language model from "OpenAI Chat Model".
    - Attach Output Parser node below.

12. **Add LangChain Output Parser Structured node ("Structured Output Parser (Example)")**
    - JSON Schema: Provide example invoice JSON model.
    - Connect to output parser of "Basic LLM Chain".

13. **Add Split Out node ("Split Out")**
    - Field to split out: `output.detalle_productos`
    - Connect from "Basic LLM Chain" output.

14. **Add Google Sheets node ("Detail")**
    - Operation: Append
    - Map product line fields: código, descripcion, cantidad, precio, subtotal, numero_factura.
    - Credentials: Google Sheets OAuth2
    - Connect from "Split Out".

15. **Add Google Sheets node ("Header")**
    - Operation: Append
    - Credentials: Google Sheets OAuth2
    - Connect from "Detail".

16. **Add Telegram node ("¿Upload to SAP?")**
    - Text: "¿Quieres enviar los datos a SAP?"
    - Chat ID: `={{ $('Capture Telegram Data').item.json.message.chat.id }}`
    - Reply Markup: Inline keyboard with buttons "Si" and "No" having callback data "respuesta_si" and "respuesta_no".
    - Credential: Telegram API
    - Connect from "Header".

17. **Add Telegram Trigger node ("Callback Waiting Answer")**
    - Event: "callback_query"
    - Credential: Telegram API
    - Configure webhook.
    - Independent entry node.

18. **Add Switch node ("Answer?")**
    - Condition on `$json.callback_query.data`
    - "SI": route to "Connect to SAP"
    - "NO": route to "No Operation, do nothing"
    - Connect from "Callback Waiting Answer".

19. **Add No Operation node ("No Operation, do nothing")**
    - Connect from "Answer?" (NO branch).

20. **Add HTTP Request node ("Connect to SAP")**
    - Method: POST
    - URL: `={{ $vars.url_sap }}Login`
    - Body JSON:
      ```
      {
        "UserName": "{{ $vars.user_sap }}",
        "Password": "{{ $vars.password_sap }}",
        "CompanyDB": "{{ $vars.company_db }}"
      }
      ```
    - Allow unauthorized certs: true
    - Connect from "Answer?" (SI branch).

21. **Add Google Sheets node ("Get Header")**
    - Operation: Read rows
    - Credentials: Google Sheets OAuth2
    - Connect from "Connect to SAP".

22. **Add Google Sheets node ("Get Row Details")**
    - Operation: Read rows
    - Credentials: Google Sheets OAuth2
    - Connect from "Get Header".

23. **Add Code node ("Generate DocumentLines")**
    - JavaScript code to map product lines into SAP API format:
      ```js
      const items = $input.all();
      const DocumentLines = [];
      for (let i = 0; i < items.length; i++) {
        const item = items[i].json;
        DocumentLines.push({
          ItemCode: item.código,
          Quantity: item.cantidad,
          UnitPrice: item.precio
        });
      }
      return [{ DocumentLines }];
      ```
    - Connect from "Get Row Details".

24. **Add Merge node ("Merge")**
    - Mode: Combine by position
    - Connect from "Generate DocumentLines" and "Get Row Details".

25. **Add Code node ("Create JSON")**
    - JavaScript code to create final SAP payload:
      ```js
      const item = $json;
      return [{
        DocDate: item.fecha_emision,
        DocDueDate: item.fecha_emision,
        CardCode: item.ruc_proveedor,
        DocumentLines: item.DocumentLines
      }];
      ```
    - Connect from "Merge".

26. **Add HTTP Request node ("POST PurchaseInvoices")**
    - Method: POST
    - URL: `={{ $vars.url_sap }}PurchaseInvoices`
    - Body: Raw JSON from "Create JSON"
    - Headers: Include `Cookie: B1SESSION={{ $('Connect to SAP').item.json.SessionId }}`
    - Allow unauthorized certs: true
    - Connect from "Create JSON".

27. **Add Telegram node ("PurchaseInvoices created")**
    - Text: `PurchaseInvoice {{ $json.DocEntry }} creada en SAP correctamente`
    - Chat ID: `={{ $('Callback Waiting Answer').item.json.callback_query.message.chat.id }}`
    - Credential: Telegram API
    - Connect from "POST PurchaseInvoices".

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                  |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow uses LlamaIndex API for OCR parsing; requires an API key stored as workflow variable `llamaindex_apikey`.                | LlamaIndex API: https://www.llamaindex.ai       |
| OpenAI GPT-4o-mini is used for AI extraction, requiring OpenAI credentials configured in n8n.                                     | OpenAI API documentation                         |
| Google Sheets used for temporary storage of invoice details; requires OAuth2 credentials setup.                                   | Google Sheets API documentation                   |
| SAP API credentials and URLs are stored as workflow variables: `url_sap`, `user_sap`, `password_sap`, `company_db`.              | SAP B1 Service Layer API                          |
| Telegram API credentials must be configured to enable sending and receiving messages and callbacks.                               | Telegram Bot API docs                             |
| Inline keyboard buttons in Telegram enable end-user confirmation for SAP upload, improving workflow control.                      | Telegram Inline Keyboard: https://core.telegram.org/bots/api#inlinekeyboardmarkup |
| The workflow handles asynchronous polling of OCR job status with retries and wait nodes to avoid excessive API calls.            | n8n Wait node docs                               |
| JSON schema validation for invoice extraction ensures consistent output for downstream SAP integration.                          | LangChain Output Parser Structured node docs    |

---

**Disclaimer:** The text provided is exclusively generated by an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.