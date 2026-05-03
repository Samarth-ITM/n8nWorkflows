Build a Knowledge Base Chatbot with OpenAI, RAG and MongoDB Vector Embeddings

https://n8nworkflows.xyz/workflows/build-a-knowledge-base-chatbot-with-openai--rag-and-mongodb-vector-embeddings-4526


# Build a Knowledge Base Chatbot with OpenAI, RAG and MongoDB Vector Embeddings

---

### 1. Workflow Overview

This workflow builds a Knowledge Base Chatbot for internal support at a technology company using OpenAI, Retrieval-Augmented Generation (RAG), and MongoDB vector embeddings. The chatbot answers user queries by searching a vectorized index of official product documentation and generating AI-driven responses grounded strictly in that knowledge base.

The workflow includes two main logical blocks:

- **1.1 Knowledge Base Indexing Pipeline:**  
  This block imports product documentation from Google Docs, processes and chunks the text, generates vector embeddings using OpenAI, and inserts these embeddings into a MongoDB Atlas vector store. This indexing enables fast semantic search over product documents.

- **1.2 User Query Handling and AI Response Generation:**  
  This block listens for chat messages, retrieves relevant documents from MongoDB vector store based on embeddings similarity, maintains conversational memory, and uses a LangChain agent with OpenAI's GPT-4o-mini model to generate supported answers. It enforces strict behavior rules to ensure answers are based solely on the indexed official documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Knowledge Base Indexing Pipeline

**Overview:**  
This block imports official product documentation from a Google Docs document, segments the content into manageable chunks, generates vector embeddings for each chunk using OpenAI, and inserts the vectors into a MongoDB Atlas vector store to create an efficient searchable knowledge base.

**Nodes Involved:**  
- When clicking "Execute Workflow"  
- Google Docs Importer  
- Document Section Loader  
- Document Chunker  
- OpenAI Embeddings Generator  
- MongoDB Vector Store Inserter  
- Sticky Note (Run this workflow manually...)

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the indexing process.  
  - Configuration: None, simple manual trigger.  
  - Inputs: None  
  - Outputs: Trigger next node Google Docs Importer  
  - Failure Types: None (manual)  
  - Notes: Starts the importing and indexing pipeline.

- **Google Docs Importer**  
  - Type: Google Docs node  
  - Role: Fetch product documentation content from a Google Docs URL.  
  - Configuration: Operation "get" with a specific document URL pointing to the official product docs.  
  - Inputs: Trigger from manual node  
  - Outputs: Document content JSON to MongoDB Vector Store Inserter  
  - Credentials: Requires Google Docs OAuth2 credentials configured.  
  - Failure Types: OAuth errors, document access denied, network timeouts.

- **Document Section Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Loads specific document sections, passing metadata for document ID and content for further processing.  
  - Configuration: Metadata includes "doc_id" from JSON input, content passed as expression data.  
  - Inputs: Receives document content from Google Docs Importer output.  
  - Outputs: Passes document to Document Chunker and MongoDB Vector Store Inserter.  
  - Failure Types: Data parsing errors if input content malformed.

- **Document Chunker**  
  - Type: LangChain Text Splitter (Recursive Character splitter)  
  - Role: Splits large document content into chunks for efficient embedding.  
  - Configuration: Markdown split code, chunk size 3000 characters, chunk overlap 200 characters.  
  - Inputs: Document content from Document Section Loader.  
  - Outputs: Chunked text for embedding generation.  
  - Failure Types: Incorrect chunk sizing may cause incomplete splits or memory issues.

- **OpenAI Embeddings Generator (OpenAI Embeddings Generator node)**  
  - Type: LangChain OpenAI Embeddings node  
  - Role: Generates 1536-dimensional vector embeddings for document chunks using OpenAI API.  
  - Configuration: Default OpenAI embedding options, no special parameters.  
  - Inputs: Text chunks from Document Chunker.  
  - Outputs: Embeddings sent to MongoDB Vector Store Inserter.  
  - Credentials: Requires OpenAI API key.  
  - Failure Types: API rate limits, network errors, invalid text input.

- **MongoDB Vector Store Inserter**  
  - Type: LangChain MongoDB Vector Store node (Insert mode)  
  - Role: Inserts generated embeddings with metadata into MongoDB Atlas vector collection "n8n-template" and vector index "data_index".  
  - Configuration: Insert mode, target vector index and collection specified.  
  - Inputs: Embeddings and document data from OpenAI Embeddings Generator and Document Section Loader.  
  - Outputs: None (end of indexing pipeline)  
  - Credentials: MongoDB Atlas credentials configured.  
  - Failure Types: Connection timeouts, authentication errors, index misconfiguration.

- **Sticky Note**  
  - Role: Instructional note explaining that this workflow should be run manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search.  
  - Position: Visible near the manual trigger and indexing nodes.

---

#### 2.2 User Query Handling and AI Response Generation

**Overview:**  
This block activates upon receiving chat messages, retrieves relevant product documentation from the MongoDB vector store using semantic search, maintains conversational memory, and uses a LangChain agent powered by OpenAI GPT-4o-mini to generate accurate and concise answers grounded in the knowledge base.

**Nodes Involved:**  
- When chat message received  
- Knowledge Base Agent  
- MongoDB Vector Search  
- OpenAI Chat Model  
- Simple Memory  
- Embeddings OpenAI  
- Sticky Note1 (RAG explanation)  
- Sticky Note2 (Search index example)

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger (Webhook)  
  - Role: Listens for incoming chat messages from users to initiate the Q&A session.  
  - Configuration: Default options, webhook ID provided for receiving external requests.  
  - Inputs: External chat messages  
  - Outputs: Sends user query to Knowledge Base Agent  
  - Failure Types: Webhook connection issues, malformed requests.

- **Knowledge Base Agent**  
  - Type: LangChain Agent Node  
  - Role: Core AI assistant processing the user chat input, consulting MongoDB vector search tool for relevant docs, and generating responses.  
  - Configuration:  
    - Input text bound to `{{$json.chatInput}}`.  
    - System message defines behavior rules:  
      - Consult productDocs vector search first  
      - Respond clearly and directly  
      - Language detection per message  
      - No links provided  
      - Professional, human tone  
      - Concise answers (2-4 lines) unless more detail requested  
      - No invented info  
      - Numbered steps sequenced properly  
  - Inputs: User chat input, memory buffer, vector search tool, language model  
  - Outputs: Final AI-generated answer  
  - Failure Types: Expression evaluation errors, API failures, tool integration errors.

- **MongoDB Vector Search**  
  - Type: LangChain MongoDB Vector Store node (Retrieve mode)  
  - Role: Performs semantic search in MongoDB vector index "data_index" to find relevant document chunks as a tool named "productDocs."  
  - Configuration: Retrieve mode, tool name "productDocs" with description "retrieve documentation," configured to be used by agent.  
  - Inputs: Embeddings generated from user query (via Embeddings OpenAI)  
  - Outputs: Search results forwarded to Knowledge Base Agent as tool input  
  - Credentials: MongoDB Atlas connection required  
  - Failure Types: Query timeouts, no results found, auth errors.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model (GPT-4o-mini)  
  - Role: Language model generating natural language answers based on context and retrieved documents.  
  - Configuration: Model set to "gpt-4o-mini", no additional options configured.  
  - Inputs: Text prompt from Knowledge Base Agent  
  - Outputs: Generated chat response  
  - Credentials: OpenAI API key required  
  - Failure Types: API rate limits, network issues.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation history context window to support multi-turn dialogue.  
  - Configuration: Default settings, no window size specified explicitly.  
  - Inputs: Conversation exchanges from Knowledge Base Agent  
  - Outputs: Memory data back to Knowledge Base Agent  
  - Failure Types: Memory overflow or truncation if conversation too long.

- **Embeddings OpenAI**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates vector embeddings for user chat input to facilitate semantic retrieval.  
  - Configuration: Default embedding generation parameters.  
  - Inputs: User chat input text  
  - Outputs: Embeddings sent to MongoDB Vector Search  
  - Credentials: OpenAI API key required  
  - Failure Types: API failures, malformed input.

- **Sticky Note1**  
  - Role: Describes the workflow’s use of RAG technique for answering user questions by searching the MongoDB vector store and generating AI responses with context.  
  - Position: Near user chat and agent nodes.

- **Sticky Note2**  
  - Role: Provides example schema for the MongoDB search index, detailing fields (_id, text, embedding vector, source, doc_id) and vector dimensions (1536) using cosine similarity.  
  - Position: Near MongoDB vector nodes.

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                               | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                                                      |
|---------------------------|---------------------------------------------|-----------------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                             | Starts the manual import and indexing process | None                         | Google Docs Importer          | Run this workflow manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search.                          |
| Google Docs Importer       | Google Docs node                            | Imports product documentation content         | When clicking "Execute Workflow" | MongoDB Vector Store Inserter |                                                                                                                                                                |
| Document Section Loader    | LangChain Document Default Data Loader     | Loads document content and metadata            | Google Docs Importer          | Document Chunker, MongoDB Vector Store Inserter |                                                                                                                                                                |
| Document Chunker          | LangChain Text Splitter                      | Splits document into chunks for embedding      | Document Section Loader       | OpenAI Embeddings Generator   |                                                                                                                                                                |
| OpenAI Embeddings Generator | LangChain OpenAI Embeddings                  | Generates vector embeddings for document chunks | Document Chunker              | MongoDB Vector Store Inserter |                                                                                                                                                                |
| MongoDB Vector Store Inserter | LangChain MongoDB Vector Store (Insert)     | Inserts embeddings into MongoDB vector index   | Google Docs Importer, Document Section Loader, OpenAI Embeddings Generator | None                         |                                                                                                                                                                |
| When chat message received | LangChain Chat Trigger                      | Listens for user chat messages                  | External                     | Knowledge Base Agent          | This workflow uses retrieval-augmented generation (RAG) to answer user questions by searching the MongoDB vector store and generating AI responses with context. |
| Knowledge Base Agent       | LangChain Agent                             | Processes chat input, uses vector search and AI to generate answers | When chat message received, Simple Memory, MongoDB Vector Search, OpenAI Chat Model | None                         |                                                                                                                                                                |
| MongoDB Vector Search      | LangChain MongoDB Vector Store (Retrieve)  | Searches knowledge base vector store for relevant docs | Embeddings OpenAI            | Knowledge Base Agent          | Search Index Example: { "mappings": { "dynamic": false, "fields": { "_id": { "type": "string" }, "text": { "type": "string" }, "embedding": { "type": "knnVector", "dimensions": 1536, "similarity": "cosine" }, "source": { "type": "string" }, "doc_id": { "type": "string" } } } } |
| OpenAI Chat Model          | LangChain OpenAI Chat Model (GPT-4o-mini)  | Generates AI natural language responses        | Knowledge Base Agent          | Knowledge Base Agent          |                                                                                                                                                                |
| Simple Memory              | LangChain Memory Buffer Window              | Maintains conversation context                  | Knowledge Base Agent          | Knowledge Base Agent          |                                                                                                                                                                |
| Embeddings OpenAI          | LangChain OpenAI Embeddings                  | Generates vector embeddings for user queries   | When chat message received    | MongoDB Vector Search         |                                                                                                                                                                |
| Sticky Note                | Sticky Note                                 | Instructional note on manual import/indexing   | None                         | None                         | Run this workflow manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search.                          |
| Sticky Note1               | Sticky Note                                 | RAG workflow explanation                        | None                         | None                         | This workflow uses retrieval-augmented generation (RAG) to answer user questions by searching the MongoDB vector store and generating AI responses with context. |
| Sticky Note2               | Sticky Note                                 | MongoDB vector index schema example             | None                         | None                         | Search Index Example: { "mappings": { "dynamic": false, "fields": { "_id": { "type": "string" }, "text": { "type": "string" }, "embedding": { "type": "knnVector", "dimensions": 1536, "similarity": "cosine" }, "source": { "type": "string" }, "doc_id": { "type": "string" } } } } |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the indexing workflow manually.

2. **Create Google Docs Importer Node**  
   - Type: Google Docs  
   - Operation: Get  
   - Document URL: Paste the official product documentation Google Docs URL (e.g., `https://docs.google.com/document/d/1gvgp71e9edob8WLqFIYCdzC7kUq3pLO37VKb-a-vVW4/edit`)  
   - Credential: Configure Google Docs OAuth2 API credentials.

3. **Create Document Section Loader Node**  
   - Type: LangChain Document Default Data Loader  
   - Parameters:  
     - Metadata: Add metadata field "doc_id" with expression `{{$json.documentId}}`  
     - JSON Data: Expression mode with `{{$json.content}}`  
   - Connect input from Google Docs Importer.

4. **Create Document Chunker Node**  
   - Type: LangChain Recursive Character Text Splitter  
   - Parameters:  
     - Split code: markdown  
     - Chunk size: 3000 characters  
     - Chunk overlap: 200 characters  
   - Connect input from Document Section Loader.

5. **Create OpenAI Embeddings Generator Node**  
   - Type: LangChain OpenAI Embeddings  
   - Credentials: Configure OpenAI API key  
   - Connect input from Document Chunker.

6. **Create MongoDB Vector Store Inserter Node**  
   - Type: LangChain MongoDB Vector Store (Insert mode)  
   - Parameters:  
     - Mongo Collection: Select or enter your MongoDB collection name (e.g., "n8n-template")  
     - Vector Index Name: "data_index"  
   - Credentials: Configure MongoDB Atlas credentials  
   - Connect inputs from Google Docs Importer, Document Section Loader, and OpenAI Embeddings Generator.

7. **Connect Manual Trigger Node output to Google Docs Importer input**.

---

8. **Create LangChain Chat Trigger Node**  
   - Type: LangChain Chat Trigger (Webhook)  
   - Configure webhook ID and URL to receive chat messages externally.

9. **Create Embeddings OpenAI Node**  
   - Type: LangChain OpenAI Embeddings  
   - Credentials: Same OpenAI API key  
   - Connect input from Chat Trigger (user message text).

10. **Create MongoDB Vector Search Node**  
    - Type: LangChain MongoDB Vector Store (Retrieve mode)  
    - Parameters:  
      - Mode: Retrieve as tool  
      - Tool Name: "productDocs"  
      - Tool Description: "retrieve documentation"  
      - Mongo Collection: Same as indexing ("n8n-template")  
      - Vector Index Name: "data_index"  
    - Credentials: MongoDB Atlas  
    - Connect input from Embeddings OpenAI node.

11. **Create OpenAI Chat Model Node**  
    - Type: LangChain OpenAI Chat Model  
    - Model: "gpt-4o-mini"  
    - Credentials: OpenAI API key.

12. **Create Simple Memory Node**  
    - Type: LangChain Memory Buffer Window  
    - Default settings.

13. **Create Knowledge Base Agent Node**  
    - Type: LangChain Agent  
    - Parameters:  
      - Text input: Expression `{{$json.chatInput}}`  
      - Options system message: Paste detailed system message defining agent behavior (see Section 2.2 Knowledge Base Agent)  
      - Prompt type: Define  
    - Connect inputs:  
      - ai_languageModel: OpenAI Chat Model  
      - ai_memory: Simple Memory  
      - ai_tool: MongoDB Vector Search  
      - ai_embedding: Embeddings OpenAI  
    - Connect output to end or response handler.

14. **Connect Chat Trigger output to Knowledge Base Agent input**.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Run this workflow manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search.       | Sticky Note near manual trigger and indexing nodes                                                                                                                          |
| This workflow uses retrieval-augmented generation (RAG) to answer user questions by searching the MongoDB vector store and generating AI responses with context. | Sticky Note near user query and agent nodes                                                                                                                                  |
| MongoDB Vector Search Index Example: schema includes fields `_id`, `text`, `embedding` (1536 dimensions, cosine similarity), `source`, and `doc_id`. | Sticky Note near MongoDB vector nodes. Useful for configuring the MongoDB Atlas vector index properly.                                                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---