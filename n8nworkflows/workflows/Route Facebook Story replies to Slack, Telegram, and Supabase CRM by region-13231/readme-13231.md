Route Facebook Story replies to Slack, Telegram, and Supabase CRM by region

https://n8nworkflows.xyz/workflows/route-facebook-story-replies-to-slack--telegram--and-supabase-crm-by-region-13231


# Route Facebook Story replies to Slack, Telegram, and Supabase CRM by region

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Route Facebook Story replies to Slack, Telegram, and Supabase CRM by region  
**Workflow name (in JSON):** Facebook Story Replies → Telegram “Human Required” Routing + Supabase CRM

**Purpose:**  
This workflow ingests **Facebook Story reply events** via an n8n **Webhook**, normalizes the payload into a consistent CRM-friendly schema, stores it in **Supabase**, performs a **time-based routing** decision (Asia / Europe / unassigned), and then notifies operators via **Telegram and Slack**. It also updates the Supabase record to reflect assignment status.

**Typical use cases:**
- Centralizing and tracking Story reply leads/messages in a CRM table
- Automatic on-call routing by time window
- Human escalation outside support hours

### Logical Blocks
**1.1 Incoming Event Reception (Webhook)**  
Receives raw POST payload from Facebook and passes it downstream.

**1.2 Normalize & Persist (Normalize → Insert to Supabase)**  
Standardizes JSON fields (sender_id, message, timestamps, raw payload) and inserts a new row in Supabase with an initial status.

**1.3 Time-Based Assignment (Time-Zone Router → IF)**  
Determines assignment based on current UTC hour and branches assigned vs unassigned.

**1.4 Notifications & Status Updates (Supabase update → Telegram → Slack)**  
Updates the stored record and alerts teams via Telegram and Slack depending on assignment outcome.

---

## 2. Block-by-Block Analysis

### 2.1 Incoming Event Reception (Webhook)

**Overview:**  
Captures Facebook Story reply events in real time through an HTTP endpoint hosted by n8n. This block is the workflow entrypoint.

**Nodes involved:**
- Facebook Story Reply Webhook

#### Node: Facebook Story Reply Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow trigger / HTTP ingress.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `/facebook-story-reply`
  - Produces the incoming request data under `items[0].json` (typically includes `body`, headers, etc., depending on n8n version/settings).
- **Key variables/expressions:** none.
- **Connections:**
  - **Output →** Normalize Story Reply Payload
- **Version-specific notes:** TypeVersion 1 (older webhook node behavior). Ensure your n8n version still matches expected output structure (notably where `body` is placed).
- **Failure/edge cases:**
  - Facebook signature verification is **not implemented** in this workflow (risk of spoofed requests).
  - If Facebook sends a payload format different from `body.entry[0].messaging[0]`, downstream normalization may produce nulls.
  - If webhook is not publicly reachable or SSL issues exist, Facebook will fail delivery.

**Sticky note context (applies to this block):**
- “Incoming Facebook Story Replies … receives … in real time using a webhook … captures sender ID, message content, story ID, and timestamp …”

---

### 2.2 Normalize, Store & Assign Replies (Normalization → Insert → Router)

**Overview:**  
Normalizes the incoming Facebook payload into a consistent schema, stores it in Supabase for traceability, then assigns the message to a region/team based on current UTC hour.

**Nodes involved:**
- Normalize Story Reply Payload
- Insert Story Reply into Supabase
- Time-Zone Router

#### Node: Normalize Story Reply Payload
- **Type / role:** `Function` — transforms raw webhook payload into a clean record.
- **Configuration choices:**
  - Reads: `items[0].json.body`
  - Extracts:
    - `entry = body.entry[0]`
    - `messaging = entry.messaging[0]`
  - Outputs a single item with fields:
    - `platform`: `"facebook_story"`
    - `sender_id`: `messaging.sender.id` or `null`
    - `sender_name`: `null` (explicitly, because replies don’t include name by default)
    - `message`: `messaging.message.text` or `null`
    - `story_id`: `messaging.message.mid` or `null` (note: using message mid as story_id)
    - `received_at`: ISO string derived from `messaging.timestamp` else now
    - `status`: `"pending"`
    - `raw_payload`: full original `body`
- **Key expressions/variables:**
  - Uses optional chaining (`?.`) heavily to avoid hard crashes on missing fields.
  - `new Date(messaging.timestamp).toISOString()` assumes timestamp is compatible with JS Date constructor.
- **Connections:**
  - **Input ←** Facebook Story Reply Webhook
  - **Output →** Insert Story Reply into Supabase
- **Edge cases / failures:**
  - If the webhook data is not under `items[0].json.body` (some configurations put it in `items[0].json` directly), `body` becomes undefined → record will be mostly null.
  - If `messaging.timestamp` is in seconds (not milliseconds), the ISO date will be wrong (far in the past). Facebook often uses milliseconds, but verify.
  - `story_id` mapped to `mid` might not match your CRM concept of a story; consider storing both `mid` and any story reference if available.

#### Node: Insert Story Reply into Supabase
- **Type / role:** `Supabase` — inserts a new row in `facebook_story_replies`.
- **Configuration choices:**
  - **Table:** `facebook_story_replies`
  - **Operation:** (implicit insert; node is configured with fields, no explicit operation field shown, but node name and config indicate insert)
  - **Fields mapped:**
    - platform, story_id, sender_id, sender_name, message, status, received_at
- **Credentials:**
  - `Supabase account 3` (must include Supabase URL + service role key or appropriate API key).
- **Connections:**
  - **Input ←** Normalize Story Reply Payload
  - **Output →** Time-Zone Router
- **Edge cases / failures:**
  - Table missing or column mismatch (common with Supabase schema drift).
  - RLS policies: if using anon key with RLS enabled, insert may fail with 401/403.
  - Insert response shape matters: downstream nodes later reference `{{$json.id}}`. This assumes Supabase returns the inserted row including `id`. If it only returns minimal data, update steps will fail.

#### Node: Time-Zone Router
- **Type / role:** `Function` — assigns routing based on **current UTC hour**.
- **Configuration choices:**
  - Computes: `const hour = new Date().getUTCHours();`
  - Rules:
    - **06–14 UTC (inclusive):** assigned to `support_asia`, status becomes `assigned`
    - **>14–22 UTC (inclusive):** assigned to `support_europe`, status becomes `assigned`
    - Otherwise: status becomes `unassigned` (and **no assigned_to** set)
- **Connections:**
  - **Input ←** Insert Story Reply into Supabase
  - **Output →** Assignment Successful?
- **Edge cases / failures:**
  - Uses **workflow execution time**, not message received time. If there’s delay/queueing, assignment may not reflect actual receipt time.
  - Boundary hours: hour==14 goes to Asia, hour==15 goes to Europe; confirm desired.
  - Unassigned branch leaves `assigned_to` undefined; Slack “unassigned” message template still prints `Assigned To: {{ $json.assigned_to }}` (will appear blank/undefined).

**Sticky note context (applies to this block):**
- “Normalize, Store & Assign Replies … cleans … saves into Supabase … assigns … based on current time zone …”

---

### 2.3 Assignment Decision (IF)

**Overview:**  
Splits processing into “assigned” vs “unassigned” based on the `status` field created by the router.

**Nodes involved:**
- Assignment Successful?

#### Node: Assignment Successful?
- **Type / role:** `IF` — branching logic.
- **Configuration choices:**
  - Condition: **String equals**
    - `value1 = {{$json.status}}`
    - `value2 = assigned`
  - **True output (0):** assigned path
  - **False output (1):** unassigned path
- **Connections:**
  - **Input ←** Time-Zone Router
  - **True →** Update Supabase – Assigned
  - **False →** Update Supabase – Unassigned
- **Edge cases / failures:**
  - If `status` is null/undefined (bad payload or earlier node failure), goes to unassigned path.
  - If you later add more statuses (e.g., “queued”), they will also fall into unassigned unless handled.

---

### 2.4 Notify Teams & Update Status (Supabase Update → Telegram → Slack)

**Overview:**  
Updates the Supabase record after assignment evaluation, then notifies stakeholders. Assigned replies produce “assigned” messages; unassigned replies trigger “action required” alerts.

**Nodes involved:**
- Update Supabase – Assigned
- Telegram Notify – Human Assigned
- Slack Notify – Human Assigned
- Update Supabase – Unassigned
- Telegram Alert – Unassigned Story Reply
- Slack Alert – Unassigned Story Reply

#### Node: Update Supabase – Assigned
- **Type / role:** `Supabase` — updates the inserted row to reflect assignment metadata.
- **Configuration choices:**
  - **Operation:** update
  - **Table:** `facebook_story_replies`
  - **Filter:** `id eq {{$json.id}}`
  - Updates many fields, including:
    - id, platform, story_id, sender_id, sender_name, message
    - status, assigned_to, received_at
    - updated_at, raw_payload
- **Key expressions/variables:**
  - Relies on `{{$json.id}}` being present from the Supabase insert result (or propagated throughout).
  - Uses `{{$json.updated_at}}` but no node sets it; likely becomes null unless Supabase auto-fills via trigger/default.
- **Connections:**
  - **Input ←** Assignment Successful? (true branch)
  - **Output →** Telegram Notify – Human Assigned
- **Edge cases / failures:**
  - If `id` is missing, filter becomes invalid → no row updated.
  - If `raw_payload` is JSON and the column type is not JSONB (or equivalent), update may fail.
  - Updating `id` field explicitly is usually unnecessary; could be rejected depending on schema.

#### Node: Telegram Notify – Human Assigned
- **Type / role:** `Telegram` — sends an “assigned” notification.
- **Configuration choices:**
  - **Chat ID:** `5758325294` (expression but constant)
  - **Parse mode:** Markdown
  - Message includes assigned_to, message, status.
- **Credentials:** `Telegram Personal Account 0001` (Telegram API token / bot).
- **Connections:**
  - **Input ←** Update Supabase – Assigned
  - **Output →** Slack Notify – Human Assigned
- **Edge cases / failures:**
  - Markdown formatting: if message text contains characters that break Markdown, rendering may be odd.
  - Bot must have permission to post in the target chat (private chat/group/channel rules apply).

#### Node: Slack Notify – Human Assigned
- **Type / role:** `Slack` — posts “assigned” message to a Slack channel.
- **Configuration choices:**
  - **Channel selection:** by channelId
  - **Channel ID:** `C09S57E2JQ2` (cached name: “n8n”)
  - Text includes sender_id, story_id, message, assigned_to, received_at.
  - `includeLinkToWorkflow: false`
- **Credentials:** `Slack Mobile 1`
- **Connections:**
  - **Input ←** Telegram Notify – Human Assigned
  - **Output:** none (end of assigned branch)
- **Edge cases / failures:**
  - Slack app scopes missing (e.g., `chat:write`) or channel not authorized.
  - `sender_name` is null by design; Slack message will show blank customer.

#### Node: Update Supabase – Unassigned
- **Type / role:** `Supabase` — intended to update record for unassigned status.
- **Configuration choices:**
  - **Table:** `facebook_story_replies`
  - **Operation:** update
  - **Important:** No filters and no field mappings are configured in the JSON shown.
- **Connections:**
  - **Input ←** Assignment Successful? (false branch)
  - **Output →** Telegram Alert – Unassigned Story Reply
- **Edge cases / failures:**
  - **High risk of misconfiguration:** without a filter, the node may fail validation, do nothing, or (worst case depending on node behavior/version) attempt a broad update. In practice, Supabase update usually requires filters; verify in UI.
  - Should likely mirror “Assigned” update behavior: filter by `id` and update `status='unassigned'` plus timestamps.

#### Node: Telegram Alert – Unassigned Story Reply
- **Type / role:** `Telegram` — escalation alert for manual action.
- **Configuration choices:**
  - **Chat ID:** `5758325294`
  - **Parse mode:** Markdown
  - Message includes only message text and “Action Required”
- **Credentials:** `Telegram Personal Account 0001`
- **Connections:**
  - **Input ←** Update Supabase – Unassigned
  - **Output →** Slack Alert – Unassigned Story Reply
- **Edge cases / failures:**
  - Same Markdown caveats as above.

#### Node: Slack Alert – Unassigned Story Reply
- **Type / role:** `Slack` — escalation post to Slack channel.
- **Configuration choices:**
  - **Channel:** `C09S57E2JQ2`
  - Text says “New Facebook Story Reply Assigned:” but this is the unassigned path; message content appears mismatched to purpose.
  - Includes assigned_to (likely undefined), sender_name (null), etc.
- **Credentials:** `Slack Mobile 1`
- **Connections:**
  - **Input ←** Telegram Alert – Unassigned Story Reply
  - **Output:** none (end of unassigned branch)
- **Edge cases / failures:**
  - Copy mismatch (“Assigned” wording) can confuse operators; adjust text to “Unassigned”.

**Sticky note context (applies to this block):**
- “Notify Teams & Update Status … sends notifications … Assigned replies notify … unassigned replies trigger alerts … Supabase records updated …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Facebook Story Reply Webhook | Webhook | Receive Facebook story reply events | — | Normalize Story Reply Payload | Incoming Facebook Story Replies — receives Facebook Story reply events in real time using a webhook… captures sender ID, message content, story ID, and timestamp… |
| Normalize Story Reply Payload | Function | Normalize raw payload into standard schema | Facebook Story Reply Webhook | Insert Story Reply into Supabase | Normalize, Store & Assign Replies — cleans and standardizes… saves into Supabase… assigns… based on the current time zone… |
| Insert Story Reply into Supabase | Supabase | Insert CRM record for the reply | Normalize Story Reply Payload | Time-Zone Router | Normalize, Store & Assign Replies — cleans and standardizes… saves into Supabase… assigns… based on the current time zone… |
| Time-Zone Router | Function | Assign to support team based on UTC hour | Insert Story Reply into Supabase | Assignment Successful? | Normalize, Store & Assign Replies — cleans and standardizes… saves into Supabase… assigns… based on the current time zone… |
| Assignment Successful? | IF | Branch assigned vs unassigned | Time-Zone Router | Update Supabase – Assigned; Update Supabase – Unassigned | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Update Supabase – Assigned | Supabase | Update record with assigned status/owner | Assignment Successful? (true) | Telegram Notify – Human Assigned | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Telegram Notify – Human Assigned | Telegram | Notify Telegram for assigned replies | Update Supabase – Assigned | Slack Notify – Human Assigned | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Slack Notify – Human Assigned | Slack | Notify Slack for assigned replies | Telegram Notify – Human Assigned | — | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Update Supabase – Unassigned | Supabase | Update record for unassigned outcome (currently incomplete) | Assignment Successful? (false) | Telegram Alert – Unassigned Story Reply | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Telegram Alert – Unassigned Story Reply | Telegram | Escalation alert for manual handling | Update Supabase – Unassigned | Slack Alert – Unassigned Story Reply | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Slack Alert – Unassigned Story Reply | Slack | Escalation alert in Slack | Telegram Alert – Unassigned Story Reply | — | Notify Teams & Update Status — sends notifications to Telegram and Slack… Supabase records are updated… |
| Sticky Note | Sticky Note | Documentation | — | — |  |
| Sticky Note1 | Sticky Note | Documentation | — | — |  |
| Sticky Note2 | Sticky Note | Documentation | — | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a Webhook trigger**
- Add node: **Webhook**
- Set:
  - **HTTP Method:** POST
  - **Path:** `facebook-story-reply`
- Activate later after testing. Copy the production URL for Facebook.

2) **Add “Normalize Story Reply Payload” (Function node)**
- Add node: **Function**
- Paste logic that:
  - Reads `items[0].json.body`
  - Extracts `entry[0].messaging[0]`
  - Outputs fields: `platform, sender_id, sender_name=null, message, story_id, received_at ISO, status='pending', raw_payload`
- Connect: **Webhook → Function**

3) **Add Supabase insert**
- Add node: **Supabase**
- Credentials:
  - Configure **Supabase API** credential (Supabase URL + key with insert rights; service role key recommended for server-side automations).
- Configure:
  - **Table:** `facebook_story_replies`
  - **Operation:** Insert (create)
  - Map fields:
    - `platform = {{$json.platform}}`
    - `story_id = {{$json.story_id}}`
    - `sender_id = {{$json.sender_id}}`
    - `sender_name = {{$json.sender_name}}`
    - `message = {{$json.message}}`
    - `status = {{$json.status}}`
    - `received_at = {{$json.received_at}}`
- Connect: **Normalize → Supabase Insert**
- Ensure the insert response returns the inserted `id` (enable “returning” / “select” options if your node version requires it).

4) **Add “Time-Zone Router” (Function node)**
- Add node: **Function**
- Implement:
  - `hour = new Date().getUTCHours()`
  - If 06–14 ⇒ `assigned_to='support_asia'`, `status='assigned'`
  - If 15–22 ⇒ `assigned_to='support_europe'`, `status='assigned'`
  - Else ⇒ `status='unassigned'`
- Connect: **Supabase Insert → Time-Zone Router**

5) **Add IF node “Assignment Successful?”**
- Add node: **IF**
- Condition:
  - String: `{{$json.status}}` equals `assigned`
- Connect: **Time-Zone Router → IF**

6) **Assigned branch: update Supabase**
- Add node: **Supabase** named “Update Supabase – Assigned”
- Configure:
  - **Operation:** Update
  - **Table:** `facebook_story_replies`
  - **Filter:** `id eq {{$json.id}}`
  - Set fields to update at minimum:
    - `status = {{$json.status}}`
    - `assigned_to = {{$json.assigned_to}}`
    - (optional) `raw_payload = {{$json.raw_payload}}`
    - (optional) `received_at = {{$json.received_at}}`
- Connect: **IF (true) → Update Supabase – Assigned**

7) **Assigned branch: Telegram notify**
- Add node: **Telegram**
- Credentials: configure Telegram bot token (must be able to send to the target chat)
- Set:
  - **Chat ID:** your destination (e.g., `5758325294`)
  - **Parse mode:** Markdown
  - Text template referencing `assigned_to`, `message`, etc.
- Connect: **Update Supabase – Assigned → Telegram Notify – Human Assigned**

8) **Assigned branch: Slack notify**
- Add node: **Slack**
- Credentials: Slack OAuth with `chat:write` scope
- Choose **Channel** (e.g., `#n8n`) and set message template.
- Connect: **Telegram Notify – Human Assigned → Slack Notify – Human Assigned**

9) **Unassigned branch: fix/update Supabase (important)**
- Add node: **Supabase** named “Update Supabase – Unassigned”
- Configure properly (recommended):
  - **Operation:** Update
  - **Table:** `facebook_story_replies`
  - **Filter:** `id eq {{$json.id}}`
  - Update fields:
    - `status = 'unassigned'` (or `{{$json.status}}`)
    - (optional) `raw_payload`, `received_at`
- Connect: **IF (false) → Update Supabase – Unassigned**

10) **Unassigned branch: Telegram alert**
- Add node: **Telegram**
- Configure chat ID + Markdown message “Unassigned… Action Required”
- Connect: **Update Supabase – Unassigned → Telegram Alert – Unassigned Story Reply**

11) **Unassigned branch: Slack alert**
- Add node: **Slack**
- Configure channel + text that clearly indicates **Unassigned** (adjust wording from “Assigned”)
- Connect: **Telegram Alert – Unassigned Story Reply → Slack Alert – Unassigned Story Reply**

12) **Facebook configuration**
- In your Facebook App / webhook subscription:
  - Point to the n8n webhook production URL + `/facebook-story-reply`
  - Configure verification (token handshake) if required by Facebook (may require additional GET handler or separate workflow).
  - Consider validating `X-Hub-Signature` / app secret proof before accepting events.

13) **Activate and test**
- Use n8n webhook test URL first, send a sample payload shaped like `body.entry[0].messaging[0]`.
- Confirm:
  - Supabase row is created and `id` is returned
  - Assigned/unassigned routing works by changing current UTC hour (or temporarily hardcode hour)
  - Telegram and Slack messages render correctly

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works” + “Setup Steps” description embedded in workflow | Internal documentation included as a sticky note; covers webhook ingestion, normalization, Supabase storage, UTC routing, notifications, testing, activation |
| Security note: no signature verification present | Consider adding verification for Facebook webhook authenticity (app secret / signature headers) before processing |
| Data completeness note: `sender_name` is always null | Facebook Story replies typically do not include profile name; enrich via Graph API if required |
| Supabase unassigned update node appears incomplete | Configure filter by `id` and set fields explicitly to avoid failed/no-op update |

