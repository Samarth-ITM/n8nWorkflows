Create a Voice & Text Telegram Assistant with Lookio RAG and GPT-4.1

https://n8nworkflows.xyz/workflows/create-a-voice---text-telegram-assistant-with-lookio-rag-and-gpt-4-1-9870


# Create a Voice & Text Telegram Assistant with Lookio RAG and GPT-4.1

### 1. Workflow Overview

This workflow implements a Telegram assistant bot capable of processing both voice and text messages, answering user queries by leveraging a Retrieval-Augmented Generation (RAG) architecture integrating Lookio’s knowledge base and GPT-4.1 language model. It is designed to accept audio messages, transcribe them via the Mistral API, and use a LangChain AI agent to query Lookio’s knowledge base when domain-specific knowledge is required. The workflow includes filtering for private use based on Telegram usernames but can be made public by removing this filter.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering:** Captures incoming Telegram messages, routes based on message type (voice or text), and optionally filters for authorized users.
- **1.2 Audio Processing:** Downloads audio files from Telegram, transcribes them using the Mistral API, and prepares the transcribed text for AI processing.
- **1.3 Text Processing:** Prepares text messages directly for AI processing.
- **1.4 User Message Consolidation:** Unifies audio-derived and text-derived messages into a single user message for the AI agent.
- **1.5 AI Agent Processing:** Uses LangChain’s AI agent with GPT-4.1 to generate responses, querying Lookio’s knowledge base tool when domain knowledge is needed.
- **1.6 Response Delivery:** Sends the AI-generated response back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Filtering

- **Overview:**  
  Captures incoming Telegram messages and filters them based on the Telegram username to restrict access to the bot (optional). Then routes messages according to whether they are audio (voice) or text.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Myself? (IF node for filtering)  
  - Message Router (Switch node)

- **Node Details:**  
  1. **Telegram Trigger**  
     - Type: **telegramTrigger**  
     - Role: Entry point capturing Telegram "message" updates (text and voice).  
     - Config: Listens for message updates only, uses Telegram API credentials.  
     - Inputs: Telegram webhook incoming messages.  
     - Outputs: Passes messages to filtering node.  
     - Edge cases: Telegram API downtime; missing message fields; webhook misconfiguration.  
  2. **Myself? (IF node)**  
     - Type: **if**  
     - Role: Filters messages so only those from a specific Telegram username proceed.  
     - Config: Compares `$json.message.from.username` to a placeholder `<Replace with your Telegram username>`.  
     - Inputs: From Telegram Trigger.  
     - Outputs: True branch to Message Router; False branch discarded.  
     - Edge cases: Username not set or mismatched; making bot public requires deleting this node.  
  3. **Message Router (Switch node)**  
     - Type: **switch**  
     - Role: Routes the message to audio or text processing based on presence of `message.voice` or `message.text`.  
     - Config: Two outputs named "audio" and "text", with conditions checking for existence of voice or text field respectively.  
     - Inputs: From Myself? node.  
     - Outputs:  
       - "audio" → Get Audio File node  
       - "text" → Prepare message from text node  
       - "extra" (fallback) unused here.  
     - Edge cases: Messages that have neither voice nor text (e.g., stickers) are routed to fallback "extra" (not connected).

#### 1.2 Audio Processing

- **Overview:**  
  Downloads the audio file from Telegram, sends it to Mistral API for transcription, and prepares the transcribed text for downstream AI processing.

- **Nodes Involved:**  
  - Get Audio File  
  - Mistral transcribe  
  - Prepare message from audio  

- **Node Details:**  
  1. **Get Audio File**  
     - Type: **telegram** (resource: file)  
     - Role: Downloads the voice message file from Telegram using file_id.  
     - Config: Uses Telegram API credentials; fetches file using `message.voice.file_id`.  
     - Inputs: From Message Router (audio route).  
     - Outputs: Sends file binary data to Mistral transcribe.  
     - Edge cases: Telegram API file retrieval failure; file_id missing or invalid.  
  2. **Mistral transcribe**  
     - Type: **httpRequest**  
     - Role: Sends the audio file to Mistral API for speech-to-text transcription.  
     - Config: POST to `https://api.mistral.ai/v1/audio/transcriptions` with multipart form data, model "voxtral-mini-2507". Uses "Mistral Cloud" credentials for authentication.  
     - Inputs: Receives binary audio file data from Get Audio File.  
     - Outputs: Transcription JSON response with transcribed text.  
     - Edge cases: API key issues, network timeout, incorrect file format, transcription errors.  
  3. **Prepare message from audio**  
     - Type: **set**  
     - Role: Extracts transcription text from Mistral response and sets it as `preset_user_message`.  
     - Config: Assigns `preset_user_message` to the first choice message content from transcription response.  
     - Inputs: From Mistral transcribe.  
     - Outputs: Passes to Consolidate user message node.  
     - Edge cases: Transcription result missing, empty or malformed response.

#### 1.3 Text Processing

- **Overview:**  
  Prepares direct text messages from Telegram for AI processing by setting the user message.

- **Nodes Involved:**  
  - Prepare message from text  

- **Node Details:**  
  1. **Prepare message from text**  
     - Type: **set**  
     - Role: Extracts the text message from Telegram JSON and assigns it as `preset_user_message`.  
     - Config: Assigns `preset_user_message` from `message.text`.  
     - Inputs: From Message Router (text route).  
     - Outputs: Passes to Consolidate user message node.  
     - Edge cases: Missing or empty text in the message.

#### 1.4 User Message Consolidation

- **Overview:**  
  Consolidates the user message variable into `final_user_message` for uniform downstream AI processing.

- **Nodes Involved:**  
  - Consolidate user message  

- **Node Details:**  
  1. **Consolidate user message**  
     - Type: **set**  
     - Role: Sets `final_user_message` to the value of `preset_user_message` from either audio or text preparation nodes.  
     - Config: Simple value assignment.  
     - Inputs: From Prepare message from text or Prepare message from audio.  
     - Outputs: Passes consolidated message to AI Agent node.  
     - Edge cases: Empty or undefined `preset_user_message`.

#### 1.5 AI Agent Processing

- **Overview:**  
  Runs a LangChain AI agent that processes the user message using a GPT-4.1 language model, queries the Lookio knowledge base as a tool when needed, and maintains a memory buffer of recent interactions.

- **Nodes Involved:**  
  - AI Agent  
  - Simple Memory  
  - OpenAI Chat Model  
  - Query knowledge base  

- **Node Details:**  
  1. **AI Agent**  
     - Type: **langchain.agent**  
     - Role: Core processing node that formulates responses using a defined system prompt, calls knowledge base when appropriate, and formats output for Telegram.  
     - Config:  
       - Input text from `final_user_message`.  
       - System message instructs the agent to answer based on the knowledge base, transparently communicate if knowledge is insufficient, and format output for Telegram.  
       - Uses "define" prompt type.  
       - Connected to language model, memory, and knowledge base tool nodes.  
     - Inputs: From Consolidate user message (main), Simple Memory (ai_memory), OpenAI Chat Model (ai_languageModel), Query knowledge base (ai_tool).  
     - Outputs: Response text to Telegram answer node.  
     - Edge cases: AI service errors, malformed input, tool call failures, memory overflow.  
  2. **Simple Memory**  
     - Type: **langchain.memoryBufferWindow**  
     - Role: Maintains conversational context window of last 7 exchanges per user session, identified by Telegram username.  
     - Config: Session key set to username from Telegram message; context window length 7.  
     - Inputs: Feeding memory to AI Agent.  
     - Outputs: Context to AI Agent.  
     - Edge cases: Missing username, session key conflicts.  
  3. **OpenAI Chat Model**  
     - Type: **langchain.lmChatOpenAi**  
     - Role: Provides GPT-4.1-mini as the language model for the AI agent.  
     - Config: Model is "gpt-4.1-mini"; uses OpenAI API credentials.  
     - Inputs: AI Agent for language model calls.  
     - Outputs: Responses to AI Agent.  
     - Edge cases: API key invalid, rate limits, latency.  
  4. **Query knowledge base**  
     - Type: **httpRequestTool**  
     - Role: Tool invoked by AI agent to query Lookio knowledge base for domain-specific answers.  
     - Config:  
       - POST to `https://api.lookio.app/webhook/query`.  
       - Body includes the query (injected by AI agent), assistant ID, and query mode "flash".  
       - Header includes Lookio API key.  
       - Tool description notes to replace API key and assistant ID placeholders.  
     - Inputs: AI Agent tool calls.  
     - Outputs: Knowledge base answers back to AI Agent.  
     - Edge cases: Invalid API key, assistant ID mismatch, network errors, malformed queries.

#### 1.6 Response Delivery

- **Overview:**  
  Sends the AI-generated answer back to the Telegram user.

- **Nodes Involved:**  
  - Telegram answer  

- **Node Details:**  
  1. **Telegram answer**  
     - Type: **telegram** (send message)  
     - Role: Sends the final AI response text to the Telegram user’s chat ID.  
     - Config:  
       - Text taken from AI Agent output `output`.  
       - Chat ID set to the original sender’s Telegram ID.  
       - Attribution disabled.  
       - Uses Telegram API credentials.  
     - Inputs: From AI Agent.  
     - Outputs: None.  
     - Edge cases: Telegram API errors, invalid chat ID, message length limits.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                          | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                          |
|-------------------------|---------------------------------|----------------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger        | telegramTrigger                 | Entry point capturing Telegram messages| -                     | Myself?                  |                                                                                                    |
| Myself?                 | if                             | Filters messages by Telegram username  | Telegram Trigger      | Message Router           | Make this bot private - filter messages by username or delete to make public                      |
| Message Router          | switch                         | Routes messages by type (audio/text)   | Myself?               | Get Audio File, Prepare message from text |                                                                                                    |
| Get Audio File          | telegram (file resource)        | Downloads voice message audio file      | Message Router (audio) | Mistral transcribe       | # Process Audio                                                                                     |
| Mistral transcribe      | httpRequest                    | Transcribes audio to text via Mistral API | Get Audio File        | Prepare message from audio | AI transcription - can replace with other models like OpenAI's                                     |
| Prepare message from audio | set                           | Sets transcribed text as user message   | Mistral transcribe    | Consolidate user message |                                                                                                    |
| Prepare message from text | set                           | Sets text message as user message       | Message Router (text) | Consolidate user message |                                                                                                    |
| Consolidate user message | set                           | Unifies user message variable            | Prepare message from audio/text | AI Agent               |                                                                                                    |
| AI Agent                | langchain.agent                | Processes user message, queries knowledge base, generates answer | Consolidate user message, Simple Memory, OpenAI Chat Model, Query knowledge base | Telegram answer       | The agent uses Lookio tool to answer using knowledge base; adapts style for Telegram               |
| Simple Memory           | langchain.memoryBufferWindow   | Maintains conversational context window | -                     | AI Agent (memory input)  |                                                                                                    |
| OpenAI Chat Model       | langchain.lmChatOpenAi         | GPT-4.1 language model for AI Agent     | -                     | AI Agent (language model input) | AI model - core LLM of the agent; connect OpenAI or other LLM provider                            |
| Query knowledge base    | httpRequestTool                | Queries Lookio knowledge base            | AI Agent (tool input) | AI Agent (tool output)   | Lookio tool - add API key and assistant ID to query knowledge base                                |
| Telegram answer         | telegram (send message)         | Sends AI response back to Telegram user | AI Agent              | -                        |                                                                                                    |
| Sticky Note5            | stickyNote                    | Visual note: "# Process Audio"           | -                     | -                        | # Process Audio                                                                                     |
| Sticky Note1            | stickyNote                    | Visual note: "AI model"                   | -                     | -                        | AI model - core AI model of your agent; connect OpenAI API or other LLM                           |
| Sticky Note2            | stickyNote                    | Visual note: "The agent"                  | -                     | -                        | The agent - distributes relevant questions to Lookio via tool                                    |
| Sticky Note             | stickyNote                    | Visual note: "Lookio tool"                 | -                     | -                        | Lookio tool - add API key and assistant ID                                                       |
| Sticky Note6            | stickyNote                    | Visual note: "Make this bot private"      | -                     | -                        | Telegram bots are public by default, filtering by username makes it private                      |
| Sticky Note8            | stickyNote                    | Visual note: "# Process text"              | -                     | -                        | # Process text                                                                                     |
| Sticky Note3            | stickyNote                    | Visual note: RAG Telegram bot summary     | -                     | -                        | Summary of workflow and setup instructions                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: telegramTrigger  
   - Parameters: Listen to "message" updates only.  
   - Credentials: Create and assign your Telegram API credentials.  
   - Position at start of workflow.

2. **Add IF Node "Myself?" for User Filtering**  
   - Type: if  
   - Condition: `$json.message.from.username` equals your Telegram username.  
   - Purpose: Restrict access to your username for private testing (optional). Remove this node to make bot public.  
   - Connect Telegram Trigger main output to this node's true branch.

3. **Add Switch Node "Message Router"**  
   - Type: switch  
   - Rules:  
     - Output "audio": Condition exists `$('Telegram Trigger').item.json.message.voice`  
     - Output "text": Condition exists `$('Telegram Trigger').item.json.message.text`  
   - Connect "Myself?" true output to Message Router.

4. **Audio Processing Branch**  
   - Add **telegram** node "Get Audio File"  
     - Resource: file  
     - File ID: `{{$json.message.voice.file_id}}`  
     - Credentials: Telegram API.  
     - Connect Message Router "audio" output here.

   - Add **httpRequest** node "Mistral transcribe"  
     - Method: POST to `https://api.mistral.ai/v1/audio/transcriptions`  
     - Authentication: Use "Mistral Cloud" credentials.  
     - Body: multipart-form-data with parameters: model="voxtral-mini-2507", file from previous node binary.  
     - Connect Get Audio File output here.

   - Add **set** node "Prepare message from audio"  
     - Assign `preset_user_message` = `{{$json.choices[0].message.content}}` (transcription text).  
     - Connect Mistral transcribe output here.

5. **Text Processing Branch**  
   - Add **set** node "Prepare message from text"  
     - Assign `preset_user_message` = `{{$json.message.text}}` (direct text message).  
     - Connect Message Router "text" output here.

6. **Consolidate User Message**  
   - Add **set** node "Consolidate user message"  
     - Assign `final_user_message` = `{{$json.preset_user_message}}`  
     - Connect outputs of both "Prepare message from audio" and "Prepare message from text" nodes into this node.

7. **Add LangChain Components for AI Agent**  
   - Add **langchain.memoryBufferWindow** node "Simple Memory"  
     - Session key: `{{$json.message.from.username}}`  
     - Context window length: 7  
     - No explicit input connections (memory node configured for agent).

   - Add **langchain.lmChatOpenAi** node "OpenAI Chat Model"  
     - Model: gpt-4.1-mini (or your preferred GPT-4.1 variant)  
     - Attach OpenAI API credentials.

   - Add **httpRequestTool** node "Query knowledge base"  
     - Method: POST to `https://api.lookio.app/webhook/query`  
     - Body parameters:  
       - query: to be filled dynamically by the agent.  
       - assistant_id: set your Lookio assistant ID.  
       - query_mode: "flash"  
     - Header: Add `api_key` with your Lookio API key.  
     - Tool description: "Call this tool when knowledge base is required."

   - Add **langchain.agent** node "AI Agent"  
     - Input text: `{{$json.final_user_message}}`  
     - System message: instruct agent to answer based on knowledge base, call Query knowledge base tool as needed, transparently communicate if insufficient knowledge, and adapt style for Telegram.  
     - Prompt type: define.  
     - Connect:  
       - `main` input from "Consolidate user message"  
       - `ai_memory` input from "Simple Memory"  
       - `ai_languageModel` input from "OpenAI Chat Model"  
       - `ai_tool` input from "Query knowledge base"

8. **Response Delivery**  
   - Add **telegram** node "Telegram answer"  
     - Text: `{{$json.output}}` (from AI Agent)  
     - Chat ID: `{{$json.message.from.id}}` (original Telegram user ID)  
     - Credentials: Telegram API.  
     - Connect AI Agent output to this node.

9. **Optional: Add Sticky Notes**  
   - Add explanatory sticky notes at logical segments for clarity (Process Audio, Process Text, AI Model, Lookio tool, etc.)

10. **Activate Workflow and Test**  
    - Ensure all credentials are correctly set: Telegram API, OpenAI API, Mistral Cloud API, Lookio API.  
    - Replace placeholders for Telegram username, Lookio API key, and assistant ID.  
    - Activate the workflow and send text or voice messages to your Telegram bot.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow was created by Guillaume Duvernay. It uses Lookio as the knowledge base provider with GPT-4.1 as the language model, and Mistral AI for audio transcription.                                                                                                                      | Creator credit                                |
| For Lookio setup, obtain your API key and assistant ID at [https://www.lookio.app/](https://www.lookio.app/).                                                                                                                                                                                  | Lookio official site                          |
| The Mistral transcription node uses the "voxtral-mini-2507" model; you can replace it with other supported models or services such as OpenAI Whisper for different transcription accuracy or cost profiles.                                                                                   | Mistral API documentation                     |
| Telegram bots are public by default. Using the "Myself?" node to filter messages based on your username makes the bot private during testing. Remove this node to make it fully public.                                                                                                        | Workflow privacy note                         |
| The AI Agent’s system message includes instructions to transparently communicate when the knowledge base lacks sufficient information, improving user trust.                                                                                                                                  | AI conversational design best practice        |
| Workflow uses LangChain’s agent APIs integrated into n8n, including memory buffer and tool calling, requiring n8n version with LangChain nodes support (v0.216.0 or later recommended).                                                                                                         | n8n LangChain nodes documentation             |

---

**Disclaimer:** The text provided is exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.