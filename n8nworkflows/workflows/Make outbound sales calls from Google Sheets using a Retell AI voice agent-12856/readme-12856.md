Make outbound sales calls from Google Sheets using a Retell AI voice agent

https://n8nworkflows.xyz/workflows/make-outbound-sales-calls-from-google-sheets-using-a-retell-ai-voice-agent-12856


# Make outbound sales calls from Google Sheets using a Retell AI voice agent

## 1. Workflow Overview

**Purpose:** Automatically place outbound sales calls to new leads added to a Google Sheet using a **Retell AI voice agent**, while avoiding calling outside business hours in the lead’s local timezone. After the call ends, the workflow **polls Retell** for completion and then **updates the Google Sheet** with call results.

**Primary use cases**
- Automated first-touch outbound calling for inbound leads captured into Google Sheets
- Timezone-aware dialing to reduce poor-call experiences and complaints
- Writing call outcomes (status, transcript/summary/etc. if mapped) back to the source sheet

### 1.1 Logical Blocks
1. **Lead intake (Google Sheets Trigger)**
2. **Phone sanitization + safety filter (avoid re-calling)**
3. **Timezone inference + business-hours gate**
4. **Call creation via Retell AI**
5. **Call completion polling loop**
6. **Write-back to Google Sheets (call info/status)**

---

## 2. Block-by-Block Analysis

### Block 2.1 — Lead Intake (Google Sheets)
**Overview:** Watches a Google Sheet and emits a workflow execution whenever a new row is added (a new lead).  
**Nodes involved:** `New lead` (+ sticky note context)

#### Node: New lead
- **Type / role:** Google Sheets Trigger (`googleSheetsTrigger`) — entry point; “row added” polling trigger.
- **Configuration (interpreted):**
  - Event: **Row Added**
  - Poll schedule: **Every minute**
  - Spreadsheet: **DocumentId** is empty in JSON (must be selected)
  - SheetName: configured to “list” mode but currently empty (must be selected)
- **Credentials:** Google Sheets Trigger OAuth2 (`Google Sheets Trigger account`)
- **Inputs/Outputs:**
  - **Input:** none (trigger)
  - **Output:** the new row data as JSON (expects columns like `Phone Number`, `Status`, `Name`)
- **Edge cases / failures:**
  - Misconfigured/empty `documentId` or sheet selection → trigger won’t function
  - OAuth token expiry / revoked permissions
  - Polling may miss rapid bursts if sheet updates exceed polling resolution
- **Sticky note(s):**
  - “## When a lead arrives”

---

### Block 2.2 — Sanitize Phone + Filter Already-Called
**Overview:** Normalizes phone numbers to digits-only and filters out leads that are missing numbers or already processed (Status not “Not Called”).  
**Nodes involved:** `Cleanse phone numbers`, `Filter numbers we've already called`

#### Node: Cleanse phone numbers
- **Type / role:** Code node — transforms data.
- **Configuration (interpreted):**
  - For each incoming item:
    - `Phone Number` is converted to a string and all non-digits removed: `replace(/\D/g, "")`
- **Key variables/fields:**
  - Reads/writes: `item.json["Phone Number"]`
- **Inputs/Outputs:**
  - **Input:** new row items from trigger
  - **Output:** same items with sanitized `Phone Number`
- **Edge cases / failures:**
  - If `Phone Number` is `null/undefined`, `String(undefined)` becomes `"undefined"` → cleans to `""` (empty). This is later filtered out, but be aware.
  - International dialing: removing `+` is intended, but you must ensure the remaining digits include country code if you rely on it later.
- **Sticky note(s):**
  - “## Santise the phone number …”

#### Node: Filter numbers we've already called
- **Type / role:** Filter node — gatekeeping to reduce duplicates/damage.
- **Configuration (interpreted):** Keeps items only when:
  1. `Phone Number` **exists** (non-empty)
  2. `Status` equals **"Not Called"**
- **Inputs/Outputs:**
  - **Input:** sanitized leads
  - **Output:** only callable leads
- **Edge cases / failures:**
  - If your sheet uses different casing (“Not called” vs “Not Called”), the strict equality will block all leads.
  - If `Status` column is missing, condition fails and lead is dropped.
  - “Exists” check is string-based; ensure it behaves as expected for `""` vs `null`.
- **Sticky note(s):**
  - “## Santise the phone number …”

---

### Block 2.3 — Timezone Inference + Business Hours Gate
**Overview:** Infers a lead timezone from the phone country code (simple mapping), computes local time, and allows calling only between 08:00–17:00 local hour; otherwise waits and retries later.  
**Nodes involved:** `Get lead's timezone`, `Is it between 8am-5pm for them?`, `Wait`

#### Node: Get lead's timezone
- **Type / role:** Code node — enriches lead with timezone and local time fields.
- **Configuration (interpreted):**
  - Cleans `Phone Number` again with `replace(/\D/g,'')`
  - Uses a hardcoded `timezoneMap` keyed by country calling codes (e.g., `44 → Europe/London`, `1 → America/New_York`, etc.)
  - Picks the **longest matching** calling code prefix
  - Uses `Intl.DateTimeFormat` to compute:
    - `localHour`, `localMinute`, `dayOfWeek`, `localTimeFormatted`
  - Defaults: timezone `Europe/London`, countryCode `44` if no match
- **Key output fields added:**
  - `timezone`, `countryCode`, `localHour`, `localMinute`, `dayOfWeek`, `localTimeFormatted`
- **Inputs/Outputs:**
  - **Input:** filtered lead
  - **Output:** lead + computed local time fields
- **Edge cases / failures:**
  - **Country code ambiguity:** NANP `1` spans multiple timezones; mapping to `America/New_York` may be wrong.
  - **Missing country code:** if numbers are local-format only, mapping may default incorrectly.
  - `Intl.DateTimeFormat` depends on runtime ICU support; in typical n8n deployments it works, but very slim containers could behave unexpectedly.
- **Sticky note(s):**
  - “## Check it's daytime (in their timezone) …”

#### Node: Is it between 8am-5pm for them?
- **Type / role:** IF node — business-hours decision gate.
- **Configuration (interpreted):**
  - True if:
    - `localHour >= 8`
    - `localHour <= 17`
- **Inputs/Outputs:**
  - **Input:** enriched lead
  - **True output:** proceed to call
  - **False output:** wait then re-check (loop)
- **Edge cases / failures:**
  - Uses hour-only; will allow calling at exactly 17:59 because only `localHour <= 17` is checked (minute ignored).
  - Weekends are computed but not used; calls on Saturday/Sunday will still be allowed if within hours.
- **Sticky note(s):**
  - “## Check it's daytime (in their timezone) …”

#### Node: Wait
- **Type / role:** Wait node — delays execution then retries the timezone/time check.
- **Configuration (interpreted):**
  - Wait duration: **2 hours**
- **Inputs/Outputs:**
  - **Input:** leads outside allowed hours
  - **Output:** same item, after delay, routed back to `Get lead's timezone`
- **Edge cases / failures:**
  - High volume of off-hours leads can create many suspended executions.
  - If you want “wait until 8am local”, a fixed 2-hour wait can be inefficient and still land outside the window.
- **Sticky note(s):**
  - “## Check it's daytime (in their timezone) …”

---

### Block 2.4 — Create Call in Retell AI
**Overview:** Initiates a phone call via Retell’s API using your Retell phone number, the lead’s phone number, and a Retell agent ID.  
**Nodes involved:** `Make phone call`

#### Node: Make phone call
- **Type / role:** HTTP Request — creates a call via Retell API.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.retellai.com/v2/create-phone-call`
  - Body parameters:
    - `from_number`: **placeholder** (“replace with your Retell AI phone number”)
    - `to_number`: `{{ $json['Phone Number'] }}`
    - `agent_id`: **placeholder** (“replace with your Retell AI agent ID”)
  - Authentication: **Generic credential type** configured as **HTTP Bearer Auth**
- **Inputs/Outputs:**
  - **Input:** approved lead row (with `Phone Number`)
  - **Output:** Retell response; expected to include a `call_id` (used later by polling)
- **Edge cases / failures:**
  - Missing/invalid API key → 401/403
  - Wrong `from_number` (not provisioned) → call creation fails
  - Incorrect `agent_id` → 4xx error
  - Retell may expect E.164 formatting; this workflow sends digits-only. Confirm Retell’s requirement—if E.164 is required, you must add `+` and correct country code.
- **Sticky note(s):**
  - “## Make Phone Call … Create a free Retell account …”

---

### Block 2.5 — Poll Until Call Ends
**Overview:** Repeatedly checks Retell for the call status every minute until the status becomes `ended`.  
**Nodes involved:** `Wait 1 minute`, `Get Call Status`, `Call has finished`

#### Node: Wait 1 minute
- **Type / role:** Wait node — pacing the polling loop.
- **Configuration (interpreted):**
  - Wait duration: **1 minute**
- **Connections:**
  - From `Make phone call` → `Wait 1 minute`
  - From `Call has finished` (false branch) → back to `Wait 1 minute`
- **Edge cases / failures:**
  - Many concurrent calls can create many suspended executions.
- **Sticky note(s):**
  - “## Wait for the call to finish … (webhook alternative link)”
  - Link: https://docs.retellai.com/features/webhook-overview

#### Node: Get Call Status
- **Type / role:** HTTP Request — retrieves status for an existing call.
- **Configuration (interpreted):**
  - Method: default GET (not explicitly set)
  - URL (expression):  
    `https://api.retellai.com/v2/get-call/{{$node['Make phone call'].json.call_id}}`
  - Response: **Full response enabled** (so status may be under `body`)
  - Authentication: predefined Bearer Auth credential (`Bearer Auth account 4`)
- **Inputs/Outputs:**
  - **Input:** item after 1-minute wait
  - **Output:** HTTP response object; call status expected at `{{$json.body.call_status}}`
- **Edge cases / failures:**
  - If `Make phone call` did not return `call_id`, expression fails or URL becomes invalid.
  - Rate limits: polling every minute is modest, but at scale it can still hit API limits.
  - Network timeouts / 5xx responses should be expected; consider retries.
- **Sticky note(s):**
  - “## Wait for the call to finish …”

#### Node: Call has finished
- **Type / role:** IF node — loop controller.
- **Configuration (interpreted):**
  - True if `{{$json.body.call_status}} == "ended"`
  - True branch → update sheet
  - False branch → wait 1 minute and poll again
- **Edge cases / failures:**
  - If Retell uses different terminal statuses (e.g., `completed`, `failed`, `no_answer`), this condition may never match, causing an infinite polling loop. Consider expanding to a set of terminal statuses.
- **Sticky note(s):**
  - “## Wait for the call to finish …”

---

### Block 2.6 — Update Google Sheets with Call Information
**Overview:** Writes call outcomes back into the Google Sheet row (intended: status + summary/sentiment/transcript/etc., depending on mapping you configure).  
**Nodes involved:** `Update Google Sheets with call information`

#### Node: Update Google Sheets with call information
- **Type / role:** Google Sheets node — updates an existing row.
- **Configuration (interpreted):**
  - Operation: **Update**
  - DocumentId: empty in JSON (must be set)
  - Sheet: `gid=0` selected by id
  - Columns mapping: currently empty (`mappingMode: defineBelow` but nothing defined)
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Inputs/Outputs:**
  - **Input:** item from `Call has finished` true branch (contains call status response)
  - **Output:** update result from Google
- **Edge cases / failures:**
  - Update requires a way to identify the row (often `Row ID` or a matching column). With empty mapping/matching config, this node will not work until configured.
  - If you want to write transcript/summary, you must extract fields from `Get Call Status` response and map them to columns.
- **Sticky note(s):**
  - “## Update Google Sheet”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / canvas annotation | — | — | ## Make Phone Call \n**⚙️ Setup:**\n**Create a free Retell account**  [Retell AI – $10 FREE credits](https://dashboard.retellai.com/?ref=retell-n8n) |
| Sticky Note1 | Sticky Note | Documentation / canvas annotation | — | — | ## When a lead arrives |
| Sticky Note2 | Sticky Note | Documentation / canvas annotation | — | — | ## Santise the phone number\nPhone numbers are messy; For the call to work, we need to strip away all spaces, dashes, plus signs, and brackets so all we have left are numbers. Next, we're filtering out any numbers we've already called for a bit of damage control. |
| Make phone call | HTTP Request | Create outbound call in Retell AI | Is it between 8am-5pm for them? (true) | Wait 1 minute | ## Make Phone Call \n**⚙️ Setup:**\n**Create a free Retell account**  [Retell AI – $10 FREE credits](https://dashboard.retellai.com/?ref=retell-n8n) |
| Sticky Note3 | Sticky Note | Documentation / canvas annotation | — | — | ## Wait for the call to finish\nThis poll's Retell AI until the call has finished. For a more elegant approach, you can use [a webhook](https://docs.retellai.com/features/webhook-overview) but this approach is simpler / beginner-friendly. |
| Sticky Note4 | Sticky Note | Documentation / canvas annotation | — | — | ## Check it's daytime (in their timezone)\nWe don't want to call leads at 2am, so this step checks the lead's timezone (based on their phone number) and only calls within the 8am-5pm window. |
| Get lead's timezone | Code | Infer timezone + compute local time fields | Filter numbers we've already called; Wait | Is it between 8am-5pm for them? | ## Check it's daytime (in their timezone)\nWe don't want to call leads at 2am, so this step checks the lead's timezone (based on their phone number) and only calls within the 8am-5pm window. |
| Cleanse phone numbers | Code | Strip non-digits from `Phone Number` | New lead | Filter numbers we've already called | ## Santise the phone number\nPhone numbers are messy; For the call to work, we need to strip away all spaces, dashes, plus signs, and brackets so all we have left are numbers. Next, we're filtering out any numbers we've already called for a bit of damage control. |
| New lead | Google Sheets Trigger | Trigger on new lead row | — | Cleanse phone numbers | ## When a lead arrives |
| Is it between 8am-5pm for them? | IF | Allow calling only during local business hours | Get lead's timezone | Make phone call (true); Wait (false) | ## Check it's daytime (in their timezone)\nWe don't want to call leads at 2am, so this step checks the lead's timezone (based on their phone number) and only calls within the 8am-5pm window. |
| Wait | Wait | Delay then re-check time window | Is it between 8am-5pm for them? (false) | Get lead's timezone | ## Check it's daytime (in their timezone)\nWe don't want to call leads at 2am, so this step checks the lead's timezone (based on their phone number) and only calls within the 8am-5pm window. |
| Wait 1 minute | Wait | Polling delay | Make phone call; Call has finished (false) | Get Call Status | ## Wait for the call to finish\nThis poll's Retell AI until the call has finished. For a more elegant approach, you can use [a webhook](https://docs.retellai.com/features/webhook-overview) but this approach is simpler / beginner-friendly. |
| Call has finished | IF | Stop polling when status is `ended` | Get Call Status | Update Google Sheets with call information (true); Wait 1 minute (false) | ## Wait for the call to finish\nThis poll's Retell AI until the call has finished. For a more elegant approach, you can use [a webhook](https://docs.retellai.com/features/webhook-overview) but this approach is simpler / beginner-friendly. |
| Get Call Status | HTTP Request | Fetch Retell call status by `call_id` | Wait 1 minute | Call has finished | ## Wait for the call to finish\nThis poll's Retell AI until the call has finished. For a more elegant approach, you can use [a webhook](https://docs.retellai.com/features/webhook-overview) but this approach is simpler / beginner-friendly. |
| Sticky Note5 | Sticky Note | Documentation / canvas annotation | — | — | ## Update Google Sheet |
| Filter numbers we've already called | Filter | Only keep uncalled leads with a phone number | Cleanse phone numbers | Get lead's timezone | ## Santise the phone number\nPhone numbers are messy; For the call to work, we need to strip away all spaces, dashes, plus signs, and brackets so all we have left are numbers. Next, we're filtering out any numbers we've already called for a bit of damage control. |
| Sticky Note7 | Sticky Note | Documentation / canvas annotation (overview + setup notes) | — | — | ## 📞 Automated Outbound Lead Caller\n\nAutomatically call new leads from a Google Sheet using [Retell AI](https://www.retellai.com/?via=intellagents), with built-in timezone awareness to ensure you're only calling during business hours.\n\n### What it does\n1. **Triggers** when a new row is added to your Google Sheet\n2. **Cleans** the phone number (removes spaces, dashes, brackets)\n3. **Checks timezone** based on country code—only calls between 8am-5pm local time\n4. **Makes the call** via Retell AI using your configured voice agent\n5. **Polls for completion** then updates the sheet with \"Called\" status, a call summary, sentiment, transcript & more.\n\n### ⚙️ Setup\n\n**1. Retell AI Account**\n[Create a free account and get $10 in credits](https://www.retellai.com/?via=intellagents\n) (~73 mins of calls)\n\n\n**2. Add Your Credentials**\n- Retell AI API key → Bearer Auth credential\n- Google Sheets OAuth\n\n\n**3. Configure the \"Make phone call\" Node**\n- `from_number`: Your Retell phone number (with country code)\n- `to_number`: Change to `{{ $json['Phone Number'] }}`\n- `agent_id`: Your Retell agent ID\n\n\n**4. Connect Your Google Sheet**\nColumns needed: `Phone Number`, `Name`, `Status`\nSet new leads to Status = \"Not called\"\n\n### 📺 Video Walkthrough\n[COMING SOON]\n\n### 💡 Customisation Ideas\n- Swap the trigger for a webhook, form, or CRM\n- Adjust calling hours in the IF node (currently 8am-5pm)\n- Add more country codes to the timezone mapping\n- Send results to Slack, CRM, or email\n\n---\nBuilt by Marcus Taylor\n📺 @intellagents | 🌐 [voiceai.guide](https://voiceai.guide) |
| Update Google Sheets with call information | Google Sheets | Write call results back to the sheet | Call has finished (true) | — | ## Update Google Sheet |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: *Outbound sales calls from Google Sheets using Retell AI voice agent* (or your preferred title)

2. **Add Google Sheets Trigger**
   - Node: **Google Sheets Trigger**
   - Event: **Row Added**
   - Polling: **Every minute**
   - Select:
     - **Document** (your spreadsheet)
     - **Sheet** (tab that receives leads)
   - Credentials: create/select **Google Sheets Trigger OAuth2**
   - Ensure your sheet has at least columns: `Phone Number`, `Status` (and optionally `Name`)

3. **Add “Cleanse phone numbers” (Code node)**
   - Node: **Code**
   - Paste logic to normalize phone numbers (digits-only), e.g.:
     - For each item: `Phone Number = String(Phone Number).replace(/\D/g,"")`
   - Connect: **New lead → Cleanse phone numbers**

4. **Add “Filter numbers we’ve already called” (Filter node)**
   - Node: **Filter**
   - Conditions (AND):
     - `Phone Number` **exists**
     - `Status` **equals** `Not Called` (match your sheet exactly)
   - Connect: **Cleanse phone numbers → Filter numbers we've already called**

5. **Add “Get lead’s timezone” (Code node)**
   - Node: **Code**
   - Implement:
     - Country code → timezone mapping
     - Compute `localHour` using `Intl.DateTimeFormat` with `timeZone`
     - Output merges original row fields plus `timezone`, `localHour`, etc.
   - Connect: **Filter numbers we've already called → Get lead's timezone**

6. **Add “Is it between 8am-5pm for them?” (IF node)**
   - Node: **IF**
   - Conditions (AND):
     - `{{$json.localHour}} >= 8`
     - `{{$json.localHour}} <= 17`
   - Connect: **Get lead's timezone → IF**

7. **Add “Wait” (2 hours) for after-hours leads**
   - Node: **Wait**
   - Unit: **Hours**
   - Amount: **2**
   - Connect: **IF (false) → Wait**
   - Connect: **Wait → Get lead's timezone** (re-check after delay)

8. **Add “Make phone call” (HTTP Request)**
   - Node: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.retellai.com/v2/create-phone-call`
   - Authentication: **Bearer Auth**
     - Create/select an **HTTP Bearer Auth** credential with your **Retell API key**
   - Body: send parameters
     - `from_number`: your Retell number (include country code as Retell expects)
     - `to_number`: `{{ $json['Phone Number'] }}`
     - `agent_id`: your Retell agent ID
   - Connect: **IF (true) → Make phone call**

9. **Add polling loop nodes**
   1) Node: **Wait** (rename “Wait 1 minute”)
      - Unit: Minutes, Amount: 1
      - Connect: **Make phone call → Wait 1 minute**
   2) Node: **HTTP Request** (rename “Get Call Status”)
      - URL: `https://api.retellai.com/v2/get-call/{{$node["Make phone call"].json.call_id}}`
      - Authentication: Bearer Auth (same Retell credential)
      - Enable “Full response” if you plan to read `body.call_status`
      - Connect: **Wait 1 minute → Get Call Status**
   3) Node: **IF** (rename “Call has finished”)
      - Condition: `{{$json.body.call_status}} equals ended`
      - Connect: **Get Call Status → Call has finished**
      - Connect: **Call has finished (false) → Wait 1 minute** (loop)

10. **Add Google Sheets “Update” node**
   - Node: **Google Sheets**
   - Operation: **Update**
   - Select the same **Document** and **Sheet**
   - Configure how to identify the row to update:
     - Prefer using the row number/row ID provided by the trigger, or a unique key column
   - Map fields to columns (examples you likely want):
     - `Status` → “Called” (or “Completed”)
     - Any call outputs you can extract from the Retell status response (transcript, summary, sentiment, recording link, etc.) into dedicated sheet columns
   - Credentials: **Google Sheets OAuth2**
   - Connect: **Call has finished (true) → Update Google Sheets with call information**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Retell signup / credits link | https://dashboard.retellai.com/?ref=retell-n8n |
| Retell webhook alternative to polling | https://docs.retellai.com/features/webhook-overview |
| Retell AI main site link in workflow notes | https://www.retellai.com/?via=intellagents |
| Project credit | Built by Marcus Taylor |
| Site link | https://voiceai.guide |
| Sheet expectations from notes | Columns needed: `Phone Number`, `Name`, `Status`; set new leads to Status = "Not called" (note: workflow filter expects **"Not Called"** — align these) |
| Customisation ideas (from sticky note) | Adjust hours, add country codes, change trigger source, send results to Slack/CRM/email |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.