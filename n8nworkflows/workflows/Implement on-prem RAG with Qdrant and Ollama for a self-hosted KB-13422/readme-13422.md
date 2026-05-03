Implement on-prem RAG with Qdrant and Ollama for a self-hosted KB

https://n8nworkflows.xyz/workflows/implement-on-prem-rag-with-qdrant-and-ollama-for-a-self-hosted-kb-13422


# Implement on-prem RAG with Qdrant and Ollama for a self-hosted KB

## 1. Workflow Overview

**Purpose:**  
This workflow implements a fully self-hosted Retrieval-Augmented Generation (RAG) setup using **Ollama** (LLM + embeddings) and **Qdrant** (vector database) to build and query a local knowledge base (KB) in n8n.

**Target use cases:**
- Internal documentation Q&A without sending data to external LLM providers
- On-prem/private KB for PDFs/docs uploaded by users
- Lightweight “chat with your documents” using a vector store + local model

### 1.1 Knowledge Base Ingestion (Upload → Chunk → Embed → Store)
Users upload a document via an n8n Form. The workflow loads and splits it into pages, generates embeddings using Ollama, and stores vectors in a Qdrant collection (`knowledge-base`).

### 1.2 Knowledge Base Query (Chat → Retrieve → Answer with Context)
A chat trigger receives user messages. An AI Agent is forced (via system instructions) to use the Qdrant retriever tool first, then answer using the retrieved context with an Ollama chat model, while maintaining short conversational memory.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / Operator Guidance (Sticky Notes)
**Overview:** Provides operator instructions for setup and usage, including required local services and models.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration (interpreted):** Large instruction panel describing how the template works, how to run it, and setup steps for Ollama + Qdrant.
- **Connections:** None (documentation-only).
- **Edge cases:** None.

#### Node: Sticky Note1
- **Type / role:** Sticky Note (section header)
- **Configuration:** Marks the “1. Update Knowledge base” area.
- **Connections:** None.
- **Edge cases:** None.

#### Node: Sticky Note2
- **Type / role:** Sticky Note (section header)
- **Configuration:** Marks the “2. Query Knowledge base” area.
- **Connections:** None.
- **Edge cases:** None.

---

### Block B — Knowledge Base Ingestion (Form Upload → Qdrant Insert)
**Overview:** Accepts a file upload, converts it into LangChain documents (split by pages), embeds content via Ollama embeddings, and inserts vectors into Qdrant collection `knowledge-base`.

**Nodes involved:**
- Upload document
- Default Data Loader
- Embeddings Ollama
- Add to Qdrant Vector Store

#### Node: Upload document
- **Type / role:** `n8n-nodes-base.formTrigger` (entry point; collects user input + file upload)
- **Configuration choices:**
  - Form title: **Upload**
  - Description: **“Poor mans Knowledge base”**
  - Fields:
    - “Doc Name” (text)
    - “File” (file upload)
  - Creates a public webhook endpoint (form)
- **Key data behavior:**
  - Produces a workflow execution containing the uploaded file in **binary data** (given workflow setting `binaryMode: separate`).
- **Connections:**
  - **Output (main)** → Add to Qdrant Vector Store (main)
- **Edge cases / failures:**
  - Large files may exceed n8n upload limits or reverse proxy limits.
  - Missing file field or unsupported mime types can break downstream document loading.
  - If running behind a proxy, webhook/form URL configuration must be correct.

#### Node: Default Data Loader
- **Type / role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` (document parsing + splitting)
- **Configuration choices:**
  - **Data type:** Binary (expects file in binary)
  - Option **Split pages = true** (splits document into per-page documents when supported)
- **Connections:**
  - **Output (ai_document)** → Add to Qdrant Vector Store (ai_document)
- **Important note on wiring:**
  - This node is not connected via a “main” line from the Form Trigger; it feeds the vector store via the special **ai_document** connection. Practically, ingestion works when the vector store node is able to receive document input from this loader during the same execution.
- **Edge cases / failures:**
  - Unsupported file types may not parse.
  - Corrupt PDFs / encrypted documents can fail to load.
  - Very large documents can cause long processing times or memory pressure.

#### Node: Embeddings Ollama
- **Type / role:** `@n8n/n8n-nodes-langchain.embeddingsOllama` (embedding generator)
- **Configuration choices:**
  - Model: `nomic-embed-text:latest`
  - Uses Ollama API credentials (“Ollama account”), pointing to your local Ollama server (commonly `http://localhost:11434`).
- **Connections:**
  - **Output (ai_embedding)** → Add to Qdrant Vector Store (ai_embedding)
  - **Output (ai_embedding)** → Read from Qdrant Vector Store (ai_embedding)
- **Edge cases / failures:**
  - Model not pulled (`ollama pull nomic-embed-text:latest`) → embedding calls fail.
  - Ollama service not reachable / wrong base URL → connection errors/timeouts.
  - Embedding dimensionality must match Qdrant collection vector size (here expected **768**).

#### Node: Add to Qdrant Vector Store
- **Type / role:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant` (vector store writer)
- **Configuration choices:**
  - Mode: **insert**
  - Collection: `knowledge-base`
  - Credentials: Qdrant API (“QdrantApi account”)
- **Inputs:**
  - **main:** comes from Upload document (execution trigger)
  - **ai_document:** comes from Default Data Loader (parsed/split documents)
  - **ai_embedding:** comes from Embeddings Ollama (vectors)
- **Outputs:** None connected downstream.
- **Edge cases / failures:**
  - Qdrant not running or wrong URL/port → connection refused.
  - Collection does not exist (`knowledge-base`) → insert fails (depending on node behavior/version).
  - Vector size mismatch (collection created with wrong dimension) → Qdrant rejects inserts.
  - If Qdrant requires an API key and credentials are missing/incorrect → auth errors.
- **Version-specific notes:**
  - Node typeVersion `1.3`: ensure your n8n includes the LangChain Qdrant integration version that supports “insert” mode and AI ports.

---

### Block C — Query / Chat RAG (Chat Trigger → Agent → Retrieval Tool → LLM)
**Overview:** Receives chat messages, uses an agent powered by an Ollama chat model, enforces retrieval from Qdrant as a tool, and uses short-window memory for continuity.

**Nodes involved:**
- When chat message received
- AI Agent
- Ollama Chat Model
- Read from Qdrant Vector Store
- Simple Memory
- Embeddings Ollama (shared dependency for retrieval)

#### Node: When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` (entry point for n8n chat)
- **Configuration choices:**
  - Public: **true**
  - Response mode: **lastNode** (final reply is taken from the last node in the chain, typically the Agent)
  - Initial message:
    - “Kwema ? 👋  
      My name is KB. How can I assist you today?”
- **Connections:**
  - **Output (main)** → AI Agent (main)
- **Edge cases / failures:**
  - If public exposure is not desired, disable public or secure access.
  - If n8n instance base URL is not configured correctly, chat UI/webhook routing may fail.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (tool-using agent orchestration)
- **Configuration choices:**
  - System message (key behavior constraint):
    - “FORCE TOOL USAGE. DO NOT guess. ALWAYS use the Vector Store tool before answering. Fetch relevant info from the Qdrant vector store. Use the retrieved results to answer the user query.”
- **Inputs:**
  - **main:** user message from chat trigger
  - **ai_languageModel:** from Ollama Chat Model
  - **ai_memory:** from Simple Memory
  - **ai_tool:** from Read from Qdrant Vector Store
- **Outputs:**
  - Final assistant response returned to chat (because chat trigger responseMode = lastNode)
- **Edge cases / failures:**
  - If the tool call fails (Qdrant down, embedding failure), the agent may fail to respond or may violate “do not guess” directive and return an error-like response depending on agent implementation.
  - Model/tool latency can cause timeouts on longer queries.
- **Version-specific notes:**
  - Node typeVersion `3.1`: ensure n8n + LangChain nodes are recent enough for tool-based agent wiring (ai_tool/ai_memory ports).

#### Node: Ollama Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOllama` (LLM for generation)
- **Configuration choices:**
  - Model: `mistral:7b`
  - Option: `topK = -1` (implementation-dependent; generally indicates “no top-k restriction” or default behavior)
  - Credentials: “Ollama account”
- **Connections:**
  - **Output (ai_languageModel)** → AI Agent (ai_languageModel)
- **Edge cases / failures:**
  - Model not installed (`ollama pull mistral:7b`) → failures.
  - Insufficient RAM/CPU on host for the chosen model → slow responses/timeouts.

#### Node: Read from Qdrant Vector Store
- **Type / role:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant` (retriever exposed as an agent tool)
- **Configuration choices:**
  - Mode: **retrieve-as-tool**
  - Collection: `knowledge-base`
  - `topK = 3` (returns 3 most relevant chunks)
  - Tool description: “ALWAYS Use this knowledge base to answer questions from the user”
  - `includeDocumentMetadata = false` (only returns text chunks, no metadata)
- **Inputs:**
  - **ai_embedding:** from Embeddings Ollama (used to embed the query for similarity search)
- **Connections:**
  - **Output (ai_tool)** → AI Agent (ai_tool)
- **Edge cases / failures:**
  - If embeddings model dimension doesn’t match stored vectors, retrieval fails.
  - If collection is empty, retrieval returns no context; agent must handle “no results” gracefully.
  - Qdrant connectivity/auth issues.

#### Node: Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` (conversation memory)
- **Configuration choices:**
  - Default parameters (buffer-window behavior; keeps recent messages up to node default window size)
- **Connections:**
  - **Output (ai_memory)** → AI Agent (ai_memory)
- **Edge cases / failures:**
  - Memory is ephemeral per chat session/execution depending on n8n chat implementation; not a durable store unless configured otherwise in the platform.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Operator/setup instructions | — | — | ## Try It … (setup Ollama/Qdrant, usage steps, Discord/Forum links) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section label for ingestion | — | — | ## 1. Update Knowledge base |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section label for query | — | — | ## 2. Query Knowledge base |
| Upload document | n8n-nodes-base.formTrigger | Upload entry point (file + name) | — | Add to Qdrant Vector Store | ## 1. Update Knowledge base |
| Default Data Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Parse/split uploaded file into documents | — (fed indirectly by execution/binary context) | Add to Qdrant Vector Store | ## 1. Update Knowledge base |
| Embeddings Ollama | @n8n/n8n-nodes-langchain.embeddingsOllama | Generate embeddings (documents + queries) | — | Add to Qdrant Vector Store; Read from Qdrant Vector Store |  |
| Add to Qdrant Vector Store | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Insert documents+vectors into Qdrant | Upload document; Default Data Loader; Embeddings Ollama | — | ## 1. Update Knowledge base |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | AI Agent | ## 2. Query Knowledge base |
| Ollama Chat Model | @n8n/n8n-nodes-langchain.lmChatOllama | LLM generation backend | — | AI Agent | ## 2. Query Knowledge base |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Short chat memory | — | AI Agent | ## 2. Query Knowledge base |
| Read from Qdrant Vector Store | @n8n/n8n-nodes-langchain.vectorStoreQdrant | Retrieval tool (topK=3) for agent | Embeddings Ollama | AI Agent | ## 2. Query Knowledge base |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates tool use + response | When chat message received; Ollama Chat Model; Simple Memory; Read from Qdrant Vector Store | — (returns chat response) | ## 2. Query Knowledge base |

---

## 4. Reproducing the Workflow from Scratch

1. **Prepare local services**
   1) Install **Ollama** on the host running n8n (or reachable from it). Ensure it listens on `http://localhost:11434` (or adjust accordingly).  
   2) Pull required models:
      - `ollama pull mistral:7b`
      - `ollama pull nomic-embed-text:latest`
      - (Optional per note) `ollama pull llama3:8b`
   3) Run **Qdrant** (example via Docker):  
      - `docker run -p 6333:6333 qdrant/qdrant`
   4) In Qdrant dashboard (`http://localhost:6333/dashboard`), **create collection**:
      - Name: `knowledge-base`
      - Vector size: **768** (must match `nomic-embed-text` dimensionality as expected here)

2. **Create credentials in n8n**
   1) Create **Ollama API** credential:
      - Base URL: `http://localhost:11434` (or your Ollama host)
   2) Create **Qdrant API** credential:
      - Base URL: `http://localhost:6333`
      - API key: only if your Qdrant setup requires it

3. **Build Block B (Ingestion)**
   1) Add node **Form Trigger** named **“Upload document”**
      - Form title: `Upload`
      - Form description: `Poor mans Knowledge base`
      - Fields:
        - Text field label: `Doc Name`
        - File field label: `File`
   2) Add node **Default Data Loader**
      - Data type: **Binary**
      - Options: enable **Split pages**
   3) Add node **Embeddings Ollama**
      - Model: `nomic-embed-text:latest`
      - Select your **Ollama API** credential
   4) Add node **Qdrant Vector Store** named **“Add to Qdrant Vector Store”**
      - Mode: **Insert**
      - Collection: choose `knowledge-base`
      - Select your **Qdrant API** credential
   5) Connect ingestion nodes:
      - Connect **Upload document (main)** → **Add to Qdrant Vector Store (main)**
      - Connect **Default Data Loader (ai_document)** → **Add to Qdrant Vector Store (ai_document)**
      - Connect **Embeddings Ollama (ai_embedding)** → **Add to Qdrant Vector Store (ai_embedding)**

4. **Build Block C (Chat Query / RAG)**
   1) Add node **When chat message received**
      - Public: enabled (if desired)
      - Response mode: **Last node**
      - Initial messages: set to the provided greeting text
   2) Add node **Ollama Chat Model**
      - Model: `mistral:7b`
      - Option `topK`: `-1`
      - Select **Ollama API** credential
   3) Add node **Simple Memory** (Buffer Window Memory)
      - Keep defaults
   4) Add node **Qdrant Vector Store** named **“Read from Qdrant Vector Store”**
      - Mode: **Retrieve as tool**
      - Collection: `knowledge-base`
      - `topK`: `3`
      - Tool description: `ALWAYS Use this knowledge base to answer questions from the user`
      - `includeDocumentMetadata`: disabled
      - Select **Qdrant API** credential
   5) Add node **AI Agent**
      - System message: paste the “FORCE TOOL USAGE…” instruction block
   6) Connect query nodes:
      - **When chat message received (main)** → **AI Agent (main)**
      - **Ollama Chat Model (ai_languageModel)** → **AI Agent (ai_languageModel)**
      - **Simple Memory (ai_memory)** → **AI Agent (ai_memory)**
      - **Read from Qdrant Vector Store (ai_tool)** → **AI Agent (ai_tool)**
      - **Embeddings Ollama (ai_embedding)** → **Read from Qdrant Vector Store (ai_embedding)**  
        (reuse the same embeddings node from ingestion)

5. **Validate end-to-end**
   1) Execute workflow and open the **Upload** form → upload a document.
   2) Open chat → ask questions that should be answered from the uploaded content.
   3) If retrieval returns nothing, verify:
      - Qdrant collection exists and has points
      - Vector size is correct (768)
      - Ollama embedding model is installed and reachable

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Join the Discord | https://discord.com/invite/XPKeKXeB7d |
| Ask in the Forum | https://community.n8n.io/ |
| Ollama install (as shown in the note) | `curl -fsSL https://ollama.com/install.sh \| sh` |
| Qdrant run example | `docker run -p 6333:6333 qdrant/qdrant` |
| Required Ollama models mentioned | `mistral:7b`, `nomic-embed-text:latest` (also mentions `llama3:8b`) |
| Qdrant collection requirement | Collection `knowledge-base`, vector length **768** |
| Persistence warning | Use a persistent Docker volume for Qdrant if you want data to survive container recreation |