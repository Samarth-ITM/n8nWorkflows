Monitor ad performance drops with Meta & Google Ads + multi-channel alerts

https://n8nworkflows.xyz/workflows/monitor-ad-performance-drops-with-meta---google-ads---multi-channel-alerts-12091


# Monitor ad performance drops with Meta & Google Ads + multi-channel alerts

## 1. Workflow Overview

**Workflow name (JSON):** Automate ad performance monitoring with Meta Ads, Google Ads & Email alerts  
**Title provided:** Monitor ad performance drops with Meta & Google Ads + multi-channel alerts  
**Status:** Inactive (`active: false`)  
**Purpose:** Monitor daily campaign performance for **Meta Ads** and **Google Ads**, detect **CTR/ROAS drops** against fixed thresholds, then send **multi-channel alerts** (Slack, Gmail, WhatsApp) and **log the alert event to Google Sheets**.

### 1.1 Input Reception (Scheduled Runs)
Two schedule triggers run at different hours:
- **09:00** → fetch Meta Ads performance for yesterday.
- **10:00** → fetch Google Ads campaign data.

### 1.2 Data Normalization (Benchmarks/Standard Fields)
Each platform’s data is reshaped into a common structure (campaign_name, impressions, clicks, spend, action_values/actions, platform).

### 1.3 Drop Detection (CTR/ROAS calculation + flagging)
A Code node computes:
- CTR = clicks / impressions * 100  
- ROAS = revenue / spend, where revenue is extracted from purchase action values  
Then flags `alert=true` if below thresholds.

### 1.4 Alert Fan-out + Logging
If `alert=true`, items are batched and alerts are sent:
- Slack message
- Gmail email
- WhatsApp send (configuration incomplete)
Then the workflow “re-hydrates” items (pulls from the detection node), waits briefly, and writes a row to Google Sheets with alert metadata.

---

## 2. Block-by-Block Analysis

### Block 1 — Fetch Ad Performance Data (Meta + Google)
**Overview:** Runs on schedules and pulls campaign performance data from each ad platform.  
**Nodes involved:** `Daily Ad Check2`, `Fetch Meta Ads Data`, `Daily Ad Check3`, `Get many campaigns`

#### Node: Daily Ad Check2
- **Type / role:** Schedule Trigger; entry point for Meta branch.
- **Config:** Runs daily at **09:00** (`triggerAtHour: 9`).
- **Outputs:** Triggers `Fetch Meta Ads Data`.
- **Edge cases:** n8n instance timezone affects “9:00”; consider setting workflow timezone in n8n settings.

#### Node: Fetch Meta Ads Data
- **Type / role:** HTTP Request; calls Meta (Facebook Graph API) insights endpoint.
- **Config choices:**
  - `GET your-facebook-graph-api-endpoint`
  - Query parameters:
    - `fields=campaign_name,impressions,clicks,spend,actions,action_values`
    - `date_preset=yesterday`
    - `level=campaign`
  - Header: `Authorization: Bearer YOUR_TOKEN_HERE`
- **Input:** From `Daily Ad Check2`.
- **Output:** To `Set Benchmarks`.
- **Failure types / edge cases:**
  - Token expiry / invalid scopes → 401/403.
  - Pagination: Meta Insights can paginate; this node doesn’t show paging handling.
  - Response shape: often Meta returns `{ data: [...] }`. If you don’t enable “Split into items” (not shown), downstream nodes may receive a single item instead of per-campaign items.

#### Node: Daily Ad Check3
- **Type / role:** Schedule Trigger; entry point for Google Ads branch.
- **Config:** Runs daily at **10:00** (`triggerAtHour: 10`; minute is empty/unspecified in JSON).
- **Outputs:** Triggers `Get many campaigns`.
- **Edge cases:** Same timezone considerations; also “empty minute” can default to 0 depending on node version/UI.

#### Node: Get many campaigns
- **Type / role:** Google Ads node; fetches multiple campaigns.
- **Config choices:** Operation is not explicitly shown (node name implies “Get many campaigns”); no query fields are specified in parameters.
- **Input:** From `Daily Ad Check3`.
- **Output:** To `Set Benchmarks4`.
- **Version-specific:** `typeVersion: 1` (older Google Ads node generation).
- **Failure types / edge cases:**
  - OAuth/Developer token / Customer ID requirements (typical for Google Ads) not visible here—misconfiguration will fail at runtime.
  - Returned metrics may not match expected schema (`impressions`, `clicks`, `spend`, `action_values`, `actions`). Google Ads usually provides conversions/value via different fields; this workflow assumes Meta-like fields.

---

### Block 2 — Normalize Data (Common Fields)
**Overview:** Converts Meta/Google outputs into a consistent JSON structure and stamps a `platform` field.  
**Nodes involved:** `Set Benchmarks`, `Set Benchmarks4`

#### Node: Set Benchmarks (Meta)
- **Type / role:** Set node; maps incoming Meta campaign fields to normalized keys.
- **Config choices:**
  - Assigns:
    - `campaign_name = $json.campaign_name`
    - `impressions = Number($json.impressions)`
    - `clicks = Number($json.clicks)`
    - `spend = Number($json.spend)`
    - `action_values = $json.action_values` (array)
    - `actions = $json.actions` (array)
    - `platform = "META"`
- **Input:** From `Fetch Meta Ads Data`.
- **Output:** To `Detect Performance Drop`.
- **Edge cases:** If Meta returns strings for metrics, conversion to number is fine; if field missing, number becomes `null` or `NaN` depending on input.

#### Node: Set Benchmarks4 (Google)
- **Type / role:** Set node; applies the same normalized schema but sets platform to Google.
- **Config choices:** Same field mapping as Meta node, but `platform = "Google"`.
- **Input:** From `Get many campaigns`.
- **Output:** To `Detect Performance Drop`.
- **Edge cases:** If Google Ads node output doesn’t provide `action_values/actions`, ROAS computation later will produce 0 revenue.

---

### Block 3 — Detect Performance Drops
**Overview:** Computes CTR, revenue, ROAS, and decides whether to raise an alert based on thresholds.  
**Nodes involved:** `Detect Performance Drop`, `If`

#### Node: Detect Performance Drop
- **Type / role:** Code (JavaScript); transforms each campaign item, adds computed fields and alert flags.
- **Key logic / thresholds:**
  - `MIN_CTR = 2` (%)
  - `MIN_ROAS = 4` (ratio)
  - CTR:
    - If impressions > 0: `(clicks / impressions) * 100`, else 0
  - Revenue extraction:
    - Looks for `action_values.find(a => a.action_type === 'purchase')`
    - Uses `purchase.value` if present
  - ROAS:
    - If spend > 0: `revenue/spend`, else 0
  - Alert decision:
    - If CTR below threshold → `alert=true`, `Performance_Status='DROPPED'`, reason includes “CTR below threshold”
    - If ROAS below threshold → also triggers; reason concatenates if both fail
- **Expressions/variables:**
  - Uses `$input.all()` (all incoming items)
  - Mentions `previous_ctr` / `previous_roas` but these are not populated elsewhere in this workflow.
- **Input:** From both `Set Benchmarks` and `Set Benchmarks4` (two branches converge).
- **Output:** To `If`.
- **Edge cases / failure types:**
  - If `action_values` schema differs (common between Meta vs Google), revenue stays 0.
  - If spend is string with currency formatting, `Number()` may yield `NaN`.
  - No “previous day comparison” exists despite variables; alerts are purely threshold-based, not “drop vs prior”.

#### Node: If
- **Type / role:** If node; routes only items with `alert === true`.
- **Condition:** `{{ $json.alert }}` is true (boolean check).
- **Input:** From `Detect Performance Drop`.
- **Output:** True branch → `Loop Over Items`; False branch unused.
- **Edge cases:** If `alert` becomes string `"true"` instead of boolean, strict validation could fail; current code sets a boolean, so OK.

---

### Block 4 — Alert & Log Results (Slack/Gmail/WhatsApp + Sheets)
**Overview:** For each alerted campaign, send notifications through multiple channels, then record the alert in Google Sheets.  
**Nodes involved:** `Loop Over Items`, `Send a message` (Slack), `Send a message4` (Gmail), `Send message1` (WhatsApp), `Code in JavaScript`, `Wait1`, `your-google-sheets-name`

#### Node: Loop Over Items
- **Type / role:** Split In Batches; controls per-item processing.
- **Config:** Default options (batch size not specified in JSON; default is typically 1).
- **Input:** From `If` (true path).
- **Outputs:**
  - Output 0: unused
  - Output 1: goes to Slack + Gmail + WhatsApp in parallel
- **Edge cases:** If you have many campaigns failing at once, this can throttle sends; but because batch configuration isn’t explicit, behavior depends on node defaults.

#### Node: Send a message (Slack)
- **Type / role:** Slack node; posts alert text to a channel.
- **Auth:** OAuth2 credential `slackOAuth2Api Credential`.
- **Config choices:**
  - Sends to a specific channel (`channelId: C0A56EXB2D8`).
  - Message body is a templated block containing:
    - campaign_name, platform, impressions, clicks, spend, ctr, roas, Performance_Status, Drop_Reason
- **Input:** From `Loop Over Items` (batch output 1).
- **Output:** To `Code in JavaScript`.
- **Failure types:** OAuth token revoked; missing chat:write scopes; channel access issues.

#### Node: Send a message4 (Gmail)
- **Type / role:** Gmail node; sends an email alert.
- **Auth:** OAuth2 credential `gmailOAuth2 Credential`.
- **Config choices:**
  - To: `your email address`
  - Subject: `🚨 Ad Performance Alert: Immediate Attention Required`
  - Message: same templated content as Slack (HTML-like tags in plain body).
- **Input:** From `Loop Over Items`.
- **Output:** To `Code in JavaScript`.
- **Edge cases:** Gmail API may treat HTML tags as plain text unless configured; quota limits; OAuth consent/scopes.

#### Node: Send message1 (WhatsApp)
- **Type / role:** WhatsApp node; intended to send WhatsApp alerts.
- **Config choices:**
  - `operation: send`
  - No recipient/message fields are defined in parameters (appears incomplete).
- **Input:** From `Loop Over Items`.
- **Output:** To `Code in JavaScript`.
- **Failure types / edge cases:**
  - Node likely fails validation at runtime due to missing required fields (recipient, message).
  - WhatsApp provider credentials not shown; depends on n8n’s WhatsApp integration used (often Twilio or WhatsApp Cloud API).

#### Node: Code in JavaScript
- **Type / role:** Code; resets the working items to the original output of `Detect Performance Drop`.
- **Key code:** `const items = $items("Detect Performance Drop"); return items.map(item => ({ json: item.json }));`
- **Purpose (important):** The Slack/Gmail/WhatsApp nodes may alter/limit output; this node ensures the subsequent logging step has the full metric payload.
- **Inputs:** From Slack, Gmail, WhatsApp (three inbound connections).
- **Output:** To `Wait1`.
- **Edge cases:**
  - If this node runs once per inbound branch, it can duplicate logging (depending on n8n merge behavior). With three alert channels feeding it, you risk **logging the same alert multiple times** unless execution merges items as expected.
  - `$items("Detect Performance Drop")` pulls all items from that node, not only currently-batched/alerted items—can cause over-logging.

#### Node: Wait1
- **Type / role:** Wait node; delays before logging (or provides async resume capability).
- **Config:** Empty (no wait time specified).
- **Input:** From `Code in JavaScript`.
- **Output:** To `your-google-sheets-name`.
- **Edge cases:** If no wait duration is set, behavior depends on node defaults/UI configuration; could pause indefinitely awaiting resume/webhook depending on mode.

#### Node: your-google-sheets-name
- **Type / role:** Google Sheets; append or update an alert log row.
- **Operation:** `appendOrUpdate`
- **Sheet/Doc:** Points to placeholders:
  - `documentId: your-google-sheets-document-id`
  - `sheetName: Ad_Performance_Log (gid=0)`
- **Matching:** `matchingColumns: ["Campaign_Name"]` (updates existing row for the same campaign name).
- **Columns mapped:**
  - `Date = {{$now.format('yyyy-MM-dd')}}`
  - `Check_Time = {{$now}}`
  - `Platform, Campaign_Name, Impressions, Clicks, Spend, CTR, ROAS`
  - `Performance_Status, Drop_Reason`
  - `Currency = "INR"`
  - `Alert_Sent = "Yes"`
  - `Alert_Channel = "Slack,Email,whatsapp"`
- **Input:** From `Wait1`.
- **Version:** `typeVersion: 4.7` (newer Sheets node).
- **Edge cases / failure types:**
  - If multiple alerts for the same campaign occur, row updates overwrite prior history (since matching is Campaign_Name). If you want a full history, use append-only with a unique key (timestamp).
  - Currency is hard-coded to INR regardless of platform/account currency.
  - If `campaign_name` is not unique across platforms, Google/META campaigns can overwrite each other unless Platform is included in matching.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Ad Check2 | Schedule Trigger | Meta daily trigger | — | Fetch Meta Ads Data | ## Step 1: Fetch Ad Performance Data<br>Pulls daily campaign metrics from Meta Ads and Google Ads using scheduled triggers. |
| Fetch Meta Ads Data | HTTP Request | Fetch Meta Ads campaign metrics (yesterday) | Daily Ad Check2 | Set Benchmarks | ## Step 1: Fetch Ad Performance Data<br>Pulls daily campaign metrics from Meta Ads and Google Ads using scheduled triggers. |
| Daily Ad Check3 | Schedule Trigger | Google Ads daily trigger | — | Get many campaigns | ## Step 1: Fetch Ad Performance Data<br>Pulls daily campaign metrics from Meta Ads and Google Ads using scheduled triggers. |
| Get many campaigns | Google Ads | Fetch Google Ads campaign list/metrics | Daily Ad Check3 | Set Benchmarks4 | ## Step 1: Fetch Ad Performance Data<br>Pulls daily campaign metrics from Meta Ads and Google Ads using scheduled triggers. |
| Set Benchmarks | Set | Normalize Meta fields + set platform=META | Fetch Meta Ads Data | Detect Performance Drop | ## Step 2: Detect Performance Drops<br>Calculates CTR and ROAS and flags campaigns that fall below defined benchmarks. |
| Set Benchmarks4 | Set | Normalize Google fields + set platform=Google | Get many campaigns | Detect Performance Drop | ## Step 2: Detect Performance Drops<br>Calculates CTR and ROAS and flags campaigns that fall below defined benchmarks. |
| Detect Performance Drop | Code | Compute CTR/ROAS + alert flag + reason | Set Benchmarks; Set Benchmarks4 | If | ## Step 2: Detect Performance Drops<br>Calculates CTR and ROAS and flags campaigns that fall below defined benchmarks. |
| If | If | Filter only alert=true items | Detect Performance Drop | Loop Over Items | ## Step 2: Detect Performance Drops<br>Calculates CTR and ROAS and flags campaigns that fall below defined benchmarks. |
| Loop Over Items | Split In Batches | Iterate through alert items and fan-out alerts | If | Send a message; Send a message4; Send message1 | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| Send a message | Slack | Send Slack alert | Loop Over Items | Code in JavaScript | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| Send a message4 | Gmail | Send email alert | Loop Over Items | Code in JavaScript | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| Send message1 | WhatsApp | Send WhatsApp alert (incomplete config) | Loop Over Items | Code in JavaScript | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| Code in JavaScript | Code | Re-load items from Detect Performance Drop before logging | Send a message; Send a message4; Send message1 | Wait1 | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| Wait1 | Wait | Delay/gate before Sheets logging | Code in JavaScript | your-google-sheets-name | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| your-google-sheets-name | Google Sheets | Append/update alert log row | Wait1 | — | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |
| Sticky Note4 | Sticky Note | Overall workflow description & setup steps | — | — | ## Ad Performance Drop Alert Automation (Google & Meta Ads)<br><br>This workflow monitors your Google Ads and Meta Ads campaigns on a daily schedule to detect sudden performance drops and notify your team immediately. It helps prevent wasted ad spend by catching issues early, before they impact results at scale.<br><br>### How it works<br>The workflow runs automatically using scheduled triggers. It pulls yesterday’s campaign performance data from Meta Ads and Google Ads, including impressions, clicks, spend, and conversion values. For each campaign, it calculates key metrics such as CTR, revenue, and ROAS. These values are compared against predefined benchmarks to identify campaigns that are underperforming. When a drop is detected, the campaign is flagged with a clear performance status and drop reason. Alerts are then sent through multiple channels, and the event is logged for tracking and reporting.<br><br>### Setup steps<br>1. Connect your Meta Ads account and provide a valid access token.<br>2. Authenticate your Google Ads account.<br>3. Review and adjust CTR and ROAS thresholds in the performance detection logic.<br>4. Connect Slack, Gmail, and WhatsApp for alerts.<br>5. Link a Google Sheet to store alert history.<br>6. Set your preferred daily schedule times.<br>7. Activate the workflow. |
| Sticky Note5 | Sticky Note | Section header (Step 1) | — | — | ## Step 1: Fetch Ad Performance Data<br>Pulls daily campaign metrics from Meta Ads and Google Ads using scheduled triggers. |
| Sticky Note6 | Sticky Note | Section header (Step 2) | — | — | ## Step 2: Detect Performance Drops<br>Calculates CTR and ROAS and flags campaigns that fall below defined benchmarks. |
| Sticky Note7 | Sticky Note | Section header (Step 3) | — | — | ## Step 3: Alert & Log Results<br>Sends alerts via Slack, Email, and WhatsApp and logs results in Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it similar to: *Automate ad performance monitoring with Meta Ads, Google Ads & Email alerts*.
   - Ensure workflow timezone is correct for your business hours.

2. **Add Schedule Trigger for Meta**
   - Node: **Schedule Trigger**
   - Set to run daily at **09:00**.
   - Name: `Daily Ad Check2`.

3. **Add HTTP Request to fetch Meta Ads Insights**
   - Node: **HTTP Request**
   - Name: `Fetch Meta Ads Data`
   - Method: GET
   - URL: your Meta Graph Insights endpoint (e.g., `https://graph.facebook.com/vXX.X/act_<AD_ACCOUNT_ID>/insights`)
   - Query parameters:
     - `fields`: `campaign_name,impressions,clicks,spend,actions,action_values`
     - `date_preset`: `yesterday`
     - `level`: `campaign`
   - Headers:
     - `Authorization`: `Bearer <META_ACCESS_TOKEN>`
   - Connect: `Daily Ad Check2` → `Fetch Meta Ads Data`.

4. **Normalize Meta output**
   - Node: **Set**
   - Name: `Set Benchmarks`
   - Add fields (set types accordingly):
     - `campaign_name` (string) → `{{$json.campaign_name}}`
     - `impressions` (number) → `{{$json.impressions}}`
     - `clicks` (number) → `{{$json.clicks}}`
     - `spend` (number) → `{{$json.spend}}`
     - `actions` (array) → `{{$json.actions}}`
     - `action_values` (array) → `{{$json.action_values}}`
     - `platform` (string) → `META`
   - Connect: `Fetch Meta Ads Data` → `Set Benchmarks`.

5. **Add Schedule Trigger for Google Ads**
   - Node: **Schedule Trigger**
   - Set to run daily at **10:00**.
   - Name: `Daily Ad Check3`.

6. **Fetch Google Ads campaigns**
   - Node: **Google Ads**
   - Name: `Get many campaigns`
   - Configure credentials (Google Ads OAuth2 / developer token requirements per your n8n setup).
   - Choose an operation that returns campaign performance fields you can map to:
     - impressions, clicks, cost/spend, conversions value (for revenue)
   - Connect: `Daily Ad Check3` → `Get many campaigns`.

7. **Normalize Google output**
   - Node: **Set**
   - Name: `Set Benchmarks4`
   - Map the Google response fields into the same normalized keys:
     - `campaign_name`, `impressions`, `clicks`, `spend`
     - Provide `action_values` in Meta-like format **or** plan to adjust the detection code to read Google’s conversion value field.
     - `platform` → `Google`
   - Connect: `Get many campaigns` → `Set Benchmarks4`.

8. **Add drop detection logic**
   - Node: **Code** (JavaScript)
   - Name: `Detect Performance Drop`
   - Paste logic that:
     - computes CTR
     - extracts revenue
     - computes ROAS
     - sets `alert`, `Performance_Status`, `Drop_Reason`
   - Set thresholds inside code (`MIN_CTR=2`, `MIN_ROAS=4`) as desired.
   - Connect:
     - `Set Benchmarks` → `Detect Performance Drop`
     - `Set Benchmarks4` → `Detect Performance Drop`

9. **Filter only dropped campaigns**
   - Node: **If**
   - Name: `If`
   - Condition: Boolean → `{{$json.alert}}` is true
   - Connect: `Detect Performance Drop` → `If`

10. **Batch over alert items**
   - Node: **Split In Batches**
   - Name: `Loop Over Items`
   - Keep default batch size or set to 1 for controlled sending.
   - Connect: `If (true)` → `Loop Over Items`

11. **Slack alert**
   - Node: **Slack**
   - Name: `Send a message`
   - Auth: Slack OAuth2 credential (scopes to post messages)
   - Send to: Channel (select your channel)
   - Message: template referencing campaign fields (campaign_name, ctr, roas, etc.)
   - Connect: `Loop Over Items` (batch output used for processing) → `Send a message`

12. **Email alert**
   - Node: **Gmail**
   - Name: `Send a message4`
   - Auth: Gmail OAuth2 credential
   - To: your target address(es)
   - Subject + body using the same fields
   - Connect: `Loop Over Items` → `Send a message4`

13. **WhatsApp alert (complete the missing pieces)**
   - Node: **WhatsApp**
   - Name: `Send message1`
   - Configure provider credentials and required fields (recipient + message/body).
   - Connect: `Loop Over Items` → `Send message1`

14. **Prepare data for logging (as implemented)**
   - Node: **Code** (JavaScript)
   - Name: `Code in JavaScript`
   - Logic: reload items from `Detect Performance Drop` using `$items("Detect Performance Drop")`.
   - Connect Slack → Code, Gmail → Code, WhatsApp → Code.

15. **Add Wait**
   - Node: **Wait**
   - Name: `Wait1`
   - Configure wait mode/time explicitly (recommended) to avoid indefinite pauses.
   - Connect: `Code in JavaScript` → `Wait1`

16. **Log to Google Sheets**
   - Node: **Google Sheets**
   - Name: `your-google-sheets-name`
   - Auth: Google Sheets credential
   - Operation: **Append or Update**
   - Document: select your spreadsheet
   - Sheet: `Ad_Performance_Log`
   - Matching column: `Campaign_Name`
   - Map columns:
     - Date: `{{$now.format('yyyy-MM-dd')}}`
     - Check_Time: `{{$now}}`
     - Platform, Campaign_Name, Impressions, Clicks, Spend, CTR, ROAS
     - Performance_Status, Drop_Reason
     - Currency (hard-coded or dynamic)
     - Alert_Sent = `Yes`
     - Alert_Channel = `Slack,Email,whatsapp`
   - Connect: `Wait1` → `your-google-sheets-name`

17. **Activate the workflow**
   - Validate each credential works by executing each branch.
   - Activate once WhatsApp and Google Ads field mapping are confirmed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Ad Performance Drop Alert Automation (Google & Meta Ads)… Setup steps 1–7…” | Embedded in Sticky Note: overall description and setup checklist |
| The workflow computes alerts based on fixed thresholds (MIN_CTR, MIN_ROAS), not “drop vs previous period”, despite `previous_ctr/previous_roas` placeholders. | Consider adding a Google Sheets lookup step before detection if you need true “drop” detection |
| Google Sheets logging uses `appendOrUpdate` keyed only on `Campaign_Name`, which can overwrite history and can collide across platforms. | Consider matching on `Campaign_Name + Platform` or appending with a timestamp key |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.