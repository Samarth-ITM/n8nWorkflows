Smart Email Manager with Gmail, GPT-4 Classification & Auto-Responses

https://n8nworkflows.xyz/workflows/smart-email-manager-with-gmail--gpt-4-classification---auto-responses-7855


# Smart Email Manager with Gmail, GPT-4 Classification & Auto-Responses

### 1. Workflow Overview

This workflow titled **"AI Email Assistant - Smart Email Processing & Response"** automates intelligent email management for Gmail accounts using GPT-4 AI classification and auto-response generation. It is designed for business productivity enhancement by automating email triage, classification, labeling, attachment handling, and drafting replies.

**Target Use Cases:**
- Automated email categorization by priority and type
- Smart labeling and tagging of emails
- Urgent email alerting via Telegram
- Attachment filtering, duplication check, and upload to Google Drive
- Drafting professional replies with AI assistance for review before sending

**Logical Blocks:**

- **1.1 Email Reception & Thread Context:** Trigger on Gmail new emails, retrieve full conversation threads, and extract the latest message content.
- **1.2 Label Management:** Fetch and filter Gmail user labels dynamically to enable AI-driven email categorization.
- **1.3 AI-Based Email Classification:** Use GPT-4 to analyze email content and metadata, output structured classification with priorities, action items, and labels.
- **1.4 Smart Routing & Actions:** Route the email processing flow based on classification results to generate responses, upload attachments, send alerts, or assign labels.
- **1.5 Attachment Handling:** Filter attachments (optionally by type), check for duplicates in Google Drive, and upload new files with a structured naming convention.
- **1.6 Response Generation:** AI-generated context-aware draft email responses matching tone and formality.
- **1.7 Post-Processing:** Assign classification labels, mark emails as processed, and generate urgent alerts if needed.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Reception & Thread Context

**Overview:**  
This block triggers on new incoming Gmail messages (excluding self-sent), retrieves the full conversation thread, and extracts the latest message’s content for analysis.

**Nodes Involved:**  
- Email_Trigger (Gmail Trigger)  
- Process One by One (SplitInBatches)  
- Get Conversation Thread (Gmail API)  
- Get Latest Message ID (Limit)  
- Get Latest Message Content (Gmail API)

**Node Details:**

- **Email_Trigger:**  
  - Type: Gmail Trigger  
  - Config: Polls every hour, excludes emails from self (`-from:me`), downloads attachments.  
  - Input: None (trigger)  
  - Output: New email event with full metadata and attachments  
  - Edge Cases: Network/API errors, authentication expiration, attachment download failures.

- **Process One by One:**  
  - Type: SplitInBatches  
  - Purpose: Process emails individually to avoid concurrency issues.  
  - Input: Email_Trigger output  
  - Output: Single email data for downstream nodes  
  - Edge Cases: Batch size defaults, possible delays if many emails arrive simultaneously.

- **Get Conversation Thread:**  
  - Type: Gmail API (Thread resource)  
  - Config: Retrieves all messages in the thread by threadId from the trigger.  
  - Input: Thread ID from Email_Trigger  
  - Output: Full thread messages array  
  - Edge Cases: Thread not found, API limits, retry enabled.

- **Get Latest Message ID:**  
  - Type: Limit  
  - Config: Keeps the last item (latest message) from the thread messages.  
  - Input: Get Conversation Thread output  
  - Output: Message ID of latest email in the thread.

- **Get Latest Message Content:**  
  - Type: Gmail API (Message resource)  
  - Config: Retrieves full content of the latest message by message ID.  
  - Input: Message ID from previous node  
  - Output: Full message content including text, HTML, headers, attachments metadata  
  - Edge Cases: Missing message, API errors, retry enabled.

---

#### 1.2 Label Management

**Overview:**  
Fetches all Gmail labels, filters to include only user-created labels, and aggregates them to provide the latest label set for AI classification.

**Nodes Involved:**  
- Get All Gmail Labels (Gmail API)  
- Filter User Labels (If)  
- Get User Labels (Aggregate)

**Node Details:**

- **Get All Gmail Labels:**  
  - Type: Gmail API (Label resource)  
  - Config: Retrieves all labels from Gmail account.  
  - Output: List of all labels, including system and user labels.

- **Filter User Labels:**  
  - Type: If  
  - Config: Filters labels where type equals "user" (excludes system labels).  
  - Input: Output from Get All Gmail Labels  
  - Output: User labels only.

- **Get User Labels:**  
  - Type: Aggregate  
  - Config: Merges list of user label IDs and names into a single list.  
  - Output: Aggregated user label data for AI input.

---

#### 1.3 AI-Based Email Classification

**Overview:**  
Uses GPT-4 (via LangChain node) to analyze the latest email content, metadata, attachments presence, and user labels to generate structured classification including priority, required actions, and label assignments.

**Nodes Involved:**  
- AI Email Classifier (LangChain chainLlm)  
- Structured Output Parser (LangChain outputParserStructured)  
- Auto-fixing Output Parser (LangChain outputParserAutofixing)  
- AI Language Model (OpenAI GPT-4 interface)

**Node Details:**

- **AI Email Classifier:**  
  - Type: LangChain Chain LLM  
  - Config: Sends structured prompt with email content, subject, metadata (sender, date), attachments flag, user labels, current Gmail labels, and conversation context.  
  - Output: JSON with metadata, classification, and actions fields.  
  - Edge Cases: Model timeouts, malformed JSON output, insufficient context.

- **Structured Output Parser:**  
  - Type: LangChain Output Parser (Structured)  
  - Config: Validates and parses AI output against JSON schema specifying required classification fields.  
  - Ensures structured data correctness.

- **Auto-fixing Output Parser:**  
  - Type: LangChain Output Parser (Autofixing)  
  - Config: Attempts to fix any output parsing errors automatically using AI.  
  - Connected in chain to ensure robust parsing.

- **AI Language Model:**  
  - Type: LangChain OpenAI GPT-4 Model  
  - Config: Model set to "gpt-4o-mini" for balance of performance and cost.  
  - Provides LLM inference for classification and response generation.

---

#### 1.4 Smart Routing & Actions

**Overview:**  
Based on AI classification output, routes email processing into multiple parallel branches for response generation, attachment processing, urgent alerting, and labeling.

**Nodes Involved:**  
- Smart Action Router (Switch)  
- AI Response Generator (LangChain chainLlm)  
- Check for Attachments (If)  
- Send Urgent Alert (Telegram)  
- Aggregate Label IDs (Aggregate)

**Node Details:**

- **Smart Action Router:**  
  - Type: Switch  
  - Config: Routes based on booleans in classification output: toReply, attachmentsToUpload, urgent, and a default label assignment path.  
  - Output branches: RESPONSE, ATTACHMENTS, ALERT, LABEL  
  - Edge Cases: Multiple true conditions trigger multiple branches.

- **AI Response Generator:**  
  - Type: LangChain Chain LLM  
  - Config: Generates professional, customized draft email responses using latest message subject and content; follows tone/style guidelines.  
  - Output: Draft email text for review.

- **Check for Attachments:**  
  - Type: If  
  - Config: Checks if the email has attachments to process (boolean from classification).  
  - Routes to attachment handling nodes if true.

- **Send Urgent Alert:**  
  - Type: Telegram  
  - Config: Sends a formatted urgent email alert message to a Telegram chat ID, including sender, subject, summary, and Gmail link.  
  - Edge Cases: Telegram API failures, invalid chat ID.

- **Aggregate Label IDs:**  
  - Type: Aggregate  
  - Config: Gathers label IDs from classification output to assign to the Gmail message.

---

#### 1.5 Attachment Handling

**Overview:**  
Handles email attachments by optionally filtering for specific file types, checking for duplicates in Google Drive, and uploading new attachments with a structured naming convention.

**Nodes Involved:**  
- Check for Attachments (If) [from previous block]  
- Get All Attachments (Function)  
- Get Specific File Types (Function) [optional]  
- Check Existing Attachments (Google Drive)  
- Check if File Exists (If)  
- Merge Attachment Data (Merge)  
- Upload to Google Drive (Google Drive)

**Node Details:**

- **Get All Attachments:**  
  - Type: Function  
  - Config: Extracts all binary attachments from the email trigger node, outputs each attachment with filename and mime type.  
  - Edge Cases: No attachments, invalid binary data.

- **Get Specific File Types:**  
  - Type: Function  
  - Config: Filters attachments to include only PDFs, XMLs, JPEGs (configurable) for security or compliance.  
  - This node is optional and can be switched with Get All Attachments.  
  - Sticky note explains file type filtering usage.

- **Check Existing Attachments:**  
  - Type: Google Drive API  
  - Config: Searches the configured Google Drive folder for files matching the naming convention based on email date, sender, and filename.  
  - Output: Existing files matching the name.  
  - Edge Cases: API errors, folder ID misconfiguration.

- **Check if File Exists:**  
  - Type: If  
  - Config: Checks if the previous search returned any existing file (duplicate detection).  
  - If no existing file, attachment proceeds to upload.

- **Merge Attachment Data:**  
  - Type: Merge  
  - Config: Chooses branch data to continue with attachment upload or skip if duplicate found.

- **Upload to Google Drive:**  
  - Type: Google Drive API  
  - Config: Uploads new attachments to a specified folder with a file naming format: `YYYY.MM.DD - sender@email.com - filename.ext`  
  - Credentials required for Google Drive OAuth2.  
  - Edge Cases: Upload failures, quota exceeded.

---

#### 1.6 Response Generation

**Overview:**  
Generates an AI-based draft reply email tailored to the received email’s tone, content, and urgency, providing two alternative responses for yes/no questions.

**Nodes Involved:**  
- AI Response Generator (LangChain chainLlm) [from Smart Routing]  
- Create Draft Response (Gmail API)

**Node Details:**

- **AI Response Generator:**  
  - As detailed above, creates professional response text.

- **Create Draft Response:**  
  - Type: Gmail API (Draft resource)  
  - Config: Creates a draft email in Gmail with generated text, formatted as HTML, linked to original thread and sender.  
  - Input: Draft text from AI Response Generator, original email thread ID and sender email.  
  - Edge Cases: API errors, email formatting issues.

---

#### 1.7 Post-Processing

**Overview:**  
Labels the processed email with classification labels, marks it as processed with a custom label, and ensures the email is flagged as handled.

**Nodes Involved:**  
- Aggregate Label IDs (Aggregate) [from Smart Routing]  
- Assign Classification Label (Gmail API)  
- Mark as Processed (Gmail API)

**Node Details:**

- **Assign Classification Label:**  
  - Type: Gmail API (Modify Message)  
  - Config: Adds label(s) derived from AI classification to the email message.  
  - Input: Label IDs aggregated earlier, email message ID.  
  - Edge Cases: Label ID invalid, API failures.

- **Mark as Processed:**  
  - Type: Gmail API (Modify Message)  
  - Config: Adds a predefined "Processed" label to mark email as handled.  
  - On error: Continues workflow without blocking (non-critical).  
  - Edge Cases: Label creation required beforehand.

---

### 3. Summary Table

| Node Name               | Node Type                                           | Functional Role                        | Input Node(s)                  | Output Node(s)                         | Sticky Note                                                                                          |
|-------------------------|----------------------------------------------------|-------------------------------------|-------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| 📋 Optional File Filtering | Sticky Note                                         | Explains optional file filtering     | -                             | -                                     | Switch between all attachments or specific file types (PDF/XML). Guide: https://docs.n8n.io/workflows/sticky-notes/ |
| 📧 Email Processing      | Sticky Note                                         | Describes email processing pipeline  | -                             | -                                     | Flow steps and features, thread-aware responses. Guide: https://docs.n8n.io/workflows/sticky-notes/  |
| ✍️ Response Generation   | Sticky Note                                         | Describes AI response generation     | -                             | -                                     | Context-aware, professional responses. Guide: https://docs.n8n.io/workflows/sticky-notes/            |
| Auto-fixing Output Parser| LangChain Output Parser Auto-fixing                 | Fixes AI output parsing errors       | Structured Output Parser       | AI Email Classifier                    |                                                                                                    |
| 🔍 Email Analysis        | Sticky Note                                         | AI classification overview           | -                             | -                                     | Priority, category, urgency detection. Guide: https://docs.n8n.io/workflows/sticky-notes/             |
| Structured Output Parser | LangChain Output Parser Structured                   | Validates AI output structure        | AI Language Model             | Auto-fixing Output Parser              |                                                                                                    |
| 🏷️ Smart Labeling         | Sticky Note                                         | Explains smart labeling system       | -                             | -                                     | Auto-assigns labels for organization. Guide: https://docs.n8n.io/workflows/sticky-notes/              |
| 📎 Attachment Handler    | Sticky Note                                         | Attachment processing explanation    | -                             | -                                     | Auto upload, file filtering, duplicate detection. Guide: https://docs.n8n.io/workflows/sticky-notes/  |
| 🚨 Alert System          | Sticky Note                                         | Urgent notifications explanation     | -                             | -                                     | Telegram/Slack alerts for urgent emails. Guide: https://docs.n8n.io/workflows/sticky-notes/          |
| Check for Attachments    | If                                                  | Checks if email has attachments      | Smart Action Router           | Get All Attachments                    |                                                                                                    |
| Email_Trigger           | Gmail Trigger                                       | Triggers on new incoming emails      | -                             | Process One by One                     |                                                                                                    |
| Get Conversation Thread | Gmail                                              | Retrieves full conversation thread   | Process One by One            | Get Latest Message ID                  | Retry enabled                                                                                      |
| Get Latest Message ID   | Limit                                              | Gets the latest message in thread    | Get Conversation Thread       | Get Latest Message Content             |                                                                                                    |
| Get Latest Message Content | Gmail                                            | Retrieves latest message content     | Get Latest Message ID         | Get All Gmail Labels                   | Retry enabled                                                                                      |
| AI Email Classifier     | LangChain Chain LLM                                | AI classifies email content          | Get User Labels               | Smart Action Router                    | Uses GPT-4 mini model                                                                             |
| Smart Action Router     | Switch                                             | Routes based on classification       | AI Email Classifier           | AI Response Generator, Check for Attachments, Send Urgent Alert, Aggregate Label IDs |                                                                                                    |
| Aggregate Label IDs     | Aggregate                                          | Aggregates labels for assignment     | Smart Action Router           | Mark as Processed, Assign Classification Label |                                                                                                    |
| AI Response Generator   | LangChain Chain LLM                                | Generates AI email response drafts   | Smart Action Router           | Create Draft Response                  |                                                                                                    |
| Create Draft Response   | Gmail                                              | Creates draft email in Gmail         | AI Response Generator         | -                                     |                                                                                                    |
| Assign Classification Label | Gmail                                           | Assigns labels to Gmail message      | Aggregate Label IDs           | -                                     |                                                                                                    |
| Mark as Processed       | Gmail                                              | Adds "Processed" label to email      | Aggregate Label IDs           | Process One by One                     | Continues on error                                                                                  |
| Send Urgent Alert       | Telegram                                           | Sends urgent email alerts            | Smart Action Router           | -                                     |                                                                                                    |
| Get All Gmail Labels    | Gmail                                              | Fetches all Gmail labels             | Get Latest Message Content    | Filter User Labels                    |                                                                                                    |
| Filter User Labels      | If                                                 | Filters only user-created labels     | Get All Gmail Labels          | Get User Labels                      |                                                                                                    |
| Get User Labels         | Aggregate                                          | Aggregates filtered user labels      | Filter User Labels            | AI Email Classifier                   |                                                                                                    |
| Check Existing Attachments | Google Drive                                     | Searches Google Drive for duplicates | Get All Attachments           | Check if File Exists                  | Folder ID must be configured                                                                     |
| Check if File Exists    | If                                                 | Checks if attachment file exists     | Check Existing Attachments    | Merge Attachment Data                 |                                                                                                    |
| Merge Attachment Data   | Merge                                              | Merges data of attachments to upload | Check if File Exists          | Upload to Google Drive                | Chooses branch for upload or skip                                                                |
| Upload to Google Drive  | Google Drive                                       | Uploads attachments to Drive         | Merge Attachment Data         | -                                     | Folder ID must be configured                                                                     |
| Get All Attachments     | Function                                           | Extracts all binary attachments      | Check for Attachments         | Check Existing Attachments            |                                                                                                    |
| Get Specific File Types | Function                                           | Filters attachments by MIME type     | (Optional alternative to Get All Attachments) | -                            | File types: PDF, XML, JPEG                                                                        |
| ☁️ File Management      | Sticky Note                                         | Explains attachment naming & storage| -                             | -                                     | Naming convention explained. Guide in note.                                                     |
| 🤖 Key Features         | Sticky Note                                         | Summarizes AI assistant capabilities | -                             | -                                     | Time-saving and feature overview                                                                |
| 🔄 Label Sync           | Sticky Note                                         | Explains label synchronization       | -                             | -                                     | Auto-syncs Gmail user labels for classification                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node ("Email_Trigger"):**  
   - Type: Gmail Trigger  
   - Filters: Query `-from:me` to exclude own emails  
   - Options: Enable `downloadAttachments`  
   - Poll Interval: Every hour  
   - Credentials: Configure Gmail OAuth2.

2. **Add SplitInBatches Node ("Process One by One"):**  
   - Set batch size as needed (default 1)  
   - Connect from Gmail Trigger output.

3. **Add Gmail Node to Get Conversation Thread ("Get Conversation Thread"):**  
   - Resource: Thread  
   - Operation: Get  
   - Thread ID: Expression `{{$json["threadId"]}}` from Email_Trigger  
   - Connect from "Process One by One" output.

4. **Add Limit Node ("Get Latest Message ID"):**  
   - Keep: last item  
   - Connect from "Get Conversation Thread".

5. **Add Gmail Node to Get Latest Message Content ("Get Latest Message Content"):**  
   - Resource: Message  
   - Operation: Get  
   - Message ID: Expression from "Get Latest Message ID" `{{$json["id"]}}`  
   - Connect from "Get Latest Message ID".

6. **Add Gmail Node to Get All Gmail Labels ("Get All Gmail Labels"):**  
   - Resource: Label  
   - Return All: true  
   - Connect from "Get Latest Message Content".

7. **Add If Node ("Filter User Labels"):**  
   - Condition: Label type equals "user"  
   - Connect from "Get All Gmail Labels".

8. **Add Aggregate Node ("Get User Labels"):**  
   - Merge Lists: true  
   - Aggregate fields: `id` and `name` from filtered labels  
   - Connect from "Filter User Labels" true output.

9. **Add LangChain AI Language Model Node ("AI Language Model"):**  
   - Model: GPT-4 (e.g., "gpt-4o-mini")  
   - Credentials: OpenAI API key  
   - No direct connection yet; will be used by classification and response nodes.

10. **Add LangChain Chain LLM Node ("AI Email Classifier"):**  
    - Use LangChain chain LLM with prompt as per workflow description, incorporating inputs: latest message content, subject, metadata, attachments boolean, user labels, thread context.  
    - Enable Output Parser with structured JSON schema for classification.  
    - Connect input from "Get User Labels" and AI Language Model node.

11. **Add LangChain Output Parser Nodes:**  
    - Structured Output Parser (validates AI output)  
    - Auto-fixing Output Parser (attempts corrections)  
    - Connect these parsers in chain before classification output.

12. **Add Switch Node ("Smart Action Router"):**  
    - Add rules for classification booleans: `toReply`, `attachmentsToUpload`, `urgent`, and default for labels.  
    - Connect from "AI Email Classifier".

13. **For Response Generation Branch:**  
    - Add LangChain Chain LLM Node ("AI Response Generator") with prompt to generate professional email reply drafts.  
    - Connect from "Smart Action Router" RESPONSE output.

14. **Add Gmail Node ("Create Draft Response"):**  
    - Resource: Draft  
    - Operation: Create  
    - Email content: Expression from AI Response Generator output  
    - Thread ID and recipient from Email_Trigger  
    - Connect from "AI Response Generator".

15. **For Attachment Handling Branch:**  
    - Add If Node ("Check for Attachments") from router output.  
    - Add Function Node ("Get All Attachments") to extract binary attachments from Email_Trigger.  
    - Optionally replace with Function Node ("Get Specific File Types") for filtering PDFs/XMLs/JPEGs.  
    - Add Google Drive Node ("Check Existing Attachments") configured to search target Drive folder with naming format:  
      `{{receivedDate}} - {{senderEmail}} - {{filename}}`  
    - Add If Node ("Check if File Exists") to detect duplicates.  
    - Add Merge Node ("Merge Attachment Data") to select attachments to upload.  
    - Add Google Drive Node ("Upload to Google Drive") with same file naming and folder ID.  
    - Connect all nodes in order from Check for Attachments branch.

16. **For Urgent Alert Branch:**  
    - Add Telegram Node ("Send Urgent Alert")  
    - Configure chat ID and message template including sender, subject, summary, and Gmail link.  
    - Connect from "Smart Action Router" ALERT output.

17. **For Label Assignment Branch:**  
    - Add Aggregate Node ("Aggregate Label IDs") to collect label IDs from classification output.  
    - Connect from "Smart Action Router" LABEL output.

18. **Add Gmail Node ("Assign Classification Label"):**  
    - Operation: Add Labels  
    - Label IDs from Aggregate Label IDs output  
    - Message ID from Email_Trigger  
    - Connect from Aggregate Label IDs.

19. **Add Gmail Node ("Mark as Processed"):**  
    - Operation: Add Labels  
    - Add predefined "Processed" label ID  
    - Message ID from Email_Trigger  
    - Set "Continue on Error" to true  
    - Connect from Aggregate Label IDs.

20. **Connect "Mark as Processed" output back to "Process One by One" to enable batch continuation.**

21. **Configure Credentials:**  
    - Gmail OAuth2 with required scopes (read, modify, send drafts)  
    - Google Drive OAuth2 with access to target folders  
    - OpenAI API key for GPT-4  
    - Telegram Bot API token and chat ID for alerts

22. **Test workflow with sample emails ensuring label IDs and folder IDs are configured in nodes.**

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                        |
|----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| For file filtering, switch between `GetSpecificBinaryData` and `GetBinaryDataALL` depending on security needs.             | See sticky note "📋 Optional File Filtering" for details and guide: https://docs.n8n.io/workflows/sticky-notes/ |
| Email processing pipeline focuses on thread context preservation and batch processing for accuracy.                        | See sticky note "📧 Email Processing" for detailed flow.              |
| AI response generation uses a professional, warm tone matching the original email and includes alternative responses.      | See sticky note "✍️ Response Generation".                            |
| Label management is automated by syncing only user-created Gmail labels to maintain flexible classification categories.    | See sticky note "🔄 Label Sync".                                      |
| Attachment handling includes duplicate detection in Google Drive and structured naming for easy retrieval.                 | See sticky note "☁️ File Management" and "📎 Attachment Handler".    |
| Urgent notifications are sent via Telegram with direct Gmail links for quick access.                                        | See sticky note "🚨 Alert System".                                   |
| AI Email Classification prompt includes strict rules to avoid hallucinations and enforces JSON output schema compliance.   | See detailed prompt in AI Email Classifier node configuration.       |
| For more on n8n sticky notes usage and editing, see official docs: https://docs.n8n.io/workflows/sticky-notes/              | -                                                                    |

---

This completes the detailed documentation and reconstruction guide for the **Smart Email Manager with Gmail, GPT-4 Classification & Auto-Responses** workflow.