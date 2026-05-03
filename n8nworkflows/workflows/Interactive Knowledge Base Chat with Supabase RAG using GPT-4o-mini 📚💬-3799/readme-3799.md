Interactive Knowledge Base Chat with Supabase RAG using GPT-4o-mini 📚💬

https://n8nworkflows.xyz/workflows/interactive-knowledge-base-chat-with-supabase-rag-using-gpt-4o-mini------3799


# Interactive Knowledge Base Chat with Supabase RAG using GPT-4o-mini 📚💬

### 1. Workflow Overview

This combined n8n workflow suite automates two core processes for managing a company knowledge base:

- **Google Drive File Ingestion to Supabase for Knowledge Base**: Automatically detects new or updated files in Google Drive, extracts and processes their content (text or tabular), generates embeddings for text data, and stores all relevant data and metadata in Supabase tables. It handles duplicates, errors, and supports multiple file formats. This workflow forms the backend ingestion pipeline for building a searchable knowledge base.

- **Interactive Knowledge Base Chat with Supabase RAG using GPT-4o-mini**: Provides a chat interface for users to query the knowledge base using natural language. It leverages Retrieval-Augmented Generation (RAG) by retrieving relevant text chunks and tabular data from Supabase, then generates context-aware responses using OpenAI’s GPT-4o-mini model. It manages conversation history to maintain context and improve interaction quality.

---

The workflows are logically grouped as follows:

**Google Drive File Ingestion Workflow**

- 1.1 File Detection  
- 1.2 File Metadata Extraction and Validation  
- 1.3 Duplicate Detection and Handling  
- 1.4 Content Extraction (Text and Tabular)  
- 1.5 Embedding Generation and Storage  
- 1.6 Metadata and Knowledge Base Record Insertion  
- 1.7 Error Logging and Notifications

**Interactive Knowledge Base Chat Workflow**

- 2.1 Chat Interface Trigger  
- 2.2 Query Processing with RAG AI Agent  
- 2.3 Data Retrieval Tools (Text Chunks, Tabular Data, Full Document Text, File List)  
- 2.4 Conversation History Management  
- 2.5 Response Formatting and Delivery

---

### 2. Block-by-Block Analysis

---

#### 1. Google Drive File Ingestion Workflow

---

##### 1.1 File Detection

- **Overview**: Listens for new or updated files in Google Drive to trigger ingestion.

- **Nodes Involved**:  
  - File Created (Google Drive Trigger)  
  - Update to File (Google Drive Trigger, currently disconnected)  
  - Loop Over Items (SplitInBatches)

- **Node Details**:

1. **File Created**  
   - Type: Google Drive Trigger  
   - Role: Triggers workflow on new file creation in Google Drive  
   - Config: Uses Google Drive OAuth2 credentials; triggers on `File Created` event  
   - Inputs: None (trigger node)  
   - Outputs: List of new files  
   - Edge Cases: Permissions errors, API rate limits, folder monitoring misconfiguration  
   - Notes: Folder can be specified to limit scope

2. **Update to File**  
   - Type: Google Drive Trigger  
   - Role: Intended to trigger on file updates (currently disconnected)  
   - Config: Google Drive OAuth2; event `File Updated`  
   - Inputs: None (trigger node)  
   - Outputs: List of updated files  
   - Edge Cases: Disabled connection means updates are not processed  
   - Notes: Reconnect if update processing is required

3. **Loop Over Items**  
   - Type: SplitInBatches  
   - Role: Processes each detected file individually  
   - Config: Default batch size; input from trigger nodes  
   - Inputs: Array of files (`{{ $json.files }}`)  
   - Outputs: Single file per iteration  
   - Edge Cases: Large batch sizes may cause timeouts or memory issues

---

##### 1.2 File Metadata Extraction and Validation

- **Overview**: Extracts key metadata from each file and validates file type.

- **Nodes Involved**:  
  - Set File ID (Set)  
  - Validate File (IF)  
  - Check for Duplicates (Supabase Postgres)  

- **Node Details**:

1. **Set File ID**  
   - Type: Set  
   - Role: Extracts metadata fields: `file_id`, `file_name`, `mime_type`, `web_view_link`  
   - Config: Uses expressions like `{{ $json.id }}`, `{{ $json.name }}`, etc.  
   - Inputs: Single file item from Loop Over Items  
   - Outputs: Metadata object for downstream use  
   - Edge Cases: Missing or malformed metadata fields

2. **Validate File**  
   - Type: IF  
   - Role: Checks if `mime_type` matches supported types (PDF, DOCX, XLSX, CSV, TXT, RTF)  
   - Config: Condition on `mime_type` field containing supported MIME types  
   - Inputs: Metadata from Set File ID  
   - Outputs: True branch for supported files, false branch for unsupported  
   - Edge Cases: Unsupported file types trigger error logging

3. **Check for Duplicates**  
   - Type: Postgres (Supabase)  
   - Role: Queries `knowledge_base` table to check if `file_id` already exists  
   - Config: Select operation with filter `file_id = {{ $node['Set File ID'].json.file_id }}`  
   - Inputs: Metadata from Validate File (true branch)  
   - Outputs: Query result array  
   - Edge Cases: DB connection errors, query failures

---

##### 1.3 Duplicate Detection and Handling

- **Overview**: Routes workflow based on duplicate presence; deletes old data if duplicate found.

- **Nodes Involved**:  
  - IF Duplicate Check (IF)  
  - Log Duplicate (Supabase)  
  - Slack Duplicate Notification (Gmail)  
  - Debug File ID (Set)  
  - Delete old Doc (Supabase)  
  - Delete Old Data Rows (Supabase)  

- **Node Details**:

1. **IF Duplicate Check**  
   - Type: IF  
   - Role: Checks if duplicates found (`length > 0`)  
   - Inputs: Output of Check for Duplicates  
   - Outputs: True branch (duplicate found), false branch (no duplicate)  
   - Edge Cases: False positives/negatives if DB inconsistent

2. **Log Duplicate**  
   - Type: Supabase  
   - Role: Inserts duplicate error record into `error_log` table  
   - Inputs: From IF Duplicate Check (true branch)  
   - Edge Cases: DB insert failures

3. **Slack Duplicate Notification**  
   - Type: Gmail  
   - Role: Sends notification email about duplicate file  
   - Inputs: From Log Duplicate  
   - Edge Cases: Email sending failures, credential issues

4. **Debug File ID**  
   - Type: Set  
   - Role: For debugging, passes file ID downstream  
   - Inputs: From IF Duplicate Check (false branch)  
   - Outputs: File ID for deletion nodes

5. **Delete old Doc**  
   - Type: Supabase  
   - Role: Deletes old text data from `documents` table for duplicate file  
   - Inputs: From Debug File ID  
   - Config: Delete operation filtering on `metadata->>'file_id'`  
   - Edge Cases: Deletion failures, partial deletes

6. **Delete Old Data Rows**  
   - Type: Supabase  
   - Role: Deletes old tabular data from `document_rows` table for duplicate file  
   - Inputs: From Delete old Doc  
   - Edge Cases: Same as above

---

##### 1.4 Content Extraction (Text and Tabular)

- **Overview**: Downloads file content and extracts text or tabular data based on MIME type.

- **Nodes Involved**:  
  - Insert Metadata (Supabase)  
  - Download File (Google Drive)  
  - Switch (Switch node)  
  - Extract from File PDF  
  - Extract from TXT  
  - Extract from CSV  
  - Extract from XLSX  
  - Extract from RTF  
  - Extract from DOC  

- **Node Details**:

1. **Insert Metadata**  
   - Type: Supabase  
   - Role: Inserts basic file metadata into `document_metadata` table before content extraction  
   - Inputs: From Delete Old Data Rows or no duplicate path  
   - Edge Cases: Insert failures

2. **Download File**  
   - Type: Google Drive  
   - Role: Downloads file binary content for extraction  
   - Config: Uses Google Drive OAuth2; file ID from metadata  
   - Edge Cases: Download failures, permission errors

3. **Switch**  
   - Type: Switch  
   - Role: Routes file processing based on `mime_type` to appropriate extraction node  
   - Inputs: Download File output  
   - Branches: PDF, TXT, CSV, XLSX, RTF, DOC  

4. **Extract from File PDF**  
   - Type: Extract from File  
   - Role: Extracts text from PDF binary content  
   - Config: Uses binary data from Download File  
   - Edge Cases: Extraction accuracy, corrupted PDFs

5. **Extract from TXT**  
   - Type: Extract from File  
   - Role: Extracts plain text from TXT files  
   - Edge Cases: Encoding issues

6. **Extract from CSV**  
   - Type: Extract from File  
   - Role: Parses CSV content into structured rows  
   - Edge Cases: Malformed CSV, delimiter issues

7. **Extract from XLSX**  
   - Type: Extract from File  
   - Role: Parses Excel spreadsheets into structured rows  
   - Edge Cases: Complex sheets, formulas

8. **Extract from RTF**  
   - Type: Extract from File  
   - Role: Extracts text from RTF files  
   - Edge Cases: Formatting issues

9. **Extract from DOC**  
   - Type: Extract from File  
   - Role: Extracts text from DOC files  
   - Edge Cases: Legacy format issues

---

##### 1.5 Embedding Generation and Storage

- **Overview**: Processes extracted text into chunks, generates embeddings, and stores them; stores tabular data rows.

- **Nodes Involved**:  
  - Aggregate (Aggregate)  
  - Summarize (Summarize)  
  - Embeddings OpenAI (OpenAI Embeddings)  
  - Supabase Vector Store (Vector Store Insert)  
  - Insert Table Rows (Supabase)  
  - Character Text Splitter (Text Splitter)  
  - Default Data Loader (Document Loader)  
  - Set Schema (Set)  
  - Schema Document Metadata (Supabase)

- **Node Details**:

1. **Aggregate**  
   - Type: Aggregate  
   - Role: Aggregates extracted tabular rows for batch insertion  
   - Inputs: Extract from CSV or XLSX  
   - Edge Cases: Large datasets causing memory issues

2. **Summarize**  
   - Type: Summarize  
   - Role: Optional summarization of text chunks before embedding  
   - Inputs: Aggregate output or extracted text  
   - Edge Cases: Summarization quality

3. **Character Text Splitter**  
   - Type: Text Splitter  
   - Role: Splits large text into chunks (default 1000 chars with 200 overlap) for embedding  
   - Inputs: Extracted text

4. **Embeddings OpenAI**  
   - Type: OpenAI Embeddings  
   - Role: Generates vector embeddings for text chunks using `text-embedding-ada-002`  
   - Config: OpenAI API key credential  
   - Inputs: Text chunks from splitter  
   - Edge Cases: API rate limits, embedding failures

5. **Supabase Vector Store**  
   - Type: Vector Store (Supabase)  
   - Role: Inserts text chunks and embeddings into `documents` table  
   - Config: Supabase credentials, insert operation  
   - Inputs: Embeddings and text chunks  
   - Edge Cases: DB insert errors

6. **Insert Table Rows**  
   - Type: Supabase  
   - Role: Inserts tabular data rows into `document_rows` table  
   - Inputs: Aggregated tabular data  
   - Edge Cases: Data validation failures

7. **Default Data Loader**  
   - Type: Document Loader  
   - Role: Loads documents for vector store insertion  
   - Inputs: Text chunks

8. **Set Schema**  
   - Type: Set  
   - Role: Sets schema metadata for document metadata insertion  
   - Inputs: After vector store insertion

9. **Schema Document Metadata**  
   - Type: Supabase  
   - Role: Inserts schema and metadata info into `document_metadata` table  
   - Inputs: From Set Schema

---

##### 1.6 Metadata and Knowledge Base Record Insertion

- **Overview**: Inserts file metadata and knowledge base records for search and management.

- **Nodes Involved**:  
  - Insert Metadata (Supabase)  
  - Schema Document Metadata (Supabase)

- **Node Details**:

- Covered above in Insert Metadata and Schema Document Metadata nodes.

---

##### 1.7 Error Logging and Notifications

- **Overview**: Logs errors for unsupported files or processing issues and sends notifications.

- **Nodes Involved**:  
  - Set Error Type (Set)  
  - Error Logger (Supabase)  
  - Error Notification (Gmail)  

- **Node Details**:

1. **Set Error Type**  
   - Type: Set  
   - Role: Sets error type and message for logging  
   - Inputs: From Validate File (false branch) or other error points

2. **Error Logger**  
   - Type: Supabase  
   - Role: Inserts error records into `error_log` table  
   - Inputs: From Set Error Type

3. **Error Notification**  
   - Type: Gmail  
   - Role: Sends email notifications for errors  
   - Inputs: From Error Logger

---

#### 2. Interactive Knowledge Base Chat Workflow

---

##### 2.1 Chat Interface Trigger

- **Overview**: Provides an interactive chat interface for users to input queries.

- **Nodes Involved**:  
  - When chat message received (Chat Trigger)

- **Node Details**:

1. **When chat message received**  
   - Type: Chat Trigger  
   - Role: Starts workflow on user chat input  
   - Config: Chat title, subtitle, welcome message, suggestions, outputs session ID and user message  
   - Edge Cases: Webhook connectivity, user input validation

---

##### 2.2 Query Processing with RAG AI Agent

- **Overview**: Processes user query using RAG, retrieving relevant data and generating responses.

- **Nodes Involved**:  
  - Edit Fields2 (Set)  
  - RAG AI Agent (AI Agent)  
  - OpenAI Chat Model (OpenAI Chat)  
  - Embeddings OpenAI2 (OpenAI Embeddings)  
  - Supabase Vector Store2 (Vector Store)  
  - Postgres Chat Memory (Chat Memory)  
  - List Documents (Supabase Tool)  
  - Query Document Rows (Supabase Tool)  
  - Get Full Document Text - Get File Contents (Supabase Tool)

- **Node Details**:

1. **Edit Fields2**  
   - Type: Set  
   - Role: Prepares input fields for AI Agent  
   - Inputs: From Chat Trigger

2. **RAG AI Agent**  
   - Type: AI Agent  
   - Role: Orchestrates RAG process using tools and OpenAI GPT-4o-mini  
   - Config: OpenAI API key, system prompt tailored for company knowledge base assistant  
   - Inputs: User message from Edit Fields2  
   - Outputs: Generated response  
   - Edge Cases: API errors, tool failures

3. **OpenAI Chat Model**  
   - Type: OpenAI Chat  
   - Role: Language model used by AI Agent  
   - Config: Model `gpt-4o-mini`  
   - Inputs: From RAG AI Agent

4. **Embeddings OpenAI2**  
   - Type: OpenAI Embeddings  
   - Role: Generates embeddings for query vector search  
   - Inputs: From RAG AI Agent

5. **Supabase Vector Store2**  
   - Type: Vector Store (Supabase)  
   - Role: Retrieves relevant text chunks from `documents` table  
   - Inputs: Embeddings from Embeddings OpenAI2

6. **Postgres Chat Memory**  
   - Type: Postgres Chat Memory  
   - Role: Stores and retrieves conversation history from `n8n_chat_history` table  
   - Inputs: Session ID from Chat Trigger

7. **List Documents**  
   - Type: Supabase Tool (Execute Select)  
   - Role: Lists all files in knowledge base from `document_metadata` table  
   - Inputs: From RAG AI Agent as tool

8. **Query Document Rows**  
   - Type: Supabase Tool (Execute Query)  
   - Role: Retrieves tabular data rows for a given file ID  
   - Inputs: From RAG AI Agent as tool

9. **Get Full Document Text - Get File Contents**  
   - Type: Supabase Tool (Execute Query)  
   - Role: Retrieves full concatenated text of a document  
   - Inputs: From RAG AI Agent as tool

---

##### 2.3 Data Retrieval Tools

- Covered above in List Documents, Query Document Rows, Get Full Document Text nodes.

---

##### 2.4 Conversation History Management

- Covered above in Postgres Chat Memory node.

---

##### 2.5 Response Formatting and Delivery

- **Nodes Involved**:  
  - None explicitly shown in JSON for final formatting; typically done in AI Agent or Set nodes downstream of AI Agent.

---

### 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                               | Input Node(s)                       | Output Node(s)                          | Sticky Note                              |
|--------------------------------|---------------------------------------|-----------------------------------------------|-----------------------------------|----------------------------------------|-----------------------------------------|
| File Created                   | Google Drive Trigger                   | Trigger on new file creation                   | None                              | Loop Over Items                        |                                         |
| Update to File                 | Google Drive Trigger                   | Trigger on file update (disconnected)         | None                              | Loop Over Items                        |                                         |
| Loop Over Items                | SplitInBatches                        | Process each file individually                  | File Created, Update to File      | Set File ID                           |                                         |
| Set File ID                   | Set                                   | Extract file metadata                           | Loop Over Items                   | Validate File                        |                                         |
| Validate File                 | IF                                    | Validate file MIME type                         | Set File ID                      | Check for Duplicates, Set Error Type |                                         |
| Check for Duplicates           | Postgres (Supabase)                   | Check if file already processed                 | Validate File                    | IF Duplicate Check                   |                                         |
| IF Duplicate Check             | IF                                    | Route based on duplicate presence               | Check for Duplicates             | Log Duplicate, Debug File ID          |                                         |
| Log Duplicate                 | Supabase                             | Log duplicate error                             | IF Duplicate Check (true)        | Slack Duplicate Notification          |                                         |
| Slack Duplicate Notification   | Gmail                                | Notify about duplicate file                     | Log Duplicate                   | None                               |                                         |
| Debug File ID                 | Set                                   | Pass file ID for deletion                       | IF Duplicate Check (false)       | Delete old Doc                      |                                         |
| Delete old Doc                | Supabase                             | Delete old text data for duplicate              | Debug File ID                   | Delete Old Data Rows                 |                                         |
| Delete Old Data Rows           | Supabase                             | Delete old tabular data for duplicate           | Delete old Doc                  | Insert Metadata                    |                                         |
| Insert Metadata               | Supabase                             | Insert file metadata                             | Delete Old Data Rows             | Download File                      |                                         |
| Download File                 | Google Drive                         | Download file content                            | Insert Metadata                 | Switch                            |                                         |
| Switch                       | Switch                              | Route by MIME type                               | Download File                   | Extract from File PDF, TXT, CSV, XLSX, RTF, DOC |                                         |
| Extract from File PDF          | Extract from File                    | Extract text from PDF                            | Switch                         | Supabase Vector Store              |                                         |
| Extract from TXT              | Extract from File                    | Extract text from TXT                            | Switch                         | Supabase Vector Store              |                                         |
| Extract from CSV              | Extract from File                    | Extract tabular data from CSV                    | Switch                         | Aggregate, Insert Table Rows       |                                         |
| Extract from XLSX             | Extract from File                    | Extract tabular data from XLSX                   | Switch                         | Aggregate, Insert Table Rows       |                                         |
| Extract from RTF              | Extract from File                    | Extract text from RTF                            | Switch                         | Supabase Vector Store              |                                         |
| Extract from DOC              | Extract from File                    | Extract text from DOC                            | Switch                         | Supabase Vector Store              |                                         |
| Aggregate                    | Aggregate                          | Aggregate tabular rows                            | Extract from CSV, XLSX          | Summarize, Insert Table Rows       |                                         |
| Summarize                   | Summarize                         | Optional summarization                            | Aggregate                      | Supabase Vector Store, Set Schema  |                                         |
| Character Text Splitter        | Text Splitter                      | Split text into chunks                            | Extracted text                 | Default Data Loader                |                                         |
| Embeddings OpenAI             | OpenAI Embeddings                  | Generate embeddings for text chunks              | Character Text Splitter         | Supabase Vector Store              |                                         |
| Supabase Vector Store         | Vector Store (Supabase)            | Store text chunks and embeddings                  | Embeddings OpenAI              | Loop Over Items                   |                                         |
| Default Data Loader           | Document Loader                   | Load documents for vector store                   | Character Text Splitter         | Supabase Vector Store              |                                         |
| Set Schema                   | Set                               | Set schema metadata                               | Supabase Vector Store          | Schema Document Metadata          |                                         |
| Schema Document Metadata      | Supabase                         | Insert document metadata                           | Set Schema                    | None                             |                                         |
| Set Error Type               | Set                               | Set error info for logging                         | Validate File (false branch)  | Error Logger                    |                                         |
| Error Logger                 | Supabase                         | Log errors                                         | Set Error Type                | Error Notification              |                                         |
| Error Notification           | Gmail                            | Notify on error                                    | Error Logger                  | None                             |                                         |
| Slack Duplicate Notification  | Gmail                            | Notify on duplicate                                | Log Duplicate                 | None                             |                                         |
| When chat message received    | Chat Trigger                     | Start chat interface on user message              | None                          | Edit Fields2                   |                                         |
| Edit Fields2                 | Set                              | Prepare input for AI Agent                         | When chat message received    | RAG AI Agent                  |                                         |
| RAG AI Agent                 | AI Agent                        | Process query with RAG and generate response       | Edit Fields2                 | None (final output)            |                                         |
| OpenAI Chat Model            | OpenAI Chat                    | Language model for AI Agent                         | RAG AI Agent                 | RAG AI Agent                  |                                         |
| Embeddings OpenAI2           | OpenAI Embeddings              | Generate embeddings for query                       | RAG AI Agent                 | Supabase Vector Store2         |                                         |
| Supabase Vector Store2       | Vector Store (Supabase)        | Retrieve relevant text chunks                        | Embeddings OpenAI2           | RAG AI Agent                  |                                         |
| Postgres Chat Memory         | Postgres Chat Memory           | Store and retrieve conversation history             | When chat message received    | RAG AI Agent                  |                                         |
| List Documents              | Supabase Tool (Select)          | List files in knowledge base                         | RAG AI Agent                 | RAG AI Agent                  |                                         |
| Query Document Rows          | Supabase Tool (Execute Query)  | Retrieve tabular data rows                            | RAG AI Agent                 | RAG AI Agent                  |                                         |
| Get Full Document Text - Get File Contents | Supabase Tool (Execute Query) | Retrieve full document text                       | RAG AI Agent                 | RAG AI Agent                  |                                         |

---

### 4. Reproducing the Workflow from Scratch

**Google Drive File Ingestion to Supabase**

1. **Create Google Drive Trigger node** named `File Created`  
   - Credential: Google Drive OAuth2  
   - Event: `File Created`  
   - Optionally specify folder to monitor

2. **(Optional) Create Google Drive Trigger node** named `Update to File`  
   - Credential: Google Drive OAuth2  
   - Event: `File Updated`  
   - Connect to Loop Over Items if update processing desired

3. **Add SplitInBatches node** named `Loop Over Items`  
   - Connect `File Created` and `Update to File` outputs to this node  
   - Default batch size (adjust if needed)

4. **Add Set node** named `Set File ID`  
   - Extract and set fields:  
     - `file_id` = `{{ $json.id }}`  
     - `file_name` = `{{ $json.name }}`  
     - `mime_type` = `{{ $json.mimeType }}`  
     - `web_view_link` = `{{ $json.webViewLink }}`  
   - Connect from `Loop Over Items`

5. **Add IF node** named `Validate File`  
   - Condition: Check if `mime_type` contains any supported MIME types (e.g., `application/pdf`, `text/plain`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `text/csv`, `application/rtf`, `application/msword`)  
   - Connect from `Set File ID`

6. **Add Postgres node** named `Check for Duplicates`  
   - Operation: Select  
   - Table: `knowledge_base`  
   - Filter: `file_id = {{ $node['Set File ID'].json.file_id }}`  
   - Connect from `Validate File` (true branch)

7. **Add IF node** named `IF Duplicate Check`  
   - Condition: `{{ $node['Check for Duplicates'].json.length > 0 }}`  
   - Connect from `Check for Duplicates`

8. **Add Supabase node** named `Log Duplicate`  
   - Operation: Insert into `error_log` with duplicate error info  
   - Connect from `IF Duplicate Check` (true branch)

9. **Add Gmail node** named `Slack Duplicate Notification`  
   - Send notification email about duplicate file  
   - Connect from `Log Duplicate`

10. **Add Set node** named `Debug File ID`  
    - Pass file ID downstream  
    - Connect from `IF Duplicate Check` (false branch)

11. **Add Supabase node** named `Delete old Doc`  
    - Operation: Delete from `documents` where `metadata->>'file_id' = {{ $node['Debug File ID'].json.file_id }}`  
    - Connect from `Debug File ID`

12. **Add Supabase node** named `Delete Old Data Rows`  
    - Operation: Delete from `document_rows` where `dataset_id = {{ $node['Debug File ID'].json.file_id }}`  
    - Connect from `Delete old Doc`

13. **Add Supabase node** named `Insert Metadata`  
    - Operation: Insert into `document_metadata` with file metadata fields  
    - Connect from `Delete Old Data Rows`

14. **Add Google Drive node** named `Download File`  
    - Credential: Google Drive OAuth2  
    - File ID: `{{ $node['Set File ID'].json.file_id }}`  
    - Connect from `Insert Metadata`

15. **Add Switch node** named `Switch`  
    - Route by `mime_type` to extraction nodes  
    - Connect from `Download File`

16. **Add Extract from File nodes** for each supported type:  
    - PDF, TXT, CSV, XLSX, RTF, DOC  
    - Configure to extract text or tabular data from downloaded file content  
    - Connect each from appropriate Switch branch

17. **Add Aggregate node** named `Aggregate`  
    - Aggregate tabular data rows from CSV/XLSX extraction  
    - Connect from Extract from CSV and Extract from XLSX

18. **Add Summarize node** named `Summarize` (optional)  
    - Summarize aggregated text if needed  
    - Connect from Aggregate

19. **Add Character Text Splitter node** named `Character Text Splitter`  
    - Chunk size: 1000 chars  
    - Overlap: 200 chars  
    - Connect from text extraction nodes or Summarize

20. **Add OpenAI Embeddings node** named `Embeddings OpenAI`  
    - Credential: OpenAI API key  
    - Model: `text-embedding-ada-002`  
    - Connect from Character Text Splitter

21. **Add Supabase Vector Store node** named `Supabase Vector Store`  
    - Operation: Insert Documents into `documents` table  
    - Connect from Embeddings OpenAI

22. **Add Postgres node** named `Insert Table Rows`  
    - Operation: Insert rows into `document_rows` table for tabular data  
    - Connect from Aggregate

23. **Add Set node** named `Set Schema`  
    - Prepare schema metadata for document metadata insertion  
    - Connect from Supabase Vector Store

24. **Add Postgres node** named `Schema Document Metadata`  
    - Insert schema and metadata info into `document_metadata` table  
    - Connect from Set Schema

25. **Add Set node** named `Set Error Type`  
    - Set error info for unsupported files  
    - Connect from Validate File (false branch)

26. **Add Supabase node** named `Error Logger`  
    - Insert error record into `error_log` table  
    - Connect from Set Error Type

27. **Add Gmail node** named `Error Notification`  
    - Send email notification for errors  
    - Connect from Error Logger

---

**Interactive Knowledge Base Chat Workflow**

1. **Add Chat Trigger node** named `When chat message received`  
   - Configure chat title, subtitle, welcome message, suggestions  
   - Enable output of session ID and user message

2. **Add Set node** named `Edit Fields2`  
   - Prepare input fields for AI Agent  
   - Connect from Chat Trigger

3. **Add AI Agent node** named `RAG AI Agent`  
   - Credential: OpenAI API key  
   - Model: `gpt-4o-mini`  
   - System prompt tailored for company knowledge base assistant  
   - Connect from Edit Fields2

4. **Add OpenAI Chat Model node** named `OpenAI Chat Model`  
   - Model: `gpt-4o-mini`  
   - Connect as AI language model for AI Agent

5. **Add OpenAI Embeddings node** named `Embeddings OpenAI2`  
   - Credential: OpenAI API key  
   - Connect as AI embedding for AI Agent

6. **Add Supabase Vector Store node** named `Supabase Vector Store2`  
   - Operation: Retrieve Documents from `documents` table  
   - Connect as AI tool for AI Agent

7. **Add Postgres Chat Memory node** named `Postgres Chat Memory`  
   - Credential: Supabase (Postgres)  
   - Table: `n8n_chat_history`  
   - Session ID: `{{ $node['When chat message received'].json.sessionId }}`  
   - Connect as AI memory for AI Agent

8. **Add Supabase Tool nodes** named `List Documents`, `Query Document Rows`, `Get Full Document Text - Get File Contents`  
   - Configure SQL queries as per description  
   - Connect as AI tools for AI Agent

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires n8n version 1.0 or higher with AI features enabled.                                                                                                                                                            | n8n official docs                                                                                |
| Supabase tables must be created with specified schemas including vector embedding support (`vector(1536)` type).                                                                                                                     | Supabase documentation                                                                          |
| The `match_documents` function is required in Supabase for vector similarity search. SQL function provided in workflow description.                                                                                                  | Supabase SQL function snippet in workflow description                                          |
| OpenAI API key with access to embedding and chat models is required; usage costs apply.                                                                                                                                               | OpenAI API documentation                                                                        |
| Gmail node used for notifications requires OAuth2 credentials with send email permission.                                                                                                                                             | Gmail API docs                                                                                  |
| Google Drive OAuth2 credentials must have access to monitored folders and files.                                                                                                                                                      | Google Drive API docs                                                                           |
| The workflow handles common file types but can be extended to support additional formats by adding extraction nodes and updating MIME type checks.                                                                                  | n8n extract from file node documentation                                                       |
| Conversation history is stored in Supabase to maintain context; ensure indexing on `session_id` and `timestamp` for performance.                                                                                                    | Supabase best practices                                                                         |
| For improved error handling, consider adding retries or alerts on node failures.                                                                                                                                                      | n8n error handling best practices                                                              |
| The chat interface can be customized with branding, welcome messages, and suggested queries to improve user experience.                                                                                                             | n8n Chat Trigger node documentation                                                            |
| Slack duplicate notifications are sent via Gmail node; adapt to your preferred notification system if needed.                                                                                                                       | Gmail node configuration                                                                        |
| The workflow is designed to be modular; ingestion and chat workflows can be deployed separately or combined as needed.                                                                                                              | Workflow design best practices                                                                 |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and extending the Google Drive ingestion and interactive chat workflows for a Supabase-backed company knowledge base using n8n and OpenAI.