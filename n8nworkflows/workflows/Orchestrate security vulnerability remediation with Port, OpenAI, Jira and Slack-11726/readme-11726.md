Orchestrate security vulnerability remediation with Port, OpenAI, Jira and Slack

https://n8nworkflows.xyz/workflows/orchestrate-security-vulnerability-remediation-with-port--openai--jira-and-slack-11726


# Orchestrate security vulnerability remediation with Port, OpenAI, Jira and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Orchestrate security vulnerability remediation with Port, OpenAI, Jira and Slack  
**Workflow name (JSON):** Remediate security vulnerabilities with n8n and Port  
**Purpose:** End-to-end automation for handling newly detected security vulnerabilities: receive an alert, enrich it with service ownership/SLA context from Port, generate an AI remediation plan, create a Jira issue based on severity, optionally trigger an automated fix action in Port (Claude Code), and notify the responsible team via Slack.

### Logical blocks
1.1 **Intake (Webhook reception)**  
Receives a vulnerability event from an external scanner.

1.2 **Context enrichment (Port catalog / agent)**  
Retrieves service metadata (owners, environment, SLA, Slack channel, Jira project key, etc.) for routing and messaging.

1.3 **AI analysis (OpenAI remediation plan)**  
Transforms vulnerability + context into a structured remediation plan (JSON) including auto-fix feasibility.

1.4 **Severity routing & ticketing (Switch + Jira)**  
Routes by severity (Critical / High / Other) and creates the appropriate Jira ticket with rich description.

1.5 **Critical remediation execution (Auto-fix vs manual) + Notifications (Slack)**  
For Critical only: decide if auto-fix is possible; if yes trigger a Port action (Claude Code) then notify Slack; otherwise notify Slack that manual remediation is required.  
For High: notify Slack after ticket creation.

---

## 2. Block-by-Block Analysis

### 2.1 Intake (Webhook reception)
**Overview:** Accepts incoming vulnerability alerts via HTTP POST and exposes fields used across the workflow.  
**Nodes involved:** Webhook Trigger

#### Node: Webhook Trigger
- **Type / role:** `n8n-nodes-base.webhook` — entry point receiving external events.
- **Key configuration (interpreted):**
  - **Method:** POST
  - **Path:** `security/vulnerability`
  - **Description:** “Triggered when a new vulnerability is detected”
- **Key fields expected in request body (used later):**
  - `body.vulnerability_id`, `body.description`, `body.severity`, optional `body.cve`, optional `body.package`
- **Connections:**
  - **Output →** Get Context From Port
- **Edge cases / failures:**
  - Missing `severity` can break `.toLowerCase()` later (Switch node). Add a default/guard if scanner payload is inconsistent.
  - Very short/empty `description` can affect substring usage (safe in JS, but may become empty summary).
  - If scanner sends severity values not matching critical/high, routing falls into “other”.

---

### 2.2 Context enrichment (Port catalog / agent)
**Overview:** Uses a Port AI agent to enrich the vulnerability with service ownership, environment, SLA, Slack channel and Jira project key.  
**Nodes involved:** Get Context From Port

#### Node: Get Context From Port
- **Type / role:** `CUSTOM.portApiAi` — custom Port AI integration node to invoke an agent.
- **Key configuration choices:**
  - **Operation:** invokeAgent
  - **Agent identifier:** `context_retriever_agent`
  - **Prompt:** Builds a vulnerability summary from the webhook payload, instructs the agent to query Port catalog, and return **raw JSON only** matching a required schema:
    - `service_name`, `repository`, `service_tier`, `sla_hours`, `owners[]`, `slack_channel`, `environment`, `dependencies[]`, `jira_project_key`
- **Key expressions / variables:**
  - Uses webhook values like `{{ $('Webhook Trigger').item.json.body.vulnerability_id }}`
  - Uses fallback for optional fields: `cve || 'N/A'`, `package || 'N/A'`
- **Connections:**
  - **Input ←** Webhook Trigger
  - **Output →** OpenAI Remediation Plan
- **Output handling note (important):**
  - Later nodes read `$('Get Context From Port').item.json.executionMessage`, and attempt `JSON.parse()` if it’s a string. This implies the custom node returns the JSON text inside `executionMessage` rather than directly as structured JSON.
- **Edge cases / failures:**
  - Port agent may return non-JSON or JSON with extra text; downstream expressions handle parsing via try/catch, but you’ll lose enrichment (“Context unavailable”).
  - Agent might omit fields; downstream uses fallbacks (e.g., SLA defaults to 24 hours, Slack defaults to `#security-alerts`).
  - Authentication/authorization errors depend on the custom node’s credential setup (not shown in JSON).

---

### 2.3 AI analysis (OpenAI remediation plan)
**Overview:** Calls OpenAI to generate a structured remediation plan (JSON only) combining vulnerability details and Port context.  
**Nodes involved:** OpenAI Remediation Plan

#### Node: OpenAI Remediation Plan
- **Type / role:** `n8n-nodes-base.openAi` — Chat completion to produce remediation plan.
- **Key configuration choices:**
  - **Resource:** chat
  - **Model:** `gpt-4o-mini`
  - **Prompt content:**  
    - Injects vulnerability fields from webhook  
    - Injects “Service Context from Port” using `$('Get Context From Port').item.json.executionMessage`
    - Enforces **single valid JSON object only** with schema:
      - `summary`, `impact`, `remediation_steps[]`, `is_auto_fixable` (boolean), `fix_prompt`, `estimated_effort (low|medium|high)`
- **Connections:**
  - **Input ←** Get Context From Port
  - **Output →** Check Severity Level
- **Edge cases / failures:**
  - OpenAI may still return fenced code blocks; downstream expressions strip ```json fences before parsing.
  - Model may return invalid JSON (trailing commas, comments). Downstream try/catch will degrade to “Analysis pending/Steps pending”.
  - Credential/quotas/timeouts typical to OpenAI node.

---

### 2.4 Severity routing & Jira ticketing
**Overview:** Routes execution based on severity and creates a Jira issue with vulnerability details plus Port and AI context.  
**Nodes involved:** Check Severity Level, Create Critical Jira Ticket, Create High Jira Ticket, Create Medium/Low Jira Ticket

#### Node: Check Severity Level
- **Type / role:** `n8n-nodes-base.switch` — branching by severity.
- **Key configuration choices:**
  - **Value to evaluate:** `{{ $('Webhook Trigger').item.json.body.severity.toLowerCase() }}`
  - **Rules (in order):**
    1. equals `critical`
    2. equals `high`
    3. regex `.*` (catch-all)
  - **Fallback output:** index 2 (the “other” branch)
- **Connections:**
  - **Input ←** OpenAI Remediation Plan
  - **Outputs →**
    - Output 0 → Create Critical Jira Ticket
    - Output 1 → Create High Jira Ticket
    - Output 2 → Create Medium/Low Jira Ticket
- **Edge cases / failures:**
  - If `severity` is null/undefined, `.toLowerCase()` throws and the node fails. Consider: `String(... || '').toLowerCase()`.

#### Node: Create Critical Jira Ticket
- **Type / role:** `n8n-nodes-base.jira` — creates Critical issue.
- **Key configuration choices:**
  - **Project:** “Port” (internal Jira project id `10000` selected from list)
  - **Issue type:** Task (`10002`)
  - **Summary:** `[CRITICAL] <vulnerability_id>: <description first 100 chars>`
  - **Labels:** `security`
  - **Priority:** “Now (Urgent)” (`10001`)
  - **Description:** Rich text including:
    - Vulnerability details
    - “Affected Service” section parsed from Port context (`executionMessage` → JSON.parse)
    - “AI Remediation Plan” / Impact / Steps parsed from OpenAI output (`message.content` → strip fences → JSON.parse)
    - SLA hours from Port context, default 24
- **Connections:**
  - **Input ←** Check Severity Level (critical branch)
  - **Output →** Is Auto-Fixable?
- **Edge cases / failures:**
  - Jira auth, permissions, field configuration mismatches (priority ids differ per Jira instance).
  - Parsing failures fall back to “Context unavailable”, “Analysis pending”, etc.
  - The Jira project key from Port is retrieved but **not used**; project is hardcoded to “Port”.

#### Node: Create High Jira Ticket
- **Type / role:** `n8n-nodes-base.jira` — creates High severity issue.
- **Key configuration choices:**
  - **Project:** “Port” (`10000`)
  - **Issue type:** Task (`10002`)
  - **Priority:** “High” (`2`)
  - **Description:** Similar to critical, but typically less content (no explicit Impact section here).
- **Connections:**
  - **Input ←** Check Severity Level (high branch)
  - **Output →** Alert High to Slack
- **Edge cases / failures:** Same categories as critical (auth/fields), plus priority id `2` may not exist in some Jira configurations.

#### Node: Create Medium/Low Jira Ticket
- **Type / role:** `n8n-nodes-base.jira` — creates issue for all other severities (medium/low/etc.).
- **Key configuration choices:**
  - **Priority:** “Low” (`4`)
  - **Summary:** Uses dynamic severity `[SEVERITY]` uppercased
  - **Description:** Includes vulnerability + basic affected service + remediation steps.
- **Connections:**
  - **Input ←** Check Severity Level (fallback/other branch)
  - **Output:** none (workflow ends on this branch)
- **Edge cases / failures:**
  - No Slack notification for medium/low in current design (intentional or missing).
  - Same Jira field/id concerns as above.

---

### 2.5 Critical remediation execution (Auto-fix vs manual) + Slack notifications
**Overview:** For Critical tickets only, decide whether the AI suggests an automated fix; if yes, trigger a Port action to create a PR, then notify Slack; otherwise notify Slack to handle manually. High severity always triggers a Slack alert after Jira creation.  
**Nodes involved:** Is Auto-Fixable?, Trigger Fix via Port AI Agent, Alert Critical to Slack, Alert Critical (Manual Fix), Alert High to Slack

#### Node: Is Auto-Fixable?
- **Type / role:** `n8n-nodes-base.if` — boolean condition gate.
- **Key configuration choices:**
  - Condition computes:
    - Parse OpenAI `message.content` as JSON (stripping code fences if present)
    - Returns true if `is_auto_fixable === true` OR `fix_prompt` exists and length > 10
  - Compares result to `true`
- **Connections:**
  - **Input ←** Create Critical Jira Ticket
  - **True output →** Trigger Fix via Port AI Agent
  - **False output →** Alert Critical (Manual Fix)
- **Edge cases / failures:**
  - Invalid JSON from OpenAI results in false (manual path).
  - If OpenAI node output format changes (e.g., different property path than `message.content`), condition will always fall back to false.

#### Node: Trigger Fix via Port AI Agent
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Port Actions API to trigger Claude Code run.
- **Key configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.getport.io/v1/actions/run_claude_code/runs`
  - **Auth:** HTTP Bearer (`genericCredentialType` with `httpBearerAuth`)
  - **Body:** JSON containing `properties.service` and `properties.prompt`
    - `service`: parsed from Port context (`repository`), fallback `default-organization/repository`
    - `prompt`: built from OpenAI `fix_prompt` (or summary) + vulnerability id + Jira reference
      - Jira key is taken from `Create Critical Jira Ticket` (`item.json.key`)
- **Connections:**
  - **Input ←** Is Auto-Fixable? (true)
  - **Output →** Alert Critical to Slack
- **Edge cases / failures:**
  - Bearer token invalid/expired → 401
  - Port action name `run_claude_code` must exist and be enabled in Port
  - API may require additional headers (e.g., `Content-Type: application/json` is usually handled by node)
  - If Jira key isn’t available or differs in Jira response shape, prompt references `N/A`

#### Node: Alert Critical to Slack
- **Type / role:** `n8n-nodes-base.slack` — posts a Slack message for critical vulnerabilities where auto-fix was triggered.
- **Key configuration choices:**
  - **Channel:** from Port context `slack_channel`, fallback `#security-alerts`
  - **Text:** includes vulnerability, service, environment, owners, AI summary/impact, Jira key, SLA hours
- **Connections:**
  - **Input ←** Trigger Fix via Port AI Agent
  - **Output:** none
- **Edge cases / failures:**
  - Slack channel may be invalid or private without bot access.
  - Uses “🚨/🔴” characters; harmless but can be removed if your Slack policy prefers plain text.
  - If Jira creation fails earlier, this node is never reached (because it depends on that branch).

#### Node: Alert Critical (Manual Fix)
- **Type / role:** `n8n-nodes-base.slack` — posts Slack message when auto-fix is not possible.
- **Key configuration choices:**
  - Same dynamic channel logic as above
  - Text includes remediation steps and states manual remediation required
- **Connections:**
  - **Input ←** Is Auto-Fixable? (false)
  - **Output:** none
- **Edge cases / failures:** Same as other Slack node.

#### Node: Alert High to Slack
- **Type / role:** `n8n-nodes-base.slack` — posts Slack message for High severity.
- **Key configuration choices:**
  - Channel from Port context (fallback `#security-alerts`)
  - Text includes AI summary and Jira key from “Create High Jira Ticket”
- **Connections:**
  - **Input ←** Create High Jira Ticket
  - **Output:** none
- **Edge cases / failures:** Same as other Slack node.

---

### 2.6 Documentation notes (Sticky Notes)
**Overview:** In-workflow annotations describing prerequisites and the logical phases.  
**Nodes involved:** Sticky Note, Sticky Note1, Sticky Note4, Sticky Note2

#### Node: Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — canvas documentation.
- **Content highlights:** overall description, steps, prerequisites (Port agent, Claude Code action, Jira, Slack, OpenAI key).

#### Node: Sticky Note1
- **Role:** documents “1. Trigger & enrichment”.

#### Node: Sticky Note4
- **Role:** documents “2. AI analysis”.

#### Node: Sticky Note2
- **Role:** documents severity routing and Jira ticket behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | n8n-nodes-base.webhook | Receives vulnerability events (POST webhook) | — | Get Context From Port | ## Remediate security vulnerabilities with n8n and Port / This workflow automates security vulnerability management from detection to remediation. / Prerequisites list |
| Get Context From Port | CUSTOM.portApiAi | Enrich vulnerability with Port catalog context via AI agent | Webhook Trigger | OpenAI Remediation Plan | ### 1. Trigger & enrichment / Queries Port's catalog to enrich the vulnerability… |
| OpenAI Remediation Plan | n8n-nodes-base.openAi | Generate structured remediation plan JSON | Get Context From Port | Check Severity Level | ### 2. AI analysis / Analyzes the vulnerability and generates… |
| Check Severity Level | n8n-nodes-base.switch | Route flow based on severity | OpenAI Remediation Plan | Create Critical Jira Ticket; Create High Jira Ticket; Create Medium/Low Jira Ticket | ### Severity routing & Jira tickets / Routes to the appropriate path based on severity… |
| Create Critical Jira Ticket | n8n-nodes-base.jira | Create critical Jira issue with urgent priority | Check Severity Level | Is Auto-Fixable? | ### Severity routing & Jira tickets / Creates tickets with full context… |
| Create High Jira Ticket | n8n-nodes-base.jira | Create high severity Jira issue | Check Severity Level | Alert High to Slack | ### Severity routing & Jira tickets / Creates tickets with full context… |
| Create Medium/Low Jira Ticket | n8n-nodes-base.jira | Create medium/low Jira issue (fallback path) | Check Severity Level | — | ### Severity routing & Jira tickets / Creates tickets with full context… |
| Is Auto-Fixable? | n8n-nodes-base.if | Decide whether to trigger automated fix for critical items | Create Critical Jira Ticket | Trigger Fix via Port AI Agent; Alert Critical (Manual Fix) |  |
| Trigger Fix via Port AI Agent | n8n-nodes-base.httpRequest | Call Port Actions API to run Claude Code and create a fix PR | Is Auto-Fixable? (true) | Alert Critical to Slack | ## Remediate security vulnerabilities… / “For critical issues, Claude Code can create a fix PR” |
| Alert Critical to Slack | n8n-nodes-base.slack | Notify Slack: critical issue + auto-fix triggered | Trigger Fix via Port AI Agent | — | ## Remediate security vulnerabilities… / “Team is notified via Slack” |
| Alert Critical (Manual Fix) | n8n-nodes-base.slack | Notify Slack: critical issue needs manual remediation | Is Auto-Fixable? (false) | — | ## Remediate security vulnerabilities… / “Team is notified via Slack” |
| Alert High to Slack | n8n-nodes-base.slack | Notify Slack: high severity created in Jira | Create High Jira Ticket | — | ## Remediate security vulnerabilities… / “Team is notified via Slack” |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation (overview + prerequisites) | — | — | ## Remediate security vulnerabilities with n8n and Port (full content) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation (Trigger & enrichment) | — | — | ### 1. Trigger & enrichment (full content) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation (AI analysis) | — | — | ### 2. AI analysis (full content) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation (Severity routing & Jira tickets) | — | — | ### Severity routing & Jira tickets (full content) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: “Remediate security vulnerabilities with n8n and Port” (or your preferred title).

2) **Add node: Webhook Trigger**
- Node type: **Webhook**
- Method: **POST**
- Path: `security/vulnerability`
- Save and copy the production webhook URL.
- Ensure your scanner sends JSON body containing at least:
  - `vulnerability_id`, `description`, `severity` (+ optional `cve`, `package`)

3) **Add node: Get Context From Port**
- Node type: **Port AI / custom Port node** (as in JSON: `CUSTOM.portApiAi`)
- Operation: **invokeAgent**
- Agent Identifier: `context_retriever_agent`
- Prompt: instruct it to return *raw JSON only* with fields:
  - `service_name, repository, service_tier, sla_hours, owners, slack_channel, environment, dependencies, jira_project_key`
- Connect: **Webhook Trigger → Get Context From Port**
- Credentials: configure Port credentials required by this custom node (depends on your Port/n8n integration).

4) **Add node: OpenAI Remediation Plan**
- Node type: **OpenAI**
- Resource: **Chat**
- Model: `gpt-4o-mini` (or compatible)
- Prompt: include webhook vulnerability fields + Port context; enforce “JSON only” output with schema:
  - `summary, impact, remediation_steps[], is_auto_fixable, fix_prompt, estimated_effort`
- Connect: **Get Context From Port → OpenAI Remediation Plan**
- Credentials: set OpenAI API credential in n8n.

5) **Add node: Check Severity Level**
- Node type: **Switch**
- Value (string): set expression to webhook severity lowercased:
  - `{{ $('Webhook Trigger').item.json.body.severity.toLowerCase() }}`
- Add rules:
  1. Equals `critical`
  2. Equals `high`
  3. Regex `.*` (catch-all)
- Connect: **OpenAI Remediation Plan → Check Severity Level**

6) **Add Jira ticket creation nodes**
- Node type: **Jira** (three separate nodes)

6.1) **Create Critical Jira Ticket**
- Operation: **Create Issue**
- Project: select your security project (in JSON it is “Port”)
- Issue Type: Task
- Priority: “Now (Urgent)” (ensure your Jira instance has this priority or map to your own)
- Summary: `[CRITICAL] {{ vulnerability_id }}: {{ description.substring(0,100) }}`
- Description: include vulnerability details, parsed Port context, parsed AI remediation plan, and SLA.
- Connect: **Check Severity Level (critical output) → Create Critical Jira Ticket**
- Credentials: Jira credential (OAuth/API token) with permission to create issues.

6.2) **Create High Jira Ticket**
- Priority: “High”
- Similar summary/description structure (can omit impact if desired).
- Connect: **Check Severity Level (high output) → Create High Jira Ticket**

6.3) **Create Medium/Low Jira Ticket**
- Priority: “Low”
- Summary uses dynamic severity.
- Connect: **Check Severity Level (fallback output) → Create Medium/Low Jira Ticket**

7) **Add node: Is Auto-Fixable?**
- Node type: **IF**
- Condition: compute boolean by parsing OpenAI message content:
  - True if `is_auto_fixable === true` OR `fix_prompt` is present and non-trivial
- Connect: **Create Critical Jira Ticket → Is Auto-Fixable?**

8) **Add node: Trigger Fix via Port AI Agent**
- Node type: **HTTP Request**
- Method: **POST**
- URL: `https://api.getport.io/v1/actions/run_claude_code/runs`
- Authentication: **Bearer Token** (HTTP Bearer Auth credential)
- Body: JSON with:
  - `properties.service`: from Port context (`repository`)
  - `properties.prompt`: built from vulnerability id + OpenAI `fix_prompt` + Jira key + instruction to create a PR
- Connect: **Is Auto-Fixable? (true) → Trigger Fix via Port AI Agent**
- Port prerequisite: a Port Action named `run_claude_code` must exist and be callable.

9) **Add Slack notification nodes**
- Node type: **Slack**
- Credentials: Slack app/bot token with permission to post to target channels.

9.1) **Alert Critical to Slack**
- Channel: from Port context `slack_channel` (fallback `#security-alerts`)
- Text: include vulnerability metadata, affected service, owners, summary, impact, Jira key, SLA, and “Auto-Fix Status: Triggered”
- Connect: **Trigger Fix via Port AI Agent → Alert Critical to Slack**

9.2) **Alert Critical (Manual Fix)**
- Channel: same dynamic logic
- Text: include remediation steps and “manual fix required”
- Connect: **Is Auto-Fixable? (false) → Alert Critical (Manual Fix)**

9.3) **Alert High to Slack**
- Channel: same dynamic logic
- Text: include summary + Jira key
- Connect: **Create High Jira Ticket → Alert High to Slack**

10) **(Optional hardening)**
- Add a Set/Code node early to normalize `severity` safely (avoid `.toLowerCase()` crashes).
- Consider posting Slack notifications for Medium/Low as well (currently none).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow automates vulnerability management: webhook → Port enrichment → OpenAI remediation plan → Jira ticket by severity → (critical) optional Claude Code PR → Slack notify | From Sticky Note (overview) |
| Prerequisites: Port catalog configured with services/ownership; Port AI agent `context_retriever_agent`; Claude Code action in Port; Jira project; Slack workspace; OpenAI API key | From Sticky Note (prerequisites) |
| Severity behavior: critical=urgent priority, high=high priority, medium/low=low priority; ticket includes vulnerability details + Port context + AI steps | From Sticky Note2 |
| Port output is consumed via `executionMessage` and parsed defensively; ensure your Port node returns JSON consistently | Observed from expressions across Jira/Slack/IF nodes |