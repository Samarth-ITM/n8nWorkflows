Moderate user content and route governance decisions with Claude and APIs

https://n8nworkflows.xyz/workflows/moderate-user-content-and-route-governance-decisions-with-claude-and-apis-13154


# Moderate user content and route governance decisions with Claude and APIs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives user-generated content via an HTTP webhook, evaluates it with a two-stage AI approach (validation then governance decision-making), optionally triggers external enforcement/monetization actions via APIs, escalates some cases to human moderators, and writes a structured audit record to an n8n Data Table.

**Target use cases:**  
- Social platforms moderating posts/comments  
- Marketplaces reviewing listings  
- Any UGC pipeline needing consistent enforcement + human escalation + auditability

### 1.1 Input Reception & Policy Configuration
Receives a submission (POST webhook) and injects configuration thresholds and external endpoint URLs that drive downstream agent decisions and API tool calls.

### 1.2 AI Content Validation (Claude) + Structured Parsing
A “Content Validation Agent” uses Claude to score toxicity/spam and flag engagement anomalies and risks. Output is forced into a known JSON schema via a structured output parser.

### 1.3 AI Governance Decisioning (Claude) + Tooling
A “Governance Orchestration Agent” consumes validation output and produces an enforcement decision. It can call two HTTP Request “tools” (enforcement + monetization) if needed. Output is also schema-validated.

### 1.4 Severity Routing, Human Escalation, and Audit Logging
Decisions are routed by severity; if human review is required, moderators are notified. All paths merge and an audit record is prepared and upserted into a Data Table.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Workflow Configuration
**Overview:** Accepts incoming moderation requests and centralizes thresholds + endpoint URLs to keep the rest of the workflow expression-driven and easily configurable.  
**Nodes involved:** Content Submission Webhook, Workflow Configuration

#### Node: Content Submission Webhook
- **Type / role:** `Webhook` trigger; entry point for content submissions.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `content-moderation`
  - **Response Mode:** `lastNode` (the HTTP response will be whatever the final executed node outputs)
- **Inputs / outputs:**
  - **Input:** External HTTP POST request body/params/headers (available in `$json`).
  - **Output:** Passes request payload to **Workflow Configuration**.
- **Failure/edge cases:**
  - If callers don’t send expected fields (e.g., missing `contentId`, `creatorId`, engagement data), downstream agents may hallucinate or output parser may fail.
  - `lastNode` response depends on route taken; if a branch errors, caller receives an error instead of a moderation result.
  - Consider adding authentication (header/token) and request validation if used in production.

#### Node: Workflow Configuration
- **Type / role:** `Set`; injects policy parameters and endpoint placeholders.
- **Key configuration (assigned fields):**
  - `toxicityThreshold` = **0.7**
  - `spamThreshold` = **0.8**
  - `engagementAnomalyThreshold` = **2.5**
  - `monetizationApiUrl` = placeholder string
  - `enforcementApiUrl` = placeholder string
  - `moderatorNotificationUrl` = placeholder string
  - **Include Other Fields:** true (keeps the original webhook submission fields)
- **Inputs / outputs:**
  - **Input:** Webhook submission `$json`
  - **Output:** Combined payload + configuration → **Content Validation Agent**
- **Failure/edge cases:**
  - Placeholder URLs must be replaced or HTTP tool nodes will fail at runtime.
  - If numeric thresholds are changed to non-numeric, agent prompts referencing them remain strings; this can degrade decision quality.

---

### Block 2 — AI Content Validation (Agent + Claude + Structured Output)
**Overview:** Produces a normalized, auditable validation result: toxicity/spam scores, anomaly detection signals, risk level, and reasoning.  
**Nodes involved:** Claude Model - Content Validation, Content Validation Output Parser, Content Validation Agent

#### Node: Claude Model - Content Validation
- **Type / role:** `lmChatAnthropic`; provides the LLM for the validation agent.
- **Key configuration:**
  - Model: `claude-sonnet-4-5-20250929` (cached name: “Claude Sonnet 4.5”)
  - Credentials: Anthropic API credential “Anthropic account”
- **Connections:**
  - Outputs via `ai_languageModel` into **Content Validation Agent**
- **Failure/edge cases:**
  - Credential invalid/expired, quota exceeded, model name unavailable.
  - Latency/timeouts for large payloads (`JSON.stringify($json)` can be big).

#### Node: Content Validation Output Parser
- **Type / role:** `outputParserStructured`; enforces a JSON schema on agent output.
- **Key configuration:**
  - Manual JSON schema requiring (minimum):  
    `contentId`, `toxicityScore`, `spamScore`, `engagementAnomaly`, `riskLevel`, `reasoning`  
  - `riskLevel` enum: `low | medium | high | critical`
- **Connections:**
  - Feeds the agent via `ai_outputParser`
- **Failure/edge cases:**
  - If the model returns non-JSON or violates schema (missing required fields, wrong types), the parser will error and stop the run.
  - Ensure `toxicityScore` and `spamScore` are numeric 0–1; schema does not enforce min/max.

#### Node: Content Validation Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates prompt + LLM + structured parsing.
- **Key configuration choices:**
  - **User text:** `Content Data: {{ JSON.stringify($json) }}` (sends entire inbound payload + config)
  - **System message:** Detailed instructions:
    - Evaluate toxicity/spam/policy violations
    - Evaluate engagement anomalies and behavior signals
    - Use thresholds from **Workflow Configuration** via expressions:
      - `{{ $('Workflow Configuration').first().json.toxicityThreshold }}`
      - `{{ $('Workflow Configuration').first().json.spamThreshold }}`
      - `{{ $('Workflow Configuration').first().json.engagementAnomalyThreshold }}`
    - Return structured JSON matching the output parser schema
  - `hasOutputParser: true` and connected to the structured parser node
- **Inputs / outputs:**
  - **Input:** Output of **Workflow Configuration**
  - **Output:** A LangChain agent result object; downstream nodes use **`$json.output`** as the parsed structured result.
  - Connects to **Governance Orchestration Agent**
- **Failure/edge cases:**
  - Very large incoming payloads can exceed model limits.
  - If inbound payload lacks a reliable `contentId`, the agent must still output one; parser requires it.
  - Expressions referencing `Workflow Configuration` assume it executed and has at least one item.

---

### Block 3 — Governance Orchestration (Agent + Claude + API Tools + Structured Output)
**Overview:** Converts validation results into an enforcement decision, optionally performing external actions through tool calls, and returning an auditable structured decision.  
**Nodes involved:** Claude Model - Governance Orchestration, Governance Decision Output Parser, Governance Orchestration Agent, Monetization API Tool, Enforcement API Tool

#### Node: Claude Model - Governance Orchestration
- **Type / role:** `lmChatAnthropic`; LLM powering the governance agent.
- **Key configuration:**
  - Model: `claude-sonnet-4-5-20250929`
  - Credentials: same Anthropic credential
- **Connections:**
  - `ai_languageModel` → **Governance Orchestration Agent**
- **Failure/edge cases:** Same as other Claude node (auth/quota/model/timeout).

#### Node: Governance Decision Output Parser
- **Type / role:** `outputParserStructured`; schema-validates governance output.
- **Key configuration:**
  - Required fields: `action`, `severity`, `requiresHumanReview`, `monetizationImpact`, `reasoning`
  - `action` enum: `approve | flag | remove | suspend_creator | demonetize`
  - `severity` enum: `low | medium | high`
  - `monetizationImpact` enum: `none | warning | temporary_suspension | permanent_demonetization`
  - Optional `enforcementDetails` object: `reason`, `duration`, `appealable`
- **Connections:**
  - `ai_outputParser` → **Governance Orchestration Agent**
- **Failure/edge cases:**
  - Schema mismatch stops execution.
  - Note: `contentId` is not required in this schema, but later nodes try to access `$json.output.contentId`.

#### Node: Monetization API Tool
- **Type / role:** `httpRequestTool`; tool callable by the governance agent.
- **Key configuration:**
  - URL from config: `{{ $('Workflow Configuration').first().json.monetizationApiUrl }}`
  - Method: POST
  - JSON body built from **$fromAI()** tool arguments:
    - `creatorId`, `contentId`, `monetizationAction`, `reason`, `duration` (default `"N/A"`)
  - Tool description: Adjusts creator monetization status
- **Connections:**
  - Connected as `ai_tool` to **Governance Orchestration Agent** (agent can invoke it)
- **Failure/edge cases:**
  - Placeholder URL not replaced.
  - External API auth not configured (this node has no auth settings shown; add headers/OAuth as needed).
  - `$fromAI()` values depend on the agent choosing to call the tool with correct argument names/types.

#### Node: Enforcement API Tool
- **Type / role:** `httpRequestTool`; tool callable by the governance agent.
- **Key configuration:**
  - URL from config: `{{ $('Workflow Configuration').first().json.enforcementApiUrl }}`
  - Method: POST
  - JSON body uses `$fromAI()` args:
    - `contentId`, `enforcementAction`, `reason`, `appealable` (default `true`)
  - Tool description: Executes content enforcement actions
- **Connections:**
  - `ai_tool` → **Governance Orchestration Agent**
- **Failure/edge cases:**
  - Same external API issues as Monetization tool.
  - The tool accepts `enforcementAction` including “restrict” (per description), but the governance schema’s `action` enum uses `flag/remove/...`; ensure the agent maps correctly (e.g., decision action `remove` might call enforcement tool with `enforcementAction=remove`).

#### Node: Governance Orchestration Agent
- **Type / role:** LangChain agent; produces final governance decision, can call tools.
- **Key configuration choices:**
  - **User text:** `Validation Results: {{ JSON.stringify($json.output) }}`
    - Assumes upstream **Content Validation Agent** output is in `$json.output`.
  - **System message:** Enforcement guidelines, human review rules, action definitions, transparency requirements.
  - `hasOutputParser: true` with the Governance Decision Output Parser.
  - Two tools available (Monetization + Enforcement).
- **Inputs / outputs:**
  - **Input:** Result from **Content Validation Agent**
  - **Output:** Structured decision in `$json.output` → **Route by Severity**
- **Failure/edge cases:**
  - If upstream validation parsing fails, governance never runs.
  - Tool-calling can fail due to external API errors; depending on agent/tool error handling, the whole run may fail.
  - Governance schema does not require `contentId`; downstream notification tries `$json.output.contentId`.

---

### Block 4 — Severity Routing, Human Review Escalation, Merge, and Audit Persistence
**Overview:** Routes by severity, triggers human moderation when required, merges all outcomes, and stores an audit entry in a Data Table.  
**Nodes involved:** Route by Severity, Check Human Review Required, Notify Human Moderators, Merge All Paths, Prepare Audit Record, Audit Log Storage

#### Node: Route by Severity
- **Type / role:** `Switch`; branches based on decision severity.
- **Key configuration:**
  - Compares `{{ $json.output.severity }}` to:
    - `low` → output “Low Severity”
    - `medium` → output “Medium Severity”
    - `high` → output “High Severity” (this path leads to human review check)
  - Ignore case enabled; fallback output renamed “Unknown Severity”
- **Inputs / outputs:**
  - **Input:** Governance agent result
  - **Outputs:**
    - Low → **Merge All Paths** (input index 0)
    - Medium → **Merge All Paths** (input index 1)
    - High → **Check Human Review Required**
- **Failure/edge cases:**
  - If `$json.output.severity` missing or not one of the expected values, it goes to fallback (“Unknown Severity”), which is **not connected**. That would end execution early (and affect webhook response).

#### Node: Check Human Review Required
- **Type / role:** `If`; decides whether to notify moderators.
- **Key configuration:**
  - Condition: `{{ $json.output.requiresHumanReview }} == true`
- **Inputs / outputs:**
  - **Input:** High-severity path from switch
  - **True:** → **Notify Human Moderators**
  - **False:** → **Merge All Paths** (input index 2)
- **Failure/edge cases:**
  - If `requiresHumanReview` is missing, it evaluates falsey and skips notification.
  - Only high severity reaches this node; medium/low cannot trigger human review in the current wiring.

#### Node: Notify Human Moderators
- **Type / role:** `HTTP Request`; sends case details to a human moderation endpoint.
- **Key configuration:**
  - URL: `{{ $('Workflow Configuration').first().json.moderatorNotificationUrl }}`
  - Method: POST, JSON body includes:
    - `contentId`: `$json.output.contentId || $json.contentId`
    - `severity`, `action`
    - `validationResults`: `$('Content Validation Agent').first().json.output`
    - `governanceDecision`: `$json.output`
    - `timestamp`: `{{ $now.toISO() }}`
    - `priority`: urgent if severity is high
- **Inputs / outputs:**
  - **Input:** IF-true branch
  - **Output:** → **Merge All Paths** (input index 3)
- **Failure/edge cases:**
  - Placeholder URL not replaced.
  - If governance output lacks `contentId`, it falls back to `$json.contentId` (may still be missing).
  - If Content Validation Agent output is not available (failed upstream), expression `$('Content Validation Agent').first()` errors.

#### Node: Merge All Paths
- **Type / role:** `Merge`; converges up to 4 paths for unified auditing.
- **Key configuration:**
  - `numberInputs: 4`
- **Inputs / outputs:**
  - Input 0: Low severity
  - Input 1: Medium severity
  - Input 2: High severity without human review
  - Input 3: Human review notification path
  - Output → **Prepare Audit Record**
- **Failure/edge cases:**
  - Merge behavior can be sensitive to item counts; here each path is typically 1 item, so it’s fine.
  - If one expected input never arrives, merge still can proceed depending on merge mode defaults; verify behavior in your n8n version.

#### Node: Prepare Audit Record
- **Type / role:** `Set`; formats a single record for persistence.
- **Key configuration (assigned fields):**
  - `auditTimestamp` = `{{ $now.toISO() }}`
  - `contentId` = `{{ $json.output?.contentId || $json.contentId || 'unknown' }}`
  - `validationResults` = `{{ JSON.stringify($('Content Validation Agent').first().json.output) }}`
  - `governanceDecision` = `{{ JSON.stringify($json.output) }}`
  - `humanReviewTriggered` = `{{ $('Check Human Review Required').item.json.output?.requiresHumanReview || false }}`
  - `workflowStatus` = `completed`
  - Include other fields: true
- **Inputs / outputs:**
  - Input: Merged output
  - Output: → **Audit Log Storage**
- **Failure/edge cases:**
  - `$('Check Human Review Required').item...` can be problematic if that node did not run in low/medium branches; this expression may error in some n8n contexts. Safer alternatives:
    - Use `$('Check Human Review Required').first()` with existence checks, or
    - Set `humanReviewTriggered` based on current item content.
  - Storing `validationResults`/`governanceDecision` as strings while field type is “object” can cause type inconsistencies depending on Data Table schema.

#### Node: Audit Log Storage
- **Type / role:** `Data Table`; persists audit record.
- **Key configuration:**
  - Operation: `upsert`
  - Data Table: `content_moderation_audit_log` (selected by name)
- **Inputs / outputs:**
  - Input: Prepared audit item
  - Output: Final node (also becomes webhook response because webhook uses `lastNode`)
- **Failure/edge cases:**
  - Data table missing or schema mismatch (required columns not present, wrong types).
  - Upsert needs a defined key strategy in the Data Table; if not configured, upsert may not behave as expected.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Content Submission Webhook | n8n-nodes-base.webhook | Receives content submissions (HTTP POST trigger) | — | Workflow Configuration | ## How It Works<br>This workflow automates intelligent content moderation and governance enforcement through multi-model AI validation. Designed for social media platforms, online communities, and user-generated content platforms, it solves the critical challenge of scaling content review while maintaining consistent policy enforcement and human oversight for edge cases. The system receives content submissions via webhook, processing them through a dual-agent AI framework for content validation and governance orchestration. It employs specialized AI models for policy violation detection, moderation API enforcement checks, and governance decision-making. The workflow intelligently routes content based on severity classification, escalating high-risk submissions for human moderator review while auto-processing clear-cut decisions. By merging parallel validation paths and maintaining comprehensive audit logs, it ensures consistent policy application across all content while preserving human judgment for nuanced cases requiring contextual understanding. |
| Workflow Configuration | n8n-nodes-base.set | Sets thresholds and endpoint URLs | Content Submission Webhook | Content Validation Agent | ## Setup Steps<br>1. Configure Content Submission Webhook trigger endpoint<br>2. Connect Workflow Configuration node with content policy parameters<br>3. Set up Content Validation Agent with Claude/OpenAI API credentials<br>4. Configure parallel AI processing nodes<br>5. Connect Governance Orchestration Agent with AI API credentials<br>6. Set up multi-model validation<br>7. Configure Route by Severity node with classification thresholds |
| Content Validation Agent | @n8n/n8n-nodes-langchain.agent | AI validation of content + structured output | Workflow Configuration | Governance Orchestration Agent | ## Dual-Agent Content Analysis<br>**What:** Processes submissions through parallel AI agents for content policy validation and governance orchestration with specialized output parsing<br>**Why:** Separates technical violation detection from governance decision-making to ensure thorough evaluation across compliance and contextual dimensions |
| Claude Model - Content Validation | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude LLM for validation agent | — | Content Validation Agent (ai_languageModel) | ## Multi-Model Policy Enforcement<br>**What:** Validates content through Claude AI, moderation APIs, and reinforcement learning models for comprehensive policy coverage<br>**Why:** Leverages specialized models for different violation types ensuring no policy breach escapes detection through single-model limitations |
| Content Validation Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces validation JSON schema | — | Content Validation Agent (ai_outputParser) | ## Dual-Agent Content Analysis<br>**What:** Processes submissions through parallel AI agents for content policy validation and governance orchestration with specialized output parsing<br>**Why:** Separates technical violation detection from governance decision-making to ensure thorough evaluation across compliance and contextual dimensions |
| Governance Orchestration Agent | @n8n/n8n-nodes-langchain.agent | Governance decision-making + tool calling | Content Validation Agent | Route by Severity | ## Dual-Agent Content Analysis<br>**What:** Processes submissions through parallel AI agents for content policy validation and governance orchestration with specialized output parsing<br>**Why:** Separates technical violation detection from governance decision-making to ensure thorough evaluation across compliance and contextual dimensions |
| Claude Model - Governance Orchestration | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude LLM for governance agent | — | Governance Orchestration Agent (ai_languageModel) | ## Multi-Model Policy Enforcement<br>**What:** Validates content through Claude AI, moderation APIs, and reinforcement learning models for comprehensive policy coverage<br>**Why:** Leverages specialized models for different violation types ensuring no policy breach escapes detection through single-model limitations |
| Governance Decision Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces governance decision JSON schema | — | Governance Orchestration Agent (ai_outputParser) | ## Dual-Agent Content Analysis<br>**What:** Processes submissions through parallel AI agents for content policy validation and governance orchestration with specialized output parsing<br>**Why:** Separates technical violation detection from governance decision-making to ensure thorough evaluation across compliance and contextual dimensions |
| Monetization API Tool | n8n-nodes-base.httpRequestTool | Tool for monetization actions (agent-callable) | — | Governance Orchestration Agent (ai_tool) | ## Multi-Model Policy Enforcement<br>**What:** Validates content through Claude AI, moderation APIs, and reinforcement learning models for comprehensive policy coverage<br>**Why:** Leverages specialized models for different violation types ensuring no policy breach escapes detection through single-model limitations |
| Enforcement API Tool | n8n-nodes-base.httpRequestTool | Tool for enforcement actions (agent-callable) | — | Governance Orchestration Agent (ai_tool) | ## Multi-Model Policy Enforcement<br>**What:** Validates content through Claude AI, moderation APIs, and reinforcement learning models for comprehensive policy coverage<br>**Why:** Leverages specialized models for different violation types ensuring no policy breach escapes detection through single-model limitations |
| Route by Severity | n8n-nodes-base.switch | Branches flow by severity | Governance Orchestration Agent | Merge All Paths; Check Human Review Required | ## Severity-Based Human Escalation<br>**What:** Routes content based on severity scores with automatic human moderator notification for edge cases requiring judgment<br>**Why:** Balances automation efficiency with human oversight ensuring nuanced decisions receive appropriate review while clear violations process instantly |
| Check Human Review Required | n8n-nodes-base.if | Determines if moderator escalation is required | Route by Severity (High) | Notify Human Moderators; Merge All Paths | ## Severity-Based Human Escalation<br>**What:** Routes content based on severity scores with automatic human moderator notification for edge cases requiring judgment<br>**Why:** Balances automation efficiency with human oversight ensuring nuanced decisions receive appropriate review while clear violations process instantly |
| Notify Human Moderators | n8n-nodes-base.httpRequest | Sends case package to human moderators | Check Human Review Required (true) | Merge All Paths | ## Severity-Based Human Escalation<br>**What:** Routes content based on severity scores with automatic human moderator notification for edge cases requiring judgment<br>**Why:** Balances automation efficiency with human oversight ensuring nuanced decisions receive appropriate review while clear violations process instantly |
| Merge All Paths | n8n-nodes-base.merge | Consolidates all routed outcomes | Route by Severity; Check Human Review Required; Notify Human Moderators | Prepare Audit Record | ## Severity-Based Human Escalation<br>**What:** Routes content based on severity scores with automatic human moderator notification for edge cases requiring judgment<br>**Why:** Balances automation efficiency with human oversight ensuring nuanced decisions receive appropriate review while clear violations process instantly |
| Prepare Audit Record | n8n-nodes-base.set | Normalizes final audit record fields | Merge All Paths | Audit Log Storage |  |
| Audit Log Storage | n8n-nodes-base.dataTable | Upserts audit record into Data Table | Prepare Audit Record | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Project notes | — | — | ## Prerequisites<br>Claude/OpenAI API credentials for content validation, moderation API access for policy enforcement<br>## Use Cases<br>Social media platforms moderating user posts and comments, online marketplaces reviewing product listings<br>## Customization<br>Adjust severity thresholds for platform-specific risk tolerance<br>## Benefits<br>Reduces content review time by 85%, ensures consistent policy enforcement across all submissions |
| Sticky Note1 | n8n-nodes-base.stickyNote | Setup notes | — | — | ## Setup Steps<br>1. Configure Content Submission Webhook trigger endpoint<br>2. Connect Workflow Configuration node with content policy parameters<br>3. Set up Content Validation Agent with Claude/OpenAI API credentials<br>4. Configure parallel AI processing nodes<br>5. Connect Governance Orchestration Agent with AI API credentials<br>6. Set up multi-model validation<br>7. Configure Route by Severity node with classification thresholds |
| Sticky Note2 | n8n-nodes-base.stickyNote | How-it-works notes | — | — | ## How It Works<br>This workflow automates intelligent content moderation and governance enforcement through multi-model AI validation. Designed for social media platforms, online communities, and user-generated content platforms, it solves the critical challenge of scaling content review while maintaining consistent policy enforcement and human oversight for edge cases. The system receives content submissions via webhook, processing them through a dual-agent AI framework for content validation and governance orchestration. It employs specialized AI models for policy violation detection, moderation API enforcement checks, and governance decision-making. The workflow intelligently routes content based on severity classification, escalating high-risk submissions for human moderator review while auto-processing clear-cut decisions. By merging parallel validation paths and maintaining comprehensive audit logs, it ensures consistent policy application across all content while preserving human judgment for nuanced cases requiring contextual understanding. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Severity routing notes | — | — | ## Severity-Based Human Escalation<br>**What:** Routes content based on severity scores with automatic human moderator notification for edge cases requiring judgment<br>**Why:** Balances automation efficiency with human oversight ensuring nuanced decisions receive appropriate review while clear violations process instantly |
| Sticky Note4 | n8n-nodes-base.stickyNote | Multi-model enforcement notes | — | — | ## Multi-Model Policy Enforcement<br>**What:** Validates content through Claude AI, moderation APIs, and reinforcement learning models for comprehensive policy coverage<br>**Why:** Leverages specialized models for different violation types ensuring no policy breach escapes detection through single-model limitations |
| Sticky Note5 | n8n-nodes-base.stickyNote | Dual-agent analysis notes | — | — | ## Dual-Agent Content Analysis<br>**What:** Processes submissions through parallel AI agents for content policy validation and governance orchestration with specialized output parsing<br>**Why:** Separates technical violation detection from governance decision-making to ensure thorough evaluation across compliance and contextual dimensions |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `AI-Powered Content Moderation and Governance Orchestration System`
   - (Optional) Keep it inactive while building.

2. **Add trigger: Webhook**
   - Node: **Webhook**
   - Name: `Content Submission Webhook`
   - HTTP Method: **POST**
   - Path: `content-moderation`
   - Response Mode: **Last Node**
   - (Recommended) Add authentication (header token) in Webhook node options if exposed publicly.

3. **Add configuration: Set**
   - Node: **Set**
   - Name: `Workflow Configuration`
   - Add fields:
     - `toxicityThreshold` (Number) = `0.7`
     - `spamThreshold` (Number) = `0.8`
     - `engagementAnomalyThreshold` (Number) = `2.5`
     - `monetizationApiUrl` (String) = your Monetization API endpoint
     - `enforcementApiUrl` (String) = your Enforcement API endpoint
     - `moderatorNotificationUrl` (String) = your moderator notification webhook endpoint
   - Enable **Include Other Fields**.
   - Connect: **Content Submission Webhook → Workflow Configuration**

4. **Add the Anthropic chat model for validation**
   - Node: **Anthropic Chat Model** (LangChain Anthropic chat node)
   - Name: `Claude Model - Content Validation`
   - Model: `claude-sonnet-4-5-20250929` (or closest available equivalent)
   - Credentials: configure **Anthropic API** credential in n8n (API key)

5. **Add structured output parser for validation**
   - Node: **Structured Output Parser**
   - Name: `Content Validation Output Parser`
   - Schema type: **Manual**
   - Paste the validation JSON schema (object with required: `contentId`, `toxicityScore`, `spamScore`, `engagementAnomaly`, `riskLevel`, `reasoning`, etc.)

6. **Add validation agent**
   - Node: **AI Agent** (LangChain Agent)
   - Name: `Content Validation Agent`
   - Prompt type: **Define**
   - Text:
     - `Content Data: {{ JSON.stringify($json) }}`
   - System message: include the instructions from the workflow (toxicity/spam/anomaly detection, thresholds referencing `Workflow Configuration`).
   - Enable **Output Parser** and connect:
     - `Claude Model - Content Validation` → agent **ai_languageModel**
     - `Content Validation Output Parser` → agent **ai_outputParser**
   - Connect main flow:
     - `Workflow Configuration → Content Validation Agent`

7. **Add the Anthropic chat model for governance**
   - Node: **Anthropic Chat Model**
   - Name: `Claude Model - Governance Orchestration`
   - Model: `claude-sonnet-4-5-20250929`
   - Credentials: same Anthropic credential

8. **Add structured output parser for governance**
   - Node: **Structured Output Parser**
   - Name: `Governance Decision Output Parser`
   - Schema type: Manual
   - Paste governance schema (required: `action`, `severity`, `requiresHumanReview`, `monetizationImpact`, `reasoning`)

9. **Add tool nodes (HTTP Request Tool)**
   - Node: **HTTP Request Tool**
     - Name: `Monetization API Tool`
     - Method: POST
     - URL: `{{ $('Workflow Configuration').first().json.monetizationApiUrl }}`
     - Body: JSON using `$fromAI()` fields for `creatorId`, `contentId`, `monetizationAction`, `reason`, `duration`
   - Node: **HTTP Request Tool**
     - Name: `Enforcement API Tool`
     - Method: POST
     - URL: `{{ $('Workflow Configuration').first().json.enforcementApiUrl }}`
     - Body: JSON using `$fromAI()` fields for `contentId`, `enforcementAction`, `reason`, `appealable`
   - Configure auth headers/OAuth inside these nodes if your APIs require it.

10. **Add governance agent**
   - Node: **AI Agent**
   - Name: `Governance Orchestration Agent`
   - Text: `Validation Results: {{ JSON.stringify($json.output) }}`
   - System message: enforcement guidelines + escalation rules (as in workflow)
   - Enable Output Parser and connect:
     - `Claude Model - Governance Orchestration` → agent **ai_languageModel**
     - `Governance Decision Output Parser` → agent **ai_outputParser**
     - `Monetization API Tool` and `Enforcement API Tool` → agent **ai_tool**
   - Connect main flow:
     - `Content Validation Agent → Governance Orchestration Agent`

11. **Add severity routing**
   - Node: **Switch**
   - Name: `Route by Severity`
   - Rules comparing `{{ $json.output.severity }}` to `low`, `medium`, `high`
   - Enable ignore case; define a fallback “Unknown Severity” (recommended: connect fallback to merge to avoid dead-end).
   - Connect:
     - `Governance Orchestration Agent → Route by Severity`

12. **Add human review check (for high severity)**
   - Node: **IF**
   - Name: `Check Human Review Required`
   - Condition: boolean equals → `{{ $json.output.requiresHumanReview }}` is `true`
   - Connect:
     - Switch “High Severity” output → `Check Human Review Required`

13. **Add moderator notification**
   - Node: **HTTP Request**
   - Name: `Notify Human Moderators`
   - Method: POST
   - URL: `{{ $('Workflow Configuration').first().json.moderatorNotificationUrl }}`
   - JSON body with contentId, severity, action, validation results, decision, timestamp, priority (as in workflow).
   - Connect:
     - IF true → `Notify Human Moderators`

14. **Merge all outcomes**
   - Node: **Merge**
   - Name: `Merge All Paths`
   - Number of inputs: **4**
   - Connect:
     - Switch low → Merge input 0
     - Switch medium → Merge input 1
     - IF false → Merge input 2
     - Notify Human Moderators → Merge input 3

15. **Prepare audit record**
   - Node: **Set**
   - Name: `Prepare Audit Record`
   - Fields:
     - `auditTimestamp` = `{{ $now.toISO() }}`
     - `contentId` = `{{ $json.output?.contentId || $json.contentId || 'unknown' }}`
     - `validationResults` = `{{ JSON.stringify($('Content Validation Agent').first().json.output) }}`
     - `governanceDecision` = `{{ JSON.stringify($json.output) }}`
     - `humanReviewTriggered` = (replicate expression, but consider making it branch-safe)
     - `workflowStatus` = `completed`
   - Connect: `Merge All Paths → Prepare Audit Record`

16. **Create / configure Data Table and store**
   - Create an n8n **Data Table** named: `content_moderation_audit_log`
   - Ensure columns exist for the fields above (string fields are safest for JSON blobs).
   - Node: **Data Table**
     - Name: `Audit Log Storage`
     - Operation: **Upsert**
     - Data Table: `content_moderation_audit_log`
   - Connect: `Prepare Audit Record → Audit Log Storage`

17. **Activate and test**
   - Execute a test POST to `/webhook/content-moderation` with sample JSON including at least `contentId`, `creatorId`, and content text + engagement metrics.
   - Verify:
     - Governance output severity routes correctly
     - Human notifications fire when `requiresHumanReview=true`
     - Audit record is created/upserted
     - Webhook response returns the final Data Table output (or adjust response mode if you want decision output instead)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Prerequisites: Claude/OpenAI API credentials for content validation, moderation API access for policy enforcement | Sticky note “Prerequisites” |
| Use cases: social media post/comment moderation; marketplace listing review | Sticky note “Use Cases” |
| Customization: adjust severity thresholds for platform-specific risk tolerance | Sticky note “Customization” |
| Benefits claim: reduces content review time by 85%, consistent enforcement | Sticky note “Benefits” |
| Design note: dual-agent approach (validation vs governance) improves separation of concerns | Sticky notes “Dual-Agent Content Analysis” / “Multi-Model Policy Enforcement” |
| Operational note: severity-based routing with human escalation for edge cases | Sticky note “Severity-Based Human Escalation” |