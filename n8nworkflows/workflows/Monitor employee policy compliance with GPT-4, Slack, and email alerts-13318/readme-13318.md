Monitor employee policy compliance with GPT-4, Slack, and email alerts

https://n8nworkflows.xyz/workflows/monitor-employee-policy-compliance-with-gpt-4--slack--and-email-alerts-13318


# Monitor employee policy compliance with GPT-4, Slack, and email alerts

## 1. Workflow Overview

**Title (given):** Monitor employee policy compliance with GPT-4, Slack, and email alerts  
**Workflow name (JSON):** AI-Driven Policy Monitoring and Compliance Orchestration System

This workflow automates periodic monitoring of employee policy/training/certification compliance. It fetches employee policy records from an n8n Data Table, uses a GPT-4-class model to assess compliance and risk, routes employees by compliance status, and then triggers either (a) AI-driven reminder/escalation notifications via Slack + email with logged actions, or (b) audit logging for compliant employees. Finally, it merges outcomes and writes an execution log entry.

### 1.1 Scheduled Input & Configuration
Runs on a schedule, sets central configuration values (table names, Slack channel, escalation email, threshold).

### 1.2 Policy Record Retrieval
Fetches all employee policy records from an n8n Data Table.

### 1.3 AI Compliance Assessment (Policy Monitoring)
An AI Agent (GPT-4o) evaluates each employee record, returns a structured compliance/risk assessment validated by a strict schema parser.

### 1.4 Intelligent Routing
Routes items into “Non-Compliant” (includes `at_risk` and `non_compliant`), “Compliant”, or “Unknown”.

### 1.5 Notifications, Escalation & Action Logging (Non-Compliant path)
A second AI Agent chooses a communication strategy and uses tools (Slack, email, escalation code) to execute actions; results are logged to a compliance actions table.

### 1.6 Audit Logging (Compliant path)
Compliant records are augmented with audit metadata and upserted to an audit reports table.

### 1.7 Merge & Final Execution Logging
Combines outputs from compliant and non-compliant branches and writes an upsert to an execution log table.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Input & Configuration
**Overview:** Triggers daily monitoring and centralizes reusable configuration variables (Data Table names, notification targets, threshold).  
**Nodes involved:** Schedule Trigger, Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entrypoint.
- **Configuration choices:** Runs at **09:00** (interval rule with `triggerAtHour: 9`). No minute specified; n8n defaults apply.
- **Inputs/outputs:** Entry node → outputs to **Workflow Configuration**.
- **Version notes:** Type version **1.3**.
- **Failure/edge cases:**
  - Timezone depends on n8n instance settings; expected runtime can shift if instance TZ changes.
  - If you intend multiple runs/day, the current rule is not “hourly”; it’s a daily hour trigger pattern.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — defines configuration variables for downstream expressions.
- **Configuration choices (interpreted):**
  - Creates/overwrites fields (while **including other fields**) with:
    - `policyTableName = "employee_policy_records"`
    - `complianceTableName = "compliance_actions_log"`
    - `auditTableName = "audit_reports"`
    - `executionLogTableName = "workflow_execution_log"`
    - `slackChannel = <placeholder Slack Channel ID>`
    - `escalationEmail = <placeholder escalation email>`
    - `complianceThresholdDays = 30`
- **Key expressions/variables used:** None inside; provides values used later via expressions like `$('Workflow Configuration').first().json.policyTableName`.
- **Inputs/outputs:** From **Schedule Trigger** → to **Fetch Policy Records**.
- **Version notes:** Type version **3.4**.
- **Failure/edge cases:**
  - Placeholder values must be replaced; otherwise Slack/email tools may fail.
  - `includeOtherFields: true` can unintentionally carry forward extra fields if the trigger ever provides them (usually fine).

**Sticky note context applied:**  
- “Scheduled Monitoring & AI Compliance Assessment …” (covers this block and adjacent early nodes)  
- “Setup Steps …”  
- “How It Works …”  
- “Prerequisites / Use Cases / Customization / Benefits …”

---

### Block 2 — Policy Record Retrieval
**Overview:** Pulls all employee policy records from an n8n Data Table whose name is provided by configuration.  
**Nodes involved:** Fetch Policy Records

#### Node: Fetch Policy Records
- **Type / role:** `n8n-nodes-base.dataTable` — reads from n8n Data Tables.
- **Configuration choices:**
  - **Operation:** `get`
  - **Return all:** `true` (fetches all rows)
  - **Data table selection:** by **name**, expression-resolved:
    - `={{ $('Workflow Configuration').first().json.policyTableName }}`
- **Inputs/outputs:** From **Workflow Configuration** → to **Policy Monitoring Agent**.
- **Version notes:** Type version **1.1**.
- **Failure/edge cases:**
  - Table not found (name mismatch) → node failure.
  - Large table could cause memory/execution-time pressure; consider pagination/limits if needed.
  - Data shape variability: missing columns/dates can degrade AI assessment quality unless normalized.

**Sticky note context applied:** “Scheduled Monitoring & AI Compliance Assessment …”, “Setup Steps …”

---

### Block 3 — AI Compliance Assessment (Policy Monitoring)
**Overview:** An AI agent analyzes each employee record for compliance status, risk, urgency, and actions; output is enforced by a structured schema parser.  
**Nodes involved:** Policy Monitoring Agent, OpenAI Model - Policy Agent, Policy Validation Output Parser

#### Node: Policy Monitoring Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain-based agent orchestrator for LLM + parsing.
- **Configuration choices:**
  - **Prompt text:** `Employee Policy Data: {{ JSON.stringify($json) }}`
    - Sends the whole record as JSON text to the model.
  - **System message:** Defines:
    - compliance rules (compliant/at_risk/non_compliant based on 30 days)
    - risk levels (low/medium/high/critical)
    - required actions (none/reminder/escalation)
    - requires detailed reasoning and structured output fields.
  - **Prompt mode:** `define`
  - **Output parser:** enabled (`hasOutputParser: true`)
- **Key expressions/variables:**
  - Uses `$json` from **Fetch Policy Records** items.
- **Inputs/outputs:**
  - **Main input:** employee policy record rows.
  - **AI language model connection:** from **OpenAI Model - Policy Agent** (ai_languageModel).
  - **AI output parser connection:** from **Policy Validation Output Parser** (ai_outputParser).
  - **Main output:** to **Route by Compliance Status**.
- **Version notes:** Type version **3.1**.
- **Failure/edge cases:**
  - If policy records include non-ISO dates or ambiguous fields, the model may miscompute “days until expiration”.
  - If the model returns output not matching schema, parsing fails (workflow error unless handled).
  - Records missing `employeeId` are allowed by schema? (Schema *requires* `employeeId`; missing it can cause parser failure.)

#### Node: OpenAI Model - Policy Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider node.
- **Configuration choices:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.2` (more deterministic)
  - **Tools:** none configured at model level (`builtInTools: {}`)
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Connections:** Provides **ai_languageModel** input to **Policy Monitoring Agent**.
- **Version notes:** Type version **1.3**.
- **Failure/edge cases:**
  - Auth errors, insufficient quota, model access restrictions.
  - Latency/timeouts on large record sets (many items → many LLM calls).

#### Node: Policy Validation Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON schema.
- **Configuration choices:**
  - **Schema type:** manual JSON schema requiring:
    - `employeeId` (string)
    - `complianceStatus` enum: compliant / at_risk / non_compliant
    - `riskLevel` enum: low / medium / high / critical
    - `requiredAction` enum: none / reminder / escalation
    - `reasoning` (string)
  - Additional optional fields: employeeName, daysUntilExpiration, trainingStatus, certificationStatus, policyAcknowledgementStatus.
- **Connections:** Provides **ai_outputParser** to **Policy Monitoring Agent**.
- **Version notes:** Type version **1.3**.
- **Failure/edge cases:**
  - Model output can be rejected if enums don’t match exactly (e.g., “noncompliant” vs “non_compliant”).
  - `daysUntilExpiration` must be a number if present; strings cause validation failure.

**Sticky note context applied:** “Scheduled Monitoring & AI Compliance Assessment …”, “Intelligent Routing …” (next block boundary)

---

### Block 4 — Intelligent Routing
**Overview:** Splits the flow based on the AI-assessed compliance status; treats both `at_risk` and `non_compliant` as “Non-Compliant” path.  
**Nodes involved:** Route by Compliance Status

#### Node: Route by Compliance Status
- **Type / role:** `n8n-nodes-base.switch` — conditional branching.
- **Configuration choices:**
  - Rule set produces named outputs:
    - **Non-Compliant** output: when `$json.output.complianceStatus` equals `non_compliant` **OR** `at_risk`
    - **Compliant** output: when equals `compliant`
  - **Fallback:** renamed to **Unknown**
- **Key expressions/variables:**
  - `={{ $json.output.complianceStatus }}`
  - Note: compliance result is assumed nested at `$json.output` (typical for LangChain agent output).
- **Inputs/outputs:**
  - Input: from **Policy Monitoring Agent**.
  - Output 0 (Non-Compliant): to **Compliance Orchestration Agent**
  - Output 1 (Compliant): to **Prepare Compliant Records**
  - Fallback (Unknown): **not connected** (items routed to Unknown are effectively dropped from further processing)
- **Version notes:** Type version **3.4**.
- **Failure/edge cases:**
  - If the agent output structure differs (e.g., complianceStatus at root), routing will misfire and send everything to Unknown.
  - Unknown output is unhandled; consider connecting it to logging or error review.

**Sticky note context applied:** “Intelligent Routing …”

---

### Block 5 — Notifications, Escalation & Action Logging (Non-Compliant path)
**Overview:** An orchestration AI agent decides and executes reminders/escalations using Slack and email tools; critical cases may invoke custom escalation logic. Then actions are logged.  
**Nodes involved:** Compliance Orchestration Agent, OpenAI Model - Compliance Agent, Slack Notification Tool, Email Notification Tool, Escalation Logic Tool, Log Compliance Actions

#### Node: Compliance Orchestration Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that can call connected tools.
- **Configuration choices:**
  - **Prompt text:** `Compliance Issue: {{ JSON.stringify($json.output) }}`
  - **System message:** defines notification strategy:
    - reminder → email employee
    - escalation (medium/high) → email + Slack
    - escalation (critical) → call Escalation Logic Tool + multi-stakeholder notifications + urgent Slack
  - **Prompt mode:** `define`
- **Key expressions/variables:**
  - Uses `$json.output` produced by Policy Monitoring Agent parser.
- **Inputs/outputs:**
  - Main input: from **Route by Compliance Status** (Non-Compliant output).
  - **ai_languageModel:** provided by **OpenAI Model - Compliance Agent**
  - **ai_tool connections:** Slack Notification Tool, Email Notification Tool, Escalation Logic Tool are connected back into this agent.
  - Main output: to **Log Compliance Actions**
- **Version notes:** Type version **3.1**.
- **Failure/edge cases:**
  - Tool calls rely on `$fromAI()` fields being supplied correctly by the model; missing fields cause tool node expression failures.
  - The system message says “Always call Escalation Logic Tool for critical risk”; the model may not comply deterministically—consider enforcing via deterministic logic node if required.

#### Node: OpenAI Model - Compliance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for orchestration.
- **Configuration choices:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.3`
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Connections:** ai_languageModel → **Compliance Orchestration Agent**.
- **Version notes:** Type version **1.3**.
- **Failure/edge cases:** same as other OpenAI model node (auth/quota/timeouts).

#### Node: Slack Notification Tool
- **Type / role:** `n8n-nodes-base.slackTool` — agent-callable Slack posting tool.
- **Configuration choices:**
  - **Authentication:** OAuth2 (Slack)
  - **Post target:** channel (by ID)
  - **Message text:** `={{ $fromAI('message', 'Compliance alert message to send', 'string') }}`
  - **Channel ID:** default from config unless AI overrides:
    - `={{ $fromAI('slackChannel', 'Slack channel ID for compliance alerts', 'string', $('Workflow Configuration').first().json.slackChannel) }}`
- **Inputs/outputs:**
  - Exposed as `ai_tool` to **Compliance Orchestration Agent**.
- **Version notes:** Type version **2.4**.
- **Failure/edge cases:**
  - Slack OAuth scopes missing (e.g., `chat:write`) → failure.
  - Invalid channel ID or bot not in channel.
  - If `$fromAI('message')` is empty/non-string → tool execution failure.

#### Node: Email Notification Tool
- **Type / role:** `n8n-nodes-base.emailSendTool` — agent-callable email sender.
- **Configuration choices:**
  - **Operation:** `sendAndWait`
  - **To:** `={{ $fromAI('recipientEmail', 'Email address of recipient', 'string') }}`
  - **Subject:** `={{ $fromAI('emailSubject', 'Email subject line', 'string') }}`
  - **Body:** `={{ $fromAI('emailBody', 'Email message body', 'string') }}`
  - **From:** placeholder sender email `<__PLACEHOLDER_VALUE__...__>`
- **Inputs/outputs:**
  - Exposed as `ai_tool` to **Compliance Orchestration Agent**.
- **Version notes:** Type version **2.1**.
- **Failure/edge cases:**
  - Missing SMTP configuration/credentials (configured at node/instance level depending on n8n setup).
  - Invalid sender/from address or relay restrictions.
  - AI failing to provide `recipientEmail` → send failure.

#### Node: Escalation Logic Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCode` — agent-callable custom JS tool to compute escalation path.
- **Configuration choices (logic summary):**
  - Reads `complianceData` from AI via:
    - `const complianceData = $fromAI('complianceData', ..., 'json');`
  - Parses JSON if string.
  - Determines:
    - `daysOverdue = abs(daysUntilExpiration || 0)` (note: treats “until expiration” negative/positive uniformly as overdue magnitude)
    - escalation path + urgency + required actions:
      - critical OR daysOverdue > 60 → Manager → Dept Head → HR Director → Compliance Officer, urgency `immediate`
      - high OR daysOverdue > 30 → Manager → HR Rep, urgency `high`
      - else → Manager only, urgency `normal`
  - Returns a JSON string report; on error returns a fallback JSON string with HR Director / high urgency.
- **Inputs/outputs:**
  - Exposed as `ai_tool` to **Compliance Orchestration Agent**.
- **Version notes:** Type version **1.3**.
- **Failure/edge cases:**
  - The tool expects `complianceData` to include fields like `riskLevel`, `daysUntilExpiration`; missing fields degrade accuracy.
  - Semantic bug risk: `daysOverdue = abs(daysUntilExpiration)` treats “expires in 10 days” as “10 days overdue”. Consider distinguishing positive vs negative if you rely on overdue logic.
  - Returns **stringified JSON**, not an object; consuming agent must interpret string.

#### Node: Log Compliance Actions
- **Type / role:** `n8n-nodes-base.dataTable` — writes compliance actions for audit trail.
- **Configuration choices:**
  - **Operation:** `upsert`
  - **Mapping mode:** auto-map input data to columns
  - **Target table:** name from configuration:
    - `={{ $('Workflow Configuration').first().json.complianceTableName }}`
- **Inputs/outputs:** From **Compliance Orchestration Agent** → to **Merge All Actions** (input index 0).
- **Version notes:** Type version **1.1**.
- **Failure/edge cases:**
  - If the Data Table schema doesn’t match the agent output shape, auto-mapping may omit fields or fail.
  - Upsert behavior depends on table primary/unique keys; if none, may insert duplicates.

**Sticky note context applied:** “Escalation & Notification …”

---

### Block 6 — Audit Logging (Compliant path)
**Overview:** Tags compliant assessments with audit metadata and stores them as audit reports.  
**Nodes involved:** Prepare Compliant Records, Store Audit Report

#### Node: Prepare Compliant Records
- **Type / role:** `n8n-nodes-base.set` — enriches compliant items with audit fields.
- **Configuration choices:**
  - Adds:
    - `auditStatus = "compliant"`
    - `auditDate = {{$now.toISO()}}`
    - `auditType = "automated_policy_check"`
  - Includes other fields (keeps existing assessment data).
- **Inputs/outputs:** From **Route by Compliance Status** (Compliant output) → to **Store Audit Report**.
- **Version notes:** Type version **3.4**.
- **Failure/edge cases:**
  - `$now.toISO()` requires n8n date object support; generally safe.
  - Data bloat: keeping all original fields may store more than needed.

#### Node: Store Audit Report
- **Type / role:** `n8n-nodes-base.dataTable` — persists audit records.
- **Configuration choices:**
  - **Operation:** `upsert`
  - **Mapping:** auto-map
  - **Target table:** `={{ $('Workflow Configuration').first().json.auditTableName }}`
- **Inputs/outputs:** From **Prepare Compliant Records** → to **Merge All Actions** (input index 1).
- **Version notes:** Type version **1.1**.
- **Failure/edge cases:** similar to other Data Table upserts (schema mismatch, missing unique key leading to duplicates).

---

### Block 7 — Merge & Final Execution Logging
**Overview:** Combines outputs from both branches and writes a final execution log entry to a Data Table.  
**Nodes involved:** Merge All Actions, Final Execution Log

#### Node: Merge All Actions
- **Type / role:** `n8n-nodes-base.merge` — combines two streams.
- **Configuration choices:**
  - **Mode:** `combine`
  - **Combine by:** `combineByPosition`
  - Intended to pair items from branch 0 (non-compliant logs) and branch 1 (audit logs).
- **Inputs/outputs:**
  - Input 0: from **Log Compliance Actions**
  - Input 1: from **Store Audit Report**
  - Output: to **Final Execution Log**
- **Version notes:** Type version **3.2**.
- **Failure/edge cases:**
  - If the two branches produce different item counts (common), `combineByPosition` may drop/merge unexpectedly. For “append all items”, you’d typically use a different merge strategy.
  - If only one branch runs (e.g., all compliant), merge behavior depends on node’s combine semantics; may yield empty output or partial output.

#### Node: Final Execution Log
- **Type / role:** `n8n-nodes-base.dataTable` — persists execution-level log.
- **Configuration choices:**
  - **Operation:** `upsert`
  - **Mapping:** auto-map
  - **Target table:** `={{ $('Workflow Configuration').first().json.executionLogTableName }}`
- **Inputs/outputs:** From **Merge All Actions** → end.
- **Version notes:** Type version **1.1**.
- **Failure/edge cases:**
  - “Execution log” content is whatever comes out of the Merge; if merge is empty/misaligned, your execution log will be incomplete.
  - Consider writing a dedicated execution summary item (timestamp, counts, status) rather than auto-mapping merged branch data.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled entrypoint | — | Workflow Configuration | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |
| Workflow Configuration | Set | Central configuration variables | Schedule Trigger | Fetch Policy Records | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |
| Fetch Policy Records | Data Table | Retrieve employee policy records | Workflow Configuration | Policy Monitoring Agent | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |
| Policy Monitoring Agent | LangChain Agent | Analyze records; produce structured compliance assessment | Fetch Policy Records | Route by Compliance Status | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |
| OpenAI Model - Policy Agent | OpenAI Chat Model | LLM backend for policy assessment | — | Policy Monitoring Agent | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |
| Policy Validation Output Parser | Structured Output Parser | Enforce schema for compliance assessment | — | Policy Monitoring Agent | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |
| Route by Compliance Status | Switch | Branch by compliance status | Policy Monitoring Agent | Compliance Orchestration Agent; Prepare Compliant Records | ## Intelligent Routing<br>**Why**: Prioritizes critical issues for immediate action while logging minor infractions |
| Compliance Orchestration Agent | LangChain Agent | Decide and execute reminder/escalation actions via tools | Route by Compliance Status | Log Compliance Actions | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| OpenAI Model - Compliance Agent | OpenAI Chat Model | LLM backend for orchestration | — | Compliance Orchestration Agent | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| Slack Notification Tool | Slack Tool | Post compliance alerts to Slack | — (tool call) | Compliance Orchestration Agent (tool result) | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| Email Notification Tool | Email Send Tool | Send reminder/escalation emails | — (tool call) | Compliance Orchestration Agent (tool result) | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| Escalation Logic Tool | Code Tool (LangChain) | Compute escalation path/urgency for critical cases | — (tool call) | Compliance Orchestration Agent (tool result) | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| Log Compliance Actions | Data Table | Upsert compliance actions log | Compliance Orchestration Agent | Merge All Actions | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| Prepare Compliant Records | Set | Add audit metadata for compliant items | Route by Compliance Status | Store Audit Report | ## Intelligent Routing<br>**Why**: Prioritizes critical issues for immediate action while logging minor infractions |
| Store Audit Report | Data Table | Upsert audit report for compliant items | Prepare Compliant Records | Merge All Actions | ## Intelligent Routing<br>**Why**: Prioritizes critical issues for immediate action while logging minor infractions |
| Merge All Actions | Merge | Combine compliant + noncompliant outcomes | Log Compliance Actions; Store Audit Report | Final Execution Log |  |
| Final Execution Log | Data Table | Persist final (merged) execution output | Merge All Actions | — |  |
| Sticky Note | Sticky Note | Documentation note | — | — | ## How It Works<br>This workflow automates enterprise policy compliance monitoring using AI agents to ensure organizational adherence to regulatory and internal policies. Designed for compliance officers, legal teams, and risk managers, it solves the challenge of manually reviewing vast policy documents and execution logs for violations.The system fetches policy records on schedule, routes them to specialized AI agents (OpenAI for compliance assessment and escalation logic), validates outputs, and logs all actions for audit trails. Email notifications alert stakeholders when violations occur. By automating detection and escalation, organizations reduce compliance risks, accelerate response times, and maintain comprehensive audit documentation—critical for regulated industries like finance, healthcare, and manufacturing. |
| Sticky Note1 | Sticky Note | Documentation note | — | — | ## Setup Steps<br>1. Connect **Schedule Trigger** (set monitoring frequency: hourly/daily)<br>2. Configure **Fetch Policy Records** node with your policy database/API credentials<br>3. Add **OpenAI API key** to Compliance Agent and Escalation Logic nodes<br>4. Connect **Email** node with SMTP credentials for alert notifications<br>5. Link **Final Execution Log** to your audit storage system<br>6. Test workflow with sample policy violations to verify routing logic |
| Sticky Note2 | Sticky Note | Documentation note | — | — | ## Prerequisites<br>OpenAI API account with GPT-4 access, policy database/API access<br>## Use Cases<br>Financial services regulatory compliance (KYC/AML), healthcare HIPAA monitoring<br>## Customization<br>Modify AI prompts for industry-specific regulations, adjust routing thresholds for violation severity<br>## Benefits<br>Reduces compliance review time by 90%, eliminates human oversight gaps |
| Sticky Note3 | Sticky Note | Documentation note | — | — | ## Escalation & Notification<br>**Why**: Enables rapid stakeholder response and creates immutable compliance trails |
| Sticky Note4 | Sticky Note | Documentation note | — | — | ## Intelligent Routing<br>**Why**: Prioritizes critical issues for immediate action while logging minor infractions |
| Sticky Note5 | Sticky Note | Documentation note | — | — | ## Scheduled Monitoring & AI Compliance Assessment<br>**Why**: OpenAI agent analyzes policy records against compliance rules to ensure continuous monitoring and scales expert-level policy interpretation across thousands of documents instantly |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **AI-Driven Policy Monitoring and Compliance Orchestration System** (or your preferred name).
   - Keep it **inactive** until credentials and placeholders are set.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: run at **09:00** (daily hour trigger).
   - Connect to **Workflow Configuration**.

3. **Add Workflow Configuration (Set)**
   - Node: **Set**
   - Add fields (and keep “Include Other Fields” enabled):
     - `policyTableName` (string): `employee_policy_records`
     - `complianceTableName` (string): `compliance_actions_log`
     - `auditTableName` (string): `audit_reports`
     - `executionLogTableName` (string): `workflow_execution_log`
     - `slackChannel` (string): your Slack channel ID (e.g., `C0123...`)
     - `escalationEmail` (string): escalation mailbox (not used elsewhere in this JSON, but keep for extension)
     - `complianceThresholdDays` (number): `30` (note: prompt hard-codes 30 as well; update prompt if you want this dynamic)
   - Connect to **Fetch Policy Records**.

4. **Create Data Tables in n8n (prerequisite)**
   - Create (or ensure existing) Data Tables:
     - `employee_policy_records`
     - `compliance_actions_log`
     - `audit_reports`
     - `workflow_execution_log`
   - Ensure each has appropriate columns/unique keys for **upsert** (e.g., `employeeId` + date, or an action/audit ID).

5. **Add Fetch Policy Records (Data Table)**
   - Node: **Data Table**
   - Operation: **Get**
   - Return All: **true**
   - Data Table: **By name** with expression:
     - `$('Workflow Configuration').first().json.policyTableName`
   - Connect to **Policy Monitoring Agent**.

6. **Add OpenAI Model - Policy Agent**
   - Node: **OpenAI Chat Model** (LangChain OpenAI node)
   - Model: **gpt-4o**
   - Temperature: **0.2**
   - Configure **OpenAI API credentials** (Settings → Credentials → OpenAI).
   - Connect this node to **Policy Monitoring Agent** via the **AI Language Model** connector.

7. **Add Policy Validation Output Parser**
   - Node: **Structured Output Parser**
   - Schema: manual JSON schema with required fields:
     - `employeeId`, `complianceStatus`, `riskLevel`, `requiredAction`, `reasoning`
     - plus optional fields as in the workflow.
   - Connect it to **Policy Monitoring Agent** via the **AI Output Parser** connector.

8. **Add Policy Monitoring Agent**
   - Node: **AI Agent (LangChain Agent)**
   - Prompt text:
     - `Employee Policy Data: {{ JSON.stringify($json) }}`
   - System message: paste the policy monitoring rules (compliance rules, risk levels, required action rules) as shown.
   - Ensure output parsing is enabled (agent must use the structured parser).
   - Main connection: from **Fetch Policy Records** to this agent; output to **Route by Compliance Status**.

9. **Add Route by Compliance Status (Switch)**
   - Node: **Switch**
   - Add output rule “Non-Compliant”:
     - Condition: `$json.output.complianceStatus` equals `non_compliant` OR equals `at_risk`
   - Add output rule “Compliant”:
     - Condition: `$json.output.complianceStatus` equals `compliant`
   - Fallback output renamed to “Unknown” (optional but matches JSON).
   - Connect:
     - Non-Compliant output → **Compliance Orchestration Agent**
     - Compliant output → **Prepare Compliant Records**
   - (Recommended) Also connect “Unknown” to a log/review node, though the provided workflow does not.

10. **Add OpenAI Model - Compliance Agent**
   - Node: **OpenAI Chat Model**
   - Model: **gpt-4o**
   - Temperature: **0.3**
   - Use the same OpenAI credential (or another).
   - Connect to **Compliance Orchestration Agent** via **AI Language Model**.

11. **Add Slack Notification Tool**
   - Node: **Slack Tool** (agent tool node)
   - Authentication: **OAuth2**
   - Credentials: connect Slack OAuth2 credential with `chat:write` permissions and channel access.
   - Channel selection: “channel” by **ID**
   - Channel ID expression (with default):
     - `$fromAI('slackChannel', ..., 'string', $('Workflow Configuration').first().json.slackChannel)`
   - Text expression:
     - `$fromAI('message', ..., 'string')`
   - Connect tool back to **Compliance Orchestration Agent** via the **AI Tool** connector.

12. **Add Email Notification Tool**
   - Node: **Email Send Tool**
   - Operation: **sendAndWait**
   - Configure SMTP (via n8n email credentials/settings as required by your environment).
   - From Email: set a real sender mailbox (replace placeholder).
   - To / Subject / Body:
     - `recipientEmail = $fromAI('recipientEmail', ..., 'string')`
     - `emailSubject = $fromAI('emailSubject', ..., 'string')`
     - `emailBody = $fromAI('emailBody', ..., 'string')`
   - Connect as an **AI Tool** to **Compliance Orchestration Agent**.

13. **Add Escalation Logic Tool (Code Tool)**
   - Node: **LangChain Code Tool**
   - Paste the JS escalation logic (computes escalation path, urgency, actions; returns JSON string).
   - Connect as an **AI Tool** to **Compliance Orchestration Agent**.

14. **Add Compliance Orchestration Agent**
   - Node: **AI Agent (LangChain Agent)**
   - Prompt text:
     - `Compliance Issue: {{ JSON.stringify($json.output) }}`
   - System message: paste the orchestration and notification strategy text.
   - Ensure the three tools are connected (Slack, Email, Escalation).
   - Connect output to **Log Compliance Actions**.

15. **Add Log Compliance Actions (Data Table)**
   - Node: **Data Table**
   - Operation: **Upsert**
   - Data Table by name expression:
     - `$('Workflow Configuration').first().json.complianceTableName`
   - Mapping: auto-map
   - Connect to **Merge All Actions** (input 0).

16. **Add Prepare Compliant Records (Set)**
   - Node: **Set**
   - Add:
     - `auditStatus = compliant`
     - `auditDate = {{$now.toISO()}}`
     - `auditType = automated_policy_check`
   - Keep “Include Other Fields” enabled.
   - Connect to **Store Audit Report**.

17. **Add Store Audit Report (Data Table)**
   - Node: **Data Table**
   - Operation: **Upsert**
   - Data Table by name:
     - `$('Workflow Configuration').first().json.auditTableName`
   - Connect to **Merge All Actions** (input 1).

18. **Add Merge All Actions**
   - Node: **Merge**
   - Mode: **Combine**
   - Combine By: **Position**
   - Connect output to **Final Execution Log**.

19. **Add Final Execution Log (Data Table)**
   - Node: **Data Table**
   - Operation: **Upsert**
   - Data Table by name:
     - `$('Workflow Configuration').first().json.executionLogTableName`
   - Mapping: auto-map

20. **Replace placeholders and test**
   - Replace:
     - Slack channel ID placeholder
     - escalation email placeholder (optional; not used directly)
     - email sender placeholder
   - Run with a small set of sample employee records that include:
     - training completion date
     - certification expiry date
     - policy acknowledgement timestamp
   - Validate:
     - Parser acceptance (schema)
     - Routing correctness
     - Slack + email delivery
     - Data Table upserts and merge output sanity

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works” note describing enterprise compliance monitoring, AI assessment, routing, notifications, and audit trails for regulated industries | Sticky Note content (workflow canvas) |
| Setup guidance: connect schedule, configure record source, OpenAI key, SMTP, audit storage, and test with violations | Sticky Note1 content (workflow canvas) |
| Prerequisites/use cases/customization/benefits: GPT-4 access, policy DB, finance/healthcare examples, prompt/routing tuning | Sticky Note2 content (workflow canvas) |
| Disclaimer (provided by user): *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n...* | Provided in the request; applies to the whole workflow documentation |