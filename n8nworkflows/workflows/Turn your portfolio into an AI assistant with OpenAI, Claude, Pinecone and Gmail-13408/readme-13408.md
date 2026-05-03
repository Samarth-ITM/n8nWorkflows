Turn your portfolio into an AI assistant with OpenAI, Claude, Pinecone and Gmail

https://n8nworkflows.xyz/workflows/turn-your-portfolio-into-an-ai-assistant-with-openai--claude--pinecone-and-gmail-13408


# Turn your portfolio into an AI assistant with OpenAI, Claude, Pinecone and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow turns a personal portfolio (documents stored in Google Drive) into a recruiter-facing AI assistant using Retrieval-Augmented Generation (RAG). It continuously ingests and indexes portfolio documents into Pinecone, then serves a chat endpoint that answers questions grounded in those documents. If a recruiter asks for a CV/resume, it automatically emails a PDF via Gmail.

**Target use cases:**
- Recruiters asking questions about skills, projects, experience, and technical capabilities
- Automated CV delivery when requested
- Always-up-to-date portfolio knowledge base synced from Google Drive

**Logical blocks (by dependency and function):**
1. **1.1 Ingestion Pipeline (Drive → Chunk → Embed → Pinecone Upsert)**  
   Detects new/updated files in a Drive folder, downloads them, adds metadata, splits content, creates embeddings, and upserts to Pinecone.
2. **1.2 Chat Agent Pipeline (Webhook → Claude Agent → RAG Tooling → Structured Output)**  
   Receives chat requests, uses Claude with memory + a vector tool to retrieve evidence from Pinecone (with Cohere reranking), and outputs structured JSON.
3. **1.3 CV Delivery Branch (cvRequested → Download CV → Gmail → Webhook Response)**  
   If the structured output indicates a CV request, downloads a fixed CV PDF from Drive and emails it to the provided address.
4. **1.4 Non-CV Response Branch (Answer → Webhook Response)**  
   If no CV requested, returns the answer as JSON to the webhook caller.
5. **1.5 Error Handling (Error Trigger → Format Error Data)**  
   Captures failures from any node run and formats error details for downstream alerting.

---

## 2. Block-by-Block Analysis

### 2.1 Ingestion Pipeline (Drive → Pinecone)

**Overview:**  
Watches a specific Google Drive folder for new or updated files. Each file is downloaded, augmented with metadata, chunked into overlapping segments, embedded via OpenAI, and inserted into a Pinecone index for later retrieval.

**Nodes involved:**
- Section 1 Background (Sticky Note)
- File Created Trigger
- File Updated Trigger
- Download File
- Enrich Metadata
- OpenAI Embeddings
- Text Splitter
- Document Loader
- Pinecone Insert

#### Node: Section 1 Background (Sticky Note)
- **Type / role:** Sticky note documentation
- **Configuration:** Describes ingestion purpose: monitor Drive, chunk, embed, upsert to Pinecone
- **Connections:** None
- **Failure modes:** None (non-executable)

#### Node: File Created Trigger
- **Type / role:** Google Drive Trigger; entry point for ingestion on new files
- **Configuration choices:**
  - Event: **fileCreated**
  - Polling: **every minute**
  - Scope: **specificFolder** (folder ID must be set)
  - File type: **all**
- **Key variables/expressions:** Folder ID currently empty; must be provided.
- **Input/Output:**
  - **Output →** Download File
- **Edge cases / failures:**
  - Missing/invalid folder ID → no events or node error
  - Drive credential scopes insufficient → auth failures
  - Polling delay/duplication possible in polling triggers

#### Node: File Updated Trigger
- **Type / role:** Google Drive Trigger; entry point for ingestion on updated files
- **Configuration choices:**
  - Event: **fileUpdated**
  - Polling: **every minute**
  - Scope: **specificFolder** (folder ID must be set)
- **Input/Output:**
  - **Output →** Download File
- **Edge cases / failures:**
  - Same as “File Created Trigger”
  - Frequent updates may cause repeated re-ingestion unless you implement dedupe (not present here)

#### Node: Download File
- **Type / role:** Google Drive node; downloads the changed file as binary for processing
- **Configuration choices:**
  - Operation: **download**
  - File ID: `={{ $json.id }}` (from trigger output)
  - Output filename option: `={{ $json.name }}`
- **Input/Output:**
  - **Input ←** File Created Trigger OR File Updated Trigger
  - **Output →** Enrich Metadata
- **Edge cases / failures:**
  - Deleted/moved files between trigger and download → 404
  - Google native formats (Docs/Sheets) may download in default format; ensure expected content type is supported by downstream loader
  - Large files may hit size/time limits

#### Node: Enrich Metadata
- **Type / role:** Set node; attaches metadata fields to the current item while keeping existing fields
- **Configuration choices:**
  - Adds:
    - `upload_date` = `={{ $now.toISO() }}`
    - `source_filename` = `={{ $json.name }}`
  - Includes other fields: **true**
- **Input/Output:**
  - **Input ←** Download File
  - **Output →** Pinecone Insert (main input)
- **Edge cases / failures:**
  - If `$json.name` missing (unexpected trigger payload), `source_filename` may be empty
  - Metadata is added at item level; ensure downstream document loader/insert preserves it (depends on loader behavior)

#### Node: OpenAI Embeddings
- **Type / role:** LangChain embeddings provider; supplies vector embeddings for Pinecone upsert
- **Configuration choices:**
  - Model: **text-embedding-3-small** (dimension typically 1536)
- **Input/Output connections:**
  - **Embedding output (ai_embedding) →** Pinecone Insert
- **Edge cases / failures:**
  - Wrong API key/organization/project → auth errors
  - Rate limits / timeouts on batch embedding during ingestion bursts
  - Model dimension mismatch with Pinecone index dimension causes insert failures

#### Node: Text Splitter
- **Type / role:** Recursive Character Text Splitter; chunks documents for embeddings
- **Configuration choices:**
  - `chunkSize`: **600**
  - `chunkOverlap`: **100**
- **Input/Output connections:**
  - **Text splitter output (ai_textSplitter) →** Document Loader
- **Edge cases / failures:**
  - Very small documents may produce 1 chunk (fine)
  - Very large documents produce many chunks → cost/rate limit pressure upstream (OpenAI embeddings) and increased Pinecone upsert time

#### Node: Document Loader
- **Type / role:** Default data loader; converts binary content into LangChain Document(s)
- **Configuration choices:**
  - Data type: **binary**
  - Binary mode: **specificField**
  - Binary property: **data** (expects the Google Drive download binary to be in `binary.data`)
- **Input/Output connections:**
  - **Input (ai_textSplitter) ←** Text Splitter
  - **Output (ai_document) →** Pinecone Insert
- **Edge cases / failures:**
  - If Google Drive node stores binary under a different key than `data`, loader will fail
  - Unsupported file types may not parse cleanly (e.g., images without OCR)

#### Node: Pinecone Insert
- **Type / role:** Pinecone Vector Store (LangChain); upserts document chunks + embeddings into an index
- **Configuration choices:**
  - Mode: **insert**
  - Index: **portfolio-docs**
- **Input/Output connections:**
  - **Main input ←** Enrich Metadata (item fields/metadata)
  - **ai_document ←** Document Loader
  - **ai_embedding ←** OpenAI Embeddings
- **Version-specific requirements:**
  - Requires `@n8n/n8n-nodes-langchain.vectorStorePinecone` node (typeVersion ~1.3)
- **Edge cases / failures:**
  - Pinecone index missing / wrong name → errors
  - Dimension mismatch (index vs embedding model) → upsert rejected
  - Namespace/metadata strategy not configured; retrieval quality depends on default behavior
  - Without document IDs/deduping, repeated updates may create duplicates depending on node’s internal ID strategy

---

### 2.2 Chat Agent Pipeline (Webhook → Claude Agent → RAG)

**Overview:**  
Exposes an HTTP POST webhook for recruiter chat queries. A Claude-based agent uses a vector tool to retrieve relevant portfolio evidence from Pinecone (with Cohere reranking) and returns a structured JSON containing an answer and whether a CV was requested.

**Nodes involved:**
- Section 2 Background (Sticky Note)
- Chat Webhook
- Portfolio AI Agent
- Claude Sonnet 4.5
- Chat Memory
- Structured Output Parser
- Portfolio Vector Tool
- Pinecone Retrieval
- OpenAI Embeddings Retrieval
- Cohere Reranker
- Claude Retrieval Model

#### Node: Section 2 Background (Sticky Note)
- **Type / role:** Sticky note documentation
- **Configuration:** Describes AI agent pipeline: webhook-triggered Claude, Pinecone RAG, reranking, CV detection, Gmail delivery
- **Connections:** None

#### Node: Chat Webhook
- **Type / role:** Webhook trigger; entry point for chat requests
- **Configuration choices:**
  - Path: **portfolio-query**
  - Method: **POST**
  - Allowed origins: `*` (CORS open)
  - Response mode: **responseNode** (responses come from Respond to Webhook nodes)
  - On error: **continueRegularOutput** (workflow tries to continue)
- **Expected input payload:**
  - `body.chatInput` (string)
  - `body.sessionId` (string)
  - `body.email` (string; used only if CV is requested)
- **Input/Output:**
  - **Output →** Portfolio AI Agent
- **Edge cases / failures:**
  - Missing `chatInput`/`sessionId` may break expressions in downstream nodes
  - Open CORS is convenient but may be undesirable for public endpoints
  - If no Respond node is reached (logic error), webhook call may hang/timeout

#### Node: Portfolio AI Agent
- **Type / role:** LangChain Agent; orchestrates LLM + tools + memory + output parser
- **Configuration choices:**
  - Input text: `={{ $json.body.chatInput }}`
  - System message: instructs grounded answers using `portfolio_knowledge` tool; sets `cvRequested=true` when asked for CV; requires structured output
  - Output parser: enabled (`hasOutputParser: true`)
- **Connections:**
  - **Main input ←** Chat Webhook
  - **ai_languageModel ←** Claude Sonnet 4.5
  - **ai_memory ←** Chat Memory
  - **ai_tool ←** Portfolio Vector Tool
  - **ai_outputParser ←** Structured Output Parser
  - **Main output →** Check CV Request
- **Edge cases / failures:**
  - If tool retrieval fails, agent may answer without evidence (prompt says “must not”, but enforcement depends on model compliance)
  - If structured output is malformed, parser auto-fix attempts recovery (see parser node), but can still fail
  - Any missing webhook fields used in expressions will produce runtime expression errors

#### Node: Claude Sonnet 4.5
- **Type / role:** Anthropic chat model; primary reasoning model for the agent + parser
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
  - Temperature: **0.3**
- **Connections:**
  - **ai_languageModel →** Portfolio AI Agent
  - **ai_languageModel →** Structured Output Parser (the parser uses an LLM to auto-fix if needed)
- **Edge cases / failures:**
  - Anthropic credential issues, rate limits, model availability
  - Temperature too high could reduce structured output compliance; 0.3 is moderate

#### Node: Chat Memory
- **Type / role:** Buffer window memory; keeps last N turns by session
- **Configuration choices:**
  - Session key: `={{ $json.body.sessionId }}`
  - Session ID type: **customKey**
  - Context window length: **10**
- **Connections:**
  - **ai_memory →** Portfolio AI Agent
- **Edge cases / failures:**
  - If `sessionId` missing/empty, different users may share memory or memory may not work as intended
  - Memory persistence behavior depends on n8n + node implementation (often ephemeral per execution unless backed by a store)

#### Node: Structured Output Parser
- **Type / role:** Structured JSON output enforcement for agent responses
- **Configuration choices:**
  - Auto-fix: **true** (uses the connected model to repair invalid JSON)
  - Schema example:
    ```json
    {
      "answer": "Your detailed response about the portfolio",
      "cvRequested": false
    }
    ```
- **Connections:**
  - **ai_outputParser →** Portfolio AI Agent
  - **ai_languageModel ←** Claude Sonnet 4.5 (for auto-fix)
- **Edge cases / failures:**
  - If the model repeatedly outputs invalid structure, auto-fix may still fail
  - If `answer` becomes excessively long, webhook response size may become an issue

#### Node: Portfolio Vector Tool
- **Type / role:** Vector store tool wrapper exposed to the agent as `portfolio_knowledge`
- **Configuration choices:**
  - Tool name: **portfolio_knowledge**
  - Description: retrieval of skills/projects/experience evidence
  - TopK: **5**
- **Connections:**
  - **ai_tool →** Portfolio AI Agent
  - **ai_vectorStore ←** Pinecone Retrieval
  - **ai_languageModel ←** Claude Retrieval Model (used internally for some tool behaviors depending on node)
- **Edge cases / failures:**
  - If the tool name does not match what the system prompt calls (`portfolio_knowledge`), the agent won’t call it (here it matches)

#### Node: Pinecone Retrieval
- **Type / role:** Pinecone Vector Store in retrieval mode for RAG queries
- **Configuration choices:**
  - Index: **portfolio-docs**
  - Use reranker: **true**
- **Connections:**
  - **ai_vectorStore →** Portfolio Vector Tool
  - **ai_embedding ←** OpenAI Embeddings Retrieval
  - **ai_reranker ←** Cohere Reranker
- **Edge cases / failures:**
  - Index missing or wrong credentials
  - If ingestion didn’t run or index is empty, retrieval returns no evidence
  - Reranking adds latency and an extra external dependency (Cohere)

#### Node: OpenAI Embeddings Retrieval
- **Type / role:** Embeddings provider used for query embeddings during retrieval
- **Configuration choices:**
  - Model: **text-embedding-3-small**
- **Connections:**
  - **ai_embedding →** Pinecone Retrieval
- **Edge cases / failures:**
  - Must match the same embedding space/dimension used in ingestion (here it does)

#### Node: Cohere Reranker
- **Type / role:** Reranks candidate matches to improve relevance
- **Configuration choices:**
  - Model: **rerank-v3.5**
  - TopN: **3**
- **Connections:**
  - **ai_reranker →** Pinecone Retrieval
- **Edge cases / failures:**
  - Cohere API key/rate limits
  - Reranking can reduce recall if TopN is too low (3)

#### Node: Claude Retrieval Model
- **Type / role:** Anthropic chat model used in the retrieval/tool subsystem
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
  - Temperature: **0.2**
- **Connections:**
  - **ai_languageModel →** Portfolio Vector Tool
- **Edge cases / failures:**
  - Same Anthropic concerns; adds additional model call overhead depending on implementation

---

### 2.3 CV Delivery Branch (Download + Email + Respond)

**Overview:**  
If the agent indicates `cvRequested=true`, the workflow downloads a preselected CV PDF from Google Drive and emails it to the requestor using Gmail, then responds to the webhook indicating the CV was sent.

**Nodes involved:**
- Check CV Request
- Download CV PDF
- Send CV Email
- Respond CV Sent

#### Node: Check CV Request
- **Type / role:** IF node; routes execution based on structured output flag
- **Configuration choices:**
  - Condition: `={{ $json.output.cvRequested }}` is **true** (boolean strict validation)
- **Connections:**
  - **Input ←** Portfolio AI Agent
  - **True →** Download CV PDF
  - **False →** Respond with Answer
- **Edge cases / failures:**
  - If `output.cvRequested` is missing or not boolean, strict validation can route unexpectedly or error
  - If the agent outputs `"true"` as string, condition may fail under strict typing

#### Node: Download CV PDF
- **Type / role:** Google Drive download node; fetches a fixed CV file
- **Configuration choices:**
  - Operation: **download**
  - File ID: **empty in JSON** (must be set to your CV file ID)
  - Output filename: `Resume.pdf`
- **Connections:**
  - **Input ←** Check CV Request (true branch)
  - **Output →** Send CV Email
- **Edge cases / failures:**
  - Missing file ID → node failure
  - Permission issues on the CV file
  - If file isn’t PDF, Gmail attachment still works but content type may differ

#### Node: Send CV Email
- **Type / role:** Gmail node; sends email with attachment
- **Configuration choices:**
  - To: `={{ $('Chat Webhook').item.json.body.email }}`
  - Subject: `Resume — Portfolio`
  - Message body: fixed text
  - Attachments: `attachmentsBinary: true` (expects binary on incoming item)
- **Connections:**
  - **Input ←** Download CV PDF (binary attachment present)
  - **Output →** Respond CV Sent
- **Edge cases / failures:**
  - Missing/invalid email in webhook payload → send failure
  - Gmail OAuth scopes/consent issues
  - Attachment mapping: Gmail node typically attaches all incoming binary fields; ensure the Drive download binary is present

#### Node: Respond CV Sent
- **Type / role:** Respond to Webhook; returns JSON response for CV path
- **Configuration choices:**
  - Response code: **200**
  - Respond with: **json**
  - Body expression:
    ```js
    {{ JSON.stringify({ answer: $('Portfolio AI Agent').item.json.output.answer, cvSent: true }) }}
    ```
- **Connections:**
  - **Input ←** Send CV Email
- **Edge cases / failures:**
  - If agent output missing `answer`, response may be incomplete
  - Uses `$('Portfolio AI Agent')...` referencing another node’s item; if execution branches/items mismatch, may cause item resolution issues in multi-item scenarios (typically single item here)

---

### 2.4 Non-CV Response Branch

**Overview:**  
When no CV is requested, the workflow responds to the webhook with the structured answer and `cvSent=false`.

**Nodes involved:**
- Respond with Answer

#### Node: Respond with Answer
- **Type / role:** Respond to Webhook; returns JSON response for non-CV path
- **Configuration choices:**
  - Response code: **200**
  - Respond with: **json**
  - Body expression:
    ```js
    {{ JSON.stringify({ answer: $json.output.answer, cvSent: false }) }}
    ```
- **Connections:**
  - **Input ←** Check CV Request (false branch)
- **Edge cases / failures:**
  - If `$json.output.answer` missing, caller receives `answer: null/undefined` depending on evaluation

---

### 2.5 Error Handling

**Overview:**  
Captures workflow execution errors and normalizes them into a compact object suitable for sending to an alerting system (not included, but suggested via sticky note).

**Nodes involved:**
- Error Handling Guide (Sticky Note)
- Error Trigger
- Format Error Data

#### Node: Error Handling Guide (Sticky Note)
- **Type / role:** Sticky note documentation
- **Configuration:** Suggests adding Slack/email/webhook after formatting node

#### Node: Error Trigger
- **Type / role:** Error Trigger; runs when any node in the workflow fails
- **Configuration choices:** Default
- **Connections:**
  - **Output →** Format Error Data
- **Edge cases / failures:**
  - If the error occurs before n8n can create execution metadata, some fields may be missing

#### Node: Format Error Data
- **Type / role:** Set node; extracts relevant error info from execution object
- **Configuration choices (fields created):**
  - `error_message` = `={{ $json.execution.error.message }}`
  - `failed_node` = `={{ $json.execution.error.node.name }}`
  - `workflow_name` = `={{ $json.workflow.name }}`
  - `execution_url` = `={{ $json.execution.url }}`
- **Connections:** None further (you’re expected to connect alerting)
- **Edge cases / failures:**
  - If `execution.url` is not available in your n8n environment, it may be empty
  - Some error shapes may not include `execution.error.node.name`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | High-level workflow description and setup checklist | — | — | ## AI Portfolio Assistant with RAG & CV Delivery / Ingestion + chat + error handling + setup steps |
| Section 1 Background | Sticky Note | Documents ingestion block description | — | — | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| File Created Trigger | Google Drive Trigger | Detect new files in watched folder | — | Download File | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| File Updated Trigger | Google Drive Trigger | Detect updated files in watched folder | — | Download File | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| Download File | Google Drive | Download changed file as binary | File Created Trigger; File Updated Trigger | Enrich Metadata | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| Enrich Metadata | Set | Add upload date and filename metadata | Download File | Pinecone Insert | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| OpenAI Embeddings | OpenAI Embeddings (LangChain) | Create embeddings for ingestion | — (used as AI connection) | Pinecone Insert (ai_embedding) | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| Text Splitter | Recursive Character Text Splitter (LangChain) | Chunk text for embedding | — (used as AI connection) | Document Loader (ai_textSplitter) | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| Document Loader | Default Data Loader (LangChain) | Parse binary into documents for indexing | Text Splitter (ai_textSplitter) | Pinecone Insert (ai_document) | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| Pinecone Insert | Pinecone Vector Store (LangChain) | Upsert chunks+vectors into Pinecone index | Enrich Metadata; Document Loader; OpenAI Embeddings | — | ## Ingestion pipeline / Monitors Google Drive... upserts into Pinecone |
| Section 2 Background | Sticky Note | Chat agent block description | — | — | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Chat Webhook | Webhook | Receive chat requests `{chatInput, sessionId, email}` | — | Portfolio AI Agent | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Portfolio AI Agent | LangChain Agent | Main agent: tool use + memory + structured response | Chat Webhook; (AI: Claude Sonnet 4.5, Memory, Tool, Output Parser) | Check CV Request | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Claude Sonnet 4.5 | Anthropic Chat Model (LangChain) | LLM for agent reasoning and parser autofix | — | Portfolio AI Agent; Structured Output Parser | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Chat Memory | Buffer Window Memory (LangChain) | Session memory for chat continuity | — | Portfolio AI Agent | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON `{answer, cvRequested}` | — | Portfolio AI Agent (ai_outputParser) | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Portfolio Vector Tool | Vector Store Tool (LangChain) | Agent tool `portfolio_knowledge` for evidence retrieval | Pinecone Retrieval (ai_vectorStore); Claude Retrieval Model (ai_languageModel) | Portfolio AI Agent (ai_tool) | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Claude Retrieval Model | Anthropic Chat Model (LangChain) | Model used in retrieval/tool subsystem | — | Portfolio Vector Tool | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Pinecone Retrieval | Pinecone Vector Store (LangChain) | Retrieve relevant chunks from Pinecone | OpenAI Embeddings Retrieval (ai_embedding); Cohere Reranker (ai_reranker) | Portfolio Vector Tool (ai_vectorStore) | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Cohere Reranker | Cohere Reranker (LangChain) | Rerank retrieved candidates | — | Pinecone Retrieval (ai_reranker) | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| OpenAI Embeddings Retrieval | OpenAI Embeddings (LangChain) | Embed query for vector search | — | Pinecone Retrieval (ai_embedding) | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Check CV Request | IF | Branch based on `output.cvRequested` | Portfolio AI Agent | Download CV PDF (true); Respond with Answer (false) | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Download CV PDF | Google Drive | Download a fixed CV PDF | Check CV Request (true) | Send CV Email | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Send CV Email | Gmail | Email CV as attachment to requester | Download CV PDF | Respond CV Sent | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Respond CV Sent | Respond to Webhook | Return JSON `{answer, cvSent:true}` | Send CV Email | — | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Respond with Answer | Respond to Webhook | Return JSON `{answer, cvSent:false}` | Check CV Request (false) | — | ## AI agent / Webhook-triggered Claude... Gmail delivery |
| Error Handling Guide | Sticky Note | Error handling guidance | — | — | ## Error handling / Connect an alerting service after Format Error Data |
| Error Trigger | Error Trigger | Start error flow on failures | — | Format Error Data | ## Error handling / Connect an alerting service after Format Error Data |
| Format Error Data | Set | Normalize error details for alerting | Error Trigger | — | ## Error handling / Connect an alerting service after Format Error Data |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   **“Transform your portfolio into an AI assistant with RAG and CV delivery”**.

2. **Add credentials (6 total)**
   - **Google Drive OAuth2** (for triggers + downloads)
   - **OpenAI API** (Embeddings)
   - **Pinecone API** (Vector store)
   - **Anthropic API** (Claude chat models)
   - **Cohere API** (Reranker)
   - **Gmail OAuth2** (Send email)

3. **Prepare Pinecone**
   - Create an index named **`portfolio-docs`**
   - Set **dimension = 1536**
   - Metric **cosine**

### Ingestion pipeline
4. **Add node: Google Drive Trigger** → rename **File Created Trigger**
   - Event: **File Created**
   - Trigger on: **Specific Folder**
   - Set **Folder ID** (the portfolio documents folder)
   - Polling: **Every minute**
   - File type: **All**

5. **Add node: Google Drive Trigger** → rename **File Updated Trigger**
   - Event: **File Updated**
   - Trigger on: **Specific Folder**
   - Set the **same Folder ID**
   - Polling: **Every minute**

6. **Add node: Google Drive** → rename **Download File**
   - Operation: **Download**
   - File ID: expression `{{$json.id}}`
   - Options → File Name: expression `{{$json.name}}`

7. **Connect**:
   - File Created Trigger → Download File
   - File Updated Trigger → Download File

8. **Add node: Set** → rename **Enrich Metadata**
   - Keep “Include Other Fields” enabled
   - Add fields:
     - `upload_date` (string) = `{{$now.toISO()}}`
     - `source_filename` (string) = `{{$json.name}}`

9. **Connect:** Download File → Enrich Metadata

10. **Add node: OpenAI Embeddings (LangChain)** → rename **OpenAI Embeddings**
   - Model: **text-embedding-3-small**

11. **Add node: Recursive Character Text Splitter (LangChain)** → rename **Text Splitter**
   - Chunk size: **600**
   - Chunk overlap: **100**

12. **Add node: Default Data Loader (LangChain)** → rename **Document Loader**
   - Data type: **Binary**
   - Binary mode: **Specific field**
   - Binary field name/key: **data**  
   (Ensure the Drive download node outputs binary under `binary.data`.)

13. **Add node: Pinecone Vector Store (LangChain)** → rename **Pinecone Insert**
   - Mode: **Insert**
   - Pinecone index: **portfolio-docs**

14. **Wire the AI connections for ingestion**
   - Text Splitter **(ai_textSplitter)** → Document Loader
   - Document Loader **(ai_document)** → Pinecone Insert
   - OpenAI Embeddings **(ai_embedding)** → Pinecone Insert
   - Enrich Metadata (main) → Pinecone Insert (main)

### Chat agent pipeline
15. **Add node: Webhook** → rename **Chat Webhook**
   - HTTP Method: **POST**
   - Path: **portfolio-query**
   - Response mode: **Using “Respond to Webhook” node**
   - Options: Allowed Origins = `*` (or restrict as needed)

16. **Add node: Anthropic Chat Model (LangChain)** → rename **Claude Sonnet 4.5**
   - Model: `claude-sonnet-4-5-20250929`
   - Temperature: **0.3**

17. **Add node: Memory Buffer Window (LangChain)** → rename **Chat Memory**
   - Session ID type: **Custom key**
   - Session key: `{{$json.body.sessionId}}`
   - Context window length: **10**

18. **Add node: Structured Output Parser (LangChain)** → rename **Structured Output Parser**
   - Auto-fix: **Enabled**
   - Schema example:
     - `answer` (string)
     - `cvRequested` (boolean)

19. **Add node: Pinecone Vector Store (LangChain)** → rename **Pinecone Retrieval**
   - Mode: retrieval/search (the node is used as a retriever)
   - Index: **portfolio-docs**
   - Enable **Use reranker**

20. **Add node: OpenAI Embeddings (LangChain)** → rename **OpenAI Embeddings Retrieval**
   - Model: **text-embedding-3-small**

21. **Add node: Cohere Reranker (LangChain)** → rename **Cohere Reranker**
   - Model: **rerank-v3.5**
   - TopN: **3**

22. **Connect AI dependencies for retrieval**
   - OpenAI Embeddings Retrieval **(ai_embedding)** → Pinecone Retrieval
   - Cohere Reranker **(ai_reranker)** → Pinecone Retrieval

23. **Add node: Anthropic Chat Model (LangChain)** → rename **Claude Retrieval Model**
   - Same model: `claude-sonnet-4-5-20250929`
   - Temperature: **0.2**

24. **Add node: Vector Store Tool (LangChain)** → rename **Portfolio Vector Tool**
   - Tool name: **portfolio_knowledge**
   - Description: retrieval of portfolio evidence
   - TopK: **5**

25. **Connect tool dependencies**
   - Pinecone Retrieval **(ai_vectorStore)** → Portfolio Vector Tool
   - Claude Retrieval Model **(ai_languageModel)** → Portfolio Vector Tool

26. **Add node: Agent (LangChain)** → rename **Portfolio AI Agent**
   - Input text: `{{$json.body.chatInput}}`
   - Prompt type: “Define”
   - System message: include the provided guidance (ground answers in tool evidence; set `cvRequested=true` when asked; structured output required)
   - Enable structured output parser usage

27. **Connect agent dependencies**
   - Chat Webhook → Portfolio AI Agent
   - Claude Sonnet 4.5 **(ai_languageModel)** → Portfolio AI Agent
   - Chat Memory **(ai_memory)** → Portfolio AI Agent
   - Portfolio Vector Tool **(ai_tool)** → Portfolio AI Agent
   - Structured Output Parser **(ai_outputParser)** → Portfolio AI Agent

### CV branch + responses
28. **Add node: IF** → rename **Check CV Request**
   - Condition: boolean **true**
   - Left value: `{{$json.output.cvRequested}}`

29. **Connect:** Portfolio AI Agent → Check CV Request

30. **Add node: Google Drive** → rename **Download CV PDF**
   - Operation: **Download**
   - File ID: set your **CV PDF file ID** (fixed)
   - Options → File Name: `Resume.pdf`

31. **Add node: Gmail** → rename **Send CV Email**
   - To: `{{$('Chat Webhook').item.json.body.email}}`
   - Subject: `Resume — Portfolio`
   - Message: as provided
   - Options: enable **Attachments Binary**

32. **Add node: Respond to Webhook** → rename **Respond CV Sent**
   - Respond with: **JSON**
   - Response code: **200**
   - Body:
     - `answer`: from `Portfolio AI Agent` output
     - `cvSent`: true  
     (Use expression similar to: `JSON.stringify({ answer: $('Portfolio AI Agent').item.json.output.answer, cvSent: true })`)

33. **Wire the CV branch**
   - Check CV Request (true) → Download CV PDF → Send CV Email → Respond CV Sent

34. **Add node: Respond to Webhook** → rename **Respond with Answer**
   - Response code: **200**
   - Body: `JSON.stringify({ answer: $json.output.answer, cvSent: false })`

35. **Wire the non-CV branch**
   - Check CV Request (false) → Respond with Answer

### Error handling
36. **Add node: Error Trigger** → rename **Error Trigger**

37. **Add node: Set** → rename **Format Error Data**
   - Set fields:
     - `error_message` = `{{$json.execution.error.message}}`
     - `failed_node` = `{{$json.execution.error.node.name}}`
     - `workflow_name` = `{{$json.workflow.name}}`
     - `execution_url` = `{{$json.execution.url}}`

38. **Connect:** Error Trigger → Format Error Data

39. *(Optional but recommended)* Add an alerting node after **Format Error Data** (Slack, Gmail, HTTP Request) to notify you.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create Pinecone index: dimension 1536, metric cosine, name `portfolio-docs` | Mentioned in the workflow “Overview” note |
| Configure 6 credentials: Google Drive, OpenAI, Pinecone, Anthropic, Cohere, Gmail | Mentioned in the workflow “Overview” note |
| Set Google Drive folder ID in both trigger nodes | Mentioned in the workflow “Overview” note |
| Set CV file ID in “Download CV PDF” | Mentioned in the workflow “Overview” note |
| Personalize the agent system prompt with your name and specialization | Mentioned in the workflow “Overview” note |
| Error Trigger exists; connect Slack/email/webhook after “Format Error Data” | Mentioned in “Error handling” note |