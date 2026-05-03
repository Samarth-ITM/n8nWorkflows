Prioritize vehicle maintenance with Anthropic Claude using telemetry and history

https://n8nworkflows.xyz/workflows/prioritize-vehicle-maintenance-with-anthropic-claude-using-telemetry-and-history-12991


# Prioritize vehicle maintenance with Anthropic Claude using telemetry and history

## 1. Workflow Overview

**Title (user-provided):** Prioritize vehicle maintenance with Anthropic Claude using telemetry and history  
**Workflow name (JSON):** Dual-Agent Predictive Vehicle Health and Maintenance Prioritization System

This workflow automates **predictive fleet maintenance** by combining **real-time telemetry** with **historical maintenance data**, then using **two Anthropic Claude AI agents** to (1) detect anomalies and (2) prioritize maintenance actions using a Remaining Useful Life (RUL) tool. It routes results into **urgent alerts vs standard records** and generates a **compliance-oriented audit log** for every processed item.

### 1.1 Scheduled execution & runtime configuration
Runs periodically and defines API endpoints + thresholds used later.

### 1.2 Dual data acquisition (telemetry + history) and merge
Fetches both datasets in parallel, then merges them into one payload for AI analysis.

### 1.3 AI anomaly detection (Claude agent + structured output)
Claude analyzes merged data and returns structured JSON about anomalies.

### 1.4 AI maintenance prioritization (Claude agent + tool-assisted RUL + structured output)
Second agent calls an internal code tool to compute RUL/failure date, then produces a prioritized, audit-ready maintenance recommendation.

### 1.5 Urgency routing, alert formatting, and audit logging
Critical items are formatted as urgent alerts; others become standard records. Both are logged with execution metadata.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled execution & workflow configuration

**Overview:** Triggers the workflow on a schedule and sets global configuration variables (API URLs, thresholds) used by later nodes.

**Nodes Involved:**
- Schedule Trigger
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / Role:** `n8n-nodes-base.scheduleTrigger` — Entry point; periodic execution.
- **Configuration choices:** Runs on an interval measured in **minutes** (exact number not specified in JSON; it uses the “minutes” field).
- **Connections:**
  - **Output →** Workflow Configuration
- **Potential failures / edge cases:**
  - Misconfigured interval (too frequent) can overload downstream APIs/Anthropic usage quotas.
  - If workflow is inactive (`active: false`), it will not run.

#### Node: Workflow Configuration
- **Type / Role:** `n8n-nodes-base.set` — Centralized configuration constants.
- **Configuration choices (interpreted):**
  - `telemetryApiUrl`: placeholder string for a telemetry endpoint.
  - `historicalApiUrl`: placeholder string for historical maintenance endpoint.
  - `anomalyThreshold`: `0.75` (note: **not used anywhere else** in this workflow as written).
  - `criticalUrgencyLevel`: `"critical"` used by the IF node.
  - “Include other fields” enabled, so any incoming JSON is preserved.
- **Key expressions/variables:** Downstream nodes reference:
  - `$('Workflow Configuration').first().json.telemetryApiUrl`
  - `$('Workflow Configuration').first().json.historicalApiUrl`
  - `$('Workflow Configuration').first().json.criticalUrgencyLevel`
- **Connections:**
  - **Output →** Fetch Real-Time Vehicle Telemetry (parallel)
  - **Output →** Fetch Historical Vehicle Data (parallel)
- **Potential failures / edge cases:**
  - Placeholder URLs not replaced → HTTP Request nodes fail with invalid URL.
  - If you later expect `anomalyThreshold` to affect logic, you must explicitly reference it in the agent prompt or routing rules.

---

### Block 2 — Dual data acquisition & merge

**Overview:** Fetches telemetry and historical records concurrently, then merges them so the anomaly agent has full context.

**Nodes Involved:**
- Fetch Real-Time Vehicle Telemetry
- Fetch Historical Vehicle Data
- Merge Telemetry and Historical Data

#### Node: Fetch Real-Time Vehicle Telemetry
- **Type / Role:** `n8n-nodes-base.httpRequest` — Pulls current telemetry.
- **Configuration choices:**
  - URL is taken from Workflow Configuration via expression.
  - Response format: **JSON**.
- **Key expressions:**
  - `={{ $('Workflow Configuration').first().json.telemetryApiUrl }}`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Merge Telemetry and Historical Data (input index 0)
- **Potential failures / edge cases:**
  - Auth missing/incorrect (if API requires headers/OAuth/API key; not configured here).
  - Non-JSON response breaks parsing.
  - Rate limiting / timeouts.

#### Node: Fetch Historical Vehicle Data
- **Type / Role:** `n8n-nodes-base.httpRequest` — Pulls historical maintenance/degradation data.
- **Configuration choices:**
  - URL from Workflow Configuration.
  - Response format: **JSON**.
- **Key expressions:**
  - `={{ $('Workflow Configuration').first().json.historicalApiUrl }}`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Merge Telemetry and Historical Data (input index 1)
- **Potential failures / edge cases:** Same as telemetry request (auth, format, rate limits).

#### Node: Merge Telemetry and Historical Data
- **Type / Role:** `n8n-nodes-base.merge` — Combines both datasets for AI consumption.
- **Configuration choices:**
  - Mode: **Combine**
  - CombineBy: **combineAll** (produces combined items; behavior depends on item counts from each branch).
- **Connections:**
  - **Inputs ←** both HTTP Request nodes
  - **Output →** Anomaly Detection Agent
- **Potential failures / edge cases:**
  - If one branch returns **zero items**, combine behavior may produce empty output or unexpected pairings.
  - If telemetry returns multiple vehicles and historical returns multiple vehicles, `combineAll` can create many combinations (cartesian-like), inflating cost/tokens and processing time. Consider merging by `vehicle_id` instead if applicable.

---

### Block 3 — AI anomaly detection (Claude agent + structured output)

**Overview:** Uses Claude to detect anomalies from merged data and enforces a strict JSON schema using a structured output parser.

**Nodes Involved:**
- Anthropic Model - Anomaly Detection Agent
- Anomaly Detection Output Parser
- Anomaly Detection Agent

#### Node: Anthropic Model - Anomaly Detection Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — LLM backend for the agent.
- **Configuration choices:**
  - Model: `claude-3-5-sonnet-20241022`
- **Credentials:**
  - Anthropic API credential: “Anthropic account”
- **Connections:**
  - **Output (ai_languageModel) →** Anomaly Detection Agent
- **Potential failures / edge cases:**
  - Invalid/expired Anthropic credential.
  - Model availability changes or org policy restrictions.
  - Token limits if merged payload is large.

#### Node: Anomaly Detection Output Parser
- **Type / Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Validates/coerces LLM output into a JSON schema.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `vehicle_id` (string)
    - `component_id` (string)
    - `anomaly_detected` (boolean)
    - `confidence_score` (0..1)
  - Optional fields include `anomaly_type`, `anomaly_severity` (enum), `sensor_readings` (object)
- **Connections:**
  - **Output (ai_outputParser) →** Anomaly Detection Agent
- **Potential failures / edge cases:**
  - If Claude returns non-conforming JSON, parsing fails.
  - `anomaly_severity` enum enforcement can fail if model outputs unexpected labels (e.g., “severe”).

#### Node: Anomaly Detection Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates prompt + LLM + parsing.
- **Configuration choices:**
  - Prompt mode: “define”
  - `text`: `={{ $json }}` (passes the merged data object as the agent input)
  - System message instructs: correlate telemetry + history, return structured JSON.
  - `hasOutputParser`: true (wired to the structured parser node).
- **Connections:**
  - **Input ←** Merge Telemetry and Historical Data
  - **ai_languageModel ←** Anthropic Model - Anomaly Detection Agent
  - **ai_outputParser ←** Anomaly Detection Output Parser
  - **Output →** Maintenance Prioritization Agent
- **Potential failures / edge cases:**
  - If merged data is too large, LLM may truncate/omit required fields → parser failure.
  - If merged data doesn’t include `vehicle_id` / `component_id`, the agent may hallucinate values; consider enforcing presence upstream.

---

### Block 4 — AI maintenance prioritization (Claude agent + RUL tool + structured output)

**Overview:** Uses a second Claude agent to convert anomaly findings into prioritized maintenance actions, calling a code tool to compute RUL and predicted failure date.

**Nodes Involved:**
- Anthropic Model - Maintenance Prioritization Agent
- RUL Calculation Tool
- Maintenance Prioritization Output Parser
- Maintenance Prioritization Agent

#### Node: Anthropic Model - Maintenance Prioritization Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — LLM backend for prioritization agent.
- **Configuration choices:**
  - Model: `claude-3-5-sonnet-20241022`
- **Credentials:** same Anthropic account credential.
- **Connections:**
  - **Output (ai_languageModel) →** Maintenance Prioritization Agent
- **Potential failures / edge cases:** Same as the other Anthropic model node.

#### Node: RUL Calculation Tool
- **Type / Role:** `@n8n/n8n-nodes-langchain.toolCode` — A callable tool the agent can use.
- **What it does (interpreted):**
  - Reads from the tool input:
    - `anomaly_severity`, `component_id`, `confidence_score`
  - Maps severity to baseline RUL days:
    - critical: 7, high: 30, medium: 90, low: 180 (default 90)
  - Adjusts by multiplying with `confidence_score` and rounding
  - Produces:
    - `remaining_useful_life_days`
    - `predicted_failure_date` (YYYY-MM-DD)
    - `calculation_timestamp`
- **Connections:**
  - **Output (ai_tool) →** Maintenance Prioritization Agent (tool is available to the agent)
- **Potential failures / edge cases:**
  - If `confidence_score` is missing/undefined → `adjustedRUL` becomes `NaN`.
  - If `anomaly_severity` missing, defaults to 90 days (may be misleading).
  - No explicit guardrails (min/max RUL) or timezone control beyond ISO date.

#### Node: Maintenance Prioritization Output Parser
- **Type / Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces structured output for maintenance decisions.
- **Configuration choices:**
  - Manual schema requiring:
    - `vehicle_id`, `component_id`, `anomaly_detected`
    - `recommended_action` (string)
    - `urgency_level` enum: low/medium/high/critical
    - `confidence_score` (0..1)
  - Optional/expected: `predicted_failure_date` (date), `remaining_useful_life_days` (number), `maintenance_priority` (1..10)
- **Connections:**
  - **Output (ai_outputParser) →** Maintenance Prioritization Agent
- **Potential failures / edge cases:**
  - Model returns date not in `YYYY-MM-DD` → parse failure.
  - `maintenance_priority` outside 1–10 can fail validation (if provided).

#### Node: Maintenance Prioritization Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — Produces prioritized maintenance output, can call RUL tool.
- **Configuration choices:**
  - `text`: `={{ $json }}` (receives anomaly agent output)
  - System message instructs tool use + deterministic, audit-ready recommendations.
  - `hasOutputParser`: true (structured parser connected).
- **Connections:**
  - **Input ←** Anomaly Detection Agent
  - **ai_languageModel ←** Anthropic Model - Maintenance Prioritization Agent
  - **ai_tool ←** RUL Calculation Tool
  - **ai_outputParser ←** Maintenance Prioritization Output Parser
  - **Output →** Check Urgency Level
- **Potential failures / edge cases:**
  - If anomaly agent output lacks `anomaly_severity` or `confidence_score`, tool computation may degrade or error.
  - Determinism claim in prompt is not enforced technically (temperature/settings not shown); to improve reproducibility, set model options accordingly if available.

---

### Block 5 — Urgency routing, alert formatting, audit log generation

**Overview:** Routes critical findings to urgent formatting; non-critical to standard formatting; both paths feed an audit log generator.

**Nodes Involved:**
- Check Urgency Level
- Format Urgent Alert
- Format Standard Maintenance Record
- Generate Audit Log

#### Node: Check Urgency Level
- **Type / Role:** `n8n-nodes-base.if` — Branches based on urgency.
- **Configuration choices:**
  - Condition: Maintenance Prioritization Agent `urgency_level` equals Workflow Configuration `criticalUrgencyLevel` (string compare).
- **Key expressions:**
  - Left: `={{ $('Maintenance Prioritization Agent').item.json.urgency_level }}`
  - Right: `={{ $('Workflow Configuration').first().json.criticalUrgencyLevel }}`
- **Connections:**
  - **Input ←** Maintenance Prioritization Agent
  - **True →** Format Urgent Alert
  - **False →** Format Standard Maintenance Record
- **Potential failures / edge cases:**
  - If `urgency_level` is missing/null, it will route to **false** branch.
  - Case sensitivity is disabled, but unexpected values (e.g., “CRIT”) won’t match.

#### Node: Format Urgent Alert
- **Type / Role:** `n8n-nodes-base.set` — Adds urgent alert metadata.
- **Configuration choices (adds fields):**
  - `alert_type`: `URGENT`
  - `notification_required`: `true`
  - `escalation_level`: `immediate`
  - `timestamp`: `={{ $now.toISO() }}`
  - Includes other fields (keeps all maintenance agent output).
- **Connections:**
  - **Input ←** Check Urgency Level (true)
  - **Output →** Generate Audit Log
- **Potential failures / edge cases:**
  - None typical; depends on upstream data presence.

#### Node: Format Standard Maintenance Record
- **Type / Role:** `n8n-nodes-base.set` — Adds standard record metadata.
- **Configuration choices (adds fields):**
  - `alert_type`: `STANDARD`
  - `notification_required`: `false`
  - `escalation_level`: `scheduled`
  - `timestamp`: `={{ $now.toISO() }}`
  - Includes other fields.
- **Connections:**
  - **Input ←** Check Urgency Level (false)
  - **Output →** Generate Audit Log
- **Potential failures / edge cases:** Same as urgent formatting node.

#### Node: Generate Audit Log
- **Type / Role:** `n8n-nodes-base.code` — Creates compliance/audit records per item.
- **Configuration choices (logic summary):**
  - Iterates over all incoming items from either branch.
  - Generates an `audit_id` like `AUDIT-<epoch>-<random>`.
  - Writes key fields from item JSON plus:
    - `workflow_execution_id: $execution.id`
    - `workflow_name: $workflow.name`
    - static arrays: `data_sources`, `ai_agents_used`
    - `audit_trail` object indicating “deterministic” and “compliance_ready” (declarative flags).
- **Connections:**
  - **Inputs ←** Format Urgent Alert OR Format Standard Maintenance Record
  - **No downstream node** (workflow ends here)
- **Potential failures / edge cases:**
  - If required upstream fields are missing (e.g., `recommended_action`), audit record may contain `undefined` (not blocked).
  - `Math.random().toString(36).substr(...)` uses `substr` (still works in Node.js, but considered legacy).
  - If multiple items processed quickly, `Date.now()` collisions are possible (random suffix reduces risk).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point | — | Workflow Configuration | ## How It Works<br>This workflow automates predictive maintenance for vehicle fleets by combining real-time telemetry analysis with historical pattern recognition to identify potential failures before they occur. Designed for fleet managers, maintenance supervisors, and transportation operations teams, it solves the critical challenge of preventing unexpected vehicle breakdowns while optimizing maintenance scheduling and resource allocation. The system triggers on schedule, fetches current vehicle telemetry data alongside historical maintenance records, merges datasets for comprehensive analysis, then deploys specialized AI agents using Anthropic's Claude to detect anomalies and prioritize maintenance interventions. The workflow calculates urgency levels using machine learning models and business rules, formats findings into standardized maintenance records and urgent alerts, generates audit logs for compliance tracking, and routes notifications to appropriate maintenance teams based on severity. |
| Workflow Configuration | n8n-nodes-base.set | Defines endpoints, thresholds, constants | Schedule Trigger | Fetch Real-Time Vehicle Telemetry; Fetch Historical Vehicle Data | ## Dual Data Stream Acquisition and Integration<br>**Why:** Simultaneously retrieves real-time vehicle telemetry (sensor readings, performance metrics, error codes) and historical maintenance records, then merges both datasets to provide AI agents with complete vehicle health context needed for accurate anomaly detection and trend analysis. |
| Fetch Real-Time Vehicle Telemetry | n8n-nodes-base.httpRequest | Pulls real-time telemetry JSON | Workflow Configuration | Merge Telemetry and Historical Data | ## Dual Data Stream Acquisition and Integration<br>**Why:** Simultaneously retrieves real-time vehicle telemetry (sensor readings, performance metrics, error codes) and historical maintenance records, then merges both datasets to provide AI agents with complete vehicle health context needed for accurate anomaly detection and trend analysis. |
| Fetch Historical Vehicle Data | n8n-nodes-base.httpRequest | Pulls historical maintenance JSON | Workflow Configuration | Merge Telemetry and Historical Data | ## Dual Data Stream Acquisition and Integration<br>**Why:** Simultaneously retrieves real-time vehicle telemetry (sensor readings, performance metrics, error codes) and historical maintenance records, then merges both datasets to provide AI agents with complete vehicle health context needed for accurate anomaly detection and trend analysis. |
| Merge Telemetry and Historical Data | n8n-nodes-base.merge | Combines telemetry + history | Fetch Real-Time Vehicle Telemetry; Fetch Historical Vehicle Data | Anomaly Detection Agent | ## Dual Data Stream Acquisition and Integration<br>**Why:** Simultaneously retrieves real-time vehicle telemetry (sensor readings, performance metrics, error codes) and historical maintenance records, then merges both datasets to provide AI agents with complete vehicle health context needed for accurate anomaly detection and trend analysis. |
| Anthropic Model - Anomaly Detection Agent | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude LLM for anomaly detection | — (AI connection) | Anomaly Detection Agent | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| Anomaly Detection Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces anomaly JSON schema | — (AI connection) | Anomaly Detection Agent | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| Anomaly Detection Agent | @n8n/n8n-nodes-langchain.agent | Detects anomalies from merged data | Merge Telemetry and Historical Data | Maintenance Prioritization Agent | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| Anthropic Model - Maintenance Prioritization Agent | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude LLM for prioritization | — (AI connection) | Maintenance Prioritization Agent | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| RUL Calculation Tool | @n8n/n8n-nodes-langchain.toolCode | Tool to compute RUL + predicted failure date | — (tool is called by agent) | Maintenance Prioritization Agent | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| Maintenance Prioritization Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces prioritization JSON schema | — (AI connection) | Maintenance Prioritization Agent | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| Maintenance Prioritization Agent | @n8n/n8n-nodes-langchain.agent | Produces prioritized maintenance actions | Anomaly Detection Agent | Check Urgency Level | ## AI-Powered Anomaly Detection and Maintenance Prioritization<br>**Why:** Deploys two specialized Anthropic Claude agents—one identifying abnormal patterns in telemetry data using anomaly detection models with output parsers, another calculating maintenance urgency through UL calculation tools and prioritization algorithms |
| Check Urgency Level | n8n-nodes-base.if | Routes critical vs non-critical | Maintenance Prioritization Agent | Format Urgent Alert; Format Standard Maintenance Record | ## Alert Formatting and Urgency-Based Routing<br>**Why:** Transforms AI findings into standardized maintenance records and urgent alerts, checks urgency thresholds to route critical issues for immediate attention while logging all findings to audit database |
| Format Urgent Alert | n8n-nodes-base.set | Adds urgent alert metadata | Check Urgency Level (true) | Generate Audit Log | ## Alert Formatting and Urgency-Based Routing<br>**Why:** Transforms AI findings into standardized maintenance records and urgent alerts, checks urgency thresholds to route critical issues for immediate attention while logging all findings to audit database |
| Format Standard Maintenance Record | n8n-nodes-base.set | Adds standard record metadata | Check Urgency Level (false) | Generate Audit Log | ## Alert Formatting and Urgency-Based Routing<br>**Why:** Transforms AI findings into standardized maintenance records and urgent alerts, checks urgency thresholds to route critical issues for immediate attention while logging all findings to audit database |
| Generate Audit Log | n8n-nodes-base.code | Builds compliance/audit records | Format Urgent Alert; Format Standard Maintenance Record | — | ## Alert Formatting and Urgency-Based Routing<br>**Why:** Transforms AI findings into standardized maintenance records and urgent alerts, checks urgency thresholds to route critical issues for immediate attention while logging all findings to audit database |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: “Dual-Agent Predictive Vehicle Health and Maintenance Prioritization System” (or your preferred name).

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Set an interval in **minutes** appropriate for your fleet monitoring frequency.
   - This is the workflow entry node.

3. **Add node: Workflow Configuration (Set)**
   - Node type: **Set**
   - Add fields:
     - `telemetryApiUrl` (string): your telemetry endpoint
     - `historicalApiUrl` (string): your historical maintenance endpoint
     - `anomalyThreshold` (number): `0.75` (optional; only useful if you reference it later)
     - `criticalUrgencyLevel` (string): `critical`
   - Enable: **Include Other Fields**
   - Connect: **Schedule Trigger → Workflow Configuration**

4. **Add node: Fetch Real-Time Vehicle Telemetry (HTTP Request)**
   - Node type: **HTTP Request**
   - URL expression: `$('Workflow Configuration').first().json.telemetryApiUrl`
   - Response: **JSON**
   - Configure authentication as required by your telemetry API (API key/header/OAuth2), if applicable.
   - Connect: **Workflow Configuration → Fetch Real-Time Vehicle Telemetry**

5. **Add node: Fetch Historical Vehicle Data (HTTP Request)**
   - Node type: **HTTP Request**
   - URL expression: `$('Workflow Configuration').first().json.historicalApiUrl`
   - Response: **JSON**
   - Configure authentication as required by your historical DB/API.
   - Connect: **Workflow Configuration → Fetch Historical Vehicle Data**

6. **Add node: Merge Telemetry and Historical Data**
   - Node type: **Merge**
   - Mode: **Combine**
   - Combine By: **Combine All**
   - Connect:
     - **Fetch Real-Time Vehicle Telemetry → Merge** (Input 1 / index 0)
     - **Fetch Historical Vehicle Data → Merge** (Input 2 / index 1)

7. **Create Anthropic credentials**
   - In n8n Credentials, create **Anthropic API** credentials and paste your Anthropic API key.
   - Ensure your n8n instance has the **LangChain/AI nodes** package available (`@n8n/n8n-nodes-langchain`).

8. **Add node: Anthropic Model - Anomaly Detection Agent**
   - Node type: **Anthropic Chat Model**
   - Model: `claude-3-5-sonnet-20241022`
   - Select your **Anthropic credential**.

9. **Add node: Anomaly Detection Output Parser (Structured)**
   - Node type: **Structured Output Parser**
   - Schema: manual JSON schema with required fields:
     - `vehicle_id` (string), `component_id` (string), `anomaly_detected` (boolean), `confidence_score` (0..1)
     - plus optional `anomaly_type`, `anomaly_severity` (enum), `sensor_readings` (object)

10. **Add node: Anomaly Detection Agent**
    - Node type: **AI Agent**
    - Prompt mode: **Define**
    - Text: `{{$json}}`
    - System message: instruct anomaly detection and require structured JSON output (as per your needs).
    - Enable: **Has Output Parser**
    - Connect AI ports:
      - **Anthropic Model - Anomaly Detection Agent → Anomaly Detection Agent** (AI Language Model)
      - **Anomaly Detection Output Parser → Anomaly Detection Agent** (AI Output Parser)
    - Connect main flow:
      - **Merge Telemetry and Historical Data → Anomaly Detection Agent**

11. **Add node: Anthropic Model - Maintenance Prioritization Agent**
    - Node type: **Anthropic Chat Model**
    - Model: `claude-3-5-sonnet-20241022`
    - Select your Anthropic credential.

12. **Add node: RUL Calculation Tool (Code Tool)**
    - Node type: **Code Tool**
    - Paste code implementing:
      - Severity→RUL mapping (critical/high/medium/low)
      - RUL adjustment by `confidence_score`
      - `predicted_failure_date` generation
    - This tool must be connected to the agent’s **Tool** input.

13. **Add node: Maintenance Prioritization Output Parser (Structured)**
    - Node type: **Structured Output Parser**
    - Schema: require:
      - `vehicle_id`, `component_id`, `anomaly_detected`
      - `recommended_action`, `urgency_level` (enum), `confidence_score`
    - Optional: `predicted_failure_date`, `remaining_useful_life_days`, `maintenance_priority` (1–10)

14. **Add node: Maintenance Prioritization Agent**
    - Node type: **AI Agent**
    - Text: `{{$json}}` (will receive anomaly agent output)
    - System message: instruct it to call the RUL tool and output audit-ready recommendations.
    - Enable: **Has Output Parser**
    - Connect AI ports:
      - **Anthropic Model - Maintenance Prioritization Agent → Maintenance Prioritization Agent** (AI Language Model)
      - **RUL Calculation Tool → Maintenance Prioritization Agent** (AI Tool)
      - **Maintenance Prioritization Output Parser → Maintenance Prioritization Agent** (AI Output Parser)
    - Connect main flow:
      - **Anomaly Detection Agent → Maintenance Prioritization Agent**

15. **Add node: Check Urgency Level (IF)**
    - Node type: **IF**
    - Condition (String equals):
      - Left: `$('Maintenance Prioritization Agent').item.json.urgency_level`
      - Right: `$('Workflow Configuration').first().json.criticalUrgencyLevel`
    - Connect: **Maintenance Prioritization Agent → Check Urgency Level**

16. **Add node: Format Urgent Alert (Set)**
    - Node type: **Set**
    - Add:
      - `alert_type = "URGENT"`
      - `notification_required = true`
      - `escalation_level = "immediate"`
      - `timestamp = {{$now.toISO()}}`
    - Enable **Include Other Fields**
    - Connect: **Check Urgency Level (true) → Format Urgent Alert**

17. **Add node: Format Standard Maintenance Record (Set)**
    - Node type: **Set**
    - Add:
      - `alert_type = "STANDARD"`
      - `notification_required = false`
      - `escalation_level = "scheduled"`
      - `timestamp = {{$now.toISO()}}`
    - Enable **Include Other Fields**
    - Connect: **Check Urgency Level (false) → Format Standard Maintenance Record**

18. **Add node: Generate Audit Log (Code)**
    - Node type: **Code**
    - Implement logic to:
      - Iterate over input items
      - Generate `audit_id`, add `$execution.id`, `$workflow.name`
      - Copy relevant fields (vehicle/component/anomaly/RUL/action/urgency/priority) + alert metadata
    - Connect:
      - **Format Urgent Alert → Generate Audit Log**
      - **Format Standard Maintenance Record → Generate Audit Log**
    - (Optional) Add a final node to store/send the audit log (DB, SIEM, email, webhook). This workflow currently ends here.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** Active Anthropic API account, fleet telemetry system with API access, historical maintenance database | Sticky note (“Prerequisites”) |
| **Use Cases:** Commercial fleet preventive maintenance, vehicle health monitoring, breakdown prediction | Sticky note (“Prerequisites”) |
| **Customization:** Modify anomaly detection thresholds for vehicle types, adjust prioritization algorithms for operational priorities | Sticky note (“Prerequisites”) |
| **Benefits:** Reduces unexpected breakdowns by 80%, decreases maintenance costs through predictive scheduling | Sticky note (“Prerequisites”) |
| **Setup Steps:** Configure schedule frequency; set API credentials; connect Anthropic credentials; update detection baseline; customize RUL tool and output schema | Sticky note (“Setup Steps”) |
| The workflow claims “deterministic, compliance_ready” in audit logs, but determinism is not technically enforced unless you also set model randomness/options accordingly. | Analyst note based on agent prompts and code |
| `anomalyThreshold` is defined but not referenced; to use it, include it in the anomaly agent prompt and/or add routing logic based on `confidence_score`/threshold comparisons. | Analyst note based on configuration usage |