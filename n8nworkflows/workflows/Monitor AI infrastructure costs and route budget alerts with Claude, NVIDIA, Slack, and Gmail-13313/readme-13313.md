Monitor AI infrastructure costs and route budget alerts with Claude, NVIDIA, Slack, and Gmail

https://n8nworkflows.xyz/workflows/monitor-ai-infrastructure-costs-and-route-budget-alerts-with-claude--nvidia--slack--and-gmail-13313


# Monitor AI infrastructure costs and route budget alerts with Claude, NVIDIA, Slack, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *AI Cost Intelligence and Optimization Coordination System*  
**Title provided:** *Monitor AI infrastructure costs and route budget alerts with Claude, NVIDIA, Slack, and Gmail*  

This workflow runs every 15 minutes to generate (mock) AI inference metrics, have Claude analyze cost/performance, coordinate optimization decisions (severity + recommendations), then route alerts to Slack/email and store history in n8n Data Tables.

### 1.1 Automated Monitoring Trigger & Configuration
Runs on a schedule and sets global thresholds and destinations (Slack channels, executive email, token pricing).

### 1.2 Metrics Collection (Mock Data Generator)
Generates 24h of hourly model metrics across multiple endpoints (cost, latency, errors, throughput).

### 1.3 Cost Intelligence (Claude + Structured Output)
Sends metrics to a LangChain Agent backed by Anthropic Claude and parses a structured cost analysis output.

### 1.4 Optimization Coordination (Claude + Tools + Structured Output)
A second agent determines alert severity, checks budget status via a code tool, generates routing recommendations via another code tool, and outputs a structured decision object.

### 1.5 Severity-Based Routing (Slack/Email)
A Switch node routes to Critical Slack, Warning Slack, or Executive email depending on `alertLevel`.

### 1.6 Historical Storage (Data Tables)
Stores cost analysis history and optimization recommendations in n8n Data Tables for trend/audit usage.

---

## 2. Block-by-Block Analysis

### Block 1 — Automated Monitoring Trigger & Configuration
**Overview:** Starts the workflow every 15 minutes and defines thresholds and destinations used later for alerting and reporting.  
**Nodes involved:**  
- Schedule Trigger - Every 15 Minutes  
- Workflow Configuration  

#### Node: Schedule Trigger - Every 15 Minutes
- **Type / role:** `scheduleTrigger` — time-based entry point.
- **Configuration:** Runs every **15 minutes**.
- **Connections:**  
  - Output → **Workflow Configuration**
- **Edge cases / failures:** If n8n instance is down or paused, executions won’t run; schedule drift can occur under heavy load.
- **Version notes:** typeVersion **1.3** (standard schedule trigger behavior).

#### Node: Workflow Configuration
- **Type / role:** `set` — defines workflow constants.
- **Key configuration (interpreted):**
  - `budgetThresholdUSD` = 1000  
  - `warningThresholdUSD` = 500  
  - `slackCriticalChannel` = placeholder (Slack channel ID)  
  - `slackWarningChannel` = placeholder (Slack channel ID)  
  - `executiveEmail` = placeholder  
  - `costPerMillionTokens` = 15  
  - **includeOtherFields = true** (passes through any upstream fields, though trigger provides none).
- **Connections:**  
  - Output → **Generate Mock AI Metrics Data**
- **Edge cases / failures:** Placeholders not replaced will break Slack/Email routing; mismatched field names later (see Budget Alert Tool) can cause incorrect budget math.
- **Version notes:** typeVersion **3.4**.

---

### Block 2 — Metrics Collection (Mock Data Generator)
**Overview:** Produces synthetic metrics for multiple LLM endpoints over the last 24 hours. This stands in for real telemetry (OpenAI/Anthropic/NVIDIA/etc.).  
**Nodes involved:**  
- Generate Mock AI Metrics Data

#### Node: Generate Mock AI Metrics Data
- **Type / role:** `code` — generates an array of metric records and summary.
- **Configuration choices (interpreted):**
  - Creates endpoint list: `gpt-4-turbo`, `claude-3-opus`, `claude-3-sonnet`, `gpt-3.5-turbo`, `llama-2-70b`
  - For each of last 24 hours, generates random:
    - tokens (input/output/total), request count, latency, error rate, cache hit rate
    - computes `totalCost = totalTokens * costPerToken * requestCount`
  - Returns one item with:
    - `json.metrics` (array of per-endpoint-per-hour rows)
    - `json.summary` (totalCost, totalRequests, avgLatency, etc.)
- **Connections:**  
  - Input ← **Workflow Configuration**  
  - Output → **Cost Intelligence Agent**
- **Edge cases / failures:** None typical beyond code errors; output size can become large (24 * 5 = 120 rows here) but still manageable.
- **Version notes:** typeVersion **2**.

---

### Block 3 — Cost Intelligence (Claude + Structured Output)
**Overview:** Claude analyzes the metrics and produces a structured cost/performance analysis with anomalies and recommendations.  
**Nodes involved:**  
- Cost Intelligence Agent  
- Anthropic Model - Cost Intelligence  
- Structured Output Parser - Cost Analysis  

#### Node: Cost Intelligence Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model + output parsing.
- **Configuration choices:**
  - **Text input:** `AI Metrics Data: {{ JSON.stringify($json.metrics) }}`
  - **System message:** cost intelligence instructions: breakdown, anomalies, bottlenecks, efficiency metrics, top drivers, actionable insights.
  - **Has output parser:** enabled (expects structured JSON).
- **Connections:**
  - Input ← **Generate Mock AI Metrics Data**
  - AI language model ← **Anthropic Model - Cost Intelligence**
  - AI output parser ← **Structured Output Parser - Cost Analysis**
  - Main output → **Optimization Coordinator Agent** and **Store Cost Analysis History**
- **Key variables/expressions:** uses `$json.metrics`.
- **Edge cases / failures:**
  - Model refusal/format drift leading to parser failures.
  - Token limits if real metrics are much larger than mock data.
  - If output parser schema is too strict for model output, execution fails at parsing.
- **Version notes:** typeVersion **3.1**.

#### Node: Anthropic Model - Cost Intelligence
- **Type / role:** Anthropic Chat LLM node (`lmChatAnthropic`) — provides Claude to the agent.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929` (as selected in node)
  - Uses Anthropic credentials: **Anthropic account**
- **Connections:**  
  - AI language model → **Cost Intelligence Agent**
- **Edge cases / failures:** Invalid/expired API key, rate limiting, Anthropic model deprecation, network timeouts.
- **Version notes:** typeVersion **1.3**.

#### Node: Structured Output Parser - Cost Analysis
- **Type / role:** Structured output parser — validates/forces JSON schema.
- **Schema (interpreted required fields):**
  - `totalCostUSD` (number)
  - `totalTokens` (number)
  - `averageLatencyMs` (number)
  - `costByModel` (array of objects)
  - `anomalies` (array of objects)
  - `topCostDrivers` (array of objects)
  - `efficiencyScore` (0–100)
  - `recommendations` (array of objects)
- **Connections:**  
  - AI output parser → **Cost Intelligence Agent**
- **Edge cases / failures:** If Claude returns strings for numbers or omits required fields → parser error.
- **Version notes:** typeVersion **1.3**.

---

### Block 4 — Optimization Coordination (Claude + Tools + Structured Output)
**Overview:** A second agent takes the cost analysis, checks budget status, generates routing recommendations, produces an executive summary, and outputs a structured decision object.  
**Nodes involved:**  
- Optimization Coordinator Agent  
- Anthropic Model - Optimization  
- Budget Alert Tool  
- Routing Recommendation Tool  
- Structured Output Parser - Optimization  

#### Node: Optimization Coordinator Agent
- **Type / role:** LangChain Agent — coordinates tool calls + structured decision output.
- **Configuration choices:**
  - **Text input:** `Cost Analysis: {{ JSON.stringify($json.output) }}`
  - **System message:** determine severity; use Budget Alert Tool + Routing Recommendation Tool; produce executive summary; do not modify production paths.
  - **Has output parser:** enabled.
- **Connections:**
  - Input ← **Cost Intelligence Agent**
  - AI language model ← **Anthropic Model - Optimization**
  - AI tools ← **Budget Alert Tool**, **Routing Recommendation Tool**
  - AI output parser ← **Structured Output Parser - Optimization**
  - Main output → **Route by Alert Level** and **Store Optimization Recommendations**
- **Key variables/expressions:** uses `$json.output` coming from the prior agent’s parsed output.
- **Edge cases / failures:**
  - Tool invocation mismatches (tools expect certain keys; see below).
  - Parser failures if agent output doesn’t match schema.
  - If upstream Cost Intelligence output is missing/renamed, this agent receives insufficient data.
- **Version notes:** typeVersion **3.1**.

#### Node: Anthropic Model - Optimization
- **Type / role:** Anthropic Chat LLM node — provides Claude to the coordinator agent.
- **Configuration choices:** same model `claude-sonnet-4-5-20250929`.
- **Connections:**  
  - AI language model → **Optimization Coordinator Agent**
- **Edge cases / failures:** Same as other Anthropic node (auth/rate limits/timeouts).
- **Version notes:** typeVersion **1.3**.

#### Node: Budget Alert Tool
- **Type / role:** LangChain code tool (`toolCode`) — computes budget status.
- **Configuration (interpreted logic):**
  - Reads:
    - `workflowConfig` from **Workflow Configuration** node item JSON
    - `costData` from **Cost Intelligence Agent** node item JSON
  - Computes exceeded/remaining/percent used vs budget threshold.
- **Important implementation issues (field mismatches):**
  - Code uses `workflowConfig.budgetThreshold` but configuration defines `budgetThresholdUSD`.
  - Code uses `costData.totalCost` but the cost analysis schema defines `totalCostUSD`, and the agent’s parsed output is typically under `$json.output` in the agent node, not directly at root.
  - Returns `remainingBudget`, `percentUsed`, `currentCosts`, etc., but the Optimization parser expects `budgetStatus.remaining` and `budgetStatus.percentUsed` and the Slack/Email templates reference `budgetStatus.totalCost` / `budgetStatus.remaining`.
- **Connections:**  
  - Tool output → **Optimization Coordinator Agent** (as `ai_tool`)
- **Edge cases / failures:** Wrong field mapping leads to always using defaults (`1000` threshold, `0` cost), producing false “OK” states; runtime errors if node lookups fail.
- **Version notes:** typeVersion **1.3**.

#### Node: Routing Recommendation Tool
- **Type / role:** LangChain code tool — generates routing optimization suggestions from metrics.
- **Configuration (interpreted logic):**
  - Expects input `query` containing JSON with `models` array and optional `totalRequests`.
  - For each model: computes `costPerRequest`, `avgLatency`, `costEfficiencyScore`; sorts; emits recommendations.
- **Potential data-shape mismatch:**
  - Upstream cost analysis schema has `costByModel` rather than `models`, so the agent must translate. If it passes costByModel directly without renaming, tool returns “No model data available”.
- **Connections:**  
  - Tool output → **Optimization Coordinator Agent**
- **Edge cases / failures:** Invalid JSON in tool input; missing fields (`requestCount`, `totalCost`) cause NaN or errors; returns strings (JSON text) which agent must parse/interpret.
- **Version notes:** typeVersion **1.3**.

#### Node: Structured Output Parser - Optimization
- **Type / role:** Structured output parser — enforces final coordinator output schema.
- **Schema (interpreted required fields):**
  - `alertLevel`: `"critical" | "warning" | "info"`
  - `budgetStatus`: `{ exceeded: boolean, remaining: number, percentUsed: number }`
  - `routingRecommendations`: array of objects
  - `executiveSummary`: string
  - `actionRequired`: boolean
- **Connections:**  
  - AI output parser → **Optimization Coordinator Agent**
- **Edge cases / failures:** If budget tool returns strings for numbers (e.g., `.toFixed()`), parser expects numbers and can fail unless agent converts.
- **Version notes:** typeVersion **1.3**.

---

### Block 5 — Severity-Based Alert Routing (Slack/Email)
**Overview:** Routes notifications based on `alertLevel` to Slack (critical/warning) or email (info branch as configured).  
**Nodes involved:**  
- Route by Alert Level  
- Slack - Critical Alert  
- Slack - Warning Alert  
- Email - Executive Report  

#### Node: Route by Alert Level
- **Type / role:** `switch` — branching by `alertLevel`.
- **Configuration:**
  - If `={{ $json.alertLevel }}` equals `critical` → output “Critical”
  - equals `warning` → output “Warning”
  - equals `info` → output “Info”
  - fallback output: `extra` (not connected)
- **Connections:**  
  - Input ← **Optimization Coordinator Agent**
  - Outputs:
    - Critical → **Slack - Critical Alert**
    - Warning → **Slack - Warning Alert**
    - Info → **Email - Executive Report**
- **Edge cases / failures:** If `alertLevel` missing/typo → goes to fallback (`extra`) and nothing happens (silent failure).
- **Version notes:** typeVersion **3.4**.

#### Node: Slack - Critical Alert
- **Type / role:** `slack` — sends a critical message.
- **Configuration choices:**
  - OAuth2 authentication (Slack account credential).
  - Channel ID pulled from config: `$('Workflow Configuration').first().json.slackCriticalChannel`
  - Message template references:
    - `$json.output.budgetStatus.totalCost` (likely mismatched with parser schema)
    - `$json.output.budgetStatus.percentUsed`
    - `$json.output.routingRecommendations.slice(0,3).map(...)` (assumes array of strings; schema says objects)
- **Connections:**  
  - Input ← **Route by Alert Level** (Critical)
- **Edge cases / failures:**
  - Invalid Slack OAuth token/scopes, channel not found, missing permissions.
  - Template runtime errors if `routingRecommendations` is not an array of strings or `$json.output` doesn’t exist in this branch item shape.
- **Version notes:** typeVersion **2.4**.

#### Node: Slack - Warning Alert
- **Type / role:** `slack` — sends a warning message.
- **Configuration:** Similar to Critical, but channel from `slackWarningChannel` and includes more recommendations.
- **Connections:**  
  - Input ← **Route by Alert Level** (Warning)
- **Edge cases / failures:** Same as critical; plus potential formatting issues in `.map(r => `- ${r}`)` if `r` is an object.
- **Version notes:** typeVersion **2.4**.

#### Node: Email - Executive Report
- **Type / role:** `emailSend` — sends HTML report to executive email.
- **Configuration choices:**
  - `toEmail`: from config `executiveEmail`
  - `fromEmail`: placeholder
  - Subject: `AI Cost Intelligence Executive Report - {{ $now.toFormat("yyyy-MM-dd") }}`
  - HTML body references:
    - `$json.output.executiveSummary`
    - `$json.output.budgetStatus.totalCost` (mismatch likely)
    - `$json.output.budgetStatus.remaining` (expected by parser)
    - `routingRecommendations.map(r => `<li>${r}</li>`)` (again assumes string-like)
- **Connections:**  
  - Input ← **Route by Alert Level** (Info)
- **Edge cases / failures:**
  - SMTP configuration/credentials missing (node depends on n8n email settings or configured transport).
  - Placeholder emails not replaced.
  - Expression errors if fields don’t exist.
- **Version notes:** typeVersion **2.1**.

---

### Block 6 — Historical Storage (Data Tables)
**Overview:** Saves analysis and recommendations for audits and trend analytics.  
**Nodes involved:**  
- Store Cost Analysis History  
- Store Optimization Recommendations  

#### Node: Store Cost Analysis History
- **Type / role:** `dataTable` — writes rows to an n8n Data Table.
- **Configuration:**
  - Data table name: `cost_analysis_history`
  - Columns: auto-map input data
- **Connections:**  
  - Input ← **Cost Intelligence Agent**
- **Edge cases / failures:** Data table missing/not created; schema drift if auto-mapping creates unexpected columns; permission issues in multi-user n8n.
- **Version notes:** typeVersion **1.1**.

#### Node: Store Optimization Recommendations
- **Type / role:** `dataTable` — writes rows to an n8n Data Table.
- **Configuration:**
  - Data table name: `optimization_recommendations`
  - Columns: auto-map input data
- **Connections:**  
  - Input ← **Optimization Coordinator Agent**
- **Edge cases / failures:** Same as above; also may store overly nested objects leading to poor queryability.
- **Version notes:** typeVersion **1.1**.

---

### Sticky Notes (Documentation Nodes)
These are non-executing annotation nodes but must be preserved for context:
- Sticky Note (Prerequisites/Use cases/Customization/Benefits)
- Sticky Note1 (Setup steps)
- Sticky Note2 (How it works narrative; mentions NVIDIA parsers though actual nodes are Anthropic parsers)
- Sticky Note3 (Historical storage rationale)
- Sticky Note4 (Severity routing rationale)
- Sticky Note5 (Automated monitoring rationale)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger - Every 15 Minutes | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point | — | Workflow Configuration | ## Automated Cost Monitoring<br>**Why:** Enables continuous financial oversight instead of monthly reviews, catching budget deviations before they impact quarterly results through parallel AI analysis. |
| Workflow Configuration | n8n-nodes-base.set | Defines thresholds, destinations, constants | Schedule Trigger - Every 15 Minutes | Generate Mock AI Metrics Data | ## Automated Cost Monitoring<br>**Why:** Enables continuous financial oversight instead of monthly reviews, catching budget deviations before they impact quarterly results through parallel AI analysis. |
| Generate Mock AI Metrics Data | n8n-nodes-base.code | Produces synthetic 24h metrics + summary | Workflow Configuration | Cost Intelligence Agent | ## Automated Cost Monitoring<br>**Why:** Enables continuous financial oversight instead of monthly reviews, catching budget deviations before they impact quarterly results through parallel AI analysis. |
| Cost Intelligence Agent | @n8n/n8n-nodes-langchain.agent | Claude-based cost/performance analysis with structured output | Generate Mock AI Metrics Data | Optimization Coordinator Agent; Store Cost Analysis History | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Anthropic Model - Cost Intelligence | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM provider for Cost Intelligence Agent | — (agent model connection) | Cost Intelligence Agent | ## Prerequisites<br>Claude API access, NVIDIA API credentials, Gmail/Google Workspace account, Slack workspace integration<br>## Use Cases<br>Multi-department budget variance analysis, cloud cost optimization, procurement pattern detection<br>## Customization<br>Integrate ERP systems, add department-specific rules, customize alert thresholds by category<br>## Benefits<br>Reduces overruns 40% through early detection, identifies 15-20% monthly savings |
| Structured Output Parser - Cost Analysis | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema for cost analysis | — (agent parser connection) | Cost Intelligence Agent | ## Prerequisites<br>Claude API access, NVIDIA API credentials, Gmail/Google Workspace account, Slack workspace integration<br>## Use Cases<br>Multi-department budget variance analysis, cloud cost optimization, procurement pattern detection<br>## Customization<br>Integrate ERP systems, add department-specific rules, customize alert thresholds by category<br>## Benefits<br>Reduces overruns 40% through early detection, identifies 15-20% monthly savings |
| Optimization Coordinator Agent | @n8n/n8n-nodes-langchain.agent | Determines severity, uses tools, outputs executive decision object | Cost Intelligence Agent | Route by Alert Level; Store Optimization Recommendations | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Anthropic Model - Optimization | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM provider for Optimization Coordinator Agent | — (agent model connection) | Optimization Coordinator Agent | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Budget Alert Tool | @n8n/n8n-nodes-langchain.toolCode | Tool: computes budget usage vs threshold | — (agent tool connection) | Optimization Coordinator Agent | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Routing Recommendation Tool | @n8n/n8n-nodes-langchain.toolCode | Tool: proposes routing optimizations | — (agent tool connection) | Optimization Coordinator Agent | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Structured Output Parser - Optimization | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema for coordinator output | — (agent parser connection) | Optimization Coordinator Agent | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Route by Alert Level | n8n-nodes-base.switch | Branching to Slack/email by severity | Optimization Coordinator Agent | Slack - Critical Alert; Slack - Warning Alert; Email - Executive Report | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Slack - Critical Alert | n8n-nodes-base.slack | Sends critical Slack notification | Route by Alert Level (Critical) | — | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Slack - Warning Alert | n8n-nodes-base.slack | Sends warning Slack notification | Route by Alert Level (Warning) | — | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Email - Executive Report | n8n-nodes-base.emailSend | Sends executive HTML report (info branch) | Route by Alert Level (Info) | — | ## Severity-Based Alert Routing<br>**Why:** Prioritizes critical budget issues for immediate leadership attention while managing routine optimizations efficiently, preventing alert fatigue. |
| Store Cost Analysis History | n8n-nodes-base.dataTable | Persists cost analysis results | Cost Intelligence Agent | — | ## Historical Analysis Storage & Reporting<br>**Why:** Maintains spending patterns for predictive analytics and documents optimization decisions for audit trails and data-driven planning. |
| Store Optimization Recommendations | n8n-nodes-base.dataTable | Persists optimization outputs | Optimization Coordinator Agent | — | ## Historical Analysis Storage & Reporting<br>**Why:** Maintains spending patterns for predictive analytics and documents optimization decisions for audit trails and data-driven planning. |
| Sticky Note | n8n-nodes-base.stickyNote | Annotation (prereqs/use cases/benefits) | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Annotation (setup steps) | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Annotation (how it works narrative) | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Annotation (storage rationale) | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Annotation (routing rationale) | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Annotation (monitoring rationale) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name: **AI Cost Intelligence and Optimization Coordination System**
- Set execution order to **v1** (default unless you require v2).

2) **Add trigger**
- Add node: **Schedule Trigger**
- Configure: interval → every **15 minutes**
- Connect: Schedule Trigger → (next) Workflow Configuration

3) **Add configuration constants**
- Add node: **Set** (rename to **Workflow Configuration**)
- Add fields:
  - `budgetThresholdUSD` (Number) = 1000
  - `warningThresholdUSD` (Number) = 500
  - `slackCriticalChannel` (String) = your Slack channel ID
  - `slackWarningChannel` (String) = your Slack channel ID
  - `executiveEmail` (String) = executive@company.com
  - `costPerMillionTokens` (Number) = 15
- Turn on **Include Other Fields**
- Connect: Workflow Configuration → Generate Mock AI Metrics Data

4) **Add metrics generator**
- Add node: **Code** (rename to **Generate Mock AI Metrics Data**)
- Paste the JS that generates `metrics` and `summary` (as in workflow behavior).
- Connect: Generate Mock AI Metrics Data → Cost Intelligence Agent

5) **Add Cost Intelligence Agent**
- Add node: **AI Agent** (LangChain) (rename: **Cost Intelligence Agent**)
- Prompt mode: **Define**
- Text: `AI Metrics Data: {{ JSON.stringify($json.metrics) }}`
- System message: cost intelligence instructions (breakdown, anomalies, efficiency, etc.)
- Enable **Structured Output Parser**.
- Add **Anthropic Chat Model** node (next step) and link it as the agent’s language model.
- Add **Structured Output Parser** node (next step) and link it as the agent’s output parser.

6) **Add Anthropic model for cost analysis**
- Add node: **Anthropic Chat Model** (rename: **Anthropic Model - Cost Intelligence**)
- Select model: **Claude Sonnet 4.5** (or the closest available in your n8n)
- Credentials: create/select **Anthropic API** credential.
- Connect (AI language model connection): Anthropic Model - Cost Intelligence → Cost Intelligence Agent

7) **Add structured output parser for cost analysis**
- Add node: **Structured Output Parser** (rename: **Structured Output Parser - Cost Analysis**)
- Schema type: **Manual**
- Paste the cost-analysis JSON schema (fields listed in section 2).
- Connect (AI output parser connection): Structured Output Parser - Cost Analysis → Cost Intelligence Agent

8) **Add Optimization Coordinator Agent**
- Add node: **AI Agent** (rename: **Optimization Coordinator Agent**)
- Prompt mode: **Define**
- Text: `Cost Analysis: {{ JSON.stringify($json.output) }}`
- System message: coordinator instructions (severity, tools, executive summary, structured output).
- Enable tools + structured output parser.
- Connect main: Cost Intelligence Agent → Optimization Coordinator Agent

9) **Add Anthropic model for optimization**
- Add node: **Anthropic Chat Model** (rename: **Anthropic Model - Optimization**)
- Choose same (or desired) Claude model.
- Use same Anthropic credentials.
- Connect (AI language model connection): Anthropic Model - Optimization → Optimization Coordinator Agent

10) **Add Budget Alert Tool**
- Add node: **AI Tool (Code)** (rename: **Budget Alert Tool**)
- Paste tool code that computes exceeded/remaining/percent used.
- Connect (AI tool connection): Budget Alert Tool → Optimization Coordinator Agent

11) **Add Routing Recommendation Tool**
- Add node: **AI Tool (Code)** (rename: **Routing Recommendation Tool**)
- Paste tool code that expects `{ models: [...] }` and emits recommendations.
- Connect (AI tool connection): Routing Recommendation Tool → Optimization Coordinator Agent

12) **Add structured output parser for optimization**
- Add node: **Structured Output Parser** (rename: **Structured Output Parser - Optimization**)
- Schema type: **Manual**
- Paste schema for: `alertLevel`, `budgetStatus`, `routingRecommendations`, `executiveSummary`, `actionRequired`.
- Connect (AI output parser connection): Structured Output Parser - Optimization → Optimization Coordinator Agent

13) **Add severity switch**
- Add node: **Switch** (rename: **Route by Alert Level**)
- Add 3 rules:
  - `{{$json.alertLevel}} == "critical"` → output “Critical”
  - `{{$json.alertLevel}} == "warning"` → output “Warning”
  - `{{$json.alertLevel}} == "info"` → output “Info”
- Set fallback output to “extra” (optional).
- Connect: Optimization Coordinator Agent → Route by Alert Level

14) **Add Slack nodes**
- Add node: **Slack** (rename: **Slack - Critical Alert**)
  - Auth: **OAuth2**
  - Channel: expression `{{$('Workflow Configuration').first().json.slackCriticalChannel}}`
  - Text: compose critical message using coordinator output
- Add node: **Slack** (rename: **Slack - Warning Alert**)
  - Channel: expression `{{$('Workflow Configuration').first().json.slackWarningChannel}}`
  - Text: compose warning message
- Connect:
  - Route by Alert Level (Critical) → Slack - Critical Alert
  - Route by Alert Level (Warning) → Slack - Warning Alert
- **Credentials:** create/select Slack OAuth2 credential with permission to post in channels.

15) **Add Email report**
- Add node: **Send Email** (rename: **Email - Executive Report**)
  - To: `{{$('Workflow Configuration').first().json.executiveEmail}}`
  - From: your verified sender
  - Subject + HTML body using coordinator output
- Connect: Route by Alert Level (Info) → Email - Executive Report
- **Credentials/transport:** configure SMTP (or n8n email settings) appropriate for Gmail/Workspace if that’s your target.

16) **Add Data Table storage**
- Add node: **Data Table** (rename: **Store Cost Analysis History**)
  - Table name: `cost_analysis_history`
  - Mapping: **Auto-map input**
- Add node: **Data Table** (rename: **Store Optimization Recommendations**)
  - Table name: `optimization_recommendations`
- Connect:
  - Cost Intelligence Agent → Store Cost Analysis History
  - Optimization Coordinator Agent → Store Optimization Recommendations
- Create the Data Tables in n8n beforehand (or ensure the node can create them depending on your n8n version/permissions).

17) **Add sticky notes (optional but recommended)**
- Add Sticky Notes with the same content for prerequisites, setup steps, “how it works”, routing rationale, and storage rationale.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Claude API access, NVIDIA API credentials, Gmail/Google Workspace account, Slack workspace integration | From “Prerequisites” sticky note (note: this workflow uses Anthropic + Slack + email; NVIDIA is mentioned but not implemented as a node). |
| Setup steps list (schedule, Claude creds, NVIDIA keys, Gmail auth, Slack creds, storage endpoints) | From “Setup Steps” sticky note; some items (NVIDIA keys, Gmail auth specifics) are not directly reflected in node configurations. |
| Workflow narrative about parallel AI processing, standardized outputs, severity routing, and storage for trend analysis | From “How It Works” sticky note. |
| Historical storage rationale: audit trails + predictive analytics | From “Historical Analysis Storage & Reporting” sticky note. |
| Severity routing rationale: reduce alert fatigue, prioritize leadership attention | From “Severity-Based Alert Routing” sticky note. |
| Automated monitoring rationale: continuous oversight vs monthly reviews | From “Automated Cost Monitoring” sticky note. |

