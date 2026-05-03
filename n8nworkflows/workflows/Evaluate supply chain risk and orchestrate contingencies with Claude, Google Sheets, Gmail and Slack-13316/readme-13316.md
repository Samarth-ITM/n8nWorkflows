Evaluate supply chain risk and orchestrate contingencies with Claude, Google Sheets, Gmail and Slack

https://n8nworkflows.xyz/workflows/evaluate-supply-chain-risk-and-orchestrate-contingencies-with-claude--google-sheets--gmail-and-slack-13316


# Evaluate supply chain risk and orchestrate contingencies with Claude, Google Sheets, Gmail and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Evaluate supply chain risk and orchestrate contingencies with Claude, Google Sheets, Gmail and Slack  
**Workflow name (in JSON):** AI-Powered Supply Chain Risk Evaluation and Contingency Orchestration

This workflow runs on a schedule (hourly), pulls supply-chain risk indicators from Google Sheets, aggregates them into structured metrics, asks Claude (Anthropic) to produce a structured risk assessment, routes execution by severity (LOW / MEDIUM / CRITICAL), and then:
- For **LOW** risk: sends a Slack acknowledgement message.
- For **MEDIUM/CRITICAL** risk: uses a **Coordination Agent** that can invoke tools (Slack notification, Gmail approval requests, and an Impact Assessment agent tool) and returns structured coordination outcomes.
Finally, results are merged and logged back into a Google Sheet for auditability.

### 1.1 Scheduling & Configuration
- Hourly trigger.
- Centralized configuration variables (sheet IDs, Slack channel, recipients, thresholds).

### 1.2 Data Retrieval & Risk Indicator Aggregation
- Read “Risk Indicators” sheet.
- Convert multiple rows into one aggregated risk object with computed averages and an overall numeric score.

### 1.3 AI Risk Assessment (Claude) + Structured Parsing
- Claude “Risk Signal Agent” generates structured JSON: per-domain scores, overall level, factors, contractual violations, reasoning.
- Strict schema validation via output parser.

### 1.4 Severity Routing & Response Orchestration
- Switch routes by `overallRiskLevel`.
- LOW → direct Slack message.
- MEDIUM/CRITICAL → Coordination Agent that can call Slack/Gmail/Impact tools and returns structured actions.

### 1.5 Results Management (Merge + Log to Sheets)
- Merge outputs and append/update into a “Risk Assessment Log” Google Sheet.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduling & Global Configuration
**Overview:** Starts the workflow hourly and sets reusable configuration values (sheet IDs, Slack channel ID, thresholds, and approval recipients).

**Nodes involved:**
- Schedule Trigger
- Workflow Configuration

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point, time-based execution.
- **Configuration (interpreted):** Runs every **1 hour** (interval rule with `field: "hours"`).
- **Inputs / outputs:** No inputs. Output → **Workflow Configuration**.
- **Version:** typeVersion **1.3**
- **Potential failures / edge cases:**
  - Node disabled workflow: the workflow is currently `active: false` (must be activated to run on schedule).
  - Timezone behavior depends on n8n instance settings.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines constants used throughout the workflow.
- **Key fields created (includeOtherFields = true):**
  - `riskDataSheetId` (placeholder) — Google Sheet ID for risk indicators.
  - `logSheetId` (placeholder) — Google Sheet ID for logging.
  - `slackChannelId` (placeholder) — target Slack channel.
  - `approvalEmailRecipients` (placeholder) — comma-separated emails.
  - `criticalRiskThreshold` = **80**
  - `mediumRiskThreshold` = **50**
- **Expressions / variables used later:**
  - `$('Workflow Configuration').first().json.riskDataSheetId`
  - `$('Workflow Configuration').first().json.logSheetId`
  - `$('Workflow Configuration').first().json.slackChannelId`
- **Inputs / outputs:** Input from **Schedule Trigger**, output → **Fetch Risk Data**.
- **Version:** typeVersion **3.4**
- **Potential failures / edge cases:**
  - Placeholders must be replaced; otherwise downstream nodes will fail (invalid Sheet ID / channel ID / recipients).

**Sticky notes covering this area (contextual):**
- **“How It Works”** (Sticky Note2): describes the 3-tier routing concept and audit logging.
- **“Setup Steps”** (Sticky Note1): lists required account connections and mentions “parser regex patterns in Code nodes” (note: this workflow uses structured output parsers, not regex).
- **“Prerequisites / Use Cases / Customization / Benefits”** (Sticky Note): high-level requirements and customization suggestions.
- **“Risk Indicator Calculation”** (Sticky Note4): explains why aggregation/classification exists.

---

### Block 2.2 — Google Sheets Risk Data Retrieval
**Overview:** Reads tabular risk indicators from Google Sheets to supply the aggregation step.

**Nodes involved:**
- Fetch Risk Data

#### Node: Fetch Risk Data
- **Type / role:** `Google Sheets` — reads rows from a sheet.
- **Configuration (interpreted):**
  - Document ID: `={{ $('Workflow Configuration').first().json.riskDataSheetId }}`
  - Sheet name: **“Risk Indicators”**
  - Operation not explicitly shown (defaults typically to “Read” / “Get many” depending on node defaults). In practice: expects to output multiple items (rows).
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Inputs / outputs:** Input from **Workflow Configuration**, output → **Aggregate Risk Indicators**
- **Version:** typeVersion **4.7**
- **Potential failures / edge cases:**
  - OAuth scope/consent issues, expired token.
  - Wrong sheet name (“Risk Indicators”) or missing tab.
  - Data typing: Google Sheets often returns numbers as strings; downstream code assumes numeric comparisons (see next block).

---

### Block 2.3 — Risk Indicator Aggregation & Scoring (Code)
**Overview:** Converts many sheet rows into one structured `riskData` object with category metrics, averages, and a computed `overallRiskScore`.

**Nodes involved:**
- Aggregate Risk Indicators

#### Node: Aggregate Risk Indicators
- **Type / role:** `Code` — performs custom aggregation and computation.
- **Key logic (interpreted):**
  - Initializes `aggregatedRisk` object with nested categories:
    - `supplier`: totalSuppliers, highRiskSuppliers, averageLeadTime, qualityIssues
    - `logistics`: delayedShipments, totalShipments, averageDelayDays, routeDisruptions
    - `inventory`: lowStockItems, totalItems, stockoutRisk, averageStockLevel
    - `rawData`: array of all original rows
  - Iterates over all incoming items:
    - Supplier:
      - increments `totalSuppliers` if `supplierRiskScore` exists
      - counts `highRiskSuppliers` if `supplierRiskScore > 70`
      - sums `leadTime`
      - counts `qualityIssues` if boolean true or string `"true"`
    - Logistics:
      - increments shipments; counts delayed if `shipmentStatus === 'delayed'`
      - sums delay days if positive
      - counts route disruptions if true or `"true"`
    - Inventory:
      - increments items; sums stockLevel
      - counts low stock if `stockLevel < reorderPoint`
      - sums `stockoutRisk`
  - Computes averages:
    - supplier avg lead time over total suppliers
    - logistics avg delay days over delayed shipments only
    - inventory avg stock and avg stockout risk over total items
  - Computes `overallRiskScore` (0–100-ish scale, weighted):
    - 40% supplier high-risk ratio
    - 30% logistics delay ratio
    - 30% inventory low-stock ratio
  - Adds `timestamp = new Date().toISOString()`
  - Returns **a single item** shaped as: `{ riskData: aggregatedRisk }`
- **Inputs / outputs:** Input from **Fetch Risk Data**, output → **Risk Signal Agent**
- **Version:** typeVersion **2**
- **Potential failures / edge cases:**
  - **Type coercion:** If Sheets returns `"75"` (string), `data.supplierRiskScore > 70` works due to JS coercion, but blanks/`""` can behave unexpectedly; consider explicit `Number(...)`.
  - Missing `reorderPoint` could produce `stockLevel < undefined` (false) and undercount low stock.
  - If no suppliers/logistics/items exist, divisions are protected via `Math.max(...,1)` but the semantics may be skewed (e.g., risk ratio becomes 0).
  - The computed `overallRiskScore` here is different from the AI agent’s `overallRiskScore` output; be clear which is used downstream.

**Sticky note (contextual):**
- **“Risk Indicator Calculation”** (Sticky Note4) applies directly: this block enables severity routing.

---

### Block 2.4 — AI Risk Signal Assessment (Claude + Structured Parser)
**Overview:** Claude analyzes the aggregated risk indicators and returns a strict JSON assessment with overall level, scores, risk factors, contractual violations, and reasoning.

**Nodes involved:**
- Risk Signal Agent
- Anthropic Model - Risk Agent
- Risk Assessment Output Parser

#### Node: Risk Signal Agent
- **Type / role:** `LangChain Agent` (`@n8n/n8n-nodes-langchain.agent`) — prompts the model and produces tool-less structured output.
- **Prompt content:**
  - User text: `Risk Indicators Data: {{ JSON.stringify($json.riskData) }}`
  - System message mandates:
    - category evaluation and scoring (0–100)
    - overall level mapping: LOW 0–49, MEDIUM 50–79, CRITICAL 80–100
    - identify risk factors, contractual boundary violations
    - return structured JSON
    - **constraint:** do not recommend actions violating agreements
- **Output parsing:** `hasOutputParser: true` → uses **Risk Assessment Output Parser**
- **Inputs / outputs:** Input from **Aggregate Risk Indicators**, output → **Route by Risk Level**
- **Version:** typeVersion **3.1**
- **Potential failures / edge cases:**
  - Model may output invalid JSON or missing required fields → parser failure.
  - Hallucinated “contractual violations” if upstream data does not contain contract/SLA facts (consider adding contract data fields to reduce speculation).

#### Node: Anthropic Model - Risk Agent
- **Type / role:** `LM Chat Anthropic` — provides Claude model to the agent.
- **Configuration (interpreted):** Model = `claude-sonnet-4-5-20250929` (cached label “Claude Sonnet 4.5”).
- **Credentials:** Anthropic API (`Anthropic account`)
- **Connections:** Connected to **Risk Signal Agent** via `ai_languageModel`.
- **Version:** typeVersion **1.3**
- **Potential failures / edge cases:**
  - API key / billing / rate limits.
  - Model availability changes; pin to a stable model if required.

#### Node: Risk Assessment Output Parser
- **Type / role:** `Structured Output Parser` — validates and parses the model response into a typed object.
- **Schema (required fields):**
  - `overallRiskLevel` (LOW/MEDIUM/CRITICAL)
  - `overallRiskScore` (number)
  - `supplierRiskScore`, `logisticsRiskScore`, `inventoryRiskScore` (numbers)
  - `riskFactors` (string array)
  - `contractualViolations` (string array)
  - `reasoning` (string)
- **Connections:** Connected to **Risk Signal Agent** via `ai_outputParser`.
- **Version:** typeVersion **1.3**
- **Potential failures / edge cases:**
  - Strict schema: any missing/extra formatting can fail parsing.
  - If the agent returns numbers as strings, parser may reject depending on implementation strictness.

---

### Block 2.5 — Risk Severity Routing (Switch)
**Overview:** Routes execution into LOW, MEDIUM, or CRITICAL paths based on the parsed AI output.

**Nodes involved:**
- Route by Risk Level

#### Node: Route by Risk Level
- **Type / role:** `Switch` — conditional branching.
- **Configuration (interpreted):**
  - Evaluates `={{ $json.output.overallRiskLevel }}`
  - Outputs:
    - “Critical Risk” when equals `CRITICAL`
    - “Medium Risk” when equals `MEDIUM`
    - “Low Risk” when equals `LOW`
  - Fallback output renamed to “Unclassified”
- **Connections (important):**
  - Critical Risk → **Coordination Agent**
  - Medium Risk → **Coordination Agent**
  - Low Risk → **Low Risk Notification**
- **Version:** typeVersion **3.4**
- **Potential failures / edge cases:**
  - If the parser output is not located at `$json.output` (depends on node behavior), expression may be wrong. This workflow assumes the agent’s parsed result is available under `output`.
  - Any unexpected label (e.g., “HIGH”) goes to fallback (“Unclassified”) and is not connected → execution may stop silently for that path.

---

### Block 2.6 — MEDIUM/CRITICAL Contingency Orchestration (Coordination Agent + Tools)
**Overview:** For MEDIUM or CRITICAL risk, a coordination agent determines actions, sends Slack notifications, requests approvals via Gmail, optionally performs impact analysis via a dedicated impact agent tool, then returns structured coordination output.

**Nodes involved:**
- Coordination Agent
- Anthropic Model - Coordination Agent
- Coordination Output Parser
- Slack Notification Tool
- Gmail Approval Tool
- Impact Assessment Agent Tool
- Anthropic Model - Impact Agent
- Impact Assessment Output Parser

#### Node: Coordination Agent
- **Type / role:** `LangChain Agent` — orchestrates multi-step actions using tools.
- **Prompt content:**
  - `Risk Assessment: {{ JSON.stringify($json.output) }}`
  - `Risk Data: {{ JSON.stringify($json.riskData) }}`
- **System message mandates:**
  - choose contingencies based on severity/factors
  - use tools:
    - Slack Notification Tool
    - Gmail Approval Tool
    - Impact Assessment Agent Tool
  - preserve contractual boundaries; approval required before contract modifications
  - return structured output of actions/notifications/constraints/next steps
- **Tools available (via ai_tool connections):**
  - Slack Notification Tool
  - Gmail Approval Tool
  - Impact Assessment Agent Tool
- **Output parsing:** `hasOutputParser: true` → **Coordination Output Parser**
- **Inputs / outputs:** Input from **Route by Risk Level** (MEDIUM/CRITICAL outputs), output → **Merge Results**
- **Version:** typeVersion **3.1**
- **Potential failures / edge cases:**
  - If the agent decides not to call tools, you may still want explicit “no-op” actions for traceability.
  - Tool invocation depends on correct `$fromAI()` parameter extraction in tool nodes (see below).
  - The agent is used for both MEDIUM and CRITICAL without explicit differentiation besides prompt content; consider adding explicit instruction to vary actions by level.

**Sticky note (contextual):**
- **“Critical Risk Processing”** (Sticky Note3): explains why deeper evaluation happens (multi-perspective via coordinated agents).
- **“Medium Risk Evaluation”** (Sticky Note5): mentions a single-agent approach; in this workflow, MEDIUM also goes to the same Coordination Agent as CRITICAL.

#### Node: Anthropic Model - Coordination Agent
- **Type / role:** `LM Chat Anthropic` — Claude model backing the coordination agent.
- **Model:** `claude-sonnet-4-5-20250929`
- **Credentials:** Anthropic API
- **Connections:** `ai_languageModel` → Coordination Agent
- **Version:** typeVersion **1.3**
- **Potential failures:** Same as other Anthropic nodes (quota, model availability).

#### Node: Coordination Output Parser
- **Type / role:** `Structured Output Parser` — validates final coordination results.
- **Schema (required):**
  - `actionsTaken` (string array)
  - `approvalsRequested` (string array)
  - `notificationsSent` (string array)
  - `contractualConstraints` (string array)
  - `recommendedNextSteps` (string array)
  - `impactAssessmentSummary` (string) is defined but **not required**
- **Connections:** `ai_outputParser` → Coordination Agent
- **Version:** typeVersion **1.3**
- **Potential failures / edge cases:**
  - If the agent includes non-string objects in arrays, parsing may fail.

#### Node: Slack Notification Tool
- **Type / role:** `slackTool` — tool node callable by the agent to send Slack messages.
- **Configuration (interpreted):**
  - Channel selection: by ID from AI: `={{ $fromAI('slackChannel', ..., 'string') }}`
  - Message text from AI: `={{ $fromAI('message', ..., 'string') }}`
  - OAuth2 authentication
- **Credentials:** Slack OAuth2 (`Slack account`)
- **Connections:** Connected as `ai_tool` into **Coordination Agent**
- **Version:** typeVersion **2.4**
- **Potential failures / edge cases:**
  - AI must supply valid `slackChannel` and `message`; otherwise expression/tool call fails.
  - Slack scopes missing (e.g., `chat:write`), channel not found, bot not in channel.

#### Node: Gmail Approval Tool
- **Type / role:** `gmailTool` — tool node callable by the agent to send approval requests.
- **Configuration (interpreted):**
  - To: `={{ $fromAI('recipients', ..., 'string') }}`
  - Subject: `={{ $fromAI('subject', ..., 'string') }}`
  - Body: `={{ $fromAI('emailBody', ..., 'string') }}`
- **Credentials:** Gmail OAuth2 (`Gmail account`)
- **Connections:** Connected as `ai_tool` into **Coordination Agent**
- **Version:** typeVersion **2.2**
- **Potential failures / edge cases:**
  - AI must provide properly formatted recipient string (comma-separated).
  - Gmail sending limits, OAuth scopes, or restricted org policies.

#### Node: Impact Assessment Agent Tool
- **Type / role:** `agentTool` — a secondary AI agent exposed as a callable tool for impact analysis.
- **Configuration (interpreted):**
  - Tool input: `={{ $fromAI("riskContext", ..., "json") }}`
  - System message requires financial, operational, reputational, cascading effects, recovery time, mitigation costs, probability-weighted thinking.
  - Has structured output parsing enabled.
- **Connections:**
  - Exposed to **Coordination Agent** as `ai_tool`
  - Uses **Anthropic Model - Impact Agent** as `ai_languageModel`
  - Uses **Impact Assessment Output Parser** as `ai_outputParser`
- **Version:** typeVersion **3**
- **Potential failures / edge cases:**
  - If `riskContext` is not valid JSON, tool invocation fails.
  - Impact outputs may be speculative without real financial baselines; consider providing revenue/cost context.

#### Node: Anthropic Model - Impact Agent
- **Type / role:** `LM Chat Anthropic` — Claude model for impact tool.
- **Model:** `claude-sonnet-4-5-20250929`
- **Credentials:** Anthropic API
- **Connections:** `ai_languageModel` → Impact Assessment Agent Tool
- **Version:** typeVersion **1.3**

#### Node: Impact Assessment Output Parser
- **Type / role:** `Structured Output Parser` — validates impact results.
- **Schema (required):**
  - `financialImpact`: { estimatedRevenueLoss, estimatedCostIncrease, potentialPenalties } (numbers)
  - `operationalImpact`: { productionDelayDays, capacityReductionPercent, affectedCustomers } (numbers)
  - `recoveryTimeEstimate` (string)
  - `mitigationCost` (number)
  - `cascadingEffects` (string array)
- **Connections:** `ai_outputParser` → Impact Assessment Agent Tool
- **Version:** typeVersion **1.3**
- **Potential failures:** numeric typing/formatting mismatches.

---

### Block 2.7 — LOW Risk Notification (Slack)
**Overview:** For LOW risk, posts a formatted Slack message summarizing scores, factors, and reasoning.

**Nodes involved:**
- Low Risk Notification

#### Node: Low Risk Notification
- **Type / role:** `Slack` — standard Slack node (not an agent tool) to send a message.
- **Configuration (interpreted):**
  - Channel ID: `={{ $('Workflow Configuration').first().json.slackChannelId }}`
  - Message includes:
    - overall/supplier/logistics/inventory scores from `$json.output.*`
    - bullet list from `$json.output.riskFactors`
    - reasoning text
- **Credentials:** Slack OAuth2 (`Slack account`)
- **Inputs / outputs:** Input from **Route by Risk Level** (LOW), output → **Merge Results** (index 1)
- **Version:** typeVersion **2.4**
- **Potential failures / edge cases:**
  - If `riskFactors` is not an array, `.map(...)` fails and node errors.
  - Long messages may hit Slack formatting/length constraints.

---

### Block 2.8 — Results Merge & Logging (Google Sheets)
**Overview:** Combines outputs from orchestration/notification paths and writes an audit record into a log sheet.

**Nodes involved:**
- Merge Results
- Log Risk Assessment

#### Node: Merge Results
- **Type / role:** `Merge` — combines streams for unified logging.
- **Configuration:** Mode = **combine**, combineBy = **position**.
- **Inputs / outputs:**
  - Input 1: from **Coordination Agent** (index 0)
  - Input 2: from **Low Risk Notification** (index 1)
  - Output → **Log Risk Assessment**
- **Version:** typeVersion **3.2**
- **Potential failures / edge cases (important):**
  - **Hanging executions:** In combine-by-position, the Merge node typically waits for both inputs. In this workflow, only one branch runs (LOW vs MEDIUM/CRITICAL), so the other input may never arrive, causing Merge to wait indefinitely or until timeout (behavior depends on n8n version/settings).
  - If you intend “log whichever path ran”, use Merge mode **Append** or implement separate logging per branch.

**Sticky note (contextual):**
- **“Results Management”** (Sticky Note6): explains logging/audit intent.

#### Node: Log Risk Assessment
- **Type / role:** `Google Sheets` — appends or updates a log row.
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Document ID: `={{ $('Workflow Configuration').first().json.logSheetId }}`
  - Sheet: **“Risk Assessment Log”**
  - Matching column: **timestamp**
  - Mapping mode: auto-map input data
- **Credentials:** Google Sheets OAuth2
- **Inputs / outputs:** Input from **Merge Results**. No further outputs.
- **Version:** typeVersion **4.7**
- **Potential failures / edge cases:**
  - `timestamp` must exist at the top level of the incoming item and match the sheet column name exactly; otherwise updates won’t match and append behavior may be inconsistent.
  - If the merged data nests the timestamp under `riskData.timestamp`, auto-mapping may not fill the correct column.
  - Permissions/sheet tab name mismatches.

---

### Block 2.9 — Documentation/Annotation Nodes (Sticky Notes)
**Overview:** These nodes do not execute logic but document intent and setup requirements. They should be preserved for maintainability.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note6
- Sticky Note5

**Potential issues:** None at runtime (annotations only).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Hourly workflow trigger | — | Workflow Configuration | ## How It Works … (3-tier routing, AI agents, logging) |
| Workflow Configuration | n8n-nodes-base.set | Central config variables (IDs, thresholds) | Schedule Trigger | Fetch Risk Data | ## Setup Steps … connect Sheets/Anthropic/Gmail/Slack, customize thresholds |
| Fetch Risk Data | n8n-nodes-base.googleSheets | Read risk indicators from Google Sheets | Workflow Configuration | Aggregate Risk Indicators | ## Risk Indicator Calculation … enables intelligent routing |
| Aggregate Risk Indicators | n8n-nodes-base.code | Aggregate rows, compute metrics and overall score | Fetch Risk Data | Risk Signal Agent | ## Risk Indicator Calculation … enables intelligent routing |
| Risk Signal Agent | @n8n/n8n-nodes-langchain.agent | Claude-based risk scoring + structured JSON | Aggregate Risk Indicators | Route by Risk Level | ## How It Works … 3-tier routing and escalation |
| Anthropic Model - Risk Agent | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backing for Risk Signal Agent | — (ai_languageModel link) | Risk Signal Agent (as model) | ## Prerequisites … Active accounts: Google Sheets, Anthropic Claude API, Gmail, Slack. |
| Risk Assessment Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured risk assessment schema | — (ai_outputParser link) | Risk Signal Agent (as parser) | ## Setup Steps … mentions parser output format alignment |
| Route by Risk Level | n8n-nodes-base.switch | Branch LOW/MEDIUM/CRITICAL | Risk Signal Agent | Coordination Agent; Low Risk Notification | ## How It Works … severity tiers and routing |
| Coordination Agent | @n8n/n8n-nodes-langchain.agent | Orchestrate contingencies using tools | Route by Risk Level | Merge Results | ## Critical Risk Processing … high-stakes multi-agent evaluation |
| Anthropic Model - Coordination Agent | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backing for Coordination Agent | — (ai_languageModel link) | Coordination Agent (as model) | ## Medium Risk Evaluation … single-agent structured parsing (note: MEDIUM also uses Coordination Agent) |
| Coordination Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured coordination results schema | — (ai_outputParser link) | Coordination Agent (as parser) | ## Results Management … accountability + stakeholder comms |
| Slack Notification Tool | n8n-nodes-base.slackTool | Agent tool: send Slack alerts | — (ai_tool link) | Coordination Agent (tool callable) | ## Critical Risk Processing … coordinated response tooling |
| Gmail Approval Tool | n8n-nodes-base.gmailTool | Agent tool: request approvals by email | — (ai_tool link) | Coordination Agent (tool callable) | ## Critical Risk Processing … approvals for critical actions |
| Impact Assessment Agent Tool | @n8n/n8n-nodes-langchain.agentTool | Agent tool: compute quantified impacts | — (ai_tool link) | Coordination Agent (tool callable) | ## Critical Risk Processing … deeper assessment via specialist tool |
| Anthropic Model - Impact Agent | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backing for impact tool | — (ai_languageModel link) | Impact Assessment Agent Tool (as model) | ## Results Management … supports transparent reporting |
| Impact Assessment Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured impact schema | — (ai_outputParser link) | Impact Assessment Agent Tool (as parser) | ## Results Management … quantified metrics for audit |
| Low Risk Notification | n8n-nodes-base.slack | Direct Slack message for LOW risk | Route by Risk Level | Merge Results | ## How It Works … low risks receive automated acknowledgment |
| Merge Results | n8n-nodes-base.merge | Combine branch outputs for logging | Coordination Agent; Low Risk Notification | Log Risk Assessment | ## Results Management … merged logging for accountability |
| Log Risk Assessment | n8n-nodes-base.googleSheets | Append/update audit log in Sheets | Merge Results | — | ## Results Management … audit documentation |
| Sticky Note | n8n-nodes-base.stickyNote | Annotation (prereqs/use cases) | — | — | ## Prerequisites … benefits and customization |
| Sticky Note1 | n8n-nodes-base.stickyNote | Annotation (setup steps) | — | — | ## Setup Steps … connections and thresholds |
| Sticky Note2 | n8n-nodes-base.stickyNote | Annotation (how it works) | — | — | ## How It Works … tiered routing and escalation |
| Sticky Note3 | n8n-nodes-base.stickyNote | Annotation (critical processing) | — | — | ## Critical Risk Processing … rationale |
| Sticky Note4 | n8n-nodes-base.stickyNote | Annotation (risk indicator calc) | — | — | ## Risk Indicator Calculation … rationale |
| Sticky Note5 | n8n-nodes-base.stickyNote | Annotation (medium evaluation) | — | — | ## Medium Risk Evaluation … rationale |
| Sticky Note6 | n8n-nodes-base.stickyNote | Annotation (results management) | — | — | ## Results Management … rationale |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name it: *AI-Powered Supply Chain Risk Evaluation and Contingency Orchestration*
   - Keep it inactive until credentials and IDs are configured.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Interval: every **1 hour**
   - Connect → **Workflow Configuration**

3) **Add Workflow Configuration (Set)**
   - Node: **Set**
   - Enable **Include Other Fields**
   - Add fields:
     - `riskDataSheetId` (string) = your risk indicators Google Sheet ID
     - `logSheetId` (string) = your logging Google Sheet ID
     - `slackChannelId` (string) = Slack channel ID
     - `approvalEmailRecipients` (string) = comma-separated recipients
     - `criticalRiskThreshold` (number) = 80
     - `mediumRiskThreshold` (number) = 50
   - Connect → **Fetch Risk Data**

4) **Add Fetch Risk Data (Google Sheets)**
   - Node: **Google Sheets**
   - Credentials: create/select **Google Sheets OAuth2**
   - Document ID: expression `{{ $('Workflow Configuration').first().json.riskDataSheetId }}`
   - Sheet name: `Risk Indicators`
   - Operation: read rows (default “Get Many” / “Read” equivalent)
   - Connect → **Aggregate Risk Indicators**

5) **Add Aggregate Risk Indicators (Code)**
   - Node: **Code**
   - Paste the aggregation logic (as in workflow) that returns:
     - a single item with `{ riskData: aggregatedRisk }` including `riskData.timestamp`
   - Connect → **Risk Signal Agent**

6) **Add Risk Signal Agent (LangChain Agent)**
   - Node: **AI Agent** (LangChain Agent)
   - Prompt (user text): `Risk Indicators Data: {{ JSON.stringify($json.riskData) }}`
   - System message: include the requirements (category scoring 0–100, overall level LOW/MEDIUM/CRITICAL, contractual boundaries, JSON output)
   - Enable **Output Parser**
   - Connect main output → **Route by Risk Level**

7) **Add Anthropic Model for Risk Agent**
   - Node: **Anthropic Chat Model**
   - Credentials: **Anthropic API key**
   - Model: `claude-sonnet-4-5-20250929` (or your preferred Claude Sonnet version)
   - Connect this node to **Risk Signal Agent** using the **ai_languageModel** connection.

8) **Add Risk Assessment Output Parser**
   - Node: **Structured Output Parser**
   - Schema type: Manual
   - Paste schema requiring: overallRiskLevel, overallRiskScore, supplierRiskScore, logisticsRiskScore, inventoryRiskScore, riskFactors[], contractualViolations[], reasoning
   - Connect to **Risk Signal Agent** via **ai_outputParser**.

9) **Add Route by Risk Level (Switch)**
   - Node: **Switch**
   - Create 3 rules on string equals:
     - `{{ $json.output.overallRiskLevel }}` equals `CRITICAL`
     - equals `MEDIUM`
     - equals `LOW`
   - Rename outputs to: “Critical Risk”, “Medium Risk”, “Low Risk”
   - (Optional) configure fallback output “Unclassified”
   - Connect:
     - Critical Risk → **Coordination Agent**
     - Medium Risk → **Coordination Agent**
     - Low Risk → **Low Risk Notification**

10) **Add Coordination Agent (LangChain Agent)**
   - Node: **AI Agent** (LangChain Agent)
   - Prompt:  
     - `Risk Assessment: {{ JSON.stringify($json.output) }}`  
     - `Risk Data: {{ JSON.stringify($json.riskData) }}`
   - System message: instruct it to pick contingencies, call tools (Slack, Gmail, Impact), preserve contractual boundaries, request approvals before contract modifications, and return structured output.
   - Enable **Output Parser**
   - Connect main output → **Merge Results**

11) **Add Anthropic Model for Coordination Agent**
   - Node: **Anthropic Chat Model**
   - Model: same Claude Sonnet
   - Credentials: Anthropic
   - Connect to **Coordination Agent** via **ai_languageModel**.

12) **Add Coordination Output Parser**
   - Node: **Structured Output Parser**
   - Manual schema requiring arrays:
     - actionsTaken[], approvalsRequested[], notificationsSent[], contractualConstraints[], recommendedNextSteps[]
     - (optional) impactAssessmentSummary string
   - Connect to **Coordination Agent** via **ai_outputParser**.

13) **Add Slack Notification Tool (Agent Tool Node)**
   - Node: **Slack Tool**
   - Credentials: **Slack OAuth2** (scopes to post messages)
   - Configure:
     - Channel ID: `{{ $fromAI('slackChannel', 'Slack channel ID for notification', 'string') }}`
     - Text: `{{ $fromAI('message', 'Notification message content', 'string') }}`
   - Connect Slack Tool to **Coordination Agent** via **ai_tool**.

14) **Add Gmail Approval Tool (Agent Tool Node)**
   - Node: **Gmail Tool**
   - Credentials: **Gmail OAuth2**
   - Configure:
     - To: `{{ $fromAI('recipients', 'Email recipients for approval request', 'string') }}`
     - Subject: `{{ $fromAI('subject', 'Email subject line', 'string') }}`
     - Body: `{{ $fromAI('emailBody', 'Email body content', 'string') }}`
   - Connect Gmail Tool to **Coordination Agent** via **ai_tool**.

15) **Add Impact Assessment Agent Tool**
   - Node: **Agent Tool** (LangChain Agent Tool)
   - Tool input: `{{ $fromAI('riskContext', 'Risk assessment context and data for impact analysis', 'json') }}`
   - System message: instruct detailed financial/operational/reputational impact and return structured metrics.
   - Enable **Output Parser**
   - Connect this tool to **Coordination Agent** via **ai_tool**.

16) **Add Anthropic Model for Impact Tool**
   - Node: **Anthropic Chat Model**
   - Model: Claude Sonnet
   - Credentials: Anthropic
   - Connect to **Impact Assessment Agent Tool** via **ai_languageModel**.

17) **Add Impact Assessment Output Parser**
   - Node: **Structured Output Parser**
   - Manual schema requiring:
     - financialImpact.{estimatedRevenueLoss, estimatedCostIncrease, potentialPenalties}
     - operationalImpact.{productionDelayDays, capacityReductionPercent, affectedCustomers}
     - recoveryTimeEstimate, mitigationCost, cascadingEffects[]
   - Connect to **Impact Assessment Agent Tool** via **ai_outputParser**.

18) **Add Low Risk Notification (Slack)**
   - Node: **Slack**
   - Credentials: Slack OAuth2
   - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannelId }}`
   - Text: compose message using `$json.output.*`, including `riskFactors.map(...)`
   - Connect → **Merge Results** (second input)

19) **Add Merge Results**
   - Node: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Input 1: from **Coordination Agent**
   - Input 2: from **Low Risk Notification**
   - Output → **Log Risk Assessment**
   - Important: if you want reliable logging, consider replacing this with:
     - Merge mode **Append**, or
     - Two separate logging nodes (one per branch).

20) **Add Log Risk Assessment (Google Sheets)**
   - Node: **Google Sheets**
   - Credentials: Google Sheets OAuth2
   - Document ID: `{{ $('Workflow Configuration').first().json.logSheetId }}`
   - Sheet: `Risk Assessment Log`
   - Operation: **Append or Update**
   - Matching column: `timestamp`
   - Ensure the incoming data contains a **top-level** `timestamp` field aligned with the sheet column.

21) **Add Sticky Notes (optional but recommended)**
   - Add notes for prerequisites, setup steps, risk indicator calculation, medium/critical handling, results management.

22) **Validate and activate**
   - Run manually once with sample sheet data.
   - Confirm:
     - Risk agent returns valid structured output.
     - Switch routes correctly.
     - Slack/Gmail tool calls succeed.
     - Logging writes correct columns.
   - Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Active accounts required: Google Sheets, Anthropic Claude API, Gmail, Slack. | From “Prerequisites” sticky note |
| Use cases: enterprise compliance monitoring, operational risk management. | From “Prerequisites” sticky note |
| Customization: modify scoring formulas, adjust severity thresholds, add custom AI criteria. | From “Prerequisites” sticky note |
| Benefits: eliminates manual triage, ensures consistent standards, accelerates critical response. | From “Prerequisites” sticky note |
| Setup steps include connecting credentials and specifying channel IDs; note mentions “parser regex patterns”, but this workflow uses structured output parsers instead. | From “Setup Steps” sticky note |
| Design intent: 3-tier routing (critical/medium/low) with audit documentation. | From “How It Works” sticky note |
| Critical risks intended to receive comprehensive multi-agent evaluation. | From “Critical Risk Processing” sticky note |
| Results management intended to support accountability via merged logging and stakeholder communications. | From “Results Management” sticky note |

