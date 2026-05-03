PDF Invoice Data Extraction & Tracking with Google Drive, Claude AI & Telegram

https://n8nworkflows.xyz/workflows/pdf-invoice-data-extraction---tracking-with-google-drive--claude-ai---telegram-6480


# PDF Invoice Data Extraction & Tracking with Google Drive, Claude AI & Telegram

### 1. Workflow Overview

This workflow automates the extraction, processing, logging, and notification of invoice data from PDF files uploaded to a specific Google Drive folder. It is designed for bookkeeping, accounting automation, and real-time billing alerts. The workflow uses AI-powered text extraction and information extraction techniques, integrates with Google Sheets for data logging, and sends notifications via Telegram to alert the billing team.

**Logical Blocks:**

- **1.1 Input Reception**: Monitors a Google Drive folder for new PDF invoice uploads and downloads the files.
- **1.2 PDF Text Extraction**: Extracts raw text from the downloaded PDF files.
- **1.3 Invoice Information Extraction (AI Processing)**: Uses a LangChain-based Information Extractor node to parse key invoice details from the extracted text.
- **1.4 Data Logging**: Appends the extracted invoice data into a Google Sheets document for record-keeping.
- **1.5 AI Notification Formatting**: Uses an Anthropic Claude AI model to generate a human-friendly formatted invoice notification message.
- **1.6 Notification Dispatch**: Sends the formatted invoice notification to a Telegram chat/channel.
- **1.7 Workflow End**: A no-op node to indicate workflow completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Watches a specified Google Drive folder for new files (invoices) and downloads the newly added PDF files.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Google Drive (Download)

- **Node Details:**

  - **Google Drive Trigger**
    - Type: Trigger node for Google Drive events.
    - Configuration: Watches the folder with ID `1PgLKqvN8CFFXWAKhZxzjuk6gMnXJ7-np` for newly created files, polling every minute.
    - Inputs: None (trigger node).
    - Outputs: Triggers when a new file is created in the folder.
    - Potential Failures: Authentication errors, API rate limits, folder ID misconfiguration.
  
  - **Google Drive**
    - Type: Google Drive file operation node.
    - Configuration: Downloads the file using the ID provided by the trigger.
    - Inputs: File ID from Google Drive Trigger node.
    - Outputs: Binary data of the downloaded PDF file.
    - Potential Failures: File not found, permission errors, download timeouts.

---

#### 1.2 PDF Text Extraction

- **Overview:**  
  Extracts the textual content from the downloaded PDF invoice files.

- **Nodes Involved:**  
  - Extract from File

- **Node Details:**

  - **Extract from File**
    - Type: File extraction node.
    - Configuration: Operation set to "pdf" to extract text from PDF binary content.
    - Inputs: Binary PDF file from Google Drive node.
    - Outputs: Extracted plain text of the PDF.
    - Potential Failures: Corrupted PDFs, unsupported PDF formats, extraction errors.

---

#### 1.3 Invoice Information Extraction (AI Processing)

- **Overview:**  
  Uses AI to extract structured invoice data such as invoice number, client name/email, amounts, and dates from the PDF text.

- **Nodes Involved:**  
  - Information Extractor  
  - OpenAI Chat Model (linked as AI language model provider, but not directly connected in main flow)

- **Node Details:**

  - **Information Extractor**
    - Type: LangChain based AI node specialized in information extraction.
    - Configuration:
      - System prompt directs the AI to extract relevant invoice data only.
      - Attributes extracted: Invoice Number (required), Client Name (required), Client Email (required), Total Amount (required), Invoice Date (required), Due Date (optional).
      - Input text is the extracted PDF text.
    - Inputs: Text from Extract from File node.
    - Outputs: JSON object with extracted invoice fields.
    - Potential Failures: AI extraction inaccuracies, incomplete data, missing required attributes, API quota limits.
    - Version Requirements: LangChain integration node (v1).
  
  - **OpenAI Chat Model**
    - Type: AI language model node (OpenAI).
    - Configuration: Connected as an AI model for potential use, but not involved in the main flow triggers.
    - Purpose: Could be used for auxiliary AI processing if configured.
    - Potential Failures: Authentication, network, quota.

---

#### 1.4 Data Logging

- **Overview:**  
  Appends the extracted invoice data to a designated Google Sheets document for tracking and bookkeeping.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**
    - Type: Google Sheets append operation node.
    - Configuration:
      - Document ID: `1lMMuPAU_rRU6VdybxgiPeVd9KRZrKhlzJ3gVc4Wz7iA`
      - Sheet Name: `Sheet1` (gid=0)
      - Operation: Append row with extracted invoice attributes mapped explicitly.
      - Columns mapped: Invoice Number, Client Name, Client Email, Total Amount, Invoice Date, Due Date.
    - Inputs: JSON output from Information Extractor.
    - Outputs: Confirmation of row append operation.
    - Potential Failures: Authentication, incorrect sheet ID or permissions, schema mismatch.

---

#### 1.5 AI Notification Formatting

- **Overview:**  
  Formats the extracted invoice data into a clear, human-readable message for notification purposes using Anthropic Claude AI.

- **Nodes Involved:**  
  - Anthropic Agent  
  - Anthropic Chat Model

- **Node Details:**

  - **Anthropic Chat Model**
    - Type: AI language model node using Anthropic Claude Sonnet 4.
    - Configuration: Model set to `claude-sonnet-4-20250514`.
    - Inputs: Connected as AI language model provider for Anthropic Agent.
    - Potential Failures: API key, rate limits, network issues.
  
  - **Anthropic Agent**
    - Type: LangChain chain node using Anthropic Claude model.
    - Configuration:
      - Input text template includes invoice details.
      - System message instructs the AI to act as a telegram notification expert and craft a billing team notification message.
    - Inputs: Data from Google Sheets node (invoice data).
    - Outputs: Formatted notification message.
    - Potential Failures: AI misinterpretation, incomplete input data, API errors.

---

#### 1.6 Notification Dispatch

- **Overview:**  
  Sends the formatted invoice notification message to a predefined Telegram chat/channel for billing team alerts.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**
    - Type: Messaging node for Telegram Bot.
    - Configuration:
      - Chat ID configured (redacted in JSON for privacy).
      - Message text set from the output of Anthropic Agent.
    - Inputs: Formatted message from Anthropic Agent.
    - Outputs: Confirmation of message sent.
    - Potential Failures: Invalid bot token or chat ID, network errors, Telegram API rate limits.

---

#### 1.7 Workflow End

- **Overview:**  
  No-operation node to mark the workflow endpoint after sending Telegram notification.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

  - **No Operation, do nothing**
    - Type: No-op node.
    - Configuration: No parameters.
    - Inputs: Output from Telegram node.
    - Outputs: None.
    - Purpose: Represents workflow completion; useful for workflow clarity.
  
---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                  | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                                                                                                                                                                                                                    |
|-------------------------|-----------------------------------------|--------------------------------|------------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger     | n8n-nodes-base.googleDriveTrigger        | Watches folder for new invoices | None                   | Google Drive          | Upload Invoice Doc Trigger                                                                                                                                                                                                                                                                    |
| Google Drive            | n8n-nodes-base.googleDrive                | Downloads the newly uploaded file | Google Drive Trigger    | Extract from File     | Upload Invoice Doc Trigger                                                                                                                                                                                                                                                                    |
| Extract from File       | n8n-nodes-base.extractFromFile            | Extracts text from PDF          | Google Drive           | Information Extractor | Extract Invoice Doc                                                                                                                                                                                                                                                                           |
| Information Extractor   | @n8n/n8n-nodes-langchain.informationExtractor | Extracts invoice data via AI    | Extract from File       | Google Sheets         | Extract Information & Log                                                                                                                                                                                                                                                                     |
| Google Sheets           | n8n-nodes-base.googleSheets                | Logs extracted data into sheet | Information Extractor   | Anthropic Agent       | Extract Information & Log                                                                                                                                                                                                                                                                     |
| Anthropic Agent         | @n8n/n8n-nodes-langchain.chainLlm          | Formats notification message   | Google Sheets           | Telegram              | Telegram Notification                                                                                                                                                                                                                                                                          |
| Anthropic Chat Model    | @n8n/n8n-nodes-langchain.lmChatAnthropic    | AI model for formatting (Claude) | Anthropic Agent (AI model) | Anthropic Agent (uses) | Telegram Notification                                                                                                                                                                                                                                                                          |
| Telegram                | n8n-nodes-base.telegram                     | Sends notification message     | Anthropic Agent         | No Operation, do nothing | Notification & End                                                                                                                                                                                                                                                                             |
| No Operation, do nothing | n8n-nodes-base.noOp                         | Marks workflow completion     | Telegram                | None                 | Notification & End                                                                                                                                                                                                                                                                             |
| Sticky Note             | n8n-nodes-base.stickyNote                    | Documentation                  | None                   | None                 | 📥 Invoice Intake & Notification Workflow - Full description with video link: https://www.youtube.com/@automatewithmarc                                                                                                                                                                        |
| Sticky Note1            | n8n-nodes-base.stickyNote                    | Documentation                  | None                   | None                 | Extract Invoice Doc                                                                                                                                                                                                                                                                           |
| Sticky Note2            | n8n-nodes-base.stickyNote                    | Documentation                  | None                   | None                 | Extract Information & Log                                                                                                                                                                                                                                                                     |
| Sticky Note3            | n8n-nodes-base.stickyNote                    | Documentation                  | None                   | None                 | Telegram Notification                                                                                                                                                                                                                                                                          |
| Sticky Note4            | n8n-nodes-base.stickyNote                    | Documentation                  | None                   | None                 | Notification & End                                                                                                                                                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Node Type: `Google Drive Trigger`  
   - Operation: Event = `fileCreated`  
   - Folder to watch: Set to specific folder ID `1PgLKqvN8CFFXWAKhZxzjuk6gMnXJ7-np`  
   - Poll frequency: Every minute  
   - Credentials: Google Drive OAuth2 credentials with access to the folder.

2. **Create Google Drive Node (Download File)**  
   - Node Type: `Google Drive`  
   - Operation: `download`  
   - File ID: Use expression `={{ $json.id }}` from the trigger output  
   - Credentials: Same Google Drive OAuth2 credentials.

3. **Create Extract from File Node**  
   - Node Type: `Extract from File`  
   - Operation: `pdf`  
   - Input: Binary data from Google Drive node  
   - No additional credentials required.

4. **Create Information Extractor Node (LangChain)**  
   - Node Type: `Information Extractor` (LangChain integration)  
   - Input Text: Expression `={{ $json.text }}` from Extract from File node  
   - Configure system prompt: "You are an expert extraction algorithm. Only extract relevant information from the text. If you do not know the value of an attribute asked to extract, you may omit the attribute's value."  
   - Attributes to extract:  
     - Invoice Number (required)  
     - Client Name (required)  
     - Client Email (required)  
     - Total Amount (required)  
     - Invoice Date (required)  
     - Due Date (optional)  
   - Version: Use LangChain compatible version (v1).

5. **Create Google Sheets Node**  
   - Node Type: `Google Sheets`  
   - Operation: `append`  
   - Document ID: `1lMMuPAU_rRU6VdybxgiPeVd9KRZrKhlzJ3gVc4Wz7iA` (your invoice tracking sheet)  
   - Sheet Name: `Sheet1` (gid=0)  
   - Columns mapping: Map extracted fields explicitly:  
     - Invoice Number, Client Name, Client Email, Total Amount, Invoice Date, Due Date  
   - Credentials: Google Sheets OAuth2 with editing permissions.

6. **Create Anthropic Chat Model Node**  
   - Node Type: `Anthropic Chat Model`  
   - Model: Select `claude-sonnet-4-20250514` or equivalent Claude Sonnet 4 model  
   - Credentials: Anthropic API key.

7. **Create Anthropic Agent Node**  
   - Node Type: `Anthropic Agent` (LangChain chainLlm)  
   - Input Text Template:  
     ```
     Invoice Number:{{ $json['Invoice Number'] }}
     Client Name:{{ $json['Client Name'] }}
     Client Email:{{ $json['Client Email'] }}
     Total Amount:{{ $json['Total Amount'] }}
     Invoice Due Date:{{ $json['Invoice Date'] }}
     ```
   - System Message: "#Overview You are a telegram notification expert. You will receive invoice information. You will craft a message notifying the billing team of the invoice and the available information of the invoice."  
   - AI Model: Use Anthropic Chat Model node as AI language model provider.

8. **Create Telegram Node**  
   - Node Type: `Telegram`  
   - Chat ID: Add your Telegram chat/group ID  
   - Message Text: Use output from Anthropic Agent node  
   - Credentials: Telegram Bot token configured.

9. **Create No Operation Node**  
   - Node Type: `No Operation`  
   - Connect output of Telegram node to this node to mark workflow completion.

10. **Connect Nodes in Sequence:**  
    Google Drive Trigger → Google Drive → Extract from File → Information Extractor → Google Sheets → Anthropic Agent → Telegram → No Operation

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| 📥 Invoice Intake & Notification Workflow description: Automates monitoring of Google Drive folder, extracts invoice data using AI, logs to Google Sheets, and notifies billing team via Telegram. Video resource for similar workflows: https://www.youtube.com/@automatewithmarc                                                                                                                                   | Workflow description and video resource                   |
| Uses LangChain AI nodes for extraction and formatting, Anthropic Claude Sonnet 4 for advanced natural language generation, and integrates with Google Drive, Sheets, and Telegram for end-to-end automation.                                                                                                                                                          | Technology stack                                           |
| Ensure proper OAuth2 credentials are set up for Google Drive and Google Sheets with required permissions. Anthropic and Telegram API keys must be valid and configured.                                                                                                                                                                                               | Credential setup note                                     |
| Workflow designed for PDFs stored in a Google Drive folder; file format or extraction errors may occur with non-standard PDFs or poorly scanned documents. Validate input files for best results.                                                                                                                                                                      | Input file format caution                                 |
| Telegram chat ID must be obtained from the bot or target channel/group; bot must have permissions to send messages there.                                                                                                                                                                                                                                             | Telegram setup note                                       |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. All data handled within is legal and publicly accessible. No illegal or offensive content is present.