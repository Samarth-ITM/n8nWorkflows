Receipt Scanning & Analysis Workflow

https://n8nworkflows.xyz/workflows/receipt-scanning---analysis-workflow-7990


# Receipt Scanning & Analysis Workflow

### 1. Workflow Overview

This workflow automates the processing of newly added invoice files in a specific Google Drive folder. It extracts text from scanned invoice PDFs, analyzes their content using AI models, structures the extracted data, and appends the relevant invoice details into a Google Sheets document for record-keeping and further processing.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception (File Detection and Download):** Detects newly created files in a designated Google Drive folder and downloads them.
- **1.2 Text Extraction:** Uses an AI-powered OCR model to extract text content from the downloaded invoice PDFs.
- **1.3 AI Invoice Analysis:** Applies an AI agent with embedded OpenAI and structured output parsers to analyze and interpret the invoice text.
- **1.4 Data Storage:** Appends the structured invoice data into a Google Sheets spreadsheet for tracking.
- **1.5 Auxiliary:** Includes a sticky note with tutorial information unrelated to main logic.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (File Detection and Download)

- **Overview:**  
  Watches a specific Google Drive folder for newly created files (invoices) and downloads each detected file for further processing.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - HTTP Request

- **Node Details:**  

  - **Google Drive Trigger**  
    - Type: Trigger node, listens for Google Drive events  
    - Configuration: Watches the folder with ID `12aSoGo009rsrhyJ_a4VP84m_NrWCsA3O` (named "Invoices") for new files created (`fileCreated` event), polling every minute  
    - Credentials: Google Drive OAuth2  
    - Inputs: None (trigger)  
    - Outputs: Emits metadata of the newly created file(s)  
    - Edge Cases:  
      - Folder access permissions issues  
      - Polling delay impacts near real-time detection  
      - Large file creation events batching  
      - Token expiration or revoked credentials causing trigger failure

  - **HTTP Request**  
    - Type: HTTP node to download file content  
    - Configuration:  
      - Constructs URL dynamically using file ID from trigger (`https://www.googleapis.com/drive/v3/files/{{ $json.id }}?alt=media`)  
      - Response format set to download the file as a binary (`responseFormat: file`)  
      - Uses Google Drive OAuth2 credentials (different account from trigger, ensure consistent permissions)  
    - Inputs: Receives file metadata from Google Drive Trigger  
    - Outputs: Binary file content of the invoice PDF  
    - Edge Cases:  
      - File deletion between detection and download  
      - API rate limits or timeouts  
      - Credential mismatches or insufficient permissions  
      - Large file size causing memory or timeout issues  
    - Version Notes: Node version 4.2 supports improved binary handling

#### 2.2 Text Extraction

- **Overview:**  
  Extracts textual content from the downloaded invoice PDF using an AI-powered OCR or text extraction model.

- **Nodes Involved:**  
  - Extract text

- **Node Details:**  

  - **Extract text**  
    - Type: Mistral AI node (AI text extraction/OCR)  
    - Configuration: Default options (no custom parameters specified)  
    - Credentials: Mistral Cloud API  
    - Inputs: Binary PDF file from HTTP Request node  
    - Outputs: JSON object containing extracted text, including page-wise markdown representation under `pages[0].markdown`  
    - Edge Cases:  
      - Poor quality scans causing inaccurate text extraction  
      - Unsupported file format or corrupted PDF  
      - API errors or authentication failures  
      - Latency or throttling on Mistral API side

#### 2.3 AI Invoice Analysis

- **Overview:**  
  Uses an AI agent incorporating OpenAI GPT-4 and a structured output parser to analyze the extracted invoice text and produce a structured JSON with invoice details.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**  

  - **AI Agent**  
    - Type: Langchain AI agent node  
    - Configuration:  
      - Prompt: Instructs the AI as an invoice analysis expert and injects the extracted markdown text from the first page of the invoice (`{{ $json.pages[0].markdown }}`)  
      - Prompt type: Define (custom prompt)  
      - Output parser enabled to parse AI output into structured JSON  
    - Inputs: Extracted text JSON from Mistral AI node  
    - Outputs: Structured JSON with invoice fields  
    - Edge Cases:  
      - Prompt formatting or expression errors  
      - Unexpected or incomplete text input causing misinterpretation  
      - AI model unavailability or latency

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI chat completions node  
    - Configuration:  
      - Model: GPT-4.1-mini (a variant of GPT-4 optimized for smaller size)  
      - No additional options specified  
    - Credentials: OpenAI API key  
    - Inputs: Connected as AI language model for the AI Agent node  
    - Outputs: Receives AI completions for the agent's prompt  
    - Edge Cases:  
      - API rate limits or quota exhaustion  
      - Model version deprecation or changes  
      - Network or authentication failures

  - **Structured Output Parser**  
    - Type: Langchain structured output parser node  
    - Configuration:  
      - Provides an example JSON schema for invoice data fields: invoiceNumber, From, To, invoiceDate, dueDate, shortDescription  
      - Ensures AI output conforms to this structure for downstream use  
    - Inputs: Connected as AI output parser for the AI Agent node  
    - Outputs: Parsed structured data used by next node  
    - Edge Cases:  
      - Parsing errors if AI output does not match schema  
      - Schema changes requiring updates in parser config

#### 2.4 Data Storage

- **Overview:**  
  Appends the structured invoice data into a Google Sheet to maintain a log of processed invoices.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**  

  - **Append row in sheet**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append  
      - Target Sheet: Sheet1 (`gid=0`) of spreadsheet ID `14tyy2at8ZtfoQq4zGYph4DG7omv9uA9aXGH5KnQbPgs`  
      - Columns mapped: Invoice Number, From, To, Invoice Date, Due Date, Short Description, Link to File  
      - Values sourced from `AI Agent` node output fields and Google Drive Trigger's file link (`$('Google Drive Trigger').item.json.webViewLink`)  
      - No type conversion or matching columns enabled  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Structured invoice data JSON from AI Agent node  
    - Outputs: Confirmation of row append  
    - Edge Cases:  
      - Sheet access/permission issues  
      - Data type mismatches or missing fields  
      - Spreadsheet quota or API limits  
      - Network errors or credential expiration

#### 2.5 Auxiliary

- **Overview:**  
  Provides tutorial link and author information as a sticky note for user reference.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  - **Sticky Note**  
    - Type: Visual annotation node, no runtime impact  
    - Content:  
      ```
      ## n8n Starter Session Tutorial

      [Watch the Tutorials](https://www.youtube.com/playlist?list=PLWYu7XaUG3XOJwOOGiX89SQ_w67vw3dq7)

      Tutor: [Aemal Sayer](https://aemalsayer.com)
      ```  
    - Position: Top-left area, separate from main flow  
    - Edge Cases: None

---

### 3. Summary Table

| Node Name             | Node Type                                   | Functional Role                  | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                              |
|-----------------------|---------------------------------------------|--------------------------------|-----------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Google Drive Trigger   | Google Drive Trigger (trigger)               | Detect new invoice files        | None                  | HTTP Request          |                                                                                                        |
| HTTP Request          | HTTP Request (file download)                 | Download invoice PDF            | Google Drive Trigger  | Extract text          |                                                                                                        |
| Extract text           | Mistral AI (OCR/text extraction)             | Extract text from PDF           | HTTP Request          | AI Agent              |                                                                                                        |
| AI Agent               | Langchain AI Agent                            | Analyze invoice text            | Extract text, OpenAI Chat Model, Structured Output Parser | Append row in sheet  |                                                                                                        |
| OpenAI Chat Model      | Langchain OpenAI Chat Completion              | Provide AI language model       | None (invoked by AI Agent) | AI Agent           |                                                                                                        |
| Structured Output Parser| Langchain Structured Output Parser            | Parse AI output                 | None (invoked by AI Agent) | AI Agent           |                                                                                                        |
| Append row in sheet    | Google Sheets Append Row                      | Store parsed invoice data       | AI Agent              | None                  |                                                                                                        |
| Sticky Note            | Sticky Note (annotation)                      | Tutorial link and credits       | None                  | None                  | ## n8n Starter Session Tutorial. [Watch the Tutorials](https://www.youtube.com/playlist?list=PLWYu7XaUG3XOJwOOGiX89SQ_w67vw3dq7). Tutor: [Aemal Sayer](https://aemalsayer.com) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Set event to `fileCreated`  
   - Set `triggerOn` to `specificFolder`  
   - Specify the folder ID to watch: `12aSoGo009rsrhyJ_a4VP84m_NrWCsA3O` (your invoices folder)  
   - Set polling interval to every 1 minute  
   - Add Google Drive OAuth2 credential with appropriate permissions

2. **Create HTTP Request node**  
   - Type: HTTP Request  
   - Connect input from Google Drive Trigger node  
   - Set URL to: `https://www.googleapis.com/drive/v3/files/{{ $json.id }}?alt=media` (use expression to insert file ID dynamically)  
   - Under options → response, set responseFormat to `file` to download binary content  
   - Authentication: Use Google Drive OAuth2 credentials (ensure same or compatible account as trigger)  
   - Node version: Prefer version 4.2+ for improved binary support

3. **Create Extract text node**  
   - Type: Mistral AI (or equivalent OCR/text extraction AI node)  
   - Connect input from HTTP Request node (binary PDF)  
   - Use default options or configure as needed for your OCR model  
   - Attach Mistral Cloud API credentials

4. **Create OpenAI Chat Model node**  
   - Type: Langchain OpenAI Chat Model  
   - Select model: `gpt-4.1-mini` or suitable GPT-4 variant  
   - Attach OpenAI API credentials (API key with chat completion access)

5. **Create Structured Output Parser node**  
   - Type: Langchain Structured Output Parser  
   - Provide JSON schema example for invoice extraction fields:  
     ```json
     {
       "invoiceNumber": "INV123",
       "From": "John street 23, 12443 Berlin, Germany",
       "To": "Smith street 33, 23222 Berlin, Germany",
       "invoiceDate": "20/08/2025",
       "dueDate": "25/08/2025",
       "shortDescription": "It is an invoice about web development costs"
     }
     ```
   - No additional parameters needed

6. **Create AI Agent node**  
   - Type: Langchain AI Agent  
   - Set prompt type to `define`  
   - Insert prompt text:  
     ```
     You are an expert in analyzing invoices.

     Here is an invoice that you should analyze: {{ $json.pages[0].markdown }}
     ```  
   - Enable output parser and select the structured output parser node  
   - Connect AI language model input to the OpenAI Chat Model node  
   - Connect AI output parser input to the Structured Output Parser node  
   - Connect input from Extract text node (which delivers OCR output)

7. **Create Append row in sheet node**  
   - Type: Google Sheets  
   - Set operation to `append`  
   - Configure document ID to target your invoice tracking spreadsheet (e.g., `14tyy2at8ZtfoQq4zGYph4DG7omv9uA9aXGH5KnQbPgs`)  
   - Set the sheet name or gid (e.g., `gid=0`)  
   - Define columns and map them to AI Agent output fields:  
     - Invoice Number → `{{ $json.output.invoiceNumber }}`  
     - From → `{{ $json.output.From }}`  
     - To → `{{ $json.output.To }}`  
     - Invoice Date → `{{ $json.output.invoiceDate }}`  
     - Due Date → `{{ $json.output.dueDate }}`  
     - Short Description → `{{ $json.output.shortDescription }}`  
     - Link to File → `{{ $('Google Drive Trigger').item.json.webViewLink }}`  
   - Attach Google Sheets OAuth2 credentials

8. **Connect the nodes in order:**  
   - Google Drive Trigger → HTTP Request → Extract text → AI Agent → Append row in sheet  
   - OpenAI Chat Model and Structured Output Parser nodes connect to AI Agent as language model and output parser respectively

9. **(Optional) Create Sticky Note node**  
   - Add for documentation or reference purposes with tutorial links and credits

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| n8n Starter Session Tutorial with step-by-step videos                                                        | [YouTube Playlist](https://www.youtube.com/playlist?list=PLWYu7XaUG3XOJwOOGiX89SQ_w67vw3dq7)     |
| Tutorial author and expert contact                                                                           | [Aemal Sayer](https://aemalsayer.com)                                                             |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.