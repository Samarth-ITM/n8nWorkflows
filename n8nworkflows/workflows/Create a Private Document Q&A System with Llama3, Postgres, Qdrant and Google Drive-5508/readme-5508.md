Create a Private Document Q&A System with Llama3, Postgres, Qdrant and Google Drive

https://n8nworkflows.xyz/workflows/create-a-private-document-q-a-system-with-llama3--postgres--qdrant-and-google-drive-5508


# Create a Private Document Q&A System with Llama3, Postgres, Qdrant and Google Drive

### 1. Workflow Overview

This workflow implements a **Private Document Question & Answer (Q&A) System** using Llama3 language models, Postgres for chat memory, Qdrant as a vector store, and Google Drive as the document source. It is designed to maintain an AI assistant that can answer questions based on documents stored privately in Google Drive. The workflow involves two main logical sections:

- **1.1 Document Ingestion & Vectorization:**  
  This block monitors specific Google Drive folders for new or updated documents, downloads and extracts text from them, splits the text into chunks, generates embeddings using Ollama’s Llama3 embedding model, and inserts these vectors into a Qdrant vector store. It also tags documents with metadata to maintain traceability.

- **1.2 Chat Interaction & AI Agent Processing:**  
  This block listens for incoming chat messages (queries), maintains conversational memory in Postgres, uses the Ollama Llama3 chat model as the language model, queries the Qdrant vector store for relevant context, and returns AI-generated answers. The AI Agent node orchestrates these components to produce context-aware responses.

The workflow is structured to enable a seamless update of the knowledge base from Google Drive documents and provide an interactive Q&A interface powered by vector search and LLM.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion & Vectorization

- **Overview:**  
  This block watches Google Drive folders for newly created or updated files, downloads and converts them to plain text, processes the text into manageable chunks, creates vector embeddings, and inserts them into the Qdrant vector store with metadata.

- **Nodes Involved:**  
  File Created, File Updated, Set File ID, Download File, Extract Document Text, Recursive Character Text Splitter, Default Data Loader, Embeddings Ollama1, Qdrant Vector Store Insert

- **Node Details:**

  - **File Created**  
    - Type: Google Drive Trigger  
    - Role: Watches a specific Google Drive folder for new files created (polls every minute).  
    - Configuration: Watches folder ID `1uf6zZN51rgAuQgid4-Oi314f6mJIQdiB`.  
    - Credentials: Google Drive OAuth2.  
    - Input/Output: Triggers on file creation event → outputs file metadata.  
    - Edge Cases: Permissions errors, API rate limits, folder ID invalid.

  - **File Updated**  
    - Type: Google Drive Trigger  
    - Role: Watches a different Google Drive folder for file updates (polls every minute).  
    - Configuration: Watches folder ID `1914m3M7kRzkd5RJqAfzRY9EBcJrKemZC`.  
    - Credentials: Same as above.  
    - Edge Cases: Similar to File Created.

  - **Set File ID**  
    - Type: Set  
    - Role: Extracts and stores file ID and parent folder ID from trigger data for downstream use.  
    - Key Variables: `file_id = {{$json.id}}`, `folder_id = {{$json.parents[0]}}`  
    - Input: File Created or File Updated outputs.  
    - Output: Passes file_id and folder_id for next nodes.  
    - Edge Cases: Missing parents array, empty file ID.

  - **Download File**  
    - Type: Google Drive  
    - Role: Downloads the file content using the extracted file ID, converts Google Docs to plain text.  
    - Configuration: Uses Google Drive API with file conversion `docsToFormat: text/plain`.  
    - Credentials: Google Drive OAuth2.  
    - Input: file_id from Set File ID node.  
    - Edge Cases: Large files, conversion failures, permission denied.

  - **Extract Document Text**  
    - Type: Extract From File  
    - Role: Extracts raw text content from the downloaded file blob.  
    - Configuration: Operation set to extract text.  
    - Input: Download File output (binary).  
    - Edge Cases: Unsupported file formats, empty files.

  - **Recursive Character Text Splitter**  
    - Type: Text Splitter  
    - Role: Splits extracted text into chunks (default chunk size 100 characters) for embedding.  
    - Configuration: Recursive splitting, chunk size 100.  
    - Input: Extracted text.  
    - Edge Cases: Very short texts, texts with no delimiters.

  - **Default Data Loader**  
    - Type: Document Default Data Loader  
    - Role: Prepares document chunks by attaching metadata (file_id, folder_id) for vector insertion.  
    - Configuration: Metadata assigned from `Set File ID` node variables.  
    - Input: Text chunks from splitter.  
    - Edge Cases: Missing metadata, malformed chunks.

  - **Embeddings Ollama1**  
    - Type: Embeddings (Ollama)  
    - Role: Generates vector embeddings for document chunks using model `llama3.2:latest`.  
    - Credentials: Ollama API account.  
    - Input: Document chunks with metadata.  
    - Edge Cases: API errors, rate limits, invalid model.

  - **Qdrant Vector Store Insert**  
    - Type: Vector Store (Qdrant)  
    - Role: Inserts generated embeddings into the Qdrant collection `midjourney`.  
    - Credentials: Qdrant API account.  
    - Configuration: Mode set to `insert`.  
    - Input: Embeddings from Ollama.  
    - Edge Cases: Connection failures, collection not found, quota exceeded.

---

#### 2.2 Chat Interaction & AI Agent Processing

- **Overview:**  
  This block handles incoming chat messages, maintains memory in Postgres, leverages Ollama Llama3 chat model, and integrates vector store queries to answer questions based on the knowledge base.

- **Nodes Involved:**  
  When chat message received, AI Agent, Ollama Chat Model, Postgres Chat Memory, Qdrant Vector Store, Ollama Model, Answer questions with a vector store

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point webhook that triggers on incoming chat messages.  
    - Configuration: Default options.  
    - Output: Chat message input.  
    - Edge Cases: Webhook misconfiguration, payload format errors.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Central orchestrator that processes chat messages, accesses memory, language models, and tools.  
    - Configuration: System message set to `"You are a helpful assistant you have access to a knowledge base"`.  
    - Input: Chat messages, memory, language model, vector tool outputs.  
    - Output: AI-generated chat responses.  
    - Edge Cases: Expression evaluation errors, memory retrieval failures, API timeouts.

  - **Ollama Chat Model**  
    - Type: LangChain Chat Model (Ollama)  
    - Role: Provides chat-based LLM responses using Llama3.2 model.  
    - Credentials: Ollama API account.  
    - Input: Chat messages from trigger or AI Agent.  
    - Output: Language model responses.  
    - Edge Cases: API errors, model unavailable.

  - **Postgres Chat Memory**  
    - Type: LangChain Memory (Postgres)  
    - Role: Stores and retrieves previous chat messages to maintain conversational context.  
    - Credentials: Postgres account.  
    - Input: Chat messages and AI responses.  
    - Output: Memory context for AI Agent.  
    - Edge Cases: Database connectivity issues, query failures.

  - **Qdrant Vector Store**  
    - Type: LangChain Vector Store (Qdrant)  
    - Role: Searches vector embeddings for relevant documents to support question answering.  
    - Credentials: Qdrant API account.  
    - Configuration: Uses collection `midjourney`.  
    - Input: Query embeddings generated by AI Agent/Ollama Model.  
    - Output: Retrieved relevant documents or text chunks.  
    - Edge Cases: Search failures, empty results.

  - **Ollama Model**  
    - Type: LangChain Language Model (Ollama)  
    - Role: Generates embeddings or language model outputs used by the vector store tool.  
    - Credentials: Ollama API account.  
    - Input: Query text or chat context.  
    - Output: Embeddings or model responses.  
    - Edge Cases: API errors.

  - **Answer questions with a vector store**  
    - Type: LangChain Tool (Vector Store)  
    - Role: Tool used by AI Agent to query the vector store and retrieve relevant knowledge to answer questions.  
    - Description: `"this tool will be used to retrieve knowledge"`.  
    - Input: Queries from AI Agent.  
    - Output: Retrieved document snippets for inclusion in the answer.  
    - Edge Cases: No relevant results found, tool invocation errors.

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                         | Input Node(s)                    | Output Node(s)                   | Sticky Note                                           |
|-------------------------------|----------------------------------------|---------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------|
| When chat message received     | LangChain Chat Trigger                  | Entry point for chat queries           | -                               | AI Agent                        |                                                       |
| AI Agent                      | LangChain Agent                        | Orchestrates chat processing           | When chat message received, Postgres Chat Memory, Ollama Chat Model, Answer questions with a vector store | -                               |                                                       |
| Ollama Chat Model             | LangChain Chat Model (Ollama)          | Provides chat LLM responses            | When chat message received       | AI Agent                        |                                                       |
| Postgres Chat Memory          | LangChain Memory (Postgres)            | Stores chat memory                     | When chat message received       | AI Agent                        |                                                       |
| Qdrant Vector Store           | LangChain Vector Store (Qdrant)        | Searches vector embeddings             | Embeddings Ollama, Ollama Model | Answer questions with a vector store |                                                       |
| Ollama Model                 | LangChain Language Model (Ollama)      | Generates embeddings or LLM outputs    | -                               | Answer questions with a vector store |                                                       |
| Answer questions with a vector store | LangChain Tool (Vector Store)          | Retrieves knowledge via vector search  | Qdrant Vector Store, Ollama Model | AI Agent                        |                                                       |
| File Created                 | Google Drive Trigger                    | Triggers on new files in Drive folder  | -                               | Set File ID                     |                                                       |
| File Updated                 | Google Drive Trigger                    | Triggers on updated files in Drive folder | -                               | Set File ID                     |                                                       |
| Set File ID                  | Set                                    | Extracts file and folder IDs           | File Created, File Updated       | Download File                   |                                                       |
| Download File                | Google Drive                           | Downloads and converts files to text   | Set File ID                     | Extract Document Text           |                                                       |
| Extract Document Text        | Extract From File                      | Extracts text content from files       | Download File                   | Qdrant Vector Store Insert      |                                                       |
| Recursive Character Text Splitter | Text Splitter                        | Splits text into chunks for embedding  | Extract Document Text           | Default Data Loader             |                                                       |
| Default Data Loader          | Document Data Loader                   | Attaches metadata to document chunks   | Recursive Character Text Splitter | Qdrant Vector Store Insert      |                                                       |
| Embeddings Ollama1           | Embeddings (Ollama)                    | Generates vector embeddings             | Default Data Loader             | Qdrant Vector Store Insert      |                                                       |
| Qdrant Vector Store Insert   | Vector Store (Qdrant)                  | Inserts embeddings into vector store   | Embeddings Ollama1, Extract Document Text | -                               |                                                       |
| Sticky Note                  | Sticky Note                           | Notes and documentation                 | -                               | -                               | ## Local Rag AI AGENT                                  |
| Sticky Note1                 | Sticky Note                           | Notes and documentation                 | -                               | -                               | ## Qdrant Vector store and Ollama Embeddings          |
| Sticky Note4                 | Sticky Note                           | Notes and documentation                 | -                               | -                               | ## Workflow to Create Local Knowledgebase from Google Drive Folder |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers:**  
   - Add two Google Drive Trigger nodes named `File Created` and `File Updated`.  
   - Configure `File Created` to watch folder ID `1uf6zZN51rgAuQgid4-Oi314f6mJIQdiB` with event `fileCreated`, polling every minute.  
   - Configure `File Updated` to watch folder ID `1914m3M7kRzkd5RJqAfzRY9EBcJrKemZC` with event `fileUpdated`, polling every minute.  
   - Connect Google Drive OAuth2 credentials.

2. **Extract File Metadata:**  
   - Add a `Set` node named `Set File ID`.  
   - Configure to assign two variables:  
     - `file_id` = `{{$json["id"]}}`  
     - `folder_id` = `{{$json["parents"][0]}}`  
   - Connect outputs of `File Created` and `File Updated` nodes to this node.

3. **Download and Convert File:**  
   - Add a `Google Drive` node named `Download File`.  
   - Set operation to `download`.  
   - Set file ID parameter to `={{$('Set File ID').item.json.file_id}}`.  
   - Enable file conversion to plain text for Google Docs (`docsToFormat: text/plain`).  
   - Connect `Set File ID` → `Download File`.

4. **Extract Text Content:**  
   - Add an `Extract From File` node named `Extract Document Text`.  
   - Operation: `text`.  
   - Connect `Download File` → `Extract Document Text`.

5. **Split Text into Chunks:**  
   - Add a `Recursive Character Text Splitter` node.  
   - Set chunk size to 100 characters.  
   - Connect `Extract Document Text` → `Recursive Character Text Splitter`.

6. **Load Document Data with Metadata:**  
   - Add a `Default Data Loader` node.  
   - Configure metadata fields:  
     - `file_id` = `={{$('Set File ID').item.json.file_id}}`  
     - `folder_id` = `={{$('Set File ID').item.json.folder_id}}`  
   - Connect `Recursive Character Text Splitter` → `Default Data Loader`.

7. **Generate Embeddings:**  
   - Add an `Embeddings Ollama` node named `Embeddings Ollama1`.  
   - Select model `llama3.2:latest`.  
   - Connect credentials for Ollama API.  
   - Connect `Default Data Loader` → `Embeddings Ollama1`.

8. **Insert Vectors into Qdrant:**  
   - Add a `Vector Store Qdrant` node named `Qdrant Vector Store Insert`.  
   - Set mode to `insert`.  
   - Choose collection `midjourney`.  
   - Connect Qdrant API credentials.  
   - Connect `Embeddings Ollama1` → `Qdrant Vector Store Insert`.

---

9. **Set up Chat Interaction:**  
   - Add a `When chat message received` node (LangChain Chat Trigger).  
   - Default options.  
   - No credentials needed.

10. **Add Postgres Chat Memory:**  
    - Add `Postgres Chat Memory` node.  
    - Connect Postgres credentials.  
    - No special parameters.

11. **Add Ollama Chat Model:**  
    - Add `Ollama Chat Model` node.  
    - Set model to `llama3.2:latest`.  
    - Connect Ollama API credentials.

12. **Add Ollama Language Model:**  
    - Add `Ollama Model` node.  
    - Use default options.  
    - Connect Ollama API credentials.

13. **Add Qdrant Vector Store (for query):**  
    - Add `Qdrant Vector Store` node.  
    - Choose collection `midjourney`.  
    - Connect Qdrant API credentials.

14. **Add Vector Store Tool:**  
    - Add `Answer questions with a vector store` (Tool Vector Store) node.  
    - Add description: `"this tool will be used to retrieve knowledge"`.

15. **Add AI Agent:**  
    - Add `AI Agent` node.  
    - Configure system message: `"You are a helpful assistant you have access to a knowledge base"`.  
    - Connect inputs:  
      - Chat input from `When chat message received`.  
      - Memory from `Postgres Chat Memory`.  
      - Language model from `Ollama Chat Model`.  
      - Tool from `Answer questions with a vector store`.  
    - Connect outputs to wherever you want to send chat responses.

16. **Connect Nodes for Chat Flow:**  
    - `When chat message received` → `AI Agent`.  
    - `Postgres Chat Memory` → `AI Agent` (memory).  
    - `Ollama Chat Model` → `AI Agent` (languageModel).  
    - `Answer questions with a vector store` → `AI Agent` (tool).  
    - `Qdrant Vector Store` → `Answer questions with a vector store`.  
    - `Ollama Model` → `Answer questions with a vector store`.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                         |
|--------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| ## Local Rag AI AGENT                                                                             | Sticky Note on chat interaction block                   |
| ## Qdrant Vector store and Ollama Embeddings                                                     | Sticky Note on vector store and embeddings block        |
| ## Workflow to Create Local Knowledgebase from Google Drive Folder                               | Sticky Note summarizing overall workflow purpose        |
| Google Drive folder URLs used for triggers:                                                      | [Folder 1](https://drive.google.com/drive/folders/1uf6zZN51rgAuQgid4-Oi314f6mJIQdiB), [Folder 2](https://drive.google.com/drive/folders/1914m3M7kRzkd5RJqAfzRY9EBcJrKemZC) |
| Ollama API credential requires setting up with Llama3 models                                    | Ensure Ollama account is configured for `llama3.2:latest` |
| Postgres database should be prepared to store chat memory                                       | Connection and schema readiness required                 |
| Qdrant collection `midjourney` must exist or be created before inserting vectors                | Qdrant API account must have insert & search permissions |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. The processing complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.