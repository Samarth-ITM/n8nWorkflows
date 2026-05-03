Analyze US equity futures and options with MarketData, NewsAPI, and Gemini

https://n8nworkflows.xyz/workflows/analyze-us-equity-futures-and-options-with-marketdata--newsapi--and-gemini-12864


# Analyze US equity futures and options with MarketData, NewsAPI, and Gemini

## 1. Workflow Overview

**Title:** Analyze US equity futures and options with MarketData, NewsAPI, and Gemini

**Purpose / use case:**  
This workflow listens for a Telegram message containing an *underlying* (e.g., an equity index future or equity symbol), fetches multi-timeframe price candles + current quote, pulls the option chain and derives options signals, retrieves related news and runs sentiment analysis, then asks a Gemini-powered agent to synthesize everything into a trading-style analysis and sends it back to Telegram.

### Logical blocks
**1.1 Telegram intake & symbol parsing**
- Receives a Telegram message and extracts/normalizes the underlying identifier used downstream.

**1.2 Market data collection (candles + quote)**
- Pulls 15m, 1h, and 1d candles; normalizes them into a consistent schema; merges into one combined candle payload.
- Pulls the latest quote to determine spot and select an options expiry.

**1.3 Options chain + signal derivation**
- Fetches the option chain from MarketData.app using the chosen expiry and computes summary signals (e.g., IV/Skew/flow-type aggregates—exact metrics depend on the Code node logic).

**1.4 News retrieval + sentiment**
- Fetches news, filters/cleans it, then uses Gemini to generate a sentiment/impact assessment.

**1.5 Synthesis (agent) + delivery**
- Merges candles, options signals, and news sentiment into one payload.
- Sends that context to a Gemini Agent (using a Gemini Chat Model node as the LLM).
- Sanitizes HTML and sends the final formatted text back via Telegram.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Telegram intake & symbol parsing
**Overview:** Triggers on Telegram messages and extracts the “underlying” symbol/ticker to drive downstream API calls.

**Nodes involved:**
- **Telegram Trigger**
- **Parse Underlying**

#### Node: Telegram Trigger
- **Type / role:** `n8n-nodes-base.telegramTrigger` — entry point; receives incoming Telegram updates.
- **Configuration (interpreted):** Uses Telegram Trigger defaults (the JSON shows empty parameters). It will fire on incoming messages based on how the Telegram Trigger is configured in the UI (typically “On message”).
- **Inputs / outputs:** No input; outputs Telegram update payload to **Parse Underlying**.
- **Version-specific notes:** v1.2.
- **Failure modes / edge cases:**
  - Telegram credential/token misconfigured → trigger won’t start.
  - Bot privacy settings may prevent receiving messages in groups.
  - Non-text messages (stickers/files) may lack the fields your Code node expects.

#### Node: Parse Underlying
- **Type / role:** `n8n-nodes-base.code` — parses the Telegram payload and produces a normalized symbol plus any parameters used by HTTP requests.
- **Configuration (interpreted):** Parameters not shown; expect custom JS that reads something like `{{$json.message.text}}` and outputs fields such as:
  - `underlying` / `symbol`
  - potentially `exchange`, `assetType`, or a canonical format for futures vs equities
- **Connections:**
  - **Outputs to (in parallel):**
    - Underlying Candles 15m
    - Underlying Candles 1h
    - Underlying Candles 1d
    - Underlying Quote
    - News
- **Failure modes / edge cases:**
  - Empty message text / unexpected Telegram payload shape → JS exceptions.
  - User sends an unsupported symbol format → downstream API 4xx.
  - If parsing attempts to map aliases (e.g., “ES” → “ES1!”) and mapping is incomplete.

---

### Block 1.2 — Market data collection (candles + quote)
**Overview:** Retrieves multi-timeframe candles and the current quote for the underlying; normalizes candle shapes; merges all candle timeframes into one structured object for the agent.

**Nodes involved:**
- Underlying Candles 15m
- Underlying Candles 1h
- Underlying Candles 1d
- Normalize 15m
- Normalize 1h
- Normalize 1d
- Merge Candles
- Combine Candles
- Underlying Quote
- Pick Spot + Expiry

#### Node: Underlying Candles 15m / 1h / 1d
- **Type / role:** `n8n-nodes-base.httpRequest` — calls a market data API endpoint (likely MarketData.app based on naming elsewhere) to fetch candles for different resolutions.
- **Configuration (interpreted):**
  - Method likely **GET**
  - URL likely parameterized by the parsed symbol and timeframe.
  - Response expected: array of OHLCV candles + timestamps.
- **Connections:**
  - Each feeds its corresponding Normalize node.
- **Version-specific notes:** HTTP Request v4.3.
- **Failure modes / edge cases:**
  - 401/403 if API key missing/invalid.
  - Rate limits (429) when triggered frequently.
  - Timezone/session issues for futures (overnight sessions).
  - Empty candle sets on holidays or invalid symbol.

#### Node: Normalize 15m / Normalize 1h / Normalize 1d
- **Type / role:** `n8n-nodes-base.set` — reshapes candle payload into a consistent schema for merging.
- **Configuration (interpreted):**
  - Typically sets fields like `timeframe`, `candles`, and standardizes property names (open/high/low/close/volume/time).
  - JSON shows empty parameters, but in practice this node must be configured in UI; if truly empty, it passes through unchanged (which may break later assumptions).
- **Connections:**
  - All three go into **Merge Candles** on input indexes 0/1/2.
- **Version-specific notes:** Set v3.4.
- **Failure modes / edge cases:**
  - Misaligned field names across timeframes leading to merge/combination issues.
  - Large candle arrays → memory / execution time overhead.

#### Node: Merge Candles
- **Type / role:** `n8n-nodes-base.merge` — combines the three normalized candle streams.
- **Configuration (interpreted):**
  - Likely configured as “Combine” or “Merge by position” to unify 15m/1h/1d outputs into a single item.
  - Inputs: index 0 = 15m, index 1 = 1h, index 2 = 1d (per connections).
- **Output:** To **Combine Candles**.
- **Version-specific notes:** Merge v3.2.
- **Failure modes / edge cases:**
  - If any candle request returns zero items while others return one item, merge mode choice can drop data.
  - If one branch errors, the workflow may stop unless “Continue On Fail” is enabled (not shown).

#### Node: Combine Candles
- **Type / role:** `n8n-nodes-base.code` — consolidates merged candle inputs into a single structured object (e.g., `{ candles15m: [...], candles1h: [...], candles1d: [...] }`).
- **Connections:** Outputs to **Merge (Candles + Options + News)** (input 0).
- **Failure modes / edge cases:**
  - Unexpected merge shape (arrays of items vs single item) causing indexing bugs.
  - Candles too large; consider truncation (last N candles) for LLM context.

#### Node: Underlying Quote
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches latest quote/price for the underlying.
- **Connections:** Outputs to **Pick Spot + Expiry**.
- **Failure modes / edge cases:**
  - Quote endpoint may differ for futures vs equities; parsing must handle both.
  - Market closed → stale price; consider using last close.

#### Node: Pick Spot + Expiry
- **Type / role:** `n8n-nodes-base.code` — determines spot (current price) and selects an options expiry date to query.
- **Configuration (interpreted):**
  - Reads quote response (spot).
  - Chooses an expiry (e.g., nearest weekly/monthly, next trading day, etc.).
  - Produces fields consumed by the option-chain request (symbol + expiry).
- **Connections:** Outputs to **Option Chain (MarketData.app)**.
- **Failure modes / edge cases:**
  - Weekend/holiday logic: “nearest expiry” might be non-trading day.
  - Underlyings without listed options → downstream empty chain.

---

### Block 1.3 — Options chain + signal derivation
**Overview:** Fetches the option chain for the selected expiry and derives a compact set of signals suitable for LLM reasoning.

**Nodes involved:**
- Option Chain (MarketData.app)
- Derive Options Signals

#### Node: Option Chain (MarketData.app)
- **Type / role:** `n8n-nodes-base.httpRequest` — calls MarketData.app option chain endpoint.
- **Configuration (interpreted):**
  - Likely GET with query params: underlying symbol + expiry date.
  - Returns calls/puts with strikes, IV, greeks, OI/volume, etc. (depends on provider).
- **Connections:** Outputs to **Derive Options Signals**.
- **Failure modes / edge cases:**
  - Large chain payload (many strikes) → heavy processing + LLM context issues.
  - 429 rate limiting.
  - Missing fields (e.g., greeks not available) breaks derivation code.

#### Node: Derive Options Signals
- **Type / role:** `n8n-nodes-base.code` — computes summary metrics/signals from the option chain.
- **Configuration (interpreted):**
  - Typical outputs could include: put/call OI ratio, ATM IV, skew, IV rank (if history available), most active strikes, gamma exposure approximations, etc.
  - Produces one compact JSON object intended for merge into the agent context.
- **Connections:** Outputs to **Merge (Candles + Options + News)** (input 1).
- **Failure modes / edge cases:**
  - Assumes numeric fields exist; nulls/strings cause NaN issues.
  - Needs a clear rule to select “ATM” strike based on spot; if spot missing, signals degrade.

---

### Block 1.4 — News retrieval + sentiment
**Overview:** Retrieves recent news, filters it down to the most relevant items, then uses Gemini to score/describe sentiment and potential market impact.

**Nodes involved:**
- News
- Filter News
- News Sentiment Analyzer

#### Node: News
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches news from NewsAPI (inferred from workflow title).
- **Configuration (interpreted):**
  - Likely GET `https://newsapi.org/...` with query based on underlying name/symbol.
  - Uses an API key credential/header.
- **Connections:** Outputs to **Filter News**.
- **Failure modes / edge cases:**
  - NewsAPI free tier limitations (rate limits, restricted sources, delayed articles).
  - Ambiguous tickers (e.g., “CAT”) returning irrelevant stories unless query is refined.

#### Node: Filter News
- **Type / role:** `n8n-nodes-base.code` — removes irrelevant/duplicate articles and formats text for LLM.
- **Configuration (interpreted):**
  - Might keep top N articles, strip HTML, keep title+source+publishedAt+description/url.
- **Connections:** Outputs to **News Sentiment Analyzer**.
- **Failure modes / edge cases:**
  - If NewsAPI returns `status:error` payload, code must handle it.
  - Very long descriptions can bloat token usage.

#### Node: News Sentiment Analyzer
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini-based analysis step (likely prompts Gemini to return sentiment and a short summary).
- **Configuration (interpreted):**
  - Uses Gemini credentials (Google AI Studio / Gemini API key depending on node setup).
  - Produces structured or semi-structured sentiment output to be merged later.
- **Connections:** Outputs to **Merge (Candles + Options + News)** (input 2).
- **Version-specific notes:** v1.
- **Failure modes / edge cases:**
  - LLM credential/auth errors.
  - Safety filters or refusal depending on prompt content (rare here).
  - Non-deterministic output; downstream expects consistent fields (mitigate by enforcing JSON schema in prompt).

---

### Block 1.5 — Synthesis (agent) & Telegram delivery
**Overview:** Consolidates all analytics into one payload, runs a Gemini Agent to generate the final narrative, sanitizes formatting, and posts to Telegram.

**Nodes involved:**
- Merge (Candles + Options + News)
- Google Gemini Chat Model
- F&O Agent
- HTML Sanitation
- Send Telegram Message

#### Node: Merge (Candles + Options + News)
- **Type / role:** `n8n-nodes-base.merge` — merges three streams into one context item for the agent.
- **Configuration (interpreted):**
  - Likely “Merge by position” / “Combine” such that one output item contains candles + options signals + news sentiment.
  - Inputs: 0 = candles, 1 = options signals, 2 = news sentiment.
- **Connections:** Outputs to **F&O Agent**.
- **Failure modes / edge cases:**
  - If one branch produces multiple items, merge behavior can create unexpected cartesian merges.
  - If news sentiment returns 0 items, merge may output nothing (depending on merge mode).

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM to the agent node.
- **Configuration (interpreted):**
  - Model selection, temperature, max tokens are configured in UI (not visible here).
- **Connections:** Connected to **F&O Agent** via the special `ai_languageModel` connection.
- **Failure modes / edge cases:**
  - Model not available in region/project.
  - Token limits exceeded if candles/options/news are too large.

#### Node: F&O Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates reasoning with Gemini to produce final analysis text.
- **Configuration (interpreted):**
  - Takes merged JSON context as input.
  - Uses **Google Gemini Chat Model** as its language model.
  - May contain a system prompt / instructions and expected output formatting (not visible).
- **Connections:** Outputs to **HTML Sanitation**.
- **Version-specific notes:** Agent v3.
- **Failure modes / edge cases:**
  - If prompt expects fields that aren’t present (e.g., `optionsSignals.atmIV`) → weak/incorrect output.
  - Long runtime / timeouts if context is huge.

#### Node: HTML Sanitation
- **Type / role:** `n8n-nodes-base.code` — cleans/sanitizes model output for Telegram (e.g., stripping unsupported tags, escaping characters).
- **Connections:** Outputs to **Send Telegram Message**.
- **Failure modes / edge cases:**
  - Telegram parse mode issues (HTML vs MarkdownV2). If sanitation mismatched to parse mode, messages can fail.

#### Node: Send Telegram Message
- **Type / role:** `n8n-nodes-base.telegram` — sends the final analysis back to Telegram chat.
- **Configuration (interpreted):**
  - Chat ID likely taken from the original trigger payload.
  - Message text likely taken from the agent output after sanitation.
- **Version-specific notes:** v1.2.
- **Failure modes / edge cases:**
  - Missing/incorrect chat_id.
  - Telegram message length limit (approx. 4096 chars) → truncate or split.
  - Parse mode errors if markup is invalid.

---

### Sticky Notes (comments)
All sticky notes in the provided JSON have **empty content**, so they do not add documentation context. They are still listed in the Summary Table.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Entry point (Telegram message trigger) | — | Parse Underlying |  |
| Parse Underlying | n8n-nodes-base.code | Extract/normalize underlying symbol | Telegram Trigger | Underlying Candles 15m; Underlying Candles 1h; Underlying Candles 1d; Underlying Quote; News |  |
| Underlying Candles 15m | n8n-nodes-base.httpRequest | Fetch 15-minute candles | Parse Underlying | Normalize 15m |  |
| Underlying Candles 1h | n8n-nodes-base.httpRequest | Fetch 1-hour candles | Parse Underlying | Normalize 1h |  |
| Underlying Candles 1d | n8n-nodes-base.httpRequest | Fetch daily candles | Parse Underlying | Normalize 1d |  |
| Normalize 15m | n8n-nodes-base.set | Standardize 15m candle schema | Underlying Candles 15m | Merge Candles |  |
| Normalize 1h | n8n-nodes-base.set | Standardize 1h candle schema | Underlying Candles 1h | Merge Candles |  |
| Normalize 1d | n8n-nodes-base.set | Standardize 1d candle schema | Underlying Candles 1d | Merge Candles |  |
| Merge Candles | n8n-nodes-base.merge | Merge normalized candle streams | Normalize 15m; Normalize 1h; Normalize 1d | Combine Candles |  |
| Combine Candles | n8n-nodes-base.code | Build a single multi-timeframe candles object | Merge Candles | Merge (Candles + Options + News) |  |
| Underlying Quote | n8n-nodes-base.httpRequest | Fetch latest quote/spot | Parse Underlying | Pick Spot + Expiry |  |
| Pick Spot + Expiry | n8n-nodes-base.code | Determine spot and choose expiry for options | Underlying Quote | Option Chain (MarketData.app) |  |
| Option Chain (MarketData.app) | n8n-nodes-base.httpRequest | Fetch option chain from MarketData.app | Pick Spot + Expiry | Derive Options Signals |  |
| Derive Options Signals | n8n-nodes-base.code | Compute summary options signals | Option Chain (MarketData.app) | Merge (Candles + Options + News) |  |
| News | n8n-nodes-base.httpRequest | Fetch relevant news (NewsAPI inferred) | Parse Underlying | Filter News |  |
| Filter News | n8n-nodes-base.code | Filter/format news items | News | News Sentiment Analyzer |  |
| News Sentiment Analyzer | @n8n/n8n-nodes-langchain.googleGemini | LLM sentiment/impact analysis | Filter News | Merge (Candles + Options + News) |  |
| Merge (Candles + Options + News) | n8n-nodes-base.merge | Combine candles, options signals, and news sentiment | Combine Candles; Derive Options Signals; News Sentiment Analyzer | F&O Agent |  |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for the agent | — | F&O Agent (ai_languageModel) |  |
| F&O Agent | @n8n/n8n-nodes-langchain.agent | Generate final analysis using Gemini | Merge (Candles + Options + News); Google Gemini Chat Model | HTML Sanitation |  |
| HTML Sanitation | n8n-nodes-base.code | Clean/escape output for Telegram | F&O Agent | Send Telegram Message |  |
| Send Telegram Message | n8n-nodes-base.telegram | Deliver final message to Telegram | HTML Sanitation | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas annotation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set the name to:  
   `Analyze US equity futures and options with MarketData, NewsAPI, and Gemini`

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Configure Telegram credentials (bot token).
   - Choose update type that captures messages (commonly “On message”).
   - This becomes the workflow entry.

3. **Add Code node: “Parse Underlying”**
   - Node: **Code**
   - Implement logic to read the incoming message text (from Telegram Trigger) and output a clean symbol.
   - Output fields you will reuse in all HTTP requests, e.g.:
     - `symbol`
     - optionally `queryText` for news (company name vs ticker)
   - Connect: **Telegram Trigger → Parse Underlying**

4. **Add 3 HTTP Request nodes for candles**
   - Nodes: **HTTP Request** named:
     - `Underlying Candles 15m`
     - `Underlying Candles 1h`
     - `Underlying Candles 1d`
   - Configure each to call your market data candle endpoint with:
     - symbol from `Parse Underlying`
     - timeframe parameter (15m/1h/1d)
   - Connect: **Parse Underlying → each candle HTTP node** (parallel)

5. **Add 3 Set nodes to normalize candle outputs**
   - Nodes: **Set** named:
     - `Normalize 15m`, `Normalize 1h`, `Normalize 1d`
   - In each Set node, map the candle API response into a standard structure, e.g.:
     - `timeframe`: `"15m"` / `"1h"` / `"1d"`
     - `candles`: normalized array
   - Connect each candle HTTP node to its Normalize node.

6. **Add Merge node: “Merge Candles”**
   - Node: **Merge**
   - Configure to combine the three inputs into a single merged output (choose a merge mode that preserves all three streams predictably; “Merge by position” is commonly used when each branch outputs exactly one item).
   - Connect:
     - Normalize 15m → Merge Candles (Input 1 / index 0)
     - Normalize 1h → Merge Candles (Input 2 / index 1)
     - Normalize 1d → Merge Candles (Input 3 / index 2)

7. **Add Code node: “Combine Candles”**
   - Node: **Code**
   - Combine the merged result into one object like:
     - `candles: { m15: [...], h1: [...], d1: [...] }`
     - Optionally truncate to last N candles per timeframe.
   - Connect: **Merge Candles → Combine Candles**

8. **Add HTTP Request node: “Underlying Quote”**
   - Node: **HTTP Request**
   - Configure to call your quote endpoint for the same symbol.
   - Connect: **Parse Underlying → Underlying Quote**

9. **Add Code node: “Pick Spot + Expiry”**
   - Node: **Code**
   - Read quote response to get `spot`.
   - Decide an `expiry` (nearest valid options expiry).
   - Output at minimum: `symbol`, `spot`, `expiry`.
   - Connect: **Underlying Quote → Pick Spot + Expiry**

10. **Add HTTP Request node: “Option Chain (MarketData.app)”**
    - Node: **HTTP Request**
    - Configure MarketData.app option chain call using `symbol` and `expiry`.
    - Add credentials/headers for MarketData.app (API token).
    - Connect: **Pick Spot + Expiry → Option Chain (MarketData.app)**

11. **Add Code node: “Derive Options Signals”**
    - Node: **Code**
    - Compute compact signals from the chain (PCR, skew, key strikes, etc.).
    - Output a single item with `optionsSignals`.
    - Connect: **Option Chain (MarketData.app) → Derive Options Signals**

12. **Add HTTP Request node: “News”**
    - Node: **HTTP Request**
    - Configure NewsAPI request using either the ticker or a broader query string.
    - Add NewsAPI credential (API key header/query param).
    - Connect: **Parse Underlying → News**

13. **Add Code node: “Filter News”**
    - Node: **Code**
    - Keep top N relevant articles; output a compact list.
    - Connect: **News → Filter News**

14. **Add Gemini node: “News Sentiment Analyzer”**
    - Node: **Google Gemini** (LangChain integration node)
    - Configure credentials for Gemini (API key / Google project depending on node).
    - Prompt it to produce consistent structured sentiment output (ideally JSON).
    - Connect: **Filter News → News Sentiment Analyzer**

15. **Add Merge node: “Merge (Candles + Options + News)”**
    - Node: **Merge**
    - Configure to merge the three branches into one final context object.
    - Connect:
      - Combine Candles → Merge (input 0)
      - Derive Options Signals → Merge (input 1)
      - News Sentiment Analyzer → Merge (input 2)

16. **Add Gemini Chat Model node: “Google Gemini Chat Model”**
    - Node: **Google Gemini Chat Model**
    - Select model and generation settings (temperature, max tokens).
    - Configure Gemini credentials.

17. **Add Agent node: “F&O Agent”**
    - Node: **Agent**
    - Connect the **Google Gemini Chat Model** to the Agent via the **Language Model** connection.
    - Provide agent instructions to generate the final analysis (structure, risk notes, key levels, etc.).
    - Connect main input: **Merge (Candles + Options + News) → F&O Agent**

18. **Add Code node: “HTML Sanitation”**
    - Node: **Code**
    - Clean the agent output for Telegram:
      - remove/escape unsupported HTML
      - enforce Telegram length limits (truncate or split)
    - Connect: **F&O Agent → HTML Sanitation**

19. **Add Telegram node: “Send Telegram Message”**
    - Node: **Telegram**
    - Operation: send message.
    - Map:
      - `chat_id` from the original trigger payload (pass it through your nodes or store it in `Parse Underlying` output).
      - `text` from the sanitized agent output.
    - Connect: **HTML Sanitation → Send Telegram Message**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Disclaimer (provided by user): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Applies to the full workflow context |

If you want, paste the **node parameter details** (especially the Code nodes + HTTP Request URLs/headers and the Agent prompts). With those, I can document the exact expressions/fields and the precise API contract end-to-end.