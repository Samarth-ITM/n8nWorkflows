Manage patient appointments and care follow-ups with OpenAI, Gmail and Twilio

https://n8nworkflows.xyz/workflows/manage-patient-appointments-and-care-follow-ups-with-openai--gmail-and-twilio-12985


# Manage patient appointments and care follow-ups with OpenAI, Gmail and Twilio

## 1. Workflow Overview

**Purpose:** This workflow acts as an AI-driven patient appointment assistant. It receives a patient’s chat message, uses OpenAI-powered agents to classify intent and extract entities, applies scheduling/EHR business rules, generates a patient-facing message, and sends it via **email or SMS**.

**Target use cases (from notes):** post-appointment follow-ups, medication adherence check-ins, preventive care scheduling, and general appointment logistics.

### 1.1 Input Reception & Session Continuity
Captures patient chat input via a public chat trigger and keeps short-term conversation context using memory.

### 1.2 Configuration & Context Injection
Sets placeholder variables for hospital name and API base URLs (EHR + scheduling) to be referenced by tools/logic.

### 1.3 AI Intake (Intent + Entity Extraction)
An Intake Agent processes the patient message, extracts structured intent/entities, and orchestrates downstream agent tools.

### 1.4 AI Care Coordination (Rules + EHR/Scheduling Tools)
A Care Coordination Agent Tool applies hospital policies and (optionally) calls HTTP Request Tools to query/update EHR and scheduling systems.

### 1.5 AI Notification Composition
A Notification Agent Tool generates an empathetic, concise patient message and proposes a notification channel.

### 1.6 Notification Delivery (Email/SMS)
Prepares final fields, checks notification method, then sends either an email or an SMS.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Memory
**Overview:** Starts the workflow from a chat interface and provides conversational continuity through buffer memory.  
**Nodes involved:** Patient Input, Conversation Memory

#### Node: **Patient Input**
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger` — public chat entry point.
- **Key configuration (interpreted):**
  - **Public webhook:** enabled (public access).
  - **Subtitle:** “Patient Appointment Assistant”.
  - **Initial message:** greeting prompt shown to patient.
  - **Load previous session:** uses memory (“memory”), enabling multi-turn chat behavior.
- **Key fields/expressions:** downstream uses `{{$json.chatInput}}` (consumed by Intake Agent).
- **Connections:**
  - **Main out →** Workflow Configuration
  - **AI memory in ←** Conversation Memory
- **Potential failures/edge cases:**
  - Public endpoint can receive spam/PHI—consider authentication, rate limiting, and HIPAA/GDPR controls.
  - Session continuity depends on correct memory wiring and n8n execution context persistence.

#### Node: **Conversation Memory**
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — short window memory for chat.
- **Key configuration:** defaults (buffer window settings not customized here).
- **Connections:**
  - **AI memory out →** Intake Agent and Patient Input
- **Potential failures/edge cases:**
  - Memory window size defaults may truncate important context.
  - Sensitive data retention risk; ensure retention policies.

---

### Block 2 — Workflow Configuration & Placeholders
**Overview:** Injects environment-like variables (API URLs, hospital name) into the execution.  
**Nodes involved:** Workflow Configuration

#### Node: **Workflow Configuration**
- **Type / role:** `n8n-nodes-base.set` — sets constants/variables.
- **Key configuration (interpreted):**
  - Adds:
    - `ehrApiUrl` = placeholder (“EHR System API URL”)
    - `schedulingApiUrl` = placeholder (“Scheduling System API URL”)
    - `hospitalName` = placeholder (“Hospital Name”)
  - **Include other fields:** true (keeps prior JSON).
- **Connections:**
  - **Main in ←** Patient Input
  - **Main out →** Intake Agent
- **Potential failures/edge cases:**
  - Values are placeholders; if not replaced, downstream messages (email subject) will contain placeholder hospital name.
  - These base URLs are not actually used by the HTTP Request Tools as configured (tools take only an endpoint string from the AI). If you intend base URL prefixing, add it explicitly.

---

### Block 3 — AI Intake Orchestration (Intent/Entities + Tool Calls)
**Overview:** The Intake Agent interprets patient text, extracts structured fields, and orchestrates care coordination + notification generation via agent tools and parsers.  
**Nodes involved:** Intake Agent, OpenAI Model - Intake, Intake Parser, Care Coordination Agent Tool, Notification Agent Tool

#### Node: **OpenAI Model - Intake**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM backend for the Intake Agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: OpenAI API account
- **Connections:**
  - **AI languageModel out →** Intake Agent
- **Potential failures/edge cases:**
  - OpenAI auth/credits issues, rate limits, model deprecation/renaming.
  - PHI handling: ensure your compliance posture and settings.

#### Node: **Intake Parser**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON output schema for intake.
- **Key configuration:**
  - Manual JSON schema with:
    - `intent` enum: appointment_booking | reschedule | cancellation | reminder_confirmation | general_enquiry
    - `entities` object: patient_name, patient_id, department, doctor, preferred_date, preferred_time, urgency, contact_method
    - `confidence_level` number 0..1
- **Connections:**
  - **AI outputParser →** Intake Agent
- **Potential failures/edge cases:**
  - Model may produce invalid JSON or omit required structure; parser will fail execution.
  - “contact_method” is defined as a string but later used as an **email address/phone number**, which is semantically inconsistent (see Delivery block).

#### Node: **Care Coordination Agent Tool**
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — sub-agent tool called by Intake Agent.
- **Key configuration (interpreted):**
  - Input text comes from AI via:  
    `{{$fromAI("patient_request", "...", "json")}}`
  - System message defines hospital rules, allowed actions, and tool usage (EHR + scheduling).
  - Has structured output enforced by Care Coordination Parser.
- **Connections:**
  - **AI tool out →** Intake Agent
  - **AI outputParser in ←** Care Coordination Parser
  - **AI tool in/out with tools ←** EHR System Tool, Scheduling System Tool
  - **AI languageModel in ←** OpenAI Model - Care Coordination
- **Potential failures/edge cases:**
  - Tool calling may hallucinate endpoints/methods if not constrained.
  - Without strict base URL enforcement, the agent may output unsafe/incorrect URLs.
  - Business rules mention “24-hour advance notice” but no actual time validation is implemented outside the agent reasoning.

#### Node: **Notification Agent Tool**
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — generates the patient message and channel.
- **Key configuration (interpreted):**
  - Input text comes from AI via:  
    `{{$fromAI("coordination_result", "...", "json")}}`
  - Message constraints: <200 words, empathetic, includes details and next steps.
  - Structured output enforced by Notification Parser.
- **Connections:**
  - **AI tool out →** Intake Agent
  - **AI outputParser in ←** Notification Parser
  - **AI languageModel in ←** OpenAI Model - Notification
- **Potential failures/edge cases:**
  - May select “both”, but downstream routing only properly handles “email” vs “not email”.
  - Could generate message not suitable for SMS length constraints.

#### Node: **Intake Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — main orchestrator.
- **Key configuration (interpreted):**
  - Text input: `{{$json.chatInput}}`
  - System message: defines intents/entities, instructs to call:
    1) Care Coordination Agent Tool  
    2) Notification Agent Tool  
    and return final structured output.
  - Has output parser enabled (but the node’s output parser connection is not explicitly shown; instead, downstream “Prepare Final Output” expects `output.*` fields).
- **Connections:**
  - **Main in ←** Workflow Configuration
  - **AI memory in ←** Conversation Memory
  - **AI languageModel in ←** OpenAI Model - Intake
  - **AI tool access ←** Care Coordination Agent Tool, Notification Agent Tool
  - **Main out →** Prepare Final Output
- **Potential failures/edge cases:**
  - If the agent does not include all expected fields (`output.intent`, `output.entities`, etc.), the Set node may produce null/undefined values.
  - If the agent doesn’t call the tools, care coordination/notification quality may degrade.

---

### Block 4 — Care Coordination Tooling (EHR + Scheduling HTTP Tools)
**Overview:** Provides two HTTP Request Tools callable by the Care Coordination agent for real system lookups/updates.  
**Nodes involved:** EHR System Tool, Scheduling System Tool, OpenAI Model - Care Coordination, Care Coordination Parser

#### Node: **OpenAI Model - Care Coordination**
- **Type / role:** LLM backend for Care Coordination Agent Tool.
- **Key configuration:** `gpt-4.1-mini` with OpenAI credentials.
- **Connections:** **AI languageModel out →** Care Coordination Agent Tool
- **Potential failures:** auth/rate limits; tool-call formatting issues.

#### Node: **Care Coordination Parser**
- **Type / role:** structured output parser for care coordination decision.
- **Key schema fields:**
  - `action_required` enum: book_appointment | reschedule_appointment | cancel_appointment | escalate_to_staff | request_more_info | provide_information
  - `escalation_reason` string
  - `appointment_details` (date/time/doctor/department/location)
  - `requires_human_review` boolean
  - `notes` string
- **Connections:** **AI outputParser →** Care Coordination Agent Tool
- **Potential failures:** invalid JSON or missing expected properties.

#### Node: **EHR System Tool**
- **Type / role:** `n8n-nodes-base.httpRequestTool` — HTTP tool callable by agent.
- **Key configuration (interpreted):**
  - URL is entirely provided by the AI via:  
    `{{$fromAI("ehr_endpoint", "...", "string")}}`
  - Authentication: predefined credential type **httpHeaderAuth**
- **Connections:** **AI tool →** Care Coordination Agent Tool
- **Potential failures/edge cases:**
  - Security risk: AI-controlled URL can be unsafe (SSRF) unless restricted.
  - Missing base URL usage; consider prefixing with `ehrApiUrl` and restricting path-only.
  - Auth misconfiguration (headers), 401/403, timeouts, non-JSON responses.

#### Node: **Scheduling System Tool**
- **Type / role:** `n8n-nodes-base.httpRequestTool` — scheduling API tool callable by agent.
- **Key configuration:**
  - URL from AI: `{{$fromAI("scheduling_endpoint", "...", "string")}}`
  - HTTP method from AI (default GET):  
    `{{$fromAI("http_method", "...", "string", "GET")}}`
  - Authentication: predefined credential type **httpHeaderAuth**
- **Connections:** **AI tool →** Care Coordination Agent Tool
- **Potential failures/edge cases:**
  - Same SSRF/endpoint safety concerns as EHR tool.
  - AI may choose POST/PUT/DELETE without correct body—this node is not configured with a request body mapping.
  - Non-idempotent actions risk accidental bookings/cancellations.

---

### Block 5 — Notification Composition (LLM + Structured Output)
**Overview:** Turns coordination decisions into a patient-facing message and channel recommendation.  
**Nodes involved:** OpenAI Model - Notification, Notification Parser

#### Node: **OpenAI Model - Notification**
- **Type / role:** LLM backend for Notification Agent Tool.
- **Key configuration:** `gpt-4.1-mini` with OpenAI credentials.
- **Connections:** **AI languageModel out →** Notification Agent Tool
- **Potential failures:** auth/rate limits; message length constraints not guaranteed.

#### Node: **Notification Parser**
- **Type / role:** structured output parser for notification content.
- **Key schema fields:**
  - `message_to_patient` string
  - `notification_method` enum: email | sms | both
  - `subject` string
  - `urgency_flag` boolean
- **Connections:** **AI outputParser →** Notification Agent Tool
- **Potential failures:** invalid JSON; “both” not handled correctly downstream.

---

### Block 6 — Final Data Shaping & Delivery (Email/SMS)
**Overview:** Extracts final fields from the agent output, decides delivery route, and sends notification.  
**Nodes involved:** Prepare Final Output, Check Notification Method, Send Email Notification, Send SMS Notification

#### Node: **Prepare Final Output**
- **Type / role:** `n8n-nodes-base.set` — normalizes output fields for delivery.
- **Key configuration:**
  - Maps from `{{$json.output.*}}` into top-level fields:
    - `intent`, `entities`, `action_required`, `message_to_patient`, `confidence_level`, `notification_method`
  - Include other fields: true
- **Connections:**
  - **Main in ←** Intake Agent
  - **Main out →** Check Notification Method
- **Potential failures/edge cases:**
  - If the Intake Agent output structure differs (e.g., fields not under `output`), expressions resolve to empty.
  - `action_required` is mapped but may not exist if the Intake Agent doesn’t surface the coordination tool output.

#### Node: **Check Notification Method**
- **Type / role:** `n8n-nodes-base.if` — routes based on delivery channel.
- **Key configuration:**
  - Condition: `{{$json.notification_method}} == "email"`
  - True branch → email; False branch → SMS
- **Connections:**
  - **True →** Send Email Notification
  - **False →** Send SMS Notification
- **Potential failures/edge cases:**
  - If `notification_method` is `"both"`, it will go to **SMS only** (false branch). No dual-send occurs.
  - If `notification_method` is missing/null, it goes to SMS branch unintentionally.

#### Node: **Send Email Notification**
- **Type / role:** `n8n-nodes-base.emailSend` — sends email to patient.
- **Key configuration:**
  - HTML body: `{{$json.message_to_patient}}`
  - Subject: `{{$json.intent.replace("_", " ").toUpperCase()}} - {{ $('Workflow Configuration').first().json.hospitalName }}`
  - To: `{{$json.entities.contact_method}}`
  - From: placeholder hospital email address
- **Connections:** **Main in ←** Check Notification Method (true branch)
- **Potential failures/edge cases:**
  - **Critical mapping issue:** `entities.contact_method` (defined as “email/sms/phone”) is used as an **email address**. This will fail unless the entity actually contains an email.
  - Requires SMTP or Gmail/OAuth configuration depending on n8n email setup (node is generic emailSend, not Gmail node).
  - Subject uses only `replace("_", " ")` which replaces the first underscore only; multi-underscore intents would be partially formatted.

#### Node: **Send SMS Notification**
- **Type / role:** `n8n-nodes-base.twilio` — sends SMS via Twilio.
- **Key configuration:**
  - To: `{{$json.entities.contact_method}}`
  - From: placeholder Twilio number
  - Message: `{{$json.message_to_patient}}`
- **Connections:** **Main in ←** Check Notification Method (false branch)
- **Potential failures/edge cases:**
  - Same mapping issue: `entities.contact_method` is used as a **phone number**.
  - Twilio failures: invalid E.164 formatting, unverified numbers (trial), region restrictions, messaging service settings.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Patient Input | @n8n/n8n-nodes-langchain.chatTrigger | Public chat entry point for patient requests | — | Workflow Configuration | ## Patient Information Capture\n**Why:** Initiates the workflow with structured patient data including medical history, appointment details, and contact preferences, establishing the foundation for personalized care coordination. |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains short-term chat context | — | Intake Agent; Patient Input | ## Patient Information Capture\n**Why:** Initiates the workflow with structured patient data including medical history, appointment details, and contact preferences, establishing the foundation for personalized care coordination. |
| Workflow Configuration | n8n-nodes-base.set | Stores hospital/API configuration placeholders | Patient Input | Intake Agent | ## Patient Information Capture\n**Why:** Initiates the workflow with structured patient data including medical history, appointment details, and contact preferences, establishing the foundation for personalized care coordination. |
| Intake Agent | @n8n/n8n-nodes-langchain.agent | Interprets request, extracts intent/entities, orchestrates tools | Workflow Configuration | Prepare Final Output | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| OpenAI Model - Intake | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Intake Agent | — | Intake Agent | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Intake Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured intake output | — | Intake Agent | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Care Coordination Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Applies hospital rules; may call EHR/scheduling tools | — | Intake Agent | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| OpenAI Model - Care Coordination | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Care Coordination Agent Tool | — | Care Coordination Agent Tool | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Care Coordination Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured coordination decision output | — | Care Coordination Agent Tool | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| EHR System Tool | n8n-nodes-base.httpRequestTool | Agent-callable HTTP tool to query EHR | — | Care Coordination Agent Tool | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Scheduling System Tool | n8n-nodes-base.httpRequestTool | Agent-callable HTTP tool to query/update scheduling | — | Care Coordination Agent Tool | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Notification Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Composes patient message + channel selection | — | Intake Agent | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| OpenAI Model - Notification | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for Notification Agent Tool | — | Notification Agent Tool | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Notification Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured notification output | — | Notification Agent Tool | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Prepare Final Output | n8n-nodes-base.set | Flattens agent output into delivery fields | Intake Agent | Check Notification Method | ## Multi-Channel Notification Delivery\n**Why:** Automatically sends personalized follow-up messages through patient-preferred channels (email/SMS), ensuring reliable communication delivery and improved patient engagement with minimal manual intervention. |
| Check Notification Method | n8n-nodes-base.if | Routes to email vs SMS delivery | Prepare Final Output | Send Email Notification; Send SMS Notification | ## Multi-Channel Notification Delivery\n**Why:** Automatically sends personalized follow-up messages through patient-preferred channels (email/SMS), ensuring reliable communication delivery and improved patient engagement with minimal manual intervention. |
| Send Email Notification | n8n-nodes-base.emailSend | Sends email message | Check Notification Method | — | ## Multi-Channel Notification Delivery\n**Why:** Automatically sends personalized follow-up messages through patient-preferred channels (email/SMS), ensuring reliable communication delivery and improved patient engagement with minimal manual intervention. |
| Send SMS Notification | n8n-nodes-base.twilio | Sends SMS message | Check Notification Method | — | ## Multi-Channel Notification Delivery\n**Why:** Automatically sends personalized follow-up messages through patient-preferred channels (email/SMS), ensuring reliable communication delivery and improved patient engagement with minimal manual intervention. |
| Sticky Note | n8n-nodes-base.stickyNote | Commentary | — | — | ## Prerequisites\nActive OpenAI API account with credits, connected email service (Gmail recommended)\n## Use Cases\nPost-appointment follow-up reminders, medication adherence check-ins, preventive care scheduling\n## Customization\nModify AI prompts in agent tools to match your clinical workflows, adjust notification timing logic\n## Benefits\nReduces administrative workload by 70%, ensures consistent patient follow-up |
| Sticky Note1 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Setup Steps\n1. Configure OpenAI credentials with API key for AI model access\n2. Set up EHR System Tool node with your electronic health records integration endpoint\n3. Configure Scheduling System Tool with your appointment management system API\n4. Connect Gmail account for email notifications with OAuth authentication\n5. Add Twilio credentials for SMS delivery with account SID and auth token\n6. Customize Care Coordination Agent Tool parameters for your clinical protocols |
| Sticky Note2 | n8n-nodes-base.stickyNote | Commentary | — | — | ## How It Works\nThis workflow automates patient care coordination in healthcare settings by intelligently processing patient information and scheduling follow-up communications through multiple channels. Designed for healthcare administrators, clinic coordinators, and medical practice managers, it solves the critical problem of manual patient follow-up management and inconsistent communication across care teams. The system receives patient intake data, uses AI-powered agents to analyze care requirements and determine appropriate notification timing, then automatically dispatches personalized messages via email and SMS. The workflow leverages OpenAI's advanced models for care coordination logic and notification content generation, ensuring contextually appropriate and timely patient communications while maintaining conversation history for continuity of care. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Patient Information Capture\n**Why:** Initiates the workflow with structured patient data including medical history, appointment details, and contact preferences, establishing the foundation for personalized care coordination. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Commentary | — | — | ## AI Care Coordination Analysis\n**Why:** Processes patient information through intelligent agents that evaluate care requirements, determine optimal follow-up schedules, and generate contextually appropriate communication strategies based on medical protocols. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Commentary | — | — | ## Multi-Channel Notification Delivery\n**Why:** Automatically sends personalized follow-up messages through patient-preferred channels (email/SMS), ensuring reliable communication delivery and improved patient engagement with minimal manual intervention. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Patient Input”**
   - Node type: **LangChain Chat Trigger**
   - Set **Public** = true
   - Options:
     - Subtitle: *Patient Appointment Assistant*
     - Load previous session: *memory*
   - Initial Messages: the provided greeting text.

2. **Create “Conversation Memory”**
   - Node type: **Memory Buffer Window**
   - Keep defaults (or set window size per policy).
   - Connect **Conversation Memory (ai_memory)** → **Patient Input (ai_memory)**.
   - Connect **Conversation Memory (ai_memory)** → **Intake Agent (ai_memory)**.

3. **Create “Workflow Configuration” (Set node)**
   - Add fields (strings):
     - `ehrApiUrl`
     - `schedulingApiUrl`
     - `hospitalName`
   - Enable **Include Other Fields**.
   - Connect **Patient Input → Workflow Configuration**.

4. **Create “OpenAI Model - Intake”**
   - Node type: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-4.1-mini`
   - Configure **OpenAI credentials** (API key).
   - Connect **OpenAI Model - Intake (ai_languageModel)** → **Intake Agent (ai_languageModel)**.

5. **Create “Intake Parser”**
   - Node type: **Structured Output Parser**
   - Schema: use the workflow’s intake JSON schema (intent/entities/confidence).
   - Connect **Intake Parser (ai_outputParser)** → **Intake Agent (ai_outputParser)**.

6. **Create “Care Coordination Parser”**
   - Node type: **Structured Output Parser**
   - Schema: use the workflow’s care coordination schema (action_required, appointment_details, etc.).
   - This will be connected to the Care Coordination Agent Tool.

7. **Create “Notification Parser”**
   - Node type: **Structured Output Parser**
   - Schema: use the workflow’s notification schema (message_to_patient, notification_method, subject, urgency_flag).
   - This will be connected to the Notification Agent Tool.

8. **Create “OpenAI Model - Care Coordination”**
   - Node type: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-4.1-mini`
   - Same OpenAI credentials.
   - Connect **OpenAI Model - Care Coordination (ai_languageModel)** → **Care Coordination Agent Tool (ai_languageModel)**.

9. **Create “OpenAI Model - Notification”**
   - Node type: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-4.1-mini`
   - Same OpenAI credentials.
   - Connect **OpenAI Model - Notification (ai_languageModel)** → **Notification Agent Tool (ai_languageModel)**.

10. **Create “EHR System Tool”**
    - Node type: **HTTP Request Tool**
    - Authentication: **Predefined Credential Type → HTTP Header Auth**
    - URL expression: `{{$fromAI("ehr_endpoint", "…", "string")}}`
    - Configure the **HTTP Header Auth credential** (e.g., add `Authorization: Bearer ...`).
    - Connect **EHR System Tool (ai_tool)** → **Care Coordination Agent Tool (ai_tool)**.

11. **Create “Scheduling System Tool”**
    - Node type: **HTTP Request Tool**
    - Authentication: **HTTP Header Auth**
    - URL expression: `{{$fromAI("scheduling_endpoint", "…", "string")}}`
    - Method expression: `{{$fromAI("http_method", "…", "string", "GET")}}`
    - Configure the same style credential (or a different one).
    - Connect **Scheduling System Tool (ai_tool)** → **Care Coordination Agent Tool (ai_tool)**.

12. **Create “Care Coordination Agent Tool”**
    - Node type: **Agent Tool**
    - Tool description: as in workflow (care coordination).
    - System message: paste the care coordination instructions/rules.
    - Input text expression: `{{$fromAI("patient_request", "…", "json")}}`
    - Enable **Output Parser** and connect:
      - **Care Coordination Parser (ai_outputParser)** → **Care Coordination Agent Tool (ai_outputParser)**
    - Ensure the node has access to:
      - **OpenAI Model - Care Coordination**
      - **EHR System Tool** and **Scheduling System Tool**

13. **Create “Notification Agent Tool”**
    - Node type: **Agent Tool**
    - System message: paste notification composition rules.
    - Input text expression: `{{$fromAI("coordination_result", "…", "json")}}`
    - Enable **Output Parser** and connect:
      - **Notification Parser (ai_outputParser)** → **Notification Agent Tool (ai_outputParser)**

14. **Create “Intake Agent”**
    - Node type: **Agent**
    - Text: `{{$json.chatInput}}`
    - System message: paste intake instructions (intents/entities + call tools).
    - Enable **hasOutputParser** and ensure it’s connected to **Intake Parser** (step 5).
    - Connect tool access:
      - **Care Coordination Agent Tool (ai_tool)** → **Intake Agent (ai_tool)**
      - **Notification Agent Tool (ai_tool)** → **Intake Agent (ai_tool)**
    - Connect **Workflow Configuration → Intake Agent**.

15. **Create “Prepare Final Output”**
    - Node type: **Set**
    - Map (expressions):
      - `intent` = `{{$json.output.intent}}`
      - `entities` = `{{$json.output.entities}}`
      - `action_required` = `{{$json.output.action_required}}`
      - `message_to_patient` = `{{$json.output.message_to_patient}}`
      - `confidence_level` = `{{$json.output.confidence_level}}`
      - `notification_method` = `{{$json.output.notification_method}}`
    - Connect **Intake Agent → Prepare Final Output**.

16. **Create “Check Notification Method”**
    - Node type: **IF**
    - Condition: string equals
      - Left: `{{$json.notification_method}}`
      - Right: `email`
    - Connect **Prepare Final Output → Check Notification Method**.

17. **Create “Send Email Notification”**
    - Node type: **Send Email**
    - From: hospital email address (replace placeholder)
    - To: `{{$json.entities.contact_method}}`
    - Subject: `{{$json.intent.replace("_", " ").toUpperCase()}} - {{ $('Workflow Configuration').first().json.hospitalName }}`
    - HTML: `{{$json.message_to_patient}}`
    - Configure email transport (SMTP) or compatible n8n email setup.
    - Connect **Check Notification Method (true)** → **Send Email Notification**.

18. **Create “Send SMS Notification”**
    - Node type: **Twilio**
    - From: Twilio phone number (replace placeholder)
    - To: `{{$json.entities.contact_method}}`
    - Message: `{{$json.message_to_patient}}`
    - Configure **Twilio credentials** (Account SID + Auth Token).
    - Connect **Check Notification Method (false)** → **Send SMS Notification**.

**Important implementation note:** As built, `entities.contact_method` is used as the destination address/number. In practice, you likely want separate fields like `contact_email` and `contact_phone`, and keep `contact_method` limited to `{email|sms|phone}`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer |
| ## How It Works (full note content) | Sticky Note2 |
| ## Prerequisites / Use Cases / Customization / Benefits (full note content) | Sticky Note |
| ## Setup Steps (full note content) | Sticky Note1 |
| ## Patient Information Capture (full note content) | Sticky Note3 |
| ## AI Care Coordination Analysis (full note content) | Sticky Note4 |
| ## Multi-Channel Notification Delivery (full note content) | Sticky Note5 |