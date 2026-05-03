Analyze legislative policies and route compliance actions with GPT-4o, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/analyze-legislative-policies-and-route-compliance-actions-with-gpt-4o--google-sheets--and-slack-13451


# Analyze legislative policies and route compliance actions with GPT-4o, Google Sheets, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Analyze legislative policies and route compliance actions with GPT-4o, Google Sheets, and Slack  
**Workflow name (JSON):** AI-driven legislative policy analysis and governance compliance orchestration

**Purpose:**  
Automates intake of legislative/regulatory documents (PDF via URL), extracts text, runs a structured policy interpretation using GPT‑4o, then orchestrates parallel specialist analyses (impact, compliance mapping, stakeholder notification). Based on an orchestration decision and **review status**, it routes the case to either (a) legal review (Slack + wait webhook) or (b) auto-approval logging to Google Sheets, while maintaining audit traceability.

**Target use cases:** regulatory change management, governance/compliance triage, rapid policy impact assessment, and stakeholder communication routing.

### 1.1 Legislative Document Ingestion
Receives a document submission via webhook, applies baseline workflow configuration, fetches the PDF from a provided URL, and extracts joined text for analysis.

### 1.2 Policy Interpretation (Structured Extraction)
Uses a GPT‑4o agent with a structured output parser to extract document metadata, scope, key provisions, requirements, risk, and whether legal review is required.

### 1.3 Governance Orchestration + Parallel Specialist Tools
A governance agent coordinates three specialist “agent tools” (impact assessment, compliance mapping, stakeholder notification). Outputs are structured and synthesized into a single orchestration result.

### 1.4 Routing, Legal Review Loop, Logging, Notifications
Routes by `reviewStatus`:
- **REQUIRES_HUMAN_REVIEW** → Wait for legal review (resume webhook) → notify legal in Slack → log to compliance tracker (Google Sheets) → notify stakeholders (Slack).
- **APPROVED** → log auto-approved to a separate Google Sheet.
Fallback path exists for unexpected statuses but is not connected downstream in this workflow.

---

## 2. Block-by-Block Analysis

### Block 1 — Legislative Document Ingestion
**Overview:** Accepts a POST request with a `documentUrl`, sets internal configuration fields, downloads the document as a file, and extracts full text from PDF for downstream LLM processing.  
**Nodes involved:**  
- Legislative Document Submission (Webhook)  
- Workflow Configuration (Set)  
- Fetch Legislative Document (HTTP Request)  
- Extract Document Text (Extract From File)

#### 2.1 Legislative Document Submission
- **Type / role:** `Webhook` — workflow entry point (HTTP POST).
- **Key configuration:**
  - **Path:** `legislative-policy-submission`
  - **Method:** `POST`
  - **Response mode:** `lastNode` (the HTTP response is whatever the final node in the executed branch returns).
- **Expected input payload:** at minimum `documentUrl` (used later by HTTP Request).
- **Outputs:** JSON to “Workflow Configuration”.
- **Edge cases / failures:**
  - Missing/invalid `documentUrl` → downstream HTTP request fails.
  - If branches end in nodes that don’t return helpful JSON, webhook response may be empty/unexpected.
  - Large requests/timeouts if clients expect immediate response while legal-review path waits (see Block 4; `Wait` will pause execution).

#### 2.2 Workflow Configuration
- **Type / role:** `Set` — injects workflow-level parameters for later logic/customization.
- **Configuration choices:**
  - Adds:
    - `legalReviewThreshold = "HIGH"`
    - `complianceFrameworks = "GDPR,CCPA,SOX"`
    - `stakeholderChannels = "#legal-compliance,#governance"`
  - **Include other fields:** enabled (keeps original webhook payload fields like `documentUrl`).
- **Outputs:** to “Fetch Legislative Document”.
- **Edge cases:**
  - These values are not actually referenced by later nodes in this JSON; changes here will not change routing unless you also update agent prompts/switch logic.

#### 2.3 Fetch Legislative Document
- **Type / role:** `HTTP Request` — downloads the legislative document file.
- **Key configuration:**
  - **URL:** `={{ $json.documentUrl }}`
  - **Response format:** `file` (binary data output)
- **Outputs:** binary file to “Extract Document Text”.
- **Edge cases / failures:**
  - 401/403/404 from source, expiring signed URLs.
  - Non-PDF content returned (HTML, DOCX) will break PDF extraction.
  - Large PDFs → memory/time limits.

#### 2.4 Extract Document Text
- **Type / role:** `Extract From File` — extracts text from PDF binary.
- **Key configuration:**
  - **Operation:** `pdf`
  - **Join pages:** `true` (single concatenated text)
- **Outputs:** `text` field to “Policy Interpretation Agent”.
- **Edge cases / failures:**
  - Scanned PDFs (image-only) will yield empty/poor text (no OCR here).
  - Encrypted PDFs can fail extraction.

---

### Block 2 — Policy Interpretation (Structured Extraction)
**Overview:** Converts raw extracted text into machine-readable policy analysis using a GPT‑4o agent and a structured schema output parser.  
**Nodes involved:**  
- OpenAI Model - Policy Interpretation  
- Policy Analysis Output Parser  
- Policy Interpretation Agent

#### 2.5 OpenAI Model - Policy Interpretation
- **Type / role:** `lmChatOpenAi` — provides GPT‑4o chat model to the agent.
- **Key configuration:**
  - **Model:** `gpt-4o`
  - **Temperature:** `0.1` (low variance; stable extraction)
- **Credentials:** OpenAI API credential (`OpenAi account`).
- **Connections:** feeds the Policy Interpretation Agent via `ai_languageModel`.
- **Failure types:**
  - Invalid API key / quota exceeded.
  - Model name not available in your region/account.
  - Context length exceeded with very large extracted text.

#### 2.6 Policy Analysis Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON output structure for policy interpretation.
- **Schema highlights:** `documentType`, `jurisdiction`, `effectiveDate`, `scope`, `keyProvisions[]`, `complianceRequirements[]`, `riskLevel`, `requiresLegalReview`, `reasoning`.
- **Connections:** attached to Policy Interpretation Agent via `ai_outputParser`.
- **Edge cases:**
  - If the model returns malformed JSON, parser can fail and stop the workflow.
  - Dates may not match `YYYY-MM-DD` (parser expects structure but can’t always validate semantics).

#### 2.7 Policy Interpretation Agent
- **Type / role:** `agent` — runs the interpretation prompt on extracted text and returns structured analysis.
- **Key configuration:**
  - **Input text:** `={{ $json.text }}`
  - **System message:** instructs triage (not legal advice), risk level assessment, and flags human review for HIGH/CRITICAL.
  - **Has output parser:** enabled (uses “Policy Analysis Output Parser”)
- **Outputs:** `output` object (parsed JSON) to “Governance Orchestration Agent”.
- **Edge cases:**
  - Empty extracted text → low-quality analysis or parser failure if fields missing.
  - Ambiguous effective dates/jurisdiction → model may guess; downstream decisions depend on this.

---

### Block 3 — Governance Orchestration + Parallel Specialist Tools
**Overview:** A governance agent calls three specialist tools (impact, compliance mapping, stakeholder notification). Each tool uses GPT‑4o + a structured parser. The governance agent synthesizes results into a coordinated plan and review status.  
**Nodes involved:**  
- OpenAI Model - Governance Orchestration  
- Governance Orchestration Output Parser  
- Governance Orchestration Agent  
- Impact Assessment Agent Tool + OpenAI Model - Impact Assessment + Impact Assessment Output Parser  
- Compliance Mapping Agent Tool + OpenAI Model - Compliance Mapping + Compliance Mapping Output Parser  
- Stakeholder Notification Agent Tool + OpenAI Model - Stakeholder Notification + Stakeholder Notification Output Parser

#### 2.8 OpenAI Model - Governance Orchestration
- **Type / role:** `lmChatOpenAi` — model provider for the governance agent.
- **Config:** `gpt-4o`, temperature `0.2`.
- **Connections:** to Governance Orchestration Agent via `ai_languageModel`.
- **Failures:** same as other OpenAI model nodes (auth/quota/context).

#### 2.9 Governance Orchestration Output Parser
- **Type / role:** structured output parser for orchestration decision.
- **Schema highlights:**
  - `orchestrationDecision`: `LEGAL_REVIEW_REQUIRED | AUTO_APPROVE | ESCALATE`
  - `reviewStatus`: `REQUIRES_HUMAN_REVIEW | APPROVED | PENDING`
  - Summaries: `impactSummary`, `complianceSummary`, `stakeholderSummary`
  - `coordinatedActions[]`, `nextSteps[]`, `reasoning`
- **Connections:** to Governance Orchestration Agent via `ai_outputParser`.
- **Edge cases:** parser failures on malformed JSON; summary fields omitted.

#### 2.10 Governance Orchestration Agent
- **Type / role:** `agent` — central coordinator; invokes 3 AI tools then synthesizes.
- **Key configuration:**
  - **Input text:** `=Policy Analysis Results: {{ $json.output }}`
    - Note: `$json.output` here is the output of Policy Interpretation Agent (structured JSON).
  - **System message:** instructs to call the three tools and decide routing; prohibits legal advice.
  - **Has output parser:** enabled.
- **Tool connections (AI Tool ports):**
  - Impact Assessment Agent Tool
  - Compliance Mapping Agent Tool
  - Stakeholder Notification Agent Tool
- **Outputs:** to “Route by Review Status”.
- **Edge cases:**
  - If any tool call fails (OpenAI error or parser error), orchestration may fail.
  - Review status values must match switch node rules exactly.

---

#### 2.11 Impact Assessment Agent Tool
- **Type / role:** `agentTool` — specialist tool callable by the governance agent.
- **Input text expression:**
  - `={{ $fromAI("policyAnalysis", "Policy interpretation results including scope, provisions, and risk level", "json") }}`
  - This relies on n8n’s AI context mapping; it expects the governance agent runtime to provide a `policyAnalysis` context object.
- **System message:** estimates affected areas, costs, resources, timeline, mitigation, `priorityScore (0-100)`.
- **Has output parser:** yes (Impact Assessment Output Parser).
- **Connections:**
  - `ai_languageModel` from “OpenAI Model - Impact Assessment”
  - `ai_outputParser` from “Impact Assessment Output Parser”
  - Tool exposed to Governance Orchestration Agent via `ai_tool`
- **Failure types:**
  - If `$fromAI(...)` cannot resolve (context naming mismatch), tool input may be empty → poor output or parser failure.

#### 2.12 OpenAI Model - Impact Assessment
- **Type / role:** model provider for Impact tool.
- **Config:** `gpt-4o`, temperature `0.2`.
- **Credentials:** OpenAI.
- **Edge cases:** same OpenAI concerns (rate limits, context).

#### 2.13 Impact Assessment Output Parser
- **Type / role:** structured parser for impact output.
- **Schema highlights:** `impactAreas[]`, `estimatedCost`, `implementationTimeline`, `resourceRequirements[]`, `businessRisks[]`, `mitigationStrategies[]`, `priorityScore`, `reasoning`.

---

#### 2.14 Compliance Mapping Agent Tool
- **Type / role:** `agentTool` — maps requirements to frameworks and actions.
- **Input text expression:** `={{ $fromAI("policyAnalysis", "Policy interpretation results", "json") }}`
- **System message:** map to frameworks (GDPR/CCPA/SOX/etc), gaps, actions, deadlines, docs, training, `complianceStatus`.
- **Has output parser:** yes (Compliance Mapping Output Parser).
- **Connections:** model + parser, exposed as `ai_tool` to governance agent.
- **Edge cases:** same `$fromAI` context resolution risk; also deadlines may be guessed if not in source document.

#### 2.15 OpenAI Model - Compliance Mapping
- **Type / role:** model provider (`gpt-4o`, temperature `0.2`).

#### 2.16 Compliance Mapping Output Parser
- **Type / role:** structured parser.
- **Schema highlights:** `applicableFrameworks[]`, `complianceGaps[]`, `requiredActions[]`, `complianceDeadlines[{framework,deadline}]`, `documentationNeeds[]`, `trainingRequirements[]`, `complianceStatus`, `reasoning`.

---

#### 2.17 Stakeholder Notification Agent Tool
- **Type / role:** `agentTool` — drafts notification strategy.
- **Input text expression:** `={{ $fromAI("policyAnalysis", "Policy interpretation results", "json") }}`
- **System message:** groups, urgency, message, action items, deadline, escalation.
- **Has output parser:** yes (Stakeholder Notification Output Parser).
- **Connections:** model + parser, exposed to governance agent.
- **Edge cases:** may propose channels/roles not present in your org; output is advisory.

#### 2.18 OpenAI Model - Stakeholder Notification
- **Type / role:** model provider (`gpt-4o`, temperature `0.3`).

#### 2.19 Stakeholder Notification Output Parser
- **Type / role:** structured parser.
- **Schema highlights:** `stakeholderGroups[]`, `notificationPriority`, `messageContent`, `actionItems[]`, `responseDeadline`, `escalationPath[]`, `reasoning`.

---

### Block 4 — Routing, Legal Review Loop, Logging, Notifications
**Overview:** Routes orchestration results by `reviewStatus`. If human review is required, it pauses for an external webhook signal, then notifies legal via Slack and logs to Sheets before notifying stakeholders. Auto-approved items are logged separately.  
**Nodes involved:**  
- Route by Review Status (Switch)  
- Wait for Legal Review (Wait)  
- Notify Legal Team (Slack)  
- Log to Compliance Tracker (Google Sheets)  
- Notify Stakeholders (Slack)  
- Log Auto-Approved (Google Sheets)

#### 2.20 Route by Review Status
- **Type / role:** `Switch` — branch on orchestration output.
- **Rules (string equals):**
  - Output “Legal Review Required” if `{{$json.output.reviewStatus}} == "REQUIRES_HUMAN_REVIEW"`
  - Output “Auto-Approved” if `{{$json.output.reviewStatus}} == "APPROVED"`
  - Fallback output renamed to “Unprocessed”
- **Connections:**
  - “Legal Review Required” → Wait for Legal Review
  - “Auto-Approved” → Log Auto-Approved
  - “Unprocessed” → not connected (execution would end there)
- **Edge cases:**
  - If reviewStatus is `"PENDING"` (allowed by schema) it will go to **Unprocessed** and stop (no logging/alerts).
  - If `output` is missing (parser failure upstream), expression errors may occur.

#### 2.21 Wait for Legal Review
- **Type / role:** `Wait` — pauses execution until a resume webhook is called.
- **Config:**
  - **Resume mode:** `webhook`
  - **Method:** `POST`
- **Outputs:** resumes to “Notify Legal Team”.
- **Important behavior:** While waiting, the original webhook call (Block 1) will not get a “lastNode” response until the wait is resumed (unless n8n responds earlier due to configuration outside this JSON). This can cause client timeouts.
- **Edge cases:**
  - Resume webhook URL must be reachable and protected appropriately; otherwise anyone could resume.
  - No validation of approval/rejection payload is implemented here; it simply resumes.

#### 2.22 Notify Legal Team
- **Type / role:** `Slack` — sends a legal review request message.
- **Config:**
  - Channel selection by **channelId** placeholder (`<__PLACEHOLDER_VALUE__Legal Team Slack Channel ID__>`).
  - Message includes:
    - Fields from **Policy Interpretation Agent** (`documentType`, `jurisdiction`, `riskLevel`, `effectiveDate`, `scope`)
    - Summaries from current JSON (governance orchestration output): `impactSummary`, `complianceSummary`, `nextSteps`
    - `Review Webhook: {{ $resumeWebhookUrl }}` (the wait node’s resume webhook URL)
  - **Auth:** Slack OAuth2 credential.
- **Connections:** to “Log to Compliance Tracker”.
- **Edge cases / failures:**
  - Slack OAuth scopes missing (e.g., `chat:write`).
  - Invalid channel ID or bot not in channel.
  - `$resumeWebhookUrl` is only meaningful in context of a Wait node; ensure the message is sent after the Wait is created/resolved.

#### 2.23 Log to Compliance Tracker
- **Type / role:** `Google Sheets` — append or update audit record.
- **Operation:** `appendOrUpdate`
- **Document / Sheet:** placeholders for Spreadsheet ID and Sheet Name.
- **Mapping fields:**
  - `documentType`, `jurisdiction`, `effectiveDate`, `riskLevel` from Policy Interpretation Agent output
  - `reviewStatus` from Governance Orchestration Agent output
  - `timestamp = {{$now.toISO()}}`
- **Matching columns:** `documentType` (this means multiple distinct laws with same documentType can overwrite/update unexpectedly).
- **Connections:** to “Notify Stakeholders”.
- **Edge cases / failures:**
  - OAuth issues, missing permissions.
  - Schema mismatch if sheet columns differ.
  - Using `documentType` as the only match key is collision-prone; consider matching on a unique document ID, URL hash, or timestamp.

#### 2.24 Notify Stakeholders
- **Type / role:** `Slack` — posts a completion/update message for stakeholders.
- **Config:**
  - ChannelId placeholder (`<__PLACEHOLDER_VALUE__Stakeholder Slack Channel ID__>`).
  - References:
    - Policy Interpretation Agent: `documentType`, `effectiveDate`
    - Governance Orchestration Agent: `coordinatedActions`, `stakeholderSummary`
- **Edge cases:** same Slack auth/channel concerns.

#### 2.25 Log Auto-Approved
- **Type / role:** `Google Sheets` — logs auto-approved items.
- **Operation:** `appendOrUpdate`
- **Document / Sheet:** placeholders for Spreadsheet ID and Sheet Name.
- **Mapping fields:**
  - `status = "AUTO_APPROVED"`
  - `documentType`, `jurisdiction`, `effectiveDate`, `riskLevel`, `timestamp`
- **Matching columns:** `documentType` (same collision risk as above).
- **Edge cases:** same Google Sheets concerns.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Legislative Document Submission | Webhook | Entry point to receive document submissions | — | Workflow Configuration | ## Legislative Document Ingestion<br>**What** –Receives submitted legislative documents and fetches full text via HTTP.<br>**Why** – Ensures complete, unprocessed source material enters the pipeline for accurate downstream analysis. |
| Workflow Configuration | Set | Adds workflow parameters and keeps incoming fields | Legislative Document Submission | Fetch Legislative Document | ## Legislative Document Ingestion<br>**What** –Receives submitted legislative documents and fetches full text via HTTP.<br>**Why** – Ensures complete, unprocessed source material enters the pipeline for accurate downstream analysis. |
| Fetch Legislative Document | HTTP Request | Downloads PDF from `documentUrl` | Workflow Configuration | Extract Document Text | ## Legislative Document Ingestion<br>**What** –Receives submitted legislative documents and fetches full text via HTTP.<br>**Why** – Ensures complete, unprocessed source material enters the pipeline for accurate downstream analysis. |
| Extract Document Text | Extract From File | Extracts joined text from PDF | Fetch Legislative Document | Policy Interpretation Agent | ## Legislative Document Ingestion<br>**What** –Receives submitted legislative documents and fetches full text via HTTP.<br>**Why** – Ensures complete, unprocessed source material enters the pipeline for accurate downstream analysis. |
| OpenAI Model - Policy Interpretation | OpenAI Chat Model (LangChain) | LLM provider for policy interpretation | — (AI wiring) | Policy Interpretation Agent (ai_languageModel) | ## Policy Interpretation Agent<br>**What** –OpenAI agent parses regulatory language and extracts structured policy signals.<br>**Why** – Converts dense legal text into machine-readable insights that governance agents can act on. |
| Policy Analysis Output Parser | Structured Output Parser | Enforces schema for policy interpretation | — (AI wiring) | Policy Interpretation Agent (ai_outputParser) | ## Policy Interpretation Agent<br>**What** –OpenAI agent parses regulatory language and extracts structured policy signals.<br>**Why** – Converts dense legal text into machine-readable insights that governance agents can act on. |
| Policy Interpretation Agent | LangChain Agent | Produces structured policy analysis from extracted text | Extract Document Text | Governance Orchestration Agent | ## Policy Interpretation Agent<br>**What** –OpenAI agent parses regulatory language and extracts structured policy signals.<br>**Why** – Converts dense legal text into machine-readable insights that governance agents can act on. |
| OpenAI Model - Governance Orchestration | OpenAI Chat Model (LangChain) | LLM provider for orchestration agent | — (AI wiring) | Governance Orchestration Agent (ai_languageModel) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Governance Orchestration Output Parser | Structured Output Parser | Enforces schema for orchestration decision | — (AI wiring) | Governance Orchestration Agent (ai_outputParser) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Governance Orchestration Agent | LangChain Agent | Calls specialist tools; synthesizes final decision | Policy Interpretation Agent | Route by Review Status | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Impact Assessment Agent Tool | Agent Tool | Specialist impact analysis callable by orchestrator | — (called by agent) | Governance Orchestration Agent (ai_tool) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| OpenAI Model - Impact Assessment | OpenAI Chat Model (LangChain) | LLM provider for impact tool | — (AI wiring) | Impact Assessment Agent Tool (ai_languageModel) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Impact Assessment Output Parser | Structured Output Parser | Enforces schema for impact results | — (AI wiring) | Impact Assessment Agent Tool (ai_outputParser) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Compliance Mapping Agent Tool | Agent Tool | Specialist compliance mapping callable by orchestrator | — (called by agent) | Governance Orchestration Agent (ai_tool) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| OpenAI Model - Compliance Mapping | OpenAI Chat Model (LangChain) | LLM provider for compliance tool | — (AI wiring) | Compliance Mapping Agent Tool (ai_languageModel) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Compliance Mapping Output Parser | Structured Output Parser | Enforces schema for compliance mapping | — (AI wiring) | Compliance Mapping Agent Tool (ai_outputParser) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Stakeholder Notification Agent Tool | Agent Tool | Specialist notification strategy callable by orchestrator | — (called by agent) | Governance Orchestration Agent (ai_tool) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| OpenAI Model - Stakeholder Notification | OpenAI Chat Model (LangChain) | LLM provider for notification tool | — (AI wiring) | Stakeholder Notification Agent Tool (ai_languageModel) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Stakeholder Notification Output Parser | Structured Output Parser | Enforces schema for stakeholder notification | — (AI wiring) | Stakeholder Notification Agent Tool (ai_outputParser) | ## Parallel Governance Agent Processing<br>**What** –Impact Assessment, Compliance Mapping, and Stakeholder Notification agents run concurrently.<br>**Why** – Simultaneous specialist analysis reduces processing time and ensures comprehensive multi-dimensional compliance coverage. |
| Route by Review Status | Switch | Routes execution based on `reviewStatus` | Governance Orchestration Agent | Wait for Legal Review; Log Auto-Approved | ## Route by Review Status, Notify, Log & Alert<br><br>**What** – Branch based on auto-approval or escalation; log outcomes in Google Sheets, alert legal via Slack, and email stakeholders.<br>**Why** – Ensures only high-risk items require human review while maintaining full traceability and timely communication. |
| Wait for Legal Review | Wait | Pauses until resume webhook is called | Route by Review Status | Notify Legal Team | ## Route by Review Status, Notify, Log & Alert<br><br>**What** – Branch based on auto-approval or escalation; log outcomes in Google Sheets, alert legal via Slack, and email stakeholders.<br>**Why** – Ensures only high-risk items require human review while maintaining full traceability and timely communication. |
| Notify Legal Team | Slack | Alerts legal team and provides resume webhook URL | Wait for Legal Review | Log to Compliance Tracker | ## Route by Review Status, Notify, Log & Alert<br><br>**What** – Branch based on auto-approval or escalation; log outcomes in Google Sheets, alert legal via Slack, and email stakeholders.<br>**Why** – Ensures only high-risk items require human review while maintaining full traceability and timely communication. |
| Log to Compliance Tracker | Google Sheets | Logs reviewed items with audit fields | Notify Legal Team | Notify Stakeholders | ## Route by Review Status, Notify, Log & Alert<br><br>**What** – Branch based on auto-approval or escalation; log outcomes in Google Sheets, alert legal via Slack, and email stakeholders.<br>**Why** – Ensures only high-risk items require human review while maintaining full traceability and timely communication. |
| Notify Stakeholders | Slack | Posts compliance update to stakeholders | Log to Compliance Tracker | — | ## Route by Review Status, Notify, Log & Alert<br><br>**What** – Branch based on auto-approval or escalation; log outcomes in Google Sheets, alert legal via Slack, and email stakeholders.<br>**Why** – Ensures only high-risk items require human review while maintaining full traceability and timely communication. |
| Log Auto-Approved | Google Sheets | Logs auto-approved items | Route by Review Status | — | ## Route by Review Status, Notify, Log & Alert<br><br>**What** – Branch based on auto-approval or escalation; log outcomes in Google Sheets, alert legal via Slack, and email stakeholders.<br>**Why** – Ensures only high-risk items require human review while maintaining full traceability and timely communication. |
| Sticky Note | Sticky Note | Embedded notes (prereqs/use cases/customization/benefits) | — | — |  |
| Sticky Note1 | Sticky Note | Embedded setup steps note | — | — |  |
| Sticky Note2 | Sticky Note | Embedded “How It Works” description | — | — |  |
| Sticky Note3 | Sticky Note | Embedded block description (ingestion) | — | — |  |
| Sticky Note4 | Sticky Note | Embedded block description (policy interpretation) | — | — |  |
| Sticky Note5 | Sticky Note | Embedded block description (parallel governance) | — | — |  |
| Sticky Note6 | Sticky Note | Embedded block description (routing/logging/alerts) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook node**
   - Type: **Webhook**
   - Method: **POST**
   - Path: `legislative-policy-submission`
   - Response: **Last node**
   - Expected body: JSON containing `documentUrl` (string).

2) **Create Set node “Workflow Configuration”**
   - Add fields:
     - `legalReviewThreshold` = `HIGH`
     - `complianceFrameworks` = `GDPR,CCPA,SOX`
     - `stakeholderChannels` = `#legal-compliance,#governance`
   - Enable **Include Other Fields**.

3) **Connect:** Webhook → Set

4) **Create HTTP Request node “Fetch Legislative Document”**
   - URL: `{{$json.documentUrl}}`
   - Response: **File** (binary)
   - Connect: Set → HTTP Request

5) **Create Extract From File node “Extract Document Text”**
   - Operation: **PDF**
   - Join pages: **true**
   - Connect: HTTP Request → Extract From File

6) **Create OpenAI Chat Model node “OpenAI Model - Policy Interpretation”**
   - Provider: OpenAI
   - Model: `gpt-4o`
   - Temperature: `0.1`
   - Configure **OpenAI API credential** in n8n (API key).
   - (No main connection; it connects via AI port to the agent.)

7) **Create Structured Output Parser “Policy Analysis Output Parser”**
   - Paste/define schema with keys:
     `documentType, jurisdiction, effectiveDate, scope, keyProvisions, affectedEntities, complianceRequirements, riskLevel, requiresLegalReview, reasoning`.

8) **Create Agent “Policy Interpretation Agent”**
   - Text/input: `{{$json.text}}`
   - System message: policy interpretation instructions (as in workflow)
   - Enable **Output Parser**
   - Connect AI ports:
     - `ai_languageModel` ← OpenAI Model - Policy Interpretation
     - `ai_outputParser` ← Policy Analysis Output Parser
   - Main connect: Extract Document Text → Policy Interpretation Agent

9) **Create OpenAI Chat Model node “OpenAI Model - Governance Orchestration”**
   - Model: `gpt-4o`, temperature `0.2`
   - Use same OpenAI credential (or another).

10) **Create Structured Output Parser “Governance Orchestration Output Parser”**
   - Schema includes:
     - `orchestrationDecision`, `reviewStatus`, `coordinatedActions`, `impactSummary`, `complianceSummary`, `stakeholderSummary`, `nextSteps`, `reasoning`.

11) **Create three specialist Agent Tool nodes**
   - **Impact Assessment Agent Tool**
     - Text: `{{$fromAI("policyAnalysis", "...", "json")}}`
     - System message: impact assessment responsibilities
     - Enable output parser
     - Tool description: impact analysis
   - **Compliance Mapping Agent Tool**
     - Text: `{{$fromAI("policyAnalysis", "Policy interpretation results", "json")}}`
     - System message: framework mapping responsibilities
     - Enable output parser
   - **Stakeholder Notification Agent Tool**
     - Text: `{{$fromAI("policyAnalysis", "Policy interpretation results", "json")}}`
     - System message: notification responsibilities
     - Enable output parser

12) **For each tool: create its OpenAI model + output parser**
   - Impact:
     - Model `gpt-4o` temp `0.2`
     - Output parser schema includes `impactAreas, estimatedCost, implementationTimeline, ... priorityScore`
   - Compliance:
     - Model `gpt-4o` temp `0.2`
     - Output parser schema includes `applicableFrameworks, complianceGaps, requiredActions, complianceDeadlines, ...`
   - Stakeholder:
     - Model `gpt-4o` temp `0.3`
     - Output parser schema includes `stakeholderGroups, notificationPriority, messageContent, ...`

13) **Wire AI ports for each tool**
   - Each tool’s `ai_languageModel` ← its OpenAI Model node
   - Each tool’s `ai_outputParser` ← its Output Parser node

14) **Create Agent “Governance Orchestration Agent”**
   - Text: `Policy Analysis Results: {{ $json.output }}`
   - System message: orchestration responsibilities (call the three tools + decide)
   - Enable output parser
   - Wire:
     - `ai_languageModel` ← OpenAI Model - Governance Orchestration
     - `ai_outputParser` ← Governance Orchestration Output Parser
     - `ai_tool` connections ← (Impact Tool, Compliance Tool, Stakeholder Tool)
   - Main connect: Policy Interpretation Agent → Governance Orchestration Agent

15) **Create Switch node “Route by Review Status”**
   - Rule 1 (rename output “Legal Review Required”):
     - Left: `{{$json.output.reviewStatus}}`
     - Equals: `REQUIRES_HUMAN_REVIEW`
   - Rule 2 (rename output “Auto-Approved”):
     - Left: `{{$json.output.reviewStatus}}`
     - Equals: `APPROVED`
   - Fallback output name: `Unprocessed`
   - Connect: Governance Orchestration Agent → Switch

16) **Create Google Sheets node “Log Auto-Approved”**
   - Auth: **Google Sheets OAuth2** credential
   - Operation: **Append or Update**
   - Spreadsheet ID: your auto-approved tracker spreadsheet
   - Sheet name: your tab name
   - Map columns:
     - `status = "AUTO_APPROVED"`
     - `documentType, jurisdiction, effectiveDate, riskLevel` from Policy Interpretation Agent output
     - `timestamp = {{$now.toISO()}}`
   - Matching column: ideally a unique ID (but to match the workflow, use `documentType`)
   - Connect: Switch (“Auto-Approved”) → Log Auto-Approved

17) **Create Wait node “Wait for Legal Review”**
   - Resume: **Webhook**
   - Method: **POST**
   - Connect: Switch (“Legal Review Required”) → Wait

18) **Create Slack node “Notify Legal Team”**
   - Auth: **Slack OAuth2** credential (bot/user token with `chat:write`)
   - Channel: set **Legal Team channel ID**
   - Message: include policy fields + orchestration summaries + `{{$resumeWebhookUrl}}`
   - Connect: Wait → Notify Legal Team

19) **Create Google Sheets node “Log to Compliance Tracker”**
   - Auth: Google Sheets OAuth2
   - Operation: Append or Update
   - Spreadsheet ID: compliance tracker spreadsheet
   - Sheet name: tracker tab
   - Columns:
     - `documentType, jurisdiction, effectiveDate, riskLevel` from Policy Interpretation Agent
     - `reviewStatus` from Governance Orchestration Agent
     - `timestamp = {{$now.toISO()}}`
   - Connect: Notify Legal Team → Log to Compliance Tracker

20) **Create Slack node “Notify Stakeholders”**
   - Auth: Slack OAuth2
   - Channel: stakeholder channel ID
   - Message: coordinated actions + stakeholder summary
   - Connect: Log to Compliance Tracker → Notify Stakeholders

21) **Validate end-to-end**
   - POST to webhook with `{ "documentUrl": "https://..." }`
   - Confirm PDF text extraction, structured parsers succeed, and routing behaves as expected.

**Credential requirements**
- OpenAI: API key with access to `gpt-4o`.
- Slack OAuth2: workspace install; bot in target channels; scopes sufficient to post messages.
- Google Sheets OAuth2: access to target spreadsheets; correct sheet/tab names.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites**: OpenAI API key, Google Sheets with OAuth2, Slack workspace with bot token | Sticky note content |
| **Use Cases**: Regulatory change management, GDPR/financial compliance monitoring, policy impact assessment | Sticky note content |
| **Customization**: Swap OpenAI for NVIDIA NIM models, add additional specialist agents | Sticky note content |
| **Benefits**: Cuts manual compliance review time by 70%, ensures no legislation goes unassessed | Sticky note content |
| **Setup Steps** (as written): includes “Set up Gmail/SMTP credentials in Notify Stakeholders node” | Sticky Note1 — note: in this workflow, **Notify Stakeholders is a Slack node**, not Gmail/SMTP. Adjust the note or swap node type if email is desired. |
| **How It Works**: multi-agent analysis, orchestration, routing, Sheets logging, Slack alerts | Sticky Note2 |
| Routing block says “email stakeholders” | Sticky Note6 — current implementation uses **Slack**, not email. Consider adding Email/Gmail node if needed. |