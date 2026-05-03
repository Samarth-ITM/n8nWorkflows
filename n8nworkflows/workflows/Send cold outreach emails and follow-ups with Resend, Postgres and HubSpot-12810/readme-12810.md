Send cold outreach emails and follow-ups with Resend, Postgres and HubSpot

https://n8nworkflows.xyz/workflows/send-cold-outreach-emails-and-follow-ups-with-resend--postgres-and-hubspot-12810


# Send cold outreach emails and follow-ups with Resend, Postgres and HubSpot

## 1. Workflow Overview

**Title:** Send cold outreach emails and follow-ups with Resend, Postgres and HubSpot

**Purpose / use case:**  
This workflow automates (1) sending initial cold outreach emails from leads stored in Postgres, (2) sending a follow-up after a delay (based on “days since Email sent”), and (3) listening for inbound Resend webhook events to stop/mark sequences and push contacts into HubSpot.

### 1.1 Block A — Initial outreach: select unsent leads → send email → mark “Email sent”
Triggered on a schedule, pulls rows where **Email sent IS NULL**, iterates through them, delays per item, sends via Resend, then updates Postgres.

### 1.2 Block B — Follow-up: select not-followed leads → filter “3 days since Email sent” → send follow-up → mark “1st follow-up”
Triggered on a schedule, pulls rows where **1st follow-up IS NULL**, filters those whose **Email sent** is exactly 3 days ago, then iterates, waits, sends via Resend, updates Postgres.

### 1.3 Block C — Inbound webhook: receive Resend event → update Postgres → upsert HubSpot contact
A webhook endpoint receives inbound payloads (intended: reply/received event), updates the Postgres table, then creates/updates a HubSpot contact.

---

## 2. Block-by-Block Analysis

### Block A — Initial outreach pipeline

**Overview:**  
Fetches leads that have not been emailed yet, loops through them one-by-one (using Split in Batches + Wait), sends an email with Resend, then stamps the “Email sent” timestamp in Postgres.

**Nodes involved:**  
- Schedule Trigger  
- Select rows from a table  
- Loop Over Items  
- Wait  
- HTTP Request  
- Update rows in a table  

#### Node: **Schedule Trigger**
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — starts Block A automatically.
- **Config (interpreted):** Uses an interval rule with default/empty interval settings (as provided, it may need a real interval like every day/hour).
- **Connections:**  
  - Output → **Select rows from a table**
- **Edge cases / failures:** Misconfigured interval can cause it not to run or run too frequently.

#### Node: **Select rows from a table**
- **Type / role:** Postgres Select (`n8n-nodes-base.postgres`) — retrieves lead rows.
- **Config:**
  - Schema: `public`
  - Table: `n8n tutorial`
  - Where: `Email sent IS NULL`
  - Return all: enabled
  - `executeOnce: true` (important: in production this can prevent repeated execution depending on how you run tests/executions)
- **Connections:**  
  - Input ← Schedule Trigger  
  - Output → Loop Over Items
- **Edge cases / failures:** credential/connection errors, table/column naming issues (spaces in column names require correct quoting by the node).

#### Node: **Loop Over Items**
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates through selected rows.
- **Config:** Defaults (batch size not explicitly set in JSON; n8n defaults apply).
- **Connections:**  
  - Input ← Select rows from a table  
  - Output (loop continue) → Wait (on output index 1)
  - Output (done) → (no next node connected)
- **Edge cases / failures:** If batch size is defaulted unexpectedly, it may process more than one row per cycle; ensure batch size = 1 if strict sequencing is required.

#### Node: **Wait**
- **Type / role:** Wait (`n8n-nodes-base.wait`) — throttles sending (delay per lead).
- **Config:** No explicit wait duration in parameters (as-is, this is likely incomplete; in n8n Wait typically needs “Wait for” config or resume via webhook).
- **Connections:**  
  - Input ← Loop Over Items  
  - Output → HTTP Request
- **Edge cases / failures:** A Wait node without a configured waiting strategy can stall the workflow indefinitely.

#### Node: **HTTP Request**
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — sends email via Resend API.
- **Config:**
  - Method: `POST`
  - URL: `https://api.resend.com/emails`
  - Auth: Generic Credential Type → `httpHeaderAuth` (credential: “Resend APP”)
  - Body parameters:
    - `from`: `user@example.com` (must be a verified sender/domain in Resend)
    - `to`: `{{$json.Email}}` (reads Email from current Postgres row)
    - `subject`: `My Elastic skin`
    - `text`: `Hi {{ $json.properties.firstname.value }}, ...` (note: references `properties.firstname.value`)
- **Key expressions / variables:**
  - `={{ $json.Email }}`
  - `=Hi {{ $json.properties.firstname.value }}` **(likely incorrect for Postgres rows; see edge cases)**
- **Connections:**  
  - Input ← Wait  
  - Output → Update rows in a table
- **Edge cases / failures:**
  - Resend auth header misconfigured → 401/403
  - `from` not verified → 403/422
  - `to` missing/invalid → 422
  - **Data mismatch:** Postgres rows appear to use `"First name"` not `properties.firstname.value`, so the email text template may render blank or throw expression errors depending on n8n settings.

#### Node: **Update rows in a table**
- **Type / role:** Postgres Update (`n8n-nodes-base.postgres`) — stamps Email sent.
- **Config:**
  - Schema: `public`, Table: `n8n tutorial`
  - Matching column: `id`
  - Values set:
    - `id`: `={{ $('Wait').item.json.id }}`
    - `Email sent`: `={{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`
- **Key expressions / variables:**
  - `$('Wait').item.json.id` (pulls the current item id from the Wait node context)
  - `$now.toFormat(...)` (Luxon-based formatting)
- **Connections:**  
  - Input ← HTTP Request  
  - Output → Loop Over Items (to fetch next batch)
- **Edge cases / failures:**
  - If Wait node doesn’t pass through the original item, `$('Wait').item.json.id` can be undefined.
  - Timestamp format must match Postgres column type expectations (it’s a `dateTime` column; the string may still coerce correctly, but ISO format is safer).

---

### Block B — Follow-up pipeline (3 days after initial email)

**Overview:**  
Pulls leads with no first follow-up, filters to those whose “Email sent” was 3 days ago, then sends follow-up emails via Resend and stamps “1st follow-up”.

**Nodes involved:**  
- Schedule Trigger1  
- Select rows from a table1  
- Filter  
- Loop Over Items1  
- Wait1  
- HTTP Request1  
- Update rows in a table1  

#### Node: **Schedule Trigger1**
- **Type / role:** Schedule Trigger — starts follow-up pipeline.
- **Config:** Interval rule with empty/default interval settings (needs real schedule).
- **Connections:**  
  - Output → Select rows from a table1
- **Edge cases:** same as Schedule Trigger.

#### Node: **Select rows from a table1**
- **Type / role:** Postgres Select — retrieves leads pending follow-up.
- **Config:**
  - Where: `1st follow-up IS NULL`
  - Return all: enabled
  - Table: `n8n tutorial` (public)
  - `executeOnce: true`
- **Connections:**  
  - Input ← Schedule Trigger1  
  - Output → Filter
- **Edge cases:** same as the first select.

#### Node: **Filter**
- **Type / role:** Filter (`n8n-nodes-base.filter`) — keep only rows at the desired age.
- **Config (logic):**
  - Condition: `{{$now.diff(DateTime.fromISO($json["Email sent"]), 'days').days}} == 3`
  - Strict type validation enabled.
- **Key expressions / variables:**
  - `DateTime.fromISO(...)` uses Luxon `DateTime`.
- **Connections:**  
  - Input ← Select rows from a table1  
  - Output: **not connected** in the provided JSON
- **Important issue:** In the current workflow JSON, **Filter has no outgoing connection**, so no follow-up will be sent. It should connect to **Loop Over Items1**.
- **Edge cases / failures:**
  - If `Email sent` is null or not ISO, `DateTime.fromISO` yields invalid → expression may fail or produce `NaN`.
  - Equality to exactly `3` days is brittle; small timing differences may miss candidates (often better: `>= 3` and `< 4`).

#### Node: **Loop Over Items1**
- **Type / role:** Split In Batches — iterates over filtered follow-up candidates.
- **Connections (as built):**
  - Input: **not connected** (should come from Filter)
  - Output (loop continue index 1) → Wait1
- **Edge cases:** same as Loop Over Items.

#### Node: **Wait1**
- **Type / role:** Wait — throttling/delay.
- **Config:** Not configured (same risk as Wait).
- **Connections:**  
  - Input ← Loop Over Items1  
  - Output → HTTP Request1

#### Node: **HTTP Request1**
- **Type / role:** HTTP Request — sends follow-up via Resend.
- **Config:** Same Resend endpoint/auth/body structure as HTTP Request (Block A).
- **Connections:**  
  - Input ← Wait1  
  - Output → Update rows in a table1
- **Edge cases:** same as HTTP Request (including likely wrong `properties.firstname.value` path).

#### Node: **Update rows in a table1**
- **Type / role:** Postgres Update — stamps first follow-up.
- **Config:**
  - Match: `id`
  - Values:
    - `id`: `={{ $('Wait1').item.json.id }}`
    - `1st follow-up`: `={{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`
    - `Email sent`: `=` **(this appears to be an invalid/placeholder value)**
- **Connections:**  
  - Input ← HTTP Request1  
  - Output → Loop Over Items1
- **Edge cases / failures:**
  - The `Email sent` field set to `"="` can cause update errors/type conversion failures. It should likely be omitted or left unchanged.
  - Same item-context risk as Block A.

---

### Block C — Resend inbound webhook → update Postgres → HubSpot upsert

**Overview:**  
Exposes a webhook endpoint intended to be called by Resend when an email is received/replied, updates the lead record in Postgres, then creates/updates the contact in HubSpot CRM.

**Nodes involved:**  
- Webhook  
- Update rows in a table2  
- Create or update a contact  

#### Node: **Webhook**
- **Type / role:** Webhook (`n8n-nodes-base.webhook`) — inbound HTTP endpoint.
- **Config:**
  - Method: `POST`
  - Path: `resend-email`
  - Full URL depends on n8n base URL (e.g., `https://<n8n-host>/webhook/resend-email`)
- **Connections:**  
  - Output: **not connected** in the provided JSON (so Block C currently won’t execute)
- **Edge cases / failures:**
  - Not validating signature/authenticity of Resend webhook (recommended to verify).
  - Payload schema changes can break expressions downstream.

#### Node: **Update rows in a table2**
- **Type / role:** Postgres Update — updates lead status based on webhook payload.
- **Config:**
  - Match: `Email`
  - Values:
    - `Email`: `={{ $json.body.data.from }}`
    - `Email sent`: hard-coded `2026-01-04T00:00:00`
    - `1st follow-up`: hard-coded `2026-01-04T00:00:00`
    - `2nd follow-up`: hard-coded `2026-01-04T00:00:00`
- **Connections:**  
  - Output → Create or update a contact
- **Edge cases / failures:**
  - Hard-coded timestamps look like placeholders; will overwrite real history.
  - `$json.body.data.from` assumes Resend posts `body.data.from`; actual Resend webhook payloads may differ (often `from` can be an object or include name/email formatting).
  - Since Webhook isn’t connected, this node is unreachable unless manually executed.

#### Node: **Create or update a contact**
- **Type / role:** HubSpot (`n8n-nodes-base.hubspot`) — upserts a CRM contact by email.
- **Config:**
  - Authentication: App Token (private app token)
  - Email: `={{ $json.Email }}`
  - Additional fields mapped from Postgres-like row fields:
    - firstName: `={{ $json["First name"] }}`
    - lastName: `={{ $json["Last name"] }}`
    - industry: `={{ $json.Industry }}`
    - websiteUrl: `={{ $json.Website }}`
- **Connections:**  
  - Input ← Update rows in a table2
- **Edge cases / failures:**
  - If Update rows in a table2 outputs only update metadata (not the full updated row), `$json["First name"]` etc. may be missing. You may need to re-select the row after update or pass through original data.
  - HubSpot property names must exist and be writable; `websiteUrl` vs `website` property mismatch can cause errors depending on HubSpot schema.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts initial outreach on a schedule | — | Select rows from a table | ## 1. Get data, send emails and update |
| Select rows from a table | Postgres | Fetch leads where “Email sent” is NULL | Schedule Trigger | Loop Over Items | ## 1. Get data, send emails and update |
| Loop Over Items | Split In Batches | Iterate through leads | Select rows from a table; Update rows in a table | Wait (loop output) | ## 1. Get data, send emails and update |
| Wait | Wait | Delay/throttle sending per lead | Loop Over Items | HTTP Request | ## 1. Get data, send emails and update |
| HTTP Request | HTTP Request | Send initial email via Resend | Wait | Update rows in a table | ## 1. Get data, send emails and update |
| Update rows in a table | Postgres | Mark “Email sent” timestamp | HTTP Request | Loop Over Items | ## 1. Get data, send emails and update |
| Schedule Trigger1 | Schedule Trigger | Starts follow-up sequence on a schedule | — | Select rows from a table1 | ## 2. Filter by last day contacted, send follow-up, then update |
| Select rows from a table1 | Postgres | Fetch leads where “1st follow-up” is NULL | Schedule Trigger1 | Filter | ## 2. Filter by last day contacted, send follow-up, then update |
| Filter | Filter | Keep only leads with Email sent exactly 3 days ago | Select rows from a table1 | (none connected) | ## 2. Filter by last day contacted, send follow-up, then update |
| Loop Over Items1 | Split In Batches | Iterate follow-up candidates | (none connected) ; Update rows in a table1 | Wait1 (loop output) | ## 2. Filter by last day contacted, send follow-up, then update |
| Wait1 | Wait | Delay/throttle follow-up sends | Loop Over Items1 | HTTP Request1 | ## 2. Filter by last day contacted, send follow-up, then update |
| HTTP Request1 | HTTP Request | Send follow-up email via Resend | Wait1 | Update rows in a table1 | ## 2. Filter by last day contacted, send follow-up, then update |
| Update rows in a table1 | Postgres | Mark “1st follow-up” timestamp | HTTP Request1 | Loop Over Items1 | ## 2. Filter by last day contacted, send follow-up, then update |
| Webhook | Webhook | Receives Resend inbound event | — | (none connected) | ## 3. Listen to email and stop when user reply. Then append to Hubspot |
| Update rows in a table2 | Postgres | Update lead record on inbound event | (unreachable; intended from Webhook) | Create or update a contact | ## 3. Listen to email and stop when user reply. Then append to Hubspot |
| Create or update a contact | HubSpot | Upsert contact into HubSpot | Update rows in a table2 | — | ## 3. Listen to email and stop when user reply. Then append to Hubspot |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Postgres credentials**
   - In n8n: Credentials → **Postgres**
   - Point to your Postgres/Supabase instance (transaction pooler if using Supabase).
   - Ensure table exists: `public."n8n tutorial"` with columns like: `id`, `First name`, `Last name`, `Email`, `Industry`, `Website`, `Email sent`, `1st follow-up`, `2nd follow-up`.

2) **Create Resend credentials**
   - Create an **HTTP Header Auth** credential (or equivalent).
   - Header should be `Authorization: Bearer <RESEND_API_KEY>` (typical Resend pattern).
   - Verify your sender domain in Resend and use a valid `from`.

3) **Create HubSpot credentials**
   - Use **HubSpot App Token** (private app token).
   - Ensure your app has permissions to write contacts.

---

### Block A (Initial send)

4) Add **Schedule Trigger** node
   - Configure an interval (e.g., every day/hour). The provided JSON interval is empty and should be set explicitly.

5) Add **Postgres** node named “Select rows from a table”
   - Operation: **Select**
   - Schema: `public`
   - Table: `n8n tutorial`
   - Where: `Email sent` → `IS NULL`
   - Return All: true

6) Add **Split In Batches** named “Loop Over Items”
   - Set **Batch Size = 1** (recommended for sequential sending).

7) Add **Wait** node
   - Configure a real wait strategy:
     - e.g., “Wait for” → 10 seconds/minutes (rate limiting), or
     - remove it if you don’t need throttling.

8) Add **HTTP Request** node
   - Method: `POST`
   - URL: `https://api.resend.com/emails`
   - Authentication: **HTTP Header Auth** credential (Resend)
   - Body parameters:
     - `from`: your verified sender (e.g., `sales@yourdomain.com`)
     - `to`: `={{ $json.Email }}`
     - `subject`: your subject
     - `text`: use Postgres fields, e.g. `=Hi {{ $json["First name"] }}, ...` (avoid `properties.firstname.value` unless your data actually has that structure)

9) Add **Postgres** node named “Update rows in a table”
   - Operation: **Update**
   - Match column: `id`
   - Set:
     - `id`: `={{ $('Wait').item.json.id }}`
     - `Email sent`: `={{ $now.toISO() }}` (recommended)
   - Connect: HTTP Request → Update → Loop Over Items (to continue batches)

10) Wire Block A
   - Schedule Trigger → Select → Loop Over Items
   - Loop Over Items (loop output) → Wait → HTTP Request → Update → Loop Over Items

---

### Block B (Follow-up)

11) Add **Schedule Trigger** node named “Schedule Trigger1”
   - Configure schedule (e.g., daily).

12) Add **Postgres Select** named “Select rows from a table1”
   - Where: `1st follow-up IS NULL`

13) Add **Filter** node
   - Condition (recommended robust version):
     - Keep rows where Email sent is between 3 and 4 days ago, or `>= 3`
   - If you keep the original logic:
     - `={{ $now.diff(DateTime.fromISO($json["Email sent"]), 'days').days }} equals 3`

14) Add **Split In Batches** named “Loop Over Items1” (batch size 1)

15) Add **Wait1** (configured with actual wait time)

16) Add **HTTP Request1** (same as HTTP Request but with follow-up content)

17) Add **Postgres Update** named “Update rows in a table1”
   - Match: `id`
   - Set:
     - `id`: `={{ $('Wait1').item.json.id }}`
     - `1st follow-up`: `={{ $now.toISO() }}`
   - **Do not** set `Email sent` to `"="`—leave it unchanged.

18) Wire Block B (fixing the missing connection)
   - Schedule Trigger1 → Select rows from a table1 → Filter → Loop Over Items1
   - Loop Over Items1 (loop output) → Wait1 → HTTP Request1 → Update rows in a table1 → Loop Over Items1

---

### Block C (Webhook → Postgres → HubSpot)

19) Add **Webhook** node
   - Method: POST
   - Path: `resend-email`
   - Activate workflow to get production URL.
   - Configure Resend Webhook in Resend dashboard to POST to this URL.

20) Add **Postgres Update** node named “Update rows in a table2”
   - Match: choose a stable identifier. If matching by Email, parse the correct field from webhook payload.
   - Replace placeholder hard-coded timestamps with logic that represents “stop follow-ups” (commonly: set a “Replied” flag or set follow-up fields to a terminal state, rather than overwriting with fixed dates).

21) Add **HubSpot** node “Create or update a contact”
   - Operation: Create/Update contact
   - Email: from the record you want to upsert (ensure it exists in the incoming item)
   - Map additional fields as needed.

22) Wire Block C (fix the missing connection)
   - Webhook → Update rows in a table2 → Create or update a contact

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Stop Manual Follow-ups with Automated Email Outreach Pipeline using n8n, Postgres and Resend | Workflow sticky note (project description) |
| Watch my tutorial: @[youtube](ZmN8jMhNJS4) | YouTube video reference from sticky note |
| Resend service | https://resend.com/ |
| Supabase setup guidance mentioned | https://supabase.com/ |
| HubSpot CRM | https://www.hubspot.com/ |
| Setup steps included: Supabase table + transaction pooler, Resend domain verification + webhook, HubSpot legacy/private app token | From sticky note content |

**Notable implementation gaps to address (as provided):**
- Filter node has no outgoing connection (Block B won’t run).
- Webhook node has no outgoing connection (Block C won’t run).
- Wait/Wait1 nodes have no explicit waiting configuration (can stall).
- Email template references `properties.firstname.value` which likely doesn’t exist for Postgres rows.
- Follow-up update sets `Email sent` to `"="` (likely invalid).
- Webhook update uses hard-coded dates (placeholder) and may parse the wrong email field.