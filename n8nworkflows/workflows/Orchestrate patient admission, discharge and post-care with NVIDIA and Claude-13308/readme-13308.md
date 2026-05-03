Orchestrate patient admission, discharge and post-care with NVIDIA and Claude

https://n8nworkflows.xyz/workflows/orchestrate-patient-admission--discharge-and-post-care-with-nvidia-and-claude-13308


# Orchestrate patient admission, discharge and post-care with NVIDIA and Claude

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Orchestrate patient admission, discharge and post-care with NVIDIA and Claude  
**Workflow name (internal):** Healthcare Operations Orchestration Agent for Patient Care Continuity

**Purpose:**  
This workflow orchestrates **non-clinical** operational processes around a patient’s **admission, discharge, and post-care continuity**. It accepts either scheduled checks or inbound patient-operation events, fetches operational readiness data from multiple systems (EHR, pharmacy, case management—nursing is referenced but not actually fetched), validates completeness, asks a Claude-powered agent to output structured operational instructions, routes execution to the correct downstream operational workflow endpoint, escalates exceptions to humans (Slack + email), and logs an audit trail plus a compliance report.

**Target use cases:**
- Coordinating operational readiness and tasks (paperwork, scheduling, transport, referrals, documentation workflows)
- Automated exception escalation (missing data, contradictory signals, compliance concerns)
- Audit/compliance traceability for operational automation

### 1.1 Entry & Triggering
Two entry points:
- Time-based polling every 15 minutes
- Webhook-based event ingestion (EHR or other system)

### 1.2 Configuration & Context Setup
Centralized configuration is set once per execution (API base URLs, workflow endpoints, escalation channels/emails, audit log URL).

### 1.3 System Data Collection
Fetches patient operational status from EHR, then pharmacy readiness, then case management status (sequential HTTP calls).

### 1.4 Aggregation & Validation
Aggregates system outputs into a single item and runs completeness checks in a Code node.

### 1.5 AI Orchestration (Claude Agent + Structured Output)
A LangChain agent (with strict non-clinical constraints) produces a structured JSON decision including operation type, readiness, blockers, actions, and compliance metadata.

### 1.6 Routing & Execution
Routes by operation type (Admission/Discharge/Post-Care) and calls the corresponding workflow endpoint.

### 1.7 Exception Escalation, Audit Logging & Compliance Output
If human review is required, it escalates via Slack and email, merges all paths, logs an audit trail to an external system, then formats a compliance report object.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggering & Inbound Event Reception
**Overview:** Starts the workflow either on a schedule or via an authenticated webhook that receives patient operational events.

**Nodes involved:**
- Schedule Patient Operations Check
- Patient Event Webhook
- Merge Triggers

#### Node: Schedule Patient Operations Check
- **Type / role:** Schedule Trigger — periodic execution.
- **Configuration:** Runs every **15 minutes**.
- **Inputs/Outputs:** Entry node → outputs to **Merge Triggers (input 0)**.
- **Version requirements:** typeVersion **1.3**.
- **Edge cases / failures:** n8n scheduling disabled; timezone considerations; overlapping executions if runtime > 15 minutes (depends on instance settings).

#### Node: Patient Event Webhook
- **Type / role:** Webhook — receives POST events from an external system (e.g., EHR).
- **Configuration:**
  - **POST** `/patient-operations-event`
  - **Authentication:** `headerAuth` (HTTP Header Auth credential: **Chart-IMG**)
  - **Response mode:** `lastNode` (responds with output of final executed node)
- **Key variables:** expects payload to include a `patientId` (later used in EHR fetch: `$json.patientId || 'current'`).
- **Inputs/Outputs:** Entry node → outputs to **Merge Triggers (input 1)**.
- **Version requirements:** typeVersion **2.1**.
- **Edge cases / failures:** missing/invalid auth header; missing `patientId`; webhook response delayed by downstream steps (timeouts at caller); PHI risk if caller sends excessive data (should minimize).

#### Node: Merge Triggers
- **Type / role:** Merge — combines the two trigger paths into one.
- **Configuration:** Default merge behavior (no explicit mode shown).
- **Inputs/Outputs:** Receives schedule + webhook → outputs to **Workflow Configuration**.
- **Version requirements:** typeVersion **3.2**.
- **Edge cases / failures:** If merge mode expects paired items, may stall; in practice, for “either/or” triggers, consider “Pass-through” or “Append” modes explicitly to avoid unexpected behavior.

**Sticky note coverage (contextual):**
- **Sticky Note2 (How It Works)** and **Sticky Note5 (Data Collection & AI Risk Analysis)** conceptually describe this part of the workflow, but they mention “NVIDIA models” and “risk assessment” which do not exist as nodes here (see “General Notes” for mismatch).

---

### Block 2.2 — Workflow Configuration (Central Parameters)
**Overview:** Defines all base URLs, workflow endpoints, escalation targets, and audit endpoints used across the workflow.

**Nodes involved:**
- Workflow Configuration

#### Node: Workflow Configuration
- **Type / role:** Set — creates configuration fields used by expressions in other nodes.
- **Configuration choices:**
  - Sets placeholders for:
    - `ehrApiUrl`, `nursingApiUrl`, `pharmacyApiUrl`, `caseManagementApiUrl`
    - `admissionWorkflowUrl`, `dischargeWorkflowUrl`, `postCareWorkflowUrl`
    - `auditLogUrl`
    - `escalationSlackChannel`, `escalationEmail`, `complianceOfficerEmail`
  - `includeOtherFields: true` (preserves inbound data like `patientId`)
- **Inputs/Outputs:** From **Merge Triggers** → to **Fetch EHR Patient Data**.
- **Version requirements:** typeVersion **3.4**.
- **Edge cases / failures:** Leaving placeholders unchanged will break all dependent HTTP/Slack/Email nodes; misconfigured Slack channel ID will fail escalation.

**Sticky note coverage:**
- **Sticky Note1 (Setup Steps)** partially applies but references Gmail/Sheets and NVIDIA nodes not present; still relevant for “Webhook setup” and “Claude Model credentials” conceptually.

---

### Block 2.3 — System Data Collection (EHR → Pharmacy → Case Management)
**Overview:** Sequentially pulls patient operational readiness signals from multiple systems using HTTP requests and a shared request correlation ID.

**Nodes involved:**
- Fetch EHR Patient Data
- Fetch Pharmacy Readiness
- Fetch Case Management Data

#### Node: Fetch EHR Patient Data
- **Type / role:** HTTP Request — fetches operational status from EHR.
- **Configuration choices:**
  - URL:
    - `{{ ehrApiUrl }}/patient/{{ $json.patientId || 'current' }}/operational-status`
  - Headers:
    - `Authorization: <EHR token placeholder>`
    - `X-Request-ID: {{ $execution.id }}`
  - Timeout: **30,000 ms**
- **Inputs/Outputs:** From **Workflow Configuration** → to **Fetch Pharmacy Readiness**.
- **Version requirements:** typeVersion **4.3**.
- **Edge cases / failures:** 401/403 auth; patientId not found; EHR API latency > 30s; payload shape mismatch downstream (expected `patientId` in response is used later).

#### Node: Fetch Pharmacy Readiness
- **Type / role:** HTTP Request — fetches medication readiness signals.
- **Configuration choices:**
  - URL:
    - `{{ pharmacyApiUrl }}/patient/{{ $('Fetch EHR Patient Data').first().json.patientId }}/medication-readiness`
  - Headers:
    - `Authorization: <Pharmacy token placeholder>`
    - `X-Request-ID: {{ $execution.id }}`
  - Timeout: **30,000 ms**
- **Inputs/Outputs:** From **Fetch EHR Patient Data** → to **Fetch Case Management Data**.
- **Version requirements:** typeVersion **4.3**.
- **Edge cases / failures:** If EHR response lacks `patientId`, expression fails or becomes `undefined`; 401/403; pharmacy endpoint mismatch.

#### Node: Fetch Case Management Data
- **Type / role:** HTTP Request — fetches case/discharge planning status.
- **Configuration choices:**
  - URL (note the probable bug):
    - `{{ caseManagementApiUrl }}/patient/={{ patientId }}/case-status`
  - Headers:
    - `Authorization: <Case Management token placeholder>`
    - `X-Request-ID: {{ $execution.id }}`
  - Timeout: **30,000 ms**
- **Inputs/Outputs:** From **Fetch Pharmacy Readiness** → to **Aggregate System Data**.
- **Version requirements:** typeVersion **4.3**.
- **Edge cases / failures:**
  - **Likely URL typo:** `.../patient/={{ ... }}` includes an extra `=` after `/patient/`. Most APIs expect `/patient/{id}/case-status`. This will often yield 404/400.
  - If the patientId is missing, URL becomes invalid.

**Important integration gap:**  
A downstream node references **Fetch Nursing Status**, but **no such node exists** in this workflow JSON. This will break aggregation or produce `undefined` nursing data.

---

### Block 2.4 — Data Aggregation & Completeness Validation
**Overview:** Consolidates all system responses into one object and checks that required operational fields exist before AI orchestration.

**Nodes involved:**
- Aggregate System Data
- Validate Data Completeness

#### Node: Aggregate System Data
- **Type / role:** Set — maps system responses into named properties.
- **Configuration choices (as implemented):**
  - Sets:
    - `ehrData = $('Fetch EHR Patient Data').first().json`
    - `nursingData = $('Fetch Nursing Status').first().json` (**broken reference**)
    - `pharmacyData = $('Fetch Pharmacy Readiness').first().json`
    - `caseManagementData = $('Fetch Case Management Data').first().json`
    - `aggregatedAt = $now.toISO()`
    - `executionId = $execution.id`
  - `includeOtherFields: true`
- **Inputs/Outputs:** From **Fetch Case Management Data** → to **Validate Data Completeness**.
- **Version requirements:** typeVersion **3.4**.
- **Edge cases / failures:**
  - **Hard failure risk** due to missing node reference `Fetch Nursing Status`.
  - Naming mismatch with validator: validator checks `data.ehr`, `data.nursing`, `data.pharmacy`, `data.caseManagement`, but this node outputs `ehrData`, `nursingData`, etc. That means validation will *always* report missing objects unless renamed.

#### Node: Validate Data Completeness
- **Type / role:** Code — validates required fields across system data.
- **Configuration choices:**
  - Iterates all input items; for each item, checks existence of:
    - `data.ehr` with required fields: `patientId, firstName, lastName, dateOfBirth, medicalRecordNumber`
    - `data.nursing` fields: `assignedNurse, vitalSigns, lastAssessment, careLevel`
    - `data.pharmacy` fields: `medicationList, allergies, pharmacistReview, readyForDispense`
    - `data.caseManagement` fields: `caseManager, dischargeStatus, followUpPlan, insuranceVerified`
  - Produces `validation` object: `isValid`, `missingFields`, `validatedAt`, `totalMissingFields`
- **Inputs/Outputs:** From **Aggregate System Data** → to **Healthcare Operations Orchestrator**.
- **Version requirements:** typeVersion **2**.
- **Edge cases / failures:**
  - Because upstream uses `ehrData` etc., `data.ehr` is undefined → validation marks everything missing.
  - If code runs but data is large, performance concerns; also ensure not to log PHI inadvertently.

**Net effect as-is:** The workflow’s “validation gate” does not actually prevent continuation; it only annotates the item. The AI agent will receive “missing” validation results unless you add an IF stop/exception branch.

---

### Block 2.5 — AI Orchestration with Claude + Structured Parsing
**Overview:** A LangChain agent uses Claude to decide the operation type and produce deterministic structured JSON with actions, blockers, escalation requirements, and compliance metadata.

**Nodes involved:**
- Healthcare Operations Orchestrator
- Claude Model
- Structured Operations Output

#### Node: Healthcare Operations Orchestrator
- **Type / role:** LangChain Agent — orchestrates reasoning and tool usage (here: model + structured output parser).
- **Configuration choices:**
  - Input text: `Patient Data: {{ JSON.stringify($json) }}`
  - System message enforces:
    - **No clinical decisions**
    - Operational-only coordination
    - HIPAA/data minimization/auditability
    - Required output JSON fields (operationType, readinessStatus, blockers, workflowActions, requiresHumanReview, etc.)
  - `promptType: define`
  - `hasOutputParser: true` (wired to Structured Operations Output)
- **Connections:**
  - Main input from **Validate Data Completeness**
  - AI language model input from **Claude Model**
  - Output parser input from **Structured Operations Output**
  - Main output to **Route by Operation Type**
- **Version requirements:** typeVersion **3.1** (@n8n LangChain nodes).
- **Edge cases / failures:** model timeouts; parser failures if output not matching schema; PHI leakage risk by stringifying entire `$json`; ensure `$json` only contains minimum operational data.

#### Node: Claude Model
- **Type / role:** Anthropic chat model provider.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
  - Temperature: **0.1** (low variance)
  - Max tokens: **4096**
  - Credential: **Anthropic account**
- **Connections:** Provides `ai_languageModel` to **Healthcare Operations Orchestrator**.
- **Version requirements:** typeVersion **1.3**.
- **Edge cases / failures:** invalid API key; rate limits; model name availability; token overrun if input data is too large.

#### Node: Structured Operations Output
- **Type / role:** Structured Output Parser — enforces JSON schema-like structure.
- **Configuration choices:**
  - Provides an example JSON schema (operationType/readinessStatus/etc.). Example uses lowercase values (`"admission"`, `"ready"`) while router expects uppercase constants (`ADMISSION`, etc.).
- **Connections:** Provides `ai_outputParser` to agent.
- **Version requirements:** typeVersion **1.3**.
- **Edge cases / failures:**
  - **Case mismatch risk:** If the model returns lowercase as in example, the Switch will fall through to fallback.
  - If `workflowActions` is not an array or fields missing, parse may error (depending on parser strictness).

**Sticky note coverage:**
- **Sticky Note4 (Intelligent Patient Triage)** conceptually applies, though the implemented workflow focuses on operational orchestration rather than clinical triage.

---

### Block 2.6 — Operation Routing & Downstream Workflow Execution
**Overview:** Routes execution to admission/discharge/post-care endpoints and passes along patientId, workflowActions, and audit metadata.

**Nodes involved:**
- Route by Operation Type
- Execute Admission Workflow
- Execute Discharge Workflow
- Execute Post-Care Workflow

#### Node: Route by Operation Type
- **Type / role:** Switch — conditional routing.
- **Configuration choices:**
  - Evaluates: `{{ $json.output.operationType }}`
  - Routes:
    - “Admission” if equals `ADMISSION`
    - “Discharge” if equals `DISCHARGE`
    - “Post-Care” if equals `POST_CARE`
  - Fallback output: `extra`
- **Inputs/Outputs:** From **Healthcare Operations Orchestrator** → to the three Execute nodes.
- **Version requirements:** typeVersion **3.4**.
- **Edge cases / failures:**
  - If the agent output path is not `$json.output.operationType` (depends on agent node output structure), routing fails.
  - If case is lowercase or different enum, all go to fallback (but fallback is not connected, so execution stops silently after this node).

#### Node: Execute Admission Workflow / Execute Discharge Workflow / Execute Post-Care Workflow
- **Type / role:** HTTP Request — calls an external workflow system endpoint.
- **Common configuration choices:**
  - Method: POST
  - URL: from Workflow Configuration (`admissionWorkflowUrl`, etc.)
  - JSON body:
    - `patientId: $json.ehrData?.patientId`
    - `workflowActions: $json.output?.workflowActions`
    - `executionId: $execution.id`
    - `timestamp: $now.toISO()`
  - Headers:
    - `Authorization: <Workflow System API Token placeholder>`
    - `Content-Type: application/json`
- **Connections:** Each → **Check Exception Flag**
- **Version requirements:** typeVersion **4.3** each.
- **Edge cases / failures:**
  - If earlier aggregation used `ehrData` but AI agent output item doesn’t include it (or was renamed), `patientId` becomes null.
  - Endpoint auth/400 errors; idempotency concerns if retried.
  - WorkflowActions may be empty or malformed; downstream system must validate.

---

### Block 2.7 — Exception Escalation, Path Merge, Audit Logging, Compliance Summary
**Overview:** If the AI indicates human review is required, it escalates to staff via Slack and email. All paths then merge and an audit log is written; finally a compliance report object is assembled.

**Nodes involved:**
- Check Exception Flag
- Escalate to Clinical Staff
- Send Exception Email
- Merge Workflow Paths
- Log Audit Trail
- Format Compliance Report

#### Node: Check Exception Flag
- **Type / role:** IF — evaluates whether escalation is needed.
- **Configuration choices:**
  - Condition: `{{ $json.output.requiresHumanReview }} == true`
- **Connections (as built):**
  - Only the **true** output is connected (to Slack + Email).
  - No explicit false path connection → if false, execution ends here and **audit logging will not run**.
- **Version requirements:** typeVersion **2.3**.
- **Edge cases / failures:** If `requiresHumanReview` is undefined or string `"true"`, strict boolean check may fail.

#### Node: Escalate to Clinical Staff
- **Type / role:** Slack — posts an exception message.
- **Configuration choices:**
  - OAuth2 Slack credential
  - Channel ID from config: `escalationSlackChannel`
  - Message includes operationType, patientId, readiness, blockers, compliance flags, executionId, timestamp
- **Connections:** → **Merge Workflow Paths (input 0)**
- **Version requirements:** typeVersion **2.4**.
- **Edge cases / failures:** Slack auth revoked; channel ID invalid; message includes PHI (patientId might be considered PHI in some contexts—confirm policy).

#### Node: Send Exception Email
- **Type / role:** Email Send — sends HTML alert.
- **Configuration choices:**
  - To: `escalationEmail`
  - From: `complianceOfficerEmail`
  - Subject: includes operationType and patientId
  - HTML body lists blockers, compliance flags, execution metadata
- **Connections:** → **Merge Workflow Paths (input 1)**
- **Version requirements:** typeVersion **2.1**.
- **Edge cases / failures:** SMTP/credential config missing; DMARC/SPF issues; sending PHI via email may violate policy unless secured.

#### Node: Merge Workflow Paths
- **Type / role:** Merge — consolidates results.
- **Configuration choices:** `numberInputs: 3` but only inputs 0 and 1 are connected.
- **Connections:** → **Log Audit Trail**
- **Version requirements:** typeVersion **3.2**.
- **Edge cases / failures:** Merge might wait for 3 inputs and stall depending on merge mode; consider setting mode to “Pass-through” and correct numberInputs.

#### Node: Log Audit Trail
- **Type / role:** HTTP Request — writes an audit event.
- **Configuration choices:**
  - POST to `auditLogUrl`
  - JSON body includes:
    - eventType `healthcare_operations_orchestration`
    - operationType, patientId, readinessStatus
    - workflowActions, requiresHumanReview, escalationReason, complianceFlags
    - executionId, timestamp
    - systemsValidated: `['EHR','Nursing','Pharmacy','CaseManagement']`
    - dataMinimization/hipaaCompliant flags set true
  - Auth header placeholder
- **Connections:** → **Format Compliance Report**
- **Version requirements:** typeVersion **4.3**.
- **Edge cases / failures:** audit endpoint down; storing PHI in audit system; mismatch “Nursing” validated even though nursing data isn’t fetched.

#### Node: Format Compliance Report
- **Type / role:** Set — produces final compliance summary objects.
- **Configuration choices:**
  - Builds `complianceReport` object with execution/time/opType/patientId/readiness/humanReviewRequired/complianceFlags, and flags (auditLogged, dataMinimizationEnforced, etc.)
  - Builds `workflowSummary` object: counts actions/blockers, validated systems=4, escalated boolean
  - `includeOtherFields: true`
- **Outputs:** Final node (also impacts webhook response if triggered via webhook).
- **Version requirements:** typeVersion **3.4**.
- **Edge cases / failures:** if `$json.output` missing, many fields null; counts default safely with `|| 0`.

**Sticky note coverage:**
- **Sticky Note3 (Documentation & Compliance Reporting)** applies to audit/compliance nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Patient Operations Check | scheduleTrigger | Time-based entry point (every 15 min) | — | Merge Triggers | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Patient Event Webhook | webhook | Event-based entry point (POST, header auth) | — | Merge Triggers | ## How It Works\nThis workflow automates patient risk assessment and clinical alerting for healthcare providers using NVIDIA AI models. Designed for hospitals, clinics, and healthcare organizations, it addresses the critical challenge of timely identification and response to high-risk patients requiring immediate intervention. The system monitors patient data webhooks, enriches records with external EHR data, and analyzes aggregated information through Claude AI for comprehensive risk stratification. Healthcare operations data is fetched and combined with patient metrics to provide contextual risk assessment. NVIDIA's structured generation capabilities ensure standardized clinical outputs, while parallel execution routes enable simultaneous processing: critical cases trigger immediate alerts via email and escalation flags, whereas routine cases follow standard documentation paths. The workflow maintains an audit trail, merges execution results, and generates detailed reports for compliance and quality improvement initiatives. |
| Merge Triggers | merge | Combine schedule + webhook starts | Schedule Patient Operations Check; Patient Event Webhook | Workflow Configuration | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Workflow Configuration | set | Central config (URLs, endpoints, contacts) | Merge Triggers | Fetch EHR Patient Data | ## Setup Steps\n1. Configure Patient Event Webhook with your EHR system endpoint URL and authentication headers\n2. Add NVIDIA API credentials (API key) in Fetch Patient Data and Structured Generation nodes\n3. Connect Claude Model node with Anthropic API key and configure healthcare risk assessment prompt\n4. Set up Gmail node with sender credentials and configure recipient email addresses for clinical alerts\n5. Enable Google Sheets integration for audit logging and specify spreadsheet ID for execution reports |
| Fetch EHR Patient Data | httpRequest | Fetch patient operational status from EHR | Workflow Configuration | Fetch Pharmacy Readiness | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Fetch Pharmacy Readiness | httpRequest | Fetch medication readiness from pharmacy system | Fetch EHR Patient Data | Fetch Case Management Data | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Fetch Case Management Data | httpRequest | Fetch case/discharge planning status | Fetch Pharmacy Readiness | Aggregate System Data | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Aggregate System Data | set | Combine system responses into one item | Fetch Case Management Data | Validate Data Completeness | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Validate Data Completeness | code | Validate required fields across systems | Aggregate System Data | Healthcare Operations Orchestrator | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |
| Healthcare Operations Orchestrator | langchain agent | Decide operation type + actions; enforce non-clinical constraints | Validate Data Completeness | Route by Operation Type | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Claude Model | Anthropic Chat Model | LLM powering the agent | — (AI connection) | Healthcare Operations Orchestrator | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Structured Operations Output | structured output parser | Enforce structured JSON output | — (AI connection) | Healthcare Operations Orchestrator | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Route by Operation Type | switch | Route to admission/discharge/post-care | Healthcare Operations Orchestrator | Execute Admission Workflow; Execute Discharge Workflow; Execute Post-Care Workflow | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Execute Admission Workflow | httpRequest | Call admission workflow endpoint | Route by Operation Type | Check Exception Flag | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Execute Discharge Workflow | httpRequest | Call discharge workflow endpoint | Route by Operation Type | Check Exception Flag | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Execute Post-Care Workflow | httpRequest | Call post-care workflow endpoint | Route by Operation Type | Check Exception Flag | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Check Exception Flag | if | Decide whether to escalate | Execute Admission Workflow; Execute Discharge Workflow; Execute Post-Care Workflow | Escalate to Clinical Staff; Send Exception Email | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Escalate to Clinical Staff | slack | Slack escalation for human review | Check Exception Flag (true) | Merge Workflow Paths | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Send Exception Email | emailSend | Email escalation for human review | Check Exception Flag (true) | Merge Workflow Paths | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Merge Workflow Paths | merge | Merge escalation paths for downstream logging | Escalate to Clinical Staff; Send Exception Email | Log Audit Trail | ## Documentation & Compliance Reporting\n**Why**\nStandardizes documentation, enables real-time clinical response, and maintains regulatory compliance through comprehensive activity logging. |
| Log Audit Trail | httpRequest | POST audit event to audit system | Merge Workflow Paths | Format Compliance Report | ## Documentation & Compliance Reporting\n**Why**\nStandardizes documentation, enables real-time clinical response, and maintains regulatory compliance through comprehensive activity logging. |
| Format Compliance Report | set | Final compliance + summary objects | Log Audit Trail | — | ## Documentation & Compliance Reporting\n**Why**\nStandardizes documentation, enables real-time clinical response, and maintains regulatory compliance through comprehensive activity logging. |
| Sticky Note | stickyNote | Comment node (no execution) | — | — | ## Prerequisites\nNVIDIA API access, Anthropic Claude API key, Google Workspace account (Gmail, Sheets)\n## Use Cases\nEmergency department triage automation, post-operative monitoring for deterioration detection\n## Customization\nModify risk scoring algorithms, add disease-specific assessment criteria\n## Benefits\nReduces clinical response time through automated risk detection |
| Sticky Note1 | stickyNote | Comment node (no execution) | — | — | ## Setup Steps\n1. Configure Patient Event Webhook with your EHR system endpoint URL and authentication headers\n2. Add NVIDIA API credentials (API key) in Fetch Patient Data and Structured Generation nodes\n3. Connect Claude Model node with Anthropic API key and configure healthcare risk assessment prompt\n4. Set up Gmail node with sender credentials and configure recipient email addresses for clinical alerts\n5. Enable Google Sheets integration for audit logging and specify spreadsheet ID for execution reports |
| Sticky Note2 | stickyNote | Comment node (no execution) | — | — | ## How It Works\nThis workflow automates patient risk assessment and clinical alerting for healthcare providers using NVIDIA AI models. Designed for hospitals, clinics, and healthcare organizations, it addresses the critical challenge of timely identification and response to high-risk patients requiring immediate intervention. The system monitors patient data webhooks, enriches records with external EHR data, and analyzes aggregated information through Claude AI for comprehensive risk stratification. Healthcare operations data is fetched and combined with patient metrics to provide contextual risk assessment. NVIDIA's structured generation capabilities ensure standardized clinical outputs, while parallel execution routes enable simultaneous processing: critical cases trigger immediate alerts via email and escalation flags, whereas routine cases follow standard documentation paths. The workflow maintains an audit trail, merges execution results, and generates detailed reports for compliance and quality improvement initiatives. |
| Sticky Note3 | stickyNote | Comment node (no execution) | — | — | ## Documentation & Compliance Reporting\n**Why**\nStandardizes documentation, enables real-time clinical response, and maintains regulatory compliance through comprehensive activity logging. |
| Sticky Note4 | stickyNote | Comment node (no execution) | — | — | ## Intelligent Patient Triage\n**Why**\nEnsures critical patients receive urgent attention while maintaining efficient processing for routine cases without unnecessary clinical burden. |
| Sticky Note5 | stickyNote | Comment node (no execution) | — | — | ## Data Collection & AI Risk Analysis\n**Why**\nCombines multiple data sources to provide comprehensive patient profiles, enabling accurate risk stratification beyond single-metric assessments. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Healthcare Operations Orchestration Agent for Patient Care Continuity* (or your preferred name).
- Keep workflow inactive until credentials and URLs are configured.

2) **Add Trigger 1: Schedule**
- Node: **Schedule Trigger**
- Configure: interval every **15 minutes**.

3) **Add Trigger 2: Webhook**
- Node: **Webhook**
- Method: **POST**
- Path: `patient-operations-event`
- Authentication: **Header Auth**
  - Create **HTTP Header Auth** credential with the shared secret/header expected by your sender.
- Response: **Last Node** (or choose “Respond Immediately” if callers can’t wait).

4) **Merge triggers**
- Node: **Merge**
- Connect:
  - Schedule Trigger → Merge (Input 1)
  - Webhook → Merge (Input 2)
- In Merge settings, explicitly set a mode suitable for “either trigger continues” (e.g., **Pass-through**) to avoid waiting behavior.

5) **Add “Workflow Configuration”**
- Node: **Set**
- Add fields (string placeholders or real values):
  - `ehrApiUrl`, `nursingApiUrl`, `pharmacyApiUrl`, `caseManagementApiUrl`
  - `admissionWorkflowUrl`, `dischargeWorkflowUrl`, `postCareWorkflowUrl`
  - `auditLogUrl`
  - `escalationSlackChannel`, `escalationEmail`, `complianceOfficerEmail`
- Enable **Include Other Fields**.
- Connect Merge → Workflow Configuration.

6) **Fetch EHR data**
- Node: **HTTP Request** (“Fetch EHR Patient Data”)
- Method: GET (default)
- URL: `{{ $json.ehrApiUrl }}/patient/{{ $json.patientId || 'current' }}/operational-status`
  - In practice, reference config via the Set node like the original (`$('Workflow Configuration').first().json.ehrApiUrl`).
- Headers:
  - `Authorization: Bearer ...` (or per your API)
  - `X-Request-ID: {{ $execution.id }}`
- Timeout: 30000ms
- Connect Workflow Configuration → Fetch EHR.

7) **Fetch Pharmacy readiness**
- Node: **HTTP Request** (“Fetch Pharmacy Readiness”)
- URL: `{{ pharmacyApiUrl }}/patient/{{ patientId }}/medication-readiness` (use the EHR response patientId)
- Headers: Authorization + X-Request-ID
- Connect Fetch EHR → Fetch Pharmacy.

8) **Fetch Case Management**
- Node: **HTTP Request** (“Fetch Case Management Data”)
- **Fix the URL pattern** (recommended):
  - `{{ caseManagementApiUrl }}/patient/{{ patientId }}/case-status`
- Headers: Authorization + X-Request-ID
- Connect Fetch Pharmacy → Fetch Case Management.

9) **(Required fix) Add missing Nursing fetch**
- Node: **HTTP Request** (“Fetch Nursing Status”) — this node is referenced but missing in the provided workflow.
- Typical URL pattern: `{{ nursingApiUrl }}/patient/{{ patientId }}/nursing-status`
- Connect it in your flow (choose either sequential after EHR, or in parallel via additional branches + Merge).
- Ensure it exists before aggregation.

10) **Aggregate system data**
- Node: **Set** (“Aggregate System Data”)
- Create objects and keep consistent naming. Recommended to match validator:
  - `ehr = Fetch EHR Patient Data.json`
  - `nursing = Fetch Nursing Status.json`
  - `pharmacy = Fetch Pharmacy Readiness.json`
  - `caseManagement = Fetch Case Management Data.json`
  - `aggregatedAt = {{ $now.toISO() }}`
  - `executionId = {{ $execution.id }}`
- Include Other Fields: enabled.
- Connect all fetches appropriately (if parallel, use Merge nodes first).

11) **Validate completeness**
- Node: **Code** (“Validate Data Completeness”)
- Paste the validation script (or adapt) but ensure it checks the same property names you used (`ehr`, `nursing`, etc.).
- Connect Aggregate → Validate.

12) **Add AI Agent: “Healthcare Operations Orchestrator”**
- Node: **AI Agent (LangChain)**
- Input text: `Patient Data: {{ JSON.stringify($json) }}`
- System message: paste constraints and output requirements (operational-only, compliance, structured JSON).
- Connect Validate → Agent.

13) **Add Anthropic model**
- Node: **Anthropic Chat Model**
- Select model (e.g., `claude-sonnet-4-5-20250929`)
- Temperature: 0.1
- Max tokens: 4096
- Create **Anthropic API** credential and select it.
- Connect Model node to Agent via **ai_languageModel** connection.

14) **Add structured output parser**
- Node: **Structured Output Parser**
- Provide a schema/example that uses the **same enums** as your router (`ADMISSION`, `DISCHARGE`, `POST_CARE`, etc.).
- Connect Parser to Agent via **ai_outputParser**.

15) **Route by operation type**
- Node: **Switch**
- Add rules on `{{ $json.output.operationType }}`:
  - equals `ADMISSION`
  - equals `DISCHARGE`
  - equals `POST_CARE`
- Add a fallback path and connect it (recommended) to an escalation or audit path.

16) **Execute downstream workflows**
- Add 3 **HTTP Request** nodes:
  - “Execute Admission Workflow” → POST to `admissionWorkflowUrl`
  - “Execute Discharge Workflow” → POST to `dischargeWorkflowUrl`
  - “Execute Post-Care Workflow” → POST to `postCareWorkflowUrl`
- Send JSON body with: patientId, workflowActions, executionId, timestamp.
- Add Authorization header for the workflow system.
- Connect each Switch output to the corresponding HTTP node.

17) **Exception decision**
- Node: **IF** (“Check Exception Flag”)
- Condition: `{{ $json.output.requiresHumanReview }} equals true`
- Connect each Execute node → IF.

18) **Escalation nodes**
- Node: **Slack**
  - OAuth2 credential setup
  - Channel ID from config
  - Message template using operationType/patientId/blockers/compliance flags
- Node: **Email Send**
  - Configure SMTP or provider credentials in n8n
  - To: escalationEmail; From: complianceOfficerEmail
- Connect IF true → Slack and IF true → Email.

19) **Merge paths and always audit**
- Add **Merge Workflow Paths** and configure it to not wait for missing inputs (or use separate merges).
- **Recommended fix:** connect the **IF false** output (and/or Switch fallback) into the merge too, so audit logging runs for routine cases.

20) **Audit logging**
- Node: **HTTP Request** (“Log Audit Trail”)
- POST to `auditLogUrl` with the audit JSON payload.
- Connect Merge → Log Audit.

21) **Compliance report formatting**
- Node: **Set** (“Format Compliance Report”)
- Build `complianceReport` and `workflowSummary` objects.
- Connect Log Audit → Format Compliance Report.

22) **Activate and test**
- Test with:
  - Webhook POST including `{ "patientId": "123" }`
  - Verify each external endpoint and credentials
  - Confirm router matches enum casing
  - Confirm audit logs execute for both escalated and non-escalated runs

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes claim “NVIDIA API access”, “structured generation”, “Gmail/Google Sheets”, and “risk assessment/triage”, but the provided workflow contains **no NVIDIA nodes**, **no Google Sheets**, and uses **Email Send** (generic) rather than Gmail. | Consistency check vs Sticky Note / Sticky Note1 / Sticky Note2 |
| There is a **missing node reference**: `Fetch Nursing Status` is referenced in aggregation but does not exist. This must be added or removed. | Breaks Aggregate System Data expressions |
| Property naming mismatch: aggregation outputs `ehrData/nursingData/...` but validator expects `ehr/nursing/pharmacy/caseManagement`. Align names or update the validator. | Affects validation accuracy and downstream reliability |
| Case Management URL likely contains a typo: `/patient/={{ patientId }}` should be `/patient/{{ patientId }}`. | Prevents Case Management fetch |
| Routing expects uppercase enums (`ADMISSION/DISCHARGE/POST_CARE`), but the structured output example uses lowercase. Align schema and router. | Prevents correct routing |
| Audit and compliance nodes currently run only on the escalation path because the IF false path is not connected. To ensure full auditability, connect non-exception flows to the merge/audit branch. | Compliance/audit completeness |

If you want, I can propose a corrected node map (minimal edits) that resolves the nursing fetch, naming alignment, merge behavior, and “always audit” requirement while keeping the workflow’s intent unchanged.