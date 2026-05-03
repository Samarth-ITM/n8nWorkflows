Reduce Meeting No-Shows with Gemini AI, Email & WhatsApp Reminders for Calendly

https://n8nworkflows.xyz/workflows/reduce-meeting-no-shows-with-gemini-ai--email---whatsapp-reminders-for-calendly-10262


# Reduce Meeting No-Shows with Gemini AI, Email & WhatsApp Reminders for Calendly

---

## 1. Workflow Overview

This workflow automates the reduction of no-shows for meetings scheduled via Calendly by sending customized reminders and follow-up requests through email and WhatsApp. It is designed for hosts who want to ensure better attendance rates by timely nudging invitees with relevant information and clarifications.

**Target use cases:**  
- Meeting organizers using Calendly who want to automate attendee reminders.  
- Businesses leveraging AI to parse meeting details from notification emails.  
- Teams requiring multi-channel reminders (email + WhatsApp) with customizable timing.

**Logical blocks and flow:**

- **1.1 Input Reception:**  
  Listens for incoming meeting notification emails (e.g., from Calendly) via IMAP.

- **1.2 AI Data Extraction:**  
  Uses Google Gemini AI to parse the notification email and extract structured meeting and invitee information.

- **1.3 Clarification Request Decision:**  
  Checks if the invitee provided sufficient information; if not, sends an email requesting clarification.

- **1.4 Waiting Time Calculation:**  
  Computes how long to wait before sending reminders (24 hours and 1 hour before the meeting).

- **1.5 Timing Decision & Wait:**  
  Routes flow based on calculated wait times and pauses execution accordingly to send reminders at optimal times.

- **1.6 Reminder Dispatch:**  
  Sends personalized reminder emails and WhatsApp messages shortly before the meeting.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block monitors an IMAP email inbox for incoming meeting notification emails, which trigger the workflow.

**Nodes Involved:**  
- Notification received

**Node Details:**  

- **Notification received**  
  - Type: Email Read (IMAP) node, version 2.1  
  - Role: Polls an IMAP account for new emails, triggering the workflow when a new meeting notification arrives.  
  - Configuration: Uses IMAP credentials configured externally; no filters beyond default.  
  - Inputs: None (trigger node)  
  - Outputs: Raw email content including plain text  
  - Edge cases:  
    - IMAP authentication failure  
    - Connectivity/timeouts to email server  
    - Email with unexpected format (may affect downstream AI extraction)  
  - Sticky notes:  
    - “Purpose: This node is the trigger. It monitors an IMAP email account for new meeting notification emails (e.g., from Calendly). Ensure the IMAP account credential is properly set up.”

---

### 2.2 AI Data Extraction

**Overview:**  
Analyzes the raw email text using a Google Gemini AI model to extract structured meeting details and attendee information in JSON format.

**Nodes Involved:**  
- extract from email (AI Agent)  
- LLM model (Google Gemini)  
- Calculator (AI tool)  
- Structured Output Parser

**Node Details:**  

- **extract from email**  
  - Type: AI Agent (Langchain Agent), version 2.2  
  - Role: Applies a large language model with a carefully designed prompt to extract key meeting data fields.  
  - Configuration:  
    - Prompt instructs to extract invitee_name, invitee_email, phone_number, meeting_date (YYYY-MM-DD), meeting_time (HH:mm), meeting_link, meeting_goal, needs_more_info (boolean), additional_info.  
    - Time zone fixed as Eastern European Time (EET).  
  - Input: Plain text email content from “Notification received” node.  
  - Output: Structured JSON object with meeting and invitee data.  
  - Expressions: Uses {{$json.textPlain}} to feed email text into the prompt.  
  - Credentials: Google Gemini API credential for the underlying LLM model node.  
  - Edge cases:  
    - AI model returning incomplete or malformed JSON  
    - API rate limits or authentication errors  
    - Unexpected email formats reducing extraction accuracy  

- **LLM model**  
  - Type: Chat Google Gemini, version 1  
  - Role: Provides the language model engine backing the AI Agent node.  
  - Configuration: Default options; credential must be set up for Google Gemini (PaLM).  
  - Input: Connected from AI Agent node as backend.  
  - Output: Model response to prompt.  
  - Edge cases: API authentication errors, timeouts.  

- **Calculator**  
  - Type: AI tool Calculator, version 1  
  - Role: Supports the AI Agent node for any required numeric or calculation tasks during extraction.  
  - Input/Output: Connected internally to AI Agent workflow.  

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured, version 1.3  
  - Role: Validates and parses AI output against a predefined JSON schema example to ensure proper structure.  
  - Schema: Defines types and fields expected from AI extraction.  
  - Input: AI Agent output.  
  - Output: Clean structured JSON data.  
  - Edge cases: Parsing errors if AI output deviates from schema.  

- Sticky notes:  
  - Explains the purpose and setup of this AI data extraction block, credential requirements, and prompt specifications.

---

### 2.3 Clarification Request Decision

**Overview:**  
Evaluates if the extracted data indicates the need for additional information from the invitee. If true, sends an email requesting clarification.

**Nodes Involved:**  
- Check if More Info is Needed (If node)  
- Send Clarification request (Email Send)

**Node Details:**  

- **Check if More Info is Needed**  
  - Type: If node, version 1  
  - Role: Checks boolean field `needs_more_info` in AI extraction result.  
  - Condition: True path if `needs_more_info` === true, else false.  
  - Input: Output JSON from “extract from email” node.  
  - Output: True branch triggers email request, false branch continues workflow.  
  - Edge cases: Missing field or invalid boolean causing condition failure.  

- **Send Clarification request**  
  - Type: Email Send, version 2  
  - Role: Sends an email to the invitee requesting additional details based on extracted `additional_info`.  
  - Configuration:  
    - To: invitee_email from JSON.  
    - From: [HOST_EMAIL] placeholder to be replaced by user.  
    - Subject: “Quick question before our strategy call”.  
    - Body: Personalized using invitee_name and additional_info from JSON.  
  - Input: True branch from If node.  
  - Output: None (end of branch).  
  - Edge cases: Email sending failures (SMTP, authentication), invalid email address.  
  - Sticky note: Instructions to replace placeholders [HOST_EMAIL], [HOST_NAME].  

---

### 2.4 Waiting Time Calculation

**Overview:**  
Calculates how many hours to wait before sending reminders at 24 hours and 1 hour before the meeting, based on email reception time and meeting datetime (in EET).

**Nodes Involved:**  
- Calculate Waiting Time (Code node)

**Node Details:**  

- **Calculate Waiting Time**  
  - Type: Code node (JavaScript), version 2  
  - Role: Parses email received date and extracted meeting date/time, calculates difference in hours, then determines wait times for 24h and 1h reminders.  
  - Key logic:  
    - Parses original email received timestamp.  
    - Parses meeting_date and meeting_time strings.  
    - Combines into a Date object with EET timezone offset (+02:00).  
    - Computes hours difference between meeting and email received.  
    - Sets wait_24h to (diffHrs - 24) if > 24h, else 0.  
    - Sets wait_1h to (diffHrs - 1) if < 24h, else 0.  
    - Ensures both wait times are non-null integers.  
  - Input: Output from “extract from email” and “Notification received” nodes.  
  - Output: JSON with `wait_24h` and `wait_1h` fields.  
  - Edge cases:  
    - Incorrect date/time formats causing parsing errors.  
    - Negative wait times handled by zero fallback.  
  - Sticky note explains purpose and code details.  

---

### 2.5 Timing Decision & Wait

**Overview:**  
Determines which reminder timing path to follow based on calculated wait times and pauses execution for the appropriate duration before sending reminders.

**Nodes Involved:**  
- When to send (If node)  
- Wait to send 24h before meeting (Wait node)  
- Wait to send 1h before meeting (Wait node)  
- Wait to send 1h before meeting after 24h (Wait node)

**Node Details:**  

- **When to send**  
  - Type: If node, version 2.2  
  - Role: Evaluates if `wait_24h` equals 0 to decide which wait path to follow:  
    - True branch: meeting less than or equal to 1 hour away -> Wait 1h reminder immediately.  
    - False branch: meeting more than 1 hour away -> Wait for 24h reminder then 1h reminder.  
  - Input: Output from “Calculate Waiting Time” node.  
  - Output: True branch connects to “Wait to send 1h before meeting” node; False branch connects to both “Wait to send 24h before meeting” and “Wait to send 1h before meeting after 24h” nodes.  

- **Wait to send 24h before meeting**  
  - Type: Wait node, version 1  
  - Role: Pauses workflow execution for `wait_24h` hours before sending 24h reminder.  
  - Input: False branch from “When to send”.  
  - Parameter: `amount` in hours, expression uses `$('Calculate Waiting Time').item.json.wait_24h`.  
  - Output: Leads to sending reminders after wait.

- **Wait to send 1h before meeting**  
  - Type: Wait node, version 1  
  - Role: Pauses workflow execution for `wait_1h` hours before sending 1h reminder if meeting is near.  
  - Input: True branch from “When to send”.  
  - Parameter: `amount` in hours, expression uses `{{$json.wait_24h}}` (should be zero in this path).  
  - Output: Leads to sending reminders.

- **Wait to send 1h before meeting after 24h**  
  - Type: Wait node, version 1  
  - Role: After the 24h reminder is sent, further waits `wait_1h` hours before sending the 1h reminder.  
  - Input: False branch from “When to send”.  
  - Parameter: `amount` is sum of `wait_1h` + `wait_24h` from “Calculate Waiting Time”.  
  - Output: Leads to sending reminders.

- Edge cases:  
  - Wait node limitations on max wait time (n8n may reset after certain period).  
  - Negative or zero wait times handled by zero fallback.  
- Sticky note clarifies the flow logic and wait node configuration.

---

### 2.6 Reminder Dispatch

**Overview:**  
Sends personalized reminder emails and WhatsApp messages shortly before the meeting to reduce no-shows.

**Nodes Involved:**  
- Send Reminder Email (Email Send)  
- Send WhatsApp Reminder (Twilio)

**Node Details:**  

- **Send Reminder Email**  
  - Type: Email Send node, version 2  
  - Role: Sends a reminder email to the meeting invitee with meeting details.  
  - Configuration:  
    - To: Uses invitee_email extracted from AI agent output.  
    - From: Placeholder [HOST_EMAIL] to be replaced.  
    - Subject: Dynamically changes to “Reminder: Your AI Strategy Call Tomorrow” if 24h reminder, or “...in 1 hour” if 1h reminder, using the ternary expression `{{ $json.type === '24h' ? 'Tomorrow' : 'in 1 hour' }}`.  
    - Body: Personalized with invitee_name, meeting time/date, meeting link, and host signature ([HOST_NAME]).  
  - Input: From any of the wait nodes after waiting period.  
  - Output: None (end of reminder branch).  
  - Edge cases: SMTP errors, invalid email addresses, missing fields.  

- **Send WhatsApp Reminder**  
  - Type: Twilio node, version 1  
  - Role: Sends WhatsApp message reminder to invitee's phone number.  
  - Configuration:  
    - To: Formatted WhatsApp number from extracted `phone_number`, cleaned of spaces and plus signs, prefixed with “whatsapp:”.  
    - From: Twilio WhatsApp sender number placeholder [TWILIO_WHATSAPP_FROM].  
    - Message: Personalized text with invitee_name, meeting date/time, meeting link, and host signature ([HOST_NAME]).  
  - Input: Same as “Send Reminder Email” node.  
  - Output: None.  
  - Edge cases: Twilio API credential failure, phone number formatting errors, message sending errors.  
  - Sticky note mentions required credential setup and placeholders replacement.

---

## 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                            | Input Node(s)               | Output Node(s)                                                | Sticky Note                                                                                                                     |
|----------------------------------|----------------------------|--------------------------------------------|-----------------------------|--------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Notification received             | Email Read IMAP             | Trigger: New meeting notification email   | None                        | extract from email                                           | Purpose: This node is the trigger. It monitors an IMAP email account for new meeting notification emails (e.g., from Calendly). Ensure the IMAP account credential is properly set up. |
| extract from email                | AI Agent                   | Extract structured meeting data via AI    | Notification received        | Check if More Info is Needed, Calculate Waiting Time         | Purpose: Uses an LLM to extract invitee_name, invitee_email, phone_number, meeting_date, meeting_time, meeting_link, meeting_goal, needs_more_info, additional_info. Requires Google Gemini API credential. |
| LLM model                        | Chat Google Gemini          | AI model engine for extraction             | extract from email (backend) | extract from email (AI output)                               | Included in AI data extraction block; requires Google Gemini (PaLM) API credential.                                            |
| Calculator                      | Langchain Calculator        | Supports AI agent with calculations        | extract from email (backend) | extract from email (AI output)                               | Included in AI data extraction block.                                                                                          |
| Structured Output Parser         | Output Parser               | Validates and parses AI output              | LLM model                   | extract from email                                           | Ensures AI output matches JSON schema.                                                                                        |
| Check if More Info is Needed     | If node                    | Checks if clarification email needed       | extract from email           | Send Clarification request (true), Calculate Waiting Time (false) | Checks `needs_more_info` boolean field.                                                                                        |
| Send Clarification request       | Email Send                 | Sends email requesting more info            | Check if More Info is Needed | None                                                        | Replace [HOST_EMAIL], [HOST_NAME] placeholders with actual info.                                                             |
| Calculate Waiting Time           | Code node (JS)             | Calculates wait times for reminders         | extract from email, Notification received | When to send                                              | Calculates wait_24h and wait_1h based on meeting time and email receipt time.                                                 |
| When to send                    | If node                    | Determines reminder timing path              | Calculate Waiting Time       | Wait to send 1h before meeting (true), Wait to send 24h before meeting and Wait to send 1h before meeting after 24h (false) | Routes based on wait_24h == 0 condition.                                                                                       |
| Wait to send 1h before meeting   | Wait node                  | Waits before sending 1h reminder             | When to send (true branch)   | Send Reminder Email, Send WhatsApp Reminder                   | Wait period equals wait_1h.                                                                                                    |
| Wait to send 24h before meeting  | Wait node                  | Waits before sending 24h reminder            | When to send (false branch)  | Send Reminder Email, Send WhatsApp Reminder                   | Wait period equals wait_24h.                                                                                                   |
| Wait to send 1h before meeting after 24h | Wait node          | Waits after 24h reminder before 1h reminder | When to send (false branch)  | Send Reminder Email, Send WhatsApp Reminder                   | Wait period equals wait_1h + wait_24h.                                                                                        |
| Send Reminder Email              | Email Send                 | Sends meeting reminder email                 | Wait nodes                  | None                                                        | Dynamic subject line with ternary expression; replace [HOST_EMAIL], [HOST_NAME].                                              |
| Send WhatsApp Reminder           | Twilio (WhatsApp)          | Sends WhatsApp meeting reminder              | Wait nodes                  | None                                                        | Requires Twilio WhatsApp credential; replace [TWILIO_WHATSAPP_FROM] and [HOST_NAME].                                            |
| Sticky Note                     | Sticky Note                | Workflow overview and explanation            | None                        | None                                                        | Describes overall workflow purpose and logic.                                                                                 |
| Sticky Note1                    | Sticky Note                | Explains Notification received node          | None                        | None                                                        | Explains trigger node and credential requirements.                                                                            |
| Sticky Note2                    | Sticky Note                | Explains AI Data Extraction block             | None                        | None                                                        | Details prompt, model, and credential setup for AI extraction.                                                                |
| Sticky Note3                    | Sticky Note                | Explains Clarification Request block          | None                        | None                                                        | Instructions for placeholders and node purpose.                                                                               |
| Sticky Note4                    | Sticky Note                | Explains Waiting Time Calculation              | None                        | None                                                        | Details calculation logic for reminder wait times.                                                                            |
| Sticky Note5                    | Sticky Note                | Explains Timing Decision & Wait block           | None                        | None                                                        | Clarifies routing logic between wait nodes.                                                                                   |
| Sticky Note6                    | Sticky Note                | Explains Reminder Dispatch block                | None                        | None                                                        | Instructions for placeholders and Twilio setup.                                                                               |
| Sticky Note7                    | Sticky Note                | Lists required credentials and placeholders     | None                        | None                                                        | IMAP, Google Gemini API, Twilio credentials; replace placeholders [HOST_EMAIL], [HOST_NAME], [TWILIO_WHATSAPP_FROM].            |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Notification received” node**  
   - Type: Email Read (IMAP)  
   - Configure IMAP credentials to monitor inbox for new meeting notification emails.  
   - Set “Post Process Action” to “nothing” (default).  
   - This node acts as the workflow trigger.

2. **Create “extract from email” node**  
   - Type: AI Agent (Langchain Agent)  
   - Configure prompt to extract the following JSON fields from raw email text:  
     - invitee_name, invitee_email, phone_number, meeting_date (YYYY-MM-DD), meeting_time (HH:mm), meeting_link, meeting_goal, needs_more_info (boolean), additional_info.  
   - Use expression to insert plain text email content: `{{$json.textPlain}}`  
   - Connect “Notification received” output to this node’s input.  
   - Set the AI backend model by adding “LLM model” node and linking it as the AI language model backend.  
   - Ensure Google Gemini (PaLM) API credential is configured and selected in the LLM model node.

3. **Create “LLM model” node**  
   - Type: Chat Google Gemini  
   - No special parameters, ensure Google Gemini API credential is set.  
   - Connect as backend AI model to “extract from email” node.

4. **Create “Calculator” node**  
   - Type: AI Tool Calculator  
   - No parameters needed.  
   - Connect as AI tool to “extract from email” node.

5. **Create “Structured Output Parser” node**  
   - Type: Langchain Output Parser Structured  
   - Configure JSON schema example matching the expected output fields and types.  
   - Connect output of “LLM model” node to this parser node, then back into “extract from email” node as output parser.

6. **Create “Check if More Info is Needed” node**  
   - Type: If node  
   - Condition: Check if `{{$json.output.needs_more_info}}` equals `true`.  
   - Connect “extract from email” output to this node.

7. **Create “Send Clarification request” node**  
   - Type: Email Send  
   - Configure:  
     - To: `{{$json.invitee_email}}`  
     - From: Replace with actual host email ([HOST_EMAIL])  
     - Subject: “Quick question before our strategy call”  
     - Body: Use template with `{{$json.invitee_name}}` and `{{$json.additional_info}}` variables.  
   - Connect true branch of “Check if More Info is Needed” node to this node.

8. **Create “Calculate Waiting Time” node**  
   - Type: Code node (JavaScript)  
   - Paste provided JS code to parse email date, meeting date/time, and calculate `wait_24h` and `wait_1h`.  
   - Connect false branch of “Check if More Info is Needed” node to this node.

9. **Create “When to send” node**  
   - Type: If node  
   - Condition: Check if `{{$json.wait_24h}}` equals 0.  
   - Connect “Calculate Waiting Time” node output to this node.

10. **Create “Wait to send 1h before meeting” node**  
    - Type: Wait node  
    - Set “Amount” (hours) parameter as `{{$json.wait_1h}}` (or `{{$json.wait_24h}}` as per actual expression).  
    - Connect true branch of “When to send” node to this node.

11. **Create “Wait to send 24h before meeting” node**  
    - Type: Wait node  
    - Set “Amount” (hours) parameter as `{{$json.wait_24h}}`.  
    - Connect false branch of “When to send” node to this node.

12. **Create “Wait to send 1h before meeting after 24h” node**  
    - Type: Wait node  
    - Set “Amount” (hours) parameter as `{{$json.wait_1h}} + {{$json.wait_24h}}`.  
    - Connect false branch of “When to send” node also to this node.

13. **Create “Send Reminder Email” node**  
    - Type: Email Send  
    - Configure:  
      - To: Use expression `={{ $('extract from email').item.json.output.invitee_email }}`  
      - From: Replace with actual host email ([HOST_EMAIL])  
      - Subject: Use dynamic expression:  
        `Reminder: Your AI Strategy Call {{ $json.type === '24h' ? 'Tomorrow' : 'in 1 hour' }}`  
      - Body: Include personalized message with invitee_name, meeting_time, meeting_date, meeting_link, and host signature ([HOST_NAME]).  
    - Connect outputs of all wait nodes to this node.

14. **Create “Send WhatsApp Reminder” node**  
    - Type: Twilio node (WhatsApp)  
    - Configure:  
      - To: Use expression to format number:  
        `whatsapp:={{ $('extract from email').item.json.output.phone_number.replace(/\s+/g,'').replace('+','') }}`  
      - From: Twilio WhatsApp sender number placeholder ([TWILIO_WHATSAPP_FROM])  
      - Message: Personalized reminder similar to email, referencing invitee_name, meeting date/time, meeting link, and host name ([HOST_NAME]).  
    - Connect outputs of all wait nodes to this node.

15. **Credential Setup**  
    - Configure IMAP credentials for “Notification received”.  
    - Setup Google Gemini (PaLM) API credentials for “LLM model”.  
    - Setup Twilio credentials for WhatsApp node.  
    - Replace all placeholders `[HOST_EMAIL]`, `[HOST_NAME]`, and `[TWILIO_WHATSAPP_FROM]` with actual values.

16. **Testing and Validation**  
    - Test by sending a sample Calendly notification email to the IMAP inbox.  
    - Confirm AI extraction accuracy, wait time calculations, and timely dispatch of reminders.  
    - Monitor logs for errors (email sending, Twilio messaging, API calls).

---

## 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow automates multi-channel reminders (email + WhatsApp) and clarification requests to reduce no-shows. | Overview of the entire workflow purpose.                                                                     |
| Replace placeholders [HOST_EMAIL], [HOST_NAME], and [TWILIO_WHATSAPP_FROM] in emails and Twilio node. | Critical for personalization and correct sender identification.                                             |
| Requires Google Gemini (PaLM) API credentials for AI extraction; can be replaced with other supported LLM providers if desired. | AI credentials setup note.                                                                                    |
| Ensure IMAP email credential is connected correctly to receive Calendly notification emails.       | Trigger node setup.                                                                                           |
| Twilio WhatsApp messaging requires a verified Twilio account with WhatsApp sender number configured. | Twilio setup and WhatsApp channel readiness.                                                                 |
| Subject line dynamically changes between “Tomorrow” and “in 1 hour” based on reminder timing.      | Expression in “Send Reminder Email” node subject.                                                            |
| The workflow assumes meeting times in Eastern European Time (EET); adjust code if your timezone differs. | Timezone awareness for date/time parsing.                                                                    |
| Wait nodes may have limitations on maximum delay; consider workflow execution environment constraints. | Possible edge case with long waiting periods in n8n.                                                        |

---

**Disclaimer:** The text and configuration described are derived exclusively from an automated n8n workflow for lawful and public data handling. All nodes and credentials should be configured respecting security and privacy policies.