Google Drive to Supabase Contextual Vector Database Sync for RAG Applications

https://n8nworkflows.xyz/workflows/google-drive-to-supabase-contextual-vector-database-sync-for-rag-applications-8200


# Google Drive to Supabase Contextual Vector Database Sync for RAG Applications

### 1. Workflow Overview

This workflow synchronizes documents stored in a specific Google Drive folder with a Supabase-based contextual vector database designed for Retrieval-Augmented Generation (RAG) AI applications. It automates the ingestion, extraction, contextualization, embedding, storage, update, and deletion of documents to maintain an up-to-date, searchable vector store that supports hybrid semantic and keyword search.

The workflow is divided into the following logical blocks:

- **1.1 Google Drive File Monitoring and Extraction**  
  Watches specific Google Drive folders for new or deleted files, downloads new files, and extracts their textual content.

- **1.2 Document Change Detection and Record Management**  
  Generates a hash of the document content, compares it against stored hashes in Supabase to detect new or modified files, and manages records accordingly.

- **1.3 Text Chunking and Contextual Metadata Generation**  
  Splits extracted text into manageable chunks, enriches each chunk with contextual metadata including document summaries and situational context for improved retrieval.

- **1.4 Embeddings Generation and Vector Store Upsert**  
  Generates embeddings for text chunks via OpenAI, and upserts these vectors with metadata into the Supabase vector database.

- **1.5 Vector Store Query and RAG AI Agent Interaction**  
  Provides an AI agent that receives queries, searches the Supabase vector store via an edge function, and returns answers strictly based on the database contents.

- **1.6 Trash Folder Monitoring and Cleanup**  
  Watches Google Drive trash folder, deletes corresponding records and vectors from Supabase, and permanently deletes files from Google Drive.

- **1.7 Sub-workflow Integration**  
  Delegates vector insertion and metadata management to an external sub-workflow for modularity.

---

### 2. Block-by-Block Analysis

#### 1.1 Google Drive File Monitoring and Extraction

- **Overview:**  
  Detects new files in the designated Google Drive folder, downloads files, and extracts text from PDFs to prepare for further processing.

- **Nodes Involved:**  
  - Watch GD RAG Files  
  - Loop Over Items  
  - Google Drive  
  - Extract from File  
  - Set Text

- **Node Details:**

  - **Watch GD RAG Files**  
    - Type: Google Drive Trigger  
    - Role: Triggers workflow when a new file is created in a specific "RAG Files" folder.  
    - Configuration: Polls every minute, watches folder ID "1e1Af14X5nlPq6oVEqbHs4h7pQtYktzAM".  
    - Potential failure: Authorization errors, Google Drive API rate limits.

  - **Loop Over Items**  
    - Type: Split in Batches  
    - Role: Processes each detected file individually to support batch processing.  
    - Configuration: Default batch size (unspecified).

  - **Google Drive**  
    - Type: Google Drive Node  
    - Role: Downloads the file content based on file ID from the trigger.  
    - Configuration: Operation set to "download".  
    - Credentials: Google Drive OAuth2.  
    - Failures: File not found, permission issues.

  - **Extract from File**  
    - Type: Extract from File (PDF)  
    - Role: Extracts text content from the downloaded PDF file.  
    - Configuration: Operation "pdf".  
    - Limitations: Works only with PDFs, extraction quality depends on file.

  - **Set Text**  
    - Type: Set Node  
    - Role: Stores extracted text in a named variable `text` for downstream use.  
    - Configuration: Assigns `text` from extracted file content.

---

#### 1.2 Document Change Detection and Record Management

- **Overview:**  
  Detects if a document is new, modified, or unchanged by hashing its content and comparing with stored hashes in Supabase. Updates or creates records accordingly.

- **Nodes Involved:**  
  - Generate Hash  
  - Search Record Manager  
  - Switch  
  - Create Row in Record Manager  
  - Update Record Manager  
  - Delete Previous Vectors  
  - Aggregate

- **Node Details:**

  - **Generate Hash**  
    - Type: Crypto (SHA256)  
    - Role: Generates a SHA256 hash of the extracted text to uniquely identify content changes.  
    - Configuration: Hashes the `text` property.  
    - Edge cases: Empty text results in empty hash.

  - **Search Record Manager**  
    - Type: Supabase Node (getAll)  
    - Role: Queries Supabase `record_managerhs` table for a record with matching Google Drive file ID.  
    - Configuration: Filters by `gd_file_id` equal to current file ID.  
    - Failures: Supabase connection issues, query timeouts.

  - **Switch**  
    - Type: Switch Node  
    - Role: Branching logic based on existence and hash comparison:  
      - "does not exist": No record found for file ID.  
      - "modified": Hash mismatch (file updated).  
      - "exist": Hash match (file unchanged).  
    - Configuration: Conditions compare hashes and record presence.  
    - Failures: Expression errors if data missing.

  - **Create Row in Record Manager**  
    - Type: Supabase Node (insert)  
    - Role: Inserts new record with Google Drive file ID and hash for new files.  
    - Configuration: Table `record_managerhs`, fields `gd_file_id` and `hash`.  
    - Failures: Insert conflicts, permission errors.

  - **Update Record Manager**  
    - Type: Supabase Node (update)  
    - Role: Updates existing record’s hash when file contents change.  
    - Configuration: Updates `hash` field filtered by record ID.  
    - Failures: Update conflicts, network errors.

  - **Delete Previous Vectors**  
    - Type: Supabase Node (delete)  
    - Role: Deletes all vector entries in `documentshs` associated with the file ID before re-inserting updated vectors.  
    - Configuration: Deletes where `metadata->>file_id` equals current file ID.  
    - Failures: Delete failures, concurrency issues.

  - **Aggregate**  
    - Type: Aggregate Node  
    - Role: Aggregates all deleted vector items as output for downstream processing.  
    - Configuration: Aggregates all item data.

---

#### 1.3 Text Chunking and Contextual Metadata Generation

- **Overview:**  
  Prepares the document text for embedding by splitting it into chunks, generating a summary, adding metadata, and enriching chunks with context to improve search relevance.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - Structured Output Parser  
  - Set Text for Chunking  
  - Call My Sub-workflow  
  - Recursive Splitter2 (code)  
  - Split Out  
  - Metadata  
  - Add Context  
  - Loop Over Items2  
  - Aggregate2  
  - OpenAI Chat Model2  
  - Set Up Chunks for Embedding  
  - Wait

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Generates a JSON summary describing the document content and classifies it by motorsport category.  
    - Configuration: Uses prompt instructing JSON output with field `document_summary`.  
    - Model: GPT-4.1 via OpenAI Chat Model1.  
    - Failures: Model API errors, invalid JSON output.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Parses JSON summary output into structured data (`documentSummary`).  
    - Configuration: Uses JSON schema example with one field.

  - **Set Text for Chunking**  
    - Type: Set Node  
    - Role: Prepares variables for sub-workflow inputs, including content, summary, file ID, and file name.

  - **Call My Sub-workflow**  
    - Type: Execute Workflow Node  
    - Role: Sends chunked text and metadata to an external workflow that handles embedding and database upsert.  
    - Inputs: `content`, `file_id`, `fileName`, `documentSummary`.  
    - Sub-workflow ID: "OB2T8YYdXtHcujuT".  
    - Failures: Sub-workflow execution errors.

  - **Recursive Splitter2**  
    - Type: Code Node  
    - Role: Splits document text into chunks (~1000 characters with 200 overlap), attempting paragraph, sentence, or word splits.  
    - Configuration: Custom JavaScript code to split text intelligently.  
    - Edge cases: Very short texts, no split points.

  - **Split Out**  
    - Type: Split Out Node  
    - Role: Splits array of chunks into individual items for processing.

  - **Metadata**  
    - Type: Set Node  
    - Role: Assigns document metadata fields to each chunk for later use (summary, category, file ID, file name).

  - **Add Context**  
    - Type: LangChain Chain LLM  
    - Role: Adds situational context to each chunk by instructing GPT-4.1-nano to provide a succinct context summary per chunk.  
    - Prompt includes instructions to reconstruct incomplete info and maintain data integrity.  
    - Failures: Model errors, rate limits.

  - **Loop Over Items2**  
    - Type: Split In Batches  
    - Role: Processes chunks in batches to respect API rate limits and chunk size constraints.

  - **Aggregate2**  
    - Type: Aggregate Node  
    - Role: Aggregates processed chunks after context enrichment.

  - **Set Up Chunks for Embedding**  
    - Type: Set Node  
    - Role: Prepares chunk text for embedding insertion, adding trailing whitespace for safety.

  - **Wait**  
    - Type: Wait Node  
    - Role: Introduces delay to handle API rate limits before sending data to vector store.

---

#### 1.4 Embeddings Generation and Vector Store Upsert

- **Overview:**  
  Generates vector embeddings for contextualized text chunks using OpenAI embeddings API and upserts these into the Supabase vector database with associated metadata.

- **Nodes Involved:**  
  - Embeddings OpenAI1  
  - Default Data Loader  
  - Supabase Vector Store1

- **Node Details:**

  - **Embeddings OpenAI1**  
    - Type: LangChain embeddings OpenAI  
    - Role: Generates vector embeddings for the text chunks.  
    - Model: OpenAI embeddings model, default parameters.  
    - Credentials: OpenAI API key.  
    - Failures: API quota exceeded, network issues.

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Prepares document objects with metadata for vector store insertion.  
    - Metadata: Includes file ID, motorsport category, file name, and document summary.

  - **Supabase Vector Store1**  
    - Type: LangChain Vector Store Supabase  
    - Role: Inserts or updates vectors and metadata into Supabase `documentshs` table.  
    - Operation: Insert mode.  
    - Credentials: Supabase API key for RAG instance.  
    - Failures: Database connection errors, insertion conflicts.

---

#### 1.5 Vector Store Query and RAG AI Agent Interaction

- **Overview:**  
  Accepts user queries via chat, runs hybrid semantic and keyword search on Supabase vector database via an edge function, and returns answers strictly sourced from database matches.

- **Nodes Involved:**  
  - When chat message received  
  - AI Agent  
  - Simple Memory  
  - OpenAI Chat Model  
  - Query Vector Store  
  - When Executed by Another Workflow  
  - Embedding  
  - Edge Function

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Webhook trigger for chat interface input.  
    - Parameters: Default, no additional options.  
    - Failures: Webhook timeout, auth errors.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates query processing ensuring answers only come from vector database results.  
    - System message instructs strict source limitation.  
    - Connections: Receives input from chat trigger; invokes query tool.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a sliding window of last 10 interactions for conversational context.  
    - Failures: Memory overflow unlikely; data consistency.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Language model used by AI Agent for answer generation.  
    - Model: GPT-4.1.  
    - Credentials: OpenAI API.

  - **Query Vector Store**  
    - Type: LangChain Tool Workflow  
    - Role: Calls this same workflow as a tool to query vector database based on user query embedding.  
    - Parameters: Passes query string; expects vector search results.  
    - Failures: Recursive call depth, parameter mismatches.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for vector store query calls from AI Agent tool node.  
    - Inputs: Expects `query` string parameter.

  - **Embedding**  
    - Type: HTTP Request  
    - Role: Calls OpenAI embedding endpoint to convert query text into vector form.  
    - Model: "text-embedding-3-small".  
    - Failures: API key issues, rate limits.

  - **Edge Function**  
    - Type: HTTP Request  
    - Role: Calls Supabase edge function that executes hybrid search combining semantic and keyword similarity.  
    - Authentication: HTTP header with Bearer token.  
    - Input: JSON with query embedding, query text, and search parameters.  
    - Failures: Endpoint unavailability, auth failures.

---

#### 1.6 Trash Folder Monitoring and Cleanup

- **Overview:**  
  Watches Google Drive trash folder for new files (deleted files), removes their records and vectors from Supabase, and deletes files permanently from Google Drive.

- **Nodes Involved:**  
  - Watch GD Trash  
  - Loop Over Items1  
  - Search Record Manager1  
  - If1  
  - Delete Previous Vectors1  
  - Aggregate1  
  - Delete Record from Record Manager1  
  - Google Drive2

- **Node Details:**

  - **Watch GD Trash**  
    - Type: Google Drive Trigger  
    - Role: Triggers on file creation in Trash folder ID `1gwf7pIH8X5wU-i0YMw-j7qPSA4MnA40L`.  
    - Polling: Every minute.  
    - Failures: Google API permission issues.

  - **Loop Over Items1**  
    - Type: Split In Batches  
    - Role: Processes each trashed file.

  - **Search Record Manager1**  
    - Type: Supabase Node (getAll)  
    - Role: Searches `record_managerhs` for records with trashed file's Google Drive ID.  
    - Failures: DB connection problems.

  - **If1**  
    - Type: If Node  
    - Role: Checks if record exists for the trashed file.  
    - Condition: Existence check on Supabase query result.

  - **Delete Previous Vectors1**  
    - Type: Supabase Node (delete)  
    - Role: Deletes all vectors linked to trashed file ID from `documentshs`.  
    - Failures: DB delete errors.

  - **Aggregate1**  
    - Type: Aggregate Node  
    - Role: Aggregates deleted vector records.

  - **Delete Record from Record Manager1**  
    - Type: Supabase Node (delete)  
    - Role: Deletes record manager entry for trashed file.  
    - Failures: Delete conflicts.

  - **Google Drive2**  
    - Type: Google Drive Node  
    - Role: Deletes the file permanently from Google Drive trash.  
    - Failures: API errors, file locking.

---

#### 1.7 Sub-workflow Integration

- **Overview:**  
  Offloads the embedding and vector insertion logic into a separate reusable workflow to maintain modularity and separation of concerns.

- **Nodes Involved:**  
  - Call My Sub-workflow

- **Node Details:**

  - **Call My Sub-workflow**  
    - Type: Execute Workflow  
    - Role: Invokes sub-workflow "OB2T8YYdXtHcujuT" to handle embedding and vector upsert.  
    - Inputs: Passes `content`, `file_id`, `fileName`, and `documentSummary`.  
    - Failures: Sub-workflow not found, execution errors.

---

### 3. Summary Table

| Node Name                  | Node Type                                  | Functional Role                                         | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                               |
|----------------------------|--------------------------------------------|---------------------------------------------------------|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| Watch GD RAG Files          | Google Drive Trigger                       | Trigger on new files in Google Drive RAG folder         |                             | Loop Over Items                | ### Watch GD folder                                                                                       |
| Loop Over Items            | Split In Batches                           | Process each new file individually                        | Watch GD RAG Files           | Google Drive                  | ### Loop over each item, as more than 1 file can be placed in the Google Drive                           |
| Google Drive               | Google Drive Node                          | Download file content                                    | Loop Over Items              | Extract from File             | ### Download file                                                                                        |
| Extract from File          | Extract from File (PDF)                    | Extract text from PDF                                    | Google Drive                 | Set Text                     | ### Extract text from pdf                                                                                |
| Set Text                   | Set Node                                  | Store extracted text                                     | Extract from File            | Generate Hash                 | ### Set Text                                                                                            |
| Generate Hash              | Crypto (SHA256)                           | Generate hash of text content                            | Set Text                    | Search Record Manager         | ### Generate Hash based on the text. If text changes we have a different hash                            |
| Search Record Manager      | Supabase (getAll)                         | Check if file record exists in Supabase                  | Generate Hash               | Switch                       | ### Search the record manager to see if we have any files in the database, that have the same file id. If it does, it will return the hash |
| Switch                     | Switch Node                              | Branch based on file existence and modification status  | Search Record Manager       | Create Row in Record Manager, Delete Previous Vectors, Loop Over Items | ### Compare the Hash from generated hash and (if it exists) hash from record manager search to determine if file exists or not and if modified |
| Create Row in Record Manager| Supabase (insert)                        | Insert new file record into record manager               | Switch                      | Basic LLM Chain              | ### Create new record in record manager since file is new and doesn't exist in database                  |
| Delete Previous Vectors    | Supabase (delete)                         | Delete old vectors for modified files                    | Switch                      | Aggregate                    | ### If the doc is modified, we delete all the vectors related to that google id and update record manager id and hash |
| Aggregate                  | Aggregate Node                           | Aggregate deleted vector data                            | Delete Previous Vectors     | Update Record Manager        |                                                                                                          |
| Update Record Manager      | Supabase (update)                        | Update hash in record manager for modified files         | Aggregate                   | Basic LLM Chain              |                                                                                                          |
| Basic LLM Chain            | LangChain LLM Chain                      | Generate document summary and classification             | Update Record Manager / Create Row in Record Manager | Set Text for Chunking     | ### Create summary of document for metadata                                                             |
| Structured Output Parser   | LangChain Output Parser                  | Parse JSON output from LLM                               | Basic LLM Chain             | Set Text for Chunking        |                                                                                                          |
| Set Text for Chunking      | Set Node                                | Prepare data for chunking and sub-workflow call         | Structured Output Parser    | Call My Sub-workflow         | ### Set text to send to sub-workflow                                                                    |
| Call My Sub-workflow       | Execute Workflow                        | Send chunked content and metadata to sub-workflow       | Set Text for Chunking       | Loop Over Items              | # This is the Call My Sub-Workflow                                                                      |
| Recursive Splitter2        | Code Node                               | Split document text into chunks (1000 chars with overlap)| Metadata                    | Split Out                   | ### Chuck document into multiple chunks based chunk size                                                |
| Split Out                  | Split Out Node                          | Split chunks into individual items for embedding         | Recursive Splitter2         | Loop Over Items2             |                                                                                                          |
| Metadata                   | Set Node                                | Add metadata fields to each chunk                         | Add Context                 | Recursive Splitter2          | ### Set up data                                                                                          |
| Add Context                | LangChain Chain LLM                     | Add situational context to each chunk                     | Loop Over Items2            | Set Up Chunks for Embedding  | ### Add context to each chunk                                                                            |
| Loop Over Items2           | Split In Batches                       | Process chunks in batches                                  | Split Out                  | Aggregate2 / Add Context     | ### Set up text chunks and add timer so create limits are not reached                                    |
| Aggregate2                 | Aggregate Node                         | Aggregate chunk processing results                        | Loop Over Items2            | Add Context                 |                                                                                                          |
| Set Up Chunks for Embedding| Set Node                                | Prepare chunks for embedding insertion                    | Add Context                 | Wait                        |                                                                                                          |
| Wait                       | Wait Node                              | Wait to avoid hitting API limits                          | Set Up Chunks for Embedding | Supabase Vector Store1       |                                                                                                          |
| Embeddings OpenAI1         | LangChain Embeddings OpenAI            | Generate vector embeddings                                | When Executed by Another Workflow | Supabase Vector Store1       |                                                                                                          |
| Default Data Loader        | LangChain Document Default Data Loader| Prepare documents with metadata for vector store         | Embeddings OpenAI1          | Supabase Vector Store1       |                                                                                                          |
| Supabase Vector Store1     | LangChain Vector Store Supabase        | Insert/update vectors and metadata into vector DB        | Default Data Loader / Wait  | Loop Over Items2             | ### Upsert into vector database.                                                                        |
| When chat message received | LangChain Chat Trigger                 | Trigger for chatbot queries                               |                             | AI Agent                    |                                                                                                          |
| AI Agent                  | LangChain Agent                        | Process queries using vector DB results                   | When chat message received  | Simple Memory / Query Vector Store | ## AI Agent to communicate with the database                                                        |
| Simple Memory             | LangChain Memory Buffer Window         | Maintain recent chat context                              | AI Agent                   | AI Agent                   |                                                                                                          |
| OpenAI Chat Model         | LangChain OpenAI Chat Model            | Language model for AI Agent                               | AI Agent                   |                             |                                                                                                          |
| Query Vector Store        | LangChain Tool Workflow                | Call vector database query workflow                       | AI Agent                   |                             | ## Query Seach Supabase Database Tool with workflow                                                     |
| When Executed by Another Workflow | Execute Workflow Trigger         | Entry point for vector store query calls                  |                             | Embedding                  |                                                                                                          |
| Embedding                 | HTTP Request                          | Get OpenAI embedding for query                            | When Executed by Another Workflow | Edge Function               |                                                                                                          |
| Edge Function             | HTTP Request                          | Call Supabase edge function hybrid search                 | Embedding                  |                             |                                                                                                          |
| Watch GD Trash            | Google Drive Trigger                   | Trigger on new file in Trash folder                        |                             | Loop Over Items1            | ### Watch GD Trash folder                                                                                |
| Loop Over Items1          | Split In Batches                      | Process each trashed file                                 | Watch GD Trash              | Search Record Manager1      |                                                                                                          |
| Search Record Manager1    | Supabase (getAll)                    | Search record for trashed file                            | Loop Over Items1            | If1                        | ### Search record manager on corresponding GD file id                                                  |
| If1                       | If Node                              | Check if record exists                                    | Search Record Manager1      | Delete Previous Vectors1    | ### if records exist for this id or not                                                                 |
| Delete Previous Vectors1  | Supabase (delete)                   | Delete vectors linked to trashed file                     | If1                        | Aggregate1                  | ### Delete records from supabase                                                                         |
| Aggregate1                | Aggregate Node                      | Aggregate deleted vector data                             | Delete Previous Vectors1    | Delete Record from Record Manager1 |                                                                                                         |
| Delete Record from Record Manager1 | Supabase (delete)               | Delete record manager entry for trashed file              | Aggregate1                 | Google Drive2              |                                                                                                          |
| Google Drive2             | Google Drive Node                    | Delete file permanently from Google Drive Trash           | Delete Record from Record Manager1 | Loop Over Items1            | ### Delete file from GD                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up Supabase environment:**

   - Create Supabase project and configure API keys.  
   - Create tables `documentsHS` and `record_managerHS` with the SQL code from sticky notes (including vector extensions and hybrid search functions).  
   - Deploy edge function for hybrid search as described in sticky notes, secure it with an API key.

2. **Connect credentials in n8n:**

   - Add Google Drive OAuth2 credentials for your Google Drive account.  
   - Add OpenAI API key credentials.  
   - Add Supabase API credentials with appropriate permissions.  
   - Add HTTP header authentication credentials for Supabase edge function.

3. **Google Drive file monitoring:**

   - Create a **Google Drive Trigger** node named "Watch GD RAG Files":  
     - Set event to "fileCreated".  
     - Set folder to watch the designated "RAG Files" folder ID.  
     - Poll every minute.

4. **Batch processing files:**

   - Add **Split In Batches** node "Loop Over Items" connected to the trigger.

5. **Download and extract text:**

   - Add **Google Drive** node to download file by ID from current batch item.  
   - Add **Extract from File** node (operation PDF) connected to downloaded file.  
   - Add **Set** node to save extracted text as `text`.

6. **Detect changes:**

   - Add **Crypto** node "Generate Hash" to hash the extracted text using SHA256.  
   - Add **Supabase GetAll** node "Search Record Manager" to check for existing record by Google Drive file ID.  
   - Add **Switch** node to branch based on existence and hash comparison:  
     - If no record: create new record.  
     - If hash mismatch: update record and delete old vectors.  
     - If unchanged: stop processing or continue as needed.

7. **Record management:**

   - Add **Supabase Insert** node to create new record with file ID and hash.  
   - Add **Supabase Update** node to update hash on modified file.  
   - Add **Supabase Delete** node to delete vectors related to file ID for modified files.  
   - Add **Aggregate** nodes as needed to consolidate results.

8. **Document summarization:**

   - Add **LangChain Basic LLM Chain** node with a prompt to generate a one-sentence summary and classify the document.  
   - Use **Structured Output Parser** node to parse JSON summary output.  
   - Use **Set** node to prepare data (`content`, `documentSummary`, `file_id`, `fileName`) for chunking.

9. **Chunking and context enrichment:**

   - Implement **Code** node "Recursive Splitter2" to split text into chunks (~1000 chars, 200 overlap).  
   - Add **Split Out** node to split chunks into individual items.  
   - Add **Set** node "Metadata" to append metadata fields to each chunk.  
   - Add **Split In Batches** node "Loop Over Items2" to batch process chunks.  
   - Add **LangChain Chain LLM** node "Add Context" with prompt to generate succinct context for each chunk using GPT-4.1-nano.  
   - Add **Aggregate2** node to aggregate chunk results.  
   - Add **Set** node "Set Up Chunks for Embedding" to prepare chunk text.  
   - Add **Wait** node to delay requests and avoid API limits.

10. **Embedding generation and vector store upsert:**

    - Add **LangChain Embeddings OpenAI** node to generate embeddings for chunks.  
    - Add **LangChain Document Default Data Loader** node to prepare metadata.  
    - Add **LangChain Vector Store Supabase** node to insert vectors into `documentshs` table.

11. **Sub-workflow call:**

    - Create a sub-workflow (ID "OB2T8YYdXtHcujuT") that handles chunk embedding and vector upsert.  
    - Add **Execute Workflow** node "Call My Sub-workflow" in main flow to send chunked content and metadata to sub-workflow.

12. **Trash folder monitoring and cleanup:**

    - Add **Google Drive Trigger** "Watch GD Trash" for Trash folder.  
    - Add **Split In Batches** node "Loop Over Items1".  
    - Add **Supabase GetAll** node "Search Record Manager1" to find records for trashed files.  
    - Add **If** node to check record existence.  
    - Add **Supabase Delete** node "Delete Previous Vectors1" to remove vectors by file ID.  
    - Add **Aggregate1** node to consolidate deletes.  
    - Add **Supabase Delete** node "Delete Record from Record Manager1" to remove record manager entry.  
    - Add **Google Drive** node "Google Drive2" to permanently delete the trashed file.

13. **AI Agent for querying:**

    - Add **LangChain Chat Trigger** node "When chat message received" for user queries.  
    - Add **LangChain Agent** node "AI Agent" with system prompt restricting answers strictly to database results.  
    - Add **Simple Memory** node for conversational context buffering.  
    - Add **LangChain OpenAI Chat Model** node as LLM.  
    - Add **LangChain Tool Workflow** node "Query Vector Store" that calls the same workflow (via **Execute Workflow Trigger** node) for vector search.  
    - Add **HTTP Request** node "Embedding" to get query embeddings from OpenAI.  
    - Add **HTTP Request** node "Edge Function" to call Supabase edge function with embedding and query for hybrid search.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                  | Context or Link                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Setting up Supabase: Create project, configure tables `documentsHS` and `record_managerHS` using provided SQL code, deploy edge function for hybrid search.                                                                                                   | https://supabase.com/docs/guides/ai/hybrid-search                                                                                      |
| Hybrid Search SQL code and functions for vector, keyword, and hybrid search modes are included in sticky notes to create required tables and functions in Supabase.                                                                                            | SQL Editor in Supabase                                                                                                                 |
| Workflow continuously synchronizes Google Drive folder and Supabase vector DB to keep vector search data current and contextualized.                                                                                                                          | Sticky Note with workflow summary                                                                                                     |
| Edge Function setup: Use Supabase edge functions with AI assistant deployment for hybrid search; requires authorization header with Bearer token.                                                                                                             | Supabase edge function deployment instructions                                                                                         |
| AI Agent system prompt enforces strict sourcing of answers from database results only, responding “Sorry, I don’t know” if no relevant data found.                                                                                                           | Node: AI Agent                                                                                                                        |
| OpenAI GPT-4.1 and GPT-4.1-nano models are used respectively for document summarization and chunk context generation to balance performance and cost.                                                                                                         | Nodes: OpenAI Chat Model1, OpenAI Chat Model2                                                                                        |
| Text chunking is carefully done with paragraph, sentence, and word boundaries to preserve semantic integrity and improve embedding quality.                                                                                                                | Code in Recursive Splitter2 node                                                                                                      |
| The workflow handles deletes from Trash folder by removing associated vectors and records to maintain data hygiene.                                                                                                                                           | Trash folder related nodes                                                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.