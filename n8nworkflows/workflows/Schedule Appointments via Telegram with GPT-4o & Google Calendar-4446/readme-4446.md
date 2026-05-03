Schedule Appointments via Telegram with GPT-4o & Google Calendar

https://n8nworkflows.xyz/workflows/schedule-appointments-via-telegram-with-gpt-4o---google-calendar-4446


# Schedule Appointments via Telegram with GPT-4o & Google Calendar

---

### 1. Workflow Overview

This workflow automates appointment scheduling and cancellation via Telegram messages by leveraging GPT-4o (OpenAI) and Google Calendar. Users interact with a Telegram bot to request creating or deleting calendar events, and the workflow interprets these requests, manages the calendar accordingly, and sends confirmations back through Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user messages from Telegram.
- **1.2 AI Processing:** Uses GPT-4o with LangChain integration to interpret user intent and extract appointment details.
- **1.3 Calendar Management:** Creates or cancels events in Google Calendar based on AI output.
- **1.4 Confirmation Messaging:** Sends confirmation messages back to the user via Telegram.
- **1.5 Memory Management:** Maintains conversation context for coherent multi-turn interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming user messages from Telegram to initiate the appointment scheduling process.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Sticky Note (Telegram Message Received Trigger)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Trigger node for Telegram updates  
    - *Configuration:* Listens for "message" updates to capture all standard messages sent to the bot.  
    - *Key Expressions:* None (direct webhook trigger)  
    - *Connections:* Outputs to "AI Agent" node  
    - *Credentials:* Requires Telegram Bot API credentials (token from BotFather)  
    - *Potential Failures:* Webhook setup issues, invalid credentials, Telegram API downtime.

  - **Sticky Note (Telegram Message Received Trigger)**  
    - *Type:* Visual note for documentation  
    - *Content:* "Telegram Message Received Trigger"  
    - *Purpose:* Marks the entry point for incoming Telegram messages.

---

#### 2.2 AI Processing

- **Overview:**  
  Processes the user's message text with GPT-4o via LangChain to identify intent (book/cancel), extract appointment details (date, time, attendees), and prepare structured data for calendar operations.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Simple Memory (Buffer Window)  
  - Sticky Note (Telegram Message Received Trigger)

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain Agent node integrating OpenAI and tools  
    - *Configuration:*  
      - Input text set to the Telegram message text (`{{$json.message.text}}`).  
      - System message guides the AI to act as an appointment scheduling assistant with specific rules:  
        1. Use the current time reference (`{{$now}}`) for relative dates.  
        2. Assume all times in GMT+08 (Singapore Time).  
        3. Default event duration is 1 hour if unspecified.  
      - Prompt type: "define" to explicitly set system instructions.  
      - Uses Google Calendar tool integration (via node connections).  
    - *Connections:*  
      - Receives input from "Telegram Trigger".  
      - Outputs to "Telegram" node (for response).  
      - Connected to "Google Calendar1" as an AI tool node (for calendar actions).  
      - Uses "OpenAI Chat Model" as language model backend.  
      - Uses "Simple Memory" for session context.  
    - *Expressions:* Uses `$json.message.text` and `$now` for dynamic context.  
    - *Potential Failures:*  
      - OpenAI API errors (rate limits, invalid keys).  
      - Expression evaluation errors if Telegram message format changes.  
      - Timezone misinterpretation without explicit user input.  
      - Tool integration failures if Google Calendar credentials are invalid.

  - **OpenAI Chat Model**  
    - *Type:* Language Model node (GPT-4o-mini)  
    - *Configuration:*  
      - Model set to "gpt-4o-mini".  
      - No special options set.  
    - *Credentials:* Requires valid OpenAI API credentials.  
    - *Connections:* Linked as the AI language model for the "AI Agent".  
    - *Potential Failures:* API key invalidity, quota exceeded, network issues.

  - **Simple Memory**  
    - *Type:* Memory Buffer Window (conversation context)  
    - *Configuration:*  
      - Session key derived from Telegram chat ID (`{{$json.message.chat.id}}`) to maintain per-user session.  
    - *Connections:* Feeds memory to "AI Agent".  
    - *Potential Failures:* Memory overflow or corruption if sessions grow too large.

  - **Sticky Note (Telegram Message Received Trigger)**  
    - *Purpose:* Visual grouping and documentation of the AI processing block.

---

#### 2.3 Calendar Management

- **Overview:**  
  Based on AI agent output, this block interacts with Google Calendar to create or delete appointment events.

- **Nodes Involved:**  
  - Google Calendar1 (Google Calendar Tool)  
  - Sticky Note (Calendar Tool)

- **Node Details:**

  - **Google Calendar1**  
    - *Type:* Google Calendar Tool node for event management  
    - *Configuration:*  
      - Dynamic fields populated from AI outputs using `$fromAI()` function overrides:  
        - `start` and `end` datetime strings for the event duration.  
        - `summary` for event title.  
        - `attendees` list (email addresses).  
        - `useDefaultReminders` boolean flag.  
      - Calendar selected from a list (e.g., `"your_calendar@example.com"`).  
    - *Connections:*  
      - Connected as AI tool to "AI Agent" (called programmatically by agent).  
    - *Credentials:* Google Calendar OAuth2 required.  
    - *Potential Failures:*  
      - OAuth token expiration or invalidity.  
      - Calendar permission issues.  
      - Date/time parsing errors from AI output.  
      - API quota limits.

  - **Sticky Note (Calendar Tool)**  
    - *Purpose:* Marks the calendar interaction block.

---

#### 2.4 Confirmation Messaging

- **Overview:**  
  Sends a confirmation or status message back to the Telegram user based on the AI agent’s response after calendar operations.

- **Nodes Involved:**  
  - Telegram  
  - Sticky Note (Telegram Response)

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram node to send messages via bot API  
    - *Configuration:*  
      - Message text set from AI agent output (`{{$json.output}}`).  
      - Chat ID dynamically pulled from the Telegram Trigger node (`{{$('Telegram Trigger').item.json.message.chat.id}}`).  
    - *Connections:* Receives input from "AI Agent".  
    - *Credentials:* Telegram API credentials required.  
    - *Potential Failures:*  
      - Telegram API downtime or rate limits.  
      - Invalid chat ID or user blocking the bot.

  - **Sticky Note (Telegram Response)**  
    - *Purpose:* Marks the output messaging step.

---

#### 2.5 Memory Management

- **Overview:**  
  Maintains conversational context per user to allow multi-turn dialogue and coherent scheduling.

- **Nodes Involved:**  
  - Simple Memory (Buffer Window)

- **Node Details:**  
  Already described in AI Processing block.

---

### 3. Summary Table

| Node Name          | Node Type                                 | Functional Role                      | Input Node(s)       | Output Node(s)      | Sticky Note                             |
|--------------------|-------------------------------------------|------------------------------------|---------------------|---------------------|---------------------------------------|
| Telegram Trigger    | n8n-nodes-base.telegramTrigger             | Receives Telegram messages          | —                   | AI Agent            | Telegram Message Received Trigger     |
| AI Agent           | @n8n/n8n-nodes-langchain.agent             | Interpret message & manage logic    | Telegram Trigger, Simple Memory, OpenAI Chat Model, Google Calendar1 | Telegram            |                                       |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Language model (GPT-4o-mini)        | —                   | AI Agent            |                                       |
| Simple Memory      | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains session context           | —                   | AI Agent            |                                       |
| Google Calendar1   | n8n-nodes-base.googleCalendarTool           | Create/cancel calendar events       | AI Agent             | —                   | Calendar Tool                        |
| Telegram           | n8n-nodes-base.telegram                      | Send response message to user       | AI Agent             | —                   | Telegram Response                    |
| Sticky Note        | n8n-nodes-base.stickyNote                     | Documentation                      | —                   | —                   | Telegram Message Received Trigger     |
| Sticky Note1       | n8n-nodes-base.stickyNote                     | Documentation                      | —                   | —                   | Telegram Message Received Trigger     |
| Sticky Note2       | n8n-nodes-base.stickyNote                     | Documentation                      | —                   | —                   | Telegram Response                    |
| Sticky Note3       | n8n-nodes-base.stickyNote                     | Documentation                      | —                   | —                   | Calendar Tool                        |
| Sticky Note4       | n8n-nodes-base.stickyNote                     | Full workflow explanation & setup  | —                   | —                   | See detailed notes below              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Obtain Credentials**  
   - Use BotFather on Telegram to create a new bot.  
   - Save the API token for n8n Telegram nodes.

2. **Add Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configure with Telegram API credentials (bot token).  
   - Set Updates to listen for "message".  
   - Position near the workflow start.

3. **Add AI Agent Node (LangChain Agent)**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - `text` set to `={{ $json.message.text }}` (input message from Telegram).  
     - Define system message instructing the AI to act as an appointment scheduler with rules:  
       - Use current time reference (`{{$now}}`).  
       - Assume GMT+08 timezone.  
       - Default event duration 1 hour.  
     - Prompt Type: "define".  
   - Connect its main input from the Telegram Trigger node.

4. **Add OpenAI Chat Model Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Configure model to `gpt-4o-mini`.  
   - Add OpenAI API credentials.  
   - Connect as AI language model to the AI Agent node (`ai_languageModel` connection).

5. **Add Simple Memory Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set `sessionKey` to `={{ $json.message.chat.id }}` to maintain per-user context.  
   - Connect as `ai_memory` input to AI Agent.

6. **Add Google Calendar Tool Node**  
   - Node Type: `n8n-nodes-base.googleCalendarTool`  
   - Select the calendar (e.g., your organizational or personal calendar email).  
   - Set dynamic fields populated using AI outputs with `$fromAI()` for:  
     - `start` and `end` (event times)  
     - `summary` (event title)  
     - `attendees` (email list)  
     - `useDefaultReminders` (boolean)  
   - Add Google Calendar OAuth2 credentials.  
   - Connect as AI tool (`ai_tool`) into AI Agent.

7. **Add Telegram Node for Sending Messages**  
   - Node Type: `n8n-nodes-base.telegram`  
   - Set `text` to output from AI Agent `={{ $json.output }}`.  
   - Set `chatId` dynamically from Telegram Trigger: `={{ $('Telegram Trigger').item.json.message.chat.id }}`.  
   - Add Telegram API credentials.  
   - Connect main input from AI Agent.

8. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes near nodes with content such as "Telegram Message Received Trigger", "Calendar Tool", "Telegram Response", and the detailed workflow explanation for clarity.

9. **Verify Connections**  
   - Telegram Trigger → AI Agent  
   - AI Agent → Telegram (response)  
   - AI Agent → Google Calendar Tool (via ai_tool connection)  
   - OpenAI Chat Model → AI Agent (ai_languageModel connection)  
   - Simple Memory → AI Agent (ai_memory connection)

10. **Test Workflow**  
    - Deploy and activate the workflow.  
    - Test by sending appointment booking or cancellation messages to the Telegram bot.  
    - Confirm events are created or deleted in Google Calendar and confirmation messages are sent back.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                    | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| 🔧 How It Works: Explains the node roles and the overall process flow from Telegram message reception to AI processing, calendar event management, and confirmation messaging.                                                                                                   | Sticky Note4 content in workflow                 |
| 🧠 Why This is Useful: Automates appointment scheduling, ideal for solopreneurs, small businesses, and individuals seeking streamlined calendar management through chat interface.                                                                                                | Sticky Note4 content in workflow                 |
| 🪜 Setup Instructions: Details on creating Telegram bot, connecting OpenAI and Google Calendar credentials, and activating the workflow.                                                                                                                                          | Sticky Note4 content in workflow                 |
| Full video tutorial available at: https://youtu.be/GzWO7_1lyI8                                                                                                                                                                                                                   | Sticky Note4 content in workflow                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---