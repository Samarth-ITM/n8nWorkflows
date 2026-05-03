Conversational Google Calendar Management with Claude 3.5 Haiku & Telegram

https://n8nworkflows.xyz/workflows/conversational-google-calendar-management-with-claude-3-5-haiku---telegram-6065


# Conversational Google Calendar Management with Claude 3.5 Haiku & Telegram

### 1. Workflow Overview

This workflow enables conversational management of a Google Calendar via Telegram chat commands, leveraging advanced AI models for natural language understanding and calendar event processing. It is designed to parse user messages describing calendar events, check for scheduling conflicts, and create or request clarification for events accordingly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives Telegram messages from authorized users.
- **1.2 AI Processing & Event Handling:** Uses AI agents to parse user input, normalize event details, check conflicts, and decide whether to create an event or request clarification.
- **1.3 Calendar Operations:** Interacts with Google Calendar to fetch existing events and create new ones.
- **1.4 User Interaction:** Sends Telegram messages back to the user, including clarifications or confirmations.
- **1.5 Language Model Support:** Uses Anthropic Claude 3.5 Haiku and OpenAI GPT-4.1-mini models as base and fallback LLMs for processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages from a specific chat and triggers the workflow upon message receipt.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Purpose: Listens for new Telegram messages (updates of type "message") from a specific chat ID stored in the workflow variable `$vars.telegram_chat_id`.  
  - Key config:  
    - updates: ["message"]  
    - additionalFields: chatIds filtered by the variable, ensuring only authorized chats trigger the workflow.  
  - Credentials: Telegram API OAuth2 credentials linked to a Telegram bot account.  
  - Input: External Telegram webhook event.  
  - Output: Passes the message JSON (including the text and chat info) downstream.  
  - Edge cases:  
    - Unauthorized chat messages won't trigger.  
    - Telegram API limits or downtime could cause missed triggers.  
  - Version: 1.2  

---

#### 2.2 AI Processing & Event Handling

**Overview:**  
This block uses a Langchain AI Agent to interpret the user’s message to extract calendar event details, normalize dates and times, check for conflicts, and decide the next action (create event, explain conflict, or fallback). It integrates with two language models for processing: Anthropic Claude 3.5 Haiku as primary and OpenAI GPT-4.1-mini as fallback.

**Nodes Involved:**  
- AI Agent  
- Haiku 3.5  
- 4.1-nano

**Node Details:**  

- **AI Agent**  
  - Type: Langchain AI Agent node  
  - Role: Core logic processor that orchestrates event extraction and tool invocation (Google Calendar Get/Create, Telegram Explain).  
  - Configuration:  
    - Prompt instructs the agent to parse user text, normalize dates (YYYY-MM-DD) and times (HH:MM:SS), determine event duration (1 hour if time given, else all-day), check conflicts using the Get tool, explain conflicts if any, else create events.  
    - Includes example user inputs in Arabic with corresponding logic.  
  - Input: Telegram message text from the Telegram Trigger node.  
  - Output: Calls Google Calendar tools or Telegram Explain based on logic, then routes response text to the Result node.  
  - Uses two AI language models as tools: Haiku 3.5 (Anthropic) primary, 4.1-nano (OpenAI) fallback.  
  - Potential failures:  
    - Parsing errors if user inputs are ambiguous or out of scope.  
    - Timeout or API errors from AI providers.  
    - Failure in tool calls (Google Calendar or Telegram).  
  - Version: 2  

- **Haiku 3.5**  
  - Type: Langchain OpenAI Anthropic Chat model  
  - Purpose: Primary large language model for parsing and reasoning.  
  - Config:  
    - Model: "claude-3-5-haiku-20241022"  
  - Credentials: Anthropic API key.  
  - Input: Text from AI Agent’s internal calls.  
  - Output: Processed AI responses to AI Agent.  
  - Edge cases: Rate limits, Anthropic API downtime.  
  - Version: 1.3  

- **4.1-nano**  
  - Type: Langchain OpenAI Chat model  
  - Purpose: Fallback LLM for robustness.  
  - Config:  
    - Model: "gpt-4.1-mini"  
  - Credentials: OpenAI API key.  
  - Input/Output: Same as Haiku 3.5, used only if fallback needed.  
  - Potential failure: OpenAI API limits or errors.  
  - Version: 1.2  

---

#### 2.3 Calendar Operations

**Overview:**  
This block interfaces with Google Calendar to verify event conflicts and create new events as instructed by the AI Agent.

**Nodes Involved:**  
- Get  
- Create

**Node Details:**  

- **Get**  
  - Type: Google Calendar Tool (getAll operation)  
  - Role: Retrieves all existing events within a specified time window to check for conflicts.  
  - Configuration:  
    - timeMin and timeMax dynamically set by AI Agent via AI-generated overrides.  
    - calendar set to "msayed.cs@gmail.com" (configured calendar).  
    - returnAll flag based on AI Agent input to fetch all relevant events.  
  - Credentials: OAuth2 Google Calendar account linked to "msayed.cs@gmail.com".  
  - Input: AI Agent’s tool call with normalized times.  
  - Output: List of existing events or empty list if no conflicts.  
  - Edge cases:  
    - OAuth token expiration or permission issues.  
    - Empty or malformed time window parameters.  
  - Version: 1.3  

- **Create**  
  - Type: Google Calendar Tool (create event operation)  
  - Role: Creates a new calendar event with title, start and end times, and optional description.  
  - Configuration:  
    - Title, start, end, and description dynamically set via AI-generated overrides from AI Agent.  
    - Calendar set to "msayed.cs@gmail.com".  
  - Credentials: Same as Get node.  
  - Input: AI Agent’s instruction to create event.  
  - Output: Confirmation of event creation.  
  - Edge cases:  
    - Invalid date/time formats.  
    - Overlapping event creation attempts if race conditions occur.  
  - Version: 1.3  

---

#### 2.4 User Interaction

**Overview:**  
This block sends Telegram messages back to the user, either to explain conflicts or confirm operation results.

**Nodes Involved:**  
- Explain  
- Result

**Node Details:**  

- **Explain**  
  - Type: Telegram Tool node  
  - Role: Sends clarifying questions to the user when event creation is blocked by conflicts. Waits for free-text user response.  
  - Configuration:  
    - chatId dynamically extracted from Telegram Trigger node JSON.  
    - message text provided by AI Agent’s override (usually a request for new time or date).  
    - Waits up to 45 minutes for user reply before timeout.  
  - Credentials: Telegram API credentials same as Telegram Trigger.  
  - Input: AI Agent tool call when conflict detected.  
  - Output: User response sent back to workflow (not shown in current connections, possibly triggers new Telegram Trigger).  
  - Edge cases:  
    - User does not reply within wait time.  
    - Telegram API message send failures.  
  - Version: 1.2  

- **Result**  
  - Type: Telegram node (message sender)  
  - Role: Sends final confirmation or completion message to the original Telegram chat after processing.  
  - Configuration:  
    - Text includes static "Done" message and appends AI Agent output.  
    - Sends to chat ID from Telegram Trigger.  
    - Attribution disabled to keep message clean.  
  - Credentials: Telegram API same as others.  
  - Input: Main output from AI Agent node after event processing.  
  - Output: Message delivery to user.  
  - Edge cases: Telegram API errors, chat ID changes.  
  - Version: 1.2  

---

#### 2.5 Language Model Support - Sticky Notes

**Overview:**  
Sticky notes provide documentation and high-level explanations within the editor for LLM usage and available tools.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**  

- These nodes are non-functional nodes used for documentation inside n8n editor.  
- Content:  
  - Sticky Note: "## LLMs Base + Fallback" indicating the usage of primary and fallback models.  
  - Sticky Note1: Explanation of tools used by the AI Agent: Explain, Get, Create.  
  - Sticky Note2: Clarifies that the Result node sends the final message back to the Telegram chat.  
- Positioning provides context near relevant nodes.  
- No inputs or outputs.  
- No version requirements.

---

### 3. Summary Table

| Node Name       | Node Type                         | Functional Role                                  | Input Node(s)          | Output Node(s)         | Sticky Note                          |
|-----------------|----------------------------------|-------------------------------------------------|-----------------------|-----------------------|------------------------------------|
| Telegram Trigger| Telegram Trigger                 | Receive Telegram messages                        | (external webhook)     | AI Agent              |                                    |
| AI Agent        | Langchain AI Agent              | Parse user message, orchestrate calendar ops   | Telegram Trigger       | Result                |                                    |
| Haiku 3.5       | Langchain Anthropic Chat Model  | Primary LLM for parsing and reasoning           | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) | LLMs Base + Fallback               |
| 4.1-nano        | Langchain OpenAI Chat Model     | Fallback LLM                                     | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) | LLMs Base + Fallback               |
| Get             | Google Calendar Tool (getAll)   | Retrieve existing events (conflict check)       | AI Agent (ai_tool)     | AI Agent (ai_tool)     | Tools: Explain, Get, Create        |
| Create          | Google Calendar Tool (create)   | Create new calendar event                         | AI Agent (ai_tool)     | AI Agent (ai_tool)     | Tools: Explain, Get, Create        |
| Explain         | Telegram Tool                   | Ask user for clarification on conflicts         | AI Agent (ai_tool)     | (awaits user input)    | Tools: Explain, Get, Create        |
| Result          | Telegram Node                   | Send final confirmation message                   | AI Agent (main)        | (external message)     | Send Result: final message to user |
| Sticky Note     | Sticky Note                    | Documentation (LLMs)                             | None                  | None                  | LLMs Base + Fallback               |
| Sticky Note1    | Sticky Note                    | Documentation (Tools)                            | None                  | None                  | Tools: Explain, Get, Create        |
| Sticky Note2    | Sticky Note                    | Documentation (Send Result)                      | None                  | None                  | Send Result: final message to user |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: ["message"]  
     - Additional Fields: chatIds = `={{ $vars.telegram_chat_id }}` (set this variable in n8n)  
   - Credentials: Connect your Telegram Bot API credentials.  
   - Position near workflow start.

2. **Create Langchain AI Agent node:**  
   - Type: @n8n/n8n-nodes-langchain.agent  
   - Parameters:  
     - Prompt: Use detailed instructions to parse event title, date, time, normalize formats, check conflicts, explain conflicts, or create events.  
     - Needs fallback: True  
   - Connect output of Telegram Trigger to AI Agent main input.  
   - Position right of Telegram Trigger.

3. **Add Claude 3.5 Haiku language model node:**  
   - Type: @n8n/n8n-nodes-langchain.lmChatAnthropic  
   - Parameters: Model = "claude-3-5-haiku-20241022"  
   - Credentials: Configure Anthropic API key.  
   - Connect as ai_languageModel input to AI Agent.  
   - Position below AI Agent.

4. **Add OpenAI GPT-4.1-mini fallback language model node:**  
   - Type: @n8n/n8n-nodes-langchain.lmChatOpenAi  
   - Parameters: Model = "gpt-4.1-mini"  
   - Credentials: Configure OpenAI API key.  
   - Connect as ai_languageModel fallback input to AI Agent.  
   - Position below or near Claude 3.5 node.

5. **Create Google Calendar Get node:**  
   - Type: n8n-nodes-base.googleCalendarTool  
   - Parameters:  
     - Operation: getAll  
     - Calendar: "msayed.cs@gmail.com" (replace with your calendar email)  
     - timeMin: `={{ $fromAI('After', '', 'string') }}`  
     - timeMax: `={{ $fromAI('Before', '', 'string') }}`  
     - returnAll: `={{ $fromAI('Return_All', '', 'boolean') }}`  
   - Credentials: Google Calendar OAuth2 for the calendar.  
   - Connect as ai_tool input to AI Agent.

6. **Create Google Calendar Create node:**  
   - Type: n8n-nodes-base.googleCalendarTool  
   - Parameters:  
     - Calendar: same as Get node  
     - Title, start, end, description fields set to AI overrides:  
       - start: `={{ $fromAI('Start', '', 'string') }}`  
       - end: `={{ $fromAI('End', '', 'string') }}`  
       - description: `={{ $fromAI('Description', '', 'string') }}`  
   - Credentials: Same Google OAuth2 credentials.  
   - Connect as ai_tool input to AI Agent.

7. **Create Telegram Explain node:**  
   - Type: n8n-nodes-base.telegramTool  
   - Parameters:  
     - chatId: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
     - message: `={{ $fromAI('Message', '', 'string') }}`  
     - Operation: sendAndWait  
     - Wait time limit: 45 minutes  
   - Credentials: Same Telegram API credentials as Telegram Trigger.  
   - Connect as ai_tool input to AI Agent.

8. **Create Telegram Result node:**  
   - Type: n8n-nodes-base.telegram  
   - Parameters:  
     - Text: `=Done\n---\n{{ $json.output }}`  
     - chatId: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
     - Append attribution: false  
   - Credentials: Same Telegram API credentials.  
   - Connect AI Agent main output to Result node.

9. **Add Sticky Notes for documentation (optional):**  
   - Create three sticky notes with content describing LLM usage, tools, and send result explanation near respective nodes.

10. **Set workflow variables:**  
    - Define `$vars.telegram_chat_id` with the authorized Telegram chat ID(s).

11. **Test the workflow:**  
    - Send messages in Telegram matching examples ("اضف ميعاد ..." or "ذكرني ب...") to verify event parsing, conflict checking, and creation.  
    - Monitor Telegram messages for confirmations or clarifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow uses Claude 3.5 Haiku as the main AI model with GPT-4.1-mini as fallback for robust natural language parsing. | See Anthropic Claude API and OpenAI GPT-4.1 API documentation for details on model capabilities.    |
| Telegram interaction requires bot token and chat ID whitelisting via `$vars.telegram_chat_id` for security.             | Telegram Bot API: https://core.telegram.org/bots/api                                                        |
| Google Calendar OAuth2 credentials must have permission to read and write events on the specified calendar email.       | Google Calendar API docs: https://developers.google.com/calendar/api/v3/reference/                   |
| The AI Agent uses a strict multi-step logic to normalize dates and times, check conflicts before event creation.        | This ensures no double-bookings and user-friendly error handling.                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.