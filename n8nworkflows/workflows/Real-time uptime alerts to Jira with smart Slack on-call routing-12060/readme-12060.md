Real-time uptime alerts to Jira with smart Slack on-call routing

https://n8nworkflows.xyz/workflows/real-time-uptime-alerts-to-jira-with-smart-slack-on-call-routing-12060


# Real-time uptime alerts to Jira with smart Slack on-call routing

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Real-Time Uptime Alerts to Jira with Smart Slack On-Call Routing  
**Purpose:** Receive real-time uptime/monitoring alerts via webhook, create a Jira incident for critical outages, then route a direct Slack notification to a selected on-call user based on Slack presence (active preferred; fallback if none are active).

**Typical use cases**
- Service outage (“down”) alerting with automatic incident ticket creation
- Basic on-call routing without a dedicated pager system
- Enforcing that only “actionable” alerts create tickets and notify humans

### 1.1 Input Reception & Status Gating
Receives the monitoring payload and filters so only critical alerts continue.

### 1.2 Incident Creation (Jira)
Creates a Jira issue (Task) containing all key incident fields from the alert.

### 1.3 On-Call Discovery & Presence Collection (Slack)
Fetches all members of a chosen Slack channel, loops through each member, and checks their Slack presence.

### 1.4 Smart User Selection & Notification (Slack DM)
Aggregates incident + presence data, selects one target user (random among active; else fallback), then sends a direct Slack message with incident details and the Jira ticket link/key.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Status Gating

**Overview:** Triggers on an incoming HTTP POST webhook from an uptime tool and allows only incidents with `status = down` to proceed.

**Nodes involved:**
- Receive Uptime Alert
- Filter for Critical Status

#### Node: Receive Uptime Alert
- **Type / role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point receiving the alert.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `7347e828-de5e-4368-9222-d53d9815789b` (the webhook endpoint path)
- **Expected input shape:** The workflow references `{{$json.body.<field>}}`, so the webhook payload must arrive as parsed body data containing:
  - `status`, `serviceName`, `timestamp`, `customerImpact`, `downtimeDuration`, `errorCode`, `priority`, `lastNotified` (and optionally others)
- **Outputs / connections:** Outputs to **Filter for Critical Status**.
- **TypeVersion:** 2.1
- **Common failure/edge cases:**
  - If the monitoring tool sends a non-JSON or differently shaped body, expressions like `$json.body.status` will be `undefined` and the IF check will fail (likely routing to “false” and effectively dropping the alert).
  - If webhook is not publicly reachable or authentication is needed (not configured here), alerts won’t arrive.

#### Node: Filter for Critical Status
- **Type / role:** IF node (`n8n-nodes-base.if`) — gates execution to only critical outages.
- **Key configuration:**
  - Condition: `{{ $json.body.status }}` **equals** `"down"`
  - Strict type validation enabled (by node’s condition options).
- **Outputs / connections:**
  - **True** path → **Create New Jira Incident**
  - **False** path → not connected (alert is ignored)
- **TypeVersion:** 2.2
- **Common failure/edge cases:**
  - Case-sensitivity: `"Down"` vs `"down"` will not pass.
  - Missing `body.status` results in false branch (no ticket, no notification).

---

### Block 2 — Incident Creation (Jira)

**Overview:** Creates a Jira Task in a specified project, with a summary and description built from the alert payload.

**Nodes involved:**
- Create New Jira Incident

#### Node: Create New Jira Incident
- **Type / role:** Jira node (`n8n-nodes-base.jira`) — creates an issue.
- **Key configuration choices:**
  - **Project:** “n8n sample project” (internal ID `10000`)
  - **Issue type:** “Task” (internal ID `10003`)
  - **Summary expression:** `{{ $json.body.serviceName }} {{ $json.body.timestamp }}`
  - **Description:** Multi-line incident details pulled from the webhook body:
    - service name, status, timestamp, customer impact, service owner (set to serviceName), downtime duration, error code, priority
- **Inputs / outputs:**
  - Input from **Filter for Critical Status** (true branch)
  - Output to **Get On-Call Channel Members**
- **TypeVersion:** 1
- **Credentials required:** Jira credentials (typically Jira Cloud OAuth2 or API token, depending on n8n setup).
- **Common failure/edge cases:**
  - Invalid project/issue type IDs for your Jira instance (must be updated).
  - Missing required fields in Jira configuration (some projects require additional custom fields).
  - Auth errors (expired token / insufficient permissions).
  - Summary/description expressions resolve to empty if webhook body fields are missing.

---

### Block 3 — On-Call Discovery & Presence Collection (Slack)

**Overview:** Pulls member IDs from a specific Slack channel, iterates members, checks each user’s presence, and prepares a normalized record combining presence + incident data.

**Nodes involved:**
- Get On-Call Channel Members
- Split/Process Each Member
- Check Slack User Presence
- Collect & Set Final Data

#### Node: Get On-Call Channel Members
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — reads Slack channel membership.
- **Key configuration:**
  - **Resource:** channel
  - **Operation:** member (fetch members)
  - **Channel:** “n8n” (channel ID `C09S57E2JQ2`)
- **Inputs / outputs:**
  - Input from **Create New Jira Incident**
  - Output to **Split/Process Each Member**
- **TypeVersion:** 2.3
- **Credentials required:** Slack credential (OAuth2 / bot token) with permissions to read channel membership.
- **Common failure/edge cases:**
  - Missing scopes like `channels:read`, `groups:read`, `conversations:read`, `conversations:members`.
  - Private channel access requires the bot/user be a member.
  - Large channels may paginate; depending on node behavior/version, membership retrieval may be limited (verify returned members count).

#### Node: Split/Process Each Member
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — loops over members.
- **Key configuration:** Defaults (batch size not explicitly set in parameters shown).
- **Inputs / outputs (important control-flow):**
  - Input from **Get On-Call Channel Members**
  - Two outgoing connections:
    - To **Check Slack User Presence** (per member processing)
    - To **Select Active On-Call User** (after loop completes, using accumulated items from downstream)
- **TypeVersion:** 3
- **Common failure/edge cases:**
  - If there are zero members, downstream code may fail when trying to select `users[0]`.
  - If batch size defaults to 1, it will iterate member-by-member; performance depends on Slack API rate limits.

#### Node: Check Slack User Presence
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — retrieves presence for a user.
- **Key configuration:**
  - **Resource:** user
  - **Operation:** getPresence
  - **User ID expression:** `{{ $json.member }}`
- **Inputs / outputs:**
  - Input from **Split/Process Each Member** (each item contains a `member` id)
  - Output to **Collect & Set Final Data**
- **TypeVersion:** 2.3
- **Credentials required:** Slack token with `users:read` and presence-related scope if required by your Slack plan/scopes.
- **Common failure/edge cases:**
  - Slack presence can be restricted by workspace settings; may return limited/unsupported values.
  - Rate limiting when checking many users in a short time.
  - If `member` is not a valid user ID, Slack API returns an error.

#### Node: Collect & Set Final Data
- **Type / role:** Set node (`n8n-nodes-base.set`) — normalizes data for selection and notification.
- **Key configuration choices:**
  - Creates fields combining Slack presence + alert + Jira key:
    - `presence` = `{{ $json.presence }}`
    - `member id` = `{{ $('Split/Process Each Member').item.json.member }}`
    - `service name`, `status`, `time stamp`, `customer impact`, `last notified`, `downtime duration`, `error code`, `priority` = read from **Filter for Critical Status** item body
    - `jira ticket` = `{{ $('Create New Jira Incident').item.json.key }}`
- **Inputs / outputs:**
  - Input from **Check Slack User Presence**
  - Output back to **Split/Process Each Member** (this is the “collect results and continue loop” pattern)
- **TypeVersion:** 3.4
- **Common failure/edge cases:**
  - Cross-node item referencing (`$('...').item`) assumes those nodes have an accessible item in the current execution context. If the execution model changes (or if nodes produce multiple items unexpectedly), you can get mismatched data.
  - If Jira creation fails, `jira ticket` will be undefined and the Slack message will lack a ticket key.
  - If presence isn’t returned as `active`/`away`, selection logic may not behave as expected.

---

### Block 4 — Smart User Selection & Notification (Slack DM)

**Overview:** After collecting all users with presence, a Code node selects a single on-call recipient (random among active users; otherwise first member), then sends a formatted Slack DM containing incident context and the Jira ticket key.

**Nodes involved:**
- Select Active On-Call User
- Notify Selected User

#### Node: Select Active On-Call User
- **Type / role:** Code node (`n8n-nodes-base.code`) — implements routing logic.
- **Key logic (interpreted):**
  - Builds `users = items.map(item => item.json)`
  - Filters to `activeUsers = users.filter(u => u.presence === 'active')`
  - If any active users exist: pick one randomly
  - Else: select the first user (`users[0]`) as fallback
  - Returns a single item: `{ json: selectedUser }`
- **Inputs / outputs:**
  - Input from **Split/Process Each Member** (the post-loop output, containing the collected items)
  - Output to **Notify Selected User**
- **TypeVersion:** 2
- **Common failure/edge cases:**
  - If `items` is empty (e.g., channel has no members, or upstream failed), `users[0]` is undefined → returns `{json: undefined}` causing failures downstream.
  - If presence values differ (e.g., `auto_away`, `offline`), no users will match and fallback will always pick first user.
  - Random selection is non-deterministic; may not be desirable for strict rotations.

#### Node: Notify Selected User
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — sends a direct message to a user.
- **Key configuration:**
  - **Select:** user (DM to a user)
  - **User ID:** `{{ $json["member id"] }}`
  - **Message text:** A formatted alert containing:
    - Service name, status, downtime duration, customer impact, error code, Jira ticket, last notified, priority
  - `includeLinkToWorkflow` disabled
- **Inputs / outputs:**
  - Input from **Select Active On-Call User**
  - Output: end of workflow
- **TypeVersion:** 2.3
- **Credentials required:** Slack bot/user token with permission to DM users (`chat:write`, and ability to open IMs if needed).
- **Common failure/edge cases:**
  - Bot may not be allowed to DM users unless they interacted before or workspace settings allow it.
  - If `member id` is missing/undefined, Slack API call fails.
  - Message fields referencing keys with spaces require bracket notation (correctly used here for several fields); if field names change, expressions must be updated.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Uptime Alert | Webhook | Entry point: receive monitoring alert | — | Filter for Critical Status | ### Start Alert & Check Status<br>The workflow is triggered by an external webhook alert containing incident details. It immediately filters the incoming payload based on the alert's status (e.g., 'down') to ensure only critical, actionable events proceed to the next stage. |
| Filter for Critical Status | IF | Gate: only process `status=down` | Receive Uptime Alert | Create New Jira Incident | ### Start Alert & Check Status<br>The workflow is triggered by an external webhook alert containing incident details. It immediately filters the incoming payload based on the alert's status (e.g., 'down') to ensure only critical, actionable events proceed to the next stage. |
| Create New Jira Incident | Jira | Create Jira Task for incident tracking | Filter for Critical Status | Get On-Call Channel Members | ### Create Ticket in Jira<br>Upon validation, a detailed Jira incident is instantly created. It uses dynamic data from the alert, including service name, downtime duration, and error code, to ensure all necessary context is logged in a new ticket for tracking. |
| Get On-Call Channel Members | Slack | Fetch member IDs from on-call channel | Create New Jira Incident | Split/Process Each Member | ### Find Available Team Member<br>The workflow fetches all member IDs from the designated on-call Slack channel. It then loops through each individual member ID to check their current Slack presence status (active or away), preparing a filtered list for routing. |
| Split/Process Each Member | SplitInBatches | Iterate through Slack members + coordinate loop | Get On-Call Channel Members; Collect & Set Final Data | Check Slack User Presence; Select Active On-Call User | ### Find Available Team Member<br>The workflow fetches all member IDs from the designated on-call Slack channel. It then loops through each individual member ID to check their current Slack presence status (active or away), preparing a filtered list for routing. |
| Check Slack User Presence | Slack | Query each member presence | Split/Process Each Member | Collect & Set Final Data | ### Find Available Team Member<br>The workflow fetches all member IDs from the designated on-call Slack channel. It then loops through each individual member ID to check their current Slack presence status (active or away), preparing a filtered list for routing. |
| Collect & Set Final Data | Set | Merge incident + Jira key + member presence into one record | Check Slack User Presence | Split/Process Each Member | ### Send Alert to User<br>All incident data and user statuses are combined for processing. A JavaScript code node implements the rotation logic to select a single active user. Finally, a direct Slack message is sent to that chosen on-call team member for immediate action. |
| Select Active On-Call User | Code | Choose one recipient (random among active; else fallback) | Split/Process Each Member | Notify Selected User | ### Send Alert to User<br>All incident data and user statuses are combined for processing. A JavaScript code node implements the rotation logic to select a single active user. Finally, a direct Slack message is sent to that chosen on-call team member for immediate action. |
| Notify Selected User | Slack | DM selected on-call user with incident details | Select Active On-Call User | — | ### Send Alert to User<br>All incident data and user statuses are combined for processing. A JavaScript code node implements the rotation logic to select a single active user. Finally, a direct Slack message is sent to that chosen on-call team member for immediate action. |
| Sticky Note | Sticky Note | Comment block | — | — | ## How it Work<br>This workflow starts when your uptime monitoring tool sends a webhook alert to n8n. The alert payload is checked to ensure the service status is down. Only critical outages pass through to the next stage. Once the alert is validated, a Jira Task is automatically created using all the incident details such as service name, downtime duration, error code, customer impact, and timestamp.<br><br>After creating the Jira incident, the workflow retrieves all member IDs from your designated Slack on-call rotation channel. Each member’s Slack presence is checked individually. The workflow then applies smart selection logic: if multiple users are active, it randomly picks one; if only one is active, that user is selected; and if no one is active, it falls back to the first member.<br><br>## Setup Steps<br>1.Import the workflow JSON into n8n.<br>2.Configure your Webhook, Slack, and Jira credentials.<br>3.Update the IF node to match your desired critical status (already set to "down").<br>4.Set your Jira project and issue type.<br>5.Select your Slack on-call channel.<br>6.Activate the workflow and send a test alert using Postman or your monitoring tool. |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note4 | Sticky Note | Comment block | — | — |  |

> Note: Sticky notes are included as nodes by n8n; they do not participate in execution flow.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Path: generate or set to a fixed value (match your monitoring tool configuration)
   - Ensure your monitoring tool sends a JSON body containing at least: `status`, `serviceName`, `timestamp`, `customerImpact`, `downtimeDuration`, `errorCode`, `priority`, `lastNotified`.

2. **Add critical-status filter**
   - Add node: **IF**
   - Condition (String): `{{$json.body.status}}` **equals** `down`
   - Connect: **Webhook → IF**

3. **Create Jira incident**
   - Add node: **Jira** (operation: **Create Issue**)
   - Configure **Credentials:** Jira Cloud (OAuth2) or API token (per your n8n setup)
   - Set:
     - Project: your target project
     - Issue Type: Task (or your incident type)
     - Summary: `{{$json.body.serviceName}} {{$json.body.timestamp}}`
     - Description: include dynamic fields from `$json.body.*` as needed
   - Connect: **IF (true) → Jira**

4. **Fetch Slack channel members**
   - Add node: **Slack**
   - Configure **Credentials:** Slack OAuth2/bot token with scopes to read channel members and query presence
   - Resource: **Channel**
   - Operation: **Member** (list members)
   - Channel: pick your on-call channel
   - Connect: **Jira → Slack (channel members)**

5. **Loop through members**
   - Add node: **Split In Batches**
   - Keep defaults (or set Batch Size = 1 for per-user checks)
   - Connect: **Slack (channel members) → Split In Batches**

6. **Check each user’s presence**
   - Add node: **Slack**
   - Resource: **User**
   - Operation: **Get Presence**
   - User: `{{$json.member}}`
   - Connect: **Split In Batches → Slack (get presence)**

7. **Normalize/merge data for each member**
   - Add node: **Set**
   - Add fields (names can match exactly to preserve later expressions):
     - `presence` = `{{$json.presence}}`
     - `member id` = `{{$('Split In Batches').item.json.member}}`
     - `service name` = `{{$('IF').item.json.body.serviceName}}`
     - `status` = `{{$('IF').item.json.body.status}}`
     - `time stamp` = `{{$('IF').item.json.body.timestamp}}`
     - `customer impact` = `{{$('IF').item.json.body.customerImpact}}`
     - `last notified` = `{{$('IF').item.json.body.lastNotified}}`
     - `downtime duration` = `{{$('IF').item.json.body.downtimeDuration}}`
     - `error code` = `{{$('IF').item.json.body.errorCode}}`
     - `priority` = `{{$('IF').item.json.body.priority}}`
     - `jira ticket` = `{{$('Jira').item.json.key}}`
   - Connect: **Slack (get presence) → Set**
   - Connect loopback: **Set → Split In Batches** (so it continues with next member)

8. **Select one target user after loop completion**
   - Add node: **Code**
   - Paste logic (equivalent):
     - Filter users with `presence === 'active'`
     - Randomly select among active users; else select first user
     - Return one item only
   - Connect: **Split In Batches (done output) → Code**
     - In many n8n patterns, the Split In Batches node’s first output is the “continue” and the second path is for per-item processing; ensure the “after all batches” path goes to the Code node (matching your n8n UI behavior).

9. **Send Slack DM to selected user**
   - Add node: **Slack**
   - Operation: **Post** (message)
   - Target: **User** (DM)
   - User ID: `{{$json["member id"]}}`
   - Text: include incident fields (use bracket notation for keys with spaces)
   - Connect: **Code → Slack (DM)**

10. **Activate and test**
   - Execute webhook test URL with a sample payload where `status = "down"`.
   - Verify:
     - Jira issue created successfully
     - Slack channel membership fetched
     - Presence checks succeed
     - A DM is sent to one chosen user

**Credential requirements (minimum)**
- **Jira:** permission to create issues in the target project/issue type.
- **Slack:** permissions/scopes to:
  - read channel members (conversations:read + conversations:members or equivalent)
  - read users / presence (users:read; presence scope depending on Slack)
  - send DMs (chat:write; and ability to open IM if required)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## How it Work … (full sticky note content embedded in workflow) | Internal workflow documentation (Sticky Note4) |
| ### Start Alert & Check Status … | Explains webhook trigger + outage filtering (Sticky Note) |
| ### Create Ticket in Jira … | Explains Jira creation step (Sticky Note1) |
| ### Find Available Team Member … | Explains Slack member + presence collection (Sticky Note2) |
| ### Send Alert to User … | Explains selection logic + Slack DM (Sticky Note3) |