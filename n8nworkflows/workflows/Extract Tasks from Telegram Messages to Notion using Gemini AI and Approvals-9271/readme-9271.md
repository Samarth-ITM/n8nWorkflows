Extract Tasks from Telegram Messages to Notion using Gemini AI and Approvals

https://n8nworkflows.xyz/workflows/extract-tasks-from-telegram-messages-to-notion-using-gemini-ai-and-approvals-9271


# Extract Tasks from Telegram Messages to Notion using Gemini AI and Approvals

### 1. Workflow Overview

This workflow automates the extraction of task details from incoming Telegram messages and creates corresponding tasks in Notion after user approval. It is designed for users who want to streamline task management by converting chat messages into structured Notion tasks with minimal manual effort. The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Triggered by a new Telegram message in a specified chat.
- **1.2 AI Extraction:** Uses Google Gemini AI to extract task name and due date from the Telegram message text.
- **1.3 User Approval:** Sends the extracted task details back to the user on Telegram, waiting for approval or decline.
- **1.4 Conditional Task Creation or Notification:** Based on user approval, either creates a Notion task or sends a decline notification.
- **1.5 Notifications:** Sends Telegram confirmations post task creation or decline.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Listens for new messages from a specific Telegram chat to start the workflow.

**Nodes Involved:**  
- Telegram New Message Trigger  
- Sticky Note (Introductory)

**Node Details:**

- **Telegram New Message Trigger**  
  - *Type:* Telegram Trigger (webhook-based)  
  - *Configuration:* Listens for "message" updates; restricted to a configured chat ID (`YOUR_TELEGRAM_CHAT_ID`) to limit scope.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Emits incoming Telegram message JSON including sender and message text.  
  - *Credentials:* Telegram API credentials (bot token) stored securely in n8n.  
  - *Potential Failures:* Invalid chat ID, Telegram API downtime, webhook misconfiguration.  
  - *Notes:* The webhook ID is unique and used internally by n8n to receive Telegram updates.

- **Sticky Note (Start Note)**  
  - *Type:* Sticky Note (documentation aid)  
  - *Content:* Describes the trigger action in human-readable form.  
  - *No inputs or outputs.*

---

#### 1.2 AI Extraction

**Overview:**  
Processes the Telegram message text with Google Gemini AI (via Langchain node) to extract structured task information: Task Name (title-cased) and Task Due Date.

**Nodes Involved:**  
- Google Gemini Chat Model  
- AI Extract: TaskName & TaskDue  
- Sticky Notes (explaining AI extraction)

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini AI (LM Chat) node  
  - *Configuration:* Uses "models/gemini-2.5-flash-lite" model; minimal options set.  
  - *Credentials:* Google Palm / Gemini API key configured in n8n credentials.  
  - *Inputs:* Receives message text for processing.  
  - *Outputs:* Provides AI-generated chat completion response.  
  - *Edge Cases:* API limits, connectivity issues, malformed input text, credential expiration.

- **AI Extract: TaskName & TaskDue**  
  - *Type:* Langchain Information Extractor  
  - *Configuration:* Extracts two attributes:  
    - TaskName (required, title case)  
    - TaskDue (date type)  
  - *Input:* Receives text from Telegram message via expression `{{$json.message.text}}`.  
  - *Output:* JSON with extracted `TaskName` and `TaskDue` fields.  
  - *Dependencies:* Uses output from Google Gemini node as language model.  
  - *Failure Modes:* Extraction failure if message text is ambiguous or lacks task info; AI response format mismatch.

- **Sticky Notes (AI Extraction Description)**  
  - *Content:* Explains AI role and flexibility to switch models.

---

#### 1.3 User Approval

**Overview:**  
Sends the extracted task details back to Telegram user and waits interactively for approval or decline.

**Nodes Involved:**  
- Send message and wait for response (Approve/Decline)  
- Sticky Note (User approval explanation)

**Node Details:**

- **Send message and wait for response (Approve/Decline)**  
  - *Type:* Telegram node (sendAndWait operation)  
  - *Configuration:*  
    - Sends message with extracted Task Name and Due Date to the sender ID obtained from the initial trigger (`$('Telegram New Message Trigger').item.json.message.from.id`).  
    - Approval options configured with "Approve" and "Decline" buttons (double approval type).  
  - *Inputs:* Task extraction output fields for message construction.  
  - *Outputs:* Waits for user response, outputs approval boolean in `data.approved`.  
  - *Credentials:* Telegram bot API credentials.  
  - *Potential Failures:* User does not respond; Telegram API issues; message formatting errors.

- **Sticky Note (Approval Explanation)**  
  - *Content:* Describes the wait for user decision and options.

---

#### 1.4 Conditional Task Creation or Notification

**Overview:**  
Checks user approval status; if approved, creates a Notion task; if declined, sends a notification.

**Nodes Involved:**  
- Approval Check (If Approved?)  
- Notion: Create Task (Page)  
- Notify: Declined - No Task (Telegram)  
- Sticky Notes (Conditional logic and decline notification)

**Node Details:**

- **Approval Check (If Approved?)**  
  - *Type:* If node (version 2.2)  
  - *Configuration:* Checks if `data.approved` is strictly boolean `true`.  
  - *Inputs:* Approval response from Telegram node.  
  - *Outputs:* Two outputs:  
    - Output 1 for approved (true) branch  
    - Output 2 for declined (false) branch  
  - *Failures:* Expression evaluation errors if response format changes.

- **Notion: Create Task (Page)**  
  - *Type:* Notion node  
  - *Configuration:*  
    - Creates a new page in specified Notion database (`YOUR_NOTION_DATABASE_ID`).  
    - Sets the Title property using `TaskName` extracted by AI.  
    - Sets the Date property using `TaskDue` extracted by AI, without time component.  
  - *Inputs:* Approval Check node's approved output.  
  - *Credentials:* Notion API integration credentials.  
  - *Edge Cases:* Notion API rate limits, invalid database ID, property mismatch, date format errors.

- **Notify: Declined - No Task (Telegram)**  
  - *Type:* Telegram node  
  - *Configuration:* Sends a message informing user that the task was declined and not created.  
  - *Inputs:* Approval Check node's declined output branch.  
  - *Credentials:* Telegram API credentials.  
  - *Failures:* Telegram API errors or invalid user ID.

- **Sticky Notes (Conditional & Decline Notification)**  
  - *Content:* Explains conditional routing and decline message purpose.

---

#### 1.5 Notifications

**Overview:**  
Sends confirmation messages to the user after task creation.

**Nodes Involved:**  
- Notify: Task Created (Telegram)  
- Sticky Note (Task creation notification)

**Node Details:**

- **Notify: Task Created (Telegram)**  
  - *Type:* Telegram node  
  - *Configuration:* Sends a confirmation message "✅ Task created in Notion." to the original sender.  
  - *Inputs:* Triggered after successful Notion task creation.  
  - *Credentials:* Telegram API credentials.  
  - *Failures:* Telegram API or message delivery failures.

- **Sticky Note (Notification Explanation)**  
  - *Content:* Describes the notification after task creation.

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                          | Input Node(s)                           | Output Node(s)                             | Sticky Note                                                                                     |
|-----------------------------------|--------------------------------------|----------------------------------------|---------------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram New Message Trigger       | Telegram Trigger                     | Start workflow on new Telegram message | None                                  | AI Extract: TaskName & TaskDue              | ## ⚡ Starts on new Telegram message. Triggers the workflow when a new Telegram message arrives. |
| Google Gemini Chat Model           | Langchain Google Gemini LM Chat      | AI language model for task extraction  | AI Extract: TaskName & TaskDue (ai_languageModel) | AI Extract: TaskName & TaskDue               | ## 🧠 Uses AI (Gemini) to return TaskName & TaskDue. Describes AI model usage and flexibility.  |
| AI Extract: TaskName & TaskDue     | Langchain Information Extractor      | Extract Task Name and Due Date          | Telegram New Message Trigger           | Send message and wait for response (Approve/Decline) |                                                                                                |
| Send message and wait for response (Approve/Decline) | Telegram (sendAndWait)               | Sends extracted task; waits for approval/decline | AI Extract: TaskName & TaskDue          | Approval Check (If Approved?)                | ## 📩 Sends extracted task; waits for Approve or Decline.                                      |
| Approval Check (If Approved?)      | If Node                             | Branch workflow based on user approval | Send message and wait for response    | Notion: Create Task (Page), Notify: Declined - No Task (Telegram) | ## ⚖️ If approved → create task. If declined → notify.                                       |
| Notion: Create Task (Page)         | Notion Node                         | Create a task in Notion database        | Approval Check (If Approved?)          | Notify: Task Created (Telegram)              | ## 📝 Create Notion Task. Creates task with extracted info after approval.                     |
| Notify: Task Created (Telegram)    | Telegram                            | Notify user of successful task creation | Notion: Create Task (Page)             | None                                       |                                                                                                |
| Notify: Declined - No Task (Telegram) | Telegram                            | Notify user of declined task             | Approval Check (If Approved?)          | None                                       | ## 📩 Decline Notification. Informs user task was declined and not created.                    |
| Sticky Note                       | Sticky Note                        | Documentation and explanation            | None                                  | None                                       | See individual notes for content.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram New Message Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates  
   - Additional Fields: Set `chatIds` to your Telegram chat ID (replace `YOUR_TELEGRAM_CHAT_ID`)  
   - Credentials: Link your Telegram bot credentials (OAuth2 or bot token)  
   - Position: Start node

2. **Add Google Gemini Chat Model node**  
   - Type: Langchain Google Gemini LM Chat Node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)  
   - Parameters: Select model `models/gemini-2.5-flash-lite`  
   - Credentials: Add Google Palm / Gemini API key in n8n credentials  
   - Connect: No direct connection needed here; used inside AI Extract node as language model

3. **Add AI Extract: TaskName & TaskDue node**  
   - Type: Langchain Information Extractor (`@n8n/n8n-nodes-langchain.informationExtractor`)  
   - Parameters:  
     - Text: Set to expression `={{ $json.message.text }}` from Telegram trigger  
     - Attributes to extract:  
       - TaskName (required, description: task name, title case)  
       - TaskDue (type: date, description: due date)  
     - Use Google Gemini Chat Model node as language model connection (`ai_languageModel` input)  
   - Connect: From Telegram New Message Trigger node main output to this node main input

4. **Add Telegram node to send message and wait for approval**  
   - Type: Telegram  
   - Operation: `sendAndWait`  
   - Parameters:  
     - Chat ID: Expression `={{ $('Telegram New Message Trigger').item.json.message.from.id }}`  
     - Message: Expression `"Task Name: {{ $json.output.TaskName }}\nDue Date: {{ $json.output.TaskDue }}"`  
     - Approval Options: Double approval type with buttons labeled "Approve" and "Decline"  
   - Credentials: Same Telegram bot credentials  
   - Connect: From AI Extract node main output

5. **Add If node for approval check**  
   - Type: If node (version 2.2)  
   - Parameters: Condition to check if `data.approved` equals boolean `true`  
   - Connect: From Telegram sendAndWait node main output

6. **Add Notion node to create task if approved**  
   - Type: Notion  
   - Parameters:  
     - Resource: Database page creation  
     - Database ID: Your Notion database ID (`YOUR_NOTION_DATABASE_ID`)  
     - Properties:  
       - Title set to expression `={{ $('AI Extract: TaskName & TaskDue').item.json.output.TaskName }}`  
       - Date property set to expression `={{ $('AI Extract: TaskName & TaskDue').item.json.output.TaskDue }}`, time excluded  
   - Credentials: Notion API credentials with correct integration permissions  
   - Connect: From If node's approved (true) output

7. **Add Telegram node to notify task created**  
   - Type: Telegram  
   - Parameters:  
     - Chat ID: Same expression as before  
     - Text: Static message `"✅ Task created in Notion."`  
     - Additional Fields: Disable attribution  
   - Credentials: Telegram bot credentials  
   - Connect: From Notion node main output

8. **Add Telegram node to notify task declined**  
   - Type: Telegram  
   - Parameters:  
     - Chat ID: Same expression  
     - Text: Static message `"❌ Task not created in Notion."`  
     - Additional Fields: Disable attribution  
   - Credentials: Telegram bot credentials  
   - Connect: From If node's declined (false) output

9. **Add Sticky Notes (optional)**  
   - Create sticky note nodes to document each logical block for clarity, including setup instructions, AI usage, approval process, and notifications.

10. **Configure credentials properly**  
    - Add Google Palm / Gemini API key as credential in n8n (named accordingly)  
    - Add Telegram bot token as Telegram API credential  
    - Add Notion integration credentials with access to the target database

11. **Replace all placeholders**  
    - Replace `YOUR_TELEGRAM_CHAT_ID` in Telegram trigger node  
    - Replace `YOUR_NOTION_DATABASE_ID` in Notion node  
    - Ensure all credential references point to your configured credentials

12. **Test the workflow**  
    - Send a sample message to your Telegram bot like `"Remind me to submit the report on 26th october 2025"`  
    - Approve the task when prompted  
    - Verify the task appears in your Notion database with correct title and due date

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Setup instructions require proper credential configuration for Google Gemini (Google Palm API), Notion integration, and Telegram bot. Replace all placeholders before deployment. Use n8n credential stores to securely manage API keys.                                                                                 | Sticky Note6 content in the workflow                                                                     |
| Use ISO date format (`YYYY-MM-DD`) for Notion date properties to avoid formatting issues.                                                                                                                                                                                                                              | Notion node configuration                                                                                 |
| Telegram sendAndWait operation supports interactive approval buttons; use this for simple user interaction without external UI.                                                                                                                                                                                        | Telegram node "Send message and wait for response"                                                      |
| Gemini AI model used here is "models/gemini-2.5-flash-lite" but can be switched to other supported models in Langchain nodes for different performance or cost profiles.                                                                                                                                               | Google Gemini Chat Model node                                                                             |
| For enhanced security and best practices, do not hardcode any API keys or tokens in node parameters; always use n8n’s credential store.                                                                                                                                                                               | Security notes in Sticky Note6                                                                            |
| Example test message format: `Remind me to submit the report on 26th october 2025` to verify extraction and task creation cycle.                                                                                                                                                                                       | Sticky Note6 example                                                                                       |

---

This document provides a detailed, stepwise, and comprehensive understanding of the "Telegram to Notion Task Automation" workflow. It enables users and automation agents to analyze, reproduce, or modify the process efficiently.