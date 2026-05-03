Monitor Google reviews and draft AI responses with Gemini and Slack

https://n8nworkflows.xyz/workflows/monitor-google-reviews-and-draft-ai-responses-with-gemini-and-slack-13562


# Monitor Google reviews and draft AI responses with Gemini and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor a Google Business’s latest Google Reviews on a schedule, use **Google Gemini** to (a) score sentiment and (b) draft a suggested reply, **log everything to Google Sheets**, send **Slack alerts for negative reviews**, and email a **daily summary** to the owner.

**Typical use cases**
- Small business reputation monitoring (near real-time alerts for bad reviews).
- Centralized review log + AI-assisted reply drafting.
- Daily digest for owners/managers.

### 1.1 Trigger & Configuration Seed
Runs hourly and sets key variables (Place ID, Sheet ID, Slack channel, etc.) used by downstream nodes.

### 1.2 Fetch Reviews from Google Places API
Calls Google Maps Places Details API to retrieve the latest reviews for the configured `place_id`.

### 1.3 Detect New Reviews + Update “last check”
Filters reviews newer than the last stored timestamp and updates the timestamp for subsequent runs.

### 1.4 AI Sentiment + Draft Response (Gemini)
For each new review, asks Gemini to produce a sentiment score and a response draft, then parses the model output.

### 1.5 Persistence, Alerts, and Summary
Appends/updates rows in Google Sheets; routes negative reviews to Slack; sends a daily summary email (note: current wiring sends email after Slack step, not time-based).

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Setup Variables
**Overview:** Starts the workflow on a schedule and defines configuration values used throughout the run.  
**Nodes involved:** `Review Check Schedule`, `Setup Variables`

#### Node: Review Check Schedule
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Config (interpreted):**
  - Cron: `0 * * * *` (at minute 0 every hour).
  - TypeVersion: **1.2**
- **Outputs:** To `Setup Variables`.
- **Failure/edge cases:**
  - Timezone depends on n8n instance settings; “hourly” may not align with business timezone unless configured globally.

#### Node: Setup Variables
- **Type / role:** Set node (`n8n-nodes-base.set`) — provides constants/state.
- **Config (interpreted):** Assigns:
  - `place_id` (string) — Google Place ID
  - `business_name` (string)
  - `sheet_id` (string) — Google Spreadsheet ID
  - `slack_channel_id` (string)
  - `owner_email` (string)
  - `last_check_timestamp` (number) — initialized to `0`
- **Key behavior note (important):**
  - This `last_check_timestamp` is **not persisted between executions**. Because it’s set to `0` every run, the workflow will treat *all* reviews as “new” on every hourly execution (unless you implement persistence via Data Store/Sheets/static data).
- **Outputs:** To `Get Google Reviews`.
- **Failure/edge cases:**
  - Mis-typed IDs will cascade into downstream auth/API failures.

---

### Block 2 — Fetch Reviews from Google Places API
**Overview:** Pulls place details including reviews from Google’s Places API.  
**Nodes involved:** `Get Google Reviews`

#### Node: Get Google Reviews
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Google API.
- **Config (interpreted):**
  - Method: default (GET).
  - URL: `https://maps.googleapis.com/maps/api/place/details/json`
  - Authentication: **HTTP Header Auth** via “genericCredentialType”.
- **Expressions/variables used:** None shown in the URL; critically, the request as provided does **not** include query parameters like `place_id`, `fields`, or `key`.
- **Inputs:** From `Setup Variables`.
- **Outputs:** To `Filter New Reviews`.
- **Version:** TypeVersion **4.2**
- **Failure/edge cases:**
  - **Likely functional gap:** Google Places Details requires `place_id` and either API key query param `key=...` (common) or another supported auth. Header-based auth may not work unless you’re using a proxy or custom gateway; Google Maps API typically uses `key` query param.
  - Missing `fields=reviews` (or `fields=reviews,rating,...`) may yield no reviews.
  - API quota limits, billing not enabled, invalid key → 4xx errors.
  - Response shape assumed later: `$json.result.reviews`.

---

### Block 3 — Filter New Reviews + Update Timestamp
**Overview:** Filters the API’s review list to only those newer than the last check time, and updates the timestamp for next run.  
**Nodes involved:** `Filter New Reviews`

#### Node: Filter New Reviews
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms items to one item per new review.
- **Config (interpreted):**
  - Reads reviews from: `$input.item.json.result.reviews || []`
  - Reads last check timestamp from: `$('Setup Variables').item.json.last_check_timestamp`
  - Sets `currentTime = Math.floor(Date.now()/1000)`
  - Filters: `review.time > lastCheckTime`
  - Mutates `setupVars.last_check_timestamp = currentTime` (but this does **not** persist outside current execution)
  - Returns: `newReviews.map(...)` producing items shaped like the original review + `place_id`, `business_name`.
- **Inputs:** From `Get Google Reviews` (primary), plus reads `Setup Variables` via node reference.
- **Outputs:** To `Sentiment Analysis & Draft Response`.
- **Version:** TypeVersion **2**
- **Failure/edge cases:**
  - If `result` or `reviews` is missing, outputs an empty list → downstream nodes won’t run.
  - `review.time` is assumed numeric epoch seconds; if absent or in ms, filtering will be wrong.
  - The “update last_check_timestamp” line does not carry state across workflow executions.

---

### Block 4 — AI Sentiment Analysis & Draft Response (Gemini)
**Overview:** For each new review item, Gemini generates a sentiment score and a response draft; then code parses the model output and enriches the review record.  
**Nodes involved:** `Google Gemini Chat Model`, `Sentiment Analysis & Draft Response`, `Parse AI Results`

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain chat model for Gemini (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — provides the LLM connection.
- **Config (interpreted):**
  - Uses default options (no model name/temperature shown).
- **Connections:**
  - Connected via `ai_languageModel` output to the chain node’s `ai_languageModel` input.
- **Version:** TypeVersion **1**
- **Failure/edge cases:**
  - Missing/invalid Gemini credentials → auth failure.
  - Model defaults may change behavior; for deterministic parsing, specify a stricter output format.

#### Node: Sentiment Analysis & Draft Response
- **Type / role:** LangChain Chain (`@n8n/n8n-nodes-langchain.chainLlm`) — orchestrates prompt + model call.
- **Config (interpreted):**
  - Parameters are empty in JSON, which usually means the node is incomplete or relies on defaults/UI-configured fields not represented here. However, it must output a field called `response` because the next node expects `$json.response`.
- **Inputs:**
  - Main input: items from `Filter New Reviews`
  - LLM input: `Google Gemini Chat Model` via `ai_languageModel`
- **Outputs:** To `Parse AI Results`.
- **Version:** TypeVersion **1.4**
- **Failure/edge cases:**
  - If the chain is not configured with a prompt, output may be empty/undefined.
  - Output format must match the parser expectations (lines with `SENTIMENT:` and `RESPONSE:`).

#### Node: Parse AI Results
- **Type / role:** Code (`n8n-nodes-base.code`) — parses AI output and merges with review data.
- **Config (interpreted):**
  - Reads AI text: `const aiResponse = $input.item.json.response;`
  - Splits by newline and extracts:
    - `SENTIMENT:` → integer sentimentScore
    - `RESPONSE:` → responseDraft (single line)
  - Pulls original review from: `$('Filter New Reviews').item.json`
  - Outputs merged object with:
    - `sentiment_score`, `ai_response_draft`, `is_negative: sentimentScore < 3`, `created_at: ISO string`
- **Inputs:** From `Sentiment Analysis & Draft Response`; references `Filter New Reviews`.
- **Outputs:** To `Log to Google Sheets`.
- **Version:** TypeVersion **2**
- **Failure/edge cases:**
  - If `$json.response` is undefined → `.split()` throws.
  - If Gemini returns multi-line response draft, only the line starting with `RESPONSE:` is captured; remaining lines are lost.
  - `parseInt` can yield `NaN`; logic will then mark `is_negative` as `false` (since `NaN < 3` is false).
  - Node reference `$('Filter New Reviews').item.json` can be unsafe if multiple items are processed; it may always reference the “current paired item” but can break if pairing is lost.

---

### Block 5 — Log, Filter, Slack Alert, Email Summary
**Overview:** Stores processed reviews in Sheets, alerts Slack for negatives, then emails a summary.  
**Nodes involved:** `Log to Google Sheets`, `Filter Negative Reviews`, `Send Slack Alert`, `Email Daily Summary`

#### Node: Log to Google Sheets
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — persistence layer.
- **Config (interpreted):**
  - Operation: **appendOrUpdate**
  - Document ID: from `Setup Variables.sheet_id`
  - Sheet name: `Reviews Log`
  - Columns mapped:
    - `text`, `rating`, `review_id` (= `time`), `created_at`, `author_name`, `business_name`, `sentiment_score`, `ai_response_draft`
- **Inputs:** From `Parse AI Results`; references `Setup Variables` for sheet id.
- **Outputs:** To `Filter Negative Reviews`.
- **Version:** TypeVersion **4.4**
- **Credential:** OAuth2 for Google Sheets.
- **Failure/edge cases:**
  - “appendOrUpdate” typically requires a matching key column configuration in the node UI; if not properly set, may behave like append-only or fail.
  - `review_id` uses `time` which is not guaranteed unique if multiple reviews share same timestamp.
  - Missing sheet/tab name `Reviews Log` → error.
  - OAuth scope/permissions issues.

#### Node: Filter Negative Reviews
- **Type / role:** Filter (`n8n-nodes-base.filter`) — routes negative reviews only.
- **Config (interpreted):**
  - Condition: `$json.is_negative` equals `true` (strict boolean validation).
- **Inputs:** From `Log to Google Sheets`.
- **Outputs:** To `Send Slack Alert` (only passing matching items).
- **Version:** TypeVersion **2**
- **Failure/edge cases:**
  - If `is_negative` missing or not boolean (e.g., string `"true"`), strict validation can drop items unexpectedly.

#### Node: Send Slack Alert
- **Type / role:** Slack (`n8n-nodes-base.slack`) — sends immediate alert for negative reviews.
- **Config (interpreted):**
  - Posts a formatted message including rating, business, author, review text, AI draft, sentiment score.
  - Channel selected by ID from `Setup Variables.slack_channel_id`
  - Auth: OAuth2
- **Inputs:** From `Filter Negative Reviews`.
- **Outputs:** To `Email Daily Summary`.
- **Version:** TypeVersion **2.1**
- **Failure/edge cases:**
  - If no negative reviews, this node won’t run, and **the email summary won’t run either** (because it is downstream).
  - Slack OAuth missing scopes (e.g., `chat:write`) or wrong channel ID → errors.

#### Node: Email Daily Summary
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends a summary email.
- **Config (interpreted):**
  - To: `Setup Variables.owner_email`
  - Subject: `Daily Google Reviews Summary - {{$now.format('YYYY-MM-DD')}}`
  - Body computes:
    - New reviews today: `$('Parse AI Results').all().length`
    - Negative reviews: `$('Filter Negative Reviews').all().length`
    - Average rating: sum(rating)/count
    - Loops through all parsed reviews and truncates text
- **Inputs:** From `Send Slack Alert`.
- **Outputs:** None.
- **Version:** TypeVersion **2.1**
- **Failure/edge cases:**
  - Division by zero if `Parse AI Results` has 0 items (average rating expression will error or return Infinity/NaN).
  - The `{% for ... %}` templating assumes n8n’s message field supports this templating mode; if not enabled, the loop will be sent literally.
  - “Daily summary” is not actually daily: it triggers only when Slack alert runs, and only when the workflow runs.

---

### Sticky Notes (contextual)
- **Sticky Note:** Describes overall workflow purpose + setup steps (Place ID, API key, Gemini, Sheets, Slack, Gmail).
- **Sticky Note1 (“Fetch reviews”):** Hourly schedule; Place ID stored in Setup Variables.
- **Sticky Note2 (“AI analysis”):** Gemini does sentiment + reply draft; code parses output.
- **Sticky Note3 (“Save, filter, and alert”):** Sheets logging, Slack alert for negatives, Gmail daily summary.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Review Check Schedule | scheduleTrigger | Hourly trigger | — | Setup Variables | ## Fetch reviews; Schedule trigger checks every hour. Setup Variables node holds the Place ID — change it to your business. |
| Setup Variables | set | Defines IDs/emails/state seed | Review Check Schedule | Get Google Reviews | ### Monitor Google reviews and draft AI responses with Gemini and Slack; Automatic Google Reviews monitoring system… Setup steps… |
| Get Google Reviews | httpRequest | Calls Google Places Details API | Setup Variables | Filter New Reviews | ## Fetch reviews; Schedule trigger checks every hour. Setup Variables node holds the Place ID — change it to your business. |
| Filter New Reviews | code | Filters reviews by timestamp, emits per-review items | Get Google Reviews | Sentiment Analysis & Draft Response | ## AI analysis; Gemini reads each review, scores sentiment 1-5, and drafts a polite response. The code node below parses the JSON output. |
| Google Gemini Chat Model | lmChatGoogleGemini | Provides Gemini chat model to chain | — | Sentiment Analysis & Draft Response (ai_languageModel) | ## AI analysis; Gemini reads each review, scores sentiment 1-5, and drafts a polite response. The code node below parses the JSON output. |
| Sentiment Analysis & Draft Response | chainLlm | Runs LLM chain to produce sentiment + response | Filter New Reviews; Google Gemini Chat Model (ai_languageModel) | Parse AI Results | ## AI analysis; Gemini reads each review, scores sentiment 1-5, and drafts a polite response. The code node below parses the JSON output. |
| Parse AI Results | code | Parses LLM output, enriches review record | Sentiment Analysis & Draft Response | Log to Google Sheets | ## AI analysis; Gemini reads each review, scores sentiment 1-5, and drafts a polite response. The code node below parses the JSON output. |
| Log to Google Sheets | googleSheets | Append/update review log in Sheets | Parse AI Results | Filter Negative Reviews | ## Save, filter, and alert; All reviews go to Sheets. Negative ones (score < 3) get a Slack alert. Daily summary goes out via Gmail. |
| Filter Negative Reviews | filter | Pass-through only negative sentiment items | Log to Google Sheets | Send Slack Alert | ## Save, filter, and alert; All reviews go to Sheets. Negative ones (score < 3) get a Slack alert. Daily summary goes out via Gmail. |
| Send Slack Alert | slack | Posts alert with AI draft for negative reviews | Filter Negative Reviews | Email Daily Summary | ## Save, filter, and alert; All reviews go to Sheets. Negative ones (score < 3) get a Slack alert. Daily summary goes out via Gmail. |
| Email Daily Summary | gmail | Sends summary email | Send Slack Alert | — | ## Save, filter, and alert; All reviews go to Sheets. Negative ones (score < 3) get a Slack alert. Daily summary goes out via Gmail. |
| Sticky Note | stickyNote | Comment block | — | — | ### Monitor Google reviews and draft AI responses with Gemini and Slack… (content as note) |
| Sticky Note1 | stickyNote | Comment block | — | — | ## Fetch reviews… |
| Sticky Note2 | stickyNote | Comment block | — | — | ## AI analysis… |
| Sticky Note3 | stickyNote | Comment block | — | — | ## Save, filter, and alert… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add node **Schedule Trigger** named `Review Check Schedule`.
   2. Set schedule to **Cron**: `0 * * * *` (hourly at minute 0).

2. **Add configuration node**
   1. Add **Set** node named `Setup Variables`.
   2. Add fields:
      - `place_id` (String): your Google Place ID
      - `business_name` (String)
      - `sheet_id` (String): target spreadsheet ID
      - `slack_channel_id` (String): target channel ID
      - `owner_email` (String)
      - `last_check_timestamp` (Number): `0` (see note below about persistence)
   3. Connect: `Review Check Schedule` → `Setup Variables`.

3. **Fetch reviews from Google**
   1. Add **HTTP Request** node named `Get Google Reviews`.
   2. Set URL: `https://maps.googleapis.com/maps/api/place/details/json`
   3. Add **Query Parameters** (required for real operation):
      - `place_id` = `{{$('Setup Variables').item.json.place_id}}`
      - `fields` = `reviews,rating,name` (adjust as needed)
      - `key` = your Google Maps API key (most common approach)
   4. If you must use header auth, configure **HTTP Header Auth** credential accordingly, but verify Google accepts it (normally it does not for Maps API keys).
   5. Connect: `Setup Variables` → `Get Google Reviews`.

4. **Filter only new reviews**
   1. Add **Code** node named `Filter New Reviews`.
   2. Paste logic equivalent to:
      - Read `result.reviews`
      - Compare `review.time` to `last_check_timestamp`
      - Output one item per new review, include `place_id` + `business_name`
   3. Connect: `Get Google Reviews` → `Filter New Reviews`.

   **Persistence requirement (recommended):**
   - Replace `last_check_timestamp` handling with one of:
     - n8n **Data Store** (get/set timestamp), or
     - Workflow **static data** (in code), or
     - Store/retrieve last timestamp from Google Sheets.
   - Otherwise you will reprocess the same reviews every hour.

5. **Configure Gemini model**
   1. Add node **Google Gemini Chat Model** named `Google Gemini Chat Model`.
   2. Configure credentials for Gemini (Google AI / Gemini API key depending on your n8n setup).
   3. Optionally set model + temperature to keep output stable.

6. **Create the LLM chain**
   1. Add **Chain LLM** node named `Sentiment Analysis & Draft Response`.
   2. Connect the model:
      - `Google Gemini Chat Model` → `Sentiment Analysis & Draft Response` (connection type **ai_languageModel**).
   3. Connect data:
      - `Filter New Reviews` → `Sentiment Analysis & Draft Response` (main).
   4. In the chain/prompt, enforce an output format that matches the parser, e.g.:
      - Line 1: `SENTIMENT: <1-5>`
      - Line 2: `RESPONSE: <single line reply draft>`

7. **Parse AI output**
   1. Add **Code** node named `Parse AI Results`.
   2. Implement parsing for `SENTIMENT:` and `RESPONSE:` and output:
      - `sentiment_score`, `ai_response_draft`, `is_negative`, `created_at` plus original fields.
   3. Connect: `Sentiment Analysis & Draft Response` → `Parse AI Results`.

8. **Log to Google Sheets**
   1. Add **Google Sheets** node named `Log to Google Sheets`.
   2. Auth: **Google Sheets OAuth2** credentials.
   3. Document ID: `{{$('Setup Variables').item.json.sheet_id}}`
   4. Sheet name: `Reviews Log` (create this tab in the spreadsheet).
   5. Operation: **Append or Update**; set column mappings:
      - `text`, `rating`, `review_id`, `created_at`, `author_name`, `business_name`, `sentiment_score`, `ai_response_draft`
   6. Connect: `Parse AI Results` → `Log to Google Sheets`.

9. **Filter negative**
   1. Add **Filter** node named `Filter Negative Reviews`.
   2. Condition: `{{$json.is_negative}}` equals `true`.
   3. Connect: `Log to Google Sheets` → `Filter Negative Reviews`.

10. **Send Slack alert**
   1. Add **Slack** node named `Send Slack Alert`.
   2. Auth: Slack OAuth2 credentials with permission to post messages.
   3. Choose “Send message” equivalent action; set channel to:
      - `{{$('Setup Variables').item.json.slack_channel_id}}`
   4. Message text: include rating, author, review text, AI response draft, sentiment score.
   5. Connect: `Filter Negative Reviews` → `Send Slack Alert`.

11. **Email summary**
   1. Add **Gmail** node named `Email Daily Summary`.
   2. Auth: Gmail OAuth2.
   3. To: `{{$('Setup Variables').item.json.owner_email}}`
   4. Subject: `Daily Google Reviews Summary - {{$now.format('YYYY-MM-DD')}}`
   5. Body: include counts and average rating; ensure your expressions handle 0 items safely.
   6. Connect: `Send Slack Alert` → `Email Daily Summary`.
   - If you truly want a **daily** email, instead create a **second Schedule Trigger** (daily) and build a separate branch that reads today’s rows from Sheets and emails them.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatic Google Reviews monitoring system that checks hourly, drafts AI responses with Gemini, logs to Sheets, alerts Slack for negative reviews, and emails an owner summary. | From sticky note “Monitor Google reviews and draft AI responses with Gemini and Slack”. |
| Setup steps: set Place ID, add Google Maps API key, connect Gemini, connect Sheets, connect Slack, set Gmail recipient. | From sticky note “Monitor Google reviews…”. |
| “Fetch reviews” block: schedule trigger checks hourly; change Place ID in Setup Variables. | From sticky note “Fetch reviews”. |
| “AI analysis” block: Gemini scores sentiment 1–5 and drafts response; code parses output. | From sticky note “AI analysis”. |
| “Save, filter, and alert” block: all reviews to Sheets; negative (score < 3) to Slack; daily summary via Gmail. | From sticky note “Save, filter, and alert”. |