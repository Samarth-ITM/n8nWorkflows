Smarter RAG Agents with Enriched Retrieval and Modular Workflows

https://n8nworkflows.xyz/workflows/smarter-rag-agents-with-enriched-retrieval-and-modular-workflows-8008


# Smarter RAG Agents with Enriched Retrieval and Modular Workflows

---

### 1. Workflow Overview

This workflow, titled **“Smarter RAG Agents with Enriched Retrieval and Modular Workflows”**, is designed to implement a robust Retrieval-Augmented Generation (RAG) system. It enables ingestion, enrichment, storage, and retrieval of document content, combined with a conversational AI agent that answers user queries based strictly on indexed documents. The system leverages advanced language models (Google Gemini), vector embeddings, semantic metadata enrichment, and integrates with Supabase as a vector store and Postgres for chat memory.

The workflow is logically divided into three main functional blocks:

- **1.1 File Ingestion Pipeline:** Handles PDF uploads, extracts text, splits into chunks, computes embeddings, and stores them in Supabase vector DB with initial metadata.

- **1.2 Enrichment Pipeline (Asynchronous):** Periodically retrieves unprocessed documents, enriches them with semantic metadata via an LLM, and updates the database to improve retrieval precision and filtering capabilities.

- **1.3 RAG Chat Agent Pipeline:** Manages user interactions via chat triggers, constructs semantic queries, retrieves relevant document fragments with filters and reranking, maintains chat memory, and produces grounded answers with references.

Each block is modular, enabling independent scaling and maintenance, while collectively delivering a production-grade, explainable, and safe RAG assistant.

---

### 2. Block-by-Block Analysis

---

#### 2.1 File Ingestion Pipeline

- **Overview:**  
  This block automates the ingestion of uploaded PDF files, extracting their text content, splitting it into manageable chunks, embedding those chunks with semantic vectors (Google Gemini embeddings), and storing them in a Supabase vector database. It also cleans up old entries related to the same document to prevent duplication.

- **Nodes Involved:**  
  - On form submission  
  - Extract from File  
  - Set File Data  
  - Delete Old Doc Rows  
  - Merge  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Embeddings Google Gemini1  
  - Insert into Supabase Vectorstore  
  - Sticky Note, Sticky Note1, Sticky Note (File Ingestion descriptive notes)

- **Node Details:**

  1. **On form submission**  
     - Type: Form Trigger  
     - Role: Entry point for PDF uploads via a web form with a single required PDF file field.  
     - Configuration: Accepts only `.pdf` files, single file upload.  
     - Inputs: User-uploaded file  
     - Outputs: Binary PDF data forwarded to extraction node  
     - Failures: File upload errors, unsupported file types  

  2. **Extract from File**  
     - Type: Extract from File (PDF)  
     - Role: Extracts full text content from uploaded PDF binary.  
     - Configuration: Operation set to PDF extraction, input binary property is "File".  
     - Inputs: Binary PDF from form trigger  
     - Outputs: Extracted text content in JSON field `.text`  
     - Failures: Corrupted PDFs, extraction timeout  

  3. **Set File Data**  
     - Type: Set node  
     - Role: Assembles metadata about the document (doc_id, title, type, source) from extracted text and upload info.  
     - Config:  
       - doc_id: filename of uploaded PDF  
       - doc_title: extracted title or filename fallback  
       - doc_type: fixed string "guide"  
       - source: fixed string "uploaded_pdf"  
     - Inputs: Extracted text JSON  
     - Outputs: JSON enriched with metadata for downstream processing  

  4. **Delete Old Doc Rows**  
     - Type: Supabase (Delete operation)  
     - Role: Cleans previous entries in the `documents` table matching the same `doc_id` to avoid duplicates.  
     - Config: SQL filter on metadata JSON field `doc_id` equals uploaded filename.  
     - Inputs: Metadata from Set File Data  
     - Outputs: Confirmation of deletion  
     - Failures: DB connectivity, permission errors  

  5. **Merge**  
     - Type: Merge node (chooseBranch mode)  
     - Role: Synchronizes parallel branches: one for deleting old rows, one for new document ingestion  
     - Inputs: From Delete Old Doc Rows and Recursive Text Splitter  
     - Outputs: Unified flow to insertion nodes  

  6. **Recursive Character Text Splitter**  
     - Type: Text Splitter (recursive character)  
     - Role: Splits extracted document text into chunks of 1500 characters with 150 overlap for semantic embedding.  
     - Config: chunkSize=1500, chunkOverlap=150  
     - Inputs: Extracted text JSON from Extract from File  
     - Outputs: Array of text chunks for embedding  

  7. **Default Data Loader**  
     - Type: LangChain Default Data Loader  
     - Role: Converts text chunks to LangChain documents with associated metadata.  
     - Config: Metadata values passed from Set File Data; `processed` flag set to false initially.  
     - Inputs: Text chunks from Recursive Text Splitter  
     - Outputs: LangChain document objects for embedding insertion  

  8. **Embeddings Google Gemini1**  
     - Type: Google Gemini Embeddings  
     - Role: Generates 768-dimensional vector embeddings for the text chunks.  
     - Config: Uses Google Gemini (PaLM) API credentials.  
     - Inputs: Document chunks from Default Data Loader  
     - Outputs: Embedded vectors with documents  

  9. **Insert into Supabase Vectorstore**  
     - Type: Supabase Vector Store (Insert mode)  
     - Role: Inserts document chunks with embeddings and metadata into Supabase `documents` table.  
     - Config: Table name "documents", query name "match_documents" for similarity search.  
     - Inputs: Embeddings from Google Gemini1 node  
     - Outputs: Confirmation of insertion  
     - Failures: DB write errors, vector dimension mismatch  

- **Edge Cases & Failures:**  
  - PDF extraction failures due to corrupt or scanned documents  
  - Supabase connectivity or permissions issues  
  - Mismatch in embedding dimensions (not matching DB schema)  
  - Duplicate document overwrite delays  

- **Sticky Notes:**  
  - Explains the pipeline extracts, chunks, embeds, and stores PDF content in a modular, clean way for extensibility.

---

#### 2.2 Enrichment Pipeline (Asynchronous)

- **Overview:**  
  This block enriches unprocessed text chunks asynchronously by running a lightweight LLM (Google Gemini) to generate semantic metadata such as topics, features, use cases, audiences, and risk factors. It updates the database rows with this enriched metadata to improve semantic search filtering and retrieval.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get many rows (Supabase getAll)  
  - Loop Over Items (splitInBatches)  
  - Edit Fields (Set node)  
  - Metadata Obtention (Google Gemini LLM)  
  - Update a row (Supabase update)  
  - Wait  
  - Sticky Note2 (Enrichment Pipeline description)

- **Node Details:**

  1. **Schedule Trigger**  
     - Type: Schedule Trigger  
     - Role: Periodically triggers the enrichment pipeline every 25 minutes to process batches of unprocessed chunks.  
     - Config: Interval set to 25 minutes  
     - Inputs: None  
     - Outputs: Triggers downstream nodes  

  2. **Get many rows**  
     - Type: Supabase (getAll operation)  
     - Role: Retrieves up to 150 document chunks where metadata.processed is not true (i.e., unprocessed).  
     - Config: Table "documents", filter metadata->>processed != true  
     - Inputs: Trigger from Schedule Trigger  
     - Outputs: List of unprocessed chunks  

  3. **Loop Over Items**  
     - Type: SplitInBatches  
     - Role: Processes retrieved rows in batches of 5 to avoid rate limits and manage load.  
     - Config: batchSize = 5  
     - Inputs: List of unprocessed rows  
     - Outputs: Individual rows forwarded in batches  

  4. **Edit Fields**  
     - Type: Set node  
     - Role: Extracts and sets the row ID as `row_id` for update operations.  
     - Config: Assigns `row_id` from the current item's `id` field  
     - Inputs: Individual chunk row  
     - Outputs: Enriched JSON with row_id  

  5. **Metadata Obtention**  
     - Type: Google Gemini LLM Chat  
     - Role: Runs a prompt to analyze the text chunk and produce a single-line minified JSON with semantic metadata fields such as topics, feature, use_case, audience, entities, key recommendations, risks, examples_present, language, and chunk_summary.  
     - Config: Prompts with the chunk content and file metadata; temperature 0.1; max tokens 512; model `models/gemma-3-12b-it`.  
     - Inputs: Chunk content and metadata  
     - Outputs: JSON enrichment metadata object  
     - Failures: LLM timeouts, API quota issues, malformed JSON output  

  6. **Update a row**  
     - Type: Supabase (update operation)  
     - Role: Updates the document row identified by `row_id` with the enriched metadata JSON, setting processed flag to true and adding enrichment version and timestamp.  
     - Config: Uses JSON merging to keep previous metadata and add enrichment results; updates metadata.processed=true.  
     - Inputs: Enriched metadata from Metadata Obtention  
     - Outputs: Confirmation of DB update  
     - Failures: DB write errors, JSON parse errors  

  7. **Wait**  
     - Type: Wait node  
     - Role: Delays next batch processing until previous batch updates complete, providing flow control.  
     - Inputs: Completion signal from Update a row  
     - Outputs: Triggers next batch or ends loop  

- **Edge Cases & Failures:**  
  - LLM output invalid JSON or fails generation  
  - Supabase update failure or race conditions on concurrent updates  
  - Rate limits on Google Gemini API  
  - Batches hanging due to wait node misconfiguration  

- **Sticky Notes:**  
  - Describes asynchronous enrichment to reduce cost, using lightweight LLMs to improve retrieval filters.

---

#### 2.3 RAG Chat Agent Pipeline

- **Overview:**  
  This block handles incoming user chat messages, transforms them into structured semantic queries, retrieves relevant document chunks with filtering and reranking, maintains chat memory in Postgres, and generates grounded, explainable answers using a Google Gemini LLM. It ensures answers rely solely on retrieved documents and provides references and fallback behaviors for ambiguous or weak queries.

- **Nodes Involved:**  
  - When chat message received (chatTrigger)  
  - Query Builder (Google Gemini LLM)  
  - RAG Agent (Langchain Agent)  
  - Supabase Vector Store (retrieve tool)  
  - Reranker (Cohere)  
  - Embeddings Google Gemini  
  - Postgres Chat Memory  
  - Google Gemini 2.0 Flash (LM Chat)  
  - Sticky Note4 (RAG Agent Pipeline description)  
  - Sticky Note5 (RAG Chat Agent header)

- **Node Details:**

  1. **When chat message received**  
     - Type: Chat Trigger  
     - Role: Webhook entry point for user chat messages, publicly accessible.  
     - Config: Listens for chatInput field  
     - Inputs: User chat messages  
     - Outputs: Forwards chatInput string downstream  
     - Failures: Webhook connectivity, invalid payloads  

  2. **Query Builder**  
     - Type: Google Gemini LLM Chat  
     - Role: Analyzes user question text to produce a single-line minified JSON containing keywords array and optional filters (feature, use_case, topics, entities) for semantic retrieval.  
     - Config: Model `models/gemma-3n-e2b-it`, temperature 0.1, maxOutputTokens 128  
     - Inputs: chatInput from chat trigger  
     - Outputs: JSON with keywords and filters  
     - Failures: LLM API errors, JSON parse errors  

  3. **RAG Agent**  
     - Type: LangChain Agent Node  
     - Role: Core orchestrator that receives the user question, calls the Supabase Vector Store retrieval tool with query filters and keywords, uses Postgres chat memory, invokes reranker for passage ranking, and produces final answer with references.  
     - Config:  
       - System message enforces strict grounding in retrieved passages only.  
       - Language auto-detection from user message.  
       - Recovery query based on Query Builder output.  
       - Output style: concise, friendly, objective, with references.  
       - Failsafe: responds with a polite fallback if no evidence found or if user greets.  
     - Inputs: User chatInput, Query Builder output, retrieved passages, chat memory  
     - Outputs: Final chat answer JSON  
     - Failures: Retrieval failures, memory DB errors, LLM generation failures  

  4. **Supabase Vector Store**  
     - Type: Vector Store Retrieval Tool (Supabase)  
     - Role: Retrieves top 48 passages matching keywords and metadata filters from documents table using the custom `match_documents` SQL function. Supports reranking.  
     - Config: Uses filters extracted from Query Builder JSON; enables reranker integration; table "documents".  
     - Inputs: Keywords and filters from Query Builder  
     - Outputs: Retrieved document chunks with metadata fields (doc_title, section, feature, use_case, topics, chunk_summary)  
     - Failures: DB connectivity, malformed filters, vector search errors  

  5. **Reranker**  
     - Type: Cohere Reranker  
     - Role: Re-ranks top N (8) retrieved passages to improve relevance before final answer generation.  
     - Config: topN=8; uses Cohere API credentials  
     - Inputs: Retrieved passages from Supabase Vector Store  
     - Outputs: Ranked passages for agent consumption  
     - Failures: API quota, reranker errors  

  6. **Embeddings Google Gemini**  
     - Type: Embeddings node  
     - Role: Provides embeddings for reranker or other components as needed.  
     - Config: Uses Google Gemini API  
     - Inputs/Outputs: Linked as ai_embedding for reranker and vector store  

  7. **Postgres Chat Memory**  
     - Type: LangChain Postgres Chat Memory  
     - Role: Maintains conversation history state to provide context-aware responses.  
     - Config: Connects to Postgres DB with credentials  
     - Inputs: Chat history state from user interactions  
     - Outputs: Memory context for RAG Agent  
     - Failures: DB connectivity, schema mismatch  

  8. **Google Gemini 2.0 Flash**  
     - Type: Language Model Chat (Google Gemini)  
     - Role: Generates the final answer text given the context and retrieved information.  
     - Config: Model "gemini-2.0-flash", temperature 0.2  
     - Inputs: Passages from reranker, memory, user query  
     - Outputs: Final answer text  
     - Failures: API rate limits, generation errors  

- **Edge Cases & Failures:**  
  - Ambiguous or empty user queries returning empty keywords  
  - No matching documents found in vector store  
  - Reranker or LM API outages  
  - Memory DB failures affecting context  

- **Sticky Notes:**  
  - Highlights modular, safe, and explainable RAG agent logic combining query building, reranking, memory, and LLM generation.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                           | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                                     |
|-----------------------------|-----------------------------------------|-----------------------------------------|------------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                            | Entry point for PDF uploads              | -                                  | Extract from File                  |                                                                                                               |
| Extract from File           | Extract from File (PDF)                 | Extracts text from uploaded PDF          | On form submission                 | Set File Data                     |                                                                                                               |
| Set File Data               | Set                                    | Sets document metadata                    | Extract from File                  | Delete Old Doc Rows, Merge         |                                                                                                               |
| Delete Old Doc Rows         | Supabase (Delete)                       | Deletes old rows with same doc_id         | Set File Data                     | Merge                            |                                                                                                               |
| Merge                      | Merge                                  | Merges deletion and ingestion branches    | Delete Old Doc Rows, Recursive ... | Insert into Supabase Vectorstore  |                                                                                                               |
| Recursive Character Text Splitter | Text Splitter (recursive character) | Splits extracted text into chunks         | Extract from File                  | Default Data Loader               |                                                                                                               |
| Default Data Loader         | LangChain Data Loader                   | Converts chunks to LangChain documents    | Recursive Character Text Splitter | Embeddings Google Gemini1         |                                                                                                               |
| Embeddings Google Gemini1   | Google Gemini Embeddings                | Generates vector embeddings for chunks    | Default Data Loader               | Insert into Supabase Vectorstore  |                                                                                                               |
| Insert into Supabase Vectorstore | Supabase Vector Store (Insert)        | Inserts embeddings and metadata into DB  | Embeddings Google Gemini1, Default Data Loader | -                      |                                                                                                               |
| Schedule Trigger           | Schedule Trigger                       | Periodic trigger for enrichment           | -                                  | Get many rows                    |                                                                                                               |
| Get many rows              | Supabase (getAll)                      | Retrieves unprocessed document chunks     | Schedule Trigger                  | Loop Over Items                  |                                                                                                               |
| Loop Over Items            | SplitInBatches                        | Processes rows in batches                  | Get many rows                    | Edit Fields, Loop Over Items (empty branch) |                                                                                                               |
| Edit Fields                | Set                                    | Sets row_id for update                      | Loop Over Items                  | Metadata Obtention              |                                                                                                               |
| Metadata Obtention         | Google Gemini LLM Chat                  | Enriches chunk with semantic metadata     | Edit Fields                     | Update a row                   |                                                                                                               |
| Update a row               | Supabase (Update)                      | Updates document row with enrichment data | Metadata Obtention               | Wait                          |                                                                                                               |
| Wait                      | Wait                                   | Controls flow between batch updates        | Update a row                    | Loop Over Items                |                                                                                                               |
| When chat message received | LangChain Chat Trigger                 | Entry point for user chat messages         | -                              | Query Builder                 |                                                                                                               |
| Query Builder             | Google Gemini LLM Chat                 | Builds JSON query with keywords and filters | When chat message received      | RAG Agent                    |                                                                                                               |
| RAG Agent                 | LangChain Agent                       | Orchestrates retrieval, memory, and LLM    | Query Builder, Supabase Vector Store, Postgres Chat Memory, Reranker, Google Gemini 2.0 Flash | -                        |                                                                                                               |
| Supabase Vector Store      | Supabase Vector Store (Retrieve)      | Retrieves relevant document chunks         | Reranker, Query Builder         | RAG Agent                    |                                                                                                               |
| Reranker                  | Cohere Reranker                      | Re-ranks retrieved document chunks          | Supabase Vector Store           | Supabase Vector Store (reranker) |                                                                                                               |
| Embeddings Google Gemini   | Google Gemini Embeddings              | Provides embeddings for reranker or retrieval | -                            | Supabase Vector Store           |                                                                                                               |
| Postgres Chat Memory       | LangChain Postgres Memory             | Maintains conversation history              | -                              | RAG Agent                    |                                                                                                               |
| Google Gemini 2.0 Flash    | Google Gemini LM Chat                 | Generates final answer text                   | RAG Agent                    | -                            |                                                                                                               |
| Sticky Note1               | Sticky Note                          | Describes File Ingestion pipeline          | -                              | -                            | ## File Ingestion pipeline                                                                                     |
| Sticky Note2               | Sticky Note                          | Describes Enrichment pipeline               | -                              | -                            | ## 🟨 Enrichment Pipeline (Async)\n\nEnriches chunks with semantic metadata using a lightweight LLM.\nImproves retrieval and enables filters like audience, use_case, risks.\nRuns asynchronously to reduce cost — ideal for free-tier models. |
| Sticky Note3               | Sticky Note                          | Describes Enrichment pipeline               | -                              | -                            | ## Enrichment Pipeline                                                                                          |
| Sticky Note4               | Sticky Note                          | Describes RAG Agent pipeline                 | -                              | -                            | ## 🟦 RAG Agent Pipeline\n\nHandles user questions with memory, filtering, reranking, and references.\nPowered by a Query Builder + Cohere Reranker + Gemini LLM.\nAnswers only with retrieved content — safe, explainable, production-grade. |
| Sticky Note5               | Sticky Note                          | Describes RAG Chat Agent Block               | -                              | -                            | ## RAG Chat Agent                                                                                               |
| Sticky Note (File Ingestion) | Sticky Note                        | Describes File Ingestion pipeline            | -                              | -                            | ## 🟩 File Ingestion Pipeline\n\nExtracts and chunks uploaded PDFs.\nEmbeds content and stores it in Supabase vector DB.\nClean and modular, ready for other sources (Notion, Drive, etc). |
| Sticky Note26              | Sticky Note                          | Database preparation instructions            | -                              | -                            | © 2025 Lucas Peyrin\n\nDetailed instructions for SQL schema setup and embedding dimension notes.              |
| Sticky Note9               | Sticky Note                          | SQL schema and functions for vector search   | -                              | -                            | Full SQL code for table and vector search function setup, including pgvector extension and match_documents function |

---

### 4. Reproducing the Workflow from Scratch

1. **Prepare Supabase Database:**

   - Run the provided SQL schema in Supabase SQL editor: create `documents` table with `id`, `content`, `metadata` (jsonb), and `embedding` vector(768).
   - Create the `match_documents` vector search function.
   - Enable `pgvector` extension if not already present.
   - Adjust embedding dimension if switching from Gemini (768) to OpenAI (1536).

2. **Create the File Ingestion Pipeline:**

   - Add a **Form Trigger** node named "On form submission":
     - Configure to accept a single `.pdf` file field named "File".
     - Make the trigger publicly accessible or secure as needed.

   - Add **Extract from File** node:
     - Set operation to "pdf".
     - Connect input binary property as "File" from form trigger.

   - Add **Set** node named "Set File Data":
     - Assign `doc_id` as the uploaded file's filename.
     - Assign `doc_title` as either extracted title or filename fallback.
     - Assign fixed values: `doc_type` = "guide", `source` = "uploaded_pdf".
     - Connect from Extract from File.

   - Add **Supabase** node "Delete Old Doc Rows":
     - Operation: Delete.
     - Table: "documents".
     - Filter: metadata->>doc_id equals uploaded filename.
     - Connect input from Set File Data.
     - Configure Supabase credentials.

   - Add **Recursive Character Text Splitter** node:
     - chunkSize: 1500, chunkOverlap: 150.
     - Connect input from Extract from File.

   - Add **Default Data Loader** node:
     - Use LangChain Default Data Loader.
     - Pass JSON data as extracted text chunks.
     - Add metadata fields from Set File Data (doc_id, doc_title, doc_type, source).
     - Add `processed` flag as false.
     - Connect input from Recursive Character Text Splitter.

   - Add **Embeddings Google Gemini1** node:
     - Set to use Google Gemini embeddings (768 dims).
     - Connect input from Default Data Loader.
     - Configure Google Palm API credentials.

   - Add **Insert into Supabase Vectorstore** node:
     - Operation: insert.
     - Table name: "documents".
     - Query name: "match_documents".
     - Connect input from Embeddings Google Gemini1.

   - Add **Merge** node with mode "chooseBranch":
     - Connect Delete Old Doc Rows and Recursive Character Text Splitter outputs to Merge.
     - Connect Merge output to Insert into Supabase Vectorstore.

3. **Create the Enrichment Pipeline:**

   - Add **Schedule Trigger** node:
     - Set interval to 25 minutes.

   - Add **Supabase** node "Get many rows":
     - Operation: getAll.
     - Table: "documents".
     - Filter: metadata->>processed != true.
     - Limit: 150.
     - Connect input from Schedule Trigger.
     - Configure Supabase credentials.

   - Add **SplitInBatches** node "Loop Over Items":
     - Batch size: 5.
     - Connect input from Get many rows.

   - Add **Set** node "Edit Fields":
     - Assign `row_id` as the current item's `id`.
     - Connect input from Loop Over Items (first output).

   - Add **Google Gemini LLM Chat** node "Metadata Obtention":
     - Model: `models/gemma-3-12b-it`.
     - Prompt: Provide chunk text and file metadata, request single-line minified JSON with semantic fields (topics, feature, use_case, audience, entities, key_recommendations, risks_or_pitfalls, examples_present, language, chunk_summary).
     - Temperature: 0.1, max tokens: 512.
     - Connect input from Edit Fields.
     - Configure Google Palm API credentials.
     - Set onError to continue (to prevent blocking).

   - Add **Supabase** node "Update a row":
     - Operation: update.
     - Table: "documents".
     - Filter: id equals `row_id`.
     - Update metadata with merged enrichment JSON, set processed=true, add enrichment version and timestamp.
     - Connect input from Metadata Obtention.
     - Configure Supabase credentials.

   - Add **Wait** node:
     - Connect input from Update a row.
     - Connect output back to Loop Over Items (second output) to continue batch processing.

4. **Create the RAG Chat Agent Pipeline:**

   - Add **Chat Trigger** node "When chat message received":
     - Public webhook enabled.
     - Accepts chatInput text.

   - Add **Google Gemini LLM Chat** node "Query Builder":
     - Model: `models/gemma-3n-e2b-it`.
     - Prompt: Analyze user question, output single-line JSON with keywords array and optional filters.
     - Temperature: 0.1, maxOutputTokens: 128.
     - Connect input from chat trigger.
     - Configure Google Palm API credentials.

   - Add **Supabase Vector Store** node:
     - Mode: retrieve-as-tool.
     - Table: "documents".
     - TopK: 48.
     - Use filters and keywords extracted from Query Builder JSON.
     - Enable reranker.
     - Connect input from Query Builder.
     - Configure Supabase credentials.

   - Add **Cohere Reranker** node:
     - topN: 8.
     - Connect input from Supabase Vector Store.
     - Configure Cohere API credentials.

   - Add **Google Gemini Embeddings** node for embeddings support:
     - Connect as needed for reranker and vector store.

   - Add **Postgres Chat Memory** node:
     - Connect chat memory context.
     - Configure Postgres credentials.

   - Add **Google Gemini 2.0 Flash** node:
     - Model: "models/gemini-2.0-flash".
     - Temperature: 0.2.
     - Connect input from the reranked passages and memory.
     - Configure Google Palm API credentials.

   - Add **LangChain Agent** node "RAG Agent":
     - Text input: user chatInput.
     - Use system message enforcing grounding in retrieved documents.
     - Use Query Builder output for retrieval parameters.
     - Use reranker and memory nodes.
     - Connect inputs from Query Builder, Supabase Vector Store, Reranker, Postgres Chat Memory, Google Gemini 2.0 Flash.
     - Connect output to chat response.

5. **Connect all nodes per described flow.**

6. **Add sticky notes for documentation in the canvas (optional).**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| © 2025 Lucas Peyrin                                                                                                                                                               | Workflow author credit.                                                                             |
| SQL schema and setup for Supabase vector store including pgvector extension and custom vector search function `match_documents`.                                                | See Sticky Note9 and Sticky Note26 in workflow for full SQL code and instructions.                 |
| Embeddings dimension note: Gemini embeddings use 768 dimensions; OpenAI embeddings require schema adjustment to 1536 dimensions.                                                | Important for DB schema compatibility.                                                             |
| Workflow architecture is modular: ingestion, enrichment, and RAG chat agent are separate blocks enabling scalability and maintenance.                                            | Sticky notes and node grouping reflect this design.                                               |
| RAG Agent design ensures safe, explainable answers strictly grounded on indexed document passages, with fallback messages for insufficient evidence.                            | System messages and answer rules in RAG Agent node.                                               |
| Enrichment pipeline runs asynchronously to reduce costs and improve retrieval precision by adding semantic metadata fields.                                                     | Allows richer filtering on use_case, audience, risks, topics, etc.                                |
| Uses Google Gemini (PaLM) API for embeddings, query building, metadata enrichment, and final generation; Cohere API for reranking; Supabase for vector DB; Postgres for memory. | Credentials must be set up accordingly with appropriate API keys and access.                       |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---