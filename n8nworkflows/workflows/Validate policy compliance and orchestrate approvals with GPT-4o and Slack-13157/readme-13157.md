Validate policy compliance and orchestrate approvals with GPT-4o and Slack

https://n8nworkflows.xyz/workflows/validate-policy-compliance-and-orchestrate-approvals-with-gpt-4o-and-slack-13157


# Validate policy compliance and orchestrate approvals with GPT-4o and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Validate policy compliance and orchestrate approvals with GPT-4o and Slack  
**Workflow name (JSON):** Intelligent Policy Validation and Compliance Orchestration with Oversight

**Purpose:**  
This workflow runs on a schedule to fetch policy data and program performance metrics, merges them, uses GPT‑4o (via LangChain nodes) to assess compliance and produce structured findings, routes outcomes by severity, generates an orchestration plan (reporting + notifications + action plan), enforces **human-gated approval**, logs audit/decision traceability to n8n Data Tables, and notifies stakeholders via **Email** and **Slack**, with escalation for **Critical** issues.

### 1.1 Scheduled data collection (multi-source)
- Runs daily at a configured time (09:00).
- Pulls two datasets from external APIs: policy definitions/data + performance metrics.

### 1.2 AI compliance assessment (dual-agent framework)
- Agent 1: “Policy Validation Agent” detects violations, assigns compliance status, produces metrics and recommendations.
- Output is forced into a strict JSON schema using a structured output parser.

### 1.3 Severity routing + escalation
- Switch routes to:
  - **Compliant** → orchestration path
  - **Critical** → immediate email escalation (and also participates in downstream merging/logging)
  - **Non-Compliant** (fallback) → currently **not connected** (important behavior gap)

### 1.4 Execution orchestration + human approval gate
- Agent 2: “Execution Orchestration Agent” produces a report summary, stakeholder message, escalation flag, action plan, audit notes, and public accountability statement (structured output).
- A Wait node pauses execution until a webhook-based human approval resumes it.

### 1.5 Audit trail + notifications + decision traceability logging
- Stores audit trail in an n8n Data Table.
- Sends email report + Slack message.
- Merges notification paths and logs a final decision traceability record in another Data Table.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduled trigger and configuration
**Overview:** Initializes the workflow on a schedule and sets runtime configuration variables (API URLs, recipients, table names).  
**Nodes involved:** Schedule Trigger, Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point.
- **Configuration (interpreted):** Runs on an interval schedule at **09:00** (hour-based trigger).
- **Outputs:** Connects to **Workflow Configuration**.
- **Edge cases / failures:**
  - Schedule/timezone mismatch if instance timezone differs from expected operational timezone.
  - If n8n is down at trigger time, run may be missed depending on instance behavior.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — centralizes configuration constants.
- **Configuration (interpreted):**
  - Sets placeholders for:
    - `policyDataApiUrl`
    - `performanceDataApiUrl`
    - `complianceReportEmail`
    - `stakeholderSlackChannel`
    - `criticalIssuesEmail`
    - `auditTableName` = `policy_audit_trail`
    - `decisionTableName` = `decision_traceability`
  - **includeOtherFields = true** so incoming fields (from trigger) are preserved.
- **Outputs:** Fan-out to **Fetch Policy Data** and **Fetch Program Performance Data**.
- **Key expressions/variables used downstream:**
  - `$('Workflow Configuration').first().json.<key>`
- **Edge cases / failures:**
  - Placeholder values not replaced → HTTP nodes fail (invalid URL), email/slack fail (missing recipients/channel), Data Table node may fail (table not found).

**Sticky note context (applies to this block):**
- **Sticky Note2 (How It Works)**: describes overall concept and flow.
- **Sticky Note5 (Multi-Source Policy Analysis)**: explains rationale for merging policy + performance metrics.
- **Sticky Note1 (Setup Steps)**: contains high-level setup checklist.
- **Sticky Note (Prerequisites/Use Cases/Customization/Benefits)**: describes prerequisites and business value.

---

### Block 2.2 — Fetch data from external APIs and merge
**Overview:** Fetches policy data and performance data over HTTP and merges them into one item for AI processing.  
**Nodes involved:** Fetch Policy Data, Fetch Program Performance Data, Merge Data Sources

#### Node: Fetch Policy Data
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves policy dataset.
- **Configuration (interpreted):**
  - URL from config: `={{ $('Workflow Configuration').first().json.policyDataApiUrl }}`
  - Sends header `Content-Type: application/json`
  - No explicit method shown → defaults to GET in n8n HTTP Request node (unless otherwise set).
- **Outputs:** To **Merge Data Sources** (input 0).
- **Edge cases / failures:**
  - 401/403 if API auth required but not configured.
  - Non-JSON response causing later JSON stringify/agent prompt issues.
  - Timeout/network errors.
  - Response shape not matching later expectations (no `policyData` field; see note under Merge).

#### Node: Fetch Program Performance Data
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves performance metrics dataset.
- **Configuration (interpreted):**
  - URL from config: `={{ $('Workflow Configuration').first().json.performanceDataApiUrl }}`
  - Header `Content-Type: application/json`
- **Outputs:** To **Merge Data Sources** (input 1).
- **Edge cases / failures:** same as above.

#### Node: Merge Data Sources
- **Type / role:** `n8n-nodes-base.merge` — combines the two HTTP results.
- **Configuration (interpreted):**
  - Mode: **combine**
  - Combine by: **position** (first item from each input merged together).
- **Outputs:** To **Policy Validation Agent**
- **Important data-shape concern (likely bug/assumption):**
  - The **Policy Validation Agent** prompt references `$json.policyData` and `$json.performanceData`.
  - This Merge node will typically produce a single JSON object that includes fields from both inputs, but **it will not automatically nest them under `policyData` and `performanceData`** unless the HTTP responses already have those top-level keys, or unless additional Set/Transform nodes are used.
  - If the HTTP responses are plain arrays/objects, the agent will see `undefined` in those placeholders.
- **Edge cases / failures:**
  - Different item counts across inputs: “combine by position” may drop/duplicate depending on n8n merge semantics.
  - Large payloads may exceed model/context limits later.

**Sticky note context (applies to this block):**
- Sticky Note5 (Multi-Source Policy Analysis)

---

### Block 2.3 — AI policy validation (Agent 1 + model + structured output)
**Overview:** Uses GPT‑4o to perform evidence-based compliance assessment and outputs a strict structured JSON result.  
**Nodes involved:** OpenAI Model - Policy Validation, Structured Output - Policy Validation, Policy Validation Agent

#### Node: OpenAI Model - Policy Validation
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model for the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - Temperature: **0.2** (more deterministic/consistent outputs)
  - Credentials: **OpenAI account** (OpenAI API key)
- **Connections:**
  - Connected to **Policy Validation Agent** via `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid API key / org restrictions / quota exhaustion.
  - Model name not available in the tenant/region.
  - Latency/timeouts for large prompts.

#### Node: Structured Output - Policy Validation
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces schema compliance.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `complianceStatus` (Compliant|Non-Compliant|Critical expected by routing)
    - `policyViolations[]` (objects with policyId, violationType, severity, description)
    - `performanceMetrics` (adherenceScore number, riskLevel string, programEffectiveness number)
    - `recommendations[]` (strings)
    - `reasoning` (string)
- **Connections:**
  - Connected to **Policy Validation Agent** via `ai_outputParser`.
- **Edge cases / failures:**
  - Model output not parseable as JSON → parser errors.
  - Missing required fields → parser rejects output.
  - Severity/status values not matching Switch logic (e.g., “CRITICAL” vs “Critical”).

#### Node: Policy Validation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — runs the compliance analysis prompt.
- **Configuration (interpreted):**
  - Prompt text uses merged data:
    - `Policy Data: {{ JSON.stringify($json.policyData) }}`
    - `Program Performance Data: {{ JSON.stringify($json.performanceData) }}`
  - System message defines:
    - classification criteria (Critical / Non-Compliant / Compliant)
    - governance and traceability requirements
  - `hasOutputParser = true` to force structured output.
- **Inputs:**
  - Main input: merged API data from **Merge Data Sources**
  - AI language model: **OpenAI Model - Policy Validation**
  - AI output parser: **Structured Output - Policy Validation**
- **Outputs:** To **Route by Compliance Status**
- **Edge cases / failures:**
  - If `$json.policyData`/`$json.performanceData` don’t exist (see merge concern), analysis quality degrades or schema may still fill with generic values.
  - Very large JSON.stringify can exceed model context window.
  - Hallucinated fields are constrained by schema, but factual correctness depends on input completeness.

**Sticky note context (applies to this block):**
- Sticky Note4 (Dual-Agent Compliance Framework)

---

### Block 2.4 — Route by compliance status (and escalation path)
**Overview:** Routes the structured compliance assessment to downstream actions based on `complianceStatus`.  
**Nodes involved:** Route by Compliance Status, Escalate Critical Issues

#### Node: Route by Compliance Status
- **Type / role:** `n8n-nodes-base.switch` — branching control.
- **Configuration (interpreted):**
  - Rule 1: if `$json.complianceStatus == "Compliant"` → output “Compliant”
  - Rule 2: if `$json.complianceStatus == "Critical"` → output “Critical”
  - Fallback output renamed to **“Non-Compliant”** (but in JSON it’s `renameFallbackOutput: "Non-Compliant"`).
- **Connections (important):**
  - Output 0 (Compliant) → **Execution Orchestration Agent**
  - Output 1 (Critical) → **Escalate Critical Issues**
  - Fallback (Non-Compliant) → **no connection configured**
- **Edge cases / failures:**
  - Any value other than exact “Compliant” or “Critical” goes to fallback → currently becomes a **dead end** (workflow stops for Non-Compliant cases).
  - Case sensitivity is disabled, but equality check expects exact normalized content after loose validation.

#### Node: Escalate Critical Issues
- **Type / role:** `n8n-nodes-base.emailSend` — sends immediate escalation email for Critical status.
- **Configuration (interpreted):**
  - To: `criticalIssuesEmail` from config
  - From: placeholder sender email
  - Subject: “🚨 CRITICAL COMPLIANCE ISSUE - Immediate Action Required”
  - HTML includes policy violations list, risk assessment, recommendations, reasoning.
- **Connections:**
  - Sends output to **Merge Notification Paths** input index 2.
- **Edge cases / failures:**
  - Email transport/SMTP credential misconfiguration (node uses n8n email settings/credentials depending on instance).
  - HTML rendering issues if arrays are empty or fields missing (e.g., `policyViolations.map` throws if not array). In n8n expressions, if `policyViolations` is `undefined`, the expression will fail and the node errors.

**Sticky note context (applies to this block):**
- Sticky Note3 (Human-Gated Approval Routing) — conceptually covers routing + governance, though in the actual graph “Critical” skips the human wait/orchestration.

---

### Block 2.5 — AI execution orchestration (Agent 2 + model + structured output)
**Overview:** Generates an operational plan: executive report summary, stakeholder message, action plan, audit notes, and accountability statement.  
**Nodes involved:** OpenAI Model - Orchestration, Structured Output - Orchestration, Execution Orchestration Agent

#### Node: OpenAI Model - Orchestration
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration:**
  - Model: **gpt-4o**
  - Temperature: **0.2**
  - Credentials: same OpenAI credential as earlier
- **Connections:** Provides `ai_languageModel` to **Execution Orchestration Agent**
- **Edge cases:** same as other OpenAI node.

#### Node: Structured Output - Orchestration
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured`
- **Configuration (schema requires):**
  - `reportSummary` (string)
  - `stakeholderNotification` (string)
  - `escalationRequired` (boolean)
  - `actionPlan[]` (objects: action, priority, owner, deadline)
  - `auditNotes` (string)
  - `publicAccountabilityStatement` (string)
- **Connections:** Provides `ai_outputParser` to **Execution Orchestration Agent**
- **Edge cases:**
  - Parser errors if the model fails to output valid JSON or misses required fields.

#### Node: Execution Orchestration Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`
- **Configuration (interpreted):**
  - Prompt text: `Compliance Assessment: {{ JSON.stringify($json) }}`
    - This expects the upstream JSON to already be the validated compliance output.
  - System message emphasizes governance, auditability, ownership, deadlines, multi-channel comms.
  - `hasOutputParser = true`
- **Connections:**
  - Input from **Route by Compliance Status** (Compliant branch only).
  - Output to **Wait for Human Approval**.
- **Edge cases / failures:**
  - If upstream included large arrays, JSON.stringify may push prompt over context.
  - Downstream nodes expect certain fields (`actionPlan`, etc.)—schema enforces presence.

**Sticky note context (applies to this block):**
- Sticky Note4 (Dual-Agent Compliance Framework)

---

### Block 2.6 — Human approval gate and audit logging
**Overview:** Pauses the workflow for human review/approval, then logs an audit trail record in a Data Table.  
**Nodes involved:** Wait for Human Approval, Store Audit Trail

#### Node: Wait for Human Approval
- **Type / role:** `n8n-nodes-base.wait` — asynchronous pause until resume.
- **Configuration (interpreted):**
  - Resume via **webhook** (a generated resume URL in n8n).
  - `limitWaitTime = true`, `resumeAmount = 24` (interpreted as 24 time units per node defaults; commonly hours depending on node configuration—verify in UI).
- **Connections:** On resume → **Store Audit Trail**
- **Edge cases / failures:**
  - If wait times out before approval, execution ends and downstream notifications won’t happen.
  - Approval webhook URL access control: if leaked, could allow unauthorized resume.
  - No explicit “approve/reject” branching is implemented—any resume continues.

#### Node: Store Audit Trail
- **Type / role:** `n8n-nodes-base.dataTable` — persists execution data to n8n Data Tables.
- **Configuration (interpreted):**
  - Table selected by **name** from config: `policy_audit_trail`
  - Column mapping: **auto-map input data**
- **Connections:** After storing, triggers:
  - **Send Compliance Report**
  - **Notify Stakeholders - Slack**
- **Edge cases / failures:**
  - Data Table not created / wrong name → node fails.
  - Auto-mapping may create inconsistent columns over time if the input structure changes.
  - Large nested objects may not store cleanly depending on Data Table constraints.

---

### Block 2.7 — Notifications and decision traceability
**Overview:** Sends email and Slack notifications, merges notification results, and logs a final decision traceability entry.  
**Nodes involved:** Send Compliance Report, Notify Stakeholders - Slack, Merge Notification Paths, Log Decision Traceability

#### Node: Send Compliance Report
- **Type / role:** `n8n-nodes-base.emailSend` — emails the compliance report.
- **Configuration (interpreted):**
  - To: `complianceReportEmail` from config
  - Subject includes date: `Policy Compliance Report - yyyy-MM-dd`
  - HTML body includes:
    - compliance status
    - reportSummary
    - actionPlan rendered via `.map(...)`
    - publicAccountabilityStatement
- **Connections:** Output to **Merge Notification Paths** input index 0.
- **Edge cases / failures:**
  - Expression failures if `actionPlan` is not an array (schema should ensure it is).
  - Sender email placeholder not replaced.
  - Email transport not configured.

#### Node: Notify Stakeholders - Slack
- **Type / role:** `n8n-nodes-base.slack` — posts message to Slack channel.
- **Configuration (interpreted):**
  - Auth: **OAuth2** (Slack OAuth2 credential required)
  - Channel ID from config: `stakeholderSlackChannel`
  - Message includes compliance status, stakeholderNotification, and actionPlan bullets.
  - Note: message includes the “📊” character; Slack supports it, but it’s not required.
- **Connections:** Output to **Merge Notification Paths** input index 1.
- **Edge cases / failures:**
  - OAuth scopes missing (e.g., `chat:write`, channel access).
  - Channel ID invalid or bot not in channel.
  - Message length limits if action plan is long.

#### Node: Merge Notification Paths
- **Type / role:** `n8n-nodes-base.merge` — joins results from up to 3 notification branches.
- **Configuration (interpreted):**
  - Mode: **combine**
  - Combine by: **position**
  - `numberInputs = 3` expecting:
    1) email report
    2) slack
    3) critical escalation email
- **Connections:** Output to **Log Decision Traceability**.
- **Behavior caveat:**
  - In “Compliant” runs, only inputs 0 and 1 will fire; input 2 will be absent.
  - In “Critical” runs, only input 2 fires; inputs 0 and 1 do not (because Critical path does not go through orchestration/wait/store/send/slack).
  - Depending on n8n Merge semantics, “combine by position” with missing inputs may produce no output or partial output—this is a common source of downstream not executing.
- **Edge cases / failures:**
  - If any expected input doesn’t arrive, merge may not emit items (workflow appears “stuck” after notifications).

#### Node: Log Decision Traceability
- **Type / role:** `n8n-nodes-base.dataTable` — stores final traceability record.
- **Configuration (interpreted):**
  - Table selected by name from config: `decision_traceability`
  - Auto-map input data
- **Edge cases / failures:**
  - Same Data Table risks as “Store Audit Trail”.
  - If Merge emits no items, this node never runs → traceability gaps.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point | — | Workflow Configuration | ## How It Works: This workflow automates policy compliance validation and approval orchestration through intelligent AI-driven assessment… |
| Workflow Configuration | n8n-nodes-base.set | Central config variables (URLs, recipients, table names) | Schedule Trigger | Fetch Policy Data; Fetch Program Performance Data | ## Setup Steps: 1. Configure Schedule Trigger with policy review frequency… |
| Fetch Policy Data | n8n-nodes-base.httpRequest | Pull policy dataset via HTTP | Workflow Configuration | Merge Data Sources | ## Multi-Source Policy Analysis: Fetches and merges policy data with audit performance metrics… |
| Fetch Program Performance Data | n8n-nodes-base.httpRequest | Pull performance dataset via HTTP | Workflow Configuration | Merge Data Sources | ## Multi-Source Policy Analysis: Fetches and merges policy data with audit performance metrics… |
| Merge Data Sources | n8n-nodes-base.merge | Combine both datasets for analysis | Fetch Policy Data; Fetch Program Performance Data | Policy Validation Agent | ## Multi-Source Policy Analysis: Fetches and merges policy data with audit performance metrics… |
| OpenAI Model - Policy Validation | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model provider for validation agent | — (AI connection) | Policy Validation Agent (ai_languageModel) | ## Dual-Agent Compliance Framework: Processes merged data through AI agents… |
| Structured Output - Policy Validation | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for validation output | — (AI connection) | Policy Validation Agent (ai_outputParser) | ## Dual-Agent Compliance Framework: Processes merged data through AI agents… |
| Policy Validation Agent | @n8n/n8n-nodes-langchain.agent | Compliance assessment + violations + status | Merge Data Sources; OpenAI Model - Policy Validation; Structured Output - Policy Validation | Route by Compliance Status | ## Dual-Agent Compliance Framework: Processes merged data through AI agents… |
| Route by Compliance Status | n8n-nodes-base.switch | Branching by Compliant/Critical/fallback | Policy Validation Agent | Execution Orchestration Agent (Compliant); Escalate Critical Issues (Critical) | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Escalate Critical Issues | n8n-nodes-base.emailSend | Immediate escalation email for Critical findings | Route by Compliance Status | Merge Notification Paths | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| OpenAI Model - Orchestration | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model provider for orchestration agent | — (AI connection) | Execution Orchestration Agent (ai_languageModel) | ## Dual-Agent Compliance Framework: Processes merged data through AI agents… |
| Structured Output - Orchestration | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for orchestration plan | — (AI connection) | Execution Orchestration Agent (ai_outputParser) | ## Dual-Agent Compliance Framework: Processes merged data through AI agents… |
| Execution Orchestration Agent | @n8n/n8n-nodes-langchain.agent | Generates report summary, notifications, action plan | Route by Compliance Status; OpenAI Model - Orchestration; Structured Output - Orchestration | Wait for Human Approval | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Wait for Human Approval | n8n-nodes-base.wait | Pause until webhook resume (human oversight) | Execution Orchestration Agent | Store Audit Trail | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Store Audit Trail | n8n-nodes-base.dataTable | Persist audit record to Data Table | Wait for Human Approval | Send Compliance Report; Notify Stakeholders - Slack | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Send Compliance Report | n8n-nodes-base.emailSend | Email report to compliance recipient | Store Audit Trail | Merge Notification Paths | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Notify Stakeholders - Slack | n8n-nodes-base.slack | Slack broadcast to stakeholder channel | Store Audit Trail | Merge Notification Paths | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Merge Notification Paths | n8n-nodes-base.merge | Combine notification outcomes (up to 3 inputs) | Send Compliance Report; Notify Stakeholders - Slack; Escalate Critical Issues | Log Decision Traceability | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Log Decision Traceability | n8n-nodes-base.dataTable | Persist decision traceability record | Merge Notification Paths | — | ## Human-Gated Approval Routing: Routes compliance findings through status-based workflows… |
| Sticky Note | n8n-nodes-base.stickyNote | Commentary | — | — | ## Prerequisites: OpenAI/Claude API credentials… |
| Sticky Note1 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Setup Steps: 1. Configure Schedule Trigger… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Commentary | — | — | ## How It Works: This workflow automates policy compliance validation… |
| Sticky Note3 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Human-Gated Approval Routing… |
| Sticky Note4 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Dual-Agent Compliance Framework… |
| Sticky Note5 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Multi-Source Policy Analysis… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **Validate policy compliance and orchestrate approvals with GPT-4o and Slack** (or the JSON name).

2) **Add node: Schedule Trigger**
   - Node type: *Schedule Trigger*
   - Set schedule to run at **09:00** (daily/hour-based as desired).
   - Connect → **Workflow Configuration**

3) **Add node: Set (Workflow Configuration)**
   - Node type: *Set*
   - Add string fields (replace placeholders with real values):
     - `policyDataApiUrl` = `https://...`
     - `performanceDataApiUrl` = `https://...`
     - `complianceReportEmail` = `recipient@...`
     - `stakeholderSlackChannel` = Slack channel ID (e.g., `C0123...`)
     - `criticalIssuesEmail` = escalation@...
     - `auditTableName` = `policy_audit_trail`
     - `decisionTableName` = `decision_traceability`
   - Turn **Keep Only Set** OFF / enable **Include Other Fields** (so original fields remain).
   - Connect to both HTTP nodes.

4) **Add node: HTTP Request (Fetch Policy Data)**
   - URL: `{{ $('Workflow Configuration').first().json.policyDataApiUrl }}`
   - Add header `Content-Type: application/json`
   - Configure authentication as required by your API (Bearer token, basic auth, etc.).
   - Connect → **Merge Data Sources** (input 1)

5) **Add node: HTTP Request (Fetch Program Performance Data)**
   - URL: `{{ $('Workflow Configuration').first().json.performanceDataApiUrl }}`
   - Add header `Content-Type: application/json`
   - Configure authentication as required.
   - Connect → **Merge Data Sources** (input 2)

6) **Add node: Merge (Merge Data Sources)**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect → **Policy Validation Agent**
   - Recommended (to match the agent prompt): insert an intermediate *Set* node that shapes the merged output into:
     - `policyData: <policy response JSON>`
     - `performanceData: <performance response JSON>`
     Otherwise keep your APIs returning those keys.

7) **Create OpenAI credential**
   - In n8n Credentials: create **OpenAI API** credential with your API key.

8) **Add node: OpenAI Chat Model (OpenAI Model - Policy Validation)**
   - Node type: *OpenAI Chat Model (LangChain)*
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Select your OpenAI credential.

9) **Add node: Structured Output Parser (Structured Output - Policy Validation)**
   - Node type: *Structured Output Parser*
   - Schema: paste the policy validation schema (object with complianceStatus, policyViolations, performanceMetrics, recommendations, reasoning).
   - Ensure required fields match the JSON.

10) **Add node: AI Agent (Policy Validation Agent)**
   - Node type: *AI Agent (LangChain)*
   - Set prompt type: **Define**
   - Text:
     - `Policy Data: {{ JSON.stringify($json.policyData) }}`
     - `Program Performance Data: {{ JSON.stringify($json.performanceData) }}`
   - System message: use the provided policy validation instructions (classification criteria included).
   - Enable output parser and connect:
     - AI Model input ← **OpenAI Model - Policy Validation**
     - Output Parser input ← **Structured Output - Policy Validation**
   - Main input ← **Merge Data Sources**
   - Output → **Route by Compliance Status**

11) **Add node: Switch (Route by Compliance Status)**
   - Create rule “Compliant”: `{{ $json.complianceStatus }}` equals `Compliant`
   - Create rule “Critical”: `{{ $json.complianceStatus }}` equals `Critical`
   - Set fallback output name to **Non-Compliant**
   - Connect:
     - “Compliant” → **Execution Orchestration Agent**
     - “Critical” → **Escalate Critical Issues**
   - Also connect fallback to something (recommended) if you want Non-Compliant handled.

12) **Add node: Email Send (Escalate Critical Issues)**
   - To: `{{ $('Workflow Configuration').first().json.criticalIssuesEmail }}`
   - From: configure a real sender address
   - Subject: critical escalation subject
   - HTML: use the template referencing `policyViolations`, `performanceMetrics`, `recommendations`, `reasoning`
   - Connect → **Merge Notification Paths** (input 3)

13) **Add node: OpenAI Chat Model (OpenAI Model - Orchestration)**
   - Model: `gpt-4o`, temperature `0.2`, same credential.

14) **Add node: Structured Output Parser (Structured Output - Orchestration)**
   - Use the orchestration schema (reportSummary, stakeholderNotification, escalationRequired, actionPlan[], auditNotes, publicAccountabilityStatement).

15) **Add node: AI Agent (Execution Orchestration Agent)**
   - Text: `Compliance Assessment: {{ JSON.stringify($json) }}`
   - System message: orchestration and governance instructions.
   - Connect AI Model input ← **OpenAI Model - Orchestration**
   - Connect Output Parser input ← **Structured Output - Orchestration**
   - Main input ← **Route by Compliance Status (Compliant)**
   - Output → **Wait for Human Approval**

16) **Add node: Wait (Wait for Human Approval)**
   - Resume mode: **Webhook**
   - Enable wait time limit and set duration to match your governance SLA (the JSON uses “24”).
   - Output → **Store Audit Trail**
   - Operational step: define who receives the resume URL and how approval is captured (this workflow does not implement an approve/reject decision).

17) **Create Data Tables in n8n**
   - Create table named: `policy_audit_trail`
   - Create table named: `decision_traceability`

18) **Add node: Data Table (Store Audit Trail)**
   - Select Data Table by name: `{{ $('Workflow Configuration').first().json.auditTableName }}`
   - Mapping: **Auto-map input**
   - Output to both:
     - **Send Compliance Report**
     - **Notify Stakeholders - Slack**

19) **Add node: Email Send (Send Compliance Report)**
   - To: `{{ $('Workflow Configuration').first().json.complianceReportEmail }}`
   - From: real sender address
   - Subject: `Policy Compliance Report - {{ $now.toFormat('yyyy-MM-dd') }}`
   - HTML: include `reportSummary`, `actionPlan`, `publicAccountabilityStatement`
   - Output → **Merge Notification Paths** (input 1)

20) **Create Slack OAuth2 credential**
   - Ensure scopes typically include `chat:write` and channel access.
   - Install the app to the workspace and invite bot to the target channel.

21) **Add node: Slack (Notify Stakeholders - Slack)**
   - Operation: post message to a channel
   - Channel ID: `{{ $('Workflow Configuration').first().json.stakeholderSlackChannel }}`
   - Text: include status/date/stakeholderNotification/actionPlan
   - Output → **Merge Notification Paths** (input 2)

22) **Add node: Merge (Merge Notification Paths)**
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: **3**
   - Connect inputs:
     - Input 1: from **Send Compliance Report**
     - Input 2: from **Notify Stakeholders - Slack**
     - Input 3: from **Escalate Critical Issues**
   - Output → **Log Decision Traceability**
   - Recommendation: consider a merge mode that tolerates missing branches, or log independently per branch to avoid “no output” scenarios.

23) **Add node: Data Table (Log Decision Traceability)**
   - Table name: `{{ $('Workflow Configuration').first().json.decisionTableName }}`
   - Auto-map input data
   - No outputs required.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** OpenAI/Claude API credentials for AI validation agents, policy management system API access | Sticky Note (Prerequisites) |
| **Use Cases:** Financial institutions validating AML policy compliance, healthcare organizations ensuring HIPAA adherence | Sticky Note (Use Cases) |
| **Customization:** Adjust validation criteria for industry-specific regulations | Sticky Note (Customization) |
| **Benefits:** Reduces compliance review cycles by 70%, eliminates manual policy monitoring | Sticky Note (Benefits) |
| **Design pattern:** Dual-agent approach (detection → orchestration) to separate assessment from remediation planning | Sticky Note4 |
| **Governance pattern:** Human-gated approval for high-risk decisions (implemented as webhook resume, but without explicit approve/reject branching) | Sticky Note3 |
| **Known functional gap:** “Non-Compliant” fallback route is not connected, so non-critical violations currently stop the workflow | Derived from Switch connections |
| **Data-shape risk:** Agent prompt expects `$json.policyData` and `$json.performanceData`; ensure merge output provides these keys | Derived from Merge + prompt expressions |
| disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Provided by user/developer instruction |