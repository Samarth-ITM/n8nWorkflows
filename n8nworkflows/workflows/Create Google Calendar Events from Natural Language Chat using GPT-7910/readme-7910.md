Create Google Calendar Events from Natural Language Chat using GPT

https://n8nworkflows.xyz/workflows/create-google-calendar-events-from-natural-language-chat-using-gpt-7910


# Create Google Calendar Events from Natural Language Chat using GPT

### 1. Workflow Overview

This workflow automates the creation of Google Calendar events from natural language chat messages using GPT-based AI processing. It is designed for scenarios where users interact via chat to schedule events conversationally. The workflow interprets user input, extracts event details, and creates calendar entries accordingly.

The logic is divided into the following functional blocks:

- **1.1 Input Reception:** Receives chat messages from users via a webhook-enabled chat trigger.
- **1.2 AI Processing:** Uses a LangChain AI agent powered by OpenAI GPT-4o-mini to understand user intent, extract structured event data, and manage dialogue context.
- **1.3 Google Calendar Integration:** Creates Google Calendar events based on AI-extracted event details.
- **1.4 (Disabled) Chat Response:** Intended to send confirmation messages back to the user after event creation (currently disabled).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages and initiates the workflow with an initial greeting prompt.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Entry point of the workflow; listens on a public webhook for chat messages.  
    - *Configuration:*  
      - `public`: true (accessible via public webhook)  
      - `initialMessages`: "Hey bestieeeee, what can I schedule for you?" (Initial prompt shown to user)  
      - `responseMode`: "responseNodes" (allows branching based on AI agent output)  
    - *Connections:* Outputs main data to AI Agent node.  
    - *Potential Failures:* Webhook connectivity issues, unauthorized access if webhook URL is public and abused.  
    - *Version:* 1.3  

---

#### 2.2 AI Processing

- **Overview:**  
  Processes chat input using an AI agent that interprets natural language, extracts structured event details, manages dialogue memory, and calls the Google Calendar tool for event creation.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  

- **Node Details:**  

  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Core logic processor that interprets user input, applies system instructions, calls external tools (Google Calendar), and controls conversation flow.  
    - *Configuration:*  
      - System message instructs the agent to only create Google Calendar events by calling the attached tool and never to return raw JSON to users.  
      - Specifies required event fields: summary, start.dateTime, end.dateTime, timeZone, etc.  
      - Includes detailed title and time handling rules and interaction guidelines (clarification questions when ambiguous).  
      - Uses current time in ISO8601 and default timezone Asia/Manila for context.  
    - *Key Expressions:*  
      - Uses n8n expression syntax for current time: `{{ $now.toISO() }}`  
    - *Connections:*  
      - Inputs: receives chat messages from "When chat message received" node.  
      - AI language model input from "OpenAI Chat Model".  
      - AI memory from "Simple Memory".  
      - AI tool output connected to "Create an event in Google Calendar".  
      - Main output connected to disabled "Respond to Chat" node.  
    - *Potential Failures:*  
      - AI model API errors or rate limits.  
      - Expression evaluation errors in system message.  
      - Ambiguous user input causing endless clarification loops.  
      - Errors in AI tool call or unexpected tool output.  
    - *Version:* 2.2  

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Provides GPT-4o-mini model for natural language understanding and generation.  
    - *Configuration:*  
      - Model: "gpt-4o-mini" selected from a list.  
      - No special options set.  
      - Uses OpenAI API credentials named "OpenAi account".  
    - *Connections:* Outputs to AI Agent as language model input.  
    - *Potential Failures:* API token invalidation, network issues, rate limiting.  
    - *Version:* 1.2  

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains dialogue context up to 10 previous messages to enable coherent multi-turn conversations.  
    - *Configuration:*  
      - Context window length: 10 messages.  
    - *Connections:* Outputs AI memory context to AI Agent.  
    - *Potential Failures:* Memory overflow or truncation might cause context loss in very long conversations.  
    - *Version:* 1.3  

---

#### 2.3 Google Calendar Integration

- **Overview:**  
  This block creates the actual event in Google Calendar using the structured data extracted by the AI agent.

- **Nodes Involved:**  
  - Create an event in Google Calendar

- **Node Details:**  
  - **Create an event in Google Calendar**  
    - *Type:* Google Calendar Tool  
    - *Role:* Creates calendar events with specified start/end times and summary from AI agent data.  
    - *Configuration:*  
      - `start` and `end` datetime fields are populated dynamically from AI agent outputs using `$fromAI` expressions to override defaults.  
      - `summary` (event title) is also pulled from AI agent output.  
      - `useDefaultReminders` is optionally set based on AI input.  
      - `calendar` field is empty in configuration, meaning it uses the default calendar or requires manual setup if multiple calendars exist.  
    - *Connections:* Receives AI tool calls from AI Agent node.  
    - *Credentials:* Uses OAuth2 credentials named "Google Calendar account".  
    - *Potential Failures:*  
      - OAuth token expiration or permission errors.  
      - Invalid date/time formats leading to API errors.  
      - Missing required fields if AI output is incomplete.  
    - *Version:* 1.3  

---

#### 2.4 Chat Response (Disabled)

- **Overview:**  
  Intended to send a confirmation chat message to the user after creating the event, but this node is currently disabled.

- **Nodes Involved:**  
  - Respond to Chat

- **Node Details:**  
  - **Respond to Chat**  
    - *Type:* LangChain Chat node  
    - *Role:* Would send a text message back to the user confirming the event creation.  
    - *Configuration:*  
      - Static message: "Your event has been created!"  
    - *Connections:* Receives main output from AI Agent.  
    - *Disabled:* Yes, so no active role currently.  
    - *Potential Failures:* None while disabled, but if enabled, potential errors include message format issues or webhook response errors.  
    - *Version:* 1  

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                      | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                  |
|-----------------------------|------------------------------------|------------------------------------|-----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When chat message received  | @n8n/n8n-nodes-langchain.chatTrigger | Input reception via chat webhook   |                             | AI Agent                         |                                                                                              |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent      | Core AI processing and tool calling | When chat message received, Simple Memory, OpenAI Chat Model | Respond to Chat (disabled), Create an event in Google Calendar |                                                                                              |
| OpenAI Chat Model           | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini language model |                             | AI Agent                         |                                                                                              |
| Simple Memory               | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains dialogue context         |                             | AI Agent                         |                                                                                              |
| Respond to Chat             | @n8n/n8n-nodes-langchain.chat       | (Disabled) Send confirmation message | AI Agent                    |                                   | Node is disabled; intended to confirm event creation to user.                                |
| Create an event in Google Calendar | n8n-nodes-base.googleCalendarTool  | Creates Google Calendar event      | AI Agent                    |                                   | Relies on AI agent output for all event details; requires valid OAuth2 Google credentials.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a node of type **LangChain Chat Trigger**.  
   - Set `public` to true for public webhook access.  
   - Set `initialMessages` to: `"Hey bestieeeee, what can I schedule for you?"`  
   - Set `responseMode` to `"responseNodes"`.  
   - This node will expose a webhook URL to receive chat messages.  

2. **Create the OpenAI Chat Model Node**  
   - Add a node of type **LangChain OpenAI Chat Model**.  
   - Select model `"gpt-4o-mini"` from the list.  
   - Add OpenAI API credentials (create if needed) with valid API key under `"OpenAi account"`.  
   - No special options needed.  

3. **Create the Simple Memory Node**  
   - Add a node of type **LangChain Memory Buffer Window**.  
   - Set `contextWindowLength` to 10 messages to maintain conversation context.  

4. **Create the AI Agent Node**  
   - Add a node of type **LangChain Agent**.  
   - Configure the system message as follows (replace placeholders with exact text):  
     ```
     You are a scheduling agent inside an n8n workflow.

     Now (ISO8601): {{ $now.toISO() }}
     Default timezone: Asia/Manila

     Your ONLY job is to create Google Calendar events by CALLING the attached tool.
     Do NOT return JSON to the user. Always call the tool.

     Required fields:
     - summary (title) → MUST NOT be empty
     - description (optional)
     - start.dateTime (ISO8601 with offset, e.g. 2025-08-24T10:00:00+08:00)
     - end.dateTime (ISO8601 with offset)
     - timeZone (IANA string, e.g. "Asia/Manila")
     - location (optional)
     - attendees (array of { "email": "user@example.com" })

     Title handling rules:
     - If the user gives a clear title (“Standup”, “Dentist”), use it directly.
     - If the user does not give a title, infer a concise one from context.
       Examples:
       - “Lunch with John” if user said “schedule lunch with John tomorrow”
       - “Dentist” if user said “dentist appointment next Wed”
       - “Meeting” if nothing else is obvious
     - NEVER leave `summary` blank.

     Time handling rules:
     - Resolve relative times (“tomorrow 10am”, “next Wed 3–4:30pm”) against Now and the default timezone.
     - If duration is given, compute end = start + duration.
     - If no end/duration given, default to 60 minutes.

     Interaction rules:
     - If either title or time is too ambiguous, ask ONE clarification question and wait.
     - Otherwise, immediately call the Calendar tool with the required fields.

     Output policy:
     - Do not output JSON.
     - After the tool runs, summarize the booked event in one sentence (e.g., “Booked Dentist on Sept 3, 2:00–2:30 PM Asia/Manila”).
     ```
   - Connect inputs:  
     - `main` input from **When chat message received** node.  
     - `ai_languageModel` input from **OpenAI Chat Model** node.  
     - `ai_memory` input from **Simple Memory** node.  
   - Configure AI tool output to connect to Google Calendar node (next step).  
   - Connect main output optionally to a chat response node (see step 6).  

5. **Create Google Calendar Tool Node**  
   - Add a **Google Calendar Tool** node.  
   - Configure the following parameters using expressions to accept AI agent overrides:  
     - `start`: `={{ $fromAI('Start', '', 'string') }}`  
     - `end`: `={{ $fromAI('End', '', 'string') }}`  
     - `additionalFields.summary`: `={{ $fromAI('Summary', '', 'string') }}`  
     - `useDefaultReminders`: `={{ $fromAI('Use_Default_Reminders', '', 'boolean') }}` (optional)  
     - Leave `calendar` empty unless a specific calendar ID is needed.  
   - Set credentials to a valid OAuth2 Google Calendar account with event creation permission.  
   - Connect `ai_tool` input from AI Agent node.  

6. **(Optional) Add Respond to Chat Node**  
   - Add a LangChain Chat node to send confirmation messages.  
   - Set message content to `"Your event has been created!"` or customize as needed.  
   - Connect from AI Agent's main output.  
   - Enable the node if confirmation feedback is desired.  

7. **Finalize and Activate**  
   - Verify all connections and credentials are correct.  
   - Activate the workflow.  
   - Test by sending chat messages to the webhook URL and check if events are created in Google Calendar.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                         |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| The AI Agent uses a carefully crafted system message to enforce strict event creation rules and avoid JSON output. | Internal system message in AI Agent node configuration.                |
| The workflow defaults to timezone "Asia/Manila" for resolving relative times.                        | System message and AI Agent configuration.                             |
| Uses LangChain nodes integrated in n8n for conversational AI workflows: chat trigger, agent, memory, and chat nodes. | Official n8n LangChain integration documentation.                      |
| Google Calendar OAuth2 credentials must have proper scopes to create events: `https://www.googleapis.com/auth/calendar.events` | Google API Console OAuth2 setup for Google Calendar.                   |
| The chat trigger node exposes a public webhook; secure usage by restricting access externally if needed. | Security best practices for public webhooks.                           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.