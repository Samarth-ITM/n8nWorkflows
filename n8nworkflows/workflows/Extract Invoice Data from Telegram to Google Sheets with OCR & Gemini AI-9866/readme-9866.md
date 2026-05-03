Extract Invoice Data from Telegram to Google Sheets with OCR & Gemini AI

https://n8nworkflows.xyz/workflows/extract-invoice-data-from-telegram-to-google-sheets-with-ocr---gemini-ai-9866


# Extract Invoice Data from Telegram to Google Sheets with OCR & Gemini AI

### 1. Workflow Overview

This workflow automates the extraction of invoice data received via Telegram and stores it into a Google Sheets invoice database, while also saving the original invoice file to Google Drive. It uses OCR (via OCR.space API) to extract text from uploaded invoice files, then leverages Google Gemini AI to parse and structure the invoice data. Finally, it sends a summarized response back to the user on Telegram.

The workflow is logically divided into these blocks:

- **1.1 Input Reception (Telegram Trigger and Download)**  
  Detects invoice document uploads from Telegram users and downloads the attached file.

- **1.2 OCR Processing**  
  Sends the downloaded file to OCR.space to extract raw text from the invoice image or PDF.

- **1.3 AI Data Extraction**  
  Uses Google Gemini AI to parse the OCR text into structured invoice data fields.

- **1.4 Database Update & File Storage**  
  Appends the extracted invoice data to a Google Sheets spreadsheet and uploads the original invoice file to a Google Drive folder.

- **1.5 User Notification**  
  Generates a concise summary message with invoice details and sends it back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (Telegram Trigger and Download)

- **Overview:**  
  Listens for new document messages in a Telegram bot, downloads the invoice file when detected.

- **Nodes Involved:**  
  - Telegram Trigger1  
  - Download File1  

- **Node Details:**

  - **Telegram Trigger1**  
    - *Type:* Telegram Trigger  
    - *Role:* Starts workflow on new incoming Telegram messages containing documents.  
    - *Configuration:* Listens to "message" updates, uses Telegram Bot API credentials.  
    - *Key expressions:* None (trigger node).  
    - *Input:* None (trigger).  
    - *Output:* Message JSON containing document metadata.  
    - *Edge cases:* User not sending a document, bot blocked by user, Telegram API rate limits.

  - **Download File1**  
    - *Type:* Telegram node (file download)  
    - *Role:* Downloads the file attached to the Telegram document message.  
    - *Configuration:* File ID taken dynamically from incoming message JSON (`$json.message.document.file_id`). Uses same Telegram credentials.  
    - *Key expressions:* `={{ $json.message.document.file_id }}`  
    - *Input:* Output of Telegram Trigger1.  
    - *Output:* Binary file data for OCR processing.  
    - *Edge cases:* File missing or expired on Telegram server, network failures.

#### 2.2 OCR Processing

- **Overview:**  
  Submits the downloaded invoice file to OCR.space API to extract raw text.

- **Nodes Involved:**  
  - Analyze Image1  

- **Node Details:**

  - **Analyze Image1**  
    - *Type:* HTTP Request  
    - *Role:* Sends multipart-form data POST request to OCR.space API with the invoice file as binary.  
    - *Configuration:*  
      - URL: `https://api.ocr.space/parse/image`  
      - Method: POST  
      - Header: `apikey` (user must replace `"your_key_here"` with their OCR.space key)  
      - Body: multipart form with file binary from previous node.  
    - *Key expressions:* File binary from Download File1 as `data`.  
    - *Input:* Output of Download File1 (binary).  
    - *Output:* JSON response with OCR text under `ParsedResults[0].ParsedText`.  
    - *Edge cases:* Invalid/missing API key, file too large, OCR service limits/rate limits, poor image quality.

#### 2.3 AI Data Extraction

- **Overview:**  
  Uses Google Gemini AI to parse OCR text into structured invoice fields like invoice number, date, amount, etc.

- **Nodes Involved:**  
  - AI Agent  
  - Structured Output Parser  
  - Google Gemini Chat Model2  

- **Node Details:**

  - **Google Gemini Chat Model2**  
    - *Type:* Google Gemini AI Chat Model  
    - *Role:* Provides language model processing for AI Agent node.  
    - *Configuration:* Uses credentials for Google Palm API (Gemini).  
    - *Input:* Raw OCR text from Analyze Image1.  
    - *Output:* Model-generated AI response (text).  
    - *Edge cases:* API key invalid/expired, rate limits, network errors.

  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Applies a prompt instructing the AI to extract invoice fields in a specified form from OCR text.  
    - *Configuration:*  
      - Input: OCR text from Analyze Image1  
      - Prompt: requests structured fields invoiceNumber, invoiceDate, totalAmount, billingAddress, dueDate, notes.  
      - Special handling: If no due date, use invoiceDate; if no notes, set "no notes".  
      - Output parser enabled.  
    - *Input:* OCR text JSON.  
    - *Output:* AI-extracted structured fields.  
    - *Edge cases:* AI misinterpretation, incomplete OCR data, prompt failure.

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Enforces JSON schema on AI Agent output ensuring fields: invoiceNumber, invoiceDate, totalAmount, billingAddress, dueDate, notes.  
    - *Configuration:* JSON schema provided as example for expected keys.  
    - *Input:* AI Agent output.  
    - *Output:* Parsed JSON with structured invoice data.  
    - *Edge cases:* AI output not matching schema, parse errors.

#### 2.4 Database Update & File Storage

- **Overview:**  
  Appends extracted invoice data to Google Sheets and uploads the original invoice file to Google Drive with a date-based file name.

- **Nodes Involved:**  
  - Update Database1  
  - Telegram1 (to get file ID for Drive upload)  
  - Add Invoice Image to Drive1  
  - Set1  

- **Node Details:**

  - **Update Database1**  
    - *Type:* Google Sheets  
    - *Role:* Adds a new row to the invoice spreadsheet with extracted fields.  
    - *Configuration:*  
      - Spreadsheet ID and sheet tab (gid=0) set to user’s invoice database.  
      - Columns mapped: Invoice Number, Date, Total Amount ($), Billing Address, Due Date, Notes.  
      - Append mode enabled.  
    - *Key expressions:* Each column populated dynamically from AI Agent output JSON (`$json.output`).  
    - *Input:* AI Agent output with structured fields.  
    - *Output:* Confirmation of append operation.  
    - *Edge cases:* Sheet ID or permissions incorrect, header mismatch, API quota exceeded.

  - **Telegram1**  
    - *Type:* Telegram node (file info)  
    - *Role:* Retrieves file metadata from Telegram using the file ID from Download File1 (to pass to Drive upload).  
    - *Configuration:* File ID extracted dynamically from `Download File1` node output.  
    - *Input:* Output of Update Database1.  
    - *Output:* File metadata JSON.  
    - *Edge cases:* Telegram file info unavailable, API errors.

  - **Add Invoice Image to Drive1**  
    - *Type:* Google Drive  
    - *Role:* Uploads the original invoice file to a specified Google Drive folder.  
    - *Configuration:*  
      - File name: `"Invoice [MMMM-dd-yyyy]"` (current date formatted).  
      - Folder ID set to user’s Invoice folder.  
      - Uses Google Drive OAuth credentials with write access.  
    - *Input:* Binary file from Telegram1 output.  
    - *Output:* Google Drive file metadata including webContentLink.  
    - *Edge cases:* Folder permissions missing, Drive API limits, file name conflicts.

  - **Set1**  
    - *Type:* Set node  
    - *Role:* Creates consolidated data object with invoice info from OCR and file name from Drive upload for next AI step.  
    - *Configuration:*  
      - Assigns `Invoice Information` from OCR ParsedText.  
      - Assigns `File` from Drive file name.  
    - *Input:* Output of Add Invoice Image to Drive1.  
    - *Output:* JSON with invoice info and file name.  
    - *Edge cases:* Missing OCR text or file name.

#### 2.5 User Notification

- **Overview:**  
  Generates a user-friendly reply message summarizing invoice details and sends it back on Telegram.

- **Nodes Involved:**  
  - Google Gemini Chat Model1  
  - Invoice Agent1  
  - Reply1  

- **Node Details:**

  - **Google Gemini Chat Model1**  
    - *Type:* Google Gemini AI Chat Model  
    - *Role:* Supports the Invoice Agent1 node with AI language model capabilities.  
    - *Configuration:* Uses same Google Palm API credentials.  
    - *Input:* Text including invoice info, file name, and invoice database link.  
    - *Output:* AI-generated user-friendly message.  
    - *Edge cases:* API key issues, response delays.

  - **Invoice Agent1**  
    - *Type:* LangChain Agent  
    - *Role:* Formats a polite, concise message thanking the user and summarizing key invoice info (total, due date, notes), includes file name and link to spreadsheet.  
    - *Configuration:*  
      - System message defines assistant role and output style.  
      - Input variables: Invoice Information, File Name, Link to Invoice Database.  
    - *Input:* From Set1 with invoice info and Drive link.  
    - *Output:* Text message to user.  
    - *Edge cases:* AI output errors, missing data.

  - **Reply1**  
    - *Type:* Telegram node (send message)  
    - *Role:* Sends the generated summary message back to the Telegram user who submitted the invoice.  
    - *Configuration:*  
      - Message text from Invoice Agent1 output.  
      - Chat ID dynamically from Telegram Trigger1 user ID.  
      - Attribution disabled.  
    - *Input:* Output of Invoice Agent1.  
    - *Output:* Confirmation of message sent.  
    - *Edge cases:* User blocked bot, chat ID missing, Telegram API errors.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                          | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                                  |
|-------------------------|-------------------------------------|----------------------------------------|-----------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger1        | Telegram Trigger                    | Trigger on new invoice document upload | -                     | Download File1          | Before you start: set credentials, replace IDs, set API keys, ensure sheet columns are correct                                 |
| Download File1           | Telegram (file download)            | Downloads invoice file from Telegram   | Telegram Trigger1      | Analyze Image1           |                                                                                                                              |
| Analyze Image1           | HTTP Request (OCR.space)            | Sends invoice file to OCR.space API    | Download File1         | AI Agent                 | Set OCR apikey header; watch for rate limits and file quality                                                                |
| AI Agent                 | LangChain Agent                    | Extracts structured invoice fields from OCR text | Analyze Image1         | Update Database1          |                                                                                                                              |
| Structured Output Parser | LangChain Output Parser (JSON)     | Validates & parses AI output as JSON   | AI Agent               | AI Agent (parser output) |                                                                                                                              |
| Update Database1         | Google Sheets                      | Appends invoice data to Google Sheet   | AI Agent               | Telegram1                | Set correct Sheet ID, tab, and column mapping; verify permissions                                                             |
| Telegram1                | Telegram (file metadata)            | Retrieves Telegram file metadata        | Update Database1       | Add Invoice Image to Drive1 |                                                                                                                              |
| Add Invoice Image to Drive1 | Google Drive                      | Uploads original invoice file to Drive | Telegram1              | Set1                     | Set Drive folder ID; file naming format; verify Drive write permissions                                                      |
| Set1                    | Set                               | Prepares combined invoice info & file name | Add Invoice Image to Drive1 | Invoice Agent1            |                                                                                                                              |
| Invoice Agent1           | LangChain Agent                    | Generates friendly invoice summary message | Set1                   | Reply1                   | Optional: customize system prompt for message style                                                                           |
| Reply1                  | Telegram (send message)             | Sends summary message back to user     | Invoice Agent1         | -                        |                                                                                                                              |
| Google Gemini Chat Model2 | Google Gemini AI Chat Model        | AI model for invoice field extraction  | Analyze Image1         | AI Agent                 |                                                                                                                              |
| Google Gemini Chat Model1 | Google Gemini AI Chat Model        | AI model for invoice message generation | Invoice Agent1         | Invoice Agent1           |                                                                                                                              |
| Sticky Note              | Sticky Note                       | Instructions & setup notes              | -                     | -                        | Covers credentials, API keys, IDs, and column headers                                                                         |
| Sticky Note1             | Sticky Note                       | What to edit instructions               | -                     | -                        | Covers OCR key, Sheet ID, Drive folder ID, optional prompt and naming tweaks                                                  |
| Sticky Note2             | Sticky Note                       | How to run & expected inputs/outputs    | -                     | -                        | Explains operation flow, inputs, outputs, and success criteria                                                                |
| Sticky Note3             | Sticky Note                       | Behavior, logic, troubleshooting tips   | -                     | -                        | Details each block’s function, troubleshooting common errors, OCR quality issues, API limits                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Listen for "message" updates.  
   - Use Telegram API credentials with your bot token.  
   - Position: Start of workflow.

2. **Add Telegram Download File Node**  
   - Type: Telegram (File Download)  
   - Set `fileId` parameter to: `={{ $json.message.document.file_id }}`  
   - Use same Telegram credentials.  
   - Connect output of Telegram Trigger to this node.

3. **Add HTTP Request Node for OCR**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.ocr.space/parse/image`  
   - Content-Type: multipart-form-data  
   - Add header parameter: `apikey` with your OCR.space API key.  
   - Add body parameter: name `file`, type formBinaryData, input data from previous node binary.  
   - Connect output of Download File node here.

4. **Add Google Gemini Chat Model Node for Invoice Extraction**  
   - Type: @n8n/n8n-nodes-langchain.lmChatGoogleGemini  
   - Use Google Palm API credentials (Gemini).  
   - Connect OCR HTTP Request output here.

5. **Add LangChain Agent Node ("AI Agent")**  
   - Type: LangChain Agent  
   - Input text: OCR Parsed Text (`={{ $json.ParsedResults[0].ParsedText }}`) with prompt to extract invoice fields: invoiceNumber, invoiceDate, totalAmount, billingAddress, dueDate, notes.  
   - Enable output parser.  
   - Connect Google Gemini Chat Model output here.

6. **Add Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Provide JSON schema example with keys for invoiceNumber, invoiceDate, totalAmount, billingAddress, dueDate, notes.  
   - Connect AI Agent output to this node’s input parser.

7. **Add Google Sheets Node ("Update Database1")**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Spreadsheet ID: your invoice database sheet ID.  
   - Sheet Name / gid: your target tab (e.g., "gid=0").  
   - Map columns: Invoice Number, Date, Total Amount ($), Billing Address, Due Date, Notes from AI Agent output JSON.  
   - Use Google Sheets OAuth2 credentials.  
   - Connect Structured Output Parser output here.

8. **Add Telegram File Info Node ("Telegram1")**  
   - Type: Telegram (file info)  
   - fileId: `={{ $('Download File1').item.json.result.file_id }}`  
   - Use Telegram credentials.  
   - Connect Google Sheets node output here.

9. **Add Google Drive Node ("Add Invoice Image to Drive1")**  
   - Type: Google Drive  
   - Operation: Upload file  
   - File name: Use expression `"Invoice [{{$now.format('MMMM-dd-yyyy')}}]"`  
   - Folder ID: your Google Drive folder ID for invoices.  
   - Use Google Drive OAuth credentials.  
   - Connect Telegram file info node output here.

10. **Add Set Node ("Set1")**  
    - Type: Set  
    - Assign two fields:  
      - Invoice Information: OCR Parsed Text from Analyze Image node  
      - File: file name from Drive upload node output  
    - Connect Google Drive node output here.

11. **Add Google Gemini Chat Model Node ("Google Gemini Chat Model1")**  
    - Type: Google Gemini AI Chat Model  
    - Use Google Palm API credentials.  
    - Connect Set node output here.

12. **Add LangChain Agent Node ("Invoice Agent1")**  
    - Type: LangChain Agent  
    - System prompt: thank user, summarize total amount, due date, notes, mention Drive file name and provide sheet link.  
    - Input text: invoice info, file name, and invoice sheet link from Set node.  
    - Connect Google Gemini Chat Model1 output here.

13. **Add Telegram Send Message Node ("Reply1")**  
    - Type: Telegram (send message)  
    - Text: AI-generated message from Invoice Agent1 output.  
    - Chat ID: `={{ $('Telegram Trigger1').item.json.message.from.id }}`  
    - Use Telegram credentials.  
    - Connect Invoice Agent1 output here.

14. **Activate Workflow**  
    - Enable the workflow to listen for Telegram document uploads and process accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                                                                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Before you start, create/select credentials for Telegram Bot, Google Sheets, Google Drive, and Gemini API. Replace API keys and IDs.        | See Sticky Note at workflow start.                                                                                                                                                                                             |
| Set OCR.space API key in HTTP Request node header `apikey` to enable OCR functionality.                                                     | Sticky Note about editing HTTP Request node.                                                                                                                                                                                  |
| Google Sheets must have columns exactly as: Invoice Number, Date, Total Amount ($), Billing Address, Due Date, Notes for proper mapping.    | Sticky Note instructions.                                                                                                                                                                                                     |
| Drive folder ID must correspond to your Invoice storage folder, and OAuth credentials must allow file write.                               | Sticky Note instructions.                                                                                                                                                                                                     |
| To run: send a document (PDF/JPG/PNG) as a file (not compressed photo) to your Telegram bot; watch executions for process flow.            | Sticky Note with run instructions.                                                                                                                                                                                            |
| Expected outputs: new Google Sheet row with invoice data; file saved in Drive; Telegram reply with invoice summary and database link.       | Sticky Note with run instructions.                                                                                                                                                                                            |
| Troubleshooting tips: check OCR quality, bot permissions, API keys validity, and rate limits on OCR.space and Google APIs.                  | Sticky Note on behavior and troubleshooting.                                                                                                                                                                                  |
| Gemini AI prompt is customizable to adjust the style and detail level of the invoice summary message sent to users.                        | Editable in Invoice Agent1 node's system prompt.                                                                                                                                                                              |
| Useful external services: OCR.space (https://ocr.space/ocrapi), Google Gemini (PaLM API) for AI, Telegram Bot API, Google Sheets & Drive APIs. | Links and service references for credential setup and API keys.                                                                                                                                                               |

---

*Disclaimer: The provided text is extracted solely from an automated n8n workflow and fully complies with content policies. All handled data is legal and public.*