Voice & Text Control for Home Assistant using Telegram, Whisper & Gemini

https://n8nworkflows.xyz/workflows/voice---text-control-for-home-assistant-using-telegram--whisper---gemini-8411


# Voice & Text Control for Home Assistant using Telegram, Whisper & Gemini

### 1. Workflow Overview

This workflow, titled **"Voice & Text Control for Home Assistant using Telegram, Whisper & Gemini,"** enables users to control their Home Assistant smart home environment through natural language commands sent via Telegram (text or voice) or the built-in n8n chat interface. It transcribes voice messages, normalizes input, uses AI models (Google Gemini) and a custom AI agent to interpret commands, and then triggers smart home actions through Home Assistant's MCP server. Responses are formatted and routed back to the user in the original channel with enhanced readability.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Captures incoming messages from Telegram (voice or text) and n8n chat.
- **1.2 Voice Processing:** Downloads voice messages, transcribes them using OpenAI Whisper, and maps transcription into a unified input text field.
- **1.3 Message Normalization:** Consolidates inputs from different sources, detects message origin, and prepares session context.
- **1.4 AI Processing:** Uses an AI agent with short-term memory and Google Gemini chat model to interpret commands and decide on Home Assistant actions.
- **1.5 Smart Home Integration:** Executes commands on Home Assistant via MCP client tools.
- **1.6 Response Formatting and Routing:** Beautifies AI responses for Telegram and sends back replies via Telegram or n8n chat webhook.
- **1.7 User Experience Enhancements:** Sends “typing…” actions to Telegram while processing to improve UX.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming messages from Telegram (both voice and text) and from n8n's built-in chat interface.
- **Nodes Involved:**  
  - Telegram Trigger  
  - When chat message received

##### Node Details:

- **Telegram Trigger**  
  - Type: Telegram Trigger (Webhook-based)  
  - Configuration: Listens for "message" updates only (text or voice). Uses Telegram API credentials named "HomeWhisper."  
  - Inputs: Incoming Telegram messages  
  - Outputs: Raw Telegram message JSON for further processing  
  - Failure points: Telegram API connectivity, webhook misconfiguration  
  - Sticky Note: Describes its function as listening for incoming Telegram messages (text or voice).

- **When chat message received**  
  - Type: n8n LangChain Chat Trigger  
  - Configuration: Public webhook, loads previous session memory for continuity  
  - Inputs: Messages from n8n's own chat frontend  
  - Outputs: Chat message JSON for normalization  
  - Failure points: Webhook availability, memory loading errors  
  - Sticky Note: Captures messages from n8n’s built-in chat and feeds them into normalization pipeline.

---

#### 1.2 Voice Processing

- **Overview:** Handles Telegram voice messages by downloading the audio file and transcribing it into text using OpenAI Whisper.
- **Nodes Involved:**  
  - Voice or Text (Set)  
  - If (Check for voice message)  
  - Get Voice File  
  - Speech to Text (OpenAI Whisper)  
  - Transcription to ChatInput (Set)

##### Node Details:

- **Voice or Text**  
  - Type: Set Node  
  - Configuration: Extracts text if present; otherwise, empty string  
  - Inputs: Telegram message JSON  
  - Outputs: Sets field `text` for downstream processing  
  - Failure points: Missing message object  
  - Sticky Note: Checks if message is voice → routes to STT; otherwise uses text directly. Branches if text missing.

- **If**  
  - Type: If  
  - Configuration: Checks if `message.text` is empty (i.e., voice message likely)  
  - Inputs: Output of Voice or Text  
  - Outputs: Two branches - voice processing or direct text processing  
  - Failure points: Expression failures if JSON structure unexpected

- **Get Voice File**  
  - Type: Telegram node (Get File)  
  - Configuration: Downloads voice file using `message.voice.file_id` from Telegram  
  - Inputs: Voice message JSON from Telegram Trigger  
  - Outputs: Binary audio file for transcription  
  - Failure points: Telegram file retrieval errors, file ID missing  
  - Sticky Note: Downloads the voice message from Telegram for transcription.

- **Speech to Text (OpenAI Whisper)**  
  - Type: LangChain OpenAI audio transcription  
  - Configuration: Uses OpenAI Whisper model to transcribe audio to text  
  - Inputs: Audio binary from Get Voice File  
  - Outputs: Transcribed text  
  - Credentials: OpenAI API account  
  - Failure points: API key limits, transcription timeouts

- **Transcription to ChatInput**  
  - Type: Set  
  - Configuration: Maps transcribed text to unified field `chatInput` for downstream AI processing  
  - Inputs: Transcription text  
  - Outputs: Structured JSON with `chatInput` field  
  - Failure points: None expected

---

#### 1.3 Message Normalization

- **Overview:** Normalizes input text from various sources (Telegram text, transcribed voice, n8n chat), detects message origin, assigns session identifiers, and flags voice messages.
- **Nodes Involved:**  
  - Process messages (Code)

##### Node Details:

- **Process messages**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Safely extracts text input from multiple possible fields (`inputText`, `chatInput`, `text`, `message.text`, etc.)  
    - Detects source platform (`telegram`, `n8n-chat`, or `chat`)  
    - Builds or infers a session ID based on message metadata  
    - Flags if input originated from voice message  
  - Inputs: Output from Transcription to ChatInput or When chat message received  
  - Outputs: JSON with normalized `inputText`, `source`, `sessionId`, and `isVoice` boolean  
  - Failure points: Expression errors if nodes referenced are missing or JSON is malformed  
  - Sticky Note: Normalizes inputText, detects source (chat/telegram), builds sessionId.

---

#### 1.4 AI Processing

- **Overview:** Core AI logic that interprets the normalized input using a LangChain AI agent powered by Google Gemini chat model, with short-term memory for context.
- **Nodes Involved:**  
  - Home Agent (AI Agent)  
  - Google Gemini Chat Model (Language model)  
  - Simple Memory (Memory buffer)  
  - Simple Memory1 (Memory buffer for chat messages)

##### Node Details:

- **Home Agent**  
  - Type: LangChain Agent node  
  - Configuration:  
    - Uses normalized text input as prompt  
    - Configured with a "define" prompt type (custom prompt setup)  
    - Receives AI language model and memory as inputs  
  - Inputs: Normalized message JSON, AI language model, memory buffer, and AI tool (Home Assistant connector)  
  - Outputs: AI-generated response JSON for routing  
  - Failure points: AI service downtime, prompt errors, memory loading issues  
  - Sticky Note: Core AI agent that orchestrates LLM and Home Assistant actions.

- **Google Gemini Chat Model**  
  - Type: LangChain Google PaLM Gemini Chat Model  
  - Configuration: Default parameters, authorized with Google Palm API credentials  
  - Inputs: Receives prompt from Home Agent  
  - Outputs: AI chat completions for intent understanding and response generation  
  - Failure points: API quota limits, authorization errors

- **Simple Memory & Simple Memory1**  
  - Type: LangChain Memory Buffer Window  
  - Configuration: Short-term memory to maintain conversational context  
  - Inputs: Connected to respective chat trigger nodes (Telegram and n8n chat)  
  - Outputs: Provides memory context to Home Agent  
  - Failure points: Memory overflow or data corruption (rare)

---

#### 1.5 Smart Home Integration

- **Overview:** Executes smart home commands as instructed by the AI agent by interacting with Home Assistant MCP server.
- **Nodes Involved:**  
  - Home assistant Connector

##### Node Details:

- **Home assistant Connector**  
  - Type: LangChain MCP Client Tool  
  - Configuration:  
    - Connects to Home Assistant MCP server via SSE endpoint `http://192.168.1.69:8123/mcp_server/sse`  
    - Includes tools for live context retrieval, turning devices on/off, adjusting lights, broadcasting messages  
    - Authenticates with HTTP Bearer token credentials  
  - Inputs: AI tool instructions from Home Agent  
  - Outputs: Execution results of smart home commands  
  - Failure points: Network issues, authentication failures, MCP server offline  
  - Sticky Note: Executes smart home actions: turn lights on/off, adjust brightness, broadcast messages via MCP.

---

#### 1.6 Response Formatting and Routing

- **Overview:** Beautifies AI responses into Telegram-compatible HTML formatted text, splits long messages into chunks, and routes responses either back to Telegram or the n8n chat frontend.
- **Nodes Involved:**  
  - Reply Router (If)  
  - Telegram Message Beautifier (Code)  
  - Telegram Send  
  - Respond to Webhook

##### Node Details:

- **Reply Router**  
  - Type: If  
  - Configuration: Routes responses based on source field (`telegram` or others)  
  - Inputs: Output from Home Agent  
  - Outputs: One branch to Telegram Message Beautifier, one to Respond to Webhook  
  - Failure points: Logic errors if source field missing or malformed  
  - Sticky Note: Routes response back to Telegram or n8n chat UI depending on origin.

- **Telegram Message Beautifier**  
  - Type: Code  
  - Configuration:  
    - Applies formatting rules to output text: converts markdown-like bold, bullets, inline code, and links to HTML tags  
    - Splits output into 4096-character chunks to respect Telegram message size limits  
  - Inputs: Text from Reply Router  
  - Outputs: Array of formatted message chunks  
  - Failure points: Regex failures if unexpected text formats  
  - Sticky Note: Formats output: bold, bullets, inline code, links – splits into chunks.

- **Telegram Send**  
  - Type: Telegram node (Send Message)  
  - Configuration:  
    - Sends formatted text chunks back to the original Telegram chat  
    - Uses HTML parse mode for rich formatting  
    - Reads chat ID from Telegram Trigger node for correct recipient  
  - Inputs: Chunks from Telegram Message Beautifier  
  - Outputs: Confirmation of message sent  
  - Failure points: Telegram API limits, chat ID missing  
  - Sticky Note: Sends formatted reply back to Telegram in HTML parse mode.

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Configuration: Streams AI response back to n8n chat frontend webhook as plain text  
  - Inputs: Output from Reply Router branch for non-Telegram sources  
  - Outputs: HTTP response to webhook caller  
  - Failure points: Webhook connectivity issues  
  - Sticky Note: Streams responses back to the n8n chat frontend webhook.

---

#### 1.7 User Experience Enhancements

- **Overview:** Sends a “typing…” chat action to Telegram while processing user input to provide feedback that the bot is working.
- **Nodes Involved:**  
  - Bot Is typing

##### Node Details:

- **Bot Is typing**  
  - Type: Telegram node (Send Chat Action)  
  - Configuration: Sends "typing" status to chat using chat ID from Telegram Trigger  
  - Inputs: Incoming Telegram message JSON  
  - Outputs: Confirmation of action sent  
  - Failure points: Telegram API errors  
  - Sticky Note: Sends “typing…” action back to Telegram for better UX feedback.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                              | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                                                  |
|---------------------------|----------------------------------|----------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger          | Telegram Trigger                 | Receives Telegram messages (text or voice)   |                            | Voice or Text, Bot Is typing   | Listens for incoming Telegram messages (text or voice).                                                                       |
| When chat message received| LangChain Chat Trigger           | Receives messages from n8n chat interface    |                            | Process messages               | Captures messages from n8n’s built-in chat and feeds into normalization pipeline.                                             |
| Voice or Text             | Set                             | Extracts text or empty string                  | Telegram Trigger            | If                            | Checks if the message is voice → routes to STT; otherwise uses text directly. Branches if text missing.                       |
| If                       | If                              | Checks if message text is empty (voice check) | Voice or Text               | Get Voice File (voice branch), Process messages (text branch) |                                                                                                                              |
| Get Voice File            | Telegram (Get File)              | Downloads voice message audio file             | If                         | Speech to Text                | Downloads the voice message from Telegram for transcription.                                                                   |
| Speech to Text            | LangChain OpenAI Audio           | Transcribes voice audio into text              | Get Voice File              | Transcription to ChatInput    | Transcribes voice input into text using OpenAI Whisper.                                                                        |
| Transcription to ChatInput| Set                             | Maps transcription text to `chatInput` field | Speech to Text              | Process messages              | Maps transcribed text into unified chatInput field.                                                                            |
| Process messages          | Code                            | Normalizes text input, detects source, sets sessionId | Transcription to ChatInput, When chat message received | Home Agent                   | Normalizes inputText, detects source (chat/telegram), builds sessionId.                                                       |
| Bot Is typing             | Telegram (Send Chat Action)      | Sends "typing..." feedback to Telegram         | Telegram Trigger            |                             | Sends “typing…” action back to Telegram for better UX feedback.                                                               |
| Home Agent                | LangChain Agent                 | Core AI agent, orchestrates LLM + Home Assistant | Process messages            | Reply Router                 | Core AI agent that orchestrates LLM and Home Assistant actions.                                                                |
| Google Gemini Chat Model  | LangChain Google Gemini Chat Model | Provides AI language model completions        | Home Agent (ai_languageModel) | Home Agent                   |                                                                                                                              |
| Simple Memory             | LangChain Memory Buffer Window  | Maintains short-term memory context - chat    |                            | Home Agent                   |                                                                                                                              |
| Simple Memory1            | LangChain Memory Buffer Window  | Maintains short-term memory context - Telegram |                            | When chat message received    |                                                                                                                              |
| Home assistant Connector  | LangChain MCP Client Tool       | Executes smart home commands on Home Assistant | Home Agent (ai_tool)        | Home Agent                   | Executes smart home actions: turn lights on/off, adjust brightness, broadcast messages via MCP.                               |
| Reply Router              | If                              | Routes AI response to Telegram or n8n chat UI | Home Agent                  | Telegram Message Beautifier, Respond to Webhook | Routes response back to Telegram or n8n chat UI depending on origin.                                                           |
| Telegram Message Beautifier| Code                           | Formats AI response text into Telegram HTML    | Reply Router                | Telegram Send                | Formats output: bold, bullets, inline code, links – splits into chunks.                                                        |
| Telegram Send             | Telegram Send                   | Sends formatted reply back to Telegram chat    | Telegram Message Beautifier |                             | Sends formatted reply back to Telegram in HTML parse mode.                                                                     |
| Respond to Webhook        | Respond to Webhook              | Streams AI response to n8n chat frontend webhook | Reply Router                |                             | Streams responses back to the n8n chat frontend webhook.                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only  
   - Set credentials to your Telegram Bot API ("HomeWhisper")  
   - Position: Left-top area

2. **Create When chat message received node**
   - Type: LangChain Chat Trigger  
   - Set webhook to public with "loadPreviousSession" enabled  
   - Position: Left-middle area

3. **Create Voice or Text node**
   - Type: Set  
   - Add field `text` with expression: `{{$json?.message?.text || ""}}`  
   - Connect Telegram Trigger output to this node

4. **Create If node**
   - Type: If  
   - Condition: Check if `message.text` is empty string (expression: `={{ $json.message.text }}` equals "")  
   - Connect Voice or Text main output to this node

5. **Create Get Voice File node**
   - Type: Telegram  
   - Operation: Get File  
   - File ID: `{{$json.message.voice.file_id}}`  
   - Set credentials to Telegram API  
   - Connect If node "true" (empty text) branch to this node

6. **Create Speech to Text node**
   - Type: LangChain OpenAI  
   - Resource: Audio  
   - Operation: Transcribe  
   - Set OpenAI credentials  
   - Connect Get Voice File output to this node

7. **Create Transcription to ChatInput node**
   - Type: Set  
   - Add field `chatInput` set to `{{$json.text}}` (transcribed text)  
   - Connect Speech to Text output to this node

8. **Connect If node "false" (non-empty text) branch directly to Process messages (step 9)**

9. **Create Process messages node**
   - Type: Code  
   - Paste the provided JavaScript code that safely extracts input text, detects source, sessionId, and voice flag  
   - Connect Transcription to ChatInput output and When chat message received output to this node (two separate inputs feeding the same node)

10. **Create Bot Is typing node**
    - Type: Telegram  
    - Operation: Send Chat Action ("typing")  
    - Set chatId as `={{$('Telegram Trigger').item.json.message.chat.id}}`  
    - Connect Telegram Trigger output to this node (parallel to Voice or Text)

11. **Create Simple Memory node**
    - Type: LangChain Memory Buffer Window  
    - No special parameters  
    - Position near AI processing nodes

12. **Create Simple Memory1 node**  
    - Type: LangChain Memory Buffer Window  
    - No special parameters  
    - Connect output to When chat message received (ai_memory input)

13. **Create Google Gemini Chat Model node**
    - Type: LangChain Google Gemini Chat Model  
    - Configure with Google Palm API credentials  
    - Connect output to Home Agent (ai_languageModel input)

14. **Create Home assistant Connector node**
    - Type: LangChain MCP Client Tool  
    - Endpoint: `http://192.168.1.69:8123/mcp_server/sse`  
    - Include tools: GetLiveContext, HassTurnOn, HassTurnOff, HassLightSet, HassBroadcast  
    - Use HTTP Bearer Auth credentials for Home Assistant  
    - Connect output to Home Agent (ai_tool input)

15. **Create Home Agent node**
    - Type: LangChain Agent  
    - Prompt type: define  
    - Input text: `={{$json.inputText}}`  
    - Connect Process messages output (main), Simple Memory (ai_memory), Google Gemini Chat Model (ai_languageModel), Home assistant Connector (ai_tool) to this node

16. **Create Reply Router node**
    - Type: If  
    - Condition: `$json.source == "telegram"`  
    - Connect Home Agent output to this node

17. **Create Telegram Message Beautifier node**
    - Type: Code  
    - Paste provided JS code for formatting (bold, bullets, inline code, links, chunking)  
    - Connect Reply Router "true" branch to this node

18. **Create Telegram Send node**
    - Type: Telegram Send  
    - Text: `={{ $json.text }}`  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Parse mode: HTML  
    - Connect Telegram Message Beautifier output to this node

19. **Create Respond to Webhook node**
    - Type: Respond to Webhook  
    - Respond with: Text  
    - Response body: `={{ $json.output }}`  
    - Connect Reply Router "false" branch (non-telegram) to this node

20. **Arrange nodes for clarity and test workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                             | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| **Workflow Name:** Home Assistant Whisper. Bridges natural language commands from Telegram and n8n chat to Home Assistant control. Supports voice transcription, AI intent processing, smart home execution, and formatted response routing. Modular and extendable.                                    | Workflow description sticky note                                                                         |
| Telegram message formatting uses HTML parse mode. Markdown-like syntax is converted to HTML tags (bold, bullets, inline code, links). Messages are chunked to respect Telegram’s 4096 character limit per message.                                                                                      | Sticky Note on Telegram Message Beautifier node                                                          |
| OpenAI Whisper is used for voice transcription; requires valid OpenAI API key. Google Gemini (PaLM) API powers the LLM for intent recognition and response generation; requires Google Palm API credentials.                                                                                             | Credential requirements                                                                                   |
| Home Assistant MCP client connects via SSE endpoint with bearer token authentication. Ensure MCP server is accessible at the configured IP and port.                                                                                                                                                   | Home assistant Connector sticky note                                                                     |
| The workflow supports multi-channel inputs and routes responses accordingly: Telegram replies go back via Telegram API, n8n chat responses are streamed back through webhook.                                                                                                                           | Reply Router sticky note                                                                                   |
| "Bot Is typing" action improves Telegram UX during processing.                                                                                                                                                                                                                                         | Sticky Note on Bot Is typing node                                                                          |
| For detailed explanation of the AI agent prompt and memory usage, refer to LangChain documentation and n8n LangChain node docs.                                                                                                                                                                       | External documentation                                                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.