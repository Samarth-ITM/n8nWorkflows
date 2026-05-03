Triage and escalate HubSpot tickets to Jira with Slack SLA alerts

https://n8nworkflows.xyz/workflows/triage-and-escalate-hubspot-tickets-to-jira-with-slack-sla-alerts-12940


# Triage and escalate HubSpot tickets to Jira with Slack SLA alerts

## 1. Workflow Overview

**Purpose:** Monitor newly created HubSpot tickets, enrich them with associated Contact business context (revenue + lifecycle stage), compute a **severity** (CRITICAL/High/Normal) using keyword + revenue logic, then **create a Jira issue**, notify **Slack**, and finally enforce an **SLA buffer**: if the Jira issue is still not being worked after **15 minutes**, send an escalation alert in Slack.

**Target use cases:**
- Customer Success / Support triage for VIP customers
- Churn-risk early warning (keyword-based)
- Ensuring response SLAs via Slack escalation if Jira status doesn’t change

### 1.1 Schedule Trigger & Ticket Intake (HubSpot Search)
Runs every 10 minutes and searches for tickets created in the last 10 minutes; stops early when no tickets are found.

### 1.2 Per-Ticket Processing & HubSpot Enrichment
Processes each ticket one by one, loads associated contact email(s), and fetches the contact’s business attributes.

### 1.3 Severity/Triage Logic → Jira Creation → Slack Notification
Computes severity and assignee, creates a Jira issue with a rich description, then posts a Slack notification.

### 1.4 SLA Timer & Escalation Check (Jira → Slack)
Waits 15 minutes, checks Jira status; if not “Done” and not “In Progress”, posts an escalation alert in Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule Trigger & Ticket Intake (HubSpot Search)
**Overview:** Triggers periodically, searches HubSpot for tickets created within the last 10 minutes, and stops if no tickets exist to avoid unnecessary processing.

**Nodes involved:**
- Every 10 Mins
- HubSpot: Search New Tickets
- Check: Are there tickets?

#### Node: **Every 10 Mins**
- **Type / role:** Schedule Trigger — initiates workflow on a fixed interval.
- **Configuration:** Runs every **10 minutes**.
- **Inputs/Outputs:** Entry node → outputs to “HubSpot: Search New Tickets”.
- **Edge cases/failures:** If n8n instance is paused/down, runs may be delayed/missed.

#### Node: **HubSpot: Search New Tickets**
- **Type / role:** HTTP Request — calls HubSpot CRM Search API for tickets.
- **Configuration choices:**
  - `POST https://api.hubapi.com/crm/v3/objects/tickets/search`
  - JSON body filters by `createdate >= $now.minus({minutes: 10}).toMillis()`
  - Sorts by `createdate DESCENDING`
  - Requests properties: `subject`, `content`, `hs_ticket_category`, `hs_pipeline_stage`
  - Uses **HubSpot App Token** credential (`hubspotAppToken`) via predefined credential type
- **Key expressions/variables:**
  - `{{ $now.minus({minutes: 10}).toMillis() }}`
- **Input/Output connections:** From trigger → to “Check: Are there tickets?”
- **Edge cases/failures:**
  - Auth failure (invalid/expired app token)
  - HubSpot rate limits (429)
  - Time window overlap can still produce duplicates if reruns occur or tickets appear late (no explicit de-dupe storage)

#### Node: **Check: Are there tickets?**
- **Type / role:** IF — guard clause to stop when no results.
- **Configuration choices:**
  - Condition: `{{ $json.total }} > 0`
- **Input/Output connections:**
  - Input: search response
  - True path → “Loop: Process Tickets”
  - False path: no outgoing connection (workflow ends)
- **Edge cases/failures:**
  - If HubSpot response shape changes or `total` missing, expression may error under strict validation.

---

### Block 2 — Per-Ticket Processing & HubSpot Enrichment
**Overview:** Splits the search results into individual tickets, fetches associated ticket data including contact emails, ensures an email exists, then looks up Contact business data by email.

**Nodes involved:**
- Loop: Process Tickets
- HubSpot: Get Associations
- Filter: Has Email?
- HubSpot: Get Contact Data

#### Node: **Loop: Process Tickets**
- **Type / role:** Split Out — iterates over `results[]` array from HubSpot search.
- **Configuration choices:** `fieldToSplitOut: results`
- **Input/Output connections:** From IF true path → to “HubSpot: Get Associations”
- **Edge cases/failures:**
  - If `results` is absent/empty, node can output nothing (effectively ends downstream execution).

#### Node: **HubSpot: Get Associations**
- **Type / role:** HubSpot node — retrieves ticket record (and requested properties).
- **Configuration choices:**
  - Resource: `ticket`
  - Operation: `get`
  - Ticket ID: `={{ $json.id }}`
  - OAuth2 authentication (HubSpot OAuth2)
  - Additional properties requested: `hs_all_associated_contact_emails`
- **Inputs/Outputs:** From SplitOut ticket → to “Filter: Has Email?”
- **Version requirements:** HubSpot node v2.2 (behavior depends on n8n HubSpot node version).
- **Edge cases/failures:**
  - OAuth token expired/revoked
  - Ticket exists but property missing
  - Assumes returned structure includes `.properties.hs_all_associated_contact_emails` and possibly `.versions[0].value`

#### Node: **Filter: Has Email?**
- **Type / role:** Filter — ensures ticket has an associated contact email before proceeding.
- **Configuration choices:**
  - Checks existence of:  
    `={{ $json.properties.hs_all_associated_contact_emails.versions[0].value }}`
  - “exists” string operator (strict validation enabled)
- **Inputs/Outputs:** From HubSpot get ticket → to “HubSpot: Get Contact Data”
- **Edge cases/failures:**
  - If `hs_all_associated_contact_emails` doesn’t have `.versions[0].value` but has `.value`, this condition can fail even when an email exists (and filtering will drop valid tickets).
  - If `versions` is undefined, expression may error depending on n8n evaluation behavior.

#### Node: **HubSpot: Get Contact Data**
- **Type / role:** HTTP Request — searches contacts by email to fetch business attributes.
- **Configuration choices:**
  - `POST https://api.hubapi.com/crm/v3/objects/contacts/search`
  - Filters `email EQ <email>` where email is derived from associated emails:
    - `($json.properties.hs_all_associated_contact_emails.value || $json.properties.hs_all_associated_contact_emails.versions[0].value).split(';')[0]`
  - Requests properties: `annualrevenue`, `lifecyclestage`, `company`
  - Uses **HubSpot App Token** credential (`hubspotAppToken`)
- **Inputs/Outputs:** From Filter → to “Code: Calculate Severity”
- **Edge cases/failures:**
  - If neither `.value` nor `.versions[0].value` exists, `.split()` will throw
  - If multiple emails are semicolon-separated, only the first is used
  - Search may return `results: []` → downstream code expects `results[0]`

---

### Block 3 — Severity/Triage Logic → Jira Creation → Slack Notification
**Overview:** Computes severity and assignee using revenue, lifecycle stage, and churn keywords; creates a Jira issue with formatted context; posts a Slack message.

**Nodes involved:**
- Code: Calculate Severity
- Jira: Create Triage Ticket
- Slack: Notify Channel

#### Node: **Code: Calculate Severity**
- **Type / role:** Code node (JavaScript) — transforms ticket+contact into triage payload.
- **Configuration choices (logic):**
  - Reads contact search response from current input.
  - Reads ticket data from **another node** via: `$('HubSpot: Get Associations').item.json`
  - Extracts:
    - `annualrevenue` → `revenue` (float)
    - `lifecyclestage` → `stage`
    - ticket `subject`, `content`
  - Churn keywords: `["cancel","refund","expensive","lawyer","competitor","broken"]`
  - Severity rules:
    - High revenue (`> 10000`) + churn risk → `CRITICAL`, owner `head-of-cs`
    - High revenue only → `High`, owner `senior-csm`
    - Churn risk only → `High`, owner `retention-team`
    - Else `Normal`, owner `support-general`
  - Outputs fields: `severity`, `assigned_to`, `slack_message`, `jira_summary`, etc.
- **Key expressions/variables:** Uses `$input.item.json`, `$() node item access`.
- **Inputs/Outputs:** From contact search → to “Jira: Create Triage Ticket”
- **Edge cases/failures:**
  - If contact search returns empty `results`, `results[0]` is undefined → runtime error
  - `contactProps.email` may be undefined because the search request does not explicitly request the `email` property (HubSpot often includes it anyway, but not guaranteed)
  - Reliance on cross-node access (`$('HubSpot: Get Associations')...`) assumes item pairing remains aligned; usually OK in a linear per-item flow, but can break with merges/parallel branches.

#### Node: **Jira: Create Triage Ticket**
- **Type / role:** Jira node — creates a Jira issue for triage.
- **Configuration choices:**
  - Project: selected from list (not specified in JSON)
  - Issue Type: selected from list (not specified in JSON)
  - Summary: `={{ $json.jira_summary }}`
  - Description includes formatted Jira markup and references:
    - Customer email from **Filter node output**:  
      `{{ $('Filter: Has Email?').item.json.properties.hs_all_associated_contact_emails.value }}`
    - Revenue/stage/severity/assignee from Code output
- **Inputs/Outputs:** From Code → to “Slack: Notify Channel”
- **Edge cases/failures:**
  - Jira auth/permission errors (cannot create issues)
  - If the referenced email path uses `.value` but HubSpot provided only `.versions[0].value`, customer line may be blank or expression error
  - Jira field constraints (required fields, project/issuetype mismatch)

#### Node: **Slack: Notify Channel**
- **Type / role:** Slack node — posts triage alert to a channel.
- **Configuration choices:**
  - Channel: `customer-success`
  - Text: `={{ $('Code: Calculate Severity').item.json.slack_message }}`
- **Inputs/Outputs:** From Jira create → to “Wait: Response Timer”
- **Edge cases/failures:**
  - Slack OAuth/auth missing scopes (chat:write)
  - Channel name vs ID ambiguity depending on Slack node configuration/version
  - Message includes `@${owner}` but those are not real Slack user IDs; it will not mention a user unless Slack interprets it as plain text.

---

### Block 4 — SLA Timer & Escalation Check (Jira → Slack)
**Overview:** Waits 15 minutes, fetches Jira issue status, and if still not being handled, escalates via Slack.

**Nodes involved:**
- Wait: Response Timer
- Jira: Get Latest Status
- Check: Escalation Needed?
- Slack: Send Alert

#### Node: **Wait: Response Timer**
- **Type / role:** Wait — delay to implement SLA buffer.
- **Configuration choices:** Waits `15 minutes`.
- **Inputs/Outputs:** From initial Slack notification → to “Jira: Get Latest Status”
- **Edge cases/failures:**
  - n8n restarts can affect wait state depending on queue/execution persistence setup
  - High volume can create many waiting executions (resource planning)

#### Node: **Jira: Get Latest Status**
- **Type / role:** Jira node — retrieves issue fields including current status.
- **Configuration choices:**
  - Operation: `get`
  - Issue key: `={{ $('Jira: Create Triage Ticket').item.json.key }}`
- **Inputs/Outputs:** From Wait → to “Check: Escalation Needed?”
- **Edge cases/failures:**
  - Issue deleted or key invalid
  - Jira rate limits / transient errors

#### Node: **Check: Escalation Needed?**
- **Type / role:** IF — decides whether to alert based on Jira status.
- **Configuration choices (string conditions):**
  - If `status.name != "Done"` AND `status.name != "In Progress"` → escalate
- **Inputs/Outputs:** From Jira get → True path to “Slack: Send Alert”; False path ends.
- **Edge cases/failures:**
  - Status naming varies by Jira workflow (e.g., “To Do”, “Selected for Development”, “Open”). This logic escalates for any status other than exactly “Done” or “In Progress”.

#### Node: **Slack: Send Alert**
- **Type / role:** Slack node — posts escalation alert.
- **Configuration choices:**
  - Channel: `customer-success`
  - Text: `🔥 *CHURN RISK ESCALATION*: VIP Ticket {{ $json.key }} untouched for 15 mins. @channel`
- **Inputs/Outputs:** From IF true path; no further nodes.
- **Edge cases/failures:**
  - `{{ $json.key }}` at this point depends on Jira Get output. Jira Get typically returns `key` at top-level; if not, it might be under a different path depending on node/version.
  - `@channel` works only if permitted and in channels (not DMs).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 10 Mins | Schedule Trigger | Periodic trigger (every 10 minutes) | — | HubSpot: Search New Tickets | ## Workflow Setup & Trigger; ## Workflow Overview |
| HubSpot: Search New Tickets | HTTP Request | Search for new HubSpot tickets created in last 10 min | Every 10 Mins | Check: Are there tickets? | ## Workflow Setup & Trigger; ## Workflow Overview |
| Check: Are there tickets? | IF | Stop workflow if no tickets | HubSpot: Search New Tickets | Loop: Process Tickets (true path) | ## Workflow Setup & Trigger; ## Workflow Overview |
| Loop: Process Tickets | Split Out | Iterate each ticket in results | Check: Are there tickets? | HubSpot: Get Associations | ## Data Enrichment; ## Workflow Overview |
| HubSpot: Get Associations | HubSpot | Fetch ticket + associated contact emails | Loop: Process Tickets | Filter: Has Email? | ## Data Enrichment; ## Workflow Overview |
| Filter: Has Email? | Filter | Ensure associated email exists before continuing | HubSpot: Get Associations | HubSpot: Get Contact Data | ## Data Enrichment; ## Workflow Overview |
| HubSpot: Get Contact Data | HTTP Request | Search contact by email; fetch revenue/stage/company | Filter: Has Email? | Code: Calculate Severity | ## Data Enrichment; ## Workflow Overview |
| Code: Calculate Severity | Code | Compute severity, owner, Slack/Jira strings | HubSpot: Get Contact Data | Jira: Create Triage Ticket | ## Logic & Triage; ## Workflow Overview |
| Jira: Create Triage Ticket | Jira | Create Jira issue for triage | Code: Calculate Severity | Slack: Notify Channel | ## Logic & Triage; ## Workflow Overview |
| Slack: Notify Channel | Slack | Post immediate triage notification | Jira: Create Triage Ticket | Wait: Response Timer | ## Logic & Triage; ## Workflow Overview |
| Wait: Response Timer | Wait | Delay 15 minutes (SLA buffer) | Slack: Notify Channel | Jira: Get Latest Status | ## SLA Monitor & Escalation; ## Workflow Overview |
| Jira: Get Latest Status | Jira | Fetch Jira issue current status | Wait: Response Timer | Check: Escalation Needed? | ## SLA Monitor & Escalation; ## Workflow Overview |
| Check: Escalation Needed? | IF | Escalate if status not Done and not In Progress | Jira: Get Latest Status | Slack: Send Alert (true path) | ## SLA Monitor & Escalation; ## Workflow Overview |
| Slack: Send Alert | Slack | Post escalation alert after SLA breach | Check: Escalation Needed? | — | ## SLA Monitor & Escalation; ## Workflow Overview |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note2 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note4 | Sticky Note | Comment block | — | — |  |
| Sticky Note7 | Sticky Note | Comment block | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Workflow**
   - Name it: **“Triage and escalate HubSpot tickets to Jira with Slack SLA alerts”**.

2. **Add Trigger**
   - Add **Schedule Trigger** node named **Every 10 Mins**.
   - Set interval: **Every 10 minutes**.

3. **Search HubSpot tickets (last 10 minutes)**
   - Add **HTTP Request** node named **HubSpot: Search New Tickets**.
   - Method: **POST**
   - URL: `https://api.hubapi.com/crm/v3/objects/tickets/search`
   - Authentication: **Predefined Credential Type → HubSpot App Token** (create/select credential).
   - Body: JSON with:
     - filter on `createdate` **GTE** `{{$now.minus({minutes: 10}).toMillis()}}`
     - request properties: `subject`, `content`, `hs_ticket_category`, `hs_pipeline_stage`
   - Connect: **Every 10 Mins → HubSpot: Search New Tickets**.

4. **Guard clause (stop if no tickets)**
   - Add **IF** node named **Check: Are there tickets?**
   - Condition (Number): `{{$json.total}}` **greater than** `0`.
   - Connect: **HubSpot: Search New Tickets → Check: Are there tickets?**

5. **Split results into individual items**
   - Add **Split Out** node named **Loop: Process Tickets**
   - Field to split out: `results`
   - Connect IF **true** output to Split Out:  
     **Check: Are there tickets? (true) → Loop: Process Tickets**

6. **Fetch ticket data including associated contact emails**
   - Add **HubSpot** node named **HubSpot: Get Associations**
   - Authentication: **OAuth2** (connect HubSpot OAuth app in n8n)
   - Resource: **Ticket**
   - Operation: **Get**
   - Ticket ID: `{{$json.id}}`
   - Additional properties: include `hs_all_associated_contact_emails`
   - Connect: **Loop: Process Tickets → HubSpot: Get Associations**

7. **Filter out tickets without email**
   - Add **Filter** node named **Filter: Has Email?**
   - Condition: “String exists” on  
     `{{$json.properties.hs_all_associated_contact_emails.versions[0].value}}`
   - Connect: **HubSpot: Get Associations → Filter: Has Email?**

8. **Search contact by email and fetch business attributes**
   - Add **HTTP Request** node named **HubSpot: Get Contact Data**
   - Method: **POST**
   - URL: `https://api.hubapi.com/crm/v3/objects/contacts/search`
   - Authentication: **HubSpot App Token** credential (same as step 3)
   - Body: JSON that filters `email EQ`:
     - `{{ ($json.properties.hs_all_associated_contact_emails.value || $json.properties.hs_all_associated_contact_emails.versions[0].value).split(';')[0] }}`
   - Request properties: `annualrevenue`, `lifecyclestage`, `company`
   - Connect: **Filter: Has Email? → HubSpot: Get Contact Data**

9. **Compute severity + payload**
   - Add **Code** node named **Code: Calculate Severity**
   - Paste JavaScript implementing:
     - revenue parsing, lifecycle stage extraction
     - churn keyword scan on ticket content
     - severity + owner selection
     - output `slack_message` + `jira_summary`
   - Ensure the code references ticket data via:  
     `$('HubSpot: Get Associations').item.json`
   - Connect: **HubSpot: Get Contact Data → Code: Calculate Severity**

10. **Create Jira issue**
   - Add **Jira** node named **Jira: Create Triage Ticket**
   - Configure Jira credentials (API token / OAuth depending on your Jira setup in n8n).
   - Operation: **Create Issue**
   - Select **Project** and **Issue Type** (must exist in Jira).
   - Summary: `{{$json.jira_summary}}`
   - Description: include customer email + revenue/stage/severity/assignee (use expressions from Code node and the Filter/HubSpot nodes as in the provided logic).
   - Connect: **Code: Calculate Severity → Jira: Create Triage Ticket**

11. **Notify Slack channel immediately**
   - Add **Slack** node named **Slack: Notify Channel**
   - Credentials: Slack OAuth with `chat:write`
   - Channel: `customer-success` (or choose channel ID)
   - Text: `{{ $('Code: Calculate Severity').item.json.slack_message }}`
   - Connect: **Jira: Create Triage Ticket → Slack: Notify Channel**

12. **Wait 15 minutes (SLA buffer)**
   - Add **Wait** node named **Wait: Response Timer**
   - Unit: minutes; Amount: **15**
   - Connect: **Slack: Notify Channel → Wait: Response Timer**

13. **Re-check Jira issue status**
   - Add **Jira** node named **Jira: Get Latest Status**
   - Operation: **Get**
   - Issue Key: `{{ $('Jira: Create Triage Ticket').item.json.key }}`
   - Connect: **Wait: Response Timer → Jira: Get Latest Status**

14. **Escalation decision**
   - Add **IF** node named **Check: Escalation Needed?**
   - Conditions (String):
     - `{{$json.fields.status.name}}` **not equal** `Done`
     - `{{$json.fields.status.name}}` **not equal** `In Progress`
   - Connect: **Jira: Get Latest Status → Check: Escalation Needed?**

15. **Send escalation Slack alert**
   - Add **Slack** node named **Slack: Send Alert**
   - Channel: `customer-success`
   - Text: `🔥 *CHURN RISK ESCALATION*: VIP Ticket {{ $json.key }} untouched for 15 mins. @channel`
   - Connect: **Check: Escalation Needed? (true) → Slack: Send Alert**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Schedule: Runs every 10 minutes to check for new tickets… prevents duplicates… stops execution if empty.” | Workflow Setup & Trigger (sticky note) |
| “Loop… Associations… Contact Details… Annual Revenue, Lifecycle Stage… identify VIP customers.” | Data Enrichment (sticky note) |
| “Severity Calculation… classify tickets… Jira Task… Slack alert.” | Logic & Triage (sticky note) |
| “Wait Timer 15 minutes… Status Check… Escalation in Slack.” | SLA Monitor & Escalation (sticky note) |
| “Automated VIP Ticket Escalation… Trigger… Triage… Sync… SLA Check…” | Workflow Overview (sticky note) |
| Contact: thomas@pollup.net | mailto:thomas@pollup.net |
| Other workflows link | https://n8n.io/creators/zeerobug/ |