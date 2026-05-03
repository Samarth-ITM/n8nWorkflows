Run multi-model research analysis and email reports with GPT-4, Claude and NVIDIA NIM

https://n8nworkflows.xyz/workflows/run-multi-model-research-analysis-and-email-reports-with-gpt-4--claude-and-nvidia-nim-12537


# Run multi-model research analysis and email reports with GPT-4, Claude and NVIDIA NIM

## 1. Workflow Overview

**Purpose:** This workflow exposes a POST webhook that accepts a research/query prompt plus governance/cost/latency requirements, evaluates available LLM providers, selects an eligible model, rewrites the prompt into the chosen provider’s request format, calls the provider endpoint, applies retry/circuit-breaker logic, normalizes the response, computes telemetry and quality signals (hallucination risk + confidence), stores results/telemetry in Postgres, detects anomalies and optionally alerts Slack, then builds and emails a consolidated HTML report.

**Target use cases:** multi-provider LLM orchestration (Azure OpenAI / AWS Bedrock / Google Vertex AI / local model endpoint), cost/performance governance, automated reporting for researchers/analysts.

### Logical blocks (node dependency-based)

1.1 **Request Intake & Global Config**  
Webhook entry, set shared endpoints and thresholds.

1.2 **Request Parsing, Validation & Prompt Complexity**  
Normalize the incoming payload into a validated structure and compute complexity/domain heuristics.

1.3 **Provider Health Checks & Model Ranking**  
Hit health endpoints, merge results, compute a ranked model list (cost/latency/quality/health weights).

1.4 **Policy/Governance Routing**  
Optionally apply regulatory/policy filtering and select a compliant provider.

1.5 **Provider-Specific Prompt Rewriting & LLM Invocation**  
Rewrite prompt into Azure/AWS/Google/local payload formats and call the selected provider endpoint.

1.6 **Resilience: Circuit Breaker + Adaptive Retry**  
Evaluate provider call success, decide retry vs fallback path, wait with exponential backoff when retrying.

1.7 **Response Aggregation & Normalization**  
Merge model response streams and normalize output into a single canonical response structure.

1.8 **Telemetry + Quality Scoring (Hallucination & Confidence)**  
Compute cost/latency/tokens, hallucination risk score, and a combined confidence score.

1.9 **Persistence (Telemetry + Results) & Anomaly Detection + Alerting**  
Write telemetry and results to Postgres; detect anomalies and optionally send a Slack alert.

1.10 **Report Generation & Email Delivery**  
Query Postgres for report datasets, format an HTML report, and send it by email.

---

## 2. Block-by-Block Analysis

### 1.1 Request Intake & Global Config

**Overview:** Receives the orchestration request and sets central configuration values (endpoints and thresholds) used across the workflow.

**Nodes involved:**  
- LLM Request Webhook  
- Workflow Configuration

#### Node: **LLM Request Webhook**
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point for external callers.
- **Config choices:**
  - HTTP Method: `POST`
  - Path: `llm-orchestration`
  - Response Mode: `lastNode` (the HTTP response comes from the final executed node on the main path)
- **Inputs/Outputs:** No input; outputs to **Workflow Configuration**.
- **Edge cases / failures:**
  - Caller sends non-JSON payload → downstream parsing may fail.
  - Large payloads may hit n8n request size limits (instance setting).
- **Version requirements:** typeVersion `2.1`.

#### Node: **Workflow Configuration**
- **Type / role:** `Set` — centralizes environment-specific endpoints and tunables.
- **Config choices (interpreted):**
  - Sets placeholders for:
    - Health endpoints: `azureHealthEndpoint`, `awsHealthEndpoint`, `googleHealthEndpoint`, `localHealthEndpoint`
    - API endpoints: `azureApiEndpoint`, `awsApiEndpoint`, `googleApiEndpoint`, `localApiEndpoint`
  - Reliability and governance defaults:
    - `maxRetries` (3), `retryBaseDelay` (1000 ms), `circuitBreakerThreshold` (5)
    - `anomalyThreshold` (0.8), `costCeilingDefault` (1), `latencySLODefault` (5000)
  - `includeOtherFields: true` keeps incoming fields.
- **Key variables:** Referenced later via expressions like `$('Workflow Configuration').first().json.azureApiEndpoint`.
- **Inputs/Outputs:** From webhook; to **Parse Request & Validate**.
- **Edge cases / failures:**
  - Placeholders not replaced → health checks and LLM calls will fail with invalid URL.
- **Version:** typeVersion `3.4`.

---

### 1.2 Request Parsing, Validation & Prompt Complexity

**Overview:** Standardizes the incoming request shape, validates required fields, and computes prompt complexity signals used for model selection and rewriting.

**Nodes involved:**  
- Parse Request & Validate  
- Analyze Prompt Complexity

#### Node: **Parse Request & Validate**
- **Type / role:** `Code` — validates and normalizes the inbound JSON.
- **Config choices:**
  - Extracts prompt from `body.prompt || body.query || body.text`.
  - Extracts `tenantId` from `tenantId || tenant_id || tenant`.
  - Builds structured objects:
    - `taskMetadata` (taskId, type, priority, timestamp)
    - `regulatoryConstraints` (dataResidency, compliance requirements, PII handling, classification)
    - `costCeilings` (maxCostPerRequest, budgetLimit, costOptimizationEnabled)
    - `latencySLOs` (maxLatencyMs, p95/p99 targets, timeoutMs)
    - `qualityExpectations` (minConfidenceScore, fact-checking flag, hallucination tolerance, outputFormat, maxTokens, temperature)
  - Adds `validationErrors` and `isValid`.
- **Key expressions/variables:** Uses `$input.all()` and per-item `item.json.body || item.json`.
- **Inputs/Outputs:** From **Workflow Configuration**; to **Analyze Prompt Complexity**.
- **Edge cases / failures:**
  - `body` may not exist depending on webhook configuration; code handles fallback to `item.json`.
  - Missing prompt/tenantId leads to `isValid=false` but the workflow does not branch on `isValid` later (potential logic gap).
- **Version:** typeVersion `2`.

#### Node: **Analyze Prompt Complexity**
- **Type / role:** `Code` — heuristic scoring for prompt size/complexity and domain.
- **Config choices:**
  - Token estimate: `ceil(chars/4)`
  - ComplexityScore uses sentence count, average word length, unique word ratio, punctuation density.
  - Keyword-based domain classification (technical/creative/analytical/conversational/business)
  - Produces `promptAnalysis` (tokenCount, complexityScore, primaryDomain, etc.)
- **Inputs/Outputs:** From **Parse Request & Validate**; outputs to both health checks:
  - **Check Azure OpenAI Health**
  - **Check AWS Bedrock Health**
- **Edge cases / failures:**
  - If prompt is empty (invalid request), analysis still runs with empty string.
- **Version:** typeVersion `2`.

---

### 1.3 Provider Health Checks & Model Ranking

**Overview:** Performs health checks on providers, merges results, and computes ranked/eligible models. (In the current JSON, only Azure and AWS health checks are wired.)

**Nodes involved:**  
- Check Azure OpenAI Health  
- Check AWS Bedrock Health  
- Merge Health Checks  
- Score & Rank Models

#### Node: **Check Azure OpenAI Health**
- **Type / role:** `HTTP Request` — health probe.
- **Config choices:**
  - URL from `Workflow Configuration.azureHealthEndpoint`
  - Timeout 3000ms, JSON response format
  - `allowUnauthorizedCerts: true` (useful for internal/self-signed endpoints)
- **Inputs/Outputs:** From **Analyze Prompt Complexity**; to **Merge Health Checks**.
- **Edge cases / failures:**
  - TLS/invalid cert issues mitigated by allowUnauthorizedCerts, but that weakens security.
  - Endpoint returns non-JSON → parsing error.
- **Version:** typeVersion `4.3`.

#### Node: **Check AWS Bedrock Health**
- **Type / role:** `HTTP Request` — health probe.
- **Config choices:** URL from `Workflow Configuration.awsHealthEndpoint`, timeout 3000ms, JSON response.
- **Inputs/Outputs:** From **Analyze Prompt Complexity**; to **Merge Health Checks**.
- **Edge cases:** Auth may be required for health endpoint; node has no auth configured.
- **Version:** typeVersion `4.3`.

#### Node: **Merge Health Checks**
- **Type / role:** `Merge` — combines the two health check streams.
- **Config choices:** Mode `combine`, `combineAll`.
- **Inputs/Outputs:** From Azure+AWS health nodes; to **Score & Rank Models**.
- **Edge cases:**
  - If one branch errors, merge may not receive data; execution may fail upstream depending on “continue on fail” (not enabled here).
- **Version:** typeVersion `3.2`.

#### Node: **Score & Rank Models**
- **Type / role:** `Code` — computes weighted scores across providers and chooses a recommendation.
- **Config choices (important):**
  - Expects input items labeled by `item.json.type` such as:
    - `health_check`, `historical_performance`, `prompt_complexity`, `config`
  - Defines characteristics for: `azure_openai`, `aws_bedrock`, `google_vertex`, `local_model`
  - Computes:
    - healthScore, costScore, latencyScore, qualityScore
    - `meetsConstraints` based on max cost/latency/min quality
  - Returns `rankedModels`, `eligibleModels`, `recommendedModel`
- **Connections:** Output to **Check Policy Constraints**.
- **Critical integration gaps (likely breaking behavior):**
  - The health check nodes do **not** produce objects with `{type:'health_check', model:'azure_openai', status:'healthy'}` etc. There is no mapping layer, so `healthChecks` may be empty → `rankedModels` becomes empty → recommendedModel `null`.
  - Prompt complexity is produced as `promptAnalysis` and does not set `type:'prompt_complexity'` nor `score`.
  - Config is not passed as `type:'config'` either.
- **Version:** typeVersion `2`.

---

### 1.4 Policy/Governance Routing

**Overview:** Determines whether policy constraints apply and either applies policy routing or routes directly to model selection. Current expressions likely don’t match available fields.

**Nodes involved:**  
- Check Policy Constraints  
- Apply Policy Routing  
- Route to Selected Model

#### Node: **Check Policy Constraints**
- **Type / role:** `IF` — decides whether governance-based routing is needed.
- **Config choices:**
  - OR conditions check existence/true on:
    - `$('Score & Rank Models').item.json.dataResidency`
    - `$('Score & Rank Models').item.json.complianceRequirements`
    - `$('Score & Rank Models').item.json.sensitiveDataFlag` equals true
- **Connections:**
  - **true** → **Apply Policy Routing**
  - **false** → **Route to Selected Model**
- **Critical issues:**
  - `Score & Rank Models` output does not include `dataResidency`, `complianceRequirements`, `sensitiveDataFlag` as written. Those live earlier in `regulatoryConstraints` from the validation node. This IF will almost always evaluate false (or behave unpredictably).
- **Version:** typeVersion `2.3`.

#### Node: **Apply Policy Routing**
- **Type / role:** `Code` — filters ranked models based on PII/PHI and residency constraints, then selects top compliant model.
- **Config choices:**
  - Expects:
    - `item.json.rankedModels`
    - `item.json.promptAnalysis` (with `containsPII`, `containsPHI`, etc.)
    - `item.json.policyConstraints`
  - Uses `modelCompliance` keyed by providers like `'azure-openai'`, `'aws-bedrock'`.
  - Filters models using `model.provider` lookup.
- **Critical issues:**
  - `rankedModels` produced by scoring uses `modelName` like `azure_openai` (underscore), not `provider` like `azure-openai` (hyphen). Filtering will likely remove everything or misbehave.
  - `promptAnalysis` currently has no PII/PHI detection fields.
  - Throws hard error if no model remains.
- **Outputs:** to **Route to Selected Model**
- **Version:** typeVersion `2`.

#### Node: **Route to Selected Model**
- **Type / role:** `Switch` — routes execution to provider-specific rewrite path.
- **Config choices:**
  - Switches on `{{ $json.selectedModel }}` equal to `azure/aws/google/local`
  - Fallback output named `Fallback`
- **Critical issues:**
  - Upstream sets `selectedModel` as an object in policy routing, or uses `recommendedModel` in ranking, not `selectedModel` string values `azure/aws/google/local`.
  - As-is, almost all items will go to fallback (but fallback is not connected to anything).
- **Version:** typeVersion `3.4`.

---

### 1.5 Provider-Specific Prompt Rewriting & LLM Invocation

**Overview:** Formats the prompt into each provider’s expected payload and calls the configured API endpoint.

**Nodes involved:**  
- Rewrite Prompt for Azure → Call Azure OpenAI  
- Rewrite Prompt for AWS → Call AWS Bedrock  
- Rewrite Prompt for Google → Call Google Vertex AI  
- Rewrite Prompt for Local → Call Local Model

#### Node: **Rewrite Prompt for Azure**
- **Type / role:** `Code` — creates an Azure Chat Completions-style request.
- **Config choices:**
  - Builds `azureRequest.messages` with system+user
  - Adjusts temperature/max_tokens by `complexity` (‘high’/‘low’)
  - Sets `provider:'azure'`
- **Critical issues:**
  - It outputs `azureRequest`, but the next node (**Call Azure OpenAI**) sends `$json.requestBody`, which is not set here.
- **Version:** typeVersion `2`.

#### Node: **Call Azure OpenAI**
- **Type / role:** `HTTP Request` — invokes Azure endpoint.
- **Config choices:**
  - URL: `Workflow Configuration.azureApiEndpoint`
  - Body: `JSON.stringify($json.requestBody)`
  - Auth: `genericCredentialType` with `httpHeaderAuth`
  - Adds header `Content-Type: application/json`
  - Timeout: `{{$json.latencySLO || 30000}}`
- **Critical issues:**
  - `requestBody` missing (should likely be `azureRequest`).
  - `latencySLO` field is not created; validation uses `latencySLOs.timeoutMs`.
- **Outputs:** to **Handle Circuit Breaker** and also to **Merge Model Responses**.
- **Version:** typeVersion `4.3`.

#### Node: **Rewrite Prompt for AWS**
- **Type / role:** `Code` — formats request for Bedrock (Claude/AI21/Titan/Cohere).
- **Config choices:**
  - Detects provider prefix from model id (e.g., `anthropic.claude-v2`)
  - Creates `bedrockRequestBody` and `bedrockRequestBodyString`
  - Sets `provider:'aws_bedrock'`
- **Critical issues:**
  - Next node sends `$json.requestBody` (not set); should use `bedrockRequestBody`.
- **Version:** typeVersion `2`.

#### Node: **Call AWS Bedrock**
- **Type / role:** `HTTP Request`
- **Config choices:** URL from `awsApiEndpoint`, body from `$json.requestBody`, header auth, JSON content type.
- **Outputs:** to **Handle Circuit Breaker** and to **Merge Model Responses** (index 1).
- **Critical issues:** same requestBody mismatch.
- **Version:** typeVersion `4.3`.

#### Node: **Rewrite Prompt for Google**
- **Type / role:** `Code` — formats Vertex AI style payload.
- **Config choices:**
  - Creates `vertexPayload.instances[0].content`
  - Adds safety settings
  - Sets `provider:'google-vertex'`
- **Critical issues:** Next node expects `$json.requestBody`, but rewrite node outputs `vertexPayload`.
- **Version:** typeVersion `2`.

#### Node: **Call Google Vertex AI**
- **Type / role:** `HTTP Request`
- **Critical issues:** requestBody mismatch; auth uses header auth generic credential but Vertex typically uses OAuth2/JWT.
- **Outputs:** to **Handle Circuit Breaker** only.
- **Version:** typeVersion `4.3`.

#### Node: **Rewrite Prompt for Local**
- **Type / role:** `Code` — formats a typical local LLM server payload (Ollama/LM Studio style).
- **Config choices:** Outputs `localModelRequest` and sets `modelProvider:'local'`.
- **Critical issues:** Next node expects `$json.requestBody`, not `localModelRequest`.
- **Version:** typeVersion `2`.

#### Node: **Call Local Model**
- **Type / role:** `HTTP Request`
- **Config choices:** URL from `localApiEndpoint`, raw JSON body from `$json.requestBody`.
- **Outputs:** to **Handle Circuit Breaker**.
- **Critical issues:** requestBody mismatch; also lacks auth settings (might be fine locally).
- **Version:** typeVersion `4.3`.

---

### 1.6 Resilience: Circuit Breaker + Adaptive Retry

**Overview:** Evaluates provider call success/failure, tracks per-model failure counts, and triggers retries with exponential backoff.

**Nodes involved:**  
- Handle Circuit Breaker  
- Check Retry Needed  
- Adaptive Retry Delay

#### Node: **Handle Circuit Breaker**
- **Type / role:** `Code` — stateful circuit breaker + retry decision.
- **Config choices:**
  - Hardcoded: `FAILURE_THRESHOLD=3`, open duration 60s, `MAX_RETRIES=2`
  - Derives success from `item.statusCode` (2xx)
  - Stores state in `$('Workflow Configuration').item.json.circuitState` (but Set node doesn’t persist state across runs)
  - Outputs `shouldRetry`, `shouldFallback`, `retryCount`, and `circuitState`
- **Critical issues:**
  - n8n executions are stateless unless using external storage; this “state” won’t persist between executions.
  - Many HTTP responses in n8n don’t expose `statusCode` inside `json` automatically unless configured; it may be under node metadata.
- **Outputs:** to **Check Retry Needed**
- **Version:** typeVersion `2`.

#### Node: **Check Retry Needed**
- **Type / role:** `IF` — checks retry gating conditions.
- **Conditions:**
  - `Handle Circuit Breaker.shouldRetry` is true
  - `retryCount < maxRetries`
  - `circuitBreakerState != 'open'` (but the code outputs `circuitBreaker.circuitStatus`, not `circuitBreakerState`)
- **Connections:**
  - **true** → **Adaptive Retry Delay**
  - **false** → **Merge Model Responses**
- **Critical issues:** field name mismatch (`circuitBreakerState` vs `circuitBreaker.circuitStatus`) makes condition unreliable.
- **Version:** typeVersion `2.3`.

#### Node: **Adaptive Retry Delay**
- **Type / role:** `Wait` — exponential backoff before retry.
- **Config choices:** wait `retryBaseDelay * 2^(retryCount)`
- **Outputs:** to **Route to Selected Model**
- **Edge cases:**
  - If retryCount undefined, defaults to 0 in expression.
- **Version:** typeVersion `1.1`.

---

### 1.7 Response Aggregation & Normalization

**Overview:** Merges responses (where connected) and converts provider-specific shapes into a consistent canonical output.

**Nodes involved:**  
- Merge Model Responses  
- Normalize Output

#### Node: **Merge Model Responses**
- **Type / role:** `Merge` — combines multiple provider response streams.
- **Config choices:** `combineAll`.
- **Inputs/Outputs:**
  - Receives from:
    - Call Azure OpenAI (index 0)
    - Call AWS Bedrock (index 1)
    - Check Retry Needed (false branch) (index 0)
  - Outputs to **Normalize Output**
- **Critical issues:**
  - Google/Local paths are not merged here (they only go through circuit breaker path).
  - Combining unrelated structures without alignment may create odd merged JSON.
- **Version:** typeVersion `3.2`.

#### Node: **Normalize Output**
- **Type / role:** `Code` — normalizes Azure/AWS/Google/Local response formats.
- **Config choices:**
  - Detects provider by `rawResponse.provider` or presence of fields (`choices`, `body`, `predictions`, `response`)
  - Produces:
    - `text`, `provider`, `model`
    - `metadata` (tokens, latency, finishReason, requestId)
    - `providerSpecific`
- **Inputs/Outputs:** from **Merge Model Responses**; to **Calculate Telemetry Metrics**
- **Critical issues:**
  - Upstream HTTP nodes may wrap responses (depending on “Response Format” settings); this code assumes direct provider-like payload.
- **Version:** typeVersion `2`.

---

### 1.8 Telemetry + Quality Scoring

**Overview:** Computes telemetry (token counts, latency, estimated cost), hallucination risk, and a final confidence score.

**Nodes involved:**  
- Calculate Telemetry Metrics  
- Assess Hallucination Risk  
- Calculate Confidence Score

#### Node: **Calculate Telemetry Metrics**
- **Type / role:** `Code` — creates a telemetry object.
- **Config choices:**
  - Reads `item.modelResponse.usage` but **Normalize Output** outputs `metadata`, not `modelResponse`.
  - Uses `selectedModel` to choose pricing, but selection fields are not consistently propagated.
- **Inputs/Outputs:** from **Normalize Output**; to **Assess Hallucination Risk**
- **Critical issues:** field mismatches mean tokens/cost likely compute as zero.
- **Version:** typeVersion `2`.

#### Node: **Assess Hallucination Risk**
- **Type / role:** `Code` — heuristic risk scoring 0..1 based on contradiction markers, uncertainty density, grounding, vagueness, repetition, and prompt-response alignment.
- **Config choices:**
  - Reads response from `item.json.response || item.json.output` but normalized output uses `text`.
- **Outputs:** to **Calculate Confidence Score**
- **Critical issues:** response text likely not found → risk computed on empty string.
- **Version:** typeVersion `2`.

#### Node: **Calculate Confidence Score**
- **Type / role:** `Code` — weighted score of certainty/completeness/1-risk/qualityIndicators.
- **Config choices:** expects numeric inputs `modelCertainty`, `responseCompleteness`, and `hallucinationRisk` as a number, but upstream produces `hallucinationRisk` as an object `{score, level,...}`.
- **Outputs:** to:
  - Log Telemetry to Database
  - Store Result in Datastore
  - Detect Anomalies
- **Critical issues:** `hallucinationRisk` object used as number leads to `1 - hallucinationRisk` → `NaN`.
- **Version:** typeVersion `2`.

---

### 1.9 Persistence & Anomaly Detection + Alerting

**Overview:** Stores telemetry and results, then runs anomaly detection and sends Slack alerts if threshold exceeded.

**Nodes involved:**  
- Log Telemetry to Database  
- Store Result in Datastore  
- Detect Anomalies  
- Check Anomaly Threshold  
- Send Anomaly Alert

#### Node: **Log Telemetry to Database**
- **Type / role:** `Postgres` — inserts into `public.llm_telemetry`.
- **Config choices:**
  - Inserts many fields via expressions like `={{ $json.cost }}`, `={{ $json.tenant_id }}`, etc.
- **Critical issues:**
  - Upstream uses `tenantId` (camelCase) not `tenant_id`.
  - Cost/latency/token fields naming is inconsistent across workflow (e.g., `telemetryMetrics.cost.totalCost` vs `$json.cost`).
  - This will likely insert nulls or fail on type mismatch depending on table schema.
- **Outputs:** triggers report queries:
  - Generate Cost Report Data
  - Generate Performance Report Data
  - Generate Governance Report Data
- **Version:** typeVersion `2.6`.

#### Node: **Store Result in Datastore**
- **Type / role:** `Postgres` — inserts into `public.llm_results`.
- **Config choices:** maps `prompt`, `metadata`, `response`, `tenant_id`, `created_at`, etc.
- **Critical issues:** normalized output uses `text` not `response`; timestamps not set as `created_at`.
- **Outputs:** to **Prepare Response**.
- **Version:** typeVersion `2.6`.

#### Node: **Detect Anomalies**
- **Type / role:** `Code` — z-score anomaly detection.
- **Config choices:**
  - Expects `latency`, `cost`, `tokenUsage`, `errorRate`
  - Uses defaults baselines if missing
  - Outputs `anomalyDetection` with `maxSeverity` and anomaly list
- **Outputs:** to **Check Anomaly Threshold**
- **Critical issues:** next node checks `anomalySeverity`, but this node outputs `anomalyDetection.maxSeverity` (string), not a numeric severity.
- **Version:** typeVersion `2`.

#### Node: **Check Anomaly Threshold**
- **Type / role:** `IF` — decides to alert.
- **Config choices:** checks `Detect Anomalies.item.json.anomalySeverity >= Workflow Configuration.anomalyThreshold`
- **Critical issues:** `anomalySeverity` doesn’t exist; `maxSeverity` is a string; comparison invalid.
- **Outputs:** to **Send Anomaly Alert**
- **Version:** typeVersion `2.3`.

#### Node: **Send Anomaly Alert**
- **Type / role:** `Slack` — sends alert message to a channel.
- **Config choices:**
  - OAuth2 Slack credential
  - Channel ID placeholder
  - Message template uses fields like `$json.model`, `$json.metric`, `$json.baselineValue` that don’t match anomaly output shape (which nests anomalies in a list).
- **Edge cases:** Slack token/channel permissions.
- **Version:** typeVersion `2.4`.

#### Node: **Prepare Response**
- **Type / role:** `Set` — prepares final webhook response fields.
- **Config choices:** sets `success`, `requestId`, `response`, `modelUsed`, `latency`, `tokens`, `cost`, `confidenceScore` from `$json.*`.
- **Critical issues:** expects snake_case fields (`request_id`, `model_provider`, `latency_ms`) that are not produced consistently.
- **Outputs:** none connected further; because webhook response mode is “lastNode”, the actual HTTP response depends on execution path and whether this node is last on that path.
- **Version:** typeVersion `3.4`.

---

### 1.10 Report Generation & Email Delivery

**Overview:** Queries telemetry/results tables for cost/performance/governance stats, merges them, formats an HTML report, and emails it.

**Nodes involved:**  
- Generate Cost Report Data  
- Generate Performance Report Data  
- Generate Governance Report Data  
- Merge Report Data  
- Format Report  
- Email Report

#### Node: **Generate Cost Report Data**
- **Type / role:** `Postgres` — aggregates 30-day cost metrics by model/tenant/day.
- **Config choices:** SQL uses `llm_telemetry.timestamp`, `SUM(cost)`, `SUM(tokens_used)`, `AVG(latency_ms)`.
- **Outputs:** to **Merge Report Data** (index 0)
- **Version:** typeVersion `2.6`.

#### Node: **Generate Performance Report Data**
- **Type / role:** `Postgres` — aggregates 24h performance metrics.
- **Config choices:** computes p95 latency and success rate percent using `status='success'`.
- **Outputs:** to **Merge Report Data** (index 1)
- **Critical issue:** Merge Report Data expects three inputs but only two are wired; Governance query is not connected.
- **Version:** typeVersion `2.6`.

#### Node: **Generate Governance Report Data**
- **Type / role:** `Postgres` — aggregates governance/compliance metrics over 30 days.
- **Config choices:** joins `llm_telemetry t` with `llm_results r`.
- **Connections:** **Not connected** in `connections` (it is triggered from Log Telemetry, but its output is not merged).
- **Version:** typeVersion `2.6`.

#### Node: **Merge Report Data**
- **Type / role:** `Merge` — combine all report datasets.
- **Config choices:** `combineAll`.
- **Inputs/Outputs:** from cost + performance (only); to **Format Report**.
- **Version:** typeVersion `3.2`.

#### Node: **Format Report**
- **Type / role:** `Code` — builds a styled HTML report.
- **Config choices:**
  - Expects `costData`, `performanceData`, `governanceData` inside input JSON; but Postgres nodes return rows arrays, not those named objects.
  - Produces `htmlReport` and metadata fields.
- **Outputs:** to **Email Report**
- **Critical issues:** mismatch between expected report data shape and actual Postgres output.
- **Version:** typeVersion `2`.

#### Node: **Email Report**
- **Type / role:** `Email Send` — sends the generated report HTML.
- **Config choices:**
  - HTML: `{{ $json.reportHtml }}` but **Format Report** outputs `htmlReport` (name mismatch).
  - To/From are placeholders.
- **Edge cases:** SMTP configuration/credentials; HTML size limitations.
- **Version:** typeVersion `2.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| LLM Request Webhook | Webhook | Receive orchestration requests | — | Workflow Configuration | ## Document Ingestion & Preprocessing\n**Why:** Extracts text from uploaded PDFs/documents and structures data for AI consumption, ensuring consistent input format across all processing models. |
| Workflow Configuration | Set | Central config for endpoints/thresholds | LLM Request Webhook | Parse Request & Validate | ## Document Ingestion & Preprocessing\n**Why:** Extracts text from uploaded PDFs/documents and structures data for AI consumption, ensuring consistent input format across all processing models. |
| Parse Request & Validate | Code | Validate/normalize input payload | Workflow Configuration | Analyze Prompt Complexity | ## Document Ingestion & Preprocessing\n**Why:** Extracts text from uploaded PDFs/documents and structures data for AI consumption, ensuring consistent input format across all processing models. |
| Analyze Prompt Complexity | Code | Heuristic complexity + domain scoring | Parse Request & Validate | Check Azure OpenAI Health; Check AWS Bedrock Health | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Check Azure OpenAI Health | HTTP Request | Health check Azure endpoint | Analyze Prompt Complexity | Merge Health Checks | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Check AWS Bedrock Health | HTTP Request | Health check AWS endpoint | Analyze Prompt Complexity | Merge Health Checks | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Merge Health Checks | Merge | Combine health results | Check Azure OpenAI Health; Check AWS Bedrock Health | Score & Rank Models | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Score & Rank Models | Code | Weighted scoring and eligibility | Merge Health Checks | Check Policy Constraints | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Check Policy Constraints | IF | Decide policy routing vs direct | Score & Rank Models | Apply Policy Routing; Route to Selected Model | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Apply Policy Routing | Code | Filter models by compliance/residency | Check Policy Constraints (true) | Route to Selected Model | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Route to Selected Model | Switch | Route to provider-specific rewrite | Check Policy Constraints (false); Apply Policy Routing; Adaptive Retry Delay | Rewrite Prompt for Azure; Rewrite Prompt for AWS; Rewrite Prompt for Google; Rewrite Prompt for Local | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Rewrite Prompt for Azure | Code | Build Azure chat payload | Route to Selected Model (Azure) | Call Azure OpenAI | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Rewrite Prompt for AWS | Code | Build Bedrock payload | Route to Selected Model (AWS) | Call AWS Bedrock | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Rewrite Prompt for Google | Code | Build Vertex payload + safety | Route to Selected Model (Google) | Call Google Vertex AI | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Rewrite Prompt for Local | Code | Build local model payload | Route to Selected Model (Local) | Call Local Model | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Call Azure OpenAI | HTTP Request | Invoke Azure LLM endpoint | Rewrite Prompt for Azure | Handle Circuit Breaker; Merge Model Responses | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Call AWS Bedrock | HTTP Request | Invoke Bedrock endpoint | Rewrite Prompt for AWS | Handle Circuit Breaker; Merge Model Responses | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Call Google Vertex AI | HTTP Request | Invoke Vertex endpoint | Rewrite Prompt for Google | Handle Circuit Breaker | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Call Local Model | HTTP Request | Invoke local model endpoint | Rewrite Prompt for Local | Handle Circuit Breaker | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Handle Circuit Breaker | Code | Retry/fallback decisioning | Call Azure OpenAI; Call AWS Bedrock; Call Google Vertex AI; Call Local Model | Check Retry Needed | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Check Retry Needed | IF | Gate exponential backoff | Handle Circuit Breaker | Adaptive Retry Delay; Merge Model Responses | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Adaptive Retry Delay | Wait | Exponential backoff delay | Check Retry Needed (true) | Route to Selected Model | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Merge Model Responses | Merge | Combine response streams | Call Azure OpenAI; Call AWS Bedrock; Check Retry Needed (false) | Normalize Output | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Normalize Output | Code | Canonicalize provider responses | Merge Model Responses | Calculate Telemetry Metrics | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Calculate Telemetry Metrics | Code | Compute tokens/latency/cost | Normalize Output | Assess Hallucination Risk | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Assess Hallucination Risk | Code | Heuristic hallucination scoring | Calculate Telemetry Metrics | Calculate Confidence Score | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Calculate Confidence Score | Code | Combine quality signals into confidence | Assess Hallucination Risk | Log Telemetry to Database; Store Result in Datastore; Detect Anomalies | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |
| Log Telemetry to Database | Postgres | Persist telemetry rows | Calculate Confidence Score | Generate Cost Report Data; Generate Performance Report Data; Generate Governance Report Data | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Store Result in Datastore | Postgres | Persist prompt/response rows | Calculate Confidence Score | Prepare Response | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Detect Anomalies | Code | Z-score anomaly detection | Calculate Confidence Score | Check Anomaly Threshold | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Check Anomaly Threshold | IF | Decide whether to alert | Detect Anomalies | Send Anomaly Alert | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Send Anomaly Alert | Slack | Alert channel on anomalies | Check Anomaly Threshold | — | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Generate Cost Report Data | Postgres | Query cost aggregates | Log Telemetry to Database | Merge Report Data | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Generate Performance Report Data | Postgres | Query performance aggregates | Log Telemetry to Database | Merge Report Data | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Generate Governance Report Data | Postgres | Query governance aggregates | Log Telemetry to Database | — (not connected) | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Merge Report Data | Merge | Combine report datasets | Generate Cost Report Data; Generate Performance Report Data | Format Report | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Format Report | Code | Build HTML report | Merge Report Data | Email Report | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Email Report | Email Send | Email the report | Format Report | — | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Prepare Response | Set | Prepare webhook response payload | Store Result in Datastore | — | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Sticky Note | Sticky Note | Documentation | — | — | ## How It Works\nThis workflow automates end-to-end research analysis by coordinating multiple AI models—including NVIDIA NIM (Llama), OpenAI GPT-4, and Claude to analyze uploaded documents, extract insights, and generate polished reports delivered via email. Built for researchers, academics, and business analysts, it enables fast, accurate synthesis of information from multiple sources. The workflow eliminates the manual burden of document review, cross-referencing, and report compilation by running parallel AI analyses, aggregating and validating model outputs, and producing structured, publication-ready documents in minutes instead of hours. Data flows from Google Sheets (user input) through document extraction, parallel AI processing, response aggregation, quality validation, structured storage in Google Sheets, automated report formatting, and final delivery via Gmail with attachments. |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Setup Steps\n1. Configure API credentials \n2. Add OpenAI API key with GPT-4 access enabled\n3. Connect Anthropic Claude API credentials\n4. Set up Google Sheets integration with read/write permissions\n5. Configure Gmail credentials with OAuth2 authentication for automated email\n6. Customize email templates and report formatting preferences |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Prerequisites\nNVIDIA NIM API access, OpenAI API key (GPT-4 enabled), Anthropic Claude API key\n## Use Cases\nAcademic literature reviews, competitive intelligence reports\n## Customization\nAdjust AI model parameters (temperature, tokens) per analysis depth needs\n## Benefits\nReduces research analysis time by 80%, eliminates single-source bias through multi-model consensus |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Automated Report Formatting & Distribution\n**Why:** Generates professional HTML/PDF reports with structured sections and delivers via Gmail, providing immediate, shareable documentation. |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Parallel Multi-Model Analysis\n**Why:** Leverages diverse AI model strengths (NVIDIA for speed, GPT-4 for reasoning, Claude for nuance) to generate comprehensive, balanced insights and reduce single-model bias. |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## Document Ingestion & Preprocessing\n**Why:** Extracts text from uploaded PDFs/documents and structures data for AI consumption, ensuring consistent input format across all processing models. |
| Sticky Note6 | Sticky Note | Documentation | — | — | ## Response Aggregation & Validation\n**Why:** Combines model outputs, identifies consensus findings, flags discrepancies, and ensures accuracy before final report compilation. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook trigger**
   1. Add **Webhook** node named **LLM Request Webhook**
   2. Method: `POST`
   3. Path: `llm-orchestration`
   4. Response mode: `Last Node`

2) **Add configuration Set node**
   1. Add **Set** node named **Workflow Configuration**
   2. Enable “Keep Only Set” = false (include other fields)
   3. Add string fields (fill with real URLs):
      - `azureHealthEndpoint`, `awsHealthEndpoint`, `googleHealthEndpoint`, `localHealthEndpoint`
      - `azureApiEndpoint`, `awsApiEndpoint`, `googleApiEndpoint`, `localApiEndpoint`
   4. Add numeric fields:
      - `maxRetries`=3, `retryBaseDelay`=1000, `circuitBreakerThreshold`=5
      - `anomalyThreshold`=0.8, `costCeilingDefault`=1, `latencySLODefault`=5000
   5. Connect **LLM Request Webhook → Workflow Configuration**

3) **Parse and validate request**
   1. Add **Code** node **Parse Request & Validate**
   2. Paste the validation JS logic (adapt if you want hard failure on invalid input)
   3. Connect **Workflow Configuration → Parse Request & Validate**

4) **Compute prompt complexity**
   1. Add **Code** node **Analyze Prompt Complexity**
   2. Paste complexity JS logic
   3. Connect **Parse Request & Validate → Analyze Prompt Complexity**

5) **Health checks**
   1. Add **HTTP Request** nodes:
      - **Check Azure OpenAI Health** (URL from `Workflow Configuration.azureHealthEndpoint`, timeout 3000, JSON response, optional allowUnauthorizedCerts)
      - **Check AWS Bedrock Health** (URL from `Workflow Configuration.awsHealthEndpoint`, timeout 3000, JSON response)
   2. Connect **Analyze Prompt Complexity → (both health nodes)**

6) **Merge health checks**
   1. Add **Merge** node **Merge Health Checks**
   2. Mode: `Combine`, “Combine By” = `Combine All`
   3. Connect both health nodes into the Merge inputs

7) **Model ranking**
   1. Add **Code** node **Score & Rank Models**
   2. Paste ranking JS logic
   3. Connect **Merge Health Checks → Score & Rank Models**
   4. (Recommended fix when rebuilding) Insert mapping Code nodes so health outputs become `{type:'health_check', model:'azure_openai', status:'healthy'}` etc, and pass config/prompt complexity into the scorer in the expected shape.

8) **Policy routing decision**
   1. Add **IF** node **Check Policy Constraints**
   2. Add conditions as needed (data residency/compliance/sensitive data)
   3. Connect **Score & Rank Models → Check Policy Constraints**

9) **Policy routing application**
   1. Add **Code** node **Apply Policy Routing** (optional path)
   2. Connect **Check Policy Constraints (true) → Apply Policy Routing**

10) **Provider switch**
   1. Add **Switch** node **Route to Selected Model**
   2. Add rules for Azure/AWS/Google/Local based on a single field (e.g., `provider`)
   3. Connect:
      - **Check Policy Constraints (false) → Route to Selected Model**
      - **Apply Policy Routing → Route to Selected Model**
   4. (Recommended fix) Ensure upstream sets a stable field like `provider: 'azure'|'aws_bedrock'|'google'|'local'`.

11) **Prompt rewrite nodes**
   1. Add **Code** nodes:
      - **Rewrite Prompt for Azure** (outputs `azureRequest`)
      - **Rewrite Prompt for AWS** (outputs `bedrockRequestBody`)
      - **Rewrite Prompt for Google** (outputs `vertexPayload`)
      - **Rewrite Prompt for Local** (outputs `localModelRequest`)
   2. Connect each output of **Route to Selected Model** to its rewrite node.

12) **Provider call nodes**
   1. Add **HTTP Request** nodes:
      - **Call Azure OpenAI** (POST, URL azureApiEndpoint)
      - **Call AWS Bedrock** (POST, URL awsApiEndpoint)
      - **Call Google Vertex AI** (POST, URL googleApiEndpoint)
      - **Call Local Model** (POST, URL localApiEndpoint)
   2. Configure each to send JSON body.
   3. Credentials:
      - For Azure/AWS/Google if using header-based keys: configure **HTTP Header Auth** credentials and reference it.
      - Vertex AI typically uses OAuth2/JWT rather than static header keys.
   4. Connect rewrite nodes to their call nodes.
   5. (Recommended fix) Set body to the right field (`azureRequest`, `bedrockRequestBody`, `vertexPayload`, `localModelRequest`) rather than a generic `requestBody`.

13) **Circuit breaker and retry**
   1. Add **Code** node **Handle Circuit Breaker**
   2. Add **IF** node **Check Retry Needed**
   3. Add **Wait** node **Adaptive Retry Delay** with exponential delay expression
   4. Connect:
      - Each Call node → Handle Circuit Breaker
      - Handle Circuit Breaker → Check Retry Needed
      - Check Retry Needed (true) → Adaptive Retry Delay → Route to Selected Model
      - Check Retry Needed (false) → Merge Model Responses

14) **Merge and normalize**
   1. Add **Merge** node **Merge Model Responses** (combineAll)
   2. Wire provider calls to it if you expect parallel multi-model output
   3. Add **Code** node **Normalize Output** and connect from merge

15) **Telemetry, hallucination risk, confidence**
   1. Add **Code** nodes:
      - **Calculate Telemetry Metrics**
      - **Assess Hallucination Risk**
      - **Calculate Confidence Score**
   2. Connect in sequence: Normalize → Telemetry → Hallucination → Confidence
   3. (Recommended fix) Align field names: use normalized `text` as response input, and use `hallucinationRisk.score` in confidence calculation.

16) **Database persistence (Postgres)**
   1. Add **Postgres** node **Log Telemetry to Database**
      - Configure Postgres credentials
      - Table: `public.llm_telemetry`
      - Map columns to your actual JSON fields
   2. Add **Postgres** node **Store Result in Datastore**
      - Table: `public.llm_results`
   3. Connect **Calculate Confidence Score → (both Postgres nodes)**

17) **Anomaly detection + Slack**
   1. Add **Code** node **Detect Anomalies** and connect from Confidence
   2. Add **IF** node **Check Anomaly Threshold**
   3. Add **Slack** node **Send Anomaly Alert**
      - Configure Slack OAuth2 credential
      - Set channelId
   4. Connect Detect → IF → Slack
   5. (Recommended fix) Compare severity properly (e.g., map `maxSeverity` to numeric).

18) **Report queries**
   1. Add Postgres executeQuery nodes:
      - **Generate Cost Report Data**
      - **Generate Performance Report Data**
      - **Generate Governance Report Data**
   2. Connect from **Log Telemetry to Database** (or from a separate scheduled trigger if you want periodic reports).

19) **Merge report datasets**
   1. Add **Merge** node **Merge Report Data** (combineAll)
   2. Connect all three report data nodes into the merge.

20) **Format and email report**
   1. Add **Code** node **Format Report** to build HTML (ensure it consumes actual query results)
   2. Add **Email Send** node **Email Report**
      - Configure SMTP/Gmail credentials
      - Set To/From
      - Set HTML field to the output of Format Report (ensure field name matches)
   3. Connect Merge Report Data → Format Report → Email Report

21) **Final webhook response**
   1. Add **Set** node **Prepare Response**
   2. Ensure it is the last node on the webhook execution path if you want the webhook to return this payload.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works” overview describing multi-model research analysis, aggregation, validation, and email delivery | Sticky Note content (high-level intent) |
| Setup steps mention OpenAI (GPT-4), Anthropic Claude, Google Sheets, Gmail OAuth2 | Sticky Note1 (note: current workflow uses Postgres + Email Send; no Google Sheets nodes exist in this JSON) |
| Prerequisites mention NVIDIA NIM, OpenAI, Anthropic Claude | Sticky Note2 (note: the workflow currently models “local model” but does not include an explicit NVIDIA NIM node) |
| Automated report formatting & distribution rationale | Sticky Note3 |
| Parallel multi-model analysis rationale | Sticky Note4 |
| Document ingestion & preprocessing rationale | Sticky Note5 (note: no PDF/doc extraction nodes exist in this JSON) |
| Response aggregation & validation rationale | Sticky Note6 |

**Important consistency note:** Several nodes reference fields that are not produced by upstream nodes (e.g., `requestBody`, `selectedModel`, `tenant_id`, `reportHtml`, anomaly severity fields). To make the workflow fully operational, you will need to add small “mapping” Set/Code nodes between blocks (or adjust the existing code) so naming and data shapes match end-to-end.