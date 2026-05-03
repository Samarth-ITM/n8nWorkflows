Generate M&A due diligence reports with OpenAI, LlamaIndex and Pinecone

https://n8nworkflows.xyz/workflows/generate-m-a-due-diligence-reports-with-openai--llamaindex-and-pinecone-13500


# Generate M&A due diligence reports with OpenAI, LlamaIndex and Pinecone

## 1. Workflow Overview

**Workflow name:** Due Diligence Automation v2.0.1  
**Template title (given):** Generate M&A due diligence reports with OpenAI, LlamaIndex and Pinecone

**Purpose:**  
This workflow receives one or more due-diligence documents via a POST webhook (multipart upload), creates/uses a deterministic **dealId**, checks whether embeddings for that deal already exist in **Pinecone** (cache), and either:
- **Cache hit:** directly performs AI analysis with retrieval from Pinecone, or
- **Cache miss:** parses documents via **LlamaIndex/LlamaParse**, embeds and upserts chunks into Pinecone, then performs AI analysis.

Finally, it renders a structured due diligence report to **HTML → PDF**, uploads the PDF to **S3**, builds a public URL, and returns it as the webhook response.

### 1.1 Intake & Request Normalization
Receives multipart upload, splits each binary file into its own item, and creates a shared `dealId`.

### 1.2 Pinecone Cache Check (Namespace existence)
Calls Pinecone index stats; if a namespace for `dealId` already has vectors, skip parsing/ingestion.

### 1.3 Document Parsing Loop (LlamaParse async)
For each file: upload → poll job status until SUCCESS → fetch parsed markdown → normalize payload.

### 1.4 Vector Ingestion (OpenAI embeddings → Pinecone upsert)
Converts parsed text into a LangChain document, generates embeddings, upserts into Pinecone under the deal namespace, then aggregates ingest results.

### 1.5 AI Due Diligence Analysis (Agent + retrieval tool + structured output)
Builds analysis context, uses Pinecone retrieval tool (multi-query enforced by prompt), runs OpenAI chat model, and parses output into strict JSON schema.

### 1.6 Report Rendering (HTML → PDF)
Maps structured analysis fields into an HTML template and renders a PDF using Puppeteer.

### 1.7 Delivery & Webhook Response (S3 + public URL)
Prepares output filename, uploads to S3, builds a public URL, merges it with analysis output, and responds to the webhook caller.

---

## 2. Block-by-Block Analysis

### Block 1 — Intake & Request Normalization
**Overview:** Receives multipart due diligence files, normalizes into one item per file, and computes a deterministic `dealId` from filenames.  
**Nodes involved:** `Receive Upload Request`, `Split Uploaded Files + Build Deal ID`

#### Node: Receive Upload Request
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point for HTTP POST uploads.
- **Config (interpreted):**
  - Method: `POST`
  - Response mode: `responseNode` (workflow must end at a Respond to Webhook node)
  - Path: `"5aed3279-e874-468b-91b6-902d99338f51"` (note: sticky note suggests `/dd-ai`, but current path is the generated ID)
- **I/O connections:**
  - Output → `Split Uploaded Files + Build Deal ID`
- **Edge cases / failures:**
  - If caller does not send `multipart/form-data` with binary fields, downstream code will throw.
  - If you change path to `/dd-ai`, ensure clients use the new route and update any reverse proxy.

#### Node: Split Uploaded Files + Build Deal ID
- **Type / role:** `Code` — converts webhook payload into multiple items and computes `dealId`.
- **Key logic:**
  - Reads `item.json.body.filenames` (expects JSON string array), otherwise derives filenames from `item.binary`.
  - Builds `dealId` by:
    1) sorting filenames  
    2) joining with `|`  
    3) base64 encoding  
    4) stripping non-alphanumerics  
    5) truncating to 20 chars
  - Splits binary keys that start with `data` into separate items, each with `binary.data`.
- **Outputs (per file item):**
  - `json.dealId`, `json.sourceFile`, `json.fileType`, `json.mimeType`, `json.fileIndex`, `json.totalFiles`
  - `binary.data` containing the file
- **I/O connections:**
  - Input ← `Receive Upload Request`
  - Output → `Get Pinecone Index Stats`
- **Edge cases / failures:**
  - Throws: `No binary files found in webhook request`
  - If binary keys don’t start with `data`, no files will be detected (depends on how the webhook maps multipart fields).
  - `dealId` collision risk exists (20 chars truncated), though usually low; increase length if needed.

---

### Block 2 — Pinecone Cache Check
**Overview:** Checks Pinecone index namespaces to see whether embeddings already exist for `dealId`, then routes to analysis or parsing/ingestion.  
**Nodes involved:** `Get Pinecone Index Stats`, ` Check Deal Namespace Cache`, `Cache Hit?`

#### Node: Get Pinecone Index Stats
- **Type / role:** `HTTP Request` — calls Pinecone `describe_index_stats`.
- **Config (interpreted):**
  - Method: `POST`
  - URL uses expression: `=[credentials.pineconeApi.environment] /describe_index_stats`
  - Adds header `Content-Type: "application/json'"` (note trailing `'`)
- **I/O connections:**
  - Input ← `Split Uploaded Files + Build Deal ID`
  - Output → ` Check Deal Namespace Cache`
- **Version notes:** typeVersion 4.3 (HTTP Request node)
- **Edge cases / failures (important):**
  - **URL expression appears malformed** (space before `/describe_index_stats`). It should typically be something like:
    - `{{$credentials.pineconeApi.environment}}/describe_index_stats` or the correct base URL for your Pinecone project.
  - Header `Content-Type` value has an extra `'` → may break Pinecone parsing.
  - Missing request body: Pinecone often allows empty body for stats, but some setups may require `{}`.
  - Auth is not shown here; ensure Pinecone API key is being applied via credentials or headers.

#### Node:  Check Deal Namespace Cache
- **Type / role:** `Code` — evaluates Pinecone stats and routes items.
- **Key variables:**
  - `stats = $input.first().json`
  - `originalItems = $('Split Uploaded Files + Build Deal ID').all()`
  - `dealId = originalItems[0]?.json?.dealId`
  - `vectorCount = stats.namespaces[dealId].vectorCount` (if exists)
- **Behavior:**
  - If `vectorCount > 0`: returns **one item** `{ dealId, cacheHit:true, vectorCount, message }`
  - Else: returns **all original file items** with `cacheHit:false` and preserves `binary`
- **I/O connections:**
  - Input ← `Get Pinecone Index Stats`
  - Output → `Cache Hit?`
- **Edge cases / failures:**
  - If Pinecone stats response shape differs (e.g., `namespaces` absent), `vectorCount` becomes 0 → forces cache miss.
  - Depends on `Split Uploaded Files + Build Deal ID` node name remaining unchanged (hard reference via `$()`).

#### Node: Cache Hit?
- **Type / role:** `IF` — conditional router.
- **Condition:** `{{$json.cacheHit}} equals true`
- **I/O connections:**
  - True → `Prepare Analysis Context`
  - False → `Iterate Files for Parsing`
- **Edge cases:**
  - If `cacheHit` missing, defaults to false path.

---

### Block 3 — Document Parsing Loop (LlamaParse)
**Overview:** For cache misses, processes each file: uploads to LlamaParse, polls async job completion, retrieves parsed markdown, and normalizes into a standard `{dealId, parsedText}` payload.  
**Nodes involved:** `Iterate Files for Parsing`, `Upload File to LlamaParse`, `Check LlamaParse Job Status`, `Is Parsing Job Complete?`, `Wait 10s Before Recheck`, `Retrieve Parsed Content`, `Normalize Parsed Text Payload`

#### Node: Iterate Files for Parsing
- **Type / role:** `Split in Batches` — iterates uploaded file items.
- **Config:** default batch options (not explicitly set).
- **I/O connections:**
  - Input ← `Cache Hit?` (false branch)
  - Output 1 → `Upload File to LlamaParse`
  - Output 2 → `Collect Ingested Deal IDs` (this is unusual; see edge cases)
- **Edge cases / failures:**
  - The workflow loops back into this node from `Upsert Chunks to Pinecone`, acting as a “next item” iterator.
  - Because it also connects directly to `Collect Ingested Deal IDs`, aggregation behavior may not match intent (it will receive items per batch step, not only at end).

#### Node: Upload File to LlamaParse
- **Type / role:** `HTTP Request` — uploads file for parsing.
- **Config (interpreted):**
  - POST `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
  - Multipart form-data with binary field: `file` from `binary.data`
  - Uses generic header auth credential: `llamaindex`
  - Header `accept: application/json`
- **I/O connections:**
  - Input ← `Iterate Files for Parsing`
  - Output → `Check LlamaParse Job Status`
- **Edge cases / failures:**
  - Auth/header missing or invalid → 401/403
  - Large files may time out depending on n8n HTTP settings and LlamaParse limits.

#### Node: Check LlamaParse Job Status
- **Type / role:** `HTTP Request` — polls the job.
- **Config:**
  - GET-like call (HTTP Request node default method is GET if not specified; here not explicitly set) to:
    - `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`
  - Uses same `llamaindex` header auth.
- **I/O connections:**
  - Input ← `Upload File to LlamaParse` or `Wait 10s Before Recheck`
  - Output → `Is Parsing Job Complete?`
- **Edge cases:**
  - If method defaults to GET but API expects GET, OK; if it expects POST, adjust.
  - If `$json.id` is missing (upload response changed), polling fails.

#### Node: Is Parsing Job Complete?
- **Type / role:** `IF` — checks `status === "SUCCESS"`.
- **I/O connections:**
  - True → `Retrieve Parsed Content`
  - False → `Wait 10s Before Recheck`
- **Edge cases:**
  - Ignores failure terminal states (e.g., `ERROR`), causing infinite polling; consider adding an ERROR branch.

#### Node: Wait 10s Before Recheck
- **Type / role:** `Wait` — pause between polls.
- **Config:** 10 seconds.
- **I/O connections:**
  - Output → `Check LlamaParse Job Status`
- **Edge cases:** Long-running jobs can exceed overall execution time limits.

#### Node: Retrieve Parsed Content
- **Type / role:** `HTTP Request` — fetches final parsed markdown.
- **Config:**
  - GET `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`
  - Header auth `llamaindex`
  - `accept: application/json`
- **I/O connections:**
  - Output → `Normalize Parsed Text Payload`
- **Edge cases:**
  - Response may be raw markdown rather than JSON; node expects JSON by default unless configured. If LlamaParse returns `text/markdown`, you may need to set “Response Format” appropriately.

#### Node: Normalize Parsed Text Payload
- **Type / role:** `Code` — produces canonical parsed text structure.
- **Key references:**
  - `loopItem = $('Iterate Files for Parsing').item.json` (uses current loop item)
  - `parsedContent = $json.markdown || $json.text || ''`
- **Output:**
  - `json.dealId`, `json.sourceFile`, `json.fileType`, `json.parsedText`
- **I/O connections:**
  - Output → `Upsert Chunks to Pinecone`
- **Edge cases:**
  - If retrieve response doesn’t contain `markdown` or `text`, ingested `parsedText` becomes empty (you should guard against empty text).

---

### Block 4 — Vector Ingestion
**Overview:** Converts parsed text into a document, generates embeddings with OpenAI, and upserts into Pinecone using `dealId` as namespace; then aggregates ingested IDs.  
**Nodes involved:** `Prepare Parsed Text Document`, `Generate Embeddings (Ingest)`, `Upsert Chunks to Pinecone`, `Collect Ingested Deal IDs`

#### Node: Prepare Parsed Text Document
- **Type / role:** `LangChain Document Default Data Loader` — wraps parsed text into a document with metadata.
- **Config:**
  - Adds metadata:
    - `deal_id = {{$json.dealId}}`
    - `source_file = {{$json.sourceFile}}`
    - `file_type = {{$json.fileType}}`
    - `timestamp = {{$now.toUTC()}}`
- **I/O connections:**
  - Provides `ai_document` input to `Upsert Chunks to Pinecone`
- **Edge cases:** If `parsedText` is empty upstream, embeddings/upsert will be low value or may fail depending on splitter behavior in Pinecone node.

#### Node: Generate Embeddings (Ingest)
- **Type / role:** `OpenAI Embeddings` (LangChain) — creates vectors for ingestion.
- **Credentials:** OpenAI (`Sumopod`)
- **I/O connections:**
  - Provides `ai_embedding` to `Upsert Chunks to Pinecone`
- **Edge cases:** OpenAI rate limits, invalid API key, model defaults (if embeddings model is not set explicitly).

#### Node: Upsert Chunks to Pinecone
- **Type / role:** `Vector Store: Pinecone` (LangChain) — inserts vectors.
- **Config:**
  - Mode: `insert`
  - Index: `poc`
  - Namespace: `{{ $('Iterate Files for Parsing').item.json.dealId }}`
  - `clearNamespace: false`
- **Credentials:** Pinecone (`w3khmuhtadin`)
- **I/O connections:**
  - Input ← `Normalize Parsed Text Payload`
  - Output → `Iterate Files for Parsing` (to continue batching/loop)
- **Edge cases:**
  - Namespace expression depends on loop item context; if called outside the loop, namespace may be undefined.
  - Index `poc` must exist and match embedding dimensionality.
  - Chunking strategy is not shown; Pinecone node may chunk internally or store whole doc; validate for large documents.

#### Node: Collect Ingested Deal IDs
- **Type / role:** `Aggregate` — collects `metadata.deal_id`.
- **Config:** aggregates field `metadata.deal_id`
- **I/O connections:**
  - Input ← `Iterate Files for Parsing` (direct)  
  - Output → `Prepare Analysis Context`
- **Edge cases:**
  - This node expects incoming items containing `metadata.deal_id`. Depending on n8n execution order and the current connections, it may receive items that **don’t** have that field (e.g., raw loop items), producing empty aggregation. Consider connecting it after `Upsert Chunks...` (or after a node that outputs the metadata) if you want guaranteed values.

---

### Block 5 — AI Due Diligence Analysis
**Overview:** Creates an analysis context item, runs a LangChain Agent with a Pinecone retrieval tool, enforces multi-query retrieval via prompt, and parses final output to strict JSON schema.  
**Nodes involved:** `Prepare Analysis Context`, `Retrieve Context from Pinecone`, ` Generate Embeddings (Retrieval)`, ` OpenAI Chat Model (5-mini)`, `Run Due Diligence AI Analysis`, `Parse Structured Analysis JSON`

#### Node: Prepare Analysis Context
- **Type / role:** `Code` — normalizes input into a single analysis request.
- **Behavior:**
  - If `cacheHit === true`: uses `items[0].json.dealId`
  - Else: expects aggregate output containing `deal_id` array; takes first.
  - Outputs: `dealId`, `filesProcessed`, `fromCache`, `timestamp`
- **I/O connections:**
  - Input ← `Cache Hit?` (true) OR `Collect Ingested Deal IDs`
  - Output → `Run Due Diligence AI Analysis`
- **Edge cases:**
  - If aggregation didn’t produce `deal_id`, falls back to `'unknown'`, which will query a wrong Pinecone namespace.

#### Node: Retrieve Context from Pinecone
- **Type / role:** `Vector Store: Pinecone` in `retrieve-as-tool` mode — exposes retrieval to the agent as a tool.
- **Config:**
  - `topK: 100`
  - Namespace: `{{$json.dealId}}`
  - Index: `poc`
  - Tool description explicitly instructs multiple focused queries.
- **I/O connections:**
  - Tool output wired to agent as `ai_tool`
  - Embeddings for queries provided by ` Generate Embeddings (Retrieval)`
- **Edge cases:**
  - If namespace doesn’t exist (wrong dealId), retrieval returns empty → model must output “Not Available”/0 per rules.

#### Node:  Generate Embeddings (Retrieval)
- **Type / role:** OpenAI embeddings for retrieval queries.
- **Credentials:** OpenAI (`Sumopod`)
- **I/O connections:**
  - Provides `ai_embedding` to `Retrieve Context from Pinecone`
- **Edge cases:** Same as ingest embeddings (rate limits, key, defaults).

#### Node:  OpenAI Chat Model (5-mini)
- **Type / role:** LangChain Chat LLM — the agent’s language model.
- **Config:**
  - Model: `gpt-5-mini`
- **I/O connections:**
  - Connected to agent via `ai_languageModel`
- **Edge cases:** Model availability, account permission, tool-calling compatibility depending on n8n/langchain versions.

#### Node: Parse Structured Analysis JSON
- **Type / role:** Structured output parser — enforces a JSON schema example.
- **Config:** Provides an example schema with:
  - `company_profile`
  - `financials` including `revenue_history`, `ebitda`, `margins`
  - `analysis` including `business_model`, `investment_thesis`, `key_risks`, `customer_concentration`
- **I/O connections:**
  - Parser wired to agent via `ai_outputParser`
- **Edge cases:** If the model outputs invalid JSON, parser fails the execution.

#### Node: Run Due Diligence AI Analysis
- **Type / role:** LangChain Agent — orchestrates multi-step retrieval + synthesis.
- **Config highlights:**
  - Prompt enforces **6 required Pinecone queries**, “no hallucination”, and strict JSON output rules.
  - `hasOutputParser: true`
- **I/O connections:**
  - Main output → `Map Analysis to Report Fields`
  - Also outputs to ` Merge Analysis + Report URL` (index 1) for later merging
  - Inputs:
    - From `Prepare Analysis Context`
    - LLM: ` OpenAI Chat Model (5-mini)`
    - Tool: `Retrieve Context from Pinecone`
    - Output parser: `Parse Structured Analysis JSON`
- **Edge cases:**
  - If the agent does not actually call the retrieval tool 6 times, you may still get output; the prompt is “policy” but not a hard enforcement unless you add additional validation.
  - Tool failures (Pinecone auth/index mismatch) can cascade into parser failures if the model outputs apologies/errors instead of JSON.

---

### Block 6 — Report Rendering (HTML → PDF)
**Overview:** Converts the analysis JSON into template fields, renders HTML, and generates a PDF via Puppeteer.  
**Nodes involved:** `Map Analysis to Report Fields`, `Render DD Report HTML`, `Render PDF from HTML`, `Convert PDF Base64 to Binary File`

#### Node: Map Analysis to Report Fields
- **Type / role:** `Code` — transforms structured JSON into HTML-ready fields.
- **Key behaviors:**
  - Uses `raw.output` if present; else uses raw JSON.
  - Normalizes missing values into “Not Available” (strings) and formats numbers.
  - Builds:
    - `revenueTableRows` HTML rows
    - `riskListItems` HTML list items with escaping
  - References `$('Prepare Analysis Context').first()?.json?.dealId`
  - Uses `DateTime.now()` (requires Luxon available in n8n Code node runtime)
- **I/O connections:**
  - Output → `Render DD Report HTML`
- **Edge cases:**
  - If Luxon `DateTime` is not available in your n8n build, this will throw. (Some n8n environments expose it; some don’t.)
  - If the agent output format changes, `raw.output` logic may select wrong object.

#### Node: Render DD Report HTML
- **Type / role:** `HTML` node — produces an HTML document using `{{$json...}}` fields.
- **Config:** Full HTML template with inline CSS and placeholders.
- **I/O connections:**
  - Output → `Render PDF from HTML`
- **Edge cases:** If any placeholder contains very large text, rendering can be slow or exceed PDF memory/time.

#### Node: Render PDF from HTML
- **Type / role:** `Puppeteer (runCustomScript)` — renders HTML into PDF.
- **Key script behavior:**
  - Reads `const html = $json.html`
  - `setContent(... networkidle0, timeout 60000)`
  - `$page.pdf({format:A4, printBackground:true, margins, scale:0.95, timeout:60000})`
  - Returns `{ pdfBase64 }`
- **I/O connections:**
  - Output → `Convert PDF Base64 to Binary File`
- **Edge cases:**
  - Requires n8n Puppeteer node installed and a runtime that supports Chromium.
  - `networkidle0` can hang if external resources are referenced (here it’s all inline, so typically safe).

#### Node: Convert PDF Base64 to Binary File
- **Type / role:** `Convert to File` — base64 → binary.
- **Config:**
  - Operation: toBinary from `pdfBase64`
  - Filename: `{{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`
- **I/O connections:**
  - Output → `Prepare S3 File Metadata`
- **Edge cases:**
  - Filename may contain characters not suitable for S3 keys; consider sanitizing.

---

### Block 7 — Delivery & Webhook Response
**Overview:** Generates a timestamped PDF filename, uploads to S3, constructs a public URL, merges with analysis output, and returns response.  
**Nodes involved:** `Prepare S3 File Metadata`, `Upload Report PDF to S3`, `Build Public Report URL`, ` Merge Analysis + Report URL`, `Return API Response`

#### Node: Prepare S3 File Metadata
- **Type / role:** `Code` — sets final S3 key name and passes along binary.
- **Logic:**
  - `companyName = $('Map Analysis to Report Fields').first().json.companyName`
  - `fileName = ${companyName}-assessment-${timestamp}.pdf`
  - Returns `{json:{fileName}, binary: $input.first().binary}`
- **I/O connections:**
  - Output → `Upload Report PDF to S3`
- **Edge cases:** Same filename sanitization concerns; also `companyName` could be “Not Available”.

#### Node: Upload Report PDF to S3
- **Type / role:** `S3` — uploads binary file.
- **Config:**
  - Operation: upload
  - Bucket: `poc`
  - File name: `{{$json.fileName}}` (object key)
- **Credentials:** `S3 account`
- **I/O connections:**
  - Output → `Build Public Report URL`
- **Edge cases:**
  - Bucket permissions, region mismatch, missing ACL/public access settings (workflow assumes a separate public-serving mechanism).

#### Node: Build Public Report URL
- **Type / role:** `Code` — builds a URL from base domain + filename.
- **Logic:**
  - `baseUrl = 'https://poc.atlr.dev'`
  - `fileName = $('Prepare S3 File Metadata').first().json.fileName`
  - `publicUrl = ${baseUrl}/${encodeURIComponent(fileName)}`
  - Returns `{ success:true, fileName, publicUrl }`
- **I/O connections:**
  - Output → ` Merge Analysis + Report URL`
- **Edge cases:**
  - This URL only works if `poc.atlr.dev` is configured to serve the S3 bucket (CloudFront/static hosting/proxy). S3 upload alone does not make it public.

#### Node:  Merge Analysis + Report URL
- **Type / role:** `Merge` (combine by position) — combines analysis output and URL output.
- **Config:** Mode `combine`, `combineByPosition`
- **Inputs:**
  - Input 1: from `Build Public Report URL`
  - Input 2: from `Run Due Diligence AI Analysis` (main output index 1)
- **Output →** `Return API Response`
- **Edge cases:**
  - If one branch produces 0 items (e.g., analysis failed), merge may output nothing or mismatch.

#### Node: Return API Response
- **Type / role:** `Respond to Webhook` — sends final response.
- **Config:** default options.
- **I/O connections:**
  - Input ← ` Merge Analysis + Report URL`
- **Edge cases:** If execution errors after webhook trigger and before this node, caller gets timeout/no response.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Upload Request | Webhook | Receive multipart POST request | — | Split Uploaded Files + Build Deal ID | ## Intake & Request Normalization  \nReceives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Split Uploaded Files + Build Deal ID | Code | Split files into items; compute deterministic dealId | Receive Upload Request | Get Pinecone Index Stats | ## Intake & Request Normalization  \nReceives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Get Pinecone Index Stats | HTTP Request | Describe Pinecone index namespaces for cache check | Split Uploaded Files + Build Deal ID |  Check Deal Namespace Cache | ## Pinecone Cache Check\nChecks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
|  Check Deal Namespace Cache | Code | Decide cache hit/miss and output either single analysis item or all file items | Get Pinecone Index Stats | Cache Hit? | ## Pinecone Cache Check\nChecks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Cache Hit? | IF | Route to analysis (hit) or parsing/ingest (miss) |  Check Deal Namespace Cache | Prepare Analysis Context; Iterate Files for Parsing | ## Pinecone Cache Check\nChecks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Iterate Files for Parsing | SplitInBatches | Iterate each uploaded file for parsing/ingest | Cache Hit? (false) ; Upsert Chunks to Pinecone | Upload File to LlamaParse; Collect Ingested Deal IDs |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Upload File to LlamaParse | HTTP Request | Upload a single file to LlamaParse | Iterate Files for Parsing | Check LlamaParse Job Status |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Check LlamaParse Job Status | HTTP Request | Poll parsing job status | Upload File to LlamaParse; Wait 10s Before Recheck | Is Parsing Job Complete? |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Is Parsing Job Complete? | IF | Check `status == SUCCESS` | Check LlamaParse Job Status | Retrieve Parsed Content; Wait 10s Before Recheck |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Wait 10s Before Recheck | Wait | Delay between polling attempts | Is Parsing Job Complete? (false) | Check LlamaParse Job Status |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Retrieve Parsed Content | HTTP Request | Fetch parsed markdown result | Is Parsing Job Complete? (true) | Normalize Parsed Text Payload |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Normalize Parsed Text Payload | Code | Standardize parsed output with dealId/file metadata | Retrieve Parsed Content | Upsert Chunks to Pinecone |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Prepare Parsed Text Document | LangChain Document Loader | Add metadata and package parsed text as document | (implicit) Normalize Parsed Text Payload | Upsert Chunks to Pinecone (ai_document) |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Generate Embeddings (Ingest) | OpenAI Embeddings (LangChain) | Create vectors for document chunks | (implicit) Normalize Parsed Text Payload | Upsert Chunks to Pinecone (ai_embedding) |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Upsert Chunks to Pinecone | Pinecone Vector Store (LangChain) | Insert vectors into Pinecone namespace | Normalize Parsed Text Payload | Iterate Files for Parsing |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Collect Ingested Deal IDs | Aggregate | Aggregate metadata.deal_id for analysis context | Iterate Files for Parsing | Prepare Analysis Context |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Prepare Analysis Context | Code | Normalize analysis inputs (cache vs ingest paths) | Cache Hit? (true); Collect Ingested Deal IDs | Run Due Diligence AI Analysis | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
|  Generate Embeddings (Retrieval) | OpenAI Embeddings (LangChain) | Embed retrieval queries | (implicit via tool wiring) | Retrieve Context from Pinecone (ai_embedding) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Retrieve Context from Pinecone | Pinecone Vector Store (LangChain) | Retrieval tool for the agent (topK=100) | Prepare Analysis Context (namespace via dealId) | Run Due Diligence AI Analysis (ai_tool) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
|  OpenAI Chat Model (5-mini) | OpenAI Chat Model (LangChain) | LLM for synthesis | — | Run Due Diligence AI Analysis (ai_languageModel) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Parse Structured Analysis JSON | Structured Output Parser (LangChain) | Enforce strict JSON structure | — | Run Due Diligence AI Analysis (ai_outputParser) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Run Due Diligence AI Analysis | LangChain Agent | Multi-query retrieval + structured due diligence JSON | Prepare Analysis Context | Map Analysis to Report Fields;  Merge Analysis + Report URL | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Map Analysis to Report Fields | Code | Convert analysis JSON to HTML template fields | Run Due Diligence AI Analysis | Render DD Report HTML | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Render DD Report HTML | HTML | Render final HTML using fields | Map Analysis to Report Fields | Render PDF from HTML | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Render PDF from HTML | Puppeteer | Convert HTML to PDF (base64) | Render DD Report HTML | Convert PDF Base64 to Binary File | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Convert PDF Base64 to Binary File | ConvertToFile | Base64 → binary PDF | Render PDF from HTML | Prepare S3 File Metadata | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Prepare S3 File Metadata | Code | Create final PDF filename and pass binary | Convert PDF Base64 to Binary File | Upload Report PDF to S3 | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Upload Report PDF to S3 | S3 | Upload PDF to S3 bucket | Prepare S3 File Metadata | Build Public Report URL | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Build Public Report URL | Code | Construct public URL from base domain + key | Upload Report PDF to S3 |  Merge Analysis + Report URL | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
|  Merge Analysis + Report URL | Merge | Combine analysis output + URL response | Build Public Report URL; Run Due Diligence AI Analysis | Return API Response | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Return API Response | RespondToWebhook | Send final response to caller |  Merge Analysis + Report URL | — | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note4 | Sticky Note | Comment block | — | — |  |
| Sticky Note5 | Sticky Note | Comment block | — | — |  |
| Sticky Note6 | Sticky Note | Comment block | — | — |  |
| Sticky Note7 | Sticky Note | Comment block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: `Due Diligence Automation v2.0.1` (or your preferred name)

2) **Add Webhook node**: `Receive Upload Request`
   - Node type: **Webhook**
   - HTTP Method: `POST`
   - Path: set to `/dd-ai` (if you want to match the note) or keep the generated ID
   - Response Mode: `Respond to Webhook` (responseNode)

3) **Add Code node**: `Split Uploaded Files + Build Deal ID`
   - Paste the provided JS logic (split items, compute `dealId`)
   - Ensure your client uploads files in multipart fields named like `data0`, `data1`, etc., or adjust `binaryKeys` detection.

4) **Add HTTP Request node**: `Get Pinecone Index Stats`
   - Method: `POST`
   - URL: your Pinecone index host + `/describe_index_stats`
     - Example form: `https://YOUR-INDEX-HOST/describe_index_stats`
   - Headers:
     - `Content-Type: application/json` (no trailing quote)
     - `Api-Key: <pinecone api key>` (either via credentials or header auth)
   - Body: `{}` (safe default)
   - Credentials: configure Pinecone credential or header auth as you prefer.

5) **Add Code node**: ` Check Deal Namespace Cache`
   - Paste the cache hit/miss logic.
   - This node must reference the earlier node by name: `Split Uploaded Files + Build Deal ID`.

6) **Add IF node**: `Cache Hit?`
   - Condition: boolean equals
     - Left: `{{$json.cacheHit}}`
     - Right: `true`
   - True output → (go to step 14)
   - False output → continue to parsing loop (step 7)

7) **Add SplitInBatches node**: `Iterate Files for Parsing`
   - Default settings are acceptable.
   - Connect `Cache Hit?` (false) → `Iterate Files for Parsing`

8) **Add HTTP Request node**: `Upload File to LlamaParse`
   - Method: `POST`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
   - Authentication: Header-based (create an **HTTP Header Auth** credential storing your LlamaIndex key/header)
   - Send Body: `multipart/form-data`
   - Add a body parameter:
     - name: `file`
     - type: **Binary**
     - input field: `data`
   - Header: `accept: application/json`

9) **Add HTTP Request node**: `Check LlamaParse Job Status`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}`
   - Use same LlamaIndex header auth
   - Header: `accept: application/json`

10) **Add IF node**: `Is Parsing Job Complete?`
   - Condition: `{{$json.status}} equals "SUCCESS"`
   - True → step 11
   - False → step 12

11) **Add HTTP Request node**: `Retrieve Parsed Content`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}/result/markdown`
   - Auth: LlamaIndex header auth
   - Ensure response is captured as text/JSON as required by your LlamaParse response format.

12) **Add Wait node**: `Wait 10s Before Recheck`
   - Wait: 10 seconds
   - Connect `Is Parsing Job Complete?` (false) → `Wait...` → `Check LlamaParse Job Status`

13) **Add Code node**: `Normalize Parsed Text Payload`
   - Paste normalization JS (uses `Iterate Files for Parsing` current item + parsed markdown/text)

14) **Add LangChain Document Loader node**: `Prepare Parsed Text Document`
   - Node type: **Document Default Data Loader**
   - Configure metadata fields:
     - `deal_id: {{$json.dealId}}`
     - `source_file: {{$json.sourceFile}}`
     - `file_type: {{$json.fileType}}`
     - `timestamp: {{$now.toUTC()}}`

15) **Add OpenAI Embeddings node**: `Generate Embeddings (Ingest)`
   - Credentials: create **OpenAI API** credential
   - Keep defaults unless you need a specific embeddings model.

16) **Add Pinecone Vector Store node**: `Upsert Chunks to Pinecone`
   - Mode: `insert`
   - Pinecone credentials: create/configure Pinecone credential
   - Index name: `poc` (must exist in Pinecone)
   - Namespace: `{{ $('Iterate Files for Parsing').item.json.dealId }}`
   - Connect:
     - Main input from `Normalize Parsed Text Payload`
     - `ai_document` input from `Prepare Parsed Text Document`
     - `ai_embedding` input from `Generate Embeddings (Ingest)`
   - Connect output back to `Iterate Files for Parsing` to continue batches.

17) **Add Aggregate node**: `Collect Ingested Deal IDs`
   - Aggregate field: `metadata.deal_id`
   - Connect it so it receives items that actually contain `metadata.deal_id` (adjust wiring if needed).

18) **Add Code node**: `Prepare Analysis Context`
   - Paste JS to produce `{dealId, filesProcessed, fromCache, timestamp}`
   - Connect:
     - `Cache Hit?` (true) → `Prepare Analysis Context`
     - `Collect Ingested Deal IDs` → `Prepare Analysis Context`

19) **Add Pinecone Vector Store node**: `Retrieve Context from Pinecone`
   - Mode: `retrieve-as-tool`
   - Index: `poc`
   - Namespace: `{{$json.dealId}}`
   - topK: `100`
   - Provide tool description instructing multi-query retrieval.

20) **Add OpenAI Embeddings node**: ` Generate Embeddings (Retrieval)`
   - Connect as `ai_embedding` to the retrieval tool.

21) **Add OpenAI Chat Model node**: ` OpenAI Chat Model (5-mini)`
   - Model: `gpt-5-mini`
   - Credentials: OpenAI

22) **Add Structured Output Parser node**: `Parse Structured Analysis JSON`
   - Paste the schema example used in the workflow.

23) **Add LangChain Agent node**: `Run Due Diligence AI Analysis`
   - Prompt: paste the full due diligence instruction block (6 mandatory queries, strict JSON rules)
   - Enable output parser
   - Wire:
     - `ai_languageModel` ← ` OpenAI Chat Model (5-mini)`
     - `ai_tool` ← `Retrieve Context from Pinecone`
     - `ai_outputParser` ← `Parse Structured Analysis JSON`
   - Main output → step 24 and also to merge later (step 31)

24) **Add Code node**: `Map Analysis to Report Fields`
   - Paste HTML field mapping JS (ensure `DateTime` is available; otherwise replace with `new Date().toISOString()` formatting)

25) **Add HTML node**: `Render DD Report HTML`
   - Paste the HTML template
   - It should output a property `html` used by Puppeteer.

26) **Add Puppeteer node**: `Render PDF from HTML`
   - Operation: run custom script
   - Paste PDF generation script (setContent + pdf + return base64)

27) **Add Convert to File node**: `Convert PDF Base64 to Binary File`
   - Operation: `toBinary`
   - Source property: `pdfBase64`
   - File name expression: `{{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`

28) **Add Code node**: `Prepare S3 File Metadata`
   - Paste JS to set `fileName = ${companyName}-assessment-${timestamp}.pdf` and pass `binary`

29) **Add S3 node**: `Upload Report PDF to S3`
   - Credentials: configure AWS S3 credentials (access key/secret or IAM role)
   - Operation: Upload
   - Bucket: `poc` (create it if missing)
   - File name: `{{$json.fileName}}`

30) **Add Code node**: `Build Public Report URL`
   - Set `baseUrl` to your public domain (CloudFront / reverse proxy)
   - URL-encode filename and return `{success, fileName, publicUrl}`

31) **Add Merge node**: ` Merge Analysis + Report URL`
   - Mode: `combine`
   - Combine by: position
   - Input 1: from `Build Public Report URL`
   - Input 2: from `Run Due Diligence AI Analysis` (the extra output connection)

32) **Add Respond to Webhook node**: `Return API Response`
   - Connect from merge node
   - This completes the webhook response.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Automated Due Diligence Report Generator** — Receives POST webhook at `/dd-ai`, checks Pinecone cache, parses via LlamaIndex, embeds + upserts to Pinecone, runs agent (gpt-5-mini) for structured due diligence JSON, renders HTML→PDF, uploads to S3, returns public URL. | Sticky note: “Automated Due Diligence Report Generator” |
| Setup checklist: configure webhook `/dd-ai`, OpenAI key + `gpt-5-mini`, Pinecone index `"poc"`, LlamaIndex parsing creds, S3 bucket `"poc"`, test via multipart POST with filenames array + binary fields. | Sticky note: “Setup” |
| disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | User-provided disclaimer (applies to the overall content) |