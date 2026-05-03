Build a website-powered customer support chatbot with Decodo, Pinecone and Gemini

https://n8nworkflows.xyz/workflows/build-a-website-powered-customer-support-chatbot-with-decodo--pinecone-and-gemini-12146


# Build a website-powered customer support chatbot with Decodo, Pinecone and Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

This workflow builds an AI-powered customer support chatbot backed by your website content. It has two main tracks:

### 1.1 Knowledge Base Ingestion (Website → Pinecone)
Accepts either a sitemap URL or a comma-separated list of page URLs, fetches and cleans the pages’ text content (via Decodo + HTML extraction), generates embeddings with Google Gemini, and stores vectors in a Pinecone index (`supportbot`). The insert step clears the namespace each run (effectively rebuilding the KB).

### 1.2 Chat Interface (RAG over Pinecone)
Exposes a public webhook-based chat entrypoint that can be embedded on any website. An AI Agent (LangChain) uses Gemini as the chat model, keeps short conversation memory (last 10 turns), and calls a Pinecone retrieval tool (RAG) to answer with grounded business information.

### 1.3 Vector Retrieval Tooling
Configures Pinecone as an agent tool (“retrieve-as-tool”) using Gemini embeddings for query vectorization, enabling the agent to search the same `supportbot` index populated in ingestion.

---

## 2. Block-by-Block Analysis

### Block A — Input Reception & URL Processing
**Overview:** Collects user input (sitemap or explicit page URLs), routes the flow accordingly, normalizes URL format, merges sources, and removes duplicates before crawling.  
**Nodes involved:** `Input Sitemap or page urls`, `Switch`, `Fetch Sitemap`, `XML Conversion`, `Extract Page URLs`, `Split Pages URL`, `Merge URLs`, `Remove Duplicate URLs`

#### Node: Input Sitemap or page urls
- **Type / role:** Form Trigger; manual entrypoint to start ingestion.
- **Config (interpreted):**
  - Form title: “Agent Knowledge Base Input”
  - Fields:
    - `Sitemap URL` (text)
    - `Page URLs` (textarea; comma-separated URLs)
  - Description: prompts for sitemap or pages.
- **Outputs:** Connects to `Switch`.
- **Failure/edge cases:** Empty input for both fields results in no matching switch path (ingestion won’t proceed).

#### Node: Switch
- **Type / role:** Switch/router; chooses between “Page URLs” path and “Sitemap URL” path.
- **Config:**
  - **Rule 1:** If `{{$json['Page URLs']}}` is **not empty** → output 1 to `Split Pages URL`.
  - **Rule 2:** If `{{$json['Sitemap URL']}}` **endsWith** `"xml"` → output 2 to `Fetch Sitemap`.
  - `allMatchingOutputs: true` means if both are provided and match, **both branches** can run and later merge.
- **Inputs:** From form trigger.
- **Outputs:** To `Split Pages URL` and/or `Fetch Sitemap`.
- **Edge cases:**
  - Sitemap URL not ending in `xml` won’t match even if valid.
  - If both provided, both will run (intended), but can increase crawl volume.

#### Node: Fetch Sitemap
- **Type / role:** HTTP Request; downloads sitemap XML.
- **Config:**
  - URL: `={{ $json['Sitemap URL'] }}`
- **Outputs:** To `XML Conversion`.
- **Failure types:** HTTP 4xx/5xx, redirects, blocked by WAF, invalid SSL, timeout.

#### Node: XML Conversion
- **Type / role:** XML node; converts sitemap XML to JSON.
- **Config:** default options.
- **Outputs:** To `Extract Page URLs`.
- **Edge cases:** Non-standard sitemap structure may not map to `urlset.url` as expected.

#### Node: Extract Page URLs
- **Type / role:** Code node; flattens sitemap JSON into items of `{ url }`.
- **Key logic:** Iterates `$input.first().json.urlset.url` and pushes `{ url: item.loc }`.
- **Outputs:** To `Merge URLs` (input index 1).
- **Edge cases / failures:**
  - If sitemap JSON doesn’t have `urlset.url`, code throws.
  - If `loc` missing, you may create undefined URLs.

#### Node: Split Pages URL
- **Type / role:** Code node; splits comma-separated URLs and normalizes.
- **Key logic:**
  - Splits `$input.first().json['Page URLs']` by commas.
  - Trims and **adds trailing slash** if missing.
  - Returns items: `{ url: 'https://…/' }`
- **Outputs:** To `Merge URLs` (input index 0).
- **Edge cases:**
  - If “Page URLs” contains newlines or semicolons, they won’t be split.
  - Adds trailing slash unconditionally; may change canonical URL for some sites.

#### Node: Merge URLs
- **Type / role:** Merge; consolidates URLs from sitemap and manual list.
- **Config:** default merge behavior (two inputs).
- **Inputs:** From `Split Pages URL` and `Extract Page URLs`.
- **Outputs:** To `Remove Duplicate URLs`.
- **Edge cases:** If only one branch runs, ensure merge node behavior still passes items as expected (n8n merge modes can affect this; default typically “append” but verify in UI).

#### Node: Remove Duplicate URLs
- **Type / role:** Remove Duplicates; deduplicates URL items.
- **Config:** default options (no explicit field specified).
- **Outputs:** To `Loop Over Page URLs`.
- **Edge cases:** If not configured to dedupe by `url`, duplicates may persist depending on node defaults.

---

### Block B — Crawling & Content Extraction (Decodo + HTML)
**Overview:** Iterates through URLs, requests rendered HTML via Decodo (for JS-heavy sites) and extracts clean body text. Adds a delay to reduce rate/anti-bot triggers.  
**Nodes involved:** `Loop Over Page URLs`, `Decodo`, `Wait 5 sec`, `Extract Content`

#### Node: Loop Over Page URLs
- **Type / role:** Split In Batches; loops over URL items.
- **Config:** Options empty (defaults apply; batch size defaults depend on n8n version/UI).
- **Inputs:** From `Remove Duplicate URLs`.
- **Outputs:**
  - Main output 0 → `Extract Content`
  - Main output 1 → `Decodo`
  - (In this workflow both are connected from the same node output, effectively running both paths per batch item.)
- **Important behavior:** Both `Extract Content` and `Decodo` receive the same `json.url` items, but they do different things:
  - `Decodo` fetches HTML remotely (JS rendering)
  - `Extract Content` extracts from incoming HTML; however it currently receives the URL item, not Decodo HTML, so extraction may not work as intended unless `Decodo` returns HTML and is actually the intended upstream.
- **Potential issue:** The `Extract Content` node is connected directly from `Loop Over Page URLs`, not from `Decodo`. If `Extract Content` expects HTML content, you likely want: `Decodo → Extract Content` (or `HTTP Request → Extract Content`) rather than parallel.
- **Failure modes:** Infinite/incorrect batching if “Continue” output isn’t used and batches aren’t advanced. In typical patterns you connect the “Continue” output back into itself; here, the looping is handled differently (see `Wait 5 sec` returning into `Loop Over Page URLs`).

#### Node: Decodo
- **Type / role:** Decodo node; fetches page HTML with JS rendering (per sticky note).
- **Config:** URL: `={{ $json.url }}`
- **Outputs:** To `Wait 5 sec`.
- **Failure types:** Decodo credential/auth issues, blocked target site, timeouts, high latency, non-HTML responses.

#### Node: Wait 5 sec
- **Type / role:** Wait; throttling / pacing.
- **Config:** default (name implies 5 seconds, but no explicit parameters shown—verify in UI; some Wait nodes require a duration setting).
- **Inputs:** From `Decodo`.
- **Outputs:** To `Loop Over Page URLs` (creating a loop).
- **Edge cases:**
  - If duration isn’t actually set, it may not wait.
  - The loop design depends on SplitInBatches semantics; if miswired, it may repeatedly process the same batch.

#### Node: Extract Content
- **Type / role:** HTML node (`extractHtmlContent`); extracts main text from HTML.
- **Config:**
  - Operation: Extract HTML Content
  - Selector: `body`
  - Skip selectors: `img`
  - `cleanUpText: true` to normalize whitespace and remove noise.
  - Output key: `content`
- **Inputs:** Currently from `Loop Over Page URLs` (see note above—may not contain HTML).
- **Outputs:** To `Pinecone KnowledgeBase` (main insert stream).
- **Failure/edge cases:** If input doesn’t contain HTML, extraction returns empty `content` and you’ll store useless embeddings/vectors.

**Sticky note context (applies to this block):**
- “## Extracting HTML from a web page using Decodo with JS Rendering”

---

### Block C — Embedding + Pinecone Storage (Knowledge Base Build)
**Overview:** Converts extracted documents into embeddings using Gemini and inserts them into Pinecone index `supportbot`. Clears the namespace each run for a fresh KB rebuild.  
**Nodes involved:** `Gemini Embeddings`, `Data Loader`, `Pinecone KnowledgeBase`

#### Node: Data Loader
- **Type / role:** LangChain Document Default Data Loader; converts incoming text into LangChain document objects for vector store ingestion.
- **Config:** default options.
- **Connections:** Outputs via `ai_document` to `Pinecone KnowledgeBase`.
- **Important observation:** In the provided connections, **no node feeds into Data Loader**. That likely means the vector store insertion may not receive proper documents unless `Pinecone KnowledgeBase` can derive documents from the main input alone.
- **Common intended wiring:** `Extract Content → Data Loader → Pinecone KnowledgeBase (ai_document)`
- **Failure/edge cases:** If no documents are passed, Pinecone insert may do nothing or error depending on node implementation.

#### Node: Gemini Embeddings
- **Type / role:** Google Gemini embeddings provider for ingestion.
- **Config:** Model `models/gemini-embedding-001`
- **Connections:** Outputs `ai_embedding` to `Pinecone KnowledgeBase`.
- **Failure types:** Missing Google credentials, model not available in region/project, quota exceeded.

#### Node: Pinecone KnowledgeBase
- **Type / role:** Pinecone Vector Store (insert mode); stores vectors for RAG.
- **Config:**
  - Mode: `insert`
  - Index: `supportbot`
  - Option: `clearNamespace: true` (clears existing vectors in the chosen namespace before insert)
- **Inputs:**
  - Main input from `Extract Content`
  - `ai_embedding` from `Gemini Embeddings`
  - `ai_document` from `Data Loader`
- **Edge cases / failures:**
  - Clearing namespace each run may delete prior knowledge unexpectedly.
  - Pinecone auth/index mismatch, dimension mismatch between embedding model and index settings, rate limits.

**Sticky note context (applies to this block):**
- “## Saving Content in Pinecone Vector Database”

---

### Block D — Chatbot Entry Point + Memory
**Overview:** Provides a public chat webhook suitable for embedding, and maintains a rolling memory window for context continuity.  
**Nodes involved:** `When chat message received`, `Simple Memory`, `Simple Memory1`, `Google Gemini Chat Model`

#### Node: When chat message received
- **Type / role:** LangChain Chat Trigger; webhook entry for chat UI/widget.
- **Config:**
  - Mode: `webhook`
  - Public: `true`
  - Allowed origins: `*` (CORS open)
  - Load previous session: `memory` (loads conversation context from memory integration)
- **Outputs:** To `AI Agent`.
- **Failure/edge cases:**
  - Open CORS can be abused; restrict origins in production.
  - Session continuity depends on client passing consistent session identifiers.

#### Node: Simple Memory
- **Type / role:** Memory Buffer Window (LangChain); stores last N messages.
- **Config:** `contextWindowLength: 10`
- **Connections:** `ai_memory` → `When chat message received`
- **Notes:** This is an unusual direction; typically the trigger feeds into the agent and agent uses memory. Here memory is wired into the trigger (“loadPreviousSession: memory”), which may be correct for this node type.
- **Failure/edge cases:** Memory not persisted across restarts unless n8n/LangChain node persists it (often it’s ephemeral).

#### Node: Simple Memory1
- **Type / role:** Memory Buffer Window (LangChain); provides memory to the agent.
- **Config:** `contextWindowLength: 10`
- **Connections:** `ai_memory` → `AI Agent`
- **Edge cases:** Duplicating memory nodes (`Simple Memory` and `Simple Memory1`) can cause confusion; ensure the intended one is used.

#### Node: Google Gemini Chat Model
- **Type / role:** Chat LLM (Gemini) used by the agent.
- **Config:** default options (model selection not shown; uses node defaults unless set in UI).
- **Connections:** `ai_languageModel` → `AI Agent`
- **Failure types:** Google auth/quota, safety filters, latency/timeouts.

**Sticky note context (applies to this block):**
- “## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly”

---

### Block E — RAG Retrieval Tool (Pinecone as Agent Tool)
**Overview:** Enables the agent to retrieve relevant website knowledge from Pinecone using Gemini embeddings for semantic search.  
**Nodes involved:** `Pinecone Vector Store`, `Embeddings Google Gemini`, `AI Agent`

#### Node: Pinecone Vector Store
- **Type / role:** Pinecone Vector Store in `retrieve-as-tool` mode; exposes retrieval as an agent tool.
- **Config:**
  - Index: `supportbot`
  - Tool description: “Business information related to a business”
- **Connections:** `ai_tool` → `AI Agent`
- **Failure/edge cases:** If the index is empty (ingestion not run or failed), retrieval returns no context.

#### Node: Embeddings Google Gemini
- **Type / role:** Embeddings provider for query-time vectorization (RAG retrieval).
- **Config:** Model `models/embedding-001` (note: different name than ingestion node; ensure Pinecone index dimension matches this model’s output).
- **Connections:** `ai_embedding` → `Pinecone Vector Store`
- **Failure/edge cases:** Model mismatch with stored vectors can degrade retrieval or break if dimensions differ.

#### Node: AI Agent
- **Type / role:** LangChain Agent; orchestrates prompt, memory, tool calls, and final response.
- **Config:** options empty (defaults: agent type/strategy depends on node version).
- **Inputs:**
  - Main: from `When chat message received`
  - `ai_languageModel`: from `Google Gemini Chat Model`
  - `ai_memory`: from `Simple Memory1`
  - `ai_tool`: from `Pinecone Vector Store`
- **Outputs:** Responds back through the chat trigger system (handled internally by the chat trigger/agent integration).
- **Edge cases:** Tool call failures (Pinecone downtime), hallucination if retrieval returns nothing, large prompts if memory grows.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation header | — | — | # AI Chatbot for businesses using Decodo, Pinecone, and Google Gemini (full description of blocks) |
| Input Sitemap or page urls | Form Trigger | Collect sitemap/pages input | — | Switch | ## Processing Page URLs |
| Switch | Switch | Route sitemap vs page list | Input Sitemap or page urls | Split Pages URL; Fetch Sitemap | ## Processing Page URLs |
| Fetch Sitemap | HTTP Request | Download sitemap XML | Switch | XML Conversion | ## Processing Page URLs |
| XML Conversion | XML | Parse sitemap XML → JSON | Fetch Sitemap | Extract Page URLs | ## Processing Page URLs |
| Extract Page URLs | Code | Extract `loc` URLs from sitemap | XML Conversion | Merge URLs | ## Processing Page URLs |
| Split Pages URL | Code | Split comma-separated URLs, normalize | Switch | Merge URLs | ## Processing Page URLs |
| Merge URLs | Merge | Combine URL sources | Split Pages URL; Extract Page URLs | Remove Duplicate URLs | ## Processing Page URLs |
| Remove Duplicate URLs | Remove Duplicates | Deduplicate URLs | Merge URLs | Loop Over Page URLs | ## Processing Page URLs |
| Loop Over Page URLs | Split In Batches | Iterate through URLs | Remove Duplicate URLs; Wait 5 sec | Extract Content; Decodo | ## Extracting HTML from a web page using Decodo with JS Rendering |
| Decodo | Decodo | Fetch rendered HTML | Loop Over Page URLs | Wait 5 sec | ## Extracting HTML from a web page using Decodo with JS Rendering |
| Wait 5 sec | Wait | Throttle + loop continuation | Decodo | Loop Over Page URLs | ## Extracting HTML from a web page using Decodo with JS Rendering |
| Extract Content | HTML | Extract clean text from HTML | Loop Over Page URLs | Pinecone KnowledgeBase | ## Saving Content in Pinecone Vector Database |
| Data Loader | LangChain Document Loader | Build documents for vector insert | (none connected) | Pinecone KnowledgeBase | ## Saving Content in Pinecone Vector Database |
| Gemini Embeddings | Gemini Embeddings | Create embeddings for stored docs | (none connected) | Pinecone KnowledgeBase | ## Saving Content in Pinecone Vector Database |
| Pinecone KnowledgeBase | Pinecone Vector Store | Insert vectors into `supportbot` | Extract Content; Data Loader; Gemini Embeddings | — | ## Saving Content in Pinecone Vector Database |
| When chat message received | Chat Trigger | Public chat webhook entrypoint | Simple Memory | AI Agent | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| Simple Memory | Memory Buffer Window | Session memory for trigger | — | When chat message received | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| Google Gemini Chat Model | Gemini Chat Model | LLM for responses | — | AI Agent | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| Simple Memory1 | Memory Buffer Window | Conversation memory for agent | — | AI Agent | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| Embeddings Google Gemini | Gemini Embeddings | Query embeddings for retrieval tool | — | Pinecone Vector Store | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| Pinecone Vector Store | Pinecone Vector Store | Retrieval tool over `supportbot` | Embeddings Google Gemini | AI Agent | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| AI Agent | LangChain Agent | Orchestrate chat + tools + memory | When chat message received; Google Gemini Chat Model; Simple Memory1; Pinecone Vector Store | — | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Extracting HTML from a web page using Decodo with JS Rendering |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Saving Content in Pinecone Vector Database |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Processing Page URLs |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Chatbot which can be embedded to any website, will show on the bottom right side, and will work perfectly |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Input Sitemap or page urls” (Form Trigger)**
   - Add fields:
     - Text: `Sitemap URL` (placeholder like `https://website.com/page-sitemap.xml`)
     - Textarea: `Page URLs` (placeholder comma-separated)
   - Keep webhook enabled.

2) **Add a “Switch” node**
   - Rule A: `{{$json['Page URLs']}}` → **not empty**
   - Rule B: `{{$json['Sitemap URL']}}` → **endsWith** `xml`
   - Enable “All matching outputs”.

3) **Sitemap branch**
   - Add **HTTP Request** “Fetch Sitemap”
     - URL: `={{ $json['Sitemap URL'] }}`
   - Add **XML** “XML Conversion” (default).
   - Add **Code** “Extract Page URLs”:
     - Iterate `urlset.url` and return items `{ url: item.loc }`.

4) **Manual URL branch**
   - Add **Code** “Split Pages URL”:
     - Split `$json['Page URLs']` by comma
     - Trim each entry
     - Optionally normalize (trailing slash, as in workflow).

5) **Merge and dedupe**
   - Add **Merge** “Merge URLs” and connect:
     - Manual list into Input 1
     - Sitemap list into Input 2
   - Add **Remove Duplicates** “Remove Duplicate URLs”
     - Configure dedupe field as `url` (recommended, even if defaults work).

6) **Looping over URLs**
   - Add **Split In Batches** “Loop Over Page URLs”
     - Set a batch size (recommended: 1–5 for stability).
   - Connect `Remove Duplicate URLs → Loop Over Page URLs`.

7) **Fetch rendered HTML via Decodo**
   - Add **Decodo** node
     - URL: `={{ $json.url }}`
     - Configure Decodo credentials (API key / account as required by the Decodo n8n node).

8) **Throttle**
   - Add **Wait** node “Wait 5 sec”
     - Set wait duration to 5 seconds (verify it’s actually set in node parameters).

9) **Important wiring recommendation (to make extraction work)**
   - Connect: `Loop Over Page URLs → Decodo → Extract Content`
   - Then connect `Extract Content → Wait 5 sec → Loop Over Page URLs` (continue batching)
   - This ensures the HTML extractor receives actual HTML, not just a URL item.
   - (The provided JSON connects `Loop Over Page URLs` directly to `Extract Content`, which may not work unless the HTML node is configured to fetch by URL—which it is not here.)

10) **Extract text**
   - Add **HTML** node “Extract Content”
     - Operation: Extract HTML Content
     - Selector: `body`
     - Skip: `img`
     - Output key: `content`
     - Clean up text: enabled

11) **Prepare documents for Pinecone**
   - Add **LangChain Document Default Data Loader** “Data Loader”
     - Connect `Extract Content → Data Loader`
     - Ensure it maps `content` into document text (defaults often do, but confirm).

12) **Embeddings for ingestion**
   - Add **Google Gemini Embeddings** “Gemini Embeddings”
     - Model: `models/gemini-embedding-001`
     - Configure Google credentials (Google AI / Vertex AI depending on node implementation in your n8n).

13) **Pinecone insert**
   - Add **Pinecone Vector Store** “Pinecone KnowledgeBase”
     - Mode: `insert`
     - Index: `supportbot` (create in Pinecone beforehand with correct dimension)
     - Option: `clearNamespace: true` (only if you want to rebuild from scratch every run)
   - Connect:
     - `Data Loader` to Pinecone via `ai_document`
     - `Gemini Embeddings` to Pinecone via `ai_embedding`
     - (Optionally also connect main stream if required by your node version, but the key is `ai_document` + `ai_embedding`.)

14) **Chat entrypoint**
   - Add **Chat Trigger** “When chat message received”
     - Mode: webhook
     - Public: true
     - Allowed origins: `*` (or restrict)
     - Load previous session: memory

15) **Chat memory**
   - Add **Memory Buffer Window** “Simple Memory” (for trigger session loading)
     - contextWindowLength: 10
     - Connect `Simple Memory (ai_memory) → When chat message received`
   - Add **Memory Buffer Window** “Simple Memory1” (for agent)
     - contextWindowLength: 10
     - Connect `Simple Memory1 (ai_memory) → AI Agent`

16) **LLM**
   - Add **Google Gemini Chat Model** node
     - Configure credentials
     - Select a Gemini chat model if required in UI
     - Connect `ai_languageModel → AI Agent`

17) **RAG tool**
   - Add **Gemini Embeddings** “Embeddings Google Gemini”
     - Model: `models/embedding-001` (recommended to match ingestion model to avoid dimension mismatch; consider using the same embedding model as ingestion)
   - Add **Pinecone Vector Store** “Pinecone Vector Store”
     - Mode: `retrieve-as-tool`
     - Index: `supportbot`
     - Tool description: “Business information related to a business”
     - Connect `Embeddings Google Gemini (ai_embedding) → Pinecone Vector Store`
     - Connect `Pinecone Vector Store (ai_tool) → AI Agent`

18) **Agent**
   - Add **AI Agent**
     - Connect `When chat message received → AI Agent` (main)
     - Ensure it has:
       - `ai_languageModel` from Gemini chat model
       - `ai_memory` from memory
       - `ai_tool` from Pinecone retrieval tool

**Credentials to prepare**
- **Decodo**: Decodo account/API key configured in n8n credentials for the Decodo node.
- **Google Gemini (Embeddings + Chat)**: Google credentials as required by the node (API key or Google Cloud/Vertex credentials depending on your n8n setup).
- **Pinecone**: Pinecone API key/environment/project, and an index named `supportbot` with a dimension matching the embedding model used.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Chatbot for businesses using Decodo, Pinecone, and Google Gemini: end-to-end ingestion + web-embeddable chat, RAG over Pinecone, Gemini embeddings + chat model, memory window of 10. | Workflow sticky note (top) |
| Extracting HTML from a web page using Decodo with JS Rendering | Sticky note near Decodo/extraction loop |
| Saving Content in Pinecone Vector Database | Sticky note near embeddings/vector store insert |
| Processing Page URLs | Sticky note near sitemap/manual URL preprocessing |
| Chatbot embeddable on any website, bottom-right widget, intended to “work perfectly” | Sticky note near chat trigger/agent |

