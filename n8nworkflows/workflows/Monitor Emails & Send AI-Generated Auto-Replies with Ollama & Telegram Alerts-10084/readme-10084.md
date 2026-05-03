Monitor Emails & Send AI-Generated Auto-Replies with Ollama & Telegram Alerts

https://n8nworkflows.xyz/workflows/monitor-emails---send-ai-generated-auto-replies-with-ollama---telegram-alerts-10084


# Monitor Emails & Send AI-Generated Auto-Replies with Ollama & Telegram Alerts

### 1. Workflow Overview

This workflow, titled **"AI-Powered IMAP Monitor with Telegram Alerts and Smart Auto-Reply"**, is designed for automated email management with integrated AI-generated responses. It continuously monitors an IMAP email inbox, sends instant Telegram notifications upon receiving new emails, filters out unwanted messages such as newsletters and no-reply addresses, and generates personalized auto-replies using a local large language model (LLM) via the Ollama API. The workflow concludes by notifying via Telegram that an auto-reply has been sent.

The workflow can be logically divided into the following blocks:

- **1.1 IMAP Email Monitoring:** Polls the IMAP mailbox for new emails.
- **1.2 Telegram Notification for Incoming Emails:** Sends immediate alerts about incoming emails to a Telegram chat.
- **1.3 Spam Filtering:** Applies rules to filter out no-reply addresses and newsletters to avoid responding to these.
- **1.4 AI-Powered Auto-Reply Generation:** Uses Ollama LLM to compose a personalized auto-reply based on the email content.
- **1.5 Sending Auto-Reply via SMTP:** Sends the generated reply to the original email sender.
- **1.6 Telegram Notification for Sent Auto-Reply:** Confirms via Telegram that the auto-reply has been sent successfully.

---

### 2. Block-by-Block Analysis

#### 1.1 IMAP Email Monitoring

- **Overview:**  
  Monitors the configured IMAP mailbox for new incoming emails in real-time or at poll intervals.

- **Nodes Involved:**  
  - Check Incoming Emails - IMAP (example: SOGo)

- **Node Details:**  
  - **Check Incoming Emails - IMAP (example: SOGo)**  
    - Type: Email Read IMAP  
    - Role: Polls the IMAP mailbox to fetch new emails.  
    - Configuration: Uses IMAP credentials (host, port, username, password, SSL/TLS enabled).  
    - Input: None (trigger node).  
    - Output: Email metadata and content in JSON format.  
    - Edge Cases: IMAP connection failures, authentication errors, mailbox empty or delayed email delivery.  
    - Sticky Notes: Describes IMAP credential requirements and polling behavior.

#### 1.2 Telegram Notification for Incoming Emails

- **Overview:**  
  Sends an immediate Telegram message notifying about the newly received email, including sender, subject, and date/time.

- **Nodes Involved:**  
  - Send Notification from Incoming Email

- **Node Details:**  
  - **Send Notification from Incoming Email**  
    - Type: Telegram node  
    - Role: Sends a formatted Telegram notification to a specific chat ID.  
    - Configuration: Uses a Telegram API credential with a bot token; chat ID must be updated to the recipient's chat.  
    - Key Expressions:  
      - Message includes sender (`{{ $json.from }}`), subject (`{{ $json.subject }}`), and date (`{{ $json.date }}`).  
    - Input: From IMAP email node.  
    - Output: Passes data to spam filtering node.  
    - Edge Cases: Telegram API failures, incorrect chat ID, message formatting errors.  
    - Sticky Notes: Instructions on Telegram setup and chat ID configuration.

#### 1.3 Spam Filtering

- **Overview:**  
  Filters out emails that should not receive an auto-reply, specifically those from addresses containing "noreply" or "no-reply" and subjects containing "newsletter."

- **Nodes Involved:**  
  - Dedicate Filtering As No-Response  
  - No Operation

- **Node Details:**  
  - **Dedicate Filtering As No-Response (If node)**  
    - Type: If node  
    - Role: Applies string-based conditions to detect no-reply addresses and newsletter subjects.  
    - Configuration: Checks that the sender's email address does NOT contain "noreply" or "no-reply", and subject does NOT contain "newsletter" (case-insensitive).  
    - Input: From Telegram notification node.  
    - Output: True path leads to AI auto-reply generation; false path leads to No Operation node.  
    - Edge Cases: Email address parsing errors, subject field missing or formatted differently.  
    - Sticky Notes: Explains filtering logic and customization advice.  
  - **No Operation**  
    - Type: NoOp node  
    - Role: Terminates workflow for filtered-out emails.  
    - Input: From filter node false branch.  
    - Output: None.  

#### 1.4 AI-Powered Auto-Reply Generation

- **Overview:**  
  Uses a large language model (LLM) hosted via Ollama API to generate a short, personalized auto-reply based on the email content and identified topic.

- **Nodes Involved:**  
  - Ollama Model  
  - Basic LLM Chain

- **Node Details:**  
  - **Ollama Model**  
    - Type: Ollama LLM node (Langchain)  
    - Role: Interfaces with the Ollama LLM to process prompts and generate text.  
    - Configuration: Uses model "llama3.1" with 4096 token context window. Requires Ollama API credential.  
    - Input: Flow from filter node true path.  
    - Output: Provides language model to the Basic LLM Chain node.  
    - Edge Cases: API connectivity issues, model not installed or accessible, token limit exceedance.  
    - Sticky Notes: Details about Ollama model setup and credential use.  
  - **Basic LLM Chain**  
    - Type: Chain LLM node (Langchain)  
    - Role: Sends a prompt to the LLM to generate an email reply text.  
    - Configuration:  
      - Prompt instructs to identify the main email topic in 2-4 words, then generate a polite auto-reply thanking the sender, mentioning the topic, and promising a personal follow-up.  
      - Uses the email plain text content as input.  
      - Returns only the generated reply text without extra explanations.  
    - Input: From Ollama Model node (ai_languageModel output).  
    - Output: Provides generated reply text for SMTP sending.  
    - Edge Cases: Prompt parsing errors, empty or malformed email body, LLM response failures.  
    - Sticky Notes: Explains prompt engineering and output expectations.

#### 1.5 Sending Auto-Reply via SMTP

- **Overview:**  
  Sends the AI-generated auto-reply to the original email sender using SMTP.

- **Nodes Involved:**  
  - Send Auto-Response in SMTP (example POSTFIX)

- **Node Details:**  
  - **Send Auto-Response in SMTP (example POSTFIX)**  
    - Type: Email Send SMTP node  
    - Role: Sends the auto-reply email.  
    - Configuration:  
      - Uses SMTP credentials (server, port, authentication, secure connection).  
      - Email subject prefixed with "Re:" followed by the original email subject.  
      - "To" address is the original sender's email.  
      - "From" address is the email monitored by IMAP (your address).  
      - Email body is the generated text from Basic LLM Chain node.  
      - Allows unauthorized certificates (configurable).  
    - Input: From Basic LLM Chain node.  
    - Output: Passes data to Telegram notification node for sent confirmation.  
    - Edge Cases: SMTP connection errors, authentication failures, invalid email addresses, blocked ports.  
    - Sticky Notes: Details SMTP credential setup and recommended security settings.

#### 1.6 Telegram Notification for Sent Auto-Reply

- **Overview:**  
  Sends a confirmation message to Telegram indicating that the auto-reply was sent, including the original email details and AI-generated response.

- **Nodes Involved:**  
  - Send Notification from Response

- **Node Details:**  
  - **Send Notification from Response**  
    - Type: Telegram node  
    - Role: Sends a notification that the auto-reply has been sent.  
    - Configuration:  
      - Sends message including sender, subject, date/time, and the AI-generated response text.  
      - Uses the same Telegram API credential and chat ID as the incoming email notification.  
    - Input: From SMTP send node.  
    - Output: End of workflow.  
    - Edge Cases: Telegram API failures, message formatting errors, incorrect chat ID.  
    - Sticky Notes: Explains confirmation message content.

---

### 3. Summary Table

| Node Name                                | Node Type                       | Functional Role                                | Input Node(s)                              | Output Node(s)                             | Sticky Note                                                                                     |
|-----------------------------------------|--------------------------------|-----------------------------------------------|--------------------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------|
| Check Incoming Emails - IMAP (example: SOGo) | Email Read IMAP                 | Polls IMAP mailbox for new emails             | None                                       | Send Notification from Incoming Email      | IMAP configuration details, credential requirements, polling intervals                         |
| Send Notification from Incoming Email   | Telegram                       | Sends Telegram notification of new email      | Check Incoming Emails - IMAP                | Dedicate Filtering As No-Response          | Telegram notification setup and chat ID instructions                                           |
| Dedicate Filtering As No-Response       | If                            | Filters out no-reply/newsletter emails         | Send Notification from Incoming Email       | Basic LLM Chain (true), No Operation (false) | Spam filtering logic and customization advice                                                  |
| No Operation                            | NoOp                          | Terminates workflow for filtered emails        | Dedicate Filtering As No-Response (false)   | None                                       |                                                                                                |
| Ollama Model                           | Ollama LLM (Langchain)         | Interfaces with Ollama LLM for response generation | Dedicate Filtering As No-Response (true)    | Basic LLM Chain                            | Ollama model setup, credential, and model download instructions                                |
| Basic LLM Chain                        | Chain LLM (Langchain)          | Generates AI auto-reply text based on email    | Ollama Model                               | Send Auto-Response in SMTP                  | Prompt engineering details for consistent auto-reply format                                   |
| Send Auto-Response in SMTP (example POSTFIX) | Email Send SMTP                | Sends AI-generated auto-reply via SMTP         | Basic LLM Chain                            | Send Notification from Response             | SMTP credential setup and security recommendations                                             |
| Send Notification from Response         | Telegram                       | Sends confirmation Telegram message after reply | Send Auto-Response in SMTP                  | None                                       | Confirmation message content including AI response                                            |
| No Operation                            | NoOp                          | Placeholder for emails filtered out             | Dedicate Filtering As No-Response (false)   | None                                       |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create IMAP Email Read Node:**  
   - Add "Email Read IMAP" node.  
   - Configure with your IMAP credentials (host, port 993 with SSL/TLS, username, password).  
   - Set polling intervals as desired to check mailbox regularly.

2. **Add Telegram Notification for Incoming Email:**  
   - Add "Telegram" node.  
   - Configure Telegram API credentials using bot token from @BotFather.  
   - Set chat ID to your Telegram chat/group ID (replace `-1234567890123`).  
   - Set message text to include sender (`{{ $json.from }}`), subject (`{{ $json.subject }}`), and date (`{{ $json.date }}`).  
   - Connect IMAP node output to this Telegram node input.

3. **Add Spam Filtering If Node:**  
   - Add "If" node named "Dedicate Filtering As No-Response".  
   - Configure conditions:  
     - Sender email address (`{{ $json.from.value[0].address }}`) does NOT contain "noreply"  
     - Sender email address does NOT contain "no-reply"  
     - Subject (`{{ $json.subject.toLowerCase() }}`) does NOT contain "newsletter"  
   - Connect Telegram notification node output to this If node input.

4. **Add No Operation Node:**  
   - Add a "No Operation" node.  
   - Connect the false output branch of the If node to this NoOp node. This ends processing for filtered emails.

5. **Add Ollama Model Node:**  
   - Add Ollama LLM node (from Langchain nodes).  
   - Configure with Ollama API credentials.  
   - Set model to "llama3.1" with a context window of 4096 tokens.  
   - Connect the true output branch of the If node to this Ollama node.

6. **Add Basic LLM Chain Node:**  
   - Add a Chain LLM node (Langchain).  
   - Configure prompt text to:  
     ```
     Read the original email and identify its main topic or subject in 2-4 words maximum.

     ORIGINAL EMAIL:
     {{ $('IMAP Email').item.json.textPlain }}

     Return only this response, filling in the [TOPIC]:

     Dear Correspondent! 
     Thank you for your message regarding [TOPIC]. 
     I will respond with a personal message as soon as possible. 
     Have a nice day!

     RULES:
     - Replace [TOPIC] with 2-4 words summarizing the email subject
     - Keep it simple and generic (e.g., "your inquiry", "the project", "your request")
     - Return ONLY the email text above
     - Do NOT add explanations or extra content
     ```  
   - Set message values with role "You are an email response assistant."  
   - Connect Ollama node output (ai_languageModel) to this LLM chain node.

7. **Add SMTP Send Node:**  
   - Add "Email Send SMTP" node.  
   - Configure SMTP credentials (host, port 587 or 465, username, password, secure connection).  
   - Set "To Email" to original sender: `={{ $('Check Incoming Emails - IMAP (example: SOGo)').item.json.from }}`  
   - Set "From Email" to your monitored email address: `={{ $('Check Incoming Emails - IMAP (example: SOGo)').item.json.to }}`  
   - Set email subject to: `=Re: {{ $('Check Incoming Emails - IMAP (example: SOGo)').item.json.subject }}`  
   - Set email body text to the output from Basic LLM Chain node: `={{ $json.text }}`  
   - Connect Basic LLM Chain output to this SMTP node.

8. **Add Telegram Notification for Sent Auto-Reply:**  
   - Add another "Telegram" node.  
   - Use same Telegram API credentials and chat ID as before.  
   - Configure message text to include:  
     - Sender (`{{ $json.from }}`)  
     - Subject (`{{ $json.subject }}`)  
     - Date (`{{ $json.date }}`)  
     - AI-generated response (`{{ $('Basic LLM Chain').item.json.text }}`)  
   - Connect SMTP send node output to this Telegram notification node.

9. **Activate and Test:**  
   - Activate the workflow.  
   - Test by sending an email to the monitored inbox and verify Telegram notifications and auto-reply delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| IMAP polling requires app-specific passwords and SSL/TLS enabled for secure and reliable email access.                                   | Sticky Note1                                                                                       |
| Telegram bot token must be obtained from @BotFather, and chat ID updated to your target chat/group.                                     | Sticky Note2                                                                                       |
| Spam filtering logic is customizable; currently excludes noreply/no-reply addresses and newsletters to prevent auto-reply spam loops.   | Sticky Note3                                                                                       |
| SMTP credentials should use secure ports (587 STARTTLS or 465 SSL) and app-specific passwords if supported by mail server for security.  | Sticky Note4                                                                                       |
| Ollama LLM model "llama3.1" requires local or remote Ollama instance with model downloaded (`ollama pull llama3.1`).                     | Sticky Note7                                                                                       |
| Prompt engineering ensures AI generates concise and polite auto-replies focused on the main topic without extraneous content.            | Sticky Note6                                                                                       |
| Telegram notifications provide real-time monitoring of workflow activity and success confirmation of auto-replies.                      | Sticky Note8                                                                                       |
| Use app-specific passwords for email and API credentials to enhance security.                                                             | Sticky Note5                                                                                       |

---

**Disclaimer:** The text provided is solely derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.