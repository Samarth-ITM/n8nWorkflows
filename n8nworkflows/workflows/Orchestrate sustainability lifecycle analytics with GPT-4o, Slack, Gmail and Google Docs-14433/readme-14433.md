Orchestrate sustainability lifecycle analytics with GPT-4o, Slack, Gmail and Google Docs

https://n8nworkflows.xyz/workflows/orchestrate-sustainability-lifecycle-analytics-with-gpt-4o--slack--gmail-and-google-docs-14433


# Orchestrate sustainability lifecycle analytics with GPT-4o, Slack, Gmail and Google Docs

# 1. Workflow Overview

This workflow orchestrates sustainability lifecycle operations across three major domains:

- **Circular economy analysis**
- **Sustainability governance and approvals**
- **ESG documentation and stakeholder reporting**

It accepts data from three different entry points, uses a central AI orchestrator powered by GPT-4o to classify and delegate work, tracks outcomes in n8n Data Tables, then aggregates and distributes summary outputs via Slack and Gmail.

## 1.1 Input Reception and Normalization

The workflow starts from three intake mechanisms:

- a scheduled trigger for recurring lifecycle monitoring
- a webhook for system-to-system sustainability data submission
- a form trigger for manual sustainability initiative submissions

These inputs are normalized through a merge node so the downstream AI orchestration can process a unified payload shape.

## 1.2 AI Orchestration and Delegation

A central **Sustainability Orchestrator** agent receives the merged payload, uses GPT-4o plus memory, and delegates tasks to one of three specialist AI tools:

- **Circular Economy Agent**
- **Sustainability Governance Agent**
- **Documentation Agent**

Its structured output determines the next route.

## 1.3 Specialist AI Processing

The specialist agents are exposed to the orchestrator as AI tools:

- Circular Economy Agent performs lifecycle and waste-reduction analysis
- Governance Agent evaluates compliance and approval needs, with Slack human-in-the-loop approval support
- Documentation Agent prepares ESG-oriented documentation and can create Google Docs

## 1.4 Action Routing and Persistence

The orchestrator’s structured result is routed by action:

- `analyze` → lifecycle analytics tracking
- `approve` → governance approval tracking
- `document` → ESG documentation tracking

Each route maps the orchestrator result into a Data Table-friendly record.

## 1.5 Aggregation and Notifications

After the three tracking branches, results are merged, aggregated, and summarized. Final outputs include:

- a Slack summary notification
- a Gmail stakeholder report

---

# 2. Block-by-Block Analysis

## 2.1 Data Ingestion & Normalisation

**Overview:**  
This block captures sustainability-related inputs from scheduled, API, webhook, and form-based sources. It consolidates them into a common stream for the orchestrator.

**Nodes Involved:**  
- Monitor Lifecycle Data
- Fetch External Sustainability Data
- Receive Sustainability Data
- Initiative Submission Form
- Combine Input Sources

### Node Details

#### Monitor Lifecycle Data
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Recurring trigger that starts periodic monitoring runs.
- **Configuration choices:**  
  Configured with an interval rule triggering at hour `6`, effectively running once daily at 06:00 based on instance timezone.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input; outputs to:
  - Combine Input Sources
  - Fetch External Sustainability Data
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  - Timezone assumptions may produce unexpected execution times
  - If relying on external services downstream, scheduled runs may produce empty or duplicate records
- **Sub-workflow reference:**  
  None.

#### Fetch External Sustainability Data
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Pulls sustainability data from an external API endpoint.
- **Configuration choices:**  
  Sends a query parameter named `date` using the current date in `yyyy-MM-dd` format.  
  Error handling is set to **continue to error output**, though no second branch is actually connected.
- **Key expressions or variables used:**  
  `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Input and output connections:**  
  Input from Monitor Lifecycle Data; main success output to Combine Input Sources.
- **Version-specific requirements:**  
  Uses `typeVersion 4.4`.
- **Edge cases / failures:**  
  - Placeholder endpoint must be replaced
  - API auth is not configured; many real APIs will require headers or credentials
  - Because error output is unconnected, failed requests will not be handled explicitly
  - Malformed API responses may break assumptions downstream
- **Sub-workflow reference:**  
  None.

#### Receive Sustainability Data
- **Type and role:** `n8n-nodes-base.webhook`  
  Receives POSTed sustainability payloads from external systems.
- **Configuration choices:**  
  - Path: `sustainability-data`
  - Method: `POST`
  - Response mode: `lastNode`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  No input; outputs to Combine Input Sources.
- **Version-specific requirements:**  
  Uses `typeVersion 2.1`.
- **Edge cases / failures:**  
  - Since response mode is `lastNode`, callers wait for the full workflow path to complete
  - Long AI execution could cause webhook timeout depending on deployment and caller expectations
  - Missing `body` structure may alter what the orchestrator sees
- **Sub-workflow reference:**  
  None.

#### Initiative Submission Form
- **Type and role:** `n8n-nodes-base.formTrigger`  
  Accepts manual user submissions for sustainability initiatives.
- **Configuration choices:**  
  Form includes:
  - Initiative Type
  - Description
  - Product/Material
  - Estimated Impact (kg waste reduced)  
  Attribution footer disabled.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input; outputs to Combine Input Sources.
- **Version-specific requirements:**  
  Uses `typeVersion 2.5`.
- **Edge cases / failures:**  
  - User-entered data may be incomplete or inconsistent
  - Number field may still be empty or semantically invalid
  - No validation/enrichment layer exists before AI analysis
- **Sub-workflow reference:**  
  None.

#### Combine Input Sources
- **Type and role:** `n8n-nodes-base.merge`  
  Consolidates the three primary intake streams into one flow.
- **Configuration choices:**  
  Configured for `3` inputs.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Inputs from:
  - Monitor Lifecycle Data
  - Fetch External Sustainability Data
  - Receive Sustainability Data
  - Initiative Submission Form  
  Practically, the node is configured for 3 inputs, while there are 4 incoming connections in the JSON. This is a structural inconsistency to review during rebuild.
- **Version-specific requirements:**  
  Uses `typeVersion 3.2`.
- **Edge cases / failures:**  
  - Potential misconfiguration: 4 incoming paths with `numberInputs: 3`
  - Merge behavior can produce unexpected execution timing or missing items if inputs do not arrive in the intended combination
- **Sub-workflow reference:**  
  None.

---

## 2.2 Sustainability Orchestrator & Shared Context

**Overview:**  
This block performs central AI-based decision-making. It converts incoming payloads into a structured classification used for routing and exposes specialist agents as callable tools.

**Nodes Involved:**  
- Sustainability Orchestrator
- Orchestrator Model
- Shared Memory
- Orchestrator Output Parser

### Node Details

#### Sustainability Orchestrator
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`  
  Central AI coordinator that classifies the incoming sustainability item and decides which specialist domain should handle it.
- **Configuration choices:**  
  - Input text is taken from `$json.body` if present; otherwise the whole `$json` is stringified
  - System message instructs the agent to return:
    - `action`
    - `priority`
    - `delegateTo`
    - `summary`
  - Structured output parser enabled
- **Key expressions or variables used:**  
  `{{ $json.body ? JSON.stringify($json.body) : JSON.stringify($json) }}`
- **Input and output connections:**  
  Main input from Combine Input Sources  
  Main output to Route by Action  
  AI connections:
  - language model from Orchestrator Model
  - memory from Shared Memory
  - output parser from Orchestrator Output Parser
  - tools from Circular Economy Agent, Sustainability Governance Agent, Documentation Agent
- **Version-specific requirements:**  
  Uses `typeVersion 3.1`.
- **Edge cases / failures:**  
  - If input payloads vary significantly, the model may classify inconsistently
  - If the LLM returns invalid JSON/schema, parser failure may occur
  - Tool-calling behavior depends on n8n LangChain node compatibility and OpenAI model support
- **Sub-workflow reference:**  
  None.

#### Orchestrator Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model backing the orchestrator.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI language model connection to Sustainability Orchestrator.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`; requires OpenAI credentials.
- **Edge cases / failures:**  
  - Invalid API key
  - model access not available on the account
  - API quota/rate limiting
- **Sub-workflow reference:**  
  None.

#### Shared Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Supplies rolling context memory to the orchestrator.
- **Configuration choices:**  
  Context window length set to `10`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI memory connection to Sustainability Orchestrator.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  - Memory is execution-context dependent and may not provide long-term persistence
  - Can introduce irrelevant previous context if item flows are heterogeneous
- **Sub-workflow reference:**  
  None.

#### Orchestrator Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces schema on orchestrator output.
- **Configuration choices:**  
  Manual JSON Schema requiring:
  - `action`: analyze / approve / document
  - `priority`: critical / high / medium / low
  - `delegateTo`: circular_economy / governance / documentation
  - `summary`: string
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI output parser connection to Sustainability Orchestrator.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  - Any deviation from enum values causes parser failure
  - Long or ambiguous model responses may need prompt tightening
- **Sub-workflow reference:**  
  None.

---

## 2.3 Specialist Agent: Circular Economy Analysis

**Overview:**  
This block provides detailed lifecycle and circular economy analysis as a callable AI tool. It can compute waste reduction metrics via a calculator tool and returns structured lifecycle insights.

**Nodes Involved:**  
- Circular Economy Agent
- Circular Economy Model
- Metrics Calculator
- Circular Economy Output

### Node Details

#### Circular Economy Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool for lifecycle analysis, reuse opportunities, and recycling strategy.
- **Configuration choices:**  
  - Task input is collected dynamically through `$fromAI('task', ...)`
  - System prompt requires structured fields such as:
    - productId
    - lifecycleStage
    - recyclingRate
    - reuseOpportunities
    - wasteReductionPotential
    - recommendedStrategy
    - complianceStatus
  - Structured parser enabled
- **Key expressions or variables used:**  
  `{{ $fromAI('task', 'The product lifecycle data or sustainability initiative to analyze') }}`
- **Input and output connections:**  
  Used as an AI tool by Sustainability Orchestrator  
  AI dependencies:
  - model from Circular Economy Model
  - calculator from Metrics Calculator
  - parser from Circular Economy Output
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases / failures:**  
  - If task text lacks product identifiers, output may hallucinate or produce weak mappings
  - Numeric calculations depend on the model actually invoking the calculator appropriately
- **Sub-workflow reference:**  
  None.

#### Circular Economy Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model for lifecycle analysis.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI language model connection to Circular Economy Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`; requires OpenAI credentials.
- **Edge cases / failures:**  
  Same OpenAI credential, access, and rate-limit concerns as above.
- **Sub-workflow reference:**  
  None.

#### Metrics Calculator
- **Type and role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  Numeric calculation tool for waste reduction and percentage calculations.
- **Configuration choices:**  
  Default calculator setup.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI tool connection to Circular Economy Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases / failures:**  
  - Garbage-in/garbage-out if the model sends malformed expressions
  - Does not validate business logic, only arithmetic
- **Sub-workflow reference:**  
  None.

#### Circular Economy Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Validates the agent’s lifecycle-analysis result.
- **Configuration choices:**  
  Requires:
  - productId
  - lifecycleStage
  - recyclingRate
  - reuseOpportunities array
  - wasteReductionPotential
  - recommendedStrategy
  - complianceStatus
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI output parser connection to Circular Economy Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  - Strict schema enforcement can fail if the model returns strings for numbers
- **Sub-workflow reference:**  
  None.

---

## 2.4 Specialist Agent: Governance Approval & Slack HITL

**Overview:**  
This block evaluates sustainability initiatives against governance and ESG criteria. It can request human approval through Slack and returns a structured approval decision.

**Nodes Involved:**  
- Sustainability Governance Agent
- Governance Model
- Governance Output
- Approval Request Tool
- Send Approval Notification

### Node Details

#### Sustainability Governance Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool for compliance evaluation, approval status, and governance actions.
- **Configuration choices:**  
  - Receives task via `$fromAI('task', ...)`
  - Prompt instructs agent to assess ESG/regulatory compliance and use approval tool for high-risk cases
  - Structured parser enabled
- **Key expressions or variables used:**  
  `{{ $fromAI('task', 'The sustainability initiative or circular economy proposal to evaluate for approval and compliance') }}`
- **Input and output connections:**  
  Used as AI tool by Sustainability Orchestrator  
  AI dependencies:
  - Governance Model
  - Governance Output
  - Approval Request Tool
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases / failures:**  
  - Approval logic depends heavily on prompt interpretation rather than deterministic rules
  - Compliance scores may be subjective without external standards data
- **Sub-workflow reference:**  
  None.

#### Governance Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model for governance decisions.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.1`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI language model connection to Sustainability Governance Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  OpenAI auth, quota, and model-availability issues.
- **Sub-workflow reference:**  
  None.

#### Governance Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces structured governance output.
- **Configuration choices:**  
  Required fields:
  - initiativeId
  - complianceScore
  - approvalStatus
  - riskLevel
  - regulatoryCompliance
  - requiredActions
  - approvalRationale
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI output parser connection to Sustainability Governance Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  - Enum mismatches for approval status or risk level
  - Boolean may be returned incorrectly as string by model
- **Sub-workflow reference:**  
  None.

#### Approval Request Tool
- **Type and role:** `n8n-nodes-base.slackHitlTool`  
  Human-in-the-loop Slack approval request tool callable by the governance agent.
- **Configuration choices:**  
  - OAuth2 Slack authentication
  - Message content generated via AI:
    `{{ $fromAI('approval_message', 'The approval request message to send to stakeholders') }}`
  - User is not preselected in the exported JSON
- **Key expressions or variables used:**  
  `$fromAI('approval_message', ...)`
- **Input and output connections:**  
  AI tool connection to Sustainability Governance Agent  
  Receives AI tool from Send Approval Notification.
- **Version-specific requirements:**  
  Uses `typeVersion 2.4`; requires Slack OAuth2.
- **Edge cases / failures:**  
  - Empty user selection may prevent usable approval routing
  - Slack bot scopes may be insufficient
  - HITL callback/webhook behavior must be supported by the environment
- **Sub-workflow reference:**  
  None.

#### Send Approval Notification
- **Type and role:** `n8n-nodes-base.slackTool`  
  Auxiliary Slack tool available to the approval request tool for notifications.
- **Configuration choices:**  
  - Channel ID is dynamically requested from AI:
    `{{ $fromAI('slack_channel', 'The Slack channel ID for approval notifications') }}`
  - Message also comes from AI
- **Key expressions or variables used:**  
  - `$fromAI('approval_message', ...)`
  - `$fromAI('slack_channel', ...)`
- **Input and output connections:**  
  AI tool connection into Approval Request Tool.
- **Version-specific requirements:**  
  Uses `typeVersion 2.4`; requires Slack OAuth2.
- **Edge cases / failures:**  
  - Model may not know a valid Slack channel ID
  - Slack tool invocation may fail if bot lacks channel access
- **Sub-workflow reference:**  
  None.

---

## 2.5 Specialist Agent: ESG Documentation

**Overview:**  
This block generates ESG-oriented reporting outputs and can create a Google Doc. It is intended for documentation, stakeholder messaging, and compliance summaries.

**Nodes Involved:**  
- Documentation Agent
- Documentation Model
- Documentation Output
- Create ESG Document

### Node Details

#### Documentation Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool for ESG-aligned documentation generation.
- **Configuration choices:**  
  - Receives task via `$fromAI('task', ...)`
  - Prompt instructs the agent to create summaries, metrics documentation, compliance reporting, and stakeholder messaging
  - Tool use allowed for Google Docs creation
  - Structured output parser enabled
- **Key expressions or variables used:**  
  `{{ $fromAI('task', 'The sustainability data, analysis results, or approval outcomes to document') }}`
- **Input and output connections:**  
  Used as AI tool by Sustainability Orchestrator  
  AI dependencies:
  - Documentation Model
  - Documentation Output
  - Create ESG Document
- **Version-specific requirements:**  
  Uses `typeVersion 3`.
- **Edge cases / failures:**  
  - Agent may create documents with vague titles if source data is weak
  - `keyMetrics` object is flexible, so output consistency may vary
- **Sub-workflow reference:**  
  None.

#### Documentation Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  GPT-4o model for report generation.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI language model connection to Documentation Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  OpenAI auth, rate limits, and access issues.
- **Sub-workflow reference:**  
  None.

#### Documentation Output
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces schema for documentation results.
- **Configuration choices:**  
  Required fields:
  - documentTitle
  - documentType
  - keyMetrics
  - recommendations
  - complianceStatus
  - stakeholderMessage
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  AI output parser connection to Documentation Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases / failures:**  
  - `documentType` must exactly match enum values
  - Free-form `keyMetrics` object may still be sparse or inconsistent
- **Sub-workflow reference:**  
  None.

#### Create ESG Document
- **Type and role:** `n8n-nodes-base.googleDocsTool`  
  Tool callable by the documentation agent to create a Google Doc.
- **Configuration choices:**  
  - Title supplied from AI:
    `{{ $fromAI('document_title', 'The title for the ESG sustainability document') }}`
  - Folder ID is placeholder and must be configured
  - Tool description text is misleading: it says “Sends approval notification to Slack channel,” which appears to be copied incorrectly
- **Key expressions or variables used:**  
  `$fromAI('document_title', ...)`
- **Input and output connections:**  
  AI tool connection to Documentation Agent.
- **Version-specific requirements:**  
  Uses `typeVersion 2`; requires Google credentials and supported n8n version with Google Docs tool node.
- **Edge cases / failures:**  
  - Placeholder folder ID must be replaced
  - Wrong Drive/Docs permissions will prevent document creation
  - Misleading tool description may affect AI tool selection quality
- **Sub-workflow reference:**  
  None.

---

## 2.6 Action Routing and Data Table Tracking

**Overview:**  
This block routes the orchestrator’s structured output into one of three tracking paths and writes records into n8n Data Tables for analytics, governance, or documentation.

**Nodes Involved:**  
- Route by Action
- Prepare Lifecycle Data
- Track Lifecycle Analysis
- Prepare Governance Data
- Track Governance Approvals
- Prepare Documentation Data
- Track ESG Documentation

### Node Details

#### Route by Action
- **Type and role:** `n8n-nodes-base.switch`  
  Rules-based router using the orchestrator’s `output.action`.
- **Configuration choices:**  
  Three branches:
  - `analyze`
  - `approve`
  - `document`
- **Key expressions or variables used:**  
  `{{ $json.output.action }}`
- **Input and output connections:**  
  Input from Sustainability Orchestrator  
  Outputs to:
  - Prepare Lifecycle Data
  - Prepare Governance Data
  - Prepare Documentation Data
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases / failures:**  
  - If parser output structure changes, `output.action` path may break
  - No fallback/default branch exists for unexpected values
- **Sub-workflow reference:**  
  None.

#### Prepare Lifecycle Data
- **Type and role:** `n8n-nodes-base.set`  
  Shapes lifecycle analysis tracking records.
- **Configuration choices:**  
  Produces fields:
  - `productId` from `output.summary`
  - `lifecycleStage` from `output.delegateTo`
  - `recyclingRate` default `0`
  - `wasteReductionPotential` default `0`
  - `recommendedStrategy` from `output.summary`
  - `complianceStatus` fixed `pending`
  - `timestamp` from current ISO time
  - `priority` from `output.priority`
- **Key expressions or variables used:**  
  - `{{ $json.output.summary }}`
  - `{{ $json.output.delegateTo }}`
  - `{{ $json.output.priority }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**  
  Input from Route by Action; output to Track Lifecycle Analysis.
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases / failures:**  
  - Mapped values are placeholders rather than true specialist-agent outputs
  - `productId` being set from summary is semantically weak
- **Sub-workflow reference:**  
  None.

#### Track Lifecycle Analysis
- **Type and role:** `n8n-nodes-base.dataTable`  
  Persists lifecycle tracking records.
- **Configuration choices:**  
  Auto-maps incoming fields to the target Data Table. Table ID is a placeholder.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from Prepare Lifecycle Data; output to Combine All Results.
- **Version-specific requirements:**  
  Uses `typeVersion 1.1`; requires Data Table support in the n8n instance.
- **Edge cases / failures:**  
  - Placeholder table ID must be replaced
  - Auto mapping may fail if table columns differ from incoming field names
- **Sub-workflow reference:**  
  None.

#### Prepare Governance Data
- **Type and role:** `n8n-nodes-base.set`  
  Shapes governance-tracking records.
- **Configuration choices:**  
  Produces fields:
  - `initiativeId` from summary
  - `complianceScore` default `0`
  - `approvalStatus` from action
  - `riskLevel` from priority
  - `regulatoryCompliance` fixed `true`
  - `approvalRationale` from summary
  - `timestamp`
  - `delegatedTo` from delegateTo
- **Key expressions or variables used:**  
  Similar output-based mappings from orchestrator result.
- **Input and output connections:**  
  Input from Route by Action; output to Track Governance Approvals.
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases / failures:**  
  - `approvalStatus` will be set to `approve`, not `approved/pending/rejected`, which may not match governance semantics
  - Values are not sourced from Governance Agent output
- **Sub-workflow reference:**  
  None.

#### Track Governance Approvals
- **Type and role:** `n8n-nodes-base.dataTable`  
  Persists governance records.
- **Configuration choices:**  
  Auto-map into a Data Table; placeholder table ID.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from Prepare Governance Data; output to Combine All Results.
- **Version-specific requirements:**  
  Uses `typeVersion 1.1`.
- **Edge cases / failures:**  
  - Table schema mismatch
  - Placeholder ID replacement required
- **Sub-workflow reference:**  
  None.

#### Prepare Documentation Data
- **Type and role:** `n8n-nodes-base.set`  
  Shapes documentation records for persistence.
- **Configuration choices:**  
  Produces:
  - `documentTitle` from summary
  - `documentType` from action
  - `complianceStatus` from delegateTo
  - `priority`
  - `timestamp`
  - `summary`
- **Key expressions or variables used:**  
  Output-derived expressions and `{{ $now.toISO() }}`
- **Input and output connections:**  
  Input from Route by Action; output to Track ESG Documentation.
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`.
- **Edge cases / failures:**  
  - `documentType` gets value `document`, but documentation schema expects `report/summary/communication`
  - Again this is orchestrator output, not Documentation Agent output
- **Sub-workflow reference:**  
  None.

#### Track ESG Documentation
- **Type and role:** `n8n-nodes-base.dataTable`  
  Persists ESG documentation records.
- **Configuration choices:**  
  Auto-map enabled; placeholder table ID.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from Prepare Documentation Data; output to Combine All Results.
- **Version-specific requirements:**  
  Uses `typeVersion 1.1`.
- **Edge cases / failures:**  
  Placeholder ID and schema mismatch concerns.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Aggregation, Reporting & Notifications

**Overview:**  
This block recombines the three possible tracking outputs, aggregates all items, and distributes final reporting notifications to Slack and Gmail.

**Nodes Involved:**  
- Combine All Results
- Aggregate Metrics
- Send Summary Notification
- Send Stakeholder Report

### Node Details

#### Combine All Results
- **Type and role:** `n8n-nodes-base.merge`  
  Merges outputs from the three tracking branches before aggregation.
- **Configuration choices:**  
  Configured for `3` inputs.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Inputs from:
  - Track Lifecycle Analysis
  - Track Governance Approvals
  - Track ESG Documentation  
  Output to Aggregate Metrics.
- **Version-specific requirements:**  
  Uses `typeVersion 3.2`.
- **Edge cases / failures:**  
  - Since Route by Action sends only one branch per execution, this merge may wait for inputs that never arrive depending on merge mode behavior
  - This is an important structural risk in the current design
- **Sub-workflow reference:**  
  None.

#### Aggregate Metrics
- **Type and role:** `n8n-nodes-base.aggregate`  
  Aggregates all incoming item data into one grouped object.
- **Configuration choices:**  
  Uses `aggregateAllItemData`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from Combine All Results; outputs to:
  - Send Summary Notification
  - Send Stakeholder Report
- **Version-specific requirements:**  
  Uses `typeVersion 1`.
- **Edge cases / failures:**  
  - Final object shape must support downstream `{{ $json.data.length }}`
  - If no data arrives, notifications may fail or report incorrect counts
- **Sub-workflow reference:**  
  None.

#### Send Summary Notification
- **Type and role:** `n8n-nodes-base.slack`  
  Sends a summary message to a Slack channel.
- **Configuration choices:**  
  - Channel mode
  - Slack channel ID placeholder
  - Includes total items processed with `{{ $json.data.length }}`
  - Workflow link disabled
- **Key expressions or variables used:**  
  - `{{ $json.data.length }}`
  - `{{ $now.toISO() }}`
- **Input and output connections:**  
  Input from Aggregate Metrics; no downstream node.
- **Version-specific requirements:**  
  Uses `typeVersion 2.4`; requires Slack OAuth2.
- **Edge cases / failures:**  
  - Placeholder channel ID must be replaced
  - If aggregate shape changes, `data.length` may be undefined
- **Sub-workflow reference:**  
  None.

#### Send Stakeholder Report
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends an HTML stakeholder summary email.
- **Configuration choices:**  
  - Recipient is placeholder
  - Subject includes current date
  - HTML message summarizes total item count and key activities
  - Attribution disabled
- **Key expressions or variables used:**  
  - `{{ $json.data.length }}`
  - `{{ $now.toISO() }}`
  - `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Input and output connections:**  
  Input from Aggregate Metrics; no downstream node.
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`; requires Gmail OAuth2.
- **Edge cases / failures:**  
  - Placeholder email must be replaced
  - Gmail OAuth scopes and sending permissions must be valid
  - HTML content may be rejected by strict enterprise mail policies if not aligned
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monitor Lifecycle Data | scheduleTrigger | Daily trigger for lifecycle monitoring |  | Combine Input Sources; Fetch External Sustainability Data | ## Data Ingestion & Normalisation<br>**Why** — Triple-source ingestion (scheduled, API, form) merged into a unified record prevents data gaps and fragmentation before orchestration. |
| Fetch External Sustainability Data | httpRequest | Pull sustainability data from external API | Monitor Lifecycle Data | Combine Input Sources | ## Data Ingestion & Normalisation<br>**Why** — Triple-source ingestion (scheduled, API, form) merged into a unified record prevents data gaps and fragmentation before orchestration. |
| Receive Sustainability Data | webhook | Receive external POSTed sustainability records |  | Combine Input Sources | ## Data Ingestion & Normalisation<br>**Why** — Triple-source ingestion (scheduled, API, form) merged into a unified record prevents data gaps and fragmentation before orchestration. |
| Initiative Submission Form | formTrigger | Manual sustainability initiative intake form |  | Combine Input Sources | ## Data Ingestion & Normalisation<br>**Why** — Triple-source ingestion (scheduled, API, form) merged into a unified record prevents data gaps and fragmentation before orchestration. |
| Combine Input Sources | merge | Merge intake sources into unified stream | Monitor Lifecycle Data; Fetch External Sustainability Data; Receive Sustainability Data; Initiative Submission Form | Sustainability Orchestrator | ## Data Ingestion & Normalisation<br>**Why** — Triple-source ingestion (scheduled, API, form) merged into a unified record prevents data gaps and fragmentation before orchestration. |
| Sustainability Orchestrator | @n8n/n8n-nodes-langchain.agent | Central AI classifier and delegator | Combine Input Sources | Route by Action | ## Sustainability Orchestrator & Specialist Agents<br>**Why** — Central coordinator with shared memory delegates tasks to Circular Economy, Governance, and Documentation agents for parallel, context-aware processing. |
| Orchestrator Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for orchestrator |  | Sustainability Orchestrator | ## Sustainability Orchestrator & Specialist Agents<br>**Why** — Central coordinator with shared memory delegates tasks to Circular Economy, Governance, and Documentation agents for parallel, context-aware processing. |
| Shared Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation memory for orchestrator |  | Sustainability Orchestrator | ## Sustainability Orchestrator & Specialist Agents<br>**Why** — Central coordinator with shared memory delegates tasks to Circular Economy, Governance, and Documentation agents for parallel, context-aware processing. |
| Orchestrator Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce orchestrator response schema |  | Sustainability Orchestrator |  |
| Circular Economy Agent | @n8n/n8n-nodes-langchain.agentTool | AI tool for lifecycle and circular economy analysis |  | Sustainability Orchestrator | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring.<br>## Sustainability Orchestrator & Specialist Agents<br>**Why** — Central coordinator with shared memory delegates tasks to Circular Economy, Governance, and Documentation agents for parallel, context-aware processing. |
| Circular Economy Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for circular analysis |  | Circular Economy Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Metrics Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Calculator tool for waste and percentage metrics |  | Circular Economy Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Circular Economy Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce circular analysis schema |  | Circular Economy Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Sustainability Governance Agent | @n8n/n8n-nodes-langchain.agentTool | AI tool for approvals and compliance |  | Sustainability Orchestrator | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Governance Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for governance evaluation |  | Sustainability Governance Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Governance Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce governance response schema |  | Sustainability Governance Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Approval Request Tool | slackHitlTool | Slack human-in-the-loop approval request |  | Sustainability Governance Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Send Approval Notification | slackTool | Slack notification tool used by approval flow |  | Approval Request Tool | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Documentation Agent | @n8n/n8n-nodes-langchain.agentTool | AI tool for ESG reporting and communications |  | Sustainability Orchestrator | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Documentation Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4o model for documentation generation |  | Documentation Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Documentation Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce documentation schema |  | Documentation Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Create ESG Document | googleDocsTool | Create Google Docs output from documentation agent |  | Documentation Agent | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Route by Action | switch | Route by orchestrator action | Sustainability Orchestrator | Prepare Lifecycle Data; Prepare Governance Data; Prepare Documentation Data | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Prepare Lifecycle Data | set | Shape lifecycle tracking record | Route by Action | Track Lifecycle Analysis | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Track Lifecycle Analysis | dataTable | Persist lifecycle analytics row | Prepare Lifecycle Data | Combine All Results | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Prepare Governance Data | set | Shape governance tracking record | Route by Action | Track Governance Approvals | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Track Governance Approvals | dataTable | Persist governance approval row | Prepare Governance Data | Combine All Results | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Prepare Documentation Data | set | Shape documentation tracking record | Route by Action | Track ESG Documentation | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Track ESG Documentation | dataTable | Persist ESG documentation row | Prepare Documentation Data | Combine All Results | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Combine All Results | merge | Merge tracked branch results | Track Lifecycle Analysis; Track Governance Approvals; Track ESG Documentation | Aggregate Metrics | ## Aggregate, Report & Notify<br>**Why** — Consolidates multi-agent outputs into a metrics summary, then delivers Gmail reports and Slack notifications to stakeholders automatically. |
| Aggregate Metrics | aggregate | Aggregate all processed records | Combine All Results | Send Summary Notification; Send Stakeholder Report | ## Aggregate, Report & Notify<br>**Why** — Consolidates multi-agent outputs into a metrics summary, then delivers Gmail reports and Slack notifications to stakeholders automatically. |
| Send Summary Notification | slack | Post workflow summary to Slack | Aggregate Metrics |  | ## Aggregate, Report & Notify<br>**Why** — Consolidates multi-agent outputs into a metrics summary, then delivers Gmail reports and Slack notifications to stakeholders automatically. |
| Send Stakeholder Report | gmail | Send HTML stakeholder email report | Aggregate Metrics |  | ## Aggregate, Report & Notify<br>**Why** — Consolidates multi-agent outputs into a metrics summary, then delivers Gmail reports and Slack notifications to stakeholders automatically. |
| Sticky Note | stickyNote | Documentation note |  |  | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Gmail account with OAuth credentials<br>- Google Sheets with tracking tabs pre-created<br>## Use Cases<br>- Enterprises automating circular economy programme scoring across product lines<br>## Customisation<br>- Swap Circular Economy scoring thresholds to align with GRI, Ellen MacArthur, or regional frameworks<br>## Benefits<br>- Triple-source ingestion eliminates sustainability data blind spots |
| Sticky Note1 | stickyNote | Setup guidance note |  |  | ## Setup Steps<br>1. Import workflow; configure the lifecycle monitor trigger interval and external sustainability API endpoint URL.<br>2. Add AI model credentials to the Sustainability Orchestrator and Documentation Agent.<br>3. Connect Slack credentials to the Approval Request Tool and Send Summary Notification nodes.<br>4. Link Gmail credentials to the Send Stakeholder Report node.<br>5. Configure Google Sheets credentials; set sheet IDs for ESG Documentation, Lifecycle Analytics, and Governance.<br>6. Set scoring thresholds in the Metrics Calculator node. |
| Sticky Note2 | stickyNote | Workflow explanation note |  |  | ## How It Works<br>This workflow automates end-to-end sustainability lifecycle management for corporate sustainability teams, ESG governance officers, and circular economy programme leads. It addresses the challenge of coordinating fragmented sustainability inputs, scheduled monitoring, platform data feeds, and manual initiative submissions into a single governed, auditable reporting pipeline. Data enters from three sources: a lifecycle monitor, an external sustainability data API, and an initiative submission form. Inputs are merged and passed to a Sustainability Orchestrator with shared memory, which delegates to three specialist agents: a Circular Economy Agent (metrics calculation and circular output scoring), a Sustainability Governance Agent (governance evaluation and Slack-based approvals), and a Documentation Agent (ESG document creation). The orchestrator then routes actions for documentation tracking, lifecycle analytics, and governance approvals, before combining results, aggregating metrics, and sending a stakeholder report via Gmail and a summary notification via Slack. |
| Sticky Note3 | stickyNote | Routing note |  |  | ## Route by Action<br>**Why** — Rules-based routing directs outputs to documentation tracking, lifecycle analytics, or governance approval paths. |
| Sticky Note4 | stickyNote | Governance/documentation note |  |  | ## Governance Approval & ESG Documentation<br>**Why** — Routes Slack-based approval requests and auto-generates ESG documents each cycle, eliminating manual oversight and authoring. |
| Sticky Note5 | stickyNote | Orchestration note |  |  | ## Sustainability Orchestrator & Specialist Agents<br>**Why** — Central coordinator with shared memory delegates tasks to Circular Economy, Governance, and Documentation agents for parallel, context-aware processing. |
| Sticky Note6 | stickyNote | Ingestion note |  |  | ## Data Ingestion & Normalisation<br>**Why** — Triple-source ingestion (scheduled, API, form) merged into a unified record prevents data gaps and fragmentation before orchestration. |
| Sticky Note7 | stickyNote | Reporting note |  |  | ## Aggregate, Report & Notify<br>**Why** — Consolidates multi-agent outputs into a metrics summary, then delivers Gmail reports and Slack notifications to stakeholders automatically. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like:  
   **AI-powered sustainability lifecycle orchestration with circular economy tracking**.

2. **Add a Schedule Trigger node** named **Monitor Lifecycle Data**.
   - Type: `Schedule Trigger`
   - Configure it to run daily at hour `6`
   - Confirm your instance timezone is correct

3. **Add an HTTP Request node** named **Fetch External Sustainability Data**.
   - Connect it from **Monitor Lifecycle Data**
   - Set method to `GET` unless your API requires another method
   - Set URL to your sustainability API endpoint
   - Enable query parameters
   - Add:
     - `date` = `{{ $now.toFormat('yyyy-MM-dd') }}`
   - Set error handling to continue on error if you want parity with the original workflow
   - Replace placeholder endpoint with the real API URL
   - Add authentication if your API requires headers, OAuth2, token, or basic auth

4. **Add a Webhook node** named **Receive Sustainability Data**.
   - Type: `Webhook`
   - Method: `POST`
   - Path: `sustainability-data`
   - Response mode: `Last Node`
   - This becomes one entry point for system submissions

5. **Add a Form Trigger node** named **Initiative Submission Form**.
   - Form title: `Sustainability Initiative Submission`
   - Description: `Submit recycling proposals, refurbishment requests, or reuse opportunities`
   - Disable attribution append option
   - Add fields:
     1. Initiative Type
     2. Description as textarea
     3. Product/Material
     4. Estimated Impact (kg waste reduced) as number

6. **Add a Merge node** named **Combine Input Sources**.
   - Configure for multiple inputs
   - Important: in the exported workflow, the node says `numberInputs: 3` but actually receives 4 inbound connections.  
   - When rebuilding, set it to support all actual incoming sources or redesign intake more safely.
   - Connect:
     - **Monitor Lifecycle Data**
     - **Fetch External Sustainability Data**
     - **Receive Sustainability Data**
     - **Initiative Submission Form**
   - Output goes to the orchestrator

7. **Add an OpenAI Chat Model node** named **Orchestrator Model**.
   - Type: `OpenAI Chat Model` from LangChain nodes
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach valid OpenAI credentials

8. **Add a Memory Buffer Window node** named **Shared Memory**.
   - Set context window length to `10`

9. **Add a Structured Output Parser node** named **Orchestrator Output Parser**.
   - Use manual schema
   - Define object fields:
     - `action`: `analyze`, `approve`, `document`
     - `priority`: `critical`, `high`, `medium`, `low`
     - `delegateTo`: `circular_economy`, `governance`, `documentation`
     - `summary`: string

10. **Add an AI Agent node** named **Sustainability Orchestrator**.
    - Input text expression:
      `{{ $json.body ? JSON.stringify($json.body) : JSON.stringify($json) }}`
    - System message should instruct it to:
      - analyze sustainability input
      - choose action
      - choose priority
      - choose delegate target
      - return a summary
    - Enable structured output parser
    - Connect:
      - Main input from **Combine Input Sources**
      - AI model from **Orchestrator Model**
      - AI memory from **Shared Memory**
      - AI output parser from **Orchestrator Output Parser**

11. **Add a specialist AI Tool Agent** named **Circular Economy Agent**.
    - Tool description: circular economy and lifecycle analysis
    - Text:
      `{{ $fromAI('task', 'The product lifecycle data or sustainability initiative to analyze') }}`
    - System message should request:
      - productId
      - lifecycleStage
      - recyclingRate
      - reuseOpportunities
      - wasteReductionPotential
      - recommendedStrategy
      - complianceStatus
    - Enable structured output parsing

12. **Add another OpenAI Chat Model** named **Circular Economy Model**.
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Connect as the language model for **Circular Economy Agent**

13. **Add a Calculator Tool node** named **Metrics Calculator**.
    - Connect it as an AI tool to **Circular Economy Agent**

14. **Add a Structured Output Parser** named **Circular Economy Output**.
    - Create schema with:
      - productId: string
      - lifecycleStage: string
      - recyclingRate: number
      - reuseOpportunities: array of strings
      - wasteReductionPotential: number
      - recommendedStrategy: string
      - complianceStatus: string
    - Connect it to **Circular Economy Agent**

15. **Add a specialist AI Tool Agent** named **Sustainability Governance Agent**.
    - Text:
      `{{ $fromAI('task', 'The sustainability initiative or circular economy proposal to evaluate for approval and compliance') }}`
    - System message should instruct:
      - ESG/regulatory assessment
      - approval requirement determination
      - validation of environmental claims
      - use human approval for high-risk cases
      - structured output fields:
        initiativeId, complianceScore, approvalStatus, riskLevel, regulatoryCompliance, requiredActions, approvalRationale

16. **Add another OpenAI Chat Model** named **Governance Model**.
    - Model: `gpt-4o`
    - Temperature: `0.1`
    - Connect to **Sustainability Governance Agent**

17. **Add a Structured Output Parser** named **Governance Output**.
    - Schema fields:
      - initiativeId: string
      - complianceScore: number
      - approvalStatus: approved/pending/rejected
      - riskLevel: low/medium/high/critical
      - regulatoryCompliance: boolean
      - requiredActions: array of strings
      - approvalRationale: string
    - Connect to **Sustainability Governance Agent**

18. **Add a Slack Tool node** named **Send Approval Notification**.
    - OAuth2 Slack credential required
    - Text:
      `{{ $fromAI('approval_message', 'The approval notification message') }}`
    - Channel ID:
      `{{ $fromAI('slack_channel', 'The Slack channel ID for approval notifications') }}`
    - This acts as a helper tool

19. **Add a Slack HITL Tool node** named **Approval Request Tool**.
    - OAuth2 Slack credential required
    - Message:
      `{{ $fromAI('approval_message', 'The approval request message to send to stakeholders') }}`
    - Configure user or approval recipient target properly; the exported JSON leaves it blank
    - Connect **Send Approval Notification** as an AI tool to **Approval Request Tool**
    - Connect **Approval Request Tool** as an AI tool to **Sustainability Governance Agent**

20. **Add a specialist AI Tool Agent** named **Documentation Agent**.
    - Text:
      `{{ $fromAI('task', 'The sustainability data, analysis results, or approval outcomes to document') }}`
    - System message should request:
      - executive summary
      - metrics and impact analysis
      - compliance validation report
      - stakeholder communications
      - ESG performance documentation
      - structured return:
        documentTitle, documentType, keyMetrics, recommendations, complianceStatus, stakeholderMessage

21. **Add another OpenAI Chat Model** named **Documentation Model**.
    - Model: `gpt-4o`
    - Temperature: `0.3`
    - Connect to **Documentation Agent**

22. **Add a Structured Output Parser** named **Documentation Output**.
    - Schema:
      - documentTitle: string
      - documentType: report/summary/communication
      - keyMetrics: object
      - recommendations: array of strings
      - complianceStatus: string
      - stakeholderMessage: string
    - Connect to **Documentation Agent**

23. **Add a Google Docs Tool node** named **Create ESG Document**.
    - Set title:
      `{{ $fromAI('document_title', 'The title for the ESG sustainability document') }}`
    - Set target Google Drive folder ID
    - Add Google credentials with access to Docs/Drive
    - Connect this node as an AI tool to **Documentation Agent**
    - Correct the tool description so it reflects document creation, not Slack notification

24. **Connect the three specialist agents as tools to the Sustainability Orchestrator**:
    - **Circular Economy Agent**
    - **Sustainability Governance Agent**
    - **Documentation Agent**

25. **Add a Switch node** named **Route by Action**.
    - Expression path for routing:
      `{{ $json.output.action }}`
    - Create three branches:
      - equals `analyze`
      - equals `approve`
      - equals `document`
    - Connect from **Sustainability Orchestrator**

26. **Add a Set node** named **Prepare Lifecycle Data** on the `analyze` branch.
    - Fields:
      - `productId` = `{{ $json.output.summary }}`
      - `lifecycleStage` = `{{ $json.output.delegateTo }}`
      - `recyclingRate` = `0`
      - `wasteReductionPotential` = `0`
      - `recommendedStrategy` = `{{ $json.output.summary }}`
      - `complianceStatus` = `pending`
      - `timestamp` = `{{ $now.toISO() }}`
      - `priority` = `{{ $json.output.priority }}`
    - Note: this reproduces the export exactly, but it does not use actual Circular Economy Agent output. A production rebuild should improve that.

27. **Add a Data Table node** named **Track Lifecycle Analysis**.
    - Map input automatically
    - Choose or create a Data Table for lifecycle analysis
    - Connect from **Prepare Lifecycle Data**

28. **Add a Set node** named **Prepare Governance Data** on the `approve` branch.
    - Fields:
      - `initiativeId` = `{{ $json.output.summary }}`
      - `complianceScore` = `0`
      - `approvalStatus` = `{{ $json.output.action }}`
      - `riskLevel` = `{{ $json.output.priority }}`
      - `regulatoryCompliance` = `true`
      - `approvalRationale` = `{{ $json.output.summary }}`
      - `timestamp` = `{{ $now.toISO() }}`
      - `delegatedTo` = `{{ $json.output.delegateTo }}`
    - Again, this mirrors the export but is only a simplified record.

29. **Add a Data Table node** named **Track Governance Approvals**.
    - Auto-map fields
    - Select/create governance approvals Data Table
    - Connect from **Prepare Governance Data**

30. **Add a Set node** named **Prepare Documentation Data** on the `document` branch.
    - Fields:
      - `documentTitle` = `{{ $json.output.summary }}`
      - `documentType` = `{{ $json.output.action }}`
      - `complianceStatus` = `{{ $json.output.delegateTo }}`
      - `priority` = `{{ $json.output.priority }}`
      - `timestamp` = `{{ $now.toISO() }}`
      - `summary` = `{{ $json.output.summary }}`

31. **Add a Data Table node** named **Track ESG Documentation**.
    - Auto-map fields
    - Select/create ESG documentation Data Table
    - Connect from **Prepare Documentation Data**

32. **Add a Merge node** named **Combine All Results**.
    - Configure for `3` inputs
    - Connect:
      - **Track Lifecycle Analysis**
      - **Track Governance Approvals**
      - **Track ESG Documentation**
    - Important: this may stall if only one switch branch runs per execution.  
      In a more robust rebuild, use append mode, pass-through design, or separate notification handling.

33. **Add an Aggregate node** named **Aggregate Metrics**.
    - Set aggregate mode to collect all item data
    - Connect from **Combine All Results**

34. **Add a Slack node** named **Send Summary Notification**.
    - OAuth2 Slack credential required
    - Send to a fixed channel
    - Text:
      - Include total items processed via `{{ $json.data.length }}`
      - Include timestamp via `{{ $now.toISO() }}`
    - Connect from **Aggregate Metrics**

35. **Add a Gmail node** named **Send Stakeholder Report**.
    - OAuth2 Gmail credential required
    - Recipient: stakeholder email address
    - Subject:
      `Sustainability & Circular Economy Report - {{ $now.toFormat('yyyy-MM-dd') }}`
    - HTML body should summarize:
      - total items processed
      - current timestamp
      - key activities
    - Connect from **Aggregate Metrics**

36. **Review credentials and placeholders**.
    - OpenAI API credential for all model nodes
    - Slack OAuth2 for:
      - Approval Request Tool
      - Send Approval Notification
      - Send Summary Notification
    - Gmail OAuth2 for Send Stakeholder Report
    - Google Docs / Drive credential for Create ESG Document
    - Replace placeholders for:
      - external sustainability API endpoint
      - Google Drive folder ID
      - Slack channel ID
      - stakeholder email
      - Data Table IDs

37. **Test each entry point separately**.
    - Schedule path
    - Webhook path
    - Form submission path

38. **Validate structural issues before production use**.
    - Ensure merge node input counts match actual connections
    - Decide whether specialist agents should only be callable tools or whether their outputs should also feed tracking nodes directly
    - Fix final merge behavior so one branch does not block the workflow
    - Align `documentType` and `approvalStatus` tracking fields with actual schema expectations

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API key (or compatible LLM) is required | Prerequisites |
| Slack workspace with bot credentials is required | Prerequisites |
| Gmail account with OAuth credentials is required | Prerequisites |
| Google Sheets with tracking tabs pre-created | Mentioned in note, though the actual exported workflow uses n8n Data Tables rather than Google Sheets |
| Enterprises automating circular economy programme scoring across product lines | Use case |
| Swap Circular Economy scoring thresholds to align with GRI, Ellen MacArthur, or regional frameworks | Customisation |
| Triple-source ingestion eliminates sustainability data blind spots | Benefit |
| Setup guidance includes configuring lifecycle trigger interval, API endpoint URL, AI credentials, Slack, Gmail, tracking storage, and calculator thresholds | Setup considerations |
| The workflow description note explains the intended end-to-end governed reporting pipeline and stakeholder communication flow | Functional context |

## Additional implementation observations

- The workflow title in metadata and the user-supplied title differ slightly; the exported workflow name is:  
  **AI-powered sustainability lifecycle orchestration with circular economy tracking**
- Several placeholder values must be replaced before execution.
- The design intention suggests rich multi-agent delegation, but the actual persistence layer mostly stores orchestrator-level summaries rather than specialist-agent outputs.
- The sticky note mentions **Google Sheets**, but the workflow uses **n8n Data Tables** and **Google Docs**.
- The merge and routing logic should be reviewed carefully because the current exported structure may not behave as intended in single-branch executions.