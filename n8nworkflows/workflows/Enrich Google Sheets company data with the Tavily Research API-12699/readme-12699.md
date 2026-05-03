Enrich Google Sheets company data with the Tavily Research API

https://n8nworkflows.xyz/workflows/enrich-google-sheets-company-data-with-the-tavily-research-api-12699


# Enrich Google Sheets company data with the Tavily Research API

## 1. Workflow Overview

**Title:** Enrich Google Sheets company data with the Tavily Research API  
**Workflow name in JSON:** `Data Enrichment`

**Purpose:**  
Reads company rows from a Google Sheet, launches a Tavily Research API job per row to find missing company attributes, polls Tavily until results are ready (up to ~5 minutes), then merges results back **only into existing columns** and appends the enriched rows into a separate (empty) destination sheet.

**Target use cases:**
- Filling missing company attributes (CEO/CTO, revenue, HQ, employees, industry, website/domain, etc.) in a structured sheet.
- Enrichment where you want to preserve your current schema (no new columns are created automatically).

### 1.1 Data Input
Reads source rows from Google Sheets and provides them as items.

### 1.2 Research Initiation
Captures the original column list, builds a dynamic `output_schema` for Tavily, and starts Tavily research requests.

### 1.3 Polling Loop (retry until completed / timeout)
Waits 30 seconds, checks research status by `request_id`, increments poll counter, and loops until completed or max polls reached (~5 minutes).

### 1.4 Data Merge + Output
Extracts structured fields from Tavily responses, maps them to the *existing* input columns using intelligent matching, then appends the enriched rows to a destination Google Sheet tab.

---

## 2. Block-by-Block Analysis

### Block 1 — Data Input

**Overview:** Loads company rows from a chosen Google Sheet/tab. This block is the entry point and provides the dataset to enrich.

**Nodes involved:**
- `Click to Start`
- `Read CSV file`

#### Node: Click to Start
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts the workflow on demand.
- **Configuration choices:** No parameters; standard manual start.
- **Inputs / outputs:**
  - **Output →** `Read CSV file`
- **Edge cases / failures:** None (only user-triggered).
- **Version notes:** TypeVersion `1`.

#### Node: Read CSV file
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — reads rows from a source tab.
- **Configuration choices (interpreted):**
  - Uses Google Sheets OAuth2 credentials (`Google Sheets account 9`).
  - `documentId`: placeholder to be replaced with the actual Google Sheet file ID.
  - `sheetName`: placeholder to be replaced with source tab name (e.g., `sample_companies`).
  - No explicit operation is shown in parameters; by default, this node is configured to read rows (typical “Read” behavior for Sheets nodes).
- **Inputs / outputs:**
  - **Input ←** `Click to Start`
  - **Output →** `Store Original Columns`
- **Edge cases / failures:**
  - OAuth permission issues (403) or expired token.
  - Wrong `documentId` / `sheetName` (404 / “sheet not found”).
  - Empty sheet → downstream code expects at least one item (see next block).
- **Version notes:** TypeVersion `4.5`.

**Sticky notes applying:**
- “Data Enrichment with Tavily Research API” (overall setup and behavior)
- “Section: Data Input — Reads company data from Google Sheets”
- “⚠️ Important Setup Step — Create an empty output sheet…”

---

### Block 2 — Research Initiation

**Overview:** Determines which columns exist in the input data, constructs a Tavily `output_schema` that matches those columns, and starts a Tavily research request per row.

**Nodes involved:**
- `Store Original Columns`
- `Start Tavily Research`
- `Combine with Request IDs`

#### Node: Store Original Columns
- **Type / role:** Code node (`n8n-nodes-base.code`) — prepares per-row research payload and preserves schema metadata.
- **Configuration choices (interpreted):**
  - Reads all input items.
  - Derives `originalColumns` from the first row’s JSON keys, excluding keys starting with `_`.
  - Creates a dynamic `output_schema` with `properties` for each original column; each property is `{ type: 'string', description: ... }`.
  - Adds helper metadata fields to each item:
    - `_row_number`: `index + 2` (assumes row 1 is header, data begins at row 2)
    - `_original_columns`: the list of columns to preserve/merge into
    - `research_body`: `{ input: "...", output_schema: ... }`
  - Builds `researchInput` using one of:
    - `company_name` (preferred), or `domain`, or `website`, else `"unknown"`.
- **Key variables/expressions:**
  - `originalColumns = Object.keys(firstItem).filter(col => !col.startsWith('_'))`
  - `research_body.input = 'Research company ' + company + '. Find information about: ' + originalColumns.join(', ') + '.';`
- **Inputs / outputs:**
  - **Input ←** `Read CSV file`
  - **Output →** `Start Tavily Research`
- **Edge cases / failures:**
  - **Empty input**: `items[0]` is undefined; `firstItem` becomes `{}`, `originalColumns` becomes `[]`, and research becomes underspecified. You may want to guard and stop workflow if no rows.
  - If sheet headers are inconsistent across rows, only the first row’s columns drive the schema.
  - If none of `company_name/domain/website` exist, query uses `"unknown"` which likely produces poor results.
- **Version notes:** TypeVersion `2`.

#### Node: Start Tavily Research
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — starts an asynchronous Tavily research job.
- **Configuration choices (interpreted):**
  - `POST https://api.tavily.com/research`
  - Sends JSON body: `JSON.stringify($json.research_body)`
  - Adds header `Content-Type: application/json`
  - Authentication: `httpHeaderAuth` credential (`Header Auth account 6`) — typically used to send Tavily API key in a header (exact header name is defined in the credential).
- **Inputs / outputs:**
  - **Input ←** `Store Original Columns`
  - **Output →** `Combine with Request IDs`
- **Edge cases / failures:**
  - Missing/incorrect Tavily API key header → 401/403.
  - Tavily API rate limiting → 429.
  - Network timeouts.
  - If Tavily changes response format and doesn’t return `request_id`, polling will fail downstream.
- **Version notes:** TypeVersion `4.2`.

#### Node: Combine with Request IDs
- **Type / role:** Code node — merges Tavily `request_id` back into the original items and initializes polling state.
- **Configuration choices (interpreted):**
  - Pulls originals from `Store Original Columns` via `$('Store Original Columns').all()`
  - Reads Tavily start results from `$input.all()`
  - Outputs items where each contains:
    - all original fields
    - `request_id` from Tavily (or `null`)
    - `poll_count: 0`
- **Key variables/expressions:**
  - `request_id: researchResults[i]?.json?.request_id || null`
- **Inputs / outputs:**
  - **Input ←** `Start Tavily Research`
  - **Output →** `Wait 30s`
- **Edge cases / failures:**
  - Assumes item order and count match between originals and Tavily responses. If an upstream node drops/filters items or Tavily errors per-item, indexes can misalign.
  - If `request_id` is null, `Check Research Status` will request an invalid URL.
- **Version notes:** TypeVersion `2`.

**Sticky notes applying:**
- “Section: Research Initiation — Prepares data and starts Tavily research requests”

---

### Block 3 — Polling Loop

**Overview:** Implements a polling mechanism: wait 30 seconds, check Tavily status, repeat until status is `completed` or the poll count reaches 10 (≈ 5 minutes).

**Nodes involved:**
- `Wait 30s`
- `Check Research Status`
- `Combine Status`
- `Research Done?`
- `Under 5 min?`

#### Node: Wait 30s
- **Type / role:** Wait (`n8n-nodes-base.wait`) — delay between polling attempts.
- **Configuration choices (interpreted):**
  - Wait amount: 30 seconds.
  - Webhook ID set to `poll-wait` (used internally for wait/resume handling).
- **Inputs / outputs:**
  - **Input ←** `Combine with Request IDs` (initial entry) and `Under 5 min?` (loop)
  - **Output →** `Check Research Status`
- **Edge cases / failures:**
  - If n8n is not configured to support wait/resume reliably (e.g., worker restarts, queue mode misconfig), waits can be disrupted.
- **Version notes:** TypeVersion `1.1`.

#### Node: Check Research Status
- **Type / role:** HTTP Request — polls Tavily research job status/result.
- **Configuration choices (interpreted):**
  - `GET https://api.tavily.com/research/{{ $json.request_id }}`
  - Authentication: same `httpHeaderAuth` credential (`Header Auth account 6`)
- **Key expressions:**
  - URL expression uses `{{$json.request_id}}`
- **Inputs / outputs:**
  - **Input ←** `Wait 30s`
  - **Output →** `Combine Status`
- **Edge cases / failures:**
  - `request_id` is null/empty → malformed request, likely 404.
  - Rate limiting (429) if many rows are polling simultaneously.
  - Tavily response shape differences: downstream expects `status` field.
- **Version notes:** TypeVersion `4.2`.

#### Node: Combine Status
- **Type / role:** Code node — merges the latest Tavily status payload into each item and increments `poll_count`.
- **Configuration choices (interpreted):**
  - Reads the pre-status items from `$('Wait 30s').all()` (so it can preserve prior fields).
  - Reads current status results from `$input.all()`
  - Outputs:
    - `poll_count: (orig.json.poll_count || 0) + 1`
    - `research_status: status.status || 'unknown'`
    - `research_result: status` (full response preserved)
- **Inputs / outputs:**
  - **Input ←** `Check Research Status`
  - **Output →** `Research Done?`
- **Edge cases / failures:**
  - Assumes alignment by index between wait items and HTTP results.
  - If Tavily returns non-JSON or error body, `statusResults[i]?.json` may not contain `status`.
- **Version notes:** TypeVersion `2`.

#### Node: Research Done?
- **Type / role:** IF (`n8n-nodes-base.if`) — checks whether Tavily finished.
- **Configuration choices (interpreted):**
  - Condition: `{{$json.research_status}} == "completed"`
  - If **true** → go to merge/output.
  - If **false** → go to timeout check.
- **Inputs / outputs:**
  - **Input ←** `Combine Status`
  - **True output →** `Merge All Data (Existing Columns Only)`
  - **False output →** `Under 5 min?`
- **Edge cases / failures:**
  - Any status not exactly `"completed"` (e.g., `"complete"`, `"succeeded"`) will never end; you may need to broaden accepted terminal statuses if Tavily differs.
- **Version notes:** TypeVersion `2`.

#### Node: Under 5 min?
- **Type / role:** IF — enforces max polling attempts.
- **Configuration choices (interpreted):**
  - Condition: `{{$json.poll_count}} < 10`
  - If **true** → loop back to `Wait 30s`
  - If **false** → stop polling and proceed to merge anyway (best-effort).
- **Inputs / outputs:**
  - **Input ←** `Research Done?` (false branch)
  - **True output →** `Wait 30s`
  - **False output →** `Merge All Data (Existing Columns Only)`
- **Edge cases / failures:**
  - If `poll_count` is missing or non-numeric, comparison may fail strict validation; code ensures it’s incremented, but initial values matter.
- **Version notes:** TypeVersion `2`.

**Sticky notes applying:**
- “Section: Polling Loop — Waits and checks research status until complete (max 5 min)”

---

### Block 4 — Data Merge + Output

**Overview:** Extracts structured fields from Tavily’s response, matches them to existing sheet columns, fills only empty original cells, and appends results to the destination sheet.

**Nodes involved:**
- `Merge All Data (Existing Columns Only)`
- `Enrich CSV file`

#### Node: Merge All Data (Existing Columns Only)
- **Type / role:** Code node — schema-preserving merge and intelligent field mapping.
- **Configuration choices (interpreted):**
  - For each item:
    - Reads `research_result` (the full Tavily status/result payload).
    - Reads `_original_columns` to decide which keys are allowed in output.
    - Tries to locate extracted structured data from multiple possible locations (in priority order):
      1. `research.structured_output` (object)
      2. `research.content` (object)
      3. `research.content` (string JSON parse attempt)
      4. `research.output` (object)
      5. `research.data` (object)
      6. `research.result` (object)
    - Produces `merged` output containing **only** the original columns (excluding internal `_...` keys).
    - If original cell is non-empty, it is preserved.
    - If empty, tries to fill from Tavily extracted fields using matching logic:
      - exact match (normalized)
      - partial match
      - variations map (ceo/cto/cfo, revenue, headquarters, employees, etc.)
    - If no value found → sets empty string `''`.
- **Key logic notes:**
  - `normalize(str)` lowercases and strips non-alphanumeric, enabling fuzzy matching.
  - Variation mapping is helpful when sheet column names differ slightly (e.g., `employee_count` vs `employees`).
- **Inputs / outputs:**
  - **Input ←** `Research Done?` (true) OR `Under 5 min?` (false timeout branch)
  - **Output →** `Enrich CSV file`
- **Edge cases / failures:**
  - If Tavily returns unstructured text only (no structured object / JSON), most columns remain blank (by design).
  - If your original sheet columns are not representative (e.g., “Notes”, “Priority”), Tavily schema will ask for those too, often producing irrelevant data.
  - If `_original_columns` is missing, output will be empty object (no columns).
- **Version notes:** TypeVersion `2`.

#### Node: Enrich CSV file
- **Type / role:** Google Sheets node — appends enriched rows to the destination tab.
- **Configuration choices (interpreted):**
  - Operation: `append`
  - Destination:
    - `documentId`: placeholder to be replaced with Google Sheet ID (same file recommended)
    - `sheetName`: placeholder to be replaced with destination tab name (e.g., `enriched_companies`)
  - Column mapping is explicitly defined for these fields:
    - `company_name`, `no_of_employees`, `hq_based`, `industry`, `website`, `domain`, `CTO`, `Revenue`
  - Mapping mode: auto-map input data, but with a provided schema and explicit values.
- **Inputs / outputs:**
  - **Input ←** `Merge All Data (Existing Columns Only)`
  - **Output:** end of workflow
- **Edge cases / failures:**
  - Destination sheet/tab must exist and be accessible; otherwise append fails.
  - If the merged output does not include the exact column names expected here (case-sensitive, e.g. `CTO` vs `cto`, `Revenue` vs `revenue`), fields may be blank or not mapped as expected.
  - If you add more columns to your input sheet, the merge node will output them, but this Sheets node will **not** automatically write them unless you also add them to this node’s column mapping/schema.
- **Version notes:** TypeVersion `4.5`.

**Sticky notes applying:**
- “Section: Data Output — Merges enriched data and writes to Google Sheets”
- “⚠️ Important Setup Step — Create an empty output sheet…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Click to Start | Manual Trigger | Manual entry point | — | Read CSV file | Data Enrichment with Tavily Research API… |
| Read CSV file | Google Sheets | Read source company rows | Click to Start | Store Original Columns | Section: Data Input — Reads company data from Google Sheets; Data Enrichment with Tavily Research API…; ⚠️ Important Setup Step… |
| Store Original Columns | Code | Capture original schema; build Tavily output schema and per-row payload | Read CSV file | Start Tavily Research | Section: Research Initiation — Prepares data and starts Tavily research requests; Data Enrichment with Tavily Research API… |
| Start Tavily Research | HTTP Request | Start Tavily async research job | Store Original Columns | Combine with Request IDs | Section: Research Initiation — Prepares data and starts Tavily research requests; Data Enrichment with Tavily Research API… |
| Combine with Request IDs | Code | Merge `request_id` into each item; init `poll_count` | Start Tavily Research | Wait 30s | Section: Research Initiation — Prepares data and starts Tavily research requests; Data Enrichment with Tavily Research API… |
| Wait 30s | Wait | Delay between polling attempts | Combine with Request IDs; Under 5 min? (true) | Check Research Status | Section: Polling Loop — Waits and checks research status until complete (max 5 min); Data Enrichment with Tavily Research API… |
| Check Research Status | HTTP Request | Poll Tavily by `request_id` | Wait 30s | Combine Status | Section: Polling Loop — Waits and checks research status until complete (max 5 min); Data Enrichment with Tavily Research API… |
| Combine Status | Code | Increment poll counter; attach status + full result | Check Research Status | Research Done? | Section: Polling Loop — Waits and checks research status until complete (max 5 min); Data Enrichment with Tavily Research API… |
| Research Done? | IF | Route based on `research_status == completed` | Combine Status | Merge All Data (true); Under 5 min? (false) | Section: Polling Loop — Waits and checks research status until complete (max 5 min); Data Enrichment with Tavily Research API… |
| Under 5 min? | IF | Enforce max attempts (`poll_count < 10`) | Research Done? (false) | Wait 30s (true); Merge All Data (false) | Section: Polling Loop — Waits and checks research status until complete (max 5 min); Data Enrichment with Tavily Research API… |
| Merge All Data (Existing Columns Only) | Code | Extract Tavily structured output; fill only existing columns | Research Done? (true); Under 5 min? (false) | Enrich CSV file | Section: Data Output — Merges enriched data and writes to Google Sheets; Data Enrichment with Tavily Research API… |
| Enrich CSV file | Google Sheets | Append enriched rows to destination tab | Merge All Data (Existing Columns Only) | — | Section: Data Output — Merges enriched data and writes to Google Sheets; ⚠️ Important Setup Step…; Data Enrichment with Tavily Research API… |
| Sticky Note | Sticky Note | Comment / instructions | — | — |  |
| Section: Data Input | Sticky Note | Comment / section header | — | — |  |
| Section: Research Initiation | Sticky Note | Comment / section header | — | — |  |
| Section: Polling Loop | Sticky Note | Comment / section header | — | — |  |
| Section: Data Output | Sticky Note | Comment / section header | — | — |  |
| Sticky Note1 | Sticky Note | Comment / important setup | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `Click to Start`

3) **Add Google Sheets (read source)**
   - Node: **Google Sheets**
   - Name: `Read CSV file`
   - Credentials: **Google Sheets OAuth2**
     - Connect the Google account that has access to the spreadsheet.
   - Set:
     - **Document ID**: your Google Sheets file ID
     - **Sheet Name**: your source tab (e.g., `sample_companies`)
   - Connect: `Click to Start` → `Read CSV file`

4) **Add Code node to store original columns + build Tavily schema**
   - Node: **Code**
   - Name: `Store Original Columns`
   - Paste the logic equivalent to:
     - Determine `originalColumns` from first row keys (exclude `_...`)
     - Add `_row_number`, `_original_columns`
     - Build `research_body = { input, output_schema }`
   - Connect: `Read CSV file` → `Store Original Columns`

5) **Add HTTP Request to start Tavily research**
   - Node: **HTTP Request**
   - Name: `Start Tavily Research`
   - Method: **POST**
   - URL: `https://api.tavily.com/research`
   - Authentication: **Header Auth** (Generic Credential → HTTP Header Auth)
     - Configure credential to send your Tavily API key in the required header (as per Tavily docs / your account setup).
   - Headers:
     - `Content-Type: application/json`
   - Body:
     - Send body: **true**
     - Body type: **JSON**
     - Body value expression: `{{ JSON.stringify($json.research_body) }}`
   - Connect: `Store Original Columns` → `Start Tavily Research`

6) **Add Code node to combine `request_id`**
   - Node: **Code**
   - Name: `Combine with Request IDs`
   - Implement:
     - Read originals from `Store Original Columns`
     - Read Tavily response from input
     - Output merged JSON with `request_id` and `poll_count: 0`
   - Connect: `Start Tavily Research` → `Combine with Request IDs`

7) **Add Wait node**
   - Node: **Wait**
   - Name: `Wait 30s`
   - Amount: **30 seconds**
   - Connect: `Combine with Request IDs` → `Wait 30s`

8) **Add HTTP Request to check Tavily status**
   - Node: **HTTP Request**
   - Name: `Check Research Status`
   - Method: **GET**
   - URL (expression): `https://api.tavily.com/research/{{ $json.request_id }}`
   - Authentication: same **Header Auth** credential as step 5
   - Connect: `Wait 30s` → `Check Research Status`

9) **Add Code node to combine status + increment poll**
   - Node: **Code**
   - Name: `Combine Status`
   - Implement:
     - Merge fields from the pre-request item with the status payload
     - `poll_count++`
     - Set `research_status` from response `status`
     - Store full response in `research_result`
   - Connect: `Check Research Status` → `Combine Status`

10) **Add IF node: research completed?**
   - Node: **IF**
   - Name: `Research Done?`
   - Condition (string equals):
     - Left: `{{ $json.research_status }}`
     - Right: `completed`
   - Connect: `Combine Status` → `Research Done?`

11) **Add IF node: under max polling?**
   - Node: **IF**
   - Name: `Under 5 min?`
   - Condition (number less than):
     - Left: `{{ $json.poll_count }}`
     - Right: `10`
   - Connect: `Research Done?` **false** → `Under 5 min?`

12) **Close the polling loop**
   - Connect: `Under 5 min?` **true** → `Wait 30s`
   - Connect: `Under 5 min?` **false** → `Merge All Data (Existing Columns Only)` (created next)

13) **Add Code node to merge Tavily results into existing columns**
   - Node: **Code**
   - Name: `Merge All Data (Existing Columns Only)`
   - Implement the merge behavior:
     - Only output keys from `_original_columns`
     - Preserve non-empty originals
     - Extract structured Tavily data from likely fields (`structured_output`, `content`, etc.)
     - Use normalized/variation matching to map fields
   - Connect: `Research Done?` **true** → `Merge All Data (Existing Columns Only)`

14) **Add Google Sheets (append to destination)**
   - Node: **Google Sheets**
   - Name: `Enrich CSV file`
   - Credentials: **Google Sheets OAuth2** (same account is fine)
   - Operation: **Append**
   - Destination:
     - **Document ID**: same Google Sheet ID (or another, if desired)
     - **Sheet Name**: destination tab (e.g., `enriched_companies`) — must already exist and ideally be empty
   - Map columns you want written (must match destination headers), e.g.:
     - `company_name`, `no_of_employees`, `hq_based`, `industry`, `website`, `domain`, `CTO`, `Revenue`
   - Connect: `Merge All Data (Existing Columns Only)` → `Enrich CSV file`

15) **Create the destination sheet tab**
   - In Google Sheets, create an empty tab (e.g., `enriched_companies`) with the same headers you expect to append.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Fill in missing company data in your Google Sheet using Tavily’s web research. Only enriches existing columns; add columns manually if you want them enriched. | Sticky note “Data Enrichment with Tavily Research API” |
| Reads rows → Tavily research → maps results to existing columns → writes to a new sheet → retries until ready (up to ~5 minutes). | Sticky note “Data Enrichment with Tavily Research API” |
| Setup steps: prepare sheet; create columns; create empty output sheet; add Google Sheets OAuth; add Tavily API key via header auth; configure IDs and sheet names; run. | Sticky note “Data Enrichment with Tavily Research API” |
| Important: create an empty output sheet in the same document (example input “companies”, output “enriched_companies”). | Sticky note “⚠️ Important Setup Step” |