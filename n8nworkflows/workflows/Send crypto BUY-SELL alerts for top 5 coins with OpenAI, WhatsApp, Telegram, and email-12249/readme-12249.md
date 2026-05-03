Send crypto BUY/SELL alerts for top 5 coins with OpenAI, WhatsApp, Telegram, and email

https://n8nworkflows.xyz/workflows/send-crypto-buy-sell-alerts-for-top-5-coins-with-openai--whatsapp--telegram--and-email-12249


# Send crypto BUY/SELL alerts for top 5 coins with OpenAI, WhatsApp, Telegram, and email

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This n8n workflow monitors the **top 5 cryptocurrencies by market cap** (via CoinGecko), computes a simple **BUY/SELL/HOLD** signal based on **24h % price change**, enriches BUY/SELL alerts with **OpenAI-generated commentary**, and sends notifications via **WhatsApp, Telegram, and Email**.

**Target use cases:**
- Daily “market pulse” alerts on major coins
- Lightweight signal generation for retail monitoring (not a full trading system)
- Multi-channel broadcasting of AI-enhanced summaries

### Logical Blocks
**1.1 Scheduled Data Retrieval**
- Runs on a schedule and fetches top 5 coin market data.

**1.2 Field Normalization & Iteration**
- Keeps only relevant fields and iterates coin-by-coin.

**1.3 Signal Calculation**
- Computes BUY/SELL/HOLD from the 24h change threshold.

**1.4 Signal Gate (Filter)**
- Allows only BUY/SELL items to continue; HOLD is dropped.

**1.5 Message Formatting + AI Enrichment**
- Builds a formatted user message and asks OpenAI to produce a professional recommendation.

**1.6 Notification Delivery**
- Sends the final AI text to WhatsApp, Telegram, and Email (plus includes a Wait node in the flow).

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled Data Retrieval

**Overview:**  
Triggers the workflow daily (configured hour) and fetches the top 5 coins from CoinGecko markets endpoint with 24h change metrics.

**Nodes involved:**
- Schedule Trigger
- HTTP Request

#### Node: **Schedule Trigger**
- **Type / Role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point on a schedule.
- **Configuration (interpreted):**
  - Runs on an interval rule with `triggerAtHour: 8` (time zone depends on n8n instance settings).
- **Inputs/Outputs:**
  - No inputs (trigger).
  - Output → **HTTP Request**
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - Time zone mismatch (instance vs expected local time).
  - If instance is sleeping (some hosted/free tiers), scheduled executions may drift.

#### Node: **HTTP Request**
- **Type / Role:** `n8n-nodes-base.httpRequest` — fetch market data from CoinGecko.
- **Configuration (interpreted):**
  - `GET https://api.coingecko.com/api/v3/coins/markets`
  - Query params:
    - `vs_currency=usd`
    - `order=market_cap_desc`
    - `per_page=5`
    - `page=1`
    - `price_change_percentage=24h`
- **Key data dependencies:**
  - Expects fields like `name`, `symbol`, `current_price`, `market_cap`, `high_24h`, `low_24h`, `price_change_percentage_24h`.
- **Inputs/Outputs:**
  - Input ← Schedule Trigger
  - Output → **Selected Fields**
- **Version notes:** typeVersion `4.3`.
- **Edge cases / failures:**
  - CoinGecko rate limits (HTTP 429).
  - Temporary API downtime / non-200 responses.
  - Field availability varies by endpoint changes; missing `price_change_percentage_24h` handled later with fallback (but only after the Code node; the Set node still references it).

---

### 2.2 Field Normalization & Iteration

**Overview:**  
Reduces payload to a consistent schema and iterates through the list of returned coins.

**Nodes involved:**
- Selected Fields
- Loop Over Items

#### Node: **Selected Fields**
- **Type / Role:** `n8n-nodes-base.set` — maps/renames fields to a smaller payload.
- **Configuration (interpreted):**
  - Adds `timestamp = {{$now}}`
  - Copies/mirrors coin data:
    - `coin = {{$json.name}}`
    - `symbol = {{$json.symbol}}`
    - `price = {{$json.current_price}}`
    - `change24h = {{$json.market_cap_change_24h}}` *(note: this is market cap change, not price change %)*  
    - `marketCap = {{$json.market_cap}}`
    - `high24h = {{$json.high_24h}}`
    - `low24h = {{$json.low_24h}}`
    - `price_change_percentage_24h = {{$json.price_change_percentage_24h}}`
- **Inputs/Outputs:**
  - Input ← HTTP Request
  - Output → Loop Over Items
- **Version notes:** typeVersion `3.4`.
- **Edge cases / failures:**
  - Some fields may be `null` or missing depending on API response; downstream nodes must handle undefined values.
  - Potential semantic mismatch: `change24h` is not used later for the signal, and it references market cap change rather than price change %.

#### Node: **Loop Over Items**
- **Type / Role:** `n8n-nodes-base.splitInBatches` — iterates items (coins) one by one (batching/looping).
- **Configuration (interpreted):**
  - Default “split in batches” behavior (batch size not explicitly set here; n8n default applies).
- **Connections (important):**
  - Output **0** → Check Signal Status
  - Output **1** → Calculate Signal node
  - Additionally, **Calculate Signal node** loops back into **Loop Over Items**
- **Inputs/Outputs:**
  - Input ← Selected Fields
  - Output(s) → Check Signal Status, Calculate Signal node
- **Version notes:** typeVersion `3`.
- **Edge cases / failures:**
  - The wiring is unconventional: typically you calculate first, then IF-check. Here, the IF node may run on items **before** signal is computed (see Block 2.3/2.4).
  - Risk of infinite/incorrect looping if batching settings are not correct or if “continue” logic is miswired.

---

### 2.3 Signal Calculation

**Overview:**  
Computes BUY/SELL/HOLD using ±2% 24h price change thresholds, and prepares a redis-like key (though no Redis node exists in this workflow).

**Nodes involved:**
- Calculate Signal node

#### Node: **Calculate Signal node**
- **Type / Role:** `n8n-nodes-base.code` — transforms coin data into a signal object.
- **Configuration (interpreted logic):**
  - Reads current item: `const coin = $input.item.json;`
  - Extracts:
    - `symbol = coin.symbol.toUpperCase()`
    - `priceChange24h = coin.price_change_percentage_24h || 0`
    - `currentPrice = coin.current_price`
    - `marketCap = coin.market_cap`
  - Signal rules:
    - `BUY` if `priceChange24h <= -2`
    - `SELL` if `priceChange24h >= 2`
    - else `HOLD`
  - Adds:
    - `redisKey = crypto:signal:${symbol}`
    - `timestamp = new Date().toISOString()`
- **Inputs/Outputs:**
  - Input ← Loop Over Items (output 1)
  - Output → Loop Over Items (feeds back to continue processing)
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If the Set node renamed fields (e.g., `price_change_percentage_24h` as a string) but the Code node expects `coin.price_change_percentage_24h`, then `priceChange24h` becomes `0` and signals become biased to HOLD.  
    - In this workflow, **Selected Fields** creates `price_change_percentage_24h`, but the Code reads `price_change_percentage_24h` (same name) only if the item at that point is the Set output. However, the Code node uses `coin.current_price` / `coin.market_cap` which the Set node renamed to `price` / `marketCap` (strings). That mismatch can cause `currentPrice` and `marketCap` to become `undefined`.
  - If `symbol` is missing, `.toUpperCase()` throws.
  - The `redisKey` is not used downstream (no persistence / de-duplication is implemented).

---

### 2.4 Signal Gate (Filter)

**Overview:**  
Allows only items whose computed `signal` is BUY or SELL to proceed to alerting. HOLD items should stop here.

**Nodes involved:**
- Check Signal Status
- Sell output
- No Operation, do nothing

#### Node: **Check Signal Status**
- **Type / Role:** `n8n-nodes-base.if` — conditional filter.
- **Configuration (interpreted):**
  - Condition: `{{$json.signal}} == "SELL"` OR `{{$json.signal}} == "BUY"`
  - Strict type validation enabled (n8n IF v2 behavior).
- **Inputs/Outputs:**
  - Input ← Loop Over Items (output 0)
  - Output (true) → Sell output
  - Output (false) is not connected (items are dropped)
- **Version notes:** typeVersion `2.2`.
- **Edge cases / failures:**
  - If `signal` is not present yet (because calculation happened on a different branch), condition evaluates false → **no alerts ever**.
  - Strict validation can fail if `$json.signal` is `null/undefined` depending on settings; usually it just won’t match.

#### Node: **Sell output**
- **Type / Role:** `n8n-nodes-base.splitInBatches` — appears intended as a routing/iteration step for alerts.
- **Configuration (interpreted):**
  - Default options; no batch size specified.
- **Connections:**
  - Output 0 → No Operation, do nothing
  - Output 1 → Add user message
- **Inputs/Outputs:**
  - Input ← Check Signal Status (true branch)
  - Outputs → No Operation, Add user message
- **Version notes:** typeVersion `3`.
- **Edge cases / failures:**
  - Unclear semantics: This node name suggests “SELL output” but the IF passes BUY and SELL.
  - The use of SplitInBatches here is unusual; it may unintentionally skip processing if not configured to iterate as expected.

#### Node: **No Operation, do nothing**
- **Type / Role:** `n8n-nodes-base.noOp` — placeholder, effectively a dead-end.
- **Configuration:** none.
- **Inputs/Outputs:**
  - Input ← Sell output (output 0)
  - No outputs connected
- **Version notes:** typeVersion `1`.
- **Edge cases / failures:**
  - None; but it can unintentionally absorb items if the batch output routing is misused.

---

### 2.5 Message Formatting + AI Enrichment

**Overview:**  
Creates a preformatted alert message and uses OpenAI to generate a more human-readable recommendation, then fans out to notification channels.

**Nodes involved:**
- Add user message
- Human-readable Message
- OpenAI Chat Model

#### Node: **Add user message**
- **Type / Role:** `n8n-nodes-base.code` — constructs a message string placed in `$json.message`.
- **Configuration (interpreted logic):**
  - Always produces a **SELL-themed** template:
    - Title: `🔴 SELL SIGNAL CONFIRMED`
    - Includes: `Coin`, `24h Change`, `Time`
    - Text: “Strong upward move detected”
  - Uses:
    - `${$json.symbol}`
    - `${$json.priceChange24h}%`
    - `${$json.timestamp}`
- **Inputs/Outputs:**
  - Input ← Sell output (output 1)
  - Output → Human-readable Message
- **Version notes:** typeVersion `2`.
- **Edge cases / failures:**
  - If upstream item contains BUY signals, the message still says SELL (logic bug).
  - If `priceChange24h` doesn’t exist (e.g., different field name), the message shows `undefined%`.
  - If `timestamp` isn’t present, shows `undefined`.

#### Node: **Human-readable Message**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — LangChain “chain” node that sends prompt + user text to a connected chat model.
- **Configuration (interpreted):**
  - Takes input text from: `{{$json.message}}`
  - System/prompt message (static):
    - “You are a professional crypto trading assistant… Include the coin symbol, signal, and a short recommendation.”
  - `promptType: define` (prompt is explicitly defined in node)
- **Model connection:**
  - Uses **OpenAI Chat Model** via `ai_languageModel` connection.
- **Outputs:**
  - Main output fans out to: Wait, Send message (WhatsApp), Send a text message (Telegram), Send email
  - Produces `$json.text` commonly in these nodes (as referenced by WhatsApp/Email); exact field depends on node version, but workflow assumes `text`.
- **Version notes:** typeVersion `1.7` (LangChain node).
- **Edge cases / failures:**
  - If OpenAI model errors/timeouts, this node fails and no notifications send.
  - If the chain output field is not exactly `$json.text`, downstream nodes will send blank messages.

#### Node: **OpenAI Chat Model**
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model for the chain.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - Uses OpenAI API credentials.
- **Inputs/Outputs:**
  - Input is via `ai_languageModel` connector from Human-readable Message (reverse “provider” wiring).
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - Credential missing/invalid → auth failure.
  - Model name not available for the account/region.
  - Token limits if message grows (less likely here).

---

### 2.6 Notification Delivery

**Overview:**  
Distributes the AI-generated message to WhatsApp, Telegram, and Email. A Wait node is also connected, likely as a throttle or placeholder, and loops back into alert batching.

**Nodes involved:**
- Wait
- Send message (WhatsApp)
- Send a text message (Telegram)
- Send email

#### Node: **Send message (WhatsApp)**
- **Type / Role:** `n8n-nodes-base.whatsApp` — sends WhatsApp message.
- **Configuration (interpreted):**
  - Operation: `send`
  - Recipient phone number: `123456789` (placeholder)
  - Text body: `{{$json.text}}`
- **Credentials:** WhatsApp API account (provider depends on n8n node implementation; often Meta/Twilio-style integration).
- **Inputs/Outputs:**
  - Input ← Human-readable Message
  - No outputs connected
- **Version notes:** typeVersion `1.1`.
- **Edge cases / failures:**
  - Recipient format invalid (missing country code).
  - WhatsApp provider restrictions (template messages required, sandbox limitations).
  - If `$json.text` missing, sends empty/invalid payload.

#### Node: **Send a text message (Telegram)**
- **Type / Role:** `n8n-nodes-base.telegram` — sends Telegram message.
- **Configuration (interpreted):**
  - Parameters are incomplete in provided JSON (no explicit chatId/text shown).
  - `additionalFields: {}`
- **Inputs/Outputs:**
  - Input ← Human-readable Message
  - No outputs connected
- **Version notes:** typeVersion `1.2`.
- **Edge cases / failures:**
  - Missing credentials and required fields (chat ID, text) will cause execution failure.
  - If relying on defaults not shown, may not send anything.

#### Node: **Send email**
- **Type / Role:** `n8n-nodes-base.emailSend` — sends an email via SMTP.
- **Configuration (interpreted):**
  - To: `user@example.com`
  - From: `user@example.com`
  - Subject: `Crypto signals + system message + alerts`
  - HTML body: `{{$json.text}}`
- **Credentials:** SMTP account.
- **Inputs/Outputs:**
  - Input ← Human-readable Message
  - No outputs connected
- **Version notes:** typeVersion `2.1`.
- **Edge cases / failures:**
  - SMTP auth errors, blocked ports, TLS issues.
  - Spam filtering (from=to, missing DKIM/SPF, etc.).
  - `$json.text` missing yields blank email.

#### Node: **Wait**
- **Type / Role:** `n8n-nodes-base.wait` — pauses execution (commonly for rate-limiting or human approval).
- **Configuration:** no wait duration configured in parameters (defaults apply; may require resume via webhook depending on node mode).
- **Connections:**
  - Input ← Human-readable Message
  - Output → Sell output (loops back)
- **Version notes:** typeVersion `1.1`.
- **Edge cases / failures:**
  - If configured as “wait for webhook” implicitly, the workflow may stall indefinitely.
  - Looping back into Sell output can create unexpected repeated processing.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Entry point (scheduled run) | — | HTTP Request | ## Overview<br>This workflow is designed to monitor the Top 5 cryptocurrencies in real-time, calculate trading signals (BUY, SELL, HOLD), and send human-readable alerts through multiple channels. It integrates data fetching, signal processing, AI-generated insights, and multi-channel notifications to provide a professional-grade crypto monitoring solution.<br><br>## How It Works - Process Flow<br><br>1. Schedule the trigger<br>2. Fetch real-time coin data (CoinGecko, Binance API)<br>3. Filter only required fields<br>4. Check each data from loop<br>5. Add the logic for minimum percentage comparison<br>6. Use AI for analysis enhanced insights<br>8. Send the notification only if signal is 'SELL' or  'BUY' |
| HTTP Request | httpRequest | Fetch top 5 coins market data | Schedule Trigger | Selected Fields | ## Fetch data and field select<br>Fetch the latest coin data from crypto APi (CoinGecko, Binance or CoinMarketCap)<br>Use last 24h API endpoint to get last 24 data from selected coins<br><br>Select only relevant field from the endpoint like symbol, priceChange24h, and so. |
| Selected Fields | set | Normalize/select fields | HTTP Request | Loop Over Items | ## Fetch data and field select<br>Fetch the latest coin data from crypto APi (CoinGecko, Binance or CoinMarketCap)<br>Use last 24h API endpoint to get last 24 data from selected coins<br><br>Select only relevant field from the endpoint like symbol, priceChange24h, and so. |
| Loop Over Items | splitInBatches | Iterate through coins | Selected Fields / Calculate Signal node | Check Signal Status, Calculate Signal node |  |
| Calculate Signal node | code | Compute BUY/SELL/HOLD + enrich fields | Loop Over Items | Loop Over Items | ## Calculate Signal Node<br>This node applies the trading rules to each coins like BUY, SELL, HOLD<br><br>If this percentage increased by 2% this will alert to SELL and if percentage drop by 2% this will alert to BUY<br><br>You can adjust the percentage based on coin circulations. |
| Check Signal Status | if | Filter only BUY/SELL signals | Loop Over Items | Sell output | ## Check Signal Status<br>This node will allow to proceed only if signal status is 'SELL' or 'BUY'<br><br>HOLD coins will not be allowed to go further steps.<br><br>You can add logic for HOLD status coins if needed. |
| Sell output | splitInBatches | Routes alert items (batch/loop) | Check Signal Status | No Operation, do nothing / Add user message | ## Decision Routing<br>Once get the signal results this will pass to alerting pipeline<br><br>Add user message node prepares a formatted message with the coin, signal, 24h changes and time |
| No Operation, do nothing | noOp | Placeholder sink | Sell output | — | ## Decision Routing<br>Once get the signal results this will pass to alerting pipeline<br><br>Add user message node prepares a formatted message with the coin, signal, 24h changes and time |
| Add user message | code | Create formatted alert message | Sell output | Human-readable Message | ## Decision Routing<br>Once get the signal results this will pass to alerting pipeline<br><br>Add user message node prepares a formatted message with the coin, signal, 24h changes and time |
| Human-readable Message | chainLlm | Turn message into AI insight text | Add user message | Wait / Send message / Send a text message / Send email | ## AI Model<br>Using OpenAI API to enhances the alerts with human-readable insights and recommendations.<br><br>Consolidated all the signals into a readable format for notifications channels |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for the chain | — (connected via ai_languageModel) | Human-readable Message (provider link) | ## AI Model<br>Using OpenAI API to enhances the alerts with human-readable insights and recommendations.<br><br>Consolidated all the signals into a readable format for notifications channels |
| Wait | wait | Pause/throttle + loopback | Human-readable Message | Sell output | ## Notifications<br>Send alert through multiple channels.<br><br>1. Whatsapp<br>2. Telegram<br>3. Email<br><br>professional multiple channel delivery. |
| Send message | whatsApp | Send WhatsApp notification | Human-readable Message | — | ## Notifications<br>Send alert through multiple channels.<br><br>1. Whatsapp<br>2. Telegram<br>3. Email<br><br>professional multiple channel delivery. |
| Send a text message | telegram | Send Telegram notification | Human-readable Message | — | ## Notifications<br>Send alert through multiple channels.<br><br>1. Whatsapp<br>2. Telegram<br>3. Email<br><br>professional multiple channel delivery. |
| Send email | emailSend | Send Email notification | Human-readable Message | — | ## Notifications<br>Send alert through multiple channels.<br><br>1. Whatsapp<br>2. Telegram<br>3. Email<br><br>professional multiple channel delivery. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
- Set it to run daily at **hour 8** (choose timezone in n8n settings as needed).
- Connect to **HTTP Request**.

2) **Create “HTTP Request” (HTTP Request node)**
- Method: GET  
- URL: `https://api.coingecko.com/api/v3/coins/markets`
- Enable “Send Query Parameters” and add:
  - `vs_currency` = `usd`
  - `order` = `market_cap_desc`
  - `per_page` = `5`
  - `page` = `1`
  - `price_change_percentage` = `24h`
- Connect to **Selected Fields**.

3) **Create “Selected Fields” (Set node)**
- Add fields (as expressions from incoming JSON):
  - `timestamp` = `{{$now}}`
  - `coin` = `{{$json.name}}`
  - `symbol` = `{{$json.symbol}}`
  - `price` = `{{$json.current_price}}`
  - `change24h` = `{{$json.market_cap_change_24h}}`
  - `marketCap` = `{{$json.market_cap}}`
  - `high24h` = `{{$json.high_24h}}`
  - `low24h` = `{{$json.low_24h}}`
  - `price_change_percentage_24h` = `{{$json.price_change_percentage_24h}}`
- Connect to **Loop Over Items**.

4) **Create “Loop Over Items” (Split In Batches node)**
- Keep default settings (or explicitly set batch size 1).
- Connect its outputs exactly like the JSON:
  - Output 0 → **Check Signal Status**
  - Output 1 → **Calculate Signal node**

5) **Create “Calculate Signal node” (Code node)**
- Paste logic to compute signal using 24h % thresholds (±2%).
- Output the computed fields (`signal`, `priceChange24h`, etc.).
- Connect **Calculate Signal node** → **Loop Over Items** (to continue the loop).

6) **Create “Check Signal Status” (IF node)**
- Condition group: OR
  - `{{$json.signal}}` equals `SELL`
  - `{{$json.signal}}` equals `BUY`
- Connect **true** output → **Sell output**.
- Leave false output unconnected (drops HOLD).

7) **Create “Sell output” (Split In Batches node)**
- Keep default settings.
- Connect:
  - Output 0 → **No Operation, do nothing**
  - Output 1 → **Add user message**

8) **Create “No Operation, do nothing” (NoOp node)**
- No config; leave as sink.

9) **Create “Add user message” (Code node)**
- Build a formatted text payload in `$json.message` using incoming fields (symbol, priceChange24h, timestamp).
- Connect to **Human-readable Message**.

10) **Create “OpenAI Chat Model” (OpenAI Chat Model node)**
- Select model: `gpt-4.1-mini`
- Configure **OpenAI credentials** in n8n:
  - Add OpenAI API key in Credentials → OpenAI.
- This node will be connected to the chain node via `ai_languageModel`.

11) **Create “Human-readable Message” (LangChain Chain LLM node)**
- Prompt type: “define”
- System/prompt message: professional crypto trading assistant instruction (as in workflow).
- Text input: `{{$json.message}}`
- Connect **OpenAI Chat Model** to **Human-readable Message** using the **AI Language Model** connector.
- Connect its main output to all notification nodes:
  - **Wait**
  - **Send message** (WhatsApp)
  - **Send a text message** (Telegram)
  - **Send email**

12) **Create “Send message” (WhatsApp node)**
- Operation: send
- Recipient phone: set your real number (E.164 format recommended, e.g. `+14155552671`)
- Text body: `{{$json.text}}`
- Configure WhatsApp credentials (provider-specific as supported by your n8n installation).

13) **Create “Send a text message” (Telegram node)**
- Configure Telegram credentials (Bot token).
- Set required fields (commonly Chat ID and Text). Use `{{$json.text}}` as the message text.

14) **Create “Send email” (Email Send node)**
- Configure SMTP credentials (host/port/user/pass/TLS).
- To/From/Subject as desired.
- HTML body: `{{$json.text}}`

15) **Create “Wait” (Wait node)**
- Add Wait node (choose mode carefully: time-based vs webhook resume).
- Connect **Wait** → **Sell output** (as in the provided workflow).

**Sub-workflows:** none (no Execute Workflow node present).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Fetch real-time coin data (CoinGecko, Binance API)” is mentioned in notes, but the implementation uses CoinGecko only. | Sticky note “Overview” |
| Signal logic is ±2% 24h change threshold; adjustable based on preference. | Sticky note “Calculate Signal Node” |
| Alerts are intended only for BUY/SELL; HOLD items are dropped. | Sticky note “Check Signal Status” |
| Notifications include WhatsApp, Telegram, Email. | Sticky note “Notifications” |
| The current wiring/field naming suggests potential mismatches (Set node renames fields but Code node expects CoinGecko raw keys). | Derived from node configs |
| The “Add user message” content is SELL-specific even for BUY signals. | Derived from node code |

