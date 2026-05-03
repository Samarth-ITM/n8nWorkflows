Manage Google Calendar Events via Telegram with Gemini AI Assistant

https://n8nworkflows.xyz/workflows/manage-google-calendar-events-via-telegram-with-gemini-ai-assistant-5854


# Manage Google Calendar Events via Telegram with Gemini AI Assistant

### 1. Workflow Overview

This workflow, titled **"Manage Google Calendar Events via Telegram with Gemini AI Assistant"**, is designed to provide an interactive Telegram bot interface that allows users to manage their Google Calendar events using natural language commands processed via an AI assistant powered by Google Gemini. It integrates Telegram messaging, Google Calendar operations, and AI language processing to enable creating, updating, retrieving, and deleting calendar events through conversational inputs.

The workflow logic is organized into the following functional blocks:

- **1.1 Input Reception and Initialization**: Captures Telegram messages via webhook, sets user-specific variables, and initializes flow control.
- **1.2 Start Command Handling**: Detects the '/start' command to send a welcome message to the user.
- **1.3 Command Type Determination**: Differentiates between start commands and calendar operation requests.
- **1.4 AI Processing with Memory**: Uses Google Gemini model and a memory buffer to understand and process user commands.
- **1.5 Calendar Operations**: Executes Google Calendar API operations (Get, Create, Update, Delete) as interpreted by the AI agent.
- **1.6 Response Delivery**: Sends processed answers back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview**: Receives incoming Telegram messages, sets initial variables, and prepares the workflow context.
- **Nodes Involved**:  
  - Telegram Trigger  
  - Variables TG  
  - Initialization

- **Node Details**:

  - **Telegram Trigger**  
    - Type: `telegramTrigger`  
    - Role: Entry point capturing Telegram messages via webhook.  
    - Configuration: Uses a dedicated webhook ID for Telegram integration; no additional parameters specified.  
    - I/O: No input; outputs message data to "Variables TG".  
    - Edge Cases: Telegram API downtime, webhook misconfiguration, malformed message data.

  - **Variables TG**  
    - Type: `set`  
    - Role: Stores or prepares variables needed for subsequent nodes.  
    - Configuration: No explicit parameters shown; likely sets message-related variables such as user ID, chat ID, message text for downstream use.  
    - I/O: Input from Telegram Trigger; output to Initialization node.  
    - Edge Cases: Missing or malformed message fields may cause expression failures.

  - **Initialization**  
    - Type: `set`  
    - Role: Initializes control variables or flags for flow branching.  
    - Configuration: No explicit parameters visible.  
    - I/O: Input from Variables TG; output to "Is start?" node.  
    - Edge Cases: Misconfiguration might disrupt conditional logic downstream.

#### 2.2 Start Command Handling

- **Overview**: Detects if the user sent a "/start" command and sends a welcome message.
- **Nodes Involved**:  
  - Is start? (If node)  
  - Welcome message

- **Node Details**:

  - **Is start?**  
    - Type: `if`  
    - Role: Checks if incoming message equals the start command.  
    - Configuration: Condition likely compares message text with "/start".  
    - I/O: Input from Initialization; outputs to Welcome message (true branch) or Define Type (false branch).  
    - Edge Cases: Case sensitivity or whitespace in commands might cause false negatives.

  - **Welcome message**  
    - Type: `telegram`  
    - Role: Sends a greeting message to the user upon start command.  
    - Configuration: Uses Telegram credentials; message content configured to welcome user.  
    - I/O: Input from Is start? node true branch; no output (ends branch).  
    - Edge Cases: Telegram API errors, message length restrictions.

#### 2.3 Command Type Determination

- **Overview**: Routes non-start messages into AI processing or terminates if unrecognized.
- **Nodes Involved**:  
  - Define Type (Switch)  
  - AI Agent

- **Node Details**:

  - **Define Type**  
    - Type: `switch`  
    - Role: Determines the type of user command (e.g., calendar event CRUD or other).  
    - Configuration: Branches based on message content or parsed intent. Has three output branches: one empty (no action), one to AI Agent, one empty.  
    - I/O: Input from Is start? false branch; outputs to AI Agent or none.  
    - Edge Cases: Ambiguous or malformed inputs may cause misrouting.

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Core AI processing node that uses language models and tools to interpret user intent and execute tasks.  
    - Configuration: Connected to Google Gemini Chat Model (language model), Simple Memory (context buffer), and Google Calendar tools (Get, Create, Update, Delete).  
    - Inputs: AI language model, memory, and calendar tool outputs.  
    - Outputs: Sends results to "Send Answer" node.  
    - Edge Cases: AI model unavailability, memory desynchronization, tool API failures.

#### 2.4 AI Processing with Memory

- **Overview**: Enhances AI understanding using conversational memory and Google Gemini language model.
- **Nodes Involved**:  
  - Google Gemini Chat Model  
  - Simple Memory

- **Node Details**:

  - **Google Gemini Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Provides conversational AI language understanding and generation.  
    - Configuration: No explicit parameters, uses default Google Gemini chat model setup.  
    - I/O: Output connects to AI Agent as language model input.  
    - Edge Cases: API quota limits, latency, model version updates.

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a sliding window of conversation context for AI continuity.  
    - Configuration: Default memory buffer window parameters (likely recent messages stored).  
    - I/O: Connects as AI memory input to AI Agent.  
    - Edge Cases: Memory overflow, incorrect context handling causing AI confusion.

#### 2.5 Calendar Operations

- **Overview**: Executes Google Calendar API operations as requested by the AI agent.
- **Nodes Involved**:  
  - Get Calendar Event  
  - Create Calendar Event  
  - Update Calendar Event  
  - Delete Calendar Event

- **Node Details**:

  - **Get Calendar Event**  
    - Type: `googleCalendarTool`  
    - Role: Retrieves event details based on AI agent request.  
    - Configuration: Uses Google Calendar credentials with appropriate scopes (read).  
    - I/O: Input from AI Agent tool interface; output back to AI Agent.  
    - Edge Cases: Authentication errors, missing events, API rate limits.

  - **Create Calendar Event**  
    - Type: `googleCalendarTool`  
    - Role: Creates new calendar events as specified.  
    - Configuration: Uses Google Calendar with write permissions.  
    - Edge Cases: Invalid date/time formats, permission issues.

  - **Update Calendar Event**  
    - Type: `googleCalendarTool`  
    - Role: Modifies existing events.  
    - Edge Cases: Event not found, conflicting updates.

  - **Delete Calendar Event**  
    - Type: `googleCalendarTool`  
    - Role: Deletes specified calendar events.  
    - Edge Cases: Attempting to delete non-existent events, permission denial.

#### 2.6 Response Delivery

- **Overview**: Sends the AI-generated response back to the Telegram user.
- **Nodes Involved**:  
  - Send Answer

- **Node Details**:

  - **Send Answer**  
    - Type: `telegram`  
    - Role: Delivers the AI's reply message to the Telegram chat.  
    - Configuration: Uses Telegram credentials, sends text messages.  
    - I/O: Input from AI Agent output; no further output.  
    - Edge Cases: Telegram API errors, message size limits.

---

### 3. Summary Table

| Node Name               | Node Type                                     | Functional Role                      | Input Node(s)             | Output Node(s)             | Sticky Note                        |
|-------------------------|-----------------------------------------------|------------------------------------|---------------------------|----------------------------|----------------------------------|
| Telegram Trigger        | telegramTrigger                               | Entry point, receives Telegram msg | -                         | Variables TG               |                                  |
| Variables TG            | set                                           | Sets initial variables              | Telegram Trigger          | Initialization             |                                  |
| Initialization          | set                                           | Initializes control variables       | Variables TG              | Is start?                  |                                  |
| Is start?               | if                                            | Checks for /start command           | Initialization            | Welcome message, Define Type |                                  |
| Welcome message         | telegram                                      | Sends welcome message               | Is start? (true branch)   | -                          |                                  |
| Define Type             | switch                                        | Routes commands by type             | Is start? (false branch)  | AI Agent                   |                                  |
| AI Agent                | langchain.agent                               | Processes user input via AI         | Define Type, Simple Memory, Google Gemini Chat Model, Google Calendar tools | Send Answer                |                                  |
| Google Gemini Chat Model| langchain.lmChatGoogleGemini                  | Provides AI language model          | -                         | AI Agent                   |                                  |
| Simple Memory           | langchain.memoryBufferWindow                   | Maintains conversational context   | -                         | AI Agent                   |                                  |
| Get Calendar Event      | googleCalendarTool                            | Retrieves calendar events           | AI Agent (tool input)     | AI Agent                   |                                  |
| Create Calendar Event   | googleCalendarTool                            | Creates calendar events             | AI Agent (tool input)     | AI Agent                   |                                  |
| Update Calendar Event   | googleCalendarTool                            | Updates calendar events             | AI Agent (tool input)     | AI Agent                   |                                  |
| Delete Calendar Event   | googleCalendarTool                            | Deletes calendar events             | AI Agent (tool input)     | AI Agent                   |                                  |
| Send Answer             | telegram                                      | Sends reply to Telegram user        | AI Agent                  | -                          |                                  |
| Sticky Note             | stickyNote                                    | Comment placeholder                 | -                         | -                          |                                  |
| Sticky Note2            | stickyNote                                    | Comment placeholder                 | -                         | -                          |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Configure webhook ID for Telegram bot integration.  
   - No additional parameters needed.

2. **Add Set Node "Variables TG"**  
   - Type: `set`  
   - Configure to capture and set variables such as `chatId`, `userId`, `messageText` from Telegram Trigger output.

3. **Add Set Node "Initialization"**  
   - Type: `set`  
   - Initialize any required control flags or variables (e.g., `isStartCommand` unset initially).

4. **Connect Nodes**:  
   - Telegram Trigger → Variables TG → Initialization

5. **Add If Node "Is start?"**  
   - Type: `if`  
   - Condition: Check if `messageText` equals "/start" (case sensitive or insensitive as preferred).  
   - True branch → "Welcome message" node  
   - False branch → "Define Type" node

6. **Add Telegram Node "Welcome message"**  
   - Type: `telegram`  
   - Configure to send a welcome message (e.g., "Hello! I am your AI assistant to manage your Google Calendar.")  
   - Use Telegram credentials.

7. **Add Switch Node "Define Type"**  
   - Type: `switch`  
   - Configure to route based on message content or intent detection.  
   - Primary branch connects to AI Agent node; other branches can be left empty or for future extension.

8. **Add Langchain Nodes for AI Processing**:

   - **Google Gemini Chat Model**  
     - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
     - Use default or configure API credentials for Google Gemini.

   - **Simple Memory**  
     - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
     - Use default parameters for buffer window size.

9. **Add Langchain Agent Node "AI Agent"**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Connect inputs:  
     - AI language model input from Google Gemini Chat Model  
     - AI memory input from Simple Memory  
     - AI tool inputs from Google Calendar Tool nodes  
   - Configure with Google Calendar tools as available AI tools.

10. **Add Google Calendar Tool Nodes** (Require Google OAuth2 credentials with appropriate scopes):

    - **Get Calendar Event**  
      - Type: `googleCalendarTool`  
      - Configure for event retrieval operation.

    - **Create Calendar Event**  
      - Type: `googleCalendarTool`  
      - Configure for event creation.

    - **Update Calendar Event**  
      - Type: `googleCalendarTool`  
      - Configure for event update.

    - **Delete Calendar Event**  
      - Type: `googleCalendarTool`  
      - Configure for event deletion.

    - Connect all these nodes as AI tools input to AI Agent node.

11. **Add Telegram Node "Send Answer"**  
    - Type: `telegram`  
    - Send the AI Agent’s output message back to the user via Telegram.  
    - Use Telegram credentials.

12. **Connect Nodes**:

    - Initialization → Is start?  
    - Is start? true → Welcome message  
    - Is start? false → Define Type  
    - Define Type primary branch → AI Agent  
    - AI Agent → Send Answer  
    - Google Gemini Chat Model → AI Agent (language model input)  
    - Simple Memory → AI Agent (memory input)  
    - Google Calendar Tool nodes → AI Agent (tool inputs)

13. **Credentials Setup**:

    - Telegram Bot: OAuth2 or Bot Token as per Telegram node requirements.  
    - Google Calendar: OAuth2 credentials with scopes for calendar read/write.  
    - Google Gemini: API key or OAuth as required by Langchain Google Gemini node.

14. **Defaults and Constraints**:

    - Ensure webhook URLs are publicly accessible and correctly configured in Telegram Bot settings.  
    - Limit memory buffer size to reasonable window (e.g., last 10 messages) to avoid excessive context.  
    - Handle error outputs for nodes with API calls for graceful degradation.

15. **Testing**:

    - Send "/start" message in Telegram to receive welcome message.  
    - Send natural language calendar commands to test AI parsing and calendar operations.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow leverages Google Gemini for advanced conversational AI capabilities.                   | Requires appropriate access to Google Gemini API via Langchain integration.                                     |
| Telegram nodes require proper webhook configuration and valid Bot Tokens for message delivery.      | https://core.telegram.org/bots/api                                                                                 |
| Google Calendar nodes require OAuth2 credentials with calendar scopes enabled (read/write access).  | https://developers.google.com/calendar/api/guides/authentication                                                |
| The use of memory buffer enhances contextual understanding but may increase processing time.         | Adjust buffer window size to balance context and performance.                                                  |
| No explicit sticky note content was present, but placeholders exist for user annotations.            | Sticky notes can be used to document workflow changes or reminders within the n8n editor.                       |

---

**Disclaimer:** The provided text exclusively derives from an automated workflow created with n8n, a tool for integration and automation. This processing strictly follows current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.