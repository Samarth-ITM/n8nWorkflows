Monitor and optimize carbon emissions for ESG with GPT‑4o, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-and-optimize-carbon-emissions-for-esg-with-gpt-4o--slack-and-google-sheets-14462


# Monitor and optimize carbon emissions for ESG with GPT‑4o, Slack and Google Sheets

# 1. Workflow Overview

This workflow implements an AI-driven carbon sustainability operations pipeline for ESG monitoring, optimization, compliance, and reporting. It ingests emissions data from two entry points, delegates analysis to a supervisor agent powered by GPT‑4o, routes the resulting action to specialized downstream branches, then stores, notifies, and reports outcomes through PostgreSQL, Slack, Google Sheets, and email.

Typical use cases include:

- Continuous enterprise carbon monitoring
- Carbon reduction strategy evaluation and approval
- Policy compliance detection and escalation
- ESG report generation for stakeholders
- KPI logging for sustainability operations

## 1.1 Input Reception and Data Unification

The workflow starts from either a scheduled trigger every 6 hours or a real-time webhook receiving POSTed emissions payloads. Both feeds are merged into a unified stream for AI processing.

## 1.2 Supervisor AI Orchestration

A central Carbon Supervisor Agent receives the unified data, uses GPT‑4o as the orchestration model, and can call four specialized AI sub-agents:
- Carbon Monitoring Agent
- Carbon Optimization Agent
- Policy Enforcement Agent
- ESG Reporting Agent

The supervisor is constrained by a structured output parser that enforces a predictable JSON-like response schema.

## 1.3 Specialized Agent Tooling Layer

Each specialist agent has its own GPT‑4o model and, where relevant, connected tools:
- Monitoring can use code-based carbon calculations, a PostgreSQL tool, and a calculator
- Optimization can use PostgreSQL, a calculator, and a Slack human-in-the-loop approval tool
- Policy enforcement can use Google Sheets as a policy source
- ESG reporting uses its own language model only

## 1.4 Action Routing

After supervisor output is parsed, a Switch node routes execution according to `action`:
- `monitor`
- `optimize`
- `enforce_policy`
- `generate_report`

## 1.5 Persistence, Approval, and Execution

Depending on the selected action:
- Monitoring results are stored in PostgreSQL
- Optimization results are normalized, then either routed to Slack for approval or auto-executed and logged
- Policy violations are sent to Slack
- ESG report rows are appended to Google Sheets and summarized by email

## 1.6 Consolidation and Final Notifications

All terminal action branches converge into a merge node. A final Slack summary is sent, then a KPI dashboard row is inserted into PostgreSQL to record workflow completion.

---

# 2. Block-by-Block Analysis

## Block A — Input Reception and Data Unification

### Overview
This block collects emissions data from either a recurring schedule or a real-time webhook. Both are merged into a single stream so downstream AI processing does not need to distinguish source type.

### Nodes Involved
- Scheduled Carbon Data Collection
- Real-time Emissions Data Webhook
- Merge Data Sources

### Node Details

#### 1. Scheduled Carbon Data Collection
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry point that fires every 6 hours.
- **Configuration choices:**  
  Uses an interval rule with `hoursInterval = 6`.
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **Merge Data Sources**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  - Workflow must be active for schedule execution
  - Timezone interpretation depends on workflow/server settings
  - No actual data fetch is performed here, so unless prior upstream data injection exists, the resulting payload may be minimal
- **Sub-workflow reference:**  
  None.

#### 2. Real-time Emissions Data Webhook
- **Type and role:** `n8n-nodes-base.webhook`  
  HTTP entry point for real-time emissions payload ingestion.
- **Configuration choices:**  
  - Method: `POST`
  - Path: `carbon-emissions-webhook`
  - Response mode: `lastNode`
- **Key expressions or variables:**  
  Incoming payload is typically accessed later via `$json.body`.
- **Input and output connections:**  
  No input. Outputs to **Merge Data Sources**.
- **Version-specific requirements:**  
  Uses typeVersion `2.1`.
- **Edge cases / failures:**  
  - Webhook path collisions with other workflows
  - Invalid or unexpectedly nested JSON body
  - Long execution time may delay HTTP response because response mode is `lastNode`
- **Sub-workflow reference:**  
  None.

#### 3. Merge Data Sources
- **Type and role:** `n8n-nodes-base.merge`  
  Combines the scheduled and webhook branches into one downstream stream.
- **Configuration choices:**  
  Default merge settings; no special merge strategy is defined in the JSON.
- **Key expressions or variables:**  
  None directly.
- **Input and output connections:**  
  Inputs from **Scheduled Carbon Data Collection** and **Real-time Emissions Data Webhook**.  
  Output to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `3.2`.
- **Edge cases / failures:**  
  - Merge behavior depends on timing and default mode in the installed n8n version
  - If one branch emits incomplete data, supervisor input may differ in shape
- **Sub-workflow reference:**  
  None.

---

## Block B — Supervisor AI Orchestration

### Overview
This block receives unified emissions input and uses a supervisor agent to decide what to do next. It can invoke specialized AI tools and must return a structured output matching a predefined schema.

### Nodes Involved
- Carbon Supervisor Agent
- Supervisor Model
- Supervisor Output Parser

### Node Details

#### 4. Carbon Supervisor Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`  
  Main orchestration agent that interprets emissions data and decides whether to monitor, optimize, enforce policy, or generate a report.
- **Configuration choices:**  
  - Input text:
    `{{ $json.emissions_data || $json.body || $json }}`
    This makes the node tolerant of different input structures.
  - System message defines the agent as a carbon sustainability supervisor coordinating sub-agents and strategy execution.
  - Structured output parser enabled.
- **Key expressions or variables:**  
  - `$json.emissions_data`
  - `$json.body`
  - `$json`
- **Input and output connections:**  
  Main input from **Merge Data Sources**.  
  Main output to **Route by Action Type**.  
  AI model input from **Supervisor Model**.  
  AI output parser input from **Supervisor Output Parser**.  
  AI tool connections from:
  - **Carbon Monitoring Agent**
  - **Carbon Optimization Agent**
  - **Policy Enforcement Agent**
  - **ESG Reporting Agent**
- **Version-specific requirements:**  
  Uses typeVersion `3.1`; requires LangChain-compatible n8n AI nodes.
- **Edge cases / failures:**  
  - LLM may fail to produce schema-compliant output
  - Tool calls may fail if downstream tool nodes are misconfigured
  - Large payloads may exceed model context limits
  - If input is not well structured, action selection may be inconsistent
- **Sub-workflow reference:**  
  None.

#### 5. Supervisor Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT‑4o model used by the supervisor agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively deterministic orchestration
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI language model output to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`. Requires OpenAI credentials.
- **Edge cases / failures:**  
  - Invalid or expired OpenAI API key
  - Rate limiting
  - Model availability issues
- **Sub-workflow reference:**  
  None.

#### 6. Supervisor Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual schema for the supervisor output.
- **Configuration choices:**  
  Manual JSON schema with:
  - `action`: enum of `monitor`, `optimize`, `enforce_policy`, `generate_report`
  - `priority`: enum of `critical`, `high`, `medium`, `low`
  - `carbon_metrics`: object
  - `recommendations`: array
  - `policy_violations`: array
  - `requires_approval`: boolean
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI output parser connection to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  - Missing required structure from the model can cause parse failure
  - Arrays/objects may be syntactically valid but semantically inconsistent with downstream expectations
- **Sub-workflow reference:**  
  None.

---

## Block C — Monitoring Specialist Layer

### Overview
This block equips the monitoring specialist agent with a dedicated model and tools for KPI calculation, database access, and custom carbon computations. It supports anomaly detection, validation, and KPI enrichment when invoked by the supervisor.

### Nodes Involved
- Carbon Monitoring Agent
- Monitoring Model
- Carbon Calculations Tool
- Carbon Database Tool
- KPI Calculator

### Node Details

#### 7. Carbon Monitoring Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized agent callable by the supervisor for monitoring and validation tasks.
- **Configuration choices:**  
  - Input text:
    `{{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
  - System message focuses on emissions collection, anomaly detection, KPI calculation, and data quality validation.
  - Tool description explains monitoring scope across cloud and enterprise operations.
- **Key expressions or variables:**  
  `$fromAI('monitoring_task', ...)`
- **Input and output connections:**  
  AI language model from **Monitoring Model**.  
  AI tools from **Carbon Calculations Tool**, **Carbon Database Tool**, **KPI Calculator**.  
  AI tool output to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `3`.
- **Edge cases / failures:**  
  - Supervisor may call this agent without enough task detail
  - Tool invocation may fail if inputs are not numeric or DB connection is absent
- **Sub-workflow reference:**  
  None.

#### 8. Monitoring Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT‑4o model for the monitoring agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1` for stable analytical behavior
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI language model output to **Carbon Monitoring Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  Same OpenAI auth and quota risks as other model nodes.
- **Sub-workflow reference:**  
  None.

#### 9. Carbon Calculations Tool
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCode`  
  Custom JavaScript tool for carbon metrics calculations.
- **Configuration choices:**  
  Code accepts AI-supplied parameters:
  - `emissions_value` as number
  - `energy_kwh` as number defaulting to `0`
  - `operation` as string among `footprint`, `intensity`, `efficiency`, `savings`
  
  Constants:
  - Grid carbon intensity = `0.475`
  - Renewable intensity = `0.05`
  
  Outputs:
  - `result`
  - `unit = 'kg CO2e'`
  - `operation`
- **Key expressions or variables:**  
  - `$fromAI('emissions_value', ...)`
  - `$fromAI('energy_kwh', ...)`
  - `$fromAI('operation', ...)`
- **Input and output connections:**  
  AI tool output to **Carbon Monitoring Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  - `efficiency` calculation can behave oddly when `energy = 0`
  - `toFixed(2)` returns a string, not a numeric type
  - Invalid `operation` falls back to returning raw emissions
  - AI-provided non-numeric values may break execution if parsing fails
- **Sub-workflow reference:**  
  None.

#### 10. Carbon Database Tool
- **Type and role:** `n8n-nodes-base.postgresTool`  
  AI-callable PostgreSQL tool for reading/updating historical carbon and KPI data.
- **Configuration choices:**  
  - Schema: `public`
  - Table not specified in the exported JSON
  - Manual tool description explains usage for carbon emissions, trends, and policy configuration
- **Key expressions or variables:**  
  None visible in configuration.
- **Input and output connections:**  
  AI tool output to both **Carbon Monitoring Agent** and **Carbon Optimization Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `2.6`. Requires PostgreSQL credentials, though none are embedded in the export.
- **Edge cases / failures:**  
  - Missing credentials or table selection
  - SQL permission issues
  - Empty table configuration may require manual completion before use
- **Sub-workflow reference:**  
  None.

#### 11. KPI Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  General-purpose AI calculator tool.
- **Configuration choices:**  
  Default configuration.
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI tool output to **Carbon Monitoring Agent** and **Approval Workflow Tool**.
- **Version-specific requirements:**  
  Uses typeVersion `1`.
- **Edge cases / failures:**  
  - Incorrect expression generation by AI
  - Limited usefulness if the LLM does not formulate valid calculations
- **Sub-workflow reference:**  
  None.

---

## Block D — Optimization Specialist Layer

### Overview
This block supports optimization analysis and downstream approval logic. The optimization agent can query historical data, calculate metrics, and involve a human approval tool through Slack.

### Nodes Involved
- Carbon Optimization Agent
- Optimization Model
- Approval Workflow Tool
- Carbon Database Tool
- KPI Calculator

### Node Details

#### 12. Carbon Optimization Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized optimization agent callable by the supervisor.
- **Configuration choices:**  
  - Input text:
    `{{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
  - System message focuses on identifying high-carbon workloads, low-carbon alternatives, savings, and prioritization.
- **Key expressions or variables:**  
  `$fromAI('optimization_task', ...)`
- **Input and output connections:**  
  AI language model from **Optimization Model**.  
  AI tools from **Carbon Database Tool**, **KPI Calculator**, **Approval Workflow Tool**.  
  AI tool output to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `3`.
- **Edge cases / failures:**  
  - Potential mismatch between optimization output structure and downstream Set node expectations
  - Approval tool may be invoked unexpectedly by the agent depending on prompt behavior
- **Sub-workflow reference:**  
  None.

#### 13. Optimization Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.3` for somewhat more exploratory recommendations
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI language model output to **Carbon Optimization Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  Standard OpenAI credential, quota, and availability concerns.
- **Sub-workflow reference:**  
  None.

#### 14. Approval Workflow Tool
- **Type and role:** `n8n-nodes-base.slackHitlTool`  
  AI-callable Slack human-in-the-loop tool for approval requests.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Message:
    `{{ $fromAI('approval_message', 'The approval request message') }}`
  - User field is empty in the export and must be configured
- **Key expressions or variables:**  
  `$fromAI('approval_message', ...)`
- **Input and output connections:**  
  AI tool output to **Carbon Optimization Agent**.  
  Also receives AI tool connection from **KPI Calculator**.
- **Version-specific requirements:**  
  Uses typeVersion `2.4`. Requires Slack OAuth2 credentials and valid webhook/tool setup.
- **Edge cases / failures:**  
  - Missing Slack target user
  - OAuth scope issues
  - Human approval delays
  - Possible confusion with the separate downstream Slack approval node used outside the AI tool layer
- **Sub-workflow reference:**  
  None.

---

## Block E — Policy Specialist Layer

### Overview
This block enables policy validation against sustainability targets and carbon reduction rules. It relies on a dedicated GPT‑4o model and Google Sheets as a policy lookup source.

### Nodes Involved
- Policy Enforcement Agent
- Policy Model
- Policy Sheets Tool

### Node Details

#### 15. Policy Enforcement Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**  
  - Input text:
    `{{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
  - System prompt directs the agent to evaluate policy compliance, severity, corrective actions, and escalation needs.
- **Key expressions or variables:**  
  `$fromAI('policy_task', ...)`
- **Input and output connections:**  
  AI language model from **Policy Model**.  
  AI tool from **Policy Sheets Tool**.  
  AI tool output to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `3`.
- **Edge cases / failures:**  
  - Policy sheet may be empty or unconfigured
  - Output may not align with Slack alert formatting assumptions
- **Sub-workflow reference:**  
  None.

#### 16. Policy Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI language model output to **Policy Enforcement Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  Standard LLM credential and quota issues.
- **Sub-workflow reference:**  
  None.

#### 17. Policy Sheets Tool
- **Type and role:** `n8n-nodes-base.googleSheetsTool`  
  AI-callable Google Sheets reader for policy and target data.
- **Configuration choices:**  
  - Document ID empty in export
  - Sheet name empty in export
  - OAuth2 credentials configured
  - Tool description clarifies usage for carbon reduction policies and sustainability targets
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI tool output to **Policy Enforcement Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `4.7`. Requires Google Sheets OAuth2 credentials.
- **Edge cases / failures:**  
  - Missing document/sheet mapping
  - Sheet access denied
  - Agent may request unavailable columns
- **Sub-workflow reference:**  
  None.

---

## Block F — ESG Reporting Specialist Layer

### Overview
This block provides a reporting-focused AI specialist that can generate stakeholder-ready ESG summaries and compliance documentation.

### Nodes Involved
- ESG Reporting Agent
- ESG Model

### Node Details

#### 18. ESG Reporting Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**  
  - Input text:
    `{{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
  - System message focuses on comprehensive ESG reporting, KPI presentation, and regulatory formatting.
- **Key expressions or variables:**  
  `$fromAI('reporting_task', ...)`
- **Input and output connections:**  
  AI language model from **ESG Model**.  
  AI tool output to **Carbon Supervisor Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `3`.
- **Edge cases / failures:**  
  - Generated report structure may differ from downstream Google Sheets/email field assumptions
- **Sub-workflow reference:**  
  None.

#### 19. ESG Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  AI language model output to **ESG Reporting Agent**.
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases / failures:**  
  Standard OpenAI authentication and rate limit issues.
- **Sub-workflow reference:**  
  None.

---

## Block G — Action Routing

### Overview
This block directs the supervisor’s structured result into one of four operational branches. The route depends entirely on the `action` property returned by the supervisor.

### Nodes Involved
- Route by Action Type

### Node Details

#### 20. Route by Action Type
- **Type and role:** `n8n-nodes-base.switch`
- **Configuration choices:**  
  Four explicit routing conditions:
  - `{{ $json.action }} == 'monitor'`
  - `== 'optimize'`
  - `== 'enforce_policy'`
  - `== 'generate_report'`
- **Key expressions or variables:**  
  `$json.action`
- **Input and output connections:**  
  Input from **Carbon Supervisor Agent**.  
  Outputs to:
  - **Store Carbon Metrics**
  - **Prepare Optimization Data**
  - **Send Policy Alert**
  - **Update ESG Report**
- **Version-specific requirements:**  
  Uses typeVersion `3.4`.
- **Edge cases / failures:**  
  - If action is absent or invalid, no branch will match
  - Any typo in the model output can stall downstream processing
- **Sub-workflow reference:**  
  None.

---

## Block H — Monitoring Result Persistence

### Overview
This branch handles the `monitor` action by storing the structured sustainability output in PostgreSQL.

### Nodes Involved
- Store Carbon Metrics

### Node Details

#### 21. Store Carbon Metrics
- **Type and role:** `n8n-nodes-base.postgres`
- **Configuration choices:**  
  Inserts into table `public.carbon_metrics` with explicit field mapping:
  - `action`
  - `priority`
  - `timestamp = $now`
  - `carbon_metrics = JSON.stringify(...)`
  - `recommendations = JSON.stringify(...)`
  - `policy_violations = JSON.stringify(...)`
  - `requires_approval`
- **Key expressions or variables:**  
  - `{{ $json.action }}`
  - `{{ $json.priority }}`
  - `{{ $now }}`
  - `{{ JSON.stringify($json.carbon_metrics) }}`
  - `{{ JSON.stringify($json.recommendations) }}`
  - `{{ JSON.stringify($json.policy_violations) }}`
  - `{{ $json.requires_approval }}`
- **Input and output connections:**  
  Input from **Route by Action Type** on the monitor route.  
  Output to **Consolidate All Actions**.
- **Version-specific requirements:**  
  Uses typeVersion `2.6`. Requires PostgreSQL credentials configured in the environment.
- **Edge cases / failures:**  
  - Table must exist with compatible column types
  - JSON strings may exceed column length if using text-constrained schema
  - Null arrays/objects can cause insert issues depending on DB schema
- **Sub-workflow reference:**  
  None.

---

## Block I — Optimization Approval and Execution

### Overview
This branch handles optimization recommendations. It first reshapes the AI output into simpler fields, then decides whether the strategy requires approval; approved strategies are logged, while low-risk ones are auto-executed and logged separately.

### Nodes Involved
- Prepare Optimization Data
- Check Approval Required
- Request Strategy Approval
- Log Approved Strategy
- Auto-Execute Strategy

### Node Details

#### 22. Prepare Optimization Data
- **Type and role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Creates four fields:
  - `optimization_summary = $json.output.recommendations`
  - `carbon_savings = $json.output.carbon_metrics.potential_savings`
  - `priority = $json.output.priority`
  - `requires_approval = $json.output.requires_approval`
- **Key expressions or variables:**  
  - `{{ $json.output.recommendations }}`
  - `{{ $json.output.carbon_metrics.potential_savings }}`
  - `{{ $json.output.priority }}`
  - `{{ $json.output.requires_approval }}`
- **Input and output connections:**  
  Input from **Route by Action Type** on the optimize route.  
  Output to **Check Approval Required**.
- **Version-specific requirements:**  
  Uses typeVersion `3.4`.
- **Edge cases / failures:**  
  - This node expects data under `$json.output`, but the supervisor parser likely emits top-level fields, not nested `output`
  - If structure is top-level, all expressions here will resolve undefined
  - `carbon_savings` typed as number may fail if the source value is missing or non-numeric
- **Sub-workflow reference:**  
  None.

#### 23. Check Approval Required
- **Type and role:** `n8n-nodes-base.if`
- **Configuration choices:**  
  Checks whether `{{ $json.requires_approval }}` is boolean true.
- **Key expressions or variables:**  
  `$json.requires_approval`
- **Input and output connections:**  
  Input from **Prepare Optimization Data**.  
  True branch to **Request Strategy Approval**.  
  False branch to **Auto-Execute Strategy**.
- **Version-specific requirements:**  
  Uses typeVersion `2.3`.
- **Edge cases / failures:**  
  - Loose validation may pass unexpected values
  - If previous node produced null/undefined, branch logic may not behave as intended
- **Sub-workflow reference:**  
  None.

#### 24. Request Strategy Approval
- **Type and role:** `n8n-nodes-base.slack`
- **Configuration choices:**  
  - Operation: `sendAndWait`
  - Sends to a configured Slack channel
  - Uses double approval with custom reject label `Reject`
  - Message includes:
    - priority
    - estimated carbon savings
    - recommendation list
- **Key expressions or variables:**  
  - `{{ $json.priority }}`
  - `{{ $json.carbon_savings }}`
  - `{{ $json.optimization_summary.map(...) }}`
- **Input and output connections:**  
  Input from **Check Approval Required** true branch.  
  Output to **Log Approved Strategy**.
- **Version-specific requirements:**  
  Uses typeVersion `2.4`. Requires Slack OAuth2 and channel ID.
- **Edge cases / failures:**  
  - Placeholder channel ID must be replaced
  - `optimization_summary.map(...)` assumes an array
  - If approval is rejected, downstream handling is not explicitly modeled; behavior depends on Slack node output semantics
- **Sub-workflow reference:**  
  None.

#### 25. Log Approved Strategy
- **Type and role:** `n8n-nodes-base.postgres`
- **Configuration choices:**  
  Inserts into `public.approved_strategies`:
  - `status = approved`
  - `priority`
  - `strategy = JSON.stringify(optimization_summary)`
  - `timestamp = $now`
  - `approved_by = $json.user`
  - `carbon_savings`
- **Key expressions or variables:**  
  - `{{ $json.priority }}`
  - `{{ JSON.stringify($json.optimization_summary) }}`
  - `{{ $now }}`
  - `{{ $json.user }}`
  - `{{ $json.carbon_savings }}`
- **Input and output connections:**  
  Input from **Request Strategy Approval**.  
  Output to **Consolidate All Actions**.
- **Version-specific requirements:**  
  Uses typeVersion `2.6`.
- **Edge cases / failures:**  
  - `user` field depends on Slack approval response shape
  - Rejected approvals may still need separate handling not present here
  - DB schema must support all mapped columns
- **Sub-workflow reference:**  
  None.

#### 26. Auto-Execute Strategy
- **Type and role:** `n8n-nodes-base.postgres`
- **Configuration choices:**  
  Inserts into `public.approved_strategies`:
  - `status = auto_executed`
  - `priority`
  - `strategy = JSON.stringify(optimization_summary)`
  - `timestamp = $now`
  - `carbon_savings`
- **Key expressions or variables:**  
  - `{{ $json.priority }}`
  - `{{ JSON.stringify($json.optimization_summary) }}`
  - `{{ $now }}`
  - `{{ $json.carbon_savings }}`
- **Input and output connections:**  
  Input from **Check Approval Required** false branch.  
  Output to **Consolidate All Actions**.
- **Version-specific requirements:**  
  Uses typeVersion `2.6`.
- **Edge cases / failures:**  
  - Despite the name, this node only logs auto-execution; it does not actually call an infrastructure API to implement a strategy
- **Sub-workflow reference:**  
  None.

---

## Block J — Policy Violation Alerting

### Overview
This branch handles policy enforcement outcomes by sending a structured Slack alert listing violations and recommended actions.

### Nodes Involved
- Send Policy Alert

### Node Details

#### 27. Send Policy Alert
- **Type and role:** `n8n-nodes-base.slack`
- **Configuration choices:**  
  Sends a formatted message to a Slack channel with:
  - priority
  - count of violations
  - bullet list of policy violations using `policy` and `description`
  - bullet list of recommendations
- **Key expressions or variables:**  
  - `{{ $json.output.priority }}`
  - `{{ $json.output.policy_violations.length }}`
  - `{{ $json.output.policy_violations.map(...) }}`
  - `{{ $json.output.recommendations.map(...) }}`
- **Input and output connections:**  
  Input from **Route by Action Type** on the enforce_policy route.  
  Output to **Consolidate All Actions**.
- **Version-specific requirements:**  
  Uses typeVersion `2.4`.
- **Edge cases / failures:**  
  - Like the optimization preparation node, this expects `$json.output.*`, which may not exist if parser output is top-level
  - Placeholder Slack channel must be configured
  - Violation items are assumed to contain `policy` and `description`
- **Sub-workflow reference:**  
  None.

---

## Block K — ESG Report Publishing

### Overview
This branch appends ESG report data to Google Sheets and then emails a formatted stakeholder summary.

### Nodes Involved
- Update ESG Report
- Send ESG Report Email

### Node Details

#### 28. Update ESG Report
- **Type and role:** `n8n-nodes-base.googleSheets`
- **Configuration choices:**  
  - Operation: `append`
  - Document ID and sheet name are not configured in the export
- **Key expressions or variables:**  
  None shown; append mapping likely relies on incoming JSON fields and manual configuration in the UI.
- **Input and output connections:**  
  Input from **Route by Action Type** on the generate_report route.  
  Output to **Send ESG Report Email**.
- **Version-specific requirements:**  
  Uses typeVersion `4.7`. Requires Google Sheets OAuth2 credentials.
- **Edge cases / failures:**  
  - Missing document ID or sheet name
  - Append schema mismatch with incoming fields
  - Sheet write permissions
- **Sub-workflow reference:**  
  None.

#### 29. Send ESG Report Email
- **Type and role:** `n8n-nodes-base.emailSend`
- **Configuration choices:**  
  Sends an HTML email with:
  - total emissions
  - carbon intensity
  - compliance rate
  - bullet list of recommendations
  - subject line using current date
  - placeholder sender and stakeholder emails
- **Key expressions or variables:**  
  - `{{ $json.carbon_metrics.total_emissions }}`
  - `{{ $json.carbon_metrics.carbon_intensity }}`
  - `{{ $json.carbon_metrics.compliance_rate }}`
  - `{{ $json.recommendations.map(...) }}`
  - `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Input and output connections:**  
  Input from **Update ESG Report**.  
  Output to **Consolidate All Actions**.
- **Version-specific requirements:**  
  Uses typeVersion `2.1`. Requires email transport configuration in n8n.
- **Edge cases / failures:**  
  - Placeholder addresses must be replaced
  - `recommendations` must be an array
  - Missing email credentials or SMTP transport settings
- **Sub-workflow reference:**  
  None.

---

## Block L — Consolidation and Final KPI Logging

### Overview
This block merges all terminal branches, sends a final Slack completion message, and logs a KPI dashboard entry for the workflow run.

### Nodes Involved
- Consolidate All Actions
- Send Summary Notification
- Update KPI Dashboard

### Node Details

#### 30. Consolidate All Actions
- **Type and role:** `n8n-nodes-base.merge`
- **Configuration choices:**  
  Configured with `numberInputs = 5`, expecting up to five terminal inputs:
  - stored monitoring result
  - approved strategy log
  - auto-executed strategy log
  - policy alert
  - report email
- **Key expressions or variables:**  
  None.
- **Input and output connections:**  
  Inputs from:
  - **Store Carbon Metrics**
  - **Log Approved Strategy**
  - **Auto-Execute Strategy**
  - **Send Policy Alert**
  - **Send ESG Report Email**
  
  Output to **Send Summary Notification**.
- **Version-specific requirements:**  
  Uses typeVersion `3.2`.
- **Edge cases / failures:**  
  - Depending on merge mode semantics, branches that do not execute may affect behavior
  - In single-route execution patterns, careful testing is needed to ensure the merge does not wait indefinitely for non-triggered inputs
- **Sub-workflow reference:**  
  None.

#### 31. Send Summary Notification
- **Type and role:** `n8n-nodes-base.slack`
- **Configuration choices:**  
  Sends a Slack completion message summarizing:
  - action or “Multiple actions”
  - priority or “N/A”
  - success status
- **Key expressions or variables:**  
  - `{{ $json.action || 'Multiple actions' }}`
  - `{{ $json.priority || 'N/A' }}`
- **Input and output connections:**  
  Input from **Consolidate All Actions**.  
  Output to **Update KPI Dashboard**.
- **Version-specific requirements:**  
  Uses typeVersion `2.4`. Requires Slack OAuth2 and configured summary channel.
- **Edge cases / failures:**  
  - Placeholder summary channel ID must be replaced
  - If merged payload lacks `action` and `priority`, fallback text is used
- **Sub-workflow reference:**  
  None.

#### 32. Update KPI Dashboard
- **Type and role:** `n8n-nodes-base.postgres`
- **Configuration choices:**  
  Inserts into `public.kpi_dashboard`:
  - `status = completed`
  - `timestamp = $now`
  - `total_actions = $json.action ? 1 : 0`
  - `workflow_run_id = $execution.id`
- **Key expressions or variables:**  
  - `{{ $now }}`
  - `{{ $json.action ? 1 : 0 }}`
  - `{{ $execution.id }}`
- **Input and output connections:**  
  Input from **Send Summary Notification**.
- **Version-specific requirements:**  
  Uses typeVersion `2.6`.
- **Edge cases / failures:**  
  - If merged output does not carry `action`, `total_actions` becomes 0 even for successful runs
  - DB schema must exist and be compatible
- **Sub-workflow reference:**  
  None.

---

## Block M — Documentation / Canvas Notes

### Overview
These sticky notes document business intent, setup expectations, prerequisites, and branch behavior directly on the canvas. They do not execute but are important for operational understanding.

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
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes full workflow purpose and end-to-end automation.
- **Input and output connections:**  
  None.
- **Edge cases / failures:**  
  None.
- **Sub-workflow reference:**  
  None.

#### 34. Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Lists setup steps for triggers, credentials, LLM, approval thresholds, and Sheets IDs.
- **Input and output connections:**  
  None.

#### 35. Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Lists prerequisites, use cases, customization ideas, and benefits.
- **Input and output connections:**  
  None.

#### 36. Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels optimization and policy evaluation area.
- **Input and output connections:**  
  None.

#### 37. Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels supervisor orchestration area.
- **Input and output connections:**  
  None.

#### 38. Sticky Note5
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels approval routing area.
- **Input and output connections:**  
  None.

#### 39. Sticky Note6
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels reporting and notification area.
- **Input and output connections:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Carbon Data Collection | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry every 6 hours |  | Merge Data Sources | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Real-time Emissions Data Webhook | n8n-nodes-base.webhook | Real-time webhook entry for emissions events |  | Merge Data Sources | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Merge Data Sources | n8n-nodes-base.merge | Unifies scheduled and webhook input streams | Scheduled Carbon Data Collection, Real-time Emissions Data Webhook | Carbon Supervisor Agent | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Carbon Supervisor Agent | @n8n/n8n-nodes-langchain.agent | Central AI orchestrator that selects action and delegates to specialist agents | Merge Data Sources | Route by Action Type | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Supervisor Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model powering supervisor orchestration |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Supervisor Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser constraining supervisor response schema |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Monitoring Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist monitoring agent for KPI calculation and anomaly detection |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Monitoring Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for monitoring agent |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Optimization Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist optimization agent for reduction recommendations |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Optimization Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for optimization agent |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Enforcement Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist policy compliance agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for policy agent |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Reporting Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist ESG report generation agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT‑4o model for ESG reporting agent |  | ESG Reporting Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Calculations Tool | @n8n/n8n-nodes-langchain.toolCode | Custom carbon computation tool |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| KPI Calculator | @n8n/n8n-nodes-langchain.toolCalculator | General math tool for AI agents |  | Carbon Monitoring Agent, Approval Workflow Tool | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Database Tool | n8n-nodes-base.postgresTool | AI-callable PostgreSQL access for historical carbon data |  | Carbon Monitoring Agent, Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Sheets Tool | n8n-nodes-base.googleSheetsTool | AI-callable Google Sheets policy lookup |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Approval Workflow Tool | n8n-nodes-base.slackHitlTool | AI-callable Slack approval tool |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Route by Action Type | n8n-nodes-base.switch | Routes supervisor output to action-specific branches | Carbon Supervisor Agent | Store Carbon Metrics, Prepare Optimization Data, Send Policy Alert, Update ESG Report | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Store Carbon Metrics | n8n-nodes-base.postgres | Persists monitoring results in PostgreSQL | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Prepare Optimization Data | n8n-nodes-base.set | Normalizes optimization output for approval routing | Route by Action Type | Check Approval Required | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Send Policy Alert | n8n-nodes-base.slack | Sends Slack alert for carbon policy violations | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update ESG Report | n8n-nodes-base.googleSheets | Appends ESG report data to Google Sheets | Route by Action Type | Send ESG Report Email | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Check Approval Required | n8n-nodes-base.if | Determines whether optimization needs human approval | Prepare Optimization Data | Request Strategy Approval, Auto-Execute Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Request Strategy Approval | n8n-nodes-base.slack | Sends Slack approval request and waits for response | Check Approval Required | Log Approved Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Log Approved Strategy | n8n-nodes-base.postgres | Logs human-approved strategy in PostgreSQL | Request Strategy Approval | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Auto-Execute Strategy | n8n-nodes-base.postgres | Logs low-risk strategy as auto-executed | Check Approval Required | Consolidate All Actions | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Send ESG Report Email | n8n-nodes-base.emailSend | Emails stakeholder ESG summary | Update ESG Report | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Consolidate All Actions | n8n-nodes-base.merge | Merges terminal branches before final notification | Store Carbon Metrics, Log Approved Strategy, Auto-Execute Strategy, Send Policy Alert, Send ESG Report Email | Send Summary Notification | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send Summary Notification | n8n-nodes-base.slack | Sends final workflow completion message to Slack | Consolidate All Actions | Update KPI Dashboard | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update KPI Dashboard | n8n-nodes-base.postgres | Logs workflow run completion in KPI dashboard table | Send Summary Notification |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  | ## How It Works<br>This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas setup guidance |  |  | ## Setup Steps<br>1. Connect scheduled trigger and webhook nodes to your emissions data sources.<br>2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account).<br>3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible).<br>4. Set approval thresholds in the Check Approval Required node.<br>5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas prerequisites and benefits note |  |  | ## Prerequisites<br>- OpenAI or compatible LLM API key<br>- Slack bot token<br>- Gmail OAuth2 credentials<br>- Google Sheets service account<br>## Use Cases<br>- Corporate sustainability teams automating monthly ESG reporting<br>## Customisation<br>- Swap LLM models per agent for cost or accuracy trade-offs<br>## Benefits<br>- Eliminates manual emissions data aggregation and report generation |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas label for strategy/policy area |  |  | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas label for supervisor area |  |  | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas label for approval routing area |  |  | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas label for reporting/notification area |  |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n.

## Prerequisites
1. Create credentials for:
   - **OpenAI API**
   - **Slack OAuth2**
   - **Google Sheets OAuth2**
   - **PostgreSQL**
   - **Email transport** for Email Send node
2. Ensure the following database tables exist, or adapt node mappings:
   - `public.carbon_metrics`
   - `public.approved_strategies`
   - `public.kpi_dashboard`
3. Prepare:
   - A Slack channel for policy alerts
   - A Slack channel for approvals
   - A Slack channel for final summaries
   - A Google Sheet for policy rules
   - A Google Sheet for ESG reporting
4. Install or enable the n8n AI/LangChain nodes if not already available.

## Build Steps

1. **Create a Schedule Trigger node**
   - Name: `Scheduled Carbon Data Collection`
   - Type: Schedule Trigger
   - Set interval to every `6 hours`

2. **Create a Webhook node**
   - Name: `Real-time Emissions Data Webhook`
   - Type: Webhook
   - Method: `POST`
   - Path: `carbon-emissions-webhook`
   - Response mode: `Last Node`

3. **Create a Merge node**
   - Name: `Merge Data Sources`
   - Connect both trigger nodes into it
   - Leave default merge behavior unless you want a stricter source-handling strategy

4. **Create the supervisor language model node**
   - Name: `Supervisor Model`
   - Type: OpenAI Chat Model
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach OpenAI credentials

5. **Create a Structured Output Parser node**
   - Name: `Supervisor Output Parser`
   - Type: Structured Output Parser
   - Use manual schema with fields:
     - `action` enum: `monitor`, `optimize`, `enforce_policy`, `generate_report`
     - `priority` enum: `critical`, `high`, `medium`, `low`
     - `carbon_metrics` object
     - `recommendations` array
     - `policy_violations` array
     - `requires_approval` boolean

6. **Create the main AI Agent node**
   - Name: `Carbon Supervisor Agent`
   - Type: AI Agent
   - Prompt/input text:
     `{{ $json.emissions_data || $json.body || $json }}`
   - System message: describe the agent as a Carbon Sustainability Supervisor that analyzes emissions data, invokes specialist sub-agents, and coordinates monitoring, optimization, policy enforcement, and ESG reporting
   - Enable structured output parser
   - Connect:
     - `Supervisor Model` to the agent’s model input
     - `Supervisor Output Parser` to the agent’s parser input
     - `Merge Data Sources` to the main input

7. **Create the monitoring model**
   - Name: `Monitoring Model`
   - Type: OpenAI Chat Model
   - Model: `gpt-4o`
   - Temperature: `0.1`

8. **Create the monitoring specialist agent tool**
   - Name: `Carbon Monitoring Agent`
   - Type: AI Agent Tool
   - Text input:
     `{{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
   - System message: monitoring, anomaly detection, KPI calculation, data-quality validation
   - Tool description: carbon monitoring across AWS/Azure/GCP and enterprise operations

9. **Create the code tool**
   - Name: `Carbon Calculations Tool`
   - Type: AI Code Tool
   - Paste the carbon calculation JavaScript from the workflow
   - Keep the operations:
     - footprint
     - intensity
     - efficiency
     - savings

10. **Create the PostgreSQL AI tool**
    - Name: `Carbon Database Tool`
    - Type: Postgres Tool
    - Schema: `public`
    - Select the appropriate table or configure generic DB access as supported in your n8n version
    - Add description for querying emissions, KPI metrics, historical trends, and policy configurations
    - Attach PostgreSQL credentials

11. **Create the AI calculator tool**
    - Name: `KPI Calculator`
    - Type: Calculator Tool
    - Default setup is sufficient

12. **Connect monitoring components**
    - Connect `Monitoring Model` to `Carbon Monitoring Agent`
    - Connect `Carbon Calculations Tool`, `Carbon Database Tool`, and `KPI Calculator` as tools into `Carbon Monitoring Agent`
    - Connect `Carbon Monitoring Agent` as a tool into `Carbon Supervisor Agent`

13. **Create the optimization model**
    - Name: `Optimization Model`
    - Type: OpenAI Chat Model
    - Model: `gpt-4o`
    - Temperature: `0.3`

14. **Create the optimization specialist agent tool**
    - Name: `Carbon Optimization Agent`
    - Type: AI Agent Tool
    - Text input:
      `{{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
    - System message: energy pattern analysis, low-carbon alternatives, potential savings, prioritization

15. **Create the Slack human-in-the-loop tool**
    - Name: `Approval Workflow Tool`
    - Type: Slack HITL Tool
    - Authentication: OAuth2
    - Configure target user
    - Message:
      `{{ $fromAI('approval_message', 'The approval request message') }}`
    - Attach Slack credentials

16. **Connect optimization components**
    - Connect `Optimization Model` to `Carbon Optimization Agent`
    - Connect `Carbon Database Tool`, `KPI Calculator`, and `Approval Workflow Tool` as tools into `Carbon Optimization Agent`
    - Connect `Carbon Optimization Agent` as a tool into `Carbon Supervisor Agent`

17. **Create the policy model**
    - Name: `Policy Model`
    - Type: OpenAI Chat Model
    - Model: `gpt-4o`
    - Temperature: `0.1`

18. **Create the Google Sheets AI tool**
    - Name: `Policy Sheets Tool`
    - Type: Google Sheets Tool
    - Select policy spreadsheet document ID
    - Select policy sheet name
    - Add description for reading carbon reduction policies and sustainability targets
    - Attach Google Sheets credentials

19. **Create the policy specialist agent tool**
    - Name: `Policy Enforcement Agent`
    - Type: AI Agent Tool
    - Text input:
      `{{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
    - System message: policy checking, target validation, corrective actions, escalation

20. **Connect policy components**
    - Connect `Policy Model` to `Policy Enforcement Agent`
    - Connect `Policy Sheets Tool` as a tool into `Policy Enforcement Agent`
    - Connect `Policy Enforcement Agent` as a tool into `Carbon Supervisor Agent`

21. **Create the ESG model**
    - Name: `ESG Model`
    - Type: OpenAI Chat Model
    - Model: `gpt-4o`
    - Temperature: `0.2`

22. **Create the ESG specialist agent tool**
    - Name: `ESG Reporting Agent`
    - Type: AI Agent Tool
    - Text input:
      `{{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
    - System message: ESG report generation, compliance formatting, stakeholder summaries

23. **Connect ESG components**
    - Connect `ESG Model` to `ESG Reporting Agent`
    - Connect `ESG Reporting Agent` as a tool into `Carbon Supervisor Agent`

24. **Create a Switch node**
    - Name: `Route by Action Type`
    - Type: Switch
    - Add 4 rules based on `{{ $json.action }}`
      - equals `monitor`
      - equals `optimize`
      - equals `enforce_policy`
      - equals `generate_report`
    - Connect `Carbon Supervisor Agent` to this node

25. **Create the monitoring persistence node**
    - Name: `Store Carbon Metrics`
    - Type: PostgreSQL
    - Operation: insert
    - Table: `public.carbon_metrics`
    - Map:
      - `action` = `{{ $json.action }}`
      - `priority` = `{{ $json.priority }}`
      - `timestamp` = `{{ $now }}`
      - `carbon_metrics` = `{{ JSON.stringify($json.carbon_metrics) }}`
      - `recommendations` = `{{ JSON.stringify($json.recommendations) }}`
      - `policy_violations` = `{{ JSON.stringify($json.policy_violations) }}`
      - `requires_approval` = `{{ $json.requires_approval }}`
    - Connect from Switch monitor output

26. **Create a Set node for optimization normalization**
    - Name: `Prepare Optimization Data`
    - Type: Set
    - Add fields:
      - `optimization_summary`
      - `carbon_savings`
      - `priority`
      - `requires_approval`
    - To reproduce the JSON exactly, use:
      - `{{ $json.output.recommendations }}`
      - `{{ $json.output.carbon_metrics.potential_savings }}`
      - `{{ $json.output.priority }}`
      - `{{ $json.output.requires_approval }}`
    - Recommended correction if needed:
      - `{{ $json.recommendations }}`
      - `{{ $json.carbon_metrics.potential_savings }}`
      - `{{ $json.priority }}`
      - `{{ $json.requires_approval }}`
    - Connect from Switch optimize output

27. **Create an IF node**
    - Name: `Check Approval Required`
    - Condition: boolean true on `{{ $json.requires_approval }}`
    - Connect from `Prepare Optimization Data`

28. **Create the Slack approval node**
    - Name: `Request Strategy Approval`
    - Type: Slack
    - Operation: `sendAndWait`
    - Send to approval channel
    - Approval type: double
    - Reject label: `Reject`
    - Message should include:
      - priority
      - carbon savings
      - recommendation bullet list
    - Connect from IF true branch

29. **Create PostgreSQL logging for approved strategies**
    - Name: `Log Approved Strategy`
    - Type: PostgreSQL
    - Table: `public.approved_strategies`
    - Map:
      - `status` = `approved`
      - `priority` = `{{ $json.priority }}`
      - `strategy` = `{{ JSON.stringify($json.optimization_summary) }}`
      - `timestamp` = `{{ $now }}`
      - `approved_by` = `{{ $json.user }}`
      - `carbon_savings` = `{{ $json.carbon_savings }}`
    - Connect from `Request Strategy Approval`

30. **Create PostgreSQL logging for auto-executed strategies**
    - Name: `Auto-Execute Strategy`
    - Type: PostgreSQL
    - Table: `public.approved_strategies`
    - Map:
      - `status` = `auto_executed`
      - `priority` = `{{ $json.priority }}`
      - `strategy` = `{{ JSON.stringify($json.optimization_summary) }}`
      - `timestamp` = `{{ $now }}`
      - `carbon_savings` = `{{ $json.carbon_savings }}`
    - Connect from IF false branch

31. **Create the policy alert Slack node**
    - Name: `Send Policy Alert`
    - Type: Slack
    - Send to policy alerts channel
    - Use the formatted message listing priority, number of violations, violation details, and recommendations
    - To reproduce the export exactly, use `$json.output.*`
    - Recommended correction if needed: use top-level `$json.*`
    - Connect from Switch enforce_policy output

32. **Create the ESG reporting Google Sheets node**
    - Name: `Update ESG Report`
    - Type: Google Sheets
    - Operation: append
    - Choose report spreadsheet and sheet
    - Connect from Switch generate_report output

33. **Create the ESG email node**
    - Name: `Send ESG Report Email`
    - Type: Email Send
    - Configure recipient and sender emails
    - Subject:
      `ESG Sustainability Report - {{ $now.toFormat('yyyy-MM-dd') }}`
    - HTML body should include emissions, carbon intensity, compliance rate, and recommendations
    - Connect from `Update ESG Report`

34. **Create a final Merge node**
    - Name: `Consolidate All Actions`
    - Type: Merge
    - Set number of inputs to `5`
    - Connect inputs from:
      - `Store Carbon Metrics`
      - `Log Approved Strategy`
      - `Auto-Execute Strategy`
      - `Send Policy Alert`
      - `Send ESG Report Email`

35. **Create final Slack summary node**
    - Name: `Send Summary Notification`
    - Type: Slack
    - Send to summary channel
    - Message should report workflow completion, action, priority, and success status
    - Connect from `Consolidate All Actions`

36. **Create KPI logging node**
    - Name: `Update KPI Dashboard`
    - Type: PostgreSQL
    - Table: `public.kpi_dashboard`
    - Map:
      - `status` = `completed`
      - `timestamp` = `{{ $now }}`
      - `total_actions` = `{{ $json.action ? 1 : 0 }}`
      - `workflow_run_id` = `{{ $execution.id }}`
    - Connect from `Send Summary Notification`

37. **Add sticky notes if desired**
    - Recreate the descriptive notes for architecture, setup, prerequisites, approval routing, and reporting.

## Credential Configuration Notes
- **OpenAI:** required on all GPT‑4o model nodes
- **Slack OAuth2:** required on `Approval Workflow Tool`, `Request Strategy Approval`, `Send Policy Alert`, and `Send Summary Notification`
- **Google Sheets OAuth2:** required on `Policy Sheets Tool` and `Update ESG Report`
- **PostgreSQL:** required on all Postgres/Postgres Tool nodes
- **Email transport:** required on `Send ESG Report Email`

## Important Rebuild Cautions
- Replace all placeholder values:
  - policy alert channel
  - approval channel
  - summary channel
  - stakeholder email
  - sender email
- The exported workflow contains likely schema mismatches in some downstream nodes using `$json.output.*`. If your supervisor parser returns top-level fields, update those expressions accordingly.
- Test the final merge carefully; depending on n8n merge behavior, you may need to configure it so non-executed branches do not block completion.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. | Workflow purpose |
| 1. Connect scheduled trigger and webhook nodes to your emissions data sources. 2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account). 3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible). 4. Set approval thresholds in the Check Approval Required node. 5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. | Setup guidance |
| OpenAI or compatible LLM API key; Slack bot token; Gmail OAuth2 credentials; Google Sheets service account. | Prerequisites |
| Corporate sustainability teams automating monthly ESG reporting. | Use case |
| Swap LLM models per agent for cost or accuracy trade-offs. | Customisation |
| Eliminates manual emissions data aggregation and report generation. | Benefit |