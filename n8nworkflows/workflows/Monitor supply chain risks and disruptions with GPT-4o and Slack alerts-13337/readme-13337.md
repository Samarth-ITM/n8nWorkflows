Monitor supply chain risks and disruptions with GPT-4o and Slack alerts

https://n8nworkflows.xyz/workflows/monitor-supply-chain-risks-and-disruptions-with-gpt-4o-and-slack-alerts-13337


# Monitor supply chain risks and disruptions with GPT-4o and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Monitor supply chain risks and disruptions with GPT-4o and Slack alerts  
**Workflow name (in JSON):** AI-powered supply chain monitoring and disruption management system

**Purpose:**  
This workflow periodically pulls procurement + warehouse/transportation data, uses two GPT-4o-based agents to (1) detect supply-chain risk signals and (2) coordinate response actions, then routes actions to Slack/email and logs an audit trail. A separate webhook entry point accepts manual approval decisions and records them.

**Target use cases:**  
- Proactive anomaly detection (delays, inventory variance, transport disruptions, compliance issues)  
- Risk scoring and severity routing (CRITICAL/HIGH/MEDIUM/LOW)  
- Operational alerting via Slack, escalation via email, and audit/traceability logging  
- Manual approval capture for governance

### Logical Blocks
1.1 **Scheduled data collection & configuration**  
1.2 **External data fetching & consolidation**  
1.3 **AI signal monitoring (risk detection + structured output)**  
1.4 **Risk-based routing to response coordination**  
1.5 **AI coordination (action plan + tool use) & action routing**  
1.6 **Execution: Slack/email + compliance audit logging + final report**  
1.7 **Manual approval intake (separate entry point) + audit logging**

---

## 2. Block-by-Block Analysis

### 1.1 Scheduled data collection & configuration
**Overview:** Starts the monitoring cycle every 15 minutes and centralizes runtime configuration (API endpoints, alert channel, thresholds, escalation email).

**Nodes involved:**
- Schedule Supply Chain Data Collection
- Workflow Configuration

#### Node: Schedule Supply Chain Data Collection
- **Type / role:** Schedule Trigger; initiates workflow on a timer.
- **Configuration:** Runs every **15 minutes**.
- **Connections:**  
  - Output → **Workflow Configuration**
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**  
  - None typical beyond instance scheduling/activation status (workflow is currently `active: false` so it will not run until enabled).

#### Node: Workflow Configuration
- **Type / role:** Set node; defines config variables used by downstream nodes.
- **Configuration choices (interpreted):**
  - Sets:
    - `procurementApiUrl` (placeholder)
    - `warehouseApiUrl` (placeholder)
    - `slackChannelId` (placeholder)
    - `escalationEmail` (placeholder)
    - `criticalThreshold` = 80
    - `highThreshold` = 60
    - `mediumThreshold` = 40
  - **Includes other fields** = true (passes through any incoming JSON).
- **Key expressions/variables:** Values are constants/placeholders; later nodes reference them via `$('Workflow Configuration').first().json...`.
- **Connections:**  
  - Output → **Fetch Procurement Data**  
  - Output → **Fetch Warehouse and Transportation Data**
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**  
  - Placeholders must be replaced or HTTP nodes will fail due to invalid URL.  
  - Thresholds are defined but **not actually used** in any routing logic; risk routing relies on the AI’s `riskLevel` string.

---

### 1.2 External data fetching & consolidation
**Overview:** Pulls procurement and warehouse/transportation datasets from two HTTP endpoints and merges them into a single item for analysis.

**Nodes involved:**
- Fetch Procurement Data
- Fetch Warehouse and Transportation Data
- Combine Supply Chain Data

#### Node: Fetch Procurement Data
- **Type / role:** HTTP Request; pulls procurement/supplier/order signals.
- **Configuration:**  
  - URL: `={{ $('Workflow Configuration').first().json.procurementApiUrl }}`
  - Options: default (no headers/auth defined in node)
- **Connections:**  
  - Input ← Workflow Configuration  
  - Output → Combine Supply Chain Data (input 0)
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:**  
  - Missing auth headers/credentials (if endpoint requires them).  
  - Non-JSON responses may break downstream assumptions (agent prompt stringifies data).  
  - Timeouts / 4xx/5xx responses will stop execution unless “Continue On Fail” is configured (not set here).

#### Node: Fetch Warehouse and Transportation Data
- **Type / role:** HTTP Request; pulls WMS/TMS signals (inventory, shipment tracking).
- **Configuration:**  
  - URL: `={{ $('Workflow Configuration').first().json.warehouseApiUrl }}`
- **Connections:**  
  - Input ← Workflow Configuration  
  - Output → Combine Supply Chain Data (input 1)
- **Version notes:** typeVersion **4.3**
- **Edge cases / failures:** same as above.

#### Node: Combine Supply Chain Data
- **Type / role:** Merge; combines both HTTP results into one item.
- **Configuration:**  
  - Mode: **combine**  
  - Combine by: **position** (pairs item 0 from procurement with item 0 from warehouse/TMS)
- **Connections:**  
  - Input 0 ← Fetch Procurement Data  
  - Input 1 ← Fetch Warehouse and Transportation Data  
  - Output → Signal Monitoring Agent
- **Version notes:** typeVersion **3.2**
- **Edge cases / failures:**  
  - If one HTTP request returns a different number of items, “combine by position” may produce missing/partial pairings.  
  - Output field names are not explicitly mapped; downstream agent prompt expects `$json.procurementData` and `$json.warehouseData`, but the Merge node will typically output a merged JSON structure based on incoming fields. If the HTTP responses do not already nest under those names, the agent prompt may stringify `undefined`.

---

### 1.3 AI signal monitoring (risk detection + structured output)
**Overview:** GPT-4o analyzes combined supply chain data, detects anomalies, assigns a risk score/level, and returns structured JSON via a schema-based parser.

**Nodes involved:**
- OpenAI Model for Signal Monitoring Agent
- Signal Analysis Output Parser
- Signal Monitoring Agent

#### Node: OpenAI Model for Signal Monitoring Agent
- **Type / role:** LangChain Chat Model (OpenAI); provides LLM for the agent.
- **Configuration:**  
  - Model: **gpt-4o**  
  - Temperature: **0.1** (low randomness for consistency)
- **Credentials:** OpenAI API credential “OpenAi account”
- **Connections:**  
  - Output (ai_languageModel) → Signal Monitoring Agent
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:** API key invalid, quota exceeded, model access restrictions, transient OpenAI errors.

#### Node: Signal Analysis Output Parser
- **Type / role:** Structured Output Parser; enforces JSON schema/shape.
- **Configuration:** Provides an example schema with:
  - `riskLevel`, `riskScore`, `anomalies[]`, `supplyChainMetrics`, `recommendations[]`, `requiresEscalation`, `reasoning`
- **Connections:**  
  - Output (ai_outputParser) → Signal Monitoring Agent
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**  
  - Model may output invalid JSON or deviate from schema → parser failure.  
  - Large payloads could increase token usage and cost.

#### Node: Signal Monitoring Agent
- **Type / role:** LangChain Agent; prompts model to analyze and produce structured result.
- **Configuration choices (interpreted):**
  - Prompt includes:
    - `Procurement Data: {{ JSON.stringify($json.procurementData) }}`
    - `Warehouse and Transportation Data: {{ JSON.stringify($json.warehouseData) }}`
  - System message defines detection criteria and mandates exact JSON output format.
  - `hasOutputParser: true` (connected to Signal Analysis Output Parser)
- **Connections:**  
  - Input ← Combine Supply Chain Data  
  - Model ← OpenAI Model for Signal Monitoring Agent  
  - Output parser ← Signal Analysis Output Parser  
  - Output (main) → Route by Risk Level
- **Version notes:** typeVersion **3.1**
- **Edge cases / failures:**  
  - As noted, `procurementData` / `warehouseData` may not exist depending on HTTP response structure; consider mapping/renaming fields before the agent.  
  - Prompt may exceed context window if API returns very large datasets; typically needs paging/summarization.

---

### 1.4 Risk-based routing to response coordination
**Overview:** Routes the AI risk assessment into the coordination stage based on `riskLevel`.

**Nodes involved:**
- Route by Risk Level

#### Node: Route by Risk Level
- **Type / role:** Switch; conditional branching.
- **Configuration:**
  - If `={{ $json.output.riskLevel }}` equals:
    - `CRITICAL` → output “CRITICAL”
    - `HIGH` → output “HIGH”
    - `MEDIUM` → output “MEDIUM_LOW”
  - Fallback output: `extra`
- **Connections:**  
  - Input ← Signal Monitoring Agent  
  - Output (all configured branches) → Coordination Agent (single connection in JSON)
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**  
  - **LOW is not explicitly handled**; LOW will go to fallback (`extra`). In this workflow, fallback is still connected onward (because Switch output is connected), but you lose explicit labeling and any branch-specific handling.  
  - Routing depends on the AI returning exact strings; any typo/case mismatch will also hit fallback.

---

### 1.5 AI coordination (action plan + tool use) & action routing
**Overview:** A second GPT-4o agent turns the risk assessment into an actionable response plan (alerts, escalations, approvals, logging) and can use a Slack “tool” for real-time messaging.

**Nodes involved:**
- OpenAI Model for Coordination Agent
- Coordination Output Parser
- Coordination Agent
- Slack Alert Tool
- Route by Action Type

#### Node: OpenAI Model for Coordination Agent
- **Type / role:** LangChain Chat Model (OpenAI) for the coordination agent.
- **Configuration:**  
  - Model: **gpt-4o**  
  - Temperature: **0.2**
- **Credentials:** OpenAI API credential “OpenAi account”
- **Connections:**  
  - Output (ai_languageModel) → Coordination Agent
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:** same OpenAI failure modes as above.

#### Node: Coordination Output Parser
- **Type / role:** Structured Output Parser.
- **Configuration:** Schema example includes:
  - `actionType` (ALERT/ESCALATE/APPROVE/LOG)
  - `priority`
  - `workflowActions[]`
  - `replenishmentPlan`
  - `complianceStatus`
  - `auditTrail` (timestamp, analysisId, approvalRequired, approvalReason)
  - `reasoning`
- **Connections:**  
  - Output (ai_outputParser) → Coordination Agent
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:** parser failure if model deviates from schema.

#### Node: Slack Alert Tool
- **Type / role:** Slack Tool node (AI tool); callable by the Coordination Agent during reasoning.
- **Configuration:**
  - Sends message text from: `{{$fromAI('alertMessage', ... )}}`
  - Channel ID from: `{{$fromAI('slackChannel', ... )}}`
  - Auth: OAuth2 Slack
- **Credentials:** “Slack account” (OAuth2)
- **Connections:**  
  - Connected as `ai_tool` to **Coordination Agent**
- **Version notes:** typeVersion **2.4**
- **Edge cases / failures:**  
  - The agent must actually provide `alertMessage` and `slackChannel` tool arguments; otherwise tool execution fails.  
  - Slack OAuth scope issues (e.g., missing `chat:write`).  
  - Channel ID must be valid and accessible by the bot/user.

#### Node: Coordination Agent
- **Type / role:** LangChain Agent; creates response plan and may invoke Slack tool.
- **Configuration:**
  - Prompt includes the prior agent’s output: `Risk Analysis: {{ JSON.stringify($json.output) }}`
  - System routing logic described:
    - CRITICAL: Slack + Email + Request approval
    - HIGH: Slack + Email
    - MEDIUM: Slack + Log audit
    - LOW: Log audit only
  - `hasOutputParser: true` (Coordination Output Parser)
- **Connections:**  
  - Input ← Route by Risk Level  
  - Model ← OpenAI Model for Coordination Agent  
  - Tool ← Slack Alert Tool  
  - Output parser ← Coordination Output Parser  
  - Output (main) → Route by Action Type
- **Version notes:** typeVersion **3.1**
- **Edge cases / failures:**  
  - The described action routing (including approvals/log-only) is **not fully implemented** in downstream node connections (see next blocks).  
  - Tool invocation is optional; yet separate Slack nodes also exist later, creating potential duplication/inconsistent messaging.

#### Node: Route by Action Type
- **Type / role:** Switch; branches by `actionType`.
- **Configuration:** If `={{ $json.output.actionType }}` equals:
  - `ALERT` → output “ALERT”
  - `ESCALATE` → output “ESCALATE”
  - `APPROVE` → output “APPROVE”
  - Fallback: `extra`
- **Connections (as wired):**
  - Output 0 → Send Critical Alert to Slack
  - Output 1 → Send Escalation Email
  - **No connection for APPROVE branch** (not wired)
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**  
  - `APPROVE` results are generated by the Coordination Agent per its rules, but this branch is **not connected** to an approval flow; such executions will effectively stop at the switch output unless fallback or another connection exists.  
  - `LOG` actionType is not explicitly routed; it will go to fallback (`extra`) and is not connected either.

---

### 1.6 Execution: Slack/email + compliance audit logging + final report
**Overview:** Sends Slack alerts and/or escalation email, then logs an audit entry and produces a final report object.

**Nodes involved:**
- Send Critical Alert to Slack
- Send Escalation Email
- Log Compliance Audit Trail
- Prepare Final Report

#### Node: Send Critical Alert to Slack
- **Type / role:** Slack node; posts a formatted message to a channel.
- **Configuration:**
  - Channel ID: `={{ $('Workflow Configuration').first().json.slackChannelId }}`
  - Message uses fields from `{{$json.output...}}` such as `priority`, `actionType`, `workflowActions[]`, `replenishmentPlan`, `complianceStatus`.
  - Auth: OAuth2 Slack
- **Connections:**  
  - Input ← Route by Action Type (ALERT branch as currently wired)  
  - Output → Log Compliance Audit Trail
- **Version notes:** typeVersion **2.4**
- **Edge cases / failures:**  
  - If `replenishmentPlan` or arrays are missing, expressions like `.map(...)` may throw.  
  - Slack rate limits or missing scopes.

#### Node: Send Escalation Email
- **Type / role:** Email Send; escalates to management email.
- **Configuration:**
  - To: `={{ $('Workflow Configuration').first().json.escalationEmail }}`
  - From: placeholder sender email must be replaced
  - Subject and HTML body built from `{{$json.output...}}` (priority, actions, audit trail, reasoning).
- **Connections:**  
  - Input ← Route by Action Type (ESCALATE branch as currently wired)  
  - Output → Log Compliance Audit Trail
- **Version notes:** typeVersion **2.1**
- **Edge cases / failures:**  
  - Requires email transport credentials/config at instance level (SMTP or provider).  
  - Missing/invalid `fromEmail` causes send failures.  
  - HTML expressions may fail if nested fields are absent.

#### Node: Log Compliance Audit Trail
- **Type / role:** Code node; creates standardized audit entries for traceability.
- **Configuration (logic):**
  - Generates:
    - `auditId` random + timestamp based
    - `eventType` from `output.actionType` or `decision`
    - `priority`, `complianceStatus`
    - `actionsTaken` from `workflowActions[].action` or approval status
    - `approvalRequired`, `approvalStatus`, `approver`
    - `reasoning`
    - `traceabilityId` from `auditTrail.analysisId` or generated
    - `metadata` including `$workflow.id` and `$execution.id`
- **Connections:**  
  - Input ← Send Critical Alert to Slack OR Send Escalation Email OR Process Approval Decision  
  - Output → Prepare Final Report
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**  
  - Uses optional chaining for many fields, reducing crashes; still assumes `$workflow` and `$execution` exist (they do in n8n).  
  - `nodeExecutionId` is set to `$execution.id` (not truly a node execution id).

#### Node: Prepare Final Report
- **Type / role:** Set node; summarizes the run.
- **Configuration:** Creates:
  - `reportId` (timestamp-based)
  - `reportTimestamp`
  - `totalAuditEntries` = `$input.all().length`
  - `executionSummary`
  - `workflowStatus` = `COMPLETED`
  - Includes other fields
- **Connections:**  
  - Input ← Log Compliance Audit Trail  
  - Output → none (end)
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**  
  - If no upstream items reach audit logging, this node is never reached.

---

### 1.7 Manual approval intake (separate entry point) + audit logging
**Overview:** A webhook endpoint receives approval decisions, normalizes them, responds to the caller, and logs an audit entry.

**Nodes involved:**
- Manual Approval Webhook
- Process Approval Decision
- Respond to Approval
- Log Compliance Audit Trail (shared with main flow)

#### Node: Manual Approval Webhook
- **Type / role:** Webhook trigger; separate workflow entry point.
- **Configuration:**  
  - Path: `POST /supply-chain-approval`
  - Response mode: `lastNode`
- **Connections:**  
  - Output → Process Approval Decision
- **Version notes:** typeVersion **2.1**
- **Edge cases / failures:**  
  - No authentication/verification is implemented; endpoint may be abused unless protected by n8n auth, reverse proxy, or custom token checks.
  - Response depends on downstream “Respond to Webhook”.

#### Node: Process Approval Decision
- **Type / role:** Code node; standardizes approval payload.
- **Configuration (logic):**
  - Reads `decision`, `approver`, `comments`
  - Adds `timestamp`, `approvalId` like `APR-<epoch>`
  - Derives `status` = APPROVED/REJECTED/PENDING
- **Connections:**  
  - Input ← Manual Approval Webhook  
  - Output → Respond to Approval  
  - Output → Log Compliance Audit Trail
- **Version notes:** typeVersion **2**
- **Edge cases / failures:**  
  - Accepts any decision string; only `approved`/`rejected` map cleanly, otherwise PENDING.

#### Node: Respond to Approval
- **Type / role:** Respond to Webhook; returns JSON confirmation.
- **Configuration:** Responds with:
  - status=success, message, echoes `decision`, and server timestamp.
- **Connections:**  
  - Input ← Process Approval Decision  
  - Output → none
- **Version notes:** typeVersion **1.5**
- **Edge cases / failures:**  
  - If execution errors before reaching this node, webhook caller may get an error/timeout.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Supply Chain Data Collection | scheduleTrigger | Periodic trigger (15 min) | — | Workflow Configuration | ## Automated Data Aggregation\n**What**: Scheduled trigger fetches procurement orders, warehouse inventory, and transportation tracking data  \n**Why**: Unified data collection eliminates siloed visibility and enables holistic supply chain optimization |
| Workflow Configuration | set | Central config variables/placeholders | Schedule Supply Chain Data Collection | Fetch Procurement Data; Fetch Warehouse and Transportation Data | ## Automated Data Aggregation\n**What**: Scheduled trigger fetches procurement orders, warehouse inventory, and transportation tracking data  \n**Why**: Unified data collection eliminates siloed visibility and enables holistic supply chain optimization |
| Fetch Procurement Data | httpRequest | Pull procurement data from API | Workflow Configuration | Combine Supply Chain Data | ## Automated Data Aggregation\n**What**: Scheduled trigger fetches procurement orders, warehouse inventory, and transportation tracking data  \n**Why**: Unified data collection eliminates siloed visibility and enables holistic supply chain optimization |
| Fetch Warehouse and Transportation Data | httpRequest | Pull WMS/TMS data from API | Workflow Configuration | Combine Supply Chain Data | ## Automated Data Aggregation\n**What**: Scheduled trigger fetches procurement orders, warehouse inventory, and transportation tracking data  \n**Why**: Unified data collection eliminates siloed visibility and enables holistic supply chain optimization |
| Combine Supply Chain Data | merge | Merge both datasets (by position) | Fetch Procurement Data; Fetch Warehouse and Transportation Data | Signal Monitoring Agent | ## Dual-Agent Intelligence\n**What**: Signal Monitoring Agent detects anomalies and trends; Coordination Agent optimizes routing and inventory decisions  \n**Why**: Parallel analysis identifies both immediate risks and strategic optimization opportunities simultaneously |
| OpenAI Model for Signal Monitoring Agent | lmChatOpenAi | LLM backend (gpt-4o) for signal analysis | — | Signal Monitoring Agent | ## Dual-Agent Intelligence\n**What**: Signal Monitoring Agent detects anomalies and trends; Coordination Agent optimizes routing and inventory decisions  \n**Why**: Parallel analysis identifies both immediate risks and strategic optimization opportunities simultaneously |
| Signal Analysis Output Parser | outputParserStructured | Enforce structured JSON output for signal analysis | — | Signal Monitoring Agent | ## Dual-Agent Intelligence\n**What**: Signal Monitoring Agent detects anomalies and trends; Coordination Agent optimizes routing and inventory decisions  \n**Why**: Parallel analysis identifies both immediate risks and strategic optimization opportunities simultaneously |
| Signal Monitoring Agent | agent | Detect anomalies, risk score/level, recommendations | Combine Supply Chain Data | Route by Risk Level | ## Dual-Agent Intelligence\n**What**: Signal Monitoring Agent detects anomalies and trends; Coordination Agent optimizes routing and inventory decisions  \n**Why**: Parallel analysis identifies both immediate risks and strategic optimization opportunities simultaneously |
| Route by Risk Level | switch | Branch by riskLevel (CRITICAL/HIGH/MEDIUM; fallback) | Signal Monitoring Agent | Coordination Agent | ## Risk-Based Routing\n**What**: Routes findings by severity—critical triggers immediate alerts and approvals, acceptable enables standard processing  \n**Why**: Priority workflows ensure urgent disruptions receive rapid response while maintaining operational efficiency |
| OpenAI Model for Coordination Agent | lmChatOpenAi | LLM backend (gpt-4o) for coordination decisions | — | Coordination Agent | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Coordination Output Parser | outputParserStructured | Enforce structured JSON output for coordination plan | — | Coordination Agent | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Slack Alert Tool | slackTool | AI-callable Slack send tool | Coordination Agent (tool connection) | Coordination Agent | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Coordination Agent | agent | Decide response actions; may call Slack tool | Route by Risk Level | Route by Action Type | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Route by Action Type | switch | Branch by actionType (ALERT/ESCALATE/APPROVE; fallback) | Coordination Agent | Send Critical Alert to Slack; Send Escalation Email | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Send Critical Alert to Slack | slack | Post formatted alert message | Route by Action Type | Log Compliance Audit Trail | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Send Escalation Email | emailSend | Send escalation email with details | Route by Action Type | Log Compliance Audit Trail | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Log Compliance Audit Trail | code | Create audit entries for traceability | Send Critical Alert to Slack; Send Escalation Email; Process Approval Decision | Prepare Final Report | ## Action-Specific Response\n**What**: Executes workflows by action type—critical sends multi-channel alerts with approval gates, routine generates reports  \n**Why**: Context-appropriate responses balance speed for emergencies with governance for strategic decisions |
| Prepare Final Report | set | Summarize run and counts | Log Compliance Audit Trail | — | ## How It Works\nThis workflow automates end-to-end supply chain visibility and logistics coordination for manufacturers, distributors, and retailers managing complex multi-tier supply networks. Designed for supply chain planners, logistics managers, and operations directors, it solves the challenge of tracking inventory across procurement, warehousing, and transportation while optimizing decisions for cost, speed, and risk mitigation. The system schedules regular data collection from procurement and warehouse/transportation systems, consolidates supply chain data, analyzes patterns through dual AI agents (Signal Monitoring identifies anomalies and trends, Coordination Agent orchestrates optimization decisions), routes findings by risk level (critical/marginal/acceptable), triggers action-specific responses (critical issues send Slack alerts, escalation emails, and compliance audit logs with approval workflows; acceptable conditions generate standard reports), and maintains complete traceability. Organizations achieve 50% reduction in stockouts, optimize logistics costs by 30%, enable proactive disruption management, and maintain real-time visibility across global supply networks. |
| Manual Approval Webhook | webhook | Receive approval decisions (POST endpoint) | — | Process Approval Decision | ## Setup Steps\n1. Connect **Schedule Trigger** for monitoring frequency \n2. Configure **procurement system APIs** with order and supplier data access credentials\n3. Link **warehouse management systems** (WMS) and **transportation platforms** (TMS) for inventory and shipment tracking\n4. Add **AI model API keys** to Signal Monitoring and Coordination Agent nodes\n5. Define **optimization parameters** in agent prompts\n6. Configure **Slack webhooks** for critical supply chain alerts to operations teams\n7. Set up **email credentials** for escalation workflows |
| Process Approval Decision | code | Normalize approval payload; generate approvalId/status | Manual Approval Webhook | Respond to Approval; Log Compliance Audit Trail | ## Setup Steps\n1. Connect **Schedule Trigger** for monitoring frequency \n2. Configure **procurement system APIs** with order and supplier data access credentials\n3. Link **warehouse management systems** (WMS) and **transportation platforms** (TMS) for inventory and shipment tracking\n4. Add **AI model API keys** to Signal Monitoring and Coordination Agent nodes\n5. Define **optimization parameters** in agent prompts\n6. Configure **Slack webhooks** for critical supply chain alerts to operations teams\n7. Set up **email credentials** for escalation workflows |
| Respond to Approval | respondToWebhook | Return JSON response to webhook caller | Process Approval Decision | — | ## Setup Steps\n1. Connect **Schedule Trigger** for monitoring frequency \n2. Configure **procurement system APIs** with order and supplier data access credentials\n3. Link **warehouse management systems** (WMS) and **transportation platforms** (TMS) for inventory and shipment tracking\n4. Add **AI model API keys** to Signal Monitoring and Coordination Agent nodes\n5. Define **optimization parameters** in agent prompts\n6. Configure **Slack webhooks** for critical supply chain alerts to operations teams\n7. Set up **email credentials** for escalation workflows |
| Sticky Note | stickyNote | Comment block | — | — | (self) |
| Sticky Note1 | stickyNote | Comment block | — | — | (self) |
| Sticky Note2 | stickyNote | Comment block | — | — | (self) |
| Sticky Note3 | stickyNote | Comment block | — | — | (self) |
| Sticky Note4 | stickyNote | Comment block | — | — | (self) |
| Sticky Note5 | stickyNote | Comment block | — | — | (self) |
| Sticky Note6 | stickyNote | Comment block | — | — | (self) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger: “Schedule Supply Chain Data Collection”**
- Add node: **Schedule Trigger**
- Set interval: every **15 minutes**
- Connect to next step.

2) **Create config node: “Workflow Configuration”**
- Add node: **Set**
- Add fields:
  - `procurementApiUrl` (string) → your procurement endpoint
  - `warehouseApiUrl` (string) → your WMS/TMS endpoint
  - `slackChannelId` (string) → Slack channel ID
  - `escalationEmail` (string) → escalation recipient
  - `criticalThreshold` (number) = 80
  - `highThreshold` (number) = 60
  - `mediumThreshold` (number) = 40
- Enable **Include Other Fields**
- Connect from Schedule Trigger → Workflow Configuration.

3) **Create HTTP nodes**
- Node A: **HTTP Request** named “Fetch Procurement Data”
  - URL: expression `$('Workflow Configuration').first().json.procurementApiUrl`
  - Configure authentication/headers as required by your API (e.g., Bearer token).
- Node B: **HTTP Request** named “Fetch Warehouse and Transportation Data”
  - URL: expression `$('Workflow Configuration').first().json.warehouseApiUrl`
  - Configure authentication/headers as required.
- Connect Workflow Configuration → both HTTP nodes.

4) **Merge the datasets**
- Add node: **Merge** named “Combine Supply Chain Data”
- Mode: **Combine**
- Combine by: **Position**
- Connect:
  - Fetch Procurement Data → Merge input 0
  - Fetch Warehouse and Transportation Data → Merge input 1

5) **Add Signal Monitoring AI stack**
- Add node: **OpenAI Chat Model** (`lmChatOpenAi`) named “OpenAI Model for Signal Monitoring Agent”
  - Model: `gpt-4o`
  - Temperature: `0.1`
  - Credentials: configure **OpenAI API** credential
- Add node: **Structured Output Parser** named “Signal Analysis Output Parser”
  - Paste/define the JSON schema example (riskLevel/riskScore/anomalies/metrics/etc.)
- Add node: **AI Agent** named “Signal Monitoring Agent”
  - Prompt text: analyze procurement + warehouse/TMS data (ensure your merged JSON actually has the referenced fields; add an intermediate Set node to map them if needed)
  - System message: include anomaly criteria and require exact JSON format
  - Enable output parser usage
- Wire:
  - Merge → Signal Monitoring Agent (main)
  - OpenAI Model → Signal Monitoring Agent (ai_languageModel)
  - Output Parser → Signal Monitoring Agent (ai_outputParser)

6) **Risk routing**
- Add node: **Switch** named “Route by Risk Level”
- Rules:
  - `$json.output.riskLevel == "CRITICAL"`
  - `$json.output.riskLevel == "HIGH"`
  - `$json.output.riskLevel == "MEDIUM"` (optionally add LOW explicitly)
- Connect Signal Monitoring Agent → Route by Risk Level.

7) **Add Coordination AI stack**
- Add node: **OpenAI Chat Model** named “OpenAI Model for Coordination Agent”
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - Credentials: same OpenAI credential
- Add node: **Structured Output Parser** named “Coordination Output Parser”
  - Schema example containing actionType/priority/workflowActions/replenishmentPlan/complianceStatus/auditTrail/reasoning
- Add node: **AI Agent** named “Coordination Agent”
  - Prompt includes the risk analysis JSON and its risk level/score
  - System message includes routing logic (CRITICAL/HIGH/MEDIUM/LOW)
  - Enable output parser usage
- (Optional but in this workflow) Add node: **Slack Tool** named “Slack Alert Tool”
  - OAuth2 Slack credentials
  - Channel selection via `$fromAI('slackChannel',...)`
  - Text via `$fromAI('alertMessage',...)`
- Wire:
  - Route by Risk Level → Coordination Agent (main)
  - OpenAI Model for Coordination → Coordination Agent (ai_languageModel)
  - Coordination Output Parser → Coordination Agent (ai_outputParser)
  - Slack Alert Tool → Coordination Agent (ai_tool)

8) **Action type routing**
- Add node: **Switch** named “Route by Action Type”
- Rules:
  - `$json.output.actionType == "ALERT"`
  - `$json.output.actionType == "ESCALATE"`
  - `$json.output.actionType == "APPROVE"` (recommended to wire; in the provided workflow it is not wired)
  - (Optional) `$json.output.actionType == "LOG"`
- Connect Coordination Agent → Route by Action Type.

9) **Slack alert execution node**
- Add node: **Slack** named “Send Critical Alert to Slack”
- OAuth2 Slack credentials
- Channel ID: `$('Workflow Configuration').first().json.slackChannelId`
- Message template using `$json.output.*` fields.
- Connect Route by Action Type (ALERT) → Send Critical Alert to Slack.

10) **Email escalation node**
- Add node: **Email Send** named “Send Escalation Email”
- Configure email transport credentials in n8n (SMTP/provider)
- To: `$('Workflow Configuration').first().json.escalationEmail`
- From: set a valid sender address
- Subject + HTML body from `$json.output.*`
- Connect Route by Action Type (ESCALATE) → Send Escalation Email.

11) **Audit logging**
- Add node: **Code** named “Log Compliance Audit Trail”
- Implement code to normalize and store an audit entry (as per workflow logic).
- Connect:
  - Send Critical Alert to Slack → Log Compliance Audit Trail
  - Send Escalation Email → Log Compliance Audit Trail
  - (Later) approval processing → Log Compliance Audit Trail

12) **Final report**
- Add node: **Set** named “Prepare Final Report”
- Add fields: reportId, timestamp, count, summary, status
- Connect Log Compliance Audit Trail → Prepare Final Report.

13) **Manual approval entry point (separate path)**
- Add node: **Webhook** named “Manual Approval Webhook”
  - Method: POST
  - Path: `supply-chain-approval`
  - Response mode: last node
- Add node: **Code** named “Process Approval Decision”
  - Normalize `decision`, `approver`, `comments`; compute `status` and `approvalId`
- Add node: **Respond to Webhook** named “Respond to Approval”
  - Respond with JSON confirmation including decision + timestamp
- Wire:
  - Manual Approval Webhook → Process Approval Decision
  - Process Approval Decision → Respond to Approval
  - Process Approval Decision → Log Compliance Audit Trail

14) **Credentials checklist**
- **OpenAI**: API key with access to `gpt-4o`
- **Slack OAuth2**: scopes to post messages to the target channel
- **Email**: SMTP/provider configuration + valid from address
- **HTTP APIs**: tokens/headers as required by ERP/WMS/TMS endpoints

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow is currently inactive (`active: false`). | Enable it in n8n for the schedule trigger to run. |
| Placeholders must be replaced: procurementApiUrl, warehouseApiUrl, slackChannelId, escalationEmail, fromEmail. | Required to avoid runtime failures. |
| Risk thresholds (80/60/40) are defined but not used in routing; routing depends on AI-produced `riskLevel`. | Consider enforcing thresholds in code or switch logic if deterministic behavior is required. |
| LOW risk is not explicitly handled in “Route by Risk Level”; LOG actionType is not explicitly handled in “Route by Action Type”; APPROVE branch is not connected. | Add explicit switch rules and wire to logging/approval flows to match the coordination agent’s stated logic. |
| Manual approval webhook has no built-in authentication/verification. | Add a shared secret, header validation, or protect endpoint via reverse proxy/auth. |
| Sticky note claims “parallel analysis”, but the agents run sequentially in the current wiring (Signal Monitoring → Coordination). | If true parallelism is desired, split paths after merge and later recombine. |