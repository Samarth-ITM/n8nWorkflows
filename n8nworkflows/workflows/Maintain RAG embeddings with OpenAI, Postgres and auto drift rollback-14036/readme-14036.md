Maintain RAG embeddings with OpenAI, Postgres and auto drift rollback

https://n8nworkflows.xyz/workflows/maintain-rag-embeddings-with-openai--postgres-and-auto-drift-rollback-14036


# Maintain RAG embeddings with OpenAI, Postgres and auto drift rollback

# 1. Workflow Overview

This workflow maintains a self-healing Retrieval-Augmented Generation (RAG) embedding pipeline. It automatically refreshes document embeddings from a source repository, evaluates whether the new candidate embeddings improve answer quality, measures semantic drift, and then either promotes the new version, rolls it back, or flags it for human review.

Typical use cases:
- Keeping a knowledge base or documentation RAG index up to date
- Detecting when source content changes and re-embedding only changed chunks
- Preventing degraded retrieval quality from being deployed automatically
- Adding a safety gate around embedding/model updates

The workflow is organized into the following logical blocks.

## 1.1 Trigger & Global Configuration
The workflow starts either on a daily schedule or from an external webhook. A central Set node defines all global parameters used later, including source URL, chunking parameters, thresholds, and notification endpoint.

## 1.2 Document Retrieval & Deterministic Chunking
The workflow fetches documents from the configured source, splits them into deterministic chunks, and computes a SHA-256 hash for each chunk. These hashes serve as change fingerprints.

## 1.3 Previous Chunk State Lookup & Change Detection
Previously stored chunk hashes are loaded from Postgres and compared against the newly computed hashes. This block identifies which chunks have changed and therefore need new embeddings.

## 1.4 Candidate Embedding Creation
Changed chunks are prepared as LangChain documents, split using the configured recursive text splitter, embedded with OpenAI, and inserted into an in-memory vector store representing the candidate version.

## 1.5 Candidate Version Metadata Persistence
The workflow extracts metadata about the candidate embedding build and stores a new row in Postgres as an embedding version candidate.

## 1.6 Golden Question Evaluation
A predefined set of golden questions is loaded from Postgres. Two AI agents answer the same questions: one using the new candidate vector store and one using the old/current vector store.

## 1.7 Quality Scoring & Drift Analysis
The workflow computes aggregate retrieval/answer quality metrics and also estimates embedding drift using cosine distance between old and new embedding vectors.

## 1.8 Promotion, Rollback, or Manual Review
An IF gate checks whether quality exceeds the configured threshold and drift remains below the allowed threshold. If yes, the candidate is promoted. If not, the workflow currently triggers both rollback and manual review in parallel.

## 1.9 Notification
A notification webhook is called to report the result to Slack, Teams, or another incoming webhook consumer.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Global Configuration

**Overview:**  
This block defines how the workflow starts and centralizes reusable configuration values. All downstream nodes rely on these values through expressions.

**Nodes Involved:**  
- Daily RAG Maintenance Schedule
- Source Change Webhook
- Workflow Configuration

### Node: Daily RAG Maintenance Schedule
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; scheduled entry point
- **Configuration choices:** Runs daily at hour 2
- **Key expressions/variables:** None
- **Input/output connections:** No input; outputs to `Workflow Configuration`
- **Version-specific requirements:** Type version 1.3
- **Edge cases/failures:** Timezone assumptions may differ from expected server timezone
- **Sub-workflow reference:** None

### Node: Source Change Webhook
- **Type and role:** `n8n-nodes-base.webhook`; event-driven entry point
- **Configuration choices:** POST webhook on path `rag-update`
- **Key expressions/variables:** None
- **Input/output connections:** No input; outputs to `Workflow Configuration`
- **Version-specific requirements:** Type version 2.1
- **Edge cases/failures:** Webhook must be activated in production; payload is not validated; malformed calls may still start the workflow
- **Sub-workflow reference:** None

### Node: Workflow Configuration
- **Type and role:** `n8n-nodes-base.set`; central configuration registry
- **Configuration choices:** Sets:
  - `documentSourceUrl`
  - `chunkSize = 1000`
  - `chunkOverlap = 200`
  - `qualityThreshold = 0.85`
  - `driftThreshold = 0.15`
  - `notificationWebhook`
  - `includeOtherFields = true`
- **Key expressions/variables:** Referenced throughout via `$('Workflow Configuration').first().json...`
- **Input/output connections:** Inputs from both triggers; outputs to `Fetch Documents from Source` and `Fetch Previous Chunk Hashes`
- **Version-specific requirements:** Type version 3.4
- **Edge cases/failures:** Placeholder values must be replaced; invalid numeric values can break chunking or threshold logic
- **Sub-workflow reference:** None

---

## 2.2 Document Retrieval & Deterministic Chunking

**Overview:**  
This block loads source content and transforms it into deterministic chunks. Each chunk receives a stable hash so the workflow can identify content changes reliably.

**Nodes Involved:**  
- Fetch Documents from Source
- Chunk Documents & Compute Hash

### Node: Fetch Documents from Source
- **Type and role:** `n8n-nodes-base.httpRequest`; loads source documents
- **Configuration choices:** GETs the URL from `documentSourceUrl`; expects JSON response
- **Key expressions/variables:** `={{ $('Workflow Configuration').first().json.documentSourceUrl }}`
- **Input/output connections:** Input from `Workflow Configuration`; output to `Chunk Documents & Compute Hash`
- **Version-specific requirements:** Type version 4.3
- **Edge cases/failures:** Auth is not configured; non-JSON responses fail; pagination is not handled; network timeouts and 4xx/5xx responses are possible
- **Sub-workflow reference:** None

### Node: Chunk Documents & Compute Hash
- **Type and role:** `n8n-nodes-base.code`; deterministic chunking and SHA-256 hash generation
- **Configuration choices:** 
  - Reads `chunkSize`, `chunkOverlap`, and optional `sourceRevision`
  - Uses `content`, `text`, or fallback `JSON.stringify(doc.json)`
  - Builds `chunkId` as `<doc id or doc>_chunk_<index>`
- **Key expressions/variables:** Reads config using `$('Workflow Configuration').first().json`
- **Input/output connections:** Input from `Fetch Documents from Source`; output to `Detect Changed Chunks`
- **Version-specific requirements:** Type version 2; depends on Node.js `crypto` module availability in Code node
- **Edge cases/failures:** 
  - If `chunkOverlap >= chunkSize`, loop progression becomes invalid
  - Missing `id` produces generic chunk IDs that may collide across documents
  - If source payload shape is unexpected, content may become serialized JSON instead of intended text
- **Sub-workflow reference:** None

---

## 2.3 Previous Chunk State Lookup & Change Detection

**Overview:**  
This block fetches the stored chunk state from Postgres and compares previous hashes with current hashes. Only changed chunks continue into the candidate embedding path.

**Nodes Involved:**  
- Fetch Previous Chunk Hashes
- Detect Changed Chunks

### Node: Fetch Previous Chunk Hashes
- **Type and role:** `n8n-nodes-base.postgres`; loads currently active chunk hashes
- **Configuration choices:** Executes:
  - `SELECT chunk_id, content_hash, embedding_version FROM document_chunks WHERE is_active = true`
- **Key expressions/variables:** None
- **Input/output connections:** Input from `Workflow Configuration`; output to `Detect Changed Chunks` input 1
- **Version-specific requirements:** Type version 2.6; requires Postgres credentials
- **Edge cases/failures:** Missing table/columns, DB auth failure, network failure, empty result set
- **Sub-workflow reference:** None

### Node: Detect Changed Chunks
- **Type and role:** `n8n-nodes-base.compareDatasets`; compares current chunks to stored chunks
- **Configuration choices:** Fuzzy comparison enabled; merges by `hash` vs `content_hash`
- **Key expressions/variables:** Field mapping:
  - current field: `hash`
  - previous field: `content_hash`
- **Input/output connections:** 
  - Input 0 from `Chunk Documents & Compute Hash`
  - Input 1 from `Fetch Previous Chunk Hashes`
  - Output to `New Vector Store (Candidate)`
- **Version-specific requirements:** Type version 2.3
- **Edge cases/failures:** 
  - Matching only on hash may miss chunk identity changes
  - Fuzzy compare can produce unintuitive matches
  - Behavior depends on node output mode and may need validation during implementation
- **Sub-workflow reference:** None

---

## 2.4 Candidate Embedding Creation

**Overview:**  
This block prepares changed chunks for embedding, splits them using a LangChain text splitter, computes OpenAI embeddings, and inserts them into an in-memory candidate vector store.

**Nodes Involved:**  
- Recursive Text Splitter
- Document Loader
- OpenAI Embeddings
- New Vector Store (Candidate)

### Node: Recursive Text Splitter
- **Type and role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`; chunk splitter for document loader
- **Configuration choices:** Uses chunk size and overlap from workflow config
- **Key expressions/variables:** 
  - `={{ $('Workflow Configuration').first().json.chunkSize }}`
  - `={{ $('Workflow Configuration').first().json.chunkOverlap }}`
- **Input/output connections:** No main input shown; connected as `ai_textSplitter` to `Document Loader`
- **Version-specific requirements:** Type version 1
- **Edge cases/failures:** Bad config values may cause splitter errors or poor chunk boundaries
- **Sub-workflow reference:** None

### Node: Document Loader
- **Type and role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`; converts incoming items to LangChain documents
- **Configuration choices:** Uses custom text splitting mode
- **Key expressions/variables:** None directly
- **Input/output connections:** Connected by `ai_textSplitter` from `Recursive Text Splitter`; outputs `ai_document` to `New Vector Store (Candidate)`
- **Version-specific requirements:** Type version 1.1
- **Edge cases/failures:** In this JSON there is no main connection into `Document Loader`, so as provided it may never receive actual chunk items
- **Sub-workflow reference:** None

### Node: OpenAI Embeddings
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; embedding model provider
- **Configuration choices:** Default options only; model not explicitly pinned in node parameters
- **Key expressions/variables:** None
- **Input/output connections:** Supplies `ai_embedding` to:
  - `New Vector Store (Candidate)`
  - `Query New Vector Store (Tool)`
  - `Query Old Vector Store (Tool)`
- **Version-specific requirements:** Type version 1.2; requires OpenAI API credentials
- **Edge cases/failures:** API auth failure, rate limits, model deprecation, token/input size constraints
- **Sub-workflow reference:** None

### Node: New Vector Store (Candidate)
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory`; inserts candidate documents into an in-memory vector store
- **Configuration choices:** 
  - Mode: `insert`
  - Memory key: `vector_store_key`
- **Key expressions/variables:** Shared memory key `vector_store_key`
- **Input/output connections:** 
  - Main input from `Detect Changed Chunks`
  - `ai_document` input from `Document Loader`
  - `ai_embedding` input from `OpenAI Embeddings`
  - Main output to `Store Embedding Metadata`
- **Version-specific requirements:** Type version 1.3
- **Edge cases/failures:** 
  - In-memory storage is ephemeral and workflow-execution-scoped
  - Same memory key is reused later for both old and new retrieval tools, which can blur separation
  - If no documents arrive, the candidate store may be empty
- **Sub-workflow reference:** None

---

## 2.5 Candidate Version Metadata Persistence

**Overview:**  
After candidate vectors are prepared, the workflow generates version metadata and stores it in Postgres. This is intended to track model/version/timestamp provenance.

**Nodes Involved:**  
- Store Embedding Metadata
- Save Embedding Version Metadata

### Node: Store Embedding Metadata
- **Type and role:** `n8n-nodes-base.code`; builds metadata record
- **Configuration choices:** Derives:
  - `embeddingModel` default `text-embedding-ada-002`
  - `modelVersion` default `v2`
  - `embeddingTimestamp`
  - `sourceRevision`
  - `chunkCount`
- **Key expressions/variables:** Reads config from `Workflow Configuration`; may fallback to `Fetch Documents from Source.revision`
- **Input/output connections:** Input from `New Vector Store (Candidate)`; output to `Save Embedding Version Metadata`
- **Version-specific requirements:** Type version 2
- **Edge cases/failures:** Output field names are camelCase, but downstream Postgres mapping expects snake_case-like names through expressions that do not match; this is a likely bug
- **Sub-workflow reference:** None

### Node: Save Embedding Version Metadata
- **Type and role:** `n8n-nodes-base.postgres`; inserts candidate version row
- **Configuration choices:** Inserts into `public.embedding_versions` with:
  - `chunk_count = {{ $json.chunk_count }}`
  - `is_candidate = true`
  - `model_version = {{ $json.model_version }}`
  - `embedding_model = {{ $json.embedding_model }}`
  - `source_revision = {{ $json.source_revision }}`
  - `embedding_timestamp = {{ $json.embedding_timestamp }}`
- **Key expressions/variables:** Expressions reference snake_case names
- **Input/output connections:** Input from `Store Embedding Metadata`; outputs to `Fetch Golden Questions` and `Fetch Previous Embeddings`
- **Version-specific requirements:** Type version 2.6; requires Postgres credentials
- **Edge cases/failures:** 
  - Likely null insert values due to field-name mismatch with upstream node
  - DB constraints may reject candidate insert
- **Sub-workflow reference:** None

---

## 2.6 Golden Question Evaluation

**Overview:**  
This block asks benchmark questions against both the new and old embedding contexts. It uses an OpenAI chat model and vector-store retrieval tools attached to AI agents.

**Nodes Involved:**  
- Fetch Golden Questions
- OpenAI Chat Model
- Generate Answers (New)
- Generate Answers (Old)
- Query New Vector Store (Tool)
- Query Old Vector Store (Tool)

### Node: Fetch Golden Questions
- **Type and role:** `n8n-nodes-base.postgres`; retrieves evaluation dataset
- **Configuration choices:** Executes:
  - `SELECT question_id, question_text, expected_passages, expected_answer_keywords FROM golden_questions WHERE is_active = true`
- **Key expressions/variables:** None
- **Input/output connections:** Input from `Save Embedding Version Metadata`; outputs to both answer-generation agents
- **Version-specific requirements:** Type version 2.6
- **Edge cases/failures:** Empty benchmark set causes weak or meaningless quality score; column data types for expected arrays must be compatible
- **Sub-workflow reference:** None

### Node: OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM provider for both agents
- **Configuration choices:** Model `gpt-4.1-mini`
- **Key expressions/variables:** None
- **Input/output connections:** Supplies `ai_languageModel` to `Generate Answers (New)` and `Generate Answers (Old)`
- **Version-specific requirements:** Type version 1.3; requires OpenAI credentials
- **Edge cases/failures:** Auth/rate limit/model access issues
- **Sub-workflow reference:** None

### Node: Generate Answers (New)
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; answers golden questions using new vector store tool
- **Configuration choices:** 
  - Prompt text: `={{ $json.question_text }}`
  - System message instructs retrieval-based detailed answering
  - Prompt type: define
- **Key expressions/variables:** `question_text`
- **Input/output connections:** 
  - Main input from `Fetch Golden Questions`
  - `ai_languageModel` from `OpenAI Chat Model`
  - `ai_tool` from `Query New Vector Store (Tool)`
  - Main output to `Calculate Quality Metrics`
- **Version-specific requirements:** Type version 3
- **Edge cases/failures:** Agent output schema is not normalized to fields expected by later Code node (`answerType`, `expectedPassages`, `retrievedPassages`, etc.); likely requires additional formatting not present
- **Sub-workflow reference:** None

### Node: Generate Answers (Old)
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; answers golden questions using old vector store tool
- **Configuration choices:** Same as new-agent, but intended for old store
- **Key expressions/variables:** `question_text`
- **Input/output connections:** 
  - Main input from `Fetch Golden Questions`
  - `ai_languageModel` from `OpenAI Chat Model`
  - `ai_tool` from `Query Old Vector Store (Tool)`
  - Main output to `Calculate Quality Metrics`
- **Version-specific requirements:** Type version 3
- **Edge cases/failures:** Same schema mismatch risk as above; old store is not actually populated in this workflow
- **Sub-workflow reference:** None

### Node: Query New Vector Store (Tool)
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory`; retrieval tool for new candidate vectors
- **Configuration choices:** 
  - Mode: `retrieve-as-tool`
  - Memory key: `vector_store_key`
  - Description: searches new embeddings
- **Key expressions/variables:** Shared memory key
- **Input/output connections:** 
  - `ai_embedding` from `OpenAI Embeddings`
  - `ai_tool` to `Generate Answers (New)`
- **Version-specific requirements:** Type version 1.3
- **Edge cases/failures:** Works only if same execution contains inserted vectors under same key
- **Sub-workflow reference:** None

### Node: Query Old Vector Store (Tool)
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory`; retrieval tool intended for old vectors
- **Configuration choices:** 
  - Mode: `retrieve-as-tool`
  - Memory key: `vector_store_key`
  - Description: searches old embeddings
- **Key expressions/variables:** Shared memory key identical to new store
- **Input/output connections:** 
  - `ai_embedding` from `OpenAI Embeddings`
  - `ai_tool` to `Generate Answers (Old)`
- **Version-specific requirements:** Type version 1.3
- **Edge cases/failures:** No node loads old embeddings into an in-memory store; using the same memory key as the new store means separation between old/new retrieval is not implemented
- **Sub-workflow reference:** None

---

## 2.7 Quality Scoring & Drift Analysis

**Overview:**  
This block compares answer quality and estimates semantic drift between embedding versions. Its output determines whether the new embeddings can be promoted safely.

**Nodes Involved:**  
- Fetch Previous Embeddings
- Calculate Quality Metrics
- Calculate Embedding Drift
- Quality Improved?

### Node: Fetch Previous Embeddings
- **Type and role:** `n8n-nodes-base.postgres`; retrieves active-version embeddings from database
- **Configuration choices:** Executes:
  - `SELECT embedding_vector, chunk_id FROM embeddings WHERE version_id = (SELECT id FROM embedding_versions WHERE is_active = true LIMIT 1)`
- **Key expressions/variables:** None
- **Input/output connections:** Input from `Save Embedding Version Metadata`; no downstream main connection in this JSON
- **Version-specific requirements:** Type version 2.6
- **Edge cases/failures:** Unused output as currently wired; if table is large, performance may be poor
- **Sub-workflow reference:** None

### Node: Calculate Quality Metrics
- **Type and role:** `n8n-nodes-base.code`; computes aggregate answer-quality metrics
- **Configuration choices:** Calculates:
  - Recall@K
  - Keyword similarity
  - Answer length variance
  - Weighted `qualityScore`
- **Key expressions/variables:** Expects item fields:
  - `answerType` = `new` or `old`
  - `expectedPassages`
  - `retrievedPassages`
  - `expectedKeywords`
  - `answer`
- **Input/output connections:** Inputs from both `Generate Answers (New)` and `Generate Answers (Old)`; output to `Calculate Embedding Drift`
- **Version-specific requirements:** Type version 2
- **Edge cases/failures:** Upstream agents do not produce the required schema in this JSON; pairing by index is fragile; empty answers skew metrics
- **Sub-workflow reference:** None

### Node: Calculate Embedding Drift
- **Type and role:** `n8n-nodes-base.code`; computes cosine-distance-based drift
- **Configuration choices:** 
  - Expects items tagged `version = old|new`
  - Computes average and max drift
  - Uses config `driftThreshold`
- **Key expressions/variables:** `$('Workflow Configuration').first().json.driftThreshold`
- **Input/output connections:** Input from `Calculate Quality Metrics`; output to `Quality Improved?`
- **Version-specific requirements:** Type version 2
- **Edge cases/failures:** As wired, it does not receive old/new embedding vectors, only quality metrics; therefore drift calculation cannot work correctly without additional merge/preparation nodes
- **Sub-workflow reference:** None

### Node: Quality Improved?
- **Type and role:** `n8n-nodes-base.if`; promotion gate
- **Configuration choices:** Both conditions must be true:
  - `qualityScore > qualityThreshold`
  - `driftScore < driftThreshold`
- **Key expressions/variables:** Reads score from current item and thresholds from config
- **Input/output connections:** Input from `Calculate Embedding Drift`; true output to `Promote New Embeddings`; false output to both `Rollback to Previous Embeddings` and `Flag for Human Review`
- **Version-specific requirements:** Type version 2.3
- **Edge cases/failures:** False branch triggers both rollback and manual review simultaneously, which may not match intended business logic
- **Sub-workflow reference:** None

---

## 2.8 Promotion, Rollback, or Manual Review

**Overview:**  
This block updates Postgres according to the evaluation result. The candidate is either promoted, deleted with rollback, or accompanied by a review flag.

**Nodes Involved:**  
- Promote New Embeddings
- Rollback to Previous Embeddings
- Flag for Human Review

### Node: Promote New Embeddings
- **Type and role:** `n8n-nodes-base.postgres`; activates candidate version
- **Configuration choices:** Runs two SQL updates:
  1. Set latest candidate to `is_active = true, is_candidate = false`
  2. Set other active non-candidates to `is_active = false`
- **Key expressions/variables:** None
- **Input/output connections:** Input from true branch of `Quality Improved?`; output to `Send Notification`
- **Version-specific requirements:** Type version 2.6
- **Edge cases/failures:** Multi-statement execution must be allowed; ordering of updates may temporarily mark multiple rows active depending on DB behavior
- **Sub-workflow reference:** None

### Node: Rollback to Previous Embeddings
- **Type and role:** `n8n-nodes-base.postgres`; deletes candidate and reactivates previous version
- **Configuration choices:** Runs:
  1. `DELETE FROM embedding_versions WHERE is_candidate = true`
  2. Reactivate most recent inactive version
- **Key expressions/variables:** None
- **Input/output connections:** Input from false branch of `Quality Improved?`; output to `Send Notification`
- **Version-specific requirements:** Type version 2.6
- **Edge cases/failures:** If there is no previous inactive version, rollback activation does nothing; deletes all candidates, not only current one
- **Sub-workflow reference:** None

### Node: Flag for Human Review
- **Type and role:** `n8n-nodes-base.set`; creates review payload
- **Configuration choices:** Sets:
  - `status = NEEDS_REVIEW`
  - `reason = Quality metrics ambiguous - manual review required`
  - `qualityScore`
  - `driftScore`
  - `stickyNote = Results unclear - needs human to decide`
- **Key expressions/variables:** Uses current JSON scores
- **Input/output connections:** Input from false branch of `Quality Improved?`; output to `Send Notification`
- **Version-specific requirements:** Type version 3.4
- **Edge cases/failures:** Because rollback already executes in parallel on same false branch, review may describe a situation that has already been reverted
- **Sub-workflow reference:** None

---

## 2.9 Notification

**Overview:**  
This final block sends a summary status to an external webhook. It is intended for Slack, Teams, or any compatible endpoint.

**Nodes Involved:**  
- Send Notification

### Node: Send Notification
- **Type and role:** `n8n-nodes-base.httpRequest`; outbound notification sender
- **Configuration choices:** POST to configured webhook with body parameters:
  - `status`
  - `message`
  - `qualityScore`
  - `driftScore`
  - `timestamp`
- **Key expressions/variables:** 
  - URL: `={{ $('Workflow Configuration').first().json.notificationWebhook }}`
  - Status: `={{ $('Quality Improved?').item.json.status || 'unknown' }}`
  - Message chooses among promote/rollback/manual-review based on available items
- **Input/output connections:** Inputs from `Promote New Embeddings`, `Rollback to Previous Embeddings`, and `Flag for Human Review`
- **Version-specific requirements:** Type version 4.3
- **Edge cases/failures:** 
  - `Quality Improved?` does not output a `status` field, so status likely resolves to `unknown`
  - Expression logic based on `.item` existence can be brittle
  - Webhook payload format may not match Slack/Teams expected schema
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily RAG Maintenance Schedule | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point |  | Workflow Configuration | ## Workflow Trigger & Config<br>This section starts the workflow via schedule or webhook trigger.<br>It defines key parameters such as document source URL, chunk size, overlap, quality threshold, drift threshold, and notification webhook used throughout the workflow. |
| Source Change Webhook | n8n-nodes-base.webhook | Event-driven workflow entry point |  | Workflow Configuration | ## Workflow Trigger & Config<br>This section starts the workflow via schedule or webhook trigger.<br>It defines key parameters such as document source URL, chunk size, overlap, quality threshold, drift threshold, and notification webhook used throughout the workflow. |
| Workflow Configuration | n8n-nodes-base.set | Central runtime configuration | Daily RAG Maintenance Schedule, Source Change Webhook | Fetch Documents from Source, Fetch Previous Chunk Hashes | ## Workflow Trigger & Config<br>This section starts the workflow via schedule or webhook trigger.<br>It defines key parameters such as document source URL, chunk size, overlap, quality threshold, drift threshold, and notification webhook used throughout the workflow. |
| Fetch Documents from Source | n8n-nodes-base.httpRequest | Fetch source documents | Workflow Configuration | Chunk Documents & Compute Hash | ## Document Retrieval & Chunking<br>Documents are fetched from the configured source and split into deterministic chunks.<br>Each chunk receives a SHA-256 hash to detect content changes and ensure only modified chunks are reprocessed. |
| Chunk Documents & Compute Hash | n8n-nodes-base.code | Deterministic chunking and hashing | Fetch Documents from Source | Detect Changed Chunks | ## Document Retrieval & Chunking<br>Documents are fetched from the configured source and split into deterministic chunks.<br>Each chunk receives a SHA-256 hash to detect content changes and ensure only modified chunks are reprocessed. |
| Fetch Previous Chunk Hashes | n8n-nodes-base.postgres | Load active chunk hash baseline | Workflow Configuration | Detect Changed Chunks | ##Chunk Change Detection<br>New document chunk hashes are compared with hashes stored in Postgres. |
| Detect Changed Chunks | n8n-nodes-base.compareDatasets | Compare new and old chunk hashes | Chunk Documents & Compute Hash, Fetch Previous Chunk Hashes | New Vector Store (Candidate) | ##Chunk Change Detection<br>New document chunk hashes are compared with hashes stored in Postgres. |
| Recursive Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split documents for embedding |  | Document Loader | ## Embedding Creation<br>Changed document chunks are embedded using OpenAI embeddings. |
| Document Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Convert items into LangChain documents | Recursive Text Splitter (AI connector) | New Vector Store (Candidate) | ## Embedding Creation<br>Changed document chunks are embedded using OpenAI embeddings. |
| OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embedding model provider |  | New Vector Store (Candidate), Query New Vector Store (Tool), Query Old Vector Store (Tool) | ## OpenAI Embeddings |
| New Vector Store (Candidate) | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Insert candidate vectors into in-memory store | Detect Changed Chunks, Document Loader, OpenAI Embeddings | Store Embedding Metadata | ## Embedding Creation<br>Changed document chunks are embedded using OpenAI embeddings. |
| Store Embedding Metadata | n8n-nodes-base.code | Build candidate version metadata | New Vector Store (Candidate) | Save Embedding Version Metadata | ## Golden Question Evaluation<br>The system runs predefined golden questions against both the new and old vector stores.<br>AI agents generate answers using retrieved context so the workflow can evaluate retrieval and answer quality. |
| Save Embedding Version Metadata | n8n-nodes-base.postgres | Persist candidate embedding version row | Store Embedding Metadata | Fetch Golden Questions, Fetch Previous Embeddings | ## Golden Question Evaluation<br>The system runs predefined golden questions against both the new and old vector stores.<br>AI agents generate answers using retrieved context so the workflow can evaluate retrieval and answer quality. |
| Fetch Golden Questions | n8n-nodes-base.postgres | Load benchmark evaluation questions | Save Embedding Version Metadata | Generate Answers (New), Generate Answers (Old) | ## Golden Question Evaluation<br>The system runs predefined golden questions against both the new and old vector stores.<br>AI agents generate answers using retrieved context so the workflow can evaluate retrieval and answer quality. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for answer generation |  | Generate Answers (New), Generate Answers (Old) |  |
| Generate Answers (New) | @n8n/n8n-nodes-langchain.agent | Generate benchmark answers with new embeddings | Fetch Golden Questions, OpenAI Chat Model, Query New Vector Store (Tool) | Calculate Quality Metrics | ## RAG Answer Generation<br>This agents answer the same questions using different vector stores.<br>One agent queries the new candidate embeddings. |
| Query New Vector Store (Tool) | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Retrieval tool for new embeddings | OpenAI Embeddings | Generate Answers (New) | ## Vector Store Retrieval Tools<br>These vector store tools allow AI agents to retrieve relevant context from embeddings.<br>Query New Vector Store searches candidate embeddings. |
| Generate Answers (Old) | @n8n/n8n-nodes-langchain.agent | Generate benchmark answers with old embeddings | Fetch Golden Questions, OpenAI Chat Model, Query Old Vector Store (Tool) | Calculate Quality Metrics | ## RAG Answer Generation<br>This agents answer the same questions using existing vector stores.<br>One agent queries the old candidate embeddings |
| Query Old Vector Store (Tool) | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Retrieval tool intended for old embeddings | OpenAI Embeddings | Generate Answers (Old) |  |
| Fetch Previous Embeddings | n8n-nodes-base.postgres | Load active embeddings for drift comparison | Save Embedding Version Metadata |  | ## Golden Question Evaluation<br>The system runs predefined golden questions against both the new and old vector stores.<br>AI agents generate answers using retrieved context so the workflow can evaluate retrieval and answer quality. |
| Calculate Quality Metrics | n8n-nodes-base.code | Compute retrieval and answer quality metrics | Generate Answers (New), Generate Answers (Old) | Calculate Embedding Drift | ## Retrieval Quality Evaluation<br>This step evaluates answers generated from the new and old embeddings.<br>Metrics calculated include:<br>• Recall@K for retrieved passages<br>• Keyword similarity in answers<br>• Answer length variance<br><br>These metrics are combined into a weighted quality score used to determine if the new embeddings improve RAG performance. |
| Calculate Embedding Drift | n8n-nodes-base.code | Compute semantic drift score | Calculate Quality Metrics | Quality Improved? | ## Retrieval Quality Evaluation<br>This step evaluates answers generated from the new and old embeddings.<br>Metrics calculated include:<br>• Recall@K for retrieved passages<br>• Keyword similarity in answers<br>• Answer length variance<br><br>These metrics are combined into a weighted quality score used to determine if the new embeddings improve RAG performance. |
| Quality Improved? | n8n-nodes-base.if | Decide promotion vs rollback/review | Calculate Embedding Drift | Promote New Embeddings, Rollback to Previous Embeddings, Flag for Human Review | ## compare scores |
| Promote New Embeddings | n8n-nodes-base.postgres | Activate candidate embedding version | Quality Improved? | Send Notification | ## Deployment Outcome<br>Based on evaluation results, the workflow either promotes the new embeddings, rolls back to the previous version, or flags the update for manual review. |
| Rollback to Previous Embeddings | n8n-nodes-base.postgres | Delete candidate and reactivate prior version | Quality Improved? | Send Notification | ## Deployment Outcome<br>Based on evaluation results, the workflow either promotes the new embeddings, rolls back to the previous version, or flags the update for manual review. |
| Flag for Human Review | n8n-nodes-base.set | Prepare manual review status payload | Quality Improved? | Send Notification | ## Deployment Outcome<br>Based on evaluation results, the workflow either promotes the new embeddings, rolls back to the previous version, or flags the update for manual review. |
| Send Notification | n8n-nodes-base.httpRequest | Notify external system of outcome | Promote New Embeddings, Rollback to Previous Embeddings, Flag for Human Review |  | A notification is then sent through the configured webhook to inform the team about the update status. |
| Sticky Note | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note14 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note13 | n8n-nodes-base.stickyNote | Annotation |  |  |  |
| Sticky Note15 | n8n-nodes-base.stickyNote | Annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is the exact build order to recreate the workflow in n8n, while also highlighting where the provided design likely needs fixes.

1. **Create a Schedule Trigger node**
   - Type: `Schedule Trigger`
   - Set it to run daily at 02:00
   - Name it `Daily RAG Maintenance Schedule`

2. **Create a Webhook node**
   - Type: `Webhook`
   - HTTP method: `POST`
   - Path: `rag-update`
   - Name it `Source Change Webhook`

3. **Create a Set node for global configuration**
   - Type: `Set`
   - Name: `Workflow Configuration`
   - Enable keeping other fields if desired
   - Add fields:
     - `documentSourceUrl` (string)
     - `chunkSize` (number, default `1000`)
     - `chunkOverlap` (number, default `200`)
     - `qualityThreshold` (number, default `0.85`)
     - `driftThreshold` (number, default `0.15`)
     - `notificationWebhook` (string)
   - Optional but recommended additions, because later code expects them:
     - `sourceRevision`
     - `embeddingModel`
     - `modelVersion`

4. **Connect both triggers to Workflow Configuration**
   - `Daily RAG Maintenance Schedule` → `Workflow Configuration`
   - `Source Change Webhook` → `Workflow Configuration`

5. **Create an HTTP Request node to fetch source documents**
   - Type: `HTTP Request`
   - Name: `Fetch Documents from Source`
   - URL: `={{ $('Workflow Configuration').first().json.documentSourceUrl }}`
   - Response format: JSON
   - Add authentication if your source requires it
   - Connect `Workflow Configuration` → `Fetch Documents from Source`

6. **Create a Postgres node to fetch previous chunk hashes**
   - Type: `Postgres`
   - Name: `Fetch Previous Chunk Hashes`
   - Operation: `Execute Query`
   - Query:
     `SELECT chunk_id, content_hash, embedding_version FROM document_chunks WHERE is_active = true`
   - Configure Postgres credentials
   - Connect `Workflow Configuration` → `Fetch Previous Chunk Hashes`

7. **Create a Code node for chunking and hashing**
   - Type: `Code`
   - Name: `Chunk Documents & Compute Hash`
   - Paste logic equivalent to:
     - read chunk size/overlap from config
     - load all documents
     - pick `content` or `text` or JSON string
     - split deterministically
     - compute SHA-256 for each chunk
     - emit `chunkId`, `content`, `hash`, `chunkSize`, `chunkOverlap`, `sourceRevision`, `timestamp`
   - Important constraint: ensure `chunkOverlap < chunkSize`
   - Connect `Fetch Documents from Source` → `Chunk Documents & Compute Hash`

8. **Create a Compare Datasets node**
   - Type: `Compare Datasets`
   - Name: `Detect Changed Chunks`
   - Configure merge-by fields:
     - input 1 field: `hash`
     - input 2 field: `content_hash`
   - Enable fuzzy compare if you want to mirror the JSON
   - Connect:
     - `Chunk Documents & Compute Hash` → input 0
     - `Fetch Previous Chunk Hashes` → input 1

9. **Create a Recursive Character Text Splitter**
   - Type: `Recursive Character Text Splitter`
   - Name: `Recursive Text Splitter`
   - Chunk size: `={{ $('Workflow Configuration').first().json.chunkSize }}`
   - Chunk overlap: `={{ $('Workflow Configuration').first().json.chunkOverlap }}`

10. **Create a Document Default Data Loader**
    - Type: `Document Default Data Loader`
    - Name: `Document Loader`
    - Text splitting mode: `custom`
    - Connect AI text splitter:
      - `Recursive Text Splitter` → `Document Loader`

11. **Important correction step: connect changed chunks into Document Loader**
    - The provided JSON does not connect `Detect Changed Chunks` to `Document Loader`
    - To make the workflow functional, add a main connection:
      - `Detect Changed Chunks` → `Document Loader`
    - Ensure the loader uses the chunk text field as document content

12. **Create an OpenAI Embeddings node**
    - Type: `OpenAI Embeddings`
    - Name: `OpenAI Embeddings`
    - Add OpenAI API credentials
    - Choose an embedding model explicitly instead of relying on defaults

13. **Create an In-Memory Vector Store node for candidate insertion**
    - Type: `Vector Store In-Memory`
    - Name: `New Vector Store (Candidate)`
    - Mode: `insert`
    - Memory key: `vector_store_key`
    - Connect:
      - Main: `Detect Changed Chunks` or preferably `Document Loader` output path as required by your version
      - AI document: `Document Loader` → `New Vector Store (Candidate)`
      - AI embedding: `OpenAI Embeddings` → `New Vector Store (Candidate)`

14. **Create a Code node for candidate version metadata**
    - Type: `Code`
    - Name: `Store Embedding Metadata`
    - Build one metadata record with:
      - `embedding_model`
      - `model_version`
      - `embedding_timestamp`
      - `source_revision`
      - `chunk_count`
    - Recommended: output snake_case keys to match the next Postgres node exactly
    - Connect `New Vector Store (Candidate)` → `Store Embedding Metadata`

15. **Create a Postgres node to save version metadata**
    - Type: `Postgres`
    - Name: `Save Embedding Version Metadata`
    - Table: `public.embedding_versions`
    - Insert columns:
      - `chunk_count`
      - `is_candidate = true`
      - `model_version`
      - `embedding_model`
      - `source_revision`
      - `embedding_timestamp`
    - Connect `Store Embedding Metadata` → `Save Embedding Version Metadata`

16. **Create a Postgres node for golden questions**
    - Type: `Postgres`
    - Name: `Fetch Golden Questions`
    - Query:
      `SELECT question_id, question_text, expected_passages, expected_answer_keywords FROM golden_questions WHERE is_active = true`
    - Connect `Save Embedding Version Metadata` → `Fetch Golden Questions`

17. **Create a Postgres node for previous embeddings**
    - Type: `Postgres`
    - Name: `Fetch Previous Embeddings`
    - Query:
      `SELECT embedding_vector, chunk_id FROM embeddings WHERE version_id = (SELECT id FROM embedding_versions WHERE is_active = true LIMIT 1)`
    - Connect `Save Embedding Version Metadata` → `Fetch Previous Embeddings`

18. **Create an OpenAI Chat Model node**
    - Type: `OpenAI Chat Model`
    - Name: `OpenAI Chat Model`
    - Model: `gpt-4.1-mini`
    - Reuse OpenAI credentials

19. **Create a vector store retrieval tool for new embeddings**
    - Type: `Vector Store In-Memory`
    - Name: `Query New Vector Store (Tool)`
    - Mode: `retrieve-as-tool`
    - Memory key: `vector_store_key`
    - Tool description: describe it as querying new candidate embeddings
    - Connect AI embedding:
      - `OpenAI Embeddings` → `Query New Vector Store (Tool)`

20. **Create a second retrieval mechanism for old embeddings**
    - The JSON uses another in-memory vector store with the same memory key, which is not enough to represent an old store
    - Recommended fix:
      - either create a second separately populated in-memory vector store with a different key
      - or use a real persistent vector DB/tool for the active version
    - If reproducing literally:
      - create `Query Old Vector Store (Tool)`
      - mode `retrieve-as-tool`
      - memory key `vector_store_key`
    - This literal build will not actually separate old vs new retrieval contexts

21. **Create the agent for new embeddings**
    - Type: `AI Agent`
    - Name: `Generate Answers (New)`
    - Prompt text: `={{ $json.question_text }}`
    - System message: instruct the agent to answer using retrieval tool context
    - Connect:
      - Main: `Fetch Golden Questions` → `Generate Answers (New)`
      - AI model: `OpenAI Chat Model` → `Generate Answers (New)`
      - AI tool: `Query New Vector Store (Tool)` → `Generate Answers (New)`

22. **Create the agent for old embeddings**
    - Type: `AI Agent`
    - Name: `Generate Answers (Old)`
    - Same prompt/system message
    - Connect:
      - Main: `Fetch Golden Questions` → `Generate Answers (Old)`
      - AI model: `OpenAI Chat Model` → `Generate Answers (Old)`
      - AI tool: `Query Old Vector Store (Tool)` → `Generate Answers (Old)`

23. **Add normalization after each agent output**
    - This is required for a working build, though absent in the JSON
    - Add a Set or Code node after each agent to standardize fields:
      - `answerType` = `new` or `old`
      - `answer`
      - `expectedPassages` from `expected_passages`
      - `expectedKeywords` from `expected_answer_keywords`
      - `retrievedPassages` if available from tool traces
    - Without this, `Calculate Quality Metrics` will not work correctly

24. **Create a Code node for quality metrics**
    - Type: `Code`
    - Name: `Calculate Quality Metrics`
    - Implement:
      - split items by `answerType`
      - compare retrieved passages to expected passages
      - compute keyword similarity
      - compute answer length variance
      - output `qualityScore`, `recallAtK`, `similarityScore`, `answerLengthVariance`
    - Connect normalized new and old answer branches into this node

25. **Add a proper drift-preparation step**
    - The provided workflow sends quality metrics directly into drift analysis, but drift analysis expects embedding vectors
    - Add merge/preparation nodes that combine:
      - old embeddings from `Fetch Previous Embeddings`
      - new embeddings from your candidate embedding output or persisted embedding table
      - tag each item as `version = old` or `version = new`

26. **Create a Code node for embedding drift**
    - Type: `Code`
    - Name: `Calculate Embedding Drift`
    - Implement cosine distance on paired vectors
    - Output:
      - `driftScore`
      - `maxDrift`
      - `avgDrift`
      - `driftThreshold`
      - `driftDetected`
    - Ensure it also carries forward `qualityScore`, or merge it with metrics before the IF node

27. **Create an IF node**
    - Type: `If`
    - Name: `Quality Improved?`
    - Conditions:
      - `qualityScore > qualityThreshold`
      - `driftScore < driftThreshold`
    - Connect `Calculate Embedding Drift` → `Quality Improved?`

28. **Create a Postgres node for promotion**
    - Type: `Postgres`
    - Name: `Promote New Embeddings`
    - Query:
      - activate latest candidate
      - deactivate previous active
    - Connect true branch of `Quality Improved?` → `Promote New Embeddings`

29. **Create a Postgres node for rollback**
    - Type: `Postgres`
    - Name: `Rollback to Previous Embeddings`
    - Query:
      - delete candidate version
      - reactivate previous inactive version
    - Connect false branch of `Quality Improved?` → `Rollback to Previous Embeddings`

30. **Create a Set node for human review**
    - Type: `Set`
    - Name: `Flag for Human Review`
    - Fields:
      - `status = NEEDS_REVIEW`
      - `reason`
      - `qualityScore`
      - `driftScore`
      - review message
    - Connect false branch of `Quality Improved?` → `Flag for Human Review`

31. **Recommended correction for branching**
    - The literal JSON causes both rollback and review on the false path
    - Better design:
      - add another IF node to distinguish hard failure vs ambiguous result
      - rollback only when clearly bad
      - human review only when borderline

32. **Create an HTTP Request node for notifications**
    - Type: `HTTP Request`
    - Name: `Send Notification`
    - Method: `POST`
    - URL: `={{ $('Workflow Configuration').first().json.notificationWebhook }}`
    - Body parameters:
      - `status`
      - `message`
      - `qualityScore`
      - `driftScore`
      - `timestamp`
    - Connect from `Promote New Embeddings`, `Rollback to Previous Embeddings`, and `Flag for Human Review`

33. **Credential setup**
    - **OpenAI credentials:** required for `OpenAI Embeddings` and `OpenAI Chat Model`
    - **Postgres credentials:** required for all Postgres nodes
    - **Optional source auth credentials:** required if `documentSourceUrl` is protected
    - **Notification webhook:** must accept generic JSON payloads or be adapted to target format

34. **Database prerequisites**
    - Create or adapt tables:
      - `document_chunks`
      - `embedding_versions`
      - `embeddings`
      - `golden_questions`
    - Ensure expected columns exist and naming matches node expressions

35. **Validation before production**
    - Test each entry point separately
    - Test with no document changes
    - Test with source API failure
    - Test with empty golden question set
    - Confirm old/new retrieval stores are truly distinct
    - Confirm notification payload format with Slack/Teams receiver

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow maintains a self-healing Retrieval-Augmented Generation (RAG) system by automatically updating document embeddings, evaluating quality, detecting embedding drift, and safely promoting or rolling back model updates. | General purpose summary |
| The workflow runs on a daily schedule or webhook trigger when source documents change. Documents are fetched from a configured source, chunked deterministically, and hashed to detect modified content. | Operational overview |
| Only changed chunks are embedded using OpenAI embeddings and stored as a candidate embedding version. | Embedding lifecycle |
| The system then evaluates the candidate embeddings by asking a set of golden test questions. Answers generated using the new embeddings are compared against answers from the current active embeddings. | Evaluation design |
| Quality metrics such as Recall@K, keyword similarity, and answer variance are calculated. The workflow also measures embedding drift using cosine distance between embedding vectors. | Scoring logic |
| If quality improves and drift is acceptable, the candidate embeddings are promoted. Otherwise, the system rolls back to the previous embeddings or flags the update for manual review. | Deployment decision logic |

## Important implementation caveats

This workflow is conceptually strong, but as provided it contains several structural issues that should be corrected before production use:

1. **Document Loader is not fed by a main input connection.**
2. **Candidate metadata field names do not match the Postgres insert expressions.**
3. **The “old vector store” is not actually populated separately from the new one.**
4. **Quality metric code expects fields not produced by the agent nodes.**
5. **Drift calculation expects embedding vectors but receives quality output instead.**
6. **The false branch triggers both rollback and manual review at the same time.**
7. **Notification status references a field not emitted by the IF node.**

If you want, I can also provide:
- a **corrected architecture proposal**
- a **database schema suggestion for the Postgres tables**
- or a **fixed n8n build plan matching the intended logic rather than the literal JSON**.