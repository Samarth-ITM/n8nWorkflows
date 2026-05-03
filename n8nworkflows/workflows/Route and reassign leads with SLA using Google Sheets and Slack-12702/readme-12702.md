Route and reassign leads with SLA using Google Sheets and Slack

https://n8nworkflows.xyz/workflows/route-and-reassign-leads-with-sla-using-google-sheets-and-slack-12702


# Route and reassign leads with SLA using Google Sheets and Slack

## 1. Workflow Overview

**Title:** Route and reassign leads with SLA using Google Sheets and Slack  
**Workflow name (internal):** Lead Routing Engine with SLA Auto-Reassignment

**Purpose:**  
This workflow ingests new leads, assigns them to sales reps using **round-robin**, enforces a **1-hour SLA** to contact the lead, **auto-reassigns** uncontacted leads every hour, and escalates to a manager after repeated SLA breaches. Google Sheets acts as the datastore; Slack is used for notification and interaction (“Mark as CONTACTED” button).

### 1.1 Intake & Initial Assignment (Round Robin)
Triggered by an n8n Form. Normalizes lead data, reads the active sales roster from Google Sheets, assigns the next rep using a stored `last_index`, writes lead + routing state back to Sheets, then notifies Slack with an action button.

### 1.2 SLA Monitor & Hourly Re-route
Triggered hourly. Pulls all `NEW` leads, checks whether the last routing timestamp is ≥ 1 hour ago, re-routes eligible leads to the next rep, updates the lead row, and notifies Slack. If reroutes reach a threshold (>=10), it performs a **one-time escalation** to a manager (anti-spam via an escalation flag in `routing_state`).

### 1.3 “CONTACTED” Stage Update API (Stop Routing)
Triggered by a webhook endpoint meant to receive Slack interactive button payloads. It parses the Slack payload, looks up the lead, authorizes the click (must be the currently assigned rep and lead must still be `NEW`), updates stage to `CONTACTED` (idempotent gate), otherwise sends Slack feedback via `response_url`.

---

## 2. Block-by-Block Analysis

### Block A — Intake & Initial Assignment (Round Robin)

**Overview:**  
Creates a normalized lead record, loads the active sales roster, assigns the lead using round-robin based on a persisted `last_index`, stores the lead in Google Sheets, and posts a Slack message with an interactive “Mark as CONTACTED” button.

**Nodes involved:**
- 1) Form Trigger (New Lead)
- 2) Prep Lead
- 3) Get Sales List (active=ON)
- 4) Build Sales List
- 5) Get routing_state (last_index)
- 6) Initial Assign (Round Robin)
- 7) Update routing_state
- 8) Upsert Lead (leads sheet)
- 9) Slack Notify (New Lead) & Button

#### 1) Form Trigger (New Lead)
- **Type / role:** `formTrigger` — entry point for lead intake.
- **Configuration:** Form titled “Lead Intake (Generic Form Trigger)” with required fields:
  - `Name`
  - `Phone` (placeholder indicates Indonesian formats like `0812...`, `+62812...`, `62812...`)
- **Outputs:** Emits form submission JSON with keys `Name`, `Phone`.
- **Edge cases / failures:** None typical besides missing required fields (handled by the form UI).

#### 2) Prep Lead
- **Type / role:** `code` — normalizes and enriches the lead record.
- **Key logic:**
  - `normalizePhone()`:
    - removes whitespace and non-numeric characters (except `+`)
    - converts:
      - `+62xxxx` → `62xxxx`
      - `0xxxx` → `62xxxx`
      - numeric-only → prefixed with `62`
  - `makeLeadId()` generates `L-<ts>-<rnd>` in uppercase using base36 timestamp + random suffix.
  - Sets initial operational fields:
    - `stage: "NEW"`
    - `assigned_sales`, `sales_slack_id`: empty placeholders
    - `route_count: 0`
    - timestamps: `created_at`, and blank `initial_assigned_at/assigned_at/last_routed_at`
- **Input:** Form data from node 1 (`$json.Name`, `$json.Phone`)
- **Output:** Single lead object with normalized structure.
- **Edge cases / failures:**
  - If phone is in an unexpected format, it may return a partially normalized value (still passes through).

#### 3) Get Sales List (active=ON)
- **Type / role:** `googleSheets` — reads the sales roster sheet.
- **Configuration:** Reads from Google Spreadsheet (documentId redacted), sheet name cached as `sales_sheet`.
- **Output:** Rows including expected columns `name`, `email`, `slack_id`, `active`.
- **Failures:** OAuth/permission errors, sheet name mismatch, API quotas.

#### 4) Build Sales List
- **Type / role:** `code` — filters and structures active sales reps.
- **Logic:**
  - Keeps rows where `active` is `ON` / `TRUE` / boolean `true`
  - Maps to `{ name, email, slack_id, active: true }`
  - Drops entries missing `name` or `slack_id`
  - Throws error if empty: `No active sales found...`
- **Output:** `{ sales_list: [...], sales_count }`
- **Edge cases:**
  - If sheet headers differ (e.g., `Slack ID` instead of `slack_id`), list becomes empty and node throws.

#### 5) Get routing_state (last_index)
- **Type / role:** `googleSheets` — reads a persisted pointer for round-robin.
- **Configuration:** Filter `key == "last_index"` in `routing_state_sheet`.
- **Output:** Typically `{ key: "last_index", value: "<number>" }`
- **Edge cases:**
  - If the key doesn’t exist, node 6 uses default `-1` (via `?? -1`) as long as the node returns an item; if the node returns *no items*, behavior depends on n8n item handling—this workflow assumes at least one item exists.

#### 6) Initial Assign (Round Robin)
- **Type / role:** `code` — chooses next rep and merges assignment into lead.
- **Key configuration choice:** Reads the lead explicitly from node 2 to avoid overwritten `$json`:
  - `const lead = $items('2) Prep Lead')[0].json;`
  - sales list from node 4
  - lastIndex from node 5: `Number(rs.value ?? -1)`
- **Assignment logic:**
  - `nextIndex = (lastIndex + 1) % sales.length`
  - sets `assigned_sales`, `sales_slack_id`, `assigned_sales_index`
  - updates timestamps `initial_assigned_at` (only if blank), `assigned_at`, `last_routed_at`
  - increments `route_count`
- **Outputs:** The assigned/enriched lead record.
- **Failures / edge cases:**
  - If node 5 returns no item, `$items('5)...[0]` could be undefined; code uses optional chaining for `[0]?.json` but still assumes at least a routing object exists. Consider ensuring `routing_state_sheet` always has `last_index`.
  - If sales list is empty, modulo by 0 would fail—but node 4 already throws earlier.

#### 7) Update routing_state
- **Type / role:** `googleSheets` — writes the new `last_index` pointer.
- **Operation:** `update` with matching on `key`.
- **Mapped values:**
  - `key = {{$json.routing_key}}` (always `last_index`)
  - `value = {{$json.assigned_sales_index}}`
- **Edge cases:**
  - If no matching row exists for `last_index`, `update` may fail or update nothing depending on node behavior. Using `appendOrUpdate` could be safer if the row might not exist.

#### 8) Upsert Lead (leads sheet)
- **Type / role:** `googleSheets` — persists lead record.
- **Operation:** `appendOrUpdate` matching on `lead_id`.
- **Key mapped fields:** `lead_id, name, phone, stage, created_at, assigned_at, initial_assigned_at, last_routed_at, route_count, assigned_sales, sales_slack_id`
- **Failures:** schema mismatch, permission issues, quota.
- **Edge cases:** If `lead_id` duplicates (unlikely but possible), it updates.

#### 9) Slack Notify (New Lead) & Button
- **Type / role:** `slack` — posts Slack message with interactive button.
- **Configuration:** Sends a **Block Kit** message to a channel (channel ID redacted).
- **Message content:** Alerts new lead assigned; includes:
  - mentions assigned user `<@{{$json.sales_slack_id}}>`
  - “Mark as CONTACTED” button with:
    - `value = {{$json.lead_id}}`
    - `action_id = "mark_contacted"`
- **Important integration requirement:** Slack interactive button clicks must be configured in Slack to POST to the workflow’s stage webhook endpoint (Block C).
- **Edge cases:**
  - If `sales_slack_id` is empty/invalid, mention won’t work and authorization will later fail.

---

### Block B — SLA Monitor, Auto Re-route & Escalation

**Overview:**  
Runs hourly, finds `NEW` leads, reassigns those untouched for ≥1 hour, updates Google Sheets, notifies Slack, and escalates after too many reroutes with a one-time flag to prevent spam.

**Nodes involved:**
- SLA Trigger (every 1 hour)
- SLA) Get Sales List
- SLA) Build Sales List
- SLA) Get Leads (stage=NEW)
- SLA) If last route >= 1 hour
- SLA) Re-route to next sales
- IF route_count >= 10
- SLA) Update lead (after reroute)
- SLA) Slack Notify (reroute) & Button
- Set Manager id
- ESC) Get escalation flag
- ESC) Normalize flag result
- ESC) If not escalated yet
- 🚨 SLACK ESCALATION
- ESC) Set escalation flag

#### SLA Trigger (every 1 hour)
- **Type / role:** `scheduleTrigger` — periodic entry point.
- **Configuration:** Runs every 1 hour.
- **Adjusting SLA:** This schedule cadence and the IF time comparison together define the SLA behavior.

#### SLA) Get Sales List / SLA) Build Sales List
- **Type / role:** Google Sheets + Code — same roster building as initial path.
- **Logic:** Identical active filtering; throws if empty.

#### SLA) Get Leads (stage=NEW)
- **Type / role:** `googleSheets` — fetches all leads needing monitoring.
- **Filter:** `stage == "NEW"`
- **Output:** Multiple lead items.

#### SLA) If last route >= 1 hour
- **Type / role:** `if` — SLA breach detection.
- **Condition (boolean expression):**
  - Computes elapsed time since the most relevant timestamp:
    - `last_routed_at || initial_assigned_at || assigned_at || created_at`
  - Compares to `60 * 60 * 1000`
- **True path:** lead is eligible for reroute.
- **False path:** no action.
- **Edge cases:**
  - If timestamps are missing or invalid strings, `new Date(...).getTime()` may be `NaN`, making the comparison false.

#### SLA) Re-route to next sales
- **Type / role:** `code` — per-lead reassignment.
- **Mode:** “Run Once for Each Item” (critical because it returns a single object, not an array).
- **Logic:**
  - identifies current assignee by `sales_slack_id`
  - if not found, defaults to index 0
  - picks next index `(safeIndex + 1) % sales.length`
  - increments `route_count`
  - sets `assigned_sales`, `sales_slack_id`, updates `assigned_at` and `last_routed_at`
- **Edge cases:**
  - If lead has `sales_slack_id` not in sales list (rep removed), it jumps to start.

#### IF route_count >= 10
- **Type / role:** `if` — escalation threshold gate.
- **Condition:** `{{$json.route_count}} >= 10`
- **True path:** manager escalation path (Set Manager id → escalation flag logic).
- **False path:** continues directly to updating lead.

**Connection note (important):**
- The node is wired so that:
  - **True output** goes to **Set Manager id** (escalation path)
  - **False output** goes to **SLA) Update lead (after reroute)**
- Because of this, escalation path does **not** automatically continue to update the lead unless separately connected (it is not). In the current wiring, leads that trigger escalation might skip `SLA) Update lead (after reroute)` unless n8n merges paths elsewhere (it does not here). If you intend both escalation *and* lead update, connect True output also to `SLA) Update lead (after reroute)` or add a Merge.

#### SLA) Update lead (after reroute)
- **Type / role:** `googleSheets` — updates assigned fields for the rerouted lead.
- **Operation:** `update` matching on `lead_id`
- **Updates:** `assigned_at, route_count, assigned_sales, last_routed_at, sales_slack_id`
- **Edge cases:** If matching fails (lead_id not found), update does nothing/fails depending on Sheets node behavior.

#### SLA) Slack Notify (reroute) & Button
- **Type / role:** `slack` — posts reroute message + button.
- **Content:** “Lead Re-routed (SLA)” with lead details; mentions new assignee.
- **Uses expressions referencing a specific node’s items by index:**
  - `{{ $items("SLA) Re-route to next sales")[$itemIndex].json.lead_id }}`
- **Edge cases:**
  - If downstream node changes item order/count, `$itemIndex` alignment issues can appear.
  - Same Slack interactivity requirement as the initial message.

#### Set Manager id
- **Type / role:** `set` — injects manager Slack user ID.
- **Configuration:** raw JSON:
  - `manager_slack_id: "<REDACTED_MANAGER_SLACK_USER_ID>"`
- **Output:** Pass-through includes existing fields plus manager id.

#### ESC) Get escalation flag
- **Type / role:** `googleSheets` — checks anti-spam flag.
- **Filter:** `key == ("escalated_" + lead_id)`
- **alwaysOutputData:** enabled, so downstream nodes still run even if no row is found.
- **Edge cases:** If sheet has multiple matching rows, normalization logic handles presence/absence only.

#### ESC) Normalize flag result
- **Type / role:** `code` — converts lookup rows to `already_escalated` boolean.
- **Logic:**
  - reads current lead from `$json`
  - reads lookup rows from `$items("ESC) Get escalation flag")`
  - `found` if any row has a non-empty `key`
  - outputs `esc_key`, `already_escalated`, and `esc_row`
- **Edge cases:** If Sheets node returns empty items but `alwaysOutputData` emits something nonstandard, detection relies on checking `key`.

#### ESC) If not escalated yet
- **Type / role:** `if` — one-time escalation gate.
- **Condition:** `{{$json.already_escalated === false}}`
- **True path:** sends escalation Slack and sets flag.
- **False path:** do nothing.

#### 🚨 SLACK ESCALATION
- **Type / role:** `slack` — posts escalation to a manager channel.
- **Message content:** Includes lead details and mentions the last assigned rep; also includes manager mention.
- **Expressions reference nodes by name:**
  - `$('IF route_count >= 10').item.json...`
  - `$('Set Manager id').item.json.manager_slack_id`
- **Edge cases:** If this node is executed without those nodes in the same execution path/item context, expressions can fail. (Current wiring executes it through that path, so it should resolve.)

#### ESC) Set escalation flag
- **Type / role:** `googleSheets` — writes `escalated_<lead_id>` in routing_state_sheet to prevent repeat alerts.
- **Operation:** `appendOrUpdate` matching on `key`
- **Value:** current timestamp ISO string.
- **Expression note:** Uses:
  - `{{$items('SLA) Re-route to next sales')[$itemIndex].json.lead_id}}`
  - This again relies on correct `$itemIndex` alignment and the presence of the SLA reroute node in the run.

---

### Block C — “CONTACTED” Stage Update API (Slack Interaction)

**Overview:**  
Receives Slack interactive button payloads, extracts `lead_id`, validates that the clicker is the currently assigned owner and that the lead is still `NEW`, updates the lead stage to `CONTACTED`, otherwise sends an access denied message back via Slack `response_url`.

**Nodes involved:**
- STAGE) Webhook (/demo-stage-update)
- STAGE) Prep Payload
- STAGE) Lookup lead by lead_id
- STAGE) Authorize click (must be assigned sales)
- STAGE) If authorized
- STAGE) Update lead stage to CONTACTED
- STAGE) Slack Feedback - Not allowed

#### STAGE) Webhook (/demo-stage-update)
- **Type / role:** `webhook` — Slack interactive endpoint.
- **Configuration:**
  - `path: demo-stage-update`
  - `POST`
  - `responseMode: lastNode`
- **Slack requirement:** Configure Slack Interactivity Request URL to this webhook URL.
- **Edge cases:**
  - Slack sends `application/x-www-form-urlencoded` with `payload=...`; n8n must be able to capture `body.payload`.

#### STAGE) Prep Payload
- **Type / role:** `code` — parses Slack’s form-encoded payload JSON string.
- **Logic:**
  - reads `const payloadStr = $json?.body?.payload`
  - `JSON.parse(payloadStr)`
  - extracts:
    - `lead_id` from `slack.actions[0].value`
    - `actor_slack_id` from `slack.user.id`
    - `response_url`
  - sets `stage = "CONTACTED"`
  - adds timestamps: `contacted_at = now`
- **Failures:** Throws if payload missing or invalid JSON.
- **Edge cases:** If Slack payload structure changes (multiple actions, different fields), extraction may fail.

#### STAGE) Lookup lead by lead_id
- **Type / role:** `googleSheets` — fetches lead row for authorization + stage checks.
- **Filter:** `lead_id == {{$json.lead_id}}`
- **Output:** lead row fields including `sales_slack_id`, `stage`.

#### STAGE) Authorize click (must be assigned sales)
- **Type / role:** `code` — authorization and idempotency gate.
- **Checks:**
  - Lead exists
  - `assigned sales` (`leadRow.sales_slack_id`) equals `actor_slack_id`
  - Current stage is exactly `NEW` (prevents repeated/late updates)
- **Outputs:**
  - `authorized_owner`, `stage_allowed`, `authorized`, `reason`
  - includes `assigned_sales_slack_id`, `current_stage`, `lead_name`, `lead_phone`
- **Edge cases:** If stage in Sheets is e.g. `New` or has trailing spaces, it uppercases and trims, so it still works.

#### STAGE) If authorized
- **Type / role:** `if` — routes to update vs deny feedback.
- **Condition:** `{{$json.authorized}}` must be true.
- **True path:** update stage to CONTACTED.
- **False path:** send “not allowed” message to Slack via response_url.

#### STAGE) Update lead stage to CONTACTED
- **Type / role:** `googleSheets` — updates lead stage.
- **Operation:** `update` matching on `lead_id`
- **Writes:** `stage`, `contacted_at`
- **Note:** `contacted_by` is produced earlier but not written here; if needed, add a mapped column.

#### STAGE) Slack Feedback - Not allowed
- **Type / role:** `httpRequest` — posts ephemeral feedback to Slack `response_url`.
- **Method:** `POST` JSON body with `replace_original: false`
- **Message includes:** assigned owner, clicker, lead id.
- **Edge cases:** If `response_url` is missing/expired, request fails (Slack response_url can expire).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1) Form Trigger (New Lead) | formTrigger | Lead intake entry point | — | 2) Prep Lead | ## Intake & Initial Assignment (Round Robin) |
| 2) Prep Lead | code | Normalize lead, generate lead_id, init timestamps/state | 1) Form Trigger (New Lead) | 3) Get Sales List (active=ON) | ## Intake & Initial Assignment (Round Robin) |
| 3) Get Sales List (active=ON) | googleSheets | Read active sales roster | 2) Prep Lead | 4) Build Sales List | ## Intake & Initial Assignment (Round Robin) |
| 4) Build Sales List | code | Filter/map active sales into sales_list | 3) Get Sales List (active=ON) | 5) Get routing_state (last_index) | ## Intake & Initial Assignment (Round Robin) |
| 5) Get routing_state (last_index) | googleSheets | Fetch round-robin pointer | 4) Build Sales List | 6) Initial Assign (Round Robin) | ## Intake & Initial Assignment (Round Robin) |
| 6) Initial Assign (Round Robin) | code | Assign lead to next rep, set timestamps/counters | 5) Get routing_state (last_index) | 7) Update routing_state; 8) Upsert Lead (leads sheet) | ## Intake & Initial Assignment (Round Robin) |
| 7) Update routing_state | googleSheets | Persist updated last_index | 6) Initial Assign (Round Robin) | — | ## Intake & Initial Assignment (Round Robin) |
| 8) Upsert Lead (leads sheet) | googleSheets | Store lead record | 6) Initial Assign (Round Robin) | 9) Slack Notify (New Lead) & Button | ## Intake & Initial Assignment (Round Robin) |
| 9) Slack Notify (New Lead) & Button | slack | Notify channel + interactive button | 8) Upsert Lead (leads sheet) | — | ## Intake & Initial Assignment (Round Robin) |
| SLA Trigger (every 1 hour) | scheduleTrigger | Hourly SLA scan entry point | — | SLA) Get Sales List | ## SLA Monitor & Hourly Re-route |
| SLA) Get Sales List | googleSheets | Read sales roster for rerouting | SLA Trigger (every 1 hour) | SLA) Build Sales List | ## SLA Monitor & Hourly Re-route |
| SLA) Build Sales List | code | Build active sales_list for SLA path | SLA) Get Sales List | SLA) Get Leads (stage=NEW) | ## SLA Monitor & Hourly Re-route |
| SLA) Get Leads (stage=NEW) | googleSheets | Fetch NEW leads | SLA) Build Sales List | SLA) If last route >= 1 hour | ## SLA Monitor & Hourly Re-route |
| SLA) If last route >= 1 hour | if | Determine SLA breach | SLA) Get Leads (stage=NEW) | SLA) Re-route to next sales (true) | ## SLA Monitor & Hourly Re-route |
| SLA) Re-route to next sales | code | Reassign lead to next rep | SLA) If last route >= 1 hour (true) | IF route_count >= 10 | ## SLA Monitor & Hourly Re-route |
| IF route_count >= 10 | if | Escalation threshold gate | SLA) Re-route to next sales | Set Manager id (true); SLA) Update lead (after reroute) (false) | ## SLA Monitor & Hourly Re-route |
| SLA) Update lead (after reroute) | googleSheets | Persist reroute assignment fields | IF route_count >= 10 (false) | SLA) Slack Notify (reroute) & Button | ## SLA Monitor & Hourly Re-route |
| SLA) Slack Notify (reroute) & Button | slack | Notify reroute + interactive button | SLA) Update lead (after reroute) | — | ## SLA Monitor & Hourly Re-route |
| Set Manager id | set | Inject manager Slack ID | IF route_count >= 10 (true) | ESC) Get escalation flag | ## SLA Monitor & Hourly Re-route |
| ESC) Get escalation flag | googleSheets | Check one-time escalation flag | Set Manager id | ESC) Normalize flag result | ## SLA Monitor & Hourly Re-route |
| ESC) Normalize flag result | code | Compute already_escalated boolean | ESC) Get escalation flag | ESC) If not escalated yet | ## SLA Monitor & Hourly Re-route |
| ESC) If not escalated yet | if | Prevent escalation spam | ESC) Normalize flag result | 🚨 SLACK ESCALATION (true) | ## SLA Monitor & Hourly Re-route |
| 🚨 SLACK ESCALATION | slack | Send escalation message to manager channel | ESC) If not escalated yet (true) | ESC) Set escalation flag | ## SLA Monitor & Hourly Re-route |
| ESC) Set escalation flag | googleSheets | Write escalated_<lead_id> flag | 🚨 SLACK ESCALATION | — | ## SLA Monitor & Hourly Re-route |
| STAGE) Webhook (/demo-stage-update) | webhook | Slack interactive callback entry point | — | STAGE) Prep Payload | ##  "Contacted" Stage Update API (Stop Routing) |
| STAGE) Prep Payload | code | Parse Slack payload + extract lead_id/user/response_url | STAGE) Webhook (/demo-stage-update) | STAGE) Lookup lead by lead_id | ##  "Contacted" Stage Update API (Stop Routing) |
| STAGE) Lookup lead by lead_id | googleSheets | Fetch lead row for auth/stage checks | STAGE) Prep Payload | STAGE) Authorize click (must be assigned sales) | ##  "Contacted" Stage Update API (Stop Routing) |
| STAGE) Authorize click (must be assigned sales) | code | Owner + idempotency authorization | STAGE) Lookup lead by lead_id | STAGE) If authorized | ##  "Contacted" Stage Update API (Stop Routing) |
| STAGE) If authorized | if | Route to update vs deny | STAGE) Authorize click (must be assigned sales) | STAGE) Update lead stage to CONTACTED (true); STAGE) Slack Feedback - Not allowed (false) | ##  "Contacted" Stage Update API (Stop Routing) |
| STAGE) Update lead stage to CONTACTED | googleSheets | Update stage/contacted_at in leads sheet | STAGE) If authorized (true) | — | ##  "Contacted" Stage Update API (Stop Routing) |
| STAGE) Slack Feedback - Not allowed | httpRequest | Post denial message to Slack response_url | STAGE) If authorized (false) | — | ##  "Contacted" Stage Update API (Stop Routing) |
| Sticky Note | stickyNote | Documentation | — | — | ## SLA-Based Lead Routing & Auto Reassignment Workflow (full description text) |
| Sticky Note1 | stickyNote | Section header | — | — | ## Intake & Initial Assignment (Round Robin) |
| Sticky Note2 | stickyNote | Section header | — | — | ## SLA Monitor & Hourly Re-route |
| Sticky Note3 | stickyNote | Section header | — | — | ##  "Contacted" Stage Update API (Stop Routing) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Sheets structure**
   1. Create a spreadsheet with at least these sheets:
      - `sales_sheet`
      - `lead_sheet`
      - `routing_state_sheet`
   2. In `sales_sheet`, create columns: `name`, `email`, `slack_id`, `active`
      - Mark active users with `ON` (or `TRUE`).
   3. In `lead_sheet`, create columns (minimum required by this workflow):
      - `lead_id` (unique key)
      - `name`, `phone`, `stage`
      - `assigned_sales`, `sales_slack_id`
      - `initial_assigned_at`, `assigned_at`, `last_routed_at`, `created_at`
      - `route_count`
      - `contacted_at` (for stage update path)
   4. In `routing_state_sheet`, create columns: `key`, `value`
      - Add an initial row: `key = last_index`, `value = -1`
      - Escalation flags will be stored as `key = escalated_<lead_id>`.

2) **Create credentials in n8n**
   1. Add **Google Sheets OAuth2** credential with access to the spreadsheet.
   2. Add **Slack** credential (Slack API token) with permissions to:
      - Post messages to target channels
      - Use Block Kit messages
   3. In Slack App settings:
      - Enable **Interactivity & Shortcuts**
      - Set **Request URL** to the n8n webhook URL you will create in step 6 (STAGE webhook).

3) **Build Block A: Intake & Initial Assignment**
   1. Add node **Form Trigger**
      - Title: “Lead Intake (Generic Form Trigger)”
      - Fields:
        - Name (required)
        - Phone (required)
   2. Add node **Code** named `2) Prep Lead`
      - Paste logic to generate `lead_id`, normalize phone to `62...`, set `stage="NEW"`, initialize timestamps and routing fields.
   3. Add node **Google Sheets** named `3) Get Sales List (active=ON)`
      - Operation: read/get many rows
      - Document: your spreadsheet
      - Sheet: `sales_sheet`
   4. Add node **Code** named `4) Build Sales List`
      - Filter `active` ON/TRUE
      - Map to `sales_list` objects (must include `name` and `slack_id`)
      - Throw error if list is empty
   5. Add node **Google Sheets** named `5) Get routing_state (last_index)`
      - Filter: `key` equals `last_index`
      - Sheet: `routing_state_sheet`
   6. Add node **Code** named `6) Initial Assign (Round Robin)`
      - Read lead from `2) Prep Lead` using `$items('2) Prep Lead')[0].json`
      - Read sales_list from `4) Build Sales List`
      - Read last index from `5) Get routing_state (last_index')`
      - Compute nextIndex, set assignee fields, increment `route_count`, set timestamps
   7. Add node **Google Sheets** named `7) Update routing_state`
      - Operation: `update`
      - Match column: `key`
      - Values:
        - `key = {{$json.routing_key}}` (use `last_index`)
        - `value = {{$json.assigned_sales_index}}`
   8. Add node **Google Sheets** named `8) Upsert Lead (leads sheet)`
      - Operation: `appendOrUpdate`
      - Match column: `lead_id`
      - Map all lead fields into columns (as listed above)
   9. Add node **Slack** named `9) Slack Notify (New Lead) & Button`
      - Post to the sales channel
      - Message type: **Block**
      - Add a button with:
        - `action_id = mark_contacted`
        - `value = {{$json.lead_id}}`

   10. **Connect nodes** in order:
      - Form Trigger → Prep Lead → Get Sales List → Build Sales List → Get routing_state → Initial Assign
      - From Initial Assign: connect to **Update routing_state** AND **Upsert Lead**
      - Upsert Lead → Slack Notify (New Lead)

4) **Build Block B: SLA monitor & reroute**
   1. Add node **Schedule Trigger** named `SLA Trigger (every 1 hour)`
      - Run every 1 hour
   2. Add node **Google Sheets** `SLA) Get Sales List` → sheet `sales_sheet`
   3. Add node **Code** `SLA) Build Sales List` (same logic as initial builder)
   4. Add node **Google Sheets** `SLA) Get Leads (stage=NEW)`
      - Filter: `stage` equals `NEW`
      - Sheet: `lead_sheet`
   5. Add node **IF** `SLA) If last route >= 1 hour`
      - Condition: boolean expression comparing elapsed time vs 1 hour using `last_routed_at` fallback chain.
   6. Add node **Code** `SLA) Re-route to next sales`
      - Set node to **Run Once for Each Item**
      - Reassign to next rep based on current `sales_slack_id`
      - Increment `route_count`, update timestamps
   7. Add node **IF** `IF route_count >= 10`
      - Condition: number `route_count >= 10`
   8. Add node **Google Sheets** `SLA) Update lead (after reroute)`
      - Operation: `update` matching on `lead_id`
      - Update assignment fields
   9. Add node **Slack** `SLA) Slack Notify (reroute) & Button`
      - Block message; include “Mark as CONTACTED” button
      - If you reference `$items("SLA) Re-route to next sales")[$itemIndex]`, keep item index alignment.
   10. **Escalation sub-branch**
       - Add **Set** node `Set Manager id` that sets `manager_slack_id`
       - Add **Google Sheets** `ESC) Get escalation flag` (sheet `routing_state_sheet`, filter `key = escalated_<lead_id>`) and enable **Always Output Data**
       - Add **Code** `ESC) Normalize flag result` to compute `already_escalated`
       - Add **IF** `ESC) If not escalated yet` (true when `already_escalated === false`)
       - Add **Slack** `🚨 SLACK ESCALATION` to manager channel
       - Add **Google Sheets** `ESC) Set escalation flag` (appendOrUpdate key/value into routing_state_sheet)

   11. **Connect nodes**:
       - Schedule Trigger → SLA Get Sales List → SLA Build Sales List → SLA Get Leads → SLA If (true) → SLA Re-route → IF route_count
       - IF route_count (false) → SLA Update lead → SLA Slack Notify
       - IF route_count (true) → Set Manager id → ESC Get flag → ESC Normalize → ESC If not escalated (true) → Slack Escalation → ESC Set flag

   **Recommended fix if you want escalation AND lead update:** also connect the **true** output of `IF route_count >= 10` to `SLA) Update lead (after reroute)` (or add a Merge), so escalated leads still get their reroute persisted and notified.

5) **Build Block C: Stage update endpoint for Slack interactivity**
   1. Add **Webhook** node `STAGE) Webhook (/demo-stage-update)`
      - POST
      - Path: `demo-stage-update`
      - Response: last node
   2. Add **Code** `STAGE) Prep Payload`
      - Parse `$json.body.payload` string as JSON
      - Extract `lead_id`, `actor_slack_id`, `response_url`
      - Set `stage = "CONTACTED"` and `contacted_at = now`
   3. Add **Google Sheets** `STAGE) Lookup lead by lead_id`
      - Filter `lead_id = {{$json.lead_id}}` on `lead_sheet`
   4. Add **Code** `STAGE) Authorize click (must be assigned sales)`
      - Allow only if:
        - lead exists
        - assigned `sales_slack_id` equals `actor_slack_id`
        - current stage is `NEW`
      - Output `authorized` boolean and include `response_url`
   5. Add **IF** `STAGE) If authorized`
      - true → update
      - false → send denial feedback
   6. Add **Google Sheets** `STAGE) Update lead stage to CONTACTED`
      - Operation: update matching `lead_id`
      - Write `stage` and `contacted_at`
   7. Add **HTTP Request** `STAGE) Slack Feedback - Not allowed`
      - POST to `{{$json.response_url}}`
      - JSON body describing access denied (do not replace original)
   8. **Connect nodes:**
      - Webhook → Prep Payload → Lookup → Authorize → If
      - If true → Update stage
      - If false → HTTP Request feedback

6) **Configure Slack button callbacks**
   - In Slack App settings: set **Interactivity Request URL** to the full URL of `STAGE) Webhook (/demo-stage-update)`.
   - Ensure your Slack messages’ button `action_id` matches your expected action (here it is `mark_contacted`; the workflow does not branch on it, but it’s good practice).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “SLA-Based Lead Routing & Auto Reassignment Workflow … ensures fast response times … auto reroutes … idempotent stage updates … one-time escalation” | Sticky note description embedded in the workflow canvas (high-level system behavior and safety properties). |
| SLA threshold is implemented via: (a) hourly schedule + (b) “last route >= 1 hour” IF condition | Adjust by changing schedule interval and/or the millisecond comparison constant. |
| Google Sheets used as lightweight datastore; logic is adaptable to CRM/DB | Mentioned in workflow notes; replace Sheets nodes with database/CRM nodes while preserving field contracts. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.