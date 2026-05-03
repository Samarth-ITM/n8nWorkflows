Monitor grid telemetry and automate compliance alerts with GPT-4o and Slack

https://n8nworkflows.xyz/workflows/monitor-grid-telemetry-and-automate-compliance-alerts-with-gpt-4o-and-slack-13915


# Monitor grid telemetry and automate compliance alerts with GPT-4o and Slack

# 1. Workflow Overview

This workflow ingests electrical grid telemetry through an HTTP webhook, uses a central AI coordination agent plus four specialized AI sub-agents to validate signals, assess regulatory compliance, generate reports, and determine operator notifications, then stores results and distributes alerts/reports through Slack and email.

Target use cases include:

- Real-time anomaly detection for energy grid operations
- Automated compliance monitoring against grid standards
- Operator-facing alerting for critical or warning conditions
- Continuous storage of validated telemetry and compliance events for audit trails

The workflow is logically organized into the following blocks.

## 1.1 Telemetry Intake and Central Orchestration

A webhook receives telemetry payloads and passes them to a central Coordination Agent. This agent uses a chat model, memory, and four tool-agents to orchestrate validation, compliance assessment, reporting, and notifications.

## 1.2 Signal Validation and Structured Parsing

A dedicated Grid Signal Agent validates generation, load, and storage telemetry. It uses a model, a code-based validation tool, and a structured output parser to normalize results into a predictable schema.

## 1.3 Compliance Assessment

A Compliance Agent evaluates validated telemetry against regulatory thresholds and can consult stored compliance history from a data table to identify trends or recurring issues.

## 1.4 Reporting and Notification Planning

A Reporting Agent generates operator-friendly compliance summaries, while a Notification Agent determines whether Slack alerts should be sent and to which channel, based on severity.

## 1.5 Persistence and Distribution

The final output from the Coordination Agent is transformed for storage and distribution. Validated telemetry is stored, compliance alerts are prepared and stored separately, and a report email is sent.

---

# 2. Block-by-Block Analysis

## 2.1 Telemetry Intake and Central Orchestration

### Overview

This block is the workflow entry point and main decision hub. It receives telemetry via HTTP POST and delegates analysis to a central AI agent that can call specialized sub-agents as tools.

### Nodes Involved

- Grid Telemetry Webhook
- Coordination Agent
- Coordination Model
- Coordination Memory

### Node Details

#### Grid Telemetry Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Receives incoming HTTP POST requests containing telemetry data.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `grid-telemetry`
  - No extra webhook options configured
- **Key expressions or variables used:**  
  None in the node itself.
- **Input and output connections:**  
  - No input; this is an entry node
  - Outputs to: `Coordination Agent`
- **Version-specific requirements:**  
  Type version `2.1`
- **Edge cases or potential failure types:**  
  - Webhook path conflicts with another workflow
  - Invalid or unexpected request body structure
  - Empty request body may lead to downstream expression issues
  - If test URL is used instead of production URL, external systems may fail after activation changes
- **Sub-workflow reference:**  
  None

#### Coordination Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Central orchestration agent that receives telemetry text and decides when to call specialized AI tools.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Input text: `={{ $json.body }}`
  - System message explicitly restricts behavior: it coordinates monitoring/reporting/notification but must never issue control commands to the grid
- **Key expressions or variables used:**  
  - `{{$json.body}}` as the primary telemetry payload
- **Input and output connections:**  
  - Main input from: `Grid Telemetry Webhook`
  - AI language model from: `Coordination Model`
  - AI memory from: `Coordination Memory`
  - AI tools available:
    - `Grid Signal Agent`
    - `Compliance Agent`
    - `Reporting Agent`
    - `Notification Agent`
  - Main output to: `Prepare Telemetry Storage`
- **Version-specific requirements:**  
  Type version `3.1`; requires a compatible n8n version supporting LangChain agent nodes and tool-agent attachments
- **Edge cases or potential failure types:**  
  - If webhook body is an object rather than a plain string, model interpretation may be inconsistent
  - Tool invocation may fail if downstream tool-agents return malformed outputs
  - Hallucinated structure can break the downstream Set nodes, which assume fields such as `output.validation_status`
  - Token/context issues if telemetry payloads are large
- **Sub-workflow reference:**  
  None; this is not an Execute Workflow node

#### Coordination Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  OpenAI chat model powering the Coordination Agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.3`
  - No built-in tools configured directly
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI language model to: `Coordination Agent`
- **Version-specific requirements:**  
  Type version `1.3`
  Requires valid OpenAI credentials in n8n
- **Edge cases or potential failure types:**  
  - Authentication or quota errors from OpenAI
  - Regional/model availability issues
  - Latency or timeout under heavy loads
- **Sub-workflow reference:**  
  None

#### Coordination Memory

- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Provides conversational memory to the Coordination Agent.
- **Configuration choices:**  
  - Default settings; no explicit window size shown
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI memory to: `Coordination Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Memory may not be very useful if each execution is stateless and independent
  - If long context accumulates, model cost and token use may increase
- **Sub-workflow reference:**  
  None

---

## 2.2 Signal Validation and Structured Parsing

### Overview

This block validates raw telemetry for operational plausibility and converts the result into a structured schema. It is the main data-quality gate before compliance analysis.

### Nodes Involved

- Grid Signal Agent
- Grid Signal Model
- Telemetry Validation Tool
- Telemetry Structure Parser

### Node Details

#### Grid Signal Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized AI tool-agent used by the Coordination Agent to validate telemetry and produce structured output.
- **Configuration choices:**  
  - Input text from AI: `={{ $fromAI('telemetry_data', 'Raw telemetry data to validate') }}`
  - System message instructs the agent to validate:
    - generation: MW output, frequency, voltage
    - load: consumption MW, power factor
    - storage: charge level and charge/discharge rate
  - Validation ranges include:
    - generation output: `0–1000 MW`
    - load: `0–800 MW`
    - storage: `0–100%`
    - frequency: `59.5–60.5 Hz`
  - Output parser enabled
- **Key expressions or variables used:**  
  - `$fromAI('telemetry_data', ...)`
- **Input and output connections:**  
  - Exposed as AI tool to: `Coordination Agent`
  - Uses AI language model: `Grid Signal Model`
  - Uses AI tool: `Telemetry Validation Tool`
  - Uses AI output parser: `Telemetry Structure Parser`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - If Coordination Agent does not pass a parseable telemetry object/string, validation quality drops
  - If parser schema and model output diverge, parsing can fail
  - The system prompt mentions data quality scores, but the code tool only returns issues/is_valid; the model must infer final structure
- **Sub-workflow reference:**  
  None

#### Grid Signal Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Chat model dedicated to telemetry validation.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1` for more deterministic validation behavior
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI language model to: `Grid Signal Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - OpenAI credential or rate-limit issues
  - Model may still produce slight schema drift despite low temperature
- **Sub-workflow reference:**  
  None

#### Telemetry Validation Tool

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  JavaScript tool callable by the Grid Signal Agent for explicit rule-based checks.
- **Configuration choices:**  
  - Reads tool input with `const telemetry = $fromAI('query');`
  - Performs checks:
    - generation output outside `0–1000`
    - frequency outside `59.5–60.5`
    - load consumption outside `0–800`
    - storage charge outside `0–100`
  - Returns a JSON string with:
    - `validation_issues`
    - `is_valid`
- **Key expressions or variables used:**  
  - `$fromAI('query')`
- **Input and output connections:**  
  - Exposed as AI tool to: `Grid Signal Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - If `telemetry` is a string instead of an object, property access may fail
  - Missing nested properties may produce `undefined` comparisons
  - Tool returns a stringified JSON object, so the agent/model must interpret it correctly
  - No voltage or power-factor validation is implemented, despite the prompt mentioning them
- **Sub-workflow reference:**  
  None

#### Telemetry Structure Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a structured JSON schema for the Grid Signal Agent output.
- **Configuration choices:**  
  - Manual JSON schema
  - Expected top-level fields:
    - `validation_status` enum: `valid`, `warning`, `critical`
    - `generation`
    - `load`
    - `storage`
    - `anomalies`
    - `timestamp`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI output parser to: `Grid Signal Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - If model output does not conform exactly, parser can fail
  - Schema does not mark fields as required; partially complete outputs may still pass, but downstream nodes may break
  - Downstream Set nodes assume `generation`, `load`, `storage`, and `anomalies` exist
- **Sub-workflow reference:**  
  None

---

## 2.3 Compliance Assessment

### Overview

This block analyzes validated telemetry against grid compliance rules and optionally compares the current event to historical compliance records.

### Nodes Involved

- Compliance Agent
- Compliance Model
- Compliance History Tool

### Node Details

#### Compliance Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized tool-agent for regulatory and operational compliance evaluation.
- **Configuration choices:**  
  - Input text from AI: `={{ $fromAI('validated_telemetry', 'Validated telemetry data to check for compliance') }}`
  - System message instructs the agent to check:
    - frequency within `59.95–60.05 Hz` per NERC BAL-001
    - voltage within ±5% nominal
    - generation-load balance within 1%
    - storage response time under 4 seconds
  - It should classify issues as informational, warning, or critical
- **Key expressions or variables used:**  
  - `$fromAI('validated_telemetry', ...)`
- **Input and output connections:**  
  - Exposed as AI tool to: `Coordination Agent`
  - Uses AI language model: `Compliance Model`
  - Uses AI tool: `Compliance History Tool`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Historical lookup may fail if data table is not configured
  - Some compliance checks rely on fields not guaranteed by upstream parser, such as response time or nominal voltage baseline
  - Agent output is not schema-constrained, so results may vary in structure
- **Sub-workflow reference:**  
  None

#### Compliance Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Language model for the Compliance Agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI language model to: `Compliance Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - OpenAI auth, quota, or timeout issues
  - Model-based compliance interpretation may not be deterministic enough for strict regulatory use without additional hard rules
- **Sub-workflow reference:**  
  None

#### Compliance History Tool

- **Type and technical role:** `n8n-nodes-base.dataTableTool`  
  AI-callable tool for retrieving historical compliance records from an n8n Data Table.
- **Configuration choices:**  
  - Operation: `get`
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__compliance_alerts_table__>`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Exposed as AI tool to: `Compliance Agent`
- **Version-specific requirements:**  
  Type version `1.1`
  Requires n8n Data Tables support
- **Edge cases or potential failure types:**  
  - Placeholder ID must be replaced with a real Data Table ID
  - Table access may fail due to permissions or non-existent table
  - Retrieved history may be too broad if query filtering is not configured
- **Sub-workflow reference:**  
  None

---

## 2.4 Reporting and Notification Planning

### Overview

This block turns analytical outputs into operator-facing communications. One agent generates comprehensive reports, and another determines whether to trigger Slack notifications and to which channel.

### Nodes Involved

- Reporting Agent
- Reporting Model
- Notification Agent
- Notification Model
- Slack Notification Tool

### Node Details

#### Reporting Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized AI tool-agent for generating compliance reports.
- **Configuration choices:**  
  - Input text from AI: `={{ $fromAI('compliance_data', 'Compliance assessment data to include in the report') }}`
  - System message asks for:
    - executive summary
    - detailed findings
    - recommendations
    - metrics, violations, severity, historical trends
- **Key expressions or variables used:**  
  - `$fromAI('compliance_data', ...)`
- **Input and output connections:**  
  - Exposed as AI tool to: `Coordination Agent`
  - Uses AI language model: `Reporting Model`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - No structured parser is attached, so report format may vary
  - If compliance output is sparse, report quality will be weak
- **Sub-workflow reference:**  
  None

#### Reporting Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Model used to generate the report text.
- **Configuration choices:**  
  - Model: `gpt-5-mini`
  - Default options; no custom temperature shown
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI language model to: `Reporting Agent`
- **Version-specific requirements:**  
  Type version `1.3`
  Requires the selected model to exist in the connected OpenAI account and n8n version
- **Edge cases or potential failure types:**  
  - `gpt-5-mini` availability may vary by account, region, or node version
  - If unavailable, the node will fail until changed to a supported model
- **Sub-workflow reference:**  
  None

#### Notification Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialized tool-agent responsible for notification routing and message drafting.
- **Configuration choices:**  
  - Input text from AI: `={{ $fromAI('alert_data', 'Alert and compliance data to notify operators about') }}`
  - System message defines routing logic:
    - CRITICAL → Slack `#grid-ops-critical`
    - WARNING → Slack `#grid-ops-alerts`
    - INFORMATIONAL → daily email only
  - Explicitly forbids suggesting control actions
- **Key expressions or variables used:**  
  - `$fromAI('alert_data', ...)`
- **Input and output connections:**  
  - Exposed as AI tool to: `Coordination Agent`
  - Uses AI language model: `Notification Model`
  - Uses AI tool: `Slack Notification Tool`
- **Version-specific requirements:**  
  Type version `3`
- **Edge cases or potential failure types:**  
  - Channel naming in the prompt may not match Slack channel IDs configured in the Slack tool
  - The agent may choose to send Slack for cases that should only be emailed unless the prompt is followed strictly
  - No deterministic branching exists outside the model
- **Sub-workflow reference:**  
  None

#### Notification Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Model powering the Notification Agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connects as AI language model to: `Notification Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - OpenAI auth/quota/timeouts
  - Slight variability in wording or routing decisions
- **Sub-workflow reference:**  
  None

#### Slack Notification Tool

- **Type and technical role:** `n8n-nodes-base.slackTool`  
  AI-callable Slack tool used by the Notification Agent to post messages to Slack.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Mode: send to channel
  - Message text: `={{ $fromAI('message', 'Alert message to send to operators') }}`
  - Channel ID: `={{ $fromAI('channel', 'Slack channel ID for the alert', 'string', '#grid-ops-alerts') }}`
  - Tool description: real-time alerts based on severity
- **Key expressions or variables used:**  
  - `$fromAI('message', ...)`
  - `$fromAI('channel', ..., 'string', '#grid-ops-alerts')`
- **Input and output connections:**  
  - Exposed as AI tool to: `Notification Agent`
- **Version-specific requirements:**  
  Type version `2.4`
  Requires Slack OAuth2 credentials and appropriate bot scopes
- **Edge cases or potential failure types:**  
  - The field is named `channelId`, but default AI value is a channel name string like `#grid-ops-alerts`; depending on Slack node behavior, actual channel ID may be required
  - Missing scopes such as chat posting permissions
  - Bot not invited to target channels
  - Invalid workspace/channel selection
- **Sub-workflow reference:**  
  None

---

## 2.5 Persistence and Distribution

### Overview

This block converts the Coordination Agent output into records suitable for storage and outbound reporting. It branches into telemetry storage, compliance alert storage, and report email delivery.

### Nodes Involved

- Prepare Telemetry Storage
- Store Validated Telemetry
- Prepare Compliance Alerts
- Store Compliance Alerts
- Send Report Email

### Node Details

#### Prepare Telemetry Storage

- **Type and technical role:** `n8n-nodes-base.set`  
  Maps the Coordination Agent result into a flatter schema for persistence and downstream outputs.
- **Configuration choices:**  
  Creates these fields:
  - `timestamp = {{$now.toISO()}}`
  - `validation_status = {{$json.output.validation_status}}`
  - `generation_output_mw = {{$json.output.generation.output_mw}}`
  - `generation_frequency_hz = {{$json.output.generation.frequency_hz}}`
  - `load_consumption_mw = {{$json.output.load.consumption_mw}}`
  - `storage_charge_percent = {{$json.output.storage.charge_percent}}`
  - `anomalies = {{$json.output.anomalies.join(', ')}}`
- **Key expressions or variables used:**  
  - `$now.toISO()`
  - `$json.output...`
- **Input and output connections:**  
  - Input from: `Coordination Agent`
  - Outputs to:
    - `Store Validated Telemetry`
    - `Prepare Compliance Alerts`
    - `Send Report Email`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Assumes `output.generation`, `output.load`, `output.storage`, and `output.anomalies` all exist
  - `anomalies.join(', ')` fails if `anomalies` is not an array
  - There is a structural mismatch risk because agent nodes do not always return data under a uniform `output` object unless configured that way
- **Sub-workflow reference:**  
  None

#### Store Validated Telemetry

- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Stores normalized telemetry records into an n8n Data Table.
- **Configuration choices:**  
  - Operation implied by node type/configuration: insert/update data row with auto-mapping
  - Mapping mode: `autoMapInputData`
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__validated_telemetry_table__>`
- **Key expressions or variables used:**  
  None directly
- **Input and output connections:**  
  - Input from: `Prepare Telemetry Storage`
- **Version-specific requirements:**  
  Type version `1.1`
  Requires Data Tables support and a pre-created target table
- **Edge cases or potential failure types:**  
  - Placeholder ID must be replaced
  - Auto-mapping fails if input fields do not match table schema
  - Permissions or table existence issues
- **Sub-workflow reference:**  
  None

#### Prepare Compliance Alerts

- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a flattened compliance alert record for persistence.
- **Configuration choices:**  
  Sets:
  - `timestamp = {{$now.toISO()}}`
  - `alert_severity = {{$json.output.validation_status}}`
  - `alert_type = compliance_check`
  - `description = {{$json.output.anomalies.join('; ')}}`
  - `generation_mw = {{$json.output.generation.output_mw}}`
  - `frequency_hz = {{$json.output.generation.frequency_hz}}`
- **Key expressions or variables used:**  
  - `$now.toISO()`
  - `$json.output...`
- **Input and output connections:**  
  - Input from: `Prepare Telemetry Storage`
  - Output to: `Store Compliance Alerts`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Same structural assumptions as previous Set node
  - Severity is derived from `validation_status`, which may not be the same as regulatory severity from the Compliance Agent
  - Could store validation warnings as compliance alerts even when compliance logic says otherwise
- **Sub-workflow reference:**  
  None

#### Store Compliance Alerts

- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Stores compliance alert/event records into a Data Table.
- **Configuration choices:**  
  - Auto-map input data to table columns
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__compliance_alerts_table__>`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from: `Prepare Compliance Alerts`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - Placeholder must be replaced
  - Schema mismatch during auto-mapping
  - If compliance alert table is the same one used by the history tool, column consistency matters
- **Sub-workflow reference:**  
  None

#### Send Report Email

- **Type and technical role:** `n8n-nodes-base.emailSend`  
  Sends a plain-text compliance report email.
- **Configuration choices:**  
  - Body text: `={{ $json.output }}`
  - Subject: `=Grid Compliance Report - {{ $now.format('yyyy-MM-dd HH:mm') }}`
  - To: placeholder operator email
  - From: placeholder sender email
  - Format: text
- **Key expressions or variables used:**  
  - `$json.output`
  - `$now.format('yyyy-MM-dd HH:mm')`
- **Input and output connections:**  
  - Input from: `Prepare Telemetry Storage`
- **Version-specific requirements:**  
  Type version `2.1`
  Requires configured email transport credentials
- **Edge cases or potential failure types:**  
  - If `$json.output` is an object, plain-text rendering may become `[object Object]` unless serialized upstream
  - Placeholder emails must be replaced
  - SMTP/OAuth auth issues
  - Deliverability/spam filtering
- **Sub-workflow reference:**  
  None

---

## 2.6 Documentation and Visual Annotation Nodes

### Overview

These nodes do not execute operational logic but provide setup guidance, architectural explanation, and block-level comments directly in the workflow canvas.

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

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation for prerequisites, use cases, customization ideas, and benefits.
- **Configuration choices:**  
  Colored note with multiline markdown content
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note1

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Setup checklist for webhook, AI credentials, Slack, email, and storage.
- **Configuration choices:**  
  Standard sticky note
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note2

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level explanation of how the entire workflow operates.
- **Configuration choices:**  
  Standard sticky note
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note3

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels reporting and notification section.
- **Configuration choices:**  
  Colored note
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note4

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels compliance-check section.
- **Configuration choices:**  
  Colored note
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note5

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels signal validation and parsing section.
- **Configuration choices:**  
  Colored note
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

#### Sticky Note6

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Labels storage and distribution section.
- **Configuration choices:**  
  Colored note
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  None operational
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Grid Telemetry Webhook | n8n-nodes-base.webhook | Receives POST telemetry payloads |  | Coordination Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Coordination Agent | @n8n/n8n-nodes-langchain.agent | Central AI orchestrator calling specialized sub-agents | Grid Telemetry Webhook; Coordination Model; Coordination Memory; Grid Signal Agent; Compliance Agent; Reporting Agent; Notification Agent | Prepare Telemetry Storage | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Coordination Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for orchestration |  | Coordination Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Coordination Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memory for the coordination agent |  | Coordination Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Grid Signal Agent | @n8n/n8n-nodes-langchain.agentTool | Validates and structures telemetry | Grid Signal Model; Telemetry Validation Tool; Telemetry Structure Parser | Coordination Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Grid Signal Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for signal validation |  | Grid Signal Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Telemetry Structure Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured telemetry output schema |  | Grid Signal Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Telemetry Validation Tool | @n8n/n8n-nodes-langchain.toolCode | Rule-based telemetry validation helper |  | Grid Signal Agent | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Compliance Agent | @n8n/n8n-nodes-langchain.agentTool | Evaluates telemetry against compliance rules | Compliance Model; Compliance History Tool | Coordination Agent | ## Compliance Check<br>**What** — Compliance Agent cross-references telemetry against compliance history.<br>**Why** — Detects violations in real time without manual policy review. |
| Compliance Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for compliance analysis |  | Compliance Agent | ## Compliance Check<br>**What** — Compliance Agent cross-references telemetry against compliance history.<br>**Why** — Detects violations in real time without manual policy review. |
| Compliance History Tool | n8n-nodes-base.dataTableTool | Retrieves historical compliance records |  | Compliance Agent | ## Compliance Check<br>**What** — Compliance Agent cross-references telemetry against compliance history.<br>**Why** — Detects violations in real time without manual policy review. |
| Reporting Agent | @n8n/n8n-nodes-langchain.agentTool | Generates operator-facing compliance reports | Reporting Model | Coordination Agent | ## Reporting & Notification<br>**What** — Reporting Agent generates summaries; Notification Agent triggers Slack alerts.<br>**Why** — Delivers actionable intelligence instantly to the right teams. |
| Reporting Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for report generation |  | Reporting Agent | ## Reporting & Notification<br>**What** — Reporting Agent generates summaries; Notification Agent triggers Slack alerts.<br>**Why** — Delivers actionable intelligence instantly to the right teams. |
| Notification Agent | @n8n/n8n-nodes-langchain.agentTool | Determines alert routing and notification text | Notification Model; Slack Notification Tool | Coordination Agent | ## Reporting & Notification<br>**What** — Reporting Agent generates summaries; Notification Agent triggers Slack alerts.<br>**Why** — Delivers actionable intelligence instantly to the right teams. |
| Notification Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for notification decisions |  | Notification Agent | ## Reporting & Notification<br>**What** — Reporting Agent generates summaries; Notification Agent triggers Slack alerts.<br>**Why** — Delivers actionable intelligence instantly to the right teams. |
| Slack Notification Tool | n8n-nodes-base.slackTool | Sends Slack alerts chosen by the notification agent |  | Notification Agent | ## Reporting & Notification<br>**What** — Reporting Agent generates summaries; Notification Agent triggers Slack alerts.<br>**Why** — Delivers actionable intelligence instantly to the right teams. |
| Prepare Telemetry Storage | n8n-nodes-base.set | Flattens agent output for storage and branching | Coordination Agent | Store Validated Telemetry; Prepare Compliance Alerts; Send Report Email | ## Store & Distribute<br>**What** — Stores validated telemetry, compliance alerts, and sends report emails.<br>**Why** — Creates persistent records for audit trails and regulatory reporting. |
| Store Validated Telemetry | n8n-nodes-base.dataTable | Stores validated telemetry rows | Prepare Telemetry Storage |  | ## Store & Distribute<br>**What** — Stores validated telemetry, compliance alerts, and sends report emails.<br>**Why** — Creates persistent records for audit trails and regulatory reporting. |
| Prepare Compliance Alerts | n8n-nodes-base.set | Builds compliance alert record | Prepare Telemetry Storage | Store Compliance Alerts | ## Store & Distribute<br>**What** — Stores validated telemetry, compliance alerts, and sends report emails.<br>**Why** — Creates persistent records for audit trails and regulatory reporting. |
| Store Compliance Alerts | n8n-nodes-base.dataTable | Stores compliance alert rows | Prepare Compliance Alerts |  | ## Store & Distribute<br>**What** — Stores validated telemetry, compliance alerts, and sends report emails.<br>**Why** — Creates persistent records for audit trails and regulatory reporting. |
| Send Report Email | n8n-nodes-base.emailSend | Sends compliance report by email | Prepare Telemetry Storage |  | ## Store & Distribute<br>**What** — Stores validated telemetry, compliance alerts, and sends report emails.<br>**Why** — Creates persistent records for audit trails and regulatory reporting. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation for prerequisites and benefits |  |  | ## Prerequisites<br>- Slack workspace and bot token<br>- Email account (SMTP or Gmail OAuth2)<br>- Database or Google Sheets for telemetry and alert storage<br>## Use Cases<br>- Real-time anomaly detection and alerting across smart grid sensor networks<br>- Automated regulatory compliance reporting for energy grid operators<br>## Customisation<br>- Extend Compliance Agent thresholds to match regional grid standards<br>- Replace Slack with Teams or PagerDuty for incident escalation<br>## Benefits<br>- Eliminates manual telemetry review — processes grid events at machine speed |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas setup checklist |  |  | ## Setup Steps<br>1. Configure webhook URL in **Grid Telemetry Webhook** node.<br>2. Set AI model credentials (OpenAI/Anthropic) in all agent and model nodes.<br>3. Connect Slack credentials and target channel to **Slack Notification Tool** node.<br>4. Configure email credentials in **Send Report Email** node.<br>5. Connect database/Google Sheets credentials. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas architecture explanation |  |  | ## How It Works<br>This workflow automates real-time energy grid telemetry ingestion, compliance validation, and multi-channel reporting for grid operators, energy managers, and compliance teams. Telemetry data arrives via webhook and is routed to a central Coordination Agent with persistent memory. Four specialised AI sub-agents operate in parallel: Grid Signal Agent (validates signals via Telemetry Validation Tool and parses structure), Compliance Agent (checks against compliance history), Reporting Agent (generates structured reports), and Notification Agent (triggers Slack alerts). Results flow into a Prepare Telemetry Storage node, then branch into three outputs, validated telemetry stored to a grid database, compliance alerts prepared and stored, and email reports dispatched. This eliminates manual grid monitoring, accelerates anomaly response, and maintains a continuous compliance audit trail across energy infrastructure. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas label for reporting/notification area |  |  | ## Reporting & Notification<br>**What** — Reporting Agent generates summaries; Notification Agent triggers Slack alerts.<br>**Why** — Delivers actionable intelligence instantly to the right teams. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas label for compliance area |  |  | ## Compliance Check<br>**What** — Compliance Agent cross-references telemetry against compliance history.<br>**Why** — Detects violations in real time without manual policy review. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas label for validation area |  |  | ## Signal Validation & Parsing<br>**What** — Grid Signal Agent validates telemetry and parses structure via dedicated tools.<br>**Why** — Ensures only clean, correctly structured data proceeds to analysis. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas label for storage/distribution area |  |  | ## Store & Distribute<br>**What** — Stores validated telemetry, compliance alerts, and sends report emails.<br>**Why** — Creates persistent records for audit trails and regulatory reporting. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Intelligent energy grid telemetry monitoring and compliance reporting`.
   - Keep execution order as default unless you need a specific custom order.

2. **Add the entry webhook**
   - Create a **Webhook** node named `Grid Telemetry Webhook`.
   - Set:
     - HTTP Method: `POST`
     - Path: `grid-telemetry`
   - Leave advanced options empty unless authentication or response customization is required.
   - Decide whether external systems will call the test URL or production URL.

3. **Add the central orchestration agent**
   - Create an **AI Agent** node named `Coordination Agent`.
   - Set prompt mode to manually defined text.
   - Set the input text to:
     - `={{ $json.body }}`
   - Add this system message:
     - The agent is the Grid Coordination Agent
     - It orchestrates compliance monitoring, reporting, and operator notifications
     - It coordinates four specialized sub-agents
     - It must never issue grid control commands
   - Connect `Grid Telemetry Webhook` → `Coordination Agent`.

4. **Attach the coordination language model**
   - Create an **OpenAI Chat Model** node named `Coordination Model`.
   - Select model `gpt-4o`.
   - Set temperature to `0.3`.
   - Add valid **OpenAI credentials**.
   - Connect it to the `Coordination Agent` as the **AI language model**.

5. **Attach memory to the coordination agent**
   - Create a **Memory Buffer Window** node named `Coordination Memory`.
   - Default settings are acceptable.
   - Connect it to `Coordination Agent` as **AI memory**.

6. **Create the validation sub-agent**
   - Add an **AI Agent Tool** node named `Grid Signal Agent`.
   - Set its input text to:
     - `={{ $fromAI('telemetry_data', 'Raw telemetry data to validate') }}`
   - Enable structured output parsing.
   - Set the system message to instruct validation of:
     - generation output, frequency, voltage
     - load consumption and power factor
     - storage charge level and charge/discharge rate
     - completeness, anomalies, quality, and operational ranges
   - Add a tool description indicating it returns validated telemetry as structured JSON.

7. **Attach the validation model**
   - Add an **OpenAI Chat Model** node named `Grid Signal Model`.
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Use the same OpenAI credentials or another suitable account.
   - Connect it to `Grid Signal Agent` as **AI language model**.

8. **Create the rule-based telemetry validation tool**
   - Add a **Code Tool** node named `Telemetry Validation Tool`.
   - Paste JavaScript implementing range checks for:
     - generation output `0–1000`
     - frequency `59.5–60.5`
     - load consumption `0–800`
     - storage charge `0–100`
   - Use:
     - `const telemetry = $fromAI('query');`
   - Return a JSON string including `validation_issues` and `is_valid`.
   - Connect this node to `Grid Signal Agent` as an **AI tool**.

9. **Create the structured output parser**
   - Add a **Structured Output Parser** node named `Telemetry Structure Parser`.
   - Use a manual JSON schema with:
     - `validation_status`
     - `generation.output_mw`
     - `generation.frequency_hz`
     - `generation.voltage_kv`
     - `generation.quality_score`
     - `load.consumption_mw`
     - `load.power_factor`
     - `load.quality_score`
     - `storage.charge_percent`
     - `storage.charge_rate_mw`
     - `storage.quality_score`
     - `anomalies`
     - `timestamp`
   - Connect it to `Grid Signal Agent` as **AI output parser**.

10. **Expose the validation agent to the coordinator**
    - Connect `Grid Signal Agent` to `Coordination Agent` as an **AI tool**.

11. **Create the compliance sub-agent**
    - Add an **AI Agent Tool** node named `Compliance Agent`.
    - Set input text to:
      - `={{ $fromAI('validated_telemetry', 'Validated telemetry data to check for compliance') }}`
    - Set the system message so it checks:
      - frequency `59.95–60.05 Hz`
      - voltage within ±5%
      - generation-load balance within 1%
      - storage response time under 4 seconds
      - historical compliance patterns
      - classifications: informational, warning, critical
    - Add a tool description explaining that it assesses NERC/FERC-style compliance.

12. **Attach the compliance model**
    - Add an **OpenAI Chat Model** node named `Compliance Model`.
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect it to `Compliance Agent` as **AI language model**.

13. **Create the compliance history lookup tool**
    - Add a **Data Table Tool** node named `Compliance History Tool`.
    - Set operation to `get`.
    - Select or create the compliance alerts data table first, then choose its ID here.
    - Replace the placeholder with the real table.
    - Connect it to `Compliance Agent` as an **AI tool**.

14. **Expose the compliance agent to the coordinator**
    - Connect `Compliance Agent` to `Coordination Agent` as an **AI tool**.

15. **Create the reporting sub-agent**
    - Add an **AI Agent Tool** node named `Reporting Agent`.
    - Set input text to:
      - `={{ $fromAI('compliance_data', 'Compliance assessment data to include in the report') }}`
    - Set the system message to generate:
      - executive summary
      - detailed findings
      - recommendations
      - violations, severity, trends, actionable insights
    - Add a tool description stating that it produces operator-ready compliance reports.

16. **Attach the reporting model**
    - Add an **OpenAI Chat Model** node named `Reporting Model`.
    - Use `gpt-5-mini` if available in your environment.
    - If unavailable, replace it with a supported model such as `gpt-4o`.
    - Connect it to `Reporting Agent` as **AI language model**.

17. **Expose the reporting agent to the coordinator**
    - Connect `Reporting Agent` to `Coordination Agent` as an **AI tool**.

18. **Create the notification sub-agent**
    - Add an **AI Agent Tool** node named `Notification Agent`.
    - Set input text to:
      - `={{ $fromAI('alert_data', 'Alert and compliance data to notify operators about') }}`
    - Configure the system message so that:
      - critical alerts go to Slack `#grid-ops-critical`
      - warning alerts go to Slack `#grid-ops-alerts`
      - informational items go to daily email only
      - messages should be clear and actionable
      - no grid control advice should be suggested
    - Add a tool description for real-time alert routing.

19. **Attach the notification model**
    - Add an **OpenAI Chat Model** node named `Notification Model`.
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Connect it to `Notification Agent` as **AI language model**.

20. **Create the Slack tool**
    - Add a **Slack Tool** node named `Slack Notification Tool`.
    - Configure:
      - Authentication: `OAuth2`
      - Send to: `Channel`
      - Message text:
        - `={{ $fromAI('message', 'Alert message to send to operators') }}`
      - Channel field:
        - `={{ $fromAI('channel', 'Slack channel ID for the alert', 'string', '#grid-ops-alerts') }}`
    - Add **Slack OAuth2 credentials** with posting permissions.
    - Make sure the bot is invited to the intended channels.
    - If your Slack node requires actual channel IDs, use IDs rather than names.
    - Connect it to `Notification Agent` as an **AI tool**.

21. **Expose the notification agent to the coordinator**
    - Connect `Notification Agent` to `Coordination Agent` as an **AI tool**.

22. **Add the post-processing Set node**
    - Create a **Set** node named `Prepare Telemetry Storage`.
    - Connect `Coordination Agent` → `Prepare Telemetry Storage`.
    - Add fields:
      1. `timestamp` as string: `={{ $now.toISO() }}`
      2. `validation_status` as string: `={{ $json.output.validation_status }}`
      3. `generation_output_mw` as number: `={{ $json.output.generation.output_mw }}`
      4. `generation_frequency_hz` as number: `={{ $json.output.generation.frequency_hz }}`
      5. `load_consumption_mw` as number: `={{ $json.output.load.consumption_mw }}`
      6. `storage_charge_percent` as number: `={{ $json.output.storage.charge_percent }}`
      7. `anomalies` as string: `={{ $json.output.anomalies.join(', ') }}`
   - Important: verify the Coordination Agent really returns data under `output`. If not, adapt expressions to the actual output shape.

23. **Create the validated telemetry storage table**
   - In n8n Data Tables, create a table for validated telemetry.
   - Add columns matching the flattened fields above.
   - Then add a **Data Table** node named `Store Validated Telemetry`.
   - Set it to auto-map input data.
   - Select the validated telemetry table.
   - Connect `Prepare Telemetry Storage` → `Store Validated Telemetry`.

24. **Create the compliance alert preparation node**
   - Add another **Set** node named `Prepare Compliance Alerts`.
   - Connect `Prepare Telemetry Storage` → `Prepare Compliance Alerts`.
   - Add fields:
     1. `timestamp` as string: `={{ $now.toISO() }}`
     2. `alert_severity` as string: `={{ $json.output.validation_status }}`
     3. `alert_type` as string: `compliance_check`
     4. `description` as string: `={{ $json.output.anomalies.join('; ') }}`
     5. `generation_mw` as number: `={{ $json.output.generation.output_mw }}`
     6. `frequency_hz` as number: `={{ $json.output.generation.frequency_hz }}`
   - Note: these expressions assume the original nested structure is still present. Because this node receives data from `Prepare Telemetry Storage`, you may need to adjust them if that node has already flattened or replaced the payload. In practice, this is one of the workflow’s likely points needing correction.

25. **Create the compliance alerts storage table**
   - Create a second Data Table for compliance alerts.
   - Add columns for:
     - timestamp
     - alert_severity
     - alert_type
     - description
     - generation_mw
     - frequency_hz
   - Add a **Data Table** node named `Store Compliance Alerts`.
   - Enable auto-mapping.
   - Select the compliance alerts table.
   - Connect `Prepare Compliance Alerts` → `Store Compliance Alerts`.

26. **Add the report email node**
   - Create an **Email Send** node named `Send Report Email`.
   - Connect `Prepare Telemetry Storage` → `Send Report Email`.
   - Configure:
     - Subject: `=Grid Compliance Report - {{ $now.format('yyyy-MM-dd HH:mm') }}`
     - To: operator email address
     - From: sender email address
     - Format: `text`
     - Body text: `={{ $json.output }}`
   - Add email credentials:
     - SMTP, Gmail OAuth2, or another supported provider
   - If the body becomes `[object Object]`, change the expression to stringify or format the report text explicitly.

27. **Create or verify required credentials**
   - **OpenAI credentials** for all model nodes
   - **Slack OAuth2 credentials** for the Slack tool
   - **Email credentials** for the email node
   - **Data Table access** in your n8n environment

28. **Replace all placeholders**
   - Replace:
     - `<__PLACEHOLDER_VALUE__validated_telemetry_table__>`
     - `<__PLACEHOLDER_VALUE__compliance_alerts_table__>`
     - `<__PLACEHOLDER_VALUE__operator_email__>`
     - `<__PLACEHOLDER_VALUE__sender_email__>`

29. **Add optional sticky notes for maintainability**
   - Add sticky notes for:
     - prerequisites
     - setup steps
     - compliance logic
     - reporting/notification
     - storage/distribution
   - This is optional for execution but useful for long-term maintenance.

30. **Test with a sample telemetry payload**
   - Send a POST request to the webhook with a realistic JSON body including:
     - generation output
     - frequency
     - voltage
     - load consumption
     - power factor
     - storage charge percent
     - charge rate
     - timestamp
   - Confirm:
     - the Coordination Agent can call sub-agents successfully
     - structured parser output is valid
     - Slack posts succeed
     - email sends correctly
     - rows appear in both data tables

31. **Harden the workflow before production**
   - Recommended improvements:
     - Add an initial **Set** or **Code** node to normalize webhook input into a consistent string/object
     - Add **IF** or **Code** nodes after the agent output to validate required fields before storage
     - Serialize report content explicitly for email
     - Use deterministic channel IDs in Slack
     - Separate validation severity from compliance severity
     - Consider adding error workflows or Error Trigger handling

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Slack workspace and bot token are required. | Prerequisite |
| Email account via SMTP or Gmail OAuth2 is required. | Prerequisite |
| Database or Google Sheets is mentioned as storage in notes, but the actual workflow uses n8n Data Tables. | Implementation note |
| Real-time anomaly detection and alerting across smart grid sensor networks. | Use case |
| Automated regulatory compliance reporting for energy grid operators. | Use case |
| Extend Compliance Agent thresholds to match regional grid standards. | Customization |
| Replace Slack with Microsoft Teams or PagerDuty for incident escalation. | Customization |
| Eliminates manual telemetry review and processes grid events at machine speed. | Claimed benefit |
| Setup checklist: configure webhook URL, AI credentials, Slack credentials, email credentials, and storage credentials. | Operational note |
| The workflow note claims the four specialized AI sub-agents operate in parallel, but in practice this is agent tool orchestration inside the Coordination Agent, not explicit n8n parallel branches. | Architecture clarification |
| The workflow is titled in the prompt as “Monitor grid telemetry and automate compliance alerts with GPT-4o and Slack,” while the JSON workflow name is “Intelligent energy grid telemetry monitoring and compliance reporting.” | Naming note |

## Important implementation observations

- The workflow depends heavily on AI agents returning a predictable structure, but only the Grid Signal Agent has an explicit structured parser.
- The `Prepare Telemetry Storage` and `Prepare Compliance Alerts` nodes both reference `{{$json.output...}}`. This should be tested carefully because agent output formats in n8n can differ.
- `Send Report Email` uses `{{$json.output}}` as plain text, which may not render well if the value is an object.
- Slack channel selection may need real Slack channel IDs rather than names beginning with `#`.
- The compliance severity currently appears to be derived from validation status rather than a dedicated compliance result, which may not reflect true regulatory severity.