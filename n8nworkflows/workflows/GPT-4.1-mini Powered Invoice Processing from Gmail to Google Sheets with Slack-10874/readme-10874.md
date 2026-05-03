GPT-4.1-mini Powered Invoice Processing from Gmail to Google Sheets with Slack

https://n8nworkflows.xyz/workflows/gpt-4-1-mini-powered-invoice-processing-from-gmail-to-google-sheets-with-slack-10874


# GPT-4.1-mini Powered Invoice Processing from Gmail to Google Sheets with Slack

### 1. Workflow Overview

This workflow automates invoice processing by integrating Gmail, AI-powered data extraction, Google Sheets logging, and Slack-based approval. It targets small business owners, finance teams, and freelancers aiming to streamline accounts payable by eliminating manual data entry.

The workflow consists of five logical blocks:

- **1.1 Input Reception:** Detect new invoice emails and fetch their attachments.
- **1.2 Data Extraction and Preparation:** Extract text from PDF attachments and prepare it for AI processing.
- **1.3 AI Processing and Logging:** Use GPT-4.1-mini to extract structured invoice data, then append it to Google Sheets.
- **1.4 Slack Approval Request:** Send invoice details with approve/reject links to Slack.
- **1.5 Approval/Rejection Handling:** Receive webhook responses from Slack links, update invoice status in Google Sheets, and send confirmation messages to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow on receipt of new Gmail messages matching the keyword "invoice," then downloads their attachments.

**Nodes Involved:**  
- `1. New Invoice Email Received`  
- `2. Get Email Attachments`

**Node Details:**

- **1. New Invoice Email Received**  
  - *Type:* Gmail Trigger  
  - *Role:* Watches Gmail inbox for new emails containing "invoice" in the query.  
  - *Config:* Search query `q` set to "invoice", excludes drafts, spam, trash; polls every minute.  
  - *Input:* None (trigger node)  
  - *Output:* Email metadata including message ID.  
  - *Potential Failures:* Gmail API rate limits, credential auth errors, no matching emails.  
  - *Sticky Note Reference:* Reminder to set Gmail credentials and customize query.

- **2. Get Email Attachments**  
  - *Type:* Gmail node  
  - *Role:* Downloads attachments from the triggered email message.  
  - *Config:* Downloads attachments (`downloadAttachments: true`), uses message ID from previous node.  
  - *Input:* Email message ID from node 1.  
  - *Output:* Binary data of attachments.  
  - *Potential Failures:* Missing attachments, download timeout, permission errors.

---

#### 1.2 Data Extraction and Preparation

**Overview:**  
Extracts raw text from the first PDF attachment and prepares a clean text field for AI processing.

**Nodes Involved:**  
- `3. Extract Text from PDF`  
- `4. Prepare Text for AI`

**Node Details:**

- **3. Extract Text from PDF**  
  - *Type:* ExtractFromFile  
  - *Role:* Extracts text content from PDF binary data.  
  - *Config:* Operation set to PDF extraction, uses binary property `attachment_0`.  
  - *Input:* Binary PDF from node 2.  
  - *Output:* Extracted raw text.  
  - *Failure Modes:* Corrupted PDF, unsupported PDF format, empty attachment.

- **4. Prepare Text for AI**  
  - *Type:* Set  
  - *Role:* Assigns a clean text field (`clean_text`) from extracted text or raw text, ensuring a string value.  
  - *Config:* Uses expression `{{$json.text || $json.rawText || ''}}`.  
  - *Input:* Extracted text from node 3.  
  - *Output:* JSON with `clean_text` field.  
  - *Failure Modes:* Missing or empty text fields.

- *Sticky Note Reference:* Emphasizes this block’s role in data preparation.

---

#### 1.3 AI Processing and Logging

**Overview:**  
Uses GPT-4.1-mini via Langchain to parse invoice details from text, then logs the data as a new row in Google Sheets with a pending status.

**Nodes Involved:**  
- `5. Extract Invoice Data with AI`  
- `Structured Output Parser`  
- `6. Log Invoice to Google Sheet`  
- `OpenAI Chat Model`

**Node Details:**

- **5. Extract Invoice Data with AI**  
  - *Type:* Langchain Agent (AI processor)  
  - *Role:* Sends prepared text to AI model to extract invoice fields strictly in JSON.  
  - *Config:* Custom prompt instructing to output JSON with keys: invoice_number, issue_date, bill_to_name, total_amount, due_date. Uses `clean_text` as input.  
  - *Input:* `clean_text` from node 4.  
  - *Output:* Structured JSON object with extracted invoice data.  
  - *Failure Modes:* AI model timeout, malformed response, empty or ambiguous text.

- **Structured Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Validates and parses AI output to ensure it matches the expected JSON schema.  
  - *Config:* JSON schema example enforcing data types and keys.  
  - *Input:* AI output from node 5.  
  - *Output:* Parsed structured invoice data.  
  - *Failure Modes:* Schema mismatch, parsing errors.

- **OpenAI Chat Model**  
  - *Type:* Langchain LM Chat OpenAI  
  - *Role:* Invokes GPT-4.1-mini model as backend for AI extraction.  
  - *Config:* Model set to "gpt-4.1-mini".  
  - *Input:* Prompt from node 5.  
  - *Output:* AI-generated JSON text.  
  - *Failure Modes:* API quota, authentication issues.

- **6. Log Invoice to Google Sheet**  
  - *Type:* Google Sheets  
  - *Role:* Appends a new row with extracted invoice data and status "pending".  
  - *Config:* Maps invoice fields to columns: invoice_number, company, amount, date, due_date, status ("pending"), rawText. Uses specific Sheet ID and sheet name placeholders.  
  - *Input:* Parsed AI data from node 5 and prepared text from node 4.  
  - *Output:* Confirmation of append operation.  
  - *Failure Modes:* Google API auth errors, incorrect Sheet ID or name, data mapping errors.

- *Sticky Note Reference:* Guidance for credentials and Google Sheet setup.

---

#### 1.4 Slack Approval Request

**Overview:**  
Sends a Slack message to a designated channel with invoice details and clickable links to approve or reject the invoice.

**Nodes Involved:**  
- `7. Send Approval Request to Slack`

**Node Details:**

- **7. Send Approval Request to Slack**  
  - *Type:* Slack node  
  - *Role:* Posts a formatted message containing key invoice data and approval/rejection links.  
  - *Config:* Uses Slack OAuth2 credentials, channel ID placeholder, text includes company, amount, due date, and two URL links referencing webhook endpoints with invoice number query parameter.  
  - *Input:* Invoice data from node 6 (Google Sheets append).  
  - *Output:* Confirmation of Slack message sent.  
  - *Failure Modes:* Slack API auth failure, incorrect channel ID, broken URLs if webhook base URL not configured.  
  - *Sticky Note Reference:* Emphasizes replacing placeholder URLs with production webhook URLs and adding Slack credentials.

---

#### 1.5 Approval/Rejection Handling

**Overview:**  
Waits for webhook calls triggered by Slack link clicks to approve or reject invoices, updates Google Sheets accordingly, and sends confirmation messages back to Slack.

**Nodes Involved:**  
- `Webhook (Approve)`  
- `Webhook (Reject)`  
- `8a. Extract Invoice ID`  
- `8b. Extract Invoice ID`  
- `9a. Update Status to Approved`  
- `9b. Update Status to Rejected`  
- `10a. Send Approval Confirmation`  
- `10b. Send Rejection Confirmation`

**Node Details:**

- **Webhook (Approve)**  
  - *Type:* Webhook  
  - *Role:* Receives HTTP requests for invoice approval via URL path `/approve`.  
  - *Config:* Path set to `approve`.  
  - *Input:* Query parameter `id` containing invoice number.  
  - *Output:* Passes payload downstream.  
  - *Failure Modes:* Incorrect webhook URL, missing query param, unauthorized calls.

- **8a. Extract Invoice ID**  
  - *Type:* Set node  
  - *Role:* Extracts `invoice_id` from webhook query parameter.  
  - *Config:* Assigns from `{{$json.query.id}}`.  
  - *Input:* Webhook data from `Webhook (Approve)`.  
  - *Output:* JSON with `invoice_id`.  
  - *Failure Modes:* Missing `id` parameter.

- **9a. Update Status to Approved**  
  - *Type:* Google Sheets  
  - *Role:* Updates the row matching `invoice_number` to set status to "approved".  
  - *Config:* Uses `invoice_id` from node 8a for matching and updates `status`.  
  - *Input:* Extracted invoice ID from node 8a.  
  - *Output:* Confirmation of update.  
  - *Failure Modes:* Row not found, auth errors.

- **10a. Send Approval Confirmation**  
  - *Type:* Slack node  
  - *Role:* Sends a confirmation message to Slack confirming invoice approval.  
  - *Config:* Uses OAuth2, posts to same channel ID, message includes invoice ID.  
  - *Input:* Output from node 9a.  
  - *Output:* Confirmation of Slack message sent.  
  - *Failure Modes:* Slack API issues.

- **Webhook (Reject)**  
  - *Type:* Webhook  
  - *Role:* Receives HTTP requests for invoice rejection via URL path `/reject`.  
  - *Config:* Path set to `reject`.  
  - *Input:* Query parameter `id`.  
  - *Output:* Passes payload downstream.  
  - *Failure Modes:* Same as approve webhook.

- **8b. Extract Invoice ID**  
  - *Type:* Set node  
  - *Role:* Extracts `invoice_id` from reject webhook query parameter.  
  - *Config:* Assigns from `{{$json.query.id}}`.  
  - *Input:* Webhook data from `Webhook (Reject)`.  
  - *Output:* JSON with `invoice_id`.  
  - *Failure Modes:* Missing `id` parameter.

- **9b. Update Status to Rejected**  
  - *Type:* Google Sheets  
  - *Role:* Updates the row matching `invoice_number` to set status to "rejected".  
  - *Config:* Uses `invoice_id` from node 8b for matching and updates `status`.  
  - *Input:* Extracted invoice ID from node 8b.  
  - *Output:* Confirmation of update.  
  - *Failure Modes:* Row not found, auth errors.

- **10b. Send Rejection Confirmation**  
  - *Type:* Slack node  
  - *Role:* Sends confirmation of invoice rejection to Slack channel.  
  - *Config:* OAuth2 authentication, channel ID same as approval confirmation.  
  - *Input:* Output from node 9b.  
  - *Output:* Slack message confirmation.  
  - *Failure Modes:* Slack API issues.

- *Sticky Note Reference:* Describes this block’s listening to Slack clicks and updating Google Sheets and Slack accordingly.

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                               | Input Node(s)                   | Output Node(s)                     | Sticky Note                                                                                                                    |
|--------------------------------|---------------------------------|----------------------------------------------|---------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 1. New Invoice Email Received   | Gmail Trigger                   | Trigger on new invoice email                  | -                               | 2. Get Email Attachments          | ## Trigger This node starts the workflow when an email matching the filter arrives. Setup: Add Gmail credentials; customize `q`. |
| 2. Get Email Attachments        | Gmail                          | Download email attachments                     | 1. New Invoice Email Received   | 3. Extract Text from PDF          | ## Data Preparation These nodes download the PDF from the email and extract its raw text content.                              |
| 3. Extract Text from PDF        | ExtractFromFile                | Extract text from PDF attachment                | 2. Get Email Attachments        | 4. Prepare Text for AI            | See above note                                                                                                                 |
| 4. Prepare Text for AI          | Set                           | Prepare clean text for AI input                  | 3. Extract Text from PDF        | 5. Extract Invoice Data with AI   | See above note                                                                                                                 |
| 5. Extract Invoice Data with AI | Langchain Agent (AI processor) | Extract structured invoice data from text      | 4. Prepare Text for AI          | 6. Log Invoice to Google Sheet    | ## AI Extraction & Logging Add OpenAI & Google Sheets credentials; enter Sheet ID & Name.                                      |
| Structured Output Parser        | Langchain Output Parser        | Validate and parse AI JSON output               | OpenAI Chat Model               | 5. Extract Invoice Data with AI   | See above note                                                                                                                 |
| OpenAI Chat Model               | Langchain LM Chat OpenAI       | Backend GPT-4.1-mini AI model                    | 5. Extract Invoice Data with AI | Structured Output Parser          | See above note                                                                                                                 |
| 6. Log Invoice to Google Sheet  | Google Sheets                  | Append extracted invoice data                    | 5. Extract Invoice Data with AI | 7. Send Approval Request to Slack | See above note                                                                                                                 |
| 7. Send Approval Request to Slack| Slack                         | Send invoice approval request with links        | 6. Log Invoice to Google Sheet  | -                                | ## Slack Approval Request Add Slack credentials; enter Slack Channel ID; replace webhook base URLs with production URLs.       |
| Webhook (Approve)              | Webhook                       | Receive invoice approval webhook call            | -                              | 8a. Extract Invoice ID            | ## Approval/Rejection Handling These nodes listen for Slack link clicks and update status in Google Sheets & Slack.            |
| 8a. Extract Invoice ID          | Set                           | Extract invoice_id from approval webhook query  | Webhook (Approve)              | 9a. Update Status to Approved     | See above note                                                                                                                 |
| 9a. Update Status to Approved   | Google Sheets                 | Update invoice status to "approved"              | 8a. Extract Invoice ID          | 10a. Send Approval Confirmation   | See above note                                                                                                                 |
| 10a. Send Approval Confirmation | Slack                        | Send approval confirmation message to Slack     | 9a. Update Status to Approved   | -                                | See above note                                                                                                                 |
| Webhook (Reject)              | Webhook                       | Receive invoice rejection webhook call           | -                              | 8b. Extract Invoice ID            | See above note                                                                                                                 |
| 8b. Extract Invoice ID          | Set                           | Extract invoice_id from rejection webhook query | Webhook (Reject)               | 9b. Update Status to Rejected     | See above note                                                                                                                 |
| 9b. Update Status to Rejected   | Google Sheets                 | Update invoice status to "rejected"              | 8b. Extract Invoice ID          | 10b. Send Rejection Confirmation  | See above note                                                                                                                 |
| 10b. Send Rejection Confirmation| Slack                        | Send rejection confirmation message to Slack    | 9b. Update Status to Rejected   | -                                | See above note                                                                                                                 |
| GPT-4.1-mini Powered Invoice Processing (Sticky Note) | Sticky Note | Overview of entire workflow and setup instructions | -                              | -                                | Workflow purpose, target users, detailed setup instructions, expected Google Sheet structure, and activation notes.             |
| Sticky Note                    | Sticky Note                   | Trigger block explanation                         | -                              | -                                | See node 1 sticky note details                                                                                                |
| Sticky Note1                   | Sticky Note                   | Data preparation block explanation                | -                              | -                                | See node 2-4 sticky note details                                                                                              |
| Sticky Note2                   | Sticky Note                   | AI extraction and logging explanation             | -                              | -                                | See node 5-6 sticky note details                                                                                              |
| Sticky Note4                   | Sticky Note                   | Slack approval request explanation                 | -                              | -                                | See node 7 sticky note details                                                                                                |
| Sticky Note5                   | Sticky Note                   | Approval/Rejection handling explanation            | -                              | -                                | See webhook and update nodes sticky note details                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Name: `1. New Invoice Email Received`  
   - Configure credentials for Gmail.  
   - Set search query `q` to `"invoice"`.  
   - Set polling interval to every minute.  
   - Exclude drafts, spam, trash.

2. **Add Gmail Node to Download Attachments**  
   - Type: Gmail  
   - Name: `2. Get Email Attachments`  
   - Configure with same Gmail credentials.  
   - Operation: "Get" message.  
   - Enable attachment download.  
   - Use expression for messageId: `={{$json.id}}`.  
   - Connect output of node 1 to this node.

3. **Add ExtractFromFile Node to Extract Text from PDF**  
   - Type: ExtractFromFile  
   - Name: `3. Extract Text from PDF`  
   - Operation: PDF extraction.  
   - Use binary property: `attachment_0`.  
   - Connect output of node 2 to this node.

4. **Add Set Node to Prepare Text for AI**  
   - Type: Set  
   - Name: `4. Prepare Text for AI`  
   - Create new field `clean_text` with expression: `{{$json.text || $json.rawText || ''}}`.  
   - Connect output of node 3 to this node.

5. **Add Langchain Agent Node for AI Extraction**  
   - Type: Langchain Agent  
   - Name: `5. Extract Invoice Data with AI`  
   - Use OpenAI GPT-4.1-mini model (configured separately).  
   - Set prompt: instruct AI to extract invoice_number, issue_date, bill_to_name, total_amount, due_date strictly in JSON as per provided prompt in original.  
   - Use `clean_text` field as input text.  
   - Connect output of node 4 to this node.

6. **Add Langchain Structured Output Parser Node**  
   - Type: Langchain Output Parser  
   - Name: `Structured Output Parser`  
   - Provide JSON schema example matching invoice data fields.  
   - Connect AI model output (OpenAI Chat Model node) to this parser, then parser output to Langchain Agent node.

7. **Add OpenAI Chat Model Node**  
   - Type: Langchain LM Chat OpenAI  
   - Name: `OpenAI Chat Model`  
   - Select model: `gpt-4.1-mini`.  
   - Connect output of Langchain Agent node to this node, then node output to Structured Output Parser.

8. **Add Google Sheets Node to Log Invoice**  
   - Type: Google Sheets  
   - Name: `6. Log Invoice to Google Sheet`  
   - Configure Google Sheets credentials.  
   - Set operation: Append.  
   - Enter your Google Sheet Document ID and Sheet Name.  
   - Map columns: invoice_number, company, amount, date, due_date, status (set to "pending"), rawText (from prepared text).  
   - Use expressions to pull these values from AI output and prepared text.  
   - Connect output of Langchain Agent node to this node.

9. **Add Slack Node to Send Approval Request**  
   - Type: Slack  
   - Name: `7. Send Approval Request to Slack`  
   - Configure Slack OAuth2 credentials.  
   - Enter Slack Channel ID.  
   - Compose message text with invoice details (company, amount, due date).  
   - Include approve/reject URLs pointing to your webhook endpoints with invoice number as query parameter.  
   - Connect output of Google Sheets append node (6) to this node.

10. **Add Webhook Nodes for Approval and Rejection**  
    - Type: Webhook  
    - Names: `Webhook (Approve)` with path `approve`, and `Webhook (Reject)` with path `reject`.  
    - No credentials needed.  
    - These receive HTTP requests from Slack link clicks.

11. **Add Set Nodes to Extract Invoice ID from Webhooks**  
    - Type: Set  
    - Names: `8a. Extract Invoice ID` (for approve webhook), `8b. Extract Invoice ID` (for reject webhook).  
    - Extract query parameter `id` into variable `invoice_id` using expression `{{$json.query.id}}`.  
    - Connect `Webhook (Approve)` to `8a`, `Webhook (Reject)` to `8b`.

12. **Add Google Sheets Nodes to Update Status**  
    - Type: Google Sheets  
    - Names: `9a. Update Status to Approved` and `9b. Update Status to Rejected`.  
    - Configure operation: Update row where `invoice_number` matches `invoice_id` extracted.  
    - Update `status` column to "approved" or "rejected" accordingly.  
    - Connect `8a` to `9a`, `8b` to `9b`.

13. **Add Slack Nodes to Send Confirmation Messages**  
    - Type: Slack  
    - Names: `10a. Send Approval Confirmation` and `10b. Send Rejection Confirmation`.  
    - Configure Slack credentials and channel ID.  
    - Compose confirmation messages including invoice ID.  
    - Connect `9a` to `10a`, `9b` to `10b`.

14. **Activate Workflow**  
    - Activate workflow in n8n.  
    - Copy production webhook URLs from `Webhook (Approve)` and `Webhook (Reject)` nodes.  
    - Replace placeholder URLs in Slack message node (`7. Send Approval Request to Slack`).  
    - Verify all credentials and IDs are correctly set.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automates your invoice processing pipeline from Gmail to Google Sheets with Slack approval, powered by GPT-4.1-mini AI model.                                                                                                                                                                                                                  | Sticky Note attached to workflow overview node                                                     |
| Expected Google Sheet columns: invoice_number, company, amount, date, due_date, status, rawText.                                                                                                                                                                                                                                                               | Workflow overview note                                                                             |
| Important: Replace `YOUR_SLACK_CHANNEL_ID`, `YOUR_GOOGLE_SHEET_ID`, `YOUR_SHEET_NAME`, and `https://YOUR_WEBHOOK_BASE_URL` placeholders with your actual configuration before activation.                                                                                                                                                                       | Multiple Slack and Google Sheets nodes notes                                                       |
| Slack messages contain clickable links to approve or reject invoices that hit webhook endpoints, triggering status updates and confirmation messages.                                                                                                                                                                                                         | Slack approval request and approval/rejection handling blocks notes                                |
| Ensure OAuth2 credentials are properly configured for Gmail, OpenAI, Google Sheets, and Slack nodes for seamless authentication.                                                                                                                                                                                                                              | Credential setup notes throughout workflow                                                        |
| Activate the workflow after all configuration; webhook URLs will be needed to replace placeholders in Slack message node for live operation.                                                                                                                                                                                                                 | Workflow overview and sticky notes                                                                |

---

**Disclaimer:** The provided content is derived exclusively from an n8n automated workflow and complies with all relevant content policies. It processes only lawful and publicly available data.