Create a dead man’s switch for solo founders with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/create-a-dead-man-s-switch-for-solo-founders-with-google-sheets-and-gmail-14236


# Create a dead man’s switch for solo founders with Google Sheets and Gmail

# 1. Workflow Overview

This workflow implements a **dead man’s switch** for solo founders. It monitors whether the founder has checked in recently, sends a reminder if the check-in is overdue, escalates to emergency contacts if the delay becomes critical, and logs escalations in Google Sheets. It also exposes a webhook endpoint the founder can bookmark and visit to reset the timer.

## 1.1 Scheduled Monitoring

A daily schedule trigger starts the monitoring flow at 9:00 AM in the workflow timezone (`America/New_York`). The workflow reads the check-in history from Google Sheets and computes how much time has passed since the most recent check-in.

## 1.2 Status Evaluation and Routing

The computed elapsed time is compared to a configurable threshold from the sheet. The workflow classifies the result as:

- `OK` if still within threshold
- `WARNING` if past threshold
- `CRITICAL` if past twice the threshold

It then routes execution accordingly.

## 1.3 Reminder and Emergency Escalation

If the status is `WARNING`, the founder receives a reminder email. If the status is `CRITICAL`, both emergency contacts are emailed sequentially, and the event is logged in an alert sheet.

## 1.4 Manual Check-in Endpoint

A webhook acts as the founder’s daily check-in URL. When visited, it appends a new timestamped row to the `CheckIns` sheet and returns a confirmation response indicating that the timer has been reset.

---

# 2. Block-by-Block Analysis

## Block 1 — Workflow Context and Setup Notes

**Overview:**  
This block is documentation embedded directly in the workflow canvas. It explains the workflow purpose, required sheet structure, and setup expectations for credentials and usage.

**Nodes Involved:**  
- `Sticky Note`

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; documentation-only canvas note.
- **Configuration choices:** Contains the high-level explanation of the dead man’s switch concept, setup steps, sheet tab requirements, seed row requirements, credential setup, and activation/bookmarking instructions.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime; informational only.
- **Sub-workflow reference:** None.

---

## Block 2 — Daily Monitoring

**Overview:**  
This block runs once per day, reads the check-in data from Google Sheets, and computes whether the founder is still within the permitted interval. It is the main monitoring engine of the workflow.

**Nodes Involved:**  
- `Daily Check (9 AM)`
- `Read Check-in Log`
- `Calculate Hours Since Last Check-in`
- `Sticky Note1`

### Node Details

#### Daily Check (9 AM)
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry point for scheduled executions.
- **Configuration choices:** Configured to trigger daily at hour `9`. The workflow timezone is `America/New_York`, so 9 AM is interpreted in that timezone.
- **Key expressions or variables used:** None.
- **Input and output connections:** No incoming connection; outputs to `Read Check-in Log`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Workflow must be active for scheduled execution.
  - Timezone misunderstandings can cause execution at an unexpected local time.
  - If the n8n instance is paused or unavailable at trigger time, execution may be delayed or skipped depending on deployment behavior.
- **Sub-workflow reference:** None.

#### Read Check-in Log
- **Type and role:** `n8n-nodes-base.googleSheets`; reads rows from the `CheckIns` tab.
- **Configuration choices:**
  - Uses Google Sheets OAuth2 credentials.
  - Targets spreadsheet `DeadMan` by URL / ID.
  - Reads sheet `gid=0`, cached as `CheckIns`.
  - No extra options are set.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Daily Check (9 AM)`; output to `Calculate Hours Since Last Check-in`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - OAuth credential expiration or revoked access.
  - Spreadsheet URL or tab mismatch.
  - Empty sheet content.
  - Missing expected columns such as `timestamp`, `threshold_hours`, `founder_name`, or contact fields.
  - Date values in invalid format causing later parsing issues.
- **Sub-workflow reference:** None.

#### Calculate Hours Since Last Check-in
- **Type and role:** `n8n-nodes-base.code`; computes elapsed time and determines status.
- **Configuration choices:**
  - Pulls all rows using `$input.all()`.
  - If the sheet is empty, returns a synthetic critical record:
    - `hoursSinceCheckIn: 9999`
    - `lastCheckIn: 'never'`
    - `status: 'CRITICAL'`
  - Uses:
    - the **first row** as the configuration row (`configRow`)
    - the **last row** as the latest check-in (`lastRow`)
  - Parses `lastRow.json.timestamp` as a JavaScript `Date`.
  - Computes hours difference from current time.
  - Uses `threshold_hours` from the first row, defaulting to `24` if absent.
  - Status logic:
    - `OK` if `hoursDiff <= threshold`
    - `WARNING` if `threshold < hoursDiff <= threshold * 2`
    - `CRITICAL` if `hoursDiff > threshold * 2`
  - Builds output fields including founder identity, contacts, and emergency message.
- **Key expressions or variables used:**
  - `$input.all()`
  - `configRow.json.threshold_hours`
  - `lastRow.json.timestamp`
  - fallback fields such as:
    - `founder_name || 'Founder'`
    - `founder_email || ''`
    - emergency contacts default to empty strings
    - default emergency message assembled from elapsed hours
- **Input and output connections:** Input from `Read Check-in Log`; output to `Status OK?`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Invalid or non-ISO timestamp values may yield `Invalid Date`.
  - If rows are not chronologically sorted, the “last row” may not actually be the most recent check-in.
  - Using the first row as configuration assumes the sheet keeps a stable seed row at the top.
  - If `threshold_hours` contains non-numeric text, `parseInt()` may return `NaN`; downstream logic may behave unpredictably.
  - Missing founder email or emergency contact emails may cause downstream Gmail node failures.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`; descriptive note for the daily monitoring area.
- **Configuration choices:** Documents the daily run, the sheet read, and the three statuses (`OK`, `WARNING`, `CRITICAL`).
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

---

## Block 3 — Status Evaluation and Routing

**Overview:**  
This block decides whether nothing should happen, whether the founder should receive a reminder, or whether emergency escalation is required.

**Nodes Involved:**  
- `Status OK?`
- `All Good - No Action`
- `Is Critical?`
- `Sticky Note2`

### Node Details

#### Status OK?
- **Type and role:** `n8n-nodes-base.if`; checks whether computed status equals `OK`.
- **Configuration choices:**
  - Uses version 2 condition format.
  - Condition:
    - left value: `{{ $json.status }}`
    - operator: string equals
    - right value: `OK`
  - True branch goes to `All Good - No Action`.
  - False branch goes to `Is Critical?`.
- **Key expressions or variables used:** `={{ $json.status }}`
- **Input and output connections:** Input from `Calculate Hours Since Last Check-in`; outputs:
  - true → `All Good - No Action`
  - false → `Is Critical?`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If `status` is missing or not a string, strict validation can affect matching.
  - Any unexpected status value other than `OK` is routed to the non-OK path.
- **Sub-workflow reference:** None.

#### All Good - No Action
- **Type and role:** `n8n-nodes-base.set`; terminal informational node for healthy status.
- **Configuration choices:** Sets a single field:
  - `message = "Check-in is current. No action needed."`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from the true branch of `Status OK?`; no outgoing connection.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Minimal; this node is unlikely to fail unless the execution environment itself fails.
- **Sub-workflow reference:** None.

#### Is Critical?
- **Type and role:** `n8n-nodes-base.if`; distinguishes `CRITICAL` from `WARNING`.
- **Configuration choices:**
  - Condition:
    - left value: `{{ $json.status }}`
    - operator: string equals
    - right value: `CRITICAL`
  - True branch → emergency contact path.
  - False branch → founder reminder path.
- **Key expressions or variables used:** `={{ $json.status }}`
- **Input and output connections:** Input from false branch of `Status OK?`; outputs:
  - true → `Alert Emergency Contact 1`
  - false → `Send Reminder to Founder`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Any non-`CRITICAL` non-`OK` status is treated as the reminder path.
  - Unexpected status labels would still flow to reminder, which may or may not be intended.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`; explains branching logic.
- **Configuration choices:** Documents `OK`, `WARNING`, and `CRITICAL` routing behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## Block 4 — Reminder and Emergency Escalation

**Overview:**  
This block handles the email actions after status routing. Warnings send a reminder to the founder. Critical states notify both emergency contacts sequentially and then append a log entry to the alert sheet.

**Nodes Involved:**  
- `Send Reminder to Founder`
- `Alert Emergency Contact 1`
- `Alert Emergency Contact 2`
- `Log Alert to Sheet`
- `Sticky Note3`

### Node Details

#### Send Reminder to Founder
- **Type and role:** `n8n-nodes-base.gmail`; sends a warning email to the founder.
- **Configuration choices:**
  - Uses Gmail OAuth2 credentials.
  - Recipient: `{{ $('Calculate Hours Since Last Check-in').item.json.founderEmail }}`
  - Subject: `Dead Mans Switch: You Havent Checked In`
  - Message body includes:
    - founder name
    - hours since last check-in
    - threshold
    - reminder to click the check-in link
    - escalation warning
- **Key expressions or variables used:**
  - `$('Calculate Hours Since Last Check-in').item.json.founderEmail`
  - `...founderName`
  - `...hoursSinceCheckIn`
  - `...thresholdHours`
- **Input and output connections:** Input from false branch of `Is Critical?`; no outgoing connection.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid founder email.
  - Gmail OAuth2 authorization problems.
  - Gmail sending limits or anti-abuse restrictions.
  - Expressions referencing upstream node output will fail if that node structure changes.
- **Sub-workflow reference:** None.

#### Alert Emergency Contact 1
- **Type and role:** `n8n-nodes-base.gmail`; first emergency escalation email.
- **Configuration choices:**
  - Uses Gmail OAuth2 credentials.
  - Recipient: `{{ $('Calculate Hours Since Last Check-in').item.json.emergencyContact1 }}`
  - Subject dynamically includes founder name.
  - Message uses `emergencyMessage` plus elapsed hours and an automated alert notice.
- **Key expressions or variables used:**
  - `$('Calculate Hours Since Last Check-in').item.json.emergencyContact1`
  - `...emergencyMessage`
  - `...hoursSinceCheckIn`
  - `...founderName`
- **Input and output connections:** Input from true branch of `Is Critical?`; output to `Alert Emergency Contact 2`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Missing emergency contact email.
  - Gmail auth issues or quota limitations.
  - If this node fails, the second emergency contact and alert logging will not run because the path is sequential.
- **Sub-workflow reference:** None.

#### Alert Emergency Contact 2
- **Type and role:** `n8n-nodes-base.gmail`; second emergency escalation email.
- **Configuration choices:**
  - Same Gmail OAuth2 credential as the other email nodes.
  - Recipient: `{{ $('Calculate Hours Since Last Check-in').item.json.emergencyContact2 }}`
  - Subject and body mirror the first emergency alert.
- **Key expressions or variables used:**
  - `$('Calculate Hours Since Last Check-in').item.json.emergencyContact2`
  - `...emergencyMessage`
  - `...hoursSinceCheckIn`
  - `...founderName`
- **Input and output connections:** Input from `Alert Emergency Contact 1`; output to `Log Alert to Sheet`.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Missing second emergency contact email.
  - Gmail auth/quota issues.
  - If this node fails, alert logging will not occur.
- **Sub-workflow reference:** None.

#### Log Alert to Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a record of a critical alert.
- **Configuration choices:**
  - Operation: `append`
  - Sheet: `AlertLog`
  - Spreadsheet identified by URL
  - Columns mapped explicitly:
    - `status = {{ $('Calculate Hours Since Last Check-in').item.json.status }}`
    - `timestamp = {{ $now.toISO() }}`
    - `hours_since_checkin = {{ $('Calculate Hours Since Last Check-in').item.json.hoursSinceCheckIn }}`
- **Key expressions or variables used:**
  - `$('Calculate Hours Since Last Check-in').item.json.status`
  - `$now.toISO()`
  - `$('Calculate Hours Since Last Check-in').item.json.hoursSinceCheckIn`
- **Input and output connections:** Input from `Alert Emergency Contact 2`; no outgoing connection.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Google Sheets auth issues.
  - Missing `AlertLog` tab or wrong columns.
  - Type mismatch if sheet formatting is restrictive.
  - Because logging is placed after both email nodes, a prior email failure prevents alert log creation.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`; explains warning vs critical action paths.
- **Configuration choices:** Documents that critical sends two emergency emails and logs, while warning only emails the founder.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## Block 5 — Check-in Webhook Endpoint

**Overview:**  
This block provides the founder-facing endpoint used to reset the dead man’s switch. It records a timestamped check-in to the sheet and returns a simple confirmation payload.

**Nodes Involved:**  
- `Check-in Webhook`
- `Record Check-in`
- `Confirm Check-in`
- `Sticky Note4`

### Node Details

#### Check-in Webhook
- **Type and role:** `n8n-nodes-base.webhook`; runtime entry point for manual check-ins.
- **Configuration choices:**
  - Path: `dead-mans-switch-checkin`
  - Response mode: `lastNode`
  - No extra options
- **Key expressions or variables used:** None.
- **Input and output connections:** No incoming connection; outputs to `Record Check-in`.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Workflow must be active for production webhook URL to work.
  - If path conflicts with another workflow route, deployment may fail or behave unexpectedly.
  - Public accessibility depends on n8n hosting/network exposure.
- **Sub-workflow reference:** None.

#### Record Check-in
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a check-in row into `CheckIns`.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `DeadMan`
  - Sheet: `CheckIns` (`gid=0`)
  - Defines two output columns:
    - `source = "webhook"`
    - `timestamp = {{ $now.toISO() }}`
  - Explicit mapping mode is used.
- **Key expressions or variables used:**
  - `$now.toISO()`
- **Input and output connections:** Input from `Check-in Webhook`; output to `Confirm Check-in`.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**
  - Sheets auth problems.
  - Tab/column mismatch.
  - If the `CheckIns` sheet structure changes, append may fail or write incomplete data.
  - Because this node writes only `source` and `timestamp`, configuration data must remain available in an earlier seed row.
- **Sub-workflow reference:** None.

#### Confirm Check-in
- **Type and role:** `n8n-nodes-base.set`; returns the webhook response payload.
- **Configuration choices:** Sets:
  - `message = "Check-in recorded. Your Dead Mans Switch timer has been reset."`
  - `checkedInAt = {{ $now.toISO() }}`
- **Key expressions or variables used:** `={{ $now.toISO() }}`
- **Input and output connections:** Input from `Record Check-in`; no outgoing connection. As the last node in webhook response mode, its output becomes the HTTP response body.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If the previous Google Sheets append fails, this response is never returned successfully.
  - Time returned depends on server time and workflow timezone handling in expressions.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`; explains the purpose of the webhook check-in URL.
- **Configuration choices:** Documents bookmarking the endpoint and its reset behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Check (9 AM) | Schedule Trigger | Starts the daily monitoring run at 9 AM |  | Read Check-in Log | ## Dead Man's Switch for Solo Founders<br>A personal safety net for solo founders, freelancers, and one-person businesses. If you stop checking in, your emergency contacts get alerted automatically.<br><br>### How it works<br>1. **Bookmark your check-in URL** and visit it daily to reset the timer<br>2. Every day at 9 AM, the workflow checks how long since your last check-in<br>3. **Past threshold** (default 24h) — you get a reminder email<br>4. **Past 2x threshold** (48h) — your emergency contacts are alerted<br>5. Every alert is logged to a Google Sheet for a full audit trail<br><br>### Setup<br>1. Create a Google Sheet with two tabs: **CheckIns** and **AlertLog**<br>2. In **CheckIns**, add columns: `timestamp`, `source`, `founder_name`, `founder_email`, `emergency_contact_1`, `emergency_contact_2`, `emergency_message`, `threshold_hours`<br>3. In **AlertLog**, add columns: `timestamp`, `status`, `hours_since_checkin`<br>4. Add one seed row in CheckIns with your details and threshold (e.g. 24)<br>5. Paste your Google Sheet URL into the three Google Sheets nodes<br>6. Connect your **Gmail OAuth2** and **Google Sheets** credentials<br>7. Activate the workflow and bookmark the webhook URL |
| Read Check-in Log | Google Sheets | Reads all check-in/config rows from the `CheckIns` sheet | Daily Check (9 AM) | Calculate Hours Since Last Check-in | ### Daily Monitoring<br>Runs every day at 9 AM. Reads all check-in rows from Google Sheets, then calculates how many hours have passed since the last check-in. Compares against your configured threshold to determine status: **OK**, **WARNING**, or **CRITICAL**. |
| Calculate Hours Since Last Check-in | Code | Computes elapsed hours, threshold, status, and contact metadata | Read Check-in Log | Status OK? | ### Daily Monitoring<br>Runs every day at 9 AM. Reads all check-in rows from Google Sheets, then calculates how many hours have passed since the last check-in. Compares against your configured threshold to determine status: **OK**, **WARNING**, or **CRITICAL**. |
| Status OK? | If | Splits healthy vs overdue cases | Calculate Hours Since Last Check-in | All Good - No Action; Is Critical? | ### Status Routing<br>If status is **OK**, no action is taken. Otherwise, it checks whether the situation is **CRITICAL** (past 2x threshold) or just a **WARNING** (past 1x threshold) and routes accordingly. |
| All Good - No Action | Set | Produces a no-action completion message | Status OK? |  | ### Status Routing<br>If status is **OK**, no action is taken. Otherwise, it checks whether the situation is **CRITICAL** (past 2x threshold) or just a **WARNING** (past 1x threshold) and routes accordingly. |
| Is Critical? | If | Splits warning reminders from critical escalation | Status OK? | Alert Emergency Contact 1; Send Reminder to Founder | ### Status Routing<br>If status is **OK**, no action is taken. Otherwise, it checks whether the situation is **CRITICAL** (past 2x threshold) or just a **WARNING** (past 1x threshold) and routes accordingly. |
| Send Reminder to Founder | Gmail | Sends a warning reminder email to the founder | Is Critical? |  | ### Alert Escalation<br>**CRITICAL path** — Emails both emergency contacts sequentially, then logs the alert to the AlertLog sheet tab for a full audit trail.<br><br>**WARNING path** — Sends a reminder email only to the founder, prompting them to check in before it escalates to emergency contacts. |
| Alert Emergency Contact 1 | Gmail | Sends the first critical alert email | Is Critical? | Alert Emergency Contact 2 | ### Alert Escalation<br>**CRITICAL path** — Emails both emergency contacts sequentially, then logs the alert to the AlertLog sheet tab for a full audit trail.<br><br>**WARNING path** — Sends a reminder email only to the founder, prompting them to check in before it escalates to emergency contacts. |
| Alert Emergency Contact 2 | Gmail | Sends the second critical alert email | Alert Emergency Contact 1 | Log Alert to Sheet | ### Alert Escalation<br>**CRITICAL path** — Emails both emergency contacts sequentially, then logs the alert to the AlertLog sheet tab for a full audit trail.<br><br>**WARNING path** — Sends a reminder email only to the founder, prompting them to check in before it escalates to emergency contacts. |
| Log Alert to Sheet | Google Sheets | Appends a critical alert audit record to `AlertLog` | Alert Emergency Contact 2 |  | ### Alert Escalation<br>**CRITICAL path** — Emails both emergency contacts sequentially, then logs the alert to the AlertLog sheet tab for a full audit trail.<br><br>**WARNING path** — Sends a reminder email only to the founder, prompting them to check in before it escalates to emergency contacts. |
| Check-in Webhook | Webhook | Public/manual endpoint for founder daily check-ins |  | Record Check-in | ### Check-in Endpoint<br>A simple webhook URL you bookmark and visit to check in. Records the timestamp to Google Sheets and returns a confirmation message. This is how you reset your dead man's switch timer each day. |
| Record Check-in | Google Sheets | Writes a new webhook check-in row to `CheckIns` | Check-in Webhook | Confirm Check-in | ### Check-in Endpoint<br>A simple webhook URL you bookmark and visit to check in. Records the timestamp to Google Sheets and returns a confirmation message. This is how you reset your dead man's switch timer each day. |
| Confirm Check-in | Set | Returns a success message and timestamp to the webhook caller | Record Check-in |  | ### Check-in Endpoint<br>A simple webhook URL you bookmark and visit to check in. Records the timestamp to Google Sheets and returns a confirmation message. This is how you reset your dead man's switch timer each day. |
| Sticky Note | Sticky Note | Canvas documentation and setup instructions |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for monitoring block |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for routing block |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for escalation block |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for webhook block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Dead Man's Switch for Solo Founders`.
   - In workflow settings, set the timezone to `America/New_York`.
   - Ensure the workflow can be activated so both the schedule and webhook can run.

2. **Prepare the Google Sheet**
   - Create a spreadsheet, for example named `DeadMan`.
   - Create two tabs:
     - `CheckIns`
     - `AlertLog`

3. **Set up the `CheckIns` sheet structure**
   - Add these columns in row 1:
     - `timestamp`
     - `source`
     - `founder_name`
     - `founder_email`
     - `emergency_contact_1`
     - `emergency_contact_2`
     - `emergency_message`
     - `threshold_hours`
   - Add one initial seed row containing:
     - founder name
     - founder email
     - emergency contacts
     - custom emergency message if desired
     - threshold, such as `24`
   - This initial row is important because the workflow treats the **first row** as the config source.

4. **Set up the `AlertLog` sheet structure**
   - Add these columns:
     - `timestamp`
     - `status`
     - `hours_since_checkin`

5. **Create Google Sheets credentials in n8n**
   - Add a `Google Sheets OAuth2 API` credential.
   - Authorize access to the spreadsheet.
   - Reuse this same credential for both read and append operations.

6. **Create Gmail credentials in n8n**
   - Add a `Gmail OAuth2` credential.
   - Authorize the Gmail account that will send reminder and emergency messages.
   - Reuse this credential for all Gmail nodes.

7. **Add the schedule trigger**
   - Create a `Schedule Trigger` node.
   - Name it `Daily Check (9 AM)`.
   - Configure it to run every day at hour `9`.

8. **Add the Google Sheets read node**
   - Create a `Google Sheets` node.
   - Name it `Read Check-in Log`.
   - Connect `Daily Check (9 AM)` → `Read Check-in Log`.
   - Select the spreadsheet.
   - Select the `CheckIns` sheet.
   - Use the same general read behavior as the imported workflow: read the sheet rows without extra filtering.

9. **Add the code node for elapsed-time calculation**
   - Create a `Code` node.
   - Name it `Calculate Hours Since Last Check-in`.
   - Connect `Read Check-in Log` → `Calculate Hours Since Last Check-in`.
   - Paste logic equivalent to:
     - load all input rows
     - if empty, return a synthetic critical payload
     - use first row as configuration
     - use last row as latest check-in
     - calculate hours since latest timestamp
     - read `threshold_hours`, default `24`
     - set status to `OK`, `WARNING`, or `CRITICAL`
     - return founder/contact fields and emergency message

   Use this behavior:
   - `OK` when hours <= threshold
   - `WARNING` when hours > threshold
   - `CRITICAL` when hours > threshold * 2

10. **Add the first routing IF node**
    - Create an `If` node.
    - Name it `Status OK?`.
    - Connect `Calculate Hours Since Last Check-in` → `Status OK?`.
    - Configure one condition:
      - left value: `{{ $json.status }}`
      - operator: `equals`
      - right value: `OK`

11. **Add the no-action node**
    - Create a `Set` node.
    - Name it `All Good - No Action`.
    - Connect the **true** output of `Status OK?` to it.
    - Add field:
      - `message` = `Check-in is current. No action needed.`

12. **Add the second routing IF node**
    - Create another `If` node.
    - Name it `Is Critical?`.
    - Connect the **false** output of `Status OK?` to it.
    - Configure condition:
      - left value: `{{ $json.status }}`
      - operator: `equals`
      - right value: `CRITICAL`

13. **Add the founder reminder Gmail node**
    - Create a `Gmail` node.
    - Name it `Send Reminder to Founder`.
    - Connect the **false** output of `Is Critical?` to it.
    - Configure Gmail credential.
    - Set recipient to:
      - `{{ $('Calculate Hours Since Last Check-in').item.json.founderEmail }}`
    - Set subject to:
      - `Dead Mans Switch: You Havent Checked In`
    - Set message to include:
      - founder name
      - hours since check-in
      - threshold
      - reminder to click the check-in link
      - warning that escalation will occur if no check-in happens soon

14. **Add the first emergency Gmail node**
    - Create another `Gmail` node.
    - Name it `Alert Emergency Contact 1`.
    - Connect the **true** output of `Is Critical?` to it.
    - Configure Gmail credential.
    - Set recipient to:
      - `{{ $('Calculate Hours Since Last Check-in').item.json.emergencyContact1 }}`
    - Set subject to:
      - `URGENT: {{ $('Calculate Hours Since Last Check-in').item.json.founderName }} Has Not Checked In`
    - Set body to include:
      - `emergencyMessage`
      - hours since check-in
      - automated alert notice

15. **Add the second emergency Gmail node**
    - Create another `Gmail` node.
    - Name it `Alert Emergency Contact 2`.
    - Connect `Alert Emergency Contact 1` → `Alert Emergency Contact 2`.
    - Configure the same Gmail credential.
    - Set recipient to:
      - `{{ $('Calculate Hours Since Last Check-in').item.json.emergencyContact2 }}`
    - Use the same message pattern as the first emergency email.

16. **Add the alert logging Google Sheets node**
    - Create a `Google Sheets` node.
    - Name it `Log Alert to Sheet`.
    - Connect `Alert Emergency Contact 2` → `Log Alert to Sheet`.
    - Configure:
      - operation: `append`
      - sheet: `AlertLog`
    - Map columns:
      - `timestamp` = `{{ $now.toISO() }}`
      - `status` = `{{ $('Calculate Hours Since Last Check-in').item.json.status }}`
      - `hours_since_checkin` = `{{ $('Calculate Hours Since Last Check-in').item.json.hoursSinceCheckIn }}`

17. **Add the webhook entry point**
    - Create a `Webhook` node.
    - Name it `Check-in Webhook`.
    - Set path to:
      - `dead-mans-switch-checkin`
    - Set response mode to:
      - `Last Node`
    - This becomes the URL the founder should bookmark.

18. **Add the check-in append node**
    - Create a `Google Sheets` node.
    - Name it `Record Check-in`.
    - Connect `Check-in Webhook` → `Record Check-in`.
    - Configure:
      - operation: `append`
      - sheet: `CheckIns`
    - Map columns:
      - `source` = `webhook`
      - `timestamp` = `{{ $now.toISO() }}`
    - Use the same Google Sheets credential as earlier.

19. **Add the webhook response node**
    - Create a `Set` node.
    - Name it `Confirm Check-in`.
    - Connect `Record Check-in` → `Confirm Check-in`.
    - Add fields:
      - `message` = `Check-in recorded. Your Dead Mans Switch timer has been reset.`
      - `checkedInAt` = `{{ $now.toISO() }}`

20. **Optionally add sticky notes for documentation**
    - Add a general note describing the system and sheet setup.
    - Add a note above the monitoring section.
    - Add a note above the routing section.
    - Add a note above the escalation section.
    - Add a note above the webhook section.

21. **Verify connection order**
    - Scheduled branch:
      - `Daily Check (9 AM)` → `Read Check-in Log` → `Calculate Hours Since Last Check-in` → `Status OK?`
      - `Status OK?` true → `All Good - No Action`
      - `Status OK?` false → `Is Critical?`
      - `Is Critical?` true → `Alert Emergency Contact 1` → `Alert Emergency Contact 2` → `Log Alert to Sheet`
      - `Is Critical?` false → `Send Reminder to Founder`
    - Webhook branch:
      - `Check-in Webhook` → `Record Check-in` → `Confirm Check-in`

22. **Test the webhook path**
    - Execute the webhook test URL from n8n.
    - Confirm a new row is appended to `CheckIns`.
    - Confirm the response contains:
      - a success message
      - a timestamp

23. **Test the monitoring path**
    - Manually execute the scheduled branch.
    - Test with:
      - a recent timestamp for `OK`
      - a timestamp older than threshold for `WARNING`
      - a timestamp older than twice threshold for `CRITICAL`

24. **Activate the workflow**
    - Activation is required for:
      - the production webhook URL
      - the 9 AM schedule
    - Bookmark the production webhook URL and use it as the daily check-in link.

25. **Recommended hardening adjustments**
    - Validate that email fields are not empty before sending.
    - Sort rows by timestamp before selecting the latest row if the sheet may be edited manually.
    - Separate configuration into its own sheet to avoid dependence on the first `CheckIns` row.
    - Consider logging warning reminders too, not only critical alerts.
    - Consider error handling branches so a first failed emergency email does not block the second contact or audit logging.

**Sub-workflow setup:**  
This workflow does **not** use sub-workflows and does not invoke any other workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Dead Man's Switch for Solo Founders — A personal safety net for solo founders, freelancers, and one-person businesses. If you stop checking in, your emergency contacts get alerted automatically. | Workflow purpose |
| Bookmark your check-in URL and visit it daily to reset the timer. | Operational usage |
| Create a Google Sheet with two tabs: `CheckIns` and `AlertLog`. | Required sheet structure |
| Add one seed row in `CheckIns` with your details and threshold (e.g. 24). | Required configuration convention |
| Connect your Gmail OAuth2 and Google Sheets credentials. | Required credentials |
| Activate the workflow and bookmark the webhook URL. | Deployment and usage |