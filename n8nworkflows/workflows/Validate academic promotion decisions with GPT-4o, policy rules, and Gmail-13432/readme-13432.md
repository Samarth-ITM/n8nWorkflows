Validate academic promotion decisions with GPT-4o, policy rules, and Gmail

https://n8nworkflows.xyz/workflows/validate-academic-promotion-decisions-with-gpt-4o--policy-rules--and-gmail-13432


# Validate academic promotion decisions with GPT-4o, policy rules, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI-driven academic promotion governance and performance validation system  
**User-provided title:** Validate academic promotion decisions with GPT-4o, policy rules, and Gmail

**Purpose:**  
This workflow runs on a schedule to validate academic promotion decisions using (1) performance data, (2) institutional policy rules, and (3) a multi-agent AI governance layer powered by GPT‑4o. It routes outcomes into approved/rejected/escalation paths, stores results in n8n Data Tables, and escalates edge cases to HR via email with a human-in-the-loop wait gate.

**Target use cases:**
- Annual/quarterly promotion governance cycles
- Consistency calibration across departments
- Policy compliance verification with an audit trail
- HR escalation for ambiguous or exception cases

### 1.1 Scheduled Initiation & Runtime Configuration
Runs daily (at a configured hour) and sets runtime variables like API URLs, thresholds, and HR escalation email.

### 1.2 Data Acquisition (Performance + Policy Rules)
Fetches performance data and policy rules from external APIs via HTTP Request nodes.

### 1.3 AI Governance Orchestration (Agents + Tools + Parsers)
A central Governance Agent (GPT‑4o) orchestrates:
- Performance validation tool (structured output)
- Calibration tool (structured output)
- Policy compliance tool (code-based rules evaluation)

### 1.4 Decision Routing & Persistence
Routes final AI decision into Approved / Rejected / Escalation; stores outcomes in Data Tables; escalations go through a Wait node and send an HR email; merges all outcomes into a single audit trail store.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Initiation & Configuration
**Overview:**  
Triggers the workflow on a schedule and defines configuration values used downstream (API endpoints, thresholds, HR email).

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration:** Triggers at **09:00** (server/workflow timezone applies).
- **Connections:**  
  - **Output →** Workflow Configuration
- **Version considerations:** typeVersion **1.3**.
- **Edge cases / failures:**  
  - Timezone mismatch can cause unexpected run time.
  - If instance is down at trigger time, run may be missed depending on n8n scheduling behavior.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — centralizes configurable constants.
- **Configuration choices (interpreted):**
  - Defines:
    - `performanceDataApiUrl` (placeholder)
    - `policyRulesApiUrl` (placeholder)
    - `hrEscalationEmail` (placeholder)
    - Threshold numbers:
      - `minResearchOutputs = 5`
      - `minTeachingScore = 7.5`
      - `minServiceContributions = 3`
  - **Include other fields:** enabled (keeps any incoming JSON).
- **Key expressions/variables:** downstream nodes reference values via:
  - `$('Workflow Configuration').first().json.<field>`
- **Connections:**  
  - **Output →** Fetch Performance Data  
  - **Output →** Fetch Policy Rules
- **Version considerations:** typeVersion **3.4**.
- **Edge cases / failures:**  
  - Placeholder URLs/emails must be replaced or downstream HTTP/email nodes fail.
  - Thresholds set here are *not automatically used* in the policy checker code (that tool uses `rules.minTeachingScore`, etc.). If you expect these Set values to drive policy, you must merge them into policy rules or update the code/tool prompts.

---

### Block 2 — Data Acquisition (External APIs)
**Overview:**  
Loads the candidate performance dataset and institutional policy rules as JSON.

**Nodes involved:**
- Fetch Performance Data
- Fetch Policy Rules

#### Node: Fetch Performance Data
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves performance dataset.
- **Configuration choices:**
  - URL: `={{ $('Workflow Configuration').first().json.performanceDataApiUrl }}`
  - Response format: JSON
- **Connections:**  
  - **Input ←** Workflow Configuration  
  - **Output →** Governance Agent
- **Version considerations:** typeVersion **4.3**.
- **Edge cases / failures:**
  - 4xx/5xx responses, timeouts, DNS errors.
  - Non-JSON response breaks downstream assumptions (agent expects structured inputs).
  - Missing auth headers (not configured here) if API requires credentials.

#### Node: Fetch Policy Rules
- **Type / role:** `n8n-nodes-base.httpRequest` — retrieves policy rules as JSON.
- **Configuration choices:**
  - URL: `={{ $('Workflow Configuration').first().json.policyRulesApiUrl }}`
  - Response format: JSON
- **Connections:**  
  - **Input ←** Workflow Configuration  
  - **Output:** **not connected** to downstream nodes in the provided workflow graph.
- **Version considerations:** typeVersion **4.3**.
- **Edge cases / failures:**
  - Same HTTP risks as above.
  - **Design gap:** This node fetches policy rules but they are never passed into the Governance Agent path. The Policy Compliance Checker Tool expects `policyRules` from `$fromAI(...)`, but there is no wiring that supplies those rules into the agent context. To make policy checks real, you must pass the policy rules into the Governance Agent (see Section 4).

---

### Block 3 — AI Governance Orchestration (LLMs, Tools, Structured Parsers)
**Overview:**  
Uses GPT‑4o to orchestrate decision-making. Specialized tools validate performance, calibrate fairness, and check policy compliance; structured parsers enforce machine-readable outputs.

**Nodes involved:**
- Governance Agent
- OpenAI Model - Governance Agent
- Performance Signal Agent Tool
- OpenAI Model - Performance Agent
- Performance Validation Output Parser
- Calibration Agent Tool
- OpenAI Model - Calibration Agent
- Calibration Output Parser
- Policy Compliance Checker Tool
- Governance Decision Output Parser

#### Node: OpenAI Model - Governance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for the Governance Agent.
- **Configuration choices:**
  - Model: **gpt-4o**
  - Temperature: **0.1** (low variance; more deterministic)
- **Credentials:** OpenAI API credential “OpenAi account”.
- **Connections:**  
  - **Output (ai_languageModel) →** Governance Agent
- **Version considerations:** typeVersion **1.3**.
- **Edge cases / failures:** API quota, model access restrictions, credential failure.

#### Node: Governance Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrator agent that can call tools and produce a final structured decision.
- **Configuration choices:**
  - Input text: `={{ $json }}` (passes incoming JSON to the agent prompt context)
  - System message defines responsibilities and decision categories (APPROVED/REJECTED/ESCALATION_REQUIRED)
  - Has output parser enabled
- **Connections:**
  - **Input ←** Fetch Performance Data
  - **Tools (ai_tool) available to agent:**  
    - Performance Signal Agent Tool  
    - Calibration Agent Tool  
    - Policy Compliance Checker Tool
  - **Output (main) →** Route by Decision
  - **Output parser (ai_outputParser) ←** Governance Decision Output Parser
  - **Language model (ai_languageModel) ←** OpenAI Model - Governance Agent
- **Version considerations:** typeVersion **3.1**.
- **Edge cases / failures:**
  - Agent may fail to call tools as intended if tool inputs are not present in context.
  - Output parser failures if the agent returns non-conforming JSON.
  - Large payloads from performance API can exceed token/context limits.

#### Node: Governance Decision Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured JSON output for final decision.
- **Configuration choices:**
  - Schema example includes keys like `decision`, `eligibilityStatus`, `calibrationCheckPassed`, `policyComplianceStatus`, `escalationRequired`, `auditTrail`, etc.
- **Connections:**  
  - **Output (ai_outputParser) →** Governance Agent
- **Version considerations:** typeVersion **1.3**.
- **Edge cases / failures:** If Governance Agent returns missing/renamed fields, routing and email templating may break (e.g., `$json.output.decision` undefined).

#### Node: Performance Signal Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by the Governance Agent to validate performance evidence vs thresholds.
- **Configuration choices:**
  - Tool input:  
    `={{ $fromAI('performanceData', 'Performance data including peer reviews, research outputs, teaching evaluations, and service contributions', 'json') }}`
  - System message includes explicit thresholds (peer review ≥7.0; outputs ≥5; teaching ≥7.5; service ≥3) and emphasizes “objective assessment”.
  - Has output parser enabled.
- **Connections:**
  - **Language model (ai_languageModel) ←** OpenAI Model - Performance Agent
  - **Output parser (ai_outputParser) ←** Performance Validation Output Parser
  - **Tool availability (ai_tool) →** Governance Agent
- **Version considerations:** typeVersion **3**.
- **Edge cases / failures:**
  - `$fromAI('performanceData', ...)` depends on the agent context containing a compatible `performanceData` object. In the current workflow, only `Fetch Performance Data` feeds the agent; if its JSON shape doesn’t match, the tool may hallucinate or fail parsing.
  - Parser mismatch if tool response isn’t valid JSON.

#### Node: OpenAI Model - Performance Agent
- **Type / role:** LLM provider for the Performance Signal Agent Tool.
- **Configuration:** gpt-4o, temperature 0.1.
- **Connections:** **ai_languageModel →** Performance Signal Agent Tool
- **Edge cases:** same as other OpenAI nodes (auth/quota/model).

#### Node: Performance Validation Output Parser
- **Type / role:** Structured output parser for performance validation.
- **Schema example keys:** `validationStatus`, `peerReviewScore`, `researchOutputsCount`, `teachingEvaluationScore`, `serviceContributionsCount`, `flaggedIssues`, `overallAssessment`, `reasoning`.
- **Connections:** **ai_outputParser →** Performance Signal Agent Tool
- **Edge cases:** response must conform; otherwise agent tool fails.

#### Node: Calibration Agent Tool
- **Type / role:** `agentTool` — calibrates results vs institutional standards/historical cases.
- **Configuration choices:**
  - Tool input:  
    `={{ $fromAI('validationResults', 'Performance validation results to calibrate against institutional standards', 'json') }}`
  - System message: fairness/consistency emphasis; detects deviations and departmental bias.
  - Has output parser enabled.
- **Connections:**
  - **Language model (ai_languageModel) ←** OpenAI Model - Calibration Agent
  - **Output parser (ai_outputParser) ←** Calibration Output Parser
  - **Tool availability (ai_tool) →** Governance Agent
- **Edge cases / failures:**
  - Requires `validationResults` to exist in agent context. If Governance Agent doesn’t pass tool outputs correctly or naming differs, calibration may be unreliable.

#### Node: OpenAI Model - Calibration Agent
- **Type / role:** LLM provider for calibration tool.
- **Configuration:** gpt-4o, temperature 0.1.
- **Connections:** **ai_languageModel →** Calibration Agent Tool

#### Node: Calibration Output Parser
- **Type / role:** Structured parser for calibration results.
- **Schema example keys:** `calibrationStatus`, `consistencyScore`, `comparisonWithPeers`, `deviationFlags`, `calibrationNotes`, `reasoning`.
- **Connections:** **ai_outputParser →** Calibration Agent Tool

#### Node: Policy Compliance Checker Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCode` — deterministic JavaScript compliance evaluation callable by the Governance Agent.
- **Configuration choices (interpreted):**
  - Reads:
    - `candidateData` via `$fromAI('candidateData', ..., 'json')`
    - `policyRules` via `$fromAI('policyRules', ..., 'json')`
  - Performs checks:
    - `yearsInCurrentRank >= rules.minYearsInRank (default 5)`
    - `researchOutputsCount >= rules.minPublications (default 5)`
    - `teachingEvaluationScore >= rules.minTeachingScore (default 7.5)`
    - `serviceContributionsCount >= rules.minServiceContributions (default 3)`
    - no ethics violations
    - `departmentApprovalStatus === 'APPROVED'`
  - Returns JSON string with `complianceStatus`, `failedChecks`, `policyViolations`, etc.  
  - Catches errors and returns `{ complianceStatus: 'ERROR', ... }`
- **Connections:** **ai_tool →** Governance Agent
- **Version considerations:** typeVersion **1.3**.
- **Edge cases / failures:**
  - If `candidateData`/`policyRules` are missing or malformed, tool returns `ERROR`.
  - Current workflow does not reliably supply `policyRules` (Fetch Policy Rules is disconnected), so compliance may frequently error or be fabricated by the agent unless fixed.

---

### Block 4 — Decision Routing & Outcome Handling
**Overview:**  
Routes the governance decision into Approved/Rejected/Escalation branches. Approved and rejected are stored immediately; escalation pauses for HR review and then emails HR.

**Nodes involved:**
- Route by Decision
- Store Approved Promotions
- Store Rejected Cases
- Wait for HR Review
- Send HR Escalation Email

#### Node: Route by Decision
- **Type / role:** `n8n-nodes-base.switch` — conditional routing based on AI output.
- **Configuration choices:**
  - Evaluates `={{ $json.output.decision }}`
  - Routes to:
    - Approved if equals `APPROVED`
    - Rejected if equals `REJECTED`
    - Escalation if equals `ESCALATION_REQUIRED`
  - Fallback output named **Unprocessed**
- **Connections:**
  - **Input ←** Governance Agent
  - **Approved →** Store Approved Promotions
  - **Rejected →** Store Rejected Cases
  - **Escalation →** Wait for HR Review
- **Version considerations:** typeVersion **3.4**.
- **Edge cases / failures:**
  - If `output.decision` missing, item goes to **Unprocessed** (currently not connected; outcome is dropped).

#### Node: Store Approved Promotions
- **Type / role:** `n8n-nodes-base.dataTable` — persists approved outcomes.
- **Configuration choices:** auto-maps all incoming fields into table columns.
- **Requires:** Data Table ID placeholder must be replaced.
- **Connections:**
  - **Input ←** Route by Decision (Approved)
  - **Output →** Merge All Outcomes (input 0)
- **Edge cases:** schema drift; table permissions; missing table ID.

#### Node: Store Rejected Cases
- **Type / role:** `n8n-nodes-base.dataTable` — persists rejected outcomes.
- **Configuration:** auto-map; Data Table ID placeholder.
- **Connections:**
  - **Input ←** Route by Decision (Rejected)
  - **Output →** Merge All Outcomes (input 1)

#### Node: Wait for HR Review
- **Type / role:** `n8n-nodes-base.wait` — human-in-the-loop gate.
- **Configuration choices:**
  - Resume mode: **webhook**
  - HTTP method: **POST**
  - A webhook ID is generated by n8n (used internally).
- **Connections:**
  - **Input ←** Route by Decision (Escalation)
  - **Output →** Send HR Escalation Email
- **Edge cases / failures:**
  - If nobody calls the resume webhook, execution remains waiting indefinitely (unless instance has retention/timeouts configured).
  - Requires exposing the webhook URL appropriately (network/public access).

#### Node: Send HR Escalation Email
- **Type / role:** `n8n-nodes-base.emailSend` — notifies HR by email for manual review.
- **Configuration choices (interpreted):**
  - To: `={{ $('Workflow Configuration').first().json.hrEscalationEmail }}`
  - Subject includes candidate name: `ESCALATION REQUIRED...`
  - HTML template references:
    - `$json.output.candidateName`, `department`, `currentRank`, `proposedRank`
    - `$json.output.reasoning`, `eligibilityStatus`, `policyComplianceStatus`
    - `$json.output.recommendedActions` (mapped into `<li>` items)
    - `$json.output.auditTrail` JSON-stringified
  - From: placeholder sender email (must be set)
- **Connections:**
  - **Input ←** Wait for HR Review
  - **Output →** Merge All Outcomes (input 2)
- **Version considerations:** typeVersion **2.1**.
- **Edge cases / failures:**
  - If `$json.output.*` fields are absent (parser mismatch), email renders with “Not specified” or may error for non-array `recommendedActions`.
  - SMTP/Gmail configuration errors (auth, less-secure policies, app password, rate limits).

---

### Block 5 — Consolidation & Audit Storage
**Overview:**  
Merges approved/rejected/escalation paths into one stream and stores an audit trail record.

**Nodes involved:**
- Merge All Outcomes
- Store Audit Trail

#### Node: Merge All Outcomes
- **Type / role:** `n8n-nodes-base.merge` — collects outputs from the three branches.
- **Configuration:** `numberInputs = 3`
- **Connections:**
  - **Input 0 ←** Store Approved Promotions
  - **Input 1 ←** Store Rejected Cases
  - **Input 2 ←** Send HR Escalation Email
  - **Output →** Store Audit Trail
- **Version considerations:** typeVersion **3.2**.
- **Edge cases / failures:**
  - Merge behavior depends on node mode (not explicitly specified here). With multiple independent branches, items may pass through as they arrive, but ordering may be non-deterministic.
  - If one branch produces no items, ensure merge mode still emits items from other inputs (otherwise executions can appear “stuck” waiting for all inputs).

#### Node: Store Audit Trail
- **Type / role:** `n8n-nodes-base.dataTable` — persists final record(s) for traceability.
- **Configuration:** auto-map; Data Table ID placeholder.
- **Connections:**  
  - **Input ←** Merge All Outcomes
- **Edge cases:** same as other Data Tables (ID/permissions/schema).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled entry point | — | Workflow Configuration | ## How It Works<br>This workflow automates performance governance and policy compliance monitoring for HR leaders, talent managers, and organizational development teams across enterprises. It solves the challenge of maintaining consistent performance standards while ensuring human judgment on promotion and termination decisions. Scheduled triggers initiate governance cycles that fetch performance data and policy rules, then orchestrate specialized AI agents working in parallel: governance assessment evaluates policy adherence, performance validation verifies metric accuracy, and calibration analysis ensures rating consistency across departments. A policy compliance checker synthesizes findings and routes outcomes intelligently—approved promotions store automatically, while exceptions requiring HR review trigger human approval gates before case creation and email escalation. |
| Workflow Configuration | set | Defines URLs, thresholds, HR email | Schedule Trigger | Fetch Performance Data; Fetch Policy Rules | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final.<br><br>## How It Works<br>This workflow automates performance governance and policy compliance monitoring for HR leaders, talent managers, and organizational development teams across enterprises. It solves the challenge of maintaining consistent performance standards while ensuring human judgment on promotion and termination decisions. Scheduled triggers initiate governance cycles that fetch performance data and policy rules, then orchestrate specialized AI agents working in parallel: governance assessment evaluates policy adherence, performance validation verifies metric accuracy, and calibration analysis ensures rating consistency across departments. A policy compliance checker synthesizes findings and routes outcomes intelligently—approved promotions store automatically, while exceptions requiring HR review trigger human approval gates before case creation and email escalation. |
| Fetch Performance Data | httpRequest | Pulls performance dataset | Workflow Configuration | Governance Agent | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final.<br><br>## How It Works<br>…(same as above)… |
| Fetch Policy Rules | httpRequest | Pulls policy rules dataset | Workflow Configuration | *(not connected)* | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final.<br><br>## How It Works<br>…(same as above)… |
| OpenAI Model - Governance Agent | lmChatOpenAi | LLM for governance agent | — | Governance Agent (ai_languageModel) | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final. |
| Governance Agent | langchain agent | Orchestrates tools and returns final decision | Fetch Performance Data | Route by Decision | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final. |
| Performance Signal Agent Tool | agentTool | Validates metrics vs thresholds | Governance Agent (ai_tool) | Governance Agent (tool result) | ## Performance Validation<br>Ensures decisions rest on accurate, defensible evidence that withstands scrutiny during disputes. |
| OpenAI Model - Performance Agent | lmChatOpenAi | LLM for performance validation tool | — | Performance Signal Agent Tool (ai_languageModel) | ## Performance Validation<br>Ensures decisions rest on accurate, defensible evidence that withstands scrutiny during disputes. |
| Performance Validation Output Parser | outputParserStructured | Enforces structured validation JSON | — | Performance Signal Agent Tool (ai_outputParser) | ## Performance Validation<br>Ensures decisions rest on accurate, defensible evidence that withstands scrutiny during disputes. |
| Calibration Agent Tool | agentTool | Calibrates consistency/fairness | Governance Agent (ai_tool) | Governance Agent (tool result) | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| OpenAI Model - Calibration Agent | lmChatOpenAi | LLM for calibration tool | — | Calibration Agent Tool (ai_languageModel) | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Calibration Output Parser | outputParserStructured | Enforces structured calibration JSON | — | Calibration Agent Tool (ai_outputParser) | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Policy Compliance Checker Tool | toolCode | Deterministic policy checks in JS | Governance Agent (ai_tool) | Governance Agent (tool result) | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final. |
| Governance Decision Output Parser | outputParserStructured | Enforces structured final decision JSON | — | Governance Agent (ai_outputParser) | ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final. |
| Route by Decision | switch | Branches by decision | Governance Agent | Store Approved Promotions; Store Rejected Cases; Wait for HR Review | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Store Approved Promotions | dataTable | Persist approved decisions | Route by Decision (Approved) | Merge All Outcomes | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Store Rejected Cases | dataTable | Persist rejected decisions | Route by Decision (Rejected) | Merge All Outcomes | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Wait for HR Review | wait | Human approval gate via webhook resume | Route by Decision (Escalation) | Send HR Escalation Email | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Send HR Escalation Email | emailSend | Emails HR escalation summary | Wait for HR Review | Merge All Outcomes | ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. |
| Merge All Outcomes | merge | Consolidates all outcomes | Store Approved Promotions; Store Rejected Cases; Send HR Escalation Email | Store Audit Trail |  |
| Store Audit Trail | dataTable | Persist consolidated audit record | Merge All Outcomes | — |  |

Additional sticky notes not clearly bound to specific nodes (still preserved in Section 5):
- “Prerequisites…”
- “Setup Steps…”

---

## 4. Reproducing the Workflow from Scratch

1. **Create the trigger**
   1) Add **Schedule Trigger**  
   2) Configure it to run at **09:00** (adjust timezone/workflow settings as needed).

2. **Add configuration constants**
   1) Add a **Set** node named **Workflow Configuration**  
   2) Add fields:
      - `performanceDataApiUrl` (string) → your performance API endpoint
      - `policyRulesApiUrl` (string) → your policy rules endpoint
      - `hrEscalationEmail` (string) → HR mailbox/group
      - `minResearchOutputs` (number) = 5
      - `minTeachingScore` (number) = 7.5
      - `minServiceContributions` (number) = 3
   3) Enable **Include Other Fields**
   4) Connect **Schedule Trigger → Workflow Configuration**

3. **Fetch external data**
   1) Add **HTTP Request** named **Fetch Performance Data**
      - URL: `{{$('Workflow Configuration').first().json.performanceDataApiUrl}}`
      - Response: JSON  
   2) Add **HTTP Request** named **Fetch Policy Rules**
      - URL: `{{$('Workflow Configuration').first().json.policyRulesApiUrl}}`
      - Response: JSON  
   3) Connect **Workflow Configuration → Fetch Performance Data**  
   4) Connect **Workflow Configuration → Fetch Policy Rules**

4. **(Important) Combine performance + policy into one payload for the agent**
   - In the provided workflow, **Fetch Policy Rules is not connected** to the Governance Agent. To reproduce a *working* policy check:
   1) Add a **Merge** node (mode: *Merge by Position* or equivalent) named e.g. **Merge Inputs for Agent**  
   2) Connect:
      - **Fetch Performance Data → Merge Inputs for Agent (Input 1)**
      - **Fetch Policy Rules → Merge Inputs for Agent (Input 2)**
   3) Add a **Set** node named e.g. **Build Agent Context** after the merge, producing a single object such as:
      - `performanceData` = performance JSON
      - `policyRules` = policy JSON  
      (Exact mapping depends on the merge output shape.)
   4) This **Build Agent Context** should feed **Governance Agent**.
   
   If you want to match the JSON exactly as provided, you can skip this step, but then policy compliance will be unreliable.

5. **Create the language models (OpenAI)**
   1) Add **OpenAI Chat Model** node named **OpenAI Model - Governance Agent**
      - Model: `gpt-4o`
      - Temperature: `0.1`
      - Credentials: configure **OpenAI API** credential
   2) Add **OpenAI Chat Model** node named **OpenAI Model - Performance Agent** (same model/temp/credential)
   3) Add **OpenAI Chat Model** node named **OpenAI Model - Calibration Agent** (same model/temp/credential)

6. **Create the structured output parsers**
   1) Add **Structured Output Parser** named **Performance Validation Output Parser** (use the schema example from the workflow)
   2) Add **Structured Output Parser** named **Calibration Output Parser**
   3) Add **Structured Output Parser** named **Governance Decision Output Parser**

7. **Create agent tools**
   1) Add **Agent Tool** named **Performance Signal Agent Tool**
      - Tool description: performance validation
      - System message: include the validation instructions + thresholds
      - Tool input expression uses `$fromAI('performanceData', ..., 'json')`
      - Attach:
        - **OpenAI Model - Performance Agent → Performance Signal Agent Tool (ai_languageModel)**
        - **Performance Validation Output Parser → Performance Signal Agent Tool (ai_outputParser)**
   2) Add **Agent Tool** named **Calibration Agent Tool**
      - Tool input uses `$fromAI('validationResults', ..., 'json')`
      - Attach:
        - **OpenAI Model - Calibration Agent → Calibration Agent Tool (ai_languageModel)**
        - **Calibration Output Parser → Calibration Agent Tool (ai_outputParser)**
   3) Add **Code Tool** named **Policy Compliance Checker Tool**
      - Paste the JS code from the workflow (adapt if you change field names)
      - Ensure the agent context provides `candidateData` and `policyRules` in a consistent way.

8. **Create the governance agent**
   1) Add **AI Agent** named **Governance Agent**
      - System message: governance orchestration instructions and decision categories
      - Input: pass the prepared JSON context (e.g., the output of **Build Agent Context**)
      - Enable output parser
   2) Connect:
      - **OpenAI Model - Governance Agent → Governance Agent (ai_languageModel)**
      - **Governance Decision Output Parser → Governance Agent (ai_outputParser)**
      - **Performance Signal Agent Tool → Governance Agent (ai_tool)**
      - **Calibration Agent Tool → Governance Agent (ai_tool)**
      - **Policy Compliance Checker Tool → Governance Agent (ai_tool)**

9. **Decision routing**
   1) Add **Switch** node named **Route by Decision**
      - Rule 1: `{{$json.output.decision}} == "APPROVED"` → output “Approved”
      - Rule 2: equals `"REJECTED"` → “Rejected”
      - Rule 3: equals `"ESCALATION_REQUIRED"` → “Escalation”
      - Fallback: “Unprocessed”
   2) Connect **Governance Agent → Route by Decision**

10. **Persistence tables**
   1) Add **Data Table** node **Store Approved Promotions**
      - Choose/create a Data Table and set its ID
      - Mapping mode: Auto-map
   2) Add **Data Table** node **Store Rejected Cases** (table ID required)
   3) Add **Data Table** node **Store Audit Trail** (table ID required)

11. **Human gate + HR email**
   1) Add **Wait** node **Wait for HR Review**
      - Resume: Webhook
      - Method: POST
   2) Add **Send Email** node **Send HR Escalation Email**
      - To: `{{$('Workflow Configuration').first().json.hrEscalationEmail}}`
      - From: configure a valid sender
      - Subject/HTML: replicate the template (ensure fields exist in `output`)
      - Credentials: configure SMTP/Gmail (app password if required)
   3) Connect:
      - **Route by Decision (Escalation) → Wait for HR Review → Send HR Escalation Email**

12. **Merge outcomes and store audit**
   1) Add **Merge** node **Merge All Outcomes**
      - Number of inputs: 3
   2) Connect:
      - **Store Approved Promotions → Merge All Outcomes (Input 1)**
      - **Store Rejected Cases → Merge All Outcomes (Input 2)**
      - **Send HR Escalation Email → Merge All Outcomes (Input 3)**
   3) Connect **Merge All Outcomes → Store Audit Trail**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Prerequisites<br>API key, performance management system data access, Gmail account with app password<br>## Use Cases<br>Annual performance review calibration, promotion decision validation<br>## Customization<br>Integrate HRIS for live performance data, add custom policy rule engines<br>## Benefits<br>Reduces governance review time by 70%, ensures consistent policy application | Sticky note (general) |
| ## Setup Steps<br>1. Configure API credentials with Llama-3.1-70B-Instruct model access<br>2. Set up schedule trigger aligned with review cycles (quarterly/annual)<br>3. Configure decision routing logic for approved versus exception cases<br>4. Connect Gmail for HR escalation alerts to designated reviewers<br>5. Set up Google Sheets for storing approved promotions and audit trails | Sticky note (general). Note: the actual workflow uses **GPT‑4o** and **n8n Data Tables**, not Llama-3.1 or Google Sheets. |
| ## How It Works<br>(full workflow description text preserved above in the Summary Table) | Sticky note (general) |
| ## Governance Assessment<br>Identifies rating inconsistencies and bias patterns requiring intervention before decisions become final. | Sticky note (block label) |
| ## Performance Validation<br>Ensures decisions rest on accurate, defensible evidence that withstands scrutiny during disputes. | Sticky note (block label) |
| ## Calibration Analysis<br>Maintains organizational fairness and prevents manager bias from creating inequitable outcomes. | Sticky note (block label) |