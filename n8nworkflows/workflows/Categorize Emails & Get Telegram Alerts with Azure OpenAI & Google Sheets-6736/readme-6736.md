Categorize Emails & Get Telegram Alerts with Azure OpenAI & Google Sheets

https://n8nworkflows.xyz/workflows/categorize-emails---get-telegram-alerts-with-azure-openai---google-sheets-6736


# Categorize Emails & Get Telegram Alerts with Azure OpenAI & Google Sheets

### 1. Workflow Overview

This workflow is a personalized email assistant designed to automatically categorize incoming emails, log relevant details in Google Sheets, and send Telegram notifications based on email categories. It also provides a Telegram chatbot interface for querying and updating the status of emails using AI-powered natural language processing via Azure OpenAI.

The workflow is logically split into two main parts:

- **1.1 Incoming Email Processing:**  
  Triggered by new unseen emails via IMAP, this block extracts email details, verifies data completeness, classifies the email content into predefined categories using Azure OpenAI, sends Telegram alerts accordingly, and appends email data to Google Sheets.

- **1.2 Telegram Chatbot Interaction and AI Agent:**  
  Triggered by Telegram messages, this block logs user chat IDs, fetches "NOT CHECKED" emails from Google Sheets, aggregates and formats the data, and uses an AI agent to answer user queries or update email status in the sheet. Replies are sent back to the user on Telegram.

Supporting components include memory management for AI context, status update tools, and multiple Telegram message nodes for category-specific notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Incoming Email Processing

- **Overview:**  
  Handles new emails detected via IMAP, extracts key info, checks data integrity, classifies emails into categories, sends Telegram alerts, and logs email details into Google Sheets.

- **Nodes Involved:**  
  - Email Trigger (IMAP)  
  - Get User ID  
  - Mail Information  
  - If Null  
  - Azure OpenAI Classifier  
  - E-Mail Text Classifier  
  - Send Security Message  
  - Send Personal Message  
  - Send Reply Message  
  - Like im young again (NoOp)  
  - Nothing will change (NoOp)  
  - Append Mail Details

- **Node Details:**  

  - **Email Trigger (IMAP)**  
    - Type: Email Read IMAP Trigger  
    - Role: Triggers workflow on new unseen emails  
    - Config: Filters for unseen emails only  
    - Input: Incoming emails via configured IMAP credential  
    - Output: Email data JSON with fields like from, subject, date, to  
    - Possible Failures: IMAP authentication errors, network timeouts  

  - **Get User ID**  
    - Type: Google Sheets Read  
    - Role: Retrieves Telegram chat IDs from a Google Sheet to know where to send notifications  
    - Config: Reads from "user_ids_mail_assistant" spreadsheet, Sheet1 (gid=0)  
    - Input: Triggered by Email Trigger output  
    - Output: List of stored Telegram IDs  
    - Failures: Google Sheets API quota limits, credential errors  

  - **Mail Information**  
    - Type: Set Node  
    - Role: Extracts and maps email fields (Sender, subject, date, receiver_mail) from the email JSON into named variables for downstream use  
    - Input: Output of Get User ID  
    - Output: Structured email info JSON  

  - **If Null**  
    - Type: If Condition  
    - Role: Validates that email fields (Sender, subject, date, receiver_mail) are non-empty  
    - Input: Mail Information output  
    - Output: Routes to NoOp (Nothing) if any field is missing, or to classification if valid  

  - **Azure OpenAI Classifier**  
    - Type: Azure OpenAI Chat Model (gpt-4o-mini)  
    - Role: Provides AI-powered classification assistance feeding into the text classifier node  
    - Input: Email details text  
    - Output: Classification results to E-Mail Text Classifier  

  - **E-Mail Text Classifier**  
    - Type: Langchain Text Classifier  
    - Role: Categorizes the email into one of five categories: Security, Personal, Updates, Advertisement, Unimportant  
    - Config: Uses subject, date, sender as input text  
    - Output: Routes flow according to category  

  - **Send Security Message / Send Personal Message / Send Reply Message**  
    - Type: Telegram Message  
    - Role: Send category-specific alerts to the user’s Telegram chat ID, dynamically inserting the email subject  
    - Config: Uses chat ID from Get User ID node  
    - Output: Passes to Append Mail Details node for logging  

  - **Like im young again / Nothing will change**  
    - Type: NoOp  
    - Role: Placeholder nodes for Advertisement and Unimportant categories (no notification or logging)  

  - **Append Mail Details**  
    - Type: Google Sheets Append  
    - Role: Logs email subject and status ("NOT CHECKED") into the "mail_data" Google Sheet for record keeping and AI agent querying  
    - Config: Appends rows to Sheet1 (gid=0) of the specified Google Sheet document  

#### 2.2 Telegram Chatbot Interaction and AI Agent

- **Overview:**  
  Manages user interactions via Telegram bot messages, stores user chat IDs, retrieves pending emails, formats data, invokes AI agent to answer or update email statuses, and sends replies.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Append User ID  
  - Get E-Mails  
  - Aggregate  
  - Set Rows as Data  
  - AI Agent  
  - Reply To User  
  - Memory  
  - Update Status  
  - Azure OpenAI Agent

- **Node Details:**  

  - **Telegram Trigger**  
    - Type: Telegram Trigger (Webhook)  
    - Role: Starts this part of the workflow when a message is received by the bot  
    - Config: Listens to all messages, can be restricted by chat ID if desired  
    - Output: Telegram message JSON with chat and message details  

  - **Append User ID**  
    - Type: Google Sheets AppendOrUpdate  
    - Role: Stores the Telegram user's chat ID in the "user_ids_mail_assistant" Google Sheet to allow sending notifications later  
    - Input: Telegram Trigger output  
    - Output: No direct downstream connections (side-effect only)  
    - Edge Cases: Duplicate chat IDs handled by appendOrUpdate  

  - **Get E-Mails**  
    - Type: Google Sheets Read  
    - Role: Retrieves emails from "mail_data" Google Sheet where Status = "NOT CHECKED" to provide pending email data to AI agent  
    - Input: Telegram Trigger output  
    - Output: Array of rows matching the filter  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all retrieved email rows into a single item to simplify handling  
    - Input: Get E-Mails output  
    - Output: Single combined rows array  

  - **Set Rows as Data**  
    - Type: Set  
    - Role: Converts the combined rows array into a stringified format to prevent JSON "[object Object]" issues in the AI agent input  
    - Output: Prepares data for AI processing  

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Processes user messages and email data, can update email statuses via the "Update Status" tool, and generates responses  
    - Config: Includes system prompt defining role as helpful email assistant, with access to "Update Status" tool  
    - Input: User message text and email data  
    - Output: AI-generated reply text to Reply To User node  
    - Edge Cases: Requires correct session key (Telegram chat ID) for memory, possible AI model timeouts or errors  

  - **Reply To User**  
    - Type: Telegram Message  
    - Role: Sends AI agent's response back to the user on Telegram  
    - Config: Uses chat ID from Telegram Trigger message  

  - **Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains conversational context limited to last 10 messages per user session to optimize token usage  
    - Config: Session key based on Telegram chat ID  

  - **Update Status**  
    - Type: Google Sheets Update via Google Sheets Tool  
    - Role: Updates the "Status" column for a specific email row in Google Sheets based on AI agent commands  
    - Input: Values extracted from AI agent tool call (row number, status)  
    - Output: Feeds back to AI Agent as tool result  

  - **Azure OpenAI Agent**  
    - Type: Azure OpenAI Chat Model (gpt-4o-mini)  
    - Role: Provides the language model backend for the AI Agent node  
    - Input/Output: Connected to AI Agent node for prompt processing  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                                                               |
|-------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Email Trigger (IMAP)     | Email Read IMAP                  | Trigger workflow on new emails         | -                            | Get User ID                         |                                                                                                                                           |
| Get User ID              | Google Sheets                   | Retrieve Telegram chat IDs              | Email Trigger (IMAP)          | Mail Information                    |                                                                                                                                           |
| Mail Information         | Set                            | Extract email fields                    | Get User ID                  | If Null                            |                                                                                                                                           |
| If Null                 | If                             | Check for empty critical email fields  | Mail Information             | Nothing / E-Mail Text Classifier    |                                                                                                                                           |
| Nothing                 | NoOp                           | Placeholder for invalid email data     | If Null                      | -                                   |                                                                                                                                           |
| Azure OpenAI Classifier  | Azure OpenAI Chat Model         | AI model assist for classification     | -                            | E-Mail Text Classifier             |                                                                                                                                           |
| E-Mail Text Classifier   | Langchain Text Classifier       | Categorize email into 5 categories     | Azure OpenAI Classifier      | Send Security/Personal/Reply etc.  |                                                                                                                                           |
| Send Security Message    | Telegram Message                | Notify user about security emails      | E-Mail Text Classifier       | Append Mail Details                |                                                                                                                                           |
| Send Personal Message    | Telegram Message                | Notify user about personal emails      | E-Mail Text Classifier       | Append Mail Details                |                                                                                                                                           |
| Send Reply Message       | Telegram Message                | Notify user about update emails        | E-Mail Text Classifier       | Append Mail Details                |                                                                                                                                           |
| Like im young again      | NoOp                           | Placeholder for Advertisement category | E-Mail Text Classifier       | -                                   |                                                                                                                                           |
| Nothing will change      | NoOp                           | Placeholder for Unimportant category    | E-Mail Text Classifier       | -                                   |                                                                                                                                           |
| Append Mail Details      | Google Sheets Append           | Log email data with status              | Send Security/Personal/Reply | -                                   |                                                                                                                                           |
| Telegram Trigger         | Telegram Trigger (Webhook)     | Start chatbot interaction block        | -                            | Get E-Mails, Append User ID        |                                                                                                                                           |
| Append User ID           | Google Sheets AppendOrUpdate   | Store Telegram user chat ID             | Telegram Trigger             | -                                   |                                                                                                                                           |
| Get E-Mails              | Google Sheets Read             | Fetch emails with status NOT CHECKED   | Telegram Trigger             | Aggregate                         |                                                                                                                                           |
| Aggregate                | Aggregate                     | Combine multiple rows into one          | Get E-Mails                  | Set Rows as Data                   |                                                                                                                                           |
| Set Rows as Data         | Set                           | Format combined rows for AI input       | Aggregate                    | AI Agent                         |                                                                                                                                           |
| AI Agent                 | Langchain Agent Node           | Process user queries, update email status | Set Rows as Data             | Reply To User, Update Status       |                                                                                                                                           |
| Reply To User            | Telegram Message              | Send AI-generated reply to Telegram     | AI Agent                    | -                                   |                                                                                                                                           |
| Memory                   | Langchain Memory Buffer Window | Maintain conversation context           | AI Agent                    | AI Agent                         |                                                                                                                                           |
| Update Status            | Google Sheets Tool Update     | Update email status in Google Sheets    | AI Agent (tool call)         | AI Agent (tool result)             |                                                                                                                                           |
| Azure OpenAI Agent       | Azure OpenAI Chat Model       | AI language model backend for AI Agent  | AI Agent                    | AI Agent                         |                                                                                                                                           |
| Sticky Note              | Sticky Note                   | Workflow title                          | -                            | -                                   | # E-Mail Assistant                                                                                                                        |
| Sticky Note1             | Sticky Note                   | Credential setup instructions           | -                            | -                                   | See detailed credential setup instructions with links for IMAP, Google Sheets OAuth2, Azure OpenAI, and Telegram Bot setup.                |
| Sticky Note2             | Sticky Note                   | Workflow info and categories explanation | -                            | -                                   | Explains categories, usage notes, and reminders about sheet names and chat ID management.                                                  |
| Sticky Note3             | Sticky Note                   | Detailed workflow description           | -                            | -                                   | Explains workflow logic in two parts with node-by-node explanations and usage recommendations.                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - IMAP Email: Configure IMAP credential with your email provider (e.g., Gmail with app password).
   - Google Sheets OAuth2: Set up OAuth2 credentials with Google Cloud Console for Sheets API access.
   - Azure OpenAI API: Create Azure OpenAI resource and generate API key.
   - Telegram API: Create a Telegram Bot via @BotFather and use the token to create Telegram credential in n8n.

2. **Incoming Email Processing Block:**

   1. Add **Email Trigger (IMAP)** node:
      - Set to trigger on unseen emails only.
      - Attach IMAP credential.

   2. Add **Get User ID** (Google Sheets) node:
      - Read from spreadsheet "user_ids_mail_assistant", Sheet1 (gid=0).
      - Attach Google Sheets OAuth2 credential.

   3. Connect Email Trigger → Get User ID.

   4. Add **Mail Information** (Set) node:
      - Map: Sender = `{{$json.from}}`, subject = `{{$json.subject}}`, date = `{{$json.date}}`, receiver_mail = `{{$json.to}}`.
      - Connect Get User ID → Mail Information.

   5. Add **If Null** node:
      - Condition checks if any of Sender, subject, date, receiver_mail is empty.
      - Connect Mail Information → If Null.

   6. Add **Nothing** (NoOp) node for null case.
      - Connect If Null (true) → Nothing.

   7. Add **Azure OpenAI Classifier** node:
      - Model: gpt-4o-mini.
      - Input text: email subject, date, sender.
      - Attach Azure OpenAI credential.
      - Connect If Null (false) → Azure OpenAI Classifier.

   8. Add **E-Mail Text Classifier** node:
      - InputText: `{{$json.subject}}\n\n{{$json.date}}\n{{$json.Sender}}`
      - Categories: Security, Personal, Updates, Advertisement, Unimportant with descriptions.
      - Connect Azure OpenAI Classifier → E-Mail Text Classifier.

   9. Add Telegram message nodes for Security, Personal, Updates:
      - **Send Security Message**: Text includes "Security Alert!" and email subject.
      - **Send Personal Message**: Text includes "Personal Message!" and email subject.
      - **Send Reply Message**: Text includes "You have an update!" and email subject.
      - Use chatId from Get User ID node.
      - Connect E-Mail Text Classifier → each appropriate Telegram message node.

   10. Add **Like im young again** and **Nothing will change** (NoOp) nodes for Advertisement and Unimportant categories.
       - Connect E-Mail Text Classifier → respective NoOp nodes.

   11. Add **Append Mail Details** Google Sheets Append node:
       - Append new rows to "mail_data" Google Sheet, Sheet1 (gid=0).
       - Columns: "E-Mail Subject" = email subject, "Status" = "NOT CHECKED".
       - Connect all Telegram message nodes and NoOp nodes → Append Mail Details.

3. **Telegram Chatbot Interaction Block:**

   1. Add **Telegram Trigger** node:
      - Listen to all messages.
      - Attach Telegram credential.

   2. Add **Append User ID** Google Sheets AppendOrUpdate node:
      - Append or update "telegram_id" column with `{{$json.message.chat.id}}`.
      - Use "user_ids_mail_assistant" Google Sheet, Sheet1 (gid=0).
      - Connect Telegram Trigger → Append User ID.

   3. Add **Get E-Mails** Google Sheets Read node:
      - Read rows where Status = "NOT CHECKED" from "mail_data" Google Sheet, Sheet1 (gid=0).
      - Connect Telegram Trigger → Get E-Mails.

   4. Add **Aggregate** node:
      - Aggregate all items into a single array named "combinedRows".
      - Connect Get E-Mails → Aggregate.

   5. Add **Set Rows as Data** node:
      - Set new field "data" as stringified `{{$json.combinedRows}}`.
      - Connect Aggregate → Set Rows as Data.

   6. Add **AI Agent** node:
      - Use Langchain Agent node.
      - System message: Defines helpful email assistant role, instructs to use "Update Status" tool.
      - Input text: User message text + email data.
      - Connect Set Rows as Data → AI Agent.
      - Attach Azure OpenAI Agent node as language model.
      - Attach Memory node for session context:
        - Session key: Telegram chat ID.
        - Context window length: 10.

   7. Add **Update Status** Google Sheets Tool Update node:
      - Update the "Status" column for a given row number in "mail_data" Google Sheet.
      - Connect AI Agent → Update Status (as AI tool).
      - Attach Google Sheets credentials.

   8. Add **Reply To User** Telegram message node:
      - Sends back AI Agent output text.
      - Chat ID from Telegram Trigger message.
      - Connect AI Agent → Reply To User.

4. **Add Sticky Notes:**
   - Add sticky notes with credential setup guides, workflow description, and category explanations as per original workflow content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| IMAP Email Trigger setup requires generating an app password and enabling 2-step verification in Gmail. https://docs.n8n.io/integrations/builtin/credentials/imap/gmail/#enable-2-step-verification                             | Credential setup for IMAP email                                                                                                         |
| Google Sheets OAuth2 requires creating a Google Cloud project, enabling APIs, configuring OAuth consent, and generating credentials. https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/              | Credential setup for Google Sheets API                                                                                                  |
| Azure OpenAI usage: model `gpt-4o-mini` is used; alternatively OpenAI, OpenRouter, Google Gemini can be integrated.                                                                                                             | AI model credential and configuration                                                                                                  |
| Telegram Bot creation via @BotFather, generate token, add credential in n8n. https://docs.n8n.io/integrations/builtin/credentials/telegram                                                                                       | Telegram Bot credential setup                                                                                                           |
| This workflow categorizes emails into five categories: Security, Personal, Updates, Advertisement, Unimportant. Notifications are sent for first three categories only.                                                          | Workflow behavior and customization                                                                                                    |
| Telegram messages are used both for user notifications on incoming emails and for chatbot queries to update email statuses and get summaries.                                                                                   | Workflow interaction details                                                                                                           |
| Sheet names and IDs must match those configured in the Google Sheets nodes to avoid errors.                                                                                                                                     | Important configuration note                                                                                                           |
| The AI agent uses conversation memory limited to 10 exchanges to control token usage. This can be adjusted for longer context.                                                                                                  | AI agent configuration                                                                                                                 |
| Telegram Trigger can be restricted to specific chat IDs to prevent unauthorized use.                                                                                                                                             | Security recommendation                                                                                                                |

---

**disclaimer** The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.