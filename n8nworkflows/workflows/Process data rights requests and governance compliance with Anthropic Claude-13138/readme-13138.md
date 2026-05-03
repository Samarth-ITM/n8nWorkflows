Process data rights requests and governance compliance with Anthropic Claude

https://n8nworkflows.xyz/workflows/process-data-rights-requests-and-governance-compliance-with-anthropic-claude-13138


# Process data rights requests and governance compliance with Anthropic Claude

## 1. Workflow Overview

**Workflow name:** AI-Powered Data Rights and Governance Compliance Orchestration System  
**Stated purpose (title provided):** Process data rights requests and governance compliance with Anthropic Claude

This workflow ingests data subject rights requests (GDPR/CCPA-style) via webhook, performs AI-assisted validation (consent + retention + rights legitimacy), routes requests based on validation outcomes, and then orchestrates governance actions: fulfillment API execution, regulatory report generation/sending, or compliance escalation to Slack—while writing audit logs to n8n Data Tables.

### 1.1 Input Reception & Runtime Configuration
Receives an inbound POST request and injects environment-specific configuration (API endpoints, Slack channel, reporting email) into the execution context.

### 1.2 AI Validation (Rights Validation Agent + Tools)
Uses an Anthropic Claude chat model-driven **agent** to validate the request. The agent calls two AI “tool” nodes:
- Consent lookup (consent status + legal basis)
- Retention policy check (retention status + delete eligibility)

### 1.3 Validation Routing & Logging
Routes by validation status:
- VALID → proceed to governance decisioning
- INVALID / REQUIRES_REVIEW → log validation results (and stops; no escalation path is implemented for invalid beyond logging)

### 1.4 Governance Decisioning (Governance Agent + Regulatory Reporting Tool)
A second Anthropic Claude agent decides the action type:
- FULFILLMENT → call fulfillment API
- REGULATORY_REPORT → email report
- ESCALATION → Slack alert

### 1.5 Action Execution, Merge, and Multi-Stream Audit Logging
Executes the chosen branch, merges branch results, logs governance actions, and separately logs compliance escalations (Slack branch).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Workflow Configuration

**Overview:** Accepts inbound requests and enriches them with runtime configuration values required downstream (endpoints, channels, emails).  
**Nodes involved:**  
- Data Rights Request Webhook  
- Workflow Configuration

#### Node: Data Rights Request Webhook
- **Type / role:** `Webhook` trigger; entrypoint for data rights requests.
- **Configuration choices:**
  - **HTTP Method:** POST
  - **Path:** `data-rights-request`
  - **Response mode:** “Respond via Response Node” (`responseMode: responseNode`)
    - **Important:** There is **no** explicit “Respond to Webhook” node in this workflow. This commonly leads to webhook calls hanging until timeout.
- **Key variables/expressions:** none.
- **Connections:**
  - **Output →** Workflow Configuration
- **Edge cases / failures:**
  - Client may time out because no response is sent.
  - Payload shape is not validated before use; missing fields may break AI expectations later.
- **Version notes:** TypeVersion 2.1.

#### Node: Workflow Configuration
- **Type / role:** `Set` node; injects config constants/placeholders into JSON.
- **Configuration choices:**
  - Adds fields:
    - `fulfillmentApiUrl`
    - `complianceSlackChannel`
    - `regulatoryEmail`
    - `consentDatabaseUrl` (note: not used by any HTTP node in this workflow)
    - `retentionPolicyUrl` (note: not used by any HTTP node in this workflow)
  - **Include other fields:** true (keeps webhook payload).
- **Key variables/expressions:**
  - Placeholder strings like `<__PLACEHOLDER_VALUE__...__>`
- **Connections:**
  - **Input ←** Data Rights Request Webhook
  - **Output →** Rights Validation Agent
- **Edge cases / failures:**
  - If placeholders are not replaced, downstream HTTP/Slack/Email steps will fail (invalid URL, invalid channel, missing email).
- **Version notes:** TypeVersion 3.4.

---

### Block 2 — AI Validation (Rights Validation Agent + Consent/Retention Tools)

**Overview:** An Anthropic-powered agent validates the request by calling two tool nodes that return structured consent and retention results; output is forced into a schema using a structured output parser.  
**Nodes involved:**  
- Rights Validation Agent  
- Anthropic Model - Rights Agent  
- Consent Lookup Tool  
- Anthropic Model - Consent Tool  
- Consent Tool Output Parser  
- Retention Policy Tool  
- Anthropic Model - Retention Tool  
- Retention Tool Output Parser  
- Rights Validation Output Parser

#### Node: Rights Validation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`; LLM agent that orchestrates validation and calls tools.
- **Configuration choices:**
  - **Input text:** `{{ JSON.stringify($json) }}` (feeds entire current item, including config + request payload).
  - **System message:** Defines responsibilities (GDPR/CCPA validation, minimization, violations) and instructs tool usage order.
  - **Has output parser:** true (uses “Rights Validation Output Parser” via `ai_outputParser` connection).
- **Key variables/expressions:**
  - `JSON.stringify($json)` may become large; can cause token bloat.
- **Connections:**
  - **Language model (ai_languageModel) ←** Anthropic Model - Rights Agent
  - **Tools (ai_tool) ←** Consent Lookup Tool, Retention Policy Tool
  - **Output parser (ai_outputParser) ←** Rights Validation Output Parser
  - **Main output →** Route by Validation Status
- **Edge cases / failures:**
  - Tool calls are purely LLM-driven; if the model does not call tools, validation may be weak.
  - If the model returns non-JSON or mismatched schema, the structured parser will error.
  - Very large inbound payload may exceed model context limits.
- **Version notes:** TypeVersion 3.1.

#### Node: Anthropic Model - Rights Agent
- **Type / role:** `lmChatAnthropic`; provides Claude model to the Rights Validation Agent.
- **Configuration choices:**
  - **Model:** `claude-sonnet-4-5-20250929` (as selected in node).
  - Uses Anthropic API credentials (“Anthropic account”).
- **Connections:**
  - **ai_languageModel →** Rights Validation Agent
- **Edge cases / failures:**
  - Credential missing/invalid → auth errors.
  - Model name availability depends on Anthropic account/region and n8n node version.
- **Version notes:** TypeVersion 1.3.

#### Node: Consent Lookup Tool
- **Type / role:** `agentTool`; tool callable by the agent to “retrieve consent”.
- **Configuration choices:**
  - **Tool input:** `dataSubjectId` via `$fromAI("dataSubjectId", ...)`
  - **System message:** Instructs to “Query the consent database…”, return structured consent fields.
  - **Important:** No actual HTTP/database node is connected. This tool is itself an LLM-driven step, not a real lookup.
  - **Has output parser:** true (Consent Tool Output Parser).
- **Connections:**
  - **ai_languageModel ←** Anthropic Model - Consent Tool
  - **ai_outputParser ←** Consent Tool Output Parser
  - **ai_tool →** Rights Validation Agent
- **Edge cases / failures:**
  - Since it’s not backed by real data access, results are synthetic unless you replace with real integrations.
  - Output parser failures if required fields are missing.
- **Version notes:** TypeVersion 3.

#### Node: Anthropic Model - Consent Tool
- **Type / role:** `lmChatAnthropic`; model provider for Consent Lookup Tool.
- **Configuration:** same model selection as above.
- **Connections:** **ai_languageModel →** Consent Lookup Tool
- **Failures:** auth/model availability.
- **Version notes:** TypeVersion 1.3.

#### Node: Consent Tool Output Parser
- **Type / role:** `outputParserStructured`; forces the tool output into a defined JSON schema.
- **Configuration choices (schema highlights):**
  - Required: `consentStatus`, `legalBasis`
  - Optional: `consentDate`, `expirationDate`, `purposes`, `violations`
- **Connections:** **ai_outputParser →** Consent Lookup Tool
- **Failure modes:**
  - LLM output not matching schema → parser error.
- **Version notes:** TypeVersion 1.3.

#### Node: Retention Policy Tool
- **Type / role:** `agentTool`; tool callable by agent to “check retention”.
- **Configuration choices:**
  - Tool input `dataSubjectId` via `$fromAI(...)`
  - System message instructs retention checks and structured output.
  - Not backed by real retention policy API in this workflow.
  - Has structured output parser.
- **Connections:**
  - **ai_languageModel ←** Anthropic Model - Retention Tool
  - **ai_outputParser ←** Retention Tool Output Parser
  - **ai_tool →** Rights Validation Agent
- **Edge cases / failures:** same as consent tool (synthetic results unless replaced).
- **Version notes:** TypeVersion 3.

#### Node: Anthropic Model - Retention Tool
- **Type / role:** `lmChatAnthropic`; model provider for Retention Policy Tool.
- **Connections:** **ai_languageModel →** Retention Policy Tool
- **Version notes:** TypeVersion 1.3.

#### Node: Retention Tool Output Parser
- **Type / role:** structured schema enforcement for retention tool output.
- **Configuration choices (schema highlights):**
  - Required: `retentionStatus`, `shouldDelete`
  - Optional: categories, dates, violations
- **Connections:** **ai_outputParser →** Retention Policy Tool
- **Failures:** schema mismatch.
- **Version notes:** TypeVersion 1.3.

#### Node: Rights Validation Output Parser
- **Type / role:** structured schema enforcement for Rights Validation Agent final output.
- **Configuration choices (schema highlights):**
  - Required: `validationStatus`, `dataSubjectId`, `requestType`, `reasoning`
  - Optional: `consentStatus`, `retentionStatus`, `complianceViolations`, `dataMinimizationRequired`
- **Connections:** **ai_outputParser →** Rights Validation Agent
- **Failures:** schema mismatch will block routing.
- **Version notes:** TypeVersion 1.3.

---

### Block 3 — Validation Routing & Validation Logging

**Overview:** Routes the validated output into either governance decisioning (VALID) or validation logging (INVALID/REQUIRES_REVIEW).  
**Nodes involved:**  
- Route by Validation Status  
- Log Validation Results

#### Node: Route by Validation Status
- **Type / role:** `Switch`; branches based on `$json.output.validationStatus`.
- **Configuration choices:**
  - Output “Valid” if `output.validationStatus == "VALID"`
  - Output “Invalid” if `output.validationStatus == "INVALID"` OR `"REQUIRES_REVIEW"`
  - Fallback output renamed to “Unhandled”
- **Key expressions:**
  - `{{ $json.output.validationStatus }}`
- **Connections:**
  - **Input ←** Rights Validation Agent
  - **Valid →** Governance Agent
  - **Invalid →** Log Validation Results
- **Edge cases / failures:**
  - If agent output doesn’t include `output.validationStatus` (parser failure or different structure), switch may route to “Unhandled” with no downstream connection (execution ends).
- **Version notes:** TypeVersion 3.4.

#### Node: Log Validation Results
- **Type / role:** `Data Table`; writes validation outcomes to a table.
- **Configuration choices:**
  - Data Table name: `validation_results`
- **Connections:**
  - **Input ←** Route by Validation Status (Invalid path)
- **Edge cases / failures:**
  - Data Table not existing / permission issues.
  - The node logs the whole incoming item structure as per Data Table defaults; ensure columns/shape compatibility in your n8n Data Table configuration.
- **Version notes:** TypeVersion 1.1.

---

### Block 4 — Governance Decisioning (Agent + Reporting Tool)

**Overview:** For validated requests, the governance agent decides whether to fulfill, report, or escalate. Regulatory reporting can be generated via a dedicated tool node with structured output.  
**Nodes involved:**  
- Governance Agent  
- Anthropic Model - Governance Agent  
- Regulatory Reporting Tool  
- Anthropic Model - Reporting Tool  
- Reporting Tool Output Parser  
- Governance Output Parser  
- Route by Action Type

#### Node: Governance Agent
- **Type / role:** `agent`; determines actionType and prepares data for execution.
- **Configuration choices:**
  - Input text: `{{ JSON.stringify($json) }}`
  - System message enforces:
    - choose `FULFILLMENT | REGULATORY_REPORT | ESCALATION`
    - use Regulatory Reporting Tool if reporting required
    - apply minimization
  - Uses structured output parser (Governance Output Parser).
- **Connections:**
  - **Input ←** Route by Validation Status (Valid)
  - **ai_languageModel ←** Anthropic Model - Governance Agent
  - **ai_tool ←** Regulatory Reporting Tool
  - **ai_outputParser ←** Governance Output Parser
  - **Main output →** Route by Action Type
- **Edge cases / failures:**
  - Parser mismatch stops flow.
  - Prepared fields like `fulfillmentActions` may be missing → downstream API body still sends `actions: undefined`.
- **Version notes:** TypeVersion 3.1.

#### Node: Anthropic Model - Governance Agent
- **Type / role:** `lmChatAnthropic`; model provider for governance agent.
- **Model:** `claude-sonnet-4-5-20250929`
- **Connections:** **ai_languageModel →** Governance Agent
- **Failures:** auth/model availability.
- **Version notes:** TypeVersion 1.3.

#### Node: Regulatory Reporting Tool
- **Type / role:** `agentTool`; generates structured regulatory report content from `reportData`.
- **Configuration choices:**
  - Input: `$fromAI("reportData", ..., "json")`
  - System message instructs report format and fields (reportId, reportType, audit trail, etc.)
  - Not connected to a real template store; LLM-generated report output.
  - Has output parser: Reporting Tool Output Parser.
- **Connections:**
  - **ai_languageModel ←** Anthropic Model - Reporting Tool
  - **ai_outputParser ←** Reporting Tool Output Parser
  - **ai_tool →** Governance Agent
- **Edge cases / failures:**
  - Schema mismatch in report output.
  - If governance agent never calls this tool, no report fields will exist beyond governance output.
- **Version notes:** TypeVersion 3.

#### Node: Anthropic Model - Reporting Tool
- **Type / role:** `lmChatAnthropic`; model provider for reporting tool.
- **Connections:** **ai_languageModel →** Regulatory Reporting Tool
- **Version notes:** TypeVersion 1.3.

#### Node: Reporting Tool Output Parser
- **Type / role:** structured schema enforcement for regulatory report.
- **Configuration choices (schema highlights):**
  - Required: `reportId`, `reportType`, `complianceStatus`
  - Optional: subject, requestDate, actionsTaken, minimization, auditTrail
- **Connections:** **ai_outputParser →** Regulatory Reporting Tool
- **Failures:** schema mismatch.
- **Version notes:** TypeVersion 1.3.

#### Node: Governance Output Parser
- **Type / role:** structured schema enforcement for governance decision.
- **Configuration choices (schema highlights):**
  - Required: `actionType`, `dataSubjectId`, `reasoning`
  - Optional: `fulfillmentActions`, `reportRequired`, `escalationReason`, `dataMinimizationApplied`, `auditLog`
- **Connections:** **ai_outputParser →** Governance Agent
- **Failures:** schema mismatch.
- **Version notes:** TypeVersion 1.3.

#### Node: Route by Action Type
- **Type / role:** `Switch`; routes to one of three action branches based on `$json.output.actionType`.
- **Configuration choices:**
  - “Fulfillment” if equals `FULFILLMENT`
  - “Regulatory Report” if equals `REGULATORY_REPORT`
  - “Escalation” if equals `ESCALATION`
  - Fallback “Unhandled”
- **Connections:**
  - **Input ←** Governance Agent
  - **Fulfillment →** Execute Fulfillment API
  - **Regulatory Report →** Send Regulatory Report
  - **Escalation →** Notify Compliance Team
- **Edge cases / failures:**
  - Missing/unknown actionType ends in “Unhandled” with no connected node.
- **Version notes:** TypeVersion 3.4.

---

### Block 5 — Action Execution, Branch Merge, and Audit Logging

**Overview:** Executes the chosen governance action, merges branch outputs, logs governance actions to a Data Table, and logs escalations separately.  
**Nodes involved:**  
- Execute Fulfillment API  
- Send Regulatory Report  
- Notify Compliance Team  
- Merge Action Branches  
- Log Governance Actions  
- Log Compliance Escalations

#### Node: Execute Fulfillment API
- **Type / role:** `HTTP Request`; calls an external fulfillment system.
- **Configuration choices:**
  - URL from config: `{{ $('Workflow Configuration').first().json.fulfillmentApiUrl }}`
  - Method: POST
  - JSON body:
    - `dataSubjectId: $json.output.dataSubjectId`
    - `actions: $json.output.fulfillmentActions`
    - `timestamp: $now.toISO()`
    - `dataMinimizationApplied: $json.output.dataMinimizationApplied`
- **Connections:**
  - **Input ←** Route by Action Type (Fulfillment)
  - **Output →** Merge Action Branches (input 0)
- **Edge cases / failures:**
  - Placeholder URL or unreachable host → request failure.
  - If `fulfillmentActions` is undefined/non-array, downstream system may reject.
  - No auth headers configured; if API requires auth, it will fail.
- **Version notes:** TypeVersion 4.3.

#### Node: Send Regulatory Report
- **Type / role:** `Email Send`; emails the regulatory report.
- **Configuration choices:**
  - To: `{{ $('Workflow Configuration').first().json.regulatoryEmail }}`
  - From: placeholder `<__PLACEHOLDER_VALUE__Sender email address for regulatory reports__>`
  - Subject: `Regulatory Compliance Report - {{ $json.output.dataSubjectId }}`
  - HTML template references both report fields (e.g., `$json.reportId`) and governance output (e.g., `$json.output.fulfillmentActions`)
- **Connections:**
  - **Input ←** Route by Action Type (Regulatory Report)
  - **Output →** Merge Action Branches (input 1)
- **Edge cases / failures:**
  - If the inbound item does not actually include `reportId/reportType` at top-level (likely, unless mapped), email shows “N/A”.
  - Email node requires configured SMTP/Sendmail credentials depending on n8n setup.
  - `fromEmail` placeholder not replaced → send failure.
- **Version notes:** TypeVersion 2.1.

#### Node: Notify Compliance Team
- **Type / role:** `Slack`; posts escalation alert.
- **Configuration choices:**
  - OAuth2 authentication (Slack credential “Slack account”)
  - Channel ID from config: `{{ $('Workflow Configuration').first().json.complianceSlackChannel }}`
  - Message body includes:
    - `output.dataSubjectId`, `output.requestType`, `output.escalationReason`
    - `output.complianceViolations` joined
    - `output.auditLog` mapped to bullet list
- **Connections:**
  - **Input ←** Route by Action Type (Escalation)
  - **Outputs →**
    - Merge Action Branches (input 2)
    - Log Compliance Escalations
- **Edge cases / failures:**
  - Channel ID placeholder not replaced or bot not in channel → Slack API error.
  - If `output.auditLog` is not an array, `.map()` expression may error.
- **Version notes:** TypeVersion 2.4.

#### Node: Merge Action Branches
- **Type / role:** `Merge`; combines results from the three possible action branches.
- **Configuration choices:**
  - Mode: combine
  - Combine by: position
  - Inputs: 3
- **Connections:**
  - **Inputs ←** Execute Fulfillment API (0), Send Regulatory Report (1), Notify Compliance Team (2)
  - **Output →** Log Governance Actions
- **Edge cases / failures:**
  - “Combine by position” expects aligned item positions; if branches produce different item counts, results can be confusing or empty.
  - If only one branch runs (typical), merge behavior should still produce one combined item, but validate in your n8n version.
- **Version notes:** TypeVersion 3.2.

#### Node: Log Governance Actions
- **Type / role:** `Data Table`; writes combined action/audit data.
- **Configuration choices:**
  - Data Table name: `governance_actions`
- **Connections:**
  - **Input ←** Merge Action Branches
- **Edge cases / failures:** table missing/permissions; schema mismatch.
- **Version notes:** TypeVersion 1.1.

#### Node: Log Compliance Escalations
- **Type / role:** `Data Table`; logs escalations specifically.
- **Configuration choices:**
  - Data Table name: `compliance_escalations`
- **Connections:**
  - **Input ←** Notify Compliance Team
- **Edge cases / failures:** table missing/permissions.
- **Version notes:** TypeVersion 1.1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Data Rights Request Webhook | Webhook | Inbound request trigger | — | Workflow Configuration | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Workflow Configuration | Set | Inject runtime configuration | Data Rights Request Webhook | Rights Validation Agent | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Rights Validation Agent | LangChain Agent | Validate rights request, call tools | Workflow Configuration | Route by Validation Status | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Consent Lookup Tool | Agent Tool | Produce consent/legal basis info | (AI tool call from Rights Validation Agent) | Rights Validation Agent (via ai_tool) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Retention Policy Tool | Agent Tool | Produce retention/expiry info | (AI tool call from Rights Validation Agent) | Rights Validation Agent (via ai_tool) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Regulatory Reporting Tool | Agent Tool | Generate regulatory report payload | (AI tool call from Governance Agent) | Governance Agent (via ai_tool) | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Anthropic Model - Rights Agent | Anthropic Chat Model | LLM provider for Rights agent | — | Rights Validation Agent (ai_languageModel) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Anthropic Model - Governance Agent | Anthropic Chat Model | LLM provider for Governance agent | — | Governance Agent (ai_languageModel) | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Anthropic Model - Consent Tool | Anthropic Chat Model | LLM provider for Consent tool | — | Consent Lookup Tool (ai_languageModel) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Anthropic Model - Retention Tool | Anthropic Chat Model | LLM provider for Retention tool | — | Retention Policy Tool (ai_languageModel) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Anthropic Model - Reporting Tool | Anthropic Chat Model | LLM provider for Reporting tool | — | Regulatory Reporting Tool (ai_languageModel) | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Rights Validation Output Parser | Structured Output Parser | Enforce schema for validation output | — | Rights Validation Agent (ai_outputParser) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Governance Output Parser | Structured Output Parser | Enforce schema for governance output | — | Governance Agent (ai_outputParser) | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Consent Tool Output Parser | Structured Output Parser | Enforce schema for consent tool | — | Consent Lookup Tool (ai_outputParser) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Retention Tool Output Parser | Structured Output Parser | Enforce schema for retention tool | — | Retention Policy Tool (ai_outputParser) | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |
| Reporting Tool Output Parser | Structured Output Parser | Enforce schema for report tool | — | Regulatory Reporting Tool (ai_outputParser) | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Route by Validation Status | Switch | Branch on validationStatus | Rights Validation Agent | Governance Agent; Log Validation Results | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Log Validation Results | Data Table | Persist invalid/review validation results | Route by Validation Status (Invalid) | — |  |
| Governance Agent | LangChain Agent | Decide actionType; orchestrate reporting/escalation/fulfillment | Route by Validation Status (Valid) | Route by Action Type | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Route by Action Type | Switch | Branch on actionType | Governance Agent | Execute Fulfillment API; Send Regulatory Report; Notify Compliance Team | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Execute Fulfillment API | HTTP Request | Call external fulfillment endpoint | Route by Action Type (Fulfillment) | Merge Action Branches | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Send Regulatory Report | Email Send | Email regulatory report | Route by Action Type (Regulatory Report) | Merge Action Branches | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Notify Compliance Team | Slack | Send escalation alert | Route by Action Type (Escalation) | Merge Action Branches; Log Compliance Escalations | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Merge Action Branches | Merge | Combine branch outputs | Execute Fulfillment API; Send Regulatory Report; Notify Compliance Team | Log Governance Actions | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Log Governance Actions | Data Table | Persist governance actions | Merge Action Branches | — | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Log Compliance Escalations | Data Table | Persist escalation events | Notify Compliance Team | — | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Sticky Note | Sticky Note | Workspace note | — | — | ## Prerequisites\nOpenAI/Nvidia/Anthropic API credentials for AI validation models\n## Use Cases\nFinancial institutions ensuring transaction compliance monitoring, \n## Customization\nAdjust AI model parameters for industry-specific compliance rules\n## Benefits\nReduces compliance review time by 80%, eliminates manual validation errors |
| Sticky Note1 | Sticky Note | Workspace note | — | — | ## Setup Steps\n1. Configure Data Ingestion Webhook trigger endpoint\n2. Connect Workflow Execution Configuration node with validation parameters\n3. Set up Fetch Validation Rules node with OpenAI/Nvidia API credentials for AI model access\n4. Configure parallel AI model nodes with respective API credentials\n5. Connect Route by Validation Status node with branching logic\n6. Set up Governance Documentation node with document template configurations\n7. Configure parallel action nodes |
| Sticky Note2 | Sticky Note | Workspace note | — | — | ## How It Works\nThis workflow automates comprehensive data validation and regulatory compliance reporting through intelligent AI-driven analysis. Designed for compliance officers, data governance teams, and regulatory affairs departments, it solves the critical challenge of ensuring data quality while generating audit-ready compliance documentation across multiple regulatory frameworks.The system receives data through webhook triggers, performs multi-layered validation using AI models to detect anomalies and policy violations, and intelligently routes findings based on validation outcomes. It orchestrates parallel processing streams for content lookup, retention policy enforcement, and rejection handling. The workflow merges validation results, generates governance documentation, and manages compliance notifications through multiple channels. By automating action routing based on compliance status and maintaining detailed audit logs across validation, governance, and action streams, it ensures regulatory adherence while eliminating manual review bottlenecks. |
| Sticky Note3 | Sticky Note | Workspace note | — | — | ## Multi-Stream Logging\n**What:** Maintains synchronized audit trails across validation results, governance documentation, and compliance actions\n**Why:** Creates complete regulatory audit trail required for compliance verification and regulatory inquiries |
| Sticky Note5 | Sticky Note | Workspace note | — | — | ## Governance Routing\n**What:** Routes validated data based on compliance status to appropriate documentation and reporting streams\n**Why:** Ensures compliant data receives proper governance documentation while violations trigger regulatory reporting workflows |
| Sticky Note6 | Sticky Note | Workspace note | — | — | ## AI Validation\n**What:** Processes incoming data through parallel AI models for content analysis, retention compliance, and rejection detection\n**Why:** Ensures comprehensive validation coverage by leveraging specialized AI models for different compliance dimensions |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *AI-Powered Data Rights and Governance Compliance Orchestration System* (or your preferred name).
- Keep it inactive until credentials and endpoints are configured.

2) **Add trigger: Webhook**
- Node: **Webhook**
- Name: **Data Rights Request Webhook**
- Method: **POST**
- Path: `data-rights-request`
- Response: **Respond using “Respond to Webhook” node**
  - Note: To match the current JSON exactly, set “Respond via Response Node”, but then you must add a Respond node (the provided workflow does not include it).

3) **Add configuration node: Set**
- Node: **Set**
- Name: **Workflow Configuration**
- Enable “Include Other Fields”
- Add string fields:
  - `fulfillmentApiUrl` (your fulfillment service endpoint)
  - `complianceSlackChannel` (Slack channel ID)
  - `regulatoryEmail` (recipient email for reports)
  - `consentDatabaseUrl` (optional; not used unless you add real lookup)
  - `retentionPolicyUrl` (optional; not used unless you add real lookup)
- Connect: **Webhook → Workflow Configuration**

4) **Add Anthropic model nodes (5x)**
Create 5 nodes of type **Anthropic Chat Model** (lmChatAnthropic):
- **Anthropic Model - Rights Agent**
- **Anthropic Model - Governance Agent**
- **Anthropic Model - Consent Tool**
- **Anthropic Model - Retention Tool**
- **Anthropic Model - Reporting Tool**
For each:
- Select model: `claude-sonnet-4-5-20250929` (or closest available)
- Set **Anthropic API credentials**.

5) **Add structured output parsers (4x)**
Add nodes of type **Structured Output Parser**:
- **Consent Tool Output Parser** (requires `consentStatus`, `legalBasis`)
- **Retention Tool Output Parser** (requires `retentionStatus`, `shouldDelete`)
- **Rights Validation Output Parser** (requires `validationStatus`, `dataSubjectId`, `requestType`, `reasoning`)
- **Reporting Tool Output Parser** (requires `reportId`, `reportType`, `complianceStatus`)
- **Governance Output Parser** (requires `actionType`, `dataSubjectId`, `reasoning`)
Use the schemas as described in the workflow (manual JSON schema entry).

6) **Add tool nodes (3x)**
Add nodes of type **Agent Tool**:
- **Consent Lookup Tool**
  - Input: `$fromAI("dataSubjectId", "...", "string")`
  - Attach **Anthropic Model - Consent Tool** as language model
  - Attach **Consent Tool Output Parser**
- **Retention Policy Tool**
  - Input: `$fromAI("dataSubjectId", "...", "string")`
  - Attach **Anthropic Model - Retention Tool**
  - Attach **Retention Tool Output Parser**
- **Regulatory Reporting Tool**
  - Input: `$fromAI("reportData", "...", "json")`
  - Attach **Anthropic Model - Reporting Tool**
  - Attach **Reporting Tool Output Parser**

7) **Add agent: Rights Validation Agent**
- Node: **LangChain Agent**
- Name: **Rights Validation Agent**
- Text: `{{ JSON.stringify($json) }}`
- System message: paste your rights validation instructions (as in the workflow)
- Attach **Anthropic Model - Rights Agent** as language model
- Attach tools: **Consent Lookup Tool**, **Retention Policy Tool**
- Attach output parser: **Rights Validation Output Parser**
- Connect: **Workflow Configuration → Rights Validation Agent**

8) **Add switch: Route by Validation Status**
- Node: **Switch**
- Name: **Route by Validation Status**
- Rule 1 (“Valid”): `{{ $json.output.validationStatus }}` equals `VALID`
- Rule 2 (“Invalid”): equals `INVALID` OR `REQUIRES_REVIEW`
- Connect: **Rights Validation Agent → Route by Validation Status**

9) **Add Data Table logging for invalid results**
- Node: **Data Table**
- Name: **Log Validation Results**
- Table: `validation_results` (create it in n8n if needed)
- Connect: **Route by Validation Status (Invalid) → Log Validation Results**

10) **Add agent: Governance Agent**
- Node: **LangChain Agent**
- Name: **Governance Agent**
- Text: `{{ JSON.stringify($json) }}`
- System message: paste governance orchestration instructions (as in the workflow)
- Attach **Anthropic Model - Governance Agent**
- Attach tool: **Regulatory Reporting Tool**
- Attach output parser: **Governance Output Parser**
- Connect: **Route by Validation Status (Valid) → Governance Agent**

11) **Add switch: Route by Action Type**
- Node: **Switch**
- Name: **Route by Action Type**
- Rules based on `{{ $json.output.actionType }}`:
  - `FULFILLMENT` → “Fulfillment”
  - `REGULATORY_REPORT` → “Regulatory Report”
  - `ESCALATION` → “Escalation”
- Connect: **Governance Agent → Route by Action Type**

12) **Add action branch A: Fulfillment HTTP**
- Node: **HTTP Request**
- Name: **Execute Fulfillment API**
- Method: POST
- URL: `{{ $('Workflow Configuration').first().json.fulfillmentApiUrl }}`
- Body: JSON with:
  - `dataSubjectId`, `actions`, `timestamp`, `dataMinimizationApplied`
- Connect: **Route by Action Type (Fulfillment) → Execute Fulfillment API**

13) **Add action branch B: Regulatory email**
- Node: **Email Send**
- Name: **Send Regulatory Report**
- To: `{{ $('Workflow Configuration').first().json.regulatoryEmail }}`
- From: your sender address (must be valid for your mail setup)
- Subject: `Regulatory Compliance Report - {{ $json.output.dataSubjectId }}`
- HTML: build from report/governance fields (as in the workflow)
- Configure email transport (SMTP, etc.) per your n8n environment.
- Connect: **Route by Action Type (Regulatory Report) → Send Regulatory Report**

14) **Add action branch C: Slack escalation**
- Node: **Slack**
- Name: **Notify Compliance Team**
- Auth: OAuth2 Slack credential
- Channel ID: `{{ $('Workflow Configuration').first().json.complianceSlackChannel }}`
- Text: compose from `$json.output.*` (as in the workflow)
- Connect: **Route by Action Type (Escalation) → Notify Compliance Team**

15) **Add escalation logging**
- Node: **Data Table**
- Name: **Log Compliance Escalations**
- Table: `compliance_escalations`
- Connect: **Notify Compliance Team → Log Compliance Escalations**

16) **Merge branches**
- Node: **Merge**
- Name: **Merge Action Branches**
- Mode: **Combine**
- Inputs: **3**
- Combine by: **Position**
- Connect:
  - Execute Fulfillment API → Merge (Input 0)
  - Send Regulatory Report → Merge (Input 1)
  - Notify Compliance Team → Merge (Input 2)

17) **Log governance actions**
- Node: **Data Table**
- Name: **Log Governance Actions**
- Table: `governance_actions`
- Connect: **Merge Action Branches → Log Governance Actions**

18) **(Recommended) Add a Webhook Response**
- Add node: **Respond to Webhook**
- Connect it so every path returns a response (e.g., after logging / after merge).
- Return at least: status + correlation id + timestamps.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Prerequisites: OpenAI/Nvidia/Anthropic API credentials for AI validation models | Sticky note “Prerequisites” |
| ## Use Cases: Financial institutions ensuring transaction compliance monitoring | Sticky note “Prerequisites” |
| ## Customization: Adjust AI model parameters for industry-specific compliance rules | Sticky note “Prerequisites” |
| ## Benefits: Reduces compliance review time by 80%, eliminates manual validation errors | Sticky note “Prerequisites” |
| Setup steps listed (webhook, config, model credentials, branching, documentation/action nodes) | Sticky note “Setup Steps” |
| High-level description of automated validation, routing, reporting, and audit logs | Sticky note “How It Works” |
| Multi-Stream Logging rationale: synchronized audit trails for regulatory inquiries | Sticky note “Multi-Stream Logging” |
| Governance Routing rationale: route compliant vs violations to appropriate workflows | Sticky note “Governance Routing” |
| AI Validation rationale: specialized models for different compliance dimensions | Sticky note “AI Validation” |