Orchestrate credit onboarding checks with GPT-4o, KYC APIs, Gmail, Slack and Airtable

https://n8nworkflows.xyz/workflows/orchestrate-credit-onboarding-checks-with-gpt-4o--kyc-apis--gmail--slack-and-airtable-14694


# Orchestrate credit onboarding checks with GPT-4o, KYC APIs, Gmail, Slack and Airtable

# 1. Workflow Overview

This workflow implements an AI-assisted credit operations orchestration pipeline for onboarding, verification screening, and compliance handling. It receives a POST request, sends the payload to a GPT-4o-powered agent, allows that agent to call verification and communication tools, enforces a structured response schema, routes the case by eligibility outcome, performs the corresponding operational action, logs the result, and returns a JSON response to the caller.

Target use cases include:
- Credit onboarding intake for fintech or lending operations
- KYC and identity verification coordination
- Compliance and sanctions escalation handling
- Documentation follow-up automation
- Audit logging of onboarding outcomes

## 1.1 Input Reception
The workflow starts from a webhook endpoint that accepts POST requests and defers the HTTP response until downstream processing is complete.

## 1.2 AI Coordination and Tooling
A LangChain agent powered by GPT-4o interprets the incoming request body, optionally uses connected tools for KYC, credit bureau, identity, sanctions, email, and Slack actions, and produces structured operational output validated by a schema.

## 1.3 Outcome Routing
The workflow routes the agent’s structured result into one of five branches based on `eligibilityStatus`:
- `eligible`
- `pending_verification`
- `requires_documentation`
- `compliance_exception`
- `ineligible`

## 1.4 Action Execution per Outcome
Each branch prepares normalized data and then performs the branch-specific action:
- Store eligible customers
- Trigger a verification sub-workflow
- Send a documentation request email
- Alert the compliance team in Slack
- Store ineligible customers

## 1.5 Audit Logging and Response
All branches converge into a common audit log preparation step, store an audit record, format a final API response, and send it back to the webhook caller.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block exposes the workflow as an HTTP endpoint for external systems such as onboarding forms, LOS platforms, or internal intake services. It is configured to wait for downstream processing before replying.

### Nodes Involved
- Credit Operations Webhook

### Node Details

#### Credit Operations Webhook
- **Type and technical role:** `n8n-nodes-base.webhook` — entry-point trigger node.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `credit-operations`
  - Response mode: `responseNode`, meaning the workflow must end with a Respond to Webhook node.
- **Key expressions or variables used:**
  - Downstream nodes expect the incoming request body at `{{$json.body}}`.
- **Input and output connections:**
  - No input node
  - Outputs to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `2.1`
- **Edge cases or potential failure types:**
  - Webhook path conflict with another workflow
  - Caller sends malformed JSON or unexpected payload shape
  - If the workflow fails before `Send Response`, caller may receive an error/timeout
- **Sub-workflow reference:** None

---

## 2.2 AI Coordination and Tooling

### Overview
This is the orchestration core. The agent receives the request body, uses GPT-4o with memory and a strict schema, and may invoke connected tool nodes to perform verification checks or communications before producing a structured decision package.

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
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — LangChain AI agent orchestrator.
- **Configuration choices:**
  - Prompt source: `define`
  - User text input: `={{ $json.body }}`
  - System message defines the role as a credit operations coordinator, explicitly forbidding final credit approval or modification of risk models.
  - Structured output enabled via output parser.
- **Key expressions or variables used:**
  - Reads request content from `{{$json.body}}`
  - Produces structured output later accessed as `{{$json.output...}}`
- **Input and output connections:**
  - Main input from `Credit Operations Webhook`
  - AI language model input from `OpenAI Chat Model`
  - AI memory input from `Conversation Memory`
  - AI tool inputs from all tool nodes listed below
  - AI output parser input from `Structured Output Parser`
  - Main output to `Route by Eligibility Status`
- **Version-specific requirements:**
  - Uses node type version `3.1`
  - Requires compatible LangChain nodes in the same n8n version family
- **Edge cases or potential failure types:**
  - Invalid or overly vague incoming body causing weak extraction
  - Model/tool call failure
  - Structured output generation may fail if the model returns invalid schema shape
  - Prompt and response assumptions may break if `body` is not a string/object the model can interpret
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM backend for the agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for more deterministic operational output
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `1.3`
  - Requires OpenAI credentials and model availability in the account
- **Edge cases or potential failure types:**
  - Invalid or expired OpenAI API key
  - Model access restrictions or quota exhaustion
  - Latency or rate limiting
- **Sub-workflow reference:** None

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — schema validator for AI output.
- **Configuration choices:**
  - Manual JSON schema
  - Enforces fields:
    - `requestType`
    - `customerId`
    - `eligibilityStatus`
    - `complianceSignals`
    - `requiredActions`
    - `escalationRequired`
    - optional `escalationReason`
    - `nextSteps`
  - Restricts enums for request type, eligibility status, risk tier, and action types
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output parser connection to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `1.3`
- **Edge cases or potential failure types:**
  - Agent returns malformed JSON
  - Missing required fields
  - Enum mismatch, especially if the prompt wording drifts from the schema
- **Sub-workflow reference:** None

#### Conversation Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short-term conversational memory.
- **Configuration choices:**
  - Context window length: `10`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI memory output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `1.3`
- **Edge cases or potential failure types:**
  - Usually low risk; mostly relevant if multi-turn interactions are expected
  - In a webhook context, memory may provide limited value unless the agent loops internally
- **Sub-workflow reference:** None

#### KYC Verification API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool` — AI-callable HTTP tool for KYC checks.
- **Configuration choices:**
  - Method: `POST`
  - URL: placeholder for external KYC API endpoint
  - Description instructs the AI to provide customer ID and personal data
- **Key expressions or variables used:** None defined in node parameters
- **Input and output connections:**
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `4.4`
- **Edge cases or potential failure types:**
  - Placeholder URL must be replaced
  - Missing auth headers if the API requires them
  - 4xx/5xx responses
  - AI may call the tool without sufficient input fields unless prompt examples are strengthened
- **Sub-workflow reference:** None

#### Credit Bureau API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool` — AI-callable credit bureau query tool.
- **Configuration choices:**
  - Method: `POST`
  - URL: placeholder credit bureau endpoint
  - Tool description mentions customer ID and SSN
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `4.4`
- **Edge cases or potential failure types:**
  - Placeholder endpoint not configured
  - Sensitive data handling issues around SSN
  - Bureau API authorization failures
  - Regulatory concerns if insufficient consent exists upstream
- **Sub-workflow reference:** None

#### Identity Verification Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool` — AI-callable identity verification service tool.
- **Configuration choices:**
  - Method: `POST`
  - URL: placeholder identity verification endpoint
  - Description references government ID and biometric checks
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `4.4`
- **Edge cases or potential failure types:**
  - Placeholder endpoint
  - Missing document payload structure
  - Timeout for biometric/ID validation APIs
- **Sub-workflow reference:** None

#### Sanctions Screening Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool` — AI-callable compliance screening tool.
- **Configuration choices:**
  - Method: `POST`
  - URL: placeholder sanctions screening endpoint
  - Description expects customer name and country
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `4.4`
- **Edge cases or potential failure types:**
  - Placeholder endpoint
  - API rate limits
  - False positives requiring human review
  - Incomplete applicant data causing screening ambiguity
- **Sub-workflow reference:** None

#### Email Communication Tool
- **Type and technical role:** `n8n-nodes-base.gmailTool` — AI-callable Gmail send tool.
- **Configuration choices:**
  - Uses `$fromAI()` for recipient, subject, and body
  - Description positions it for status updates and documentation requests
- **Key expressions or variables used:**
  - `{{$fromAI('toEmail', 'Recipient email address', 'string')}}`
  - `{{$fromAI('subject', 'Email subject line', 'string')}}`
  - `{{$fromAI('message', 'Email message body', 'string')}}`
- **Input and output connections:**
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `2.2`
  - Requires Gmail OAuth2 credentials
- **Edge cases or potential failure types:**
  - OAuth token expiry
  - Gmail sending limits
  - AI-generated invalid email address
- **Sub-workflow reference:** None

#### Risk Team Escalation Tool
- **Type and technical role:** `n8n-nodes-base.slackHitlTool` — AI-callable Slack human-in-the-loop escalation tool.
- **Configuration choices:**
  - Fixed Slack channel placeholder for risk approval/escalation channel
  - Message text supplied by AI through `$fromAI()`
  - Authentication: OAuth2
- **Key expressions or variables used:**
  - `{{$fromAI('escalationMessage', 'Detailed escalation message with customer context and risk factors', 'string')}}`
- **Input and output connections:**
  - Receives nested AI tool connection from `Risk Escalation Slack Tool`
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `2.4`
  - Slack OAuth2 required
- **Edge cases or potential failure types:**
  - Placeholder channel not configured
  - Slack permissions insufficient for target channel
  - HITL behavior may require additional environment support depending on n8n version
- **Sub-workflow reference:** None

#### Risk Escalation Slack Tool
- **Type and technical role:** `n8n-nodes-base.slackTool` — subordinate Slack send tool chained into the HITL escalation tool.
- **Configuration choices:**
  - Message and channel both supplied from AI using `$fromAI()`
  - Description: sends notification to risk team Slack channel
- **Key expressions or variables used:**
  - `{{$fromAI('message', 'Risk escalation message', 'string')}}`
  - `{{$fromAI('channel', 'Risk team Slack channel ID', 'string')}}`
- **Input and output connections:**
  - AI tool output to `Risk Team Escalation Tool`
- **Version-specific requirements:**
  - Uses node type version `2.4`
- **Edge cases or potential failure types:**
  - AI may supply an invalid Slack channel
  - Channel resolution can fail if workspace access is limited
- **Sub-workflow reference:** None

#### Compliance Alert Tool
- **Type and technical role:** `n8n-nodes-base.slackTool` — AI-callable Slack alerting tool for compliance.
- **Configuration choices:**
  - Message and channel are AI-supplied through `$fromAI()`
  - Description targets sanctions hits, PEP matches, and compliance exceptions
- **Key expressions or variables used:**
  - `{{$fromAI('alertMessage', 'Compliance alert message with violation details', 'string')}}`
  - `{{$fromAI('channel', 'Slack channel ID or name for compliance alerts', 'string')}}`
- **Input and output connections:**
  - AI tool output to `Credit Operations Coordinator`
- **Version-specific requirements:**
  - Uses node type version `2.4`
- **Edge cases or potential failure types:**
  - AI chooses wrong channel
  - Slack auth/scope issues
  - Duplicate notifications if the AI tool and the deterministic downstream Slack branch are both used
- **Sub-workflow reference:** None

---

## 2.3 Outcome Routing

### Overview
This block examines the structured AI result and selects one of five deterministic processing paths. It is the control point that turns the AI recommendation into operational execution.

### Nodes Involved
- Route by Eligibility Status

### Node Details

#### Route by Eligibility Status
- **Type and technical role:** `n8n-nodes-base.switch` — branch router.
- **Configuration choices:**
  - Evaluates `{{$json.output.eligibilityStatus}}`
  - Defines five equality rules for:
    - `eligible`
    - `pending_verification`
    - `requires_documentation`
    - `compliance_exception`
    - `ineligible`
- **Key expressions or variables used:**
  - `{{$json.output.eligibilityStatus}}`
- **Input and output connections:**
  - Input from `Credit Operations Coordinator`
  - Output 0 → `Prepare Eligible Customer Data`
  - Output 1 → `Prepare Verification Request`
  - Output 2 → `Prepare Documentation Request`
  - Output 3 → `Prepare Compliance Escalation`
  - Output 4 → `Prepare Ineligible Customer Data`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - If the AI emits a value outside the schema, ideally parser blocks it first
  - If parser is bypassed or malformed output passes, no branch may match
- **Sub-workflow reference:** None

---

## 2.4 Eligible Customer Handling

### Overview
This branch normalizes data for eligible applicants and stores it in a data table before continuing to central audit logging.

### Nodes Involved
- Prepare Eligible Customer Data
- Store Eligible Customers

### Node Details

#### Prepare Eligible Customer Data
- **Type and technical role:** `n8n-nodes-base.set` — field mapping and normalization.
- **Configuration choices:**
  - Creates fields:
    - `customerId`
    - `status = eligible`
    - `eligibilityStatus`
    - `kycComplete`
    - `identityVerified`
    - `sanctionsCheckPassed`
    - `documentsComplete`
    - `riskTier`
    - `processedAt = {{$now.toISO()}}`
    - `nextSteps`
- **Key expressions or variables used:**
  - Reads from `{{$json.output.*}}`
  - Timestamp from `{{$now.toISO()}}`
- **Input and output connections:**
  - Input from `Route by Eligibility Status`
  - Output to `Store Eligible Customers`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Missing `output.complianceSignals.*` fields if upstream schema enforcement fails
- **Sub-workflow reference:** None

#### Store Eligible Customers
- **Type and technical role:** `n8n-nodes-base.dataTable` — persistence to an n8n data table.
- **Configuration choices:**
  - Mapping mode: auto-map input data
  - Data table ID: placeholder for eligible customer table
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Prepare Eligible Customer Data`
  - Output to `Prepare Audit Log`
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - Placeholder table ID not set
  - Table schema mismatch with mapped fields
  - Permissions/storage issues
- **Sub-workflow reference:** None

---

## 2.5 Pending Verification Handling

### Overview
This branch prepares a pending-verification payload and invokes another workflow to continue verification processing. It waits for the sub-workflow to finish before continuing.

### Nodes Involved
- Prepare Verification Request
- Trigger Verification Workflow

### Node Details

#### Prepare Verification Request
- **Type and technical role:** `n8n-nodes-base.set` — packaging of verification handoff data.
- **Configuration choices:**
  - Creates:
    - `customerId`
    - `status = pending_verification`
    - `eligibilityStatus`
    - `requiredActions`
    - `complianceSignals`
    - `processedAt`
    - `nextSteps`
  - Notably uses `JSON.stringify(...)` for `requiredActions` and `complianceSignals`, despite assigning types `array` and `object`
- **Key expressions or variables used:**
  - `{{$json.output.customerId}}`
  - `{{ JSON.stringify($json.output.requiredActions) }}`
  - `{{ JSON.stringify($json.output.complianceSignals) }}`
  - `{{$now.toISO()}}`
- **Input and output connections:**
  - Input from `Route by Eligibility Status`
  - Output to `Trigger Verification Workflow`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Potential type mismatch because stringified JSON is placed into array/object-typed assignments
  - Downstream sub-workflow may receive strings instead of structured objects
- **Sub-workflow reference:** Handoff to `Trigger Verification Workflow`

#### Trigger Verification Workflow
- **Type and technical role:** `n8n-nodes-base.executeWorkflow` — sub-workflow invocation.
- **Configuration choices:**
  - `waitForSubWorkflow: true`
  - Workflow ID is empty in the provided export and must be configured
- **Key expressions or variables used:** None visible
- **Input and output connections:**
  - Input from `Prepare Verification Request`
  - Output to `Prepare Audit Log`
- **Version-specific requirements:**
  - Uses node type version `1.3`
- **Edge cases or potential failure types:**
  - No target workflow selected, so this node will fail as-is
  - Sub-workflow input contract undefined in this workflow and must be designed
  - If sub-workflow throws, the main workflow will fail before responding
- **Sub-workflow reference:** Yes — target verification workflow must be provided manually

---

## 2.6 Documentation Request Handling

### Overview
This branch handles applicants who need to submit more documents. It prepares a request payload and sends an email via Gmail.

### Nodes Involved
- Prepare Documentation Request
- Send Documentation Request Email

### Node Details

#### Prepare Documentation Request
- **Type and technical role:** `n8n-nodes-base.set` — mapping for documentation follow-up.
- **Configuration choices:**
  - Creates:
    - `customerId`
    - `status = requires_documentation`
    - `eligibilityStatus`
    - `requiredActions`
    - `complianceSignals`
    - `processedAt`
    - `nextSteps`
  - Also stringifies `requiredActions` and `complianceSignals`
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.output.requiredActions) }}`
  - `{{ JSON.stringify($json.output.complianceSignals) }}`
  - `{{$now.toISO()}}`
- **Input and output connections:**
  - Input from `Route by Eligibility Status`
  - Output to `Send Documentation Request Email`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Same stringification/type inconsistency as the verification branch
  - Email node later expects array/object behavior, which can break if these fields are strings
- **Sub-workflow reference:** None

#### Send Documentation Request Email
- **Type and technical role:** `n8n-nodes-base.gmail` — deterministic Gmail email send node.
- **Configuration choices:**
  - Recipient: `{{$json.customerEmail || '<__PLACEHOLDER_VALUE__customer_email__>'}}`
  - Subject: `Documentation Required for Credit Application`
  - Attribution disabled
  - Message template references:
    - `customerId`
    - `nextSteps`
    - `requiredActions.map(...)`
- **Key expressions or variables used:**
  - `{{$json.customerEmail || '<__PLACEHOLDER_VALUE__customer_email__>'}}`
  - Body expression uses JavaScript template syntax and array mapping:
    - `{{ $json.requiredActions.map(action => '- ' + action.details).join('\n') }}`
- **Input and output connections:**
  - Input from `Prepare Documentation Request`
  - Output to `Prepare Audit Log`
- **Version-specific requirements:**
  - Uses node type version `2.2`
  - Gmail OAuth2 required
- **Edge cases or potential failure types:**
  - As currently built, `customerEmail` is never set in the preparation node
  - Fallback placeholder recipient must be replaced or email will fail/misroute
  - If `requiredActions` is a string rather than an array, `.map(...)` will fail
  - Gmail auth/token issues
- **Sub-workflow reference:** None

---

## 2.7 Compliance Escalation Handling

### Overview
This branch prepares a compliance exception record and posts a formatted alert to Slack for human action.

### Nodes Involved
- Prepare Compliance Escalation
- Alert Compliance Team

### Node Details

#### Prepare Compliance Escalation
- **Type and technical role:** `n8n-nodes-base.set` — normalization of compliance escalation data.
- **Configuration choices:**
  - Creates:
    - `customerId`
    - `status = compliance_exception`
    - `eligibilityStatus`
    - `escalationRequired`
    - `escalationReason`
    - `complianceSignals`
    - `requiredActions`
    - `processedAt`
    - `nextSteps`
  - Again stringifies `complianceSignals` and `requiredActions`
- **Key expressions or variables used:**
  - `{{$json.output.escalationRequired}}`
  - `{{$json.output.escalationReason}}`
  - `{{ JSON.stringify($json.output.complianceSignals) }}`
  - `{{ JSON.stringify($json.output.requiredActions) }}`
- **Input and output connections:**
  - Input from `Route by Eligibility Status`
  - Output to `Alert Compliance Team`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Stringification breaks downstream object access
  - Missing `escalationReason` may be acceptable per schema, but message quality degrades
- **Sub-workflow reference:** None

#### Alert Compliance Team
- **Type and technical role:** `n8n-nodes-base.slack` — deterministic Slack message sender.
- **Configuration choices:**
  - Sends to a fixed compliance Slack channel placeholder
  - Text template includes:
    - `customerId`
    - `eligibilityStatus`
    - `complianceSignals.riskTier`
    - KYC/identity/sanctions/document flags
    - `nextSteps`
    - `requiredActions.map(...)`
- **Key expressions or variables used:**
  - Body references nested properties such as `{{$json.complianceSignals.riskTier}}`
  - Uses `{{$json.requiredActions.map(...)}}`
- **Input and output connections:**
  - Input from `Prepare Compliance Escalation`
  - Output to `Prepare Audit Log`
- **Version-specific requirements:**
  - Uses node type version `2.4`
  - Slack OAuth2 required
- **Edge cases or potential failure types:**
  - This branch is likely broken as exported if `complianceSignals` and `requiredActions` remain JSON strings rather than object/array values
  - Placeholder channel must be replaced
  - Slack auth/scope issues
- **Sub-workflow reference:** None

---

## 2.8 Ineligible Customer Handling

### Overview
This branch stores normalized data for ineligible applicants for later review or reporting.

### Nodes Involved
- Prepare Ineligible Customer Data
- Store Ineligible Customers

### Node Details

#### Prepare Ineligible Customer Data
- **Type and technical role:** `n8n-nodes-base.set` — field mapping for ineligible outcomes.
- **Configuration choices:**
  - Creates:
    - `customerId`
    - `status = ineligible`
    - `eligibilityStatus`
    - `complianceSignals`
    - `processedAt`
    - `nextSteps`
  - `complianceSignals` is stringified
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.output.complianceSignals) }}`
  - `{{$now.toISO()}}`
- **Input and output connections:**
  - Input from `Route by Eligibility Status`
  - Output to `Store Ineligible Customers`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Stringification may be acceptable for storage, but later consumers must know it is JSON text
- **Sub-workflow reference:** None

#### Store Ineligible Customers
- **Type and technical role:** `n8n-nodes-base.dataTable` — persistence node.
- **Configuration choices:**
  - Auto-map input data
  - Data table ID placeholder for ineligible customer storage
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Prepare Ineligible Customer Data`
  - Output to `Prepare Audit Log`
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - Table ID not configured
  - Schema mismatch
- **Sub-workflow reference:** None

---

## 2.9 Audit Logging and Final Response

### Overview
All deterministic branches converge here. The workflow prepares a unified audit record, writes it to a data table, formats a concise API response, and returns it to the original webhook caller.

### Nodes Involved
- Prepare Audit Log
- Log All Operations
- Format Response
- Send Response

### Node Details

#### Prepare Audit Log
- **Type and technical role:** `n8n-nodes-base.set` — common log record assembler.
- **Configuration choices:**
  - Creates:
    - `customerId`
    - `requestType` from the AI coordinator output
    - `eligibilityStatus` or fallback `status`
    - `processedAt`
    - `complianceSignals`
    - `requiredActions`
    - `escalationRequired` default false
    - `escalationReason` default `N/A`
    - `nextSteps`
  - References another node directly using n8n expression lookup
- **Key expressions or variables used:**
  - `{{ $('Credit Operations Coordinator').item.json.output.requestType }}`
  - `{{$json.eligibilityStatus || $json.status}}`
  - `{{ JSON.stringify($json.complianceSignals) }}`
  - `{{ JSON.stringify($json.requiredActions || []) }}`
- **Input and output connections:**
  - Inputs from:
    - `Store Eligible Customers`
    - `Trigger Verification Workflow`
    - `Send Documentation Request Email`
    - `Alert Compliance Team`
    - `Store Ineligible Customers`
  - Output to `Log All Operations`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - If upstream fields are already strings, double-stringification may occur
  - Cross-node expression to `Credit Operations Coordinator` assumes matching item context remains valid
- **Sub-workflow reference:** None

#### Log All Operations
- **Type and technical role:** `n8n-nodes-base.dataTable` — centralized audit persistence.
- **Configuration choices:**
  - Auto-map input data
  - Data table ID placeholder for audit log table
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Prepare Audit Log`
  - Output to `Format Response`
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - Placeholder table ID not configured
  - Data table schema mismatch
- **Sub-workflow reference:** None

#### Format Response
- **Type and technical role:** `n8n-nodes-base.set` — creates response payload.
- **Configuration choices:**
  - Sets:
    - `success = true`
    - `customerId`
    - `status`
    - `message`
    - `processedAt`
    - `escalationRequired`
- **Key expressions or variables used:**
  - `{{$json.eligibilityStatus || $json.status}}`
  - `{{$json.nextSteps}}`
  - `{{$json.escalationRequired || false}}`
- **Input and output connections:**
  - Input from `Log All Operations`
  - Output to `Send Response`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - If previous nodes fail, no response is returned
- **Sub-workflow reference:** None

#### Send Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook` — final HTTP response node.
- **Configuration choices:**
  - Respond with `json`
  - Response body: `={{ $json }}`
- **Key expressions or variables used:**
  - `{{$json}}`
- **Input and output connections:**
  - Input from `Format Response`
  - No output
- **Version-specific requirements:**
  - Uses node type version `1.1`
- **Edge cases or potential failure types:**
  - If any prior branch errors, webhook caller may receive execution failure instead of this structured response
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Credit Operations Webhook | webhook | Receives POST onboarding/credit operations requests |  | Credit Operations Coordinator | ## How It Works<br>This workflow automates credit operations onboarding by running KYC verification, credit bureau checks, identity validation, and sanctions screening through a single AI-powered agent. Built for credit operations teams, compliance officers, and fintech platforms, it eliminates manual eligibility reviews that are slow and error-prone. Triggered via webhook, the Credit Operations Agent orchestrates all verification tools simultaneously, then routes customers by eligibility status, eligible, ineligible, pending documentation, or compliance escalation. Each path prepares structured data stored in Airtable, triggers appropriate follow-up actions (email, Slack alerts), and logs a full audit trail. A final formatted response is returned to the originating system, closing the loop end-to-end with no manual handoffs.<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Credit Operations Coordinator | @n8n/n8n-nodes-langchain.agent | AI orchestration of verification and action planning | Credit Operations Webhook | Route by Eligibility Status | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model backend for the AI agent |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured AI output schema |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Provides short-term memory to the agent |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| KYC Verification API Tool | httpRequestTool | AI-callable KYC verification API |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## Setup Steps<br>1. Set webhook URL and connect Credit Operations webhook node to your intake system.<br>2. Add OpenAI API key to the OpenAI Chat Model node.<br>3. Configure KYC, Credit Bureau, Identity, and Sanctions tool credentials.<br>4. Add Gmail OAuth2 and Slack bot token for notification nodes.<br>5. Connect Airtable API key; set base/table IDs for eligible and ineligible customer stores.<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Credit Bureau API Tool | httpRequestTool | AI-callable credit bureau lookup API |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## Setup Steps<br>1. Set webhook URL and connect Credit Operations webhook node to your intake system.<br>2. Add OpenAI API key to the OpenAI Chat Model node.<br>3. Configure KYC, Credit Bureau, Identity, and Sanctions tool credentials.<br>4. Add Gmail OAuth2 and Slack bot token for notification nodes.<br>5. Connect Airtable API key; set base/table IDs for eligible and ineligible customer stores.<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Identity Verification Tool | httpRequestTool | AI-callable identity verification API |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## Setup Steps<br>1. Set webhook URL and connect Credit Operations webhook node to your intake system.<br>2. Add OpenAI API key to the OpenAI Chat Model node.<br>3. Configure KYC, Credit Bureau, Identity, and Sanctions tool credentials.<br>4. Add Gmail OAuth2 and Slack bot token for notification nodes.<br>5. Connect Airtable API key; set base/table IDs for eligible and ineligible customer stores.<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Sanctions Screening Tool | httpRequestTool | AI-callable sanctions and watchlist screening API |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Email Communication Tool | gmailTool | AI-callable outbound customer email tool |  | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Risk Team Escalation Tool | slackHitlTool | AI-callable human-in-the-loop escalation toward risk team | Risk Escalation Slack Tool | Credit Operations Coordinator | ## Prerequisites<br>- KYC & Credit Bureau API credentials<br>- Sanctions screening API access<br>- Gmail OAuth2 and Slack bot token<br>- Airtable API key<br>## Use Cases<br>- Fintech platforms automating loan application eligibility screening<br>## Customisation<br>- Add extra verification tools (e.g., biometric or document OCR APIs)<br>## Benefits<br>- Eliminates manual KYC and sanctions review bottlenecks<br>## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Compliance Alert Tool | slackTool | AI-callable compliance Slack alert tool |  | Credit Operations Coordinator | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Route by Eligibility Status | switch | Branches execution by AI eligibility result | Credit Operations Coordinator | Prepare Eligible Customer Data; Prepare Verification Request; Prepare Documentation Request; Prepare Compliance Escalation; Prepare Ineligible Customer Data | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Eligible Customer Data | set | Normalizes eligible-case fields | Route by Eligibility Status | Store Eligible Customers | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Verification Request | set | Packages pending-verification handoff data | Route by Eligibility Status | Trigger Verification Workflow | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Documentation Request | set | Packages documentation follow-up data | Route by Eligibility Status | Send Documentation Request Email | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Prepare Compliance Escalation | set | Normalizes compliance exception data | Route by Eligibility Status | Alert Compliance Team | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Store Eligible Customers | dataTable | Persists eligible customer records | Prepare Eligible Customer Data | Prepare Audit Log | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Trigger Verification Workflow | executeWorkflow | Calls a verification sub-workflow and waits for completion | Prepare Verification Request | Prepare Audit Log | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Send Documentation Request Email | gmail | Sends deterministic documentation request email | Prepare Documentation Request | Prepare Audit Log | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Alert Compliance Team | slack | Sends deterministic Slack alert to compliance team | Prepare Compliance Escalation | Prepare Audit Log | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Log All Operations | dataTable | Writes unified audit log record | Prepare Audit Log | Format Response | ### Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Prepare Audit Log | set | Builds a branch-independent audit payload | Store Eligible Customers; Trigger Verification Workflow; Send Documentation Request Email; Alert Compliance Team; Store Ineligible Customers | Log All Operations | ### Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Send Response | respondToWebhook | Returns final JSON response to HTTP caller | Format Response |  | ### Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Format Response | set | Shapes the final webhook response body | Log All Operations | Send Response | ### Store & Notify<br>**Why:** Persists results to Airtable and dispatches Gmail or Slack notifications per outcome. |
| Risk Escalation Slack Tool | slackTool | Nested Slack send tool used by the risk escalation HITL tool |  | Risk Team Escalation Tool | ## KYC, Bureau & Sanctions Checks<br>**Why:** Validates identity, creditworthiness, and watchlist status before any routing decision. |
| Prepare Ineligible Customer Data | set | Normalizes ineligible-case fields | Route by Eligibility Status | Store Ineligible Customers | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Store Ineligible Customers | dataTable | Persists ineligible customer records | Prepare Ineligible Customer Data | Prepare Audit Log | ## Route by Eligibility Status<br>**Why:** Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths. |
| Sticky Note | stickyNote | Canvas annotation for prerequisites, use cases, and benefits |  |  |  |
| Sticky Note1 | stickyNote | Canvas annotation for setup guidance |  |  |  |
| Sticky Note2 | stickyNote | Canvas annotation describing overall behavior |  |  |  |
| Sticky Note3 | stickyNote | Canvas annotation for persistence and notification area |  |  |  |
| Sticky Note4 | stickyNote | Canvas annotation for routing area |  |  |  |
| Sticky Note5 | stickyNote | Canvas annotation for AI verification and screening area |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:
   - `Intelligent credit operations agent for verification screening and compliance`

2. **Add a Webhook node** named `Credit Operations Webhook`.
   - Type: `Webhook`
   - HTTP method: `POST`
   - Path: `credit-operations`
   - Response mode: `Using Respond to Webhook node`
   - This node will be the main entry point.

3. **Add an AI Agent node** named `Credit Operations Coordinator`.
   - Type: `AI Agent` / `LangChain Agent`
   - Set the input text to:
     - `={{ $json.body }}`
   - Set prompt type to manually defined/system prompt mode.
   - Paste this system behavior in substance:
     - AI acts as a Credit Operations Coordinator
     - It validates onboarding and compliance signals
     - It can coordinate KYC, identity, credit review, servicing, communications
     - It cannot approve credit or alter risk models
     - It must escalate final approvals and compliance exceptions to humans
     - It should return a structured action plan
   - Enable structured output.

4. **Connect `Credit Operations Webhook` to `Credit Operations Coordinator`.**

5. **Add an OpenAI Chat Model node** named `OpenAI Chat Model`.
   - Type: `OpenAI Chat Model`
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Add OpenAI credentials:
     - Create or select an `OpenAI API` credential
     - Ensure the account can access `gpt-4o`
   - Connect it to the agent’s **AI Language Model** input.

6. **Add a Memory node** named `Conversation Memory`.
   - Type: `Buffer Window Memory`
   - Context window length: `10`
   - Connect it to the agent’s **AI Memory** input.

7. **Add a Structured Output Parser node** named `Structured Output Parser`.
   - Type: `Structured Output Parser`
   - Schema type: manual
   - Define a JSON schema with these top-level fields:
     - `requestType`: enum of `onboarding`, `kyc_verification`, `credit_review`, `servicing`, `compliance_check`
     - `customerId`: string
     - `eligibilityStatus`: enum of `eligible`, `pending_verification`, `requires_documentation`, `compliance_exception`, `ineligible`
     - `complianceSignals` object with:
       - `kycComplete`: boolean
       - `identityVerified`: boolean
       - `sanctionsCheckPassed`: boolean
       - `documentsComplete`: boolean
       - `riskTier`: enum `low`, `medium`, `high`, `critical`
     - `requiredActions`: array of objects with:
       - `action`: enum including `trigger_kyc`, `verify_identity`, `request_documents`, `credit_bureau_check`, `compliance_review`, `send_notification`, `escalate_to_risk`, `escalate_to_compliance`
       - `priority`: enum `immediate`, `high`, `medium`, `low`
       - `details`: string
     - `escalationRequired`: boolean
     - `escalationReason`: string
     - `nextSteps`: string
   - Mark required fields as in the export.
   - Connect it to the agent’s **AI Output Parser** input.

8. **Add an HTTP Request Tool node** named `KYC Verification API Tool`.
   - Type: `HTTP Request Tool`
   - Method: `POST`
   - URL: your real KYC API endpoint
   - Tool description should explain that it performs KYC verification and expects customer ID and personal information.
   - If the API requires auth:
     - Add headers or credentials as needed
   - Connect it to the agent’s **AI Tool** input.

9. **Add another HTTP Request Tool** named `Credit Bureau API Tool`.
   - Method: `POST`
   - URL: your credit bureau endpoint
   - Description: retrieves credit history and score using customer ID and SSN
   - Add auth headers/credential config
   - Connect to the agent’s **AI Tool** input.

10. **Add another HTTP Request Tool** named `Identity Verification Tool`.
    - Method: `POST`
    - URL: your identity verification endpoint
    - Description: verifies identity using government ID and biometric checks
    - Connect to the agent’s **AI Tool** input.

11. **Add another HTTP Request Tool** named `Sanctions Screening Tool`.
    - Method: `POST`
    - URL: your sanctions/watchlist endpoint
    - Description: screens against sanctions, PEP, and watchlists
    - Connect to the agent’s **AI Tool** input.

12. **Add a Gmail Tool node** named `Email Communication Tool`.
    - Type: `Gmail Tool`
    - Configure:
      - To: `={{ $fromAI('toEmail', 'Recipient email address', 'string') }}`
      - Subject: `={{ $fromAI('subject', 'Email subject line', 'string') }}`
      - Message: `={{ $fromAI('message', 'Email message body', 'string') }}`
    - Add Gmail OAuth2 credentials
    - Connect to the agent’s **AI Tool** input.

13. **Add a Slack Tool node** named `Risk Escalation Slack Tool`.
    - Type: `Slack Tool`
    - Configure:
      - Text: `={{ $fromAI('message', 'Risk escalation message', 'string') }}`
      - Channel: `={{ $fromAI('channel', 'Risk team Slack channel ID', 'string') }}`
      - Authentication: OAuth2
    - Add Slack OAuth2 credentials

14. **Add a Slack HITL Tool node** named `Risk Team Escalation Tool`.
    - Type: `Slack Human-in-the-Loop Tool`
    - Select channel mode
    - Channel ID: fixed risk team channel
    - Message:
      - `={{ $fromAI('escalationMessage', 'Detailed escalation message with customer context and risk factors', 'string') }}`
    - Authentication: OAuth2
    - Connect `Risk Escalation Slack Tool` to this node as an AI tool input.
    - Connect `Risk Team Escalation Tool` to the agent’s **AI Tool** input.

15. **Add another Slack Tool node** named `Compliance Alert Tool`.
    - Text: `={{ $fromAI('alertMessage', 'Compliance alert message with violation details', 'string') }}`
    - Channel: `={{ $fromAI('channel', 'Slack channel ID or name for compliance alerts', 'string') }}`
    - Authentication: OAuth2
    - Add Slack OAuth2 credentials
    - Connect to the agent’s **AI Tool** input.

16. **Add a Switch node** named `Route by Eligibility Status`.
    - Expression to evaluate: `={{ $json.output.eligibilityStatus }}`
    - Create 5 rules:
      1. equals `eligible`
      2. equals `pending_verification`
      3. equals `requires_documentation`
      4. equals `compliance_exception`
      5. equals `ineligible`
    - Connect `Credit Operations Coordinator` to this switch.

17. **Create the eligible branch Set node** named `Prepare Eligible Customer Data`.
    - Fields:
      - `customerId` = `={{ $json.output.customerId }}`
      - `status` = `eligible`
      - `eligibilityStatus` = `={{ $json.output.eligibilityStatus }}`
      - `kycComplete` = `={{ $json.output.complianceSignals.kycComplete }}`
      - `identityVerified` = `={{ $json.output.complianceSignals.identityVerified }}`
      - `sanctionsCheckPassed` = `={{ $json.output.complianceSignals.sanctionsCheckPassed }}`
      - `documentsComplete` = `={{ $json.output.complianceSignals.documentsComplete }}`
      - `riskTier` = `={{ $json.output.complianceSignals.riskTier }}`
      - `processedAt` = `={{ $now.toISO() }}`
      - `nextSteps` = `={{ $json.output.nextSteps }}`
    - Connect switch output `eligible` to this node.

18. **Add a Data Table node** named `Store Eligible Customers`.
    - Mapping mode: auto-map input data
    - Select or create a table for eligible customer records
    - Connect from `Prepare Eligible Customer Data`

19. **Create the pending verification branch Set node** named `Prepare Verification Request`.
    - Fields:
      - `customerId`
      - `status = pending_verification`
      - `eligibilityStatus`
      - `requiredActions`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`
    - Important recommendation:
      - Prefer storing `requiredActions` and `complianceSignals` as real JSON values, not stringified JSON, unless your sub-workflow explicitly expects strings.
    - Connect switch output `pending_verification` to this node.

20. **Add an Execute Workflow node** named `Trigger Verification Workflow`.
    - Enable `Wait for sub-workflow to finish`
    - Select the target verification workflow
    - Connect from `Prepare Verification Request`

21. **Design the verification sub-workflow** expected by `Trigger Verification Workflow`.
    - Recommended input contract:
      - `customerId`
      - `status`
      - `eligibilityStatus`
      - `requiredActions`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`
    - Recommended output contract:
      - Return updated verification status and any resolution metadata
    - Ensure the parent workflow can continue even if no major transformation is returned.

22. **Create the documentation branch Set node** named `Prepare Documentation Request`.
    - Fields:
      - `customerId`
      - `status = requires_documentation`
      - `eligibilityStatus`
      - `requiredActions`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`
    - Recommended improvement:
      - Also add `customerEmail` from the webhook payload if available, for example from original request data.
    - Connect switch output `requires_documentation` to this node.

23. **Add a Gmail node** named `Send Documentation Request Email`.
    - Recipient:
      - Prefer `={{ $json.customerEmail }}`
      - Avoid leaving a placeholder in production
    - Subject:
      - `Documentation Required for Credit Application`
    - Message body should include:
      - Customer ID
      - `nextSteps`
      - list of `requiredActions`
    - Add Gmail OAuth2 credentials
    - Connect from `Prepare Documentation Request`

24. **Ensure `requiredActions` stays an array** for the documentation email.
    - The exported workflow uses `.map(...)` in the email body.
    - If you stringify `requiredActions` in the Set node, the email node will fail.
    - Best practice: keep it as an array or parse it back before email sending.

25. **Create the compliance branch Set node** named `Prepare Compliance Escalation`.
    - Fields:
      - `customerId`
      - `status = compliance_exception`
      - `eligibilityStatus`
      - `escalationRequired`
      - `escalationReason`
      - `complianceSignals`
      - `requiredActions`
      - `processedAt`
      - `nextSteps`
    - Connect switch output `compliance_exception` to this node.

26. **Add a Slack node** named `Alert Compliance Team`.
    - Authentication: Slack OAuth2
    - Send to a fixed compliance channel
    - Message should include:
      - Customer ID
      - eligibility status
      - risk tier
      - compliance signal flags
      - escalation reason
      - next steps
      - required action list
    - Connect from `Prepare Compliance Escalation`

27. **Ensure `complianceSignals` remains an object and `requiredActions` remains an array** in the compliance branch.
    - The exported message template expects nested property access and array `.map(...)`.
    - If you stringify these fields, Slack message rendering will fail.

28. **Create the ineligible branch Set node** named `Prepare Ineligible Customer Data`.
    - Fields:
      - `customerId`
      - `status = ineligible`
      - `eligibilityStatus`
      - `complianceSignals`
      - `processedAt`
      - `nextSteps`
    - Connect switch output `ineligible` to this node.

29. **Add a Data Table node** named `Store Ineligible Customers`.
    - Mapping mode: auto-map input data
    - Select or create a table for ineligible customer records
    - Connect from `Prepare Ineligible Customer Data`

30. **Add a Set node** named `Prepare Audit Log`.
    - This node should receive inputs from all five branches:
      - `Store Eligible Customers`
      - `Trigger Verification Workflow`
      - `Send Documentation Request Email`
      - `Alert Compliance Team`
      - `Store Ineligible Customers`
    - Create fields:
      - `customerId = {{$json.customerId}}`
      - `requestType = {{ $('Credit Operations Coordinator').item.json.output.requestType }}`
      - `eligibilityStatus = {{$json.eligibilityStatus || $json.status}}`
      - `processedAt = {{$json.processedAt}}`
      - `complianceSignals`
      - `requiredActions`
      - `escalationRequired = {{$json.escalationRequired || false}}`
      - `escalationReason = {{$json.escalationReason || 'N/A'}}`
      - `nextSteps = {{$json.nextSteps}}`
    - Recommendation:
      - Decide whether the audit table should store raw JSON objects or JSON strings, and keep that consistent across all branches.

31. **Add a Data Table node** named `Log All Operations`.
    - Mapping mode: auto-map input data
    - Select or create the audit log table
    - Connect from `Prepare Audit Log`

32. **Add a Set node** named `Format Response`.
    - Fields:
      - `success = true`
      - `customerId = {{$json.customerId}}`
      - `status = {{$json.eligibilityStatus || $json.status}}`
      - `message = {{$json.nextSteps}}`
      - `processedAt = {{$json.processedAt}}`
      - `escalationRequired = {{$json.escalationRequired || false}}`
    - Connect from `Log All Operations`

33. **Add a Respond to Webhook node** named `Send Response`.
    - Respond with: `JSON`
    - Response body: `={{ $json }}`
    - Connect from `Format Response`

34. **Configure all credentials and placeholders.**
    - Replace all placeholder API endpoints
    - Replace all placeholder Slack channel IDs
    - Replace all placeholder data table IDs
    - Select the actual verification sub-workflow in `Trigger Verification Workflow`
    - Configure OpenAI, Gmail OAuth2, and Slack OAuth2 credentials

35. **Test each branch independently.**
    - Send test webhook payloads that force:
      - `eligible`
      - `pending_verification`
      - `requires_documentation`
      - `compliance_exception`
      - `ineligible`
    - Verify:
      - storage works
      - notification works
      - audit log is written
      - webhook receives final JSON response

36. **Recommended hardening before production.**
    - Add validation for required webhook fields before the AI step
    - Add error handling branches for API failures and notification failures
    - Add default routing if no switch rule matches
    - Normalize all object/array fields consistently instead of mixing raw objects and stringified JSON
    - Include `customerEmail` explicitly if documentation emails are required

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: KYC & Credit Bureau API credentials; sanctions screening API access; Gmail OAuth2 and Slack bot token; Airtable API key | Workflow canvas notes |
| Use case: Fintech platforms automating loan application eligibility screening | Workflow canvas notes |
| Customisation: Add extra verification tools such as biometric or document OCR APIs | Workflow canvas notes |
| Benefit: Eliminates manual KYC and sanctions review bottlenecks | Workflow canvas notes |
| Setup steps: set webhook URL, configure OpenAI, verification APIs, Gmail, Slack, and Airtable/data-table targets | Workflow canvas notes |
| Workflow behavior summary: AI-driven credit onboarding orchestration with routing, storage, notifications, audit trail, and final webhook response | Workflow canvas notes |
| Store & Notify note: Persists results and dispatches Gmail/Slack notifications per outcome | Workflow canvas notes |
| Route by Eligibility Status note: Splits outcomes into eligible, ineligible, documentation-required, or compliance-escalation paths | Workflow canvas notes |
| KYC, Bureau & Sanctions Checks note: Validates identity, creditworthiness, and watchlist status before routing | Workflow canvas notes |

## Additional implementation observations
- The title says Airtable, but the exported workflow uses **n8n Data Table** nodes, not Airtable nodes.
- The workflow includes **one explicit entry point**: `Credit Operations Webhook`.
- The workflow includes **one explicit sub-workflow dependency**: `Trigger Verification Workflow`, but its target workflow is not configured in the export.
- Several `Set` nodes stringify arrays/objects and later downstream nodes treat them as arrays/objects. This should be corrected before deployment.