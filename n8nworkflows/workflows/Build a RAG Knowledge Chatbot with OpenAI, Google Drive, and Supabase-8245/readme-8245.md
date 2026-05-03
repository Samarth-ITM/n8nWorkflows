Build a RAG Knowledge Chatbot with OpenAI, Google Drive, and Supabase

https://n8nworkflows.xyz/workflows/build-a-rag-knowledge-chatbot-with-openai--google-drive--and-supabase-8245


# Build a RAG Knowledge Chatbot with OpenAI, Google Drive, and Supabase

### 1. Workflow Overview

This workflow implements a Retrieval-Augmented Generation (RAG) chatbot that integrates Google Drive, OpenAI, and Supabase to enable conversational AI over custom document knowledge bases. The workflow automatically ingests PDF files uploaded or updated in specific Google Drive folders, processes and stores their textual content as vector embeddings in a Supabase vector database, and enables a chat interface that queries these embeddings to provide knowledgeable answers.

The workflow is logically divided into three main blocks:

- **1.1 Data Ingestion and Vectorization:** Watches Google Drive folders for new or updated PDF files, extracts text, splits it into chunks, generates embeddings via OpenAI, and inserts them into Supabase vector store.
- **1.2 Conversational Chat Trigger and Memory:** Listens for chat messages, manages chat session memory in a PostgreSQL store, and triggers the AI agent.
- **1.3 RAG AI Agent Query and Response:** Uses the Supabase vector store to retrieve relevant document chunks, invokes OpenAI chat model to generate context-aware answers, and returns responses to users.

Additional nodes provide setup instructions and documentation to assist with configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion and Vectorization

**Overview:**  
This block manages document ingestion from Google Drive. It triggers on file creation or updates in designated folders, downloads and extracts text from PDFs, splits text into manageable chunks, generates embeddings, and inserts them into the Supabase vector database, replacing any existing data for updated files.

**Nodes Involved:**  
- File Created (Google Drive Trigger)  
- File Updated (Google Drive Trigger)  
- Set File ID (Set)  
- Loop Over Items (Split In Batches)  
- Delete a row (Supabase)  
- Download the file (Google Drive)  
- Extract from File (Extract from File)  
- Recursive Character Text Splitter (Text Splitter)  
- Default Data Loader (Document Data Loader)  
- Embeddings OpenAI (Embeddings)  
- Supabase Vector Store (Vector Store Insert)  
- Sticky Note1 (Sticky Note)

**Node Details:**

- **File Created**  
  - *Type:* Google Drive Trigger  
  - *Config:* Watches a specific Google Drive folder (configured by folder ID) for new files every hour.  
  - *Inputs:* None (trigger)  
  - *Outputs:* Emits new file metadata  
  - *Edge Cases:* Folder ID must be correct; trigger frequency may cause delay.

- **File Updated**  
  - *Type:* Google Drive Trigger  
  - *Config:* Watches another specified Google Drive folder for file updates every minute.  
  - *Inputs:* None (trigger)  
  - *Outputs:* Emits updated file metadata  
  - *Edge Cases:* Handling rapid file changes; API rate limits.

- **Set File ID**  
  - *Type:* Set  
  - *Config:* Extracts and sets `file_id` variable from triggered file metadata (`$json.id`).  
  - *Inputs:* From File Created or File Updated nodes  
  - *Outputs:* Passes `file_id` downstream  
  - *Edge Cases:* Missing `id` field or unexpected payload format.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Config:* Processes items one by one for sequential operations. No batch size specified (defaults apply).  
  - *Inputs:* From Set File ID node (array expected)  
  - *Outputs:* Batches processed downstream  
  - *Edge Cases:* Empty input arrays, batch size limits.

- **Delete a row**  
  - *Type:* Supabase  
  - *Config:* Deletes existing rows in `documents` table where metadata's `file_id` matches current file's ID (using a JSON filter string). Ensures deduplication before insertion.  
  - *Inputs:* Batch item from Loop Over Items  
  - *Outputs:* Confirms deletion success  
  - *Edge Cases:* Supabase connection errors, filter syntax errors, race conditions.

- **Download the file**  
  - *Type:* Google Drive  
  - *Config:* Downloads file by `file_id`, converting Google Docs format to plain text (`text/plain`).  
  - *Inputs:* From Delete a row node  
  - *Outputs:* File content as text stream  
  - *Edge Cases:* File permissions, unsupported formats, download failures.

- **Extract from File**  
  - *Type:* ExtractFromFile  
  - *Config:* Extracts raw text from downloaded file content. Operation set to extract text.  
  - *Inputs:* File content stream  
  - *Outputs:* Extracted text content  
  - *Edge Cases:* Unsupported file types or corrupt files.

- **Recursive Character Text Splitter**  
  - *Type:* TextSplitter  
  - *Config:* Splits extracted text into smaller chunks recursively based on character count for efficient embedding. Defaults used.  
  - *Inputs:* Extracted text content  
  - *Outputs:* Array of text chunks  
  - *Edge Cases:* Very large documents, chunk size settings impact retrieval quality.

- **Default Data Loader**  
  - *Type:* DocumentDefaultDataLoader  
  - *Config:* Loads text chunks into document objects; adds metadata field `file_id` with current file ID for traceability.  
  - *Inputs:* Text chunks from splitter  
  - *Outputs:* Documents with metadata for embedding insertion  
  - *Edge Cases:* Metadata extraction failures, empty chunks.

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings  
  - *Config:* Calls OpenAI API to generate vector embeddings for document chunks. Uses configured OpenAI credentials.  
  - *Inputs:* Document chunks  
  - *Outputs:* Embeddings vectors  
  - *Edge Cases:* API rate limits, invalid API keys, network errors.

- **Supabase Vector Store**  
  - *Type:* VectorStoreSupabase  
  - *Config:* Inserts embeddings into `documents` table in Supabase vector DB. Operates in insert mode with query name `match_documents`.  
  - *Inputs:* Embeddings from OpenAI node  
  - *Outputs:* Confirmation of insertion  
  - *Edge Cases:* Supabase connection issues, table schema mismatches.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Content:* "Data Ingestion: Adding Google Drive PDF files to Vector DB" — serves as a header for this block.

---

#### 2.2 Conversational Chat Trigger and Memory

**Overview:**  
This block listens for incoming chat messages, manages chat session state stored in a PostgreSQL database (can be Supabase), and prepares the session context for the AI agent.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- Postgres Chat Memory (Postgres Memory)  
- Sticky Note (Sticky Note)

**Node Details:**

- **When chat message received**  
  - *Type:* ChatTrigger  
  - *Config:* Exposes webhook to receive chat messages from users. Webhook ID specified.  
  - *Inputs:* External chat message requests  
  - *Outputs:* Chat message payload  
  - *Edge Cases:* Webhook security, malformed requests, webhook downtime.

- **Postgres Chat Memory**  
  - *Type:* MemoryPostgresChat  
  - *Config:* Stores chat session history keyed by user identifier in PostgreSQL. Session key derived from `$json.user`.  
  - *Inputs:* Chat messages from trigger  
  - *Outputs:* Session memory context for AI agent  
  - *Edge Cases:* DB connection failures, session key missing, data consistency.

- **Sticky Note**  
  - *Content:* "Conversational RAG AI agent" — header for chat-related nodes.

---

#### 2.3 RAG AI Agent Query and Response

**Overview:**  
This block implements the AI conversational agent that uses the Supabase vector store as a knowledge base. It retrieves relevant documents based on user queries, uses OpenAI chat models to generate contextual answers, and returns responses to the chat interface.

**Nodes Involved:**  
- RAG Vector store (VectorStoreSupabase retrieval)  
- RAG AI Agent (Agent)  
- OpenAI Chat Model (Language Model)  
- Embeddings OpenAI1 (Embeddings)  

**Node Details:**

- **RAG Vector store**  
  - *Type:* VectorStoreSupabase  
  - *Config:* Operates in retrieval mode as a tool; retrieves top 6 relevant documents using `match_documents` query from `documents` table. Tool description guides the AI agent to use this knowledge base.  
  - *Inputs:* Embeddings from Embeddings OpenAI1 (used internally by agent)  
  - *Outputs:* Retrieved documents for agent  
  - *Edge Cases:* Empty vector store, query failures.

- **RAG AI Agent**  
  - *Type:* Agent  
  - *Config:* AI assistant configured with system message instructing how to behave when answering queries using the knowledge base. It uses retrieved documents and chat memory to generate responses.  
  - *Inputs:* Chat messages, memory, vector store retrieval results, and OpenAI chat model outputs  
  - *Outputs:* Chatbot responses  
  - *Edge Cases:* Empty knowledge base, ambiguous queries, API errors.

- **OpenAI Chat Model**  
  - *Type:* LMChatOpenAi  
  - *Config:* OpenAI chat language model invoked by the agent to generate natural language responses. Uses configured OpenAI credentials.  
  - *Inputs:* Prompt from agent  
  - *Outputs:* Generated text response  
  - *Edge Cases:* API rate limits, timeout, invalid prompt.

- **Embeddings OpenAI1**  
  - *Type:* EmbeddingsOpenAi  
  - *Config:* Used internally by the AI agent for embedding user queries to perform vector retrieval.  
  - *Inputs:* User chat query text  
  - *Outputs:* Query embeddings for retrieval  
  - *Edge Cases:* API failures, inconsistent embedding versions.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                              | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                     |
|-------------------------|------------------------------------|----------------------------------------------|-------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Complete Setup Guide     | Sticky Note                        | Provides detailed setup instructions          |                         |                              | ## 🚀 COMPLETE SETUP GUIDE - READ FIRST! (Full configuration and setup steps)                   |
| Sticky Note1            | Sticky Note                        | Header for data ingestion block                |                         |                              | ## Data Ingestion: Adding Google Drive PDF files to Vector DB                                  |
| File Created            | Google Drive Trigger               | Trigger on new file creation in Drive folder  |                         | Set File ID                  |                                                                                                |
| File Updated            | Google Drive Trigger               | Trigger on file updates in Drive folder       |                         | Set File ID                  |                                                                                                |
| Set File ID             | Set                              | Extracts and sets file_id from trigger data   | File Created, File Updated | Loop Over Items              |                                                                                                |
| Loop Over Items         | Split In Batches                  | Processes file(s) sequentially                 | Set File ID              | Delete a row (batch 2), (empty batch 1) |                                                                                                |
| Delete a row            | Supabase                         | Deletes existing vector entries for file_id   | Loop Over Items (batch 2) | Download the file            |                                                                                                |
| Download the file       | Google Drive                     | Downloads file content as plain text           | Delete a row             | Extract from File            |                                                                                                |
| Extract from File       | ExtractFromFile                  | Extracts text content from downloaded file     | Download the file        | Supabase Vector Store        |                                                                                                |
| Recursive Character Text Splitter | Text Splitter                    | Splits extracted text into chunks               |                          | Default Data Loader          |                                                                                                |
| Default Data Loader     | DocumentDefaultDataLoader         | Loads chunks as documents with metadata        | Recursive Character Text Splitter | Supabase Vector Store        |                                                                                                |
| Embeddings OpenAI       | OpenAI Embeddings                | Generates vector embeddings for document chunks | Default Data Loader       | Supabase Vector Store        |                                                                                                |
| Supabase Vector Store   | VectorStoreSupabase              | Inserts embeddings into Supabase vector table  | Extract from File, Embeddings OpenAI | Loop Over Items              |                                                                                                |
| Sticky Note             | Sticky Note                      | Header for chat conversational block           |                         |                              | ## Conversational RAG AI agent                                                                |
| When chat message received | ChatTrigger                     | Receives chat messages from users              |                         | RAG AI Agent                |                                                                                                |
| Postgres Chat Memory    | MemoryPostgresChat               | Handles chat session memory in PostgreSQL      | When chat message received | RAG AI Agent                |                                                                                                |
| RAG Vector store        | VectorStoreSupabase              | Retrieves documents from vector DB for chat    | Embeddings OpenAI1       | RAG AI Agent                |                                                                                                |
| RAG AI Agent            | Agent                           | Coordinates retrieval and response generation  | When chat message received, Postgres Chat Memory, OpenAI Chat Model, RAG Vector store |                  |                                                                                                |
| OpenAI Chat Model       | LMChatOpenAi                    | Generates chat responses                        | RAG AI Agent             | RAG AI Agent                |                                                                                                |
| Embeddings OpenAI1      | OpenAI Embeddings                | Embeds user chat queries for retrieval         | RAG AI Agent             | RAG Vector store            |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note - Complete Setup Guide**  
   - Type: StickyNote  
   - Content: Paste the full setup instructions as provided in the original workflow.  
   - Position accordingly.

2. **Create Google Drive Trigger - File Created**  
   - Event: fileCreated  
   - Poll Interval: every hour  
   - Watch Folder: Set to your Google Drive folder ID for new files  
   - Credentials: Google Drive OAuth2

3. **Create Google Drive Trigger - File Updated**  
   - Event: fileUpdated  
   - Poll Interval: every minute  
   - Watch Folder: Set to your Google Drive folder ID for updated files  
   - Credentials: Google Drive OAuth2

4. **Create Set Node - Set File ID**  
   - Assign variable `file_id` = `{{$json.id}}` (from trigger output)  
   - Connect outputs from both File Created and File Updated triggers to this node.

5. **Create SplitInBatches Node - Loop Over Items**  
   - Default batch size (process items one by one)  
   - Connect output of Set File ID to input.

6. **Create Supabase Node - Delete a row**  
   - Operation: delete  
   - Table: `documents`  
   - Filter Type: string  
   - Filter String: `=metadata->>file_id=like.*{{$json.file_id}}*` (to remove existing entries for this file)  
   - Credentials: Supabase API key and URL  
   - Connect second output of Loop Over Items (batch 2) to this node.

7. **Create Google Drive Node - Download the file**  
   - Operation: download  
   - File ID: `{{$json.file_id}}` (from previous)  
   - Conversion: docsToFormat = `text/plain`  
   - Credentials: Google Drive OAuth2  
   - Connect output of Delete a row to this node.

8. **Create ExtractFromFile Node - Extract from File**  
   - Operation: text extraction  
   - Connect output of Download the file to this node.

9. **Create Text Splitter Node - Recursive Character Text Splitter**  
   - Use default recursive character splitting options  
   - Connect output of Extract from File to this node.

10. **Create Document Data Loader Node - Default Data Loader**  
    - Metadata: Add field `file_id` with value `{{$node["Set File ID"].json["file_id"]}}`  
    - JSON Data: Pass text chunks from the splitter  
    - Connect output of Recursive Character Text Splitter to this node.

11. **Create Embeddings Node - Embeddings OpenAI**  
    - Use OpenAI credentials  
    - Connect output of Default Data Loader to this node.

12. **Create Vector Store Node - Supabase Vector Store (Insert Mode)**  
    - Mode: insert  
    - Table Name: `documents`  
    - Query Name: `match_documents`  
    - Credentials: Supabase API key and URL  
    - Connect output of Embeddings OpenAI to this node.

13. **Connect output of Extract from File to Supabase Vector Store node's main input**  
    - (This connection exists to ensure vector store receives documents and embeddings.)

14. **Create Sticky Note - Data Ingestion Header**  
    - Content: "Data Ingestion: Adding Google Drive PDF files to Vector DB"

15. **Create Chat Trigger Node - When chat message received**  
    - Webhook ID: generate or use existing  
    - Connect webhook to receive chat messages.

16. **Create Memory Node - Postgres Chat Memory**  
    - Session Key: `{{$json.user}}`  
    - Session Id Type: customKey  
    - Credentials: PostgreSQL (can be Supabase)  
    - Connect output of chat trigger to this node.

17. **Create Vector Store Node - RAG Vector store (Retrieve Mode)**  
    - Mode: retrieve-as-tool  
    - Table Name: `documents`  
    - Query Name: `match_documents`  
    - Top K: 6  
    - Tool Description: "Use this knowledge base to answer questions from the user"  
    - Credentials: Supabase  
    - Connect output of Embeddings OpenAI1 to this node (for query embeddings).

18. **Create Embeddings Node - Embeddings OpenAI1**  
    - For query embeddings from chat user text  
    - Use OpenAI credentials  
    - Connect output of RAG AI Agent to this node as input for embeddings.

19. **Create OpenAI Chat Model Node - OpenAI Chat Model**  
    - Use OpenAI credentials  
    - Connect to RAG AI Agent as language model input.

20. **Create Agent Node - RAG AI Agent**  
    - System Message: (Use the detailed instructions from the original workflow for AI behavior)  
    - Connect inputs: chat trigger (main), Postgres Chat Memory (ai_memory), OpenAI Chat Model (ai_languageModel), RAG Vector store (ai_tool)  
    - Connect output to chat response endpoint.

21. **Create Sticky Note - Conversational RAG AI agent header**

22. **Activate workflow and test** with a PDF upload to Google Drive watch folder, verify Supabase document insertion, and chat query responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Supabase vector DB setup requires running specific SQL commands to create necessary tables and functions. Reference: https://supabase.com/docs/guides/ai/langchain?database-method=sql                                                                                                                                                                                        | Supabase SQL editor setup guide                                                                                |
| Google Drive credentials require enabling Drive API and setting up OAuth2 credentials in Google Cloud Console.                                                                                                                                                                                                                                                               | Google Drive API documentation                                                                                 |
| OpenAI API key must be obtained from OpenAI dashboard and configured in n8n credentials.                                                                                                                                                                                                                                                                                      | OpenAI API key setup                                                                                            |
| PostgreSQL can be Supabase or other PostgreSQL instance for chat memory persistence.                                                                                                                                                                                                                                                                                           | PostgreSQL connection details                                                                                   |
| Test with a single PDF file initially to debug ingestion and embedding process before scaling.                                                                                                                                                                                                                                                                                | Workflow testing recommendation                                                                                 |
| Workflow designed to handle PDF files but expects Google Drive docs to be converted to plain text for embedding.                                                                                                                                                                                                                                                             | File format and extraction note                                                                                  |
| The RAG AI Agent system message contains instructions for graceful handling of empty knowledge bases and user guidance.                                                                                                                                                                                                                                                      | AI agent configuration note                                                                                      |

---

Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.