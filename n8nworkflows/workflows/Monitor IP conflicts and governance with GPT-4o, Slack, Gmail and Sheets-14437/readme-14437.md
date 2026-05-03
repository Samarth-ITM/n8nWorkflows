Monitor IP conflicts and governance with GPT-4o, Slack, Gmail and Sheets

https://n8nworkflows.xyz/workflows/monitor-ip-conflicts-and-governance-with-gpt-4o--slack--gmail-and-sheets-14437


# Monitor IP conflicts and governance with GPT-4o, Slack, Gmail and Sheets

# 1. Workflow Overview

This workflow automates intellectual property monitoring and governance. It combines scheduled checks and real-time external alerts, uses GPT-4o-powered AI agents to detect IP conflicts and lifecycle events, routes findings into governance review, stores records for auditability, exports analytics to Google Sheets, and sends notifications through Slack and Gmail based on severity.

Typical use cases include:
- Daily portfolio monitoring for patents, trademarks, and copyrights
- Real-time intake of external IP alerts
- Conflict detection against an internal or external IP knowledge base
- Renewal/lifecycle tracking
- Licensing compliance and governance review
- Compliance reporting and stakeholder notifications

## 1.1 Input Reception
The workflow has two entry points:
- A **Schedule Trigger** for daily monitoring
- A **Webhook** for external IP alerts

These two inputs converge into the monitoring stage.

## 1.2 AI-Based IP Monitoring
An **IP Monitoring Agent** orchestrates:
- A conflict detection specialist agent
- A lifecycle tracking specialist agent
- A Google Sheets tool for IP tracking data
- Shared memory
- GPT-4o as the main reasoning model
- A structured output parser

This block produces structured monitoring results such as conflict severity, conflicting assets, lifecycle alerts, and recommendations.

## 1.3 AI-Based Governance Processing
The monitoring output is passed to an **IP Governance Agent**, which uses:
- A licensing compliance specialist agent
- A documentation generation specialist agent
- Shared memory
- GPT-4o
- A structured output parser

This block determines approval status, compliance results, documentation output, stakeholder notifications, and required actions.

## 1.4 Conflict Routing and Notification
The workflow routes monitoring results by conflict severity:
- Critical and high/medium severities are routed differently
- Conflict records are normalized and stored
- Slack and Gmail are used for operational alerts

## 1.5 Governance Logging and Compliance Reporting
Governance results are normalized, stored in an n8n Data Table, and distributed via Gmail as compliance reports.

## 1.6 Analytics Export
A compact analytics summary is generated from AI outputs and appended to Google Sheets for portfolio-level reporting.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block collects incoming monitoring requests from two sources: a daily schedule and a real-time webhook. The webhook payload is normalized before entering the AI monitoring stage, while the scheduled trigger goes directly into monitoring.

### Nodes Involved
- Schedule IP Monitoring
- External IP Alert Webhook
- Parse Webhook Data

### Node Details

#### Schedule IP Monitoring
- **Type and role:** `n8n-nodes-base.scheduleTrigger` — time-based workflow entry point.
- **Configuration:** Runs on an interval rule configured to trigger at hour 9.
- **Key expressions/variables:** None.
- **Connections:** Outputs directly to **IP Monitoring Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:**
  - Timezone assumptions may differ from business expectations
  - If workflow is inactive, no scheduled execution occurs

#### External IP Alert Webhook
- **Type and role:** `n8n-nodes-base.webhook` — HTTP POST entry point for real-time external alerts.
- **Configuration:**  
  - Path: `ip-alert-webhook`
  - Method: `POST`
  - Response mode: `lastNode`
- **Key expressions/variables:** Consumes request body fields later in the workflow.
- **Connections:** Outputs to **Parse Webhook Data**.
- **Version-specific notes:** Type version `2.1`.
- **Edge cases/failures:**
  - Invalid or unexpected request body shape
  - Missing `body.alert_message`, `body.message`, or `body.alert_type`
  - Webhook URL changes between test/production modes in n8n

#### Parse Webhook Data
- **Type and role:** `n8n-nodes-base.set` — normalizes incoming webhook payload into the format expected by the monitoring agent.
- **Configuration choices:**
  - Sets `monitoring_request` from:
    - `$json.body.alert_message`
    - fallback `$json.body.message`
    - fallback `'External IP alert received'`
  - Sets `source` to `webhook`
  - Sets `alert_type` from `$json.body.alert_type` or fallback `external`
- **Key expressions/variables:**
  - `={{ $json.body.alert_message || $json.body.message || 'External IP alert received' }}`
  - `={{ $json.body.alert_type || 'external' }}`
- **Connections:** Input from **External IP Alert Webhook**; output to **IP Monitoring Agent**.
- **Version-specific notes:** Type version `3.4`.
- **Edge cases/failures:**
  - Expression failure if `body` is absent and n8n does not coerce as expected
  - Incomplete upstream payload may reduce AI accuracy

---

## 2.2 AI-Based IP Monitoring

### Overview
This block performs the main IP monitoring analysis. The central agent uses GPT-4o plus specialist tools to detect conflicts, assess lifecycle deadlines, consult tracking data, and return structured results.

### Nodes Involved
- IP Monitoring Agent
- Monitoring Agent Model
- Monitoring Memory
- IP Conflict Detection Agent
- Conflict Detection Model
- IP Knowledge Base
- IP Embeddings
- IP Lifecycle Tracking Agent
- Lifecycle Tracking Model
- IP Tracking Sheet Tool
- Monitoring Output Parser

### Node Details

#### IP Monitoring Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent` — main orchestration agent for portfolio monitoring.
- **Configuration choices:**
  - Prompt text:
    - Uses `$json.monitoring_request`
    - Falls back to a default daily monitoring instruction
  - System message defines responsibilities:
    - lifecycle monitoring
    - conflict detection
    - coordination with specialist sub-agents
    - structured reporting
  - Prompt type: `define`
  - Structured output enabled
- **Key expressions/variables:**
  - `={{ $json.monitoring_request || 'Perform daily IP monitoring: check all active patents, trademarks, and copyrights for conflicts, upcoming renewals, and lifecycle status changes.' }}`
- **Connections:**
  - Main input from **Schedule IP Monitoring** and **Parse Webhook Data**
  - AI language model from **Monitoring Agent Model**
  - AI memory from **Monitoring Memory**
  - AI tools from:
    - **IP Conflict Detection Agent**
    - **IP Lifecycle Tracking Agent**
    - **IP Tracking Sheet Tool**
  - AI output parser from **Monitoring Output Parser**
  - Main output to **IP Governance Agent**
- **Version-specific notes:** Type version `3.1`.
- **Edge cases/failures:**
  - LLM may produce output not compliant with parser schema
  - Tool calls may fail if specialist agents or Sheets credentials fail
  - If input context is too vague, output may be generic
- **Sub-workflow reference:** None; this is a parent agent using tool nodes, not a sub-workflow node.

#### Monitoring Agent Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT-4o chat model for the monitoring agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively stable outputs
- **Key expressions/variables:** None.
- **Connections:** AI language model input to **IP Monitoring Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:**
  - OpenAI credential/auth failure
  - Rate limits or token limits
  - Model availability issues

#### Monitoring Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term contextual memory for monitoring conversations/tool use.
- **Configuration choices:** Defaults.
- **Key expressions/variables:** None.
- **Connections:** AI memory input to **IP Monitoring Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:**
  - Minimal direct failure risk
  - Memory window behavior depends on node defaults and execution context

#### IP Conflict Detection Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist tool agent for IP conflict analysis.
- **Configuration choices:**
  - Input text comes from AI-collected variable `ip_asset_details`
  - System message instructs the agent to:
    - compare assets against IP databases
    - use vector similarity and keyword analysis
    - classify severity: critical/high/medium/low
  - Tool description explains expected result content
- **Key expressions/variables:**
  - `={{ $fromAI('ip_asset_details', 'The IP asset to analyze for conflicts including type, description, claims, and identifiers') }}`
- **Connections:**
  - AI language model from **Conflict Detection Model**
  - AI tool from **IP Knowledge Base**
  - AI tool output to **IP Monitoring Agent**
- **Version-specific notes:** Type version `3`.
- **Edge cases/failures:**
  - If the parent agent does not provide enough asset detail, analysis quality degrades
  - Similarity search may be ineffective if the vector store is empty
  - LLM output may overstate legal certainty

#### Conflict Detection Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT-4o model for the conflict specialist.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Connections:** AI language model input to **IP Conflict Detection Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:** Same OpenAI-related issues as above.

#### IP Knowledge Base
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStoreInMemory` — vector retrieval tool for conflict analysis.
- **Configuration choices:**
  - Mode: `retrieve-as-tool`
  - Memory key: `vector_store_key`
  - Tool description says it searches patents, trademarks, copyrights, and legal documentation
- **Key expressions/variables:** Memory key selection is configured statically.
- **Connections:**
  - AI embedding input from **IP Embeddings**
  - AI tool output to **IP Conflict Detection Agent**
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:**
  - This node is only useful if the in-memory vector store is actually populated elsewhere; in this workflow JSON, no population step is present
  - Empty vector store means weak or null retrieval
- **Sub-workflow reference:** None.

#### IP Embeddings
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi` — embeddings provider for the vector store.
- **Configuration choices:** Default options.
- **Connections:** AI embedding output to **IP Knowledge Base**.
- **Version-specific notes:** Type version `1.2`.
- **Edge cases/failures:**
  - OpenAI auth/rate-limit failures
  - No practical benefit if vector store is never loaded with documents

#### IP Lifecycle Tracking Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist tool for lifecycle and renewal tracking.
- **Configuration choices:**
  - Input text uses AI-collected variable `ip_asset_info`
  - System message covers:
    - filing/publication/grant dates
    - renewals and maintenance fees
    - expiration calculations
    - priority windows: within 30 days, 30–90 days, 90+ days
- **Key expressions/variables:**
  - `={{ $fromAI('ip_asset_info', 'The IP asset to track including type, filing date, grant date, and jurisdiction') }}`
- **Connections:**
  - AI language model from **Lifecycle Tracking Model**
  - AI tool output to **IP Monitoring Agent**
- **Version-specific notes:** Type version `3`.
- **Edge cases/failures:**
  - Missing filing/grant/jurisdiction data reduces usefulness
  - Date interpretation can be wrong if input formatting is inconsistent
  - Jurisdiction-specific legal deadlines may require external authoritative data not present here

#### Lifecycle Tracking Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT-4o model for lifecycle tracking.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Connections:** AI language model input to **IP Lifecycle Tracking Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:** Standard OpenAI credential/model issues.

#### IP Tracking Sheet Tool
- **Type and role:** `n8n-nodes-base.googleSheetsTool` — Google Sheets tool exposed to the monitoring agent.
- **Configuration choices:**
  - Reads IP asset tracking data from a target spreadsheet and sheet
  - Tool description states it should contain patent numbers, registrations, records, filing dates, and status information
  - Both `documentId` and `sheetName` are currently blank placeholders in the JSON
- **Connections:** AI tool output to **IP Monitoring Agent**.
- **Version-specific notes:** Type version `4.7`.
- **Edge cases/failures:**
  - Will not work until spreadsheet and tab are selected
  - Google OAuth credential issues
  - The agent may not reliably query Sheets unless the tab schema is structured and consistent

#### Monitoring Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON output for monitoring.
- **Configuration choices:**
  - Manual JSON schema with fields:
    - `conflicts_detected` boolean
    - `conflict_severity` enum: critical/high/medium/low/none
    - `conflicting_assets` array of objects
    - `lifecycle_alerts` array of objects
    - `recommendations` array of strings
- **Connections:** AI output parser input to **IP Monitoring Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:**
  - Agent output may fail schema validation
  - Missing required structure may halt downstream assumptions

---

## 2.3 AI-Based Governance Processing

### Overview
This block takes monitoring results and determines governance actions. It assesses approval requirements, licensing compliance, documentation needs, stakeholder notifications, and required next steps.

### Nodes Involved
- IP Governance Agent
- Governance Agent Model
- Governance Memory
- Licensing Compliance Agent
- Licensing Compliance Model
- Documentation Generation Agent
- Documentation Model
- Governance Output Parser

### Node Details

#### IP Governance Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent` — governance orchestration layer.
- **Configuration choices:**
  - Input text:
    - uses `$json.output`
    - falls back to a governance processing instruction
  - System message defines responsibilities:
    - approvals
    - compliance checks
    - legal documentation
    - stakeholder coordination
  - Prompt type: `define`
  - Structured output enabled
- **Key expressions/variables:**
  - `={{ $json.output || 'Process IP monitoring results and determine governance actions: approvals needed, compliance checks, documentation requirements, and stakeholder notifications.' }}`
- **Connections:**
  - Main input from **IP Monitoring Agent**
  - AI language model from **Governance Agent Model**
  - AI memory from **Governance Memory**
  - AI tools from:
    - **Licensing Compliance Agent**
    - **Documentation Generation Agent**
  - AI output parser from **Governance Output Parser**
  - Main outputs to:
    - **Route by Severity**
    - **Prepare Governance Record**
    - **Format Analytics Data**
- **Version-specific notes:** Type version `3.1`.
- **Edge cases/failures:**
  - `$json.output` may be an object; the agent input handling depends on node internals and may stringify it
  - Structured parser mismatch can fail execution
  - Governance conclusions may be only as good as upstream monitoring detail

#### Governance Agent Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT-4o for governance orchestration.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Connections:** AI language model input to **IP Governance Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:** OpenAI auth/rate-limit/model failures.

#### Governance Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term governance context storage.
- **Configuration choices:** Defaults.
- **Connections:** AI memory input to **IP Governance Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:** Low direct risk.

#### Licensing Compliance Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist governance tool for license compliance analysis.
- **Configuration choices:**
  - Input from AI variable `ip_and_license_info`
  - System message checks:
    - terms and conditions
    - usage restrictions
    - royalty obligations
    - sublicensing rights
    - territorial limitations
    - expiration dates
  - Severity labels: critical/high/medium/low
- **Key expressions/variables:**
  - `={{ $fromAI('ip_and_license_info', 'The IP asset and licensing information to verify for compliance') }}`
- **Connections:**
  - AI language model from **Licensing Compliance Model**
  - AI tool output to **IP Governance Agent**
- **Version-specific notes:** Type version `3`.
- **Edge cases/failures:**
  - Missing license data prevents meaningful compliance checks
  - Risk of AI-generated legal interpretation without authoritative contract data

#### Licensing Compliance Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Connections:** AI language model input to **Licensing Compliance Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:** Standard OpenAI issues.

#### Documentation Generation Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool` — legal/documentation specialist.
- **Configuration choices:**
  - Input from AI variable `documentation_request`
  - System message instructs generation of:
    - approval memos
    - compliance audit reports
    - licensing summaries
    - conflict resolution reports
    - renewal notices
    - governance decision records
  - Requires formal legal structure
- **Key expressions/variables:**
  - `={{ $fromAI('documentation_request', 'The type of documentation needed and the IP/governance data to include') }}`
- **Connections:**
  - AI language model from **Documentation Model**
  - AI tool output to **IP Governance Agent**
- **Version-specific notes:** Type version `3`.
- **Edge cases/failures:**
  - Output may be long and token-heavy
  - Content may require legal review before operational use

#### Documentation Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Connections:** AI language model input to **Documentation Generation Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:** Standard OpenAI issues.

#### Governance Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces governance output schema.
- **Configuration choices:**
  - Manual schema with fields:
    - `approval_required`
    - `approval_status`
    - `compliance_status`
    - `compliance_violations`
    - `documentation_generated`
    - `stakeholders_to_notify`
    - `actions_required`
- **Connections:** AI output parser input to **IP Governance Agent**.
- **Version-specific notes:** Type version `1.3`.
- **Edge cases/failures:**
  - Invalid schema output from the agent
  - Downstream nodes assume these fields exist under `$json.output`

---

## 2.4 Conflict Routing and Notification

### Overview
This block routes monitoring results by conflict severity, stores normalized conflict records, and sends alerts. The routing is partly declarative through a Switch node and partly procedural through an If node.

### Nodes Involved
- Route by Severity
- Prepare Conflict Record
- Store IP Conflicts
- Alert Critical Conflicts
- Check Severity Level
- Email High Priority Conflicts
- Notify Medium Conflicts

### Node Details

#### Route by Severity
- **Type and role:** `n8n-nodes-base.switch` — routes governance output according to monitoring conflict severity.
- **Configuration choices:**
  - Three rules:
    - `critical`
    - `high`
    - `medium`
  - All three outputs currently connect to the same node: **Prepare Conflict Record**
- **Key expressions/variables:**
  - `={{ $json.output.conflict_severity }}`
- **Connections:**
  - Input from **IP Governance Agent**
  - All three outputs to **Prepare Conflict Record**
- **Version-specific notes:** Type version `3.4`.
- **Edge cases/failures:**
  - This node references `conflict_severity` under governance output, but that field belongs to the monitoring output schema, not the governance output schema
  - As a result, the Switch may not match anything unless the governance agent reproduces this field incidentally
  - Since all branches go to the same node anyway, the switch currently does not differentiate downstream processing

#### Prepare Conflict Record
- **Type and role:** `n8n-nodes-base.set` — normalizes conflict-related data for storage and alerts.
- **Configuration choices:**
  - Sets:
    - `conflict_severity`
    - `conflicts_detected`
    - `conflicting_assets` as JSON string
    - `recommendations` as JSON string
    - `timestamp`
    - `monitoring_date`
- **Key expressions/variables:**
  - `={{ $json.output.conflict_severity }}`
  - `={{ JSON.stringify($json.output.conflicting_assets) }}`
  - `={{ JSON.stringify($json.output.recommendations) }}`
  - `={{ $now.toISO() }}`
  - `={{ $now.toFormat('yyyy-MM-dd') }}`
- **Connections:** Input from **Route by Severity**; output to **Store IP Conflicts**.
- **Version-specific notes:** Type version `3.4`.
- **Edge cases/failures:**
  - Same schema mismatch risk as above: this node expects monitoring fields inside `$json.output` after the governance agent
  - Stringifying undefined values may produce `"undefined"` or fail depending on execution context expectations

#### Store IP Conflicts
- **Type and role:** `n8n-nodes-base.dataTable` — stores conflict records in an n8n Data Table.
- **Configuration choices:**
  - Auto-maps input data to table columns
  - Data table ID is a placeholder: `ip_conflicts_table`
- **Connections:** Input from **Prepare Conflict Record**; output to **Alert Critical Conflicts**.
- **Version-specific notes:** Type version `1.1`.
- **Edge cases/failures:**
  - Will fail until a valid Data Table ID is selected
  - Schema mismatches between mapped fields and table columns

#### Alert Critical Conflicts
- **Type and role:** `n8n-nodes-base.slack` — sends Slack alert for stored conflict records.
- **Configuration choices:**
  - OAuth2 authentication
  - Sends to configured channel placeholder `critical_alerts_channel`
  - Message contains severity, conflicts, recommendations, and urgent wording
- **Key expressions/variables:**
  - Uses `{{ $json.conflict_severity }}`, `{{ $json.conflicting_assets }}`, `{{ $json.recommendations }}`
- **Connections:** Input from **Store IP Conflicts**; output to **Check Severity Level**.
- **Version-specific notes:** Type version `2.4`.
- **Edge cases/failures:**
  - Slack auth or channel access failure
  - Despite its name, it runs for every record that reaches it, not only critical ones
  - Can create over-alerting if medium/high records also flow here

#### Check Severity Level
- **Type and role:** `n8n-nodes-base.if` — checks whether the current conflict severity is `high`.
- **Configuration choices:**
  - Case-insensitive string equality on `={{ $json.conflict_severity }} == high`
- **Connections:**
  - Input from **Alert Critical Conflicts**
  - True output to **Email High Priority Conflicts**
  - False output to **Notify Medium Conflicts**
- **Version-specific notes:** Type version `2.3`.
- **Edge cases/failures:**
  - Critical items go to the false branch and therefore to the medium Slack notifier
  - Low or none severities are not explicitly handled but may also fall through if upstream routing changes
  - Logic is inconsistent with node names

#### Email High Priority Conflicts
- **Type and role:** `n8n-nodes-base.gmail` — sends high-priority email to the legal team.
- **Configuration choices:**
  - Recipient placeholder: `legal_team_email`
  - Subject includes severity
  - Body includes severity, conflicting assets, recommendations
- **Connections:** True branch from **Check Severity Level**.
- **Version-specific notes:** Type version `2.2`.
- **Edge cases/failures:**
  - Gmail OAuth issues
  - Placeholder recipient must be replaced
  - No attachment support used despite legal-review use case

#### Notify Medium Conflicts
- **Type and role:** `n8n-nodes-base.slack` — posts lower-priority conflict messages to a monitoring channel.
- **Configuration choices:**
  - OAuth2 authentication
  - Channel placeholder: `ip_monitoring_channel`
  - Message text labels conflict as medium priority
- **Connections:** False branch from **Check Severity Level**.
- **Version-specific notes:** Type version `2.4`.
- **Edge cases/failures:**
  - Because of current logic, critical items can reach this node
  - Slack credential/channel access issues

---

## 2.5 Governance Logging and Compliance Reporting

### Overview
This block formats governance results for storage and sends a compliance report email. It provides an audit trail of governance decisions.

### Nodes Involved
- Prepare Governance Record
- Store Governance Decisions
- Send Compliance Report

### Node Details

#### Prepare Governance Record
- **Type and role:** `n8n-nodes-base.set` — normalizes governance output into storable fields.
- **Configuration choices:**
  - Sets:
    - `approval_status`
    - `compliance_status`
    - `compliance_violations`
    - `documentation_generated`
    - `stakeholders_to_notify`
    - `actions_required`
    - `timestamp`
  - Several fields are labeled as array but assigned via `JSON.stringify(...)`
- **Key expressions/variables:**
  - `={{ $json.output.approval_status }}`
  - `={{ JSON.stringify($json.output.compliance_violations) }}`
  - `={{ JSON.stringify($json.output.documentation_generated) }}`
  - `={{ JSON.stringify($json.output.stakeholders_to_notify) }}`
  - `={{ JSON.stringify($json.output.actions_required) }}`
  - `={{ $now.toISO() }}`
- **Connections:** Input from **IP Governance Agent**; output to **Store Governance Decisions**.
- **Version-specific notes:** Type version `3.4`.
- **Edge cases/failures:**
  - Potential type inconsistency: fields declared as array while values are stringified JSON
  - If output parser failed upstream, these expressions may evaluate to undefined

#### Store Governance Decisions
- **Type and role:** `n8n-nodes-base.dataTable` — stores governance records in an n8n Data Table.
- **Configuration choices:**
  - Auto-maps fields
  - Data Table ID placeholder: `governance_decisions_table`
- **Connections:** Input from **Prepare Governance Record**; output to **Send Compliance Report**.
- **Version-specific notes:** Type version `1.1`.
- **Edge cases/failures:**
  - Requires a real Data Table selection
  - Data type mismatches may occur if columns are strongly typed

#### Send Compliance Report
- **Type and role:** `n8n-nodes-base.gmail` — emails compliance/governance summary.
- **Configuration choices:**
  - Recipient placeholder: `compliance_team_email`
  - Subject uses current date
  - Message includes approval status, compliance status, violations, and actions
- **Key expressions/variables:**
  - `={{ $now.toFormat('yyyy-MM-dd') }}`
- **Connections:** Input from **Store Governance Decisions**.
- **Version-specific notes:** Type version `2.2`.
- **Edge cases/failures:**
  - Gmail OAuth issues
  - Placeholder recipient must be configured
  - No actual file attachment is generated despite wording "Attached is the IP compliance report"

---

## 2.6 Analytics Export

### Overview
This block produces a lightweight analytics record from AI outputs and appends it to Google Sheets. It supports reporting on conflict volume, severity, lifecycle alerts, and compliance violations.

### Nodes Involved
- Format Analytics Data
- Export Analytics to Sheets

### Node Details

#### Format Analytics Data
- **Type and role:** `n8n-nodes-base.set` — derives summary metrics.
- **Configuration choices:**
  - Computes:
    - `total_conflicts`
    - `critical_count`
    - `high_count`
    - `lifecycle_alerts_count`
    - `compliance_violations_count`
    - `report_date`
- **Key expressions/variables:**
  - `={{ $json.output.conflicting_assets ? $json.output.conflicting_assets.length : 0 }}`
  - `={{ $json.output.conflict_severity === 'critical' ? 1 : 0 }}`
  - `={{ $json.output.conflict_severity === 'high' ? 1 : 0 }}`
  - `={{ $json.output.lifecycle_alerts ? $json.output.lifecycle_alerts.length : 0 }}`
  - `={{ $('IP Governance Agent').item.json.output.compliance_violations ? $('IP Governance Agent').item.json.output.compliance_violations.length : 0 }}`
  - `={{ $now.toFormat('yyyy-MM-dd') }}`
- **Connections:** Input from **IP Governance Agent**; output to **Export Analytics to Sheets**.
- **Version-specific notes:** Type version `3.4`.
- **Edge cases/failures:**
  - Like the routing block, this node expects monitoring output fields under the governance agent output, which is structurally inconsistent
  - Cross-node expression `$('IP Governance Agent').item...` can fail if item linking differs from expectations
  - Compliance count depends on governance parser success

#### Export Analytics to Sheets
- **Type and role:** `n8n-nodes-base.googleSheets` — appends analytics row to a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet and sheet tab are currently blank placeholders
- **Connections:** Input from **Format Analytics Data**.
- **Version-specific notes:** Type version `4.7`.
- **Edge cases/failures:**
  - Will fail until `documentId` and `sheetName` are configured
  - Header mismatch may lead to unexpected column mapping

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule IP Monitoring | n8n-nodes-base.scheduleTrigger | Daily scheduled entry point |  | IP Monitoring Agent | ## How It Works<br>This workflow automates intellectual property (IP) monitoring, conflict detection, and governance reporting for IP counsel, legal operations teams, and compliance officers. It eliminates the manual effort of tracking IP lifecycle events, identifying conflicting registrations, and routing compliance actions across severity levels. IP alerts arrive via webhook and scheduled monitoring triggers. Both feeds are parsed and passed to the IP Monitoring Agent, backed by a monitoring model, shared memory, a Conflict Detection Agent (with knowledge base), an IP Lifecycle Tracking Agent, and an IP Tracking Sheet Tool. Structured outputs feed the IP Governance Agent, which coordinates a Governance Agent (with memory), a Licensing Compliance Agent, and a Documentation Generation Agent. Outputs are routed by severity: analytics data is formatted and exported to Google Sheets; conflicts are prepared, shared to Slack, and routed by severity level, critical conflicts trigger Gmail alerts while medium conflicts receive Slack notifications. Governance records, compliance reports, and governance decisions are stored and distributed in parallel.<br>## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| External IP Alert Webhook | n8n-nodes-base.webhook | Real-time external alert entry point |  | Parse Webhook Data | ## How It Works<br>This workflow automates intellectual property (IP) monitoring, conflict detection, and governance reporting for IP counsel, legal operations teams, and compliance officers. It eliminates the manual effort of tracking IP lifecycle events, identifying conflicting registrations, and routing compliance actions across severity levels. IP alerts arrive via webhook and scheduled monitoring triggers. Both feeds are parsed and passed to the IP Monitoring Agent, backed by a monitoring model, shared memory, a Conflict Detection Agent (with knowledge base), an IP Lifecycle Tracking Agent, and an IP Tracking Sheet Tool. Structured outputs feed the IP Governance Agent, which coordinates a Governance Agent (with memory), a Licensing Compliance Agent, and a Documentation Generation Agent. Outputs are routed by severity: analytics data is formatted and exported to Google Sheets; conflicts are prepared, shared to Slack, and routed by severity level, critical conflicts trigger Gmail alerts while medium conflicts receive Slack notifications. Governance records, compliance reports, and governance decisions are stored and distributed in parallel.<br>## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| Parse Webhook Data | n8n-nodes-base.set | Normalizes webhook payload for monitoring | External IP Alert Webhook | IP Monitoring Agent | ## How It Works<br>This workflow automates intellectual property (IP) monitoring, conflict detection, and governance reporting for IP counsel, legal operations teams, and compliance officers. It eliminates the manual effort of tracking IP lifecycle events, identifying conflicting registrations, and routing compliance actions across severity levels. IP alerts arrive via webhook and scheduled monitoring triggers. Both feeds are parsed and passed to the IP Monitoring Agent, backed by a monitoring model, shared memory, a Conflict Detection Agent (with knowledge base), an IP Lifecycle Tracking Agent, and an IP Tracking Sheet Tool. Structured outputs feed the IP Governance Agent, which coordinates a Governance Agent (with memory), a Licensing Compliance Agent, and a Documentation Generation Agent. Outputs are routed by severity: analytics data is formatted and exported to Google Sheets; conflicts are prepared, shared to Slack, and routed by severity level, critical conflicts trigger Gmail alerts while medium conflicts receive Slack notifications. Governance records, compliance reports, and governance decisions are stored and distributed in parallel.<br>## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Monitoring Agent | @n8n/n8n-nodes-langchain.agent | Main AI orchestrator for IP monitoring | Schedule IP Monitoring, Parse Webhook Data, Monitoring Agent Model, Monitoring Memory, IP Conflict Detection Agent, IP Lifecycle Tracking Agent, IP Tracking Sheet Tool, Monitoring Output Parser | IP Governance Agent | ## How It Works<br>This workflow automates intellectual property (IP) monitoring, conflict detection, and governance reporting for IP counsel, legal operations teams, and compliance officers. It eliminates the manual effort of tracking IP lifecycle events, identifying conflicting registrations, and routing compliance actions across severity levels. IP alerts arrive via webhook and scheduled monitoring triggers. Both feeds are parsed and passed to the IP Monitoring Agent, backed by a monitoring model, shared memory, a Conflict Detection Agent (with knowledge base), an IP Lifecycle Tracking Agent, and an IP Tracking Sheet Tool. Structured outputs feed the IP Governance Agent, which coordinates a Governance Agent (with memory), a Licensing Compliance Agent, and a Documentation Generation Agent. Outputs are routed by severity: analytics data is formatted and exported to Google Sheets; conflicts are prepared, shared to Slack, and routed by severity level, critical conflicts trigger Gmail alerts while medium conflicts receive Slack notifications. Governance records, compliance reports, and governance decisions are stored and distributed in parallel.<br>## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| Monitoring Agent Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for monitoring agent |  | IP Monitoring Agent | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Gmail account with OAuth credentials<br>- Google Sheets with IP analytics and governance tabs pre-created<br>- IP knowledge base or registry API access<br>## Use Cases<br>- IP counsel automating conflict detection across trademark and patent registrations<br>## Customisation<br>- Extend severity routing to include additional tiers such as low-risk advisory notifications<br>## Benefits<br>- Dual-trigger ingestion eliminates IP monitoring blind spots between scheduled and real-time events<br>## Setup Steps<br>1. Import workflow; configure the External IP Alert Webhook and Schedule IP Monitoring trigger interval.<br>2. Add AI model credentials to the IP Monitoring Agent<br>3. Connect Slack credentials to the Share IP Conflicts and Notify Medium Conflicts nodes.<br>4. Link Gmail credentials to the Email High Priority Conflicts and Send Compliance Report nodes.<br>5. Link Google Sheets credentials; set sheet IDs for IP Analytics, Governance Decisions, and Compliance Report tabs.<br>6. Configure the IP Knowledge Base tool with your IP registry or database source.<br>## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| Monitoring Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Context memory for monitoring agent |  | IP Monitoring Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Conflict Detection Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist tool for conflict detection | Conflict Detection Model, IP Knowledge Base | IP Monitoring Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| Conflict Detection Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for conflict detection tool |  | IP Conflict Detection Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Knowledge Base | @n8n/n8n-nodes-langchain.vectorStoreInMemory | Vector retrieval tool for IP similarity search | IP Embeddings | IP Conflict Detection Agent | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Gmail account with OAuth credentials<br>- Google Sheets with IP analytics and governance tabs pre-created<br>- IP knowledge base or registry API access<br>## Use Cases<br>- IP counsel automating conflict detection across trademark and patent registrations<br>## Customisation<br>- Extend severity routing to include additional tiers such as low-risk advisory notifications<br>## Benefits<br>- Dual-trigger ingestion eliminates IP monitoring blind spots between scheduled and real-time events<br>## Setup Steps<br>1. Import workflow; configure the External IP Alert Webhook and Schedule IP Monitoring trigger interval.<br>2. Add AI model credentials to the IP Monitoring Agent<br>3. Connect Slack credentials to the Share IP Conflicts and Notify Medium Conflicts nodes.<br>4. Link Gmail credentials to the Email High Priority Conflicts and Send Compliance Report nodes.<br>5. Link Google Sheets credentials; set sheet IDs for IP Analytics, Governance Decisions, and Compliance Report tabs.<br>6. Configure the IP Knowledge Base tool with your IP registry or database source.<br>## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Embeddings backend for vector retrieval |  | IP Knowledge Base | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Lifecycle Tracking Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist tool for lifecycle/renewal analysis | Lifecycle Tracking Model | IP Monitoring Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| Lifecycle Tracking Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for lifecycle tracking tool |  | IP Lifecycle Tracking Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Tracking Sheet Tool | n8n-nodes-base.googleSheetsTool | AI tool to read IP asset tracking sheet |  | IP Monitoring Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| Monitoring Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for monitoring results |  | IP Monitoring Agent | ## IP Monitoring Agent & Specialist Sub-Agents<br>**Why** — Coordinates Conflict Detection, Lifecycle Tracking, and IP Tracking Sheet tools using shared memory for consistent, context-aware IP surveillance. |
| IP Governance Agent | @n8n/n8n-nodes-langchain.agent | Main AI orchestrator for governance actions | IP Monitoring Agent, Governance Agent Model, Governance Memory, Licensing Compliance Agent, Documentation Generation Agent, Governance Output Parser | Route by Severity, Prepare Governance Record, Format Analytics Data | ## How It Works<br>This workflow automates intellectual property (IP) monitoring, conflict detection, and governance reporting for IP counsel, legal operations teams, and compliance officers. It eliminates the manual effort of tracking IP lifecycle events, identifying conflicting registrations, and routing compliance actions across severity levels. IP alerts arrive via webhook and scheduled monitoring triggers. Both feeds are parsed and passed to the IP Monitoring Agent, backed by a monitoring model, shared memory, a Conflict Detection Agent (with knowledge base), an IP Lifecycle Tracking Agent, and an IP Tracking Sheet Tool. Structured outputs feed the IP Governance Agent, which coordinates a Governance Agent (with memory), a Licensing Compliance Agent, and a Documentation Generation Agent. Outputs are routed by severity: analytics data is formatted and exported to Google Sheets; conflicts are prepared, shared to Slack, and routed by severity level, critical conflicts trigger Gmail alerts while medium conflicts receive Slack notifications. Governance records, compliance reports, and governance decisions are stored and distributed in parallel.<br>## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Governance Agent Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for governance agent |  | IP Governance Agent | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Gmail account with OAuth credentials<br>- Google Sheets with IP analytics and governance tabs pre-created<br>- IP knowledge base or registry API access<br>## Use Cases<br>- IP counsel automating conflict detection across trademark and patent registrations<br>## Customisation<br>- Extend severity routing to include additional tiers such as low-risk advisory notifications<br>## Benefits<br>- Dual-trigger ingestion eliminates IP monitoring blind spots between scheduled and real-time events<br>## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Governance Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Context memory for governance agent |  | IP Governance Agent | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Licensing Compliance Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist tool for licensing compliance analysis | Licensing Compliance Model | IP Governance Agent | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Licensing Compliance Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for compliance tool |  | Licensing Compliance Agent | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Documentation Generation Agent | @n8n/n8n-nodes-langchain.agentTool | Specialist tool for legal documentation generation | Documentation Model | IP Governance Agent | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Documentation Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for documentation tool |  | Documentation Generation Agent | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Governance Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for governance results |  | IP Governance Agent | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready. |
| Route by Severity | n8n-nodes-base.switch | Routes conflict outputs by severity | IP Governance Agent | Prepare Conflict Record | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Prepare Conflict Record | n8n-nodes-base.set | Normalizes conflict data for storage/alerting | Route by Severity | Store IP Conflicts | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Store IP Conflicts | n8n-nodes-base.dataTable | Persists conflict records | Prepare Conflict Record | Alert Critical Conflicts | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Alert Critical Conflicts | n8n-nodes-base.slack | Sends Slack alert for conflict records | Store IP Conflicts | Check Severity Level | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Check Severity Level | n8n-nodes-base.if | Distinguishes high from non-high conflict records | Alert Critical Conflicts | Email High Priority Conflicts, Notify Medium Conflicts | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Email High Priority Conflicts | n8n-nodes-base.gmail | Emails legal team for high-priority conflicts | Check Severity Level |  | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Notify Medium Conflicts | n8n-nodes-base.slack | Sends lower-priority Slack notification | Check Severity Level |  | ## Route by Severity & Conflict Notification<br>**Why** — Rules-based severity routing sends critical conflicts to Gmail and medium conflicts to Slack, ensuring proportionate and timely response. |
| Prepare Governance Record | n8n-nodes-base.set | Normalizes governance data for storage | IP Governance Agent | Store Governance Decisions | ## IP Governance Agent & Compliance Agents<br>**Why** — Delegates licensing compliance checks and documentation generation in parallel, ensuring governance outputs are structured and audit-ready.<br>## Prepare Governance Record, Store Decisions & Send Compliance Report<br>**Why** — Captures governance outcomes, stores decisions, and distributes compliance reports via Gmail to maintain a complete, audit-ready IP governance trail. |
| Store Governance Decisions | n8n-nodes-base.dataTable | Persists governance decisions | Prepare Governance Record | Send Compliance Report | ## Prepare Governance Record, Store Decisions & Send Compliance Report<br>**Why** — Captures governance outcomes, stores decisions, and distributes compliance reports via Gmail to maintain a complete, audit-ready IP governance trail. |
| Send Compliance Report | n8n-nodes-base.gmail | Emails governance/compliance summary | Store Governance Decisions |  | ## Prepare Governance Record, Store Decisions & Send Compliance Report<br>**Why** — Captures governance outcomes, stores decisions, and distributes compliance reports via Gmail to maintain a complete, audit-ready IP governance trail. |
| Format Analytics Data | n8n-nodes-base.set | Builds analytics summary row | IP Governance Agent | Export Analytics to Sheets | ## Format Analytics & Export to Google Sheets<br>**Why** — Standardises analytics output and logs to Sheets for stakeholder visibility and long-term IP portfolio reporting. |
| Export Analytics to Sheets | n8n-nodes-base.googleSheets | Appends analytics row to Google Sheets | Format Analytics Data |  | ## Format Analytics & Export to Google Sheets<br>**Why** — Standardises analytics output and logs to Sheets for stakeholder visibility and long-term IP portfolio reporting. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `AI-powered IP monitoring and governance with conflict detection and compliance`.

2. **Add a Schedule Trigger node** named **Schedule IP Monitoring**.
   - Type: `Schedule Trigger`
   - Configure it to run daily at 09:00.
   - This will be one workflow entry point.

3. **Add a Webhook node** named **External IP Alert Webhook**.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `ip-alert-webhook`
   - Response Mode: `Last Node`
   - This is the real-time alert entry point.

4. **Add a Set node** named **Parse Webhook Data** after the webhook.
   - Create fields:
     - `monitoring_request` = `{{ $json.body.alert_message || $json.body.message || 'External IP alert received' }}`
     - `source` = `webhook`
     - `alert_type` = `{{ $json.body.alert_type || 'external' }}`
   - Connect **External IP Alert Webhook → Parse Webhook Data**.

5. **Add an AI Agent node** named **IP Monitoring Agent**.
   - Type: `AI Agent` / LangChain agent node
   - Prompt source: define manually
   - User text:
     - `{{ $json.monitoring_request || 'Perform daily IP monitoring: check all active patents, trademarks, and copyrights for conflicts, upcoming renewals, and lifecycle status changes.' }}`
   - Enable structured output parser
   - Add system instructions describing:
     - lifecycle monitoring
     - conflict detection
     - specialist coordination
     - structured recommendations
   - Connect:
     - **Schedule IP Monitoring → IP Monitoring Agent**
     - **Parse Webhook Data → IP Monitoring Agent**

6. **Add an OpenAI Chat Model node** named **Monitoring Agent Model**.
   - Type: OpenAI Chat Model
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Credential: OpenAI API credential
   - Connect this as the AI language model of **IP Monitoring Agent**.

7. **Add a Memory node** named **Monitoring Memory**.
   - Type: Buffer Window Memory
   - Default configuration is acceptable
   - Connect as AI memory to **IP Monitoring Agent**.

8. **Add a specialist AI tool node** named **IP Conflict Detection Agent**.
   - Type: AI Agent Tool
   - Input text:
     - `{{ $fromAI('ip_asset_details', 'The IP asset to analyze for conflicts including type, description, claims, and identifiers') }}`
   - Tool description: explain it analyzes IP conflicts and returns severity, conflicting asset IDs, and recommendations.
   - System message should instruct:
     - vector similarity search
     - keyword overlap checks
     - severity classification: CRITICAL, HIGH, MEDIUM, LOW
   - This node will be attached as a tool to **IP Monitoring Agent**.

9. **Add an OpenAI Chat Model node** named **Conflict Detection Model**.
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Connect it to **IP Conflict Detection Agent** as the model.

10. **Add an Embeddings node** named **IP Embeddings**.
    - Type: OpenAI Embeddings
    - Default options
    - Credential: OpenAI API credential

11. **Add a Vector Store node** named **IP Knowledge Base**.
    - Type: In-memory Vector Store
    - Mode: `Retrieve as Tool`
    - Memory key: `vector_store_key`
    - Tool description: reference patents, trademarks, copyrights, and legal docs
    - Connect **IP Embeddings → IP Knowledge Base**
    - Connect **IP Knowledge Base** as an AI tool to **IP Conflict Detection Agent**
    - Important: to make this useful, you must separately add a document-ingestion mechanism or replace this node with your real IP registry/vector backend.

12. **Add another specialist AI tool node** named **IP Lifecycle Tracking Agent**.
    - Input:
      - `{{ $fromAI('ip_asset_info', 'The IP asset to track including type, filing date, grant date, and jurisdiction') }}`
    - System message should instruct:
      - renewal tracking
      - maintenance fee checks
      - expiration calculations
      - categorization into urgent/upcoming/long-term windows
    - Tool description should mention lifecycle deadlines and remaining protection periods.

13. **Add an OpenAI Chat Model node** named **Lifecycle Tracking Model**.
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Connect it to **IP Lifecycle Tracking Agent**.

14. **Add a Google Sheets Tool node** named **IP Tracking Sheet Tool**.
    - Type: `Google Sheets Tool`
    - Connect Google Sheets OAuth2 credentials
    - Select:
      - Spreadsheet document ID
      - Sheet/tab containing IP asset tracking data
    - Tool description should mention patent numbers, trademark registrations, filing dates, statuses, and related IP data
    - Connect this node as an AI tool to **IP Monitoring Agent**.

15. **Add a Structured Output Parser node** named **Monitoring Output Parser**.
    - Use manual schema.
    - Create fields:
      - `conflicts_detected` boolean
      - `conflict_severity` enum: `critical`, `high`, `medium`, `low`, `none`
      - `conflicting_assets` array of objects with:
        - `ip_id`
        - `ip_type`
        - `similarity_score`
      - `lifecycle_alerts` array of objects with:
        - `ip_id`
        - `alert_type`
        - `days_until_action`
        - `action_required`
      - `recommendations` array of strings
    - Connect this parser to **IP Monitoring Agent**.

16. **Connect the monitoring specialist tools to the monitoring agent**.
    - **IP Conflict Detection Agent → IP Monitoring Agent** as AI tool
    - **IP Lifecycle Tracking Agent → IP Monitoring Agent** as AI tool
    - **IP Tracking Sheet Tool → IP Monitoring Agent** as AI tool

17. **Add another AI Agent node** named **IP Governance Agent**.
    - User text:
      - `{{ $json.output || 'Process IP monitoring results and determine governance actions: approvals needed, compliance checks, documentation requirements, and stakeholder notifications.' }}`
    - Enable structured output
    - System message should define responsibilities around:
      - approval workflows
      - licensing compliance
      - documentation generation
      - stakeholder notifications
    - Connect **IP Monitoring Agent → IP Governance Agent**.

18. **Add an OpenAI Chat Model node** named **Governance Agent Model**.
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect it to **IP Governance Agent**.

19. **Add a Memory node** named **Governance Memory**.
    - Type: Buffer Window Memory
    - Connect it to **IP Governance Agent**.

20. **Add a specialist AI tool node** named **Licensing Compliance Agent**.
    - Input:
      - `{{ $fromAI('ip_and_license_info', 'The IP asset and licensing information to verify for compliance') }}`
    - System message should check:
      - terms and conditions
      - restrictions
      - royalties
      - sublicensing
      - territorial limits
      - expiry dates
    - Tool description should mention compliance violation detection and remediation steps.

21. **Add an OpenAI Chat Model node** named **Licensing Compliance Model**.
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Connect it to **Licensing Compliance Agent**.

22. **Add a specialist AI tool node** named **Documentation Generation Agent**.
    - Input:
      - `{{ $fromAI('documentation_request', 'The type of documentation needed and the IP/governance data to include') }}`
    - System message should instruct generation of formal legal/governance documents with sections such as:
      - Executive Summary
      - Details
      - Analysis
      - Recommendations
      - Required Actions

23. **Add an OpenAI Chat Model node** named **Documentation Model**.
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Connect it to **Documentation Generation Agent**.

24. **Add a Structured Output Parser node** named **Governance Output Parser**.
    - Manual schema fields:
      - `approval_required` boolean
      - `approval_status` enum: `approved`, `rejected`, `pending_review`, `conditional`
      - `compliance_status` enum: `compliant`, `non_compliant`, `needs_review`
      - `compliance_violations` array of objects:
        - `violation_type`
        - `severity`
        - `description`
      - `documentation_generated` array of objects:
        - `doc_type`
        - `content`
      - `stakeholders_to_notify` array of strings
      - `actions_required` array of strings
    - Connect it to **IP Governance Agent**.

25. **Connect the governance specialist tools to the governance agent**.
    - **Licensing Compliance Agent → IP Governance Agent** as AI tool
    - **Documentation Generation Agent → IP Governance Agent** as AI tool

26. **Add a Switch node** named **Route by Severity**.
    - Create branches for:
      - `critical`
      - `high`
      - `medium`
    - Expression to inspect:
      - `{{ $json.output.conflict_severity }}`
    - Connect **IP Governance Agent → Route by Severity**
    - Note: as designed, this is structurally inconsistent because governance output schema does not define `conflict_severity`. To rebuild exactly, keep it. To improve it, route directly from **IP Monitoring Agent** instead.

27. **Add a Set node** named **Prepare Conflict Record**.
    - Fields:
      - `conflict_severity` = `{{ $json.output.conflict_severity }}`
      - `conflicts_detected` = `{{ $json.output.conflicts_detected }}`
      - `conflicting_assets` = `{{ JSON.stringify($json.output.conflicting_assets) }}`
      - `recommendations` = `{{ JSON.stringify($json.output.recommendations) }}`
      - `timestamp` = `{{ $now.toISO() }}`
      - `monitoring_date` = `{{ $now.toFormat('yyyy-MM-dd') }}`
    - Connect all Switch outputs to this node if you want to reproduce the original exactly.

28. **Add a Data Table node** named **Store IP Conflicts**.
    - Enable automatic field mapping
    - Create or select a Data Table for IP conflicts
    - Ensure it has compatible columns for the conflict record fields
    - Connect **Prepare Conflict Record → Store IP Conflicts**

29. **Add a Slack node** named **Alert Critical Conflicts**.
    - Authentication: Slack OAuth2
    - Select a channel for critical alerts
    - Message:
      - include severity, conflicting assets, recommendations, and an urgent warning
    - Connect **Store IP Conflicts → Alert Critical Conflicts**
    - Note: despite its name, this receives every record that reaches this path.

30. **Add an If node** named **Check Severity Level**.
    - Condition:
      - `{{ $json.conflict_severity }} equals high`
    - Connect **Alert Critical Conflicts → Check Severity Level**

31. **Add a Gmail node** named **Email High Priority Conflicts**.
    - Gmail OAuth2 credential required
    - To: legal team email address
    - Subject: `High Priority IP Conflict Alert - {{ $json.conflict_severity }}`
    - Message body: include severity, conflicting assets, recommendations
    - Connect the **true** output of **Check Severity Level** to this node.

32. **Add a Slack node** named **Notify Medium Conflicts**.
    - Slack OAuth2 credential required
    - Send to the monitoring channel
    - Message should indicate medium priority
    - Connect the **false** output of **Check Severity Level** to this node.

33. **Add a Set node** named **Prepare Governance Record**.
    - Fields:
      - `approval_status` = `{{ $json.output.approval_status }}`
      - `compliance_status` = `{{ $json.output.compliance_status }}`
      - `compliance_violations` = `{{ JSON.stringify($json.output.compliance_violations) }}`
      - `documentation_generated` = `{{ JSON.stringify($json.output.documentation_generated) }}`
      - `stakeholders_to_notify` = `{{ JSON.stringify($json.output.stakeholders_to_notify) }}`
      - `actions_required` = `{{ JSON.stringify($json.output.actions_required) }}`
      - `timestamp` = `{{ $now.toISO() }}`
    - Connect **IP Governance Agent → Prepare Governance Record**

34. **Add a Data Table node** named **Store Governance Decisions**.
    - Select or create a Data Table for governance decisions
    - Auto-map input data
    - Connect **Prepare Governance Record → Store Governance Decisions**

35. **Add a Gmail node** named **Send Compliance Report**.
    - Gmail OAuth2 credential
    - To: compliance team email
    - Subject: `IP Compliance Report - {{ $now.toFormat('yyyy-MM-dd') }}`
    - Message body: include approval status, compliance status, violations, and actions required
    - Connect **Store Governance Decisions → Send Compliance Report**

36. **Add a Set node** named **Format Analytics Data**.
    - Fields:
      - `total_conflicts` = `{{ $json.output.conflicting_assets ? $json.output.conflicting_assets.length : 0 }}`
      - `critical_count` = `{{ $json.output.conflict_severity === 'critical' ? 1 : 0 }}`
      - `high_count` = `{{ $json.output.conflict_severity === 'high' ? 1 : 0 }}`
      - `lifecycle_alerts_count` = `{{ $json.output.lifecycle_alerts ? $json.output.lifecycle_alerts.length : 0 }}`
      - `compliance_violations_count` = `{{ $('IP Governance Agent').item.json.output.compliance_violations ? $('IP Governance Agent').item.json.output.compliance_violations.length : 0 }}`
      - `report_date` = `{{ $now.toFormat('yyyy-MM-dd') }}`
    - Connect **IP Governance Agent → Format Analytics Data**
    - Note: exact reproduction keeps the current schema mismatch. A more robust version would merge monitoring and governance outputs before analytics.

37. **Add a Google Sheets node** named **Export Analytics to Sheets**.
    - Operation: `Append`
    - Connect Google Sheets OAuth2
    - Select spreadsheet and analytics tab
    - Connect **Format Analytics Data → Export Analytics to Sheets**

38. **Configure all credentials**:
    - **OpenAI**
      - Needed for:
        - Monitoring Agent Model
        - Conflict Detection Model
        - Lifecycle Tracking Model
        - Governance Agent Model
        - Licensing Compliance Model
        - Documentation Model
        - IP Embeddings
    - **Slack OAuth2**
      - Needed for:
        - Alert Critical Conflicts
        - Notify Medium Conflicts
    - **Gmail OAuth2**
      - Needed for:
        - Email High Priority Conflicts
        - Send Compliance Report
    - **Google Sheets OAuth2**
      - Needed for:
        - IP Tracking Sheet Tool
        - Export Analytics to Sheets

39. **Create external data structures before activating**:
    - Google Sheets workbook with:
      - IP tracking tab
      - Analytics tab
    - n8n Data Tables with suitable columns for:
      - IP conflicts
      - governance decisions

40. **Test both entry points**:
    - Trigger the schedule manually if supported
    - Send a POST request to the webhook with a body like:
      - `alert_message`
      - `alert_type`
    - Validate:
      - monitoring output schema
      - governance output schema
      - Slack delivery
      - Gmail delivery
      - Data Table inserts
      - Google Sheets append

41. **Recommended corrective improvements after reproduction**
    - Route severity from **IP Monitoring Agent** instead of **IP Governance Agent**
    - Send critical items directly to the critical alert path
    - Separate critical/high/medium branches explicitly
    - Populate or replace the in-memory vector store with a real knowledge source
    - Avoid storing stringified arrays in fields typed as arrays
    - Add error branches for Gmail, Slack, OpenAI, and Google Sheets failures

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API key or compatible LLM is required. | Prerequisite |
| Slack workspace with bot credentials is required. | Prerequisite |
| Gmail account with OAuth credentials is required. | Prerequisite |
| Google Sheets with IP analytics and governance tabs pre-created is required. | Prerequisite |
| IP knowledge base or registry API access is expected. | Prerequisite |
| Primary use case: IP counsel automating conflict detection across trademark and patent registrations. | Use case |
| Suggested customization: extend severity routing to include additional tiers such as low-risk advisory notifications. | Design note |
| Main stated benefit: dual-trigger ingestion reduces blind spots between scheduled and real-time events. | Design note |
| Setup guidance in canvas notes says to configure webhook path, schedule interval, AI credentials, Slack, Gmail, Google Sheets, and the knowledge base source. | Canvas guidance |
| The workflow description on the canvas states that governance records, compliance reports, and decisions are stored and distributed in parallel. | Process note |
| Important implementation caveat: the current routing and analytics sections rely on `conflict_severity` and other monitoring fields after the governance agent, even though those fields are not defined in the governance output schema. | Structural warning |
| Important implementation caveat: the in-memory vector store has embeddings configured but no document-ingestion step in this workflow, so conflict retrieval may be ineffective unless another process populates it. | Structural warning |