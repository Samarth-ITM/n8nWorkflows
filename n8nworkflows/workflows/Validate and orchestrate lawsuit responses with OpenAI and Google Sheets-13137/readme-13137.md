Validate and orchestrate lawsuit responses with OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/validate-and-orchestrate-lawsuit-responses-with-openai-and-google-sheets-13137


# Validate and orchestrate lawsuit responses with OpenAI and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name (JSON):** Automated compliance monitoring with AI risk assessment  
**User-provided title:** Validate and orchestrate lawsuit responses with OpenAI and Google Sheets

### Purpose
This workflow receives an execution request via webhook, validates it against compliance/authority boundaries using an OpenAI-based agent, routes the request by authority level (auto vs human checkpoints), then generates an orchestration/execution plan with traceability using a second OpenAI agent, and finally returns a JSON response to the caller.

### Target use cases
- Compliance-gated execution requests where **budget**, **allowed actions**, and **authority levels** must be enforced.
- Multi-level approval flows (low/medium/high authority) with optional human sign-off.
- Audit/traceability generation for regulated environments (e.g., legal/compliance operations).

### Logical blocks
**1.1 Input Reception & Normalization**  
Webhook intake → static config → normalize request payload

**1.2 Policy/Boundary Validation (AI)**  
(Optionally fetch authority rules) → boundary enforcement agent → structured validation result

**1.3 Decisioning & Authority Routing**  
IF approved → route by authority level → auto-execute or wait for human approval

**1.4 Orchestration & Traceability (AI)**  
Merge authority paths → orchestration agent produces structured execution plan (with tool access available but currently not wired)

**1.5 Outcome Assembly & Webhook Response**  
Prepare final response (approved path) or rejection response → merge → respond to webhook

**Important implementation note (wiring gaps):**  
Three nodes exist but are **not connected** in the current workflow graph: **Fetch Authority Rules**, **HTTP Tool - Dynamic Actions**, and **Log to Audit Trail**. As a result, authority rules are never fetched, external HTTP “tool” actions are not actually executed, and audit logging is not performed—despite prompts/sticky notes implying these features.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Normalization

**Overview:**  
Accepts a POST request and creates a normalized internal request object (IDs, defaults, timestamp). Defines static governance limits (budget, duration, allowed actions).

**Nodes involved:**  
- Workflow Execution Request  
- Workflow Configuration  
- Prepare Request Data  

#### Node: **Workflow Execution Request**
- **Type / role:** `Webhook` trigger; entry point.
- **Configuration (interpreted):**
  - **Method:** POST
  - **Path:** `/workflow-execution`
  - **Response mode:** “responseNode” (workflow must end with Respond to Webhook).
- **Inputs/Outputs:**
  - **Output →** Workflow Configuration
- **Key data expected:**
  - `body.requestId` (optional)
  - `body.action` (required for meaningful processing)
  - `body.budget` (optional)
  - `body.requestor`, `body.priority`, `body.description` (optional)
- **Edge cases / failures:**
  - Missing or malformed JSON body → downstream expressions may evaluate to defaults but `action` could become `undefined`.
  - If workflow does not reach Respond to Webhook, request may time out.
- **Version notes:** typeVersion **2.1**.

#### Node: **Workflow Configuration**
- **Type / role:** `Set` node; provides governance constraints and defaults.
- **Configuration:**
  - `maxBudget = 10000`
  - `maxDuration = 3600` (seconds)
  - `allowedActions = ["read","write","execute"]`
  - `complianceLevel = "strict"`
  - `requireApproval = true`
- **Inputs/Outputs:**
  - Input from Webhook
  - **Output →** Prepare Request Data
- **Edge cases / failures:**
  - None typical; static constants.
- **Version notes:** typeVersion **3.4**.

#### Node: **Prepare Request Data**
- **Type / role:** `Set` node; maps/normalizes incoming request into consistent fields.
- **Configuration (expressions):**
  - `requestId = body.requestId || 'REQ-' + Date.now()`
  - `requestedAction = body.action`
  - `requestedBudget = body.budget || 0`
  - `requestor = body.requestor || 'unknown'`
  - `priority = body.priority || 'medium'`
  - `description = body.description || ''`
  - `timestamp = new Date().toISOString()`
- **Inputs/Outputs:**
  - Input from Workflow Configuration
  - **No outgoing connection in JSON** (but logically should feed validation/policy steps).  
    In practice, validation agent still references it indirectly via expressions like `$( "Workflow Execution Request").item.json...` and `$("Workflow Configuration").item.json...`, but without a direct main connection you may get item-context surprises depending on execution.
- **Edge cases / failures:**
  - If body is not present, most fields default; `requestedAction` may become `undefined`, weakening validation.
- **Version notes:** typeVersion **3.4**.

---

### 2.2 Policy/Boundary Validation (AI)

**Overview:**  
Uses an OpenAI chat model plus a structured output parser to decide whether the request is approved and what authority level is required, including compliance status and violations.

**Nodes involved:**  
- Fetch Authority Rules (unwired)  
- OpenAI Model - Boundary Enforcement  
- Validation Result Parser  
- Boundary Enforcement Agent  

#### Node: **Fetch Authority Rules** (UNWIRED)
- **Type / role:** `HTTP Request`; intended to fetch external authority/policy rules.
- **Configuration:** Not set (empty parameters).
- **Inputs/Outputs:** None (not connected).
- **Likely intended usage:** Populate `$json.authorityRules` consumed in Boundary Enforcement Agent’s system message.
- **Edge cases / failures (if wired later):**
  - Auth failures, timeouts, non-JSON responses.
  - Need to store fetched rules into a field like `authorityRules`.
- **Version notes:** typeVersion **4.4**.

#### Node: **OpenAI Model - Boundary Enforcement**
- **Type / role:** LangChain Chat Model node `lmChatOpenAi`; provides the LLM to the agent.
- **Configuration:**
  - **Model:** `gpt-5-mini`
  - **Temperature:** 0.1 (more deterministic)
  - Built-in tools: none
- **Credentials:** `openAiApi` (must be configured in n8n).
- **Inputs/Outputs:**
  - **AI languageModel output →** Boundary Enforcement Agent
- **Edge cases / failures:**
  - Invalid API key, model access issues, rate limits.
  - Model name availability depends on your OpenAI account/region.
- **Version notes:** typeVersion **1.3**.

#### Node: **Validation Result Parser**
- **Type / role:** LangChain structured output parser; forces JSON schema output.
- **Schema (key fields):**
  - `approved` (boolean, required)
  - `authorityLevel` (enum: low|medium|high)
  - `complianceStatus` (string, required)
  - `reasoning` (string, required)
  - `violations` (array of strings)
  - `recommendedActions` (array of strings)
- **Inputs/Outputs:**
  - **ai_outputParser →** Boundary Enforcement Agent
- **Edge cases / failures:**
  - If the model returns non-conforming JSON, parser throws an error and workflow fails unless error handling is configured.
- **Version notes:** typeVersion **1.3**.

#### Node: **Boundary Enforcement Agent**
- **Type / role:** LangChain Agent; performs validation reasoning and outputs structured decision.
- **Configuration:**
  - Prompt type: “define”
  - Uses:
    - Chat model: OpenAI Model - Boundary Enforcement
    - Output parser: Validation Result Parser
  - **System message** includes:
    - Request details from current `$json` (expects fields created by Prepare Request Data)
    - Authority rules from `$json.authorityRules || "Standard rules apply"`
    - Configuration limits from **Workflow Configuration** via expressions:
      - `$( "Workflow Configuration").item.json.maxBudget` etc.
  - `text` field: `Validate request: {requestedAction} from {requestor}`
- **Inputs/Outputs:**
  - **Main output →** Check Validation Result
- **Edge cases / failures:**
  - If Prepare Request Data is not upstream in the same item context, `$json.requestedAction` may be missing.
  - Parser errors if the agent doesn’t follow schema exactly.
- **Version notes:** typeVersion **3.1**.

---

### 2.3 Decisioning & Authority Routing

**Overview:**  
Approves or rejects the request. Approved requests are routed by required authority level: low is auto-prepared; medium/high pause at human checkpoints (webhook resume).

**Nodes involved:**  
- Check Validation Result  
- Route by Authority Level  
- Prepare Low Authority Execution  
- Human Checkpoint - Medium Authority  
- Human Checkpoint - High Authority  
- Merge Authority Paths  
- Prepare Rejection Response  

#### Node: **Check Validation Result**
- **Type / role:** `IF`; gate based on `$json.approved`.
- **Condition:** `$json.approved == true` (boolean equals).
- **Outputs:**
  - **True →** Route by Authority Level
  - **False →** Prepare Rejection Response
- **Edge cases / failures:**
  - If `approved` missing/not boolean, “loose” validation may behave unexpectedly.
- **Version notes:** typeVersion **2.3**.

#### Node: **Route by Authority Level**
- **Type / role:** `Switch`; routes approved requests by `$json.authorityLevel`.
- **Rules (strict, case-sensitive):**
  - “Low Authority” if equals `"low"`
  - “Medium Authority” if equals `"medium"`
  - “High Authority” if equals `"high"`
- **Outputs:**
  - Low → Prepare Low Authority Execution
  - Medium → Human Checkpoint - Medium Authority
  - High → Human Checkpoint - High Authority
- **Edge cases / failures:**
  - If authorityLevel is missing or not exactly one of these strings (case mismatch), nothing will match and downstream will not execute.
- **Version notes:** typeVersion **3.4**.

#### Node: **Prepare Low Authority Execution**
- **Type / role:** `Set`; creates a normalized “approved execution context” for low authority path.
- **Fields set:**
  - `executionPath = "low-authority"`
  - `requiresCheckpoint = false`
  - `requestId = $('Prepare Request Data').item.json.requestId`
  - `approvalStatus = "auto-approved"`
  - `authorityLevel = $json.authorityLevel`
  - `validationData = JSON.stringify($json)` (string, despite being typed as “object” in UI—watch for type mismatch)
- **Outputs:**
  - → Merge Authority Paths (input 0)
- **Edge cases / failures:**
  - If `Prepare Request Data` not executed or not in scope, requestId expression can fail.
- **Version notes:** typeVersion **3.4**.

#### Node: **Human Checkpoint - Medium Authority**
- **Type / role:** `Wait`; pauses until resume webhook or time limit reached.
- **Configuration:**
  - Resume: webhook
  - Max wait: 3600 seconds (1 hour), enforced
- **Outputs:**
  - → Merge Authority Paths (input 1)
- **Edge cases / failures:**
  - If no resume call arrives before timeout, execution stops/ends (n8n marks as waiting then timed out).
  - You must securely manage the resume webhook URL/token.
- **Version notes:** typeVersion **1.1**.

#### Node: **Human Checkpoint - High Authority**
- **Type / role:** `Wait`; similar to medium but longer window.
- **Configuration:**
  - Resume: webhook
  - Max wait: 7200 seconds (2 hours), enforced
- **Outputs:**
  - → Merge Authority Paths (input 2)
- **Edge cases / failures:** same as Medium Authority.
- **Version notes:** typeVersion **1.1**.

#### Node: **Merge Authority Paths**
- **Type / role:** `Merge`; consolidates the three authority paths into a single stream.
- **Configuration:**
  - `numberInputs = 3`
- **Outputs:**
  - → Orchestration Agent
- **Edge cases / failures:**
  - Depending on merge mode defaults, you may get waiting behavior if certain inputs never arrive. Ensure merge mode is appropriate (the JSON doesn’t specify mode; verify in UI).
- **Version notes:** typeVersion **3.2**.

#### Node: **Prepare Rejection Response**
- **Type / role:** `Set`; standardizes rejection payload returned to caller.
- **Fields:**
  - `status = "rejected"`
  - `requestId = $('Prepare Request Data').item.json.requestId`
  - `reason = $json.reasoning`
  - `violations = $json.violations`
  - `complianceStatus = $json.complianceStatus`
  - `timestamp = now ISO`
  - `message = "Request rejected due to authority or compliance violations"`
- **Outputs:**
  - → Merge All Outcomes (input 1)
- **Edge cases / failures:**
  - If violations undefined, output may contain null/empty.
  - If Prepare Request Data not available, requestId expression may fail.
- **Version notes:** typeVersion **3.4**.

---

### 2.4 Orchestration & Traceability (AI)

**Overview:**  
Generates a structured execution plan and audit-friendly traceability ID for approved requests. A dynamic HTTP tool is defined but not actually attached to the agent in this workflow wiring.

**Nodes involved:**  
- OpenAI Model - Orchestration  
- HTTP Tool - Dynamic Actions (unwired)  
- Execution Plan Parser  
- Orchestration Agent  
- Log to Audit Trail (unwired)

#### Node: **OpenAI Model - Orchestration**
- **Type / role:** LangChain Chat Model node `lmChatOpenAi`; model for orchestration.
- **Configuration:**
  - Model: `gpt-5-mini`
  - Temperature: 0.2
- **Credentials:** OpenAI API credential.
- **Outputs:**
  - ai_languageModel → Orchestration Agent
- **Edge cases / failures:** same OpenAI risks (auth, rate limit, model availability).
- **Version notes:** typeVersion **1.3**.

#### Node: **HTTP Tool - Dynamic Actions** (UNWIRED)
- **Type / role:** `HTTP Request Tool`; intended as a callable tool for the Orchestration Agent to execute external HTTP actions.
- **Configuration:** Empty (no defaults defined).
- **Inputs/Outputs:** None connected.
- **Impact:** Although the Orchestration Agent system message mentions “Dynamic Action Executor tool”, the agent currently has **no wired tool node** in this JSON.
- **Version notes:** typeVersion **4.4**.

#### Node: **Execution Plan Parser**
- **Type / role:** Structured output parser for orchestration results.
- **Schema highlights (required):**
  - `executionPlan` (array of step objects)
  - `status` (string)
  - `traceabilityId` (string)
  - Optional: `checkpoints` (array), `estimatedDuration` (number), `actions` (array of objects)
- **Outputs:**
  - ai_outputParser → Orchestration Agent
- **Edge cases / failures:**
  - Parser failures if model output doesn’t conform.
- **Version notes:** typeVersion **1.3**.

#### Node: **Orchestration Agent**
- **Type / role:** LangChain Agent; produces execution plan and traceability.
- **Configuration:**
  - Prompt type: define
  - Uses OpenAI Model - Orchestration + Execution Plan Parser
  - System message includes:
    - Approved request details from **Prepare Request Data**
    - Validation results from current `$json` (authorityLevel, approvalStatus, executionPath)
    - Instruction to use a “Dynamic Action Executor tool” (but tool not connected)
- **Inputs/Outputs:**
  - Main input from Merge Authority Paths
  - Output used by Prepare Final Response via `$('Orchestration Agent').item.json.output.*`
- **Edge cases / failures:**
  - If output parser fails, workflow halts.
  - If Prepare Request Data references fail (scope issues), prompt loses critical fields.
- **Version notes:** typeVersion **3.1**.

#### Node: **Log to Audit Trail** (UNWIRED)
- **Type / role:** `HTTP Request`; intended to log orchestration outputs to an audit system.
- **Configuration:** Empty.
- **Inputs/Outputs:** None connected.
- **Impact:** Final response sets `auditLogged = true` even though no audit call is executed.
- **Version notes:** typeVersion **4.4**.

---

### 2.5 Outcome Assembly & Webhook Response

**Overview:**  
Builds a unified JSON response for either approved+orchestrated completion or rejection, merges outcomes, and responds to the original webhook request.

**Nodes involved:**  
- Prepare Final Response  
- Merge All Outcomes  
- Return Response  

#### Node: **Prepare Final Response**
- **Type / role:** `Set`; constructs final success payload.
- **Fields (key expressions):**
  - `status = "completed"`
  - `requestId = $('Prepare Request Data').item.json.requestId`
  - `traceabilityId = $('Orchestration Agent').item.json.output.traceabilityId`
  - `executionPlan = $('Orchestration Agent').item.json.output.executionPlan`
  - `executionStatus = $('Orchestration Agent').item.json.output.status`
  - `checkpoints = JSON.stringify($('Orchestration Agent').item.json.output.checkpoints)` (stringified array)
  - `authorityLevel = $('Boundary Enforcement Agent').item.json.output.authorityLevel`
  - `complianceStatus = $('Boundary Enforcement Agent').item.json.output.complianceStatus`
  - `auditLogged = true` (note: audit node not wired)
  - timestamp + message
- **Outputs:**
  - → Merge All Outcomes (input 0)
- **Edge cases / failures:**
  - If Orchestration Agent did not run (e.g., merge behavior), these expressions fail.
  - `executionPlan` is set as type “string” but expression returns an array/object; verify n8n coercion.
- **Version notes:** typeVersion **3.4**.

#### Node: **Merge All Outcomes**
- **Type / role:** `Merge`; merges success path (input 0) and rejection path (input 1) into one stream for response.
- **Configuration:** Not specified (defaults apply).
- **Outputs:**
  - → Return Response
- **Edge cases / failures:**
  - Depending on merge mode, may require both inputs. Ensure it’s set to pass-through/append appropriately.
- **Version notes:** typeVersion **3.2**.

#### Node: **Return Response**
- **Type / role:** `Respond to Webhook`; returns JSON payload with HTTP 200.
- **Configuration:**
  - Response code: 200
  - Header: `Content-Type: application/json`
  - Body: `={{ $json }}`
- **Inputs/Outputs:** Terminal.
- **Edge cases / failures:**
  - If multiple items reach this node, response behavior depends on n8n settings (usually first item).
- **Version notes:** typeVersion **1.5**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Execution Request | Webhook | Receive execution request via HTTP POST | — | Workflow Configuration | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| Workflow Configuration | Set | Define governance limits (budget, duration, allowed actions) | Workflow Execution Request | Prepare Request Data | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| Prepare Request Data | Set | Normalize request payload, defaults, timestamp | Workflow Configuration | (none) | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| Fetch Authority Rules | HTTP Request | Retrieve external authority/compliance rules | (none) | (none) | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| OpenAI Model - Boundary Enforcement | OpenAI Chat Model (LangChain) | LLM for validation | (AI wired) | Boundary Enforcement Agent | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| Validation Result Parser | Structured Output Parser (LangChain) | Enforce schema for validation output | (AI wired) | Boundary Enforcement Agent | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| Boundary Enforcement Agent | LangChain Agent | Validate request vs rules; output approved/authority/violations | (implicit context) | Check Validation Result | Validates incoming lawsuit data against compliance requirements and boundary rules; Why: Ensures data integrity before initiating costly legal response processes |
| Check Validation Result | IF | Gate approved vs rejected | Boundary Enforcement Agent | Route by Authority Level; Prepare Rejection Response | Routes cases to appropriate human authority based on validation results and organizational hierarchy; Why: Ensures proper oversight by qualified personnel matching case complexity |
| Route by Authority Level | Switch | Route by authority level low/medium/high | Check Validation Result (true) | Prepare Low Authority Execution; Human Checkpoint - Medium Authority; Human Checkpoint - High Authority | Routes cases to appropriate human authority based on validation results and organizational hierarchy; Why: Ensures proper oversight by qualified personnel matching case complexity |
| Prepare Low Authority Execution | Set | Create auto-approved execution context | Route by Authority Level (low) | Merge Authority Paths | Routes cases to appropriate human authority based on validation results and organizational hierarchy; Why: Ensures proper oversight by qualified personnel matching case complexity |
| Human Checkpoint - Medium Authority | Wait | Pause for human approval (webhook resume) | Route by Authority Level (medium) | Merge Authority Paths | Routes cases to appropriate human authority based on validation results and organizational hierarchy; Why: Ensures proper oversight by qualified personnel matching case complexity |
| Human Checkpoint - High Authority | Wait | Pause for human approval (webhook resume) | Route by Authority Level (high) | Merge Authority Paths | Routes cases to appropriate human authority based on validation results and organizational hierarchy; Why: Ensures proper oversight by qualified personnel matching case complexity |
| Merge Authority Paths | Merge | Consolidate low/medium/high paths | Prepare Low Authority Execution; Human Checkpoint - Medium Authority; Human Checkpoint - High Authority | Orchestration Agent | Routes cases to appropriate human authority based on validation results and organizational hierarchy; Why: Ensures proper oversight by qualified personnel matching case complexity |
| OpenAI Model - Orchestration | OpenAI Chat Model (LangChain) | LLM for orchestration planning | (AI wired) | Orchestration Agent | Generates orchestration reports and prepares final lawsuit responses with execution plans; Why: Creates audit-ready documentation while automating response assembly for timely submission |
| HTTP Tool - Dynamic Actions | HTTP Request Tool | Intended agent tool for external actions | (none) | (none) | Generates orchestration reports and prepares final lawsuit responses with execution plans; Why: Creates audit-ready documentation while automating response assembly for timely submission |
| Execution Plan Parser | Structured Output Parser (LangChain) | Enforce schema for orchestration output | (AI wired) | Orchestration Agent | Generates orchestration reports and prepares final lawsuit responses with execution plans; Why: Creates audit-ready documentation while automating response assembly for timely submission |
| Orchestration Agent | LangChain Agent | Build execution plan + traceability | Merge Authority Paths | (used by expressions); (no direct main output node) | Generates orchestration reports and prepares final lawsuit responses with execution plans; Why: Creates audit-ready documentation while automating response assembly for timely submission |
| Log to Audit Trail | HTTP Request | Intended external audit logging | (none) | (none) | Generates orchestration reports and prepares final lawsuit responses with execution plans; Why: Creates audit-ready documentation while automating response assembly for timely submission |
| Prepare Final Response | Set | Build “completed” response object | (not explicitly connected; implied after orchestration) | Merge All Outcomes | Generates orchestration reports and prepares final lawsuit responses with execution plans; Why: Creates audit-ready documentation while automating response assembly for timely submission |
| Merge All Outcomes | Merge | Merge success + rejection outputs | Prepare Final Response; Prepare Rejection Response | Return Response |  |
| Return Response | Respond to Webhook | Return JSON to caller | Merge All Outcomes | — |  |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note5 | Sticky Note | Comment block | — | — |  |
| Sticky Note6 | Sticky Note | Comment block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: `workflow-execution`
   - Response mode: **Using “Respond to Webhook” node**

2) **Add governance configuration**
   - Add node: **Set** named “Workflow Configuration”
   - Add fields:
     - `maxBudget` (number) = `10000`
     - `maxDuration` (number) = `3600`
     - `allowedActions` (array) = `["read","write","execute"]`
     - `complianceLevel` (string) = `strict`
     - `requireApproval` (boolean) = `true`
   - Connect: **Webhook → Workflow Configuration**

3) **Normalize request payload**
   - Add node: **Set** named “Prepare Request Data”
   - Map fields from webhook body (Expressions):
     - `requestId` = `{{ $('Workflow Execution Request').item.json.body.requestId || 'REQ-' + Date.now() }}`
     - `requestedAction` = `{{ $('Workflow Execution Request').item.json.body.action }}`
     - `requestedBudget` = `{{ $('Workflow Execution Request').item.json.body.budget || 0 }}`
     - `requestor` = `{{ $('Workflow Execution Request').item.json.body.requestor || 'unknown' }}`
     - `priority` = `{{ $('Workflow Execution Request').item.json.body.priority || 'medium' }}`
     - `description` = `{{ $('Workflow Execution Request').item.json.body.description || '' }}`
     - `timestamp` = `{{ new Date().toISOString() }}`
   - Connect: **Workflow Configuration → Prepare Request Data**

4) **(Optional but recommended) Fetch authority rules**
   - Add node: **HTTP Request** named “Fetch Authority Rules”
   - Configure URL/auth to your policy source (e.g., internal endpoint)
   - Ensure it outputs a field like `authorityRules`
   - Connect: **Prepare Request Data → Fetch Authority Rules**
   - If you don’t have a rules endpoint, skip this node and keep the agent fallback (“Standard rules apply”).

5) **Add OpenAI model for validation**
   - Add node: **OpenAI Chat Model (LangChain)** named “OpenAI Model - Boundary Enforcement”
   - Model: `gpt-5-mini`
   - Temperature: `0.1`
   - Credentials: configure **OpenAI API** in n8n Credentials

6) **Add structured output parser for validation**
   - Add node: **Structured Output Parser (LangChain)** named “Validation Result Parser”
   - Schema: include `approved`, `authorityLevel (low|medium|high)`, `complianceStatus`, `reasoning`, optional `violations`, `recommendedActions`.

7) **Add validation agent**
   - Add node: **AI Agent (LangChain)** named “Boundary Enforcement Agent”
   - Set **Chat Model** to “OpenAI Model - Boundary Enforcement”
   - Set **Output Parser** to “Validation Result Parser”
   - System message: include request details + configuration limits + authority rules field.
   - Connect main path into this agent:
     - If using rules: **Fetch Authority Rules → Boundary Enforcement Agent**
     - If not: **Prepare Request Data → Boundary Enforcement Agent**

8) **Add approval gate**
   - Add node: **IF** named “Check Validation Result”
   - Condition: boolean `{{ $json.approved }}` equals `true`
   - Connect: **Boundary Enforcement Agent → Check Validation Result**

9) **Add authority routing**
   - Add node: **Switch** named “Route by Authority Level”
   - Value to evaluate: `{{ $json.authorityLevel }}`
   - Add rules for `low`, `medium`, `high` (case-sensitive)
   - Connect: **Check Validation Result (true) → Route by Authority Level**

10) **Low authority: auto-prepare**
   - Add node: **Set** named “Prepare Low Authority Execution”
   - Add fields as in section 2.3
   - Connect: **Route (low) → Prepare Low Authority Execution**

11) **Medium/high authority: human checkpoints**
   - Add node: **Wait** named “Human Checkpoint - Medium Authority”
     - Resume: **Webhook**
     - Limit wait time: enabled
     - 3600 seconds
   - Add node: **Wait** named “Human Checkpoint - High Authority”
     - Resume: **Webhook**
     - Limit wait time: enabled
     - 7200 seconds
   - Connect: **Route (medium) → Wait (medium)** and **Route (high) → Wait (high)**

12) **Merge authority paths**
   - Add node: **Merge** named “Merge Authority Paths”
   - Number of inputs: **3**
   - Connect:
     - Low set → Merge input 0
     - Medium wait → Merge input 1
     - High wait → Merge input 2

13) **Add OpenAI model for orchestration**
   - Add node: **OpenAI Chat Model (LangChain)** named “OpenAI Model - Orchestration”
   - Model: `gpt-5-mini`
   - Temperature: `0.2`
   - Credentials: OpenAI API

14) **Add structured output parser for orchestration**
   - Add node: **Structured Output Parser (LangChain)** named “Execution Plan Parser”
   - Schema: require `executionPlan[]`, `status`, `traceabilityId`; optionally `checkpoints`, etc.

15) **(Optional) Add HTTP Tool for agent actions**
   - Add node: **HTTP Request Tool** named “HTTP Tool - Dynamic Actions”
   - Configure base URL/auth if needed
   - In the Orchestration Agent, ensure the tool is actually attached/available (in n8n UI: add as a tool node for the agent).  
   - Without this, the agent can only plan, not execute external HTTP actions.

16) **Add orchestration agent**
   - Add node: **AI Agent (LangChain)** named “Orchestration Agent”
   - Attach:
     - Chat model: “OpenAI Model - Orchestration”
     - Output parser: “Execution Plan Parser”
     - Tools: “HTTP Tool - Dynamic Actions” (if you want real actions)
   - Connect: **Merge Authority Paths → Orchestration Agent**

17) **(Optional) Audit logging**
   - Add node: **HTTP Request** named “Log to Audit Trail”
   - Configure endpoint to receive audit events (requestId, traceabilityId, executionPlan, timestamps)
   - Connect: **Orchestration Agent → Log to Audit Trail**

18) **Prepare success response**
   - Add node: **Set** named “Prepare Final Response”
   - Populate fields from Prepare Request Data + Orchestration Agent output + Boundary Enforcement Agent output.
   - Connect: from **Orchestration Agent** (or **Log to Audit Trail** if used) → **Prepare Final Response**

19) **Prepare rejection response**
   - Add node: **Set** named “Prepare Rejection Response”
   - Connect: **Check Validation Result (false) → Prepare Rejection Response**

20) **Merge outcomes**
   - Add node: **Merge** named “Merge All Outcomes”
   - Connect:
     - Prepare Final Response → Merge input 0
     - Prepare Rejection Response → Merge input 1

21) **Respond to webhook**
   - Add node: **Respond to Webhook** named “Return Response”
   - Respond with JSON: `{{ $json }}`
   - HTTP code: 200
   - Header: `Content-Type: application/json`
   - Connect: **Merge All Outcomes → Return Response**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Prerequisites: OpenAI or Nvidia API credentials for validation processing, Google Sheets access for orchestration logging” | Sticky Note1 (note: Google Sheets is mentioned but no Google Sheets node exists in this workflow JSON) |
| “Use Cases: Government litigation departments managing multi-level approval workflows” | Sticky Note1 |
| “Customization: Modify authority routing logic for organizational hierarchies” | Sticky Note1 |
| “Benefits: Reduces response coordination time by 70%, eliminates manual routing errors” | Sticky Note1 |
| Setup steps list includes “Orchestration Export node with Google Sheets credentials” | Sticky Note2 (not present; likely planned extension) |
| Workflow narrative describing lawsuit notifications, routing, and audit logging | Sticky Note3 |
| Validation block intent: “Validates incoming lawsuit data…” | Sticky Note |
| Routing block intent: “Routes cases to appropriate human authority…” | Sticky Note5 |
| Orchestration block intent: “Generates orchestration reports and prepares final lawsuit responses…” | Sticky Note6 |

