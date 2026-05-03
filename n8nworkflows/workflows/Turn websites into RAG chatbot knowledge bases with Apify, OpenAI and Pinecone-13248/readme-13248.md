Turn websites into RAG chatbot knowledge bases with Apify, OpenAI and Pinecone

https://n8nworkflows.xyz/workflows/turn-websites-into-rag-chatbot-knowledge-bases-with-apify--openai-and-pinecone-13248


# Turn websites into RAG chatbot knowledge bases with Apify, OpenAI and Pinecone

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Turn websites into RAG chatbot knowledge bases with Apify, OpenAI and Pinecone

**Purpose:**  
This workflow crawls a website, extracts and chunks its content into structured text segments, generates OpenAI embeddings for each chunk, and stores them in a Pinecone vector index. The result is a **RAG-ready knowledge base** suitable for semantic search and chatbot retrieval with citations/metadata.

**Target use cases**
- Turning documentation/marketing sites into searchable vector knowledge bases
- Preparing website content for RAG chatbots (semantic retrieval by similarity)
- Creating embeddings with rich metadata (URL, title, section, token counts, etc.) for better attribution and filtering

### Logical Blocks
**1.1 Workflow Entry & Website Capture**  
Manual execution triggers a web crawl using the AI Training Scraper (community node), producing structured “chunks”.

**1.2 Chunk Normalization & Metadata Flattening**  
Scraper output is transformed into a flat list of chunk items with consistent fields, including detailed metadata.

**1.3 Batch Processing & Embedding Generation**  
Chunks are processed in batches and sent to OpenAI Embeddings.

**1.4 Embedding/Data Merge & Vector Storage**  
Embeddings are merged back with metadata and inserted into Pinecone as vectors.

---

## 2. Block-by-Block Analysis

### 2.1 Workflow Entry & Website Capture

**Overview:**  
Starts the workflow manually and crawls the target website to extract clean, chunked content suitable for embedding.

**Nodes Involved**
- When clicking ‘Execute workflow’
- AI Training Scraper

#### Node: When clicking ‘Execute workflow’
- **Type / Role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration:** No parameters; user runs workflow manually.
- **Inputs / Outputs:**
  - **Output →** `AI Training Scraper`
- **Edge cases / failures:**
  - None (only triggers execution).

#### Node: AI Training Scraper
- **Type / Role:** AI Training Data Scraper community node (`n8n-nodes-ai-training-scraper.aiTrainingScraper`) — crawls pages and returns structured chunks.
- **Configuration choices (interpreted):**
  - `startUrls`: `https://docs.n8n.io/workflows/`
  - `additionalOptions`: empty (defaults used)
- **Key data produced (expected shape, inferred from downstream code):**
  - `chunks[]` (each chunk has `id`, `text`, and `metadata.*`)
  - `crawl_info.*` (e.g., `crawled_at`, `crawl_depth`)
- **Inputs / Outputs:**
  - **Input:** from Manual Trigger
  - **Output →** `Process Chunks`
- **Version-specific requirements:**
  - Requires installing the community package: `n8n-nodes-ai-training-scraper` (per sticky note).
- **Edge cases / failures:**
  - Missing/invalid Apify credentials (auth failure)
  - Crawl blocked by robots/WAF, rate limits, timeouts
  - Unexpected output format (missing `chunks`, missing `crawl_info`)
  - Some pages may not yield `keywords`, `section_title`, etc., which can break downstream code if not guarded (see “Process Chunks” notes).

**Sticky note context (applies to this block):**
- “Website Capture … uses the AI Training Data Scraper … Create an Apify account, get an API token … paste it into the Apify credential section in n8n.”

---

### 2.2 Chunk Normalization & Metadata Flattening

**Overview:**  
Converts scraper output into a flat item list where each item represents a single chunk with consistent, Pinecone-ready fields.

**Nodes Involved**
- Process Chunks

#### Node: Process Chunks
- **Type / Role:** Code node (`n8n-nodes-base.code`) — transforms/normalizes data.
- **Configuration choices (interpreted):**
  - Iterates over all incoming items (supports single/batch output).
  - Reads `data.chunks` and emits one n8n item per chunk with fields:
    - **Vector fields:** `id`, `text`
    - **Page metadata:** `pageUrl`, `pageTitle`, `websiteDomain`
    - **Chunk metadata:** `chunkIndex`, `tokenCount`, `startPosition`, `endPosition`
    - **Classification:** `sectionTitle`, `headingLevel`, `contentType`, `hasCode`, `language`, `keywords`
    - **Crawl info:** `crawledAt`, `crawlDepth`
- **Key expressions / variables:**
  - `new URL(chunk.metadata.source_url).hostname` to derive `websiteDomain`
  - `chunk.metadata.section_title || 'No Section'`
  - `chunk.metadata.keywords.join(', ')`
- **Inputs / Outputs:**
  - **Input:** `AI Training Scraper`
  - **Output →** `Split In Batches`
- **Edge cases / failures:**
  - `chunk.metadata.source_url` missing or invalid URL → `new URL()` throws.
  - `chunk.metadata.keywords` missing/null/not an array → `.join()` throws.
  - `data.crawl_info` missing → access errors.
  - `data.chunks` missing is handled (`|| []`), but metadata assumptions are not fully guarded.

---

### 2.3 Batch Processing & Embedding Generation

**Overview:**  
Splits chunk items into manageable batches and generates embeddings using OpenAI for each chunk.

**Nodes Involved**
- Split In Batches
- OpenAI - Create Embeddings

#### Node: Split In Batches
- **Type / Role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — throttles/controls batch size.
- **Configuration choices (interpreted):**
  - `batchSize: 50`
  - Default options otherwise.
- **Inputs / Outputs:**
  - **Input:** `Process Chunks`
  - **Main output (batch) →** `OpenAI - Create Embeddings`
  - **Second output:** unused (would normally be used to loop/continue batches in classic patterns)
- **Edge cases / failures:**
  - If downstream does not loop batches, only the first batch might be processed in some designs.  
    (In this workflow, the second output is not connected; depending on n8n behavior and node configuration, you may need a loopback to process all batches reliably.)

#### Node: OpenAI - Create Embeddings
- **Type / Role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — creates embeddings.
- **Configuration choices (interpreted):**
  - `resource: embedding`
  - Model not specified in JSON (will use node defaults or credential defaults).
- **Inputs / Outputs:**
  - **Input:** from `Split In Batches`
  - **Output →** `Combine Embedding with Data`
- **Credential requirements:**
  - OpenAI API key configured in n8n credentials.
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials
  - Model/endpoint mismatch, quota exceeded, rate limiting (429)
  - Token limits if the node embeds unexpectedly large text fields
  - Output shape differences: downstream assumes `item.json.data[0].embedding`

**Sticky note context (applies to this block):**
- “Embedding & Vector Storage … Create an OpenAI API key … Create a Pinecone account and index … generate an API key … add it as a Pinecone node.”

---

### 2.4 Embedding/Data Merge & Vector Storage

**Overview:**  
Merges embeddings back into each chunk item and inserts (upserts) vectors with metadata into Pinecone.

**Nodes Involved**
- Combine Embedding with Data
- Pinecone Vector Store

#### Node: Combine Embedding with Data
- **Type / Role:** Code node (`n8n-nodes-base.code`) — merges embedding output into the original item structure.
- **Configuration choices (interpreted):**
  - For each incoming item, returns:
    - all existing JSON fields
    - plus `embedding: item.json.data[0].embedding`
- **Inputs / Outputs:**
  - **Input:** `OpenAI - Create Embeddings`
  - **Output →** `Pinecone Vector Store`
- **Edge cases / failures:**
  - If OpenAI node returns a different schema (no `data`, empty array), `data[0]` access fails.
  - If embeddings are nested differently (provider/model changes), this merge must be adapted.

#### Node: Pinecone Vector Store
- **Type / Role:** Pinecone Vector Store (LangChain) node (`@n8n/n8n-nodes-langchain.vectorStorePinecone`) — inserts vectors into Pinecone.
- **Configuration choices (interpreted):**
  - `mode: insert` (insert/upsert behavior depending on node implementation)
  - `pineconeIndex`: set to “list” mode but **value is blank** in the JSON export (must be selected manually).
  - `options`: empty (defaults)
- **Inputs / Outputs:**
  - **Input:** `Combine Embedding with Data`
  - **Output:** none connected (terminal node)
- **Credential requirements:**
  - Pinecone API credential: **“PineconeApi account”**
- **Edge cases / failures:**
  - Index not selected / missing index name
  - Dimension mismatch between Pinecone index dimension and embedding vector dimension
  - Namespace/metadata constraints (if Pinecone node expects specific fields)
  - Rate limits, request size limits, network timeouts
  - Duplicate IDs: if `id` repeats, behavior depends on insert/upsert semantics

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | AI Training Scraper | ## Website Capture  \n- This block then uses the AI Training Data Scraper to crawl pages and extract structured content.  \n### User Action Required:  \n- Create an Apify account, get an API token from console.apify.com, and paste it into the Apify credential section in n8n. |
| AI Training Scraper | AI Training Scraper (community) | Crawl site + extract structured chunks | When clicking ‘Execute workflow’ | Process Chunks | ## Website Capture  \n- This block then uses the AI Training Data Scraper to crawl pages and extract structured content.  \n### User Action Required:  \n- Create an Apify account, get an API token from console.apify.com, and paste it into the Apify credential section in n8n. |
| Process Chunks | Code | Flatten chunks; normalize metadata | AI Training Scraper | Split In Batches | ## Website Capture  \n- This block then uses the AI Training Data Scraper to crawl pages and extract structured content.  \n### User Action Required:  \n- Create an Apify account, get an API token from console.apify.com, and paste it into the Apify credential section in n8n. |
| Split In Batches | Split In Batches | Batch control (50 items) | Process Chunks | OpenAI - Create Embeddings | ## Embedding & Vector Storage  \n- Generates semantic embeddings for each chunk and merges them with rich metadata, preserving full context for retrieval and citation.  \n- Upserts all embeddings into the vector store, grouped by website domain, making the site searchable via semantic similarity.  \n### User Action Required:  \n- Create an OpenAI API key from platform.openai.com/account/api-keys and add it in the credential secton.  \n- Create a Pinecone account and index at pinecone.io, generate an API key, and add it as a Pinecone node. |
| OpenAI - Create Embeddings | OpenAI (LangChain) | Generate embeddings for chunk text | Split In Batches | Combine Embedding with Data | ## Embedding & Vector Storage  \n- Generates semantic embeddings for each chunk and merges them with rich metadata, preserving full context for retrieval and citation.  \n- Upserts all embeddings into the vector store, grouped by website domain, making the site searchable via semantic similarity.  \n### User Action Required:  \n- Create an OpenAI API key from platform.openai.com/account/api-keys and add it in the credential secton.  \n- Create a Pinecone account and index at pinecone.io, generate an API key, and add it as a Pinecone node. |
| Combine Embedding with Data | Code | Merge embedding vector into each item | OpenAI - Create Embeddings | Pinecone Vector Store | ## Embedding & Vector Storage  \n- Generates semantic embeddings for each chunk and merges them with rich metadata, preserving full context for retrieval and citation.  \n- Upserts all embeddings into the vector store, grouped by website domain, making the site searchable via semantic similarity.  \n### User Action Required:  \n- Create an OpenAI API key from platform.openai.com/account/api-keys and add it in the credential secton.  \n- Create a Pinecone account and index at pinecone.io, generate an API key, and add it as a Pinecone node. |
| Pinecone Vector Store | Pinecone Vector Store (LangChain) | Insert vectors + metadata into Pinecone | Combine Embedding with Data | — | ## Embedding & Vector Storage  \n- Generates semantic embeddings for each chunk and merges them with rich metadata, preserving full context for retrieval and citation.  \n- Upserts all embeddings into the vector store, grouped by website domain, making the site searchable via semantic similarity.  \n### User Action Required:  \n- Create an OpenAI API key from platform.openai.com/account/api-keys and add it in the credential secton.  \n- Create a Pinecone account and index at pinecone.io, generate an API key, and add it as a Pinecone node. |
| Sticky Note | Sticky Note | Documentation/comment | — | — | (node is itself a note; content already represented where applied) |
| Sticky Note1 | Sticky Note | Documentation/comment | — | — | (node is itself a note; content already represented where applied) |
| Sticky Note2 | Sticky Note | Documentation/comment | — | — | (node is itself a note; content already represented where applied) |

---

## 4. Reproducing the Workflow from Scratch

1) **Install required community node**
   - In n8n: **Settings → Community Nodes**
   - Install package: `n8n-nodes-ai-training-scraper`
   - Restart n8n if required.

2) **Create credentials**
   - **Apify credential** (used by the scraper node)
     - Create Apify account, generate API token at: `console.apify.com`
     - Add token to the scraper’s credential section in n8n.
   - **OpenAI credential**
     - Create API key: `https://platform.openai.com/account/api-keys`
     - Add OpenAI credential in n8n (API key).
   - **Pinecone credential**
     - Create Pinecone account and an index at `pinecone.io`
     - Ensure the **index dimension matches** the embedding model dimension you will use.
     - Generate Pinecone API key and add as Pinecone credentials in n8n.

3) **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

4) **Add node: AI Training Scraper**
   - Node type: **AI Training Scraper** (from `n8n-nodes-ai-training-scraper`)
   - Name: `AI Training Scraper`
   - Set **Start URLs** to: `https://docs.n8n.io/workflows/` (or your target site)
   - Configure Apify credentials in this node.

5) **Connect:** Manual Trigger → AI Training Scraper

6) **Add node: Code**
   - Node type: **Code**
   - Name: `Process Chunks`
   - Paste logic that:
     - Iterates through incoming items
     - For each `chunk` in `data.chunks`, outputs a new item containing:
       - `id`, `text`
       - metadata fields (URL, title, domain, indices, token counts, etc.)
       - `crawledAt` / `crawlDepth`
   - Connect: AI Training Scraper → Process Chunks

7) **Add node: Split In Batches**
   - Node type: **Split In Batches**
   - Name: `Split In Batches`
   - Set **Batch Size**: `50`
   - Connect: Process Chunks → Split In Batches

8) **Add node: OpenAI Embeddings**
   - Node type: **OpenAI (LangChain)** (resource: Embedding)
   - Name: `OpenAI - Create Embeddings`
   - Select OpenAI credentials
   - Ensure the node embeds the **chunk text field** (depending on node UI, map it to `text`)
   - Connect: Split In Batches → OpenAI - Create Embeddings

9) **Add node: Code**
   - Node type: **Code**
   - Name: `Combine Embedding with Data`
   - Configure to merge the returned embedding into each item, adding:
     - `embedding` = the embedding vector from the OpenAI response (commonly `data[0].embedding`)
   - Connect: OpenAI - Create Embeddings → Combine Embedding with Data

10) **Add node: Pinecone Vector Store**
   - Node type: **Pinecone Vector Store (LangChain)**
   - Name: `Pinecone Vector Store`
   - Set **Mode**: `Insert`
   - Select Pinecone credentials
   - Select your **Pinecone Index** (required; it is blank in the provided export)
   - Map fields as required by the node UI:
     - Vector: `embedding`
     - Text/document: `text`
     - Metadata: the remaining fields (URL, title, domain, chunkIndex, etc.)
   - Connect: Combine Embedding with Data → Pinecone Vector Store

11) **(Recommended) Ensure batching processes all items**
   - If your n8n version requires explicit looping:
     - Connect the **second output** of Pinecone insert (or a “NoOp”) back into **Split In Batches** “Continue” pattern, or use a standard SplitInBatches loop design.
   - Validate by counting inserted vectors vs. chunk count.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow turns any website into a RAG‑ready knowledge base for AI chatbots. It crawls the site, cleans and chunks content, generates embeddings, and stores them in a vector database for fast, semantic retrieval.” | Workflow purpose (Sticky Note: “Website RAGFlow”) |
| “To enable it, go to Settings → Community Nodes in n8n and add the package name n8n-nodes-ai-training-scraper.” | Community node requirement |
| “Create an Apify account, get an API token from console.apify.com, and paste it into the Apify credential section in n8n.” | Apify setup (`console.apify.com`) |
| “Create an OpenAI API key from platform.openai.com/account/api-keys and add it in the credential secton.” | OpenAI setup (`https://platform.openai.com/account/api-keys`) |
| “Create a Pinecone account and index at pinecone.io, generate an API key, and add it as a Pinecone node.” | Pinecone setup (`pinecone.io`) |
| “Created by Blukaze Automations \| blukaze.com” | Credit / branding (`blukaze.com`) |