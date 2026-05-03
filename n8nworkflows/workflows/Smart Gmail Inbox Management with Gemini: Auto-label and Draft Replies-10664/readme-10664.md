Smart Gmail Inbox Management with Gemini: Auto-label and Draft Replies

https://n8nworkflows.xyz/workflows/smart-gmail-inbox-management-with-gemini--auto-label-and-draft-replies-10664


# Smart Gmail Inbox Management with Gemini: Auto-label and Draft Replies

### 1. Workflow Overview

This workflow, titled **"Smart Gmail Inbox Management with Gemini: Auto-label and Draft Replies"**, automates the classification and initial response drafting for incoming Gmail messages using AI, specifically Google’s Gemini language model. Its purpose is to help users keep their inbox organized by applying appropriate Gmail labels based on email content and context, and to assist in composing draft replies for emails that require a response—without sending them automatically.

The workflow is structured into two primary logical blocks:

- **1.1 Email Reception and Classification**: Detects new emails, gathers context from email threads and past sent messages, and uses AI to classify emails with the correct Gmail label or mark them for deletion.

- **1.2 Drafting Replies for Required Responses**: For emails flagged as requiring a response, generates a short, professional draft reply using AI, incorporating calendar availability if scheduling is involved, and creates a Gmail draft for user review.

Supporting nodes handle Gmail label management (removing and adding labels), trashing emails marked for deletion, and parsing AI outputs into structured JSON.


---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Classification

**Overview:**  
This block monitors the Gmail inbox for new messages, removes existing default or irrelevant labels, collects contextual email thread and sent message data, and uses the Gemini AI model to classify the email. Based on AI output, the workflow either trashes the email or applies the appropriate label.

**Nodes Involved:**  
- Gmail Trigger  
- RemoveLabel  
- GetThread  
- CheckSent  
- AI Email Classifier  
- Structured Output Parser  
- IF Delete  
- TrashMessage  
- AddLabel  
- Sticky Note (explains classification logic)

**Node Details:**

- **Gmail Trigger**  
  - *Type:* Gmail Trigger  
  - *Role:* Detects new incoming emails every minute.  
  - *Config:* No filters applied; polling mode set to every minute.  
  - *Credentials:* Google OAuth2.  
  - *Input:* Gmail inbox messages.  
  - *Output:* Emits new email data with metadata including sender, subject, snippet, labels, message ID, thread ID.  
  - *Failures:* OAuth token expiration, Gmail API rate limits, network timeouts.

- **RemoveLabel**  
  - *Type:* Gmail node (modify labels)  
  - *Role:* Removes default or pre-existing labels (INBOX, IMPORTANT, SOCIAL, etc.) to prepare for fresh labeling.  
  - *Config:* Removes multiple standard Gmail category labels by IDs from the current email.  
  - *Input:* Message ID from Gmail Trigger.  
  - *Output:* Passes email data forward.  
  - *Failures:* Invalid message ID, authorization failure.

- **GetThread**  
  - *Type:* GmailTool node  
  - *Role:* Fetches all messages from the sender’s email address to provide conversation history context.  
  - *Config:* Query set to "from: [sender’s email]"; returns all matching messages.  
  - *Input:* Email sender from Gmail Trigger.  
  - *Output:* Full thread messages.  
  - *Failures:* Query syntax errors, API errors, empty results.

- **CheckSent**  
  - *Type:* GmailTool node  
  - *Role:* Checks if the user has previously sent messages to this sender, indicating ongoing conversations.  
  - *Config:* Query set to "to: [sender’s email]"; returns all matching messages sent by user.  
  - *Input:* Email sender from Gmail Trigger.  
  - *Output:* Past sent messages data.  
  - *Failures:* Similar to GetThread.

- **AI Email Classifier**  
  - *Type:* LangChain Agent node (Google Gemini)  
  - *Role:* Uses AI to classify the email into one Gmail label or mark it for deletion, based on email content and context from GetThread and CheckSent. Also determines if a reply is required.  
  - *Config:* Prompt instructs AI to analyze email details, thread history, and sent history; respond strictly with JSON containing label, labelID, message ID, and requiresResponse boolean.  
  - *Key Expressions:* Injects Gmail Trigger data and AI tools outputs as context.  
  - *Credentials:* Google Palm API (Gemini).  
  - *Input:* Email data and AI tool outputs.  
  - *Output:* Classification JSON.  
  - *Failures:* AI API limits, parsing errors, malformed JSON from AI, latency.

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser  
  - *Role:* Parses AI Email Classifier’s raw AI output into structured JSON for downstream processing.  
  - *Config:* Defines JSON schema with keys: label, labelID, id, requiresResponse.  
  - *Input:* AI Email Classifier raw output.  
  - *Output:* Parsed JSON with classification details.  
  - *Failures:* Parsing errors if AI output deviates from schema.

- **IF Delete**  
  - *Type:* Conditional (If) node  
  - *Role:* Checks if AI classified the email label as "DELETE" (meaning it should be trashed).  
  - *Config:* Condition: output.label equals "DELETE".  
  - *Input:* Parsed classification JSON.  
  - *Output:* Routes to TrashMessage if true; else to AddLabel.  
  - *Failures:* Missing or malformed label value.

- **TrashMessage**  
  - *Type:* Gmail node (modify labels)  
  - *Role:* Adds the TRASH label to the message to move it to trash.  
  - *Config:* Adds label ID "TRASH" to the current message ID.  
  - *Input:* Email ID from Gmail Trigger.  
  - *Output:* Ends flow for deleted emails.  
  - *Failures:* Authorization issues, invalid message ID.

- **AddLabel**  
  - *Type:* Gmail node (modify labels)  
  - *Role:* Applies the AI-selected label to the message.  
  - *Config:* Adds label ID from AI output to the current message ID.  
  - *Input:* Email ID from Gmail Trigger and labelID from AI output.  
  - *Output:* Proceeds to check if a response is required.  
  - *Failures:* Invalid label ID, API errors.

- **Sticky Note (Classification Explanation)**  
  - *Type:* Sticky Note  
  - *Content:* Explains that Gemini AI classifies incoming emails using conversation context from GetThread and CheckSent, outputting structured JSON with label and response flags.

---

#### 2.2 Drafting Replies for Required Responses

**Overview:**  
This block activates if the email classification indicates a reply is needed. It gathers conversation context and calendar availability, uses Gemini AI to generate a short, polite draft reply, and saves this draft in Gmail for user review.

**Nodes Involved:**  
- IF Response  
- GetThread1  
- CheckCalendar  
- AI Draft Maker  
- Structured Output Parser1  
- Create a draft  
- Sticky Note (explains drafting logic)

**Node Details:**

- **IF Response**  
  - *Type:* Conditional (If) node  
  - *Role:* Checks if the classification output indicates `requiresResponse` is true.  
  - *Config:* Condition: classification’s requiresResponse equals true.  
  - *Input:* AI classification JSON.  
  - *Output:* Routes to AI Draft Maker if true; else ends.  
  - *Failures:* Missing requiresResponse flag.

- **GetThread1**  
  - *Type:* GmailTool node  
  - *Role:* Fetches the full email thread for the sender to provide complete context for drafting.  
  - *Config:* Query "from: [sender’s email]", returns all.  
  - *Input:* Email sender from Gmail Trigger.  
  - *Output:* Full thread messages for drafting context.  
  - *Failures:* Same as GetThread.

- **CheckCalendar**  
  - *Type:* Google CalendarTool node  
  - *Role:* Retrieves calendar availability for the next 7 days within the user’s local time zone, to propose meeting times in the draft if scheduling is detected.  
  - *Config:* Time range from AI-provided start and end times, user’s timezone and calendar email specified in parameters.  
  - *Input:* Time range from AI prompt overrides.  
  - *Output:* Available calendar slots.  
  - *Credentials:* Google Calendar OAuth2.  
  - *Failures:* Credential expiration, calendar API errors, timezone misconfiguration.

- **AI Draft Maker**  
  - *Type:* LangChain Agent node (Google Gemini)  
  - *Role:* Generates a short, friendly, professional draft reply based on full email thread, calendar availability, and original message data.  
  - *Config:* Prompt instructs AI to draft a reply in the sender’s language, propose meeting times if relevant, keep tone friendly, and avoid sending messages automatically. Output is JSON with `"draft"` and `"notes"`.  
  - *Credentials:* Google Palm API (Gemini).  
  - *Input:* Email data, GetThread1 messages, CheckCalendar availability.  
  - *Output:* Draft text and optional notes.  
  - *Failures:* AI API limits, malformed output, latency.

- **Structured Output Parser1**  
  - *Type:* LangChain Output Parser  
  - *Role:* Parses AI Draft Maker’s output into JSON with keys `"draft"` and `"notes"`.  
  - *Input:* Raw AI output.  
  - *Output:* Parsed draft JSON.  
  - *Failures:* Parsing errors if AI output is invalid.

- **Create a draft**  
  - *Type:* Gmail node (create draft)  
  - *Role:* Saves the generated draft reply into Gmail, linked to the original message thread, addressed to the original sender.  
  - *Config:* Draft subject prepended with "Re: ", thread ID set to original thread, recipient set to original sender, message body from AI draft output.  
  - *Input:* Draft JSON, Gmail Trigger original email data.  
  - *Credentials:* Google OAuth2.  
  - *Output:* Draft saved in Gmail; no automatic sending.  
  - *Failures:* Invalid thread ID, message formatting issues, authorization errors.

- **Sticky Note (Drafting Explanation)**  
  - *Type:* Sticky Note  
  - *Content:* Explains that this block creates draft replies with AI, incorporating conversation context and calendar availability, outputting draft text and reviewer notes, and never sends messages automatically.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                               | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                         |
|------------------------|----------------------------------|-----------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger          | Gmail Trigger                    | Detects new incoming emails                    | -                       | RemoveLabel             |                                                                                                                     |
| RemoveLabel            | Gmail (modify labels)            | Removes default/irrelevant labels              | Gmail Trigger           | AI Email Classifier     |                                                                                                                     |
| GetThread              | GmailTool                       | Retrieves sender’s full email thread           | AI Email Classifier (tool) | AI Email Classifier     |                                                                                                                     |
| CheckSent              | GmailTool                       | Retrieves past sent messages to sender         | AI Email Classifier (tool) | AI Email Classifier     |                                                                                                                     |
| AI Email Classifier    | LangChain Agent (Google Gemini) | Classifies email with label and response need  | RemoveLabel, GetThread, CheckSent | IF Delete              |                                                                                                                     |
| Structured Output Parser | LangChain Output Parser          | Parses AI classification output to JSON       | AI Email Classifier     | IF Delete               |                                                                                                                     |
| IF Delete              | If Node                         | Routes email to trash if label = DELETE        | Structured Output Parser | TrashMessage, AddLabel  |                                                                                                                     |
| TrashMessage           | Gmail (modify labels)            | Moves email to trash                            | IF Delete (true)         | -                       |                                                                                                                     |
| AddLabel               | Gmail (modify labels)            | Applies AI-selected label to email             | IF Delete (false)        | IF Response             |                                                                                                                     |
| IF Response            | If Node                         | Checks if reply is required                     | AddLabel                | AI Draft Maker          |                                                                                                                     |
| GetThread1             | GmailTool                       | Retrieves full thread for drafting context     | AI Draft Maker (tool)    | AI Draft Maker          |                                                                                                                     |
| CheckCalendar          | Google CalendarTool             | Retrieves calendar availability for scheduling | AI Draft Maker (tool)    | AI Draft Maker          |                                                                                                                     |
| AI Draft Maker         | LangChain Agent (Google Gemini) | Generates draft reply text and notes            | IF Response             | Create a draft          |                                                                                                                     |
| Structured Output Parser1 | LangChain Output Parser          | Parses draft AI output to JSON                  | AI Draft Maker          | Create a draft          |                                                                                                                     |
| Create a draft         | Gmail (create draft)             | Saves draft reply in Gmail                      | Structured Output Parser1 | -                       |                                                                                                                     |
| Sticky Note            | Sticky Note                     | Explains email classification process          | -                       | -                       | ## Classify emails: This step uses Gemini AI to categorize emails using Gmail labels and email thread context.       |
| Sticky Note1           | Sticky Note                     | Explains draft response process                 | -                       | -                       | ## Draft response: AI drafts short replies, incorporating calendar availability, never sends automatically.          |
| Sticky Note2           | Sticky Note                     | Overview and workflow benefits                   | -                       | -                       | # AI Gmail Manager: Automates classification and draft replies using Gemini AI; safe, label-based, and context-aware.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Gmail Trigger" node**  
   - Type: Gmail Trigger  
   - Poll interval: every minute  
   - Credentials: Google OAuth2 with appropriate Gmail scopes (read emails, modify labels).  
   - No filters applied.

2. **Create "RemoveLabel" node**  
   - Type: Gmail node (modify labels)  
   - Operation: removeLabels  
   - Label IDs to remove: `YELLOW_STAR, INBOX, IMPORTANT, CATEGORY_UPDATES, CATEGORY_SOCIAL, CATEGORY_PROMOTIONS, CATEGORY_PERSONAL, CATEGORY_FORUMS`  
   - Message ID: `={{ $json.id }}` (from Gmail Trigger)  
   - Connect Gmail Trigger main output → RemoveLabel input.

3. **Create "GetThread" node**  
   - Type: GmailTool node  
   - Operation: getAll  
   - Query: `from:{{ $('Gmail Trigger').item.json.From }}`  
   - Return all: true (can use AI override expression)  
   - Credentials: Google OAuth2  
   - Connect RemoveLabel ai_tool output → AI Email Classifier’s GetThread input.

4. **Create "CheckSent" node**  
   - Type: GmailTool node  
   - Operation: getAll  
   - Query: `to:{{ $('Gmail Trigger').item.json.From }}`  
   - Return all: true  
   - Credentials: Google OAuth2  
   - Connect RemoveLabel ai_tool output → AI Email Classifier’s CheckSent input.

5. **Create "AI Email Classifier" node**  
   - Type: LangChain Agent (Google Gemini)  
   - Credentials: Google Palm API (Gemini)  
   - Prompt: Detailed instructions to classify incoming email with JSON output including label, labelID, id, and requiresResponse flag. Include access to GetThread and CheckSent tools for context.  
   - Inputs: RemoveLabel main output, GetThread ai_tool output, CheckSent ai_tool output.  
   - Connect RemoveLabel main output → AI Email Classifier main input.  
   - Connect GetThread and CheckSent nodes as AI tools for context.

6. **Create "Structured Output Parser" node**  
   - Type: LangChain Output Parser  
   - JSON schema example:  
     ```json
     {
       "label": "Label name",
       "labelID": "Label ID",
       "id": "Email message ID",
       "requiresResponse": true
     }
     ```  
   - Connect AI Email Classifier ai_outputParser output → Structured Output Parser input.

7. **Create "IF Delete" node**  
   - Type: If node (version 2)  
   - Condition: `{{$json.output.label}}` equals `"DELETE"` (case sensitive)  
   - Connect Structured Output Parser main output → IF Delete input.

8. **Create "TrashMessage" node**  
   - Type: Gmail node (modify labels)  
   - Operation: addLabels  
   - Label IDs: ["TRASH"]  
   - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
   - Credentials: Google OAuth2  
   - Connect IF Delete True output → TrashMessage input.

9. **Create "AddLabel" node**  
   - Type: Gmail node (modify labels)  
   - Operation: addLabels  
   - Label IDs: `={{ $json.output.labelID }}` (from Structured Output Parser)  
   - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
   - Credentials: Google OAuth2  
   - Connect IF Delete False output → AddLabel input.

10. **Create "IF Response" node**  
    - Type: If node (version 2)  
    - Condition: `{{$json.output.requiresResponse}}` equals true  
    - Connect AddLabel main output → IF Response input.

11. **Create "GetThread1" node**  
    - Type: GmailTool node  
    - Operation: getAll  
    - Query: `from:{{ $('Gmail Trigger').item.json.From }}`  
    - Return all: true  
    - Credentials: Google OAuth2  
    - Connect IF Response ai_tool output → AI Draft Maker’s GetThread input.

12. **Create "CheckCalendar" node**  
    - Type: Google CalendarTool node  
    - Resource: calendar  
    - Timezone: Set your local timezone here (e.g., "Europe/Paris")  
    - Calendar: Your primary calendar email  
    - timeMin/timeMax: Set via AI prompt overrides or fixed window of next 7 days  
    - Credentials: Google Calendar OAuth2  
    - Connect IF Response ai_tool output → AI Draft Maker’s CheckCalendar input.

13. **Create "AI Draft Maker" node**  
    - Type: LangChain Agent (Google Gemini)  
    - Credentials: Google Palm API (Gemini)  
    - Prompt: Instruct AI to create a concise, friendly draft reply based on full thread, calendar availability if scheduling, and original email info. Output JSON with "draft" and "notes".  
    - Inputs: IF Response main output, GetThread1 and CheckCalendar as AI tools.  
    - Connect IF Response main output → AI Draft Maker main input.

14. **Create "Structured Output Parser1" node**  
    - Type: LangChain Output Parser  
    - JSON schema example:  
      ```json
      {
        "draft": "string",
        "notes": "string"
      }
      ```  
    - Connect AI Draft Maker ai_outputParser output → Structured Output Parser1 input.

15. **Create "Create a draft" node**  
    - Type: Gmail node (create draft)  
    - Resource: draft  
    - Message: `={{ $json.output.draft }}` (from Structured Output Parser1)  
    - Send To: `={{ $('Gmail Trigger').item.json.From }}`  
    - Thread ID: `={{ $('Gmail Trigger').item.json.threadId }}`  
    - Subject: `=Re: {{ $('Gmail Trigger').item.json.Subject }}`  
    - Credentials: Google OAuth2  
    - Connect Structured Output Parser1 main output → Create a draft input.

16. **Optional: Add Sticky Note nodes**  
    - Add explanatory sticky notes near classification and drafting blocks with content describing their respective logic and purpose.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow integrates Google Gemini (via LangChain) for AI classification and drafting in Gmail automation.                   | Requires valid Google Palm API credentials configured in n8n.                                                    |
| Gmail OAuth2 credential must have permissions to read emails, modify labels, and create drafts.                                   | Google Cloud project setup with Gmail API enabled.                                                                |
| Google Calendar OAuth2 credential must have calendar read access to propose meeting times in drafts.                             | Proper timezone configuration is critical for accurate scheduling suggestions.                                    |
| Drafts are saved in Gmail but never sent automatically, ensuring safe review before sending any reply.                           |                                                                                                                   |
| Workflow uses structured JSON output parsing to ensure reliable data flow between AI nodes and Gmail operations.                 |                                                                                                                   |
| Gemini AI prompt includes tools access (GetThread, CheckSent) to leverage full email context for better classification results. |                                                                                                                   |
| Sticky Notes in workflow explain key functional blocks and improve maintainability and handover.                                 |                                                                                                                   |

---

**Disclaimer:**  
The text above is an exclusive analysis of an n8n workflow automation designed for Gmail inbox management. It complies strictly with all content policies and handles only legal and public data.