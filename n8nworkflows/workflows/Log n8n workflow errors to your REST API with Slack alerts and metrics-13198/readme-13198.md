Log n8n workflow errors to your REST API with Slack alerts and metrics

https://n8nworkflows.xyz/workflows/log-n8n-workflow-errors-to-your-rest-api-with-slack-alerts-and-metrics-13198


# Log n8n workflow errors to your REST API with Slack alerts and metrics

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow listens for failed executions in n8n, extracts useful diagnostic context, classifies and deduplicates errors, logs them to an external REST API, optionally sends Slack alerts based on severity, then records operational metrics and runs a log retention cleanup.

**Target use cases**
- Centralize n8n execution errors into your own datastore/API.
- Reduce alert noise by deduplicating identical errors.
- Route Slack alerts by severity (critical/warning/none).
- Maintain metrics (per-error, aggregates) and enforce retention.

**Logical blocks**
1.1 **Error reception**: Trigger on workflow execution errors.  
1.2 **Metadata & environment enrichment**: Normalize error payload, detect environment.  
1.3 **Error classification & hashing**: Determine severity/type and compute a stable hash for dedupe.  
1.4 **Deduplication check**: Ask REST API whether the error is new.  
1.5 **Log persistence**: Format and save the error log entry to REST API.  
1.6 **Alert routing & Slack notifications**: Switch by severity and send Slack messages.  
1.7 **Metrics & aggregation**: Compute metrics, store them, update aggregates.  
1.8 **Retention cleanup**: Compute retention cutoff and delete old logs via REST API.  
1.9 **Fallback logging (logger failure path)**: Dedicated Slack alert when the logger itself errors (present as nodes, but not wired in this JSON).

---

## 2. Block-by-Block Analysis

### 2.1 Error reception
**Overview:** Captures any execution error occurring in the n8n instance and passes the error payload downstream for processing.  
**Nodes involved:** `Error Trigger`

#### Node: Error Trigger
- **Type / role:** `Error Trigger` — entry point that fires on failed workflow executions.
- **Configuration (interpreted):** Default error trigger behavior (no parameters shown), so it emits standard n8n error-trigger payload (execution, workflow, error, timestamp, etc.).
- **Key variables/fields (typical):** error message/stack, workflow ID/name, execution ID, node that failed.
- **Connections:**  
  - **Output →** `Extract Metadata`
- **Version notes:** TypeVersion `1` (standard).
- **Edge cases / failures:**  
  - May not fire for manual executions depending on instance settings.  
  - Payload shape differs by n8n version; downstream Code nodes must tolerate missing fields.

---

### 2.2 Metadata & environment enrichment
**Overview:** Normalizes raw error-trigger payload into a consistent internal structure, then derives environment/runtime context (e.g., production vs staging, host, instance).  
**Nodes involved:** `Extract Metadata`, `Extract Environment`

#### Node: Extract Metadata
- **Type / role:** `Code` — parse/reshape error-trigger output.
- **Configuration choices:** Not provided in JSON (empty parameters), but intended to extract key fields like workflow name/id, execution id, error message, stack, failed node, timestamps.
- **Connections:**  
  - **Input ←** `Error Trigger`  
  - **Output →** `Extract Environment`
- **Version notes:** Code node TypeVersion `2` (newer runtime; supports `items` API).
- **Edge cases / failures:**  
  - Missing properties in the trigger payload can break naive destructuring.  
  - Large stack traces can exceed downstream API limits if not truncated.

#### Node: Extract Environment
- **Type / role:** `Code` — derive environment metadata.
- **Configuration choices:** Not provided, but commonly uses env vars (e.g., `N8N_HOST`, `N8N_ENV`, custom `ENVIRONMENT`) and/or workflow tags to set `environment`, `instance`, `region`.
- **Connections:**  
  - **Input ←** `Extract Metadata`  
  - **Output →** `Classify Error`
- **Edge cases / failures:**  
  - If environment variables are not defined, should default safely (e.g., “unknown”).  
  - If relying on URL parsing, malformed host values can cause errors.

---

### 2.3 Error classification & hashing
**Overview:** Assigns severity/category to the error and generates a deterministic hash used for deduplication.  
**Nodes involved:** `Classify Error`, `Generate Error Hash`

#### Node: Classify Error
- **Type / role:** `Code` — classify error severity/type.
- **Configuration choices:** Not provided; typically maps known error patterns (HTTP 4xx/5xx, auth errors, timeouts, rate limits) to `severity` (critical/warning/info) and `category`.
- **Connections:**  
  - **Input ←** `Extract Environment`  
  - **Output →** `Generate Error Hash`
- **Edge cases / failures:**  
  - Over-broad regex/pattern matching can misclassify.  
  - Must handle missing stack/message gracefully.

#### Node: Generate Error Hash
- **Type / role:** `Code` — compute stable identifier for an error “signature”.
- **Configuration choices:** Not provided; usually hashes a subset like `{workflowId, nodeName, errorMessageNormalized, errorType}` (often excluding timestamps and volatile IDs).
- **Connections:**  
  - **Input ←** `Classify Error`  
  - **Output →** `API - Check Duplicate`
- **Edge cases / failures:**  
  - If the hash includes volatile fields, dedupe becomes ineffective.  
  - If it excludes too much, distinct errors may collide.

---

### 2.4 Deduplication check (REST API)
**Overview:** Queries your REST API to determine whether this error hash has already been logged recently; branches based on result.  
**Nodes involved:** `API - Check Duplicate`, `Is New Error?`, `Skip Duplicate`

#### Node: API - Check Duplicate
- **Type / role:** `HTTP Request` — call REST endpoint to check existence.
- **Configuration choices:** Not shown (empty parameters). Typically:
  - Method: `GET` (or `POST` with hash payload)
  - URL: your API endpoint (e.g., `/errors/exists?hash=...`)
  - Authentication: header token / basic / OAuth2 depending on your API
- **Connections:**  
  - **Input ←** `Generate Error Hash`  
  - **Output →** `Is New Error?`
- **Version notes:** TypeVersion `4.2` (modern HTTP Request node).
- **Edge cases / failures:**  
  - Auth errors (401/403), DNS failures, timeouts.  
  - Non-JSON responses can break subsequent IF conditions if expected fields are absent.

#### Node: Is New Error?
- **Type / role:** `IF` — branch on duplicate check response.
- **Configuration choices:** Not provided; typically checks something like `exists === false` or `count === 0`.
- **Connections:**  
  - **Input ←** `API - Check Duplicate`  
  - **True/Output 0 →** `Format Log Entry` (new error)  
  - **False/Output 1 →** `Skip Duplicate` (duplicate)
- **Edge cases / failures:**  
  - If API response schema changes, condition may route incorrectly.  
  - If API call failed but returned an error object, condition may mistakenly treat as “new”.

#### Node: Skip Duplicate
- **Type / role:** `No Operation` — explicit sink for duplicates.
- **Connections:**  
  - **Input ←** `Is New Error?` (duplicate branch)
- **Edge cases / failures:** None (acts as terminator).

---

### 2.5 Log persistence (REST API)
**Overview:** Formats the error record into the API’s expected schema and stores it.  
**Nodes involved:** `Format Log Entry`, `API - Save Log`

#### Node: Format Log Entry
- **Type / role:** `Code` — build final log payload.
- **Configuration choices:** Not provided; typically constructs:
  - `hash`, `severity`, `category`
  - workflow/execution identifiers
  - message/stack (possibly truncated)
  - environment fields
  - timestamps
- **Connections:**  
  - **Input ←** `Is New Error?` (new branch)  
  - **Output →** `API - Save Log`
- **Edge cases / failures:**  
  - Payload size limits; stack traces may need truncation/compression.  
  - Date formatting inconsistencies (ISO strings recommended).

#### Node: API - Save Log
- **Type / role:** `HTTP Request` — persist log record.
- **Configuration choices:** Not shown; typically:
  - Method: `POST`
  - URL: `/error-logs`
  - Body: JSON from `Format Log Entry`
- **Connections:**  
  - **Input ←** `Format Log Entry`  
  - **Output →** `Alert Router`
- **Edge cases / failures:**  
  - API rejects schema (400) or rate limits (429).  
  - If API is down, alerts/metrics won’t run (unless separately error-handled).

---

### 2.6 Alert routing & Slack notifications
**Overview:** Decides whether to send Slack alerts and which channel/message format to use based on severity.  
**Nodes involved:** `Alert Router`, `Format Critical Alert`, `Slack - Critical Alert`, `Format Warning Alert`, `Slack - Warning Alert`, `No Alert Needed`

#### Node: Alert Router
- **Type / role:** `Switch` — routes by severity/category.
- **Configuration choices:** Not provided; based on connections, it has **3 outputs**:
  1) to critical formatting  
  2) to warning formatting  
  3) to no-op
- **Connections:**  
  - **Input ←** `API - Save Log`  
  - **Output(critical) →** `Format Critical Alert`  
  - **Output(warning) →** `Format Warning Alert`  
  - **Output(none) →** `No Alert Needed`
- **Edge cases / failures:**  
  - If severity field missing/unknown, may route to default output (ensure switch has a default).

#### Node: Format Critical Alert
- **Type / role:** `Code` — create Slack message payload for critical.
- **Connections:**  
  - **Input ←** `Alert Router` (critical branch)  
  - **Output →** `Slack - Critical Alert`
- **Edge cases / failures:**  
  - Slack formatting issues (invalid blocks/attachments if used).  
  - Must ensure message includes key links (execution URL) if desired.

#### Node: Slack - Critical Alert
- **Type / role:** `Slack` — send critical alert message.
- **Configuration choices:** Uses Slack node (TypeVersion `2.2`). The JSON includes a `webhookId`, implying a Slack credential/webhook is configured in the instance.
- **Connections:**  
  - **Input ←** `Format Critical Alert`  
  - **Output →** `Calculate Metrics`
- **Edge cases / failures:**  
  - Invalid/rotated webhook, workspace permission issues, Slack rate limits.

#### Node: Format Warning Alert
- **Type / role:** `Code` — create Slack message payload for warning.
- **Connections:**  
  - **Input ←** `Alert Router` (warning branch)  
  - **Output →** `Slack - Warning Alert`

#### Node: Slack - Warning Alert
- **Type / role:** `Slack` — send warning alert.
- **Connections:**  
  - **Input ←** `Format Warning Alert`  
  - **Output →** `Calculate Metrics`

#### Node: No Alert Needed
- **Type / role:** `No Operation` — continues pipeline without sending Slack.
- **Connections:**  
  - **Input ←** `Alert Router`  
  - **Output →** `Calculate Metrics`

---

### 2.7 Metrics & aggregation (REST API)
**Overview:** Computes metrics about the error event, stores a metrics record, then updates aggregate counters (e.g., per hash/day/environment).  
**Nodes involved:** `Calculate Metrics`, `API - Save Metrics`, `Prepare Aggregate`, `API - Update Aggregate`

#### Node: Calculate Metrics
- **Type / role:** `Code` — compute metric fields.
- **Configuration choices:** Not shown; likely derives:
  - counts, severity score, durations (if available), environment, workflow id
  - time bucketing (hour/day)
- **Connections:**  
  - **Input ←** `Slack - Critical Alert` OR `Slack - Warning Alert` OR `No Alert Needed`  
  - **Output →** `API - Save Metrics`
- **Edge cases / failures:**  
  - Metrics may require fields only present in successful API log response; ensure robustness.

#### Node: API - Save Metrics
- **Type / role:** `HTTP Request` — persist per-event metric record.
- **Connections:**  
  - **Input ←** `Calculate Metrics`  
  - **Output →** `Prepare Aggregate`
- **Edge cases / failures:** Same as other API calls (timeouts/auth/schema).

#### Node: Prepare Aggregate
- **Type / role:** `Code` — build payload for aggregate update.
- **Configuration choices:** Not shown; likely prepares keys like `{hash, dateBucket, environment}` and increment operations.
- **Connections:**  
  - **Input ←** `API - Save Metrics`  
  - **Output →** `API - Update Aggregate`

#### Node: API - Update Aggregate
- **Type / role:** `HTTP Request` — update rollups/aggregates.
- **Connections:**  
  - **Input ←** `Prepare Aggregate`  
  - **Output →** `Calculate Retention`
- **Edge cases / failures:**  
  - Idempotency: ensure aggregate updates are safe if n8n retries.

---

### 2.8 Retention cleanup
**Overview:** Determines a retention cutoff timestamp and calls the API to purge old logs.  
**Nodes involved:** `Calculate Retention`, `API - Cleanup Old Logs`

#### Node: Calculate Retention
- **Type / role:** `Code` — compute retention thresholds.
- **Configuration choices:** Not shown; typically uses a constant like `RETENTION_DAYS` and computes `cutoff = now - days`.
- **Connections:**  
  - **Input ←** `API - Update Aggregate`  
  - **Output →** `API - Cleanup Old Logs`
- **Edge cases / failures:**  
  - Timezone mismatch between n8n and API storage.  
  - Aggressive retention can delete needed diagnostics.

#### Node: API - Cleanup Old Logs
- **Type / role:** `HTTP Request` — delete/purge old records.
- **Configuration choices:** Not shown; typically `DELETE /error-logs?before=cutoff`.
- **Connections:**  
  - **Input ←** `Calculate Retention`
- **Edge cases / failures:**  
  - Dangerous endpoint if cutoff is wrong; ensure server-side validation.

---

### 2.9 Fallback logging (logger failure path)
**Overview:** Provides a Slack notification path intended to alert when this error-logging workflow itself fails. In the provided JSON, these nodes are **not connected** to the main flow or to an Error Trigger dedicated to this workflow.  
**Nodes involved:** `Format Logger Error`, `Slack - Logger Error`

#### Node: Format Logger Error
- **Type / role:** `Code` — format an internal “logger failed” message.
- **Connections:**  
  - **Output →** `Slack - Logger Error`  
  - **Input:** Not connected in this workflow JSON.
- **Edge cases / failures:**  
  - Without an input connection, it will never run unless manually executed or wired to a trigger.

#### Node: Slack - Logger Error
- **Type / role:** `Slack` — send logger failure alert.
- **Connections:**  
  - **Input ←** `Format Logger Error`  
  - **Output:** none
- **Edge cases / failures:** Same Slack webhook/auth/rate-limit concerns.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Canvas annotation | — | — |  |
| Error capture | Sticky Note | Canvas annotation | — | — |  |
| Error Trigger | Error Trigger | Entry point for failed executions | — | Extract Metadata |  |
| Extract Metadata | Code | Normalize/shape error payload | Error Trigger | Extract Environment |  |
| Extract Environment | Code | Add environment/runtime context | Extract Metadata | Classify Error |  |
| Error classification | Sticky Note | Canvas annotation | — | — |  |
| Classify Error | Code | Determine severity/category | Extract Environment | Generate Error Hash |  |
| Generate Error Hash | Code | Stable signature for dedupe | Classify Error | API - Check Duplicate |  |
| API - Check Duplicate | HTTP Request | Query API for duplicate | Generate Error Hash | Is New Error? |  |
| Is New Error? | IF | Branch new vs duplicate | API - Check Duplicate | Format Log Entry; Skip Duplicate |  |
| Log storage | Sticky Note | Canvas annotation | — | — |  |
| Format Log Entry | Code | Build API log payload | Is New Error? | API - Save Log |  |
| API - Save Log | HTTP Request | Persist error log | Format Log Entry | Alert Router |  |
| Alert routing | Sticky Note | Canvas annotation | — | — |  |
| Alert Router | Switch | Route alerts by severity | API - Save Log | Format Critical Alert; Format Warning Alert; No Alert Needed |  |
| Critical alerts | Sticky Note | Canvas annotation | — | — |  |
| Format Critical Alert | Code | Build critical Slack message | Alert Router | Slack - Critical Alert |  |
| Slack - Critical Alert | Slack | Send critical Slack alert | Format Critical Alert | Calculate Metrics |  |
| Format Warning Alert | Code | Build warning Slack message | Alert Router | Slack - Warning Alert |  |
| Slack - Warning Alert | Slack | Send warning Slack alert | Format Warning Alert | Calculate Metrics |  |
| No Alert Needed | No Operation | Continue without alert | Alert Router | Calculate Metrics |  |
| Performance metrics | Sticky Note | Canvas annotation | — | — |  |
| Calculate Metrics | Code | Compute metrics fields | Slack - Critical Alert; Slack - Warning Alert; No Alert Needed | API - Save Metrics |  |
| API - Save Metrics | HTTP Request | Persist metrics | Calculate Metrics | Prepare Aggregate |  |
| Prepare Aggregate | Code | Build aggregate update payload | API - Save Metrics | API - Update Aggregate |  |
| API - Update Aggregate | HTTP Request | Update rollup counters | Prepare Aggregate | Calculate Retention |  |
| Retention policy | Sticky Note | Canvas annotation | — | — |  |
| Calculate Retention | Code | Compute retention cutoff | API - Update Aggregate | API - Cleanup Old Logs |  |
| API - Cleanup Old Logs | HTTP Request | Purge old logs | Calculate Retention | — |  |
| Fallback logging | Sticky Note | Canvas annotation | — | — |  |
| Format Logger Error | Code | Format logger-failure alert | — | Slack - Logger Error |  |
| Slack - Logger Error | Slack | Send logger-failure Slack alert | Format Logger Error | — |  |
| Skip Duplicate | No Operation | End execution for duplicates | Is New Error? | — |  |

*Note:* All sticky notes in the provided JSON have **empty content**, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: **“Log n8n workflow errors to your REST API with Slack alerts and metrics”**.

2) **Add the error entry point**
- Add node: **Error Trigger**
- Leave defaults.

3) **Add metadata extraction**
- Add node: **Code** → name: `Extract Metadata`
- Implement code to map Error Trigger output into a normalized object, for example:
  - `workflowId`, `workflowName`
  - `executionId`, `mode`
  - `errorMessage`, `errorStack`, `failingNode`
  - `timestamp` (ISO)
- Connect: `Error Trigger` → `Extract Metadata`.

4) **Add environment enrichment**
- Add node: **Code** → name: `Extract Environment`
- Read environment variables (recommended) such as:
  - `ENVIRONMENT` (prod/staging/dev)
  - `N8N_HOST` or a custom `INSTANCE_NAME`
- Append fields like `environment`, `instance`.
- Connect: `Extract Metadata` → `Extract Environment`.

5) **Add classification**
- Add node: **Code** → name: `Classify Error`
- Produce fields like:
  - `severity`: `critical` / `warning` / `none`
  - `category`: `auth` / `timeout` / `rate_limit` / `unknown`
- Connect: `Extract Environment` → `Classify Error`.

6) **Add error hashing**
- Add node: **Code** → name: `Generate Error Hash`
- Compute deterministic hash from stable fields (avoid timestamps/executionId). Example inputs:
  - workflowId, failingNode, normalized error message/type
- Connect: `Classify Error` → `Generate Error Hash`.

7) **Add duplicate check via REST API**
- Add node: **HTTP Request** → name: `API - Check Duplicate`
- Configure:
  - Method: `GET` (or `POST`, depending on your API)
  - URL: your endpoint (e.g., `https://api.example.com/error-logs/exists`)
  - Send hash as query/body (e.g., `hash={{$json.hash}}`)
  - Auth: configure (Header token / Basic / OAuth2) in **Credentials**
- Connect: `Generate Error Hash` → `API - Check Duplicate`.

8) **Branch on duplicate**
- Add node: **IF** → name: `Is New Error?`
- Condition: based on API response (examples)
  - `{{$json.exists}}` is `false`, or `{{$json.count}} == 0`
- Connect: `API - Check Duplicate` → `Is New Error?`

9) **Duplicate sink**
- Add node: **No Operation** → name: `Skip Duplicate`
- Connect IF **false** output → `Skip Duplicate`.

10) **Format log entry**
- Add node: **Code** → name: `Format Log Entry`
- Build the JSON body expected by your “save log” endpoint, including:
  - hash, severity/category, environment, workflow/execution info, message/stack, timestamps
- Connect IF **true** output → `Format Log Entry`.

11) **Save log to API**
- Add node: **HTTP Request** → name: `API - Save Log`
- Configure:
  - Method: `POST`
  - URL: e.g., `https://api.example.com/error-logs`
  - Body: JSON from `Format Log Entry`
  - Auth credentials as needed
- Connect: `Format Log Entry` → `API - Save Log`.

12) **Route alerts**
- Add node: **Switch** → name: `Alert Router`
- Configure rules on `severity` (or your chosen field):
  - Output 1: `critical`
  - Output 2: `warning`
  - Output 3: default / otherwise (no alert)
- Connect: `API - Save Log` → `Alert Router`.

13) **Critical alert path**
- Add node: **Code** → `Format Critical Alert` (create Slack text/blocks)
- Add node: **Slack** → `Slack - Critical Alert`
  - Configure Slack credentials (Webhook or OAuth depending on node operation).
  - Select channel / webhook target, set message from previous node.
- Connect: `Alert Router (critical)` → `Format Critical Alert` → `Slack - Critical Alert`.

14) **Warning alert path**
- Add node: **Code** → `Format Warning Alert`
- Add node: **Slack** → `Slack - Warning Alert`
- Connect: `Alert Router (warning)` → `Format Warning Alert` → `Slack - Warning Alert`.

15) **No-alert path**
- Add node: **No Operation** → `No Alert Needed`
- Connect: `Alert Router (default)` → `No Alert Needed`.

16) **Metrics computation**
- Add node: **Code** → `Calculate Metrics`
- Connect all three branches to it:
  - `Slack - Critical Alert` → `Calculate Metrics`
  - `Slack - Warning Alert` → `Calculate Metrics`
  - `No Alert Needed` → `Calculate Metrics`

17) **Save metrics**
- Add node: **HTTP Request** → `API - Save Metrics` (POST to metrics endpoint)
- Connect: `Calculate Metrics` → `API - Save Metrics`.

18) **Prepare and update aggregates**
- Add node: **Code** → `Prepare Aggregate` (build rollup update payload)
- Add node: **HTTP Request** → `API - Update Aggregate` (POST/PATCH to aggregates endpoint)
- Connect: `API - Save Metrics` → `Prepare Aggregate` → `API - Update Aggregate`.

19) **Retention**
- Add node: **Code** → `Calculate Retention` (compute `cutoff` from retention days)
- Add node: **HTTP Request** → `API - Cleanup Old Logs` (DELETE or POST cleanup)
- Connect: `API - Update Aggregate` → `Calculate Retention` → `API - Cleanup Old Logs`.

20) **(Optional but recommended) Fallback logging**
- Add a second **Error Trigger** dedicated to this logger workflow, or enable workflow-level error handling so failures here are captured.
- Add node: **Code** → `Format Logger Error`
- Add node: **Slack** → `Slack - Logger Error`
- Wire the logger’s own error trigger to `Format Logger Error` → `Slack - Logger Error`.
- This ensures you get alerted if the REST API/Slack steps break.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided workflow JSON. | Canvas annotations are present as placeholders: Overview, Error capture, Error classification, Log storage, Alert routing, Critical alerts, Performance metrics, Retention policy, Fallback logging. |
| Several nodes (HTTP Request, Code, IF, Switch) have empty parameter objects in the JSON, so exact field mappings, endpoints, and conditions must be defined to match your REST API contract and Slack message format. | Applies across the workflow. |
| Fallback logging nodes are not connected; to make them effective, wire them to an error path for the logger workflow itself. | Ensures “logger failure” alerts are emitted. |