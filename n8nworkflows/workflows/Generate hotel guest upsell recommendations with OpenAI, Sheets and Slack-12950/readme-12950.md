Generate hotel guest upsell recommendations with OpenAI, Sheets and Slack

https://n8nworkflows.xyz/workflows/generate-hotel-guest-upsell-recommendations-with-openai--sheets-and-slack-12950


# Generate hotel guest upsell recommendations with OpenAI, Sheets and Slack

## 1. Workflow Overview

**Purpose:** Automate daily, AI-generated upsell recommendations for hotel guests using **Google Sheets** as the guest database, **OpenAI (GPT-4o-mini)** for selecting the best single upsell offer, and **Slack** for notifying the team. It also includes an **error-monitoring path** that alerts Slack if the workflow fails.

**Primary use cases**
- Daily pre-arrival and in-stay upsell targeting
- Centralized tracking of the recommended upsell type in a Sheet
- Team visibility via Slack notifications

### 1.1 Scheduling & Guest Data Retrieval
Runs every day at 9:00 AM, reads all guest rows from Google Sheets.

### 1.2 Guest Filtering (Stay Status Routing)
Splits each guest record into two paths based on **Stay Status**:
- `upcoming` → before arrival
- `checked_in` → during stay

### 1.3 Guest Context Preparation
Normalizes guest fields into a consistent JSON structure for the AI prompt (name, room type, spend level, preferences, etc.), and labels the stay phase.

### 1.4 AI Upsell Generation & Parsing
Calls OpenAI to generate **one** upsell recommendation in strict JSON, then parses the response and normalizes outputs (`upsell_type`, `offer_message`, `reason`).

### 1.5 Persist & Notify
Updates the guest’s row in Google Sheets with the `upsell_type`, then posts a formatted Slack message to a channel.

### 1.6 Error Monitoring
A separate Error Trigger catches failures anywhere in the workflow and posts a Slack alert.

---

## 2. Block-by-Block Analysis

### Block A — Documentation, Setup Notes (Non-executable)
**Overview:** Sticky notes describing purpose, setup, and credential requirements. No runtime behavior.  
**Nodes involved:** `Note`, `Note1`, `Note2`, `Note3`, `Note4`, `Note5`, `Sticky Note6`

#### Node: Note (Sticky Note)
- **Type/role:** Sticky Note (documentation)
- **Configuration:** Explains workflow purpose, setup steps, required sheet columns, testing advice
- **Connections:** None
- **Edge cases:** None

#### Node: Note1 (Sticky Note)
- **Type/role:** Sticky Note
- **Configuration:** Describes data collection & filtering block
- **Connections:** None

#### Node: Note2 (Sticky Note)
- **Type/role:** Sticky Note
- **Configuration:** Describes guest context preparation
- **Connections:** None

#### Node: Note3 (Sticky Note)
- **Type/role:** Sticky Note
- **Configuration:** Describes AI upsell generation and JSON parsing
- **Connections:** None

#### Node: Note4 (Sticky Note)
- **Type/role:** Sticky Note
- **Configuration:** Describes updating Sheets and notifying Slack
- **Connections:** None

#### Node: Note5 (Sticky Note)
- **Type/role:** Sticky Note
- **Configuration:** Lists required credentials: Google Sheets OAuth2, OpenAI API key, Slack token scopes
- **Connections:** None

#### Node: Sticky Note6 (Sticky Note)
- **Type/role:** Sticky Note
- **Configuration:** Explains error monitoring and Slack alerting
- **Connections:** None

---

### Block B — Error Monitoring & Alerting
**Overview:** If the workflow errors at runtime, it triggers an alert message to a Slack channel.  
**Nodes involved:** `Error Trigger`, `Alert on Workflow Failure`

#### Node: Error Trigger
- **Type/role:** `n8n-nodes-base.errorTrigger` — workflow error entry point
- **Configuration choices:** No parameters; triggers on workflow failures
- **Input/Output:** No inputs; outputs to Slack node
- **Failure types:** It triggers *because* of upstream failures; the node itself rarely “fails” unless workflow environment is broken.

#### Node: Alert on Workflow Failure
- **Type/role:** `n8n-nodes-base.slack` — posts a message to Slack
- **Configuration choices:**
  - Operation: send message to a **channel**
  - Channel: `workflow-errors` (channelId `C09GNB90TED`)
  - Text: “⚠️ Hotel Pre-Arrival Workflow Error Detected…”
- **Credentials:** Slack API credential (“Slack account vivek”)
- **Connections:** Input from `Error Trigger`
- **Failure types / edge cases:**
  - Slack auth/token revoked
  - Missing `chat:write` scope
  - Channel not accessible to the Slack app
  - Slack API rate limits

---

### Block C — Scheduling & Reading Guests from Google Sheets
**Overview:** Runs daily at 9AM and loads guest rows from the “guest” sheet.  
**Nodes involved:** `Schedule Trigger1`, `Google Sheets - Read Guests1`

#### Node: Schedule Trigger1
- **Type/role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point
- **Configuration choices:** Cron expression `0 9 * * *` (daily at 09:00 server time)
- **Output:** Triggers a single execution into the Sheets read node
- **Edge cases:**
  - Timezone depends on n8n instance settings
  - Missed runs if instance is down at scheduled time

#### Node: Google Sheets - Read Guests1
- **Type/role:** `n8n-nodes-base.googleSheets` — read data from Google Sheets
- **Configuration choices (interpreted):**
  - Document: `sample_leads_50` (spreadsheet ID `17rcNd_Z...`)
  - Sheet/tab: `guest` (gid `1525755670`)
  - Operation is implied as **Read/Get Many** (not explicitly shown in JSON, but it’s a read node)
- **Credentials:** Google Sheets OAuth2 (“automations@techdome.ai”)
- **Output connections:** Sends all rows to both IF nodes (`IF - Before Arrival1` and `IF - During Stay1`)
- **Failure types / edge cases:**
  - OAuth token expiration / revoked access
  - Spreadsheet or sheet renamed
  - Header mismatch (column names referenced later must match exactly)
  - Large sheets may cause performance issues depending on n8n limits

---

### Block D — Filtering by Stay Status
**Overview:** Routes each guest row into the appropriate processing branch by checking the `Stay Status` column.  
**Nodes involved:** `IF - Before Arrival1`, `IF - During Stay1`

#### Node: IF - Before Arrival1
- **Type/role:** `n8n-nodes-base.if` — conditional routing
- **Condition:** `{{ $json['Stay Status'] }}` equals `"upcoming"`
- **Output connections:**
  - True path → `Set - Guest Context (Before)1`
  - False path → not connected (guests not matching are dropped for this branch)
- **Edge cases:**
  - Case sensitivity: strict equality; `Upcoming` or `UPCOMING` will not match
  - Missing column yields `undefined` and condition fails

#### Node: IF - During Stay1
- **Type/role:** IF node
- **Condition:** `{{ $json['Stay Status'] }}` equals `"checked_in"`
- **Output connections:**
  - True path → `Set - Guest Context (During)1`
- **Edge cases:** Same as above

---

### Block E — Guest Context Preparation (Normalization)
**Overview:** Creates a clean JSON payload for AI with consistent field names and adds stay-phase context.  
**Nodes involved:** `Set - Guest Context (Before)1`, `Set - Guest Context (During)1`

#### Node: Set - Guest Context (Before)1
- **Type/role:** `n8n-nodes-base.set` — maps sheet columns into prompt-ready fields
- **Key mappings:**
  - `guest_name` ← `Guest Name`
  - `email` ← `Email`
  - `room_type` ← `Room Type`
  - `repeat_guest` ← `Repeat Guest`
  - `spend_level` ← `Spend Level`
  - `preferences` ← `Preferences`
  - `occasion` ← `Special Occasion`
  - `stay_phase` ← `"before_arrival"`
- **Output:** To `AI - Generate Upsell1`
- **Important edge case (functional bug):**
  - **Does not set `row_number`**, but later the workflow attempts to update a row in Google Sheets using `row_number`. This means “upcoming” guests may not be written back to Sheets (or update step may fail / update wrong row depending on how the update node is configured).

#### Node: Set - Guest Context (During)1
- **Type/role:** Set node
- **Key mappings:** Same as above, plus:
  - `stay_phase` ← `"during_stay"`
  - `row_number` ← `{{ $json['__rowNumber'] }}`
- **Output:** To `AI - Generate Upsell1`
- **Edge cases:**
  - `__rowNumber` is provided by n8n’s Google Sheets read output depending on operation/node version and settings; if not present, updates will fail.

---

### Block F — AI Upsell Generation (OpenAI) & Response Parsing
**Overview:** Generates one upsell recommendation via OpenAI, requiring strict JSON output, then parses it and normalizes fields for downstream update + Slack notification.  
**Nodes involved:** `AI - Generate Upsell1`, `Code - Parse AI Response1`

#### Node: AI - Generate Upsell1
- **Type/role:** `@n8n/n8n-nodes-langchain.openAi` — chat completion via OpenAI
- **Model:** `gpt-4o-mini`
- **Options:** `maxTokens: 300`, `temperature: 0.7`
- **Prompt structure:**
  - **System message:** must return *only* valid JSON with keys: `upsell_type`, `offer_message`, `reason`
  - **User message:** guest profile interpolated from `$json.*` fields, with rules based on `stay_phase`
- **Input:** From either Set node (before/during)
- **Output:** To parsing code node
- **Failure types / edge cases:**
  - OpenAI credential invalid / quota exceeded
  - Model unavailable or renamed
  - Non-JSON output despite instruction (handled later by parser)
  - Latency/timeouts during high load

#### Node: Code - Parse AI Response1
- **Type/role:** `n8n-nodes-base.code` — parses AI output into structured fields
- **Logic (interpreted):**
  - Reads AI response from `item.json.message?.content` or `item.json.text`
  - Strips ```json fences if present
  - `JSON.parse` the content
  - On parse failure: sets defaults (`upsell_type: 'Error'`, etc.)
  - Outputs a normalized object including:
    - `guest_name`, `email`, `room_type`, `stay_phase`, `row_number`
    - `upsell_type`, `offer_message`, `reason`
- **Connections:** Output → `Google Sheets - Update Row1`
- **Edge cases:**
  - If OpenAI node output schema changes (e.g., content not found at `message.content`), parser may always fall back to empty string and thus parse-fail
  - `row_number` may be `undefined` for the before-arrival branch (see earlier bug), causing update issues
  - If AI returns JSON with missing keys, code defaults some fields but not all semantics

---

### Block G — Update Google Sheets & Notify Slack
**Overview:** Writes `upsell_type` back to the guest’s row and posts a Slack message containing upsell details and reasoning.  
**Nodes involved:** `Google Sheets - Update Row1`, `Slack - Notify Team1`

#### Node: Google Sheets - Update Row1
- **Type/role:** Google Sheets node — **Update** operation
- **Configuration choices (interpreted):**
  - Operation: `update`
  - Matches row using `matchingColumns: ["row_number"]`
  - Writes:
    - `upsell_type` from `{{ $json.upsell_type }}`
    - `row_number` expression currently set to `={{ $('Google Sheets - Read Guests1').item.json.row_number }}`
- **Credentials:** Google Sheets OAuth2 (“automations@techdome.ai”)
- **Input:** From `Code - Parse AI Response1`
- **Output:** To Slack notify node
- **Critical edge cases / likely misconfiguration:**
  1. **Row number source mismatch**
     - The Sheet read items typically expose `__rowNumber`, and the “during stay” Set stores it as `row_number`.
     - But this update node references `$('Google Sheets - Read Guests1').item.json.row_number` (which likely does not exist).
     - Correct approach is usually `{{ $json.row_number }}` (coming from the parser/code node), or map `row_number` from `__rowNumber` earlier for both branches.
  2. **Before-arrival path has no row_number**
     - Updates for “upcoming” guests will likely fail or not update any row.
- **Failure types:**
  - OAuth issues, sheet not writable
  - “No matching row found” if row_number missing/wrong
  - Type mismatch if row_number is string vs number (depends on sheet node settings)

#### Node: Slack - Notify Team1
- **Type/role:** Slack node — send channel message
- **Configuration choices:**
  - Channel: `general-information` (channelId `C09GNB90TED`)
  - Message uses expressions:
    - Guest name from `$('Google Sheets - Read Guests1').item.json['Guest Name']`
    - Upsell type from current `$json.upsell_type`
    - Offer message & reason from `$('Code - Parse AI Response1').item.json.*`
- **Input:** From `Google Sheets - Update Row1`
- **Edge cases / pairing considerations:**
  - Cross-node item referencing (`$('Google Sheets - Read Guests1').item...`) can break if item pairing is lost or if multiple items are processed and the referenced node item index doesn’t match.
  - Prefer using the current item fields (e.g., `$json.guest_name`, `$json.offer_message`, `$json.reason`) after parsing to avoid mismatch.
  - Slack auth/scope/rate limit issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note6 | stickyNote | Documentation (error monitoring) |  |  | ## ⚠️ Error Monitoring\n\nCatches any workflow failures and sends alerts to Slack's general-information channel. Helps maintain reliability and enables quick troubleshooting. |
| Error Trigger | errorTrigger | Error entry point |  | Alert on Workflow Failure | ## ⚠️ Error Monitoring\n\nCatches any workflow failures and sends alerts to Slack's general-information channel. Helps maintain reliability and enables quick troubleshooting. |
| Alert on Workflow Failure | slack | Post error alert to Slack | Error Trigger |  | ## ⚠️ Error Monitoring\n\nCatches any workflow failures and sends alerts to Slack's general-information channel. Helps maintain reliability and enables quick troubleshooting. |
| Note | stickyNote | Documentation (overall description & setup) |  |  | ## 🏨 Automate Hotel Guest Upsells with AI using OpenAI, Sheets & Slack\n\n### How it works\nThis workflow reads guest data from Google Sheets daily at 9 AM, categorizes guests by stay status (before arrival or currently checked in), and uses AI to generate personalized upsell recommendations. The system suggests room upgrades or airport pickup for upcoming guests, and spa, dining, or experiences for current guests. Results are written back to the spreadsheet and posted to Slack for team visibility.\n\n### Setup steps\n1. Connect Google Sheets OAuth2 credentials and point to your guest spreadsheet\n2. Add OpenAI API credentials for GPT-4o-mini\n3. Configure Slack credentials and select your notification channel\n4. Ensure your spreadsheet has columns: Guest Name, Email, Stay Status, Room Type, Repeat Guest, Spend Level, Preferences, Special Occasion, upsell_type\n5. Test with a manual trigger before activating the daily schedule |
| Note1 | stickyNote | Documentation (data collection/filtering) |  |  | ## 📥 Data Collection & Filtering\n\nRetrieves all guest records from Google Sheets and splits them into two paths based on stay status: guests arriving soon vs. currently checked in. |
| Note2 | stickyNote | Documentation (context preparation) |  |  | ## 🎯 Guest Context Preparation\n\nExtracts relevant guest attributes (name, room type, preferences, spending level, occasion) and adds stay phase context for AI processing. |
| Note3 | stickyNote | Documentation (AI generation/parsing) |  |  | ## 🤖 AI Upsell Generation\n\nUses OpenAI to analyze guest profiles and recommend the single best upsell opportunity. Parses JSON response to extract offer type, personalized message, and reasoning. |
| Note4 | stickyNote | Documentation (update/notify) |  |  | ## 💾 Update & Notify\n\nWrites the AI-generated upsell type back to the spreadsheet row and posts a formatted notification to Slack with guest details and offer reasoning. |
| Note5 | stickyNote | Documentation (credentials) |  |  | ## 🔐 Credentials Required\n\n**Google Sheets:** OAuth2 with read/write permissions  \n**OpenAI:** API key with GPT-4o-mini access  \n**Slack:** App token with chat:write scope\n\nReplace all credential IDs with your own before deploying. |
| Schedule Trigger1 | scheduleTrigger | Daily trigger |  | Google Sheets - Read Guests1 |  |
| Google Sheets - Read Guests1 | googleSheets | Read guest rows | Schedule Trigger1 | IF - Before Arrival1; IF - During Stay1 |  |
| IF - Before Arrival1 | if | Route upcoming guests | Google Sheets - Read Guests1 | Set - Guest Context (Before)1 |  |
| IF - During Stay1 | if | Route checked-in guests | Google Sheets - Read Guests1 | Set - Guest Context (During)1 |  |
| Set - Guest Context (Before)1 | set | Normalize fields (before arrival) | IF - Before Arrival1 | AI - Generate Upsell1 |  |
| Set - Guest Context (During)1 | set | Normalize fields (during stay) + row number | IF - During Stay1 | AI - Generate Upsell1 |  |
| AI - Generate Upsell1 | openAi (langchain) | Generate single upsell recommendation | Set - Guest Context (Before)1; Set - Guest Context (During)1 | Code - Parse AI Response1 |  |
| Code - Parse AI Response1 | code | Parse/normalize AI JSON response | AI - Generate Upsell1 | Google Sheets - Update Row1 |  |
| Google Sheets - Update Row1 | googleSheets | Update row with upsell_type | Code - Parse AI Response1 | Slack - Notify Team1 |  |
| Slack - Notify Team1 | slack | Notify team in Slack | Google Sheets - Update Row1 |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set a **Cron** rule: `0 9 * * *` (daily at 09:00).
3. **Add Google Sheets (Read)**
   - Node: **Google Sheets**
   - Credentials: **Google Sheets OAuth2** (read/write)
   - Select your **Spreadsheet** and **Sheet** (e.g., `guest`)
   - Configure to read all rows (typical “Get Many” behavior).
4. **Add two IF nodes for routing**
   - Node A: **IF - Before Arrival**
     - Condition: `{{ $json['Stay Status'] }}` equals `upcoming`
   - Node B: **IF - During Stay**
     - Condition: `{{ $json['Stay Status'] }}` equals `checked_in`
   - Connect: `Google Sheets (Read)` → both IF nodes.
5. **Add Set node for “before arrival” context**
   - Node: **Set**
   - Create fields:
     - `guest_name` = `{{ $json['Guest Name'] }}`
     - `email` = `{{ $json['Email'] }}`
     - `room_type` = `{{ $json['Room Type'] }}`
     - `repeat_guest` = `{{ $json['Repeat Guest'] }}`
     - `spend_level` = `{{ $json['Spend Level'] }}`
     - `preferences` = `{{ $json['Preferences'] }}`
     - `occasion` = `{{ $json['Special Occasion'] }}`
     - `stay_phase` = `before_arrival`
     - **Recommended fix:** also add `row_number` = `{{ $json['__rowNumber'] }}`
   - Connect: IF (true) → this Set node.
6. **Add Set node for “during stay” context**
   - Same fields as above
   - `stay_phase` = `during_stay`
   - `row_number` = `{{ $json['__rowNumber'] }}`
   - Connect: IF (true) → this Set node.
7. **Add OpenAI node**
   - Node: **OpenAI (LangChain)**
   - Credentials: OpenAI API key
   - Model: `gpt-4o-mini`
   - Temperature: `0.7`, Max tokens: `300`
   - Messages:
     - System: enforce “JSON only” with keys `upsell_type`, `offer_message`, `reason`
     - User: include guest profile fields and rules based on `stay_phase`
   - Connect: both Set nodes → OpenAI node.
8. **Add Code node to parse response**
   - Node: **Code**
   - Paste logic similar to the provided parser (strip code fences, JSON.parse, fallback on error).
   - Ensure output includes: `row_number`, `upsell_type`, `offer_message`, `reason`, plus guest identifiers.
   - Connect: OpenAI → Code.
9. **Add Google Sheets (Update)**
   - Node: **Google Sheets**
   - Operation: **Update**
   - Match on: `row_number`
   - Set fields to update:
     - `row_number` = `{{ $json.row_number }}`
     - `upsell_type` = `{{ $json.upsell_type }}`
   - Connect: Code → Sheets Update.
10. **Add Slack notify node**
   - Node: **Slack**
   - Credentials: Slack API token with `chat:write`
   - Choose channel (e.g., `general-information`)
   - Message text (recommended to avoid cross-node references):
     - Guest: `{{ $json.guest_name }}`
     - Upsell Type: `{{ $json.upsell_type }}`
     - Offer Message: `{{ $json.offer_message }}`
     - Reason: `{{ $json.reason }}`
   - Connect: Sheets Update → Slack notify.
11. **Add Error Trigger + Slack alert**
   - Node: **Error Trigger**
   - Node: **Slack** (send message to an error channel)
   - Connect: Error Trigger → Slack error alert node.
12. **Credentials checklist**
   - Google Sheets OAuth2: access to the spreadsheet, read/write
   - OpenAI API: permitted to use `gpt-4o-mini`
   - Slack: bot token installed in workspace, channel access, `chat:write`
13. **Test**
   - Run once manually (or add a Manual Trigger temporarily).
   - Verify:
     - Guests route correctly by `Stay Status`
     - AI response parses to JSON
     - Row updates occur for both stay phases
     - Slack posts match the correct guest (no item pairing mismatch)
14. **Activate workflow** after validation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Spreadsheet must include columns: Guest Name, Email, Stay Status, Room Type, Repeat Guest, Spend Level, Preferences, Special Occasion, upsell_type | Mentioned in the workflow documentation sticky note |
| Credentials required: Google Sheets OAuth2 (read/write), OpenAI API key (GPT-4o-mini), Slack token with chat:write | Mentioned in “Credentials Required” sticky note |
| Reliability feature: error monitoring posts failures to Slack | Mentioned in “Error Monitoring” sticky note |
| Implementation warning: ensure `row_number` is present for *both* branches and Sheets Update uses `{{ $json.row_number }}` | Derived from the current node configurations and connections |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.