Build a Telegram Q&A Bot with Linkup Web Search, GPT-4.1 & Mistral Voice

https://n8nworkflows.xyz/workflows/build-a-telegram-q-a-bot-with-linkup-web-search--gpt-4-1---mistral-voice-9871


# Build a Telegram Q&A Bot with Linkup Web Search, GPT-4.1 & Mistral Voice

---

### 1. Workflow Overview

This workflow builds a Telegram Q&A Bot that leverages web search powered by Linkup API, GPT-4.1 for AI reasoning, and Mistral voice transcription to handle both text and voice inputs. Its primary use case is enabling Telegram users to ask questions either by text or voice message and receive AI-generated answers sourced from live web searches. The workflow is designed to be optionally private by filtering users based on Telegram usernames.

The workflow is logically divided into:

- **1.1 Input Reception & Filtering**: Handles incoming Telegram messages, distinguishes between text and voice, and optionally filters messages to a specific user.
- **1.2 Audio Processing**: Downloads voice messages and transcribes them to text using Mistral API.
- **1.3 Text Processing**: Prepares user text messages or transcribed voice messages into a unified format.
- **1.4 AI Agent Processing**: Uses a LangChain AI agent with GPT-4.1 model and Linkup Web Search tool to generate answers based on user queries and web search results.
- **1.5 Output Delivery**: Sends the AI-generated answer back to the respective Telegram user.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Filtering

**Overview:**  
This initial block receives Telegram updates, filters messages optionally by a specific Telegram username to make the bot private or public, then routes the message based on its content type (voice or text).

**Nodes Involved:**  
- Telegram Trigger  
- Myself? (If node for filtering username)  
- Message Router

**Node Details:**  

- **Telegram Trigger**  
  - Type: telegramTrigger  
  - Role: Entry point webhook to receive incoming Telegram messages of type "message" only.  
  - Config: Uses Telegram API credential "Duv Brain Telegram".  
  - Inputs: Telegram webhook updates.  
  - Outputs: Raw Telegram message JSON.  
  - Edge cases: No messages received if webhook not activated or Telegram API credential invalid.  
  - Version: 1.1  

- **Myself?**  
  - Type: if  
  - Role: Filters messages by Telegram username to restrict bot usage to a specific user for privacy.  
  - Config: Checks if `$json.message.from.username` equals the specified username (placeholder to be replaced).  
  - Inputs: Telegram Trigger output.  
  - Outputs: Passes messages from the specified username forward; others are blocked.  
  - Edge cases: If username not set or malformed, filtering fails; deleting this node makes bot public.  
  - Version: 2.2  

- **Message Router**  
  - Type: switch  
  - Role: Routes messages based on content type: routes to audio processing if voice message exists; otherwise to text processing.  
  - Config: Checks existence of `message.voice` or `message.text` fields in the Telegram message JSON.  
  - Inputs: Output of Myself? node.  
  - Outputs: Two outputs, "audio" for voice messages and "text" for text messages.  
  - Edge cases: Messages without text or voice go to fallback "extra" (not connected).  
  - Version: 3.2  

---

#### 2.2 Audio Processing

**Overview:**  
This block handles voice messages by downloading the audio file from Telegram, sending it to Mistral API for transcription, and preparing the transcribed text for further AI processing.

**Nodes Involved:**  
- Get Audio File  
- Mistral transcribe  
- Prepare message from audio

**Node Details:**  

- **Get Audio File**  
  - Type: telegram (file download)  
  - Role: Downloads the voice message audio file from Telegram using the file_id.  
  - Config: Uses Telegram API credential "Duv Brain Telegram". Extracts `voice.file_id` from message JSON.  
  - Inputs: Message Router output "audio".  
  - Outputs: Binary audio file data.  
  - Edge cases: File not found or download failure due to expired file_id or API issues.  
  - Version: 1.2  

- **Mistral transcribe**  
  - Type: httpRequest  
  - Role: Sends binary audio file to Mistral API for speech-to-text transcription.  
  - Config: POST to `https://api.mistral.ai/v1/audio/transcriptions` with model "voxtral-mini-2507". Uses Mistral Cloud API credential.  
  - Inputs: Binary audio from Get Audio File.  
  - Outputs: JSON transcription result.  
  - Edge cases: API auth failure, timeouts, file format errors.  
  - Version: 4.2  

- **Prepare message from audio**  
  - Type: set  
  - Role: Extracts transcribed text from Mistral response and assigns it to `preset_user_message` for uniform processing downstream.  
  - Config: Sets `preset_user_message` to `choices[0].message.content` (transcription text).  
  - Inputs: Mistral transcribe output.  
  - Outputs: JSON with `preset_user_message`.  
  - Version: 3.4  

---

#### 2.3 Text Processing

**Overview:**  
This block handles direct text messages from Telegram, preparing them into a consistent variable for further AI processing.

**Nodes Involved:**  
- Prepare message from text

**Node Details:**  

- **Prepare message from text**  
  - Type: set  
  - Role: Reads raw text from the Telegram message and assigns it to `preset_user_message`.  
  - Config: Sets `preset_user_message` to `message.text`.  
  - Inputs: Message Router output "text".  
  - Outputs: JSON with `preset_user_message`.  
  - Version: 3.4  

---

#### 2.4 Consolidation & AI Agent Processing

**Overview:**  
This block consolidates the user message from either audio transcription or text input, manages session memory for conversation context, and invokes the AI agent to generate an answer using GPT-4.1 and Linkup web search.

**Nodes Involved:**  
- Consolidate user message  
- Simple Memory  
- AI Agent  
- OpenAI Chat Model  
- Web search

**Node Details:**  

- **Consolidate user message**  
  - Type: set  
  - Role: Assigns the prepared user message (`preset_user_message`) to `final_user_message` for AI agent input.  
  - Config: Sets `final_user_message` = `preset_user_message`.  
  - Inputs: Outputs from both Prepare message from audio and Prepare message from text.  
  - Outputs: JSON with `final_user_message`.  
  - Version: 3.4  

- **Simple Memory**  
  - Type: memoryBufferWindow (LangChain)  
  - Role: Maintains a conversation memory buffer for context in AI interactions, keyed by Telegram username.  
  - Config: Session key set to the Telegram username, context window length 7 messages.  
  - Inputs: AI Agent memory input.  
  - Outputs: Provides context messages to AI Agent.  
  - Version: 1.3  

- **AI Agent**  
  - Type: LangChain agent  
  - Role: Core AI engine that processes the user query, calls the Linkup web search tool if needed, and formats the response.  
  - Config:  
    - System message instructs the agent to call "Web search" tool for queries requiring web insights, provide sources, and adapt style for Telegram.  
    - Prompt type: define (custom prompt).  
    - Inputs: `final_user_message` text, AI memory from Simple Memory, language model from OpenAI Chat Model, and AI tool Web search.  
  - Outputs: AI-generated answer text.  
  - Version: 1.8  
  - Edge cases: API quota limits, tool call failures, improper prompt formatting.  

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides GPT-4.1-mini language model for the AI Agent to generate answers.  
  - Config: Model set to "gpt-4.1-mini". Uses OpenAI API credential "Duv's OpenAI".  
  - Inputs: AI languageModel input for AI Agent.  
  - Outputs: Processed language model completions to AI Agent.  
  - Version: 1.2  

- **Web search**  
  - Type: httpRequestTool  
  - Role: Tool invoked by AI Agent to perform web searches via Linkup API to enrich answers.  
  - Config: POST request to `https://api.linkup.so/v1/search` with parameters: query `q`, depth "standard", outputType "sourcedAnswer", and other flags false.  
  - Header includes Authorization with Bearer token placeholder `<your Linkup API key>`.  
  - Inputs: AI Agent tool input.  
  - Outputs: Search results to AI Agent.  
  - Version: 4.2  
  - Edge cases: Invalid API keys, network failures, response format changes.  

---

#### 2.5 Output Delivery

**Overview:**  
Sends the AI-generated answer text back to the Telegram user who initiated the conversation.

**Nodes Involved:**  
- Telegram answer

**Node Details:**  

- **Telegram answer**  
  - Type: telegram  
  - Role: Sends a Telegram message containing the AI answer to the chat ID of the original user.  
  - Config: Uses Telegram API credential "Duv Brain Telegram". Message text set to AI Agent output `output`. ChatId extracted from Telegram Trigger message sender ID.  
  - Inputs: AI Agent output.  
  - Outputs: None (message sent).  
  - Version: 1.2  
  - Edge cases: Telegram API limits, user blocked bot, network issues.  

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                          | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                             |
|------------------------|------------------------------------|----------------------------------------|---------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger       | telegramTrigger                    | Entry point for Telegram messages       | -                         | Myself?                 |                                                                                                                         |
| Myself?                | if                                | Filters messages by Telegram username   | Telegram Trigger          | Message Router          | Make this bot private: filter by username to restrict access or delete to make public                                   |
| Message Router         | switch                           | Routes messages by type (voice or text) | Myself?                   | Get Audio File, Prepare message from text |                                                                                                                         |
| Get Audio File         | telegram                         | Downloads voice message audio file      | Message Router (audio)    | Mistral transcribe      |                                                                                                                         |
| Mistral transcribe     | httpRequest                      | Transcribes audio to text via Mistral   | Get Audio File            | Prepare message from audio | AI transcription: replace with another model like OpenAI's                                                              |
| Prepare message from audio | set                           | Prepares transcribed text for AI input  | Mistral transcribe        | Consolidate user message |                                                                                                                         |
| Prepare message from text | set                            | Prepares raw text message for AI input  | Message Router (text)     | Consolidate user message |                                                                                                                         |
| Consolidate user message | set                            | Consolidates user message for AI agent  | Prepare message from audio, Prepare message from text | AI Agent               |                                                                                                                         |
| Simple Memory          | memoryBufferWindow (LangChain)   | Maintains conversation context memory   | AI Agent                  | AI Agent                |                                                                                                                         |
| AI Agent               | LangChain agent                  | Core AI processing and tool orchestration | Consolidate user message, Simple Memory, OpenAI Chat Model, Web search | Telegram answer          | The agent: distributes questions to Linkup web search tool; customize system message and response style                |
| OpenAI Chat Model      | LangChain LM Chat OpenAI          | Provides GPT-4.1 language model         | AI Agent                  | AI Agent                | AI model: Connect OpenAI API key or another LLM provider                                                                |
| Web search             | httpRequestTool                  | Performs Linkup API web searches         | AI Agent                  | AI Agent                | Linkup web search: replace API key placeholder with your own or store as credential                                      |
| Telegram answer        | telegram                         | Sends AI-generated answer back to user  | AI Agent                  | -                       |                                                                                                                         |
| Sticky Note5           | stickyNote                      | Visual note: "# Process Audio"          | -                         | -                       |                                                                                                                         |
| Sticky Note8           | stickyNote                      | Visual note: "# Process text"           | -                         | -                       |                                                                                                                         |
| Sticky Note6           | stickyNote                      | Visual note: "AI transcription"         | -                         | -                       |                                                                                                                         |
| Sticky Note7           | stickyNote                      | Visual note: "Make this bot private"    | -                         | -                       |                                                                                                                         |
| Sticky Note1           | stickyNote                      | Visual note: "AI model"                  | -                         | -                       |                                                                                                                         |
| Sticky Note            | stickyNote                      | Visual note: "Linkup web search"         | -                         | -                       |                                                                                                                         |
| Sticky Note2           | stickyNote                      | Visual note: "The agent"                 | -                         | -                       |                                                                                                                         |
| Sticky Note3           | stickyNote                      | Visual note: Workflow summary and setup | -                         | -                       | Detailed setup instructions and summary including links to https://linkup.so                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: telegramTrigger  
   - Trigger on "message" updates only.  
   - Configure Telegram API credentials (OAuth or token).  
   - Position it as the workflow entry point.

2. **Add an If node named "Myself?"**  
   - Condition: Check if expression `$json.message.from.username` equals your Telegram username (replace `<Replace with your Telegram username>`).  
   - This filters messages to only allow your private use.  
   - Connect Telegram Trigger's output to this node.

3. **Add a Switch node named "Message Router"**  
   - Add 2 rules:  
     - Rule "audio": check if `message.voice` exists in Telegram message JSON.  
     - Rule "text": check if `message.text` exists.  
   - Connect "Myself?" true output to "Message Router".

4. **Audio Processing Path**:  
   - Add "Get Audio File" node:  
     - Type: telegram (File download)  
     - Parameter: fileId = `{{$json.message.voice.file_id}}`  
     - Connect Message Router "audio" output here.  
     - Use Telegram API credentials.  
   - Add "Mistral transcribe" node:  
     - Type: HTTP Request (multipart/form-data)  
     - URL: `https://api.mistral.ai/v1/audio/transcriptions`  
     - Method: POST  
     - Body parameters:  
       - model = "voxtral-mini-2507"  
       - file = binary from "Get Audio File" node  
     - Use Mistral Cloud API credentials.  
     - Connect "Get Audio File" output here.  
   - Add "Prepare message from audio" node:  
     - Type: set  
     - Set variable `preset_user_message` = `{{$json.choices[0].message.content}}`  
     - Connect "Mistral transcribe" output here.

5. **Text Processing Path**:  
   - Add "Prepare message from text" node:  
     - Type: set  
     - Set variable `preset_user_message` = `{{$json.message.text}}`  
     - Connect Message Router "text" output here.

6. **Consolidate user message**:  
   - Add a "set" node named "Consolidate user message"  
   - Set variable `final_user_message` = `{{$json.preset_user_message}}`  
   - Connect outputs of both "Prepare message from audio" and "Prepare message from text" nodes to this node.

7. **Add LangChain Simple Memory node**:  
   - Type: memoryBufferWindow  
   - Set `sessionKey` to `{{$('Telegram Trigger').item.json.message.from.username}}`  
   - Context window length: 7 messages  
   - Connect output to AI Agent memory input.

8. **Add LangChain OpenAI Chat Model node**:  
   - Type: lmChatOpenAi  
   - Model: "gpt-4.1-mini"  
   - Use OpenAI API credentials.  
   - Connect output to AI Agent languageModel input.

9. **Add HTTP Request Tool node named "Web search"**:  
   - POST `https://api.linkup.so/v1/search`  
   - Body parameters:  
     - q = (dynamic query from AI Agent)  
     - depth = "standard"  
     - outputType = "sourcedAnswer"  
     - includeImages = false  
     - includeInlineCitations = false  
   - Headers: Authorization: Bearer `<your Linkup API key>` (replace with your actual key or use credential system)  
   - Connect as AI tool input for AI Agent.

10. **Add LangChain Agent node named "AI Agent"**  
    - Text input: `{{$json.final_user_message}}`  
    - System message: instruct to use "Web search" tool for web-based queries, provide sources, adapt style for Telegram.  
    - Connect inputs:  
      - text input from "Consolidate user message"  
      - ai_memory from Simple Memory  
      - ai_languageModel from OpenAI Chat Model  
      - ai_tool from Web search  
    - Connect output to Telegram answer node.

11. **Add Telegram node "Telegram answer"**  
    - Text: `{{$json.output}}` (AI Agent answer)  
    - ChatId: `{{$('Telegram Trigger').item.json.message.from.id}}`  
    - Use Telegram API credentials.  
    - Connect AI Agent output here.

12. **Optional:** Add Sticky Notes for documentation and visual guidance.

13. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Create a Linkup account at https://linkup.so and obtain your API key to replace the placeholder in the "Web search" node header Authorization field.                                                                                                       | Linkup API key required for web search tool                                                      |
| Ensure Telegram API credentials are correctly set up with your bot token.                                                                                                                                                                                    | Telegram Trigger and Telegram answer nodes                                                        |
| Mistral Cloud API credentials are required for audio transcription.                                                                                                                                                                                          | Mistral transcribe node                                                                           |
| The "Myself?" node is optional and used to make the bot private by filtering messages to a specific Telegram username. Remove or modify this node to make the bot public.                                                                                     | Privacy control                                                                                   |
| The AI Agent uses a LangChain agent configured to call the Web search tool. You can customize the system prompt to adapt the assistant’s style or add instructions.                                                                                          | AI Agent node configuration                                                                      |
| This workflow supports both text and voice queries in Telegram seamlessly, transcribing voice messages before processing.                                                                                                                                   | Overall workflow feature                                                                          |
| The AI model used is GPT-4.1-mini via OpenAI. You can replace it by connecting other LLM providers compatible with LangChain nodes.                                                                                                                        | OpenAI Chat Model node configuration                                                             |
| The workflow memory buffer uses Telegram usernames as session keys to maintain conversation context per user.                                                                                                                                                | Simple Memory node configuration                                                                 |
| For detailed setup, see the sticky note with summary and instructions in the workflow (node "Sticky Note3").                                                                                                                                                 | Workflow summary and setup guide                                                                 |
| Template created by Guillaume Duvernay.                                                                                                                                                                                                                      | Author credit                                                                                     |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---