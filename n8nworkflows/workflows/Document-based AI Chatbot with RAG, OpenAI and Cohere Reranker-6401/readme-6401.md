Document-based AI Chatbot with RAG, OpenAI and Cohere Reranker

https://n8nworkflows.xyz/workflows/document-based-ai-chatbot-with-rag--openai-and-cohere-reranker-6401


# Document-based AI Chatbot with RAG, OpenAI and Cohere Reranker

### 1. Workflow Overview

This workflow implements a Document-based AI Chatbot leveraging Retrieval-Augmented Generation (RAG) by integrating OpenAI’s GPT-4 language model, Cohere’s reranker, and a Supabase vector database for knowledge storage and retrieval. It supports both real-time conversational AI interactions and document ingestion for knowledge base updating.

**Target Use Cases:**  
- Conversational AI agents that answer user queries based on an up-to-date knowledge base derived from PDF documents.  
- Dynamic knowledge base updates by loading, processing, embedding, and storing new documents.  
- Enhanced search relevance via AI reranking of retrieved documents.

**Logical Blocks:**  
- **1.1 Chat Input Reception:** Receiving user messages via a chat interface trigger.  
- **1.2 RAG Agent Processing:** Orchestrating conversation using language model, memory, and knowledge search tools.  
- **1.3 Knowledge Base Search:** Querying a vector-based document database with embeddings and reranking to find relevant information.  
- **1.4 Document Ingestion & Embedding:** Downloading PDF documents, extracting text, embedding content, and storing into vector database.  
- **1.5 Configuration & Notes:** Workflow variables and documentation notes for maintainability.

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Input Reception

- **Overview:** Receives incoming chat messages from users via a webhook-enabled chat interface node. Initiates the conversation flow.

- **Nodes Involved:**  
  - Chat Interface  
  - Note: Chat Trigger (sticky note)

- **Node Details:**  
  - **Chat Interface**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point for user chat messages, exposes a webhook for chat clients.  
    - Configuration: Default options, webhook ID assigned.  
    - Inputs: External user requests via webhook.  
    - Outputs: Passes message data to RAG Agent.  
    - Edge Cases: Webhook connectivity issues, malformed input data.  
    - Version: 1.1  

  - **Note: Chat Trigger**  
    - Type: Sticky Note  
    - Content: Highlights purpose of the chat trigger node.  
    - No inputs or outputs.

---

#### 1.2 RAG Agent Processing

- **Overview:** Central orchestrator that handles conversation context, memory, language model responses, and knowledge base tool integration.

- **Nodes Involved:**  
  - RAG Agent  
  - AI Model (OpenAI)  
  - Conversation Memory  
  - Knowledge Base Search  
  - Cohere Reranker  
  - Note: RAG Agent (sticky note)  
  - Note: Knowledge Search (sticky note)  
  - Note: Reranker (sticky note)

- **Node Details:**  

  - **RAG Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Manages the AI assistant logic, integrating language model, tools, and memory.  
    - Configuration: System message instructs assistant to search knowledge base before answering and cite sources.  
    - Inputs: Receives chat messages from Chat Interface; also receives outputs from AI Model, Knowledge Search, and Memory nodes.  
    - Outputs: Final assistant replies.  
    - Edge Cases: Failure in any connected tool or memory node, timeouts, or API quota limits.  
    - Version: 2  

  - **AI Model (OpenAI)**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Generates conversational responses using OpenAI GPT-4 model.  
    - Configuration: Model set to "gpt-4-mini" for balanced performance.  
    - Inputs: Receives prompts from RAG Agent.  
    - Outputs: Responses back to RAG Agent.  
    - Edge Cases: API key issues, rate limits, or generation errors.  
    - Version: 1.2  

  - **Conversation Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a sliding window of last 10 conversational turns to provide context.  
    - Configuration: `contextWindowLength` set to 10 messages.  
    - Inputs/Outputs: Connected to RAG Agent for reading/writing conversation history.  
    - Edge Cases: Memory overflow or corrupted context.  
    - Version: 1.3  

  - **Knowledge Base Search**  
    - Type: `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
    - Role: Searches Supabase vector database for relevant documents using embeddings.  
    - Configuration:  
      - Mode: "retrieve-as-tool" (used as a tool by RAG Agent)  
      - `topK`: 10 results returned  
      - `queryName` and `tableName` dynamically set by workflow variables  
      - `useReranker`: Enabled to improve search results  
      - Tool description instructs usage before answering questions  
    - Inputs/Outputs: Receives embeddings and reranker inputs; outputs search results to RAG Agent.  
    - Credentials: Supabase API key configured.  
    - Edge Cases: Supabase connectivity, authentication errors, empty search results.  
    - Version: 1.3  

  - **Cohere Reranker**  
    - Type: `@n8n/n8n-nodes-langchain.rerankerCohere`  
    - Role: Reranks search results from Knowledge Base Search for relevance.  
    - Configuration: Default parameters.  
    - Inputs: Receives search results from Knowledge Base Search.  
    - Outputs: Passes reranked results back to Knowledge Base Search node.  
    - Edge Cases: API key issues, reranking failures.  
    - Version: 1  

  - **Note: RAG Agent**  
    - Sticky note explaining the orchestrator role.  

  - **Note: Knowledge Search**  
    - Sticky note describing knowledge base search purpose.  

  - **Note: Reranker**  
    - Sticky note describing reranker role.

---

#### 1.3 Document Ingestion & Embedding

- **Overview:** Allows manual loading of PDF documents from Google Drive, extracts text, processes it for embeddings, and stores it in the Supabase vector database.

- **Nodes Involved:**  
  - Load Documents Trigger  
  - Download PDF from Drive  
  - Extract PDF Content  
  - Process Document Text  
  - Document Embeddings  
  - Store in Vector Database  
  - Note: Configuration (sticky note)

- **Node Details:**  

  - **Load Documents Trigger**  
    - Type: `n8n-nodes-base.manualTrigger`  
    - Role: Manual start node to initiate document loading and embedding flow.  
    - Configuration: No parameters; user must trigger manually.  
    - Outputs: Starts document download process.  
    - Edge Cases: User forgetting to trigger.  
    - Version: 1  

  - **Download PDF from Drive**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Role: Downloads PDF file from Google Drive based on URL passed via workflow variable `GOOGLE_DRIVE_FILE_URL`.  
    - Configuration:  
      - Operation: "download"  
      - `fileId`: Extracted from URL expression `{{GOOGLE_DRIVE_FILE_URL}}`.  
    - Inputs: Initiated by manual trigger.  
    - Outputs: Passes file binary data downstream.  
    - Edge Cases: File not found, permission denied, invalid URL.  
    - Version: 3  

  - **Extract PDF Content**  
    - Type: `n8n-nodes-base.extractFromFile`  
    - Role: Extracts text content from downloaded PDF file.  
    - Configuration:  
      - Operation: "pdf"  
      - Max pages: 500 (to handle large documents)  
    - Inputs: PDF binary data from Google Drive node.  
    - Outputs: Extracted text content for embedding.  
    - Edge Cases: Corrupted PDF, extraction failure.  
    - Version: 1  

  - **Process Document Text**  
    - Type: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
    - Role: Converts extracted text to a structured document format with metadata for embedding.  
    - Configuration:  
      - Metadata includes:  
        - `source`: filename from previous node  
        - `type`: "pdf"  
      - JSON data is expression-bound to extracted text content.  
    - Inputs: Extracted text JSON.  
    - Outputs: Formatted document object.  
    - Edge Cases: Missing or empty text data.  
    - Version: 1.1  

  - **Document Embeddings**  
    - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
    - Role: Generates vector embeddings of document content using OpenAI embeddings API.  
    - Configuration: Default embedding options.  
    - Inputs: Document from previous node.  
    - Outputs: Embeddings for storage.  
    - Edge Cases: API limits, empty documents.  
    - Version: 1.2  

  - **Store in Vector Database**  
    - Type: `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
    - Role: Inserts new document embeddings into Supabase vector database.  
    - Configuration:  
      - Mode: "insert"  
      - Uses workflow variables for `queryName` and `tableName`.  
    - Inputs: Receives embeddings and document metadata.  
    - Outputs: Confirmation of storage.  
    - Edge Cases: DB connection errors, duplicate entries.  
    - Version: 1.3  

  - **Note: Configuration**  
    - Sticky note listing workflow variables to replace:  
      - `GOOGLE_DRIVE_FILE_URL`  
      - `VECTOR_TABLE_NAME`  
      - `MATCH_FUNCTION_NAME`

---

### 3. Summary Table

| Node Name               | Node Type                                       | Functional Role                         | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                  |
|-------------------------|------------------------------------------------|---------------------------------------|------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| Chat Interface          | @n8n/n8n-nodes-langchain.chatTrigger           | Receives user chat messages            | (Webhook input)               | RAG Agent                    | ### 1️⃣ Chat Trigger<br>Receives messages from users through the chat interface                              |
| RAG Agent              | @n8n/n8n-nodes-langchain.agent                  | Orchestrates conversation, tools, memory | Chat Interface, AI Model, Memory, Knowledge Base Search | (Chat response output)     | ### 2️⃣ RAG Agent<br>Orchestrates the conversation using tools and memory                                   |
| AI Model (OpenAI)       | @n8n/n8n-nodes-langchain.lmChatOpenAi           | Generates AI responses                  | RAG Agent                    | RAG Agent                    |                                                                                                              |
| Conversation Memory     | @n8n/n8n-nodes-langchain.memoryBufferWindow     | Maintains chat context                  | RAG Agent                    | RAG Agent                    |                                                                                                              |
| Knowledge Base Search   | @n8n/n8n-nodes-langchain.vectorStoreSupabase    | Searches knowledge vector DB            | Search Embeddings, Cohere Reranker | RAG Agent                    | ### 3️⃣ Knowledge Search<br>Searches the vector database for relevant information                           |
| Cohere Reranker          | @n8n/n8n-nodes-langchain.rerankerCohere          | Reranks search results                  | Knowledge Base Search         | Knowledge Base Search         | ### 4️⃣ Reranker<br>Improves search quality by reordering results                                            |
| Search Embeddings       | @n8n/n8n-nodes-langchain.embeddingsOpenAi        | Generates embeddings for search queries |                              | Knowledge Base Search         |                                                                                                              |
| Load Documents Trigger  | n8n-nodes-base.manualTrigger                      | Manually triggers document ingestion   |                              | Download PDF from Drive       |                                                                                                              |
| Download PDF from Drive | n8n-nodes-base.googleDrive                         | Downloads PDF from Google Drive         | Load Documents Trigger        | Extract PDF Content           |                                                                                                              |
| Extract PDF Content     | n8n-nodes-base.extractFromFile                     | Extracts text from PDF                  | Download PDF from Drive       | Store in Vector Database      |                                                                                                              |
| Process Document Text   | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Prepares extracted text for embedding  | Extract PDF Content           | Store in Vector Database      |                                                                                                              |
| Document Embeddings     | @n8n/n8n-nodes-langchain.embeddingsOpenAi          | Creates embeddings from document text  | Process Document Text         | Store in Vector Database      |                                                                                                              |
| Store in Vector Database | @n8n/n8n-nodes-langchain.vectorStoreSupabase      | Inserts embeddings into vector DB       | Document Embeddings, Process Document Text, Extract PDF Content |                              |                                                                                                              |
| Note: Chat Trigger     | n8n-nodes-base.stickyNote                          | Documentation                          |                              |                              | ### 1️⃣ Chat Trigger<br>Receives messages from users through the chat interface                              |
| Note: RAG Agent        | n8n-nodes-base.stickyNote                          | Documentation                          |                              |                              | ### 2️⃣ RAG Agent<br>Orchestrates the conversation using tools and memory                                   |
| Note: Knowledge Search | n8n-nodes-base.stickyNote                          | Documentation                          |                              |                              | ### 3️⃣ Knowledge Search<br>Searches the vector database for relevant information                           |
| Note: Reranker         | n8n-nodes-base.stickyNote                          | Documentation                          |                              |                              | ### 4️⃣ Reranker<br>Improves search quality by reordering results                                            |
| Note: Configuration    | n8n-nodes-base.stickyNote                          | Documentation                          |                              |                              | ### Configuration Variables<br>Replace these in the workflow:<br>- GOOGLE_DRIVE_FILE_URL<br>- VECTOR_TABLE_NAME<br>- MATCH_FUNCTION_NAME |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Interface Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure default options.  
   - This node will expose a webhook to receive user messages.

2. **Create RAG Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set `systemMessage` to:  
     _"You are an intelligent assistant with access to a knowledge base. Always search for relevant information before answering questions. Be helpful, accurate, and cite your sources when providing information from the knowledge base."_  
   - Connect input from Chat Interface node.

3. **Create AI Model Node (OpenAI)**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set model to `"gpt-4-mini"` (or suitable GPT-4 variant).  
   - Connect output to RAG Agent’s AI Language Model input.

4. **Create Conversation Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set `contextWindowLength` to 10.  
   - Connect to RAG Agent’s AI Memory input.

5. **Create Knowledge Base Search Node**  
   - Type: `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
   - Set mode to `"retrieve-as-tool"`.  
   - Set `topK` to 10.  
   - Use workflow variables for `queryName` and `tableName` (e.g., `MATCH_FUNCTION_NAME`, `VECTOR_TABLE_NAME`).  
   - Enable `useReranker`.  
   - Add tool description: _"Use this tool to search for information in the knowledge base. Always use this before answering questions to ensure accurate, up-to-date responses."_  
   - Set credentials for Supabase API.  
   - Connect AI embedding input from Search Embeddings node (to be created next).  
   - Connect AI reranker input from Cohere Reranker node (to be created next).  
   - Connect AI tool output to RAG Agent’s AI Tool input.

6. **Create Cohere Reranker Node**  
   - Type: `@n8n/n8n-nodes-langchain.rerankerCohere`  
   - Use default config.  
   - Connect its output to Knowledge Base Search’s AI reranker input.

7. **Create Search Embeddings Node**  
   - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
   - Use default options.  
   - Connect output to Knowledge Base Search’s AI embedding input.

8. **Connect Chat Interface node’s main output to RAG Agent’s main input.**

9. **Create Document Ingestion Trigger**  
   - Type: `n8n-nodes-base.manualTrigger`  
   - This node manually starts the document ingestion flow.

10. **Create Download PDF from Drive Node**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Set operation to "download".  
    - Set `fileId` parameter to expression referencing a workflow variable `GOOGLE_DRIVE_FILE_URL`.  
    - Connect input from Load Documents Trigger node.

11. **Create Extract PDF Content Node**  
    - Type: `n8n-nodes-base.extractFromFile`  
    - Set operation to "pdf".  
    - Set max pages to 500.  
    - Connect input from Download PDF from Drive node.

12. **Create Process Document Text Node**  
    - Type: `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
    - Configure metadata with:  
      - `source`: expression referencing filename from Download PDF node  
      - `type`: "pdf"  
    - Set JSON data as expression binding extracted text.  
    - Connect input from Extract PDF Content node.

13. **Create Document Embeddings Node**  
    - Type: `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
    - Use default options.  
    - Connect input from Process Document Text node.

14. **Create Store in Vector Database Node**  
    - Type: `@n8n/n8n-nodes-langchain.vectorStoreSupabase`  
    - Set mode to "insert".  
    - Use workflow variables `MATCH_FUNCTION_NAME` and `VECTOR_TABLE_NAME` for options.  
    - Connect inputs from Document Embeddings, Process Document Text, and Extract PDF Content nodes as per original connections.

15. **Ensure all credentials are configured:**  
    - OpenAI API key for embeddings and language model nodes.  
    - Supabase API key for vector database nodes.  
    - Google Drive OAuth2 credentials for PDF download node.  
    - Cohere API key for reranker node.

16. **Add sticky notes for documentation:**  
    - Add notes for Chat Trigger, RAG Agent, Knowledge Search, Reranker, and Configuration Variables with the provided content to improve maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                  |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Replace workflow variables for Google Drive file URL, vector table name, and match function name.  | Workflow Configuration Variables                 |
| Workflow leverages RAG pattern combining vector search and LLM responses for accurate chatbot.     | Conceptual reference                              |
| Cohere Reranker improves retrieval quality by reordering search results based on relevance.        | https://cohere.ai/reranking                       |
| GPT-4-mini model variant used for balanced performance and cost efficiency.                         | OpenAI model selection                            |
| Supabase used as vector database with API keys securely stored in n8n credentials.                  | Supabase vector DB integration                     |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.