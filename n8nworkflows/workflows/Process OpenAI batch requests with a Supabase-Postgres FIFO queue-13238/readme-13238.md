Process OpenAI batch requests with a Supabase/Postgres FIFO queue

https://n8nworkflows.xyz/workflows/process-openai-batch-requests-with-a-supabase-postgres-fifo-queue-13238


# Process OpenAI batch requests with a Supabase/Postgres FIFO queue

## 1. Workflow Overview

**Purpose:**  
Process multiple OpenAI requests cheaply via the **OpenAI Batch API**, while tracking jobs in a persistent **Supabase/Postgres FIFO queue**. The workflow has two phases:

### 1.1 Phase 1 — Submit Batch (manual / upstream-triggered)
Takes a `systemPrompt` plus an array of `inputs`, converts them into OpenAI Batch `.jsonl` format, uploads the file to OpenAI **Files API**, creates a batch job via **Batches API**, then stores the batch `id` and initial status in Supabase (`openai_batches`).

### 1.2 Phase 2 — Poll & Retrieve (scheduled)
Every 5 minutes, pulls the earliest batch whose status is *not* terminal, retrieves the batch status from OpenAI, and:
- If **not completed**: updates status in Supabase.
- If **completed** (and has `output_file_id`): downloads output `.jsonl`, parses it, extracts responses, and updates Supabase with status + results.

---

## 2. Block-by-Block Analysis

### Block 1 — Submit Batch Input Preparation & Upload
**Overview:** Converts an array of prompts into the OpenAI Batch `.jsonl` format, then uploads that `.jsonl` to OpenAI Files API for batch processing.

**Nodes involved:**
- `Start (mock data)`
- `Convert to batch requests in .jsonl`
- `Call Files API`

#### Node: Start (mock data)
- **Type / role:** Manual Trigger (`manualTrigger`) to simulate upstream input.
- **Configuration choices:** Uses pinned data with:
  - `systemPrompt` (string)
  - `inputs` (array of strings)
- **Inputs/Outputs:** Entry point → outputs one item containing the mock JSON.
- **Edge cases:**
  - If `inputs` is missing or not an array, downstream Code node fails.
  - If prompts contain unexpected types (objects), `.jsonl` line creation may still serialize but the OpenAI API may reject content.

#### Node: Convert to batch requests in .jsonl
- **Type / role:** Code node (`code`) that generates OpenAI Batch `.jsonl` and returns it as **binary** (base64).
- **Key logic/config:**
  - Reads:
    - `const inputs = $input.first().json.inputs`
    - `const systemPrompt = $input.first().json.systemPrompt`
  - Builds one JSON line per input with:
    - `custom_id: `${index}_${Date.now()}``
    - `method: "POST"`
    - `url: "/v1/responses"`
    - `body: { model: "gpt-5-mini", instructions: systemPrompt, input }`
  - Joins lines with `\n` to create NDJSON.
  - Base64-encodes content and returns as:
    - `binary.data.data` (base64)
    - `mimeType: application/x-ndjson`
    - `fileName: batch_input.jsonl`
- **Connections:** Input from `Start (mock data)` → output to `Call Files API`.
- **Version notes:** Code node v2; uses Node.js `Buffer` (available in n8n Code node).
- **Edge cases / failure modes:**
  - Large `inputs` can create oversized `.jsonl` → OpenAI file upload limits or timeouts.
  - `Date.now()` inside map can produce identical timestamps within same ms; uniqueness relies also on `index`.
  - Uses `/v1/responses` format; if your OpenAI account/model doesn’t support Responses API in batch, batch creation will fail.

#### Node: Call Files API
- **Type / role:** HTTP Request node uploading the `.jsonl` to OpenAI Files.
- **Configuration choices:**
  - `POST https://api.openai.com/v1/files`
  - `multipart-form-data`
  - Form fields:
    - `purpose = batch`
    - `file` = binary field `data`
  - Auth: `predefinedCredentialType` → `openAiApi` credential
  - `alwaysOutputData: true` (so it still outputs something on some error conditions)
- **Inputs/Outputs:**
  - Expects binary `data` from previous node.
  - Outputs OpenAI file object (notably `id`) used by the next step.
- **Edge cases / failure modes:**
  - Missing/invalid OpenAI credential → 401/403.
  - Incorrect binary field name (must be `data`) → empty upload or 400.
  - Network timeout / file too large → request fails.

---

### Block 2 — Create Batch Job & Persist in Supabase
**Overview:** Creates the OpenAI batch using the uploaded file id, then inserts a tracking row in Supabase.

**Nodes involved:**
- `Call Batch API`
- `Create a row in batch table`
- `Submission done`

#### Node: Call Batch API
- **Type / role:** HTTP Request node to create a batch job.
- **Configuration choices:**
  - `POST https://api.openai.com/v1/batches`
  - JSON body:
    - `input_file_id: {{ $json.id }}` (from Files API response)
    - `endpoint: "/v1/responses"`
    - `completion_window: "24h"`
  - Auth: OpenAI credential
  - `alwaysOutputData: true`
- **Connections:** From `Call Files API` → to `Create a row in batch table`.
- **Edge cases / failure modes:**
  - If `$json.id` is missing (failed upload), body becomes invalid.
  - Endpoint mismatch: file lines target `/v1/responses` and batch endpoint also `/v1/responses`—if you change one, you must change the other.
  - OpenAI may reject unsupported models or invalid request schema in the `.jsonl`.

#### Node: Create a row in batch table
- **Type / role:** Supabase node (`supabase`) to insert (or create) tracking record.
- **Configuration choices:**
  - Table: `openai_batches`
  - Fields set:
    - `id = {{ $json.id }}`
    - `batch_status = {{ $json.status }}`
- **Connections:** From `Call Batch API` → to `Submission done`.
- **Credentials:** `supabaseApi` (named `b2b`)
- **Edge cases / failure modes:**
  - If `id` is PK and already exists → insert conflict (depends on Supabase node behavior; likely fails unless configured for upsert).
  - Schema mismatch: if `openai_batches` table/columns differ, update fails.
  - `batch_status` null not allowed per note; if OpenAI response lacks `status`, insert fails.

#### Node: Submission done
- **Type / role:** NoOp node marking end of submit phase.
- **Connections:** From `Create a row in batch table`.
- **Edge cases:** None (pure terminator/visual marker).

---

### Block 3 — FIFO Polling: Select Next Batch to Check
**Overview:** On a schedule, selects batches that are not terminal (completed/failed/expired/cancelled), ordered by creation time (FIFO intent), then requests status from OpenAI.

**Nodes involved:**
- `Cron Job (5 mins)`
- `Get the earliest uncompleted batch`
- `Call Batch API to retrieve batch object`

#### Node: Cron Job (5 mins)
- **Type / role:** Schedule Trigger (`scheduleTrigger`) entry point for polling.
- **Configuration choices:** Interval: every minute field (configured as minutes interval; effectively “every 5 mins” per node name, but JSON shows interval with `{field:"minutes"}` and no explicit “5” value).
- **Connections:** → `Get the earliest uncompleted batch`
- **Edge cases:**
  - If interval is misconfigured (e.g., every 1 minute), may spam OpenAI status checks.
  - If too infrequent, completed batches will sit longer before retrieval.

#### Node: Get the earliest uncompleted batch
- **Type / role:** Postgres node (`postgres`) selecting candidate batch rows.
- **Configuration choices:**
  - Schema: `public`
  - Table: `openai_batches`
  - Sort: by `created_at` ascending (FIFO ordering)
  - Where filters (all applied):
    - `batch_status != failed`
    - `batch_status != expired`
    - `batch_status != cancelled`
    - `batch_status != completed`
  - `returnAll: true` (returns all matching rows, not just the earliest)
- **Connections:** → `Call Batch API to retrieve batch object`
- **FIFO note:** Despite node name, **it does not `LIMIT 1`**, so multiple batches may be processed in a single run (n8n will execute downstream once per item).
- **Edge cases / failure modes:**
  - If table is empty → outputs no items; downstream doesn’t run (expected).
  - If statuses include other terminal values not excluded, they’ll keep being polled.
  - If many rows returned, you may hit OpenAI rate limits with many status calls per trigger.
  - Credential/network errors to Postgres.

#### Node: Call Batch API to retrieve batch object
- **Type / role:** HTTP Request to fetch batch status/details from OpenAI.
- **Configuration choices:**
  - `GET https://api.openai.com/v1/batches/{{ $json.id }}`
  - Auth: OpenAI credential
- **Connections:** → `If status = completed`
- **Edge cases / failure modes:**
  - If `$json.id` missing/blank → invalid URL.
  - 404 if batch id doesn’t exist (row corruption/manual edits).
  - 401/403 auth issues.

---

### Block 4 — Branch: Completed vs Not Completed
**Overview:** If batch is completed and has an output file, download and parse results; otherwise update only the batch status.

**Nodes involved:**
- `If status = completed`
- `Update status` (not completed path)
- `Download .jsonl result` (completed path)

#### Node: If status = completed
- **Type / role:** IF node branching on OpenAI batch object fields.
- **Conditions (AND):**
  1. `{{ $json.status }}` equals `completed`
  2. `{{ $json.output_file_id }}` exists
- **Outputs:**
  - **True** → `Download .jsonl result`
  - **False** → `Update status`
- **Edge cases:**
  - Some batches could be `completed` but have no `output_file_id` due to API changes/errors → goes to “false” branch and only updates status.
  - If OpenAI returns status `failed/expired/cancelled`, it will go “false” and be updated, but will also be excluded from future polling by the SQL filter.

#### Node: Update status
- **Type / role:** Supabase update to persist latest status for non-completed (or completed-without-output) batches.
- **Configuration choices:**
  - Table: `openai_batches`
  - Filter: `id eq {{ $json.id }}`
  - Updates:
    - `updated_at = {{ $now }}`
    - `batch_status = {{ $json.status }}`
- **Connections:** None further (ends that branch).
- **Edge cases / failure modes:**
  - If row does not exist → update affects 0 rows (silent or error depending on node behavior).
  - `$now` formatting: Supabase expects timestamp; n8n typically provides ISO string—ensure column type is compatible (`timestamptz` is fine).

#### Node: Download .jsonl result
- **Type / role:** HTTP Request to download the output file content from OpenAI Files API.
- **Configuration choices:**
  - `GET https://api.openai.com/v1/files/{{ $json.output_file_id }}/content`
  - Response format: `file` (binary)
  - Auth: OpenAI credential
- **Connections:** → `.jsonl to base64`
- **Edge cases / failure modes:**
  - Output file not ready immediately (rare if status says completed, but possible timing issues) → 404/409.
  - Large output file → memory pressure in n8n when handled as binary.

---

### Block 5 — Decode Output, Extract Results, Persist
**Overview:** Converts the downloaded NDJSON file into text, parses each line, extracts the assistant message text from each response, then writes results back into Supabase.

**Nodes involved:**
- `.jsonl to base64`
- `Decode base64`
- `Update status and result`
- `Retrieval done`

#### Node: .jsonl to base64
- **Type / role:** Extract From File node (`extractFromFile`) converting binary content into a JSON property.
- **Configuration choices:**
  - Operation: `binaryToPropery` (note the node’s operation name; intended meaning is “binary to property”)
- **Inputs/Outputs:**
  - Input: binary file from `Download .jsonl result`
  - Output: JSON field `data` holding base64 (as used by next node)
- **Edge cases / failure modes:**
  - If binary property name differs from default (usually `data`), conversion may fail or produce empty output.
  - Very large files increase memory usage.

#### Node: Decode base64
- **Type / role:** Code node parsing the NDJSON batch output and extracting texts.
- **Key logic/config:**
  - Reads base64 from `const base64Content = $input.first().json.data;`
  - Decodes to UTF-8, splits by newline, `JSON.parse` per line (skipping invalid lines)
  - For each parsed line:
    - Skips if `line.error` exists or `line.response.status_code !== 200`
    - Extracts `line.response.body.output`
    - Finds element where `type === 'message'`
    - Extracts `messageOutput.content[0].text`
  - Counts `parsed` and `errors`
  - Pulls batch metadata from earlier node:
    - `const batch = $('If status = completed').first().json;`
  - Outputs:
    - `id`, `status`, `output_file_id`, `result` (array of texts), `parsed`, `errors`
- **Connections:** → `Update status and result`
- **Version notes:** Code node v2; uses `Buffer` and the `$('Node Name')` selector.
- **Edge cases / failure modes:**
  - If OpenAI output schema differs (e.g., content is not `[ {text: ...} ]`) → extraction yields empty results and increments errors.
  - If multiple message parts exist, only the first `content[0].text` is taken.
  - If `$('If status = completed')` has no item (unexpected execution path), `.first()` will throw.
  - NDJSON lines that fail JSON parsing are dropped.

#### Node: Update status and result
- **Type / role:** Supabase update to store final status and extracted results.
- **Configuration choices:**
  - Filter: `id eq {{ $json.id }}`
  - Updates:
    - `updated_at = {{ $now }}`
    - `batch_status = {{ $json.status }}`
    - `output_file_id = {{ $json.output_file_id }}`
    - `result = {{ $json.result }}`
- **Connections:** → `Retrieval done`
- **Edge cases / failure modes:**
  - `result` column type must accept an array (per sticky note: `text[]`). If set as `text`/JSON incorrectly, update fails.
  - Large arrays may hit row size or request limits.

#### Node: Retrieval done
- **Type / role:** NoOp node marking end of retrieval phase.
- **Connections:** From `Update status and result`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start (mock data) | Manual Trigger | Entry point for submit phase with sample inputs | — | Convert to batch requests in .jsonl | ## 📤 Phase 1: Submit Batch |
| Convert to batch requests in .jsonl | Code | Build NDJSON batch file and output as binary | Start (mock data) | Call Files API | ## 📤 Phase 1: Submit Batch |
| Call Files API | HTTP Request | Upload `.jsonl` file to OpenAI Files API | Convert to batch requests in .jsonl | Call Batch API | ## 📤 Phase 1: Submit Batch |
| Call Batch API | HTTP Request | Create OpenAI batch job | Call Files API | Create a row in batch table | ## 📤 Phase 1: Submit Batch |
| Create a row in batch table | Supabase | Insert batch record into `openai_batches` | Call Batch API | Submission done | ## 📤 Phase 1: Submit Batch |
| Submission done | NoOp | End marker for submission | Create a row in batch table | — | ## 📤 Phase 1: Submit Batch |
| Cron Job (5 mins) | Schedule Trigger | Entry point for polling phase | — | Get the earliest uncompleted batch | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Get the earliest uncompleted batch | Postgres | Select not-terminal batches ordered by created time | Cron Job (5 mins) | Call Batch API to retrieve batch object | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Call Batch API to retrieve batch object | HTTP Request | Fetch batch status/details from OpenAI | Get the earliest uncompleted batch | If status = completed | ## 📥 Phase 2: Poll & Retrieve (cron) |
| If status = completed | IF | Branch completed vs not completed | Call Batch API to retrieve batch object | Download .jsonl result (true), Update status (false) | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Update status | Supabase | Update status for non-completed batch | If status = completed (false) | — | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Download .jsonl result | HTTP Request | Download output file content (binary) | If status = completed (true) | .jsonl to base64 | ## 📥 Phase 2: Poll & Retrieve (cron) |
| .jsonl to base64 | Extract From File | Convert downloaded binary to base64 in JSON | Download .jsonl result | Decode base64 | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Decode base64 | Code | Parse NDJSON and extract response texts | .jsonl to base64 | Update status and result | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Update status and result | Supabase | Persist final status + output file id + results array | Decode base64 | Retrieval done | ## 📥 Phase 2: Poll & Retrieve (cron) |
| Retrieval done | NoOp | End marker for retrieval | Update status and result | — | ## 📥 Phase 2: Poll & Retrieve (cron) |

**Global sticky note content applicable to the whole workflow (context):**  
“OpenAI Batch API (FIFO with Supabase / Postgres)” note describing table schema, credentials, customization, and FIFO `LIMIT 1` recommendation.

---

## 4. Reproducing the Workflow from Scratch

1. **Create database table in Supabase (and ensure Postgres access):**
   - Table name: `openai_batches`
   - Columns:
     - `id` (text, primary key)
     - `batch_status` (text, not null)
     - `output_file_id` (text, nullable)
     - `result` (text[], nullable)
     - `created_at` (timestamptz, default `now()`)
     - `updated_at` (timestamptz, nullable)

2. **Create/Open credentials in n8n:**
   - **OpenAI API credential** (used by all HTTP Request nodes to `api.openai.com`)
   - **Supabase API credential** (used by Supabase nodes)
   - **Postgres credential** pointing to the Supabase Postgres database (used by Postgres node)

3. **Phase 1 nodes (Submit):**
   1. Add **Manual Trigger** named `Start (mock data)`.
      - Provide test pinned data (optional) with:
        - `systemPrompt` (string)
        - `inputs` (array of strings)
   2. Add **Code** node named `Convert to batch requests in .jsonl`.
      - Paste logic to:
        - Convert `inputs` + `systemPrompt` into NDJSON lines targeting `/v1/responses`
        - Output as `binary.data` with base64 content and filename `batch_input.jsonl`
      - Set model (e.g., `gpt-5-mini`) in the code.
   3. Add **HTTP Request** node named `Call Files API`.
      - Method: `POST`
      - URL: `https://api.openai.com/v1/files`
      - Authentication: **OpenAI predefined credential**
      - Body: `multipart/form-data`
        - Field `purpose` = `batch`
        - Field `file` = **binary** from input, property name `data`
   4. Add **HTTP Request** node named `Call Batch API`.
      - Method: `POST`
      - URL: `https://api.openai.com/v1/batches`
      - Authentication: OpenAI credential
      - Body: JSON:
        - `input_file_id` = expression `{{ $json.id }}`
        - `endpoint` = `"/v1/responses"`
        - `completion_window` = `"24h"`
   5. Add **Supabase** node named `Create a row in batch table`.
      - Operation: Insert/Create row
      - Table: `openai_batches`
      - Fields:
        - `id` = `{{ $json.id }}`
        - `batch_status` = `{{ $json.status }}`
   6. Add **NoOp** node `Submission done`.

   7. **Connect Phase 1:**  
      `Start (mock data)` → `Convert to batch requests in .jsonl` → `Call Files API` → `Call Batch API` → `Create a row in batch table` → `Submission done`

4. **Phase 2 nodes (Poll & Retrieve):**
   1. Add **Schedule Trigger** named `Cron Job (5 mins)`.
      - Configure interval to **every 5 minutes** (ensure the value is actually 5 in the UI).
   2. Add **Postgres** node named `Get the earliest uncompleted batch`.
      - Operation: Select from `public.openai_batches`
      - Filter: exclude terminal statuses:
        - `batch_status != completed`
        - `batch_status != failed`
        - `batch_status != expired`
        - `batch_status != cancelled`
      - Sort by `created_at` ascending
      - For strict FIFO, set **Limit = 1** (recommended; otherwise it will process all matching rows).
   3. Add **HTTP Request** node named `Call Batch API to retrieve batch object`.
      - Method: `GET`
      - URL: `https://api.openai.com/v1/batches/{{ $json.id }}`
      - Authentication: OpenAI credential
   4. Add **IF** node named `If status = completed`.
      - Condition 1: `{{ $json.status }}` equals `completed`
      - Condition 2: `{{ $json.output_file_id }}` exists
      - Combine with AND
   5. False branch (not completed): add **Supabase** node `Update status`
      - Operation: Update
      - Table: `openai_batches`
      - Filter: `id eq {{ $json.id }}`
      - Fields:
        - `updated_at = {{ $now }}`
        - `batch_status = {{ $json.status }}`
   6. True branch (completed): add **HTTP Request** node `Download .jsonl result`
      - Method: `GET`
      - URL: `https://api.openai.com/v1/files/{{ $json.output_file_id }}/content`
      - Authentication: OpenAI credential
      - Response: **File** (binary)
   7. Add **Extract From File** node named `.jsonl to base64`
      - Operation: “binary to property”
      - Keep default binary property name `data` unless you changed it.
   8. Add **Code** node `Decode base64`
      - Decode base64 to string
      - Parse NDJSON lines
      - Extract `message` output text from each line
      - Also reference the completed batch object via `$('If status = completed').first().json`
   9. Add **Supabase** node `Update status and result`
      - Operation: Update
      - Table: `openai_batches`
      - Filter: `id eq {{ $json.id }}`
      - Fields:
        - `updated_at = {{ $now }}`
        - `batch_status = {{ $json.status }}`
        - `output_file_id = {{ $json.output_file_id }}`
        - `result = {{ $json.result }}`
   10. Add **NoOp** node `Retrieval done`.

   11. **Connect Phase 2:**  
      `Cron Job (5 mins)` → `Get the earliest uncompleted batch` → `Call Batch API to retrieve batch object` → `If status = completed`  
      - True: `Download .jsonl result` → `.jsonl to base64` → `Decode base64` → `Update status and result` → `Retrieval done`  
      - False: `Update status`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI Batch API (FIFO with Supabase / Postgres): submit `.jsonl` file, create batch, store status in `openai_batches`, cron polls earliest uncompleted, downloads output and stores results. | Workflow design note (global) |
| Supabase table schema required: `id` (text PK), `batch_status` (text), `output_file_id` (text nullable), `result` (text[] nullable), `created_at` default now(), `updated_at` nullable. | Database prerequisite |
| Customization: change model in “Convert to batch requests…” (currently `gpt-5-mini`), adjust cron interval, and for strict FIFO set Postgres select to `LIMIT 1`. | Operational tuning |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.