Stacey – Your Telegram AI Assistant (Powered by MCP, Gemini & Google Tools)

https://n8nworkflows.xyz/workflows/stacey---your-telegram-ai-assistant--powered-by-mcp--gemini---google-tools--4025


# Stacey – Your Telegram AI Assistant (Powered by MCP, Gemini & Google Tools)

### 1. Workflow Overview

This workflow, titled **"Stacey – Your Telegram AI Assistant (Powered by MCP, Gemini & Google Tools)"**, implements an intelligent AI assistant inside Telegram. Stacey listens to Telegram messages (both text and voice), interprets user intents using AI models (Google Gemini by default, optionally OpenAI GPT-4o), and routes commands to various integrated tools through MCP (Modular Command Protocol) logic. The tools include Gmail, Google Calendar, a content creation sub-workflow, a calculator, Tavily web search, and Google Sheets for contact management.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Preprocessing:** Telegram Trigger, message type switch, file download, transcription.
- **1.2 AI Natural Language Understanding:** AI Agent (Gemini model), user context memory.
- **1.3 MCP Command Routing:** MCP Server Trigger routes user intent to specific tool nodes.
- **1.4 Tools Execution:** Gmail, Google Calendar, Google Sheets, Tavily, Calculator, Content Creator sub-workflow.
- **1.5 Response Handling:** Response parsing and sending replies back to Telegram.

Each block is composed of multiple nodes tightly connected by data flow to achieve seamless interaction between user, AI understanding, and tool execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

**Overview:**  
This block listens for incoming messages on Telegram, distinguishes between text and voice inputs, downloads voice messages if present, and optionally transcribes audio into text.

**Nodes Involved:**  
- Telegram Trigger1  
- Switch input  
- Download File (disabled by default)  
- Transcribe (disabled by default)  
- Set 'Text'  

**Node Details:**  

- **Telegram Trigger1**  
  - Type: Telegram Trigger  
  - Role: Entry point for all Telegram messages (text, voice) to the workflow.  
  - Configuration: Connects to Telegram Bot Token credential. Listens to messages and updates.  
  - Inputs: External Telegram webhook events  
  - Outputs: Message data passed to Switch input node  
  - Edge Cases: Missing webhook connection, invalid bot token, Telegram privacy mode restrictions.  

- **Switch input**  
  - Type: Switch  
  - Role: Branching logic to route based on message type (voice or text).  
  - Configuration: Checks if incoming message contains a file (voice message) or plain text.  
  - Inputs: Telegram Trigger1 output  
  - Outputs: Two branches: Download File if voice message; Set 'Text' if text message.  
  - Edge Cases: Unexpected message formats, unsupported file types.  

- **Download File** (disabled)  
  - Type: Telegram  
  - Role: Downloads voice message file from Telegram servers.  
  - Configuration: Uses Telegram API to download the voice file.  
  - Inputs: Voice message from Switch input node  
  - Outputs: File binary data to Transcribe node  
  - Edge Cases: API errors, file not found, disabled by default (requires OpenAI Whisper key).  

- **Transcribe** (disabled)  
  - Type: OpenAI (Whisper)  
  - Role: Transcribes audio file to text using Whisper API.  
  - Configuration: Requires OpenAI Whisper API key, configured for speech-to-text.  
  - Inputs: Binary audio from Download File  
  - Outputs: Transcribed text to AI Agent node  
  - Edge Cases: API failures, audio quality issues, disabled if no key provided.  

- **Set 'Text'**  
  - Type: Set  
  - Role: Prepares text input for AI processing by standardizing input data structure.  
  - Configuration: Extracts and formats text from Telegram message.  
  - Inputs: Switch input text branch  
  - Outputs: Clean text input to AI Agent  
  - Edge Cases: Empty or malformed text messages.  

---

#### 2.2 AI Natural Language Understanding

**Overview:**  
This block uses a large language model to interpret the user's intent, maintain conversation context, and decide which MCP tool to invoke.

**Nodes Involved:**  
- Gemini model  
- User Context  
- AI Agent  
- MCP Client  

**Node Details:**  

- **Gemini model**  
  - Type: Google Gemini Chat Language Model  
  - Role: Provides the core AI understanding and reasoning capabilities.  
  - Configuration: Uses preconfigured Google Gemini API credentials. Can be replaced with OpenAI GPT-4o for enhanced performance.  
  - Inputs: AI Agent node’s language model input  
  - Outputs: AI Agent receives generated completions  
  - Edge Cases: API rate limits, connectivity issues, model syntax errors.  

- **User Context**  
  - Type: Memory Buffer Window  
  - Role: Maintains conversation history to enable context-aware AI responses.  
  - Configuration: Stores a sliding window of past messages for context in AI Agent.  
  - Inputs: AI Agent conversation data  
  - Outputs: Contextualized input to AI Agent  
  - Edge Cases: Memory overflow, context loss if misconfigured.  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates AI interaction, selects MCP tools based on intent, and formats tool invocation commands.  
  - Configuration: System prompt defines Stacey’s personality and delegation logic, including tool usage rules and examples.  
  - Inputs: User text (from Set 'Text' or Transcribe), user context, language model (Gemini)  
  - Outputs: Parsed AI response to MCP Client and response parser  
  - Edge Cases: Prompt misconfiguration, unexpected AI outputs, syntax errors in generated commands.  

- **MCP Client**  
  - Type: MCP Client Tool  
  - Role: Sends AI-decided commands to MCP Server Trigger for tool execution.  
  - Configuration: Relays parsed instructions from AI Agent downstream.  
  - Inputs: AI Agent’s ai_tool output  
  - Outputs: MCP Server Trigger node  
  - Edge Cases: Command format errors, connectivity to MCP server.  

---

#### 2.3 MCP Command Routing

**Overview:**  
Receives commands from the MCP Client and triggers the appropriate tool node to perform user-requested actions.

**Nodes Involved:**  
- MCP Server Trigger  

**Node Details:**  

- **MCP Server Trigger**  
  - Type: MCP Trigger  
  - Role: Receives MCP Client commands and routes them to the correct tool node (e.g., Gmail, Calendar, Content Creator).  
  - Configuration: Configured with webhook endpoint and listens for MCP commands.  
  - Inputs: MCP Client commands routed to ai_tool input  
  - Outputs: Routes to tool nodes based on command type  
  - Edge Cases: Misrouted commands, webhook failures, authentication issues.  

---

#### 2.4 Tools Execution

**Overview:**  
This block contains various integrated tools that execute user commands like sending emails, managing calendar events, creating content, searching the web, calculating math, and managing contacts.

**Nodes Involved:**  
- Send Email (Gmail Tool)  
- Email Reply (Gmail Tool)  
- Get Emails (Gmail Tool)  
- Create Draft (Gmail Tool)  
- Get Labels (Gmail Tool)  
- Label Emails (Gmail Tool)  
- Mark Unread (Gmail Tool)  
- Create Event (Google Calendar Tool)  
- Create Event with Attendee (Google Calendar Tool)  
- Get Events (Google Calendar Tool)  
- Update Event (Google Calendar Tool)  
- Delete Event (Google Calendar Tool)  
- Content Creator Agent (LangChain Tool Workflow) — Sub-workflow  
- Tavily (LangChain HTTP Request Tool)  
- Calculate Maths (LangChain Calculator Tool)  
- Update Contact Data (Google Sheets Tool)  
- Search Contact Data (Google Sheets Tool)  

**Node Details (sample highlights):**  

- **Send Email**  
  - Type: Gmail Tool  
  - Role: Sends emails based on AI-generated content and parameters.  
  - Config: OAuth2 with Gmail scope, parameters dynamically set by MCP Server Trigger.  

- **Create Event**  
  - Type: Google Calendar Tool  
  - Role: Creates calendar events with or without attendees.  
  - Config: OAuth2 with Calendar scope, event details dynamically set.  

- **Content Creator Agent**  
  - Type: LangChain Tool Workflow (Sub-workflow)  
  - Role: Specialized tool invoked for content generation tasks like blog posts or emails.  
  - Config: Imported separately; triggered through MCP logic.  
  - Integration: Connected via MCP Server Trigger.  

- **Tavily**  
  - Type: LangChain HTTP Request  
  - Role: Performs live web searches to enrich responses.  
  - Config: Requires Tavily API key.  

- **Calculate Maths**  
  - Type: LangChain Calculator Tool  
  - Role: Executes mathematical calculations requested by the user.  

- **Update/Search Contact Data**  
  - Type: Google Sheets Tool  
  - Role: Reads and updates contact information stored in Google Sheets.  

Edge cases for tools include authentication failures, insufficient permissions, API limits, malformed inputs, and service unavailability.

---

#### 2.5 Response Handling

**Overview:**  
This block parses AI-generated responses and sends natural language replies back to the Telegram user.

**Nodes Involved:**  
- response parser (Code node)  
- Response1 (Telegram)  

**Node Details:**  

- **response parser**  
  - Type: Code  
  - Role: Processes AI Agent output to format a user-friendly response message.  
  - Config: Custom JavaScript to extract text, handle JSON or structured data if needed.  
  - Inputs: AI Agent main output  
  - Outputs: Formatted message text to Response1 node  
  - Edge Cases: Parsing errors, unexpected AI output format.  

- **Response1**  
  - Type: Telegram  
  - Role: Sends message replies back to the user via Telegram bot.  
  - Config: Uses Telegram API credential; sends text messages to the chat ID from trigger.  
  - Inputs: Parsed response from response parser  
  - Outputs: Message sent confirmation  
  - Edge Cases: Telegram API errors, invalid chat IDs, message size limits.  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                        | Input Node(s)             | Output Node(s)                     | Sticky Note                                            |
|-------------------------|--------------------------------|-------------------------------------|---------------------------|----------------------------------|--------------------------------------------------------|
| Telegram Trigger1        | Telegram Trigger               | Receives Telegram messages           | -                         | Switch input                     | Ensure correct bot token and webhook                    |
| Switch input            | Switch                        | Routes message by type (text/voice) | Telegram Trigger1          | Download File, Set 'Text'         |                                                        |
| Download File           | Telegram (disabled)            | Downloads voice message file         | Switch input              | Transcribe                      | Optional: requires OpenAI Whisper API key               |
| Transcribe              | OpenAI Whisper (disabled)      | Transcribes audio to text            | Download File             | AI Agent                       | Optional: enable for voice transcription                 |
| Set 'Text'              | Set                           | Prepares text input for AI           | Switch input              | AI Agent                       |                                                        |
| Gemini model            | Google Gemini LM               | AI language model                    | AI Agent (lm input)       | AI Agent                       | Replaceable with OpenAI GPT-4o                           |
| User Context            | Memory Buffer Window           | Maintains conversation context      | AI Agent                  | AI Agent                       |                                                        |
| AI Agent                | LangChain Agent                | Core AI logic and tool selection    | Set 'Text', Transcribe, User Context, Gemini model | response parser, MCP Client, AI Agent (main) | Main AI orchestrator; customize system prompt            |
| MCP Client              | MCP Client Tool                | Sends commands to MCP Server         | AI Agent                   | MCP Server Trigger             |                                                        |
| MCP Server Trigger      | MCP Trigger                   | Routes commands to tools             | MCP Client, various tools | Tool nodes                     |                                                        |
| Send Email              | Gmail Tool                    | Sends email                        | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Email Reply             | Gmail Tool                    | Replies to email                    | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Get Emails              | Gmail Tool                    | Retrieves emails                   | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Create Draft            | Gmail Tool                    | Creates email draft                 | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Get Labels              | Gmail Tool                    | Lists Gmail labels                 | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Label Emails            | Gmail Tool                    | Labels emails                     | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Mark Unread             | Gmail Tool                    | Marks emails unread                | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Create Event            | Google Calendar Tool          | Creates calendar event             | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Create Event with Attendee | Google Calendar Tool        | Creates event with attendees       | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Get Events              | Google Calendar Tool          | Lists calendar events              | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Update Event            | Google Calendar Tool          | Updates calendar event             | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Delete Event            | Google Calendar Tool          | Deletes calendar event             | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| Content Creator Agent   | LangChain Tool Workflow       | Content generation sub-workflow    | MCP Server Trigger         | MCP Server Trigger              | Imported sub-workflow handling blog/email writing       |
| Tavily                  | LangChain HTTP Request        | Performs live web search           | MCP Server Trigger         | MCP Server Trigger              | Optional; requires Tavily API key                        |
| Calculate Maths         | LangChain Calculator Tool     | Performs calculations              | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| update contact data     | Google Sheets Tool            | Updates contact info in Sheets     | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| search contact data     | Google Sheets Tool            | Searches contact info in Sheets    | MCP Server Trigger         | MCP Server Trigger              |                                                        |
| response parser         | Code                          | Parses AI output for reply formatting | AI Agent                  | Response1                      |                                                        |
| Response1               | Telegram Send Message         | Sends reply to Telegram user       | response parser           | -                              | Ensure correct Telegram bot token                       |
| Sticky Note2            | Sticky Note                   | -                                   | -                         | -                              |                                                        |
| Sticky Note4            | Sticky Note                   | -                                   | -                         | -                              |                                                        |
| Sticky Note5            | Sticky Note                   | -                                   | -                         | -                              |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot Token credential (from BotFather).  
   - Set to listen for message updates (text & voice).  

2. **Add Switch Node ("Switch input")**  
   - Add a Switch node connected to Telegram Trigger.  
   - Configure to check if incoming message contains a voice file or text.  
   - Create two branches: one for voice messages, one for text messages.  

3. **Add "Download File" Telegram Node (optional)**  
   - Disabled by default; enable if voice transcription desired.  
   - Connect voice branch from Switch to this node.  
   - Configure to download voice message file from Telegram servers.  

4. **Add "Transcribe" OpenAI Node (optional)**  
   - Disabled by default; enable if voice transcription desired.  
   - Connect Download File node output to Transcribe node.  
   - Configure with OpenAI Whisper API key.  
   - Set to transcribe audio to text.  

5. **Add "Set 'Text'" Node**  
   - Connect text branch of Switch node to this Set node.  
   - Configure to extract and clean text input for AI processing.  

6. **Add "Gemini model" Node**  
   - Type: Google Gemini Chat Language Model  
   - Configure with Google API credentials for Gemini.  
   - Alternatively, prepare OpenAI Chat node with GPT-4o credentials for upgrade.  

7. **Add "User Context" Node**  
   - Type: Memory Buffer Window  
   - Configure to store conversation history for context-aware AI.  

8. **Add "AI Agent" Node**  
   - Connect inputs from Set 'Text', Transcribe (if enabled), User Context, and Gemini model.  
   - Configure systemMessage prompt defining Stacey’s role, behavior, and tool usage rules.  
   - Enable outputs: ai_tool, main (for response parsing), ai_languageModel, ai_memory.  

9. **Add "MCP Client" Node**  
   - Connect AI Agent’s ai_tool output to MCP Client node.  
   - This will forward commands to MCP Server Trigger.  

10. **Add "MCP Server Trigger" Node**  
    - Configure webhook URL and connect MCP Client’s output here.  
    - This node routes commands to various tool nodes based on MCP logic.  

11. **Add Tool Nodes and Connect to MCP Server Trigger:**  
    - Gmail Tools: Send Email, Email Reply, Get Emails, Create Draft, Get Labels, Label Emails, Mark Unread  
    - Google Calendar Tools: Create Event, Create Event with Attendee, Get Events, Update Event, Delete Event  
    - Google Sheets Tools: Update Contact Data, Search Contact Data  
    - Content Creator Agent: Import content_creator_tool.json as a separate workflow, connect via MCP Server Trigger  
    - Tavily HTTP Request (optional): Configure with API key for live web search  
    - Calculate Maths Tool: For math calculations  
    - Connect each tool node’s ai_tool input to MCP Server Trigger’s matching output.  

12. **Add "response parser" Code Node**  
    - Connect AI Agent main output to response parser.  
    - Add JavaScript code to parse AI response text for user-friendly output.  

13. **Add "Response1" Telegram Node**  
    - Connect response parser output to Response1.  
    - Configure with Telegram API credential and set to send messages back to the user.  

14. **Credentials Setup:**  
    - Add Telegram API credential with BotFather token.  
    - Add Google OAuth2 credentials with Gmail and Calendar scopes.  
    - Add Gemini or OpenAI API credentials.  
    - Optional: OpenAI Whisper API key for transcription.  
    - Optional: Tavily API key for web search.  

15. **Test the Workflow:**  
    - Send messages via your Telegram bot.  
    - Observe AI intent detection, tool execution, and response delivery.  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                             |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Gemini is used as a free AI alternative; upgrading to OpenAI GPT-4o improves reasoning and flexibility. | See step 4 customization instructions.                      |
| Content Creator is a modular sub-workflow and must be imported separately.                              | Import content_creator_tool.json and link in MCP Server.    |
| Telegram bot privacy mode can block message reception; set `/setprivacy` correctly in BotFather.       | Telegram Bot setup instructions.                             |
| OAuth scopes needed: Gmail modify, Calendar read/write scopes for full functionality.                   | Google Cloud Console configuration.                          |
| For voice transcription, OpenAI Whisper key is required; otherwise, disable related nodes.              | Optional voice transcription feature.                        |
| The MCP protocol allows modular extension of tools; add new tools by connecting to MCP Server Trigger. | Extend functionality easily by adding new MCP tool nodes.   |
| Created by David Olusola; free to use and resell with attribution.                                      | Follow on n8n Creator Page for updates and support.         |
| Troubleshooting tips included in the workflow description.                                              | Common errors include API auth, webhook setup, and prompt tuning. |

---

**Disclaimer:** The text above is derived solely from an n8n automated workflow created with n8n, respecting all content policies and handling only legal and public data.