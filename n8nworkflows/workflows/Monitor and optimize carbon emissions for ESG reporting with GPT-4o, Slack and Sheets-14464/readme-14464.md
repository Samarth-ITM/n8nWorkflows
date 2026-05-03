Monitor and optimize carbon emissions for ESG reporting with GPT-4o, Slack and Sheets

https://n8nworkflows.xyz/workflows/monitor-and-optimize-carbon-emissions-for-esg-reporting-with-gpt-4o--slack-and-sheets-14464


# Monitor and optimize carbon emissions for ESG reporting with GPT-4o, Slack and Sheets

# AI carbon supervisor agent for ESG monitoring and strategy execution

## 1. Workflow Overview

This workflow automates carbon emissions monitoring, optimization decisioning, policy compliance checks, and ESG reporting through a supervisor-style AI architecture in n8n. It is designed for sustainability managers, ESG teams, and operations leaders who need to collect emissions-related data, interpret it with AI, decide whether action is needed, route approvals when required, and distribute reporting outputs to stakeholders.

At a high level, the workflow starts on a schedule, sends incoming emissions payloads to a central **Carbon Supervisor Agent**, and lets that supervisor invoke specialized AI sub-agents for monitoring, optimization, policy enforcement, or reporting. The supervisor returns a structured decision payload, which is then routed into one of several action branches: storage, approval handling, Slack alerts, Google Sheets reporting, email notification, and KPI dashboard updates.

### 1.1 Trigger and Input Reception
The workflow currently has one active entry point: a scheduled trigger that runs every 6 hours. The workflow description mentions webhook-based real-time input, but no webhook trigger node is present in the JSON.

### 1.2 Supervisor Orchestration
A central LangChain agent receives the input payload and decides what kind of action is needed: monitoring, optimization, policy enforcement, or ESG report generation. It uses a structured output parser so downstream routing can rely on deterministic fields.

### 1.3 Specialized AI Tools and Models
The supervisor can call four specialized agent tools:
- Carbon Monitoring Agent
- Carbon Optimization Agent
- Policy Enforcement Agent
- ESG Reporting Agent

These agents each have their own GPT-4o model connection and, where applicable, supporting tools such as a code calculator, PostgreSQL tool, Google Sheets tool, calculator tool, and Slack human-in-the-loop tool.

### 1.4 Action Routing and Decision Execution
The structured supervisor output is sent into a Switch node that routes execution into one of four operational paths:
- monitor
- optimize
- enforce_policy
- generate_report

### 1.5 Approval and Governance
Optimization actions are normalized with a Set node, then passed through an approval gate. If approval is required, a Slack approval request is sent and the approved strategy is logged. If no approval is required, the strategy is auto-executed by writing directly to the database.

### 1.6 Reporting, Notification, and KPI Closure
Branch outputs are merged and finalized with:
- Slack summary notification
- PostgreSQL KPI dashboard update
- Slack policy alerts
- Google Sheets ESG report append
- Email delivery of ESG summary

This closes the operational loop from AI evaluation to stakeholder communication.

---

## 2. Block-by-Block Analysis

## 2.1 Block: Trigger and Input Reception

**Overview:**  
This block launches the workflow at a fixed interval and passes the collected item directly into the AI supervisor. It is the sole actual entry point defined in the workflow JSON.

**Nodes Involved:**  
- Scheduled Carbon Data Collection

### Node Details

#### Scheduled Carbon Data Collection
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger node that starts the workflow automatically.
- **Configuration choices:**  
  Configured to run every 6 hours.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Carbon Supervisor Agent`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - If n8n scheduling is disabled or the instance is paused, the workflow will not run.
  - No built-in input acquisition exists after the trigger, so unless upstream execution data is injected externally or this trigger is later paired with collection nodes, the supervisor may receive minimal or empty JSON.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block: Supervisor Orchestration

**Overview:**  
This block is the control center of the workflow. It takes incoming data, analyzes it using GPT-4o, optionally delegates work to specialized agents, and returns a structured JSON payload for deterministic routing.

**Nodes Involved:**  
- Carbon Supervisor Agent
- Supervisor Model
- Supervisor Output Parser

### Node Details

#### Carbon Supervisor Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI orchestration agent that decides which specialized tools to invoke and what action should be taken.
- **Configuration choices:**  
  - Input text: `{{ $json.emissions_data || $json.body || $json }}`
  - System message defines the role as a carbon sustainability supervisor coordinating monitoring, optimization, policy enforcement, and ESG reporting.
  - Structured output parser enabled.
- **Key expressions or variables used:**  
  - `{{$json.emissions_data || $json.body || $json}}`
- **Input and output connections:**  
  - Main input: `Scheduled Carbon Data Collection`
  - AI language model input: `Supervisor Model`
  - AI tool inputs:
    - `Carbon Monitoring Agent`
    - `Carbon Optimization Agent`
    - `Policy Enforcement Agent`
    - `ESG Reporting Agent`
  - AI output parser input: `Supervisor Output Parser`
  - Main output: `Route by Action Type`
- **Version-specific requirements:**  
  Type version `3.1`; requires compatible LangChain agent support in the installed n8n version.
- **Edge cases or potential failure types:**  
  - If the inbound payload is missing expected fields, the agent may still respond but with low-quality decisions.
  - LLM timeout or API quota failure can halt the entire workflow.
  - If the model outputs content that cannot be parsed into the required schema, downstream routing fails.
- **Sub-workflow reference:**  
  None.

#### Supervisor Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model supplying the reasoning engine for the supervisor agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Carbon Supervisor Agent` as `ai_languageModel`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - Unsupported model name in the connected account
  - Rate limiting or token-limit failures
- **Sub-workflow reference:**  
  None.

#### Supervisor Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the supervisor to emit a predictable object for routing and persistence.
- **Configuration choices:**  
  Manual JSON schema with:
  - `action`: one of `monitor`, `optimize`, `enforce_policy`, `generate_report`
  - `priority`: one of `critical`, `high`, `medium`, `low`
  - `carbon_metrics`: object
  - `recommendations`: array
  - `policy_violations`: array
  - `requires_approval`: boolean
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Carbon Supervisor Agent` as `ai_outputParser`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Schema mismatch if the model returns malformed JSON
  - Missing required properties can break switch-based routing and later expressions
- **Sub-workflow reference:**  
  None.

---

## 2.3 Block: Specialized Monitoring and Optimization Intelligence

**Overview:**  
This block contains the specialist agents and helper tools used by the supervisor. Each agent has a dedicated role and supporting integrations to calculate metrics, inspect historical data, check policies, or request approval.

**Nodes Involved:**  
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

#### Carbon Monitoring Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized AI tool callable by the supervisor for carbon monitoring and KPI analysis.
- **Configuration choices:**  
  - Input text supplied dynamically with `$fromAI('monitoring_task', ...)`
  - System message instructs the agent to collect emissions data, validate quality, identify anomalies, and calculate KPIs.
  - Tool description explains its monitoring function to the supervisor.
- **Key expressions or variables used:**  
  - `{{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
- **Input and output connections:**  
  - AI language model: `Monitoring Model`
  - AI tools:
    - `Carbon Calculations Tool`
    - `Carbon Database Tool`
    - `KPI Calculator`
  - Output back to `Carbon Supervisor Agent`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - The supervisor may fail to provide a meaningful `monitoring_task`
  - Tool invocation errors from DB or code tool can degrade the answer
- **Sub-workflow reference:**  
  None.

#### Monitoring Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Carbon Monitoring Agent`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  Same class of OpenAI failures as the supervisor model.
- **Sub-workflow reference:**  
  None.

#### Carbon Calculations Tool
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  JavaScript-based AI tool that performs carbon-related calculations.
- **Configuration choices:**  
  Reads AI-provided arguments:
  - `emissions_value`
  - `energy_kwh`
  - `operation`
  
  Supported operations:
  - `footprint`
  - `intensity`
  - `efficiency`
  - `savings`
  
  Uses constants:
  - Grid carbon intensity = `0.475`
  - Renewable intensity = `0.05`
  
  Returns:
  - `result`
  - `unit`
  - `operation`
- **Key expressions or variables used:**  
  `$fromAI(...)` to obtain numeric/string tool inputs from the calling agent.
- **Input and output connections:**  
  - Output to `Carbon Monitoring Agent`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - If the AI supplies non-numeric values where numbers are expected
  - Efficiency calculation can become misleading when energy is zero or near zero
  - `toFixed(2)` returns a string, not a numeric type
- **Sub-workflow reference:**  
  None.

#### Carbon Database Tool
- **Type and technical role:** `n8n-nodes-base.postgresTool`  
  Database tool exposed to AI agents for querying and updating historical carbon data.
- **Configuration choices:**  
  - Schema: `public`
  - Tool description says it should query/update emissions data, KPI metrics, historical trends, and policy configurations.
  - Table is not selected in the exported JSON.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Carbon Monitoring Agent`
  - Output to `Carbon Optimization Agent`
- **Version-specific requirements:**  
  Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Database credentials are not included in the JSON and must be configured
  - Empty table selection means the tool is incomplete until manually configured
  - AI-generated SQL or tool requests may fail on schema mismatch or permissions
- **Sub-workflow reference:**  
  None.

#### Carbon Optimization Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  AI tool for emissions reduction strategy generation.
- **Configuration choices:**  
  - Input task from `$fromAI('optimization_task', ...)`
  - System message asks it to identify high-carbon workloads, suggest region migration, rightsizing, renewable sourcing, and estimate carbon savings.
- **Key expressions or variables used:**  
  - `{{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
- **Input and output connections:**  
  - AI language model: `Optimization Model`
  - AI tools:
    - `Carbon Database Tool`
    - `KPI Calculator`
    - `Approval Workflow Tool`
  - Output back to `Carbon Supervisor Agent`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If recommendations are returned as a string instead of array, downstream formatting may fail
  - Potential overuse of human approval tooling if the agent invokes it too freely
- **Sub-workflow reference:**  
  None.

#### Optimization Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Carbon Optimization Agent`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  Standard OpenAI connectivity, quota, and latency issues.
- **Sub-workflow reference:**  
  None.

#### KPI Calculator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  Generic calculator tool usable by AI agents.
- **Configuration choices:**  
  No custom parameters.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Carbon Monitoring Agent`
  - Output to `Approval Workflow Tool`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Limited usefulness unless the agent frames calculations clearly
- **Sub-workflow reference:**  
  None.

#### Approval Workflow Tool
- **Type and technical role:** `n8n-nodes-base.slackHitlTool`  
  Slack human-in-the-loop tool for approval interactions from within the AI tooling ecosystem.
- **Configuration choices:**  
  - Message body uses `$fromAI('approval_message', ...)`
  - Authentication: OAuth2
  - Slack user field is unselected in the JSON
- **Key expressions or variables used:**  
  - `{{ $fromAI('approval_message', 'The approval request message') }}`
- **Input and output connections:**  
  - Output to `Carbon Optimization Agent`
  - Receives `KPI Calculator` as an AI tool
- **Version-specific requirements:**  
  Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Slack OAuth misconfiguration
  - Missing target user
  - HITL callback/webhook issues
  - This tool exists in addition to the later standard Slack approval node, which can create overlapping approval logic
- **Sub-workflow reference:**  
  None.

#### Policy Enforcement Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  AI policy compliance specialist.
- **Configuration choices:**  
  - Input task from `$fromAI('policy_task', ...)`
  - System message directs it to check emissions against policies, identify violations, severity, and escalation needs.
- **Key expressions or variables used:**  
  - `{{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
- **Input and output connections:**  
  - AI language model: `Policy Model`
  - AI tool: `Policy Sheets Tool`
  - Output back to `Carbon Supervisor Agent`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If the sheet mapping is incomplete, policy checks may return low-confidence answers
  - Can produce violation objects in formats incompatible with the Slack alert template
- **Sub-workflow reference:**  
  None.

#### Policy Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Policy Enforcement Agent`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  Standard LLM API failures.
- **Sub-workflow reference:**  
  None.

#### Policy Sheets Tool
- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`  
  AI-callable tool for reading policy and target data from Google Sheets.
- **Configuration choices:**  
  - Document ID not selected
  - Sheet name not selected
  - Manual description indicates it stores carbon reduction policies, approved strategies, and sustainability targets
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `Policy Enforcement Agent`
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Incomplete configuration prevents useful sheet access
  - OAuth credential or permission errors
- **Sub-workflow reference:**  
  None.

#### ESG Reporting Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized AI report generator for ESG outputs.
- **Configuration choices:**  
  - Input task from `$fromAI('reporting_task', ...)`
  - System message instructs report generation for carbon metrics, KPIs, reduction achievements, compliance status, and stakeholder-ready formatting.
- **Key expressions or variables used:**  
  - `{{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
- **Input and output connections:**  
  - AI language model: `ESG Model`
  - Output back to `Carbon Supervisor Agent`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If required carbon metrics are absent, generated reports can be incomplete
- **Sub-workflow reference:**  
  None.

#### ESG Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to `ESG Reporting Agent`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  Standard OpenAI failures.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block: Action Routing

**Overview:**  
This block dispatches the supervisor’s structured output into one of four downstream operational paths. It is the key deterministic branching point of the workflow.

**Nodes Involved:**  
- Route by Action Type

### Node Details

#### Route by Action Type
- **Type and technical role:** `n8n-nodes-base.switch`  
  Conditional branch router based on the `action` field emitted by the supervisor.
- **Configuration choices:**  
  Four rules:
  - `monitor`
  - `optimize`
  - `enforce_policy`
  - `generate_report`
- **Key expressions or variables used:**  
  - `{{ $json.action }}`
- **Input and output connections:**  
  - Input: `Carbon Supervisor Agent`
  - Outputs:
    - `Store Carbon Metrics`
    - `Prepare Optimization Data`
    - `Send Policy Alert`
    - `Update ESG Report`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If `action` is missing or outside the enum, no route will match
  - Since there is no default branch, unmatched outputs disappear from workflow processing
- **Sub-workflow reference:**  
  None.

---

## 2.5 Block: Monitoring Persistence

**Overview:**  
This path persists structured monitoring output into PostgreSQL. It is used when the supervisor classifies the result as a monitoring action.

**Nodes Involved:**  
- Store Carbon Metrics

### Node Details

#### Store Carbon Metrics
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts monitoring result data into the `carbon_metrics` table.
- **Configuration choices:**  
  Table: `carbon_metrics` in schema `public`  
  Mapped columns:
  - `action`
  - `priority`
  - `timestamp`
  - `carbon_metrics` as JSON string
  - `recommendations` as JSON string
  - `policy_violations` as JSON string
  - `requires_approval`
- **Key expressions or variables used:**  
  - `{{ $json.action }}`
  - `{{ $json.priority }}`
  - `{{ $now }}`
  - `{{ JSON.stringify($json.carbon_metrics) }}`
  - `{{ JSON.stringify($json.recommendations) }}`
  - `{{ JSON.stringify($json.policy_violations) }}`
  - `{{ $json.requires_approval }}`
- **Input and output connections:**  
  - Input: `Route by Action Type`
  - Output: `Consolidate All Actions`
- **Version-specific requirements:**  
  Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Database schema must support these columns and data types
  - If arrays/objects are unexpectedly null and DB columns are constrained, inserts may fail
- **Sub-workflow reference:**  
  None.

---

## 2.6 Block: Optimization Approval and Execution

**Overview:**  
This block reformats optimization output, determines whether human sign-off is required, and either sends a Slack approval request or auto-executes the strategy by logging it to PostgreSQL.

**Nodes Involved:**  
- Prepare Optimization Data
- Check Approval Required
- Request Strategy Approval
- Log Approved Strategy
- Auto-Execute Strategy

### Node Details

#### Prepare Optimization Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes optimization branch output into fields expected by the approval logic.
- **Configuration choices:**  
  Sets:
  - `optimization_summary` = `{{$json.output.recommendations}}`
  - `carbon_savings` = `{{$json.output.carbon_metrics.potential_savings}}`
  - `priority` = `{{$json.output.priority}}`
  - `requires_approval` = `{{$json.output.requires_approval}}`
- **Key expressions or variables used:**  
  Assumes data lives under `$json.output.*`
- **Input and output connections:**  
  - Input: `Route by Action Type`
  - Output: `Check Approval Required`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - This is a major structural risk: the supervisor output elsewhere appears to expose fields directly as `$json.action`, `$json.priority`, etc., not under `$json.output`.
  - If no `output` wrapper exists, all mapped values will be undefined and the approval branch will malfunction.
- **Sub-workflow reference:**  
  None.

#### Check Approval Required
- **Type and technical role:** `n8n-nodes-base.if`  
  Splits optimization flow based on whether human approval is needed.
- **Configuration choices:**  
  Condition checks whether `{{$json.requires_approval}}` is true.
- **Key expressions or variables used:**  
  - `{{ $json.requires_approval }}`
- **Input and output connections:**  
  - Input: `Prepare Optimization Data`
  - True output: `Request Strategy Approval`
  - False output: `Auto-Execute Strategy`
- **Version-specific requirements:**  
  Type version `2.3`.
- **Edge cases or potential failure types:**  
  - If `requires_approval` is undefined because of upstream mapping issues, the false branch will likely be taken
- **Sub-workflow reference:**  
  None.

#### Request Strategy Approval
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack approval request and waits for a human response.
- **Configuration choices:**  
  - Operation: `sendAndWait`
  - Approval type: double
  - Reject label: `Reject`
  - Message includes:
    - priority
    - estimated carbon savings
    - recommendation list
- **Key expressions or variables used:**  
  - `{{ $json.priority }}`
  - `{{ $json.carbon_savings }}`
  - `{{ $json.optimization_summary.map(r => `• ${r}`).join('\n') }}`
- **Input and output connections:**  
  - Input: `Check Approval Required` true branch
  - Output: `Log Approved Strategy`
- **Version-specific requirements:**  
  Type version `2.4`.
- **Edge cases or potential failure types:**  
  - If `optimization_summary` is not an array, `.map()` will throw
  - Channel ID is still a placeholder and must be replaced
  - This node logs only approved interactions downstream; reject handling is not explicitly modeled
- **Sub-workflow reference:**  
  None.

#### Log Approved Strategy
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Persists an approved strategy to the database.
- **Configuration choices:**  
  Table: `approved_strategies`  
  Fields:
  - `status` = `approved`
  - `priority`
  - `strategy` as JSON string
  - `timestamp`
  - `approved_by` = `{{$json.user}}`
  - `carbon_savings`
- **Key expressions or variables used:**  
  - `{{ $json.priority }}`
  - `{{ JSON.stringify($json.optimization_summary) }}`
  - `{{ $now }}`
  - `{{ $json.user }}`
  - `{{ $json.carbon_savings }}`
- **Input and output connections:**  
  - Input: `Request Strategy Approval`
  - Output: `Consolidate All Actions`
- **Version-specific requirements:**  
  Type version `2.6`.
- **Edge cases or potential failure types:**  
  - The exact Slack approval response field containing approver identity may differ from `$json.user`
  - Rejected approvals are not stored separately
- **Sub-workflow reference:**  
  None.

#### Auto-Execute Strategy
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Logs low-risk strategies as auto-executed.
- **Configuration choices:**  
  Table: `approved_strategies`  
  Fields:
  - `status` = `auto_executed`
  - `priority`
  - `strategy`
  - `timestamp`
  - `carbon_savings`
- **Key expressions or variables used:**  
  - `{{ $json.priority }}`
  - `{{ JSON.stringify($json.optimization_summary) }}`
  - `{{ $now }}`
  - `{{ $json.carbon_savings }}`
- **Input and output connections:**  
  - Input: `Check Approval Required` false branch
  - Output: `Consolidate All Actions`
- **Version-specific requirements:**  
  Type version `2.6`.
- **Edge cases or potential failure types:**  
  - Same mapping risks as above if optimization data is undefined
- **Sub-workflow reference:**  
  None.

---

## 2.7 Block: Policy Enforcement Notification

**Overview:**  
This branch sends a Slack alert when the supervisor decides a policy enforcement action is required.

**Nodes Involved:**  
- Send Policy Alert

### Node Details

#### Send Policy Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted policy violation message to a Slack channel.
- **Configuration choices:**  
  Message includes:
  - priority
  - violation count
  - per-violation bullet list
  - recommended actions
- **Key expressions or variables used:**  
  - `{{ $json.output.priority }}`
  - `{{ $json.output.policy_violations.length }}`
  - `{{ $json.output.policy_violations.map(v => `• ${v.policy}: ${v.description}`).join('\n') }}`
  - `{{ $json.output.recommendations.map(r => `• ${r}`).join('\n') }}`
- **Input and output connections:**  
  - Input: `Route by Action Type`
  - Output: `Consolidate All Actions`
- **Version-specific requirements:**  
  Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Same likely schema mismatch as optimization branch: this node expects `$json.output.*`, while the switch routes on top-level `$json.action`
  - If arrays are missing, `.length` or `.map()` will fail
  - Channel ID is still placeholder text
- **Sub-workflow reference:**  
  None.

---

## 2.8 Block: ESG Report Delivery

**Overview:**  
This branch appends report data to Google Sheets and then emails a formatted summary to stakeholders.

**Nodes Involved:**  
- Update ESG Report
- Send ESG Report Email

### Node Details

#### Update ESG Report
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new row to an ESG report sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Document ID: not selected
  - Sheet name: not selected
- **Key expressions or variables used:**  
  None visible in the node parameters.
- **Input and output connections:**  
  - Input: `Route by Action Type`
  - Output: `Send ESG Report Email`
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Incomplete configuration: sheet name and document ID are blank
  - Without explicit column mapping, appended row behavior depends on incoming data structure and node defaults
- **Sub-workflow reference:**  
  None.

#### Send ESG Report Email
- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends an HTML email containing key ESG metrics and recommendations.
- **Configuration choices:**  
  - Subject includes current date
  - HTML body includes:
    - total emissions
    - carbon intensity
    - policy compliance rate
    - recommendations list
  - `toEmail` and `fromEmail` are placeholders
- **Key expressions or variables used:**  
  - `{{ $json.carbon_metrics.total_emissions }}`
  - `{{ $json.carbon_metrics.carbon_intensity }}`
  - `{{ $json.carbon_metrics.compliance_rate }}`
  - `{{ $json.recommendations.map(r => `• ${r}`).join('<br>') }}`
  - `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Input and output connections:**  
  - Input: `Update ESG Report`
  - Output: `Consolidate All Actions`
- **Version-specific requirements:**  
  Type version `2.1`.
- **Edge cases or potential failure types:**  
  - The workflow description mentions Gmail OAuth2, but this node is a generic email send node; SMTP or another configured mail transport may be needed depending on n8n setup
  - Placeholder sender/recipient addresses must be replaced
  - If recommendations is not an array, `.map()` will fail
- **Sub-workflow reference:**  
  None.

---

## 2.9 Block: Consolidation and Final KPI Update

**Overview:**  
This block gathers outputs from the various action branches, sends a completion summary, and records workflow-level KPI execution data.

**Nodes Involved:**  
- Consolidate All Actions
- Send Summary Notification
- Update KPI Dashboard

### Node Details

#### Consolidate All Actions
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines results from up to five branch endpoints before final notification.
- **Configuration choices:**  
  Configured for `5` inputs.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Inputs from:
  - `Store Carbon Metrics`
  - `Log Approved Strategy`
  - `Auto-Execute Strategy`
  - `Send Policy Alert`
  - `Send ESG Report Email`
  
  Output to:
  - `Send Summary Notification`
- **Version-specific requirements:**  
  Type version `3.2`.
- **Edge cases or potential failure types:**  
  - Merge behavior depends on execution semantics; if only one branch executes per run, ensure the selected merge mode is compatible with sparse inputs
  - A five-input merge after mutually exclusive routing can behave unexpectedly if n8n expects all branches
- **Sub-workflow reference:**  
  None.

#### Send Summary Notification
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a final success message to a summary Slack channel.
- **Configuration choices:**  
  Message includes:
  - action or fallback to “Multiple actions”
  - priority or fallback to `N/A`
  - generic success text
- **Key expressions or variables used:**  
  - `{{ $json.action || 'Multiple actions' }}`
  - `{{ $json.priority || 'N/A' }}`
- **Input and output connections:**  
  - Input: `Consolidate All Actions`
  - Output: `Update KPI Dashboard`
- **Version-specific requirements:**  
  Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Channel ID is still placeholder text
  - Merged payload may not consistently contain `action` and `priority`
- **Sub-workflow reference:**  
  None.

#### Update KPI Dashboard
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Logs workflow-run completion stats to a KPI dashboard table.
- **Configuration choices:**  
  Table: `kpi_dashboard`  
  Fields:
  - `status` = `completed`
  - `timestamp`
  - `total_actions` = `{{$json.action ? 1 : 0}}`
  - `workflow_run_id` = `{{$execution.id}}`
- **Key expressions or variables used:**  
  - `{{ $now }}`
  - `{{ $json.action ? 1 : 0 }}`
  - `{{ $execution.id }}`
- **Input and output connections:**  
  - Input: `Send Summary Notification`
  - Output: none
- **Version-specific requirements:**  
  Type version `2.6`.
- **Edge cases or potential failure types:**  
  - If merged payload does not retain `action`, total_actions may always become 0
  - Requires DB schema support for workflow run tracking
- **Sub-workflow reference:**  
  None.

---

## 2.10 Block: Informational Sticky Notes

**Overview:**  
These nodes are documentation aids inside the canvas. They do not affect execution but contain useful context about workflow purpose, setup, prerequisites, governance, and reporting.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Contains the high-level “How It Works” description.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Contains setup steps.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Contains prerequisites, use cases, customization notes, and benefits.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes strategy and policy evaluation.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes supervisor orchestration.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes approval routing.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes reporting and notification.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Carbon Data Collection | Schedule Trigger | Starts workflow every 6 hours |  | Carbon Supervisor Agent | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Carbon Supervisor Agent | LangChain Agent | Main AI orchestrator and action selector | Scheduled Carbon Data Collection | Route by Action Type | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Supervisor Model | OpenAI Chat Model | LLM for supervisor agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Supervisor Output Parser | Structured Output Parser | Enforces structured JSON output from supervisor |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Monitoring Agent | Agent Tool | Monitoring specialist for emissions validation and KPI analysis |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Monitoring Model | OpenAI Chat Model | LLM for monitoring specialist |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Optimization Agent | Agent Tool | Optimization specialist for reduction strategies |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Optimization Model | OpenAI Chat Model | LLM for optimization specialist |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Enforcement Agent | Agent Tool | Policy compliance specialist |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Model | OpenAI Chat Model | LLM for policy specialist |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Reporting Agent | Agent Tool | ESG report generation specialist |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Model | OpenAI Chat Model | LLM for ESG reporting specialist |  | ESG Reporting Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Calculations Tool | Code Tool | Carbon formula and intensity calculations |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| KPI Calculator | Calculator Tool | General-purpose numeric calculator for AI tools |  | Carbon Monitoring Agent; Approval Workflow Tool | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Database Tool | Postgres Tool | AI-accessible emissions/history database operations |  | Carbon Monitoring Agent; Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Sheets Tool | Google Sheets Tool | AI-accessible policy/target lookup |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Approval Workflow Tool | Slack HITL Tool | Human-in-the-loop approval support for AI tools | KPI Calculator | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Route by Action Type | Switch | Routes supervisor result by action enum | Carbon Supervisor Agent | Store Carbon Metrics; Prepare Optimization Data; Send Policy Alert; Update ESG Report | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Store Carbon Metrics | Postgres | Persists monitoring output | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Prepare Optimization Data | Set | Normalizes optimization output for approval logic | Route by Action Type | Check Approval Required | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Check Approval Required | If | Splits approved vs auto-executed strategy path | Prepare Optimization Data | Request Strategy Approval; Auto-Execute Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Request Strategy Approval | Slack | Sends approval request and waits for response | Check Approval Required | Log Approved Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Log Approved Strategy | Postgres | Stores approved strategy execution record | Request Strategy Approval | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Auto-Execute Strategy | Postgres | Stores auto-executed strategy record | Check Approval Required | Consolidate All Actions | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Send Policy Alert | Slack | Sends compliance violation alerts | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update ESG Report | Google Sheets | Appends ESG reporting data to sheet | Route by Action Type | Send ESG Report Email | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send ESG Report Email | Email Send | Emails ESG summary to stakeholders | Update ESG Report | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Consolidate All Actions | Merge | Merges branch outputs before final notification | Store Carbon Metrics; Log Approved Strategy; Auto-Execute Strategy; Send Policy Alert; Send ESG Report Email | Send Summary Notification | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send Summary Notification | Slack | Sends final workflow completion message | Consolidate All Actions | Update KPI Dashboard | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update KPI Dashboard | Postgres | Writes workflow completion stats to KPI table | Send Summary Notification |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Sticky Note | Sticky Note | Canvas documentation |  |  | ## How It Works<br>This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. |
| Sticky Note1 | Sticky Note | Canvas setup instructions |  |  | ## Setup Steps<br>1. Connect scheduled trigger and webhook nodes to your emissions data sources.<br>2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account).<br>3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible).<br>4. Set approval thresholds in the Check Approval Required node.<br>5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. |
| Sticky Note2 | Sticky Note | Canvas prerequisites and context |  |  | ## Prerequisites<br>- OpenAI or compatible LLM API key<br>- Slack bot token<br>- Gmail OAuth2 credentials<br>- Google Sheets service account<br>## Use Cases<br>- Corporate sustainability teams automating monthly ESG reporting<br>## Customisation<br>- Swap LLM models per agent for cost or accuracy trade-offs<br>## Benefits<br>- Eliminates manual emissions data aggregation and report generation |
| Sticky Note3 | Sticky Note | Canvas explanation of strategy/policy block |  |  | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Sticky Note4 | Sticky Note | Canvas explanation of supervisor block |  |  | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Sticky Note5 | Sticky Note | Canvas explanation of approval block |  |  | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Sticky Note6 | Sticky Note | Canvas explanation of reporting block |  |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `AI carbon supervisor agent for ESG monitoring and strategy execution`.

2. **Add a Schedule Trigger node** named `Scheduled Carbon Data Collection`.
   - Set interval mode to run every `6 hours`.

3. **Add the main LangChain Agent node** named `Carbon Supervisor Agent`.
   - Set its input text to:  
     `{{ $json.emissions_data || $json.body || $json }}`
   - Enable structured output parsing.
   - Add this system message:  
     “You are the Carbon Sustainability Supervisor Agent. Your role is to orchestrate carbon monitoring, optimization, policy enforcement, and ESG reporting across multi-cloud and enterprise environments. Analyze incoming emissions data, delegate tasks to specialized sub-agents (monitoring, optimization, policy enforcement, ESG reporting), and coordinate carbon reduction strategies. Determine which sub-agents to invoke based on the data type and required actions.”

4. **Connect** `Scheduled Carbon Data Collection` → `Carbon Supervisor Agent`.

5. **Add an OpenAI Chat Model node** named `Supervisor Model`.
   - Choose model `gpt-4o`
   - Set temperature to `0.2`
   - Attach OpenAI credentials
   - Connect it to `Carbon Supervisor Agent` as the language model.

6. **Add a Structured Output Parser node** named `Supervisor Output Parser`.
   - Use manual schema mode.
   - Define a JSON schema with:
     - `action`: enum of `monitor`, `optimize`, `enforce_policy`, `generate_report`
     - `priority`: enum of `critical`, `high`, `medium`, `low`
     - `carbon_metrics`: object
     - `recommendations`: array
     - `policy_violations`: array
     - `requires_approval`: boolean
   - Connect it to `Carbon Supervisor Agent` as output parser.

7. **Add an Agent Tool node** named `Carbon Monitoring Agent`.
   - Set text to:  
     `{{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
   - System message should instruct monitoring, KPI calculation, anomaly detection, and data quality validation.
   - Add a tool description explaining it collects and validates emissions data.
   - Connect this node to `Carbon Supervisor Agent` as an AI tool.

8. **Add an OpenAI Chat Model node** named `Monitoring Model`.
   - Model `gpt-4o`
   - Temperature `0.1`
   - Connect to `Carbon Monitoring Agent`.

9. **Add a Code Tool node** named `Carbon Calculations Tool`.
   - Paste the carbon calculation JavaScript logic from the workflow:
     - read `emissions_value`, `energy_kwh`, and `operation` using `$fromAI`
     - support `footprint`, `intensity`, `efficiency`, `savings`
     - return `{ result, unit, operation }`
   - Connect it to `Carbon Monitoring Agent` as an AI tool.

10. **Add a Postgres Tool node** named `Carbon Database Tool`.
    - Configure PostgreSQL credentials.
    - Set schema to `public`.
    - Select the relevant table or tables for emissions/history access.
    - Add tool description: querying and updating carbon emissions data, KPI metrics, historical trends, and policy configurations.
    - Connect it to `Carbon Monitoring Agent` and also later to `Carbon Optimization Agent`.

11. **Add a Calculator Tool node** named `KPI Calculator`.
    - Keep default settings.
    - Connect it to `Carbon Monitoring Agent`.

12. **Add an Agent Tool node** named `Carbon Optimization Agent`.
    - Text:  
      `{{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
    - Use a system message focused on energy optimization, low-carbon alternatives, region migration, rightsizing, renewable energy sourcing, and carbon savings estimation.
    - Connect it to `Carbon Supervisor Agent` as an AI tool.

13. **Add an OpenAI Chat Model node** named `Optimization Model`.
    - Model `gpt-4o`
    - Temperature `0.3`
    - Connect it to `Carbon Optimization Agent`.

14. **Connect** `Carbon Database Tool` to `Carbon Optimization Agent` as an AI tool.

15. **Add a Slack HITL Tool node** named `Approval Workflow Tool`.
    - Configure Slack OAuth2 credentials.
    - Set message to:  
      `{{ $fromAI('approval_message', 'The approval request message') }}`
    - Pick a Slack user or leave it dynamic if your setup supports it.
    - Connect `KPI Calculator` to it as an AI tool.
    - Connect `Approval Workflow Tool` to `Carbon Optimization Agent` as an AI tool.

16. **Add an Agent Tool node** named `Policy Enforcement Agent`.
    - Text:  
      `{{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
    - Use a system message for compliance checks, policy violation identification, severity assessment, corrective actions, and escalation.
    - Connect to `Carbon Supervisor Agent` as an AI tool.

17. **Add an OpenAI Chat Model node** named `Policy Model`.
    - Model `gpt-4o`
    - Temperature `0.1`
    - Connect to `Policy Enforcement Agent`.

18. **Add a Google Sheets Tool node** named `Policy Sheets Tool`.
    - Configure Google Sheets OAuth2 credentials.
    - Select the policy spreadsheet document ID.
    - Select the sheet name containing policies and sustainability targets.
    - Add tool description about reading carbon reduction policies, approved strategies, and targets.
    - Connect to `Policy Enforcement Agent`.

19. **Add an Agent Tool node** named `ESG Reporting Agent`.
    - Text:  
      `{{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
    - Use a system message for generating ESG-ready reports, KPIs, compliance summaries, and stakeholder formatting.
    - Connect to `Carbon Supervisor Agent` as an AI tool.

20. **Add an OpenAI Chat Model node** named `ESG Model`.
    - Model `gpt-4o`
    - Temperature `0.2`
    - Connect to `ESG Reporting Agent`.

21. **Add a Switch node** named `Route by Action Type`.
    - Create four conditions based on `{{ $json.action }}`:
      - equals `monitor`
      - equals `optimize`
      - equals `enforce_policy`
      - equals `generate_report`
    - Connect `Carbon Supervisor Agent` → `Route by Action Type`.

22. **Add a Postgres node** named `Store Carbon Metrics`.
    - Configure PostgreSQL credentials.
    - Table: `public.carbon_metrics`
    - Map fields:
      - action = `{{ $json.action }}`
      - priority = `{{ $json.priority }}`
      - timestamp = `{{ $now }}`
      - carbon_metrics = `{{ JSON.stringify($json.carbon_metrics) }}`
      - recommendations = `{{ JSON.stringify($json.recommendations) }}`
      - policy_violations = `{{ JSON.stringify($json.policy_violations) }}`
      - requires_approval = `{{ $json.requires_approval }}`
    - Connect monitor output of `Route by Action Type` → `Store Carbon Metrics`.

23. **Add a Set node** named `Prepare Optimization Data`.
    - Create fields:
      - `optimization_summary`
      - `carbon_savings`
      - `priority`
      - `requires_approval`
    - Recommended safer mapping if rebuilding from scratch:
      - `optimization_summary = {{ $json.recommendations }}`
      - `carbon_savings = {{ $json.carbon_metrics.potential_savings }}`
      - `priority = {{ $json.priority }}`
      - `requires_approval = {{ $json.requires_approval }}`
    - Note: the exported workflow uses `$json.output.*`; that may need correction.
    - Connect optimize output of `Route by Action Type` → `Prepare Optimization Data`.

24. **Add an IF node** named `Check Approval Required`.
    - Condition: `{{ $json.requires_approval }}` is true.
    - Connect `Prepare Optimization Data` → `Check Approval Required`.

25. **Add a Slack node** named `Request Strategy Approval`.
    - Configure Slack OAuth2.
    - Operation: `sendAndWait`
    - Choose target channel for approvals.
    - Enable approval options:
      - approval type `double`
      - disapprove label `Reject`
    - Message should include priority, carbon savings, and recommendations.
    - Connect true branch of `Check Approval Required` → `Request Strategy Approval`.

26. **Add a Postgres node** named `Log Approved Strategy`.
    - Table: `public.approved_strategies`
    - Map:
      - status = `approved`
      - priority = `{{ $json.priority }}`
      - strategy = `{{ JSON.stringify($json.optimization_summary) }}`
      - timestamp = `{{ $now }}`
      - approved_by = `{{ $json.user }}`
      - carbon_savings = `{{ $json.carbon_savings }}`
    - Connect `Request Strategy Approval` → `Log Approved Strategy`.

27. **Add a Postgres node** named `Auto-Execute Strategy`.
    - Table: `public.approved_strategies`
    - Map:
      - status = `auto_executed`
      - priority = `{{ $json.priority }}`
      - strategy = `{{ JSON.stringify($json.optimization_summary) }}`
      - timestamp = `{{ $now }}`
      - carbon_savings = `{{ $json.carbon_savings }}`
    - Connect false branch of `Check Approval Required` → `Auto-Execute Strategy`.

28. **Add a Slack node** named `Send Policy Alert`.
    - Configure Slack OAuth2.
    - Choose the policy alerts channel.
    - Build a message showing:
      - priority
      - violation count
      - violation details
      - recommendations
    - Recommended safer mapping if rebuilding:
      - use top-level fields like `{{ $json.priority }}`, `{{ $json.policy_violations }}`, `{{ $json.recommendations }}`
    - Connect enforce_policy output of `Route by Action Type` → `Send Policy Alert`.

29. **Add a Google Sheets node** named `Update ESG Report`.
    - Configure Google Sheets credentials.
    - Operation: `append`
    - Choose the ESG reporting spreadsheet document ID.
    - Choose the target worksheet.
    - Map report fields explicitly if possible.
    - Connect generate_report output of `Route by Action Type` → `Update ESG Report`.

30. **Add an Email Send node** named `Send ESG Report Email`.
    - Configure your mail transport or SMTP credentials.
    - Set recipient and sender addresses.
    - Subject: include the current date.
    - HTML body: include total emissions, carbon intensity, compliance rate, and recommendation bullets.
    - Connect `Update ESG Report` → `Send ESG Report Email`.

31. **Add a Merge node** named `Consolidate All Actions`.
    - Set number of inputs to `5`.
    - Connect these nodes into it:
      - `Store Carbon Metrics`
      - `Log Approved Strategy`
      - `Auto-Execute Strategy`
      - `Send Policy Alert`
      - `Send ESG Report Email`

32. **Validate merge behavior carefully.**
    - Because the switch routes to only one branch per execution, you may need to choose a merge mode that does not wait for all 5 branches.
    - If necessary, replace this with a simpler pattern such as separate notifications per branch or a pass-through merge design.

33. **Add a Slack node** named `Send Summary Notification`.
    - Configure Slack OAuth2.
    - Choose the summary channel.
    - Message can include:
      - action
      - priority
      - success state
    - Connect `Consolidate All Actions` → `Send Summary Notification`.

34. **Add a Postgres node** named `Update KPI Dashboard`.
    - Table: `public.kpi_dashboard`
    - Map:
      - status = `completed`
      - timestamp = `{{ $now }}`
      - total_actions = `{{ $json.action ? 1 : 0 }}`
      - workflow_run_id = `{{ $execution.id }}`
    - Connect `Send Summary Notification` → `Update KPI Dashboard`.

35. **Configure credentials** for all external services:
    - **OpenAI** for all GPT-4o model nodes
    - **Slack OAuth2** for `Approval Workflow Tool`, `Request Strategy Approval`, `Send Policy Alert`, and `Send Summary Notification`
    - **Google Sheets OAuth2** for `Policy Sheets Tool` and `Update ESG Report`
    - **PostgreSQL credentials** for all Postgres and Postgres Tool nodes
    - **Email/SMTP credentials** for `Send ESG Report Email`

36. **Replace all placeholder values**:
    - Slack channel IDs
    - email addresses
    - Google Sheet document IDs
    - Google Sheet names
    - database table structures where needed

37. **Optionally add a webhook trigger** if you want real-time emissions ingestion.
    - The descriptive notes mention webhook-based ingestion, but the actual workflow JSON does not include one.
    - If you add it, merge or route it into the same `Carbon Supervisor Agent`.

38. **Test each branch separately** by mocking supervisor outputs:
    - `monitor`
    - `optimize`
    - `enforce_policy`
    - `generate_report`

39. **Correct data path inconsistencies before production use.**
    - The exported workflow mixes top-level fields like `$json.action` with nested fields like `$json.output.priority`.
    - Standardize on one structure.

40. **Activate the workflow** only after confirming:
    - all credentials work
    - all branch expressions resolve safely
    - merge behavior is correct for single-branch runs

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. | Workflow purpose note |
| 1. Connect scheduled trigger and webhook nodes to your emissions data sources. 2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account). 3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible). 4. Set approval thresholds in the Check Approval Required node. 5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. | Setup guidance |
| Prerequisites: OpenAI or compatible LLM API key; Slack bot token; Gmail OAuth2 credentials; Google Sheets service account | Prerequisites |
| Use case: Corporate sustainability teams automating monthly ESG reporting | Use case |
| Customisation: Swap LLM models per agent for cost or accuracy trade-offs | Customisation |
| Benefit: Eliminates manual emissions data aggregation and report generation | Benefit |

## Important implementation observations

- The workflow description mentions **real-time webhook ingestion**, but there is **no webhook trigger node** in the JSON.
- Several nodes still contain **blank configuration values**:
  - `Carbon Database Tool` table
  - `Policy Sheets Tool` document ID and sheet name
  - `Update ESG Report` document ID and sheet name
  - Slack target channels
  - Email sender/recipient placeholders
- There is a likely **schema inconsistency** between:
  - routing logic using `$json.action`
  - downstream nodes expecting `$json.output.*`
- The merge node may need redesign depending on how you want single-branch executions to complete.
- The approval pattern exists in **two forms**:
  - AI-level `Approval Workflow Tool`
  - explicit downstream Slack approval node `Request Strategy Approval`  
  This is not invalid, but it may be redundant unless intentionally layered.