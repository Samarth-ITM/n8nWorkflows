Detect procurement fraud and monitor supplier compliance with GPT-4o and Slack

https://n8nworkflows.xyz/workflows/detect-procurement-fraud-and-monitor-supplier-compliance-with-gpt-4o-and-slack-13330


# Detect procurement fraud and monitor supplier compliance with GPT-4o and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs every 15 minutes to analyze incoming “events” (currently simulated sample events), prioritize them with GPT‑4o, decide delivery strategy with a second GPT‑4o agent, store events by priority in n8n Data Tables, notify via Slack/email for higher priorities, and optionally trigger an executive escalation email when required.

**Target use cases:** Despite the title mentioning procurement fraud and supplier compliance, the implemented sample data and prompts currently model **IT/ops/security event triage** (CPU alerts, login failures, API latency, etc.). The same architecture can be adapted to procurement by replacing the sample event generator and adjusting prompts/fields.

### Logical blocks
1.1 **Scheduling & Configuration** → runs periodically and sets shared variables (Slack channels, emails, thresholds).  
1.2 **Event Intake (Simulated)** → generates 5 sample events as separate items.  
1.3 **AI Prioritization (+ Enrichment Tool)** → GPT‑4o agent assigns score/level and can call an enrichment tool.  
1.4 **AI Delivery Orchestration (with Slack/Email tools)** → GPT‑4o agent determines channel(s) and message plan (tools are available).  
1.5 **Routing, Storage, Notifications & Audit Logging** → switch by priority, store in Data Tables, send Slack/email alerts, merge, log to audit table.  
1.6 **Escalation Path** → if `requiresEscalation=true`, run escalation agent and send escalation email.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Runtime Configuration
**Overview:** Triggers the workflow every 15 minutes and defines runtime configuration values used by later Slack/email nodes.  
**Nodes involved:** `Schedule Trigger`, `Workflow Configuration`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — periodic workflow entrypoint.
- **Key configuration:** Runs every **15 minutes**.
- **Outputs:** Sends a single empty item to `Workflow Configuration`.
- **Edge cases / failures:** None typical beyond workflow being inactive or instance scheduling disabled.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — injects configuration fields into the item.
- **Configuration choices (interpreted):**
  - Adds variables:
    - `criticalSlackChannel` (placeholder)
    - `highPrioritySlackChannel` (placeholder)
    - `escalationEmail` (placeholder)
    - `criticalEmailRecipient` (placeholder)
    - Threshold numbers: `priorityThresholdCritical=90`, `priorityThresholdHigh=70`, `priorityThresholdMedium=40`
  - **Include other fields:** enabled (keeps incoming fields too).
- **Used later by expressions:**
  - Slack nodes reference:
    - `{{ $('Workflow Configuration').first().json.criticalSlackChannel }}`
    - `{{ $('Workflow Configuration').first().json.highPrioritySlackChannel }}`
  - Email nodes reference:
    - `{{ $('Workflow Configuration').first().json.criticalEmailRecipient }}`
    - `{{ $('Workflow Configuration').first().json.escalationEmail }}`
- **Outputs:** To `Generate Sample Events`.
- **Edge cases / failures:**
  - If placeholders remain unresolved, Slack/email notifications will fail (invalid channel ID / invalid email).

**Sticky note context (applies to this block):**
- “## Automated Transaction Monitoring & Dual-Agent Risk Assessment …” (note content mismatches implemented nodes; still describes intended concept).

---

### 2.2 Event Intake (Simulated Sample Events)
**Overview:** Creates a batch of 5 events with different severities and emits them as separate items for downstream AI processing.  
**Nodes involved:** `Generate Sample Events`

#### Node: Generate Sample Events
- **Type / role:** `n8n-nodes-base.code` — generates synthetic dataset.
- **Configuration choices:**
  - JavaScript creates 5 objects (EVT‑001…EVT‑005) with fields:  
    `event_id`, `event_type`, `severity`, `description`, `timestamp`, `source_system`, `metadata{...}`
  - Returns as separate items: `return events.map(event => ({ json: event }));`
- **Outputs:** Each item goes to `Signal Prioritization Agent`.
- **Edge cases / failures:**
  - Code node runtime errors if edited incorrectly.
  - Field naming mismatch downstream: later notification templates expect fields like `eventType` and `sourceSystem`, but generated fields are `event_type` and `source_system` (see Block 2.5 for impact).

---

### 2.3 AI Prioritization (+ Enrichment Tool)
**Overview:** A GPT‑4o agent scores and classifies each event, generating structured fields (`priorityScore`, `priorityLevel`, etc.). The agent can call an enrichment tool (also GPT‑4o) for extra context.  
**Nodes involved:**  
- `Signal Prioritization Agent`  
- `OpenAI Model - Prioritization Agent`  
- `Structured Output - Prioritization`  
- `Enrichment Agent Tool`  
- `OpenAI Model - Enrichment Agent`  
- `Structured Output - Enrichment`

#### Node: OpenAI Model - Prioritization Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for the prioritization agent.
- **Configuration choices:** model `gpt-4o`, temperature `0.2`.
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Connections:** Provides **AI Language Model** input to `Signal Prioritization Agent`.
- **Failures / edge cases:** invalid API key, rate limits, model unavailable, network timeouts.

#### Node: Structured Output - Prioritization
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema.
- **Schema highlights (required):**
  - `priorityScore` number 0–100
  - `priorityLevel` enum: `critical|high|medium|low`
  - `riskAssessment` string
  - `impactAnalysis` string
  - `recommendedActions` array of strings
  - `requiresEscalation` boolean
  - `reasoning` string
- **Connections:** Output parser for `Signal Prioritization Agent`.
- **Failures / edge cases:** LLM output not matching schema → parser error; missing required fields.

#### Node: Enrichment Agent Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.agentTool` — tool callable by the prioritization agent.
- **Key expression / variable:**
  - `text`: `{{ $fromAI("enrichmentQuery", "...", "string") }}`
  - This means the **agent** decides the query string at runtime.
- **System behavior:** Uses its own GPT‑4o model + structured output to return enrichment payload.
- **Connections:**
  - Tool is connected to `Signal Prioritization Agent` as an `ai_tool`.
  - Receives `OpenAI Model - Enrichment Agent` as `ai_languageModel`.
  - Uses `Structured Output - Enrichment` as `ai_outputParser`.
- **Failures / edge cases:** tool call loops or vague queries; schema mismatch; higher latency due to nested LLM call.

#### Node: OpenAI Model - Enrichment Agent
- **Type / role:** LLM provider for the enrichment tool.
- **Configuration choices:** model `gpt-4o`, temperature `0.3`.
- **Failures:** same as other OpenAI nodes.

#### Node: Structured Output - Enrichment
- **Type / role:** Structured schema for enrichment results.
- **Schema fields (not required list specified):**
  - `enrichedData` object
  - `historicalContext` string
  - `relatedIncidents` array
  - `systemHealth` object `{ status, metrics }`
  - `userImpactDetails` object
  - `recommendations` array of strings
- **Edge cases:** permissive schema (many fields optional) but still can fail if tool returns non-JSON.

#### Node: Signal Prioritization Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — core prioritization logic.
- **Prompt input:** `Event Data: {{ JSON.stringify($json) }}`
- **System message (interpreted):**
  - Scores 0–100, maps to levels: critical 90–100, high 70–89, medium 40–69, low 0–39
  - Produces risk/impact/recommended actions
  - May call “Enrichment Agent Tool”
- **Outputs:**
  - Emits the agent result under `output` (typical n8n LangChain agent behavior), e.g. `$json.output.priorityLevel`.
  - Forwards to `Delivery Orchestration Agent`.
- **Edge cases / failures:**
  - Schema enforcement failures.
  - If event fields are missing/renamed, prioritization quality degrades but still typically returns valid output.

**Sticky note context (global/nearby):**
- “Orchestrated Risk Evaluation …” is conceptually related, though the actual orchestration agent is the delivery agent in this implementation.

---

### 2.4 AI Delivery Orchestration (with Slack/Email tools)
**Overview:** A second GPT‑4o agent creates a delivery plan (channels, urgency, content, escalation path). It has access to a Slack tool node and an email tool node, but in this workflow the actual downstream notifications are handled by separate Slack/email nodes based on the Switch routing.  
**Nodes involved:**  
- `Delivery Orchestration Agent`  
- `OpenAI Model - Delivery Agent`  
- `Structured Output - Delivery`  
- `Slack Tool`  
- `Email Tool`

#### Node: OpenAI Model - Delivery Agent
- **Type / role:** OpenAI chat model for delivery orchestration.
- **Configuration:** `gpt-4o`, temperature `0.2`.
- **Connections:** `ai_languageModel` → `Delivery Orchestration Agent`.

#### Node: Structured Output - Delivery
- **Type / role:** Structured output parser for delivery plan.
- **Schema highlights:**
  - `deliveryChannels`: array of `slack|email|both`
  - `notificationUrgency`: `immediate|scheduled|batched`
  - `messageContent`: `{ subject, body }`
  - `recipientGroups`: array of strings
  - `deliveryTiming`: string
  - `escalationPath`: array of strings
- **Connections:** `ai_outputParser` → `Delivery Orchestration Agent`.
- **Edge cases:** parser failures if LLM deviates.

#### Node: Slack Tool
- **Type / role:** `n8n-nodes-base.slackTool` — tool interface callable by the agent (LangChain tool).
- **Configuration choices:**
  - Sends `text` from agent: `{{ $fromAI('slackMessage', ...) }}`
  - Channel ID from agent: `{{ $fromAI('slackChannel', ...) }}`
  - OAuth2 Slack authentication.
- **Connections:** tool is attached to `Delivery Orchestration Agent` as `ai_tool`.
- **Edge cases:** invalid channel ID, Slack OAuth scopes missing, Slack API rate limits.

#### Node: Email Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCode` — custom code tool callable by the agent.
- **Behavior:** **Simulates** sending email; returns a confirmation string instead of actually emailing.
  - Inputs via `$fromAI('to')`, `$fromAI('subject')`, `$fromAI('messageBody')`.
- **Connections:** tool is attached to `Delivery Orchestration Agent` as `ai_tool`.
- **Edge cases:** since it’s simulated, it will not notify real stakeholders; may mislead if assumed operational.

#### Node: Delivery Orchestration Agent
- **Type / role:** LangChain agent for intelligent routing/formatting.
- **Prompt input:** `Prioritized Event Data: {{ JSON.stringify($json) }}`
- **System message (interpreted):**
  - Critical (>=90): Slack + Email, immediate, include full risk/actions
  - High: Slack, immediate
  - Medium/low: scheduled/batched
  - Should use Slack Tool / Email Tool
- **Actual workflow behavior:** Even though tools exist, the main path after this node is:
  - `Route by Priority` (Switch) and `Check Escalation Required` (IF)
- **Edge cases:**
  - If you expect the agent to actually send notifications via tools, note that downstream nodes are the ones sending (Slack/Email send nodes), not the agent tools.

**Sticky note context:** “Orchestrated Risk Evaluation …” and “Priority-Based Response …” apply to this general area.

---

### 2.5 Priority Routing, Storage, Notifications & Audit Logging
**Overview:** Routes items by the AI-assigned `priorityLevel`, stores them in separate Data Tables, sends Slack/email notifications for critical and Slack for high, merges all branches, and writes an audit log entry.  
**Nodes involved:**  
- `Route by Priority`  
- `Store Critical Events`, `Store High Priority Events`, `Store Medium Priority Events`, `Store Low Priority Events`  
- `Notify Critical - Slack`, `Notify Critical - Email`, `Notify High - Slack`  
- `Merge All Notifications`  
- `Audit Log`

#### Node: Route by Priority
- **Type / role:** `n8n-nodes-base.switch` — branch by priority level.
- **Rules:** checks `{{ $json.output.priorityLevel }}` equals `critical|high|medium|low`.
- **Outputs:** Four named outputs: Critical/High/Medium/Low; fallback “Unclassified”.
- **Connections:** Each output goes to its corresponding “Store … Events” node.
- **Edge cases:**
  - If `$json.output.priorityLevel` missing or unexpected → goes to “Unclassified” (but no nodes are connected to fallback in this workflow, so items may be dropped).

#### Nodes: Store Critical/High/Medium/Low Priority Events
- **Type / role:** `n8n-nodes-base.dataTable` — writes each item into an n8n Data Table.
- **Tables used:**
  - `critical_events`
  - `high_priority_events`
  - `medium_priority_events`
  - `low_priority_events`
- **Mapping:** `autoMapInputData` (stores all incoming fields/structure).
- **Connections:**
  - Critical → `Notify Critical - Slack`
  - High → `Notify High - Slack`
  - Medium → `Merge All Notifications` (input index 2)
  - Low → `Merge All Notifications` (input index 3)
- **Edge cases:**
  - Data Table not created / permissions → node fails.
  - Schema drift: storing nested objects is allowed, but querying later may be harder.

#### Node: Notify Critical - Slack
- **Type / role:** `n8n-nodes-base.slack` — sends Slack message for critical items.
- **Channel:** `{{ $('Workflow Configuration').first().json.criticalSlackChannel }}`
- **Message template:** includes priority score, event type, source, description, risk, impact, actions.
- **Important field mismatch risk:**
  - Template references `{{ $json.eventType }}` and `{{ $json.sourceSystem }}`, but sample events use `event_type` and `source_system`.
  - Result: Slack messages may show blank/undefined for those fields unless you add a mapping step or change template.
- **Connection:** then triggers `Notify Critical - Email`.
- **Failures:** Slack OAuth scope issues, invalid channel, rate limiting.

#### Node: Notify Critical - Email
- **Type / role:** `n8n-nodes-base.emailSend` — sends HTML email for critical items.
- **To:** `{{ $('Workflow Configuration').first().json.criticalEmailRecipient }}`
- **From:** placeholder `<__PLACEHOLDER_VALUE__Sender Email Address__>`
- **Subject:** `CRITICAL: {{ $json.eventType }} - Priority {{ $json.output.priorityScore }}/100`
- **Body:** HTML including risk/impact/actions; same field mismatch issue (`eventType`, `sourceSystem`).
- **Connection:** to `Merge All Notifications` (input index 0).
- **Failures:** SMTP configuration/credentials missing, invalid from address, blocked relay.

#### Node: Notify High - Slack
- **Type / role:** `n8n-nodes-base.slack` — sends Slack message for high items.
- **Channel:** `{{ $('Workflow Configuration').first().json.highPrioritySlackChannel }}`
- **Template:** similar mismatch risk (`eventType`, `sourceSystem`).
- **Connection:** to `Merge All Notifications` (input index 1).

#### Node: Merge All Notifications
- **Type / role:** `n8n-nodes-base.merge` — combines the four priority branches.
- **Mode:** `combine`, `combineByPosition`, `numberInputs=4`.
- **Connections:** output goes to `Audit Log`.
- **Edge cases (important):**
  - If one branch produces fewer items than another, combine-by-position can yield unexpected pairing or item loss.
  - For event streams where only some priorities occur, merge may behave unintuitively. A different merge strategy (append) is often safer for logging.

#### Node: Audit Log
- **Type / role:** `n8n-nodes-base.dataTable` — stores merged results into `audit_log`.
- **Mapping:** `autoMapInputData`.
- **Edge cases:** same Data Table availability issues; merged structure may be complex.

**Sticky note context:** “Priority-Based Response …” applies to this whole block.

---

### 2.6 Escalation Path
**Overview:** If an item’s AI output indicates escalation is required, an escalation agent produces executive-ready structured details and sends an escalation email.  
**Nodes involved:**  
`Check Escalation Required`, `Escalation Agent`, `OpenAI Model - Escalation Agent`, `Structured Output - Escalation`, `Escalation Email`

#### Node: Check Escalation Required
- **Type / role:** `n8n-nodes-base.if` — gates escalation.
- **Condition:** `{{ $json.output.requiresEscalation }} == true`
- **Connections:** True branch → `Escalation Agent`. (False branch not connected.)
- **Edge cases:** if `output.requiresEscalation` missing → treated as false; escalation won’t happen.

#### Node: OpenAI Model - Escalation Agent
- **Type / role:** OpenAI chat model for escalation agent.
- **Config:** `gpt-4o`, temperature `0.2`.
- **Connections:** `ai_languageModel` → `Escalation Agent`.

#### Node: Structured Output - Escalation
- **Type / role:** Structured schema for escalation output.
- **Schema fields:**
  - `escalationLevel`: `executive|senior_management|technical_lead`
  - `escalationReason`, `executiveSummary`, `businessImpact` (strings)
  - `urgentActions` array
  - `stakeholders` array
  - `estimatedResolutionTime` string
- **Connections:** `ai_outputParser` → `Escalation Agent`.

#### Node: Escalation Agent
- **Type / role:** LangChain agent that creates executive escalation content.
- **Prompt input references (risk):**
  - Uses fields like `{{ $json.eventType }}`, `{{ $json.priority }}`, `{{ $json.impact }}`, etc.
  - These fields are **not produced** by the sample events or earlier agents as-is (sample uses `event_type`, and priority is in `$json.output.priorityScore/priorityLevel`).
  - You will likely get blank sections unless you normalize fields before escalation.
- **Connection:** to `Escalation Email`.
- **Failures:** schema mismatch; missing expected input context.

#### Node: Escalation Email
- **Type / role:** `n8n-nodes-base.emailSend` — sends escalation HTML email.
- **To:** `{{ $('Workflow Configuration').first().json.escalationEmail }}`
- **From:** placeholder sender
- **Subject:** `ESCALATION REQUIRED: {{ $json.output.escalationLevel }} - {{ $json.eventType }}`
- **Body:** HTML populated from escalation agent output; includes `eventType` again (field mismatch risk).
- **Failures:** SMTP/credential issues as with other email send.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Time-based workflow entrypoint | — | Workflow Configuration | ## Automated Transaction Monitoring & Dual-Agent Risk Assessment  \n**What**: Price Reasonableness Agent validates pricing; Delivery Agent evaluates supplier performance metrics  \n**Why**: Parallel expert analysis identifies both pricing fraud and fulfillment violations across procurement lifecycle |
| Workflow Configuration | set | Central config variables (channels/emails/thresholds) | Schedule Trigger | Generate Sample Events | ## Automated Transaction Monitoring & Dual-Agent Risk Assessment  \n**What**: Price Reasonableness Agent validates pricing; Delivery Agent evaluates supplier performance metrics  \n**Why**: Parallel expert analysis identifies both pricing fraud and fulfillment violations across procurement lifecycle |
| Generate Sample Events | code | Simulated event generator (5 items) | Workflow Configuration | Signal Prioritization Agent | ## Automated Transaction Monitoring & Dual-Agent Risk Assessment  \n**What**: Price Reasonableness Agent validates pricing; Delivery Agent evaluates supplier performance metrics  \n**Why**: Parallel expert analysis identifies both pricing fraud and fulfillment violations across procurement lifecycle |
| Signal Prioritization Agent | langchain.agent | AI scoring/classification of events | Generate Sample Events | Delivery Orchestration Agent | ## Automated Transaction Monitoring & Dual-Agent Risk Assessment  \n**What**: Price Reasonableness Agent validates pricing; Delivery Agent evaluates supplier performance metrics  \n**Why**: Parallel expert analysis identifies both pricing fraud and fulfillment violations across procurement lifecycle |
| OpenAI Model - Prioritization Agent | lmChatOpenAi | LLM backend for prioritization | — | Signal Prioritization Agent (ai_languageModel) |  |
| Structured Output - Prioritization | outputParserStructured | Enforce prioritization schema | — | Signal Prioritization Agent (ai_outputParser) |  |
| Enrichment Agent Tool | agentTool | Tool callable by prioritization agent | — | Signal Prioritization Agent (ai_tool) |  |
| OpenAI Model - Enrichment Agent | lmChatOpenAi | LLM backend for enrichment tool | — | Enrichment Agent Tool (ai_languageModel) |  |
| Structured Output - Enrichment | outputParserStructured | Enforce enrichment schema | — | Enrichment Agent Tool (ai_outputParser) |  |
| Delivery Orchestration Agent | langchain.agent | AI delivery planning + tool access | Signal Prioritization Agent | Route by Priority; Check Escalation Required | ## Orchestrated Risk Evaluation  \n**What**: Orchestration Agent synthesizes findings, applies risk scoring, generates prioritized recommendations  \n**Why**: Unified assessment enables clear fraud/compliance determination and appropriate escalation decisions |
| OpenAI Model - Delivery Agent | lmChatOpenAi | LLM backend for delivery agent | — | Delivery Orchestration Agent (ai_languageModel) | ## Orchestrated Risk Evaluation  \n**What**: Orchestration Agent synthesizes findings, applies risk scoring, generates prioritized recommendations  \n**Why**: Unified assessment enables clear fraud/compliance determination and appropriate escalation decisions |
| Structured Output - Delivery | outputParserStructured | Enforce delivery plan schema | — | Delivery Orchestration Agent (ai_outputParser) | ## Orchestrated Risk Evaluation  \n**What**: Orchestration Agent synthesizes findings, applies risk scoring, generates prioritized recommendations  \n**Why**: Unified assessment enables clear fraud/compliance determination and appropriate escalation decisions |
| Slack Tool | slackTool | Agent-callable Slack sending tool | — | Delivery Orchestration Agent (ai_tool) | ## Orchestrated Risk Evaluation  \n**What**: Orchestration Agent synthesizes findings, applies risk scoring, generates prioritized recommendations  \n**Why**: Unified assessment enables clear fraud/compliance determination and appropriate escalation decisions |
| Email Tool | toolCode | Agent-callable email tool (simulated) | — | Delivery Orchestration Agent (ai_tool) | ## Orchestrated Risk Evaluation  \n**What**: Orchestration Agent synthesizes findings, applies risk scoring, generates prioritized recommendations  \n**Why**: Unified assessment enables clear fraud/compliance determination and appropriate escalation decisions |
| Route by Priority | switch | Branching by `$json.output.priorityLevel` | Delivery Orchestration Agent | Store Critical Events; Store High Priority Events; Store Medium Priority Events; Store Low Priority Events | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Store Critical Events | dataTable | Persist critical items | Route by Priority (Critical) | Notify Critical - Slack | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Notify Critical - Slack | slack | Slack alert for critical | Store Critical Events | Notify Critical - Email | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Notify Critical - Email | emailSend | Email alert for critical | Notify Critical - Slack | Merge All Notifications | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Store High Priority Events | dataTable | Persist high-priority items | Route by Priority (High) | Notify High - Slack | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Notify High - Slack | slack | Slack alert for high | Store High Priority Events | Merge All Notifications | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Store Medium Priority Events | dataTable | Persist medium items | Route by Priority (Medium) | Merge All Notifications | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Store Low Priority Events | dataTable | Persist low items | Route by Priority (Low) | Merge All Notifications | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Merge All Notifications | merge | Combine all branches for unified logging | Notify Critical - Email; Notify High - Slack; Store Medium Priority Events; Store Low Priority Events | Audit Log | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Audit Log | dataTable | Persist final audit trail | Merge All Notifications | — | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Check Escalation Required | if | Gate escalation path | Delivery Orchestration Agent | Escalation Agent | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Escalation Agent | langchain.agent | Generate exec escalation package | Check Escalation Required (true) | Escalation Email | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| OpenAI Model - Escalation Agent | lmChatOpenAi | LLM backend for escalation | — | Escalation Agent (ai_languageModel) | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Structured Output - Escalation | outputParserStructured | Enforce escalation schema | — | Escalation Agent (ai_outputParser) | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Escalation Email | emailSend | Send escalation email | Escalation Agent | — | ## Priority-Based Response  \n**What**: Routes by severity—critical triggers immediate alerts and audit logs, lower priorities enable planned reviews  \n**Why**: Risk-stratified workflows ensure urgent fraud receives instant attention while maintaining comprehensive documentation |
| Sticky Note | stickyNote | Documentation (global) | — | — | ## How It Works… |
| Sticky Note1 | stickyNote | Documentation (global) | — | — | ## Setup Steps… |
| Sticky Note2 | stickyNote | Documentation (global) | — | — | ## Prerequisites… |
| Sticky Note4 | stickyNote | Documentation (block note) | — | — | ## Orchestrated Risk Evaluation… |
| Sticky Note5 | stickyNote | Documentation (block note) | — | — | ## Priority-Based Response… |
| Sticky Note6 | stickyNote | Documentation (block note) | — | — | ## Automated Transaction Monitoring & Dual-Agent Risk Assessment… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
1. Add **Schedule Trigger**.
2. Set interval to **Every 15 minutes**.

2) **Add configuration Set node**
3. Add **Set** node named `Workflow Configuration`.
4. Add fields:
   - `criticalSlackChannel` (string) → your Slack channel ID for critical alerts
   - `highPrioritySlackChannel` (string) → your Slack channel ID for high alerts
   - `escalationEmail` (string) → escalation distribution email
   - `criticalEmailRecipient` (string) → critical alerts email
   - `priorityThresholdCritical` (number) = 90
   - `priorityThresholdHigh` (number) = 70
   - `priorityThresholdMedium` (number) = 40
5. Enable “Include other fields”.
6. Connect: **Schedule Trigger → Workflow Configuration**.

3) **Create an event source (simulated or real)**
7. Add **Code** node named `Generate Sample Events`.
8. Paste JS that returns multiple items (or replace with your real procurement data ingestion).
9. Connect: **Workflow Configuration → Generate Sample Events**.

4) **Add Prioritization AI agent**
10. Add **OpenAI Chat Model** node named `OpenAI Model - Prioritization Agent`.
    - Model: `gpt-4o`, Temperature: `0.2`
    - Credentials: configure **OpenAI API** credential in n8n.
11. Add **Structured Output Parser** named `Structured Output - Prioritization` with the schema for:
    - `priorityScore`, `priorityLevel`, `riskAssessment`, `impactAnalysis`, `recommendedActions`, `requiresEscalation`, `reasoning`
12. Add **AI Agent** node named `Signal Prioritization Agent`.
    - Text: `Event Data: {{ JSON.stringify($json) }}`
    - System message: include your scoring rubric and escalation rules
    - Enable “Has Output Parser”
13. Connect:
    - **Generate Sample Events → Signal Prioritization Agent**
    - `OpenAI Model - Prioritization Agent` → `Signal Prioritization Agent` (AI Language Model connection)
    - `Structured Output - Prioritization` → `Signal Prioritization Agent` (AI Output Parser connection)

5) **Add Enrichment tool (optional but included)**
14. Add **OpenAI Chat Model** node `OpenAI Model - Enrichment Agent` (gpt‑4o, temp 0.3).
15. Add **Structured Output Parser** node `Structured Output - Enrichment` (enrichment schema).
16. Add **Agent Tool** node `Enrichment Agent Tool`.
    - Text uses `$fromAI("enrichmentQuery", ...)`
    - Add system message describing what enrichment returns
    - Enable output parser
17. Connect:
    - `OpenAI Model - Enrichment Agent` → `Enrichment Agent Tool` (AI Language Model)
    - `Structured Output - Enrichment` → `Enrichment Agent Tool` (AI Output Parser)
    - `Enrichment Agent Tool` → `Signal Prioritization Agent` (AI Tool)

6) **Add Delivery Orchestration AI agent + tools**
18. Add **OpenAI Chat Model** `OpenAI Model - Delivery Agent` (gpt‑4o, temp 0.2).
19. Add **Structured Output Parser** `Structured Output - Delivery` (delivery plan schema).
20. Add **Slack Tool** node (LangChain tool):
    - OAuth2 Slack credential (configure Slack OAuth2 in n8n)
    - Text/channel from `$fromAI(...)`
21. Add **Tool (Code)** node `Email Tool`:
    - Keep as simulation or replace with real email node/tool
22. Add **AI Agent** `Delivery Orchestration Agent`:
    - Text: `Prioritized Event Data: {{ JSON.stringify($json) }}`
    - System message defining channel/urgency rules
    - Enable output parser
23. Connect:
    - **Signal Prioritization Agent → Delivery Orchestration Agent**
    - `OpenAI Model - Delivery Agent` → `Delivery Orchestration Agent` (AI Language Model)
    - `Structured Output - Delivery` → `Delivery Orchestration Agent` (AI Output Parser)
    - `Slack Tool` → `Delivery Orchestration Agent` (AI Tool)
    - `Email Tool` → `Delivery Orchestration Agent` (AI Tool)

7) **Route by priority**
24. Add **Switch** node `Route by Priority`.
25. Create 4 rules on `{{ $json.output.priorityLevel }}` equals: `critical`, `high`, `medium`, `low`.
26. Connect: **Delivery Orchestration Agent → Route by Priority**.

8) **Create Data Tables and storage nodes**
27. In n8n, create Data Tables (or let nodes target existing):  
    `critical_events`, `high_priority_events`, `medium_priority_events`, `low_priority_events`, `audit_log`
28. Add 4 **Data Table** nodes:
    - `Store Critical Events` → table `critical_events`
    - `Store High Priority Events` → table `high_priority_events`
    - `Store Medium Priority Events` → table `medium_priority_events`
    - `Store Low Priority Events` → table `low_priority_events`
    - Use auto-map input
29. Connect each switch output to the matching store node.

9) **Notifications**
30. Add **Slack** node `Notify Critical - Slack` (OAuth2).
    - Channel: `{{ $('Workflow Configuration').first().json.criticalSlackChannel }}`
    - Message template using `$json.output.*` fields.
31. Add **Email Send** node `Notify Critical - Email`.
    - SMTP/Email credentials: configure in n8n (or your provider)
    - To: `{{ $('Workflow Configuration').first().json.criticalEmailRecipient }}`
    - From: set a real sender address
32. Connect: `Store Critical Events → Notify Critical - Slack → Notify Critical - Email`.
33. Add **Slack** node `Notify High - Slack`.
    - Channel: `{{ $('Workflow Configuration').first().json.highPrioritySlackChannel }}`
34. Connect: `Store High Priority Events → Notify High - Slack`.

10) **Merge & audit log**
35. Add **Merge** node `Merge All Notifications`.
    - Mode: Combine
    - Combine by: Position
    - Inputs: 4
36. Connect:
    - `Notify Critical - Email` → Merge (input 0)
    - `Notify High - Slack` → Merge (input 1)
    - `Store Medium Priority Events` → Merge (input 2)
    - `Store Low Priority Events` → Merge (input 3)
37. Add **Data Table** node `Audit Log` → table `audit_log`.
38. Connect: **Merge All Notifications → Audit Log**.

11) **Escalation path**
39. Add **IF** node `Check Escalation Required` with condition:
    - Boolean equals `true` on `{{ $json.output.requiresEscalation }}`
40. Connect: **Delivery Orchestration Agent → Check Escalation Required** (parallel to switch, like in the JSON).
41. Add **OpenAI Chat Model** `OpenAI Model - Escalation Agent` (gpt‑4o, temp 0.2).
42. Add **Structured Output Parser** `Structured Output - Escalation` (schema fields listed above).
43. Add **AI Agent** `Escalation Agent` with an executive-oriented system message.
44. Connect:
    - IF true → `Escalation Agent`
    - `OpenAI Model - Escalation Agent` → `Escalation Agent` (AI Language Model)
    - `Structured Output - Escalation` → `Escalation Agent` (AI Output Parser)
45. Add **Email Send** node `Escalation Email`.
    - To: `{{ $('Workflow Configuration').first().json.escalationEmail }}`
    - From: real sender address
46. Connect: **Escalation Agent → Escalation Email**.

**Strongly recommended adjustment when rebuilding:** add a **Set** (or **Rename Keys**) node after event intake (or after prioritization) to normalize fields (`eventType` ↔ `event_type`, `sourceSystem` ↔ `source_system`) so Slack/email templates and escalation prompts render correctly.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## How It Works … automates procurement fraud detection and supplier compliance monitoring …” | Sticky note describing intended business framing (procurement), while current sample data is IT event monitoring. |
| “## Setup Steps … connect Schedule Trigger … configure procurement systems … AI model API keys … Slack webhooks … email credentials …” | Sticky note with setup guidance. |
| “## Prerequisites … Procurement system API access … market pricing databases …” | Sticky note listing prerequisites/use cases/benefits. |
| “## Orchestrated Risk Evaluation … Orchestration Agent synthesizes findings …” | Sticky note describing orchestration concept; implemented as Delivery Orchestration Agent + prioritization agent. |
| “## Priority-Based Response … Routes by severity … maintaining comprehensive documentation” | Sticky note describing the routing/storage/alerting/audit approach. |
| “## Automated Transaction Monitoring & Dual-Agent Risk Assessment …” | Sticky note describing dual-agent concept; current implementation uses prioritization + delivery agents, with enrichment tool. |