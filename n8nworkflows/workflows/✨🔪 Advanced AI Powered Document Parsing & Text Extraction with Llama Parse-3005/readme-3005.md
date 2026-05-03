✨🔪 Advanced AI Powered Document Parsing & Text Extraction with Llama Parse

https://n8nworkflows.xyz/workflows/----advanced-ai-powered-document-parsing---text-extraction-with-llama-parse-3005


# ✨🔪 Advanced AI Powered Document Parsing & Text Extraction with Llama Parse

### 1. Workflow Overview

This workflow automates advanced document parsing and text extraction using LlamaParse, targeting use cases such as invoice processing, document classification, and multi-channel delivery of insights. It supports ingestion from Gmail attachments or direct webhook uploads, validates file formats, extracts structured data with AI, and distributes results via Google Sheets, Google Drive, and Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Document Ingestion & Validation**  
  Handles incoming documents from Gmail or webhook, checks for attachments, validates file extensions against supported types, and prepares files for processing.

- **1.2 Document Parsing with LlamaParse**  
  Uploads validated documents to LlamaParse API for advanced text extraction and receives parsed markdown responses.

- **1.3 Document Classification and Summarization**  
  Classifies documents (e.g., invoice vs. non-invoice), generates summaries, and extracts structured invoice data using AI language models.

- **1.4 Data Storage and Multi-Channel Delivery**  
  Saves original and parsed documents to Google Drive, updates Google Sheets with extracted data, and sends summaries and invoice details via Telegram.

- **1.5 Error Handling**  
  Sends error notifications via Telegram if AI extraction or processing fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion & Validation

**Overview:**  
This block monitors Gmail for new emails with attachments from a specific sender and also accepts documents via a webhook. It validates the file extensions of attachments against supported LlamaParse formats before proceeding.

**Nodes Involved:**  
- Gmail Trigger  
- Gmail  
- Limit  
- Get Message  
- Is there an Email Attachement (If node)  
- HTTP Request (to get supported file extensions)  
- Aggregate  
- Edit Fields  
- If Supported File Extensions (If node)  
- Merge  
- No Operation, do nothing (fallback)  
- No Operation, do nothing1 (fallback)  
- Webhook

**Node Details:**

- **Gmail Trigger**  
  - Type: Trigger node for Gmail  
  - Configured to poll every minute for new emails  
  - Credentials: Gmail OAuth2  
  - Output: Triggers workflow on new emails

- **Gmail**  
  - Type: Gmail node to fetch emails  
  - Filters: Emails with attachments from "joe@example.com"  
  - Limit: 28 emails per fetch  
  - Credentials: Gmail OAuth2  
  - Output: List of emails matching criteria

- **Limit**  
  - Type: Limit node  
  - Keeps only the last items (to process recent emails)

- **Get Message**  
  - Type: Gmail node  
  - Fetches full email message including attachments  
  - Downloads attachments with prefix "file"  
  - Credentials: Gmail OAuth2

- **Is there an Email Attachement**  
  - Type: If node  
  - Checks if the email contains binary attachment data  
  - Branches workflow based on presence of attachment

- **HTTP Request (Supported File Extensions)**  
  - Type: HTTP Request  
  - Calls LlamaParse API endpoint to retrieve supported file extensions  
  - No authentication needed  
  - Output: List of supported extensions

- **Aggregate**  
  - Type: Aggregate node  
  - Aggregates all supported file extensions into a single array

- **Edit Fields**  
  - Type: Set node  
  - Extracts the file extension from the email attachment (e.g., ".pdf")

- **If Supported File Extensions**  
  - Type: If node  
  - Checks if the attachment's file extension is in the supported list  
  - True branch proceeds to parsing  
  - False branch leads to no operation (skip)

- **Merge**  
  - Combines data streams for further processing

- **No Operation, do nothing / No Operation, do nothing1**  
  - Type: NoOp nodes  
  - Used as placeholders for unsupported files or empty branches

- **Webhook**  
  - Type: Webhook node  
  - Accepts POST requests with documents for parsing  
  - Path: /parse

**Edge Cases & Failures:**  
- Missing or unsupported file extensions cause the workflow to skip processing.  
- Gmail API rate limits or OAuth token expiration may cause failures.  
- Webhook must be publicly accessible and secured to avoid unauthorized uploads.

---

#### 2.2 Document Parsing with LlamaParse

**Overview:**  
Uploads validated documents to LlamaParse API for parsing and receives parsed markdown content. The parsed document is then saved for further analysis.

**Nodes Involved:**  
- Parse Document with LlamaParse (HTTP Request)  
- Get Parsed Markdown (Set)  
- Save Parsed Document to Google Drive

**Node Details:**

- **Parse Document with LlamaParse**  
  - Type: HTTP Request  
  - Method: POST to LlamaParse upload endpoint  
  - Content-Type: multipart/form-data  
  - Sends file binary data, webhook URL for callback, and parsing options (accurate_mode: true, premium_mode: false)  
  - Authentication: HTTP Header Auth with API key  
  - Output: JSON response with jobId and parsed data

- **Get Parsed Markdown**  
  - Type: Set node  
  - Extracts markdown content from LlamaParse response (`body.md`) into a field named `data`

- **Save Parsed Document to Google Drive**  
  - Type: Google Drive node  
  - Saves the parsed markdown as a text file named `{jobId}-parsed.md` in root folder  
  - Credentials: Google Drive OAuth2

**Edge Cases & Failures:**  
- HTTP request failures due to network or API key issues.  
- Large files may cause timeouts or require chunked uploads (not shown).  
- Google Drive quota limits or permission errors.

---

#### 2.3 Document Classification and Summarization

**Overview:**  
Classifies the parsed document as invoice or not, generates a detailed summary, and extracts structured invoice data as JSON using AI language models.

**Nodes Involved:**  
- Classify Parsed Document (LangChain Text Classifier)  
- Summarize Document (LangChain Chain LLM)  
- Extract Invoice Details as JSON (LangChain Chain LLM)  
- Invoice Details (Set)  
- Prepare Message (Set)  
- Summarize Email (LangChain Chain LLM)  
- gpt-4o-mini, gpt-4o-mini1, gpt-4o-mini2, gpt-4o-mini3 (OpenAI LLM nodes)

**Node Details:**

- **Classify Parsed Document**  
  - Type: LangChain text classifier  
  - Input: Parsed markdown text  
  - Categories: "invoice" or "not invoice"  
  - Output: Classification result

- **Summarize Document**  
  - Type: LangChain chain LLM  
  - Prompt: Detailed document analysis with sections for summary, key points, recommendations  
  - Input: Parsed markdown text  
  - Output: Text summary  
  - On error: continues with error output

- **Extract Invoice Details as JSON**  
  - Type: LangChain chain LLM  
  - Prompt: Converts markdown invoice content into a strict JSON schema with detailed invoice fields (dates, transactions, payment details, summary, policies)  
  - Input: Parsed markdown text  
  - Output: Structured JSON data  
  - On error: continues with error output

- **Invoice Details**  
  - Type: Set node  
  - Assigns extracted JSON text to a field named `output`

- **Prepare Message**  
  - Type: Set node  
  - Prepares structured fields from extracted JSON for downstream use (invoice details, payment details, summary)

- **Summarize Email**  
  - Type: LangChain chain LLM  
  - Summarizes email text content (used for email attachment context)

- **OpenAI LLM nodes (gpt-4o-mini, etc.)**  
  - Used as underlying AI engines for classification, summarization, and extraction  
  - Credentials: OpenAI API key

**Edge Cases & Failures:**  
- AI model errors or timeouts.  
- Incorrect or incomplete extraction due to ambiguous input.  
- JSON parsing errors if AI output is malformed.  
- On error, workflow continues but sends error notifications.

---

#### 2.4 Data Storage and Multi-Channel Delivery

**Overview:**  
Saves original and summarized documents to Google Drive, updates Google Sheets with extracted invoice data, and sends summaries and invoice details via Telegram for immediate review.

**Nodes Involved:**  
- Save Document to Google Drive  
- Save Summarized Document to Google Drive  
- Save LlamaParse ID and Summary to Google Sheets  
- Update Google Sheet by LlamaParse ID  
- Send Invoice Details as Telegram Message  
- Send Document Summary as Telegram Message

**Node Details:**

- **Save Document to Google Drive**  
  - Saves original email attachment file to Google Drive root folder  
  - Filename includes email ID and original attachment name  
  - Credentials: Google Drive OAuth2

- **Save Summarized Document to Google Drive**  
  - Saves AI-generated summary as a markdown text file named `{jobId}-summary.md`

- **Save LlamaParse ID and Summary to Google Sheets**  
  - Appends or updates a Google Sheet with job ID, summary, and document URL  
  - Credentials: Google Sheets OAuth2  
  - Sheet: "Expenses" in specified spreadsheet

- **Update Google Sheet by LlamaParse ID**  
  - Updates Google Sheet rows matching job ID with detailed invoice fields extracted from JSON  
  - Credentials: Google Sheets OAuth2

- **Send Invoice Details as Telegram Message**  
  - Sends invoice summary and details as a Telegram message to configured chat ID  
  - Credentials: Telegram API

- **Send Document Summary as Telegram Message**  
  - Sends document summary text via Telegram to configured chat ID

**Edge Cases & Failures:**  
- Google Drive or Sheets API quota exceeded or permission denied.  
- Telegram API failures or invalid chat ID.  
- Data mismatches causing update failures in Sheets.

---

#### 2.5 Error Handling

**Overview:**  
Sends error notifications via Telegram if invoice extraction or document summarization fails.

**Nodes Involved:**  
- Send Error Message 1 (Telegram)  
- Send Error Message 2 (Telegram)

**Node Details:**

- Both nodes send a simple "Error in workflow" message to the Telegram chat ID environment variable.  
- Triggered on AI node errors (e.g., extraction or summarization failures).

**Edge Cases & Failures:**  
- Telegram API errors or invalid credentials may prevent error notification delivery.

---

### 3. Summary Table

| Node Name                          | Node Type                      | Functional Role                                | Input Node(s)                        | Output Node(s)                              | Sticky Note                                                                                                  |
|-----------------------------------|--------------------------------|-----------------------------------------------|------------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                     | Gmail Trigger                  | Trigger on new Gmail emails                    | -                                  | Gmail                                       | ## Get Emails with Attachments<br>☀️Disclaimer: Only processes first attachment.                              |
| Gmail                            | Gmail                         | Fetch emails with attachments                  | Gmail Trigger                      | Limit                                       | ## Get Emails with Attachments<br>☀️Disclaimer: Only processes first attachment.                              |
| Limit                           | Limit                         | Limit number of emails processed               | Gmail                             | Get Message                                 |                                                                                                              |
| Get Message                     | Gmail                         | Fetch full email message and attachments       | Limit                             | Is there an Email Attachement                |                                                                                                              |
| Is there an Email Attachement    | If                            | Check if email has attachment                   | Get Message                      | HTTP Request (true), Merge (true) / NoOp (false) |                                                                                                              |
| HTTP Request                    | HTTP Request                  | Get supported file extensions from LlamaParse  | Is there an Email Attachement     | Aggregate                                   | ## Check for Supported File Extension<br>https://docs.cloud.llamaindex.ai/API/get-supported-file-extensions-api-v-1-parsing-supported-file-extensions-get |
| Aggregate                      | Aggregate                    | Aggregate supported extensions into array      | HTTP Request                     | Edit Fields                                 |                                                                                                              |
| Edit Fields                    | Set                          | Extract file extension from attachment          | Aggregate                       | If Supported File Extensions                 |                                                                                                              |
| If Supported File Extensions    | If                            | Check if attachment extension is supported      | Edit Fields                     | Merge (true), No Operation, do nothing1 (false) |                                                                                                              |
| Merge                          | Merge                        | Combine data streams for processing             | If Supported File Extensions, Is there an Email Attachement | Parse Document with LlamaParse, Save Document to Google Drive, Summarize Email |                                                                                                              |
| Parse Document with LlamaParse  | HTTP Request                  | Upload document to LlamaParse API                | Merge                           | Merge Email Processing                       | ## Send to LlamaParse<br>https://docs.cloud.llamaindex.ai/API/upload-file-api-v-1-parsing-upload-post          |
| Save Document to Google Drive    | Google Drive                  | Save original document to Google Drive           | Parse Document with LlamaParse   | Prepare Data                                | ## Save Document to Google Drive                                                                             |
| Prepare Data                   | Set                          | Prepare Google Drive file metadata               | Save Document to Google Drive     | Merge Email Processing                       |                                                                                                              |
| Merge Email Processing          | Merge                        | Combine data for saving and notifications        | Parse Document with LlamaParse, Prepare Data, Summarize Email | Save LlamaParse ID and Summary to Google Sheets |                                                                                                              |
| Save LlamaParse ID and Summary to Google Sheets | Google Sheets                | Save job ID and summary to Google Sheets         | Merge Email Processing           | -                                           | ## Save To Google Sheets                                                                                      |
| Get Parsed Markdown             | Set                          | Extract markdown content from LlamaParse response | Webhook                         | Classify Parsed Document, Save Parsed Document to Google Drive | ## Parsed Markdown from LlamaParse                                                                            |
| Save Parsed Document to Google Drive | Google Drive                  | Save parsed markdown document to Google Drive    | Get Parsed Markdown              | -                                           | ## Save Parsed Document to Google Drive                                                                       |
| Classify Parsed Document        | LangChain Text Classifier     | Classify document type (invoice or not)           | Get Parsed Markdown              | Summarize Document, Extract Invoice Details as JSON | ## Classify Parsed Document<br>Add More Classifications as Required                                           |
| Summarize Document              | LangChain Chain LLM           | Generate detailed document summary                 | Classify Parsed Document         | Send Document Summary as Telegram Message, Save Summarized Document to Google Drive, Send Error Message 1 | ## Summarize Document<br>☀️Update User & System Prompt for Your Specific Use Case                             |
| Extract Invoice Details as JSON | LangChain Chain LLM           | Extract invoice details as structured JSON         | Classify Parsed Document         | Invoice Details, Send Error Message 2       | ## Extract Invoice as JSON<br>☀️Update User & System Prompt for Your Specific Use Case                        |
| Invoice Details                | Set                          | Assign extracted invoice JSON to output field      | Extract Invoice Details as JSON  | Prepare Message, Update Google Sheet by LlamaParse ID |                                                                                                              |
| Prepare Message               | Set                          | Prepare structured invoice data for messaging      | Invoice Details                 | Send Invoice Details as Telegram Message    |                                                                                                              |
| Update Google Sheet by LlamaParse ID | Google Sheets                | Update Google Sheet with detailed invoice data     | Invoice Details                 | -                                           |                                                                                                              |
| Send Invoice Details as Telegram Message | Telegram                      | Send invoice summary and details via Telegram      | Prepare Message                | -                                           |                                                                                                              |
| Send Document Summary as Telegram Message | Telegram                      | Send document summary via Telegram                  | Summarize Document             | -                                           |                                                                                                              |
| Send Error Message 1           | Telegram                      | Send error notification on summarization failure   | Summarize Document (error)      | -                                           |                                                                                                              |
| Send Error Message 2           | Telegram                      | Send error notification on invoice extraction failure | Extract Invoice Details as JSON (error) | -                                           |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Poll every minute  
   - Credentials: Gmail OAuth2  
   - No filters (or customize as needed)

2. **Create Gmail Node**  
   - Operation: Get All  
   - Filter: `has:attachment` and sender `joe@example.com`  
   - Limit: 28  
   - Credentials: Gmail OAuth2  
   - Connect Gmail Trigger → Gmail

3. **Add Limit Node**  
   - Keep last items (default)  
   - Connect Gmail → Limit

4. **Add Get Message Node**  
   - Operation: Get full message  
   - Download attachments enabled, prefix "file"  
   - Credentials: Gmail OAuth2  
   - Connect Limit → Get Message

5. **Add If Node "Is there an Email Attachement"**  
   - Condition: Check if binary data exists in input item  
   - Connect Get Message → If

6. **Add HTTP Request Node to fetch supported file extensions**  
   - URL: `https://api.cloud.llamaindex.ai/api/parsing/supported_file_extensions`  
   - Method: GET (default)  
   - Connect If (true branch) → HTTP Request

7. **Add Aggregate Node**  
   - Aggregate all item data into field "extensions"  
   - Connect HTTP Request → Aggregate

8. **Add Set Node "Edit Fields"**  
   - Assign field "extension" = `".{{$json.binary.file0.fileExtension}}"`  
   - Connect Aggregate → Edit Fields

9. **Add If Node "If Supported File Extensions"**  
   - Condition: Check if `extensions` array includes `extension`  
   - Connect Edit Fields → If

10. **Add Merge Node**  
    - Mode: Combine by position  
    - Number of inputs: 3  
    - Connect If (true branch) → Merge (input 2)  
    - Connect If (false branch) → No Operation node (skip)  
    - Connect If (false branch) → No Operation node 1 (skip)

11. **Add HTTP Request Node "Parse Document with LlamaParse"**  
    - URL: `https://api.cloud.llamaindex.ai/api/parsing/upload`  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Body parameters:  
      - file: binary data from attachment  
      - webhook_url: your n8n webhook URL `/webhook/parse`  
      - accurate_mode: true  
      - premium_mode: false  
    - Authentication: HTTP Header Auth with LlamaParse API key  
    - Connect Merge → Parse Document with LlamaParse

12. **Add Google Drive Node "Save Document to Google Drive"**  
    - Operation: Upload file  
    - Filename: `{email_id}_{attachment_filename}`  
    - Folder: root or desired folder  
    - Credentials: Google Drive OAuth2  
    - Connect Parse Document with LlamaParse → Save Document to Google Drive

13. **Add Set Node "Prepare Data"**  
    - Assign fields:  
      - google_drive_fileid = `{file id}`  
      - webViewLink = `{file webViewLink}`  
    - Connect Save Document to Google Drive → Prepare Data

14. **Add Merge Node "Merge Email Processing"**  
    - Mode: Combine by position, 3 inputs  
    - Connect Parse Document with LlamaParse → Merge Email Processing (input 1)  
    - Connect Prepare Data → Merge Email Processing (input 2)  
    - Connect Summarize Email (see below) → Merge Email Processing (input 3)

15. **Add Google Sheets Node "Save LlamaParse ID and Summary to Google Sheets"**  
    - Operation: Append or update  
    - Columns: jobid, summary, image_url  
    - Sheet: "Expenses" or your sheet  
    - Credentials: Google Sheets OAuth2  
    - Connect Merge Email Processing → Google Sheets

16. **Add Webhook Node**  
    - Path: `/parse`  
    - Method: POST  
    - This receives LlamaParse callback with parsed data

17. **Add Set Node "Get Parsed Markdown"**  
    - Assign field `data` = `{{$json.body.md}}`  
    - Connect Webhook → Get Parsed Markdown

18. **Add Google Drive Node "Save Parsed Document to Google Drive"**  
    - Operation: Create file from text  
    - Filename: `{jobId}-parsed.md`  
    - Content: `{{$json.data}}`  
    - Folder: root or desired folder  
    - Credentials: Google Drive OAuth2  
    - Connect Get Parsed Markdown → Save Parsed Document to Google Drive

19. **Add LangChain Text Classifier Node "Classify Parsed Document"**  
    - Input text: `{{$json.data}}`  
    - Categories: "invoice", "not invoice"  
    - Connect Get Parsed Markdown → Classify Parsed Document

20. **Add LangChain Chain LLM Node "Summarize Document"**  
    - Prompt: Detailed document analysis and recommendations (customize prompt)  
    - Input: `{{$json.data}}`  
    - Connect Classify Parsed Document → Summarize Document

21. **Add LangChain Chain LLM Node "Extract Invoice Details as JSON"**  
    - Prompt: Convert markdown to strict JSON schema for invoice (customize prompt)  
    - Input: `{{$json.data}}`  
    - Connect Classify Parsed Document → Extract Invoice Details as JSON

22. **Add Set Node "Invoice Details"**  
    - Assign field `output` = `{{$json.text}}` (extracted JSON)  
    - Connect Extract Invoice Details as JSON → Invoice Details

23. **Add Set Node "Prepare Message"**  
    - Assign structured fields from `output` for invoice details, payment, summary  
    - Connect Invoice Details → Prepare Message

24. **Add Google Sheets Node "Update Google Sheet by LlamaParse ID"**  
    - Operation: Update row by jobid  
    - Columns: detailed invoice fields (subtotal, gst, payment, etc.)  
    - Credentials: Google Sheets OAuth2  
    - Connect Invoice Details → Update Google Sheet by LlamaParse ID

25. **Add Telegram Node "Send Invoice Details as Telegram Message"**  
    - Text: invoice summary and details  
    - Chat ID: from environment variable `TELEGRAM_CHAT_ID`  
    - Credentials: Telegram API  
    - Connect Prepare Message → Send Invoice Details as Telegram Message

26. **Add Telegram Node "Send Document Summary as Telegram Message"**  
    - Text: summary text  
    - Chat ID: from environment variable  
    - Credentials: Telegram API  
    - Connect Summarize Document → Send Document Summary as Telegram Message

27. **Add Telegram Nodes "Send Error Message 1" and "Send Error Message 2"**  
    - Text: "Error in workflow"  
    - Chat ID: environment variable  
    - Connect error outputs of Summarize Document and Extract Invoice Details as JSON respectively

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates document processing using LlamaParse to extract and analyze text from various file formats. It intelligently processes documents, extracts structured data, and delivers actionable insights through multiple channels.                                                                                                                                       | Workflow Description (Sticky Note)                                                                                            |
| Document ingestion supports Gmail monitoring and webhook uploads. Only the first attachment of each email is processed. Adjust search and limit settings as needed.                                                                                                                                                                                                                  | Sticky Note near Gmail Trigger and Gmail nodes                                                                                |
| Supported file extensions are dynamically fetched from LlamaParse API to ensure compatibility.                                                                                                                                                                                                                                                                                       | Sticky Note near HTTP Request node fetching supported extensions                                                              |
| LlamaParse API documentation for file upload: https://docs.cloud.llamaindex.ai/API/upload-file-api-v-1-parsing-upload-post                                                                                                                                                                                                                                                           | Sticky Note near Parse Document with LlamaParse node                                                                          |
| Customize AI prompts for document classification, summarization, and invoice extraction to fit specific use cases.                                                                                                                                                                                                                                                                   | Sticky Notes near LangChain nodes                                                                                             |
| Telegram notifications require environment variable `TELEGRAM_CHAT_ID` and Telegram bot credentials configured in n8n.                                                                                                                                                                                                                                                               | Telegram nodes configuration                                                                                                |
| Google Drive and Google Sheets integrations require OAuth2 credentials with appropriate permissions.                                                                                                                                                                                                                                                                                   | Google Drive and Sheets nodes                                                                                                |
| Test workflow with sample documents of various formats to verify extraction accuracy and notification delivery before production deployment.                                                                                                                                                                                                                                         | Setup instructions in workflow description                                                                                   |

---

This structured documentation provides a comprehensive understanding of the workflow’s architecture, node configurations, data flow, and integration points, enabling advanced users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.