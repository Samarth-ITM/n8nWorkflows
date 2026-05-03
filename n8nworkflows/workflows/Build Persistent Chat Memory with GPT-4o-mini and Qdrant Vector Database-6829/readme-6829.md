Build Persistent Chat Memory with GPT-4o-mini and Qdrant Vector Database

https://n8nworkflows.xyz/workflows/build-persistent-chat-memory-with-gpt-4o-mini-and-qdrant-vector-database-6829


# Build Persistent Chat Memory with GPT-4o-mini and Qdrant Vector Database

---

## 1. Workflow Overview

This workflow titled **"Build Persistent Chat Memory with GPT-4o-mini and Qdrant Vector Database"** implements a sophisticated long-term memory system for AI chat assistants. Its primary purpose is to enable conversational AI to remember, retrieve, and build upon past interactions with users across multiple sessions, thereby enhancing personalization, context retention, and response relevance.

### Target Use Cases
- Customer support bots requiring context continuity
- Personal AI assistants that learn user preferences
- Knowledge management and retrieval systems
- Educational tutoring systems with memory of past lessons

### High-Level Logical Blocks

- **1.1 Chat Input Reception:** Handles incoming chat messages as the entry point.
- **1.2 AI Agent Processing:** Processes input with memory-aware AI models, integrates long-term memory retrieval, and generates context-aware responses.
- **1.3 Memory Retrieval System:** Queries and retrieves relevant historical conversation vectors from Qdrant.
- **1.4 Embeddings Generation:** Converts text to vector representations for both storage and retrieval.
- **1.5 Text Processing Pipeline:** Prepares conversation data for vectorization through splitting and formatting.
- **1.6 Memory Storage:** Stores new conversation vectors and metadata into Qdrant for persistent memory.
- **1.7 Output Formatting:** Parses and formats AI responses into a structured format suitable for user display.

---

## 2. Block-by-Block Analysis

### 2.1 Chat Input Reception

**Overview:**  
This block initiates the workflow by receiving incoming chat messages via a webhook trigger, enabling real-time user interaction.

**Nodes Involved:**  
- When chat message received  
- Sticky Note 4

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (webhook-based)  
  - *Role:* Entry point capturing user messages in real-time  
  - *Configuration:* Default webhook, no special options  
  - *Inputs:* External user chat events  
  - *Outputs:* Passes chat message JSON to AI Agent  
  - *Version:* 1.1  
  - *Failure Modes:* Webhook unavailability or authentication errors if webhook URL not accessible  
  - *Sticky Note:* Explains chat interface role and webhook availability

- **Sticky Note 4**  
  - *Role:* Documentation for Chat Input block  
  - *Content Summary:* Describes features, integration options, and webhook URL usage

---

### 2.2 AI Agent Processing

**Overview:**  
Core AI logic for processing chat messages, integrating long-term memory retrieval, generating responses using GPT-4o-mini, and applying output parsing.

**Nodes Involved:**  
- AI Agent  
- GPT-4o-mini (Main)  
- OpenAI Chat Model  
- Structured Output Parser  
- Reranker Cohere  
- Sticky Note 10  
- Sticky Note 6

**Node Details:**

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Orchestrates AI model calls, memory querying, and response generation  
  - *Configuration:* Uses a detailed system prompt that governs memory retrieval, personalization, privacy, and response protocol  
  - *Inputs:* Receives chat messages from trigger; uses RAG_MEMORY tool for vector retrieval; uses GPT-4o-mini as language model  
  - *Outputs:* Structured AI response JSON passed to Store Conversation node  
  - *Version:* 2  
  - *Failure Modes:* API authentication errors, timeout, structured output parsing failures  
  - *Sticky Note 10:* Explains AI Agent’s core features, system prompt, performance, and costs

- **GPT-4o-mini (Main)**  
  - *Type:* OpenAI Chat Model (gpt-4o-mini)  
  - *Role:* Language model providing conversational response generation with controlled temperature and penalties  
  - *Configuration:* topP=0.7, temperature=0.2, presencePenalty=0.3, frequencyPenalty=0.6  
  - *Inputs:* Prompt and context from AI Agent  
  - *Outputs:* Textual response for AI Agent consumption  
  - *Version:* 1.2  
  - *Failure Modes:* API errors, rate limits

- **OpenAI Chat Model**  
  - *Type:* OpenAI Chat Model (secondary)  
  - *Role:* Used for structured output parsing downstream  
  - *Configuration:* Model gpt-4o-mini, maxTokens=2000, temperature=0.7  
  - *Inputs & Outputs:* Linked to Structured Output Parser  
  - *Version:* 1.2

- **Structured Output Parser**  
  - *Type:* Output Parser (JSON Schema)  
  - *Role:* Enforces a consistent response schema including sessionId, input, output, timestamp, relevanceScore  
  - *Configuration:* AutoFix enabled to correct minor schema violations  
  - *Inputs:* Raw AI response  
  - *Outputs:* Structured JSON for storage and further processing  
  - *Version:* 1.3  
  - *Failure Modes:* Parsing errors if AI output deviates significantly from schema

- **Reranker Cohere**  
  - *Type:* Cohere Re-ranker  
  - *Role:* Improves relevance of retrieved memory vectors by re-ranking top candidates  
  - *Configuration:* Uses Cohere API credentials  
  - *Inputs:* Retrieved vectors from RAG_MEMORY  
  - *Outputs:* Re-ranked vectors back to RAG_MEMORY  
  - *Version:* 1  
  - *Failure Modes:* API authentication/timeouts, optional node can be disabled for cost savings  
  - *Sticky Note 6:* Explains benefits, costs, and optional usage

- **Sticky Note 10**  
  - *Role:* Documentation on AI Agent’s features and system prompt

- **Sticky Note 6**  
  - *Role:* Documentation on relevance reranker benefits and costs

---

### 2.3 Memory Retrieval System

**Overview:**  
This block manages semantic search against stored conversation vectors in the Qdrant vector database to provide relevant historical context.

**Nodes Involved:**  
- RAG_MEMORY  
- Embeddings for Retrieval  
- Sticky Note 7  
- Sticky Note 5

**Node Details:**

- **RAG_MEMORY**  
  - *Type:* Vector Store Qdrant  
  - *Role:* Retrieves top 20 relevant conversation vectors from 'ltm' collection for AI Agent use  
  - *Configuration:* retrieval mode "retrieve-as-tool", topK=20, reranker enabled (Cohere)  
  - *Inputs:* Embeddings vectors for query from Embeddings for Retrieval node, re-ranked by Reranker Cohere  
  - *Outputs:* Relevant vectors passed to AI Agent via ai_tool input  
  - *Version:* 1.2  
  - *Failure Modes:* Qdrant API connection issues, data inconsistency, rate limits  
  - *Sticky Note 7:* Explains memory retrieval system configuration and performance

- **Embeddings for Retrieval**  
  - *Type:* OpenAI Embeddings  
  - *Role:* Generates 1024-dimensional vectors for user query text for semantic search  
  - *Configuration:* Model text-embedding-3-small, dimensions=1024 (must match vector DB)  
  - *Inputs:* Query text from AI Agent  
  - *Outputs:* Embeddings to RAG_MEMORY  
  - *Version:* 1.2  
  - *Failure Modes:* API errors, embedding generation failure  
  - *Sticky Note 5:* Emphasizes importance of embedding model consistency

- **Sticky Note 7**  
  - *Role:* Describes memory retrieval configurations and workflow

- **Sticky Note 5**  
  - *Role:* Notes on retrieval embeddings model and usage caution

---

### 2.4 Embeddings Generation & Text Processing

**Overview:**  
Processes conversation data by loading, splitting, and embedding text chunks for efficient vector storage.

**Nodes Involved:**  
- Default Data Loader  
- Recursive Character Text Splitter  
- Embeddings OpenAI (for storage)  
- Sticky Note (named "Embeddings OpenAI")  
- Sticky Note 2  
- Sticky Note 3

**Node Details:**

- **Default Data Loader**  
  - *Type:* Document Default Data Loader  
  - *Role:* Standardizes conversation data format before chunking  
  - *Inputs:* Output from Recursive Character Text Splitter  
  - *Outputs:* Prepared documents to Embeddings OpenAI for storage  
  - *Version:* 1

- **Recursive Character Text Splitter**  
  - *Type:* Text Splitter  
  - *Role:* Splits text into chunks of 200 characters with 40-character overlap to maintain context continuity  
  - *Inputs:* Raw conversation text from Default Data Loader  
  - *Outputs:* Text chunks for embedding  
  - *Version:* 1  
  - *Sticky Note 3:* Explains chunk size and overlap rationale, optimized for real-time chat

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings  
  - *Role:* Converts text chunks to 1024-dimensional embeddings for semantic storage  
  - *Configuration:* Model text-embedding-3-small, 1024 dimensions  
  - *Inputs:* Text chunks from Default Data Loader  
  - *Outputs:* Vectors to Store Conversation node  
  - *Version:* 1.2  
  - *Sticky Note (Embeddings OpenAI):* Details model choice, cost, and configuration tips

- **Sticky Note 2**  
  - *Role:* Describes document preparation for vector storage

- **Sticky Note 3**  
  - *Role:* Describes text chunking strategy and performance

---

### 2.5 Memory Storage

**Overview:**  
Stores new conversation embeddings and metadata into the Qdrant vector database for persistent long-term memory.

**Nodes Involved:**  
- Store Conversation  
- Sticky Note 11

**Node Details:**

- **Store Conversation**  
  - *Type:* Vector Store Qdrant  
  - *Role:* Inserts new conversation embeddings and related metadata into 'ltm' collection  
  - *Configuration:* Insert mode, batch processing, collection ‘ltm’  
  - *Inputs:* Embeddings from Embeddings OpenAI, structured output from AI Agent  
  - *Outputs:* Passes formatted response downstream  
  - *Version:* 1.2  
  - *Failure Modes:* Qdrant API errors, insert failures, data format issues  
  - *Sticky Note 11:* Describes memory storage contents, configuration, and scaling advice

---

### 2.6 Output Formatting

**Overview:**  
Extracts and formats the AI assistant’s response, ensuring users receive clean messages without meta-information.

**Nodes Involved:**  
- Format Response  
- Sticky Note 9

**Node Details:**

- **Format Response**  
  - *Type:* Set node  
  - *Role:* Extracts the AI response text from structured JSON and assigns it to "output" field for chat UI  
  - *Configuration:* Assigns output string expression from AI Agent’s structured output  
  - *Inputs:* Structured output JSON from Store Conversation  
  - *Outputs:* Clean text for chat interface display  
  - *Version:* 3.4

- **Sticky Note 9**  
  - *Role:* Explains the purpose of clean response formatting for user display

---

## 3. Summary Table

| Node Name                   | Node Type                                       | Functional Role                    | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                       |
|-----------------------------|------------------------------------------------|----------------------------------|--------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received  | Chat Trigger                                    | Entry point for chat input       | External webhook               | AI Agent                     | ## 💬 CHAT INTERFACE ... Webhook URL available after activation                                  |
| Sticky Note 4               | Sticky Note                                     | Chat input documentation         | —                              | —                            | ## 💬 CHAT INTERFACE ...                                                                          |
| AI Agent                   | Langchain Agent                                 | Core AI processing with memory   | When chat message received, RAG_MEMORY, GPT-4o-mini, Structured Output Parser | Store Conversation            | ## 🧠 INTELLIGENT AI AGENT ... Core features, system prompt, cost                              |
| GPT-4o-mini (Main)          | OpenAI Chat Model                              | Language model for responses     | AI Agent                      | AI Agent                     |                                                                                                |
| OpenAI Chat Model           | OpenAI Chat Model                              | Language model for output parsing| AI Agent                      | Structured Output Parser      |                                                                                                |
| Structured Output Parser    | Output Parser                                  | Enforces response schema         | OpenAI Chat Model             | AI Agent                     | ## 📐 OUTPUT FORMATTER ... Schema and auto-fix                                                  |
| Reranker Cohere             | Cohere Re-ranker                               | Re-ranks retrieval results       | RAG_MEMORY                   | RAG_MEMORY                   | ## 🎯 RELEVANCE OPTIMIZER ... Optional for cost savings                                         |
| Sticky Note 6               | Sticky Note                                     | Re-ranker documentation          | —                              | —                            | ## 🎯 RELEVANCE OPTIMIZER ...                                                                   |
| RAG_MEMORY                  | Vector Store Qdrant                            | Retrieves relevant memories      | Embeddings for Retrieval, Reranker Cohere  | AI Agent                     | ## 🧠 MEMORY RETRIEVAL SYSTEM ... Config and performance                                       |
| Embeddings for Retrieval    | OpenAI Embeddings                              | Generates query embeddings       | AI Agent                      | RAG_MEMORY                   | ## 🔍 RETRIEVAL EMBEDDINGS ... Must match storage embeddings                                    |
| Sticky Note 7               | Sticky Note                                     | Memory retrieval documentation   | —                              | —                            | ## 🧠 MEMORY RETRIEVAL SYSTEM ...                                                             |
| Default Data Loader         | Document Data Loader                           | Prepares conversation data       | Recursive Character Text Splitter | Embeddings OpenAI            | ## 📄 DOCUMENT PROCESSOR ... Data preparation                                                  |
| Recursive Character Text Splitter | Text Splitter                              | Splits text into chunks          | Default Data Loader           | Default Data Loader          | ## ✂️ TEXT CHUNKING STRATEGY ... Chunk size and overlap                                        |
| Embeddings OpenAI           | OpenAI Embeddings                              | Generates storage embeddings     | Default Data Loader           | Store Conversation           | ## 🔤 TEXT VECTORIZATION ... Model details and costs                                           |
| Sticky Note                 | Sticky Note                                     | Embeddings generation notes      | —                              | —                            | ## 🔤 TEXT VECTORIZATION ...                                                                |
| Store Conversation          | Vector Store Qdrant                            | Stores conversation embeddings   | AI Agent, Embeddings OpenAI   | Format Response              | ## 💾 MEMORY STORAGE ... Storage config and tips                                               |
| Sticky Note 11              | Sticky Note                                     | Memory storage documentation     | —                              | —                            | ## 💾 MEMORY STORAGE ...                                                                        |
| Format Response             | Set                                            | Formats AI response for UI       | Store Conversation            | —                            | ## 🎨 RESPONSE FORMATTER ... Cleans AI output                                                  |
| Sticky Note 9               | Sticky Note                                     | Response formatting documentation| —                              | —                            | ## 🎨 RESPONSE FORMATTER ...                                                                  |
| Sticky Note 3               | Sticky Note                                     | Text chunking strategy notes     | —                              | —                            | ## ✂️ TEXT CHUNKING STRATEGY ...                                                              |
| Sticky Note 2               | Sticky Note                                     | Document processor notes         | —                              | —                            | ## 📄 DOCUMENT PROCESSOR ...                                                                   |
| Sticky Note (Embeddings OpenAI) | Sticky Note                                 | Embeddings generation notes      | —                              | —                            | ## 🔤 TEXT VECTORIZATION ...                                                                 |
| Sticky Note 10              | Sticky Note                                     | AI Agent system prompt notes     | —                              | —                            | ## 🧠 INTELLIGENT AI AGENT ...                                                                |
| Workflow Overview           | Sticky Note                                     | Workflow general overview        | —                              | —                            | ## 🚀 WORKFLOW OVERVIEW ... Includes resource links                                            |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a webhook trigger node:**  
   - Name: `When chat message received`  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure default webhook options to receive incoming chat messages.

2. **Add the AI Agent node:**  
   - Name: `AI Agent`  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Paste the detailed system message prompt defining the long-term memory usage, privacy, and response guidelines.  
   - Enable output parser.  
   - Connect input from `When chat message received`.  
   - Configure to use two tools:  
     - `RAG_MEMORY` (vector store tool for memory retrieval)  
     - `GPT-4o-mini (Main)` as the language model.  
   - Set credentials for OpenAI API.

3. **Add GPT-4o-mini language model node:**  
   - Name: `GPT-4o-mini (Main)`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o-mini`  
   - Options: `topP=0.7`, `temperature=0.2`, `presencePenalty=0.3`, `frequencyPenalty=0.6`  
   - Connect output to AI Agent.

4. **Add OpenAI Chat Model for output parsing:**  
   - Name: `OpenAI Chat Model`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: `gpt-4o-mini`  
   - Options: `maxTokens=2000`, `temperature=0.7`  
   - Connect output to `Structured Output Parser`.

5. **Add Structured Output Parser:**  
   - Name: `Structured Output Parser`  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Enable AutoFix.  
   - Provide JSON schema example with keys: sessionId, chatInput, output, timestamp, relevanceScore.  
   - Connect input from `OpenAI Chat Model` output.  
   - Connect output to `AI Agent`.

6. **Set up vector store retrieval node:**  
   - Name: `RAG_MEMORY`  
   - Type: `@n8n/n8n-nodes-langchain.vectorStoreQdrant`  
   - Mode: `retrieve-as-tool`  
   - TopK: 20  
   - Collection: `ltm` (long-term memory)  
   - Enable reranker.  
   - Credentials: Qdrant API key.  
   - Connect input embeddings from `Embeddings for Retrieval`.  
   - Connect output to AI Agent `ai_tool` input.

7. **Add Cohere Re-ranker node:**  
   - Name: `Reranker Cohere`  
   - Type: `@n8n/n8n-nodes-langchain.rerankerCohere`  
   - Connect input from `RAG_MEMORY`.  
   - Connect output back to `RAG_MEMORY` reranker input.  
   - Provide Cohere API credentials.

8. **Add embeddings generation for retrieval:**  
   - Name: `Embeddings for Retrieval`  
   - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
   - Model: `text-embedding-3-small`  
   - Dimensions: 1024  
   - Credentials: OpenAI API key.  
   - Connect input from AI Agent query text.  
   - Connect output to `RAG_MEMORY`.

9. **Add document processing and chunking pipeline:**  
   - Add `Default Data Loader` node (document loader, default options).  
   - Add `Recursive Character Text Splitter` node:  
     - Chunk size: 200 characters  
     - Chunk overlap: 40 characters  
   - Connect `Recursive Character Text Splitter` output to `Default Data Loader`.  

10. **Add embeddings generation for storage:**  
    - Name: `Embeddings OpenAI`  
    - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
    - Model: `text-embedding-3-small`  
    - Dimensions: 1024  
    - Credentials: OpenAI API key  
    - Connect input from `Default Data Loader` output.  
    - Connect output to `Store Conversation`.

11. **Add vector store node to store conversation:**  
    - Name: `Store Conversation`  
    - Type: `@n8n/n8n-nodes-langchain.vectorStoreQdrant`  
    - Mode: `insert`  
    - Collection: `ltm`  
    - Credentials: Qdrant API key  
    - Connect input from `AI Agent` and `Embeddings OpenAI`.  
    - Connect output to `Format Response`.

12. **Add response formatting node:**  
    - Name: `Format Response`  
    - Type: `Set` node  
    - Assign the output field with expression: `={{ $('AI Agent').first().json.output.output }}`  
    - Connect input from `Store Conversation`.

13. **Configure credentials:**  
    - OpenAI API key for embedding and chat nodes  
    - Qdrant API key for vector store nodes  
    - Cohere API key for reranker (optional, can disable reranker for cost savings)

14. **Add sticky notes for documentation at relevant positions:**  
    - Workflow Overview  
    - Chat input explanation  
    - AI Agent system prompt and features  
    - Memory retrieval and storage details  
    - Embeddings and chunking strategy  
    - Output formatting notes

15. **Activate workflow and obtain webhook URL from `When chat message received` node.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow implements a persistent AI assistant memory using OpenAI GPT-4o-mini, Qdrant vector DB, and Cohere reranker for enhanced relevance.                           | Workflow Overview Sticky Note                        |
| Embeddings use `text-embedding-3-small` model with 1024 dimensions — optimal balance of cost (~$0.02 per 1M tokens) and performance for semantic search and storage.    | Embeddings OpenAI Sticky Note                        |
| Text chunking uses 200 characters with 40-character overlap to maintain conversational context continuity efficiently.                                                 | Recursive Character Text Splitter Sticky Note       |
| Cohere reranker improves retrieval relevance by 30-40%, reducing hallucinations but adds cost (~$1 per 1000 re-rankings); optional node for cost management.            | Reranker Cohere Sticky Note                          |
| Qdrant vector database stores long-term memory in ‘ltm’ collection; supports cloud or self-hosted setups. Cleanup policies recommended for production scalability.      | Store Conversation Sticky Note                       |
| AI Agent system prompt emphasizes privacy, ethical context separation, and continuous learning from conversation history.                                              | AI Agent Sticky Note                                 |
| Response formatting extracts clean AI messages for user display, omitting metadata to improve UX.                                                                       | Format Response Sticky Note                           |
| Useful external links:  
  - [n8n Documentation](https://docs.n8n.io)  
  - [Qdrant Setup Guide](https://qdrant.tech)  
  - [OpenAI Pricing](https://openai.com/pricing)                                                                 | Workflow Overview Sticky Note (links section)       |

---

*Disclaimer: The text provided is derived solely from an automated workflow created with n8n, adhering strictly to content policies, containing no illegal, offensive, or protected elements. All data processed is legal and public.*