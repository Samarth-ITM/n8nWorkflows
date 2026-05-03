Cybersecurity Assistant with GPT-4, Telegram Bot & Command Execution

https://n8nworkflows.xyz/workflows/cybersecurity-assistant-with-gpt-4--telegram-bot---command-execution-7023


# Cybersecurity Assistant with GPT-4, Telegram Bot & Command Execution

### 1. Workflow Overview

This workflow implements a **Cybersecurity Assistant** powered by GPT-4, integrated with Telegram for conversational interaction, and equipped with command execution and Google Calendar management capabilities. It targets cybersecurity professionals or enthusiasts who need an AI assistant capable of advanced cybersecurity research, operational support, system command execution, calendar event management, and real-time internet search — all accessible through a Telegram bot interface.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception via Telegram:** Captures user messages from Telegram and triggers the workflow.
- **1.2 AI Agent Processing:** Uses a sophisticated AI agent configured with GPT-4, Langchain tools, and memory to process user requests, perform cybersecurity-specific reasoning, and decide on tool usage.
- **1.3 AI Tools & External Integrations:** Includes specialized tools and integrations invoked by the AI agent:
  - OpenAI GPT-4 model for language understanding and generation.
  - Calculator for mathematical operations.
  - Search_tool for real-time internet web searches.
  - Command execution on the host or connected systems.
  - Google Calendar tools to create, update, delete, and query calendar events.
  - Memory buffer to keep conversation context.
- **1.4 Telegram Output and User Feedback:** Sends AI-generated replies back to the Telegram user and manages typing indicators to improve user experience.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception via Telegram

- **Overview:** Listens for incoming messages from a specific Telegram user or group and initiates the workflow.
- **Nodes Involved:** `chatbot`

**Node Details:**

- **Node:** chatbot  
  - Type: Telegram Trigger  
  - Role: Entry point that receives all Telegram updates/messages, filtered to a specific chat ID (`831690003`).  
  - Configuration: Listens to all update types, downloads media when present, credentials tied to "Shadowark AI" Telegram API.  
  - Inputs: External (Telegram messages)  
  - Outputs: Passes message data to the `Agent` node.  
  - Edge Cases:  
    - If Telegram API rate limits or downtime occur, messages may be missed or delayed.  
    - Only messages from the specified chat ID are processed; others ignored.

---

#### 2.2 AI Agent Processing

- **Overview:** Processes the user's Telegram message using an advanced AI agent configured with GPT-4 and integrated tools. The agent executes cybersecurity-specific logic, commands, and research autonomously.
- **Nodes Involved:** `Agent`, `OpenAI Model`, `Simple Memory`

**Node Details:**

- **Node:** Agent  
  - Type: Langchain Agent  
  - Role: Central AI logic processor that interprets user input, manages tool usage, and produces final outputs.  
  - Configuration:  
    - Input text is the Telegram user message text.  
    - System message defines agent personality and capabilities: elite cybersecurity AI named QuantumDefender AI with operational autonomy, extensive domain expertise, safe execution environment, and Telegram integration conventions.  
    - Prompt instructs agent to always send typing indicators, use Search_tool for internet queries, execute shell commands safely, and handle multi-step responses properly.  
  - Inputs: Message text from `chatbot`, tools outputs, memory context.  
  - Outputs: Final response text sent to Telegram, and intermediate tool invocations.  
  - Version: 1.7 (Langchain agent)  
  - Edge Cases:  
    - Expression errors if message text is missing or malformed.  
    - Failures if OpenAI API is unreachable or credentials fail.  
    - Misbehavior if system prompt is overridden or malformed.  
  - Sub-workflow: None.

- **Node:** OpenAI Model  
  - Type: Langchain Chat OpenAI  
  - Role: Provides GPT-4 language model capabilities to the Agent.  
  - Configuration: Uses GPT-4.1-nano model with no special options.  
  - Inputs: Agent language model requests.  
  - Outputs: Language generation responses to Agent.  
  - Credentials: OpenAI API account "OpenAi account".  
  - Edge Cases: API rate limits, network failures.

- **Node:** Simple Memory  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversation history per Telegram user to provide context for ongoing sessions.  
  - Configuration: Session key derived from Telegram user ID (`chatbot` node message sender ID).  
  - Inputs: Conversation context to Agent.  
  - Outputs: Contextual memory data back to Agent.  
  - Edge Cases: Memory overflow or reset if keys mismatch.

---

#### 2.3 AI Tools & External Integrations

- **Overview:** This block includes multiple specialized tools invoked by the Agent to fulfill tasks such as searching the web, executing shell commands, performing calculations, and managing Google Calendar events.
- **Nodes Involved:**  
  - `send_typing`  
  - `Search_tool`  
  - `Execute Command`  
  - `Calculator`  
  - `Delete-event`  
  - `create-event`  
  - `update-events`  
  - `Google Calendar`

**Node Details:**

- **Node:** send_typing  
  - Type: Telegram Tool  
  - Role: Sends "typing..." chat action in Telegram to indicate the bot is processing.  
  - Configuration: Fixed chat ID `831690003`.  
  - Inputs: Triggered by Agent when it starts processing.  
  - Outputs: None (side-effect only).  
  - Credentials: Telegram API "Shadowark AI".  
  - Edge Cases: Fails if Telegram chat ID changes or network issues occur.

- **Node:** Search_tool  
  - Type: HTTP Request Tool  
  - Role: Performs web searches through Langsearch API for real-time internet data.  
  - Configuration: POST request to `https://api.langsearch.com/v1/web-search` with query, freshness, summary, and count parameters. Authorization header with API key present.  
  - Inputs: Query string from Agent.  
  - Outputs: JSON search results to Agent.  
  - Edge Cases: API key expiration, rate limits, response format changes.

- **Node:** Execute Command  
  - Type: Execute Command Tool  
  - Role: Runs shell commands on the host or connected systems, enabling remote SSH commands or local execution.  
  - Configuration: Command and executeOnce flag dynamically passed from Agent inputs.  
  - Inputs: Command strings from Agent.  
  - Outputs: Command output to Agent.  
  - Edge Cases: Command failure, permission errors, security risks if misused.

- **Node:** Calculator  
  - Type: Langchain Calculator Tool  
  - Role: Performs mathematical calculations for the Agent.  
  - Configuration: No parameters; invoked by Agent as needed.  
  - Inputs: Math expressions from Agent.  
  - Outputs: Calculation results.  
  - Edge Cases: Syntax errors in calculations.

- **Node:** Delete-event  
  - Type: Google Calendar Tool  
  - Role: Deletes specified events from Google Calendar.  
  - Configuration: Event ID and Calendar ID dynamically obtained from Agent inputs.  
  - Inputs: Event details from Agent.  
  - Outputs: Confirmation or error to Agent.  
  - Credentials: Google Calendar OAuth2 API.  
  - Edge Cases: Invalid event ID, insufficient permissions.

- **Node:** create-event  
  - Type: Google Calendar Tool  
  - Role: Creates new events in Google Calendar with specified start/end times and summary.  
  - Configuration: Dynamic start, end, calendar ID, and summary from Agent.  
  - Inputs: Event data from Agent.  
  - Outputs: Event creation confirmation.  
  - Credentials: Google Calendar OAuth2 API.  
  - Edge Cases: Date format errors, calendar access issues.

- **Node:** update-events  
  - Type: Google Calendar Tool  
  - Role: Updates existing calendar events with new data.  
  - Configuration: Event ID, calendar ID dynamically from Agent, with update fields specified.  
  - Inputs: Event update info from Agent.  
  - Outputs: Update confirmation.  
  - Credentials: Google Calendar OAuth2 API.  
  - Edge Cases: Event not found, permission errors.

- **Node:** Google Calendar  
  - Type: Google Calendar Tool  
  - Role: Retrieves multiple calendar events within a time range.  
  - Configuration: Calendar set to "admin@gmail.com", timeMin and timeMax dynamic from Agent.  
  - Inputs: Query parameters from Agent.  
  - Outputs: List of events.  
  - Credentials: Google Calendar OAuth2 API.  
  - Edge Cases: Large data sets, API limits.

---

#### 2.4 Telegram Output and User Feedback

- **Overview:** Sends the AI agent’s final responses back to the Telegram user and provides typing feedback during processing.
- **Nodes Involved:** `Telegram`, `send_typing`

**Node Details:**

- **Node:** Telegram  
  - Type: Telegram Node (Message Sending)  
  - Role: Sends text messages (AI-generated responses) back to the user on Telegram.  
  - Configuration: Text content set from Agent output, chat ID from the Telegram message sender ID. Attribution disabled for cleaner output.  
  - Inputs: Final output from Agent.  
  - Outputs: Message delivery side-effect.  
  - Credentials: Telegram API "Shadowark AI".  
  - Edge Cases: Failures if chat ID changes or Telegram API errors.

- **Node:** send_typing  
  - (Detailed above in 2.3 but also part of feedback loop.)

---

### 3. Summary Table

| Node Name       | Node Type                           | Functional Role                      | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                               |
|-----------------|-----------------------------------|------------------------------------|------------------------|-----------------------|---------------------------------------------------------------------------------------------------------------------------|
| chatbot         | Telegram Trigger                  | Input reception from Telegram      | External (Telegram API) | Agent                 |                                                                                                                           |
| Agent           | Langchain Agent                  | AI processing and orchestration    | chatbot, OpenAI Model, Simple Memory, tools | Telegram             |                                                                                                                           |
| OpenAI Model    | Langchain Chat OpenAI            | GPT-4 language model               | Agent (ai_languageModel) | Agent                 |                                                                                                                           |
| Simple Memory   | Langchain Memory Buffer Window   | Conversation memory management     | Agent (ai_memory)       | Agent                 |                                                                                                                           |
| send_typing     | Telegram Tool                   | Shows "typing..." indicator         | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| Search_tool     | HTTP Request Tool               | Real-time internet search          | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| Execute Command | Execute Command Tool            | Runs shell/system commands         | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| Calculator     | Langchain Calculator Tool       | Performs calculations              | Agent (ai_tool)         | Agent                 | you have full access to the calculator                                                                                     |
| Delete-event    | Google Calendar Tool            | Deletes calendar events            | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| create-event    | Google Calendar Tool            | Creates calendar events            | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| update-events   | Google Calendar Tool            | Updates calendar events            | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| Google Calendar | Google Calendar Tool            | Retrieves calendar events          | Agent (ai_tool)         | Agent                 |                                                                                                                           |
| Telegram        | Telegram Node                  | Sends messages to Telegram user    | Agent                   |                       |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook for receiving all updates (`*`).  
   - Set `chatIds` to `831690003` to filter messages.  
   - OAuth credentials: Link your Telegram bot API credentials ("Shadowark AI").  
   - Position: Start node.

2. **Create Langchain Agent Node**  
   - Type: Langchain Agent  
   - Input: Message text from Telegram trigger (`={{ $json.message.text }}`).  
   - System message: Paste the detailed AI persona prompt as in the original (QuantumDefender AI cybersecurity assistant).  
   - Prompt type: `define`.  
   - Version: 1.7.  
   - Connect input from Telegram Trigger.

3. **Create OpenAI Model Node**  
   - Type: Langchain Chat OpenAI  
   - Model: GPT-4.1-nano.  
   - Credentials: Link your OpenAI API credentials.  
   - Connect output to Agent node as language model input.

4. **Create Simple Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Session key: `={{ $('chatbot').item.json.message.from.id }}` to maintain per-user context.  
   - Connect memory input/output to Agent node.

5. **Create send_typing Node**  
   - Type: Telegram Tool  
   - Operation: Send Chat Action `typing...`.  
   - Chat ID: Fixed to `831690003` or dynamic from incoming message if preferred.  
   - Credentials: Telegram API credentials.  
   - Connect as an AI tool input to Agent to trigger typing indicator at start of processing.

6. **Create Search_tool Node**  
   - Type: HTTP Request Tool  
   - Method: POST to `https://api.langsearch.com/v1/web-search`.  
   - Headers: Authorization with Langsearch API key.  
   - Body parameters: `query` from Agent input, freshness `noLimit`, summary `true`, count `10`.  
   - Connect as AI tool input to Agent.

7. **Create Execute Command Node**  
   - Type: Execute Command Tool  
   - Parameters: Command and executeOnce flag dynamically passed from Agent.  
   - Connect as AI tool input.

8. **Create Calculator Node**  
   - Type: Langchain Calculator Tool  
   - No special parameters.  
   - Connect as AI tool input.

9. **Create Google Calendar Nodes**  
   - Four nodes: Delete-event, create-event, update-events, Google Calendar.  
   - Configure each with dynamic event IDs, calendar IDs, start/end times, and other fields passed from Agent.  
   - Use Google Calendar OAuth2 credentials.  
   - Connect all as AI tool inputs to Agent.

10. **Create Telegram Output Node**  
    - Type: Telegram Node (send message).  
    - Text: Set from Agent output (`={{ $('Agent').item.json.output }}`).  
    - Chat ID: Dynamic from Telegram trigger message sender ID (`={{ $('chatbot').item.json.message.from.id }}`).  
    - Credentials: Telegram API credentials.  
    - Connect Agent output to Telegram node.

11. **Wire all AI tools to Agent**  
    - Connect outputs of send_typing, Search_tool, Execute Command, Calculator, and Google Calendar nodes as AI tool inputs into Agent node.

12. **Validate all credentials and webhook URLs**  
    - Ensure all API credentials (OpenAI, Telegram, Google Calendar, Langsearch) are valid and authorized.  
    - Test webhook connectivity for Telegram trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The AI Agent "QuantumDefender AI" is designed with strict ethical and operational constraints to avoid illegal or harmful actions. It operates in a secure temporary folder environment and prioritizes safe execution.               | System message in Agent node prompt.                                                           |
| Telegram typing indicator is used consistently to improve user experience during multi-step processing.                                                                                                                        | send_typing node description.                                                                  |
| Langsearch API key used for real-time web search must be valid and authorized. Ensure key renewal and monitor API usage quotas.                                                                                                 | Search_tool node configuration.                                                                |
| Google Calendar integration requires OAuth2 credentials with proper scope to manage calendar events. Calendar ID format must be a valid email address.                                                                          | Google Calendar nodes configuration.                                                           |
| Calculator tool enables on-the-fly mathematical reasoning and is fully accessible to the AI agent.                                                                                                                              | Calculator node sticky note: "you have full access to the calculator."                         |
| Telegram user chat ID is fixed to `831690003` for message filtering and replies. Modify if expanding to multiple users.                                                                                                         | chatbot node and Telegram output node configuration.                                          |
| The workflow uses Langchain version-specific nodes (Agent 1.7, OpenAI Model 1.0, Memory 1.3) which must be compatible with your n8n instance.                                                                                   | Node version information in node details.                                                     |

---

**Disclaimer:** The content provided is exclusively derived from an automated n8n workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.