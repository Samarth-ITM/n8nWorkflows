Send application follow-up reminders from Airtable via email and SMS

https://n8nworkflows.xyz/workflows/send-application-follow-up-reminders-from-airtable-via-email-and-sms-13058


# Send application follow-up reminders from Airtable via email and SMS

## 1. Workflow Overview

**Workflow title:** Send application follow-up reminders from Airtable via email and SMS  
**Internal workflow name (n8n):** `wf3` (inactive)

**Purpose / use case**  
This workflow runs daily and sends follow-up reminders (email and/or SMS) to candidates stored in Airtable who were previously sent an application link (“Outreach Sent”) but have not yet completed the application. It uses elapsed time since outreach plus Airtable “sent” flags to ensure each reminder is sent only once.

### 1.1 Scheduling + Candidate Retrieval
Runs at 09:00 daily and searches Airtable for “eligible leads” (status/flags driven by Airtable).

### 1.2 Data Normalization + Routing Decision
Normalizes Airtable fields (name/email/phone), computes hours since outreach, and assigns a `routeCode`:
- `0`: no reminder
- `1`: Reminder 1 (>= 48 hours since outreach, not yet sent)
- `2`: Reminder 2 (>= 96 hours since outreach, not yet sent; also marks “Urgent Call Required”)

### 1.3 Reminder Dispatch (Email/SMS) + Airtable State Updates
For each route, sends email if an email exists and sends SMS if a phone exists, then updates Airtable flags immediately to prevent duplicates.

---

## 2. Block-by-Block Analysis

### Block A — Daily trigger and Airtable search
**Overview:** Triggers daily at 9AM, retrieves Airtable records that match eligibility criteria (outreach already sent).  
**Nodes involved:** `Cron — Daily 9AM`, `Airtable — List Eligible Leads1`

#### Node: Cron — Daily 9AM
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) – starts workflow on a cron schedule.
- **Configuration:** Cron expression `0 9 * * *` (every day at 09:00).
- **Connections:** Outputs to **Airtable — List Eligible Leads1**.
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - Timezone behavior depends on n8n instance timezone settings.
  - If n8n is down at 09:00, execution may be missed (no catch-up by default).

#### Node: Airtable — List Eligible Leads1
- **Type / role:** Airtable node (`n8n-nodes-base.airtable`) – searches table for candidates.
- **Operation:** `search`
- **Base/Table:** Base ID `appmHaHEnNwsKqWmK`, Table ID `tblFQ18WoHvTNiKCN`
- **Filter formula:**
  - `AND({Status}='Leads',{Outreach Sent}=1,NOT({Outreach Sent At}=''))`
  - Meaning: only records still in “Leads”, outreach already sent, and timestamp exists.
- **Credentials:** Airtable Personal Access Token (`airtableTokenApi`).
- **Connections:** Outputs to **Prepare Candidate Data**.
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - Airtable auth/permission errors (token scope must include read access to base/table).
  - `filterByFormula` depends on exact field names and types (checkbox vs boolean, etc.).
  - Pagination/large datasets: Airtable node handles paging, but execution time can increase.

---

### Block B — Candidate data preparation and reminder routing
**Overview:** Converts Airtable fields into consistent keys, computes time since outreach, and selects the reminder route.  
**Nodes involved:** `Prepare Candidate Data`, `router`

#### Node: Prepare Candidate Data
- **Type / role:** Code node (`n8n-nodes-base.code`) – transforms records and computes routing metadata.
- **Key logic:**
  - Extracts:
    - `recordId` from Airtable `id`
    - `firstName` from `First Name` or `First  Name` (handles spacing variation)
    - `email` from `Email`
    - `phoneE164` from `Phone Number` (prepends `+` if missing)
    - `sentAt` from `Outreach Sent At`
  - Computes:
    - `hoursSinceOutreach = floor((now - sentAt) in hours)`
    - `rem1Sent` true if `Reminder 1 Sent` is `true` or `"checked"`
    - `rem2Sent` true if `Reminder 2 Sent` is `true` or `"checked"`
  - Determines `routeCode`:
    - If `hoursSinceOutreach >= 96` and reminder 2 not sent → `routeCode = 2`
    - Else if `hoursSinceOutreach >= 48` and reminder 1 not sent → `routeCode = 1`
    - Else `routeCode = 0`
  - Adds `applyLink: "https://www.morestarts.co.uk/welding-course/"`
- **Connections:** Outputs to **router**.
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If `Outreach Sent At` is blank/invalid, `hoursSinceOutreach` remains `null` and `routeCode` stays `0`.
  - If Airtable field names change, extraction may break (especially `First  Name` vs `First Name`).
  - Phone normalization is minimal (only adds `+`); non-numeric formatting may cause SMS API rejection.

#### Node: router
- **Type / role:** IF node (`n8n-nodes-base.if`) used as a “router” (two outputs).
- **Intended behavior:** Route candidates into Reminder 2 path vs Reminder 1 path based on `routeCode`.
- **Current configuration (important):**
  - The IF node’s conditions list contains **two conditions combined with `AND`**:
    - `$json.routeCode === 2` is true
    - `=={{ $json.routeCode === 1 }}` (note the **extra `=`** and unusual expression form)
  - With `AND`, **no item can satisfy both `routeCode === 2` and `routeCode === 1`** simultaneously.
  - Result: the “true” branch is effectively unreachable; most/all items will go to the “false” branch.
- **Connections (as wired):**
  - Output 0 (true) → `R2 — IF Has Phone`, `R2 — IF Has Email`
  - Output 1 (false) → `R1 — IF Has Email`, `R1 — IF Has Phone`
- **Version notes:** typeVersion `2.3`.
- **Edge cases / failures:**
  - Because of the `AND` logic and malformed second expression, routing may be incorrect (Reminder 2 likely never triggers).
  - Expression parsing may fail due to `=={{ ... }}` depending on n8n validation/runtime behavior.

**Recommended fix (conceptual):**
- Use a Switch node (recommended) or two separate IF checks.
- If keeping IF:
  - Condition should be a single boolean: `{{ $json.routeCode === 2 }}`
  - Then true branch = Reminder 2, false branch can go to a second IF for Reminder 1 (`routeCode === 1`) or simply ignore.

---

### Block C — Reminder 1 delivery + Airtable update
**Overview:** If candidate is due for Reminder 1, send email and/or SMS (depending on available contact fields) then mark “Reminder 1 Sent” in Airtable.  
**Nodes involved:** `R1 — IF Has Email`, `Send a transactional email3`, `R1 — IF Has Phone`, `Brevo Sms1`, `R1 — Airtable Mark Reminder 1 Sent`

#### Node: R1 — IF Has Email
- **Type / role:** IF node – checks whether `email` exists.
- **Condition:** string `exists` on `{{ $json.email }}`
- **Connections:** True → **Send a transactional email3**
- **Edge cases:**
  - If email is whitespace, it is trimmed in Code node; good.
  - “False” path is not connected, so email-less items simply stop on that branch.

#### Node: Send a transactional email3
- **Type / role:** Brevo/Sendinblue transactional email (`n8n-nodes-base.sendInBlue`).
- **To:** `{{ $json.email }}`
- **Subject:** “Still time to apply (few spaces left)”
- **Body:** Plain text with `{{ $json.firstName }}` and a link to `https://www.morestarts.co.uk/welding-course`
- **Credentials:** `sendInBlueApi` (Brevo token)
- **Connections:** Output → **R1 — Airtable Mark Reminder 1 Sent**
- **Edge cases / failures:**
  - Invalid email → provider rejection.
  - Provider quota/rate limit.
  - Sender `user@example.com` must be an approved sender in Brevo.

#### Node: R1 — IF Has Phone
- **Type / role:** IF node – checks whether `phoneE164` exists.
- **Condition:** string `exists` on `{{ $json.phoneE164 }}`
- **Connections:** True → **Brevo Sms1**
- **Edge cases:**
  - “Exists” only checks non-empty; invalid formats will pass and may fail downstream.

#### Node: Brevo Sms1
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) sending Brevo Transactional SMS.
- **Endpoint:** `POST https://api.brevo.com/v3/transactionalSMS/send`
- **Headers:** `accept: application/json`, `content-type: application/json`, `api-key: ...` (hardcoded in node)
- **Body:** Includes `recipient` from `{{ $json.phoneE164 || $json.fields['Phone Number'] }}` and message content.
- **Connections:** Output → **R1 — Airtable Mark Reminder 1 Sent**
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - Hardcoded API key is a security risk; should be moved to credentials/environment variables.
  - If phone is not valid E.164, Brevo may reject.
  - If Brevo returns non-2xx, node fails unless “Continue On Fail” is enabled (not shown).

#### Node: R1 — Airtable Mark Reminder 1 Sent
- **Type / role:** Airtable update – sets the sent flag for reminder 1.
- **Operation:** `update`
- **Record matching:** Uses `id` = `{{ $('Airtable — List Eligible Leads1').item.json.id }}`
  - This explicitly references the Airtable search node’s item.
- **Fields updated:** `Reminder 1 Sent = true`
- **Connections:** Output → **WF3 Complete1**
- **Edge cases / failures:**
  - Cross-node item reference can break if item ordering diverges; safer is using `{{ $json.recordId }}` produced by Code node.
  - Airtable field must be a checkbox/boolean compatible with `true`.

---

### Block D — Reminder 2 delivery + Airtable update (plus urgent call flag)
**Overview:** If candidate is due for Reminder 2, send email and/or SMS, then mark Reminder 2 as sent and also flag “Urgent Call Required”.  
**Nodes involved:** `R2 — IF Has Email`, `Send a transactional email2`, `R2 — IF Has Phone`, `Brevo Sms`, `R2 — Airtable Mark Reminder 2 + Urgent Call`

#### Node: R2 — IF Has Email
- **Type / role:** IF node – checks whether `email` exists.
- **Condition:** string `exists` on `{{ $json.email }}`
- **Connections:** True → **Send a transactional email2**

#### Node: Send a transactional email2
- **Type / role:** Brevo/Sendinblue transactional email.
- **To:** `{{ $json.email }}`
- **Subject/body:** Same theme as reminder email (“Still time to apply…”).
- **Connections:** Output → **R2 — Airtable Mark Reminder 2 + Urgent Call**
- **Edge cases:** Same as other email node (sender verification, quota, invalid email).

#### Node: R2 — IF Has Phone
- **Type / role:** IF node – checks whether `phoneE164` exists.
- **Connections:** True → **Brevo Sms**

#### Node: Brevo Sms
- **Type / role:** HTTP Request to Brevo transactional SMS endpoint.
- **Endpoint/headers:** Same as Brevo Sms1 (hardcoded API key).
- **Body:** Reminder 2 SMS content (mentions “few spaces left” and the application link).
- **Connections:** Output → **R2 — Airtable Mark Reminder 2 + Urgent Call**
- **Edge cases:** Same as Brevo Sms1.

#### Node: R2 — Airtable Mark Reminder 2 + Urgent Call
- **Type / role:** Airtable update – marks Reminder 2 sent and flags urgent call.
- **Operation:** `update`
- **Record matching:** `id` = `{{ $('Airtable — List Eligible Leads1').item.json.id }}`
- **Fields updated:**
  - `Reminder 2 Sent = true`
  - `Urgent Call Required = true`
- **Connections:** Output → **WF3 Complete1**
- **Edge cases:** Same concerns about cross-node item reference; plus field names must exist and be writable.

---

### Block E — Completion marker
**Overview:** Terminates the workflow with a constant output (“WF3 finished”) for easier debugging/logging.  
**Nodes involved:** `WF3 Complete1`

#### Node: WF3 Complete1
- **Type / role:** Set node (`n8n-nodes-base.set`) – produces final message.
- **Configuration:** Keep only set values; sets `result = "WF3 finished"`.
- **Connections:** None.
- **Edge cases:** None significant.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Cron — Daily 9AM | scheduleTrigger | Daily scheduler (09:00) | — | Airtable — List Eligible Leads1 | ## Cron node + Airtable search/list node; Runs daily and retrieves candidates who have received outreach but have not yet applied.; Airtable controls eligibility via timestamps and flags. |
| Airtable — List Eligible Leads1 | airtable (search) | Retrieve eligible candidate records from Airtable | Cron — Daily 9AM | Prepare Candidate Data | ## Cron node + Airtable search/list node; Runs daily and retrieves candidates who have received outreach but have not yet applied.; Airtable controls eligibility via timestamps and flags. |
| Prepare Candidate Data | code | Normalize fields, compute hours since outreach, set routeCode | Airtable — List Eligible Leads1 | router | ## Code node + Switch node; Calculates elapsed time since outreach and outputs a single route key.; Ensures only one reminder path can run per record.. |
| router | if | Route items to Reminder 2 vs Reminder 1 path (currently misconfigured) | Prepare Candidate Data | R2 — IF Has Phone; R2 — IF Has Email; R1 — IF Has Email; R1 — IF Has Phone | ## Code node + Switch node; Calculates elapsed time since outreach and outputs a single route key.; Ensures only one reminder path can run per record.. |
| R1 — IF Has Email | if | Gate: send Reminder 1 email only if email exists | router | Send a transactional email3 | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| Send a transactional email3 | sendInBlue | Send Reminder 1 email via Brevo | R1 — IF Has Email | R1 — Airtable Mark Reminder 1 Sent | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| R1 — IF Has Phone | if | Gate: send Reminder 1 SMS only if phone exists | router | Brevo Sms1 | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| Brevo Sms1 | httpRequest | Send Reminder 1 SMS via Brevo Transactional SMS API | R1 — IF Has Phone | R1 — Airtable Mark Reminder 1 Sent | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| R1 — Airtable Mark Reminder 1 Sent | airtable (update) | Mark “Reminder 1 Sent” in Airtable | Send a transactional email3; Brevo Sms1 | WF3 Complete1 | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| R2 — IF Has Email | if | Gate: send Reminder 2 email only if email exists | router | Send a transactional email2 | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| Send a transactional email2 | sendInBlue | Send Reminder 2 email via Brevo | R2 — IF Has Email | R2 — Airtable Mark Reminder 2 + Urgent Call | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| R2 — IF Has Phone | if | Gate: send Reminder 2 SMS only if phone exists | router | Brevo Sms | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| Brevo Sms | httpRequest | Send Reminder 2 SMS via Brevo Transactional SMS API | R2 — IF Has Phone | R2 — Airtable Mark Reminder 2 + Urgent Call | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| R2 — Airtable Mark Reminder 2 + Urgent Call | airtable (update) | Mark “Reminder 2 Sent” and “Urgent Call Required” | Send a transactional email2; Brevo Sms | WF3 Complete1 | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| WF3 Complete1 | set | Output completion marker | R1 — Airtable Mark Reminder 1 Sent; R2 — Airtable Mark Reminder 2 + Urgent Call | — | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |
| Sticky Note | stickyNote | Documentation block | — | — | ## Automated Application Reminder Sequence (Email + SMS); How it works; This workflow automatically sends follow-up reminders to candidates who have received an application link but have not yet applied. It runs on a daily schedule and scans Airtable for eligible candidates based on how much time has passed since outreach was sent.; The workflow uses Airtable as the source of truth for timing and state. It calculates how many hours have elapsed since outreach and decides whether to send a first reminder, second reminder, or no message at all. Each reminder is sent only once using checkbox “sent” flags to ensure idempotency.; All routing logic is centralized in a Code node that outputs a simple route key. This avoids fragile IF conditions and prevents duplicate messages if the workflow retries or runs again the next day.; Setup steps; Connect Airtable and select the table containing candidate records.; Ensure Airtable includes a timestamp field for when outreach was sent.; Ensure checkbox fields exist for each reminder (e.g. “Reminder 1 Sent”, “Reminder 2 Sent”).; Connect your email provider (Brevo) and SMS provider.; Set the Cron node to run once per day at your preferred time.; Initial setup typically takes 10–15 minutes.; Customization; You can adjust reminder timing, message content, or disable SMS/email independently without changing the core logic. |
| Sticky Note1 | stickyNote | Documentation for schedule/search | — | — | ## Cron node + Airtable search/list node; Runs daily and retrieves candidates who have received outreach but have not yet applied.; Airtable controls eligibility via timestamps and flags. |
| Sticky Note2 | stickyNote | Documentation for routing | — | — | ## Code node + Switch node; Calculates elapsed time since outreach and outputs a single route key.; Ensures only one reminder path can run per record.. |
| Sticky Note3 | stickyNote | Documentation for sending/updating | — | — | ## Email/SMS nodes + Airtable update nodes; Sends the appropriate reminder and immediately updates Airtable “sent” flags.; Prevents duplicate reminders on future runs. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name it: `Send application follow-up reminders from Airtable via email and SMS` (or keep internal `wf3`).

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Mode: **Cron**
   - Expression: `0 9 * * *` (daily 09:00)
   - Connect to next node.

3) **Add Airtable Search (eligible leads)**
   - Node: **Airtable**
   - Credentials: create **Airtable Personal Access Token** credential with access to the base.
   - Operation: **Search**
   - Base: select by ID `appmHaHEnNwsKqWmK`
   - Table: select by ID `tblFQ18WoHvTNiKCN`
   - Filter by formula:
     - `AND({Status}='Leads',{Outreach Sent}=1,NOT({Outreach Sent At}=''))`
   - Connect to Code node.

4) **Add Code node (prepare/route)**
   - Node: **Code**
   - Paste logic that:
     - Reads Airtable fields: `First Name` / `First  Name`, `Email`, `Phone Number`, `Outreach Sent At`, `Reminder 1 Sent`, `Reminder 2 Sent`
     - Outputs normalized keys: `recordId`, `firstName`, `email`, `phoneE164`, `hoursSinceOutreach`, `routeCode`, `applyLink`
     - Uses thresholds: 48h for reminder 1, 96h for reminder 2 (priority to reminder 2)
   - Connect to a routing node.

5) **Add Routing (recommended: Switch node)**
   - Node: **Switch**
   - Value to evaluate: `{{ $json.routeCode }}`
   - Add rules:
     - `equals 2` → Reminder 2 path
     - `equals 1` → Reminder 1 path
     - (optional) default → end/no-op
   - (If you keep an IF node instead, ensure it does not require both route codes simultaneously.)

6) **Reminder 1 path: Email gate + send**
   - Node: **IF** (`R1 — IF Has Email`)
     - Condition: String → “exists” → `{{ $json.email }}`
   - Node: **Brevo / Sendinblue** (`Send a transactional email`)
     - Credentials: create **Brevo (Sendinblue) API** credential
     - Sender: an approved sender (replace `user@example.com`)
     - To: `{{ $json.email }}`
     - Subject/body: your Reminder 1 copy (use `{{ $json.firstName }}` and `{{ $json.applyLink }}`)
   - Connect email node to the “R1 Airtable update” node (step 8).

7) **Reminder 1 path: SMS gate + send**
   - Node: **IF** (`R1 — IF Has Phone`)
     - Condition: String → “exists” → `{{ $json.phoneE164 }}`
   - Node: **HTTP Request** (`Brevo Sms1`)
     - Method: POST
     - URL: `https://api.brevo.com/v3/transactionalSMS/send`
     - Authentication: preferably store API key in an n8n credential or env var (avoid hardcoding)
     - Headers: `accept: application/json`, `content-type: application/json`, `api-key: <your key>`
     - JSON body: includes recipient `{{ $json.phoneE164 }}` and your SMS content
   - Connect SMS node to the “R1 Airtable update” node (step 8).

8) **Reminder 1 path: Airtable update**
   - Node: **Airtable** (`R1 — Airtable Mark Reminder 1 Sent`)
   - Operation: **Update**
   - Record ID: `{{ $json.recordId }}` (recommended; avoid referencing another node’s item)
   - Fields to set: `Reminder 1 Sent = true`

9) **Reminder 2 path: Email gate + send**
   - Node: **IF** (`R2 — IF Has Email`) with same “exists” check on `{{ $json.email }}`
   - Node: **Brevo / Sendinblue** (`Send a transactional email`)
     - To: `{{ $json.email }}`
     - Reminder 2 copy (often more urgent)
   - Connect to “R2 Airtable update” node (step 11).

10) **Reminder 2 path: SMS gate + send**
   - Node: **IF** (`R2 — IF Has Phone`) with “exists” check on `{{ $json.phoneE164 }}`
   - Node: **HTTP Request** (`Brevo Sms`)
     - Same endpoint/headers structure as Reminder 1 SMS
     - Reminder 2 SMS content
   - Connect to “R2 Airtable update” node (step 11).

11) **Reminder 2 path: Airtable update**
   - Node: **Airtable** (`R2 — Airtable Mark Reminder 2 + Urgent Call`)
   - Operation: **Update**
   - Record ID: `{{ $json.recordId }}`
   - Fields to set:
     - `Reminder 2 Sent = true`
     - `Urgent Call Required = true`

12) **Optional: completion marker**
   - Node: **Set** (`WF3 Complete`)
   - Keep only set: true
   - Set `result = "WF3 finished"`
   - Connect both Airtable update nodes to it (or end directly).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automated Application Reminder Sequence (Email + SMS)” explanation, setup steps, and customization guidance | Provided in the large workflow sticky note (internal documentation inside the canvas). |
| Apply link used in messages | https://www.morestarts.co.uk/welding-course/ |
| Security note: Brevo SMS API key is hardcoded in HTTP nodes | Move to credentials or environment variables to avoid leaking secrets and ease rotation. |
| Routing correctness note | Current `router` IF logic is inconsistent (checks for routeCode 2 AND routeCode 1). Consider a Switch node keyed on `routeCode`. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.