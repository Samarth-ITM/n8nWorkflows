Validate student progress and orchestrate interventions with Claude and email

https://n8nworkflows.xyz/workflows/validate-student-progress-and-orchestrate-interventions-with-claude-and-email-13156


# Validate student progress and orchestrate interventions with Claude and email

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Validate student progress and orchestrate interventions with Claude and email  
**Workflow name (JSON):** AI-Driven Academic Orchestration and Student Progress Validation System

**Purpose:**  
Receives student progress data via webhook, enriches it with historical learning records from an LMS API, validates the combined dataset for completeness/compliance using a Claude-powered validation agent, and—based on the validation status—either logs compliance or triggers an orchestration agent to recommend/execute interventions (API call) and notify staff via email.

**Target use cases:**  
- Institutions monitoring enrollment/assessment/engagement signals and flagging missing/inconsistent/compliance-risk data.  
- Orchestrating academic interventions (content delivery, instructor outreach, exam scheduling, escalations) with audit logging.

### Logical blocks
1. **1.1 Input Reception & Base Configuration**: Webhook intake + centralized configuration variables.
2. **1.2 Data Enrichment (LMS History Fetch) & Aggregation**: Fetch learning history and merge with webhook payload.
3. **1.3 AI Validation (Claude) + Structured Parsing**: Validate the merged dataset and produce structured validation output.
4. **1.4 Validation-Based Routing & Logging**: If valid, log; if invalid/review, send to orchestration.
5. **1.5 AI Orchestration (Claude) + Structured Parsing**: Produce a structured action plan.
6. **1.6 Action Execution & Post-Routing (Email / Log)**: Execute orchestration via API, then route to email notifications or compliance logging.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Base Configuration

**Overview:**  
Accepts incoming student progress payloads and injects workflow-wide configuration values (API base URLs, emails, thresholds/standards) used later by HTTP and email nodes.

**Nodes involved:**  
- Student Data Webhook  
- Workflow Configuration

#### Node: Student Data Webhook
- **Type / role:** `Webhook` trigger; entry point for POSTed student progress payloads.
- **Key configuration:**
  - HTTP Method: **POST**
  - Path: **`student-progress-data`**
  - Response mode: **Last node** (the webhook response will be whatever the last executed node returns).
- **Inputs / outputs:**  
  - No input (trigger). Output is the incoming request body as `$json` (plus standard webhook metadata depending on n8n settings).
- **Edge cases / failures:**
  - Invalid JSON / unexpected payload shape can break downstream expressions (e.g., missing `studentId`).
  - “Last node” response mode may cause slow webhook responses if AI calls are long; consider switching to “On Received” + async processing if needed.
- **Version notes:** typeVersion **2.1**.

#### Node: Workflow Configuration
- **Type / role:** `Set` node; centralizes configurable constants and placeholders.
- **Key configuration choices (interpreted):**
  - Adds fields while **keeping incoming fields** (`includeOtherFields: true`).
  - Sets:
    - `learningApiUrl` (placeholder)
    - `orchestrationApiUrl` (placeholder)
    - `instructorEmail` (placeholder)
    - `complianceEmail` (placeholder)
    - `ferpaComplianceThreshold` = **0.95**
    - `accreditationStandard` = **SACSCOC**
- **Inputs / outputs:**
  - Input: webhook JSON.
  - Output: original webhook JSON plus the added config fields.
- **Edge cases / failures:**
  - Placeholders must be replaced; otherwise HTTP requests will fail due to invalid URL.
  - `ferpaComplianceThreshold` is defined but not actually used by any downstream node in this workflow (potential design gap).
- **Version notes:** typeVersion **3.4**.

---

### 2.2 Data Enrichment (LMS History Fetch) & Aggregation

**Overview:**  
Retrieves student learning history from an LMS endpoint using `studentId`, then consolidates the webhook data and LMS history into a single normalized object for validation.

**Nodes involved:**  
- Fetch Student Learning History  
- Merge Student Data

#### Node: Fetch Student Learning History
- **Type / role:** `HTTP Request`; enriches data with LMS history.
- **Key configuration choices:**
  - URL expression:  
    `{{ $('Workflow Configuration').first().json.learningApiUrl }}/students/{{ $json.studentId }}/history`
  - Sends headers:
    - `Content-Type: application/json`
    - `Authorization: Bearer <placeholder token>`
- **Inputs / outputs:**
  - Input: output of Workflow Configuration (contains `studentId` and config).
  - Output: LMS response JSON (learning history).
- **Edge cases / failures:**
  - **Missing `studentId`** → malformed URL and/or 404.
  - Auth/token placeholder not replaced → 401/403.
  - LMS downtime/timeouts → workflow fails unless “Continue On Fail” is enabled (not set here).
  - LMS response not JSON → parsing errors.
- **Version notes:** typeVersion **4.3**.

#### Node: Merge Student Data
- **Type / role:** `Code` node; produces a normalized “mergedData” object.
- **Key logic:**
  - Takes:
    - `webhookData = $input.first().json`
    - `learningHistory = $input.last().json`
  - Builds:
    - `studentId`
    - `enrollmentData` default `{}` if missing
    - `assessmentData` default `{}`
    - `learningSignals` default `{}`
    - `learningHistory` default `{}`
    - `submittedAt` default `new Date().toISOString()`
    - `institutionalPolicies` default `{}`
- **Inputs / outputs:**
  - Input: the node expects **two inputs** (webhook + LMS history).
  - Output: one item `{ json: mergedData }`.
- **Important integration note:**  
  In the provided connections, **only “Fetch Student Learning History” is connected into this node**. There is **no direct connection bringing the original webhook payload** into “Merge Student Data”.  
  As written, `$input.first()` and `$input.last()` will both be the same (LMS history), so `webhookData.studentId` may be undefined. To truly merge, you must either:
  - Use a **Merge** node (Combine) to join the two branches before this Code node, or
  - Wire both webhook/config branch and HTTP response branch into this Code node (two inputs).
- **Edge cases / failures:**
  - If webhook payload is not present at this node, merged structure will be incorrect and validation will be unreliable.
- **Version notes:** typeVersion **2**.

---

### 2.3 AI Validation (Claude) + Structured Parsing

**Overview:**  
A dedicated validation agent checks the merged student dataset for completeness, data integrity, FERPA and accreditation compliance, and outputs a structured validation result enforced by a JSON schema parser.

**Nodes involved:**  
- Claude Model - Student Progress  
- Student Progress Output Parser  
- Student Progress Validation Agent

#### Node: Claude Model - Student Progress
- **Type / role:** `lmChatAnthropic`; provides the Anthropic Claude chat model to the agent.
- **Key configuration:**
  - Model: `claude-sonnet-4-5-20250929` (as selected in node)
  - Credentials: **Anthropic account** (Anthropic API key)
- **Inputs / outputs / connections:**
  - Connected via `ai_languageModel` to the validation agent.
- **Edge cases / failures:**
  - Missing/invalid Anthropic credentials → auth failure.
  - Model availability/name mismatch → request error.
  - Token limits if `learningHistory` is large (consider truncation/summarization).
- **Version notes:** typeVersion **1.3**.

#### Node: Student Progress Output Parser
- **Type / role:** `outputParserStructured`; enforces structured JSON output from the agent.
- **Schema highlights:**
  - Required fields include: `validationStatus`, booleans for validity/compliance, `complianceScore` (0–1 conceptually), `reasoning`.
  - `validationStatus` enum: `VALID | INVALID | REQUIRES_REVIEW`
- **Connections:**
  - Connected via `ai_outputParser` to the validation agent.
- **Edge cases / failures:**
  - If the model outputs non-JSON or violates schema (missing required fields / wrong types), parser fails and the workflow errors.
- **Version notes:** typeVersion **1.3**.

#### Node: Student Progress Validation Agent
- **Type / role:** `LangChain Agent`; executes the validation prompt with model + parser.
- **Key configuration choices:**
  - Text passed to agent:  
    `Student Data: {{ JSON.stringify($json) }}`
  - System message: strict validation-only constraints (no grading), FERPA/accreditation checks, compute compliance score, list issues.
  - `hasOutputParser: true` ensures outputs go through the structured parser.
- **Inputs / outputs:**
  - Input: merged student dataset from “Merge Student Data”.
  - Output: agent result placed under `$json.output` (n8n AI agent convention), containing the parsed structured fields.
- **Edge cases / failures:**
  - Large JSON string → token overflow / higher cost.
  - Ambiguous institutional policies (if absent) may lead to “REQUIRES_REVIEW”.
  - If upstream merge is wrong (see Block 2.2), validation becomes meaningless.
- **Version notes:** typeVersion **3.1**.

---

### 2.4 Validation-Based Routing & Logging

**Overview:**  
Routes execution based on validation status. “VALID” goes to audit logging; “INVALID/REQUIRES_REVIEW” proceeds to orchestration planning (and valid also logs).

**Nodes involved:**  
- Route by Validation Status  
- Log Compliance Audit Trail

#### Node: Route by Validation Status
- **Type / role:** `Switch`; branches by `$json.output.validationStatus`.
- **Rules (as configured):**
  - Output “Valid” if `equals` `VALID`.
  - Output “Invalid or Review” uses an **array contains** operation against `["INVALID","REQUIRES_REVIEW"]`.
- **Important correctness note:**  
  The condition uses:  
  - Left: `{{ $json.output.validationStatus }}` (a string)  
  - Operator: `array contains` with right value `["INVALID","REQUIRES_REVIEW"]`  
  Many n8n switch operations expect the **left** to be an array for “contains”. This may not behave as intended. A safer rule is:
  - Use `string` → `equals` twice with OR, or
  - Use expression: `{{ ["INVALID","REQUIRES_REVIEW"].includes($json.output.validationStatus) }}`
- **Fallback:** enabled (“Fallback” output).
- **Connections:**
  - “Valid” → Log Compliance Audit Trail
  - “Invalid or Review” → Academic Orchestration Agent
- **Edge cases / failures:**
  - If `$json.output` is missing due to parser failure, conditions will error or route to fallback.
- **Version notes:** typeVersion **3.4**.

#### Node: Log Compliance Audit Trail
- **Type / role:** `Code`; builds and prints a compliance/audit log record for validation or orchestration.
- **Key logic:**
  - Detects log type:
    - If `inputData.output.validationStatus` → `logType='VALIDATION'` and logs validation details.
    - Else if `inputData.output.actionType` → `logType='ORCHESTRATION'` and logs action details.
  - Adds:
    - `timestamp`
    - `workflowExecutionId: $execution.id`
    - `complianceFrameworks.ferpa = true`
    - `complianceFrameworks.accreditation` from Workflow Configuration
  - `console.log(...)` (logs to n8n execution logs; does **not** persist externally).
- **Inputs / outputs:**
  - Input: either validation output item or orchestration output item.
  - Output: a single `auditLog` JSON object.
- **Edge cases / failures:**
  - If upstream doesn’t contain `studentId`, it logs `UNKNOWN`.
  - Logging is not persisted unless you add a DB/storage node afterwards.
- **Version notes:** typeVersion **2**.

---

### 2.5 AI Orchestration (Claude) + Structured Parsing

**Overview:**  
For invalid/review cases, a second agent recommends orchestration actions (notifications, scheduling, escalations, logging) and outputs a structured action plan.

**Nodes involved:**  
- Claude Model - Orchestration  
- Orchestration Output Parser  
- Academic Orchestration Agent

#### Node: Claude Model - Orchestration
- **Type / role:** `lmChatAnthropic`; model provider for orchestration agent.
- **Key configuration:**
  - Model: `claude-sonnet-4-5-20250929`
  - Credentials: same Anthropic account.
- **Connections:** `ai_languageModel` → Academic Orchestration Agent
- **Edge cases:** same as validation model (credentials, limits).
- **Version notes:** typeVersion **1.3**.

#### Node: Orchestration Output Parser
- **Type / role:** `outputParserStructured`; enforces a structured orchestration plan.
- **Schema highlights:**
  - Required: `actionType`, `priority`, `reasoning`
  - `actionType` enum includes: `CONTENT_DELIVERY`, `INSTRUCTOR_NOTIFICATION`, `EXAM_SCHEDULING`, `EXCEPTION_ESCALATION`, `COMPLIANCE_LOG`
  - Optional nested objects: `contentDeliveryPlan`, `instructorNotification`, `examScheduling`, `exceptionEscalation`
- **Connections:** `ai_outputParser` → Academic Orchestration Agent
- **Edge cases / failures:**
  - If `actionType=INSTRUCTOR_NOTIFICATION` but `instructorNotification` object is missing, downstream email node will fail due to missing fields (expressions evaluate to undefined).
- **Version notes:** typeVersion **1.3**.

#### Node: Academic Orchestration Agent
- **Type / role:** `LangChain Agent`; creates intervention plan and compliance requirements.
- **Key configuration:**
  - Text: `Validated Student Data: {{ JSON.stringify($json) }}`
  - System message: constraints (no grading/standing decisions), privacy requirements, escalation rules.
- **Inputs / outputs:**
  - Input: the validated data object (including `$json.output.validationStatus...` from prior agent).
  - Output: structured orchestration plan under `$json.output`.
- **Edge cases / failures:**
  - Over-sharing risk: the prompt embeds full validated data; ensure no unnecessary PII is included in the webhook payload.
- **Version notes:** typeVersion **3.1**.

---

### 2.6 Action Execution & Post-Routing (Email / Log)

**Overview:**  
Executes the orchestration plan via an external orchestration API, then routes results to instructor notification email, compliance escalation email, or compliance logging.

**Nodes involved:**  
- Execute Orchestration Actions  
- Route by Action Type  
- Send Instructor Notification  
- Send Exception Escalation  
- Log Compliance Audit Trail (reused)

#### Node: Execute Orchestration Actions
- **Type / role:** `HTTP Request`; posts the structured orchestration output to an external system.
- **Key configuration:**
  - Method: **POST**
  - URL: `{{ orchestrationApiUrl }}/actions/execute`
  - JSON body: `{{ $json.output }}`
  - Headers: Content-Type + Bearer token placeholder.
- **Inputs / outputs:**
  - Input: item containing `$json.output` from orchestration agent.
  - Output: API response (note: downstream switch expects `$json.output.actionType`, so the API response must preserve/return that structure, or you must pass-through the original data).
- **Critical integration note:**  
  After an HTTP Request, `$json` becomes the **response body** by default. If the orchestration API does not echo back `output.actionType`, the next switch will not work. Common fixes:
  - Enable “Full Response” and/or “Include Input in Output”, or
  - Use a Merge/Set node to keep the original orchestration plan for routing.
- **Edge cases / failures:**
  - Token placeholder not replaced → 401/403.
  - API errors/5xx → workflow fails.
- **Version notes:** typeVersion **4.3**.

#### Node: Route by Action Type
- **Type / role:** `Switch`; routes based on `$json.output.actionType`.
- **Rules:**
  - “Instructor Notification” if equals `INSTRUCTOR_NOTIFICATION`
  - “Exception Escalation” if equals `EXCEPTION_ESCALATION`
  - “Compliance Log” uses **array contains** on left `{{ $json.output.actionType }}` against `["CONTENT_DELIVERY","EXAM_SCHEDULING","COMPLIANCE_LOG"]`
- **Correctness note (same issue as earlier):**  
  This “array contains” likely should be replaced with a proper string-in-list check.
- **Connections:**
  - Instructor Notification → Send Instructor Notification
  - Exception Escalation → Send Exception Escalation
  - Compliance Log → Log Compliance Audit Trail
- **Edge cases:** missing `$json.output.actionType` (common if prior HTTP response overwrote it).
- **Version notes:** typeVersion **3.4**.

#### Node: Send Instructor Notification
- **Type / role:** `Email Send`; sends an HTML email to the instructor.
- **Key configuration:**
  - To: `{{ instructorEmail }}`
  - From: `{{ complianceEmail }}`
  - Subject: `Student Progress Alert: {{ $json.output.instructorNotification.subject }}`
  - HTML body references:
    - `$json.output.instructorNotification.studentId`
    - `$json.output.priority`
    - `$json.output.instructorNotification.urgency`
    - `$json.output.instructorNotification.message`
    - `$json.output.complianceRequirements` (rendered as `<li>` list if present)
    - `$json.output.reasoning`
- **Inputs / outputs:**  
  - Input: orchestration plan item with required `instructorNotification` fields.
- **Edge cases / failures:**
  - Missing instructorNotification subfields → broken email content or expression errors.
  - Email node requires SMTP/Email credentials configured in n8n (not shown in JSON).
  - FERPA risk: ensure the message contains only allowed data for the recipient.
- **Version notes:** typeVersion **2.1**.

#### Node: Send Exception Escalation
- **Type / role:** `Email Send`; sends a critical escalation to compliance/admin.
- **Key configuration:**
  - To: `{{ complianceEmail }}`
  - From: `{{ complianceEmail }}`
  - Subject: `CRITICAL: Exception Escalation Required - {{ issueType }}`
  - HTML body references `exceptionEscalation.issueType/severity/description/recommendedAction`, plus compliance requirements and reasoning.
- **Edge cases / failures:**
  - Same as above (credentials, missing fields).
- **Version notes:** typeVersion **2.1**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Student Data Webhook | Webhook | Receives student payload via HTTP POST | — | Workflow Configuration | ## How It Works … (full sticky note “How It Works” applies to overall flow) |
| Workflow Configuration | Set | Defines API URLs, emails, thresholds, standards | Student Data Webhook | Fetch Student Learning History | ## Setup Steps … |
| Fetch Student Learning History | HTTP Request | Retrieves LMS student history | Workflow Configuration | Merge Student Data | ## Comprehensive Progress Aggregation … |
| Merge Student Data | Code | Consolidates webhook + LMS history into one object | Fetch Student Learning History *(but should also receive webhook branch)* | Student Progress Validation Agent | ## Comprehensive Progress Aggregation … |
| Claude Model - Student Progress | Anthropic Chat Model | LLM provider for validation agent | — (AI connection) | Student Progress Validation Agent | ## Dual-Agent Academic Assessment … |
| Student Progress Output Parser | Structured Output Parser | Enforces schema for validation output | — (AI connection) | Student Progress Validation Agent | ## Dual-Agent Academic Assessment … |
| Student Progress Validation Agent | LangChain Agent | Validates structure/compliance, computes compliance score | Merge Student Data | Route by Validation Status | ## Dual-Agent Academic Assessment … |
| Route by Validation Status | Switch | Routes VALID vs INVALID/REQUIRES_REVIEW | Student Progress Validation Agent | Academic Orchestration Agent; Log Compliance Audit Trail | ## Validation-Driven Intervention Routing … |
| Claude Model - Orchestration | Anthropic Chat Model | LLM provider for orchestration agent | — (AI connection) | Academic Orchestration Agent | ## Validation-Driven Intervention Routing … |
| Orchestration Output Parser | Structured Output Parser | Enforces schema for orchestration plan | — (AI connection) | Academic Orchestration Agent | ## Validation-Driven Intervention Routing … |
| Academic Orchestration Agent | LangChain Agent | Produces structured intervention/action plan | Route by Validation Status | Execute Orchestration Actions | ## Validation-Driven Intervention Routing … |
| Execute Orchestration Actions | HTTP Request | Calls orchestration API to execute plan | Academic Orchestration Agent | Route by Action Type | ## Validation-Driven Intervention Routing … |
| Route by Action Type | Switch | Routes to instructor email, escalation email, or log | Execute Orchestration Actions | Send Instructor Notification; Send Exception Escalation; Log Compliance Audit Trail | ## Validation-Driven Intervention Routing … |
| Send Instructor Notification | Email Send | Emails instructor intervention notice | Route by Action Type | — | ## Validation-Driven Intervention Routing … |
| Send Exception Escalation | Email Send | Emails compliance officer for critical issues | Route by Action Type | — | ## Validation-Driven Intervention Routing … |
| Log Compliance Audit Trail | Code | Creates audit log (validation or orchestration) | Route by Validation Status; Route by Action Type | — | ## Validation-Driven Intervention Routing … |
| Sticky Note | Sticky Note | Notes (prereqs/use cases/customization/benefits) | — | — | ## Prerequisites … |
| Sticky Note1 | Sticky Note | Setup steps notes | — | — | ## Setup Steps … |
| Sticky Note2 | Sticky Note | “How it works” description | — | — | ## How It Works … |
| Sticky Note3 | Sticky Note | Validation-driven routing description | — | — | ## Validation-Driven Intervention Routing … |
| Sticky Note4 | Sticky Note | Dual-agent assessment description | — | — | ## Dual-Agent Academic Assessment … |
| Sticky Note5 | Sticky Note | Aggregation description | — | — | ## Comprehensive Progress Aggregation … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it (e.g.) **“AI-Driven Academic Orchestration and Student Progress Validation System”**.
   - Keep workflow **Inactive** until credentials and URLs are set.

2. **Add trigger: Webhook**
   - Node: **Webhook** named **Student Data Webhook**
   - Method: **POST**
   - Path: **student-progress-data**
   - Response Mode: **Last Node** (or change later if you want faster acknowledgements).

3. **Add configuration: Set node**
   - Node: **Set** named **Workflow Configuration**
   - Enable **Include Other Fields**.
   - Add fields:
     - `learningApiUrl` (string) = your LMS API base URL
     - `orchestrationApiUrl` (string) = your orchestration API base URL
     - `instructorEmail` (string)
     - `complianceEmail` (string)
     - `ferpaComplianceThreshold` (number) = 0.95 (optional, but define if you will use it)
     - `accreditationStandard` (string) = SACSCOC (or your standard)

4. **Connect:** Student Data Webhook → Workflow Configuration

5. **Add enrichment: HTTP Request (LMS history)**
   - Node: **HTTP Request** named **Fetch Student Learning History**
   - URL (expression):  
     `{{ $('Workflow Configuration').first().json.learningApiUrl }}/students/{{ $json.studentId }}/history`
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer <LMS_TOKEN>`
   - Connect: Workflow Configuration → Fetch Student Learning History

6. **Fix/implement the merge properly (recommended approach)**
   - Add a **Merge** node (n8n Merge) named **Join Webhook+LMS**
     - Mode: **Combine** (by position) is usually simplest for 1:1 items.
   - Connect:
     - Workflow Configuration → Join Webhook+LMS (Input 1)
     - Fetch Student Learning History → Join Webhook+LMS (Input 2)
   - Then add **Code** node named **Merge Student Data**
     - Paste the provided merge code, but ensure it reads:
       - `webhookData = $input.first().json` (from Input 1)
       - `learningHistory = $input.last().json` (from Input 2)
   - Connect: Join Webhook+LMS → Merge Student Data  
   *(If you do not add this merge step, you will not truly have both datasets available.)*

7. **Add AI model (Anthropic) for validation**
   - Node: **Anthropic Chat Model** named **Claude Model - Student Progress**
   - Choose model: `claude-sonnet-4-5-20250929` (or closest available in your environment).
   - Create/select **Anthropic API credentials** in n8n.

8. **Add structured output parser for validation**
   - Node: **Structured Output Parser** named **Student Progress Output Parser**
   - Schema: paste the provided JSON schema for validation output.

9. **Add AI Agent for validation**
   - Node: **AI Agent (LangChain Agent)** named **Student Progress Validation Agent**
   - Prompt type: **Define**
   - Text: `Student Data: {{ JSON.stringify($json) }}`
   - System message: paste the provided validation system message.
   - Enable **Output Parser** and connect:
     - Claude Model - Student Progress → agent via **AI Language Model**
     - Student Progress Output Parser → agent via **AI Output Parser**
   - Connect: Merge Student Data → Student Progress Validation Agent

10. **Add routing: Switch by validation status**
   - Node: **Switch** named **Route by Validation Status**
   - Condition should robustly check status. Recommended:
     - Output “Valid”: `{{ $json.output.validationStatus === "VALID" }}`
     - Output “Invalid or Review”: `{{ ["INVALID","REQUIRES_REVIEW"].includes($json.output.validationStatus) }}`
   - Connect: Student Progress Validation Agent → Route by Validation Status

11. **Add audit logging: Code node**
   - Node: **Code** named **Log Compliance Audit Trail**
   - Paste the provided audit logging code.
   - Connect:
     - Route by Validation Status (“Valid”) → Log Compliance Audit Trail
     - Later you’ll also connect a branch from action routing to this same node.

12. **Add AI model for orchestration**
   - Node: **Anthropic Chat Model** named **Claude Model - Orchestration**
   - Use same Anthropic credentials; select the same model.

13. **Add structured output parser for orchestration**
   - Node: **Structured Output Parser** named **Orchestration Output Parser**
   - Paste the provided orchestration schema.

14. **Add orchestration AI agent**
   - Node: **AI Agent** named **Academic Orchestration Agent**
   - Text: `Validated Student Data: {{ JSON.stringify($json) }}`
   - System message: paste the provided orchestration system message.
   - Connect AI ports:
     - Claude Model - Orchestration → agent (AI Language Model)
     - Orchestration Output Parser → agent (AI Output Parser)
   - Connect: Route by Validation Status (“Invalid or Review”) → Academic Orchestration Agent

15. **Add orchestration execution call**
   - Node: **HTTP Request** named **Execute Orchestration Actions**
   - Method: **POST**
   - URL: `{{ $('Workflow Configuration').first().json.orchestrationApiUrl }}/actions/execute`
   - Body: JSON = `{{ $json.output }}`
   - Headers:
     - `Content-Type: application/json`
     - `Authorization: Bearer <ORCH_TOKEN>`
   - Important: enable a setting like **“Include Input in Output”** (if available) or add a **Set/Merge** after this node to preserve `$json.output.actionType` for routing.
   - Connect: Academic Orchestration Agent → Execute Orchestration Actions

16. **Add routing: Switch by action type**
   - Node: **Switch** named **Route by Action Type**
   - Recommended conditions:
     - Instructor: `{{ $json.output.actionType === "INSTRUCTOR_NOTIFICATION" }}`
     - Escalation: `{{ $json.output.actionType === "EXCEPTION_ESCALATION" }}`
     - Compliance Log: `{{ ["CONTENT_DELIVERY","EXAM_SCHEDULING","COMPLIANCE_LOG"].includes($json.output.actionType) }}`
   - Connect: Execute Orchestration Actions → Route by Action Type

17. **Add email: instructor notification**
   - Node: **Email Send** named **Send Instructor Notification**
   - Configure email credentials (SMTP or provider-supported credentials in n8n).
   - To: `{{ $('Workflow Configuration').first().json.instructorEmail }}`
   - From: `{{ $('Workflow Configuration').first().json.complianceEmail }}`
   - Subject and HTML: use the provided expressions/template.
   - Connect: Route by Action Type (“Instructor Notification”) → Send Instructor Notification

18. **Add email: exception escalation**
   - Node: **Email Send** named **Send Exception Escalation**
   - To/From: `{{ complianceEmail }}`
   - Subject and HTML: use provided template.
   - Connect: Route by Action Type (“Exception Escalation”) → Send Exception Escalation

19. **Connect compliance log branch**
   - Connect: Route by Action Type (“Compliance Log”) → Log Compliance Audit Trail

20. **Test end-to-end**
   - Send a sample POST to `/webhook/student-progress-data` with at least `studentId` and representative `enrollmentData/assessmentData/learningSignals`.
   - Verify:
     - LMS history fetch succeeds
     - Validation output is structured and routes correctly
     - Orchestration API call preserves/returns needed fields for action routing
     - Emails send with required fields present
     - Audit log prints expected entries

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites**: Claude/OpenAI API credentials for AI agents, learning management system API access | Sticky note “Prerequisites” |
| **Use Cases**: Universities identifying students requiring academic support, online learning platforms detecting engagement drops | Sticky note “Prerequisites” |
| **Customization**: Adjust validation thresholds for institutional academic standards | Sticky note “Prerequisites” |
| **Benefits**: Reduces student identification lag by 75%, eliminates manual progress tracking | Sticky note “Prerequisites” |
| **Setup Steps** list (1–9) describing node setup order | Sticky note “Setup Steps” |
| **How It Works**: end-to-end description of the dual-agent validation + orchestration and routing/logging | Sticky note “How It Works” |
| **Design caution**: current wiring does not actually merge webhook payload with LMS history unless you add a Merge / second input to the code node | Derived from the provided connections |
| **Design caution**: after HTTP Request, routing on `$json.output.actionType` may fail unless input is preserved or API echoes the plan | Derived from node behavior |
| **Rule caution**: Switch “array contains” operations are likely misapplied to string values; use explicit string checks or `includes()` | Derived from node rule configuration |