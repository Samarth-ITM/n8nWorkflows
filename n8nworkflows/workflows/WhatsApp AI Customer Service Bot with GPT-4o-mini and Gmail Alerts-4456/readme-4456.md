WhatsApp AI Customer Service Bot with GPT-4o-mini and Gmail Alerts

https://n8nworkflows.xyz/workflows/whatsapp-ai-customer-service-bot-with-gpt-4o-mini-and-gmail-alerts-4456


# WhatsApp AI Customer Service Bot with GPT-4o-mini and Gmail Alerts

### 1. Workflow Overview

This workflow implements a **WhatsApp AI Customer Service Bot** enhanced by GPT-4o-mini and integrated with Gmail alerts for monitoring. It automates customer interactions via WhatsApp, processes text inputs with an AI language model, sends appropriate replies through WhatsApp, and generates Gmail notifications for alerting purposes.

The workflow can be logically broken down into the following blocks:

- **1.1 Input Reception:** Receives WhatsApp messages triggered by user submissions.
- **1.2 Message Type Routing:** Determines if the incoming message is text and directs flow accordingly.
- **1.3 AI Processing:** Uses GPT-4o-mini (via LangChain OpenAI node) with conversation memory to generate AI-based responses.
- **1.4 Response Dispatch:** Sends AI-generated replies back to WhatsApp users.
- **1.5 Fallback Handling:** Handles non-text messages or AI failures by sending fallback responses and notifying via Gmail.
- **1.6 Notification:** Sets and sends Gmail notifications based on WhatsApp interactions and fallback events.
- **1.7 No Operation:** A terminal node ensuring workflow stability post-notification.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming WhatsApp messages as triggers and initiates the workflow.

- **Nodes Involved:**  
  - Input Submissions (WhatsApp Trigger)

- **Node Details:**

  - **Input Submissions**  
    - *Type:* WhatsApp Trigger  
    - *Role:* Entry point for incoming WhatsApp messages.  
    - *Configuration:* Uses a webhook ID assigned by n8n to listen for WhatsApp messages. No additional filters configured.  
    - *Inputs:* External WhatsApp message events.  
    - *Outputs:* Forward message data to the next node (Signpost).  
    - *Edge Cases:* Network interruptions, invalid webhook setup, or unrecognized message formats could cause trigger failures.

#### 1.2 Message Type Routing

- **Overview:**  
  Evaluates incoming messages to decide if they are text or non-text and routes processing to AI or fallback handlers.

- **Nodes Involved:**  
  - Signpost (If node)  
  - Is Text Message? (If node)

- **Node Details:**

  - **Signpost**  
    - *Type:* If node (Boolean branching)  
    - *Role:* Initial routing decision, likely based on message content or metadata.  
    - *Configuration:* Condition(s) unspecified here; presumably checks message type or presence of text.  
    - *Inputs:* From Input Submissions.  
    - *Outputs:* True branch to AI Agent; False branch not specified but leads to fallback indirectly.  
    - *Failure Modes:* Expression errors if message data missing or malformed.

  - **Is Text Message?**  
    - *Type:* If node  
    - *Role:* Determines whether the message is textual.  
    - *Configuration:* Condition checks if message content is text (specific expression not provided).  
    - *Inputs:* From Signpost.  
    - *Outputs:* True branch to AI Agent, False branch to Fallback Text.  
    - *Edge Cases:* Non-text messages, media messages, or unsupported formats cause fallback. Expression evaluation errors possible if data missing.

#### 1.3 AI Processing

- **Overview:**  
  Processes textual messages through GPT-4o-mini AI with conversation memory to generate intelligent replies.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - OpenAI Chat Model (LangChain OpenAI)  
  - Simple Memory (LangChain Memory Buffer Window)

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Coordinates AI response generation using configured language model and memory.  
    - *Configuration:* Default parameters; connects to OpenAI Chat Model and Simple Memory nodes as AI language model and memory inputs.  
    - *Inputs:* From Signpost and Is Text Message? (true branch).  
    - *Outputs:* Sends AI-generated response to WhatsApp Reply Sender.  
    - *Edge Cases:* API rate limits, invalid credentials, or unexpected AI errors may cause failures.

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model node  
    - *Role:* Provides GPT-4o-mini based language model interface.  
    - *Configuration:* No explicit parameters shown; assumed API key configured via credentials.  
    - *Inputs:* AI Agent node as language model input.  
    - *Outputs:* AI-generated chat completions.  
    - *Failure Modes:* Authentication errors, API timeouts.

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains conversational context in a sliding window buffer for AI agent.  
    - *Configuration:* Default buffer size and parameters assumed.  
    - *Inputs:* Connected as AI memory input to AI Agent.  
    - *Outputs:* Context data to AI Agent to influence response generation.  
    - *Edge Cases:* Memory overflow or empty context may affect response relevance.

#### 1.4 Response Dispatch

- **Overview:**  
  Sends AI-generated replies back to WhatsApp users and triggers notification setup.

- **Nodes Involved:**  
  - WhatsApp Reply Sender (WhatsApp)  
  - Gmail Output Set (Set node)

- **Node Details:**

  - **WhatsApp Reply Sender**  
    - *Type:* WhatsApp node  
    - *Role:* Sends messages back to WhatsApp via webhook integration.  
    - *Configuration:* Uses webhook ID for WhatsApp outbound messages; no additional parameters shown.  
    - *Inputs:* From AI Agent.  
    - *Outputs:* Triggers Gmail Output Set node.  
    - *Failure Modes:* WhatsApp API errors, message formatting issues.

  - **Gmail Output Set**  
    - *Type:* Set node  
    - *Role:* Prepares data for Gmail notification.  
    - *Configuration:* No explicit parameters shown; likely sets email fields (subject, body) based on WhatsApp reply data.  
    - *Inputs:* From WhatsApp Reply Sender.  
    - *Outputs:* Leads to Send Gmail Notifications node.  
    - *Edge Cases:* Missing or malformed email data.

#### 1.5 Fallback Handling

- **Overview:**  
  Handles cases where incoming messages are non-text or AI fails, sending fallback WhatsApp responses and alert emails.

- **Nodes Involved:**  
  - Fallback Text (WhatsApp)  
  - Send Gmail Notifications (Gmail node)  
  - No Operation, do nothing (NoOp)

- **Node Details:**

  - **Fallback Text**  
    - *Type:* WhatsApp node  
    - *Role:* Sends fallback or error messages to users when input is unsupported or AI cannot respond.  
    - *Configuration:* Uses webhook ID for outbound WhatsApp messages; fallback message content likely static or preset.  
    - *Inputs:* From Is Text Message? (false branch).  
    - *Outputs:* Triggers Send Gmail Notifications.  
    - *Edge Cases:* WhatsApp API failures or webhook misconfiguration.

  - **Send Gmail Notifications**  
    - *Type:* Gmail node  
    - *Role:* Sends email alerts for fallback events or system notifications.  
    - *Configuration:* Requires Gmail OAuth2 credentials; email fields set by Gmail Output Set or fallback chain.  
    - *Inputs:* From Fallback Text or Gmail Output Set.  
    - *Outputs:* Leads to No Operation node.  
    - *Failure Modes:* Authentication errors, quota limits.

  - **No Operation, do nothing**  
    - *Type:* NoOp node  
    - *Role:* Terminal endpoint to gracefully end workflow executions following notifications.  
    - *Inputs:* From Send Gmail Notifications.  
    - *Outputs:* None.  
    - *Notes:* Prevents dangling active executions, improving stability.

#### 1.6 Notification

- **Overview:**  
  Orchestrates the preparation and sending of Gmail alerts based on WhatsApp interactions and fallback responses.

- **Nodes Involved:**  
  - Gmail Output Set (Set node)  
  - Send Gmail Notifications (Gmail node)

- **Node Details:**  
  Covered above in 1.4 and 1.5 sections.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                            | Input Node(s)              | Output Node(s)              | Sticky Note                     |
|-----------------------|----------------------------------|-------------------------------------------|----------------------------|-----------------------------|--------------------------------|
| Input Submissions     | WhatsApp Trigger                  | Entry point for incoming WhatsApp messages | —                          | Signpost                    |                                |
| Signpost              | If node                          | Routes messages based on type or content  | Input Submissions           | AI Agent, Is Text Message?  |                                |
| Is Text Message?      | If node                          | Checks if message is text and routes flow | Signpost                   | AI Agent (true), Fallback Text (false) |                                |
| AI Agent              | LangChain Agent                  | Processes message to generate AI response | Signpost, Is Text Message?  | WhatsApp Reply Sender       |                                |
| OpenAI Chat Model     | LangChain OpenAI Chat Model      | GPT-4o-mini for generating AI completions | AI Agent                   | AI Agent                    |                                |
| Simple Memory         | LangChain Memory Buffer Window   | Maintains conversation context            | —                          | AI Agent                    |                                |
| WhatsApp Reply Sender | WhatsApp node                    | Sends AI responses back to WhatsApp users | AI Agent                   | Gmail Output Set            |                                |
| Gmail Output Set      | Set node                        | Prepares data for Gmail notifications     | WhatsApp Reply Sender       | Send Gmail Notifications    |                                |
| Send Gmail Notifications | Gmail node                    | Sends email alerts for notifications      | Gmail Output Set, Fallback Text | No Operation, do nothing    |                                |
| Fallback Text         | WhatsApp node                    | Sends fallback WhatsApp messages           | Is Text Message? (false)    | Send Gmail Notifications    |                                |
| No Operation, do nothing | NoOp node                     | Terminates workflow execution gracefully   | Send Gmail Notifications    | —                           |                                |
| Sticky Note           | Sticky Note                     | Comments or notes for developers/users    | —                          | —                           | Empty content in this workflow |
| Sticky Note1          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note2          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note3          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note4          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note5          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note6          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note7          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note8          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note9          | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |
| Sticky Note10         | Sticky Note                     | Comments or notes                         | —                          | —                           | Empty content in this workflow |

---

### 4. Reproducing the Workflow from Scratch

1. **Create 'Input Submissions' node**  
   - Type: WhatsApp Trigger  
   - Configuration: Add webhook with appropriate WhatsApp integration credentials. No filters necessary.  
   - Position: Start of workflow.

2. **Create 'Signpost' node**  
   - Type: If node  
   - Configuration: Add condition to determine message type or presence of text. For example, check if `{{$json["message"]}}` exists or contains text.  
   - Connect Input Submissions → Signpost.

3. **Create 'Is Text Message?' node**  
   - Type: If node  
   - Configuration: Set condition to check if message content type is text. For example, `{{$json["messageType"]}} === "text"`.  
   - Connect Signpost → Is Text Message?.

4. **Create 'AI Agent' node**  
   - Type: LangChain Agent  
   - Configuration: Use default settings.  
   - Connect Signpost (true branch) and Is Text Message? (true branch) → AI Agent.

5. **Create 'OpenAI Chat Model' node**  
   - Type: LangChain OpenAI Chat Model  
   - Configuration: Configure with OpenAI credentials (API key). Model set to GPT-4o-mini or equivalent.  
   - Connect OpenAI Chat Model → AI Agent (as AI language model input).

6. **Create 'Simple Memory' node**  
   - Type: LangChain Memory Buffer Window  
   - Configuration: Default buffer size (e.g., 5 messages).  
   - Connect Simple Memory → AI Agent (as AI memory input).

7. **Create 'WhatsApp Reply Sender' node**  
   - Type: WhatsApp node  
   - Configuration: Use outbound WhatsApp webhook credentials and webhook ID for sending messages.  
   - Connect AI Agent → WhatsApp Reply Sender.

8. **Create 'Gmail Output Set' node**  
   - Type: Set node  
   - Configuration: Map WhatsApp reply data to email fields, such as subject, body, recipient.  
   - Connect WhatsApp Reply Sender → Gmail Output Set.

9. **Create 'Send Gmail Notifications' node**  
   - Type: Gmail node  
   - Configuration: Use Gmail OAuth2 credentials. Set 'From', 'To', 'Subject', and 'Body' fields using data from Gmail Output Set or fallback.  
   - Connect Gmail Output Set → Send Gmail Notifications.

10. **Create 'Fallback Text' node**  
    - Type: WhatsApp node  
    - Configuration: Use outbound WhatsApp webhook with a predefined fallback message (e.g., "Sorry, I can only process text messages currently.").  
    - Connect Is Text Message? (false branch) → Fallback Text.

11. **Connect Fallback Text → Send Gmail Notifications**  
    - Ensures fallback events also trigger email alerts.

12. **Create 'No Operation, do nothing' node**  
    - Type: NoOp node  
    - Configuration: No parameters needed.  
    - Connect Send Gmail Notifications → No Operation.

13. **Verify all connections** to ensure flow from input to AI processing or fallback, response sending, and notifications.

14. **Optional:** Add Sticky Notes for documentation or reminders within the workflow canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                        |
|--------------------------------------------------------------------------------------------------|-------------------------------------|
| Workflow uses WhatsApp webhook integration requiring valid WhatsApp API credentials.              | WhatsApp API documentation          |
| OpenAI API key with access to GPT-4o-mini model required for AI Agent functionality.               | https://platform.openai.com/docs/models/gpt-4o-mini |
| Gmail node requires OAuth2 credentials with permission to send emails via Gmail API.              | https://developers.google.com/gmail/api |
| No explicit error handling nodes implemented; consider adding retry or error catch nodes for production use. | Best practice in n8n workflows      |
| The workflow assumes messages are primarily text; media or unsupported types trigger fallback.     | Workflow design assumption           |

---

**Disclaimer:** The provided content is generated exclusively from an automated workflow created with n8n, respecting all content policies and legal constraints. All data processed is public and lawful.