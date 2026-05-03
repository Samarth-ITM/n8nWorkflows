Send candidate outcome emails and SMS and notify referrers with Airtable

https://n8nworkflows.xyz/workflows/send-candidate-outcome-emails-and-sms-and-notify-referrers-with-airtable-13062


# Send candidate outcome emails and SMS and notify referrers with Airtable

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title (given):** Send candidate outcome emails and SMS and notify referrers with Airtable  
**Workflow name (in JSON):** `wf3` (inactive)  
**Purpose:**  
This workflow listens for outcome changes on Airtable candidate records (info event outcomes and course outcomes). When an outcome changes, it sends the appropriate candidate email and/or SMS via Brevo (Sendinblue) and then updates Airtable “sent/notified” checkbox flags to ensure each message is only sent once. It also notifies referrers at certain milestones.

### 1.1 Trigger & Event Detection (Airtable-driven)
Two Airtable Trigger nodes poll timestamp fields that Airtable updates when outcomes change. Each trigger is tagged with an `eventType` (“info” or “course”) so the workflow can share a common downstream logic path.

### 1.2 Normalize Data & Decide Actions (central decision layer)
A single Code node normalizes candidate fields (name, email, phone), reads outcomes + “already sent” flags, and decides:
- the **candidate messaging action** route (exactly one of: `info_attended`, `info_noshow`, `course`, or `none`)
- whether to notify the referrer at specific milestones.

### 1.3 Candidate Messaging (Info outcomes + Course outcomes)
A Switch routes to message nodes:
- Info Attended: email + SMS
- Info No-Show: email + SMS
- Course outcome: email (subject/body vary by normalized outcome key)

### 1.4 Referrer Notifications + Commit Flags
After candidate messaging, Airtable is updated to mark messages as sent; then a referrer email may be sent and committed as notified. Finally, a “complete” Set node ends the workflow.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Event Detection

**Overview:**  
Detects outcome changes in Airtable by polling “updated at” timestamp fields. Adds an `eventType` to distinguish info-event vs course-event executions.

**Nodes involved:**
- Airtable Trigger — Info Outcome Updated At1
- Tag eventType = info
- Course Outcome Updated At
- Tag eventType = course
- Merge — Any Trigger

#### Node: Airtable Trigger — Info Outcome Updated At1
- **Type / role:** Airtable Trigger (polling). Starts workflow when `InfoOutcomeUpdatedAt_TMP` changes.
- **Config highlights:**
  - Base ID / Table ID: placeholder values (`Your_Base_id`, `Your_table_id`)
  - Poll schedule: every minute
  - Trigger field: `InfoOutcomeUpdatedAt_TMP`
  - Auth: Airtable Personal Access Token
- **Outputs:** Airtable record payload, including `id` and `fields`.
- **Downstream connection:** → `Tag eventType = info`
- **Edge cases / failures:**
  - Misconfigured base/table IDs cause runtime errors.
  - If Airtable does not update the trigger field reliably, workflow won’t fire.
  - Polling triggers can miss rapid consecutive updates depending on Airtable + polling cadence.

#### Node: Tag eventType = info
- **Type / role:** Set node. Adds `eventType = "info"` to the item.
- **Config:** Sets a single string value `eventType`.
- **Downstream connection:** → `Merge — Any Trigger`
- **Edge cases:** none significant.

#### Node: Course Outcome Updated At
- **Type / role:** Airtable Trigger (polling). Starts workflow when `Course Outcome Updated At` changes.
- **Config highlights:**
  - Base ID / Table ID: placeholders (`Your_Base_ID`, `Your_Table_ID`)
  - Poll schedule: every minute
  - Trigger field: `Course Outcome Updated At`
  - Auth: Airtable Personal Access Token
- **Downstream connection:** → `Tag eventType = course`
- **Edge cases / failures:** same as Info trigger (IDs/auth/polling cadence).

#### Node: Tag eventType = course
- **Type / role:** Set node. Adds `eventType = "course"`.
- **Downstream connection:** → `Merge — Any Trigger`

#### Node: Merge — Any Trigger
- **Type / role:** Merge node. Combines the two trigger entry points into a single flow.
- **Config:** Default merge configuration (acts as a funnel here).
- **Downstream connection:** → `Code — Normalize + Decide Action`
- **Edge cases:** if both triggers fire close together, you get separate executions (not a true join), which is typically desired here.

**Associated sticky note (applies to this block):**
- “## Trigger & Event Detection … Watches Airtable-managed timestamp fields for outcome changes. Airtable controls when the workflow runs.”

---

### Block 2.2 — Normalize & Decide Action (core logic)

**Overview:**  
Normalizes Airtable record fields into stable keys (first name, email, E.164 phone). Determines a single candidate messaging action and computes referrer notification booleans based on outcomes and Airtable “sent/notified” flags.

**Nodes involved:**
- Code — Normalize + Decide Action
- Switch — Route Action

#### Node: Code — Normalize + Decide Action
- **Type / role:** Code node (JavaScript). Central business logic.
- **What it does (interpreted):**
  - Normalizes strings with trimming (`safeStr`)
  - Normalizes checkbox-ish values into booleans (`normalizeBool`)
  - Normalizes phone into E.164-ish format by prefixing `+` if missing (`toE164`)
  - Reads Airtable fields:
    - Candidate identity: `First  Name` / `First Name` variants, `Email`, `Phone Number`
    - Outcomes: `Info Event Outcome`, `Course Outcome`
    - Candidate sent flags: `Post-Info Attended Sent`, `Post-Info NoShow Sent`, `Course Outcome Message Sent`
    - Referrer email: `Referrer Email` or `Referring Person Email` (note: later nodes use a different field name)
    - Referrer notified flags: 
      - `Referrer Notified – Info Outcome` (code expects this exact name)
      - `Referrer Notified – Course Start`
      - `Referrer Notified – Course Outcome` (for “end”)
  - Computes:
    - `courseOutcomeKey`: `started`, `not_started_withdrawn`, `completed`, `other`, or empty
    - `action`: one of `info_attended`, `info_noshow`, `course`, `none`
    - Referrer booleans:
      - `notifyReferrerInfo`
      - `notifyReferrerCourseStart` (only `started`)
      - `notifyReferrerCourseEnd` (only `completed` or `not_started_withdrawn`)
  - Adds constants like `applyLink`, `senderName`, `senderEmail`
- **Inputs:** From `Merge — Any Trigger`. Expects `item.json.fields` and `item.json.id`, plus `item.json.eventType`.
- **Outputs:** Replaces each item with a normalized `json` object containing (key set):
  - `recordId`, `eventType`, `action`
  - `firstName`, `email`, `phoneE164`, `referrerEmail`
  - `infoOutcome`, `courseOutcome`, `courseOutcomeKey`
  - `notifyReferrerInfo`, `notifyReferrerCourseStart`, `notifyReferrerCourseEnd`
- **Downstream connection:** → `Switch — Route Action`
- **Edge cases / failures:**
  - Field name mismatches: Airtable uses `Reffering Person's Email` elsewhere, but code reads `Referrer Email` / `Referring Person Email`. This can cause `referrerEmail` to be empty even when the table has a referrer.
  - Boolean normalization assumes values like `"checked"` or `"true"`; Airtable checkbox fields are typically boolean true/false, which is fine.
  - Phone normalization is simplistic (does not validate country codes).

#### Node: Switch — Route Action
- **Type / role:** Switch router. Sends item to the correct messaging branch based on `action`.
- **Config highlights:**
  - Value: `{{ $json.action }}`
  - Rules:
    - `info_attended` → output 0
    - `info_noshow` → output 1
    - `course` → output 2
- **Outputs:**
  - Output 0 → attended messaging nodes
  - Output 1 → no-show messaging nodes
  - Output 2 → course outcome email node
- **Edge cases:** if `action = none` there is **no default route configured**, so the execution effectively stops here without updating any flags. That may be intended, but it means referrer notifications will not happen unless tied to later branches.

**Associated sticky note (applies to this block):**
- “## Normalize & Decide Action … Prevents fragile IF chains.”

---

### Block 2.3 — Candidate Messaging — Info Event Outcomes

**Overview:**  
Sends email + SMS for info-event outcomes (attended or no-show) and commits “sent” flags to Airtable to guarantee one-time sending.

**Nodes involved:**
- Attended email
- Attended sms1
- Airtable — Commit Post-Info Attended Sent
- Did Not Attended sms
- Did Not Attended email
- Airtable — Commit Post-Info NoShow Sent

#### Node: Attended email
- **Type / role:** Brevo (Sendinblue) transactional email send.
- **Config highlights:**
  - Sender: `user@example.com`
  - Recipient: `{{ $json.email }}`
  - Subject: `Course Place Confirmed`
  - Body: static template text, with moustache-like placeholders inside a raw string (not all of these are n8n expressions).
- **Input:** From Switch output 0.
- **Output:** → `Airtable — Commit Post-Info Attended Sent`
- **Edge cases / failures:**
  - If `$json.email` empty/invalid, Brevo will likely error.
  - The text uses `{{ $json.firstName || $json.fields['First Name'] }}` but after normalization you typically only have `$json.firstName`; `$json.fields` no longer exists. The literal `{{ ... }}` inside `textContent` will not be evaluated unless the node specifically supports templating beyond n8n expressions. Result: candidate may receive unrendered placeholders.

#### Node: Attended sms1
- **Type / role:** HTTP Request to Brevo transactional SMS API.
- **Config highlights:**
  - POST `https://api.brevo.com/v3/transactionalSMS/send`
  - Headers include `api-key: YOUR-API-KEY` (placeholder), JSON content-type
  - JSON body includes:
    - recipient: `"{{ $json.phoneE164 || $json.fields['Phone Number'] }}"`
    - content contains `{{ ... }}` placeholders
- **Input:** From Switch output 0.
- **Output:** → `Airtable — Commit Post-Info Attended Sent`
- **Edge cases / failures:**
  - Uses a hardcoded API key header; in production should be a credential or env var.
  - Same placeholder issue: `$json.fields` likely not present and `{{ }}` is not evaluated by n8n unless inside an n8n expression context.
  - If phone is missing/invalid, API returns error.

#### Node: Airtable — Commit Post-Info Attended Sent
- **Type / role:** Airtable Update. Marks the attended message as sent.
- **Config highlights:**
  - Operation: Update record
  - **Record ID expression:** `{{ $('Airtable Trigger — Info Outcome Updated At1').item.json.id }}`
  - **Important:** In the provided mapping, `Post-Info Attended Sent` does **not** appear explicitly set to `true` in `columns.value` (unlike the NoShow commit node). This means the update may not actually set the flag unless configured elsewhere in the UI.
- **Input:** From `Attended email` and `Attended sms1` (both feed this node).
- **Output:** → `Brevo Email — Referrer Info Outcome`
- **Edge cases / failures:**
  - If the execution was triggered by the **course** trigger, this node still references the **info trigger** record id, which could be undefined. However, it only runs on info branches, so typically OK.
  - If both email and SMS run, this node is called twice (once per path) → can cause redundant writes, and downstream referrer email could also run twice.

#### Node: Did Not Attended sms
- **Type / role:** HTTP Request to Brevo transactional SMS API (no-show content).
- **Input:** Switch output 1.
- **Output:** → `Airtable — Commit Post-Info NoShow Sent`
- **Edge cases:** same as attended SMS.

#### Node: Did Not Attended email
- **Type / role:** Brevo email (no-show).
- **Input:** Switch output 1.
- **Output:** → `Airtable — Commit Post-Info NoShow Sent`
- **Edge cases:** same as attended email (placeholders + missing `$json.fields`).

#### Node: Airtable — Commit Post-Info NoShow Sent
- **Type / role:** Airtable Update.
- **Config highlights:**
  - Record ID expression: `{{ $('Airtable Trigger — Info Outcome Updated At1').item.json.id }}`
  - Sets `Post-Info NoShow Sent: true`
- **Input:** From both no-show email and SMS.
- **Output:** → `Brevo Email — Referrer Info Outcome`
- **Edge cases:**
  - Like attended commit: may run twice (email + SMS) causing duplicate downstream referrer email unless guarded.

**Associated sticky note (applies to this block):**
- “## Candidate Messaging — Info Event Outcomes … Each message is sent once only.”

---

### Block 2.4 — Candidate Messaging — Course Outcomes

**Overview:**  
Sends a course outcome email to the candidate (started/withdrawn/completed) and then commits a “course outcome message sent” flag to Airtable.

**Nodes involved:**
- Course outcome email
- Airtable — Commit Course Outcome Message Sent1

#### Node: Course outcome email
- **Type / role:** Brevo email with dynamic subject/body based on `courseOutcomeKey`.
- **Config highlights:**
  - Recipient: `{{ $json.email }}`
  - Subject: expression chooses subject text based on `courseOutcomeKey`
  - Body: expression builds a string template for started / not_started_withdrawn / completed / fallback
- **Input:** Switch output 2.
- **Output:** → `Airtable — Commit Course Outcome Message Sent1`
- **Edge cases:**
  - Missing/invalid email.
  - `courseOutcomeKey` might be `other` or empty depending on Airtable values; fallback branch used.

#### Node: Airtable — Commit Course Outcome Message Sent1
- **Type / role:** Airtable Update.
- **Config highlights:**
  - Record ID: `{{ $('Course Outcome Updated At').item.json.id }}`
  - Sets `Course Outcome Message Sent`: **false** (this looks like a bug; should almost certainly be `true`)
  - Base/table values: table is set to `Your_Base_Id` (appears incorrect; should be a table ID)
- **Input:** from course outcome email.
- **Output:** → `If1`
- **Edge cases / failures:**
  - Incorrect “table” parameter will cause Airtable errors.
  - Writing `false` breaks idempotency: workflow will keep re-sending because the “sent” flag never becomes true.

**Associated sticky note (applies to this block):**
- “## Candidate Messaging — Course Outcomes … Content is driven by normalized outcome keys.”

---

### Block 2.5 — Referrer Notifications

**Overview:**  
Sends emails to referrers after certain events. In the current wiring, info referrer notifications always run after the info branches’ “commit sent” nodes; course referrer notification is guarded by an If node (but only for course start in the provided logic).

**Nodes involved:**
- Brevo Email — Referrer Info Outcome
- Airtable — Commit Referrer Notified (Info) [No-Show branch]
- If1
- Brevo Email — Referrer Course Outcome1
- Referrer notified course start

#### Node: Brevo Email — Referrer Info Outcome
- **Type / role:** Brevo email to the referrer with info outcome.
- **Config highlights:**
  - Recipient: `{{ $json.fields['Reffering Person\'s Email'] }}`
  - Body references `${$json.firstName}` and `${$json.infoOutcome}`
- **Input:** from both info commit nodes.
- **Output:** → `Airtable — Commit Referrer Notified (Info) [No-Show branch]`
- **Edge cases / failures:**
  - After Code normalization, `$json.fields` is not present. So the recipient expression likely resolves to empty → email fails.
  - There is no conditional guard checking `notifyReferrerInfo`; it will attempt to send whenever it reaches this node.
  - Potential duplicates: because both email and SMS paths can trigger commit nodes independently, this referrer email can be sent twice.

#### Node: Airtable — Commit Referrer Notified (Info) [No-Show branch]
- **Type / role:** Airtable Update marking referrer notified (info).
- **Config highlights:**
  - Record ID: `{{ $('Airtable Trigger — Info Outcome Updated At1').item.json.id }}`
  - Sets `Referrer Notified – Info Event: true` (note naming differs from Code node expectation “Referrer Notified – Info Outcome”)
- **Input:** from `Brevo Email — Referrer Info Outcome`
- **Output:** → `WF6 Complete`
- **Edge cases:**
  - Field name mismatch vs Code node means Code may not see it as already notified, causing repeat sends.

#### Node: If1
- **Type / role:** If node guarding course referrer notification.
- **Condition (interpreted):**
  - Checks `{{ $json.notifyReferrerCourseStart }}` is true (configured as a “boolean true” operation).
- **Input:** from `Airtable — Commit Course Outcome Message Sent1`
- **Output:** TRUE → `Brevo Email — Referrer Course Outcome1` (no FALSE branch connected)
- **Edge cases:**
  - If node’s `leftValue` is written as `=={{ $json.notifyReferrerCourseStart }}` (as shown), that leading `==` may cause evaluation issues. It should generally be just `{{ $json.notifyReferrerCourseStart }}`.
  - There is no separate handling for `notifyReferrerCourseEnd`, even though Code computes it.

#### Node: Brevo Email — Referrer Course Outcome1
- **Type / role:** Brevo email to referrer about course outcome.
- **Config highlights:**
  - Recipient: `{{ $('Merge — Any Trigger').item.json.fields['Reffering Person\'s Email'] }}`
  - Body contains `Course Outcome: $json.fields['Course Outcome']` (missing `${...}` interpolation, likely not evaluated)
- **Input:** from `If1` TRUE.
- **Output:** → `Referrer notified course start`
- **Edge cases:**
  - References original trigger payload via `$('Merge — Any Trigger').item.json.fields[...]`; this can work, but it’s brittle if multiple items are processed.
  - Body contains malformed interpolation; likely sends literal text.
  - Not aligned with “course start” vs “course end” email content (subject says “Course Outcome Update” but node name suggests outcome).

#### Node: Referrer notified course start
- **Type / role:** Airtable Update sets `Referrer Notified – Course Start: true`.
- **Config highlights:**
  - Record ID: `{{ $('Course Outcome Updated At').item.json.id }}`
- **Input:** from `Brevo Email — Referrer Course Outcome1`
- **Output:** → `WF6 Complete`
- **Edge cases:** field naming must match Airtable exactly; otherwise update fails.

**Associated sticky note (applies to this block):**
- “## Referrer Notifications … Runs independently from candidate messaging.”  
(Implementation note: in the current connections, info notifications are coupled to info messaging commits; course notification is coupled to course messaging commit + If.)

---

### Block 2.6 — Commit Flags & Complete

**Overview:**  
Persists “sent” and “notified” flags back to Airtable and ends the run.

**Nodes involved:**
- WF6 Complete

#### Node: WF6 Complete
- **Type / role:** Set node. Marks workflow completion.
- **Config:** Sets `status = "WF6 Complete"` and keeps only that field.
- **Inputs:** From:
  - `Referrer notified course start`
  - `Airtable — Commit Referrer Notified (Info) [No-Show branch]`
- **Output:** none
- **Edge cases:** none.

**Associated sticky note (applies to this block):**
- “## Commit Flags & Complete … Guarantees idempotency across retries.”  
(Important: due to several mis-set flags noted above, the current implementation may not actually be idempotent.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Airtable Trigger — Info Outcome Updated At1 | Airtable Trigger | Poll Airtable timestamp for info outcome changes | — | Tag eventType = info | ## Trigger & Event Detection; Watches Airtable-managed timestamp fields for outcome changes. Airtable controls when the workflow runs. |
| Tag eventType = info | Set | Label execution as “info” | Airtable Trigger — Info Outcome Updated At1 | Merge — Any Trigger | ## Trigger & Event Detection; Watches Airtable-managed timestamp fields for outcome changes. Airtable controls when the workflow runs. |
| Course Outcome Updated At | Airtable Trigger | Poll Airtable timestamp for course outcome changes | — | Tag eventType = course | ## Trigger & Event Detection; Watches Airtable-managed timestamp fields for outcome changes. Airtable controls when the workflow runs. |
| Tag eventType = course | Set | Label execution as “course” | Course Outcome Updated At | Merge — Any Trigger | ## Trigger & Event Detection; Watches Airtable-managed timestamp fields for outcome changes. Airtable controls when the workflow runs. |
| Merge — Any Trigger | Merge | Funnel both triggers into one path | Tag eventType = info; Tag eventType = course | Code — Normalize + Decide Action | ## Candidate outcomes and referrer notifications (full workflow description). |
| Code — Normalize + Decide Action | Code | Normalize record + compute action + referrer flags | Merge — Any Trigger | Switch — Route Action | ## Normalize & Decide Action; Central logic layer that normalizes fields and outputs one action path. Prevents fragile IF chains. |
| Switch — Route Action | Switch | Route by computed `action` | Code — Normalize + Decide Action | Attended email + Attended sms1; Did Not Attended sms + Did Not Attended email; Course outcome email | ## Normalize & Decide Action; Central logic layer that normalizes fields and outputs one action path. Prevents fragile IF chains. |
| Attended email | Sendinblue (Brevo) | Candidate email for info attended | Switch — Route Action | Airtable — Commit Post-Info Attended Sent | ## Candidate Messaging — Info Event Outcomes; Handles post-info-event emails and SMS for attended and no-show outcomes. Each message is sent once only. |
| Attended sms1 | HTTP Request | Candidate SMS for info attended (Brevo SMS API) | Switch — Route Action | Airtable — Commit Post-Info Attended Sent | ## Candidate Messaging — Info Event Outcomes; Handles post-info-event emails and SMS for attended and no-show outcomes. Each message is sent once only. |
| Airtable — Commit Post-Info Attended Sent | Airtable | Mark attended messages as sent | Attended email; Attended sms1 | Brevo Email — Referrer Info Outcome | ## Commit Flags & Complete; Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries. |
| Did Not Attended sms | HTTP Request | Candidate SMS for info no-show | Switch — Route Action | Airtable — Commit Post-Info NoShow Sent | ## Candidate Messaging — Info Event Outcomes; Handles post-info-event emails and SMS for attended and no-show outcomes. Each message is sent once only. |
| Did Not Attended email | Sendinblue (Brevo) | Candidate email for info no-show | Switch — Route Action | Airtable — Commit Post-Info NoShow Sent | ## Candidate Messaging — Info Event Outcomes; Handles post-info-event emails and SMS for attended and no-show outcomes. Each message is sent once only. |
| Airtable — Commit Post-Info NoShow Sent | Airtable | Mark no-show messages as sent | Did Not Attended sms; Did Not Attended email | Brevo Email — Referrer Info Outcome | ## Commit Flags & Complete; Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries. |
| Brevo Email — Referrer Info Outcome | Sendinblue (Brevo) | Email referrer about info outcome | Airtable — Commit Post-Info Attended Sent; Airtable — Commit Post-Info NoShow Sent | Airtable — Commit Referrer Notified (Info) [No-Show branch] | ## Referrer Notifications; Notifies referring partners when candidates progress or exit the course. Runs independently from candidate messaging. |
| Airtable — Commit Referrer Notified (Info) [No-Show branch] | Airtable | Mark info referrer as notified | Brevo Email — Referrer Info Outcome | WF6 Complete | ## Commit Flags & Complete; Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries. |
| Course outcome email | Sendinblue (Brevo) | Candidate email for course outcomes | Switch — Route Action | Airtable — Commit Course Outcome Message Sent1 | ## Candidate Messaging — Course Outcomes; Sends course outcome messages (started, withdrawn, completed). Content is driven by normalized outcome keys. |
| Airtable — Commit Course Outcome Message Sent1 | Airtable | Mark course outcome message as sent | Course outcome email | If1 | ## Commit Flags & Complete; Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries. |
| If1 | If | Gate referrer notification for course start | Airtable — Commit Course Outcome Message Sent1 | Brevo Email — Referrer Course Outcome1 | ## Referrer Notifications; Notifies referring partners when candidates progress or exit the course. Runs independently from candidate messaging. |
| Brevo Email — Referrer Course Outcome1 | Sendinblue (Brevo) | Email referrer about course outcome | If1 | Referrer notified course start | ## Referrer Notifications; Notifies referring partners when candidates progress or exit the course. Runs independently from candidate messaging. |
| Referrer notified course start | Airtable | Mark referrer notified for course start | Brevo Email — Referrer Course Outcome1 | WF6 Complete | ## Commit Flags & Complete; Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries. |
| WF6 Complete | Set | End marker/status output | Airtable — Commit Referrer Notified (Info) [No-Show branch]; Referrer notified course start | — | ## Commit Flags & Complete; Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries. |
| Sticky Note | Sticky Note | Documentation | — | — | ## Candidate outcomes and referrer notifications … (see note content) |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Trigger & Event Detection … |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Normalize & Decide Action … |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Candidate Messaging — Info Event Outcomes … |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Candidate Messaging — Course Outcomes … |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## Referrer Notifications … |
| Sticky Note6 | Sticky Note | Documentation | — | — | ## Commit Flags & Complete … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n (name it e.g. “Send candidate outcome emails and SMS and notify referrers with Airtable”). Keep it inactive until credentials/IDs are configured.

2) **Add Airtable Trigger (Info outcome)**
   - Node: **Airtable Trigger**
   - Auth: **Airtable Personal Access Token**
   - Set **Base** and **Table**
   - Polling: **Every minute**
   - Trigger field: `InfoOutcomeUpdatedAt_TMP`

3) **Add Set node “Tag eventType = info”**
   - Add a string field: `eventType` = `info`
   - Connect: **Airtable Trigger (Info)** → **Tag eventType = info**

4) **Add Airtable Trigger (Course outcome)**
   - Node: **Airtable Trigger**
   - Same Airtable credential
   - Set Base/Table
   - Polling every minute
   - Trigger field: `Course Outcome Updated At`

5) **Add Set node “Tag eventType = course”**
   - String field: `eventType` = `course`
   - Connect: **Airtable Trigger (Course)** → **Tag eventType = course**

6) **Add Merge node “Merge — Any Trigger”**
   - Node: **Merge**
   - Leave default settings (used as a funnel)
   - Connect:
     - **Tag eventType = info** → **Merge — Any Trigger**
     - **Tag eventType = course** → **Merge — Any Trigger**

7) **Add Code node “Code — Normalize + Decide Action”**
   - Node: **Code**
   - Paste logic that:
     - Reads `item.json.fields`, `item.json.id`, and `item.json.eventType`
     - Produces normalized output with `action`, `courseOutcomeKey`, and referrer notify booleans
   - Connect: **Merge — Any Trigger** → **Code — Normalize + Decide Action**

8) **Add Switch node “Switch — Route Action”**
   - Node: **Switch**
   - Value: `{{ $json.action }}`
   - Rules for string:
     - `info_attended` → output 0
     - `info_noshow` → output 1
     - `course` → output 2
   - Connect: **Code — Normalize + Decide Action** → **Switch — Route Action**

9) **Info attended messaging**
   - Add **Brevo (Sendinblue) Email** node “Attended email”
     - Credential: Brevo/Sendinblue API credential
     - Sender: your verified sender
     - Recipient: `{{ $json.email }}`
     - Subject/body: your attended template
   - Add **HTTP Request** node “Attended sms1”
     - POST `https://api.brevo.com/v3/transactionalSMS/send`
     - Headers: `accept: application/json`, `content-type: application/json`, `api-key: <brevo key>` (prefer storing securely)
     - JSON body uses recipient + content (use n8n expressions)
   - Connect Switch output 0 → both email and sms nodes.

10) **Commit attended sent flag**
   - Add **Airtable** node “Airtable — Commit Post-Info Attended Sent”
   - Operation: **Update**
   - Record ID: use the trigger record ID for info runs (e.g. `{{ $('Airtable Trigger — Info Outcome Updated At1').item.json.id }}`)
   - Set checkbox field: `Post-Info Attended Sent = true`
   - Connect:
     - **Attended email** → commit node
     - **Attended sms1** → commit node  
   (If you want to avoid duplicate commits, insert a Merge/Wait/IF gating so only one path commits.)

11) **Info no-show messaging**
   - Add Brevo email “Did Not Attended email” (recipient `{{ $json.email }}`)
   - Add HTTP Request “Did Not Attended sms” (Brevo SMS API)
   - Connect Switch output 1 → both nodes.

12) **Commit no-show sent flag**
   - Add Airtable update “Airtable — Commit Post-Info NoShow Sent”
   - Update record ID from info trigger
   - Set `Post-Info NoShow Sent = true`
   - Connect both no-show email and sms → this commit node.

13) **Referrer info outcome email + commit**
   - Add Brevo email node “Brevo Email — Referrer Info Outcome”
     - Recipient should reference the **normalized** `referrerEmail` (recommended): `{{ $json.referrerEmail }}`
     - Body includes `{{ $json.firstName }}` and `{{ $json.infoOutcome }}`
   - Add Airtable update node “Airtable — Commit Referrer Notified (Info) …”
     - Set `Referrer Notified – Info Event = true` (or rename consistently with your Code logic)
   - Connect both info commit nodes → referrer info email → referrer-notified commit.

14) **Course outcome email + commit**
   - Add Brevo email node “Course outcome email”
     - Recipient: `{{ $json.email }}`
     - Subject/body: expression based on `{{ $json.courseOutcomeKey }}`
   - Add Airtable update node “Airtable — Commit Course Outcome Message Sent”
     - Record ID from course trigger
     - Set `Course Outcome Message Sent = true`
   - Connect Switch output 2 → course email → course commit.

15) **Course referrer notification (start)**
   - Add If node “If1”
     - Condition: `{{ $json.notifyReferrerCourseStart }}` is true
   - Add Brevo email node “Brevo Email — Referrer Course Outcome”
     - Recipient: `{{ $json.referrerEmail }}`
     - Body includes `{{ $json.courseOutcome }}` or `{{ $json.courseOutcomeKey }}`
   - Add Airtable update node “Referrer notified course start”
     - Set `Referrer Notified – Course Start = true`
   - Connect: course commit → If → (true) referrer email → referrer-notified update.

16) **Add Set node “WF6 Complete”**
   - Set `status = WF6 Complete`
   - Turn on “Keep Only Set”
   - Connect the final Airtable commit nodes (info referrer commit and course referrer commit) → WF6 Complete.

17) **Credentials**
   - Airtable: Personal Access Token with read + write access to the base.
   - Brevo Email node: Brevo API key credential.
   - Brevo SMS: use a secure secret for the API key header (avoid hardcoding in node).

18) **Activate workflow** after confirming:
   - Airtable field names exactly match what Code + commit nodes expect
   - “Sent” fields are checkboxes
   - Trigger timestamp fields are reliably updated by Airtable automations/formulas.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates post-event and post-course communications for candidates, while also notifying referring partners at the correct milestones… Airtable controls when the workflow runs… Each message is sent only once using checkbox “sent” flags stored in Airtable.” | Sticky note: “Candidate outcomes and referrer notifications” |
| “Watches Airtable-managed timestamp fields for outcome changes. Airtable controls when the workflow runs.” | Sticky note: “Trigger & Event Detection” |
| “Central logic layer that normalizes fields and outputs one action path. Prevents fragile IF chains.” | Sticky note: “Normalize & Decide Action” |
| “Handles post-info-event emails and SMS for attended and no-show outcomes. Each message is sent once only.” | Sticky note: “Candidate Messaging — Info Event Outcomes” |
| “Sends course outcome messages (started, withdrawn, completed). Content is driven by normalized outcome keys.” | Sticky note: “Candidate Messaging — Course Outcomes” |
| “Notifies referring partners when candidates progress or exit the course. Runs independently from candidate messaging.” | Sticky note: “Referrer Notifications” |
| “Writes “sent” flags back to Airtable and ends the workflow. Guarantees idempotency across retries.” | Sticky note: “Commit Flags & Complete” |

--- 

### Notable implementation risks to address (from the provided workflow)
- **Course outcome sent flag is set to `false`** in Airtable commit node (should be `true`).
- **Inconsistent referrer field naming** (`Reffering Person's Email` vs `Referrer Email` vs `Referring Person Email`) causing missing recipients.
- **Multiple branches converge causing potential duplicate referrer emails** (email+SMS both commit and both continue).
- **Use of `$json.fields[...]` after normalization**: the Code node outputs a flattened structure; downstream nodes should use `$json.referrerEmail`, `$json.firstName`, etc.
- **Switch has no default route** for `action = none`, so referrer-only notifications (if desired) won’t fire unless explicitly routed.