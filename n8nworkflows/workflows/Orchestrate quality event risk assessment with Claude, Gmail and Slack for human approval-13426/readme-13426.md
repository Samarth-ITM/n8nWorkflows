Orchestrate quality event risk assessment with Claude, Gmail and Slack for human approval

https://n8nworkflows.xyz/workflows/orchestrate-quality-event-risk-assessment-with-claude--gmail-and-slack-for-human-approval-13426


# Orchestrate quality event risk assessment with Claude, Gmail and Slack for human approval

## 1. Workflow Overview

**Workflow name:** Smart quality event risk orchestration for human supervision  
**Stated title:** Orchestrate quality event risk assessment with Claude, Gmail and Slack for human approval

**Purpose:**  
This workflow receives a *quality event* (e.g., defect, contamination, nonconformance) via HTTP webhook, runs a **multi-agent AI analysis** (traceability validation → risk scoring → containment/recall planning), then **routes notifications** (email + Slack) and enforces **human approval** for executive-level actions. It also produces a final **audit trail log**.

**Target use cases:**  
Manufacturing defect escalation, food safety incident management, quality/compliance incident triage where automated analysis is allowed but final high-impact decisions must be human-approved.

### 1.1 Input Reception & Configuration
Receives the event payload and injects configurable recipients, thresholds, and channel IDs.

### 1.2 AI Multi-Agent Orchestration (Claude via Anthropic)
A master agent coordinates three specialized agents:
- Traceability validation (data completeness/lineage/compliance)
- Risk assessment (safety/regulatory/financial scoring)
- Recall/containment orchestration (plans actions, never executes)

### 1.3 Risk Routing & Human Approval Gate
Routes by overall risk level and checks whether executive approval is required. If required, the workflow pauses until a webhook-based resume occurs.

### 1.4 Notifications & Audit Trail
Sends Slack notifications and/or critical/executive emails based on routing, merges branches, and logs a structured audit record.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception & Runtime Configuration
**Overview:** Accepts POSTed quality events and sets workflow-level configuration values (recipients, thresholds, Slack channel).  
**Nodes involved:** `Quality Event Webhook`, `Workflow Configuration`

#### Node: Quality Event Webhook
- **Type / role:** `Webhook` — entry point to receive the quality event payload.
- **Key configuration:**
  - **Method:** POST
  - **Path:** `/quality-event`
  - **Response mode:** *Last node* (the HTTP response is whatever the last executed node returns)
- **Input / output:**
  - **Output:** to `Workflow Configuration`
- **Edge cases / failures:**
  - Invalid/missing JSON body can break downstream AI expectations (agent prompts assume structured data).
  - If workflow errors before reaching a “last node”, webhook caller may get error/timeout.
- **Version notes:** Webhook `typeVersion 2.1` (n8n Webhook v2 behavior).

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines configuration constants and keeps the incoming event payload.
- **Key configuration choices:**
  - `includeOtherFields: true` → preserves inbound webhook payload fields.
  - Assigns:
    - `criticalEmailRecipient` (placeholder)
    - `executiveEmailRecipient` (placeholder)
    - `slackChannelId` (placeholder)
    - `qualityThresholdScore` = **85** (note: currently not used in routing logic)
    - `recallRiskThreshold` = **"MEDIUM"** (note: currently not used in routing logic)
- **Input / output:**
  - Input: webhook payload
  - Output: to `Master Orchestrator Agent`
- **Edge cases / failures:**
  - Placeholders must be replaced or downstream email/Slack nodes will fail.
  - Threshold variables are defined but not referenced later; users may expect them to affect routing (they currently do not).
- **Version notes:** Set `typeVersion 3.4`.

**Sticky note context (applies to this block):**
- **How It Works:** Explains multi-agent analysis + human oversight.
- **Setup Steps / Prerequisites:** Mentions NVIDIA NIM and Gmail app password (but this workflow actually uses **Anthropic** nodes and `Email Send` SMTP, not Gmail nodes).

---

### Block 2.2 — AI Orchestration (Master Agent + Specialized Tools)
**Overview:** A master AI agent coordinates three specialized AI “tool agents” in sequence and returns one structured summary (`output`) used for routing.  
**Nodes involved:**  
`Master Orchestrator Agent`, `Anthropic Model - Orchestrator`, `Orchestrator Output Parser`,  
`Traceability Agent Tool`, `Anthropic Model - Traceability Agent`, `Traceability Output Parser`,  
`Risk Assessment Agent Tool`, `Anthropic Model - Risk Assessment Agent`, `Risk Assessment Output Parser`,  
`Recall Orchestration Agent Tool`, `Anthropic Model - Recall Orchestration Agent`, `Recall Orchestration Output Parser`

#### Node: Master Orchestrator Agent
- **Type / role:** `LangChain Agent` — the coordinator that calls the three tools and synthesizes results.
- **Key configuration choices:**
  - **Prompt input:** `text = {{$json}}` (passes the whole item: webhook payload + configuration fields)
  - **System message:** instructs the agent to:
    1) call Traceability tool, 2) call Risk tool, 3) call Recall tool, 4) synthesize results, 5) output summary.
  - **Has output parser:** yes (structured output expected)
- **Connections:**
  - **AI language model:** from `Anthropic Model - Orchestrator`
  - **Tools available:** `Traceability Agent Tool`, `Risk Assessment Agent Tool`, `Recall Orchestration Agent Tool`
  - **Output parser:** `Orchestrator Output Parser`
  - **Main output:** to `Route by Risk Level`
- **Output shape (expected):** stored under `{{$json.output...}}` downstream (not raw text).
- **Edge cases / failures:**
  - If the agent does not produce output matching the parser schema, parsing fails and routing breaks.
  - Any tool failure (rate limit, invalid JSON) can cascade to orchestrator failure.
- **Version notes:** Agent `typeVersion 3.1` (LangChain agent node).

#### Node: Anthropic Model - Orchestrator
- **Type / role:** `lmChatAnthropic` — Claude chat model provider for the master agent.
- **Key configuration:**
  - Model: `claude-sonnet-4-5-20250929`
  - Credential: `Anthropic account`
- **Connections:** provides `ai_languageModel` to `Master Orchestrator Agent`
- **Edge cases / failures:**
  - Anthropic API auth errors, quota/rate limiting, model name availability.
- **Version notes:** `typeVersion 1.3`.

#### Node: Orchestrator Output Parser
- **Type / role:** `Structured Output Parser` — forces master agent output into a known JSON schema.
- **Schema (example fields):**
  - `overallRiskLevel` (LOW/MEDIUM/HIGH/CRITICAL)
  - `traceabilityValidated`, `riskAssessmentComplete`, `recallOrchestrationComplete`
  - `actionRequired` (e.g., `EXECUTIVE_APPROVAL`)
  - `summary`, `agentsCalled`
- **Connections:** attached via `ai_outputParser` to `Master Orchestrator Agent`
- **Edge cases:** if the agent returns unparseable JSON or missing required keys, workflow errors.

---

#### Node: Traceability Agent Tool
- **Type / role:** `agentTool` — specialized tool used by the master agent to validate traceability.
- **Key configuration:**
  - **Tool input expression:**  
    `{{$fromAI("qualityEventData", "Quality event data including part numbers, batch numbers, and quality metrics", "json")}}`
  - **System message:** strict scope: validate data only, no recall decisions.
  - **Has output parser:** yes → `Traceability Output Parser`
- **Connections:**
  - `ai_languageModel` from `Anthropic Model - Traceability Agent`
  - `ai_outputParser` to `Traceability Output Parser`
  - Tool is connected into `Master Orchestrator Agent` as an available tool.
- **Edge cases / failures:**
  - `$fromAI(...)` expects the master agent to supply a compatible JSON payload; if missing/malformed, the tool may get unusable input.
  - Parser mismatch causes failures.
- **Version notes:** `typeVersion 3`.

#### Node: Anthropic Model - Traceability Agent
- **Type / role:** Claude model provider for the traceability tool.
- **Config:** same model `claude-sonnet-4-5-20250929`, same Anthropic credential.
- **Edge cases:** same as other Anthropic model nodes.

#### Node: Traceability Output Parser
- **Type / role:** structured parser for traceability results.
- **Schema (example fields):**
  - `validationStatus` (VALID/INVALID/SUSPICIOUS)
  - `partNumber`, `batchNumber`, `lineageComplete`, `missingData`
  - `qualityScore` (0–100), `traceabilityChain`, `complianceStatus`, `reasoning`
- **Edge cases:** traceability tool output not matching schema.

---

#### Node: Risk Assessment Agent Tool
- **Type / role:** `agentTool` — evaluates risk using validated traceability results.
- **Key configuration:**
  - Tool input:  
    `{{$fromAI("traceabilityResults", "Validated traceability data from the Traceability Agent", "json")}}`
  - System message: compute risk score/level, recommend action; does not initiate recalls.
  - Output parser: `Risk Assessment Output Parser`
- **Connections:**
  - `ai_languageModel` from `Anthropic Model - Risk Assessment Agent`
  - `ai_outputParser` to `Risk Assessment Output Parser`
  - Tool registered to `Master Orchestrator Agent`
- **Edge cases:**
  - If traceability output is missing, risk tool may hallucinate or fail parsing.
  - Risk levels must align with routing rules later (LOW/MEDIUM/HIGH/CRITICAL).

#### Node: Anthropic Model - Risk Assessment Agent
- **Type / role:** Claude model provider for risk tool.
- **Config:** `claude-sonnet-4-5-20250929`, Anthropic credential.

#### Node: Risk Assessment Output Parser
- **Type / role:** structured parser for risk results.
- **Schema (example fields):**
  - `riskLevel`, `riskScore`, `affectedUnits`
  - `customerImpact`, `safetyRisk`, `regulatoryRisk`
  - `financialImpact`, `recommendedAction`, `reasoning`

---

#### Node: Recall Orchestration Agent Tool
- **Type / role:** `agentTool` — prepares containment workflow artifacts and escalation needs.
- **Key configuration:**
  - Tool input:  
    `{{$fromAI("riskAssessmentResults", "Risk assessment results from the Risk Assessment Agent", "json")}}`
  - System message: **never initiate recalls autonomously**, prepares plans only.
  - Output parser: `Recall Orchestration Output Parser`
- **Connections:**
  - `ai_languageModel` from `Anthropic Model - Recall Orchestration Agent`
  - `ai_outputParser` to `Recall Orchestration Output Parser`
  - Tool registered to `Master Orchestrator Agent`
- **Edge cases:**
  - If risk output is inconsistent (e.g., riskLevel missing), orchestration plan may be invalid or parser may fail.
  - “requiresHumanApproval” is produced here but the workflow’s approval gate is actually driven by **master output** field `actionRequired`.

#### Node: Anthropic Model - Recall Orchestration Agent
- **Type / role:** Claude model provider for recall orchestration tool.
- **Config:** `claude-sonnet-4-5-20250929`, Anthropic credential.

#### Node: Recall Orchestration Output Parser
- **Type / role:** structured parser for orchestration plan.
- **Schema (example fields):**
  - `recallRequired` (boolean)
  - `containmentWorkflow` (array)
  - `reportingArtifacts` (array)
  - `executiveAlertRequired` (boolean)
  - `estimatedTimeline`, `stakeholders`, `requiresHumanApproval`, `reasoning`

**Sticky note context (applies to this block):**
- **Traceability Analysis:** “Identifies contamination sources and affected product scope…”
- **Risk Assessment:** “Quantifies business impact and legal obligations…”
- **Recall Evaluation:** “Ensures regulatory compliance and protects consumer safety…”

---

### Block 2.3 — Risk Routing & Human Approval Gate
**Overview:** Routes the master agent’s structured output by risk level and enforces a pause if executive approval is required.  
**Nodes involved:** `Route by Risk Level`, `Check Requires Human Approval`, `Wait for Human Approval`

#### Node: Route by Risk Level
- **Type / role:** `Switch` — routes by `{{$json.output.overallRiskLevel}}`.
- **Rules / outputs:**
  - **High Risk**: equals `HIGH` → to `Check Requires Human Approval`
  - **Critical Risk**: equals `CRITICAL` → to `Send Critical Alert Email`
  - **Low/Medium Risk**: equals `LOW` or `MEDIUM` → to `Send Slack Notification`
  - **Fallback:** `Unclassified` (if none matched) → no downstream connection defined (potential dead-end)
- **Edge cases / failures:**
  - If `overallRiskLevel` is missing/typoed (e.g., “High”), it will go to fallback and effectively stop (no connected node).
- **Version notes:** Switch `typeVersion 3.4`.

#### Node: Check Requires Human Approval
- **Type / role:** `If` — checks whether action requires executive approval.
- **Condition:** `{{$json.output.actionRequired}} == "EXECUTIVE_APPROVAL"`
- **Outputs:**
  - **True branch:** to `Wait for Human Approval`
  - **False branch:** to `Send Executive Alert Email` (note: still sends executive email even when not requiring approval)
- **Edge cases / behavior notes:**
  - If `overallRiskLevel` is HIGH but `actionRequired` is not `EXECUTIVE_APPROVAL`, the workflow still sends the “EXECUTIVE APPROVAL REQUIRED” email on the **false** branch, which may be logically inconsistent. (This is likely a design bug: false branch probably intended a different email or Slack notification.)
- **Version notes:** IF `typeVersion 2.3`.

#### Node: Wait for Human Approval
- **Type / role:** `Wait` — pauses execution until resumed via webhook (human approval action).
- **Key configuration:**
  - Resume mode: `webhook`
  - HTTP method: POST
- **Output / next node:** resumes to `Send Executive Alert Email`
- **Edge cases / failures:**
  - If the resume webhook is never called, executions remain waiting indefinitely (until manual intervention/cleanup).
  - The workflow does not parse/validate the approval payload; it only resumes and proceeds to send an email.
  - Email content says “approve or reject… use webhook URL…” but no decision is processed in logic.
- **Version notes:** Wait `typeVersion 1.1`.

---

### Block 2.4 — Notifications & Audit Trail
**Overview:** Sends Slack and email notifications based on risk and merges branches into an audit log record.  
**Nodes involved:** `Send Critical Alert Email`, `Send Executive Alert Email`, `Send Slack Notification`, `Merge Notification Paths`, `Log Audit Trail`

#### Node: Send Critical Alert Email
- **Type / role:** `Email Send` — sends critical alert email (HTML).
- **Recipients:**
  - To: `{{$('Workflow Configuration').first().json.criticalEmailRecipient}}`
  - From: placeholder `Sender Email Address`
- **Subject:** `CRITICAL QUALITY ALERT: {{ overallRiskLevel }} Risk - timestamp`
- **Body:** HTML includes summary, agents list, action required.
- **Connections:** to `Merge Notification Paths`
- **Edge cases / failures:**
  - SMTP credentials required in n8n (not shown in JSON). Missing/invalid SMTP config will fail.
  - If `output.agentsCalled` is missing/not an array, `.map(...)` will throw in expressions.
- **Version notes:** EmailSend `typeVersion 2.1`.

#### Node: Send Executive Alert Email
- **Type / role:** `Email Send` — sends “Executive approval required” email.
- **Recipients:**
  - To: `{{$('Workflow Configuration').first().json.executiveEmailRecipient}}`
  - From: placeholder sender email
- **Body:** includes statuses and instructs to use approval webhook URL from execution context.
- **Connections:** to `Merge Notification Paths` (connected as branch index 1)
- **Edge cases / mismatches:**
  - The workflow does not embed the actual resume webhook URL into the email automatically; the email just says to use it.
  - This node can be reached both:
    - immediately from IF false branch, and
    - after `Wait for Human Approval` resumes,
    which may cause duplicate/confusing emails.
- **Version notes:** EmailSend `typeVersion 2.1`.

#### Node: Send Slack Notification
- **Type / role:** `Slack` — posts alert summary to a channel.
- **Authentication:** OAuth2 (`Slack account` credential)
- **Channel:** `{{$('Workflow Configuration').first().json.slackChannelId}}`
- **Message:** includes risk, action, summary, agents list, timestamp.
- **Connections:** to `Merge Notification Paths`
- **Edge cases / failures:**
  - Invalid channel ID, insufficient Slack scopes, token revoked.
  - If `agentsCalled` isn’t an array, `.map(...)` expression may fail.
- **Version notes:** Slack `typeVersion 2.4`.

#### Node: Merge Notification Paths
- **Type / role:** `Merge` — joins one of multiple notification branches.
- **Mode:** `chooseBranch` (passes through whichever branch produced data)
- **Inputs:** from Slack, Critical Email, Executive Email
- **Output:** to `Log Audit Trail`
- **Edge cases:**
  - With multiple branches firing simultaneously (not typical here), chooseBranch behavior depends on arrival/order and may drop data from one branch.

#### Node: Log Audit Trail
- **Type / role:** `Code` — emits a normalized audit log JSON and prints it.
- **Logic:**
  - Uses `$json.output` (or `{}`) to create `auditLog`:
    - timestamp, riskLevel, actionRequired, booleans, agentsCalled, summary
    - `notificationsSent: true`, `workflowStatus: "COMPLETED"`
- **Output:** returns `{ json: auditLog }` (this becomes webhook response if last node executed)
- **Edge cases:**
  - If upstream item at this point is the Slack/email node output and lacks `.output`, audit will record UNKNOWN/NONE, losing orchestrator output. (Currently it explicitly reads `$json.output`, which may not exist after notification nodes depending on how n8n merges item data.)
  - Mitigation: store orchestrator output in a separate field before notifications, or merge by key.
- **Version notes:** Code `typeVersion 2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Quality Event Webhook | Webhook | Receives quality event via HTTP POST | — | Workflow Configuration | ## Traceability Analysis  \nIdentifies contamination sources and affected product scope for containment decisions. |
| Workflow Configuration | Set | Sets recipients/thresholds/channel while keeping inbound fields | Quality Event Webhook | Master Orchestrator Agent | ## How It Works\nThis workflow automates quality event risk assessment through AI-powered multi-agent analysis with mandatory human oversight for critical decisions. Designed for quality managers, compliance officers, and risk analysts in manufacturing, healthcare, or service industries, it solves the challenge of consistent, transparent risk evaluation while maintaining human accountability. When quality events are detected, the system orchestrates specialized AI agents (traceability, risk assessment, and recall evaluation) to analyze different risk dimensions simultaneously. Results are synthesized, routed through human approval gates based on risk severity, and distributed via automated notifications. This ensures high-risk decisions receive proper scrutiny while low-risk events flow efficiently through automated channels. |
| Anthropic Model - Traceability Agent | lmChatAnthropic | Claude model for traceability tool | — | Traceability Agent Tool (ai_languageModel) | ## Traceability Analysis  \nIdentifies contamination sources and affected product scope for containment decisions. |
| Traceability Agent Tool | agentTool | Validates part/batch/lineage/compliance; returns structured results | Master Orchestrator Agent (as tool call) | Master Orchestrator Agent (ai_tool); Traceability Output Parser (ai_outputParser) | ## Traceability Analysis  \nIdentifies contamination sources and affected product scope for containment decisions. |
| Traceability Output Parser | outputParserStructured | Enforces traceability result schema | Traceability Agent Tool | — | ## Traceability Analysis  \nIdentifies contamination sources and affected product scope for containment decisions. |
| Anthropic Model - Risk Assessment Agent | lmChatAnthropic | Claude model for risk tool | — | Risk Assessment Agent Tool (ai_languageModel) | ## Risk Assessment\nQuantifies business impact and legal obligations to prioritize response actions. |
| Risk Assessment Agent Tool | agentTool | Computes risk score/level and recommendation | Master Orchestrator Agent (as tool call) | Master Orchestrator Agent (ai_tool); Risk Assessment Output Parser (ai_outputParser) | ## Risk Assessment\nQuantifies business impact and legal obligations to prioritize response actions. |
| Risk Assessment Output Parser | outputParserStructured | Enforces risk assessment schema | Risk Assessment Agent Tool | — | ## Risk Assessment\nQuantifies business impact and legal obligations to prioritize response actions. |
| Anthropic Model - Recall Orchestration Agent | lmChatAnthropic | Claude model for recall/containment planning tool | — | Recall Orchestration Agent Tool (ai_languageModel) | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Recall Orchestration Agent Tool | agentTool | Produces containment steps, artifacts, stakeholders; never executes recall | Master Orchestrator Agent (as tool call) | Master Orchestrator Agent (ai_tool); Recall Orchestration Output Parser (ai_outputParser) | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Recall Orchestration Output Parser | outputParserStructured | Enforces recall orchestration schema | Recall Orchestration Agent Tool | — | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Anthropic Model - Orchestrator | lmChatAnthropic | Claude model for master agent | — | Master Orchestrator Agent (ai_languageModel) | ## How It Works\nThis workflow automates quality event risk assessment through AI-powered multi-agent analysis with mandatory human oversight for critical decisions. Designed for quality managers, compliance officers, and risk analysts in manufacturing, healthcare, or service industries, it solves the challenge of consistent, transparent risk evaluation while maintaining human accountability. When quality events are detected, the system orchestrates specialized AI agents (traceability, risk assessment, and recall evaluation) to analyze different risk dimensions simultaneously. Results are synthesized, routed through human approval gates based on risk severity, and distributed via automated notifications. This ensures high-risk decisions receive proper scrutiny while low-risk events flow efficiently through automated channels. |
| Orchestrator Output Parser | outputParserStructured | Enforces master orchestration summary schema | Master Orchestrator Agent | — | ## How It Works\nThis workflow automates quality event risk assessment through AI-powered multi-agent analysis with mandatory human oversight for critical decisions. Designed for quality managers, compliance officers, and risk analysts in manufacturing, healthcare, or service industries, it solves the challenge of consistent, transparent risk evaluation while maintaining human accountability. When quality events are detected, the system orchestrates specialized AI agents (traceability, risk assessment, and recall evaluation) to analyze different risk dimensions simultaneously. Results are synthesized, routed through human approval gates based on risk severity, and distributed via automated notifications. This ensures high-risk decisions receive proper scrutiny while low-risk events flow efficiently through automated channels. |
| Master Orchestrator Agent | agent | Coordinates 3 AI tools and outputs final summary | Workflow Configuration | Route by Risk Level | ## How It Works\nThis workflow automates quality event risk assessment through AI-powered multi-agent analysis with mandatory human oversight for critical decisions. Designed for quality managers, compliance officers, and risk analysts in manufacturing, healthcare, or service industries, it solves the challenge of consistent, transparent risk evaluation while maintaining human accountability. When quality events are detected, the system orchestrates specialized AI agents (traceability, risk assessment, and recall evaluation) to analyze different risk dimensions simultaneously. Results are synthesized, routed through human approval gates based on risk severity, and distributed via automated notifications. This ensures high-risk decisions receive proper scrutiny while low-risk events flow efficiently through automated channels. |
| Route by Risk Level | Switch | Routes LOW/MEDIUM vs HIGH vs CRITICAL | Master Orchestrator Agent | Check Requires Human Approval; Send Critical Alert Email; Send Slack Notification | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Check Requires Human Approval | If | Checks `actionRequired == EXECUTIVE_APPROVAL` | Route by Risk Level (High Risk output) | Wait for Human Approval; Send Executive Alert Email | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Wait for Human Approval | Wait | Pauses until resume webhook is called | Check Requires Human Approval | Send Executive Alert Email | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Send Critical Alert Email | Email Send | Emails critical alert for CRITICAL risk | Route by Risk Level (Critical Risk output) | Merge Notification Paths | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Send Executive Alert Email | Email Send | Emails executive approval request/status | Check Requires Human Approval; Wait for Human Approval | Merge Notification Paths | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Send Slack Notification | Slack | Posts Slack alert for LOW/MEDIUM | Route by Risk Level (Low/Medium Risk output) | Merge Notification Paths | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Merge Notification Paths | Merge | Consolidates notification branches | Send Slack Notification; Send Critical Alert Email; Send Executive Alert Email | Log Audit Trail | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Log Audit Trail | Code | Emits final audit log JSON and logs to console | Merge Notification Paths | — | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Sticky Note | Sticky Note | Comment | — | — | ## Traceability Analysis  \nIdentifies contamination sources and affected product scope for containment decisions. |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Prerequisites\nNVIDIA NIM API key, Gmail account with app password\n## Use Cases\nManufacturing defect escalation, food safety incident management\n## Customization\nModify risk scoring thresholds, add industry-specific compliance agents\n## Benefits\nReduces risk assessment time by 75%, ensures consistent evaluation methodology |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Setup Steps\n1. Configure NVIDIA NIM API credentials with Llama-3.1-70B-Instruct model access\n2. Set up routing logic thresholds\n3. Connect Gmail SMTP for executive alerts and Slack webhook for team notifications\n4. Configure human approval nodes with designated approver email addresses\n5. Customize AI agent prompts for industry-specific risk criteria |
| Sticky Note3 | Sticky Note | Comment | — | — | ## How It Works\nThis workflow automates quality event risk assessment through AI-powered multi-agent analysis with mandatory human oversight for critical decisions. Designed for quality managers, compliance officers, and risk analysts in manufacturing, healthcare, or service industries, it solves the challenge of consistent, transparent risk evaluation while maintaining human accountability. When quality events are detected, the system orchestrates specialized AI agents (traceability, risk assessment, and recall evaluation) to analyze different risk dimensions simultaneously. Results are synthesized, routed through human approval gates based on risk severity, and distributed via automated notifications. This ensures high-risk decisions receive proper scrutiny while low-risk events flow efficiently through automated channels. |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Recall Evaluation \nEnsures regulatory compliance and protects consumer safety through proper escalation protocols. |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Risk Assessment\nQuantifies business impact and legal obligations to prioritize response actions. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: `quality-event`
   - Response Mode: **Last Node**
   - Save to generate the production/test URLs.

2. **Add configuration node**
   - Add node: **Set** (name: `Workflow Configuration`)
   - Enable: **Include Other Fields**
   - Add fields:
     - `criticalEmailRecipient` (string)
     - `executiveEmailRecipient` (string)
     - `slackChannelId` (string)
     - `qualityThresholdScore` (number) = 85
     - `recallRiskThreshold` (string) = "MEDIUM"
   - Connect: `Quality Event Webhook` → `Workflow Configuration`

3. **Create the Master AI agent**
   - Add node: **AI Agent** (LangChain) (name: `Master Orchestrator Agent`)
   - Prompt input: `{{$json}}`
   - System message: describe sequential tool calling (traceability → risk → recall) and synthesis (as in workflow).
   - Enable **Structured Output** and attach an output parser (step 5).
   - Connect: `Workflow Configuration` → `Master Orchestrator Agent`

4. **Add Claude model for the Master agent**
   - Add node: **Anthropic Chat Model** (name: `Anthropic Model - Orchestrator`)
   - Select model: `claude-sonnet-4-5-20250929` (or closest available)
   - Set **Anthropic API credentials**
   - Connect model to agent via **ai_languageModel**.

5. **Add Structured Output Parser for the Master agent**
   - Add node: **Structured Output Parser**
   - Provide a JSON schema example with keys:
     `overallRiskLevel`, `traceabilityValidated`, `riskAssessmentComplete`, `recallOrchestrationComplete`, `actionRequired`, `summary`, `agentsCalled`
   - Connect parser to agent via **ai_outputParser**.

6. **Create Traceability Tool + model + parser**
   - Add node: **AI Agent Tool** (name: `Traceability Agent Tool`)
     - Tool input expression:  
       `{{$fromAI("qualityEventData","Quality event data including part numbers, batch numbers, and quality metrics","json")}}`
     - System message: validate traceability only.
     - Enable structured output and attach parser (next).
   - Add node: **Anthropic Chat Model** (name: `Anthropic Model - Traceability Agent`) with same credentials/model.
   - Add node: **Structured Output Parser** (name: `Traceability Output Parser`) with the example schema.
   - Connect:
     - Model → Tool via **ai_languageModel**
     - Parser → Tool via **ai_outputParser**
     - Tool → `Master Orchestrator Agent` via **ai_tool**

7. **Create Risk Assessment Tool + model + parser**
   - Add node: **AI Agent Tool** (name: `Risk Assessment Agent Tool`)
     - Tool input:  
       `{{$fromAI("traceabilityResults","Validated traceability data from the Traceability Agent","json")}}`
     - System message: compute risk level/score; no recall execution.
   - Add node: **Anthropic Chat Model** (name: `Anthropic Model - Risk Assessment Agent`)
   - Add node: **Structured Output Parser** (name: `Risk Assessment Output Parser`)
   - Connect model/parser to tool, and tool to master agent (ai_tool).

8. **Create Recall Orchestration Tool + model + parser**
   - Add node: **AI Agent Tool** (name: `Recall Orchestration Agent Tool`)
     - Tool input:  
       `{{$fromAI("riskAssessmentResults","Risk assessment results from the Risk Assessment Agent","json")}}`
     - System message: prepare containment workflow; never initiate recalls; human approval required for high risk actions.
   - Add node: **Anthropic Chat Model** (name: `Anthropic Model - Recall Orchestration Agent`)
   - Add node: **Structured Output Parser** (name: `Recall Orchestration Output Parser`)
   - Connect model/parser to tool, and tool to master agent (ai_tool).

9. **Add risk routing**
   - Add node: **Switch** (name: `Route by Risk Level`)
   - Switch on: `{{$json.output.overallRiskLevel}}`
   - Create outputs:
     - HIGH → output “High Risk”
     - CRITICAL → output “Critical Risk”
     - LOW or MEDIUM → output “Low/Medium Risk”
   - Connect: `Master Orchestrator Agent` → `Route by Risk Level`

10. **Add approval check for HIGH**
   - Add node: **IF** (name: `Check Requires Human Approval`)
   - Condition: `{{$json.output.actionRequired}}` equals `EXECUTIVE_APPROVAL`
   - Connect: `Route by Risk Level (High Risk)` → `Check Requires Human Approval`

11. **Add wait-for-approval**
   - Add node: **Wait** (name: `Wait for Human Approval`)
   - Resume: **Webhook**
   - Method: POST
   - Connect: `Check Requires Human Approval (true)` → `Wait for Human Approval`

12. **Add email notifications (SMTP)**
   - Add node: **Email Send** (name: `Send Critical Alert Email`)
     - To: `{{$('Workflow Configuration').first().json.criticalEmailRecipient}}`
     - From: your sender address
     - Subject/body using `{{$json.output...}}`
     - Configure SMTP credentials in n8n (host/port/user/pass or provider settings).
   - Add node: **Email Send** (name: `Send Executive Alert Email`)
     - To: `{{$('Workflow Configuration').first().json.executiveEmailRecipient}}`
     - From: sender address
     - HTML body including summary/status fields.
   - Connect:
     - `Route by Risk Level (Critical Risk)` → `Send Critical Alert Email`
     - `Check Requires Human Approval (false)` → `Send Executive Alert Email`
     - `Wait for Human Approval` → `Send Executive Alert Email`

13. **Add Slack notification (OAuth2)**
   - Add node: **Slack**
   - Auth: **OAuth2**
   - Operation: send message to channel
   - Channel ID: `{{$('Workflow Configuration').first().json.slackChannelId}}`
   - Text uses `{{$json.output...}}`
   - Connect: `Route by Risk Level (Low/Medium Risk)` → `Send Slack Notification`

14. **Merge notification branches**
   - Add node: **Merge** (name: `Merge Notification Paths`)
   - Mode: **Choose Branch**
   - Connect:
     - `Send Slack Notification` → Merge
     - `Send Critical Alert Email` → Merge
     - `Send Executive Alert Email` → Merge

15. **Add audit log**
   - Add node: **Code** (name: `Log Audit Trail`)
   - Implement the audit JSON construction (timestamp, riskLevel, actionRequired, flags, agentsCalled, summary).
   - Connect: `Merge Notification Paths` → `Log Audit Trail`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Prerequisites NVIDIA NIM API key, Gmail account with app password …” | Sticky note states NVIDIA NIM + Gmail, but the actual workflow uses **Anthropic Claude** nodes and **Email Send (SMTP)** + **Slack OAuth2**. Update prerequisites to match your real credentials. |
| “## Setup Steps … Configure NVIDIA NIM … Connect Gmail SMTP … Slack webhook …” | Sticky note guidance is partially inconsistent with node choices (Slack node uses OAuth2, not incoming webhook). Adjust to your environment. |
| “Workflow automates quality event risk assessment … mandatory human oversight …” | High-level design intent: multi-agent analysis + escalation gates. |
| Human approval mechanism | Wait node resumes via webhook, but the workflow does **not** parse an approval/rejection decision. If you need real approvals, add a Set/IF node after resume to validate the posted decision and branch accordingly. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.