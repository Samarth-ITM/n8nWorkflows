Recover missed calls with Twilio, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/recover-missed-calls-with-twilio--slack-and-google-sheets-13535


# Recover missed calls with Twilio, Slack and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Recover missed calls with Twilio, Slack and Google Sheets

**Purpose:**  
This workflow captures inbound call events from Twilio, filters for *missed* calls (failed/busy/no-answer), logs each missed call to Google Sheets, sends an automatic SMS response tailored to business hours, and notifies a Slack channel so the team can follow up quickly.

**Target use cases:**
- Lead capture / sales callbacks when inbound calls are missed
- Customer support missed-call recovery
- Creating an auditable missed-call log with real-time team alerting

### 1.1 Call Intake & Filtering
Receives Twilio webhook call events and routes only missed calls (failed/busy/no-answer) into the recovery process; answered calls are skipped.

### 1.2 Business Hours Evaluation & Logging
Computes whether “now” is within business hours (timezone-aware via Intl) and appends call details to Google Sheets.

### 1.3 Automated Customer SMS & Slack Notification
Sends an SMS to the caller based on business hours and posts a Slack notification to prompt immediate follow-up.

---

## 2. Block-by-Block Analysis

### Block 1 — Call Intake & Filtering
**Overview:**  
Receives Twilio’s call event payload via webhook and checks `CallStatus`. Only “no-answer”, “busy”, or “failed” proceed; all other statuses are treated as answered/ignored.

**Nodes Involved:**
- Receive Missed Call Webhook
- Check Call Failed/Busy/No-Answer
- Skip - Call Was Answered

#### Node: Receive Missed Call Webhook
- **Type / role:** Webhook trigger (`n8n-nodes-base.webhook`) — entry point receiving HTTP POST from Twilio.
- **Configuration choices:**
  - **HTTP Method:** POST  
  - **Path:** `missed-calls` (webhook endpoint will be based on your n8n base URL)
- **Key fields used downstream (expected from Twilio):**
  - `CallStatus`, `CallSid`, `From`, `To`, `Direction`
- **Connections:**
  - **Output →** Check Call Failed/Busy/No-Answer
- **Version-specific:** typeVersion **2.1**
- **Edge cases / failures:**
  - Twilio not configured to call the correct URL (404/405) or wrong method.
  - Payload missing expected fields (expressions later may resolve to `undefined`).
  - If n8n is behind auth/firewall, Twilio cannot reach the webhook.
- **Notes:** Node note says **“No worker available”** (often indicates execution capacity/queue/worker availability issues in some n8n setups).

#### Node: Check Call Failed/Busy/No-Answer
- **Type / role:** IF (`n8n-nodes-base.if`) — filters for missed calls.
- **Configuration choices (OR condition):**  
  Continues if any is true:
  - `{{$json.CallStatus}} == "no-answer"`
  - `{{$json.CallStatus}} == "busy"`
  - `{{$json.CallStatus}} == "failed"`
- **Connections:**
  - **True →** Check Business Hours
  - **False →** Skip - Call Was Answered
- **Version-specific:** typeVersion **2.3**
- **Edge cases / failures:**
  - Twilio may send other statuses (e.g., `completed`, `canceled`) depending on which Twilio webhook event is used; those will be treated as “answered” by this logic.
  - Missing `CallStatus` will evaluate as not matching, routing to “Skip”.

#### Node: Skip - Call Was Answered
- **Type / role:** NoOp (`n8n-nodes-base.noOp`) — terminates the non-missed path.
- **Configuration choices:** none.
- **Connections:** none (end of branch).
- **Version-specific:** typeVersion **1**
- **Edge cases:** none; used as an explicit sink to document the branch.

---

### Block 2 — Business Hours Evaluation & Logging
**Overview:**  
Determines whether the current time is within business hours (Mon–Fri, 9:00–17:00) in a specified timezone, then logs the missed call to Google Sheets for tracking.

**Nodes Involved:**
- Check Business Hours
- Log Missed Call to Google Sheets
- Is It Business Hours?

#### Node: Check Business Hours
- **Type / role:** Code (`n8n-nodes-base.code`) — computes a boolean flag for business hours.
- **Configuration choices (interpreted):**
  - Creates `now = new Date()`
  - Uses `Intl.DateTimeFormat` with `timeZone: 'America/New_York'`
  - Extracts:
    - `weekday` (e.g., Mon/Tue…)
    - `hour` (0–23)
  - Business hours definition:
    - Weekdays only (Mon–Fri)
    - Hours: `hour >= 9 && hour < 17`
  - **Returns:** `{ isBusinessHours: boolean }`
- **Connections:**
  - **Output →** Log Missed Call to Google Sheets
- **Version-specific:** typeVersion **2**
- **Edge cases / failures:**
  - Daylight Saving Time is handled by the IANA timezone, but your *business rules* might require different logic on holidays.
  - If the n8n runtime lacks full ICU data (rare in modern Node builds), timezone formatting could be unreliable.
  - Note: comment mentions `$now`, but code actually uses `new Date()`.

#### Node: Log Missed Call to Google Sheets
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — appends a row with missed call details.
- **Configuration choices:**
  - **Operation:** Append
  - **Document:** “Missed call recovery system” (Spreadsheet ID configured)
  - **Sheet:** “Sheet1”
  - **Mapping mode:** Define columns explicitly
  - **Columns written:**
    - **To:** `{{$('Receive Missed Call Webhook').item.json.To}}`
    - **SID:** `{{$('Receive Missed Call Webhook').item.json.CallSid}}`
    - **Time:** `{{$now}}` (n8n “now” value)
    - **Direction:** `{{$('Receive Missed Call Webhook').item.json.Direction}}`
    - **CallStatus:** `{{$('Receive Missed Call Webhook').item.json.CallStatus}}`
    - **Lead Number:** `{{$('Receive Missed Call Webhook').item.json.From}}`
- **Connections:**
  - **Output →** Is It Business Hours?
- **Credentials:** Google Sheets OAuth2
- **Version-specific:** typeVersion **4.7**
- **Edge cases / failures:**
  - OAuth token expired / missing permissions to spreadsheet.
  - Sheet schema mismatch (renamed columns, changed sheet name, different header row).
  - Rate limits or transient Google API errors.
  - Using `$('Receive Missed Call Webhook').item...` assumes that item still exists and aligns; if you later add nodes that split/merge items, this could break.

#### Node: Is It Business Hours?
- **Type / role:** IF (`n8n-nodes-base.if`) — routes to the correct SMS response.
- **Configuration choices:**
  - Condition: `{{$('Check Business Hours').item.json.isBusinessHours}} == true`
- **Connections:**
  - **True →** SMS - Call Back in 5 Minutes
  - **False →** SMS - Will Contact in Business Hours
- **Version-specific:** typeVersion **2.3**
- **Edge cases / failures:**
  - If the Code node output changes shape or is missing, condition will fail and route to “False” path by default behavior.

---

### Block 3 — Automated Customer SMS & Slack Notification
**Overview:**  
Sends an automatic SMS to the caller acknowledging the missed call, with different wording depending on business hours, then posts a Slack message to alert the team.

**Nodes Involved:**
- SMS - Call Back in 5 Minutes
- SMS - Will Contact in Business Hours
- Notify Team on Slack

#### Node: SMS - Call Back in 5 Minutes
- **Type / role:** Twilio (`n8n-nodes-base.twilio`) — sends SMS during business hours.
- **Configuration choices:**
  - **To:** `{{$('Receive Missed Call Webhook').item.json.From}}` (caller)
  - **From:** `{{$('Receive Missed Call Webhook').item.json.To}}` (your Twilio number)
  - **Message:** acknowledges missed call and asks to reply YES; includes STOP opt-out language
- **Connections:**
  - **Output →** Notify Team on Slack
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:**
  - Twilio credential/auth issues.
  - “From” number not SMS-capable or not owned in the Twilio account.
  - Regulatory compliance: “STOP” language present, but you may also need HELP handling depending on region.
  - If `From` is not an E.164 number (or is blocked), SMS send can fail.

#### Node: SMS - Will Contact in Business Hours
- **Type / role:** Twilio (`n8n-nodes-base.twilio`) — sends SMS outside business hours.
- **Configuration choices:**
  - **To/From:** same expressions as above
  - **Message:** explains business hours (Mon–Fri, 9–5 ET) and promises next-business-day outreach; includes STOP opt-out
- **Connections:**
  - **Output →** Notify Team on Slack
- **Version-specific:** typeVersion **1**
- **Edge cases / failures:** same as the other Twilio SMS node, plus:
  - Message content references ET; ensure it matches the timezone used in your business-hours code.

#### Node: Notify Team on Slack
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts a channel message for follow-up.
- **Configuration choices:**
  - **Mode:** Send message to a channel
  - **Channel:** `new-channel` (Channel ID `C0ABBRA98R3`)
  - **Text:**  
    `Missed call from {{ From }} at {{ $now.format('hh:mm a') }}. Click to call back.`
- **Connections:** none (end node).
- **Credentials:** Slack API OAuth
- **Version-specific:** typeVersion **2.4**
- **Edge cases / failures:**
  - Slack token lacks permission to post in the channel.
  - Channel ID changed or the bot removed from channel.
  - `$now.format(...)` depends on n8n’s date object behavior; if formatting fails in your version, replace with an explicit DateTime formatting approach.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Missed Call Webhook | Webhook | Ingest Twilio call events | — | Check Call Failed/Busy/No-Answer | This workflow automatically recovers missed inbound calls so no potential lead is lost.\nWhen a call is missed in Twilio, the workflow captures the event, checks the call status, determines whether it occurred during business hours, logs the call to Google Sheets, sends an automatic SMS to the caller, and notifies your team in Slack.\nIt ensures callers receive immediate acknowledgment and your team has visibility for follow up.\n\n## How it works\n\n- Receives inbound call events from Twilio\n\n- Filters for failed, busy, or no answer calls\n\n- Checks whether the call happened during business hours\n\n- Logs call details to Google Sheets\n\n- Sends an SMS reply based on business hours\n\n- Notifies your team in Slack\n\n## Setup steps\n\n- Connect your Twilio account and set the webhook URL\n\n- Connect Google Sheets and choose your tracking sheet\n\n- Connect Slack and select a notification channel\n\n- Adjust business hours and timezone in the code node\n\n- Customize the SMS message if needed |
| Check Call Failed/Busy/No-Answer | IF | Filter missed vs answered calls | Receive Missed Call Webhook | Check Business Hours; Skip - Call Was Answered | ## Call intake and filtering\n\nReceives inbound call events from Twilio via webhook. Filters for missed, busy, or failed calls and ignores answered ones. Only valid missed calls continue through the recovery flow to prevent unnecessary messages or notifications. |
| Skip - Call Was Answered | NoOp | Sink for non-missed calls | Check Call Failed/Busy/No-Answer | — | ## Call intake and filtering\n\nReceives inbound call events from Twilio via webhook. Filters for missed, busy, or failed calls and ignores answered ones. Only valid missed calls continue through the recovery flow to prevent unnecessary messages or notifications. |
| Check Business Hours | Code | Compute `isBusinessHours` boolean | Check Call Failed/Busy/No-Answer | Log Missed Call to Google Sheets | ## Logging and business hour check\n\nLogs missed call details to Google Sheets for tracking and reporting. Then checks whether the call occurred during defined business hours to determine the correct SMS response logic. |
| Log Missed Call to Google Sheets | Google Sheets | Append missed call row to sheet | Check Business Hours | Is It Business Hours? | ## Logging and business hour check\n\nLogs missed call details to Google Sheets for tracking and reporting. Then checks whether the call occurred during defined business hours to determine the correct SMS response logic. |
| Is It Business Hours? | IF | Route to correct SMS template | Log Missed Call to Google Sheets | SMS - Call Back in 5 Minutes; SMS - Will Contact in Business Hours | ## Logging and business hour check\n\nLogs missed call details to Google Sheets for tracking and reporting. Then checks whether the call occurred during defined business hours to determine the correct SMS response logic. |
| SMS - Call Back in 5 Minutes | Twilio | Send during-hours acknowledgement SMS | Is It Business Hours? | Notify Team on Slack | ## Automated SMS and team notification\n\nSends an automatic SMS to the caller based on business hours, either promising a quick callback or setting next day expectations. Notifies the team in Slack with caller details so follow up can happen immediately. |
| SMS - Will Contact in Business Hours | Twilio | Send after-hours acknowledgement SMS | Is It Business Hours? | Notify Team on Slack | ## Automated SMS and team notification\n\nSends an automatic SMS to the caller based on business hours, either promising a quick callback or setting next day expectations. Notifies the team in Slack with caller details so follow up can happen immediately. |
| Notify Team on Slack | Slack | Alert team in Slack channel | SMS - Call Back in 5 Minutes; SMS - Will Contact in Business Hours | — | ## Automated SMS and team notification\n\nSends an automatic SMS to the caller based on business hours, either promising a quick callback or setting next day expectations. Notifies the team in Slack with caller details so follow up can happen immediately. |
| Sticky Note | Sticky Note | Documentation / context | — | — | This workflow automatically recovers missed inbound calls so no potential lead is lost.\nWhen a call is missed in Twilio, the workflow captures the event, checks the call status, determines whether it occurred during business hours, logs the call to Google Sheets, sends an automatic SMS to the caller, and notifies your team in Slack.\nIt ensures callers receive immediate acknowledgment and your team has visibility for follow up.\n\n## How it works\n\n- Receives inbound call events from Twilio\n\n- Filters for failed, busy, or no answer calls\n\n- Checks whether the call happened during business hours\n\n- Logs call details to Google Sheets\n\n- Sends an SMS reply based on business hours\n\n- Notifies your team in Slack\n\n## Setup steps\n\n- Connect your Twilio account and set the webhook URL\n\n- Connect Google Sheets and choose your tracking sheet\n\n- Connect Slack and select a notification channel\n\n- Adjust business hours and timezone in the code node\n\n- Customize the SMS message if needed |
| Sticky Note1 | Sticky Note | Documentation / block label | — | — | ## Call intake and filtering\n\nReceives inbound call events from Twilio via webhook. Filters for missed, busy, or failed calls and ignores answered ones. Only valid missed calls continue through the recovery flow to prevent unnecessary messages or notifications. |
| Sticky Note2 | Sticky Note | Documentation / block label | — | — | ## Logging and business hour check\n\nLogs missed call details to Google Sheets for tracking and reporting. Then checks whether the call occurred during defined business hours to determine the correct SMS response logic. |
| Sticky Note3 | Sticky Note | Documentation / block label | — | — | ## Automated SMS and team notification\n\nSends an automatic SMS to the caller based on business hours, either promising a quick callback or setting next day expectations. Notifies the team in Slack with caller details so follow up can happen immediately. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: “Receive Missed Call Webhook”**
   - Type: **Webhook**
   - Method: **POST**
   - Path: **`missed-calls`**
   - Save the node to generate the **Test** and **Production** webhook URLs.

3. **Configure Twilio to call the webhook**
   - In Twilio Console (Phone Number settings), set the Voice webhook (or the appropriate status callback, depending on your Twilio design) to the n8n webhook **Production URL**.
   - Ensure Twilio sends parameters like `CallSid`, `CallStatus`, `From`, `To`, `Direction`.

4. **Add node: “Check Call Failed/Busy/No-Answer”**
   - Type: **IF**
   - Condition group: **OR**
   - Add string equals conditions:
     - `{{$json.CallStatus}}` equals `no-answer`
     - `{{$json.CallStatus}}` equals `busy`
     - `{{$json.CallStatus}}` equals `failed`
   - Connect:
     - Webhook → IF

5. **Add node: “Skip - Call Was Answered”**
   - Type: **No Operation (NoOp)**
   - Connect:
     - IF (false output) → Skip node

6. **Add node: “Check Business Hours”**
   - Type: **Code**
   - Paste logic equivalent to:
     - Timezone: `America/New_York`
     - Business hours: Mon–Fri, 9:00–17:00
     - Output: `{ isBusinessHours: ... }`
   - Connect:
     - IF (true output) → Check Business Hours
   - Adjust:
     - Timezone string to your IANA timezone (e.g., `Europe/Paris`)
     - Working days/hours as needed

7. **Add node: “Log Missed Call to Google Sheets”**
   - Type: **Google Sheets**
   - Credentials: **Google Sheets OAuth2** (connect an account with access to the target sheet)
   - Operation: **Append**
   - Select Spreadsheet (document) and Sheet tab
   - Map columns (create headers in the sheet to match), e.g.:
     - SID ← `{{$('Receive Missed Call Webhook').item.json.CallSid}}`
     - Lead Number ← `{{$('Receive Missed Call Webhook').item.json.From}}`
     - To ← `{{$('Receive Missed Call Webhook').item.json.To}}`
     - CallStatus ← `{{$('Receive Missed Call Webhook').item.json.CallStatus}}`
     - Direction ← `{{$('Receive Missed Call Webhook').item.json.Direction}}`
     - Time ← `{{$now}}`
   - Connect:
     - Check Business Hours → Google Sheets

8. **Add node: “Is It Business Hours?”**
   - Type: **IF**
   - Condition: boolean equals
     - Left: `{{$('Check Business Hours').item.json.isBusinessHours}}`
     - Right: `true`
   - Connect:
     - Google Sheets → Is It Business Hours?

9. **Add node: “SMS - Call Back in 5 Minutes”**
   - Type: **Twilio**
   - Credentials: connect Twilio account (Account SID/Auth Token or configured credential method)
   - Configure SMS send:
     - To: `{{$('Receive Missed Call Webhook').item.json.From}}`
     - From: `{{$('Receive Missed Call Webhook').item.json.To}}`
     - Message: your “business hours” template
   - Connect:
     - IF (true output) → this Twilio node

10. **Add node: “SMS - Will Contact in Business Hours”**
    - Type: **Twilio**
    - Same credentials
    - To/From same expressions
    - Message: your “after hours” template (include business hours text)
    - Connect:
      - IF (false output) → this Twilio node

11. **Add node: “Notify Team on Slack”**
    - Type: **Slack**
    - Credentials: Slack OAuth credential with permission to post messages
    - Operation: post message to channel
    - Select the channel
    - Text example:
      - `Missed call from {{ $('Receive Missed Call Webhook').item.json.From }} at {{ $now.format('hh:mm a') }}. Click to call back.`
    - Connect:
      - Both Twilio nodes → Slack node

12. **Activate the workflow** and test with a real Twilio call scenario that results in `no-answer`, `busy`, or `failed`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically recovers missed inbound calls so no potential lead is lost. When a call is missed in Twilio, the workflow captures the event, checks the call status, determines whether it occurred during business hours, logs the call to Google Sheets, sends an automatic SMS to the caller, and notifies your team in Slack. It ensures callers receive immediate acknowledgment and your team has visibility for follow up. | Sticky note (workflow-level description) |
| Setup steps: Connect Twilio and set webhook URL; connect Google Sheets and choose tracking sheet; connect Slack and select channel; adjust business hours/timezone in Code node; customize SMS messages. | Sticky note (setup guidance) |
| “Call intake and filtering” block note: filters for missed/busy/failed and ignores answered ones to prevent unnecessary messages. | Sticky note (block description) |
| “Logging and business hour check” block note: logs to Sheets then determines correct SMS logic based on business hours. | Sticky note (block description) |
| “Automated SMS and team notification” block note: sends SMS based on hours and notifies Slack for immediate follow-up. | Sticky note (block description) |