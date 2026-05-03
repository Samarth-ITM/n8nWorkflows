Build Multi-Client Agentic RAG Document Processing Pipeline with Supabase Vector DB

https://n8nworkflows.xyz/workflows/build-multi-client-agentic-rag-document-processing-pipeline-with-supabase-vector-db-8916


# Build Multi-Client Agentic RAG Document Processing Pipeline with Supabase Vector DB

---

# Comprehensive Reference for Workflow:  
**Build Multi-Client Agentic RAG Document Processing Pipeline with Supabase Vector DB**

---

### 1. Workflow Overview

This workflow implements an automated, multi-client document ingestion, processing, and vector storage pipeline designed to enable Retrieval-Augmented Generation (RAG) with client-specific data isolation using Supabase’s PostgreSQL vector database. It handles real-time monitoring of client Google Drive folders, supports multiple document formats (PDF, Google Docs, Excel, CSV), extracts and processes content, generates semantic vector embeddings with OpenAI, and stores processed data in dedicated client vector stores and metadata tables.

The workflow is logically divided into six main phases:

- **1.1 Client-Specific Database Infrastructure Creation:**  
  Establishes isolated PostgreSQL tables per client with vector support and custom search functions.

- **1.2 Google Drive Folder Monitoring Configuration:**  
  Monitors client-specific Google Drive folders for new or updated documents triggering processing.

- **1.3 Document Processing and Content Extraction:**  
  Downloads documents, identifies file types, and extracts text or tabular data accordingly.

- **1.4 Data Aggregation and Schema Management:**  
  Aggregates extracted tabular data, summarizes content, manages metadata and database schema updates.

- **1.5 Advanced Vector Embedding and Text Processing:**  
  Splits texts, generates OpenAI embeddings, and prepares documents for vector storage.

- **1.6 Client-Specific Vector Database Storage and Workflow Completion:**  
  Inserts processed vectors and metadata into client-dedicated Supabase tables to complete the cycle.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Client-Specific Database Infrastructure Creation

**Overview:**  
This block initializes the client-specific database environment upon receiving a client identifier via chat. It creates isolated PostgreSQL tables for documents, metadata, and tabular rows, including a custom vector similarity search function, enabling secure, segregated vector storage for each client.

**Nodes Involved:**  
- When chat message received  
- Create Documents Table and Match Function1  
- Create Document Metadata Table1  
- Create Document Rows Table (for Tabular Data)1  
- Sticky Note (Phase 1 and detailed explanation)

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Entry point receiving client name/id via chat input.  
  - *Config:* Uses webhook ID for external invocation.  
  - *Inputs:* External chat message containing client name.  
  - *Outputs:* Passes client name for table creation.  
  - *Failures:* Invalid client name, improper formatting can cause DB errors.  
  - *Notes:* Client name must comply with SQL table naming rules (no spaces/special chars).

- **Create Documents Table and Match Function1**  
  - *Type:* Postgres node (execute query)  
  - *Role:* Creates `[client_name]_documents` table with embedding vector column, sets up index and a `match_[client_name]_documents` search function.  
  - *Config:* Executes dynamic SQL with client name substitution.  
  - *Inputs:* Client name from chat trigger.  
  - *Outputs:* Triggers metadata table creation.  
  - *Failures:* SQL syntax errors if client name invalid; permission issues.  
  - *Notes:* Uses pgvector extension; require PostgreSQL version supporting vector types.

- **Create Document Metadata Table1**  
  - *Type:* Postgres node (execute query)  
  - *Role:* Creates `[client_name]_document_metadata` table to track documents.  
  - *Config:* SQL CREATE TABLE with fields id, title, url, created_at, schema.  
  - *Inputs:* Client name from previous node.  
  - *Outputs:* Triggers creation of document rows table.  
  - *Failures:* Same as above; table name collision if reused client name.  

- **Create Document Rows Table (for Tabular Data)1**  
  - *Type:* Postgres node (execute query)  
  - *Role:* Creates `[client_name]_document_rows` table for storing tabular data rows.  
  - *Config:* References metadata table id as foreign key; stores JSONB row_data.  
  - *Inputs:* Client name.  
  - *Failures:* Foreign key constraint issues if metadata table missing.  

- **Sticky Notes (Phase 1 and detailed explanation)**  
  - Provide context on client initialization, database table creation, and multi-tenant isolation.  
  - Emphasize naming conventions, data separation, and multi-format support.

---

#### 1.2 Google Drive Folder Monitoring Configuration

**Overview:**  
Monitors a specific Google Drive folder for each client, triggering the workflow on file creation or update events. This ensures real-time ingestion of new or changed documents.

**Nodes Involved:**  
- File Created (Google Drive Trigger)  
- File Updated (Google Drive Trigger)  
- Loop Over Items (SplitInBatches)  
- Sticky Notes (Phase 2 and detailed instructions)

**Node Details:**

- **File Created**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches client-specific folder for newly created files.  
  - *Config:* Polls every minute, restricted to a specific folder URL.  
  - *Credentials:* Google Drive OAuth2 (client-specific).  
  - *Outputs:* Initiates batch processing of new files.  
  - *Failures:* API permission errors, folder URL misconfiguration.

- **File Updated**  
  - *Type:* Google Drive Trigger  
  - *Role:* Watches same folder for file updates.  
  - *Config:* Same as File Created but for updates.  
  - *Outputs:* Same batch processing node.  
  - *Failures:* Same as above.

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes files one by one to maintain isolation.  
  - *Config:* No reset between batches, ensuring continuous processing.  
  - *Inputs:* Files from triggers.  
  - *Outputs:* Passes individual file data downstream.  

- **Sticky Notes (Phase 2 and detailed instructions)**  
  - Explain configuration for folder URL monitoring, update requirements, and permissions.  
  - Emphasize dual triggers for comprehensive coverage.

---

#### 1.3 Document Processing and Content Extraction

**Overview:**  
Downloads the triggered files, detects file types, and routes files to appropriate extraction nodes for text or tabular content extraction.

**Nodes Involved:**  
- Insert Document Metadata (Postgres)  
- Download File (Google Drive)  
- Switch (type-based routing)  
- Extract PDF Text  
- Extract Document Text (for Google Docs)  
- Extract from Excel  
- Extract from CSV  
- Sticky Notes (Phase 3 and explanations)

**Node Details:**

- **Insert Document Metadata**  
  - *Type:* Postgres Upsert  
  - *Role:* Inserts metadata (file id, url, title) into `document_metadata` table.  
  - *Inputs:* File info from Set File ID node.  
  - *Outputs:* Triggers file download.  

- **Download File**  
  - *Type:* Google Drive node (download operation)  
  - *Role:* Retrieves file content from Drive, converts Google Docs to plain text.  
  - *Inputs:* File ID from Set File ID.  
  - *Outputs:* Passes file content for type routing.  
  - *Failures:* Download errors, file access denied.  

- **Switch**  
  - *Type:* Switch (conditional routing)  
  - *Role:* Routes files based on MIME type to appropriate extraction nodes.  
  - *Cases:* application/pdf → Extract PDF Text  
              application/vnd.openxmlformats-officedocument.spreadsheetml.sheet → Extract from Excel  
              application/vnd.google-apps.spreadsheet → Extract from Excel  
              application/vnd.google-apps.document → Extract Document Text  
              Fallback → Extract from CSV  

- **Extract PDF Text**  
  - *Type:* ExtractFromFile node (pdf operation)  
  - *Role:* Extracts text preserving PDF structure.  

- **Extract Document Text**  
  - *Type:* ExtractFromFile node (text operation)  
  - *Role:* Extracts plain text from Google Docs files.  

- **Extract from Excel**  
  - *Type:* ExtractFromFile node (xlsx operation)  
  - *Role:* Extracts structured spreadsheet data.  

- **Extract from CSV**  
  - *Type:* ExtractFromFile node (default operation)  
  - *Role:* Extracts CSV data as structured rows.  

- **Sticky Notes (Phase 3 and detailed explanation)**  
  - Describe multi-format extraction capabilities and error handling.

---

#### 1.4 Data Aggregation and Schema Management

**Overview:**  
Aggregates data extracted from tabular files (Excel/CSV), summarizes content, inserts individual rows into database, and updates document metadata with schema details.

**Nodes Involved:**  
- Aggregate  
- Summarize  
- Set Schema  
- Update Schema for Document Metadata (Postgres upsert)  
- Insert Table Rows (Postgres insert)  
- Sticky Notes (Phase 4 and explanations)

**Node Details:**

- **Aggregate**  
  - *Type:* Aggregate node  
  - *Role:* Combines all extracted data items into a single aggregated structure.  

- **Summarize**  
  - *Type:* Summarize node  
  - *Role:* Concatenates aggregated data field values into a summary text for embeddings.  

- **Set Schema**  
  - *Type:* Set node  
  - *Role:* Determines the schema of the data based on keys from extracted Excel or CSV data.  
  - *Logic:* Uses condition to check which extractor ran and extracts keys accordingly.

- **Update Schema for Document Metadata**  
  - *Type:* Postgres upsert  
  - *Role:* Updates the `schema` field in document metadata with detected column structure JSON string.  

- **Insert Table Rows**  
  - *Type:* Postgres insert  
  - *Role:* Inserts individual data rows into `[client_name]_document_rows` table, preserving structured data.  
  - *Input:* Extracted rows from Excel or CSV.  

- **Sticky Notes (Phase 4 explanations)**  
  - Explain purpose of schema detection, aggregation, row preservation, and metadata updates.

---

#### 1.5 Advanced Vector Embedding and Text Processing

**Overview:**  
Splits aggregated text into chunks, generates semantic vector embeddings using OpenAI’s text-embedding-3-small model, and prepares documents with metadata for vector storage.

**Nodes Involved:**  
- Character Text Splitter (LangChain)  
- Default Data Loader (LangChain)  
- Embeddings OpenAI1 (LangChain)  
- Sticky Notes (Phase 5 and detailed explanation)

**Node Details:**

- **Character Text Splitter**  
  - *Type:* LangChain Text Splitter  
  - *Role:* Splits long texts into manageable chunks based on character count to preserve context.  

- **Default Data Loader**  
  - *Type:* LangChain Document Loader  
  - *Role:* Loads split text chunks and attaches metadata such as file_id and file_title for client identification.  

- **Embeddings OpenAI1**  
  - *Type:* LangChain OpenAI Embeddings node  
  - *Role:* Generates 1536-dimensional vector embeddings using model `text-embedding-3-small`.  
  - *Credentials:* OpenAI API key configured.  
  - *Edge Cases:* API rate limits, network issues, invalid API key.  

- **Sticky Notes (Phase 5 explanations)**  
  - Describe embedding generation, metadata attribution, and text splitting rationale.

---

#### 1.6 Client-Specific Vector Database Storage and Workflow Completion

**Overview:**  
Inserts processed document vectors and metadata into client-specific Supabase vector tables, completing the processing cycle and enabling semantic search capabilities.

**Nodes Involved:**  
- Insert into Supabase Vectorstore (LangChain)  
- Delete Old Doc Rows (Supabase delete)  
- Delete Old Data Rows (Supabase delete)  
- Sticky Notes (Phase 6 and detailed instructions)

**Node Details:**

- **Insert into Supabase Vectorstore**  
  - *Type:* LangChain VectorStore Supabase node  
  - *Role:* Inserts new vector embeddings and associated metadata into `[client_name]_documents` table in Supabase.  
  - *Config:* Mode set to "insert", uses client-specific table name dynamically.  
  - *Credentials:* Supabase API key configured.  
  - *Failures:* Table not found, insert conflicts, API failures.  
  - *Notes:* Critical to update table name per client for data isolation.

- **Delete Old Doc Rows**  
  - *Type:* Supabase node (delete operation)  
  - *Role:* Deletes previous document rows linked to the file to prevent duplicates before inserting new rows.  
  - *Filter:* Deletes where metadata file_id matches current file id.  

- **Delete Old Data Rows**  
  - *Type:* Supabase node (delete operation)  
  - *Role:* Deletes old tabular data rows associated with the file from `document_rows` table.  

- **Sticky Notes (Phase 6 explanations)**  
  - Emphasize importance of client-specific table referencing and completion of processing cycle.

---

### 3. Summary Table

| Node Name                              | Node Type                                   | Functional Role                                           | Input Node(s)                          | Output Node(s)                        | Sticky Note                                                                                             |
|--------------------------------------|---------------------------------------------|----------------------------------------------------------|--------------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------|
| When chat message received            | LangChain Chat Trigger                      | Receives client name to initiate DB creation             | —                                    | Create Documents Table and Match Function1 | Phase 1: Client-Specific Database Infrastructure Creation                                              |
| Create Documents Table and Match Function1 | Postgres Execute Query                      | Creates client vector document table and search function | When chat message received           | Create Document Metadata Table1     | Phase 1 detailed explanation                                                                           |
| Create Document Metadata Table1       | Postgres Execute Query                      | Creates client document metadata table                    | Create Documents Table and Match Function1 | Create Document Rows Table (for Tabular Data)1 | Phase 1 detailed explanation                                                                           |
| Create Document Rows Table (for Tabular Data)1 | Postgres Execute Query                      | Creates client table for tabular data rows                | Create Document Metadata Table1      | —                                   | Phase 1 detailed explanation                                                                           |
| File Created                         | Google Drive Trigger                        | Triggers workflow on new files                            | —                                    | Loop Over Items                     | Phase 2: Google Drive Folder Monitoring Configuration                                                 |
| File Updated                        | Google Drive Trigger                        | Triggers workflow on file updates                         | —                                    | Loop Over Items                     | Phase 2 detailed explanation                                                                           |
| Loop Over Items                     | SplitInBatches                             | Splits file list for individual processing                | File Created, File Updated           | Set File ID (on second output)      | Phase 2 detailed explanation                                                                           |
| Set File ID                         | Set Node                                   | Assigns file metadata variables                            | Loop Over Items                      | Delete Old Doc Rows                 |                                                                                                       |
| Delete Old Doc Rows                 | Supabase Delete                            | Deletes old document rows linked to file                  | Set File ID                         | Delete Old Data Rows                |                                                                                                       |
| Delete Old Data Rows                | Supabase Delete                            | Deletes old tabular data rows                              | Delete Old Doc Rows                  | Insert Document Metadata            |                                                                                                       |
| Insert Document Metadata            | Postgres Upsert                            | Inserts/upserts document metadata                          | Delete Old Data Rows                 | Download File                      |                                                                                                       |
| Download File                      | Google Drive                               | Downloads file content and converts Google Docs           | Insert Document Metadata             | Switch                           | Phase 3: Document Processing and Content Extraction                                                   |
| Switch                            | Switch                                    | Routes files to extraction nodes based on MIME type        | Download File                      | Extract PDF Text, Extract from Excel, Extract from CSV, Extract Document Text |                                                                                                       |
| Extract PDF Text                   | ExtractFromFile (pdf)                      | Extracts text from PDFs                                   | Switch (PDF case)                   | Insert into Supabase Vectorstore   | Phase 3 detailed explanation                                                                           |
| Extract from Excel                 | ExtractFromFile (xlsx)                     | Extracts structured data from Excel                       | Switch (Excel case)                 | Aggregate, Insert Table Rows       | Phase 3 detailed explanation                                                                           |
| Extract from CSV                  | ExtractFromFile                            | Extracts structured data from CSV                         | Switch (CSV fallback)               | Aggregate, Insert Table Rows       | Phase 3 detailed explanation                                                                           |
| Extract Document Text             | ExtractFromFile (text)                     | Extracts text from Google Docs                            | Switch (Google Docs case)           | Insert into Supabase Vectorstore   | Phase 3 detailed explanation                                                                           |
| Aggregate                        | Aggregate                                 | Aggregates extracted tabular data                         | Extract from Excel, Extract from CSV | Summarize                        | Phase 4: Data Aggregation and Schema Management                                                       |
| Summarize                       | Summarize                                 | Concatenates aggregated data for embedding               | Aggregate                          | Set Schema, Insert into Supabase Vectorstore | Phase 4 detailed explanation                                                                           |
| Set Schema                      | Set                                       | Extracts and sets schema metadata                         | Summarize                         | Update Schema for Document Metadata | Phase 4 detailed explanation                                                                           |
| Update Schema for Document Metadata | Postgres Upsert                            | Updates document metadata with detected schema            | Set Schema                        | —                                 | Phase 4 detailed explanation                                                                           |
| Insert Table Rows               | Postgres Insert                            | Inserts individual tabular data rows                      | Extract from Excel, Extract from CSV | —                                 | Phase 4 detailed explanation                                                                           |
| Character Text Splitter         | LangChain Text Splitter                    | Splits text into chunks for embedding generation          | Summarize                        | Default Data Loader               | Phase 5: Advanced Vector Embedding and Text Processing                                               |
| Default Data Loader            | LangChain Document Loader                  | Loads chunks with metadata for embedding                  | Character Text Splitter           | Insert into Supabase Vectorstore  | Phase 5 detailed explanation                                                                           |
| Embeddings OpenAI1             | LangChain OpenAI Embeddings                | Generates vector embeddings                               | Default Data Loader              | Insert into Supabase Vectorstore  | Phase 5 detailed explanation                                                                           |
| Insert into Supabase Vectorstore | LangChain VectorStore Supabase             | Inserts embeddings and metadata into client vector table  | Extract PDF Text, Extract Document Text, Summarize, Embeddings OpenAI1 | Loop Over Items                   | Phase 6: Client-Specific Vector Database Storage and Workflow Completion                             |
| Sticky Notes (various phases)   | Sticky Note                                | Provide detailed explanations and instructions            | —                                | —                                 | Includes step-by-step phase descriptions, configuration notes, and critical reminders               |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up credentials:**  
   - Configure Google Drive OAuth2 API credentials for file access.  
   - Configure OpenAI API credentials for embedding generation.  
   - Configure Supabase API credentials for vector database operations.  
   - Configure PostgreSQL credentials for client metadata and rows tables.

2. **Create chat trigger node:**  
   - Add a LangChain Chat Trigger node.  
   - Configure webhook ID.  
   - Configure to receive client name or identifier.

3. **Create client-specific tables and functions:**  
   - Add Postgres Execute Query nodes to:  
     - Create `[client_name]_documents` table with embedding vector column and index.  
     - Create custom vector similarity search function `match_[client_name]_documents()`.  
     - Create `[client_name]_document_metadata` table.  
     - Create `[client_name]_document_rows` table with foreign key to metadata.

4. **Configure Google Drive triggers:**  
   - Add two Google Drive Trigger nodes: one for `fileCreated`, one for `fileUpdated`.  
   - Set both to monitor the client-specific folder URL.  
   - Set polling interval to every minute.  
   - Connect both to a SplitInBatches node to process files individually.

5. **Set file metadata extraction:**  
   - Add a Set node to extract and store file metadata fields: file_id, file_type, file_title, file_url.

6. **Delete old document rows:**  
   - Add Supabase Delete node to remove old document rows matching `file_id` in metadata JSON.  
   - Chain to another Supabase Delete node to remove old tabular data rows based on dataset_id.

7. **Insert document metadata:**  
   - Add Postgres Upsert node to insert or update document metadata (id, url, title).

8. **Download file content:**  
   - Add Google Drive node with download operation.  
   - Configure to convert Google Docs to plain text format.

9. **Switch based on file type:**  
   - Add Switch node checking `file_type` MIME string.  
   - Route:  
     - PDFs → Extract PDF Text (ExtractFromFile node with pdf operation)  
     - Excel/Google Sheets → Extract from Excel (ExtractFromFile with xlsx operation)  
     - Google Docs → Extract Document Text (ExtractFromFile with text operation)  
     - Fallback → Extract from CSV (ExtractFromFile default operation)

10. **Process extracted tabular data:**  
    - For Excel and CSV paths, connect to Aggregate node to combine data.  
    - Connect Aggregate to Summarize node, which concatenates data fields.  
    - Connect Summarize to Set node to extract schema from keys of first row.  
    - Connect Set node to Postgres Upsert node to update schema in document metadata.  
    - Insert Table Rows node to insert individual rows into `[client_name]_document_rows`.

11. **Text splitting and embedding generation:**  
    - Connect Summarize node output to LangChain Character Text Splitter node.  
    - Connect splitter to Default Data Loader node, adding metadata tags (file_id, file_title).  
    - Connect Default Data Loader to LangChain Embeddings OpenAI node with model `text-embedding-3-small`.

12. **Insert vectors into Supabase vector storage:**  
    - Add LangChain VectorStore Supabase node set to “insert” mode.  
    - Ensure table name is dynamically set to `[client_name]_documents`.  
    - Connect outputs from all text extraction paths and embedding nodes to this node.

13. **Loop and cleanup:**  
    - Connect Insert into Supabase Vectorstore node output back to SplitInBatches node for continued processing.  
    - Ensure deletion nodes execute before insertions to prevent duplicates.

14. **Add sticky notes for clarity:**  
    - Add Sticky Note nodes summarizing each phase:  
      - Client DB Setup  
      - Drive Folder Monitoring  
      - Document Processing  
      - Data Aggregation  
      - Embeddings  
      - Vector Storage and Completion

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                          | Context or Link                                                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Need more advanced automation solutions? Contact us for custom enterprise workflows! Growth-AI.fr<br>LinkedIn: https://www.linkedin.com/in/allanvaccarizi/<br>LinkedIn: https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/ | Project credits and contact for enterprise consultation                                                                                                                             |
| This workflow requires PostgreSQL with pgvector extension enabled for vector storage and similarity search. Ensure database supports vector data types.                                                              | PostgreSQL database requirement                                                                                                                                                      |
| Google Drive API needs permissions for Drive file monitoring and download. Folder URL must be updated per client folder.                                                                                              | Google Drive API configuration note                                                                                                                                                  |
| OpenAI API usage limited by rate and token limits. Monitor usage for embedding generation to avoid throttling.                                                                                                          | OpenAI API usage considerations                                                                                                                                                      |
| Client names used as table name suffixes must comply with SQL naming conventions and avoid reserved keywords.                                                                                                          | Naming convention best practice                                                                                                                                                       |
| Vector embedding dimension currently set for OpenAI `text-embedding-3-small` model (1536 dimensions). Modify if embedding model changes.                                                                               | Embedding model version dependency                                                                                                                                                   |
| The workflow supports multi-format document ingestion: PDFs, Google Docs, Excel, CSV, Google Sheets.                                                                                                                   | Multi-format support overview                                                                                                                                                         |
| The workflow features multi-tenant data isolation per client via dedicated database tables and vector stores.                                                                                                         | Multi-tenancy architecture note                                                                                                                                                       |
| Ensure that the "Insert into Supabase Vectorstore" node's table name is updated dynamically per client to maintain data integrity and isolation.                                                                       | Critical configuration reminder                                                                                                                                                       |
| The workflow uses LangChain nodes integrated in n8n for advanced AI processing tasks including embeddings and document loading.                                                                                       | LangChain node integration                                                                                                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. The content respects all current policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---