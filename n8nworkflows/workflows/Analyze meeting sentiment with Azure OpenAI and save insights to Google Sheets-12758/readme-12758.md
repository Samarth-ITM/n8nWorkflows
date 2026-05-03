Analyze meeting sentiment with Azure OpenAI and save insights to Google Sheets

https://n8nworkflows.xyz/workflows/analyze-meeting-sentiment-with-azure-openai-and-save-insights-to-google-sheets-12758


# Analyze meeting sentiment with Azure OpenAI and save insights to Google Sheets

## 1. Workflow Overview

This workflow (“**Smart Trade Alert Dispatcher for Slack & Telegram**”) monitors **Apple (AAPL)** market data using the **TwelveData API** (RSI + quote/volume), derives simple technical signals, asks **GPT-4o** to convert those signals into a concise human-readable update, then **routes alerts** based on computed priority: **HIGH** to Telegram and **MEDIUM/LOW** to Slack. It also includes **global error handling** that emails a failure notification.

> Note: The title you provided (“Analyze meeting sentiment with Azure OpenAI and save insights to Google Sheets”) does not match the JSON workflow content. The JSON is a stock-alert workflow using TwelveData + OpenAI + Slack/Telegram (no Azure OpenAI, no Google Sheets).

### 1.1 Scheduled Trigger (Entry Point)
Runs on a schedule (default appears to be “every minute” per sticky note guidance) and starts data fetching.

### 1.2 Data Collection (Parallel HTTP Calls)
Fetches RSI and quote/volume data in parallel from TwelveData, then merges the results.

### 1.3 Signal Processing (Deterministic Rules)
Computes momentum classification, volume signal, and a suggested action.

### 1.4 AI Market Summary (LLM Formatting)
Uses GPT-4o via LangChain Agent to generate a Slack-ready update from structured signals.

### 1.5 Priority Routing & Alerts (Slack vs Telegram)
Assigns HIGH/MEDIUM/LOW priority and sends to Telegram for HIGH; otherwise sends to Slack.

### 1.6 Error Handling (Separate Entry)
On any workflow error, triggers an email alert via Gmail.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Scheduled Trigger (Entry Point)
**Overview:** Starts the workflow on a schedule and fans out into two parallel HTTP requests for market data.  
**Nodes involved:** `Schedule Trigger1`

#### Node: Schedule Trigger1
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based trigger.
- **Configuration (interpreted):** Uses an interval-based schedule. The JSON has `rule.interval: [{}]` (template-like); the sticky note suggests **every minute**.
- **Connections:**
  - **Output →** `Fetch Price & Volume Data`
  - **Output →** `Fetch RSI Indicator` (parallel branch)
- **Edge cases / failures:**
  - Misconfigured interval could run too frequently (API rate limits) or not at all.
  - Timezone considerations if later extended to time-window trading hours.

---

### Block 2.2 — Data Collection (TwelveData API)
**Overview:** Pulls RSI(14) on 5-minute candles and current quote/volume, then merges both payloads for downstream logic.  
**Nodes involved:** `Fetch RSI Indicator`, `Fetch Price & Volume Data`, `Merge API Responses`

#### Node: Fetch RSI Indicator
- **Type / role:** `n8n-nodes-base.httpRequest` — calls TwelveData RSI endpoint.
- **Configuration choices:**
  - GET URL: `https://api.twelvedata.com/rsi?symbol=AAPL&interval=5min&time_period=14&apikey=YOUR_TOKEN_HERE`
  - No extra options configured.
- **Output expectations:** TwelveData RSI response typically includes `meta.indicator.name` and `values[0].rsi`.
- **Connections:**
  - **Output →** `Merge API Responses` (input index 0)
- **Edge cases / failures:**
  - Invalid/expired API key → 401/4xx response.
  - Rate limiting on free tier.
  - Response shape changes or missing `values[0]` (e.g., market closed, insufficient data).

#### Node: Fetch Price & Volume Data
- **Type / role:** `n8n-nodes-base.httpRequest` — calls TwelveData quote endpoint.
- **Configuration choices:**
  - GET URL: `https://api.twelvedata.com/quote?symbol=AAPL&apikey=YOUR_TOKEN_HERE`
- **Output expectations:** Quote response commonly includes `symbol`, `close`, `volume`, `average_volume`.
- **Connections:**
  - **Output →** `Merge API Responses` (input index 1)
- **Edge cases / failures:**
  - Same as above (auth/rate limits).
  - Missing `average_volume` for some symbols or tiers.
  - Values returned as strings → downstream code converts to `Number()` (can become `NaN` if unexpected).

#### Node: Merge API Responses
- **Type / role:** `n8n-nodes-base.merge` — combines parallel API outputs into one stream for processing.
- **Configuration choices:** Default merge behavior (no explicit mode shown).
- **Connections:**
  - **Inputs:** RSI branch + Quote branch
  - **Output →** `Calculate Trading Signals`
- **Edge cases / failures:**
  - If one branch errors or returns zero items, merge behavior may yield incomplete input for the Code node.
  - If merge mode is not “wait for both” (depends on node defaults/version), it may pass through one side only.

---

### Block 2.3 — Signal Processing (Rules)
**Overview:** Converts RSI + volume data into normalized fields: momentum class, volume signal, and an action label.  
**Nodes involved:** `Calculate Trading Signals`

#### Node: Calculate Trading Signals
- **Type / role:** `n8n-nodes-base.code` — custom JS transformation.
- **Configuration choices (logic):**
  - Reads all merged items: `const items = $input.all();`
  - **RSI extraction:** finds item where `json.meta.indicator.name` includes `"RSI"`, then `Number(values[0].rsi)`.
  - **Price extraction:** finds item where `json.symbol` exists, then reads `close`, `volume`, `average_volume`.
  - **Momentum buckets:**
    - `< 30` → `OVERSOLD`
    - `< 35` → `NEAR_OVERSOLD`
    - `< 60` → `NEUTRAL`
    - `< 70` → `NEAR_OVERBOUGHT`
    - otherwise → `OVERBOUGHT`
  - **Volume signal:** `HIGH_VOLUME` if `volume > avgVolume`, else `NORMAL_VOLUME`
  - **Action:**
    - `OVERSOLD` → `PREPARE_BUY`
    - `NEAR_OVERSOLD` → `WATCH`
    - else → `WAIT`
  - Outputs a single item with:
    - `symbol, price, rsi, momentum, volume, avgVolume, volumeSignal, action`
- **Connections:**
  - **Input:** `Merge API Responses`
  - **Output →** `Generate AI Summary`
- **Edge cases / failures:**
  - If RSI item not found: `rsi = null` → momentum remains `UNKNOWN` (handled).
  - If quote item not found: `priceItem` becomes undefined, but the code later uses `priceItem.json.symbol` (this would throw). In practice, if the quote API fails or merge doesn’t include it, the workflow can crash here.
  - If numeric conversion yields `NaN`, comparisons may behave unexpectedly.

---

### Block 2.4 — AI Market Analysis (GPT-4o via LangChain)
**Overview:** Uses an LLM agent to generate a concise “Slack-ready” market update from structured signals, with a strict system message to avoid financial advice.  
**Nodes involved:** `Generate AI Summary`, `OpenAI GPT-4 Model`

#### Node: OpenAI GPT-4 Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model to the agent.
- **Configuration choices:**
  - Model selected: **`gpt-4o`**
  - Responses API disabled (`responsesApiEnabled: false`)
- **Connections:**
  - **AI language model output →** `Generate AI Summary` (as its language model provider)
- **Version-specific notes:**
  - Requires the LangChain nodes package available in your n8n instance.
- **Edge cases / failures:**
  - Missing/invalid OpenAI credentials.
  - Model access restrictions, quota exhaustion, or transient 5xx errors.

#### Node: Generate AI Summary
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompts the model and generates formatted text.
- **Configuration choices:**
  - Prompt instructs:
    - “Slack-ready market update”
    - Short title, bullet points using `•`, **bold Action line**, no markdown headers.
  - Inserts live fields via expressions like `{{$json.symbol}}`, `{{$json.rsi}}`, etc.
  - Agent options:
    - `maxIterations: 30`
    - System message: professional, concise, neutral; **no financial advice**, no exaggeration.
- **Outputs:**
  - Downstream nodes reference `{{$json.output}}`, implying this agent returns its final text in an `output` field.
- **Connections:**
  - **Input:** `Calculate Trading Signals`
  - **Output →** `Assign Alert Priority`
- **Edge cases / failures:**
  - If upstream fields are null/NaN, output quality may degrade (LLM may “smooth over” missing data unless prompted to handle nulls).
  - Token limits if prompt or outputs get extended (unlikely here).
  - If the agent output field name differs by version/config, Slack/Telegram nodes may send blank messages.

---

### Block 2.5 — Priority Routing & Notifications
**Overview:** Computes alert priority based on RSI and volume conditions, then routes HIGH priority to Telegram and everything else to Slack.  
**Nodes involved:** `Assign Alert Priority`, `Route by Priority`, `Send to Telegram (High Priority)`, `Send to Slack (Low Priority)`

#### Node: Assign Alert Priority
- **Type / role:** `n8n-nodes-base.code` — derives `priority`.
- **Logic:**
  - Default: `LOW`
  - If `rsi < 30` and `volumeSignal === "HIGH_VOLUME"` → `HIGH`
  - Else if `rsi < 40` **or** `volumeSignal === "HIGH_VOLUME"` → `MEDIUM`
  - Outputs original JSON + `priority`
- **Connections:**
  - **Input:** `Generate AI Summary`
  - **Output →** `Route by Priority`
- **Edge cases / failures:**
  - If `rsi` is null/NaN, comparisons may evaluate to `false` and remain `LOW`, potentially suppressing urgent alerts.

#### Node: Route by Priority
- **Type / role:** `n8n-nodes-base.if` — conditional branching.
- **Condition:** `{{$json.priority}} equals "HIGH"` (strict validation enabled).
- **Connections:**
  - **True (HIGH) →** `Send to Telegram (High Priority)`
  - **False (not HIGH) →** `Send to Slack (Low Priority)`
- **Edge cases / failures:**
  - If `priority` missing or not a string, strict validation may cause unexpected false branch (or evaluation issues).

#### Node: Send to Telegram (High Priority)
- **Type / role:** `n8n-nodes-base.telegram` — sends urgent alert to Telegram chat.
- **Configuration choices:**
  - Chat ID: `YOUR_TELEGRAM_CHAT_ID` (must be replaced)
  - Message text:
    - Includes `🚨`, priority, `{{$json.output}}`, and `_Action:_ {{$json.action}}`
  - Uses Telegram credentials (bot token) referenced as “Telegram account -anuj”.
- **Connections:** Input from `Route by Priority` (true branch).
- **Edge cases / failures:**
  - Wrong chat ID, bot not in chat, or lacking permission → message fails.
  - Telegram formatting: message includes markdown-like asterisks/underscores; parse mode isn’t explicitly set—formatting may not render as intended depending on node defaults.

#### Node: Send to Slack (Low Priority)
- **Type / role:** `n8n-nodes-base.slack` — posts non-urgent updates to a Slack channel.
- **Configuration choices:**
  - Sends `text: {{$json.output}}`
  - Channel selection by ID: `YOUR_SLACK_CHANNEL_ID` (must be replaced)
- **Connections:** Input from `Route by Priority` (false branch).
- **Edge cases / failures:**
  - OAuth scopes missing (e.g., `chat:write`) or invalid token.
  - Incorrect channel ID or bot not added to channel.

---

### Block 2.6 — Error Handling
**Overview:** If any node errors during execution, sends an email containing node name, error message, and timestamp.  
**Nodes involved:** `Workflow Error Handler`, `Send a message1`

#### Node: Workflow Error Handler
- **Type / role:** `n8n-nodes-base.errorTrigger` — starts when workflow execution fails.
- **Connections:**
  - **Output →** `Send a message1`
- **Edge cases / failures:**
  - Only triggers on unhandled workflow errors; if nodes are configured to “continue on fail”, errors may not trigger this path.

#### Node: Send a message1
- **Type / role:** `n8n-nodes-base.gmail` — sends an email alert.
- **Configuration choices:**
  - To: `user@example.com` (placeholder)
  - Subject: “Workflow Error Alert”
  - Body includes expressions:
    - Error node: `{{ $json.node.name }}`
    - Error message: `{{ $json.error.message }}`
    - Timestamp: `{{ $now.toISO() }}`
  - Email type: `text` (so markdown styling in message won’t render as rich text)
- **Credentials:** Gmail OAuth2 required.
- **Connections:** Input from `Workflow Error Handler`.
- **Edge cases / failures:**
  - Gmail OAuth token expired/revoked.
  - Sending limits or blocked “from” configuration depending on Google policies.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger1 | scheduleTrigger | Scheduled entry point | — | Fetch Price & Volume Data; Fetch RSI Indicator | (From “Overview Sticky”) Monitors AAPL using RSI + price; GPT summary; routes Telegram vs Slack; setup steps incl. schedule interval. |
| Fetch RSI Indicator | httpRequest | Fetch RSI(14) 5min from TwelveData | Schedule Trigger1 | Merge API Responses | (From “Data Collection Section”) Fetches RSI + price/volume in parallel, then merges. |
| Fetch Price & Volume Data | httpRequest | Fetch quote + volume from TwelveData | Schedule Trigger1 | Merge API Responses | (From “Data Collection Section”) Fetches RSI + price/volume in parallel, then merges. |
| Merge API Responses | merge | Combine API payloads | Fetch RSI Indicator; Fetch Price & Volume Data | Calculate Trading Signals | (From “Data Collection Section”) Fetches RSI + price/volume in parallel, then merges. |
| Calculate Trading Signals | code | Compute momentum/volume/action | Merge API Responses | Generate AI Summary | (From “Signal Processing Section”) Classifies momentum, compares volume, determines action. |
| OpenAI GPT-4 Model | lmChatOpenAi | LLM model provider (gpt-4o) | — | Generate AI Summary (AI model connection) | (From “AI Analysis Section”) GPT produces neutral, factual Slack-ready summaries; no advice. |
| Generate AI Summary | langchain agent | Generate formatted market update text | Calculate Trading Signals | Assign Alert Priority | (From “AI Analysis Section”) GPT produces neutral, factual Slack-ready summaries; no advice. |
| Assign Alert Priority | code | Compute HIGH/MEDIUM/LOW priority | Generate AI Summary | Route by Priority | (From “Alerts Section”) HIGH to Telegram; otherwise Slack; based on RSI + volume. |
| Route by Priority | if | Branch on priority == HIGH | Assign Alert Priority | Send to Telegram (High Priority); Send to Slack (Low Priority) | (From “Alerts Section”) HIGH to Telegram; otherwise Slack; based on RSI + volume. |
| Send to Telegram (High Priority) | telegram | Send urgent alert to Telegram | Route by Priority | — | (From “Alerts Section”) HIGH to Telegram; otherwise Slack; based on RSI + volume. |
| Send to Slack (Low Priority) | slack | Send update to Slack channel | Route by Priority | — | (From “Alerts Section”) HIGH to Telegram; otherwise Slack; based on RSI + volume. |
| Workflow Error Handler | errorTrigger | Error entry point | — | Send a message1 | (From “Sticky Note7”) Error Handling: Sends alerts when the workflow fails. |
| Send a message1 | gmail | Email error notification | Workflow Error Handler | — | (From “Sticky Note7”) Error Handling: Sends alerts when the workflow fails. |

Additional sticky note not tied to a specific node region but relevant:
- “Security Notice”: credential requirements and reminder to remove personal IDs before sharing.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Smart Trade Alert Dispatcher for Slack & Telegram** (or your preferred name).

2. **Add the schedule trigger**
   - Add node: **Schedule Trigger**
   - Configure an interval (recommended per notes: **every 1 minute**).
   - This is the main entry point.

3. **Add TwelveData HTTP Request nodes (parallel)**
   1) Add node: **HTTP Request** → rename to **Fetch RSI Indicator**
   - Method: GET  
   - URL: `https://api.twelvedata.com/rsi?symbol=AAPL&interval=5min&time_period=14&apikey=YOUR_TOKEN_HERE`
   - Replace `YOUR_TOKEN_HERE` with your TwelveData key (or use an expression/credential pattern you prefer).

   2) Add node: **HTTP Request** → rename to **Fetch Price & Volume Data**
   - Method: GET  
   - URL: `https://api.twelvedata.com/quote?symbol=AAPL&apikey=YOUR_TOKEN_HERE`

   3) Connect **Schedule Trigger → Fetch RSI Indicator**  
   4) Connect **Schedule Trigger → Fetch Price & Volume Data**  
   (This creates the parallel fan-out.)

4. **Merge the API responses**
   - Add node: **Merge** → rename to **Merge API Responses**
   - Connect:
     - **Fetch RSI Indicator → Merge API Responses** (Input 1)
     - **Fetch Price & Volume Data → Merge API Responses** (Input 2)

5. **Compute trading signals**
   - Add node: **Code** → rename to **Calculate Trading Signals**
   - Paste the logic (adapt as needed) that:
     - extracts RSI from the RSI response,
     - extracts price/volume from quote response,
     - computes `momentum`, `volumeSignal`, `action`,
     - returns one consolidated item.
   - Connect **Merge API Responses → Calculate Trading Signals**.

6. **Add the OpenAI chat model (LangChain)**
   - Add node: **OpenAI Chat Model** (LangChain) → rename to **OpenAI GPT-4 Model**
   - Select model: **gpt-4o**
   - Configure **OpenAI credentials** (API key) in n8n Credentials.

7. **Add the AI Agent to generate the summary**
   - Add node: **AI Agent** (LangChain) → rename to **Generate AI Summary**
   - Set prompt type to “Define” (or equivalent) and use a prompt that injects:
     - `symbol, price, rsi, momentum, volume, avgVolume, volumeSignal, action`
   - Set **System message** to enforce:
     - professional, concise, neutral, factual,
     - no financial advice, no exaggeration.
   - Connect **OpenAI GPT-4 Model → Generate AI Summary** using the node’s **AI Language Model** connection type.
   - Connect **Calculate Trading Signals → Generate AI Summary** (main connection).

8. **Assign priority**
   - Add node: **Code** → rename to **Assign Alert Priority**
   - Implement:
     - HIGH if `rsi < 30` and `volumeSignal == HIGH_VOLUME`
     - MEDIUM if `rsi < 40` OR `volumeSignal == HIGH_VOLUME`
     - else LOW
   - Connect **Generate AI Summary → Assign Alert Priority**.

9. **Route by priority**
   - Add node: **IF** → rename to **Route by Priority**
   - Condition: `priority` equals `HIGH`
   - Connect **Assign Alert Priority → Route by Priority**.

10. **Send notifications**
   1) **Telegram**
   - Add node: **Telegram** → rename to **Send to Telegram (High Priority)**
   - Configure Telegram credentials (bot token).
   - Set **Chat ID** (replace `YOUR_TELEGRAM_CHAT_ID`).
   - Text should include `{{$json.output}}` and optionally `{{$json.action}}`.
   - Connect **Route by Priority (true) → Send to Telegram (High Priority)**.

   2) **Slack**
   - Add node: **Slack** → rename to **Send to Slack (Low Priority)**
   - Configure Slack OAuth2 credentials with `chat:write`.
   - Select channel by ID (replace `YOUR_SLACK_CHANNEL_ID`).
   - Text: `{{$json.output}}`
   - Connect **Route by Priority (false) → Send to Slack (Low Priority)**.

11. **Add workflow-wide error email notifications**
   1) Add node: **Error Trigger** → rename to **Workflow Error Handler**
   2) Add node: **Gmail** → rename to **Send a message1**
   - Configure Gmail OAuth2 credentials.
   - To: your email address
   - Subject: “Workflow Error Alert”
   - Message body using:
     - `{{$json.node.name}}`
     - `{{$json.error.message}}`
     - `{{$now.toISO()}}`
   3) Connect **Workflow Error Handler → Send a message1**

12. **Test**
   - Manually execute once to confirm TwelveData responses merge correctly and that `output` exists from the agent.
   - Then activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow monitors Apple stock (AAPL) in real-time using RSI indicators and price data from TwelveData API… High-priority signals trigger immediate Telegram alerts, while lower-priority updates go to Slack.” | From “Overview Sticky” |
| Setup steps: add TwelveData key to both HTTP nodes; configure OpenAI; connect Telegram + Slack; update chat/channel IDs; adjust schedule interval; test manually before enabling. | From “Overview Sticky” |
| Required credentials: TwelveData API, OpenAI API, Telegram Bot Token, Slack OAuth2. Remove personal IDs before sharing. | From “Security Notice” |
| Error Handling: Sends alerts when the workflow fails. | From “Sticky Note7” |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.