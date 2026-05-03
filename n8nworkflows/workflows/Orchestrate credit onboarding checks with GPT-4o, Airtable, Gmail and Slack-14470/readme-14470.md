Orchestrate credit onboarding checks with GPT-4o, Airtable, Gmail and Slack

https://n8nworkflows.xyz/workflows/orchestrate-credit-onboarding-checks-with-gpt-4o--airtable--gmail-and-slack-14470


# Orchestrate credit onboarding checks with GPT-4o, Airtable, Gmail and Slack

# 1. Workflow Overview

This workflow implements an AI-assisted credit operations orchestration layer for onboarding and verification screening. It receives a POST request through a webhook, asks a GPT-4o-based agent to assess the case, optionally use operational tools such as KYC, identity, sanctions, communication, and escalation channels, then produces a structured decision payload. That payload is routed into one of several operational branches: eligible, pending verification, documentation required, compliance exception, or ineligible.

The workflow is designed for fintech or credit operations teams that need a centralized intake and triage process without delegating final credit approval to AI. The AI agent is constrained to validate completeness, identify verification steps, and escalate exceptions.

## 1.1 Input Reception and AI Orchestration

The workflow starts with a webhook that receives inbound application or servicing payloads. The incoming body is passed directly to a LangChain agent configured as a “Credit Operations Coordinator,” backed by GPT-4o, memory, external verification tools, communication tools, and a structured output parser.

## 1.2 AI Tooling and Structured Decision Generation

Within the agent block, the model can call external tools for KYC, credit bureau lookup, identity verification, sanctions screening, Gmail outreach, and Slack escalation/alerting. The agent must return a JSON structure matching a strict schema that includes request type, customer ID, eligibility status, compliance signals, actions, escalation flags, and next steps.

## 1.3 Outcome Routing

The structured result is evaluated by a Switch node using `output.eligibilityStatus`. The workflow splits into five branches:
- eligible
- pending_verification
- requires_documentation
- compliance_exception
- ineligible

## 1.4 Outcome Handling

Each branch prepares a normalized data payload. Depending on the branch, the workflow stores customer records, triggers a sub-workflow, sends a documentation email, or posts a compliance Slack alert.

## 1.5 Audit Logging and API Response

All operational branches converge into a common audit logging stage. The workflow writes a final log record, formats a response object, and returns JSON to the original webhook caller via a Respond to Webhook node.

---

# 2. Block-by-Block Analysis

## Block 1 — Webhook Intake

### Overview
This block receives the external request and hands the request body to the AI coordinator. It is the workflow’s entry point and uses deferred webhook response handling so the final response can be returned after downstream processing completes.

### Nodes Involved
- Credit Operations Webhook

### Node Details

#### Credit Operations Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for HTTP POST requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `credit-operations`
  - Response mode: `responseNode`, meaning the request remains open until a later Respond to Webhook node sends the reply.
- **Key expressions or variables used:**  
  No custom expression in parameters, but downstream the workflow reads from `{{$json.body}}`.
- **Input and output connections:**  
  - No input
  - Output to: `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Uses node type version `2.1`.
- **Edge cases or potential failure types:**  
  - Invalid caller payload structure
  - Large request bodies causing parsing issues
  - If downstream execution fails before `Send Response`, the caller may receive timeout behavior depending on environment/proxy
- **Sub-workflow reference:**  
  None

---

## Block 2 — AI Coordination, Memory, Tools, and Structured Output

### Overview
This is the core intelligence block. The AI agent reads the inbound request body, reasons about the onboarding/compliance case, optionally uses connected tools, and must return a structured object conforming to the schema defined in the output parser.

### Nodes Involved
- Credit Operations Coordinator
- OpenAI Chat Model
- Structured Output Parser
- Conversation Memory
- KYC Verification API Tool
- Credit Bureau API Tool
- Identity Verification Tool
- Sanctions Screening Tool
- Email Communication Tool
- Risk Team Escalation Tool
- Risk Escalation Slack Tool
- Compliance Alert Tool

### Node Details

#### Credit Operations Coordinator
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main orchestration agent that coordinates decisioning logic and tool invocation.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Input text: `={{ $json.body }}`
  - Has output parser: enabled
  - System message explicitly constrains the agent:
    - may validate completeness and compliance
    - may orchestrate verification and communication
    - may escalate
    - may not approve credit or alter scoring models
- **Key expressions or variables used:**  
  - `{{$json.body}}` as the model input
- **Input and output connections:**  
  - Main input from: `Credit Operations Webhook`
  - AI language model input from: `OpenAI Chat Model`
  - AI memory input from: `Conversation Memory`
  - AI output parser from: `Structured Output Parser`
  - AI tools from all connected tool nodes
  - Main output to: `Route by Eligibility Status`
- **Version-specific requirements:**  
  Uses type version `3.1`; requires n8n builds with LangChain agent support.
- **Edge cases or potential failure types:**  
  - Invalid or ambiguous inbound body causing low-quality model output
  - Tool invocation failures
  - Output parser validation errors if model returns non-conforming JSON
  - Token/context overflow if body is large
- **Sub-workflow reference:**  
  None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the GPT-4o model used by the agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively deterministic behavior
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output to AI language model input of `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `1.3`; requires valid OpenAI credentials.
- **Edge cases or potential failure types:**  
  - Authentication or quota errors
  - Model availability issues
  - API rate limits
- **Sub-workflow reference:**  
  None

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the agent output into a strict schema.
- **Configuration choices:**  
  Manual JSON schema with required top-level fields:
  - `requestType`
  - `customerId`
  - `eligibilityStatus`
  - `complianceSignals`
  - `requiredActions`
  - `escalationRequired`
  - `nextSteps`
  
  Allowed `requestType` values:
  - onboarding
  - kyc_verification
  - credit_review
  - servicing
  - compliance_check
  
  Allowed `eligibilityStatus` values:
  - eligible
  - pending_verification
  - requires_documentation
  - compliance_exception
  - ineligible
  
  `complianceSignals` requires:
  - `kycComplete`
  - `identityVerified`
  - `sanctionsCheckPassed`
  - `documentsComplete`
  - `riskTier`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output to AI output parser input of `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Model returns malformed object
  - Enum mismatch
  - Missing required fields
  - Incompatible nested object/array shape
- **Sub-workflow reference:**  
  None

#### Conversation Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Provides short rolling memory for the agent.
- **Configuration choices:**  
  - Context window length: `10`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output to AI memory input of `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Limited usefulness in stateless webhook scenarios unless sessioning is configured elsewhere
  - Memory may not persist meaningfully across unrelated executions
- **Sub-workflow reference:**  
  None

#### KYC Verification API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`  
  AI-callable HTTP tool for KYC verification.
- **Configuration choices:**  
  - Method: `POST`
  - URL: placeholder KYC endpoint
  - Tool description instructs the model to provide customer ID and personal information
- **Key expressions or variables used:**  
  None shown in URL/body; actual request body is not configured in this export
- **Input and output connections:**  
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Placeholder URL must be replaced
  - Missing auth headers or request body
  - Timeouts, non-200 responses, malformed JSON
- **Sub-workflow reference:**  
  None

#### Credit Bureau API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`
- **Configuration choices:**  
  - Method: `POST`
  - URL: placeholder credit bureau endpoint
  - Tool description expects customer ID and SSN
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `4.4`
- **Edge cases or potential failure types:**  
  - Placeholder endpoint
  - Sensitive data handling concerns
  - Bureau API auth/rate limits
- **Sub-workflow reference:**  
  None

#### Identity Verification Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`
- **Configuration choices:**  
  - Method: `POST`
  - URL: placeholder identity verification endpoint
  - Tool description references government ID and biometric checks
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `4.4`
- **Edge cases or potential failure types:**  
  - Placeholder endpoint
  - Vendor-specific payload mismatch
  - Potential latency on verification APIs
- **Sub-workflow reference:**  
  None

#### Sanctions Screening Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool`
- **Configuration choices:**  
  - Method: `POST`
  - URL: placeholder sanctions endpoint
  - Tool description requests customer name and country
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `4.4`
- **Edge cases or potential failure types:**  
  - Placeholder endpoint
  - False positive/partial match ambiguity
  - Country/name normalization issues
- **Sub-workflow reference:**  
  None

#### Email Communication Tool
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  AI-callable Gmail sending tool.
- **Configuration choices:**  
  - To: `{{ $fromAI('toEmail', 'Recipient email address', 'string') }}`
  - Subject: `{{ $fromAI('subject', 'Email subject line', 'string') }}`
  - Message: `{{ $fromAI('message', 'Email message body', 'string') }}`
  - Manual tool description for customer communication use cases
- **Key expressions or variables used:**  
  `$fromAI(...)` fields for AI-supplied arguments
- **Input and output connections:**  
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `2.2`; requires Gmail OAuth2.
- **Edge cases or potential failure types:**  
  - Gmail auth expiration
  - Invalid recipient format
  - Message generated without required compliance wording
- **Sub-workflow reference:**  
  None

#### Risk Team Escalation Tool
- **Type and technical role:** `n8n-nodes-base.slackHitlTool`  
  AI-callable Slack human-in-the-loop escalation tool.
- **Configuration choices:**  
  - Sends to a fixed channel placeholder
  - Message text comes from AI:
    `{{ $fromAI('escalationMessage', 'Detailed escalation message with customer context and risk factors', 'string') }}`
  - Authentication: OAuth2
- **Key expressions or variables used:**  
  `$fromAI('escalationMessage', ...)`
- **Input and output connections:**  
  - Receives an AI tool connection from `Risk Escalation Slack Tool`
  - Sends AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `2.4`
- **Edge cases or potential failure types:**  
  - Placeholder channel ID must be replaced
  - Slack permission scope issues
  - Ambiguity because it is chained with another Slack tool
- **Sub-workflow reference:**  
  None

#### Risk Escalation Slack Tool
- **Type and technical role:** `n8n-nodes-base.slackTool`  
  A secondary AI-callable Slack tool connected to the HITL Slack tool.
- **Configuration choices:**  
  - Text: `{{ $fromAI('message', 'Risk escalation message', 'string') }}`
  - Channel: `{{ $fromAI('channel', 'Risk team Slack channel ID', 'string') }}`
  - Manual description: sends notification to risk team Slack channel
- **Key expressions or variables used:**  
  `$fromAI('message', ...)`, `$fromAI('channel', ...)`
- **Input and output connections:**  
  - Output connected to AI tool input of `Risk Team Escalation Tool`
  - Not connected directly to the coordinator
- **Version-specific requirements:**  
  Type version `2.4`
- **Edge cases or potential failure types:**  
  - This chained AI-tool-to-AI-tool pattern is unusual and may be hard to reason about
  - Dynamic channel may fail if AI outputs a non-existent channel ID
- **Sub-workflow reference:**  
  None

#### Compliance Alert Tool
- **Type and technical role:** `n8n-nodes-base.slackTool`  
  AI-callable Slack alerting tool for compliance notifications.
- **Configuration choices:**  
  - Text: `{{ $fromAI('alertMessage', 'Compliance alert message with violation details', 'string') }}`
  - Channel ID: `{{ $fromAI('channel', 'Slack channel ID or name for compliance alerts', 'string') }}`
  - Manual tool description focused on sanctions hits and compliance exceptions
- **Key expressions or variables used:**  
  `$fromAI('alertMessage', ...)`, `$fromAI('channel', ...)`
- **Input and output connections:**  
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**  
  Type version `2.4`
- **Edge cases or potential failure types:**  
  - AI may choose wrong channel
  - Slack OAuth scopes may not allow posting
- **Sub-workflow reference:**  
  None

---

## Block 3 — Routing by Eligibility Status

### Overview
This block evaluates the AI-generated structured result and directs the execution into one of five downstream branches. It is the main control point after model inference.

### Nodes Involved
- Route by Eligibility Status

### Node Details

#### Route by Eligibility Status
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branches execution based on `output.eligibilityStatus`.
- **Configuration choices:**  
  Five string equality conditions:
  1. `eligible`
  2. `pending_verification`
  3. `requires_documentation`
  4. `compliance_exception`
  5. `ineligible`
- **Key expressions or variables used:**  
  - `={{ $json.output.eligibilityStatus }}`
- **Input and output connections:**  
  - Input from: `Credit Operations Coordinator`
  - Outputs to:
    - `Prepare Eligible Customer Data`
    - `Prepare Verification Request`
    - `Prepare Documentation Request`
    - `Prepare Compliance Escalation`
    - `Prepare Ineligible Customer Data`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - If parser/schema fails upstream, this node is never reached
  - If the AI somehow returns an unknown value outside enum constraints, no branch may match
- **Sub-workflow reference:**  
  None

---

## Block 4 — Eligible Branch

### Overview
This branch normalizes data for customers deemed eligible and stores the result in a data table. It then forwards the record to the common audit logging stage.

### Nodes Involved
- Prepare Eligible Customer Data
- Store Eligible Customers

### Node Details

#### Prepare Eligible Customer Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a clean eligible-customer record.
- **Configuration choices:**  
  Assigns:
  - `customerId` from `output.customerId`
  - `status` = `eligible`
  - `eligibilityStatus`
  - compliance signal booleans individually
  - `riskTier`
  - `processedAt` = current ISO time
  - `nextSteps`
- **Key expressions or variables used:**  
  - `{{$json.output.customerId}}`
  - `{{$json.output.complianceSignals.kycComplete}}`
  - `{{$now.toISO()}}`
- **Input and output connections:**  
  - Input from: `Route by Eligibility Status`
  - Output to: `Store Eligible Customers`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Missing nested fields if AI output is incomplete
- **Sub-workflow reference:**  
  None

#### Store Eligible Customers
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists eligible customer records.
- **Configuration choices:**  
  - Auto-map input data
  - Data table ID is a placeholder
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from: `Prepare Eligible Customer Data`
  - Output to: `Prepare Audit Log`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - Placeholder table ID must be replaced
  - Schema mismatch between incoming fields and table columns
- **Sub-workflow reference:**  
  None

---

## Block 5 — Pending Verification Branch

### Overview
This branch is used when further verification must occur before processing can continue. It packages the verification request and hands it off to another workflow.

### Nodes Involved
- Prepare Verification Request
- Trigger Verification Workflow

### Node Details

#### Prepare Verification Request
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Assigns:
  - `customerId`
  - `status` = `pending_verification`
  - `eligibilityStatus`
  - `requiredActions`
  - `complianceSignals`
  - `processedAt`
  - `nextSteps`
  
  Note: `requiredActions` and `complianceSignals` are stringified with `JSON.stringify(...)` despite being typed as array/object in the Set node.
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json.output.requiredActions) }}`
  - `={{ JSON.stringify($json.output.complianceSignals) }}`
- **Input and output connections:**  
  - Input from: `Route by Eligibility Status`
  - Output to: `Trigger Verification Workflow`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Downstream consumers may expect native arrays/objects but receive serialized strings
- **Sub-workflow reference:**  
  Feeds an Execute Workflow node

#### Trigger Verification Workflow
- **Type and technical role:** `n8n-nodes-base.executeWorkflow`  
  Calls a sub-workflow and waits for completion.
- **Configuration choices:**  
  - `waitForSubWorkflow: true`
  - Workflow ID is currently blank
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from: `Prepare Verification Request`
  - Output to: `Prepare Audit Log`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - No sub-workflow configured, so this will fail until a workflow ID is selected
  - Input contract for the sub-workflow is not explicitly configured here
  - Wait-for-completion means slow sub-workflows delay webhook response
- **Sub-workflow reference:**  
  Yes. Intended verification sub-workflow, but not specified in the export.

---

## Block 6 — Documentation Request Branch

### Overview
This branch handles cases where the customer must provide more documents. It prepares a branch-specific payload, sends an email, then forwards the result to the audit stage.

### Nodes Involved
- Prepare Documentation Request
- Send Documentation Request Email

### Node Details

#### Prepare Documentation Request
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Assigns:
  - `customerId`
  - `status` = `requires_documentation`
  - `eligibilityStatus`
  - `requiredActions`
  - `complianceSignals`
  - `processedAt`
  - `nextSteps`
  
  Like the verification branch, nested structures are stringified.
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json.output.requiredActions) }}`
  - `={{ JSON.stringify($json.output.complianceSignals) }}`
- **Input and output connections:**  
  - Input from: `Route by Eligibility Status`
  - Output to: `Send Documentation Request Email`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Stringified arrays/objects may break downstream template logic
- **Sub-workflow reference:**  
  None

#### Send Documentation Request Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a fixed-format email to request documentation.
- **Configuration choices:**  
  - To: `{{$json.customerEmail || '<__PLACEHOLDER_VALUE__customer_email__>'}}`
  - Subject: `Documentation Required for Credit Application`
  - Attribution disabled
  - Message body references:
    - customer ID
    - next steps
    - required actions list via `.map(...)`
- **Key expressions or variables used:**  
  - `{{$json.customerEmail || '<__PLACEHOLDER_VALUE__customer_email__>'}}`
  - `{{$json.requiredActions.map(action => '- ' + action.details).join('\n')}}`
- **Input and output connections:**  
  - Input from: `Prepare Documentation Request`
  - Output to: `Prepare Audit Log`
- **Version-specific requirements:**  
  Type version `2.2`; requires Gmail OAuth2.
- **Edge cases or potential failure types:**  
  - **Important:** `requiredActions` was stringified upstream, but this node treats it like an array with `.map(...)`. That can cause expression failure unless the data remains a native array in practice.
  - Placeholder fallback email must be replaced
  - Missing `customerEmail`
- **Sub-workflow reference:**  
  None

---

## Block 7 — Compliance Exception Branch

### Overview
This branch handles compliance exceptions by preparing an escalation payload and sending a Slack alert to the compliance team. It then routes the result to audit logging.

### Nodes Involved
- Prepare Compliance Escalation
- Alert Compliance Team

### Node Details

#### Prepare Compliance Escalation
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Assigns:
  - `customerId`
  - `status` = `compliance_exception`
  - `eligibilityStatus`
  - `escalationRequired`
  - `escalationReason`
  - `complianceSignals`
  - `requiredActions`
  - `processedAt`
  - `nextSteps`
  
  Again, nested objects/arrays are stringified.
- **Key expressions or variables used:**  
  - `={{ $json.output.escalationReason }}`
  - `={{ JSON.stringify($json.output.complianceSignals) }}`
  - `={{ JSON.stringify($json.output.requiredActions) }}`
- **Input and output connections:**  
  - Input from: `Route by Eligibility Status`
  - Output to: `Alert Compliance Team`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Stringification can break downstream nested-property references
- **Sub-workflow reference:**  
  None

#### Alert Compliance Team
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a compliance exception alert to Slack.
- **Configuration choices:**  
  - Fixed Slack channel placeholder
  - Message includes:
    - customer ID
    - status
    - risk tier
    - escalation requirement and reason
    - compliance signals
    - next steps
    - required actions
- **Key expressions or variables used:**  
  - `{{$json.complianceSignals.riskTier}}`
  - `{{$json.requiredActions.map(action => action.priority + ': ' + action.details).join('\n')}}`
- **Input and output connections:**  
  - Input from: `Prepare Compliance Escalation`
  - Output to: `Prepare Audit Log`
- **Version-specific requirements:**  
  Type version `2.4`; requires Slack OAuth2.
- **Edge cases or potential failure types:**  
  - **Important:** upstream `complianceSignals` and `requiredActions` are stringified, but this node expects objects/arrays. This is likely to fail unless corrected.
  - Placeholder Slack channel must be replaced
  - Slack permissions or invalid channel errors
- **Sub-workflow reference:**  
  None

---

## Block 8 — Ineligible Branch

### Overview
This branch handles customers deemed ineligible. It stores the case in a separate table and forwards the data into the audit logging stage.

### Nodes Involved
- Prepare Ineligible Customer Data
- Store Ineligible Customers

### Node Details

#### Prepare Ineligible Customer Data
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Assigns:
  - `customerId`
  - `status` = `ineligible`
  - `eligibilityStatus`
  - `complianceSignals`
  - `processedAt`
  - `nextSteps`
  
  `complianceSignals` is stringified.
- **Key expressions or variables used:**  
  - `={{ JSON.stringify($json.output.complianceSignals) }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input from: `Route by Eligibility Status`
  - Output to: `Store Ineligible Customers`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - Stringified compliance object may make later reuse harder
- **Sub-workflow reference:**  
  None

#### Store Ineligible Customers
- **Type and technical role:** `n8n-nodes-base.dataTable`
- **Configuration choices:**  
  - Auto-map input data
  - Data table ID is a placeholder
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from: `Prepare Ineligible Customer Data`
  - Output to: `Prepare Audit Log`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - Placeholder table ID
  - Table column mismatch
- **Sub-workflow reference:**  
  None

---

## Block 9 — Unified Audit Logging and Webhook Response

### Overview
All branches converge here to create a normalized audit log, persist it, then return a compact API response to the caller. This ensures a single operational trail regardless of branch taken.

### Nodes Involved
- Prepare Audit Log
- Log All Operations
- Format Response
- Send Response

### Node Details

#### Prepare Audit Log
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes branch outputs into a common audit schema.
- **Configuration choices:**  
  Assigns:
  - `customerId`
  - `requestType` from the original coordinator output
  - `eligibilityStatus` using current branch value fallback
  - `processedAt`
  - `complianceSignals`
  - `requiredActions`
  - `escalationRequired`
  - `escalationReason`
  - `nextSteps`
  
  It references the coordinator node directly for `requestType`.
- **Key expressions or variables used:**  
  - `={{ $('Credit Operations Coordinator').item.json.output.requestType }}`
  - `={{ $json.eligibilityStatus || $json.status }}`
  - `={{ JSON.stringify($json.complianceSignals) }}`
  - `={{ JSON.stringify($json.requiredActions || []) }}`
- **Input and output connections:**  
  - Inputs from:
    - `Store Eligible Customers`
    - `Trigger Verification Workflow`
    - `Send Documentation Request Email`
    - `Alert Compliance Team`
    - `Store Ineligible Customers`
  - Output to: `Log All Operations`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - If branch nodes transform data unexpectedly, fields may be missing
  - Nested values may be double-stringified
- **Sub-workflow reference:**  
  None

#### Log All Operations
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists the audit log.
- **Configuration choices:**  
  - Auto-map input data
  - Audit table ID is a placeholder
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from: `Prepare Audit Log`
  - Output to: `Format Response`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - Placeholder table ID
  - Auto-mapping issues if table schema differs from log shape
- **Sub-workflow reference:**  
  None

#### Format Response
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a clean response object for the original webhook caller.
- **Configuration choices:**  
  Assigns:
  - `success` = true
  - `customerId`
  - `status`
  - `message`
  - `processedAt`
  - `escalationRequired`
- **Key expressions or variables used:**  
  - `={{ $json.eligibilityStatus || $json.status }}`
  - `={{ $json.nextSteps }}`
- **Input and output connections:**  
  - Input from: `Log All Operations`
  - Output to: `Send Response`
- **Version-specific requirements:**  
  Type version `3.4`
- **Edge cases or potential failure types:**  
  - If logging strips or omits source fields, response may be incomplete
- **Sub-workflow reference:**  
  None

#### Send Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns the final JSON payload to the original HTTP caller.
- **Configuration choices:**  
  - Respond with: JSON
  - Response body: `={{ $json }}`
- **Key expressions or variables used:**  
  - `{{$json}}`
- **Input and output connections:**  
  - Input from: `Format Response`
- **Version-specific requirements:**  
  Type version `1.1`
- **Edge cases or potential failure types:**  
  - If this node is never reached, webhook callers may hang until timeout
- **Sub-workflow reference:**  
  None

---

## Block 10 — Sticky Notes and Embedded Operational Guidance

### Overview
These nodes contain visual documentation inside the canvas. They do not affect execution, but they are important for understanding prerequisites, setup, use cases, and intent.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas annotation.
- **Configuration choices:**  
  Contains prerequisites, use cases, customization ideas, and benefits.
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- Same role as above.
- Contains setup steps for webhook, OpenAI, verification tools, Gmail, Slack, and Airtable-like storage.

#### Sticky Note2
- Same role as above.
- Contains a high-level explanation of how the workflow operates end-to-end.

#### Sticky Note3
- Same role as above.
- Labels the storage and notification area.

#### Sticky Note4
- Same role as above.
- Labels the routing area.

#### Sticky Note5
- Same role as above.
- Labels the AI verification and checks area.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Credit Operations Webhook | webhook | Receives POST requests for credit operations intake |  | Credit Operations Coordinator | ## How It Works<br>This workflow automates credit operations onboarding by running KYC verification, credit bureau checks, identity validation, and sanctions screening through a single AI-powered agent. Built for credit operations teams, compliance officers, and fintech platforms, it eliminates manual eligibility reviews that are slow and error-prone. Triggered via webhook, the Credit Operations Agent orchestrates all verification tools simultaneously, then routes customers by eligibility status, eligible, ineligible, pending documentation, or compliance escalation. Each path prepares structured data stored in Airtable, triggers appropriate follow-up actions (email, Slack alerts), and logs a full audit trail. A final formatted response is returned to the originating system, closing the loop end-to-end with no manual handoffs.<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Credit Operations Coordinator | @n8n/n8n-nodes-langchain.agent | Main AI coordinator and decision engine | Credit Operations Webhook | Route by Eligibility Status | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o model to the agent |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output from the agent |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Supplies short-term conversational memory |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| KYC Verification API Tool | httpRequestTool | AI-callable KYC verification integration |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Credit Bureau API Tool | httpRequestTool | AI-callable credit bureau integration |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Identity Verification Tool | httpRequestTool | AI-callable identity verification integration |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Sanctions Screening Tool | httpRequestTool | AI-callable sanctions screening integration |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Email Communication Tool | gmailTool | AI-callable Gmail notification tool |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Risk Team Escalation Tool | slackHitlTool | AI-callable Slack human escalation tool | Risk Escalation Slack Tool | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Compliance Alert Tool | slackTool | AI-callable Slack compliance alert tool |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Route by Eligibility Status | switch | Routes execution by eligibility status | Credit Operations Coordinator | Prepare Eligible Customer Data; Prepare Verification Request; Prepare Documentation Request; Prepare Compliance Escalation; Prepare Ineligible Customer Data | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Eligible Customer Data | set | Normalizes eligible case data | Route by Eligibility Status | Store Eligible Customers | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Verification Request | set | Normalizes pending verification payload | Route by Eligibility Status | Trigger Verification Workflow | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Documentation Request | set | Normalizes documentation-request payload | Route by Eligibility Status | Send Documentation Request Email | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Compliance Escalation | set | Normalizes compliance exception payload | Route by Eligibility Status | Alert Compliance Team | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Store Eligible Customers | dataTable | Stores eligible customer records | Prepare Eligible Customer Data | Prepare Audit Log | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Trigger Verification Workflow | executeWorkflow | Invokes sub-workflow for verification processing | Prepare Verification Request | Prepare Audit Log | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Send Documentation Request Email | gmail | Sends documentation request email | Prepare Documentation Request | Prepare Audit Log | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Alert Compliance Team | slack | Sends Slack alert for compliance exceptions | Prepare Compliance Escalation | Prepare Audit Log | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Log All Operations | dataTable | Stores audit log records | Prepare Audit Log | Format Response | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Prepare Audit Log | set | Builds unified audit log payload | Store Eligible Customers; Trigger Verification Workflow; Send Documentation Request Email; Alert Compliance Team; Store Ineligible Customers | Log All Operations | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Send Response | respondToWebhook | Returns final JSON webhook response | Format Response |  | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Format Response | set | Formats final API response body | Log All Operations | Send Response | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Risk Escalation Slack Tool | slackTool | Secondary AI-callable Slack risk notification tool |  | Risk Team Escalation Tool | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Prepare Ineligible Customer Data | set | Normalizes ineligible case data | Route by Eligibility Status | Store Ineligible Customers | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Store Ineligible Customers | dataTable | Stores ineligible customer records | Prepare Ineligible Customer Data | Prepare Audit Log | ## Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Sticky Note | stickyNote | Canvas documentation for prerequisites and benefits |  |  | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks |
| Sticky Note1 | stickyNote | Canvas documentation for setup steps |  |  | ## Setup Steps<br>1. Set webhook URL and connect Credit Operations webhook node to your intake system.<br>2. Add OpenAI API key to the OpenAI Chat Model node.<br>3. Configure KYC, Credit Bureau, Identity, and Sanctions tool credentials.<br>4. Add Gmail OAuth2 and Slack bot token for notification nodes.<br>5. Connect Airtable API key; set base/table IDs for eligible and ineligible customer stores. |
| Sticky Note2 | stickyNote | Canvas documentation for workflow behavior |  |  | ## How It Works<br>This workflow automates credit operations onboarding by running KYC verification, credit bureau checks, identity validation, and sanctions screening through a single AI-powered agent. Built for credit operations teams, compliance officers, and fintech platforms, it eliminates manual eligibility reviews that are slow and error-prone. Triggered via webhook, the Credit Operations Agent orchestrates all verification tools simultaneously, then routes customers by eligibility status, eligible, ineligible, pending documentation, or compliance escalation. Each path prepares structured data stored in Airtable, triggers appropriate follow-up actions (email, Slack alerts), and logs a full audit trail. A final formatted response is returned to the originating system, closing the loop end-to-end with no manual handoffs. |
| Sticky Note3 | stickyNote | Canvas label for store and notify section |  |  | ### Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Sticky Note4 | stickyNote | Canvas label for routing section |  |  | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Sticky Note5 | stickyNote | Canvas label for AI verification section |  |  | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Intelligent credit operations agent for verification screening and compliance`.

2. **Add a Webhook node** named `Credit Operations Webhook`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `credit-operations`.
   - Set **Response Mode** to `Using Respond to Webhook node` / `responseNode`.

3. **Add an AI Agent node** named `Credit Operations Coordinator`.
   - Choose the LangChain Agent node.
   - Set the main input text to:
     `={{ $json.body }}`
   - Use a custom system prompt describing the AI as a credit operations coordinator.
   - Include explicit constraints:
     - no final credit approvals
     - no risk model changes
     - only validation, workflow coordination, and escalation
   - Enable **structured output parsing**.

4. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Select model `gpt-4o`.
   - Set temperature to `0.2`.
   - Connect OpenAI credentials.

5. **Connect `OpenAI Chat Model` to `Credit Operations Coordinator`** using the AI language model connection.

6. **Add a Memory node** named `Conversation Memory`.
   - Use Buffer Window Memory.
   - Set context window length to `10`.
   - Connect it to the agent’s AI memory port.

7. **Add a Structured Output Parser node** named `Structured Output Parser`.
   - Use manual schema mode.
   - Define the schema with these top-level fields:
     - `requestType`
     - `customerId`
     - `eligibilityStatus`
     - `complianceSignals`
     - `requiredActions`
     - `escalationRequired`
     - `escalationReason` optional
     - `nextSteps`
   - Enforce enums:
     - request types: onboarding, kyc_verification, credit_review, servicing, compliance_check
     - eligibility statuses: eligible, pending_verification, requires_documentation, compliance_exception, ineligible
     - risk tiers: low, medium, high, critical
     - action priorities: immediate, high, medium, low
   - Connect it to the agent’s output parser port.

8. **Add an HTTP Request Tool node** named `KYC Verification API Tool`.
   - Method: `POST`
   - URL: your KYC provider endpoint
   - Add auth headers/body as required by your provider
   - Tool description should explain that it performs KYC checks
   - Connect to the agent as an AI tool.

9. **Add another HTTP Request Tool** named `Credit Bureau API Tool`.
   - Method: `POST`
   - URL: your credit bureau endpoint
   - Configure required auth and payload fields
   - Connect to the agent as an AI tool.

10. **Add another HTTP Request Tool** named `Identity Verification Tool`.
    - Method: `POST`
    - URL: your identity verification endpoint
    - Configure provider-specific auth
    - Connect to the agent as an AI tool.

11. **Add another HTTP Request Tool** named `Sanctions Screening Tool`.
    - Method: `POST`
    - URL: your sanctions/watchlist endpoint
    - Configure auth and expected body
    - Connect to the agent as an AI tool.

12. **Add a Gmail Tool node** named `Email Communication Tool`.
    - Connect Gmail OAuth2 credentials.
    - Set:
      - To: `={{ $fromAI('toEmail', 'Recipient email address', 'string') }}`
      - Subject: `={{ $fromAI('subject', 'Email subject line', 'string') }}`
      - Message: `={{ $fromAI('message', 'Email message body', 'string') }}`
    - Add a manual tool description for customer communications.
    - Connect it to the agent as an AI tool.

13. **Add a Slack HITL Tool node** named `Risk Team Escalation Tool`.
    - Connect Slack OAuth2 credentials.
    - Select a fixed risk review channel, or replace later with your actual channel ID.
    - Set message field to:
      `={{ $fromAI('escalationMessage', 'Detailed escalation message with customer context and risk factors', 'string') }}`
    - Connect it to the agent as an AI tool.

14. **Add a Slack Tool node** named `Risk Escalation Slack Tool`.
    - Connect Slack OAuth2 credentials.
    - Set text:
      `={{ $fromAI('message', 'Risk escalation message', 'string') }}`
    - Set channel:
      `={{ $fromAI('channel', 'Risk team Slack channel ID', 'string') }}`
    - Connect this tool to `Risk Team Escalation Tool` exactly as in the source workflow if you want to preserve the same design.
    - Note: this chaining is unusual; in many implementations you may connect only one risk Slack tool directly to the agent.

15. **Add a Slack Tool node** named `Compliance Alert Tool`.
    - Connect Slack OAuth2 credentials.
    - Set text:
      `={{ $fromAI('alertMessage', 'Compliance alert message with violation details', 'string') }}`
    - Set channel:
      `={{ $fromAI('channel', 'Slack channel ID or name for compliance alerts', 'string') }}`
    - Connect it to the agent as an AI tool.

16. **Connect the Webhook main output to the Agent main input.**

17. **Add a Switch node** named `Route by Eligibility Status`.
    - Set the switch value to:
      `={{ $json.output.eligibilityStatus }}`
    - Create five branches:
      1. equals `eligible`
      2. equals `pending_verification`
      3. equals `requires_documentation`
      4. equals `compliance_exception`
      5. equals `ineligible`
    - Connect the agent output to this switch.

18. **Create the eligible branch Set node** named `Prepare Eligible Customer Data`.
    - Add fields:
      - `customerId = {{$json.output.customerId}}`
      - `status = eligible`
      - `eligibilityStatus = {{$json.output.eligibilityStatus}}`
      - `kycComplete = {{$json.output.complianceSignals.kycComplete}}`
      - `identityVerified = {{$json.output.complianceSignals.identityVerified}}`
      - `sanctionsCheckPassed = {{$json.output.complianceSignals.sanctionsCheckPassed}}`
      - `documentsComplete = {{$json.output.complianceSignals.documentsComplete}}`
      - `riskTier = {{$json.output.complianceSignals.riskTier}}`
      - `processedAt = {{$now.toISO()}}`
      - `nextSteps = {{$json.output.nextSteps}}`

19. **Add a Data Table node** named `Store Eligible Customers`.
    - Use auto-map input data.
    - Select the target table for eligible customers.
    - Connect `Prepare Eligible Customer Data` to it.

20. **Create the pending verification Set node** named `Prepare Verification Request`.
    - Add:
      - `customerId`
      - `status = pending_verification`
      - `eligibilityStatus`
      - `requiredActions`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`
    - Prefer storing `requiredActions` and `complianceSignals` as native JSON if your downstream sub-workflow expects structured values. The source workflow stringifies them.

21. **Add an Execute Workflow node** named `Trigger Verification Workflow`.
    - Enable **Wait for Sub-Workflow**.
    - Select the verification sub-workflow.
    - Connect `Prepare Verification Request` to it.
    - Expected sub-workflow input should include:
      - `customerId`
      - `status`
      - `eligibilityStatus`
      - `requiredActions`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`
    - Expected sub-workflow output should be a JSON object suitable for audit logging.

22. **Create the documentation Set node** named `Prepare Documentation Request`.
    - Add:
      - `customerId`
      - `status = requires_documentation`
      - `eligibilityStatus`
      - `requiredActions`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`

23. **Add a Gmail node** named `Send Documentation Request Email`.
    - Connect Gmail OAuth2 credentials.
    - Set recipient to:
      `={{ $json.customerEmail || '<your-default-customer-email>' }}`
    - Subject:
      `Documentation Required for Credit Application`
    - Message body should include customer ID, next steps, and required action details.
    - Important: if `requiredActions` is stored as a string, parse it before mapping, or do not stringify it upstream.

24. **Create the compliance Set node** named `Prepare Compliance Escalation`.
    - Add:
      - `customerId`
      - `status = compliance_exception`
      - `eligibilityStatus`
      - `escalationRequired`
      - `escalationReason`
      - `complianceSignals`
      - `requiredActions`
      - `processedAt`
      - `nextSteps`

25. **Add a Slack node** named `Alert Compliance Team`.
    - Connect Slack OAuth2 credentials.
    - Select the compliance Slack channel.
    - Build a message including:
      - customer ID
      - status
      - risk tier
      - escalation requirement
      - escalation reason
      - compliance signals
      - next steps
      - required actions
    - Important: keep `complianceSignals` and `requiredActions` as objects/arrays, or parse them before using nested fields.

26. **Create the ineligible Set node** named `Prepare Ineligible Customer Data`.
    - Add:
      - `customerId`
      - `status = ineligible`
      - `eligibilityStatus`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`

27. **Add a Data Table node** named `Store Ineligible Customers`.
    - Auto-map the fields.
    - Select the ineligible customers table.
    - Connect `Prepare Ineligible Customer Data` to it.

28. **Connect the Switch outputs**:
    - `eligible` → `Prepare Eligible Customer Data`
    - `pending_verification` → `Prepare Verification Request`
    - `requires_documentation` → `Prepare Documentation Request`
    - `compliance_exception` → `Prepare Compliance Escalation`
    - `ineligible` → `Prepare Ineligible Customer Data`

29. **Add a Set node** named `Prepare Audit Log`.
    - Connect all branch-completion nodes into it:
      - `Store Eligible Customers`
      - `Trigger Verification Workflow`
      - `Send Documentation Request Email`
      - `Alert Compliance Team`
      - `Store Ineligible Customers`
    - Add fields:
      - `customerId = {{$json.customerId}}`
      - `requestType = {{ $('Credit Operations Coordinator').item.json.output.requestType }}`
      - `eligibilityStatus = {{$json.eligibilityStatus || $json.status}}`
      - `processedAt = {{$json.processedAt}}`
      - `complianceSignals = {{$json.complianceSignals}}`
      - `requiredActions = {{$json.requiredActions || []}}`
      - `escalationRequired = {{$json.escalationRequired || false}}`
      - `escalationReason = {{$json.escalationReason || 'N/A'}}`
      - `nextSteps = {{$json.nextSteps}}`
    - Prefer avoiding double-stringification unless your table requires text.

30. **Add a Data Table node** named `Log All Operations`.
    - Auto-map fields.
    - Select the audit log table.
    - Connect from `Prepare Audit Log`.

31. **Add a Set node** named `Format Response`.
    - Add:
      - `success = true`
      - `customerId = {{$json.customerId}}`
      - `status = {{$json.eligibilityStatus || $json.status}}`
      - `message = {{$json.nextSteps}}`
      - `processedAt = {{$json.processedAt}}`
      - `escalationRequired = {{$json.escalationRequired || false}}`

32. **Add a Respond to Webhook node** named `Send Response`.
    - Set response type to JSON.
    - Response body:
      `={{ $json }}`

33. **Connect**:
    - `Log All Operations` → `Format Response`
    - `Format Response` → `Send Response`

34. **Add optional sticky notes** to preserve operational documentation:
    - prerequisites
    - setup steps
    - workflow explanation
    - route section label
    - store/notify section label
    - AI verification section label

35. **Configure credentials**:
    - OpenAI API credentials for GPT-4o
    - Gmail OAuth2 for both Gmail nodes
    - Slack OAuth2 for Slack nodes
    - External auth for KYC, bureau, identity, sanctions APIs
    - Data table access and table IDs

36. **Replace all placeholders**:
    - KYC API endpoint
    - Credit bureau endpoint
    - Identity verification endpoint
    - Sanctions endpoint
    - Risk Slack channel
    - Compliance Slack channel
    - Eligible customers table
    - Ineligible customers table
    - Audit log table
    - Default customer email
    - Verification sub-workflow ID

37. **Test with representative input payloads** for each expected outcome:
    - eligible
    - pending verification
    - requires documentation
    - compliance exception
    - ineligible

38. **Strongly recommended correction before production**:
    - Do not stringify `requiredActions` and `complianceSignals` in the Set nodes if downstream nodes need array/object access.
    - If you must store them as strings, parse them before email/slack templating.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: KYC & Credit Bureau API credentials; Sanctions screening API access; Gmail OAuth2 and Slack bot token; Airtable API key | Canvas note |
| Use case: Fintech platforms automating loan application eligibility screening | Canvas note |
| Customisation: Add extra verification tools such as biometric or document OCR APIs | Canvas note |
| Benefit: Eliminates manual KYC and sanctions review bottlenecks | Canvas note |
| Setup: Set webhook URL and connect Credit Operations webhook node to your intake system | Canvas note |
| Setup: Add OpenAI API key to the OpenAI Chat Model node | Canvas note |
| Setup: Configure KYC, Credit Bureau, Identity, and Sanctions tool credentials | Canvas note |
| Setup: Add Gmail OAuth2 and Slack bot token for notification nodes | Canvas note |
| Setup: Connect Airtable API key; set base/table IDs for eligible and ineligible customer stores | Canvas note |
| Workflow intent: Automates credit operations onboarding with AI-powered orchestration and full audit trail | Canvas note |
| Section label: Route by Eligibility Status — splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths | Canvas note |
| Section label: Store & Notify — persists results and dispatches Gmail or Slack notifications per outcome | Canvas note |
| Section label: KYC, Bureau & Sanctions Checks — validates identity, creditworthiness, and watchlist status before routing | Canvas note |

## Additional implementation notes
- The workflow title provided by the user and the workflow name in JSON differ. The actual workflow JSON name is: **Intelligent credit operations agent for verification screening and compliance**.
- The Data Table nodes are labeled conceptually like Airtable storage in the sticky notes, but the actual node type in the JSON is **Data Table**, not an Airtable node.
- The branch nodes for documentation and compliance appear to have a likely data-shape issue: stringified arrays/objects are later used as native arrays/objects. This should be corrected before deployment.
- The `Trigger Verification Workflow` node is incomplete because no target workflow ID is configured.