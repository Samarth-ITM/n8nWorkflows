Multilingual Email Auto-Replies with DeepL, GPT-4o and Slack Human Approval

https://n8nworkflows.xyz/workflows/multilingual-email-auto-replies-with-deepl--gpt-4o-and-slack-human-approval-10907


# Multilingual Email Auto-Replies with DeepL, GPT-4o and Slack Human Approval

---

### 1. Workflow Overview

This workflow automates multilingual email responses from a Gmail inbox by integrating translation, AI-generated drafting, and human approval via Slack before sending replies. It targets users who receive emails in various languages and want to automate initial processing while preserving quality control through human oversight.

The workflow is logically divided into the following blocks:

- **1.1 Email Detection:** Monitors Gmail inbox for new emails with a specific label.
- **1.2 Configuration Setup:** Sets runtime configuration parameters, mainly the Slack channel ID for notifications.
- **1.3 Email Translation:** Translates the incoming email snippet to English using DeepL.
- **1.4 AI Processing:** Uses GPT-4o-mini model to generate a concise summary and a professional draft reply based on the translated content.
- **1.5 Human Approval Request:** Sends a message to a Slack channel with the AI-generated content, requesting human approval via a clickable link.
- **1.6 Wait for Approval:** Pauses workflow execution until human approval is received through the Slack link.
- **1.7 Send Reply:** Sends the approved reply email back via Gmail.
- **1.8 Post-Processing:** Removes the applied label on the email to avoid reprocessing.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Detection

- **Overview:** Watches the Gmail inbox for incoming emails labeled "INBOX" (or user-specified label) and triggers the workflow on every new email found.
- **Nodes Involved:**  
  - Gmail Trigger
- **Node Details:**  
  - **Gmail Trigger**  
    - *Type:* Trigger node for Gmail email reception.  
    - *Configuration:* Filters set to label "INBOX"; polls every minute.  
    - *Expressions:* None; direct filter on label ID.  
    - *Input:* External event (new email in Gmail).  
    - *Output:* Email metadata and snippet passed to next node.  
    - *Potential Failures:* Authentication errors with Gmail OAuth2, rate limiting, label misconfiguration, or no new emails triggering.  
    - *Notes:* User should specify custom label if desired (e.g., "To_Process").

#### 1.2 Configuration Setup

- **Overview:** Assigns static runtime parameters, specifically the Slack channel ID where approval requests will be sent.
- **Nodes Involved:**  
  - Configuration
- **Node Details:**  
  - **Configuration**  
    - *Type:* Set node for static variable assignment.  
    - *Configuration:* Slack channel ID placeholder "YOUR_SLACK_CHANNEL_ID_HERE" must be replaced by user.  
    - *Expressions:* None; value is static string.  
    - *Input:* Receives data from Gmail Trigger.  
    - *Output:* Adds `slackChannel` property to JSON output.  
    - *Failure Cases:* Missing or incorrect Slack channel ID will cause Slack node failures downstream.

#### 1.3 Email Translation

- **Overview:** Translates the email snippet content to English using DeepL API.
- **Nodes Involved:**  
  - Translate Email
- **Node Details:**  
  - **Translate Email**  
    - *Type:* DeepL translation node.  
    - *Configuration:* Source text is the email snippet from Gmail Trigger (`{{$json.snippet}}`), target language set to English ("EN").  
    - *Expressions:* `={{ $json.snippet }}` extracts snippet.  
    - *Input:* Receives email data and configuration.  
    - *Output:* Outputs translated text as `translatedText`.  
    - *Failures:* DeepL API key missing or invalid, API quota exceeded, network timeout, or empty snippet text.  
    - *Notes:* Target language can be changed as needed.

#### 1.4 AI Processing

- **Overview:** Generates an email summary (3 bullet points) and a professional draft reply using GPT-4o-mini, based on the translated email content.
- **Nodes Involved:**  
  - Generate AI Response
- **Node Details:**  
  - **Generate AI Response**  
    - *Type:* LangChain OpenAI node configured for GPT-4o-mini model.  
    - *Configuration:* Input prompt instructs the AI to:  
      1. Summarize the translated email in 3 bullet points.  
      2. Draft a professional polite reply.  
      - Prompt uses expression to inject translated text from "Translate Email" node (`{{ $('Translate Email').item.json.translatedText }}`).  
    - *Expressions:*  
      - Prompt content with embedded translated text.  
    - *Input:* Receives translated text.  
    - *Output:* JSON message with AI-generated summary and draft reply.  
    - *Failures:* OpenAI API key issues, rate limits, malformed prompt, or unexpected AI responses.  
    - *Notes:* Prompt and model can be customized.

#### 1.5 Human Approval Request

- **Overview:** Sends a formatted Slack message to a configured channel containing email metadata and the AI-generated content. Includes a clickable link for the user to approve sending the reply.
- **Nodes Involved:**  
  - Request Approval
- **Node Details:**  
  - **Request Approval**  
    - *Type:* Slack node for sending messages.  
    - *Configuration:*  
      - Text includes: sender, subject, AI summary and draft reply, and approval action link (`{{$execution.resumeUrl}}`).  
      - Channel ID dynamically assigned from Configuration node's `slackChannel`.  
      - Authenticated via OAuth2 credentials.  
    - *Expressions:* Multiple expressions to pull data from Gmail Trigger and AI Response nodes.  
    - *Input:* Receives AI-generated message content.  
    - *Output:* Triggers next node on message sent.  
    - *Failures:* Slack API auth errors, invalid channel ID, network issues.  
    - *Notes:* Approval link pauses workflow until user clicks.

#### 1.6 Wait for Approval

- **Overview:** Pauses workflow execution until human approval is received via the Slack link, which resumes the workflow.
- **Nodes Involved:**  
  - Wait for Approval
- **Node Details:**  
  - **Wait for Approval**  
    - *Type:* Wait node configured for webhook resume.  
    - *Configuration:* Waits with webhook ID "approval-webhook" for resuming.  
    - *Input:* Triggered from Slack approval message.  
    - *Output:* Continues to sending email reply.  
    - *Failures:* Timeout if no approval; webhook URL must be accessible publicly; user must click approval link.  
    - *Notes:* Requires n8n instance to be externally accessible.

#### 1.7 Send Reply

- **Overview:** Sends the approved AI-generated reply back to the original email thread using Gmail reply operation.
- **Nodes Involved:**  
  - Send Email Reply
- **Node Details:**  
  - **Send Email Reply**  
    - *Type:* Gmail node for sending email replies.  
    - *Configuration:*  
      - Message body extracted from the AI response, specifically the text after "【Draft Reply】".  
      - Includes signature line: "--- Sent via n8n automation".  
      - Uses the original email's message ID for threading.  
      - Operation set to "reply".  
    - *Expressions:*  
      - Extract reply text: `={{ $('Generate AI Response').item.json.message.content.split('【Draft Reply】')[1].trim() }}`  
      - Uses original email ID to reply in thread.  
    - *Input:* Resumed workflow after approval.  
    - *Output:* Passes to label removal node.  
    - *Failures:* Gmail OAuth2 errors, malformed message, reply threading errors.

#### 1.8 Post-Processing

- **Overview:** Removes the initial processing label from the email to avoid duplicate processing.
- **Nodes Involved:**  
  - Mark as Processed
- **Node Details:**  
  - **Mark as Processed**  
    - *Type:* Gmail node to modify email labels.  
    - *Configuration:*  
      - Removes label ID originally attached to the email (extracted dynamically).  
      - Uses original email message ID.  
      - Operation: removeLabels.  
    - *Expressions:*  
      - Label ID: `={{ $('Gmail Trigger').item.json.labelIds[0] }}`  
      - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
    - *Input:* Receives from Send Email Reply node.  
    - *Output:* Ends workflow.  
    - *Failures:* Gmail API permission errors, label ID missing or already removed.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role                     | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                   |
|---------------------|-----------------------|-----------------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------|
| Gmail Trigger       | Gmail Trigger          | Detect new incoming emails        | External trigger      | Configuration         | Set your specific Gmail label here (e.g., 'To_Process')                                       |
| Configuration       | Set                   | Define Slack channel ID            | Gmail Trigger         | Translate Email       | Enter your Slack Channel ID here                                                              |
| Translate Email     | DeepL                  | Translate email snippet to English | Configuration         | Generate AI Response  |                                                                                               |
| Generate AI Response| LangChain OpenAI       | Create summary and draft reply    | Translate Email       | Request Approval      |                                                                                               |
| Request Approval    | Slack                  | Send approval request message     | Generate AI Response  | Wait for Approval     |                                                                                               |
| Wait for Approval   | Wait                   | Pause until human approval        | Request Approval      | Send Email Reply      |                                                                                               |
| Send Email Reply    | Gmail                  | Send approved reply email         | Wait for Approval     | Mark as Processed     |                                                                                               |
| Mark as Processed   | Gmail                  | Remove email label to prevent loop | Send Email Reply      | None                  | Removes label to prevent reprocessing                                                         |
| 📋 Template Overview| Sticky Note            | Information and instructions      | None                  | None                  | ## 💡 How to Use This Template\n\nThis workflow creates an intelligent email assistant that...|
| Step 1              | Sticky Note            | Email Detection block description | None                  | None                  | ### 1️⃣ Email Detection\nMonitors Gmail for emails with your specified label                   |
| Step 2              | Sticky Note            | Configuration block description   | None                  | None                  | ### 2️⃣ Configuration\nSet your Slack channel ID for notifications                            |
| Step 3              | Sticky Note            | Translation block description     | None                  | None                  | ### 3️⃣ Translation\nAutomatically translates email content using DeepL                       |
| Step 4              | Sticky Note            | AI Processing block description   | None                  | None                  | ### 4️⃣ AI Processing\nGenerates summary and professional reply draft                        |
| Step 5              | Sticky Note            | Human Approval block description  | None                  | None                  | ### 5️⃣ Human Approval\nSends approval request to Slack with one-click link                   |
| Step 6              | Sticky Note            | Send Reply block description      | None                  | None                  | ### 6️⃣ Send Reply\nAutomatically sends the approved email response                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Parameters:  
     - Filter by label "INBOX" or your custom label (e.g., "To_Process")  
     - Poll interval: every minute  
   - Credentials: Configure Gmail OAuth2 with appropriate scopes.

2. **Add Configuration (Set) node**  
   - Type: Set  
   - Parameters:  
     - Assign variable `slackChannel` with your Slack channel ID as a string (e.g., "C12345678").  
   - Connect output from Gmail Trigger node.

3. **Add DeepL Translate node**  
   - Type: DeepL  
   - Parameters:  
     - Text: expression `={{ $json.snippet }}` (email snippet from Gmail Trigger)  
     - Target language: "EN" (or your preferred language code)  
   - Credentials: Provide DeepL API key.  
   - Connect output from Configuration node.

4. **Add LangChain OpenAI node**  
   - Type: LangChain OpenAI  
   - Parameters:  
     - Model: "gpt-4o-mini"  
     - Messages: Use a system prompt that instructs AI to generate:  
       1. A concise summary (3 bullet points)  
       2. A professional, polite reply draft  
     - Inject translated text with expression: `{{ $('Translate Email').item.json.translatedText }}`  
   - Credentials: Configure OpenAI API key.  
   - Connect output from Translate Email node.

5. **Add Slack node for approval request**  
   - Type: Slack  
   - Parameters:  
     - Text: Format message including:  
       - Sender (`{{ $('Gmail Trigger').item.json.from }}`)  
       - Subject (`{{ $('Gmail Trigger').item.json.subject }}`)  
       - AI-generated content (`{{ $json.message.content }}`)  
       - Approval link: `{{ $execution.resumeUrl }}`  
     - Channel: Use expression `={{ $('Configuration').item.json.slackChannel }}`  
   - Credentials: Configure Slack OAuth2 with chat permissions.  
   - Connect output from Generate AI Response node.

6. **Add Wait node**  
   - Type: Wait  
   - Parameters:  
     - Resume on webhook  
     - Webhook ID: "approval-webhook" (or your generated webhook ID)  
   - Connect output from Request Approval node.

7. **Add Gmail node to send reply**  
   - Type: Gmail (Send)  
   - Parameters:  
     - Operation: Reply  
     - Message: Extract draft reply portion from AI response using expression:  
       `={{ $('Generate AI Response').item.json.message.content.split('【Draft Reply】')[1].trim() }}`  
     - Include signature: "\n\n---\nSent via n8n automation"  
     - Message ID: Use original email ID `={{ $('Gmail Trigger').item.json.id }}`  
   - Credentials: Gmail OAuth2 with send scope.  
   - Connect output from Wait node.

8. **Add Gmail node to remove label**  
   - Type: Gmail  
   - Parameters:  
     - Operation: Remove Labels  
     - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
     - Label IDs: `={{ $('Gmail Trigger').item.json.labelIds[0] }}`  
   - Credentials: Same Gmail credentials as above.  
   - Connect output from Send Email Reply node.

9. **Add sticky notes for documentation** (optional)  
   - Add notes describing each step for clarity and maintenance.

10. **Test the workflow**  
    - Ensure n8n is publicly accessible for webhook resume.  
    - Verify OAuth2 credentials for Gmail, Slack, DeepL, OpenAI.  
    - Replace placeholder Slack channel ID before activation.  
    - Confirm emails are labeled correctly and processed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow requires your n8n instance to be accessible via webhook for approval step; use tunneling services like ngrok for testing or deploy on a public server.                                                                                                                                                                     | General deployment requirement                                                                          |
| Labels must be configured carefully in Gmail to prevent emails from being processed multiple times; processed emails have their labels removed to avoid loops.                                                                                                                                                                          | Workflow logic                                                                                         |
| Slack OAuth2 token must have chat permissions to post messages to channels.                                                                                                                                                                                                                                                               | Slack node authentication                                                                               |
| OpenAI prompt can be customized to generate different styles of replies or summaries.                                                                                                                                                                                                                                                     | AI customization                                                                                       |
| DeepL API key required for translation; alternative translation services can be integrated if needed with adapted node.                                                                                                                                                                                                                   | Translation service flexibility                                                                         |
| For more info on n8n node configurations and OAuth2 setup, visit official n8n docs: https://docs.n8n.io/                                                                                                                                                                                                                                  | Official documentation                                                                                  |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---