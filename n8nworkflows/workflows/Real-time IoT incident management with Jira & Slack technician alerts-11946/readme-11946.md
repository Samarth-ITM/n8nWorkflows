Real-time IoT incident management with Jira & Slack technician alerts

https://n8nworkflows.xyz/workflows/real-time-iot-incident-management-with-jira---slack-technician-alerts-11946


# Real-time IoT incident management with Jira & Slack technician alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow ingests real-time IoT machine telemetry via a webhook, detects potential failure conditions (temperature/vibration thresholds), creates a **Jira maintenance ticket**, then checks **Slack technician availability** (presence) to route an alert to either the technician channel (if someone is active) or escalate to an emergency channel (if nobody is active).

**Target use cases:**
- Predictive maintenance / condition-based monitoring (temperature, vibration)
- Automated IT/OT incident creation in Jira
- Technician dispatching based on real-time Slack presence
- Escalation paths when no technicians are online

### 1.1 Input Reception & Threshold Detection
Receives IoT payload → checks if telemetry exceeds thresholds → proceeds only on “failure” path (as wired).

### 1.2 Ticketing (Jira)
Creates a Jira issue summarizing the machine incident for maintenance tracking.

### 1.3 Technician Discovery & Presence Checking (Slack)
Fetches members of a Slack “technicians” channel → loops through members → queries presence.

### 1.4 Technician Selection & Alert/Escalation
Aggregates loop results → selects first active technician → sends alert if available, else escalates.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Failure Detection
**Overview:**  
Accepts IoT alerts via HTTP POST and evaluates the machine telemetry against predefined thresholds (temperature > 90 OR vibration > 5). Only the “true” branch is connected onward.

**Nodes involved:**
- Receive IoT Machine Alert
- Check Failure Threshold

#### Node: Receive IoT Machine Alert
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point receiving IoT telemetry.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: auto-generated UUID-like path (`0afced...`)
  - Expects incoming JSON in `body` (used later as `$json.body.*`).
- **Key expressions/variables:**
  - Downstream nodes reference: `{{$json.body.temperature}}`, `{{$json.body.vibration}}`, `{{$json.body.machineId}}`, `{{$json.body.timestamp}}`.
- **Connections:**
  - Output → **Check Failure Threshold**
- **Version notes:** typeVersion **2.1**.
- **Edge cases / failures:**
  - Missing fields in body (e.g., no `temperature`) will cause IF node numeric comparisons to fail strict validation.
  - If device sends strings instead of numbers, strict type validation may block comparisons.
  - Webhook authentication is not configured; endpoint may be publicly callable unless protected at n8n level.

#### Node: Check Failure Threshold
- **Type / role:** `IF` (n8n-nodes-base.if) — threshold gate.
- **Configuration (interpreted):**
  - Condition group: **OR**
  - Condition A: `temperature` **> 90**
  - Condition B: `vibration` **> 5**
  - Uses “strict” type validation.
- **Key expressions:**
  - `={{ $json.body.temperature }}`
  - `={{ $json.body.vibration }}`
- **Connections:**
  - **True** output (index 0) → **Create Jira Maintenance Ticket**
  - **False** output is **not connected** (no action if below thresholds).
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - If payload is missing `body` or fields, expression resolves to `undefined` → condition evaluation may error or behave unexpectedly under strict validation.
  - Thresholds are hardcoded; no hysteresis/debouncing (may create repeated tickets under noisy readings).

**Sticky note context (applies to this block):**
- **“Data Intake & Failure Detection”**: Receives real-time machine data and evaluates safety thresholds; creates Jira ticket if failure detected.

---

### Block 2 — Jira Ticket Creation
**Overview:**  
Creates a high-priority Jira “Task” in a selected project, embedding machine telemetry in summary/description.

**Nodes involved:**
- Create Jira Maintenance Ticket

#### Node: Create Jira Maintenance Ticket
- **Type / role:** `Jira` (n8n-nodes-base.jira) — issue creation.
- **Configuration (interpreted):**
  - Operation: **Create Issue** (implied by node setup)
  - Project: ID `10000` (“n8n sample project” in cached name)
  - Issue type: ID `10003` (“Task”)
  - Summary: `Predictive Failure Alert: <machineId>`
  - Priority: `High` (id `2`)
  - Description includes machineId, timestamp, temperature, vibration
- **Key expressions:**
  - Summary: `=Predictive Failure Alert: {{ $json.body.machineId }}`
  - Description: `=Machine {{ $json.body.machineId }} triggered alert at {{ $json.body.timestamp }}. Temperature: {{ $json.body.temperature }}°C Vibration: {{ $json.body.vibration }}`
- **Connections:**
  - Input ← **Check Failure Threshold** (true path)
  - Output → **Fetch Technician Channel Members**
- **Version notes:** typeVersion **1** (older Jira node generation; UI/options may differ from later versions).
- **Edge cases / failures:**
  - Jira credential/auth failure (API token/OAuth, base URL, permissions).
  - Invalid project/issueType IDs if moved across Jira instances.
  - If required Jira fields exist in the target project (custom required fields), create will fail unless provided.
  - Description references webhook payload; if those fields missing, the ticket text will be incomplete.

---

### Block 3 — Technician Identification & Presence Check (Slack loop)
**Overview:**  
Pulls all members of a Slack channel, iterates through them in batches, and fetches each member’s presence. Presence results are normalized and fed back into the batch loop until complete.

**Nodes involved:**
- Fetch Technician Channel Members
- Prepare Technician Iteration Data
- Loop Through Technicians
- Get Technician Presence Status
- Check If Technician Active
- Format Technician Availability Result

#### Node: Fetch Technician Channel Members
- **Type / role:** `Slack` (n8n-nodes-base.slack) — retrieves channel members list.
- **Configuration (interpreted):**
  - Resource: **channel**
  - Operation: **member** (list members)
  - Channel: `C09S57E2JQ2` (cached name “n8n”)
- **Connections:**
  - Input ← **Create Jira Maintenance Ticket**
  - Output → **Prepare Technician Iteration Data**
- **Version notes:** typeVersion **2.3**.
- **Edge cases / failures:**
  - Slack scopes required: `channels:read` (or `conversations:read`) and membership listing permissions; for private channels also need appropriate access.
  - Large channels may return many members; API pagination/rate limits may apply.
  - If the workflow’s Slack app isn’t in the channel, member listing can fail.

#### Node: Prepare Technician Iteration Data
- **Type / role:** `Set` (n8n-nodes-base.set) — shapes one item per technician for iteration; attaches machine + Jira data.
- **Configuration (interpreted):**
  - Sets fields:
    - `technicianAvailable` = `"false"` (string)
    - `technician member id` = current channel member id (`$json.member`)
    - machine telemetry and timestamp pulled from **Check Failure Threshold**
    - `jira ticket` from **Create Jira Maintenance Ticket** (`.key`)
- **Key expressions:**
  - `={{ $json.member }}`
  - `={{ $('Check Failure Threshold').item.json.body.machineId }}`
  - `={{ $('Create Jira Maintenance Ticket').item.json.key }}`
- **Connections:**
  - Input ← **Fetch Technician Channel Members**
  - Output → **Loop Through Technicians**
- **Version notes:** typeVersion **3.4**.
- **Edge cases / failures:**
  - Assumes Slack member list output has `member` field; if Slack node returns a different structure (array `members`), mapping will break.
  - Uses cross-node item references (`$('...').item...`) which can fail if item pairing differs (multi-item flows).

#### Node: Loop Through Technicians
- **Type / role:** `Split In Batches` — iterates through technician items.
- **Configuration (interpreted):**
  - Batch options not explicitly set (defaults apply; commonly 1 item per batch unless configured).
  - Two outputs used:
    - Main output 0: signals completion when no items left (wired to final selection)
    - Main output 1: yields each batch item(s) (wired to presence check)
- **Connections:**
  - Output 0 → **Select Final Available Technician** (end-of-loop aggregation trigger)
  - Output 1 → **Get Technician Presence Status**
  - Receives loopback input from **Format Technician Availability Result** to continue batching.
- **Version notes:** typeVersion **3**.
- **Edge cases / failures:**
  - Misconfigured batch size can hit Slack rate limits (too many presence checks too fast) or slow down.
  - If loopback wiring is wrong, it can stop after first batch or never reach completion.

#### Node: Get Technician Presence Status
- **Type / role:** `Slack` — queries a user’s real-time presence.
- **Configuration (interpreted):**
  - Resource: **user**
  - Operation: **getPresence**
  - User ID: `={{ $json["technician member id"] }}`
- **Connections:**
  - Input ← **Loop Through Technicians** (batch items)
  - Output → **Check If Technician Active**
- **Version notes:** typeVersion **2.3**.
- **Edge cases / failures:**
  - Slack scope requirement: `users:read.presence` (and `users:read`).
  - Presence may be restricted in some Slack plans/workspaces; may return limited data.
  - API rate limits likely if many members.

#### Node: Check If Technician Active
- **Type / role:** `IF` — intended to branch on active presence.
- **Configuration (as built):**
  - Condition: compares `$json.presence` **equals** `$json.presence`
  - This is **always true** when `presence` exists (a self-equality check).
- **Connections:**
  - True output → **Format Technician Availability Result**
  - False output unused
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - Logical bug: it does not check for `"active"`; it checks equality with itself. This means all users pass through as if “valid”.
  - If `presence` is missing/undefined, strict validation may behave unexpectedly.

#### Node: Format Technician Availability Result
- **Type / role:** `Set` — normalizes presence result and re-attaches machine/Jira data; feeds loop continuation.
- **Configuration (interpreted):**
  - Sets:
    - `technicianAvailable` = `={{ $json.presence }}` (typically `"active"` or `"away"`)
    - `technician member id` back from the current loop item
    - machine/Jira fields again from earlier nodes
- **Key expressions:**
  - `={{ $json.presence }}`
  - `={{ $('Loop Through Technicians').item.json["technician member id"] }}`
  - `={{ $('Create Jira Maintenance Ticket').item.json.key }}`
- **Connections:**
  - Output → **Loop Through Technicians** (loopback)
- **Version notes:** typeVersion **3.4**.
- **Edge cases / failures:**
  - Item linking assumptions: cross-references to `Loop Through Technicians` and `Check Failure Threshold` may break if concurrency/multi-item changes.
  - Field naming inconsistency: earlier node sets `machin id` (typo) while later expects `machine id`. This node sets `machine id`, so downstream code relies on this corrected field.

**Sticky note context (applies to this block):**
- **“Technician Identification & Presence Check”**: Fetches channel members, checks presence in batches for scalability.

---

### Block 4 — Technician Selection & Escalation Logic
**Overview:**  
After the loop completes, a Code node selects the first technician with `technicianAvailable === "active"`. If found, sends an alert; otherwise escalates.

**Nodes involved:**
- Select Final Available Technician
- Check If Any Technician Available
- Send Technicians Alert (Active Tech)
- Escalate to Ops Emergency Channel

#### Node: Select Final Available Technician
- **Type / role:** `Code` — aggregates all technician presence items and chooses first active.
- **Configuration (interpreted):**
  - Reads `items.map(i => i.json)` from all incoming items.
  - Builds `baseData` using `itemsList[0]`:
    - `machine_id` from `itemsList[0]["machine id"]`
    - temperature, vibration, timestamp, jira ticket
  - Finds first item where `technicianAvailable === "active"`
  - Outputs a single item:
    - `finalTechnicianAvailable: true/false`
    - `technicianId: <slack user id or null>`
    - plus baseData
- **Connections:**
  - Input ← **Loop Through Technicians** output 0 (completion path)
  - Output → **Check If Any Technician Available**
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If no items reach this node, `itemsList[0]` will be undefined → runtime error.
  - If earlier nodes never set `"machine id"` consistently, `machine_id` may become undefined.
  - Presence values must exactly match `"active"` (case-sensitive).

#### Node: Check If Any Technician Available
- **Type / role:** `IF` — branches based on boolean.
- **Configuration (interpreted):**
  - Condition: `$json.finalTechnicianAvailable === true`
- **Connections:**
  - True output → **Send Technicians Alert (Active Tech)**
  - False output → **Escalate to Ops Emergency Channel**
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - If `finalTechnicianAvailable` is missing/not boolean, strict validation can fail.

#### Node: Send Technicians Alert (Active Tech)
- **Type / role:** `Slack` — posts incident message to technician channel.
- **Configuration (interpreted):**
  - Post text includes machine_id, temperature, vibration, jira ticket
  - Sends to channel `C09S57E2JQ2` (currently same as escalation channel in this workflow)
  - “include link to workflow” disabled
- **Connections:**
  - Input ← **Check If Any Technician Available** (true branch)
- **Version notes:** typeVersion **2.3**.
- **Edge cases / failures:**
  - `chat:write` scope required.
  - Channel selection: currently targets the same channel as escalation; typically you’d use different IDs.
  - Message includes “degree calcius” typo; purely cosmetic.

#### Node: Escalate to Ops Emergency Channel
- **Type / role:** `Slack` — posts urgent escalation message.
- **Configuration (interpreted):**
  - Posts urgent formatted message with machine_id, temperature, vibration, jira ticket
  - Sends to channel `C09S57E2JQ2` (same as technician channel in this JSON)
- **Connections:**
  - Input ← **Check If Any Technician Available** (false branch)
- **Version notes:** typeVersion **2.3**.
- **Edge cases / failures:**
  - Same Slack permission and channel membership requirements as above.
  - If escalation must page an on-call system, Slack alone may not be sufficient.

**Sticky note context (applies to this block):**
- **“Technician Selection & Escalation Logic”**: Selects first available tech; otherwise escalates.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive IoT Machine Alert | Webhook | Entry point for IoT telemetry | — | Check Failure Threshold | ## Data Intake & Failure Detection<br>### Receives real-time machine data from IoT devices and evaluates it against defined safety thresholds (temperature, vibration, etc.). If a failure is detected, automatically creates a Jira maintenance ticket with machine details, summary, labels, priority, and optional technician assignment. |
| Check Failure Threshold | IF | Detects threshold breach (temp/vibration) | Receive IoT Machine Alert | Create Jira Maintenance Ticket | ## Data Intake & Failure Detection<br>### Receives real-time machine data from IoT devices and evaluates it against defined safety thresholds (temperature, vibration, etc.). If a failure is detected, automatically creates a Jira maintenance ticket with machine details, summary, labels, priority, and optional technician assignment. |
| Create Jira Maintenance Ticket | Jira | Creates maintenance issue in Jira | Check Failure Threshold | Fetch Technician Channel Members | ## Data Intake & Failure Detection<br>### Receives real-time machine data from IoT devices and evaluates it against defined safety thresholds (temperature, vibration, etc.). If a failure is detected, automatically creates a Jira maintenance ticket with machine details, summary, labels, priority, and optional technician assignment. |
| Fetch Technician Channel Members | Slack | Lists members of technician channel | Create Jira Maintenance Ticket | Prepare Technician Iteration Data | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Prepare Technician Iteration Data | Set | Builds per-technician items + machine/Jira context | Fetch Technician Channel Members | Loop Through Technicians | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Loop Through Technicians | SplitInBatches | Iterates over technicians and controls loop | Prepare Technician Iteration Data; Format Technician Availability Result | Get Technician Presence Status; Select Final Available Technician | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Get Technician Presence Status | Slack | Retrieves Slack presence for a user | Loop Through Technicians | Check If Technician Active | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Check If Technician Active | IF | (Intended) filter/branch on presence | Get Technician Presence Status | Format Technician Availability Result | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Format Technician Availability Result | Set | Normalizes presence result and continues loop | Check If Technician Active | Loop Through Technicians | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Select Final Available Technician | Code | Chooses first active tech; produces final routing decision | Loop Through Technicians | Check If Any Technician Available | ## Technician Selection & Escalation Logic <br>### Determines the first available technician using custom logic. If a technician is online, alerts are routed to the designated channel; if none are available, the workflow triggers fallback escalation to the emergency channel, ensuring no downtime goes unnoticed. |
| Check If Any Technician Available | IF | Routes to alert vs escalation | Select Final Available Technician | Send Technicians Alert (Active Tech); Escalate to Ops Emergency Channel | ## Technician Selection & Escalation Logic <br>### Determines the first available technician using custom logic. If a technician is online, alerts are routed to the designated channel; if none are available, the workflow triggers fallback escalation to the emergency channel, ensuring no downtime goes unnoticed. |
| Send Technicians Alert (Active Tech) | Slack | Posts technician alert message | Check If Any Technician Available | — | ## Technician Selection & Escalation Logic <br>### Determines the first available technician using custom logic. If a technician is online, alerts are routed to the designated channel; if none are available, the workflow triggers fallback escalation to the emergency channel, ensuring no downtime goes unnoticed. |
| Escalate to Ops Emergency Channel | Slack | Posts escalation message | Check If Any Technician Available | — | ## Technician Selection & Escalation Logic <br>### Determines the first available technician using custom logic. If a technician is online, alerts are routed to the designated channel; if none are available, the workflow triggers fallback escalation to the emergency channel, ensuring no downtime goes unnoticed. |
| Sticky Note | Sticky Note | Documentation / setup guidance | — | — | ## How it Works<br>This workflow automates predictive maintenance by linking IoT devices, Jira, and Slack. When a machine sends a webhook with data like temperature or vibration, the workflow checks if values exceed safety thresholds. If a potential failure is detected, it automatically creates a Jira maintenance ticket with all machine details.<br><br>Next, the workflow fetches Slack channel members, checks each technician’s real-time presence, and determines the first available technician. If someone is active, an alert is sent to the technician channel; if no one is available, it escalates to the emergency channel, ensuring no downtime goes unnoticed.<br><br>## Setup Steps<br>Import Workflow: Go to n8n → Workflows → Import from File and select the JSON.<br><br>Configure Slack: Add required scopes (users:read, users:read.presence, channels:read, chat:write) and reconnect credentials.<br><br>Update Channels: Use your preferred Slack channels for technicians and emergency alerts.<br><br>Configure Jira: Add credentials, select project and issue type, and set priority mapping if needed.<br><br>Deploy Webhook: Copy the n8n webhook URL and configure your IoT device to POST machine data.<br><br>Test System: Send a test payload to ensure Jira tickets are created and Slack notifications route correctly based on technician availability. |
| Sticky Note2 | Sticky Note | Documentation: intake/detection | — | — | ## Data Intake & Failure Detection<br>### Receives real-time machine data from IoT devices and evaluates it against defined safety thresholds (temperature, vibration, etc.). If a failure is detected, automatically creates a Jira maintenance ticket with machine details, summary, labels, priority, and optional technician assignment. |
| Sticky Note3 | Sticky Note | Documentation: tech presence loop | — | — | ## Technician Identification & Presence Check<br>### Fetches members from a selected Slack channel and processes them in batches to check real-time presence. Ensures scalable, efficient monitoring, providing a list of active technicians and supporting flexible configuration for any chosen technician channel. |
| Sticky Note1 | Sticky Note | Documentation: selection/escalation | — | — | ## Technician Selection & Escalation Logic <br>### Determines the first available technician using custom logic. If a technician is online, alerts are routed to the designated channel; if none are available, the workflow triggers fallback escalation to the emergency channel, ensuring no downtime goes unnoticed. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Workflow**
- New workflow named: **“Real-Time IoT Incident Management with Jira & Slack Technician Alerts”**
- (Optional) Settings: Execution Order = **v1**

2) **Add Webhook node**
- Node: **Webhook**
- Name: **Receive IoT Machine Alert**
- Method: **POST**
- Path: generate a unique path (n8n will provide a Test/Production URL)
- Expected payload example (device POST body):
  - `machineId` (string)
  - `timestamp` (string/ISO)
  - `temperature` (number)
  - `vibration` (number)

3) **Add IF node for thresholds**
- Node: **IF**
- Name: **Check Failure Threshold**
- Conditions:
  - OR:
    - `{{$json.body.temperature}}` **Number > 90**
    - `{{$json.body.vibration}}` **Number > 5**
- Connect: **Webhook → IF**

4) **Add Jira node: create issue**
- Node: **Jira**
- Name: **Create Jira Maintenance Ticket**
- Credentials: configure Jira (API token/OAuth) and base URL; ensure permission to create issues.
- Project: select your target project
- Issue type: choose “Task” (or your maintenance type)
- Priority: **High**
- Summary (expression): `Predictive Failure Alert: {{ $json.body.machineId }}`
- Description (expression):  
  `Machine {{ $json.body.machineId }} triggered alert at {{ $json.body.timestamp }}. Temperature: {{ $json.body.temperature }}°C Vibration: {{ $json.body.vibration }}`
- Connect: **IF (true) → Jira**

5) **Add Slack node: fetch channel members**
- Node: **Slack**
- Name: **Fetch Technician Channel Members**
- Credentials: Slack OAuth2 with scopes:
  - `users:read`
  - `users:read.presence`
  - `channels:read` (or `conversations:read` depending on Slack app type)
  - `chat:write`
- Resource: **Channel**
- Operation: **Member** (list members)
- Channel: select your **technician channel**
- Connect: **Jira → Slack (members)**

6) **Add Set node to prepare per-technician items**
- Node: **Set**
- Name: **Prepare Technician Iteration Data**
- Add fields:
  - `technicianAvailable` = `"false"` (string)
  - `technician member id` = `{{$json.member}}` (ensure your Slack members output matches this; if it outputs `members[]`, map accordingly)
  - `machin id` = `{{$('Check Failure Threshold').item.json.body.machineId}}` (note: typo preserved from workflow)
  - `temperature` = `{{$('Check Failure Threshold').item.json.body.temperature}}`
  - `vibration` = `{{$('Check Failure Threshold').item.json.body.vibration}}`
  - `timestamp` = `{{$('Check Failure Threshold').item.json.body.timestamp}}`
  - `jira ticket` = `{{$('Create Jira Maintenance Ticket').item.json.key}}`
- Connect: **Slack (members) → Set**

7) **Add Split In Batches node**
- Node: **Split In Batches**
- Name: **Loop Through Technicians**
- (Optional) Batch size: set to 1 to reduce Slack rate limit risk.
- Connect: **Prepare Technician Iteration Data → Split In Batches**

8) **Add Slack node: get presence**
- Node: **Slack**
- Name: **Get Technician Presence Status**
- Resource: **User**
- Operation: **Get Presence**
- User ID expression: `{{$json["technician member id"]}}`
- Connect: **Split In Batches (batch output) → Get Presence**

9) **Add IF node (presence check)**
- Node: **IF**
- Name: **Check If Technician Active**
- Replicate as-is from JSON (even though it is logically ineffective):
  - Condition: `{{$json.presence}}` equals `{{$json.presence}}`
- Connect: **Get Presence → IF**
- Connect IF (true) → next Set node

10) **Add Set node to format result and continue loop**
- Node: **Set**
- Name: **Format Technician Availability Result**
- Fields:
  - `technicianAvailable` = `{{$json.presence}}`
  - `technician member id` = `{{$('Loop Through Technicians').item.json["technician member id"]}}`
  - `machine id` = `{{$('Check Failure Threshold').item.json.body.machineId}}`
  - `temperature` = `{{$('Check Failure Threshold').item.json.body.temperature}}`
  - `vibration` = `{{$('Check Failure Threshold').item.json.body.vibration}}`
  - `timestamp` = `{{$('Check Failure Threshold').item.json.body.timestamp}}`
  - `jira ticket` = `{{$('Create Jira Maintenance Ticket').item.json.key}}`
- Connect: **IF (true) → Format Technician Availability Result**
- Loopback: **Format Technician Availability Result → Split In Batches** (input) to fetch next batch.

11) **Add Code node for final selection**
- Node: **Code**
- Name: **Select Final Available Technician**
- Paste logic:
  - Build `baseData` from first item’s machine fields
  - Select first item with `technicianAvailable === "active"`
  - Output `{finalTechnicianAvailable, technicianId, machine_id, temperature, vibration, timestamp, jira_ticket}`
- Connect: **Split In Batches (done/completed output) → Code**

12) **Add IF node to route**
- Node: **IF**
- Name: **Check If Any Technician Available**
- Condition: `{{$json.finalTechnicianAvailable}}` boolean equals `true`
- Connect: **Code → IF**

13) **Add Slack node to alert technicians**
- Node: **Slack**
- Name: **Send Technicians Alert (Active Tech)**
- Operation: post message to channel
- Channel: technician channel
- Text (expression):
  ```
  ⚠️ Predictive Failure Alert:
  Machine: {{ $json.machine_id }}
  Temperature: {{ $json.temperature }} degree calcius
  Vibration: {{ $json.vibration }}
  Jira Ticket : {{ $json.jira_ticket }}
  ```
- Connect: **IF (true) → Send Technicians Alert**

14) **Add Slack node for escalation**
- Node: **Slack**
- Name: **Escalate to Ops Emergency Channel**
- Channel: emergency channel (should typically differ from technician channel)
- Text (expression):
  ```
  🚨 URGENT MACHINE FAILURE ESCALATION : Machine: {{ $json.machine_id }} Temperature: {{ $json.temperature }} degree calcius Vibration: {{ $json.vibration }} Jira Ticket : {{ $json.jira_ticket }}
  ```
- Connect: **IF (false) → Escalate**

15) **(Optional) Add Sticky Notes**
- Add sticky notes with the provided content to document the workflow internally.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates predictive maintenance by linking IoT devices, Jira, and Slack; creates Jira ticket on threshold breach; checks Slack presence for routing; escalates if nobody active. | Sticky note “How it Works” |
| Slack scopes needed: `users:read`, `users:read.presence`, `channels:read`, `chat:write`. | Sticky note “Setup Steps” |
| Update technician and emergency channel IDs to match your workspace. | Sticky note “Setup Steps” |
| Configure Jira credentials, project, issue type, and priority mapping. | Sticky note “Setup Steps” |
| Deploy webhook URL to IoT device for POSTing machine data. | Sticky note “Setup Steps” |

### Important implementation cautions (from analysis)
- **Bug:** “Check If Technician Active” compares `$json.presence` to itself; it does not filter for `"active"`. The real filtering happens later in the Code node, but the IF node is effectively redundant.
- **Channel IDs:** Both alert and escalation currently post to the same Slack channel ID (`C09S57E2JQ2`). Update to distinct channels if desired.
- **Data shape risk:** The Slack “channel members” node output must match the Set node’s assumption (`$json.member`). If your Slack node returns an array (common: `members[]`), you must add a node to split the array into items first.