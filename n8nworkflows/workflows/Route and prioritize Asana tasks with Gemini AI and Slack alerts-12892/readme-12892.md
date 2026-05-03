Route and prioritize Asana tasks with Gemini AI and Slack alerts

https://n8nworkflows.xyz/workflows/route-and-prioritize-asana-tasks-with-gemini-ai-and-slack-alerts-12892


# Route and prioritize Asana tasks with Gemini AI and Slack alerts

## 1. Workflow Overview

**Purpose:** Automatically triage newly added Asana tasks in a specific project, classify them with **Gemini**, assign them to the right owner based on category, enrich the task with an AI summary, and send a **Slack alert** only for **High** priority tasks.

**Target use cases:**
- Teams using an Asana “Inbox” project to capture untriaged work
- Automatic routing to bug/feature/support owners
- Fast escalation for urgent items via Slack

### 1.1 Configuration & Routing Rules
Defines the Asana Project ID to watch and the email-based routing rules (who handles Bugs/Features/Support), plus the Slack channel for alerts.

### 1.2 Asana Intake (Trigger)
Receives events when a task is added to the configured project.

### 1.3 AI Analysis (Gemini)
Gemini analyzes task title/notes and returns a strict JSON object: category, priority, and a one-sentence summary.

### 1.4 Data Normalization & Assignment Mapping
Parses Gemini output safely, applies defaults on parsing failure, maps category → assignee email, and carries the Asana task GID forward.

### 1.5 Update in Asana
Updates the task name (adds a red indicator for High priority), sets the assignee, and prepends AI summary metadata to the task notes.

### 1.6 Conditional Slack Escalation
If priority is High, send an urgent Slack message to the configured channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Configuration & Routing Rules
**Overview:** Centralizes editable parameters (Project ID, routing emails, Slack channel) so the rest of the workflow can reference them consistently.

**Nodes Involved:** `Configuration`

#### Node: Configuration
- **Type / role:** `Set` node; produces a config object used by other nodes.
- **Key configuration:**
  - Sets:
    - `projectId` (string; currently blank in JSON and must be filled)
    - `email_bug`, `email_feature`, `email_support`
    - `slackChannel` (defaults to `alerts`)
  - **Include other fields:** enabled (`includeOtherFields: true`) meaning any incoming JSON fields would be preserved (here it mostly acts as a config source).
- **Key expressions/variables used by others:**
  - `$('Configuration').first().json.projectId`
  - `$('Configuration').first().json.slackChannel`
  - `$('Configuration').first().json.email_bug` / `email_feature` / `email_support`
- **Connections:**
  - Output → `Asana Trigger`
- **Edge cases / failures:**
  - Empty/invalid `projectId` will prevent the Asana Trigger from correctly subscribing or filtering events.
  - Assignee emails must match real Asana users in the workspace or the update may fail.
- **Version notes:** Node typeVersion `3.4` (modern Set node UI).

---

### Block 2 — Asana Intake (Trigger)
**Overview:** Listens for newly added tasks in the configured Asana project and passes task data downstream.

**Nodes Involved:** `Asana Trigger`

#### Node: Asana Trigger
- **Type / role:** `asanaTrigger`; event-driven intake from Asana via webhook.
- **Key configuration:**
  - **Resource:** `task`
  - **Events:** `added`
  - **Project:** dynamic expression: `={{ $('Configuration').first().json.projectId }}`
  - **Webhook ID:** `asana-triage-webhook` (internal n8n identifier for the webhook)
- **Credentials:**
  - Uses **Asana API** credential (`asanaApi`)
- **Connections:**
  - Input ← `Configuration`
  - Output → `Gemini: Analyze`
- **Edge cases / failures:**
  - Asana credential revoked/expired → webhook cannot authenticate.
  - If projectId is wrong or not accessible by the credential user, no events arrive (or setup may fail).
  - Webhook delivery can be delayed or retried by Asana; duplicate events are possible depending on Asana behavior and n8n execution settings (workflow has no explicit deduplication).
- **Version notes:** typeVersion `1`.

---

### Block 3 — AI Analysis (Gemini)
**Overview:** Uses Gemini to classify the task into one of three categories and assign a priority plus a one-line summary, requesting JSON-only output.

**Nodes Involved:** `Gemini: Analyze`

#### Node: Gemini: Analyze
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini`; LLM call to Gemini.
- **Key configuration:**
  - **Model:** `models/gemini-1.5-flash`
  - **Prompt content:** Instructs Gemini to act as a PM and output **JSON only** with:
    - `category`: `Bug` | `Feature` | `Support`
    - `priority`: `High` | `Normal`
    - `summary`: one sentence
  - Injects task fields:
    - `Title: {{ $json.name }}`
    - `Notes: {{ $json.notes }}`
- **Credentials:**
  - Uses **Google Gemini(PaLM) API** credential (`googlePalmApi`)
- **Connections:**
  - Input ← `Asana Trigger`
  - Output → `Map Data`
- **Edge cases / failures:**
  - Model may return non-JSON (markdown fences, extra commentary). Downstream code attempts to clean and parse, but can still fail.
  - Token/size limits if Asana notes are very large.
  - API quota/rate limits, transient 5xx errors, or timeouts.
- **Version notes:** typeVersion `1` (LangChain-based node).

---

### Block 4 — Data Normalization & Assignment Mapping
**Overview:** Extracts Gemini text, attempts JSON parsing, falls back to safe defaults, maps category to the correct assignee email, and attaches the original Asana task GID.

**Nodes Involved:** `Map Data`

#### Node: Map Data
- **Type / role:** `Code` node; transforms Gemini output into a clean JSON structure for the update step.
- **Key logic/configuration choices (interpreted):**
  1. **Extract model output text** from:  
     `const text = $input.first().json.content.parts[0].text;`
  2. **Load config** from the Set node:  
     `const config = $('Configuration').first().json;`
  3. **Safe parse with cleanup**:
     - Removes ```json fences and ``` then `JSON.parse`.
     - On parse failure, defaults to:  
       `{ category: "Support", priority: "Normal", summary: text }`
  4. **Map category → assignee email**
     - Default: `email_support`
     - If category `Bug` → `email_bug`
     - If category `Feature` → `email_feature`
  5. **Carry task identifier**
     - `data.taskGid = $('Asana Trigger').first().json.gid;`
  6. **Return** `{ json: data }`
- **Connections:**
  - Input ← `Gemini: Analyze`
  - Output → `Update Task`
- **Edge cases / failures:**
  - If Gemini node output structure changes (e.g., `content.parts[0].text` missing), this will throw.
  - If Asana Trigger output doesn’t include `gid` (or the event payload differs), `taskGid` may be undefined and the update will fail.
  - Category mismatches (e.g., “BUG” or “Inquiry”) won’t match exact checks; assignee will default to support.
- **Version notes:** Code node typeVersion `2` (current JS runtime behavior depends on your n8n version).

---

### Block 5 — Route & Update in Asana
**Overview:** Updates the Asana task: sets assignee by email, modifies the name to mark urgent tasks, and prepends an AI summary to notes while preserving original notes.

**Nodes Involved:** `Update Task`

#### Node: Update Task
- **Type / role:** `asana` node; performs task update.
- **Key configuration:**
  - **Resource:** `task`
  - **Operation:** `update`
  - **Task ID:** `={{ $json.taskGid }}`
  - **Name:** prefixes red indicator if High:  
    `={{ $json.priority === 'High' ? '🔴 ' : '' }}{{ $('Asana Trigger').first().json.name }}`
  - **Notes:** prepends AI metadata, then original notes:
    ```
    🤖 **AI Summary:** {{ $json.summary }}
    **Category:** {{ $json.category }}

    ---
    {{ $('Asana Trigger').first().json.notes }}
    ```
  - **Assignee:** `={{ $json.assignee_email }}`
- **Credentials:**
  - Uses **Asana API** credential (`asanaApi`)
- **Connections:**
  - Input ← `Map Data`
  - Output → `Is High Priority?`
- **Edge cases / failures:**
  - Assigning by email can fail if the email is not in the workspace or if Asana API expects a user gid depending on account settings/API behavior.
  - Emoji or markdown in task name/notes is generally fine, but formatting may render differently in Asana.
  - If original notes are null/empty, the expression still works but may append “undefined” depending on payload; consider defaulting to empty string if that happens in your Asana trigger payload.
- **Version notes:** typeVersion `1`.

---

### Block 6 — Conditional Slack Escalation
**Overview:** Checks if the AI-assigned priority is High; only then sends an urgent Slack message using an incoming webhook.

**Nodes Involved:** `Is High Priority?`, `Notify Slack`

#### Node: Is High Priority?
- **Type / role:** `If` node; branching logic.
- **Key configuration:**
  - Condition: `={{ $json.priority }}` **equals** `High` (string equals, strict validation enabled)
- **Connections:**
  - Input ← `Update Task`
  - **True output** → `Notify Slack`
  - False output is unused (workflow ends silently for non-high priority).
- **Edge cases / failures:**
  - If priority is missing or not exactly `High` (case/whitespace), Slack won’t trigger.
- **Version notes:** typeVersion `2.2`.

#### Node: Notify Slack
- **Type / role:** `slackweb`; posts message to Slack via incoming webhook.
- **Authentication:** webhook-based (`Slack Incoming Webhook` credential)
- **Key configuration:**
  - **Channel:** `={{ $('Configuration').first().json.slackChannel }}`
  - **Message text:** includes task name, assignee email, summary; references Asana Trigger for original task name:
    ```
    🚨 **Urgent Task Assigned**

    Task: {{ $('Asana Trigger').first().json.name }}
    Assigned To: {{ $json.assignee_email }}
    Summary: {{ $json.summary }}

    👉 Check Asana for details.
    ```
- **Connections:**
  - Input ← `Is High Priority?` (true path)
  - Output: none
- **Edge cases / failures:**
  - Webhook misconfigured/revoked → Slack returns 4xx.
  - Channel routing depends on Slack webhook capabilities (some webhooks are locked to a single channel/workspace).
- **Version notes:** typeVersion `1.1`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note Main | stickyNote | Documentation / operator guidance | — | — | # Asana Triage Agent / How it works / Setup steps (Asana, Gemini, Slack credentials; set Project ID + emails; emails must match Asana users) |
| Sticky Note Config | stickyNote | Section label for config block | — | — | ## 1. Config Define your Routing Rules (Who handles what) and Project ID. |
| Sticky Note AI | stickyNote | Section label for AI block | — | — | ## 2. AI Analysis Gemini acts as a PM, categorizing the task based on context. |
| Sticky Note Route | stickyNote | Section label for routing/update block | — | — | ## 3. Route & Update Maps category to email and updates the task. |
| Sticky Note Alert | stickyNote | Section label for alert block | — | — | ## 4. Emergency Alert Notifies Slack only if the task is urgent. |
| Configuration | set | Stores project ID, routing emails, Slack channel | — | Asana Trigger | ## 1. Config Define your Routing Rules (Who handles what) and Project ID. |
| Asana Trigger | asanaTrigger | Triggers when a task is added to a project | Configuration | Gemini: Analyze | # Asana Triage Agent (Trigger: detects new tasks in project) |
| Gemini: Analyze | googleGemini (LangChain) | Classifies task and returns JSON category/priority/summary | Asana Trigger | Map Data | ## 2. AI Analysis Gemini acts as a PM, categorizing the task based on context. |
| Map Data | code | Parses Gemini output and maps category to assignee email | Gemini: Analyze | Update Task | ## 3. Route & Update Maps category to email and updates the task. |
| Update Task | asana | Updates task title, notes, and assignee | Map Data | Is High Priority? | ## 3. Route & Update Maps category to email and updates the task. |
| Is High Priority? | if | Branches only when priority == High | Update Task | Notify Slack (true path) | ## 4. Emergency Alert Notifies Slack only if the task is urgent. |
| Notify Slack | slackweb | Sends urgent Slack alert via webhook | Is High Priority? (true) | — | ## 4. Emergency Alert Notifies Slack only if the task is urgent. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Route and prioritize Asana tasks using AI agent”** (or your preferred title).

2. **Add a Set node named “Configuration”**
   - Add string fields:
     - `projectId` = *(your Asana project GID as a string)*
     - `email_bug` = *(email of bug triage owner)*
     - `email_feature` = *(email of feature requests owner)*
     - `email_support` = *(email of support/questions owner)*
     - `slackChannel` = `alerts` (or your channel name)
   - Enable **Include Other Fields** (optional but matches the workflow).

3. **Add an Asana Trigger node named “Asana Trigger”**
   - Credentials: connect **Asana API** (OAuth/Personal token per your n8n setup).
   - Resource: **Task**
   - Event: **Added**
   - Project: set expression to: `$('Configuration').first().json.projectId`
   - Connect: **Configuration → Asana Trigger**

4. **Add a Gemini node named “Gemini: Analyze”**
   - Node type: **Google Gemini (LangChain)** (`@n8n/n8n-nodes-langchain.googleGemini`)
   - Credentials: connect **Google Gemini(PaLM) API**
   - Model: `models/gemini-1.5-flash`
   - Message/prompt: instruct JSON-only output and include:
     - `Title: {{ $json.name }}`
     - `Notes: {{ $json.notes }}`
   - Connect: **Asana Trigger → Gemini: Analyze**

5. **Add a Code node named “Map Data”**
   - Paste logic that:
     - Reads Gemini text output from `content.parts[0].text`
     - Removes code fences and parses JSON
     - Defaults to Support/Normal on parse failure
     - Maps category → config email fields
     - Sets `taskGid` from `$('Asana Trigger').first().json.gid`
     - Returns `{ json: data }`
   - Connect: **Gemini: Analyze → Map Data**

6. **Add an Asana node named “Update Task”**
   - Credentials: same Asana credential as trigger.
   - Resource: **Task**
   - Operation: **Update**
   - Task ID: expression `{{ $json.taskGid }}`
   - Assignee: expression `{{ $json.assignee_email }}`
   - Name: expression  
     `{{ $json.priority === 'High' ? '🔴 ' : '' }}{{ $('Asana Trigger').first().json.name }}`
   - Notes: expression that prepends AI summary + category and then original notes from the trigger.
   - Connect: **Map Data → Update Task**

7. **Add an If node named “Is High Priority?”**
   - Condition: String equals
     - Left: `{{ $json.priority }}`
     - Right: `High`
   - Connect: **Update Task → Is High Priority?**

8. **Add a Slack node named “Notify Slack” (Slack Incoming Webhook)**
   - Node type: `slackweb`
   - Authentication: **Incoming Webhook**
   - Credentials: create/connect **Slack Incoming Webhook** credential.
   - Channel: expression `{{ $('Configuration').first().json.slackChannel }}`
   - Text: include task name, assignee_email, summary (as in the workflow).
   - Connect: **Is High Priority? (true output) → Notify Slack**

9. **(Optional) Add sticky notes**
   - Add notes for: main description, config, AI analysis, route/update, emergency alert—mirroring the provided content for maintainability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Asana Triage Agent**: Detects added tasks in a project, Gemini categorizes + prioritizes, routes to an assignee via Config, updates task with AI summary, sends Slack alert if High priority. | Sticky note “Asana Triage Agent” |
| **Setup steps:** Connect Asana, Gemini, Slack; set Project ID + routing emails in “Configuration”; emails must match real Asana users. | Sticky note “Asana Triage Agent” |
| **Config:** Define routing rules and Project ID. | Sticky note “1. Config” |
| **AI Analysis:** Gemini acts as a PM categorizing based on context. | Sticky note “2. AI Analysis” |
| **Route & Update:** Map category to email and update task. | Sticky note “3. Route & Update” |
| **Emergency Alert:** Notify Slack only if urgent. | Sticky note “4. Emergency Alert” |