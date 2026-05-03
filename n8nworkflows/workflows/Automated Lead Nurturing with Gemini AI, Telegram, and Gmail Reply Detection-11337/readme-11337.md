Automated Lead Nurturing with Gemini AI, Telegram, and Gmail Reply Detection

https://n8nworkflows.xyz/workflows/automated-lead-nurturing-with-gemini-ai--telegram--and-gmail-reply-detection-11337


# Automated Lead Nurturing with Gemini AI, Telegram, and Gmail Reply Detection

### 1. Workflow Overview

This workflow automates lead nurturing by integrating Telegram, Google Gemini AI models, Gmail, and data tables for tracking. It targets sales or customer success teams who want to streamline lead intake, personalize follow-up emails, and track communication status automatically. The workflow consists of three main logical blocks:

- **1.1 Lead Intake & Parsing:** Receives lead information via Telegram from agents, uses Gemini AI to parse and structure lead data, and inserts it into a data table for tracking.

- **1.2 Initial and Follow-up Email Generation & Sending:** Generates personalized emails using Gemini AI models at initial contact and two subsequent follow-ups, sends them via email, updates tracking status, and notifies agents via Telegram.

- **1.3 Reply Detection & Lead Status Update:** Runs scheduled checks on Gmail inbox to detect replies from leads, updates lead status accordingly, and notifies agents.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Intake & Parsing

**Overview:**  
This block collects raw lead data sent by agents through Telegram, leverages Gemini AI to parse the data into structured JSON, and stores the lead details in a data table for further processing.

**Nodes Involved:**  
- Telegram Input - Agent Submits Lead  
- AI - Parse Lead Data  
- Gemini Model - Parser  
- JSON Output Parser  
- Insert row  
- Initialize Lead Tracking  

**Node Details:**

- **Telegram Input - Agent Submits Lead**  
  - Type: Telegram Trigger  
  - Role: Entry point for lead data submitted by agents via Telegram messages.  
  - Configuration: Listens to Telegram messages; expects a specific message format including name, company, email, and notes.  
  - Inputs: None (trigger node)  
  - Outputs: To "AI - Parse Lead Data"  
  - Edge Cases: Unexpected message format may cause parsing failures downstream. Telegram authentication and webhook configuration required.

- **AI - Parse Lead Data**  
  - Type: LangChain Chain LLM  
  - Role: Orchestrates LLM processing chain for lead data parsing.  
  - Configuration: Uses "Gemini Model - Parser" as language model and "JSON Output Parser" to parse output.  
  - Inputs: Raw Telegram message text  
  - Outputs: Structured JSON lead data to "Insert row"  
  - Edge Cases: Model timeout or malformed output from language model may cause failures.

- **Gemini Model - Parser**  
  - Type: LangChain LM Chat Google Gemini  
  - Role: The AI language model that processes raw lead text into structured data.  
  - Configuration: Invoked by "AI - Parse Lead Data"  
  - Inputs: Raw lead text  
  - Outputs: AI raw response to JSON Output Parser  
  - Edge Cases: API quota limits, model errors, or unexpected input formats.

- **JSON Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI response into structured JSON format.  
  - Inputs: AI raw text response  
  - Outputs: JSON lead object to "AI - Parse Lead Data" node  
  - Edge Cases: Parsing errors if AI output is malformed.

- **Insert row**  
  - Type: Data Table  
  - Role: Inserts parsed lead data as a new row in the persistent data table for leads.  
  - Inputs: Parsed JSON lead data  
  - Outputs: To "Initialize Lead Tracking"  
  - Edge Cases: Data table write errors, schema mismatches.

- **Initialize Lead Tracking**  
  - Type: Set  
  - Role: Prepares initial tracking variables for the new lead (e.g., status, timestamps).  
  - Inputs: Newly inserted lead data  
  - Outputs: To "AI - Generate Personalized Email" for first email generation  
  - Edge Cases: Missing or invalid data fields.

---

#### 2.2 Initial and Follow-up Email Generation & Sending

**Overview:**  
Generates personalized emails at three stages: initial outreach, second follow-up, and third follow-up. Uses Gemini AI models to create email content, sends emails via SMTP, updates the lead tracking data, and notifies agents via Telegram of sent emails.

**Nodes Involved:**  
- AI - Generate Personalized Email  
- Gemini Model - Email Generator  
- Email JSON Parser  
- Set Email  
- Send email  
- Update row(s)  
- Notify Agent - Email Sent  
- Daily Follow-up Check (10 AM)  
- Get row(s)1  
- Initialize Lead Follow Up  
- AI - Generate Personalized 2nd Follow Up Email  
- Gemini Model - Email Generator1  
- Email JSON Parser1  
- Set 2nd Follow Up  
- Send email 2  
- Update row(s)1  
- Notify Agent - Email Sent1  
- Daily Follow-up Check (10 AM)1  
- Get row(s)2  
- If lastSentDate was 24 hours ago  
- Initialize Lead 3rd Follow Up  
- AI - Generate Personalized 3rd Follow Up Email  
- Gemini Model - Email Generator2  
- Email JSON Parser2  
- Set 3rd Follow Up  
- Send email 3  
- Update row(s) and Update Status  
- Notify Agent - Sequence Complete1  

**Node Details:**

- **AI - Generate Personalized Email**  
  - Type: LangChain Chain LLM  
  - Role: Initiates AI chain to create a personalized initial email for the lead.  
  - Inputs: Lead info and tracking variables  
  - Outputs: To "Set Email" after parsing  
  - Edge Cases: AI latency, malformed output, or quota issues.

- **Gemini Model - Email Generator**  
  - Type: LangChain LM Chat Google Gemini  
  - Role: Language model that generates email content.  
  - Inputs: Lead data and prompt context  
  - Outputs: Raw AI generated email text  
  - Edge Cases: API errors, inconsistent output.

- **Email JSON Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI email text into structured JSON (e.g., subject, body).  
  - Inputs: Raw AI email text  
  - Outputs: Parsed email data to "AI - Generate Personalized Email"  
  - Edge Cases: Parsing errors.

- **Set Email**  
  - Type: Set  
  - Role: Sets email parameters (recipient, subject, body) for sending.  
  - Inputs: Parsed email data  
  - Outputs: To "Send email" node  
  - Edge Cases: Missing email addresses or malformed fields.

- **Send email**  
  - Type: Email Send  
  - Role: Sends the email via configured SMTP or email credentials.  
  - Inputs: Email parameters  
  - Outputs: To "Update row(s)"  
  - Edge Cases: SMTP auth failures, network timeouts.

- **Update row(s)**  
  - Type: Data Table  
  - Role: Updates lead tracking row with email sent status and timestamp.  
  - Inputs: Success status from "Send email"  
  - Outputs: To "Notify Agent - Email Sent"  
  - Edge Cases: Data write errors.

- **Notify Agent - Email Sent**  
  - Type: Telegram  
  - Role: Sends Telegram notification to agents confirming email sent.  
  - Inputs: Updated lead info  
  - Outputs: None  
  - Edge Cases: Telegram API errors.

- **Daily Follow-up Check (10 AM)**  
  - Type: Schedule Trigger  
  - Role: Scheduled daily trigger to initiate sending follow-up emails.  
  - Inputs: None (time-based trigger)  
  - Outputs: To "Get row(s)1" or "Get row(s)2"

- **Get row(s)1 / Get row(s)2**  
  - Type: Data Table  
  - Role: Retrieves leads eligible for follow-up emails.  
  - Edge Cases: Empty datasets, data retrieval failures.

- **Initialize Lead Follow Up / Initialize Lead 3rd Follow Up**  
  - Type: Set  
  - Role: Sets tracking variables for 2nd and 3rd follow-ups respectively.  
  - Outputs: To respective AI generation nodes.

- **AI - Generate Personalized 2nd Follow Up Email / AI - Generate Personalized 3rd Follow Up Email**  
  - Type: LangChain Chain LLM  
  - Role: Generate personalized follow-up emails using Gemini AI.  
  - Inputs: Lead data and previous interaction info  
  - Outputs: To respective "Set" nodes post-parsing.

- **Gemini Model - Email Generator1 / Gemini Model - Email Generator2**  
  - Type: LangChain LM Chat Google Gemini  
  - Role: Language model for 2nd and 3rd follow-up email generation.

- **Email JSON Parser1 / Email JSON Parser2**  
  - Type: LangChain Output Parser Structured  
  - Role: Parse AI email content into structured JSON.

- **Set 2nd Follow Up / Set 3rd Follow Up**  
  - Type: Set  
  - Role: Set parameters for the follow-up emails before sending.

- **Send email 2 / Send email 3**  
  - Type: Email Send  
  - Role: Send the 2nd and 3rd follow-up emails.

- **Update row(s)1 / Update row(s) and Update Status**  
  - Type: Data Table  
  - Role: Update lead tracking after follow-up emails, including marking sequence complete after 3rd email.

- **Notify Agent - Email Sent1 / Notify Agent - Sequence Complete1**  
  - Type: Telegram  
  - Role: Notify agents of follow-up email sent or sequence completed.

- **If lastSentDate was 24 hours ago**  
  - Type: If  
  - Role: Checks if at least 24 hours passed since last email before sending next follow-up.  
  - Edge Cases: Date parsing errors, timezone issues.

---

#### 2.3 Reply Detection & Lead Status Update

**Overview:**  
Periodically checks Gmail inbox for replies from leads, identifies replies related to tracked leads, updates status to prevent further follow-ups, and notifies agents via Telegram.

**Nodes Involved:**  
- Daily Reply Check (9 AM)  
- Get row(s)  
- Loop Over Items  
- Is This a Reply from Lead?  
- Notify Agent - Reply Received  
- Mark Lead as Replied  
- Mark As Replied  
- Get many messages  

**Node Details:**

- **Daily Reply Check (9 AM)**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 9 AM to scan for replies.  

- **Get row(s)**  
  - Type: Data Table  
  - Role: Retrieves leads to check for replies.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes leads in batches to manage volume and API limits.

- **Is This a Reply from Lead?**  
  - Type: If  
  - Role: Determines whether a received Gmail message is a reply from a lead based on email matching or message content.

- **Notify Agent - Reply Received**  
  - Type: Telegram  
  - Role: Notifies agent that a reply from the lead was received. The Telegram chat ID must be replaced with the actual agent's ID.

- **Mark Lead as Replied**  
  - Type: Set  
  - Role: Flags lead status as "replied" to stop further follow-ups.

- **Mark As Replied**  
  - Type: Data Table  
  - Role: Updates the data table to reflect the lead has replied.

- **Get many messages**  
  - Type: Gmail  
  - Role: Retrieves emails/messages from Gmail inbox to check for replies. Requires Gmail OAuth2 credentials.

- **Edge Cases:**  
  - Gmail API rate limits or auth failures, false positive or negative reply detection due to ambiguous email content, Telegram notification failures.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                             | Input Node(s)                        | Output Node(s)                       | Sticky Note                                                                                         |
|----------------------------------|-------------------------------------|---------------------------------------------|------------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------|
| Telegram Input - Agent Submits Lead | Telegram Trigger                    | Entry point for agents submitting leads    | -                                  | AI - Parse Lead Data                | Agent sends: Name: John Doe; Company: Acme Inc; Email: john@acme.com; Notes: Interested in X product, needs Y feature |
| AI - Parse Lead Data              | LangChain Chain LLM                 | Parses raw lead text using AI chain         | Telegram Input - Agent Submits Lead | Insert row                        |                                                                                                   |
| Gemini Model - Parser             | LangChain LM Chat Google Gemini    | AI model parsing raw lead data               | AI - Parse Lead Data                | JSON Output Parser                 |                                                                                                   |
| JSON Output Parser                | LangChain Output Parser Structured | Parses AI output into structured JSON       | Gemini Model - Parser              | AI - Parse Lead Data               |                                                                                                   |
| Insert row                       | Data Table                         | Inserts new lead data into data table        | AI - Parse Lead Data                | Initialize Lead Tracking           |                                                                                                   |
| Initialize Lead Tracking          | Set                                | Initializes tracking variables for lead     | Insert row                        | AI - Generate Personalized Email  |                                                                                                   |
| AI - Generate Personalized Email | LangChain Chain LLM                 | Generates initial personalized email        | Initialize Lead Tracking           | Set Email                        |                                                                                                   |
| Gemini Model - Email Generator    | LangChain LM Chat Google Gemini    | AI model generating email content            | AI - Generate Personalized Email  | Email JSON Parser                 |                                                                                                   |
| Email JSON Parser                | LangChain Output Parser Structured | Parses AI generated email into JSON          | Gemini Model - Email Generator     | AI - Generate Personalized Email  |                                                                                                   |
| Set Email                       | Set                                | Sets email parameters for sending            | AI - Generate Personalized Email  | Send email                      |                                                                                                   |
| Send email                      | Email Send                        | Sends email via SMTP                          | Set Email                        | Update row(s)                    |                                                                                                   |
| Update row(s)                   | Data Table                         | Updates lead status after sending email      | Send email                      | Notify Agent - Email Sent         |                                                                                                   |
| Notify Agent - Email Sent        | Telegram                          | Notifies agent of email sent                  | Update row(s)                    | -                                |                                                                                                   |
| Daily Reply Check (9 AM)          | Schedule Trigger                   | Triggers daily reply check                     | -                                | Get row(s)                      | Runs every day at 9 AM to check for replies                                                       |
| Get row(s)                      | Data Table                         | Retrieves leads for reply checking             | Daily Reply Check (9 AM)           | Loop Over Items                 |                                                                                                   |
| Loop Over Items                 | Split In Batches                   | Processes leads in batches                      | Get row(s)                      | Is This a Reply from Lead? and Get many messages |                                                                                                   |
| Is This a Reply from Lead?        | If                               | Checks if email is a reply from lead           | Loop Over Items                 | Notify Agent - Reply Received     |                                                                                                   |
| Notify Agent - Reply Received     | Telegram                          | Notifies agent that a reply was received       | Is This a Reply from Lead?         | Mark Lead as Replied            | Replace AGENT_CHAT_ID_HERE with actual agent's Telegram chat ID                                   |
| Mark Lead as Replied             | Set                                | Flags lead as replied                          | Notify Agent - Reply Received    | Mark As Replied                 |                                                                                                   |
| Mark As Replied                 | Data Table                         | Updates data table to mark lead replied        | Mark Lead as Replied             | -                                |                                                                                                   |
| Get many messages               | Gmail                            | Fetches Gmail messages                          | Loop Over Items                 | Loop Over Items                 |                                                                                                   |
| Daily Follow-up Check (10 AM)     | Schedule Trigger                   | Triggers daily follow-up email checks          | -                                | Get row(s)1                    | Checks if it's time to send next email in sequence                                               |
| Get row(s)1                    | Data Table                         | Retrieves leads for 2nd follow-up                | Daily Follow-up Check (10 AM)     | Initialize Lead Follow Up      |                                                                                                   |
| Initialize Lead Follow Up        | Set                                | Initializes tracking variables for 2nd follow-up | Get row(s)1                    | AI - Generate Personalized 2nd Follow Up Email |                                                                                                   |
| AI - Generate Personalized 2nd Follow Up Email | LangChain Chain LLM       | Generates 2nd follow-up email content          | Initialize Lead Follow Up        | Set 2nd Follow Up             |                                                                                                   |
| Gemini Model - Email Generator1  | LangChain LM Chat Google Gemini    | AI model generating 2nd follow-up email content | AI - Generate Personalized 2nd Follow Up Email | Email JSON Parser1           |                                                                                                   |
| Email JSON Parser1              | LangChain Output Parser Structured | Parses 2nd follow-up AI email output            | Gemini Model - Email Generator1  | AI - Generate Personalized 2nd Follow Up Email |                                                                                                   |
| Set 2nd Follow Up              | Set                                | Sets parameters for the 2nd follow-up email     | AI - Generate Personalized 2nd Follow Up Email | Send email 2                 |                                                                                                   |
| Send email 2                   | Email Send                        | Sends the 2nd follow-up email                   | Set 2nd Follow Up               | Update row(s)1                |                                                                                                   |
| Update row(s)1                 | Data Table                         | Updates lead tracking after 2nd email sent      | Send email 2                   | Notify Agent - Email Sent1      |                                                                                                   |
| Notify Agent - Email Sent1        | Telegram                          | Notifies agent of 2nd follow-up email sent       | Update row(s)1                 | -                                |                                                                                                   |
| Daily Follow-up Check (10 AM)1    | Schedule Trigger                   | Triggers daily check for 3rd follow-up           | -                                | Get row(s)2                   | Checks if it's time to send next email in sequence                                               |
| Get row(s)2                    | Data Table                         | Retrieves leads for 3rd follow-up                | Daily Follow-up Check (10 AM)1    | If lastSentDate was 24 hours ago |                                                                                                   |
| If lastSentDate was 24 hours ago | If                               | Checks if 24 hours passed since last email       | Get row(s)2                    | Initialize Lead 3rd Follow Up  |                                                                                                   |
| Initialize Lead 3rd Follow Up    | Set                                | Initializes variables for 3rd follow-up email    | If lastSentDate was 24 hours ago | AI - Generate Personalized 3rd Follow Up Email |                                                                                                   |
| AI - Generate Personalized 3rd Follow Up Email | LangChain Chain LLM       | Generates 3rd follow-up email content          | Initialize Lead 3rd Follow Up    | Set 3rd Follow Up             |                                                                                                   |
| Gemini Model - Email Generator2  | LangChain LM Chat Google Gemini    | AI model generating 3rd follow-up email content | AI - Generate Personalized 3rd Follow Up Email | Email JSON Parser2           |                                                                                                   |
| Email JSON Parser2              | LangChain Output Parser Structured | Parses 3rd follow-up AI email output            | Gemini Model - Email Generator2  | AI - Generate Personalized 3rd Follow Up Email |                                                                                                   |
| Set 3rd Follow Up              | Set                                | Sets parameters for the 3rd follow-up email     | AI - Generate Personalized 3rd Follow Up Email | Send email 3                 |                                                                                                   |
| Send email 3                   | Email Send                        | Sends the 3rd follow-up email                   | Set 3rd Follow Up               | Update row(s) and Update Status |                                                                                                   |
| Update row(s) and Update Status  | Data Table                         | Updates lead status and marks sequence complete  | Send email 3                   | Notify Agent - Sequence Complete1 |                                                                                                   |
| Notify Agent - Sequence Complete1 | Telegram                          | Notifies agent that email sequence has completed | Update row(s) and Update Status | -                                |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Input Node**  
   - Type: Telegram Trigger  
   - Configure with bot credentials and webhook.  
   - No additional parameters; listens for new messages from agents.

2. **Create AI Parsing Chain**  
   - Create LangChain Chain LLM node named "AI - Parse Lead Data".  
   - Attach a LangChain LM Chat Google Gemini node "Gemini Model - Parser".  
   - Attach a LangChain Output Parser Structured node "JSON Output Parser" to parse Gemini output.  
   - Chain these together accordingly.

3. **Create Data Table Insert Node**  
   - Create a Data Table node "Insert row".  
   - Configure to point to your leads data table with appropriate schema (name, company, email, notes, status, timestamps).  
   - Connect output from "AI - Parse Lead Data".

4. **Initialize Lead Tracking Variables**  
   - Add a Set node "Initialize Lead Tracking" to add tracking fields (e.g., status = "new", followUpCount = 0, lastSentDate = null).

5. **Create First Email Generation Chain**  
   - Add LangChain Chain LLM node "AI - Generate Personalized Email".  
   - Connect LangChain LM Chat Google Gemini node "Gemini Model - Email Generator".  
   - Attach LangChain Output Parser Structured node "Email JSON Parser" to parse email content (subject, body).  
   - Connect output to a Set node "Set Email" to set email recipient, subject, and body fields.

6. **Create Send Email Node**  
   - Add Email Send node "Send email".  
   - Configure with SMTP or email service credentials.  
   - Connect from "Set Email".

7. **Update Data Table after Email Sent**  
   - Add Data Table node "Update row(s)" to update lead record with email sent timestamp and status.

8. **Notify Agent via Telegram**  
   - Add Telegram node "Notify Agent - Email Sent" configured with agent chat ID to confirm email sent.

9. **Set Up Scheduled Triggers for Reply Checks and Follow-ups**  
   - Add Schedule Trigger node "Daily Reply Check (9 AM)".  
   - Add Schedule Trigger nodes "Daily Follow-up Check (10 AM)" and "Daily Follow-up Check (10 AM)1" for follow-ups.

10. **Reply Detection Logic**  
    - Create Data Table node "Get row(s)" to fetch leads for reply check.  
    - Add Split In Batches node "Loop Over Items" to iterate leads.  
    - Add Gmail node "Get many messages" to fetch recent emails from inbox (configure using Gmail OAuth2).  
    - Add If node "Is This a Reply from Lead?" to check if messages are replies from leads.  
    - Connect to Telegram node "Notify Agent - Reply Received" (replace with actual agent chat ID).  
    - Add Set node "Mark Lead as Replied" and Data Table node "Mark As Replied" to update lead status.

11. **Follow-up Email Sequence (2nd and 3rd Emails)**  
    - For 2nd and 3rd follow-ups, replicate the AI email generation chain with separate LangChain Chain LLM nodes, Gemini models, and parsers (e.g., "AI - Generate Personalized 2nd Follow Up Email" and "Gemini Model - Email Generator1", etc.).  
    - Use Set nodes to configure follow-up email parameters.  
    - Use Send Email nodes "Send email 2" and "Send email 3".  
    - Update data table nodes accordingly for each step.  
    - Add appropriate Telegram notifications for follow-up emails sent and sequence completion.

12. **Conditional Logic for Timing Follow-ups**  
    - Add If node "If lastSentDate was 24 hours ago" to ensure follow-ups only send after 24 hours have passed since last email.

13. **Data Table Schema and Credentials**  
    - Ensure your data table schema supports fields like lead info, status, lastSentDate, followUpCount, replied flags.  
    - Configure credentials for Telegram bots, Gmail OAuth2, SMTP email sending, and Google Gemini AI access.

14. **Testing and Validation**  
    - Test each block independently before end-to-end run.  
    - Verify Telegram input reception, AI parsing accuracy, email generation correctness, email sending, data updates, and reply detection.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Agent message format must be consistent (Name, Company, Email, Notes) for AI parser to function correctly.    | Telegram Input node instructions.                                                               |
| Replace placeholder AGENT_CHAT_ID_HERE in "Notify Agent - Reply Received" node with actual Telegram chat ID. | Telegram notification setup.                                                                     |
| Gmail node requires OAuth2 credentials with Gmail API scope for reading inbox messages.                       | Gmail node configuration.                                                                        |
| Google Gemini AI nodes require API access and appropriate environment setup for LangChain integration.         | Google Gemini AI and LangChain nodes documentation.                                             |
| Follow-up emails are spaced by at least 24 hours to avoid spamming leads.                                      | Controlled by "If lastSentDate was 24 hours ago" node.                                          |
| Telegram notifications keep agents informed of process status and lead engagement.                            | Improves team responsiveness and tracking.                                                     |
| Workflow includes robust error handling edges but watch for API rate limits on Telegram, Gmail, and Gemini.   | Consider adding retry or error workflow branches if needed.                                     |
| For more on managing lead nurturing automations with AI, see external blog: https://n8n.io/blog/lead-nurturing | External resource for conceptual understanding and best practices.                              |

---

**Disclaimer:**  
The text provided is exclusively generated from an n8n automated workflow. All data processing respects applicable content policies and contains no illegal or protected information. All handled data is legal and public.