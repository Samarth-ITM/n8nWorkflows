Local Document Question Answering with Ollama AI, Agentic RAG & PGVector

https://n8nworkflows.xyz/workflows/local-document-question-answering-with-ollama-ai--agentic-rag---pgvector-10157


# Local Document Question Answering with Ollama AI, Agentic RAG & PGVector

### 1. Workflow Overview

This n8n workflow implements a **Local Document Question Answering system** using an **Agentic Retrieval-Augmented Generation (RAG)** approach combined with Ollama AI embeddings, a PostgreSQL vector store (PGVector), and advanced document processing. It is designed to ingest and process multiple local document types (PDF, Excel, CSV, TXT), store their contents and metadata in a PostgreSQL database with vector similarity search capabilities, and provide a chat interface that intelligently selects tools and methods to answer user questions from the stored knowledge base.

**Key use cases include:**

- Local document ingestion and indexing for question answering
- Hybrid knowledge retrieval combining vector similarity search and SQL queries on tabular data
- Interactive chat interface that reasons about documents and decides on the best retrieval method
- Support for multiple file types and automatic schema extraction for tabular data
- Self-managed local AI-powered RAG system without cloud dependencies

**Logical Blocks:**

- **1.1 Database Setup:** Nodes to create and manage PostgreSQL tables for document metadata, document rows (tabular data), and vector storage.
- **1.2 Local File Monitoring & Processing:** Nodes that watch a local folder for document changes, extract text or data, and prepare documents for embedding and storage.
- **1.3 Document Text Processing & Embeddings:** Text extraction, chunking, embedding generation using Ollama, and insertion into PGVector.
- **1.4 Data Ingestion & Metadata Management:** Insertion and updating of document metadata and tabular data rows in PostgreSQL.
- **1.5 Agentic RAG Chat Interface:** Webhook and chat trigger nodes that receive user queries, intelligently select retrieval methods (vector search, full document retrieval, or SQL queries), and respond interactively.
- **1.6 Cleanup & Maintenance:** Nodes to delete old document and data records to keep the database consistent when files are updated.
- **1.7 Notes & Documentation:** Sticky notes with detailed explanations, usage instructions, and author credits.

---

### 2. Block-by-Block Analysis

#### 1.1 Database Setup

**Overview:** Initializes the PostgreSQL database schema by creating tables needed for storing document metadata and tabular data rows.

**Nodes Involved:**  
- Create Document Metadata Table  
- Create Document Rows Table (for Tabular Data)  
- Sticky Note3 (instructional note)

**Node Details:**

- **Create Document Metadata Table**  
  - *Type:* PostgreSQL node executing raw SQL  
  - *Role:* Creates `document_metadata` table with columns: `id` (PK), `title`, `created_at`, `schema` (for tabular data schema)  
  - *Input:* None (manual trigger recommended)  
  - *Output:* Confirmation of table creation  
  - *Edge Cases:* Table already exists (safe due to conditional logic)  
  - *Failures:* SQL syntax errors, connection issues  

- **Create Document Rows Table (for Tabular Data)**  
  - *Type:* PostgreSQL node executing raw SQL  
  - *Role:* Creates `document_rows` table for storing JSONB row data linked to `document_metadata` by `dataset_id`  
  - *Input:* None (manual trigger recommended)  
  - *Output:* Confirmation of table creation  
  - *Edge Cases:* Table already exists  
  - *Failures:* SQL errors, connection failures  

- **Sticky Note3**  
  - *Content:* Advises to run these nodes once to set up database tables  

---

#### 1.2 Local File Monitoring & Processing

**Overview:** Watches a local folder for new or changed files and processes them sequentially.

**Nodes Involved:**  
- Local File Trigger  
- Loop Over Items

**Node Details:**

- **Local File Trigger**  
  - *Type:* Local file trigger node  
  - *Role:* Monitors `/data/shared` folder for added or changed files, triggers workflow on events  
  - *Configuration:* Uses polling and follows symlinks, triggers on folder events  
  - *Edge Cases:* File access permissions, rapid successive changes causing duplicate triggers  
  - *Failures:* File system access errors  

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes each detected file individually in sequence  
  - *Input:* Files from trigger  
  - *Output:* Each file item passed downstream for processing  
  - *Edge Cases:* Large batch sizes can overload downstream nodes  

---

#### 1.3 Document Text Processing & Embeddings

**Overview:** Extracts text or data from documents based on file type, generates embeddings using Ollama AI, and stores them in the PostgreSQL vector store.

**Nodes Involved:**  
- Set File ID  
- Delete Old Doc Records  
- Delete Old Data Records  
- Extract PDF Text  
- Extract Document Text (for plain text files)  
- Extract from Excel  
- Extract from CSV  
- Aggregate  
- Summarize  
- Set Schema  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Ollama  
- Postgres PGVector Store  

**Node Details:**

- **Set File ID**  
  - *Type:* Set node  
  - *Role:* Parses file path to extract `file_id`, `file_type`, and `file_title` variables for downstream use  
  - *Expressions:* Uses JavaScript string split and regex to extract extensions and filenames  
  - *Input:* File item from loop  
  - *Output:* JSON with parsed file metadata  
  - *Failures:* Incorrect path structure may cause parsing errors  

- **Delete Old Doc Records**  
  - *Type:* PostgreSQL node  
  - *Role:* Deletes old vectorized document text from `documents_pg` table matching the current file_id  
  - *Input:* file_id from Set File ID  
  - *Failures:* Database connection issues  

- **Delete Old Data Records**  
  - *Type:* PostgreSQL node  
  - *Role:* Deletes old tabular data rows from `document_rows` for current dataset (file_id)  
  - *Input:* file_id from Set File ID  
  - *Failures:* DB errors  

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Routes file processing depending on `file_type` (pdf, xlsx, csv, txt)  
  - *Input:* file_type from Set File ID  
  - *Outputs:* PDF → Extract PDF Text, XLSX → Extract from Excel, CSV → Extract from CSV, TXT → Extract Document Text  

- **Extract PDF Text**  
  - *Type:* Extract from file (PDF operation)  
  - *Role:* Extracts text from PDF documents  
  - *Output:* Text data for embedding  

- **Extract Document Text**  
  - *Type:* Extract from file (text operation)  
  - *Role:* Extracts text from plain text files  

- **Extract from Excel / Extract from CSV**  
  - *Type:* Extract from file (XLSX/CSV)  
  - *Role:* Extracts tabular data rows from spreadsheets or CSV files  
  - *Output:* JSON rows for aggregation and insertion  

- **Aggregate**  
  - *Type:* Aggregate node  
  - *Role:* Concatenates all extracted rows for summarization or insertion  

- **Summarize**  
  - *Type:* Summarize node  
  - *Role:* Concatenates all text chunks into a single string (for schema extraction or embedding)  

- **Set Schema**  
  - *Type:* Set node  
  - *Role:* Extracts the schema (column names) from the first row of Excel or CSV data and prepares data for storage  
  - *Edge Cases:* Empty files or malformed rows may cause failures  

- **Recursive Character Text Splitter**  
  - *Type:* LangChain text splitter  
  - *Role:* Chunks large text into 400-character pieces for embedding  

- **Default Data Loader**  
  - *Type:* LangChain document loader  
  - *Role:* Loads chunked text data with attached metadata for vectorization  

- **Embeddings Ollama / Embeddings Ollama1**  
  - *Type:* Ollama embedding nodes  
  - *Role:* Generates text embeddings using the `nomic-embed-text:latest` model from Ollama AI  
  - *Edge Cases:* Ollama server down, model not available  

- **Postgres PGVector Store / Postgres PGVector Store1**  
  - *Type:* LangChain PGVector vector store nodes  
  - *Role:* Inserts embeddings and metadata into PostgreSQL vector table `documents_pg` or retrieves vectors as a tool  
  - *Input:* Embeddings and document metadata  
  - *Failures:* DB connection, insertion errors  

---

#### 1.4 Data Ingestion & Metadata Management

**Overview:** Manages document metadata and tabular data insertion and updates in PostgreSQL.

**Nodes Involved:**  
- Insert Document Metadata  
- Insert Table Rows  
- Update Schema for Document Metadata  
- Read/Write Files from Disk  

**Node Details:**

- **Insert Document Metadata**  
  - *Type:* PostgreSQL upsert node  
  - *Role:* Inserts or updates metadata for a document including id and title  
  - *Input:* file_id and file_title from Set File ID  
  - *Failures:* DB errors, upsert conflicts  

- **Insert Table Rows**  
  - *Type:* PostgreSQL insert node  
  - *Role:* Inserts each row of tabular data as JSONB into `document_rows` linked to document id  
  - *Input:* Aggregated rows from CSV/XLSX extraction  
  - *Edge Cases:* Large datasets may require batching  

- **Update Schema for Document Metadata**  
  - *Type:* PostgreSQL upsert node  
  - *Role:* Updates the schema field of `document_metadata` to store column names for tabular docs  
  - *Input:* schema JSON string from Set Schema  
  - *Failures:* DB errors  

- **Read/Write Files from Disk**  
  - *Type:* ReadWriteFile node  
  - *Role:* Reads full file content for embedding or processing based on file_id  
  - *Input:* file_id from Set File ID  
  - *Failures:* Missing files, read errors  

---

#### 1.5 Agentic RAG Chat Interface

**Overview:** Provides a webhook and chat-triggered interactive AI agent that uses multi-tool reasoning to answer queries by dynamically selecting RAG vector searches, SQL queries, or document retrieval.

**Nodes Involved:**  
- When chat message received  
- Edit Fields  
- RAG AI Agent  
- Respond to Webhook  
- List Documents  
- Get File Contents  
- Query Document Rows  
- Postgres Chat Memory  
- Ollama (Change Base URL)  

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain chat trigger with public webhook  
  - *Role:* Entry point for chat messages to start agent workflow  
  - *Output:* Raw incoming chat data  

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Normalizes input JSON to extract `chatInput` and `sessionId` from either body or root JSON, preparing inputs for agent  
  - *Expressions:* Handles presence or absence of both keys gracefully  

- **RAG AI Agent**  
  - *Type:* LangChain agent node  
  - *Role:* Core AI agent that reasons over the knowledge base, selects tools dynamically, and generates answers  
  - *Configuration Highlights:*  
    - System prompt defines agent as personal assistant with access to text and tabular documents  
    - Agent uses tools: RAG vector search, document metadata lookup, SQL queries on tabular data  
    - Agent decides reasoning path: RAG first, then document lookups or SQL queries as needed  
  - *Input:* chatInput from Edit Fields  
  - *Output:* AI-generated response  
  - *Edge Cases:* Model errors, tool misconfiguration, missing data  
  - *Version:* 1.6  

- **Respond to Webhook**  
  - *Type:* Respond to webhook  
  - *Role:* Sends AI agent response back to chat client  

- **List Documents**  
  - *Type:* PostgreSQL tool node  
  - *Role:* Provides a tool to the AI agent for listing all available documents and their schema  
  - *Used by:* RAG AI Agent (ai_tool connection)  

- **Get File Contents**  
  - *Type:* PostgreSQL tool node  
  - *Role:* Allows AI to fetch full text content of a document by file_id  
  - *Used by:* RAG AI Agent  

- **Query Document Rows**  
  - *Type:* PostgreSQL tool node  
  - *Role:* Allows AI to run SQL queries on tabular data rows for precise numerical or categorical analysis  
  - *Used by:* RAG AI Agent  

- **Postgres Chat Memory**  
  - *Type:* LangChain postgres memory node  
  - *Role:* Maintains chat session memory for context in multi-turn conversations  
  - *Used by:* RAG AI Agent  

- **Ollama (Change Base URL)**  
  - *Type:* LangChain OpenAI-compatible chat model node pointing to Ollama local LLM server  
  - *Role:* Uses Ollama LLM by changing base URL to local Ollama server (http://ollama:11434/v1) instead of OpenAI  
  - *Note:* Used instead of direct Ollama chat model node due to incompatibility with RAG nodes  
  - *Edge Cases:* Ollama server offline, network issues  

---

#### 1.6 Cleanup & Maintenance

**Overview:** Ensures that old document and data records are removed before inserting updated content to maintain database integrity.

**Nodes Involved:**  
- Delete Old Doc Records  
- Delete Old Data Records  

**Node Details:**

- **Delete Old Doc Records**  
  - *Role:* Deletes vectorized document text matching current file_id before re-insertion  
  - *Edge Cases:* Race conditions if multiple updates happen simultaneously  

- **Delete Old Data Records**  
  - *Role:* Deletes tabular rows linked to current dataset for fresh data insertions  

---

#### 1.7 Notes & Documentation

**Overview:** Multiple sticky notes provide explanation, instructions, and links for users.

**Nodes Involved:**  
- Sticky Note (Agent Tools for RAG)  
- Sticky Note1 (Tool to Add a Google Drive File to Vector DB)  
- Sticky Note2 (RAG AI Agent with Chat Interface)  
- Sticky Note3 (Run Each Node Once to Set Up Database Tables)  
- Sticky Note4 (Note about Ollama Chat Model Node Limitation)  
- Sticky Note9 (Comprehensive workflow description, author, and links)  

---

### 3. Summary Table

| Node Name                      | Node Type                                  | Functional Role                                | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                                                                                                                                                          |
|--------------------------------|--------------------------------------------|-----------------------------------------------|----------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Create Document Metadata Table | n8n-nodes-base.postgres                     | Create metadata table in PostgreSQL           |                                  | Create Document Rows Table         | Run once to set up database tables (Sticky Note3)                                                                                                                                                                                  |
| Create Document Rows Table (for Tabular Data) | n8n-nodes-base.postgres          | Create tabular data rows table                 | Create Document Metadata Table   |                                  | Run once to set up database tables (Sticky Note3)                                                                                                                                                                                  |
| Local File Trigger             | n8n-nodes-base.localFileTrigger             | Watch folder for document changes              |                                  | Loop Over Items                   |                                                                                                                                                                                                                                     |
| Loop Over Items               | n8n-nodes-base.splitInBatches                | Process files one by one                        | Local File Trigger               | Set File ID                      |                                                                                                                                                                                                                                     |
| Set File ID                   | n8n-nodes-base.set                           | Extract file metadata (id, type, title)        | Loop Over Items                 | Delete Old Doc Records           |                                                                                                                                                                                                                                     |
| Delete Old Doc Records        | n8n-nodes-base.postgres                      | Remove old vectorized document data            | Set File ID                    | Delete Old Data Records          |                                                                                                                                                                                                                                     |
| Delete Old Data Records       | n8n-nodes-base.postgres                      | Remove old tabular data rows                    | Delete Old Doc Records           | Insert Document Metadata         |                                                                                                                                                                                                                                     |
| Insert Document Metadata      | n8n-nodes-base.postgres                      | Insert or update document metadata              | Delete Old Data Records          | Read/Write Files from Disk       |                                                                                                                                                                                                                                     |
| Read/Write Files from Disk    | n8n-nodes-base.readWriteFile                 | Read full file content for processing           | Insert Document Metadata         | Switch                         |                                                                                                                                                                                                                                     |
| Switch                       | n8n-nodes-base.switch                         | Route processing based on file type             | Read/Write Files from Disk       | Extract PDF Text, Extract Excel, Extract CSV, Extract Document Text |                                                                                                                                                                                                                                     |
| Extract PDF Text             | n8n-nodes-base.extractFromFile (pdf operation) | Extracts text from PDF files                     | Switch (PDF output)              | Postgres PGVector Store          |                                                                                                                                                                                                                                     |
| Extract Document Text        | n8n-nodes-base.extractFromFile (text operation) | Extracts text from TXT files                      | Switch (txt output)              | Postgres PGVector Store          |                                                                                                                                                                                                                                     |
| Extract from Excel           | n8n-nodes-base.extractFromFile (xlsx operation) | Extracts rows from Excel spreadsheets             | Switch (xlsx output)             | Aggregate, Insert Table Rows     |                                                                                                                                                                                                                                     |
| Extract from CSV             | n8n-nodes-base.extractFromFile (csv operation) | Extracts rows from CSV files                       | Switch (csv output)              | Aggregate, Insert Table Rows     |                                                                                                                                                                                                                                     |
| Aggregate                   | n8n-nodes-base.aggregate                       | Concatenate extracted rows                        | Extract from Excel, Extract from CSV | Summarize, Insert Table Rows |                                                                                                                                                                                                                                     |
| Summarize                   | n8n-nodes-base.summarize                       | Concatenate all text chunks                        | Aggregate                      | Set Schema, Postgres PGVector Store |                                                                                                                                                                                                                                     |
| Set Schema                  | n8n-nodes-base.set                             | Extracts and sets schema for tabular data         | Summarize                     | Update Schema for Document Metadata |                                                                                                                                                                                                                                     |
| Update Schema for Document Metadata | n8n-nodes-base.postgres                   | Update metadata record with schema json           | Set Schema                    |                                  |                                                                                                                                                                                                                                     |
| Insert Table Rows           | n8n-nodes-base.postgres                         | Insert tabular rows as JSONB in PostgreSQL         | Aggregate                     |                                  |                                                                                                                                                                                                                                     |
| Postgres PGVector Store     | @n8n/n8n-nodes-langchain.vectorStorePGVector | Insert document embeddings into PGVector store    | Extract PDF/Text, Summarize     | Loop Over Items                 |                                                                                                                                                                                                                                     |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Chunk large text into smaller pieces               |                                | Default Data Loader             |                                                                                                                                                                                                                                     |
| Default Data Loader         | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Load chunked document text with metadata            | Recursive Character Text Splitter | Postgres PGVector Store         |                                                                                                                                                                                                                                     |
| Embeddings Ollama           | @n8n/n8n-nodes-langchain.embeddingsOllama      | Generate embeddings using Ollama AI model            | Default Data Loader             | Postgres PGVector Store          |                                                                                                                                                                                                                                     |
| Embeddings Ollama1          | @n8n/n8n-nodes-langchain.embeddingsOllama      | Generate embeddings (alternative)                   | Postgres PGVector Store1       | Postgres PGVector Store1         |                                                                                                                                                                                                                                     |
| When chat message received  | @n8n/n8n-nodes-langchain.chatTrigger           | Entry point for chat messages to agent              |                                | Edit Fields                    |                                                                                                                                                                                                                                     |
| Edit Fields                | n8n-nodes-base.set                              | Normalize chat input and session id                  | When chat message received     | RAG AI Agent                   |                                                                                                                                                                                                                                     |
| RAG AI Agent               | @n8n/n8n-nodes-langchain.agent                   | AI agent that dynamically selects tools for answering | Edit Fields                   | Respond to Webhook             |                                                                                                                                                                                                                                     |
| Respond to Webhook         | n8n-nodes-base.respondToWebhook                  | Send agent response back to webhook caller           | RAG AI Agent                  |                                |                                                                                                                                                                                                                                     |
| List Documents             | n8n-nodes-base.postgresTool                       | Tool to list document metadata for AI agent          |                                | RAG AI Agent (ai_tool)          |                                                                                                                                                                                                                                     |
| Get File Contents          | n8n-nodes-base.postgresTool                       | Tool to get full document text by file_id             |                                | RAG AI Agent (ai_tool)          |                                                                                                                                                                                                                                     |
| Query Document Rows        | n8n-nodes-base.postgresTool                       | Tool to run SQL queries on tabular data               |                                | RAG AI Agent (ai_tool)          |                                                                                                                                                                                                                                     |
| Postgres Chat Memory       | @n8n/n8n-nodes-langchain.memoryPostgresChat      | Maintains chat session context                         |                                | RAG AI Agent (ai_memory)        |                                                                                                                                                                                                                                     |
| Ollama (Change Base URL)   | @n8n/n8n-nodes-langchain.lmChatOpenAi             | Uses Ollama LLM via OpenAI-compatible node            |                                | RAG AI Agent (ai_languageModel) | Note: Ollama chat model node incompatible with RAG nodes; this workaround uses OpenAI node with base URL pointing to Ollama local server. (Sticky Note4)                                                                             |
| Sticky Note(s)             | n8n-nodes-base.stickyNote                         | Provide explanations, instructions, and author credits | Various                       |                                | Multiple sticky notes provide context on Agent Tools, Google Drive integration, RAG AI Agent description, setup instructions, and detailed author notes with links. (Sticky Note9 includes full workflow overview and link to paid version) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL Tables**  
   - Add two PostgreSQL nodes:  
     - One to create the `document_metadata` table with columns: `id` (TEXT PRIMARY KEY), `title` (TEXT), `created_at` (TIMESTAMP default now), `schema` (TEXT).  
     - One to create the `document_rows` table with columns: `id` (SERIAL PK), `dataset_id` (TEXT referencing `document_metadata.id`), `row_data` (JSONB).  
   - Run these nodes once to initialize the database.

2. **Set Up Local File Monitoring**  
   - Add a Local File Trigger node watching `/data/shared` folder for added or changed files, with polling enabled and symlink following.

3. **Split Incoming Files**  
   - Add a SplitInBatches node to process files one at a time.

4. **Extract File Metadata**  
   - Add a Set node to parse file path into `file_id` (full path), `file_type` (extension), and `file_title` (filename without extension).

5. **Clean Old Records**  
   - Add two PostgreSQL nodes to delete old records from `documents_pg` and `document_rows` tables for the given `file_id`.

6. **Insert or Update Document Metadata**  
   - Add a PostgreSQL upsert node to insert or update `document_metadata` with `id` and `title`.

7. **Read File Contents**  
   - Add a Read/Write File node to read the full file content from disk using `file_id`.

8. **Route File Processing by Type**  
   - Add a Switch node routing on `file_type`:  
     - PDF → Extract PDF Text  
     - XLSX → Extract from Excel  
     - CSV → Extract from CSV  
     - TXT → Extract Document Text  

9. **Extract Text or Tabular Data**  
   - For PDF and TXT, extract text directly.  
   - For XLSX and CSV, extract rows.

10. **Aggregate Rows for Tabular Files**  
    - Add Aggregate node to combine extracted rows.

11. **Summarize Text Data**  
    - Add Summarize node to concatenate text fields.

12. **Extract and Set Schema for Tabular Data**  
    - Add Set node to extract column names from the first row and prepare schema JSON string.

13. **Update Document Metadata Schema**  
    - Add PostgreSQL upsert node to update `schema` field in `document_metadata`.

14. **Insert Tabular Rows**  
    - Add PostgreSQL insert node to insert each row as JSONB in `document_rows`.

15. **Text Splitting and Embeddings**  
    - Add Recursive Character Text Splitter node to chunk text into ~400 character pieces.  
    - Add Default Data Loader node to load chunked data with metadata.

16. **Generate Embeddings**  
    - Add Ollama embeddings node with model `nomic-embed-text:latest` to generate vector embeddings.

17. **Insert Embeddings into PostgreSQL PGVector Store**  
    - Add PGVector Store node configured for insert mode into `documents_pg` table.

18. **Loop Back for Next File**  
    - Connect PGVector Store node output back to the SplitInBatches node to process next file.

19. **Set Up Chat Interface**  
    - Add a LangChain Chat Trigger node with a public webhook for receiving chat messages.

20. **Normalize Chat Input**  
    - Add Set node to extract `chatInput` and `sessionId` from incoming JSON.

21. **Configure AI Agent**  
    - Add LangChain Agent node configured with:  
      - System prompt describing agent as personal assistant with access to RAG tools, document metadata, and SQL queries.  
      - Tools connected: Postgres tool nodes for listing documents, getting file contents, running SQL queries.  
      - Memory node for chat session context.  
      - Language model node using OpenAI-compatible node with base URL set to local Ollama server (`http://ollama:11434/v1`) and model `qwen2.5:14b-8k`.

22. **Add PostgreSQL Tool Nodes for AI Agent**  
    - Add PostgreSQL tool nodes for:  
      - Listing documents (`document_metadata` table)  
      - Getting file contents (aggregated text from `documents_pg`)  
      - Querying document rows (SQL on `document_rows`)  

23. **Add Postgres Chat Memory Node**  
    - To maintain session state and history across chat interactions.

24. **Respond to Webhook**  
    - Add Respond to Webhook node to send AI agent’s answer back to the client.

25. **Add Sticky Notes**  
    - Add descriptive sticky notes throughout the workflow for documentation and instructions.

**Credentials Needed:**  
- PostgreSQL credentials configured to access your database (e.g., Supabase).  
- Custom OpenAI credentials with base URL overridden to local Ollama server for language model node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Provides an entirely local Agentic RAG system using n8n, Ollama AI embeddings, and PGVector to enable sophisticated question answering over local documents. Includes multi-tool reasoning and dynamic tool selection for improved accuracy over standard RAG.                                                                                                                       | Sticky Note9 includes full workflow description and author credit: [Jadai kongolo](https://my.jadaikongolo.tech)                   |
| Ollama chat model node is incompatible with the RAG nodes due to known n8n issue. Workaround uses OpenAI chat model node with base URL changed to Ollama local server (`http://ollama:11434/v1`). The API key can be dummy as it is unused for local LLMs.                                                                                                                         | Sticky Note4                                                                                                                       |
| Run the database table creation nodes once before using the workflow to ensure database schema exists.                                                                                                                                                                                                                                                                          | Sticky Note3                                                                                                                       |
| The workflow supports ingestion of multiple file types: PDF, TXT, XLSX, CSV. Tabular data is stored in JSONB format in PostgreSQL to avoid creating separate tables per dataset.                                                                                                                                                                                                 | Sticky Note1                                                                                                                       |
| The RAG AI Agent uses multiple tools: vector search on documents, full document retrieval, and SQL queries on tabular data for precise numerical analysis. It reasons dynamically about which tool to use based on user queries to provide accurate and contextual answers.                                                                                                     | Sticky Note and RAG AI Agent node configuration                                                                                   |
| Non-local (cloud) version of this Agentic RAG agent is available commercially at https://kongolo.gumroad.com/l/anxwv                                                                                                                                                                                                                                                           | Sticky Note9 link                                                                                                                  |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.