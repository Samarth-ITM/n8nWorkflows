Chat with Internal Documents using Ollama, Supabase Vector DB & Google Drive

https://n8nworkflows.xyz/workflows/chat-with-internal-documents-using-ollama--supabase-vector-db---google-drive-6894


# Chat with Internal Documents using Ollama, Supabase Vector DB & Google Drive

---

### 1. Workflow Overview

This workflow enables a Retrieval-Augmented Generation (RAG) AI chat interface that interacts with internal documents stored in Google Drive, indexed in a Supabase vector database, and processed using Ollama language models. The key use cases include automatic ingestion of documents added or updated in a specific Google Drive folder, vectorizing their content for semantic search, and answering user chat queries contextually based on these documents.

**Logical Blocks:**

- **1.1 Document Ingestion and Vectorization:**  
  Detects new or updated files in a designated Google Drive folder, downloads and extracts text depending on file type (PDF, Excel, Google Docs), processes and summarizes the content, generates embeddings with Ollama, and inserts the vectors into Supabase.

- **1.2 Chat Interface and RAG Agent:**  
  Provides a webhook chat endpoint and chat trigger to receive user messages, extracts user input, fetches relevant documents from Supabase vector store, and uses an Ollama-powered RAG AI agent configured with memory and tools to generate responses based on retrieved context.

- **1.3 Supporting Nodes and Utilities:**  
  Includes nodes for setting file metadata, cleaning old vector data on updates, text splitting, and memory management with Postgres.

---

### 2. Block-by-Block Analysis

#### 1.1 Document Ingestion and Vectorization

**Overview:**  
This block automatically triggers when a file is created or updated in a specific Google Drive folder. It downloads the file, extracts text based on the document type (PDF, Excel, or Google Docs), optionally summarizes the content, generates embeddings with Ollama, and inserts the processed data into Supabase vector storage for semantic search.

**Nodes Involved:**  
- File Created (Google Drive Trigger)  
- File Updated (Google Drive Trigger)  
- Set File ID (Set)  
- Delete Old Doc Rows (Supabase)  
- Download File (Google Drive)  
- Switch (Switch)  
- Extract PDF Text (Extract from File)  
- Extract from Excel (Extract from File)  
- Extract Document Text (Extract from File)  
- Aggregate (Aggregate)  
- Summarize (Summarize)  
- Character Text Splitter (Text Splitter - Langchain)  
- Default Data Loader (Langchain Document Loader)  
- Embeddings Ollama (Langchain Embeddings)  
- Insert into Supabase Vectorstore (Langchain Vector Store)

**Node Details:**

- **File Created / File Updated**  
  - Type: Google Drive Trigger  
  - Role: Detects file creation or update events in a specific Google Drive folder (`folderToWatch` with ID `1kxxE-cSJYZA1EwRohcgNL2PNFZDzAyhw`)  
  - Input: Google Drive event stream  
  - Output: Metadata of triggered file (id, mimeType, etc.)  
  - Auth: Google Drive OAuth2  
  - Edge Cases: Folder permission issues, API rate limits, missed triggers if polling interval is too large

- **Set File ID**  
  - Type: Set Node  
  - Role: Extracts and sets `file_id` and `file_type` from trigger data for downstream use  
  - Key Expressions:  
    - `file_id = {{ $json.id }}`  
    - `file_type = {{ $json.mimeType }}`  
  - Input: File metadata from trigger  
  - Output: Structured JSON with file info

- **Delete Old Doc Rows**  
  - Type: Supabase Node  
  - Role: Removes previous vector entries in the `documents` table related to the current `file_id` to avoid duplicates  
  - Filter: `metadata->>file_id like .*{{ $json.file_id }}*` (string filter)  
  - Input: File ID context  
  - Output: Confirmation of deletion  
  - Auth: Supabase API credentials  
  - Edge Cases: Failure if Supabase is down, filter syntax errors

- **Download File**  
  - Type: Google Drive Node  
  - Role: Downloads the actual file content using the file ID  
  - Options: Converts Google Docs to plain text (`text/plain`) on download  
  - Input: File ID from `Set File ID`  
  - Output: Binary file content  
  - Auth: Google Drive OAuth2  
  - Edge Cases: File not found, permission denied, large file size causing timeout

- **Switch**  
  - Type: Switch Node  
  - Role: Routes the workflow based on the file MIME type to appropriate text extraction:  
    - PDF → Extract PDF Text  
    - Excel → Extract from Excel  
    - Google Docs → Extract Document Text  
  - Input: File type string  
  - Output: Branch to corresponding extraction node  
  - Edge Cases: Unsupported file types fallback to default

- **Extract PDF Text / Extract from Excel / Extract Document Text**  
  - Type: Extract from File  
  - Role: Extracts textual content from respective file formats  
  - Input: Downloaded file binary  
  - Output: Extracted text data  
  - Edge Cases: Extraction failure due to corrupted files or unsupported features

- **Aggregate**  
  - Type: Aggregate Node  
  - Role: Aggregates extracted data chunks from Excel extraction into one cohesive text block  
  - Input: Multiple extracted data items  
  - Output: Single aggregated text string

- **Summarize**  
  - Type: Summarize Node  
  - Role: Concatenates or summarizes aggregated document text (field: `data`) to reduce size or enhance content quality before embedding  
  - Input: Aggregated text  
  - Output: Summarized text string

- **Character Text Splitter**  
  - Type: Langchain Text Splitter  
  - Role: Splits summarized text into manageable chunks for embedding generation  
  - Input: Summarized text string  
  - Output: Text chunks array

- **Default Data Loader**  
  - Type: Langchain Document Default Data Loader  
  - Role: Loads text chunks into Langchain document format with metadata linking back to `file_id`  
  - Input: Text chunks  
  - Output: Langchain documents ready for embedding

- **Embeddings Ollama**  
  - Type: Langchain Embeddings (Ollama)  
  - Role: Generates vector embeddings for text chunks using Ollama model `nomic-embed-text:latest`  
  - Input: Langchain documents  
  - Output: Embeddings vectors  
  - Auth: Ollama API credentials  
  - Edge Cases: API rate limits, model availability

- **Insert into Supabase Vectorstore**  
  - Type: Langchain Vector Store Supabase  
  - Role: Inserts embeddings with metadata into Supabase `documents` table for later semantic search  
  - Table: `documents`  
  - Input: Embeddings vectors  
  - Output: Confirmation of insertion  
  - Auth: Supabase API credentials

---

#### 1.2 Chat Interface and RAG Agent

**Overview:**  
This block exposes a public webhook and chat trigger to receive user chat messages, processes user input, retrieves relevant documents from Supabase vector store, and uses an Ollama-powered RAG AI agent with memory to generate context-aware answers.

**Nodes Involved:**  
- When chat message received (Langchain Chat Trigger)  
- Webhook (n8n Webhook)  
- Edit Fields (Set)  
- RAG AI Agent (Langchain Agent)  
- User_documents (Langchain Tool Vector Store)  
- Supabase Vector Store (Langchain Vector Store)  
- Ollama Chat Model (Langchain LM Chat)  
- Ollama Model (Langchain LM)  
- Postgres Chat Memory (Langchain Memory Postgres)  
- Respond to Webhook (Respond to Webhook)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Public chat interface webhook that supports file uploads and initiates chat with greeting message  
  - Initial Message: "Hi there! 👋 My name is Laki. How can I assist you today?"  
  - Output: Captured user chat input and session id  
  - WebhookId: Registered in n8n instance  
  - Edge Cases: Webhook downtime, malformed messages

- **Webhook**  
  - Type: n8n Webhook  
  - Role: Receives POST chat requests at path `/rag-chat` and passes data downstream  
  - Response Mode: Responds with all incoming items  
  - Edge Cases: HTTP errors, authentication if added

- **Edit Fields**  
  - Type: Set Node  
  - Role: Normalizes user input fields for chat (`chatInput` and `sessionId`) from multiple possible JSON paths in incoming request  
  - Expressions:  
    - `chatInput = {{ $json?.chatInput || $json.body.chatInput || $json.body.message }}`  
    - `sessionId = {{ $json?.sessionId || $json.body.sessionId }}`

- **RAG AI Agent**  
  - Type: Langchain Agent  
  - Role: Core AI agent that processes user chat input with context retrieved from documents  
  - Prompt Instructions: Use only provided context to answer, call tools only if context is insufficient, keep answers clear  
  - Inputs:  
    - `chatInput` (user question)  
    - `user_documents` (retrieved context)  
  - Uses: Ollama Chat Model as LM, Postgres memory for chat history, User_documents as vector store tool  
  - Edge Cases: Model response errors, missing context, memory DB downtime

- **User_documents**  
  - Type: Langchain Tool Vector Store  
  - Role: Provides access to Supabase vector store results for context retrieval  
  - Description: Contains user document vectors for semantic search  
  - Input: Vector store (Supabase Vector Store) output  
  - Output: Contextual documents for agent

- **Supabase Vector Store**  
  - Type: Langchain Vector Store Supabase  
  - Role: Performs similarity search on Supabase `documents` table to retrieve relevant documents for query  
  - Uses Ollama embeddings for query matching  
  - Input: Embeddings from Ollama  
  - Output: Matching documents to `User_documents` tool

- **Ollama Chat Model / Ollama Model**  
  - Type: Langchain Language Model (Chat and Text)  
  - Role: Generate natural language responses (chat) or embeddings (text) via Ollama models (`llama3.1:latest`)  
  - Config: Chat model uses temperature=0.5 for controlled creativity  
  - Input/Output: Receives prompt from agent, produces answer

- **Postgres Chat Memory**  
  - Type: Langchain Memory Postgres Chat  
  - Role: Stores and retrieves conversation history to maintain chat context across interactions  
  - Input: Chat sessionId and messages  
  - Output: Memory context for agent

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends final AI-generated response back to the API caller  
  - Input: Output of RAG AI Agent  

---

#### 1.3 Supporting Nodes and Utilities

**Overview:**  
Supporting nodes provide metadata management, note-taking for workflow organization, and minor helper utilities.

**Nodes Involved:**  
- Sticky Note, Sticky Note1, Sticky Note2

**Node Details:**

- **Sticky Notes**  
  - Type: n8n Sticky Note  
  - Role: Provide visual workflow comments and section labeling  
  - Contents:  
    - "Agent Tools for RAG"  
    - "Tool to Add a Google Drive File to Vector DB"  
    - "RAG AI Agent with Chat Interface"  
  - No functional impact, purely documentation aids

---

### 3. Summary Table

| Node Name               | Node Type                                      | Functional Role                                  | Input Node(s)              | Output Node(s)             | Sticky Note                                            |
|-------------------------|------------------------------------------------|-------------------------------------------------|----------------------------|----------------------------|--------------------------------------------------------|
| File Created            | Google Drive Trigger                            | Trigger on new file in Drive folder              |                            | Set File ID                | Tool to Add a Google Drive File to Vector DB            |
| File Updated            | Google Drive Trigger                            | Trigger on updated file in Drive folder          |                            | Set File ID                | Tool to Add a Google Drive File to Vector DB            |
| Set File ID             | Set                                            | Extract file metadata (id, type)                  | File Created, File Updated  | Delete Old Doc Rows        | Tool to Add a Google Drive File to Vector DB            |
| Delete Old Doc Rows     | Supabase                                       | Delete previous vectors for this file             | Set File ID                 | Download File              | Tool to Add a Google Drive File to Vector DB            |
| Download File           | Google Drive                                   | Download file content                              | Delete Old Doc Rows         | Switch                    | Tool to Add a Google Drive File to Vector DB            |
| Switch                  | Switch                                         | Route based on file MIME type                      | Download File               | Extract PDF Text, Extract from Excel, Extract Document Text | Tool to Add a Google Drive File to Vector DB            |
| Extract PDF Text        | Extract From File                              | Extract text from PDF                              | Switch (PDF branch)         | Insert into Supabase Vectorstore | Tool to Add a Google Drive File to Vector DB            |
| Extract from Excel      | Extract From File                              | Extract text from Excel                            | Switch (Excel branch)       | Aggregate                  | Tool to Add a Google Drive File to Vector DB            |
| Aggregate               | Aggregate                                      | Aggregate Excel extraction chunks                  | Extract from Excel          | Summarize                  | Tool to Add a Google Drive File to Vector DB            |
| Summarize               | Summarize                                      | Summarize aggregated text                          | Aggregate                   | Insert into Supabase Vectorstore | Tool to Add a Google Drive File to Vector DB            |
| Extract Document Text   | Extract From File                              | Extract text from Google Docs                      | Switch (Google Docs branch) | Insert into Supabase Vectorstore | Tool to Add a Google Drive File to Vector DB            |
| Character Text Splitter | Langchain Text Splitter                        | Split text into chunks for embeddings             | Summarize                  | Default Data Loader        | Tool to Add a Google Drive File to Vector DB            |
| Default Data Loader     | Langchain Document Loader                      | Load text chunks as Langchain document             | Character Text Splitter     | Embeddings Ollama          | Tool to Add a Google Drive File to Vector DB            |
| Embeddings Ollama       | Langchain Embeddings                           | Generate vector embeddings with Ollama             | Default Data Loader         | Insert into Supabase Vectorstore | Tool to Add a Google Drive File to Vector DB            |
| Insert into Supabase Vectorstore | Langchain Vector Store Supabase               | Insert embeddings into Supabase vector DB          | Embeddings Ollama, Extract PDF Text, Summarize, Extract Document Text |                            | Tool to Add a Google Drive File to Vector DB            |
| When chat message received | Langchain Chat Trigger                        | Public chat webhook trigger                         |                            | Edit Fields                | RAG AI Agent with Chat Interface                         |
| Webhook                 | n8n Webhook                                    | Receives chat POST requests                         |                            | Edit Fields                | RAG AI Agent with Chat Interface                         |
| Edit Fields             | Set                                            | Normalize chat input and sessionId                  | When chat message received, Webhook | RAG AI Agent              | RAG AI Agent with Chat Interface                         |
| RAG AI Agent            | Langchain Agent                               | Core AI agent for answering based on context       | Edit Fields, User_documents, Postgres Chat Memory, Ollama Chat Model, Ollama Model | Respond to Webhook        | RAG AI Agent with Chat Interface                         |
| User_documents          | Langchain Tool Vector Store                   | Tool for accessing relevant documents from vector store | Supabase Vector Store       | RAG AI Agent              | RAG AI Agent with Chat Interface                         |
| Supabase Vector Store   | Langchain Vector Store Supabase               | Retrieve relevant documents for queries             | Embeddings Ollama1          | User_documents             | RAG AI Agent with Chat Interface                         |
| Embeddings Ollama1      | Langchain Embeddings                           | Generate query embeddings with Ollama               |                           | Supabase Vector Store      | RAG AI Agent with Chat Interface                         |
| Ollama Chat Model       | Langchain LM Chat                             | Ollama chat model for RAG AI Agent                  |                           | RAG AI Agent              | RAG AI Agent with Chat Interface                         |
| Ollama Model            | Langchain LM                                  | Ollama text model for embeddings                     |                           | User_documents             | RAG AI Agent with Chat Interface                         |
| Postgres Chat Memory    | Langchain Memory Postgres Chat                | Stores chat history for session memory               |                           | RAG AI Agent              | RAG AI Agent with Chat Interface                         |
| Respond to Webhook      | Respond to Webhook                            | Sends final AI response back to caller               | RAG AI Agent               |                            | RAG AI Agent with Chat Interface                         |
| Sticky Note             | Sticky Note                                   | Workflow section comment                             |                            |                            | Agent Tools for RAG                                      |
| Sticky Note1            | Sticky Note                                   | Workflow section comment                             |                            |                            | Tool to Add a Google Drive File to Vector DB            |
| Sticky Note2            | Sticky Note                                   | Workflow section comment                             |                            |                            | RAG AI Agent with Chat Interface                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Folder Watch Triggers:**  
   - Add two Google Drive Trigger nodes:  
     - One for `fileCreated` event on folder ID `1kxxE-cSJYZA1EwRohcgNL2PNFZDzAyhw`  
     - One for `fileUpdated` event on the same folder  
   - Use Google Drive OAuth2 credentials.

2. **Set File Metadata:**  
   - Add a Set node named `Set File ID` connected from both triggers.  
   - Extract and assign:  
     - `file_id` = `{{ $json.id }}`  
     - `file_type` = `{{ $json.mimeType }}`

3. **Delete Old Vector Entries:**  
   - Add Supabase node `Delete Old Doc Rows` connected from `Set File ID`.  
   - Configure operation: Delete from table `documents` where `metadata->>file_id like .*{{ $json.file_id }}*`.  
   - Use Supabase credentials.

4. **Download the File:**  
   - Add Google Drive node `Download File` connected from `Delete Old Doc Rows`.  
   - Configure to download by file ID `={{ $('Set File ID').item.json.file_id }}`.  
   - Enable Google Docs to plain text conversion.

5. **Switch by File Type:**  
   - Add Switch node connected from `Download File`.  
   - Add 3 conditions based on `={{ $('Set File ID').item.json.file_type }}`:  
     - Equals `application/pdf` → `Extract PDF Text` node  
     - Equals `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` → `Extract from Excel` node  
     - Equals `application/vnd.google-apps.document` → `Extract Document Text` node  
   - Default fallback to last output (Extract Document Text).

6. **Extract Text Based on Type:**  
   - Add Extract From File nodes for PDF, Excel, Google Docs connected from Switch outputs.  
   - Excel extraction connects to `Aggregate` node to combine chunks.  
   - Aggregate connects to `Summarize` node to summarize text.

7. **Text Splitting and Loading:**  
   - Connect `Summarize` output to `Character Text Splitter` (Langchain).  
   - Connect splitter output to `Default Data Loader`.  
   - `Default Data Loader` is configured to attach metadata `file_id`.

8. **Generate Embeddings:**  
   - Add `Embeddings Ollama` node connected from `Default Data Loader`.  
   - Use Ollama API credentials, model `nomic-embed-text:latest`.

9. **Insert into Supabase Vectorstore:**  
   - Add `Insert into Supabase Vectorstore` node connected from `Embeddings Ollama` and also from extraction nodes where appropriate.  
   - Configure table `documents`, query `match_documents`.  
   - Use Supabase credentials.

10. **Setup Chat Trigger and Webhook:**  
    - Add Langchain Chat Trigger node `When chat message received` (public webhook, allow file uploads, initial message greeting).  
    - Add n8n Webhook node `Webhook` with POST path `/rag-chat`.  
    - Both connect to Set node `Edit Fields` to normalize `chatInput` and `sessionId`.

11. **Configure RAG AI Agent:**  
    - Add Langchain Agent node `RAG AI Agent` connected from `Edit Fields`.  
    - Configure prompt to use only retrieved context, avoid unnecessary tool calls.  
    - Set system message accordingly.  
    - Connect Ollama Chat Model and Ollama Model nodes as LM providers.  
    - Connect `Postgres Chat Memory` node for chat memory.  
    - Connect Langchain Tool Vector Store `User_documents` node, which connects to Langchain Vector Store Supabase node for document retrieval.  
    - Use appropriate Ollama, Supabase, and Postgres credentials.

12. **Respond to Webhook:**  
    - Add `Respond to Webhook` node connected from `RAG AI Agent` to send responses back.

13. **Add Sticky Notes for Clarity:**  
    - Add Sticky Note nodes with content:  
      - "Agent Tools for RAG" near chat-related nodes  
      - "Tool to Add a Google Drive File to Vector DB" near ingestion nodes  
      - "RAG AI Agent with Chat Interface" near chat and agent nodes

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow integrates Google Drive, Supabase Vector DB, and Ollama LLMs to implement an enterprise document RAG chat assistant.      | Workflow description                                                                            |
| Ollama models `llama3.1:latest` and `nomic-embed-text:latest` are used for chat generation and embeddings respectively.                  | Ollama API credentials required                                                                |
| Supabase table `documents` stores vector embeddings with metadata including file IDs for retrieval.                                       | Supabase schema design                                                                          |
| Postgres is used to store chat conversation memory to maintain context across sessions.                                                   | Postgres connection and schema required                                                        |
| Google Drive folder ID `1kxxE-cSJYZA1EwRohcgNL2PNFZDzAyhw` is central for monitoring new or updated documents.                            | Google Drive folder configuration                                                              |
| For efficient document ingestion, large files or unsupported formats may require additional handling or error management.                | Potential workflow extension                                                                    |
| See n8n Langchain documentation for detailed configuration of Langchain nodes: https://docs.n8n.io/nodes/collection/langchain/           | Official n8n Langchain node documentation                                                      |
| Ollama API usage requires a valid API key and network access to Ollama server or cloud.                                                    | Ollama API setup details                                                                        |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---