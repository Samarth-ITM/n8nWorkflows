Automate ESG carbon monitoring and strategy execution with GPT-4o, Slack and Sheets

https://n8nworkflows.xyz/workflows/automate-esg-carbon-monitoring-and-strategy-execution-with-gpt-4o--slack-and-sheets-14463


# Automate ESG carbon monitoring and strategy execution with GPT-4o, Slack and Sheets

# 1. Workflow Overview

This workflow implements an AI-driven carbon monitoring and ESG operations pipeline in n8n. Its purpose is to ingest emissions data from scheduled and real-time sources, analyze it with a supervisor-agent architecture powered by GPT-4o, route the result into operational branches, and then notify stakeholders through Slack, Google Sheets, email, and PostgreSQL logging.

Typical use cases include:

- Continuous carbon emissions monitoring across cloud or enterprise systems
- Automated recommendation of optimization strategies
- Policy compliance checking and violation alerting
- ESG report generation and stakeholder communication
- Human approval routing for sensitive sustainability strategies

## 1.1 Input Reception

The workflow starts from two entry points:

- A scheduled trigger running every 6 hours
- A webhook receiving real-time emissions payloads via HTTP POST

These two sources are merged into one unified processing stream.

## 1.2 AI Supervisor Orchestration

A central Carbon Supervisor Agent receives the merged data, uses GPT-4o as its language model, and can call four specialist AI tool-agents:

- Carbon Monitoring Agent
- Carbon Optimization Agent
- Policy Enforcement Agent
- ESG Reporting Agent

The supervisor is constrained by a structured output parser that forces a predictable JSON response.

## 1.3 Specialist Agent Tooling

Each specialist agent has its own GPT-4o model and, where relevant, access to tools such as:

- Custom carbon calculation code
- PostgreSQL query tool
- Calculator tool
- Google Sheets policy lookup
- Slack human-in-the-loop approval tool

This allows the supervisor to delegate focused tasks instead of solving everything in a single prompt.

## 1.4 Action Routing

Once the supervisor returns a structured action, a Switch node dispatches the flow into one of four action categories:

- `monitor`
- `optimize`
- `enforce_policy`
- `generate_report`

Each branch performs different follow-up actions.

## 1.5 Storage, Approval, Reporting, and Notifications

Depending on the selected action, the workflow may:

- Store carbon metrics in PostgreSQL
- Prepare optimization recommendations
- Ask for approval in Slack
- Auto-execute low-risk strategies
- Send policy violation alerts
- Append ESG data to Google Sheets
- Send ESG report emails
- Consolidate branch outcomes and send a completion summary
- Update a KPI dashboard table in PostgreSQL

---

# 2. Block-by-Block Analysis

## Block 1 — Data Intake and Unification

### Overview

This block receives emissions data from scheduled and real-time sources and combines them into a single stream for AI processing. It is the dual-entry ingestion layer of the workflow.

### Nodes Involved

- Scheduled Carbon Data Collection
- Real-time Emissions Data Webhook
- Merge Data Sources

### Node Details

#### 1. Scheduled Carbon Data Collection

- **Type and role:** `n8n-nodes-base.scheduleTrigger` — time-based workflow entry point.
- **Configuration:** Runs every 6 hours using an interval rule.
- **Key expressions/variables:** None.
- **Input connections:** None; this is an entry node.
- **Output connections:** Sends output to **Merge Data Sources**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failures:**
  - Workflow must be active for schedule execution.
  - No data payload is explicitly generated here, so downstream logic must tolerate minimal trigger data.
- **Sub-workflow reference:** None.

#### 2. Real-time Emissions Data Webhook

- **Type and role:** `n8n-nodes-base.webhook` — real-time HTTP entry point for incoming emissions data.
- **Configuration:**
  - Method: `POST`
  - Path: `carbon-emissions-webhook`
  - Response mode: `lastNode`
- **Key expressions/variables:** Incoming payload expected in `$json.body` or raw `$json`.
- **Input connections:** None; this is another entry node.
- **Output connections:** Sends output to **Merge Data Sources**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases/failures:**
  - Invalid or empty POST payload may reduce supervisor accuracy.
  - Because response mode is `lastNode`, webhook callers wait until full workflow completion.
  - Long AI or external service latency can cause caller timeouts.
- **Sub-workflow reference:** None.

#### 3. Merge Data Sources

- **Type and role:** `n8n-nodes-base.merge` — combines inputs from schedule and webhook into one downstream path.
- **Configuration:** Default merge behavior; no custom parameters shown.
- **Key expressions/variables:** None.
- **Input connections:**  
  - **Scheduled Carbon Data Collection**
  - **Real-time Emissions Data Webhook**
- **Output connections:** **Carbon Supervisor Agent**
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases/failures:**
  - Merge behavior depends on arrival pattern and node defaults; scheduled and webhook executions are independent, not truly simultaneous.
  - Scheduled branch may produce sparse data compared to webhook payloads, so downstream prompts should handle heterogeneous input shape.
- **Sub-workflow reference:** None.

---

## Block 2 — AI Supervisor Orchestration

### Overview

This block is the core decision engine. It receives unified emissions data, uses GPT-4o and a strict output schema, and decides which specialized agent(s) to invoke and what action should be returned.

### Nodes Involved

- Carbon Supervisor Agent
- Supervisor Model
- Supervisor Output Parser

### Node Details

#### 4. Carbon Supervisor Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agent` — top-level AI orchestrator.
- **Configuration:**
  - Input text is taken from:
    - `$json.emissions_data`
    - or `$json.body`
    - or the whole `$json`
  - Uses a system message defining the agent as a carbon sustainability supervisor coordinating monitoring, optimization, policy enforcement, and ESG reporting.
  - Output parser enabled.
- **Key expressions/variables:**
  - `={{ $json.emissions_data || $json.body || $json }}`
- **Input connections:** **Merge Data Sources**
- **Output connections:** **Route by Action Type**
- **AI side-connections:**
  - Language model: **Supervisor Model**
  - Output parser: **Supervisor Output Parser**
  - Tools:
    - **Carbon Monitoring Agent**
    - **Carbon Optimization Agent**
    - **Policy Enforcement Agent**
    - **ESG Reporting Agent**
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain/AI nodes in n8n.
- **Edge cases/failures:**
  - If upstream payload is malformed, the agent may choose the wrong action or produce incomplete structured output.
  - Tool invocation depends on model behavior; poorly shaped prompts can lead to underuse or misuse of specialist agents.
  - If parser validation fails, the node can error due to schema mismatch.
- **Sub-workflow reference:** None.

#### 5. Supervisor Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI chat model backing the supervisor.
- **Configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions/variables:** None.
- **Input connections:** None in main flow; AI-model connection to **Carbon Supervisor Agent**.
- **Output connections:** **Carbon Supervisor Agent** via `ai_languageModel`.
- **Version-specific requirements:** Type version `1.3`.
- **Credentials:** OpenAI API credential named **OpenAi account**.
- **Edge cases/failures:**
  - Invalid API key, quota issues, rate limits, or model availability errors.
  - Temperature is low for consistency, but not fully deterministic.
- **Sub-workflow reference:** None.

#### 6. Supervisor Output Parser

- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON output.
- **Configuration:** Manual JSON schema requiring:
  - `action`: one of `monitor`, `optimize`, `enforce_policy`, `generate_report`
  - `priority`: one of `critical`, `high`, `medium`, `low`
  - `carbon_metrics`: object
  - `recommendations`: array
  - `policy_violations`: array
  - `requires_approval`: boolean
- **Key expressions/variables:** None.
- **Input connections:** None in main flow; parser attached to **Carbon Supervisor Agent**.
- **Output connections:** **Carbon Supervisor Agent** via `ai_outputParser`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failures:**
  - If the model returns invalid JSON or omits required fields, parsing fails.
  - The schema does not define deep structure for arrays/objects, so downstream expressions may still fail if expected nested fields are absent.
- **Sub-workflow reference:** None.

---

## Block 3 — Specialist Agent Layer and Tools

### Overview

This block provides four specialized AI agents and their supporting tools. The supervisor can call them to perform focused monitoring, optimization, policy compliance, and reporting tasks.

### Nodes Involved

- Carbon Monitoring Agent
- Monitoring Model
- Carbon Calculations Tool
- Carbon Database Tool
- Carbon Optimization Agent
- Optimization Model
- KPI Calculator
- Approval Workflow Tool
- Policy Enforcement Agent
- Policy Model
- Policy Sheets Tool
- ESG Reporting Agent
- ESG Model

### Node Details

#### 7. Carbon Monitoring Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist tool-agent for emissions analysis and validation.
- **Configuration:**
  - Receives a delegated AI-generated task via `$fromAI('monitoring_task', ...)`
  - System message focuses on collecting emissions data, validating quality, identifying anomalies, and calculating KPIs.
- **Key expressions/variables:**
  - `={{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
- **Input connections:** None in main flow; available as AI tool to **Carbon Supervisor Agent**.
- **Output connections:** **Carbon Supervisor Agent** via `ai_tool`.
- **Attached model/tools:**
  - **Monitoring Model**
  - **Carbon Calculations Tool**
  - **Carbon Database Tool**
  - **KPI Calculator**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - Depends on the supervisor correctly passing a task.
  - Tool results may be inconsistent if database schema or AI-supplied numeric values are invalid.
- **Sub-workflow reference:** None.

#### 8. Monitoring Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model for the monitoring agent.
- **Configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Input connections:** AI-model link to **Carbon Monitoring Agent**
- **Output connections:** **Carbon Monitoring Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Credentials:** OpenAI API.
- **Edge cases/failures:** Standard OpenAI auth, quota, rate-limit, or latency issues.
- **Sub-workflow reference:** None.

#### 9. Carbon Calculations Tool

- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode` — custom JS calculation tool for carbon metrics.
- **Configuration:**
  - Uses `$fromAI()` to request:
    - `emissions_value`
    - `energy_kwh`
    - `operation`
  - Supports operations:
    - `footprint`
    - `intensity`
    - `efficiency`
    - `savings`
  - Uses constants:
    - grid intensity `0.475`
    - renewable intensity `0.05`
  - Returns `{ result, unit, operation }`
- **Key expressions/variables:**
  - `$fromAI('emissions_value', ..., 'number')`
  - `$fromAI('energy_kwh', ..., 'number', 0)`
  - `$fromAI('operation', ..., 'string')`
- **Input connections:** AI tool for **Carbon Monitoring Agent**
- **Output connections:** **Carbon Monitoring Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases/failures:**
  - `toFixed(2)` returns a string, not number.
  - Division safety only partially handled via `Math.max(energy, 1)`.
  - Negative or nonsensical AI-provided values are not validated.
- **Sub-workflow reference:** None.

#### 10. Carbon Database Tool

- **Type and role:** `n8n-nodes-base.postgresTool` — AI-callable PostgreSQL tool.
- **Configuration:**
  - Schema: `public`
  - Table not preselected
  - Tool description says it can query/update carbon emissions data, KPIs, historical trends, and policy configs
- **Key expressions/variables:** None directly shown.
- **Input connections:** AI tool for:
  - **Carbon Monitoring Agent**
  - **Carbon Optimization Agent**
- **Output connections:** back to those agents
- **Version-specific requirements:** Type version `2.6`.
- **Credentials:** PostgreSQL credential required, but not shown in exported snippet.
- **Edge cases/failures:**
  - Missing database credentials or permissions.
  - Empty table setting means the tool may rely on AI-driven table selection/querying; this requires careful DB access constraints.
- **Sub-workflow reference:** None.

#### 11. Carbon Optimization Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist agent for emissions reduction opportunities.
- **Configuration:**
  - Delegated task via `$fromAI('optimization_task', ...)`
  - System message focuses on energy consumption patterns, low-carbon alternatives, savings calculation, and prioritization.
- **Key expressions/variables:**
  - `={{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
- **Input connections:** AI tool for **Carbon Supervisor Agent**
- **Output connections:** **Carbon Supervisor Agent**
- **Attached model/tools:**
  - **Optimization Model**
  - **Carbon Database Tool**
  - **Approval Workflow Tool**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - May produce recommendations without enough historical data if DB access is weak.
  - If human approval tooling is misconfigured, governance handoff may fail.
- **Sub-workflow reference:** None.

#### 12. Optimization Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Input connections:** AI-model link to **Carbon Optimization Agent**
- **Output connections:** **Carbon Optimization Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Credentials:** OpenAI API.
- **Edge cases/failures:** Standard OpenAI issues.
- **Sub-workflow reference:** None.

#### 13. KPI Calculator

- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator` — general-purpose calculator available to AI agents.
- **Configuration:** Default configuration.
- **Input connections:** AI tool for:
  - **Carbon Monitoring Agent**
  - **Approval Workflow Tool**
- **Output connections:** those AI-capable nodes
- **Version-specific requirements:** Type version `1`.
- **Edge cases/failures:**
  - Incorrect AI-generated formulas can still yield wrong results.
- **Sub-workflow reference:** None.

#### 14. Approval Workflow Tool

- **Type and role:** `n8n-nodes-base.slackHitlTool` — Slack human-in-the-loop approval tool callable by AI.
- **Configuration:**
  - Message text is generated by AI:
    - `={{ $fromAI('approval_message', 'The approval request message') }}`
  - Auth: OAuth2
  - User target not selected in exported JSON
- **Key expressions/variables:**
  - `$fromAI('approval_message', ...)`
- **Input connections:** AI tool for **Carbon Optimization Agent**
- **Output connections:** **Carbon Optimization Agent**
- **Also receives AI tool from:** **KPI Calculator**
- **Version-specific requirements:** Type version `2.4`.
- **Credentials:** Slack OAuth2.
- **Edge cases/failures:**
  - Missing user selection may prevent message delivery.
  - Slack app scopes must support HITL operations.
  - Confusing overlap exists with the later standard Slack approval branch; both governance patterns coexist.
- **Sub-workflow reference:** None.

#### 15. Policy Enforcement Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist compliance agent.
- **Configuration:**
  - Delegated task via `$fromAI('policy_task', ...)`
  - System message focuses on checking data against green policies and reduction targets, identifying violations, and recommending corrective actions.
- **Key expressions/variables:**
  - `={{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
- **Input connections:** AI tool for **Carbon Supervisor Agent**
- **Output connections:** **Carbon Supervisor Agent**
- **Attached model/tools:**
  - **Policy Model**
  - **Policy Sheets Tool**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - Quality of policy enforcement depends heavily on Google Sheets policy structure and completeness.
- **Sub-workflow reference:** None.

#### 16. Policy Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Input connections:** AI-model link to **Policy Enforcement Agent**
- **Output connections:** **Policy Enforcement Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Credentials:** OpenAI API.
- **Edge cases/failures:** Standard OpenAI issues.
- **Sub-workflow reference:** None.

#### 17. Policy Sheets Tool

- **Type and role:** `n8n-nodes-base.googleSheetsTool` — AI-callable Google Sheets reader for policy data.
- **Configuration:**
  - Document ID blank in export
  - Sheet name blank in export
  - Tool description: read carbon reduction policies, approved strategies, and sustainability targets
- **Input connections:** AI tool for **Policy Enforcement Agent**
- **Output connections:** **Policy Enforcement Agent**
- **Version-specific requirements:** Type version `4.7`.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases/failures:**
  - Blank document/sheet settings must be completed before use.
  - OAuth permissions must allow sheet access.
  - Structured policy sheet design is critical for reliable AI interpretation.
- **Sub-workflow reference:** None.

#### 18. ESG Reporting Agent

- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist reporting agent.
- **Configuration:**
  - Delegated task via `$fromAI('reporting_task', ...)`
  - System prompt instructs creation of stakeholder-ready ESG reports and compliance narratives.
- **Key expressions/variables:**
  - `={{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
- **Input connections:** AI tool for **Carbon Supervisor Agent**
- **Output connections:** **Carbon Supervisor Agent**
- **Attached model/tools:**
  - **ESG Model**
- **Version-specific requirements:** Type version `3`.
- **Edge cases/failures:**
  - Output may be too narrative unless the supervisor schema forces enough structure.
- **Sub-workflow reference:** None.

#### 19. ESG Model

- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Input connections:** AI-model link to **ESG Reporting Agent**
- **Output connections:** **ESG Reporting Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Credentials:** OpenAI API.
- **Edge cases/failures:** Standard OpenAI issues.
- **Sub-workflow reference:** None.

---

## Block 4 — Action Routing and Monitoring Storage

### Overview

This block interprets the supervisor’s structured `action` field and routes execution into the corresponding operational branch. The monitoring branch stores carbon metrics directly into PostgreSQL.

### Nodes Involved

- Route by Action Type
- Store Carbon Metrics

### Node Details

#### 20. Route by Action Type

- **Type and role:** `n8n-nodes-base.switch` — branch router based on action type.
- **Configuration:** Four equality rules on `{{$json.action}}`:
  - `monitor`
  - `optimize`
  - `enforce_policy`
  - `generate_report`
- **Key expressions/variables:**
  - `={{ $json.action }}`
- **Input connections:** **Carbon Supervisor Agent**
- **Output connections:**
  - Rule 1 → **Store Carbon Metrics**
  - Rule 2 → **Prepare Optimization Data**
  - Rule 3 → **Send Policy Alert**
  - Rule 4 → **Update ESG Report**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - Any unexpected action value drops out of all configured branches.
  - No default branch is configured.
- **Sub-workflow reference:** None.

#### 21. Store Carbon Metrics

- **Type and role:** `n8n-nodes-base.postgres` — persists monitoring outputs.
- **Configuration:**
  - Table: `carbon_metrics`
  - Schema: `public`
  - Explicit column mapping:
    - `action`
    - `priority`
    - `timestamp` = `$now`
    - `carbon_metrics` = stringified JSON
    - `recommendations` = stringified JSON
    - `policy_violations` = stringified JSON
    - `requires_approval`
- **Key expressions/variables:**
  - `={{ $json.action }}`
  - `={{ $json.priority }}`
  - `={{ $now }}`
  - `={{ JSON.stringify($json.carbon_metrics) }}`
  - `={{ JSON.stringify($json.recommendations) }}`
  - `={{ JSON.stringify($json.policy_violations) }}`
  - `={{ $json.requires_approval }}`
- **Input connections:** **Route by Action Type**
- **Output connections:** **Consolidate All Actions**
- **Version-specific requirements:** Type version `2.6`.
- **Credentials:** PostgreSQL credential required.
- **Edge cases/failures:**
  - DB table must exist with compatible column types.
  - JSON stringification assumes arrays/objects are present.
- **Sub-workflow reference:** None.

---

## Block 5 — Optimization Approval Routing

### Overview

This block prepares optimization data, checks whether approval is needed, then either requests approval in Slack or auto-executes the strategy by logging it to PostgreSQL.

### Nodes Involved

- Prepare Optimization Data
- Check Approval Required
- Request Strategy Approval
- Log Approved Strategy
- Auto-Execute Strategy

### Node Details

#### 22. Prepare Optimization Data

- **Type and role:** `n8n-nodes-base.set` — reshapes optimization output for approval logic.
- **Configuration:** Creates four fields:
  - `optimization_summary` from `$json.output.recommendations`
  - `carbon_savings` from `$json.output.carbon_metrics.potential_savings`
  - `priority` from `$json.output.priority`
  - `requires_approval` from `$json.output.requires_approval`
- **Key expressions/variables:**
  - `={{ $json.output.recommendations }}`
  - `={{ $json.output.carbon_metrics.potential_savings }}`
  - `={{ $json.output.priority }}`
  - `={{ $json.output.requires_approval }}`
- **Input connections:** **Route by Action Type**
- **Output connections:** **Check Approval Required**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases/failures:**
  - This node expects data under `$json.output.*`, but the supervisor parser appears to emit top-level fields. If the actual output is top-level, this mapping will fail or produce `undefined`.
  - `optimization_summary` is typed as string, but later used like an array in Slack message formatting.
- **Sub-workflow reference:** None.

#### 23. Check Approval Required

- **Type and role:** `n8n-nodes-base.if` — decides whether optimization strategy needs human sign-off.
- **Configuration:** True if `{{$json.requires_approval}}` is boolean true.
- **Key expressions/variables:**
  - `={{ $json.requires_approval }}`
- **Input connections:** **Prepare Optimization Data**
- **Output connections:**
  - True → **Request Strategy Approval**
  - False → **Auto-Execute Strategy**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases/failures:**
  - Loose validation may still pass odd values; however condition explicitly checks boolean true.
- **Sub-workflow reference:** None.

#### 24. Request Strategy Approval

- **Type and role:** `n8n-nodes-base.slack` — sends interactive approval request and waits.
- **Configuration:**
  - Operation: `sendAndWait`
  - Channel-based Slack destination
  - Approval type: `double`
  - Disapprove label: `Reject`
  - Message includes priority, estimated savings, and bullet list of recommendations
- **Key expressions/variables:**
  - `{{ $json.priority }}`
  - `{{ $json.carbon_savings }}`
  - `{{ $json.optimization_summary.map(...) }}`
- **Input connections:** **Check Approval Required** (true branch)
- **Output connections:** **Log Approved Strategy**
- **Version-specific requirements:** Type version `2.4`.
- **Credentials:** Slack OAuth2.
- **Edge cases/failures:**
  - If `optimization_summary` is not an array, `.map()` fails.
  - Channel placeholder must be replaced.
  - Approval workflow depends on Slack app scopes and supported interactive features.
  - The downstream path logs approval but does not explicitly branch on rejection outcome.
- **Sub-workflow reference:** None.

#### 25. Log Approved Strategy

- **Type and role:** `n8n-nodes-base.postgres` — records approved strategies.
- **Configuration:**
  - Table: `approved_strategies`
  - Writes:
    - `status = approved`
    - `priority`
    - `strategy` as JSON string
    - `timestamp`
    - `approved_by = $json.user`
    - `carbon_savings`
- **Key expressions/variables:**
  - `={{ $json.priority }}`
  - `={{ JSON.stringify($json.optimization_summary) }}`
  - `={{ $now }}`
  - `={{ $json.user }}`
  - `={{ $json.carbon_savings }}`
- **Input connections:** **Request Strategy Approval**
- **Output connections:** **Consolidate All Actions**
- **Version-specific requirements:** Type version `2.6`.
- **Credentials:** PostgreSQL.
- **Edge cases/failures:**
  - Rejection path is not handled separately.
  - `user` field depends on Slack response payload structure.
- **Sub-workflow reference:** None.

#### 26. Auto-Execute Strategy

- **Type and role:** `n8n-nodes-base.postgres` — records non-approved/low-risk strategies as auto-executed.
- **Configuration:**
  - Table: `approved_strategies`
  - Writes:
    - `status = auto_executed`
    - `priority`
    - `strategy`
    - `timestamp`
    - `carbon_savings`
- **Key expressions/variables:**
  - `={{ $json.priority }}`
  - `={{ JSON.stringify($json.optimization_summary) }}`
  - `={{ $now }}`
  - `={{ $json.carbon_savings }}`
- **Input connections:** **Check Approval Required** (false branch)
- **Output connections:** **Consolidate All Actions**
- **Version-specific requirements:** Type version `2.6`.
- **Credentials:** PostgreSQL.
- **Edge cases/failures:**
  - This logs execution but does not actually call any infrastructure automation endpoint; “execute” here effectively means “record as auto-executed”.
- **Sub-workflow reference:** None.

---

## Block 6 — Policy Alerting

### Overview

This block handles policy enforcement outcomes by sending a formatted Slack alert summarizing detected violations and recommended actions.

### Nodes Involved

- Send Policy Alert

### Node Details

#### 27. Send Policy Alert

- **Type and role:** `n8n-nodes-base.slack` — sends violation alert to a Slack channel.
- **Configuration:**
  - Sends message to selected channel
  - Text contains:
    - priority
    - count of policy violations
    - formatted list of violations
    - formatted recommendations
- **Key expressions/variables:**
  - `{{ $json.output.priority }}`
  - `{{ $json.output.policy_violations.length }}`
  - `{{ $json.output.policy_violations.map(...) }}`
  - `{{ $json.output.recommendations.map(...) }}`
- **Input connections:** **Route by Action Type**
- **Output connections:** **Consolidate All Actions**
- **Version-specific requirements:** Type version `2.4`.
- **Credentials:** Slack OAuth2.
- **Edge cases/failures:**
  - Like optimization branch, this expects `$json.output.*`; if parsed output is top-level, expressions fail.
  - Placeholder channel ID must be replaced.
  - Empty arrays are fine, but undefined arrays will break `.map()` and `.length`.
- **Sub-workflow reference:** None.

---

## Block 7 — ESG Reporting Delivery

### Overview

This block appends generated ESG data to Google Sheets and then sends an HTML summary email to stakeholders.

### Nodes Involved

- Update ESG Report
- Send ESG Report Email

### Node Details

#### 28. Update ESG Report

- **Type and role:** `n8n-nodes-base.googleSheets` — appends reporting data to a spreadsheet.
- **Configuration:**
  - Operation: `append`
  - Document ID blank in export
  - Sheet name blank in export
- **Key expressions/variables:** No explicit field mapping shown; it appends incoming JSON.
- **Input connections:** **Route by Action Type**
- **Output connections:** **Send ESG Report Email**
- **Version-specific requirements:** Type version `4.7`.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases/failures:**
  - Spreadsheet and sheet name must be configured.
  - Appending raw incoming structure may create inconsistent columns if JSON shape varies.
- **Sub-workflow reference:** None.

#### 29. Send ESG Report Email

- **Type and role:** `n8n-nodes-base.emailSend` — sends stakeholder-facing HTML report email.
- **Configuration:**
  - Subject includes current date via Luxon formatting
  - HTML body includes:
    - total emissions
    - carbon intensity
    - compliance rate
    - bullet list of recommendations
  - Static placeholder sender and recipient emails
- **Key expressions/variables:**
  - `{{ $json.carbon_metrics.total_emissions }}`
  - `{{ $json.carbon_metrics.carbon_intensity }}`
  - `{{ $json.carbon_metrics.compliance_rate }}`
  - `{{ $json.recommendations.map(...) }}`
  - `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Input connections:** **Update ESG Report**
- **Output connections:** **Consolidate All Actions**
- **Version-specific requirements:** Type version `2.1`.
- **Credentials:** Depends on n8n email transport setup or node-level mail config.
- **Edge cases/failures:**
  - Sender/recipient placeholders must be replaced.
  - Recommendations must be an array.
  - Mail server configuration is not included in the workflow JSON and must exist in n8n.
- **Sub-workflow reference:** None.

---

## Block 8 — Consolidation, Notification, and KPI Logging

### Overview

This block merges all branch completions, sends a final Slack summary, and records a KPI dashboard entry for the workflow run.

### Nodes Involved

- Consolidate All Actions
- Send Summary Notification
- Update KPI Dashboard

### Node Details

#### 30. Consolidate All Actions

- **Type and role:** `n8n-nodes-base.merge` — combines up to five branch outcomes.
- **Configuration:** `numberInputs = 5`
- **Input connections:**
  - **Store Carbon Metrics**
  - **Log Approved Strategy**
  - **Auto-Execute Strategy**
  - **Send Policy Alert**
  - **Send ESG Report Email**
- **Output connections:** **Send Summary Notification**
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases/failures:**
  - In most runs, only one branch path is likely active, so merge semantics should be validated.
  - If configured as wait-for-all under defaults, it may hang waiting for inactive branches. This is an important implementation risk with multi-branch exclusive routing.
- **Sub-workflow reference:** None.

#### 31. Send Summary Notification

- **Type and role:** `n8n-nodes-base.slack` — sends completion notice.
- **Configuration:**
  - Posts to Slack channel
  - Reports action, priority, and success status
- **Key expressions/variables:**
  - `{{ $json.action || 'Multiple actions' }}`
  - `{{ $json.priority || 'N/A' }}`
- **Input connections:** **Consolidate All Actions**
- **Output connections:** **Update KPI Dashboard**
- **Version-specific requirements:** Type version `2.4`.
- **Credentials:** Slack OAuth2.
- **Edge cases/failures:**
  - Placeholder channel ID must be replaced.
  - Depending on merge behavior, fields may not always be present as expected.
- **Sub-workflow reference:** None.

#### 32. Update KPI Dashboard

- **Type and role:** `n8n-nodes-base.postgres` — stores workflow completion KPI.
- **Configuration:**
  - Table: `kpi_dashboard`
  - Writes:
    - `status = completed`
    - `timestamp = $now`
    - `total_actions = $json.action ? 1 : 0`
    - `workflow_run_id = $execution.id`
- **Key expressions/variables:**
  - `={{ $now }}`
  - `={{ $json.action ? 1 : 0 }}`
  - `={{ $execution.id }}`
- **Input connections:** **Send Summary Notification**
- **Output connections:** None.
- **Version-specific requirements:** Type version `2.6`.
- **Credentials:** PostgreSQL.
- **Edge cases/failures:**
  - If merge strips or alters `action`, `total_actions` may be incorrect.
  - Dashboard table must already exist.
- **Sub-workflow reference:** None.

---

## Block 9 — Documentation Sticky Notes

### Overview

These nodes are non-executable annotations embedded in the canvas. They describe setup, purpose, prerequisites, governance logic, and reporting intent.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### 33. Sticky Note

- **Type and role:** `n8n-nodes-base.stickyNote` — visual documentation.
- **Content summary:** Explains end-to-end workflow behavior from ingestion through AI supervision to approvals and reporting.
- **Connections:** None.
- **Version:** `1`.

#### 34. Sticky Note1

- **Type and role:** `n8n-nodes-base.stickyNote`
- **Content summary:** Lists setup steps for triggers, credentials, LLM, approval thresholds, and Google Sheets mapping.
- **Connections:** None.
- **Version:** `1`.

#### 35. Sticky Note2

- **Type and role:** `n8n-nodes-base.stickyNote`
- **Content summary:** Lists prerequisites, use cases, customization ideas, and benefits.
- **Connections:** None.
- **Version:** `1`.

#### 36. Sticky Note3

- **Type and role:** `n8n-nodes-base.stickyNote`
- **Content summary:** Describes strategy and policy evaluation zone.
- **Connections:** None.
- **Version:** `1`.

#### 37. Sticky Note4

- **Type and role:** `n8n-nodes-base.stickyNote`
- **Content summary:** Describes supervisor orchestration zone.
- **Connections:** None.
- **Version:** `1`.

#### 38. Sticky Note5

- **Type and role:** `n8n-nodes-base.stickyNote`
- **Content summary:** Describes approval routing zone.
- **Connections:** None.
- **Version:** `1`.

#### 39. Sticky Note6

- **Type and role:** `n8n-nodes-base.stickyNote`
- **Content summary:** Describes reporting and notification zone.
- **Connections:** None.
- **Version:** `1`.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Carbon Data Collection | scheduleTrigger | Time-based trigger every 6 hours |  | Merge Data Sources | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Real-time Emissions Data Webhook | webhook | Receives POST emissions events |  | Merge Data Sources | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Merge Data Sources | merge | Unifies schedule and webhook input streams | Scheduled Carbon Data Collection, Real-time Emissions Data Webhook | Carbon Supervisor Agent | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Carbon Supervisor Agent | @n8n/n8n-nodes-langchain.agent | Main AI orchestrator deciding action and delegating to specialist agents | Merge Data Sources | Route by Action Type | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Supervisor Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for supervisor agent |  | Carbon Supervisor Agent | ## Setup Steps<br>1. Connect scheduled trigger and webhook nodes to your emissions data sources.<br>2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account).<br>3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible).<br>4. Set approval thresholds in the Check Approval Required node.<br>5. Map Google Sheets document ID for ESG report and KPI dashboard nodes.<br><br>## Prerequisites<br>- OpenAI or compatible LLM API key<br>- Slack bot token<br>- Gmail OAuth2 credentials<br>- Google Sheets service account<br>## Use Cases<br>- Corporate sustainability teams automating monthly ESG reporting<br>## Customisation<br>- Swap LLM models per agent for cost or accuracy trade-offs<br>## Benefits<br>- Eliminates manual emissions data aggregation and report generation |
| Supervisor Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output from supervisor |  | Carbon Supervisor Agent |  |
| Carbon Monitoring Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for emissions monitoring and KPI validation |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Monitoring Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for monitoring agent |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Optimization Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for carbon reduction strategy generation |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Optimization Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for optimization agent |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Enforcement Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for policy compliance checking |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for policy agent |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Reporting Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist agent for ESG report generation |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for ESG reporting agent |  | ESG Reporting Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Calculations Tool | @n8n/n8n-nodes-langchain.toolCode | JS tool for carbon footprint, intensity, efficiency, and savings calculations |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| KPI Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Generic calculator tool for AI-assisted numeric operations |  | Carbon Monitoring Agent, Approval Workflow Tool | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Database Tool | n8n-nodes-base.postgresTool | AI-callable PostgreSQL access for emissions history and KPI data |  | Carbon Monitoring Agent, Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Sheets Tool | n8n-nodes-base.googleSheetsTool | AI-callable Google Sheets reader for policy/target definitions |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Approval Workflow Tool | n8n-nodes-base.slackHitlTool | AI-callable Slack human approval tool |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Route by Action Type | switch | Routes supervisor output into operational branches | Carbon Supervisor Agent | Store Carbon Metrics, Prepare Optimization Data, Send Policy Alert, Update ESG Report | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Store Carbon Metrics | postgres | Stores monitoring results in PostgreSQL | Route by Action Type | Consolidate All Actions | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Prepare Optimization Data | set | Reshapes optimization output for approval decision | Route by Action Type | Check Approval Required | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Check Approval Required | if | Branches based on approval requirement boolean | Prepare Optimization Data | Request Strategy Approval, Auto-Execute Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Request Strategy Approval | slack | Sends Slack approval request and waits for response | Check Approval Required | Log Approved Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Log Approved Strategy | postgres | Records approved optimization strategies | Request Strategy Approval | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Auto-Execute Strategy | postgres | Logs non-approved strategies as auto-executed | Check Approval Required | Consolidate All Actions | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Send Policy Alert | slack | Sends Slack alert for policy violations | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update ESG Report | googleSheets | Appends ESG reporting data to Google Sheets | Route by Action Type | Send ESG Report Email | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send ESG Report Email | emailSend | Sends stakeholder ESG summary email | Update ESG Report | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Consolidate All Actions | merge | Merges branch completions before final notification | Store Carbon Metrics, Log Approved Strategy, Auto-Execute Strategy, Send Policy Alert, Send ESG Report Email | Send Summary Notification | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send Summary Notification | slack | Sends final Slack completion summary | Consolidate All Actions | Update KPI Dashboard | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update KPI Dashboard | postgres | Logs workflow completion KPI to PostgreSQL | Send Summary Notification |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Sticky Note | stickyNote | Canvas documentation note |  |  | ## How It Works<br>This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. |
| Sticky Note1 | stickyNote | Canvas setup note |  |  | ## Setup Steps<br>1. Connect scheduled trigger and webhook nodes to your emissions data sources.<br>2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account).<br>3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible).<br>4. Set approval thresholds in the Check Approval Required node.<br>5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. |
| Sticky Note2 | stickyNote | Canvas prerequisites/customization note |  |  | ## Prerequisites<br>- OpenAI or compatible LLM API key<br>- Slack bot token<br>- Gmail OAuth2 credentials<br>- Google Sheets service account<br>## Use Cases<br>- Corporate sustainability teams automating monthly ESG reporting<br>## Customisation<br>- Swap LLM models per agent for cost or accuracy trade-offs<br>## Benefits<br>- Eliminates manual emissions data aggregation and report generation |
| Sticky Note3 | stickyNote | Canvas strategy/policy zone note |  |  | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Sticky Note4 | stickyNote | Canvas orchestration zone note |  |  | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Sticky Note5 | stickyNote | Canvas approval-routing note |  |  | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Sticky Note6 | stickyNote | Canvas reporting/notification note |  |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence for n8n.

## Preparation

1. **Create required credentials in n8n**
   - OpenAI API credential for GPT-4o nodes
   - Slack OAuth2 credential for Slack messaging and HITL approval
   - Google Sheets OAuth2 credential
   - PostgreSQL credential
   - Email transport configuration or email credential supported by your n8n setup

2. **Prepare external resources**
   - PostgreSQL tables:
     - `carbon_metrics`
     - `approved_strategies`
     - `kpi_dashboard`
   - Google Sheets document for:
     - ESG report append target
     - policy/target source sheet
   - Slack channels for:
     - policy alerts
     - strategy approvals
     - summary notifications

3. **Enable AI nodes**
   - Ensure your n8n version supports LangChain AI nodes used here:
     - Agent
     - Agent Tool
     - OpenAI Chat Model
     - Structured Output Parser
     - Code Tool
     - Calculator Tool

---

## Build Steps

### Entry and merge layer

1. **Add a Schedule Trigger node**
   - Name: `Scheduled Carbon Data Collection`
   - Configure interval: every `6` hours

2. **Add a Webhook node**
   - Name: `Real-time Emissions Data Webhook`
   - HTTP Method: `POST`
   - Path: `carbon-emissions-webhook`
   - Response Mode: `Last Node`

3. **Add a Merge node**
   - Name: `Merge Data Sources`
   - Leave default merge settings unless you want a specific mode

4. **Connect**
   - `Scheduled Carbon Data Collection` → `Merge Data Sources`
   - `Real-time Emissions Data Webhook` → `Merge Data Sources`

---

### Supervisor AI layer

5. **Add an AI Agent node**
   - Name: `Carbon Supervisor Agent`
   - Prompt/input text:
     - `{{ $json.emissions_data || $json.body || $json }}`
   - Enable structured output parser
   - System message:
     - Define the agent as the Carbon Sustainability Supervisor Agent
     - Instruct it to analyze emissions data, delegate to specialized sub-agents, and coordinate strategy execution
     - Mention the four action families: monitoring, optimization, policy enforcement, ESG reporting

6. **Add an OpenAI Chat Model node**
   - Name: `Supervisor Model`
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach OpenAI credential

7. **Add a Structured Output Parser node**
   - Name: `Supervisor Output Parser`
   - Manual schema with fields:
     - `action` enum: `monitor`, `optimize`, `enforce_policy`, `generate_report`
     - `priority` enum: `critical`, `high`, `medium`, `low`
     - `carbon_metrics` object
     - `recommendations` array
     - `policy_violations` array
     - `requires_approval` boolean

8. **Connect AI links**
   - `Supervisor Model` → `Carbon Supervisor Agent` as language model
   - `Supervisor Output Parser` → `Carbon Supervisor Agent` as output parser

9. **Connect main flow**
   - `Merge Data Sources` → `Carbon Supervisor Agent`

---

### Monitoring specialist agent

10. **Add an AI Agent Tool node**
    - Name: `Carbon Monitoring Agent`
    - Tool input text:
      - `{{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
    - System message:
      - Collect emissions data
      - Validate data quality
      - Identify anomalies
      - Calculate carbon KPIs
      - Use tools for historical context

11. **Add an OpenAI Chat Model node**
    - Name: `Monitoring Model`
    - Model: `gpt-4o`
    - Temperature: `0.1`

12. **Add a Code Tool node**
    - Name: `Carbon Calculations Tool`
    - Description:
      - Performs carbon footprint, intensity, efficiency, and savings calculations
    - Paste the JavaScript logic equivalent to:
      - Inputs from AI:
        - emissions value
        - energy kWh
        - operation
      - Return result, unit, and operation

13. **Add a Postgres Tool node**
    - Name: `Carbon Database Tool`
    - Schema: `public`
    - Description:
      - Query/update carbon emissions, KPI trends, and policy configs
    - Attach PostgreSQL credential

14. **Add a Calculator Tool node**
    - Name: `KPI Calculator`

15. **Connect AI links**
    - `Monitoring Model` → `Carbon Monitoring Agent`
    - `Carbon Calculations Tool` → `Carbon Monitoring Agent`
    - `Carbon Database Tool` → `Carbon Monitoring Agent`
    - `KPI Calculator` → `Carbon Monitoring Agent`

---

### Optimization specialist agent

16. **Add an AI Agent Tool node**
    - Name: `Carbon Optimization Agent`
    - Tool input text:
      - `{{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
    - System message:
      - Analyze consumption patterns
      - Identify high-carbon workloads
      - Recommend low-carbon alternatives
      - Estimate savings
      - Prioritize opportunities

17. **Add an OpenAI Chat Model node**
    - Name: `Optimization Model`
    - Model: `gpt-4o`
    - Temperature: `0.3`

18. **Add a Slack Human-in-the-Loop Tool node**
    - Name: `Approval Workflow Tool`
    - Authentication: Slack OAuth2
    - Message:
      - `{{ $fromAI('approval_message', 'The approval request message') }}`
    - Select a user or configure as appropriate for your Slack setup

19. **Connect AI links**
    - `Optimization Model` → `Carbon Optimization Agent`
    - `Carbon Database Tool` → `Carbon Optimization Agent`
    - `Approval Workflow Tool` → `Carbon Optimization Agent`

20. **Optional parity with original**
    - Connect `KPI Calculator` → `Approval Workflow Tool` as AI tool
    - This mirrors the export, though it is not essential to the main branch logic

---

### Policy specialist agent

21. **Add an AI Agent Tool node**
    - Name: `Policy Enforcement Agent`
    - Tool input text:
      - `{{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
    - System message:
      - Compare emissions data with green policies and reduction targets
      - Identify violations
      - Recommend corrective actions
      - Determine escalation/approval needs

22. **Add an OpenAI Chat Model node**
    - Name: `Policy Model`
    - Model: `gpt-4o`
    - Temperature: `0.1`

23. **Add a Google Sheets Tool node**
    - Name: `Policy Sheets Tool`
    - Attach Google Sheets OAuth2 credential
    - Select:
      - document ID
      - sheet name
    - Description:
      - Read carbon reduction policies, approved strategies, and sustainability targets

24. **Connect AI links**
    - `Policy Model` → `Policy Enforcement Agent`
    - `Policy Sheets Tool` → `Policy Enforcement Agent`

---

### ESG reporting specialist agent

25. **Add an AI Agent Tool node**
    - Name: `ESG Reporting Agent`
    - Tool input text:
      - `{{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
    - System message:
      - Generate ESG reports with carbon metrics, KPIs, achievements, compliance, and trend analysis

26. **Add an OpenAI Chat Model node**
    - Name: `ESG Model`
    - Model: `gpt-4o`
    - Temperature: `0.2`

27. **Connect**
    - `ESG Model` → `ESG Reporting Agent`

---

### Attach specialist tools to supervisor

28. **Connect AI tool links into the supervisor**
   - `Carbon Monitoring Agent` → `Carbon Supervisor Agent`
   - `Carbon Optimization Agent` → `Carbon Supervisor Agent`
   - `Policy Enforcement Agent` → `Carbon Supervisor Agent`
   - `ESG Reporting Agent` → `Carbon Supervisor Agent`

---

### Action router

29. **Add a Switch node**
   - Name: `Route by Action Type`
   - Create four rules based on `{{ $json.action }}`:
     - equals `monitor`
     - equals `optimize`
     - equals `enforce_policy`
     - equals `generate_report`

30. **Connect**
   - `Carbon Supervisor Agent` → `Route by Action Type`

---

### Monitoring branch

31. **Add a Postgres node**
   - Name: `Store Carbon Metrics`
   - Table: `carbon_metrics`
   - Schema: `public`
   - Map fields:
     - `action` = `{{ $json.action }}`
     - `priority` = `{{ $json.priority }}`
     - `timestamp` = `{{ $now }}`
     - `carbon_metrics` = `{{ JSON.stringify($json.carbon_metrics) }}`
     - `recommendations` = `{{ JSON.stringify($json.recommendations) }}`
     - `policy_violations` = `{{ JSON.stringify($json.policy_violations) }}`
     - `requires_approval` = `{{ $json.requires_approval }}`

32. **Connect**
   - `Route by Action Type` monitor output → `Store Carbon Metrics`

---

### Optimization branch

33. **Add a Set node**
   - Name: `Prepare Optimization Data`
   - Create fields:
     - `optimization_summary`
     - `carbon_savings`
     - `priority`
     - `requires_approval`

34. **Important implementation note**
   - The original workflow uses `{{$json.output...}}`
   - In many n8n AI setups, the structured parser output is top-level, so a safer mapping is often:
     - `optimization_summary = {{ $json.recommendations }}`
     - `carbon_savings = {{ $json.carbon_metrics.potential_savings }}`
     - `priority = {{ $json.priority }}`
     - `requires_approval = {{ $json.requires_approval }}`
   - If your agent returns nested `output`, then use the original mapping

35. **Add an If node**
   - Name: `Check Approval Required`
   - Condition:
     - boolean true on `{{ $json.requires_approval }}`

36. **Add a Slack node**
   - Name: `Request Strategy Approval`
   - Authentication: Slack OAuth2
   - Operation: `sendAndWait`
   - Destination: approval channel
   - Approval type: `double`
   - Disapprove label: `Reject`
   - Message should include:
     - priority
     - carbon savings
     - bullet recommendations

37. **Add a Postgres node**
   - Name: `Log Approved Strategy`
   - Table: `approved_strategies`
   - Columns:
     - `status = approved`
     - `priority = {{ $json.priority }}`
     - `strategy = {{ JSON.stringify($json.optimization_summary) }}`
     - `timestamp = {{ $now }}`
     - `approved_by = {{ $json.user }}`
     - `carbon_savings = {{ $json.carbon_savings }}`

38. **Add another Postgres node**
   - Name: `Auto-Execute Strategy`
   - Table: `approved_strategies`
   - Columns:
     - `status = auto_executed`
     - `priority = {{ $json.priority }}`
     - `strategy = {{ JSON.stringify($json.optimization_summary) }}`
     - `timestamp = {{ $now }}`
     - `carbon_savings = {{ $json.carbon_savings }}`

39. **Connect**
   - `Route by Action Type` optimize output → `Prepare Optimization Data`
   - `Prepare Optimization Data` → `Check Approval Required`
   - `Check Approval Required` true → `Request Strategy Approval`
   - `Request Strategy Approval` → `Log Approved Strategy`
   - `Check Approval Required` false → `Auto-Execute Strategy`

---

### Policy enforcement branch

40. **Add a Slack node**
   - Name: `Send Policy Alert`
   - Authentication: Slack OAuth2
   - Destination: policy alerts channel
   - Build message with:
     - priority
     - violation count
     - formatted list of policy violations
     - formatted recommended actions

41. **Important implementation note**
   - The original workflow uses `{{$json.output.priority}}` and similar nested fields.
   - If your parsed supervisor output is top-level, use:
     - `{{ $json.priority }}`
     - `{{ $json.policy_violations.length }}`
     - `{{ $json.policy_violations.map(...) }}`
     - `{{ $json.recommendations.map(...) }}`

42. **Connect**
   - `Route by Action Type` enforce_policy output → `Send Policy Alert`

---

### ESG reporting branch

43. **Add a Google Sheets node**
   - Name: `Update ESG Report`
   - Operation: `append`
   - Select ESG tracking spreadsheet
   - Select target sheet

44. **Add an Email Send node**
   - Name: `Send ESG Report Email`
   - Subject:
     - `ESG Sustainability Report - {{ $now.toFormat('yyyy-MM-dd') }}`
   - HTML body should include:
     - total emissions
     - carbon intensity
     - compliance rate
     - bullet list of recommendations
   - Set sender email and stakeholder recipient email

45. **Connect**
   - `Route by Action Type` generate_report output → `Update ESG Report`
   - `Update ESG Report` → `Send ESG Report Email`

---

### Consolidation and completion logging

46. **Add a Merge node**
   - Name: `Consolidate All Actions`
   - Number of inputs: `5`

47. **Important implementation note**
   - The original design routes mutually exclusive branches into one merge.
   - To avoid waiting forever for non-executed branches, verify merge mode carefully.
   - If needed, use a merge mode that does not require all branches, or redesign with separate terminal notifications.

48. **Connect to the merge**
   - `Store Carbon Metrics` → input 1
   - `Log Approved Strategy` → input 2
   - `Auto-Execute Strategy` → input 3
   - `Send Policy Alert` → input 4
   - `Send ESG Report Email` → input 5

49. **Add a Slack node**
   - Name: `Send Summary Notification`
   - Destination: summary channel
   - Message should include:
     - action or “Multiple actions”
     - priority or “N/A”
     - completed status

50. **Add a Postgres node**
   - Name: `Update KPI Dashboard`
   - Table: `kpi_dashboard`
   - Map fields:
     - `status = completed`
     - `timestamp = {{ $now }}`
     - `total_actions = {{ $json.action ? 1 : 0 }}`
     - `workflow_run_id = {{ $execution.id }}`

51. **Connect**
   - `Consolidate All Actions` → `Send Summary Notification`
   - `Send Summary Notification` → `Update KPI Dashboard`

---

### Final validation

52. **Replace placeholders**
   - Slack channel IDs
   - stakeholder email
   - sender email
   - Google Sheets document IDs and sheet names

53. **Test each entry point separately**
   - Trigger manual test from webhook with sample JSON
   - Validate scheduled run behavior
   - Confirm the supervisor produces one of the four valid actions

54. **Check schema compatibility**
   - Ensure downstream nodes expect either top-level parser fields or nested `output` fields consistently
   - This is the most important correction likely needed from the exported version

55. **Activate the workflow**
   - Webhook URL becomes live only when appropriate for your environment
   - Schedule trigger runs only when workflow is active

---

## Suggested sample webhook payload

A practical payload for testing could include:

- source system identifier
- timestamp
- total emissions
- energy consumption
- region or cloud provider
- service breakdown
- policy tags
- anomaly flags

Example shape to emulate:
- `emissions_data.total_emissions`
- `emissions_data.energy_kwh`
- `emissions_data.provider`
- `emissions_data.region`
- `emissions_data.services`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. | Workflow canvas note |
| 1. Connect scheduled trigger and webhook nodes to your emissions data sources. 2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account). 3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible). 4. Set approval thresholds in the Check Approval Required node. 5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. | Setup guidance from canvas note |
| Prerequisites: OpenAI or compatible LLM API key; Slack bot token; Gmail OAuth2 credentials; Google Sheets service account. | Canvas prerequisites note |
| Use case: Corporate sustainability teams automating monthly ESG reporting. | Canvas note |
| Customisation: Swap LLM models per agent for cost or accuracy trade-offs. | Canvas note |
| Benefit: Eliminates manual emissions data aggregation and report generation. | Canvas note |
| Strategy & Policy Evaluation — What: Optimisation and Policy Enforcement Agents propose and validate actions. Why: Keeps reduction strategies compliant before execution. | Canvas note |
| Supervisor Orchestration — What: Carbon Supervisor Agent delegates tasks across four specialised sub-agents. Why: Parallel processing improves speed and separates concerns cleanly. | Canvas note |
| Approval Routing — What: Routes strategies requiring sign-off to Slack; auto-executes low-risk ones. Why: Balances automation with governance oversight. | Canvas note |
| Reporting & Notification — What: Updates ESG report, sends email summary, refreshes KPI dashboard. Why: Closes the feedback loop for stakeholders in real time. | Canvas note |

## Additional implementation cautions

- The workflow mixes top-level AI parser output expectations with `$json.output.*` references. This should be normalized before production use.
- The consolidation merge may require redesign if only one branch executes per run.
- Several required configuration values are placeholders or blank:
  - Slack channel IDs
  - Google Sheets document IDs and sheet names
  - sender and recipient emails
  - some Slack HITL user selection
  - PostgreSQL credentials for tool and storage nodes
- “Auto-Execute Strategy” currently logs execution rather than performing real infrastructure changes. If true execution is needed, add cloud/API action nodes after approval logic.