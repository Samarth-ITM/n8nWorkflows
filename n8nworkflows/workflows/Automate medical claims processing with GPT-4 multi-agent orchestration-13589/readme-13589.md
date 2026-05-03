Automate medical claims processing with GPT-4 multi-agent orchestration

https://n8nworkflows.xyz/workflows/automate-medical-claims-processing-with-gpt-4-multi-agent-orchestration-13589


# Automate medical claims processing with GPT-4 multi-agent orchestration

## 1. Workflow Overview

**Title (given):** Automate medical claims processing with GPT-4 multi-agent orchestration  
**Workflow name (in n8n):** AI-powered medical claims processing with multi-agent orchestration

This workflow runs on a schedule and processes batches of medical claims (simulated EHR/billing data in this version). It uses a **central “Orchestrator Agent”** (GPT-4.1-mini via LangChain in n8n) that can call four specialist agent tools—**Coding Validation**, **Claims Submission**, **Denial Detection**, and **Payer Follow-up**—and returns a structured decision summary. The output is then **routed by risk level**, **escalated by email** when compliance flags or escalation are required, **stored in Data Tables** (audit/claims/denials), and finally **merged** to calculate metrics and format a final report.

### 1.1 Scheduled Trigger & Global Configuration
Runs daily at a configured hour and sets operational constants (emails, thresholds, retry limits).

### 1.2 Claim Intake (Simulated Source)
Generates 10 realistic-looking claims with demographics, coding, payer, amounts, and status fields.

### 1.3 Multi-Agent Orchestration (LLM + Tools + Structured Parsing)
A primary agent coordinates four specialist tools, each backed by its own OpenAI chat model and structured output parser.

### 1.4 Risk Routing, Compliance Gate, and Escalation
Routes orchestrator outputs into storage paths and triggers email escalation based on compliance/escalation conditions.

### 1.5 Persistence, Metrics, and Final Report
Writes audit/claims/denials records, merges outputs, computes operational metrics, and formats a final report object.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger & Workflow Configuration

**Overview:** Starts the workflow on a daily schedule and sets global parameters used later (notably escalation email).  
**Nodes involved:**  
- Schedule Trigger - Process Claims  
- Workflow Configuration

#### Node: Schedule Trigger - Process Claims
- **Type / role:** `Schedule Trigger` — entry point, time-based execution.
- **Key configuration:** Runs daily at **02:00** (`triggerAtHour: 2`).
- **Inputs/outputs:** No input; outputs to **Workflow Configuration**.
- **Edge cases / failures:**
  - Timezone considerations (n8n instance timezone). If you expect “2 AM local time”, verify instance timezone settings.
  - Missed executions if n8n is down at trigger time.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines configurable constants.
- **Key configuration (interpreted):**
  - `revenueSpecialistEmail`: placeholder to be replaced with a real mailbox.
  - `complianceThreshold`: 0.85 (currently not directly referenced downstream).
  - `highRiskThreshold`: 0.75 (currently not directly referenced downstream).
  - `denialRetryLimit`: 3 (currently not directly referenced downstream).
  - **Include other fields:** enabled (passes through incoming data, though trigger provides none).
- **Inputs/outputs:** From trigger → to **Simulate EHR/Billing Data**.
- **Edge cases / failures:**
  - Placeholders not replaced cause escalation emails to fail (invalid recipient / sender).
  - Thresholds are defined but not used in expressions; if intended, risk scoring logic must be implemented in the agent prompts or post-processing.

---

### Block 2 — Claim Intake (Simulated EHR/Billing Data)

**Overview:** Produces a list of 10 sample claim objects to mimic EHR/billing system output.  
**Nodes involved:**  
- Simulate EHR/Billing Data

#### Node: Simulate EHR/Billing Data
- **Type / role:** `Code` — generates mock claim records.
- **Key behavior:**
  - Creates 10 items shaped like:
    - `claimId`
    - `patientDemographics` (ID, name, DOB, address, etc.)
    - `diagnosisInfo` (ICD-10 code + description)
    - `procedureInfo` (CPT code + description + service date)
    - `billingInfo` (billed/allowed/payment/responsibility)
    - `insuranceInfo` (payer, payerId, policy/group)
    - `claimStatus` (status, dates, denialReason if denied, riskLevel, complianceFlag)
  - Returns `claims.map(claim => ({ json: claim }))` so downstream gets 10 items.
- **Inputs/outputs:** From **Workflow Configuration** → to **Orchestrator Agent**.
- **Edge cases / failures:**
  - Random values can produce inconsistent scenarios (e.g., “approved” with complianceFlag true). If you need realistic distributions, adjust generation logic.
  - Dates are hard-coded to year 2024 for service dates; adjust if using live metrics based on “now”.

---

### Block 3 — Multi-Agent Orchestration (Orchestrator + Specialist Tools + Structured Parsers + Models)

**Overview:** A central agent receives a claim and (optionally) calls specialist tools. Each tool uses its own OpenAI model and returns structured JSON enforced by output parsers. The orchestrator then emits a structured summary used for routing and escalation.  
**Nodes involved:**  
- Orchestrator Agent  
- Orchestrator Output Parser  
- Coding Validation Agent Tool  
- Coding Validation Output Parser  
- Claims Submission Agent Tool  
- Claims Submission Output Parser  
- Denial Detection Agent Tool  
- Denial Detection Output Parser  
- Payer Follow-up Agent Tool  
- Payer Follow-up Output Parser  
- OpenAI Model - Orchestrator  
- OpenAI Model - Coding Validation  
- OpenAI Model - Claims Submission  
- OpenAI Model - Denial Detection  
- OpenAI Model - Payer Follow-up

#### Node: OpenAI Model - Orchestrator
- **Type / role:** `lmChatOpenAi` — LLM provider for the Orchestrator Agent.
- **Key configuration:** Model `gpt-4.1-mini`; uses OpenAI credential “OpenAi account”.
- **Connections:** Supplies language model to **Orchestrator Agent** via `ai_languageModel`.
- **Edge cases / failures:** OpenAI auth errors, rate limits, model availability, cost spikes with large inputs.

#### Node: Orchestrator Output Parser
- **Type / role:** `outputParserStructured` — forces orchestrator output into a schema.
- **Schema highlights (required fields):**
  - `orchestrationSummary` (string)
  - `agentsCalled` (string[])
  - `riskLevel` enum: LOW/MEDIUM/HIGH/CRITICAL
  - `complianceFlags` (string[])
  - `requiresEscalation` (boolean)
  - `nextActions` (string[])
  - `financialImpact` (number)
- **Connections:** Attached to **Orchestrator Agent** via `ai_outputParser`.
- **Edge cases / failures:** Model output that doesn’t match schema causes parser failure. The workflow currently has no error branch/try-catch around agent execution.

#### Node: Orchestrator Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — central coordinator that can call tools.
- **Key configuration:**
  - Input text: `={{ $json }}` (the whole claim object serialized).
  - System message: defines strict responsibilities and “critical rules” (no diagnosis inference, audit trails, HIPAA/CMS compliance, escalation).
  - `promptType: define`
  - `hasOutputParser: true` (paired with Orchestrator Output Parser).
- **Tooling connections:** Can call the four Agent Tools via `ai_tool`.
- **Main output connection:** To **Route by Risk Level**.
- **Edge cases / failures:**
  - Because input is the full claim JSON, token usage can rise if the claim structure grows (in real EHR payloads).
  - The orchestrator is instructed to “maintain audit trails” and “coordinate retries”, but no deterministic retry logic exists in nodes; it relies on the model to describe actions, not execute resubmissions.
  - The downstream nodes expect certain fields like `riskLevel`, `complianceFlags`, `requiresEscalation`. If the agent fails schema parsing, the run fails.

---

#### Node: OpenAI Model - Coding Validation
- **Type / role:** LLM for Coding Validation tool (`gpt-4.1-mini`).
- **Connection:** `ai_languageModel` → **Coding Validation Agent Tool**.

#### Node: Coding Validation Output Parser
- **Type / role:** Structured parser for coding validation response.
- **Schema highlights:**
  - `validationStatus`: VALID / INVALID / NEEDS_REVIEW
  - `icd10Codes[]` and `cptCodes[]` with `valid` and `issues[]`
  - `complianceRisks[]`, `recommendations[]`
- **Connection:** `ai_outputParser` → **Coding Validation Agent Tool**.

#### Node: Coding Validation Agent Tool
- **Type / role:** `agentTool` — specialist tool callable by the Orchestrator.
- **Key configuration:**
  - Tool input text uses: `={{ $fromAI("claimData", "Claim data requiring coding validation", "json") }}`
  - System message: CMS/LCD/NCD guidance, bundling/modifiers checks, no diagnosis inference, audit reasoning.
  - Has output parser enabled.
  - Tool description included (helps orchestrator decide to call it).
- **Connections:** Tool + model + output parser connect to orchestrator/tooling network.
- **Edge cases / failures:**
  - `$fromAI(...)` requires orchestrator/tool calling context; if this tool is executed outside an agent call, it will not have the expected AI-provided variables.
  - Model may “hallucinate” coding rules; for production you’d typically add deterministic code-set validation via APIs.

---

#### Node: OpenAI Model - Claims Submission
- **Type / role:** LLM for Claims Submission tool (`gpt-4.1-mini`).
- **Connection:** `ai_languageModel` → **Claims Submission Agent Tool**.

#### Node: Claims Submission Output Parser
- **Type / role:** Structured parser for submission response.
- **Schema highlights:**
  - `submissionStatus`: SUBMITTED / REJECTED / PENDING / ERROR
  - `confirmationNumber`, `payerName`, `submissionDate`
  - `errors[]`, `correctionNeeded` (boolean), `estimatedPayment` (number)

#### Node: Claims Submission Agent Tool
- **Type / role:** `agentTool` — prepares/submits claims (conceptually).
- **Key configuration:**
  - Input: `={{ $fromAI("validatedClaim", "Validated claim data ready for submission", "json") }}`
  - System message: CMS-1500/UB-04 preparation, payer rules, audit trails.
  - Structured output parser enabled.
- **Edge cases / failures:**
  - This node does not actually integrate with EDI/clearinghouse—submission is “simulated” by the LLM unless you add real integrations.
  - If you later add real payer submission APIs, split this into deterministic steps plus LLM for narrative/triage only.

---

#### Node: OpenAI Model - Denial Detection
- **Type / role:** LLM for Denial Detection tool (`gpt-4.1-mini`).
- **Connection:** `ai_languageModel` → **Denial Detection Agent Tool**.

#### Node: Denial Detection Output Parser
- **Type / role:** Structured parser for denial analysis.
- **Schema highlights:**
  - `denialDetected` (boolean)
  - `denialReason`, `denialCode`
  - `denialCategory` enum (CODING_ERROR, MEDICAL_NECESSITY, etc.)
  - `appealViable`, `retryRecommended` (booleans)
  - `correctionStrategy`, `financialImpact` (number)

#### Node: Denial Detection Agent Tool
- **Type / role:** `agentTool` — analyzes denials and strategies.
- **Key configuration:**
  - Input: `={{ $fromAI("submittedClaim", "Submitted claim data for denial analysis", "json") }}`
  - System message references 835/EOB analysis and trend detection.
- **Edge cases / failures:**
  - In this workflow there is no actual 835/EOB ingestion; the tool can only infer from provided data.
  - For real implementations, feed payer response payloads into the orchestrator/tool chain.

---

#### Node: OpenAI Model - Payer Follow-up
- **Type / role:** LLM for Payer Follow-up tool (`gpt-4.1-mini`).
- **Connection:** `ai_languageModel` → **Payer Follow-up Agent Tool**.

#### Node: Payer Follow-up Output Parser
- **Type / role:** Structured parser for follow-up plan.
- **Schema highlights:**
  - `followUpRequired` (boolean)
  - `followUpChannel` enum (PHONE/PORTAL/EMAIL/FAX)
  - `payerContactInfo` (string)
  - `claimAge` (number), `priority` enum, `escalationNeeded` (boolean)
  - `actionPlan[]`

#### Node: Payer Follow-up Agent Tool
- **Type / role:** `agentTool` — produces follow-up steps and escalation plan.
- **Key configuration:**
  - Input: `={{ $fromAI("denialData", "Denial data requiring payer follow-up", "json") }}`
  - System message: timely filing, logging communications, prioritization by aging/value.
- **Edge cases / failures:**
  - Without real claim age calculation and payer contact directory, outputs are heuristic.

---

### Block 4 — Risk Routing, Compliance Gate, and Escalation

**Overview:** Uses orchestrator risk level to route outputs into storage flows, then checks compliance flags/escalation and emails a revenue specialist if needed.  
**Nodes involved:**  
- Route by Risk Level  
- Check Compliance Flags  
- Escalate to Revenue Specialist

#### Node: Route by Risk Level
- **Type / role:** `Switch` — routes items by `$json.riskLevel`.
- **Rules (interpreted):**
  - Output **High Risk**: `riskLevel` equals HIGH or CRITICAL
  - Output **Standard Claims**: `riskLevel` equals MEDIUM
  - Output **Denials**: `riskLevel` equals LOW
  - Fallback: **Unclassified**
- **Connections:**
  - High Risk → **Check Compliance Flags**
  - Standard Claims → **Store Claims Records**
  - Denials → **Store Denial Records**
- **Edge cases / failures:**
  - The naming “Denials = LOW risk” is conceptually odd; if you intended denial items to be routed by a denial flag, adjust rules.
  - If orchestrator outputs lowercase risk (“high”) parser would block it; schema enforces uppercase values.

#### Node: Check Compliance Flags
- **Type / role:** `If` — decides whether to escalate.
- **Condition logic (OR):**
  - `length(complianceFlags) > 0` using: `={{ $('Route by Risk Level').item.json.complianceFlags }}`
  - OR `requiresEscalation == true` using: `={{ $('Route by Risk Level').item.json.requiresEscalation }}`
- **Connections:**
  - True branch → **Escalate to Revenue Specialist**
  - False branch → **Store Audit Trail**
- **Edge cases / failures:**
  - The expressions reference `$('Route by Risk Level').item.json...` instead of `$json...`. This can break if item linkage changes or if run context differs. Prefer `$json.complianceFlags` and `$json.requiresEscalation` because the IF node receives the switch output item directly.
  - If `complianceFlags` is missing/null, `lengthGt` may error depending on runtime coercion.

#### Node: Escalate to Revenue Specialist
- **Type / role:** `Email Send` — human escalation for high-risk/compliance cases.
- **Key configuration:**
  - To: `={{ $('Workflow Configuration').first().json.revenueSpecialistEmail }}`
  - From: placeholder `<__PLACEHOLDER_VALUE__Sender email address__>`
  - Subject: `ESCALATION: Revenue Cycle Compliance Issue - {{ $json.output.riskLevel }} Risk`
  - HTML body: rich formatted message with many references like `$json.output.*`
- **Connections:** After sending → **Merge Results** (index 1).
- **Edge cases / failures (important):**
  - **Data shape mismatch:** The orchestrator parser outputs `riskLevel`, `complianceFlags`, etc. at the top-level of `$json`, but the email template references `$json.output.riskLevel`, `$json.output.complianceFlags` (array of objects with `type/description/severity`), etc. Unless another node wraps data under `output`, the template will render many `N/A` values.
  - Credential/auth failures (SMTP/Gmail). This node requires configured email credentials in n8n.
  - HTML includes a “🚨” icon; if your instruction set disallows icons in subject/body for deliverability, remove it.

---

### Block 5 — Persistence (Data Tables), Merge, Metrics, Report

**Overview:** Writes records to three n8n Data Tables, merges outputs from different branches, calculates aggregate metrics, and formats a final report.  
**Nodes involved:**  
- Store Audit Trail  
- Store Claims Records  
- Store Denial Records  
- Merge Results  
- Calculate Metrics  
- Format Final Report

#### Node: Store Audit Trail
- **Type / role:** `Data Table` — inserts audit events into `RevenueCycleAuditTrail`.
- **Columns mapping (expects these fields):**
  - `status`, `claim_id`, `patient_id`, `risk_level`, `total_amount`, `actions_taken`, `agents_called`, `compliance_flags`, plus `timestamp = $now.toISO()`
- **Connections:** To **Merge Results** (index 0).
- **Edge cases / failures (important):**
  - **Field mismatch:** Upstream orchestrator output uses `riskLevel` (camelCase) and `claimId`; simulated data uses nested structures. This node expects snake_case (`claim_id`, `risk_level`) and flat fields that likely do not exist → many null/empty inserts or expression errors depending on Data Table strictness.
  - DataTable permissions/availability: the Data Table must exist (`RevenueCycleAuditTrail`).

#### Node: Store Claims Records
- **Type / role:** `Data Table` — inserts claim submission records into `ClaimsRecords`.
- **Expected fields:** `payer`, `claim_id`, `patient_id`, `patient_name`, `submission_date`, `estimated_payment`, `submission_status`, `confirmation_number`
- **Connections:** To **Merge Results** (index 0).
- **Edge cases / failures:**
  - No node maps orchestrator output into these fields; likely empty unless the orchestrator itself outputs them at top-level.
  - Data table must exist.

#### Node: Store Denial Records
- **Type / role:** `Data Table` — inserts denial info into `DenialRecords`.
- **Expected fields:** `claim_id`, `denial_code`, `denial_reason`, `denial_category`, `appeal_viability`, `financial_impact`, `correction_strategy`
- **Connections:** To **Merge Results** (index 1).
- **Edge cases / failures:**
  - Similar field mismatches: Denial output parser uses `appealViable`, not `appeal_viability`; orchestrator output may not expose denial fields.
  - Data table must exist.

#### Node: Merge Results
- **Type / role:** `Merge` — combines multiple branch outputs.
- **Mode:** `combine` with `combineByPosition`.
- **Inputs:**
  - Input 1: from **Store Audit Trail** and **Store Claims Records** (both go to index 0)
  - Input 2: from **Store Denial Records** and **Escalate to Revenue Specialist** (both go to index 1)
- **Output:** To **Calculate Metrics**
- **Edge cases / failures:**
  - `combineByPosition` assumes both inputs have items aligned by index. Here, the number of items per branch can differ drastically (e.g., only some claims are high risk). This can lead to partial merges, unexpected null pairing, or dropped items.

#### Node: Calculate Metrics
- **Type / role:** `Code` — aggregates operational KPIs across merged items.
- **Logic (interpreted):**
  - Counts claims if `claimId` or `claim_id` exists.
  - Counts denied if `status === 'denied'` or `denial_detected === true` or `isDenied === true`
  - Sums claim value from `claimAmount` / `claim_amount` / `amount`
  - Compliance issues if `complianceFlag` or variants are true
  - Escalations if `escalated` or `requiresEscalation` true
  - Returns a single item with `metrics` and `summary`.
- **Connections:** To **Format Final Report**.
- **Edge cases / failures:**
  - Because upstream data is inconsistent (camelCase vs snake_case; nested objects), many counters may remain zero unless normalized first.
  - `denialRate` is computed as a string via `toFixed(2)` then compared as number later in `performanceStatus` logic; JS coercion can be surprising. (It works here because comparisons coerce, but better to keep as number.)

#### Node: Format Final Report
- **Type / role:** `Set` — final report packaging.
- **Fields set:**
  - `reportTitle`: “Revenue Cycle Processing Report”
  - `reportDate`: `={{ $now.toISO() }}`
  - `summary`: concatenates:
    - `Metrics: ` + JSON.stringify(`$('Calculate Metrics').item.json`)
    - ` | Orchestration: ` + JSON.stringify(`$('Orchestrator Agent').item.json`)
  - `status`: “COMPLETED”
  - Includes other fields (passes through).
- **Edge cases / failures:**
  - Uses `$('Orchestrator Agent').item.json` which refers to *a specific item*; with multiple claims, this may reference only one orchestrator item (depending on execution context) rather than the whole batch.
  - The metrics node returns a single item; orchestration produces many items—this mismatch can cause confusing summaries.

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger - Process Claims | Schedule Trigger | Starts scheduled batch run | — | Workflow Configuration | ## How It Works… (multi-agent orchestration description) |
| Workflow Configuration | Set | Stores global parameters (email, thresholds) | Schedule Trigger - Process Claims | Simulate EHR/Billing Data | ## How It Works… (multi-agent orchestration description) |
| Simulate EHR/Billing Data | Code | Generates mock EHR/billing claims | Workflow Configuration | Orchestrator Agent | ## Simulate Billing Data **What:** Generates or loads EHR/billing records for processing. **Why:** Provides structured claim input for downstream AI agents. |
| Orchestrator Agent | LangChain Agent | Coordinates specialist tools and outputs structured orchestration result | Simulate EHR/Billing Data | Route by Risk Level | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Coding Validation Agent Tool | LangChain Agent Tool | Validates ICD/CPT/HCPCS and compliance risks | (tool-called by Orchestrator Agent) | (returns to Orchestrator Agent) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Claims Submission Agent Tool | LangChain Agent Tool | Prepares/submits claims (conceptually) | (tool-called by Orchestrator Agent) | (returns to Orchestrator Agent) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Denial Detection Agent Tool | LangChain Agent Tool | Detects denials and recommends fixes/appeals | (tool-called by Orchestrator Agent) | (returns to Orchestrator Agent) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Payer Follow-up Agent Tool | LangChain Agent Tool | Produces follow-up/escalation plan | (tool-called by Orchestrator Agent) | (returns to Orchestrator Agent) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Orchestrator Output Parser | Structured Output Parser | Enforces orchestrator JSON schema | — (AI connection) | Orchestrator Agent (ai_outputParser) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Coding Validation Output Parser | Structured Output Parser | Enforces coding tool schema | — (AI connection) | Coding Validation Agent Tool (ai_outputParser) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Claims Submission Output Parser | Structured Output Parser | Enforces submission tool schema | — (AI connection) | Claims Submission Agent Tool (ai_outputParser) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Denial Detection Output Parser | Structured Output Parser | Enforces denial tool schema | — (AI connection) | Denial Detection Agent Tool (ai_outputParser) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Payer Follow-up Output Parser | Structured Output Parser | Enforces follow-up tool schema | — (AI connection) | Payer Follow-up Agent Tool (ai_outputParser) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| OpenAI Model - Orchestrator | OpenAI Chat Model | LLM for orchestrator | — (AI connection) | Orchestrator Agent (ai_languageModel) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| OpenAI Model - Coding Validation | OpenAI Chat Model | LLM for coding tool | — (AI connection) | Coding Validation Agent Tool (ai_languageModel) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| OpenAI Model - Claims Submission | OpenAI Chat Model | LLM for submission tool | — (AI connection) | Claims Submission Agent Tool (ai_languageModel) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| OpenAI Model - Denial Detection | OpenAI Chat Model | LLM for denial tool | — (AI connection) | Denial Detection Agent Tool (ai_languageModel) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| OpenAI Model - Payer Follow-up | OpenAI Chat Model | LLM for follow-up tool | — (AI connection) | Payer Follow-up Agent Tool (ai_languageModel) | ## Orchestrator Agent **What:** Central AI agent delegates tasks to four specialist sub-agents. **Why:** Enables parallel, specialized processing—improving accuracy and speed. |
| Route by Risk Level | Switch | Routes by orchestrator `riskLevel` | Orchestrator Agent | Check Compliance Flags; Store Claims Records; Store Denial Records | ## Risk Routing & Storage **What:** Claims are routed by risk level; stored in Claims, Denial, or Audit records. **Why:** Prioritizes high-risk cases and maintains full compliance audit trails. |
| Check Compliance Flags | IF | Escalates if compliance flags exist or escalation required | Route by Risk Level | Escalate to Revenue Specialist; Store Audit Trail | ## Risk Routing & Storage **What:** Claims are routed by risk level; stored in Claims, Denial, or Audit records. **Why:** Prioritizes high-risk cases and maintains full compliance audit trails. |
| Escalate to Revenue Specialist | Email Send | Sends escalation email to specialist | Check Compliance Flags (true) | Merge Results | ## Escalation & Reporting **What:** High-risk claims email Revenue Specialist; metrics calculated; final report generated. **Why:** Ensures human oversight on complex cases while automating routine outcomes. |
| Store Audit Trail | Data Table | Persists compliance/audit event | Check Compliance Flags (false) | Merge Results | ## Escalation & Reporting **What:** High-risk claims email Revenue Specialist; metrics calculated; final report generated. **Why:** Ensures human oversight on complex cases while automating routine outcomes. |
| Store Claims Records | Data Table | Persists claim submission records | Route by Risk Level (Standard Claims) | Merge Results | ## Risk Routing & Storage **What:** Claims are routed by risk level; stored in Claims, Denial, or Audit records. **Why:** Prioritizes high-risk cases and maintains full compliance audit trails. |
| Store Denial Records | Data Table | Persists denial records | Route by Risk Level (Denials) | Merge Results | ## Risk Routing & Storage **What:** Claims are routed by risk level; stored in Claims, Denial, or Audit records. **Why:** Prioritizes high-risk cases and maintains full compliance audit trails. |
| Merge Results | Merge | Combines branch outputs for metrics | Store Audit Trail / Store Claims Records; Store Denial Records / Escalate to Revenue Specialist | Calculate Metrics | ## Escalation & Reporting **What:** High-risk claims email Revenue Specialist; metrics calculated; final report generated. **Why:** Ensures human oversight on complex cases while automating routine outcomes. |
| Calculate Metrics | Code | Computes batch KPIs | Merge Results | Format Final Report | ## Escalation & Reporting **What:** High-risk claims email Revenue Specialist; metrics calculated; final report generated. **Why:** Ensures human oversight on complex cases while automating routine outcomes. |
| Format Final Report | Set | Produces final report object | Calculate Metrics | — | ## Escalation & Reporting **What:** High-risk claims email Revenue Specialist; metrics calculated; final report generated. **Why:** Ensures human oversight on complex cases while automating routine outcomes. |
| Sticky Note | Sticky Note | Comment | — | — | ## Prerequisites… (OpenAI key, Gmail/SMTP, use cases, customization, benefits) |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Setup Steps (import JSON, creds, schedule, config, email, sheets/db, test) |
| Sticky Note2 | Sticky Note | Comment | — | — | ## How It Works (full narrative) |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Escalation & Reporting (what/why) |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Orchestrator Agent (what/why) |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Simulate Billing Data (what/why) |
| Sticky Note6 | Sticky Note | Comment | — | — | ## Risk Routing & Storage (what/why) |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

1. **Create a new workflow** in n8n  
   - Name it: *AI-powered medical claims processing with multi-agent orchestration* (or your preferred name).

2. **Add Trigger**
   - Add node: **Schedule Trigger**
   - Configure: run daily at **02:00** (or your desired hour).
   - Connect → next node.

3. **Add “Workflow Configuration”**
   - Add node: **Set**
   - Add fields:
     - `revenueSpecialistEmail` (string): your escalation recipient email
     - `complianceThreshold` (number): `0.85`
     - `highRiskThreshold` (number): `0.75`
     - `denialRetryLimit` (number): `3`
   - Enable “Include Other Fields”.
   - Connect Schedule Trigger → Workflow Configuration.

4. **Add Claim Source (simulation)**
   - Add node: **Code**
   - Paste logic to generate claims (or replace with real EHR/billing ingestion such as HTTP Request / Database).
   - Ensure output is **one item per claim** (`return claims.map(...)`).
   - Connect Workflow Configuration → Code.

5. **Create the Orchestrator LLM node**
   - Add node: **OpenAI Chat Model** (LangChain)  
   - Select model: `gpt-4.1-mini`
   - Create/OpenAI credentials in n8n:
     - Credentials → **OpenAI API** → set API key
   - Keep default options unless you need temperature/tool settings.

6. **Add Orchestrator Output Parser**
   - Add node: **Structured Output Parser**
   - Schema type: **Manual**
   - Provide schema with required fields:
     - `orchestrationSummary`, `agentsCalled`, `riskLevel`, `complianceFlags`, `requiresEscalation`, `nextActions`, `financialImpact`.

7. **Add Orchestrator Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Input text: `{{ $json }}` (entire claim)
   - System message: paste orchestrator responsibilities and critical rules (from workflow).
   - Attach connections:
     - Connect **OpenAI Model - Orchestrator** → Orchestrator Agent via **ai_languageModel**
     - Connect **Orchestrator Output Parser** → Orchestrator Agent via **ai_outputParser**

8. **Create Specialist Tools (repeat for 4 tools)**
   For each tool, do:
   - Add node: **OpenAI Chat Model** with `gpt-4.1-mini` (can reuse same credential).
   - Add node: **Structured Output Parser** with the tool’s schema.
   - Add node: **AI Agent Tool** with:
     - A tool description (helps the orchestrator choose it)
     - The system message for that role
     - The input expression using `$fromAI(...)`:
       - Coding Validation Tool: `$fromAI("claimData", ..., "json")`
       - Claims Submission Tool: `$fromAI("validatedClaim", ..., "json")`
       - Denial Detection Tool: `$fromAI("submittedClaim", ..., "json")`
       - Payer Follow-up Tool: `$fromAI("denialData", ..., "json")`
   - Wire:
     - Model → Tool via `ai_languageModel`
     - Output Parser → Tool via `ai_outputParser`
     - Tool → Orchestrator Agent via `ai_tool`

9. **Risk routing**
   - Add node: **Switch**
   - Value to evaluate: `{{ $json.riskLevel }}`
   - Create outputs:
     - High Risk: equals HIGH or CRITICAL
     - Standard Claims: equals MEDIUM
     - Denials: equals LOW
   - Connect Orchestrator Agent → Switch.

10. **Compliance gate**
    - Add node: **IF**
    - Condition (OR):
      - `complianceFlags` length > 0
      - OR `requiresEscalation` equals true
    - Prefer expressions based on current item:
      - `{{ $json.complianceFlags }}` and `{{ $json.requiresEscalation }}`
    - Connect Switch “High Risk” → IF.

11. **Escalation email**
    - Add node: **Email Send**
    - Configure credentials: SMTP/Gmail (per your environment).
    - To: `{{ $('Workflow Configuration').first().json.revenueSpecialistEmail }}`
    - Subject and HTML body: paste your escalation template.
    - Connect IF (true) → Email Send.

12. **Create n8n Data Tables**
    - In n8n, create Data Tables:
      - `RevenueCycleAuditTrail`
      - `ClaimsRecords`
      - `DenialRecords`

13. **Store Audit Trail**
    - Add node: **Data Table**
    - Table: `RevenueCycleAuditTrail`
    - Map columns to your chosen canonical field names.
    - Connect IF (false) → Store Audit Trail.

14. **Store Claims Records / Denial Records**
    - Add node: **Data Table** → `ClaimsRecords`
      - Connect Switch “Standard Claims” → Store Claims Records.
    - Add node: **Data Table** → `DenialRecords`
      - Connect Switch “Denials” → Store Denial Records.

15. **Merge results**
    - Add node: **Merge**
    - Mode: **Combine**
    - Combine by: **Position**
    - Connect:
      - Store Audit Trail → Merge input 1
      - Store Claims Records → Merge input 1
      - Store Denial Records → Merge input 2
      - Escalation Email → Merge input 2

16. **Calculate metrics**
    - Add node: **Code**
    - Paste metrics aggregation logic (or implement your own).
    - Connect Merge → Calculate Metrics.

17. **Format final report**
    - Add node: **Set**
    - Set `reportTitle`, `reportDate`, `summary`, `status`
    - Connect Calculate Metrics → Format Final Report.

18. **(Recommended) Add normalization step**
    - Add a **Set** (or Code) node right after Orchestrator Agent to map outputs into consistent fields (`claim_id`, `patient_id`, etc.) before storage/email, to avoid mismatches.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** n8n, OpenAI API key (GPT-4) and Gmail or SMTP account | Sticky note “Prerequisites” |
| **Use Cases:** Hospital billing departments automating claims submission and denial follow-up | Sticky note “Prerequisites” |
| **Customization:** Swap OpenAI for NVIDIA NIM or Anthropic models in any agent node and add Slack alerts alongside email escalation | Sticky note “Prerequisites” |
| **Benefits:** Reduces manual claims review by 80%+ through parallel AI agent processing | Sticky note “Prerequisites” |
| **Setup Steps:** import workflow JSON; add OpenAI credentials; configure schedule; update configuration; set email credentials; connect Google Sheets or database; test with simulated data | Sticky note “Setup Steps” |
| **How It Works:** scheduled trigger → orchestrator → specialist agents → parsed/merged/routed → metrics/report | Sticky note “How It Works” |
| **Escalation & Reporting:** High-risk claims email specialist; metrics calculated; final report generated | Sticky note “Escalation & Reporting” |

--- 

### Key Implementation Warnings (cross-cutting)
- **Field-name mismatches** (camelCase vs snake_case; nested vs flat) will cause empty Data Table inserts and broken email templating unless you add a normalization/mapping node.
- **Merge by position** is fragile for branching flows with unequal item counts; consider merging by `claimId` instead (e.g., “Merge by Key”) after you normalize IDs.
- The “Claims Submission” and “Denial Detection” steps are **conceptual** unless you integrate real EDI/clearinghouse/payer response ingestion.