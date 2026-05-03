Search hardware inventory with Supabase vector RAG and Google Gemini

https://n8nworkflows.xyz/workflows/search-hardware-inventory-with-supabase-vector-rag-and-google-gemini-13410


# Search hardware inventory with Supabase vector RAG and Google Gemini

## 1. Workflow Overview

**Purpose:**  
This n8n workflow implements an AI-assisted hardware inventory assistant using **Supabase Vector Store (Vector RAG)** for scalable semantic search and **Google Gemini** for embeddings and chat responses. It supports two main use cases:
- **Path A (Indexing / Sync):** Ingest rows from a Google Sheet (hardware pricelist), convert each row into a document with metadata, generate embeddings, and insert into Supabase.
- **Path B (Chat / Retrieval):** Provide a chat interface where an AI Agent uses a Supabase-backed “inventory_search” tool (vector retrieval) to answer user questions about products, prices, and categories, with conversational memory.

### 1.1 Path A — Inventory Indexing (Google Sheets → Supabase Vector DB)
- Manual trigger → read sheet rows → load each row into a LangChain document (text + metadata) → embed via Gemini → insert into Supabase table `documents`.

### 1.2 Path B — Chat Agent (Chat Trigger → Agent → Tool-based Retrieval)
- Chat trigger → AI Agent (Gemini LLM + memory) → uses Supabase Vector Store retrieval tool to search inventory semantically and respond using system rules.

---

## 2. Block-by-Block Analysis

### Block A — Inventory Indexing to Supabase (Vector RAG ingestion)

**Overview:**  
This block is designed to populate/refresh the Supabase vector database from a Google Sheet. It transforms each row into a searchable document, attaches metadata (SKU, quantity, category), generates embeddings with Gemini, and inserts documents into Supabase.

**Nodes Involved:**
- When clicking ‘Execute workflow’
- Get row(s) in sheet
- Default Data Loader
- Embeddings Google Gemini
- Supabase Vector Store

#### 2.1 When clicking ‘Execute workflow’
- **Type / Role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts the indexing run on demand.
- **Configuration (interpreted):** No parameters; user runs it manually.
- **Connections:**
  - **Output →** `Get row(s) in sheet` (main)
- **Edge cases / failures:**
  - None specific; only runs when manually executed.

#### 2.2 Get row(s) in sheet
- **Type / Role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads inventory rows from a spreadsheet.
- **Configuration (interpreted):**
  - Auth: **Service Account** (`googleApi` credential).
  - Document: “Hardware Pricelist” (Spreadsheet ID: `1wvnyqtsrlwZwpZG9GpBhmc1S0n_bH4jcXE5paMdXaxc`)
  - Sheet tab: `Sheet1` (`gid=0`)
  - Operation implied by node name: fetch rows (exact operation not shown in JSON, but node is configured as a “Get row(s)” style node).
- **Connections:**
  - **Input ←** `When clicking ‘Execute workflow’`
  - **Output →** `Supabase Vector Store` (main)
- **Key data expectations:**
  - Rows should include at least these columns used later:
    - `SKU`, `Quantity`, `Category`, `Name`, `Price`
- **Edge cases / failures:**
  - Service account not shared on the spreadsheet → 403/permission errors.
  - Column name mismatches (e.g., `Sku` vs `SKU`) → downstream expressions evaluate to empty.
  - Large sheets → longer execution times; potential pagination/limits depending on node settings.

#### 2.3 Default Data Loader
- **Type / Role:** LangChain Document Loader (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) — converts each row JSON into a “document” with text content + metadata for embedding/storage.
- **Configuration (interpreted):**
  - **Text/content (jsonData):**  
    `Category: {{ $json.Category }}, Product: {{ $json.Name }}, Price: {{ $json.Price }}`
  - **Metadata fields added:**
    - `sku` = `{{ $json.SKU }}`
    - `quantity` = `{{ $json.Quantity }}`
    - `category` = `{{ $json.Category }}`
  - Mode: expression-driven (“jsonMode”: expressionData)
- **Connections:**
  - **Output (ai_document) →** `Supabase Vector Store` (ai_document)
- **Edge cases / failures:**
  - Any missing fields (Category/Name/Price/SKU/Quantity) reduce retrieval quality or produce incomplete metadata.
  - If `Price` contains non-text formats, it will still be embedded as text; formatting inconsistencies can hurt search accuracy.

#### 2.4 Embeddings Google Gemini
- **Type / Role:** Embeddings Model (`@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`) — generates vector embeddings for the documents.
- **Configuration (interpreted):**
  - Model: `models/gemini-embedding-001`
  - Credential: Google Gemini(PaLM) API
- **Connections:**
  - **Output (ai_embedding) →** `Supabase Vector Store` (ai_embedding)
- **Edge cases / failures:**
  - Invalid/expired API key → auth failures.
  - Rate limits / quota exhaustion → intermittent 429 errors.
  - Large batch sizes → timeouts depending on n8n/Gemini limits.

#### 2.5 Supabase Vector Store
- **Type / Role:** Supabase Vector Store (`@n8n/n8n-nodes-langchain.vectorStoreSupabase`) — inserts documents + embeddings into Supabase.
- **Configuration (interpreted):**
  - Mode: **insert**
  - Table: `documents`
  - Query function name: `match_documents` (used by Supabase to perform vector similarity search; also referenced by retrieval tool in Path B)
  - Credential: Supabase account (commonly uses Service Role Key for writes)
- **Connections:**
  - **Input (main) ←** `Get row(s) in sheet`
  - **Input (ai_document) ←** `Default Data Loader`
  - **Input (ai_embedding) ←** `Embeddings Google Gemini`
- **Version-specific notes:**
  - Uses node version `1.3`; ensure the Supabase Vector Store node is available in your n8n build that includes LangChain nodes.
- **Edge cases / failures:**
  - Supabase schema not created (missing `documents` table or `match_documents` function) → insert/query errors.
  - Using anon key instead of service role key → permission denied on inserts.
  - Inconsistent row IDs / duplicates: without a defined upsert key, repeated runs may create duplicates unless the node/table is configured to deduplicate (not shown here).

---

### Block B — Chat Agent with Vector Retrieval Tool (Production interface)

**Overview:**  
This block exposes a chat-triggered AI agent that answers inventory questions. The agent uses Gemini as its language model, keeps short-term conversation memory, and **must** use a Supabase vector retrieval tool for inventory queries.

**Nodes Involved:**
- When chat message received
- AI Agent
- Google Gemini
- Simple Memory
- Supabase Vector Store1
- Embeddings Google Gemini1

#### 2.6 When chat message received
- **Type / Role:** Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) — entry point for chat-based executions (e.g., n8n Chat UI).
- **Configuration (interpreted):**
  - Webhook-based trigger for chat messages (webhookId present).
  - No special options configured.
- **Connections:**
  - **Output →** `AI Agent` (main)
- **Edge cases / failures:**
  - If chat channel/UI is not configured or webhook not reachable, messages won’t trigger executions.

#### 2.7 AI Agent
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates reasoning, tool use, and response generation.
- **Configuration (interpreted):**
  - **System message** defines behavior as a “Hardware Sales Specialist”.
  - Key rules:
    1. **Always use** `inventory_search` tool for products/prices/availability queries.
    2. Use tool for category listings (e.g., monitors).
    3. Preserve exact model details returned by tool.
    4. Use memory for follow-ups (“How much is it?”).
    5. Put **Price first**; mention stock/quantity **only if explicitly asked**.
    6. If not found, suggest similar items or spelling check.
  - Tone: professional, helpful, technically knowledgeable.
- **Connections:**
  - **Input (main) ←** `When chat message received`
  - **Input (ai_languageModel) ←** `Google Gemini`
  - **Input (ai_memory) ←** `Simple Memory`
  - **Input (ai_tool) ←** `Supabase Vector Store1` (this becomes the agent tool; described as inventory search)
- **Edge cases / failures:**
  - If the tool is unavailable or errors, the agent may fail or provide lower-quality responses (depending on agent configuration).
  - If memory is too short (buffer window settings), follow-up resolution can be inconsistent.

#### 2.8 Google Gemini
- **Type / Role:** Chat LLM (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — generates final natural language responses.
- **Configuration (interpreted):**
  - Uses Google Gemini(PaLM) API credential.
  - No explicit model/options configured in the node (defaults apply).
- **Connections:**
  - **Output (ai_languageModel) →** `AI Agent`
- **Edge cases / failures:**
  - Credential issues, quota/rate limits, latency/timeouts.

#### 2.9 Simple Memory
- **Type / Role:** Memory Buffer Window (`@n8n/n8n-nodes-langchain.memoryBufferWindow`) — keeps recent conversation context for follow-up questions.
- **Configuration (interpreted):**
  - Default settings (buffer window size not specified in JSON).
- **Connections:**
  - **Output (ai_memory) →** `AI Agent`
- **Edge cases / failures:**
  - Small/default window may forget older references.
  - If the user references an item discussed far earlier, the agent may not resolve it correctly.

#### 2.10 Supabase Vector Store1 (Retrieval Tool)
- **Type / Role:** Supabase Vector Store (`@n8n/n8n-nodes-langchain.vectorStoreSupabase`) in **retrieve-as-tool** mode — exposes semantic search as an agent tool.
- **Configuration (interpreted):**
  - Mode: **retrieve-as-tool**
  - Table: `documents`
  - Query function: `match_documents`
  - Tool description: “Use this tool to find information about computer hardware inventory, including categories, prices, and technical specifications.”
  - Credential: Supabase account
- **Connections:**
  - **Output (ai_tool) →** `AI Agent`
  - **Input (ai_embedding) ←** `Embeddings Google Gemini1`
- **Tool naming note:**
  - The system prompt refers to the tool as `'inventory_search'`. In n8n, the actual exposed tool name may be derived from the node/tool configuration. Ensure the tool name presented to the agent matches the prompt expectation (or adjust the system prompt to match the actual tool name).
- **Edge cases / failures:**
  - Missing table/function in Supabase → retrieval errors.
  - Embedding model mismatch between indexing and querying can reduce similarity accuracy (here both use the same model, which is correct).

#### 2.11 Embeddings Google Gemini1
- **Type / Role:** Embeddings Model for Retrieval (`@n8n/n8n-nodes-langchain.embeddingsGoogleGemini`) — embeds user queries for vector search.
- **Configuration (interpreted):**
  - Model: `models/gemini-embedding-001`
  - Credential: Google Gemini(PaLM) API
- **Connections:**
  - **Output (ai_embedding) →** `Supabase Vector Store1`
- **Edge cases / failures:**
  - Same as embeddings in Path A: quota, auth, rate limits.

---

### Block C — Documentation / On-canvas Notes

**Overview:**  
A sticky note explains the intent, setup steps, and links to related resources.

**Nodes Involved:**
- Sticky Note

#### 2.12 Sticky Note
- **Type / Role:** Sticky Note (`n8n-nodes-base.stickyNote`) — documentation only.
- **Content highlights:**
  - Describes upgrade from “simple sheet reading” to Vector RAG
  - Setup steps: Supabase SQL, credentials, run Path A indexing, use Path B for chat, customize prompt
  - Includes links:
    - https://n8nplaybook.com/post/2026/02/simple-n8n-inventory-ai-agent/
    - https://n8nplaybook.com/post/2026/02/scaling-n8n-inventory-ai-agent-supabase-vector-rag/
- **Edge cases:** None (non-executable).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point (Path B) | — | AI Agent | ## Advanced AI Inventory Agent: Supabase Vector RAG & Gemini; Upgrades from simple sheet reading to Vector RAG; Setup: create Supabase table+function, connect creds, run Path A indexing, use Path B chat, customize prompt; Links: https://n8nplaybook.com/post/2026/02/simple-n8n-inventory-ai-agent/ and https://n8nplaybook.com/post/2026/02/scaling-n8n-inventory-ai-agent-supabase-vector-rag/ |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Agent orchestration + tool use | When chat message received; Google Gemini; Simple Memory; Supabase Vector Store1 | — | (same sticky note content as above) |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversational memory | — | AI Agent | (same sticky note content as above) |
| Google Gemini | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM responses for agent | — | AI Agent | (same sticky note content as above) |
| Supabase Vector Store1 | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Retrieval tool (vector search) for agent | Embeddings Google Gemini1 | AI Agent | (same sticky note content as above) |
| Embeddings Google Gemini1 | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Query embedding for retrieval | — | Supabase Vector Store1 | (same sticky note content as above) |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual entry point (Path A) | — | Get row(s) in sheet | (same sticky note content as above) |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Read inventory rows from Google Sheets | When clicking ‘Execute workflow’ | Supabase Vector Store | (same sticky note content as above) |
| Default Data Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Build document text + metadata per row | — | Supabase Vector Store | (same sticky note content as above) |
| Embeddings Google Gemini | @n8n/n8n-nodes-langchain.embeddingsGoogleGemini | Document embedding for indexing | — | Supabase Vector Store | (same sticky note content as above) |
| Supabase Vector Store | @n8n/n8n-nodes-langchain.vectorStoreSupabase | Insert documents+embeddings into Supabase | Get row(s) in sheet; Default Data Loader; Embeddings Google Gemini | — | (same sticky note content as above) |
| Sticky Note | n8n-nodes-base.stickyNote | On-canvas documentation | — | — | ## Advanced AI Inventory Agent: Supabase Vector RAG & Gemini; Upgrades from simple sheet reading to Vector RAG; Setup: create Supabase table+function, connect creds, run Path A indexing, use Path B chat, customize prompt; Links: https://n8nplaybook.com/post/2026/02/simple-n8n-inventory-ai-agent/ and https://n8nplaybook.com/post/2026/02/scaling-n8n-inventory-ai-agent-supabase-vector-rag/ |

---

## 4. Reproducing the Workflow from Scratch

### Path A — Build the Indexing Flow (Google Sheets → Supabase)

1. **Create Trigger**
   1. Add node: **Manual Trigger**
   2. Name it: `When clicking ‘Execute workflow’`

2. **Add Google Sheets Reader**
   1. Add node: **Google Sheets**
   2. Operation: “Get row(s)” (or equivalent to read all rows)
   3. Authentication: **Service Account**
   4. Configure credentials:
      - Create/attach a **Google Service Account** credential in n8n.
      - Share the spreadsheet with the service account email.
   5. Select:
      - Document: your inventory spreadsheet (e.g., “Hardware Pricelist”)
      - Sheet: the target tab (e.g., “Sheet1”)
   6. Connect: `Manual Trigger` → `Google Sheets` (main)

3. **Add Document Builder**
   1. Add node: **Default Data Loader** (LangChain)
   2. Configure it to create document text from each row:
      - Content/text expression (example):
        - `Category: {{ $json.Category }}, Product: {{ $json.Name }}, Price: {{ $json.Price }}`
   3. Add metadata mappings:
      - `sku` → `{{ $json.SKU }}`
      - `quantity` → `{{ $json.Quantity }}`
      - `category` → `{{ $json.Category }}`

4. **Add Embeddings Model (for indexing)**
   1. Add node: **Embeddings Google Gemini**
   2. Model: `models/gemini-embedding-001`
   3. Credentials:
      - Create/attach **Google Gemini(PaLM) API** credential (API key)

5. **Add Supabase Vector Store (Insert)**
   1. Add node: **Supabase Vector Store**
   2. Mode: **Insert**
   3. Table name: `documents`
   4. Query/function name: `match_documents`
   5. Credentials:
      - Create/attach a **Supabase** credential (prefer **Service Role Key** for writes)
   6. Connect:
      - `Google Sheets` → `Supabase Vector Store` (main)
      - `Default Data Loader` → `Supabase Vector Store` (ai_document)
      - `Embeddings Google Gemini` → `Supabase Vector Store` (ai_embedding)

6. **Supabase prerequisites (must exist before running)**
   - Create a `documents` table suitable for vector embeddings and metadata.
   - Create a `match_documents` SQL function for similarity search (the workflow expects this name).
   - Ensure extensions (commonly `pgvector`) are enabled if your schema uses vector columns.

---

### Path B — Build the Chat Retrieval Agent (Chat → Agent → Tool)

7. **Create Chat Entry**
   1. Add node: **When chat message received** (Chat Trigger)

8. **Add LLM**
   1. Add node: **Google Gemini** (Chat Model)
   2. Attach the same **Google Gemini(PaLM) API** credential (or another with sufficient quota)

9. **Add Memory**
   1. Add node: **Simple Memory** (Memory Buffer Window)
   2. Leave defaults or configure buffer size to match your needs.

10. **Add Embeddings Model (for retrieval queries)**
   1. Add node: **Embeddings Google Gemini**
   2. Model: `models/gemini-embedding-001`
   3. Same Gemini API credential as above.

11. **Add Supabase Vector Store (Retrieve as Tool)**
   1. Add node: **Supabase Vector Store**
   2. Mode: **Retrieve-as-tool**
   3. Table name: `documents`
   4. Query/function name: `match_documents`
   5. Tool description (recommended): explain it searches inventory with categories/prices/specs
   6. Connect:
      - `Embeddings Google Gemini (retrieval)` → `Supabase Vector Store (tool)` (ai_embedding)

12. **Add AI Agent**
   1. Add node: **AI Agent**
   2. Set the **System Message** to your sales/inventory rules (including “Price first” and “use tool for inventory questions”).
   3. Connect:
      - `When chat message received` → `AI Agent` (main)
      - `Google Gemini (LLM)` → `AI Agent` (ai_languageModel)
      - `Simple Memory` → `AI Agent` (ai_memory)
      - `Supabase Vector Store (retrieve-as-tool)` → `AI Agent` (ai_tool)

13. **Verify tool naming vs prompt**
   - If the agent does not recognize the tool as `inventory_search`, either:
     - Adjust the system prompt to reference the actual tool name shown in n8n, or
     - Configure/rename the tool (where supported) so the agent sees it as `inventory_search`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Upgrades an inventory assistant from simple sheet reading to high-performance Vector RAG for fast semantic search (handles typos/terminology). | Sticky note description in the workflow canvas |
| Setup steps: create Supabase `documents` + `match_documents`, connect Supabase + Gemini credentials, run Path A to index, use Path B for production chat, customize system prompt. | Sticky note description in the workflow canvas |
| Related article: simple sheet-based inventory agent | https://n8nplaybook.com/post/2026/02/simple-n8n-inventory-ai-agent/ |
| Related article: scaling inventory agent with Supabase vector RAG | https://n8nplaybook.com/post/2026/02/scaling-n8n-inventory-ai-agent-supabase-vector-rag/ |