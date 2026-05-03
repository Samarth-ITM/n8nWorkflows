Review legal policies with GPT-4o, Gmail, Slack, and Google Sheets

https://n8nworkflows.xyz/workflows/review-legal-policies-with-gpt-4o--gmail--slack--and-google-sheets-14436


# Review legal policies with GPT-4o, Gmail, Slack, and Google Sheets

# 1. Workflow Overview

This workflow automates legal policy intake, AI-assisted review, compliance scoring, approval classification, stakeholder notification, and governance recordkeeping.

Its main purpose is to help legal, compliance, and policy operations teams process submitted policy documents consistently. A user uploads a legal or compliance-related document through an n8n form, the file is converted to text, a central AI governance agent coordinates several specialist AI agents, and the resulting decision is stored across multiple tracking tables while stakeholders are notified by email and Slack.

## 1.1 Input Reception and Document Extraction
A form collects policy metadata, contractual requirements, stakeholder emails, and the source file. The uploaded file is then extracted into plain text so it can be analyzed by AI components.

## 1.2 AI Governance Orchestration
A central Legal Governance Agent receives the extracted text and metadata. It uses a supervising prompt and delegates work to three specialist AI tool-agents:
- Policy Analysis Agent
- Compliance Tracking Agent
- Legal Summary Agent

It can also invoke notification tools for Gmail and Slack, and its output is constrained by a structured parser.

## 1.3 Structured Decision Routing
The governance result is normalized into a fixed schema and then routed by approval status:
- `approved`
- `requires_review`
- `rejected`
- fallback output

In the current implementation, all switch outputs lead to the same downstream preparation path.

## 1.4 Governance Data Preparation and Persistence
A normalized policy record is created first, then branched into four parallel recordkeeping tracks:
- Policy record storage
- Approval tracking
- Compliance tracking
- Audit trail logging

These are written into n8n Data Tables, although the sticky notes mention Google Sheets. The actual JSON uses `dataTable` nodes, not Google Sheets nodes.

## 1.5 Stakeholder Notifications
After policy storage, two communication paths run:
- Email notification path via Gmail
- Slack channel update path via Slack

Email sending is intended to operate per stakeholder, but the current node configuration sends the whole stakeholder string rather than the split individual value.

---

# 2. Block-by-Block Analysis

## Block 1 — Policy Submission and Document Extraction

### Overview
This block serves as the workflow entry point. It captures policy submission details via an n8n form and extracts text from the uploaded policy file so downstream AI nodes can analyze it.

### Nodes Involved
- Policy Submission Form
- Extract Policy Document

### Node Details

#### 1. Policy Submission Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that exposes a hosted form and starts the workflow upon submission.
- **Configuration choices:**
  - Form title: `Legal Policy Submission & Review`
  - Attribution disabled
  - Description explains AI-driven legal/compliance review
  - Required fields:
    - `policyTitle`
    - `policyType` as dropdown
    - `policyDocument` as file upload
    - `stakeholders`
  - Optional/semistructured textarea:
    - `contractualRequirements`
- **Key expressions or variables used:** None in-node; it outputs submitted values as JSON and binary.
- **Input and output connections:**
  - No input; this is a trigger
  - Output → `Extract Policy Document`
- **Version-specific requirements:** `typeVersion 2.5`; requires n8n version supporting Form Trigger v2.x behavior.
- **Edge cases or potential failure types:**
  - Missing required upload or stakeholder field
  - User uploads unsupported file type or malformed binary
  - Large uploads may exceed instance limits
  - Stakeholder list is collected as comma-separated text and is not validated as individual email addresses
- **Sub-workflow reference:** None

#### 2. Extract Policy Document
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Converts the uploaded file binary into extractable text for AI consumption.
- **Configuration choices:**
  - Reads from binary property `policyDocument`
  - No extra extraction options configured
- **Key expressions or variables used:** Reads the form-uploaded binary from `policyDocument`
- **Input and output connections:**
  - Input ← `Policy Submission Form`
  - Output → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - PDF/DOCX parsing can fail if the file is corrupted, scanned-only, encrypted, or unsupported
  - Very large files may hit memory or execution constraints
  - Poor OCR availability means image-only PDFs may extract little or no text
- **Sub-workflow reference:** None

---

## Block 2 — AI Governance Orchestration

### Overview
This is the core decision-making block. A supervisor AI agent coordinates three specialized AI agents, optionally uses Gmail/Slack tools, and returns a structured governance decision parsed into a fixed schema.

### Nodes Involved
- Legal Governance Agent
- Legal Governance Model
- Policy Analysis Agent
- Policy Analysis Model
- Compliance Tracking Agent
- Compliance Tracking Model
- Legal Summary Agent
- Legal Summary Model
- Email Notification Tool
- Slack Notification Tool
- Governance Output Parser

### Node Details

#### 3. Legal Governance Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Primary LLM attached to the supervisor agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2` for stable, lower-variance outputs
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output (AI language model) → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 1.3`; requires LangChain/OpenAI nodes available in the n8n instance.
- **Edge cases or potential failure types:**
  - OpenAI auth error
  - Rate limits
  - Model availability changes
  - Token/context limit issues for large policies
- **Sub-workflow reference:** None

#### 4. Policy Analysis Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  LLM backing the Policy Analysis Agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output → `Policy Analysis Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:** same general OpenAI issues as above
- **Sub-workflow reference:** None

#### 5. Policy Analysis Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist AI tool callable by the supervisor agent to analyze document content.
- **Configuration choices:**
  - Tool input text comes from:
    - `{{ $fromAI('policy_content', 'The policy document content and contractual requirements to analyze', 'string') }}`
  - System prompt instructs it to produce:
    - Summary
    - Key Findings
    - Compliance Status
    - Risk Assessment
    - Recommendations
  - Tool description explains legal policy and contractual analysis role
- **Key expressions or variables used:**
  - `$fromAI('policy_content', ...)`
- **Input and output connections:**
  - AI language model input ← `Policy Analysis Model`
  - AI tool output → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases or potential failure types:**
  - Supervisor agent may pass incomplete or poorly formatted input
  - Long policy text may be truncated by model limits
  - Output may be semistructured rather than perfectly standardized
- **Sub-workflow reference:** None

#### 6. Compliance Tracking Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.2`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output → `Compliance Tracking Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:** OpenAI auth/rate/token issues
- **Sub-workflow reference:** None

#### 7. Compliance Tracking Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist tool for compliance validation and scoring.
- **Configuration choices:**
  - Tool input:
    - `{{ $fromAI('policy_analysis', 'The policy analysis results to validate for compliance', 'string') }}`
  - System message requests:
    - Compliance Score (0–100)
    - Framework Alignment
    - Regulatory Requirements
    - Gaps Identified
    - Remediation Actions
- **Key expressions or variables used:**
  - `$fromAI('policy_analysis', ...)`
- **Input and output connections:**
  - AI language model input ← `Compliance Tracking Model`
  - AI tool output → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases or potential failure types:**
  - Agent may infer frameworks like GDPR/HIPAA/SOC2 even when not relevant unless prompt grounding is improved
  - Compliance score may be subjective and inconsistent across documents
- **Sub-workflow reference:** None

#### 8. Legal Summary Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.3`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output → `Legal Summary Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:** OpenAI auth/rate/token issues
- **Sub-workflow reference:** None

#### 9. Legal Summary Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agentTool`  
  Specialist summarization tool for executive and audit-ready reporting.
- **Configuration choices:**
  - Tool input:
    - `{{ $fromAI('analysis_data', 'The policy analysis and compliance data to summarize', 'string') }}`
  - System prompt requests:
    - Executive Summary
    - Key Legal Findings
    - Compliance Status
    - Approval Recommendations
    - Audit Trail Information
- **Key expressions or variables used:**
  - `$fromAI('analysis_data', ...)`
- **Input and output connections:**
  - AI language model input ← `Legal Summary Model`
  - AI tool output → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 3`
- **Edge cases or potential failure types:**
  - Summaries may omit critical caveats if upstream agent outputs are vague
  - Audit information may be prose rather than structured fields
- **Sub-workflow reference:** None

#### 10. Email Notification Tool
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  AI-callable Gmail tool that the supervisor agent can invoke dynamically.
- **Configuration choices:**
  - Recipient, subject, and message are all determined by AI:
    - `recipient_email`
    - `email_subject`
    - `email_body`
  - Tool description tells the agent to notify stakeholders about review outcomes
- **Key expressions or variables used:**
  - `$fromAI('recipient_email', ...)`
  - `$fromAI('email_subject', ...)`
  - `$fromAI('email_body', ...)`
- **Input and output connections:**
  - AI tool output → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - Gmail OAuth permission issues
  - AI may choose invalid addresses or poor email formatting
  - Deliverability issues or Gmail send limits
  - Duplicates may occur because the workflow also has a separate direct Gmail node later
- **Sub-workflow reference:** None

#### 11. Slack Notification Tool
- **Type and technical role:** `n8n-nodes-base.slackTool`  
  AI-callable Slack messaging tool.
- **Configuration choices:**
  - AI supplies:
    - `message`
    - `channel_id`
  - Uses OAuth2 authentication
  - Tool description tells the agent to post governance updates and alerts
- **Key expressions or variables used:**
  - `$fromAI('message', ...)`
  - `$fromAI('channel_id', ...)`
- **Input and output connections:**
  - AI tool output → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 2.4`
- **Edge cases or potential failure types:**
  - Invalid channel ID
  - Bot not invited to channel
  - Slack OAuth scope issues
  - Duplicate notifications, since a later normal Slack node also posts an update
- **Sub-workflow reference:** None

#### 12. Governance Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the supervisor agent output into a known JSON schema.
- **Configuration choices:**
  - Manual JSON schema with fields:
    - `approvalStatus` enum: approved / requires_review / rejected
    - `riskLevel` enum: low / medium / high / critical
    - `complianceScore` number
    - `policyAnalysisSummary`
    - `complianceFindings`
    - `legalSummary`
    - `approvalRecommendation`
    - `stakeholderNotifications` array of strings
    - `auditNotes`
  - Required fields exclude `stakeholderNotifications` and `auditNotes`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - AI output parser → `Legal Governance Agent`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - Parsing failure if model output violates schema
  - `complianceScore` may be returned as text instead of number
  - Missing required fields causes parser errors
- **Sub-workflow reference:** None

#### 13. Legal Governance Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Main orchestrator that consumes extracted document data, invokes specialist tools, and emits the final governance decision.
- **Configuration choices:**
  - Input text combines form metadata and extracted text:
    - policy title
    - policy type
    - policy content from extracted text
    - contractual requirements
    - stakeholders
  - System message instructs exact orchestration order:
    1. Call policy analysis agent
    2. Call compliance tracking agent
    3. Call legal summary agent
    4. Determine approval status, risk level, score
    5. Notify stakeholders via email or Slack
    6. Maintain governance records
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{ 'Policy Title: ' + $json.policyTitle + '\nPolicy Type: ' + $json.policyType + '\nPolicy Content: ' + $json.text + '\nContractual Requirements: ' + $json.contractualRequirements + '\nStakeholders: ' + $json.stakeholders }}`
- **Input and output connections:**
  - Main input ← `Extract Policy Document`
  - AI language model input ← `Legal Governance Model`
  - AI tool inputs available from:
    - `Policy Analysis Agent`
    - `Compliance Tracking Agent`
    - `Legal Summary Agent`
    - `Email Notification Tool`
    - `Slack Notification Tool`
  - AI output parser input ← `Governance Output Parser`
  - Main output → `Route by Approval Status`
- **Version-specific requirements:** `typeVersion 3.1`
- **Edge cases or potential failure types:**
  - Extracted text may be empty or noisy
  - AI tool chaining may not occur exactly as instructed
  - The agent output structure in downstream nodes is referenced inconsistently (`$json.field` vs `$json.output.field`)
  - If the agent invokes notification tools and downstream Gmail/Slack nodes also notify, stakeholders may receive duplicates
- **Sub-workflow reference:** None

---

## Block 3 — Decision Routing

### Overview
This block routes the parsed governance decision according to approval status. In practice, all configured outcomes currently converge to the same next node, so it behaves more like labeled classification than divergent processing.

### Nodes Involved
- Route by Approval Status

### Node Details

#### 14. Route by Approval Status
- **Type and technical role:** `n8n-nodes-base.switch`  
  Conditional router based on the `approvalStatus` field.
- **Configuration choices:**
  - Output 0: when `approvalStatus = approved`
  - Output 1: when `approvalStatus = requires_review`
  - Output 2: when `approvalStatus = rejected`
  - Fallback output enabled as `extra`
- **Key expressions or variables used:**
  - `{{ $json.approvalStatus }}`
- **Input and output connections:**
  - Input ← `Legal Governance Agent`
  - All outputs currently → `Prepare Policy Record`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - If parser fails upstream, this node may not receive valid fields
  - Fallback branch is not meaningfully differentiated because it also goes to the same path
  - No specialized remediation path exists for rejected or review-required policies
- **Sub-workflow reference:** None

---

## Block 4 — Record Preparation and Persistence

### Overview
This block converts the AI governance decision into normalized records and stores them across four separate tracking datasets. It creates a central policy record first, then branches into approval, compliance, and audit record preparation.

### Nodes Involved
- Prepare Policy Record
- Store Policy Records
- Prepare Approval Record
- Track Approvals
- Prepare Compliance Record
- Track Compliance
- Prepare Audit Log
- Audit Trail

### Node Details

#### 15. Prepare Policy Record
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds the master normalized policy record used for storage and later parallel tracking.
- **Configuration choices:**
  - Creates:
    - `policyId`
    - `policyTitle`
    - `policyType`
    - `submissionDate`
    - `approvalStatus`
    - `riskLevel`
    - `complianceScore`
    - `stakeholders`
    - `contractualRequirements`
    - `policyAnalysisSummary`
    - `complianceFindings`
    - `legalSummary`
    - `approvalRecommendation`
    - `auditNotes`
  - `policyId` is generated from current ISO timestamp plus a slugified policy title
  - Uses form values via explicit node references for original submission fields
- **Key expressions or variables used:**
  - `{{ $now.toISO() + '-' + $json.policyTitle.replace(/\s+/g, '-').toLowerCase() }}`
  - `{{ $('Policy Submission Form').item.json.policyTitle }}`
  - `{{ $('Policy Submission Form').item.json.policyType }}`
  - `{{ $json.auditNotes || 'N/A' }}`
- **Input and output connections:**
  - Input ← `Route by Approval Status`
  - Outputs →
    - `Store Policy Records`
    - `Prepare Approval Record`
    - `Prepare Compliance Record`
    - `Prepare Audit Log`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - Expression uses `$json.policyTitle` for slug generation even though current item after routing may not contain that field directly unless propagated as expected
  - Special characters in title may still produce awkward IDs
  - Duplicate IDs can occur if identical titles are submitted in the same timestamp window
- **Sub-workflow reference:** None

#### 16. Store Policy Records
- **Type and technical role:** `n8n-nodes-base.dataTable`  
  Persists the master policy record to an n8n Data Table.
- **Configuration choices:**
  - Auto-map input data to table columns
  - Requires configured `dataTableId`
- **Key expressions or variables used:** Table ID placeholder:
  - `<__PLACEHOLDER_VALUE__policy_records_table__>`
- **Input and output connections:**
  - Input ← `Prepare Policy Record`
  - Outputs →
    - `Split Stakeholders`
    - `Post Governance Update`
- **Version-specific requirements:** `typeVersion 1.1`; needs n8n Data Tables feature available.
- **Edge cases or potential failure types:**
  - Placeholder table ID must be replaced
  - Auto-mapping fails if destination columns differ or are missing
  - Permission errors if Data Table access is restricted
- **Sub-workflow reference:** None

#### 17. Prepare Approval Record
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Creates:
    - `approvalId`
    - `policyId`
    - `approvalStatus`
    - `approvalDate`
    - `approvalRecommendation`
    - `riskLevel`
    - `stakeholdersNotified`
  - `stakeholdersNotified` joins `stakeholderNotifications` array if present, else `None`
- **Key expressions or variables used:**
  - `{{ $now.toISO() + '-' + $json.policyId }}`
  - `{{ $json.stakeholderNotifications ? $json.stakeholderNotifications.join(', ') : 'None' }}`
- **Input and output connections:**
  - Input ← `Prepare Policy Record`
  - Output → `Track Approvals`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - `stakeholderNotifications` may not exist if agent did not include it
  - Potential mismatch if notification tools sent messages but the array was not populated
- **Sub-workflow reference:** None

#### 18. Track Approvals
- **Type and technical role:** `n8n-nodes-base.dataTable`
- **Configuration choices:**
  - Auto-maps approval tracking fields
  - Requires `approval_tracking_table` ID
- **Key expressions or variables used:**
  - `<__PLACEHOLDER_VALUE__approval_tracking_table__>`
- **Input and output connections:**
  - Input ← `Prepare Approval Record`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Placeholder not configured
  - Schema mismatch with destination table
- **Sub-workflow reference:** None

#### 19. Prepare Compliance Record
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Creates:
    - `complianceId`
    - `policyId`
    - `complianceScore`
    - `complianceFindings`
    - `assessmentDate`
    - `riskLevel`
    - `policyType`
- **Key expressions or variables used:**
  - `{{ $now.toISO() + '-' + $json.policyId }}`
  - `{{ $json.policyType }}`
- **Input and output connections:**
  - Input ← `Prepare Policy Record`
  - Output → `Track Compliance`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - `policyType` must have been preserved in upstream Set node; here it should be, but breakage is possible if mapping changes
- **Sub-workflow reference:** None

#### 20. Track Compliance
- **Type and technical role:** `n8n-nodes-base.dataTable`
- **Configuration choices:**
  - Auto-maps compliance record fields
  - Requires `compliance_tracking_table` ID
- **Key expressions or variables used:**
  - `<__PLACEHOLDER_VALUE__compliance_tracking_table__>`
- **Input and output connections:**
  - Input ← `Prepare Compliance Record`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Placeholder not configured
  - Data type mismatch for numeric compliance score
- **Sub-workflow reference:** None

#### 21. Prepare Audit Log
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**
  - Creates:
    - `auditId`
    - `policyId`
    - `policyTitle`
    - `auditDate`
    - `legalSummary`
    - `policyAnalysisSummary`
    - `complianceScore`
    - `approvalStatus`
    - `riskLevel`
    - `auditNotes`
    - `stakeholders`
  - `auditId` includes `-AUDIT-`
- **Key expressions or variables used:**
  - `{{ $now.toISO() + '-AUDIT-' + $json.policyId }}`
- **Input and output connections:**
  - Input ← `Prepare Policy Record`
  - Output → `Audit Trail`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases or potential failure types:**
  - Large text fields may exceed practical storage size depending on table design
- **Sub-workflow reference:** None

#### 22. Audit Trail
- **Type and technical role:** `n8n-nodes-base.dataTable`
- **Configuration choices:**
  - Auto-maps audit fields
  - Requires `audit_trail_table` ID
- **Key expressions or variables used:**
  - `<__PLACEHOLDER_VALUE__audit_trail_table__>`
- **Input and output connections:**
  - Input ← `Prepare Audit Log`
- **Version-specific requirements:** `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Placeholder not configured
  - Destination schema may not support long-form audit text cleanly
- **Sub-workflow reference:** None

---

## Block 5 — Outbound Notifications

### Overview
This block sends governance results to stakeholders via Gmail and Slack after the master policy record is stored. It includes a stakeholder split step, but the Gmail node does not actually use the split item correctly.

### Nodes Involved
- Split Stakeholders
- Send Stakeholder Email
- Post Governance Update

### Node Details

#### 23. Split Stakeholders
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits an array field into one item per element for iterative processing.
- **Configuration choices:**
  - Field to split: `stakeholders`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Store Policy Records`
  - Output → `Send Stakeholder Email`
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:**
  - Upstream `stakeholders` is stored as a single comma-separated string, not an array
  - If `Split Out` expects an array, this may fail or behave unexpectedly
  - A preceding transformation node is missing to convert `"a@x.com, b@y.com"` into `["a@x.com","b@y.com"]`
- **Sub-workflow reference:** None

#### 24. Send Stakeholder Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Direct Gmail send node used outside the AI-agent tool system.
- **Configuration choices:**
  - Subject includes policy title and approval status
  - Message contains:
    - approval status
    - risk level
    - compliance score
    - legal summary
    - recommendation
    - compliance findings
  - Email type is `text`
- **Key expressions or variables used:**
  - Recipient:
    - `{{ $('Policy Submission Form').item.json.stakeholders }}`
  - Subject:
    - `{{ 'Policy Review Complete: ' + $('Policy Submission Form').item.json.policyTitle + ' - ' + $('Legal Governance Agent').item.json.output.approvalStatus.toUpperCase() }}`
  - Message references `$('Legal Governance Agent').item.json.output...`
- **Input and output connections:**
  - Input ← `Split Stakeholders`
- **Version-specific requirements:** `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - Recipient uses the full original stakeholder string, not the split current item
  - If the agent output is not nested under `.output`, expressions will fail
  - Plain text email contains Markdown-like `**` formatting which will not render as bold in text mode
  - Gmail OAuth/send quota issues
- **Sub-workflow reference:** None

#### 25. Post Governance Update
- **Type and technical role:** `n8n-nodes-base.slack`  
  Direct Slack channel message posted after record storage.
- **Configuration choices:**
  - Posts summary to a fixed channel placeholder
  - Message includes:
    - policy title
    - type
    - status
    - risk level
    - compliance score
    - recommendation
- **Key expressions or variables used:**
  - Channel placeholder:
    - `<__PLACEHOLDER_VALUE__legal_governance_channel__>`
  - Message references `$('Legal Governance Agent').item.json.approvalStatus`, not `.output.approvalStatus`
- **Input and output connections:**
  - Input ← `Store Policy Records`
- **Version-specific requirements:** `typeVersion 2.4`
- **Edge cases or potential failure types:**
  - Possible expression mismatch because email node assumes `.output.*` while Slack node assumes top-level fields
  - Slack bot/channel permissions may be insufficient
  - Placeholder channel ID must be replaced
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Policy Submission Form | n8n-nodes-base.formTrigger | Collect policy metadata, file upload, and stakeholder list |  | Extract Policy Document | ## Policy Submission & Document Extraction<br>**Why** — Form-triggered ingestion and structured text extraction ensure every policy document enters the pipeline in a consistent, processable format. |
| Extract Policy Document | n8n-nodes-base.extractFromFile | Convert uploaded file into text | Policy Submission Form | Legal Governance Agent | ## Policy Submission & Document Extraction<br>**Why** — Form-triggered ingestion and structured text extraction ensure every policy document enters the pipeline in a consistent, processable format. |
| Policy Analysis Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing policy review tool-agent |  | Policy Analysis Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Policy Analysis Agent | @n8n/n8n-nodes-langchain.agentTool | AI tool for legal/policy content review | Policy Analysis Model | Legal Governance Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Compliance Tracking Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing compliance validation tool-agent |  | Compliance Tracking Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Compliance Tracking Agent | @n8n/n8n-nodes-langchain.agentTool | AI tool for compliance scoring and gap detection | Compliance Tracking Model | Legal Governance Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Legal Summary Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing executive/audit summary tool-agent |  | Legal Summary Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Legal Summary Agent | @n8n/n8n-nodes-langchain.agentTool | AI tool for executive and audit-ready summary generation | Legal Summary Model | Legal Governance Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Legal Governance Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Primary LLM for the supervisor agent |  | Legal Governance Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Email Notification Tool | n8n-nodes-base.gmailTool | AI-callable email notification tool |  | Legal Governance Agent | ## Email & Slack Notification Tools<br>**Why** — Multi-channel notifications ensure governance decisions reach stakeholders immediately without manual distribution. |
| Slack Notification Tool | n8n-nodes-base.slackTool | AI-callable Slack notification tool |  | Legal Governance Agent | ## Email & Slack Notification Tools<br>**Why** — Multi-channel notifications ensure governance decisions reach stakeholders immediately without manual distribution. |
| Governance Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured governance output schema |  | Legal Governance Agent | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Legal Governance Agent | @n8n/n8n-nodes-langchain.agent | Supervisor agent orchestrating analysis, compliance, summary, and decisions | Extract Policy Document; Legal Governance Model; Policy Analysis Agent; Compliance Tracking Agent; Legal Summary Agent; Email Notification Tool; Slack Notification Tool; Governance Output Parser | Route by Approval Status | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Route by Approval Status | n8n-nodes-base.switch | Route items based on approval classification | Legal Governance Agent | Prepare Policy Record | ## Route by Approval Status<br>**Why** — Rules-based routing separates approved, compliance-flagged, and audit-required records for proportionate, parallel handling. |
| Prepare Policy Record | n8n-nodes-base.set | Normalize the main policy record | Route by Approval Status | Store Policy Records; Prepare Approval Record; Prepare Compliance Record; Prepare Audit Log | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Store Policy Records | n8n-nodes-base.dataTable | Store master policy record | Prepare Policy Record | Split Stakeholders; Post Governance Update | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Prepare Approval Record | n8n-nodes-base.set | Build approval tracking record | Prepare Policy Record | Track Approvals | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Track Approvals | n8n-nodes-base.dataTable | Store approval decision record | Prepare Approval Record |  | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Prepare Compliance Record | n8n-nodes-base.set | Build compliance tracking record | Prepare Policy Record | Track Compliance | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Track Compliance | n8n-nodes-base.dataTable | Store compliance assessment record | Prepare Compliance Record |  | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Prepare Audit Log | n8n-nodes-base.set | Build audit trail record | Prepare Policy Record | Audit Trail | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Audit Trail | n8n-nodes-base.dataTable | Store audit log record | Prepare Audit Log |  | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Split Stakeholders | n8n-nodes-base.splitOut | Split stakeholder values for iterative emailing | Store Policy Records | Send Stakeholder Email | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Send Stakeholder Email | n8n-nodes-base.gmail | Send detailed review outcome email | Split Stakeholders |  | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Post Governance Update | n8n-nodes-base.slack | Post final policy review summary to Slack | Store Policy Records |  | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Prerequisites<br>- OpenAI API key (or compatible LLM)<br>- Gmail account with OAuth credentials<br>- Slack workspace with bot credentials<br>- Google Sheets with policy, approval, compliance, and audit tabs pre-created<br>## Use Cases<br>- Legal teams automating policy submission review and approval classification<br>## Customisation<br>- Extend routing logic to include partial-approval or escalation tiers for complex policies<br>## Benefits<br>- Eliminates manual policy triage and stakeholder notification across large submission volumes |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Setup Steps<br>1. Import workflow; configure the Policy Submission Form trigger.<br>2. Add AI model credentials to the Legal Governance Agent, Policy Analysis Agent, Compliance Tracking Agent, and Legal Summary Agent.<br>3. Connect Gmail credentials to the Email Notification Tool; link Slack credentials to the Slack Notification Tool.<br>4. Link Google Sheets credentials; set sheet IDs for Policy Records, Approvals, Compliance, and Audit Trail tabs. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## How It Works<br>This workflow automates legal policy governance for legal teams, policy managers, and compliance officers, eliminating manual document review, approval classification, and multi-channel stakeholder distribution. Submitted policy documents are form-triggered, extracted into structured text, and passed to the Legal Governance Agent. Backed by shared memory and a governance model, it coordinates three specialist agents: Policy Analysis (content evaluation), Compliance Tracking (regulatory alignment), and Legal Summary (structured report generation). Gmail and Slack notification tools handle multi-channel alerts in parallel. Outputs are parsed and routed by approval status across four concurrent Google Sheets tracks: policy record storage with stakeholder notifications, approval preparation and tracking, compliance record preparation and tracking, and audit log generation, ensuring a complete, audit-ready governance trail for every submission. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Email & Slack Notification Tools<br>**Why** — Multi-channel notifications ensure governance decisions reach stakeholders immediately without manual distribution. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Legal Governance Agent & Specialist Agents<br>**Why** — Coordinates Policy Analysis, Compliance Tracking, and Legal Summary agents in parallel using shared memory for consistent, context-aware policy evaluation. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Policy Submission & Document Extraction<br>**Why** — Form-triggered ingestion and structured text extraction ensure every policy document enters the pipeline in a consistent, processable format. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Route by Approval Status<br>**Why** — Rules-based routing separates approved, compliance-flagged, and audit-required records for proportionate, parallel handling. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## Policy Records, Approvals, Compliance & Audit Trail<br>**Why** — Four parallel Google Sheets tracks capture policy storage, approval decisions, compliance status, and audit logs — maintaining a complete, audit-ready governance record. |

---

# 4. Reproducing the Workflow from Scratch

Below is a manual build sequence that reproduces the workflow as closely as possible.

## Prerequisites
1. Create or verify credentials in n8n for:
   - OpenAI API
   - Gmail OAuth2
   - Slack OAuth2
2. Ensure your n8n instance supports:
   - Form Trigger
   - Extract From File
   - LangChain AI Agent / AI Tool / OpenAI Chat Model nodes
   - Structured Output Parser
   - Data Tables
3. Create four n8n Data Tables, or decide to replace them with Google Sheets if desired:
   - Policy records
   - Approval tracking
   - Compliance tracking
   - Audit trail
4. Decide on a Slack channel for final governance updates.

## Step-by-step build

1. **Create a Form Trigger node**
   - Name: `Policy Submission Form`
   - Type: `Form Trigger`
   - Set title to `Legal Policy Submission & Review`
   - Set description to:  
     `Submit legal policies and contracts for AI-driven analysis, compliance validation, and governance approval workflow`
   - Disable attribution if desired
   - Add fields:
     1. `policyTitle` — text — required
     2. `policyType` — dropdown — required  
        Options:
        - Contract
        - Internal Policy
        - Compliance Document
        - Legal Agreement
        - Regulatory Requirement
     3. `policyDocument` — file — required
     4. `contractualRequirements` — textarea
     5. `stakeholders` — textarea — required  
        Suggested placeholder: `user@example.com, user@example.com`

2. **Create an Extract From File node**
   - Name: `Extract Policy Document`
   - Type: `Extract From File`
   - Set binary property to `policyDocument`
   - Connect:
     - `Policy Submission Form` → `Extract Policy Document`

3. **Create the supervisor model node**
   - Name: `Legal Governance Model`
   - Type: `OpenAI Chat Model` (LangChain)
   - Credential: OpenAI
   - Model: `gpt-4o`
   - Temperature: `0.2`

4. **Create the policy analysis model**
   - Name: `Policy Analysis Model`
   - Type: `OpenAI Chat Model` (LangChain)
   - Credential: OpenAI
   - Model: `gpt-4o`
   - Temperature: `0.2`

5. **Create the compliance tracking model**
   - Name: `Compliance Tracking Model`
   - Type: `OpenAI Chat Model` (LangChain)
   - Credential: OpenAI
   - Model: `gpt-4o`
   - Temperature: `0.2`

6. **Create the legal summary model**
   - Name: `Legal Summary Model`
   - Type: `OpenAI Chat Model` (LangChain)
   - Credential: OpenAI
   - Model: `gpt-4o`
   - Temperature: `0.3`

7. **Create the Policy Analysis Agent tool**
   - Name: `Policy Analysis Agent`
   - Type: `AI Agent Tool`
   - Connect the model:
     - `Policy Analysis Model` → `Policy Analysis Agent` using AI language model connection
   - Set tool input text to:  
     `{{ $fromAI('policy_content', 'The policy document content and contractual requirements to analyze', 'string') }}`
   - Set tool description to indicate it reviews legal policies, contractual obligations, compliance gaps, and risks.
   - System message should instruct it to return:
     - Summary
     - Key Findings
     - Compliance Status
     - Risk Assessment
     - Recommendations

8. **Create the Compliance Tracking Agent tool**
   - Name: `Compliance Tracking Agent`
   - Type: `AI Agent Tool`
   - Connect:
     - `Compliance Tracking Model` → `Compliance Tracking Agent`
   - Tool input text:  
     `{{ $fromAI('policy_analysis', 'The policy analysis results to validate for compliance', 'string') }}`
   - System message should instruct it to return:
     - Compliance Score (0-100)
     - Framework Alignment
     - Regulatory Requirements
     - Gaps Identified
     - Remediation Actions

9. **Create the Legal Summary Agent tool**
   - Name: `Legal Summary Agent`
   - Type: `AI Agent Tool`
   - Connect:
     - `Legal Summary Model` → `Legal Summary Agent`
   - Tool input text:  
     `{{ $fromAI('analysis_data', 'The policy analysis and compliance data to summarize', 'string') }}`
   - System message should instruct it to return:
     - Executive Summary
     - Key Legal Findings
     - Compliance Status
     - Approval Recommendations
     - Audit Trail Information

10. **Create the AI-callable Gmail tool**
    - Name: `Email Notification Tool`
    - Type: `Gmail Tool`
    - Credential: Gmail OAuth2
    - Set recipient to:  
      `{{ $fromAI('recipient_email', 'The email address of the stakeholder to notify', 'string') }}`
    - Set subject to:  
      `{{ $fromAI('email_subject', 'The email subject line', 'string') }}`
    - Set message to:  
      `{{ $fromAI('email_body', 'The email message content', 'string') }}`
    - Add a tool description explaining it sends status updates and review notifications.

11. **Create the AI-callable Slack tool**
    - Name: `Slack Notification Tool`
    - Type: `Slack Tool`
    - Credential: Slack OAuth2
    - Authentication: OAuth2
    - Channel mode: by channel ID
    - Channel ID expression:  
      `{{ $fromAI('channel_id', 'The Slack channel ID to post to', 'string') }}`
    - Message expression:  
      `{{ $fromAI('message', 'The Slack message content', 'string') }}`
    - Add a tool description explaining it posts governance updates and alerts.

12. **Create the structured output parser**
    - Name: `Governance Output Parser`
    - Type: `Structured Output Parser`
    - Use manual JSON schema
    - Add fields:
      - `approvalStatus`: string enum `approved`, `requires_review`, `rejected`
      - `riskLevel`: string enum `low`, `medium`, `high`, `critical`
      - `complianceScore`: number
      - `policyAnalysisSummary`: string
      - `complianceFindings`: string
      - `legalSummary`: string
      - `approvalRecommendation`: string
      - `stakeholderNotifications`: array of strings
      - `auditNotes`: string
    - Make these required:
      - `approvalStatus`
      - `riskLevel`
      - `complianceScore`
      - `policyAnalysisSummary`
      - `complianceFindings`
      - `legalSummary`
      - `approvalRecommendation`

13. **Create the supervisor AI agent**
    - Name: `Legal Governance Agent`
    - Type: `AI Agent`
    - Connect:
      - `Extract Policy Document` → `Legal Governance Agent` (main)
      - `Legal Governance Model` → `Legal Governance Agent` (AI language model)
      - `Policy Analysis Agent` → `Legal Governance Agent` (AI tool)
      - `Compliance Tracking Agent` → `Legal Governance Agent` (AI tool)
      - `Legal Summary Agent` → `Legal Governance Agent` (AI tool)
      - `Email Notification Tool` → `Legal Governance Agent` (AI tool)
      - `Slack Notification Tool` → `Legal Governance Agent` (AI tool)
      - `Governance Output Parser` → `Legal Governance Agent` (AI output parser)
    - Set input text to:
      `Policy Title: {{policyTitle}}`
      `Policy Type: {{policyType}}`
      `Policy Content: {{text}}`
      `Contractual Requirements: {{contractualRequirements}}`
      `Stakeholders: {{stakeholders}}`
      
      In n8n expression form:
      `{{ 'Policy Title: ' + $json.policyTitle + '\nPolicy Type: ' + $json.policyType + '\nPolicy Content: ' + $json.text + '\nContractual Requirements: ' + $json.contractualRequirements + '\nStakeholders: ' + $json.stakeholders }}`
    - Add a system prompt instructing the agent to:
      1. Call the policy analysis agent
      2. Call the compliance tracking agent
      3. Call the legal summary agent
      4. Determine approval status, risk level, and compliance score
      5. Notify stakeholders as needed
      6. Return a structured governance decision
    - Enable output parser usage

14. **Create a Switch node**
    - Name: `Route by Approval Status`
    - Type: `Switch`
    - Connect:
      - `Legal Governance Agent` → `Route by Approval Status`
    - Add rules:
      - If `{{ $json.approvalStatus }}` equals `approved`
      - If `{{ $json.approvalStatus }}` equals `requires_review`
      - If `{{ $json.approvalStatus }}` equals `rejected`
    - Enable fallback output
    - Connect every output, including fallback, to the same next node:
      - `Prepare Policy Record`

15. **Create the master record Set node**
    - Name: `Prepare Policy Record`
    - Type: `Set`
    - Add fields:
      - `policyId` = `{{ $now.toISO() + '-' + $json.policyTitle.replace(/\s+/g, '-').toLowerCase() }}`
      - `policyTitle` = `{{ $('Policy Submission Form').item.json.policyTitle }}`
      - `policyType` = `{{ $('Policy Submission Form').item.json.policyType }}`
      - `submissionDate` = `{{ $now.toISO() }}`
      - `approvalStatus` = `{{ $json.approvalStatus }}`
      - `riskLevel` = `{{ $json.riskLevel }}`
      - `complianceScore` = `{{ $json.complianceScore }}`
      - `stakeholders` = `{{ $('Policy Submission Form').item.json.stakeholders }}`
      - `contractualRequirements` = `{{ $('Policy Submission Form').item.json.contractualRequirements }}`
      - `policyAnalysisSummary` = `{{ $json.policyAnalysisSummary }}`
      - `complianceFindings` = `{{ $json.complianceFindings }}`
      - `legalSummary` = `{{ $json.legalSummary }}`
      - `approvalRecommendation` = `{{ $json.approvalRecommendation }}`
      - `auditNotes` = `{{ $json.auditNotes || 'N/A' }}`
    - Connect:
      - `Route by Approval Status` → `Prepare Policy Record`

16. **Create the policy storage node**
    - Name: `Store Policy Records`
    - Type: `Data Table`
    - Configure target table as your policy-records table
    - Use auto-mapping
    - Connect:
      - `Prepare Policy Record` → `Store Policy Records`

17. **Create the approval record Set node**
    - Name: `Prepare Approval Record`
    - Type: `Set`
    - Fields:
      - `approvalId` = `{{ $now.toISO() + '-' + $json.policyId }}`
      - `policyId` = `{{ $json.policyId }}`
      - `approvalStatus` = `{{ $json.approvalStatus }}`
      - `approvalDate` = `{{ $now.toISO() }}`
      - `approvalRecommendation` = `{{ $json.approvalRecommendation }}`
      - `riskLevel` = `{{ $json.riskLevel }}`
      - `stakeholdersNotified` = `{{ $json.stakeholderNotifications ? $json.stakeholderNotifications.join(', ') : 'None' }}`
    - Connect:
      - `Prepare Policy Record` → `Prepare Approval Record`

18. **Create the approval tracking storage node**
    - Name: `Track Approvals`
    - Type: `Data Table`
    - Configure target table as your approvals table
    - Use auto-mapping
    - Connect:
      - `Prepare Approval Record` → `Track Approvals`

19. **Create the compliance record Set node**
    - Name: `Prepare Compliance Record`
    - Type: `Set`
    - Fields:
      - `complianceId` = `{{ $now.toISO() + '-' + $json.policyId }}`
      - `policyId` = `{{ $json.policyId }}`
      - `complianceScore` = `{{ $json.complianceScore }}`
      - `complianceFindings` = `{{ $json.complianceFindings }}`
      - `assessmentDate` = `{{ $now.toISO() }}`
      - `riskLevel` = `{{ $json.riskLevel }}`
      - `policyType` = `{{ $json.policyType }}`
    - Connect:
      - `Prepare Policy Record` → `Prepare Compliance Record`

20. **Create the compliance tracking storage node**
    - Name: `Track Compliance`
    - Type: `Data Table`
    - Configure target table as your compliance table
    - Use auto-mapping
    - Connect:
      - `Prepare Compliance Record` → `Track Compliance`

21. **Create the audit record Set node**
    - Name: `Prepare Audit Log`
    - Type: `Set`
    - Fields:
      - `auditId` = `{{ $now.toISO() + '-AUDIT-' + $json.policyId }}`
      - `policyId` = `{{ $json.policyId }}`
      - `policyTitle` = `{{ $json.policyTitle }}`
      - `auditDate` = `{{ $now.toISO() }}`
      - `legalSummary` = `{{ $json.legalSummary }}`
      - `policyAnalysisSummary` = `{{ $json.policyAnalysisSummary }}`
      - `complianceScore` = `{{ $json.complianceScore }}`
      - `approvalStatus` = `{{ $json.approvalStatus }}`
      - `riskLevel` = `{{ $json.riskLevel }}`
      - `auditNotes` = `{{ $json.auditNotes }}`
      - `stakeholders` = `{{ $json.stakeholders }}`
    - Connect:
      - `Prepare Policy Record` → `Prepare Audit Log`

22. **Create the audit storage node**
    - Name: `Audit Trail`
    - Type: `Data Table`
    - Configure target table as your audit table
    - Use auto-mapping
    - Connect:
      - `Prepare Audit Log` → `Audit Trail`

23. **Create the stakeholder split node**
    - Name: `Split Stakeholders`
    - Type: `Split Out`
    - Field to split: `stakeholders`
    - Connect:
      - `Store Policy Records` → `Split Stakeholders`

24. **Create the direct Gmail notification node**
    - Name: `Send Stakeholder Email`
    - Type: `Gmail`
    - Credential: Gmail OAuth2
    - Email type: text
    - Set subject and body using the same style as the source workflow:
      - Subject should include policy title and approval status
      - Body should include legal summary, recommendation, and compliance findings
    - Connect:
      - `Split Stakeholders` → `Send Stakeholder Email`

25. **Create the direct Slack notification node**
    - Name: `Post Governance Update`
    - Type: `Slack`
    - Credential: Slack OAuth2
    - Authentication: OAuth2
    - Select a fixed channel ID
    - Compose a concise governance update message with title, type, status, risk level, score, and recommendation
    - Connect:
      - `Store Policy Records` → `Post Governance Update`

---

## Recommended fixes while rebuilding
To make the workflow function more reliably, apply these improvements even though they are not present in the JSON exactly as-is:

1. **Convert stakeholder string to array before `Split Stakeholders`**
   - Add a Code or Set node after `Prepare Policy Record`:
     - Split by comma
     - Trim whitespace
     - Remove empty values

2. **Use the split item in Gmail**
   - In `Send Stakeholder Email`, use the current item value rather than the original full stakeholder string.

3. **Normalize AI output path usage**
   - Choose one format and keep it consistent:
     - either `$('Legal Governance Agent').item.json.approvalStatus`
     - or `$('Legal Governance Agent').item.json.output.approvalStatus`
   - The current workflow mixes both.

4. **Avoid duplicate notifications**
   - Decide whether notifications should be:
     - AI-agent initiated via tool nodes
     - or deterministic via normal Gmail/Slack nodes
   - Using both can send duplicates.

5. **Replace placeholder IDs**
   - Data table IDs
   - Slack channel ID

6. **If you really want Google Sheets**
   - Replace each `Data Table` node with a Google Sheets append/update node
   - Create worksheets for:
     - Policy Records
     - Approvals
     - Compliance
     - Audit Trail

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites listed in the canvas mention OpenAI, Gmail, Slack, and Google Sheets. However, the actual workflow JSON uses n8n Data Tables rather than Google Sheets nodes. | Workflow design note |
| The canvas says specialist agents operate “in parallel,” but the supervisor prompt explicitly asks the Legal Governance Agent to call them in sequence: policy analysis first, then compliance tracking, then legal summary. | Behavioral clarification |
| The title provided by the user is “Review legal policies with GPT-4o, Gmail, Slack, and Google Sheets,” while the workflow object name is “AI-powered legal policy governance with compliance tracking and notification.” | Naming clarification |
| Sticky note setup instructions say to configure Google Sheets credentials and sheet IDs. In practice, this JSON requires Data Table IDs instead. | Configuration mismatch |
| Sticky note use case: legal teams automating policy submission review and approval classification. | Canvas note |
| Sticky note customization: extend routing logic to support partial approval or escalation tiers for complex policies. | Canvas note |
| Sticky note benefit: eliminates manual policy triage and stakeholder notification across large submission volumes. | Canvas note |