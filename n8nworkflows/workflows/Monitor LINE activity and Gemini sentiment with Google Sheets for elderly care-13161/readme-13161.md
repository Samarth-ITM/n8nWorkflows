Monitor LINE activity and Gemini sentiment with Google Sheets for elderly care

https://n8nworkflows.xyz/workflows/monitor-line-activity-and-gemini-sentiment-with-google-sheets-for-elderly-care-13161


# Monitor LINE activity and Gemini sentiment with Google Sheets for elderly care

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor an elderly relative’s activity on LINE in real time, log each interaction into Google Sheets, analyze message sentiment + potential health risk with Google Gemini, and automate inactivity follow-ups (reminder → escalation).

**Typical use cases**
- Family caregivers want passive monitoring without constant manual check-ins.
- Immediate alerting when the parent expresses distress (pain, dizziness, etc.).
- Daily watchdog that detects “no messages today” and escalates if silence persists.

### 1.1 Real-time Reception & AI Risk Detection (LINE → Gemini → Sheets → Alert)
Triggered by a LINE webhook. Formats the message, asks Gemini to classify sentiment and detect health risk, logs to Google Sheets, and alerts the child if a risk is detected.

### 1.2 Daily Inactivity Watchdog (Schedule → Sheets → Reminder → Wait → Re-check → Emergency Alert)
Runs daily at 11:00. Reads today’s logs from Google Sheets; if none exist, reminds the parent. After 3 hours, re-checks logs; if still no activity, sends an emergency alert to the child.

### 1.3 Configuration Layer
Two Set nodes store the same configuration values (parent/child LINE IDs and Sheet ID), used separately by the real-time path and the daily watchdog path.

---

## 2. Block-by-Block Analysis

### Block 1 — Configuration (shared constants)
**Overview:** Provides workflow-level constants (LINE user IDs and Google Sheet document ID) via Set nodes. In the current JSON, these values are empty and must be filled.

**Nodes involved**
- **Config**
- **Config for Daily**

#### Node: Config
- **Type / role:** Set node; injects constants for the webhook (real-time) branch.
- **Configuration choices:** Adds three string fields: `PARENT_LINE_ID`, `CHILD_LINE_ID`, `SHEET_ID` (currently empty).
- **Key variables/expressions:** None (static values).
- **Connections:**
  - **Input:** `LINE Webhook`
  - **Output:** `Format Message`
- **Edge cases / failures:**
  - Empty `SHEET_ID` breaks Google Sheets nodes downstream (documentId resolution fails).
  - Empty LINE IDs won’t break the logging branch, but will break messaging nodes if referenced (in this workflow, LINE message nodes are not fully configured in JSON).
- **Version-specific notes:** Set node `typeVersion 3.4` uses “assignments” structure.

#### Node: Config for Daily
- **Type / role:** Set node; injects constants for the scheduled watchdog branch.
- **Configuration choices:** Same three string fields as above (currently empty).
- **Connections:**
  - **Input:** `Daily 11 AM`
  - **Output:** `Read Logs`
- **Edge cases / failures:** Same as `Config`.
- **Version-specific notes:** Set node `typeVersion 3.4`.

---

### Block 2 — Real-time Monitoring: LINE webhook ingestion → message normalization
**Overview:** Receives LINE webhook payloads and normalizes them into a consistent message string and timestamp for downstream AI analysis.

**Nodes involved**
- **LINE Webhook**
- **Format Message**

#### Node: LINE Webhook
- **Type / role:** Webhook trigger (HTTP POST). Entry point for LINE Messaging API events.
- **Configuration choices:**
  - **HTTP Method:** POST
  - **Path:** `webhook` (your full URL will be `https://<n8n-host>/webhook/webhook` for test and `/webhook/webhook` production depending on n8n setup)
- **Connections:**
  - **Output:** `Config`
- **Edge cases / failures:**
  - LINE signature validation is not implemented here (LINE typically sends `X-Line-Signature`). Without validation, anyone could POST to the endpoint if it’s exposed.
  - Payload shape assumptions: downstream expects `body.events[0].message...`. Non-message events (follow/unfollow/join/leave/postback) can cause expression errors unless handled.
- **Version-specific notes:** Webhook `typeVersion 1.1`.

#### Node: Format Message
- **Type / role:** Set node; extracts or substitutes message content; adds timestamp.
- **Configuration choices / key expressions:**
  - `userMessage`:
    ```n8n
    {{ $json.body.events[0].message.type === 'text'
        ? $json.body.events[0].message.text
        : '[Sent Media/Sticker]' }}
    ```
    Treats non-text (stickers/media) as a neutral placeholder.
  - `timestamp`:
    ```n8n
    {{ $now.toISO() }}
    ```
- **Connections:**
  - **Input:** `Config`
  - **Output:** `Gemini: Analyze`
- **Edge cases / failures:**
  - If `events[0]` or `message` is missing (e.g., webhook event type is not “message”), this expression can throw.
  - Only the first event in `events[]` is processed; LINE can batch multiple events in one webhook call.

---

### Block 3 — AI analysis with Gemini → robust parsing
**Overview:** Sends the normalized message to Gemini with strict JSON output instructions, then parses Gemini’s response defensively to produce clean structured fields.

**Nodes involved**
- **Gemini: Analyze**
- **Parse JSON**

#### Node: Gemini: Analyze
- **Type / role:** Google Gemini (LangChain) chat/inference node; performs sentiment classification and health-risk detection.
- **Configuration choices:**
  - **Model:** `models/gemini-1.5-flash` (fast, lower cost; good for short classification tasks)
  - **Prompt content:** Asks for:
    - `sentiment`: `Positive/Neutral/Negative`
    - `health_risk`: boolean
    - `summary`: short summary
    - Special rule: “Stickers are Neutral.”
- **Key expressions:**
  - Injects message:
    ```n8n
    Analyze this message: "{{ $json.userMessage }}"
    ```
- **Connections:**
  - **Input:** `Format Message`
  - **Output:** `Parse JSON`
- **Edge cases / failures:**
  - Credential/auth errors for Google Gemini.
  - Model may return non-JSON or extra text; handled by the next node.
  - Prompt injection: user text could try to override instructions; you partially mitigate by parsing only `{...}` but output can still be malformed.
- **Version-specific notes:** Node type `@n8n/n8n-nodes-langchain.googleGemini` `typeVersion 1` requires the LangChain/Gemini integration available in your n8n build and valid Google AI credentials.

#### Node: Parse JSON
- **Type / role:** Code node; extracts a JSON object from Gemini’s response and normalizes failure to safe defaults.
- **Configuration choices (interpreted logic):**
  - Reads Gemini output text from:
    - `$input.first().json.content.parts[0].text`
  - Extracts the first `{ ... }` block via regex and attempts `JSON.parse`.
  - Fallback defaults on missing JSON or parse error:
    - `sentiment: "Neutral"`
    - `health_risk: false`
    - `summary: "No text to analyze"` or `"Parse Error"`
- **Connections:**
  - **Input:** `Gemini: Analyze`
  - **Output:** `Log to Sheets`
- **Edge cases / failures:**
  - If Gemini node output structure changes (e.g., content path differs), parsing fails and defaults will be used (silent degradation).
  - Regex can match unintended braces if Gemini includes multiple JSON-like fragments.
- **Version-specific notes:** Code node `typeVersion 2`.

---

### Block 4 — Logging + conditional risk alert
**Overview:** Appends an activity record to Google Sheets and, if health risk is true, alerts the child via LINE.

**Nodes involved**
- **Log to Sheets**
- **Is Risk?**
- **Alert Child**

#### Node: Log to Sheets
- **Type / role:** Google Sheets node; writes/updates log rows in `ActivityLog`.
- **Configuration choices:**
  - **Operation:** `appendOrUpdate` (note: without a defined key/lookup column in the node config, behavior can be ambiguous; many setups effectively “append”)
  - **Document ID:** from config:
    ```n8n
    {{ $('Config').first().json.SHEET_ID }}
    ```
  - **Sheet name:** `ActivityLog`
  - **Column mapping:**
    - `Date`: `{{ $today.format('yyyy-MM-dd') }}`
    - `Time`: `{{ $now.format('HH:mm') }}`
    - `Message`: `{{ $('Format Message').first().json.userMessage }}`
    - `Sentiment`: `{{ $json.sentiment }}`
    - `Alert`: `{{ $json.health_risk ? 'YES' : 'NO' }}`
- **Connections:**
  - **Input:** `Parse JSON` (sentiment + health_risk)
  - **Output:** `Is Risk?`
- **Edge cases / failures:**
  - Google Sheets auth/permissions (sheet not shared with service account / OAuth user).
  - Missing worksheet `ActivityLog` or headers mismatch (`Date`, `Time`, `Message`, `Sentiment`, `Alert` expected).
  - Timezone differences: `$today` and `$now` depend on n8n instance timezone; could mismatch what “today” means for the family.
- **Version-specific notes:** Google Sheets node `typeVersion 4.7`.

#### Node: Is Risk?
- **Type / role:** IF node; routes only when `health_risk === true`.
- **Configuration choices:**
  - Condition: `{{ $json.health_risk }}` is boolean true
- **Connections:**
  - **Input:** `Log to Sheets`
  - **True output:** `Alert Child`
  - **False output:** (not connected; workflow ends for non-risk messages)
- **Edge cases / failures:**
  - If `health_risk` is missing or not boolean, strict validation may treat it as false (or fail depending on n8n validation behavior/settings).
- **Version-specific notes:** IF node `typeVersion 2.2`.

#### Node: Alert Child
- **Type / role:** LINE node; sends a message to the child when risk detected.
- **Configuration choices:** Only `resource: message` is shown in JSON; **recipient and message text are not configured** here and must be set in n8n UI.
- **Connections:**
  - **Input:** `Is Risk?` (true branch)
  - **Output:** none
- **Edge cases / failures:**
  - Missing LINE credentials / channel access token.
  - Missing “to” userId and message content will prevent sending.
  - LINE rate limiting or invalid user ID.
- **Version-specific notes:** LINE node `typeVersion 1`.

---

### Block 5 — Daily inactivity watchdog (11:00 check → reminder → 3h wait → escalation)
**Overview:** At 11:00 daily, checks whether any log entry exists for today. If none, reminds the parent. After 3 hours, re-checks; if still no activity, sends an emergency alert to the child.

**Nodes involved**
- **Daily 11 AM**
- **Config for Daily**
- **Read Logs**
- **Check Activity**
- **If Inactive**
- **Remind Parent**
- **Wait 3 Hours**
- **Read Logs Again**
- **Check Activity 2**
- **If Still Inactive**
- **Emergency Alert**

#### Node: Daily 11 AM
- **Type / role:** Schedule Trigger; entry point for daily watchdog.
- **Configuration choices:** Triggers at hour **11** (minutes default to 0 unless configured).
- **Connections:**
  - **Output:** `Config for Daily`
- **Edge cases / failures:**
  - Timezone: executes at 11:00 in the n8n instance timezone, not necessarily caregiver’s local timezone.
- **Version-specific notes:** Schedule Trigger `typeVersion 1.2`.

#### Node: Read Logs
- **Type / role:** Google Sheets read; loads `ActivityLog` rows.
- **Configuration choices:**
  - Document ID:
    ```n8n
    {{ $('Config for Daily').first().json.SHEET_ID }}
    ```
  - `alwaysOutputData: true` ensures downstream executes even if sheet returns no rows.
- **Connections:**
  - **Input:** `Config for Daily`
  - **Output:** `Check Activity`
- **Edge cases / failures:**
  - Large sheet: reading all rows daily can be slow; consider filtering or limiting rows.
  - Header mismatch can produce missing `Date` fields.
- **Version-specific notes:** Google Sheets `typeVersion 4.7`.

#### Node: Check Activity
- **Type / role:** Code node; determines if any row has `Date === today`.
- **Configuration choices (logic):**
  - `today` computed as `new Date().toISOString().split('T')[0]` (UTC date).
  - Checks `log.json.Date === today`
  - Outputs `{ hasActivity: true/false }`
- **Connections:**
  - **Input:** `Read Logs`
  - **Output:** `If Inactive`
- **Edge cases / failures:**
  - **Timezone mismatch:** sheet uses `$today` from n8n (instance timezone) when logging, but watchdog uses `toISOString()` (UTC). Around midnight, “today” can differ and cause false inactivity.
  - If `Date` is stored as a real date type (not string `YYYY-MM-DD`), comparisons may fail.
- **Version-specific notes:** Code `typeVersion 2`.

#### Node: If Inactive
- **Type / role:** IF node; proceeds when `hasActivity === false`.
- **Connections:**
  - **True output:** `Remind Parent`
  - **False output:** none
- **Edge cases / failures:** `hasActivity` missing → strict boolean check may not behave as intended.
- **Version-specific notes:** IF `typeVersion 2.2`.

#### Node: Remind Parent
- **Type / role:** LINE node; sends a gentle reminder to the parent.
- **Configuration choices:** Only `resource: message` is shown; **recipient/message not configured in JSON**.
- **Connections:**
  - **Input:** `If Inactive` (true)
  - **Output:** `Wait 3 Hours`
- **Edge cases / failures:** Same LINE messaging issues as above.

#### Node: Wait 3 Hours
- **Type / role:** Wait node; delays the flow 3 hours before re-check.
- **Configuration choices:** `unit: hours`, `amount: 3`
- **Connections:**
  - **Input:** `Remind Parent`
  - **Output:** `Read Logs Again`
- **Edge cases / failures:**
  - If n8n restarts and your wait mode/storage isn’t configured properly, delayed executions may be lost depending on your n8n deployment mode and queue setup.
- **Version-specific notes:** Wait `typeVersion 1.1`.

#### Node: Read Logs Again
- **Type / role:** Google Sheets read; same as `Read Logs` after waiting.
- **Configuration choices:** Same documentId expression using `Config for Daily`; `alwaysOutputData: true`.
- **Connections:**
  - **Input:** `Wait 3 Hours`
  - **Output:** `Check Activity 2`
- **Edge cases / failures:** Same as `Read Logs`.

#### Node: Check Activity 2
- **Type / role:** Code node; repeats activity check.
- **Configuration choices:** Same code as `Check Activity`.
- **Connections:**
  - **Input:** `Read Logs Again`
  - **Output:** `If Still Inactive`
- **Edge cases / failures:** Same timezone/date-type issues.

#### Node: If Still Inactive
- **Type / role:** IF node; proceeds when `hasActivity === false` after the wait.
- **Connections:**
  - **True output:** `Emergency Alert`
  - **False output:** none
- **Version-specific notes:** IF `typeVersion 2.2`.

#### Node: Emergency Alert
- **Type / role:** LINE node; sends an escalation alert to the child if still no activity.
- **Configuration choices:** Only `resource: message` is shown; **recipient/message not configured in JSON**.
- **Connections:**
  - **Input:** `If Still Inactive` (true)
  - **Output:** none
- **Edge cases / failures:** Same LINE messaging issues as other LINE nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main | Sticky Note | Workflow description and setup guidance | — | — | ## 👴 Elderly Care Monitor (contains setup steps and sheet headers) |
| Sticky Note - Config | Sticky Note | Visual grouping for configuration | — | — | ## ⚙️ Configuration Set User IDs here. |
| Sticky Note - Monitor | Sticky Note | Visual grouping for real-time monitoring | — | — | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Sticky Note - Watchdog | Sticky Note | Visual grouping for inactivity watchdog | — | — | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| LINE Webhook | Webhook | Receives LINE events (entry point) | — | Config | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Config | Set | Stores IDs + Sheet ID for real-time branch | LINE Webhook | Format Message | ## ⚙️ Configuration Set User IDs here. |
| Format Message | Set | Extracts text or placeholder; adds timestamp | Config | Gemini: Analyze | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Gemini: Analyze | Google Gemini (LangChain) | Sentiment + health risk inference | Format Message | Parse JSON | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Parse JSON | Code | Extract/parse JSON from Gemini output | Gemini: Analyze | Log to Sheets | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Log to Sheets | Google Sheets | Append/update ActivityLog row | Parse JSON | Is Risk? | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Is Risk? | IF | Branch when health_risk is true | Log to Sheets | Alert Child | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Alert Child | LINE | Notify child on health risk | Is Risk? | — | ## 📨 Real-time Monitoring Logs messages & detects risks. |
| Daily 11 AM | Schedule Trigger | Daily watchdog trigger at 11:00 | — | Config for Daily | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Config for Daily | Set | Stores IDs + Sheet ID for watchdog branch | Daily 11 AM | Read Logs | ## ⚙️ Configuration Set User IDs here. |
| Read Logs | Google Sheets | Read ActivityLog rows | Config for Daily | Check Activity | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Check Activity | Code | Determine if any row exists for today | Read Logs | If Inactive | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| If Inactive | IF | Branch when no activity today | Check Activity | Remind Parent | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Remind Parent | LINE | Send reminder to parent | If Inactive | Wait 3 Hours | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Wait 3 Hours | Wait | Delay then re-check logs | Remind Parent | Read Logs Again | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Read Logs Again | Google Sheets | Re-read ActivityLog rows | Wait 3 Hours | Check Activity 2 | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Check Activity 2 | Code | Re-evaluate activity for today | Read Logs Again | If Still Inactive | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| If Still Inactive | IF | Branch when still no activity | Check Activity 2 | Emergency Alert | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |
| Emergency Alert | LINE | Escalation alert to child | If Still Inactive | — | ## ⏰ Daily Inactivity Check Checks logs & sends reminders if silent. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **“Track LINE activity and AI sentiment for elderly care”** (or your preferred title).
   - Ensure the workflow can accept webhooks (publicly reachable n8n URL).

2. **Add configuration nodes**
   1) Add **Set** node named **Config**  
      - Add fields (String): `PARENT_LINE_ID`, `CHILD_LINE_ID`, `SHEET_ID`  
      - Fill them with your real values.
   2) Add **Set** node named **Config for Daily**  
      - Add the same three fields and values (or replace both with a single shared config node if you want to refactor).

3. **Real-time branch (LINE → AI → Sheets → conditional alert)**
   1) Add **Webhook** node named **LINE Webhook**
      - Method: **POST**
      - Path: **webhook**
      - Activate “Production URL” in LINE Developers webhook settings (Messaging API).
      - Connect: **LINE Webhook → Config**
   2) Add **Set** node named **Format Message**
      - Add field `userMessage` (String) with expression:  
        `{{ $json.body.events[0].message.type === 'text' ? $json.body.events[0].message.text : '[Sent Media/Sticker]' }}`
      - Add field `timestamp` (String) with expression:  
        `{{ $now.toISO() }}`
      - Connect: **Config → Format Message**
   3) Add **Google Gemini (LangChain)** node named **Gemini: Analyze**
      - Credentials: configure Google AI / Gemini credentials in n8n.
      - Model: **models/gemini-1.5-flash**
      - Message/prompt (single user message) that requests strict JSON:
        - Include the analyzed text: `{{ $json.userMessage }}`
        - Require output:
          `{ "sentiment": "...", "health_risk": true/false, "summary": "..." }`
      - Connect: **Format Message → Gemini: Analyze**
   4) Add **Code** node named **Parse JSON**
      - Use code equivalent to:
        - Read: `$input.first().json.content.parts[0].text`
        - Regex extract `{...}`
        - `JSON.parse` with fallback to Neutral/false on error
      - Connect: **Gemini: Analyze → Parse JSON**
   5) Add **Google Sheets** node named **Log to Sheets**
      - Credentials: Google Sheets OAuth2/service account with access to the target sheet.
      - Document ID: expression `{{ $('Config').first().json.SHEET_ID }}`
      - Sheet name: `ActivityLog`
      - Operation: **appendOrUpdate**
      - Map columns:
        - Date → `{{ $today.format('yyyy-MM-dd') }}`
        - Time → `{{ $now.format('HH:mm') }}`
        - Message → `{{ $('Format Message').first().json.userMessage }}`
        - Sentiment → `{{ $json.sentiment }}`
        - Alert → `{{ $json.health_risk ? 'YES' : 'NO' }}`
      - Connect: **Parse JSON → Log to Sheets**
   6) Add **IF** node named **Is Risk?**
      - Condition: boolean `{{ $json.health_risk }}` is **true**
      - Connect: **Log to Sheets → Is Risk?**
   7) Add **LINE** node named **Alert Child**
      - Credentials: LINE Messaging API channel access token (configure in n8n LINE credentials).
      - Resource: **message**
      - Set recipient to the child user ID (typically `to` = `{{ $('Config').first().json.CHILD_LINE_ID }}`).
      - Compose message text, e.g. include `sentiment` and `summary`.
      - Connect: **Is Risk? (true) → Alert Child**

4. **Daily watchdog branch (Schedule → Sheets → reminder → wait → re-check → escalation)**
   1) Add **Schedule Trigger** named **Daily 11 AM**
      - Trigger time: **11:00** (hour 11).
      - Connect: **Daily 11 AM → Config for Daily**
   2) Add **Google Sheets** node named **Read Logs**
      - Document ID: `{{ $('Config for Daily').first().json.SHEET_ID }}`
      - Sheet: `ActivityLog`
      - Enable **Always Output Data**.
      - Connect: **Config for Daily → Read Logs**
   3) Add **Code** node named **Check Activity**
      - Implement: read all rows; if any row has `Date === today`, output `{hasActivity:true}` else false.
      - Connect: **Read Logs → Check Activity**
   4) Add **IF** node named **If Inactive**
      - Condition: `{{ $json.hasActivity }}` is **false**
      - Connect: **Check Activity → If Inactive**
   5) Add **LINE** node named **Remind Parent**
      - Resource: **message**
      - Recipient: `{{ $('Config for Daily').first().json.PARENT_LINE_ID }}`
      - Message: gentle check-in request.
      - Connect: **If Inactive (true) → Remind Parent**
   6) Add **Wait** node named **Wait 3 Hours**
      - Amount: **3**
      - Unit: **hours**
      - Connect: **Remind Parent → Wait 3 Hours**
   7) Add **Google Sheets** node named **Read Logs Again**
      - Same settings as **Read Logs** (including Always Output Data).
      - Connect: **Wait 3 Hours → Read Logs Again**
   8) Add **Code** node named **Check Activity 2**
      - Same logic as **Check Activity**
      - Connect: **Read Logs Again → Check Activity 2**
   9) Add **IF** node named **If Still Inactive**
      - Condition: `{{ $json.hasActivity }}` is **false**
      - Connect: **Check Activity 2 → If Still Inactive**
   10) Add **LINE** node named **Emergency Alert**
      - Recipient: `{{ $('Config for Daily').first().json.CHILD_LINE_ID }}`
      - Message: escalation indicating no activity after reminder + 3 hours.
      - Connect: **If Still Inactive (true) → Emergency Alert**

5. **Google Sheet preparation**
   - Create a spreadsheet and a tab named **ActivityLog**
   - Add headers exactly: `Date`, `Time`, `Message`, `Sentiment`, `Alert`

6. **Credentials to configure**
   - **Google Sheets:** OAuth2 or service account; ensure access to the spreadsheet.
   - **LINE:** Messaging API credentials (channel access token) with permission to push messages to the specified user IDs.
   - **Gemini:** Google AI credentials supported by your n8n Gemini node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create Google Sheet headers: `Date`, `Time`, `Message`, `Sentiment`, `Alert`. | From “Elderly Care Monitor” sticky note |
| Configure LINE webhook in LINE Developers console pointing to the workflow’s Production URL. | From “Elderly Care Monitor” sticky note |
| Set `PARENT_LINE_ID`, `CHILD_LINE_ID`, `SHEET_ID` in the Config node(s). | From “Elderly Care Monitor” and “Configuration” sticky notes |
| Real-time monitoring logs messages and detects risks. | Sticky note “Real-time Monitoring” |
| Daily inactivity check runs at 11:00, reminds if silent, escalates after 3 hours. | Sticky note “Daily Inactivity Check” |