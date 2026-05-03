Automate privacy risk detection, approvals, and audit reports with GPT-4o, Slack, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/automate-privacy-risk-detection--approvals--and-audit-reports-with-gpt-4o--slack--gmail--and-google-sheets-14441


# Automate privacy risk detection, approvals, and audit reports with GPT-4o, Slack, Gmail, and Google Sheets

# 1. Workflow Overview

This workflow automates privacy governance for two kinds of inputs:

- **Real-time privacy/data usage events** received through a webhook
- **Scheduled compliance audits** triggered on a recurring schedule

Its purpose is to evaluate data handling activities against privacy regulations, classify privacy risk, optionally involve human approval through Slack, log the outcome for auditability, and distribute a compliance report by email.

## 1.1 Input Reception

The workflow has **two entry points**:

- **Data Usage Event Trigger** receives POST requests for live privacy-related events
- **Scheduled Compliance Audit** starts periodic audits automatically

Both paths feed the same AI governance orchestration layer.

## 1.2 AI Governance Orchestration

The central **Privacy Governance Agent** uses GPT-4o and several connected tools to:

- interpret the incoming event or audit request
- delegate legal/privacy-law analysis
- delegate privacy risk assessment
- decide whether approval is needed
- generate a structured compliance result

This block depends on:
- a governance model
- a structured output parser
- specialist AI tools for legal/privacy and risk evaluation
- supporting tools for audit logging, approval lookup, and human approval requests

## 1.3 Specialist Privacy and Risk Evaluation

Two specialist AI tool chains support the governance agent:

- **Data Privacy Agent Tool** evaluates regulatory compliance using a dedicated model and an external legal database API
- **Risk Detection Agent Tool** evaluates privacy/security risk using a dedicated model

Additional governance support tools include:
- **Audit Log Tool**
- **Approval History Tool**
- **Approval Request Tool**
- **Slack Notification Tool**

## 1.4 Risk-Based Routing and Alerting

After the governance agent returns structured output, a **Switch** node routes the result by risk level:

- **CRITICAL** → immediate critical Slack alert
- **HIGH** → high-risk Slack alert
- **MEDIUM / LOW** → no immediate alert branch, but still continue to record/reporting

All paths converge into audit record creation.

## 1.5 Audit Storage and Reporting

Every evaluated event or audit is transformed into a normalized audit/compliance record, stored in a data table, formatted into a report, and emailed via Gmail to a designated recipient such as a DPO or privacy lead.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block receives either real-time privacy events or scheduled audit runs. It standardizes the starting point so both operational monitoring and periodic review use the same downstream governance logic.

### Nodes Involved
- Data Usage Event Trigger
- Scheduled Compliance Audit

### Node Details

#### Data Usage Event Trigger
- **Type / role:** `n8n-nodes-base.webhook`  
  Receives incoming HTTP POST payloads representing privacy/data usage events.
- **Configuration choices:**
  - Path: `privacy-event`
  - Method: `POST`
  - No extra webhook options are configured
- **Key expressions / variables:**
  - Downstream nodes expect fields such as `eventId`, `eventType`, and other event details in `$json`
- **Input / output connections:**
  - Entry node, no inputs
  - Outputs to **Privacy Governance Agent**
- **Version-specific requirements:**
  - Type version `2.1`
- **Edge cases / failures:**
  - Invalid request body shape may still pass to downstream AI, but may degrade result quality
  - Missing `eventId` is tolerated later because fallback IDs are generated
  - If webhook is not activated/published, external systems cannot call it
- **Sub-workflow reference:** None

#### Scheduled Compliance Audit
- **Type / role:** `n8n-nodes-base.scheduleTrigger`  
  Starts recurring compliance audits automatically.
- **Configuration choices:**
  - Interval rule configured with `triggerAtHour: 9`
  - This implies a scheduled trigger around 09:00 according to the workflow/server timezone
- **Key expressions / variables:**
  - No explicit payload is created here, so downstream logic relies on trigger output plus governance prompt logic
  - The governance prompt checks `eventType === 'scheduled_audit'`, but this node does not itself set that field
- **Input / output connections:**
  - Entry node, no inputs
  - Outputs to **Privacy Governance Agent**
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - Timezone ambiguity can lead to unexpected trigger time
  - Because no Set node adds `eventType: scheduled_audit`, the governance prompt may not enter its intended audit-specific branch unless n8n payload is enriched elsewhere
- **Sub-workflow reference:** None

---

## Block 2 — AI Governance Orchestration

### Overview
This is the central decision-making layer. The Privacy Governance Agent receives either a live event or a scheduled audit trigger, uses GPT-4o, invokes specialist tools as needed, and returns structured compliance output.

### Nodes Involved
- Privacy Governance Agent
- Governance Agent Model
- Compliance Output Parser

### Node Details

#### Privacy Governance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`  
  Main orchestration agent coordinating analysis, tool usage, approvals, and final structured response.
- **Configuration choices:**
  - Prompt text dynamically changes:
    - if `$json.eventType === 'scheduled_audit'`: perform comprehensive privacy audit
    - otherwise: evaluate a specific data usage event with `JSON.stringify($json)`
  - Rich system message defines the governance responsibilities:
    - analyze event/audit
    - delegate legal evaluation
    - delegate risk assessment
    - decide approvals/access control
    - request human approval for high-risk actions
    - generate comprehensive compliance reports
  - Structured output is enabled through `hasOutputParser: true`
- **Key expressions / variables:**
  - `={{ $json.eventType === 'scheduled_audit' ? '...' : 'Evaluate ... ' + JSON.stringify($json) }}`
- **Input / output connections:**
  - Main inputs from:
    - **Data Usage Event Trigger**
    - **Scheduled Compliance Audit**
  - AI language model input from **Governance Agent Model**
  - AI output parser input from **Compliance Output Parser**
  - AI tool connections from:
    - **Data Privacy Agent Tool**
    - **Risk Detection Agent Tool**
    - **Audit Log Tool**
    - **Approval Request Tool**
    - **Approval History Tool**
  - Main output to **Route by Risk Level**
- **Version-specific requirements:**
  - Type version `3.1`
  - Requires compatible LangChain/AI node support in the n8n version used
- **Edge cases / failures:**
  - LLM may produce incomplete fields unless parser/schema enforcement works properly
  - Tool-use failure can reduce response quality or break execution
  - If the scheduled path lacks `eventType`, audit wording may not be used
  - Token/context issues may occur with very large incoming event payloads
- **Sub-workflow reference:** None

#### Governance Agent Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM backend for the governance agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for relatively stable but still flexible orchestration
  - Uses OpenAI credentials
- **Key expressions / variables:** None
- **Input / output connections:**
  - AI language model output to **Privacy Governance Agent**
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires valid OpenAI credentials and model access
- **Edge cases / failures:**
  - Authentication errors
  - Model access restrictions
  - Quota/rate limiting
  - OpenAI timeout/network failures
- **Sub-workflow reference:** None

#### Compliance Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the agent’s final output into a specific JSON schema.
- **Configuration choices:**
  - Manual schema with fields:
    - `complianceStatus` string
    - `riskLevel` string
    - `regulationViolations` array of strings
    - `approvalRequired` boolean
    - `findings` string
    - `recommendations` array of strings
    - `auditTrail` object
- **Key expressions / variables:** None
- **Input / output connections:**
  - AI output parser connection to **Privacy Governance Agent**
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - If model output cannot be coerced into the schema, execution may fail
  - No enum restriction exists on `riskLevel`, so values outside LOW/MEDIUM/HIGH/CRITICAL are possible and would later fail to match switch cases cleanly
- **Sub-workflow reference:** None

---

## Block 3 — Specialist Privacy Evaluation

### Overview
This block provides legal/privacy-law analysis. The governance agent can delegate privacy compliance evaluation to a dedicated AI tool, which itself can query an external legal/regulatory API.

### Nodes Involved
- Data Privacy Agent Tool
- Privacy Agent Model
- Legal Database API Tool

### Node Details

#### Data Privacy Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool for evaluating privacy-law compliance.
- **Configuration choices:**
  - Tool text comes from the parent AI using `$fromAI(...)`
  - Detailed system message covers GDPR, PDPA, CCPA, and global privacy-law analysis
  - Tool description explains its purpose for the orchestrator
- **Key expressions / variables:**
  - `={{ $fromAI('privacy_evaluation_task', 'The data handling practice or event to evaluate for privacy law compliance') }}`
- **Input / output connections:**
  - AI tool available to **Privacy Governance Agent**
  - Uses **Privacy Agent Model** as its language model
  - Can invoke **Legal Database API Tool**
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases / failures:**
  - If the parent agent gives vague instructions, legal analysis may be generic
  - Hallucinated legal references remain possible if external API is not used
  - Tool-call chains may fail if model or HTTP tool are unavailable
- **Sub-workflow reference:** None

#### Privacy Agent Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Dedicated GPT-4o model for privacy-law evaluation.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.1` for high consistency
- **Key expressions / variables:** None
- **Input / output connections:**
  - AI language model output to **Data Privacy Agent Tool**
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - Same OpenAI auth, quota, timeout, and rate-limit risks as above
- **Sub-workflow reference:** None

#### Legal Database API Tool
- **Type / role:** `n8n-nodes-base.httpRequestTool`  
  AI-invokable HTTP tool for querying external legal/compliance APIs.
- **Configuration choices:**
  - URL is AI-provided via `$fromAI`
  - Tool description states it can query regulations or compliance endpoints
  - No explicit auth or headers are configured in the JSON
- **Key expressions / variables:**
  - `={{ $fromAI('api_endpoint', 'The legal database API endpoint to query (e.g., /api/regulations/gdpr or /api/compliance/pdpa)') }}`
- **Input / output connections:**
  - AI tool available to **Data Privacy Agent Tool**
- **Version-specific requirements:**
  - Type version `4.4`
- **Edge cases / failures:**
  - The AI may generate an incomplete or invalid URL
  - Relative endpoints like `/api/regulations/gdpr` will fail unless a full base URL is actually configured in the environment or manually adjusted
  - Missing authentication, headers, or query params could prevent access
  - External API latency or downtime can break the tool path
- **Sub-workflow reference:** None

---

## Block 4 — Risk Evaluation, Audit Logging, and Approval Support

### Overview
This block gives the governance agent supporting tools for risk scoring, historical approval lookup, audit logging, and human approval escalation through Slack.

### Nodes Involved
- Risk Detection Agent Tool
- Risk Agent Model
- Audit Log Tool
- Approval Request Tool
- Slack Notification Tool
- Approval History Tool

### Node Details

#### Risk Detection Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool for privacy/security risk analysis.
- **Configuration choices:**
  - Task input delegated from parent AI via `$fromAI`
  - System message covers:
    - breach/security risk
    - access overreach
    - retention/deletion risk
    - third-party sharing
    - consent gaps
    - sensitive data exposure
    - risk scoring LOW/MEDIUM/HIGH/CRITICAL
- **Key expressions / variables:**
  - `={{ $fromAI('risk_assessment_task', 'The data handling activity or event to assess for privacy risks') }}`
- **Input / output connections:**
  - AI tool available to **Privacy Governance Agent**
  - Uses **Risk Agent Model**
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases / failures:**
  - If risk terminology returned is inconsistent with switch routing, later branch selection may fail
  - AI may produce nuanced labels not exactly matching expected values
- **Sub-workflow reference:** None

#### Risk Agent Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Dedicated GPT-4o model for risk classification.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.15`
- **Key expressions / variables:** None
- **Input / output connections:**
  - AI language model output to **Risk Detection Agent Tool**
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases / failures:**
  - OpenAI auth, access, quota, or timeout issues
- **Sub-workflow reference:** None

#### Audit Log Tool
- **Type / role:** `n8n-nodes-base.dataTableTool`  
  AI-invokable tool to record audit information into an n8n Data Table.
- **Configuration choices:**
  - Data table ID is a placeholder and must be replaced
  - Columns are auto-mapped from tool input
  - Tool description says it records compliance evaluations and governance decisions
- **Key expressions / variables:** None explicit
- **Input / output connections:**
  - AI tool available to **Privacy Governance Agent**
- **Version-specific requirements:**
  - Type version `1.1`
  - Requires n8n Data Tables availability
- **Edge cases / failures:**
  - Placeholder table ID must be replaced
  - Auto-mapping may store inconsistent columns if AI sends varying fields
  - Permission or schema mismatch can fail inserts
- **Sub-workflow reference:** None

#### Approval Request Tool
- **Type / role:** `n8n-nodes-base.slackHitlTool`  
  Human-in-the-loop Slack approval tool.
- **Configuration choices:**
  - OAuth2 Slack authentication
  - User field is currently blank
  - Webhook-based interaction for approvals
- **Key expressions / variables:** None directly configured in the visible parameters
- **Input / output connections:**
  - AI tool available to **Privacy Governance Agent**
  - Has AI tool access to **Slack Notification Tool**
- **Version-specific requirements:**
  - Type version `2.4`
  - Requires Slack OAuth2 credentials and proper app permissions
- **Edge cases / failures:**
  - Empty target user value may prevent proper approval routing depending on configuration expectations
  - Slack interactive/webhook permissions may be missing
  - Human approval delays can stall the workflow if synchronous handling is expected
- **Sub-workflow reference:** None

#### Slack Notification Tool
- **Type / role:** `n8n-nodes-base.slackTool`  
  AI-invokable Slack messaging tool used by the approval process or governance messaging.
- **Configuration choices:**
  - Message text is supplied by AI
  - Channel ID is also supplied dynamically by AI through `$fromAI`
  - OAuth2 authentication
- **Key expressions / variables:**
  - Text: `={{ $fromAI('notification_message', 'The compliance notification or alert message to send') }}`
  - Channel: `={{ $fromAI('channel', 'The Slack channel to send the notification to') }}`
- **Input / output connections:**
  - AI tool available to **Approval Request Tool**
- **Version-specific requirements:**
  - Type version `2.4`
- **Edge cases / failures:**
  - AI-generated channel IDs may be invalid; Slack generally expects a real channel ID, not a name unless properly resolved
  - Permission errors if bot is not in the destination channel
- **Sub-workflow reference:** None

#### Approval History Tool
- **Type / role:** `n8n-nodes-base.dataTableTool`  
  AI-invokable lookup against approval history.
- **Configuration choices:**
  - Operation: `rowExists`
  - Table ID is a placeholder
  - Tool description indicates use for referencing past governance decisions and approval patterns
- **Key expressions / variables:** None visible
- **Input / output connections:**
  - AI tool available to **Privacy Governance Agent**
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases / failures:**
  - Placeholder table ID must be replaced
  - `rowExists` is a narrow operation; it may not return rich historical detail despite the description implying broader retrieval
  - Filter/query logic is not shown, so practical usefulness may be limited until refined
- **Sub-workflow reference:** None

---

## Block 5 — Risk-Based Routing and Alerting

### Overview
This block routes the structured governance result according to risk level. Critical and high-risk findings trigger dedicated Slack alerts, while medium and low risk proceed directly to record creation.

### Nodes Involved
- Route by Risk Level
- Prepare Critical Alert
- Send Critical Alert
- Prepare High Risk Alert
- Send High Risk Alert

### Node Details

#### Route by Risk Level
- **Type / role:** `n8n-nodes-base.switch`  
  Branches execution based on `riskLevel`.
- **Configuration choices:**
  - Four string equality conditions:
    - `CRITICAL`
    - `HIGH`
    - `MEDIUM`
    - `LOW`
- **Key expressions / variables:**
  - `={{ $json.riskLevel }}`
- **Input / output connections:**
  - Input from **Privacy Governance Agent**
  - Outputs:
    - CRITICAL → **Prepare Critical Alert**
    - HIGH → **Prepare High Risk Alert**
    - MEDIUM → **Prepare Audit Record**
    - LOW → **Prepare Audit Record**
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - Any value outside exact uppercase matches will not route as expected
  - Lowercase or descriptive outputs like “High risk” will miss branches unless normalized first
- **Sub-workflow reference:** None

#### Prepare Critical Alert
- **Type / role:** `n8n-nodes-base.set`  
  Formats critical-alert data for Slack.
- **Configuration choices:**
  - Adds:
    - `alertType = CRITICAL PRIVACY RISK`
    - `priority = URGENT`
    - `message = 'CRITICAL: ' + $json.output.findings`
    - `violations = $json.output.regulationViolations`
    - `recommendations = $json.output.recommendations`
    - `timestamp = $now.toISO()`
  - Includes other fields
- **Key expressions / variables:**
  - Uses `$json.output.*`
- **Input / output connections:**
  - Input from **Route by Risk Level**
  - Output to **Send Critical Alert**
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - This node expects data under `$json.output.*`, but later nodes often expect top-level fields like `$json.findings`
  - If the agent output is top-level structured JSON rather than wrapped in `output`, these expressions will fail or produce empty values
- **Sub-workflow reference:** None

#### Send Critical Alert
- **Type / role:** `n8n-nodes-base.slack`  
  Sends an urgent Slack message for critical findings.
- **Configuration choices:**
  - OAuth2 Slack authentication
  - Sends to placeholder `critical_alerts_channel`
  - Message includes violations, recommendations, and timestamp
- **Key expressions / variables:**
  - Slack message uses:
    - `$json.message`
    - `JSON.stringify($json.violations)`
    - `JSON.stringify($json.recommendations)`
    - `$json.timestamp`
- **Input / output connections:**
  - Input from **Prepare Critical Alert**
  - Output to **Prepare Audit Record**
- **Version-specific requirements:**
  - Type version `2.4`
- **Edge cases / failures:**
  - Channel placeholder must be replaced
  - Bot must have posting rights
  - Formatting can be hard to read because arrays are stringified inline
- **Sub-workflow reference:** None

#### Prepare High Risk Alert
- **Type / role:** `n8n-nodes-base.set`  
  Formats high-risk alert data for Slack.
- **Configuration choices:**
  - Adds:
    - `alertType = HIGH PRIVACY RISK`
    - `priority = HIGH`
    - `message = 'High Risk Detected: ' + $json.output.findings`
    - `violations = $json.output.regulationViolations`
    - `recommendations = $json.output.recommendations`
    - `timestamp = $now.toISO()`
- **Key expressions / variables:**
  - Uses `$json.output.*`
- **Input / output connections:**
  - Input from **Route by Risk Level**
  - Output to **Send High Risk Alert**
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - Same structural mismatch risk as critical branch due to `$json.output.*`
- **Sub-workflow reference:** None

#### Send High Risk Alert
- **Type / role:** `n8n-nodes-base.slack`  
  Sends Slack alerts for high-risk findings.
- **Configuration choices:**
  - OAuth2 Slack authentication
  - Sends to placeholder `privacy_alerts_channel`
  - Message includes findings, violations, recommendations, timestamp
- **Key expressions / variables:**
  - Uses formatted fields from previous node
- **Input / output connections:**
  - Input from **Prepare High Risk Alert**
  - Output to **Prepare Audit Record**
- **Version-specific requirements:**
  - Type version `2.4`
- **Edge cases / failures:**
  - Channel placeholder must be replaced
  - Permission/auth errors possible
- **Sub-workflow reference:** None

---

## Block 6 — Audit Record Creation, Storage, and Report Distribution

### Overview
This final block normalizes the result, stores it in a compliance data table, creates a human-readable report, and emails it to the designated stakeholder.

### Nodes Involved
- Prepare Audit Record
- Store Compliance Record
- Prepare Compliance Report
- Send Compliance Report

### Node Details

#### Prepare Audit Record
- **Type / role:** `n8n-nodes-base.set`  
  Normalizes governance output into an audit/compliance record.
- **Configuration choices:**
  - Creates:
    - `eventId` from webhook event or fallback `AUDIT_<timestamp>`
    - `timestamp`
    - `complianceStatus`
    - `riskLevel`
    - `findings`
    - stringified `violations`
    - stringified `recommendations`
    - `approvalRequired`
    - stringified `auditTrail`
- **Key expressions / variables:**
  - `={{ $('Data Usage Event Trigger').item.json.eventId || 'AUDIT_' + $now.toMillis() }}`
  - Top-level fields such as `$json.complianceStatus`, `$json.riskLevel`, `$json.findings`
- **Input / output connections:**
  - Inputs from:
    - **Route by Risk Level** for MEDIUM and LOW
    - **Send Critical Alert**
    - **Send High Risk Alert**
  - Output to **Store Compliance Record**
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - If execution started from the schedule trigger, direct reference to `$('Data Usage Event Trigger').item.json.eventId` may fail depending on n8n execution context and branch availability
  - It creates stringified `violations` and `recommendations`, but downstream node references top-level arrays again rather than these newly prepared strings
- **Sub-workflow reference:** None

#### Store Compliance Record
- **Type / role:** `n8n-nodes-base.dataTable`  
  Stores final compliance records in an n8n Data Table.
- **Configuration choices:**
  - Mapping mode: define fields manually
  - Fields stored:
    - `eventId`
    - `findings`
    - `riskLevel`
    - `timestamp`
    - `auditTrail`
    - `violations`
    - `recommendations`
    - `approvalRequired`
    - `complianceStatus`
  - Uses placeholder table ID for compliance records
- **Key expressions / variables:**
  - Recomputes several values directly from upstream JSON
  - Again references `$('Data Usage Event Trigger').item.json.eventId || 'AUDIT_' + $now.toMillis()`
- **Input / output connections:**
  - Input from **Prepare Audit Record**
  - Output to **Prepare Compliance Report**
- **Version-specific requirements:**
  - Type version `1.1`
  - Requires Data Tables support
- **Edge cases / failures:**
  - Placeholder table ID must be replaced
  - Field references like `$json.regulationViolations` may be missing after **Prepare Audit Record**, because that node renamed/stringified some content under different field names
  - Scheduled executions may have the same source-node reference issue as above
- **Sub-workflow reference:** None

#### Prepare Compliance Report
- **Type / role:** `n8n-nodes-base.set`  
  Builds an email-friendly compliance report payload.
- **Configuration choices:**
  - Creates:
    - dated `reportTitle`
    - `executiveSummary`
    - `detailedFindings`
    - pretty-printed `regulationViolations`
    - pretty-printed `recommendations`
    - pretty-printed `auditTrail`
    - `generatedAt`
    - `approvalStatus` derived from `approvalRequired`
- **Key expressions / variables:**
  - Uses top-level fields such as `$json.complianceStatus`, `$json.riskLevel`, `$json.findings`
  - `approvalStatus = $json.approvalRequired ? 'PENDING APPROVAL' : 'AUTO-APPROVED'`
- **Input / output connections:**
  - Input from **Store Compliance Record**
  - Output to **Send Compliance Report**
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases / failures:**
  - If prior node did not preserve `regulationViolations` or `recommendations` as arrays, pretty-printing may output `undefined`
  - The report is text-only, so formatting is basic
- **Sub-workflow reference:** None

#### Send Compliance Report
- **Type / role:** `n8n-nodes-base.gmail`  
  Sends the final compliance report by email.
- **Configuration choices:**
  - Recipient is a placeholder DPO email
  - Subject is the generated report title
  - Email body concatenates summary, findings, violations, recommendations, audit trail, generation time, approval status
- **Key expressions / variables:**
  - `sendTo: <__PLACEHOLDER_VALUE__dpo_email__>`
  - `subject = $json.reportTitle`
  - `message` from report fields
- **Input / output connections:**
  - Input from **Prepare Compliance Report**
  - No downstream node
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail OAuth2 credentials
- **Edge cases / failures:**
  - Placeholder email must be replaced
  - Gmail OAuth scopes and sender permissions must be valid
  - Long report bodies may be hard to read in plain text
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Data Usage Event Trigger | Webhook | Receives live privacy/data usage events by HTTP POST |  | Privacy Governance Agent | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Scheduled Compliance Audit | Schedule Trigger | Launches periodic compliance audits |  | Privacy Governance Agent | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Privacy Governance Agent | LangChain Agent | Central AI orchestrator for compliance, risk, approval, and reporting decisions | Data Usage Event Trigger; Scheduled Compliance Audit; Governance Agent Model (AI model); Compliance Output Parser (AI parser); Data Privacy Agent Tool; Risk Detection Agent Tool; Audit Log Tool; Approval Request Tool; Approval History Tool | Route by Risk Level | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Governance Agent Model | OpenAI Chat Model | LLM backend for the governance agent |  | Privacy Governance Agent | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Compliance Output Parser | Structured Output Parser | Enforces structured final JSON from the governance agent |  | Privacy Governance Agent |  |
| Data Privacy Agent Tool | LangChain Agent Tool | Specialist privacy-law assessment tool | Privacy Agent Model (AI model); Legal Database API Tool | Privacy Governance Agent | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Privacy Agent Model | OpenAI Chat Model | LLM backend for privacy-law analysis |  | Data Privacy Agent Tool | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Legal Database API Tool | HTTP Request Tool | Queries external legal/regulatory data sources |  | Data Privacy Agent Tool | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Risk Detection Agent Tool | LangChain Agent Tool | Specialist privacy/security risk assessment tool | Risk Agent Model (AI model) | Privacy Governance Agent | ## Risk Detection Agent & Audit Log Tool<br>**Why** — Classifies each event by risk level and logs it<br> immediately, creating an unbroken audit trail before any routing decision is made. |
| Risk Agent Model | OpenAI Chat Model | LLM backend for risk scoring |  | Risk Detection Agent Tool | ## Risk Detection Agent & Audit Log Tool<br>**Why** — Classifies each event by risk level and logs it<br> immediately, creating an unbroken audit trail before any routing decision is made. |
| Audit Log Tool | Data Table Tool | AI tool for writing audit events/logs |  | Privacy Governance Agent | ## Risk Detection Agent & Audit Log Tool<br>**Why** — Classifies each event by risk level and logs it<br> immediately, creating an unbroken audit trail before any routing decision is made. |
| Approval Request Tool | Slack Human-in-the-Loop Tool | Sends approval requests for human governance decisions | Slack Notification Tool | Privacy Governance Agent | ## Approval Request & Slack Notification Tool<br>**Why** — Routes privacy approval requests via Slack, enabling rapid human-in-the-loop governance without email bottlenecks. |
| Slack Notification Tool | Slack Tool | AI-driven Slack notifications used by the approval flow |  | Approval Request Tool | ## Approval Request & Slack Notification Tool<br>**Why** — Routes privacy approval requests via Slack, enabling rapid human-in-the-loop governance without email bottlenecks. |
| Approval History Tool | Data Table Tool | Looks up prior approval history |  | Privacy Governance Agent | ## Approval Request & Slack Notification Tool<br>**Why** — Routes privacy approval requests via Slack, enabling rapid human-in-the-loop governance without email bottlenecks. |
| Route by Risk Level | Switch | Routes cases by risk severity | Privacy Governance Agent | Prepare Critical Alert; Prepare High Risk Alert; Prepare Audit Record; Prepare Audit Record | ## Route by Risk Level<br>**Why** — Rules-based routing separates critical and high-risk events for proportionate, parallel alert handling without manual triage. |
| Prepare Critical Alert | Set | Formats payload for critical Slack alert | Route by Risk Level | Send Critical Alert | ## Route by Risk Level<br>**Why** — Rules-based routing separates critical and high-risk events for proportionate, parallel alert handling without manual triage. |
| Send Critical Alert | Slack | Posts critical privacy alerts to Slack | Prepare Critical Alert | Prepare Audit Record | ## Route by Risk Level<br>**Why** — Rules-based routing separates critical and high-risk events for proportionate, parallel alert handling without manual triage. |
| Prepare High Risk Alert | Set | Formats payload for high-risk Slack alert | Route by Risk Level | Send High Risk Alert | ## Route by Risk Level<br>**Why** — Rules-based routing separates critical and high-risk events for proportionate, parallel alert handling without manual triage. |
| Send High Risk Alert | Slack | Posts high-risk privacy alerts to Slack | Prepare High Risk Alert | Prepare Audit Record | ## Route by Risk Level<br>**Why** — Rules-based routing separates critical and high-risk events for proportionate, parallel alert handling without manual triage. |
| Prepare Audit Record | Set | Normalizes the result into an audit/compliance record | Route by Risk Level; Send Critical Alert; Send High Risk Alert | Store Compliance Record | ## Audit Record, Compliance Storage & Report Distribution<br>**Why** — Every event generates an audit record stored in Google Sheets; a compliance report is prepared and distributed via Gmail to maintain a complete, stakeholder-ready governance trail. |
| Store Compliance Record | Data Table | Stores compliance records in a table | Prepare Audit Record | Prepare Compliance Report | ## Audit Record, Compliance Storage & Report Distribution<br>**Why** — Every event generates an audit record stored in Google Sheets; a compliance report is prepared and distributed via Gmail to maintain a complete, stakeholder-ready governance trail. |
| Prepare Compliance Report | Set | Builds report content for email distribution | Store Compliance Record | Send Compliance Report | ## Audit Record, Compliance Storage & Report Distribution<br>**Why** — Every event generates an audit record stored in Google Sheets; a compliance report is prepared and distributed via Gmail to maintain a complete, stakeholder-ready governance trail. |
| Send Compliance Report | Gmail | Emails the final privacy compliance report | Prepare Compliance Report |  | ## Audit Record, Compliance Storage & Report Distribution<br>**Why** — Every event generates an audit record stored in Google Sheets; a compliance report is prepared and distributed via Gmail to maintain a complete, stakeholder-ready governance trail. |
| Sticky Note | Sticky Note | Documentation/comment block |  |  | ## How It Works<br>This workflow automates data privacy compliance governance for privacy officers, legal operations teams, and data protection leads. It eliminates the manual effort of monitoring data usage events, classifying privacy risks, routing approval requests, and generating audit-ready compliance reports. Data usage events arrive via a webhook trigger while a scheduled audit runs in parallel, ensuring continuous and periodic coverage. Both feeds pass to the Privacy Governance Agent, backed by a governance model and shared memory — which coordinates three specialist tools: a Data Privacy Agent Tool (privacy policy assessment using a privacy model and Legal Database API), a Risk Detection Agent Tool (risk classification using a dedicated risk model), and an Audit Log Tool. Approval requests are routed via an Approval Request Tool with Slack notifications, and outputs are structured via a Compliance Output Parser and Approval History Tool. Results are routed by risk level, critical alerts trigger Slack notifications immediately, high-risk alerts follow a parallel Slack path, before all cases converge to prepare an audit record, store a compliance record in Google Sheets, prepare a compliance report, and distribute it via Gmail. |
| Sticky Note1 | Sticky Note | Documentation/comment block |  |  | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Slack workspace with bot credentials<br>- Gmail account with OAuth credentials<br>- Google Sheets with compliance and audit tabs pre-created<br>## Use Cases<br>- Privacy officers automating GDPR and PDPA data usage event monitoring and risk classification<br>## Customisation<br>- Swap the Legal Database API to target jurisdiction-specific frameworks (GDPR, CCPA, PDPA, HIPAA)<br>## Benefits<br>- Dual-trigger ingestion ensures continuous and scheduled privacy coverage with no monitoring gaps |
| Sticky Note2 | Sticky Note | Documentation/comment block |  |  | ## Setup Steps<br>1. Import workflow; configure the Data Usage Event Trigger webhook URL and Scheduled Compliance Audit interval.<br>2. Add AI model credentials to the Privacy Governance Agent, Data Privacy Agent Tool, and Risk Detection Agent Tool.<br>3. Connect the Legal Database API Tool with your privacy regulatory database endpoint and credentials.<br>4. Link Slack credentials to the Slack Notification Tool, Send Critical Alert, and Send High Risk Alert nodes.<br>5. Link Gmail credentials to the Send Compliance Report node.<br>6. Connect Google Sheets credentials; set sheet IDs for Compliance Record and Audit Log tabs. |
| Sticky Note3 | Sticky Note | Documentation/comment block |  |  | ## Approval Request & Slack Notification Tool<br>**Why** — Routes privacy approval requests via Slack, enabling rapid human-in-the-loop governance without email bottlenecks. |
| Sticky Note4 | Sticky Note | Documentation/comment block |  |  | ## Risk Detection Agent & Audit Log Tool<br>**Why** — Classifies each event by risk level and logs it<br> immediately, creating an unbroken audit trail before any routing decision is made. |
| Sticky Note5 | Sticky Note | Documentation/comment block |  |  | ## Data Privacy Agent & Legal Database API Tool<br>**Why** — Evaluates data usage against legal privacy frameworks, ensuring policy assessments are grounded in current regulatory references. |
| Sticky Note6 | Sticky Note | Documentation/comment block |  |  | ## Route by Risk Level<br>**Why** — Rules-based routing separates critical and high-risk events for proportionate, parallel alert handling without manual triage. |
| Sticky Note7 | Sticky Note | Documentation/comment block |  |  | ## Audit Record, Compliance Storage & Report Distribution<br>**Why** — Every event generates an audit record stored in Google Sheets; a compliance report is prepared and distributed via Gmail to maintain a complete, stakeholder-ready governance trail. |

---

# 4. Reproducing the Workflow from Scratch

Below is a manual rebuild sequence for n8n.

## Prerequisites
1. Prepare credentials:
   - **OpenAI API** credential with access to `gpt-4o`
   - **Slack OAuth2** credential with bot permissions for posting messages and interactive approval flows
   - **Gmail OAuth2** credential for sending email
2. Prepare n8n **Data Tables**:
   - one for audit logs
   - one for compliance records
   - one for approval history
3. Decide target Slack channels:
   - critical alerts channel
   - privacy/high-risk alerts channel
4. Decide recipient email:
   - DPO or privacy team mailbox
5. If using a legal API, prepare:
   - base URL or full endpoints
   - API auth method if required

## Build Steps

1. **Create a Webhook node**
   - Name: **Data Usage Event Trigger**
   - Type: **Webhook**
   - HTTP Method: `POST`
   - Path: `privacy-event`

2. **Create a Schedule Trigger node**
   - Name: **Scheduled Compliance Audit**
   - Type: **Schedule Trigger**
   - Configure it to run daily at hour `9`
   - Optional improvement: add a Set node after it to define `eventType = scheduled_audit` so the governance prompt works exactly as intended

3. **Create the main AI Agent node**
   - Name: **Privacy Governance Agent**
   - Type: **AI Agent / LangChain Agent**
   - Prompt text:
     - If `eventType` is `scheduled_audit`, instruct the model to perform a comprehensive privacy compliance audit
     - Otherwise instruct it to evaluate the incoming data usage event serialized as JSON
   - Add the system message describing the governance role:
     - analyze inputs
     - delegate privacy-law evaluation
     - delegate risk assessment
     - determine governance approvals
     - request human approval where needed
     - generate structured compliance output
   - Enable **structured output parser**

4. **Create an OpenAI Chat Model node**
   - Name: **Governance Agent Model**
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach OpenAI credential
   - Connect it to **Privacy Governance Agent** as the language model

5. **Create a Structured Output Parser node**
   - Name: **Compliance Output Parser**
   - Schema type: Manual
   - Create this JSON schema conceptually:
     - `complianceStatus`: string
     - `riskLevel`: string
     - `regulationViolations`: array of strings
     - `approvalRequired`: boolean
     - `findings`: string
     - `recommendations`: array of strings
     - `auditTrail`: object
   - Connect it to **Privacy Governance Agent** as the output parser

6. **Create the privacy-law specialist tool**
   - Name: **Data Privacy Agent Tool**
   - Type: **AI Agent Tool**
   - Tool input text should come from AI using a named variable like `privacy_evaluation_task`
   - Add a system message describing expertise in GDPR, PDPA, CCPA, consent, minimization, retention, subject rights, cross-border transfer, breach obligations
   - Add a clear tool description so the main agent knows when to use it

7. **Create the privacy-law model**
   - Name: **Privacy Agent Model**
   - Type: **OpenAI Chat Model**
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Connect it to **Data Privacy Agent Tool**

8. **Create the legal API tool**
   - Name: **Legal Database API Tool**
   - Type: **HTTP Request Tool**
   - Configure the URL as AI-supplied if you want dynamic endpoints
   - Better practical option: configure a fixed base URL or fixed endpoint patterns to reduce invalid calls
   - Add any needed authentication headers or credentials
   - Connect it as a tool to **Data Privacy Agent Tool**

9. **Connect Data Privacy Agent Tool to the main governance agent**
   - As an **AI tool** connection

10. **Create the risk specialist tool**
    - Name: **Risk Detection Agent Tool**
    - Type: **AI Agent Tool**
    - Tool input text should come from AI using a named variable like `risk_assessment_task`
    - Add system instructions for breach risk, sensitive data exposure, unauthorized access, excessive data collection, retention gaps, third-party risks, and scoring LOW/MEDIUM/HIGH/CRITICAL
    - Add the tool description

11. **Create the risk model**
    - Name: **Risk Agent Model**
    - Type: **OpenAI Chat Model**
    - Model: `gpt-4o`
    - Temperature: `0.15`
    - Connect it to **Risk Detection Agent Tool**

12. **Connect Risk Detection Agent Tool to the main governance agent**
    - As an **AI tool** connection

13. **Create the audit logging tool**
    - Name: **Audit Log Tool**
    - Type: **Data Table Tool**
    - Select the audit log data table
    - Use column mapping that can accept the expected tool payload
    - Add a description indicating it records evaluations and governance decisions
    - Connect it as an **AI tool** to **Privacy Governance Agent**

14. **Create the approval history lookup tool**
    - Name: **Approval History Tool**
    - Type: **Data Table Tool**
    - Operation: `rowExists` as in the source workflow, or improve it to a richer lookup operation if supported
    - Select the approval history table
    - Connect it as an **AI tool** to **Privacy Governance Agent**

15. **Create the human approval Slack tool**
    - Name: **Approval Request Tool**
    - Type: **Slack Human-in-the-Loop Tool**
    - Attach Slack OAuth2 credential
    - Configure the target approver user or approval destination
    - Connect it as an **AI tool** to **Privacy Governance Agent**

16. **Create the Slack notification helper tool**
    - Name: **Slack Notification Tool**
    - Type: **Slack Tool**
    - Attach Slack OAuth2 credential
    - Configure message text to come from AI
    - Configure channel selection
    - If possible, prefer a fixed approved list of channel IDs rather than fully free AI-generated values
    - Connect it as an **AI tool** to **Approval Request Tool**

17. **Connect entry nodes to the main governance agent**
    - **Data Usage Event Trigger** → **Privacy Governance Agent**
    - **Scheduled Compliance Audit** → **Privacy Governance Agent**
    - Optional best practice: insert a Set node after the schedule trigger with:
      - `eventType = scheduled_audit`
      - `eventId = AUDIT_<timestamp>`

18. **Create a Switch node**
    - Name: **Route by Risk Level**
    - Type: **Switch**
    - Add four string-equals rules on `riskLevel`:
      - `CRITICAL`
      - `HIGH`
      - `MEDIUM`
      - `LOW`
    - Connect **Privacy Governance Agent** → **Route by Risk Level**

19. **Create a Set node for critical alerts**
    - Name: **Prepare Critical Alert**
    - Add fields:
      - `alertType = CRITICAL PRIVACY RISK`
      - `priority = URGENT`
      - `message = 'CRITICAL: ' + findings`
      - `violations = regulationViolations`
      - `recommendations = recommendations`
      - `timestamp = now`
    - Important improvement: reference top-level agent output fields directly unless your agent output is actually wrapped inside `output`

20. **Create the critical Slack alert node**
    - Name: **Send Critical Alert**
    - Type: **Slack**
    - Attach Slack OAuth2 credential
    - Choose the critical alerts channel
    - Build a message including message, violations, recommendations, timestamp
    - Connect:
      - **Route by Risk Level** CRITICAL → **Prepare Critical Alert**
      - **Prepare Critical Alert** → **Send Critical Alert**

21. **Create a Set node for high-risk alerts**
    - Name: **Prepare High Risk Alert**
    - Add fields:
      - `alertType = HIGH PRIVACY RISK`
      - `priority = HIGH`
      - `message = 'High Risk Detected: ' + findings`
      - `violations = regulationViolations`
      - `recommendations = recommendations`
      - `timestamp = now`
    - Again, prefer top-level field references unless wrapper output exists

22. **Create the high-risk Slack alert node**
    - Name: **Send High Risk Alert**
    - Type: **Slack**
    - Attach Slack OAuth2 credential
    - Choose the high-risk/privacy alerts channel
    - Connect:
      - **Route by Risk Level** HIGH → **Prepare High Risk Alert**
      - **Prepare High Risk Alert** → **Send High Risk Alert**

23. **Create the audit record formatter**
    - Name: **Prepare Audit Record**
    - Type: **Set**
    - Add fields:
      - `eventId`
      - `timestamp`
      - `complianceStatus`
      - `riskLevel`
      - `findings`
      - `violations` as JSON string
      - `recommendations` as JSON string
      - `approvalRequired`
      - `auditTrail` as JSON string
    - Recommended improvement:
      - avoid direct dependency on the webhook node for `eventId`
      - use current item data with fallback generation, especially for scheduled runs

24. **Connect all branches into Prepare Audit Record**
    - MEDIUM output of switch → **Prepare Audit Record**
    - LOW output of switch → **Prepare Audit Record**
    - **Send Critical Alert** → **Prepare Audit Record**
    - **Send High Risk Alert** → **Prepare Audit Record**

25. **Create the compliance storage node**
    - Name: **Store Compliance Record**
    - Type: **Data Table**
    - Select the compliance records table
    - Map fields:
      - eventId
      - findings
      - riskLevel
      - timestamp
      - auditTrail
      - violations
      - recommendations
      - approvalRequired
      - complianceStatus
    - Connect **Prepare Audit Record** → **Store Compliance Record**

26. **Create the report preparation node**
    - Name: **Prepare Compliance Report**
    - Type: **Set**
    - Build fields:
      - `reportTitle`
      - `executiveSummary`
      - `detailedFindings`
      - `regulationViolations`
      - `recommendations`
      - `auditTrail`
      - `generatedAt`
      - `approvalStatus`
    - Connect **Store Compliance Record** → **Prepare Compliance Report**

27. **Create the Gmail report node**
    - Name: **Send Compliance Report**
    - Type: **Gmail**
    - Attach Gmail OAuth2 credential
    - Set recipient to DPO/privacy mailbox
    - Subject from `reportTitle`
    - Body from the report fields
    - Connect **Prepare Compliance Report** → **Send Compliance Report**

28. **Replace all placeholders**
    - Audit log table ID
    - Compliance records table ID
    - Approval history table ID
    - Critical Slack channel ID
    - High-risk Slack channel ID
    - DPO email address

29. **Test webhook path**
    - Send a sample POST payload such as:
      - `eventId`
      - `eventType`
      - `dataCategory`
      - `processingPurpose`
      - `storageLocation`
      - `sharedWith`
      - `retentionPolicy`
      - `containsPII`
    - Verify structured output includes:
      - compliance status
      - risk level
      - violations
      - recommendations

30. **Test scheduled path**
    - Execute the schedule branch manually
    - Confirm it gets an `eventType` or otherwise update the workflow so the audit prompt branch is used

## Recommended Corrections While Rebuilding
To make the workflow more robust than the source JSON:

1. **Normalize agent output immediately**
   - If the agent returns top-level fields, do not use `$json.output.findings`
   - Use one consistent structure everywhere

2. **Add a normalization Set node after the agent**
   - Force `riskLevel` to uppercase
   - Ensure arrays exist even when empty
   - Guarantee `approvalRequired` is boolean

3. **Fix scheduled audit payload**
   - Add `eventType = scheduled_audit`
   - Add generated `eventId`

4. **Use Data Table terminology consistently**
   - The sticky notes mention Google Sheets, but the actual workflow uses **n8n Data Tables**, not Google Sheets nodes

5. **Constrain Slack destination selection**
   - Avoid free AI choice of arbitrary channel IDs unless you trust the model and have validation

6. **Improve approval history retrieval**
   - Replace `rowExists` with a richer lookup if historical context is important

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates privacy compliance governance for privacy officers, legal operations teams, and data protection leads. It reduces manual work for monitoring events, classifying risk, routing approvals, and generating audit-ready reports. | Workflow note |
| Both real-time webhook events and scheduled audits feed the same governance pipeline, providing continuous and periodic coverage. | Workflow design note |
| The governance layer coordinates specialist tools for privacy-law analysis, risk detection, audit logging, approval handling, and structured output generation. | Architecture note |
| Prerequisites listed in the workflow: OpenAI API key or compatible LLM, Slack bot credentials, Gmail OAuth credentials, and pre-created storage tables/sheets. | Setup note |
| Example use case listed in the workflow: privacy officers automating GDPR and PDPA monitoring and risk classification. | Use case |
| Suggested customization in the workflow: adapt the Legal Database API to jurisdiction-specific frameworks such as GDPR, CCPA, PDPA, and HIPAA. | Customization note |
| Important implementation mismatch: the notes refer to Google Sheets, but the actual workflow stores records using n8n Data Table and Data Table Tool nodes. | Important correction |
| Important implementation risk: scheduled audits do not explicitly inject `eventType = scheduled_audit`, even though the agent prompt expects it. | Important correction |
| Important implementation risk: alert-preparation nodes reference `$json.output.*`, which may not match the structured parser output shape if fields are returned at the top level. | Important correction |