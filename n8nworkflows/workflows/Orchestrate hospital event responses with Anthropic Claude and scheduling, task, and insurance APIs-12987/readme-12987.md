Orchestrate hospital event responses with Anthropic Claude and scheduling, task, and insurance APIs

https://n8nworkflows.xyz/workflows/orchestrate-hospital-event-responses-with-anthropic-claude-and-scheduling--task--and-insurance-apis-12987


# Orchestrate hospital event responses with Anthropic Claude and scheduling, task, and insurance APIs

## 1. Workflow Overview

**Workflow name:** *Hospital Operations and Patient Care Orchestration Agent*  
**Provided title:** *Orchestrate hospital event responses with Anthropic Claude and scheduling, task, and insurance APIs*  

**Purpose:**  
This workflow receives hospital operational events via a webhook, uses an AI orchestration agent (Anthropic Claude via LangChain nodes) to interpret the event and produce **structured JSON output**, then routes follow-up actions to external systems (appointment scheduling, insurance verification, task management). It consolidates results, **masks PII/PHI**, and returns an audit-friendly response.

### 1.1 Event Reception & Configuration
Receives a POST event and injects configurable thresholds/weights used downstream.

### 1.2 AI Orchestration (Claude + Tools + Structured Output)
Uses Claude with a strict system message and a structured output parser to produce deterministic, schema-conformant JSON.

### 1.3 Action Routing & Execution (Multi-system APIs)
Routes based on `action_required` into scheduling, insurance verification, and/or task creation (with task priority calculation).

### 1.4 Consolidation, PHI Masking, Webhook Response
Merges all action outcomes, masks sensitive identifiers, and responds to the original webhook caller.

---

## 2. Block-by-Block Analysis

### Block 1 — Event Reception & Configuration

**Overview:**  
Accepts hospital event payloads via webhook and sets operational constants (thresholds, weights, time windows) used by the agent and task scoring logic.

**Nodes involved:**
- Hospital Event Webhook
- Workflow Configuration

#### Node: Hospital Event Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point receiving event triggers.
- **Key configuration:**
  - **HTTP Method:** `POST`
  - **Path:** `hospital-orchestration` (becomes part of the webhook URL)
  - **Response mode:** `Last Node` (final HTTP response is determined by the last executed response node path)
- **Inputs / outputs:**
  - **Output:** to **Workflow Configuration**
- **Edge cases / failures:**
  - Wrong HTTP method/path → 404/405 at n8n endpoint.
  - Large payloads may exceed instance limits.
  - If downstream path doesn’t hit a response node appropriately, caller may see unexpected output (though a Respond node exists later).
- **Version notes:** typeVersion **2.1**.

#### Node: Workflow Configuration
- **Type / role:** `Set` (n8n-nodes-base.set) — injects configuration parameters into the running item.
- **Key configuration (interpreted):**
  - Adds constants:
    - `dischargeReadinessThreshold = 0.85`
    - Task scoring weights:
      - `taskPriorityWeightUrgency = 0.4`
      - `taskPriorityWeightWorkload = 0.3`
      - `taskPriorityWeightSkillMatch = 0.3`
    - `followUpWindowDays = 7`
    - `medicationReminderHours = 24`
    - `complianceAuditEnabled = true`
  - **Include other fields:** true (keeps the webhook payload, adds these fields alongside it).
- **Inputs / outputs:**
  - **Input:** from **Hospital Event Webhook**
  - **Output:** to **Hospital Orchestration Agent**
- **Edge cases / failures:**
  - If later expressions assume these exist and you delete/rename them, agent prompt expressions and scoring code will fail.
- **Version notes:** typeVersion **3.4**.

---

### Block 2 — AI Orchestration (Claude + Tools + Structured Output)

**Overview:**  
Processes the incoming event with a hospital-specific system policy, uses optional tools, and enforces a strict JSON schema for downstream automation.

**Nodes involved:**
- Hospital Orchestration Agent
- Anthropic Chat Model
- Structured Output Parser
- Staff Availability Lookup Tool
- Calculator

#### Node: Hospital Orchestration Agent
- **Type / role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — central reasoning/orchestration component.
- **Key configuration (interpreted):**
  - **Input text:** `{{ $json }}` (passes the entire current JSON item to the agent as text/context).
  - **System message:** extensive operational policy:
    - Uses **expressions** to inject config values from `Workflow Configuration`, e.g.  
      `{{ $('Workflow Configuration').first().json.dischargeReadinessThreshold }}`
    - Forces constraints:
      - No clinical judgment/medical advice
      - Mask PII/PHI
      - Include audit metadata
      - Output must be structured JSON with required fields
  - **Output parser enabled:** yes (`hasOutputParser: true`) via the Structured Output Parser node.
- **Tools / model connections:**
  - Uses **Anthropic Chat Model** as the language model.
  - Has tools available:
    - **Staff Availability Lookup Tool**
    - **Calculator**
- **Outputs / routing:**
  - Main output goes to:
    - **Respond to Webhook**
    - **Route by Action Type**
- **Edge cases / failures:**
  - Model may refuse or produce invalid JSON if prompt/tooling is misconfigured, but the output parser enforces schema and can error if non-conformant.
  - If expressions referencing `Workflow Configuration` break (node renamed, missing values), system message resolution can fail.
  - PII masking is required by instruction but not technically enforced until later Mask node; if the agent outputs PII, it will still be sanitized later (assuming fields match masking rules).
- **Version notes:** typeVersion **3.1**.

#### Node: Anthropic Chat Model
- **Type / role:** `LangChain Anthropic Chat Model` (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — provides Claude inference.
- **Key configuration:**
  - **Model:** `claude-3-5-sonnet-20241022`
  - Uses Anthropic API credentials: **“Anthropic account”**
- **Connections:**
  - Provides `ai_languageModel` connection into **Hospital Orchestration Agent**
- **Edge cases / failures:**
  - Credential/auth failures (401/403), quota/rate limiting (429), model deprecation, network timeouts.
- **Version notes:** typeVersion **1.3**.

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces a manual JSON schema.
- **Key configuration:**
  - **Schema type:** manual JSON schema requiring:
    - `workflow_state`, `intent`, `entities`, `action_required`, `message_to_patient_or_staff`,
      `audit_metadata`, `confidence_level`, `risk_flags[]`, `task_assignments[]`, `follow_up_actions[]`
- **Connections:**
  - Provides `ai_outputParser` into **Hospital Orchestration Agent**
- **Edge cases / failures:**
  - If agent returns non-JSON or missing required fields → parser throws and workflow fails.
  - Schema is permissive for nested objects (`entities` is generic object; array items are generic objects), but top-level fields must exist.
- **Version notes:** typeVersion **1.3**.

#### Node: Staff Availability Lookup Tool
- **Type / role:** `LangChain Tool (Code)` (`@n8n/n8n-nodes-langchain.toolCode`) — tool callable by the agent.
- **Key configuration:**
  - Returns mocked JSON string with fields: `staff_id`, `available`, `current_workload`, `skills`, `shift_end`
  - Input referenced as `$input.staff_id` (LangChain tool input, not n8n `$json`).
- **Connections:**
  - Exposed as `ai_tool` to **Hospital Orchestration Agent**
- **Edge cases / failures:**
  - It’s mock data; production use requires replacing with real staff system integration.
  - If the agent calls it with unexpected input structure, tool may default to `'all'`.
- **Version notes:** typeVersion **1.3**.

#### Node: Calculator
- **Type / role:** `LangChain Calculator Tool` (`@n8n/n8n-nodes-langchain.toolCalculator`) — arithmetic tool for the agent.
- **Connections:**
  - Exposed as `ai_tool` to **Hospital Orchestration Agent**
- **Edge cases / failures:**
  - Typically minimal; failures arise if tool interface changes or agent misuses it.
- **Version notes:** typeVersion **1**.

---

### Block 3 — Action Routing & Execution (APIs + Priority Calculation)

**Overview:**  
Routes the agent’s required action to the appropriate external system. For task assignment, it enriches tasks with a weighted priority score before consolidation.

**Nodes involved:**
- Route by Action Type
- Schedule Appointment API
- Insurance Verification API
- Task Management API
- Calculate Task Priority Score

#### Node: Route by Action Type
- **Type / role:** `Switch` (n8n-nodes-base.switch) — routes by `action_required`.
- **Key configuration:**
  - Evaluates: `{{ $json.output.action_required }}`
  - Rules (string “contains”):
    - If contains `schedule_appointment` → output “Schedule Appointment”
    - If contains `verify_insurance` → output “Verify Insurance”
    - If contains `assign_task` → output “Assign Task”
  - Fallback output renamed: **“Other Actions”**
- **Connections:**
  - Input: from **Hospital Orchestration Agent**
  - Outputs:
    - Schedule Appointment → **Schedule Appointment API**
    - Verify Insurance → **Insurance Verification API**
    - Assign Task → **Task Management API**
    - Other Actions → **Merge Action Results** (input index 3)
- **Edge cases / failures:**
  - If `output.action_required` is missing (parser failure would usually stop earlier) or empty → goes to fallback.
  - “contains” can misroute if action strings overlap or contain multiple keywords.
- **Version notes:** typeVersion **3.4**.

#### Node: Schedule Appointment API
- **Type / role:** `HTTP Request` — posts follow-up actions to a scheduling system.
- **Key configuration:**
  - **POST** to placeholder URL: `<Appointment scheduling API endpoint>`
  - JSON body: `{{ $json.output.follow_up_actions }}`
  - Header: `Content-Type: application/json`
- **Connections:**
  - Input: from **Route by Action Type** (Schedule Appointment)
  - Output: to **Merge Action Results** (input index 0)
- **Edge cases / failures:**
  - Placeholder endpoint must be replaced.
  - Authentication not configured (likely required in real scheduling API).
  - If follow_up_actions is empty/invalid, scheduling API may reject (400).
- **Version notes:** typeVersion **4.3**.

#### Node: Insurance Verification API
- **Type / role:** `HTTP Request` — verifies insurance/eligibility using extracted entities.
- **Key configuration:**
  - **POST** to placeholder URL: `<Insurance verification API endpoint>`
  - JSON body: `{{ $json.output.entities }}`
  - Header: `Content-Type: application/json`
- **Connections:**
  - Input: from **Route by Action Type** (Verify Insurance)
  - Output: to **Merge Action Results** (input index 1)
- **Edge cases / failures:**
  - Placeholder endpoint must be replaced.
  - Likely requires payer-network auth, may return partial approvals/coverage limitations.
  - Ensure agent outputs only minimum necessary identifiers; masking is applied later but the request here is unmasked by this workflow.
- **Version notes:** typeVersion **4.3**.

#### Node: Task Management API
- **Type / role:** `HTTP Request` — creates/updates tasks in a task assignment system.
- **Key configuration:**
  - **POST** to placeholder URL: `<Task management API endpoint>`
  - JSON body: `{{ $json.output.task_assignments }}`
  - Header: `Content-Type: application/json`
- **Connections:**
  - Input: from **Route by Action Type** (Assign Task)
  - Output: to **Calculate Task Priority Score**
- **Edge cases / failures:**
  - Placeholder endpoint must be replaced.
  - In this workflow, **priority scoring happens after calling the Task API**, so tasks may be created without the computed `priority_score` unless your API itself calculates it or you reorder nodes.
- **Version notes:** typeVersion **4.3**.

#### Node: Calculate Task Priority Score
- **Type / role:** `Code` — enriches task assignments with a weighted priority score and metadata.
- **Key configuration (logic):**
  - Reads weights from `Workflow Configuration`:
    - `urgencyWeight`, `workloadWeight`, `skillMatchWeight`
  - Reads tasks from: `$input.item.json.task_assignments || []`
  - Computes:  
    `priorityScore = urgency*w1 + workload*w2 + skillMatch*w3`  
    rounds to 3 decimals, adds `calculation_metadata`, sets `priority_calculation_complete: true`
- **Connections:**
  - Input: from **Task Management API**
  - Output: to **Merge Action Results** (input index 2)
- **Edge cases / failures:**
  - If upstream item doesn’t contain `task_assignments` at the root (likely true here, because the agent output is under `$json.output.task_assignments`), the code will compute on an empty array. This is a structural mismatch risk.
  - Accessing `$('Workflow Configuration').first()` fails if node renamed or run data missing.
- **Version notes:** typeVersion **2**.

---

### Block 4 — Consolidation, PHI Masking, Webhook Response

**Overview:**  
Combines results from all action branches, applies PII/PHI masking recursively, and returns a JSON response.

**Nodes involved:**
- Merge Action Results
- Mask PII and PHI Data
- Respond to Webhook

#### Node: Merge Action Results
- **Type / role:** `Merge` — consolidates up to 4 inputs (schedule, insurance, task scoring, fallback).
- **Key configuration:**
  - `numberInputs = 4`
- **Connections:**
  - Inputs:
    - (0) from **Schedule Appointment API**
    - (1) from **Insurance Verification API**
    - (2) from **Calculate Task Priority Score**
    - (3) from **Route by Action Type** fallback (“Other Actions”)
  - Output: to **Mask PII and PHI Data**
- **Edge cases / failures:**
  - If only one branch runs per execution, merge behavior depends on n8n merge semantics and input availability; ensure it doesn’t wait indefinitely for missing inputs in your execution mode.
- **Version notes:** typeVersion **3.2**.

#### Node: Mask PII and PHI Data
- **Type / role:** `Code` — recursively sanitizes sensitive fields for HIPAA-style masking.
- **Key configuration (logic):**
  - Uses Node.js `crypto` to SHA-256 hash sensitive IDs to 16-hex substring.
  - Masks:
    - SSN-like fields → `XXX-XX-1234`
    - Credit card → `XXXX-XXXX-XXXX-1234`
    - patient names → initial + asterisks
    - patient IDs / MRN / medical record → `HASH_<hash>`
    - emails → keep domain, mask local part
    - phone numbers → keep last 4 digits
  - Recurses through nested objects/arrays.
- **Connections:**
  - Input: from **Merge Action Results**
  - Output: (not connected further in the JSON; response is handled separately by Respond node path)
- **Edge cases / failures:**
  - Masking is key-name based; if sensitive fields use different naming conventions, they may pass unmasked.
  - Potential over-masking: keys containing “name” + “patient” logic is simplistic and may miss variants.
- **Version notes:** typeVersion **2**.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` — returns HTTP response to the original caller.
- **Key configuration:**
  - Respond with: JSON
  - Response body: `{{ $json.output }}`
- **Connections:**
  - Input: from **Hospital Orchestration Agent**
- **Important behavior note:**
  - This responds with the **agent output** directly, not the merged/masked results. If the intention is “return merged + masked”, you would instead connect **Mask PII and PHI Data** to **Respond to Webhook** and respond with the sanitized merged payload.
- **Edge cases / failures:**
  - If the agent output contains PII, this response may expose it (despite later masking branch).
- **Version notes:** typeVersion **1.5**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Hospital Event Webhook | Webhook | Receives hospital events via HTTP POST | — | Workflow Configuration | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Workflow Configuration | Set | Adds workflow thresholds/weights and timing settings | Hospital Event Webhook | Hospital Orchestration Agent | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Hospital Orchestration Agent | LangChain Agent | Interprets event, decides action, outputs structured JSON | Workflow Configuration | Respond to Webhook; Route by Action Type | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Anthropic Chat Model | LangChain Anthropic Chat Model | Claude LLM inference backend | — | Hospital Orchestration Agent | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Structured Output Parser | LangChain Structured Output Parser | Enforces required JSON schema | — | Hospital Orchestration Agent | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Staff Availability Lookup Tool | LangChain Tool (Code) | Tool for staff availability/workload/skills lookup (mock) | — | Hospital Orchestration Agent | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Calculator | LangChain Tool (Calculator) | Arithmetic tool accessible by the agent | — | Hospital Orchestration Agent | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |
| Route by Action Type | Switch | Routes execution based on `output.action_required` | Hospital Orchestration Agent | Schedule Appointment API; Insurance Verification API; Task Management API; Merge Action Results | ## Multi-System Task Routing with Priority Calculation\n**Why:** Dynamically routes orchestrated actions to appropriate hospital systems (scheduling, task management, insurance verification) based on event type, calculates priority scores using business rules and calculator tools |
| Schedule Appointment API | HTTP Request | Sends follow-up actions to scheduling system | Route by Action Type | Merge Action Results | ## Multi-System Task Routing with Priority Calculation\n**Why:** Dynamically routes orchestrated actions to appropriate hospital systems (scheduling, task management, insurance verification) based on event type, calculates priority scores using business rules and calculator tools |
| Insurance Verification API | HTTP Request | Sends entities to insurance verification system | Route by Action Type | Merge Action Results | ## Multi-System Task Routing with Priority Calculation\n**Why:** Dynamically routes orchestrated actions to appropriate hospital systems (scheduling, task management, insurance verification) based on event type, calculates priority scores using business rules and calculator tools |
| Task Management API | HTTP Request | Sends task assignments to task management system | Route by Action Type | Calculate Task Priority Score | ## Multi-System Task Routing with Priority Calculation\n**Why:** Dynamically routes orchestrated actions to appropriate hospital systems (scheduling, task management, insurance verification) based on event type, calculates priority scores using business rules and calculator tools |
| Calculate Task Priority Score | Code | Computes weighted priority scores for tasks | Task Management API | Merge Action Results | ## Multi-System Task Routing with Priority Calculation\n**Why:** Dynamically routes orchestrated actions to appropriate hospital systems (scheduling, task management, insurance verification) based on event type, calculates priority scores using business rules and calculator tools |
| Merge Action Results | Merge | Consolidates outcomes from action branches | Schedule Appointment API; Insurance Verification API; Calculate Task Priority Score; Route by Action Type (fallback) | Mask PII and PHI Data | ## Result Consolidation with PHI Protection\n**Why:** Merges outcomes from multiple system interactions, applies data masking to protected health information for HIPAA compliance, responds to originating webhook with actionable results, creating complete audit trail while safeguarding patient privacy throughout the automated workflow. |
| Mask PII and PHI Data | Code | Recursively masks/hashes PII/PHI-like fields | Merge Action Results | — | ## Result Consolidation with PHI Protection\n**Why:** Merges outcomes from multiple system interactions, applies data masking to protected health information for HIPAA compliance, responds to originating webhook with actionable results, creating complete audit trail while safeguarding patient privacy throughout the automated workflow. |
| Respond to Webhook | Respond to Webhook | Returns JSON response to webhook caller | Hospital Orchestration Agent | — | ## Result Consolidation with PHI Protection\n**Why:** Merges outcomes from multiple system interactions, applies data masking to protected health information for HIPAA compliance, responds to originating webhook with actionable results, creating complete audit trail while safeguarding patient privacy throughout the automated workflow. |
| Sticky Note | Sticky Note | Commentary / prerequisites & benefits | — | — | ## Prerequisites\nActive Anthropic API account, hospital event management system with webhook capability\n## Use Cases\nPatient admission coordination, equipment failure response, code blue orchestration\n## Customization\nModify orchestration agent prompts for facility-specific protocols\n## Benefits\nReduces event response time by 75%, ensures consistent protocol adherence |
| Sticky Note1 | Sticky Note | Commentary / setup steps | — | — | ## Setup Steps\n1. Configure webhook URL endpoint for hospital event system integration\n2. Set up Anthropic API credentials for Claude model access in orchestration agent\n3. Configure Hospital Orchestration Agent Tool with your facility's event protocols\n4. Connect Schedule Appointment API with hospital scheduling system credentials\n5. Set up Task Management API integration for staff assignment system\n6. Configure Insurance Verification API with payer network access credentials |
| Sticky Note2 | Sticky Note | Commentary / how it works | — | — | ## How It Works\nThis workflow automates hospital operational event management by intelligently processing incoming events and orchestrating appropriate responses across multiple hospital systems. Designed for hospital operations managers, healthcare IT teams, and clinical administrators, it solves the complex challenge of coordinating rapid responses to diverse hospital events including patient admissions, equipment alerts, staffing emergencies, and clinical escalations. The system receives event triggers via webhook, uses AI-powered orchestration to analyze event context and determine required actions, then intelligently routes tasks to appropriate systems including appointment scheduling, task management, and insurance verification. It calculates priority scores, assigns tasks, verifies insurance coverage, and merges results while masking sensitive PHI data for compliance. The workflow leverages Anthropic's Claude and multiple AI tools to ensure context-aware decision-making aligned with hospital protocols. |
| Sticky Note3 | Sticky Note | Commentary / consolidation & privacy | — | — | ## Result Consolidation with PHI Protection\n**Why:** Merges outcomes from multiple system interactions, applies data masking to protected health information for HIPAA compliance, responds to originating webhook with actionable results, creating complete audit trail while safeguarding patient privacy throughout the automated workflow. |
| Sticky Note4 | Sticky Note | Commentary / routing & scoring | — | — | ## Multi-System Task Routing with Priority Calculation\n**Why:** Dynamically routes orchestrated actions to appropriate hospital systems (scheduling, task management, insurance verification) based on event type, calculates priority scores using business rules and calculator tools |
| Sticky Note5 | Sticky Note | Commentary / reception & AI analysis | — | — | ## Event Reception and AI-Powered Analysis\n**Why:** Captures hospital events through webhook triggers and processes them through AI orchestration agent using Claude and specialized lookup tools, ensuring intelligent interpretation of event context, urgency, and required response pathways based on hospital operational protocols. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Hospital Operations and Patient Care Orchestration Agent”**.

2. **Add node: Webhook**
   - Name: **Hospital Event Webhook**
   - Method: **POST**
   - Path: **hospital-orchestration**
   - Response Mode: **Last Node**
   - Save to generate the production URL; configure your hospital event system to POST JSON to it.

3. **Add node: Set**
   - Name: **Workflow Configuration**
   - Enable **Include Other Fields**
   - Add fields:
     - `dischargeReadinessThreshold` (number) = `0.85`
     - `taskPriorityWeightUrgency` (number) = `0.4`
     - `taskPriorityWeightWorkload` (number) = `0.3`
     - `taskPriorityWeightSkillMatch` (number) = `0.3`
     - `followUpWindowDays` (number) = `7`
     - `medicationReminderHours` (number) = `24`
     - `complianceAuditEnabled` (boolean) = `true`
   - Connect: **Hospital Event Webhook → Workflow Configuration**

4. **Add node: LangChain Agent**
   - Name: **Hospital Orchestration Agent**
   - Input text: set to expression `{{ $json }}`
   - Prompt type: **Define**
   - System message: paste the workflow’s system message and ensure expressions reference the **Workflow Configuration** node (keep node name exact or update expressions accordingly).
   - Connect: **Workflow Configuration → Hospital Orchestration Agent**

5. **Add node: Anthropic Chat Model**
   - Name: **Anthropic Chat Model**
   - Model: **claude-3-5-sonnet-20241022**
   - Credentials: create/select **Anthropic API** credential (API key from Anthropic).
   - Connect model to agent using the **AI Language Model** connection:  
     **Anthropic Chat Model → Hospital Orchestration Agent (ai_languageModel)**

6. **Add node: Structured Output Parser**
   - Name: **Structured Output Parser**
   - Schema type: **Manual**
   - Paste the JSON schema requiring: `workflow_state`, `intent`, `entities`, `action_required`, `message_to_patient_or_staff`, `audit_metadata`, `confidence_level`, `risk_flags`, `task_assignments`, `follow_up_actions`.
   - Connect parser to agent using **AI Output Parser** connection:  
     **Structured Output Parser → Hospital Orchestration Agent (ai_outputParser)**

7. **Add tools for the agent**
   - **Tool 1: LangChain Tool (Code)**
     - Name: **Staff Availability Lookup Tool**
     - Paste code that returns staff availability JSON (replace mock with real integration later).
     - Connect as tool: **Staff Availability Lookup Tool → Hospital Orchestration Agent (ai_tool)**
   - **Tool 2: LangChain Calculator**
     - Name: **Calculator**
     - Connect as tool: **Calculator → Hospital Orchestration Agent (ai_tool)**

8. **Add node: Respond to Webhook**
   - Name: **Respond to Webhook**
   - Respond with: **JSON**
   - Response Body: expression `{{ $json.output }}`
   - Connect: **Hospital Orchestration Agent → Respond to Webhook**

9. **Add node: Switch**
   - Name: **Route by Action Type**
   - Add rules (String → Contains) on left value `{{ $json.output.action_required }}`
     - contains `schedule_appointment` → output “Schedule Appointment”
     - contains `verify_insurance` → output “Verify Insurance”
     - contains `assign_task` → output “Assign Task”
   - Enable fallback output renamed to **Other Actions**
   - Connect: **Hospital Orchestration Agent → Route by Action Type**

10. **Add node: HTTP Request (Scheduling)**
    - Name: **Schedule Appointment API**
    - Method: **POST**
    - URL: your scheduling endpoint
    - Send JSON body: expression `{{ $json.output.follow_up_actions }}`
    - Header: `Content-Type: application/json`
    - Connect: **Route by Action Type (Schedule Appointment) → Schedule Appointment API**

11. **Add node: HTTP Request (Insurance)**
    - Name: **Insurance Verification API**
    - Method: **POST**
    - URL: your insurance verification endpoint
    - Send JSON body: expression `{{ $json.output.entities }}`
    - Header: `Content-Type: application/json`
    - Connect: **Route by Action Type (Verify Insurance) → Insurance Verification API**

12. **Add node: HTTP Request (Tasks)**
    - Name: **Task Management API**
    - Method: **POST**
    - URL: your task management endpoint
    - Send JSON body: expression `{{ $json.output.task_assignments }}`
    - Header: `Content-Type: application/json`
    - Connect: **Route by Action Type (Assign Task) → Task Management API**

13. **Add node: Code (Task scoring)**
    - Name: **Calculate Task Priority Score**
    - Mode: **Run once for each item**
    - Paste the weighted scoring code using `$('Workflow Configuration').first().json` weights.
    - Connect: **Task Management API → Calculate Task Priority Score**

14. **Add node: Merge**
    - Name: **Merge Action Results**
    - Number of inputs: **4**
    - Connect:
      - **Schedule Appointment API → Merge Action Results (input 0)**
      - **Insurance Verification API → Merge Action Results (input 1)**
      - **Calculate Task Priority Score → Merge Action Results (input 2)**
      - **Route by Action Type (Other Actions/fallback) → Merge Action Results (input 3)**

15. **Add node: Code (Masking)**
    - Name: **Mask PII and PHI Data**
    - Paste the recursive masking/hashing code.
    - Connect: **Merge Action Results → Mask PII and PHI Data**

16. **(Recommended adjustment for compliance)**
    - If you want the webhook response to always be masked/merged, connect **Mask PII and PHI Data → Respond to Webhook** and set the response body to the sanitized merged payload (instead of `{{ $json.output }}` from the agent).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Active Anthropic API account, hospital event management system with webhook capability | Prerequisites |
| Use cases: patient admission coordination, equipment failure response, code blue orchestration | Workflow applicability |
| Customization: modify orchestration agent prompts for facility-specific protocols | Agent system message is the main policy surface |
| Benefits: reduces event response time by 75%, ensures consistent protocol adherence | Claimed outcome in notes |
| Setup steps 1–6: webhook integration, Anthropic creds, customize agent tool/prompt, connect 3 external APIs | Deployment checklist |
| “How it works” description: webhook → AI orchestration → routing to scheduling/tasks/insurance → merge → PHI masking | High-level operating model |
| Result consolidation with PHI protection: merge outcomes, apply masking, respond with audit trail | Intended compliance posture (note: current response node returns agent output unless adjusted) |

