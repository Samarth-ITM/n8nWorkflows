Voice AI Chatbot with OpenAI, RAG (Qdrant) & Guardrails for WordPress

https://n8nworkflows.xyz/workflows/voice-ai-chatbot-with-openai--rag--qdrant----guardrails-for-wordpress-10924


# Voice AI Chatbot with OpenAI, RAG (Qdrant) & Guardrails for WordPress

### 1. Workflow Overview

This workflow implements a **Voice AI Chatbot system for WordPress** that integrates speech recognition, safety guardrails, document retrieval using RAG (Retrieval-Augmented Generation) with a Qdrant vector database, and audio response generation. It is designed to connect with a WordPress Voicebot AI plugin through a webhook endpoint.

**Target Use Cases:**  
- Voice-enabled AI chatbots on WordPress sites  
- Real-time audio query processing and response generation  
- Contextual question answering using company-specific documents  
- Safe and moderated AI interactions to prevent unsafe or jailbreak content

**Logical Blocks:**

- **1.1 Input Reception and Speech-to-Text (STT):** Receives audio input via webhook and converts it to text.  
- **1.2 Guardrails Content Moderation:** Applies safety guardrails to filter or block unsafe/jailbreak queries.  
- **1.3 Retrieval-Augmented Generation (RAG) and AI Agent Processing:** Uses vector search on Qdrant to retrieve relevant documents, applies an AI agent with memory and calculator tools to generate a contextual response.  
- **1.4 Text-to-Speech (TTS) and Response Delivery:** Converts AI-generated text to audio and responds back through the webhook.  
- **2. Document Preparation and Vector Store Management:** Creates and manages Qdrant collections, imports documents from Google Drive, splits text, generates embeddings, and inserts vectors into Qdrant for retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Speech-to-Text (STT)

**Overview:**  
This block receives audio input from the WordPress voice plugin through a webhook, then transcribes the audio to text using OpenAI's speech-to-text capabilities.

**Nodes Involved:**  
- Webhook  
- Generate text (STT)

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook Receiver  
  - Role: Entry point to receive POST requests containing audio data from WordPress plugin  
  - Config: Path set to a unique webhook endpoint; allows all origins (CORS: *)  
  - Inputs: External HTTP POST with audio binary data and headers (including session-id)  
  - Outputs: Passes binary audio data to next node  
  - Edge Cases: Invalid/missing audio data, malformed requests, CORS issues, webhook URL misconfiguration  

- **Generate text (STT)**  
  - Type: OpenAI Audio Transcription Node  
  - Role: Transcribes incoming audio data to text  
  - Config: Uses OpenAI API credentials; expects binary property named "audio"  
  - Inputs: Binary audio from webhook node  
  - Outputs: Transcribed text in JSON format  
  - Edge Cases: API quota limits, network timeouts, unsupported audio formats, transcription errors

#### 1.2 Guardrails Content Moderation

**Overview:**  
Applies safety checks to the transcribed user query to detect potential jailbreak or unsafe content. Depending on outcome, routes either to AI agent or to a default safe response.

**Nodes Involved:**  
- Guardrails  
- OpenAI Chat Model2  
- Voicebot AI Agent  
- Default response (TTS)  
- Respond default to Webhook

**Node Details:**

- **Guardrails**  
  - Type: Content moderation node using LangChain Guardrails  
  - Role: Analyze input text for safety thresholds (jailbreak detection threshold 0.7)  
  - Config: Input text from STT node; jailbreaking filter configured with threshold  
  - Outputs: Two outputs: main (safe) to AI agent, alternative (unsafe) to default response  
  - Edge Cases: False positives/negatives, expression evaluation failures, API errors

- **OpenAI Chat Model2**  
  - Type: OpenAI Chat model (GPT-4.1-mini)  
  - Role: Intermediate natural language processing (likely used inside guardrail or for enhanced filtering)  
  - Config: Uses a dedicated OpenAI account; model choice optimized for moderation or lightweight tasks  
  - Edge Cases: API limits, model update requirements

- **Voicebot AI Agent**  
  - Type: LangChain Agent integrating AI model, memory, calculator, and RAG tools  
  - Role: Processes safe user queries with context and external tools to generate response text  
  - Config: Uses OpenAI Chat Model (GPT-5.1), linked with window buffer memory (context window 200), calculator tool, and RAG via Qdrant  
  - Inputs: Guardrails-validated text  
  - Outputs: AI-generated text response  
  - Edge Cases: Memory session ID issues, calculation errors, vector store failures

- **Default response (TTS)**  
  - Type: OpenAI Text-to-Speech Node  
  - Role: Generates fallback audio response for unsafe queries  
  - Config: Fixed text apologizing for policy violation; voice “onyx”  
  - Edge Cases: TTS API failures

- **Respond default to Webhook**  
  - Type: Webhook response node  
  - Role: Sends fallback audio binary back to caller  
  - Config: Responds with binary payload  
  - Edge Cases: Response timeout, connection errors

#### 1.3 Retrieval-Augmented Generation (RAG) and AI Agent Processing

**Overview:**  
Retrieves relevant company documents from Qdrant vector store to augment AI agent responses, enabling contextually informed answers.

**Nodes Involved:**  
- Qdrant Vector Store  
- Embeddings OpenAI  
- RAG (Tool Vector Store)  
- OpenAI Chat Model1  
- Window Buffer Memory  
- Calculator  

**Node Details:**

- **Qdrant Vector Store**  
  - Type: LangChain Vector Store node for Qdrant  
  - Role: Performs vector similarity search on document embeddings  
  - Config: Uses “documents” collection; authenticated via Qdrant API credentials  
  - Inputs: Embeddings from OpenAI node; query from RAG node  
  - Outputs: Relevant documents for context  
  - Edge Cases: API authentication errors, collection not found, connectivity issues

- **Embeddings OpenAI**  
  - Type: Embeddings generation using OpenAI  
  - Role: Creates vector embeddings for textual data  
  - Config: Uses OpenAI API; no special options  
  - Inputs: Text chunks of documents or queries  
  - Outputs: Embedding vectors to feed Qdrant  
  - Edge Cases: API limits, bad input data

- **RAG (Tool Vector Store)**  
  - Type: LangChain Tool for retrieval using vector store  
  - Role: Combines vector search results with AI prompt generation  
  - Config: Named “n3witalia_regreat_knowledgebase” with description for business data retrieval  
  - Inputs: Query text, vector store results  
  - Outputs: Augmented prompt to AI model  
  - Edge Cases: Empty results, slow response from vector store

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat model (GPT-based)  
  - Role: Generates refined AI responses based on retrieved documents and query  
  - Config: Uses OpenAI API credentials  
  - Edge Cases: Quota, model version updates

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer (windowed)  
  - Role: Maintains conversational context per session using session ID from webhook header  
  - Config: Custom session key expression extracting “session-id” from webhook headers, context window length 200  
  - Edge Cases: Missing session IDs, memory overflow, expression errors

- **Calculator**  
  - Type: LangChain Calculator tool  
  - Role: Enables AI agent to perform on-the-fly calculations as part of response generation  
  - Edge Cases: Calculation errors, unsupported expressions

#### 1.4 Text-to-Speech (TTS) and Response Delivery

**Overview:**  
Converts AI-generated text responses into audio (voice synthesis) and sends audio back through the webhook for playback in WordPress.

**Nodes Involved:**  
- Generate audio (TTS)  
- Respond to Webhook

**Node Details:**

- **Generate audio (TTS)**  
  - Type: OpenAI TTS Node  
  - Role: Converts text output from AI agent into speech audio  
  - Config: Voice “onyx”, uses OpenAI API for audio synthesis  
  - Inputs: Text from Voicebot AI Agent  
  - Outputs: Audio binary data for response  
  - Edge Cases: TTS API failure, unsupported text input

- **Respond to Webhook**  
  - Type: Webhook response node  
  - Role: Sends synthesized audio binary back to the caller (WordPress plugin)  
  - Config: Responds with binary content  
  - Edge Cases: Network errors, response timeouts

---

#### 2. Document Preparation and Vector Store Management

**Overview:**  
Manages creation and clearing of Qdrant collections, retrieves documents from Google Drive, processes and splits text, generates embeddings, and inserts document vectors into Qdrant. This enables the RAG functionality to provide contextual document retrieval.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (manual trigger)  
- Create collection (HTTP Request)  
- Clear collection (HTTP Request)  
- Search files (Google Drive)  
- Loop Over Items (Split in Batches)  
- Get files (Google Drive)  
- Default Data Loader1  
- Recursive Character Text Splitter  
- Embeddings OpenAI1  
- Insert file (Qdrant Vector Store)  
- Wait 5 sec.  
- Sticky Notes (instructions for setup)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts collection creation and clearing process for testing or setup  
  - Outputs: Triggers Create collection and Clear collection nodes

- **Create collection**  
  - Type: HTTP Request  
  - Role: Creates a new Qdrant collection with specified vector size (1536) and distance metric (Cosine)  
  - Config: PUT request to Qdrant API URL (replace placeholder QDRANTURL), JSON body defines collection parameters  
  - Auth: HTTP Header Auth with Qdrant API key  
  - Edge Cases: API errors, incorrect URL, permission issues

- **Clear collection**  
  - Type: HTTP Request  
  - Role: Deletes all points in specified Qdrant collection before re-import  
  - Config: POST request with empty filter to the points/delete endpoint  
  - Auth: Same as above  
  - Edge Cases: API errors, insufficient permissions

- **Search files**  
  - Type: Google Drive API  
  - Role: Lists files in a specific Google Drive folder (configured by folderId and driveId) for document import  
  - Auth: Google OAuth2 credentials  
  - Edge Cases: Permission denied, folder missing, empty folder

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes files batch-wise for scalable document ingestion

- **Get files**  
  - Type: Google Drive API  
  - Role: Downloads individual files for processing  
  - Config: Converts Google Docs format to plain text  
  - Edge Cases: File not found, conversion failure

- **Default Data Loader1**  
  - Type: LangChain Document Loader  
  - Role: Loads downloaded document text into LangChain pipeline, attaches metadata (file_id, file_name)  
  - Config: Reads binary data from specific field

- **Recursive Character Text Splitter**  
  - Type: LangChain Text Splitter  
  - Role: Splits large documents into chunks of 500 characters with 50 characters overlap, optimizing for embedding generation

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings Generator  
  - Role: Creates vector embeddings for document chunks

- **Insert file**  
  - Type: Qdrant Vector Store Insert  
  - Role: Inserts embeddings and metadata into Qdrant collection (“negozio-emporio-verde” in example)  
  - Auth: Qdrant API credentials  
  - Edge Cases: API limits, insert failures

- **Wait 5 sec.**  
  - Type: Wait node  
  - Role: Adds delay between insertions for rate limiting or API stability

- **Sticky Notes**  
  - Provide step-by-step setup instructions for each stage: collection creation, document upload/vectorization, WordPress plugin installation.

---

### 3. Summary Table

| Node Name             | Node Type                                | Functional Role                                 | Input Node(s)             | Output Node(s)                | Sticky Note                                                                                                          |
|-----------------------|----------------------------------------|------------------------------------------------|---------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook               | n8n-nodes-base.webhook                  | Receives audio input from WordPress plugin     | -                         | Generate text (STT)           |                                                                                                                      |
| Generate text (STT)    | @n8n/n8n-nodes-langchain.openAi        | Transcribes audio to text                        | Webhook                   | Guardrails                   |                                                                                                                      |
| Guardrails            | @n8n/n8n-nodes-langchain.guardrails    | Filters unsafe/jailbreak content                 | Generate text (STT)        | Voicebot AI Agent, Default response (TTS) | Sticky Note6: Add Guardrail to prevent jailbreak                                                                  |
| OpenAI Chat Model2     | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Supports guardrail evaluation                    | -                         | Guardrails                   |                                                                                                                      |
| Voicebot AI Agent     | @n8n/n8n-nodes-langchain.agent          | Processes safe queries with AI, memory, tools   | Guardrails                 | Generate audio (TTS)          |                                                                                                                      |
| Window Buffer Memory  | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains session conversation context           | Webhook                   | Voicebot AI Agent            |                                                                                                                      |
| Calculator            | @n8n/n8n-nodes-langchain.toolCalculator | Provides calculation capabilities to AI agent  | -                         | Voicebot AI Agent            |                                                                                                                      |
| RAG                   | @n8n/n8n-nodes-langchain.toolVectorStore | Retrieves relevant documents from Qdrant         | Qdrant Vector Store, OpenAI Chat Model1 | Voicebot AI Agent            |                                                                                                                      |
| Qdrant Vector Store    | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Searches vector database for document similarity | Embeddings OpenAI          | RAG                          |                                                                                                                      |
| Embeddings OpenAI     | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for queries/documents       | -                         | Qdrant Vector Store          |                                                                                                                      |
| OpenAI Chat Model1    | @n8n/n8n-nodes-langchain.lmChatOpenAi  | Generates AI response using retrieved context    | RAG                       | Voicebot AI Agent            |                                                                                                                      |
| Generate audio (TTS)  | @n8n/n8n-nodes-langchain.openAi        | Converts AI text response to speech audio        | Voicebot AI Agent          | Respond to Webhook           | Sticky Note7: Voicebot AI agent and generate output audio                                                            |
| Respond to Webhook    | n8n-nodes-base.respondToWebhook          | Sends audio response back to WordPress plugin   | Generate audio (TTS)       | -                           |                                                                                                                      |
| Default response (TTS)| @n8n/n8n-nodes-langchain.openAi        | Generates safe fallback audio response           | Guardrails (unsafe output) | Respond default to Webhook   |                                                                                                                      |
| Respond default to Webhook | n8n-nodes-base.respondToWebhook       | Sends fallback audio response                     | Default response (TTS)     | -                           |                                                                                                                      |
| When clicking ‘Test workflow’ | n8n-nodes-base.manualTrigger        | Manual start for collection setup                 | -                         | Create collection, Clear collection |                                                                                                                  |
| Create collection     | n8n-nodes-base.httpRequest               | Creates Qdrant vector collection                   | When clicking ‘Test workflow’ | Clear collection          | Sticky Note3: Create Qdrant Collection; update QDRANTURL, COLLECTION                                                 |
| Clear collection      | n8n-nodes-base.httpRequest               | Clears all points in Qdrant collection             | Create collection          | Search files                 |                                                                                                                      |
| Search files          | n8n-nodes-base.googleDrive               | Lists Google Drive files in target folder          | Clear collection           | Loop Over Items              | Sticky Note4: Documents vectorization with Qdrant and Google Drive; update QDRANTURL, COLLECTION                     |
| Loop Over Items       | n8n-nodes-base.splitInBatches            | Iterates through files for processing              | Search files               | Get files, (empty batch)     |                                                                                                                      |
| Get files             | n8n-nodes-base.googleDrive               | Downloads individual file from Google Drive        | Loop Over Items            | Default Data Loader1         |                                                                                                                      |
| Default Data Loader1  | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads document text with metadata                    | Get files                  | Recursive Character Text Splitter |                                                                                                                  |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits documents into chunks for embeddings         | Default Data Loader1       | Embeddings OpenAI1           |                                                                                                                      |
| Embeddings OpenAI1    | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for document chunks            | Recursive Character Text Splitter | Insert file               |                                                                                                                      |
| Insert file           | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Inserts embeddings into Qdrant collection            | Embeddings OpenAI1         | Wait 5 sec.                 |                                                                                                                      |
| Wait 5 sec.           | n8n-nodes-base.wait                      | Delays processing for rate limiting or stability    | Insert file                | Loop Over Items              |                                                                                                                      |
| Sticky Note           | n8n-nodes-base.stickyNote                | Setup instructions and comments                      | -                         | -                           | Sticky Note: Install WordPress Voicebot AI Agent plugin and set webhook URL                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - Method: POST  
   - Path: Unique identifier (e.g., `7de25617-5e89-4a54-b9aa-5a353705ddc4`)  
   - Allowed Origins: `*` (for CORS)  
   - Response Mode: `responseNode` (to send response from another node)  

2. **Add OpenAI STT Node**  
   - Type: `OpenAI` (resource: audio, operation: transcribe)  
   - Binary Property Name: `audio`  
   - Credentials: OpenAI API (STT account)  
   - Connect Webhook main output to this node  

3. **Add Guardrails Node**  
   - Type: `Guardrails` (LangChain)  
   - Input Text: Expression `{{$json.text}}` from STT output  
   - Guardrails Config: Jailbreak threshold set at 0.7  
   - Connect STT output to Guardrails input  

4. **Add OpenAI Chat Model for Guardrail Support**  
   - Type: `OpenAI Chat Model` (model: `gpt-4.1-mini`)  
   - Credentials: OpenAI API (moderation account)  
   - Connect as required to Guardrails model input  

5. **Add Voicebot AI Agent Node**  
   - Type: `LangChain Agent`  
   - Text Input: From Guardrails safe output property (e.g., `{{$json.guardrailsInput}}`)  
   - System Message: Defines assistant persona (e.g., “Jarvis”), instructions to answer company questions, consider current date/time, admit unknowns  
   - Tools: Attach Calculator, RAG tool, and Memory nodes  
   - Credentials: OpenAI API (chat account)  
   - Connect Guardrails safe output to this node  

6. **Add Window Buffer Memory Node**  
   - Type: `LangChain Memory Buffer Window`  
   - Session Key: Expression `={{ $('Webhook').item.json.headers["session-id"] }}`  
   - Context Window Length: 200  
   - Connect Webhook node output to Memory node input  
   - Connect Memory output to Voicebot AI Agent memory input  

7. **Add Calculator Node**  
   - Type: `LangChain Calculator Tool`  
   - Connect Calculator output to Voicebot AI Agent tool input  

8. **Add RAG Tool Node**  
   - Type: `LangChain Tool Vector Store`  
   - Name: `n3witalia_regreat_knowledgebase`  
   - Description: "Retrieve data about the business"  
   - Connect Qdrant Vector Store output to RAG input  
   - Connect RAG output to Voicebot AI Agent tool input  

9. **Add Qdrant Vector Store Node**  
   - Type: `LangChain Vector Store Qdrant`  
   - Collection: `documents` (or your collection name)  
   - Credentials: Qdrant API with appropriate URL and key  
   - Connect Embeddings OpenAI output to this node  
   - Connect this node output to RAG input  

10. **Add Embeddings OpenAI Node**  
    - Type: `OpenAI Embeddings`  
    - Credentials: OpenAI API  
    - Connect text input (documents or queries) to this node  
    - Connect output to Qdrant Vector Store input  

11. **Add OpenAI Chat Model1 Node**  
    - Type: `OpenAI Chat Model`  
    - Credentials: OpenAI API (chat account)  
    - Connect RAG output to this node  
    - Connect output to Voicebot AI Agent AI language model input  

12. **Add Generate Audio (TTS) Node**  
    - Type: `OpenAI` (resource: audio)  
    - Voice: `onyx`  
    - Input: Text output from Voicebot AI Agent  
    - Credentials: OpenAI API (TTS account)  
    - Connect Voicebot AI Agent output to this node  

13. **Add Respond to Webhook Node**  
    - Type: `Respond to Webhook`  
    - Respond with: binary (audio)  
    - Connect Generate Audio (TTS) output to this node  

14. **Add Default Response (TTS) Node**  
    - Type: `OpenAI` (resource: audio)  
    - Voice: `onyx`  
    - Input: Fixed text "I’m sorry, but I’m unable to answer this question because it does not comply with our policies."  
    - Credentials: OpenAI API (TTS account)  

15. **Add Respond Default to Webhook Node**  
    - Type: `Respond to Webhook`  
    - Respond with: binary  
    - Connect Default Response (TTS) output to this node  
    - Connect Guardrails unsafe output to Default Response (TTS) node  

16. **Document Preparation Sub-Workflow Setup:**  
    - Manually trigger node to start process  
    - HTTP Request node to create Qdrant collection (PUT, with JSON body specifying vector size, distance, etc.)  
    - HTTP Request node to clear Qdrant collection (POST, points/delete with empty filter)  
    - Google Drive node to search files in target folder (requires Google OAuth credentials)  
    - SplitInBatches node to loop over files  
    - Google Drive node to download each file (convert Google Docs to plain text)  
    - LangChain Default Data Loader node to load document text with metadata  
    - Recursive Character Text Splitter node to chunk text (chunk size 500, overlap 50)  
    - OpenAI Embeddings node to generate vector embeddings  
    - Qdrant Vector Store insert node to insert embeddings into collection  
    - Wait node (5 seconds) between inserts to avoid rate limits  
    - Loop back to process all files  

17. **WordPress Plugin Integration**  
    - Install the WordPress Voicebot AI Agent plugin from: https://n3wstorage.b-cdn.net/n3witalia/voice-chatbot-n8n.zip  
    - Configure plugin webhook URL to point to the deployed n8n webhook endpoint  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow integrates multiple OpenAI accounts dedicated to different tasks: STT, TTS, chat moderation, and chat generation.                                                                                                   | Best practice for API quota management and role separation                                        |
| Guardrails node helps prevent unsafe or jailbreak queries, enhancing security and compliance.                                                                                                                                     | Guardrails docs: https://docs.langchain.com/docs/integrations/guardrails                          |
| The WordPress plugin (Voicebot AI Agent) must be installed and correctly configured to utilize this workflow effectively.                                                                                                        | Plugin download: https://n3wstorage.b-cdn.net/n3witalia/voice-chatbot-n8n.zip                     |
| Qdrant vector database endpoint and API key must be configured properly; placeholders like QDRANTURL and COLLECTION must be replaced with actual values.                                                                         | Qdrant docs: https://qdrant.tech/documentation/                                                   |
| Google Drive OAuth2 credentials must have permission to access the specified Drive and folder for document ingestion.                                                                                                            | Google Drive API docs: https://developers.google.com/drive/api/v3/about-sdk                        |
| Conversation memory is managed per user session via session ID header from webhook, enabling contextual dialogues.                                                                                                              | Session ID header must be provided by the WordPress plugin in requests                            |
| Rate limiting and delays (Wait node) are used in document ingestion to avoid API throttling or server overload.                                                                                                                  | Adjust wait times based on API limits and server performance                                     |
| System message in AI agent defines assistant persona "Jarvis" and instructs to answer with date/time awareness and honesty about unknowns.                                                                                       | Can be customized for different personalities or business rules                                  |
| This workflow requires n8n version supporting LangChain integration nodes (version 1.3+ for memory, 1.7+ for openAi audio nodes).                                                                                                | Check compatibility before importing                                                             |

---

**Disclaimer:**  
The provided content derives exclusively from an automated n8n workflow and adheres strictly to current content policies. It contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.