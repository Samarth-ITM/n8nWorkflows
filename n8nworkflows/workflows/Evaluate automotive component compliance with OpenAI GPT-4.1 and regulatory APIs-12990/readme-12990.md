Evaluate automotive component compliance with OpenAI GPT-4.1 and regulatory APIs

https://n8nworkflows.xyz/workflows/evaluate-automotive-component-compliance-with-openai-gpt-4-1-and-regulatory-apis-12990


# Evaluate automotive component compliance with OpenAI GPT-4.1 and regulatory APIs

## 1. Workflow Overview

**Purpose:**  
This workflow receives automotive component/design data via an HTTP webhook and evaluates regulatory compliance using an OpenAI (GPT-4.1-mini) agent augmented with two tools (a performance simulation tool and a calculator) plus a **structured JSON output parser**. It supports **two routing paths**: (1) split-and-evaluate multiple components individually, or (2) fetch regulatory context from an external regulatory database API and then evaluate.

**Target use cases:** compliance and safety assessment for pre-production validation, supplier certification, QA/regulatory reviews.

### 1.1 Webhook Reception & Configuration
Receives a POST request and sets global “standards”/URLs used later (safety, emissions, thresholds, API endpoints).

### 1.2 Evaluation Type Routing (Dual Path)
Determines whether the incoming payload contains `body.components` (multi-component) and routes accordingly:
- **Path A:** split each component and evaluate separately
- **Path B:** fetch regulatory database context (by component type) then evaluate

### 1.3 AI Compliance Assessment (Agent + Tools + Structured Output)
Runs a LangChain Agent using GPT-4.1-mini with:
- **Performance Simulation Tool** (code tool)
- **Calculator tool**
- **Structured Output Parser** enforcing a compliance report schema

### 1.4 Aggregation, Risk Scoring, Metadata, Logging, Response
Aggregates results, computes an overall risk score, enriches with metadata, checks if escalation/logging is needed, logs non-compliance to an external API, and responds to the webhook caller.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Reception & Workflow Configuration

**Overview:**  
Accepts incoming compliance evaluation requests and sets baseline standards plus external endpoint placeholders used throughout the workflow.

**Nodes involved:**
- Webhook Trigger
- Workflow Configuration

#### Node: **Webhook Trigger**
- **Type / role:** `n8n-nodes-base.webhook` — Entry point (HTTP POST).
- **Key configuration:**
  - **Method:** POST
  - **Path:** `automotive-compliance-evaluation`
  - **Response mode:** `lastNode` (response comes from the final node reached in the executed branch)
- **Input/Output:**
  - No inputs; outputs to **Workflow Configuration**.
- **Edge cases / failures:**
  - If the caller sends non-JSON or unexpected shape, downstream expressions that reference `$json.body...` may become `undefined`.
  - Large payloads may hit n8n webhook/body size limits depending on instance config.

#### Node: **Workflow Configuration**
- **Type / role:** `n8n-nodes-base.set` — Stores constants/config and passes through incoming payload.
- **Key configuration choices:**
  - `includeOtherFields: true` (keeps the inbound webhook payload while adding fields)
  - Adds:
    - `safetyStandards`: `"FMVSS, ECE R94, ISO 26262, NCAP"`
    - `emissionsStandards`: `"EPA Tier 3, Euro 6, CARB LEV III"`
    - `performanceThresholds`: `"Crashworthiness: 5-star NCAP, Emissions: <50mg/km NOx, Energy Efficiency: >100 MPGe"`
    - `regulatoryDatabaseUrl`: placeholder string
    - `complianceLogUrl`: placeholder string
- **Input/Output:**
  - Input: Webhook payload
  - Output: **Check Evaluation Type**
- **Edge cases / failures:**
  - The URL fields are placeholders; if not replaced with valid URLs, HTTP nodes will fail (invalid URL / DNS / connection).

---

### Block 2 — Evaluation Type Routing (Dual Path)

**Overview:**  
Checks whether the request includes a `body.components` array. If yes, it splits and evaluates per component; otherwise, it fetches regulatory context first and then evaluates.

**Nodes involved:**
- Check Evaluation Type
- Split Components
- Fetch Regulatory Database

#### Node: **Check Evaluation Type**
- **Type / role:** `n8n-nodes-base.if` — Branching decision.
- **Key configuration choices:**
  - Condition: checks if `{{ $('Workflow Configuration').item.json.body.components }}` **exists** (array exists operator).
  - Output 1 (true): **Split Components**
  - Output 2 (false): **Fetch Regulatory Database**
- **Input/Output connections:**
  - Input from **Workflow Configuration**
  - True → **Split Components**
  - False → **Fetch Regulatory Database**
- **Edge cases / failures:**
  - If the webhook payload doesn’t contain `body` (e.g., caller sends data at top-level), the condition may evaluate as “does not exist” and route to the regulatory fetch path unintentionally.
  - “Loose” type validation helps, but shape mismatches still lead to wrong branch selection.

#### Node: **Split Components**
- **Type / role:** `n8n-nodes-base.splitOut` — Creates one item per component.
- **Key configuration:**
  - `fieldToSplitOut: "body.components"`
- **Input/Output:**
  - Input: item containing `body.components[]`
  - Output: each item contains one element from that array (plus relevant context depending on SplitOut behavior)
  - Next node: **Automotive Compliance Evaluator**
- **Edge cases / failures:**
  - If `body.components` is not an array, SplitOut fails.
  - Component items may lack fields expected by the agent schema (e.g., `component_id`), increasing “Needs Review” outcomes or parser failures.

#### Node: **Fetch Regulatory Database**
- **Type / role:** `n8n-nodes-base.httpRequest` — Retrieves regulatory reference/context.
- **Key configuration choices:**
  - URL: `{{ $('Workflow Configuration').first().json.regulatoryDatabaseUrl }}`
  - Query param enabled:
    - `component_type = {{ $json.body.component_type || "general" }}`
- **Input/Output:**
  - Input: from **Check Evaluation Type** (false path)
  - Output: HTTP response JSON → **Automotive Compliance Evaluator**
- **Edge cases / failures:**
  - If `regulatoryDatabaseUrl` remains a placeholder/empty → request fails.
  - Auth not configured (none shown in node) → 401/403 depending on API.
  - If webhook payload is top-level and not under `body`, `body.component_type` will be undefined and defaults to `"general"`.

---

### Block 3 — AI Compliance Assessment (Agent + Tools + Structured Output)

**Overview:**  
A LangChain agent evaluates a component/design against standards, can call tools for simulation/math, and must output a strict JSON schema.

**Nodes involved:**
- Automotive Compliance Evaluator (Agent)
- OpenAI Chat Model
- Performance Simulation Tool
- Calculator
- Structured Output Parser

#### Node: **OpenAI Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM backend for the agent.
- **Key configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: “OpenAi account” (OpenAI API key)
- **Connections:**
  - Connected to **Automotive Compliance Evaluator** via `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid/expired OpenAI credentials → auth error.
  - Rate limits / quota exhaustion → 429 errors.
  - Model availability changes can break executions; keep an alternative model handy.

#### Node: **Performance Simulation Tool**
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCode` — Custom tool callable by the agent.
- **Key behavior:**
  - Parses `query` as JSON if string; otherwise uses it directly.
  - Produces simulated metrics (stress tests, temperature performance, load scenarios), identifies failure points, computes an overall performance score.
  - Returns a **JSON string** (stringified object).
- **Connections:**
  - Connected to **Automotive Compliance Evaluator** via `ai_tool`.
- **Edge cases / failures:**
  - Non-JSON `query` string → `JSON.parse` throws.
  - Uses `Math.random()` → non-deterministic results (may be undesirable for audits).
  - Tool returns a string; the agent must interpret it correctly.

#### Node: **Calculator**
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator` — General math tool for the agent.
- **Connections:**
  - Connected to **Automotive Compliance Evaluator** via `ai_tool`.
- **Edge cases:**
  - The agent may not use it unless prompted by need; results depend on prompt/tool-calling behavior.

#### Node: **Structured Output Parser**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces strict JSON schema output.
- **Key configuration:**
  - Manual JSON schema requiring fields:
    - `vehicle_model` (string)
    - `component_id` (string)
    - `observed_issues` (array of strings)
    - `recommended_modifications` (array of strings)
    - `compliance_status` enum: `Compliant | Non-Compliant | Needs Review`
    - `regulatory_reference` (array of strings)
    - `confidence_score` (number 0–100)
- **Connections:**
  - Connected to **Automotive Compliance Evaluator** via `ai_outputParser`.
- **Edge cases / failures:**
  - If the model outputs non-JSON or missing required fields → parsing failure, node errors.
  - If confidence score is out of range or wrong type → schema validation failure.

#### Node: **Automotive Compliance Evaluator**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates LLM + tools and outputs structured compliance report.
- **Key configuration choices:**
  - **Input text:** `{{ $json.component || $json.body.design_data || $json.body }}`
    - Supports multiple payload shapes; prefers `component`, then `body.design_data`, else `body`.
  - **System message:** embeds the standards from **Workflow Configuration** via expressions:
    - `{{ $('Workflow Configuration').first().json.safetyStandards }}`
    - `{{ $('Workflow Configuration').first().json.emissionsStandards }}`
    - `{{ $('Workflow Configuration').first().json.performanceThresholds }}`
  - Tools available: Performance Simulation Tool + Calculator
  - Output parser enabled: must match Structured Output Parser schema
- **Input/Output:**
  - Input from:
    - **Split Components** (Path A) OR
    - **Fetch Regulatory Database** (Path B)
  - Output to **Aggregate Results**
- **Edge cases / failures:**
  - If `$json.body` is an object but not descriptive enough, agent may output “Needs Review”.
  - If upstream Path B returns regulatory API response that doesn’t include the original component details, `text` may become the regulatory response rather than the component spec (depending on API response shape), reducing evaluation quality. Consider merging original payload + regulatory results explicitly.

---

### Block 4 — Aggregation, Risk Scoring, Metadata, Logging, Response

**Overview:**  
Collects all evaluation outputs, computes a portfolio-level risk score, enriches result metadata, optionally logs non-compliance to an external API, and returns JSON to the webhook caller.

**Nodes involved:**
- Aggregate Results
- Calculate Risk Score
- Enrich with Metadata
- Check Compliance Status
- Log Non-Compliant to Database
- Respond to Webhook

#### Node: **Aggregate Results**
- **Type / role:** `n8n-nodes-base.aggregate` — Aggregates per-item agent outputs.
- **Key configuration:**
  - `aggregate: aggregateAllItemData`
  - `destinationFieldName: "output"`
- **Input/Output:**
  - Input: multiple items from **Automotive Compliance Evaluator**
  - Output: aggregated data → **Calculate Risk Score**
- **Edge cases / failures:**
  - Downstream code expects individual items with `compliance_status` etc.; but aggregation packs data under `output`. This creates a potential mismatch with **Calculate Risk Score** (see next node).

#### Node: **Calculate Risk Score**
- **Type / role:** `n8n-nodes-base.code` — Business-rule risk scoring.
- **Key behavior:**
  - Reads `const items = $input.all();`
  - For each `item.json`, expects:
    - `compliance_status`
    - `observed_issues[]`
    - `confidence_score`
  - Computes:
    - `overall_risk_score` (0–100)
    - `risk_level` (Low/Medium/High/Critical)
    - totals and rates
    - `evaluation_items: items.map(item => item.json)`
- **Input/Output:**
  - Input: output of **Aggregate Results**
  - Output: single item JSON → **Enrich with Metadata**
- **Important integration issue:**
  - **Aggregate Results** stores aggregated content under `output`, but this code iterates items as if each item is a single evaluation result.  
  - Likely outcomes:
    - If Aggregate outputs **one item** like `{ output: [...] }`, then `data.compliance_status` is undefined and risk score becomes distorted.
  - Fix options:
    1. Remove aggregation before scoring; score directly on the per-component items.
    2. Or change the code to iterate `item.json.output` array.
- **Edge cases / failures:**
  - If `confidence_score` is missing, averageConfidence may undercount.
  - If `observed_issues` is not an array, issue count may be wrong.

#### Node: **Enrich with Metadata**
- **Type / role:** `n8n-nodes-base.set` — Adds timestamps and summary counts.
- **Key configuration:**
  - `evaluation_timestamp = {{ $now.toISO() }}`
  - `total_components_evaluated = {{ $json.aggregated_results.length }}`
  - `overall_risk_score = {{ $json.risk_score }}`
- **Critical mapping issues:**
  - The incoming JSON from **Calculate Risk Score** is:
    - `overall_risk_score` (not `risk_score`)
    - `total_items_evaluated` (not `aggregated_results.length`)
    - `evaluation_items` (not `aggregated_results`)
  - As written, `total_components_evaluated` and `overall_risk_score` will likely become `undefined` or error depending on n8n expression evaluation.
- **Input/Output:**
  - Input: **Calculate Risk Score**
  - Output: **Check Compliance Status**
- **Edge cases / failures:**
  - Expression failures due to missing properties.

#### Node: **Check Compliance Status**
- **Type / role:** `n8n-nodes-base.if` — Determines whether to log non-compliance / critical risk.
- **Key configuration:**
  - OR conditions:
    1. `{{ $('Calculate Risk Score').item.json.overall_risk_score }}` > 70
    2. `{{ $('Enrich with Metadata').item.json.compliance_status }}` equals `Non-Compliant`
- **Logic concern:**
  - `Enrich with Metadata` does not set `compliance_status`. Unless it is preserved from upstream (it won’t be if upstream is a portfolio object), this condition may never match.
  - This node appears to mix **portfolio-level** risk score with **item-level** compliance status.
- **Outputs:**
  - True → **Log Non-Compliant to Database**
  - False → **Respond to Webhook**
- **Edge cases / failures:**
  - If the referenced nodes don’t have `.item` in the current context (multi-item), expression may not resolve as expected.

#### Node: **Log Non-Compliant to Database**
- **Type / role:** `n8n-nodes-base.httpRequest` — Logs details to an external compliance logging API.
- **Key configuration:**
  - URL: `{{ $('Workflow Configuration').first().json.complianceLogUrl }}`
  - Method: POST
  - Body fields map from `$json`:
    - `vehicle_model`, `component_id`, `compliance_status`, `observed_issues`, `recommended_modifications`, `regulatory_reference`, `confidence_score`
    - `timestamp = {{ $now.toISO() }}`
- **Input/Output:**
  - Input: True branch from **Check Compliance Status**
  - Output: **Respond to Webhook**
- **Edge cases / failures:**
  - Placeholder URL or missing auth → request fails.
  - Body fields may not exist if upstream is aggregated/portfolio-level → logs incomplete or wrong data.
  - Sending arrays (`observed_issues`, etc.) depends on API expectations (string vs JSON array).

#### Node: **Respond to Webhook**
- **Type / role:** `n8n-nodes-base.respondToWebhook` — Final response to caller.
- **Key configuration:**
  - Respond with: JSON
  - Body: `{{ $json }}` (passes through final node’s output)
- **Input/Output:**
  - Receives either:
    - result from **Check Compliance Status** (false path), or
    - result from **Log Non-Compliant to Database** (true path)
- **Edge cases:**
  - If the last executed node returns an HTTP response payload rather than the evaluation summary, the caller will receive that instead. Consider explicitly shaping the response before this node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Entry point (HTTP POST) | — | Workflow Configuration | ## Webhook Reception and Evaluation Type Routing\n**Why:** Captures compliance evaluation requests containing component specifications and regulatory requirements, intelligently determines whether assessment requires component-level split analysis or consolidated regulatory database lookup |
| Workflow Configuration | n8n-nodes-base.set | Set standards + endpoint URLs; pass through payload | Webhook Trigger | Check Evaluation Type | ## Webhook Reception and Evaluation Type Routing\n**Why:** Captures compliance evaluation requests containing component specifications and regulatory requirements, intelligently determines whether assessment requires component-level split analysis or consolidated regulatory database lookup |
| Check Evaluation Type | n8n-nodes-base.if | Route based on presence of components array | Workflow Configuration | Split Components; Fetch Regulatory Database | ## Webhook Reception and Evaluation Type Routing\n**Why:** Captures compliance evaluation requests containing component specifications and regulatory requirements, intelligently determines whether assessment requires component-level split analysis or consolidated regulatory database lookup |
| Split Components | n8n-nodes-base.splitOut | Split `body.components[]` into items | Check Evaluation Type (true) | Automotive Compliance Evaluator | ## Webhook Reception and Evaluation Type Routing\n**Why:** Captures compliance evaluation requests containing component specifications and regulatory requirements, intelligently determines whether assessment requires component-level split analysis or consolidated regulatory database lookup |
| Fetch Regulatory Database | n8n-nodes-base.httpRequest | Pull regulatory context by component type | Check Evaluation Type (false) | Automotive Compliance Evaluator | ## Webhook Reception and Evaluation Type Routing\n**Why:** Captures compliance evaluation requests containing component specifications and regulatory requirements, intelligently determines whether assessment requires component-level split analysis or consolidated regulatory database lookup |
| Automotive Compliance Evaluator | @n8n/n8n-nodes-langchain.agent | AI agent performing compliance evaluation | Split Components; Fetch Regulatory Database | Aggregate Results | ## Parallel AI Compliance Assessment with Specialized Tools\n**Why:** Processes components through OpenAI Chat Model equipped with Performance Simulation Tool for technical validation, Calculator for metrics computation, and Structured Output Parser for standardized compliance reporting |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for agent | — (wired as model) | Automotive Compliance Evaluator | ## Parallel AI Compliance Assessment with Specialized Tools\n**Why:** Processes components through OpenAI Chat Model equipped with Performance Simulation Tool for technical validation, Calculator for metrics computation, and Structured Output Parser for standardized compliance reporting |
| Performance Simulation Tool | @n8n/n8n-nodes-langchain.toolCode | Tool: simulate component performance | — (tool callable) | Automotive Compliance Evaluator | ## Parallel AI Compliance Assessment with Specialized Tools\n**Why:** Processes components through OpenAI Chat Model equipped with Performance Simulation Tool for technical validation, Calculator for metrics computation, and Structured Output Parser for standardized compliance reporting |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Tool: calculations for thresholds/margins | — (tool callable) | Automotive Compliance Evaluator | ## Parallel AI Compliance Assessment with Specialized Tools\n**Why:** Processes components through OpenAI Chat Model equipped with Performance Simulation Tool for technical validation, Calculator for metrics computation, and Structured Output Parser for standardized compliance reporting |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema output | — (wired as parser) | Automotive Compliance Evaluator | ## Parallel AI Compliance Assessment with Specialized Tools\n**Why:** Processes components through OpenAI Chat Model equipped with Performance Simulation Tool for technical validation, Calculator for metrics computation, and Structured Output Parser for standardized compliance reporting |
| Aggregate Results | n8n-nodes-base.aggregate | Aggregate evaluation results | Automotive Compliance Evaluator | Calculate Risk Score | ## Risk Scoring and Compliance Status Reporting\n**Why:** Aggregates multi-dimensional assessment results, calculates overall risk scores using defined business rules, enriches findings with compliance metadata including regulation references and remediation guidance, logs non-compliant items to regulatory database |
| Calculate Risk Score | n8n-nodes-base.code | Compute overall risk score & risk level | Aggregate Results | Enrich with Metadata | ## Risk Scoring and Compliance Status Reporting\n**Why:** Aggregates multi-dimensional assessment results, calculates overall risk scores using defined business rules, enriches findings with compliance metadata including regulation references and remediation guidance, logs non-compliant items to regulatory database |
| Enrich with Metadata | n8n-nodes-base.set | Add timestamp and summary stats | Calculate Risk Score | Check Compliance Status | ## Risk Scoring and Compliance Status Reporting\n**Why:** Aggregates multi-dimensional assessment results, calculates overall risk scores using defined business rules, enriches findings with compliance metadata including regulation references and remediation guidance, logs non-compliant items to regulatory database |
| Check Compliance Status | n8n-nodes-base.if | Decide whether to log non-compliance | Enrich with Metadata | Log Non-Compliant to Database; Respond to Webhook | ## Risk Scoring and Compliance Status Reporting\n**Why:** Aggregates multi-dimensional assessment results, calculates overall risk scores using defined business rules, enriches findings with compliance metadata including regulation references and remediation guidance, logs non-compliant items to regulatory database |
| Log Non-Compliant to Database | n8n-nodes-base.httpRequest | POST non-compliance record to logging API | Check Compliance Status (true) | Respond to Webhook | ## Risk Scoring and Compliance Status Reporting\n**Why:** Aggregates multi-dimensional assessment results, calculates overall risk scores using defined business rules, enriches findings with compliance metadata including regulation references and remediation guidance, logs non-compliant items to regulatory database |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return JSON response to caller | Check Compliance Status (false); Log Non-Compliant to Database | — | ## Risk Scoring and Compliance Status Reporting\n**Why:** Aggregates multi-dimensional assessment results, calculates overall risk scores using defined business rules, enriches findings with compliance metadata including regulation references and remediation guidance, logs non-compliant items to regulatory database |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named:  
   “AI Automotive Compliance Evaluator with Dual-Path Regulatory Assessment”.

2. **Add “Webhook Trigger”** (`Webhook` node):
   - HTTP Method: **POST**
   - Path: **automotive-compliance-evaluation**
   - Response Mode: **Last Node**
   - Save to generate the production/test URL.

3. **Add “Workflow Configuration”** (`Set` node) and connect:
   - Webhook Trigger → Workflow Configuration
   - Turn on **Include Other Fields**
   - Add string fields:
     - `safetyStandards` = `FMVSS, ECE R94, ISO 26262, NCAP`
     - `emissionsStandards` = `EPA Tier 3, Euro 6, CARB LEV III`
     - `performanceThresholds` = `Crashworthiness: 5-star NCAP, Emissions: <50mg/km NOx, Energy Efficiency: >100 MPGe`
     - `regulatoryDatabaseUrl` = *(your regulatory API endpoint URL)*
     - `complianceLogUrl` = *(your logging API endpoint URL)*

4. **Add “Check Evaluation Type”** (`IF` node) and connect:
   - Workflow Configuration → Check Evaluation Type
   - Condition: **Array exists**
     - Value: expression referencing the incoming payload components array, e.g. `{{$json.body.components}}`
     - (In the provided workflow it references `$('Workflow Configuration').item.json.body.components`.)

5. **Create Path A (multi-component):**
   1. Add **“Split Components”** (`Split Out` node):
      - Field to split out: `body.components`
      - Connect: Check Evaluation Type (true) → Split Components

6. **Create Path B (single / no components array):**
   1. Add **“Fetch Regulatory Database”** (`HTTP Request` node):
      - URL: `{{$('Workflow Configuration').first().json.regulatoryDatabaseUrl}}`
      - Enable query parameters:
        - `component_type` = `{{$json.body.component_type || "general"}}`
      - Configure authentication as required by your API (Header token / OAuth2 / etc.).
      - Connect: Check Evaluation Type (false) → Fetch Regulatory Database

7. **Add the AI layer:**
   1. Add **“OpenAI Chat Model”** (`OpenAI Chat Model` / LangChain node):
      - Model: **gpt-4.1-mini**
      - Credentials: create/select **OpenAI API** credentials.
   2. Add **“Performance Simulation Tool”** (`Tool Code` node):
      - Paste the simulation JavaScript code (as provided).
      - Keep description explaining tool purpose.
   3. Add **“Calculator”** (LangChain Calculator tool node).
   4. Add **“Structured Output Parser”** (Structured Output Parser node):
      - Schema type: manual
      - Paste the JSON schema requiring: vehicle_model, component_id, observed_issues, recommended_modifications, compliance_status, regulatory_reference, confidence_score.
   5. Add **“Automotive Compliance Evaluator”** (LangChain Agent node):
      - Prompt type: “Define”
      - Text input: `{{$json.component || $json.body.design_data || $json.body}}`
      - System message: include standards via expressions from Workflow Configuration.
      - Connect model/tools/parser to the agent:
        - OpenAI Chat Model → Agent (AI Language Model connection)
        - Performance Simulation Tool → Agent (AI Tool connection)
        - Calculator → Agent (AI Tool connection)
        - Structured Output Parser → Agent (AI Output Parser connection)
      - Connect data inputs:
        - Split Components → Agent
        - Fetch Regulatory Database → Agent

8. **Add aggregation and scoring:**
   1. Add **“Aggregate Results”** (`Aggregate` node):
      - Mode: aggregate all item data
      - Destination field: `output`
      - Connect: Agent → Aggregate Results
   2. Add **“Calculate Risk Score”** (`Code` node):
      - Paste the provided JavaScript risk scoring code.
      - Connect: Aggregate Results → Calculate Risk Score
   3. Add **“Enrich with Metadata”** (`Set` node):
      - Include other fields: ON
      - Add:
        - `evaluation_timestamp` = `{{$now.toISO()}}`
        - (Adjust the other two fields to match your real JSON keys; the provided workflow’s expressions do not match the risk-score node output.)
      - Connect: Calculate Risk Score → Enrich with Metadata

9. **Add compliance decision + logging + response:**
   1. Add **“Check Compliance Status”** (`IF` node):
      - OR conditions:
        - `{{$('Calculate Risk Score').item.json.overall_risk_score}}` > `70`
        - `{{$json.compliance_status}}` == `Non-Compliant` *(recommended to reference the correct item-level field if you keep per-item outputs)*
      - Connect: Enrich with Metadata → Check Compliance Status
   2. Add **“Log Non-Compliant to Database”** (`HTTP Request` node):
      - Method: POST
      - URL: `{{$('Workflow Configuration').first().json.complianceLogUrl}}`
      - Send body fields for vehicle_model/component_id/etc.
      - Configure authentication as required.
      - Connect: Check Compliance Status (true) → Log Non-Compliant to Database
   3. Add **“Respond to Webhook”** (`Respond to Webhook` node):
      - Respond with JSON
      - Body: `{{$json}}`
      - Connect:
        - Check Compliance Status (false) → Respond to Webhook
        - Log Non-Compliant to Database → Respond to Webhook

10. **Validate with test calls** (e.g., Postman/curl):
   - Multi-component payload includes `body.components` array.
   - Single payload includes `body.component_type` and component specs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## How It Works\nThis workflow automates automotive regulatory compliance evaluation by intelligently routing assessments through parallel evaluation paths based on component type. Designed for automotive compliance officers, quality assurance teams, and regulatory affairs managers, it solves the complex challenge of ensuring vehicle components meet diverse regulatory standards across safety, emissions, and performance requirements. The system receives compliance evaluation requests via webhook, determines whether components require split assessment or integrated regulatory database checks, then processes each path using OpenAI-powered compliance agents with specialized tools for performance simulation and structured output parsing. Results are aggregated, risk scores calculated using business rules, enriched with compliance metadata, and logged to regulatory databases while responding to the originating system with actionable compliance status and required remediation actions. | Sticky note (workflow-level description) |
| ## Prerequisites\nActive OpenAI API account, automotive compliance evaluation system with webhook capability\n## Use Cases\nPre-production component compliance validation, supplier part certification\n## Customization\nModify compliance agent prompts for region-specific regulations, adjust risk scoring thresholds\n## Benefits\nAccelerates compliance evaluation by 70%, ensures systematic multi-regulation assessment | Sticky note (project prerequisites/benefits) |
| ## Setup Steps\n1. Configure webhook endpoint URL for compliance evaluation system integration\n2. Set up OpenAI API credentials for Automotive Compliance Agent access\n3. Configure Check Evaluation Type node with component classification rules\n4. Set up Fetch Regulatory Database node with regulatory standards API credentials\n5. Update Performance Simulation Tool with automotive testing parameters\n6. Configure Calculator node with compliance scoring algorithms\n7. Customize Structured Output Parser for regulatory reporting format requirements | Sticky note (setup checklist) |

---

**Compliance disclaimer (provided by user):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.