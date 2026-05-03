Generate due diligence reports with LlamaIndex, Pinecone, and GPT-5-mini

https://n8nworkflows.xyz/workflows/generate-due-diligence-reports-with-llamaindex--pinecone--and-gpt-5-mini-13501


# Generate due diligence reports with LlamaIndex, Pinecone, and GPT-5-mini

## 1. Workflow Overview

**Workflow name:** Due Diligence Automation v2.0.1  
**Provided title:** Generate due diligence reports with LlamaIndex, Pinecone, and GPT-5-mini

**Purpose:**  
This workflow accepts a multipart file upload (due diligence documents), parses and indexes the content into Pinecone (with caching by “deal namespace”), runs a retrieval-augmented AI analysis using **gpt-5-mini**, and produces a structured due diligence report rendered to **HTML → PDF**, uploaded to **S3**, returning a public link via webhook response.

**Primary use cases**
- Automated due diligence memo generation from uploaded deal documents (PDF/DOCX/etc.)
- Reuse previously-ingested deal documents via Pinecone namespace cache (avoid re-parsing and re-embedding)
- Programmatic API endpoint returning a report URL

### 1.1 Intake & Normalization
Receives a webhook request with multipart binary files and an optional `filenames` list; splits into one item per file and builds a unified `dealId`.

### 1.2 Pinecone Namespace Cache Check
Fetches Pinecone index stats, checks whether a namespace for `dealId` already exists and contains vectors. Routes to:
- **Cache hit:** skip parsing/ingestion and go straight to AI analysis
- **Cache miss:** parse and ingest documents first

### 1.3 Document Parsing Loop (LlamaIndex / LlamaParse)
For each file: upload to LlamaParse, poll until the job completes, then download the parsed markdown and normalize it into a `{dealId, sourceFile, fileType, parsedText}` payload.

### 1.4 Vector Ingestion (Embeddings + Pinecone Upsert)
Turns parsed text into a LangChain document with metadata, generates OpenAI embeddings, upserts vectors to Pinecone under the deal namespace, and aggregates ingestion results.

### 1.5 AI Due Diligence Analysis (RAG Agent)
Builds analysis context, enforces a multi-query retrieval strategy against Pinecone, and uses **gpt-5-mini** to output strict structured JSON.

### 1.6 Report Rendering & Delivery
Maps the structured analysis into report fields, renders HTML, generates a PDF via Puppeteer, uploads to S3, builds a public URL, merges it with analysis output, and responds to the original webhook call.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Intake & Request Normalization
**Overview:** Receives the upload request and converts the multipart payload into separate items (one per file) while generating a deterministic `dealId` based on filenames.  
**Nodes involved:**  
- Receive Upload Request  
- Split Uploaded Files + Build Deal ID  

#### Node: Receive Upload Request
- **Type / role:** `Webhook` — entry point for POST uploads.
- **Configuration (interpreted):**
  - HTTP Method: `POST`
  - Response mode: `responseNode` (workflow must end with Respond to Webhook)
  - Path is currently set to the node’s UUID-like string, not `/dd-ai`.
- **Inputs/Outputs:** Output → Split Uploaded Files + Build Deal ID
- **Edge cases / failures:**
  - If clients post to `/dd-ai` but the node path is not `/dd-ai`, requests will 404.
  - Multipart parsing depends on n8n webhook binary handling; large files may hit limits/timeouts.

#### Node: Split Uploaded Files + Build Deal ID
- **Type / role:** `Code` — normalizes webhook payload into one item per binary file.
- **Key logic / variables:**
  - Reads `$input.first().json.body` and `$input.first().binary`
  - Tries to parse `body.filenames` as JSON array; if it fails, derives names from binary metadata.
  - `dealId` = base64(sorted filenames joined with `|`) → alphanumeric only → first 20 chars.
  - Splits binary keys that start with `data` into separate items.
- **Outputs:** One item per file, each with:
  - `json`: `dealId`, `sourceFile`, `fileType`, `mimeType`, `fileIndex`, `totalFiles`
  - `binary.data`: the file
- **Edge cases / failures:**
  - Throws if no binary keys start with `data` (common if the form field name differs).
  - `dealId` depends only on filenames, not file contents—renames change cache identity; same names with different contents may incorrectly reuse cache.

---

### Block 2.2 — Pinecone Cache Check
**Overview:** Checks Pinecone index stats to detect whether vectors already exist for the `dealId` namespace.  
**Nodes involved:**  
- Get Pinecone Index Stats  
- Check Deal Namespace Cache  
- Cache Hit?  

#### Node: Get Pinecone Index Stats
- **Type / role:** `HTTP Request` — calls Pinecone `describe_index_stats`.
- **Configuration choices:**
  - Method: `POST`
  - URL uses an expression: `=[credentials.pineconeApi.environment] /describe_index_stats`
- **Inputs/Outputs:** Input from Split Uploaded Files… → Output to Check Deal Namespace Cache
- **Potential issues:**
  - The URL expression appears malformed:
    - It includes a space before `/describe_index_stats`
    - `credentials.pineconeApi.environment` may not be a resolvable expression in n8n (depends on how the Pinecone credential is defined/exposed).
  - Header has `Content-Type: "application/json'"` (extra trailing `'`), which can break requests.
- **Failure modes:**
  - 401/403 if Pinecone API key missing/invalid
  - 404 if wrong environment/base URL
  - 400 if headers/body are invalid

#### Node: Check Deal Namespace Cache
- **Type / role:** `Code` — routes execution based on namespace vector count.
- **Key logic / variables:**
  - `stats = $input.first().json`
  - Reads original items from `$('Split Uploaded Files + Build Deal ID').all()`
  - Extracts `dealId` from first original item
  - Checks `stats.namespaces[dealId]` and `vectorCount`
  - If `vectorCount > 0`: returns a single `{dealId, cacheHit:true, vectorCount}`
  - Else: returns all original items, each with `cacheHit:false` and binary preserved
- **Outputs:** → Cache Hit?
- **Edge cases:**
  - If Pinecone stats response shape differs, `stats.namespaces` may be undefined and cache always misses.
  - If `dealId` is undefined (earlier node changed), routing breaks.

#### Node: Cache Hit?
- **Type / role:** `IF` — split path.
- **Condition:** `{{$json.cacheHit}} === true`
- **Outputs:**
  - **True** → Prepare Analysis Context (skip parsing/ingestion)
  - **False** → Iterate Files for Parsing (parse and ingest)

---

### Block 2.3 — Document Parsing Loop (LlamaParse)
**Overview:** For cache-miss paths, processes each file sequentially: upload to LlamaParse, poll until success, then fetch parsed markdown and normalize into consistent text payload.  
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
- **Configuration:** default batching (no explicit batch size shown).
- **Connections:**
  - Main output 0 → Upload File to LlamaParse
  - Main output 1 → Collect Ingested Deal IDs (used as “done” output)
- **Edge cases:**
  - If batch size defaults to 1, it processes one file at a time (good for rate limits).
  - Large numbers of files can increase execution time; webhook might time out if caller expects quick response.

#### Node: Upload File to LlamaParse
- **Type / role:** `HTTP Request` — uploads binary file to LlamaIndex Cloud parsing API.
- **Configuration choices:**
  - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
  - Method: `POST`
  - Content-Type: multipart/form-data
  - Body param: form binary `file` taken from input binary field `data`
  - Auth: HTTP Header Auth credential named `llamaindex`
  - Header `accept: application/json`
- **Outputs:** → Check LlamaParse Job Status
- **Failure modes:**
  - 401 if header auth missing/invalid
  - 413 if file too large for endpoint
  - 429 rate limiting

#### Node: Check LlamaParse Job Status
- **Type / role:** `HTTP Request` — checks parsing job status.
- **Configuration:** GET-like call (method default) to:
  - `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}`
- **Outputs:** → Is Parsing Job Complete?
- **Edge cases:**
  - If upload response does not include `id`, expression fails and URL becomes invalid.

#### Node: Is Parsing Job Complete?
- **Type / role:** `IF` — polling gate.
- **Condition:** `{{$json.status}} == "SUCCESS"`
- **Outputs:**
  - **True** → Retrieve Parsed Content
  - **False** → Wait 10s Before Recheck
- **Edge cases:**
  - If API uses statuses like `SUCCEEDED` or lowercase, polling never completes.

#### Node: Wait 10s Before Recheck
- **Type / role:** `Wait` — delay between poll attempts.
- **Configuration:** waits 10 seconds, then loops back.
- **Outputs:** → Check LlamaParse Job Status
- **Failure modes:**
  - Excessively long parsing jobs can cause very long workflow runtime.

#### Node: Retrieve Parsed Content
- **Type / role:** `HTTP Request` — downloads parsed markdown.
- **Configuration:**
  - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`
  - Header auth: `llamaindex`
  - Header `accept: application/json`
- **Outputs:** → Normalize Parsed Text Payload
- **Edge cases:**
  - If result format changes (e.g., returns `text`), downstream normalization accounts for both `markdown` and `text`.

#### Node: Normalize Parsed Text Payload
- **Type / role:** `Code` — produces a consistent ingestion payload.
- **Key logic / variables:**
  - Uses loop item metadata: `$('Iterate Files for Parsing').item.json`
  - `parsedContent = $json.markdown || $json.text || ''`
  - Output JSON: `dealId`, `sourceFile`, `fileType`, `parsedText`
- **Outputs:** → Upsert Chunks to Pinecone
- **Edge cases:**
  - If parsed content is empty, vectors may be meaningless; should guard/skip empty docs (not implemented).

---

### Block 2.4 — Vector Ingestion
**Overview:** Converts parsed text into a document with metadata, generates embeddings, upserts into Pinecone namespace, and aggregates ingestion metadata.  
**Nodes involved:**  
- Prepare Parsed Text Document  
- Generate Embeddings (Ingest)  
- Upsert Chunks to Pinecone  
- Collect Ingested Deal IDs  

#### Node: Prepare Parsed Text Document
- **Type / role:** `LangChain Document Default Data Loader` — constructs document objects for vector storage.
- **Configuration choices:**
  - Adds metadata fields:
    - `deal_id = {{$json.dealId}}`
    - `source_file = {{$json.sourceFile}}`
    - `file_type = {{$json.fileType}}`
    - `timestamp = {{$now.toUTC()}}`
- **Connections:** Provides `ai_document` input to Upsert Chunks to Pinecone
- **Edge cases:**
  - Assumes downstream node can access document text from the incoming item; ensure parsedText is recognized by this loader’s defaults (n8n node behavior/version dependent).

#### Node: Generate Embeddings (Ingest)
- **Type / role:** `OpenAI Embeddings` — embeddings for ingestion.
- **Credentials:** OpenAI credential “Sumopod”
- **Connections:** Provides `ai_embedding` to Upsert Chunks to Pinecone
- **Failure modes:**
  - 401 invalid OpenAI key
  - Model/endpoint issues, rate limits, timeouts
  - Cost scaling with document size

#### Node: Upsert Chunks to Pinecone
- **Type / role:** `Vector Store: Pinecone` — inserts vectors.
- **Configuration choices:**
  - Mode: `insert`
  - Index: `poc`
  - Namespace: `{{$('Iterate Files for Parsing').item.json.dealId}}`
  - `clearNamespace: false` (appends; does not wipe prior vectors)
- **Connections:** After upsert → Iterate Files for Parsing (to continue batch loop)
- **Edge cases / risks:**
  - If re-ingesting same dealId, duplicates may accumulate (since `clearNamespace` is false and there’s no idempotent vector IDs shown).
  - Namespace expression depends on SplitInBatches context; if run outside loop, may be undefined.

#### Node: Collect Ingested Deal IDs
- **Type / role:** `Aggregate` — collects ingestion results.
- **Configuration:** Aggregates field `metadata.deal_id`
- **Connections:** → Prepare Analysis Context
- **Edge cases:**
  - If the upstream upsert output doesn’t include `metadata.deal_id`, aggregation will be empty.

---

### Block 2.5 — AI Due Diligence Analysis
**Overview:** Creates a clean analysis context, uses a tool-enabled agent with strict instructions to query Pinecone multiple times, and enforces a structured JSON output schema.  
**Nodes involved:**  
- Prepare Analysis Context  
- Retrieve Context from Pinecone  
- Generate Embeddings (Retrieval)  
- OpenAI Chat Model (5-mini)  
- Parse Structured Analysis JSON  
- Run Due Diligence AI Analysis  

#### Node: Prepare Analysis Context
- **Type / role:** `Code` — normalizes analysis entry whether from cache hit or ingest path.
- **Key logic:**
  - If `items[0].json.cacheHit === true`: use `items[0].json.dealId`
  - Else: reads aggregated `deal_id` array and uses first element
  - Outputs: `dealId`, `filesProcessed`, `fromCache`, `timestamp`
- **Outputs:** → Run Due Diligence AI Analysis
- **Edge cases:**
  - If aggregate output shape differs, `dealId` may become `'unknown'`.

#### Node: Retrieve Context from Pinecone
- **Type / role:** `Vector Store: Pinecone` in `retrieve-as-tool` mode — exposes retrieval as an agent tool.
- **Configuration choices:**
  - Index: `poc`
  - Namespace: `{{$json.dealId}}`
  - `topK: 100`
  - Tool description explicitly requires multiple focused queries.
- **Connections:** Provides `ai_tool` to Run Due Diligence AI Analysis
- **Failure modes:**
  - Namespace not found (returns empty context)
  - Authentication issues to Pinecone
  - topK=100 may increase latency and token usage depending on chunk sizes

#### Node: Generate Embeddings (Retrieval)
- **Type / role:** `OpenAI Embeddings` for retrieval queries.
- **Credentials:** OpenAI “Sumopod”
- **Connections:** Provides `ai_embedding` to Retrieve Context from Pinecone
- **Failure modes:** same as ingestion embeddings

#### Node: OpenAI Chat Model (5-mini)
- **Type / role:** `Chat Model` — LLM for the agent.
- **Configuration:**
  - Model: `gpt-5-mini`
  - No extra options configured
- **Connections:** Provides `ai_languageModel` to Run Due Diligence AI Analysis
- **Failure modes:**
  - Model unavailable in the account/region
  - Rate limits / token limits

#### Node: Parse Structured Analysis JSON
- **Type / role:** `Structured Output Parser` — enforces JSON schema-like structure.
- **Configuration:** Example schema includes:
  - `company_profile` (name, industry, location, employee_count)
  - `financials` (revenue_history array, ebitda, margins)
  - `analysis` (business_model, investment_thesis, key_risks array, customer_concentration)
- **Connections:** Provides `ai_outputParser` to Run Due Diligence AI Analysis
- **Edge cases:**
  - If the model returns invalid JSON, parsing will fail and stop the workflow.

#### Node: Run Due Diligence AI Analysis
- **Type / role:** `LangChain Agent` — orchestrates retrieval and synthesis.
- **Configuration choices:**
  - Prompt type: “define”
  - Has output parser: true
  - Prompt mandates **6 required Pinecone queries** before output, strict non-hallucination rules, missing data handling, and “Return ONLY valid JSON”.
- **Connections (main):**
  - Output 0 → Map Analysis to Report Fields
  - Output 1 → Merge Analysis + Report URL (used later to merge analysis with final URL)
- **Edge cases / failure modes:**
  - The agent cannot be forced deterministically to call tools exactly 6 times; it’s instruction-based. Consider adding tooling constraints or validation if strict compliance is required.
  - If Pinecone retrieval returns insufficient evidence, output may contain many “Not Available” fields.

---

### Block 2.6 — Report Rendering (HTML → PDF)
**Overview:** Converts structured analysis JSON into template-ready fields, renders HTML, generates a PDF, and produces a binary file for upload.  
**Nodes involved:**  
- Map Analysis to Report Fields  
- Render DD Report HTML  
- Render PDF from HTML  
- Convert PDF Base64 to Binary File  

#### Node: Map Analysis to Report Fields
- **Type / role:** `Code` — transforms analysis JSON into fields consumed by the HTML template.
- **Key expressions / variables:**
  - Reads `raw.output` if present (agent-style output), otherwise uses `raw`
  - Pulls `dealId` from `$('Prepare Analysis Context').first().json.dealId`
  - Uses `DateTime.now()` (Luxon in n8n Code node runtime) to format report date
  - Sanitizes/escapes HTML for table and risk list entries
  - Converts numeric fields with formatting; treats `0` as “Not Available” for display
- **Outputs:** → Render DD Report HTML
- **Edge cases:**
  - If Luxon `DateTime` is not available in your n8n code runtime, this will throw (many n8n builds provide it, but not universally).
  - Treating numeric `0` as “Not Available” may hide legitimate zeros.

#### Node: Render DD Report HTML
- **Type / role:** `HTML` — template rendering.
- **Configuration:** Large inline HTML/CSS template using `{{ $json.field }}` placeholders.
- **Outputs:** → Render PDF from HTML
- **Edge cases:**
  - If any required field is missing, template still renders but may show “Not Available”.

#### Node: Render PDF from HTML
- **Type / role:** `Puppeteer` — generates PDF by running a custom script.
- **Key logic:**
  - Reads `$json.html`
  - `page.setContent(... waitUntil: networkidle0, timeout: 60000)`
  - `page.pdf({format:'A4', printBackground:true, margins..., scale:0.95, timeout:60000})`
  - Returns `{ pdfBase64 }`
- **Outputs:** → Convert PDF Base64 to Binary File
- **Failure modes:**
  - Puppeteer node requires a compatible runtime (Chromium dependencies in container/host).
  - Large HTML or slow resource loading can exceed timeouts.

#### Node: Convert PDF Base64 to Binary File
- **Type / role:** `Convert to File` — converts base64 string into binary.
- **Configuration:**
  - Operation: toBinary
  - Source property: `pdfBase64`
  - File name: `{{ $('Map Analysis to Report Fields').item.json.companyName }}-Analysis.pdf`
- **Outputs:** → Prepare S3 File Metadata
- **Edge cases:**
  - If company name contains characters not safe for filenames/URLs, later steps may need stronger normalization.

---

### Block 2.7 — Delivery & Webhook Response
**Overview:** Creates an S3-safe name, uploads the PDF to S3, constructs a public URL, merges with analysis data, and responds to the original webhook.  
**Nodes involved:**  
- Prepare S3 File Metadata  
- Upload Report PDF to S3  
- Build Public Report URL  
- Merge Analysis + Report URL  
- Return API Response  

#### Node: Prepare S3 File Metadata
- **Type / role:** `Code` — sets final PDF filename for S3 upload.
- **Key logic:**
  - `companyName = $('Map Analysis to Report Fields').first().json.companyName`
  - `fileName = `${companyName}-assessment-${Date.now()}.pdf``
  - Passes through binary from input
- **Outputs:** → Upload Report PDF to S3
- **Edge cases:**
  - Company name may include slashes or other characters that create unintended S3 “folders” or invalid URLs.

#### Node: Upload Report PDF to S3
- **Type / role:** `S3` — uploads binary to bucket.
- **Configuration:**
  - Operation: upload
  - Bucket: `poc`
  - File name: `{{$json.fileName}}`
- **Outputs:** → Build Public Report URL
- **Failure modes:**
  - Missing/invalid AWS credentials
  - Bucket doesn’t exist or policy denies PutObject
  - Upload size limits/timeouts

#### Node: Build Public Report URL
- **Type / role:** `Code` — constructs an externally accessible URL.
- **Key logic:**
  - `baseUrl = 'https://poc.atlr.dev'`
  - Reads `fileName` from `$('Prepare S3 File Metadata').first().json.fileName`
  - `publicUrl = `${baseUrl}/${encodeURIComponent(fileName)}``
  - Returns `{success:true, fileName, publicUrl}`
- **Outputs:** → Merge Analysis + Report URL
- **Critical dependency:** This assumes `https://poc.atlr.dev/<filename>` is publicly serving the S3 object (via CDN, reverse proxy, or static site). S3 upload alone does not guarantee public access.
- **Edge cases:**
  - If distribution/public mapping differs (e.g., needs path prefix), URL will be wrong.

#### Node: Merge Analysis + Report URL
- **Type / role:** `Merge` — combines analysis output with URL output.
- **Configuration:** `combineByPosition`
- **Inputs:**
  - Input 0: from Build Public Report URL
  - Input 1: from Run Due Diligence AI Analysis (secondary output)
- **Output:** → Return API Response
- **Edge cases:**
  - `combineByPosition` is fragile if either side produces multiple items or none; results can mismatch.

#### Node: Return API Response
- **Type / role:** `Respond to Webhook` — sends the final HTTP response.
- **Configuration:** default options
- **Input:** merged JSON
- **Edge cases:**
  - If earlier nodes error after webhook received, caller gets an error/timeout.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Upload Request | Webhook | Receives multipart upload and starts workflow | — | Split Uploaded Files + Build Deal ID | ## Intake & Request Normalization  ; Receives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Split Uploaded Files + Build Deal ID | Code | Splits binary files into items; generates deterministic dealId | Receive Upload Request | Get Pinecone Index Stats | ## Intake & Request Normalization  ; Receives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Get Pinecone Index Stats | HTTP Request | Calls Pinecone describe_index_stats | Split Uploaded Files + Build Deal ID | Check Deal Namespace Cache | ## Pinecone Cache Check ; Checks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Check Deal Namespace Cache | Code | Determines cache hit/miss and routes items | Get Pinecone Index Stats | Cache Hit? | ## Pinecone Cache Check ; Checks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Cache Hit? | IF | Branches: cache hit → analysis, miss → parsing | Check Deal Namespace Cache | Prepare Analysis Context; Iterate Files for Parsing | ## Pinecone Cache Check ; Checks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Iterate Files for Parsing | Split In Batches | Loops through uploaded files for parsing/ingestion | Cache Hit? (false path) ; Upsert Chunks to Pinecone | Upload File to LlamaParse; Collect Ingested Deal IDs |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Upload File to LlamaParse | HTTP Request | Uploads one file to LlamaParse | Iterate Files for Parsing | Check LlamaParse Job Status |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Check LlamaParse Job Status | HTTP Request | Polls job status | Upload File to LlamaParse; Wait 10s Before Recheck | Is Parsing Job Complete? |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Is Parsing Job Complete? | IF | Checks for SUCCESS status | Check LlamaParse Job Status | Retrieve Parsed Content; Wait 10s Before Recheck |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Wait 10s Before Recheck | Wait | Delay between polling attempts | Is Parsing Job Complete? (false path) | Check LlamaParse Job Status |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Retrieve Parsed Content | HTTP Request | Fetches parsed markdown result | Is Parsing Job Complete? (true path) | Normalize Parsed Text Payload |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Normalize Parsed Text Payload | Code | Creates normalized `{dealId, sourceFile, parsedText}` payload | Retrieve Parsed Content | Upsert Chunks to Pinecone |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Prepare Parsed Text Document | LangChain Document Loader | Adds metadata and constructs document for vector store | (implicit in-chain) Normalize Parsed Text Payload | Upsert Chunks to Pinecone (ai_document) |  ## Vector Ingestion  ; Converts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Generate Embeddings (Ingest) | OpenAI Embeddings | Generates embeddings for ingestion | (implicit in-chain) Normalize Parsed Text Payload | Upsert Chunks to Pinecone (ai_embedding) |  ## Vector Ingestion  ; Converts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Upsert Chunks to Pinecone | Pinecone Vector Store | Upserts vectors into Pinecone namespace | Normalize Parsed Text Payload; Prepare Parsed Text Document; Generate Embeddings (Ingest) | Iterate Files for Parsing |  ## Vector Ingestion  ; Converts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Collect Ingested Deal IDs | Aggregate | Aggregates `metadata.deal_id` after loop completion | Iterate Files for Parsing (done output) | Prepare Analysis Context |  ## Vector Ingestion  ; Converts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Prepare Analysis Context | Code | Normalizes analysis entry data for cache hit/miss | Cache Hit? (true path); Collect Ingested Deal IDs | Run Due Diligence AI Analysis | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Retrieve Context from Pinecone | Pinecone Vector Store | Retrieval tool for agent (RAG) | (tool call by agent) | Run Due Diligence AI Analysis (ai_tool) | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Generate Embeddings (Retrieval) | OpenAI Embeddings | Embeddings for retrieval queries | (tool chain) | Retrieve Context from Pinecone (ai_embedding) | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
|  OpenAI Chat Model (5-mini) | OpenAI Chat Model | LLM powering the agent | — | Run Due Diligence AI Analysis (ai_languageModel) | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Parse Structured Analysis JSON | Structured Output Parser | Enforces valid JSON output schema | — | Run Due Diligence AI Analysis (ai_outputParser) | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Run Due Diligence AI Analysis | LangChain Agent | Executes multi-query retrieval + synthesis into structured JSON | Prepare Analysis Context; (ai connections above) | Map Analysis to Report Fields; Merge Analysis + Report URL | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Map Analysis to Report Fields | Code | Maps structured JSON to HTML template fields | Run Due Diligence AI Analysis | Render DD Report HTML | ## Report Rendering (HTML to PDF)  ;  Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Render DD Report HTML | HTML | Renders report HTML using mapped fields | Map Analysis to Report Fields | Render PDF from HTML | ## Report Rendering (HTML to PDF)  ;  Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Render PDF from HTML | Puppeteer | Generates PDF from HTML | Render DD Report HTML | Convert PDF Base64 to Binary File | ## Report Rendering (HTML to PDF)  ;  Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Convert PDF Base64 to Binary File | Convert to File | Converts base64 PDF to binary | Render PDF from HTML | Prepare S3 File Metadata | ## Report Rendering (HTML to PDF)  ;  Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Prepare S3 File Metadata | Code | Creates upload filename; passes binary through | Convert PDF Base64 to Binary File | Upload Report PDF to S3 | ## Delivery & Webhook Response ; Prepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Upload Report PDF to S3 | S3 | Uploads PDF to S3 bucket | Prepare S3 File Metadata | Build Public Report URL | ## Delivery & Webhook Response ; Prepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Build Public Report URL | Code | Constructs public URL for uploaded report | Upload Report PDF to S3 | Merge Analysis + Report URL | ## Delivery & Webhook Response ; Prepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
|  Merge Analysis + Report URL | Merge | Combines analysis JSON with public URL | Build Public Report URL; Run Due Diligence AI Analysis | Return API Response | ## Delivery & Webhook Response ; Prepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Return API Response | Respond to Webhook | Returns final response to caller | Merge Analysis + Report URL | — | ## Delivery & Webhook Response ; Prepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |
| Sticky Note | Sticky Note | Documentation block | — | — | ## Automated Due Diligence Report Generator ; (content includes setup checklist and flow summary) |
| Sticky Note1 | Sticky Note | Documentation block | — | — | ## Intake & Request Normalization  ; Receives multipart upload, parses filenames/binaries, and creates one normalized item per file with a shared dealId. |
| Sticky Note2 | Sticky Note | Documentation block | — | — | ## Pinecone Cache Check ; Checks namespace stats in Pinecone to detect cache hit/miss and routes flow to direct analysis or document parsing. |
| Sticky Note3 | Sticky Note | Documentation block | — | — |  ## Document Parsing Loop ; Uploads each file for parsing, polls async status until complete, fetches markdown output, and normalizes parsed text payload. |
| Sticky Note4 | Sticky Note | Documentation block | — | — |  ## Vector Ingestion  ; Converts parsed content into documents, generates embeddings, upserts vectors into deal namespace, and aggregates ingestion results. |
| Sticky Note5 | Sticky Note | Documentation block | — | — | ## AI Due Diligence Analysis ; Builds analysis context, retrieves supporting evidence from Pinecone, runs LLM agent, and enforces structured JSON output. |
| Sticky Note6 | Sticky Note | Documentation block | — | — | ## Report Rendering (HTML to PDF)  ;  Maps AI output to report fields, renders HTML template, generates PDF, and converts output to binary file. |
| Sticky Note7 | Sticky Note | Documentation block | — | — | ## Delivery & Webhook Response ; Prepares filename metadata, uploads PDF to S3, builds public URL, merges outputs, and returns final API response. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: `Due Diligence Automation v2.0.1` (or your preferred name)
   - Ensure workflow settings allow webhook execution and adequate timeout for long runs.

2) **Add Webhook node**
   - Node: **Webhook** → name it `Receive Upload Request`
   - Method: `POST`
   - Path: set to `/dd-ai` (recommended)  
   - Response mode: `Using 'Respond to Webhook' node`

3) **Add Code node to split files**
   - Node: **Code** → `Split Uploaded Files + Build Deal ID`
   - Paste logic that:
     - reads webhook `binary`
     - parses optional `body.filenames`
     - creates deterministic `dealId`
     - outputs one item per binary file with `binary.data`

4) **Add Pinecone stats request**
   - Node: **HTTP Request** → `Get Pinecone Index Stats`
   - Configure Pinecone base URL correctly for your deployment (example patterns vary by Pinecone version):
     - For Pinecone serverless, you typically use the index host URL; for classic, environment-based host.
   - Endpoint operation: `POST /describe_index_stats`
   - Headers:
     - `Content-Type: application/json` (no trailing quote)
   - Auth:
     - Either use a Header Auth credential (e.g., `Api-Key: <PINECONE_KEY>`) or configure per your Pinecone setup.
   - Connect: Webhook split node → Pinecone stats node

5) **Add Code node for cache check**
   - Node: **Code** → `Check Deal Namespace Cache`
   - Implement:
     - read `dealId` from the original split items
     - check `stats.namespaces[dealId].vectorCount`
     - output either a single cache-hit item or all original items with binary

6) **Add IF node for cache branching**
   - Node: **IF** → `Cache Hit?`
   - Condition: Boolean equals `{{$json.cacheHit}}` is `true`
   - True output → later “Prepare Analysis Context”
   - False output → document parsing loop start

7) **Add Split In Batches for parsing loop**
   - Node: **Split In Batches** → `Iterate Files for Parsing`
   - Batch size: 1 (recommended to reduce rate-limit risk)
   - Connect: Cache Hit? (false) → Iterate Files for Parsing

8) **Add LlamaParse upload request**
   - Node: **HTTP Request** → `Upload File to LlamaParse`
   - Method: `POST`
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`
   - Content type: `multipart/form-data`
   - Body: form field `file` from binary property `data`
   - Auth: **Header Auth** credential containing LlamaIndex Cloud API key/header required
   - Connect: Iterate Files for Parsing → Upload File to LlamaParse

9) **Add status polling**
   - Node: **HTTP Request** → `Check LlamaParse Job Status`
     - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}`
     - Same auth
   - Node: **IF** → `Is Parsing Job Complete?`
     - Condition: `{{$json.status}} == "SUCCESS"`
   - Node: **Wait** → `Wait 10s Before Recheck` (10 seconds)
   - Connect:
     - Upload → Status
     - Status → IF
     - IF false → Wait → Status (loop)

10) **Add retrieve parsed content**
   - Node: **HTTP Request** → `Retrieve Parsed Content`
     - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{$json.id}}/result/markdown`
     - Same auth
   - Connect: IF true → Retrieve Parsed Content

11) **Normalize parsed text**
   - Node: **Code** → `Normalize Parsed Text Payload`
   - Output JSON: `{dealId, sourceFile, fileType, parsedText}`
   - Connect: Retrieve Parsed Content → Normalize

12) **Vector ingestion (LangChain)**
   - Node: **Document Default Data Loader** → `Prepare Parsed Text Document`
     - Configure metadata fields: deal_id, source_file, file_type, timestamp
   - Node: **OpenAI Embeddings** → `Generate Embeddings (Ingest)`
     - Credential: OpenAI API key
   - Node: **Pinecone Vector Store** → `Upsert Chunks to Pinecone`
     - Mode: `Insert`
     - Index name: `poc` (create in Pinecone first)
     - Namespace: `{{$('Iterate Files for Parsing').item.json.dealId}}`
     - Do not clear namespace (or set clear=true if you want overwrite behavior)
   - Connect:
     - Normalize (main) → Upsert (main)
     - Prepare Parsed Text Document (ai_document) → Upsert
     - Generate Embeddings (ai_embedding) → Upsert

13) **Complete the loop**
   - Connect: Upsert Chunks to Pinecone → Iterate Files for Parsing (to continue batches)
   - Connect: Iterate Files for Parsing “done/second output” → Aggregate node (next step)

14) **Aggregate ingestion results**
   - Node: **Aggregate** → `Collect Ingested Deal IDs`
   - Aggregate field: `metadata.deal_id`
   - Connect: Iterate done output → Aggregate

15) **Prepare analysis context**
   - Node: **Code** → `Prepare Analysis Context`
   - Logic: handle cache hit item vs aggregate output; output `{dealId,...}`
   - Connect:
     - Cache Hit? true → Prepare Analysis Context
     - Aggregate → Prepare Analysis Context

16) **Set up AI analysis agent**
   - Node: **OpenAI Chat Model** → `OpenAI Chat Model (5-mini)`
     - Model: `gpt-5-mini`
     - Credential: OpenAI
   - Node: **OpenAI Embeddings** → `Generate Embeddings (Retrieval)`
   - Node: **Pinecone Vector Store** → `Retrieve Context from Pinecone`
     - Mode: `Retrieve as tool`
     - Index: `poc`
     - Namespace: `{{$json.dealId}}`
     - topK: 100
     - Tool description: include required multi-query instructions
   - Node: **Structured Output Parser** → `Parse Structured Analysis JSON`
     - Provide your JSON example/schema
   - Node: **Agent** → `Run Due Diligence AI Analysis`
     - Prompt: include strict rules and required queries
     - Attach:
       - Language model input from `OpenAI Chat Model (5-mini)`
       - Tool from `Retrieve Context from Pinecone`
       - Output parser from `Parse Structured Analysis JSON`
   - Connect: Prepare Analysis Context → Agent

17) **Map analysis into report fields**
   - Node: **Code** → `Map Analysis to Report Fields`
   - Implement HTML escaping, revenue rows, risk list, and `reportDate`.
   - Connect: Agent main output → Map Analysis

18) **Render HTML**
   - Node: **HTML** → `Render DD Report HTML`
   - Paste the full HTML template with placeholders.
   - Connect: Map Analysis → Render HTML

19) **Generate PDF**
   - Node: **Puppeteer** → `Render PDF from HTML`
   - Operation: run custom script
   - Script:
     - `page.setContent(html, waitUntil networkidle0, timeout 60000)`
     - `page.pdf(...)`
     - return `{pdfBase64}`
   - Connect: Render HTML → Puppeteer

20) **Convert PDF to binary**
   - Node: **Convert to File** → `Convert PDF Base64 to Binary File`
   - Operation: `To Binary`
   - Source property: `pdfBase64`
   - Filename expression: `{{$('Map Analysis to Report Fields').item.json.companyName}}-Analysis.pdf`
   - Connect: Puppeteer → Convert to File

21) **Prepare S3 filename**
   - Node: **Code** → `Prepare S3 File Metadata`
   - Set `fileName = <company>-assessment-<timestamp>.pdf`
   - Pass through binary
   - Connect: Convert to File → Prepare S3 File Metadata

22) **Upload to S3**
   - Node: **S3** → `Upload Report PDF to S3`
   - Operation: `Upload`
   - Bucket: `poc` (create/select)
   - File name: `{{$json.fileName}}`
   - Credential: AWS (access key/secret or IAM role)
   - Connect: Prepare S3 File Metadata → S3 upload

23) **Build public URL**
   - Node: **Code** → `Build Public Report URL`
   - Base URL: `https://poc.atlr.dev` (adjust to your CDN/domain)
   - public URL: `${baseUrl}/${encodeURIComponent(fileName)}`
   - Connect: S3 upload → URL builder

24) **Merge analysis + URL**
   - Node: **Merge** → `Merge Analysis + Report URL`
   - Mode: `Combine by position`
   - Connect:
     - URL builder → Merge input 0
     - Agent secondary output (as in your design) → Merge input 1  
       *(Alternatively, merge agent output 0 if you want the same payload used for mapping.)*

25) **Respond to webhook**
   - Node: **Respond to Webhook** → `Return API Response`
   - Connect: Merge → Respond

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Automated Due Diligence Report Generator** — Flow summary and setup checklist (webhook path `/dd-ai`, OpenAI `gpt-5-mini`, Pinecone index `poc`, LlamaIndex parsing creds, S3 bucket `poc`, multipart test instructions). | From workflow sticky note “Automated Due Diligence Report Generator” |
| Pinecone stats request likely needs correction (URL composition and `Content-Type` header has an extra `'`). | Integration reliability note (Pinecone HTTP Request node) |
| Public URL construction assumes an external domain serves the uploaded S3 objects (S3 upload alone does not guarantee public accessibility). | Delivery block dependency (`https://poc.atlr.dev`) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.