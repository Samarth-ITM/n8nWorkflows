Build a RAG chat system using Aryn DocParse, AWS S3, Pinecone and GPT-4o

https://n8nworkflows.xyz/workflows/build-a-rag-chat-system-using-aryn-docparse--aws-s3--pinecone-and-gpt-4o-12531


# Build a RAG chat system using Aryn DocParse, AWS S3, Pinecone and GPT-4o

## 1. Workflow Overview

**Workflow name:** `ArynPineconeChatWithData`  
**Purpose:** Build a Retrieval-Augmented Generation (RAG) chat system by ingesting documents from **AWS S3**, parsing them with **Aryn DocParse**, chunking and embedding the extracted content with **OpenAI embeddings**, storing vectors in **Pinecone**, then answering chat questions via an **AI Agent** using a **Pinecone retrieval tool** and an **OpenAI chat model**.

**Typical use cases**
- Ask questions over a private set of PDFs / Word docs stored in S3.
- Extract text/tables/properties via Aryn, store as searchable chunks in Pinecone, and chat over them.

### 1.1 Document Ingestion (S3 → Aryn → Chunk → Embed → Pinecone)
Triggered manually. Lists files in S3, downloads them one-by-one, parses via Aryn, then stores parsed content into Pinecone using OpenAI embeddings.

### 1.2 RAG Chat (Chat Trigger → Agent → Pinecone Tool → OpenAI Model)
Triggered by incoming chat messages. The agent uses Pinecone as a tool (topK=100) to retrieve relevant chunks, then answers using an OpenAI chat model (`gpt-4o-mini`).

---

## 2. Block-by-Block Analysis

### Block A — Entry + S3 File Listing + Iteration
**Overview:** Starts ingestion manually, lists all files in an S3 “folder”, then iterates through them to download each file for parsing.

**Nodes involved**
- When clicking ‘Test workflow’
- Get Files from S3
- Loop Over Items
- Download Files from AWS

#### Node: When clicking ‘Test workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts ingestion on demand.
- **Configuration:** No parameters.
- **Outputs:** Sends a single empty item to the next node.
- **Connections:**  
  - **Output →** Get Files from S3
- **Edge cases:** None (only manual execution).

#### Node: Get Files from S3
- **Type / role:** AWS S3 (`n8n-nodes-base.awsS3`) — lists objects in a bucket/prefix.
- **Configuration (interpreted):**
  - **Operation:** `getAll`
  - **Folder key:** empty string (`""`) meaning root of the bucket unless the credential/node UI specifies a default prefix.
- **Credentials:** `AWS account` (must allow `s3:ListBucket` and possibly `s3:GetObject` depending on how the node operates).
- **Outputs:** Items each containing metadata including `Key` (used downstream).
- **Connections:**  
  - **Input ←** Manual Trigger  
  - **Output →** Loop Over Items
- **Edge cases / failures:**
  - Access denied (bad IAM policy / wrong bucket).
  - Wrong region/bucket configuration in credentials.
  - Large buckets: listing can be slow or paginated depending on node behavior.

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates through S3 file list.
- **Configuration (interpreted):**
  - **Batch size expression:** `={{ $json.Key.length }}`
  - This expression uses the current item’s `Key` string length, which is unusual for batching. Practically, it may create unpredictable batch sizes (long keys → large batch size). In many workflows, batch size is a fixed number like `1` for strict sequential downloads.
- **Connections:**  
  - **Input ←** Get Files from S3  
  - **Output (batch) →** *(none)*  
  - **Output (next batch / continue) →** Download Files from AWS
- **Edge cases / failures:**
  - If `Key` is missing, the expression can fail.
  - If `Key.length` evaluates to `0` or very large, behavior may be unexpected.
  - If you intended “one file at a time”, set **Batch Size = 1**.

#### Node: Download Files from AWS
- **Type / role:** AWS S3 (`n8n-nodes-base.awsS3`) — downloads each file’s content.
- **Configuration (interpreted):**
  - **fileKey:** `={{ $json.Key }}`
- **Credentials:** `AWS account` (requires `s3:GetObject` for the relevant keys).
- **Outputs:** Binary data (file contents) plus metadata.
- **Connections:**  
  - **Input ←** Loop Over Items  
  - **Output →** Aryn
- **Edge cases / failures:**
  - Missing key / object deleted after listing.
  - Large files causing memory/time limits.
  - Permission denied on some objects.

---

### Block B — Document Parsing + Chunking + Embedding + Vector Insertion
**Overview:** Parses downloaded documents with Aryn DocParse, then inserts extracted content into Pinecone. Chunking is handled via a text splitter + default loader, and embeddings come from OpenAI.

**Nodes involved**
- Aryn
- Pinecone Vector Store
- Recursive Character Text Splitter
- Default Data Loader
- Embeddings OpenAI

#### Node: Aryn
- **Type / role:** Aryn DocParse node (`@aryn-ai/n8n-nodes-aryn.aryn`) — extracts text/tables/properties from documents.
- **Configuration (interpreted):**
  - No explicit options set in JSON, so default parsing behavior is used unless defaults are configured in the node UI.
  - Consumes the downloaded file (typically via binary input).
- **Credentials:** `Aryn account` (API key).
- **Outputs:** Parsed document content (structured extraction).
- **Connections:**  
  - **Input ←** Download Files from AWS  
  - **Output →** Pinecone Vector Store
- **Edge cases / failures:**
  - API key invalid / quota exceeded.
  - OCR/table extraction may require paid tier (noted in sticky note).
  - Unsupported file types or corrupted documents.
  - Large documents → long processing times.

#### Node: Recursive Character Text Splitter
- **Type / role:** LangChain Text Splitter (`@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`) — chunks extracted text for embedding.
- **Configuration (interpreted):**
  - Uses recursive character splitting.
  - **splitCode:** `python` (biases how code blocks are treated; useful if docs contain code).
- **Connections:**  
  - **Output (ai_textSplitter) →** Default Data Loader
- **Important note:** This node is not connected from the Aryn output in the “main” graph. It is used as an **AI sub-connection** into the data loader (see next node). The actual text to split is supplied through the vector store insertion pipeline (below).
- **Edge cases / failures:**
  - Empty documents → zero chunks.
  - Very large single fields → high chunk counts and embedding cost.

#### Node: Default Data Loader
- **Type / role:** LangChain Document Loader (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) — normalizes content into LangChain “Document” objects and applies the splitter.
- **Configuration:** Defaults (no options specified).
- **Connections:**  
  - **Input (ai_textSplitter) ←** Recursive Character Text Splitter  
  - **Output (ai_document) →** Pinecone Vector Store
- **Edge cases / failures:**
  - If upstream parsed data is not in a format it can load, it may emit no documents or error.
  - Metadata handling differences depending on node version.

#### Node: Embeddings OpenAI
- **Type / role:** OpenAI embeddings provider (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`) — produces vectors for chunked documents.
- **Configuration:** Defaults; model not explicitly specified (node default applies).
- **Credentials:** `OpenAi account`.
- **Connections:**  
  - **Output (ai_embedding) →** Pinecone Vector Store  
  - **Output (ai_embedding) →** Pinecone Vector Store Tool
- **Edge cases / failures:**
  - Invalid API key / rate limits.
  - Cost increases with number/size of chunks.
  - If default embeddings model changes, Pinecone index dimension must match.

#### Node: Pinecone Vector Store
- **Type / role:** Pinecone vector store (`@n8n/n8n-nodes-langchain.vectorStorePinecone`) — inserts vectors into Pinecone.
- **Configuration (interpreted):**
  - **Mode:** `insert`
  - **Index:** `n8n`
- **Credentials:** `PineconeApi account`.
- **Connections:**  
  - **Input (main) ←** Aryn  
  - **Input (ai_document) ←** Default Data Loader  
  - **Input (ai_embedding) ←** Embeddings OpenAI
- **How it works together:** Aryn provides parsed content; Default Data Loader + Splitter create documents/chunks; OpenAI embeddings converts chunks to vectors; this node writes them to the Pinecone index.
- **Edge cases / failures:**
  - Pinecone index not found, wrong environment/region, invalid API key.
  - Dimension mismatch between embeddings model and Pinecone index configuration.
  - Upsert payload too large if too many chunks in one execution.

---

### Block C — RAG Chat Runtime (Trigger → Agent → Retrieval Tool → LLM)
**Overview:** Listens for chat messages, routes them to an AI Agent configured with an OpenAI chat model and a Pinecone retrieval tool, then returns grounded answers based on retrieved chunks.

**Nodes involved**
- When chat message received
- AI Agent
- OpenAI Chat Model
- Pinecone Vector Store Tool
- Embeddings OpenAI (shared dependency)

#### Node: When chat message received
- **Type / role:** LangChain Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) — webhook-like trigger for chat messages.
- **Configuration:** Defaults; has a webhookId internally.
- **Outputs:** Chat message payload to the agent.
- **Connections:**  
  - **Output →** AI Agent
- **Edge cases / failures:**
  - If used in n8n Chat UI, ensure workflow is active and accessible.
  - Payload structure changes across node versions may require prompt/tool adjustments.

#### Node: AI Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates tool calls and final answer generation.
- **Configuration:** Defaults (no custom system prompt shown in JSON).
- **Connections:**  
  - **Input (main) ←** When chat message received  
  - **Input (ai_languageModel) ←** OpenAI Chat Model  
  - **Input (ai_tool) ←** Pinecone Vector Store Tool
- **Edge cases / failures:**
  - If no relevant docs exist in Pinecone, answers may be generic.
  - Without explicit agent instructions, tool usage may be inconsistent; consider adding guidance (system message) if needed.

#### Node: OpenAI Chat Model
- **Type / role:** Chat LLM (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — generates final responses and/or tool reasoning.
- **Configuration (interpreted):**
  - **Model:** `gpt-4o-mini`
  - Other options default.
- **Credentials:** `OpenAi account`.
- **Connections:**  
  - **Output (ai_languageModel) →** AI Agent
- **Edge cases / failures:**
  - Rate limits, auth errors.
  - Model availability changes.

#### Node: Pinecone Vector Store Tool
- **Type / role:** Pinecone retrieval tool (`@n8n/n8n-nodes-langchain.vectorStorePinecone`) — exposes Pinecone similarity search as an agent tool.
- **Configuration (interpreted):**
  - **Mode:** `retrieve-as-tool`
  - **Index:** `n8n`
  - **topK:** `100`
  - **Tool description:** “Contains data about Pinecone releases.”
- **Credentials:** `PineconeApi account`.
- **Dependencies:** Receives embeddings from **Embeddings OpenAI** (used for query embedding).
- **Connections:**  
  - **Input (ai_embedding) ←** Embeddings OpenAI  
  - **Output (ai_tool) →** AI Agent
- **Edge cases / failures:**
  - Same Pinecone issues as insertion (index, auth, region).
  - topK=100 can increase latency/cost; may return too much context.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Test workflow’ | Manual Trigger | Manual start for ingestion | — | Get Files from S3 | ## Build an awesome RAG system using Aryn DocParse to extract text, images, tables and properties from your documents in your ingestion pipeline… (includes setup steps + links: https://docs.aryn.ai/docparse/processing_options, https://aryn.ai/signup, https://pinecone.io, https://openai.com) |
| Get Files from S3 | AWS S3 | List documents in bucket/prefix | When clicking ‘Test workflow’ | Loop Over Items | ## Build an awesome RAG system… (same note, includes links) / ## Document parsing and ingestion |
| Loop Over Items | Split In Batches | Iterate over S3 objects | Get Files from S3 | Download Files from AWS | ## Document parsing and ingestion |
| Download Files from AWS | AWS S3 | Download each S3 object | Loop Over Items | Aryn | ## Document parsing and ingestion |
| Aryn | Aryn DocParse | Parse docs (text/tables/properties) | Download Files from AWS | Pinecone Vector Store | ## Document parsing and ingestion |
| Recursive Character Text Splitter | LangChain Text Splitter | Chunk documents for embedding | (AI-linked) | Default Data Loader (ai_textSplitter) | ## Document parsing and ingestion |
| Default Data Loader | LangChain Document Loader | Convert content to documents + apply splitter | Recursive Character Text Splitter (ai_textSplitter) | Pinecone Vector Store (ai_document) | ## Document parsing and ingestion |
| Embeddings OpenAI | OpenAI Embeddings | Create embeddings for insert + retrieval | — | Pinecone Vector Store Tool; Pinecone Vector Store | ## Document parsing and ingestion / ## RAG |
| Pinecone Vector Store | Pinecone Vector Store | Insert vectors into Pinecone index | Aryn (main); Default Data Loader (ai_document); Embeddings OpenAI (ai_embedding) | — | ## Document parsing and ingestion |
| When chat message received | Chat Trigger | Entry point for chat Q&A | — | AI Agent | ## RAG |
| Pinecone Vector Store Tool | Pinecone Vector Store (Tool) | Retrieval tool (similarity search) | Embeddings OpenAI (ai_embedding) | AI Agent (ai_tool) | ## RAG |
| OpenAI Chat Model | OpenAI Chat Model | LLM used by agent | — | AI Agent (ai_languageModel) | ## RAG |
| AI Agent | LangChain Agent | Orchestrate tool use + answer generation | When chat message received; Pinecone Vector Store Tool; OpenAI Chat Model | — | ## RAG |
| Sticky Note | Sticky Note | Documentation | — | — | ## Build an awesome RAG system… (contains steps + links) |
| Sticky Note1 | Sticky Note | Section header | — | — | ## Document parsing and ingestion |
| Sticky Note2 | Sticky Note | Section header | — | — | ## RAG |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named `ArynPineconeChatWithData`.

### A) Document ingestion path (manual)
2. Add **Manual Trigger** node named **“When clicking ‘Test workflow’”**.
3. Add **AWS S3** node named **“Get Files from S3”**:
   - **Operation:** *Get All*
   - **Folder key / prefix:** set to your desired “folder” (leave blank for root)
   - Select AWS credentials with permission to list the bucket.
4. Connect: **Manual Trigger → Get Files from S3**.
5. Add **Split In Batches** node named **“Loop Over Items”**:
   - Recommended: **Batch Size = 1** (for predictable sequential downloads).  
   - If you want to replicate exactly: set **Batch Size** expression to `{{ $json.Key.length }}` (not recommended).
6. Connect: **Get Files from S3 → Loop Over Items**.
7. Add **AWS S3** node named **“Download Files from AWS”**:
   - Operation: download/get object (node UI varies)
   - **File Key:** `{{ $json.Key }}`
   - Ensure it outputs the file as **binary** data.
8. Connect: **Loop Over Items → Download Files from AWS** (use the looping output).

### B) Parsing + chunking + insertion to Pinecone
9. Add **Aryn** node named **“Aryn”**:
   - Configure desired DocParse options (text/table/property extraction) as needed.
   - Set **Aryn API key** credentials.
10. Connect: **Download Files from AWS → Aryn**.
11. Add **Recursive Character Text Splitter** node named **“Recursive Character Text Splitter”**:
   - Set **Split code** to `python` (or adjust to your document type).
12. Add **Default Data Loader** node named **“Default Data Loader”**.
13. Connect the splitter to the loader using the **AI Text Splitter** connection type:
   - **Recursive Character Text Splitter (ai_textSplitter) → Default Data Loader**
14. Add **OpenAI Embeddings** node named **“Embeddings OpenAI”**:
   - Set OpenAI credentials.
   - Keep defaults or explicitly choose an embeddings model compatible with your Pinecone index dimension.
15. Add **Pinecone Vector Store** node named **“Pinecone Vector Store”**:
   - **Mode:** `insert`
   - **Index:** create/select an index (example: `n8n`)
   - Set Pinecone credentials.
16. Wire the insert pipeline:
   - **Aryn (main) → Pinecone Vector Store (main)**
   - **Default Data Loader (ai_document) → Pinecone Vector Store (ai_document)**
   - **Embeddings OpenAI (ai_embedding) → Pinecone Vector Store (ai_embedding)**

### C) RAG chat path (runtime)
17. Add **Chat Trigger** node named **“When chat message received”**.
18. Add **AI Agent** node named **“AI Agent”**.
19. Connect: **When chat message received → AI Agent**.
20. Add **OpenAI Chat Model** node named **“OpenAI Chat Model”**:
   - **Model:** `gpt-4o-mini`
   - Set OpenAI credentials.
21. Connect using **AI Language Model** connection:
   - **OpenAI Chat Model (ai_languageModel) → AI Agent**
22. Add **Pinecone Vector Store** node named **“Pinecone Vector Store Tool”**:
   - **Mode:** `retrieve-as-tool`
   - **Index:** same as ingestion (example `n8n`)
   - **topK:** `100` (adjust as needed)
   - Tool description as desired.
23. Connect tool + embeddings:
   - **Embeddings OpenAI (ai_embedding) → Pinecone Vector Store Tool (ai_embedding)**
   - **Pinecone Vector Store Tool (ai_tool) → AI Agent (ai_tool)**

### D) Credentials checklist
24. Configure these credentials in n8n:
   - **AWS**: access key/secret (or role-based), correct region; permissions for `ListBucket` and `GetObject`.
   - **Aryn**: API key from https://aryn.ai/signup
   - **Pinecone**: API key + environment/region; ensure index exists (e.g., `n8n`) and matches embedding dimension.
   - **OpenAI**: API key from https://openai.com

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Processing options reference for Aryn DocParse | https://docs.aryn.ai/docparse/processing_options |
| Get an Aryn API key (paid tiers may be required for vision/OCR/table extraction) | https://aryn.ai/signup |
| Create a Pinecone account and index + API key | https://pinecone.io |
| Obtain an OpenAI API key | https://openai.com |
| Ingestion flow summary: S3 → Aryn parsing → chunk & ingest into Pinecone → then chat over documents | Sticky note “Build an awesome RAG system…” |
| RAG requires at least one document ingested into the Pinecone index before chat results are meaningful | Sticky note “Build an awesome RAG system…” |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.