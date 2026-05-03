Build a Google Drive internal knowledge base with OpenAI and Pinecone

https://n8nworkflows.xyz/workflows/build-a-google-drive-internal-knowledge-base-with-openai-and-pinecone-13959


# Build a Google Drive internal knowledge base with OpenAI and Pinecone

# 1. Workflow Overview

This workflow builds and maintains an internal AI knowledge base from Google Drive documents, stores semantic embeddings in Pinecone, and exposes a webhook endpoint for question answering using OpenAI. It also includes a scheduled refresh process to detect new or updated documents and re-ingest them automatically.

Primary use cases:
- Index internal company documents from Google Drive
- Answer employee questions through semantic search + grounded LLM responses
- Keep the vector knowledge base synchronized weekly
- Log ingestion and Q&A activity in Google Sheets
- Notify operators by email when ingestion or weekly maintenance completes

The workflow is organized into three main operational branches plus documentation notes:

## 1.1 Initial Manual Ingestion
A manually triggered branch reads all documents in a specific Google Drive folder, downloads them as text, splits them into chunks, generates embeddings, stores them in Pinecone, logs ingestion in a Google Sheet, and sends a completion email.

## 1.2 Question Answering via Webhook
A webhook accepts a POST request containing a question and optional user identity, performs semantic search in Pinecone, builds a grounded RAG prompt, asks OpenAI to answer strictly from retrieved context, parses the structured answer, logs the Q&A in Google Sheets, and returns JSON to the caller.

## 1.3 Weekly Sync and Re-ingestion
A weekly schedule checks the Google Drive folder against a document registry sheet, detects new or updated documents, re-ingests only those files into Pinecone, updates the registry, and sends a weekly digest email. If no new documents are found, it still sends a digest through an alternate branch.

## 1.4 In-Canvas Documentation
Several sticky notes explain setup, workflow purpose, and branch behavior. These notes are operationally important because they describe assumptions such as Pinecone index dimensions, webhook usage, and maintenance expectations.

---

# 2. Block-by-Block Analysis

## 2.1 Initial Manual Ingestion

### Overview
This block performs a full initial ingestion of all Google Docs located in the designated knowledge base folder. It converts files to plain text, chunks content, generates embeddings, inserts vectors into Pinecone, records document ingestion in a registry sheet, and sends a single completion email.

### Nodes Involved
- Manual Ingestion Trigger
- List Knowledge Base Docs
- Download Doc as Text
- Generate Embeddings
- Split Into Chunks
- Load Document Data
- Store Embeddings in Pinecone
- Log to Document Registry
- Notify Ingestion Complete

### Node Details

#### Manual Ingestion Trigger
- **Type and role:** `n8n-nodes-base.manualTrigger`; manual entry point for full ingestion.
- **Configuration choices:** No parameters; intended for operator-initiated runs.
- **Key expressions or variables:** None.
- **Input and output connections:** Starts the branch; outputs to **List Knowledge Base Docs**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** None intrinsic; branch depends on downstream credentials and folder configuration.
- **Sub-workflow reference:** None.

#### List Knowledge Base Docs
- **Type and role:** `n8n-nodes-base.googleDrive`; lists files/folders from a specific Google Drive folder.
- **Configuration choices:** Resource is `fileFolder`; filter by folder ID pointing to a folder named **Knowledge Base**.
- **Key expressions or variables:** Uses a fixed folder ID, not a dynamic expression.
- **Input and output connections:** Receives from **Manual Ingestion Trigger**; outputs one item per Drive file to **Download Doc as Text**.
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:**
  - Google OAuth permission issues
  - Folder not found / wrong folder ID
  - Mixed file types in folder may affect downstream conversion
- **Sub-workflow reference:** None.

#### Download Doc as Text
- **Type and role:** `n8n-nodes-base.googleDrive`; downloads each file and converts Google Docs to plain text.
- **Configuration choices:** Operation `download`; `fileId` from current item `{{$json.id}}`; Google file conversion `docsToFormat: text/plain`.
- **Key expressions or variables:** `={{ $json.id }}`
- **Input and output connections:** Receives file list from **List Knowledge Base Docs**; sends binary/text output to **Store Embeddings in Pinecone**.
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:**
  - Non-Google-Docs files may not convert as expected
  - Empty or inaccessible documents
  - Conversion limitations for unsupported formats
- **Sub-workflow reference:** None.

#### Generate Embeddings
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; provides embedding model for vectorization.
- **Configuration choices:** Uses OpenAI credentials; no custom options set.
- **Key expressions or variables:** None directly.
- **Input and output connections:** Supplies `ai_embedding` input to **Store Embeddings in Pinecone**.
- **Version-specific requirements:** Type version 1.2; requires LangChain-compatible n8n AI nodes and valid OpenAI API access.
- **Edge cases / failures:**
  - Invalid OpenAI credentials
  - API quota/rate limits
  - Embedding model changes affecting Pinecone dimension compatibility
- **Sub-workflow reference:** None.

#### Split Into Chunks
- **Type and role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`; splits document text into retrieval chunks.
- **Configuration choices:** `chunkSize: 500`, `chunkOverlap: 100`.
- **Key expressions or variables:** None.
- **Input and output connections:** Sends `ai_textSplitter` into **Load Document Data**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:**
  - Very short docs may produce minimal chunking
  - Very large docs may create many chunks and increase cost/storage
- **Sub-workflow reference:** None.

#### Load Document Data
- **Type and role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`; loads binary file data and attaches metadata for ingestion.
- **Configuration choices:** `dataType: binary`; `textSplittingMode: custom`; metadata field `doc_name` set from the downloaded file name.
- **Key expressions or variables:** `={{ $('Download Doc as Text').item.json.name }}`
- **Input and output connections:** Receives `ai_textSplitter` from **Split Into Chunks**; sends `ai_document` to **Store Embeddings in Pinecone**.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases / failures:**
  - Missing binary payload from download
  - Expression lookup failure if item linking breaks
  - Metadata omission can weaken source attribution later
- **Sub-workflow reference:** None.

#### Store Embeddings in Pinecone
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`; inserts chunk embeddings and metadata into Pinecone.
- **Configuration choices:** Mode `insert`; namespace `Test Team`; index `meeting-sentiment-index`.
- **Key expressions or variables:** Namespace is fixed; embeddings and documents come from connected AI inputs.
- **Input and output connections:**
  - Main input from **Download Doc as Text**
  - `ai_embedding` from **Generate Embeddings**
  - `ai_document` from **Load Document Data**
  - Main output to **Log to Document Registry**
- **Version-specific requirements:** Type version 1.3; Pinecone index must exist and match OpenAI embedding dimensionality.
- **Edge cases / failures:**
  - Pinecone auth/index/namespace errors
  - Index dimension mismatch
  - Duplicate re-inserts may create redundant vectors because no delete/upsert strategy is configured
- **Sub-workflow reference:** None.

#### Log to Document Registry
- **Type and role:** `n8n-nodes-base.googleSheets`; appends ingestion status to a tracking sheet.
- **Configuration choices:** Append to sheet **Document Registry** in spreadsheet **Internal Knowledge Brain automation**. Writes:
  - `doc_id` from **Download Doc as Text**
  - `status` = `ingested`
  - `doc_name` from metadata
  - `last_ingested` = current date in `YYYY-MM-DD`
- **Key expressions or variables:**
  - `={{ $('Download Doc as Text').item.json.id }}`
  - `={{ $json.metadata.doc_name }}`
  - `={{ new Date().toISOString().split('T')[0] }}`
- **Input and output connections:** Receives from **Store Embeddings in Pinecone**; outputs to **Notify Ingestion Complete**.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases / failures:**
  - Append-only design creates multiple rows for the same document over time
  - Sheet schema mismatch or missing columns
  - OAuth permission issues
- **Sub-workflow reference:** None.

#### Notify Ingestion Complete
- **Type and role:** `n8n-nodes-base.gmail`; sends completion email after ingestion.
- **Configuration choices:** Sends plain-text email to placeholder `your_email_id`; subject `✅ Knowledge Base Updated`; configured `executeOnce: true`.
- **Key expressions or variables:** Date inserted with `{{ new Date().toISOString().split('T')[0] }}`
- **Input and output connections:** Receives from **Log to Document Registry**; no downstream nodes.
- **Version-specific requirements:** Type version 2.2.
- **Edge cases / failures:**
  - Invalid Gmail OAuth
  - Placeholder recipient not replaced
  - Since `executeOnce` is enabled, one email is sent for the overall run rather than per ingested document
- **Sub-workflow reference:** None.

---

## 2.2 Question Answering via Webhook

### Overview
This block exposes a POST webhook endpoint that accepts a user question, performs semantic retrieval against Pinecone, generates a grounded answer using OpenAI, logs the interaction, and returns a structured JSON response.

### Nodes Involved
- Question Webhook
- Extract Question & User
- Generate Query Embedding
- Search Knowledge Base
- Build RAG Prompt
- Answer Generator
- Parse AI Answer
- Log Q&A to Sheets
- Send Answer Response

### Node Details

#### Question Webhook
- **Type and role:** `n8n-nodes-base.webhook`; public API entry point for asking questions.
- **Configuration choices:** Path `knowledge-brain`; HTTP method `POST`; response mode `responseNode`.
- **Key expressions or variables:** Expects request body fields such as `question` and optionally `asked_by`.
- **Input and output connections:** Starts the branch; outputs to **Extract Question & User**.
- **Version-specific requirements:** Type version 2.1.
- **Edge cases / failures:**
  - If workflow is inactive, production webhook is unavailable
  - Missing `question` field causes downstream issues
  - Malformed JSON request body
- **Sub-workflow reference:** None.

#### Extract Question & User
- **Type and role:** `n8n-nodes-base.code`; normalizes input payload.
- **Configuration choices:** JavaScript extracts:
  - `question = items[0].json.body.question`
  - `asked_by = items[0].json.body.asked_by || 'anonymous'`
  - `timestamp = new Date().toISOString()`
- **Key expressions or variables:** Direct body access in code.
- **Input and output connections:** Receives from **Question Webhook**; outputs to **Search Knowledge Base**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Throws if `body` is undefined
  - Empty question not validated here
- **Sub-workflow reference:** None.

#### Generate Query Embedding
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; provides query embedding generator to Pinecone search.
- **Configuration choices:** Default options only.
- **Key expressions or variables:** None directly.
- **Input and output connections:** Connected as `ai_embedding` input to **Search Knowledge Base**.
- **Version-specific requirements:** Type version 1.2.
- **Edge cases / failures:** Same OpenAI credential/rate-limit issues as ingestion embeddings.
- **Sub-workflow reference:** None.

#### Search Knowledge Base
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`; performs semantic similarity search.
- **Configuration choices:** Mode `load`; `topK = 5`; prompt text from current question; namespace `Test Team`; same Pinecone index.
- **Key expressions or variables:** `={{ $json.question }}`
- **Input and output connections:**
  - Main input from **Extract Question & User**
  - `ai_embedding` from **Generate Query Embedding**
  - Main output to **Build RAG Prompt**
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Empty query string
  - No matching vectors found
  - Pinecone latency or namespace/index mismatch
- **Sub-workflow reference:** None.

#### Build RAG Prompt
- **Type and role:** `n8n-nodes-base.code`; filters retrieved chunks, assembles context, and creates a strict answering instruction.
- **Configuration choices:**
  - Reads original question and user from **Extract Question & User**
  - Filters docs where `score > 0.3`
  - Builds source-labeled context blocks
  - Falls back to `"No relevant documents found in the knowledge base."`
  - Instructs model to answer only from context and return JSON:
    - `answer`
    - `sources`
    - `confidence`
    - `found_in_kb`
- **Key expressions or variables:**
  - `$('Extract Question & User').first().json.question`
  - `doc.json.score`
  - `doc.json.document?.pageContent`
  - `doc.json.document?.metadata?.doc_name`
- **Input and output connections:** Receives search results from **Search Knowledge Base**; outputs prompt payload to **Answer Generator**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - If Pinecone score semantics differ, threshold `0.3` may be too low or too high
  - Missing metadata may produce `Unknown` sources
  - If all docs are filtered out, answer relies on fallback context
- **Sub-workflow reference:** None.

#### Answer Generator
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`; generates final answer from assembled RAG prompt.
- **Configuration choices:** Model `gpt-4o`; content passed as a response input value from `{{$json.prompt}}`.
- **Key expressions or variables:** `={{ $json.prompt }}`
- **Input and output connections:** Receives from **Build RAG Prompt**; outputs to **Parse AI Answer**.
- **Version-specific requirements:** Type version 2.1; requires chat/completions-compatible OpenAI credentials in n8n’s AI node implementation.
- **Edge cases / failures:**
  - Model may still return non-JSON or fenced JSON
  - Token limits if retrieved context becomes too large
  - API quota/auth issues
- **Sub-workflow reference:** None.

#### Parse AI Answer
- **Type and role:** `n8n-nodes-base.code`; parses LLM output into normalized fields.
- **Configuration choices:**
  - Reads `items[0].json.output[0].content[0].text`
  - Removes markdown code fences
  - `JSON.parse(cleaned)`
  - Reattaches original `question` and `asked_by`
- **Key expressions or variables:**
  - `response.replace(/```json|```/g, '').trim()`
  - `JSON.parse(cleaned)`
  - `$('Extract Question & User').first().json.question`
- **Input and output connections:** Receives from **Answer Generator**; outputs to **Log Q&A to Sheets**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Most fragile node in this branch
  - Fails if model returns invalid JSON
  - Fails if output structure from OpenAI node changes between versions
- **Sub-workflow reference:** None.

#### Log Q&A to Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`; appends each answered question to a log sheet.
- **Configuration choices:** Append to **Q&A Log** sheet with fields:
  - `date`
  - `question`
  - `answer`
  - `sources`
  - `asked_by`
  - `confidence`
  - `found_in_kb`
- **Key expressions or variables:** Values mapped from current item fields such as `{{$json.timestamp}}`, `{{$json.answer}}`, etc.
- **Input and output connections:** Receives from **Parse AI Answer**; outputs to **Send Answer Response**.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases / failures:**
  - Sheet access/schema issues
  - Boolean `found_in_kb` may be stored as string depending on sheet behavior
- **Sub-workflow reference:** None.

#### Send Answer Response
- **Type and role:** `n8n-nodes-base.respondToWebhook`; returns the final API response.
- **Configuration choices:** Responds with JSON body containing question, answer, sources, confidence, and asked_by; `executeOnce: true`.
- **Key expressions or variables:** Uses inline response template expressions like `{{ $json.answer }}`.
- **Input and output connections:** Receives from **Log Q&A to Sheets**; terminates webhook flow.
- **Version-specific requirements:** Type version 1.5.
- **Edge cases / failures:**
  - If upstream node errors, webhook caller receives workflow failure instead of structured response
  - `sources` is returned as a string, not an array
- **Sub-workflow reference:** None.

---

## 2.3 Weekly Sync and Re-ingestion

### Overview
This block runs on a weekly schedule, compares current Google Drive documents to a registry sheet, identifies documents that are new or updated since their last ingestion date, re-ingests those files into Pinecone, updates the registry, and emails a digest. If no updates are found, it sends a digest from a separate no-op branch.

### Nodes Involved
- Weekly Sunday 11AM Trigger
- List Knowledge Base Docs1
- Read Document Registry
- Detect New or Updated Docs
- IF New Docs Found
- Prepare Docs for Re-ingestion
- Download Updated Docs
- Generate Embeddings1
- Split Into Chunks1
- Load Document Data1
- Re-ingest into Pinecone
- Update Document Registry
- Build KB Digest Email
- Send Weekly KB Digest
- Build KB Digest Email1
- Send Weekly KB Digest1

### Node Details

#### Weekly Sunday 11AM Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; starts the maintenance run weekly.
- **Configuration choices:** Interval-based weekly trigger, set to fire at hour 11.
- **Key expressions or variables:** None.
- **Input and output connections:** Outputs to **List Knowledge Base Docs1**.
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Timezone depends on workflow/server settings
  - The node name says Sunday 11AM, but exact weekday interpretation should be verified in n8n UI because interval rules can be subtle
- **Sub-workflow reference:** None.

#### List Knowledge Base Docs1
- **Type and role:** `n8n-nodes-base.googleDrive`; lists current documents in the same Knowledge Base folder.
- **Configuration choices:** Same folder-based listing as initial ingestion.
- **Key expressions or variables:** Fixed folder ID.
- **Input and output connections:** Receives from **Weekly Sunday 11AM Trigger**; outputs to **Read Document Registry**.
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:** Same as the first Drive listing node.
- **Sub-workflow reference:** None.

#### Read Document Registry
- **Type and role:** `n8n-nodes-base.googleSheets`; reads the registry sheet for comparison.
- **Configuration choices:** Reads sheet **Document Registry** from the spreadsheet.
- **Key expressions or variables:** None.
- **Input and output connections:** Receives after **List Knowledge Base Docs1**; outputs to **Detect New or Updated Docs**.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases / failures:**
  - Empty registry on first scheduled run
  - Duplicate rows per `doc_id` may affect comparison because later rows overwrite earlier ones only if they appear later in the returned dataset
- **Sub-workflow reference:** None.

#### Detect New or Updated Docs
- **Type and role:** `n8n-nodes-base.code`; compares Drive docs with registry entries.
- **Configuration choices:**
  - Uses `$('List Knowledge Base Docs1').all()` and `$('Read Document Registry').all()`
  - Builds `registryMap` keyed by `doc_id`
  - Marks docs as:
    - `new` if absent from registry
    - `updated` if Drive `modifiedTime` is newer than `last_ingested`
    - `current` otherwise
  - Returns summary counts and a `needs_ingestion` boolean
- **Key expressions or variables:**
  - `doc.json.modifiedTime`
  - `new Date(inRegistry.last_ingested)`
  - `new Date(modifiedTime)`
- **Input and output connections:** Receives from **Read Document Registry**; outputs to **IF New Docs Found**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Uses `inRegistry.last_ingested` instead of `inRegistry.json.last_ingested`; this is likely a logic bug and may produce `Invalid Date`
  - Date-only comparison against full Drive timestamp may be coarse
  - Duplicate registry rows may produce inconsistent status based on row order
- **Sub-workflow reference:** None.

#### IF New Docs Found
- **Type and role:** `n8n-nodes-base.if`; branches based on whether any doc requires ingestion.
- **Configuration choices:** Condition checks `{{$json.needs_ingestion}}` is true.
- **Key expressions or variables:** `={{ $json.needs_ingestion }}`
- **Input and output connections:**
  - True branch to **Prepare Docs for Re-ingestion**
  - False branch to **Build KB Digest Email1**
- **Version-specific requirements:** Type version 2.3.
- **Edge cases / failures:**
  - If `needs_ingestion` is missing or not boolean, strict validation may fail
- **Sub-workflow reference:** None.

#### Prepare Docs for Re-ingestion
- **Type and role:** `n8n-nodes-base.code`; expands the list of changed docs into one item per document.
- **Configuration choices:** Maps `new_or_updated` array into items containing `id`, `name`, `status`.
- **Key expressions or variables:** `items[0].json.new_or_updated`
- **Input and output connections:** Receives from **IF New Docs Found** true branch; outputs to **Download Updated Docs**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - Empty array would produce no items
  - Assumes `new_or_updated` exists
- **Sub-workflow reference:** None.

#### Download Updated Docs
- **Type and role:** `n8n-nodes-base.googleDrive`; downloads changed documents as plain text.
- **Configuration choices:** Same conversion pattern as initial ingestion; `fileId` from `{{$json.id}}`.
- **Key expressions or variables:** `={{ $json.id }}`
- **Input and output connections:** Receives from **Prepare Docs for Re-ingestion**; outputs to **Re-ingest into Pinecone**.
- **Version-specific requirements:** Type version 3.
- **Edge cases / failures:** Same as initial download node.
- **Sub-workflow reference:** None.

#### Generate Embeddings1
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`; embedding generator for re-ingestion.
- **Configuration choices:** Default options.
- **Key expressions or variables:** None.
- **Input and output connections:** Supplies `ai_embedding` to **Re-ingest into Pinecone**.
- **Version-specific requirements:** Type version 1.2.
- **Edge cases / failures:** Same as embedding node in initial ingestion.
- **Sub-workflow reference:** None.

#### Split Into Chunks1
- **Type and role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`; chunking for updated docs.
- **Configuration choices:** `chunkSize: 500`, `chunkOverlap: 100`.
- **Key expressions or variables:** None.
- **Input and output connections:** Sends `ai_textSplitter` to **Load Document Data1**.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** Same as initial splitter.
- **Sub-workflow reference:** None.

#### Load Document Data1
- **Type and role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`; prepares downloaded content and metadata for Pinecone insertion.
- **Configuration choices:** `dataType: binary`; custom text splitting; metadata `doc_name` from **Prepare Docs for Re-ingestion**.
- **Key expressions or variables:** `={{ $('Prepare Docs for Re-ingestion').item.json.name }}`
- **Input and output connections:** Receives `ai_textSplitter` from **Split Into Chunks1**; sends `ai_document` to **Re-ingest into Pinecone**.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases / failures:** Same as first loader node.
- **Sub-workflow reference:** None.

#### Re-ingest into Pinecone
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`; inserts vectors for new or updated docs.
- **Configuration choices:** Mode `insert`; namespace `Test Team`; index `meeting-sentiment-index`.
- **Key expressions or variables:** None directly.
- **Input and output connections:**
  - Main input from **Download Updated Docs**
  - `ai_embedding` from **Generate Embeddings1**
  - `ai_document` from **Load Document Data1**
  - Main output to **Update Document Registry**
- **Version-specific requirements:** Type version 1.3.
- **Edge cases / failures:**
  - Same as initial Pinecone insert
  - Updated docs are inserted again without deleting old vectors, likely creating duplicate versions
- **Sub-workflow reference:** None.

#### Update Document Registry
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a new registry row after re-ingestion.
- **Configuration choices:** Writes:
  - `doc_id` from **Prepare Docs for Re-ingestion**
  - `status` from the doc’s detected status (`new` or `updated`)
  - `doc_name`
  - `last_ingested` current date
- **Key expressions or variables:**
  - `={{ $('Prepare Docs for Re-ingestion').item.json.id }}`
  - `={{ $('Prepare Docs for Re-ingestion').item.json.status }}`
  - `={{ $('Prepare Docs for Re-ingestion').item.json.name }}`
- **Input and output connections:** Receives from **Re-ingest into Pinecone**; outputs to **Build KB Digest Email**.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases / failures:**
  - Append-only history rather than update-in-place
  - Duplicate records may complicate later comparisons
- **Sub-workflow reference:** None.

#### Build KB Digest Email
- **Type and role:** `n8n-nodes-base.code`; builds weekly digest after a run with changes.
- **Configuration choices:** Uses summary from **Detect New or Updated Docs** to compose plain text report with totals, changed docs, current docs, and webhook reminder.
- **Key expressions or variables:** `$('Detect New or Updated Docs').first().json`
- **Input and output connections:** Receives from **Update Document Registry**; outputs to **Send Weekly KB Digest**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:**
  - If multiple docs are re-ingested, this node may run multiple times because it sits after append-per-doc processing
  - No `executeOnce` protection on this node or its downstream email node
- **Sub-workflow reference:** None.

#### Send Weekly KB Digest
- **Type and role:** `n8n-nodes-base.gmail`; sends digest email after updates were detected.
- **Configuration choices:** Subject uses total docs and updated count; body from `{{$json.email_body}}`.
- **Key expressions or variables:** Subject `=🧠 Weekly KB Digest — {{ $json.total_docs }} Docs | {{ $json.new_count }} Updated`
- **Input and output connections:** Receives from **Build KB Digest Email**; no downstream nodes.
- **Version-specific requirements:** Type version 2.2.
- **Edge cases / failures:**
  - Placeholder recipient must be replaced
  - If multiple upstream items arrive, multiple emails may be sent
- **Sub-workflow reference:** None.

#### Build KB Digest Email1
- **Type and role:** `n8n-nodes-base.code`; alternate digest builder for the branch where no changes are detected.
- **Configuration choices:** Logic is effectively identical to **Build KB Digest Email**.
- **Key expressions or variables:** `$('Detect New or Updated Docs').first().json`
- **Input and output connections:** Receives from **IF New Docs Found** false branch; outputs to **Send Weekly KB Digest1**.
- **Version-specific requirements:** Type version 2.
- **Edge cases / failures:** Duplicate logic means maintenance burden if message format changes.
- **Sub-workflow reference:** None.

#### Send Weekly KB Digest1
- **Type and role:** `n8n-nodes-base.gmail`; sends digest email when no re-ingestion is needed.
- **Configuration choices:** Same subject/body pattern as the other digest email node.
- **Key expressions or variables:** Same as **Send Weekly KB Digest**.
- **Input and output connections:** Receives from **Build KB Digest Email1**; no downstream nodes.
- **Version-specific requirements:** Type version 2.2.
- **Edge cases / failures:** Same Gmail and recipient concerns.
- **Sub-workflow reference:** None.

---

## 2.4 In-Canvas Documentation and Notes

### Overview
These nodes are not executed. They provide human-readable context about purpose, setup, and runtime behavior directly inside the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level setup and architecture note.
- **Configuration choices:** Contains workflow overview, setup steps, and Pinecone dimension requirement.
- **Key expressions or variables:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** Not executed.
- **Sub-workflow reference:** Mentions “SW1”, “SW2”, “SW3” conceptually, but no actual sub-workflow nodes exist.

#### Sticky Note1
- **Type and role:** sticky note describing manual ingestion branch.
- **Configuration choices:** Explains the branch reads Google Docs, chunks, embeds, stores in Pinecone, logs to registry.
- **Input and output connections:** None.
- **Edge cases / failures:** Not executed.

#### Sticky Note2
- **Type and role:** sticky note describing webhook question-answering branch.
- **Configuration choices:** Explains POST behavior, top 5 search, GPT-4o, source citation, confidence score, and logging.
- **Input and output connections:** None.
- **Edge cases / failures:** Not executed.

#### Sticky Note3
- **Type and role:** sticky note describing weekly maintenance branch.
- **Configuration choices:** Explains weekly comparison, re-ingestion, and email digest.
- **Input and output connections:** None.
- **Edge cases / failures:** Not executed.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Log to Document Registry | Google Sheets | Append ingestion record to registry sheet | Store Embeddings in Pinecone | Notify Ingestion Complete | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Notify Ingestion Complete | Gmail | Send completion notification after manual ingestion | Log to Document Registry |  | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Read Document Registry | Google Sheets | Read registry for weekly comparison | List Knowledge Base Docs1 | Detect New or Updated Docs | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Update Document Registry | Google Sheets | Append re-ingestion record | Re-ingest into Pinecone | Build KB Digest Email | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Manual Ingestion Trigger | Manual Trigger | Manual entry point for initial/full ingestion |  | List Knowledge Base Docs | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| List Knowledge Base Docs | Google Drive | List all files in knowledge base folder for initial ingestion | Manual Ingestion Trigger | Download Doc as Text | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Download Doc as Text | Google Drive | Download each Google Doc as plain text | List Knowledge Base Docs | Store Embeddings in Pinecone | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Store Embeddings in Pinecone | Pinecone Vector Store | Insert document chunks and embeddings into Pinecone | Download Doc as Text, Generate Embeddings, Load Document Data | Log to Document Registry | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Load Document Data | Document Default Data Loader | Load binary text and attach metadata for ingestion | Split Into Chunks | Store Embeddings in Pinecone | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Split Into Chunks | Recursive Character Text Splitter | Split documents into chunks for embedding |  | Load Document Data | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Generate Embeddings | OpenAI Embeddings | Generate embeddings for ingestion |  | Store Embeddings in Pinecone | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Question Webhook | Webhook | Receive incoming question requests |  | Extract Question & User | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Extract Question & User | Code | Normalize webhook body into question payload | Question Webhook | Search Knowledge Base | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Search Knowledge Base | Pinecone Vector Store | Retrieve top semantic matches for the question | Extract Question & User, Generate Query Embedding | Build RAG Prompt | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Generate Query Embedding | OpenAI Embeddings | Generate embedding for query search |  | Search Knowledge Base | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Build RAG Prompt | Code | Build strict grounded prompt from retrieved chunks | Search Knowledge Base | Answer Generator | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Answer Generator | OpenAI Chat Model | Generate grounded answer in JSON format | Build RAG Prompt | Parse AI Answer | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Parse AI Answer | Code | Parse model JSON output into normalized fields | Answer Generator | Log Q&A to Sheets | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Log Q&A to Sheets | Google Sheets | Append question/answer event to log sheet | Parse AI Answer | Send Answer Response | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Send Answer Response | Respond to Webhook | Return final JSON API response | Log Q&A to Sheets |  | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Weekly Sunday 11AM Trigger | Schedule Trigger | Start weekly sync run |  | List Knowledge Base Docs1 | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| List Knowledge Base Docs1 | Google Drive | List current files in knowledge base folder for weekly sync | Weekly Sunday 11AM Trigger | Read Document Registry | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Detect New or Updated Docs | Code | Compare Drive files to registry and summarize changes | Read Document Registry | IF New Docs Found | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| IF New Docs Found | If | Branch based on whether ingestion is needed | Detect New or Updated Docs | Prepare Docs for Re-ingestion, Build KB Digest Email1 | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Prepare Docs for Re-ingestion | Code | Expand changed docs into one item per file | IF New Docs Found | Download Updated Docs | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Download Updated Docs | Google Drive | Download changed docs as plain text | Prepare Docs for Re-ingestion | Re-ingest into Pinecone | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Re-ingest into Pinecone | Pinecone Vector Store | Insert vectors for new or updated docs | Download Updated Docs, Generate Embeddings1, Load Document Data1 | Update Document Registry | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Load Document Data1 | Document Default Data Loader | Load updated file content and metadata | Split Into Chunks1 | Re-ingest into Pinecone | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Split Into Chunks1 | Recursive Character Text Splitter | Split updated docs into chunks |  | Load Document Data1 | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Generate Embeddings1 | OpenAI Embeddings | Generate embeddings for re-ingestion |  | Re-ingest into Pinecone | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Build KB Digest Email | Code | Build weekly digest after changes are ingested | Update Document Registry | Send Weekly KB Digest | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Build KB Digest Email1 | Code | Build weekly digest when no ingestion is required | IF New Docs Found | Send Weekly KB Digest1 | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Send Weekly KB Digest | Gmail | Send weekly digest after updates branch | Build KB Digest Email |  | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Send Weekly KB Digest1 | Gmail | Send weekly digest after no-change branch | Build KB Digest Email1 |  | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |
| Sticky Note | Sticky Note | Canvas documentation for overall workflow and setup |  |  | ## Workflow Overview  Stop answering the same company questions over and over. This workflow turns your Google Drive documents into a searchable AI knowledge base. Ask it anything about your company — HR policies, financials, sales process, product details — and it answers instantly with the exact source it pulled from. |
| Sticky Note1 | Sticky Note | Canvas documentation for manual ingestion branch |  |  | Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. |
| Sticky Note2 | Sticky Note | Canvas documentation for webhook Q&A branch |  |  | Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. |
| Sticky Note3 | Sticky Note | Canvas documentation for weekly sync branch |  |  | Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the required external resources first**
   1. Create a Google Drive folder named something like **Knowledge Base**.
   2. Add your internal company documents as Google Docs.
   3. Create a Google Sheets spreadsheet for workflow tracking with at least two sheets:
      - **Document Registry** with columns:
        - `doc_id`
        - `doc_name`
        - `last_ingested`
        - `status`
      - **Q&A Log** with columns:
        - `date`
        - `question`
        - `answer`
        - `sources`
        - `asked_by`
        - `confidence`
        - `found_in_kb`
   4. Create a Pinecone index.
      - Ensure the embedding dimension matches the OpenAI embedding model used by the n8n embeddings node.
      - The sticky note explicitly recommends **1536 dimensions** and **cosine** metric.
   5. Prepare credentials in n8n:
      - Google Drive OAuth2
      - Google Sheets OAuth2
      - Gmail OAuth2
      - OpenAI API
      - Pinecone API

2. **Create the manual ingestion entry point**
   1. Add a **Manual Trigger** node.
   2. Name it **Manual Ingestion Trigger**.

3. **Add the Google Drive listing node for initial ingestion**
   1. Add a **Google Drive** node.
   2. Name it **List Knowledge Base Docs**.
   3. Set **Resource** to `File/Folder`.
   4. Configure the folder filter to your Knowledge Base folder ID.
   5. Connect **Manual Ingestion Trigger -> List Knowledge Base Docs**.

4. **Add the document download node**
   1. Add another **Google Drive** node.
   2. Name it **Download Doc as Text**.
   3. Set **Operation** to `Download`.
   4. Set **File ID** to `{{$json.id}}`.
   5. Enable Google file conversion and set Google Docs conversion to **text/plain**.
   6. Connect **List Knowledge Base Docs -> Download Doc as Text**.

5. **Add the AI ingestion support nodes**
   1. Add an **OpenAI Embeddings** node.
   2. Name it **Generate Embeddings**.
   3. Attach your OpenAI credential.
   4. Add a **Recursive Character Text Splitter** node.
   5. Name it **Split Into Chunks**.
   6. Set:
      - `chunkSize = 500`
      - `chunkOverlap = 100`
   7. Add a **Document Default Data Loader** node.
   8. Name it **Load Document Data**.
   9. Set:
      - `dataType = binary`
      - `textSplittingMode = custom`
   10. Add metadata:
       - name: `doc_name`
       - value: `{{ $('Download Doc as Text').item.json.name }}`
   11. Connect **Split Into Chunks -> Load Document Data** using the `ai_textSplitter` connection.

6. **Add the Pinecone insert node for initial ingestion**
   1. Add a **Pinecone Vector Store** node.
   2. Name it **Store Embeddings in Pinecone**.
   3. Set **Mode** to `Insert`.
   4. Choose your Pinecone index.
   5. Set namespace to `Test Team` or your preferred namespace.
   6. Connect:
      - **Download Doc as Text -> Store Embeddings in Pinecone** via main connection
      - **Generate Embeddings -> Store Embeddings in Pinecone** via `ai_embedding`
      - **Load Document Data -> Store Embeddings in Pinecone** via `ai_document`

7. **Add registry logging for initial ingestion**
   1. Add a **Google Sheets** node.
   2. Name it **Log to Document Registry**.
   3. Set operation to **Append**.
   4. Select your spreadsheet and the **Document Registry** sheet.
   5. Map fields:
      - `doc_id = {{ $('Download Doc as Text').item.json.id }}`
      - `doc_name = {{ $json.metadata.doc_name }}`
      - `last_ingested = {{ new Date().toISOString().split('T')[0] }}`
      - `status = ingested`
   6. Connect **Store Embeddings in Pinecone -> Log to Document Registry**.

8. **Add completion email for manual ingestion**
   1. Add a **Gmail** node.
   2. Name it **Notify Ingestion Complete**.
   3. Configure Gmail OAuth2 credentials.
   4. Set recipient to your actual email address.
   5. Set subject to `✅ Knowledge Base Updated`.
   6. Set body to plain text such as:
      - “Knowledge Base Ingestion Complete”
      - “Documents successfully stored in Pinecone.”
      - current date
      - Pinecone verification reminder
   7. Enable **Execute Once**.
   8. Connect **Log to Document Registry -> Notify Ingestion Complete**.

9. **Create the webhook entry point**
   1. Add a **Webhook** node.
   2. Name it **Question Webhook**.
   3. Set method to `POST`.
   4. Set path to `knowledge-brain`.
   5. Set response mode to **Using Respond to Webhook Node**.
   6. This endpoint should accept JSON like:
      ```json
      {
        "question": "What is our PTO policy?",
        "asked_by": "alice@company.com"
      }
      ```

10. **Normalize incoming webhook payload**
    1. Add a **Code** node.
    2. Name it **Extract Question & User**.
    3. Use code that returns one item with:
       - `question` from `items[0].json.body.question`
       - `asked_by` from `items[0].json.body.asked_by || 'anonymous'`
       - `timestamp` as current ISO timestamp
    4. Connect **Question Webhook -> Extract Question & User**.

11. **Add query embedding and Pinecone search**
    1. Add an **OpenAI Embeddings** node.
    2. Name it **Generate Query Embedding**.
    3. Add a **Pinecone Vector Store** node.
    4. Name it **Search Knowledge Base**.
    5. Set mode to `Load`.
    6. Set `topK = 5`.
    7. Set prompt/query to `{{ $json.question }}`.
    8. Use the same Pinecone index and namespace as ingestion.
    9. Connect:
       - **Extract Question & User -> Search Knowledge Base**
       - **Generate Query Embedding -> Search Knowledge Base** via `ai_embedding`

12. **Build the RAG prompt**
    1. Add a **Code** node.
    2. Name it **Build RAG Prompt**.
    3. Implement logic to:
       - read original question and user
       - inspect Pinecone search results
       - keep only results with score above `0.3`
       - assemble context with source headers
       - fallback to “No relevant documents found in the knowledge base.”
       - instruct the model to answer only from context
       - require JSON output with `answer`, `sources`, `confidence`, `found_in_kb`
    4. Connect **Search Knowledge Base -> Build RAG Prompt**.

13. **Add the answer generation node**
    1. Add an **OpenAI Chat/LLM** node compatible with the LangChain package.
    2. Name it **Answer Generator**.
    3. Select model **gpt-4o**.
    4. Pass the generated prompt as message content: `{{ $json.prompt }}`.
    5. Connect **Build RAG Prompt -> Answer Generator**.

14. **Parse the AI response**
    1. Add a **Code** node.
    2. Name it **Parse AI Answer**.
    3. Parse the LLM output text from the node response structure.
    4. Strip markdown fences like ```json.
    5. Run `JSON.parse(...)`.
    6. Return a normalized object containing:
       - `question`
       - `asked_by`
       - `answer`
       - `sources` joined into a comma-separated string
       - `confidence`
       - `found_in_kb`
       - `timestamp`
    7. Connect **Answer Generator -> Parse AI Answer**.

15. **Log webhook Q&A**
    1. Add a **Google Sheets** node.
    2. Name it **Log Q&A to Sheets**.
    3. Set operation to **Append**.
    4. Select your spreadsheet and the **Q&A Log** sheet.
    5. Map:
       - `date = {{ $json.timestamp }}`
       - `question = {{ $json.question }}`
       - `answer = {{ $json.answer }}`
       - `sources = {{ $json.sources }}`
       - `asked_by = {{ $json.asked_by }}`
       - `confidence = {{ $json.confidence }}`
       - `found_in_kb = {{ $json.found_in_kb }}`
    6. Connect **Parse AI Answer -> Log Q&A to Sheets**.

16. **Return the webhook response**
    1. Add a **Respond to Webhook** node.
    2. Name it **Send Answer Response**.
    3. Set response type to **JSON**.
    4. Return fields:
       - question
       - answer
       - sources
       - confidence
       - asked_by
    5. Connect **Log Q&A to Sheets -> Send Answer Response**.
    6. Enable **Execute Once**.

17. **Create the weekly maintenance trigger**
    1. Add a **Schedule Trigger** node.
    2. Name it **Weekly Sunday 11AM Trigger**.
    3. Configure a weekly schedule at 11:00.
    4. Confirm server timezone so the actual fire time matches expectations.

18. **Add weekly Drive listing**
    1. Add a **Google Drive** node.
    2. Name it **List Knowledge Base Docs1**.
    3. Configure it exactly like the initial listing node, using the same folder ID.
    4. Connect **Weekly Sunday 11AM Trigger -> List Knowledge Base Docs1**.

19. **Add registry read for comparison**
    1. Add a **Google Sheets** node.
    2. Name it **Read Document Registry**.
    3. Point it to the **Document Registry** sheet.
    4. Connect **List Knowledge Base Docs1 -> Read Document Registry**.

20. **Add comparison logic**
    1. Add a **Code** node.
    2. Name it **Detect New or Updated Docs**.
    3. Implement logic to:
       - load all Drive docs
       - load all registry rows
       - build a `registryMap` keyed by `doc_id`
       - compare Drive `modifiedTime` against `last_ingested`
       - return:
         - `new_or_updated`
         - `existing`
         - `all_docs`
         - `total_docs`
         - `new_count`
         - `existing_count`
         - `needs_ingestion`
    4. Important: when reading registry rows, use `r.json.doc_id` and `r.json.last_ingested` consistently to avoid invalid date bugs.
    5. Connect **Read Document Registry -> Detect New or Updated Docs**.

21. **Add the IF branch**
    1. Add an **If** node.
    2. Name it **IF New Docs Found**.
    3. Configure condition: `{{ $json.needs_ingestion }}` is true.
    4. Connect **Detect New or Updated Docs -> IF New Docs Found**.

22. **Build the true branch for changed docs**
    1. Add a **Code** node named **Prepare Docs for Re-ingestion**.
    2. Convert `new_or_updated` into one item per document with:
       - `id`
       - `name`
       - `status`
    3. Connect the true output of **IF New Docs Found -> Prepare Docs for Re-ingestion**.

23. **Download changed docs**
    1. Add a **Google Drive** node named **Download Updated Docs**.
    2. Set operation to `Download`.
    3. Set file ID to `{{ $json.id }}`.
    4. Enable Google Docs conversion to `text/plain`.
    5. Connect **Prepare Docs for Re-ingestion -> Download Updated Docs**.

24. **Add re-ingestion AI helper nodes**
    1. Add **OpenAI Embeddings** node named **Generate Embeddings1**.
    2. Add **Recursive Character Text Splitter** node named **Split Into Chunks1** with:
       - `chunkSize = 500`
       - `chunkOverlap = 100`
    3. Add **Document Default Data Loader** node named **Load Document Data1** with:
       - `dataType = binary`
       - `textSplittingMode = custom`
       - metadata `doc_name = {{ $('Prepare Docs for Re-ingestion').item.json.name }}`
    4. Connect **Split Into Chunks1 -> Load Document Data1** via `ai_textSplitter`.

25. **Add Pinecone re-ingestion node**
    1. Add a **Pinecone Vector Store** node named **Re-ingest into Pinecone**.
    2. Set mode to `Insert`.
    3. Use the same index and namespace as before.
    4. Connect:
       - **Download Updated Docs -> Re-ingest into Pinecone**
       - **Generate Embeddings1 -> Re-ingest into Pinecone** via `ai_embedding`
       - **Load Document Data1 -> Re-ingest into Pinecone** via `ai_document`

26. **Update the registry after weekly re-ingestion**
    1. Add a **Google Sheets** node named **Update Document Registry**.
    2. Set operation to **Append**.
    3. Map:
       - `doc_id = {{ $('Prepare Docs for Re-ingestion').item.json.id }}`
       - `doc_name = {{ $('Prepare Docs for Re-ingestion').item.json.name }}`
       - `status = {{ $('Prepare Docs for Re-ingestion').item.json.status }}`
       - `last_ingested = {{ new Date().toISOString().split('T')[0] }}`
    4. Connect **Re-ingest into Pinecone -> Update Document Registry**.

27. **Build the digest email for changed-doc runs**
    1. Add a **Code** node named **Build KB Digest Email**.
    2. Use `$('Detect New or Updated Docs').first().json` to build a plain-text summary including:
       - total docs
       - new/updated count
       - current count
       - changed document names
       - webhook reminder
    3. Connect **Update Document Registry -> Build KB Digest Email**.

28. **Send the digest for changed-doc runs**
    1. Add a **Gmail** node named **Send Weekly KB Digest**.
    2. Set recipient to a real email address.
    3. Subject example:
       - `🧠 Weekly KB Digest — {{ $json.total_docs }} Docs | {{ $json.new_count }} Updated`
    4. Message body:
       - `{{ $json.email_body }}`
    5. Connect **Build KB Digest Email -> Send Weekly KB Digest**.

29. **Build the no-change branch**
    1. Add another **Code** node named **Build KB Digest Email1**.
    2. Use the same email-building logic as the previous digest node.
    3. Connect the false output of **IF New Docs Found -> Build KB Digest Email1**.

30. **Send the digest for no-change runs**
    1. Add another **Gmail** node named **Send Weekly KB Digest1**.
    2. Use the same subject/body as the previous email node.
    3. Connect **Build KB Digest Email1 -> Send Weekly KB Digest1**.

31. **Add optional sticky notes**
    1. Add one overall note describing:
       - purpose
       - setup steps
       - Pinecone dimensions and metric
       - webhook usage
    2. Add one note above each major branch:
       - manual ingestion
       - webhook answering
       - weekly maintenance

32. **Activate and test**
    1. Run **Manual Ingestion Trigger** once to populate Pinecone and the registry.
    2. Test the webhook with a POST request.
    3. Verify:
       - vectors exist in Pinecone
       - document rows appear in **Document Registry**
       - Q&A entries appear in **Q&A Log**
       - emails are delivered
    4. Activate the workflow if you want webhook and schedule behavior in production.

### Reproduction caveats
- The current design uses **append-only registry rows**, not updates. This means the sheet becomes historical, not canonical.
- The current Pinecone design uses **insert**, not delete/upsert. Re-ingesting updated docs can create duplicate vector records unless the node internally manages IDs.
- The OpenAI parsing logic assumes a specific response structure and valid JSON output. In production, consider adding fallback parsing or structured output enforcement.
- Replace placeholder `your_email_id` everywhere.
- Keep index, namespace, and embedding model compatible.

### Sub-workflow setup
This workflow contains **no Execute Workflow or sub-workflow nodes**. All logic is contained in a single workflow, despite sticky note references to “SW1”, “SW2”, and “SW3”.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Stop answering the same company questions over and over. This workflow turns your Google Drive documents into a searchable AI knowledge base. Ask it anything about your company — HR policies, financials, sales process, product details — and it answers instantly with the exact source it pulled from. | Overall workflow note |
| HOW IT WORKS: SW1 ingests your Google Docs into Pinecone as vector embeddings. SW2 takes any question via webhook, finds the most relevant chunks using semantic search, and sends them to GPT-4o for a grounded answer with source citation. SW3 runs every Sunday to check for new or updated documents and keeps the knowledge base fresh. | Overall workflow note |
| SETUP STEPS: 1. Create a Google Drive folder called Knowledge Base 2. Add your company documents as Google Docs 3. Create a Pinecone index — dimensions 1536, metric cosine 4. Add Pinecone, OpenAI, Google Drive, Sheets and Gmail credentials 5. Send POST requests to the webhook to ask questions | Overall workflow note |
| Run this manually when adding new documents. Reads all Google Docs from your Knowledge Base folder, splits them into chunks, converts to embeddings, and stores everything in Pinecone. Logs each doc to the Document Registry sheet. | Manual ingestion branch note |
| Always active via webhook. Takes any question as a POST request, searches Pinecone for the 5 most relevant chunks, sends context to GPT-4o, and returns a grounded answer with source citation and confidence score. All Q&A logged to Sheets. | Webhook branch note |
| Runs every Sunday at 11AM. Compares your Drive folder against the Document Registry, detects new or updated docs, re-ingests them into Pinecone automatically, and emails a weekly digest of your full knowledge base status. | Weekly maintenance branch note |