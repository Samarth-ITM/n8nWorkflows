Monitor programme performance and governance with OpenAI and Slack

https://n8nworkflows.xyz/workflows/monitor-programme-performance-and-governance-with-openai-and-slack-13158


# Monitor programme performance and governance with OpenAI and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor programme performance and governance with OpenAI and Slack  
**Workflow name (internal):** AI-Powered Programme Monitoring and Governance Reporting Automation

**Purpose:**  
Runs on a schedule to fetch programme/portfolio data from a Programme Management System API, uses OpenAI-powered agents to validate performance (budget, milestones, KPIs), orchestrates governance actions (exception escalation + briefing preparation), then routes outputs by severity to email and Slack.

**Target use cases:** PMOs, multi-project portfolio governance, executive reporting, consulting engagement health monitoring.

### 1.1 Scheduled Execution & Configuration
Runs daily (configured hour) and sets configurable thresholds, routing destinations, and API URL.

### 1.2 Data Acquisition
Fetches programme data from an external API endpoint.

### 1.3 AI Performance Validation (Monitoring)
An AI agent analyzes the programme data and returns **structured validation results** (budget, milestones, KPIs, risk, issues, recommendations).

### 1.4 AI Governance Orchestration (Tools + Synthesis)
A second AI agent reviews validation results, optionally calls specialized tools:
- **Exception Escalation Tool** (classify, urgency, escalation path, stakeholders)
- **Briefing Preparation Tool** (structured ministerial/parliamentary briefing)
Then synthesizes a structured **governance report** including severity.

### 1.5 Severity-Based Delivery
Routes by severity:
- **Critical** → Email + Slack alert
- **High** (labeled “Standard” in switch) → formatted report → standard email
- **Fallback** (“Default”) → currently not connected further (no delivery)

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Execution & Configuration
**Overview:** Triggers the workflow on a schedule and injects runtime configuration (API URL, recipients, thresholds).

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `scheduleTrigger` — workflow entry point on a time schedule.
- **Configuration (interpreted):** Triggers at **09:00** (server/workflow timezone as configured in n8n).
- **Inputs/Outputs:** No inputs. Output to **Workflow Configuration**.
- **Version:** 1.3
- **Failure / edge cases:**
  - Timezone mismatches can cause unexpected firing time.
  - If n8n instance is down at trigger time, run may be missed (depends on n8n scheduling behavior and hosting).

#### Node: Workflow Configuration
- **Type / role:** `set` — centralizes configurable parameters and keeps other fields.
- **Key fields set:**
  - `programmeApiUrl` (placeholder)
  - `criticalAlertEmail` (placeholder)
  - `standardReportEmail` (placeholder)
  - `slackChannelId` (placeholder)
  - `budgetThresholdPercent` = 85
  - `milestoneDelayDays` = 7
- **Key behavior:** `includeOtherFields: true` so the trigger payload (if any) is preserved.
- **Inputs/Outputs:** Input from **Schedule Trigger**, output to **Fetch Programme Data**.
- **Version:** 3.4
- **Failure / edge cases:**
  - If placeholders are not replaced, downstream HTTP/Email/Slack will fail.
  - Expressions referencing this node use `$('Workflow Configuration').first()`; if multiple items were ever introduced, only the first item’s config would be used.

**Sticky note context (applies to this block):**
- **How It Works** (describes end-to-end intent and multi-channel routing)
- **AI-Driven Performance Assessment** (positions monitoring agent and parsing)
- **Setup Steps** (high-level build steps)
- **Prerequisites / Use Cases / Customization / Benefits**

---

### Block 2 — Data Acquisition
**Overview:** Pulls programme data from an external Programme Management System API.

**Nodes involved:**
- Fetch Programme Data

#### Node: Fetch Programme Data
- **Type / role:** `httpRequest` — retrieves the raw programme dataset for analysis.
- **Configuration (interpreted):**
  - **URL:** `={{ $('Workflow Configuration').first().json.programmeApiUrl }}`
  - **Authentication:** `predefinedCredentialType` (credential not shown in JSON; must be configured in n8n UI)
  - Default options (no explicit method shown; typically GET unless configured otherwise)
- **Inputs/Outputs:** Input from **Workflow Configuration**, output to **Programme Monitoring Agent**.
- **Version:** 4.3
- **Failure / edge cases:**
  - Missing/invalid credentials → 401/403.
  - API downtime/timeouts → request failure.
  - Non-JSON responses or unexpected schema → monitoring agent may produce invalid structured output.
  - If API returns large payloads, token limits can affect the AI step (agent prompt includes `JSON.stringify($json)`).

---

### Block 3 — AI Performance Validation (Monitoring)
**Overview:** Uses an OpenAI-backed agent to validate programme performance and produces a strict structured JSON output via an output parser.

**Nodes involved:**
- Programme Monitoring Agent
- OpenAI Model - Monitoring
- Monitoring Output Parser

#### Node: OpenAI Model - Monitoring
- **Type / role:** `lmChatOpenAi` — chat model provider for the monitoring agent.
- **Configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: **OpenAi account**
- **Connections:** Connected to **Programme Monitoring Agent** via `ai_languageModel`.
- **Version:** 1.3
- **Failure / edge cases:**
  - Invalid OpenAI credentials / quota exceeded.
  - Model availability changes.
  - Rate limits under frequent schedules.

#### Node: Monitoring Output Parser
- **Type / role:** `outputParserStructured` — enforces the monitoring agent output schema.
- **Configuration:**
  - Manual JSON schema requiring:
    - `programmeName` (string)
    - `budgetStatus` (object with allocated/spent/remaining/percentageUsed/variance/riskLevel)
    - `milestoneStatus` (array of milestone objects)
    - `performanceIndicators` (array of KPI objects)
    - `overallRiskScore` (0–100 implied)
    - `criticalIssues` (array of strings)
    - `recommendations` (array of strings)
- **Connections:** Feeds into **Programme Monitoring Agent** via `ai_outputParser`.
- **Version:** 1.3
- **Failure / edge cases:**
  - If the model outputs fields not matching schema or wrong types, parsing fails and the agent step errors.
  - Missing required properties can break governance downstream.

#### Node: Programme Monitoring Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that analyzes programme data and returns structured validation results.
- **Configuration (interpreted):**
  - **Input text:** `Programme Data: {{ JSON.stringify($json) }}`
  - **System message:** detailed instructions to analyze budget/milestones/KPIs, compute risks, identify critical issues, and **not** alter execution.
  - **Prompt type:** “define”
  - **Output parser enabled:** yes (uses Monitoring Output Parser)
- **Inputs/Outputs:**
  - Input from **Fetch Programme Data**
  - Uses **OpenAI Model - Monitoring** via `ai_languageModel`
  - Uses **Monitoring Output Parser** via `ai_outputParser`
  - Main output goes to **Governance Agent**
- **Version:** 3.1
- **Failure / edge cases:**
  - Large API payload → prompt length/token overflow.
  - Ambiguous/missing data → model may hallucinate unless the prompt encourages explicit unknowns (not explicitly required here).
  - If programme thresholds (`budgetThresholdPercent`, `milestoneDelayDays`) are intended to be used, they are **not injected** into the agent prompt currently (only programme data is provided). This can reduce accuracy vs stated intent in the system message.

---

### Block 4 — AI Governance Orchestration (Tools + Synthesis)
**Overview:** A governance agent consumes monitoring results, calls specialized tools (each with its own model + structured parser), and outputs a structured governance report used for routing.

**Nodes involved:**
- Governance Agent
- OpenAI Model - Governance
- Governance Output Parser
- Exception Escalation Tool
- OpenAI Model - Exception Tool
- Exception Output Parser
- Briefing Preparation Tool
- OpenAI Model - Briefing Tool
- Briefing Output Parser

#### Node: OpenAI Model - Governance
- **Type / role:** `lmChatOpenAi` — model provider for Governance Agent.
- **Configuration:** `gpt-4.1-mini`, OpenAI credentials.
- **Connections:** to **Governance Agent** via `ai_languageModel`.
- **Version:** 1.3
- **Failure / edge cases:** same class as other OpenAI nodes (auth, quota, rate limiting).

#### Node: Governance Output Parser
- **Type / role:** `outputParserStructured` — enforces governance report structure.
- **Schema highlights:** `governanceReport` object containing:
  - `reportDate`, `programmeName`, `overallStatus`, `severity` (Critical/High/Medium/Low),
  - `validationSummary`, `exceptionsSummary`,
  - `briefingPrepared` (boolean), `briefingType`,
  - `actionRequired`, `stakeholdersToNotify[]`, `nextSteps[]`
- **Connections:** feeds **Governance Agent** via `ai_outputParser`.
- **Version:** 1.3
- **Failure / edge cases:** parsing failures if agent output doesn’t match schema.

#### Node: Exception Escalation Tool
- **Type / role:** `agentTool` — callable tool for the Governance Agent to analyze exceptions and escalation paths.
- **Input mapping:**  
  - `text = {{ $fromAI("validationData", "Programme validation data from monitoring agent", "json") }}`
  - This means the Governance Agent must provide `validationData` to the tool call.
- **System message:** defines severity thresholds and escalation paths; prohibits execution changes.
- **Tool description:** “Analyzes programme validation data…”
- **Model + parser:** uses **OpenAI Model - Exception Tool** and **Exception Output Parser**.
- **Connections:** connected to **Governance Agent** via `ai_tool`.
- **Version:** 3
- **Failure / edge cases:**
  - If the Governance Agent fails to pass proper `validationData`, the tool call may be incomplete.
  - Threshold logic is embedded in prompt; any mismatch with organisation policy requires prompt edits.

#### Node: OpenAI Model - Exception Tool
- **Type / role:** `lmChatOpenAi` — model provider for Exception Escalation Tool.
- **Configuration:** `gpt-4.1-mini`, OpenAI credentials.
- **Connections:** to **Exception Escalation Tool** via `ai_languageModel`.
- **Version:** 1.3

#### Node: Exception Output Parser
- **Type / role:** structured parser for exception tool outputs.
- **Schema highlights:**
  - `exceptions[]` with `exceptionType`, `severity`, `description`, `escalationPath`, `urgency`, `stakeholders[]`, `recommendedAction`
  - `requiresImmediateAction` boolean
- **Connections:** feeds **Exception Escalation Tool** via `ai_outputParser`.
- **Version:** 1.3
- **Failure / edge cases:** parser failures if tool output deviates.

#### Node: Briefing Preparation Tool
- **Type / role:** `agentTool` — callable tool to generate a structured briefing document.
- **Input mapping:**
  - `text = {{ $fromAI("programmeData", "Programme validation and exception data for briefing preparation", "json") }}`
- **System message:** defines briefing structure and neutrality constraints; prohibits execution changes.
- **Model + parser:** uses **OpenAI Model - Briefing Tool** and **Briefing Output Parser**.
- **Connections:** callable by **Governance Agent** via `ai_tool`.
- **Version:** 3
- **Failure / edge cases:**
  - Governance Agent must supply the combined data under the expected variable name `programmeData`.
  - Briefing can become lengthy; token limits may apply depending on input size and expected sections.

#### Node: OpenAI Model - Briefing Tool
- **Type / role:** `lmChatOpenAi` — model provider for briefing tool.
- **Configuration:** `gpt-4.1-mini`, OpenAI credentials.
- **Connections:** to **Briefing Preparation Tool** via `ai_languageModel`.
- **Version:** 1.3

#### Node: Briefing Output Parser
- **Type / role:** structured parser for briefing outputs.
- **Schema highlights:**
  - `briefingType` (Parliamentary/Ministerial/Executive)
  - narrative fields like `executiveSummary`, `programmeOverview`, etc.
  - `anticipatedQuestions[]` objects
  - `recommendations[]`
- **Connections:** feeds **Briefing Preparation Tool** via `ai_outputParser`.
- **Version:** 1.3

#### Node: Governance Agent
- **Type / role:** `agent` — orchestrates exception escalation + briefing preparation, synthesizes governance report with severity.
- **Configuration (interpreted):**
  - **Input text:** `Programme Validation Results: {{ JSON.stringify($json.output) }}`
    - Note: the monitoring agent’s structured output is typically available under an `output` property in LangChain agent nodes; this workflow explicitly stringifies `$json.output`.
  - **System message:** defines decision logic, tool usage expectations, and severity classes; prohibits execution changes.
  - **Output parser enabled:** yes (Governance Output Parser)
- **Inputs/Outputs:**
  - Input from **Programme Monitoring Agent**
  - Uses **OpenAI Model - Governance** via `ai_languageModel`
  - Can call **Exception Escalation Tool** and **Briefing Preparation Tool** via `ai_tool`
  - Uses **Governance Output Parser** via `ai_outputParser`
  - Main output to **Route by Severity**
- **Version:** 3.1
- **Failure / edge cases:**
  - If the incoming monitoring result is not at `$json.output`, the prompt will contain `undefined` and governance synthesis will degrade.
  - Tool calling may fail if tool input variables are missing or mismatched (`validationData`, `programmeData`).
  - The workflow does not persist briefings/exceptions anywhere except what the agent includes in governance report (no storage node).

**Sticky note context (applies to this block):**
- **Multi-Tool Governance Orchestration** (explains parallel specialized tools approach)

---

### Block 5 — Severity-Based Delivery
**Overview:** Routes governance outputs based on severity. Sends critical alerts via email and Slack; sends standard reports via email for “High” severity; fallback path is not used.

**Nodes involved:**
- Route by Severity
- Email - Critical Alert
- Slack - Critical Alert
- Format Report Data
- Email - Standard Report

#### Node: Route by Severity
- **Type / role:** `switch` — conditional routing by `severity`.
- **Rules (interpreted):**
  - Output **Critical** if `{{$json.output.governanceReport.severity}} == "Critical"`
  - Output **Standard** if `... == "High"` (note: name says Standard, condition is High only)
  - **Fallback output:** renamed **Default** (covers Medium/Low or any unexpected value)
- **Outputs/Connections:**
  - Critical → **Email - Critical Alert** and **Slack - Critical Alert**
  - Standard → **Format Report Data**
  - Default → not connected (no notifications for Medium/Low)
- **Version:** 3.4
- **Failure / edge cases:**
  - If `severity` is missing or not one of the expected strings, it will go to **Default** and effectively stop (no downstream nodes).
  - Medium/Low severities currently do nothing (likely unintended if routine monitoring is desired).

#### Node: Email - Critical Alert
- **Type / role:** `emailSend` — sends HTML email for critical alerts.
- **Configuration (interpreted):**
  - To: `{{$('Workflow Configuration').first().json.criticalAlertEmail}}`
  - From: placeholder sender email
  - Subject: `CRITICAL ALERT: <programmeName> - <severity> Severity`
  - HTML body includes validation summary, exceptions summary, action required, stakeholders, next steps using JS `.map(...).join("")`.
- **Inputs/Outputs:** From **Route by Severity (Critical)**. No outgoing connections.
- **Version:** 2.1
- **Failure / edge cases:**
  - Email server/SMTP credential configuration required in n8n (not shown).
  - If `stakeholdersToNotify` or `nextSteps` is not an array, `.map` will throw and the node may fail.
  - Placeholders must be replaced.

#### Node: Slack - Critical Alert
- **Type / role:** `slack` — posts critical alert to a Slack channel.
- **Configuration (interpreted):**
  - Authentication: OAuth2 (Slack credential “Slack account”)
  - Channel: by ID from config: `={{ $('Workflow Configuration').first().json.slackChannelId }}`
  - Message text includes governance fields; uses `.join(", ")` for stakeholders.
  - Note: message includes an alert emoji in the template.
- **Inputs/Outputs:** From **Route by Severity (Critical)**. No outgoing connections.
- **Version:** 2.4
- **Failure / edge cases:**
  - Invalid Slack OAuth scopes, revoked token, or missing permission to post in channel.
  - If `stakeholdersToNotify` is missing/not array, `.join` may fail.
  - Slack formatting limits (very long messages may be truncated).

#### Node: Format Report Data
- **Type / role:** `set` — builds email-friendly report fields for standard reporting.
- **Fields created:**
  - `reportTitle = "Programme Governance Report - <programmeName>"`
  - `reportDate = {{$now.toFormat("yyyy-MM-dd HH:mm")}}`
  - `formattedReport` multi-line summary string
- **Connections:** Input from **Route by Severity (Standard)**; output to **Email - Standard Report**.
- **Version:** 3.4
- **Failure / edge cases:**
  - Uses `$json.output.governanceReport...`; if structure differs, fields become blank.
  - Time formatting relies on n8n Luxon `$now`.

#### Node: Email - Standard Report
- **Type / role:** `emailSend` — sends HTML standard governance report.
- **Configuration (interpreted):**
  - To: `{{$('Workflow Configuration').first().json.standardReportEmail}}`
  - From: placeholder sender email
  - Subject: `={{ $json.reportTitle }}`
  - HTML body uses:
    - Programme name from `$json.output.governanceReport.programmeName`
    - **Report date incorrectly references** `{{ $json.reportDate }}` in one place but also earlier uses `{{ $json.reportDate }}` while programme report date line is `{{ $json.reportDate }}`; however there’s also a line `Report Date: {{ $json.reportDate }}` but the template includes `{{ $json.reportDate }}` while another node sets it—this is fine *only if* the incoming item is from **Format Report Data**.
- **Inputs/Outputs:** From **Format Report Data**. No outgoing connections.
- **Version:** 2.1
- **Failure / edge cases:**
  - Same email credential requirements as critical email.
  - If execution reaches this node without passing through **Format Report Data**, subject/date will be missing.

**Sticky note context (applies to this block):**
- **Severity-Based Multi-Channel Delivery** (explains alert fatigue avoidance and urgent escalation)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point (daily run) | — | Workflow Configuration | ## AI-Driven Performance Assessment<br>**What:** Processes programme data through monitoring agent with specialized output parsing for performance trend analysis<br>**Why:** Leverages AI to detect early warning indicators and performance degradation patterns that manual reviews miss<br><br>## How It Works<br>This workflow automates programme performance monitoring and governance oversight through intelligent AI-driven analysis and multi-tool orchestration. Designed for programme managers, portfolio management offices, and executive leadership teams, it solves the critical challenge of tracking programme health while coordinating interventions across escalation, exception handling, and briefing preparation workflows. The system operates on scheduled intervals, fetching programme data and processing it through dual AI agents for performance monitoring and governance assessment. It orchestrates parallel specialized tools for exception escalation, briefing preparation, and governance reporting, each with dedicated AI models and output parsers. The workflow intelligently routes findings based on severity classification, delivering critical alerts through multiple channels including email and Slack while generating formatted standard reports. By maintaining comprehensive documentation and coordinating multi-faceted governance responses, it ensures programme stakeholders receive timely, actionable intelligence while creating complete audit trails for portfolio reviews. |
| Workflow Configuration | n8n-nodes-base.set | Central configuration (API URL, thresholds, recipients) | Schedule Trigger | Fetch Programme Data | ## AI-Driven Performance Assessment<br>… (same as above)<br><br>## How It Works<br>… (same as above) |
| Fetch Programme Data | n8n-nodes-base.httpRequest | Pull programme data from external API | Workflow Configuration | Programme Monitoring Agent | ## AI-Driven Performance Assessment<br>… (same as above)<br><br>## How It Works<br>… (same as above) |
| Programme Monitoring Agent | @n8n/n8n-nodes-langchain.agent | AI validation of budget/milestones/KPIs with structured output | Fetch Programme Data | Governance Agent | ## AI-Driven Performance Assessment<br>… (same as above)<br><br>## How It Works<br>… (same as above) |
| OpenAI Model - Monitoring | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for monitoring agent | — (AI connection) | Programme Monitoring Agent | ## AI-Driven Performance Assessment<br>… (same as above) |
| Monitoring Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces monitoring schema | — (AI connection) | Programme Monitoring Agent | ## AI-Driven Performance Assessment<br>… (same as above) |
| Governance Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates tools + produces governance report with severity | Programme Monitoring Agent | Route by Severity | ## Multi-Tool Governance Orchestration<br>**What:** Coordinates governance agent with parallel specialized tools for exception escalation, briefing preparation, and governance reporting<br>**Why:** Ensures comprehensive response by deploying specialized AI models for distinct governance functions requiring different analytical approaches |
| OpenAI Model - Governance | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for governance agent | — (AI connection) | Governance Agent | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| Governance Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces governance report schema | — (AI connection) | Governance Agent | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| Exception Escalation Tool | @n8n/n8n-nodes-langchain.agentTool | Tool: classify exceptions + escalation path | — (AI tool callable) | Governance Agent | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| OpenAI Model - Exception Tool | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for exception tool | — (AI connection) | Exception Escalation Tool | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| Exception Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces exception schema | — (AI connection) | Exception Escalation Tool | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| Briefing Preparation Tool | @n8n/n8n-nodes-langchain.agentTool | Tool: prepare structured briefings | — (AI tool callable) | Governance Agent | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| OpenAI Model - Briefing Tool | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for briefing tool | — (AI connection) | Briefing Preparation Tool | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| Briefing Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces briefing schema | — (AI connection) | Briefing Preparation Tool | ## Multi-Tool Governance Orchestration<br>… (same as above) |
| Route by Severity | n8n-nodes-base.switch | Branching based on governance severity | Governance Agent | Email - Critical Alert; Slack - Critical Alert; Format Report Data | ## Severity-Based Multi-Channel Delivery<br>**What:** Routes governance outputs through severity classification to appropriate delivery channels with formatted reports and critical alerts<br>**Why:** Ensures urgent issues receive immediate executive attention while routine updates follow standard reporting protocols without alert fatigue |
| Email - Critical Alert | n8n-nodes-base.emailSend | Send critical HTML email alert | Route by Severity (Critical) | — | ## Severity-Based Multi-Channel Delivery<br>… (same as above) |
| Slack - Critical Alert | n8n-nodes-base.slack | Post critical alert to Slack | Route by Severity (Critical) | — | ## Severity-Based Multi-Channel Delivery<br>… (same as above) |
| Format Report Data | n8n-nodes-base.set | Prepare standard email subject/date/body fields | Route by Severity (Standard/High) | Email - Standard Report | ## Severity-Based Multi-Channel Delivery<br>… (same as above) |
| Email - Standard Report | n8n-nodes-base.emailSend | Send standard HTML governance report | Format Report Data | — | ## Severity-Based Multi-Channel Delivery<br>… (same as above) |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | ## Severity-Based Multi-Channel Delivery<br>**What:** Routes governance outputs through severity classification to appropriate delivery channels with formatted reports and critical alerts<br>**Why:** Ensures urgent issues receive immediate executive attention while routine updates follow standard reporting protocols without alert fatigue |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | ## Multi-Tool Governance Orchestration<br>**What:** Coordinates governance agent with parallel specialized tools for exception escalation, briefing preparation, and governance reporting<br>**Why:** Ensures comprehensive response by deploying specialized AI models for distinct governance functions requiring different analytical approaches |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | ## AI-Driven Performance Assessment<br>**What:** Processes programme data through monitoring agent with specialized output parsing for performance trend analysis<br>**Why:** Leverages AI to detect early warning indicators and performance degradation patterns that manual reviews miss |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | ## Prerequisites<br>OpenAI API credentials for multiple AI agents and specialized tools<br>## Use Cases<br>PMOs monitoring multi-project portfolios, consulting firms tracking client engagement health<br>## Customization<br>Adjust monitoring frequency for programme urgency levels<br>## Benefits<br>Reduces programme review overhead by 75%, eliminates manual status compilation |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment | — | — | ## Setup Steps<br>1. Configure Schedule Trigger with programme review frequency<br>2. Connect Workflow Configuration node with programme parameters<br>3. Set up Fetch Programme Data node with project management API credentials<br>4. Configure Programme Monitoring Agent with OpenAI API credentials<br>5. Set up monitoring processing<br>6. Connect Governance Agent with OpenAI API credentials for orchestration<br>7. Configure parallel specialized tools with respective OpenAI models<br>8. Set up tool-specific parsers |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment | — | — | ## How It Works<br>This workflow automates programme performance monitoring and governance oversight through intelligent AI-driven analysis and multi-tool orchestration. Designed for programme managers, portfolio management offices, and executive leadership teams, it solves the critical challenge of tracking programme health while coordinating interventions across escalation, exception handling, and briefing preparation workflows. The system operates on scheduled intervals, fetching programme data and processing it through dual AI agents for performance monitoring and governance assessment. It orchestrates parallel specialized tools for exception escalation, briefing preparation, and governance reporting, each with dedicated AI models and output parsers. The workflow intelligently routes findings based on severity classification, delivering critical alerts through multiple channels including email and Slack while generating formatted standard reports. By maintaining comprehensive documentation and coordinating multi-faceted governance responses, it ensures programme stakeholders receive timely, actionable intelligence while creating complete audit trails for portfolio reviews. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name: *AI-Powered Programme Monitoring and Governance Reporting Automation* (or your preferred name)

2) **Add “Schedule Trigger”** (`Schedule Trigger`)  
   - Configure trigger time (e.g., **every day at 09:00**).  
   - Connect: **Schedule Trigger → Workflow Configuration**

3) **Add “Workflow Configuration”** (`Set`)  
   - Enable: **Include Other Fields = true**  
   - Add fields:
     - `programmeApiUrl` (string) — your programme management API endpoint
     - `criticalAlertEmail` (string)
     - `standardReportEmail` (string)
     - `slackChannelId` (string) — Slack channel ID
     - `budgetThresholdPercent` (number) — 85
     - `milestoneDelayDays` (number) — 7
   - Connect: **Workflow Configuration → Fetch Programme Data**

4) **Add “Fetch Programme Data”** (`HTTP Request`)  
   - URL expression: `{{$('Workflow Configuration').first().json.programmeApiUrl}}`
   - Set authentication to match your API (the JSON indicates **Predefined Credential Type**):
     - Create/select the appropriate n8n credential (e.g., Header Auth, OAuth2, Basic Auth, etc.).
   - Ensure response is JSON (typical default).
   - Connect: **Fetch Programme Data → Programme Monitoring Agent**

5) **Add “OpenAI Model - Monitoring”** (`OpenAI Chat Model` / `lmChatOpenAi`)  
   - Model: **gpt-4.1-mini** (or equivalent)
   - Credentials: create **OpenAI API** credential in n8n and select it.

6) **Add “Monitoring Output Parser”** (`Structured Output Parser`)  
   - Schema type: Manual  
   - Paste/define the monitoring schema (programmeName, budgetStatus, milestoneStatus, performanceIndicators, overallRiskScore, criticalIssues, recommendations).

7) **Add “Programme Monitoring Agent”** (`AI Agent` / LangChain agent)  
   - Prompt/Input text: `Programme Data: {{ JSON.stringify($json) }}`
   - System message: paste the monitoring system message (budget/milestones/KPIs + risk scoring + “must not alter execution”).
   - Attach connections:
     - `ai_languageModel` ← **OpenAI Model - Monitoring**
     - `ai_outputParser` ← **Monitoring Output Parser**
   - Main connection: **Programme Monitoring Agent → Governance Agent**

8) **Add “OpenAI Model - Governance”** (`lmChatOpenAi`)  
   - Model: **gpt-4.1-mini**
   - Credentials: same OpenAI credential (or another if desired)

9) **Add “Governance Output Parser”** (`Structured Output Parser`)  
   - Define governance schema with `governanceReport` and its properties (severity, summaries, stakeholders, nextSteps, etc.).

10) **Add Exception tool trio**
   - **Exception Escalation Tool** (`Agent Tool`)
     - Tool input text expression: `{{$fromAI("validationData","Programme validation data from monitoring agent","json")}}`
     - System message: paste escalation criteria and paths.
   - **OpenAI Model - Exception Tool** (`lmChatOpenAi`): model `gpt-4.1-mini`, OpenAI credentials
   - **Exception Output Parser** (`Structured Output Parser`): schema for `exceptions[]` + `requiresImmediateAction`
   - Wire:
     - Exception Tool `ai_languageModel` ← OpenAI Model - Exception Tool
     - Exception Tool `ai_outputParser` ← Exception Output Parser

11) **Add Briefing tool trio**
   - **Briefing Preparation Tool** (`Agent Tool`)
     - Tool input text expression: `{{$fromAI("programmeData","Programme validation and exception data for briefing preparation","json")}}`
     - System message: paste briefing structure + neutrality constraints.
   - **OpenAI Model - Briefing Tool** (`lmChatOpenAi`): model `gpt-4.1-mini`
   - **Briefing Output Parser** (`Structured Output Parser`): briefing schema (briefingType, executiveSummary, Q&A, etc.)
   - Wire:
     - Briefing Tool `ai_languageModel` ← OpenAI Model - Briefing Tool
     - Briefing Tool `ai_outputParser` ← Briefing Output Parser

12) **Add “Governance Agent”** (`AI Agent`)  
   - Prompt/Input text: `Programme Validation Results: {{ JSON.stringify($json.output) }}`
   - System message: paste governance orchestration instructions (call tools when needed; classify severity).
   - Attach:
     - `ai_languageModel` ← **OpenAI Model - Governance**
     - `ai_outputParser` ← **Governance Output Parser**
     - `ai_tool` ← **Exception Escalation Tool**
     - `ai_tool` ← **Briefing Preparation Tool**
   - Connect main: **Governance Agent → Route by Severity**

13) **Add “Route by Severity”** (`Switch`)  
   - Rule 1 (rename output): **Critical** when `{{$json.output.governanceReport.severity}}` equals `Critical`
   - Rule 2 (rename output): **Standard** when equals `High`
   - Fallback output renamed: **Default**
   - Connect:
     - **Critical** → Email - Critical Alert AND Slack - Critical Alert
     - **Standard** → Format Report Data
     - (Optional improvement) **Default** → Format Report Data or a “no-op/log” node

14) **Add “Email - Critical Alert”** (`Send Email`)  
   - To: `{{$('Workflow Configuration').first().json.criticalAlertEmail}}`
   - From: set your sender email
   - Subject + HTML: use the provided templates (ensure arrays exist before `.map`)
   - Configure email credentials in n8n (SMTP or provider integration depending on your setup).

15) **Add “Slack - Critical Alert”** (`Slack`)  
   - Authentication: OAuth2; configure Slack credential with scopes to post messages.
   - Channel ID: `{{$('Workflow Configuration').first().json.slackChannelId}}`
   - Text: use the provided message template.

16) **Add “Format Report Data”** (`Set`)  
   - Include other fields = true
   - Set `reportTitle`, `reportDate` via `$now.toFormat(...)`, and `formattedReport`.
   - Connect: **Format Report Data → Email - Standard Report**

17) **Add “Email - Standard Report”** (`Send Email`)  
   - To: `{{$('Workflow Configuration').first().json.standardReportEmail}}`
   - Subject: `{{$json.reportTitle}}`
   - HTML: use provided template.
   - Configure email credentials as in step 14.

18) **Add sticky notes (optional, for documentation)**  
   - Copy the content from the provided sticky notes to preserve design intent.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Prerequisites: OpenAI API credentials for multiple AI agents and specialized tools | Sticky note “Prerequisites” |
| ## Use Cases: PMOs monitoring multi-project portfolios, consulting firms tracking client engagement health | Sticky note “Use Cases” |
| ## Customization: Adjust monitoring frequency for programme urgency levels | Sticky note “Customization” |
| ## Benefits: Reduces programme review overhead by 75%, eliminates manual status compilation | Sticky note “Benefits” |
| Medium/Low severities currently route to **Default** and stop (no email/Slack). Consider connecting Default to standard reporting or logging if routine monitoring is required. | Observed from Switch configuration |
| The monitoring agent system message mentions thresholds from configuration, but the prompt only includes programme data; consider injecting `budgetThresholdPercent` and `milestoneDelayDays` into the monitoring agent text for consistent analysis. | Observed from agent prompt vs configuration |

