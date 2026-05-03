Make outbound sales calls from Google Sheets using a VAPI voice agent

https://n8nworkflows.xyz/workflows/make-outbound-sales-calls-from-google-sheets-using-a-vapi-voice-agent-13114


# Make outbound sales calls from Google Sheets using a VAPI voice agent

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically place outbound sales calls to newly added leads in a Google Sheet using a VAPI voice agent, while enforcing “business hours” calling based on the lead’s inferred timezone. After the call ends, the workflow polls VAPI until completion and then updates the Google Sheet.

**Target use cases:**
- Outbound lead calling for sales/qualification
- Rapid follow-up on inbound leads captured in Google Sheets
- Timezone-safe calling windows (avoid calling at night)

### 1.1 Lead Intake (Google Sheets)
Triggers when a new row is added to a target sheet.

### 1.2 Phone Number Sanitization + Eligibility Filter
Normalizes phone numbers to digits only and filters to avoid re-calling leads already processed (based on `Status` column).

### 1.3 Timezone Derivation + Business Hours Gate
Infers timezone from country code prefix and only allows calls between 08:00–17:00 local time. If outside the window, waits and rechecks later.

### 1.4 Call Placement (VAPI)
Creates a VAPI call using your configured VAPI assistant and VAPI phone number.

### 1.5 Call Completion Polling
Polls VAPI call status every minute until the call is `ended`.

### 1.6 Sheet Update
Updates the Google Sheet with call information (configuration is present but column mapping is not fully defined in the provided workflow).

---

## 2. Block-by-Block Analysis

### Block 1 — Lead Intake (Google Sheets Trigger)

**Overview:** Watches a Google Sheet for new rows and starts the workflow for each added lead.

**Nodes involved:**
- **New lead** (Google Sheets Trigger)

#### Node: New lead
- **Type / role:** `googleSheetsTrigger` — Polling trigger for “rowAdded”.
- **Key configuration (interpreted):**
  - **Event:** Row added
  - **Polling:** Every minute
  - **Document ID / Sheet name:** Present but not filled in the JSON export (must be configured in UI).
- **Inputs/Outputs:**
  - **Output →** `Cleanse phone numbers`
- **Credentials:** Google Sheets Trigger OAuth2 (required).
- **Edge cases / failures:**
  - Missing/invalid Google OAuth2 credentials → auth failures.
  - Document ID or Sheet not selected → node won’t run correctly.
  - Polling every minute may miss rapid changes if sheet edits are unusual (rare) or may create duplicates if rows are added/modified unexpectedly.

**Sticky note covering this block:**
- “## When a lead arrives”

---

### Block 2 — Phone Sanitization + “Not Called” Filter

**Overview:** Cleans the phone number into a digits-only string and filters out rows that are not eligible (missing number or already called).

**Nodes involved:**
- **Cleanse phone numbers** (Code)
- **Filter numbers we’ve already called** (Filter)

#### Node: Cleanse phone numbers
- **Type / role:** `code` — Data normalization.
- **Key configuration choices:**
  - For every incoming item:
    - `item.json["Phone Number"] = String(item.json["Phone Number"]).replace(/\D/g, "")`
  - Removes all non-digit characters (spaces, `+`, parentheses, dashes, etc.).
- **Inputs/Outputs:**
  - **Input ←** `New lead`
  - **Output →** `Filter numbers we've already called`
- **Edge cases / failures:**
  - If `Phone Number` is `null/undefined`, `String(undefined)` becomes `"undefined"` → cleans to `""` (empty). The subsequent Filter checks existence, so it will be blocked, but you should be aware of this implicit behavior.
  - International numbers lose the `+` sign; downstream logic assumes country code is still present as leading digits.

#### Node: Filter numbers we've already called
- **Type / role:** `filter` — Gatekeeping / damage control.
- **Key configuration choices (AND conditions):**
  1. `Phone Number` **exists**
  2. `Status` equals **"Not Called"** (note: capitalization matters)
- **Inputs/Outputs:**
  - **Input ←** `Cleanse phone numbers`
  - **Output →** `Get lead's timezone`
- **Edge cases / failures:**
  - If your sheet uses `Not called` (lowercase “c”) but node expects `Not Called`, everything will be filtered out.
  - If “Status” column is missing, condition fails.
  - “Exists” check ensures the field exists, but does not guarantee it’s non-empty; you may want an additional “not empty” check in practice.

**Sticky note covering this block:**
- “## Santise the phone number … filtering out any numbers we’ve already called …”

---

### Block 3 — Timezone Derivation + Business Hours Gate

**Overview:** Infers a timezone based on the phone number’s leading country code, computes local time via the `Intl.DateTimeFormat` API, and allows calls only between 08:00 and 17:00 local hour. Outside that range, waits 2 hours and rechecks.

**Nodes involved:**
- **Get lead’s timezone** (Code)
- **Is it between 8am-5pm for them?** (IF)
- **Wait** (Wait)

#### Node: Get lead’s timezone
- **Type / role:** `code` — Enrichment node to add timezone/time fields to the lead.
- **Key configuration choices:**
  - Reads: `$input.first().json['Phone Number']`
  - Cleans to digits only again: `cleaned.replace(/\D/g,'')`
  - Uses a **hardcoded country code → timezone map**, e.g.:
    - `44 → Europe/London`
    - `1 → America/New_York`
    - `61 → Australia/Sydney`
    - `33 → Europe/Paris`
    - …etc.
  - Matches **longest codes first** to avoid `3` matching before `353`, etc.
  - Uses `Intl.DateTimeFormat(..., { timeZone })` to compute:
    - `localHour`, `localMinute`, `weekdayName`, `dayOfWeek`
    - `localTimeFormatted` (e.g., `Monday 09:15`)
  - Outputs merged object: `{ ...originalLeadFields, timezone, countryCode, localHour, ... }`
- **Inputs/Outputs:**
  - **Input ←** `Filter numbers we've already called` (or after a `Wait` loop)
  - **Output →** `Is it between 8am-5pm for them?`
- **Edge cases / failures:**
  - **Wrong timezone inference:** Country code ≠ timezone (e.g., US has multiple timezones; this defaults to `America/New_York`).
  - **Missing/short numbers:** Defaults to `Europe/London` / `44` if no prefix matches.
  - `Intl` timezone support depends on Node.js runtime in n8n (generally available; but custom/self-hosted minimal builds could cause issues).
  - Lead phone numbers without country code will misclassify.

#### Node: Is it between 8am-5pm for them?
- **Type / role:** `if` — Business-hours gate.
- **Key configuration choices:**
  - Condition AND:
    - `{{$json.localHour}} >= 8`
    - `{{$json.localHour}} <= 17`
- **Branching:**
  - **True →** `Make phone call`
  - **False →** `Wait` (2 hours), then loops back to timezone check
- **Inputs/Outputs:**
  - **Input ←** `Get lead’s timezone`
  - **Outputs →** `Make phone call` (true), `Wait` (false)
- **Edge cases / failures:**
  - Only checks hour, not weekday (so it may call weekends).
  - 17 is included; if you want strict “before 5pm”, use `< 17`.
  - No consideration for holidays or per-lead calling preferences.

#### Node: Wait
- **Type / role:** `wait` — Delay then retry.
- **Key configuration choices:**
  - Wait **2 hours**, then continue.
- **Inputs/Outputs:**
  - **Input ←** IF (false branch)
  - **Output →** `Get lead's timezone`
- **Edge cases / failures:**
  - Creates long-running executions; ensure your n8n instance supports waits (and you understand execution retention).
  - High volume of leads outside calling window can accumulate many waiting executions.

**Sticky note covering this block:**
- “## Check it’s daytime (in their timezone) … only calls within the 8am-5pm window.”

---

### Block 4 — Call Placement (VAPI)

**Overview:** Places an outbound call via VAPI’s `/call` endpoint using a specific VAPI assistant and a VAPI phone number identity.

**Nodes involved:**
- **Make phone call** (HTTP Request)

#### Node: Make phone call
- **Type / role:** `httpRequest` — Creates a call in VAPI.
- **Key configuration choices:**
  - **Method:** POST
  - **URL:** `https://api.vapi.ai/call`
  - **Body:** JSON (currently contains placeholders):
    - `assistantId`: “REPLACE WITH YOUR ASSISTANT ID”
    - `phoneNumberId`: “REPLACE WITH YOUR PHONE NUMBER ID”
    - `customer.number`: “REPLACE WITH THE NUMBER YOU WANT TO CALL e.g. +1234567890”
  - **Auth:** Generic credential type → **HTTP Bearer Auth** (VAPI API key expected).
- **Inputs/Outputs:**
  - **Input ←** `Is it between 8am-5pm for them?` (true branch)
  - **Output →** `Wait 1 minute`
- **Important expression to implement (per sticky note guidance):**
  - Set `customer.number` to: `{{ $json['Phone Number'] }}`
  - Note: VAPI often expects E.164 (`+` prefixed). This workflow cleans to digits only; you may need to re-add `+` and ensure country code is present.
- **Edge cases / failures:**
  - 401/403 if Bearer token is missing/invalid.
  - 400 if phone number format is unacceptable or IDs are wrong.
  - If VAPI returns an error response without an `id`, downstream polling (`Make phone call`.json.id) will break.

**Sticky note covering this block:**
- “## Make Phone Call … Create a free VAPI account … (link)”

---

### Block 5 — Call Completion Polling Loop

**Overview:** After initiating a call, the workflow waits one minute, checks the call status from VAPI, and loops until the status is `ended`.

**Nodes involved:**
- **Wait 1 minute** (Wait)
- **Get Call Status** (HTTP Request)
- **Call has finished** (IF)

#### Node: Wait 1 minute
- **Type / role:** `wait` — Poll interval throttle.
- **Key configuration choices:** Wait **1 minute**.
- **Inputs/Outputs:**
  - **Input ←** `Make phone call` or `Call has finished` (false branch loop)
  - **Output →** `Get Call Status`
- **Edge cases / failures:**
  - Many concurrent calls lead to many concurrent executions waiting.

#### Node: Get Call Status
- **Type / role:** `httpRequest` — Fetch call details/status from VAPI.
- **Key configuration choices:**
  - **URL expression:**
    - `https://api.vapi.ai/call/{{ $node['Make phone call'].json.id }}`
  - **Response:** “Full response” enabled (the node returns `statusCode`, headers, and `body`)
  - **Authentication:** Set to “predefinedCredentialType” in JSON, but header parameters appear empty; in practice you must ensure Bearer auth is applied (either via credentials or explicit header).
- **Inputs/Outputs:**
  - **Input ←** `Wait 1 minute`
  - **Output →** `Call has finished`
- **Edge cases / failures:**
  - If `Make phone call` didn’t return `id`, URL becomes invalid → runtime error.
  - Auth misconfiguration will cause 401 and infinite looping unless handled.
  - API rate limits: polling every minute per call is usually fine, but at scale can hit limits.

#### Node: Call has finished
- **Type / role:** `if` — Determines whether to exit polling loop.
- **Key configuration choices:**
  - Checks: `={{ $json.body.status }}` equals `"ended"`
- **Branching:**
  - **True →** `Update Google Sheets with call information`
  - **False →** `Wait 1 minute` (loop)
- **Inputs/Outputs:**
  - **Input ←** `Get Call Status`
  - **Outputs →** `Update Google Sheets with call information` (true), `Wait 1 minute` (false)
- **Edge cases / failures:**
  - If VAPI returns a different terminal status (e.g., `completed`, `failed`, `canceled`), the workflow will loop forever. Consider expanding conditions to handle other terminal states.

**Sticky note covering this block:**
- “## Wait for the call to finish … polls Vapi until the call has finished.”

---

### Block 6 — Update Google Sheets with Call Information

**Overview:** Writes call results back into the sheet. In the provided workflow, the node is configured for an update operation but does not define the actual column mappings.

**Nodes involved:**
- **Update Google Sheets with call information** (Google Sheets)

#### Node: Update Google Sheets with call information
- **Type / role:** `googleSheets` — Updates an existing row in a sheet.
- **Key configuration choices:**
  - **Operation:** Update
  - **Sheet:** `gid=0` (first sheet tab), document ID is empty in export (must be configured)
  - **Columns mapping:** `mappingMode: defineBelow` but the mapping object is empty in the JSON → you must define:
    - Which row to update (usually via row ID from trigger)
    - Which columns to set (Status, transcript, summary, etc.)
- **Inputs/Outputs:**
  - **Input ←** `Call has finished` (true branch)
  - **Output →** none (end)
- **Credentials:** Google Sheets OAuth2 (required).
- **Edge cases / failures:**
  - Without a row identifier / matching column configuration, update may fail or update the wrong row.
  - If the trigger does not provide a stable row ID, you must use a unique key column to match.
  - Concurrency: two executions updating the same row can race (uncommon for “rowAdded” but possible if duplicates).

**Sticky note covering this block:**
- “## Update Google Sheet”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / annotation | — | — | ## Make Phone Call; **⚙️ Setup:**; **Create a free VAPI account** [VAPI – $10 FREE credits](https://vapi.ai/?aff=n8ntemplate) |
| Sticky Note1 | Sticky Note | Documentation / annotation | — | — | ## When a lead arrives |
| Sticky Note2 | Sticky Note | Documentation / annotation | — | — | ## Santise the phone number; Phone numbers are messy… strip… filter out any numbers we've already called… |
| Make phone call | HTTP Request | Create outbound call in VAPI | Is it between 8am-5pm for them? (true) | Wait 1 minute | ## Make Phone Call; **⚙️ Setup:**; **Create a free VAPI account** [VAPI – $10 FREE credits](https://vapi.ai/?aff=n8ntemplate) |
| Sticky Note3 | Sticky Note | Documentation / annotation | — | — | ## Wait for the call to finish; This poll's Vapi until the call has finished. |
| Sticky Note4 | Sticky Note | Documentation / annotation | — | — | ## Check it's daytime (in their timezone); checks the lead's timezone… only calls within the 8am-5pm window. |
| Get lead's timezone | Code | Infer timezone + compute local time fields | Filter numbers we've already called; Wait | Is it between 8am-5pm for them? | ## Check it's daytime (in their timezone); checks the lead's timezone… only calls within the 8am-5pm window. |
| Cleanse phone numbers | Code | Remove non-digits from Phone Number | New lead | Filter numbers we've already called | ## Santise the phone number; Phone numbers are messy… strip… filter out any numbers we've already called… |
| New lead | Google Sheets Trigger | Start workflow on new row | — | Cleanse phone numbers | ## When a lead arrives |
| Is it between 8am-5pm for them? | IF | Business hours gate | Get lead's timezone | Make phone call (true); Wait (false) | ## Check it's daytime (in their timezone); checks the lead's timezone… only calls within the 8am-5pm window. |
| Wait | Wait | Delay 2 hours then retry business-hours check | Is it between 8am-5pm for them? (false) | Get lead's timezone | ## Check it's daytime (in their timezone); checks the lead's timezone… only calls within the 8am-5pm window. |
| Wait 1 minute | Wait | Poll interval delay | Make phone call; Call has finished (false) | Get Call Status | ## Wait for the call to finish; This poll's Vapi until the call has finished. |
| Call has finished | IF | Exit condition for polling loop | Get Call Status | Update Google Sheets with call information (true); Wait 1 minute (false) | ## Wait for the call to finish; This poll's Vapi until the call has finished. |
| Get Call Status | HTTP Request | Fetch VAPI call status/details | Wait 1 minute | Call has finished | ## Wait for the call to finish; This poll's Vapi until the call has finished. |
| Sticky Note5 | Sticky Note | Documentation / annotation | — | — | ## Update Google Sheet |
| Filter numbers we've already called | Filter | Only allow eligible leads (Status=Not Called) | Cleanse phone numbers | Get lead's timezone | ## Santise the phone number; Phone numbers are messy… strip… filter out any numbers we've already called… |
| Sticky Note7 | Sticky Note | Workflow description / setup notes | — | — | ## 📞 Automated Outbound Lead Caller; Uses [VAPI](https://vapi.ai/?aff=n8ntemplate)… Setup steps; Built by Marcus Taylor; [voiceai.guide](https://voiceai.guide) |
| Update Google Sheets with call information | Google Sheets | Write call results back to the sheet | Call has finished (true) | — | ## Update Google Sheet |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add Google Sheets Trigger node**
   - Node: **Google Sheets Trigger**
   - Name: **New lead**
   - Event: **Row Added**
   - Poll interval: **Every minute**
   - Select **Document** (Spreadsheet) and **Sheet**
   - Credentials: create/select **Google Sheets Trigger OAuth2** credential.

3. **Add Code node to sanitize phone number**
   - Node: **Code**
   - Name: **Cleanse phone numbers**
   - Paste logic equivalent to: remove all non-digits from `Phone Number` for each item.
   - Connect: `New lead` → `Cleanse phone numbers`.

4. **Add Filter node to only process new/uncalled leads**
   - Node: **Filter**
   - Name: **Filter numbers we've already called**
   - Conditions (AND):
     - `Phone Number` exists (and ideally “is not empty” if you add it)
     - `Status` equals **Not Called**
   - Connect: `Cleanse phone numbers` → `Filter numbers we've already called`.

5. **Add Code node to infer timezone and compute local time**
   - Node: **Code**
   - Name: **Get lead's timezone**
   - Implement:
     - A country code → timezone map
     - Longest-prefix matching for country codes
     - Compute `localHour` in that timezone using `Intl.DateTimeFormat`
     - Output merged JSON including `timezone`, `localHour`, etc.
   - Connect: `Filter numbers we've already called` → `Get lead's timezone`.

6. **Add IF node for business-hours check**
   - Node: **IF**
   - Name: **Is it between 8am-5pm for them?**
   - Conditions (AND):
     - `{{$json.localHour}} >= 8`
     - `{{$json.localHour}} <= 17`
   - Connect: `Get lead's timezone` → `Is it between 8am-5pm for them?`.

7. **Add Wait node for outside-hours retry**
   - Node: **Wait**
   - Name: **Wait**
   - Mode: time interval
   - Amount: **2 hours**
   - Connect: `Is it between 8am-5pm for them?` **false** → `Wait`
   - Connect: `Wait` → `Get lead's timezone` (loop).

8. **Add HTTP Request node to place the call via VAPI**
   - Node: **HTTP Request**
   - Name: **Make phone call**
   - Method: **POST**
   - URL: `https://api.vapi.ai/call`
   - Authentication: **Bearer Auth** (create a credential with your VAPI API key)
   - Body: JSON including:
     - `assistantId`: your VAPI assistant ID
     - `phoneNumberId`: your VAPI phone number ID
     - `customer.number`: set to the lead number from the sheet
       - Recommended: ensure E.164 format; if your sheet lacks `+`, add it consistently.
   - Connect: IF **true** → `Make phone call`.

9. **Add Wait node for polling interval**
   - Node: **Wait**
   - Name: **Wait 1 minute**
   - Amount: **1 minute**
   - Connect: `Make phone call` → `Wait 1 minute`.

10. **Add HTTP Request node to fetch call status**
   - Node: **HTTP Request**
   - Name: **Get Call Status**
   - Method: **GET**
   - URL: `https://api.vapi.ai/call/{{ $node["Make phone call"].json.id }}`
   - Ensure the same VAPI Bearer Auth is applied.
   - Enable “Full response” if you want to check `body.status` exactly as in the workflow.
   - Connect: `Wait 1 minute` → `Get Call Status`.

11. **Add IF node to test completion**
   - Node: **IF**
   - Name: **Call has finished**
   - Condition: `{{$json.body.status}}` equals `ended`
   - Connect: `Get Call Status` → `Call has finished`
   - Connect: IF **false** → `Wait 1 minute` (loop)
   - Connect: IF **true** → Google Sheets update node.

12. **Add Google Sheets node to update the row**
   - Node: **Google Sheets**
   - Name: **Update Google Sheets with call information**
   - Operation: **Update**
   - Select Document and Sheet (same as trigger)
   - Define how to find the row to update (common approaches):
     - Use the row number / row ID provided by the trigger (preferred if available)
     - Or match on a unique column (e.g., a Lead ID)
   - Map columns to update (minimum suggested):
     - `Status` → “Called”
     - Optionally write `Transcript`, `Summary`, `Sentiment`, `Call Id`, etc. (depending on what VAPI returns in the status payload)
   - Credentials: **Google Sheets OAuth2** credential.
   - Connect: `Call has finished` **true** → `Update Google Sheets with call information`.

13. **(Optional) Add Sticky Notes**
   - Add annotations for setup links, calling window logic, and required sheet columns as in the original.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| VAPI account / credits link: “VAPI – $10 FREE credits” | https://vapi.ai/?aff=n8ntemplate |
| Workflow concept: automated outbound lead caller with timezone-aware business-hours gating | Sticky note “📞 Automated Outbound Lead Caller” |
| Required Google Sheet columns mentioned: `Phone Number`, `Name`, `Status` (new leads set to “Not called/Not Called”) | Sticky note “📞 Automated Outbound Lead Caller” |
| Author credit: Built by Marcus Taylor; “@intellagents”; website link | https://voiceai.guide |

