Automatic Email Invoice Archiving & Data Extraction with Gmail, Drive & AI

https://n8nworkflows.xyz/workflows/automatic-email-invoice-archiving---data-extraction-with-gmail--drive---ai-10923


# Automatic Email Invoice Archiving & Data Extraction with Gmail, Drive & AI

### 1. Workflow Overview

This workflow automates the process of archiving invoice emails, extracting structured invoice data, and logging it for financial tracking. It is designed primarily for invoices received by email (e.g., from ISP or utility providers) and uses AI to parse PDF invoices. The workflow integrates Gmail, Google Drive, Google Sheets, and optionally FTP/SFTP, alongside AI parsing via LangChain’s AI Agent and OpenRouter language model.

**Use Cases:**  
- Automatically collect invoice emails from a specified sender  
- Download and store invoice PDFs securely in Google Drive (and optionally FTP)  
- Extract detailed structured invoice data (invoice number, vendor, dates, line items, taxes) using AI  
- Append extracted data to Google Sheets for monitoring, reporting, or further processing  
- Optionally clean up emails and temporary files after processing  

**Logical Blocks:**  
- 1.1 Trigger & Email Intake  
- 1.2 Attachment Filtering & Download  
- 1.3 File Upload & Optional FTP Transfer  
- 1.4 PDF Text Extraction  
- 1.5 AI-Based Invoice Parsing  
- 1.6 JSON Cleaning & Data Extraction  
- 1.7 Data Logging to Google Sheets  
- 1.8 Cleanup (Optional Email & File Deletion)  

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Email Intake

**Overview:**  
Schedules the workflow operation and fetches emails from a specific sender to process invoice attachments.

**Nodes Involved:**  
- Schedule Trigger  
- Get many messages  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Runs workflow automatically at defined intervals (hourly, daily, etc.)  
  - Configuration: Interval set to every hour at minute 25  
  - Input: None (trigger node)  
  - Output: Triggers the next node to fetch messages  
  - Edge Cases: Misconfigured intervals could cause missed runs or overload  

- **Get many messages**  
  - Type: Gmail node  
  - Role: Fetch all emails from a specified sender address  
  - Configuration: Filter by sender email configured in parameters; download attachments enabled  
  - Credentials: Gmail OAuth2 required  
  - Input: Trigger from Schedule Trigger  
  - Output: List of emails matching criteria  
  - Edge Cases: Gmail API limits, auth failures, empty inbox, or no matching messages  

---

#### 2.2 Attachment Filtering & Download

**Overview:**  
Filters emails to only those containing attachments and downloads the invoice PDFs.

**Nodes Involved:**  
- Filter-contains_attachment  
- Gmail-Get_Invoice  
- downloadFile  

**Node Details:**  

- **Filter-contains_attachment**  
  - Type: Filter node  
  - Role: Pass through only emails containing attachments (based on existence of binary data)  
  - Configuration: Checks that the binary data field exists in incoming email items  
  - Input: Output from Get many messages  
  - Output: Filtered emails with attachments  
  - Edge Cases: Emails without attachments filtered out; false negatives if attachments are malformed  

- **Gmail-Get_Invoice**  
  - Type: Gmail node  
  - Role: Retrieves the full email and attachments for a given message ID  
  - Configuration: Uses messageId from previous filtered node; downloads attachments  
  - Credentials: Gmail OAuth2  
  - Input: Filtered email items  
  - Output: Detailed email data including attachments  
  - Edge Cases: API limits, missing message ID, attachments not downloadable  

- **downloadFile**  
  - Type: HTTP Request node  
  - Role: Downloads the invoice PDF file from Google Drive webContentLink provided in email attachment metadata  
  - Configuration: URL dynamically set to file’s webContentLink  
  - Input: Output from GoogleDrive-upload-file (see next block)  
  - Output: Binary file content for further processing  
  - Edge Cases: Broken or expired links, network issues, file not found  

---

#### 2.3 File Upload & Optional FTP Transfer

**Overview:**  
Uploads the downloaded invoice PDF to a designated Google Drive folder and optionally uploads it to an FTP/SFTP server.

**Nodes Involved:**  
- GoogleDrive-upload-file  
- FTP-upload-octopus  
- Delete a file1  

**Node Details:**  

- **GoogleDrive-upload-file**  
  - Type: Google Drive node  
  - Role: Uploads PDF invoice to a specific Google Drive folder  
  - Configuration: File name dynamically set to sender name + date; folder ID specified; uses OAuth2 credentials  
  - Input: Invoice PDF binary from Gmail-Get_Invoice  
  - Output: Google Drive file metadata including file ID and name  
  - Edge Cases: Permission issues, quota exceeded, invalid folder ID  

- **FTP-upload-octopus** (Optional)  
  - Type: FTP node (SFTP protocol)  
  - Role: Uploads a copy of the invoice PDF to a remote FTP/SFTP server  
  - Configuration: Path dynamically built from uploaded file name, sanitized for invalid characters; credentials required  
  - Input: Output from Extract from File1 (triggered after upload and text extraction)  
  - Output: Confirmation of file upload  
  - Edge Cases: FTP server unavailable, permission denied, network failures  

- **Delete a file1**  
  - Type: Google Drive node  
  - Role: Deletes the uploaded Google Drive file after successful FTP upload  
  - Configuration: Uses fileId from GoogleDrive-upload-file node  
  - Input: Output of FTP-upload-octopus  
  - Output: Confirmation of file deletion  
  - Edge Cases: File may have been already deleted or inaccessible  

---

#### 2.4 PDF Text Extraction

**Overview:**  
Extracts text content from the uploaded PDF invoice to prepare for AI parsing.

**Nodes Involved:**  
- Extract from File1  

**Node Details:**  

- **Extract from File1**  
  - Type: Extract From File node  
  - Role: Converts PDF binary to text content  
  - Configuration: Operation set to PDF text extraction (non-OCR)  
  - Input: Binary PDF file from downloadFile node  
  - Output: Extracted plain text from PDF  
  - Edge Cases: Image-only PDFs will fail or produce empty text; might require OCR alternative  

---

#### 2.5 AI-Based Invoice Parsing

**Overview:**  
Uses an AI agent to parse the extracted text and produce structured invoice data as JSON.

**Nodes Involved:**  
- AI_Agent-fields  
- OpenRouter Chat Model1  

**Node Details:**  

- **AI_Agent-fields**  
  - Type: LangChain Agent node  
  - Role: Sends extracted text to AI for structured extraction of invoice fields  
  - Configuration:  
    - System message defines detailed extraction schema (invoice number, vendor, dates, line items, taxes)  
    - Input text includes invoice text from previous node  
    - Enforces JSON-only raw output without markdown or wrappers  
    - Logic to ensure all line items are extracted until totals match  
  - Input: Extracted text from Extract from File1  
  - Output: Raw AI JSON output with invoice data  
  - Edge Cases: API failures, incomplete extraction, malformed output, rate limits  
  - Version: LangChain Agent 2.2  

- **OpenRouter Chat Model1**  
  - Type: LangChain OpenRouter Language Model node  
  - Role: Provides AI language model backend for AI_Agent-fields node  
  - Configuration: Connects to OpenRouter API with valid credentials  
  - Input: AI_Agent-fields (ai_languageModel channel)  
  - Output: AI-generated text response with extracted data  
  - Edge Cases: API quota, network issues, invalid API key  

---

#### 2.6 JSON Cleaning & Data Extraction

**Overview:**  
Cleans and parses the raw AI JSON output to produce usable structured data.

**Nodes Involved:**  
- Code_extractFields  

**Node Details:**  

- **Code_extractFields**  
  - Type: Code node (JavaScript)  
  - Role: Cleans AI raw output, removes markdown or backticks, unescapes JSON, and safely parses it  
  - Configuration:  
    - Strips any code block formatting (```json or ```)  
    - Finds JSON block from first '{' to last '}'  
    - Unescapes common escape sequences  
    - Tries JSON.parse twice if needed  
    - Returns error info if parsing fails  
  - Input: Raw AI Agent output JSON string  
  - Output: Cleaned JSON object with parsed invoice data or error info  
  - Edge Cases: Malformed AI output, empty or incomplete data, JSON parse errors  

---

#### 2.7 Data Logging to Google Sheets

**Overview:**  
Appends extracted invoice information into a Google Sheets document for tracking.

**Nodes Involved:**  
- GoogleSheets_save  

**Node Details:**  

- **GoogleSheets_save**  
  - Type: Google Sheets node  
  - Role: Appends invoice fields (date, amount, vendor, description) as a new row  
  - Configuration:  
    - Maps invoice_date, total_amount, vendor_name, and first line item description to columns  
    - Uses service account credentials for authentication  
    - Document ID and Sheet Name specified  
  - Input: Parsed JSON invoice data from Code_extractFields  
  - Output: Confirmation of row append  
  - Edge Cases: Sheet access denied, incorrect sheet name, service account permissions  

---

#### 2.8 Cleanup (Optional Email & File Deletion)

**Overview:**  
Deletes processed emails and temporary files to maintain a clean mailbox and storage.

**Nodes Involved:**  
- Delete a message  
- Delete a file1  

**Node Details:**  

- **Delete a message**  
  - Type: Gmail node  
  - Role: Deletes the processed email message from Gmail inbox  
  - Configuration: Uses message ID from the initial Get many messages node  
  - Credentials: Gmail OAuth2  
  - Input: After FTP upload and file deletion  
  - Output: Confirmation of message deletion  
  - Edge Cases: Email already deleted, permission denied  

- **Delete a file1** (also part of this block as above)  
  - See previous details in 2.3  

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                        | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                                                                                                                                                       |
|-----------------------|--------------------------------|-------------------------------------|---------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                | Triggers workflow at set interval   | None                      | Get many messages             | ### SECTION 1 Trigger: Runs the workflow on a schedule you choose. Adjust interval as needed (hourly, daily, etc.).                                                                                                              |
| Get many messages     | Gmail                          | Retrieves emails from specific sender | Schedule Trigger           | Filter-contains_attachment    | ### SECTION 2 — Email Intake & Attachment filtering: Fetches messages from a specific sender and filters only emails containing PDF attachments. Setup Gmail OAuth2.                                                             |
| Filter-contains_attachment | Filter                     | Filters emails to those with attachments | Get many messages          | Gmail-Get_Invoice             | See Section 2 note above.                                                                                                                                                                                                        |
| Gmail-Get_Invoice     | Gmail                          | Downloads full email and attachments | Filter-contains_attachment | GoogleDrive-upload-file       |                                                                                                                                                                                                                                 |
| GoogleDrive-upload-file | Google Drive                  | Uploads invoice PDF to Drive folder  | Gmail-Get_Invoice          | downloadFile                 | ### SECTION 3 — Download & Storage: Uploads PDF to Google Drive folder. Optional FTP upload. Setup Drive OAuth2 and folder ID.                                                                                                   |
| downloadFile          | HTTP Request                   | Downloads invoice file from Drive    | GoogleDrive-upload-file    | Extract from File1            | See Section 3 note above.                                                                                                                                                                                                        |
| Extract from File1    | Extract From File              | Extracts text from PDF               | downloadFile               | AI_Agent-fields              | ### SECTION 4 — PDF Text Extraction: Converts the PDF into text before sending it to the AI model.                                                                                                                                  |
| AI_Agent-fields       | LangChain Agent                | Parses invoice text into structured JSON | Extract from File1         | Code_extractFields           | ### SECTION 5 — AI Parsing: AI extracts structured data from invoice text. Enter LLM provider API key.                                                                                                                           |
| OpenRouter Chat Model1 | LangChain LM OpenRouter       | Provides AI model backend            | AI_Agent-fields (ai_languageModel) | AI_Agent-fields (ai_languageModel) | See Section 5 note above.                                                                                                                                                                                                        |
| Code_extractFields    | Code                          | Cleans and parses AI JSON output    | AI_Agent-fields            | GoogleSheets_save            |                                                                                                                                                                                                                                 |
| GoogleSheets_save     | Google Sheets                 | Appends extracted invoice data to sheet | Code_extractFields         |                              | ### SECTION 6 - GoogleSheets Logging: Connect service account, set Document ID and Sheet Name, create expected columns.                                                                                                          |
| FTP-upload-octopus    | FTP (SFTP)                    | Uploads PDF to FTP/SFTP server (optional) | Extract from File1         | Delete a file1               | See Section 3 note above.                                                                                                                                                                                                        |
| Delete a file1        | Google Drive                  | Deletes uploaded file from Drive    | FTP-upload-octopus         | Delete a message             | See above, cleanup step.                                                                                                                                                                                                         |
| Delete a message      | Gmail                         | Deletes processed email from inbox  | Delete a file1             |                              | See above, cleanup step.                                                                                                                                                                                                         |
| Sticky Note           | Sticky Note                   | Documentation and instructions      | None                      | None                        | Contains overview and detailed setup documentation with links ([Full setup Guide](https://paoloronco.it/n8n-template-automated-invoice-archiving/))                                                                             |
| Sticky Note2          | Sticky Note                   | Documentation - Trigger section     | None                      | None                        | See above Section 1 note.                                                                                                                                                                                                        |
| Sticky Note3          | Sticky Note                   | Documentation - Email intake section | None                      | None                        | See above Section 2 note.                                                                                                                                                                                                        |
| Sticky Note4          | Sticky Note                   | Documentation - Download & storage  | None                      | None                        | See above Section 3 note.                                                                                                                                                                                                        |
| Sticky Note5          | Sticky Note                   | Documentation - PDF text extraction | None                      | None                        | See above Section 4 note.                                                                                                                                                                                                        |
| Sticky Note6          | Sticky Note                   | Documentation - AI parsing           | None                      | None                        | See above Section 5 note.                                                                                                                                                                                                        |
| Sticky Note7          | Sticky Note                   | Documentation - Sheets logging       | None                      | None                        | See above Section 6 note.                                                                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to hourly, trigger at minute 25 (or your preference).  

2. **Create Get many messages node:**  
   - Type: Gmail  
   - Configure operation to "getAll" with filter: sender email = your invoice sender address  
   - Enable download attachments  
   - Connect input from Schedule Trigger output  
   - Set up Gmail OAuth2 credentials  

3. **Create Filter node (Filter-contains_attachment):**  
   - Type: Filter  
   - Condition: Check that binary data exists (e.g., `{{$node["Get many messages"].item.binary}}` exists)  
   - Connect input from Get many messages output  

4. **Create Gmail-Get_Invoice node:**  
   - Type: Gmail  
   - Operation: Get message by ID  
   - Use `{{$node["Get many messages"].json.id}}` for messageId  
   - Enable download attachments  
   - Connect input from Filter output  
   - Use same Gmail OAuth2 credentials  

5. **Create GoogleDrive-upload-file node:**  
   - Type: Google Drive  
   - Operation: Upload file  
   - Set file name dynamically as `"{{$json.from.value[0].name}}-{{$json.date}}"`  
   - Specify Drive (My Drive) and Folder ID where to save invoices  
   - Input data field: first binary file from Gmail-Get_Invoice  
   - Connect from Gmail-Get_Invoice output  
   - Use Google Drive OAuth2 credentials  

6. **Create downloadFile node (HTTP Request):**  
   - Type: HTTP Request  
   - URL: `{{$json.webContentLink}}` from GoogleDrive-upload-file output  
   - Connect input from GoogleDrive-upload-file output  

7. **Create Extract from File node:**  
   - Type: Extract From File  
   - Operation: PDF (extract text)  
   - Connect input from downloadFile output  

8. **Create OpenRouter Chat Model node:**  
   - Type: LangChain LM OpenRouter  
   - Configure with OpenRouter API credentials  
   - No extra parameters needed  
   
9. **Create AI_Agent-fields node (LangChain Agent):**  
   - Use LangChain Agent node type  
   - System message: Provide detailed instructions for invoice field extraction (invoice number, vendor, dates, line items, taxes, totals)  
   - Prompt text: pass extracted text from Extract from File node (`{{$json.text}}`)  
   - Connect AI language model input to OpenRouter Chat Model node  
   - Connect main input from Extract from File output  

10. **Create Code_extractFields node:**  
    - Type: Code node (JavaScript)  
    - Paste provided JS code that cleans raw AI output and parses JSON safely  
    - Connect input from AI_Agent-fields output  

11. **Create GoogleSheets_save node:**  
    - Type: Google Sheets  
    - Operation: Append  
    - Map columns:  
      - Vendor = `{{$json.vendor_name}}`  
      - Date = `{{$json.invoice_date}}`  
      - Amount = `{{$json.total_amount}}`  
      - Description = `{{$json.line_items[0].description}}`  
    - Specify Document ID and Sheet Name  
    - Use Google Sheets Service Account credentials  
    - Connect input from Code_extractFields output  

12. **(Optional) Create FTP-upload-octopus node:**  
    - Type: FTP (SFTP)  
    - Operation: Upload  
    - Path: Construct path with sanitized file name from GoogleDrive-upload-file output  
    - Credentials: FTP/SFTP credentials  
    - Connect input from Extract from File output  

13. **Create Delete a file1 node:**  
    - Type: Google Drive  
    - Operation: Delete File  
    - File ID: from GoogleDrive-upload-file output (`{{$json.id}}`)  
    - Connect input from FTP-upload-octopus output (or GoogleDrive-upload-file if FTP not used)  

14. **Create Delete a message node:**  
    - Type: Gmail  
    - Operation: Delete message  
    - Message ID: from Get many messages (`{{$node["Get many messages"].json.id}}`)  
    - Connect input from Delete a file1 output  
    - Use Gmail OAuth2 credentials  

15. **Add Sticky Notes nodes:**  
    - Add descriptive notes at workflow start and above each block for clarity and user guidance  
    - Include links to relevant documentation and setup guides  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Full setup guide for automated invoice archiving using n8n, including detailed instructions and video walkthroughs.                                                                                                            | https://paoloronco.it/n8n-template-automated-invoice-archiving/ |
| Gmail Node documentation for OAuth2 setup and usage.                                                                                                                                                                            | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ |
| Google Drive Node documentation for file upload and folder management.                                                                                                                                                           | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/ |
| FTP Node documentation for FTP/SFTP upload configuration.                                                                                                                                                                       | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.ftp/ |
| Important: The AI parsing relies on text-extractable PDFs. Image-only PDFs require OCR step not included here.                                                                                                                  | Note for users                                                    |
| Ensure all API credentials (Gmail, Google Drive, Google Sheets, OpenRouter, FTP) have correct permissions and are securely stored.                                                                                             | Security best practice                                           |
| This workflow safely deletes processed emails and temporary files to avoid clutter but can be adjusted to retain data if desired.                                                                                              | Optional cleanup configuration                                  |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, a workflow automation tool. This process fully complies with current content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.