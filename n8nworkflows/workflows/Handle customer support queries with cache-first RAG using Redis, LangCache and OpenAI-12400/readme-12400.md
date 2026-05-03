Handle customer support queries with cache-first RAG using Redis, LangCache and OpenAI

https://n8nworkflows.xyz/workflows/handle-customer-support-queries-with-cache-first-rag-using-redis--langcache-and-openai-12400


# Handle customer support queries with cache-first RAG using Redis, LangCache and OpenAI

## 1. Workflow Overview

**Title:** Handle customer support queries with cache-first RAG using Redis, LangCache and OpenAI

**Purpose:**  
A cache-first customer support assistant that (1) decomposes a user’s chat query into cacheable sub-questions, (2) attempts to answer each sub-question via **LangCache** (semantic cache), (3) falls back to **Redis Vector Store RAG** (OpenAI embeddings + Redis retrieval tool) on cache misses, (4) **quality-scores** the retrieved answer, (5) **saves only high-quality answers** back to LangCache, and finally (6) **synthesizes** all sub-answers into one user-facing response.

**Target use cases:** support chatbots, internal help desks, KB assistants where speed, cost control, and reducing hallucinations matter.

### 1.1 Entry Points & Major Logical Blocks
1. **Knowledge Base Preparation (scheduled ingestion)**  
   Loads example KB documents and inserts them into a Redis vector index.
2. **Chat Intake + Configuration**  
   Receives chat messages and sets LangCache parameters.
3. **Query Decomposition**  
   Uses an LLM to decide whether to split the query into 2–4 sub-questions (structured output).
4. **Per-Question Loop + Cache-First Lookup**  
   Iterates each sub-question; checks LangCache similarity search first.
5. **RAG Retrieval on Cache Miss + Quality Gate + Retry Control**  
   Uses Redis vector retrieval as an LLM tool, evaluates answer quality, retries within a max iteration limit, then caches acceptable results.
6. **Aggregation + Final Response Synthesis**  
   Aggregates gathered Q/A data and synthesizes a final customer-facing response.

---

## 2. Block-by-Block Analysis

### Block A — Knowledge Base Preparation (Example Data → Redis Vector Insert)
**Overview:** Populates a Redis vector index with example customer support documents using OpenAI embeddings. Triggered on a schedule.  
**Nodes involved:** Schedule Trigger, example Data, Default Data Loader, Embeddings OpenAI1, Redis Vector Store, Sticky Note9

#### A1) Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — periodic trigger to run KB ingestion.
- **Config choices:** Interval rule present (template default; not customized in JSON).
- **Connections:**  
  - **Out:** `example Data`
- **Edge cases:** Misconfigured interval may run too frequently (cost/overwrites) or never.

#### A2) example Data
- **Type / role:** `n8n-nodes-base.set` — defines an in-workflow KB as an array `raw_docs`.
- **Config choices:** Sets `raw_docs` to an array of support policy strings (plans/pricing, rate limits, exports, integrations, security, billing, account recovery).
- **Connections:**  
  - **In:** Schedule Trigger  
  - **Out:** Redis Vector Store
- **Edge cases:** If `raw_docs` is empty or not an array, ingestion may fail downstream.

#### A3) Default Data Loader
- **Type / role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader` — converts incoming docs/fields to LangChain “Document” objects.
- **Config choices:** Default options.
- **Connections:**  
  - **AI document out:** to Redis Vector Store
- **Edge cases:** If upstream format is unexpected, document creation may produce zero docs.

#### A4) Embeddings OpenAI1
- **Type / role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — generates embeddings for documents.
- **Config choices:** Uses OpenAI credential “n8n free OpenAI API credits”.
- **Connections:**  
  - **AI embedding out:** to Redis Vector Store
- **Edge cases:** OpenAI auth/quota issues; model changes; rate limits; large document sizes.

#### A5) Redis Vector Store
- **Type / role:** `@n8n/n8n-nodes-langchain.vectorStoreRedis` — inserts documents into Redis vector index.
- **Config choices:**  
  - **Mode:** `insert`  
  - **Index:** `kb-3accd7ed`  
  - Uses Redis credential “Redis account”.
- **Connections:**  
  - **In:** example Data (main), Default Data Loader (ai_document), Embeddings OpenAI1 (ai_embedding)
- **Edge cases:** Wrong index name/schema, Redis auth/connection errors, missing Redis Search module / vector capability, dimension mismatch if embedding model changes.

**Sticky Note coverage:**  
- **Sticky Note9:** “## Prepare the Knowledge Base -  Example Data”

---

### Block B — Chat Intake + LangCache Configuration
**Overview:** Receives incoming chat messages and sets LangCache parameters used throughout the run.  
**Nodes involved:** When chat message received, LangCache Config, Sticky Note1

#### B1) When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for chat-based executions.
- **Config choices:** Default options; provides `chatInput` and `sessionId`.
- **Key fields used later:**  
  - `$('When chat message received').item.json.chatInput`  
  - `$('When chat message received').item.json.sessionId`
- **Connections:**  
  - **Out:** LangCache Config
- **Edge cases:** Missing `chatInput` (empty messages), missing sessionId (memory nodes rely on it).

#### B2) LangCache Config
- **Type / role:** `n8n-nodes-base.set` — central configuration for LangCache and retry parameters.
- **Config choices (assignments):**
  - `langcacheBaseUrl`: `https://aws-us-east-1.langcache.redis.io`
  - `langcacheCacheId`: `b83aa61d58be484ebc37c64f1f30c2fa`
  - `similarityThreshold`: `0.75`
  - `max_iterations`: `"2"` (string)
- **Connections:**  
  - **In:** When chat message received  
  - **Out:** decompose_query
- **Edge cases / integration issues:**
  - `max_iterations` is stored as **string**, but later compared numerically; n8n may coerce, but can also cause strict-type issues in some contexts.
  - Wrong cache ID/base URL leads to HTTP 401/404 from LangCache endpoints.

**Sticky Note coverage:**  
- **Sticky Note1:** Configuration instructions for LangCache parameters.

---

### Block C — Query Decomposition (LLM + Structured Parsing)
**Overview:** Uses an OpenAI chat model to decide whether the user query should be split into multiple cacheable sub-questions; outputs a structured JSON object `{ questions: [...] }`.  
**Nodes involved:** decompose_query, Structured Output Parser, Simple Memory, OpenAI Chat Model, Sticky Note2

#### C1) Simple Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — conversation memory for decomposition step.
- **Config choices:**  
  - `sessionKey`: `={{ $('When chat message received').item.json.sessionId }}`
  - window length: 10
- **Connections:**  
  - **AI memory out:** to decompose_query
- **Edge cases:** If sessionId missing/unstable, memory won’t persist across turns.

#### C2) OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — shared LLM used by multiple agent nodes.
- **Config choices:** `gpt-4.1-mini`
- **Connections:**  
  - **AI languageModel out:** to decompose_query, search_node1, synthesize_response_node
- **Edge cases:** Model availability, quota, rate limits, high concurrency.

#### C3) Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — forces JSON output matching a schema.
- **Config choices:** Manual JSON schema requiring:
  - `questions` (array of strings)
- **Connections:**  
  - **AI outputParser out:** to decompose_query
- **Edge cases:** If the LLM returns invalid JSON or wrong shape, parsing fails (agent node may error).

#### C4) decompose_query
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that produces the list of sub-questions.
- **Config choices:**
  - **Input text:** user chat input
  - **System message:** rules for SINGLE_QUESTION vs. 2–4 sub-questions
  - **Has output parser:** enabled (expects structured output)
  - **PromptType:** define
- **Important note:** The prompt says “If keeping as single question, respond with exactly: SINGLE_QUESTION”, but the structured schema *always* requires an array `questions`. This mismatch can break parsing unless the agent always maps SINGLE_QUESTION into `{"questions":[...]}`
- **Connections:**  
  - **In:** LangCache Config (main), OpenAI Chat Model (ai_languageModel), Simple Memory (ai_memory), Structured Output Parser (ai_outputParser)  
  - **Out:** Split Out
- **Edge cases:**
  - Output-parser failures due to SINGLE_QUESTION behavior.
  - Multi-language queries might produce unexpected decomposition formatting.

**Sticky Note coverage:**  
- **Sticky Note2:** Query decomposition rationale.

---

### Block D — Split + Loop Over Sub-Questions
**Overview:** Converts the `questions` array into individual items and processes them in batches.  
**Nodes involved:** Split Out, Loop Over Items, Sticky Note3

#### D1) Split Out
- **Type / role:** `n8n-nodes-base.splitOut` — splits `output.questions` into separate items.
- **Config choices:**
  - Field: `output.questions`
  - Destination field name: `question`
- **Connections:**  
  - **In:** decompose_query  
  - **Out:** Loop Over Items
- **Edge cases:** If `output.questions` missing/not an array, produces zero items or errors.

#### D2) Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterates items (sub-questions).
- **Config choices:** Default batch settings (no explicit batch size shown).
- **Connections:**  
  - **In:** Split Out; also receives feedback loop from Save to LangCache  
  - **Out(0):** Aggregate (collects data)  
  - **Out(1):** Search LangCache (cache lookup)
- **Edge cases:** If used incorrectly, can create loops; here it is intentionally used with a feedback connection.

**Sticky Note coverage:**  
- **Sticky Note3:** Cache-first strategy explanation.

---

### Block E — LangCache Lookup + Cache Hit Routing
**Overview:** Performs semantic cache search in LangCache; if hit, returns cached response; if miss, proceeds to retrieval.  
**Nodes involved:** Search LangCache, Is Cache Hit?, current_iteration

#### E1) Search LangCache
- **Type / role:** `n8n-nodes-base.httpRequest` — calls LangCache search endpoint.
- **Config choices:**
  - **Method:** POST
  - **URL:** `{{langcacheBaseUrl}}/v1/caches/{{langcacheCacheId}}/entries/search`
  - **Body:** `prompt` = current `question`, `similarityThreshold` from config
  - **Auth:** HTTP Bearer (generic credential type)
  - **Header:** `accept: application/json`
  - **onError:** `continueErrorOutput` (workflow continues even on request failure)
- **Connections:**  
  - **In:** Loop Over Items  
  - **Out(0):** Is Cache Hit?  
  - **Out(1):** current_iteration (secondary path)
- **Edge cases / failure modes:**
  - 401/403 invalid bearer token
  - 404 wrong cache ID
  - Network timeouts
  - Because `continueErrorOutput` is enabled, downstream nodes may receive error-shaped JSON; conditions relying on `$json.data` may evaluate unexpectedly.

#### E2) Is Cache Hit?
- **Type / role:** `n8n-nodes-base.if` — checks if similarity score meets threshold.
- **Config choices:** boolean condition:
  - `{{ $json.data?.[0]?.similarity >= similarityThreshold }}`
- **Connections:**  
  - **True path:** Loop Over Items (immediately continue loop; implicitly treating cache hit as “done”)  
  - **False path:** current_iteration (start retrieval pipeline)
- **Edge cases:**
  - If `data[0]` missing, expression yields `false` and forces retrieval (safe default).
  - If LangCache API changes response structure, condition breaks.

#### E3) current_iteration
- **Type / role:** `n8n-nodes-base.set` — initializes/keeps the retry iteration counter.
- **Config choices:**  
  - `current_iteration = {{ $json.current_iteration ?? 1 }}`
- **Connections:**  
  - **In:** Is Cache Hit? (miss path) and Search LangCache (secondary output) and increase iteration  
  - **Out:** search_node1
- **Edge cases:**
  - Field naming inconsistency later (`current_iterration` typo) breaks retry logic.

---

### Block F — Redis Vector Retrieval (Tool) + Answer Generation
**Overview:** On cache miss, an LLM agent answers using ONLY the Redis-backed KB via a retrieval tool.  
**Nodes involved:** search_node1, Redis Vector Store2, Embeddings OpenAI, Simple Memory1, Sticky Note4

#### F1) Redis Vector Store2
- **Type / role:** `@n8n/n8n-nodes-langchain.vectorStoreRedis` — exposes Redis retrieval as an LLM tool.
- **Config choices:**
  - **Mode:** `retrieve-as-tool`
  - **Index:** `kb-3accd7ed`
  - Tool description: “Using search_knowledge_base tool for query”
- **Connections:**  
  - **AI tool out:** to search_node1
  - **AI embedding in:** from Embeddings OpenAI
- **Edge cases:** Redis index not populated; embedding dimension mismatch; tool not invoked depending on agent behavior.

#### F2) Embeddings OpenAI
- **Type / role:** OpenAI embeddings for query-time retrieval.
- **Connections:**  
  - **AI embedding out:** to Redis Vector Store2
- **Edge cases:** auth/quota/rate limits.

#### F3) Simple Memory1
- **Type / role:** conversation memory for retrieval agent step.
- **Config choices:** same sessionKey, window length 10.
- **Connections:**  
  - **AI memory out:** to search_node1

#### F4) search_node1
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — “research engine” that answers sub-question strictly from KB.
- **Config choices:**
  - **Text:** current `question`
  - **System message:** forbids external knowledge; if not in KB respond exactly `no info found`
  - **Has output parser:** enabled (but no explicit parser node connected here)
- **Connections:**  
  - **In:** current_iteration (main), OpenAI Chat Model (ai_languageModel), Redis Vector Store2 (ai_tool), Simple Memory1 (ai_memory)  
  - **Out:** evaluate_quality
- **Edge cases:**
  - If the agent doesn’t call the retrieval tool, it may output `no info found` often.
  - “hasOutputParser” without a parser node can be benign depending on node defaults, but can also cause runtime configuration expectations.

**Sticky Note coverage:**  
- **Sticky Note4:** Redis vector retrieval only on cache miss.

---

### Block G — Quality Evaluation + Retry Control + Cache Save
**Overview:** Scores each sub-answer; if acceptable, saves to LangCache; if low quality, retries retrieval up to `max_iterations`.  
**Nodes involved:** evaluate_quality, getScore, low quality ?, increase iteration, Save to LangCache, Sticky Note5, Sticky Note6, Sticky Note7

#### G1) evaluate_quality
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — direct OpenAI call (not the agent node) to evaluate result quality.
- **Config choices:**
  - Model: `gpt-4.1-mini`
  - Response format: `json_object` (n8n option)
  - Prompts include original sub-question and research result; system asks for:
    - `SCORE: 0.X`
    - `FEEDBACK: ...`
- **Connections:**  
  - **In:** search_node1  
  - **Out:** getScore
- **Edge cases:** The instruction requests a “SCORE: …” textual format, while the node enforces JSON output. This can produce parsing/shape inconsistencies unless the model outputs JSON with `SCORE` and `FEEDBACK`.

#### G2) getScore
- **Type / role:** `n8n-nodes-base.set` — extracts `SCORE` and `FEEDBACK` from the evaluation response.
- **Config choices:**
  - `SCORE = {{ $json.output[0].content[0].text.SCORE }}`
  - `FEEDBACK = {{ $json.output[0].content[0].text.FEEDBACK }}`
- **Connections:**  
  - **Out:** low quality ?
- **Edge cases:** This path is highly dependent on the exact response structure; if evaluate_quality returns a different JSON layout, these expressions fail.

#### G3) low quality ?
- **Type / role:** `n8n-nodes-base.if` — decides retry vs accept-and-cache.
- **Config choices:** condition:
  - `{{ $json.SCORE < 0.7 && $('current_iteration').item.json.current_iterration >= $('LangCache Config').item.json.max_iterations }}`
- **Important issues:**
  - Uses `current_iterration` (typo) while the field created is `current_iteration`. This likely makes the comparison evaluate as `undefined >= ...` (false), breaking retry gating.
  - Logic reads “low quality AND current >= max” which means it retries when already at/above max; typically you want retry when `current < max`.
- **Connections:**  
  - **True path:** increase iteration (then loops back to retrieval)  
  - **False path:** Save to LangCache
- **Edge cases:** Risk of unintended looping or never retrying depending on expression evaluation.

#### G4) increase iteration
- **Type / role:** `n8n-nodes-base.set` — increments retry counter.
- **Config choices:**  
  - `current_iteration = {{ $('current_iteration').item.json.current_iterration + 1 }}`
- **Issue:** Same typo `current_iterration` prevents incrementing properly.
- **Connections:**  
  - **Out:** current_iteration (back into retrieval path)

#### G5) Save to LangCache
- **Type / role:** `n8n-nodes-base.httpRequest` — saves prompt/response pair to LangCache entries.
- **Config choices:**
  - **Method:** POST
  - **URL:** `{{baseUrl}}/v1/caches/{{cacheId}}/entries`
  - **Body:**  
    - `prompt`: current `question`  
    - `response`: `{{ $('search_node1').item.json.output }}`
  - **Auth:** HTTP Bearer
- **Connections:**  
  - **Out:** Loop Over Items (continues batch loop)
- **Edge cases:** If response is `no info found`, you may cache unhelpful answers unless you explicitly gate on that. Also susceptible to auth/timeouts.

**Sticky Note coverage:**  
- **Sticky Note5:** Quality evaluation score threshold (≥0.7 accept)  
- **Sticky Note6:** Retry control via max_iterations  
- **Sticky Note7:** Save to cache only high-quality answers (intended; current logic may not fully enforce this due to issues above)

---

### Block H — Aggregation + Final Response Synthesis
**Overview:** Collects all per-question results and produces a single user-facing answer.  
**Nodes involved:** Aggregate, synthesize_response_node, Simple Memory2, Sticky Note8

#### H1) Aggregate
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates all item data for synthesis.
- **Config choices:** `aggregateAllItemData`
- **Connections:**  
  - **In:** Loop Over Items  
  - **Out:** synthesize_response_node
- **Edge cases:** If loop outputs inconsistent item shapes (cache hit vs miss), aggregation may include mixed schemas.

#### H2) Simple Memory2
- **Type / role:** memory for final response agent.
- **Config choices:** same sessionKey, window length 10.
- **Connections:**  
  - **AI memory out:** synthesize_response_node

#### H3) synthesize_response_node
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — combines gathered info into final response.
- **Config choices:**
  - **Text input:** original query + `{{ $json.data.toJsonString() }}`
  - **System message:** instructs to produce a coherent support answer; if insufficient info, return an apology fallback sentence.
  - **Has output parser:** enabled (no explicit parser node connected here)
- **Connections:**  
  - **In:** Aggregate (main), OpenAI Chat Model (ai_languageModel), Simple Memory2 (ai_memory)
- **Edge cases:** If `$json.data` doesn’t exist (aggregate output differs), `.toJsonString()` may fail. Also may over-apologize if inputs are sparse.

**Sticky Note coverage:**  
- **Sticky Note8:** “## Generate the respoonse”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | LangCache Config |  |
| LangCache Config | n8n-nodes-base.set | Central config for LangCache + retry params | When chat message received | decompose_query | #### Configuration (Edit First) / Update in **LangCache Config**: / - `langcacheBaseUrl` / - `langcacheCacheId` / - `similarityThreshold` (default `0.75`) / - `max_iterations` (default `2`) |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for decomposition | When chat message received (sessionId) | decompose_query (ai_memory) | ## Query Decomposition / Splits complex user input into focused questions to improve retrieval and caching. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Shared LLM for agents | — | decompose_query; search_node1; synthesize_response_node |  |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces `{questions:[...]}` schema | — | decompose_query (ai_outputParser) | ## Query Decomposition / Splits complex user input into focused questions to improve retrieval and caching. |
| decompose_query | @n8n/n8n-nodes-langchain.agent | Decompose query into cacheable sub-questions | LangCache Config; OpenAI Chat Model; Simple Memory; Structured Output Parser | Split Out | ## Query Decomposition / Splits complex user input into focused questions to improve retrieval and caching. |
| Split Out | n8n-nodes-base.splitOut | Split questions array into items | decompose_query | Loop Over Items |  |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate sub-questions | Split Out; Save to LangCache; Is Cache Hit? (hit loop) | Aggregate; Search LangCache | #### Cache-First Strategy / Each question is checked in **LangCache** first. / - Hit → reuse answer / - Miss → search Redis / Reduces latency and API cost. |
| Search LangCache | n8n-nodes-base.httpRequest | LangCache semantic search | Loop Over Items | Is Cache Hit?; current_iteration | #### Cache-First Strategy / Each question is checked in **LangCache** first. / - Hit → reuse answer / - Miss → search Redis / Reduces latency and API cost. |
| Is Cache Hit? | n8n-nodes-base.if | Route hit vs miss | Search LangCache | Loop Over Items (hit); current_iteration (miss) | #### Cache-First Strategy / Each question is checked in **LangCache** first. / - Hit → reuse answer / - Miss → search Redis / Reduces latency and API cost. |
| current_iteration | n8n-nodes-base.set | Initialize retry counter | Is Cache Hit?; Search LangCache; increase iteration | search_node1 |  |
| Redis Vector Store2 | @n8n/n8n-nodes-langchain.vectorStoreRedis | Retrieval tool (Redis vector search) | Embeddings OpenAI (ai_embedding) | search_node1 (ai_tool) | #### Redis Vector Retrieval / Runs only on cache miss. / Uses embeddings to retrieve relevant knowledge from Redis. |
| Embeddings OpenAI | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Query embeddings for retrieval | — | Redis Vector Store2 | #### Redis Vector Retrieval / Runs only on cache miss. / Uses embeddings to retrieve relevant knowledge from Redis. |
| Simple Memory1 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for retrieval agent | When chat message received (sessionId) | search_node1 (ai_memory) | #### Redis Vector Retrieval / Runs only on cache miss. / Uses embeddings to retrieve relevant knowledge from Redis. |
| search_node1 | @n8n/n8n-nodes-langchain.agent | Answer sub-question from KB only | current_iteration; OpenAI Chat Model; Redis Vector Store2; Simple Memory1 | evaluate_quality | #### Redis Vector Retrieval / Runs only on cache miss. / Uses embeddings to retrieve relevant knowledge from Redis. |
| evaluate_quality | @n8n/n8n-nodes-langchain.openAi | Score answer quality | search_node1 | getScore | ## Quality Evaluation / Each answer is scored (`0.0 – 1.0`). / - ≥ `0.7` → accept / - < `0.7` → retry if allowed |
| getScore | n8n-nodes-base.set | Extract SCORE/FEEDBACK | evaluate_quality | low quality ? | ## Quality Evaluation / Each answer is scored (`0.0 – 1.0`). / - ≥ `0.7` → accept / - < `0.7` → retry if allowed |
| low quality ? | n8n-nodes-base.if | Retry gate vs accept | getScore | increase iteration; Save to LangCache | ## Retry Control / Retries are limited by `max_iterations` to avoid loops and high cost. |
| increase iteration | n8n-nodes-base.set | Increment retry iteration | low quality ? | current_iteration | ## Retry Control / Retries are limited by `max_iterations` to avoid loops and high cost. |
| Save to LangCache | n8n-nodes-base.httpRequest | Save high-quality answers to cache | low quality ? | Loop Over Items | ## ## Save to Cache / Only high-quality answers are saved to **LangCache** for future reuse. |
| Aggregate | n8n-nodes-base.aggregate | Collect results for final response | Loop Over Items | synthesize_response_node | ## Generate the respoonse |
| Simple Memory2 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for synthesis | When chat message received (sessionId) | synthesize_response_node (ai_memory) | ## Generate the respoonse |
| synthesize_response_node | @n8n/n8n-nodes-langchain.agent | Produce final customer response | Aggregate; OpenAI Chat Model; Simple Memory2 | — | ## Generate the respoonse |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled KB ingestion entry | — | example Data | ## Prepare the Knowledge Base -  Example Data |
| example Data | n8n-nodes-base.set | Example KB documents | Schedule Trigger | Redis Vector Store | ## Prepare the Knowledge Base -  Example Data |
| Default Data Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Build Document objects | — | Redis Vector Store (ai_document) |  |
| Embeddings OpenAI1 | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Doc embeddings for ingestion | — | Redis Vector Store (ai_embedding) |  |
| Redis Vector Store | @n8n/n8n-nodes-langchain.vectorStoreRedis | Insert KB docs into Redis index | example Data; Default Data Loader; Embeddings OpenAI1 | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | # Customer support RAG workflow: / ## Workflow Overview / Cache-first **RAG** workflow for customer support. / **Flow:** / Chat → Decompose → Cache → Redis Search → Quality Check → Cache → Respond / **Goals:** Fast, accurate, no hallucinations, cost-controlled. / (full note content continues) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | #### Configuration (Edit First) / Update in **LangCache Config**: / - `langcacheBaseUrl` / - `langcacheCacheId` / - `similarityThreshold` (default `0.75`) / - `max_iterations` (default `2`) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Query Decomposition / Splits complex user input into focused questions to improve retrieval and caching. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | #### Cache-First Strategy / Each question is checked in **LangCache** first. / - Hit → reuse answer / - Miss → search Redis / Reduces latency and API cost. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | #### Redis Vector Retrieval / Runs only on cache miss. / Uses embeddings to retrieve relevant knowledge from Redis. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Quality Evaluation / Each answer is scored (`0.0 – 1.0`). / - ≥ `0.7` → accept / - < `0.7` → retry if allowed |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Retry Control / Retries are limited by `max_iterations` to avoid loops and high cost. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## ## Save to Cache / Only high-quality answers are saved to **LangCache** for future reuse. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Generate the respoonse |
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## Prepare the Knowledge Base -  Example Data |

---

## 4. Reproducing the Workflow from Scratch

### A) Create credentials (required)
1. **OpenAI API credential**
   - Create an OpenAI credential in n8n.
   - Ensure it can access:
     - Chat model: `gpt-4.1-mini` (or your chosen model)
     - Embeddings model (default for the embeddings nodes).
2. **Redis credential**
   - Create Redis connection credential (host/port/password/TLS as required).
   - Ensure Redis supports vector search (Redis Stack / RediSearch with vector).
3. **HTTP Bearer credential for LangCache**
   - Create an “HTTP Bearer Auth” credential.
   - Paste LangCache API token.

### B) Build the scheduled KB ingestion branch (optional but included here)
1. Add **Schedule Trigger** (Schedule Trigger node).
2. Add **Set** node named `example Data`
   - Create field `raw_docs` as an *array of strings* (your KB entries).
3. Add **Default Data Loader** (LangChain Document Default Data Loader).
4. Add **Embeddings OpenAI** node named `Embeddings OpenAI1` (OpenAI embeddings) and select your OpenAI credential.
5. Add **Redis Vector Store** node
   - Mode: **insert**
   - Redis index: choose/create something like `kb-3accd7ed`
   - Select Redis credential.
6. Wire:
   - Schedule Trigger → example Data → Redis Vector Store (main)
   - Default Data Loader (ai_document) → Redis Vector Store
   - Embeddings OpenAI1 (ai_embedding) → Redis Vector Store

### C) Build the chat/RAG branch
1. Add **When chat message received** (Chat Trigger).
2. Add **Set** node `LangCache Config` with fields:
   - `langcacheBaseUrl` (e.g. `https://aws-us-east-1.langcache.redis.io`)
   - `langcacheCacheId` (your cache ID)
   - `similarityThreshold` (number, e.g. `0.75`)
   - `max_iterations` (number recommended; template uses string `"2"`)
3. Add **OpenAI Chat Model** (LangChain Chat Model OpenAI)
   - Set model to `gpt-4.1-mini`.
4. Add **Memory Buffer Window** node `Simple Memory`
   - sessionKey: expression using chat sessionId
   - contextWindowLength: 10
5. Add **Structured Output Parser** with schema:
   - Object with required `questions: string[]`
6. Add **Agent** node `decompose_query`
   - Text: chat input
   - System message: decomposition rules
   - Enable structured output parsing by connecting the parser
   - Connect AI language model (OpenAI Chat Model) and AI memory (Simple Memory).
7. Add **Split Out** node
   - Field to split: `output.questions`
   - Destination field: `question`
8. Add **Split In Batches** node `Loop Over Items`
9. Add **HTTP Request** node `Search LangCache`
   - POST to: `{{langcacheBaseUrl}}/v1/caches/{{langcacheCacheId}}/entries/search`
   - Body: `prompt={{$json.question}}`, `similarityThreshold={{similarityThreshold}}`
   - Auth: Bearer credential
   - Consider enabling “Continue on Fail” (template does).
10. Add **IF** node `Is Cache Hit?`
    - Condition: `{{$json.data?.[0]?.similarity >= $('LangCache Config').item.json.similarityThreshold}}`
11. Add **Set** node `current_iteration`
    - `current_iteration = {{$json.current_iteration ?? 1}}`
12. Add **Embeddings OpenAI** node `Embeddings OpenAI` (for retrieval), set OpenAI credential.
13. Add **Redis Vector Store** node `Redis Vector Store2`
    - Mode: **retrieve-as-tool**
    - Redis index: same as ingestion
14. Add **Memory Buffer Window** `Simple Memory1` (sessionKey = sessionId, window=10)
15. Add **Agent** node `search_node1`
    - Text: `{{$json.question}}`
    - System message: “research engine” constraints + `no info found` fallback
    - Connect: OpenAI Chat Model (ai_languageModel), Redis Vector Store2 (ai_tool), Simple Memory1 (ai_memory)
16. Add **OpenAI** node `evaluate_quality`
    - Model: `gpt-4.1-mini`
    - Configure to return JSON object
    - Prompt with original question + research result; ask for fields `SCORE` and `FEEDBACK` in JSON.
17. Add **Set** node `getScore` to map `SCORE` and `FEEDBACK` from the evaluation output.
18. Add **IF** node `low quality ?`
    - Implement intended logic:
      - retry if `SCORE < 0.7` **and** `current_iteration < max_iterations`
19. Add **Set** node `increase iteration`
    - `current_iteration = current_iteration + 1`
20. Add **HTTP Request** node `Save to LangCache`
    - POST to: `{{langcacheBaseUrl}}/v1/caches/{{langcacheCacheId}}/entries`
    - Body: `prompt={{$json.question}}`, `response={{$('search_node1').item.json.output}}`
21. Add **Aggregate** node `Aggregate` (aggregate all item data).
22. Add **Memory Buffer Window** `Simple Memory2` (session-based).
23. Add **Agent** node `synthesize_response_node`
    - Text: original query + aggregated gathered info
    - System: combine Q/A pairs; if insufficient info output the apology message
    - Connect OpenAI Chat Model + Simple Memory2.

### D) Wire the main branch
1. When chat message received → LangCache Config → decompose_query → Split Out → Loop Over Items
2. Loop Over Items → Search LangCache
3. Search LangCache → Is Cache Hit?
4. Is Cache Hit?:
   - **True** → Loop Over Items (continue)
   - **False** → current_iteration → search_node1 → evaluate_quality → getScore → low quality ?
5. low quality ?:
   - **Retry** → increase iteration → current_iteration (back to search_node1)
   - **Accept** → Save to LangCache → Loop Over Items
6. Loop Over Items (other output) → Aggregate → synthesize_response_node

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Customer support RAG workflow… Chat → Decompose → Cache → Redis Search → Quality Check → Cache → Respond” | From Sticky Note (overall workflow explanation on canvas) |
| Configuration reminders: set LangCache base URL, cache ID, similarity threshold, max iterations | From Sticky Note1 |
| Design intent: cache-first to reduce latency and API cost; Redis retrieval runs on cache miss | From Sticky Note3 & Sticky Note4 |
| Important implementation caveat: retry logic currently appears broken due to `current_iterration` typo and inverted comparison; fix to `current_iteration < max_iterations` | Derived from node expressions in `low quality ?` and `increase iteration` |
| Disclaimer (provided by user): “Le texte fourni provient exclusivement…” | User-provided compliance disclaimer (non-node content) |