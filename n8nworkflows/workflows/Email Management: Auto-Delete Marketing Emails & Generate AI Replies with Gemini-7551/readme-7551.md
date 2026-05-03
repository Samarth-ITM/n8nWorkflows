Email Management: Auto-Delete Marketing Emails & Generate AI Replies with Gemini

https://n8nworkflows.xyz/workflows/email-management--auto-delete-marketing-emails---generate-ai-replies-with-gemini-7551


# Email Management: Auto-Delete Marketing Emails & Generate AI Replies with Gemini

### 1. Workflow Overview

This workflow automates the management of incoming emails by detecting marketing emails and handling them accordingly. Its main purpose is to:

- Automatically classify incoming emails as marketing or non-marketing.
- Delete marketing emails after logging their details.
- Generate AI-powered custom replies for non-marketing emails and send responses.
- Log all processed emails (both deleted and replied) in Google Sheets for tracking and auditing.

The workflow is logically divided into the following blocks:

**1.1 Input Reception**  
- Triggered either manually or automatically on new emails arriving via IMAP or on-demand.

**1.2 AI Classification and Reply Generation**  
- Uses Google Gemini AI model to determine if emails are marketing-related.  
- If non-marketing, generates a respectful custom reply.

**1.3 Email Action Routing**  
- Routes emails for deletion if marketing.  
- Routes emails for replying if non-marketing.

**1.4 Email Action Execution**  
- Deletes marketing emails via Gmail node.  
- Replies to non-marketing emails via Gmail node.

**1.5 Tracking and Logging**  
- Logs deleted emails to one Google Sheet tab.  
- Logs replied emails to another Google Sheet tab.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block handles the start of the workflow, either by manual trigger or automatically when a new email is received. It fetches emails for processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Email Trigger (IMAP)  
- Get many messages (Gmail)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual workflow start for on-demand processing.  
  - Config: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Get many messages" node.  
  - Failure types: None specific; user action required to start.

- **Email Trigger (IMAP)**  
  - Type: Email Read (IMAP)  
  - Role: Automatically triggers workflow when a new email arrives in IMAP inbox.  
  - Config: Default IMAP inbox monitoring, no filters applied.  
  - Inputs: None  
  - Outputs: Connected to "Message a model" node.  
  - Failure types: Possible IMAP connection/authentication failures, network timeouts.

- **Get many messages**  
  - Type: Gmail node, operation "getAll"  
  - Role: Fetches a batch of emails; limited to 2 messages per execution for testing or batching.  
  - Config: No filters; fetches latest 2 emails.  
  - Inputs: Triggered by manual trigger node.  
  - Outputs: Feeds data into "Message a model".  
  - Failure types: Gmail API rate limits, authentication failures.

---

#### 2.2 AI Classification and Reply Generation

**Overview:**  
This block sends email content to Google Gemini AI to classify emails as marketing or not, and generate a custom reply message if needed.

**Nodes Involved:**  
- Message a model (Google Gemini)  
- AI response formatter (Set)  

**Node Details:**

- **Message a model**  
  - Type: Google Gemini (LangChain integration)  
  - Role: Sends email Subject, From, snippet, and id to Gemini AI for classification.  
  - Config: Uses "models/gemini-2.5-flash" model.  
  - Message Content: Includes email fields and requests Gemini to flag "isMarketing" as true/false, and if false, generate a respectful reply message in the response.  
  - Inputs: Receives emails from "Get many messages" (manual) or "Email Trigger (IMAP)".  
  - Outputs: JSON output with fields including `isMarketing` and optionally `replyMessage`.  
  - Failure types: AI model invocation errors, API quota issues, malformed responses.

- **AI response formatter**  
  - Type: Set node  
  - Role: Extracts and restructures AI response JSON to a usable format for routing.  
  - Config: Assigns the nested AI response under `content.parts[0].text`.  
  - Inputs: From "Message a model".  
  - Outputs: To "categories emails" switch node.  
  - Failure types: Expression errors if AI response structure changes.

---

#### 2.3 Email Action Routing

**Overview:**  
Routes emails based on the AI classification flag `isMarketing`. Marketing emails are sent to deletion workflow; non-marketing emails proceed to the reply workflow.

**Nodes Involved:**  
- categories emails (Switch node)

**Node Details:**

- **categories emails**  
  - Type: Switch  
  - Role: Checks if `isMarketing` flag is true or false in the AI response.  
  - Config: Two rules - one for `isMarketing` equals true (marketing email), another for false.  
  - Inputs: From "AI response formatter".  
  - Outputs:  
    - True (marketing) → "Delete a message" node  
    - False (non-marketing) → "Reply to a message" node  
  - Failure types: Expression evaluation errors if flag missing or incorrectly formatted.

---

#### 2.4 Email Action Execution

**Overview:**  
Performs the actual email operations: deleting marketing emails and replying to non-marketing emails using Gmail nodes.

**Nodes Involved:**  
- Delete a message (Gmail)  
- Reply to a message (Gmail)

**Node Details:**

- **Delete a message**  
  - Type: Gmail node  
  - Role: Deletes the email identified by message ID.  
  - Config: Uses message ID from AI response JSON.  
  - Inputs: From "categories emails" switch node (marketing branch).  
  - Outputs: Connected to "Append or update row in sheet" for logging.  
  - Failure types: Gmail API errors, permission issues, invalid messageId.

- **Reply to a message**  
  - Type: Gmail node  
  - Role: Sends a reply to the original sender using AI-generated response.  
  - Config: Reply message content from AI response JSON, message ID to specify which email to reply to.  
  - Inputs: From "categories emails" switch node (non-marketing branch).  
  - Outputs: Connected to "Append or update row in sheet1" for logging.  
  - Failure types: Gmail API errors, invalid messageId, insufficient permissions.

---

#### 2.5 Tracking and Logging

**Overview:**  
Logs details of deleted and replied emails into separate Google Sheet tabs for auditing and monitoring.

**Nodes Involved:**  
- Append or update row in sheet (Google Sheets)  
- Append or update row in sheet1 (Google Sheets)

**Node Details:**

- **Append or update row in sheet**  
  - Type: Google Sheets node  
  - Role: Logs deleted emails with subject and email ID into a sheet tab named "deleted emails".  
  - Config: Document ID and sheet name specified. Matching column set to "email id" for upsert behavior.  
  - Inputs: From "Delete a message".  
  - Outputs: None.  
  - Failure types: Google Sheets API errors, permission denials.

- **Append or update row in sheet1**  
  - Type: Google Sheets node  
  - Role: Logs replied emails with subject and email ID into a separate tab identified by gid "1619439968".  
  - Config: Document ID same as above, matching on "email id".  
  - Inputs: From "Reply to a message".  
  - Outputs: None.  
  - Failure types: Google Sheets API errors, permission denials.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                              | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                   |
|-----------------------------|-------------------------|----------------------------------------------|----------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Manual start of workflow                      | None                       | Get many messages                | # 🖐️ Manual Trigger ⚡ Triggered based on user actions – the workflow starts when the user decides to take action 🧠🕹️ |
| Email Trigger (IMAP)         | Email Read (IMAP)       | Auto start on new email arrival               | None                       | Message a model                 | # ⏰ Scheduled Trigger ⚡ Instantly kicks off the workflow whenever a new 📩 email lands in the account – no delays, fully automated! 🤖✨ |
| Get many messages            | Gmail                   | Fetches emails batch                          | When clicking ‘Execute workflow’ | Message a model                 |                                                                                                               |
| Message a model              | Google Gemini           | Classifies email, generates reply message    | Get many messages, Email Trigger (IMAP) | AI response formatter          | # 🚀 Gemini Model Tasks 📌 Objectives: - 🕵️‍♂️ Detect if an incoming email is a marketing email - 🏷️ If yes → Add a classification flag like isMarketing: true ✅ - ✉️ If not a marketing email → Prepare a customized response for clients 🤝 |
| AI response formatter       | Set                     | Formats AI response to usable JSON object    | Message a model             | categories emails               | # 🏷️ Classification of Emails 📌 Based on the feature flag present in the response, this node decides whether the email should be: ✉️ Replied to → Sent to the Reply Workflow 🔁 🗑️ Deleted → Sent to the Delete Workflow ❌ 🤖 Smart decision-making to route emails efficiently! |
| categories emails           | Switch                  | Routes emails based on isMarketing flag      | AI response formatter       | Delete a message, Reply to a message |                                                                                                               |
| Delete a message            | Gmail                   | Deletes marketing emails                      | categories emails           | Append or update row in sheet   | # 📊 Tracking 🛑 This is the final step of the workflow where all actioned emails are recorded. 🗂️ It keeps track of: 🗑️ Deleted emails ✉️ Replied emails, along with their 📌 email subjects 🔍 Useful for monitoring, auditing, and future reference! |
| Reply to a message          | Gmail                   | Sends AI-generated replies                    | categories emails           | Append or update row in sheet1  |                                                                                                               |
| Append or update row in sheet | Google Sheets           | Logs deleted emails                           | Delete a message            | None                          |                                                                                                               |
| Append or update row in sheet1 | Google Sheets           | Logs replied emails                           | Reply to a message          | None                          |                                                                                                               |
| Sticky Note                 | Sticky Note             | Explains Gemini model task objectives         | None                       | None                          | See above node-specific sticky notes.                                                                         |
| Sticky Note1                | Sticky Note             | Explains scheduled trigger                    | None                       | None                          | See above node-specific sticky notes.                                                                         |
| Sticky Note2                | Sticky Note             | Explains classification switch logic          | None                       | None                          | See above node-specific sticky notes.                                                                         |
| Sticky Note3                | Sticky Note             | Explains tracking and logging                  | None                       | None                          | See above node-specific sticky notes.                                                                         |
| Sticky Note4                | Sticky Note             | Explains manual trigger                        | None                       | None                          | See above node-specific sticky notes.                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: When clicking ‘Execute workflow’  
   - Type: Manual Trigger (built-in)  
   - No parameters needed.

2. **Create Email Trigger (IMAP) node**  
   - Name: Email Trigger (IMAP)  
   - Type: Email Read (IMAP)  
   - Parameters: Configure IMAP credentials and server settings for your email account.  
   - No filters applied for new emails.  
   - Connect no inputs; this node is a trigger.

3. **Create Gmail node to Get many messages**  
   - Name: Get many messages  
   - Type: Gmail  
   - Operation: getAll  
   - Limit: 2 (adjust as needed)  
   - Connect input from "When clicking ‘Execute workflow’" manual trigger.

4. **Create Google Gemini node (LangChain integration)**  
   - Name: Message a model  
   - Type: @n8n/n8n-nodes-langchain.googleGemini  
   - Parameters:  
     - Model ID: Select "models/gemini-2.5-flash"  
     - Messages: Configure a text prompt that includes:  
       - Email Subject, From, snippet, id fields interpolated from the input JSON.  
       - Instructions to classify email as marketing or not (`isMarketing` flag).  
       - If not marketing, generate a custom reply message.  
     - Enable JSON output.  
   - Connect inputs from both "Get many messages" and "Email Trigger (IMAP)".

5. **Create Set node to format AI response**  
   - Name: AI response formatter  
   - Type: Set  
   - Parameters: Assign the property `content.parts[0].text` from the incoming JSON to the same path in the output JSON to make it easier to access downstream.  
   - Connect input from "Message a model".

6. **Create Switch node for email categorization**  
   - Name: categories emails  
   - Type: Switch  
   - Parameters:  
     - Add two rules:  
       - Rule 1: `isMarketing` equals true → marketing email branch  
       - Rule 2: `isMarketing` equals false → non-marketing email branch  
     - Use boolean operations checking the field path `content.parts[0].text.isMarketing`.  
   - Connect input from "AI response formatter".

7. **Create Gmail node to delete messages**  
   - Name: Delete a message  
   - Type: Gmail  
   - Operation: delete  
   - Message ID: Map from `content.parts[0].text.id` from AI response JSON.  
   - Connect input from "categories emails" marketing branch output.

8. **Create Gmail node to reply to messages**  
   - Name: Reply to a message  
   - Type: Gmail  
   - Operation: reply  
   - Message: Map reply content from `content.parts[0].text.replyMessage` from AI response JSON.  
   - Message ID: Map from `content.parts[0].text.id`.  
   - Connect input from "categories emails" non-marketing branch output.

9. **Create Google Sheets node to log deleted emails**  
   - Name: Append or update row in sheet  
   - Type: Google Sheets  
   - Operation: appendOrUpdate  
   - Document ID: Use your target Google Sheet document ID.  
   - Sheet Name: Use tab named "deleted emails" or equivalent.  
   - Columns: Map "subject" and "email id" from corresponding Gmail message fields.  
   - Matching Columns: "email id" for upsert.  
   - Connect input from "Delete a message".

10. **Create Google Sheets node to log replied emails**  
    - Name: Append or update row in sheet1  
    - Type: Google Sheets  
    - Operation: appendOrUpdate  
    - Document ID: Same as above.  
    - Sheet Name: Use tab identified by gid or named "replied email".  
    - Columns: Map "subject" and "email id" similarly.  
    - Matching Columns: "email id".  
    - Connect input from "Reply to a message".

11. **Credential Setup**  
    - Gmail nodes require OAuth2 credentials configured with Gmail API access.  
    - Google Sheets nodes require OAuth2 credentials with Sheet editing rights.  
    - Google Gemini node requires authentication with Google Cloud or LangChain-compatible AI model API credentials.

12. **Testing and Validation**  
    - Test manual trigger to fetch emails and process.  
    - Test IMAP trigger with incoming emails.  
    - Validate AI response correctness and routing.  
    - Check logs in Google Sheets tabs.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| # 🚀 Gemini Model Tasks: Detect marketing emails and generate custom replies using Google Gemini AI.               | Sticky Note attached near "Message a model" node.                                                  |
| # ⏰ Scheduled Trigger: Automated workflow start on new email arrival, no delays.                                  | Sticky Note near Email Trigger (IMAP) node.                                                       |
| # 🏷️ Classification of Emails: Routes emails for reply or deletion based on AI flag `isMarketing`.                | Sticky Note near "categories emails" switch node.                                                 |
| # 📊 Tracking: Logs all deleted and replied emails for audit and monitoring.                                       | Sticky Note near Google Sheets nodes.                                                             |
| Workflow respects n8n content policies and handles only public, legal data.                                       | Disclaimer: Workflow content is fully compliant and legal.                                        |
| Google Gemini model used: "models/gemini-2.5-flash"                                                               | Model selection for AI classification and reply generation.                                       |
| Google Sheets document used for logging: https://docs.google.com/spreadsheets/d/1EsLx3e71u3YIboqIlc7pUM81YJAjNuhMSqv7kgyGkzw | Referenced for appending/deleting email logs.                                                     |

---

This completes the detailed analysis and documentation of the "Email Management: Auto-Delete Marketing Emails & Generate AI Replies with Gemini" workflow.