Generate M&A due diligence PDF reports with LlamaIndex, OpenAI, Pinecone, and S3

https://n8nworkflows.xyz/workflows/generate-m-a-due-diligence-pdf-reports-with-llamaindex--openai--pinecone--and-s3-13364


# Generate M&A due diligence PDF reports with LlamaIndex, OpenAI, Pinecone, and S3

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate M&A due diligence PDF reports with LlamaIndex, OpenAI, Pinecone, and S3  
**Workflow name (n8n):** Due Diligence Automation v2.0.1

This workflow receives multiple due-diligence documents via a POST webhook, creates a stable **dealId** based on filenames, and uses that ID as a Pinecone **namespace** for caching. If the namespace already contains vectors, it skips ingestion and immediately runs an AI due-diligence analysis using a LangChain Agent (OpenAI `gpt-5-mini`) backed by a Pinecone retrieval tool. If not cached, it parses each document via LlamaIndex (LlamaParse), normalizes the parsed text, generates embeddings, upserts them into Pinecone, then runs the same analysis. Finally, it renders an HTML report, converts it to PDF via Puppeteer, uploads it to S3, builds a public URL, and responds to the webhook caller with the result.

### 1.1 Intake & Deal Normalization
Receives multipart upload and transforms it into one item per file, all sharing a computed `dealId`.

### 1.2 Pinecone Cache Check
Checks Pinecone namespace stats to detect whether this deal has already been ingested (cache hit) or needs parsing/embedding (cache miss).

### 1.3 Document Parsing Loop (Cache Miss Path)
Uploads each file to LlamaParse, polls until the job completes, retrieves markdown, normalizes text payload per file.

### 1.4 Vector Ingestion (Cache Miss Path)
Wraps parsed text as a LangChain document with metadata, generates embeddings, upserts into Pinecone under the `dealId` namespace, and aggregates ingestion IDs.

### 1.5 AI Due Diligence Analysis (Both Paths)
Builds analysis context, uses Pinecone retrieval tool + enforced multi-query instructions, generates structured JSON via an output parser.

### 1.6 Report Rendering + Delivery
Maps structured JSON to HTML fields, renders PDF, uploads to S3, builds a public link, merges outputs, returns response.

---

## 2. Block-by-Block Analysis

### Block A — Intake & Request Normalization
**Overview:** Receives the webhook request containing multipart files and creates one normalized n8n item per file. A shared `dealId` is generated deterministically from uploaded filenames so repeated uploads resolve to the same Pinecone namespace.

**Nodes involved:**
- Receive Upload Request
- Split Uploaded Files + Build Deal ID

#### Node: Receive Upload Request
- **Type / role:** `Webhook` — entry point, receives POST requests with multipart form-data.
- **Key configuration:**
  - HTTP method: `POST`
  - Response mode: `responseNode` (the workflow will reply via a Respond node)
  - Path: currently set to the internal UUID-like path `5aed3279-e874-468b-91b6-902d99338f51` (not `/dd-ai` as suggested in the sticky note).
- **Input/Output:**
  - **Output:** One item with `json.body` and `binary` fields (typical for multipart).
  - Connects to **Split Uploaded Files + Build Deal ID**.
- **Edge cases / failures:**
  - Missing multipart binary payload → downstream code throws “No binary files found…”.
  - If caller expects `/dd-ai` but webhook path differs, the endpoint won’t match.

#### Node: Split Uploaded Files + Build Deal ID
- **Type / role:** `Code` — splits incoming binaries into per-file items and computes `dealId`.
- **Key logic:**
  - Attempts to parse `body.filenames` as JSON array; fallback: uses `binary` file names.
  - `dealId` = base64 of concatenated sorted filenames (joined with `|`), sanitized to alphanumerics, truncated to 20 chars.
  - Creates items for keys that start with `data` in `binary` (e.g., `data0`, `data1`).
  - Each output item includes:
    - `json.dealId`, `sourceFile`, `fileType`, `mimeType`, `fileIndex`, `totalFiles`
    - `binary.data` = the specific file
- **Input/Output:**
  - **Input:** Webhook item (single).
  - **Output:** Multiple items (one per file).
  - Connects to **Get Pinecone Index Stats**.
- **Edge cases / failures:**
  - If binary keys don’t start with `data`, files are ignored → throws “No binary files found…”.
  - Different filename order from caller is normalized via sorting, but *renaming files changes dealId*, causing cache misses.

**Sticky note coverage:**  
“## Intake & Request Normalization …” applies to nodes in this block.

---

### Block B — Pinecone Cache Check
**Overview:** Checks Pinecone index stats to see whether vectors already exist for the computed deal namespace. Routes to either direct analysis (cache hit) or parsing/ingestion (cache miss).

**Nodes involved:**
- Get Pinecone Index Stats
- Check Deal Namespace Cache
- Cache Hit?

#### Node: Get Pinecone Index Stats
- **Type / role:** `HTTP Request` — calls Pinecone `describe_index_stats`.
- **Key configuration (interpreted):**
  - Method: `POST`
  - Endpoint expression: `=[credentials.pineconeApi.environment] /describe_index_stats`
  - Sends header: `Content-Type: "application/json'"` (note trailing `'`).
- **Input/Output:**
  - **Input:** Items from split step (but this node produces one response used to decide routing).
  - **Output:** Pinecone stats JSON, connected to **Check Deal Namespace Cache**.
- **Version requirements:** Node type v4.3.
- **Potential misconfigurations / failures:**
  - The URL expression contains a **space** before `/describe_index_stats` and depends on how `credentials.pineconeApi.environment` is formatted. It should generally be a full base URL like `https://<index-host>` or `https://controller.<region>.pinecone.io` depending on API usage.
  - Header value includes an extra `'`: `application/json'` which may cause request issues in stricter servers.
  - Missing API key header/auth configuration here (unlike the LangChain Pinecone node). If Pinecone requires an API key for this request, it must be added (either via credentials or explicit headers).

#### Node: Check Deal Namespace Cache
- **Type / role:** `Code` — decides cache hit/miss and shapes output accordingly.
- **Key logic:**
  - Reads stats from previous node.
  - Re-reads original split items via `$('Split Uploaded Files + Build Deal ID').all()`.
  - Gets `dealId` from the first original item.
  - Checks `stats.namespaces[dealId].vectorCount`.
  - **Cache HIT:** returns a single item: `{ dealId, cacheHit: true, vectorCount, message }`
  - **Cache MISS:** returns *all original items* and keeps binary; adds `{ cacheHit: false }`.
- **Input/Output:**
  - Connects to **Cache Hit?**.
- **Edge cases / failures:**
  - If Pinecone stats response format differs (no `namespaces`), `namespaceExists` may be undefined → treated as miss.
  - If multiple deals are mixed in one request (not expected), only first `dealId` is used.

#### Node: Cache Hit?
- **Type / role:** `IF` — routes based on boolean `cacheHit`.
- **Condition:** `{{$json.cacheHit}} equals true`.
- **Outputs:**
  - **True (cache hit):** to **Prepare Analysis Context**.
  - **False (cache miss):** to **Iterate Files for Parsing**.
- **Edge cases:**
  - If `cacheHit` missing, condition false → parsing path.

**Sticky note coverage:**  
“## Pinecone Cache Check …” applies to nodes in this block.

---

### Block C — Document Parsing Loop (LlamaParse)
**Overview:** For each uploaded file, upload to LlamaParse, poll until parsing is complete, retrieve markdown results, and normalize the parsed text into a consistent structure for embedding.

**Nodes involved:**
- Iterate Files for Parsing
- Upload File to LlamaParse
- Check LlamaParse Job Status
- Is Parsing Job Complete?
- Wait 10s Before Recheck
- Retrieve Parsed Content
- Normalize Parsed Text Payload

#### Node: Iterate Files for Parsing
- **Type / role:** `Split In Batches` — iterates through file items.
- **Key configuration:** default options; batch size not explicitly set (defaults apply).
- **Connections:**
  - One output goes to **Upload File to LlamaParse** (per file).
  - Another output goes to **Collect Ingested Deal IDs** (this is unusual; see edge case below).
- **Edge cases / concerns:**
  - The connection to **Collect Ingested Deal IDs** from the iterator can cause aggregation of items *before* parsing/embedding completes, depending on execution semantics. Typically aggregation should come after upsert completion, not directly from the iterator.

#### Node: Upload File to LlamaParse
- **Type / role:** `HTTP Request` — uploads the binary file to LlamaIndex parsing API.
- **Key configuration:**
  - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
  - Method: `POST`
  - Body: multipart-form-data with `file` taken from binary field `data`
  - Auth: `httpHeaderAuth` credential named `llamaindex`
  - Header: `accept: application/json`
- **Input/Output:** Receives one file item, returns JSON including a parsing `id`. Connected to **Check LlamaParse Job Status**.
- **Failures:**
  - Credential missing/invalid → 401/403.
  - Unsupported file types/size → 4xx.

#### Node: Check LlamaParse Job Status
- **Type / role:** `HTTP Request` — polls job status.
- **Key configuration:**
  - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`
  - Auth: same `llamaindex` header auth
- **Output:** Job status payload containing `status`. Connected to **Is Parsing Job Complete?**
- **Failures:** transient 5xx; rate limits if polling too frequently.

#### Node: Is Parsing Job Complete?
- **Type / role:** `IF` — checks parsing completion.
- **Condition:** `{{$json.status}} == "SUCCESS"`.
- **True path:** **Retrieve Parsed Content**  
- **False path:** **Wait 10s Before Recheck**
- **Edge cases:** if LlamaParse uses other terminal states (e.g., `ERROR`, `FAILED`), this workflow will loop forever unless `status` becomes `SUCCESS`.

#### Node: Wait 10s Before Recheck
- **Type / role:** `Wait` — pauses 10 seconds.
- **Then:** returns to **Check LlamaParse Job Status** (poll loop).
- **Failures:** very long parsing times can lead to long-running executions; ensure n8n execution timeouts/queue mode are compatible.

#### Node: Retrieve Parsed Content
- **Type / role:** `HTTP Request` — retrieves markdown result.
- **Key configuration:**
  - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`
  - Auth: `llamaindex` header auth
  - Header: `accept: application/json` (even though markdown is returned; API may wrap it in JSON)
- **Output:** expected to contain `markdown` (or `text`). Connected to **Normalize Parsed Text Payload**.

#### Node: Normalize Parsed Text Payload
- **Type / role:** `Code` — standardizes parsed output and reinjects metadata from the loop item.
- **Key logic:**
  - Reads the current loop’s item metadata via `$('Iterate Files for Parsing').item.json`.
  - `parsedText = $json.markdown || $json.text || ''`
  - Outputs `{ dealId, sourceFile, fileType, parsedText }`
- **Connected to:** **Upsert Chunks to Pinecone** (as main input for ingestion).
- **Edge cases:** if parsed result is empty, embeddings/upsert will still proceed, likely producing low-value vectors.

**Sticky note coverage:**  
“## Document Parsing Loop …” applies to nodes in this block.

---

### Block D — Vector Ingestion (Embeddings + Pinecone Upsert)
**Overview:** Converts normalized parsed text into a document with metadata, generates embeddings via OpenAI, and inserts them into Pinecone under the `dealId` namespace.

**Nodes involved:**
- Prepare Parsed Text Document
- Generate Embeddings (Ingest)
- Upsert Chunks to Pinecone
- Collect Ingested Deal IDs

#### Node: Prepare Parsed Text Document
- **Type / role:** `LangChain Document Default Data Loader` — builds document objects with metadata for vector storage.
- **Key configuration:**
  - Adds metadata:
    - `deal_id = {{$json.dealId}}`
    - `source_file = {{$json.sourceFile}}`
    - `file_type = {{$json.fileType}}`
    - `timestamp = {{$now.toUTC()}}`
- **Connections:** provides `ai_document` to **Upsert Chunks to Pinecone**.
- **Edge cases:** If the node expects a particular text field (often `text` or `pageContent`) but input is `parsedText`, ingestion may be incomplete unless the node automatically maps; verify in n8n UI that `parsedText` is used as document content.

#### Node: Generate Embeddings (Ingest)
- **Type / role:** `OpenAI Embeddings` — generates vectors for ingestion.
- **Credentials:** OpenAI API credential `Sumopod`.
- **Connections:** provides `ai_embedding` to **Upsert Chunks to Pinecone**.
- **Edge cases:** OpenAI rate limits; large documents may require chunking upstream (not present here). If Upsert node does internal chunking, ensure it is enabled/works as expected.

#### Node: Upsert Chunks to Pinecone
- **Type / role:** `Vector Store: Pinecone` (LangChain) — inserts embeddings/documents.
- **Key configuration:**
  - Mode: `insert`
  - Index: `poc`
  - Namespace: `={{ $('Iterate Files for Parsing').item.json.dealId }}`
  - `clearNamespace: false` (keeps prior vectors)
- **Credentials:** Pinecone API credential `w3khmuhtadin`.
- **Input requirements:**
  - Main input: comes from **Normalize Parsed Text Payload**
  - `ai_document`: from **Prepare Parsed Text Document**
  - `ai_embedding`: from **Generate Embeddings (Ingest)**
- **Output:** connected back to **Iterate Files for Parsing** (to continue the loop).
- **Edge cases / failures:**
  - Namespace expression references the iterator’s current item; if called outside the loop context, it can break.
  - If index `poc` doesn’t exist, upsert fails.
  - If Pinecone project/region mismatch, connectivity errors.

#### Node: Collect Ingested Deal IDs
- **Type / role:** `Aggregate` — collects `metadata.deal_id`.
- **Key configuration:** aggregates field `metadata.deal_id`.
- **Connected to:** **Prepare Analysis Context**.
- **Important concern:** In the current wiring, **Iterate Files for Parsing** also feeds into this node directly, while the upsert feeds back to the iterator. This can lead to aggregation not reflecting completed upserts. The intended design is usually: `Upsert -> Aggregate -> Continue/Finish`.

**Sticky note coverage:**  
“## Vector Ingestion …” applies to nodes in this block.

---

### Block E — AI Due Diligence Analysis (Agent + Retrieval + Structured Output)
**Overview:** Creates a minimal analysis context (`dealId`, cache info), then runs a LangChain Agent using OpenAI chat model and a Pinecone retrieval tool. The agent is instructed to perform six mandatory queries and return only JSON matching the schema.

**Nodes involved:**
- Prepare Analysis Context
- Retrieve Context from Pinecone
- Generate Embeddings (Retrieval)
- OpenAI Chat Model (5-mini)
- Parse Structured Analysis JSON
- Run Due Diligence AI Analysis

#### Node: Prepare Analysis Context
- **Type / role:** `Code` — normalizes context for analysis regardless of cache hit/miss path.
- **Key logic:**
  - If `cacheHit === true`, uses `items[0].json.dealId`.
  - Else uses aggregated `deal_id` field and takes first element.
  - Outputs: `{ dealId, filesProcessed, fromCache, timestamp }`
- **Connected to:** **Run Due Diligence AI Analysis**.
- **Edge cases:**
  - If aggregate output shape differs, `dealId` may become `'unknown'`, breaking retrieval namespace.

#### Node: Retrieve Context from Pinecone
- **Type / role:** `Vector Store: Pinecone (retrieve-as-tool)` — exposed as an agent tool.
- **Key configuration:**
  - Mode: `retrieve-as-tool`
  - `topK: 100`
  - Namespace: `={{ $json.dealId }}`
  - Tool description explicitly instructs multiple focused queries.
- **Credentials:** Pinecone API.
- **Connections:** Provided to agent as `ai_tool`.
- **Edge cases:** Large `topK` can increase latency/cost; namespace mismatch yields empty context.

#### Node: Generate Embeddings (Retrieval)
- **Type / role:** `OpenAI Embeddings` — used by retrieval tool for query embedding.
- **Connections:** `ai_embedding` to **Retrieve Context from Pinecone**.
- **Edge cases:** same as embeddings ingest (rate limits).

#### Node: OpenAI Chat Model (5-mini)
- **Type / role:** `ChatOpenAI` — LLM used by agent.
- **Model:** `gpt-5-mini`.
- **Connections:** supplies `ai_languageModel` to **Run Due Diligence AI Analysis**.
- **Failures:** model availability; API key permissions; high token usage.

#### Node: Parse Structured Analysis JSON
- **Type / role:** `Structured Output Parser` — constrains agent output to a JSON schema example.
- **Key configuration:** schema example includes:
  - `company_profile` (name, industry, location, employee_count)
  - `financials` (revenue_history array, ebitda, margins)
  - `analysis` (business_model, investment_thesis, key_risks array, customer_concentration)
- **Connections:** provides `ai_outputParser` to **Run Due Diligence AI Analysis**.
- **Edge cases:** If agent returns invalid JSON, parser errors; prompts attempt to prevent this (“Return ONLY valid JSON”).

#### Node: Run Due Diligence AI Analysis
- **Type / role:** `LangChain Agent` — orchestrates tool calls and final structured output.
- **Key configuration:**
  - Prompt contains **mandatory retrieval strategy**: execute 6 required queries before generating output.
  - Strict rules about missing values: `"Not Available"` for strings, `0` for numerics.
  - Output must be JSON matching parser schema.
- **Inputs:**
  - Main JSON: from **Prepare Analysis Context**
  - LLM: from **OpenAI Chat Model (5-mini)**
  - Tool: from **Retrieve Context from Pinecone**
  - Output parser: from **Parse Structured Analysis JSON**
- **Outputs:**
  - To **Map Analysis to Report Fields** (for PDF generation)
  - To **Merge Analysis + Report URL** (input index 1), to merge analysis with URL later.
- **Edge cases / failures:**
  - Agent may not actually execute all 6 tool calls; enforcement is prompt-based only (no hard validator).
  - Large retrieved context can exceed token limits; topK=100 increases this risk.

**Sticky note coverage:**  
“## AI Due Diligence Analysis …” applies to nodes in this block.

---

### Block F — Report Rendering (HTML → PDF)
**Overview:** Transforms structured analysis into HTML-friendly fields, renders a styled HTML report, then uses Puppeteer to generate a PDF and converts it to binary for upload.

**Nodes involved:**
- Map Analysis to Report Fields
- Render DD Report HTML
- Render PDF from HTML
- Convert PDF Base64 to Binary File

#### Node: Map Analysis to Report Fields
- **Type / role:** `Code` — maps and sanitizes analysis output into template variables.
- **Key behaviors:**
  - Accepts either `{output: {...}}` or raw object.
  - Provides HTML escaping helpers and “missing value” normalization.
  - Builds:
    - `revenueTableRows` (HTML rows)
    - `riskListItems` (HTML list)
  - Uses `Prepare Analysis Context` to set `dealId`.
  - Uses Luxon `DateTime.now()` for `reportDate`.
- **Output:** JSON fields used by HTML node.
- **Failures / edge cases:**
  - If Luxon `DateTime` isn’t available in your n8n code environment, this line fails. (Many n8n builds include it, but verify.)
  - Treats numeric `0` as missing (by design); this matches earlier “use 0 when missing” but will display as “Not Available”.

#### Node: Render DD Report HTML
- **Type / role:** `HTML` — renders a full HTML document using `{{ $json.* }}` variables.
- **Output:** `json.html` containing full HTML.
- **Edge cases:** if any mapped fields contain unescaped HTML, it could break layout; mapper escapes for table rows and risks but not for all fields (e.g., `companyName` is inserted raw).

#### Node: Render PDF from HTML
- **Type / role:** `Puppeteer (runCustomScript)` — headless browser renders HTML and exports PDF.
- **Key script:**
  - `await $page.setContent(html, { waitUntil: 'networkidle0', timeout: 60000 })`
  - `$page.pdf({ format: 'A4', printBackground: true, margins, scale: 0.95, timeout: 60000 })`
  - Returns `{ pdfBase64 }`
- **Edge cases / failures:**
  - Puppeteer node requires n8n environment with Chromium dependencies.
  - `networkidle0` can hang if the HTML references external resources (it doesn’t here).
  - Large HTML may slow rendering; timeouts are set to 60s.

#### Node: Convert PDF Base64 to Binary File
- **Type / role:** `Convert to File` — converts `pdfBase64` into `binary`.
- **Key configuration:**
  - Operation: `toBinary`
  - Source property: `pdfBase64`
  - File name: `{{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`
- **Edge cases:**
  - Company name with characters not suitable for filenames can cause issues; later steps override filename anyway.

**Sticky note coverage:**  
“## Report Rendering (HTML to PDF) …” applies to nodes in this block.

---

### Block G — Delivery & Webhook Response (S3 + URL)
**Overview:** Creates a timestamped PDF filename, uploads the binary PDF to S3 bucket `poc`, constructs a public URL using a fixed base domain, merges the URL with analysis output, and responds to the webhook request.

**Nodes involved:**
- Prepare S3 File Metadata
- Upload Report PDF to S3
- Build Public Report URL
- Merge Analysis + Report URL
- Return API Response

#### Node: Prepare S3 File Metadata
- **Type / role:** `Code` — creates a unique fileName and forwards binary.
- **Key logic:**
  - `companyName` from **Map Analysis to Report Fields**
  - `fileName = ${companyName}-assessment-${timestamp}.pdf`
  - Returns `{ json: { fileName }, binary: $input.first().binary }`
- **Edge cases:** companyName “Not Available” leads to generic filenames.

#### Node: Upload Report PDF to S3
- **Type / role:** `S3` — uploads PDF.
- **Key configuration:**
  - Operation: `upload`
  - Bucket: `poc`
  - File name: `{{$json.fileName}}`
- **Credentials:** `S3 account`.
- **Edge cases / failures:**
  - Bucket not existing or no permission.
  - Object ACL/public access is not configured here; “public URL” assumes a separate mechanism (CDN/static site) serves the object.

#### Node: Build Public Report URL
- **Type / role:** `Code` — constructs a public link.
- **Key logic:**
  - `baseUrl = 'https://poc.atlr.dev'`
  - Reads fileName from **Prepare S3 File Metadata**
  - URL = `${baseUrl}/${encodeURIComponent(fileName)}`
  - Returns `{ success: true, fileName, publicUrl }`
- **Edge cases:** This URL will only work if `poc.atlr.dev` maps to the S3 bucket (e.g., via CloudFront or static hosting).

#### Node: Merge Analysis + Report URL
- **Type / role:** `Merge` — combines the AI analysis output with the URL payload.
- **Mode:** `combineByPosition`.
- **Inputs:**
  - Input 0: from **Build Public Report URL**
  - Input 1: from **Run Due Diligence AI Analysis** (direct branch)
- **Edge cases:**
  - If one branch produces multiple items or timing differs, combine-by-position can mis-pair results. The design assumes single-item outputs on both inputs.

#### Node: Return API Response
- **Type / role:** `Respond to Webhook` — sends final response to the original caller.
- **Key configuration:** default options (response body will be the incoming merged JSON item).
- **Edge cases:** If execution errors occur before this node, caller gets an error/timeout.

**Sticky note coverage:**  
“## Delivery & Webhook Response …” applies to nodes in this block.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Overall workflow description & setup | — | — | ## Automated Due Diligence Report Generator / How it works / Setup checklist |
| Receive Upload Request | Webhook | Receives multipart POST request | — | Split Uploaded Files + Build Deal ID | ## Intake & Request Normalization  \nReceives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Split Uploaded Files + Build Deal ID | Code | Split binaries into per-file items; compute dealId | Receive Upload Request | Get Pinecone Index Stats | ## Intake & Request Normalization  \nReceives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Get Pinecone Index Stats | HTTP Request | Query Pinecone describe_index_stats | Split Uploaded Files + Build Deal ID | Check Deal Namespace Cache | ## Pinecone Cache Check\nChecks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Check Deal Namespace Cache | Code | Decide cache hit/miss; reshape items | Get Pinecone Index Stats | Cache Hit? | ## Pinecone Cache Check\nChecks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Cache Hit? | IF | Route to analysis or ingestion | Check Deal Namespace Cache | Prepare Analysis Context; Iterate Files for Parsing | ## Pinecone Cache Check\nChecks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Iterate Files for Parsing | Split In Batches | Iterate each file for parsing/ingest | Cache Hit? (false path) | Upload File to LlamaParse; Collect Ingested Deal IDs |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Upload File to LlamaParse | HTTP Request | Upload file to LlamaParse | Iterate Files for Parsing | Check LlamaParse Job Status |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Check LlamaParse Job Status | HTTP Request | Poll parsing job status | Upload File to LlamaParse; Wait 10s Before Recheck | Is Parsing Job Complete? |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Is Parsing Job Complete? | IF | Loop until SUCCESS | Check LlamaParse Job Status | Retrieve Parsed Content; Wait 10s Before Recheck |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Wait 10s Before Recheck | Wait | Delay between polls | Is Parsing Job Complete? (false path) | Check LlamaParse Job Status |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Retrieve Parsed Content | HTTP Request | Fetch markdown parsing result | Is Parsing Job Complete? (true path) | Normalize Parsed Text Payload |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Normalize Parsed Text Payload | Code | Standardize parsed text + metadata | Retrieve Parsed Content | Upsert Chunks to Pinecone |  ## Document Parsing Loop\nUploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Prepare Parsed Text Document | LangChain Document Loader | Build document + metadata for vector store | (implicit via Upsert node inputs) | Upsert Chunks to Pinecone (ai_document) |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Generate Embeddings (Ingest) | OpenAI Embeddings | Embeddings for ingestion | (implicit) | Upsert Chunks to Pinecone (ai_embedding) |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Upsert Chunks to Pinecone | Pinecone Vector Store | Insert vectors into Pinecone namespace | Normalize Parsed Text Payload | Iterate Files for Parsing |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Collect Ingested Deal IDs | Aggregate | Aggregate deal IDs after ingestion | Iterate Files for Parsing | Prepare Analysis Context |  ## Vector Ingestion \nConverts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Prepare Analysis Context | Code | Normalize analysis input (cache hit/miss) | Cache Hit? (true path); Collect Ingested Deal IDs | Run Due Diligence AI Analysis | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Retrieve Context from Pinecone | Pinecone Vector Store (tool) | Retrieval tool for agent | (agent tool wiring) | Run Due Diligence AI Analysis (ai_tool) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
|  Generate Embeddings (Retrieval) | OpenAI Embeddings | Query embeddings for retrieval | (tool wiring) | Retrieve Context from Pinecone (ai_embedding) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
|  OpenAI Chat Model (5-mini) | ChatOpenAI | LLM for agent reasoning | (agent wiring) | Run Due Diligence AI Analysis (ai_languageModel) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Parse Structured Analysis JSON | Structured Output Parser | Enforce JSON schema output | (agent wiring) | Run Due Diligence AI Analysis (ai_outputParser) | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Run Due Diligence AI Analysis | LangChain Agent | Multi-query retrieval + synthesis to JSON | Prepare Analysis Context | Map Analysis to Report Fields; Merge Analysis + Report URL | ## AI Due Diligence Analysis\nBuilds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Map Analysis to Report Fields | Code | Convert JSON to HTML template fields | Run Due Diligence AI Analysis | Render DD Report HTML | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Render DD Report HTML | HTML | Build styled HTML report | Map Analysis to Report Fields | Render PDF from HTML | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Render PDF from HTML | Puppeteer | Render HTML → PDF base64 | Render DD Report HTML | Convert PDF Base64 to Binary File | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Convert PDF Base64 to Binary File | Convert to File | Convert base64 → binary PDF | Render PDF from HTML | Prepare S3 File Metadata | ## Report Rendering (HTML to PDF)  \n Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Prepare S3 File Metadata | Code | Timestamped filename + forward binary | Convert PDF Base64 to Binary File | Upload Report PDF to S3 | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Upload Report PDF to S3 | S3 | Upload PDF to bucket | Prepare S3 File Metadata | Build Public Report URL | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Build Public Report URL | Code | Construct public URL from base domain | Upload Report PDF to S3 | Merge Analysis + Report URL | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
|  Merge Analysis + Report URL | Merge | Combine analysis + URL payload | Build Public Report URL; Run Due Diligence AI Analysis | Return API Response | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Return API Response | Respond to Webhook | Return final JSON to caller | Merge Analysis + Report URL | — | ## Delivery & Webhook Response\nPrepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Sticky Note1 | Sticky Note | Block label/comment | — | — | ## Intake & Request Normalization … |
| Sticky Note2 | Sticky Note | Block label/comment | — | — | ## Pinecone Cache Check … |
| Sticky Note3 | Sticky Note | Block label/comment | — | — | ## Document Parsing Loop … |
| Sticky Note4 | Sticky Note | Block label/comment | — | — | ## Vector Ingestion … |
| Sticky Note5 | Sticky Note | Block label/comment | — | — | ## AI Due Diligence Analysis … |
| Sticky Note6 | Sticky Note | Block label/comment | — | — | ## Report Rendering (HTML to PDF) … |
| Sticky Note7 | Sticky Note | Block label/comment | — | — | ## Delivery & Webhook Response … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** named `Due Diligence Automation v2.0.1` (or your preferred name). Ensure it is set to respond via a Respond node (webhook “Respond” mode).

2) **Add Webhook node**: `Receive Upload Request`
   - Type: **Webhook**
   - Method: `POST`
   - Response: `Respond to Webhook` node
   - Path: set to your desired endpoint (e.g. `/dd-ai`).  
   - Save to generate a production/test URL.

3) **Add Code node**: `Split Uploaded Files + Build Deal ID`
   - Paste logic to:
     - Read `item.json.body.filenames` (JSON array string) or fallback to binary filenames
     - Generate `dealId` deterministically from sorted filenames
     - Output one item per binary file in `binary.data*`
   - Connect: `Receive Upload Request` → `Split Uploaded Files + Build Deal ID`

4) **Add HTTP Request node**: `Get Pinecone Index Stats`
   - Method: `POST`
   - URL: use the Pinecone controller or index host endpoint you intend to call for `describe_index_stats`.
     - Prefer a clean expression or fixed URL without stray spaces.
   - Headers:
     - `Content-Type: application/json`
     - Add Pinecone API key header if required (commonly `Api-Key: <key>`), or use an n8n credential mechanism.
   - Connect: `Split Uploaded Files + Build Deal ID` → `Get Pinecone Index Stats`

5) **Add Code node**: `Check Deal Namespace Cache`
   - Implement:
     - Read stats response (`namespaces`)
     - Fetch original split items via `$('Split Uploaded Files + Build Deal ID').all()`
     - If `vectorCount > 0`: output single `{dealId, cacheHit:true, vectorCount}`
     - Else: output all original items with `{cacheHit:false}` and keep binary
   - Connect: `Get Pinecone Index Stats` → `Check Deal Namespace Cache`

6) **Add IF node**: `Cache Hit?`
   - Condition: boolean `{{$json.cacheHit}}` equals `true`
   - True output will go to analysis; false to parsing loop
   - Connect: `Check Deal Namespace Cache` → `Cache Hit?`

7) **Add Code node**: `Prepare Analysis Context`
   - Implement logic to accept:
     - Cache hit object with `dealId`
     - Or aggregated ingestion output containing `deal_id`
   - Output: `{ dealId, filesProcessed, fromCache, timestamp }`
   - Connect: `Cache Hit?` (true) → `Prepare Analysis Context`

8) **Add Split In Batches node**: `Iterate Files for Parsing`
   - Connect: `Cache Hit?` (false) → `Iterate Files for Parsing`

9) **Add HTTP Request node**: `Upload File to LlamaParse`
   - Method: `POST`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
   - Content type: multipart/form-data
   - Body parameter:
     - name `file`, type **binary**, input field `data`
   - Auth: **HTTP Header Auth** credential (store LlamaIndex/LlamaParse API key as required)
   - Connect: `Iterate Files for Parsing` → `Upload File to LlamaParse`

10) **Add HTTP Request node**: `Check LlamaParse Job Status`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`
   - Same header auth
   - Connect: `Upload File to LlamaParse` → `Check LlamaParse Job Status`

11) **Add IF node**: `Is Parsing Job Complete?`
   - Condition: `{{$json.status}}` equals `SUCCESS`
   - True → retrieve result; False → wait and recheck
   - Connect: `Check LlamaParse Job Status` → `Is Parsing Job Complete?`

12) **Add Wait node**: `Wait 10s Before Recheck`
   - Wait: 10 seconds
   - Connect: `Is Parsing Job Complete?` (false) → `Wait 10s Before Recheck` → `Check LlamaParse Job Status`

13) **Add HTTP Request node**: `Retrieve Parsed Content`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`
   - Same auth
   - Connect: `Is Parsing Job Complete?` (true) → `Retrieve Parsed Content`

14) **Add Code node**: `Normalize Parsed Text Payload`
   - Output `{ dealId, sourceFile, fileType, parsedText }` using:
     - loop metadata from `$('Iterate Files for Parsing').item.json`
     - parsed content from `$json.markdown || $json.text`
   - Connect: `Retrieve Parsed Content` → `Normalize Parsed Text Payload`

15) **Add LangChain node**: `Prepare Parsed Text Document`
   - Type: **Document Default Data Loader**
   - Configure metadata fields:
     - `deal_id`, `source_file`, `file_type`, `timestamp`
   - Ensure the node uses the parsed text field as document content (verify in UI mapping).
   - (This node connects into the Pinecone upsert node via the `ai_document` connection.)

16) **Add LangChain node**: `Generate Embeddings (Ingest)`
   - Type: **OpenAI Embeddings**
   - Credentials: OpenAI API key
   - (Connect to Pinecone upsert via `ai_embedding`.)

17) **Add LangChain node**: `Upsert Chunks to Pinecone`
   - Type: **Vector Store Pinecone**
   - Mode: `insert`
   - Index: `poc` (create this index in Pinecone first)
   - Namespace: the `dealId` (commonly `{{$json.dealId}}` or from the iterator context)
   - Credentials: Pinecone API key/host settings
   - Connections into this node:
     - Main: from `Normalize Parsed Text Payload`
     - `ai_document`: from `Prepare Parsed Text Document`
     - `ai_embedding`: from `Generate Embeddings (Ingest)`
   - Connect output back to `Iterate Files for Parsing` to continue batch iteration.

18) **Add Aggregate node**: `Collect Ingested Deal IDs`
   - Aggregate field: `metadata.deal_id`
   - Intended connection: from the upsert completion path (recommended) so the aggregate reflects actual ingests.
   - Connect: `Collect Ingested Deal IDs` → `Prepare Analysis Context`

19) **Add LangChain node**: `Retrieve Context from Pinecone`
   - Type: **Vector Store Pinecone**
   - Mode: `retrieve-as-tool`
   - Namespace: `{{$json.dealId}}`
   - topK: 100 (adjust if needed)
   - Provide tool description requiring multiple queries.

20) **Add LangChain node**: `Generate Embeddings (Retrieval)`
   - Type: **OpenAI Embeddings**
   - Connect via `ai_embedding` to `Retrieve Context from Pinecone`.

21) **Add LangChain node**: `OpenAI Chat Model (5-mini)`
   - Type: **ChatOpenAI**
   - Model: `gpt-5-mini`
   - Credentials: OpenAI API key.

22) **Add LangChain node**: `Parse Structured Analysis JSON`
   - Type: **Structured Output Parser**
   - Provide a JSON schema example matching your required fields.

23) **Add LangChain Agent node**: `Run Due Diligence AI Analysis`
   - Prompt: include mandatory 6-query instruction set and strict JSON-only output requirement.
   - Wire:
     - Main input from `Prepare Analysis Context`
     - `ai_languageModel` from `OpenAI Chat Model (5-mini)`
     - `ai_tool` from `Retrieve Context from Pinecone`
     - `ai_outputParser` from `Parse Structured Analysis JSON`
   - Outputs:
     - To report rendering path
     - To merge node (for including raw analysis in response)

24) **Add Code node**: `Map Analysis to Report Fields`
   - Map structured JSON to template-friendly fields:
     - table rows, list items, missing value handling, escaping
   - Connect: `Run Due Diligence AI Analysis` → `Map Analysis to Report Fields`

25) **Add HTML node**: `Render DD Report HTML`
   - Use a full HTML template referencing `{{$json.companyName}}`, `{{$json.revenueTableRows}}`, etc.
   - Connect: `Map Analysis to Report Fields` → `Render DD Report HTML`

26) **Add Puppeteer node**: `Render PDF from HTML`
   - Operation: run custom script:
     - setContent with timeouts
     - generate PDF
     - return base64
   - Connect: `Render DD Report HTML` → `Render PDF from HTML`

27) **Add Convert to File node**: `Convert PDF Base64 to Binary File`
   - Convert `pdfBase64` to binary
   - Connect: `Render PDF from HTML` → `Convert PDF Base64 to Binary File`

28) **Add Code node**: `Prepare S3 File Metadata`
   - Create `fileName` with timestamp and pass through binary
   - Connect: `Convert PDF Base64 to Binary File` → `Prepare S3 File Metadata`

29) **Add S3 node**: `Upload Report PDF to S3`
   - Operation: upload
   - Bucket: `poc` (create/select)
   - File name: `{{$json.fileName}}`
   - Credentials: AWS access key/secret or IAM role depending on your n8n setup
   - Connect: `Prepare S3 File Metadata` → `Upload Report PDF to S3`

30) **Add Code node**: `Build Public Report URL`
   - Base URL: your domain or CDN mapping to the bucket
   - Use `encodeURIComponent(fileName)`
   - Connect: `Upload Report PDF to S3` → `Build Public Report URL`

31) **Add Merge node**: `Merge Analysis + Report URL`
   - Mode: `combineByPosition`
   - Input 0: `Build Public Report URL`
   - Input 1: second branch from `Run Due Diligence AI Analysis`
   - Connect: `Build Public Report URL` → Merge (input 0) and `Run Due Diligence AI Analysis` → Merge (input 1)

32) **Add Respond to Webhook node**: `Return API Response`
   - Connect: `Merge Analysis + Report URL` → `Return API Response`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automated Due Diligence Report Generator” overview with setup checklist (webhook path `/dd-ai`, OpenAI model `gpt-5-mini`, Pinecone index `poc`, S3 bucket `poc`) | Included as the main sticky note content in the workflow canvas |
| Public URL is built as `https://poc.atlr.dev/<encoded filename>` and assumes that domain serves the S3 objects | Adjust `baseUrl` or configure S3/CloudFront/static hosting accordingly |
| Pinecone stats call likely needs cleanup (URL formatting and Content-Type header) | Review `Get Pinecone Index Stats` node configuration before production use |
| Parsing loop only terminates on `SUCCESS`; failed jobs can loop indefinitely | Add handling for `ERROR/FAILED` statuses and a max retry counter if needed |