Parse PDF with LlamaParse and save to Airtable

https://n8nworkflows.xyz/workflows/parse-pdf-with-llamaparse-and-save-to-airtable-2661


# Parse PDF with LlamaParse and save to Airtable

### 1. Workflow Overview

This workflow automates the extraction and structured storage of invoice data from PDF files uploaded to a specific Google Drive folder. It is designed primarily for finance teams, accountants, and business operations managers who want to reduce manual data entry and increase accuracy in invoice processing.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Detects newly created invoice files in a designated Google Drive folder and downloads them.
- **1.2 Invoice Parsing:** Uploads the invoice files to LlamaParse via an HTTP request for AI-powered parsing, with a webhook set up to receive parsed results.
- **1.3 Data Processing and Extraction:** Uses OpenAI API to further extract and structure line item details from the parsed data.
- **1.4 Storage in Airtable:** Creates records for invoices and their individual line items in Airtable for organized tracking.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block monitors a specific Google Drive folder for new invoice files. When a new file is detected, it downloads the file content to prepare it for parsing.

**Nodes Involved:**  
- Google Drive Trigger  
- Google Drive

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node for Google Drive  
  - Configuration: Watches a specific folder by folder ID (`1IC39VXU8rewBU85offxYlBd9QlYzf8S7`) with a polling interval of every minute, triggering on new file creation events.  
  - Inputs: None (trigger node)  
  - Outputs: Passes the detected file metadata (including file ID) to the next node.  
  - Edge Cases: Missed triggers if polling interval too long; permissions errors if OAuth token lacks folder access.  
  - Credentials: Google Drive OAuth2 account (named "Google Drive account 2").

- **Google Drive**  
  - Type: Google Drive file download node  
  - Configuration: Uses the file ID from the trigger to download the file content.  
  - Inputs: File metadata from Google Drive Trigger  
  - Outputs: File binary data and metadata to be uploaded to LlamaParse.  
  - Edge Cases: File unavailable or deleted before download; OAuth token expiration; large file size causing timeouts.  
  - Credentials: Same Google Drive OAuth2 account.

---

#### 2.2 Invoice Parsing

**Overview:**  
Uploads the downloaded invoice file to LlamaParse via an HTTP POST request with multipart form data. The request includes a webhook URL to receive the parsed invoice data asynchronously.

**Nodes Involved:**  
- Upload File  
- Webhook

**Node Details:**

- **Upload File**  
  - Type: HTTP Request  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.cloud.llamaindex.ai/api/parsing/upload`  
    - Content-Type: multipart/form-data with the file attached as binary data under the name `file`.  
    - Body Parameters: includes `webhook_url` pointing to the Webhook node URL, disables OCR and image extraction to optimize parsing for text-based PDFs.  
    - Headers: Accepts JSON, requires an Authorization Bearer token (to be replaced by user), includes a parsing instruction specifically requesting extraction of invoice line items (Name, Quantity, Unit Price, Amount).  
  - Inputs: Binary file data from Google Drive node  
  - Outputs: Response from LlamaParse API (likely contains parsing initiation confirmation).  
  - Edge Cases: Network issues, authorization failure due to missing or invalid API key, malformed body or missing webhook URL, LlamaParse service downtime.  
  - Credentials: None explicitly shown; Authorization header must be set with a valid Bearer token.

- **Webhook**  
  - Type: Webhook (POST)  
  - Configuration: Listens on path `0f7f5ebb-8b66-453b-a818-20cc3647c783` for POST requests containing parsed invoice data from LlamaParse.  
  - Inputs: Receives JSON payload from LlamaParse with parsed invoice data including line items.  
  - Outputs: Passes JSON to the next processing node.  
  - Edge Cases: Missed webhook calls if n8n instance is down; invalid or malformed JSON payloads; security considerations for webhook exposure.  
  - Sub-workflow: This is the asynchronous callback endpoint for the upload process.

---

#### 2.3 Data Processing and Extraction

**Overview:**  
Processes the raw parsed data from LlamaParse by requesting OpenAI to extract and structure line item details according to a defined JSON schema, then formats the output for Airtable insertion.

**Nodes Involved:**  
- Set Fields  
- OpenAI - Extract Line Items  
- Process Line Items

**Node Details:**

- **Set Fields**  
  - Type: Set node  
  - Configuration: Assigns two string fields:  
    - `prompt`: A short instruction string for OpenAI to process parsed data and return only needed information.  
    - `schema`: A JSON schema definition describing the expected structure of invoice line items (an array of objects with description, quantity, unit price, and amount, all strings).  
  - Inputs: JSON from Webhook node  
  - Outputs: Adds prompt and schema for the OpenAI request.  
  - Edge Cases: Improper schema formatting causing OpenAI errors; missing or empty input JSON.

- **OpenAI - Extract Line Items**  
  - Type: HTTP Request to OpenAI API  
  - Configuration:  
    - URL: OpenAI Chat Completion endpoint `https://api.openai.com/v1/chat/completions`  
    - Method: POST  
    - Body: JSON including:  
      - Model: `gpt-4o-mini`  
      - Messages:  
        - System role message with the prompt from Set Fields  
        - User role message with stringified raw line items array from webhook JSON  
      - Response format: JSON schema as defined in Set Fields, enforcing a strict structure.  
    - Authentication: Uses predefined OpenAI API credentials.  
  - Inputs: prompt and schema from Set Fields node  
  - Outputs: OpenAI response containing structured line items data.  
  - Edge Cases: API key missing or invalid; OpenAI API rate limiting or downtime; malformed JSON body or response; schema validation errors.  
  - Credentials: OpenAI API key.

- **Process Line Items**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Parses the `content` field from OpenAI response JSON, which is stringified JSON of line items.  
    - Converts this string into an array of item objects.  
    - Outputs each item as a separate JSON object for subsequent Airtable insertion.  
  - Inputs: OpenAI JSON response  
  - Outputs: Array of individual line item JSON objects for record creation.  
  - Edge Cases: JSON parsing failures; empty or missing content field; unexpected response structure.

---

#### 2.4 Storage in Airtable

**Overview:**  
Creates a main invoice record in Airtable, then iterates over each extracted line item to create corresponding linked records in a separate Airtable table.

**Nodes Involved:**  
- Create Invoice  
- Create Line Item

**Node Details:**

- **Create Invoice**  
  - Type: Airtable node (create operation)  
  - Configuration:  
    - Base: User’s Airtable base (example ID `appndgSF4faN4jPXi`)  
    - Table: Invoices table (example ID `tbloPc7Eay4Cvwysq`)  
    - Fields:  
      - Name (string) - mapped from parsed data (implicit from context)  
      - Line Items (array) - likely placeholder or linked field to line items  
    - Operation: Create a new invoice record with parsed invoice details.  
  - Inputs: Structured JSON from OpenAI extraction  
  - Outputs: Created invoice record with its Airtable ID for linkage.  
  - Edge Cases: Airtable API quota limits; invalid data types; missing required fields; credential expiration.  
  - Credentials: Airtable Personal Access Token.

- **Create Line Item**  
  - Type: Airtable node (create operation)  
  - Configuration:  
    - Base: Same Airtable base as invoice  
    - Table: Line Items table (example ID `tblIuVR9ocAomznzK`)  
    - Fields:  
      - Description (string)  
      - Qty (number) parsed from string with trimming and dollar sign removal  
      - Unit price (number) similarly parsed  
      - Amount (number) similarly parsed  
      - Invoices (array) linking to the parent invoice record created above by its ID  
    - Operation: Creates individual line item records linked to the invoice.  
  - Inputs: Each item JSON from Process Line Items node  
  - Outputs: Created line item records in Airtable.  
  - Edge Cases: Data parsing errors; missing or malformed numeric fields; Airtable API limits.  
  - Credentials: Airtable Personal Access Token.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                            | Input Node(s)                 | Output Node(s)                | Sticky Note                                                    |
|----------------------------|------------------------|--------------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------|
| Google Drive Trigger        | Google Drive Trigger   | Detects new invoice files in Drive folder | None                         | Google Drive                 |                                                               |
| Google Drive               | Google Drive           | Downloads invoice file from Drive          | Google Drive Trigger          | Upload File                  | ### Replace Google Drive connection                            |
| Upload File                | HTTP Request           | Uploads invoice file to LlamaParse API     | Google Drive                 | Webhook                     | ### Replace API key in header                                  |
| Webhook                   | Webhook (POST)         | Receives parsed invoice data from LlamaParse | Upload File                  | Set Fields                  |                                                               |
| Set Fields                | Set                    | Sets prompt and JSON schema for OpenAI     | Webhook                      | OpenAI - Extract Line Items |                                                               |
| OpenAI - Extract Line Items| HTTP Request           | Requests OpenAI to extract line items      | Set Fields                   | Create Invoice              | ### Replace OpenAI connection                                  |
| Create Invoice            | Airtable               | Creates invoice record in Airtable          | OpenAI - Extract Line Items  | Process Line Items          | ### Replace Airtable connection                                |
| Process Line Items        | Code                   | Parses OpenAI response into line items array | Create Invoice              | Create Line Item            |                                                               |
| Create Line Item          | Airtable               | Creates individual line item records linked to invoice | Process Line Items          | None                        | ### Replace Airtable connection                                |
| Sticky Note1              | Sticky Note            | Scenario 1 note                            | None                         | None                        |                                                               |
| Sticky Note               | Sticky Note            | Scenario 2 note                            | None                         | None                        |                                                               |
| Sticky Note2              | Sticky Note            | Reminder to replace Google Drive connection | None                         | None                        | ### Replace Google Drive connection                            |
| Sticky Note3              | Sticky Note            | Reminder to replace API key in header      | None                         | None                        | ### Replace API key in header                                  |
| Sticky Note4              | Sticky Note            | Reminder to replace OpenAI connection      | None                         | None                        | ### Replace OpenAI connection                                  |
| Sticky Note5              | Sticky Note            | Reminder to replace Airtable connection    | None                         | None                        | ### Replace Airtable connection                                |
| Sticky Note6              | Sticky Note            | Setup steps overview                        | None                         | None                        |                                                               |
| Sticky Note7              | Sticky Note            | Branding and project overview               | None                         | None                        |                                                               |
| Sticky Note9              | Sticky Note            | Video setup guide link                       | None                         | None                        | ### ... or watch set up video [7 min] https://youtu.be/E4I0nru-fa8 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Configure:  
     - Event: `fileCreated`  
     - Trigger on: `specificFolder`  
     - Folder to watch: Enter your invoices folder ID in Google Drive  
     - Poll times: Every minute  
   - Set credentials with your Google Drive OAuth2 account.

2. **Create Google Drive Node**  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: Set to `={{ $json.id }}` from the trigger output  
   - Use same Google Drive credentials.

3. **Create HTTP Request Node ("Upload File")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cloud.llamaindex.ai/api/parsing/upload`  
   - Content-Type: `multipart/form-data`  
   - Body Parameters:  
     - `webhook_url`: Set to your n8n instance webhook URL (step 5)  
     - `file`: Select `formBinaryData` type, input data field name `data` (from Google Drive download)  
     - `disable_ocr`: `true`  
     - `disable_image_extraction`: `True`  
   - Header Parameters:  
     - `accept`: `application/json`  
     - `Authorization`: `Bearer YOUR_API_KEY` (replace with your LlamaParse API key)  
     - `parsing_instruction`: `Please extract invoice line items: Name, Quantity, Unit Price, Amount`  
   - No authentication node, header carries token.

4. **Create Webhook Node**  
   - Type: Webhook (POST)  
   - Path: Generate a unique path (e.g., `0f7f5ebb-8b66-453b-a818-20cc3647c783`)  
   - This node receives parsed invoice data from LlamaParse asynchronously.

5. **Create Set Node ("Set Fields")**  
   - Type: Set  
   - Add two fields:  
     - `prompt` (string): A message like "Please, process parsed data and return only needed."  
     - `schema` (string): JSON schema describing the expected line items structure (copy schema from workflow)  
   - Connect Webhook output to this node.

6. **Create HTTP Request Node ("OpenAI - Extract Line Items")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Authentication: OpenAI API credentials  
   - Body (JSON):  
     - Model: `gpt-4o-mini`  
     - Messages: system role with prompt from Set Fields, user role with stringified line items JSON from webhook  
     - Response format: JSON schema from Set Fields  
   - Connect Set Fields output here.

7. **Create Code Node ("Process Line Items")**  
   - Type: Code (JavaScript)  
   - Paste provided code to parse OpenAI response content into individual item JSON objects.  
   - Connect OpenAI node output.

8. **Create Airtable Node ("Create Invoice")**  
   - Type: Airtable (Create)  
   - Configure with your Airtable base and Invoices table  
   - Map fields such as Name and Line Items from parsed data  
   - Connect OpenAI node output here.

9. **Create Airtable Node ("Create Line Item")**  
   - Type: Airtable (Create)  
   - Configure with your Airtable base and Line Items table  
   - Map fields: Description, Qty, Unit price, Amount (parse strings to numbers), and link to parent invoice record ID from "Create Invoice" node  
   - Connect output from Process Line Items node.

10. **Connect nodes in order:**  
    - Google Drive Trigger → Google Drive → Upload File → (asynchronously) Webhook → Set Fields → OpenAI → Create Invoice → Process Line Items → Create Line Item

11. **Credentials Setup:**  
    - Google Drive OAuth2 account with access to target folder  
    - LlamaParse API key configured in HTTP header  
    - OpenAI API key configured in HTTP Request node (OpenAI)  
    - Airtable Personal Access Token with write access to bases and tables

12. **Testing and Validation:**  
    - Upload a test invoice PDF to the Google Drive folder  
    - Confirm that the webhook is triggered with parsed data  
    - Verify OpenAI extracts line items correctly  
    - Check Airtable for created invoice and linked line item records

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Comprehensive video guide detailing invoice parsing automation with n8n and LlamaParse                                          | [YouTube Video](https://youtu.be/E4I0nru-fa8)                                                      |
| Workflow ideal for finance teams, accountants, and business operations managers to reduce manual invoice processing errors      | --                                                                                                 |
| Replace all placeholder API keys and credentials before running the workflow                                                    | Sticky Notes in workflow highlight these replacements                                               |
| Branding and project credits: Made by Mark Shcherbakov from 5minAI community                                                     | [LinkedIn](https://www.linkedin.com/in/marklowcoding/), [5minAI Community](https://www.skool.com/5minai) |
| Watch the 7-minute setup video for a practical walkthrough                                                                      | [YouTube Video](https://youtu.be/E4I0nru-fa8)                                                      |

---

This documentation provides a detailed understanding of the workflow components, their interactions, and setup instructions to reproduce or modify the workflow confidently. It also highlights potential failure points and necessary credential configurations critical for stable operation.