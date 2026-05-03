Triage emergency patients via chat with GPT-4.1 and hospital systems

https://n8nworkflows.xyz/workflows/triage-emergency-patients-via-chat-with-gpt-4-1-and-hospital-systems-12986


# Triage emergency patients via chat with GPT-4.1 and hospital systems

## 1. Workflow Overview

**Title:** Triage emergency patients via chat with GPT-4.1 and hospital systems  
**Workflow name (n8n):** Hospital Patient Interaction State-Driven Automation Agent

This workflow implements a **state-driven hospital/patient chat intake and triage automation**. A public chat interface collects patient messages, an OpenAI-powered agent classifies intent and extracts entities into **strict structured JSON**, then the workflow either (a) enriches/validates context, scores priority, and executes authorized actions (e.g., appointment changes + notifications) or (b) escalates to a human when confidence is low. All interactions are routed and logged for auditability.

### 1.1 Patient Input & Session Context (Chat + Memory)
Receives patient messages via a public chat webhook and maintains short-term conversation history.

### 1.2 AI Triage (Intent + Entities + State Machine Output)
An LLM agent (GPT-4.1-mini) produces **JSON-only** output conforming to a defined schema; optional tools can check consent and department availability.

### 1.3 Confidence Gate (Automation vs Human Escalation)
If the AI confidence is high enough, proceed with data enrichment/validation; otherwise create escalation payload and stop automation.

### 1.4 Patient Context Enrichment + Rule Validation + Priority Scoring
Pulls patient history and eligibility from Postgres, validates business rules, computes a priority score/tier.

### 1.5 Action Execution, Notification, Patient Response, and Audit Logging
If an authorized action is required, call hospital appointment API and send notifications; always format a patient-facing response and log the interaction to Postgres.

---

## 2. Block-by-Block Analysis

### Block 1 — Patient Input & Session Context
**Overview:** Captures inbound patient chat messages and attaches a rolling conversation memory to support multi-turn disambiguation.  
**Nodes involved:** Patient Chat Interface, Conversation Memory

#### Node: Patient Chat Interface
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — public chat entry point (webhook-based).
- **Key configuration choices:**
  - `public: true` (anyone with the URL can access).
  - Loads previous session: `memory` (ties into LangChain memory node).
- **Inputs/Outputs:**
  - **Main output →** Hospital Triage Agent
  - **AI memory connection →** Conversation Memory (shared memory link)
- **Edge cases / failures:**
  - Public endpoint abuse (spam, prompt injection attempts).
  - Session continuity depends on correct memory wiring; if memory is reset, the agent may lose context.
- **Version notes:** TypeVersion `1.4` (LangChain chat trigger behavior can differ across minor versions, especially session handling).

#### Node: Conversation Memory
- **Type / Role:** `memoryBufferWindow` — maintains a window of prior turns.
- **Key configuration choices:** Default parameters (buffer/window size not explicitly set in JSON; relies on node defaults).
- **Inputs/Outputs:**
  - **AI memory →** Hospital Triage Agent and Patient Chat Interface (enables “loadPreviousSession”).
- **Edge cases / failures:**
  - If default window is too small, time reasoning or entity references (“that time”, “same doctor”) may fail.
  - Memory growth/retention policies may matter for compliance depending on deployment settings.

---

### Block 2 — AI Triage (Agent + Model + Tools + Structured Output)
**Overview:** Uses GPT-4.1-mini to classify intent, extract entities, enforce constraints, and output strictly validated JSON.  
**Nodes involved:** Hospital Triage Agent, OpenAI Chat Model, Structured JSON Output Parser, Department Availability Checker Tool, Patient Consent Validator Tool

#### Node: OpenAI Chat Model
- **Type / Role:** `lmChatOpenAi` — provides the chat model backing the agent.
- **Key configuration choices:**
  - Model: `gpt-4.1-mini`
  - Credentials: “OpenAi account” (OpenAI API key).
- **Inputs/Outputs:**
  - Connected as **AI Language Model** into Hospital Triage Agent.
- **Edge cases / failures:**
  - OpenAI auth errors, quota/rate limits, timeouts.
  - Model changes can alter JSON compliance unless parser rejects invalid output.

#### Node: Structured JSON Output Parser
- **Type / Role:** `outputParserStructured` — enforces a JSON schema; prevents free-form responses.
- **Key configuration choices:**
  - Manual JSON schema including:
    - `workflow_state` enum (state machine)
    - `intent` enum
    - `entities` object (patient_name, date_time, department, etc.)
    - `action_required`, `action_authorized`, rationale, patient message, audit metadata, `confidence_level`
  - Required fields: `workflow_state`, `intent`, `action_required`, `action_authorized`, `message_to_patient`, `confidence_level`
- **Inputs/Outputs:**
  - Connected as **AI Output Parser** into Hospital Triage Agent.
- **Edge cases / failures:**
  - If the model outputs non-JSON or violates schema (e.g., missing required fields, invalid enum), parsing fails and the agent may error/stop.
  - `entities.date_time` requires `date-time` format; natural language not converted correctly will fail validation.

#### Node: Department Availability Checker Tool
- **Type / Role:** `toolCode` — agent tool to check department slot availability.
- **Key configuration choices:**
  - Simulated slot table in code; expects params: `department`, `date_time`.
  - Returns availability boolean and next slots.
- **Inputs/Outputs:**
  - Connected as **AI Tool** into Hospital Triage Agent.
- **Edge cases / failures:**
  - In real use, must replace simulated data with an API call; network failures and latency are common.
  - Timezone mismatch risk: example slots are Zulu time; patient times may be local.

#### Node: Patient Consent Validator Tool
- **Type / Role:** `toolCode` — agent tool to validate consent for requested action.
- **Key configuration choices:**
  - Simulated consent record map; expects params: `patient_name`, `action_type`.
  - Validates expiration and scope (`consent_types` includes action).
- **Inputs/Outputs:**
  - Connected as **AI Tool** into Hospital Triage Agent.
- **Edge cases / failures:**
  - Real deployments must query a system of record; handle missing patient, multiple matches, consent revocation.
  - The tool uses `action_type` matching against `consent_types`; ensure naming matches your consent taxonomy.

#### Node: Hospital Triage Agent
- **Type / Role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM + tools + memory, produces structured JSON state output.
- **Key configuration choices:**
  - Strong system message defining responsibilities and **strict prohibitions** (no diagnosis/treatment, no bypassing consent/ID rules).
  - `hasOutputParser: true` (requires parser output).
- **Key variables/outputs:**
  - Outputs JSON fields consumed downstream:
    - `intent`, `entities.*`, `workflow_state`
    - `action_required`, `action_authorized`, `message_to_patient`
    - `audit_metadata.*`, `confidence_level`
- **Inputs/Outputs:**
  - **Main input:** Patient Chat Interface
  - **AI Language Model:** OpenAI Chat Model
  - **AI Memory:** Conversation Memory
  - **AI Tools:** Department Availability Checker Tool, Patient Consent Validator Tool
  - **AI Output Parser:** Structured JSON Output Parser
  - **Main output →** Check Confidence Threshold
- **Edge cases / failures:**
  - Tool invocation mismatch (tool expects params not provided by agent).
  - Output schema mismatch leading to parser rejection.
  - Safety/compliance: patient may request diagnosis; system prompt forbids it but must still handle gracefully.

---

### Block 3 — Confidence Gate (Proceed vs Escalate)
**Overview:** Uses the model confidence to decide whether to proceed with automated clinical/operational processing or escalate to a human review path.  
**Nodes involved:** Check Confidence Threshold, Prepare Escalation Data

#### Node: Check Confidence Threshold
- **Type / Role:** IF node — gates workflow by confidence.
- **Key configuration choices:**
  - Condition: `confidence_level >= 0.75`
- **Inputs/Outputs:**
  - **True →** Retrieve Patient History (automation path)
  - **False →** Prepare Escalation Data (human review path)
- **Edge cases / failures:**
  - If `confidence_level` missing or non-numeric (should be required by schema), IF may behave unexpectedly under “loose” validation.
  - Threshold may be too low/high depending on risk tolerance.

#### Node: Prepare Escalation Data
- **Type / Role:** Set node — attaches escalation metadata.
- **Key configuration choices:**
  - Adds:
    - `escalation_reason = "Low confidence score - requires human review"`
    - `escalation_priority = "high"`
    - `original_confidence = {{$json.confidence_level}}`
- **Inputs/Outputs:**
  - **Main output →** Format Response to Patient
- **Edge cases / failures:**
  - This block does not notify staff directly; it only prepares fields. If you need paging/ticket creation, add nodes here.

---

### Block 4 — Context Enrichment + Rule Validation + Priority Scoring
**Overview:** Enriches the AI triage output with historical/eligibility data, validates scheduling rules, and computes a priority score/tier used downstream and for auditing.  
**Nodes involved:** Retrieve Patient History, Enrich with Patient Context, Validate Business Rules, Check Patient Eligibility, Calculate Priority Score

#### Node: Retrieve Patient History
- **Type / Role:** Postgres executeQuery — fetch recent visits/history.
- **Key configuration choices:**
  - Query: `SELECT * FROM patient_history WHERE patient_name = $1 ORDER BY visit_date DESC LIMIT 10`
  - Replacement parameter: `{{$json.entities.patient_name}}`
- **Inputs/Outputs:**
  - **Main output →** Enrich with Patient Context
- **Edge cases / failures:**
  - Patient name ambiguity (same names); should use patient ID in production.
  - SQL errors / missing table / credentials.
  - If no rows returned, downstream `.first()` usage can break (see next node).

#### Node: Enrich with Patient Context
- **Type / Role:** Set node — merges history into the working JSON.
- **Key configuration choices:**
  - `previous_visits = {{ $('Retrieve Patient History').all() }}`
  - `patient_risk_level = {{ $('Retrieve Patient History').first().json.risk_level }}`
- **Inputs/Outputs:**
  - **Main output →** Validate Business Rules
- **Edge cases / failures:**
  - If history is empty, `first()` may be undefined and expression can error.
  - `previous_visits` becomes an array of items (n8n item objects), not a normalized array of rows—may be larger than expected.

#### Node: Validate Business Rules
- **Type / Role:** Code node — checks appointment constraints.
- **Key configuration choices / logic:**
  - Reads: `intent`, `entities.date_time`, `entities.department`
  - Rules:
    - Business hours: 08:00–18:00
    - No weekends
    - Emergency departments require `intent === emergency_indicator`
  - Outputs `business_rules_validation: { passed, violations[], validated_at }`
- **Inputs/Outputs:**
  - **Main output →** Check Patient Eligibility
- **Edge cases / failures:**
  - If `date_time` is null/invalid, `new Date(dateTime)` yields invalid date; `getHours()`/`getDay()` can become `NaN`, producing incorrect validation.
  - Timezone: `new Date()` interprets ISO with timezone, but local offsets can affect “business hours” logic.

#### Node: Check Patient Eligibility
- **Type / Role:** Postgres executeQuery — checks insurance/eligibility/consent-on-file.
- **Key configuration choices:**
  - Query: `SELECT eligibility_status, insurance_active, consent_on_file FROM patient_eligibility WHERE patient_name = $1`
  - Replacement parameter: `{{$json.entities.patient_name}}`
- **Inputs/Outputs:**
  - **Main output →** Calculate Priority Score
- **Edge cases / failures:**
  - Same-name issue (needs stable patient identifier).
  - Eligibility result is returned as rows; current workflow does not explicitly merge fields into main JSON (depends on n8n item merging behavior and query node output).

#### Node: Calculate Priority Score
- **Type / Role:** Code node — computes urgency score & tier.
- **Key configuration choices / logic:**
  - Base score 50
  - Adds intent-based increments (emergency_indicator +100, medication +40, etc.)
  - Multiplies by risk level multiplier (`critical` 2.0, high 1.5, … low 0.8)
  - Confidence penalty: if `< 0.75`, multiply by 0.7
  - Exception escalation: +15 per exception in `audit_metadata.exceptions_detected`
  - Outputs:
    - `priority_score` (rounded)
    - `priority_tier` (urgent/high/medium/low)
- **Inputs/Outputs:**
  - **Main output →** Check Action Required
- **Edge cases / failures:**
  - `patient_risk_level` default is `'low'`; if history enrichment fails, scoring may under-triage.
  - If `confidence_level` missing, comparisons may behave unexpectedly.
  - Priority is used later in audit log; if not present, DB insert may fail depending on schema constraints.

---

### Block 5 — Action Execution, Notification, Patient Response, Routing, and Audit
**Overview:** If an authorized non-none action is required, calls hospital systems; regardless, formats a patient message, routes by intent for downstream handling, and writes an audit log row.  
**Nodes involved:** Check Action Required, Execute Appointment Action, Send Notification, Format Response to Patient, Route by Intent Type, Log Interaction to Audit DB

#### Node: Check Action Required
- **Type / Role:** IF node — decides whether to execute an external appointment action.
- **Key configuration choices:**
  - Conditions (AND):
    - `action_required != "none"`
    - `action_authorized == true`
- **Inputs/Outputs:**
  - **True →** Execute Appointment Action
  - **False →** Format Response to Patient
- **Edge cases / failures:**
  - If `action_required` is a string not in the enum or missing, behavior depends on “loose” type validation.
  - Some intents (e.g., emergency_indicator) may require escalation, not appointment API calls—ensure agent sets `action_required` appropriately.

#### Node: Execute Appointment Action
- **Type / Role:** HTTP Request — calls hospital appointment system.
- **Key configuration choices:**
  - POST to placeholder endpoint: `<Hospital appointment system API endpoint>`
  - JSON body includes:
    - `action`, patient fields, appointment fields, `session_id`
  - Headers:
    - `Content-Type: application/json`
    - `Authorization: <API authentication token>` (placeholder)
- **Inputs/Outputs:**
  - **Main output →** Send Notification
- **Edge cases / failures:**
  - JSON body uses templating like `{{ $json.action_required }}` without quotes; if n8n doesn’t stringify automatically, this can create invalid JSON. Safer pattern is `{{$json.action_required}}` inside quotes in the JSON template.
  - 4xx/5xx errors, retries, idempotency issues (duplicate scheduling).
  - Missing `entities.*` values can lead to downstream API rejection.

#### Node: Send Notification
- **Type / Role:** HTTP Request — sends patient notification (confirmation).
- **Key configuration choices:**
  - POST to placeholder endpoint: `<Hospital notification system API endpoint>`
  - Sends `recipient`, `message`, type `appointment_confirmation`, and metadata.
  - Authorization header placeholder.
- **Inputs/Outputs:**
  - **Main output →** Format Response to Patient
- **Edge cases / failures:**
  - Same JSON-stringification risk as above for `{{ }}` usage.
  - If appointment API failed but still continues, you may notify incorrectly unless you add error handling / success checks.

#### Node: Format Response to Patient
- **Type / Role:** Set node — produces `output` field for chat response payload.
- **Key configuration choices:**
  - `output = {{$json.message_to_patient}}`
  - Keeps other fields (`includeOtherFields: true`)
- **Inputs/Outputs:**
  - **Main output →** Route by Intent Type
- **Edge cases / failures:**
  - If `message_to_patient` missing (should be required by schema), output becomes empty.

#### Node: Route by Intent Type
- **Type / Role:** Switch — categorizes the interaction for downstream handling/audit grouping.
- **Key configuration choices:**
  - Rules (renamed outputs):
    - `appointment`: intent contains `"appointment"`
    - `emergency`: intent equals `"emergency_indicator"`
    - `medication`: intent equals `"medication_inquiry"`
    - `verification`: intent contains `"verification"`
  - Fallback renamed to `general`
- **Inputs/Outputs:**
  - All routes currently connect to **Log Interaction to Audit DB** (single downstream).
- **Edge cases / failures:**
  - Case sensitivity is `true` and type validation `strict` inside the switch rules; if intent casing deviates, routing may fall into `general`.

#### Node: Log Interaction to Audit DB
- **Type / Role:** Postgres insert — writes audit record.
- **Key configuration choices:**
  - Table: `public.interaction_audit_log`
  - Inserted fields include:
    - `intent`, `timestamp`, `session_id`, `action_taken`, `patient_name`
    - `priority_score`, `workflow_state`, `confidence_level`, `authorization_status`
- **Inputs/Outputs:**
  - Input from Route by Intent Type (all branches).
- **Edge cases / failures:**
  - `audit_metadata.timestamp` and `audit_metadata.session_id` are not required by schema; if absent, DB may reject depending on constraints.
  - If DB expects `timestamp` as a timestamp type but receives a string, casting may fail unless column is text.
  - `priority_score` may not exist on the escalation branch (low confidence path) because scoring happens only on the high-confidence branch—this can cause insert failures unless nullable/defaulted.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Patient Chat Interface | LangChain Chat Trigger | Public chat entry point + session handling | — | Hospital Triage Agent | ## Patient Data Collection via Chat Interface<br>**Why:** Captures initial symptoms, medical history, and vital signs through conversational interface, reducing intake time while gathering comprehensive information needed for accurate triage assessment. |
| Conversation Memory | LangChain Memory Buffer Window | Maintains conversation context | — (AI memory wiring) | Hospital Triage Agent; Patient Chat Interface | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for agent reasoning | — | Hospital Triage Agent (AI languageModel) | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Department Availability Checker Tool | LangChain Tool (Code) | Tool for slot availability checks | — | Hospital Triage Agent (AI tool) | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Patient Consent Validator Tool | LangChain Tool (Code) | Tool for consent validation | — | Hospital Triage Agent (AI tool) | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Structured JSON Output Parser | LangChain Structured Output Parser | Enforces JSON schema output | — | Hospital Triage Agent (AI outputParser) | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Hospital Triage Agent | LangChain Agent | Intent/entity extraction + state-driven decision JSON | Patient Chat Interface | Check Confidence Threshold | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Check Confidence Threshold | IF | Gate: automate vs escalate | Hospital Triage Agent | Retrieve Patient History (true); Prepare Escalation Data (false) | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Retrieve Patient History | Postgres (executeQuery) | Fetch recent patient visits | Check Confidence Threshold (true) | Enrich with Patient Context | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Enrich with Patient Context | Set | Merge history + risk level into payload | Retrieve Patient History | Validate Business Rules | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Validate Business Rules | Code | Business hours/weekend/department rules | Enrich with Patient Context | Check Patient Eligibility | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Check Patient Eligibility | Postgres (executeQuery) | Eligibility/insurance/consent lookup | Validate Business Rules | Calculate Priority Score | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Calculate Priority Score | Code | Compute priority_score and tier | Check Patient Eligibility | Check Action Required | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Check Action Required | IF | Execute action only if authorized and needed | Calculate Priority Score | Execute Appointment Action (true); Format Response to Patient (false) | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Execute Appointment Action | HTTP Request | Call appointment system API | Check Action Required (true) | Send Notification | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Send Notification | HTTP Request | Notify patient (confirmation) | Execute Appointment Action | Format Response to Patient | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Prepare Escalation Data | Set | Add escalation metadata | Check Confidence Threshold (false) | Format Response to Patient | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Format Response to Patient | Set | Produce `output` for chat UI | Send Notification OR Check Action Required (false) OR Prepare Escalation Data | Route by Intent Type | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Route by Intent Type | Switch | Route by intent category | Format Response to Patient | Log Interaction to Audit DB | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Log Interaction to Audit DB | Postgres (insert) | Write audit record | Route by Intent Type | — | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Sticky Note | Sticky Note | Comment | — | — | ## Prerequisites<br>Active OpenAI API account, hospital system API access for appointments and notifications<br>## Use Cases<br>Emergency department patient intake, urgent care prioritization, virtual triage for telehealth<br>## Customization<br>Modify triage agent prompts to reflect your clinical protocols, adjust priority scoring algorithms<br>## Benefits<br>Accelerates triage processing by 60%, ensures standardized clinical assessment |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Setup Steps<br>1. Configure OpenAI credentials with API key for AI agent access<br>2. Set up Hospital Triage Agent node with your clinical triage protocols<br>3. Configure Patient Consent and Structured JSON checkers with validation rules<br>4. Connect notification endpoints for Execute Appointment and Send Notification nodes<br>5. Set up audit logging system integration in Log Interactions node<br>6. Customize business rule validation parameters for your facility's triage categories |
| Sticky Note2 | Sticky Note | Comment | — | — | ## How It Works<br>This workflow automates hospital emergency department triage by intelligently processing patient intake information through multiple AI-powered assessment stages. Designed for emergency departments, urgent care centers, and hospital admission teams, it solves the critical challenge of rapid, accurate patient prioritization during high-volume periods. The system captures initial patient data through a chat interface, uses specialized AI agents to analyze medical history and current symptoms, validates business rules for priority assignment, performs stability checks, calculates priority scores, and determines required actions. It then routes patients to appropriate care pathways while sending notifications to relevant medical teams and logging all interactions for audit compliance. The workflow leverages OpenAI models and structured JSON parsing to ensure consistent, protocol-driven triage decisions. |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Priority Calculation and Care Routing<br>**Why:** Computes urgency scores based on assessment data, determines appropriate care pathways (immediate, urgent, standard), triggers notifications to assigned medical teams, and creates audit logs for quality assurance and regulatory compliance. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Multi-Agent AI Medical Assessment<br>**Why:** Analyzes patient data through specialized AI models including OpenAI for symptom interpretation, structured JSON checkers for protocol validation, and stability assessment to ensure systematic evaluation aligned with clinical guidelines. |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Patient Data Collection via Chat Interface<br>**Why:** Captures initial symptoms, medical history, and vital signs through conversational interface, reducing intake time while gathering comprehensive information needed for accurate triage assessment. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Chat entry node**
   - Add node: **Chat Trigger** (`@n8n/n8n-nodes-langchain.chatTrigger`) named **Patient Chat Interface**
   - Set **Public = true**
   - Set option **Load Previous Session = memory**

2) **Add conversation memory**
   - Add node: **Conversation Memory** (`memoryBufferWindow`)
   - Connect **Conversation Memory (AI memory)** to:
     - **Hospital Triage Agent (AI memory)**
     - **Patient Chat Interface (AI memory)**

3) **Add OpenAI model**
   - Add node: **OpenAI Chat Model** (`lmChatOpenAi`)
   - Select model **gpt-4.1-mini**
   - Create/Open credentials: **OpenAI API** credential with your API key
   - Connect **OpenAI Chat Model (AI languageModel)** → **Hospital Triage Agent (AI languageModel)**

4) **Add structured output parser**
   - Add node: **Structured JSON Output Parser** (`outputParserStructured`)
   - Choose **Schema Type: Manual**
   - Paste the JSON schema that enforces:
     - `workflow_state` enum
     - `intent` enum
     - `entities` object (patient_name, date_time, department, doctor_name, appointment_id, symptoms[], medication_name)
     - `action_required` enum and `action_authorized` boolean
     - `message_to_patient`, `audit_metadata`, `confidence_level`
   - Connect **Structured JSON Output Parser (AI outputParser)** → **Hospital Triage Agent (AI outputParser)**

5) **Add agent tools (optional but wired here)**
   - Add node: **Tool (Code)** named **Department Availability Checker Tool**
     - Implement params: `department`, `date_time`
     - Return `{is_available, next_available_slots, ...}`
   - Add node: **Tool (Code)** named **Patient Consent Validator Tool**
     - Implement params: `patient_name`, `action_type`
     - Return `{consent_valid, expiration_date, ...}`
   - Connect each tool as **AI Tool** into **Hospital Triage Agent**

6) **Create the agent**
   - Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`) named **Hospital Triage Agent**
   - Set the **System Message** to include:
     - intent disambiguation, entity extraction, context, time reasoning, rule enforcement, escalation detection
     - strict prohibitions (no diagnosis/treatment; consent/ID compliance)
     - “MUST output structured JSON only”
   - Ensure output parser is enabled (`hasOutputParser`)
   - Connect **Patient Chat Interface → Hospital Triage Agent** (main)

7) **Add confidence gating**
   - Add node: **IF** named **Check Confidence Threshold**
   - Condition: `{{$json.confidence_level}} >= 0.75`
   - Connect **Hospital Triage Agent → Check Confidence Threshold**

8) **Low-confidence escalation branch**
   - Add node: **Set** named **Prepare Escalation Data**
   - Add fields:
     - `escalation_reason` (string) = “Low confidence score - requires human review”
     - `escalation_priority` (string) = “high”
     - `original_confidence` (number) = `{{$json.confidence_level}}`
   - Connect **Check Confidence Threshold (false)** → **Prepare Escalation Data**

9) **High-confidence enrichment: patient history**
   - Add node: **Postgres** named **Retrieve Patient History**
   - Operation: **Execute Query**
   - Query: `SELECT * FROM patient_history WHERE patient_name = $1 ORDER BY visit_date DESC LIMIT 10`
   - Query replacement / parameter: `{{$json.entities.patient_name}}`
   - Connect **Check Confidence Threshold (true)** → **Retrieve Patient History**
   - Configure **Postgres credentials** to your hospital DB (host, db, user, password/SSL).

10) **Enrich context**
   - Add node: **Set** named **Enrich with Patient Context**
   - Add:
     - `previous_visits` (array) = `{{ $('Retrieve Patient History').all() }}`
     - `patient_risk_level` (string) = `{{ $('Retrieve Patient History').first().json.risk_level }}`
   - Connect **Retrieve Patient History → Enrich with Patient Context**
   - (Recommended hardening: handle empty results before using `first()`.)

11) **Validate business rules**
   - Add node: **Code** named **Validate Business Rules**
   - Implement checks:
     - business hours 8–18
     - no weekends
     - emergency departments require emergency intent
   - Connect **Enrich with Patient Context → Validate Business Rules**

12) **Eligibility lookup**
   - Add node: **Postgres** named **Check Patient Eligibility**
   - Operation: **Execute Query**
   - Query: `SELECT eligibility_status, insurance_active, consent_on_file FROM patient_eligibility WHERE patient_name = $1`
   - Parameter: `{{$json.entities.patient_name}}`
   - Connect **Validate Business Rules → Check Patient Eligibility**

13) **Priority scoring**
   - Add node: **Code** named **Calculate Priority Score**
   - Implement the scoring algorithm (base 50 + intent weights, multiply by risk, penalize low confidence, add exception points)
   - Output `priority_score` and `priority_tier`
   - Connect **Check Patient Eligibility → Calculate Priority Score**

14) **Check whether to execute an action**
   - Add node: **IF** named **Check Action Required**
   - Conditions (AND):
     - `{{$json.action_required}} != "none"`
     - `{{$json.action_authorized}} == true`
   - Connect **Calculate Priority Score → Check Action Required**

15) **Execute appointment action (authorized branch)**
   - Add node: **HTTP Request** named **Execute Appointment Action**
   - Method: **POST**
   - URL: your appointment system endpoint
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer <token>` (or your scheme)
   - Body: JSON with `action`, patient/appointment fields, session_id
   - Connect **Check Action Required (true)** → **Execute Appointment Action**

16) **Send notification**
   - Add node: **HTTP Request** named **Send Notification**
   - Method: **POST**
   - URL: your notification system endpoint
   - Headers similar to above
   - Body includes recipient (patient), message, type, metadata (appointment_id, timestamp)
   - Connect **Execute Appointment Action → Send Notification**

17) **Format response to patient (all branches converge)**
   - Add node: **Set** named **Format Response to Patient**
   - Set `output` = `{{$json.message_to_patient}}`
   - Keep other fields
   - Connect:
     - **Send Notification → Format Response to Patient**
     - **Check Action Required (false) → Format Response to Patient**
     - **Prepare Escalation Data → Format Response to Patient**

18) **Route by intent**
   - Add node: **Switch** named **Route by Intent Type**
   - Rules:
     - appointment: `{{$json.intent}} contains "appointment"`
     - emergency: intent equals `emergency_indicator`
     - medication: intent equals `medication_inquiry`
     - verification: intent contains `verification`
   - Fallback output named **general**
   - Connect **Format Response to Patient → Route by Intent Type**

19) **Audit logging**
   - Add node: **Postgres** named **Log Interaction to Audit DB**
   - Operation: **Insert**
   - Table: `public.interaction_audit_log`
   - Map columns from JSON:
     - `intent`, `audit_metadata.timestamp`, `audit_metadata.session_id`
     - `action_required`, `entities.patient_name`
     - `priority_score`, `workflow_state`, `confidence_level`, `action_authorized`
   - Connect **Route by Intent Type** outputs (all branches) → **Log Interaction to Audit DB**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Active OpenAI API account, hospital system API access for appointments and notifications | Sticky note “Prerequisites” |
| Use cases: Emergency department patient intake, urgent care prioritization, virtual triage for telehealth | Sticky note “Prerequisites” |
| Customization: Modify triage agent prompts to reflect your clinical protocols, adjust priority scoring algorithms | Sticky note “Prerequisites” |
| Benefits claim: “Accelerates triage processing by 60%, ensures standardized clinical assessment” | Sticky note “Prerequisites” |
| Setup steps (OpenAI creds, agent protocols, validation rules, notification endpoints, audit logging, business rules tuning) | Sticky note “Setup Steps” |
| “How It Works” overview describing AI assessment stages, rule validation, scoring, routing, notifications, and audit compliance | Sticky note “How It Works” |

