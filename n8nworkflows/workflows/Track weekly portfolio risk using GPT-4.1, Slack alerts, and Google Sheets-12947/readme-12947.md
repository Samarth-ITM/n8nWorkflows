Track weekly portfolio risk using GPT-4.1, Slack alerts, and Google Sheets

https://n8nworkflows.xyz/workflows/track-weekly-portfolio-risk-using-gpt-4-1--slack-alerts--and-google-sheets-12947


# Track weekly portfolio risk using GPT-4.1, Slack alerts, and Google Sheets

## 1. Workflow Overview

**Purpose:** This workflow runs weekly to assess a stock portfolio’s risk profile (sector concentration, volatility, and correlations) using historical daily prices from Alpha Vantage. If risk is detected, it generates a short AI explanation (GPT-4.1-mini via LangChain) and sends a Slack alert, then appends a weekly snapshot to Google Sheets.

**Target use cases:**
- Ongoing monitoring of diversification risk for a personal/team portfolio
- Lightweight risk scoring for weekly reporting
- Slack-based alerting with an auditable history in Google Sheets

### 1.1 Scheduling & Configuration
A weekly trigger starts the workflow, then a Set node defines thresholds (sector concentration limit, correlation threshold, etc.) and toggles (alerts/AI summary).

### 1.2 Portfolio Input (Google Sheets) + Validation
Reads the “Portfolio” tab from a Google Sheet and validates required columns and values. Normalizes rows to a consistent internal schema.

### 1.3 Market Data Collection (Batching + Rate Limiting)
Processes stocks in batches, fetching daily prices from Alpha Vantage. A wait buffer reduces rate-limit errors. Market data is normalized into prices/returns.

### 1.4 Risk Computation
Aggregates all normalized stock data and computes:
- Sector concentration vs threshold
- Per-stock volatility and average portfolio volatility
- Pairwise correlations vs threshold  
Outputs `risk_detected` and a `risk_score` (0–100).

### 1.5 Conditional AI Summary + Alerting + Storage
If risk is detected, generates a brief AI summary, builds a payload, sends a Slack alert, and appends a weekly record to “Weekly Risk Snapshots” in Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Risk Settings
**Overview:** Starts weekly and defines thresholds/toggles used later in the risk engine and alerting logic.

**Nodes involved:**
- Weekly Schedule Trigger
- Risk Thresholds and Settings

#### Node: **Weekly Schedule Trigger**
- **Type / role:** `Schedule Trigger` — entry point.
- **Config choices:** Runs on a weekly interval at **09:00** (server/account timezone as configured in n8n).
- **Outputs to:** Risk Thresholds and Settings
- **Edge cases / failures:**
  - Timezone mismatches can cause unexpected run times.
  - Workflow is currently **inactive** (`active: false`), so it won’t run until enabled.

#### Node: **Risk Thresholds and Settings**
- **Type / role:** `Set` — central configuration.
- **Config choices (key fields):**
  - `sector_concentration_limit`: **35** (%)
  - `correlation_threshold`: **0.75**
  - `volatility_lookback_days`: **30** (present but not actually used in code)
  - `volatility_alert_change_pct`: **10** (present but not used in code)
  - `enable_alerts`: **true** (present but not used to gate Slack node)
  - `enable_ai_summary`: **true** (present but not used to gate AI node)
  - `base_currency`: **INR** (present but not used in calculations)
- **Outputs to:** Read Portfolio Sheet
- **Edge cases / failures:**
  - The workflow logic does **not** currently branch on `enable_alerts` / `enable_ai_summary`, so toggling them has no effect unless you add IF nodes.

---

### Block 2 — Portfolio Input & Validation (Google Sheets)
**Overview:** Loads portfolio rows from Google Sheets and ensures required fields are present and valid before proceeding.

**Nodes involved:**
- Read Portfolio Sheet
- Validate Portfolio Input

#### Node: **Read Portfolio Sheet**
- **Type / role:** `Google Sheets` — read portfolio table.
- **Config choices (interpreted):**
  - Document: **“NEWS IMPACT TRACKER”** (Spreadsheet ID `1Mz-woY...`)
  - Sheet/tab: **“Portfolio”**
  - Operation: implied read (the node configuration shown is consistent with reading rows; no append/update set)
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`)
- **Outputs to:** Validate Portfolio Input
- **Edge cases / failures:**
  - OAuth expiry/permission errors (most common).
  - Sheet/tab renamed or gid changed.
  - Empty range if headers mismatch or sheet has no rows.

#### Node: **Validate Portfolio Input**
- **Type / role:** `Code` — validates schema and normalizes output.
- **Config choices (logic):**
  - Fails if sheet is empty.
  - Requires columns: `Symbol`, `Sector`, `Quantity`
  - Quantity must be `> 0`
  - Normalizes each row to:
    - `symbol`: trimmed, uppercased
    - `sector`: trimmed
    - `quantity`: number
    - `avg_price`: optional numeric from “Avg Price” column or `null`
- **Outputs to:** Batch Portfolio Symbols
- **Key variables/expressions:** uses `items.map(item => item.json)`
- **Edge cases / failures:**
  - If headers differ even slightly (e.g., “SYMBOL”), validation fails.
  - Non-numeric Quantity becomes `NaN` → `Number(row.Quantity) <= 0` evaluates `false`? (Actually `NaN <= 0` is `false`, so a non-numeric could slip through. Consider explicitly checking `Number.isFinite()`.)
  - “Avg Price” parsing to number may become `NaN` if formatted unexpectedly.

---

### Block 3 — Market Data Fetching, Rate Limiting & Normalization
**Overview:** Iterates through portfolio symbols in batches, fetches daily prices from Alpha Vantage, waits to avoid throttling, and transforms raw API output into consistent arrays for analysis.

**Nodes involved:**
- Batch Portfolio Symbols
- Fetch Market Data
- Rate Limit Buffer (Alpha Vantage)
- Normalize Alpha Vantage Data

#### Node: **Batch Portfolio Symbols**
- **Type / role:** `Split In Batches` — controls iteration over items.
- **Config choices:** Batch size not explicitly set (defaults apply; in n8n, default is typically 1 unless configured).
- **Connections (important):**
  - **Output 1** goes to **Portfolio Risk Engine** (this is unusual; see edge case below)
  - **Output 2** goes to **Fetch Market Data**
  - Receives items again from **Normalize Alpha Vantage Data** to continue batching loop
- **Edge cases / failures:**
  - The wiring suggests the node feeds both the risk engine and market fetch. In typical patterns, the risk engine should run **after** all items are enriched/normalized, not in parallel with fetching.
  - However, given that **Normalize Alpha Vantage Data** loops back into this node, the intention is: fetch→wait→normalize→loop until complete, then on completion proceed. In n8n, `SplitInBatches` commonly sends remaining items iteratively and then stops; but connecting output 1 directly to the risk engine can cause the risk engine to run **per batch** (or before all symbols are processed), depending on n8n execution behavior/version.

#### Node: **Fetch Market Data**
- **Type / role:** `HTTP Request` — calls Alpha Vantage daily time series.
- **Config choices:**
  - URL: `https://www.alphavantage.co/query`
  - Query params:
    - `function=TIME_SERIES_DAILY`
    - `symbol={{$json.symbol}}`
    - `apikey=Q75H5WJH88KACPLP` (hard-coded)
    - `outputsize=compact`
- **Outputs to:** Rate Limit Buffer (Alpha Vantage)
- **Edge cases / failures:**
  - **Rate limits** (Alpha Vantage is strict). You may receive “Thank you for using Alpha Vantage! … frequency” messages or empty payloads.
  - Invalid symbols return payloads without `"Time Series (Daily)"`.
  - Hard-coded API key is a security risk; prefer n8n credentials or environment variables.

#### Node: **Rate Limit Buffer (Alpha Vantage)**
- **Type / role:** `Wait` — throttling buffer.
- **Config choices:** Waits **12** (unit depends on node settings; typically seconds in n8n Wait node when “amount” is set without additional fields).
- **Outputs to:** Normalize Alpha Vantage Data
- **Edge cases / failures:**
  - Too low wait still causes throttling.
  - Too high wait lengthens total runtime; may hit workflow timeout on large portfolios.

#### Node: **Normalize Alpha Vantage Data**
- **Type / role:** `Code` — converts Alpha Vantage response into numeric arrays.
- **Config choices (logic):**
  - Reads `items[0].json`
  - Extracts `series = raw["Time Series (Daily)"] || {}`
  - Sorts dates ascending, maps closing prices from `"4. close"`
  - `current_price` = last close or `0`
  - `returns` = simple differences (`prices[i] - prices[i-1]`) not percent/log returns
  - Outputs:
    - `symbol` from `raw["Meta Data"]["2. Symbol"]` or `"UNKNOWN"`
    - `current_price`, `prices`, `returns`
- **Outputs to:** Batch Portfolio Symbols (loop-back)
- **Edge cases / failures:**
  - If API returns a rate-limit notice, `series` becomes `{}` → `prices=[]`, `returns=[]`, `current_price=0`, symbol may become `"UNKNOWN"`.  
  - This is handled later by filtering invalid entries in the risk engine.

---

### Block 4 — Portfolio Risk Calculation
**Overview:** Aggregates normalized stock items, filters out invalid ones, calculates concentration/volatility/correlation flags, and computes a composite risk score.

**Nodes involved:**
- Portfolio Risk Engine
- Risk Check

#### Node: **Portfolio Risk Engine**
- **Type / role:** `Code` — portfolio-level analytics and scoring.
- **Inputs:** Expects items that include at least:
  - `symbol`, `sector`, `quantity`, `current_price`, `returns`
- **Configuration choices (core logic):**
  1. **Cleaning/filtering:** removes items where:
     - missing/`UNKNOWN` symbol
     - `current_price` not a positive number
     - `returns` missing or too short
  2. **Config read:** `const config = $node["Risk Thresholds and Settings"].json;`
     - uses `sector_concentration_limit` and `correlation_threshold`
  3. **Market value:** `market_value = current_price * (quantity || 1)`
  4. **Sector concentration:** sums market value by sector, computes `weight_pct`, flags `risk` if `weight > sectorLimitPct`
  5. **Volatility:** uses standard deviation of *difference returns* for each stock; portfolio volatility is mean of stock volatilities
  6. **Correlation flags:** pairwise correlation of returns; flags if `abs(corr) >= corrThreshold`
  7. **Risk score:** additive:
     - sector risk: +35
     - correlation risk: +35
     - volatility risk: +30 (this is effectively always true if portfolio_volatility > 0)
  8. Outputs:
     - `portfolio_size`, `portfolio_value`
     - `sector_concentration`, `stock_volatility`, `portfolio_volatility`
     - `correlation_flags`
     - `risk_detected` boolean
     - `risk_score`
- **Outputs to:** Risk Check
- **Edge cases / failures:**
  - If all items are invalid after normalization, it throws: **“No valid portfolio items after normalization”**.
  - `volatilityRisk` is `portfolio_volatility > 0`, which will almost always be true when any returns vary. This biases `risk_detected` and the score. If you intended “volatility above threshold,” you need a configurable threshold.
  - Correlation and volatility are computed on **price differences**, not percent returns; assets with different price scales can distort comparisons.

#### Node: **Risk Check**
- **Type / role:** `IF` — gates downstream AI/alerts/storage.
- **Condition:** `{{$json.risk_detected}}` is **true**
- **True output to:** Portfolio Risk Summary
- **False output:** not connected (workflow ends with no action when no risk)
- **Edge cases / failures:**
  - If you want to also store “no-risk” snapshots, you’d add a false branch to append a record with `risk_detected=false`.

---

### Block 5 — AI Summary, Slack Notification, and Snapshot Storage
**Overview:** When risk is detected, generates an educational summary with GPT, builds a structured alert payload, posts to Slack, and appends a snapshot row into Google Sheets.

**Nodes involved:**
- OpenAI Chat Model
- Portfolio Risk Summary
- Build Alert Payload
- Send Notification
- Store Weekly Risk Snapshot

#### Node: **OpenAI Chat Model**
- **Type / role:** `LM Chat OpenAI` (LangChain) — provides the language model for the agent.
- **Config choices:**
  - Model: **gpt-4.1-mini**
  - No special options/tools enabled
- **Connections:** Provides `ai_languageModel` input to Portfolio Risk Summary
- **Credentials:** OpenAI API credential (“OpenAi account 2”)
- **Edge cases / failures:**
  - Credential/account limits, model access restrictions, or rate limits.
  - If you switch models, token limits may affect output.

#### Node: **Portfolio Risk Summary**
- **Type / role:** `LangChain Agent` — generates the short narrative risk summary.
- **Config choices:**
  - Prompt instructs:
    - Summarize key risks, why they matter, general diversification considerations
    - Must not give financial advice or buy/sell recommendations
    - Under 120 words
  - Injects risk engine output via:
    - `{{ $('Portfolio Risk Engine').item.json.toJsonString() }}`
- **Inputs:** Comes from Risk Check (true branch) and uses OpenAI Chat Model as the language model connection.
- **Outputs to:** Build Alert Payload
- **Edge cases / failures:**
  - If the risk engine output is large (many flags), prompt length can grow; may require truncation or summarizing structured data before sending to LLM.
  - If LangChain agent expects tool usage, but tools are not configured, keep prompt simple (as done here).

#### Node: **Build Alert Payload**
- **Type / role:** `Set` — constructs a single payload consumed by Slack + Sheets.
- **Config choices (fields):**
  - `title`: “Weekly Portfolio Risk Alert”
  - `risk_score`: `{{ $('Risk Check').item.json.risk_score }}`
  - `risk_detected`: `{{ $('Risk Check').item.json.risk_detected }}`
  - `portfolio_value`: `{{ $('Risk Check').item.json.portfolio_value }}`
  - `sector_risks`: `{{ $('Risk Check').item.json.sector_concentration.filter(s => s.risk) }}`
  - `correlation_flags`: `{{ $('Risk Check').item.json.correlation_flags }}`
  - `portfolio_volatility`: `{{ $('Risk Check').item.json.portfolio_volatility }}`
  - `ai_summary`: `{{ $json.output }}` (expects agent output to be in `output`)
  - `generated_at`: `{{ new Date().toISOString() }}`
- **Outputs to:** Send Notification and Store Weekly Risk Snapshot
- **Edge cases / failures:**
  - If the agent node output field differs (sometimes `text` or `response` depending on node/version), `{{$json.output}}` may be undefined. Validate the actual agent output shape in your n8n version.

#### Node: **Send Notification**
- **Type / role:** `Slack` — sends the alert message.
- **Config choices:**
  - Sends a formatted text message to **user** “slackbot” (select=user, value=USLACKBOT)
  - Message includes score, portfolio value, JSON sector risks, correlation flags, volatility, AI summary
- **Credentials:** Slack OAuth2 (“Slack account 5”)
- **Edge cases / failures:**
  - Slack scopes missing (chat:write, im:write, etc.).
  - Posting to slackbot user may behave differently than posting to a channel; for teams, a channel is usually more reliable.
  - Large JSON in message (`sector_risks.toJsonString()`) can exceed Slack message limits.

#### Node: **Store Weekly Risk Snapshot**
- **Type / role:** `Google Sheets` — appends a record for longitudinal tracking.
- **Config choices:**
  - Document: same spreadsheet “NEWS IMPACT TRACKER”
  - Sheet/tab: **“Weekly Risk Snapshots”**
  - Operation: **Append**
  - Columns written:
    - `timestamp`: `generated_at`
    - `ai_summary`
    - `risk_score`: stored as string like `"85/100"`
    - `sector_risks`: array/object written directly (often becomes JSON-like string)
    - `portfolio_value`, `correlation_flags`, `portfolio_volatility`
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai`)
- **Edge cases / failures:**
  - If the sheet schema changes, append may fail or misalign columns.
  - Writing arrays/objects may produce `[object Object]` depending on n8n conversion settings; consider `toJsonString()` for consistent storage.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Schedule Trigger | scheduleTrigger | Weekly entry point | — | Risk Thresholds and Settings | ## Workflow Overview … Adjust thresholds in the Risk Thresholds and Settings node. |
| Risk Thresholds and Settings | set | Define thresholds/toggles | Weekly Schedule Trigger | Read Portfolio Sheet | ### Config and portfolio output … thresholds and feature toggles … |
| Read Portfolio Sheet | googleSheets | Load portfolio rows | Risk Thresholds and Settings | Validate Portfolio Input | ### Config and portfolio output … read from Google Sheets and validated … |
| Validate Portfolio Input | code | Validate required columns; normalize schema | Read Portfolio Sheet | Batch Portfolio Symbols | ### Config and portfolio output … validated to ensure required columns … |
| Batch Portfolio Symbols | splitInBatches | Batch/loop controller for symbols | Validate Portfolio Input; Normalize Alpha Vantage Data | Portfolio Risk Engine; Fetch Market Data | ### Market data and rate limiting … processed in batches with a wait step … |
| Fetch Market Data | httpRequest | Pull daily time series from Alpha Vantage | Batch Portfolio Symbols | Rate Limit Buffer (Alpha Vantage) | ### Market data and rate limiting … fetch historical price data … |
| Rate Limit Buffer (Alpha Vantage) | wait | Reduce API throttling risk | Fetch Market Data | Normalize Alpha Vantage Data | ### Market data and rate limiting … wait step to respect API limits … |
| Normalize Alpha Vantage Data | code | Convert raw API payload to prices/returns | Rate Limit Buffer (Alpha Vantage) | Batch Portfolio Symbols | ### Market data and rate limiting … normalized into a consistent structure … |
| Portfolio Risk Engine | code | Compute concentration/volatility/correlation + score | Batch Portfolio Symbols | Risk Check | ### Risk calculation engine … produces risk score and risk detected flag. |
| Risk Check | if | Conditional: proceed only if risk detected | Portfolio Risk Engine | Portfolio Risk Summary (true) | ### Ai summary and storage … run only when risk is detected … |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for summary | — | Portfolio Risk Summary (ai_languageModel) | ### Ai summary and storage … AI summary explains the risks … |
| Portfolio Risk Summary | langchain.agent | Generate short educational risk summary | Risk Check (true) | Build Alert Payload | ### Ai summary and storage … analytical language only … |
| Build Alert Payload | set | Assemble payload for Slack + Sheets | Portfolio Risk Summary; Risk Check (via expressions) | Send Notification; Store Weekly Risk Snapshot | ### Ai summary and storage … sends Slack alert and stores in Sheets … |
| Send Notification | slack | Slack alert delivery | Build Alert Payload | — | ### Ai summary and storage … weekly risk alert to Slack … |
| Store Weekly Risk Snapshot | googleSheets | Append weekly snapshot row | Build Alert Payload | — | ### Ai summary and storage … stored in Google Sheets … |
| Sticky Note | stickyNote | Comment block | — | — | ## Workflow Overview (content shown in node) |
| Sticky Note1 | stickyNote | Comment block | — | — | ### Config and portfolio output (content shown in node) |
| Sticky Note2 | stickyNote | Comment block | — | — | ### Market data and rate limiting (content shown in node) |
| Sticky Note3 | stickyNote | Comment block | — | — | ### Risk calculation engine (content shown in node) |
| Sticky Note4 | stickyNote | Comment block | — | — | ### Ai summary and storage (content shown in node) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create node: “Weekly Schedule Trigger” (Schedule Trigger)**
   - Set interval to **Weeks**
   - Set **Trigger at hour = 9** (confirm timezone in n8n settings)

2. **Create node: “Risk Thresholds and Settings” (Set)**
   - Add fields:
     - `sector_concentration_limit` (Number) = 35
     - `correlation_threshold` (Number) = 0.75
     - `volatility_lookback_days` (Number) = 30
     - `volatility_alert_change_pct` (Number) = 10
     - `enable_alerts` (Boolean) = true
     - `enable_ai_summary` (Boolean) = true
     - `base_currency` (String) = INR
   - Connect: **Weekly Schedule Trigger → Risk Thresholds and Settings**

3. **Create node: “Read Portfolio Sheet” (Google Sheets)**
   - Credential: **Google Sheets OAuth2**
   - Select spreadsheet (document) and tab:
     - Document: your spreadsheet (same as “NEWS IMPACT TRACKER” in the JSON)
     - Sheet: **Portfolio**
   - Configure to **read rows** (use default “Read” operation in your n8n version)
   - Connect: **Risk Thresholds and Settings → Read Portfolio Sheet**

4. **Create node: “Validate Portfolio Input” (Code)**
   - Paste logic equivalent to:
     - Ensure sheet not empty
     - Require `Symbol`, `Sector`, `Quantity`
     - Quantity > 0
     - Output normalized `symbol/sector/quantity/avg_price`
   - Connect: **Read Portfolio Sheet → Validate Portfolio Input**

5. **Create node: “Batch Portfolio Symbols” (Split In Batches)**
   - Set batch size (recommended **1–5** depending on API limits; Alpha Vantage is often safest at **1**)
   - Connect: **Validate Portfolio Input → Batch Portfolio Symbols**

6. **Create node: “Fetch Market Data” (HTTP Request)**
   - Method: GET
   - URL: `https://www.alphavantage.co/query`
   - Enable “Send Query Parameters”
   - Query parameters:
     - `function` = `TIME_SERIES_DAILY`
     - `symbol` = `{{$json.symbol}}`
     - `apikey` = your Alpha Vantage key (store in environment variable or credential if possible)
     - `outputsize` = `compact`
   - Connect: **Batch Portfolio Symbols → Fetch Market Data** (use the loop/batch output)

7. **Create node: “Rate Limit Buffer (Alpha Vantage)” (Wait)**
   - Wait amount: **12 seconds** (adjust based on your Alpha Vantage tier)
   - Connect: **Fetch Market Data → Rate Limit Buffer (Alpha Vantage)**

8. **Create node: “Normalize Alpha Vantage Data” (Code)**
   - Implement:
     - Extract `"Time Series (Daily)"`
     - Sort dates
     - Create `prices[]`, `returns[]`, `current_price`
     - Output `{symbol, current_price, prices, returns}`
   - Connect: **Rate Limit Buffer → Normalize Alpha Vantage Data**
   - Loop back: **Normalize Alpha Vantage Data → Batch Portfolio Symbols** (to process next batch)

9. **Create node: “Portfolio Risk Engine” (Code)**
   - Implement the sector concentration + volatility + correlation logic.
   - Read thresholds from: `$node["Risk Thresholds and Settings"].json`
   - Connect: **Batch Portfolio Symbols → Portfolio Risk Engine**
   - Note: Ensure this runs only after batching completes. If needed, restructure to collect all normalized items (e.g., with Merge/NoOp patterns) before the risk engine.

10. **Create node: “Risk Check” (IF)**
    - Condition: `{{$json.risk_detected}}` is true
    - Connect: **Portfolio Risk Engine → Risk Check**

11. **Create node: “OpenAI Chat Model” (LM Chat OpenAI / LangChain)**
    - Credential: OpenAI API key
    - Model: **gpt-4.1-mini**
    - Connect its **AI language model** output to the agent node (next step)

12. **Create node: “Portfolio Risk Summary” (LangChain Agent)**
    - Prompt: include the risk engine JSON and constraints (no financial advice; <120 words)
    - Reference the risk output with an expression like:
      - `{{ $('Portfolio Risk Engine').item.json.toJsonString() }}`
    - Connect: **Risk Check (true) → Portfolio Risk Summary**
    - Connect: **OpenAI Chat Model → Portfolio Risk Summary** (ai_languageModel connection)

13. **Create node: “Build Alert Payload” (Set)**
    - Map fields from Risk Check + agent output:
      - `risk_score`, `risk_detected`, `portfolio_value`, `sector_risks`, `correlation_flags`, `portfolio_volatility`
      - `ai_summary` from agent output field (verify in your n8n version)
      - `generated_at` = `{{ new Date().toISOString() }}`
    - Connect: **Portfolio Risk Summary → Build Alert Payload**

14. **Create node: “Send Notification” (Slack)**
    - Credential: Slack OAuth2
    - Destination: user “slackbot” or choose a channel
    - Message: interpolate fields from Build Alert Payload
    - Connect: **Build Alert Payload → Send Notification**

15. **Create node: “Store Weekly Risk Snapshot” (Google Sheets)**
    - Credential: Google Sheets OAuth2
    - Operation: **Append**
    - Sheet: **Weekly Risk Snapshots**
    - Map columns:
      - `timestamp` = `{{$json.generated_at}}`
      - `ai_summary` = `{{$json.ai_summary}}`
      - `risk_score` = `{{$json.risk_score}}/100`
      - `sector_risks`, `portfolio_value`, `correlation_flags`, `portfolio_volatility`
    - Connect: **Build Alert Payload → Store Weekly Risk Snapshot**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Provided compliance disclaimer (French) |
| Alpha Vantage API key is hard-coded in the HTTP Request node | Consider moving to n8n credentials, environment variables, or encrypted variables |
| Threshold/toggle fields exist but some are not enforced in flow | `enable_alerts`, `enable_ai_summary`, `volatility_lookback_days`, `volatility_alert_change_pct`, `base_currency` are currently not used to change behavior |
| Risk engine uses price-difference returns (not percent/log returns) | Can distort volatility/correlation across differently priced assets; consider switching to percent returns |

