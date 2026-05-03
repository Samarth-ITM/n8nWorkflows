Build a WordPress RAG chatbot with OpenAI, Qdrant or MongoDB

https://n8nworkflows.xyz/workflows/build-a-wordpress-rag-chatbot-with-openai--qdrant-or-mongodb-13291


# Build a WordPress RAG chatbot with OpenAI, Qdrant or MongoDB

## 1. Workflow Overview

**Purpose:** This n8n workflow implements a WordPress-powered RAG chatbot. It supports:
- **Indexing WordPress posts** into a **Qdrant** vector collection (for retrieval).
- **Serving chat requests via Webhook**, routing intents (smalltalk vs knowledge lookup vs skills/profile via MongoDB Atlas vectors), retrieving context from Qdrant or MongoDB Atlas, generating answers with OpenAI, and finally **logging requests** to Google Cloud Storage (with hashed IP).

**Typical use cases**
- Embedding a chatbot on a WordPress site to answer questions grounded in site content.
- Adding a “skills/profile” retrieval layer from MongoDB Atlas vectors (e.g., resume/profile + skills KB).
- Capturing anonymized analytics logs (IP hashed) to GCS.

### 1.1 Indexing Block (WP → Documents → Qdrant)
Manual trigger fetches WP posts, prepares documents, splits text, embeds, and stores vectors in Qdrant.

### 1.2 Chat Ingress & Intent Routing (Webhook → Parse → OpenAI Router → Switch)
Webhook receives a chat payload, parses it, uses an OpenAI-based router to classify intent, and branches via a Switch.

### 1.3 Smalltalk Response Path
For smalltalk intents, generate a lightweight response and return to webhook.

### 1.4 Qdrant RAG Path (two variants)
For knowledge intents, run Qdrant similarity search with OpenAI embeddings + Cohere reranking, build context, and generate a structured answer via an AI Agent.

### 1.5 MongoDB Atlas “Skills/Profile” RAG Path
For a specific intent, query MongoDB Atlas Vector collections (SKILLS then PROFILE), run an AI Agent, and post-process.

### 1.6 Response Post-processing & Delivery + Logging (Respond → Hash IP → Date/Name → GCS)
All response paths converge to Respond to Webhook, then hash IP and upload a structured log object to Google Cloud Storage.

---

## 2. Block-by-Block Analysis

### Block 2.1 — WordPress Indexing to Qdrant
**Overview:** Pulls all WordPress posts on demand, transforms them into LangChain documents, splits content, generates embeddings, and writes them to a Qdrant collection.

**Nodes involved:**
- Manual Indexing
- WP Get all post
- Prepare Documents for Indexing
- Qdrant Vector Store1
- Embeddings OpenAI
- Default Data Loader
- Recursive Character Text Splitter

#### Manual Indexing
- **Type/role:** Manual Trigger; starts indexing run.
- **Configuration:** Default manual trigger.
- **Inputs/outputs:** No input; outputs into **WP Get all post**.
- **Failure/edge cases:** None (manual).

#### WP Get all post
- **Type/role:** WordPress node; retrieves posts.
- **Configuration choices (expected):**
  - Credentials: WordPress (likely Application Password or OAuth depending on setup).
  - Operation should be “Get Many” / “Get All posts”.
- **Inputs/outputs:** Input from Manual Indexing → output into **Prepare Documents for Indexing**.
- **Edge cases/failures:**
  - WP auth failure, pagination limits, large sites timing out.
  - HTML content may need stripping/sanitization.

#### Prepare Documents for Indexing
- **Type/role:** Code node; converts WP post records into document objects for embedding/indexing.
- **Configuration (expected):**
  - Build a list of “documents” containing:
    - `pageContent`: post title + cleaned content/excerpt
    - `metadata`: `{ id, slug, url, date, language, tags/categories }`
- **Connections:** From WP Get all post → to **Qdrant Vector Store1** (main input).
- **Edge cases:**
  - Missing fields (empty content), HTML entities, very long posts.
  - Ensure stable unique IDs in metadata for upsert behavior.

#### Qdrant Vector Store1
- **Type/role:** LangChain Qdrant Vector Store; used here for **index/upsert**.
- **Configuration (expected):**
  - Qdrant endpoint + API key (if cloud).
  - Collection name (must be set).
  - Upsert mode / id strategy (often metadata id).
- **Key connections:**
  - **Main input:** documents from Prepare Documents for Indexing.
  - **ai_embedding input:** from **Embeddings OpenAI**.
  - **ai_document input:** from **Default Data Loader**.
- **Edge cases:**
  - Collection missing/mismatched vector size vs embedding model.
  - Network timeouts; Qdrant rate limits.

#### Embeddings OpenAI
- **Type/role:** OpenAI embeddings provider.
- **Configuration (expected):**
  - Credentials: OpenAI API key.
  - Model: e.g. `text-embedding-3-small` or `text-embedding-3-large`.
- **Outputs:** Connected to **Qdrant Vector Store1** via `ai_embedding`.
- **Edge cases:**
  - Large batch sizes, token limits per document chunk.

#### Default Data Loader
- **Type/role:** LangChain document loader wrapper; prepares/supplies documents to the vector store node.
- **Configuration (expected):**
  - Pass-through of incoming items to LangChain document format.
- **Inputs/outputs:**
  - Receives splitter via `ai_textSplitter` from Recursive Character Text Splitter.
  - Outputs documents via `ai_document` to Qdrant Vector Store1.
- **Edge cases:** If the upstream Code node does not output expected fields, loader may produce empty docs.

#### Recursive Character Text Splitter
- **Type/role:** Text splitting for chunking documents prior to embedding.
- **Configuration (expected):**
  - `chunkSize`, `chunkOverlap` tuned to your content (e.g., 800/100).
- **Outputs:** `ai_textSplitter` → Default Data Loader.
- **Edge cases:** Too small chunks reduce coherence; too large chunks increase cost and retrieval noise.

---

### Block 2.2 — Chat Ingress, Parsing, Intent Routing
**Overview:** Accepts inbound chat requests via Webhook, parses payload into normalized fields, classifies intent with OpenAI, then branches with Switch.

**Nodes involved:**
- Webhook
- Code-Request_Parsing
- OpenAI Intent Router
- Switch

#### Webhook
- **Type/role:** Webhook trigger; entry point for chatbot requests.
- **Configuration (expected):**
  - HTTP method (often POST).
  - Path defined in node (not shown in JSON excerpt); `webhookId` present.
  - Response handled later by Respond to Webhook node.
- **Output:** → Code-Request_Parsing.
- **Edge cases:**
  - Invalid JSON body; missing fields (message, session/user).
  - Public endpoint security (need secret, signature, or auth).

#### Code-Request_Parsing
- **Type/role:** Code node; normalizes inbound request into a consistent schema for downstream AI nodes.
- **Configuration (expected):**
  - Extract user query text, language, chat history, user identifiers, IP, user-agent.
  - Provide fields used by the router and response generator.
- **Output:** → OpenAI Intent Router.
- **Edge cases:** Undefined nested fields cause runtime errors if not guarded.

#### OpenAI Intent Router
- **Type/role:** LangChain OpenAI node used for classification/routing (prompt-based intent detection).
- **Configuration (expected):**
  - OpenAI credentials + model (fast chat model).
  - Prompt instructing output as an intent label.
- **Output:** → Switch.
- **Edge cases:** Non-deterministic labels; enforce strict output (few-shot + constrained options).

#### Switch
- **Type/role:** Switch node; routes flow based on intent.
- **Connections:** 5 outputs configured:
  1) → SmallTalk Response  
  2) → SmallTalk Response  
  3) → Qdrant Search (Collection_name) *(node id d029…)*  
  4) → Qdrant Search (Collection_name) *(node id b242…)*  
  5) → MongoDB Atlas Vector SKILLS
- **Edge cases:**
  - If intent doesn’t match any rule, workflow may stop (no default route shown).
  - Two different downstream nodes share the same display name “Qdrant Search (Collection_name)”—this can confuse maintenance.

---

### Block 2.3 — Smalltalk Response Path
**Overview:** Generates a conversational response without retrieval, then returns it to the webhook response pipeline.

**Nodes involved:**
- SmallTalk Response
- Code-Smalltalk_response
- Respond to Webhook

#### SmallTalk Response
- **Type/role:** OpenAI (LangChain OpenAI node) for direct response.
- **Input:** From Switch (two separate cases point here).
- **Output:** → Code-Smalltalk_response.
- **Edge cases:** Should be constrained to avoid hallucinating “site facts” since no RAG context is used.

#### Code-Smalltalk_response
- **Type/role:** Code node; formats smalltalk output into response schema expected by Respond to Webhook.
- **Output:** → Respond to Webhook.
- **Edge cases:** Ensure consistent JSON structure with other paths.

#### Respond to Webhook
- **Type/role:** Sends HTTP response back to caller.
- **Input:** From Code-Smalltalk_response OR Code: remove followup (other paths).
- **Output:** → Crypto: IP Hash (logging continues after responding).
- **Edge cases:** If configured to “Respond immediately” vs “Last node”, mismatches can cause hanging requests or double responses.

---

### Block 2.4 — Qdrant RAG Path (Variant A)
**Overview:** Runs a Qdrant retrieval with embeddings + Cohere reranking, builds a context block, generates a structured answer via an AI Agent, then parses and post-processes before responding.

**Nodes involved:**
- Qdrant Search (Collection_name) *(id d029…)*
- Embeddings OpenAI1
- Reranker Cohere
- Build context from Qdrant results1
- AI Agent
- OpenAI Chat Model
- Structured Output Parser
- Code-Response_Parsing
- Code: remove followup
- Respond to Webhook

#### Qdrant Search (Collection_name) — (id d029…)
- **Type/role:** Qdrant vector store node used for **search**.
- **Configuration (expected):**
  - Collection name must match indexing collection.
  - `topK` / score threshold.
- **Connections:**
  - Main input from Switch (intent).
  - `ai_embedding` from Embeddings OpenAI1.
  - `ai_reranker` from Reranker Cohere (note: reranker node is wired into this Qdrant node).
  - Main output → Build context from Qdrant results1.
- **Edge cases:** Empty results; low-quality matches; schema mismatch in stored metadata.

#### Embeddings OpenAI1
- **Type/role:** Embeddings for query vectorization during search.
- **Output:** `ai_embedding` → Qdrant Search (id d029…).

#### Reranker Cohere
- **Type/role:** Cohere reranker to improve ordering of retrieved chunks.
- **Output:** `ai_reranker` → Qdrant Search (id d029…).
- **Edge cases:** Requires Cohere API key; additional latency/cost; reranker expects text fields.

#### Build context from Qdrant results1
- **Type/role:** Code node; concatenates retrieved chunks into a prompt-ready context.
- **Output:** → AI Agent.
- **Edge cases:** Context too long for model; should cap by tokens/chars and include source metadata.

#### AI Agent
- **Type/role:** LangChain Agent; generates final answer using provided context and tools (here: model + structured parser).
- **Connections:**
  - Main input from Build context from Qdrant results1.
  - `ai_languageModel` from OpenAI Chat Model.
  - `ai_outputParser` from Structured Output Parser.
- **Edge cases:** If parser schema is strict and model deviates, parsing fails.

#### OpenAI Chat Model
- **Type/role:** Chat LLM used by AI Agent.
- **Configuration (expected):**
  - Model (e.g., GPT-4o-mini / GPT-4.1-mini), temperature, max tokens.
- **Output:** `ai_languageModel` → AI Agent.

#### Structured Output Parser
- **Type/role:** Enforces structured JSON output from AI Agent (e.g., `{ answer, sources, followup }`).
- **Output:** `ai_outputParser` → AI Agent.
- **Edge cases:** Schema mismatch, invalid JSON from model.

#### Code-Response_Parsing
- **Type/role:** Code node; converts agent output into final API response format.
- **Output:** → Code: remove followup.

#### Code: remove followup
- **Type/role:** Code node; removes/filters follow-up questions or cleans response payload.
- **Inputs:** From Code-Response_Parsing OR url_en OR Code in JavaScript (skills path).
- **Output:** → Respond to Webhook.

---

### Block 2.5 — Qdrant RAG Path (Variant B)
**Overview:** Similar to Variant A, but uses a different Qdrant search node and a different agent chain (AI Agent3), plus a URL normalization step.

**Nodes involved:**
- Qdrant Search (Collection_name) *(id b242…)*
- Embeddings OpenAI3
- Reranker Cohere3
- Build context from Qdrant results
- AI Agent3
- OpenAI Chat Model3
- Structured Output Parser3
- url_en
- Code: remove followup
- Respond to Webhook

#### Qdrant Search (Collection_name) — (id b242…)
- **Type/role:** Qdrant search (second instance).
- **Connections:**
  - Main input from Switch.
  - `ai_embedding` from Embeddings OpenAI3.
  - `ai_reranker` from Reranker Cohere3.
  - Output → Build context from Qdrant results.
- **Maintenance note:** Same display name as the other Qdrant search node; rename one to avoid confusion.

#### Build context from Qdrant results
- **Role:** Code node to assemble context.
- **Output:** → AI Agent3.

#### AI Agent3 + OpenAI Chat Model3 + Structured Output Parser3
- **Role:** Generates structured response similarly to Variant A.
- **Output:** AI Agent3 → url_en.

#### url_en
- **Type/role:** Code node; likely normalizes/rewrites URLs to English or ensures consistent URL formatting in sources.
- **Output:** → Code: remove followup.

---

### Block 2.6 — MongoDB Atlas “Skills/Profile” Retrieval Path
**Overview:** Routes a specific intent to MongoDB Atlas vector search (SKILLS then PROFILE), runs an AI Agent to respond, then post-processes and returns.

**Nodes involved:**
- MongoDB Atlas Vector SKILLS
- Embeddings OpenAI4
- MongoDB Atlas Vector PROFILE
- Embeddings OpenAI5
- Code in JavaScript3
- AI Agent - SKILLS
- OpenAI Chat Model1
- Code in JavaScript
- Code: remove followup
- Respond to Webhook

#### MongoDB Atlas Vector SKILLS
- **Type/role:** MongoDB Atlas Vector Store node; performs vector search against a “SKILLS” collection/index.
- **Input:** From Switch.
- **Embedding:** From Embeddings OpenAI4 via `ai_embedding`.
- **Output:** → MongoDB Atlas Vector PROFILE (chained).
- **Edge cases:**
  - Atlas Search index misconfiguration (vector index, dims).
  - Credentials/connection string issues, IP allowlist.

#### MongoDB Atlas Vector PROFILE
- **Type/role:** Second Atlas vector search, likely enriching context with profile documents.
- **Embedding:** From Embeddings OpenAI5.
- **Output:** → Code in JavaScript3.

#### Code in JavaScript3
- **Type/role:** Code node; likely combines SKILLS + PROFILE results into a context payload for the agent.
- **Output:** → AI Agent - SKILLS.

#### AI Agent - SKILLS
- **Type/role:** Agent that generates final response using OpenAI Chat Model1.
- **Language model:** OpenAI Chat Model1 connected via `ai_languageModel`.
- **Output:** → Code in JavaScript.

#### Code in JavaScript
- **Type/role:** Code node; formats agent response to match the unified response schema.
- **Output:** → Code: remove followup.

#### OpenAI Chat Model1
- **Type/role:** Chat model for skills/profile agent.

---

### Block 2.7 — Response Delivery + Logging to GCS (post-response)
**Overview:** After responding to the webhook, the workflow hashes the request IP, builds a dated object name and JSON log, then uploads it to Google Cloud Storage.

**Nodes involved:**
- Respond to Webhook
- Crypto: IP Hash
- Set: date
- set: ObjectName_Date/Folder & data
- Code: parsing
- GCS: Object_Log

#### Crypto: IP Hash
- **Type/role:** Crypto node; hashes client IP for anonymized logging.
- **Input:** From Respond to Webhook.
- **Output:** → Set: date.
- **Edge cases:** If IP is missing, hashing may error unless handled.

#### Set: date
- **Type/role:** Set node; adds timestamp/date fields.
- **Output:** → set: ObjectName_Date/Folder & data.

#### set: ObjectName_Date/Folder & data
- **Type/role:** Set node; constructs GCS object name (folder/date-based) and prepares data payload.
- **Output:** → Code: parsing.

#### Code: parsing
- **Type/role:** Code node; likely serializes payload to JSON/binary and maps to GCS node expected fields.
- **Output:** → GCS: Object_Log.

#### GCS: Object_Log
- **Type/role:** Google Cloud Storage node; uploads the log object.
- **Configuration (expected):**
  - GCP credentials (service account JSON or workload identity).
  - Bucket name, object name, content type.
- **Edge cases:** Permission denied, bucket not found, object naming collisions.

---

## 3. Summary Table

> Sticky notes exist but all have **empty content** in the provided JSON, so the “Sticky Note” column is blank for all nodes.

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Indexing | manualTrigger | Start WP indexing manually | — | WP Get all post |  |
| WP Get all post | wordpress | Fetch WordPress posts | Manual Indexing | Prepare Documents for Indexing |  |
| Prepare Documents for Indexing | code | Transform WP posts into documents | WP Get all post | Qdrant Vector Store1 |  |
| Embeddings OpenAI | embeddingsOpenAi | Embeddings for indexing | — | Qdrant Vector Store1 (ai_embedding) |  |
| Recursive Character Text Splitter | textSplitterRecursiveCharacterTextSplitter | Chunk documents | — | Default Data Loader (ai_textSplitter) |  |
| Default Data Loader | documentDefaultDataLoader | Convert items to LangChain documents | — | Qdrant Vector Store1 (ai_document) |  |
| Qdrant Vector Store1 | vectorStoreQdrant | Upsert documents into Qdrant | Prepare Documents for Indexing | — |  |
| Webhook | webhook | Chat entrypoint | — | Code-Request_Parsing |  |
| Code-Request_Parsing | code | Normalize inbound request | Webhook | OpenAI Intent Router |  |
| OpenAI Intent Router | openAi | Intent classification | Code-Request_Parsing | Switch |  |
| Switch | switch | Route by intent | OpenAI Intent Router | SmallTalk Response / Qdrant Search (x2) / MongoDB Atlas Vector SKILLS |  |
| SmallTalk Response | openAi | Smalltalk generation | Switch | Code-Smalltalk_response |  |
| Code-Smalltalk_response | code | Format smalltalk output | SmallTalk Response | Respond to Webhook |  |
| Qdrant Search (Collection_name) *(id d029…)* | vectorStoreQdrant | Qdrant retrieval (variant A) | Switch | Build context from Qdrant results1 |  |
| Embeddings OpenAI1 | embeddingsOpenAi | Query embeddings (variant A) | — | Qdrant Search (id d029…) (ai_embedding) |  |
| Reranker Cohere | rerankerCohere | Rerank retrieved chunks (A) | — | Qdrant Search (id d029…) (ai_reranker) |  |
| Build context from Qdrant results1 | code | Assemble context (A) | Qdrant Search (id d029…) | AI Agent |  |
| OpenAI Chat Model | lmChatOpenAi | LLM for Agent (A) | — | AI Agent (ai_languageModel) |  |
| Structured Output Parser | outputParserStructured | Enforce structured output (A) | — | AI Agent (ai_outputParser) |  |
| AI Agent | agent | Answer generation (A) | Build context from Qdrant results1 | Code-Response_Parsing |  |
| Code-Response_Parsing | code | Format agent output (A) | AI Agent | Code: remove followup |  |
| Qdrant Search (Collection_name) *(id b242…)* | vectorStoreQdrant | Qdrant retrieval (variant B) | Switch | Build context from Qdrant results |  |
| Embeddings OpenAI3 | embeddingsOpenAi | Query embeddings (B) | — | Qdrant Search (id b242…) (ai_embedding) |  |
| Reranker Cohere3 | rerankerCohere | Rerank retrieved chunks (B) | — | Qdrant Search (id b242…) (ai_reranker) |  |
| Build context from Qdrant results | code | Assemble context (B) | Qdrant Search (id b242…) | AI Agent3 |  |
| OpenAI Chat Model3 | lmChatOpenAi | LLM for Agent (B) | — | AI Agent3 (ai_languageModel) |  |
| Structured Output Parser3 | outputParserStructured | Enforce structured output (B) | — | AI Agent3 (ai_outputParser) |  |
| AI Agent3 | agent | Answer generation (B) | Build context from Qdrant results | url_en |  |
| url_en | code | URL normalization/translation (B) | AI Agent3 | Code: remove followup |  |
| MongoDB Atlas Vector SKILLS | vectorStoreMongoDBAtlas | Vector retrieval: skills | Switch | MongoDB Atlas Vector PROFILE |  |
| Embeddings OpenAI4 | embeddingsOpenAi | Query embeddings for skills | — | MongoDB Atlas Vector SKILLS (ai_embedding) |  |
| MongoDB Atlas Vector PROFILE | vectorStoreMongoDBAtlas | Vector retrieval: profile | MongoDB Atlas Vector SKILLS | Code in JavaScript3 |  |
| Embeddings OpenAI5 | embeddingsOpenAi | Query embeddings for profile | — | MongoDB Atlas Vector PROFILE (ai_embedding) |  |
| Code in JavaScript3 | code | Merge skills/profile results | MongoDB Atlas Vector PROFILE | AI Agent - SKILLS |  |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for skills agent | — | AI Agent - SKILLS (ai_languageModel) |  |
| AI Agent - SKILLS | agent | Answer generation (skills/profile) | Code in JavaScript3 | Code in JavaScript |  |
| Code in JavaScript | code | Format skills response | AI Agent - SKILLS | Code: remove followup |  |
| Code: remove followup | code | Remove followups / unify payload | Code-Response_Parsing / url_en / Code in JavaScript | Respond to Webhook |  |
| Respond to Webhook | respondToWebhook | Return HTTP response | Code: remove followup / Code-Smalltalk_response | Crypto: IP Hash |  |
| Crypto: IP Hash | crypto | Hash IP for logging | Respond to Webhook | Set: date |  |
| Set: date | set | Add timestamp fields | Crypto: IP Hash | set: ObjectName_Date/Folder & data |  |
| set: ObjectName_Date/Folder & data | set | Build object name + log payload | Set: date | Code: parsing |  |
| Code: parsing | code | Serialize/prepare for GCS upload | set: ObjectName_Date/Folder & data | GCS: Object_Log |  |
| GCS: Object_Log | googleCloudStorage | Upload log object | Code: parsing | — |  |
| Sticky Note / Sticky Note1..19 | stickyNote | Canvas annotation (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named: `WP-AI_Chatbot-Template`.

### A. Build the Indexing pipeline (WordPress → Qdrant)
2) Add **Manual Trigger** node named **Manual Indexing**.  
3) Add **WordPress** node named **WP Get all post**:
   - Set WordPress credentials.
   - Operation: fetch all posts (ensure pagination/all items).
4) Connect **Manual Indexing → WP Get all post**.
5) Add **Code** node named **Prepare Documents for Indexing**:
   - Transform each WP post into fields suitable for LangChain docs:
     - `pageContent` (plain text)
     - `metadata` (id/slug/url/title/date)
6) Connect **WP Get all post → Prepare Documents for Indexing**.
7) Add **Recursive Character Text Splitter** node:
   - Configure `chunkSize` and `chunkOverlap` (example: 800 and 100).
8) Add **Default Data Loader** node:
   - Connect **Recursive Character Text Splitter** to **Default Data Loader** using the **AI Text Splitter** connection type.
9) Add **Embeddings OpenAI** node:
   - Configure OpenAI credentials.
   - Choose an embeddings model (must match Qdrant vector size expectations).
10) Add **Qdrant Vector Store** node named **Qdrant Vector Store1**:
   - Configure Qdrant credentials (URL + API key if needed).
   - Set the **Collection name** (create it if needed; ensure correct vector dimension).
   - Set mode to **upsert/index documents**.
11) Connect:
   - **Prepare Documents for Indexing (main) → Qdrant Vector Store1 (main)**
   - **Embeddings OpenAI (ai_embedding) → Qdrant Vector Store1**
   - **Default Data Loader (ai_document) → Qdrant Vector Store1**

### B. Build the Webhook ingress and intent router
12) Add **Webhook** node named **Webhook**:
   - Method: POST (recommended).
   - Response mode: “Using Respond to Webhook node”.
13) Add **Code** node named **Code-Request_Parsing**:
   - Parse `body.message` (or equivalent), optional `history`, `language`, `ip`.
   - Output normalized fields (e.g., `query`, `sessionId`, `clientIp`).
14) Connect **Webhook → Code-Request_Parsing**.
15) Add **OpenAI** (LangChain OpenAI) node named **OpenAI Intent Router**:
   - Model: fast chat model.
   - Prompt: classify into a fixed set of intents (e.g. `smalltalk`, `knowledge_a`, `knowledge_b`, `skills`).
16) Connect **Code-Request_Parsing → OpenAI Intent Router**.
17) Add **Switch** node named **Switch**:
   - Create **5 rules/outputs** matching your router labels (two can point to smalltalk).
18) Connect **OpenAI Intent Router → Switch**.

### C. Smalltalk path
19) Add **OpenAI** node named **SmallTalk Response** (chat completion):
   - Prompt: produce a friendly short answer; no sources.
20) Add **Code** node named **Code-Smalltalk_response**:
   - Map output to final response JSON (e.g., `{ answer: "...", sources: [] }`).
21) Add **Respond to Webhook** node named **Respond to Webhook**:
   - Return the formatted JSON.
22) Connect **Switch (smalltalk outputs) → SmallTalk Response → Code-Smalltalk_response → Respond to Webhook**.

### D. Qdrant RAG path (Variant A)
23) Add **Embeddings OpenAI** node named **Embeddings OpenAI1** (query embeddings).
24) Add **Cohere Reranker** node named **Reranker Cohere** (configure Cohere API key).
25) Add **Qdrant Vector Store** node named **Qdrant Search (Collection_name)** (Variant A):
   - Set to **search** in your collection.
   - Configure `topK`.
   - Wire embeddings + reranker into this node.
26) Add **Code** node named **Build context from Qdrant results1** to concatenate retrieved chunks into a `context` string and attach sources metadata.
27) Add **OpenAI Chat Model** node named **OpenAI Chat Model**.
28) Add **Structured Output Parser** node named **Structured Output Parser**:
   - Define a schema (answer text + sources + optional followup).
29) Add **AI Agent** node named **AI Agent**:
   - Connect `ai_languageModel` from OpenAI Chat Model.
   - Connect `ai_outputParser` from Structured Output Parser.
30) Add **Code** node named **Code-Response_Parsing** and **Code** node named **Code: remove followup**.
31) Connect:
   - **Switch (intent A) → Qdrant Search (A) → Build context…1 → AI Agent → Code-Response_Parsing → Code: remove followup → Respond to Webhook**
   - Plus AI connections:
     - **Embeddings OpenAI1 → Qdrant Search (A)** (ai_embedding)
     - **Reranker Cohere → Qdrant Search (A)** (ai_reranker)
     - **OpenAI Chat Model → AI Agent** (ai_languageModel)
     - **Structured Output Parser → AI Agent** (ai_outputParser)

### E. Qdrant RAG path (Variant B)
32) Repeat steps similar to A but with separate nodes:
   - **Embeddings OpenAI3**, **Reranker Cohere3**
   - Another **Qdrant Search** node (rename it to avoid confusion, e.g., “Qdrant Search (EN)”)
   - **Build context from Qdrant results**
   - **OpenAI Chat Model3**, **Structured Output Parser3**, **AI Agent3**
   - **url_en** Code node (normalize/translate URLs)
33) Connect:
   - **Switch (intent B) → Qdrant Search (B) → Build context… → AI Agent3 → url_en → Code: remove followup → Respond to Webhook**
   - and the AI side connections as above.

### F. MongoDB Atlas skills/profile path
34) Add **MongoDB Atlas Vector Store** node named **MongoDB Atlas Vector SKILLS**:
   - Configure MongoDB Atlas connection string credentials.
   - Select collection/index for SKILLS vector search.
35) Add **Embeddings OpenAI4** and connect it to SKILLS node via `ai_embedding`.
36) Add **MongoDB Atlas Vector Store** node named **MongoDB Atlas Vector PROFILE** (second search):
   - Configure PROFILE collection/index.
37) Add **Embeddings OpenAI5** and connect it to PROFILE via `ai_embedding`.
38) Add **Code in JavaScript3** to merge results into context.
39) Add **OpenAI Chat Model1** and **AI Agent - SKILLS**; connect model to agent (`ai_languageModel`).
40) Add **Code in JavaScript** to format output, then connect to **Code: remove followup** → **Respond to Webhook**.
41) Connect the branch: **Switch (skills intent) → MongoDB SKILLS → MongoDB PROFILE → Code JS3 → AI Agent - SKILLS → Code JS → Code: remove followup → Respond to Webhook**.

### G. Logging pipeline (after responding)
42) Add **Crypto** node named **Crypto: IP Hash**:
   - Hash algorithm: SHA-256 (recommended).
   - Input: `clientIp` extracted earlier.
43) Add **Set** node **Set: date** to add timestamp fields.
44) Add **Set** node **set: ObjectName_Date/Folder & data** to build:
   - Object path: e.g. `logs/YYYY/MM/DD/<session>-<timestamp>.json`
   - Data: response + intent + hashed IP + query metadata
45) Add **Code** node **Code: parsing** to serialize/prepare for GCS upload (binary or text, depending on GCS node setup).
46) Add **Google Cloud Storage** node **GCS: Object_Log**:
   - Configure GCP credentials (service account).
   - Set bucket and object name expression.
47) Connect **Respond to Webhook → Crypto: IP Hash → Set: date → set: ObjectName… → Code: parsing → GCS: Object_Log**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes are present in the workflow but their `content` fields are empty in the provided JSON. | Applies to all Sticky Note nodes |
| Two different nodes share the same display name “Qdrant Search (Collection_name)”. Rename them (e.g., “Qdrant Search A” / “Qdrant Search B”) to reduce operational risk. | Maintenance/operability note |
| The JSON omits most node parameters (credentials, prompts, schemas, collection names). You must supply these in n8n for the workflow to function. | Applies to OpenAI/Qdrant/MongoDB/GCS/WordPress nodes |