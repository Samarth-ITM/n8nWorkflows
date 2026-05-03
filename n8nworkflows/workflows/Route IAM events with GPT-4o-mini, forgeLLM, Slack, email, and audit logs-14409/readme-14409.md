Route IAM events with GPT-4o-mini, forgeLLM, Slack, email, and audit logs

https://n8nworkflows.xyz/workflows/route-iam-events-with-gpt-4o-mini--forgellm--slack--email--and-audit-logs-14409


# Route IAM events with GPT-4o-mini, forgeLLM, Slack, email, and audit logs

# 1. Workflow Overview

This workflow automates IAM event governance from intake to decision routing, notification, and audit storage. It receives IAM events over HTTP, evaluates them with a primary AI governance agent, enriches the analysis through a secondary validation agent and external tools, then routes the event into approved, revoked, or escalated storage paths before returning a JSON response.

Typical use cases include:
- Automated review of IAM permission changes
- Access approval or revocation decisions
- Compliance-aware escalation of risky IAM events
- Audit logging for governance and security teams

## 1.1 Input Reception

The workflow starts with a webhook that accepts IAM event payloads via `POST /iam-events`. The webhook does not answer immediately; instead, the workflow finishes by sending a structured JSON response through a dedicated response node.

## 1.2 AI Governance Evaluation

A central LangChain agent acts as the orchestrator. It reads the incoming event payload, uses a governance system prompt, relies on a chat model plus conversational memory, and can call multiple tools:
- Access Signal Agent for event validation
- forgeLLM API Tool for specialized IAM analysis
- Slack Notification Tool
- Email Notification Tool
- Audit Log Tool
- Compliance Query Tool

Its output is constrained by a structured parser so downstream routing can safely depend on a predictable schema.

## 1.3 Validation and Structured AI Outputs

A secondary AI tool-agent validates IAM event content, checks required fields, risk, anomalies, and compliance flags. Both the primary and secondary agents use structured output parsers to normalize AI responses.

## 1.4 Decision Routing

After the governance agent returns a decision, a Switch node routes the item by `approved`, `revoked`, `escalated`, or `pending`. In practice, only three downstream branches are implemented:
- Approved
- Revoked
- Escalated

There is a `pending` rule in the router, but no connected handling path.

## 1.5 Storage and Response

Each implemented decision branch prepares a normalized record with selected event and decision fields, stores it in a corresponding data table, and then returns the final JSON payload to the webhook caller.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Receive & Start Workflow

### Overview
This block exposes the public entry point for IAM events. It receives the raw HTTP payload and forwards it to the governance agent for analysis.

### Nodes Involved
- Receive IAM Event

### Node Details

#### Receive IAM Event
- **Type and technical role:** `n8n-nodes-base.webhook` — workflow trigger and HTTP entry point.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `iam-events`
  - Response mode: `responseNode`, meaning the webhook waits for a later Respond to Webhook node.
- **Key expressions or variables used:** None in parameters, but downstream nodes consume the request data from `$json.body`.
- **Input and output connections:**
  - No input; this is an entry node.
  - Output goes to `Governance Agent`.
- **Version-specific requirements:** Uses `typeVersion 2.1`; response-node mode must be supported by the installed n8n version.
- **Edge cases or potential failure types:**
  - Invalid or unexpected request body shape
  - Caller timeout if downstream AI/tool execution is slow
  - Webhook path conflicts with other workflows
  - If `body` is missing or malformed, expressions in later nodes may fail or produce poor AI output
- **Sub-workflow reference:** None.

---

## 2.2 Block: Governance Orchestration

### Overview
This is the workflow’s main intelligence layer. The governance agent interprets the IAM event, consults validation and external tools, then emits a structured decision used for routing and persistence.

### Nodes Involved
- Governance Agent
- Governance Model
- Conversation Memory
- Governance Output Parser

### Node Details

#### Governance Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — main LangChain-style orchestration agent.
- **Configuration choices:**
  - Prompt type: defined manually
  - Input text: `={{ $json.body }}`
  - System message positions the agent as an IAM governance orchestrator that validates events, decides approvals/revocations/escalations, and generates compliance reporting behavior.
  - Output parser enabled.
- **Key expressions or variables used:**
  - `{{$json.body}}` as the event content
- **Input and output connections:**
  - Main input from `Receive IAM Event`
  - AI language model from `Governance Model`
  - AI memory from `Conversation Memory`
  - AI tools from:
    - `Access Signal Agent`
    - `forgeLLM API Tool`
    - `Slack Notification Tool`
    - `Email Notification Tool`
    - `Audit Log Tool`
    - `Compliance Query Tool`
  - AI output parser from `Governance Output Parser`
  - Main output to `Route by Decision`
- **Version-specific requirements:** Uses `typeVersion 3.1`; requires LangChain-capable n8n version and compatible AI node packages.
- **Edge cases or potential failure types:**
  - LLM returning content that fails structured parsing
  - Tool invocation errors propagating into incomplete decisions
  - Ambiguous event body causing hallucinated fields or poor decisions
  - Long-running execution if multiple tools are called
  - If `body` is an object rather than a serialized string, behavior depends on the node’s internal coercion
- **Sub-workflow reference:** None.

#### Governance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — primary OpenAI chat model backing the governance agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.2` for relatively deterministic outputs with limited creativity
  - No built-in tools configured here
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 1.3`; requires valid OpenAI credentials and availability of `gpt-4o-mini` in the account/environment.
- **Edge cases or potential failure types:**
  - Invalid OpenAI API key
  - Model access not enabled
  - Rate limiting or quota exhaustion
  - Timeout or provider-side transient errors
- **Sub-workflow reference:** None.

#### Conversation Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — memory component for the governance agent.
- **Configuration choices:**
  - No explicit custom settings shown; defaults apply.
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI memory output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - Memory may be ineffective depending on execution context and session handling
  - If no session identity is maintained, memory may only help within a single run rather than across related requests
- **Sub-workflow reference:** None.

#### Governance Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured AI output for governance decisions.
- **Configuration choices:**
  - Manual JSON schema
  - Required fields:
    - `decision`
    - `risk_level`
    - `action_taken`
    - `reasoning`
  - Supported `decision` values:
    - `approved`
    - `revoked`
    - `escalated`
    - `pending`
  - Supported `risk_level` values:
    - `low`
    - `medium`
    - `high`
    - `critical`
  - Optional fields include:
    - `compliance_status`
    - `requires_human_review`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output parser connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - Parser failure if the LLM emits invalid JSON or unsupported enum values
  - Downstream switch depends specifically on `output.decision`; malformed parser output breaks routing
- **Sub-workflow reference:** None.

---

## 2.3 Block: IAM Validation Sub-Agent

### Overview
This block gives the governance agent a specialized validation tool. It performs event quality checks, risk classification, anomaly detection, and compliance flagging before the main agent finalizes a governance decision.

### Nodes Involved
- Access Signal Agent
- Access Signal Model
- Validation Output Parser

### Node Details

#### Access Signal Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool` — AI tool-agent callable by another agent.
- **Configuration choices:**
  - Input text comes from AI tool argument:
    - `={{ $fromAI('iam_event', 'The IAM event data to validate') }}`
  - System message defines validation responsibilities:
    - verify required fields
    - assess risk level
    - detect anomalies
    - check compliance requirements
  - Output parser enabled
  - Tool description explains its capability to validate IAM events
- **Key expressions or variables used:**
  - `$fromAI('iam_event', ...)`
- **Input and output connections:**
  - AI language model from `Access Signal Model`
  - AI output parser from `Validation Output Parser`
  - AI tool output to `Governance Agent`
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases or potential failure types:**
  - If the parent agent passes incomplete `iam_event` input, validation may be weak or fail
  - Parser mismatch if model returns nonconforming structure
  - Risk labeling may differ from the main agent’s judgment
- **Sub-workflow reference:** None.

#### Access Signal Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — OpenAI model powering the validation agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.1` for more deterministic validation behavior
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI language model output to `Access Signal Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - Same OpenAI credential/model-access risks as the main model
  - Latency adds to overall execution time
- **Sub-workflow reference:** None.

#### Validation Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — normalizes validation output.
- **Configuration choices:**
  - Manual schema
  - Required fields:
    - `is_valid`
    - `risk_level`
    - `validation_message`
  - Optional arrays:
    - `anomalies_detected`
    - `compliance_flags`
    - `missing_fields`
  - `risk_level` enum:
    - `low`
    - `medium`
    - `high`
    - `critical`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output parser connected to `Access Signal Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - AI returns strings instead of arrays
  - Enum mismatch on `risk_level`
  - Missing required fields cause parser failure
- **Sub-workflow reference:** None.

---

## 2.4 Block: External Analysis, Notifications, and Audit Tools

### Overview
These nodes are tools available to the governance agent. They extend the agent with external inference, communications, audit logging, and compliance lookups.

### Nodes Involved
- forgeLLM API Tool
- Slack Notification Tool
- Email Notification Tool
- Audit Log Tool
- Compliance Query Tool

### Node Details

#### forgeLLM API Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool` — AI-callable HTTP tool for specialized external model inference.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.runpod.ai/v2/l39v5drmx0q5sc/runsync`
  - Authentication: generic credential type using HTTP header auth
  - JSON body sends:
    - `prompt` from AI via `$fromAI('prompt', ...)`
    - `client` fixed to `iam_governance_system`
  - Header includes `Content-Type: application/json`
  - Tool description presents it as a domain-specific IAM analysis model
- **Key expressions or variables used:**
  - `$fromAI('prompt', 'The prompt to send to the fine-tuned LLM')`
- **Input and output connections:**
  - AI tool output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 4.4`; requires HTTP Request Tool support and AI tool integration in current n8n.
- **Edge cases or potential failure types:**
  - Invalid/missing header auth credential
  - Runpod endpoint unavailable or changed
  - Synchronous run timing out on large prompts
  - Response shape may not match what the agent expects
  - Credential is named `runway_video`, which may be unrelated semantically and could cause maintenance confusion
- **Sub-workflow reference:** None.

#### Slack Notification Tool
- **Type and technical role:** `n8n-nodes-base.slackTool` — AI-callable Slack sender.
- **Configuration choices:**
  - Sends to a channel
  - Channel ID is AI-supplied with default `#iam-approvals`
  - Message text comes from AI
  - OAuth2 authentication
  - Tool intended for escalations, approvals, and compliance notifications
- **Key expressions or variables used:**
  - `$fromAI('message', 'The notification message to send')`
  - `$fromAI('channel', 'Slack channel name or ID', 'string', '#iam-approvals')`
- **Input and output connections:**
  - AI tool output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 2.4`; requires valid Slack OAuth2 credentials with channel posting permissions.
- **Edge cases or potential failure types:**
  - OAuth token missing scopes
  - Invalid channel name/ID
  - Bot not invited to target channel
  - AI may generate an unsuitable or empty channel value
- **Sub-workflow reference:** None.

#### Email Notification Tool
- **Type and technical role:** `n8n-nodes-base.gmailTool` — AI-callable Gmail sender.
- **Configuration choices:**
  - Recipient, subject, and message body are all AI-supplied
  - Used for approval/revocation/compliance notifications
- **Key expressions or variables used:**
  - `$fromAI('recipient', 'Email recipient address', 'string')`
  - `$fromAI('subject', 'Email subject line', 'string')`
  - `$fromAI('body', 'Email message body')`
- **Input and output connections:**
  - AI tool output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 2.2`; requires Gmail OAuth2 credentials and a Google account authorized for sending mail.
- **Edge cases or potential failure types:**
  - Invalid email recipient
  - OAuth refresh or consent issues
  - Sending limits or anti-abuse restrictions
  - AI could omit subject/body if not instructed well enough
- **Sub-workflow reference:** None.

#### Audit Log Tool
- **Type and technical role:** `n8n-nodes-base.dataTableTool` — AI-callable insertion tool for audit storage.
- **Configuration choices:**
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__iam_audit_log__>`
  - Column mapping mode: auto-map input data
  - Intended to store events, validations, decisions, and actions
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - AI tool output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 1.1`; requires n8n Data Tables support and a real table ID.
- **Edge cases or potential failure types:**
  - Placeholder not replaced with an actual data table ID
  - AI-generated field structure not matching the table schema
  - Write permission or environment limitations on Data Tables
- **Sub-workflow reference:** None.

#### Compliance Query Tool
- **Type and technical role:** `n8n-nodes-base.dataTableTool` — AI-callable read/query tool on audit/compliance data.
- **Configuration choices:**
  - Operation: `get`
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__iam_audit_log__>`
  - Tool intended for querying historical audit data
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - AI tool output connected to `Governance Agent`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Placeholder table ID not replaced
  - Tool may query a log table rather than a dedicated compliance rules table; functionally acceptable, but naming may be misleading
  - Query semantics depend on Data Table Tool defaults and agent-generated parameters
- **Sub-workflow reference:** None.

---

## 2.5 Block: Decision Routing

### Overview
This block converts the structured AI decision into deterministic branch logic. It is the boundary between probabilistic AI output and explicit workflow execution.

### Nodes Involved
- Route by Decision

### Node Details

#### Route by Decision
- **Type and technical role:** `n8n-nodes-base.switch` — routes items to decision-specific branches.
- **Configuration choices:**
  - Evaluates `={{ $json.output.decision }}`
  - Four rules:
    - equals `approved`
    - equals `revoked`
    - equals `escalated`
    - equals `pending`
- **Key expressions or variables used:**
  - `{{$json.output.decision}}`
- **Input and output connections:**
  - Main input from `Governance Agent`
  - Output 0 to `Prepare Approved Data`
  - Output 1 to `Store Revoked Events` and `Prepare Revoked Data`
  - Output 2 to `Store Escalated Events` and `Prepare Escalated Data`
  - No connected path for `pending`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - If parser output is absent or `output.decision` missing, no branch matches
  - `pending` has no downstream connection, so such items may silently stop
  - Revoked and escalated outputs are each connected both directly to storage and via preparation nodes, which can cause duplicate/incomplete writes
- **Sub-workflow reference:** None.

**Important design issue:**  
For the `revoked` and `escalated` branches, the switch is wired directly to storage nodes *and* to their corresponding preparation nodes. That means:
- `Store Revoked Events` may receive one item directly from the switch and another from `Prepare Revoked Data`
- `Store Escalated Events` may receive one item directly from the switch and another from `Prepare Escalated Data`

This is likely unintended and can result in:
- duplicate inserts
- malformed inserts from raw agent output instead of prepared records
- inconsistent response payloads

The approved branch does not have this duplication issue.

---

## 2.6 Block: Approved Path

### Overview
This branch handles events the governance agent approves. It shapes the payload into a storage-ready schema, writes it to the approved-events table, and then returns the workflow response.

### Nodes Involved
- Prepare Approved Data
- Store Approved Events

### Node Details

#### Prepare Approved Data
- **Type and technical role:** `n8n-nodes-base.set` — creates a normalized approved-event record.
- **Configuration choices:**
  - Assigns:
    - `event_id` from `body.event_id`
    - `user_id` from `body.user_id`
    - `resource` from `body.resource`
    - `action` from `body.action`
    - `decision` from `output.decision`
    - `risk_level` from `output.risk_level`
    - `action_taken` from `output.action_taken`
    - `reasoning` from `output.reasoning`
    - `timestamp` from `$now`
- **Key expressions or variables used:**
  - `{{$json.body.event_id}}`
  - `{{$json.body.user_id}}`
  - `{{$json.body.resource}}`
  - `{{$json.body.action}}`
  - `{{$json.output.decision}}`
  - `{{$json.output.risk_level}}`
  - `{{$json.output.action_taken}}`
  - `{{$json.output.reasoning}}`
  - `{{$now}}`
- **Input and output connections:**
  - Input from `Route by Decision`
  - Output to `Store Approved Events`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - Missing fields in `body` produce null/empty values
  - If the webhook body is nested differently, expressions break
- **Sub-workflow reference:** None.

#### Store Approved Events
- **Type and technical role:** `n8n-nodes-base.dataTable` — persists approved records.
- **Configuration choices:**
  - Auto-map input data to the target table
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__approved_events__>`
- **Key expressions or variables used:** None directly
- **Input and output connections:**
  - Input from `Prepare Approved Data`
  - Output to `Return Response`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Auto-mapping assumes table columns align with prepared fields
- **Sub-workflow reference:** None.

---

## 2.7 Block: Revoked Path

### Overview
This branch processes revoked events. It is intended to prepare and store normalized revocation records, but the current wiring creates a duplication risk.

### Nodes Involved
- Prepare Revoked Data
- Store Revoked Events

### Node Details

#### Prepare Revoked Data
- **Type and technical role:** `n8n-nodes-base.set` — shapes revoked-event output.
- **Configuration choices:**
  - Same field mapping pattern as the approved branch:
    - `event_id`
    - `user_id`
    - `resource`
    - `action`
    - `decision`
    - `risk_level`
    - `action_taken`
    - `reasoning`
    - `timestamp`
- **Key expressions or variables used:** Same body/output expressions as approved branch.
- **Input and output connections:**
  - Input from `Route by Decision`
  - Output to `Store Revoked Events`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - Same field-shape risks as approved branch
- **Sub-workflow reference:** None.

#### Store Revoked Events
- **Type and technical role:** `n8n-nodes-base.dataTable` — persists revoked-event records.
- **Configuration choices:**
  - Auto-map input data
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__revoked_events__>`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs from:
    - `Route by Decision` directly
    - `Prepare Revoked Data`
  - Output to `Return Response`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Duplicate insert risk because of dual inputs
  - Direct switch input may not match the table schema
  - Placeholder table ID must be replaced
- **Sub-workflow reference:** None.

---

## 2.8 Block: Escalated Path

### Overview
This branch handles high-risk or human-review events. It preserves the same base event fields plus a human-review flag, stores them, and returns a response, but it has the same duplicate-input issue as the revoked branch.

### Nodes Involved
- Prepare Escalated Data
- Store Escalated Events

### Node Details

#### Prepare Escalated Data
- **Type and technical role:** `n8n-nodes-base.set` — shapes escalated-event output.
- **Configuration choices:**
  - Assigns:
    - `event_id`
    - `user_id`
    - `resource`
    - `action`
    - `decision`
    - `risk_level`
    - `action_taken`
    - `reasoning`
    - `requires_human_review`
    - `timestamp`
- **Key expressions or variables used:**
  - Same body/output expressions as prior Set nodes
  - Includes `{{$json.output.requires_human_review}}`
- **Input and output connections:**
  - Input from `Route by Decision`
  - Output to `Store Escalated Events`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - `requires_human_review` may be missing because it is optional in the parser schema
  - Body field assumptions remain the same
- **Sub-workflow reference:** None.

#### Store Escalated Events
- **Type and technical role:** `n8n-nodes-base.dataTable` — persists escalated-event records.
- **Configuration choices:**
  - Auto-map input data
  - Data table ID placeholder: `<__PLACEHOLDER_VALUE__escalated_events__>`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs from:
    - `Route by Decision` directly
    - `Prepare Escalated Data`
  - Output to `Return Response`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Duplicate or malformed inserts due to dual inputs
  - Placeholder table ID must be replaced
- **Sub-workflow reference:** None.

---

## 2.9 Block: Final Webhook Response

### Overview
This node returns the workflow’s final JSON payload to the original caller. It closes the request-response cycle started by the webhook.

### Nodes Involved
- Return Response

### Node Details

#### Return Response
- **Type and technical role:** `n8n-nodes-base.respondToWebhook` — explicit HTTP response sender.
- **Configuration choices:**
  - Response format: JSON
  - Response body: `={{ $json }}`
- **Key expressions or variables used:**
  - `{{$json}}`
- **Input and output connections:**
  - Input from:
    - `Store Approved Events`
    - `Store Revoked Events`
    - `Store Escalated Events`
- **Version-specific requirements:** `typeVersion 1.5`
- **Edge cases or potential failure types:**
  - If multiple items arrive because of duplicate branch wiring, response behavior may be inconsistent
  - Returning the inserted table row rather than the original governance decision may or may not match caller expectations
- **Sub-workflow reference:** None.

---

## 2.10 Documentation and Sticky Notes

### Overview
These nodes are non-executable documentation embedded in the canvas. They provide business context, setup expectations, prerequisites, and block-level explanations.

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
- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation.
- **Configuration choices:** Describes the full workflow purpose and end-to-end behavior.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Lists setup steps for webhook, LLM, forgeLLM, compliance, Gmail, and Slack.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Lists prerequisites, use cases, customization, and benefits.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the compliance and audit area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the routing area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the receive/evaluate area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels the storage/response area.
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive IAM Event | n8n-nodes-base.webhook | Receives IAM event HTTP POST requests |  | Governance Agent | ## Receive & Evaluate IAM Event<br>**What:** Webhook triggers on incoming IAM event payload; LLM agent immediately assesses it using memory.<br>**Why:** Enables real-time, event-driven governance while applying consistent, context-aware policy judgement at scale without polling. |
| Governance Agent | @n8n/n8n-nodes-langchain.agent | Main AI orchestrator for governance decisions | Receive IAM Event; Governance Model; Conversation Memory; Access Signal Agent; forgeLLM API Tool; Slack Notification Tool; Email Notification Tool; Audit Log Tool; Compliance Query Tool; Governance Output Parser | Route by Decision | ## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment.<br><br>## Receive & Evaluate IAM Event<br>**What:** Webhook triggers on incoming IAM event payload; LLM agent immediately assesses it using memory.<br>**Why:** Enables real-time, event-driven governance while applying consistent, context-aware policy judgement at scale without polling. |
| Governance Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI chat model for the governance agent |  | Governance Agent | ## Setup Steps<br>1. Configure the Webhook node with your IAM event source endpoint.<br>2. Add LLM credentials to the forgeLLM API Tool node.<br>3. Set up Governance Model with your policy prompt and connect Conversation Memory.<br>4. Configure Access Signal Agent with your access data source credentials.<br>5. Connect Compliance Query Tool to your compliance database or API.<br>6. Add Gmail/SMTP credentials to the Email Notification Tool.<br>7. Add Slack Bot token to the Slack Notification Tool. |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Provides memory context to the governance agent |  | Governance Agent | ## Receive & Evaluate IAM Event<br>**What:** Webhook triggers on incoming IAM event payload; LLM agent immediately assesses it using memory.<br>**Why:** Enables real-time, event-driven governance while applying consistent, context-aware policy judgement at scale without polling. |
| Access Signal Agent | @n8n/n8n-nodes-langchain.agentTool | Specialized AI validation tool for IAM events | Access Signal Model; Validation Output Parser | Governance Agent | ## Setup Steps<br>1. Configure the Webhook node with your IAM event source endpoint.<br>2. Add LLM credentials to the forgeLLM API Tool node.<br>3. Set up Governance Model with your policy prompt and connect Conversation Memory.<br>4. Configure Access Signal Agent with your access data source credentials.<br>5. Connect Compliance Query Tool to your compliance database or API.<br>6. Add Gmail/SMTP credentials to the Email Notification Tool.<br>7. Add Slack Bot token to the Slack Notification Tool.<br><br>## Receive & Evaluate IAM Event<br>**What:** Webhook triggers on incoming IAM event payload; LLM agent immediately assesses it using memory.<br>**Why:** Enables real-time, event-driven governance while applying consistent, context-aware policy judgement at scale without polling. |
| Access Signal Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI chat model for validation agent |  | Access Signal Agent | ## Receive & Evaluate IAM Event<br>**What:** Webhook triggers on incoming IAM event payload; LLM agent immediately assesses it using memory.<br>**Why:** Enables real-time, event-driven governance while applying consistent, context-aware policy judgement at scale without polling. |
| Governance Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces schema for governance decisions |  | Governance Agent | ## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment. |
| Validation Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces schema for IAM validation results |  | Access Signal Agent | ## Receive & Evaluate IAM Event<br>**What:** Webhook triggers on incoming IAM event payload; LLM agent immediately assesses it using memory.<br>**Why:** Enables real-time, event-driven governance while applying consistent, context-aware policy judgement at scale without polling. |
| forgeLLM API Tool | n8n-nodes-base.httpRequestTool | Calls external fine-tuned model API |  | Governance Agent | ## Setup Steps<br>1. Configure the Webhook node with your IAM event source endpoint.<br>2. Add LLM credentials to the forgeLLM API Tool node.<br>3. Set up Governance Model with your policy prompt and connect Conversation Memory.<br>4. Configure Access Signal Agent with your access data source credentials.<br>5. Connect Compliance Query Tool to your compliance database or API.<br>6. Add Gmail/SMTP credentials to the Email Notification Tool.<br>7. Add Slack Bot token to the Slack Notification Tool.<br><br>## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment. |
| Slack Notification Tool | n8n-nodes-base.slackTool | Sends Slack messages from agent decisions |  | Governance Agent | ## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment. |
| Email Notification Tool | n8n-nodes-base.gmailTool | Sends email notifications from agent decisions |  | Governance Agent | ## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment. |
| Audit Log Tool | n8n-nodes-base.dataTableTool | Writes audit data via agent tool access |  | Governance Agent | ## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment. |
| Compliance Query Tool | n8n-nodes-base.dataTableTool | Reads historical audit/compliance data |  | Governance Agent | ## Compliance & Audit<br>**What:** Queries compliance rules; logs event details to audit store.<br>**Why:** Ensures regulatory traceability and policy alignment. |
| Route by Decision | n8n-nodes-base.switch | Routes decisions into approved/revoked/escalated/pending branches | Governance Agent | Prepare Approved Data; Store Revoked Events; Prepare Revoked Data; Store Escalated Events; Prepare Escalated Data | ## Route by Decision<br>**What:** Rules engine splits events into Approved, Revoked, or Escalated paths.<br>**Why:** Automates downstream handling based on AI verdict. |
| Prepare Approved Data | n8n-nodes-base.set | Normalizes approved-event data for storage | Route by Decision | Store Approved Events | ## Route by Decision<br>**What:** Rules engine splits events into Approved, Revoked, or Escalated paths.<br>**Why:** Automates downstream handling based on AI verdict. |
| Store Approved Events | n8n-nodes-base.dataTable | Stores approved-event records | Prepare Approved Data | Return Response | ## Store & Respond<br>**What:** Prepares and stores classified event data; returns response.<br>**Why:** Maintains a structured audit trail and closes the governance loop. |
| Prepare Revoked Data | n8n-nodes-base.set | Normalizes revoked-event data for storage | Route by Decision | Store Revoked Events | ## Route by Decision<br>**What:** Rules engine splits events into Approved, Revoked, or Escalated paths.<br>**Why:** Automates downstream handling based on AI verdict. |
| Store Revoked Events | n8n-nodes-base.dataTable | Stores revoked-event records | Route by Decision; Prepare Revoked Data | Return Response | ## Store & Respond<br>**What:** Prepares and stores classified event data; returns response.<br>**Why:** Maintains a structured audit trail and closes the governance loop. |
| Prepare Escalated Data | n8n-nodes-base.set | Normalizes escalated-event data for storage | Route by Decision | Store Escalated Events | ## Route by Decision<br>**What:** Rules engine splits events into Approved, Revoked, or Escalated paths.<br>**Why:** Automates downstream handling based on AI verdict. |
| Store Escalated Events | n8n-nodes-base.dataTable | Stores escalated-event records | Route by Decision; Prepare Escalated Data | Return Response | ## Store & Respond<br>**What:** Prepares and stores classified event data; returns response.<br>**Why:** Maintains a structured audit trail and closes the governance loop. |
| Return Response | n8n-nodes-base.respondToWebhook | Returns final JSON response to the webhook caller | Store Approved Events; Store Revoked Events; Store Escalated Events |  | ## Store & Respond<br>**What:** Prepares and stores classified event data; returns response.<br>**Why:** Maintains a structured audit trail and closes the governance loop. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas setup guidance |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas prerequisites and benefits note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas section label for compliance/audit |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas section label for routing |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas section label for intake/evaluation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas section label for storage/response |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `AI agent for IAM event governance with compliance routing and audit`.
   - Ensure your n8n instance supports:
     - LangChain AI nodes
     - Data Tables
     - Slack Tool
     - Gmail Tool
     - Respond to Webhook

2. **Add the webhook trigger**
   - Create a `Webhook` node named `Receive IAM Event`.
   - Set:
     - **HTTP Method:** `POST`
     - **Path:** `iam-events`
     - **Response Mode:** `Using Respond to Webhook Node` / `responseNode`
   - Leave options at defaults unless you need authentication or custom response codes.

3. **Add the main AI agent**
   - Create an `AI Agent` node named `Governance Agent`.
   - Set:
     - **Prompt Type:** Define manually
     - **Text/Input:** `{{ $json.body }}`
     - Enable structured output parsing.
   - Set the system prompt to instruct the agent to:
     - orchestrate IAM approvals and revocations
     - validate events via a secondary access signal tool
     - base decisions on risk and compliance
     - escalate high-risk events for human review
     - generate compliance-oriented results

4. **Connect webhook to the governance agent**
   - Main connection:
     - `Receive IAM Event` → `Governance Agent`

5. **Add the primary OpenAI model**
   - Create an `OpenAI Chat Model` node named `Governance Model`.
   - Choose model `gpt-4o-mini`.
   - Set temperature to `0.2`.
   - Attach valid OpenAI credentials.
   - Connect:
     - `Governance Model` → `Governance Agent` using the AI language model connection.

6. **Add memory**
   - Create a `Memory Buffer Window` node named `Conversation Memory`.
   - Leave defaults unless you want custom memory size/session behavior.
   - Connect:
     - `Conversation Memory` → `Governance Agent` using the AI memory connection.

7. **Add the governance output parser**
   - Create a `Structured Output Parser` node named `Governance Output Parser`.
   - Use manual schema with these fields:
     - `decision`: string enum `approved|revoked|escalated|pending`
     - `risk_level`: string enum `low|medium|high|critical`
     - `action_taken`: string
     - `compliance_status`: string
     - `reasoning`: string
     - `requires_human_review`: boolean
   - Mark as required:
     - `decision`
     - `risk_level`
     - `action_taken`
     - `reasoning`
   - Connect:
     - `Governance Output Parser` → `Governance Agent` using the AI output parser connection.

8. **Add the validation sub-agent**
   - Create an `AI Agent Tool` node named `Access Signal Agent`.
   - Set input text to:
     - `{{ $fromAI('iam_event', 'The IAM event data to validate') }}`
   - Add a system message instructing it to:
     - validate IAM event structure
     - verify `user_id`, `resource`, `action`, `timestamp`
     - classify risk
     - detect anomalies
     - flag compliance issues
   - Enable output parser.
   - Add a meaningful tool description stating that it validates structured IAM events.

9. **Add the validation model**
   - Create another `OpenAI Chat Model` node named `Access Signal Model`.
   - Set:
     - model `gpt-4o-mini`
     - temperature `0.1`
   - Use the same or another OpenAI credential.
   - Connect:
     - `Access Signal Model` → `Access Signal Agent` using AI language model.

10. **Add the validation output parser**
    - Create a `Structured Output Parser` node named `Validation Output Parser`.
    - Configure manual schema:
      - `is_valid`: boolean
      - `risk_level`: enum `low|medium|high|critical`
      - `anomalies_detected`: array of strings
      - `compliance_flags`: array of strings
      - `missing_fields`: array of strings
      - `validation_message`: string
    - Required:
      - `is_valid`
      - `risk_level`
      - `validation_message`
    - Connect:
      - `Validation Output Parser` → `Access Signal Agent` using AI output parser.

11. **Expose the validation agent as a tool to the governance agent**
    - Connect:
      - `Access Signal Agent` → `Governance Agent` using AI tool connection.

12. **Add the external forgeLLM tool**
    - Create an `HTTP Request Tool` node named `forgeLLM API Tool`.
    - Configure:
      - **Method:** `POST`
      - **URL:** `https://api.runpod.ai/v2/l39v5drmx0q5sc/runsync`
      - **Authentication:** Generic credential type → HTTP Header Auth
      - Add header:
        - `Content-Type: application/json`
      - **Body type:** JSON
      - **Send body:** enabled
      - **JSON body:**
        - `input.prompt` = `{{ $fromAI('prompt', 'The prompt to send to the fine-tuned LLM') }}`
        - `input.client` = `iam_governance_system`
    - Add a tool description indicating specialized IAM inference.
    - Create HTTP Header Auth credentials with the required API header for the Runpod/forgeLLM endpoint.
    - Connect:
      - `forgeLLM API Tool` → `Governance Agent` using AI tool connection.

13. **Add the Slack notification tool**
    - Create a `Slack Tool` node named `Slack Notification Tool`.
    - Configure:
      - send destination type: channel
      - text: `{{ $fromAI('message', 'The notification message to send') }}`
      - channel: `{{ $fromAI('channel', 'Slack channel name or ID', 'string', '#iam-approvals') }}`
    - Add a tool description for approval/escalation notifications.
    - Connect Slack OAuth2 credentials.
    - Connect:
      - `Slack Notification Tool` → `Governance Agent` using AI tool connection.

14. **Add the email notification tool**
    - Create a `Gmail Tool` node named `Email Notification Tool`.
    - Configure:
      - recipient: `{{ $fromAI('recipient', 'Email recipient address', 'string') }}`
      - subject: `{{ $fromAI('subject', 'Email subject line', 'string') }}`
      - message body: `{{ $fromAI('body', 'Email message body') }}`
    - Add a tool description for stakeholder notifications.
    - Connect Gmail OAuth2 credentials.
    - Connect:
      - `Email Notification Tool` → `Governance Agent` using AI tool connection.

15. **Add the audit-write tool**
    - Create a `Data Table Tool` node named `Audit Log Tool`.
    - Configure it for inserting rows into an audit table.
    - Select your Data Table for IAM audit records.
    - Use auto-mapping if your table schema is flexible, or define exact columns if stricter control is required.
    - Suggested columns:
      - `event_id`
      - `user_id`
      - `resource`
      - `action`
      - `decision`
      - `risk_level`
      - `reasoning`
      - `timestamp`
      - `compliance_status`
      - `requires_human_review`
    - Add a tool description explaining it stores audit events.
    - Connect:
      - `Audit Log Tool` → `Governance Agent` using AI tool connection.

16. **Add the compliance-read tool**
    - Create a `Data Table Tool` node named `Compliance Query Tool`.
    - Set operation to `Get`.
    - Point it to the relevant data table.
    - In the provided workflow it uses the same audit table ID placeholder, but a better implementation may use a dedicated compliance or policy table.
    - Add a tool description indicating historical query/compliance reporting use.
    - Connect:
      - `Compliance Query Tool` → `Governance Agent` using AI tool connection.

17. **Add the decision router**
    - Create a `Switch` node named `Route by Decision`.
    - Create rules on `{{ $json.output.decision }}`:
      1. equals `approved`
      2. equals `revoked`
      3. equals `escalated`
      4. equals `pending`
    - Connect:
      - `Governance Agent` → `Route by Decision`

18. **Add the approved preparation node**
    - Create a `Set` node named `Prepare Approved Data`.
    - Add fields:
      - `event_id` = `{{ $json.body.event_id }}`
      - `user_id` = `{{ $json.body.user_id }}`
      - `resource` = `{{ $json.body.resource }}`
      - `action` = `{{ $json.body.action }}`
      - `decision` = `{{ $json.output.decision }}`
      - `risk_level` = `{{ $json.output.risk_level }}`
      - `action_taken` = `{{ $json.output.action_taken }}`
      - `reasoning` = `{{ $json.output.reasoning }}`
      - `timestamp` = `{{ $now }}`
    - Connect the `approved` output of `Route by Decision` to this node.

19. **Add approved storage**
    - Create a `Data Table` node named `Store Approved Events`.
    - Configure insert/write behavior to your approved-events table.
    - Select the target approved data table.
    - Prefer columns matching the Set node fields.
    - Connect:
      - `Prepare Approved Data` → `Store Approved Events`

20. **Add the revoked preparation node**
    - Create a `Set` node named `Prepare Revoked Data`.
    - Use the same field structure as approved:
      - `event_id`, `user_id`, `resource`, `action`, `decision`, `risk_level`, `action_taken`, `reasoning`, `timestamp`
    - Connect the `revoked` output of `Route by Decision` to this node.

21. **Add revoked storage**
    - Create a `Data Table` node named `Store Revoked Events`.
    - Configure insert/write behavior to a revoked-events table.
    - Connect:
      - `Prepare Revoked Data` → `Store Revoked Events`

22. **Add the escalated preparation node**
    - Create a `Set` node named `Prepare Escalated Data`.
    - Add fields:
      - `event_id`
      - `user_id`
      - `resource`
      - `action`
      - `decision`
      - `risk_level`
      - `action_taken`
      - `reasoning`
      - `requires_human_review` = `{{ $json.output.requires_human_review }}`
      - `timestamp` = `{{ $now }}`
    - Connect the `escalated` output of `Route by Decision` to this node.

23. **Add escalated storage**
    - Create a `Data Table` node named `Store Escalated Events`.
    - Configure insert/write behavior to an escalated-events table.
    - Connect:
      - `Prepare Escalated Data` → `Store Escalated Events`

24. **Do not reproduce the routing mistake unless you intentionally want it**
    - The provided JSON directly connects:
      - `Route by Decision` → `Store Revoked Events`
      - `Route by Decision` → `Store Escalated Events`
    - This is likely a design flaw.
    - Recommended rebuild:
      - only connect each storage node from its corresponding preparation node
      - do **not** connect the switch directly to revoked/escalated storage

25. **Add the webhook response node**
    - Create a `Respond to Webhook` node named `Return Response`.
    - Set:
      - respond with `JSON`
      - response body: `{{ $json }}`
    - Connect:
      - `Store Approved Events` → `Return Response`
      - `Store Revoked Events` → `Return Response`
      - `Store Escalated Events` → `Return Response`

26. **Handle pending decisions explicitly**
    - The source workflow defines a `pending` rule but leaves it unconnected.
    - Recommended options:
      - connect `pending` to its own Set + Data Table branch
      - or connect it directly to `Return Response` with a fallback payload
      - or change the parser schema to remove `pending` if unsupported operationally

27. **Create the required credentials**
    - **OpenAI credential**
      - Used by `Governance Model` and `Access Signal Model`
    - **HTTP Header Auth credential**
      - Used by `forgeLLM API Tool`
      - Configure the exact authorization header required by your Runpod/forgeLLM deployment
    - **Slack OAuth2 credential**
      - Used by `Slack Notification Tool`
      - Ensure channel posting scopes are included
    - **Gmail OAuth2 credential**
      - Used by `Email Notification Tool`
      - Ensure send-mail permissions are authorized

28. **Create the required Data Tables**
    - At minimum:
      - IAM audit log table
      - approved events table
      - revoked events table
      - escalated events table
    - Replace all placeholder IDs with real table IDs.
    - Keep column names aligned with the fields created in Set nodes and AI tool writes.

29. **Test with a sample IAM event**
    - Example expected payload shape:
      - `event_id`
      - `user_id`
      - `resource`
      - `action`
      - `timestamp`
    - Include other useful context such as:
      - `requested_role`
      - `source_ip`
      - `geo`
      - `request_reason`
      - `manager_approval`
    - Confirm the governance agent returns structured output under `output`.

30. **Validate branch behavior**
    - Test at least:
      - low-risk approval event
      - suspicious revocation case
      - high-risk escalation case
      - malformed event with missing fields
    - Confirm:
      - parser outputs are valid
      - switch routes correctly
      - data writes succeed
      - response payload is returned before webhook timeout

31. **Optional hardening improvements**
    - Add error handling branches for:
      - OpenAI failures
      - HTTP tool failures
      - Slack/Gmail errors
      - parser failures
    - Add validation before AI invocation using an `IF` or `Code` node
    - Normalize webhook body shape before feeding it to AI
    - Add a dedicated `pending` branch
    - Ensure audit logging happens deterministically outside the agent if compliance traceability is critical

### Sub-workflow setup
There are **no sub-workflow nodes** and no workflow-execution child workflows in this design.  
The AI tool-agent `Access Signal Agent` is an internal callable agent node, not a separate n8n workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How It Works: This workflow automates Identity and Access Management (IAM) event governance using an AI agent, targeting security operations teams, compliance officers, and IT governance teams managing cloud or enterprise IAM systems. The core problem it solves is the manual, error-prone review of IAM events, such as permission grants, role changes, and access revocations, which are high-risk and require rapid, consistent decision-making at scale. When an IAM event is received via webhook (POST), a Governance Agent powered by an LLM evaluates it using contextual memory, an Access Signal Agent, and a forgeLLM API. It cross-references compliance rules via a Compliance Query Tool and logs findings through an Audit Log Tool. Notifications are dispatched via Email and Slack. Based on the agent's decision, a Rules-based Router directs the event into one of three branches, namely: Approved, Revoked, or Escalated, where event data is prepared and stored accordingly. A unified response is then returned to the caller, ensuring every IAM event is audited, classified, and actioned without human bottlenecks. | Workflow canvas note |
| Setup Steps: 1. Configure the Webhook node with your IAM event source endpoint. 2. Add LLM credentials to the forgeLLM API Tool node. 3. Set up Governance Model with your policy prompt and connect Conversation Memory. 4. Configure Access Signal Agent with your access data source credentials. 5. Connect Compliance Query Tool to your compliance database or API. 6. Add Gmail/SMTP credentials to the Email Notification Tool. 7. Add Slack Bot token to the Slack Notification Tool. | Workflow canvas note |
| Prerequisites: forgeLLM or compatible LLM API key; Slack Bot token; Gmail/SMTP credentials. | Workflow canvas note |
| Use Cases: Automatically approve or revoke IAM role assignments based on policy. | Workflow canvas note |
| Customization: Swap forgeLLM for OpenAI or Anthropic models. | Workflow canvas note |
| Benefits: Eliminates manual IAM review bottlenecks. | Workflow canvas note |

## Additional implementation observations
- The workflow title provided by the user differs slightly from the JSON workflow name; the JSON name is: **AI agent for IAM event governance with compliance routing and audit**.
- The `pending` decision exists in the parser and switch but has no implementation branch.
- The `revoked` and `escalated` branches have direct switch-to-storage connections in addition to preparation-to-storage connections; this should usually be corrected.
- Several Data Table IDs are placeholders and must be replaced before the workflow can function.