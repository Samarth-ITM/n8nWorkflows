Create a Self-Updating RAG Chatbot with Google Drive, Gemini, and Supabase

https://n8nworkflows.xyz/workflows/create-a-self-updating-rag-chatbot-with-google-drive--gemini--and-supabase-9007


# Create a Self-Updating RAG Chatbot with Google Drive, Gemini, and Supabase

### 1. Workflow Overview

This workflow creates a production-ready Retrieval-Augmented Generation (RAG) chatbot system that automatically syncs documents from a designated Google Drive folder into a vector database (Supabase), enabling AI-powered question answering based on these documents. It features automated ingestion, vectorization, and cleanup processes to keep the knowledge base current, combined with a conversational AI agent leveraging Google Gemini and Cohere for embeddings and reranking.

The workflow is structured into three main logical blocks:

- **1.1 RAG Ingestion System (Step 1):**  
  Handles fetching documents from Google Drive (on manual trigger or via Google Drive triggers), extracting text from PDFs or Google Docs, splitting text into chunks, embedding content with Google Gemini embeddings, and storing both document content and metadata into Supabase.

- **1.2 RAG Agent (Step 2):**  
  Implements the chatbot interface using LangChain's agent functionality. It receives chat messages, uses a Postgres-based chat memory, a Google Gemini language model, Cohere for reranking retrieved documents, and Supabase as a vector store to answer user queries contextually from the ingested documents.

- **1.3 RAG Cleanup (Step 3):**  
  A scheduled cleanup process that compares file IDs in Google Drive with those stored in Supabase and deletes orphaned data from both the main documents table and the document metadata table, ensuring the vector store stays synchronized with the folder content.

---

### 2. Block-by-Block Analysis

#### 2.1 RAG Ingestion System (Step 1)

**Overview:**  
This block handles the ingestion of documents from a specific Google Drive folder. It extracts text from PDFs or Google Docs, splits the text into manageable chunks, generates embeddings using Google Gemini, and inserts both the vectorized content and document metadata into Supabase. It supports both manual ingestion and triggers on file creation or update.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Search files and folders (Google Drive)  
- Loop Over Items (Batch processing)  
- Set File ID (Set metadata fields)  
- Delete Old Doc Rows (Supabase delete operation)  
- Insert Document Metadata (Postgres upsert)  
- Download File1 (Google Drive file download)  
- Switch (File type switch: PDF vs Google Doc)  
- Extract PDF Text (PDF text extraction)  
- Extract Document Text (Google Docs text extraction)  
- Character Text Splitter (Text chunking)  
- Default Data Loader1 (Prepare document data for vector insertion)  
- Embeddings Google Gemini1 (Generate embeddings)  
- Insert into Supabase Vectorstore (Insert vectors into Supabase table)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual ingestion initiation.  
  - Inputs: None  
  - Outputs: Trigger downstream ingestion nodes.  
  - Edge cases: None (manual start).

- **Search files and folders**  
  - Type: Google Drive API (fileFolder resource)  
  - Role: Queries all files in a specific Google Drive folder (using folder ID placeholder).  
  - Parameters: Returns all files with fields id, name, mimeType, webViewLink.  
  - Outputs: List of files for ingestion.  
  - Edge cases: Empty folder, permission errors.

- **Loop Over Items**  
  - Type: Split in Batches  
  - Role: Processes each file separately for ingestion.  
  - Parameters: Default batch settings.  
  - Inputs: Files list.  
  - Outputs: Single file per iteration.  
  - Edge cases: Large batch size may cause timeouts.

- **Set File ID**  
  - Type: Set  
  - Role: Extracts and sets file metadata variables: file_id, file_type, file_title, file_url.  
  - Parameters: Uses expressions referencing current item JSON.  
  - Inputs: Single file JSON.  
  - Outputs: JSON enriched with metadata.  
  - Edge cases: Missing file properties.

- **Delete Old Doc Rows**  
  - Type: Supabase (delete operation)  
  - Role: Deletes existing vector data rows from Supabase for the current file_id to avoid duplicates before re-insertion.  
  - Parameters: Filter based on metadata->>file_id field matching current file_id.  
  - Inputs: File metadata JSON.  
  - Outputs: Confirmation of deletion.  
  - Edge cases: Supabase connectivity, delete failures.

- **Insert Document Metadata**  
  - Type: Postgres (upsert)  
  - Role: Inserts or updates document metadata (id, url, title) into the `document_metadata` table in Supabase/Postgres.  
  - Parameters: Upsert on id with title and url fields.  
  - Inputs: File metadata JSON.  
  - Outputs: Confirmation of upsert.  
  - Edge cases: Unique key conflicts, connection issues.

- **Download File1**  
  - Type: Google Drive (download)  
  - Role: Downloads the actual content of the file using its file_id.  
  - Parameters: Converts Google Docs to plain text if applicable.  
  - Inputs: File metadata JSON.  
  - Outputs: File content binary or text.  
  - Edge cases: Large files, API errors, conversion failures.

- **Switch**  
  - Type: Switch  
  - Role: Routes based on file_type MIME to either PDF or Google Docs text extraction.  
  - Parameters: Checks for "application/pdf" or "application/vnd.google-apps.document".  
  - Inputs: Downloaded file content.  
  - Outputs: Extract PDF Text or Extract Document Text.  
  - Edge cases: Unsupported file types fallback.

- **Extract PDF Text**  
  - Type: Extract from File  
  - Role: Extracts textual content from PDF files.  
  - Parameters: Operation = “pdf”.  
  - Inputs: PDF file binary.  
  - Outputs: Extracted text.  
  - Edge cases: Corrupt PDFs, extraction failures.

- **Extract Document Text**  
  - Type: Extract from File  
  - Role: Extracts textual content from Google Docs files.  
  - Parameters: Operation = “text”.  
  - Inputs: Google Docs file content.  
  - Outputs: Extracted text.  
  - Edge cases: Conversion errors.

- **Character Text Splitter**  
  - Type: Text Splitter (LangChain)  
  - Role: Splits large text into chunks of 750 characters with 200 character overlap to preserve context.  
  - Parameters: chunkSize=750, chunkOverlap=200.  
  - Inputs: Extracted text.  
  - Outputs: Array of text chunks.  
  - Edge cases: Very short documents.

- **Default Data Loader1**  
  - Type: LangChain Document Default Data Loader  
  - Role: Prepares each text chunk document with metadata (file_id, title, url) for vector insertion.  
  - Parameters: Metadata fields set dynamically from file metadata; JSON data expression pulls text chunks.  
  - Inputs: Text chunks from splitter.  
  - Outputs: Documents enriched with metadata.  
  - Edge cases: Missing metadata.

- **Embeddings Google Gemini1**  
  - Type: LangChain Embeddings (Google Gemini)  
  - Role: Generates vector embeddings for each text chunk document.  
  - Parameters: Default Google Gemini embedding model.  
  - Inputs: Prepared documents with text.  
  - Outputs: Vector embeddings.  
  - Edge cases: API limits, authentication errors.

- **Insert into Supabase Vectorstore**  
  - Type: LangChain Vector Store (Supabase)  
  - Role: Inserts vector embeddings and document metadata into Supabase documents table.  
  - Parameters: Mode = insert, tableName = “documents”, queryName = “match_documents”.  
  - Inputs: Embeddings with metadata.  
  - Outputs: Confirmation of vector insertion.  
  - Edge cases: Database connection or insertion errors.

---

#### 2.2 RAG Agent (Step 2)

**Overview:**  
This block provides the conversational AI interface. It triggers on chat messages, uses Google Gemini as the language model, Cohere for reranking document retrievals, stores conversation history in Postgres chat memory, and queries Supabase vector store for relevant document context. The agent responds based on provided documents or handles simple conversational filler messages.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- Postgres Chat Memory  
- Google Gemini Chat Model  
- Reranker Cohere  
- Supabase Vector Store  
- RAG Agent

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point webhook for chat messages to start the conversation flow.  
  - Parameters: Default webhook with no extra options.  
  - Inputs: User chat message.  
  - Outputs: Triggers RAG Agent.  
  - Edge cases: Webhook connectivity, security.

- **Postgres Chat Memory**  
  - Type: LangChain Memory (Postgres)  
  - Role: Stores and retrieves conversation history for context continuity.  
  - Parameters: Configured to connect to Supabase Postgres database.  
  - Inputs: Incoming messages.  
  - Outputs: Provides memory context to RAG Agent.  
  - Edge cases: Database downtime, memory loss.

- **Google Gemini Chat Model**  
  - Type: LangChain Language Model (Google Gemini)  
  - Role: Generates language responses based on input and context.  
  - Parameters: Default Google Gemini chat model options.  
  - Inputs: User query and context.  
  - Outputs: Chat response text.  
  - Edge cases: API rate limits, auth failures.

- **Reranker Cohere**  
  - Type: LangChain Reranker (Cohere)  
  - Role: Ranks retrieved documents from Supabase to improve answer relevance.  
  - Parameters: Default reranker from Cohere API.  
  - Inputs: Candidate documents from vector store.  
  - Outputs: Ranked documents for RAG Agent.  
  - Edge cases: API key errors, timeouts.

- **Supabase Vector Store**  
  - Type: LangChain Vector Store (Supabase)  
  - Role: Retrieves top K (20) relevant documents related to query, supporting reranker.  
  - Parameters: Uses “documents” table, retrieval mode “retrieve-as-tool”, topK=20, reranking enabled.  
  - Inputs: Query embeddings.  
  - Outputs: Candidate context documents.  
  - Edge cases: DB connection errors, missing data.

- **RAG Agent**  
  - Type: LangChain Agent (RAG)  
  - Role: Central AI agent that analyzes user messages, decides intent (conversational or info query), uses context from Supabase vector store, and generates final answers.  
  - Parameters: Custom system prompt specifying strict behavior rules, including handling greetings without context and contextual answering with citations and source links.  
  - Inputs: Chat message, memory, tools (Supabase Vector Store).  
  - Outputs: Final chatbot reply.  
  - Edge cases: Prompt expression errors, incomplete context, API failures.

---

#### 2.3 RAG Cleanup (Step 3)

**Overview:**  
A scheduled maintenance process that runs daily to synchronize Supabase vector store and metadata tables with the actual files in Google Drive. It detects "orphaned" entries (files deleted from Drive but still present in Supabase) and deletes those entries to keep the knowledge base accurate.

**Nodes Involved:**  
- Schedule Trigger (disabled)  
- Get File IDs (Google Drive)  
- Supabase (get all documents)  
- Merge  
- Code1 (filter orphaned Supabase rows)  
- Delete Rows (Supabase delete in documents)  
- Get File IDs2 (Google Drive)  
- Supabase2 (get all document_metadata)  
- Merge2  
- Code2 (filter orphaned metadata rows)  
- Delete Rows2 (Supabase delete in document_metadata)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Runs the cleanup process automatically on a set interval (disabled by default).  
  - Parameters: Default interval (likely daily).  
  - Outputs: Triggers cleanup nodes.  
  - Edge cases: Disabled by default; must be activated manually.

- **Get File IDs**  
  - Type: Google Drive API (fileFolder resource)  
  - Role: Retrieves all file IDs from the designated Google Drive folder.  
  - Parameters: Query filtering by folder ID, not trashed.  
  - Outputs: List of current Drive file IDs.  
  - Edge cases: Empty folder, API rate limits.

- **Supabase**  
  - Type: Supabase (getAll operation)  
  - Role: Retrieves all entries from the `documents` vector store table.  
  - Outputs: All stored document entries.  
  - Edge cases: Connection issues.

- **Merge**  
  - Type: Merge  
  - Role: Combines Google Drive file IDs and Supabase documents data for comparison.  
  - Outputs: Combined dataset.  
  - Edge cases: Data format mismatch.

- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Compares Drive file IDs and Supabase document entries to find entries in Supabase without matching Drive files (orphans).  
  - Outputs: List of orphaned document rows to delete.  
  - Edge cases: Empty inputs, data inconsistencies.

- **Delete Rows**  
  - Type: Supabase (delete)  
  - Role: Deletes orphaned documents from `documents` table.  
  - Inputs: IDs from Code1 output.  
  - Edge cases: Delete failures.

- **Get File IDs2**  
  - Similar to Get File IDs but used for metadata table cleanup.

- **Supabase2**  
  - Type: Supabase (getAll)  
  - Role: Retrieves all entries from `document_metadata` table.  
  - Outputs: Metadata records.

- **Merge2**  
  - Type: Merge  
  - Role: Combines Drive file IDs and metadata entries for comparison.  
  - Outputs: Combined dataset.

- **Code2**  
  - Type: Code (JavaScript)  
  - Role: Determines orphaned metadata entries not present in Drive and marks them for deletion.  
  - Outputs: List of orphaned metadata rows.

- **Delete Rows2**  
  - Type: Supabase (delete)  
  - Role: Deletes orphaned entries from `document_metadata` table.  
  - Inputs: IDs from Code2 output.  
  - Edge cases: Delete failures.

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                          | Input Node(s)                        | Output Node(s)                    | Sticky Note                                                                                                                                                                                                                              |
|-------------------------|---------------------------------------------|----------------------------------------|------------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                             | Manual start of ingestion process      | None                               | Search files and folders          | #### This Manual Trigger is Used for the Initial Ingestion of Documents to the Vectorbase. After initial ingestion, activate the 2 Google Drive triggers to make this workflow production ready.                                       |
| Search files and folders | Google Drive (fileFolder resource)          | Fetch files in Drive folder             | When clicking ‘Test workflow’      | Loop Over Items                   |                                                                                                                                                                                                                                          |
| Loop Over Items          | Split in Batches                            | Process files one-by-one                 | Search files and folders           | Set File ID                      |                                                                                                                                                                                                                                          |
| Set File ID              | Set                                         | Set file metadata variables             | Loop Over Items                   | Delete Old Doc Rows              |                                                                                                                                                                                                                                          |
| Delete Old Doc Rows      | Supabase (delete)                           | Delete existing vectors for file        | Set File ID                      | Insert Document Metadata         |                                                                                                                                                                                                                                          |
| Insert Document Metadata | Postgres (upsert)                           | Upsert document metadata record         | Delete Old Doc Rows               | Download File1                  |                                                                                                                                                                                                                                          |
| Download File1           | Google Drive (download)                      | Download file content                    | Insert Document Metadata          | Switch                         |                                                                                                                                                                                                                                          |
| Switch                  | Switch                                      | Route based on file type                 | Download File1                   | Extract PDF Text, Extract Document Text |                                                                                                                                                                                                                                          |
| Extract PDF Text         | Extract from File                            | Extract text from PDF                    | Switch                         | Insert into Supabase Vectorstore |                                                                                                                                                                                                                                          |
| Extract Document Text    | Extract from File                            | Extract text from Google Docs            | Switch                         | Insert into Supabase Vectorstore |                                                                                                                                                                                                                                          |
| Character Text Splitter  | LangChain Text Splitter                      | Chunk text into smaller pieces           | Extract PDF Text / Extract Document Text (via Insert into Supabase Vectorstore input) | Default Data Loader1             |                                                                                                                                                                                                                                          |
| Default Data Loader1     | LangChain Document Data Loader               | Prepare documents with metadata          | Character Text Splitter           | Embeddings Google Gemini1        |                                                                                                                                                                                                                                          |
| Embeddings Google Gemini1 | LangChain Embeddings (Google Gemini)         | Generate embeddings for document chunks  | Default Data Loader1             | Insert into Supabase Vectorstore |                                                                                                                                                                                                                                          |
| Insert into Supabase Vectorstore | LangChain Vector Store (Supabase)           | Insert embeddings and metadata into DB   | Embeddings Google Gemini1        | Loop Over Items                  |                                                                                                                                                                                                                                          |
| When chat message received | LangChain Chat Trigger                      | Entry point for chat queries             | None                           | RAG Agent                      |                                                                                                                                                                                                                                          |
| Postgres Chat Memory     | LangChain Memory (Postgres)                  | Maintain conversation history            | When chat message received       | RAG Agent                      |                                                                                                                                                                                                                                          |
| Google Gemini Chat Model | LangChain Language Model (Google Gemini)     | Generate language model responses        | RAG Agent                      | RAG Agent                      |                                                                                                                                                                                                                                          |
| Reranker Cohere          | LangChain Reranker (Cohere)                   | Rerank retrieved documents                | Supabase Vector Store            | Supabase Vector Store           | **Go to this website, https://dashboard.cohere.com/ create an account and copy the API key to connect Reranker Cohere**                                                                                                               |
| Supabase Vector Store    | LangChain Vector Store (Supabase)              | Retrieve relevant documents               | Embeddings Google Gemini         | RAG Agent                      |                                                                                                                                                                                                                                          |
| RAG Agent                | LangChain Agent                              | Central conversational AI agent          | When chat message received, Postgres Chat Memory, Google Gemini Chat Model, Supabase Vector Store, Reranker Cohere | Chat response output            | **--> The Prompt inside the RAG agent can be fine tuned according to your needs**                                                                                                                                                        |
| Schedule Trigger         | Schedule Trigger                             | Periodic trigger for cleanup process     | None                           | Get File IDs, Supabase, Get File IDs2, Supabase2 |                                                                                                                                                                                                                                          |
| Get File IDs             | Google Drive API (fileFolder resource)         | Get current file IDs from Drive folder   | Schedule Trigger               | Merge                         |                                                                                                                                                                                                                                          |
| Supabase                 | Supabase (getAll)                            | Get all documents from vector store      | Schedule Trigger               | Merge                         |                                                                                                                                                                                                                                          |
| Merge                   | Merge                                        | Combine Drive and Supabase documents      | Get File IDs, Supabase          | Code1                        |                                                                                                                                                                                                                                          |
| Code1                   | Code (JavaScript)                            | Identify orphaned documents               | Merge                         | Delete Rows                   |                                                                                                                                                                                                                                          |
| Delete Rows             | Supabase (delete)                           | Delete orphaned document rows             | Code1                         | None                          |                                                                                                                                                                                                                                          |
| Get File IDs2            | Google Drive API (fileFolder resource)         | Get current file IDs for metadata cleanup | Schedule Trigger               | Merge2                        |                                                                                                                                                                                                                                          |
| Supabase2                | Supabase (getAll)                            | Get all document metadata                  | Schedule Trigger               | Merge2                        |                                                                                                                                                                                                                                          |
| Merge2                  | Merge                                        | Combine Drive and metadata records         | Get File IDs2, Supabase2       | Code2                        |                                                                                                                                                                                                                                          |
| Code2                   | Code (JavaScript)                            | Identify orphaned metadata records         | Merge2                        | Delete Rows2                  |                                                                                                                                                                                                                                          |
| Delete Rows2            | Supabase (delete)                           | Delete orphaned metadata rows              | Code2                         | None                          |                                                                                                                                                                                                                                          |
| Sticky Note1             | Sticky Note                                  | Label "RAG Agent (Step 2)"                 | None                           | None                          |                                                                                                                                                                                                                                          |
| Sticky Note2             | Sticky Note                                  | Label "Vector Store"                        | None                           | None                          |                                                                                                                                                                                                                                          |
| Sticky Note3             | Sticky Note                                  | Branding & contact info                     | None                           | None                          | ========================= RAG Agent with G-Drive Sync ... Contact: anirudh.n.aeran@gmail.com, LinkedIn: https://www.linkedin.com/in/anirudh-narayan-a-/                                                                                  |
| Sticky Note4             | Sticky Note                                  | Explanation of Cleanup process              | None                           | None                          | How the Cleanup Works: Scheduled daily, compares Drive folder and Supabase tables, deletes orphaned entries from `documents` and `document_metadata`.                                                                                  |
| Sticky Note5             | Sticky Note                                  | Label "RAG: Clean up (Step 3)"              | None                           | None                          |                                                                                                                                                                                                                                          |
| Sticky Note6             | Sticky Note                                  | Cohere API key instructions                  | None                           | None                          | **Go to this website, https://dashboard.cohere.com/ create an account and copy the API key to connect Reranker Cohere**                                                                                                               |
| Sticky Note7             | Sticky Note                                  | Prompt fine-tune hint for RAG Agent          | None                           | None                          | **--> The Prompt inside the RAG agent can be fine tuned according to your needs**                                                                                                                                                        |
| Sticky Note8             | Sticky Note                                  | General usage instructions and setup guide  | None                           | None                          | Comprehensive instructions for setup, credential connections, table creation, folder ID replacement, initial ingestion, chat testing, and activation of Google Drive triggers. Includes link to SQL code doc: https://docs.google.com/document/d/1tLJ7fndrDjYMfyQ1R61q5NbqDs5ZMgvhzcMzkoIzj5M/edit?usp=sharing |
| Sticky Note9             | Sticky Note                                  | Postgres connection setup instructions      | None                           | None                          | Detailed instructions for setting up Postgres connection pooling in Supabase with a link to an image: https://i.postimg.cc/htxnsprc/Screenshot-2025-10-02-143424.png                                                                  |
| Sticky Note10            | Sticky Note                                  | Cohere API key setup                          | None                           | None                          | **Go to this website, https://dashboard.cohere.com/ create an account and copy the API key to connect Reranker Cohere**                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Type: Manual Trigger  
   - Purpose: To manually start the ingestion process.

2. **Create Google Drive node to search files and folders:**  
   - Type: Google Drive (fileFolder resource)  
   - Parameters:  
     - Query: `'Your Folder ID' in parents and trashed=false` (replace with actual folder ID)  
     - Fields: `webViewLink, id, name, mimeType`  
     - Return All: true  
   - Connect from Manual Trigger.

3. **Add SplitInBatches node (Loop Over Items):**  
   - Default settings (batch processing each file).  
   - Connect from Search files and folders.

4. **Add Set node (Set File ID):**  
   - Create variables:  
     - file_id = `{{$json["id"]}}`  
     - file_type = `{{$json["mimeType"]}}`  
     - file_title = `{{$json["name"]}}`  
     - file_url = `{{$json["webViewLink"]}}`  
   - Connect from Loop Over Items output (batch item).

5. **Add Supabase node to Delete Old Doc Rows:**  
   - Operation: Delete  
   - Table: `documents`  
   - Filter: SQL string filter: `=metadata->>file_id=like.*{{$json.file_id}}*`  
   - Connect from Set File ID.

6. **Add Postgres node to Insert Document Metadata:**  
   - Operation: Upsert  
   - Table: `document_metadata` (schema: public)  
   - Columns: id=`{{$json.file_id}}`, title=`{{$json.file_title}}`, url=`{{$json.file_url}}`  
   - Matching column: id  
   - Connect from Delete Old Doc Rows.

7. **Add Google Drive node to Download File:**  
   - Operation: Download  
   - File ID: `{{$json.file_id}}`  
   - Google Docs conversion: convert to `text/plain`  
   - Connect from Insert Document Metadata.

8. **Add Switch node to check file type:**  
   - Two rules:  
     - If `file_type` equals `"application/pdf"` → route to Extract PDF Text  
     - If `file_type` equals `"application/vnd.google-apps.document"` → route to Extract Document Text  
     - Fallback: no action  
   - Connect from Download File.

9. **Add Extract From File node (Extract PDF Text):**  
   - Operation: pdf  
   - Connect from Switch PDF route.

10. **Add Extract From File node (Extract Document Text):**  
    - Operation: text  
    - Connect from Switch Google Docs route.

11. **Add Character Text Splitter node:**  
    - chunkSize: 750  
    - chunkOverlap: 200  
    - Connect from both Extract PDF and Extract Document Text nodes (merge outputs if necessary).

12. **Add LangChain Document Default Data Loader:**  
    - Metadata: Set metadata keys to `file_id`, `file_title`, `url` from Set File ID node expressions.  
    - JSON Data: Expression combining extracted text chunks.  
    - Connect from Character Text Splitter.

13. **Add LangChain Embeddings Google Gemini node:**  
    - Default embedding configuration.  
    - Connect from Default Data Loader.

14. **Add LangChain Vector Store Supabase node (Insert mode):**  
    - Mode: Insert  
    - Table: `documents`  
    - Options: queryName = `match_documents`  
    - Connect from Embeddings node.

15. **Connect Vector Store node output back to Loop Over Items:**  
    - To process next file.

---

16. **Create Chat Trigger node (When chat message received):**  
    - Default webhook, name accordingly.

17. **Create LangChain Postgres Chat Memory node:**  
    - Connect from Chat Trigger.

18. **Create LangChain Google Gemini Chat Model node:**  
    - Connect from Chat Memory.

19. **Create LangChain Cohere Reranker node:**  
    - Ensure connection to Cohere API with API key.  
    - Connect from Supabase Vector Store.

20. **Create LangChain Supabase Vector Store node (Retrieve-as-tool mode):**  
    - Table: `documents`  
    - topK: 20  
    - Use reranker: true  
    - Connect from Embeddings Google Gemini (or input query embeddings).  
    - Connect output to Reranker Cohere node input.

21. **Create LangChain Agent node (RAG Agent):**  
    - Provide system prompt with strict behavior rules (see overview).  
    - Tools: Add Supabase Vector Store node.  
    - Language Model: Google Gemini Chat Model node.  
    - Memory: Postgres Chat Memory.  
    - Connect from Chat Trigger and other nodes accordingly.

---

22. **Create Scheduled Trigger node (disabled by default):**  
    - Set to run daily or preferred interval.

23. **Create Google Drive nodes Get File IDs and Get File IDs2:**  
    - Same query for both, retrieving IDs from Drive folder.

24. **Create Supabase nodes Supabase and Supabase2:**  
    - Supabase: getAll on `documents` table.  
    - Supabase2: getAll on `document_metadata` table.

25. **Create Merge nodes Merge and Merge2:**  
    - Merge Google Drive and Supabase results.

26. **Create Code nodes Code1 and Code2:**  
    - Code1: Identify orphaned document rows by comparing Drive IDs vs Supabase `documents`.  
    - Code2: Identify orphaned metadata rows similarly for `document_metadata`.

27. **Create Supabase delete nodes Delete Rows and Delete Rows2:**  
    - Delete Rows: deletes orphaned entries from `documents`.  
    - Delete Rows2: deletes orphaned entries from `document_metadata`.

28. **Connect nodes in cleanup chain:**  
    - Schedule Trigger → Get File IDs → Supabase → Merge → Code1 → Delete Rows  
    - Schedule Trigger → Get File IDs2 → Supabase2 → Merge2 → Code2 → Delete Rows2

---

29. **Set up all required credentials:**  
    - Google Drive OAuth2 (with appropriate scopes for file access)  
    - Supabase API credentials (for database and vector store access)  
    - Postgres credentials for chat memory (can be Supabase Postgres)  
    - Google Gemini API credentials for embeddings and chat model  
    - Cohere API key for reranker node

30. **Create Supabase tables:**  
    - Run SQL scripts from provided doc link to create `documents` and `document_metadata` tables.

31. **Replace placeholder 'Your Folder ID' with actual Google Drive folder ID** in all relevant nodes.

32. **Activate workflow:**  
    - Initially trigger manual ingestion to populate vector store.  
    - After verification, enable Google Drive triggers (fileCreated, fileUpdated) to automate ingestion.  
    - Optionally, enable schedule trigger for cleanup.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Comprehensive setup instructions, including credential connections and table creation, are detailed in a sticky note with a link to SQL schema: https://docs.google.com/document/d/1tLJ7fndrDjYMfyQ1R61q5NbqDs5ZMgvhzcMzkoIzj5M/edit?usp=sharing | Sticky Note8 |
| Postgres connection pooling must use “Transaction” pooler mode in Supabase as shown here: https://i.postimg.cc/htxnsprc/Screenshot-2025-10-02-143424.png | Sticky Note9 |
| Cohere reranker setup requires creating an account and copying the API key from https://dashboard.cohere.com/ | Sticky Note6 and Sticky Note10 |
| The RAG agent prompt can be fine-tuned to customize chatbot behavior | Sticky Note7 |
| Contact and branding info: Created by Anirudh Aeran. LinkedIn profile: https://www.linkedin.com/in/anirudh-narayan-a-/ | Sticky Note3 |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected content. All data processed is legal and public.