Process Invoices & Engineering Emails with Gmail, AI Analysis & Telegram Alerts

https://n8nworkflows.xyz/workflows/process-invoices---engineering-emails-with-gmail--ai-analysis---telegram-alerts-6709


# Process Invoices & Engineering Emails with Gmail, AI Analysis & Telegram Alerts

### 1. Workflow Overview

This workflow automates the processing of incoming emails related to invoices and engineering correspondences for Zenith Engineering, a conveyor manufacturing company. The workflow targets use cases involving:

- Monitoring new Gmail messages continuously.
- Downloading and analyzing attachments, especially invoices and engineering documents.
- Using AI to categorize emails, draft professional replies, and extract structured invoice data.
- Sending alerts and documents to the engineering team via Telegram.
- Replying automatically to sender threads based on AI-generated responses.

The workflow’s logic is organized into the following blocks:

- **1.1 Email Reception & Initial Filtering:** Trigger on new Gmail messages and filter for those with attachments.
- **1.2 Email Content Retrieval & Attachment Detection:** Get full email content and identify invoice-related attachments.
- **1.3 AI-based Email Analysis & Auto-Reply:** Use an AI agent to categorize emails and generate replies.
- **1.4 Attachment Processing & Invoice Data Extraction:** Split, filter, and extract detailed information from PDF invoices.
- **1.5 Telegram Notifications & Document Forwarding:** Send extracted invoice details and documents to the engineering department via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Reception & Initial Filtering

**Overview:**  
This block continuously monitors Gmail for new incoming emails and filters those that have attachments, ensuring only relevant emails proceed.

**Nodes Involved:**  
- Gmail Trigger  
- Filter1

**Node Details:**

- **Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Poll Gmail account every minute for new emails; downloads attachments automatically.  
  - Config: Poll every minute, downloads attachments with prefix "attachment_".  
  - Connections: Outputs to Filter1.  
  - Failure modes: OAuth authentication errors, Gmail API rate limits, network timeouts.

- **Filter1**  
  - Type: Filter  
  - Role: Passes only emails that have any attachments (checks if `$binary` data exists).  
  - Config: Condition checks existence of binary data in the incoming Gmail Trigger node.  
  - Input: Gmail Trigger  
  - Output: Get a message node  
  - Edge cases: Emails without attachments are filtered out; if binary data structure changes, filter may fail.

---

#### 1.2 Email Content Retrieval & Attachment Detection

**Overview:**  
Retrieves full email details, collects attachment metadata, and detects if any attachments are invoices.

**Nodes Involved:**  
- Get a message  
- Edit Fields1  
- Edit Fields2  
- Split Out  
- Split Out1  
- Merge  
- Filter  
- Merge1  
- Debugged code to find attachment

**Node Details:**

- **Get a message**  
  - Type: Gmail  
  - Role: Fetches full email message by ID including attachments.  
  - Config: Uses message ID from Gmail Trigger, downloads attachments.  
  - Inputs: Filter1  
  - Outputs: Edit Fields1, Edit Fields2, Merge1, Debugged code to find attachment  
  - Failures: API limits, invalid message ID, OAuth errors.

- **Edit Fields1**  
  - Type: Set  
  - Role: Creates a JSON field `attachments` listing attachment keys (binary keys). Also sets `message_id`.  
  - Config: Uses expression to get binary keys from "Get a message".  
  - Inputs: Get a message  
  - Outputs: Split Out

- **Edit Fields2**  
  - Type: Set  
  - Role: Creates `attachments_metadata` containing binary values (actual files) and sets `message_id`.  
  - Inputs: Get a message  
  - Outputs: Split Out1

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of attachment keys into individual items for processing, preserving fields.  
  - Inputs: Edit Fields1  
  - Outputs: Merge (position 0)

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits array of attachment file metadata into individual items.  
  - Inputs: Edit Fields2  
  - Outputs: Merge (position 1)

- **Merge**  
  - Type: Merge  
  - Role: Combines the split attachment keys and metadata back together by position.  
  - Inputs: Split Out (position 0), Split Out1 (position 1)  
  - Outputs: Filter

- **Filter**  
  - Type: Filter  
  - Role: Filters attachments to retain only PDFs (checking `fileType` equals "pdf").  
  - Inputs: Merge  
  - Outputs: Merge1  
  - Edge cases: Non-PDFs filtered out; if fileType metadata missing or malformed, may exclude valid files.

- **Merge1**  
  - Type: Merge  
  - Role: Joins email metadata with filtered attachments by matching `message_id`.  
  - Inputs: Get a message (position 1), Filter (position 0)  
  - Outputs: Extract from File, Convert Names to download files

- **Debugged code to find attachment**  
  - Type: Code (JavaScript)  
  - Role: Scans all binary attachments for filenames containing "invoice" (case-insensitive), flags if invoice attachment exists, and lists invoice filenames.  
  - Inputs: Get a message  
  - Outputs: AI Agent  
  - Edge cases: Assumes filenames use "invoice"; other naming conventions may miss invoices.

---

#### 1.3 AI-based Email Analysis & Auto-Reply

**Overview:**  
An AI LangChain agent categorizes emails based on content and attachment presence, generates professional responses, and replies in the original Gmail thread.

**Nodes Involved:**  
- OpenRouter Chat Model1  
- Structured Output Parser1  
- AI Agent  
- Reply to same thread

**Node Details:**

- **OpenRouter Chat Model1**  
  - Type: AI Language Model (OpenRouter)  
  - Role: Runs the "google/gemini-2.5-flash-lite-preview-06-17" model for email analysis.  
  - Credentials: OpenRouter API  
  - Input: Text prompt from "Get a message" and attachment flags.  
  - Outputs: AI Agent

- **Structured Output Parser1**  
  - Type: Structured Output Parser for AI  
  - Role: Parses AI model output into a strict JSON with "Label" and "Response".  
  - Input: AI model output  
  - Outputs: AI Agent

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Combines email subject, body, attachment presence, and system instructions to categorize and draft replies.  
  - Config: System message defines rules, labels ("Important", "Fishy", etc.), and response format.  
  - Inputs: Debugged code to find attachment (for attachment info), OpenRouter Chat Model1, Structured Output Parser1  
  - Outputs: Reply to same thread

- **Reply to same thread**  
  - Type: Gmail (reply)  
  - Role: Sends AI-generated reply to the original email thread, maintaining conversation context.  
  - Inputs: AI Agent output (response)  
  - Credentials: Gmail OAuth2  
  - Edge cases: Gmail API limits, thread ID invalid, message ID missing.

---

#### 1.4 Attachment Processing & Invoice Data Extraction

**Overview:**  
Processes PDF attachments, extracts invoice information using AI, and prepares the data for forwarding.

**Nodes Involved:**  
- Extract from File  
- Gemini  
- Information Extractor  
- Extracting File Details  
- Convert Names to download files  
- Remove Duplicates

**Node Details:**

- **Extract from File**  
  - Type: Extract from File (PDF)  
  - Role: Extracts text content from PDF binary attachment for analysis.  
  - Inputs: Merge1  
  - Outputs: Information Extractor  
  - Edge cases: Corrupted PDFs, unsupported PDFs, extraction errors.

- **Gemini**  
  - Type: AI Language Model (OpenRouter)  
  - Role: Uses the same Gemini model to assist in invoice data extraction.  
  - Credentials: OpenRouter API  
  - Inputs: Information Extractor (as AI language model input)  
  - Outputs: Information Extractor

- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Extracts structured invoice data (invoice number, dates, vendor, total, currency, items, tax) from text.  
  - Config: Manual JSON schema defining expected fields.  
  - Inputs: Extract from File, Gemini  
  - Outputs: Extracting File Details

- **Extracting File Details**  
  - Type: Set  
  - Role: Extracts and formats key invoice info from AI output into separate JSON fields.  
  - Inputs: Information Extractor  
  - Outputs: Send important details (Telegram notification)

- **Convert Names to download files**  
  - Type: Code (JavaScript)  
  - Role: Converts binary attachment data into a list of downloadable files with filenames for Telegram sending.  
  - Inputs: Merge1  
  - Outputs: Remove Duplicates

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Ensures no duplicate files are sent to Telegram.  
  - Inputs: Convert Names to download files  
  - Outputs: Send a document (e.g., dwg files)

---

#### 1.5 Telegram Notifications & Document Forwarding

**Overview:**  
Sends invoice details and accompanying documents to the Engineering department via Telegram.

**Nodes Involved:**  
- Send important details  
- Send a document (e.g., dwg files)  
- Sticky Note5  

**Node Details:**

- **Send important details**  
  - Type: Telegram  
  - Role: Sends a formatted text message with extracted invoice details (number, date, vendor) to Telegram chat.  
  - Inputs: Extracting File Details  
  - Credentials: Telegram API  
  - Edge cases: Telegram API limits, message formatting errors.

- **Send a document (e.g., dwg files)**  
  - Type: Telegram  
  - Role: Sends binary documents (e.g., DWG or other engineering files) to Telegram chat.  
  - Inputs: Remove Duplicates  
  - Credentials: Telegram API  
  - Edge cases: File size limits, unsupported file types.

- **Sticky Note5**  
  - Type: Sticky Note  
  - Content: "Telegram - Invoice and details to Engineering dept."  
  - Purpose: Visual annotation emphasizing Telegram notification block.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                              | Input Node(s)                | Output Node(s)                      | Sticky Note                                         |
|-------------------------------|-----------------------------------|----------------------------------------------|-----------------------------|------------------------------------|-----------------------------------------------------|
| Gmail Trigger                 | Gmail Trigger                    | Poll Gmail for new emails with attachments   |                             | Filter1                            |                                                     |
| Filter1                      | Filter                          | Filters emails that have attachments         | Gmail Trigger               | Get a message                     |                                                     |
| Get a message                | Gmail                           | Retrieves full email content and attachments | Filter1                    | Edit Fields1, Edit Fields2, Merge1, Debugged code to find attachment |                                                     |
| Edit Fields1                 | Set                             | Extracts attachment keys and message ID      | Get a message              | Split Out                        |                                                     |
| Edit Fields2                 | Set                             | Extracts attachment metadata and message ID | Get a message              | Split Out1                       |                                                     |
| Split Out                   | Split Out                       | Splits attachment keys array                  | Edit Fields1               | Merge (position 0)               |                                                     |
| Split Out1                  | Split Out                       | Splits attachment metadata array              | Edit Fields2               | Merge (position 1)               |                                                     |
| Merge                       | Merge                          | Combines keys and metadata by position        | Split Out, Split Out1       | Filter                         |                                                     |
| Filter                      | Filter                         | Filters only PDF attachments                   | Merge                      | Merge1                        |                                                     |
| Merge1                      | Merge                          | Combines email metadata with filtered attachments | Get a message, Filter      | Extract from File, Convert Names to download files |                                                     |
| Debugged code to find attachment | Code (JavaScript)              | Detects presence of invoice attachments       | Get a message              | AI Agent                      |                                                     |
| OpenRouter Chat Model1       | AI Language Model (OpenRouter) | Runs AI model for email categorization        |                            | AI Agent                      |                                                     |
| Structured Output Parser1    | AI Output Parser               | Parses AI model output into structured JSON   | OpenRouter Chat Model1      | AI Agent                      |                                                     |
| AI Agent                    | LangChain Agent                 | Categorizes email and drafts reply             | Debugged code to find attachment, OpenRouter Chat Model1, Structured Output Parser1 | Reply to same thread          |                                                     |
| Reply to same thread         | Gmail                          | Sends AI-generated reply in original thread   | AI Agent                   |                                |                                                     |
| Extract from File            | Extract from File (PDF)          | Extracts text from PDF attachments             | Merge1                     | Information Extractor           |                                                     |
| Gemini                      | AI Language Model (OpenRouter) | Assists in invoice data extraction             | Information Extractor       | Information Extractor           |                                                     |
| Information Extractor        | LangChain Information Extractor | Extracts structured invoice data               | Extract from File, Gemini   | Extracting File Details         |                                                     |
| Extracting File Details      | Set                            | Formats extracted invoice data                  | Information Extractor       | Send important details          |                                                     |
| Convert Names to download files | Code (JavaScript)              | Prepares binary files for Telegram sending     | Merge1                     | Remove Duplicates              |                                                     |
| Remove Duplicates            | Remove Duplicates               | Removes duplicate files                          | Convert Names to download files | Send a document (e.g., dwg files) |                                                     |
| Send important details       | Telegram                       | Sends invoice details text to Engineering chat | Extracting File Details     |                                | Sticky Note5: "Telegram - Invoice and details to Engineering dept." |
| Send a document (e.g., dwg files) | Telegram                       | Sends binary documents to Telegram chat         | Remove Duplicates           |                                | Sticky Note5: "Telegram - Invoice and details to Engineering dept." |
| Sticky Note                  | Sticky Note                    | Visual annotation for multiple documents extractor |                             |                                | "## MULTIPLE DOCUMENTS EXTRACTOR"                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Poll every minute for new emails  
   - Enable attachment download with prefix "attachment_"  
   - Connect to Filter1  
   - Credential: Gmail OAuth2 (set up with your Gmail account)

2. **Add Filter1 node**  
   - Type: Filter  
   - Condition: `$binary` exists (checks for attachments)  
   - Input from Gmail Trigger  
   - Output to Get a message

3. **Add Get a message node**  
   - Type: Gmail  
   - Operation: Get message by ID (use `={{ $('Gmail Trigger').item.json.id }}`)  
   - Download attachments with prefix "attachment_"  
   - Credential: Gmail OAuth2  
   - Connect output to Edit Fields1, Edit Fields2, Merge1, Debugged code to find attachment

4. **Add Edit Fields1 (Set) node**  
   - Create field `attachments`: array of binary keys (`={{ $('Get a message').item.binary.keys() }}`)  
   - Create field `message_id`: string with message ID (`={{ $json.id }}`)  
   - Connect output to Split Out

5. **Add Edit Fields2 (Set) node**  
   - Create field `attachments_metadata`: array of binary values (`={{ $('Get a message').item.binary.values() }}`)  
   - Create field `message_id`: string with message ID (`={{ $json.id }}`)  
   - Connect output to Split Out1

6. **Add Split Out node** (for attachments keys)  
   - Field to split out: `attachments`  
   - Include all other fields and binary data  
   - Connect output to Merge (position 0)

7. **Add Split Out1 node** (for attachments metadata)  
   - Field to split out: `attachments_metadata`  
   - Include all other fields and binary data  
   - Connect output to Merge (position 1)

8. **Add Merge node**  
   - Mode: Combine  
   - Combine by position  
   - Inputs: Split Out (pos 0), Split Out1 (pos 1)  
   - Output to Filter

9. **Add Filter node** (to filter PDFs)  
   - Condition: `attachments_metadata.fileType` equals `pdf` (case sensitive)  
   - Input: Merge  
   - Output: Merge1

10. **Add Merge1 node**  
    - Mode: Combine  
    - Advanced: true  
    - Merge by fields: `message_id` (left), `id` (right)  
    - Inputs: Get a message (pos 1), Filter (pos 0)  
    - Outputs: Extract from File, Convert Names to download files

11. **Add Debugged code to find attachment node**  
    - Type: Code (JavaScript)  
    - Logic: Scan binary attachments for filenames containing "invoice" (case-insensitive), create flags and list invoice filenames  
    - Input: Get a message  
    - Output: AI Agent

12. **Add OpenRouter Chat Model1 node**  
    - Model: google/gemini-2.5-flash-lite-preview-06-17  
    - Credential: OpenRouter API  
    - Connect output to Structured Output Parser1

13. **Add Structured Output Parser1 node**  
    - JSON Schema Example: `{ "Label": "Important", "Response": "Response" }`  
    - Connect output to AI Agent

14. **Add AI Agent node**  
    - Type: LangChain Agent  
    - Prompt:  
      - Include email subject, body text, attachment presence, and system message with categorization rules and response format (strict JSON with Label and Response).  
    - Inputs: Debugged code to find attachment, OpenRouter Chat Model1, Structured Output Parser1  
    - Output: Reply to same thread

15. **Add Reply to same thread node**  
    - Type: Gmail (reply in thread)  
    - Message: `={{ $json.output.Response }}`  
    - Thread ID: `={{ $('Gmail Trigger').item.json.threadId }}`  
    - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
    - Credential: Gmail OAuth2

16. **Add Extract from File node**  
    - Operation: PDF text extraction  
    - Binary Property Name: `={{ $json.attachments }}`  
    - Input: Merge1  
    - Output: Information Extractor

17. **Add Gemini node**  
    - Model: google/gemini-2.5-flash-lite-preview-06-17  
    - Credential: OpenRouter API  
    - Input: Information Extractor  
    - Output: Information Extractor

18. **Add Information Extractor node**  
    - Type: LangChain Information Extractor  
    - Input Text: `={{ $json.text }}` (from Extract from File node)  
    - Schema:  
      ```json
      {
        "invoice_number": "",
        "invoice_date": "",
        "due_date": "",
        "vendor_name": "",
        "total_amount": "",
        "currency": "",
        "items": [
          {
            "description": "",
            "amount": ""
          }
        ],
        "tax": "",
        "category": ""
      }
      ```  
    - System prompt instructs to extract relevant invoice info only  
    - Outputs: Extracting File Details

19. **Add Extracting File Details (Set) node**  
    - Extract key values from AI output into `output.invoice_number`, `output.invoice_date`, `output.vendor_name` etc.  
    - Input: Information Extractor  
    - Output: Send important details (Telegram)

20. **Add Convert Names to download files (Code) node**  
    - JavaScript to convert binary attachments into downloadable files with filenames for Telegram sending  
    - Input: Merge1  
    - Output: Remove Duplicates

21. **Add Remove Duplicates node**  
    - Removes duplicate files before sending to Telegram  
    - Input: Convert Names to download files  
    - Output: Send a document (e.g., dwg files)

22. **Add Send important details (Telegram) node**  
    - Text message template with invoice number, date, vendor, and full output JSON  
    - Credential: Telegram API  
    - Input: Extracting File Details

23. **Add Send a document (e.g., dwg files) (Telegram) node**  
    - Operation: sendDocument with binary data enabled  
    - Credential: Telegram API  
    - Input: Remove Duplicates

24. **Add Sticky Notes at appropriate places for visual grouping:**

    - "## MULTIPLE DOCUMENTS EXTRACTOR" near attachment splitting nodes  
    - "## Telegram - Invoice and details to Engineering dept." near Telegram sending nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                 |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Workflow uses Google Gemini 2.5 AI model via OpenRouter API for natural language processing and extraction      | OpenRouter API: https://openrouter.ai/                         |
| Gmail OAuth2 credentials required for Gmail Trigger, Get a message, and Reply nodes                             | Setup OAuth2 in n8n credentials panel                          |
| Telegram Bot API credentials required for sending messages and documents                                       | Telegram API setup: https://core.telegram.org/bots/api          |
| AI Agent system prompt enforces strict JSON output format with Label and Response fields                        | Ensures consistent parsing and automatic reply generation       |
| Invoice filename detection is based on presence of "invoice" in filename (case-insensitive)                    | May require adjustment for other naming conventions             |
| Invoice extraction schema covers common invoice fields: invoice number, dates, vendor, total, currency, items, tax | Customizable depending on invoice formats                         |
| Filtering attachments by fileType assumes metadata field `fileType` exists and correctly set                   | If missing, PDFs may be missed                                   |
| Telegram file size limits and supported document types apply for sending attachments                            | Large files may fail or require alternative handling            |
| System date/time (`$now`) is passed to AI prompt for contextual awareness                                      | Useful for timely and relevant AI responses                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.