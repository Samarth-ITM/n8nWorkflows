Build an OpenAI RAG system with document upload, semantic search and caching

https://n8nworkflows.xyz/workflows/build-an-openai-rag-system-with-document-upload--semantic-search-and-caching-14827


# Build an OpenAI RAG system with document upload, semantic search and caching

# 1. Workflow Overview

This workflow implements a two-path Retrieval-Augmented Generation (RAG) system exposed through a single **POST webhook**. It supports:

- **Document upload**: receive a PDF, extract text, split it into chunks, generate embeddings, and store them in **PGVector**
- **Question answering**: receive a user query, check a **Postgres cache**, and if needed retrieve relevant chunks from PGVector and generate an answer with **OpenAI**

The workflow is designed for multi-user usage through `user_id`, so uploaded content and query retrieval are scoped per user.

## 1.1 Input Reception and Runtime Configuration
The workflow starts from a webhook endpoint and immediately sets reusable configuration values such as chunk size, chunk overlap, vector table name, cache table name, and retrieval count.

## 1.2 Action Routing
The request is routed based on `body.action` into either:
- an **upload flow**
- a **query flow**

## 1.3 Upload Processing
For uploads, the workflow extracts text from a PDF, prepares chunking through a LangChain document loader and recursive text splitter, generates embeddings with OpenAI, and stores the chunks in PGVector.

## 1.4 Upload Logging and Response
After successful ingestion, the workflow records the upload in Postgres and returns a success response to the caller.

## 1.5 Query Cache Lookup
For queries, the workflow first checks whether an answer for the same user and same query already exists in `query_cache` and is less than one hour old.

## 1.6 Cached Response Path
If a valid cache entry exists, the workflow formats a normalized JSON response and returns it directly without calling the model again.

## 1.7 Retrieval and Answer Generation
If there is no valid cache entry, the workflow uses OpenAI embeddings plus PGVector retrieval to expose relevant chunks as a vector-store tool to an AI agent backed by an OpenAI chat model.

## 1.8 Cache Writeback and Final Response
The newly generated answer is written back into the query cache and then returned to the webhook caller.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Runtime Configuration

### Overview
This block receives all external requests and defines global parameters used later in both upload and query branches. It centralizes chunking and database table settings so the workflow can be adjusted from one place.

### Nodes Involved
- `Webhook Trigger`
- `Workflow Configuration`

### Node Details

#### 1. Webhook Trigger
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point of the workflow.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `rag-system`
  - Response mode: `lastNode`
- **Key expressions or variables used:**
  - Downstream nodes read values such as:
    - `$('Webhook Trigger').first().json.body.action`
    - `$('Webhook Trigger').first().json.body.user_id`
    - `$('Webhook Trigger').first().json.body.query`
    - `$('Webhook Trigger').first().json.body.document_name`
- **Input and output connections:**
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:**
  - Uses node type version `2.1`
- **Edge cases or potential failure types:**
  - Missing expected request body fields
  - Upload requests without binary file data may fail later in extraction
  - Query requests with empty `query` value will break the semantic search/usefulness of answer generation
  - If webhook is not activated in production mode, endpoint may not be available
- **Sub-workflow reference:** none

#### 2. Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates reusable configuration fields for downstream expressions.
- **Configuration choices:**
  - Preserves incoming fields with `includeOtherFields: true`
  - Assigns:
    - `chunkSize = 1000`
    - `chunkOverlap = 200`
    - `topK = 5`
    - `tableName = "documents"`
    - `cacheTableName = "query_cache"`
- **Key expressions or variables used:**
  - Referenced later as:
    - `$('Workflow Configuration').first().json.chunkSize`
    - `$('Workflow Configuration').first().json.chunkOverlap`
    - `$('Workflow Configuration').first().json.tableName`
- **Input and output connections:**
  - Input: `Webhook Trigger`
  - Output: `Route by Action`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Hardcoded values mean runtime changes require editing the workflow
  - `topK` is defined but not actually used in any visible downstream node configuration
  - `cacheTableName` is also defined but SQL queries still directly reference `query_cache`
- **Sub-workflow reference:** none

---

## Block 2 — Action Routing

### Overview
This block branches execution based on the incoming action. It decides whether the workflow should ingest a document or answer a question.

### Nodes Involved
- `Route by Action`

### Node Details

#### 3. Route by Action
- **Type and technical role:** `n8n-nodes-base.switch`  
  Evaluates request payload and routes items to upload or query processing.
- **Configuration choices:**
  - Rule 1: output `Upload` if `body.action == "upload"`
  - Rule 2: output `Query` if `body.action == "query"`
  - Fallback output: `extra`
- **Key expressions or variables used:**
  - `={{ $json.body.action }}`
- **Input and output connections:**
  - Input: `Workflow Configuration`
  - Output 0: `Extract Text from Document`
  - Output 1: `Check Query Cache`
  - Fallback: defined but not connected
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Any action other than `upload` or `query` is sent to fallback and effectively dropped because nothing is connected there
  - Strict validation means malformed or non-string `action` values will not match
- **Sub-workflow reference:** none

---

## Block 3 — Upload Processing

### Overview
This block processes uploaded PDF documents into vectorized chunks. It handles text extraction, chunking, document loading, embedding generation, and vector storage.

### Nodes Involved
- `Extract Text from Document`
- `Text Splitter`
- `Document Loader`
- `OpenAI Embeddings`
- `Store Embeddings in PGVector`

### Node Details

#### 4. Extract Text from Document
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts text content from an uploaded PDF file.
- **Configuration choices:**
  - Operation: `pdf`
- **Key expressions or variables used:**
  - Outputs extracted `text`, which is later consumed by `Document Loader`
- **Input and output connections:**
  - Input: `Route by Action` (upload branch)
  - Output: `Store Embeddings in PGVector`
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - Non-PDF files will likely fail
  - Missing binary file data
  - Corrupted or encrypted PDFs may not extract correctly
  - Large PDFs may cause memory/time issues
- **Sub-workflow reference:** none

#### 5. Text Splitter
- **Type and technical role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`  
  Defines how extracted text should be chunked before embedding.
- **Configuration choices:**
  - Chunk size from workflow config: `1000`
  - Chunk overlap from workflow config: `200`
  - Uses recursive character splitting
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.chunkSize }}`
  - `={{ $('Workflow Configuration').first().json.chunkOverlap }}`
- **Input and output connections:**
  - Input: none through main path; connected as an AI helper to `Document Loader`
  - Output: `Document Loader` via `ai_textSplitter`
- **Version-specific requirements:**
  - Uses node type version `1`
- **Edge cases or potential failure types:**
  - Very small chunk size may reduce answer quality
  - Very large chunk size may reduce retrieval precision and increase embedding cost
- **Sub-workflow reference:** none

#### 6. Document Loader
- **Type and technical role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  Converts extracted raw text into LangChain document objects and applies the custom splitter.
- **Configuration choices:**
  - JSON mode: `expressionData`
  - Source text: `={{ $json.text }}`
  - Text splitting mode: `custom`
- **Key expressions or variables used:**
  - `={{ $json.text }}`
- **Input and output connections:**
  - Main/AI inputs:
    - Receives text splitter via `ai_textSplitter`
  - Output:
    - `Store Embeddings in PGVector` via `ai_document`
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - Empty extracted text results in empty or low-value documents
  - If extraction output field changes, this expression breaks
- **Sub-workflow reference:** none

#### 7. OpenAI Embeddings
- **Type and technical role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Generates vector embeddings for both storage and retrieval operations.
- **Configuration choices:**
  - Default options only
  - Shared by both vector insert and vector retrieval nodes
- **Key expressions or variables used:**
  - No custom expression in parameters
- **Input and output connections:**
  - Output to:
    - `Store Embeddings in PGVector` via `ai_embedding`
    - `Retrieve Relevant Chunks` via `ai_embedding`
- **Version-specific requirements:**
  - Uses node type version `1.2`
  - Requires valid OpenAI credentials
- **Edge cases or potential failure types:**
  - Invalid API key
  - Rate limits
  - Model availability/region restrictions depending on account
  - Embedding dimension mismatch with existing PGVector table schema if table was created with a different model setup
- **Sub-workflow reference:** none

#### 8. Store Embeddings in PGVector
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePGVector`  
  Inserts chunk embeddings into a PGVector-backed vector store.
- **Configuration choices:**
  - Mode: `insert`
  - Table name from config: `documents`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.tableName }}`
- **Input and output connections:**
  - Main input: `Extract Text from Document`
  - AI document input: `Document Loader`
  - AI embedding input: `OpenAI Embeddings`
  - Main output: `Log Upload to Cache`
- **Version-specific requirements:**
  - Uses node type version `1.3`
  - Requires Postgres/PGVector credentials configured in the node
- **Edge cases or potential failure types:**
  - Missing PGVector extension
  - Table schema not initialized correctly
  - Embedding dimension mismatch
  - Missing metadata strategy may limit later filtering if user-level metadata is expected in retrieval
- **Sub-workflow reference:** none

**Important implementation note:**  
The retrieval node later filters by `user_id`, but the insert node configuration shown here does **not explicitly define metadata assignment** for `user_id`. If user metadata is not being attached automatically elsewhere, retrieval filtering may not work as intended and could return no results or cross-user data issues depending on table setup.

---

## Block 4 — Upload Logging and Upload Response

### Overview
After embeddings are stored, the workflow logs the upload event in Postgres and returns a success payload to the webhook caller.

### Nodes Involved
- `Log Upload to Cache`
- `Respond Upload Success`

### Node Details

#### 9. Log Upload to Cache
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Executes a SQL insert to track uploads.
- **Configuration choices:**
  - Operation: `executeQuery`
  - SQL:
    - `INSERT INTO upload_log (user_id, document_name, uploaded_at) VALUES ($1, $2, $3)`
  - Query replacements:
    - user ID from webhook
    - document name from webhook
    - current timestamp
- **Key expressions or variables used:**
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
  - `={{ $('Webhook Trigger').first().json.body.document_name }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**
  - Input: `Store Embeddings in PGVector`
  - Output: `Respond Upload Success`
- **Version-specific requirements:**
  - Uses node type version `2.6`
  - Requires Postgres credentials
- **Edge cases or potential failure types:**
  - Table `upload_log` must exist
  - Null `document_name` or `user_id` may violate schema
  - Name is misleading: this node logs uploads, not cache
- **Sub-workflow reference:** none

#### 10. Respond Upload Success
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns final JSON confirming upload success.
- **Configuration choices:**
  - Response format: JSON
  - Body includes:
    - `status: success`
    - `message: Document uploaded and processed`
    - `user_id`
    - `document_name`
- **Key expressions or variables used:**
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
  - `={{ $('Webhook Trigger').first().json.body.document_name }}`
- **Input and output connections:**
  - Input: `Log Upload to Cache`
  - Output: none
- **Version-specific requirements:**
  - Uses node type version `1.5`
- **Edge cases or potential failure types:**
  - If prior nodes fail, no response is returned unless workflow error handling is added
- **Sub-workflow reference:** none

---

## Block 5 — Query Cache Lookup

### Overview
This block checks whether the exact same query from the same user has already been answered recently. It prevents unnecessary model calls and reduces response time.

### Nodes Involved
- `Check Query Cache`
- `Cache Hit or Miss`

### Node Details

#### 11. Check Query Cache
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Performs SQL lookup in the query cache table.
- **Configuration choices:**
  - Operation: `executeQuery`
  - SQL:
    - Select `answer, created_at`
    - Match on `user_id`
    - Match on `MD5(query)`
    - Restrict cache to last `1 hour`
- **Key expressions or variables used:**
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
  - `={{ $('Webhook Trigger').first().json.body.query }}`
- **Input and output connections:**
  - Input: `Route by Action` (query branch)
  - Output: `Cache Hit or Miss`
- **Version-specific requirements:**
  - Uses node type version `2.6`
  - Requires Postgres credentials
- **Edge cases or potential failure types:**
  - Table `query_cache` must exist
  - MD5 matching is exact-string based; whitespace/casing differences create misses
  - Cache duration is hardcoded to one hour
  - The configured `cacheTableName` is not used here
- **Sub-workflow reference:** none

#### 12. Cache Hit or Miss
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branches depending on whether the SQL query returned rows.
- **Configuration choices:**
  - `Cache Hit` if `{{$json.length}} > 0`
  - `Cache Miss` if `{{$json.length}} == 0`
- **Key expressions or variables used:**
  - `={{ $json.length }}`
- **Input and output connections:**
  - Input: `Check Query Cache`
  - Output 0: `Format Cached Response`
  - Output 1: `Answer Query with Context`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Depends on the Postgres node returning array-length-like output in the current n8n behavior
  - If output structure differs, the switch may misroute
- **Sub-workflow reference:** none

---

## Block 6 — Cached Response Path

### Overview
If a fresh cache entry exists, this block normalizes the cached record into a response payload and returns it immediately.

### Nodes Involved
- `Format Cached Response`
- `Respond with Cached Answer`

### Node Details

#### 13. Format Cached Response
- **Type and technical role:** `n8n-nodes-base.set`  
  Shapes the cached DB result into a consistent response object.
- **Configuration choices:**
  - Assigns:
    - `answer = {{$json.answer}}`
    - `cached = true`
    - `user_id = webhook body user_id`
- **Key expressions or variables used:**
  - `={{ $json.answer }}`
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
- **Input and output connections:**
  - Input: `Cache Hit or Miss` (cache hit branch)
  - Output: `Respond with Cached Answer`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - If multiple cache rows are returned, item handling may need attention
  - If Postgres output nests rows differently than expected, `answer` may be undefined
- **Sub-workflow reference:** none

#### 14. Respond with Cached Answer
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns cached answer JSON.
- **Configuration choices:**
  - Response format: JSON
  - Body:
    - `answer`
    - `cached: true`
    - `user_id`
- **Key expressions or variables used:**
  - `={{ $json.answer }}`
  - `={{ $json.user_id }}`
- **Input and output connections:**
  - Input: `Format Cached Response`
  - Output: none
- **Version-specific requirements:**
  - Uses node type version `1.5`
- **Edge cases or potential failure types:**
  - If `answer` is empty or null, caller still gets a successful cached response unless validation is added
- **Sub-workflow reference:** none

---

## Block 7 — Retrieval and Answer Generation

### Overview
On a cache miss, the workflow performs semantic retrieval and lets an AI agent answer using the retrieved chunks. The vector store is exposed as a tool to the agent rather than directly concatenating context in a prompt.

### Nodes Involved
- `Retrieve Relevant Chunks`
- `Answer questions with a vector store`
- `OpenAI Chat Model`
- `Answer Query with Context`
- `OpenAI Embeddings`

### Node Details

#### 15. Retrieve Relevant Chunks
- **Type and technical role:** `@n8n/n8n-nodes-langchain.vectorStorePGVector`  
  Performs vector search over PGVector-backed documents.
- **Configuration choices:**
  - Table name from config: `documents`
  - Metadata filter:
    - `user_id = webhook body user_id`
- **Key expressions or variables used:**
  - `={{ $('Workflow Configuration').first().json.tableName }}`
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
- **Input and output connections:**
  - AI embedding input: `OpenAI Embeddings`
  - Output: `Answer questions with a vector store` via `ai_vectorStore`
- **Version-specific requirements:**
  - Uses node type version `1.3`
  - Requires Postgres/PGVector credentials
- **Edge cases or potential failure types:**
  - If stored documents do not include matching `user_id` metadata, retrieval may return no chunks
  - `topK` is not explicitly set despite being configured upstream
  - Empty vector store for a user will produce poor/no context
- **Sub-workflow reference:** none

#### 16. Answer questions with a vector store
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolVectorStore`  
  Wraps the retriever as a callable tool for the AI agent.
- **Configuration choices:**
  - No visible custom parameters
- **Key expressions or variables used:**
  - None in the visible config
- **Input and output connections:**
  - AI vector store input: `Retrieve Relevant Chunks`
  - AI tool output: `Answer Query with Context`
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - Tool behavior depends on correct retriever configuration upstream
  - Minimal explicit configuration means defaults may vary with node version
- **Sub-workflow reference:** none

#### 17. OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - No extra options configured
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - Output: `Answer Query with Context` via `ai_languageModel`
- **Version-specific requirements:**
  - Uses node type version `1.3`
  - Requires OpenAI credentials
- **Edge cases or potential failure types:**
  - API authentication errors
  - Rate limits
  - Model deprecation or account access restrictions
- **Sub-workflow reference:** none

#### 18. Answer Query with Context
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main reasoning node that receives the user query, can call the vector-store tool, and generates the final answer.
- **Configuration choices:**
  - Prompt type: `define`
  - User text: webhook body query
  - System message instructs the model to:
    - answer using provided document context
    - say it lacks enough information if context is insufficient
    - cite document/section when possible
    - remain concise and professional
- **Key expressions or variables used:**
  - `={{ $('Webhook Trigger').first().json.body.query }}`
- **Input and output connections:**
  - Main input: `Cache Hit or Miss` (cache miss branch)
  - AI language model input: `OpenAI Chat Model`
  - AI tool input: `Answer questions with a vector store`
  - Main output: `Save to Query Cache`
- **Version-specific requirements:**
  - Uses node type version `3`
- **Edge cases or potential failure types:**
  - Hallucination risk remains if retrieval returns weak context
  - If tool invocation fails, answer quality may degrade
  - If no context is found, expected behavior depends on prompt compliance
- **Sub-workflow reference:** none

---

## Block 8 — Cache Writeback and Final Response

### Overview
This block persists newly generated answers for future reuse and returns the final uncached response to the client.

### Nodes Involved
- `Save to Query Cache`
- `Respond with Answer`

### Node Details

#### 19. Save to Query Cache
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts or updates the generated answer in the query cache.
- **Configuration choices:**
  - Operation: `executeQuery`
  - SQL:
    - Insert `user_id`, `MD5(query)`, raw query, answer, timestamp
    - On conflict `(user_id, query_hash)`, update answer and timestamp
- **Key expressions or variables used:**
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
  - `={{ $('Webhook Trigger').first().json.body.query }}`
  - `={{ $json.output }}`
- **Input and output connections:**
  - Input: `Answer Query with Context`
  - Output: `Respond with Answer`
- **Version-specific requirements:**
  - Uses node type version `2.6`
  - Requires Postgres credentials
  - Requires a unique constraint on `(user_id, query_hash)` for the conflict clause
- **Edge cases or potential failure types:**
  - Table or unique constraint missing
  - If agent output field changes from `output`, insert will fail or save null
  - Again, `cacheTableName` config is not reused
- **Sub-workflow reference:** none

#### 20. Respond with Answer
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns the newly generated answer.
- **Configuration choices:**
  - Response format: JSON
  - Body includes:
    - `answer`
    - `cached: false`
    - `user_id`
- **Key expressions or variables used:**
  - `={{ $('Answer Query with Context').first().json.output }}`
  - `={{ $('Webhook Trigger').first().json.body.user_id }}`
- **Input and output connections:**
  - Input: `Save to Query Cache`
  - Output: none
- **Version-specific requirements:**
  - Uses node type version `1.5`
- **Edge cases or potential failure types:**
  - If cache save fails, response is not sent because this node sits after DB write
  - Consider moving response before cache save or adding error handling if availability matters
- **Sub-workflow reference:** none

---

## Additional Nodes — Documentation / Visual Annotation

These nodes do not affect execution logic but are part of the workflow and must be accounted for.

### 21. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for the input layer
- **Content:** `## Input Layer Receive upload or query via webhook and Set chunking, topK, and database tables`

### 22. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for action routing
- **Content:** `## Action Routing Route request to upload or query flow`

### 23. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for cache lookup
- **Content:** `## Cache Check Check if query result exists in cache`

### 24. Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for cached response formatting
- **Content:** `## Cache Response Formatting Format cached query results into a consistent response structure.`

### 25. Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for cached response return
- **Content:** `## Response Layer Return answer or cached result via webhook`

### 26. Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for final response return
- **Content:** `## Response Layer Return answer or cached result via webhook`

### 27. Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for cache storage
- **Content:** `## Cache Storage Store query and response for reuse`

### 28. Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for answer generation
- **Content:** `## Answer Generation Generate answer using context + AI model`

### 29. Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for document extraction
- **Content:** `## Document Processing Extract text from uploaded documents`

### 30. Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for chunking
- **Content:** `## Chunking Engine Split text into overlapping chunks`

### 31. Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for embeddings storage / retrieval area
- **Content:** `## Embeddings Storage Generate embeddings and store in PGVector`

### 32. Sticky Note12
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Visual annotation for upload logging
- **Content:** `## Upload Logging Track document uploads in database`

### 33. Sticky Note13
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Global explanation and setup notes
- **Content:**  
  - Complete RAG explanation  
  - Setup steps:
    - Configure webhook endpoint for upload and query actions
    - Add OpenAI API credentials for embeddings and chat
    - Set up Postgres with PGVector extension
    - Create tables for documents and query cache
    - Adjust chunk size, overlap, and topK settings

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Receives POST requests for upload or query actions |  | Workflow Configuration | ## Input Layer<br>Receive upload or query via webhook and Set chunking, topK, and database tables |
| Workflow Configuration | n8n-nodes-base.set | Defines global runtime settings like chunk size and table names | Webhook Trigger | Route by Action | ## Input Layer<br>Receive upload or query via webhook and Set chunking, topK, and database tables |
| Route by Action | n8n-nodes-base.switch | Branches workflow by `body.action` into upload or query paths | Workflow Configuration | Extract Text from Document; Check Query Cache | ## Action Routing<br>Route request to upload or query flow |
| Extract Text from Document | n8n-nodes-base.extractFromFile | Extracts text from uploaded PDF documents | Route by Action | Store Embeddings in PGVector | ## Document Processing<br>Extract text from uploaded documents |
| Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits extracted text into overlapping chunks |  | Document Loader | ## Chunking Engine<br>Split text into overlapping chunks |
| Document Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Converts extracted text into LangChain documents and applies chunking | Text Splitter | Store Embeddings in PGVector | ## Chunking Engine<br>Split text into overlapping chunks |
| OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for storage and retrieval |  | Store Embeddings in PGVector; Retrieve Relevant Chunks | ## Embeddings Storage<br>Generate embeddings and store in PGVector |
| Store Embeddings in PGVector | @n8n/n8n-nodes-langchain.vectorStorePGVector | Stores chunk embeddings in PGVector | Extract Text from Document; Document Loader; OpenAI Embeddings | Log Upload to Cache | ## Embeddings Storage<br>Generate embeddings and store in PGVector |
| Log Upload to Cache | n8n-nodes-base.postgres | Inserts upload audit record into Postgres | Store Embeddings in PGVector | Respond Upload Success | ## Upload Logging<br>Track document uploads in database |
| Respond Upload Success | n8n-nodes-base.respondToWebhook | Returns upload success JSON | Log Upload to Cache |  | ## Upload Logging<br>Track document uploads in database |
| Check Query Cache | n8n-nodes-base.postgres | Looks up recent cached answer for the user/query pair | Route by Action | Cache Hit or Miss | ## Cache Check<br>Check if query result exists in cache |
| Cache Hit or Miss | n8n-nodes-base.switch | Routes to cached response or fresh answer generation | Check Query Cache | Format Cached Response; Answer Query with Context | ## Cache Check<br>Check if query result exists in cache |
| Format Cached Response | n8n-nodes-base.set | Normalizes cached DB result into response payload | Cache Hit or Miss | Respond with Cached Answer | ## Cache Response Formatting<br>Format cached query results into a consistent response structure. |
| Respond with Cached Answer | n8n-nodes-base.respondToWebhook | Returns cached answer JSON | Format Cached Response |  | ## Response Layer<br>Return answer or cached result via webhook |
| Retrieve Relevant Chunks | @n8n/n8n-nodes-langchain.vectorStorePGVector | Retrieves semantically relevant chunks from PGVector | OpenAI Embeddings | Answer questions with a vector store | ## Embeddings Storage<br>Generate embeddings and store in PGVector |
| Answer questions with a vector store | @n8n/n8n-nodes-langchain.toolVectorStore | Exposes the retriever as a tool to the agent | Retrieve Relevant Chunks | Answer Query with Context | ## Answer Generation<br>Generate answer using context + AI model |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the LLM used by the agent |  | Answer Query with Context | ## Answer Generation<br>Generate answer using context + AI model |
| Answer Query with Context | @n8n/n8n-nodes-langchain.agent | Generates grounded answer using query plus retrieval tool | Cache Hit or Miss; OpenAI Chat Model; Answer questions with a vector store | Save to Query Cache | ## Answer Generation<br>Generate answer using context + AI model |
| Save to Query Cache | n8n-nodes-base.postgres | Upserts generated answer into query cache | Answer Query with Context | Respond with Answer | ## Cache Storage<br>Store query and response for reuse |
| Respond with Answer | n8n-nodes-base.respondToWebhook | Returns newly generated answer JSON | Save to Query Cache |  | ## Response Layer<br>Return answer or cached result via webhook |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |
| Sticky Note13 | n8n-nodes-base.stickyNote | Visual documentation node |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites
Before building the workflow, prepare the following:

1. **OpenAI credentials**
   - One credential usable for:
     - `OpenAI Embeddings`
     - `OpenAI Chat Model`

2. **Postgres database with PGVector**
   - Enable the `vector` extension
   - Create tables for:
     - document embeddings storage
     - query cache
     - upload log

3. **n8n instance with LangChain nodes available**
   - Required node families:
     - OpenAI embeddings/chat
     - PGVector vector store
     - document loader
     - recursive text splitter
     - vector store tool
     - agent

---

## Suggested database setup
You must create the required tables yourself. At minimum:

1. **Vector table** for stored chunks  
   This must be compatible with the n8n PGVector node and the embedding dimension of the chosen OpenAI embeddings model.

2. **Query cache table**
   - Needs columns similar to:
     - `user_id`
     - `query_hash`
     - `query`
     - `answer`
     - `created_at`
   - Add a unique constraint on:
     - `(user_id, query_hash)`

3. **Upload log table**
   - Needs columns similar to:
     - `user_id`
     - `document_name`
     - `uploaded_at`

---

## Build steps

### Step 1 — Create the webhook entry point
1. Add a **Webhook** node.
2. Name it **Webhook Trigger**.
3. Set:
   - **HTTP Method**: `POST`
   - **Path**: `rag-system`
   - **Response Mode**: `Last Node`
4. Expect request body fields such as:
   - `action`
   - `user_id`
   - for queries: `query`
   - for uploads: `document_name`
5. If handling file uploads, ensure the request format actually sends a binary PDF that the extraction node can read.

### Step 2 — Add runtime configuration
1. Add a **Set** node after the webhook.
2. Name it **Workflow Configuration**.
3. Enable keeping incoming fields.
4. Add fields:
   - `chunkSize` = `1000` as number
   - `chunkOverlap` = `200` as number
   - `topK` = `5` as number
   - `tableName` = `documents` as string
   - `cacheTableName` = `query_cache` as string
5. Connect:
   - `Webhook Trigger` → `Workflow Configuration`

### Step 3 — Route by action
1. Add a **Switch** node.
2. Name it **Route by Action**.
3. Add rule:
   - output name `Upload`
   - condition: `{{$json.body.action}} equals "upload"`
4. Add rule:
   - output name `Query`
   - condition: `{{$json.body.action}} equals "query"`
5. Optionally keep fallback output for invalid actions.
6. Connect:
   - `Workflow Configuration` → `Route by Action`

---

## Upload branch

### Step 4 — Extract text from PDF
1. Add an **Extract From File** node.
2. Name it **Extract Text from Document**.
3. Set operation to **PDF extraction**.
4. Connect:
   - `Route by Action` upload output → `Extract Text from Document`
5. Ensure the webhook request delivers a PDF in binary form.

### Step 5 — Add a text splitter
1. Add a **Recursive Character Text Splitter** node.
2. Name it **Text Splitter**.
3. Set:
   - **Chunk Size**: `{{$('Workflow Configuration').first().json.chunkSize}}`
   - **Chunk Overlap**: `{{$('Workflow Configuration').first().json.chunkOverlap}}`

### Step 6 — Add a document loader
1. Add a **Document Default Data Loader** node.
2. Name it **Document Loader**.
3. Configure:
   - data source from expression mode
   - source text: `{{$json.text}}`
   - text splitting mode: `custom`
4. Connect the text splitter as AI input:
   - `Text Splitter` → `Document Loader` via `ai_textSplitter`

### Step 7 — Add OpenAI embeddings
1. Add an **OpenAI Embeddings** node.
2. Name it **OpenAI Embeddings**.
3. Attach OpenAI credentials.
4. Keep default settings unless you need a specific embedding model.

### Step 8 — Store embeddings in PGVector
1. Add a **PGVector Vector Store** node.
2. Name it **Store Embeddings in PGVector**.
3. Set:
   - **Mode**: `Insert`
   - **Table Name**: `{{$('Workflow Configuration').first().json.tableName}}`
4. Attach Postgres/PGVector credentials.
5. Connect:
   - `Extract Text from Document` → `Store Embeddings in PGVector` (main)
   - `Document Loader` → `Store Embeddings in PGVector` via `ai_document`
   - `OpenAI Embeddings` → `Store Embeddings in PGVector` via `ai_embedding`

### Step 9 — Important metadata recommendation
To support user-scoped retrieval later, configure document metadata during insert so each chunk carries at least:
- `user_id`
- optionally `document_name`

Without this, the retrieval filter on `user_id` may fail. The supplied workflow JSON does not visibly configure this on insertion, so you should add it manually if your node version supports metadata assignment.

### Step 10 — Log upload event
1. Add a **Postgres** node.
2. Name it **Log Upload to Cache**.
3. Set operation to **Execute Query**.
4. SQL:
   - `INSERT INTO upload_log (user_id, document_name, uploaded_at) VALUES ($1, $2, $3)`
5. Query replacements:
   - `{{$('Webhook Trigger').first().json.body.user_id}}`
   - `{{$('Webhook Trigger').first().json.body.document_name}}`
   - `{{$now.toISO()}}`
6. Connect:
   - `Store Embeddings in PGVector` → `Log Upload to Cache`

### Step 11 — Return upload success
1. Add a **Respond to Webhook** node.
2. Name it **Respond Upload Success**.
3. Respond with JSON body similar to:
   - `status: success`
   - `message: Document uploaded and processed`
   - `user_id`
   - `document_name`
4. Use expressions:
   - `{{$('Webhook Trigger').first().json.body.user_id}}`
   - `{{$('Webhook Trigger').first().json.body.document_name}}`
5. Connect:
   - `Log Upload to Cache` → `Respond Upload Success`

---

## Query branch

### Step 12 — Check query cache
1. Add a **Postgres** node.
2. Name it **Check Query Cache**.
3. Set operation to **Execute Query**.
4. SQL:
   - `SELECT answer, created_at FROM query_cache WHERE user_id = $1 AND query_hash = MD5($2) AND created_at > NOW() - INTERVAL '1 hour'`
5. Query replacements:
   - `{{$('Webhook Trigger').first().json.body.user_id}}`
   - `{{$('Webhook Trigger').first().json.body.query}}`
6. Connect:
   - `Route by Action` query output → `Check Query Cache`

### Step 13 — Add cache hit/miss switch
1. Add a **Switch** node.
2. Name it **Cache Hit or Miss**.
3. Add rule `Cache Hit`:
   - `{{$json.length}} > 0`
4. Add rule `Cache Miss`:
   - `{{$json.length}} == 0`
5. Connect:
   - `Check Query Cache` → `Cache Hit or Miss`

### Step 14 — Format cached response
1. Add a **Set** node.
2. Name it **Format Cached Response**.
3. Set fields:
   - `answer` = `{{$json.answer}}`
   - `cached` = `true`
   - `user_id` = `{{$('Webhook Trigger').first().json.body.user_id}}`
4. Connect:
   - `Cache Hit or Miss` cache-hit output → `Format Cached Response`

### Step 15 — Return cached answer
1. Add a **Respond to Webhook** node.
2. Name it **Respond with Cached Answer**.
3. Respond with JSON:
   - `answer` = `{{$json.answer}}`
   - `cached` = `true`
   - `user_id` = `{{$json.user_id}}`
4. Connect:
   - `Format Cached Response` → `Respond with Cached Answer`

---

## Retrieval + answer generation branch

### Step 16 — Add retriever on PGVector
1. Add another **PGVector Vector Store** node.
2. Name it **Retrieve Relevant Chunks**.
3. Configure it for retrieval/search mode appropriate for your node version.
4. Set:
   - **Table Name**: `{{$('Workflow Configuration').first().json.tableName}}`
5. Add metadata filtering:
   - `user_id = {{$('Webhook Trigger').first().json.body.user_id}}`
6. If your node version supports it, explicitly set retrieval count to:
   - `{{$('Workflow Configuration').first().json.topK}}`
7. Connect:
   - `OpenAI Embeddings` → `Retrieve Relevant Chunks` via `ai_embedding`

### Step 17 — Expose retriever as a tool
1. Add a **Vector Store Tool** node.
2. Name it **Answer questions with a vector store**.
3. Connect:
   - `Retrieve Relevant Chunks` → `Answer questions with a vector store` via `ai_vectorStore`

### Step 18 — Add chat model
1. Add an **OpenAI Chat Model** node.
2. Name it **OpenAI Chat Model**.
3. Select model:
   - `gpt-4.1-mini`
4. Attach OpenAI credentials.
5. Connect:
   - `OpenAI Chat Model` → `Answer Query with Context` via `ai_languageModel`

### Step 19 — Add the agent
1. Add an **AI Agent** node.
2. Name it **Answer Query with Context**.
3. Set prompt type to **Define**.
4. Set user text:
   - `{{$('Webhook Trigger').first().json.body.query}}`
5. Add a system message instructing the model to:
   - answer based only on provided document context
   - say it lacks enough information if context is insufficient
   - cite source document/section when possible
   - be concise and professional
6. Connect:
   - `Cache Hit or Miss` cache-miss output → `Answer Query with Context`
   - `Answer questions with a vector store` → `Answer Query with Context` via `ai_tool`
   - `OpenAI Chat Model` → `Answer Query with Context` via `ai_languageModel`

### Step 20 — Save generated answer to cache
1. Add a **Postgres** node.
2. Name it **Save to Query Cache**.
3. Set operation to **Execute Query**.
4. SQL:
   - `INSERT INTO query_cache (user_id, query_hash, query, answer, created_at) VALUES ($1, MD5($2), $3, $4, NOW()) ON CONFLICT (user_id, query_hash) DO UPDATE SET answer = EXCLUDED.answer, created_at = NOW()`
5. Query replacements:
   - `{{$('Webhook Trigger').first().json.body.user_id}}`
   - `{{$('Webhook Trigger').first().json.body.query}}`
   - `{{$('Webhook Trigger').first().json.body.query}}`
   - `{{$json.output}}`
6. Connect:
   - `Answer Query with Context` → `Save to Query Cache`

### Step 21 — Return generated answer
1. Add a **Respond to Webhook** node.
2. Name it **Respond with Answer**.
3. Respond with JSON:
   - `answer` = `{{$('Answer Query with Context').first().json.output}}`
   - `cached` = `false`
   - `user_id` = `{{$('Webhook Trigger').first().json.body.user_id}}`
4. Connect:
   - `Save to Query Cache` → `Respond with Answer`

---

## Optional visual documentation
Add sticky notes matching the original visual organization if desired:

1. Input Layer
2. Action Routing
3. Cache Check
4. Cache Response Formatting
5. Response Layer
6. Cache Storage
7. Answer Generation
8. Document Processing
9. Chunking Engine
10. Embeddings Storage
11. Upload Logging
12. Global explanation/setup note

---

## Expected request patterns

### Upload request
Send a POST request to the webhook with:
- `action = upload`
- `user_id`
- `document_name`
- a binary PDF file accessible to the extraction node

### Query request
Send a POST request to the webhook with:
- `action = query`
- `user_id`
- `query`

---

## Recommended improvements while rebuilding
To make the workflow more robust than the source version:

1. **Handle invalid `action` values**
   - Connect the switch fallback to an error response node

2. **Use config variables consistently**
   - Replace hardcoded `query_cache` in SQL with the configured `cacheTableName` if your implementation supports dynamic SQL safely

3. **Store metadata on insert**
   - Ensure uploaded chunks include `user_id` and `document_name`

4. **Apply `topK` explicitly**
   - The original flow defines it but does not visibly use it

5. **Add validation**
   - Reject missing `query`, `user_id`, or binary upload data early

6. **Add error responses**
   - Especially for Postgres failures, OpenAI errors, and extraction failures

7. **Normalize queries before hashing**
   - Lowercasing and trimming may improve cache hit rate if appropriate

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow implements a complete Retrieval-Augmented Generation (RAG) system for document ingestion and intelligent querying. | Global workflow note |
| Users can upload documents or send queries via webhook. Uploaded files are processed by extracting text, splitting it into chunks, generating embeddings, and storing them in a vector database (PGVector). | Global workflow note |
| When a query is received, the workflow first checks a cache for recent answers. If no cache is found, it retrieves relevant document chunks using vector search and generates a contextual answer using an AI model. | Global workflow note |
| Responses are cached for faster future queries and returned to the user via webhook. | Global workflow note |
| Configure webhook endpoint for upload and query actions. | Setup note |
| Add OpenAI API credentials for embeddings and chat. | Setup note |
| Set up Postgres with PGVector extension. | Setup note |
| Create tables for documents and query cache. | Setup note |
| Adjust chunk size, overlap, and topK settings. | Setup note |

## Final implementation observations
- The workflow has **one entry point**: `Webhook Trigger`
- It has **no sub-workflow invocation nodes**
- It uses **two external systems**:
  - **OpenAI** for embeddings and chat generation
  - **Postgres/PGVector** for vector storage, cache, and upload logging
- The most important technical risk in the current design is the likely mismatch between:
  - retrieval filtering on `user_id`
  - and the lack of explicit metadata assignment during vector insertion

If you want, I can also produce:
1. a **database schema proposal** for the required Postgres/PGVector tables, or  
2. a **request/response API contract** for both upload and query calls.