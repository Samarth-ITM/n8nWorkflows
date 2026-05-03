Predict incidents and run autonomous remediation with GPT-4 and Slack

https://n8nworkflows.xyz/workflows/predict-incidents-and-run-autonomous-remediation-with-gpt-4-and-slack-12686


# Predict incidents and run autonomous remediation with GPT-4 and Slack

## 1. Workflow Overview

**Title (provided):** Predict incidents and run autonomous remediation with GPT-4 and Slack  
**Workflow name (JSON):** Predictive AI Ops Incident Prevention and Autonomous Remediation System

**Purpose:**  
This workflow periodically pulls observability signals (metrics, logs, traces, alerts), detects anomalies, forecasts near-future incidents, performs causal inference to estimate root cause/service propagation, then uses GPT-based agents to (a) correlate root cause, (b) assess blast radius and decide if autonomous remediation is safe, and (c) either execute remediation automatically or request manual approval via Slack. Finally, it stores incident context to Postgres + PGVector, generates a post-mortem, updates a learning database, and emails the post-mortem.

**Primary use cases:**
- Predictive incident prevention (early warning via anomaly + forecasting)
- Automated or human-approved remediation
- Consistent escalation packets for on-call (Slack) with AI diagnostics
- Incident knowledge base + vector search memory for continuous learning

### 1.1 Trigger & Configuration
Runs every 2 minutes, initializes API endpoints and policy thresholds.

### 1.2 Observability Data Acquisition & Normalization
Fetches metrics/logs/traces/alerts via HTTP and merges into a single combined stream.

### 1.3 Detection & Prediction (Python analytics)
Computes anomaly score, forecasts incident likelihood and time-to-impact, then performs causal inference on service interactions.

### 1.4 AI Correlation & Impact Assessment (LLM agents)
If any detector indicates risk, GPT agents produce structured root-cause correlation and blast-radius assessment.

### 1.5 Remediation Decisioning
If safe and enabled, plans remediation with GPT agent and executes it; otherwise requests approval via Slack and waits for webhook response.

### 1.6 Verification, Rollback, Escalation, and Persistence/Learning
Verifies remediation success, rolls back if needed, escalates with diagnostics if human intervention is required, stores data (Postgres + PGVector), generates and emails post-mortem, updates learning DB.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Runtime Configuration
**Overview:** Starts the workflow on a timer and sets all environment-specific endpoints and policy flags used by later nodes.

**Nodes involved:**
- Metrics Collection Schedule
- Workflow Configuration

#### Metrics Collection Schedule
- **Type/role:** `Schedule Trigger` — periodic workflow entrypoint.
- **Config:** Every **2 minutes**.
- **Inputs/outputs:** Entry node → outputs to **Workflow Configuration**.
- **Edge cases/failures:** n8n instance downtime delays runs; overlapping executions possible if a run lasts >2 min (consider concurrency settings).

#### Workflow Configuration
- **Type/role:** `Set` — centralizes constants (URLs, thresholds, flags).
- **Config choices:**
  - Defines placeholders for: `metricsApiUrl`, `logsApiUrl`, `tracesApiUrl`, `alertsApiUrl`, `remediationApiUrl`
  - Policy values: `anomalyThreshold (0.85)`, `forecastHorizonMinutes (30)`, `blastRadiusThreshold (0.7)`, `autoRemediationEnabled (true)`, `maxWaitTimeSeconds (300)`
  - **includeOtherFields=true** (keeps incoming fields, though upstream trigger has none)
- **Key expressions/variables:** Later nodes reference it like `$('Workflow Configuration').first().json.metricsApiUrl`.
- **Outputs:** Fans out to the 4 HTTP fetch nodes.
- **Edge cases:** Placeholder URLs/tokens must be replaced; any missing property breaks dependent expressions.

---

### Block 2 — Observability Fetching & Merge
**Overview:** Pulls data from four sources and merges them into a single combined stream for analysis.

**Nodes involved:**
- Fetch Metrics Data
- Fetch Logs Data
- Fetch Traces Data
- Fetch Alerts Data
- Merge Observability Data

#### Fetch Metrics Data
- **Type/role:** `HTTP Request` — pulls metrics.
- **Config:** URL from configuration: `metricsApiUrl`. Adds `Authorization: Bearer ...` (placeholder).
- **Inputs/outputs:** From **Workflow Configuration** → to **Merge Observability Data** (input 0).
- **Failure types:** 401/403 auth; DNS/timeout; non-JSON responses; rate limiting.

#### Fetch Logs Data
- **Type/role:** `HTTP Request` — pulls logs.
- **Config:** URL: `logsApiUrl`, bearer auth placeholder.
- **Outputs:** → Merge input 1.
- **Failures:** same as above; large payload risk (memory).

#### Fetch Traces Data
- **Type/role:** `HTTP Request` — pulls traces.
- **Config:** URL: `tracesApiUrl`, bearer auth placeholder.
- **Outputs:** → Merge input 2.
- **Failures:** same; trace volume can be large.

#### Fetch Alerts Data
- **Type/role:** `HTTP Request` — pulls alert events.
- **Config:** URL: `alertsApiUrl`, bearer auth placeholder.
- **Outputs:** → Merge input 3.
- **Failures:** same.

#### Merge Observability Data
- **Type/role:** `Merge` — combines 4 inputs.
- **Config choices:**
  - Mode: **combine**
  - Combine by: **position**
  - Inputs: **4**
- **Inputs/outputs:** Receives the 4 HTTP outputs → to **Anomaly Detection Engine**.
- **Edge cases:** “combineByPosition” assumes each source returns compatible item counts/order. If APIs return different lengths, merged rows may misalign signals (common source of incorrect correlation). Consider merging by key/timestamp instead.

---

### Block 3 — Anomaly Detection (Python)
**Overview:** Computes anomaly scores across metrics/logs/traces/alerts and emits a consolidated anomaly verdict.

**Nodes involved:**
- Anomaly Detection Engine

#### Anomaly Detection Engine
- **Type/role:** `Code` (Python) — statistical anomaly detection and scoring.
- **Config choices / logic highlights:**
  - Reads all merged items: `_items('all')`
  - Categorizes by `item.json.type` in `{metrics, logs, traces, alerts}`.
  - Metrics: z-score + IQR + simplified isolation-like score (median absolute deviation), averaged.
  - Logs: error rate-based score (errors *10 normalized to 1.0).
  - Traces: z-score + IQR on durations.
  - Alerts: critical/high count normalized (5+ → 1.0).
  - Weighted combination: metrics 0.3, logs 0.25, traces 0.25, alerts 0.2.
  - **Hard-coded** anomaly threshold: `combined_score > 0.5` (note: does *not* use `anomalyThreshold` from config).
  - Extracts `affectedServices` from `item.json.service` values.
- **Inputs/outputs:** From **Merge Observability Data** → to **Temporal Forecasting Model**.
- **Version-specific notes:** Python Code node availability depends on n8n build/plan; SciPy usage (`from scipy import stats`) requires that the runtime image includes SciPy. Many n8n environments do not ship SciPy by default.
- **Edge cases/failures:**
  - If upstream data doesn’t include `type`, everything is ignored and scores stay 0.
  - If metrics values are non-numeric, NumPy operations can error.
  - `scipy` import can fail; `stats` is imported but not used (still can break execution).
  - Uses `_items('all')` (nonstandard in some n8n code runtimes); in many contexts you’d use `_input.all()`—confirm your n8n Python API.

---

### Block 4 — Forecasting & Causal Inference (Python)
**Overview:** Forecasts incident likelihood/time-to-impact and infers causal relationships across services.

**Nodes involved:**
- Temporal Forecasting Model
- Causal Inference Analysis
- Incident Predicted?

#### Temporal Forecasting Model
- **Type/role:** `Code` (Python) — simplified incident forecasting.
- **Config/logic highlights:**
  - Uses `_input.all()` and expects the anomaly result in the first item.
  - Forecasted incident condition: `anomaly_score > 0.6 and anomaly_count > 2`
  - Builds `predictedMetrics` (cpu/memory/errors/latency) from `anomalies.get('metrics', {})...`
- **Critical data mismatch (likely bug):**
  - The prior node outputs keys like `anomalyDetected`, `anomalyScore`, `anomalyDetails`.  
  - This node expects `anomalyCount`, `detectedAnomalies`, and `metrics.cpu/memory/errors/latency`, which are not produced upstream. Result: `anomaly_count` defaults to 0, `forecastedIncident` tends to **False**, predicted metrics use defaults.
- **Inputs/outputs:** From **Anomaly Detection Engine** → to **Causal Inference Analysis**.
- **Edge cases:** If no input, it returns a safe “no data” object.

#### Causal Inference Analysis
- **Type/role:** `Code` (Python) — service dependency causal scoring.
- **Config/logic highlights:**
  - Attempts to extract `metrics/logs/traces/alerts` only if `'anomalies' in item.json`. That key is never set by upstream nodes, so it likely never loads observability arrays.
  - Builds service graph primarily from `traces_data` dict fields `service`, `parent_service`, `duration`, `latency`, `error`.
  - Computes simplified Granger and transfer-entropy approximations via lagged correlation.
  - Picks root cause as service with highest outgoing causal influence weighted by error count.
  - Outputs: `causalChainDetected`, `rootCauseService`, `causalPath`, `causalStrength`, `inferenceDetails`.
- **Inputs/outputs:** From **Temporal Forecasting Model** → to **Incident Predicted?**
- **Edge cases/failures:**
  - If trace fields aren’t present or input extraction fails, service_graph stays empty → root cause becomes `unknown`, causalChainDetected false.
  - SciPy imported but unused; can still break if missing.
  - Correlation on constant series may yield NaN; not handled.

#### Incident Predicted?
- **Type/role:** `IF` — gating node to proceed only if risk detected.
- **Condition (OR):**
  - `$json.anomalyDetected == true` OR
  - `$json.forecastedIncident == true` OR
  - `$json.causalChainDetected == true`
- **Inputs/outputs:** From **Causal Inference Analysis** → true path goes to **Root Cause Correlation Agent**. (No false-path handling is defined; workflow effectively stops if false.)
- **Edge cases:** If upstream outputs don’t include these booleans, they evaluate falsy and stop processing.

---

### Block 5 — Root Cause Correlation (LLM agent with structured output)
**Overview:** Uses GPT to synthesize anomaly/forecast/causal data into a structured root cause analysis.

**Nodes involved:**
- Root Cause Correlation Agent
- OpenAI GPT-4
- Root Cause Output Schema

#### Root Cause Correlation Agent
- **Type/role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM call with system prompt + structured parsing.
- **Config choices:**
  - Input text: `JSON.stringify($json)` (entire current context)
  - System message defines tasks and demands JSON per schema.
  - `hasOutputParser=true` with the schema node.
- **Connections:**
  - Uses **OpenAI GPT-4** as `ai_languageModel`.
  - Uses **Root Cause Output Schema** as `ai_outputParser`.
  - Main output → **Blast Radius Evaluator**.
- **Failure types:** OpenAI auth/model access; schema mismatch (parser errors); context too large; latency/timeouts.

#### OpenAI GPT-4
- **Type/role:** `LM Chat OpenAI`
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API credential referenced.
- **Edge cases:** model availability differs by account/region; rate limits.

#### Root Cause Output Schema
- **Type/role:** `Structured Output Parser`
- **Schema fields:** `rootCauseService`, `rootCauseProbability (0..1)`, `contributingFactors[]`, `failurePropagationPath[]`, `correlatedSignals (object)`, `confidenceLevel (0..1)`, `analysisTimestamp`.
- **Failure types:** LLM returns non-conforming JSON → node errors.

---

### Block 6 — Blast Radius Assessment (LLM agent with structured output)
**Overview:** GPT agent estimates impact scope and decides whether auto-remediation is safe.

**Nodes involved:**
- Blast Radius Evaluator
- OpenAI GPT-4 Blast Radius
- Blast Radius Output Schema
- Auto-Remediation Safe?

#### Blast Radius Evaluator
- **Type/role:** LangChain Agent — business/impact assessment.
- **Config:** Input `JSON.stringify($json)`; system message requests structured JSON.
- **Connections:** uses **OpenAI GPT-4 Blast Radius** + **Blast Radius Output Schema**; main output → **Auto-Remediation Safe?**
- **Failure types:** same as other agent nodes.

#### OpenAI GPT-4 Blast Radius
- **Type/role:** LM Chat OpenAI
- **Model:** `gpt-4.1-mini`
- **Credentials:** OpenAI API.
- **Edge cases:** rate limits, latency.

#### Blast Radius Output Schema
- **Fields:** `affectedServices[]`, `affectedUsers`, `estimatedRevenueLoss`, `slaViolationRisk (0..1)`, `blastRadiusScore (0..1)`, `downstreamImpact[]`, `autoRemediationSafe (bool)`, `requiresApproval (bool)`, `riskAssessment (string)`.

#### Auto-Remediation Safe?
- **Type/role:** `IF` — policy gate to auto-remediate vs manual approval.
- **Condition (AND):**
  - `$json.autoRemediationSafe == true`
  - `$json.requiresApproval == false`
  - `$('Workflow Configuration').first().json.autoRemediationEnabled == true`
- **Outputs:**
  - True path → **Remediation Strategy Planner**
  - False path → **Request Manual Approval**
- **Edge cases:** If blast radius agent output lacks booleans, condition fails → manual approval path.

---

### Block 7 — Autonomous Remediation Planning & Execution
**Overview:** Plans remediation steps with GPT, executes the first action, waits, verifies success, and rolls back on failure.

**Nodes involved:**
- Remediation Strategy Planner
- OpenAI GPT-4 Remediation
- Remediation Plan Output Schema
- Execute Remediation Action
- Wait for Remediation Effect
- Verify Remediation Success
- Remediation Successful?
- Execute Rollback

#### Remediation Strategy Planner
- **Type/role:** LangChain Agent — produces a structured remediation plan.
- **Config:** Input `JSON.stringify($json)`; system message asks for actions, safety constraints, validation, rollback.
- **Connections:** Uses **OpenAI GPT-4 Remediation** + **Remediation Plan Output Schema**; main output → **Execute Remediation Action**.
- **Edge cases:** If plan omits expected fields, downstream expressions break.

#### OpenAI GPT-4 Remediation
- **Type/role:** LM Chat OpenAI
- **Model:** `gpt-4.1-mini`

#### Remediation Plan Output Schema
- **Fields:** `remediationActions[]` (each has `action`, `endpoint`, `method`, `payload`), plus `safetyConstraints`, `validationCriteria`, `rollbackProcedure`, `estimatedDurationSeconds`, `successProbability (0..1)`.

#### Execute Remediation Action
- **Type/role:** `HTTP Request` — executes first remediation action only.
- **Config choices:**
  - URL: `{{$json.remediationActions[0].endpoint}}`
  - Method: `{{$json.remediationActions[0].method}}`
  - Body: raw JSON stringified from payload
  - Headers: `Content-Type: application/json` and placeholder `Authorization`
- **Inputs/outputs:** From planner → to **Wait for Remediation Effect**
- **Edge cases/failures:**
  - If `remediationActions` is empty, `[0]` expression fails.
  - If endpoint is not absolute URL or blocked by allowlist, request fails.
  - No looping over multiple actions; only first action is executed.

#### Wait for Remediation Effect
- **Type/role:** `Wait`
- **Config:** amount from config `maxWaitTimeSeconds` (default 300).
- **Inputs/outputs:** → **Verify Remediation Success**
- **Edge cases:** Wait node holds execution state; high volume can pressure n8n execution storage.

#### Verify Remediation Success
- **Type/role:** `HTTP Request` — re-fetches metrics to validate outcome.
- **Config:** Calls `metricsApiUrl` with placeholder auth header.
- **Outputs:** → **Remediation Successful?**
- **Edge cases:** This node assumes the response includes `validationCriteriaMet`; not enforced.

#### Remediation Successful?
- **Type/role:** `IF`
- **Condition:** `$('Verify Remediation Success').item.json.validationCriteriaMet == true`
- **Outputs:**
  - True → **Merge Remediation Paths** (input 0)
  - False → **Execute Rollback** (then merge input 1)
- **Edge cases:** If verification response shape differs, expression resolves undefined and fails condition.

#### Execute Rollback
- **Type/role:** `HTTP Request`
- **Config:** URL/payload pulled from **Remediation Strategy Planner**:
  - URL: `$('Remediation Strategy Planner').item.json.rollbackEndpoint`
  - Body: `rollbackPayload`
- **Critical mismatch (likely bug):**
  - The remediation schema defines `rollbackProcedure` but not `rollbackEndpoint/rollbackPayload`. Unless the agent outputs extra fields, this request will fail.
- **Outputs:** → **Merge Remediation Paths** (input 1)

---

### Block 8 — Manual Approval & Escalation Path
**Overview:** If not safe to auto-remediate, requests approval in Slack and waits for a webhook response; if approved, proceeds with remediation, otherwise escalates with AI-generated diagnostics.

**Nodes involved:**
- Request Manual Approval
- Approval Response Webhook
- Approval Granted?
- Escalation Diagnostics Generator
- OpenAI GPT-4 Diagnostics
- Diagnostics Output Schema
- Escalate to On-Call Team

#### Request Manual Approval
- **Type/role:** `Slack` — posts approval request.
- **Config:** OAuth2 auth; sends a formatted message to a configured channel ID (placeholder).
- **Message includes:** references `$('Approval Response Webhook').item.json.webhookUrl` (but webhook node does not produce `webhookUrl` by default).
- **Edge cases/bugs:**
  - Slack node text contains fields like `$json.incidentSummary`, `$json.blastRadius`, `$json.proposedRemediation` which are not produced by upstream blast radius output schema unless added.
  - Approval webhook URL is not automatically available as `webhookUrl`. You usually must hardcode it or construct from n8n base URL + path.

#### Approval Response Webhook
- **Type/role:** `Webhook` — second entrypoint for approvals.
- **Config:** POST `/approval-response`, `responseMode=lastNode`.
- **Outputs:** → **Approval Granted?**
- **Edge cases:** Must be publicly reachable or behind appropriate network; validate signature/auth to prevent unauthorized approvals.

#### Approval Granted?
- **Type/role:** `IF`
- **Condition:** `$json.body.approved == true`
- **Outputs:**
  - True → **Remediation Strategy Planner**
  - False → **Escalation Diagnostics Generator**
- **Edge cases:** Payload must be JSON with `{"approved": true}` in body; otherwise false.

#### Escalation Diagnostics Generator
- **Type/role:** LangChain Agent — creates structured escalation packet.
- **Connections:** uses **OpenAI GPT-4 Diagnostics** + **Diagnostics Output Schema**; output → **Escalate to On-Call Team**
- **Edge cases:** schema mismatch; large context.

#### OpenAI GPT-4 Diagnostics / Diagnostics Output Schema
- **Model:** `gpt-4.1-mini`
- **Schema fields:** `incidentSummary`, `severity`, `rootCauseAnalysis`, `blastRadiusDetails`, `recommendedActions[]`, `troubleshootingSteps[]`, `relevantLogs[]`, `relevantMetrics`, `escalationReason`.

#### Escalate to On-Call Team
- **Type/role:** `Slack` — posts escalation message.
- **Config:** OAuth2; sends to on-call channel (placeholder).
- **Edge cases/bugs:** Message template references fields like `$json.incident_id`, `$json.detected_at`, `$json.root_cause`, `$json.blast_radius.*`, `$json.diagnostics`, etc. These do not match the diagnostics schema (camelCase vs snake_case), so message may render blanks unless upstream data is transformed.

---

### Block 9 — Merge Outcomes, Store, Embed, Post-Mortem, Learning, Email
**Overview:** Consolidates remediation outcomes (success/rollback/escalation), stores incident records, inserts vector embeddings, generates post-mortem, updates learning DB, and emails the report.

**Nodes involved:**
- Merge Remediation Paths
- Store Incident in Knowledge Graph
- Incident Vector Store
- OpenAI Embeddings
- Incident Document Loader
- Post-Mortem Generator
- OpenAI GPT-4 Post-Mortem
- Post-Mortem Output Schema
- Update Learning Database
- Send Post-Mortem Report

#### Merge Remediation Paths
- **Type/role:** `Merge`
- **Config:** `numberInputs=3` (success, rollback, escalation).
- **Inputs:**
  - Input 0 from **Remediation Successful?** true path
  - Input 1 from **Execute Rollback**
  - Input 2 from **Escalate to On-Call Team**
- **Outputs:** → **Store Incident in Knowledge Graph** and → **Incident Vector Store**
- **Edge cases:** Merge mode not specified (defaults can be confusing). If some paths don’t execute, merge behavior may stall or output partials depending on n8n merge semantics.

#### Store Incident in Knowledge Graph
- **Type/role:** `Postgres` — inserts/updates incident row.
- **Config choices:**
  - Table: `public.incidents`
  - Matching column: `incident_id`
  - Writes: `timestamp, incident_id, incident_data, resolution_time, blast_radius_score, remediation_status, root_cause_service`
- **Edge cases/bugs:** Upstream data likely does not provide these exact keys; expressions like `{{$json.incident_id}}` vs `incidentId` mismatch. Will insert nulls unless mapped beforehand.
- **Requirements:** Postgres credentials must be configured in n8n (not shown in JSON).

#### Incident Vector Store
- **Type/role:** `PGVector Vector Store` — inserts embeddings.
- **Config:** mode insert; collection name `incidents`; table `incident_embeddings`.
- **Connections:** Receives embeddings from **OpenAI Embeddings** and documents from **Incident Document Loader**, and then outputs to **Post-Mortem Generator**.
- **Requirements:** Postgres with pgvector extension; correct table schema and permissions.

#### Incident Document Loader
- **Type/role:** `Document Loader` — converts current JSON into documents for embedding.
- **Config:** default loader; no explicit mapping shown.
- **Edge cases:** Ensure it actually ingests meaningful text (incident summary, actions, etc.), otherwise embeddings are low value.

#### OpenAI Embeddings
- **Type/role:** OpenAI embeddings provider node.
- **Credentials:** OpenAI API.
- **Edge cases:** Model not specified (uses default for node/version); rate limiting.

#### Post-Mortem Generator
- **Type/role:** LangChain Agent — generates structured post-mortem.
- **Connections:** uses **OpenAI GPT-4 Post-Mortem** + **Post-Mortem Output Schema**; outputs to **Update Learning Database**.
- **Edge cases:** schema mismatch, large payload.

#### Update Learning Database
- **Type/role:** `Postgres` — stores pattern learning.
- **Config:** `public.learning_database`, matching columns include many fields (pattern_id, incident_type, root_cause_pattern, etc.).
- **Edge cases:** Upstream post-mortem schema does not contain these learning DB fields; likely writes nulls unless transformed.

#### Send Post-Mortem Report
- **Type/role:** `Send Email`
- **Config:** HTML template referencing `$json.*` fields such as `incidentId`, `incidentDate`, `executiveSummary`, nested `rootCause.*`, `impact.*`, etc.
- **Edge cases:** Post-mortem schema in this workflow does not match these fields (it defines `incidentId`, `timeline[]`, `rootCauseAnalysis`, `businessImpact`, etc.). Email will likely be missing content unless additional mapping is added.
- **Requirements:** SMTP credentials/config in n8n.

---

### Block 10 — Sticky Notes (Comments / Documentation)
The workflow includes several sticky notes whose content appears unrelated to this AIOps workflow (they describe NVIDIA NIM / customer support routing). They still must be preserved in the node summary table as “Sticky Note” content for covered nodes; however, their positions suggest they may not meaningfully annotate this workflow’s nodes.

Sticky notes present:
- Sticky Note: “Prerequisites / Use Cases / Customization / Benefits …”
- Sticky Note1: “Setup Steps … NVIDIA / OpenAI / Anthropic / Gmail / Sheets / webhook …”
- Sticky Note2: “How It Works … customer journey management …”
- Sticky Note3: “Response Delivery & Logging …”
- Sticky Note4: “Multi-Model Response Generation …”
- Sticky Note5: “Intent Classification & Routing …”
- Sticky Note6: “Trigger & Data Capture …”

(If you want accurate per-node coverage mapping, you typically need the UI canvas bounds; JSON position/size is present, but coverage is ambiguous without rendering.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Metrics Collection Schedule | scheduleTrigger | Entry point timer | — | Workflow Configuration |  |
| Workflow Configuration | set | Central config/constants | Metrics Collection Schedule | Fetch Metrics Data; Fetch Logs Data; Fetch Traces Data; Fetch Alerts Data |  |
| Fetch Metrics Data | httpRequest | Pull metrics from observability API | Workflow Configuration | Merge Observability Data |  |
| Fetch Logs Data | httpRequest | Pull logs from observability API | Workflow Configuration | Merge Observability Data |  |
| Fetch Traces Data | httpRequest | Pull traces from observability API | Workflow Configuration | Merge Observability Data |  |
| Fetch Alerts Data | httpRequest | Pull alerts from observability API | Workflow Configuration | Merge Observability Data |  |
| Merge Observability Data | merge | Combine 4 observability streams | Fetch Metrics Data; Fetch Logs Data; Fetch Traces Data; Fetch Alerts Data | Anomaly Detection Engine |  |
| Anomaly Detection Engine | code (python) | Compute anomaly scores and affected services | Merge Observability Data | Temporal Forecasting Model |  |
| Temporal Forecasting Model | code (python) | Predict incident likelihood & time-to-impact | Anomaly Detection Engine | Causal Inference Analysis |  |
| Causal Inference Analysis | code (python) | Infer causal chain/root cause via traces | Temporal Forecasting Model | Incident Predicted? |  |
| Incident Predicted? | if | Gate: proceed only if risk detected | Causal Inference Analysis | Root Cause Correlation Agent |  |
| Root Cause Correlation Agent | langchain agent | LLM-based RCA synthesis (structured) | Incident Predicted? | Blast Radius Evaluator |  |
| OpenAI GPT-4 | lmChatOpenAi | LLM for RCA agent | — (AI connection) | Root Cause Correlation Agent (AI) |  |
| Root Cause Output Schema | outputParserStructured | Enforce RCA JSON structure | — (AI connection) | Root Cause Correlation Agent (AI) |  |
| Blast Radius Evaluator | langchain agent | LLM impact/blast radius assessment | Root Cause Correlation Agent | Auto-Remediation Safe? |  |
| OpenAI GPT-4 Blast Radius | lmChatOpenAi | LLM for blast radius agent | — (AI connection) | Blast Radius Evaluator (AI) |  |
| Blast Radius Output Schema | outputParserStructured | Enforce blast radius JSON | — (AI connection) | Blast Radius Evaluator (AI) |  |
| Auto-Remediation Safe? | if | Decide auto-remediate vs approval | Blast Radius Evaluator | Remediation Strategy Planner; Request Manual Approval |  |
| Remediation Strategy Planner | langchain agent | LLM remediation plan (structured) | Auto-Remediation Safe?; Approval Granted? | Execute Remediation Action |  |
| OpenAI GPT-4 Remediation | lmChatOpenAi | LLM for remediation planner | — (AI connection) | Remediation Strategy Planner (AI) |  |
| Remediation Plan Output Schema | outputParserStructured | Enforce remediation plan JSON | — (AI connection) | Remediation Strategy Planner (AI) |  |
| Execute Remediation Action | httpRequest | Execute first remediation action | Remediation Strategy Planner | Wait for Remediation Effect |  |
| Wait for Remediation Effect | wait | Wait for changes to take effect | Execute Remediation Action | Verify Remediation Success |  |
| Verify Remediation Success | httpRequest | Re-check metrics for success | Wait for Remediation Effect | Remediation Successful? |  |
| Remediation Successful? | if | Decide success vs rollback | Verify Remediation Success | Merge Remediation Paths; Execute Rollback |  |
| Execute Rollback | httpRequest | Roll back remediation | Remediation Successful? | Merge Remediation Paths |  |
| Request Manual Approval | slack | Ask humans to approve remediation | Auto-Remediation Safe? | — |  |
| Approval Response Webhook | webhook | Receive approval decision (2nd entrypoint) | — | Approval Granted? |  |
| Approval Granted? | if | Branch: approved vs escalate | Approval Response Webhook | Remediation Strategy Planner; Escalation Diagnostics Generator |  |
| Escalation Diagnostics Generator | langchain agent | Create structured escalation diagnostics | Approval Granted? | Escalate to On-Call Team |  |
| OpenAI GPT-4 Diagnostics | lmChatOpenAi | LLM for diagnostics agent | — (AI connection) | Escalation Diagnostics Generator (AI) |  |
| Diagnostics Output Schema | outputParserStructured | Enforce diagnostics JSON | — (AI connection) | Escalation Diagnostics Generator (AI) |  |
| Escalate to On-Call Team | slack | Notify on-call with context | Escalation Diagnostics Generator | Merge Remediation Paths |  |
| Merge Remediation Paths | merge | Consolidate success/rollback/escalation | Remediation Successful?; Execute Rollback; Escalate to On-Call Team | Store Incident in Knowledge Graph; Incident Vector Store |  |
| Store Incident in Knowledge Graph | postgres | Persist incident record | Merge Remediation Paths | — |  |
| Incident Vector Store | vectorStorePGVector | Store incident embeddings in PGVector | Merge Remediation Paths; OpenAI Embeddings; Incident Document Loader | Post-Mortem Generator |  |
| OpenAI Embeddings | embeddingsOpenAi | Generate embeddings | — (AI connection) | Incident Vector Store (AI) |  |
| Incident Document Loader | documentDefaultDataLoader | Build documents to embed | — (AI connection) | Incident Vector Store (AI) |  |
| Post-Mortem Generator | langchain agent | Generate structured post-mortem | Incident Vector Store | Update Learning Database |  |
| OpenAI GPT-4 Post-Mortem | lmChatOpenAi | LLM for post-mortem agent | — (AI connection) | Post-Mortem Generator (AI) |  |
| Post-Mortem Output Schema | outputParserStructured | Enforce post-mortem JSON | — (AI connection) | Post-Mortem Generator (AI) |  |
| Update Learning Database | postgres | Persist learnings/patterns | Post-Mortem Generator | Send Post-Mortem Report |  |
| Send Post-Mortem Report | emailSend | Email post-mortem HTML | Update Learning Database | — |  |
| Sticky Note | stickyNote | Comment | — | — | ## Prerequisites\nNVIDIA NIM API access, OpenAI API key, Anthropic API credentials \n## Use Cases\nCustomer support automation with tiered response complexity \n## Customization\nAdjust AI model selection criteria based on query keywords or customer segments. \n## Benefits\nReduces response time by 80% through instant AI-powered replies. |
| Sticky Note1 | stickyNote | Comment | — | — | ## Setup Steps\n1. Configure NVIDIA API credentials with appropriate model access \n2. Add OpenAI API key with GPT-4 access for general query handling\n3. Set up Anthropic Claude API credentials for complex reasoning tasks\n4. Connect Gmail account for automated email sending and monitoring\n5. Configure Google Sheets with customer interaction tracking template\n6. Set webhook URL for external system integrations |
| Sticky Note2 | stickyNote | Comment | — | — | ## How It Works\nThis workflow automates end-to-end customer journey management by intelligently routing queries through multiple AI models (OpenAI, Claude) based on complexity and context. Designed for customer success teams, support operations, and sales organizations, it solves the challenge of delivering personalized, context-aware responses at scale while maintaining conversation continuity. The system captures customer interactions, analyzes sentiment and intent, routes to appropriate AI models, generates tailored responses, and tracks engagement metrics. It integrates email automation, database logging, and multi-channel communication to create a seamless experience. By combining NVIDIA's specialized models for technical queries, OpenAI for general assistance, and Claude for complex reasoning, it ensures optimal response quality while reducing manual workload by 70%. |
| Sticky Note3 | stickyNote | Comment | — | — | ## Response Delivery & Logging\n**Why:** Sends formatted responses via preferred channels, logs interactions to Google Sheets for analytics, and triggers follow-up sequences based on customer engagement patterns. |
| Sticky Note4 | stickyNote | Comment | — | — | ## Multi-Model Response Generation\n**Why:** Leverages specialized AI capabilities for technical depth, OpenAI for conversational flow, Claude for nuanced reasoning—producing contextually appropriate answers. |
| Sticky Note5 | stickyNote | Comment | — | — | ## Intent Classification & Routing\n**Why:** Uses AI to analyze query complexity, sentiment, and type, then intelligently routes to the most suitable model, ensuring cost-effective and accurate responses. |
| Sticky Note6 | stickyNote | Comment | — | — | ## Trigger & Data Capture\n**Why:** Initiates workflow from multiple sources (email, webhook, form submission) and normalizes customer data for consistent processing across channels. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (name it as desired).
2. **Add node: Schedule Trigger**  
   - Name: `Metrics Collection Schedule`  
   - Interval: Every **2 minutes**.
3. **Add node: Set**  
   - Name: `Workflow Configuration`  
   - Add fields (example values):  
     - `metricsApiUrl` (string)  
     - `logsApiUrl` (string)  
     - `tracesApiUrl` (string)  
     - `alertsApiUrl` (string)  
     - `remediationApiUrl` (string)  
     - `anomalyThreshold` (number, e.g. 0.85)  
     - `forecastHorizonMinutes` (number, e.g. 30)  
     - `blastRadiusThreshold` (number, e.g. 0.7)  
     - `autoRemediationEnabled` (boolean, true)  
     - `maxWaitTimeSeconds` (number, e.g. 300)
   - Enable “Include other fields” (optional).
4. **Connect** `Metrics Collection Schedule` → `Workflow Configuration`.

5. **Add 4 HTTP Request nodes** (GET unless your APIs require otherwise):
   - `Fetch Metrics Data`
     - URL: `={{ $('Workflow Configuration').first().json.metricsApiUrl }}`
     - Header: `Authorization: Bearer <token>`
   - `Fetch Logs Data`
     - URL: `={{ $('Workflow Configuration').first().json.logsApiUrl }}`
     - Header: `Authorization: Bearer <token>`
   - `Fetch Traces Data`
     - URL: `={{ $('Workflow Configuration').first().json.tracesApiUrl }}`
     - Header: `Authorization: Bearer <token>`
   - `Fetch Alerts Data`
     - URL: `={{ $('Workflow Configuration').first().json.alertsApiUrl }}`
     - Header: `Authorization: Bearer <token>`
6. **Connect** `Workflow Configuration` → each of the 4 HTTP nodes (fan-out).

7. **Add node: Merge**
   - Name: `Merge Observability Data`
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: **4**
8. **Connect** each fetch node to the merge node inputs in order:
   - Metrics → input 0, Logs → input 1, Traces → input 2, Alerts → input 3.

9. **Add node: Code (Python)**
   - Name: `Anomaly Detection Engine`
   - Paste the anomaly detection Python logic.
   - Ensure your n8n environment supports Python and required libs (NumPy; SciPy if kept).
10. **Connect** `Merge Observability Data` → `Anomaly Detection Engine`.

11. **Add node: Code (Python)**
   - Name: `Temporal Forecasting Model`
   - Paste the forecasting Python logic.
12. **Connect** `Anomaly Detection Engine` → `Temporal Forecasting Model`.

13. **Add node: Code (Python)**
   - Name: `Causal Inference Analysis`
   - Paste the causal inference Python logic.
14. **Connect** `Temporal Forecasting Model` → `Causal Inference Analysis`.

15. **Add node: IF**
   - Name: `Incident Predicted?`
   - Condition group: OR
     - `={{ $json.anomalyDetected }}` is true
     - `={{ $json.forecastedIncident }}` is true
     - `={{ $json.causalChainDetected }}` is true
16. **Connect** `Causal Inference Analysis` → `Incident Predicted?` (true output used; false can remain unconnected).

17. **Configure OpenAI credential**
   - Add an OpenAI API credential in n8n (API key with access to the selected model).

18. **Root cause agent trio**
   1) Add `LM Chat OpenAI` node: `OpenAI GPT-4`
      - Model: `gpt-4.1-mini`
      - Credential: OpenAI
   2) Add `Structured Output Parser` node: `Root Cause Output Schema`
      - Schema: as defined in the workflow (manual JSON schema).
   3) Add `AI Agent` node: `Root Cause Correlation Agent`
      - Prompt type: define
      - Text: `={{ JSON.stringify($json) }}`
      - System message: copy from workflow
      - Enable output parser
   4) Connect AI ports:
      - `OpenAI GPT-4` → `Root Cause Correlation Agent` (AI Language Model)
      - `Root Cause Output Schema` → `Root Cause Correlation Agent` (AI Output Parser)
   5) Connect main flow:
      - `Incident Predicted?` (true) → `Root Cause Correlation Agent`

19. **Blast radius agent trio**
   1) `LM Chat OpenAI`: `OpenAI GPT-4 Blast Radius` (same model/credential)
   2) `Structured Output Parser`: `Blast Radius Output Schema`
   3) `AI Agent`: `Blast Radius Evaluator` (text = JSON.stringify($json), system message as provided)
   4) Connect AI ports to the agent and connect:
      - `Root Cause Correlation Agent` → `Blast Radius Evaluator`

20. **Add node: IF**
   - Name: `Auto-Remediation Safe?`
   - Condition group: AND
     - `={{ $json.autoRemediationSafe }}` equals true
     - `={{ $json.requiresApproval }}` equals false
     - `={{ $('Workflow Configuration').first().json.autoRemediationEnabled }}` equals true
21. **Connect** `Blast Radius Evaluator` → `Auto-Remediation Safe?`.

22. **Remediation planning agent trio**
   1) `LM Chat OpenAI`: `OpenAI GPT-4 Remediation`
   2) `Structured Output Parser`: `Remediation Plan Output Schema`
   3) `AI Agent`: `Remediation Strategy Planner`
      - Text: `={{ JSON.stringify($json) }}`
      - System message as provided
   4) Connect AI ports, then connect:
      - `Auto-Remediation Safe?` (true) → `Remediation Strategy Planner`

23. **Execute remediation**
   - Add `HTTP Request`: `Execute Remediation Action`
     - URL: `={{ $json.remediationActions[0].endpoint }}`
     - Method: `={{ $json.remediationActions[0].method }}`
     - Send body: true, raw JSON
     - Body: `={{ JSON.stringify($json.remediationActions[0].payload) }}`
     - Headers: `Content-Type: application/json`, `Authorization: Bearer <token>`
   - Connect `Remediation Strategy Planner` → `Execute Remediation Action`.

24. **Add `Wait` node:** `Wait for Remediation Effect`
   - Amount seconds: `={{ $('Workflow Configuration').first().json.maxWaitTimeSeconds }}`
   - Connect `Execute Remediation Action` → `Wait for Remediation Effect`.

25. **Add `HTTP Request` node:** `Verify Remediation Success`
   - URL: `={{ $('Workflow Configuration').first().json.metricsApiUrl }}`
   - Header Authorization for verification endpoint
   - Connect `Wait for Remediation Effect` → `Verify Remediation Success`.

26. **Add `IF` node:** `Remediation Successful?`
   - Condition: `={{ $('Verify Remediation Success').item.json.validationCriteriaMet }}` is true
   - Connect `Verify Remediation Success` → `Remediation Successful?`.

27. **Add `HTTP Request` node:** `Execute Rollback`
   - POST to rollback endpoint and payload
   - Connect `Remediation Successful?` (false) → `Execute Rollback`.

28. **Manual approval branch**
   1) Create Slack OAuth2 credential in n8n.
   2) Add `Slack` node: `Request Manual Approval`
      - Post to approvals channel
      - Compose message and include a link to your webhook endpoint `/approval-response` (construct manually using your n8n base URL).
   3) Connect `Auto-Remediation Safe?` (false) → `Request Manual Approval`.

29. **Approval webhook entrypoint**
   - Add `Webhook` node: `Approval Response Webhook`
     - POST, path `approval-response`
     - Response mode: last node
   - Add `IF` node: `Approval Granted?`
     - Condition: `={{ $json.body.approved }}` is true
   - Connect `Approval Response Webhook` → `Approval Granted?`.
   - Connect `Approval Granted?` (true) → `Remediation Strategy Planner`.

30. **Escalation diagnostics**
   - Add `LM Chat OpenAI`: `OpenAI GPT-4 Diagnostics`
   - Add `Structured Output Parser`: `Diagnostics Output Schema`
   - Add `AI Agent`: `Escalation Diagnostics Generator` (system message as provided)
   - Connect AI ports.
   - Connect `Approval Granted?` (false) → `Escalation Diagnostics Generator`.

31. **Escalate to on-call in Slack**
   - Add `Slack` node: `Escalate to On-Call Team` (on-call channel)
   - Connect `Escalation Diagnostics Generator` → `Escalate to On-Call Team`.

32. **Merge remediation outcomes**
   - Add `Merge` node: `Merge Remediation Paths` with **3 inputs**
   - Connect:
     - `Remediation Successful?` (true) → Merge input 0
     - `Execute Rollback` → Merge input 1
     - `Escalate to On-Call Team` → Merge input 2

33. **Persistence (Postgres)**
   - Configure Postgres credential.
   - Add `Postgres` node: `Store Incident in Knowledge Graph`
     - Table: `public.incidents`
     - Map the columns as needed (you will likely need an intermediate Set node to align field names).
   - Connect `Merge Remediation Paths` → `Store Incident in Knowledge Graph`.

34. **Vector storage (PGVector)**
   - Ensure pgvector extension and `incident_embeddings` table exist.
   - Add `Document Loader`: `Incident Document Loader`
   - Add `OpenAI Embeddings`: `OpenAI Embeddings`
   - Add `PGVector Vector Store`: `Incident Vector Store` (mode insert, collection `incidents`, table `incident_embeddings`)
   - Connect:
     - Main: `Merge Remediation Paths` → `Incident Vector Store`
     - AI document: `Incident Document Loader` → `Incident Vector Store`
     - AI embedding: `OpenAI Embeddings` → `Incident Vector Store`

35. **Post-mortem generation & learning update**
   - Add `LM Chat OpenAI`: `OpenAI GPT-4 Post-Mortem`
   - Add `Structured Output Parser`: `Post-Mortem Output Schema`
   - Add `AI Agent`: `Post-Mortem Generator`
   - Connect `Incident Vector Store` → `Post-Mortem Generator` and wire AI ports.
   - Add `Postgres`: `Update Learning Database` and connect `Post-Mortem Generator` → it.
   - Add `Send Email`: `Send Post-Mortem Report` and connect `Update Learning Database` → it.
   - Configure SMTP / email settings and recipients.

---

### Notable integration gaps to fix when reproducing
- Align field names between analytics outputs → forecasting → causal inference (currently inconsistent).
- Ensure the Slack message templates reference fields that actually exist (or add Set/Code mapping).
- Ensure rollback endpoint/payload fields exist (schema vs HTTP rollback node mismatch).
- Decide merge behavior explicitly and ensure merges won’t block when only one path executes.
- Verify Python environment supports NumPy/SciPy or remove SciPy imports.

If you want, I can propose a minimal set of “mapping” Set/Code nodes (and corrected schemas) to make the current workflow run end-to-end without missing-field issues.