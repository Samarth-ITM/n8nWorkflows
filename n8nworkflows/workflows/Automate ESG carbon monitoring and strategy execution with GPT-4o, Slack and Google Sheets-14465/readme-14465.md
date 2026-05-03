Automate ESG carbon monitoring and strategy execution with GPT-4o, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/automate-esg-carbon-monitoring-and-strategy-execution-with-gpt-4o--slack-and-google-sheets-14465


# Automate ESG carbon monitoring and strategy execution with GPT-4o, Slack and Google Sheets

# 1. Workflow Overview

This workflow implements an AI-orchestrated carbon sustainability operations pipeline in n8n. Its purpose is to collect emissions data on a schedule, analyze it with a supervisor LLM agent, delegate work to specialized carbon/ESG sub-agents, and then route the result into one of several action paths: monitoring storage, optimization approval/execution, policy alerting, or ESG reporting.

Target use cases include:

- ESG and sustainability monitoring across cloud or enterprise environments
- Carbon reduction strategy evaluation
- Policy compliance enforcement
- Automated stakeholder reporting through Slack, Google Sheets, email, and PostgreSQL

The workflow is organized into the following logical blocks.

## 1.1 Trigger and Input Intake

A scheduled trigger launches the workflow every 6 hours and passes incoming emissions-related payloads to the AI supervisor.

## 1.2 Supervisor Orchestration Layer

A central supervisor agent receives the emissions payload, uses GPT-4o, delegates tasks to four specialized sub-agents, and returns a structured decision payload.

## 1.3 Specialized Carbon/ESG Agent Tools

Four specialized AI agents handle distinct domains:

- Carbon monitoring
- Carbon optimization
- Policy enforcement
- ESG reporting

These agents use supporting tools such as PostgreSQL, Google Sheets, Slack human-in-the-loop, code-based calculations, and a calculator tool.

## 1.4 Action Routing

A Switch node routes the supervisor’s structured output into one of four downstream branches based on the `action` field:

- `monitor`
- `optimize`
- `enforce_policy`
- `generate_report`

## 1.5 Optimization Approval and Execution

Optimization recommendations are normalized, checked for approval requirements, then either:

- sent to Slack for human approval and logged as approved, or
- auto-executed and logged directly

## 1.6 Policy Alerting

Policy violations are formatted and sent to Slack to alert an operations or governance channel.

## 1.7 ESG Reporting Distribution

Generated ESG report data is appended to Google Sheets, then a stakeholder email summary is sent.

## 1.8 Consolidation, Notifications, and KPI Tracking

Outputs from all branches converge into a Merge node, which triggers a Slack completion notification and records execution status in PostgreSQL.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Input Intake

### Overview

This block starts the workflow on a recurring schedule. It acts as the single active entry point in the provided JSON and feeds emissions data into the orchestration layer.

### Nodes Involved

- Scheduled Carbon Data Collection

### Node Details

#### Scheduled Carbon Data Collection

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; periodic workflow trigger
- **Configuration choices:** Runs every 6 hours using an interval rule on hours
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none
  - Output: Carbon Supervisor Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - No external auth failure risk
  - Trigger may run with empty or unexpected payload context if upstream data fetching is not later added
  - Since there is no separate fetch node, the workflow depends on whatever JSON context is present at runtime
- **Sub-workflow reference:** None

---

## 2.2 Supervisor Orchestration Layer

### Overview

This is the core decision-making layer. It accepts incoming emissions data, uses GPT-4o with a supervisor prompt, can invoke specialized sub-agents as tools, and returns a structured action object for downstream routing.

### Nodes Involved

- Carbon Supervisor Agent
- Supervisor Model
- Supervisor Output Parser

### Node Details

#### Carbon Supervisor Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; primary LLM agent orchestrator
- **Configuration choices:**
  - Input text is taken from `emissions_data`, then `body`, then the whole item:
    - `={{ $json.emissions_data || $json.body || $json }}`
  - Uses a strong system prompt positioning it as a carbon sustainability supervisor
  - Explicitly instructed to choose among monitoring, optimization, policy enforcement, and ESG reporting
  - Structured output parsing is enabled
- **Key expressions or variables used:**
  - `{{$json.emissions_data || $json.body || $json}}`
- **Input and output connections:**
  - Main input: Scheduled Carbon Data Collection
  - AI language model input: Supervisor Model
  - AI tool inputs: Carbon Monitoring Agent, Carbon Optimization Agent, Policy Enforcement Agent, ESG Reporting Agent
  - AI output parser input: Supervisor Output Parser
  - Main output: Route by Action Type
- **Version-specific requirements:** Type version `3.1`
- **Edge cases or potential failure types:**
  - OpenAI model/tool invocation failures
  - Output may not conform to parser schema if the LLM ignores instructions
  - If incoming JSON shape differs from expectations, semantic routing may be poor
  - Tool delegation may fail if tool nodes or their credentials are invalid
- **Sub-workflow reference:** None

#### Supervisor Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat LLM backend for the supervisor
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for controlled but still flexible orchestration
  - No built-in tools configured at model level
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to Carbon Supervisor Agent as AI language model
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials
  - API quota/rate limits
  - Model availability changes
- **Sub-workflow reference:** None

#### Supervisor Output Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; schema-enforced structured parser
- **Configuration choices:**
  - Manual JSON schema with fields:
    - `action`
    - `priority`
    - `carbon_metrics`
    - `recommendations`
    - `policy_violations`
    - `requires_approval`
  - Allowed actions:
    - `monitor`
    - `optimize`
    - `enforce_policy`
    - `generate_report`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output parser attached to Carbon Supervisor Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - LLM returns malformed JSON
  - Arrays or object fields may be absent despite schema expectations
  - Downstream nodes assume nested fields exist, especially in optimization/policy/report paths
- **Sub-workflow reference:** None

---

## 2.3 Specialized Carbon/ESG Agent Tools

### Overview

These nodes implement the specialized reasoning and tool-use capabilities available to the supervisor. Each agent has its own model and domain-specific system message; some agents also connect to data tools or approval tools.

### Nodes Involved

- Carbon Monitoring Agent
- Monitoring Model
- Carbon Calculations Tool
- KPI Calculator
- Carbon Database Tool
- Carbon Optimization Agent
- Optimization Model
- Approval Workflow Tool
- Policy Enforcement Agent
- Policy Model
- Policy Sheets Tool
- ESG Reporting Agent
- ESG Model

### Node Details

#### Carbon Monitoring Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialized AI tool-agent for emissions monitoring
- **Configuration choices:**
  - Task text comes from AI-generated tool parameter:
    - `={{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
  - System message emphasizes data collection, validation, anomaly detection, KPI calculation, and use of tools
- **Key expressions or variables used:**
  - `$fromAI('monitoring_task', ...)`
- **Input and output connections:**
  - AI language model: Monitoring Model
  - AI tools: Carbon Calculations Tool, KPI Calculator, Carbon Database Tool
  - Output to Carbon Supervisor Agent as AI tool
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Tool-generated parameters may be missing or badly typed
  - Database tool may not be configured with a table
  - Monitoring tasks may ask for unavailable historical data
- **Sub-workflow reference:** None

#### Monitoring Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend for monitoring
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1` for stable analytical behavior
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to Carbon Monitoring Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** OpenAI auth, quota, timeout, model availability
- **Sub-workflow reference:** None

#### Carbon Calculations Tool

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`; custom JS tool for carbon computations
- **Configuration choices:**
  - Reads AI-supplied values:
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
- **Key expressions or variables used:**
  - `$fromAI('emissions_value', ..., 'number')`
  - `$fromAI('energy_kwh', ..., 'number', 0)`
  - `$fromAI('operation', ..., 'string')`
- **Input and output connections:**
  - Output to Carbon Monitoring Agent as AI tool
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:**
  - Division and efficiency math can behave oddly if `energy_kwh` is 0
  - `result.toFixed(2)` returns a string, not a numeric value
  - Unsupported operation falls back to raw emissions
  - AI may supply nonnumeric values
- **Sub-workflow reference:** None

#### KPI Calculator

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCalculator`; generic calculator tool
- **Configuration choices:** No explicit parameters configured
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Output to Carbon Monitoring Agent as AI tool
  - Output to Approval Workflow Tool as AI tool
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:**
  - LLM may use it inconsistently if prompts are underspecified
  - Unclear practical role for Approval Workflow Tool integration unless approval messages require calculations
- **Sub-workflow reference:** None

#### Carbon Database Tool

- **Type and technical role:** `n8n-nodes-base.postgresTool`; AI-accessible PostgreSQL tool
- **Configuration choices:**
  - Schema: `public`
  - Table value is blank in the JSON, meaning it still requires configuration
  - Tool description says it can query/update emissions data, KPI metrics, trends, and policy configs
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to Carbon Monitoring Agent and Carbon Optimization Agent as AI tool
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - Missing database credentials in JSON
  - Blank table selection can cause tool failure or limited functionality
  - Permissions may block updates
  - Query/tool calls may be slow on large tables
- **Sub-workflow reference:** None

#### Carbon Optimization Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; specialized AI optimization agent
- **Configuration choices:**
  - Task comes from:
    - `={{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
  - System prompt focuses on identifying high-carbon workloads, low-carbon alternatives, savings, and prioritization
- **Key expressions or variables used:**
  - `$fromAI('optimization_task', ...)`
- **Input and output connections:**
  - AI language model: Optimization Model
  - AI tools: Carbon Database Tool, Approval Workflow Tool
  - Output to Carbon Supervisor Agent as AI tool
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Potential savings may not map to downstream expected field names
  - Approval tool use within the agent may overlap conceptually with later explicit Slack approval branch
- **Sub-workflow reference:** None

#### Optimization Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to Carbon Optimization Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** OpenAI auth, rate limit, malformed tool planning
- **Sub-workflow reference:** None

#### Approval Workflow Tool

- **Type and technical role:** `n8n-nodes-base.slackHitlTool`; Slack human-in-the-loop tool
- **Configuration choices:**
  - User field is blank and must be configured
  - Message uses AI-generated content:
    - `={{ $fromAI('approval_message', 'The approval request message') }}`
  - Authentication: OAuth2
- **Key expressions or variables used:**
  - `$fromAI('approval_message', ...)`
- **Input and output connections:**
  - Receives AI tool support from KPI Calculator
  - Output to Carbon Optimization Agent as AI tool
- **Version-specific requirements:** Type version `2.4`
- **Edge cases or potential failure types:**
  - Missing Slack user target
  - Slack OAuth misconfiguration
  - HITL interaction timeout or no user response
  - Potential duplication with explicit `Request Strategy Approval` Slack node downstream
- **Sub-workflow reference:** None

#### Policy Enforcement Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; policy/compliance agent
- **Configuration choices:**
  - Task comes from:
    - `={{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
  - System prompt checks emissions data against green policies and targets
- **Key expressions or variables used:**
  - `$fromAI('policy_task', ...)`
- **Input and output connections:**
  - AI language model: Policy Model
  - AI tools: Policy Sheets Tool
  - Output to Carbon Supervisor Agent as AI tool
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Policy data may be missing because Sheet and document are blank
  - Inconsistent violation object structure can break Slack formatting later
- **Sub-workflow reference:** None

#### Policy Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to Policy Enforcement Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** Standard OpenAI issues
- **Sub-workflow reference:** None

#### Policy Sheets Tool

- **Type and technical role:** `n8n-nodes-base.googleSheetsTool`; AI-readable Google Sheets policy source
- **Configuration choices:**
  - Document ID blank
  - Sheet name blank
  - Tool description indicates carbon reduction policies, approved strategies, and sustainability targets
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to Policy Enforcement Agent as AI tool
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Missing document or worksheet config
  - Google OAuth permission errors
  - Incorrect sharing settings on the sheet
- **Sub-workflow reference:** None

#### ESG Reporting Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`; reporting-focused AI tool-agent
- **Configuration choices:**
  - Task comes from:
    - `={{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
  - System prompt focuses on stakeholder/regulatory reporting
- **Key expressions or variables used:**
  - `$fromAI('reporting_task', ...)`
- **Input and output connections:**
  - AI language model: ESG Model
  - Output to Carbon Supervisor Agent as AI tool
- **Version-specific requirements:** Type version `3`
- **Edge cases or potential failure types:**
  - Report content may not contain all fields expected by downstream email and Sheets nodes
- **Sub-workflow reference:** None

#### ESG Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to ESG Reporting Agent
- **Version-specific requirements:** Type version `1.3`
- **Edge cases or potential failure types:** Standard OpenAI issues
- **Sub-workflow reference:** None

---

## 2.4 Action Routing

### Overview

This block routes the supervisor’s structured result into one of four downstream business branches. The `action` field is the key contract between the AI orchestration layer and operational execution.

### Nodes Involved

- Route by Action Type

### Node Details

#### Route by Action Type

- **Type and technical role:** `n8n-nodes-base.switch`; conditional branch router
- **Configuration choices:**
  - Routes on `={{ $json.action }}`
  - Cases:
    - `monitor`
    - `optimize`
    - `enforce_policy`
    - `generate_report`
- **Key expressions or variables used:**
  - `{{$json.action}}`
- **Input and output connections:**
  - Input: Carbon Supervisor Agent
  - Outputs:
    - monitor → Store Carbon Metrics
    - optimize → Prepare Optimization Data
    - enforce_policy → Send Policy Alert
    - generate_report → Update ESG Report
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - If `action` is missing or outside enum, no branch will fire
  - Some downstream nodes expect `output.*` while parser schema places fields at top level, creating a likely mapping mismatch
- **Sub-workflow reference:** None

---

## 2.5 Monitoring Data Storage Branch

### Overview

This branch persists structured monitoring output into PostgreSQL. It is the simplest branch and acts as a historical storage path for carbon metrics and recommendations.

### Nodes Involved

- Store Carbon Metrics

### Node Details

#### Store Carbon Metrics

- **Type and technical role:** `n8n-nodes-base.postgres`; database insert/update node
- **Configuration choices:**
  - Writes to table `public.carbon_metrics`
  - Explicitly maps columns:
    - `action`
    - `priority`
    - `timestamp`
    - `carbon_metrics` as JSON string
    - `recommendations` as JSON string
    - `policy_violations` as JSON string
    - `requires_approval`
- **Key expressions or variables used:**
  - `={{ $json.action }}`
  - `={{ $json.priority }}`
  - `={{ $now }}`
  - `={{ JSON.stringify($json.carbon_metrics) }}`
  - `={{ JSON.stringify($json.recommendations) }}`
  - `={{ JSON.stringify($json.policy_violations) }}`
  - `={{ $json.requires_approval }}`
- **Input and output connections:**
  - Input: Route by Action Type (monitor branch)
  - Output: Consolidate All Actions
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - Missing Postgres credentials
  - Table schema must support mapped columns
  - JSON stringify can produce `"undefined"` issues if fields are absent
- **Sub-workflow reference:** None

---

## 2.6 Optimization Approval and Execution Branch

### Overview

This branch handles optimization outcomes. It reshapes the optimization payload, checks whether approval is required, then either sends a Slack approval request and logs approval, or auto-executes and logs it directly.

### Nodes Involved

- Prepare Optimization Data
- Check Approval Required
- Request Strategy Approval
- Log Approved Strategy
- Auto-Execute Strategy

### Node Details

#### Prepare Optimization Data

- **Type and technical role:** `n8n-nodes-base.set`; field reshaping/normalization node
- **Configuration choices:**
  - Creates:
    - `optimization_summary` from `{{$json.output.recommendations}}`
    - `carbon_savings` from `{{$json.output.carbon_metrics.potential_savings}}`
    - `priority` from `{{$json.output.priority}}`
    - `requires_approval` from `{{$json.output.requires_approval}}`
- **Key expressions or variables used:**
  - `{{$json.output.recommendations}}`
  - `{{$json.output.carbon_metrics.potential_savings}}`
  - `{{$json.output.priority}}`
  - `{{$json.output.requires_approval}}`
- **Input and output connections:**
  - Input: Route by Action Type (optimize branch)
  - Output: Check Approval Required
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or potential failure types:**
  - Major schema mismatch risk: supervisor output parser produces top-level fields, not `output.*`
  - `optimization_summary` is declared as string but likely receives an array
  - `potential_savings` is not guaranteed by the parser schema
- **Sub-workflow reference:** None

#### Check Approval Required

- **Type and technical role:** `n8n-nodes-base.if`; conditional branch node
- **Configuration choices:**
  - Tests whether `={{ $json.requires_approval }}` is true
- **Key expressions or variables used:**
  - `{{$json.requires_approval}}`
- **Input and output connections:**
  - Input: Prepare Optimization Data
  - True output: Request Strategy Approval
  - False output: Auto-Execute Strategy
- **Version-specific requirements:** Type version `2.3`
- **Edge cases or potential failure types:**
  - If `requires_approval` is null/undefined due to mapping issues, false branch will likely be taken
- **Sub-workflow reference:** None

#### Request Strategy Approval

- **Type and technical role:** `n8n-nodes-base.slack`; sends approval request and waits for response
- **Configuration choices:**
  - Operation: `sendAndWait`
  - Slack channel is placeholder-based and must be configured
  - Approval type is `double`
  - Reject button label is `Reject`
  - Message includes priority, carbon savings, and bullet list of recommendations
- **Key expressions or variables used:**
  - `{{$json.priority}}`
  - `{{$json.carbon_savings}}`
  - `{{$json.optimization_summary.map(r => `• ${r}`).join('\n')}}`
- **Input and output connections:**
  - Input: Check Approval Required (true branch)
  - Output: Log Approved Strategy
- **Version-specific requirements:** Type version `2.4`
- **Edge cases or potential failure types:**
  - If `optimization_summary` is not an array, `.map()` will fail
  - Send-and-wait may pause execution for long periods
  - Channel placeholder must be replaced
  - Approval response payload shape must include `user` for downstream logging
- **Sub-workflow reference:** None

#### Log Approved Strategy

- **Type and technical role:** `n8n-nodes-base.postgres`; approval audit logging
- **Configuration choices:**
  - Writes to `public.approved_strategies`
  - Stores:
    - `status = approved`
    - priority
    - serialized strategy
    - timestamp
    - `approved_by`
    - carbon_savings
- **Key expressions or variables used:**
  - `={{ $json.priority }}`
  - `={{ JSON.stringify($json.optimization_summary) }}`
  - `={{ $now }}`
  - `={{ $json.user }}`
  - `={{ $json.carbon_savings }}`
- **Input and output connections:**
  - Input: Request Strategy Approval
  - Output: Consolidate All Actions
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - `user` field may not exist depending on Slack response payload
  - Database schema must support inserted fields
- **Sub-workflow reference:** None

#### Auto-Execute Strategy

- **Type and technical role:** `n8n-nodes-base.postgres`; logs non-approved automatic execution
- **Configuration choices:**
  - Writes to `public.approved_strategies`
  - Stores:
    - `status = auto_executed`
    - priority
    - serialized strategy
    - timestamp
    - carbon_savings
- **Key expressions or variables used:**
  - `={{ $json.priority }}`
  - `={{ JSON.stringify($json.optimization_summary) }}`
  - `={{ $now }}`
  - `={{ $json.carbon_savings }}`
- **Input and output connections:**
  - Input: Check Approval Required (false branch)
  - Output: Consolidate All Actions
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - This node logs execution but does not perform actual external infrastructure changes
  - Naming may imply execution, but behavior is only database logging
- **Sub-workflow reference:** None

---

## 2.7 Policy Alerting Branch

### Overview

This branch formats policy violations and recommended actions into a Slack alert. It is used when the supervisor decides the incoming data requires policy enforcement handling.

### Nodes Involved

- Send Policy Alert

### Node Details

#### Send Policy Alert

- **Type and technical role:** `n8n-nodes-base.slack`; outbound Slack alert
- **Configuration choices:**
  - Sends to a configured channel placeholder
  - Message includes priority, violation count, bullet list of violations, and recommended actions
  - Assumes payload fields are nested under `output`
- **Key expressions or variables used:**
  - `{{$json.output.priority}}`
  - `{{$json.output.policy_violations.length}}`
  - `{{$json.output.policy_violations.map(v => `• ${v.policy}: ${v.description}`).join('\n')}}`
  - `{{$json.output.recommendations.map(r => `• ${r}`).join('\n')}}`
- **Input and output connections:**
  - Input: Route by Action Type (enforce_policy branch)
  - Output: Consolidate All Actions
- **Version-specific requirements:** Type version `2.4`
- **Edge cases or potential failure types:**
  - Strong schema mismatch risk because supervisor parser returns top-level fields, not `output.*`
  - `policy_violations` elements must contain `policy` and `description`
  - Empty arrays may create blank sections
  - Channel placeholder must be configured
- **Sub-workflow reference:** None

---

## 2.8 ESG Reporting Distribution Branch

### Overview

This branch writes report data to Google Sheets and then emails stakeholders a summary. It is the reporting endpoint for generated ESG insights.

### Nodes Involved

- Update ESG Report
- Send ESG Report Email

### Node Details

#### Update ESG Report

- **Type and technical role:** `n8n-nodes-base.googleSheets`; append report rows
- **Configuration choices:**
  - Operation: `append`
  - Document ID blank
  - Sheet name blank
- **Key expressions or variables used:** None directly in parameters shown
- **Input and output connections:**
  - Input: Route by Action Type (generate_report branch)
  - Output: Send ESG Report Email
- **Version-specific requirements:** Type version `4.7`
- **Edge cases or potential failure types:**
  - Blank document ID and sheet name must be filled
  - Append structure depends on incoming JSON keys and sheet columns
  - OAuth scope/sharing issues may block writes
- **Sub-workflow reference:** None

#### Send ESG Report Email

- **Type and technical role:** `n8n-nodes-base.emailSend`; email delivery node
- **Configuration choices:**
  - HTML email body includes total emissions, carbon intensity, compliance rate, and recommendations
  - Subject uses current date
  - Sender and recipient are placeholders
- **Key expressions or variables used:**
  - `{{$json.carbon_metrics.total_emissions}}`
  - `{{$json.carbon_metrics.carbon_intensity}}`
  - `{{$json.carbon_metrics.compliance_rate}}`
  - `{{$json.recommendations.map(r => `• ${r}`).join('<br>')}}`
  - `{{$now.toFormat('yyyy-MM-dd')}}`
- **Input and output connections:**
  - Input: Update ESG Report
  - Output: Consolidate All Actions
- **Version-specific requirements:** Type version `2.1`
- **Edge cases or potential failure types:**
  - No email credential object is shown in JSON; must be configured manually
  - Missing `carbon_metrics` fields will create broken email content
  - Placeholder emails must be replaced
- **Sub-workflow reference:** None

---

## 2.9 Consolidation, Final Notification, and KPI Tracking

### Overview

This block merges all possible branch completions, sends a summary Slack message, and records workflow completion to PostgreSQL.

### Nodes Involved

- Consolidate All Actions
- Send Summary Notification
- Update KPI Dashboard

### Node Details

#### Consolidate All Actions

- **Type and technical role:** `n8n-nodes-base.merge`; collects outputs from multiple branches
- **Configuration choices:**
  - Configured for 5 inputs
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs:
    - Store Carbon Metrics
    - Log Approved Strategy
    - Auto-Execute Strategy
    - Send Policy Alert
    - Send ESG Report Email
  - Output: Send Summary Notification
- **Version-specific requirements:** Type version `3.2`
- **Edge cases or potential failure types:**
  - Merge behavior depends on execution timing and active branch count
  - With mutually exclusive routing upstream, only one branch typically produces output per run
  - If merge mode expects all inputs, some executions could stall or behave unexpectedly depending on default merge behavior in this version
- **Sub-workflow reference:** None

#### Send Summary Notification

- **Type and technical role:** `n8n-nodes-base.slack`; completion notification
- **Configuration choices:**
  - Sends a success summary to a Slack channel placeholder
  - Displays action and priority if present, otherwise fallback text
- **Key expressions or variables used:**
  - `{{$json.action || 'Multiple actions'}}`
  - `{{$json.priority || 'N/A'}}`
- **Input and output connections:**
  - Input: Consolidate All Actions
  - Output: Update KPI Dashboard
- **Version-specific requirements:** Type version `2.4`
- **Edge cases or potential failure types:**
  - Depending on upstream branch output, `action` and `priority` may not still exist
  - Channel placeholder must be configured
- **Sub-workflow reference:** None

#### Update KPI Dashboard

- **Type and technical role:** `n8n-nodes-base.postgres`; final KPI logging
- **Configuration choices:**
  - Writes to `public.kpi_dashboard`
  - Stores:
    - `status = completed`
    - timestamp
    - `total_actions = {{$json.action ? 1 : 0}}`
    - workflow execution ID
- **Key expressions or variables used:**
  - `=completed`
  - `={{ $now }}`
  - `={{ $json.action ? 1 : 0 }}`
  - `={{ $execution.id }}`
- **Input and output connections:**
  - Input: Send Summary Notification
  - Output: none
- **Version-specific requirements:** Type version `2.6`
- **Edge cases or potential failure types:**
  - `action` may not survive branch transformations, making `total_actions` misleading
  - DB schema must exist and match inserted columns
- **Sub-workflow reference:** None

---

## 2.10 Documentation Sticky Notes

### Overview

These nodes are documentation-only notes embedded in the canvas. They do not execute but provide architecture, setup, prerequisites, and business-context guidance.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note

- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation
- **Configuration choices:** Describes end-to-end workflow behavior and stakeholder value
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Setup guidance for triggers, credentials, model selection, approval thresholds, and Sheets IDs
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Prerequisites, use case, customization, and benefits
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents strategy and policy evaluation section
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents supervisor orchestration section
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents approval routing section
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

#### Sticky Note6

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents reporting and notification section
- **Input and output connections:** none
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Scheduled Carbon Data Collection | scheduleTrigger | Starts workflow every 6 hours |  | Carbon Supervisor Agent | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Carbon Supervisor Agent | langchain.agent | Central AI orchestrator deciding action and delegating to sub-agents | Scheduled Carbon Data Collection | Route by Action Type | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Supervisor Model | lmChatOpenAi | GPT-4o model used by supervisor |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Supervisor Output Parser | outputParserStructured | Enforces structured supervisor output schema |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Monitoring Agent | langchain.agentTool | Monitoring specialist agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Monitoring Model | lmChatOpenAi | GPT-4o model for monitoring agent |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Optimization Agent | langchain.agentTool | Optimization specialist agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Optimization Model | lmChatOpenAi | GPT-4o model for optimization agent |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Enforcement Agent | langchain.agentTool | Compliance and policy specialist agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Model | lmChatOpenAi | GPT-4o model for policy agent |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Reporting Agent | langchain.agentTool | Reporting specialist agent |  | Carbon Supervisor Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| ESG Model | lmChatOpenAi | GPT-4o model for ESG reporting agent |  | ESG Reporting Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Calculations Tool | langchain.toolCode | Custom carbon formula tool |  | Carbon Monitoring Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| KPI Calculator | toolCalculator | Generic math tool for monitoring and approval processes |  | Carbon Monitoring Agent, Approval Workflow Tool | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Carbon Database Tool | postgresTool | AI-accessible PostgreSQL data tool |  | Carbon Monitoring Agent, Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Policy Sheets Tool | googleSheetsTool | AI-readable policy source from Google Sheets |  | Policy Enforcement Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Approval Workflow Tool | slackHitlTool | AI-driven Slack human-in-the-loop approval helper |  | Carbon Optimization Agent | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Route by Action Type | switch | Routes supervisor result to action-specific branch | Carbon Supervisor Agent | Store Carbon Metrics, Prepare Optimization Data, Send Policy Alert, Update ESG Report | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Store Carbon Metrics | postgres | Stores monitoring results in PostgreSQL | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Prepare Optimization Data | set | Normalizes optimization payload for approval logic | Route by Action Type | Check Approval Required | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Check Approval Required | if | Decides whether optimization requires human sign-off | Prepare Optimization Data | Request Strategy Approval, Auto-Execute Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Request Strategy Approval | slack | Sends Slack approval request and waits | Check Approval Required | Log Approved Strategy | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Log Approved Strategy | postgres | Logs approved optimization strategy | Request Strategy Approval | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Auto-Execute Strategy | postgres | Logs auto-executed low-risk strategy | Check Approval Required | Consolidate All Actions | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Send Policy Alert | slack | Sends Slack alert for policy violations | Route by Action Type | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update ESG Report | googleSheets | Appends ESG report output to Google Sheets | Route by Action Type | Send ESG Report Email | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send ESG Report Email | emailSend | Emails ESG summary to stakeholders | Update ESG Report | Consolidate All Actions | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Consolidate All Actions | merge | Collects branch outputs before completion notification | Store Carbon Metrics, Log Approved Strategy, Auto-Execute Strategy, Send Policy Alert, Send ESG Report Email | Send Summary Notification | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Send Summary Notification | slack | Sends final workflow completion message | Consolidate All Actions | Update KPI Dashboard | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Update KPI Dashboard | postgres | Logs final execution metrics | Send Summary Notification |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |
| Sticky Note | stickyNote | Canvas documentation |  |  | ## How It Works<br>This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. |
| Sticky Note1 | stickyNote | Canvas setup guidance |  |  | ## Setup Steps<br>1. Connect scheduled trigger and webhook nodes to your emissions data sources.<br>2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account).<br>3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible).<br>4. Set approval thresholds in the Check Approval Required node.<br>5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. |
| Sticky Note2 | stickyNote | Canvas prerequisites and benefits |  |  | ## Prerequisites<br>- OpenAI or compatible LLM API key<br>- Slack bot token<br>- Gmail OAuth2 credentials<br>- Google Sheets service account<br>## Use Cases<br>- Corporate sustainability teams automating monthly ESG reporting<br>## Customisation<br>- Swap LLM models per agent for cost or accuracy trade-offs<br>## Benefits<br>- Eliminates manual emissions data aggregation and report generation |
| Sticky Note3 | stickyNote | Canvas section note for strategy/policy area |  |  | ## Strategy & Policy Evaluation<br>**What:** Optimisation and Policy Enforcement Agents propose and validate actions.<br>**Why:** Keeps reduction strategies compliant before execution. |
| Sticky Note4 | stickyNote | Canvas section note for supervisor layer |  |  | ## Supervisor Orchestration<br>**What:** Carbon Supervisor Agent delegates tasks across four specialised sub-agents.<br>**Why:** Parallel processing improves speed and separates concerns cleanly. |
| Sticky Note5 | stickyNote | Canvas section note for approval routing |  |  | ## Approval Routing<br>**What:** Routes strategies requiring sign-off to Slack; auto-executes low-risk ones.<br>**Why:** Balances automation with governance oversight. |
| Sticky Note6 | stickyNote | Canvas section note for reporting area |  |  | ## Reporting & Notification<br>**What:** Updates ESG report, sends email summary, refreshes KPI dashboard.<br>**Why:** Closes the feedback loop for stakeholders in real time. |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence in n8n.

1. **Create a new workflow**
   - Name it something like: `AI carbon supervisor agent for ESG monitoring and strategy execution`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `Scheduled Carbon Data Collection`
   - Set interval to every `6 hours`

3. **Add the supervisor model**
   - Node type: `OpenAI Chat Model` / `lmChatOpenAi`
   - Name: `Supervisor Model`
   - Credential: OpenAI API credential
   - Model: `gpt-4o`
   - Temperature: `0.2`

4. **Add the supervisor output parser**
   - Node type: `Structured Output Parser`
   - Name: `Supervisor Output Parser`
   - Use manual schema with fields:
     - `action` enum: `monitor`, `optimize`, `enforce_policy`, `generate_report`
     - `priority` enum: `critical`, `high`, `medium`, `low`
     - `carbon_metrics` object
     - `recommendations` array
     - `policy_violations` array
     - `requires_approval` boolean

5. **Add the central AI Agent**
   - Node type: `AI Agent`
   - Name: `Carbon Supervisor Agent`
   - Input text:
     - `={{ $json.emissions_data || $json.body || $json }}`
   - Enable structured output parsing
   - System message should describe the node as the carbon sustainability supervisor that orchestrates monitoring, optimization, policy enforcement, and ESG reporting
   - Connect:
     - `Scheduled Carbon Data Collection` → `Carbon Supervisor Agent`
     - `Supervisor Model` → `Carbon Supervisor Agent` as AI language model
     - `Supervisor Output Parser` → `Carbon Supervisor Agent` as AI output parser

6. **Add the monitoring model**
   - Node type: `OpenAI Chat Model`
   - Name: `Monitoring Model`
   - Credential: OpenAI
   - Model: `gpt-4o`
   - Temperature: `0.1`

7. **Add the carbon calculations code tool**
   - Node type: `Tool Code`
   - Name: `Carbon Calculations Tool`
   - Paste JS logic implementing:
     - `emissions_value`
     - `energy_kwh`
     - `operation`
     - formulas for `footprint`, `intensity`, `efficiency`, `savings`
   - Return fields like `result`, `unit`, `operation`

8. **Add the calculator tool**
   - Node type: `Calculator Tool`
   - Name: `KPI Calculator`
   - Keep default configuration

9. **Add the PostgreSQL AI tool**
   - Node type: `Postgres Tool`
   - Name: `Carbon Database Tool`
   - Configure PostgreSQL credentials
   - Schema: `public`
   - Select the emissions/KPI table you want the AI to use
   - Add a clear tool description explaining that it can query/update carbon emissions, KPIs, trends, and policies

10. **Add the monitoring agent**
    - Node type: `AI Agent Tool`
    - Name: `Carbon Monitoring Agent`
    - Text:
      - `={{ $fromAI('monitoring_task', 'The carbon monitoring task to perform') }}`
    - System message: emphasize collection, validation, anomaly detection, KPI tracking
    - Connect:
      - `Monitoring Model` → `Carbon Monitoring Agent` as AI language model
      - `Carbon Calculations Tool` → `Carbon Monitoring Agent` as AI tool
      - `KPI Calculator` → `Carbon Monitoring Agent` as AI tool
      - `Carbon Database Tool` → `Carbon Monitoring Agent` as AI tool

11. **Add the optimization model**
    - Node type: `OpenAI Chat Model`
    - Name: `Optimization Model`
    - Credential: OpenAI
    - Model: `gpt-4o`
    - Temperature: `0.3`

12. **Add the Slack HITL tool**
    - Node type: `Slack Human-in-the-Loop Tool`
    - Name: `Approval Workflow Tool`
    - Configure Slack OAuth2 credential
    - Select a Slack user or workspace target
    - Message:
      - `={{ $fromAI('approval_message', 'The approval request message') }}`
    - Authentication: OAuth2

13. **Connect calculator to the Slack HITL tool**
    - Connect `KPI Calculator` to `Approval Workflow Tool` as AI tool support

14. **Add the optimization agent**
    - Node type: `AI Agent Tool`
    - Name: `Carbon Optimization Agent`
    - Text:
      - `={{ $fromAI('optimization_task', 'The carbon optimization task to perform') }}`
    - System message: recommend low-carbon alternatives, prioritize by impact and efficiency
    - Connect:
      - `Optimization Model` → `Carbon Optimization Agent`
      - `Carbon Database Tool` → `Carbon Optimization Agent` as AI tool
      - `Approval Workflow Tool` → `Carbon Optimization Agent` as AI tool

15. **Add the policy model**
    - Node type: `OpenAI Chat Model`
    - Name: `Policy Model`
    - Credential: OpenAI
    - Model: `gpt-4o`
    - Temperature: `0.1`

16. **Add the Google Sheets AI tool for policies**
    - Node type: `Google Sheets Tool`
    - Name: `Policy Sheets Tool`
    - Configure Google Sheets OAuth2 credential
    - Select the policy document ID
    - Select the sheet containing sustainability policies
    - Add description saying it stores carbon reduction policies, approved strategies, and targets

17. **Add the policy agent**
    - Node type: `AI Agent Tool`
    - Name: `Policy Enforcement Agent`
    - Text:
      - `={{ $fromAI('policy_task', 'The policy enforcement task to perform') }}`
    - System message: check compliance, detect violations, recommend corrective action
    - Connect:
      - `Policy Model` → `Policy Enforcement Agent`
      - `Policy Sheets Tool` → `Policy Enforcement Agent` as AI tool

18. **Add the ESG reporting model**
    - Node type: `OpenAI Chat Model`
    - Name: `ESG Model`
    - Credential: OpenAI
    - Model: `gpt-4o`
    - Temperature: `0.2`

19. **Add the ESG reporting agent**
    - Node type: `AI Agent Tool`
    - Name: `ESG Reporting Agent`
    - Text:
      - `={{ $fromAI('reporting_task', 'The ESG reporting task to perform') }}`
    - System message: generate ESG reports, KPIs, compliance summaries, trend analysis
    - Connect:
      - `ESG Model` → `ESG Reporting Agent`

20. **Attach all four specialist agents to the supervisor**
    - Connect as AI tools:
      - `Carbon Monitoring Agent` → `Carbon Supervisor Agent`
      - `Carbon Optimization Agent` → `Carbon Supervisor Agent`
      - `Policy Enforcement Agent` → `Carbon Supervisor Agent`
      - `ESG Reporting Agent` → `Carbon Supervisor Agent`

21. **Add a Switch node**
    - Node type: `Switch`
    - Name: `Route by Action Type`
    - Expression: `={{ $json.action }}`
    - Create four cases:
      - `monitor`
      - `optimize`
      - `enforce_policy`
      - `generate_report`
    - Connect `Carbon Supervisor Agent` → `Route by Action Type`

22. **Create the monitoring storage branch**
    - Add node type: `Postgres`
    - Name: `Store Carbon Metrics`
    - Credential: PostgreSQL
    - Table: `public.carbon_metrics`
    - Map:
      - action = `{{$json.action}}`
      - priority = `{{$json.priority}}`
      - timestamp = `{{$now}}`
      - carbon_metrics = `{{ JSON.stringify($json.carbon_metrics) }}`
      - recommendations = `{{ JSON.stringify($json.recommendations) }}`
      - policy_violations = `{{ JSON.stringify($json.policy_violations) }}`
      - requires_approval = `{{$json.requires_approval}}`
    - Connect monitor output of `Route by Action Type` to `Store Carbon Metrics`

23. **Create the optimization preparation node**
    - Add node type: `Set`
    - Name: `Prepare Optimization Data`
    - Add fields:
      - `optimization_summary`
      - `carbon_savings`
      - `priority`
      - `requires_approval`
    - Important: to make this workflow robust, prefer using top-level fields:
      - `recommendations`
      - `carbon_metrics.potential_savings`
      - `priority`
      - `requires_approval`
    - If reproducing exactly from JSON, use `output.*` paths, but that may fail unless your supervisor returns nested output

24. **Add the approval check**
    - Node type: `If`
    - Name: `Check Approval Required`
    - Condition:
      - boolean true on `={{ $json.requires_approval }}`
    - Connect `Prepare Optimization Data` → `Check Approval Required`

25. **Add the Slack approval request**
    - Node type: `Slack`
    - Name: `Request Strategy Approval`
    - Configure Slack OAuth2
    - Operation: `sendAndWait`
    - Choose approval channel
    - Approval type: `double`
    - Reject label: `Reject`
    - Message should include:
      - priority
      - estimated carbon savings
      - optimization recommendations
    - Connect true branch of `Check Approval Required` → `Request Strategy Approval`

26. **Add approval logging**
    - Node type: `Postgres`
    - Name: `Log Approved Strategy`
    - Table: `public.approved_strategies`
    - Map:
      - status = `approved`
      - priority = `{{$json.priority}}`
      - strategy = `{{ JSON.stringify($json.optimization_summary) }}`
      - timestamp = `{{$now}}`
      - approved_by = `{{$json.user}}`
      - carbon_savings = `{{$json.carbon_savings}}`
    - Connect `Request Strategy Approval` → `Log Approved Strategy`

27. **Add auto-execution logging**
    - Node type: `Postgres`
    - Name: `Auto-Execute Strategy`
    - Table: `public.approved_strategies`
    - Map:
      - status = `auto_executed`
      - priority = `{{$json.priority}}`
      - strategy = `{{ JSON.stringify($json.optimization_summary) }}`
      - timestamp = `{{$now}}`
      - carbon_savings = `{{$json.carbon_savings}}`
    - Connect false branch of `Check Approval Required` → `Auto-Execute Strategy`

28. **Create the policy alert branch**
    - Add node type: `Slack`
    - Name: `Send Policy Alert`
    - Configure Slack OAuth2
    - Choose policy alerts channel
    - Message should include:
      - priority
      - number of violations
      - bullet list of violations
      - bullet list of recommendations
    - If you want a stable build, use top-level fields instead of `output.*`
    - Connect `enforce_policy` branch of `Route by Action Type` → `Send Policy Alert`

29. **Create the ESG reporting branch**
    - Add node type: `Google Sheets`
    - Name: `Update ESG Report`
    - Configure Google Sheets OAuth2
    - Operation: `append`
    - Select document ID and target sheet
    - Connect `generate_report` branch of `Route by Action Type` → `Update ESG Report`

30. **Add the email summary**
    - Node type: `Send Email`
    - Name: `Send ESG Report Email`
    - Configure email credentials
    - Set sender email
    - Set stakeholder recipient email
    - Subject:
      - `ESG Sustainability Report - {{$now.toFormat('yyyy-MM-dd')}}`
    - HTML body should include:
      - total emissions
      - carbon intensity
      - compliance rate
      - recommendation list
    - Connect `Update ESG Report` → `Send ESG Report Email`

31. **Add a Merge node**
    - Node type: `Merge`
    - Name: `Consolidate All Actions`
    - Set number of inputs to `5`
    - Connect inputs from:
      - `Store Carbon Metrics`
      - `Log Approved Strategy`
      - `Auto-Execute Strategy`
      - `Send Policy Alert`
      - `Send ESG Report Email`

32. **Add the final Slack notification**
    - Node type: `Slack`
    - Name: `Send Summary Notification`
    - Configure Slack OAuth2
    - Choose summary channel
    - Message:
      - action
      - priority
      - success status
    - Connect `Consolidate All Actions` → `Send Summary Notification`

33. **Add final KPI logging**
    - Node type: `Postgres`
    - Name: `Update KPI Dashboard`
    - Table: `public.kpi_dashboard`
    - Map:
      - status = `completed`
      - timestamp = `{{$now}}`
      - total_actions = `{{$json.action ? 1 : 0}}`
      - workflow_run_id = `{{$execution.id}}`
    - Connect `Send Summary Notification` → `Update KPI Dashboard`

34. **Configure credentials**
    - OpenAI credential for all model nodes
    - Slack OAuth2 for:
      - Approval Workflow Tool
      - Request Strategy Approval
      - Send Policy Alert
      - Send Summary Notification
    - Google Sheets OAuth2 for:
      - Policy Sheets Tool
      - Update ESG Report
    - PostgreSQL credentials for:
      - Carbon Database Tool
      - Store Carbon Metrics
      - Log Approved Strategy
      - Auto-Execute Strategy
      - Update KPI Dashboard
    - Email credential for `Send ESG Report Email`

35. **Create required database tables**
    - `carbon_metrics`
    - `approved_strategies`
    - `kpi_dashboard`
   Ensure column types can accept strings, booleans, timestamps, and JSON/text payloads as mapped.

36. **Replace all placeholders**
    - Slack channel IDs:
      - policy alerts
      - approvals
      - summary
    - Email addresses:
      - sender
      - stakeholder recipient
    - Google Sheets document and worksheet IDs
    - PostgreSQL table/tool configuration values
    - Slack HITL user selection

37. **Recommended hardening before production**
    - Normalize all downstream expressions to use top-level supervisor output instead of `output.*`, unless you explicitly wrap the agent response into an `output` object
    - Add default/fallback values for arrays before using `.map()`
    - Add error branches or `Error Trigger` notifications
    - Verify Merge behavior with single-branch execution

38. **Test each action path separately**
    - Provide synthetic inputs that drive:
      - `monitor`
      - `optimize`
      - `enforce_policy`
      - `generate_report`
    - Confirm payload shapes match downstream expressions

### Required sub-workflow setup

This workflow does **not** invoke any separate n8n sub-workflows.  
All “sub-agents” are AI tool agents inside the same workflow, not `Execute Workflow` nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates end-to-end carbon emissions monitoring, strategy optimisation, and ESG reporting using a multi-agent AI supervisor architecture in n8n. Designed for sustainability managers, ESG teams, and operations leads, it eliminates the manual effort of tracking emissions, evaluating reduction strategies, and producing compliance reports. Data enters via scheduled pulls and real-time webhooks, then merges into a unified feed processed by a Carbon Supervisor Agent. Sub-agents handle monitoring, optimisation, policy enforcement, and ESG reporting. Approved strategies are auto-executed or routed for human sign-off. Outputs are consolidated and pushed to Slack, Google Sheets, and email, keeping all stakeholders informed. The workflow closes the loop from raw sensor data to actionable ESG dashboards with minimal human intervention. | High-level embedded workflow note |
| 1. Connect scheduled trigger and webhook nodes to your emissions data sources. 2. Add credentials for Slack (bot token), Gmail (OAuth2), and Google Sheets (service account). 3. Configure the Carbon Supervisor Agent with your preferred LLM (OpenAI or compatible). 4. Set approval thresholds in the Check Approval Required node. 5. Map Google Sheets document ID for ESG report and KPI dashboard nodes. | Embedded setup note |
| OpenAI or compatible LLM API key; Slack bot token; Gmail OAuth2 credentials; Google Sheets service account. | Embedded prerequisites |
| Corporate sustainability teams automating monthly ESG reporting. | Embedded use case |
| Swap LLM models per agent for cost or accuracy trade-offs. | Embedded customization note |
| Eliminates manual emissions data aggregation and report generation. | Embedded benefit note |

## Additional implementation observations

- The workflow currently has only one actual trigger: the schedule node. The sticky note mentions real-time webhooks, but no webhook node exists in the provided JSON.
- Several operational nodes reference `output.*` fields, while the structured parser defines top-level fields. This is the main implementation inconsistency to resolve before production use.
- Some required external configuration is incomplete in the JSON:
  - Google Sheets document IDs and sheet names are blank
  - Postgres AI tool table is blank
  - Slack user/channel placeholders remain
  - Email credential details are not provided
- The “Auto-Execute Strategy” node only logs execution status in PostgreSQL. It does not actually modify infrastructure or apply cloud optimization changes.