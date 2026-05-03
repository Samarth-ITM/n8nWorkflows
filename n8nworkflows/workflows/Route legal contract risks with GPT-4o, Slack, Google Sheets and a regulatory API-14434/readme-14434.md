Route legal contract risks with GPT-4o, Slack, Google Sheets and a regulatory API

https://n8nworkflows.xyz/workflows/route-legal-contract-risks-with-gpt-4o--slack--google-sheets-and-a-regulatory-api-14434


# Route legal contract risks with GPT-4o, Slack, Google Sheets and a regulatory API

# 1. Workflow Overview

This workflow automates legal contract intake, AI-based review, compliance validation, risk-based routing, and audit logging. A contract is submitted through a webhook, its text is extracted, and a governance agent orchestrates specialized AI tools to assess legal risk, validate regulatory compliance, notify stakeholders through Slack, and generate structured output for downstream processing.

It is designed for legal teams, contract managers, procurement teams, and risk/compliance functions that need to triage contracts consistently and maintain an auditable trail.

## 1.1 Contract Intake and Text Extraction
The workflow begins with an HTTP webhook that receives a contract file. The uploaded file is processed by a file extraction node to produce plain text for AI analysis.

## 1.2 AI Governance Orchestration
A central Legal Governance Agent uses GPT-4o, shared memory, specialist legal/compliance agent tools, a regulatory API tool, a Slack tool, and a structured output parser. This block produces a normalized JSON-style legal review result including risk level, obligations, compliance status, approvals, and recommended actions.

## 1.3 Risk-Based Routing and Alert Preparation
The structured output is routed by `riskLevel` using a Switch node. Depending on whether the contract is critical, high, or medium/other, the workflow adds alert metadata such as priority, action requirements, and target notification channel.

## 1.4 Audit Logging and Clause-Level Persistence
The routed contract result is transformed into an audit record, saved to a data table, and split into individual risk clauses. Each clause is normalized into a record and stored separately for downstream review and reporting.

## 1.5 Approval Tracking and Final Summary
The workflow checks whether approval is required. If yes, it creates and stores an approval log; otherwise, it skips directly to a final summary. In both cases, the workflow ends with a consolidated record indicating completion and audit readiness.

---

# 2. Block-by-Block Analysis

## 2.1 Contract Intake and Text Extraction

### Overview
This block receives an uploaded contract and converts it into text. It is the entry point for the entire review lifecycle.

### Nodes Involved
- Contract Upload Webhook
- Extract Contract Text

### Node Details

#### Contract Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook` — trigger node that starts the workflow on HTTP POST.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `contract-review`
  - No extra webhook options are configured.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - No input; this is a trigger.
  - Outputs to **Extract Contract Text**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Incorrect webhook URL/path usage
  - Wrong HTTP method
  - Missing uploaded file/binary payload
  - Payload format incompatible with extraction node
- **Sub-workflow reference:** None.

#### Extract Contract Text
- **Type and technical role:** `n8n-nodes-base.extractFromFile` — extracts text from an uploaded file.
- **Configuration choices:**  
  - Operation: `text`
  - No custom extraction options configured.
- **Key expressions or variables used:** Uses incoming binary file data from webhook payload.
- **Input and output connections:**  
  - Input from **Contract Upload Webhook**
  - Output to **Legal Governance Agent**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Unsupported file format
  - Empty or corrupt file
  - OCR is not configured, so scanned PDFs/images may not produce usable text
  - Large files may cause performance or timeout issues
- **Sub-workflow reference:** None.

---

## 2.2 AI Governance Orchestration

### Overview
This is the core intelligence block. It uses a central governance agent that delegates analysis to specialist tools for contract review and compliance validation, optionally sends Slack alerts, and forces a structured machine-readable result.

### Nodes Involved
- Legal Governance Agent
- Governance Model
- Contract Review Memory
- Structured Output Parser
- Contract Review Agent
- Contract Review Model
- Compliance Validation Agent
- Compliance Model
- Regulatory Database Tool
- Slack Alert Tool

### Node Details

#### Legal Governance Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — main orchestrator agent.
- **Configuration choices:**  
  - Prompt type: defined in node
  - User text input: `={{ $json.text }}`
  - System message instructs the agent to:
    - delegate contract analysis to the Contract Review Agent
    - validate compliance through the Compliance Validation Agent
    - determine risk and approvals
    - generate legal summary and audit notes
    - produce structured output
  - Output parser enabled
- **Key expressions or variables used:**  
  - `{{$json.text}}` for extracted contract text
- **Input and output connections:**  
  - Main input from **Extract Contract Text**
  - Main output to **Route by Risk Level**
  - AI language model from **Governance Model**
  - AI memory from **Contract Review Memory**
  - AI tools:
    - **Contract Review Agent**
    - **Compliance Validation Agent**
    - **Slack Alert Tool**
  - AI output parser from **Structured Output Parser**
- **Version-specific requirements:** Type version `3.1`; requires n8n LangChain/AI nodes support.
- **Edge cases or potential failure types:**  
  - LLM output may fail schema validation
  - Prompt may be too large for model context window with very long contracts
  - Tool invocation may not happen as expected if prompting is insufficient
  - Hallucinated outputs if source text is poor quality
- **Sub-workflow reference:** None.

#### Governance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT chat model for the governance agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output to **Legal Governance Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - OpenAI auth errors
  - Model availability/rate limits
  - Cost spikes for long contract inputs
- **Sub-workflow reference:** None.

#### Contract Review Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — conversational memory attached to the governance agent.
- **Configuration choices:** Default buffer window behavior; no extra parameters configured.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI memory to **Legal Governance Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Limited usefulness in single-shot executions
  - Memory window constraints may truncate context in multi-turn scenarios
- **Sub-workflow reference:** None.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces a JSON schema for the governance agent response.
- **Configuration choices:**  
  - Manual JSON schema with required fields:
    - `contractId`
    - `riskLevel`
    - `riskClauses`
    - `obligations`
    - `complianceStatus`
    - `complianceIssues`
    - `regulatoryRequirements`
    - `approvalRequired`
    - `approvers`
    - `legalSummary`
    - `auditNotes`
    - `recommendedActions`
    - `timestamp`
  - `riskLevel` allowed values: `low`, `medium`, `high`, `critical`
  - `complianceStatus` allowed values: `compliant`, `non-compliant`, `requires-review`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI output parser to **Legal Governance Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Missing required fields from model output
  - Wrong enum values
  - Invalid nested structures in `riskClauses` or `obligations`
- **Sub-workflow reference:** None.

#### Contract Review Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist AI tool for legal contract analysis.
- **Configuration choices:**  
  - Input text: `={{ $fromAI('contract_text', 'The full contract text to analyze') }}`
  - Tool description says it analyzes risk clauses, obligations, liability terms, indemnification, termination, and commitments
  - System message focuses on clause-level legal risk review
- **Key expressions or variables used:**  
  - `$fromAI('contract_text', ...)`
- **Input and output connections:**  
  - AI tool connected to **Legal Governance Agent**
  - AI language model from **Contract Review Model**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Governance agent may pass malformed or incomplete contract text
  - Long-text truncation
  - Legal interpretation inconsistency due to prompt/model behavior
- **Sub-workflow reference:** None.

#### Contract Review Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT model backing the contract review tool.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model to **Contract Review Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - OpenAI credential issues
  - Rate limits
  - Cost and latency with large contracts
- **Sub-workflow reference:** None.

#### Compliance Validation Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool` — specialist AI tool for compliance assessment.
- **Configuration choices:**  
  - Input text: `={{ $fromAI('contract_text', 'The contract text to validate for compliance') }}`
  - System message covers GDPR, SOC2, HIPAA, industry-specific regulations, and governance policies
  - Tool description frames it as a compliance gap/risk validator
- **Key expressions or variables used:**  
  - `$fromAI('contract_text', ...)`
- **Input and output connections:**  
  - AI tool connected to **Legal Governance Agent**
  - AI language model from **Compliance Model**
  - AI tool access to **Regulatory Database Tool**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Regulatory scope may be too broad if jurisdiction is unspecified
  - Incomplete tool results if regulatory API is unavailable
  - Model may infer unsupported regulations if not constrained
- **Sub-workflow reference:** None.

#### Compliance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — GPT model for the compliance validation tool.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model to **Compliance Validation Agent**
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Same OpenAI-related issues as other model nodes
- **Sub-workflow reference:** None.

#### Regulatory Database Tool
- **Type and technical role:** `n8n-nodes-base.httpRequestTool` — AI-callable HTTP tool for external regulatory lookups.
- **Configuration choices:**  
  - URL is a placeholder and must be replaced:
    - `<__PLACEHOLDER_VALUE__regulatory_api_endpoint__>`
  - Tool description explains that it queries regulatory/legal reference systems
- **Key expressions or variables used:** None configured in URL/body yet.
- **Input and output connections:**  
  - AI tool to **Compliance Validation Agent**
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**  
  - Placeholder not replaced
  - Missing auth headers or API credentials
  - Unstructured or incompatible response payload
  - Timeouts or 4xx/5xx API errors
- **Sub-workflow reference:** None.

#### Slack Alert Tool
- **Type and technical role:** `n8n-nodes-base.slackTool` — AI-callable Slack messaging tool.
- **Configuration choices:**  
  - Sends a message to a channel selected by AI
  - Message text: `={{ $fromAI('message', 'Alert message content') }}`
  - Channel mode: by channel ID/value
  - Channel value: `={{ $fromAI('channel', 'Slack channel for legal alerts', 'string', '#legal-alerts') }}`
  - Authentication: OAuth2
  - Tool description explains it is used for high-risk alerts, approval requests, and compliance issues
- **Key expressions or variables used:**  
  - `$fromAI('message', ...)`
  - `$fromAI('channel', ..., '#legal-alerts')`
- **Input and output connections:**  
  - AI tool to **Legal Governance Agent**
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Slack OAuth issues
  - Bot lacks permission to post to chosen channel
  - Channel naming vs channel ID mismatch
  - AI may output a channel string not accepted by Slack configuration
- **Sub-workflow reference:** None.

---

## 2.3 Risk-Based Routing and Alert Preparation

### Overview
This block maps the AI-generated `riskLevel` to operational metadata. It standardizes how each contract should be prioritized and where it should be routed for downstream handling.

### Nodes Involved
- Route by Risk Level
- Prepare Critical Alert
- Prepare High Risk Alert
- Prepare Standard Review

### Node Details

#### Route by Risk Level
- **Type and technical role:** `n8n-nodes-base.switch` — conditional router based on risk level.
- **Configuration choices:**  
  - Output 1: `Critical Risk` when `{{$json.riskLevel}} == "critical"`
  - Output 2: `High Risk` when `{{$json.riskLevel}} == "high"`
  - Output 3: `Medium Risk` when `{{$json.riskLevel}} == "medium"`
  - Fallback output enabled as `extra`
- **Key expressions or variables used:**  
  - `={{ $json.riskLevel }}`
- **Input and output connections:**  
  - Input from **Legal Governance Agent**
  - Outputs to:
    - **Prepare Critical Alert**
    - **Prepare High Risk Alert**
    - **Prepare Standard Review**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - `low` is not explicitly mapped and will go to fallback output
  - In the current connections, fallback is not visibly connected, so low-risk records may not continue
  - Case-sensitive comparison may reject `High`, `CRITICAL`, etc.
- **Sub-workflow reference:** None.

#### Prepare Critical Alert
- **Type and technical role:** `n8n-nodes-base.set` — enriches critical items with urgent handling metadata.
- **Configuration choices:**  
  - Adds:
    - `alertType = "CRITICAL"`
    - `priority = "P0"`
    - `requiresImmediateAction = true`
    - `notificationChannel = "#legal-critical"`
  - Keeps other fields
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**  
  - Input from **Route by Risk Level**
  - Output to **Prepare Audit Record**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Assumes upstream payload already contains valid review fields
- **Sub-workflow reference:** None.

#### Prepare High Risk Alert
- **Type and technical role:** `n8n-nodes-base.set` — enriches high-risk items.
- **Configuration choices:**  
  - Adds:
    - `alertType = "HIGH_RISK"`
    - `priority = "P1"`
    - `requiresImmediateAction = true`
    - `notificationChannel = "#legal-alerts"`
  - Keeps other fields
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**  
  - Input from **Route by Risk Level**
  - Output to **Prepare Audit Record**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Minimal; mostly inherited from upstream payload quality.
- **Sub-workflow reference:** None.

#### Prepare Standard Review
- **Type and technical role:** `n8n-nodes-base.set` — enriches medium/standard cases.
- **Configuration choices:**  
  - Adds:
    - `alertType = "STANDARD_REVIEW"`
    - `priority = "P2"`
    - `requiresImmediateAction = false`
    - `notificationChannel = "#legal-reviews"`
  - Keeps other fields
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**  
  - Input from **Route by Risk Level**
  - Output to **Prepare Audit Record**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Despite the name, this branch only receives the explicitly configured third switch output (`medium`) unless fallback is manually connected
- **Sub-workflow reference:** None.

---

## 2.4 Audit Logging and Clause-Level Persistence

### Overview
This block creates an auditable contract review record and also stores each risk clause independently. It supports compliance tracking, reporting, and downstream remediation.

### Nodes Involved
- Prepare Audit Record
- Track Contract Reviews
- Split Risk Clauses
- Prepare Risk Clause Record
- Store Risk Clauses

### Node Details

#### Prepare Audit Record
- **Type and technical role:** `n8n-nodes-base.set` — maps selected fields into an audit-friendly shape.
- **Configuration choices:** Adds or remaps:
  - `contractId`
  - `riskLevel`
  - `complianceStatus`
  - `approvalRequired`
  - `legalSummary`
  - `auditNotes`
  - `reviewTimestamp` from `timestamp`
  - `alertType`
  - `priority`
- **Key expressions or variables used:**  
  - `={{ $json.contractId }}`
  - `={{ $json.riskLevel }}`
  - `={{ $json.complianceStatus }}`
  - `={{ $json.approvalRequired }}`
  - `={{ $json.legalSummary }}`
  - `={{ $json.auditNotes }}`
  - `={{ $json.timestamp }}`
  - `={{ $json.alertType }}`
  - `={{ $json.priority }}`
- **Input and output connections:**  
  - Input from:
    - **Prepare Critical Alert**
    - **Prepare High Risk Alert**
    - **Prepare Standard Review**
  - Outputs to:
    - **Track Contract Reviews**
    - **Split Risk Clauses**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - `riskClauses` is not explicitly assigned here and `includeOtherFields` is not enabled; depending on Set-node behavior/version, downstream split may receive no `riskClauses`
  - If upstream schema fails, key fields may be empty
- **Sub-workflow reference:** None.

#### Track Contract Reviews
- **Type and technical role:** `n8n-nodes-base.dataTable` — stores the audit record in an n8n Data Table.
- **Configuration choices:**  
  - Column mapping: auto-map input data
  - Data table ID is placeholder:
    - `<__PLACEHOLDER_VALUE__contract_reviews_audit_trail__>`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Prepare Audit Record**
  - Output to **Check Approval Required**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Placeholder table ID not configured
  - Schema mismatches in table columns
  - Data Table feature availability depends on n8n version/plan
- **Sub-workflow reference:** None.

#### Split Risk Clauses
- **Type and technical role:** `n8n-nodes-base.splitOut` — explodes the `riskClauses` array into one item per clause.
- **Configuration choices:**  
  - Field to split out: `riskClauses`
- **Key expressions or variables used:**  
  - `riskClauses`
- **Input and output connections:**  
  - Input from **Prepare Audit Record**
  - Output to **Prepare Risk Clause Record**
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - If `riskClauses` is missing, null, or not an array, splitting will fail or produce zero items
  - This is especially likely because the upstream Set node may not preserve the original field
- **Sub-workflow reference:** None.

#### Prepare Risk Clause Record
- **Type and technical role:** `n8n-nodes-base.set` — normalizes each clause into a standalone storage record.
- **Configuration choices:** Adds:
  - `contractId` from **Prepare Audit Record**
  - `clauseText` from current split item
  - `riskType` from current split item
  - `riskLevel` from **Prepare Audit Record**
  - `identifiedAt` as current timestamp
- **Key expressions or variables used:**  
  - `={{ $('Prepare Audit Record').item.json.contractId }}`
  - `={{ $json.clauseText }}`
  - `={{ $json.riskType }}`
  - `={{ $('Prepare Audit Record').item.json.riskLevel }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input from **Split Risk Clauses**
  - Output to **Store Risk Clauses**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Cross-item reference to `Prepare Audit Record` can become ambiguous if multiple executions/items are processed concurrently
  - Missing `clauseText` or `riskType` from AI output
- **Sub-workflow reference:** None.

#### Store Risk Clauses
- **Type and technical role:** `n8n-nodes-base.dataTable` — stores per-clause risk findings.
- **Configuration choices:**  
  - Column mapping: auto-map input data
  - Data table ID placeholder:
    - `<__PLACEHOLDER_VALUE__risk_clauses_tracking__>`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Prepare Risk Clause Record**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Placeholder table ID not replaced
  - Storage schema mismatch
- **Sub-workflow reference:** None.

---

## 2.5 Approval Tracking and Final Summary

### Overview
This block determines whether approval is necessary and records an approval outcome when required. It then produces a final completion summary.

### Nodes Involved
- Check Approval Required
- Prepare Approval Log
- Log Approval Decision
- Prepare Final Summary

### Node Details

#### Check Approval Required
- **Type and technical role:** `n8n-nodes-base.if` — conditional branching on approval requirement.
- **Configuration choices:**  
  - Checks boolean true for:
    - `={{ $json.output.approvalRequired }}`
- **Key expressions or variables used:**  
  - `={{ $json.output.approvalRequired }}`
- **Input and output connections:**  
  - Input from **Track Contract Reviews**
  - True output to **Prepare Approval Log**
  - False output to **Prepare Final Summary**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Likely expression bug: upstream records appear to use top-level `approvalRequired`, not `output.approvalRequired`
  - Because of this, approvals may always evaluate false unless data is wrapped under `output`
- **Sub-workflow reference:** None.

#### Prepare Approval Log
- **Type and technical role:** `n8n-nodes-base.set` — creates an approval log item.
- **Configuration choices:** Adds:
  - `contractId`
  - `approvalStatus = "APPROVED"`
  - `approver = {{$json.approvers}}`
  - `approvalTimestamp = {{$now.toISO()}}`
  - `riskLevel`
  - `complianceStatus`
- **Key expressions or variables used:**  
  - `={{ $json.contractId }}`
  - `={{ $json.approvers }}`
  - `={{ $now.toISO() }}`
  - `={{ $json.riskLevel }}`
  - `={{ $json.complianceStatus }}`
- **Input and output connections:**  
  - Input from **Check Approval Required**
  - Output to **Log Approval Decision**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - `approvers` may be an array; assigning into a string field may serialize unexpectedly
  - Hardcoded `APPROVED` is operationally unrealistic if actual approval is pending
- **Sub-workflow reference:** None.

#### Log Approval Decision
- **Type and technical role:** `n8n-nodes-base.dataTable` — stores approval records.
- **Configuration choices:**  
  - Auto-maps input data
  - Data table ID placeholder:
    - `<__PLACEHOLDER_VALUE__approval_decisions_log__>`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Prepare Approval Log**
  - Output to **Prepare Final Summary**
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Placeholder table ID not configured
  - Table schema mismatch
- **Sub-workflow reference:** None.

#### Prepare Final Summary
- **Type and technical role:** `n8n-nodes-base.set` — marks the workflow as completed and audit-ready.
- **Configuration choices:**  
  - Adds:
    - `workflowStatus = "COMPLETED"`
    - `processingTimestamp = {{$now.toISO()}}`
    - `auditReady = true`
  - Keeps other fields
- **Key expressions or variables used:**  
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Inputs from:
    - **Check Approval Required** false branch
    - **Log Approval Decision**
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If approval logging fails, final summary will not be produced on the approval-required branch
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Contract Upload Webhook | Webhook | Receives uploaded contract via HTTP POST |  | Extract Contract Text | ## Contract Upload, Extraction & Governance Orchestration<br>**Why** — Webhook ingestion and text extraction feed the Legal Governance Agent, which coordinates Contract Review, Compliance Validation, and Slack alerting in parallel using shared memory for context continuity. |
| Extract Contract Text | Extract From File | Converts uploaded file into text | Contract Upload Webhook | Legal Governance Agent | ## Contract Upload, Extraction & Governance Orchestration<br>**Why** — Webhook ingestion and text extraction feed the Legal Governance Agent, which coordinates Contract Review, Compliance Validation, and Slack alerting in parallel using shared memory for context continuity. |
| Legal Governance Agent | LangChain Agent | Orchestrates legal review, compliance, alerts, and structured output | Extract Contract Text | Route by Risk Level | ## Contract Upload, Extraction & Governance Orchestration<br>**Why** — Webhook ingestion and text extraction feed the Legal Governance Agent, which coordinates Contract Review, Compliance Validation, and Slack alerting in parallel using shared memory for context continuity. |
| Governance Model | OpenAI Chat Model | LLM backing the governance agent |  | Legal Governance Agent | ## Contract Upload, Extraction & Governance Orchestration<br>**Why** — Webhook ingestion and text extraction feed the Legal Governance Agent, which coordinates Contract Review, Compliance Validation, and Slack alerting in parallel using shared memory for context continuity. |
| Contract Review Memory | Buffer Window Memory | Provides conversational context to governance agent |  | Legal Governance Agent | ## Contract Upload, Extraction & Governance Orchestration<br>**Why** — Webhook ingestion and text extraction feed the Legal Governance Agent, which coordinates Contract Review, Compliance Validation, and Slack alerting in parallel using shared memory for context continuity. |
| Structured Output Parser | Structured Output Parser | Enforces schema on final AI response |  | Legal Governance Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Contract Review Agent | Agent Tool | Specialist legal clause/risk analysis tool |  | Legal Governance Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Contract Review Model | OpenAI Chat Model | LLM for contract review tool |  | Contract Review Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Compliance Validation Agent | Agent Tool | Specialist compliance and regulatory validation tool |  | Legal Governance Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Compliance Model | OpenAI Chat Model | LLM for compliance tool |  | Compliance Validation Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Regulatory Database Tool | HTTP Request Tool | External regulatory lookup tool |  | Compliance Validation Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Slack Alert Tool | Slack Tool | Sends legal alerts to Slack from AI tool calls |  | Legal Governance Agent | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Route by Risk Level | Switch | Routes output by risk classification | Legal Governance Agent | Prepare Critical Alert, Prepare High Risk Alert, Prepare Standard Review | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Prepare Critical Alert | Set | Adds critical priority metadata | Route by Risk Level | Prepare Audit Record | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Prepare High Risk Alert | Set | Adds high-risk metadata | Route by Risk Level | Prepare Audit Record | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Prepare Standard Review | Set | Adds standard review metadata | Route by Risk Level | Prepare Audit Record | ## Compliance Validation & Risk Routing<br>**Why** — Cross-references clauses against live regulatory data, then routes contracts by risk level — critical, high, or standard — ensuring proportionate response without manual triage. |
| Prepare Audit Record | Set | Creates top-level audit record | Prepare Critical Alert, Prepare High Risk Alert, Prepare Standard Review | Track Contract Reviews, Split Risk Clauses | ## Audit Record, Approval Tracking & Risk Clause Storage<br>**Why** — Generates auditable records for every contract, checks approval requirements, logs decisions, and stores extracted risk clauses separately for targeted downstream review. |
| Track Contract Reviews | Data Table | Stores contract review audit trail | Prepare Audit Record | Check Approval Required | ## Audit Record, Approval Tracking & Risk Clause Storage<br>**Why** — Generates auditable records for every contract, checks approval requirements, logs decisions, and stores extracted risk clauses separately for targeted downstream review. |
| Split Risk Clauses | Split Out | Splits risk clause array into items | Prepare Audit Record | Prepare Risk Clause Record | ## Audit Record, Approval Tracking & Risk Clause Storage<br>**Why** — Generates auditable records for every contract, checks approval requirements, logs decisions, and stores extracted risk clauses separately for targeted downstream review. |
| Check Approval Required | If | Branches based on approval requirement | Track Contract Reviews | Prepare Approval Log, Prepare Final Summary | ## Audit Record, Approval Tracking & Risk Clause Storage<br>**Why** — Generates auditable records for every contract, checks approval requirements, logs decisions, and stores extracted risk clauses separately for targeted downstream review. |
| Prepare Approval Log | Set | Builds approval decision record | Check Approval Required | Log Approval Decision | ## Audit Record, Approval Tracking & Risk Clause Storage<br>**Why** — Generates auditable records for every contract, checks approval requirements, logs decisions, and stores extracted risk clauses separately for targeted downstream review. |
| Log Approval Decision | Data Table | Stores approval decision record | Prepare Approval Log | Prepare Final Summary | ## Prepare Final Summary<br>**Why** — Consolidates review outcomes, approval status, and risk findings into a single structured summary for stakeholders. |
| Prepare Final Summary | Set | Marks process complete and audit-ready | Check Approval Required, Log Approval Decision |  | ## Prepare Final Summary<br>**Why** — Consolidates review outcomes, approval status, and risk findings into a single structured summary for stakeholders. |
| Prepare Risk Clause Record | Set | Normalizes individual risk clause record | Split Risk Clauses | Store Risk Clauses | ## Audit Record, Approval Tracking & Risk Clause Storage<br>**Why** — Generates auditable records for every contract, checks approval requirements, logs decisions, and stores extracted risk clauses separately for targeted downstream review. |
| Store Risk Clauses | Data Table | Stores clause-level risk findings | Prepare Risk Clause Record |  | ## Prepare Final Summary<br>**Why** — Consolidates review outcomes, approval status, and risk findings into a single structured summary for stakeholders. |
| Sticky Note | Sticky Note | Visual documentation |  |  |  |
| Sticky Note1 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note2 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note3 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note5 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note6 | Sticky Note | Visual documentation |  |  |  |
| Sticky Note7 | Sticky Note | Visual documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence in n8n.

## 4.1 Prerequisites
1. Create or obtain:
   - OpenAI API credentials for GPT-4o
   - Slack OAuth2 credentials for a bot that can post in your target channels
   - A regulatory API endpoint and any required auth
   - Three n8n Data Tables for:
     - contract reviews audit trail
     - approval decisions log
     - risk clauses tracking
2. Ensure your n8n instance supports:
   - LangChain AI nodes
   - Data Table nodes
   - Webhooks
   - File extraction

## 4.2 Create the Intake Nodes
1. Add a **Webhook** node named **Contract Upload Webhook**.
   - Method: `POST`
   - Path: `contract-review`
2. Add an **Extract From File** node named **Extract Contract Text**.
   - Operation: `Text`
3. Connect **Contract Upload Webhook → Extract Contract Text**.

## 4.3 Create the Main Governance AI Stack
4. Add a **LangChain Agent** node named **Legal Governance Agent**.
   - Set input text to: `{{$json.text}}`
   - Use a system message instructing the agent to:
     - delegate legal analysis
     - validate compliance
     - determine risk level and approvals
     - generate legal summary and audit notes
     - return structured output
   - Enable structured output parsing
5. Connect **Extract Contract Text → Legal Governance Agent**.

6. Add an **OpenAI Chat Model** node named **Governance Model**.
   - Model: `gpt-4o`
   - Temperature: `0.2`
   - Attach OpenAI credentials
7. Connect **Governance Model** to the **ai_languageModel** input of **Legal Governance Agent**.

8. Add a **Memory Buffer Window** node named **Contract Review Memory**.
   - Leave defaults unless you want to tune buffer size
9. Connect it to the **ai_memory** input of **Legal Governance Agent**.

10. Add a **Structured Output Parser** node named **Structured Output Parser**.
    - Choose manual schema
    - Define a schema with required fields:
      - `contractId` string
      - `riskLevel` enum: `low`, `medium`, `high`, `critical`
      - `riskClauses` array of objects with `clauseText`, `riskType`
      - `obligations` array of objects with `obligationDescription`, `partyResponsible`
      - `complianceStatus` enum: `compliant`, `non-compliant`, `requires-review`
      - `complianceIssues` array of strings
      - `regulatoryRequirements` array of strings
      - `approvalRequired` boolean
      - `approvers` array of strings
      - `legalSummary` string
      - `auditNotes` string
      - `recommendedActions` array of strings
      - `timestamp` string
11. Connect it to the **ai_outputParser** input of **Legal Governance Agent**.

## 4.4 Create the Specialist Legal Review Tool
12. Add an **Agent Tool** node named **Contract Review Agent**.
    - Set text input to: `{{$fromAI('contract_text', 'The full contract text to analyze')}}`
    - Add a system message focused on:
      - high-risk clauses
      - indemnity
      - liability caps
      - termination
      - IP assignment
      - obligations and ambiguities
    - Add a tool description explaining that this tool analyzes contracts and returns clause-level risks
13. Connect **Contract Review Agent** to the **ai_tool** input of **Legal Governance Agent**.

14. Add an **OpenAI Chat Model** node named **Contract Review Model**.
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Attach the same or another OpenAI credential
15. Connect it to the **ai_languageModel** input of **Contract Review Agent**.

## 4.5 Create the Specialist Compliance Tool
16. Add an **Agent Tool** node named **Compliance Validation Agent**.
    - Set text input to: `{{$fromAI('contract_text', 'The contract text to validate for compliance')}}`
    - Use a system message focused on:
      - GDPR
      - SOC2
      - HIPAA
      - industry-specific regulation
      - governance/policy adherence
    - Add a tool description saying it validates contracts against regulations and compliance policies
17. Connect **Compliance Validation Agent** to the **ai_tool** input of **Legal Governance Agent**.

18. Add an **OpenAI Chat Model** node named **Compliance Model**.
    - Model: `gpt-4o`
    - Temperature: `0.2`
    - Attach OpenAI credentials
19. Connect it to the **ai_languageModel** input of **Compliance Validation Agent**.

20. Add an **HTTP Request Tool** node named **Regulatory Database Tool**.
    - Replace placeholder URL with your real regulatory endpoint
    - Configure auth headers, API key, OAuth2, or query parameters as required by your provider
    - Add a tool description such as: queries external regulatory databases and legal reference systems
21. Connect it to the **ai_tool** input of **Compliance Validation Agent**.

## 4.6 Create the Slack Tool
22. Add a **Slack Tool** node named **Slack Alert Tool**.
    - Authentication: OAuth2
    - Text: `{{$fromAI('message', 'Alert message content')}}`
    - Select by channel
    - Channel field: `{{$fromAI('channel', 'Slack channel for legal alerts', 'string', '#legal-alerts')}}`
    - Add a tool description indicating it is used for high-risk legal alerts and approval requests
23. Attach Slack OAuth2 credentials.
24. Connect it to the **ai_tool** input of **Legal Governance Agent**.

## 4.7 Create Risk Routing
25. Add a **Switch** node named **Route by Risk Level**.
    - Compare `{{$json.riskLevel}}`
    - Rule 1: equals `critical`
    - Rule 2: equals `high`
    - Rule 3: equals `medium`
    - Optional but recommended: add explicit rule 4 for `low`
26. Connect **Legal Governance Agent → Route by Risk Level**.

27. Add a **Set** node named **Prepare Critical Alert**.
    - Keep other fields: enabled
    - Add:
      - `alertType = CRITICAL`
      - `priority = P0`
      - `requiresImmediateAction = true`
      - `notificationChannel = #legal-critical`
28. Connect the critical output of the switch to it.

29. Add a **Set** node named **Prepare High Risk Alert**.
    - Keep other fields: enabled
    - Add:
      - `alertType = HIGH_RISK`
      - `priority = P1`
      - `requiresImmediateAction = true`
      - `notificationChannel = #legal-alerts`
30. Connect the high-risk output to it.

31. Add a **Set** node named **Prepare Standard Review**.
    - Keep other fields: enabled
    - Add:
      - `alertType = STANDARD_REVIEW`
      - `priority = P2`
      - `requiresImmediateAction = false`
      - `notificationChannel = #legal-reviews`
32. Connect the medium output to it.
33. Recommended improvement: also connect the fallback or explicit `low` output to **Prepare Standard Review** so low-risk contracts are not dropped.

## 4.8 Create Audit Logging
34. Add a **Set** node named **Prepare Audit Record**.
    - Map:
      - `contractId = {{$json.contractId}}`
      - `riskLevel = {{$json.riskLevel}}`
      - `complianceStatus = {{$json.complianceStatus}}`
      - `approvalRequired = {{$json.approvalRequired}}`
      - `legalSummary = {{$json.legalSummary}}`
      - `auditNotes = {{$json.auditNotes}}`
      - `reviewTimestamp = {{$json.timestamp}}`
      - `alertType = {{$json.alertType}}`
      - `priority = {{$json.priority}}`
    - Recommended improvement: enable **Include Other Input Fields** so `riskClauses`, `approvers`, and other downstream fields remain available.
35. Connect all three alert-prep nodes to **Prepare Audit Record**.

36. Add a **Data Table** node named **Track Contract Reviews**.
    - Choose your contract reviews audit table
    - Auto-map incoming columns
37. Connect **Prepare Audit Record → Track Contract Reviews**.

## 4.9 Create Clause-Level Storage
38. Add a **Split Out** node named **Split Risk Clauses**.
    - Field to split: `riskClauses`
39. Connect **Prepare Audit Record → Split Risk Clauses**.

40. Add a **Set** node named **Prepare Risk Clause Record**.
    - Map:
      - `contractId = {{$('Prepare Audit Record').item.json.contractId}}`
      - `clauseText = {{$json.clauseText}}`
      - `riskType = {{$json.riskType}}`
      - `riskLevel = {{$('Prepare Audit Record').item.json.riskLevel}}`
      - `identifiedAt = {{$now.toISO()}}`
41. Connect **Split Risk Clauses → Prepare Risk Clause Record**.

42. Add a **Data Table** node named **Store Risk Clauses**.
    - Choose your risk-clause table
    - Auto-map incoming columns
43. Connect **Prepare Risk Clause Record → Store Risk Clauses**.

## 4.10 Create Approval Branch
44. Add an **If** node named **Check Approval Required**.
    - Recommended expression: `{{$json.approvalRequired}}`
    - Condition: boolean is true
    - Note: the original workflow uses `{{$json.output.approvalRequired}}`, which likely does not match the actual payload
45. Connect **Track Contract Reviews → Check Approval Required**.

46. Add a **Set** node named **Prepare Approval Log**.
    - Map:
      - `contractId = {{$json.contractId}}`
      - `approvalStatus = APPROVED`
      - `approver = {{$json.approvers}}`
      - `approvalTimestamp = {{$now.toISO()}}`
      - `riskLevel = {{$json.riskLevel}}`
      - `complianceStatus = {{$json.complianceStatus}}`
    - Recommended improvement: if `approvers` is an array, convert it to a string with join logic
47. Connect the true branch of **Check Approval Required** to **Prepare Approval Log**.

48. Add a **Data Table** node named **Log Approval Decision**.
    - Choose your approval log table
    - Auto-map input columns
49. Connect **Prepare Approval Log → Log Approval Decision**.

## 4.11 Create Final Summary
50. Add a **Set** node named **Prepare Final Summary**.
    - Keep other fields: enabled
    - Add:
      - `workflowStatus = COMPLETED`
      - `processingTimestamp = {{$now.toISO()}}`
      - `auditReady = true`
51. Connect:
    - False branch of **Check Approval Required → Prepare Final Summary**
    - **Log Approval Decision → Prepare Final Summary**

## 4.12 Credential Configuration
52. In **Governance Model**, **Contract Review Model**, and **Compliance Model**, attach valid OpenAI credentials.
53. In **Slack Alert Tool**, attach Slack OAuth2 credentials and confirm the bot can post in:
    - `#legal-alerts`
    - `#legal-critical`
    - `#legal-reviews`
54. In **Regulatory Database Tool**, configure endpoint auth and test response structure.
55. In the three **Data Table** nodes, select actual n8n Data Tables.

## 4.13 Testing
56. Activate the workflow only after testing manually with the webhook test URL.
57. Submit a supported text-based contract file.
58. Validate:
    - text extraction quality
    - structured AI output schema compliance
    - proper routing for each risk level
    - audit record insertion
    - risk clause splitting and storage
    - approval logic behavior
59. Strongly recommended fixes before production:
    - preserve `riskClauses` in **Prepare Audit Record**
    - change approval condition to `{{$json.approvalRequired}}`
    - explicitly handle `low` risk in the switch
    - reconsider auto-setting approval status to `APPROVED`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI API key (or compatible LLM) required | Prerequisites |
| Slack workspace with bot credentials required | Prerequisites |
| Google Sheets with review and risk log tabs pre-created | Stated prerequisite in note, but the actual workflow uses n8n Data Tables rather than Google Sheets |
| Regulatory database API endpoint access required | Prerequisites |
| Legal teams auto-triaging uploaded vendor contracts by compliance risk level | Use case |
| Swap the Regulatory Database Tool endpoint to target jurisdiction-specific compliance frameworks such as GDPR, CCPA, MAS | Customization |
| Eliminates manual contract triage, reducing review cycle time significantly | Benefit |
| Setup steps note: import workflow, configure webhook URL, AI credentials, Slack credentials, storage targets, and regulatory API | Configuration guidance |
| The workflow automates end-to-end legal contract review and compliance governance for legal teams, contract managers, and risk officers | Functional context |
| Important inconsistency: notes mention Google Sheets, but implementation uses Data Table nodes | Implementation note |
| Important implementation concern: low-risk contracts are not explicitly connected from the switch fallback | Design note |
| Important implementation concern: approval check likely references the wrong field path | Design note |