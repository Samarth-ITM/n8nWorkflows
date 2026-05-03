Chat with Google Drive documents using OpenAI and Pinecone RAG search

https://n8nworkflows.xyz/workflows/chat-with-google-drive-documents-using-openai-and-pinecone-rag-search-11870


# Chat with Google Drive documents using OpenAI and Pinecone RAG search

## 1. Workflow Overview

**Title:** Chat with Google Drive documents using OpenAI and Pinecone RAG search

**Purpose / use case**  
This workflow implements a Retrieval-Augmented Generation (RAG) system over documents stored in **Google Drive**. It has two main capabilities:
1) **Ingestion pipeline**: automatically downloads new files from a specific Drive folder, chunks them, embeds them with OpenAI embeddings, and upserts them into a **Pinecone** index (namespace-scoped).
2) **Chat pipeline**: exposes a chat entry point where an **AI Agent** answers questions by retrieving relevant context from Pinecone (with optional **Cohere reranking**) and responding using an OpenAI chat model, including citations (file name + URL) as instructed.

### 1.1 Document Ingestion (Drive → chunk → embed → Pinecone upsert)
Triggered on file creation in a specific Google Drive folder; downloads the file and inserts its chunked embeddings into Pinecone.

### 1.2 Chat / RAG (Chat → agent → Pinecone retrieval tool → answer)
Triggered by a chat message; agent uses a Pinecone “retrieve-as-tool” integration (plus reranker) to fetch relevant chunks and answer.

---

## 2. Block-by-Block Analysis

### Block 1 — Ingestion: Google Drive → Pinecone
**Overview:** Watches a Drive folder for newly created files, downloads the file contents, then loads/chunks text and upserts embeddings to Pinecone under a specific namespace.

**Nodes involved:**
- Sticky Note (section label)
- Google Drive Trigger
- Download file
- Character Text Splitter - markdown
- Default Data Loader
- Embeddings OpenAI
- Pinecone Vector Store
- Sticky Note2 (chunking strategy explanation)
- Sticky Note7 (global instructions/prereqs)

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Config:** “## 1. Download, chunk, embed, and upsert Pinecone release notes to Pinecone index”
- **Connections:** None
- **Failure modes:** None (non-executing)

#### Node: Google Drive Trigger
- **Type / role:** `googleDriveTrigger` — polling trigger for Drive events.
- **What it does:** Polls every minute; fires when a **file is created** in a **specific folder**.
- **Key configuration:**
  - Event: `fileCreated`
  - Trigger scope: `specificFolder`
  - Folder: `n8n-pinecone-demo` (configured by folder ID)
  - Polling: every minute
- **Credentials:** Google Drive OAuth2 (must have access to the watched folder).
- **Outputs:** Emits file metadata (not the file content). Notably includes `id` and `name` used downstream.
- **Connections:**
  - Output → **Download file**
- **Edge cases / failures:**
  - OAuth token expiry / insufficient scopes
  - Folder ID invalid or moved
  - Polling delays; may miss rapid create/delete cycles depending on Drive behavior

#### Node: Download file
- **Type / role:** `googleDrive` — downloads a Drive file as binary.
- **What it does:** Downloads the created file using the incoming `id`.
- **Key configuration:**
  - Operation: `download`
  - File ID: expression `={{ $json.id }}`
  - File name option: `={{ $json.name }}`
- **Credentials:** Same Google Drive OAuth2.
- **Input:** File metadata from Google Drive Trigger.
- **Output:** Binary data representing the downloaded file + original metadata fields.
- **Connections:**
  - Output → **Pinecone Vector Store** (main input)
- **Edge cases / failures:**
  - File permissions change after creation
  - Unsupported file types for later text extraction (depends on loader)
  - Large file download timeouts

#### Node: Character Text Splitter - markdown
- **Type / role:** LangChain Character Text Splitter — defines chunking strategy.
- **What it does:** Splits text into chunks designed around Pinecone release-note “Update” blocks.
- **Key configuration:**
  - Separator: `<Update label="`
  - Chunk size: `2000`
  - Chunk overlap: `500`
- **Connections:**
  - Output (ai_textSplitter) → **Default Data Loader** (ai_textSplitter input)
- **Edge cases / failures:**
  - If documents don’t contain the separator, chunking may become less semantically aligned (still chunks by size).
  - Overlap + chunk size can increase total vectors/cost.

#### Node: Default Data Loader
- **Type / role:** LangChain default document loader — converts binary file content into “documents” + metadata; applies splitter.
- **What it does:** Reads the binary content (from Drive download) and produces chunked LangChain documents with metadata.
- **Key configuration choices:**
  - Data type: `binary`
  - Binary mode: `specificField` (expects file content in a specific binary property produced by Download)
  - Text splitting: `custom` (uses the connected Character Text Splitter)
  - Metadata: adds `external_file_id` set to `={{ $json.id }}`
- **Inputs / outputs:**
  - **Input:** Binary file + JSON fields (must include `id`)
  - **Output:** LangChain documents (chunks) on `ai_document`
- **Connections:**
  - Output (ai_document) → **Pinecone Vector Store** (ai_document input)
- **Edge cases / failures:**
  - Loader cannot extract text from certain binaries (e.g., encrypted PDFs, unusual encodings)
  - Missing/renamed binary field can break extraction
  - Expression failure if `id` not present in the item

#### Node: Embeddings OpenAI
- **Type / role:** OpenAI embeddings provider for LangChain nodes.
- **What it does:** Generates embeddings for chunks during upsert and for retrieval queries.
- **Key configuration:**
  - Dimensions: `1536` (aligned to OpenAI `text-embedding-3-small` typical dimension)
- **Credentials:** OpenAI API key.
- **Connections:**
  - Output (ai_embedding) → **Pinecone Vector Store** (ai_embedding)
  - Output (ai_embedding) → **Pinecone Vector Store Tool** (ai_embedding)
- **Edge cases / failures:**
  - Model/dimension mismatch with Pinecone index configuration (must match index dimension)
  - Rate limits / quota
  - Large batch sizes causing timeouts (depends on node internals)

#### Node: Pinecone Vector Store (Insert)
- **Type / role:** Pinecone vector store — **upsert** mode.
- **What it does:** Inserts chunk vectors + metadata into Pinecone.
- **Key configuration:**
  - Mode: `insert`
  - Index: `n8n-dense-index`
  - Namespace: `release-notes-namespace`
- **Credentials:** Pinecone API key.
- **Inputs:**
  - Main input from **Download file** (item context)
  - `ai_document` from **Default Data Loader** (chunked docs)
  - `ai_embedding` from **Embeddings OpenAI**
- **Outputs:** Typically confirmation/upsert results (depends on node implementation).
- **Edge cases / failures:**
  - Index does not exist / wrong region/project
  - Namespace mismatch (retrieval must use same namespace to find data)
  - Dimension mismatch (Pinecone index dimension must equal embeddings dimension)
  - Metadata size limits / invalid characters

#### Node: Sticky Note2
- **Type / role:** Sticky Note (documentation for chunking)
- **Content highlights:** Explains why splitting on `<Update label="` is chosen; includes link: https://www.pinecone.io/learn/chunking-strategies/
- **Connections:** None
- **Failure modes:** None

#### Node: Sticky Note7
- **Type / role:** Sticky Note (global documentation)
- **Content highlights:** Prereqs, setup steps, sample prompts, and links:
  - Pinecone Assistant quickstart: https://docs.pinecone.io/guides/assistant/quickstart#n8n
  - Full workflow link: https://n8n.io/workflows/9942-rag-powered-document-chat-with-google-drive-openai-and-pinecone-assistant/
  - Pinecone account/index/API key links; n8n Google OAuth link; OpenAI/Cohere signup links; Discord/issues links.
- **Connections:** None
- **Failure modes:** None

---

### Block 2 — Chat / RAG: Chat Trigger → Agent → Pinecone Retrieval Tool → OpenAI response
**Overview:** Receives a chat message, routes it to an AI Agent configured to use only the Pinecone retrieval tool, optionally reranks retrieved passages via Cohere, and answers using an OpenAI chat model.

**Nodes involved:**
- Sticky Note1 (section label)
- When chat message received
- AI Agent
- OpenAI Chat Model
- Pinecone Vector Store Tool
- Cohere Reranker
- Embeddings OpenAI (shared dependency)

#### Node: Sticky Note1
- **Type / role:** Sticky Note (documentation)
- **Config:** “## 2. Chat with the Pinecone release notes”
- **Connections:** None

#### Node: When chat message received
- **Type / role:** LangChain Chat Trigger — entry point for chat UI/messages.
- **What it does:** Starts the chat flow when a user sends a message to this workflow’s chat endpoint.
- **Key configuration:**
  - Uses an n8n-managed webhook ID (chat trigger)
  - No special options set
- **Output:** Chat message payload to the agent.
- **Connections:**
  - Output → **AI Agent**
- **Edge cases / failures:**
  - If chat is not enabled in the n8n instance or webhook URL not reachable
  - Concurrent chat sessions may require conversation memory configuration (not present here)

#### Node: AI Agent
- **Type / role:** LangChain Agent — orchestrates tool use + final response.
- **What it does:** Uses the connected OpenAI chat model for reasoning and calls the Pinecone retrieval tool to fetch context.
- **Key configuration:**
  - System message:  
    “You are a helpful assistant. Only use the Pinecone Vector Store Tool to retrieve data about Pinecone releases. Include the file name and file url in citations wherever referenced in output.”
- **Inputs:**
  - Main chat message from **When chat message received**
  - `ai_languageModel` from **OpenAI Chat Model**
  - `ai_tool` from **Pinecone Vector Store Tool**
- **Outputs:** Final assistant response back to chat (n8n chat output mechanism).
- **Edge cases / failures:**
  - If retrieved documents do not include `file url` metadata, the agent cannot produce the requested citations (you may need to store URL metadata at ingest time).
  - Tool-only instruction can cause refusal-like behavior if tool returns nothing.
  - Hallucinated citations if metadata isn’t present—mitigate by ensuring metadata fields exist and by tightening system message.

#### Node: OpenAI Chat Model
- **Type / role:** Chat LLM used by the agent.
- **What it does:** Provides the agent’s reasoning and response generation.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
- **Credentials:** OpenAI API key.
- **Connections:**
  - Output (ai_languageModel) → **AI Agent**
- **Edge cases / failures:**
  - Rate limits; model availability; organization restrictions
  - Large context if topK is high and chunks are large (can increase token usage)

#### Node: Pinecone Vector Store Tool (Retrieve-as-tool)
- **Type / role:** Pinecone vector store configured as a LangChain **tool** for retrieval.
- **What it does:** Given a query, retrieves the top matching chunks from Pinecone, using embeddings and optional reranking.
- **Key configuration:**
  - Mode: `retrieve-as-tool`
  - Index: `n8n-dense-index`
  - Namespace: `release-notes-namespace` (must match ingestion)
  - `topK`: `20`
  - `useReranker`: enabled
  - Tool description: “Contains data about Pinecone releases.”
- **Inputs:**
  - `ai_embedding` from **Embeddings OpenAI** (to embed the query)
  - `ai_reranker` from **Cohere Reranker** (because reranking enabled)
- **Connections:**
  - Output (ai_tool) → **AI Agent**
- **Edge cases / failures:**
  - No results if namespace wrong or ingestion not run
  - Reranker credential missing while `useReranker` enabled
  - If stored metadata doesn’t include file name/URL, citations requirement can’t be met reliably

#### Node: Cohere Reranker
- **Type / role:** Reranker model provider (Cohere).
- **What it does:** Reranks retrieved candidates to select the best passages.
- **Key configuration:**
  - `topN`: `5` (final number of passages after reranking)
- **Credentials:** Cohere API key.
- **Connections:**
  - Output (ai_reranker) → **Pinecone Vector Store Tool**
- **Edge cases / failures:**
  - Cohere rate limits / auth issues
  - If `topN` > retrieved candidates, behavior depends on implementation (usually returns all available)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Pinecone Vector Store | Pinecone Vector Store (LangChain) | Upsert chunk embeddings into Pinecone | Download file; Default Data Loader (ai_document); Embeddings OpenAI (ai_embedding) | (none) | ## 1. Download, chunk, embed, and upsert Pinecone release notes to Pinecone index |
| Embeddings OpenAI | OpenAI Embeddings (LangChain) | Create embeddings for documents + queries | (none) | Pinecone Vector Store; Pinecone Vector Store Tool |  |
| When chat message received | Chat Trigger (LangChain) | Entry point for chat messages | (none) | AI Agent | ## 2. Chat with the Pinecone release notes |
| AI Agent | Agent (LangChain) | Orchestrate LLM + tool retrieval and produce final answer | When chat message received; OpenAI Chat Model (ai_languageModel); Pinecone Vector Store Tool (ai_tool) | (chat response) | ## 2. Chat with the Pinecone release notes |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM powering the agent | (none) | AI Agent | ## 2. Chat with the Pinecone release notes |
| Pinecone Vector Store Tool | Pinecone Vector Store (LangChain) | Retrieval tool for RAG over Pinecone | Embeddings OpenAI (ai_embedding); Cohere Reranker (ai_reranker) | AI Agent | ## 2. Chat with the Pinecone release notes |
| Sticky Note7 | Sticky Note | Global prereqs/setup notes | (none) | (none) | ![Pinecone logo](https://www.pinecone.io/images/pinecone-logo-for-n8n-templates.png) / Links: https://docs.pinecone.io/guides/assistant/quickstart#n8n ; https://n8n.io/workflows/9942-rag-powered-document-chat-with-google-drive-openai-and-pinecone-assistant/ ; Discord https://discord.gg/tJ8V62S3sH ; Issues https://github.com/pinecone-io/n8n-templates/issues/new/choose |
| Default Data Loader | Default Data Loader (LangChain) | Convert binary file to documents + attach metadata + apply splitter | Character Text Splitter - markdown (ai_textSplitter); (binary from upstream context) | Pinecone Vector Store (ai_document) | ## 1. Download, chunk, embed, and upsert Pinecone release notes to Pinecone index |
| Cohere Reranker | Cohere Reranker (LangChain) | Rerank retrieved passages | (none) | Pinecone Vector Store Tool (ai_reranker) | ## 2. Chat with the Pinecone release notes |
| Sticky Note | Sticky Note | Section label | (none) | (none) | ## 1. Download, chunk, embed, and upsert Pinecone release notes to Pinecone index |
| Sticky Note1 | Sticky Note | Section label | (none) | (none) | ## 2. Chat with the Pinecone release notes |
| Character Text Splitter - markdown | Character Text Splitter (LangChain) | Chunking strategy for markdown-like release notes | (none) | Default Data Loader (ai_textSplitter) | ## What chunking strategy should I use? https://www.pinecone.io/learn/chunking-strategies/ |
| Sticky Note2 | Sticky Note | Chunking strategy explanation | (none) | (none) | ## What chunking strategy should I use? https://www.pinecone.io/learn/chunking-strategies/ |
| Google Drive Trigger | Google Drive Trigger | Detect new files in a Drive folder | (none) | Download file | ## 1. Download, chunk, embed, and upsert Pinecone release notes to Pinecone index |
| Download file | Google Drive | Download created file as binary | Google Drive Trigger | Pinecone Vector Store | ## 1. Download, chunk, embed, and upsert Pinecone release notes to Pinecone index |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Pinecone API credential (API key from Pinecone console).
   2. OpenAI API credential (API key).
   3. Google Drive OAuth2 credential (enable Drive API in GCP; connect in n8n).
   4. Cohere API credential (API key).

2) **Prepare Pinecone**
   1. Create an index named **`n8n-dense-index`**.
   2. Configure it for an embedding dimension of **1536** (must match the embeddings node).
   3. Decide a namespace; this workflow uses **`release-notes-namespace`**.

3) **Create the ingestion trigger**
   1. Add node: **Google Drive Trigger**
      - Event: *File Created*
      - Trigger on: *Specific folder*
      - Folder: choose your target folder (example: `n8n-pinecone-demo`)
      - Polling: every minute
      - Select Google Drive OAuth2 credential
   2. Add node: **Google Drive** → operation **Download**
      - File ID: expression from trigger item: `{{$json.id}}`
      - (Optional) File name option: `{{$json.name}}`
      - Connect **Google Drive Trigger → Download file**

4) **Add chunking + document loading**
   1. Add node: **Character Text Splitter**
      - Chunk size: `2000`
      - Chunk overlap: `500`
      - Separator: `<Update label="`
   2. Add node: **Default Data Loader**
      - Data type: **Binary**
      - Binary mode: **Specific field** (ensure it points to the binary output of the Download node)
      - Text splitting: **Custom**
      - Add metadata field:
        - `external_file_id` = `{{$json.id}}`
   3. Connect **Character Text Splitter (ai_textSplitter) → Default Data Loader (ai_textSplitter input)**

5) **Add embeddings provider**
   1. Add node: **Embeddings OpenAI**
      - Dimensions: `1536`
      - Select OpenAI credential

6) **Add Pinecone upsert node**
   1. Add node: **Pinecone Vector Store**
      - Mode: **Insert**
      - Index: `n8n-dense-index`
      - Namespace: `release-notes-namespace`
      - Select Pinecone credential
   2. Wire AI inputs:
      - **Default Data Loader (ai_document) → Pinecone Vector Store (ai_document)**
      - **Embeddings OpenAI (ai_embedding) → Pinecone Vector Store (ai_embedding)**
   3. Also connect the “main” execution path:
      - **Download file (main) → Pinecone Vector Store (main)**

7) **Create the chat entry point**
   1. Add node: **When chat message received** (Chat Trigger).
   2. Add node: **AI Agent**
      - System message:  
        “You are a helpful assistant. Only use the Pinecone Vector Store Tool to retrieve data about Pinecone releases. Include the file name and file url in citations wherever referenced in output.”
   3. Connect **When chat message received → AI Agent**

8) **Add the chat LLM**
   1. Add node: **OpenAI Chat Model**
      - Model: `gpt-4.1-mini`
      - Select OpenAI credential
   2. Connect **OpenAI Chat Model (ai_languageModel) → AI Agent (ai_languageModel)**

9) **Add Pinecone retrieval tool (+ reranking)**
   1. Add node: **Pinecone Vector Store** configured as a tool
      - Mode: **Retrieve as tool**
      - Index: `n8n-dense-index`
      - Namespace: `release-notes-namespace`
      - topK: `20`
      - Enable reranker: **true**
      - Tool description: “Contains data about Pinecone releases.”
      - Select Pinecone credential
   2. Add node: **Cohere Reranker**
      - topN: `5`
      - Select Cohere credential
   3. Connect:
      - **Embeddings OpenAI (ai_embedding) → Pinecone Vector Store Tool (ai_embedding)**
      - **Cohere Reranker (ai_reranker) → Pinecone Vector Store Tool (ai_reranker)**
      - **Pinecone Vector Store Tool (ai_tool) → AI Agent (ai_tool)**

10) **Run**
   - Activate workflow (so Drive polling works).
   - Add release-note files into the watched Drive folder to trigger ingestion.
   - Open chat and ask questions (example prompts in Sticky Note7).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Pinecone Assistant quickstart for n8n | https://docs.pinecone.io/guides/assistant/quickstart#n8n |
| Full workflow variant (Drive + OpenAI + Pinecone Assistant) | https://n8n.io/workflows/9942-rag-powered-document-chat-with-google-drive-openai-and-pinecone-assistant/ |
| Pinecone chunking strategies article | https://www.pinecone.io/learn/chunking-strategies/ |
| Pinecone Discord community | https://discord.gg/tJ8V62S3sH |
| Issues tracker for templates | https://github.com/pinecone-io/n8n-templates/issues/new/choose |
| Suggested source documents (release notes 2022–2026) | https://docs.pinecone.io/release-notes/2022.md (and 2023–2026 equivalents) |
| Index setup expectation | Index name `n8n-dense-index`, embedding model aligned to OpenAI `text-embedding-3-small`, dimension `1536` (keep nodes/index consistent) |