Automated Forex Portfolio Manager with AI, MT5 & News Sentiment Analysis

https://n8nworkflows.xyz/workflows/automated-forex-portfolio-manager-with-ai--mt5---news-sentiment-analysis-14862


# Automated Forex Portfolio Manager with AI, MT5 & News Sentiment Analysis

Now I have all the information I need to create a comprehensive, structured reference document. Let me write it out.### 1. Workflow Overview

This workflow implements a fully automated Forex portfolio management system that runs every hour. It aggregates portfolio state and multi-source market intelligence (news headlines, macro sentiment, fear & greed index, economic calendar), feeds everything into an Anthropic-powered AI analyst for decision-making, and then routes any recommended trade actions (buy market/limit, sell market/limit, close position) through an execution layer that submits orders to a MetaTrader 5 (MT5) signal handler via HTTP. Both portfolio status updates and trade execution confirmations are pushed to Discord in real time.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Trigger & Configuration | Hourly scheduling and runtime parameter setup |
| 2 | Data Aggregation | Parallel fetching of portfolio state and five external market data sources, then merging |
| 3 | Context Building & AI Analysis | Assembling a unified context string and invoking an Anthropic LLM chain for portfolio analysis |
| 4 | Response Processing & Notification | Parsing the AI response, formatting and sending a Discord portfolio update |
| 5 | Action Extraction & Routing | Extracting trade actions, checking for their presence, and routing by action type (buy / sell / close) |
| 6 | Order Execution | Sub-routing buy/sell into market vs. limit orders and submitting to the MT5 handler; submitting close-position signals |
| 7 | Confirmation | Building and sending a Discord trade-execution confirmation |

---

### 2. Block-by-Block Analysis

#### Block 1 — Trigger & Configuration

**Overview:** A cron-based schedule trigger fires once per hour, then a Set node establishes runtime configuration values (API keys, endpoint URLs, portfolio identifiers, etc.) that downstream nodes reference via expressions.

**Nodes Involved:** Every Hour Trigger, Config, Sticky Note

---

**Every Hour Trigger**
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.3)
- **Technical Role:** Workflow entry point; emits an empty item on a fixed hourly schedule.
- **Configuration:** Default hourly recurrence (no custom cron expression specified).
- **Key Expressions/Variables:** None — outputs a single empty item to kick off execution.
- **Input Connections:** None (entry node).
- **Output Connections:** → Config
- **Edge Cases / Failures:** If the n8n instance is paused or the worker is unavailable at the scheduled time, that hour's execution is skipped entirely; no retry mechanism exists at the trigger level.
- **Version Requirements:** v1.3 of the schedule trigger.

---

**Config**
- **Type:** `n8n-nodes-base.set` (v3.4)
- **Technical Role:** Centralized runtime configuration holder. Defines static and computed values (API keys, base URLs, portfolio ID, Discord webhook references, MT5 handler endpoint) consumed by all downstream nodes via `$('Config').item.json.*` expressions.
- **Configuration:** Values are set via the Set node's field list (not visible in JSON but architecturally required). Expected fields include at minimum: `NEWSAPI_KEY`, `FINNHUB_KEY`, `ALPHAVANTAGE_KEY`, `MT5_HANDLER_URL`, `PORTFOLIO_ID`, `DISCORD_WEBHOOK_UPDATE`, `DISCORD_WEBHOOK_TRADE`, `ANTHROPIC_API_KEY` reference.
- **Key Expressions/Variables:** All downstream HTTP and Discord nodes reference `$('Config').item.json.*` for dynamic parameterization.
- **Input Connections:** ← Every Hour Trigger
- **Output Connections:** → Get Portfolio from Signal Handler, NewsAPI Headlines, FinnHub Market News, Alpha Vantage Macro Sentiment, Fear & Greed Index, Forex Factory Calendar (all six data-source nodes fan out from this single output)
- **Edge Cases / Failures:** If any required field is missing or empty, downstream HTTP requests may fail with 401/403 or malformed URLs. No validation is performed inside this node.
- **Version Requirements:** v3.4 of the Set node.

---

**Sticky Note** (positioned near the trigger area)
- **Type:** `n8n-nodes-base.stickyNote` (v1)
- **Content:** *(empty in JSON — likely a placeholder for a future note or an authoring artifact)*
- **Context:** Visually groups the trigger and configuration area.

---

#### Block 2 — Data Aggregation

**Overview:** Six HTTP Request nodes execute in parallel (all receiving input from Config). Each fetches a different data source: the current portfolio from the MT5 signal handler, news headlines from NewsAPI, market news from FinnHub, macro sentiment from Alpha Vantage, the Fear & Greed Index, and the Forex Factory economic calendar. All six outputs converge into a Merge node that produces a single combined item containing every source's payload.

**Nodes Involved:** Get Portfolio from Signal Handler, NewsAPI Headlines, FinnHub Market News, Alpha Vantage Macro Sentiment, Fear & Greed Index, Forex Factory Calendar, Merge All Data, Data Sources (sticky note)

---

**Get Portfolio from Signal Handler**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Retrieves the current Forex portfolio (open positions, balances, P&L) from the MT5 signal handler service.
- **Configuration:** HTTP GET to the MT5 handler endpoint; URL and authentication likely derived from Config fields. Retry on fail enabled with 2 max attempts. On error, continues to regular output (allowing partial data to flow downstream).
- **Key Expressions/Variables:** URL referencing `$('Config').item.json.MT5_HANDLER_URL`; authentication headers from Config.
- **Input Connections:** ← Config
- **Output Connections:** → Merge All Data (input index 0)
- **Edge Cases / Failures:** MT5 service downtime → empty/partial portfolio; authentication failure → 401; timeout after retries → continues with error output, downstream nodes must handle missing portfolio data.
- **Version Requirements:** v4.2 HTTP Request with retry support.

---

**NewsAPI Headlines**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Fetches recent Forex/market-related news headlines from the NewsAPI service.
- **Configuration:** HTTP GET to `https://newsapi.org/v2/everything` (or `/top-headlines`) with query parameters for Forex keywords, API key from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** `$('Config').item.json.NEWSAPI_KEY` injected as query parameter or header.
- **Input Connections:** ← Config
- **Output Connections:** → Merge All Data (input index 1)
- **Edge Cases / Failures:** API rate limit (429) → continues with error output; invalid/expired key → 401; no articles found → empty `articles` array.
- **Version Requirements:** v4.2 HTTP Request.

---

**FinnHub Market News**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Fetches market news specifically from FinnHub's market news endpoint (often Forex/commodities category).
- **Configuration:** HTTP GET to `https://finnhub.io/api/v1/news` with category parameter and API token from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** `$('Config').item.json.FINNHUB_KEY` as query parameter `token`.
- **Input Connections:** ← Config
- **Output Connections:** → Merge All Data (input index 2)
- **Edge Cases / Failures:** Free-tier rate limits; FinnHub category parameter must be valid; returns an array of news objects — empty if no relevant news.
- **Version Requirements:** v4.2 HTTP Request.

---

**Alpha Vantage Macro Sentiment**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Retrieves macro-economic sentiment indicators from Alpha Vantage (e.g., `SENTIMENT` or `ECONOMIC_INDICATOR` endpoints).
- **Configuration:** HTTP GET to Alpha Vantage API with function, symbol(s), and API key from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** `$('Config').item.json.ALPHAVANTAGE_KEY` as `apikey` query parameter.
- **Input Connections:** ← Config
- **Output Connections:** → Merge All Data (input index 3)
- **Edge Cases / Failures:** Alpha Vantage free tier is limited to 25 requests/day — may exhaust quota; 5 calls/minute rate limit; response may be `None` for some indicators.
- **Version Requirements:** v4.2 HTTP Request.

---

**Fear & Greed Index**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Fetches the current CNN Fear & Greed Index value (0–100 scale) providing a market sentiment gauge.
- **Configuration:** HTTP GET to the Fear & Greed Index API endpoint; no API key typically required but may use a proxy or wrapper service configured via Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** Possibly references a URL from Config.
- **Input Connections:** ← Config
- **Output Connections:** → Merge All Data (input index 4)
- **Edge Cases / Failures:** Service unavailability → continues with error; the index is typically US-market-focused which may not directly map to Forex sentiment.
- **Version Requirements:** v4.2 HTTP Request.

---

**Forex Factory Calendar**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Scrapes or retrieves upcoming Forex economic calendar events (high-impact news releases, central bank decisions) from a Forex Factory data wrapper or scraping API.
- **Configuration:** HTTP GET to a Forex Factory calendar endpoint or scraper service; parameters may include date range and impact filter from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** URL from Config; possibly date filtering expressions.
- **Input Connections:** ← Config
- **Output Connections:** → Merge All Data (input index 5)
- **Edge Cases / Failures:** Scraping services can be blocked or rate-limited; data structure may change if Forex Factory updates their HTML; proxy may be required.
- **Version Requirements:** v4.2 HTTP Request.

---

**Merge All Data**
- **Type:** `n8n-nodes-base.merge` (v3.2)
- **Technical Role:** Combines the six parallel data-source outputs into a single item with all fields available for the AI context builder.
- **Configuration:** Default Merge mode (Append/Combine) — each input index (0–5) corresponds to one data source. Given a single item per branch, the result is one combined item containing fields from all six sources.
- **Key Expressions/Variables:** None — structural node.
- **Input Connections:** ← Get Portfolio from Signal Handler (0), NewsAPI Headlines (1), FinnHub Market News (2), Alpha Vantage Macro Sentiment (3), Fear & Greed Index (4), Forex Factory Calendar (5)
- **Output Connections:** → Build Full Context
- **Edge Cases / Failures:** If any upstream HTTP request errored and produced no valid output, the merge may receive fewer than 6 inputs. With "continue on error" enabled, error objects may be present — Build Full Context must handle potentially missing or null fields. If a branch returns multiple items, the merge's behavior depends on the combine mode (by position or by index).
- **Version Requirements:** v3.2 Merge node.

---

**Data Sources** (sticky note)
- **Type:** `n8n-nodes-base.stickyNote` (v1)
- **Content:** *(empty in JSON)*
- **Context:** Visually labels the data-fetching area of the workflow.

---

#### Block 3 — Context Building & AI Analysis

**Overview:** A Code node assembles all merged data into a structured prompt/context string. This is fed into an Anthropic LLM chain node that acts as the AI Portfolio Analyst, producing a structured recommendation (portfolio assessment + actionable trade signals).

**Nodes Involved:** Build Full Context, AI Portfolio Analyst, Anthropic Chat Model, AI Engine (sticky note)

---

**Build Full Context**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Transforms the merged raw data into a well-structured text context for the AI model, including portfolio positions, news summaries, sentiment indicators, and upcoming economic events. Likely also injects a system prompt or analysis instructions.
- **Configuration:** JavaScript code block that accesses `$('Merge All Data').item.json` and constructs a formatted string or JSON object assigned to a field (e.g., `context`). May include a hardcoded system prompt section for the AI.
- **Key Expressions/Variables:** References all merged data fields; outputs a `context` (or similarly named) field containing the full prompt.
- **Input Connections:** ← Merge All Data
- **Output Connections:** → AI Portfolio Analyst
- **Edge Cases / Failures:** If any upstream source returned an error object, the code must defensively check for null/undefined fields; malformed data structures may cause runtime JS errors; string concatenation limits (Anthropic context window) — if context exceeds ~200K tokens for Claude, the request will fail.
- **Version Requirements:** v2 Code node.

---

**AI Portfolio Analyst**
- **Type:** `@n8n/n8n-nodes-langchain.chainLlm` (v1.7)
- **Technical Role:** The core AI decision engine. Receives the full context, invokes the connected Anthropic chat model, and returns the AI's analysis as a structured output containing portfolio assessment and recommended trade actions.
- **Configuration:** Prompt template referencing the context from Build Full Context; output schema likely instructed to return a JSON structure with fields like `analysis`, `actions` (array of trade signals). The chain connects to the Anthropic Chat Model as its `ai_languageModel` sub-node.
- **Key Expressions/Variables:** `{{ $json.context }}` or similar in the prompt template; output parsing may be configured.
- **Input Connections:** ← Build Full Context (main input); ← Anthropic Chat Model (ai_languageModel connection, index 0)
- **Output Connections:** → Parse AI Response
- **Edge Cases / Failures:** Anthropic API key missing/invalid → 401; rate limit exceeded → 429; model returning non-JSON output → Parse AI Response must handle gracefully; context window overflow → 400 error; timeout on long generations.
- **Version Requirements:** v1.7 LangChain Chain LLM node; requires n8n's LangChain nodes package.
- **Sub-workflow Reference:** None — uses an in-workflow LLM sub-node.

---

**Anthropic Chat Model**
- **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1.3)
- **Technical Role:** The LLM provider sub-node for the AI Portfolio Analyst chain. Connects to Anthropic's Claude API and provides the model configuration (model name, temperature, max tokens).
- **Configuration:** Anthropic API credential selection; model likely set to `claude-3-5-sonnet` or similar; temperature tuned for analytical precision (e.g., 0.2–0.3); max tokens set high enough for detailed analysis + trade signals output.
- **Key Expressions/Variables:** API credential reference; model name.
- **Input Connections:** None (sub-node — receives calls from the chain).
- **Output Connections:** → AI Portfolio Analyst (via `ai_languageModel` connection)
- **Edge Cases / Failures:** Credential not configured → chain fails; model name deprecated → error; token limit hit → truncated response; Anthropic service outage.
- **Version Requirements:** v1.3 Anthropic LangChain chat model node.

---

**AI Engine** (sticky note)
- **Type:** `n8n-nodes-base.stickyNote` (v1)
- **Content:** *(empty in JSON)*
- **Context:** Visually labels the AI analysis section of the workflow.

---

#### Block 4 — Response Processing & Notification

**Overview:** The AI's raw text/JSON response is parsed into a structured object. One branch formats the analysis portion into a human-readable Discord message and posts it as a portfolio update. The other branch extracts the actionable trade signals for further processing (Block 5).

**Nodes Involved:** Parse AI Response, Format Discord Update, Send Update to Discord

---

**Parse AI Response**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Parses the AI Portfolio Analyst's output (which may be markdown-wrapped JSON or plain text with embedded JSON) into a clean JavaScript object. Separates the `analysis` text from the `actions` array.
- **Configuration:** JavaScript code that extracts JSON from the LLM response (possibly using regex or `JSON.parse`), handles cases where the model wraps JSON in markdown code blocks, and outputs two fields: `analysis` (string) and `actions` (array of action objects like `{type, pair, orderType, lots, price, stopLoss, takeProfit, reason}`).
- **Key Expressions/Variables:** Input from AI Portfolio Analyst output; defensive parsing for non-JSON or malformed responses.
- **Input Connections:** ← AI Portfolio Analyst
- **Output Connections:** → Format Discord Update (output 0), → Extract Actions (output 0)
- **Edge Cases / Failures:** AI returns unparseable text → code must return empty actions array and raw text as analysis; AI returns empty actions → downstream IF node will evaluate false; JSON wrapped in triple backticks must be stripped.
- **Version Requirements:** v2 Code node.

---

**Format Discord Update**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Transforms the `analysis` portion of the parsed response into a formatted Discord message (with markdown, embeds, or rich text structure).
- **Configuration:** JavaScript code that builds a Discord-compatible message payload from `$('Parse AI Response').item.json.analysis`, including formatting like bold headers, bullet points, emoji indicators (📈/📉/⚠️), and timestamp.
- **Key Expressions/Variables:** References parsed analysis text; constructs `content` field for Discord node.
- **Input Connections:** ← Parse AI Response
- **Output Connections:** → Send Update to Discord
- **Edge Cases / Failures:** Discord message length limit (2000 characters) — long analyses may need truncation or splitting into multiple messages; special characters must be escaped for Discord markdown.
- **Version Requirements:** v2 Code node.

---

**Send Update to Discord**
- **Type:** `n8n-nodes-base.discord` (v2)
- **Technical Role:** Posts the formatted portfolio analysis message to a Discord channel via a webhook.
- **Configuration:** Discord webhook URL (configured as a credential or directly); resource set to send a message. Retry on fail (3 max attempts). A dedicated webhook ID (`8cfd3e9c-7271-4899-bcc2-2b6037f23951`) is referenced, suggesting this is a distinct channel from trade confirmations.
- **Key Expressions/Variables:** Message content from Format Discord Update output.
- **Input Connections:** ← Format Discord Update
- **Output Connections:** None (terminal notification node).
- **Edge Cases / Failures:** Webhook URL revoked → 404; Discord rate limit → 429 (retried up to 3 times); message body exceeds Discord limit → 400; webhook channel permissions changed.
- **Version Requirements:** v2 Discord node.

---

#### Block 5 — Action Extraction & Routing

**Overview:** Extracts the array of trade actions from the parsed AI response, checks whether any actions exist, and if so, routes each action by type (Buy, Sell, Close Position) into the appropriate execution sub-path.

**Nodes Involved:** Extract Actions, Has Actions?, Action Type?, Execution Router (sticky note)

---

**Extract Actions**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Takes the `actions` array from Parse AI Response and splits it into individual items (one per action), enabling downstream nodes to process each action independently.
- **Configuration:** JavaScript code that reads `$('Parse AI Response').item.json.actions`, validates each action object (required fields: `type`, `pair`, `orderType`, `lots`), and returns them as separate items for downstream processing.
- **Key Expressions/Variables:** `actions` array iteration; validation of action structure.
- **Input Connections:** ← Parse AI Response
- **Output Connections:** → Has Actions?
- **Edge Cases / Failures:** If `actions` is undefined or not an array, returns zero items (IF node gets no input); invalid action objects (missing fields) should be filtered or flagged.
- **Version Requirements:** v2 Code node.

---

**Has Actions?**
- **Type:** `n8n-nodes-base.if` (v2.2)
- **Technical Role:** Conditional gate that checks whether the Extract Actions node produced any items (i.e., whether the AI recommended any trades). If true, flow continues to the Action Type router.
- **Configuration:** Condition likely checks for the existence/non-emptiness of items (e.g., `{{ $json.type }}` exists or item count > 0). Only the **true** branch is connected; the **false** branch has no connection (workflow simply ends if no actions).
- **Key Expressions/Variables:** Condition on the presence of an action type field or item existence.
- **Input Connections:** ← Extract Actions
- **Output Connections:** → Action Type? (true branch only)
- **Edge Cases / Failures:** No items reach this node → workflow ends silently; malformed action items that pass the condition but lack a `type` field → Action Type? switch falls to default/no match.
- **Version Requirements:** v2.2 IF node.

---

**Action Type?**
- **Type:** `n8n-nodes-base.switch` (v3.4)
- **Technical Role:** Routes each action based on its `type` field. Three defined routes: "Buy" → Buy sub-router, "Sell" → Sell sub-router, "Close" → Close Position Signal.
- **Configuration:** Three output rules matching `type` values: Output 0 = "Buy", Output 1 = "Sell", Output 2 = "Close". A fallback/unmatched route is not connected.
- **Key Expressions/Variables:** `{{ $json.type }}` as the routing field.
- **Input Connections:** ← Has Actions?
- **Output Connections:** Output 0 → Buy: Market or Limit?; Output 1 → Sell: Market or Limit?; Output 2 → Close Position Signal
- **Edge Cases / Failures:** Unrecognized action type (e.g., "Hold", "Modify") → falls through to no output, silently dropped; case sensitivity in `type` field must match exactly.
- **Version Requirements:** v3.4 Switch node.

---

**Execution Router** (sticky note)
- **Type:** `n8n-nodes-base.stickyNote` (v1)
- **Content:** *(empty in JSON)*
- **Context:** Visually labels the action routing and execution area.

---

#### Block 6 — Order Execution

**Overview:** Based on the routed action type, this block further distinguishes between market and limit orders for both buy and sell actions, then submits the appropriate HTTP request to the MT5 signal handler. Close Position signals are sent directly. All five execution paths converge into a single confirmation builder.

**Nodes Involved:** Buy: Market or Limit?, Sell: Market or Limit?, Buy Market Order, Buy Limit Order, Sell Market Order, Sell Limit Order, Close Position Signal

---

**Buy: Market or Limit?**
- **Type:** `n8n-nodes-base.switch` (v3.4)
- **Technical Role:** Sub-router for Buy actions. Distinguishes between `orderType: "market"` and `orderType: "limit"` and routes accordingly.
- **Configuration:** Two output rules: Output 0 = "market", Output 1 = "limit", matching against `{{ $json.orderType }}`.
- **Key Expressions/Variables:** `{{ $json.orderType }}` for routing.
- **Input Connections:** ← Action Type? (output 0)
- **Output Connections:** Output 0 → Buy Market Order; Output 1 → Buy Limit Order
- **Edge Cases / Failures:** Unrecognized orderType (e.g., "stop") → no output, action dropped; case sensitivity must match.
- **Version Requirements:** v3.4 Switch node.

---

**Sell: Market or Limit?**
- **Type:** `n8n-nodes-base.switch` (v3.4)
- **Technical Role:** Sub-router for Sell actions. Distinguishes between `orderType: "market"` and `orderType: "limit"`.
- **Configuration:** Two output rules: Output 0 = "market", Output 1 = "limit", matching against `{{ $json.orderType }}`.
- **Key Expressions/Variables:** `{{ $json.orderType }}` for routing.
- **Input Connections:** ← Action Type? (output 1)
- **Output Connections:** Output 0 → Sell Market Order; Output 1 → Sell Limit Order
- **Edge Cases / Failures:** Same as Buy sub-router — unrecognized types are dropped.
- **Version Requirements:** v3.4 Switch node.

---

**Buy Market Order**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Submits a buy market order to the MT5 signal handler via HTTP POST/GET.
- **Configuration:** HTTP request to the MT5 handler endpoint with parameters: action=buy, orderType=market, symbol (pair), volume (lots), and optional stopLoss/takeProfit. URL and auth from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** `{{ $json.pair }}`, `{{ $json.lots }}`, `{{ $json.stopLoss }}`, `{{ $json.takeProfit }}`, `$('Config').item.json.MT5_HANDLER_URL`.
- **Input Connections:** ← Buy: Market or Limit? (output 0)
- **Output Connections:** → Build Execution Confirmation
- **Edge Cases / Failures:** MT5 service down → order not placed; invalid symbol → MT5 rejects; insufficient margin → MT5 error response; network timeout after retries → continues with error output.
- **Version Requirements:** v4.2 HTTP Request.

---

**Buy Limit Order**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Submits a buy limit order to the MT5 signal handler.
- **Configuration:** HTTP request with action=buy, orderType=limit, symbol, volume, price (the limit price), stopLoss, takeProfit. URL and auth from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** Same as Buy Market Order plus `{{ $json.price }}` for the limit price.
- **Input Connections:** ← Buy: Market or Limit? (output 1)
- **Output Connections:** → Build Execution Confirmation
- **Edge Cases / Failures:** Same as Buy Market Order; additionally, limit price may be rejected if too far from current market price.
- **Version Requirements:** v4.2 HTTP Request.

---

**Sell Market Order**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Submits a sell market order to the MT5 signal handler.
- **Configuration:** HTTP request with action=sell, orderType=market, symbol, volume, stopLoss, takeProfit. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** Same pattern as Buy Market Order.
- **Input Connections:** ← Sell: Market or Limit? (output 0)
- **Output Connections:** → Build Execution Confirmation
- **Edge Cases / Failures:** Same category as buy orders.
- **Version Requirements:** v4.2 HTTP Request.

---

**Sell Limit Order**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Submits a sell limit order to the MT5 signal handler.
- **Configuration:** HTTP request with action=sell, orderType=limit, symbol, volume, price, stopLoss, takeProfit. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** Same pattern as Buy Limit Order.
- **Input Connections:** ← Sell: Market or Limit? (output 1)
- **Output Connections:** → Build Execution Confirmation
- **Edge Cases / Failures:** Same category as buy limit orders.
- **Version Requirements:** v4.2 HTTP Request.

---

**Close Position Signal**
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Technical Role:** Sends a close-position signal to the MT5 handler for a specific existing position (identified by ticket or symbol).
- **Configuration:** HTTP request with action=close, symbol or position ticket, and volume. URL and auth from Config. Retry on fail (2 attempts); continue on error.
- **Key Expressions/Variables:** `{{ $json.pair }}` or `{{ $json.ticket }}`, `$('Config').item.json.MT5_HANDLER_URL`.
- **Input Connections:** ← Action Type? (output 2)
- **Output Connections:** → Build Execution Confirmation
- **Edge Cases / Failures:** Position already closed or not found → MT5 error; invalid ticket → rejection; partial close not supported → may require full volume.
- **Version Requirements:** v4.2 HTTP Request.

---

#### Block 7 — Confirmation

**Overview:** All five execution paths converge into a single Code node that builds a human-readable trade confirmation message, which is then sent to a dedicated Discord channel (separate from the portfolio update channel).

**Nodes Involved:** Build Execution Confirmation, Send Trade Confirmation

---

**Build Execution Confirmation**
- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Assembles a trade execution confirmation message from the order response data, including action type, pair, order type, lots, price, timestamps, and any MT5 response details (order ticket, status).
- **Configuration:** JavaScript code that reads the incoming item (from any of the 5 execution nodes) and constructs a formatted Discord message payload, likely with emoji indicators (✅ executed, ❌ failed) and structured fields.
- **Key Expressions/Variables:** References the HTTP response from the MT5 handler plus the original action fields.
- **Input Connections:** ← Buy Market Order, ← Buy Limit Order, ← Sell Market Order, ← Sell Limit Order, ← Close Position Signal (all five converge here)
- **Output Connections:** → Send Trade Confirmation
- **Edge Cases / Failures:** If the MT5 handler returned an error response, the confirmation should indicate failure; missing fields in the response → defensive coding required.
- **Version Requirements:** v2 Code node.

---

**Send Trade Confirmation**
- **Type:** `n8n-nodes-base.discord` (v2)
- **Technical Role:** Posts the trade execution confirmation to a Discord channel via a webhook (separate from the portfolio update webhook).
- **Configuration:** Discord webhook URL (configured as credential or directly); separate webhook ID (`d052a1a5-25b6-413d-8c0d-ca2a84e6bcea`) indicating a different channel. Retry on fail (3 max attempts).
- **Key Expressions/Variables:** Message content from Build Execution Confirmation.
- **Input Connections:** ← Build Execution Confirmation
- **Output Connections:** None (terminal notification node).
- **Edge Cases / Failures:** Same as Send Update to Discord — webhook revocation, rate limits, message length.
- **Version Requirements:** v2 Discord node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Hour Trigger | scheduleTrigger (v1.3) | Hourly workflow entry point | None | Config | Sticky Note |
| Config | set (v3.4) | Runtime configuration and credential references | Every Hour Trigger | Get Portfolio from Signal Handler, NewsAPI Headlines, FinnHub Market News, Alpha Vantage Macro Sentiment, Fear & Greed Index, Forex Factory Calendar | Sticky Note, Overview |
| Get Portfolio from Signal Handler | httpRequest (v4.2) | Fetch current portfolio from MT5 handler | Config | Merge All Data | Data Sources, Overview |
| NewsAPI Headlines | httpRequest (v4.2) | Fetch Forex news headlines from NewsAPI | Config | Merge All Data | Data Sources |
| FinnHub Market News | httpRequest (v4.2) | Fetch market news from FinnHub | Config | Merge All Data | Data Sources |
| Alpha Vantage Macro Sentiment | httpRequest (v4.2) | Fetch macro sentiment indicators from Alpha Vantage | Config | Merge All Data | Data Sources |
| Fear & Greed Index | httpRequest (v4.2) | Fetch Fear & Greed sentiment index | Config | Merge All Data | Data Sources |
| Forex Factory Calendar | httpRequest (v4.2) | Fetch economic calendar events from Forex Factory | Config | Merge All Data | Data Sources |
| Merge All Data | merge (v3.2) | Combine all 6 data sources into single item | Get Portfolio from Signal Handler, NewsAPI Headlines, FinnHub Market News, Alpha Vantage Macro Sentiment, Fear & Greed Index, Forex Factory Calendar | Build Full Context | Data Sources |
| Build Full Context | code (v2) | Assemble structured AI prompt from merged data | Merge All Data | AI Portfolio Analyst | AI Engine |
| AI Portfolio Analyst | chainLlm (v1.7) | AI-powered portfolio analysis and trade signal generation | Build Full Context | Parse AI Response | AI Engine |
| Anthropic Chat Model | lmChatAnthropic (v1.3) | LLM provider for AI Portfolio Analyst (Claude) | *(sub-node of AI Portfolio Analyst)* | AI Portfolio Analyst (ai_languageModel) | AI Engine |
| Parse AI Response | code (v2) | Parse AI output into analysis text and actions array | AI Portfolio Analyst | Format Discord Update, Extract Actions | AI Engine, Execution Router |
| Format Discord Update | code (v2) | Format portfolio analysis for Discord message | Parse AI Response | Send Update to Discord | Execution Router |
| Send Update to Discord | discord (v2) | Post portfolio analysis update to Discord webhook | Format Discord Update | None | Execution Router |
| Extract Actions | code (v2) | Split actions array into individual items | Parse AI Response | Has Actions? | Execution Router |
| Has Actions? | if (v2.2) | Gate: proceed only if trade actions exist | Extract Actions | Action Type? (true only) | Execution Router |
| Action Type? | switch (v3.4) | Route action by type: Buy / Sell / Close | Has Actions? | Buy: Market or Limit?, Sell: Market or Limit?, Close Position Signal | Execution Router |
| Buy: Market or Limit? | switch (v3.4) | Sub-route buy orders by order type | Action Type? | Buy Market Order, Buy Limit Order | Execution Router |
| Buy Market Order | httpRequest (v4.2) | Submit buy market order to MT5 handler | Buy: Market or Limit? | Build Execution Confirmation | Execution Router |
| Buy Limit Order | httpRequest (v4.2) | Submit buy limit order to MT5 handler | Buy: Market or Limit? | Build Execution Confirmation | Execution Router |
| Sell: Market or Limit? | switch (v3.4) | Sub-route sell orders by order type | Action Type? | Sell Market Order, Sell Limit Order | Execution Router |
| Sell Market Order | httpRequest (v4.2) | Submit sell market order to MT5 handler | Sell: Market or Limit? | Build Execution Confirmation | Execution Router |
| Sell Limit Order | httpRequest (v4.2) | Submit sell limit order to MT5 handler | Sell: Market or Limit? | Build Execution Confirmation | Execution Router |
| Close Position Signal | httpRequest (v4.2) | Submit close position signal to MT5 handler | Action Type? | Build Execution Confirmation | Execution Router |
| Build Execution Confirmation | code (v2) | Build trade execution confirmation message | Buy Market Order, Buy Limit Order, Sell Market Order, Sell Limit Order, Close Position Signal | Send Trade Confirmation | Execution Router |
| Send Trade Confirmation | discord (v2) | Post trade execution confirmation to Discord webhook | Build Execution Confirmation | None | Execution Router |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Automated Forex Portfolio Manager with AI, MT5 & News Sentiment Analysis".

2. **Add the Schedule Trigger:**
   - Node type: **Schedule Trigger** (v1.3)
   - Name: `Every Hour Trigger`
   - Configuration: Set to trigger every hour (default hourly interval).

3. **Add the Config node:**
   - Node type: **Set** (v3.4)
   - Name: `Config`
   - Add the following fields (key-value pairs):
     - `MT5_HANDLER_URL` — base URL of your MT5 signal handler API (e.g., `https://your-server.com/mt5`)
     - `MT5_AUTH_KEY` — authentication key/token for the MT5 handler
     - `NEWSAPI_KEY` — your NewsAPI.org API key
     - `FINNHUB_KEY` — your FinnHub API token
     - `ALPHAVANTAGE_KEY` — your Alpha Vantage API key
     - `PORTFOLIO_ID` — identifier for the portfolio to query
     - `DISCORD_WEBHOOK_UPDATE_URL` — Discord webhook URL for portfolio updates
     - `DISCORD_WEBHOOK_TRADE_URL` — Discord webhook URL for trade confirmations
     - `ANTHROPIC_MODEL` — model name (e.g., `claude-3-5-sonnet-20241022`)
   - Connect: **Every Hour Trigger** → **Config**

4. **Add six HTTP Request nodes** (data sources) in parallel, all connected from Config:
   - **Get Portfolio from Signal Handler:**
     - Type: **HTTP Request** (v4.2)
     - Method: GET
     - URL: `{{ $('Config').item.json.MT5_HANDLER_URL }}/portfolio/{{ $('Config').item.json.PORTFOLIO_ID }}`
     - Authentication: Header Auth with key from `MT5_AUTH_KEY`
     - Retry on Fail: Enabled, max tries: 2
     - On Error: Continue Regular Output
     - Connect: Config → this node → Merge All Data (input 0)
   
   - **NewsAPI Headlines:**
     - Type: **HTTP Request** (v4.2)
     - Method: GET
     - URL: `https://newsapi.org/v2/top-headlines`
     - Query Parameters: `q=forex OR currency OR "interest rate"`, `language=en`, `apiKey={{ $('Config').item.json.NEWSAPI_KEY }}`
     - Retry on Fail: Enabled, max tries: 2; On Error: Continue Regular Output
     - Connect: Config → this node → Merge All Data (input 1)
   
   - **FinnHub Market News:**
     - Type: **HTTP Request** (v4.2)
     - Method: GET
     - URL: `https://finnhub.io/api/v1/news`
     - Query Parameters: `category=forex`, `token={{ $('Config').item.json.FINNHUB_KEY }}`
     - Retry on Fail: Enabled, max tries: 2; On Error: Continue Regular Output
     - Connect: Config → this node → Merge All Data (input 2)
   
   - **Alpha Vantage Macro Sentiment:**
     - Type: **HTTP Request** (v4.2)
     - Method: GET
     - URL: `https://www.alphavantage.co/query`
     - Query Parameters: `function=SENTIMENT`, `symbol=USD`, `apikey={{ $('Config').item.json.ALPHAVANTAGE_KEY }}`
     - Retry on Fail: Enabled, max tries: 2; On Error: Continue Regular Output
     - Connect: Config → this node → Merge All Data (input 3)
   
   - **Fear & Greed Index:**
     - Type: **HTTP Request** (v4.2)
     - Method: GET
     - URL: `https://api.alternative.me/fng/` (or your preferred endpoint)
     - Retry on Fail: Enabled, max tries: 2; On Error: Continue Regular Output
     - Connect: Config → this node → Merge All Data (input 4)
   
   - **Forex Factory Calendar:**
     - Type: **HTTP Request** (v4.2)
     - Method: GET
     - URL: Your Forex Factory calendar scraper/API endpoint
     - Query Parameters: impact filter for high-impact events, date range for upcoming week
     - Retry on Fail: Enabled, max tries: 2; On Error: Continue Regular Output
     - Connect: Config → this node → Merge All Data (input 5)

5. **Add the Merge node:**
   - Type: **Merge** (v3.2)
   - Name: `Merge All Data`
   - Mode: Combine (by position or index — adjust based on how you want the 6 inputs combined)
   - Connect all six data-source outputs into inputs 0–5 respectively.
   - Connect output → Build Full Context

6. **Add Build Full Context:**
   - Type: **Code** (v2)
   - Name: `Build Full Context`
   - Language: JavaScript
   - Code logic:
     - Retrieve all fields from the merged item
     - Construct a structured prompt string containing: portfolio positions, news summaries, sentiment indicators, economic calendar events
     - Prepend a system prompt instructing the AI to analyze the portfolio and return a JSON object with `analysis` (string) and `actions` (array)
     - Output as a field named `context`
   - Connect: Merge All Data → this node → AI Portfolio Analyst

7. **Add the Anthropic Chat Model sub-node:**
   - Type: **Anthropic Chat Model** (LangChain, v1.3)
   - Name: `Anthropic Chat Model`
   - Credential: Create an Anthropic API credential with your API key
   - Model: `claude-3-5-sonnet-20241022` (or your preferred model)
   - Temperature: 0.2 (low for analytical consistency)
   - Max Tokens: 4096 or higher depending on expected output size
   - Do **not** connect this node from the main flow — it is a sub-node connected via the `ai_languageModel` input on the chain node

8. **Add AI Portfolio Analyst:**
   - Type: **Chain LLM** (LangChain, v1.7)
   - Name: `AI Portfolio Analyst`
   - Prompt/Message: Reference `{{ $json.context }}` as the user message content
   - Connect the `ai_languageModel` input to **Anthropic Chat Model** (drag from Anthropic Chat Model's output connector to the chain's ai_languageModel input)
   - Connect main input: Build Full Context → AI Portfolio Analyst
   - Connect main output: AI Portfolio Analyst → Parse AI Response

9. **Add Parse AI Response:**
   - Type: **Code** (v2)
   - Name: `Parse AI Response`
   - Code logic:
     - Extract the AI output text
     - Strip markdown code fences if present (e.g., ````json ... ``` ``)
     - Parse JSON to obtain `{analysis, actions}`
     - If parsing fails, set `analysis` to the raw text and `actions` to an empty array
     - Output two fields: `analysis` (string) and `actions` (array)
   - Connect output 0 → Format Discord Update
   - Connect output 0 → Extract Actions (same output, two connections)

10. **Add Format Discord Update:**
    - Type: **Code** (v2)
    - Name: `Format Discord Update`
    - Code logic: Build a Discord message from `analysis` with emoji headers, section dividers, and timestamps. Output a `content` field.
    - Connect: Parse AI Response → this node → Send Update to Discord

11. **Add Send Update to Discord:**
    - Type: **Discord** (v2)
    - Name: `Send Update to Discord`
    - Resource: Send Message
    - Webhook URL: `{{ $('Config').item.json.DISCORD_WEBHOOK_UPDATE_URL }}` (or configure as a Discord Webhook credential)
    - Retry on Fail: Enabled, max tries: 3
    - Connect: Format Discord Update → this node (terminal)

12. **Add Extract Actions:**
    - Type: **Code** (v2)
    - Name: `Extract Actions`
    - Code logic: Read `actions` array from Parse AI Response, validate each action has `type`, `pair`, `orderType`, `lots` (at minimum), and return each as a separate item.
    - Connect: Parse AI Response → this node → Has Actions?

13. **Add Has Actions?:**
    - Type: **IF** (v2.2)
    - Name: `Has Actions?`
    - Condition: Check that `{{ $json.type }}` exists and is not empty (or check item count)
    - True branch → Action Type?
    - False branch: leave unconnected (workflow ends for this path)
    - Connect: Extract Actions → this node

14. **Add Action Type?:**
    - Type: **Switch** (v3.4)
    - Name: `Action Type?`
    - Routing field: `{{ $json.type }}`
    - Rules:
      - Output 0: Value equals `Buy`
      - Output 1: Value equals `Sell`
      - Output 2: Value equals `Close`
    - Connect: Has Actions? → this node
    - Output 0 → Buy: Market or Limit?
    - Output 1 → Sell: Market or Limit?
    - Output 2 → Close Position Signal

15. **Add Buy: Market or Limit?:**
    - Type: **Switch** (v3.4)
    - Name: `Buy: Market or Limit?`
    - Routing field: `{{ $json.orderType }}`
    - Rules:
      - Output 0: Value equals `market`
      - Output 1: Value equals `limit`
    - Connect: Action Type? output 0 → this node
    - Output 0 → Buy Market Order
    - Output 1 → Buy Limit Order

16. **Add Sell: Market or Limit?:**
    - Type: **Switch** (v3.4)
    - Name: `Sell: Market or Limit?`
    - Routing field: `{{ $json.orderType }}`
    - Rules:
      - Output 0: Value equals `market`
      - Output 1: Value equals `limit`
    - Connect: Action Type? output 1 → this node
    - Output 0 → Sell Market Order
    - Output 1 → Sell Limit Order

17. **Add the five order execution HTTP Request nodes:**
    - **Buy Market Order:**
      - Type: **HTTP Request** (v4.2)
      - Method: POST
      - URL: `{{ $('Config').item.json.MT5_HANDLER_URL }}/order`
      - Body (JSON): `{ "action": "buy", "orderType": "market", "symbol": "{{ $json.pair }}", "volume": "{{ $json.lots }}", "stopLoss": "{{ $json.stopLoss }}", "takeProfit": "{{ $json.takeProfit }}" }`
      - Auth: Header with `MT5_AUTH_KEY`
      - Retry on Fail: Enabled, max 2; On Error: Continue
      - Connect: Buy: Market or Limit? output 0 → this node → Build Execution Confirmation

    - **Buy Limit Order:**
      - Same as above but `orderType: "limit"` and add `"price": "{{ $json.price }}"` to body
      - Connect: Buy: Market or Limit? output 1 → this node → Build Execution Confirmation

    - **Sell Market Order:**
      - Same structure as Buy Market Order but `action: "sell"`
      - Connect: Sell: Market or Limit? output 0 → this node → Build Execution Confirmation

    - **Sell Limit Order:**
      - Same structure as Buy Limit Order but `action: "sell"`
      - Connect: Sell: Market or Limit? output 1 → this node → Build Execution Confirmation

    - **Close Position Signal:**
      - Type: **HTTP Request** (v4.2)
      - Method: POST
      - URL: `{{ $('Config').item.json.MT5_HANDLER_URL }}/close`
      - Body: `{ "symbol": "{{ $json.pair }}", "volume": "{{ $json.lots }}" }` (or `"ticket": "{{ $json.ticket }}"`)
      - Auth: Same as above
      - Retry on Fail: Enabled, max 2; On Error: Continue
      - Connect: Action Type? output 2 → this node → Build Execution Confirmation

18. **Add Build Execution Confirmation:**
    - Type: **Code** (v2)
    - Name: `Build Execution Confirmation`
    - Code logic: Read the original action fields and the MT5 HTTP response, construct a confirmation message (✅/❌ indicator, action type, pair, lots, price, order ticket if available, timestamp). Output `content` field.
    - Connect all five execution nodes → this node → Send Trade Confirmation

19. **Add Send Trade Confirmation:**
    - Type: **Discord** (v2)
    - Name: `Send Trade Confirmation`
    - Resource: Send Message
    - Webhook URL: `{{ $('Config').item.json.DISCORD_WEBHOOK_TRADE_URL }}` (or separate Discord Webhook credential)
    - Retry on Fail: Enabled, max tries: 3
    - Connect: Build Execution Confirmation → this node (terminal)

20. **Add Sticky Notes for documentation** (optional but recommended):
    - `Overview` — placed near the trigger/config area
    - `Data Sources` — placed above the 6 HTTP data-source nodes
    - `AI Engine` — placed around Build Full Context, AI Portfolio Analyst, Anthropic Chat Model, and Parse AI Response
    - `Execution Router` — placed around Extract Actions through Build Execution Confirmation

21. **Credential Setup:**
    - **Anthropic API Key:** Navigate to Credentials → Add Credential → Anthropic API → paste your API key. Assign this credential to the Anthropic Chat Model node.
    - **Discord Webhooks:** Create two Discord webhook credentials (or use URL-based configuration): one for portfolio updates, one for trade confirmations. Assign to the respective Discord nodes.
    - **MT5 Handler Authentication:** If your MT5 handler requires API-key-based auth, create a Header Auth credential and assign to all 6 MT5-related HTTP Request nodes (Get Portfolio + 5 order nodes).
    - **External API Keys:** NewsAPI, FinnHub, and Alpha Vantage keys are referenced via the Config node's Set fields and injected as query parameters in the HTTP Request nodes. No separate n8n credentials are needed unless you prefer credential-based auth.

22. **Sub-workflow Note:** This workflow does not call or include sub-workflows. All logic is self-contained. The Anthropic Chat Model is a LangChain sub-node (connected via `ai_languageModel` input), not a separate workflow.

23. **Test the workflow:**
    - Manually trigger by clicking "Execute Node" on the Every Hour Trigger.
    - Verify Config outputs contain all expected fields.
    - Verify each HTTP Request node returns data (check for 401s indicating missing keys).
    - Verify Merge All Data produces a single combined item.
    - Verify Build Full Context output is a well-formed prompt string.
    - Verify AI Portfolio Analyst returns a response containing analysis and actions JSON.
    - Verify Parse AI Response correctly extracts both fields.
    - Verify Discord updates appear in the correct channels.
    - Verify action routing: submit a test with `actions: [{type: "Buy", pair: "EURUSD", orderType: "market", lots: 0.1}]` and trace through to the Buy Market Order and confirmation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is designed for Forex trading automation and connects to live financial markets. Use on demo accounts first. | Risk warning — real-money Forex trading carries significant financial risk. |
| The MT5 Signal Handler is an external service (not bundled with n8n). It must expose REST endpoints for portfolio retrieval, order placement, and position closing. | Requires a separate MT5 integration bridge (e.g., a custom Python/Node service or a pre-built MT5 REST API wrapper). |
| Anthropic Claude models support up to 200K context tokens. Ensure the Build Full Context output does not exceed this limit, especially with verbose news feeds. | https://docs.anthropic.com/en/docs/about-claude/models |
| NewsAPI free tier allows 100 requests/day and 100 results per request. The hourly trigger makes 24 calls/day, well within limits. | https://newsapi.org/pricing |
| FinnHub free tier allows 60 calls/minute. Hourly usage is well within limits. | https://finnhub.io/docs/api/rate-limit |
| Alpha Vantage free tier is limited to 25 requests/day and 5 per minute. With hourly execution, you will exhaust the daily quota. Consider a premium tier or caching. | https://www.alphavantage.co/premium/ |
| The Fear & Greed Index from Alternative.me has no official rate limit documentation but is generally permissive. | https://alternative.me/crypto/fear-and-greed-index/ |
| Forex Factory does not offer an official API. The workflow assumes a scraping wrapper or third-party calendar API is available. | Requires self-hosted or third-party Forex Factory calendar scraper. |
| Two separate Discord webhooks are used: one for portfolio analysis updates, one for trade execution confirmations. This separates informational updates from actionable alerts. | Configure in Discord: Server Settings → Integrations → Webhooks |
| The workflow's "continue on error" pattern on all HTTP data-source nodes means partial data (e.g., only 4 of 6 sources succeed) will still flow through. Build Full Context and downstream nodes must handle missing/null fields gracefully. | Defensive coding best practice for production resilience. |
| The `Has Actions?` IF node only has a true branch connected. When no actions exist, the workflow silently completes after the Discord update is sent. This is intentional — the portfolio update always goes out, but execution only occurs when the AI recommends trades. | Design decision: analysis is always reported; trades are conditionally executed. |
| All switch nodes use exact string matching on `type` and `orderType` fields. The AI prompt must enforce consistent casing and values (e.g., "Buy"/"Sell"/"Close", "market"/"limit"). | Prompt engineering critical for reliable routing. |