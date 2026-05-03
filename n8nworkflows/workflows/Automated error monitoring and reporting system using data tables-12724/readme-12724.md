Automated error monitoring and reporting system using data tables

https://n8nworkflows.xyz/workflows/automated-error-monitoring-and-reporting-system-using-data-tables-12724


# Automated error monitoring and reporting system using data tables

## 1. Workflow Overview

**Workflow name:** ✅ Error Handling Workflow  
**Purpose:** Centralize n8n workflow failures into a Data Table, then periodically generate an AI-assisted consolidated error report and email it, while marking reported errors so they are not emailed again.

**Target use cases**
- Production monitoring: capture failures from multiple workflows and keep a log.
- Noise control: ignore “manual” (test) executions.
- Periodic reporting: send a single digest only when there are enough errors or when errors have been pending too long.

### 1.1 Error capture & storage (event-driven)
Triggered whenever any workflow errors. Filters out manual runs, maps key fields, and inserts the error into a Data Table.

### 1.2 Scheduled retrieval & threshold gating (time-driven)
Runs hourly, fetches errors that have not been emailed, sorts them, aggregates them, then continues only if there are >5 errors or the oldest pending error is at least 24 hours old.

### 1.3 Report generation (AI + HTML)
Builds an HTML table of errors and asks an OpenAI agent to create an HTML summary with suggested fixes and quantitative counts.

### 1.4 Notification & state update
Emails the combined AI summary + HTML table, then updates each included record’s `lastEmailedAt` so it will not be picked up next time.

---

## 2. Block-by-Block Analysis

### Block 1 — Store errors (automatic executions only)
**Overview:** Captures workflow failures via Error Trigger, ignores manual runs, and stores error details in the “Workflow Errors” Data Table.

**Nodes involved:**
- Error Trigger
- Ignore Manual Failures
- map fields to data table
- Time Saved
- Insert Error Details

#### Node: **Error Trigger**
- **Type / role:** `n8n-nodes-base.errorTrigger` — entry point fired on workflow execution errors.
- **Configuration:** Default; no parameters.
- **Inputs/Outputs:** No inputs. Output flows to **Ignore Manual Failures**.
- **Key data used downstream:**  
  - `execution.mode`, `execution.id`, `execution.url`  
  - `execution.error.message`, `execution.error.stack`  
  - `execution.lastNodeExecuted`  
  - `workflow.id`, `workflow.name`
- **Edge cases / failures:**
  - Some error payloads may not include `execution.error.message`/`stack` (depending on failure type); later expressions should guard against missing fields.

#### Node: **Ignore Manual Failures**
- **Type / role:** `n8n-nodes-base.filter` — prevents storing errors from manual/test runs.
- **Configuration choices:**
  - Condition: `execution.mode != "manual"`
  - Expression: `{{ $('Error Trigger').item.json.execution.mode }}`
- **Inputs/Outputs:** Input from **Error Trigger**, output to **map fields to data table**.
- **Edge cases / failures:**
  - If `execution.mode` is missing, strict validation may cause unexpected behavior; consider defaulting to `"automatic"` or handling null.

#### Node: **map fields to data table**
- **Type / role:** `n8n-nodes-base.set` — normalizes the error payload into consistent fields.
- **Configuration choices (fields set):**
  - `workflowId` = Error Trigger workflow id
  - `executionId` = Error Trigger execution id
  - `errorMessage` = `execution.error.message` or fallback to `$('Error Trigger').item.json.values().toJsonString()`
  - `errorStack` = stack trace
  - `lastNodeExecuted` = last node executed
  - `workflowName` = workflow name
  - `executionDate` = `$now.format('yyyy-MM-dd')`
  - `executionUrl` = execution URL
- **Inputs/Outputs:** From **Ignore Manual Failures** → to **Time Saved**.
- **Notable expressions:**
  - Fallback: `execution.error.message ?? ...toJsonString()`
- **Edge cases / failures:**
  - `values().toJsonString()` is uncommon in many n8n examples; if this fails in your environment/version, replace with `JSON.stringify($('Error Trigger').item.json)`.

#### Node: **Time Saved**
- **Type / role:** `n8n-nodes-base.timeSaved` — n8n cloud feature node used for measuring/estimating time saved.
- **Configuration:** `minutesSaved = 1`
- **Inputs/Outputs:** From **map fields to data table** → to **Insert Error Details**.
- **Edge cases / failures:** Usually none; if removed, it won’t affect logic—only metrics.

#### Node: **Insert Error Details**
- **Type / role:** `n8n-nodes-base.dataTable` — inserts a new error record into a Data Table.
- **Configuration choices:**
  - Operation: **Insert**
  - Target Data Table: **Workflow Errors** (`dataTableId` cached name “Workflow Errors”)
  - Columns written:
    - `workflowId`, `executionId`, `errorMessage`, `errorStack`, `lastNodeExecuted`, `workflowName`, `executionDate`, `executionUrl`
  - Note: `lastEmailedAt` exists in schema but is not set on insert (so it stays empty).
- **Inputs/Outputs:** From **Time Saved**. No downstream connections shown.
- **Edge cases / failures:**
  - Data Table permissions/project mismatch can fail.
  - Schema mismatch (missing columns or different types) will fail.
  - If `execution.error.message` is null and you don’t use the Set node’s fallback, you may store empty messages.

---

### Block 2 — Filter & prepare errors to report (hourly)
**Overview:** Every hour, fetches all errors not yet emailed, sorts them by creation time, aggregates them into one item, and gates reporting based on volume or age.

**Nodes involved:**
- Run every hour
- Get Errors that were not Emailed
- Sort
- Aggregate
- high error count or been a day1

#### Node: **Run every hour**
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration:** Interval rule: every **1 hour**.
- **Outputs:** To **Get Errors that were not Emailed**.
- **Edge cases / failures:** None typical; ensure workflow is active.

#### Node: **Get Errors that were not Emailed**
- **Type / role:** `n8n-nodes-base.dataTable` — retrieves pending errors.
- **Configuration choices:**
  - Operation: **Get**
  - Filter: `lastEmailedAt` **isEmpty**
  - Data Table: **Workflow Errors**
- **Outputs:** To **Sort**.
- **Edge cases / failures:**
  - If Data Table returns no items, downstream nodes must handle empty arrays (see filter’s “oldest error” calculation).

#### Node: **Sort**
- **Type / role:** `n8n-nodes-base.sort` — orders errors so “oldest” can be detected reliably.
- **Configuration:** Sort field: `createdAt` (ascending by default).
- **Inputs/Outputs:** From **Get Errors that were not Emailed** → to **Aggregate**.
- **Edge cases / failures:**
  - Assumes Data Table records include `createdAt`. If not present, sorting may be ineffective or error out.

#### Node: **Aggregate**
- **Type / role:** `n8n-nodes-base.aggregate` — combines all items into a single item for downstream processing.
- **Configuration:** “Aggregate All Item Data” (produces a single item with `data` array).
- **Inputs/Outputs:** From **Sort** → to **high error count or been a day1**.
- **Edge cases / failures:**
  - If no incoming items, output may be empty or may create one item with empty `data` depending on node behavior/version. The next filter should safely handle both.

#### Node: **high error count or been a day1**
- **Type / role:** `n8n-nodes-base.filter` — decides whether to send a report now.
- **Configuration choices (OR logic):**
  1. Condition A: length of `{{$json.data}}` > 5  
  2. Condition B: oldest pending error is >= 24 hours old  
     Expression:  
     `{{ DateTime.now().diff(DateTime.fromISO($('Sort').first()?.json?.createdAt), 'hours').hours.round() >= 24 }}`
- **Outputs:** Two parallel outputs to:
  - **Generate Workflow Errors Table HTML**
  - **Split Out**
- **Edge cases / failures:**
  - If there are no errors, `$('Sort').first()?.json?.createdAt` becomes `undefined`; `DateTime.fromISO(undefined)` can produce an invalid DateTime and may throw or evaluate unexpectedly. A safer guard is recommended, e.g. check item count first.

---

### Block 3 — Generate & send error notification (AI + HTML + email)
**Overview:** Creates an HTML table from aggregated errors, uses an OpenAI agent to summarize and suggest fixes (HTML output), then emails both the summary and the table.

**Nodes involved:**
- Generate Workflow Errors Table HTML
- OpenAI Chat Model
- Calculator
- AI Error Summarizer
- Email Error Details

#### Node: **Generate Workflow Errors Table HTML**
- **Type / role:** `n8n-nodes-base.html` — renders an HTML table using the aggregated error list.
- **Configuration choices:**
  - HTML template loops over `$('Aggregate').item.json.data`
  - Creates table rows with alternating background colors
  - Formats timestamp from `record.createdAt` into UTC-like string
  - Includes a “View” link to `record.executionUrl`
- **Inputs/Outputs:** From **high error count or been a day1** → to **AI Error Summarizer**.
- **Key expressions:**
  - `$('Aggregate').item.json.data.map(...)`
- **Edge cases / failures:**
  - If `Aggregate` produced no `data`, the template may fail unless `data` is always an array.
  - If `executionUrl` is missing, link will be broken.

#### Node: **OpenAI Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM backend to the agent.
- **Configuration:** Model `gpt-4.1-mini`.
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Connections:** Provides `ai_languageModel` input to **AI Error Summarizer**.
- **Edge cases / failures:**
  - Auth/insufficient quota/model access errors.
  - Network timeouts; consider retries in production.

#### Node: **Calculator**
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — optional tool available to the agent.
- **Connections:** Provided as `ai_tool` to **AI Error Summarizer**.
- **Edge cases / failures:** Minimal; only used if agent calls it.

#### Node: **AI Error Summarizer**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — produces an HTML summary of errors with prioritization and fix suggestions.
- **Configuration choices:**
  - Prompt text includes the entire aggregated errors list:  
    `Workflow Errors: {{ JSON.stringify($('Aggregate').item.json.data) }}`
  - System message:
    - Summarize in concise pointers
    - Include quantitative representation
    - Prioritize frequent errors
    - Output **HTML only** (no backticks or extra text)
    - Current time is injected: `Time right now: {{ $now }}`
- **Inputs/Outputs:**
  - Main input from **Generate Workflow Errors Table HTML**
  - `ai_languageModel` from **OpenAI Chat Model**
  - `ai_tool` from **Calculator**
  - Main output to **Email Error Details**
- **Edge cases / failures:**
  - If `Aggregate` contains a large array, `JSON.stringify` may exceed token limits; consider truncation or grouping.
  - Agent might not strictly comply with “HTML only”; you may want a cleanup/validation step before emailing.

#### Node: **Email Error Details**
- **Type / role:** `n8n-nodes-base.gmail` — sends the digest email.
- **Configuration choices:**
  - To: `user@example.com`
  - Subject: `n8n Workflow Errors - {{ $now.minus(1,'days').format('yyyy-MM-dd') }}`
  - Message body combines:
    - AI summary: `{{ $json.output[0].content[0].text }}`
    - HTML table from node named **Code in JavaScript**: `{{ $('Code in JavaScript').item.json.html }}`
  - Sender name: “Harshal Patil”
  - Append attribution: disabled
- **Credentials:** Gmail OAuth2 (“Gmail account hpg99”).
- **Inputs/Outputs:** From **AI Error Summarizer**; no outgoing connections.
- **Critical issue (likely bug):**
  - The workflow has **no node named “Code in JavaScript”**. The HTML table node is named **Generate Workflow Errors Table HTML**.
  - This expression will fail at runtime. It should probably be:  
    `{{ $('Generate Workflow Errors Table HTML').item.json.html }}`
- **Edge cases / failures:**
  - Gmail OAuth token expiry/permission issues.
  - Sending limits/quota.
  - Invalid HTML or overly large email content.

---

### Block 4 — Update errors that were notified
**Overview:** After deciding to send a report, split the aggregated list back into individual records and update each row’s `lastEmailedAt` timestamp.

**Nodes involved:**
- Split Out
- Update Last Emailed At

#### Node: **Split Out**
- **Type / role:** `n8n-nodes-base.splitOut` — converts the aggregated `data` array into individual items.
- **Configuration:** Field to split: `data`
- **Inputs/Outputs:** From **high error count or been a day1** → to **Update Last Emailed At**
- **Edge cases / failures:**
  - If `data` is missing or not an array, split will fail.

#### Node: **Update Last Emailed At**
- **Type / role:** `n8n-nodes-base.dataTable` — marks each emailed error as “emailed”.
- **Configuration choices:**
  - Operation: **Update**
  - Filter: `id = {{ $json.id }}`
  - Column update: `lastEmailedAt = {{ $now }}`
  - Data Table: **Workflow Errors**
- **Inputs/Outputs:** From **Split Out**; no outgoing connections.
- **Edge cases / failures:**
  - Requires each split item to include the Data Table row `id`. If the “Get” operation returns it, OK; if not, updates will do nothing.
  - If you email fails but updates still run (current workflow branches from the filter, not from the email success), errors may be marked emailed even when not actually sent.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Error Trigger | n8n-nodes-base.errorTrigger | Capture workflow failures | — | Ignore Manual Failures | ## Store errors; Ignore Manual Errors and store automatic errors into our Error Data Table |
| Ignore Manual Failures | n8n-nodes-base.filter | Drop manual execution errors | Error Trigger | map fields to data table | ## Store errors; Ignore Manual Errors and store automatic errors into our Error Data Table |
| map fields to data table | n8n-nodes-base.set | Normalize error fields for storage | Ignore Manual Failures | Time Saved | ## Store errors; Ignore Manual Errors and store automatic errors into our Error Data Table |
| Time Saved | n8n-nodes-base.timeSaved | Metrics/time-saved tracking | map fields to data table | Insert Error Details | ## Store errors; Ignore Manual Errors and store automatic errors into our Error Data Table |
| Insert Error Details | n8n-nodes-base.dataTable | Insert error row into Data Table | Time Saved | — | ## Store errors; Ignore Manual Errors and store automatic errors into our Error Data Table |
| Run every hour | n8n-nodes-base.scheduleTrigger | Hourly trigger for digest | — | Get Errors that were not Emailed | ## Filter & Prepare errors to report; Sort errors that are to be reported and filter only if error count is beyond a threshold |
| Get Errors that were not Emailed | n8n-nodes-base.dataTable | Fetch pending (unemailed) errors | Run every hour | Sort | ## Filter & Prepare errors to report; Sort errors that are to be reported and filter only if error count is beyond a threshold |
| Sort | n8n-nodes-base.sort | Order pending errors by age | Get Errors that were not Emailed | Aggregate | ## Filter & Prepare errors to report; Sort errors that are to be reported and filter only if error count is beyond a threshold |
| Aggregate | n8n-nodes-base.aggregate | Consolidate errors into one item | Sort | high error count or been a day1 | ## Filter & Prepare errors to report; Sort errors that are to be reported and filter only if error count is beyond a threshold |
| high error count or been a day1 | n8n-nodes-base.filter | Gate reporting by count or age | Aggregate | Generate Workflow Errors Table HTML; Split Out | ## Filter & Prepare errors to report; Sort errors that are to be reported and filter only if error count is beyond a threshold |
| Generate Workflow Errors Table HTML | n8n-nodes-base.html | Build HTML error table | high error count or been a day1 | AI Error Summarizer | ## Generate & Send Error Notification; Consolidate errors into a single table with an email with AI Powered Insights & sends the Email |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | — | AI Error Summarizer (ai_languageModel) | ## Generate & Send Error Notification; Consolidate errors into a single table with an email with AI Powered Insights & sends the Email |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Optional agent tool | — | AI Error Summarizer (ai_tool) | ## Generate & Send Error Notification; Consolidate errors into a single table with an email with AI Powered Insights & sends the Email |
| AI Error Summarizer | @n8n/n8n-nodes-langchain.agent | Create HTML summary + remediation hints | Generate Workflow Errors Table HTML | Email Error Details | ## Generate & Send Error Notification; Consolidate errors into a single table with an email with AI Powered Insights & sends the Email |
| Email Error Details | n8n-nodes-base.gmail | Email the report | AI Error Summarizer | — | ## Generate & Send Error Notification; Consolidate errors into a single table with an email with AI Powered Insights & sends the Email |
| Split Out | n8n-nodes-base.splitOut | Expand aggregated data back to rows | high error count or been a day1 | Update Last Emailed At | ## Update Errors that were notified; Update Errors that were emailed to they're not picked up next time |
| Update Last Emailed At | n8n-nodes-base.dataTable | Set lastEmailedAt on each row | Split Out | — | ## Update Errors that were notified; Update Errors that were emailed to they're not picked up next time |
| Sticky Note | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ## Automated error monitoring and reporting system (content preserved in Notes section) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ## Generate & Send Error Notification (content preserved in Notes section) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ## Update Errors that were notified (content preserved in Notes section) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ## Filter & Prepare errors to report (content preserved in Notes section) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ## Store errors (content preserved in Notes section) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Data Table**
   1. In n8n, create a Data Table named **Workflow Errors**.
   2. Add columns (types as shown):
      - `workflowId` (string)
      - `executionId` (string)
      - `errorMessage` (string)
      - `errorStack` (string)
      - `lastNodeExecuted` (string)
      - `workflowName` (string)
      - `executionDate` (string)
      - `executionUrl` (string)
      - `lastEmailedAt` (dateTime)

2) **Add the error-ingestion path**
   1. Add node **Error Trigger** (Error Trigger).
   2. Add node **Ignore Manual Failures** (Filter):
      - Condition: String → `{{ $('Error Trigger').item.json.execution.mode }}` **notEquals** `manual`
   3. Add node **map fields to data table** (Set):
      - Add fields with these expressions:
        - `workflowId`: `{{ $('Error Trigger').item.json.workflow.id }}`
        - `executionId`: `{{ $('Error Trigger').item.json.execution.id }}`
        - `errorMessage`: `{{ $('Error Trigger').item.json.execution.error.message ?? JSON.stringify($('Error Trigger').item.json) }}`
        - `errorStack`: `{{ $('Error Trigger').item.json.execution.error.stack }}`
        - `lastNodeExecuted`: `{{ $('Error Trigger').item.json.execution.lastNodeExecuted }}`
        - `workflowName`: `{{ $('Error Trigger').item.json.workflow.name }}`
        - `executionDate`: `{{ $now.format('yyyy-MM-dd') }}`
        - `executionUrl`: `{{ $('Error Trigger').item.json.execution.url }}`
   4. (Optional) Add node **Time Saved** (Time Saved) with `minutesSaved = 1`.
   5. Add node **Insert Error Details** (Data Table):
      - Operation: **Insert**
      - Data Table: **Workflow Errors**
      - Map the same fields as above (or map from the Set node outputs).
   6. Connect: **Error Trigger → Ignore Manual Failures → map fields to data table → Time Saved → Insert Error Details**

3) **Add the hourly reporting path**
   1. Add node **Run every hour** (Schedule Trigger):
      - Interval: every 1 hour
   2. Add node **Get Errors that were not Emailed** (Data Table):
      - Operation: **Get**
      - Filter: `lastEmailedAt` → **isEmpty**
      - Data Table: **Workflow Errors**
   3. Add node **Sort** (Sort):
      - Sort by `createdAt` (ascending)
   4. Add node **Aggregate** (Aggregate):
      - Mode: **Aggregate All Item Data** (so output contains `data` array)
   5. Add node **high error count or been a day1** (Filter):
      - Combinator: **OR**
      - Condition A: Array length `{{ $json.data }}` **lengthGt** `5`
      - Condition B (recommended safer version):  
        `{{ ($json.data?.length ?? 0) > 0 && DateTime.now().diff(DateTime.fromISO($('Sort').first().json.createdAt), 'hours').hours >= 24 }}`
   6. Connect: **Run every hour → Get Errors that were not Emailed → Sort → Aggregate → high error count or been a day1**

4) **Generate the HTML table**
   1. Add node **Generate Workflow Errors Table HTML** (HTML):
      - Paste the table template.
      - Ensure it iterates over `$('Aggregate').item.json.data`.
   2. Connect: **high error count or been a day1 → Generate Workflow Errors Table HTML**

5) **Set up the AI summarizer**
   1. Add node **OpenAI Chat Model** (OpenAI Chat Model / LangChain):
      - Model: `gpt-4.1-mini`
      - Configure **OpenAI API credentials** (API key) in n8n credentials.
   2. Add node **Calculator** (LangChain Tool: Calculator).
   3. Add node **AI Error Summarizer** (LangChain Agent):
      - Prompt text: `Workflow Errors: {{ JSON.stringify($('Aggregate').item.json.data) }}`
      - System message: use the provided system instructions (HTML-only output).
   4. Wire model/tool:
      - Connect **OpenAI Chat Model** to **AI Error Summarizer** via `ai_languageModel`
      - Connect **Calculator** to **AI Error Summarizer** via `ai_tool`
   5. Connect main line: **Generate Workflow Errors Table HTML → AI Error Summarizer**

6) **Email the report**
   1. Add node **Email Error Details** (Gmail → Send):
      - Set Gmail OAuth2 credentials.
      - To: your recipient(s)
      - Subject: `n8n Workflow Errors - {{ $now.minus(1,'days').format('yyyy-MM-dd') }}`
      - Body (fix the HTML table reference):
        - AI summary: `{{ $json.output[0].content[0].text }}`
        - Table: `{{ $('Generate Workflow Errors Table HTML').item.json.html }}`
   2. Connect: **AI Error Summarizer → Email Error Details**

7) **Mark errors as emailed**
   1. Add node **Split Out** (Split Out):
      - Field to split: `data`
   2. Add node **Update Last Emailed At** (Data Table):
      - Operation: **Update**
      - Filter: `id = {{ $json.id }}`
      - Set column: `lastEmailedAt = {{ $now }}`
   3. Connect: **high error count or been a day1 → Split Out → Update Last Emailed At**
   4. (Recommended) To avoid marking as emailed if email fails, instead branch **after** the email node (or use a success path) so updates only run when email succeeds.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automated error monitoring and reporting system… Setup steps: Configure Error Trigger… Set up Gmail… Configure OpenAI API key… Customize data table schema… Adjust schedule…” | Sticky note: Automated error monitoring and reporting system |
| “Generate & Send Error Notification… AI Powered Insights & sends the Email” | Sticky note: Generate & Send Error Notification |
| “Update Errors that were notified… not picked up next time” | Sticky note: Update Errors that were notified |
| “Filter & Prepare errors to report… filter only if error count is beyond a threshold” | Sticky note: Filter & Prepare errors to report |
| “Store errors… Ignore Manual Errors and store automatic errors into our Error Data Table” | Sticky note: Store errors |
| **Known issue to fix:** Email body references `$('Code in JavaScript')` but no such node exists; should reference **Generate Workflow Errors Table HTML**. | Prevents email node expression from resolving |

Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… (content compliant, legal, public data).