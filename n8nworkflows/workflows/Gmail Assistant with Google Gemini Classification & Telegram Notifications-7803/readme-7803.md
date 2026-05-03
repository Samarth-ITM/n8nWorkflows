Gmail Assistant with Google Gemini Classification & Telegram Notifications

https://n8nworkflows.xyz/workflows/gmail-assistant-with-google-gemini-classification---telegram-notifications-7803


# Gmail Assistant with Google Gemini Classification & Telegram Notifications

### 1. Workflow Overview

This workflow implements a **Smart Email Assistant** that automates Gmail email processing using AI classification and integrates with Telegram for notifications and human approvals. It targets users who want real-time intelligent handling of incoming emails, automated professional replies with approval, and daily summarized email reports. Key logical blocks include:

- **1.1 Real-Time Email Processing:** Triggered on new Gmail messages; filters emails in the inbox, classifies them via AI into three categories (Reply Needed, Important Notification, Spam/Unimportant), and routes accordingly.
- **1.2 Automated Reply Generation with Approval:** For emails requiring replies, AI generates a professional response, sends it to Telegram for user approval, and upon approval, sends the reply via Gmail.
- **1.3 Notification and Spam Handling:** Important notifications are summarized briefly by AI and sent as Telegram messages; spam or unimportant emails are ignored.
- **1.4 Daily Email Summary Report:** Every day at 8 AM, the workflow fetches emails from the past 24 hours, aggregates and summarizes them using AI, and sends a comprehensive report via Telegram.
- **1.5 AI Model & Output Parsing Infrastructure:** Uses Google Gemini AI models for classification, summarization, and reply generation, with structured JSON output parsing to ensure reliable data handling.
- **1.6 Workflow Control and Exit Nodes:** Multiple no-op nodes cleanly exit the workflow in cases like non-inbox emails, unapproved replies, or unimportant emails to prevent errors or unnecessary processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Real-Time Email Processing

- **Overview:**  
Monitors new Gmail emails in real-time, filters to inbox emails, classifies each email into one of three categories using AI, and routes the email for further processing based on classification.

- **Nodes Involved:**  
  - Real-time Email Trigger  
  - Check: Is Email in Inbox? (If node)  
  - AI Email Classifier (AI agent)  
  - Structured Output Parser1 (AI output parser)  
  - Google Gemini Chat Model4 (AI model)  
  - Route by Category (Switch)  
  - Exit: Not in Inbox (NoOp)  

- **Node Details:**

  - **Real-time Email Trigger**  
    - *Type:* Gmail Trigger  
    - *Role:* Watches Gmail inbox for new email messages, polling every minute.  
    - *Configuration:* No filters applied on trigger; all new emails are captured.  
    - *Input/Output:* Outputs new Gmail email data.  
    - *Failure Modes:* Gmail API auth errors, trigger polling delays.

  - **Check: Is Email in Inbox?**  
    - *Type:* If  
    - *Role:* Filters emails to only those labeled "INBOX".  
    - *Configuration:* Checks if the first label id contains "INBOX".  
    - *Input:* Output from Gmail Trigger.  
    - *Output:* Routes emails labeled INBOX to classifier; others exit workflow.  
    - *Failure Modes:* Label data missing or malformed JSON.

  - **AI Email Classifier**  
    - *Type:* Langchain AI Agent  
    - *Role:* Classifies email into category 0 (Reply Needed), 1 (Important Notification), or 2 (Spam/Unimportant).  
    - *Configuration:* Uses Google Gemini model (via linked Google Gemini Chat Model4) with a detailed system prompt for classification logic.  
    - *Input:* Email metadata (From, Subject, snippet, Date) from inbox emails.  
    - *Output:* JSON with `category`, `confidence`, and `reason`.  
    - *Failure Modes:* AI model timeout, parsing errors, classification ambiguity.

  - **Structured Output Parser1**  
    - *Type:* Langchain Structured Output Parser  
    - *Role:* Parses the AI classification JSON output into structured fields for workflow use.  
    - *Input:* AI Email Classifier raw output.  
    - *Output:* Parsed JSON object with category data.  
    - *Failure Modes:* Invalid AI output format, JSON parsing errors.

  - **Google Gemini Chat Model4**  
    - *Type:* Langchain AI Model (Google Gemini)  
    - *Role:* Provides AI language model backend for AI Email Classifier.  
    - *Input:* Classification prompt text.  
    - *Output:* AI-generated classification response.  
    - *Failure Modes:* Google API quota limits, network errors.

  - **Route by Category**  
    - *Type:* Switch  
    - *Role:* Routes emails based on category: 0 → reply generation, 1 → summary creation, 2 → exit as not important.  
    - *Input:* Parsed classification category.  
    - *Output:* Branches to next steps accordingly.  
    - *Failure Modes:* Category value missing or invalid.

  - **Exit: Not in Inbox**  
    - *Type:* NoOp  
    - *Role:* Terminates workflow processing cleanly for emails outside inbox.  
    - *Input:* Emails filtered out by Inbox check.  
    - *Output:* None.  
    - *Failure Modes:* None.

---

#### 2.2 Automated Reply Generation with Approval

- **Overview:**  
For emails classified as requiring replies, the workflow generates a professional reply using AI, sends the reply content to Telegram for human approval, and upon approval, sends the reply via Gmail.

- **Nodes Involved:**  
  - AI: Generate Email Reply (Langchain Agent)  
  - Google Gemini Chat Model1 (AI Model)  
  - Structured Output Parser (AI Output Parser)  
  - Telegram: Send + Approve  
  - Check: Telegram Approved? (If)  
  - Send Email Reply (Gmail)  
  - Exit: Not Approved (NoOp)  

- **Node Details:**

  - **AI: Generate Email Reply**  
    - *Type:* Langchain Agent  
    - *Role:* Generates a professional email reply based on the incoming email content.  
    - *Configuration:* Uses a system prompt instructing to produce JSON with email, subject, and body fields; suppresses unnecessary greetings/sign-offs; outputs empty JSON if no reply needed.  
    - *Input:* Email details (From, Subject, snippet, current date).  
    - *Output:* JSON with reply fields.  
    - *Failure Modes:* AI generation errors, output formatting issues.

  - **Google Gemini Chat Model1**  
    - *Type:* Langchain AI Model (Google Gemini)  
    - *Role:* AI backend for reply generation node.  
    - *Input:* Prompt text from AI: Generate Email Reply node.  
    - *Output:* AI-generated email reply JSON.  
    - *Failure Modes:* API limits, latency.

  - **Structured Output Parser**  
    - *Type:* Langchain Structured Output Parser  
    - *Role:* Parses AI reply JSON to structured fields.  
    - *Input:* AI reply raw text.  
    - *Output:* Parsed email, subject, and body fields.  
    - *Failure Modes:* Parsing errors from invalid JSON.

  - **Telegram: Send + Approve**  
    - *Type:* Telegram node  
    - *Role:* Sends message to Telegram chat containing original email info and AI reply, waits for user approval (double confirmation).  
    - *Configuration:* Sends formatted message with email details and AI reply; waits up to 5 minutes for approval; requires double approval.  
    - *Input:* Parsed AI reply and original email details.  
    - *Output:* Approval status in Telegram response.  
    - *Failure Modes:* Telegram API errors, approval timeout, user not responding.

  - **Check: Telegram Approved?**  
    - *Type:* If  
    - *Role:* Checks if Telegram approval response is `true`.  
    - *Input:* Approval data from Telegram node.  
    - *Output:* If approved, proceed to send email reply; else exit workflow.  
    - *Failure Modes:* Missing approval data.

  - **Send Email Reply**  
    - *Type:* Gmail  
    - *Role:* Sends the AI-generated and approved reply as a Gmail reply to the original email.  
    - *Configuration:* Uses Gmail OAuth2 credentials; replies to the original message by messageId; sends plain text email.  
    - *Input:* Reply body from AI output, original email message ID.  
    - *Output:* Confirmation of sent email.  
    - *Failure Modes:* Gmail API errors, permission denied.

  - **Exit: Not Approved**  
    - *Type:* NoOp  
    - *Role:* Terminates workflow if Telegram approval denied or timed out.  
    - *Input:* Telegram approval false.  
    - *Output:* None.  
    - *Failure Modes:* None.

---

#### 2.3 Notification and Spam Handling

- **Overview:**  
Emails classified as important notifications (category 1) are summarized by AI into short Telegram-style messages and sent to Telegram. Emails classified as spam or unimportant (category 2) are silently ignored.

- **Nodes Involved:**  
  - Summarize Email with AI (Langchain Agent)  
  - Google Gemini Chat Model2 (AI Model)  
  - Send Summary to Telegram (Telegram)  
  - Exit: Not Important (NoOp)  

- **Node Details:**

  - **Summarize Email with AI**  
    - *Type:* Langchain Agent  
    - *Role:* Creates a concise, emoji-enhanced summary of the email content for Telegram notification.  
    - *Configuration:* System prompt instructs to generate summaries max 200 characters, highlight key info, use light emojis, and write in plain English.  
    - *Input:* Email sender, subject, and snippet text.  
    - *Output:* Short summary text.  
    - *Failure Modes:* AI generation or formatting issues.

  - **Google Gemini Chat Model2**  
    - *Type:* Langchain AI Model (Google Gemini)  
    - *Role:* AI backend for email summarization.  
    - *Input:* Summarization prompt text.  
    - *Output:* Generated summary text.  
    - *Failure Modes:* API latency or quota issues.

  - **Send Summary to Telegram**  
    - *Type:* Telegram  
    - *Role:* Sends the AI-generated concise summary message to the configured Telegram chat.  
    - *Input:* Summary text from AI.  
    - *Output:* Confirmation of message sent.  
    - *Failure Modes:* Telegram API connectivity.

  - **Exit: Not Important**  
    - *Type:* NoOp  
    - *Role:* Ends workflow processing for spam or unimportant emails.  
    - *Input:* Emails classified as category 2.  
    - *Output:* None.  
    - *Failure Modes:* None.

---

#### 2.4 Daily Email Summary Report

- **Overview:**  
Every day at 8 AM, the workflow fetches all Gmail emails from the past 24 hours, aggregates key fields, creates a comprehensive AI-generated summary report, and sends the report to Telegram.

- **Nodes Involved:**  
  - Daily 8AM Trigger (Schedule Trigger)  
  - Fetch Emails - Past 24 Hours1 (Gmail)  
  - Organize Email Data (Aggregate)  
  - Email Summarizer (Langchain Agent)  
  - Google Gemini Chat Model3 (AI Model)  
  - Send Daily Report (Telegram)  

- **Node Details:**

  - **Daily 8AM Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Triggers workflow once daily at 8 AM.  
    - *Input:* None (time-based trigger).  
    - *Output:* Trigger event to start fetching emails.  
    - *Failure Modes:* Scheduler misconfiguration.

  - **Fetch Emails - Past 24 Hours1**  
    - *Type:* Gmail  
    - *Role:* Retrieves all Gmail emails from last 24 hours using Gmail search query.  
    - *Configuration:* Query `newer_than:1d`; returns all matching emails.  
    - *Input:* Trigger event.  
    - *Output:* List of emails with fields (id, From, Subject, snippet, text).  
    - *Failure Modes:* Gmail API limits, auth errors.

  - **Organize Email Data**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all email data fields into a single JSON array for summarization.  
    - *Configuration:* Includes fields: id, From, Subject, snippet, text.  
    - *Input:* Email list from fetch node.  
    - *Output:* Aggregated JSON for AI input.  
    - *Failure Modes:* Empty input, malformed data.

  - **Email Summarizer**  
    - *Type:* Langchain Agent  
    - *Role:* Creates a detailed, structured summary report of all emails including summaries, issues, action items, and follow-ups formatted for Telegram.  
    - *Configuration:* System prompt defines complex formatting rules, priority indicators, and output structure.  
    - *Input:* Aggregated email data JSON string.  
    - *Output:* Formatted summary text.  
    - *Failure Modes:* Large input causing AI timeout, formatting errors.

  - **Google Gemini Chat Model3**  
    - *Type:* Langchain AI Model (Google Gemini)  
    - *Role:* AI backend for daily email summarization.  
    - *Input:* Summary prompt text.  
    - *Output:* Generated summary report text.  
    - *Failure Modes:* API quota limits.

  - **Send Daily Report**  
    - *Type:* Telegram  
    - *Role:* Sends the comprehensive daily email report to the Telegram chat.  
    - *Input:* Text summary from Email Summarizer node.  
    - *Output:* Confirmation of message sent.  
    - *Failure Modes:* Telegram connectivity issues.

---

#### 2.5 AI Model & Output Parsing Infrastructure

- **Overview:**  
Provides AI model nodes for various tasks (classification, reply generation, summarization) and output parser nodes to convert AI JSON text outputs into structured data for workflow logic.

- **Nodes Involved:**  
  - Google Gemini Chat Model1, 2, 3, 4 (AI Models)  
  - Structured Output Parser, Structured Output Parser1 (Output Parsers)  

- **Node Details:**

  - **Google Gemini Chat Model Nodes**  
    - *Type:* Langchain AI Model (Google Gemini)  
    - *Role:* Provide Google Palm API based AI language model capabilities for classification, summarization, and reply generation.  
    - *Input:* Prompt text from respective Langchain Agent nodes.  
    - *Output:* AI-generated text responses.  
    - *Failure Modes:* API limits, network latency.

  - **Structured Output Parsers**  
    - *Type:* Langchain Structured Output Parser  
    - *Role:* Parse raw AI JSON text output into structured JSON fields accessible by workflow nodes.  
    - *Input:* Raw AI text response.  
    - *Output:* Parsed JSON object with defined schema.  
    - *Failure Modes:* Invalid format, parsing failures.

---

#### 2.6 Workflow Control and Exit Nodes

- **Overview:**  
No Operation (NoOp) nodes serve as clean exit points to halt workflow processing for emails outside inbox, unapproved replies, and unimportant emails, preventing errors or unnecessary execution.

- **Nodes Involved:**  
  - Exit: Not in Inbox  
  - Exit: Not Approved  
  - Exit: Not Important  

- **Node Details:**

  - Each NoOp node simply receives input and ends execution without side effects.  
  - Used strategically for workflow branching and error avoidance.  
  - No configuration or credentials needed.  
  - No failure modes.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                                           | Input Node(s)                            | Output Node(s)                       | Sticky Note                                                                                 |
|----------------------------|---------------------------------------|-----------------------------------------------------------|----------------------------------------|------------------------------------|---------------------------------------------------------------------------------------------|
| Real-time Email Trigger     | Gmail Trigger                         | Triggers on new Gmail emails                              | -                                      | Check: Is Email in Inbox?           | # Smart Email Assistant with AI Classification and Telegram Integration overview           |
| Check: Is Email in Inbox?   | If                                   | Filters emails to inbox only                              | Real-time Email Trigger                 | AI Email Classifier, Exit: Not in Inbox |                                                                                             |
| Exit: Not in Inbox          | NoOp                                 | Ends processing for non-inbox emails                      | Check: Is Email in Inbox?               | -                                  |                                                                                             |
| AI Email Classifier         | Langchain Agent                      | Classifies email into 3 categories                        | Check: Is Email in Inbox?               | Route by Category                   |                                                                                             |
| Structured Output Parser1   | Langchain Output Parser              | Parses classification JSON output                         | AI Email Classifier                     | Route by Category                   |                                                                                             |
| Google Gemini Chat Model4   | Langchain AI Model                  | AI backend for classification                            | AI Email Classifier                     | AI Email Classifier                 |                                                                                             |
| Route by Category           | Switch                              | Routes emails based on category                           | Structured Output Parser1               | AI: Generate Email Reply, Summarize Email with AI, Exit: Not Important |                                                                                             |
| AI: Generate Email Reply    | Langchain Agent                     | Generates professional email reply                        | Route by Category (category 0)          | Telegram: Send + Approve            |                                                                                             |
| Google Gemini Chat Model1   | Langchain AI Model                  | AI backend for reply generation                           | AI: Generate Email Reply                | AI: Generate Email Reply            |                                                                                             |
| Structured Output Parser    | Langchain Output Parser             | Parses AI reply JSON output                               | AI: Generate Email Reply                | Telegram: Send + Approve            |                                                                                             |
| Telegram: Send + Approve    | Telegram                            | Sends AI reply to Telegram and waits for approval        | AI: Generate Email Reply                | Check: Telegram Approved?           |                                                                                             |
| Check: Telegram Approved?   | If                                 | Checks if Telegram approval is true                       | Telegram: Send + Approve                 | Send Email Reply, Exit: Not Approved |                                                                                             |
| Send Email Reply            | Gmail                              | Sends the approved email reply                            | Check: Telegram Approved?                | -                                  |                                                                                             |
| Exit: Not Approved          | NoOp                              | Ends workflow if reply not approved                       | Check: Telegram Approved?                | -                                  |                                                                                             |
| Summarize Email with AI     | Langchain Agent                     | Summarizes important emails for Telegram notifications   | Route by Category (category 1)           | Send Summary to Telegram            |                                                                                             |
| Google Gemini Chat Model2   | Langchain AI Model                  | AI backend for email summarization                        | Summarize Email with AI                 | Summarize Email with AI             |                                                                                             |
| Send Summary to Telegram    | Telegram                            | Sends summarized email info to Telegram                   | Summarize Email with AI                 | -                                  |                                                                                             |
| Exit: Not Important         | NoOp                              | Ends workflow for spam/unimportant emails                | Route by Category (category 2)           | -                                  |                                                                                             |
| Daily 8AM Trigger           | Schedule Trigger                   | Triggers daily email summary processing                   | -                                      | Fetch Emails - Past 24 Hours1       |                                                                                             |
| Fetch Emails - Past 24 Hours1| Gmail                            | Retrieves emails from past 24 hours                       | Daily 8AM Trigger                      | Organize Email Data                 |                                                                                             |
| Organize Email Data         | Aggregate                         | Aggregates email data for summarization                   | Fetch Emails - Past 24 Hours1            | Email Summarizer                   |                                                                                             |
| Email Summarizer            | Langchain Agent                   | Creates comprehensive daily email summary report          | Organize Email Data                    | Send Daily Report                  |                                                                                             |
| Google Gemini Chat Model3   | Langchain AI Model                | AI backend for daily email summarization                  | Email Summarizer                      | Email Summarizer                   |                                                                                             |
| Send Daily Report           | Telegram                         | Sends daily summary report to Telegram                    | Email Summarizer                      | -                                  |                                                                                             |
| Google Gemini Chat Model1   | Langchain AI Model                | AI backend for reply generation (see above)               | AI: Generate Email Reply                | AI: Generate Email Reply            |                                                                                             |
| Sticky Note                | Sticky Note                      | Workflow documentation and overview                       | -                                      | -                                  | # Smart Email Assistant with AI Classification and Telegram Integration … [full content]   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail OAuth2 Credential:**  
   - Set up OAuth2 credentials in n8n with Gmail API access for reading and sending emails.

2. **Create Telegram Bot API Credential:**  
   - Create Telegram bot, get bot token, add bot to your chat, and configure Telegram API credentials in n8n.

3. **Create Google Palm API Credential:**  
   - Configure Google Cloud project with Palm API access and add credentials to n8n.

4. **Add Node: Real-time Email Trigger**  
   - Type: Gmail Trigger  
   - Poll every minute (default)  
   - Credential: Gmail OAuth2  
   - No filters.

5. **Add Node: Check: Is Email in Inbox? (If node)**  
   - Condition: Check if first element of `$json.labels[0].id` contains "INBOX" (string contains)  
   - Connect Real-time Email Trigger > Check: Is Email in Inbox?  

6. **Add Node: Exit: Not in Inbox (NoOp)**  
   - Connect Check: Is Email in Inbox? (False) > Exit: Not in Inbox  

7. **Add Node: AI Email Classifier (Langchain Agent)**  
   - Input: email From, Subject, snippet, date from Gmail trigger  
   - Use system prompt for AI classification as per overview  
   - Credential: Google Palm API  
   - Connect Check: Is Email in Inbox? (True) > AI Email Classifier  

8. **Add Node: Structured Output Parser1**  
   - JSON schema: `{ "category": 0, "confidence": 85, "reason": "..." }`  
   - Connect AI Email Classifier > Structured Output Parser1  

9. **Add Node: Google Gemini Chat Model4**  
   - Credential: Google Palm API  
   - Connect AI Email Classifier (AI language model) > Google Gemini Chat Model4  

10. **Add Node: Route by Category (Switch)**  
    - Rules:  
      - Case 0: equals "0" → AI: Generate Email Reply  
      - Case 1: equals "1" → Summarize Email with AI  
      - Case 2: equals "2" → Exit: Not Important  
    - Connect Structured Output Parser1 > Route by Category  

11. **Add Node: AI: Generate Email Reply (Langchain Agent)**  
    - Input: email details with system prompt for professional reply generation  
    - Credential: Google Palm API  
    - Connect Route by Category (Case 0) > AI: Generate Email Reply  

12. **Add Node: Google Gemini Chat Model1**  
    - Credential: Google Palm API  
    - Connect AI: Generate Email Reply (AI language model) > Google Gemini Chat Model1  

13. **Add Node: Structured Output Parser**  
    - JSON schema sample: `{ "email": "example@email.com", "subject": "Re: Subject", "body": "Reply message" }`  
    - Connect AI: Generate Email Reply > Structured Output Parser  

14. **Add Node: Telegram: Send + Approve**  
    - Chat ID: your Telegram chat ID  
    - Message: formatted with original email and AI-generated reply details  
    - Operation: sendAndWait with 5-minute timeout, double approval required  
    - Credential: Telegram API  
    - Connect Structured Output Parser > Telegram: Send + Approve  

15. **Add Node: Check: Telegram Approved? (If node)**  
    - Condition: `$json.data.approved == true`  
    - Connect Telegram: Send + Approve > Check: Telegram Approved?  

16. **Add Node: Send Email Reply (Gmail)**  
    - Operation: reply to email  
    - Message body: AI reply body  
    - Message ID: original email message ID  
    - Credential: Gmail OAuth2  
    - Connect Check: Telegram Approved? (True) > Send Email Reply  

17. **Add Node: Exit: Not Approved (NoOp)**  
    - Connect Check: Telegram Approved? (False) > Exit: Not Approved  

18. **Add Node: Summarize Email with AI (Langchain Agent)**  
    - Input: email from, subject, snippet  
    - System prompt for short, emoji-enhanced Telegram summary  
    - Credential: Google Palm API  
    - Connect Route by Category (Case 1) > Summarize Email with AI  

19. **Add Node: Google Gemini Chat Model2**  
    - Credential: Google Palm API  
    - Connect Summarize Email with AI (AI language model) > Google Gemini Chat Model2  

20. **Add Node: Send Summary to Telegram**  
    - Chat ID: your Telegram chat ID  
    - Text: summary from AI  
    - Credential: Telegram API  
    - Connect Summarize Email with AI > Send Summary to Telegram  

21. **Add Node: Exit: Not Important (NoOp)**  
    - Connect Route by Category (Case 2) > Exit: Not Important  

22. **Add Node: Daily 8AM Trigger (Schedule Trigger)**  
    - Trigger at hour 8 daily  
    - Connect to Fetch Emails - Past 24 Hours1  

23. **Add Node: Fetch Emails - Past 24 Hours1 (Gmail)**  
    - Query: `newer_than:1d`  
    - Return all matching emails  
    - Credential: Gmail OAuth2  
    - Connect Daily 8AM Trigger > Fetch Emails - Past 24 Hours1  

24. **Add Node: Organize Email Data (Aggregate)**  
    - Aggregate all item data fields: id, From, Subject, snippet, text  
    - Connect Fetch Emails - Past 24 Hours1 > Organize Email Data  

25. **Add Node: Email Summarizer (Langchain Agent)**  
    - Input: aggregated email JSON string  
    - System prompt for detailed, formatted summary report for Telegram  
    - Credential: Google Palm API  
    - Connect Organize Email Data > Email Summarizer  

26. **Add Node: Google Gemini Chat Model3**  
    - Credential: Google Palm API  
    - Connect Email Summarizer (AI language model) > Google Gemini Chat Model3  

27. **Add Node: Send Daily Report (Telegram)**  
    - Chat ID: your Telegram chat ID  
    - Text: summary report from AI  
    - Credential: Telegram API  
    - Connect Email Summarizer > Send Daily Report  

28. **Add Sticky Note**  
    - Content: Full workflow overview and instructions (copy from Sticky Note node content)  
    - Position it for clarity in the editor.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| # Smart Email Assistant with AI Classification and Telegram Integration<br><br>## 📋 Overview<br>Intelligent email automation system that processes Gmail messages in real-time and provides daily summaries via Telegram notifications.<br><br>## 🔄 How It Works<br><br>### Real-Time Processing<br>1. 📧 Gmail Trigger → Monitors inbox for new emails<br>2. 🔍 Inbox Filter → Only processes emails in INBOX label<br>3. 🤖 AI Classifier → Categorizes emails into 3 types:<br>    Category 0: Needs Reply (questions, meetings, urgent)<br>   * Category 1: Important Notification (bills, confirmations)<br>   * Category 2: Spam/Unimportant (marketing, promotions)<br><br>### Automated Responses<br> Reply Branch: AI generates professional responses → Telegram approval → Sends reply<br> Notification Branch: Creates quick summaries → Sends to Telegram<br> Spam Branch: Ignores automatically<br><br>### Daily Summary (8 AM)<br> Fetches last 24 hours of emails<br> Creates comprehensive report with:<br>    Email summaries<br>   * Action items<br>   * Issues & concerns<br> Sends formatted report to Telegram<br><br>## ⚙️ Configuration Required<br><br>### Credentials Needed:<br> Gmail OAuth2: For reading/sending emails<br> Telegram Bot API: For notifications and approvals<br> Google Palm API: For Gemini AI processing<br><br>### Settings:<br> Telegram Chat ID: Your chat id (update to your chat)<br> Trigger Schedule: Every minute + Daily 8 AM<br> Timeout: 5 minutes for approvals<br><br>## 🎯 Key Features<br>✅ Smart AI classification (Google Gemini)<br>✅ Human approval for replies<br>✅ Automatic spam filtering<br>✅ Daily email digest<br>✅ Professional response generation<br>✅ Telegram integration for mobile access<br><br>## 📝 Notes<br> Uses Google Gemini models for all AI processing<br> Structured JSON outputs ensure reliability<br> Multiple exit points prevent workflow errors<br> Aggregates daily data for efficient processing | Sticky Note content in workflow editor            |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.