Automated Customer Feedback Triage with Gemini, Jotform, Google Sheets & Gmail Replies

https://n8nworkflows.xyz/workflows/automated-customer-feedback-triage-with-gemini--jotform--google-sheets---gmail-replies-9636


# Automated Customer Feedback Triage with Gemini, Jotform, Google Sheets & Gmail Replies

### 1. Workflow Overview

This workflow automates the triage and response process for customer feedback collected via a Jotform. It is designed to classify incoming feedback into categories (comments, questions, suggestions), analyze sentiment, and route or respond accordingly:

- **1.1 Input Reception:** Listens for new Jotform submissions containing customer feedback.
- **1.2 Feedback Classification:** Uses a Switch node to categorize feedback into comments, questions, or suggestions.
- **1.3 Sentiment Analysis & Logging for Comments:** Performs sentiment analysis on comments and logs them in a Google Sheet.
- **1.4 Question Handling:** Uses a LangChain Q&A agent backed by a Google Sheets knowledge base to generate friendly replies, which are emailed to the customer.
- **1.5 Suggestion Handling:** Summarizes suggestions, sends a Telegram alert, and logs suggestions in a Google Sheet.
- **1.6 Urgent Alerting:** If sentiment analysis detects angry feedback, sends an urgent Telegram notification to support.

The workflow integrates multiple services: Jotform, Google Sheets, Gmail, Telegram, and Google Gemini (Google Palm API) for AI-powered language processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures new customer feedback submissions from a configured Jotform.
- **Nodes Involved:** JotForm Trigger
- **Node Details:**
  - **JotForm Trigger**
    - Type: Trigger node specifically for Jotform.
    - Configuration: Linked to a specific Jotform form (ID configured via credentials).
    - Key variables: Feedback fields such as "Name", "E-mail", and "Describe Your Feedback:" are extracted.
    - Input: Incoming webhook from Jotform submission.
    - Output: Passes submission JSON to downstream nodes.
    - Failure modes: Webhook failures, API credential errors, invalid form mapping.
    - Version: 1

#### 2.2 Feedback Classification

- **Overview:** Routes feedback into categories "comments", "questions", or "suggestions" based on the "Feedback Type" field from the form.
- **Nodes Involved:** Switch
- **Node Details:**
  - **Switch**
    - Type: Conditional routing node.
    - Configuration: Matches lowercase "Feedback Type" to one of three outputs: comments, questions, suggestions.
    - Expressions: `{{$json['Feedback Type'].toLowerCase()}}` used for case-insensitive comparison.
    - Inputs: From JotForm Trigger.
    - Outputs: Three separate branches for each feedback type.
    - Failure modes: Missing or malformed "Feedback Type" field causing no match.
    - Version: 3.3

#### 2.3 Sentiment Analysis & Logging for Comments

- **Overview:** Processes feedback labeled as comments by analyzing sentiment and storing results in a Google Sheet. If sentiment is angry, sends Telegram alert.
- **Nodes Involved:** Google Gemini Chat Model → Sentiment Analysis → Store to Comments Sheet → Send to Support Group (conditional)
- **Node Details:**
  - **Google Gemini Chat Model**
    - Type: AI language model node (Google Gemini/Google Palm API).
    - Configuration: Default options for language model.
    - Role: Likely provides context or preprocessing for sentiment node.
    - Inputs: From Switch node’s “comments” output.
    - Outputs: To Sentiment Analysis.
    - Failure modes: API authentication, rate limits.
    - Version: 1
  - **Sentiment Analysis**
    - Type: LangChain sentiment analysis node.
    - Configuration: Analyzes text from `{{$json['Describe Your Feedback:']}}`.
    - Inputs: From Gemini model.
    - Outputs: To Store to Comments Sheet and Send to Support Group (depending on sentiment).
    - Failure modes: Empty text, API errors.
    - Version: 1.1
  - **Store to Comments Sheet**
    - Type: Google Sheets append node.
    - Configuration: Adds Name, Email, Comments, Sentiment, and current date to “comments” sheet.
    - Inputs: Sentiment Analysis output.
    - Outputs: None (terminal).
    - Failure modes: Sheet access, permission errors.
    - Version: 4.7
  - **Send to Support Group**
    - Type: Telegram message node.
    - Configuration: Sends alert message with customer details and feedback if sentiment is angry/urgent.
    - Inputs: Sentiment Analysis output (conditional).
    - Outputs: To Store to Comments Sheet (duplicates? Possibly fallback).
    - Failure modes: Telegram API limits, wrong chat ID.
    - Version: 1.2

#### 2.4 Question Handling

- **Overview:** For feedback tagged as questions, queries a Google Sheets knowledge base using a LangChain Q&A agent to generate an answer, then sends a styled email reply.
- **Nodes Involved:** Read Database → QnA Agent → Reply Customer
- **Node Details:**
  - **Read Database**
    - Type: Google Sheets Tool node.
    - Configuration: Reads FAQ/knowledge base sheet by document ID and sheet name.
    - Inputs: None (triggered via downstream “ai_tool” input from QnA Agent).
    - Outputs: Data for QnA Agent.
    - Failure modes: Sheet access errors, invalid document or sheet IDs.
    - Version: 4.7
  - **QnA Agent**
    - Type: LangChain agent node.
    - Configuration: Uses feedback text as user prompt, system message sets role as helpful assistant with politeness constraints, fed with Google Sheets data.
    - Expressions: `"user: {{ $json['Describe Your Feedback:'] }}"`.
    - Inputs: From Switch “questions” output and Read Database.
    - Outputs: To Reply Customer.
    - Failure modes: Model API errors, prompt formatting issues.
    - Version: 2.2
  - **Reply Customer**
    - Type: Gmail node.
    - Configuration: Sends an HTML email reply with personalized content, including original question and AI-generated answer.
    - Expressions: Uses Mustache-like template to inject name, email, question, and answer.
    - Inputs: QnA Agent output.
    - Outputs: Terminal node.
    - Failure modes: Gmail OAuth issues, template rendering errors.
    - Version: 2.1

#### 2.5 Suggestion Handling

- **Overview:** For suggestions, summarizes content with AI, alerts the product Telegram channel, and logs the suggestion in a dedicated Google Sheet.
- **Nodes Involved:** Google Gemini Chat Model1 → Summarize Suggestions → Send a text message → Add to Suggestions backlog
- **Node Details:**
  - **Google Gemini Chat Model1**
    - Type: Google Gemini language model.
    - Configuration: Temperature set to 0.4 for balanced creativity.
    - Inputs: Switch “suggestions” output.
    - Outputs: To Summarize Suggestions and QnA Agent (appears in connections).
    - Failure modes: API limits.
    - Version: 1
  - **Summarize Suggestions**
    - Type: LangChain chain summarization node.
    - Configuration: Summarizes suggestion text.
    - Inputs: From Gemini Model1.
    - Outputs: To Send a text message.
    - Failure modes: Empty input, API errors.
    - Version: 2.1
  - **Send a text message**
    - Type: Telegram node.
    - Configuration: Sends summary and full suggestion text to product Telegram chat.
    - Inputs: From Summarize Suggestions.
    - Outputs: To Add to Suggestions backlog.
    - Failure modes: Telegram API or chat ID issues.
    - Version: 1.2
  - **Add to Suggestions backlog**
    - Type: Google Sheets append node.
    - Configuration: Logs Name, Email, Summary, Suggestions, and Created Date to suggestions sheet.
    - Inputs: From Send a text message.
    - Outputs: Terminal.
    - Failure modes: Sheet access, mapping errors.
    - Version: 4.7

#### 2.6 General Logging for Comments

- **Overview:** Stores all comments along with sentiment and metadata to a Google Sheet for reporting and dashboards.
- **Nodes Involved:** Store to Comments Sheet (also involved in 2.3)
- **Node Details:**
  - Same as detailed in 2.3.

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                              | Input Node(s)          | Output Node(s)                    | Sticky Note                                                                                                                                                |
|-----------------------|----------------------------------|----------------------------------------------|-----------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger       | n8n-nodes-base.jotFormTrigger    | Capture new customer feedback form submissions | (Trigger)             | Switch                          | New submission from Jotform: starts when new entry arrives with mapped fields.                                                                             |
| Switch                | n8n-nodes-base.switch            | Classify feedback into comments/questions/suggestions | JotForm Trigger       | Sentiment Analysis, QnA Agent, Summarize Suggestions |                                                                                                                                                            |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provide AI context/preprocessing for sentiment analysis | Switch (comments)      | Sentiment Analysis              |                                                                                                                                                            |
| Sentiment Analysis    | @n8n/n8n-nodes-langchain.sentimentAnalysis | Analyze sentiment of comments                    | Google Gemini Chat Model | Store to Comments Sheet, Send to Support Group | Log for reporting: appends comments with sentiment and metadata.                                                                                          |
| Store to Comments Sheet | n8n-nodes-base.googleSheets      | Log comments and sentiment into Google Sheet    | Sentiment Analysis, Send to Support Group | None                           |                                                                                                                                                            |
| Send to Support Group | n8n-nodes-base.telegram          | Alert support via Telegram on angry feedback     | Sentiment Analysis      | Store to Comments Sheet          |                                                                                                                                                            |
| Read Database         | n8n-nodes-base.googleSheetsTool  | Load FAQ knowledge base for Q&A                    | (ai_tool input from QnA Agent) | QnA Agent                     | Answer with Q&A agent + email reply: loads Q&A sheet for grounding.                                                                                       |
| QnA Agent             | @n8n/n8n-nodes-langchain.agent   | Generate paraphrased, polite answers for questions | Switch (questions), Read Database | Reply Customer               |                                                                                                                                                            |
| Reply Customer        | n8n-nodes-base.gmail             | Send styled HTML email replies to customers       | QnA Agent               | None                           |                                                                                                                                                            |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI model for summarizing suggestions            | Switch (suggestions)     | Summarize Suggestions, QnA Agent | Summarize + alert + log: summarization and Telegram alert for suggestions.                                                                               |
| Summarize Suggestions | @n8n/n8n-nodes-langchain.chainSummarization | Summarize suggestion text                        | Google Gemini Chat Model1 | Send a text message            |                                                                                                                                                            |
| Send a text message   | n8n-nodes-base.telegram          | Notify product Telegram channel with summary and full suggestion | Summarize Suggestions  | Add to Suggestions backlog       |                                                                                                                                                            |
| Add to Suggestions backlog | n8n-nodes-base.googleSheets      | Log suggestions and metadata into Google Sheet    | Send a text message     | None                           |                                                                                                                                                            |
| Sticky Note           | n8n-nodes-base.stickyNote        | Workflow description and overview                  | None                   | None                           | Jotform feedback triage & auto-reply system overview with setup tips.                                                                                     |
| Sticky Note1          | n8n-nodes-base.stickyNote        | Description of Jotform trigger input               | None                   | None                           | New submission from Jotform explanation.                                                                                                                 |
| Sticky Note2          | n8n-nodes-base.stickyNote        | Explanation of Q&A agent and email reply process  | None                   | None                           | Answer with Q&A agent + email reply details.                                                                                                             |
| Sticky Note3          | n8n-nodes-base.stickyNote        | Explanation of suggestions flow                     | None                   | None                           | Summarize + alert + log for suggestions.                                                                                                                 |
| Sticky Note4          | n8n-nodes-base.stickyNote        | Explanation of comment logging                       | None                   | None                           | Log for reporting details.                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a JotForm Trigger Node**
   - Select type: JotForm Trigger.
   - Configure with your Jotform API credentials.
   - Select the specific form ID to listen for new submissions.
   - Map form fields: ensure "Name", "E-mail", "Feedback Type", and "Describe Your Feedback:" are included.

2. **Add a Switch Node to Classify Feedback**
   - Add a Switch node connected to JotForm Trigger.
   - Configure three outputs with rules:
     - Output "comments": when `{{$json["Feedback Type"].toLowerCase()}}` equals "comments"
     - Output "questions": when equals "questions"
     - Output "suggestions": when equals "suggestions"
   - Use case-insensitive matching.

3. **Set Up Google Gemini Chat Model Node for Comments**
   - Add Google Gemini Chat Model node connected to Switch “comments” output.
   - Use default options for the AI model.
   - Authenticate with Google Palm API credentials.

4. **Add Sentiment Analysis Node**
   - Connect Google Gemini Chat Model node output to Sentiment Analysis node.
   - Configure input text as `{{$json["Describe Your Feedback:"]}}`.
   - Use default options.
   - Version 1.1 or later recommended.

5. **Configure Store to Comments Sheet Node**
   - Connect Sentiment Analysis node output to Google Sheets node.
   - Set operation to "append".
   - Map columns: Name, Email, Comments, Sentiment category, Created Date (format: yyyy-MM-dd).
   - Provide Google Sheets OAuth2 credentials.
   - Specify target spreadsheet ID and sheet name for comments.

6. **Set Up Conditional Telegram Alert for Angry Feedback**
   - From Sentiment Analysis node, add Telegram node.
   - Configure message template to include customer name, email, message, and urgent alert text.
   - Use Telegram API credentials and specify target chat ID.
   - Connect Telegram node output back to Store to Comments Sheet (optional redundancy).

7. **Configure Read Database Node for Q&A**
   - Add Google Sheets Tool node.
   - Configure to read your FAQ knowledge base sheet via document ID and sheet name.
   - Authenticate with Google Sheets OAuth2.
   - No direct trigger connection; will be triggered as AI tool input.

8. **Add QnA Agent Node**
   - Connect Switch “questions” output to QnA Agent node.
   - Set prompt text: `"user: {{ $json['Describe Your Feedback:'] }}"`.
   - Define system message with role instructions and polite tone constraints.
   - Link AI tool input from Read Database node.
   - Authenticate with Google Palm API credentials.

9. **Add Gmail Reply Node**
   - Connect QnA Agent output to Gmail node.
   - Configure to send email to `{{$json['E-mail']}}`.
   - Compose styled HTML message:
     - Include customer name, original question, AI-generated answer.
   - Set subject line including short snippet of feedback.
   - Authenticate with Gmail OAuth2 credentials.

10. **Set Up Google Gemini Chat Model1 Node for Suggestions**
    - Connect Switch “suggestions” output to Google Gemini Chat Model1 node.
    - Set temperature to 0.4.
    - Authenticate with Google Palm API.

11. **Add Summarize Suggestions Node**
    - Connect Google Gemini Chat Model1 output to Summarize Suggestions node.
    - Use default summarization options.

12. **Configure Telegram Notification for Suggestions**
    - Connect Summarize Suggestions node to Telegram node.
    - Compose message with customer name, email, summary, and full suggestion.
    - Authenticate with Telegram API, specify target chat ID.

13. **Add Suggestions Logging Node**
    - Connect Telegram node to Google Sheets append node.
    - Map columns: Name, Email, Summary, Suggestions, Created Date.
    - Authenticate with Google Sheets OAuth2.
    - Specify suggestions spreadsheet and sheet.

14. **Add Sticky Notes for Documentation**
    - Add Sticky Note nodes with provided descriptive content for:
      - Workflow overview
      - Jotform input
      - Q&A and email reply
      - Suggestions handling
      - Comments logging

15. **Test Workflow**
    - Submit test entries with each feedback type.
    - Verify branching, AI processing, alerts, logging, and emails.
    - Monitor for errors and resolve credential or API issues.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow automates triage of Jotform feedback into categories with AI-powered processing.     | General project description.                                                                         |
| Urgent/angry messages trigger immediate Telegram alerts to meet a 6-hour SLA.                 | Support alerting SLA note.                                                                           |
| Q&A Agent uses Google Sheets as a knowledge base to ground AI-generated answers.              | See Sticky Note2 for detailed explanation.                                                          |
| Suggestions are summarized, alerted via Telegram, and logged for product team review.         | See Sticky Note3 for workflow details.                                                              |
| Comments are logged with sentiment data for dashboards and periodic analysis.                 | See Sticky Note4 for purpose and usage.                                                             |
| Credentials (Jotform API, Google Palm API, Google Sheets OAuth2, Gmail OAuth2, Telegram API)  | Must be configured securely in node credentials, not hardcoded.                                     |
| Recommended to rename nodes to reflect your environment and maintain credentials separately.   | Setup tip from Sticky Note.                                                                          |
| For more info on LangChain nodes and Google Gemini integration, consult official n8n docs.    | n8n official docs and Google Palm API references.                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies and containing only legal and public data.