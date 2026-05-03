Extract Invoice Data from Slack PDFs to Google Sheets with GPT-4o

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-slack-pdfs-to-google-sheets-with-gpt-4o-8431


# Extract Invoice Data from Slack PDFs to Google Sheets with GPT-4o

### 1. Workflow Overview

This workflow automates the extraction of invoice data from PDF files uploaded to a specific Slack channel and appends the structured data into a Google Sheets document. Leveraging AI (GPT-4o) for parsing, it extracts key invoice fields such as billing entity, email, total amount, invoice date, and due date. After successful data extraction and storage, it sends a confirmation summary back to the Slack channel.

The workflow’s logical blocks are:

- **1.1 Input Reception:** Triggered by a PDF invoice uploaded in a designated Slack channel.
- **1.2 PDF Retrieval and Text Extraction:** Downloads the PDF from Slack and extracts its textual content.
- **1.3 AI-Powered Invoice Parsing:** Uses GPT-4o to identify and structure relevant invoice fields from extracted text.
- **1.4 Data Structuring:** Formats the AI output into a consistent JSON structure.
- **1.5 Data Storage:** Appends the structured invoice data as a new row in a specified Google Sheet.
- **1.6 Slack Notification:** Posts a summary message back to Slack confirming data capture.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens to a designated Slack channel for new messages containing PDF invoices; initiates the workflow upon detection.
- **Nodes Involved:**  
  - Receive invoice pdf (Slack Trigger)
- **Node Details:**  
  - **Receive invoice pdf**  
    - Type: Slack Trigger  
    - Configuration: Triggered on new message event in Slack channel with ID `C09ECEMB012` (invoice channel).  
    - Credentials: Slack API credentials for authentication.  
    - Inputs: None (trigger node).  
    - Outputs: JSON containing Slack message data including file metadata.  
    - Potential failures: Missing permissions to read files, webhook misconfiguration, channel ID invalid or changed, no files attached in message.

#### 2.2 PDF Retrieval and Text Extraction

- **Overview:** Downloads the private Slack URL of the uploaded PDF and extracts its textual content for further processing.
- **Nodes Involved:**  
  - Fetch the pdf (HTTP Request)  
  - Extract information from pdf (Extract from File)
- **Node Details:**  
  - **Fetch the pdf**  
    - Type: HTTP Request  
    - Configuration: Downloads the file from the URL stored in the Slack message (`$json.files[0].url_private_download`).  
    - Authentication: Uses Slack API credentials to authorize file download.  
    - Input: Output of Slack trigger node.  
    - Output: Binary data of the PDF file.  
    - Potential failures: Expired or invalid download URL, network timeout, Slack API rate limits.  
  - **Extract information from pdf**  
    - Type: Extract from File  
    - Configuration: Extracts text from the binary PDF data property named `data`.  
    - Input: Binary data from HTTP Request node.  
    - Output: Text content extracted from PDF.  
    - Potential failures: Unsupported PDF format, corrupted file, extraction errors.

#### 2.3 AI-Powered Invoice Parsing

- **Overview:** Sends the extracted text to GPT-4o to identify and extract structured invoice details.
- **Nodes Involved:**  
  - AI model (OpenAI GPT-4o)  
  - Extract invoice information (LangChain Agent)
- **Node Details:**  
  - **AI model**  
    - Type: OpenAI GPT-4o via LangChain Language Model node  
    - Configuration: Model set to `gpt-4o`, no additional options specified.  
    - Credentials: OpenAI API key.  
    - Input: Text extracted from PDF.  
    - Output: Raw AI response containing extracted data.  
    - Potential failures: API key issues, rate limits, model unavailability, malformed prompt.  
  - **Extract invoice information**  
    - Type: LangChain Agent with output parser  
    - Configuration: Prompt instructs AI to extract these fields: billing to, email, total amount, invoice date, due date from input text.  
    - Input: AI model output.  
    - Output: Structured JSON with invoice fields.  
    - Potential failures: AI misinterpretation, JSON parsing errors, incomplete fields.

#### 2.4 Data Structuring

- **Overview:** Parses and enforces a defined JSON schema on the AI output, ensuring consistent data formatting.
- **Nodes Involved:**  
  - Structure Output (LangChain Structured Output Parser)
- **Node Details:**  
  - **Structure Output**  
    - Type: LangChain Output Parser Structured  
    - Configuration: Uses a JSON schema example specifying expected invoice fields and formats (billing to, total amount, invoice date, due date, email).  
    - Input: Output from AI agent node.  
    - Output: Well-structured JSON confirming to schema.  
    - Potential failures: Schema mismatch, invalid JSON format, missing data fields.

#### 2.5 Data Storage

- **Overview:** Appends the structured invoice data as a new row in a specified Google Sheet for record keeping.
- **Nodes Involved:**  
  - Append row in sheet (Google Sheets)
- **Node Details:**  
  - **Append row in sheet**  
    - Type: Google Sheets node  
    - Configuration: Operation set to append, targeting a specified Google Sheets document and sheet name (values set dynamically or from credentials).  
    - Credentials: Google Sheets OAuth2 account.  
    - Input: Structured JSON invoice data.  
    - Output: Confirmation of row append operation.  
    - Potential failures: Credential expiration, sheet or document ID invalid, insufficient permissions, rate limits, data formatting errors.

#### 2.6 Slack Notification

- **Overview:** Sends a message back to the Slack channel summarizing the extracted invoice details as confirmation.
- **Nodes Involved:**  
  - Send a message (Slack node)
- **Node Details:**  
  - **Send a message**  
    - Type: Slack message node  
    - Configuration: Posts to the same Slack channel ID `C09ECEMB012`. Message text template includes invoice date, billing entity, email, total amount, and due date from JSON data.  
    - Credentials: Slack API credentials.  
    - Input: Output from Google Sheets append node (invoice data).  
    - Output: Slack message post confirmation.  
    - Potential failures: Message post denied due to permissions, invalid channel, Slack API rate limits.

---

### 3. Summary Table

| Node Name                | Node Type                            | Functional Role                    | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                                  |
|--------------------------|------------------------------------|----------------------------------|---------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| Receive invoice pdf       | Slack Trigger                      | Input reception                  | -                         | Fetch the pdf                | See sticky note: Workflow overview and usage instructions                                                    |
| Fetch the pdf            | HTTP Request                      | Download PDF                    | Receive invoice pdf       | Extract information from pdf  | See sticky note: Workflow overview and usage instructions                                                    |
| Extract information from pdf | Extract from File               | Text extraction from PDF        | Fetch the pdf             | Extract invoice information   | See sticky note: Workflow overview and usage instructions                                                    |
| AI model                 | LangChain Language Model (OpenAI) | AI-powered parsing              | Extract information from pdf | Extract invoice information   | See sticky note: Workflow overview and usage instructions                                                    |
| Extract invoice information | LangChain Agent with parser      | Invoice data extraction         | AI model                  | Append row in sheet           | See sticky note: Workflow overview and usage instructions                                                    |
| Structure Output         | LangChain Output Parser Structured | Data structuring                | Extract invoice information | Extract invoice information   |                                                                                                              |
| Append row in sheet      | Google Sheets                     | Data storage                   | Extract invoice information | Send a message               |                                                                                                              |
| Send a message           | Slack Node                       | Slack confirmation notification | Append row in sheet       | -                            |                                                                                                              |
| Sticky Note              | Sticky Note                      | Documentation                   | -                         | -                            | ## How it works: Detailed workflow explanation, use instructions, and requirements                          |
| Sticky Note1             | Sticky Note                      | Documentation                   | -                         | -                            | ## Workflow intro: Purpose and business value overview                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Type: Slack Trigger  
   - Trigger event: New message in channel  
   - Channel ID: `C09ECEMB012` (set your target channel)  
   - Credentials: Authenticate with Slack API having read file permissions  
   - Purpose: Initiates workflow when invoice PDF uploaded  

2. **Add HTTP Request Node to Fetch PDF**  
   - Type: HTTP Request  
   - URL: `={{ $json.files[0].url_private_download }}` (extract download URL from Slack message JSON)  
   - Authentication: Use Slack API credentials (same as trigger) for authorized download  
   - Response type: File (binary)  
   - Connect input from Slack Trigger node  

3. **Add Extract from File Node**  
   - Type: Extract from File  
   - Operation: PDF text extraction  
   - Binary Property Name: `data` (output from HTTP Request node)  
   - Connect input from HTTP Request node  

4. **Add LangChain Language Model Node (OpenAI GPT-4o)**  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o`  
   - Credentials: OpenAI API key with access to GPT-4o model  
   - Connect input from Extract from File node  

5. **Add LangChain Agent Node for Invoice Extraction**  
   - Type: LangChain Agent node with output parser  
   - Prompt:  
     ```
     please read and understand the input data({{ $json.text }}). I would like you to extract billing to, email, total amount, invoice date and due date.
     ```  
   - System Message: blank  
   - Connect input from LangChain LM node  

6. **Add LangChain Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - JSON Schema Example:  
     ```
     [{
       "billing to": "Toshiki Hirao",
       "total_amount": "1000",
       "invoice_date": "2025/04/29",
       "due_date" : "2025/05/29",
       "email": "xxx@yyy.jp"
     }]
     ```  
   - Connect input from LangChain Agent node  

7. **Add Google Sheets Node to Append Row**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Specify target Google Sheet ID  
   - Sheet Name: Specify target sheet name within the document  
   - Credentials: Google Sheets OAuth2 account with write access  
   - Connect input from Structured Output node  

8. **Add Slack Message Node for Confirmation**  
   - Type: Slack  
   - Operation: Send a message  
   - Channel ID: `C09ECEMB012` (same Slack channel)  
   - Message Text:  
     ```
     =---
     Invoice date: {{ $json.invoice_date }}
     Billing to: {{ $json['billing to'] }}
     Email: {{ $json.email }}
     Total Amount: {{ $json.total_amount }}
     Due date: {{ $json.due_date }}
     ```  
   - Credentials: Slack API with chat.postMessage permission  
   - Connect input from Google Sheets append node  

9. **Test the workflow**  
   - Upload a sample invoice PDF file to the Slack channel  
   - Confirm extraction, Google Sheets append, and Slack confirmation message  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                  | Context or Link                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates invoice data extraction from PDFs shared in Slack and uses GPT-4o for AI parsing before saving data to Google Sheets and notifying users on Slack. It requires Slack API permissions for reading files and posting messages, OpenAI API access, and Google Sheets OAuth2 authorization. | Workflow description and setup instructions embedded in sticky notes nodes within the workflow JSON                                   |
| To customize fields extracted by AI, adjust the prompt in the LangChain Agent node and modify the JSON schema in the Structured Output node accordingly.                                                                                                      | Customization hint for AI parsing and data structure                                                                                   |
| Ensure the Slack bot has scopes: `files:read`, `channels:read`, `chat:write`. The Google Sheets OAuth2 must have write access to the target spreadsheet.                                                                                                        | Permissions reminder                                                                                                                  |
| Link for Slack API scopes and OAuth: https://api.slack.com/authentication/oauth-v2                                                                                                                                                                            | Slack API documentation                                                                                                              |
| OpenAI GPT-4o documentation and usage: https://platform.openai.com/docs/models/gpt-4o                                                                                                                                                                           | OpenAI documentation                                                                                                                 |
| Google Sheets API documentation: https://developers.google.com/sheets/api                                                                                                                                                                                     | Google Sheets API reference                                                                                                          |

---

**Disclaimer:** The text provided is extracted exclusively from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.