Telegram AI Assistant with Gmail, Google Calendar, Google Sheets & MCP Tools

https://n8nworkflows.xyz/workflows/telegram-ai-assistant-with-gmail--google-calendar--google-sheets---mcp-tools-4902


# Telegram AI Assistant with Gmail, Google Calendar, Google Sheets & MCP Tools

### 1. Workflow Overview

This workflow implements a Telegram AI Assistant integrated with Gmail, Google Calendar, Google Sheets, and MCP (Modular Chatbot Platform) tools to provide a conversational AI experience enriched with various productivity tools. It listens for Telegram messages, processes them through an AI agent powered by Google Gemini Chat Model and LangChain technology, manages conversational memory, and interacts with Gmail, Google Calendar, and Google Sheets via MCP tools.

**Target Use Cases:**
- Receiving user queries from Telegram and responding with AI-generated answers.
- Accessing and manipulating Gmail messages, Google Calendar events, and Google Sheets data dynamically.
- Leveraging conversational memory to maintain context in dialogues.
- Using MCP tools to modularize and streamline AI tool calls and server triggers.

**Logical Blocks:**

- **1.1 Input Reception:** Telegram Trigger node listens for incoming Telegram messages.
- **1.2 AI Processing:** AI Agent node powered by Google Gemini Chat Model and LangChain processes input, augmented by Simple Memory.
- **1.3 MCP Tools Integration:** MCP Client and MCP Server Trigger nodes act as intermediaries to invoke Gmail, Google Calendar, and Google Sheets tools.
- **1.4 External Service Interaction:** Gmail, Google Calendar, and Google Sheets nodes handle actual API calls for email, calendar events, and spreadsheet data.
- **1.5 Output Delivery:** Telegram node sends AI-generated responses back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming Telegram messages as the entry point to the workflow.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - *Type & Role:* Trigger node that listens for Telegram messages via webhook.  
    - *Configuration:* Uses a webhook ID to receive external Telegram updates. No filters configured, accepts all incoming messages.  
    - *Expressions/Variables:* Outputs message data for AI processing.  
    - *Connections:* Output connected to AI Agent main input.  
    - *Version Requirements:* Uses Telegram Trigger v1.2.  
    - *Potential Failures:* Webhook connectivity issues, Telegram API downtime, invalid webhook setup.  
    - *Sub-workflow:* None.

#### 2.2 AI Processing

- **Overview:**  
  Processes Telegram input with an AI Agent using Google Gemini Chat Model and LangChain framework, maintaining conversational context with Simple Memory.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model  
  - Simple Memory

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* LangChain AI Agent node orchestrates AI tool usage and memory.  
    - *Configuration:* Connected to Google Gemini Chat Model as language model; uses Simple Memory for context buffering; connected to MCP Client tool for external API calls.  
    - *Expressions/Variables:* Receives Telegram message as input; outputs AI responses to Telegram node.  
    - *Connections:*  
      - Input from Telegram Trigger main output.  
      - AI language model input from Google Gemini Chat Model.  
      - AI memory input from Simple Memory node.  
      - AI tool input/output with MCP Client.  
      - Output connected to Telegram node.  
    - *Version:* v1.8  
    - *Failures:* Model API errors, memory context corruption, expression evaluation errors, connection failures with MCP Client.  
    - *Sub-workflow:* None.

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node providing chat completions using Google Gemini technology.  
    - *Configuration:* Default settings; no custom parameters detailed.  
    - *Connections:* Outputs to AI Agent AI language model input.  
    - *Version:* v1  
    - *Failures:* API authentication errors, rate limits, network issues.  
    - *Sub-workflow:* None.

  - **Simple Memory**  
    - *Type & Role:* Buffer window memory node to maintain recent conversation history for context.  
    - *Configuration:* Default buffer window settings (e.g., number of messages to retain).  
    - *Connections:* Outputs to AI Agent AI memory input.  
    - *Version:* v1.3  
    - *Failures:* Memory overflow or misconfiguration, data corruption.  
    - *Sub-workflow:* None.

#### 2.3 MCP Tools Integration

- **Overview:**  
  Uses MCP Client and MCP Server Trigger nodes to modularize and route AI tool calls for Gmail, Google Calendar, and Google Sheets.

- **Nodes Involved:**  
  - MCP Client  
  - MCP Server Trigger

- **Node Details:**

  - **MCP Client**  
    - *Type & Role:* LangChain MCP Client Tool node, making AI tool calls from the AI Agent.  
    - *Configuration:* Default; designed to receive AI Agent calls and forward to MCP Server Trigger.  
    - *Connections:* Input from AI Agent’s ai_tool output; output to MCP Server Trigger ai_tool input.  
    - *Version:* v1  
    - *Failures:* Communication errors with MCP Server Trigger, misrouting requests.  
    - *Sub-workflow:* None.

  - **MCP Server Trigger**  
    - *Type & Role:* LangChain MCP Server Trigger node that listens for AI tool calls and dispatches them to specific tools like Gmail, Calendar, Sheets.  
    - *Configuration:* Associated with a webhook ID allowing external invocation.  
    - *Connections:* Receives ai_tool inputs from Gmail, Gmail1, Gmail2, Google Calendar nodes, contact data (Google Sheets), and sends back results.  
    - *Version:* v1  
    - *Failures:* Webhook errors, timeout, invalid tool invocation.  
    - *Sub-workflow:* None.

#### 2.4 External Service Interaction

- **Overview:**  
  Actual API interactions with Gmail, Google Calendar, and Google Sheets services through their respective nodes, invoked by MCP Server Trigger.

- **Nodes Involved:**  
  - Gmail, Gmail1, Gmail2  
  - Google Calendar, Google Calendar1, Google Calendar2  
  - contact data (Google Sheets)

- **Node Details:**

  - **Gmail, Gmail1, Gmail2**  
    - *Type & Role:* Gmail Tool nodes performing Gmail API operations (read, send, manage emails).  
    - *Configuration:* OAuth2 credentials for Gmail; webhook IDs for triggering.  
    - *Connections:* Inputs from MCP Server Trigger ai_tool outputs; no outputs chained in workflow (assumed responses handled via MCP Server Trigger).  
    - *Version:* v2.1  
    - *Failures:* Authentication failure, quota limits, invalid queries.  
    - *Sub-workflow:* None.

  - **Google Calendar, Google Calendar1, Google Calendar2**  
    - *Type & Role:* Google Calendar Tool nodes for calendar event management.  
    - *Configuration:* OAuth2 credentials for Google Calendar API.  
    - *Connections:* Inputs from MCP Server Trigger ai_tool output.  
    - *Version:* v1.3  
    - *Failures:* Authentication errors, API limits, invalid parameters.  
    - *Sub-workflow:* None.

  - **contact data (Google Sheets)**  
    - *Type & Role:* Google Sheets Tool node to access spreadsheet data.  
    - *Configuration:* OAuth2 credentials for Google Sheets API.  
    - *Connections:* Input from MCP Server Trigger ai_tool output.  
    - *Version:* v4.6  
    - *Failures:* Sheet access issues, permission errors, invalid ranges.  
    - *Sub-workflow:* None.

#### 2.5 Output Delivery

- **Overview:**  
  Sends AI-generated replies back to the Telegram user.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - *Type & Role:* Telegram node to send messages back to users.  
    - *Configuration:* Uses Telegram bot credentials; webhook ID for sending messages.  
    - *Connections:* Receives main input from AI Agent output.  
    - *Version:* v1.2  
    - *Failures:* Telegram API errors, message formatting issues, rate limits.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name             | Node Type                                | Functional Role                  | Input Node(s)          | Output Node(s)         | Sticky Note                                      |
|-----------------------|----------------------------------------|--------------------------------|-----------------------|------------------------|-------------------------------------------------|
| Telegram Trigger       | telegramTrigger (v1.2)                  | Entry point for Telegram input | -                     | AI Agent               |                                                 |
| AI Agent              | langchain.agent (v1.8)                  | AI processing & orchestration   | Telegram Trigger, Google Gemini Chat Model, Simple Memory | Telegram, MCP Client |                                                 |
| Google Gemini Chat Model | langchain.lmChatGoogleGemini (v1)     | Language model for AI Agent     | -                     | AI Agent               |                                                 |
| Simple Memory         | langchain.memoryBufferWindow (v1.3)    | Conversational memory buffer    | -                     | AI Agent               |                                                 |
| MCP Client            | langchain.mcpClientTool (v1)            | Calls AI tools via MCP Server   | AI Agent               | MCP Server Trigger     |                                                 |
| MCP Server Trigger    | langchain.mcpTrigger (v1)                | Receives tool calls, dispatches | Gmail, Gmail1, Gmail2, Google Calendar nodes, contact data | Gmail, Google Calendar, Google Sheets |                                                 |
| Gmail                 | gmailTool (v2.1)                        | Gmail API interaction           | MCP Server Trigger    | -                      |                                                 |
| Gmail1                | gmailTool (v2.1)                        | Gmail API interaction           | MCP Server Trigger    | -                      |                                                 |
| Gmail2                | gmailTool (v2.1)                        | Gmail API interaction           | MCP Server Trigger    | -                      |                                                 |
| Google Calendar       | googleCalendarTool (v1.3)               | Google Calendar API interaction | MCP Server Trigger    | -                      |                                                 |
| Google Calendar1      | googleCalendarTool (v1.3)               | Google Calendar API interaction | MCP Server Trigger    | -                      |                                                 |
| Google Calendar2      | googleCalendarTool (v1.3)               | Google Calendar API interaction | MCP Server Trigger    | -                      |                                                 |
| contact data          | googleSheetsTool (v4.6)                  | Google Sheets API interaction   | MCP Server Trigger    | -                      |                                                 |
| Telegram              | telegram (v1.2)                         | Sends messages back to Telegram | AI Agent               | -                      |                                                 |
| Sticky Note           | stickyNote (v1)                         | Annotation                     | -                     | -                      |                                                 |
| Sticky Note1          | stickyNote (v1)                         | Annotation                     | -                     | -                      |                                                 |
| Sticky Note2          | stickyNote (v1)                         | Annotation                     | -                     | -                      |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger (v1.2)  
   - Configure webhook credentials for your Telegram Bot API.  
   - Leave default settings to listen to all incoming messages.  
   - Position at workflow start.

2. **Create Google Gemini Chat Model node**  
   - Type: langchain.lmChatGoogleGemini (v1)  
   - No special parameters needed if defaults suffice.  
   - Position near AI Agent node.

3. **Create Simple Memory node**  
   - Type: langchain.memoryBufferWindow (v1.3)  
   - Configure buffer window size as needed (default recommended).  
   - Position near AI Agent node.

4. **Create AI Agent node**  
   - Type: langchain.agent (v1.8)  
   - Connect the Telegram Trigger main output to AI Agent main input.  
   - In AI Agent settings, set:  
     - Language Model input from Google Gemini Chat Model node output.  
     - AI Memory input from Simple Memory output.  
     - AI Tool input/output from MCP Client node (to be created).  
   - Position centrally.

5. **Create MCP Client node**  
   - Type: langchain.mcpClientTool (v1)  
   - Connect AI Agent ai_tool output to MCP Client ai_tool input.  
   - Position between AI Agent and MCP Server Trigger.

6. **Create MCP Server Trigger node**  
   - Type: langchain.mcpTrigger (v1)  
   - Setup webhook ID for external communication.  
   - Connect MCP Client ai_tool output to MCP Server Trigger ai_tool input.  
   - Position after MCP Client.

7. **Create Gmail nodes (Gmail, Gmail1, Gmail2)**  
   - Type: gmailTool (v2.1)  
   - Configure OAuth2 Gmail credentials for each node.  
   - Connect MCP Server Trigger ai_tool output to each Gmail node ai_tool input.  
   - Position grouped under MCP Server Trigger.

8. **Create Google Calendar nodes (Google Calendar, Google Calendar1, Google Calendar2)**  
   - Type: googleCalendarTool (v1.3)  
   - Configure OAuth2 Google Calendar credentials.  
   - Connect MCP Server Trigger ai_tool output to each Google Calendar node ai_tool input.  
   - Position grouped.

9. **Create Google Sheets node ("contact data")**  
   - Type: googleSheetsTool (v4.6)  
   - Configure OAuth2 Google Sheets credentials.  
   - Connect MCP Server Trigger ai_tool output to the Google Sheets node ai_tool input.  
   - Position near calendar nodes.

10. **Create Telegram node**  
    - Type: telegram (v1.2)  
    - Configure Telegram Bot credentials and webhook ID.  
    - Connect AI Agent main output to Telegram main input.  
    - Position at workflow end.

11. **Wire all connections exactly as per above**:  
    - Telegram Trigger → AI Agent main input  
    - Google Gemini Chat Model → AI Agent ai_languageModel input  
    - Simple Memory → AI Agent ai_memory input  
    - AI Agent ai_tool output → MCP Client ai_tool input  
    - MCP Client ai_tool output → MCP Server Trigger ai_tool input  
    - MCP Server Trigger ai_tool output → Gmail, Google Calendar, Google Sheets ai_tool inputs  
    - AI Agent main output → Telegram main input

12. **Set credentials** for all external service nodes (Telegram, Gmail, Google Calendar, Google Sheets) using OAuth2 or API keys as required.

13. **Test webhook connectivity** for Telegram Trigger and MCP Server Trigger nodes.

14. **Run workflow** and verify Telegram messages are processed and responded to with AI-generated answers, leveraging Gmail, Calendar, and Sheets data when applicable.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| n8n version used supports LangChain and Google Gemini Chat Model nodes (ensure n8n is up to date).  | n8n official docs: https://docs.n8n.io/                   |
| OAuth2 credentials must be configured properly for Gmail, Google Calendar, Google Sheets, and Telegram integrations. | Refer to each API provider’s OAuth2 setup guides.         |
| MCP tools modularize AI calls, enabling easy extension or replacement of external tools.            | MCP tools GitHub: https://github.com/n8n-io/n8n           |
| Telegram Bot setup requires webhook URL registration with Telegram BotFather.                        | Telegram Bot API docs: https://core.telegram.org/bots/api  |

---

*Disclaimer: This document is generated based exclusively on an automated n8n workflow export. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.*