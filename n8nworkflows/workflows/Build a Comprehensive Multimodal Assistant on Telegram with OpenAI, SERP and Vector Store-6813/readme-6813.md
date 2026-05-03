Build a Comprehensive Multimodal Assistant on Telegram with OpenAI, SERP and Vector Store

https://n8nworkflows.xyz/workflows/build-a-comprehensive-multimodal-assistant-on-telegram-with-openai--serp-and-vector-store-6813


# Build a Comprehensive Multimodal Assistant on Telegram with OpenAI, SERP and Vector Store

### 1. Workflow Overview

This workflow, titled **"Build a Comprehensive Multimodal Assistant on Telegram with OpenAI, SERP and Vector Store"**, implements a powerful multimodal AI assistant named **J.A.R.V.I.S.** on the Telegram platform. It is designed to handle various message types — text, voice, images, and documents — and respond with contextually rich answers either as text or audio. The assistant leverages OpenAI’s language and vision models, web search APIs, web scraping, vector stores for document referencing, and text-to-speech generation.

**Target Use Cases:**  
- Advanced Telegram bot for customer support, personal assistant, or research assistant.  
- Multimodal input processing: text, voice, image, and document analysis.  
- Contextual AI responses enriched by external knowledge bases and web data.  
- Interactive and natural communication with users, including voice replies.

---

**Logical Blocks and Flow:**

- **1.1 Input Reception**  
  Telegram trigger node receives incoming messages and initializes the workflow.

- **1.2 Message Type Switching**  
  A Switch node routes messages based on type: text, voice, image, or document.

- **1.3 Processing by Type**  
  - **Image Processing:** Download, fix metadata, analyze content with OpenAI vision model.  
  - **Voice Processing:** Download voice file, transcribe to text using OpenAI.  
  - **Text Processing:** Format text input for AI processing.  
  - **Document Processing:** Download document, convert and store in temporary vector store for later retrieval.

- **1.4 AI Agent Core (J.A.R.V.I.S.)**  
  The main AI agent node uses OpenAI chat models, conversational memory, and multiple integrated tools (Google Search, Web Scraper, Image Generator, Calculator, Vector Store) to generate an intelligent, context-aware response.

- **1.5 Response Handling**  
  Conditional logic to send the response either as a text message or generate and send an audio reply.

- **1.6 Supporting Tools and Memory**  
  Includes vector store management for document context, token splitting for large messages, and session-based memory buffering.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  Captures incoming Telegram messages to trigger the workflow.

- **Nodes Involved:**  
  - Receive Message  
  - API Setup

- **Node Details:**  

| Node Name       | Details                                                                                                                                                   |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| Receive Message | - Type: Telegram Trigger node. <br>- Configured to listen for "message" updates only.<br>- Receives all message types from Telegram bot.<br>- No credentials shown but requires Telegram Bot API token.<br>- Output: raw Telegram message JSON.<br>- Potential failures: webhook issues, authentication errors, malformed payloads. |
| API Setup       | - Type: Set node.<br>- Assigns Jina API key (empty by default, to be configured).<br>- Passes data to the Switch node.<br>- No input connections (triggered by Receive Message).<br>- Potential failure: missing or invalid Jina API key could affect web scraping later. |

---

#### 2.2 Message Type Switching

- **Overview:**  
  Routes the message based on content type: image, voice, text, or document.

- **Nodes Involved:**  
  - Switch

- **Node Details:**  

| Node Name | Details                                                                                                                                                                              |
|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Switch    | - Type: Switch node.<br>- Conditions check existence of `message.photo`, `message.voice.file_id`, `message.text`, and `message.document.file_id` to classify input.<br>- Routes to 4 outputs:<br>1) Image branch<br>2) Voice branch<br>3) Text branch<br>4) Document branch.<br>- Version 3.2.<br>- Edge cases: messages with multiple types, unsupported types, or empty messages may cause routing failure or unhandled paths.<br>- Connects to respective processing nodes. |

---

#### 2.3 Processing by Message Type

---

##### 2.3.1 Image Processing

- **Overview:**  
  Downloads the image from Telegram, fixes file metadata, and analyzes the image content using the OpenAI vision model.

- **Nodes Involved:**  
  - Download Image  
  - Fix File Extension  
  - Analyze Image  
  - Format-image-output  
  - Text Response

- **Node Details:**  

| Node Name      | Details                                                                                                           |
|----------------|-------------------------------------------------------------------------------------------------------------------|
| Download Image | - Type: Telegram node.<br>- Downloads the highest resolution photo (index 2) from the message.<br>- Input: file ID from `message.photo[2].file_id`.<br>- Output: binary image data.<br>- Potential failures: file not found, download timeout, invalid file ID. |
| Fix File Extension | - Type: Code node.<br>- Dynamically sets MIME type based on the file extension from binary data.<br>- Ensures correct MIME type `image/{extension}` for downstream processing.<br>- Key code: updates `binary.data.mimeType`.<br>- Input: binary data from Download Image.<br>- Output: same item with fixed MIME type.<br>- Errors: missing fileExtension in binary data. |
| Analyze Image  | - Type: OpenAI node.<br>- Resource: image.<br>- Operation: analyze.<br>- Model: GPT-4O-MINI (OpenAI vision model).<br>- Input: base64-encoded image binary.<br>- Text parameter: uses message caption or default "Describe this image".<br>- Output: textual analysis of image content.<br>- Failures: API errors, rate limits, invalid image data. |
| Format-image-output | - Type: Set node.<br>- Prepares output JSON with keys `output` (text from analysis) and `type` set to "image".<br>- Input: output from Analyze Image.<br>- Output: JSON object for AI agent consumption.<br>- Edge cases: empty or null analysis results. |
| Text Response  | - Type: Telegram node.<br>- Sends text message response back to the user.<br>- Text content: JSON stringified analysis output.<br>- Chat ID from original message.<br>- Failures: Telegram API errors, invalid chat ID. |

---

##### 2.3.2 Voice Processing

- **Overview:**  
  Downloads voice message audio, transcribes it to text using OpenAI, and formats the output for AI processing.

- **Nodes Involved:**  
  - Get Audio File  
  - Transcribe  
  - Format-voice-output  
  - J.A.R.V.I.S.

- **Node Details:**  

| Node Name     | Details                                                                                                                  |
|---------------|--------------------------------------------------------------------------------------------------------------------------|
| Get Audio File | - Type: Telegram node.<br>- Downloads voice file using `message.voice.file_id`.<br>- Outputs binary audio data.<br>- Failures: invalid file ID, download failure. |
| Transcribe    | - Type: OpenAI node.<br>- Resource: audio.<br>- Operation: transcribe.<br>- Uses OpenAI speech-to-text API.<br>- Input: binary audio.<br>- Output: transcribed text.<br>- Failures: API errors, unsupported audio format, low quality audio. |
| Format-voice-output | - Type: Set node.<br>- Sets variables `chat_input` with transcribed text and `type` as "voice".<br>- Prepares data for the AI agent.<br>- Input: transcribed text.<br>- Output: formatted JSON.<br>- Edge cases: empty transcription. |
| J.A.R.V.I.S.  | - Main AI agent node.<br>- Receives formatted chat_input.<br>- Uses the transcription as AI input.<br>- Details described later in AI Agent block. |

---

##### 2.3.3 Text Processing

- **Overview:**  
  Formats incoming text messages for AI processing.

- **Nodes Involved:**  
  - Format-text-output  
  - J.A.R.V.I.S.

- **Node Details:**  

| Node Name        | Details                                                                                                    |
|------------------|------------------------------------------------------------------------------------------------------------|
| Format-text-output | - Type: Set node.<br>- Assigns `chat_input` with raw text message content.<br>- Sets `type` as "text".<br>- Input: raw message text.<br>- Output: formatted JSON for AI.<br>- Edge cases: empty or malformed text. |
| J.A.R.V.I.S.      | - Receives formatted text for AI processing.                                                             |

---

##### 2.3.4 Document Processing

- **Overview:**  
  Downloads documents, converts metadata, stores content in a temporary vector store to enable AI referencing during chat.

- **Nodes Involved:**  
  - Download Document  
  - Convert Doc  
  - Simple Vector Store  
  - Format-doc-output  
  - J.A.R.V.I.S.

- **Node Details:**  

| Node Name        | Details                                                                                                            |
|------------------|--------------------------------------------------------------------------------------------------------------------|
| Download Document | - Type: Telegram node.<br>- Downloads document file using `message.document.file_id`.<br>- Output: binary document data.<br>- Failures: file not found, large file size, unsupported format. |
| Convert Doc       | - Type: Code node.<br>- Sets MIME type dynamically based on Telegram document metadata.<br>- Prepares document for vector store ingestion.<br>- Input: binary from Download Document.<br>- Output: prepared binary with correct MIME type.<br>- Edge cases: missing mime_type field. |
| Simple Vector Store | - Type: Vector Store In-Memory node.<br>- Mode: insert.<br>- Stores document content indexed by vector embeddings.<br>- Clears previous store before insert to keep only last document.<br>- Input: document binary.<br>- Failures: memory overflow, embedding API errors. |
| Format-doc-output | - Type: Set node.<br>- Assigns `chat_input` with a message informing a document was uploaded and stored.<br>- Sets type as "document".<br>- Input: confirmation from vector store.<br>- Output: formatted message to AI agent.<br>- Edge cases: no document metadata. |
| J.A.R.V.I.S.      | - Receives document upload notification as input for context-aware responses.                                     |

---

#### 2.4 AI Agent Core (J.A.R.V.I.S.)

- **Overview:**  
  The central AI agent node that processes formatted inputs, manages context and memory, and orchestrates various AI tools to generate responses.

- **Nodes Involved:**  
  - J.A.R.V.I.S.  
  - Window Buffer Memory  
  - OpenAI Chat Model  
  - Basic Google Search (tool)  
  - Webpage Scraper (tool)  
  - Image Generator (tool)  
  - Calculator (tool)  
  - Think tool  
  - Simple Vector Store1 (temporary vector store retrieval)  
  - Embeddings OpenAI & Embeddings OpenAI1 (embedding creation)  
  - Default Data Loader & Token Splitter (document processing helpers)

- **Node Details:**  

| Node Name          | Details                                                                                                                                                                                                                 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| J.A.R.V.I.S.       | - Type: Langchain Agent Node.<br>- Receives `chat_input` and `type` to decide processing.<br>- System message sets assistant personality, session info, and tools available.<br>- Tools integrated: Search (Google via SerpAPI), Web Scraper (Jina API), Calculator, Image Generator (DALL·E), Temporary Vector Store.<br>- Uses conversational memory from Window Buffer Memory.<br>- Configured for max 10 iterations.<br>- Output: AI-generated textual response.<br>- Edge cases: API rate limits, tool failures, incomplete inputs.<br>- Requires OpenAI credentials and API keys for all tools (SerpAPI, Jina). |
| Window Buffer Memory | - Type: Langchain Memory Buffer Window.<br>- Session key: chat ID from Telegram message.<br>- Context window length: 10 messages.<br>- Provides conversational context memory to J.A.R.V.I.S.<br>- Edge cases: memory overflow, session key missing. |
| OpenAI Chat Model   | - Type: Langchain OpenAI Chat Model.<br>- Model: GPT-4.1.<br>- Used internally by J.A.R.V.I.S. for generating conversational responses.<br>- Requires OpenAI API credentials.<br>- Edge cases: model availability, API errors. |
| Basic Google Search | - Type: SerpAPI tool node.<br>- Enables Google search queries.<br>- Used by J.A.R.V.I.S. as a tool.<br>- Requires SerpAPI credentials.<br>- Failures: quota limits, network errors. |
| Webpage Scraper     | - Type: HTTP Request tool node.<br>- Scrapes web pages using Jina AI API.<br>- Requires Jina API key from API Setup node.<br>- Used by J.A.R.V.I.S. as a tool.<br>- Failures: invalid API key, network issues. |
| Image Generator     | - Type: HTTP Request tool node.<br>- Calls OpenAI’s DALL·E 3 API to generate images.<br>- Requires OpenAI credentials.<br>- Used by J.A.R.V.I.S. as a tool.<br>- Failures: API errors, invalid prompts. |
| Calculator         | - Type: Langchain Calculator tool.<br>- Performs math calculations.<br>- Used by J.A.R.V.I.S.<br>- Failures: invalid expressions. |
| Think              | - Type: Langchain Think tool.<br>- Used internally by J.A.R.V.I.S. as a scratchpad for reasoning before finalizing answers.<br>- No external input required.<br>- Helps ensure policy compliance and result validation. |
| Simple Vector Store1 | - Type: Vector Store In-Memory.<br>- Mode: retrieve-as-tool.<br>- Provides access to the temporary vector store containing uploaded document embeddings.<br>- Used by J.A.R.V.I.S. to reference document context.<br>- Failures: memory issues, missing documents. |
| Embeddings OpenAI & Embeddings OpenAI1 | - Type: Langchain Embeddings node.<br>- Creates vector embeddings from text or documents.<br>- Used with Simple Vector Stores for document indexing and retrieval.<br>- Requires OpenAI credentials.<br>- Failures: API issues, malformed input. |
| Default Data Loader | - Type: Langchain Document Default Data Loader.<br>- Loads binary documents for embedding.<br>- Works with Vector Store.<br>- Failures: unsupported document types. |
| Token Splitter     | - Type: Langchain Text Splitter.<br>- Splits documents into manageable chunks for embeddings.<br>- Works with Default Data Loader.<br>- Edge cases: incorrect splitting, loss of context. |

---

#### 2.5 Response Handling

- **Overview:**  
  Depending on the original message type (voice or others), either generates an audio response or sends a text reply.

- **Nodes Involved:**  
  - If Audio Response  
  - Split MSG  
  - Generate Audio  
  - Audio Response  
  - Text Response (also used here)

- **Node Details:**  

| Node Name       | Details                                                                                                                                                            |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| If Audio Response | - Type: If node.<br>- Condition: checks if the original message does NOT have a voice file ID (i.e., non-voice messages).<br>- Routes to either text or audio response branches.<br>- Edge cases: missing voice field, unexpected message types. |
| Split MSG       | - Type: Code node.<br>- Splits long AI response texts into chunks (~3600 chars) to avoid Telegram message length limits.<br>- Outputs array of message chunks.<br>- Input: AI-generated output text.<br>- Edge cases: very large messages, splitting mid-sentence. |
| Generate Audio  | - Type: OpenAI node.<br>- Resource: audio.<br>- Model: gpt-4o-mini-tts.<br>- Voice: echo.<br>- Speed: 0.9.<br>- Generates audio from AI text output.<br>- Input: AI response text.<br>- Failures: TTS API errors, unsupported text. |
| Audio Response  | - Type: Telegram node.<br>- Sends audio message back to user.<br>- Binary data enabled.<br>- Input: audio file from Generate Audio.<br>- Failures: Telegram audio upload issues. |
| Text Response   | - Also used here for non-voice messages.<br>- Sends textual response to Telegram chat.<br>- Input: chunks from Split MSG node. |

---

### 3. Summary Table

| Node Name            | Node Type                                  | Functional Role                          | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                                                      |
|----------------------|--------------------------------------------|----------------------------------------|------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Receive Message      | Telegram Trigger                           | Entry point, receives Telegram messages | -                      | API Setup                         | ## 1. Input message:                                                                                                             |
| API Setup            | Set                                        | Sets API keys (Jina)                    | Receive Message         | Switch                          | ## 2. Set API Keys<br>- Set Jina API                                                                                             |
| Switch               | Switch                                     | Routes by message type                  | API Setup               | Download Image, Get Audio File, Format-text-output, Download Document | ## 3. Switch by the kind of message:<br>- Text<br>- Voice<br>- Image<br>- Document                                                |
| Download Image       | Telegram                                   | Downloads photo from Telegram           | Switch (image)          | Fix File Extension               | ## 4. Image Chat<br>- Analyze the image using OpenAI Node                                                                        |
| Fix File Extension   | Code                                       | Fixes image MIME type                   | Download Image          | Analyze Image                   |                                                                                                                                |
| Analyze Image        | OpenAI                                     | Analyzes image content                  | Fix File Extension      | Format-image-output             |                                                                                                                                |
| Format-image-output  | Set                                        | Formats image analysis output           | Analyze Image           | Text Response                   |                                                                                                                                |
| Text Response        | Telegram                                   | Sends text response                     | Format-image-output, Split MSG | -                              | ## 9. Send Responses                                                                                                              |
| Get Audio File       | Telegram                                   | Downloads voice audio                   | Switch (voice)          | Transcribe                     | ## 5. Audio Chat<br>- Transcribe the audio using OpenAI Node                                                                    |
| Transcribe           | OpenAI                                     | Transcribes audio to text               | Get Audio File          | Format-voice-output            |                                                                                                                                |
| Format-voice-output  | Set                                        | Formats transcribed text for AI input  | Transcribe              | J.A.R.V.I.S.                   |                                                                                                                                |
| Format-text-output   | Set                                        | Formats text message for AI input       | Switch (text)           | J.A.R.V.I.S.                   | ## 6. Text Chat<br>- Format the text using Set Node                                                                              |
| Download Document    | Telegram                                   | Downloads document file                 | Switch (document)       | Convert Doc                    | ## 7. Document Chat<br>- Download Doc and store in Vector Store                                                                 |
| Convert Doc          | Code                                       | Sets MIME type metadata on document     | Download Document       | Simple Vector Store            |                                                                                                                                |
| Simple Vector Store  | Vector Store In-Memory                      | Stores document embeddings               | Convert Doc             | Format-doc-output             |                                                                                                                                |
| Format-doc-output    | Set                                        | Formats message about document upload   | Simple Vector Store     | J.A.R.V.I.S.                  |                                                                                                                                |
| J.A.R.V.I.S.         | Langchain Agent                            | Core AI agent processing user input    | Format-voice-output, Format-text-output, Format-doc-output | If Audio Response              | ## 8. J.A.R.V.I.S.<br>- This is the Agent Main Brain                                                                             |
| Window Buffer Memory | Langchain Memory Buffer Window              | Provides conversational memory          | -                      | J.A.R.V.I.S.                   | ## 8. Handle Responses<br>Using OpenAI Node<br>IF Node split messages: Audio/Text                                                  |
| OpenAI Chat Model    | Langchain OpenAI Chat Model                  | Language model used by AI agent         | -                      | J.A.R.V.I.S.                   |                                                                                                                                |
| Basic Google Search  | Langchain SerpAPI Tool                       | Provides Google search functionality    | -                      | J.A.R.V.I.S.                   | ## Search and Research Tools                                                                                                     |
| Webpage Scraper      | HTTP Request Tool                            | Scrapes web pages using Jina API        | -                      | J.A.R.V.I.S.                   |                                                                                                                                |
| Image Generator      | HTTP Request Tool                            | Generates images using DALL·E API       | -                      | J.A.R.V.I.S.                   | ## Image Tools                                                                                                                   |
| Calculator           | Langchain Calculator Tool                     | Performs math calculations               | -                      | J.A.R.V.I.S.                   | ## Reasoning Tools                                                                                                               |
| Think                | Langchain Think Tool                          | AI agent scratchpad for reasoning       | J.A.R.V.I.S.            | J.A.R.V.I.S.                   |                                                                                                                                |
| If Audio Response    | If Node                                     | Checks if response should be audio or text | J.A.R.V.I.S.          | Split MSG, Generate Audio       |                                                                                                                                |
| Split MSG            | Code                                        | Splits long messages into chunks        | If Audio Response       | Text Response                  |                                                                                                                                |
| Generate Audio       | OpenAI (Langchain audio)                      | Converts text response to audio          | If Audio Response       | Audio Response                 |                                                                                                                                |
| Audio Response       | Telegram                                    | Sends audio response to user             | Generate Audio          | -                              |                                                                                                                                |
| Embeddings OpenAI    | Langchain Embeddings                         | Creates embeddings for document storage | -                      | Simple Vector Store1           |                                                                                                                                |
| Simple Vector Store1 | Vector Store In-Memory                       | Retrieves from temporary vector store    | Embeddings OpenAI       | J.A.R.V.I.S.                   |                                                                                                                                |
| Default Data Loader  | Langchain Document Loader                    | Loads documents for embedding            | Token Splitter          | Simple Vector Store            |                                                                                                                                |
| Token Splitter       | Langchain Text Splitter                      | Splits documents into chunks             | -                      | Default Data Loader            |                                                                                                                                |
| Download Document    | Telegram                                    | Downloads document                        | Switch (document)       | Convert Doc                    | ## 7. Document Chat                                                                                                              |
| Sticky Notes         | Sticky Note                                 | Comments and explanations                 | -                      | -                              | Various notes throughout workflow explaining blocks and nodes with helpful external links.                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Configure Credentials:**  
   - Use @BotFather to create Telegram bot and get Bot Token.  
   - Add Telegram API credentials to n8n credentials.

2. **Add OpenAI API Credentials:**  
   - For all OpenAI, Langchain nodes, and AI tools.

3. **Add SerpAPI Credentials:**  
   - For Google Search tool node.

4. **Add Jina AI API Key:**  
   - For the Webpage Scraper node.

5. **Nodes Setup:**

   **Input Reception:**  
   1. Add **Telegram Trigger** node named "Receive Message". Configure to listen to "message" updates. Assign Telegram credentials.  
   2. Add a **Set** node named "API Setup" to store API keys (Jina API key). Connect from "Receive Message".

   **Switch Message Type:**  
   3. Add **Switch** node named "Switch" with four outputs:  
      - Condition 1: Check if `message.photo` exists => output "image".  
      - Condition 2: Check if `message.voice.file_id` exists => output "voice".  
      - Condition 3: Check if `message.text` exists => output "text".  
      - Condition 4: Check if `message.document.file_id` exists => output "document".  
   Connect "API Setup" to "Switch".

   **Image Processing Branch:**  
   4. Add **Telegram** node "Download Image" to download photo at index 2. Use `message.photo[2].file_id`.  
   5. Add **Code** node "Fix File Extension" to set MIME type dynamically based on file extension from binary data.  
   6. Add **OpenAI** node "Analyze Image" with resource as image, operation analyze, model GPT-4O-MINI. Input base64 binary from previous node, text input as message caption or default.  
   7. Add **Set** node "Format-image-output" to prepare output JSON with keys `output` and `type`="image".  
   8. Add **Telegram** node "Text Response" to send text back to user. Use chat ID from message.

   **Voice Processing Branch:**  
   9. Add **Telegram** node "Get Audio File" to download voice message audio file.  
   10. Add **OpenAI** node "Transcribe" with resource audio, operation transcribe.  
   11. Add **Set** node "Format-voice-output" to prepare `chat_input` with transcribed text and `type`="voice".  
   12. Connect to AI Agent "J.A.R.V.I.S." node.

   **Text Processing Branch:**  
   13. Add **Set** node "Format-text-output" to assign `chat_input` with raw message text and `type`="text".  
   14. Connect to AI Agent "J.A.R.V.I.S." node.

   **Document Processing Branch:**  
   15. Add **Telegram** node "Download Document" using `message.document.file_id`.  
   16. Add **Code** node "Convert Doc" to set MIME type using Telegram document metadata.  
   17. Add **Vector Store In-Memory** node "Simple Vector Store" in insert mode, clearing previous store.  
   18. Add **Set** node "Format-doc-output" to prepare a message informing the agent about document upload and store.  
   19. Connect to AI Agent "J.A.R.V.I.S." node.

   **AI Agent and Tools:**  
   20. Add Langchain **Agent** node "J.A.R.V.I.S." configured with:  
       - System prompt describing assistant role, tools, and user context.  
       - Max iterations 10.  
       - Integrate tools:  
         - Langchain OpenAI Chat Model (model GPT-4.1)  
         - Basic Google Search (SerpAPI)  
         - Webpage Scraper (Jina API)  
         - Image Generator (DALL·E 3)  
         - Calculator  
         - Think tool (scratchpad)  
         - Temporary Vector Store (retrieve-as-tool)  
       - Connect memory buffer node as conversational memory provider.

   21. Add **Window Buffer Memory** node for conversational context keyed by chat ID.

   22. Add **OpenAI Chat Model** node (GPT-4.1) linked to AI Agent.

   23. Add **Basic Google Search** node with SerpAPI credentials linked to AI Agent.

   24. Add **Webpage Scraper** node with Jina API key linked to AI Agent.

   25. Add **Image Generator** node using OpenAI DALL·E API linked to AI Agent.

   26. Add **Calculator** tool node linked to AI Agent.

   27. Add **Think** tool node linked to AI Agent.

   28. Add **Simple Vector Store1** node in retrieve-as-tool mode linked to AI Agent.

   29. Add **Embeddings OpenAI** nodes and **Default Data Loader**, **Token Splitter** nodes connected appropriately for document embedding and storage.

   **Response Handling:**  
   30. Add **If** node "If Audio Response" checking if original message has no voice file ID; routes to:  
       - Text response: Connect to **Code** node "Split MSG" to chunk long messages, then to Telegram "Text Response" node.  
       - Audio response: Connect to OpenAI TTS node "Generate Audio" (model gpt-4o-mini-tts, voice echo, speed 0.9), then to Telegram "Audio Response" node sending audio message.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow transforms your Telegram bot into a multimodal AI assistant capable of understanding text, voice, images, and documents, providing context-aware responses with integrated web search and document referencing. | See "Sticky Note12" for detailed project overview. |
| Helpful external links included for OpenAI Node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/ | Referenced in Sticky Notes 1, 4, 5. |
| Join the n8n Discord or Forum for community support and questions. Discord: https://discord.com/invite/XPKeKXeB7d; Forum: https://community.n8n.io/ | Included in project description. |
| Jina AI API key setup is required for Webpage Scraper to function correctly. | See API Setup node. |
| SerpAPI credentials are needed for the Basic Google Search node to enable web search. | See Basic Google Search node. |
| You can customize AI personality, models, and tools by editing the J.A.R.V.I.S. Agent system prompt and node configurations. | Described in project notes. |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively from an automated n8n workflow. The content complies with all applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.