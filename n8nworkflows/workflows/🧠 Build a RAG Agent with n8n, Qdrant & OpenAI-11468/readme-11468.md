🧠 Build a RAG Agent with n8n, Qdrant & OpenAI

https://n8nworkflows.xyz/workflows/---build-a-rag-agent-with-n8n--qdrant---openai-11468


# 🧠 Build a RAG Agent with n8n, Qdrant & OpenAI

---

### 1. Workflow Overview

This workflow builds a Retrieval-Augmented Generation (RAG) Agent using n8n, integrating Google Drive, Qdrant vector database, and OpenAI language models. It automates the ingestion of documents from a Google Drive folder, transforms them into searchable embeddings stored in Qdrant, and provides a chat interface for users to ask questions answered based on the uploaded documents.

The workflow is logically divided into two main blocks:

- **1.1 Data Pipeline:** Detects new files in a specific Google Drive folder, downloads them as Markdown, adds metadata, splits content into chunks, generates embeddings via OpenAI, and inserts these into the Qdrant vector store.

- **1.2 RAG Agent Chat Interface:** Listens to user queries via chat, retrieves relevant document chunks from Qdrant using semantic search, manages conversation memory, and generates context-aware responses using GPT-4o.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Pipeline

**Overview:**  
This block handles file detection, content retrieval, processing, and storage into the vector database. It continuously monitors a specified Google Drive folder for new files, converts them to Markdown, enriches them with metadata, splits the text to manageable chunks, generates semantic embeddings, and inserts them into the Qdrant vector store for later retrieval.

**Nodes Involved:**  
- Detect New Files (Google Drive Trigger)  
- Download as Markdown (HTTP Request)  
- Add Metadata (Merge)  
- Load File Content & Metadata (Document Loader)  
- Split into Chunks (Recursive Character Text Splitter)  
- Generate Embeddings (OpenAI Embeddings)  
- Insert into Vector Store (Qdrant Insert)  

**Node Details:**

- **Detect New Files**  
  - *Type:* Google Drive Trigger node  
  - *Role:* Watches a specific Google Drive folder (`folderToWatch` set to a dedicated folder ID) for any newly created files.  
  - *Configuration:* Polls every minute, triggers on any new file created, accepts all file types.  
  - *Credentials:* Google Drive OAuth2 configured.  
  - *Input:* External trigger (Google Drive event).  
  - *Output:* File metadata including export links.  
  - *Potential Failures:* Authentication issues, folder access permissions, Google API rate limits.

- **Download as Markdown**  
  - *Type:* HTTP Request node  
  - *Role:* Downloads the newly detected file as Markdown using the file’s export link (`$json.exportLinks['text/markdown']`).  
  - *Configuration:* Uses Google Drive OAuth2 credentials for authorization.  
  - *Input:* File metadata from the trigger.  
  - *Output:* Raw Markdown content of the file.  
  - *Potential Failures:* Invalid or missing export links, HTTP errors, auth token expiration.

- **Add Metadata**  
  - *Type:* Merge node  
  - *Role:* Combines file metadata (such as file name, id, created time, size, owner email) with the downloaded Markdown content.  
  - *Configuration:* Combines inputs by position (i.e., aligns metadata and content by item index).  
  - *Input:* Output from Download as Markdown and Detect New Files nodes.  
  - *Output:* Composite JSON containing content plus metadata.  
  - *Potential Failures:* Mismatch in item counts, missing metadata fields.

- **Load File Content & Metadata**  
  - *Type:* Document Default Data Loader (LangChain)  
  - *Role:* Prepares documents for embedding generation by packaging text and metadata into a standardized document format.  
  - *Configuration:* Custom text splitting mode disabled here (handled later); metadata fields assigned from merged JSON.  
  - *Input:* Combined content and metadata JSON.  
  - *Output:* Structured document object with text and metadata.  
  - *Potential Failures:* Incorrect metadata mapping, unsupported content formats.

- **Split into Chunks**  
  - *Type:* Recursive Character Text Splitter (LangChain)  
  - *Role:* Splits document text into chunks of 1500 characters with 300 characters overlap to maintain context between chunks.  
  - *Configuration:* Default options, chunk size 1500, chunk overlap 300.  
  - *Input:* Document data from the loader node.  
  - *Output:* Array of text chunks for embedding.  
  - *Potential Failures:* Text encoding issues, unexpected text formats.

- **Generate Embeddings**  
  - *Type:* OpenAI Embeddings (LangChain)  
  - *Role:* Calls OpenAI API to generate vector embeddings for each text chunk.  
  - *Configuration:* Uses OpenAI API credentials; no additional parameters set.  
  - *Input:* Text chunks.  
  - *Output:* Embedding vectors for each chunk.  
  - *Potential Failures:* API rate limits, invalid API keys, network timeouts.

- **Insert into Vector Store**  
  - *Type:* Qdrant Vector Store node (LangChain)  
  - *Role:* Inserts document chunk embeddings with metadata into the Qdrant collection `n8n-rag-tutorial`.  
  - *Configuration:* Batch insert with batch size 50; collection specified by ID.  
  - *Input:* Embeddings with associated metadata.  
  - *Output:* Confirmation of insertion.  
  - *Credentials:* Qdrant API key required.  
  - *Potential Failures:* Qdrant API errors, collection not found, authentication failures.

---

#### 2.2 RAG Agent Chat Interface

**Overview:**  
This block enables interactive question answering by users via a chat interface. It receives user queries, searches the vector database for relevant document chunks, manages conversational context, and generates precise answers grounded in the documents using GPT-4o.

**Nodes Involved:**  
- Chat with a RAG Agent (Chat Trigger)  
- Answer Questions (LangChain Agent)  
- Search Documents (Qdrant Retrieve Tool)  
- Store Conversation (Memory Buffer Window)  
- Generate Response (OpenAI Chat LM)

**Node Details:**

- **Chat with a RAG Agent**  
  - *Type:* Chat Trigger node  
  - *Role:* Acts as the webhook entry point for incoming user chat messages.  
  - *Configuration:* Webhook ID assigned; waits for user input to start processing.  
  - *Input:* Incoming chat messages from external clients.  
  - *Output:* Passes user messages downstream for processing.  
  - *Potential Failures:* Webhook connectivity, payload parsing errors.

- **Answer Questions**  
  - *Type:* LangChain Agent node  
  - *Role:* Core RAG agent logic combining document search, conversation memory, and language model response generation.  
  - *Configuration:*  
    - Provides a detailed system prompt instructing the agent to always search documents first before answering, strictly use document content, cite sources, and handle ambiguous questions carefully.  
    - Enforces strict guidelines to prevent hallucinations and ensure factual responses based on document knowledge base.  
  - *Input:* User query from chat trigger; relevant documents from Search Documents; conversation history from Store Conversation; language model output from Generate Response.  
  - *Output:* Final textual answer to user query.  
  - *Potential Failures:* Expression evaluation errors in prompt, incomplete context, language model API issues.

- **Search Documents**  
  - *Type:* Qdrant Vector Store (Retrieve as Tool)  
  - *Role:* Performs semantic similarity search on the Qdrant collection to find top 10 relevant document chunks matching user query.  
  - *Configuration:* Retrieval mode with topK=10, collection set to `n8n-rag-tutorial`.  
  - *Input:* User question embedding or text query.  
  - *Output:* List of document chunks as context for the agent.  
  - *Credentials:* Qdrant API required.  
  - *Potential Failures:* Search API errors, empty search results.

- **Store Conversation**  
  - *Type:* Memory Buffer Window (LangChain)  
  - *Role:* Maintains a conversational context window up to 20 messages to preserve dialogue continuity.  
  - *Configuration:* Context window length set to 20 messages.  
  - *Input:* User messages and agent responses.  
  - *Output:* Conversation memory passed to agent for context.  
  - *Potential Failures:* Memory overflow, data synchronization issues.

- **Generate Response**  
  - *Type:* OpenAI Chat LM (LangChain)  
  - *Role:* Generates natural language answers based on the system prompt and retrieved document context using the GPT-4o model.  
  - *Configuration:* Model set to GPT-4o, temperature 0.4 for balanced creativity and accuracy.  
  - *Input:* System prompt, user query, and retrieved document context.  
  - *Output:* Textual response to user question.  
  - *Credentials:* OpenAI API key required.  
  - *Potential Failures:* Model unavailability, API limits, slow response times.

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                      | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                     |
|-------------------------|----------------------------------------|------------------------------------|------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------|
| Detect New Files        | Google Drive Trigger                    | Watches folder for new files       | -                      | Download as Markdown, Add Metadata | # Build a RAG Agent with n8n, Qdrant & OpenAI<br>Workflow overview, setup instructions, credentials required.   |
| Download as Markdown    | HTTP Request                           | Downloads file as Markdown          | Detect New Files        | Add Metadata              |                                                                                                                 |
| Add Metadata            | Merge                                  | Combines file metadata and content | Detect New Files, Download as Markdown | Insert into Vector Store | ## 1. Import your data and store it in a vector database<br>Upload files to Google Drive → convert to Markdown → chunk text → create embeddings → store in Qdrant vector store. |
| Load File Content & Metadata | Document Default Data Loader (LangChain) | Prepares document with metadata      | Split into Chunks       | Insert into Vector Store  |                                                                                                                 |
| Split into Chunks       | Recursive Character Text Splitter (LangChain) | Splits text into manageable chunks | Load File Content & Metadata | Load File Content & Metadata |                                                                                                                 |
| Generate Embeddings     | OpenAI Embeddings (LangChain)          | Creates embeddings for chunks      | Split into Chunks       | Insert into Vector Store, Search Documents |                                                                                                                 |
| Insert into Vector Store| Qdrant Vector Store Insert (LangChain) | Stores embeddings & metadata       | Add Metadata, Generate Embeddings, Load File Content & Metadata | -                         |                                                                                                                 |
| Chat with a RAG Agent   | Chat Trigger                          | Receives user chat queries         | -                      | Answer Questions          | ## 2. Chat with your data<br>Ask the RAG Agent → retrieve chunks from Qdrant → generate contextual responses.    |
| Answer Questions        | LangChain Agent                        | Core RAG agent logic               | Chat with a RAG Agent, Search Documents, Store Conversation, Generate Response | -                         |                                                                                                                 |
| Search Documents        | Qdrant Vector Store Retrieve Tool (LangChain) | Finds relevant document chunks     | Generate Embeddings     | Answer Questions          |                                                                                                                 |
| Store Conversation      | Memory Buffer Window (LangChain)       | Maintains conversation context    | -                      | Answer Questions          |                                                                                                                 |
| Generate Response       | OpenAI Chat LM (LangChain)              | Generates answer text              | -                      | Answer Questions          |                                                                                                                 |
| Sticky Note             | Sticky Note                            | Documentation / commentary        | -                      | -                         | ## 1. Import your data and store it in a vector database - See above.                                           |
| Sticky Note2            | Sticky Note                            | Documentation / commentary        | -                      | -                         | ## 2. Chat with your data - See above.                                                                          |
| Sticky Note1            | Sticky Note                            | Documentation / commentary        | -                      | -                         | # Build a RAG Agent with n8n, Qdrant & OpenAI - Full workflow explanation and setup instructions.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Set to watch a specific folder (use folder ID `1BevhU5qdgNDFbK4D9oAYGeK0Dt5sEaxQ`)  
   - Trigger on event: fileCreated  
   - Poll every minute  
   - Set Google Drive OAuth2 credentials  

2. **Create HTTP Request Node (Download as Markdown)**  
   - Type: HTTP Request  
   - URL expression: `{{$json.exportLinks['text/markdown']}}`  
   - Authentication: Use Google Drive OAuth2 credentials  
   - Connect input from Google Drive Trigger  

3. **Create Merge Node (Add Metadata)**  
   - Type: Merge  
   - Mode: Combine by position  
   - Connect inputs from Google Drive Trigger and HTTP Request nodes  

4. **Create Document Loader Node (Load File Content & Metadata)**  
   - Type: Document Default Data Loader (LangChain)  
   - Set metadata fields: file_name, file_id, created_time, size, owner  
   - Map metadata values from merged JSON  
   - Set `jsonData` to the Markdown content field in merged JSON  
   - Use expression mode for input data  

5. **Create Text Splitter Node (Split into Chunks)**  
   - Type: Recursive Character Text Splitter (LangChain)  
   - Set chunk size to 1500 characters  
   - Set chunk overlap to 300 characters  
   - Connect input from Document Loader Node  

6. **Create Embeddings Node (Generate Embeddings)**  
   - Type: OpenAI Embeddings (LangChain)  
   - Set OpenAI API credentials  
   - Connect input from Text Splitter Node  

7. **Create Vector Store Node (Insert into Vector Store)**  
   - Type: Qdrant Vector Store Insert (LangChain)  
   - Set mode to insert  
   - Use collection ID `n8n-rag-tutorial`  
   - Set batch size to 50  
   - Provide Qdrant API credentials  
   - Connect inputs from Add Metadata Node, Generate Embeddings Node, and Load File Content & Metadata Node as per workflow  

8. **Create Chat Trigger Node (Chat with a RAG Agent)**  
   - Type: Chat Trigger  
   - Assign a webhook ID  
   - This node is the entry point for user questions  

9. **Create LangChain Agent Node (Answer Questions)**  
   - Type: LangChain Agent  
   - Paste the detailed system prompt instructing strict document-based answering, citation, and handling ambiguous queries  
   - Set version 2.2 or compatible  
   - Connect input from Chat Trigger Node  

10. **Create Qdrant Vector Store Node (Search Documents)**  
    - Type: Qdrant Vector Store Retrieve as Tool  
    - Set mode to retrieve-as-tool  
    - Set topK to 10  
    - Use collection `n8n-rag-tutorial`  
    - Provide Qdrant API credentials  
    - Connect output to LangChain Agent’s search input  

11. **Create Memory Buffer Node (Store Conversation)**  
    - Type: Memory Buffer Window (LangChain)  
    - Set context window length to 20  
    - Connect output to LangChain Agent’s memory input  

12. **Create OpenAI Chat LM Node (Generate Response)**  
    - Type: OpenAI Chat LM  
    - Set model to GPT-4o  
    - Set temperature to 0.4  
    - Provide OpenAI API credentials  
    - Connect output to LangChain Agent’s language model input  

13. **Connect LangChain Agent Node output back to Chat Trigger response**  
    - This completes the chat response cycle  

14. **Add Sticky Notes (Optional)**  
    - Add descriptive sticky notes summarizing the data pipeline, chat interface, and general workflow instructions for maintainability and clarity  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Workflow designed to create a RAG agent answering queries from uploaded documents integrating n8n, Qdrant, and OpenAI.                   | Workflow purpose summary.                                                                                              |
| Google Drive folder monitored: https://drive.google.com/drive/u/2/folders/1BevhU5qdgNDFbK4D9oAYGeK0Dt5sEaxQ                              | Folder ID used for the trigger node.                                                                                   |
| Qdrant vector database documentation: https://qdrant.tech/                                                                               | Reference for Qdrant API and collection setup.                                                                          |
| OpenAI API platform: https://platform.openai.com/api-keys                                                                               | For API key creation and management.                                                                                   |
| System prompt emphasizes strict retrieval from documents and citation to prevent hallucination, critical for legal and business domains. | Best practice note for configuring LLM behavior in RAG setups.                                                         |
| The workflow uses GPT-4o model, which is optimized for chat and reasoning tasks.                                                        | Model selection rationale for generating contextual AI responses.                                                      |

---

**Disclaimer:** The text provided originates solely from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.

---