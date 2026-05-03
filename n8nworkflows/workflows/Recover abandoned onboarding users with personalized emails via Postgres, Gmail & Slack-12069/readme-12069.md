Recover abandoned onboarding users with personalized emails via Postgres, Gmail & Slack

https://n8nworkflows.xyz/workflows/recover-abandoned-onboarding-users-with-personalized-emails-via-postgres--gmail---slack-12069


# Recover abandoned onboarding users with personalized emails via Postgres, Gmail & Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Abandoned Signup Recovery Automation  
**Purpose:** Automatically detect users who abandoned onboarding (inactive + “Incomplete”), send them a personalized recovery email, start an email tracking/follow-up sequence, mark them as “recovery email sent” in Postgres to avoid duplicates, and notify the sales team in Slack.

### 1.1 Scheduled Intake (Trigger)
Runs every 24 hours to start the recovery cycle.

### 1.2 Database Targeting (Postgres query)
Fetches candidate users who match inactivity and status conditions and who have not yet received a recovery email.

### 1.3 Validation + Controlled Processing (IF + batching loop)
Validates that results contain a usable user identifier, then processes users one-by-one (rate-limit friendly).

### 1.4 Email Generation + Send + Tracking (InboxPlus + Gmail)
Builds a personalized message from a template (InboxPlus), sends it via Gmail, retrieves the sent message metadata, then starts an InboxPlus sequence for tracking/follow-ups.

### 1.5 State Update + Sales Notification (Postgres update + Slack)
Updates the user row to prevent re-emailing, and posts a Slack alert to the sales team. Slack then routes back into the batch loop to process the next user.

---

## 2. Block-by-Block Analysis

### Block A — Scheduled Intake & User Selection
**Overview:** Starts the workflow every 24 hours and queries Postgres for abandoned users who meet the inactivity rules and have not been contacted yet.  
**Nodes involved:** Schedule Trigger, Find Abandoned Users

#### Node: Schedule Trigger
- **Type / role:** `scheduleTrigger` — time-based workflow entrypoint.
- **Key configuration:** Runs every **24 hours** (`hoursInterval: 24`).
- **Outputs:** Sends a single trigger item to **Find Abandoned Users**.
- **Edge cases / failures:**
  - If the n8n instance is down at trigger time, the run is missed unless external scheduling/backfill is implemented.
  - Timezone considerations depend on n8n instance settings.

#### Node: Find Abandoned Users
- **Type / role:** `postgres` (Execute Query) — pulls candidate users.
- **Key configuration (interpreted):**
  - Executes SQL:
    - `Status = 'Incomplete'`
    - `last_activity < NOW() - INTERVAL '24 hours'`
    - `recovery_email_sent = FALSE`
  - Returns all columns (`SELECT *`) from `public."Users"`.
- **Credentials:** Postgres credential required.
- **Inputs:** From **Schedule Trigger**.
- **Outputs:** Rows (one item per user row) into **If**.
- **Edge cases / failures:**
  - SQL errors (schema/table/column mismatch, quoting issues with `"Users"`, `"Status"` etc.).
  - `last_activity` null values: SQL condition will exclude them unless explicitly handled (may miss candidates).
  - If `recovery_email_sent` is NULL rather than FALSE, those rows won’t match; consider `COALESCE(recovery_email_sent, FALSE) = FALSE` if needed.
  - Large result sets can lead to long runs unless batching/limits are enforced at SQL level.

**Sticky note context (applies to this block):**  
“Step 1: Start the check and find inactive users…”

---

### Block B — Validation & Batch Loop Control
**Overview:** Ensures each item has a usable ID, then iterates through results in controlled batches (effectively one-by-one here) to reduce rate-limit risk and to connect a “continue loop” cycle after Slack notification.  
**Nodes involved:** If, Loop Over Items

#### Node: If
- **Type / role:** `if` — validates that a user record exists/is usable.
- **Key configuration:**
  - Condition: **ID exists** (`leftValue: {{$json.ID}}`, operator: number → exists).
  - Only the **true** output is connected.
- **Inputs:** From **Find Abandoned Users**.
- **Outputs:**
  - **True** → **Loop Over Items**
  - **False** is not connected (items failing validation are silently dropped).
- **Edge cases / failures:**
  - If the Postgres column is `id` not `ID`, expression fails logically (no match) and users are skipped.
  - Strict type validation is enabled; non-numeric IDs could cause mismatches if schema differs.

#### Node: Loop Over Items
- **Type / role:** `splitInBatches` — processes users incrementally.
- **Key configuration:**
  - Batch size is not explicitly set (defaults apply). The wiring indicates it’s used as a loop controller.
- **Connections (important wiring detail):**
  - **Output 1** (the “current batch/items” output in n8n’s SplitInBatches pattern) → **PrepareEmail email**
  - **Input 0** is fed by **If** (initial items) and later by **Alert Sales Team** to continue the loop.
- **Loop behavior in this workflow:**
  1. Items arrive from **If**.
  2. SplitInBatches emits the next item(s) to **PrepareEmail email**.
  3. After processing (email → update → slack), **Alert Sales Team** connects back into **Loop Over Items** input to fetch the next item.
- **Edge cases / failures:**
  - If any downstream node errors, the loop stops and remaining users are not processed that run.
  - Without explicit batch size and error handling, very large sets may still run long (though the loop pattern helps).

**Sticky note context (applies to this block and the next):**  
“Step 2: Prepare users and generate recovery emails…”

---

### Block C — Personalization, Send, and Email Tracking
**Overview:** Generates a personalized email using InboxPlus, sends it via Gmail, fetches the sent message to obtain thread/message metadata, then starts an InboxPlus sequence for tracking and follow-ups.  
**Nodes involved:** PrepareEmail email, Send a message, Get a message, StartSequence email

#### Node: PrepareEmail email
- **Type / role:** `@itechnotion/n8n-nodes-inboxplus.inboxPlus` — InboxPlus template renderer.
- **Key configuration:**
  - Uses `templateId: YOUR_EMAIL_TEMPLATE_ID`.
  - `recipientEmail: {{$json.Email}}` from the current user row.
- **Credentials:** InboxPlus API credential required.
- **Inputs:** From **Loop Over Items** (current user item).
- **Outputs:** An item expected to include fields used later:
  - `recipientEmail` (used by Gmail send)
  - `subject` and `body` (used by Gmail send)
  - `trackingId` (later referenced by StartSequence)
- **Edge cases / failures:**
  - Missing/invalid template ID or API key.
  - If the template expects variables not provided, InboxPlus may error or render incomplete content.
  - If the user row lacks `Email`, downstream Gmail send will fail.

#### Node: Send a message
- **Type / role:** `gmail` (send) — sends the prepared email.
- **Key configuration:**
  - To: `{{$json.recipientEmail}}`
  - Subject: `{{$json.subject}}`
  - Message body: `{{$json.body}}`
- **Credentials:** Gmail OAuth2 required.
- **Inputs:** From **PrepareEmail email**.
- **Outputs:** Sent message metadata (must include an `id` for the next node).
- **Edge cases / failures:**
  - OAuth token expiration / insufficient Gmail scopes.
  - Gmail API rate limits or sending limits.
  - Invalid recipient email address formatting.

#### Node: Get a message
- **Type / role:** `gmail` (get) — retrieves the just-sent message to capture headers and thread info.
- **Key configuration:**
  - Operation: Get message
  - `messageId: {{$json.id}}` (from Gmail send output)
- **Credentials:** Same Gmail OAuth2.
- **Inputs:** From **Send a message**.
- **Outputs:** Full message details, including `threadId`, `id`, and header-derived fields (often `From`, `To`, `Subject` depending on node output mapping).
- **Edge cases / failures:**
  - If the send output doesn’t include `id` in the expected path, this fails.
  - Gmail eventual consistency is rare but possible (immediate fetch could fail transiently).

#### Node: StartSequence email
- **Type / role:** `@itechnotion/n8n-nodes-inboxplus.inboxPlus` (startSequence) — associates the sent message with a follow-up/tracking sequence.
- **Key configuration:**
  - Operation: `startSequence`
  - `sequenceId: YOUR_SEQUENCE_ID`
  - Uses Gmail message data:
    - `threadId: {{$json.threadId}}`
    - `messageId: {{$json.id}}`
    - `subject: {{$json.Subject}}`
    - `sequenceSenderEmail: {{$json.From}}`
    - `sequenceRecipientEmail: {{$json.To}}`
  - **Cross-node reference:**  
    - `trackingId: {{ $('PrepareEmail email').item.json.trackingId }}`
- **Credentials:** InboxPlus API credential required.
- **Inputs:** From **Get a message**.
- **Outputs:** Sequence start result passed to **Update rows in a table**.
- **Edge cases / failures:**
  - `Subject/From/To` field names may differ depending on Gmail node output; if missing, sequence start may fail.
  - The cross-node expression referencing `PrepareEmail email` assumes item alignment; if batching >1 and concurrency changes, ensure correct pairing.
  - Invalid sequence ID or InboxPlus API errors.

---

### Block D — Postgres State Update & Slack Notification (and loop continuation)
**Overview:** Marks the user as contacted (`recovery_email_sent = true`) and notifies sales in Slack. Slack then feeds back into the batch node to continue processing the next user.  
**Nodes involved:** Update rows in a table, Alert Sales Team

#### Node: Update rows in a table
- **Type / role:** `postgres` (update) — updates the Users record.
- **Key configuration:**
  - Schema: `public`
  - Table: `Users`
  - Matching column: `ID`
  - Values set:
    - `ID: {{ $('Loop Over Items').item.json.ID }}` (match key)
    - `recovery_email_sent: true`
- **Credentials:** Postgres credential required.
- **Inputs:** From **StartSequence email**.
- **Outputs:** Update result into **Alert Sales Team**.
- **Edge cases / failures:**
  - If `ID` is not available (or mismatched type), update affects 0 rows.
  - If Postgres permissions disallow update, node fails.
  - If multiple items are in-flight, referencing `$('Loop Over Items').item.json.ID` depends on correct item context.

#### Node: Alert Sales Team
- **Type / role:** `slack` — posts a message to a channel for sales visibility.
- **Key configuration:**
  - Authentication: Slack OAuth2
  - Channel: `YOUR_SLACK_CHANNEL_ID`
  - Message text:
    ```
    Recovery Email Sent to:
    ID: {{ $json.ID }}
    Name: {{ $json.Name }}
    Email: {{ $json.Email }}
    --
    ```
- **Credentials:** Slack OAuth2 credential required.
- **Inputs:** From **Update rows in a table**.
- **Outputs / loop wiring:** Output goes back to **Loop Over Items** (to continue processing next user).
- **Edge cases / failures:**
  - If Slack channel ID is wrong or the bot/user lacks permission, notification fails and the loop will not continue.
  - The Slack message uses `$json.ID/Name/Email`. Depending on what the Postgres update node returns, these fields may not be present unless preserved from earlier steps. If missing, Slack messages will show blank values (or the node may still send with empty placeholders).

**Sticky note context (applies to this block):**  
“Step 3: Update records and notify the sales team…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts workflow every 24h | — | Find Abandoned Users | ## Step 1: Start the check and find inactive users |
| Find Abandoned Users | Postgres (Execute Query) | Fetch abandoned/inactive users | Schedule Trigger | If | ## Step 1: Start the check and find inactive users |
| If | IF | Validate item has an ID | Find Abandoned Users | Loop Over Items | ## Step 2: Prepare users and generate recovery emails |
| Loop Over Items | Split In Batches | Iterate users one-by-one / loop control | If, Alert Sales Team | PrepareEmail email | ## Step 2: Prepare users and generate recovery emails |
| PrepareEmail email | InboxPlus | Render personalized email from template | Loop Over Items | Send a message | ## Step 2: Prepare users and generate recovery emails |
| Send a message | Gmail (Send) | Send recovery email | PrepareEmail email | Get a message | ## Step 2: Prepare users and generate recovery emails |
| Get a message | Gmail (Get) | Fetch sent email metadata (thread/message) | Send a message | StartSequence email | ## Step 2: Prepare users and generate recovery emails |
| StartSequence email | InboxPlus (startSequence) | Start tracking/follow-up sequence | Get a message | Update rows in a table | ## Step 2: Prepare users and generate recovery emails |
| Update rows in a table | Postgres (Update) | Mark user as contacted | StartSequence email | Alert Sales Team | ## Step 3: Update records and notify the sales team |
| Alert Sales Team | Slack | Notify sales + continue loop | Update rows in a table | Loop Over Items | ## Step 3: Update records and notify the sales team |
| Sticky Note | Sticky Note | Documentation / context | — | — | ## Automated User Recovery Email Workflow… |
| Sticky Note1 | Sticky Note | Block label (Step 1) | — | — | ## Step 1: Start the check and find inactive users |
| Sticky Note2 | Sticky Note | Block label (Step 2) | — | — | ## Step 2: Prepare users and generate recovery emails |
| Sticky Note3 | Sticky Note | Block label (Step 3) | — | — | ## Step 3: Update records and notify the sales team |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name it: **Abandoned Signup Recovery Automation**
   - Keep workflow **inactive** until testing is complete.

2. **Add node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Configure: **Every 24 hours** (interval: hours = 24)
   - Connect: **Schedule Trigger → Find Abandoned Users**

3. **Add node: Find Abandoned Users (Postgres)**
   - Node type: **Postgres**
   - Operation: **Execute Query**
   - Credentials: create/select **Postgres credential** with host/db/user/password (or SSL as required)
   - SQL query (adjust table/column names as needed):
     - Filter `Status = 'Incomplete'`
     - Filter `last_activity < NOW() - INTERVAL '24 hours'`
     - Filter `recovery_email_sent = FALSE`
   - Connect: **Find Abandoned Users → If**

4. **Add node: If**
   - Node type: **IF**
   - Condition: check that `{{$json.ID}}` **exists** (number → exists)
   - Connect: **If (true) → Loop Over Items**

5. **Add node: Loop Over Items**
   - Node type: **Split In Batches**
   - Batch size: keep default (or set to `1` explicitly if you want strict one-by-one behavior)
   - Connect: **Loop Over Items (items output) → PrepareEmail email**
   - Later you will connect **Alert Sales Team → Loop Over Items** to continue.

6. **Add node: PrepareEmail email (InboxPlus)**
   - Node type: **InboxPlus** (community node `@itechnotion/n8n-nodes-inboxplus.inboxPlus`)
   - Credentials: create/select **InboxPlus API** credential
   - Configure:
     - `templateId`: your InboxPlus template ID
     - `recipientEmail`: `{{$json.Email}}`
   - Connect: **PrepareEmail email → Send a message**

7. **Add node: Send a message (Gmail)**
   - Node type: **Gmail**
   - Operation: **Send**
   - Credentials: create/select **Gmail OAuth2** credential (scopes for sending email)
   - Map fields:
     - To: `{{$json.recipientEmail}}`
     - Subject: `{{$json.subject}}`
     - Message: `{{$json.body}}`
   - Connect: **Send a message → Get a message**

8. **Add node: Get a message (Gmail)**
   - Node type: **Gmail**
   - Operation: **Get**
   - Message ID: `{{$json.id}}`
   - Connect: **Get a message → StartSequence email**

9. **Add node: StartSequence email (InboxPlus)**
   - Node type: **InboxPlus**
   - Operation: **startSequence**
   - Configure:
     - `sequenceId`: your InboxPlus sequence ID
     - `threadId`: `{{$json.threadId}}`
     - `messageId`: `{{$json.id}}`
     - `subject`: `{{$json.Subject}}` (adjust to match Gmail output if needed)
     - `trackingId`: `{{ $('PrepareEmail email').item.json.trackingId }}`
     - `sequenceSenderEmail`: `{{$json.From}}`
     - `sequenceRecipientEmail`: `{{$json.To}}`
   - Connect: **StartSequence email → Update rows in a table**

10. **Add node: Update rows in a table (Postgres)**
   - Node type: **Postgres**
   - Operation: **Update**
   - Schema: `public`
   - Table: `Users`
   - Matching columns: `ID`
   - Set values:
     - `ID`: `{{ $('Loop Over Items').item.json.ID }}`
     - `recovery_email_sent`: `true`
   - Connect: **Update rows in a table → Alert Sales Team**

11. **Add node: Alert Sales Team (Slack)**
   - Node type: **Slack**
   - Authentication: **OAuth2**
   - Channel: set your **Slack channel ID**
   - Text: include ID/Name/Email expressions (ensure those fields exist at this point in the flow)
   - Connect: **Alert Sales Team → Loop Over Items** (this is what continues the batch loop)

12. **Add sticky notes (optional but recommended)**
   - Add three notes labeling Steps 1–3 and one overview note with operational checklist (as in the provided workflow).

13. **Test execution**
   - In Postgres, create one test user that matches the SQL conditions.
   - Run the workflow manually.
   - Verify:
     - Gmail email arrives.
     - InboxPlus sequence starts successfully.
     - Postgres sets `recovery_email_sent = true`.
     - Slack message posts.
     - No second email is sent on the next run for the same user.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated User Recovery Email Workflow: scheduled run, Postgres query for incomplete/inactive users, validate results, process one-by-one, prepare email with InboxPlus template, send via Gmail, track via sequence, update Postgres to prevent duplicates, notify sales in Slack, test with sample user before activating. | From the workflow’s main sticky note content |
| Step 1: Start the check and find inactive users. | Sticky note labeling the initial trigger + Postgres query block |
| Step 2: Prepare users and generate recovery emails. | Sticky note labeling validation, batching, email generation/sending/tracking |
| Step 3: Update records and notify the sales team. | Sticky note labeling Postgres update + Slack notification block |