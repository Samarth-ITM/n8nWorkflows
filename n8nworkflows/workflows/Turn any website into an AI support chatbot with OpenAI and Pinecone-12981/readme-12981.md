Turn any website into an AI support chatbot with OpenAI and Pinecone

https://n8nworkflows.xyz/workflows/turn-any-website-into-an-ai-support-chatbot-with-openai-and-pinecone-12981


# Turn any website into an AI support chatbot with OpenAI and Pinecone

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow turns a website into a retrieval-based AI customer-support chatbot by (1) crawling and chunking website pages, (2) generating OpenAI embeddings, (3) storing them in Pinecone, and (4) exposing a public chat webhook where an agent retrieves relevant chunks from Pinecone before answering.

**Primary use cases:**
- Fast “knowledge base” chatbot from existing public website content (products, policies, FAQs).
- Low-hallucination support assistant by forcing answers to come from retrieval results.

### 1.1 URL Discovery (Sitemap mapping)
Firecrawl maps the website sitemap and returns a list of URLs (optionally including subdomains).

### 1.2 URL Cleaning + Controlled Crawling
A Code node filters undesired URLs, normalizes them, and de-duplicates. SplitInBatches then processes URLs one by one.

### 1.3 Page Fetch + Text Extraction
Each URL is fetched via HTTP, HTML is extracted, then converted to plain text.

### 1.4 Validation + Chunking
Pages with too little text are skipped; valid pages are chunked with overlap for embedding.

### 1.5 Embedding + Pinecone Upsert (Ingestion)
Each chunk is wrapped as a document, embedded with OpenAI, and inserted into Pinecone under a namespace.

### 1.6 Chat Webhook + Agent Retrieval (Serving)
A public Chat Trigger receives user messages. A LangChain agent uses a chat model plus a Pinecone “retrieve-as-tool” integration and memory buffer to respond, with strict system rules to avoid hallucinations.

---

## 2. Block-by-Block Analysis

### Block 1 — URL Discovery (Sitemap mapping)

**Overview:**  
Maps the target website’s sitemap to produce a list of page URLs for ingestion.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Map Website URLs

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) to start ingestion on demand.
- **Configuration choices:** No parameters; used for manual runs while building or re-indexing.
- **Inputs / outputs:**  
  - Input: none  
  - Output → `Map Website URLs` (main)
- **Failure modes:** none (except platform execution issues).
- **Version notes:** typeVersion 1.

#### Node: Map Website URLs
- **Type / role:** Firecrawl node (`@mendable/n8n-nodes-firecrawl.firecrawl`) to map URLs from sitemap.
- **Configuration choices:**
  - **Operation:** `map`
  - **Sitemap:** `only` (restricts discovery to sitemap entries rather than crawling arbitrarily)
  - **URL:** `<your-website-url>` (must be replaced)
  - **Include subdomains:** enabled
  - **Retry on fail:** enabled (`retryOnFail: true`)
- **Key data produced:** typically `{ success: true, links: [ { url: "..." }, ... ] }`
- **Inputs / outputs:**  
  - Input: Manual trigger  
  - Output → `Filter and Normalize URLs`
- **Failure modes / edge cases:**
  - Firecrawl auth/plan limitations, invalid API key
  - Sitemap missing or blocked by robots/auth
  - Large sitemaps causing long runtimes/timeouts
  - Returned links may include unwanted sections (handled later)
- **Version notes:** typeVersion 1.

---

### Block 2 — URL Cleaning & Controlled Crawling

**Overview:**  
Filters out irrelevant URLs, normalizes them, removes duplicates, then processes URLs sequentially to control load.

**Nodes involved:**
- Filter and Normalize URLs
- Process URLs One by One

#### Node: Filter and Normalize URLs
- **Type / role:** Code (`n8n-nodes-base.code`) for URL filtering and normalization.
- **Configuration choices (interpreted):**
  - Reads `links` array from the previous node output.
  - Blocks URLs containing keywords/paths:
    - `/press-blog`, `/newsletter`, `/assets-vault`, `press`, `gallery`, `project`, `newsletter`
  - Excludes `.xml` URLs
  - Removes trailing `/`
  - De-duplicates via a `Set()`
  - Outputs items shaped as `{ url: "https://..." }`
- **Key expressions / variables:**
  - `$json.links` input
  - `blockedKeywords` list
  - `seen` set for dedupe
- **Inputs / outputs:**  
  - Input ← `Map Website URLs`  
  - Output → `Process URLs One by One`
- **Failure modes / edge cases:**
  - If `$json.links` is missing or not an array: outputs empty list (no ingestion)
  - Over-broad blocked keywords (e.g., “press” matches legitimate URLs)
  - Trailing slash normalization may cause collisions between distinct routes in rare setups
- **Version notes:** typeVersion 2.

#### Node: Process URLs One by One
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) for sequential processing.
- **Configuration choices:**
  - Batch sizing is not explicitly set here; default behavior applies (commonly 1 item per batch unless configured).
  - Used to avoid parallel fetching and reduce risk of rate limiting.
- **Inputs / outputs / looping:**
  - Input ← `Filter and Normalize URLs`
  - Output (batch items) → `Fetch Page Content`
  - After downstream completion, workflow routes back from `Store in Vector Database` → `Process URLs One by One` to continue the next batch.
- **Failure modes / edge cases:**
  - If upstream returns 0 URLs: nothing runs.
  - If batch size is changed, can increase load and trigger rate limits.
- **Version notes:** typeVersion 3.

---

### Block 3 — Page Fetch & Text Extraction

**Overview:**  
Fetches each page’s HTML, normalizes fields, and converts HTML into plain text suitable for chunking and embeddings.

**Nodes involved:**
- Fetch Page Content
- Extract HTML and URL
- Clean HTML to Text

#### Node: Fetch Page Content
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) to fetch the page.
- **Configuration choices:**
  - **URL:** `={{ $json.url }}`
  - **Follow redirects:** enabled
  - **Retry on fail:** enabled
  - **Continue on fail:** enabled (important: ingestion continues even if some URLs fail)
- **Inputs / outputs:**
  - Input ← `Process URLs One by One`
  - Output → `Extract HTML and URL`
- **Failure modes / edge cases:**
  - 403/401 for protected pages
  - Rate limiting (429), transient failures (retry helps)
  - Non-HTML content (PDFs, binary) will produce unusable “HTML” for the simplistic cleaner
  - Because `continueOnFail` is true, downstream may receive error-shaped items; the Set node tries to mitigate by fallback expressions.
- **Version notes:** typeVersion 3.

#### Node: Extract HTML and URL
- **Type / role:** Set (`n8n-nodes-base.set`) to normalize expected fields for later steps.
- **Configuration choices:**
  - Sets:
    - `html = {{ $json.data ?? $json.html ?? "" }}`
    - `url = {{ $json.url ?? $('Process URLs One by One').item.json.url }}`
  - This handles differing HTTP response shapes and ensures `url` is preserved even on partial failures.
- **Inputs / outputs:**
  - Input ← `Fetch Page Content`
  - Output → `Clean HTML to Text`
- **Failure modes / edge cases:**
  - If neither `$json.data` nor `$json.html` exists, `html` becomes `""` and the page will be skipped later by length check.
  - If the batch reference `$('Process URLs One by One').item.json.url` is not available due to execution context mismatch, `url` could be empty (rare but possible when items are reshaped unexpectedly).
- **Version notes:** typeVersion 3.4.

#### Node: Clean HTML to Text
- **Type / role:** Code (`n8n-nodes-base.code`) to strip markup and produce plain text.
- **Configuration choices:**
  - Runs **once per item**
  - Removes `<script>` and `<style>` blocks, then strips remaining tags, collapses whitespace.
  - Returns `{ url, project_id, text }`
- **Key expressions / variables:**
  - Uses `$json.html` and `$json.url`
  - Returns `$json.project_id` even though upstream doesn’t set it in this workflow (will be `undefined` unless added elsewhere).
- **Inputs / outputs:**
  - Input ← `Extract HTML and URL`
  - Output → `Check Text Length`
- **Failure modes / edge cases:**
  - This is a very simple cleaner: navigation/footer boilerplate can dominate text.
  - If `html` is not a string (e.g., object), `.replace()` will throw.
  - Non-UTF8 or malformed HTML may yield poor text.
- **Version notes:** typeVersion 2.

---

### Block 4 — Validation & Chunking

**Overview:**  
Skips pages with minimal content and splits valid text into overlapping chunks for better retrieval quality.

**Nodes involved:**
- Check Text Length
- Split Text into Chunks

#### Node: Check Text Length
- **Type / role:** If (`n8n-nodes-base.if`) to filter out low-content pages.
- **Configuration choices:**
  - Condition evaluates: `={{ $json.text.length > 50 }}`
  - Only the **true** branch is connected (to chunking).
- **Inputs / outputs:**
  - Input ← `Clean HTML to Text`
  - True output → `Split Text into Chunks`
  - False output: not connected (items dropped)
- **Failure modes / edge cases:**
  - If `text` is undefined/null, `.length` can throw; in practice `Clean HTML to Text` always sets `text`, but if it fails and `continueOnFail` propagates unusual items, this could break.
- **Version notes:** typeVersion 2.3.

#### Node: Split Text into Chunks
- **Type / role:** Code (`n8n-nodes-base.code`) to chunk text for embeddings.
- **Configuration choices:**
  - Normalizes whitespace again.
  - `chunkSize = 1000`, `overlap = 200`
  - Iterates through text and outputs an array of items:
    - `{ content: chunk, url, project_id }`
  - Discards chunks shorter than 50 chars.
- **Inputs / outputs:**
  - Input ← `Check Text Length` (true path)
  - Output → `Store in Vector Database`
- **Failure modes / edge cases:**
  - Very large pages produce many chunks → higher cost/time.
  - Overlap/size are character-based, not token-based (may split awkwardly).
- **Version notes:** typeVersion 2.

---

### Block 5 — Embedding + Pinecone Upsert (Ingestion)

**Overview:**  
Converts chunks into documents, embeds them with OpenAI, and inserts them into Pinecone under a given namespace.

**Nodes involved:**
- Document Loader
- Embeddings OpenAI
- Store in Vector Database

#### Node: Document Loader
- **Type / role:** LangChain Document Loader (`@n8n/n8n-nodes-langchain.documentDefaultDataLoader`) to package each chunk as a document with metadata.
- **Configuration choices:**
  - Adds metadata:
    - `url = {{ $json.url }}`
    - `content = {{ $json.content }}`
  - Note: many vector-store patterns set the *pageContent* as the chunk and *metadata* as URL; here the metadata includes `content` as well, which may be redundant but workable depending on downstream node expectations.
- **Inputs / outputs:**
  - Output (ai_document) → `Store in Vector Database`
- **Connection note:** This node is not on the main ingestion line, but is wired into `Store in Vector Database` via the `ai_document` connection type.
- **Failure modes / edge cases:**
  - If `content` is missing, documents may be empty; ensure chunk node outputs `content`.
- **Version notes:** typeVersion 1.1.

#### Node: Embeddings OpenAI (Ingestion)
- **Type / role:** OpenAI Embeddings (`@n8n/n8n-nodes-langchain.embeddingsOpenAi`) to compute vectors for chunks.
- **Configuration choices:**
  - Uses OpenAI credentials: “n8n free OpenAI API credits”
  - No explicit model configured here (defaults depend on node/version and account availability).
- **Inputs / outputs:**
  - Output (ai_embedding) → `Store in Vector Database`
- **Failure modes / edge cases:**
  - Credential quota exhausted / billing issues
  - Model default mismatch with Pinecone index dimensions (must match your index)
  - Rate limits for large ingestions
- **Version notes:** typeVersion 1.2.

#### Node: Store in Vector Database
- **Type / role:** Pinecone Vector Store (`@n8n/n8n-nodes-langchain.vectorStorePinecone`) in **insert** mode.
- **Configuration choices:**
  - **Mode:** `insert`
  - **Namespace:** `<your-pinecone-namespace-name>` (must be replaced; typically site identifier)
  - **Index:** selected from list (currently empty in template and must be set)
- **Inputs / outputs:**
  - Main input ← `Split Text into Chunks`
  - ai_document input ← `Document Loader`
  - ai_embedding input ← `Embeddings OpenAI`
  - Main output → `Process URLs One by One` (to continue loop)
- **Failure modes / edge cases:**
  - Pinecone auth errors, wrong environment/project settings
  - Index not created or wrong dimension (must match embedding model output size)
  - Namespace mismatch between ingestion and retrieval (chat will “find nothing”)
  - Upsert payload too large if documents are oversized
- **Version notes:** typeVersion 1.3.

---

### Block 6 — Chat Webhook + Agent Retrieval (Serving)

**Overview:**  
Exposes a public chat webhook, runs a constrained support agent that must use Pinecone retrieval for business questions, and uses buffer memory for limited personal details.

**Nodes involved:**
- When chat message received
- Customer Support Agent
- OpenAI Chat Model
- Pinecone Vector Storage
- OpenAI Embeddings (Retrieval)
- Chat Memory

#### Node: When chat message received
- **Type / role:** LangChain Chat Trigger (`@n8n/n8n-nodes-langchain.chatTrigger`) entry point via webhook.
- **Configuration choices:**
  - **Mode:** webhook
  - **Public:** true
  - **Allowed origins:** `*` (CORS wide open)
  - **Available in chat:** true
- **Inputs / outputs:**
  - Output → `Customer Support Agent` (main)
- **Failure modes / edge cases:**
  - Public endpoint can be abused (rate limiting / auth not implemented)
  - CORS `*` allows any origin; consider restricting in production
- **Version notes:** typeVersion 1.1.

#### Node: Customer Support Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) orchestrates tool use + final response.
- **Configuration choices:**
  - **Max iterations:** 10
  - **System message:** highly constrained policy:
    - All business info must come from tools (Pinecone).
    - Greeting special-case response.
    - Failure response must be exact fixed sentence.
    - No mention of tools, no reasoning exposition, no “as an AI”.
    - Links must be Markdown and come only from tool output.
  - **Return intermediate steps:** false (user only sees final).
- **Inputs / outputs:**
  - Main input ← `When chat message received`
  - ai_languageModel ← `OpenAI Chat Model`
  - ai_tool ← `Pinecone Vector Storage`
  - ai_memory ← `Chat Memory`
- **Failure modes / edge cases:**
  - If Pinecone tool fails/returns empty, system prompt enforces the fixed fallback response.
  - If the agent cannot comply due to tool misconfiguration, it may still attempt an answer; the system prompt tries to prevent it, but correct tool wiring is critical.
- **Version notes:** typeVersion 3.

#### Node: OpenAI Chat Model
- **Type / role:** Chat LLM (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) for the agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Credentials: “OpenAi account”
- **Inputs / outputs:**
  - Output (ai_languageModel) → `Customer Support Agent`
- **Failure modes / edge cases:**
  - Model access not enabled for the account
  - Rate limits/timeouts
- **Version notes:** typeVersion 1.3.

#### Node: Pinecone Vector Storage (Tool)
- **Type / role:** Pinecone Vector Store in **retrieve-as-tool** mode (tool callable by agent).
- **Configuration choices:**
  - **Mode:** `retrieve-as-tool`
  - **topK:** 10
  - **Namespace:** `<your-pinecone-namespace-name>` (must match ingestion)
  - **Index:** must be selected
  - **Tool description:** instructs when to use it (mentions “Blushu” explicitly; should be customized to your company/brand).
- **Inputs / outputs:**
  - ai_embedding ← `OpenAI Embeddings` (retrieval embeddings)
  - ai_tool output → `Customer Support Agent`
- **Failure modes / edge cases:**
  - Namespace mismatch → empty results → forced fallback response
  - Index dimension mismatch or wrong index
  - Tool description brand mismatch (“Blushu”) may mislead agent behavior
- **Version notes:** typeVersion 1.3.

#### Node: OpenAI Embeddings (Retrieval)
- **Type / role:** Embeddings model used at query time to embed user questions for Pinecone similarity search.
- **Configuration choices:** uses “OpenAi account” credentials; default embedding model depends on node defaults.
- **Inputs / outputs:**
  - Output (ai_embedding) → `Pinecone Vector Storage`
- **Failure modes / edge cases:**
  - Must match the embedding model used for ingestion (or at least same dimension/space); otherwise retrieval quality breaks or errors.
- **Version notes:** typeVersion 1.2.

#### Node: Chat Memory
- **Type / role:** Memory buffer (`@n8n/n8n-nodes-langchain.memoryBufferWindow`) for conversation context.
- **Configuration choices:**
  - Context window length: 10 (turns/messages depending on node behavior)
- **Inputs / outputs:**
  - Output (ai_memory) → `Customer Support Agent`
- **Failure modes / edge cases:**
  - System prompt restricts allowed memory usage, but the node itself will still store context; governance relies on prompt compliance.
- **Version notes:** typeVersion 1.3.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual start for ingestion | — | Map Website URLs | ## Turn any website into an AI‑powered customer‑support chatbot (Open AI + Pinecone)<br><br>### How it works<br>1️⃣ **Discover URLs** – Firecrawl reads the sitemap (including sub‑domains) and returns every page URL.<br>2️⃣ **Clean & de‑duplicate** – JavaScript removes unwanted paths (press‑blog, newsletter, assets, .xml), trailing slashes and duplicate links.<br>3️⃣ **Controlled crawling** – the remaining URLs are processed one‑by‑one, fetching the raw HTML.<br>4️⃣ **HTML → plain text** – script/style tags are stripped and the rest of the markup is turned into readable text.<br>5️⃣ **Validate & chunk** – pages shorter than 50 chars are ignored; the rest is split into 1 000‑char chunks with a 200‑char overlap.<br>6️⃣ **Embed & store** – each chunk is turned into an OpenAI embedding and up‑serted into a Pinecone index (namespace = your‑site‑name).<br>7️⃣ **Chat endpoint** – a LangChain agent receives a user message, decides (via the system prompt) whether to call the Pinecone retrieval tool, and replies **only** with information that actually exists in the vector store.<br><br>### Setup steps<br>1️⃣ Add your **Firecrawl** API key (Credentials → Firecrawl).<br>2️⃣ Add your **OpenAI** API key (Credentials → OpenAI).<br>3️⃣ Create a **Pinecone** index (1536 dimensions for `text‑embedding‑ada‑002`).<br>4️⃣ Put the index name and a namespace into the “Store in Vector Database” node.<br>5️⃣ Replace `<your-website-url>` in the *Map Website URLs* node.<br>6️⃣ Execute the workflow – it will fill the vector store.<br>7️⃣ Deploy the **Chat Trigger** webhook (copy the URL) and embed it in your front‑end.<br><br>When the index is populated the bot will answer any product, policy or FAQ question without hallucinations. |
| Map Website URLs | Firecrawl | Discover site URLs from sitemap | When clicking ‘Execute workflow’ | Filter and Normalize URLs | URL Discovery – Firecrawl reads the sitemap (including sub‑domains) and returns every page URL. |
| Filter and Normalize URLs | Code | Filter/normalize/dedupe URL list | Map Website URLs | Process URLs One by One | ## URL Cleaning & Batching<br>JavaScript removes press‑blog, newsletter, assets‑vault, .xml files, trailing slashes and duplicate URLs.<br><br>Then SplitInBatches processes the list one‑by‑one. |
| Process URLs One by One | Split In Batches | Sequential URL processing loop | Filter and Normalize URLs; Store in Vector Database | Fetch Page Content | ## URL Cleaning & Batching<br>JavaScript removes press‑blog, newsletter, assets‑vault, .xml files, trailing slashes and duplicate URLs.<br><br>Then SplitInBatches processes the list one‑by‑one. |
| Fetch Page Content | HTTP Request | Fetch raw HTML for a URL | Process URLs One by One | Extract HTML and URL | Page Fetch & Text Extraction – HTTP request gets the HTML, a Set node normalises fields, and a small JS step strips script/style tags and converts the markup to plain readable text. |
| Extract HTML and URL | Set | Normalize `html` and `url` fields | Fetch Page Content | Clean HTML to Text | Page Fetch & Text Extraction – HTTP request gets the HTML, a Set node normalises fields, and a small JS step strips script/style tags and converts the markup to plain readable text. |
| Clean HTML to Text | Code | Strip HTML to plain text | Extract HTML and URL | Check Text Length | Page Fetch & Text Extraction – HTTP request gets the HTML, a Set node normalises fields, and a small JS step strips script/style tags and converts the markup to plain readable text. |
| Check Text Length | If | Skip short/empty pages | Clean HTML to Text | Split Text into Chunks | ## Chunking & Embedding<br><br>If the cleaned text > 50 chars it is split into 1 000‑char chunks (200‑char overlap). Each chunk is sent to OpenAI embeddings and up‑serted into Pinecone. |
| Split Text into Chunks | Code | Chunk text with overlap | Check Text Length | Store in Vector Database | ## Chunking & Embedding<br><br>If the cleaned text > 50 chars it is split into 1 000‑char chunks (200‑char overlap). Each chunk is sent to OpenAI embeddings and up‑serted into Pinecone. |
| Document Loader | Default Data Loader (LangChain) | Build documents + metadata for vector store | (implicit from chunk items) | Store in Vector Database (ai_document) | ## Chunking & Embedding<br><br>If the cleaned text > 50 chars it is split into 1 000‑char chunks (200‑char overlap). Each chunk is sent to OpenAI embeddings and up‑serted into Pinecone. |
| Embeddings OpenAI | OpenAI Embeddings (LangChain) | Create embeddings for ingestion | (implicit from chunk items) | Store in Vector Database (ai_embedding) | ## Chunking & Embedding<br><br>If the cleaned text > 50 chars it is split into 1 000‑char chunks (200‑char overlap). Each chunk is sent to OpenAI embeddings and up‑serted into Pinecone. |
| Store in Vector Database | Pinecone Vector Store (LangChain) | Upsert chunk embeddings into Pinecone | Split Text into Chunks; Document Loader (ai_document); Embeddings OpenAI (ai_embedding) | Process URLs One by One | ## Chunking & Embedding<br><br>If the cleaned text > 50 chars it is split into 1 000‑char chunks (200‑char overlap). Each chunk is sent to OpenAI embeddings and up‑serted into Pinecone. |
| When chat message received | Chat Trigger (LangChain) | Public chat webhook entrypoint | — | Customer Support Agent |  |
| Customer Support Agent | Agent (LangChain) | Orchestrate tool retrieval + response policy | When chat message received; OpenAI Chat Model (ai_languageModel); Pinecone Vector Storage (ai_tool); Chat Memory (ai_memory) | (final chat response) |  |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM powering the agent | — | Customer Support Agent (ai_languageModel) |  |
| Pinecone Vector Storage | Pinecone Vector Store (LangChain) | Retrieval tool callable by agent | OpenAI Embeddings (ai_embedding) | Customer Support Agent (ai_tool) |  |
| OpenAI Embeddings | OpenAI Embeddings (LangChain) | Embed user queries for retrieval | — | Pinecone Vector Storage (ai_embedding) |  |
| Chat Memory | Memory Buffer Window (LangChain) | Retain recent conversation context | — | Customer Support Agent (ai_memory) |  |
| Sticky Note | Sticky Note | Documentation only | — | — | (same content as shown in node) |
| Sticky Note1 | Sticky Note | Documentation only | — | — | (same content as shown in node) |
| Sticky Note2 | Sticky Note | Documentation only | — | — | (same content as shown in node) |
| Sticky Note3 | Sticky Note | Documentation only | — | — | (same content as shown in node) |
| Sticky Note4 | Sticky Note | Documentation only | — | — | (same content as shown in node) |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites (accounts/credentials)
1. **Firecrawl API key** (n8n Credentials → Firecrawl).
2. **OpenAI API key** (n8n Credentials → OpenAI).
3. **Pinecone project + index**:
   - Create an index and note its name.
   - Dimension must match your chosen embedding model.
   - Decide on a **namespace** (e.g., `mycompany-site`).

---

### A) Build the ingestion (crawl → chunk → embed → upsert)

1. **Create node:** *Manual Trigger*  
   - Name: `When clicking ‘Execute workflow’`

2. **Create node:** *Firecrawl* (`Map` operation)  
   - Name: `Map Website URLs`  
   - URL: set to your site root, e.g. `https://example.com`  
   - Operation: `map`  
   - Sitemap: `only`  
   - Include subdomains: enabled (optional)  
   - Credentials: select your Firecrawl credential  
   - Connect: Manual Trigger → Map Website URLs

3. **Create node:** *Code*  
   - Name: `Filter and Normalize URLs`  
   - Paste logic to:
     - Read `$json.links`
     - Remove blocked paths/keywords
     - Remove `.xml`
     - Normalize trailing slash
     - De-duplicate
     - Output items as `{ url }`  
   - Connect: Map Website URLs → Filter and Normalize URLs

4. **Create node:** *Split In Batches*  
   - Name: `Process URLs One by One`  
   - Leave defaults (or set batch size = 1 explicitly if your n8n version exposes it)  
   - Connect: Filter and Normalize URLs → Process URLs One by One

5. **Create node:** *HTTP Request*  
   - Name: `Fetch Page Content`  
   - URL: `{{ $json.url }}`  
   - Options: enable **Follow Redirects**  
   - Enable **Retry on fail**  
   - Enable **Continue on fail**  
   - Connect: Process URLs One by One → Fetch Page Content

6. **Create node:** *Set*  
   - Name: `Extract HTML and URL`  
   - Add fields:
     - `html` (String): `{{ $json.data ?? $json.html ?? "" }}`
     - `url` (String): `{{ $json.url ?? $('Process URLs One by One').item.json.url }}`
   - Connect: Fetch Page Content → Extract HTML and URL

7. **Create node:** *Code*  
   - Name: `Clean HTML to Text`  
   - Implement basic HTML stripping (remove script/style, remove tags, normalize whitespace)  
   - Output: `{ url: $json.url, text, project_id: $json.project_id }` (project_id optional)  
   - Connect: Extract HTML and URL → Clean HTML to Text

8. **Create node:** *If*  
   - Name: `Check Text Length`  
   - Condition: `{{ $json.text.length > 50 }}` is true  
   - Connect: Clean HTML to Text → Check Text Length (main)

9. **Create node:** *Code*  
   - Name: `Split Text into Chunks`  
   - Chunk settings:
     - chunk size 1000 characters
     - overlap 200 characters
     - output items: `{ content, url, project_id }`
   - Connect: Check Text Length (true) → Split Text into Chunks

10. **Create node:** *OpenAI Embeddings (LangChain)*  
    - Name: `Embeddings OpenAI`  
    - Credentials: choose OpenAI key for ingestion  
    - (Optional but recommended) explicitly select an embedding model and ensure Pinecone dimensions match.

11. **Create node:** *Default Data Loader (LangChain)*  
    - Name: `Document Loader`  
    - Metadata fields:
      - `url = {{ $json.url }}`
      - `content = {{ $json.content }}`

12. **Create node:** *Pinecone Vector Store (LangChain)*  
    - Name: `Store in Vector Database`  
    - Mode: `insert`  
    - Index: select your Pinecone index  
    - Namespace: set your namespace string (must later match retrieval tool)  

13. **Wire AI connections into the Pinecone insert node**
    - Connect `Embeddings OpenAI` → `Store in Vector Database` using **ai_embedding**.
    - Connect `Document Loader` → `Store in Vector Database` using **ai_document**.
    - Connect `Split Text into Chunks` → `Store in Vector Database` using **main**.

14. **Close the batching loop**
    - Connect `Store in Vector Database` (main output) → `Process URLs One by One` (main input)  
    This makes the workflow continue with the next URL after each upsert.

15. **Test ingestion**
    - Replace `<your-website-url>` and set Pinecone index + namespace.
    - Execute the workflow manually until it completes and the Pinecone namespace contains vectors.

---

### B) Build the chat endpoint (webhook → agent → Pinecone retrieval tool)

16. **Create node:** *Chat Trigger (LangChain)*  
    - Name: `When chat message received`  
    - Mode: `webhook`  
    - Public: enabled  
    - Allowed origins: `*` (or restrict)  
    - Available in chat: enabled

17. **Create node:** *Memory Buffer Window (LangChain)*  
    - Name: `Chat Memory`  
    - Context window length: `10`

18. **Create node:** *OpenAI Chat Model (LangChain)*  
    - Name: `OpenAI Chat Model`  
    - Model: `gpt-4.1-mini` (or your choice)  
    - Credentials: OpenAI key for chat

19. **Create node:** *OpenAI Embeddings (LangChain)* (for retrieval)  
    - Name: `OpenAI Embeddings`  
    - Credentials: OpenAI key  
    - Ensure it matches ingestion embedding dimensionality/model family.

20. **Create node:** *Pinecone Vector Store (LangChain)* (tool mode)  
    - Name: `Pinecone Vector Storage`  
    - Mode: `retrieve-as-tool`  
    - Index: same Pinecone index as ingestion  
    - Namespace: same namespace as ingestion  
    - topK: `10`  
    - Tool description: customize to your company name (replace “Blushu” if needed).

21. **Create node:** *Agent (LangChain)*  
    - Name: `Customer Support Agent`  
    - Max iterations: `10`  
    - System message: paste your policy (greeting rule, forced tool usage, fallback sentence, markdown link rule, etc.)  
    - Return intermediate steps: off

22. **Wire chat flow**
    - Connect `When chat message received` → `Customer Support Agent` (main)
    - Connect `OpenAI Chat Model` → `Customer Support Agent` via **ai_languageModel**
    - Connect `Chat Memory` → `Customer Support Agent` via **ai_memory**
    - Connect `OpenAI Embeddings` → `Pinecone Vector Storage` via **ai_embedding**
    - Connect `Pinecone Vector Storage` → `Customer Support Agent` via **ai_tool**

23. **Deploy and embed**
    - Activate workflow.
    - Copy the Chat Trigger webhook URL and integrate it into your frontend chat widget/app.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Pinecone index sized for your embedding model (template note mentions 1536 dims for `text-embedding-ada-002`). | Pinecone setup requirement; ensure ingestion and retrieval embeddings match the index dimension. |
| Replace placeholders: `<your-website-url>` and `<your-pinecone-namespace-name>` in both ingestion and retrieval nodes. | Prevents “empty results” retrieval and ensures correct ingestion target. |
| The HTML-to-text conversion is intentionally “very simple” and may need upgrading (readability, boilerplate removal). | Improves retrieval quality and reduces irrelevant chunks. |
| The Chat Trigger is public and allows all origins (`*`). Consider restricting origins and adding abuse protections. | Production hardening for the webhook endpoint. |
| The retrieval tool description mentions “Blushu”; update to your actual brand/company to avoid agent confusion. | Tool instruction alignment with your business. |