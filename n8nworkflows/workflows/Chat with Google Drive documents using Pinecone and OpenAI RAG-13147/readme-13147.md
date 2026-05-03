Chat with Google Drive documents using Pinecone and OpenAI RAG

https://n8nworkflows.xyz/workflows/chat-with-google-drive-documents-using-pinecone-and-openai-rag-13147


# Chat with Google Drive documents using Pinecone and OpenAI RAG

## 1. Workflow Overview

**Title:** Chat with Google Drive documents using Pinecone and OpenAI RAG  
**Workflow name (internal):** Retrieve Google Drive documents using Pinecone and OpenAI.  
**Purpose:** Maintain a continuously updated vector knowledge base from a specific Google Drive folder (PDF/TXT) in **Pinecone**, and expose a **public chat** endpoint that answers questions using **RAG** (retrieval + OpenAI chat generation) grounded in those documents.

### 1.1 Logical Blocks

1. **1.1 Full Folder Re-Embedding (Manual Rebuild)**
   - Clears the Pinecone index (deleteAll) then re-downloads and re-embeds up to 500 files from the target folder in batches.

2. **1.2 Incremental Ingestion (Drive Created/Updated)**
   - Polls for created/updated files in a Drive folder, filters PDFs, downloads them, and inserts embeddings into Pinecone.

3. **1.3 Chat RAG (Public Chat UI + Agent)**
   - Provides a public chat interface with custom CSS. The agent uses Pinecone retrieval as a tool plus a short memory buffer, then answers via OpenAI.

> Important implementation note: The ingestion nodes include **Text Splitter** and **Default Data Loader** nodes connected to Pinecone via *AI ports*, but the **Google Drive “Download” nodes are connected directly to Pinecone on the main flow**, which can lead to ambiguous/incorrect ingestion unless the Pinecone node is actually configured to read the binary content from upstream items by default. This is an integration edge case to validate (details in Block 2).

---

## 2. Block-by-Block Analysis

### Block 1.1 — Full Folder Re-Embedding (Manual Rebuild)

**Overview:** Rebuilds the entire Pinecone index from the Google Drive folder. It first deletes all vectors in the index, then searches and loops through files, downloading each and sending it for vector insertion. Also sends an email notification per batch iteration.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Pinecone – Delete All Vectors
- Search files and folders
- Loop Items
- Download File From Google Drive
- Pinecone Vector Store
- Wait
- Send a message
- (Associated AI ingestion chain nodes used by Pinecone Vector Store: Recursive Character Text Splitter, Default Data Loader, Embeddings OpenAI1)

#### Node: **When clicking ‘Execute workflow’** (Manual Trigger)
- **Type/role:** `n8n-nodes-base.manualTrigger` — manual entry point to rebuild the index.
- **Config:** No parameters.
- **Outputs:** To **Pinecone – Delete All Vectors**.
- **Edge cases:** None (manual only).

#### Node: **Pinecone – Delete All Vectors** (HTTP Request)
- **Type/role:** `n8n-nodes-base.httpRequest` — calls Pinecone REST API to wipe the index.
- **Config choices:**
  - `POST https://<index-host>/vectors/delete`
  - JSON body: `{ "deleteAll": true }`
  - Headers include `Api-Key` and `Content-Type: application/json`
  - **onError:** “continueRegularOutput” (workflow continues even if delete fails)
- **Outputs:** To **Search files and folders**
- **Failure modes / edge cases:**
  - Wrong Pinecone host URL (region/project mismatch) → 404/401.
  - Invalid API key → 401.
  - `deleteAll` is destructive; if it fails but workflow continues, the rebuild will **append/duplicate** vectors rather than replace them.

#### Node: **Search files and folders** (Google Drive)
- **Type/role:** `n8n-nodes-base.googleDrive` — lists items to ingest.
- **Config choices:**
  - Resource: file/folder search (query method)
  - Query:
    - Folder constraint: `'<folderId>' in parents`
    - Types: PDF (`application/pdf`) or TXT (`text/plain`)
    - `trashed = false`
  - Limit: 500
- **Inputs:** From **Pinecone – Delete All Vectors**
- **Outputs:** To **Loop Items**
- **Failure modes / edge cases:**
  - Folder permissions missing → 403.
  - Query returns Google Docs formats excluded by mimeType; only PDF/TXT will be ingested.
  - If >500 files exist, remaining files won’t be indexed unless pagination is added.

#### Node: **Loop Items** (Split In Batches)
- **Type/role:** `n8n-nodes-base.splitInBatches` — processes the search results in batches of 30.
- **Config choices:**
  - `batchSize: 30`
  - `reset: false` (keeps internal cursor during a single run)
- **Inputs:** From **Search files and folders**, and also loops back from **Wait**
- **Outputs:**
  - Output 1 → **Send a message**
  - Output 2 → **Download File From Google Drive**
- **Edge cases:**
  - Current wiring sends an email for *every batch cycle* (and possibly every loop step depending on n8n execution). If not desired, move Gmail to the end of processing.
  - If any downstream node “continues on error”, batches may advance while missing documents.

#### Node: **Send a message** (Gmail)
- **Type/role:** `n8n-nodes-base.gmail` — sends notification email.
- **Config choices:**
  - To: `user@example.com`
  - Subject: `Flow terminated {{ $workflow.name }}`
  - Message: `Il flusso è terminato con successo.`
- **Inputs:** From **Loop Items** (first output)
- **Edge cases:**
  - OAuth token revoked → auth error.
  - Triggers too many emails during long rebuilds (rate limits / spam).

#### Node: **Download File From Google Drive** (Google Drive)
- **Type/role:** `n8n-nodes-base.googleDrive` — downloads each file as binary.
- **Config choices:**
  - Operation: download
  - File ID: `={{ $json.id }}`
  - File name option: `={{ $json.name }}`
  - **onError:** continueErrorOutput
  - **alwaysOutputData:** true (outputs item even on error)
- **Inputs:** From **Loop Items** (second output)
- **Outputs:** To **Pinecone Vector Store**
- **Failure modes / edge cases:**
  - Large PDFs may exceed memory/timeouts.
  - Download errors still produce items; downstream Pinecone insertion may fail unless guarded.

#### Node: **Pinecone Vector Store** (Insert)
- **Type/role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone` — inserts document chunks + embeddings into Pinecone index.
- **Config choices:**
  - Mode: **insert**
  - Index: `open-ai-3072`
  - Credentials: Pinecone API credential
- **Inputs:**
  - Main input: from **Download File From Google Drive**
  - AI input ports:
    - `ai_embedding` from **Embeddings OpenAI1**
    - `ai_document` from **Default Data Loader**
- **Outputs:** Main output → **Wait**
- **Version requirements:** This is the n8n LangChain node set; requires n8n version that supports these nodes (typeVersion 1 in workflow).
- **Critical integration edge case:**
  - The **Default Data Loader** is configured to load from **binary** (“specificField”) but **its binary field name is not specified in the node parameters shown**. If it expects a specific binary property (commonly `data`), and Drive downloads store binary under a different key (often `data` but depends on node settings), ingestion can silently fail or produce empty documents.
  - The **Text Splitter** node exists but its connectivity is inverted: `Recursive Character Text Splitter -> Default Data Loader` (see below). Typically, you want **Loader → Splitter → VectorStore**. As wired, the loader receives `ai_textSplitter` from the splitter, which is unusual and may still work if the loader accepts a splitter as an option, but you must confirm actual runtime behavior in your n8n/LangChain version.

#### Node: **Wait**
- **Type/role:** `n8n-nodes-base.wait` — used as a pacing/loop control step.
- **Config:** no explicit wait time configured (defaults depend on node usage); has a webhookId assigned (used internally by n8n wait/resume).
- **Inputs:** From **Pinecone Vector Store**
- **Outputs:** Back to **Loop Items** to continue next batch.
- **Edge cases:**
  - If configured to “wait for webhook” by default, executions may pause indefinitely unless resumed. Validate the Wait node mode in UI.

#### Node: **Recursive Character Text Splitter** (AI Text Splitter)
- **Type/role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`
- **Config choices:**
  - `chunkOverlap: 100`
  - (Chunk size not shown; node defaults apply.)
- **Connections:** `ai_textSplitter` → **Default Data Loader**
- **Edge cases:** If chunk size default is too large/small, retrieval quality suffers; PDFs with long pages may need smaller chunks.

#### Node: **Default Data Loader**
- **Type/role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`
- **Config choices:**
  - Data type: **binary**
  - Binary mode: **specificField**
  - **onError:** continueErrorOutput
- **Connections:** `ai_document` → **Pinecone Vector Store**
- **Edge cases:** missing/incorrect binary field name; unsupported PDF parsing depending on n8n build.

#### Node: **Embeddings OpenAI1**
- **Type/role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — embedding model for ingestion.
- **Config:** `text-embedding-3-large`
- **Connections:** `ai_embedding` → **Pinecone Vector Store**
- **Edge cases:** model availability, rate limits, large batch throughput.

**Sticky note context (applies to this block):**
- “## Embedding entire folder — Upload all documents from a folder”
- “## RAG with Google Drive, Pinecone, and OpenAI …” (general description; see table)

---

### Block 1.2 — Incremental Ingestion (Drive Created/Updated)

**Overview:** Watches a Google Drive folder for newly created or updated files (polling every minute), filters to PDFs, downloads the file, then inserts it into the same Pinecone index using an embeddings model.

**Nodes involved:**
- Google Drive File Created
- Filter PDF files
- If length > 0
- Download File From Google Drive1
- Pinecone Vector Store1
- (AI ingestion chain nodes: Recursive Character Text Splitter1, Default Data Loader1, Embeddings OpenAI2)
- Google Drive File Updated
- Filter PDF files1
- If length > 0_

#### Node: **Google Drive File Created** (Google Drive Trigger)
- **Type/role:** `n8n-nodes-base.googleDriveTrigger` — polling trigger.
- **Config choices:**
  - Event: `fileCreated`
  - Trigger on: specific folder (`1-4lBhk3UPcaMSb1_7uE7-jsgRMqcP6M8`)
  - Poll: every minute
  - File type option set to “all” (filtering done downstream)
- **Outputs:** To **Filter PDF files**
- **Edge cases:** Polling can miss rapid create/delete, duplicates possible if Drive changes; ensure idempotency in Pinecone metadata.

#### Node: **Filter PDF files** (Filter)
- **Type/role:** `n8n-nodes-base.filter` — keeps only PDFs.
- **Condition:** `$json.mimeType` contains `application/pdf`
- **Outputs:** To **If length > 0**
- **Edge cases:** “contains” is lenient; fine here, but mimeType should usually be exact equals.

#### Node: **If length > 0** (IF)
- **Type/role:** `n8n-nodes-base.if` — checks that there is at least one item.
- **Condition:** `={{ $items().length > 0 }}`
- **Outputs (true):** to **Download File From Google Drive1**
- **Edge cases:** This pattern is often redundant because filters output 0 items naturally; but it can help avoid downstream node execution with empty input.

#### Node: **Google Drive File Updated** (Google Drive Trigger)
- **Type/role:** Drive polling trigger for modifications.
- **Config:** Event `fileUpdated`, same folder, every minute.
- **Outputs:** To **Filter PDF files1**
- **Edge cases:** Updates can be frequent; without deduplication you may re-embed same content repeatedly.

#### Node: **Filter PDF files1** (Filter)
- Same logic as “Filter PDF files”
- Output → **If length > 0_**

#### Node: **If length > 0_** (IF)
- Same as above
- Output (true) → **Download File From Google Drive1**

#### Node: **Download File From Google Drive1**
- **Type/role:** Google Drive download (binary)
- **Config:** fileId `={{ $json.id }}`, fileName `={{ $json.name }}`
- **onError:** continueErrorOutput; **alwaysOutputData:** true
- **Output:** to **Pinecone Vector Store1**
- **Edge cases:** same as other download node.

#### Node: **Pinecone Vector Store1** (Insert)
- **Type/role:** Pinecone insert (incremental)
- **Index:** `open-ai-3072`
- **AI connections:**
  - Embeddings from **Embeddings OpenAI2**
  - Documents from **Default Data Loader1**
- **Main input:** from **Download File From Google Drive1**
- **Edge cases:** same ingestion wiring concerns as in Block 1.1 (binary field, loader/splitter chain correctness).

#### Node: **Recursive Character Text Splitter1**
- chunkOverlap 100
- Connected `ai_textSplitter` → **Default Data Loader1**
- Same potential inversion concern.

#### Node: **Default Data Loader1**
- Binary, specificField, continue on error
- Connected `ai_document` → **Pinecone Vector Store1**

#### Node: **Embeddings OpenAI2**
- `text-embedding-3-large`
- Connected `ai_embedding` → **Pinecone Vector Store1**

**Sticky note context (applies to this block):**
- “## Add documents to vector store when updating or creating new documents in Google Drive”
- “## Embedding documents information into Pinecone vector database”
- “## RAG with Google Drive, Pinecone, and OpenAI …” (general description)

---

### Block 1.3 — Chat RAG (Public Chat UI + Agent)

**Overview:** Exposes a public chat endpoint (with custom UI styling). An AI agent uses a vector-store retrieval tool backed by Pinecone + OpenAI embeddings, keeps short conversation memory, and generates answers via OpenAI chat model.

**Nodes involved:**
- When chat message received
- Documents finder (Agent)
- Window Buffer Memory
- OpenAI Chat Model
- Vector Store Tool
- Pinecone Vector Store (Retrieval)
- Embeddings OpenAI
- OpenAI Chat Model1

#### Node: **When chat message received** (Chat Trigger)
- **Type/role:** `@n8n/n8n-nodes-langchain.chatTrigger` — public chat entry point.
- **Config choices:**
  - **public: true**
  - UI: title “Finding documents…”, placeholder “Tell me what you need.”
  - Allows file uploads: true
  - Allowed MIME types: `application/pdf`
  - Response mode: “lastNode”
  - Extensive `customCss` for theming and RAG formatting
  - Initial message: “Hi! How cal I help you?”
- **Outputs:** To **Documents finder**
- **Edge cases:**
  - Public endpoint must be protected if sensitive docs are indexed.
  - File upload enabled but the workflow does not show downstream ingestion of uploaded PDFs into Pinecone; uploads may be unused unless the agent is configured to read them.

#### Node: **Documents finder** (AI Agent)
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool use and final response.
- **Config choices:**
  - System message: instructs assistant to retrieve information from RAG archive and include file names where possible.
  - `returnIntermediateSteps: false`
- **Inputs via AI ports:**
  - Language model from **OpenAI Chat Model**
  - Memory from **Window Buffer Memory**
  - Tool from **Vector Store Tool**
- **Outputs:** (No downstream nodes; chat trigger uses “lastNode” response mode)
- **Edge cases:** If tool retrieval fails, the agent may answer generically; consider enforcing “answer only from retrieved context” in system message.

#### Node: **Window Buffer Memory**
- **Type/role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — retains recent chat turns.
- **Config:** defaults (window size not shown; node defaults apply)
- **Connection:** `ai_memory` → **Documents finder**
- **Edge cases:** Too large window can cause context bloat; too small can lose conversational continuity.

#### Node: **OpenAI Chat Model**
- **Type/role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — main LLM for the agent.
- **Model:** `gpt-5.2`
- **Connection:** `ai_languageModel` → **Documents finder**
- **Edge cases:** model name availability depends on account/region; handle rate limits and token limits.

#### Node: **Vector Store Tool**
- **Type/role:** `@n8n/n8n-nodes-langchain.toolVectorStore` — exposes vector search as an agent tool.
- **Config choices:**
  - Tool name: `company_documents_tool`
  - Description: “Retrieve information from any company documents”
  - `topK: 10`
- **Connections:**
  - `ai_tool` → **Documents finder**
  - `ai_languageModel` input comes from **OpenAI Chat Model1** (tool may use LLM internally for query rewriting depending on implementation)
  - `ai_vectorStore` input comes from **Pinecone Vector Store (Retrieval)**
- **Edge cases:** If vectorStore connection missing/misconfigured, tool calls fail and agent won’t retrieve.

#### Node: **Pinecone Vector Store (Retrieval)**
- **Type/role:** Pinecone vector store configured for retrieval use.
- **Index:** `open-ai-3072`
- **Connections:** `ai_vectorStore` → **Vector Store Tool**
- **Embedding connection:** receives `ai_embedding` from **Embeddings OpenAI**
- **Edge cases:** Must match same embedding dimensionality used at insert time; index `open-ai-3072` suggests 3072-dim embeddings compatible with `text-embedding-3-large`.

#### Node: **Embeddings OpenAI**
- **Type/role:** embeddings for query-time retrieval
- **Model:** `text-embedding-3-large`
- **Connection:** `ai_embedding` → **Pinecone Vector Store (Retrieval)**
- **Edge cases:** If ingestion used different embedding model, retrieval quality breaks or Pinecone rejects dimension mismatch.

#### Node: **OpenAI Chat Model1**
- **Type/role:** LLM wired into the Vector Store Tool node.
- **Model:** `gpt-5.2`
- **Connection:** `ai_languageModel` → **Vector Store Tool**
- **Edge cases:** Often optional; if unused by your tool version, it can be removed. If used (e.g., query rewriting), ensure prompts don’t leak data.

**Sticky note context (applies to this block):**
- “## Chat with company documents — Available chat tool with customized css style”
- “## RAG with Google Drive, Pinecone, and OpenAI …” (general description)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entrypoint for full re-index | — | Pinecone – Delete All Vectors | ## Embedding entire folder / Upload all documents from a folder |
| Pinecone – Delete All Vectors | HTTP Request | Destructively clears Pinecone index | When clicking ‘Execute workflow’ | Search files and folders | ## Embedding entire folder / Upload all documents from a folder |
| Search files and folders | Google Drive | Lists PDF/TXT files in folder | Pinecone – Delete All Vectors | Loop Items | ## Embedding entire folder / Upload all documents from a folder |
| Loop Items | Split In Batches | Batch iteration controller | Search files and folders; Wait | Send a message; Download File From Google Drive | ## Embedding entire folder / Upload all documents from a folder |
| Send a message | Gmail | Email notification | Loop Items | — | ## Embedding entire folder / Upload all documents from a folder |
| Download File From Google Drive | Google Drive | Downloads file binaries (manual rebuild path) | Loop Items | Pinecone Vector Store | ## Embedding entire folder / Upload all documents from a folder |
| Pinecone Vector Store | Pinecone Vector Store (LangChain) | Inserts downloaded documents into Pinecone | Download File From Google Drive | Wait | ## Embedding documents information into Pinecone vector database |
| Wait | Wait | Pacing/loop continuation | Pinecone Vector Store | Loop Items | ## Embedding entire folder / Upload all documents from a folder |
| Recursive Character Text Splitter | LangChain Text Splitter | Chunking configuration for ingestion | — | Default Data Loader (AI port) | ## Embedding documents information into Pinecone vector database |
| Default Data Loader | LangChain Document Loader | Parses binary into documents for ingestion | — | Pinecone Vector Store (AI port) | ## Embedding documents information into Pinecone vector database |
| Embeddings OpenAI1 | OpenAI Embeddings (LangChain) | Embeddings for ingestion (manual path) | — | Pinecone Vector Store (AI port) | ## Embedding documents information into Pinecone vector database |
| Google Drive File Created | Google Drive Trigger | Polls for new files in folder | — | Filter PDF files | ## Add documents to vector store when updating or creating new documents in Google Drive |
| Filter PDF files | Filter | Keep only PDFs (created path) | Google Drive File Created | If length > 0 | ## Add documents to vector store when updating or creating new documents in Google Drive |
| If length > 0 | IF | Guard: only proceed when items exist | Filter PDF files | Download File From Google Drive1 | ## Add documents to vector store when updating or creating new documents in Google Drive |
| Google Drive File Updated | Google Drive Trigger | Polls for updated files in folder | — | Filter PDF files1 | ## Add documents to vector store when updating or creating new documents in Google Drive |
| Filter PDF files1 | Filter | Keep only PDFs (updated path) | Google Drive File Updated | If length > 0_ | ## Add documents to vector store when updating or creating new documents in Google Drive |
| If length > 0_ | IF | Guard: only proceed when items exist | Filter PDF files1 | Download File From Google Drive1 | ## Add documents to vector store when updating or creating new documents in Google Drive |
| Download File From Google Drive1 | Google Drive | Downloads file binaries (incremental path) | If length > 0; If length > 0_ | Pinecone Vector Store1 | ## Embedding documents information into Pinecone vector database |
| Pinecone Vector Store1 | Pinecone Vector Store (LangChain) | Inserts incremental updates into Pinecone | Download File From Google Drive1 | — | ## Embedding documents information into Pinecone vector database |
| Recursive Character Text Splitter1 | LangChain Text Splitter | Chunking configuration for incremental ingestion | — | Default Data Loader1 (AI port) | ## Embedding documents information into Pinecone vector database |
| Default Data Loader1 | LangChain Document Loader | Parses binary into documents for incremental ingestion | — | Pinecone Vector Store1 (AI port) | ## Embedding documents information into Pinecone vector database |
| Embeddings OpenAI2 | OpenAI Embeddings (LangChain) | Embeddings for incremental ingestion | — | Pinecone Vector Store1 (AI port) | ## Embedding documents information into Pinecone vector database |
| When chat message received | LangChain Chat Trigger | Public chat UI entrypoint | — | Documents finder | ## Chat with company documents / Available chat tool with customized css style |
| Documents finder | LangChain Agent | Runs RAG agent with tool + memory | When chat message received | — | ## Chat with company documents / Available chat tool with customized css style |
| Window Buffer Memory | LangChain Memory | Short-term conversation memory | — | Documents finder (AI port) | ## Chat with company documents / Available chat tool with customized css style |
| OpenAI Chat Model | OpenAI Chat (LangChain) | Primary LLM for agent | — | Documents finder (AI port) | ## Chat with company documents / Available chat tool with customized css style |
| Vector Store Tool | LangChain VectorStore Tool | Tool wrapper for retrieval | — | Documents finder (AI port) | ## Chat with company documents / Available chat tool with customized css style |
| Pinecone Vector Store (Retrieval) | Pinecone Vector Store (LangChain) | Vector retrieval backend for tool | — | Vector Store Tool (AI port) | ## Chat with company documents / Available chat tool with customized css style |
| Embeddings OpenAI | OpenAI Embeddings (LangChain) | Query embeddings for retrieval | — | Pinecone Vector Store (Retrieval) (AI port) | ## Chat with company documents / Available chat tool with customized css style |
| OpenAI Chat Model1 | OpenAI Chat (LangChain) | LLM dependency for the tool (if used) | — | Vector Store Tool (AI port) | ## Chat with company documents / Available chat tool with customized css style |
| Sticky Note | Sticky Note | Comment block | — | — | ## Add documents to vector store when updating or creating new documents in Google Drive |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## RAG with Google Drive, Pinecone, and OpenAI (full description text) |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Chat with company documents / Available chat tool with customized css style |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Embedding entire folder / Upload all documents from a folder |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Embedding documents information into Pinecone vector database |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Google Drive OAuth2 credential (access to the target folder).
   2. OpenAI API credential (for embeddings + chat).
   3. Pinecone API credential (for vector store nodes).
   4. Gmail OAuth2 credential (optional, for notifications).

2. **Set up the Pinecone index**
   - Create an index compatible with `text-embedding-3-large` dimensionality (commonly **3072**).
   - Name it (as in workflow) `open-ai-3072`.
   - Note the **index host URL** (used by the HTTP delete call).

3. **Block: Full Folder Re-Embedding**
   1. Add **Manual Trigger** node: “When clicking ‘Execute workflow’”.
   2. Add **HTTP Request** node: “Pinecone – Delete All Vectors”
      - Method: POST
      - URL: `https://<your-index-host>/vectors/delete`
      - JSON body: `{ "deleteAll": true }`
      - Headers: `Api-Key: <pinecone-key>`, `Content-Type: application/json`
      - Set **Continue On Fail** (equivalent to “continueRegularOutput”) if you want it non-blocking.
   3. Add **Google Drive** node: “Search files and folders”
      - Operation: Search (query)
      - Query: `'<folderId>' in parents and (mimeType = 'application/pdf' or mimeType = 'text/plain') and trashed = false`
      - Limit: 500
   4. Add **Split In Batches**: “Loop Items”
      - Batch size: 30
   5. Add **Google Drive** node: “Download File From Google Drive”
      - Operation: Download
      - File ID: expression `{{$json.id}}`
      - Options: file name `{{$json.name}}`
      - Enable **Continue On Fail** and **Always Output Data** if you want the same behavior.
   6. Add **Pinecone Vector Store (LangChain)** node: “Pinecone Vector Store”
      - Mode: Insert
      - Index: `open-ai-3072`
   7. Add **Wait** node
      - Configure appropriately (time-based wait is safest) so it does not pause forever.
   8. Wire main connections:
      - Manual Trigger → Delete All → Search → Loop Items
      - Loop Items (batch output) → Download → Pinecone Vector Store → Wait → back to Loop Items
   9. (Optional) Add **Gmail** node “Send a message” and connect from Loop Items if you truly want per-batch notifications; otherwise connect after the loop is complete.

4. **Configure ingestion AI dependencies for Pinecone insertion (recommended canonical wiring)**
   - Create:
     1. **Default Data Loader** (binary; pick the correct binary property name used by Drive download, commonly `data`)
     2. **Recursive Character Text Splitter** (set chunk size and overlap; overlap 100)
     3. **OpenAI Embeddings** (text-embedding-3-large)
   - Recommended AI-port wiring:
     - Default Data Loader → (documents) → Text Splitter → (split docs) → Pinecone Vector Store
     - Embeddings → Pinecone Vector Store
   - If your n8n version expects the splitter to be provided *to the loader* (as in the JSON), replicate that, but validate by running a single document and checking Pinecone vector counts.

5. **Block: Incremental ingestion (Created/Updated)**
   1. Add **Google Drive Trigger**: “Google Drive File Created”
      - Event: fileCreated
      - Watch: specific folder (same folderId)
      - Poll every minute
   2. Add **Filter**: “Filter PDF files” where `$json.mimeType` contains `application/pdf`
   3. Add **IF**: “If length > 0” with condition `{{$items().length > 0}}`
   4. Reuse (or create) **Google Drive Download** node: “Download File From Google Drive1”
   5. Add **Pinecone Vector Store** node: “Pinecone Vector Store1” (Insert, same index)
   6. Repeat similarly for **Google Drive Trigger** “File Updated” → Filter → IF → same Download → Pinecone Vector Store1.
   7. Attach embeddings + loader + splitter AI ports for Pinecone Vector Store1.

6. **Block: Chat RAG**
   1. Add **Chat Trigger (LangChain)**: “When chat message received”
      - Set to public if desired
      - Set response mode: “lastNode”
      - Configure UI texts and custom CSS (paste as needed)
      - Enable file uploads if you plan to handle them
   2. Add **AI Agent**: “Documents finder”
      - System message: instruct grounded retrieval and include file names
   3. Add **Window Buffer Memory** and connect to agent `ai_memory`.
   4. Add **OpenAI Chat Model** (gpt-5.2) and connect to agent `ai_languageModel`.
   5. Add **Pinecone Vector Store (Retrieval)** node pointing to the same index.
   6. Add **OpenAI Embeddings** for retrieval (text-embedding-3-large) and connect to Pinecone retrieval `ai_embedding`.
   7. Add **Vector Store Tool**:
      - Name: `company_documents_tool`
      - topK: 10
      - Connect:
        - Pinecone retrieval node → tool `ai_vectorStore`
        - Tool → agent `ai_tool`
      - If required by your version, add a second **OpenAI Chat Model** and connect to tool `ai_languageModel`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## RAG with Google Drive, Pinecone, and OpenAI … (full explanation of ingestion + deletion + chat)” | Sticky note text embedded in the workflow canvas |
| The workflow describes handling deletions (“When a file is deleted… removes vectors”), but **no Google Drive “fileDeleted” trigger or Pinecone per-file delete implementation exists** in the provided nodes. | Gap between description and implementation; add a delete trigger + delete-by-metadata/vector-id to prevent stale vectors. |
| The Pinecone delete operation uses `deleteAll: true` and continues on error. | High risk of accidental data loss or duplication depending on failure mode. |
| Chat trigger is **public** and uses a custom CSS UI. | Ensure access control if documents are sensitive. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.