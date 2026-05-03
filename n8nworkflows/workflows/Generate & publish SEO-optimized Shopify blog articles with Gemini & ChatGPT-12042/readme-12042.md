Generate & publish SEO-optimized Shopify blog articles with Gemini & ChatGPT

https://n8nworkflows.xyz/workflows/generate---publish-seo-optimized-shopify-blog-articles-with-gemini---chatgpt-12042


# Generate & publish SEO-optimized Shopify blog articles with Gemini & ChatGPT

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Shopify Blog Automation  
**User-facing title:** Generate & publish SEO-optimized Shopify blog articles with Gemini & ChatGPT  
**Primary purpose:** Automatically discover news/trends from RSS feeds, enrich them with AI and product context from Shopify, generate SEO-optimized blog content (and social snippets), run quality checks, and publish to Shopify.  
**Core storage layer:** MongoDB (documents + chat history) and MongoDB Atlas Vector Search (embeddings/RAG).

### 1.1 News ingestion & deduplication (daily)
- Pull RSS feed URLs from Google Sheets, read items, filter/limit, normalize fields, fetch article pages, hash/deduplicate, then store new content and embeddings (vector store).

### 1.2 Topic/brief creation from accumulated signals (RAG-assisted)
- Aggregate stored items and run an “Article Intelligence Agent” that extracts structured insights/angles to be used later.

### 1.3 Production run (scheduled): product + task definition + writing
- On schedule, fetch run configuration from Google Sheets, fetch Shopify products, select relevant products, build a task definition/brief, then write the article with tools (web search, link checking, image generation) and MongoDB chat memory.

### 1.4 QA and iterative improvement loop
- Quality Check Agent outputs a score/decision. If below threshold, loop content through a Content Modifier agent and re-store/re-check.

### 1.5 Metadata, publishing, and promotion assets
- Generate titles, meta data, categories, create Shopify Storefront token + publish blog post via HTTP (GraphQL/REST), generate a tweet, and persist outputs to MongoDB.

**Entry points (multiple):**
1. `Get Articles Daily` (Schedule Trigger) → RSS ingestion pipeline  
2. `Schedule Trigger` (Schedule Trigger) → main content production pipeline  

---

## 2. Block-by-Block Analysis

### Block 2.1 — RSS feed list retrieval & RSS item intake (Daily)
**Overview:** Loads RSS feed URLs from Google Sheets, then pulls RSS entries and prepares them for filtering and downstream processing.

**Nodes involved:**
- Get Articles Daily
- Get rss feed (Google Sheets)
- Split Out
- Read RSS News Feeds
- Filter
- Limit
- Set and Normalize Fields

**Node details:**

1) **Get Articles Daily**  
- **Type/role:** Schedule Trigger; daily kickoff for ingestion.  
- **Config (interpreted):** Runs on a schedule (not specified in JSON parameters—defaults must be set in UI).  
- **Output:** Triggers `Get rss feed`.  
- **Failure modes:** Misconfigured schedule/timezone; workflow inactive (workflow `active:false`).

2) **Get rss feed** (Google Sheets)  
- **Type/role:** Google Sheets node; fetches feed URL list.  
- **Config:** Operation not visible (parameters empty), but connection indicates it returns rows containing feed URLs.  
- **Output:** `Split Out`  
- **Failure modes:** OAuth/Service Account auth errors; sheet/range not set; empty result.

3) **Split Out**  
- **Type/role:** Split Out; explodes an array/list of sheet rows into individual items.  
- **Output:** `Read RSS News Feeds`  
- **Failure modes:** If sheet output isn’t an array field, Split Out yields 0 items.

4) **Read RSS News Feeds**  
- **Type/role:** RSS Feed Read; reads RSS URL per item.  
- **Output:** `Filter`  
- **Failure modes:** Invalid feed URL; timeouts; non-XML responses; rate limits.

5) **Filter**  
- **Type/role:** Filter; keeps only entries matching criteria (not shown).  
- **Output:** `Limit`  
- **Failure modes:** Mis-specified filter expression drops everything; missing fields referenced in conditions.

6) **Limit**  
- **Type/role:** Limit; caps number of RSS items per run.  
- **Output:** `Set and Normalize Fields`  
- **Failure modes:** Set too low (starves pipeline) or too high (later timeouts/cost).

7) **Set and Normalize Fields**  
- **Type/role:** Set; normalizes fields (title/url/date/source etc.).  
- **Output:** `Loop Over Items2`  
- **Failure modes:** Overwrites needed fields; invalid expressions.

---

### Block 2.2 — Article fetch, hashing, deduplication, and storage + embeddings
**Overview:** For each RSS item, fetch the article page, compute a hash, check MongoDB for duplicates, and insert only new documents; then embed and upsert into MongoDB Atlas Vector Store.

**Nodes involved:**
- Loop Over Items2
- HTTP Request4
- Code17
- Crypto
- Find documents
- If
- Code2
- Insert documents
- Default Data Loader
- Recursive Character Text Splitter
- openai-text-embedding-3-small
- MongoDB Atlas Vector Store

**Node details:**

1) **Loop Over Items2** (SplitInBatches)  
- **Type/role:** Batch loop for RSS items (controls throughput).  
- **Outputs:**  
  - Main batch output to `HTTP Request4` (process next item)  
  - “Done” branch to `Aggregate` (used for later steps)  
- **Failure modes:** Batch size mis-set; loop not completing if downstream errors block.

2) **HTTP Request4** *(onError: continueErrorOutput)*  
- **Type/role:** Fetches article HTML/content from the RSS entry URL.  
- **Config:** Not shown; error handling continues.  
- **Outputs:**  
  - Success → `Code17`  
  - Error output also returns to `Loop Over Items2` (keeps loop moving)  
- **Failure modes:** 403/blocked by bot protection; timeouts; redirected pages; paywalls; malformed HTML.

3) **Code17**  
- **Type/role:** Parses/cleans HTTP response (likely extracts main text).  
- **Output:** `Crypto`  
- **Failure modes:** Parser assumptions fail; missing response body field.

4) **Crypto**  
- **Type/role:** Generates a hash/signature (dedupe key) from URL/content.  
- **Output:** `Find documents`  
- **Failure modes:** Hashing wrong field → poor dedupe; empty input leads to identical hashes.

5) **Find documents** *(alwaysOutputData: true)*  
- **Type/role:** MongoDB query to find existing documents by hash/url.  
- **Output:** `If`  
- **Failure modes:** MongoDB credential/host issues; query filter wrong; alwaysOutputData may mask empty results vs error.

6) **If**  
- **Type/role:** Branch on “already exists?”  
- **Outputs:**  
  - True branch → `Loop Over Items2` (skip insert, continue)  
  - False branch → `Code2` (prepare new document)  
- **Failure modes:** Condition inverted; missing fields lead to wrong branch.

7) **Code2**  
- **Type/role:** Shapes the new document for insertion + embedding pipeline.  
- **Output:** `Insert documents`  
- **Failure modes:** Produces invalid MongoDB document; missing required fields for vector store.

8) **Insert documents** (MongoDB)  
- **Type/role:** Inserts article doc into MongoDB collection.  
- **Output:** `MongoDB Atlas Vector Store`  
- **Failure modes:** Duplicate key errors if unique index; schema constraints; connectivity.

9) **MongoDB Atlas Vector Store** (LangChain)  
- **Type/role:** Upserts documents into MongoDB Atlas Vector Search index using embeddings.  
- **Inputs:**  
  - `Insert documents` (main docs)  
  - `Default Data Loader` (ai_document)  
  - `openai-text-embedding-3-small` (ai_embedding)  
- **Output:** back to `Loop Over Items2` (continue loop)  
- **Failure modes:** Atlas Search index misconfigured; embedding dimension mismatch; OpenAI auth/rate limits.

10) **Default Data Loader** (LangChain)  
- **Type/role:** Converts incoming docs to LangChain “Document” objects.  
- **Input:** `Recursive Character Text Splitter` (ai_textSplitter)  
- **Output:** to vector store (ai_document)  
- **Failure modes:** Wrong field mapping (content field empty).

11) **Recursive Character Text Splitter**  
- **Type/role:** Splits long article text into chunks for embeddings/RAG.  
- **Output:** feeds `Default Data Loader`  
- **Failure modes:** Chunk size/overlap not set → too large chunks, token overrun; too small chunks → poor retrieval.

12) **openai-text-embedding-3-small**  
- **Type/role:** OpenAI embeddings model.  
- **Output:** to both vector stores (Atlas Vector Store & Atlas Vector Store1)  
- **Failure modes:** Missing OpenAI credentials; quota; model deprecation.

---

### Block 2.3 — Build “news intelligence” from stored chunks
**Overview:** Pulls stored content, aggregates it, and uses an AI agent to produce structured insights (angles, key points, facts), then updates the stored records.

**Nodes involved:**
- Loop Over Items
- delete news chunks
- delete news articles (disabled)
- If4
- Split Out1
- Find documents2
- Aggregate3
- Article Intelligence Agent
- Structured Output Parser1
- Code1
- Update documents
- Code (from earlier branch)
- Insert documents1

**Node details:**

1) **Loop Over Items** (SplitInBatches)  
- **Type/role:** Iterates over stored items requiring intelligence extraction.  
- **Outputs:**  
  - To `delete news chunks` (cleanup path)  
  - Loop continuation via `If4`  
- **Failure modes:** Loop logic can be confusing due to dual outputs; ensure correct “continue” wiring.

2) **delete news chunks** (MongoDB)  
- **Type/role:** Deletes chunk docs (likely previous embeddings/chunks) before reprocessing.  
- **Output:** `delete news articles` (disabled)  
- **Failure modes:** Deletes too broadly; missing filter; disabled downstream means it stops after delete unless parallel path continues.

3) **delete news articles** *(disabled:true)*  
- **Type/role:** Intended to delete article docs; currently disabled.  
- **Failure modes:** None at runtime (disabled). Risk if enabled without safe filter.

4) **If4**  
- **Type/role:** Controls whether to process intelligence now or move batching.  
- **Outputs:**  
  - Branch 0 → `Split Out1`  
  - Branch 1 → `Loop Over Items` (continue)  
- **Failure modes:** Wrong condition may cause infinite loop or skip processing.

5) **Split Out1**  
- **Type/role:** Explodes aggregated documents list into individual items for MongoDB retrieval.  
- **Output:** `Find documents2`

6) **Find documents2** (MongoDB)  
- **Type/role:** Loads documents to be summarized/understood.  
- **Output:** `Aggregate3`

7) **Aggregate3**  
- **Type/role:** Aggregates documents into a single payload for the agent (context bundle).  
- **Output:** `Article Intelligence Agent`

8) **Article Intelligence Agent** *(alwaysOutputData:true, retryOnFail:true)*  
- **Type/role:** LangChain Agent using `OpenAI Chat Model` + tools from `Think2` and structured parser.  
- **Model:** `OpenAI Chat Model` (connected via ai_languageModel).  
- **Parser:** `Structured Output Parser1` (ai_outputParser).  
- **Output:** `Code1`  
- **Failure modes:** Tool calls failing; token limits; structured output not matching schema; hallucinated fields.

9) **Structured Output Parser1**  
- **Type/role:** Enforces JSON schema for intelligence output.  
- **Failure modes:** Model output not valid JSON → parser error; needs aligned prompt/schema.

10) **Code1**  
- **Type/role:** Maps parsed intelligence into DB update structure.  
- **Output:** `Update documents`

11) **Update documents** *(alwaysOutputData:true)*  
- **Type/role:** MongoDB update to attach intelligence results to documents.  
- **Output:** `Loop Over Items` (continue batching)  
- **Failure modes:** Update filter mismatch; partial update overwrites fields.

12) **Code** → **Insert documents1**  
- **Type/role:** Separate insert path from earlier `If1` branch (see Block 2.4) that also feeds `Loop Over Items`.  
- **Risk:** Two different insert/update flows converge; ensure document identifiers don’t collide.

---

### Block 2.4 — Feed “row(s) in sheet” to an initial agent and store results
**Overview:** Takes aggregated RSS/DB outputs, calls an AI agent to produce structured items, then inserts them into MongoDB and triggers intelligence loop.

**Nodes involved:**
- Aggregate
- Get row(s) in sheet
- Aggregate4
- AI Agent12
- MongoDB Atlas Vector Store1
- Structured Output Parser7
- Split Out2
- If1
- Code
- Insert documents1

**Node details:**

1) **Aggregate**  
- **Type/role:** Aggregates data after RSS loop completion.  
- **Output:** `Get row(s) in sheet`

2) **Get row(s) in sheet** (Google Sheets)  
- **Type/role:** Reads rows (likely configuration/keywords).  
- **Output:** `Aggregate4`  
- **Failure modes:** Wrong range; empty rows.

3) **Aggregate4**  
- **Type/role:** Bundles rows for agent.  
- **Output:** `AI Agent12`

4) **AI Agent12** *(retryOnFail:true)*  
- **Type/role:** Agent to transform sheet+news context into structured units.  
- **Model:** `OpenAI Chat Model`  
- **Tool:** `MongoDB Atlas Vector Store1` (as ai_tool) for retrieval; also connected embeddings.  
- **Parser:** `Structured Output Parser7`  
- **Output:** `Split Out2`  
- **Failure modes:** Retrieval returns irrelevant chunks; schema mismatch; prompt/tool setup not shown (parameters empty).

5) **MongoDB Atlas Vector Store1**  
- **Type/role:** Retrieval tool over MongoDB Atlas Vector Search for RAG.  
- **Embedding:** `openai-text-embedding-3-small`  
- **Failure modes:** Index name mismatch; no documents in index.

6) **Split Out2** → **If1**  
- **Type/role:** Iterates each structured result and conditionally proceeds.  
- **If1 output:** only wired to `Code` (true branch); false branch unused (drops).  
- **Failure modes:** If1 condition too strict → no inserts.

7) **Code** → **Insert documents1** (MongoDB)  
- **Type/role:** Prepare & insert structured results into MongoDB.  
- **Output:** `Loop Over Items` (kicks intelligence processing)  
- **Failure modes:** Invalid document shapes; missing required keys.

---

### Block 2.5 — Main production run trigger and Shopify product selection
**Overview:** On schedule, loads run details from Sheets, fetches Shopify products, selects the most relevant products using an LLM agent, and stores the selection.

**Nodes involved:**
- Schedule Trigger
- Get details (Google Sheets)
- MongoDB9
- Get many products (Shopify)
- Code in JavaScript1
- product selector (Agent v3)
- Structured Output Parser9
- MongoDB12
- If5

**Node details:**

1) **Schedule Trigger**  
- **Type/role:** Scheduled kickoff for content production.  
- **Output:** `Get details`

2) **Get details** (Google Sheets)  
- **Type/role:** Loads config: brand, tone, language, blog settings, thresholds, etc. (implied).  
- **Output:** `MongoDB9`

3) **MongoDB9**  
- **Type/role:** Stores run config / creates run record.  
- **Output:** `Get many products`

4) **Get many products** (Shopify)  
- **Type/role:** Fetches products list for internal linking/product highlights.  
- **Output:** `Code in JavaScript1`  
- **Failure modes:** Shopify credentials; pagination; API limits.

5) **Code in JavaScript1**  
- **Type/role:** Normalizes product objects for the selector agent (titles, handles, tags, URLs).  
- **Output:** `product selector`

6) **product selector** *(Agent v3)*  
- **Type/role:** Chooses products relevant to the planned content.  
- **Model:** `OpenAI Chat Model`  
- **Parser:** `Structured Output Parser9`  
- **Output:** `MongoDB12`  
- **Failure modes:** Over-selecting; invalid product IDs; schema mismatch.

7) **MongoDB12**  
- **Type/role:** Persist selected products.  
- **Output:** `Task Definition Agent`

8) **If5**  
- **Type/role:** Branch: either proceed with writing pipeline (`Loop Over Items1`) or go directly to task definition (`MongoDB12`).  
- **Outputs:**  
  - True → `Loop Over Items1`  
  - False → `MongoDB12`  
- **Failure modes:** Logic inversion; “no products” edge case should route to a fallback.

---

### Block 2.6 — Task definition + content writing with tools and memory
**Overview:** Creates a structured writing brief (tasks/outline), then writes the article using Gemini + tools (web search, link checking, image generation) and MongoDB chat memory.

**Nodes involved:**
- Task Definition Agent
- OpenAI Chat Model2
- Structured Output Parser8
- Split Out3
- Code in JavaScript2
- MongoDB10
- Content Writer Agent
- Google Gemini Chat Model
- Memory
- Structured Output Parser
- Generate image (sub-workflow tool)
- search the web (sub-workflow tool)
- check link status (sub-workflow tool)
- Code13
- MongoDB11
- Code18

**Node details:**

1) **Task Definition Agent** *(retryOnFail:true)*  
- **Type/role:** Agent that turns inputs (selected products + config + intelligence) into a task list/outline.  
- **Model:** `OpenAI Chat Model2` (dedicated)  
- **Parser:** `Structured Output Parser8`  
- **Output:** `Split Out3`  
- **Failure modes:** Outline not aligned to parser schema; token limit if too much context.

2) **Split Out3**  
- **Type/role:** Splits tasks into items.  
- **Output:** `Code in JavaScript2`

3) **Code in JavaScript2**  
- **Type/role:** Maps tasks into a loopable structure.  
- **Output:** `If5` (feeds either loop or direct).

4) **Loop Over Items1** (SplitInBatches)  
- **Type/role:** Iterates tasks/articles to write.  
- **Outputs:**  
  - Done → `Aggregate1`  
  - Each → `Code15`  
- **Failure modes:** Batch size too high causes LLM saturation.

5) **Code15** → **MongoDB10**  
- **Type/role:** Prepare DB record for a writing job; persist.  
- **Output:** `Content Writer Agent`

6) **Content Writer Agent** *(typeVersion 2.1, maxTries:5, waitBetweenTries:1000, retryOnFail:true)*  
- **Type/role:** Main writing agent generating SEO blog article, likely with internal links/products.  
- **Model:** `Google Gemini Chat Model`  
- **Tools:** from `Think2`:  
  - `search the web` (toolWorkflow)  
  - `check link status` (toolWorkflow)  
  - `Generate image` (toolWorkflow)  
- **Memory:** `Memory` (MongoDB chat)  
- **Parser:** `Structured Output Parser`  
- **Output:** `Code13`  
- **Failure modes:** Tool sub-workflows missing; Gemini credential errors; structured output mismatch; long-form generation token limits; web search returns unusable data.

7) **Code13** *(onError: continueErrorOutput)*  
- **Type/role:** Post-process writer output (sanitize HTML/markdown, extract fields).  
- **Output:** `MongoDB11`  
- **Failure modes:** If parser output missing keys, code throws; continueErrorOutput may hide partial failures.

8) **MongoDB11** → **Code18**  
- **Type/role:** Store generated draft; then `Code18` loops back into `Loop Over Items1`.  
- **Failure modes:** Update/insert mismatch; writing job state not tracked robustly.

---

### Block 2.7 — Quality check and conditional improvement loop
**Overview:** Aggregates finished drafts, runs a quality evaluation agent, and if below threshold loops through a content modifier agent and persists improved drafts.

**Nodes involved:**
- Aggregate1
- MongoDB14
- Quality Check Agent
- Structured Output Parser16
- If content meets threshold
- Code32
- MongoDB18
- Code20
- Split Out5
- Loop Over Items3
- Content modifier
- Structured Output Parser5
- Code4
- MongoDB
- Code40
- Limit1
- MongoDB29
- Code33
- MongoDB30
- Code41
- MongoDB3

**Node details:**

1) **Aggregate1**  
- **Type/role:** Aggregates all completed drafts from loop end.  
- **Output:** `MongoDB14`

2) **MongoDB14**  
- **Type/role:** Stores aggregation or loads the draft set for QA.  
- **Output:** `Quality Check Agent`

3) **Quality Check Agent** *(retryOnFail:true)*  
- **Type/role:** Evaluates content vs a threshold (SEO, readability, compliance, link validity).  
- **Model:** `Google Gemini Chat Model`  
- **Parser:** `Structured Output Parser16`  
- **Output:** `If content meets threshold`  
- **Failure modes:** Non-deterministic scoring; schema mismatch; excessive strictness causes endless revisions.

4) **If content meets threshold**  
- **Type/role:** Gate.  
- **Outputs:**  
  - Pass → `Code32` (persist and continue downstream)  
  - Fail → `Split Out5` (revision loop)  
- **Failure modes:** Condition references missing score field; threshold misconfigured.

5) **Revision loop path:**  
- **Split Out5** → **Loop Over Items3** (SplitInBatches) → **Content modifier**  
- **Content modifier** *(Agent 2.1, maxTries:5)*  
  - **Model:** `Google Gemini Chat Model`  
  - **Tools:** `search the web`, `check link status`  
  - **Parser:** `Structured Output Parser5`  
  - **Output:** `Code4`  
- **Code4** *(continueErrorOutput)* → **MongoDB** → **Code40** (feeds back to Loop Over Items3)  
- Additional persistence/controls: `Limit1` → `MongoDB29` → `Code33` → `MongoDB30(alwaysOutputData)` → `Code41` → `MongoDB3`  
- **Failure modes:** Infinite revision loop if no stop condition; `Limit1` suggests a cap but parameters not shown; repeated rewrites may drift from original intent.

6) **Pass path:**  
- **Code32** → **MongoDB18(alwaysOutputData)** → **Code20** → **MongoDB3**  
- **Role:** Persist “approved” state and route into the title/metadata/publishing chain (MongoDB3 is the start of that chain).

---

### Block 2.8 — Title generation, metadata generation, and chat cleanup
**Overview:** Generates blog title ideas, selects the best title, generates meta data, optionally branches, and clears chat history.

**Nodes involved:**
- MongoDB3
- Blog title generator
- Structured Output Parser2
- Code7
- MongoDB7
- Code19
- MongoDB5
- Meta data generator
- Structured Output Parser4
- If3
- Code6
- MongoDB6
- Code22
- delete chat history
- MongoDB8

**Node details:**

1) **MongoDB3**  
- **Type/role:** Loads the “approved draft” record(s) for title/meta generation.  
- **Output:** `Blog title generator`

2) **Blog title generator** *(Agent, notes: “generate blog title ideas”)*  
- **Type/role:** Produces multiple SEO titles.  
- **Model:** `Google Gemini Chat Model`  
- **Parser:** `Structured Output Parser2`  
- **Output:** `Code7`  
- **Failure modes:** Titles not matching schema; language mismatch.

3) **Code7** *(notes: “rank and choose the best one”)*  
- **Type/role:** Ranks and selects best title.  
- **Output:** `MongoDB7`

4) **MongoDB7** → **Code19**  
- **Type/role:** Save selected title; Code19 continues ranking/selection (also notes “rank and choose the best one”).  
- **Output:** `MongoDB5`

5) **MongoDB5** → **Meta data generator**  
- **Type/role:** Provide draft+title to metadata agent.  
- **Meta data generator** *(Agent, notes: “generate meta data”)*  
  - **Model:** `Google Gemini Chat Model`  
  - **Parser:** `Structured Output Parser4`  
  - **Output:** `If3`  
- **Failure modes:** Meta description length constraints; invalid JSON.

6) **If3** → **Code6** → **MongoDB6** → **Code22** → **delete chat history** → **MongoDB8**  
- **Role:** If meta generation conditions pass, persists meta, then cleans up chat history in MongoDB to prevent memory bloat/cross-contamination, then routes into category/publishing prep chain (`MongoDB8`).

---

### Block 2.9 — Gemini sub-workflow call + Shopify publishing preparation + category selection
**Overview:** Calls an external “Gemini tool” workflow, persists results, creates Shopify Storefront token, fetches categories, and publishes the blog.

**Nodes involved:**
- AI Agent1
- Structured Output Parser3
- Call gemini tool (Execute Workflow)
- Code12
- MongoDB15
- Code24
- Create Storefront Token (GraphQL)
- MongoDB22
- Split Out4
- AI Agent
- OpenAI Chat Model3
- Code in JavaScript
- Aggregate2
- Edit Fields1
- If2
- Get categories (Google Sheets)
- Create Blog And Publish (HTTP Request)
- Code23
- MongoDB19
- MongoDB26

**Node details:**

1) **MongoDB8** → **AI Agent1**  
- **Type/role:** Agent to prepare a payload for Gemini tool or refine content.  
- **Model:** `Google Gemini Chat Model`  
- **Parser:** `Structured Output Parser3`  
- **Output:** `Call gemini tool`  
- **Failure modes:** Tool-workflow expected fields missing.

2) **Call gemini tool** (Execute Workflow)  
- **Type/role:** Invokes a separate n8n workflow (sub-workflow).  
- **Config:** Not shown (parameters empty), but must specify **Workflow ID** and map inputs/outputs in UI.  
- **Output:** `Code12`  
- **Failure modes:** Sub-workflow not found/disabled; missing “Execute Workflow” permissions; incompatible input contract.

3) **Code12** → **MongoDB15** → **Code24**  
- **Role:** Persist sub-workflow results; then prepare Shopify token creation request.

4) **Create Storefront Token (GraphQL)** (HTTP Request)  
- **Type/role:** Calls Shopify Admin API (GraphQL) to create a Storefront token.  
- **Output:** `MongoDB22`  
- **Failure modes:** Shopify auth (Admin API token); wrong endpoint/version; missing scopes.

5) **MongoDB22** → **Split Out4** → **AI Agent**  
- **Type/role:** AI Agent (v2.1) further prepares publishing payload (likely: HTML, handle, tags, image alt text).  
- **Model:** `OpenAI Chat Model3`  
- **Output:** `Code in JavaScript`  
- **Failure modes:** Schema drift; content too large; model refusal if policy triggers.

6) **Code in JavaScript** → **Aggregate2** → **Edit Fields1** → **If2**  
- **Role:** Build final publishing object; gate whether to proceed to categories/publish.

7) **Get categories** (Google Sheets)  
- **Type/role:** Loads category mapping/taxonomy.  
- **Output:** `Create Blog And Publish`

8) **Create Blog And Publish** (HTTP Request)  
- **Type/role:** Publishes the blog post to Shopify (likely GraphQL blogPostCreate/blogPostUpdate + publish).  
- **Output:** `Code23`  
- **Failure modes:** Missing required blog ID; HTML rejected; image hosting issues; API limits; idempotency not handled.

9) **Code23** → **MongoDB19** → **MongoDB26**  
- **Role:** Persist publish result and feed promotion generation chain.

---

### Block 2.10 — Tweet generation and persistence
**Overview:** Produces a tweet from the published article, posts it to Twitter/X, and stores the result.

**Nodes involved:**
- Basic LLM Chain
- OpenAI Chat Model (as LLM Chain input index 1)
- Google Gemini Chat Model (as LLM Chain input index 0)
- Create Tweet (Twitter)
- Code26
- MongoDB24
- e4a16461… MongoDB26 (upstream)

**Node details:**

1) **MongoDB26**  
- **Type/role:** Loads necessary fields (URL, title, key takeaways) for tweet creation.  
- **Output:** `Basic LLM Chain`

2) **Basic LLM Chain** (LangChain)  
- **Type/role:** Deterministic chain to generate tweet copy.  
- **Models attached:**  
  - `Google Gemini Chat Model` (index 0)  
  - `OpenAI Chat Model` (index 1)  
- **Output:** `Create Tweet`  
- **Failure modes:** Tweet length constraints; chain prompt not aligned; model selection ambiguity if not configured.

3) **Create Tweet** (Twitter node)  
- **Type/role:** Publishes tweet.  
- **Output:** `Code26`  
- **Failure modes:** Twitter auth; API tier restrictions; duplicate content errors.

4) **Code26** → **MongoDB24**  
- **Role:** Persist tweet metadata (tweet ID, URL, status).  
- **Failure modes:** Missing tweet response fields.

---

## 3. Summary Table

> Sticky notes exist but their `content` fields are empty in the provided JSON, so the Sticky Note column is blank unless a node has in-node notes.

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Articles Daily | scheduleTrigger | Daily ingestion trigger | — | Get rss feed |  |
| Get rss feed | googleSheets | Load RSS feed URLs | Get Articles Daily | Split Out |  |
| Split Out | splitOut | Split sheet rows into items | Get rss feed | Read RSS News Feeds |  |
| Read RSS News Feeds | rssFeedRead | Read RSS entries | Split Out | Filter |  |
| Filter | filter | Filter RSS items | Read RSS News Feeds | Limit |  |
| Limit | limit | Cap RSS items | Filter | Set and Normalize Fields |  |
| Set and Normalize Fields | set | Normalize RSS fields | Limit | Loop Over Items2 |  |
| Loop Over Items2 | splitInBatches | Batch RSS processing | If, HTTP Request4, Set and Normalize Fields, MongoDB Atlas Vector Store | Aggregate, HTTP Request4 |  |
| HTTP Request4 | httpRequest | Fetch article page | Loop Over Items2 | Code17, Loop Over Items2 |  |
| Code17 | code | Parse/clean fetched content | HTTP Request4 | Crypto |  |
| Crypto | crypto | Hash for dedupe | Code17 | Find documents |  |
| Find documents | mongoDb | Check duplicates | Crypto | If |  |
| If | if | Branch: exists vs new | Find documents | Loop Over Items2, Code2 |  |
| Code2 | code | Prepare new doc | If | Insert documents |  |
| Insert documents | mongoDb | Insert article | Code2 | MongoDB Atlas Vector Store |  |
| Recursive Character Text Splitter | textSplitterRecursiveCharacterTextSplitter | Chunk text for embeddings | — | Default Data Loader |  |
| Default Data Loader | documentDefaultDataLoader | Build LangChain Documents | Recursive Character Text Splitter | MongoDB Atlas Vector Store |  |
| openai-text-embedding-3-small | embeddingsOpenAi | Generate embeddings | — | MongoDB Atlas Vector Store, MongoDB Atlas Vector Store1 |  |
| MongoDB Atlas Vector Store | vectorStoreMongoDBAtlas | Store embeddings | Insert documents, Default Data Loader, openai-text-embedding-3-small | Loop Over Items2 |  |
| Aggregate | aggregate | Aggregate after ingestion | Loop Over Items2 | Get row(s) in sheet |  |
| Get row(s) in sheet | googleSheets | Load rows for agent | Aggregate | Aggregate4 |  |
| Aggregate4 | aggregate | Bundle rows | Get row(s) in sheet | AI Agent12 |  |
| AI Agent12 | agent | Transform rows with RAG | Aggregate4, OpenAI Chat Model, MongoDB Atlas Vector Store1, Structured Output Parser7 | Split Out2 |  |
| MongoDB Atlas Vector Store1 | vectorStoreMongoDBAtlas | Retrieval tool for RAG | openai-text-embedding-3-small | AI Agent12 |  |
| Structured Output Parser7 | outputParserStructured | Enforce schema | — | AI Agent12 |  |
| Split Out2 | splitOut | Split agent outputs | AI Agent12 | If1 |  |
| If1 | if | Gate inserts | Split Out2 | Code |  |
| Code | code | Prep docs for insert | If1 | Insert documents1 |  |
| Insert documents1 | mongoDb | Insert structured results | Code | Loop Over Items |  |
| Loop Over Items | splitInBatches | Batch “intelligence” processing | Update documents, Insert documents1, If4 | delete news chunks, If4 |  |
| delete news chunks | mongoDb | Cleanup chunks | Loop Over Items | delete news articles |  |
| delete news articles | mongoDb | Cleanup articles (disabled) | delete news chunks | — |  |
| If4 | if | Control flow in loop | Loop Over Items | Split Out1, Loop Over Items |  |
| Split Out1 | splitOut | Split list into items | If4 | Find documents2 |  |
| Find documents2 | mongoDb | Load docs for intelligence | Split Out1 | Aggregate3 |  |
| Aggregate3 | aggregate | Bundle docs | Find documents2 | Article Intelligence Agent |  |
| Article Intelligence Agent | agent | Extract structured insights | Aggregate3, OpenAI Chat Model, Structured Output Parser1, Think2 | Code1 |  |
| Structured Output Parser1 | outputParserStructured | Enforce intelligence schema | — | Article Intelligence Agent |  |
| Code1 | code | Map intelligence to update | Article Intelligence Agent | Update documents |  |
| Update documents | mongoDb | Update docs with insights | Code1 | Loop Over Items |  |
| Schedule Trigger | scheduleTrigger | Main production trigger | — | Get details |  |
| Get details | googleSheets | Load run config | Schedule Trigger | MongoDB9 |  |
| MongoDB9 | mongoDb | Store/load run record | Get details | Get many products |  |
| Get many products | shopify | Fetch products | MongoDB9 | Code in JavaScript1 |  |
| Code in JavaScript1 | code | Normalize products | Get many products | product selector |  |
| product selector | agent | Select relevant products | Code in JavaScript1, OpenAI Chat Model, Structured Output Parser9 | MongoDB12 |  |
| Structured Output Parser9 | outputParserStructured | Enforce product selection schema | — | product selector |  |
| MongoDB12 | mongoDb | Store selected products | product selector, If5 | Task Definition Agent |  |
| If5 | if | Branch: loop vs direct | Code in JavaScript2 | Loop Over Items1, MongoDB12 |  |
| Task Definition Agent | agent | Create outline/tasks | MongoDB12, OpenAI Chat Model2, Structured Output Parser8, Think2 | Split Out3 |  |
| OpenAI Chat Model2 | lmChatOpenAi | LLM for task definition | — | Task Definition Agent |  |
| Structured Output Parser8 | outputParserStructured | Enforce task schema | — | Task Definition Agent |  |
| Split Out3 | splitOut | Split tasks | Task Definition Agent | Code in JavaScript2 |  |
| Code in JavaScript2 | code | Prepare items for writing | Split Out3 | If5 |  |
| Loop Over Items1 | splitInBatches | Iterate writing tasks | If5, Code18 | Aggregate1, Code15 |  |
| Code15 | code | Prep writing job record | Loop Over Items1 | MongoDB10 |  |
| MongoDB10 | mongoDb | Store writing job | Code15 | Content Writer Agent |  |
| Memory | memoryMongoDbChat | Persist agent memory | — | Content Writer Agent |  |
| Content Writer Agent | agent | Write article w/ tools | MongoDB10, Google Gemini Chat Model, Memory, Structured Output Parser, Think2 | Code13 |  |
| Google Gemini Chat Model | lmChatGoogleGemini | Main writer/QA model | — | Content Writer Agent, Quality Check Agent, Content modifier, AI Agent1, Blog title generator, Basic LLM Chain, Meta data generator |  |
| Structured Output Parser | outputParserStructured | Enforce writer schema | — | Content Writer Agent |  |
| search the web | toolWorkflow | Sub-workflow tool | — | Content Writer Agent, Content modifier |  |
| check link status | toolWorkflow | Sub-workflow tool | — | Content Writer Agent, Content modifier |  |
| Generate image | toolWorkflow | Sub-workflow tool | — | Content Writer Agent |  |
| Think2 | toolThink | Shared “Think/tool router” | — | Multiple agents |  |
| Code13 | code | Post-process draft | Content Writer Agent | MongoDB11 |  |
| MongoDB11 | mongoDb | Store draft | Code13 | Code18 |  |
| Code18 | code | Continue loop | MongoDB11 | Loop Over Items1 |  |
| Aggregate1 | aggregate | Bundle drafts | Loop Over Items1 | MongoDB14 |  |
| MongoDB14 | mongoDb | Store/load for QA | Aggregate1 | Quality Check Agent |  |
| Quality Check Agent | agent | Score/evaluate content | MongoDB14, Google Gemini Chat Model, Structured Output Parser16, Think2 | If content meets threshold |  |
| Structured Output Parser16 | outputParserStructured | Enforce QA schema | — | Quality Check Agent |  |
| If content meets threshold | if | Pass/fail gate | Quality Check Agent | Code32, Split Out5 |  |
| Code32 | code | Persist pass state | If content meets threshold | MongoDB18 |  |
| MongoDB18 | mongoDb | Store approved result | Code32 | Code20 |  |
| Code20 | code | Route to title/meta chain | MongoDB18 | MongoDB3 |  |
| Split Out5 | splitOut | Split revisions | If content meets threshold | Loop Over Items3 |  |
| Loop Over Items3 | splitInBatches | Iterate revisions | Split Out5, Code40 | Limit1, Content modifier |  |
| Content modifier | agent | Rewrite/improve content | Loop Over Items3, Google Gemini Chat Model, Structured Output Parser5, Think2 | Code4 |  |
| Structured Output Parser5 | outputParserStructured | Enforce rewrite schema | — | Content modifier |  |
| Code4 | code | Post-process rewrite | Content modifier | MongoDB |  |
| MongoDB | mongoDb | Store rewritten draft | Code4 | Code40 |  |
| Code40 | code | Continue revision loop | MongoDB | Loop Over Items3 |  |
| Limit1 | limit | Cap revision iterations | Loop Over Items3 | MongoDB29 |  |
| MongoDB29 | mongoDb | Persist loop state | Limit1 | Code33 |  |
| Code33 | code | Prepare routing | MongoDB29 | MongoDB30 |  |
| MongoDB30 | mongoDb | Persist routing decision | Code33 | Code41 |  |
| Code41 | code | Route to next stage | MongoDB30 | MongoDB3 |  |
| MongoDB3 | mongoDb | Start title/meta chain | Code20, Code41 | Blog title generator |  |
| Blog title generator | agent | Generate title ideas | MongoDB3, Google Gemini Chat Model, Structured Output Parser2, Think2 | Code7 | generate blog title ideas |
| Structured Output Parser2 | outputParserStructured | Enforce title schema | — | Blog title generator |  |
| Code7 | code | Pick best title | Blog title generator | MongoDB7 | rank and choose the best one |
| MongoDB7 | mongoDb | Store chosen title | Code7 | Code19 |  |
| Code19 | code | Further selection step | MongoDB7 | MongoDB5 | rank and choose the best one |
| MongoDB5 | mongoDb | Load for meta generation | Code19 | Meta data generator |  |
| Meta data generator | agent | Generate meta data | MongoDB5, Google Gemini Chat Model, Structured Output Parser4 | If3 | generate meta data |
| Structured Output Parser4 | outputParserStructured | Enforce meta schema | — | Meta data generator |  |
| If3 | if | Gate meta step | Meta data generator | Code6 |  |
| Code6 | code | Persist meta | If3 | MongoDB6 | rank and choose the best one |
| MongoDB6 | mongoDb | Store meta | Code6 | Code22 |  |
| Code22 | code | Prepare cleanup | MongoDB6 | delete chat history | rank and choose the best one |
| delete chat history | mongoDb | Delete agent chat history | Code22 | MongoDB8 |  |
| MongoDB8 | mongoDb | Store state for publish prep | delete chat history | AI Agent1 |  |
| AI Agent1 | agent | Prep for Gemini subflow | MongoDB8, Google Gemini Chat Model, Structured Output Parser3 | Call gemini tool |  |
| Structured Output Parser3 | outputParserStructured | Enforce schema | — | AI Agent1 |  |
| Call gemini tool | executeWorkflow | Invoke sub-workflow | AI Agent1 | Code12 |  |
| Code12 | code | Map subflow output | Call gemini tool | MongoDB15 |  |
| MongoDB15 | mongoDb | Store subflow result | Code12 | Code24 |  |
| Code24 | code | Prepare token request | MongoDB15 | Create Storefront Token (GraphQL) |  |
| Create Storefront Token (GraphQL) | httpRequest | Create Shopify token | Code24 | MongoDB22 |  |
| MongoDB22 | mongoDb | Store token/payload | Create Storefront Token (GraphQL) | Split Out4 |  |
| Split Out4 | splitOut | Split publish items | MongoDB22 | AI Agent |  |
| AI Agent | agent | Prepare publish payload | Split Out4, OpenAI Chat Model3, Think2 | Code in JavaScript |  |
| OpenAI Chat Model3 | lmChatOpenAi | LLM for publish prep | — | AI Agent |  |
| Code in JavaScript | code | Normalize publish payload | AI Agent | Aggregate2 |  |
| Aggregate2 | aggregate | Bundle for publish | Code in JavaScript | Edit Fields1 |  |
| Edit Fields1 | set | Final field mapping | Aggregate2 | If2 |  |
| If2 | if | Gate category/publish | Edit Fields1 | Get categories |  |
| Get categories | googleSheets | Load category mapping | If2 | Create Blog And Publish |  |
| Create Blog And Publish | httpRequest | Publish to Shopify | Get categories | Code23 |  |
| Code23 | code | Map publish result | Create Blog And Publish | MongoDB19 |  |
| MongoDB19 | mongoDb | Store publish result | Code23 | MongoDB26 |  |
| MongoDB26 | mongoDb | Load for tweet gen | MongoDB19 | Basic LLM Chain |  |
| Basic LLM Chain | chainLlm | Generate tweet copy | MongoDB26, Google Gemini Chat Model, OpenAI Chat Model | Create Tweet |  |
| OpenAI Chat Model | lmChatOpenAi | LLM for some agents/chain | — | AI Agent12, Article Intelligence Agent, Basic LLM Chain, product selector |  |
| Create Tweet | twitter | Post tweet | Basic LLM Chain | Code26 |  |
| Code26 | code | Persist tweet fields | Create Tweet | MongoDB24 |  |
| MongoDB24 | mongoDb | Store tweet result | Code26 | — |  |
| Sticky Note* nodes | stickyNote | Visual annotations | — | — | (empty content in JSON) |
| Structured Output Parser8/9/16 etc. | outputParserStructured | Schema enforcement | — | respective agents |  |

---

## 4. Reproducing the Workflow from Scratch

> Because most node `parameters` are empty in the provided JSON, you must configure operations, fields, prompts, and schemas manually. The steps below recreate the **architecture and wiring** and list what must be configured.

### 4.1 Prerequisites (credentials & external systems)
1) **MongoDB credential** in n8n:
   - Connection string to MongoDB Atlas (or self-hosted).
   - Collections you will use (examples): `news_articles`, `news_chunks`, `drafts`, `runs`, `chat_history`, `publishes`, `tweets`.
2) **MongoDB Atlas Vector Search index**:
   - Create a Vector Search index on a collection storing chunks (e.g., `news_chunks`).
   - Ensure embedding dimension matches `text-embedding-3-small`.
3) **Google Sheets credential** (OAuth2 or Service Account):
   - Sheet A: list of RSS feed URLs
   - Sheet B: run configuration/details
   - Sheet C: categories mapping/taxonomy
4) **Shopify credentials**:
   - Admin API access token (for GraphQL/REST publishing and token creation).
   - Store domain and API version.
5) **OpenAI credential**:
   - For Chat model nodes and embedding node.
6) **Google Gemini credential**:
   - For Gemini chat model nodes.
7) **Twitter/X credential**:
   - For posting tweets.
8) **Sub-workflows** used as tools:
   - A workflow for **web search**
   - A workflow for **link status checking**
   - A workflow for **image generation**
   - A workflow invoked by **Execute Workflow** (“Call gemini tool”)

### 4.2 Build Block: Daily RSS ingestion
1) Add **Schedule Trigger** → name it `Get Articles Daily`; set daily time.
2) Add **Google Sheets** → `Get rss feed`; configure:
   - Operation: *Read* / *Get many rows*
   - Spreadsheet + sheet with RSS URLs
3) Connect `Get Articles Daily` → `Get rss feed`.
4) Add **Split Out**; connect `Get rss feed` → `Split Out`.
5) Add **RSS Feed Read** → `Read RSS News Feeds`; set URL from item field (expression).
6) Connect `Split Out` → `Read RSS News Feeds`.
7) Add **Filter** to keep valid/new enough entries; connect after RSS read.
8) Add **Limit** to cap items; connect.
9) Add **Set** → `Set and Normalize Fields`; map to canonical fields: `title`, `link`, `pubDate`, `source`, etc.
10) Add **Split In Batches** → `Loop Over Items2`; connect from `Set and Normalize Fields`.

### 4.3 Build Block: Fetch article page + dedupe + store + embed
11) Add **HTTP Request** → `HTTP Request4`; set:
   - Method GET
   - URL = normalized `link`
   - Response: string/body
   - Set **Continue On Fail** (or node onError as in JSON)
12) Connect `Loop Over Items2` → `HTTP Request4`.
13) Add **Code** → `Code17`; parse HTML to text (Readability, regex, etc.).
14) Connect `HTTP Request4` → `Code17`.
15) Add **Crypto** → `Crypto`; create hash from URL + text.
16) Connect `Code17` → `Crypto`.
17) Add **MongoDB** → `Find documents`; operation *Find* with filter by hash/url; enable **Always Output Data**.
18) Connect `Crypto` → `Find documents`.
19) Add **If** → `If`; condition: “found any documents?”.
20) Wire `Find documents` → `If`.
21) If **exists**: connect `If (true)` → `Loop Over Items2` (continue).
22) If **new**: add **Code** → `Code2` to shape doc; connect `If (false)` → `Code2`.
23) Add **MongoDB** → `Insert documents`; operation *Insert* into `news_articles`; connect `Code2` → `Insert documents`.
24) Add **Recursive Character Text Splitter**; configure chunk size/overlap.
25) Add **Default Data Loader**; map the text field to document content; connect splitter → loader (ai_textSplitter).
26) Add **OpenAI Embeddings** → `openai-text-embedding-3-small`; set model accordingly.
27) Add **MongoDB Atlas Vector Store**; configure:
   - Atlas connection/collection/index name
   - Upsert mode
   - Connect: `Insert documents` (main) → vector store (main)
   - Connect: data loader → vector store (ai_document)
   - Connect: embeddings → vector store (ai_embedding)
28) Connect vector store output back to `Loop Over Items2` to proceed.

### 4.4 Build Block: Transform sheet rows + RAG agent + insert
29) Add **Aggregate** → `Aggregate`; connect from `Loop Over Items2` “done” output.
30) Add **Google Sheets** → `Get row(s) in sheet`; configure to load rows for prompt/context.
31) Connect `Aggregate` → `Get row(s) in sheet`.
32) Add **Aggregate** → `Aggregate4`; connect from sheet.
33) Add **AI Agent** → `AI Agent12`; configure:
   - Attach **OpenAI Chat Model** (create `OpenAI Chat Model` node and connect as ai_languageModel)
   - Attach **MongoDB Atlas Vector Store1** as retrieval tool (create node and connect ai_tool + embeddings)
   - Attach **Structured Output Parser7** with a schema that outputs an array of objects to insert
34) Add **Split Out2** then **If1** to filter valid objects.
35) Add **Code** → `Code` to shape docs; then **MongoDB** → `Insert documents1`.
36) Connect `Insert documents1` into the intelligence loop starter (`Loop Over Items`).

### 4.5 Build Block: Article intelligence extraction
37) Add **Split In Batches** → `Loop Over Items`.
38) Add MongoDB deletes (optional): `delete news chunks` and (disabled) `delete news articles`.
39) Add **If4** to route between splitting docs and continuing loop.
40) Add `Split Out1` → `Find documents2` → `Aggregate3`.
41) Add **Article Intelligence Agent**; configure:
   - LLM: `OpenAI Chat Model`
   - Parser: `Structured Output Parser1`
   - (Optional) attach `Think2` toolThink for shared tools
42) Add **Code1** to map into updates; add **MongoDB** → `Update documents` and loop back into `Loop Over Items`.

### 4.6 Build Block: Main scheduled production + product selection
43) Add second **Schedule Trigger** → `Schedule Trigger`.
44) Add **Google Sheets** → `Get details` (run config), connect.
45) Add **MongoDB9** to store run record; connect.
46) Add **Shopify** → `Get many products`; configure product list retrieval; connect.
47) Add **Code in JavaScript1** to normalize products.
48) Add **product selector** agent (v3) with `OpenAI Chat Model` + `Structured Output Parser9`; connect to **MongoDB12**.

### 4.7 Build Block: Task definition + writing
49) Add **Task Definition Agent** with `OpenAI Chat Model2` + `Structured Output Parser8`.
50) Add `Split Out3` → `Code in JavaScript2` → `If5` to decide loop.
51) Add **Loop Over Items1**; inside loop:
   - `Code15` → `MongoDB10` → `Content Writer Agent`
52) Add **Memory** (MongoDB chat memory) and connect as ai_memory to writer agent.
53) Create tool nodes:
   - `search the web` (Tool Workflow)
   - `check link status` (Tool Workflow)
   - `Generate image` (Tool Workflow)
   Connect them to writer agent as ai_tool.
54) Attach `Google Gemini Chat Model` + `Structured Output Parser` to writer agent.
55) Add `Code13` (continue on fail) → `MongoDB11` → `Code18` back to loop.

### 4.8 Build Block: QA + revision loop
56) `Aggregate1` → `MongoDB14` → `Quality Check Agent` (Gemini + `Structured Output Parser16`).
57) Add `If content meets threshold`:
   - Pass: `Code32` → `MongoDB18` → `Code20` → `MongoDB3`
   - Fail: `Split Out5` → `Loop Over Items3` → `Content modifier` (Gemini + `Structured Output Parser5`) → `Code4` → `MongoDB` → `Code40` back to loop
58) Add a hard stop: configure `Limit1` in the revision loop, and store revision state in `MongoDB29/MongoDB30`.

### 4.9 Build Block: Title + meta + cleanup
59) From `MongoDB3`: `Blog title generator` (Gemini + `Structured Output Parser2`) → `Code7` → `MongoDB7` → `Code19` → `MongoDB5`.
60) `Meta data generator` (Gemini + `Structured Output Parser4`) → `If3` → `Code6` → `MongoDB6` → `Code22` → `delete chat history` → `MongoDB8`.

### 4.10 Build Block: Sub-workflow + Shopify publish + tweet
61) `AI Agent1` (Gemini + `Structured Output Parser3`) → `Call gemini tool` (Execute Workflow; configure workflow target + input mapping).
62) `Code12` → `MongoDB15` → `Code24` → `Create Storefront Token (GraphQL)` → `MongoDB22`.
63) `Split Out4` → `AI Agent` (OpenAI Chat Model3) → `Code in JavaScript` → `Aggregate2` → `Edit Fields1` → `If2`.
64) `Get categories` (Sheets) → `Create Blog And Publish` (HTTP Request to Shopify) → `Code23` → `MongoDB19` → `MongoDB26`.
65) `Basic LLM Chain` (Gemini/OpenAI) → `Create Tweet` (Twitter) → `Code26` → `MongoDB24`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky Note nodes exist but contain empty text in the provided workflow JSON. | Visual annotations are present but no content to preserve. |
| Several nodes contain in-node notes: “generate blog title ideas”, “rank and choose the best one”, “generate meta data”. | Embedded as node notes on Blog/title/meta chain nodes. |
| Multiple Sub-Workflow tools are required (`search the web`, `check link status`, `Generate image`, and `Call gemini tool`). | You must create/enable these workflows and map their inputs/outputs to match the agents’ expectations. |

