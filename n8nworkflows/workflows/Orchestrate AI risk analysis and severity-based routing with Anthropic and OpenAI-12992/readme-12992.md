Orchestrate AI risk analysis and severity-based routing with Anthropic and OpenAI

https://n8nworkflows.xyz/workflows/orchestrate-ai-risk-analysis-and-severity-based-routing-with-anthropic-and-openai-12992


# Orchestrate AI risk analysis and severity-based routing with Anthropic and OpenAI

## 1. Workflow Overview

**Purpose:** This workflow accepts engineering/data-engineering related payloads via a webhook, orchestrates a multi-agent AI analysis (Anthropic as the main orchestrator plus OpenAI-powered specialist tools), enforces a **structured JSON output**, computes a **composite risk score**, enriches/flags findings based on **severity**, merges results into a final report, and notifies downstream systems via **two HTTP calls** before responding to the original webhook.

**Target use cases (from sticky notes + node intent):**
- ETL pipeline quality monitoring
- Data anomaly detection / dataset validation before production deployment
- Engineering risk assessment with compliance verification and severity-based escalation

### 1.1 Webhook Intake and Configuration
Receives POST requests and attaches workflow-level configuration values (thresholds, standards, limits).

### 1.2 Multi-Agent AI Orchestration (Anthropic + OpenAI tools)
Anthropic “Engineering Orchestration Agent” produces a structured risk assessment and can invoke specialist tools:
- Compliance verification tool (OpenAI model)
- Risk analysis tool (OpenAI model)

### 1.3 Risk Scoring + Contextual Enrichment
Computes a composite risk score, optionally fetches historical context, then routes by severity (critical/high/medium-low) with extra enrichment for critical and flags for high.

### 1.4 Aggregation, Reporting, Downstream Notifications, Webhook Response
Merges the routed results, aggregates issues, prepares a final report, rate-limits, posts to two downstream systems, and responds to the original webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Intake and AI Orchestration Deployment
**Overview:** Accepts incoming requests and injects configuration defaults used by later scoring/routing and by the orchestration prompt context.

**Nodes involved:**
- Webhook Trigger
- Workflow Configuration

#### Node: Webhook Trigger
- **Type / role:** `n8n-nodes-base.webhook` — entry point for external systems.
- **Key config:**
  - **Path:** `engineering-orchestration`
  - **Method:** `POST`
  - **Response mode:** `lastNode` (response is produced by the final node path).
- **I/O:**
  - **Output →** Workflow Configuration
- **Edge cases / failures:**
  - Incorrect HTTP method/path → 404/405.
  - Large payloads can hit n8n limits; consider request size and execution timeouts.
  - If downstream nodes error, the webhook caller may receive an error rather than a success response (depending on n8n error handling settings).

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — adds configuration fields while keeping original payload.
- **Config choices:**
  - Adds:
    - `max_http_requests` (number) = `2`
    - `compliance_standards` (array) = `["ISO 9001","ISO 14001","OSHA","CE Marking"]`
    - `severity_threshold` (string) = `"medium"` *(note: not directly used by the Switch node; see routing block)*
    - `confidence_threshold` (number) = `0.7` *(also not directly enforced later in logic)*
  - `includeOtherFields: true` preserves incoming webhook payload fields.
- **I/O:**
  - **Input ←** Webhook Trigger
  - **Output →** Engineering Orchestration Agent
- **Edge cases:**
  - If incoming payload already has fields with the same names, they will be overwritten by this node’s assignments (could be desirable or not).

---

### Block 2 — Multi-Agent Parallel Analysis with Risk Assessment (Anthropic orchestrator + OpenAI tools)
**Overview:** The main agent (Anthropic) receives the engineering data plus config context, must emit deterministic, audit-ready structured JSON, and can call specialist “tools” for risk and compliance sub-analysis (each powered by OpenAI).

**Nodes involved:**
- Engineering Orchestration Agent
- Anthropic Chat Model
- Structured Output Parser
- Compliance Verification Agent Tool
- OpenAI Chat Model - Compliance
- Risk Analysis Agent Tool
- OpenAI Chat Model - Risk

#### Node: Engineering Orchestration Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central orchestrator that produces the core analysis.
- **Configuration choices:**
  - **Prompt text:** Embeds:
    - `Engineering Data: {{ JSON.stringify($json, null, 2) }}`
    - Config echoes: max requests, standards, thresholds
  - **System message:** Strong governance constraints:
    - No unauthorized engineering decisions; analysis only
    - Deterministic/audit-ready; traceability
    - Must output JSON matching the output parser schema
  - **Tools enabled:** Receives tools via `ai_tool` connections (Compliance + Risk).
  - **Output parser enabled:** `hasOutputParser: true` and connected to Structured Output Parser.
- **I/O:**
  - **Main input ←** Workflow Configuration
  - **Main output →** Calculate Risk Score
  - **ai_languageModel ←** Anthropic Chat Model
  - **ai_outputParser ←** Structured Output Parser
  - **ai_tool ←** Compliance Verification Agent Tool, Risk Analysis Agent Tool
- **Edge cases / failures:**
  - If the model returns non-conforming JSON, the parser may fail and halt execution.
  - Tool invocation may fail (OpenAI auth/rate limits), causing incomplete analysis or agent failure depending on n8n agent behavior.
  - “Deterministic” is a policy requirement in the prompt, but model outputs can still vary unless temperature/seed controls are set (not shown here).

#### Node: Anthropic Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — language model backend for the orchestrator.
- **Key config:**
  - Model: `claude-sonnet-4-5-20250929` (as selected in node).
  - Credentials: Anthropic API credential “Anthropic account”.
- **I/O:**
  - **ai_languageModel →** Engineering Orchestration Agent
- **Edge cases / failures:**
  - Invalid/expired Anthropic key, quota exhaustion, regional restrictions.
  - Model name availability changes can break executions if the model is deprecated.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces schema-conformant JSON.
- **Key config:**
  - Manual JSON Schema defining fields such as:
    - `entity_id` (string)
    - `workflow_state` enum: `design_validation | prototype_testing | risk_assessment | compliance_verification | operational_optimization | mitigation_tracking`
    - `risk_type` enum: technical/performance/safety/regulatory/resource/scheduling/maintenance
    - `severity` enum: `critical|high|medium|low`
    - `priority` 1–10
    - `confidence_score` 0–1
    - plus recommended action, data_sources, mitigation_status, etc.
- **I/O:**
  - **ai_outputParser →** Engineering Orchestration Agent
- **Edge cases / failures:**
  - Missing required fields are not explicitly marked “required” in the schema, but enum/type mismatches can still cause parse failures.
  - If agent returns `severity` outside enum (e.g., “sev1”), routing will not match any rule later.

#### Node: Compliance Verification Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool the orchestrator can call for compliance checks.
- **Key config:**
  - Tool prompt: `Verify compliance for: {{ $fromAI("compliance_check_data") }}`
  - System message: compliance specialist against ISO 9001/14001, OSHA, CE.
  - Description: “Verifies compliance with industry standards and regulations”
- **Model binding:** Connected to OpenAI Chat Model - Compliance (via `ai_languageModel`).
- **I/O:**
  - **ai_tool →** Engineering Orchestration Agent
  - **ai_languageModel ←** OpenAI Chat Model - Compliance
- **Edge cases:**
  - If `$fromAI("compliance_check_data")` is not provided by the orchestrator, tool calls may be ill-formed or empty.

#### Node: OpenAI Chat Model - Compliance
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM backend for compliance tool.
- **Key config:** model `gpt-4.1-mini`, OpenAI credential “OpenAi account”.
- **I/O:** **ai_languageModel →** Compliance Verification Agent Tool
- **Edge cases:** OpenAI rate limits, invalid key, model access restrictions.

#### Node: Risk Analysis Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable for deeper risk analysis.
- **Key config:**
  - Prompt: `Analyze risks for: {{ $fromAI("risk_analysis_data") }}`
  - System message: risk specialist; probability/impact; mitigation strategies.
  - Description: “Performs detailed risk analysis on engineering data”
- **Model binding:** Connected to OpenAI Chat Model - Risk.
- **I/O:**
  - **ai_tool →** Engineering Orchestration Agent
  - **ai_languageModel ←** OpenAI Chat Model - Risk
- **Edge cases:** Same `$fromAI(...)` input availability + OpenAI API reliability constraints.

#### Node: OpenAI Chat Model - Risk
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Key config:** model `gpt-4.1-mini`, OpenAI credential “OpenAi account”.
- **I/O:** **ai_languageModel →** Risk Analysis Agent Tool

---

### Block 3 — Risk Scoring + Historical Context + Severity Routing
**Overview:** Takes the orchestrator’s structured output, computes a composite risk score, retrieves historical data for context, and routes items by severity into different processing paths.

**Nodes involved:**
- Calculate Risk Score
- Fetch Historical Data
- Route by Severity
- Enrich Critical Issues
- Process High Severity

#### Node: Calculate Risk Score
- **Type / role:** `n8n-nodes-base.code` — computes numeric scoring and escalation flag.
- **Logic highlights:**
  - Maps severity to numeric: critical=4, high=3, medium=2, low=1.
  - `compositeRiskScore = severityValue * priority * confidenceScore`
  - Risk category thresholds:
    - >=10 → critical
    - >=6 → high
    - >=3 → medium
    - else low
  - `escalation_required = compositeRiskScore >= 6 || severity === 'critical'`
  - Adds fields:
    - `composite_risk_score` (rounded to 2 decimals)
    - `risk_category`
    - `escalation_required`
- **I/O:**
  - **Input ←** Engineering Orchestration Agent
  - **Output →** Fetch Historical Data
- **Edge cases / failures:**
  - If `confidence_score` is missing, defaults to 0.5; if `priority` missing, defaults to 1; may under/over-estimate.
  - If `confidence_score` is a string, multiplication may yield `NaN` (depending on JS coercion and content).
  - Severity casing handled via `toLowerCase()`, but only after reading `severity`; if severity is non-string, `toLowerCase()` will throw.

#### Node: Fetch Historical Data
- **Type / role:** `n8n-nodes-base.httpRequest` — pulls historical context from an external API.
- **Key config:**
  - URL is a placeholder: `<__PLACEHOLDER_VALUE__Historical data API endpoint URL__>`
  - Query parameters:
    - `entity_id = {{ $json.entity_id }}`
    - `lookback_days = 90`
  - Header: `Authorization = <token placeholder>`
- **I/O:**
  - **Input ←** Calculate Risk Score
  - **Output →** Route by Severity
- **Edge cases / failures:**
  - Placeholder URL/token must be replaced; otherwise node fails.
  - If historical API returns a different shape, it may overwrite fields expected downstream (because it becomes the new `$json` unless “Response Format/Keep Input Data” options are used—none shown).
  - Timeouts/rate limits can disrupt severity routing.

#### Node: Route by Severity
- **Type / role:** `n8n-nodes-base.switch` — routes by `severity`.
- **Rules / outputs:**
  - Output “Critical” if `{{$json.severity}} == "critical"`
  - Output “High” if `{{$json.severity}} == "high"`
  - Output “Medium/Low” if severity is “medium” OR “low”
  - Each rule uses strict type validation.
- **Connections:**
  - **Critical →** Enrich Critical Issues
  - **High →** Process High Severity
  - **Medium/Low →** Merge Analysis Results (input index 2)
- **Edge cases:**
  - If severity is absent or not in the expected enum, the item may produce **no output** from the switch (data effectively dropped).
  - Note: `severity_threshold` from config is not used here; routing is hard-coded to explicit severities.

#### Node: Enrich Critical Issues
- **Type / role:** `n8n-nodes-base.code` — adds escalation metadata and executive summary for critical items.
- **Logic highlights:**
  - Creates `executive_summary` string including risk_type, workflow_state, severity, priority, confidence % and responsible team.
  - Builds `immediate_actions_required` array from `observed_issues` and `recommended_action`; falls back to default emergency steps.
  - Adds `escalation_timestamp`, `critical_alert_sent: true`.
- **I/O:**
  - **Input ←** Route by Severity (Critical)
  - **Output →** Merge Analysis Results (input index 0)
- **Edge cases:**
  - If `confidence_score` missing, `(item.confidence_score * 100)` yields `NaN%` in summary; consider defaulting.
  - If `observed_issues` is not an array, it will be skipped and fallback actions may be used.

#### Node: Process High Severity
- **Type / role:** `n8n-nodes-base.set` — flags high severity items for expedited review.
- **Config:**
  - `high_priority_flag = true`
  - `review_deadline = {{ $now.plus({ days: 3 }).toISO() }}`
  - `assigned_reviewer = "Senior Engineering Team"`
  - `includeOtherFields: true`
- **I/O:**
  - **Input ←** Route by Severity (High)
  - **Output →** Merge Analysis Results (input index 1)
- **Edge cases:**
  - None major; relies on $now time math supported by n8n expressions.

---

### Block 4 — Severity-Based Routing and Report Generation + Downstream Notifications
**Overview:** Merges outputs from severity paths, aggregates issues, prepares a final report, waits to limit API calls, posts results to two systems, and responds to the webhook with a consolidated payload.

**Nodes involved:**
- Merge Analysis Results
- Aggregate Issues
- Prepare Final Report
- Wait for Rate Limit
- HTTP Request 1
- HTTP Request 2
- Respond to Webhook

#### Node: Merge Analysis Results
- **Type / role:** `n8n-nodes-base.merge` — combines up to 3 input streams.
- **Config:** `numberInputs: 3`
- **Inputs:**
  - Input 0 ← Enrich Critical Issues
  - Input 1 ← Process High Severity
  - Input 2 ← Route by Severity (Medium/Low)
- **Output →** Aggregate Issues
- **Edge cases:**
  - Merge behavior depends on n8n Merge node mode (not explicitly shown). With multiple streams, timing/empty branches can lead to partial merges.
  - If only one branch outputs items, ensure the merge node’s behavior still passes them through as intended.

#### Node: Aggregate Issues
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates all item data.
- **Config:**
  - Mode: `aggregateAllItemData`
  - Destination field: `observed_issues`
- **I/O:**
  - **Input ←** Merge Analysis Results
  - **Output →** Prepare Final Report
- **Edge cases:**
  - This node will reshape the data into an aggregated structure; ensure downstream expects aggregated format.
  - Destination field name `observed_issues` can conflict with existing `observed_issues` arrays (it will become the aggregate output container).

#### Node: Prepare Final Report
- **Type / role:** `n8n-nodes-base.set` — adds report metadata.
- **Fields added:**
  - `report_generated_at = {{ $now.toISO() }}`
  - `total_issues_count = {{ $json.observed_issues ? $json.observed_issues.length : 0 }}`
  - `workflow_execution_id = {{ $execution.id }}`
  - `report_version = "1.0"`
  - Keeps other fields.
- **I/O:** **Output →** Wait for Rate Limit
- **Edge cases:** If `observed_issues` becomes an object rather than array after aggregation, `.length` may be undefined; count may be wrong.

#### Node: Wait for Rate Limit
- **Type / role:** `n8n-nodes-base.wait` — delays flow (basic rate limiting).
- **Config:** `amount: 2` (seconds by default).
- **I/O:** **Output →** HTTP Request 1
- **Edge cases:** Wait node can increase execution duration; watch n8n execution time limits.

#### Node: HTTP Request 1
- **Type / role:** `n8n-nodes-base.httpRequest` — notifies first downstream system.
- **Config:**
  - POST to placeholder URL
  - JSON body: `{{ $json }}` (whatever comes from Wait/Prepare Final Report path)
  - Headers: `Content-Type: application/json`, `Authorization: <placeholder>`
- **I/O:** **Output →** HTTP Request 2
- **Edge cases:**
  - Placeholder URL/token must be replaced.
  - If downstream returns non-JSON or errors, subsequent expression references may fail.

#### Node: HTTP Request 2
- **Type / role:** `n8n-nodes-base.httpRequest` — notifies second downstream system.
- **Config:**
  - JSON body: `{{ $('Prepare Final Report').item.json }}`
  - Headers include Authorization placeholder.
- **I/O:** **Output →** Respond to Webhook
- **Edge cases:**
  - Cross-node reference `$('Prepare Final Report').item.json` assumes that node executed and has an item in scope.
  - If there are multiple items, `.item` picks the current item context; ensure it matches expected (n8n can behave unexpectedly with multiple items).

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — sends final HTTP response back to webhook caller.
- **Config:**
  - Respond with JSON using expression:
    - Includes `final_report` from Prepare Final Report
    - `analysis_result` from Engineering Orchestration Agent
    - `http_request_1_status` uses `$('HTTP Request 1').item.json.statusCode || "completed"`
    - `http_request_2_status` similarly
    - `timestamp: $now.toISO()`
    - `execution_id: $execution.id`
- **Edge cases:**
  - Many HTTP Request nodes place status code in `$json.statusCode`, but response shape varies by configuration; expression may yield undefined.
  - If HTTP Request nodes fail and stop execution, response is never returned (unless you enable “Continue On Fail” on those nodes or workflow-level error handling).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Entry point (POST webhook) | — | Workflow Configuration | ## Webhook Intake and AI Orchestration Deployment<br>**Why:** Receives data processing requests containing dataset references and analysis requirements, immediately deploys Orchestrating Orchestration Agent using Anthropic's Claude to intelligently determine which specialized analysis agents |
| Workflow Configuration | n8n-nodes-base.set | Inject runtime configuration defaults | Webhook Trigger | Engineering Orchestration Agent | ## Webhook Intake and AI Orchestration Deployment<br>**Why:** Receives data processing requests containing dataset references and analysis requirements, immediately deploys Orchestrating Orchestration Agent using Anthropic's Claude to intelligently determine which specialized analysis agents |
| Engineering Orchestration Agent | @n8n/n8n-nodes-langchain.agent | Main orchestrator agent; structured risk assessment; calls tools | Workflow Configuration | Calculate Risk Score | ## Webhook Intake and AI Orchestration Deployment<br>**Why:** Receives data processing requests containing dataset references and analysis requirements, immediately deploys Orchestrating Orchestration Agent using Anthropic's Claude to intelligently determine which specialized analysis agents |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for orchestrator | — (AI model link) | Engineering Orchestration Agent (ai_languageModel) | ## Webhook Intake and AI Orchestration Deployment<br>**Why:** Receives data processing requests containing dataset references and analysis requirements, immediately deploys Orchestrating Orchestration Agent using Anthropic's Claude to intelligently determine which specialized analysis agents |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema output | — (parser link) | Engineering Orchestration Agent (ai_outputParser) | ## Webhook Intake and AI Orchestration Deployment<br>**Why:** Receives data processing requests containing dataset references and analysis requirements, immediately deploys Orchestrating Orchestration Agent using Anthropic's Claude to intelligently determine which specialized analysis agents |
| Compliance Verification Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Specialist compliance tool callable by agent | — (tool link) | Engineering Orchestration Agent (ai_tool) | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| OpenAI Chat Model - Compliance | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for compliance tool | — (AI model link) | Compliance Verification Agent Tool (ai_languageModel) | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Risk Analysis Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Specialist risk tool callable by agent | — (tool link) | Engineering Orchestration Agent (ai_tool) | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| OpenAI Chat Model - Risk | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for risk tool | — (AI model link) | Risk Analysis Agent Tool (ai_languageModel) | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Calculate Risk Score | n8n-nodes-base.code | Computes composite risk score/category/escalation | Engineering Orchestration Agent | Fetch Historical Data | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Fetch Historical Data | n8n-nodes-base.httpRequest | Pulls historical context from external API | Calculate Risk Score | Route by Severity | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Route by Severity | n8n-nodes-base.switch | Routes findings into severity paths | Fetch Historical Data | Enrich Critical Issues; Process High Severity; Merge Analysis Results | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Enrich Critical Issues | n8n-nodes-base.code | Adds executive summary + immediate actions | Route by Severity (Critical) | Merge Analysis Results | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Process High Severity | n8n-nodes-base.set | Flags high severity items, sets deadline/reviewer | Route by Severity (High) | Merge Analysis Results | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Merge Analysis Results | n8n-nodes-base.merge | Combines critical/high/medium-low branches | Enrich Critical Issues; Process High Severity; Route by Severity (Medium/Low) | Aggregate Issues | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| Aggregate Issues | n8n-nodes-base.aggregate | Aggregates item data into a consolidated field | Merge Analysis Results | Prepare Final Report | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| Prepare Final Report | n8n-nodes-base.set | Adds reporting metadata (time/count/version) | Aggregate Issues | Wait for Rate Limit | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| Wait for Rate Limit | n8n-nodes-base.wait | Simple delay before outbound notifications | Prepare Final Report | HTTP Request 1 | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| HTTP Request 1 | n8n-nodes-base.httpRequest | Notify downstream system #1 | Wait for Rate Limit | HTTP Request 2 | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| HTTP Request 2 | n8n-nodes-base.httpRequest | Notify downstream system #2 | HTTP Request 1 | Respond to Webhook | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns consolidated response to caller | HTTP Request 2 | — | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block | — | — | ## Prerequisites<br>Active Anthropic and OpenAI API accounts, data processing system with webhook capability<br>## Use Cases<br>ETL pipeline quality monitoring, data anomaly detection, dataset validation before production deployment<br>## Customization<br>Modify orchestration agent logic for custom analysis pathways<br>## Benefits<br>Accelerates data quality assessment by 70%, enables proactive issue detection before production impact |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Setup Steps<br>1. Configure webhook trigger endpoint for data processing system integration<br>2. Set up Anthropic API credentials for Orchestrating Orchestration Agent node<br>3. Configure specialized agent tools <br>4. Update Calculate Risk Score node with your risk scoring methodology<br>5. Set up Fetch Historical Data node with data warehouse API credentials<br>6. Configure severity threshold in Route by Severity node for alert triggering<br>7. Connect HTTP Request nodes with stakeholder notification endpoints |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block | — | — | ## How It Works<br>This workflow automates complex data engineering operations by orchestrating multiple specialized AI agents to analyze datasets, calculate risk metrics, and route findings based on severity levels. Designed for data engineers, analytics teams, and business intelligence managers, it solves the challenge of processing diverse datasets through appropriate analytical frameworks while ensuring critical insights reach stakeholders immediately. The system receives data processing requests via webhook, deploys an orchestration agent that determines which specialized analysis agents to invoke (Anthropic Chat Model for general analysis, Risk Analysis Verification Agent, and Test Validation Agent), calculates risk scores, fetches relevant historical context, then routes results by severity. High-severity findings trigger immediate HTTP notifications to stakeholders, while all results are aggregated into comprehensive reports, formatted for clarity, and logged with appropriate priority markers before webhook response. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Severity-Based Routing and Report Generation<br>**Why:** Routes high-severity findings through immediate HTTP notifications for rapid stakeholder response, merges all analysis results, aggregates issues by category, formats comprehensive task reports with structured output parsing, waits for stakeholder acknowledgment, logs prioritized findings before webhook response. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Multi-Agent Parallel Analysis with Risk Assessment<br>**Why:** Executes specialized agents (Anthropic Chat Model, Risk Analysis Verification, Test Validation) equipped with OpenAI models for domain-specific analysis, calculates risk scores using defined algorithms, fetches historical data for trend context, ensuring comprehensive multi-dimensional dataset evaluation. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment block | — | — | ## Webhook Intake and AI Orchestration Deployment<br>**Why:** Receives data processing requests containing dataset references and analysis requirements, immediately deploys Orchestrating Orchestration Agent using Anthropic's Claude to intelligently determine which specialized analysis agents |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `AI Data Engineering Orchestrator with Multi-Agent and Severity-Based Routing` (or your preferred name).

2. **Add “Webhook Trigger”** (`Webhook` node):
   - HTTP Method: `POST`
   - Path: `engineering-orchestration`
   - Response Mode: `Last Node`
   - Save and note the production URL for your caller system.

3. **Add “Workflow Configuration”** (`Set` node) and connect:
   - Connect: **Webhook Trigger → Workflow Configuration**
   - Add fields (and keep other fields):
     - `max_http_requests` (Number) = `2`
     - `compliance_standards` (Array) = `["ISO 9001","ISO 14001","OSHA","CE Marking"]`
     - `severity_threshold` (String) = `medium`
     - `confidence_threshold` (Number) = `0.7`
   - Ensure: **Include Other Fields = true**

4. **Add “Anthropic Chat Model”** (`LangChain > Anthropic Chat Model`):
   - Choose model: `claude-sonnet-4-5-20250929` (or closest available)
   - Create/attach **Anthropic API credentials** (API key).
   - This node will be connected as an AI language model (not main flow).

5. **Add “OpenAI Chat Model - Compliance”** (`LangChain > OpenAI Chat Model`):
   - Model: `gpt-4.1-mini`
   - Attach **OpenAI API credentials**.

6. **Add “OpenAI Chat Model - Risk”** (another OpenAI chat model node):
   - Model: `gpt-4.1-mini`
   - Reuse the same OpenAI credentials.

7. **Add “Compliance Verification Agent Tool”** (`LangChain > Agent Tool`):
   - Tool Description: “Verifies compliance with industry standards and regulations”
   - System message: compliance specialist for ISO 9001/14001, OSHA, CE Marking
   - Tool text/prompt: `Verify compliance for: {{ $fromAI("compliance_check_data") }}`
   - Connect **OpenAI Chat Model - Compliance → Compliance Verification Agent Tool** via **ai_languageModel** connection.

8. **Add “Risk Analysis Agent Tool”** (`LangChain > Agent Tool`):
   - Tool Description: “Performs detailed risk analysis on engineering data”
   - System message: risk specialist (failures, hazards, bottlenecks, probability/impact, mitigations)
   - Tool text/prompt: `Analyze risks for: {{ $fromAI("risk_analysis_data") }}`
   - Connect **OpenAI Chat Model - Risk → Risk Analysis Agent Tool** via **ai_languageModel**.

9. **Add “Structured Output Parser”** (`LangChain > Structured Output Parser`):
   - Schema type: Manual
   - Paste the JSON Schema (object with fields: entity_id, workflow_state enum, risk_type enum, severity enum, observed_issues[], recommended_action, priority 1–10, responsible_team, predicted_impact, confidence_score 0–1, timestamp, data_sources[], mitigation_status).

10. **Add “Engineering Orchestration Agent”** (`LangChain > Agent`) and connect:
   - Main connection: **Workflow Configuration → Engineering Orchestration Agent**
   - Set the agent prompt to include engineering data and configuration (stringify JSON + echo settings).
   - Set the system message to the governance/audit constraints and instruct it to output parser-compliant JSON.
   - Connect AI components:
     - **Anthropic Chat Model → Engineering Orchestration Agent** (ai_languageModel)
     - **Structured Output Parser → Engineering Orchestration Agent** (ai_outputParser)
     - **Compliance Verification Agent Tool → Engineering Orchestration Agent** (ai_tool)
     - **Risk Analysis Agent Tool → Engineering Orchestration Agent** (ai_tool)

11. **Add “Calculate Risk Score”** (`Code` node) and connect:
   - **Engineering Orchestration Agent → Calculate Risk Score**
   - Paste the provided JS scoring logic (severity map, composite score, category, escalation_required).

12. **Add “Fetch Historical Data”** (`HTTP Request` node) and connect:
   - **Calculate Risk Score → Fetch Historical Data**
   - Configure:
     - URL: your historical API endpoint
     - Query params:
       - `entity_id = {{ $json.entity_id }}`
       - `lookback_days = 90`
     - Header: `Authorization = <your token>`
   - Consider enabling “Keep Input Data” (if available) to avoid losing the risk assessment fields.

13. **Add “Route by Severity”** (`Switch` node) and connect:
   - **Fetch Historical Data → Route by Severity**
   - Create 3 outputs/rules:
     - `Critical`: `{{ $json.severity }}` equals `critical`
     - `High`: equals `high`
     - `Medium/Low`: equals `medium` OR equals `low`

14. **Add “Enrich Critical Issues”** (`Code` node):
   - Connect: **Route by Severity (Critical) → Enrich Critical Issues**
   - Paste the enrichment JS (executive summary + immediate actions + timestamps).

15. **Add “Process High Severity”** (`Set` node):
   - Connect: **Route by Severity (High) → Process High Severity**
   - Add fields (keep others):
     - `high_priority_flag` = true
     - `review_deadline = {{ $now.plus({ days: 3 }).toISO() }}`
     - `assigned_reviewer = Senior Engineering Team`

16. **Add “Merge Analysis Results”** (`Merge` node):
   - Set number of inputs to `3`
   - Connect:
     - **Enrich Critical Issues → Merge** (Input 0)
     - **Process High Severity → Merge** (Input 1)
     - **Route by Severity (Medium/Low) → Merge** (Input 2)

17. **Add “Aggregate Issues”** (`Aggregate` node) and connect:
   - **Merge Analysis Results → Aggregate Issues**
   - Operation: aggregate all item data
   - Destination field name: `observed_issues`

18. **Add “Prepare Final Report”** (`Set` node) and connect:
   - **Aggregate Issues → Prepare Final Report**
   - Add fields (keep others):
     - `report_generated_at = {{ $now.toISO() }}`
     - `total_issues_count = {{ $json.observed_issues ? $json.observed_issues.length : 0 }}`
     - `workflow_execution_id = {{ $execution.id }}`
     - `report_version = 1.0`

19. **Add “Wait for Rate Limit”** (`Wait` node) and connect:
   - **Prepare Final Report → Wait for Rate Limit**
   - Amount: `2` (seconds)

20. **Add “HTTP Request 1”** (`HTTP Request` node) and connect:
   - **Wait for Rate Limit → HTTP Request 1**
   - POST to downstream system #1:
     - JSON body: `{{ $json }}`
     - Headers: `Content-Type: application/json`, `Authorization: <token>`

21. **Add “HTTP Request 2”** (`HTTP Request` node) and connect:
   - **HTTP Request 1 → HTTP Request 2**
   - POST to downstream system #2:
     - JSON body: `{{ $('Prepare Final Report').item.json }}`
     - Headers: same pattern

22. **Add “Respond to Webhook”** and connect:
   - **HTTP Request 2 → Respond to Webhook**
   - Respond with JSON body containing:
     - status/message
     - `final_report` from Prepare Final Report
     - `analysis_result` from Engineering Orchestration Agent
     - statuses from HTTP requests (statusCode if present)
     - timestamp + execution id

23. **Credentials checklist:**
   - Anthropic API credential configured and selected in Anthropic model node.
   - OpenAI API credential configured and selected in both OpenAI model nodes.
   - Historical API + downstream notification endpoints: replace placeholders; add required auth headers.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Active Anthropic and OpenAI API accounts, data processing system with webhook capability | Prerequisites (Sticky Note) |
| ETL pipeline quality monitoring; data anomaly detection; dataset validation before production deployment | Use cases (Sticky Note) |
| Modify orchestration agent logic for custom analysis pathways | Customization (Sticky Note) |
| “Accelerates data quality assessment by 70%, enables proactive issue detection before production impact” | Benefits claim (Sticky Note) |
| Setup steps list (webhook, credentials, tools, risk score methodology, historical API, routing threshold, HTTP notifications) | Operational guidance (Sticky Note1) |
| High-level “How It Works” description (multi-agent orchestration → scoring → historical context → severity routing → notifications → report → webhook response) | Conceptual overview (Sticky Note2) |

