Triage fleet telemetry and route safety compliance with GPT-4o, Gmail and Sheets

https://n8nworkflows.xyz/workflows/triage-fleet-telemetry-and-route-safety-compliance-with-gpt-4o--gmail-and-sheets-14425


# Triage fleet telemetry and route safety compliance with GPT-4o, Gmail and Sheets

# 1. Workflow Overview

This workflow processes incoming vehicle telemetry, uses GPT-4o-based AI agents to validate faults and determine operational actions, then routes the case into service handling and compliance logging paths. Its purpose is to help fleet operators triage telemetry alerts faster, prioritize service actions, notify stakeholders, and keep auditable records without issuing any vehicle control commands.

Typical use cases include:

- Logistics and transport fleets receiving frequent telemetry fault events
- Safety compliance review for potentially reportable incidents
- Automated prioritization of urgent versus routine maintenance actions
- Customer and fleet-operations notification workflows
- Traceability logging into Google Sheets for audits

## 1.1 Input Reception

The workflow starts with a webhook that receives POST telemetry payloads from an external system.

## 1.2 Telemetry Validation

An AI validation agent analyzes the raw payload, checks integrity, classifies severity, identifies anomalies, and returns structured output through a schema-enforced parser.

## 1.3 Fleet Orchestration

A second AI agent receives the validated telemetry and coordinates downstream actions. It can use specialist tools and sub-agents for safety compliance analysis, service scheduling, notifications, alerts, and API lookups.

## 1.4 Service Priority Routing

The orchestration result is routed by `servicePriority` into one of four branches: urgent, high, normal, or low. Each branch prepares or triggers service actions differently.

## 1.5 Compliance Escalation

In parallel to service routing, the workflow checks whether compliance escalation is required. If true, it prepares a compliance record and logs it.

## 1.6 Logging and Webhook Response

Service traceability and compliance events are appended to Google Sheets. The workflow then returns a JSON response to the original webhook caller summarizing the result.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block exposes the workflow as an HTTP endpoint for telemetry ingestion. It accepts POST requests and defers the HTTP response until the final response node runs.

### Nodes Involved
- Receive Vehicle Telemetry

### Node Details

#### Receive Vehicle Telemetry
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry-point trigger node for inbound telemetry events.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `fleet-telemetry`
  - Response mode: `responseNode`, meaning the request stays open until a `Respond to Webhook` node completes it
- **Key expressions or variables used:**
  - Downstream nodes access payload via `{{ $json.body }}`
- **Input and output connections:**
  - Input: none, this is a trigger
  - Output: `Telemetry Validation Agent`
- **Version-specific requirements:**
  - Uses webhook node version `2.1`
- **Edge cases or potential failure types:**
  - Wrong HTTP method returns method-related failure
  - External caller timeout if downstream AI processing takes too long
  - Missing or malformed JSON body may cause poor validation-agent output
- **Sub-workflow reference:** none

---

## 2.2 Telemetry Validation

### Overview
This block validates incoming telemetry before any orchestration occurs. It uses GPT-4o plus a structured output parser so downstream steps receive predictable fields such as fault severity, anomalies, and safety-critical status.

### Nodes Involved
- Telemetry Validation Agent
- Validation Agent Model
- Validation Output Parser

### Node Details

#### Telemetry Validation Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main AI agent for interpreting and validating telemetry content.
- **Configuration choices:**
  - Prompt mode: manually defined
  - Input text: `{{ $json.body }}`
  - System message instructs the model to:
    - validate data integrity
    - classify severity into `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`
    - identify safety-critical issues
    - flag anomalies
    - avoid control commands
    - return structured validation results
  - Structured parser enabled
- **Key expressions or variables used:**
  - `{{ $json.body }}`
- **Input and output connections:**
  - Main input: `Receive Vehicle Telemetry`
  - AI language model input: `Validation Agent Model`
  - AI output parser input: `Validation Output Parser`
  - Main output: `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Agent version `3.1`
- **Edge cases or potential failure types:**
  - If payload structure is inconsistent, model may hallucinate missing fields
  - Parser failures if the model output does not comply with schema
  - OpenAI auth/rate-limit issues
- **Sub-workflow reference:** none

#### Validation Agent Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o chat model backing the validation agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for more deterministic classification
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_languageModel` to `Telemetry Validation Agent`
- **Version-specific requirements:**
  - Node version `1.3`
  - Requires OpenAI credential configured
- **Edge cases or potential failure types:**
  - Invalid API key
  - quota exhaustion
  - API latency
- **Sub-workflow reference:** none

#### Validation Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces schema-valid AI output.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `validationStatus`: `VALID | INVALID | ANOMALY`
    - `faultSeverity`: `CRITICAL | HIGH | MEDIUM | LOW | NONE`
    - `safetyCritical`: boolean
    - `faultCodes`: array of strings
    - `healthScore`: number
    - `recommendedAction`: string
    - `anomalies`: array of strings
  - Required fields:
    - `validationStatus`
    - `faultSeverity`
    - `safetyCritical`
    - `recommendedAction`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_outputParser` to `Telemetry Validation Agent`
- **Version-specific requirements:**
  - Node version `1.3`
- **Edge cases or potential failure types:**
  - Model output may omit required fields
  - Type mismatch, such as string instead of boolean
- **Sub-workflow reference:** none

---

## 2.3 Fleet Orchestration

### Overview
This is the decision-making core of the workflow. It receives the validated telemetry result and uses GPT-4o with multiple tools and specialist sub-agents to decide service action, priority, notifications, and compliance escalation.

### Nodes Involved
- Fleet Orchestration Agent
- Orchestration Agent Model
- Orchestration Output Parser
- Safety Threshold Calculator
- Safety Compliance Sub-Agent
- Safety Compliance Model
- Service Scheduling Sub-Agent
- Service Scheduling Model
- Customer Email Notification Tool
- Fleet Operations Alert Tool
- Fleet Management API Tool

### Node Details

#### Fleet Orchestration Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main orchestration AI agent coordinating downstream actions.
- **Configuration choices:**
  - Input text: `{{ $json }}`
  - System message defines responsibilities:
    - coordinate service actions
    - customer communications
    - compliance escalation
    - use specialist sub-agents
    - avoid real-time vehicle control
    - maintain traceability
  - Structured output parser enabled
- **Key expressions or variables used:**
  - `{{ $json }}`
- **Input and output connections:**
  - Main input: `Telemetry Validation Agent`
  - AI language model: `Orchestration Agent Model`
  - AI output parser: `Orchestration Output Parser`
  - AI tools:
    - `Safety Threshold Calculator`
    - `Safety Compliance Sub-Agent`
    - `Service Scheduling Sub-Agent`
    - `Customer Email Notification Tool`
    - `Fleet Operations Alert Tool`
    - `Fleet Management API Tool`
  - Main outputs:
    - `Route by Service Priority`
    - `Check Compliance Escalation`
- **Version-specific requirements:**
  - Agent version `3.1`
- **Edge cases or potential failure types:**
  - Tool-call failures can affect final output quality
  - If validated input lacks expected fields, the agent may make weak decisions
  - Notification tool usage may happen before downstream logging, so failures could produce partial execution
- **Sub-workflow reference:** none

#### Orchestration Agent Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_languageModel` to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `1.3`
- **Edge cases or potential failure types:**
  - OpenAI auth or quota issues
  - Slower responses may delay webhook completion
- **Sub-workflow reference:** none

#### Orchestration Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`
- **Configuration choices:**
  - Manual schema with fields:
    - `serviceAction`: `IMMEDIATE_SERVICE | SCHEDULED_SERVICE | MONITOR | NO_ACTION`
    - `servicePriority`: `URGENT | HIGH | NORMAL | LOW`
    - `customerNotification`: object with `required`, `channel`, `message`
    - `complianceEscalation`: boolean
    - `complianceDetails`: string
    - `scheduledDate`: string
  - Required:
    - `serviceAction`
    - `servicePriority`
    - `customerNotification`
    - `complianceEscalation`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_outputParser` to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `1.3`
- **Edge cases or potential failure types:**
  - The model may fail to populate nested `customerNotification` correctly
  - Missing `scheduledDate` is allowed by schema but may affect service-preparation nodes
- **Sub-workflow reference:** none

#### Safety Threshold Calculator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`  
  AI-callable code tool for risk scoring.
- **Configuration choices:**
  - Reads tool arguments from AI using:
    - `faultSeverity`
    - `healthScore`
    - `faultCodes`
  - Calculates:
    - `riskScore = 100 - healthScore`
    - adds severity penalties:
      - `+50` for `CRITICAL`
      - `+30` for `HIGH`
      - `+15` for `MEDIUM`
    - `exceedsThreshold` if risk score > 30
    - `requiresImmediateAction` if risk score > 50
  - Returns `riskScore`, threshold flags, and threshold value
- **Key expressions or variables used:**
  - `$fromAI("faultSeverity", ...)`
  - `$fromAI("healthScore", ...)`
  - `$fromAI("faultCodes", ...)`
- **Input and output connections:**
  - AI tool output to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `1.3`
- **Edge cases or potential failure types:**
  - If `healthScore` is missing or not numeric, code may produce invalid math
  - `faultCodes` is collected but not currently used
  - Threshold naming is slightly confusing: `criticalThreshold = 30`, `highThreshold = 50`
- **Sub-workflow reference:** none

#### Safety Compliance Sub-Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  AI sub-agent exposed as a tool to the orchestration agent.
- **Configuration choices:**
  - Text input from AI:
    - `{{ $fromAI('telemetryData', 'The validated telemetry and fault data to analyze for compliance') }}`
  - System prompt focuses on:
    - safety compliance violations
    - regulatory reporting
    - recall assessment
    - safety bulletins
    - traceability requirements
- **Key expressions or variables used:**
  - `$fromAI('telemetryData', ...)`
- **Input and output connections:**
  - AI language model: `Safety Compliance Model`
  - AI tool output to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `3`
- **Edge cases or potential failure types:**
  - Free-form output may be difficult for the orchestrator to interpret consistently
  - Compliance recommendations depend heavily on prompt quality and telemetry completeness
- **Sub-workflow reference:** none

#### Safety Compliance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_languageModel` to `Safety Compliance Sub-Agent`
- **Version-specific requirements:**
  - Node version `1.3`
- **Edge cases or potential failure types:**
  - OpenAI authentication and quota failures
- **Sub-workflow reference:** none

#### Service Scheduling Sub-Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`
- **Configuration choices:**
  - Text input:
    - `{{ $fromAI('serviceRequest', 'The service action request with fault details and priority level') }}`
  - System prompt focuses on:
    - timing
    - prioritization
    - capacity
    - parts availability
    - operational impact
  - Tool description clarifies it determines maintenance planning strategy
- **Key expressions or variables used:**
  - `$fromAI('serviceRequest', ...)`
- **Input and output connections:**
  - AI language model: `Service Scheduling Model`
  - AI tool output to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `3`
- **Edge cases or potential failure types:**
  - No structured parser attached, so output remains unconstrained
  - Scheduling quality depends on data richness
- **Sub-workflow reference:** none

#### Service Scheduling Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Output via `ai_languageModel` to `Service Scheduling Sub-Agent`
- **Version-specific requirements:**
  - Node version `1.3`
- **Edge cases or potential failure types:**
  - OpenAI failures or latency
- **Sub-workflow reference:** none

#### Customer Email Notification Tool
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  AI-callable Gmail sending tool.
- **Configuration choices:**
  - Recipient: `{{ $fromAI('recipientEmail', 'Customer email address', 'string') }}`
  - Subject: `{{ $fromAI('emailSubject', 'Email subject line', 'string') }}`
  - Message body: `{{ $fromAI('emailBody', 'Email message content', 'string') }}`
  - Manual tool description explains usage
- **Key expressions or variables used:**
  - `$fromAI('recipientEmail', ...)`
  - `$fromAI('emailSubject', ...)`
  - `$fromAI('emailBody', ...)`
- **Input and output connections:**
  - AI tool output to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `2.2`
  - Requires Gmail OAuth2 credential
- **Edge cases or potential failure types:**
  - Gmail auth failure
  - invalid email format
  - sending restrictions/quota
  - AI may invoke without valid recipient data
- **Sub-workflow reference:** none

#### Fleet Operations Alert Tool
- **Type and technical role:** `n8n-nodes-base.slackTool`  
  AI-callable Slack alerting tool.
- **Configuration choices:**
  - Message text: `{{ $fromAI('alertMessage', 'Alert message content', 'string') }}`
  - Send to channel by ID/name from AI:
    - `{{ $fromAI('slackChannel', 'Slack channel ID or name', 'string') }}`
  - Authentication: OAuth2
  - Manual tool description explains urgent fleet alerts
- **Key expressions or variables used:**
  - `$fromAI('alertMessage', ...)`
  - `$fromAI('slackChannel', ...)`
- **Input and output connections:**
  - AI tool output to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `2.4`
  - Requires Slack OAuth2 credential
- **Edge cases or potential failure types:**
  - Wrong channel ID/name
  - missing bot permissions
  - auth failure
- **Sub-workflow reference:** none

#### Fleet Management API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`  
  AI-callable HTTP tool for external systems.
- **Configuration choices:**
  - URL from AI:
    - `{{ $fromAI('apiUrl', 'The API endpoint URL to call', 'string') }}`
  - Method from AI with default `GET`:
    - `{{ $fromAI('httpMethod', 'HTTP method (GET, POST, PUT, DELETE)', 'string', 'GET') }}`
  - No extra options configured
  - Tool description references fleet history, service records, and compliance databases
- **Key expressions or variables used:**
  - `$fromAI('apiUrl', ...)`
  - `$fromAI('httpMethod', ...)`
- **Input and output connections:**
  - AI tool output to `Fleet Orchestration Agent`
- **Version-specific requirements:**
  - Node version `4.4`
- **Edge cases or potential failure types:**
  - AI-selected URL may be invalid or unsafe if not constrained
  - authentication requirements for the target API are not configured here
  - network errors and timeout failures
- **Sub-workflow reference:** none

---

## 2.4 Service Priority Routing

### Overview
This block routes orchestration results into execution paths based on `servicePriority`. Urgent cases call an external API immediately; other priorities are normalized into structured records before logging.

### Nodes Involved
- Route by Service Priority
- Urgent Service API Call
- Prepare High Priority Service
- Prepare Normal Service
- Prepare Low Priority Service

### Node Details

#### Route by Service Priority
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branches flow according to orchestration output.
- **Configuration choices:**
  - Evaluates `{{ $json.servicePriority }}`
  - Four explicit rules:
    - `URGENT`
    - `HIGH`
    - `NORMAL`
    - `LOW`
- **Key expressions or variables used:**
  - `{{ $json.servicePriority }}`
- **Input and output connections:**
  - Input: `Fleet Orchestration Agent`
  - Outputs:
    - Output 0: `Urgent Service API Call`
    - Output 1: `Prepare High Priority Service`
    - Output 2: `Prepare Normal Service`
    - Output 3: `Prepare Low Priority Service`
- **Version-specific requirements:**
  - Switch version `3.4`
- **Edge cases or potential failure types:**
  - If `servicePriority` is absent or outside allowed values, no branch may execute
  - Case sensitivity is enabled in rule comparisons
- **Sub-workflow reference:** none

#### Urgent Service API Call
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls an external urgent-service endpoint.
- **Configuration choices:**
  - URL is a placeholder and must be replaced
  - Method: `POST`
  - Timeout: 30 seconds
  - Sends body fields:
    - `serviceAction` from `{{ $json.output.serviceAction }}`
    - `servicePriority` from `{{ $json.output.servicePriority }}`
    - `scheduledDate` from `{{ $json.output.scheduledDate }}`
    - `complianceEscalation` from `{{ $json.output.complianceEscalation }}`
- **Key expressions or variables used:**
  - `{{ $json.output.serviceAction }}`
  - `{{ $json.output.servicePriority }}`
  - `{{ $json.output.scheduledDate }}`
  - `{{ $json.output.complianceEscalation }}`
- **Input and output connections:**
  - Input: `Route by Service Priority`
  - Output: `Log Safety Traceability`
- **Version-specific requirements:**
  - HTTP Request version `4.4`
- **Edge cases or potential failure types:**
  - The expressions expect `output.*`; if the orchestration node emits flattened JSON rather than nested `output`, this can fail
  - Placeholder URL must be replaced
  - timeout or non-2xx API response
- **Sub-workflow reference:** none

#### Prepare High Priority Service
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Creates normalized fields:
    - `priority = HIGH`
    - `serviceType = {{ $json.serviceAction }}`
    - `scheduledDate = {{ $json.scheduledDate }}`
    - `notificationSent = {{ $json.customerNotification.required }}`
    - `timestamp = {{ $now.toISO() }}`
- **Key expressions or variables used:**
  - `{{ $json.serviceAction }}`
  - `{{ $json.scheduledDate }}`
  - `{{ $json.customerNotification.required }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**
  - Input: `Route by Service Priority`
  - Output: `Log Safety Traceability`
- **Version-specific requirements:**
  - Set version `3.4`
- **Edge cases or potential failure types:**
  - Missing nested `customerNotification.required`
- **Sub-workflow reference:** none

#### Prepare Normal Service
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Same structure as high-priority branch, but `priority = NORMAL`
- **Key expressions or variables used:**
  - same as above
- **Input and output connections:**
  - Input: `Route by Service Priority`
  - Output: `Log Safety Traceability`
- **Version-specific requirements:**
  - Set version `3.4`
- **Edge cases or potential failure types:**
  - Missing scheduling or notification fields
- **Sub-workflow reference:** none

#### Prepare Low Priority Service
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Same structure as high-priority branch, but `priority = LOW`
- **Key expressions or variables used:**
  - same as above
- **Input and output connections:**
  - Input: `Route by Service Priority`
  - Output: `Log Safety Traceability`
- **Version-specific requirements:**
  - Set version `3.4`
- **Edge cases or potential failure types:**
  - Missing fields or null date values
- **Sub-workflow reference:** none

---

## 2.5 Compliance Escalation

### Overview
This branch checks whether the orchestration agent determined a compliance escalation is required. If yes, it constructs a compliance record by combining orchestration output with validation results and original webhook data.

### Nodes Involved
- Check Compliance Escalation
- Prepare Compliance Report

### Node Details

#### Check Compliance Escalation
- **Type and technical role:** `n8n-nodes-base.if`
- **Configuration choices:**
  - Condition: boolean true on `{{ $json.complianceEscalation }}`
  - Loose validation, case-insensitive settings enabled in node options
- **Key expressions or variables used:**
  - `{{ $json.complianceEscalation }}`
- **Input and output connections:**
  - Input: `Fleet Orchestration Agent`
  - True output: `Prepare Compliance Report`
  - False output: no connection configured
- **Version-specific requirements:**
  - If node version `2.3`
- **Edge cases or potential failure types:**
  - If field is missing, false branch likely results in no downstream compliance record
  - Unconnected false branch is acceptable, but means no explicit no-op handling
- **Sub-workflow reference:** none

#### Prepare Compliance Report
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Builds compliance log fields:
    - `reportType = COMPLIANCE_ESCALATION`
    - `complianceDetails = {{ $json.complianceDetails }}`
    - `faultSeverity = {{ $('Telemetry Validation Agent').item.json.output.faultSeverity }}`
    - `safetyCritical = {{ $('Telemetry Validation Agent').item.json.output.safetyCritical }}`
    - `timestamp = {{ $now.toISO() }}`
    - `vehicleId = {{ $('Receive Vehicle Telemetry').item.json.body.vehicleId }}`
- **Key expressions or variables used:**
  - `{{ $json.complianceDetails }}`
  - `{{ $('Telemetry Validation Agent').item.json.output.faultSeverity }}`
  - `{{ $('Telemetry Validation Agent').item.json.output.safetyCritical }}`
  - `{{ $('Receive Vehicle Telemetry').item.json.body.vehicleId }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**
  - Input: `Check Compliance Escalation`
  - Output: `Log Compliance Escalation`
- **Version-specific requirements:**
  - Set version `3.4`
- **Edge cases or potential failure types:**
  - Cross-node expressions assume the validation agent output is nested under `.output`
  - Missing `vehicleId` in webhook body
- **Sub-workflow reference:** none

---

## 2.6 Logging and Final Response

### Overview
This block writes service and compliance traceability events to Google Sheets, then returns a consistent response to the webhook caller.

### Nodes Involved
- Log Safety Traceability
- Log Compliance Escalation
- Send Webhook Response

### Node Details

#### Log Safety Traceability
- **Type and technical role:** `n8n-nodes-base.googleSheets`
- **Configuration choices:**
  - Operation: `append`
  - Google Sheets document ID: not yet selected
  - Sheet name: not yet selected
- **Key expressions or variables used:** none explicitly configured in JSON
- **Input and output connections:**
  - Inputs:
    - `Urgent Service API Call`
    - `Prepare High Priority Service`
    - `Prepare Normal Service`
    - `Prepare Low Priority Service`
  - Output: `Send Webhook Response`
- **Version-specific requirements:**
  - Google Sheets node version `4.7`
  - Requires Google Sheets OAuth2 credential
- **Edge cases or potential failure types:**
  - Document ID and sheet name are blank, so the node is incomplete until configured
  - Schema mismatch between incoming item fields and target columns may lead to unexpected append behavior
- **Sub-workflow reference:** none

#### Log Compliance Escalation
- **Type and technical role:** `n8n-nodes-base.googleSheets`
- **Configuration choices:**
  - Operation: `append`
  - Google Sheets document ID: blank
  - Sheet name: blank
- **Key expressions or variables used:** none explicitly configured
- **Input and output connections:**
  - Input: `Prepare Compliance Report`
  - Output: `Send Webhook Response`
- **Version-specific requirements:**
  - Google Sheets version `4.7`
  - Requires Google Sheets OAuth2 credential
- **Edge cases or potential failure types:**
  - Incomplete sheet configuration
  - Different incoming schema than service log branch
- **Sub-workflow reference:** none

#### Send Webhook Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`
- **Configuration choices:**
  - Respond with JSON
  - Response body includes:
    - `status = processed`
    - validation status from validation agent
    - fault severity from validation agent
    - service action from orchestration agent
    - service priority from orchestration agent
    - compliance escalation from orchestration agent
    - `traceabilityLogged = true`
    - message confirming successful processing and no control commands issued
- **Key expressions or variables used:**
  - `{{ $('Telemetry Validation Agent').item.json.output.validationStatus }}`
  - `{{ $('Telemetry Validation Agent').item.json.output.faultSeverity }}`
  - `{{ $('Fleet Orchestration Agent').item.json.output.serviceAction }}`
  - `{{ $('Fleet Orchestration Agent').item.json.output.servicePriority }}`
  - `{{ $('Fleet Orchestration Agent').item.json.output.complianceEscalation }}`
- **Input and output connections:**
  - Inputs:
    - `Log Safety Traceability`
    - `Log Compliance Escalation`
  - Output: HTTP response to original webhook
- **Version-specific requirements:**
  - Respond to Webhook version `1.1`
- **Edge cases or potential failure types:**
  - If neither logging path reaches this node, the webhook may remain unanswered
  - If both branches reach it in one execution, response timing and duplicate execution behavior should be tested carefully
  - Cross-node expressions again assume `output.*` nesting
- **Sub-workflow reference:** none

---

## 2.7 Documentation and In-Canvas Notes

### Overview
These nodes are informational and do not execute logic. They document setup steps, rationale, use cases, and operational benefits directly on the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Contains prerequisites, use cases, customization, and benefits
- **Input and output connections:** none
- **Version-specific requirements:** version `1`
- **Edge cases or potential failure types:** none
- **Sub-workflow reference:** none

#### Sticky Note1
- Same technical nature as above; contains setup checklist.

#### Sticky Note2
- Same technical nature as above; contains end-to-end workflow explanation.

#### Sticky Note3
- Same technical nature as above; describes safety compliance and service scheduling sub-agents.

#### Sticky Note4
- Same technical nature as above; describes the fleet orchestration agent.

#### Sticky Note5
- Same technical nature as above; describes the telemetry validation agent.

#### Sticky Note6
- Same technical nature as above; describes routing by service priority.

#### Sticky Note7
- Same technical nature as above; describes logging and response behavior.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Vehicle Telemetry | Webhook | Receives POST telemetry payloads |  | Telemetry Validation Agent | ## How It Works<br>This workflow automates intelligent fleet operations management for transport operators, logistics companies, and smart mobility teams. It solves the problem of manually triaging high-volume vehicle telemetry data, a process prone to delays, missed safety thresholds, and inconsistent service prioritisation. Incoming vehicle telemetry is received via webhook and validated by a Telemetry Validation Agent using an AI model and output parser. Validated data is passed to a Fleet Orchestration Agent that coordinates three specialist sub-agents: a Safety Compliance Sub-Agent (checking thresholds and escalating breaches), a Service Scheduling Sub-Agent (optimising maintenance windows), and a Customer Email Notification Tool. The orchestrator then routes each case by service priority, namely: urgent, high, normal, or low, triggering the appropriate service preparation step and logging traceability data to Google Sheets. Compliance escalations follow a parallel path: checking status, preparing reports, and logging to Sheets. All branches converge into a unified webhook response, ensuring downstream systems receive a consistent, structured reply.<br>## Telemetry Validation Agent<br>**Why** — AI validates data integrity before orchestration to prevent bad data propagation. |
| Telemetry Validation Agent | LangChain Agent | Validates telemetry and classifies faults | Receive Vehicle Telemetry | Fleet Orchestration Agent | ## How It Works<br>This workflow automates intelligent fleet operations management for transport operators, logistics companies, and smart mobility teams. It solves the problem of manually triaging high-volume vehicle telemetry data, a process prone to delays, missed safety thresholds, and inconsistent service prioritisation. Incoming vehicle telemetry is received via webhook and validated by a Telemetry Validation Agent using an AI model and output parser. Validated data is passed to a Fleet Orchestration Agent that coordinates three specialist sub-agents: a Safety Compliance Sub-Agent (checking thresholds and escalating breaches), a Service Scheduling Sub-Agent (optimising maintenance windows), and a Customer Email Notification Tool. The orchestrator then routes each case by service priority, namely: urgent, high, normal, or low, triggering the appropriate service preparation step and logging traceability data to Google Sheets. Compliance escalations follow a parallel path: checking status, preparing reports, and logging to Sheets. All branches converge into a unified webhook response, ensuring downstream systems receive a consistent, structured reply.<br>## Telemetry Validation Agent<br>**Why** — AI validates data integrity before orchestration to prevent bad data propagation. |
| Validation Agent Model | OpenAI Chat Model | LLM backing the validation agent |  | Telemetry Validation Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Telemetry Validation Agent<br>**Why** — AI validates data integrity before orchestration to prevent bad data propagation. |
| Validation Output Parser | Structured Output Parser | Enforces validation schema |  | Telemetry Validation Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Telemetry Validation Agent<br>**Why** — AI validates data integrity before orchestration to prevent bad data propagation. |
| Fleet Orchestration Agent | LangChain Agent | Coordinates actions, notifications, and compliance | Telemetry Validation Agent | Route by Service Priority; Check Compliance Escalation | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Gmail account with OAuth credentials<br>- Google Sheets with log tabs pre-created<br>## Use Cases<br>- Logistics fleets auto-triaging vehicle fault alerts by severity<br>## Customisation<br>- Swap OpenAI for any LangChain-compatible model<br>## Benefits<br>- Eliminates manual telemetry triage, reducing response lag<br>## How It Works<br>This workflow automates intelligent fleet operations management for transport operators, logistics companies, and smart mobility teams. It solves the problem of manually triaging high-volume vehicle telemetry data, a process prone to delays, missed safety thresholds, and inconsistent service prioritisation. Incoming vehicle telemetry is received via webhook and validated by a Telemetry Validation Agent using an AI model and output parser. Validated data is passed to a Fleet Orchestration Agent that coordinates three specialist sub-agents: a Safety Compliance Sub-Agent (checking thresholds and escalating breaches), a Service Scheduling Sub-Agent (optimising maintenance windows), and a Customer Email Notification Tool. The orchestrator then routes each case by service priority, namely: urgent, high, normal, or low, triggering the appropriate service preparation step and logging traceability data to Google Sheets. Compliance escalations follow a parallel path: checking status, preparing reports, and logging to Sheets. All branches converge into a unified webhook response, ensuring downstream systems receive a consistent, structured reply.<br>## Fleet Orchestration Agent<br>**Why** — Central coordinator that delegates tasks to specialised sub-agents for parallel processing. |
| Orchestration Agent Model | OpenAI Chat Model | LLM backing the orchestration agent |  | Fleet Orchestration Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Orchestration Output Parser | Structured Output Parser | Enforces orchestration schema |  | Fleet Orchestration Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Safety Threshold Calculator | LangChain Code Tool | Calculates risk and threshold flags |  | Fleet Orchestration Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Safety Compliance Sub-Agent | LangChain Agent Tool | Specialist compliance analysis tool |  | Fleet Orchestration Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Safety Compliance Model | OpenAI Chat Model | LLM for compliance sub-agent |  | Safety Compliance Sub-Agent | ## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Service Scheduling Sub-Agent | LangChain Agent Tool | Specialist maintenance scheduling tool |  | Fleet Orchestration Agent | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node.<br>## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Service Scheduling Model | OpenAI Chat Model | LLM for scheduling sub-agent |  | Service Scheduling Sub-Agent | ## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Customer Email Notification Tool | Gmail Tool | Sends customer email notices |  | Fleet Orchestration Agent | ## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Fleet Operations Alert Tool | Slack Tool | Sends urgent fleet alerts to Slack |  | Fleet Orchestration Agent | ## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Fleet Management API Tool | HTTP Request Tool | Calls external fleet systems as an AI tool |  | Fleet Orchestration Agent | ## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Route by Service Priority | Switch | Branches by urgency level | Fleet Orchestration Agent | Urgent Service API Call; Prepare High Priority Service; Prepare Normal Service; Prepare Low Priority Service | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention. |
| Urgent Service API Call | HTTP Request | Triggers immediate urgent-service API | Route by Service Priority | Log Safety Traceability | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention.<br>## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Prepare High Priority Service | Set | Normalizes high-priority service record | Route by Service Priority | Log Safety Traceability | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention.<br>## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Prepare Normal Service | Set | Normalizes normal-priority service record | Route by Service Priority | Log Safety Traceability | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention.<br>## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Prepare Low Priority Service | Set | Normalizes low-priority service record | Route by Service Priority | Log Safety Traceability | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention.<br>## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Check Compliance Escalation | If | Checks whether escalation is required | Fleet Orchestration Agent | Prepare Compliance Report | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention. |
| Prepare Compliance Report | Set | Builds compliance log payload | Check Compliance Escalation | Log Compliance Escalation | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention.<br>## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Log Safety Traceability | Google Sheets | Appends service traceability records | Urgent Service API Call; Prepare High Priority Service; Prepare Normal Service; Prepare Low Priority Service | Send Webhook Response | ## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Log Compliance Escalation | Google Sheets | Appends compliance escalation records | Prepare Compliance Report | Send Webhook Response | ## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Send Webhook Response | Respond to Webhook | Returns final JSON response to caller | Log Safety Traceability; Log Compliance Escalation |  | ## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |
| Sticky Note | Sticky Note | Canvas documentation |  |  | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Gmail account with OAuth credentials<br>- Google Sheets with log tabs pre-created<br>## Use Cases<br>- Logistics fleets auto-triaging vehicle fault alerts by severity<br>## Customisation<br>- Swap OpenAI for any LangChain-compatible model<br>## Benefits<br>- Eliminates manual telemetry triage, reducing response lag |
| Sticky Note1 | Sticky Note | Canvas setup instructions |  |  | ## Setup Steps<br>1. Import workflow and configure the webhook trigger URL.<br>2. Add AI model credentials to Validation Agent, Orchestration Agent, and both Sub-Agents.<br>3. Connect Gmail credentials to the Customer Email Notification Tool.<br>4. Link Google Sheets credentials; set target sheet IDs for Safety Traceability and Compliance Escalation logs.<br>5. Configure the Fleet Management API Tool and Urgent Service API Call with your fleet service endpoint URLs.<br>6. Set safety threshold values in the Safety Threshold Calculator node. |
| Sticky Note2 | Sticky Note | Canvas functional overview |  |  | ## How It Works<br>This workflow automates intelligent fleet operations management for transport operators, logistics companies, and smart mobility teams. It solves the problem of manually triaging high-volume vehicle telemetry data, a process prone to delays, missed safety thresholds, and inconsistent service prioritisation. Incoming vehicle telemetry is received via webhook and validated by a Telemetry Validation Agent using an AI model and output parser. Validated data is passed to a Fleet Orchestration Agent that coordinates three specialist sub-agents: a Safety Compliance Sub-Agent (checking thresholds and escalating breaches), a Service Scheduling Sub-Agent (optimising maintenance windows), and a Customer Email Notification Tool. The orchestrator then routes each case by service priority, namely: urgent, high, normal, or low, triggering the appropriate service preparation step and logging traceability data to Google Sheets. Compliance escalations follow a parallel path: checking status, preparing reports, and logging to Sheets. All branches converge into a unified webhook response, ensuring downstream systems receive a consistent, structured reply. |
| Sticky Note3 | Sticky Note | Canvas rationale for sub-agents |  |  | ## Safety Compliance & Service Scheduling Sub-Agents<br>**Why** — Detects threshold breaches to trigger escalation and optimises maintenance scheduling to minimise fleet downtime. |
| Sticky Note4 | Sticky Note | Canvas rationale for orchestration agent |  |  | ## Fleet Orchestration Agent<br>**Why** — Central coordinator that delegates tasks to specialised sub-agents for parallel processing. |
| Sticky Note5 | Sticky Note | Canvas rationale for validation agent |  |  | ## Telemetry Validation Agent<br>**Why** — AI validates data integrity before orchestration to prevent bad data propagation. |
| Sticky Note6 | Sticky Note | Canvas rationale for routing block |  |  | ## Route by Service Priority<br>**Why** — Rules-based routing ensures urgent cases receive immediate attention. |
| Sticky Note7 | Sticky Note | Canvas rationale for logging/response |  |  | ## Log Safety Traceability / Log Compliance Escalation and Response<br>**Why** — Google Sheets logging maintains auditable records for regulatory compliance. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI-powered fleet telemetry triage with safety compliance and routing`.
   - Keep the workflow inactive until credentials and endpoint URLs are fully configured.

2. **Add the webhook trigger**
   - Add a **Webhook** node named `Receive Vehicle Telemetry`.
   - Set:
     - Method: `POST`
     - Path: `fleet-telemetry`
     - Response Mode: `Using Respond to Webhook node`
   - Save the workflow so n8n generates the test/production webhook URLs.

3. **Add the telemetry validation agent**
   - Add a **LangChain Agent** node named `Telemetry Validation Agent`.
   - Set input text to:
     - `{{ $json.body }}`
   - Use defined prompt mode.
   - Add this system message:
     - You are a Telemetry Validation Agent for fleet vehicle health monitoring. Analyze incoming vehicle telemetry data containing health metrics, fault codes, and usage signals. Validate data integrity, classify fault severity (CRITICAL, HIGH, MEDIUM, LOW), identify safety-critical issues, and flag anomalies. Never issue control commands. Output structured validation results with fault classifications and recommended actions.
   - Enable structured output parser.
   - Connect `Receive Vehicle Telemetry` → `Telemetry Validation Agent`.

4. **Add the validation model**
   - Add an **OpenAI Chat Model** node named `Validation Agent Model`.
   - Select model `gpt-4o`.
   - Set temperature to `0.2`.
   - Connect it to the validation agent using the **AI Language Model** connector.
   - Configure OpenAI credentials.

5. **Add the validation output parser**
   - Add a **Structured Output Parser** node named `Validation Output Parser`.
   - Choose manual schema.
   - Define a schema with:
     - `validationStatus`: string enum `VALID`, `INVALID`, `ANOMALY`
     - `faultSeverity`: string enum `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `NONE`
     - `safetyCritical`: boolean
     - `faultCodes`: array of strings
     - `healthScore`: number
     - `recommendedAction`: string
     - `anomalies`: array of strings
   - Mark as required:
     - `validationStatus`
     - `faultSeverity`
     - `safetyCritical`
     - `recommendedAction`
   - Connect it to the validation agent using the **AI Output Parser** connector.

6. **Add the fleet orchestration agent**
   - Add another **LangChain Agent** node named `Fleet Orchestration Agent`.
   - Set input text to:
     - `{{ $json }}`
   - Use defined prompt mode.
   - Add this system message:
     - You are a Fleet Orchestration Agent coordinating service actions, customer notifications, and compliance escalation for vehicle fleet management. Based on validated telemetry and fault analysis, determine appropriate service scheduling, customer communication strategies, and regulatory compliance actions. Use your sub-agents for safety compliance analysis and service scheduling. Never issue real-time vehicle control commands. Focus on coordination, notification, and traceability.
   - Enable structured output parser.
   - Connect `Telemetry Validation Agent` → `Fleet Orchestration Agent`.

7. **Add the orchestration model**
   - Add an **OpenAI Chat Model** node named `Orchestration Agent Model`.
   - Set model to `gpt-4o`.
   - Set temperature to `0.3`.
   - Connect it to `Fleet Orchestration Agent` with the **AI Language Model** connector.
   - Use OpenAI credentials.

8. **Add the orchestration output parser**
   - Add a **Structured Output Parser** node named `Orchestration Output Parser`.
   - Define manual schema:
     - `serviceAction`: enum `IMMEDIATE_SERVICE`, `SCHEDULED_SERVICE`, `MONITOR`, `NO_ACTION`
     - `servicePriority`: enum `URGENT`, `HIGH`, `NORMAL`, `LOW`
     - `customerNotification`: object with:
       - `required`: boolean
       - `channel`: enum `EMAIL`, `SLACK`, `BOTH`
       - `message`: string
     - `complianceEscalation`: boolean
     - `complianceDetails`: string
     - `scheduledDate`: string
   - Required fields:
     - `serviceAction`
     - `servicePriority`
     - `customerNotification`
     - `complianceEscalation`
   - Connect it to `Fleet Orchestration Agent` with the **AI Output Parser** connector.

9. **Add the safety threshold tool**
   - Add a **LangChain Code Tool** node named `Safety Threshold Calculator`.
   - Paste code implementing:
     - input extraction with `$fromAI()` for `faultSeverity`, `healthScore`, `faultCodes`
     - `riskScore = 100 - healthScore`
     - severity increments for `CRITICAL`, `HIGH`, `MEDIUM`
     - threshold flags `exceedsThreshold` and `requiresImmediateAction`
   - Set tool description to indicate it calculates safety thresholds and risk scores.
   - Connect it to `Fleet Orchestration Agent` with an **AI Tool** connector.

10. **Add the safety compliance specialist**
    - Add a **LangChain Agent Tool** node named `Safety Compliance Sub-Agent`.
    - Set text to:
      - `{{ $fromAI('telemetryData', 'The validated telemetry and fault data to analyze for compliance') }}`
    - Set system message to focus on:
      - safety compliance violations
      - reporting requirements
      - recall assessment
      - safety bulletins
      - traceability requirements
    - Connect it as an **AI Tool** to `Fleet Orchestration Agent`.

11. **Add the compliance model**
    - Add an **OpenAI Chat Model** node named `Safety Compliance Model`.
    - Set model to `gpt-4o`.
    - Set temperature to `0.1`.
    - Connect it to `Safety Compliance Sub-Agent` with the **AI Language Model** connector.
    - Use OpenAI credentials.

12. **Add the scheduling specialist**
    - Add a **LangChain Agent Tool** node named `Service Scheduling Sub-Agent`.
    - Set text to:
      - `{{ $fromAI('serviceRequest', 'The service action request with fault details and priority level') }}`
    - Set system message for:
      - service timing
      - maintenance prioritization
      - capacity
      - parts availability
      - operational impact
    - Optionally set the same tool description used in the source workflow.
    - Connect it as an **AI Tool** to `Fleet Orchestration Agent`.

13. **Add the scheduling model**
    - Add an **OpenAI Chat Model** node named `Service Scheduling Model`.
    - Set model to `gpt-4o`.
    - Set temperature to `0.3`.
    - Connect it to `Service Scheduling Sub-Agent` with the **AI Language Model** connector.

14. **Add the customer email tool**
    - Add a **Gmail Tool** node named `Customer Email Notification Tool`.
    - Configure:
      - To: `{{ $fromAI('recipientEmail', 'Customer email address', 'string') }}`
      - Subject: `{{ $fromAI('emailSubject', 'Email subject line', 'string') }}`
      - Body: `{{ $fromAI('emailBody', 'Email message content', 'string') }}`
    - Add a manual tool description explaining it sends customer notices.
    - Connect Gmail OAuth2 credentials.
    - Connect it as an **AI Tool** to `Fleet Orchestration Agent`.

15. **Add the Slack alert tool**
    - Add a **Slack Tool** node named `Fleet Operations Alert Tool`.
    - Configure:
      - Message text: `{{ $fromAI('alertMessage', 'Alert message content', 'string') }}`
      - Destination type: channel
      - Channel ID/name: `{{ $fromAI('slackChannel', 'Slack channel ID or name', 'string') }}`
      - Authentication: OAuth2
    - Add tool description for urgent fleet alerts.
    - Connect Slack OAuth2 credentials.
    - Connect it as an **AI Tool** to `Fleet Orchestration Agent`.

16. **Add the external fleet API tool**
    - Add an **HTTP Request Tool** node named `Fleet Management API Tool`.
    - Configure:
      - URL: `{{ $fromAI('apiUrl', 'The API endpoint URL to call', 'string') }}`
      - Method: `{{ $fromAI('httpMethod', 'HTTP method (GET, POST, PUT, DELETE)', 'string', 'GET') }}`
    - Add a description explaining it queries fleet history, service records, or compliance systems.
    - If your external API needs auth, add headers or credentials here.
    - Connect it as an **AI Tool** to `Fleet Orchestration Agent`.

17. **Add the priority router**
    - Add a **Switch** node named `Route by Service Priority`.
    - Create four string-equals rules on:
      - `{{ $json.servicePriority }}`
    - Rule values:
      - `URGENT`
      - `HIGH`
      - `NORMAL`
      - `LOW`
    - Connect `Fleet Orchestration Agent` → `Route by Service Priority`.

18. **Add the urgent-service API call**
    - Add an **HTTP Request** node named `Urgent Service API Call`.
    - Set:
      - Method: `POST`
      - URL: your urgent service endpoint
      - Timeout: `30000`
      - Send body: enabled
    - Add body fields:
      - `serviceAction`
      - `servicePriority`
      - `scheduledDate`
      - `complianceEscalation`
    - Important: verify whether your orchestration agent output is exposed as `$json.serviceAction` or `$json.output.serviceAction` in your n8n version. The source workflow uses `output.*`, but this should be tested.
    - Connect urgent output of the switch to this node.

19. **Add the high-priority preparation node**
    - Add a **Set** node named `Prepare High Priority Service`.
    - Create fields:
      - `priority = HIGH`
      - `serviceType = {{ $json.serviceAction }}`
      - `scheduledDate = {{ $json.scheduledDate }}`
      - `notificationSent = {{ $json.customerNotification.required }}`
      - `timestamp = {{ $now.toISO() }}`
    - Connect the `HIGH` branch to this node.

20. **Add the normal-priority preparation node**
    - Add a **Set** node named `Prepare Normal Service`.
    - Same fields as above, but `priority = NORMAL`.
    - Connect the `NORMAL` branch to this node.

21. **Add the low-priority preparation node**
    - Add a **Set** node named `Prepare Low Priority Service`.
    - Same fields as above, but `priority = LOW`.
    - Connect the `LOW` branch to this node.

22. **Add the compliance check**
    - Add an **If** node named `Check Compliance Escalation`.
    - Condition:
      - boolean true on `{{ $json.complianceEscalation }}`
    - Connect `Fleet Orchestration Agent` → `Check Compliance Escalation`.

23. **Add the compliance report builder**
    - Add a **Set** node named `Prepare Compliance Report`.
    - Create fields:
      - `reportType = COMPLIANCE_ESCALATION`
      - `complianceDetails = {{ $json.complianceDetails }}`
      - `faultSeverity = {{ $('Telemetry Validation Agent').item.json.output.faultSeverity }}`
      - `safetyCritical = {{ $('Telemetry Validation Agent').item.json.output.safetyCritical }}`
      - `timestamp = {{ $now.toISO() }}`
      - `vehicleId = {{ $('Receive Vehicle Telemetry').item.json.body.vehicleId }}`
    - Connect the true output of `Check Compliance Escalation` to this node.

24. **Add the service traceability log**
    - Add a **Google Sheets** node named `Log Safety Traceability`.
    - Operation: `Append`
    - Select your spreadsheet document
    - Select the pre-created sheet/tab for service traceability
    - Connect Google Sheets OAuth2 credentials
    - Connect all service branches to it:
      - `Urgent Service API Call`
      - `Prepare High Priority Service`
      - `Prepare Normal Service`
      - `Prepare Low Priority Service`

25. **Add the compliance log**
    - Add another **Google Sheets** node named `Log Compliance Escalation`.
    - Operation: `Append`
    - Select your spreadsheet document
    - Select a dedicated compliance escalation sheet/tab
    - Connect `Prepare Compliance Report` → `Log Compliance Escalation`

26. **Add the final response node**
    - Add a **Respond to Webhook** node named `Send Webhook Response`.
    - Set response type to JSON.
    - Build a response body similar to:
      - `status: processed`
      - validation status
      - fault severity
      - service action
      - service priority
      - compliance escalation
      - `traceabilityLogged: true`
      - human-readable message confirming no control commands were issued
    - Use cross-node expressions to reference validation and orchestration output.
    - Connect:
      - `Log Safety Traceability` → `Send Webhook Response`
      - `Log Compliance Escalation` → `Send Webhook Response`

27. **Add credentials**
    - Configure:
      - **OpenAI API** credential for all model nodes
      - **Gmail OAuth2** for the email tool
      - **Slack OAuth2** for the alert tool
      - **Google Sheets OAuth2** for both Sheets nodes
    - Add HTTP auth/headers if your fleet APIs require them

28. **Pre-create and map Google Sheets columns**
    - Create one tab for service traceability
    - Create another tab for compliance escalation
    - Ensure column names match what the incoming records contain, or explicitly map fields in the Sheets nodes

29. **Replace placeholders**
    - Replace the urgent-service placeholder URL with a real endpoint
    - Constrain the fleet API tool if needed so AI cannot call arbitrary URLs
    - Tune risk thresholds in the code tool

30. **Test with sample payloads**
    - Use a sample telemetry payload containing:
      - vehicle ID
      - health score
      - fault codes
      - safety-related telemetry values
    - Confirm:
      - validation output parses correctly
      - orchestration output contains all required fields
      - switch routes properly
      - Google Sheets append succeeds
      - final webhook response is returned once

31. **Validate expression paths carefully**
    - The source workflow references some agent results as `item.json.output.*`.
    - Depending on your n8n/LangChain node behavior, actual fields may appear at:
      - `{{$json.output.field}}`
      - or `{{$json.field}}`
    - Verify these paths in execution data and update:
      - urgent API body expressions
      - compliance report expressions
      - final webhook response expressions

32. **Activate only after error handling is acceptable**
    - Consider adding:
      - webhook authentication
      - retry/error branches
      - fallback response on AI failure
      - rate limiting
      - validation for required telemetry fields before AI calls

### Sub-workflow setup
There are **no separate sub-workflows** invoked through Execute Workflow nodes.  
The so-called “sub-agents” are AI tool-style agent nodes embedded within the same workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: OpenAI API key or compatible LLM, Gmail OAuth credentials, Google Sheets with log tabs pre-created | Canvas note |
| Use case: logistics fleets auto-triaging vehicle fault alerts by severity | Canvas note |
| Customization: swap OpenAI for any LangChain-compatible model | Canvas note |
| Benefit: eliminates manual telemetry triage and reduces response lag | Canvas note |
| Setup checklist: configure webhook URL, AI credentials, Gmail, Google Sheets, fleet API endpoints, and threshold values | Canvas note |
| Workflow rationale: AI validates telemetry first, then orchestrates service, compliance, notifications, routing, logging, and unified response | Canvas note |
| Safety Compliance & Service Scheduling Sub-Agents: used to detect threshold breaches and optimize maintenance timing | Canvas note |
| Fleet Orchestration Agent: central coordinator delegating tasks to specialized tools | Canvas note |
| Telemetry Validation Agent: prevents bad data propagation by validating before orchestration | Canvas note |
| Route by Service Priority: rules-based routing gives urgent cases immediate attention | Canvas note |
| Logging in Google Sheets supports auditable records for regulatory compliance | Canvas note |

## Additional implementation notes
- The workflow has **one entry point**: `Receive Vehicle Telemetry`.
- It has **no Execute Workflow sub-workflow nodes**.
- Two nodes are incomplete until configured:
  - `Urgent Service API Call` URL placeholder
  - both Google Sheets nodes have blank spreadsheet and sheet selections
- The design uses AI tools for operational actions. In production, constrain external URLs and notification targets to avoid unsafe or unintended calls.
- Because both service logging and compliance logging can feed the same `Respond to Webhook` node, execution behavior should be tested to ensure exactly one response is sent per request.