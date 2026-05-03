Build a PDF-Based RAG System with OpenAI, Pinecone and Cohere Reranking

https://n8nworkflows.xyz/workflows/build-a-pdf-based-rag-system-with-openai--pinecone-and-cohere-reranking-5734


# Build a PDF-Based RAG System with OpenAI, Pinecone and Cohere Reranking

### 1. Workflow Overview

This workflow implements a PDF-Based Retrieval-Augmented Generation (RAG) system leveraging OpenAI, Pinecone vector database, and Cohere reranking. Its primary purpose is to enable users to upload PDF documents, automatically convert the content into vector embeddings, store these embeddings in Pinecone, and subsequently perform AI-driven question answering using chat models enhanced by vector retrieval and reranking capabilities.

The workflow is logically structured into two main blocks:

- **1.1 Data Ingestion and Indexing**  
  Handles PDF upload, document loading, text splitting, embedding generation, and insertion into Pinecone vector store.

- **1.2 AI Chat Interaction and Retrieval**  
  Manages chat input reception, query processing with vector retrieval from Pinecone, reranking results with Cohere, and responding via an AI chat agent mediated by OpenAI.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion and Indexing

**Overview:**  
This block receives a PDF file from a user submission form, extracts and splits the text content, generates embeddings using OpenAI, and stores the resulting vectors in the Pinecone vector database.

**Nodes Involved:**  
- On form submission  
- Default Data Loader  
- Recursive Character Text Splitter  
- Embeddings OpenAI  
- Pinecone Vector Store  
- Sticky Note (Insert Data to Pinecone)

**Node Details:**

- **On form submission**  
  - *Type:* `formTrigger` (Webhook trigger)  
  - *Role:* Initiates the workflow by capturing PDF file uploads via a web form titled "Upload RAG PDF".  
  - *Config:* Accepts only one PDF file, required field, no additional options.  
  - *Connections:* Outputs to `Pinecone Vector Store` (via document data)  
  - *Failure modes:* Upload errors, unsupported file types, large file sizes exceeding limits.

- **Default Data Loader**  
  - *Type:* `documentDefaultDataLoader`  
  - *Role:* Loads binary PDF data from the form submission and prepares it for processing.  
  - *Config:* Mode set to handle binary data, text splitting configured as "custom" for downstream splitting.  
  - *Input:* Binary PDF file from submission  
  - *Output:* Document data stream for splitting  
  - *Failure modes:* Corrupted PDF, unsupported encoding.

- **Recursive Character Text Splitter**  
  - *Type:* `textSplitterRecursiveCharacterTextSplitter`  
  - *Role:* Splits the entire document text into smaller chunks recursively to optimize embedding quality and retrieval performance.  
  - *Config:* Default options without customization.  
  - *Input:* Document text from loader  
  - *Output:* Split text chunks  
  - *Failure modes:* Extremely large documents may cause performance issues.

- **Embeddings OpenAI**  
  - *Type:* `embeddingsOpenAi`  
  - *Role:* Converts each text chunk into vector embeddings using OpenAI embedding models.  
  - *Config:* Embedding dimension set to 1024, using configured OpenAI credentials.  
  - *Input:* Text chunks  
  - *Output:* Embeddings passed to Pinecone insertion  
  - *Failure modes:* API rate limits, invalid credentials, network issues.

- **Pinecone Vector Store (Insert mode)**  
  - *Type:* `vectorStorePinecone`  
  - *Role:* Inserts the generated embeddings into the Pinecone vector index named "n8n".  
  - *Config:* Mode set to "insert", no advanced options, uses Pinecone API credentials.  
  - *Input:* Embeddings  
  - *Output:* None (end of insert chain)  
  - *Failure modes:* Pinecone API errors, authentication failures, index capacity limits.

- **Sticky Note (Insert Data to Pinecone)**  
  - *Type:* `stickyNote`  
  - *Role:* Documentation placeholder visually grouping this block.  
  - *Content:* "## Insert Data to Pinecone"  

---

#### 2.2 AI Chat Interaction and Retrieval

**Overview:**  
This block listens for chat messages, queries the Pinecone vector database for relevant document embeddings, reranks them with Cohere, and uses OpenAI chat models to generate responses based on retrieved knowledge, maintaining chat context with simple memory.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory  
- VectorDB (Pinecone retrieve mode)  
- Reranker Cohere  
- Sticky Note1 (Chat AI Agent)  
- Sticky Note2 (Embedding Model)

**Node Details:**

- **When chat message received**  
  - *Type:* `chatTrigger` (Webhook trigger)  
  - *Role:* Triggers the workflow upon receiving a user chat message for querying the RAG system.  
  - *Config:* Default, no extra filters.  
  - *Output:* Connects to the AI Agent node.  
  - *Failure modes:* Webhook issues, malformed input messages.

- **AI Agent**  
  - *Type:* `agent` (LangChain AI agent node)  
  - *Role:* Coordinates between memory, language model, vector database tool, and reranker to process the chat query.  
  - *Config:* System message instructs the agent to answer only based on the VectorDB tool data, else respond "I don't know".  
  - *Input:* Chat message trigger  
  - *Outputs:* AI response to chat  
  - *Connections:* Receives input from chat trigger, connects to Simple Memory, OpenAI Chat Model, and VectorDB nodes.  
  - *Failure modes:* Model errors, incomplete tool integration, expression errors.

- **OpenAI Chat Model**  
  - *Type:* `lmChatOpenAi`  
  - *Role:* Provides the language model capabilities using GPT-4.1 to generate conversational responses.  
  - *Config:* Model set explicitly to "gpt-4.1" via a cached list selection, uses OpenAI credentials.  
  - *Input:* Query and context from AI Agent  
  - *Output:* Chat completion results  
  - *Failure modes:* API limits, invalid credentials.

- **Simple Memory**  
  - *Type:* `memoryBufferWindow`  
  - *Role:* Maintains a windowed context memory of the conversation to provide continuity in chat sessions.  
  - *Config:* Default buffer window, no additional parameters.  
  - *Input:* Chat history  
  - *Output:* Contextual memory data for AI Agent  
  - *Failure modes:* Memory overflow, context misalignment.

- **VectorDB (Pinecone retrieve-as-tool mode)**  
  - *Type:* `vectorStorePinecone`  
  - *Role:* Retrieves top 20 relevant vectors from Pinecone based on the chat query, acting as a knowledge base tool.  
  - *Config:* Mode set to "retrieve-as-tool", topK=20 results, reranking enabled, uses Pinecone API credentials.  
  - *Input:* Query vector from embeddings  
  - *Output:* Retrieved documents to AI Agent  
  - *Failure modes:* API errors, empty results, authentication issues.

- **Reranker Cohere**  
  - *Type:* `rerankerCohere`  
  - *Role:* Re-ranks the retrieved results from Pinecone to improve relevance using Cohere's reranking API.  
  - *Config:* Default settings, uses Cohere API credentials.  
  - *Input:* Retrieved vectors from VectorDB  
  - *Output:* Reranked results forwarded to VectorDB node  
  - *Failure modes:* API rate limits, invalid credentials.

- **Sticky Note1 (Chat AI Agent)**  
  - *Type:* `stickyNote`  
  - *Role:* Visually groups nodes involved in chat interaction.  
  - *Content:* "## Chat AI Agent"

- **Sticky Note2 (Embedding Model)**  
  - *Type:* `stickyNote`  
  - *Role:* Documentation for embedding-related nodes.  
  - *Content:* "## Embedding Model"

---

### 3. Summary Table

| Node Name                | Node Type                                | Functional Role                     | Input Node(s)                         | Output Node(s)                  | Sticky Note               |
|--------------------------|----------------------------------------|-----------------------------------|-------------------------------------|-------------------------------|---------------------------|
| On form submission       | formTrigger                            | Receive PDF upload                 | -                                   | Pinecone Vector Store          |                           |
| Default Data Loader      | documentDefaultDataLoader              | Load binary PDF                   | Pinecone Vector Store (via form)    | Recursive Character Text Splitter |                           |
| Recursive Character Text Splitter | textSplitterRecursiveCharacterTextSplitter | Split document text into chunks | Default Data Loader                  | Embeddings OpenAI             |                           |
| Embeddings OpenAI        | embeddingsOpenAi                       | Generate text embeddings          | Recursive Character Text Splitter   | Pinecone Vector Store          | Sticky Note2: Embedding Model |
| Pinecone Vector Store    | vectorStorePinecone                   | Insert embeddings into Pinecone   | Embeddings OpenAI, On form submission | -                            | Sticky Note: Insert Data to Pinecone |
| Sticky Note              | stickyNote                            | Visual documentation              | -                                   | -                             | Insert Data to Pinecone    |
| When chat message received | chatTrigger                          | Trigger on chat input             | -                                   | AI Agent                      |                           |
| AI Agent                 | agent                                | Coordinate query processing       | When chat message received           | OpenAI Chat Model, VectorDB, Simple Memory | Sticky Note1: Chat AI Agent |
| OpenAI Chat Model        | lmChatOpenAi                         | Generate chat responses           | AI Agent                           | AI Agent                      | Sticky Note1: Chat AI Agent |
| Simple Memory            | memoryBufferWindow                   | Maintain chat context             | AI Agent                           | AI Agent                      | Sticky Note1: Chat AI Agent |
| VectorDB                 | vectorStorePinecone                  | Retrieve vectors from Pinecone    | Reranker Cohere, AI Agent           | AI Agent                      | Sticky Note1: Chat AI Agent |
| Reranker Cohere          | rerankerCohere                      | Rerank retrieved results          | VectorDB                           | VectorDB                      | Sticky Note1: Chat AI Agent |
| Sticky Note1             | stickyNote                          | Visual documentation              | -                                   | -                             | Chat AI Agent             |
| Sticky Note2             | stickyNote                          | Visual documentation              | -                                   | -                             | Embedding Model           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "On form submission" node:**  
   - Type: `formTrigger`  
   - Configure webhook with form title "Upload RAG PDF".  
   - Add one required file field accepting only `.pdf`.  

2. **Create "Default Data Loader":**  
   - Type: `documentDefaultDataLoader`  
   - Set data type to "binary" and text splitting mode to "custom".  
   - Connect input from "On form submission" node (PDF file).  

3. **Create "Recursive Character Text Splitter":**  
   - Type: `textSplitterRecursiveCharacterTextSplitter`  
   - Use default settings.  
   - Connect input from "Default Data Loader".  

4. **Create "Embeddings OpenAI":**  
   - Type: `embeddingsOpenAi`  
   - Set embedding dimension to 1024.  
   - Attach OpenAI API credentials.  
   - Connect input from "Recursive Character Text Splitter".  

5. **Create "Pinecone Vector Store" (Insert mode):**  
   - Type: `vectorStorePinecone`  
   - Set mode to "insert".  
   - Select Pinecone index named "n8n".  
   - Attach Pinecone API credentials.  
   - Connect input from "Embeddings OpenAI".  

6. **Create "When chat message received" node:**  
   - Type: `chatTrigger`  
   - Set up webhook listening for incoming chat messages.  

7. **Create "AI Agent":**  
   - Type: `agent`  
   - Configure system message: "Hanya jawab berdasarkan data yang ada di tools \"VectorDB\". Kalau data disitu gak ada, jawab saja kamu tidak tahu."  
   - Connect input from "When chat message received".  

8. **Create "OpenAI Chat Model":**  
   - Type: `lmChatOpenAi`  
   - Choose model "gpt-4.1" from the list.  
   - Attach OpenAI API credentials.  
   - Connect input from "AI Agent" (ai_languageModel).  

9. **Create "Simple Memory":**  
   - Type: `memoryBufferWindow`  
   - Use default parameters.  
   - Connect to "AI Agent" (ai_memory).  

10. **Create "VectorDB" (Pinecone retrieve-as-tool mode):**  
    - Type: `vectorStorePinecone`  
    - Set mode to "retrieve-as-tool".  
    - Set topK to 20.  
    - Enable reranking.  
    - Select Pinecone index "n8n".  
    - Attach Pinecone API credentials.  
    - Connect to "AI Agent" (ai_tool).  

11. **Create "Reranker Cohere":**  
    - Type: `rerankerCohere`  
    - Attach Cohere API credentials.  
    - Connect input from "VectorDB" (ai_reranker).  
    - Connect output back to "VectorDB".  

12. **Add Sticky Notes for clarity:**  
    - Insert a sticky note near data insertion nodes with content "## Insert Data to Pinecone".  
    - Insert sticky notes near chat-related nodes with "## Chat AI Agent".  
    - Insert sticky note near embedding nodes with "## Embedding Model".  

13. **Verify all connections and credentials:**  
    - Ensure OpenAI and Pinecone API credentials are valid and linked.  
    - Ensure Cohere API credentials are valid for reranking.  
    - Test form submission and chat webhook triggers.  

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                          |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| The system message in AI Agent is in Indonesian: "Hanya jawab berdasarkan data yang ada di tools \"VectorDB\". Kalau data disitu gak ada, jawab saja kamu tidak tahu." | Indicates a localized prompt restricting AI answers to vector DB data. |
| Pinecone index used is named "n8n" and must exist with proper API keys configured.                | Pinecone vector database setup required prior to workflow use.          |
| Cohere API is used specifically for reranking retrieved documents to improve answer relevance.    | Requires Cohere API Trial or subscription.                              |
| OpenAI GPT-4.1 is used as the chat language model, requiring appropriate API access.              | OpenAI API key must support GPT-4.1 usage.                              |
| Sticky notes are used extensively to visually group workflow logic areas for maintainability.    | Improves user understanding and maintenance of workflow logic.          |

---

**Disclaimer:** The provided workflow is an automated n8n integration respecting all content policies. It handles only legal and public data.