Monitor asset health and predict maintenance with Anthropic Claude and Slack

https://n8nworkflows.xyz/workflows/monitor-asset-health-and-predict-maintenance-with-anthropic-claude-and-slack-13595


# Monitor asset health and predict maintenance with Anthropic Claude and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Monitor asset health and predict maintenance with Anthropic Claude and Slack  
**Workflow name (in JSON):** AI-powered asset health monitoring and predictive maintenance

**Purpose:**  
This workflow performs scheduled industrial asset health monitoring, uses Anthropic Claude (via n8n’s LangChain Agent nodes) to assess degradation risk, optionally consults auxiliary “specialist” agents/tools (maintenance scheduling, parts readiness, lifecycle reporting, plus an MCP external data tool), then routes outcomes by risk level to Slack (CRITICAL), email escalation (HIGH), or a routine log (everything else).

**Target use cases:**  
Facility/maintenance teams monitoring fleets of pumps, motors, valves, compressors, turbines; predictive maintenance coordination; proactive alerting and escalation with AI-generated summaries.

### 1.1 Scheduled Run & Configuration
Runs every 6 hours, loads workflow parameters (thresholds, Slack channel, escalation email, MCP endpoint).

### 1.2 Asset Data Ingestion (Simulated)
Generates per-asset sensor-like metrics and derived “healthStatus/degradationScore” for downstream analysis.

### 1.3 AI Orchestration (Performance Agent + Tools)
A central **Performance Evaluation Agent** analyzes each asset record and can call tool-agents:
- **Maintenance Scheduling Agent Tool**
- **Parts Readiness Agent Tool**
- **Lifecycle Reporting Agent Tool**
- **MCP External Data Tool** (context enrichment)

### 1.4 Risk Routing & Notifications
Routes structured AI output:
- **CRITICAL** → Slack alert  
- **HIGH** → HTML email escalation report  
- **Routine (fallback)** → structured “log” Set node

### 1.5 Consolidation
Merges all routed paths into one stream for downstream consumption (currently ends after merge).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Run & Workflow Configuration
**Overview:** Triggers the workflow every 6 hours and defines runtime configuration values used throughout the workflow (Slack channel, thresholds, escalation email, MCP endpoint).  
**Nodes involved:** Schedule Asset Health Check, Workflow Configuration

#### Node: Schedule Asset Health Check
- **Type / role:** `Schedule Trigger` — periodic workflow entrypoint.
- **Configuration (interpreted):** Runs every **6 hours**.
- **Connections:**
  - **Output →** Workflow Configuration
- **Version notes:** typeVersion **1.3** (Schedule Trigger behavior is stable; interval-based schedule).
- **Edge cases / failures:**
  - Timezone expectations depend on n8n instance settings.
  - If executions overlap (long AI calls), you may need concurrency controls at workflow level.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes constants and placeholders.
- **Key fields set:**
  - `criticalThreshold` (number): **85**
  - `highThreshold` (number): **70**
  - `slackChannel` (string): placeholder (expects Slack channel ID)
  - `escalationEmail` (string): placeholder (recipient for escalations)
  - `mcpEndpoint` (string): placeholder (MCP server URL)
- **Configuration choice:** `includeOtherFields: true` (passes through upstream fields if any).
- **Connections:**
  - **Input ←** Schedule Asset Health Check
  - **Output →** Generate Asset Health Data
- **Edge cases / failures:**
  - Placeholders must be replaced; otherwise Slack/email/MCP steps will fail (invalid channel/email/URL).
  - Threshold values are defined but **not actually used** in routing logic in this workflow (routing depends on AI-produced `riskLevel`).

---

### Block 2 — Asset Data Ingestion (Simulation)
**Overview:** Produces a list of assets with simulated telemetry and degradation indicators, outputting one item per asset.  
**Nodes involved:** Generate Asset Health Data

#### Node: Generate Asset Health Data
- **Type / role:** `Code` — generates sensor/performance metrics (simulated).
- **Behavior (interpreted):**
  - Defines 5 assets (Pump, Motor, Valve, Compressor, Turbine).
  - For each asset, generates:
    - `temperature` (20–100 °C)
    - `vibration` (0–10 mm/s)
    - `pressure` (1–10 bar)
    - `operatingHours` (0–49999)
    - `lastMaintenanceDate` (random date within last 365 days; YYYY-MM-DD)
    - `degradationScore` (0–100)
    - `healthStatus` derived from degradationScore:
      - <20 Excellent, <40 Good, <60 Fair, <80 Poor, else Critical
    - `timestamp` ISO string
  - Returns **multiple items** (one per asset), enabling per-asset AI evaluation.
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Performance Evaluation Agent
- **Version notes:** typeVersion **2** for Code node.
- **Edge cases / failures:**
  - In real deployments, replace simulation with ingestion from IoT/SCADA/CMMS; ensure consistent units and ranges.
  - If downstream AI expects additional fields (e.g., baseline thresholds), include them here or in config.

---

### Block 3 — AI Orchestration (Performance Evaluation + Specialist Tools + MCP)
**Overview:** A central LangChain agent (Claude) analyzes each asset record, produces structured output, and can call specialist agent tools and an MCP tool for enrichment.  
**Nodes involved:** Performance Evaluation Agent, Anthropic Model - Performance Agent, Performance Analysis Output Parser, Maintenance Scheduling Agent Tool, Anthropic Model - Maintenance Tool, Maintenance Scheduling Output Parser, Parts Readiness Agent Tool, Anthropic Model - Parts Tool, Parts Readiness Output Parser, Lifecycle Reporting Agent Tool, Anthropic Model - Lifecycle Tool, Lifecycle Reporting Output Parser, MCP External Data Tool

#### Node: Performance Evaluation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrator agent performing the primary analysis and tool-calling.
- **Input mapping:**
  - `text` = `{{ $json }}` (passes the entire current item JSON as the message context/content).
- **System message (key instructions):**
  - Analyze temperature/vibration/pressure/operating hours/degradation score.
  - Output risk level: `CRITICAL | HIGH | MEDIUM | LOW`.
  - If `HIGH` or `CRITICAL`, call Maintenance Scheduling tool.
  - Always (or as needed) call Parts Readiness + Lifecycle Reporting.
  - Use MCP External Data Tool when contextual data is needed.
  - Escalate when uncertain.
- **Tool availability (connections):** Receives tool connections from:
  - Maintenance Scheduling Agent Tool
  - Parts Readiness Agent Tool
  - Lifecycle Reporting Agent Tool
  - MCP External Data Tool
- **Output parsing:** `hasOutputParser: true` with `Performance Analysis Output Parser` connected.
- **Connections:**
  - **Input ←** Generate Asset Health Data
  - **Output →** Route by Risk Level
  - **AI language model ←** Anthropic Model - Performance Agent
  - **AI output parser ←** Performance Analysis Output Parser
  - **AI tools ←** (4 tools listed above)
- **Version notes:** typeVersion **3.1** (agent node behavior varies across n8n versions; ensure LangChain nodes are installed/enabled).
- **Edge cases / failures:**
  - If the agent returns non-conforming JSON, the structured output parser can fail the execution.
  - Tool-call loops or excessive tool calls can increase cost/latency; consider limiting tool usage in system message.
  - Because the agent is run **once per asset item**, tool calls may happen per asset, multiplying runtime.

#### Node: Anthropic Model - Performance Agent
- **Type / role:** `lmChatAnthropic` — provides Claude chat model for the Performance Evaluation Agent.
- **Model:** `claude-sonnet-4-5-20250929` (as configured).
- **Options:** temperature 0.2, max tokens 4096.
- **Credentials:** Anthropic API credential named “Anthropic account”.
- **Connections:**
  - **Output (ai_languageModel) →** Performance Evaluation Agent
- **Edge cases / failures:**
  - Invalid API key, quota exhaustion, model deprecation/renaming, network timeouts.

#### Node: Performance Analysis Output Parser
- **Type / role:** `outputParserStructured` — enforces structured JSON output from the Performance Evaluation Agent.
- **Schema expectation (high level):**
  - `assetId`, `assetName`
  - `riskLevel` in {CRITICAL,HIGH,MEDIUM,LOW}
  - `overallHealthScore` numeric (0–100 implied)
  - arrays: `criticalSignals`, `recommendedActions`, `uncertaintyFactors`, `agentsCalled`
  - `maintenanceUrgency` in {IMMEDIATE,URGENT,SCHEDULED,ROUTINE}
  - `confidenceScore`, `escalationRequired`, `escalationReason`, `timestamp`
- **Connections:**
  - **Output (ai_outputParser) →** Performance Evaluation Agent
- **Edge cases / failures:**
  - Claude output not matching schema example (missing fields, wrong types) → parser error.
  - If you change routing logic, ensure schema still includes `riskLevel`.

---

#### Node: Maintenance Scheduling Agent Tool
- **Type / role:** `agentTool` — callable tool for scheduling recommendations.
- **Input mapping:**  
  - `text` = `{{ $fromAI("assetData", "Asset health data and performance analysis results", "json") }}`
  - This indicates the tool expects the *agent* to supply a JSON argument named `assetData`.
- **System message:** Determines maintenance windows, duration, resources, conflicts, prioritization, minimize downtime.
- **Tool description:** Coordinates scheduling and resource allocation.
- **Connections:**
  - **AI language model ←** Anthropic Model - Maintenance Tool
  - **AI output parser ←** Maintenance Scheduling Output Parser
  - **AI tool output →** Performance Evaluation Agent (tool channel)
- **Version notes:** typeVersion **3**
- **Edge cases / failures:**
  - If the calling agent does not provide `assetData` correctly, `$fromAI(...)` may be empty/invalid → tool prompt degraded.
  - Output parser mismatch can fail execution.

#### Node: Anthropic Model - Maintenance Tool
- **Type / role:** `lmChatAnthropic` — model for maintenance scheduling tool.
- **Options:** temperature 0.2, max tokens 2048.
- **Credentials:** same Anthropic account.
- **Connections:** **→** Maintenance Scheduling Agent Tool

#### Node: Maintenance Scheduling Output Parser
- **Type / role:** `outputParserStructured` — enforces tool’s structured response.
- **Schema expectation:** maintenance window, duration, resources array, priority, conflicts, alternatives, downtime impact.
- **Connections:** **→** Maintenance Scheduling Agent Tool
- **Edge cases:** schema mismatch / missing keys.

---

#### Node: Parts Readiness Agent Tool
- **Type / role:** `agentTool` — callable tool to assess parts inventory/procurement readiness.
- **Input mapping:**  
  - `text` = `{{ $fromAI("maintenanceRequirements", "Maintenance requirements and parts needed", "json") }}`
- **System message:** identifies required parts, checks inventory, lead times, shortages, alternatives, escalations.
- **Connections:**
  - **AI language model ←** Anthropic Model - Parts Tool
  - **AI output parser ←** Parts Readiness Output Parser
  - **AI tool output →** Performance Evaluation Agent
- **Edge cases:**
  - If the agent never supplies `maintenanceRequirements`, the tool may produce generic output.
  - In real use, connect this tool to actual inventory/ERP data sources; currently it is purely LLM-generated unless MCP is used.

#### Node: Anthropic Model - Parts Tool
- **Type / role:** `lmChatAnthropic`
- **Options:** temperature 0.2, max tokens 2048
- **Connections:** **→** Parts Readiness Agent Tool

#### Node: Parts Readiness Output Parser
- **Type / role:** `outputParserStructured`
- **Schema expectation:** requiredParts array with `partId, partName, quantity, available, leadTime`, plus inventory status, shortages, timeline, alternatives, cost.
- **Connections:** **→** Parts Readiness Agent Tool

---

#### Node: Lifecycle Reporting Agent Tool
- **Type / role:** `agentTool` — callable tool to produce lifecycle/TCO/RUL-style reporting.
- **Input mapping:**  
  - `text` = `{{ $fromAI("assetLifecycleData", "Asset lifecycle data and maintenance history", "json") }}`
- **System message:** maintenance history, operating trends, TCO, remaining useful life, recommendations.
- **Connections:**
  - **AI language model ←** Anthropic Model - Lifecycle Tool
  - **AI output parser ←** Lifecycle Reporting Output Parser
  - **AI tool output →** Performance Evaluation Agent
- **Edge cases:**
  - If no lifecycle history is provided by the agent, results may be speculative.
  - Consider connecting to CMMS history data for reliable lifecycle analysis.

#### Node: Anthropic Model - Lifecycle Tool
- **Type / role:** `lmChatAnthropic`
- **Options:** temperature 0.2, max tokens 2048
- **Connections:** **→** Lifecycle Reporting Agent Tool

#### Node: Lifecycle Reporting Output Parser
- **Type / role:** `outputParserStructured`
- **Schema expectation:** operating hours, maintenance history array, cost totals, downtime, reliability score, remaining life, replacement recommendation, tips.
- **Connections:** **→** Lifecycle Reporting Agent Tool

---

#### Node: MCP External Data Tool
- **Type / role:** `mcpClientTool` — enables the Performance Evaluation Agent to query an external MCP server for contextual data.
- **Configuration:**
  - `endpointUrl` = `{{ $('Workflow Configuration').first().json.mcpEndpoint }}`
  - `authentication` = `headerAuth` (expects headers-based auth configuration inside the node/credentials/environment depending on your MCP setup).
- **Connections:**
  - **AI tool output →** Performance Evaluation Agent
- **Version notes:** typeVersion **1.2**
- **Edge cases / failures:**
  - Invalid/empty MCP endpoint placeholder → connection failure.
  - Auth header misconfiguration → 401/403.
  - MCP server latency/timeouts; tool errors may propagate into agent failure depending on agent behavior.

---

### Block 4 — Risk Routing & Notification
**Overview:** Routes the agent’s structured result based on `riskLevel` into Slack, email escalation, or routine logging.  
**Nodes involved:** Route by Risk Level, Notify Critical Alert, Email Escalation Report, Log Routine Maintenance

#### Node: Route by Risk Level
- **Type / role:** `Switch` — branching by `riskLevel`.
- **Rules:**
  - Output “Critical” when `{{ $json.riskLevel }}` equals `"CRITICAL"`
  - Output “High” when equals `"HIGH"`
  - Fallback renamed to “Routine”
- **Connections:**
  - **Input ←** Performance Evaluation Agent
  - **Critical →** Notify Critical Alert
  - **High →** Email Escalation Report
  - **Routine →** Log Routine Maintenance
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**
  - If the agent output is wrapped (common with parsers) and `riskLevel` is not at root, routing fails.  
    *Note:* Your downstream nodes reference `{{ $json.output... }}`, but this Switch checks `{{ $json.riskLevel }}`. This mismatch is a key potential bug unless the Agent node truly outputs `riskLevel` at root.
  - Case sensitivity: `"critical"` would not match.

#### Node: Notify Critical Alert
- **Type / role:** `Slack` — posts a formatted alert to a channel.
- **Authentication:** OAuth2 (Slack).
- **Channel selection:** `channelId` from config: `{{ $('Workflow Configuration').first().json.slackChannel }}`
- **Message formatting:** Uses Slack markdown and interpolates fields like:
  - `{{ $json.output.assetName }}`, `{{ $json.output.assetId }}`, `{{ $json.output.riskLevel }}`, etc.
  - Joins arrays: `criticalSignals.join(", ")`
  - Formats numbered actions: `recommendedActions.map(...).join("\n")`
  - Conditional escalation line.
- **Connections:**
  - **Input ←** Route by Risk Level (Critical)
  - **Output →** Merge All Paths (input 1)
- **Version notes:** typeVersion **2.4**
- **Edge cases / failures:**
  - If the inbound data does not contain `output.*` keys, expressions will error.
  - Slack permission issues (missing `chat:write`, channel access).
  - Invalid channel ID placeholder.

#### Node: Email Escalation Report
- **Type / role:** `Email Send` — sends an HTML escalation report (HIGH risk).
- **To:** `{{ $('Workflow Configuration').first().json.escalationEmail }}`
- **From:** placeholder sender email (must be set).
- **Subject:** `{{ $json.output.riskLevel }} Risk Alert: {{ $json.output.assetName }} - Maintenance Required`
- **HTML body:** Rich formatted report with tables, conditional color blocks, list rendering, and optional escalation section.
- **Connections:**
  - **Input ←** Route by Risk Level (High)
  - **Output →** Merge All Paths (input 2)
- **Version notes:** typeVersion **2.1**
- **Edge cases / failures:**
  - SMTP/Gmail credentials missing or misconfigured.
  - From address not allowed by SMTP provider.
  - Same `output.*` structure dependency as Slack node.

#### Node: Log Routine Maintenance
- **Type / role:** `Set` — creates a structured record for routine/non-critical cases.
- **Fields set (from `output.*`):**
  - `logType` = `routine_maintenance`
  - `assetId`, `assetName`, `riskLevel`, `healthScore`, `maintenanceUrgency`, `recommendedActions`, `timestamp`
  - `status` = `logged`
- **Connections:**
  - **Input ←** Route by Risk Level (Routine)
  - **Output →** Merge All Paths (input 3)
- **Edge cases / failures:**
  - If `recommendedActions` is an array, it’s stored as-is into a string field (depending on n8n coercion); you may want `JSON.stringify()` or join.
  - Again depends on `output.*` existing.

---

### Block 5 — Consolidation
**Overview:** Merges the three possible branches into one unified output stream (useful for later storage, dashboards, or audit logs).  
**Nodes involved:** Merge All Paths

#### Node: Merge All Paths
- **Type / role:** `Merge` — combines results from Critical/High/Routine paths.
- **Mode:** `numberInputs: 3` (explicitly expects three inputs).
- **Connections:**
  - **Input 1 ←** Notify Critical Alert
  - **Input 2 ←** Email Escalation Report
  - **Input 3 ←** Log Routine Maintenance
  - **Output →** (none; workflow ends)
- **Version notes:** typeVersion **3.2**
- **Edge cases / failures:**
  - If one branch never runs in an execution, Merge behavior depends on merge mode semantics. With “numberInputs”, it may wait for all inputs; in practice, for switch branches, only one path runs per item. Consider using a merge mode that doesn’t block (or remove merge) if you see hanging behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Asset Health Check | n8n-nodes-base.scheduleTrigger | Triggers workflow every 6 hours | — | Workflow Configuration | ## How It Works<br>This workflow automates industrial asset health monitoring and predictive maintenance using Anthropic Claude across coordinated specialist agents. It targets facility managers, maintenance engineers, and operations teams in manufacturing, energy, and infrastructure sectors where reactive maintenance leads to costly unplanned downtime and asset failures. On schedule, the system ingests asset health data and routes it through a Performance Evaluation Agent that coordinates three specialist agents: Maintenance Scheduling, Parts Readiness, and Lifecycle Reporting. An MCP External Data Tool enriches analysis with real-time contextual data. Results are risk-routed—Critical assets trigger immediate Slack alerts, High-risk assets escalate via email reports, and Routine cases are logged for scheduled maintenance. All paths merge into a unified maintenance log, giving operations teams proactive, audit-ready asset intelligence before failures occur. |
| Workflow Configuration | n8n-nodes-base.set | Stores thresholds, Slack channel, escalation email, MCP endpoint | Schedule Asset Health Check | Generate Asset Health Data | ## Setup Steps<br>1. Import workflow JSON into your n8n instance.<br>2. Add Anthropic API credentials.<br>3. Set Schedule Trigger frequency aligned to your asset monitoring cycle.<br>4. Update Workflow Configuration node with asset thresholds.<br>5. Configure MCP External Data Tool with your external data source endpoint and authentication.<br>6. Add Slack credentials and set the target channel in the Notify Critical Alert node.<br>7. Set Gmail/SMTP credentials for the Email Escalation Report node. |
| Generate Asset Health Data | n8n-nodes-base.code | Simulates per-asset health/telemetry items | Workflow Configuration | Performance Evaluation Agent | ## Generate Asset Health Data<br>**What:** Loads or simulates sensor and performance metrics per asset.<br>**Why:** Provides structured operational data for downstream AI evaluation. |
| Performance Evaluation Agent | @n8n/n8n-nodes-langchain.agent | Central AI evaluator + tool orchestrator | Generate Asset Health Data | Route by Risk Level | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Anthropic Model - Performance Agent | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for performance agent | — | Performance Evaluation Agent | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Performance Analysis Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured JSON output for performance analysis | — | Performance Evaluation Agent | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Maintenance Scheduling Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for scheduling maintenance windows/resources | — (called by agent) | Performance Evaluation Agent (ai_tool) | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Anthropic Model - Maintenance Tool | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for maintenance tool | — | Maintenance Scheduling Agent Tool | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Maintenance Scheduling Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured scheduling output | — | Maintenance Scheduling Agent Tool | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Parts Readiness Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for parts availability/procurement | — (called by agent) | Performance Evaluation Agent (ai_tool) | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Anthropic Model - Parts Tool | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for parts tool | — | Parts Readiness Agent Tool | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Parts Readiness Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured parts readiness output | — | Parts Readiness Agent Tool | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Lifecycle Reporting Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Tool-agent for lifecycle/TCO/RUL reporting | — (called by agent) | Performance Evaluation Agent (ai_tool) | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Anthropic Model - Lifecycle Tool | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for lifecycle tool | — | Lifecycle Reporting Agent Tool | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Lifecycle Reporting Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured lifecycle report output | — | Lifecycle Reporting Agent Tool | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| MCP External Data Tool | @n8n/n8n-nodes-langchain.mcpClientTool | External contextual data retrieval for agent | Workflow Configuration (via expression) | Performance Evaluation Agent (ai_tool) | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Route by Risk Level | n8n-nodes-base.switch | Branches to Slack/email/log based on risk | Performance Evaluation Agent | Notify Critical Alert; Email Escalation Report; Log Routine Maintenance | ## Risk Routing & Notification<br>**What:** Critical alerts fire via Slack, High-risk cases escalate by email, Routine cases are logged.<br>**Why:** Ensures the right stakeholders act immediately on the most urgent asset conditions. |
| Notify Critical Alert | n8n-nodes-base.slack | Sends CRITICAL alert to Slack channel | Route by Risk Level (Critical) | Merge All Paths | ## Risk Routing & Notification<br>**What:** Critical alerts fire via Slack, High-risk cases escalate by email, Routine cases are logged.<br>**Why:** Ensures the right stakeholders act immediately on the most urgent asset conditions. |
| Email Escalation Report | n8n-nodes-base.emailSend | Sends HIGH risk HTML escalation email | Route by Risk Level (High) | Merge All Paths | ## Risk Routing & Notification<br>**What:** Critical alerts fire via Slack, High-risk cases escalate by email, Routine cases are logged.<br>**Why:** Ensures the right stakeholders act immediately on the most urgent asset conditions. |
| Log Routine Maintenance | n8n-nodes-base.set | Creates routine maintenance log record | Route by Risk Level (Routine) | Merge All Paths | ## Risk Routing & Notification<br>**What:** Critical alerts fire via Slack, High-risk cases escalate by email, Routine cases are logged.<br>**Why:** Ensures the right stakeholders act immediately on the most urgent asset conditions. |
| Merge All Paths | n8n-nodes-base.merge | Merges branch outputs into unified stream | Notify Critical Alert; Email Escalation Report; Log Routine Maintenance | — | ## Risk Routing & Notification<br>**What:** Critical alerts fire via Slack, High-risk cases escalate by email, Routine cases are logged.<br>**Why:** Ensures the right stakeholders act immediately on the most urgent asset conditions. |
| Sticky Note | n8n-nodes-base.stickyNote | Comment: prerequisites/use cases/customization/benefits | — | — | ## Prerequisites<br>n8n (cloud or self-hosted), Anthropic API key (Claude), Slack workspace with bot token<br>## Use Cases<br>Facility managers automating condition-based maintenance scheduling across multiple assets<br>## Customization<br>Replace Anthropic Claude with OpenAI GPT-4 or NVIDIA NIM in any agent node<br>## Benefits<br>Shifts maintenance from reactive to predictive, reducing unplanned downtime significantly |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment: setup steps | — | — | ## Setup Steps<br>1. Import workflow JSON into your n8n instance.<br>2. Add Anthropic API credentials.<br>3. Set Schedule Trigger frequency aligned to your asset monitoring cycle.<br>4. Update Workflow Configuration node with asset thresholds.<br>5. Configure MCP External Data Tool with your external data source endpoint and authentication.<br>6. Add Slack credentials and set the target channel in the Notify Critical Alert node.<br>7. Set Gmail/SMTP credentials for the Email Escalation Report node. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment: full “How it works” narrative | — | — | ## How It Works<br>This workflow automates industrial asset health monitoring and predictive maintenance using Anthropic Claude across coordinated specialist agents. It targets facility managers, maintenance engineers, and operations teams in manufacturing, energy, and infrastructure sectors where reactive maintenance leads to costly unplanned downtime and asset failures. On schedule, the system ingests asset health data and routes it through a Performance Evaluation Agent that coordinates three specialist agents: Maintenance Scheduling, Parts Readiness, and Lifecycle Reporting. An MCP External Data Tool enriches analysis with real-time contextual data. Results are risk-routed—Critical assets trigger immediate Slack alerts, High-risk assets escalate via email reports, and Routine cases are logged for scheduled maintenance. All paths merge into a unified maintenance log, giving operations teams proactive, audit-ready asset intelligence before failures occur. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment: risk routing rationale | — | — | ## Risk Routing & Notification<br>**What:** Critical alerts fire via Slack, High-risk cases escalate by email, Routine cases are logged.<br>**Why:** Ensures the right stakeholders act immediately on the most urgent asset conditions. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment: performance agent rationale | — | — | ## Performance Evaluation Agent<br>**What:** Anthropic Claude assesses overall asset condition and delegates to specialist agents.<br>**Why:** Centralises intelligence, ensuring consistent evaluation before action is taken. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment: data generation rationale | — | — | ## Generate Asset Health Data<br>**What:** Loads or simulates sensor and performance metrics per asset.<br>**Why:** Provides structured operational data for downstream AI evaluation. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *AI-powered asset health monitoring and predictive maintenance* (or your preferred name).

2) **Add Trigger: “Schedule Trigger”**
   - Node name: **Schedule Asset Health Check**
   - Configure interval: **Every 6 hours**
   - This is the main entry point.

3) **Add “Set” node for configuration**
   - Node name: **Workflow Configuration**
   - Turn on “Keep Only Set” = **false** (equivalent to includeOtherFields = true)
   - Add fields:
     - `criticalThreshold` (Number) = `85`
     - `highThreshold` (Number) = `70`
     - `slackChannel` (String) = *(your Slack channel ID, e.g. C0123…)*  
     - `escalationEmail` (String) = *(recipient email)*
     - `mcpEndpoint` (String) = *(https URL of your MCP server endpoint)*
   - Connect: **Schedule Asset Health Check → Workflow Configuration**

4) **Add “Code” node to generate/ingest asset data**
   - Node name: **Generate Asset Health Data**
   - Paste logic that outputs **one item per asset** with fields like:
     `assetId, assetName, assetType, temperature, vibration, pressure, operatingHours, lastMaintenanceDate, degradationScore, healthStatus, timestamp`
   - Connect: **Workflow Configuration → Generate Asset Health Data**
   - (In production, replace simulation with your real data ingestion.)

5) **Add Anthropic model node for the main agent**
   - Node name: **Anthropic Model - Performance Agent**
   - Node type: *LangChain → Anthropic Chat Model*
   - Model: `claude-sonnet-4-5-20250929` (or closest available in your instance)
   - Temperature: `0.2`
   - Max tokens: `4096`
   - **Credential:** Create/choose **Anthropic API** credential.

6) **Add “Structured Output Parser” for the main agent**
   - Node name: **Performance Analysis Output Parser**
   - Provide a schema example containing at least:
     - `assetId`, `assetName`, `riskLevel`, `overallHealthScore`, `recommendedActions`, `maintenanceUrgency`, `confidenceScore`, `timestamp` (plus other fields you want).
   - Keep the riskLevel enum consistent with routing.

7) **Add the central Agent node**
   - Node name: **Performance Evaluation Agent**
   - Node type: *LangChain Agent*
   - Message input: set `text` to expression `{{ $json }}`
   - Add the provided system message (performance evaluation instructions).
   - Enable structured output (connect parser).
   - Connect:
     - **Generate Asset Health Data → Performance Evaluation Agent**
     - **Anthropic Model - Performance Agent (ai_languageModel) → Performance Evaluation Agent**
     - **Performance Analysis Output Parser (ai_outputParser) → Performance Evaluation Agent**

8) **Add specialist tool: Maintenance Scheduling**
   - Add **Anthropic Model - Maintenance Tool**
     - temperature 0.2, max tokens 2048, same Anthropic credential.
   - Add **Maintenance Scheduling Output Parser** with scheduling schema example.
   - Add **Maintenance Scheduling Agent Tool**
     - Set its `text` to `{{ $fromAI("assetData", "...", "json") }}`
     - Add its system message and tool description.
   - Connect:
     - **Anthropic Model - Maintenance Tool → Maintenance Scheduling Agent Tool** (ai_languageModel)
     - **Maintenance Scheduling Output Parser → Maintenance Scheduling Agent Tool** (ai_outputParser)
     - **Maintenance Scheduling Agent Tool → Performance Evaluation Agent** (ai_tool)

9) **Add specialist tool: Parts Readiness**
   - Add **Anthropic Model - Parts Tool** (0.2, 2048, Anthropic credential)
   - Add **Parts Readiness Output Parser** with parts schema example
   - Add **Parts Readiness Agent Tool**
     - `text` = `{{ $fromAI("maintenanceRequirements", "...", "json") }}`
   - Connect:
     - Model → Tool (ai_languageModel)
     - Parser → Tool (ai_outputParser)
     - Tool → Performance Evaluation Agent (ai_tool)

10) **Add specialist tool: Lifecycle Reporting**
   - Add **Anthropic Model - Lifecycle Tool** (0.2, 2048)
   - Add **Lifecycle Reporting Output Parser**
   - Add **Lifecycle Reporting Agent Tool**
     - `text` = `{{ $fromAI("assetLifecycleData", "...", "json") }}`
   - Connect:
     - Model → Tool (ai_languageModel)
     - Parser → Tool (ai_outputParser)
     - Tool → Performance Evaluation Agent (ai_tool)

11) **Add MCP external data tool**
   - Node name: **MCP External Data Tool**
   - Endpoint URL: `{{ $('Workflow Configuration').first().json.mcpEndpoint }}`
   - Authentication: **Header Auth**
   - Configure required headers/tokens per your MCP server requirements.
   - Connect: **MCP External Data Tool → Performance Evaluation Agent** (ai_tool)

12) **Add “Switch” node for risk routing**
   - Node name: **Route by Risk Level**
   - Create rules:
     - If `{{ $json.riskLevel }}` equals `CRITICAL` → output **Critical**
     - If `{{ $json.riskLevel }}` equals `HIGH` → output **High**
     - Fallback output name: **Routine**
   - Connect: **Performance Evaluation Agent → Route by Risk Level**
   - Important: ensure the field path you check matches your agent output (see note below).

13) **Add Slack notification for CRITICAL**
   - Node name: **Notify Critical Alert**
   - Node type: Slack → “Post message” (chat:write)
   - Auth: Slack OAuth2 credential
   - Channel: expression `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Message text: format using your structured fields (asset, risk, health score, actions, etc.)
   - Connect: **Route by Risk Level (Critical) → Notify Critical Alert**

14) **Add Email escalation for HIGH**
   - Node name: **Email Escalation Report**
   - Node type: Email Send (SMTP/Gmail)
   - To: `{{ $('Workflow Configuration').first().json.escalationEmail }}`
   - From: set a valid sender mailbox for your provider
   - Subject + HTML body using your structured fields
   - Connect: **Route by Risk Level (High) → Email Escalation Report**

15) **Add routine logging “Set” node**
   - Node name: **Log Routine Maintenance**
   - Create fields for log record (assetId, assetName, riskLevel, etc.)
   - Connect: **Route by Risk Level (Routine) → Log Routine Maintenance**

16) **Add Merge node to consolidate**
   - Node name: **Merge All Paths**
   - Set number of inputs to **3**
   - Connect:
     - Notify Critical Alert → Merge (Input 1)
     - Email Escalation Report → Merge (Input 2)
     - Log Routine Maintenance → Merge (Input 3)
   - (Optional) Add a final storage node (DB/Sheet/S3) after merge.

### Critical reproduction note (data shape consistency)
Your Slack/Email/Log nodes reference fields under `{{$json.output.*}}`, while the Switch routes on `{{$json.riskLevel}}`. When rebuilding, pick **one** consistent structure:
- Either output fields at the root (`riskLevel`, `assetName`, …) and update Slack/Email/Log to use `$json.riskLevel` etc.
- Or keep a wrapper object `output` everywhere and update the Switch to use `{{$json.output.riskLevel}}`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| n8n (cloud or self-hosted), Anthropic API key (Claude), Slack workspace with bot token | Prerequisites (Sticky Note) |
| Use case: facility managers automating condition-based maintenance scheduling across multiple assets | Use Cases (Sticky Note) |
| Customization: Replace Anthropic Claude with OpenAI GPT-4 or NVIDIA NIM in any agent node | Customization (Sticky Note) |
| Benefit: Shifts maintenance from reactive to predictive, reducing unplanned downtime significantly | Benefits (Sticky Note) |
| Setup steps include configuring Anthropic credentials, schedule frequency, thresholds, MCP endpoint/auth, Slack credentials/channel, Gmail/SMTP credentials | Setup Steps (Sticky Note1) |
| Risk routing intent: CRITICAL→Slack, HIGH→Email, Routine→Log | Risk Routing & Notification (Sticky Note3) |
| Central design: performance agent delegates to specialist agents for consistency | Performance Evaluation Agent (Sticky Note4) |
| Data generation block is currently simulation; replace with real sensor/CMMS ingestion in production | Generate Asset Health Data (Sticky Note5) |