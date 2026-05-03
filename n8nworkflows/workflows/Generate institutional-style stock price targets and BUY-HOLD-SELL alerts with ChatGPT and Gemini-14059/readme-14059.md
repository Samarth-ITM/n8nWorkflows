Generate institutional-style stock price targets and BUY/HOLD/SELL alerts with ChatGPT and Gemini

https://n8nworkflows.xyz/workflows/generate-institutional-style-stock-price-targets-and-buy-hold-sell-alerts-with-chatgpt-and-gemini-14059


# Generate institutional-style stock price targets and BUY/HOLD/SELL alerts with ChatGPT and Gemini

## 1. Workflow Overview

This workflow is an automated institutional-style stock valuation pipeline built in n8n. It periodically reads a stock watchlist from Google Sheets, fetches or reuses cached financial fundamentals, enriches them with Seeking Alpha news, runs two independent LLM valuation rounds in parallel (ChatGPT and Gemini), resolves disagreements with a bull/bear tiebreaker if needed, stores the final structured valuation output in Google Sheets, and sends Telegram alerts and a batch summary.

### Main use cases
- Scheduled portfolio/watchlist review
- Automated BUY/HOLD/SELL signal generation
- Scenario-based price target generation (bear/base/bull)
- Historical logging of AI-generated stock valuations
- Telegram push alerts for actionable or thesis-changing signals

### 1.1 Entry, Watchlist Intake, and Batch Control
The workflow starts on a schedule, loads tickers from a Google Sheet, and processes them one by one using a batch loop.

### 1.2 Financial Cache Lookup and Fundamentals Refresh
For each ticker, the workflow checks a financial cache sheet. If the ticker is not cached or manually flagged for refresh, it calls Alpha Vantage endpoints with wait nodes to avoid throttling, then computes normalized metrics and Piotroski F-Score and updates/inserts cache rows.

### 1.3 News Retrieval and Normalization
In parallel with financial handling, the workflow fetches Seeking Alpha XML news, validates whether usable XML was returned, parses it, filters recent articles, deduplicates them, and creates a compact text block for prompting.

### 1.4 Dual-LLM Valuation Round
Financial data and news data are merged and sent simultaneously to ChatGPT and Gemini. Each model is prompted to produce disciplined JSON-only valuation outputs with bear/base/bull targets, confidence, and rationale.

### 1.5 Disagreement Detection and Tiebreaker Resolution
The workflow compares the first-round outputs. If verdicts differ or model base targets diverge materially, a tiebreaker is triggered: ChatGPT is forced into a bull stance and Gemini into a bear stance. Their outputs are then combined into a final tiebreaker result.

### 1.6 Output Validation, Enrichment, and Alert Preparation
The final result is normalized into one or more rows, invalid/error rows are filtered out, previous verdict history is checked, thesis reversals are detected, and alert formatting is prepared.

### 1.7 Technical Entry Filters and Telegram Delivery
For alert-worthy signals, the workflow optionally enriches with 50-day SMA, 200-day SMA, RSI, and earnings date data. It then routes results either to an immediate Telegram alert or to a watchlist-style alert if the trend filter is weaker.

### 1.8 Persistence, Loop Continuation, and Batch Summary
The final structured rows are appended to a results sheet. After each item, the workflow loops back for the next ticker. When the batch finishes, it builds and sends a Telegram summary report.

---

## 2. Block-by-Block Analysis

## 2.1 Entry, Watchlist Intake, and Batch Control

**Overview:**  
This block launches the workflow on a schedule, reads the watchlist from Google Sheets, and iterates through each stock ticker one at a time. It also handles the end-of-batch summary path through the Split in Batches node’s completion output.

**Nodes Involved:**  
- Schedule Trigger
- Read_tickers_from_Sheet
- loop_over_tickers
- Build Summary
- Send a text message
- Wait

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; starts the workflow automatically.
- **Configuration choices:** Runs every 3 days at hour 16.
- **Key expressions/variables used:** None.
- **Input and output connections:**  
  - Output → `Read_tickers_from_Sheet`
- **Version-specific requirements:** v1.2.
- **Edge cases/failures:** Timezone interpretation depends on workflow/server timezone; if timezone is not what the author expects, execution time may drift.

#### Read_tickers_from_Sheet
- **Type and role:** Google Sheets read node; loads watchlist rows.
- **Configuration choices:** Reads from spreadsheet `List of stocks`, sheet `Sheet1` (`gid=0`).
- **Key expressions/variables used:** None visible; assumes rows contain `stock`.
- **Input and output connections:**  
  - Input ← `Schedule Trigger`
  - Output → `loop_over_tickers`
- **Version-specific requirements:** v4.6 with Google Sheets OAuth2 credentials.
- **Edge cases/failures:** OAuth expiry, missing sheet access, empty rows, column mismatch if `stock` header is renamed.

#### loop_over_tickers
- **Type and role:** `Split In Batches`; iterates watchlist items sequentially.
- **Configuration choices:** Default batch behavior, one-by-one processing pattern.
- **Key expressions/variables used:** Downstream nodes refer to `$('loop_over_tickers').item.json.stock`.
- **Input and output connections:**  
  - Input ← `Read_tickers_from_Sheet`, `If` false branch, `Wait`
  - Main output → `Cache Lookup`, `Seekingalpha Articles`
  - Done output → `Build Summary`
- **Version-specific requirements:** v3.
- **Edge cases/failures:** If incoming rows lack `stock`, almost all downstream expressions fail or return invalid requests.

#### Build Summary
- **Type and role:** Code node; aggregates completed stock results into one final summary message.
- **Configuration choices:** Reads all accumulated items, formats icons, verdict, confidence, MA signal, price, and target.
- **Key expressions/variables used:** Uses fields like `stock`, `alert_verdict`, `confidence`, `current_price`, `pt_base`, `ma_signal`.
- **Input and output connections:**  
  - Input ← `loop_over_tickers` done output
  - Output → `Send a text message`
- **Version-specific requirements:** v2 code node.
- **Edge cases/failures:** If no items reach the done branch, emits “no stocks analyzed”; if partial rows lack fields, summary still degrades gracefully.

#### Send a text message
- **Type and role:** Telegram node; sends final batch summary.
- **Configuration choices:** Sends `{{$json.summary_message}}` to static `chatId` `123456789`.
- **Key expressions/variables used:** `summary_message`
- **Input and output connections:**  
  - Input ← `Build Summary`
- **Version-specific requirements:** v1.2 with Telegram credentials.
- **Edge cases/failures:** Invalid bot token/chat ID, Telegram markdown formatting issues are avoided here because no parse mode is set.

#### Wait
- **Type and role:** Wait node; delays before returning to batch loop.
- **Configuration choices:** Waits 15 units (in n8n wait node this is typically seconds unless otherwise configured).
- **Key expressions/variables used:** None.
- **Input and output connections:**  
  - Input ← `write_sentiment_to_sheets`
  - Output → `loop_over_tickers`
- **Version-specific requirements:** v1.1.
- **Edge cases/failures:** Long batch runs can accumulate significant time; webhook-based resume must remain valid.

---

## 2.2 Financial Cache Lookup and Fundamentals Refresh

**Overview:**  
This block determines whether fresh financial data must be fetched or whether cached data can be reused. When fetching is required, it throttles Alpha Vantage API calls with Wait nodes, normalizes responses, computes valuation-relevant metrics and F-Score, and upserts the financial cache sheet.

**Nodes Involved:**  
- Cache Lookup
- Check the cache
- Is Cache Valid?
- Wait1
- Wait2
- Wait3
- Wait4
- Wait5
- Wait6
- alphavantage - Balance Sheet
- alphavantage - Profile
- alphavantage - Income Statement
- alphavantage - CashFlow
- alphavantage - Current Price
- alphavantage - Current Price Second
- Clean balance sheet
- Clean Profile
- Clean Income statement
- Clean Ccashflow
- Clean Current Price
- Clean Current Price1
- Merge3
- Merge2
- Merge5
- Merge4
- Clean Read Financial
- If2
- Update row in sheet
- Insert Row
- Update row in sheet1
- Get row(s) in sheet
- Clean Old Financial
- Merge New Financial

### Node Details

#### Cache Lookup
- **Type and role:** Google Sheets lookup; checks cache by stock symbol.
- **Configuration choices:** Reads first matching row from `Financial_Data_Cache`, sheet `Sheet1`.
- **Key expressions/variables used:** `={{ $json.stock }}`
- **Input and output connections:**  
  - Input ← `loop_over_tickers`
  - Output → `Check the cache`
- **Version-specific requirements:** v4.7.
- **Edge cases/failures:** `alwaysOutputData: true` helps the branch continue even if there is no match.

#### Check the cache
- **Type and role:** Code node; decides whether to fetch or reuse financials.
- **Configuration choices:**  
  - Checks `need_fresh_financial` flag from cache row or input.
  - Converts various truthy formats safely.
  - Sets `cacheHit`, `shouldFetch`, `reason`.
- **Key expressions/variables used:** `row?.need_fresh_financial`, `$json.stock`
- **Input and output connections:**  
  - Input ← `Cache Lookup`
  - Output → `Is Cache Valid?`
- **Edge cases/failures:** If the cache schema changes, manual refresh flag may stop working. TTL freshness is not actually date-based here; it is effectively cache-hit or manual-refresh driven.

#### Is Cache Valid?
- **Type and role:** If node; routes to fetch path if `shouldFetch === true`.
- **Configuration choices:** Tests `{{$json.shouldFetch}}`.
- **Input and output connections:**  
  - Input ← `Check the cache`
  - True → `Wait1`, `Wait2`, `Wait3`, `Wait4`, `Wait5`, `Wait6`
  - False branch unused directly from this node, but cached path resumes later via `Update row in sheet1` chain
- **Edge cases/failures:** Naming is slightly misleading; true branch means fetch is needed.

#### Wait1 / Wait2 / Wait3 / Wait4 / Wait5 / Wait6
- **Type and role:** Wait nodes; stagger Alpha Vantage calls.
- **Configuration choices:**  
  - Wait1: 15  
  - Wait2: 20  
  - Wait3: 30  
  - Wait4: 25  
  - Wait5: 20  
  - Wait6: 20
- **Input and output connections:**  
  - Wait1 → `alphavantage - Balance Sheet`
  - Wait2 → `alphavantage - Profile`
  - Wait3 → `alphavantage - Income Statement`
  - Wait4 → `alphavantage - CashFlow`
  - Wait5 → `alphavantage - Current Price`
  - Wait6 → `alphavantage - Current Price Second`
- **Edge cases/failures:** These are essential due to Alpha Vantage throttling. If API plan changes, timings may be too short or overly conservative.

#### alphavantage - Balance Sheet
- **Type and role:** HTTP Request; fetches balance sheet JSON.
- **Configuration choices:** Uses Alpha Vantage `BALANCE_SHEET` function with ticker expression.
- **Key expressions/variables used:** `{{ $('loop_over_tickers').item.json.stock }}`
- **Input and output connections:**  
  - Input ← `Wait1`
  - Output → `Clean balance sheet`
- **Edge cases/failures:** API key invalid, rate limit note payload instead of expected JSON, invalid symbols.

#### alphavantage - Profile
- **Type and role:** HTTP Request; fetches overview/profile JSON.
- **Configuration choices:** Calls `OVERVIEW`.
- **Input/output:**  
  - Input ← `Wait2`
  - Output → `Clean Profile`

#### alphavantage - Income Statement
- **Type and role:** HTTP Request; fetches `INCOME_STATEMENT`.
- **Input/output:**  
  - Input ← `Wait3`
  - Output → `Clean Income statement`

#### alphavantage - CashFlow
- **Type and role:** HTTP Request; fetches `CASH_FLOW`.
- **Input/output:**  
  - Input ← `Wait4`
  - Output → `Clean Ccashflow`

#### alphavantage - Current Price
- **Type and role:** HTTP Request; fetches `GLOBAL_QUOTE`.
- **Input/output:**  
  - Input ← `Wait5`
  - Output → `Clean Current Price`

#### alphavantage - Current Price Second
- **Type and role:** Second HTTP Request to `GLOBAL_QUOTE`.
- **Technical role:** Appears to refresh current price again before retrieving cached row for downstream consistency/update.
- **Input/output:**  
  - Input ← `Wait6`
  - Output → `Clean Current Price1`
- **Edge cases/failures:** Redundant API usage; increases rate-limit exposure.

#### Clean balance sheet
- **Type and role:** Code node; wraps balance JSON into `balance_data`.
- **Configuration choices:** Tries to preserve `stock` from multiple possible key names.
- **Input/output:**  
  - Input ← `alphavantage - Balance Sheet`
  - Output → `Merge2`

#### Clean Profile
- **Type and role:** Code node; wraps overview JSON into `overview_data`.
- **Input/output:**  
  - Input ← `alphavantage - Profile`
  - Output → `Merge3`

#### Clean Income statement
- **Type and role:** Code node; wraps income statement JSON into `income_data`.
- **Input/output:**  
  - Input ← `alphavantage - Income Statement`
  - Output → `Merge3`

#### Clean Ccashflow
- **Type and role:** Code node; wraps cash flow JSON into `cashflow_data`.
- **Input/output:**  
  - Input ← `alphavantage - CashFlow`
  - Output → `Merge5`

#### Clean Current Price
- **Type and role:** Code node; extracts `Global Quote` current price.
- **Configuration choices:** Reads `"05. price"` from Alpha Vantage quote payload.
- **Input/output:**  
  - Input ← `alphavantage - Current Price`
  - Output → `Merge5`
- **Edge cases/failures:** If Alpha Vantage returns a note/error object, `current_price` becomes null.

#### Clean Current Price1
- **Type and role:** Same as above; used for the second quote refresh path.
- **Input/output:**  
  - Input ← `alphavantage - Current Price Second`
  - Output → `Update row in sheet1`

#### Merge3
- **Type and role:** Merge node; combines profile and income by `stock`.
- **Input/output:**  
  - Inputs ← `Clean Profile`, `Clean Income statement`
  - Output → `Merge2`

#### Merge2
- **Type and role:** Merge node; combines balance with profile+income.
- **Input/output:**  
  - Inputs ← `Clean balance sheet`, `Merge3`
  - Output → `Merge4`

#### Merge5
- **Type and role:** Merge node; combines cashflow and current price.
- **Input/output:**  
  - Inputs ← `Clean Ccashflow`, `Clean Current Price`
  - Output → `Merge4`

#### Merge4
- **Type and role:** Merge node; combines the two merged financial branches.
- **Input/output:**  
  - Inputs ← `Merge2`, `Merge5`
  - Output → `Clean Read Financial`

#### Clean Read Financial
- **Type and role:** Core code node; computes normalized financial dataset.
- **Configuration choices:**  
  - Safe numeric parsing
  - Revenue, net income, margin, debt, cash extraction
  - Revenue growth YoY from 8 quarters
  - Real FCF from cash flow data
  - Piotroski F-Score 0–9
  - `graham_number`
  - `dcf_anchor`
  - `sector_median_pe`
  - `net_debt_latest`, `net_cash_flag`
- **Key expressions/variables used:** Reads `overview_data`, `income_data`, `balance_data`, `cashflow_data`, `current_price`.
- **Input/output:**  
  - Input ← `Merge4`
  - Output → `If2`
- **Edge cases/failures:** Missing annual or quarterly reports reduce signal quality; code is defensive but may output null-heavy payloads. Financial sector handling is partly prompt-driven later rather than here.

#### If2
- **Type and role:** If node; decides update vs insert using `cacheHit`.
- **Configuration choices:** Tests `{{$json.cacheHit}}`.
- **Input/output:**  
  - Input ← `Clean Read Financial`
  - True → `Update row in sheet`
  - False → `Insert Row`
- **Edge cases/failures:** `cacheHit` is not produced directly by `Clean Read Financial`, so it depends on merged/retained data context; this works only if upstream item state preserves it correctly.

#### Update row in sheet
- **Type and role:** Google Sheets update; updates existing financial cache row by `stock`.
- **Configuration choices:** Maps many computed fields including F-Score and margins.
- **Key expressions/variables used:** Mostly `$('Clean Read Financial').item.json...`
- **Input/output:**  
  - Input ← `If2`
  - Output → `Merge New Financial`
- **Edge cases/failures:** Matching on `stock` assumes uniqueness; duplicate stock rows in cache could update unpredictably.

#### Insert Row
- **Type and role:** Google Sheets append; inserts new financial cache row.
- **Configuration choices:** Writes computed fundamentals for first-time stocks.
- **Input/output:**  
  - Input ← `If2`
  - Output → `Merge New Financial`

#### Update row in sheet1
- **Type and role:** Google Sheets update; refreshes current price in cache before reusing old financials.
- **Configuration choices:** Only updates `stock`, `last_updated`, and `current_price`.
- **Input/output:**  
  - Input ← `Clean Current Price1`
  - Output → `Get row(s) in sheet`
- **Edge cases/failures:** This path effectively keeps cached fundamentals but refreshes quote.

#### Get row(s) in sheet
- **Type and role:** Google Sheets lookup; fetches cached row after current price refresh.
- **Configuration choices:** Filters by ticker from `loop_over_tickers`.
- **Input/output:**  
  - Input ← `Update row in sheet1`
  - Output → `Clean Old Financial`

#### Clean Old Financial
- **Type and role:** Code node; normalizes cached-sheet values into typed fields.
- **Configuration choices:** Converts strings to numbers/booleans where possible and labels cache source as `fresh`.
- **Input/output:**  
  - Input ← `Get row(s) in sheet`
  - Output → `Merge New Financial`
- **Edge cases/failures:** Array-like fields such as `revenue_last_4q` remain as stored sheet values and may not be true arrays unless sheet storage format is consistent.

#### Merge New Financial
- **Type and role:** Pass-through code node (`return $input.all()`); unifies refreshed and cached branches.
- **Input/output:**  
  - Inputs ← `Update row in sheet`, `Insert Row`, `Clean Old Financial`
  - Output → `Merge1`
- **Edge cases/failures:** If multiple branches unexpectedly emit for the same stock, downstream merges may receive duplicate financial records.

---

## 2.3 News Retrieval and Normalization

**Overview:**  
This block collects Seeking Alpha XML feeds for each ticker, ensures valid XML exists, parses it into JSON, extracts recent news items, and compacts them into a prompt-ready text context.

**Nodes Involved:**  
- Seekingalpha Articles
- Clean the news
- If1
- XML
- Edit Fields2
- return the news
- Final version of news

### Node Details

#### Seekingalpha Articles
- **Type and role:** HTTP Request; fetches Seeking Alpha combined XML feed for the ticker.
- **Configuration choices:** Returns response as text into `sa_xml`.
- **Key expressions/variables used:** `https://seekingalpha.com/api/sa/combined/{{ $json.stock }}.xml`
- **Input/output:**  
  - Input ← `loop_over_tickers`
  - Output → `Clean the news`
- **Edge cases/failures:** Invalid ticker, anti-bot restrictions, XML structure changes, empty body.

#### Clean the news
- **Type and role:** Code node; standardizes response and extracts XML text.
- **Configuration choices:**  
  - Checks `sa_xml`, `body`, `data`, `responseBody`, `text`
  - Attempts binary decode if needed
  - Outputs `stock`, `sa_xml`, `sa_ok`
- **Input/output:**  
  - Input ← `Seekingalpha Articles`
  - Output → `If1`

#### If1
- **Type and role:** If node; checks that `sa_xml` is not empty.
- **Configuration choices:** `sa_xml` not empty.
- **Input/output:**  
  - Input ← `Clean the news`
  - True → `XML`
  - False → `return the news`

#### XML
- **Type and role:** XML node; parses XML string to JSON under `sa_xml`.
- **Configuration choices:** `dataPropertyName: sa_xml`
- **Input/output:**  
  - Input ← `If1`
  - Output → `Edit Fields2`
- **Edge cases/failures:** XML parsing errors if feed format changes or response contains HTML/error page.

#### Edit Fields2
- **Type and role:** Set node; restructures parsed news object.
- **Configuration choices:** Sets:
  - `stock` from `Clean the news`
  - `rss` from parsed `$json.rss`
- **Input/output:**  
  - Input ← `XML`
  - Output → `Final version of news`

#### return the news
- **Type and role:** Code node; fallback when no usable XML exists.
- **Configuration choices:** Emits empty news payload with error note.
- **Input/output:**  
  - Input ← `If1` false branch
  - Output → `Final version of news`

#### Final version of news
- **Type and role:** Code node; filters, deduplicates, windows, and compacts news.
- **Configuration choices:**  
  - 96-hour recent window
  - Fallback to latest 3 items if no recent news
  - HTML cleaning
  - Dedup by link
  - Outputs `seekingAlphaNewsText`, `seekingAlphaCount`, `seekingAlphaNews`, latest date, and recency flags
- **Input/output:**  
  - Inputs ← `Edit Fields2`, `return the news`
  - Output → `Merge1`
- **Edge cases/failures:** RSS schema changes can break item extraction. If all items are malformed or missing dates, result may be empty.

---

## 2.4 Dual-LLM Valuation Round

**Overview:**  
This block merges financial and news context, sends them in parallel to ChatGPT and Gemini using highly constrained prompts, and then parses their outputs into normalized JSON payloads for comparison.

**Nodes Involved:**  
- Merge1
- First round ChatGPT
- First round Gemini
- Clean Up ChatGPT
- Clean up Gemini
- Merge

### Node Details

#### Merge1
- **Type and role:** Merge node; combines financial dataset and news dataset by `stock`.
- **Configuration choices:** `keepEverything`, match on `stock`.
- **Input/output:**  
  - Inputs ← `Merge New Financial`, `Final version of news`
  - Output → `First round ChatGPT`, `First round Gemini`
- **Edge cases/failures:** If either branch emits duplicate rows per stock, merged outputs may multiply.

#### First round ChatGPT
- **Type and role:** OpenAI LangChain node; first independent valuation model.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - Output text format: `json_object`
  - Prompt enforces:
    - use only supplied values
    - classify phase
    - compute base/bear/bull targets
    - confidence score
    - rationale with structured sections
- **Key expressions/variables used:** extensive `{{ $json.* }}` including financials and `seekingAlphaNewsText`.
- **Input/output:**  
  - Input ← `Merge1`
  - Output → `Clean Up ChatGPT`
- **Version-specific requirements:** `@n8n/n8n-nodes-langchain.openAi` v2.1, OpenAI credentials.
- **Edge cases/failures:** Non-JSON responses despite prompt, model truncation, cost, rate limits, malformed null/array rendering in prompt.

#### First round Gemini
- **Type and role:** Google Gemini LangChain node; parallel independent valuation model.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-pro`
  - `jsonOutput: true`
  - Prompt largely mirrors ChatGPT prompt
- **Input/output:**  
  - Input ← `Merge1`
  - Output → `Clean up Gemini`
- **Version-specific requirements:** `@n8n/n8n-nodes-langchain.googleGemini` v1.1, Google Gemini credentials.
- **Edge cases/failures:** JSON formatting inconsistencies, quota/auth failures.

#### Clean Up ChatGPT
- **Type and role:** Code node; parses ChatGPT output and reattaches financial context.
- **Configuration choices:**  
  - Reads `output[0].content[0].text`
  - Tries JSON.parse
  - Returns `chatgpt_result` plus copied financial fields from `Merge1`
- **Input/output:**  
  - Input ← `First round ChatGPT`
  - Output → `Merge`
- **Edge cases/failures:** If OpenAI response schema changes, parse path fails and raw object may pass through.

#### Clean up Gemini
- **Type and role:** Code node; parses Gemini output and reattaches financial context.
- **Configuration choices:**  
  - Removes markdown code fences if needed
  - Parses `output` or `content.parts[0].text`
  - Returns `gemini_result` plus copied financial fields
- **Input/output:**  
  - Input ← `First round Gemini`
  - Output → `Merge`

#### Merge
- **Type and role:** Merge node; combines first-round model outputs by `stock`.
- **Configuration choices:** `keepEverything`, match on `stock`.
- **Input/output:**  
  - Inputs ← `Clean Up ChatGPT`, `Clean up Gemini`
  - Output → `Clean Up Results from First Round`

---

## 2.5 Disagreement Detection and Tiebreaker Resolution

**Overview:**  
This block determines whether the two first-round model outputs are close enough to accept. If not, it triggers a structured tiebreaker with ChatGPT forced into the bull case and Gemini forced into the bear case, then synthesizes a final tiebreaker result.

**Nodes Involved:**  
- Clean Up Results from First Round
- Needs Tiebreaker?
- Tide Breaker - Bull
- Tide Breaker - Bear
- Clean up Chatgpt 2
- Clean up Gemini 2
- Merge6
- Code in JavaScript3
- Code in JavaScript
- Merge7

### Node Details

#### Clean Up Results from First Round
- **Type and role:** Code node; compares first-round outputs.
- **Configuration choices:**  
  - Extracts verdict from rationale prefix
  - Computes `gap_pct = abs(pt1 - pt2) / current_price * 100`
  - Flags `needs_tiebreaker` if verdicts differ or `gap_pct > 25`
- **Key expressions/variables used:** pulls source data from `$('Merge').first().json`
- **Input/output:**  
  - Input ← `Merge`
  - Output → `Needs Tiebreaker?`
- **Edge cases/failures:** If rationale does not start with BUY/SELL, verdict defaults to HOLD. Description says >20% threshold, but actual code uses >25%.

#### Needs Tiebreaker?
- **Type and role:** If node; routes either to tiebreaker or straight consensus formatting.
- **Configuration choices:** Tests `{{$json.needs_tiebreaker}} == true`.
- **Input/output:**  
  - Input ← `Clean Up Results from First Round`
  - True → `Tide Breaker - Bull`, `Tide Breaker - Bear`
  - False → `Code in JavaScript`

#### Tide Breaker - Bull
- **Type and role:** OpenAI LangChain node; forced bull-case valuation.
- **Configuration choices:**  
  - Model `gpt-4o`
  - JSON object output
  - Prompt explicitly instructs model to maximize justifiable opportunity value
- **Input/output:**  
  - Input ← `Needs Tiebreaker?` true
  - Output → `Clean up Chatgpt 2`
- **Edge cases/failures:** Same as first-round OpenAI node; also may overfit prompt bias.

#### Tide Breaker - Bear
- **Type and role:** Gemini node; forced bear-case valuation.
- **Configuration choices:**  
  - Gemini 2.5 Pro
  - JSON output
  - Prompt explicitly instructs model to minimize justifiable risk-adjusted value
- **Input/output:**  
  - Input ← `Needs Tiebreaker?` true
  - Output → `Clean up Gemini 2`

#### Clean up Chatgpt 2
- **Type and role:** Code node; parses tiebreaker bull output.
- **Configuration choices:** Adds `model: CHATGPT_BULL`, carries `current_price`, `gap_pct`, and prior verdict metadata.
- **Input/output:**  
  - Input ← `Tide Breaker - Bull`
  - Output → `Merge6`

#### Clean up Gemini 2
- **Type and role:** Code node; parses tiebreaker bear output.
- **Configuration choices:** Adds `model: GEMINI_BEAR`, carries same metadata.
- **Input/output:**  
  - Input ← `Tide Breaker - Bear`
  - Output → `Merge6`

#### Merge6
- **Type and role:** Merge node; combines tiebreaker bull and bear outputs by stock.
- **Input/output:**  
  - Inputs ← `Clean up Chatgpt 2`, `Clean up Gemini 2`
  - Output → `Code in JavaScript3`

#### Code in JavaScript3
- **Type and role:** Code node; synthesizes final tiebreaker result.
- **Configuration choices:**  
  - Base target = average of bull and bear base targets
  - `pt_bull` from bull output
  - `pt_bear` from bear output
  - Confidence = average
  - Final verdict derived from base target upside/downside vs current price
  - Model labeled `TIEBREAKER`
- **Input/output:**  
  - Input ← `Merge6`
  - Output → `Merge7`
- **Edge cases/failures:** If one side is missing, outputs `skip_row: true` error row.

#### Code in JavaScript
- **Type and role:** Code node; consensus-path formatter for first-round results.
- **Configuration choices:**  
  - Validates rows
  - Keeps highest-confidence row per model
  - Adds `resolution_method` = `CONSENSUS` or `TIEBREAKER`
  - Adds `conviction` from `gap_pct`
  - Emits one row per model
- **Input/output:**  
  - Input ← `Needs Tiebreaker?` false
  - Output → `Merge7`
- **Edge cases/failures:** If neither model yields valid numeric targets, outputs an error row with `skip_row: true`.

#### Merge7
- **Type and role:** Merge node; unifies consensus and tiebreaker paths.
- **Input/output:**  
  - Inputs ← `Code in JavaScript3`, `Code in JavaScript`
  - Output → `If`

---

## 2.6 Output Filtering, History Enrichment, and Alert Preparation

**Overview:**  
This block filters out invalid rows, checks historical verdicts in the results sheet, detects thesis reversals, and prepares alert payloads.

**Nodes Involved:**  
- If
- Get Previous Verdict
- Thesis Reversal Enricher
- Alert Filter
- If Alert?

### Node Details

#### If
- **Type and role:** If node; drops rows marked `skip_row = true`.
- **Configuration choices:** Requires `skip_row` to be false.
- **Input/output:**  
  - Input ← `Merge7`
  - True → `Get Previous Verdict`
  - False → `loop_over_tickers`
- **Edge cases/failures:** Error rows are silently skipped and batch continues.

#### Get Previous Verdict
- **Type and role:** Google Sheets lookup; fetches prior sentiment rows for the same stock.
- **Configuration choices:** Reads all matching rows from results spreadsheet `Sentiments of my stocks`.
- **Input/output:**  
  - Input ← `If` true
  - Output → `Thesis Reversal Enricher`
- **Edge cases/failures:** If sheet grows large, per-stock scan may become slower.

#### Thesis Reversal Enricher
- **Type and role:** Code node; compares latest prior verdict with current verdict.
- **Configuration choices:**  
  - Extracts verdict from prior/current rationale
  - Sorts previous rows by date descending
  - Computes `thesis_reversal`
  - Builds `thesis_reversal_msg`
- **Key expressions/variables used:** Uses `$('If').first().json` for current analysis row.
- **Input/output:**  
  - Input ← `Get Previous Verdict`
  - Output → `Alert Filter`
- **Edge cases/failures:** Date sorting is lexical; works if dates are ISO `YYYY-MM-DD`, which they are.

#### Alert Filter
- **Type and role:** Code node; determines whether to trigger action alert.
- **Configuration choices:**  
  - Verdict from rationale prefix
  - Alert only if verdict is BUY or SELL and confidence >= 50
  - Builds Markdown alert text
- **Input/output:**  
  - Input ← `Thesis Reversal Enricher`
  - Output → `If Alert?`
- **Edge cases/failures:** If rationale structure changes, verdict extraction may misclassify HOLD.

#### If Alert?
- **Type and role:** If node; routes alert-worthy rows into technical filter path, otherwise writes directly to sheet.
- **Configuration choices:** Tests `{{$json.should_alert}}`.
- **Input/output:**  
  - Input ← `Alert Filter`
  - True → `Get 50MA`
  - False → `write_sentiment_to_sheets`

---

## 2.7 Technical Entry Filters and Telegram Delivery

**Overview:**  
This block enriches alert-worthy results with moving averages, RSI, and earnings date context. It then sends either an immediate Telegram alert for stronger setups or a watchlist-style message for names below the 50-day average, and separately notifies on thesis reversals.

**Nodes Involved:**  
- Get 50MA
- Get 200MA
- MA Check
- Get RSI
- Get Earnings
- Entry Signal Enricher
- Above 50MA?
- Telegram Alert
- Watchlist Telegram
- Is Thesis Reversal?
- Thesis Reversal Alert

### Node Details

#### Get 50MA
- **Type and role:** HTTP Request; Alpha Vantage SMA 50-day.
- **Configuration choices:** `function=SMA`, `time_period=50`.
- **Input/output:**  
  - Input ← `If Alert?` true
  - Output → `Get 200MA`

#### Get 200MA
- **Type and role:** HTTP Request; Alpha Vantage SMA 200-day.
- **Configuration choices:** `time_period=200`.
- **Input/output:**  
  - Input ← `Get 50MA`
  - Output → `MA Check`

#### MA Check
- **Type and role:** Code node; evaluates price relative to moving averages.
- **Configuration choices:**  
  - Reads alert data from `Alert Filter`
  - Calculates `ma_50`, `ma_200`
  - Sets `ma_check_passed`, `ma_signal`, `trend_tier`
  - Fails open if MA unavailable
- **Input/output:**  
  - Input ← `Get 200MA`
  - Output → `Get RSI`
- **Edge cases/failures:** Relies on `$('Get 200MA').first()` and `$('Alert Filter').first()`, so parallel execution behavior should remain sequential per stock.

#### Get RSI
- **Type and role:** HTTP Request; fetches 14-day RSI.
- **Input/output:**  
  - Input ← `MA Check`
  - Output → `Get Earnings`

#### Get Earnings
- **Type and role:** HTTP Request; fetches earnings schedule/history.
- **Input/output:**  
  - Input ← `Get RSI`
  - Output → `Entry Signal Enricher`

#### Entry Signal Enricher
- **Type and role:** Code node; adds RSI and next earnings context.
- **Configuration choices:**  
  - Computes `rsi_signal`
  - Finds next future earnings date
  - Adds `earnings_warning` if within 14 days
- **Input/output:**  
  - Input ← `Get Earnings`
  - Output → `Above 50MA?`

#### Above 50MA?
- **Type and role:** If node; stronger-vs-watchlist routing.
- **Configuration choices:** Tests `ma_check_passed`.
- **Input/output:**  
  - Input ← `Entry Signal Enricher`
  - True → `Telegram Alert`
  - False → `Watchlist Telegram`

#### Telegram Alert
- **Type and role:** Telegram node; sends main actionable alert.
- **Configuration choices:** Sends `alert_message` with Markdown parse mode.
- **Input/output:**  
  - Input ← `Above 50MA?` true
  - Output → `Is Thesis Reversal?`
- **Edge cases/failures:** Markdown escaping issues may occur if rationale includes problematic characters.

#### Watchlist Telegram
- **Type and role:** Telegram node; sends softer watchlist message when below 50MA.
- **Configuration choices:** Custom text includes target, confidence, RSI, earnings warning, trend tier.
- **Input/output:**  
  - Input ← `Above 50MA?` false
  - Output → `write_sentiment_to_sheets`
- **Edge cases/failures:** Uses multiple inline expressions; if nulls are rendered unexpectedly, formatting can look uneven.

#### Is Thesis Reversal?
- **Type and role:** If node; checks `thesis_reversal`.
- **Configuration choices:** boolean equals true.
- **Input/output:**  
  - Input ← `Telegram Alert`
  - True → `Thesis Reversal Alert`
  - False → `write_sentiment_to_sheets`

#### Thesis Reversal Alert
- **Type and role:** Telegram node; sends verdict-change notification.
- **Configuration choices:**  
  - Text from `thesis_reversal_msg`
  - Chat ID pulled from `Send a text message` node parameters
- **Input/output:**  
  - Input ← `Is Thesis Reversal?`
  - Output → `write_sentiment_to_sheets`
- **Edge cases/failures:** Cross-node parameter reference can break if the summary Telegram node is edited or removed.

---

## 2.8 Persistence and Reporting

**Overview:**  
This block appends the final analysis rows to the results Google Sheet and then resumes the loop. It is the durable output store for the workflow.

**Nodes Involved:**  
- write_sentiment_to_sheets

### Node Details

#### write_sentiment_to_sheets
- **Type and role:** Google Sheets append; stores valuation outputs.
- **Configuration choices:** Appends to spreadsheet `Sentiments of my stocks`, sheet `Sheet1`.
- **Mapped fields include:**  
  `date`, `stock`, `model`, `current_price`, `pt_bear`, `pt_base`, `pt_bull`, `f_score`, `confidence`, `gap_pct`, `resolution_method`, `rationale`, `conviction`, `verdict_chatgpt`, `verdict_gemini`, `ma_50`, `ma_200`, `ma_signal`, `rsi_14`, `rsi_signal`, `trend_tier`, `next_earnings_date`, `previous_verdict`, `thesis_reversal`
- **Input and output connections:**  
  - Inputs ← `If Alert?` false, `Watchlist Telegram`, `Is Thesis Reversal?` false, `Thesis Reversal Alert`
  - Output → `Wait`
- **Version-specific requirements:** v4.6.
- **Edge cases/failures:** Append-only means duplicates are expected over time; historical log is intentional.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled workflow entry point |  | Read_tickers_from_Sheet | # 1. Daily Trigger and Stock Ticker Retrieval<br>- **Schedule Trigger:** This workflow is set to run automatically every three day at 4:00 PM (Asia/Jerusalem time). This ensures that the script runs just before the markets open and you get a daily update on the sentiment of the stocks you are tracking.<br><br>- **Read_tickers_from_Sheet:** This node connects to a Google Sheet named "Stock Sentiment" and reads the list of stock tickers from the "stocks" sheet. This is the source of the stocks that the workflow will analyze. I have make every three days to avoid extra fees from Alphavintage since, and you can modify it if you need.<br><br>- **loop_over_tickers:** This node takes the list of tickers from the Google Sheet and processes them one by one. This allows the workflow to perform the same set of actions for each stock ticker individually. |
| Read_tickers_from_Sheet | googleSheets | Reads stock watchlist from Google Sheets | Schedule Trigger | loop_over_tickers | # 1. Daily Trigger and Stock Ticker Retrieval<br>- **Schedule Trigger:** This workflow is set to run automatically every three day at 4:00 PM (Asia/Jerusalem time). This ensures that the script runs just before the markets open and you get a daily update on the sentiment of the stocks you are tracking.<br><br>- **Read_tickers_from_Sheet:** This node connects to a Google Sheet named "Stock Sentiment" and reads the list of stock tickers from the "stocks" sheet. This is the source of the stocks that the workflow will analyze. I have make every three days to avoid extra fees from Alphavintage since, and you can modify it if you need.<br><br>- **loop_over_tickers:** This node takes the list of tickers from the Google Sheet and processes them one by one. This allows the workflow to perform the same set of actions for each stock ticker individually. |
| loop_over_tickers | splitInBatches | Processes tickers sequentially and exposes done branch | Read_tickers_from_Sheet, If, Wait | Cache Lookup, Seekingalpha Articles, Build Summary | # 1. Daily Trigger and Stock Ticker Retrieval<br>- **Schedule Trigger:** This workflow is set to run automatically every three day at 4:00 PM (Asia/Jerusalem time). This ensures that the script runs just before the markets open and you get a daily update on the sentiment of the stocks you are tracking.<br><br>- **Read_tickers_from_Sheet:** This node connects to a Google Sheet named "Stock Sentiment" and reads the list of stock tickers from the "stocks" sheet. This is the source of the stocks that the workflow will analyze. I have make every three days to avoid extra fees from Alphavintage since, and you can modify it if you need.<br><br>- **loop_over_tickers:** This node takes the list of tickers from the Google Sheet and processes them one by one. This allows the workflow to perform the same set of actions for each stock ticker individually. |
| Cache Lookup | googleSheets | Looks up cached fundamentals by ticker | loop_over_tickers | Check the cache | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Check the cache | code | Decides fetch vs cache reuse | Cache Lookup | Is Cache Valid? | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Is Cache Valid? | if | Routes to fetch path when financial refresh is needed | Check the cache | Wait1, Wait2, Wait3, Wait4, Wait5, Wait6 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Wait1 | wait | Rate-limit delay before balance sheet call | Is Cache Valid? | alphavantage - Balance Sheet | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| alphavantage - Balance Sheet | httpRequest | Fetches balance sheet from Alpha Vantage | Wait1 | Clean balance sheet | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean balance sheet | code | Wraps balance sheet JSON with stock key | alphavantage - Balance Sheet | Merge2 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Wait2 | wait | Rate-limit delay before overview call | Is Cache Valid? | alphavantage - Profile | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| alphavantage - Profile | httpRequest | Fetches company overview from Alpha Vantage | Wait2 | Clean Profile | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Profile | code | Wraps overview JSON with stock key | alphavantage - Profile | Merge3 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Wait3 | wait | Rate-limit delay before income statement call | Is Cache Valid? | alphavantage - Income Statement | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| alphavantage - Income Statement | httpRequest | Fetches income statement from Alpha Vantage | Wait3 | Clean Income statement | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Income statement | code | Wraps income statement JSON with stock key | alphavantage - Income Statement | Merge3 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Wait4 | wait | Rate-limit delay before cash flow call | Is Cache Valid? | alphavantage - CashFlow | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| alphavantage - CashFlow | httpRequest | Fetches cash flow from Alpha Vantage | Wait4 | Clean Ccashflow | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Ccashflow | code | Wraps cash flow JSON with stock key | alphavantage - CashFlow | Merge5 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Wait5 | wait | Rate-limit delay before first current price call | Is Cache Valid? | alphavantage - Current Price | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| alphavantage - Current Price | httpRequest | Fetches current quote from Alpha Vantage | Wait5 | Clean Current Price | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Current Price | code | Extracts price from GLOBAL_QUOTE response | alphavantage - Current Price | Merge5 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Wait6 | wait | Rate-limit delay before second current price call | Is Cache Valid? | alphavantage - Current Price Second | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| alphavantage - Current Price Second | httpRequest | Refreshes current quote for cached branch | Wait6 | Clean Current Price1 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Current Price1 | code | Extracts refreshed quote for cached branch | alphavantage - Current Price Second | Update row in sheet1 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Update row in sheet1 | googleSheets | Updates cached current price before reading cached fundamentals | Clean Current Price1 | Get row(s) in sheet | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Get row(s) in sheet | googleSheets | Reads cached financial row after price refresh | Update row in sheet1 | Clean Old Financial | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Old Financial | code | Normalizes cached fundamentals into typed fields | Get row(s) in sheet | Merge New Financial | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Merge3 | merge | Merges overview and income statement | Clean Profile, Clean Income statement | Merge2 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Merge2 | merge | Merges balance with profile+income | Clean balance sheet, Merge3 | Merge4 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Merge5 | merge | Merges cash flow with current price | Clean Ccashflow, Clean Current Price | Merge4 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Merge4 | merge | Combines all refreshed financial data streams | Merge2, Merge5 | Clean Read Financial | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Clean Read Financial | code | Computes normalized fundamentals, F-Score, Graham, DCF anchor | Merge4 | If2 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| If2 | if | Chooses cache update vs insert | Clean Read Financial | Update row in sheet, Insert Row | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Update row in sheet | googleSheets | Updates existing financial cache record | If2 | Merge New Financial | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Insert Row | googleSheets | Inserts new financial cache record | If2 | Merge New Financial | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Merge New Financial | code | Unifies refreshed and cached financial branches | Update row in sheet, Insert Row, Clean Old Financial | Merge1 | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Seekingalpha Articles | httpRequest | Downloads Seeking Alpha XML feed | loop_over_tickers | Clean the news | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| Clean the news | code | Normalizes XML/news payload | Seekingalpha Articles | If1 | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| If1 | if | Checks whether Seeking Alpha XML exists | Clean the news | XML, return the news | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| XML | xml | Parses Seeking Alpha XML into JSON | If1 | Edit Fields2 | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| Edit Fields2 | set | Reshapes parsed RSS payload | XML | Final version of news | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| return the news | code | Emits empty fallback news payload | If1 | Final version of news | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| Final version of news | code | Filters and compacts recent news into prompt text | Edit Fields2, return the news | Merge1 | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| Merge1 | merge | Combines financials and news by stock | Merge New Financial, Final version of news | First round ChatGPT, First round Gemini | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| First round ChatGPT | @n8n/n8n-nodes-langchain.openAi | First independent ChatGPT valuation pass | Merge1 | Clean Up ChatGPT | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| First round Gemini | @n8n/n8n-nodes-langchain.googleGemini | First independent Gemini valuation pass | Merge1 | Clean up Gemini | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| Clean Up ChatGPT | code | Parses ChatGPT output and reattaches financial context | First round ChatGPT | Merge | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| Clean up Gemini | code | Parses Gemini output and reattaches financial context | First round Gemini | Merge | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| Merge | merge | Combines first-round model results | Clean Up ChatGPT, Clean up Gemini | Clean Up Results from First Round | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| Clean Up Results from First Round | code | Compares model verdicts and target gap | Merge | Needs Tiebreaker? | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Needs Tiebreaker? | if | Routes to consensus path or tiebreaker path | Clean Up Results from First Round | Tide Breaker - Bull, Tide Breaker - Bear, Code in JavaScript | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Tide Breaker - Bull | @n8n/n8n-nodes-langchain.openAi | Bull-biased tiebreaker valuation | Needs Tiebreaker? | Clean up Chatgpt 2 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Tide Breaker - Bear | @n8n/n8n-nodes-langchain.googleGemini | Bear-biased tiebreaker valuation | Needs Tiebreaker? | Clean up Gemini 2 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Clean up Chatgpt 2 | code | Parses bull tiebreaker output | Tide Breaker - Bull | Merge6 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Clean up Gemini 2 | code | Parses bear tiebreaker output | Tide Breaker - Bear | Merge6 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Merge6 | merge | Combines bull and bear tiebreaker outputs | Clean up Chatgpt 2, Clean up Gemini 2 | Code in JavaScript3 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Code in JavaScript3 | code | Synthesizes final tiebreaker output | Merge6 | Merge7 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Code in JavaScript | code | Formats consensus-path analyst results | Needs Tiebreaker? | Merge7 | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Merge7 | merge | Unifies consensus and tiebreaker outputs | Code in JavaScript3, Code in JavaScript | If | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| If | if | Filters out invalid/error rows | Merge7 | Get Previous Verdict, loop_over_tickers | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Get Previous Verdict | googleSheets | Reads historical rows for current stock | If | Thesis Reversal Enricher | # 4.Strong Buy Alerts |
| Thesis Reversal Enricher | code | Detects verdict changes vs prior entry | Get Previous Verdict | Alert Filter | # 4.Strong Buy Alerts |
| Alert Filter | code | Builds alert payload and filters by verdict/confidence | Thesis Reversal Enricher | If Alert? | # 4.Strong Buy Alerts |
| If Alert? | if | Routes actionable vs non-actionable outputs | Alert Filter | Get 50MA, write_sentiment_to_sheets | # 4.Strong Buy Alerts |
| Get 50MA | httpRequest | Fetches 50-day moving average | If Alert? | Get 200MA | # 4.Strong Buy Alerts |
| Get 200MA | httpRequest | Fetches 200-day moving average | Get 50MA | MA Check | # 4.Strong Buy Alerts |
| MA Check | code | Computes MA signals and trend tier | Get 200MA | Get RSI | # 4.Strong Buy Alerts |
| Get RSI | httpRequest | Fetches 14-day RSI | MA Check | Get Earnings | # 4.Strong Buy Alerts |
| Get Earnings | httpRequest | Fetches earnings data/schedule | Get RSI | Entry Signal Enricher | # 4.Strong Buy Alerts |
| Entry Signal Enricher | code | Adds RSI and earnings warnings | Get Earnings | Above 50MA? | # 4.Strong Buy Alerts |
| Above 50MA? | if | Routes to direct alert or watchlist alert | Entry Signal Enricher | Telegram Alert, Watchlist Telegram | # 4.Strong Buy Alerts |
| Telegram Alert | telegram | Sends primary Telegram buy/sell alert | Above 50MA? | Is Thesis Reversal? | # 4.Strong Buy Alerts |
| Watchlist Telegram | telegram | Sends weaker technical setup watchlist alert | Above 50MA? | write_sentiment_to_sheets | # 4.Strong Buy Alerts |
| Is Thesis Reversal? | if | Checks whether current verdict differs from previous one | Telegram Alert | Thesis Reversal Alert, write_sentiment_to_sheets | # 4.Strong Buy Alerts |
| Thesis Reversal Alert | telegram | Sends thesis reversal Telegram message | Is Thesis Reversal? | write_sentiment_to_sheets | # 4.Strong Buy Alerts |
| write_sentiment_to_sheets | googleSheets | Appends final valuation rows to results sheet | If Alert?, Watchlist Telegram, Is Thesis Reversal?, Thesis Reversal Alert | Wait | # 5. Storing the Results<br><br><br>- **write_sentiment_to_sheets:** Once a valid sentiment analysis result is obtained and formatted, this node appends the data to "Sheet1" of the "Stock Sentiment" Google Sheet. It records the current date, the stock ticker, the sentiment score, and the rationale provided by the AI. After this step, the workflow loops back to process the next ticker from the initial list. |
| Sticky Note | stickyNote | Documentation note |  |  | # Workflow Overview <br>**This workflow automates the process of analyzing the sentiment of stock market news.**<br><br>- retrieves a list of stock tickers from a Google Sheet <br>- fetchs recent news articles for each ticker<br>- uses a 2 large language model to perform sentiment analysis on the articles<br>- records the sentiment scores and rationale back into a Google Sheet. |
| Sticky Note1 | stickyNote | Documentation note |  |  | # 1. Daily Trigger and Stock Ticker Retrieval<br>- **Schedule Trigger:** This workflow is set to run automatically every three day at 4:00 PM (Asia/Jerusalem time). This ensures that the script runs just before the markets open and you get a daily update on the sentiment of the stocks you are tracking.<br><br>- **Read_tickers_from_Sheet:** This node connects to a Google Sheet named "Stock Sentiment" and reads the list of stock tickers from the "stocks" sheet. This is the source of the stocks that the workflow will analyze. I have make every three days to avoid extra fees from Alphavintage since, and you can modify it if you need.<br><br>- **loop_over_tickers:** This node takes the list of tickers from the Google Sheet and processes them one by one. This allows the workflow to perform the same set of actions for each stock ticker individually. |
| Sticky Note2 | stickyNote | Documentation note |  |  | # 2. Financial and News Data Retrieval & Validation<br>2.1 Financial Data Fetching (Alpha Vantage APIs)<br>Fetch Financial Data (Alpha Vantage):<br>For each ticker, the workflow sends multiple HTTP requests to Alpha Vantage to retrieve structured financial data, including:<br>Company overview (EPS, BVPS, margins, shares, revenue).<br>Income statement (quarterly revenue and net income).<br>Balance sheet (debt and cash position).<br>Cash flow statement (operating cash flow and capital expenditures for FCF calculation).<br>Current stock price.<br>Data Normalization and Processing:<br>The responses from the APIs are processed using code nodes to:<br>Extract required financial fields.<br>Calculate derived metrics such as revenue growth, gross margins, total debt, cash position, and free cash flow.<br>Standardize numeric values and prepare structured financial data for analysis.<br>Financial Cache Validation:<br>The workflow checks Google Sheets to determine whether financial data for the ticker already exists and whether it is still fresh (within a defined time window).<br>If the data is missing or outdated → financial APIs are called.<br>If the data is still valid → cached data is reused to avoid unnecessary API calls.<br>Update or Insert Financial Records:<br>Based on the cache result:<br>New records are inserted for first-time tickers.<br>Existing records are updated when refreshed data is retrieved. |
| Sticky Note3 | stickyNote | Documentation note |  |  | # 3. Sentiment Analysis with AI<br><br>- **AI Agent & Google Gemini Chat Model/ChatGPT:** This is the core of the sentiment analysis. The "AI Agent" node is configured with a detailed prompt that instructs the "Google Gemini Chat Model & ChatGPT" to act as a stock sentiment analyzer. The prompt specifies the input format (stock symbol, price targets,confidance, rational), and the desired JSON output format. The combined text of the news articles and the current stock ticker are passed to the model.l analyze all the articles at once. |
| Sticky Note4 | stickyNote | Documentation note |  |  | # 4. Output Formatting and Error Handling<br><br>- **format_output_as_json:** The output from the AI models is a raw string that includes a JSON object. This code node extracts the clean JSON from the string and prepares it for the next steps.<br><br>- **if_format_succesful:** This conditional node checks if the previous step of formatting the AI's output into a clean JSON was successful. If there was an error, it sends the workflow back to the "AI Agent" to try again. |
| Sticky Note5 | stickyNote | Documentation note |  |  | # 5. Storing the Results<br><br><br>- **write_sentiment_to_sheets:** Once a valid sentiment analysis result is obtained and formatted, this node appends the data to "Sheet1" of the "Stock Sentiment" Google Sheet. It records the current date, the stock ticker, the sentiment score, and the rationale provided by the AI. After this step, the workflow loops back to process the next ticker from the initial list. |
| Sticky Note6 | stickyNote | Documentation note |  |  | # 2. SeekingAlpha News Data Retrieval & Validation<br>Get Articles from Seeking Alpha:<br>For each ticker, the workflow sends an HTTP request to retrieve recent Seeking Alpha articles within a defined time window. These articles provide analyst-grade insights and market context.<br>Ticker Validation Check:<br>A conditional node verifies whether valid news results were returned and whether the ticker is recognized.<br>If no articles are found → the ticker may be invalid or have no recent coverage.<br>Handle Invalid Tickers:<br>If the ticker is invalid or returns no data, the workflow logs the ticker in the Google Sheet with an "Invalid Ticker" status. This enables tracking of symbols that cannot be processed.<br>News Aggregation:<br>For valid tickers, multiple articles are merged into a single structured text block.<br>This prepares consolidated news context for the AI model to analyze in one step |
| Sticky Note — Accuracy Tracker | stickyNote | Documentation note |  |  | ## IDEA 7: Prediction Accuracy Tracker<br><br>**Create a separate scheduled workflow (monthly) that:**<br><br>1. Read the Sentiments sheet for all predictions older than 30 days<br>2. For each stock+date+model row:<br>   - Fetch `alphavantage Current Price` for the stock<br>   - Compute: `return_pct = (actual_price_30d - current_price) / current_price * 100`<br>   - Determine `was_correct`: BUY verdict + return > 5% = correct<br>3. Append to a new **Accuracy_Tracking** Google Sheet with columns:<br>   `stock | predicted_on | verdict | pt_base | current_price_at_prediction | actual_price_30d | return_pct | was_correct | model | resolution_method`<br>4. Send weekly Telegram summary: *Model accuracy last 90 days: ChatGPT 68%, Gemini 71%, Tiebreaker 74%*<br><br>**This turns the system into a learning engine.**<br>Sheet ID for Sentiments: `1fbptcVE0mBjaIZJHkJzFBTdoVJVLgQOWmJXpCx40YyE` |
| Sticky Note7 | stickyNote | Documentation note |  |  | # 4.Strong Buy Alerts |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the trigger**
   1. Add a **Schedule Trigger** node.
   2. Configure it to run every 3 days at 16:00.
   3. Confirm the workflow/server timezone matches your intended market schedule.

2. **Create the watchlist input sheet**
   1. In Google Sheets, create a spreadsheet for tickers.
   2. Add a sheet with a `stock` column.
   3. Add a **Google Sheets** node named `Read_tickers_from_Sheet`.
   4. Authenticate with **Google Sheets OAuth2**.
   5. Configure it to read rows from the watchlist sheet.

3. **Add batch iteration**
   1. Add a **Split In Batches** node named `loop_over_tickers`.
   2. Connect `Read_tickers_from_Sheet` → `loop_over_tickers`.

4. **Build the financial cache lookup path**
   1. Create a cache spreadsheet with columns such as:  
      `stock, last_updated, eps_current, bvps_current, shares_outstanding, revenue_ttm, total_debt_ttm, gross_margin_ttm, operating_margin_ttm, fcf_ttm, revenue_last_4q, net_income_last_4q, gross_margin_last_4q, total_debt_last_4q, cash_last_4q, net_income_ttm, current_price, revenue_growth_yoy, f_score, f_score_data_ok, sector, net_debt_latest, net_cash_flag`
   2. Add `Cache Lookup` as a **Google Sheets** node filtering by `stock = {{$json.stock}}`.
   3. Enable first match and always output data.
   4. Add a **Code** node `Check the cache` using the same logic as the workflow:
      - Read cache row
      - Read optional `need_fresh_financial`
      - Output `cacheHit`, `shouldFetch`, `reason`
   5. Add an **If** node `Is Cache Valid?` checking `shouldFetch == true`.

5. **Add Alpha Vantage credential strategy**
   1. This workflow uses direct API-key URLs, not n8n credentials.
   2. Replace the hardcoded Alpha Vantage key with your own.
   3. Prefer moving the key into an environment variable or credential-safe expression.

6. **Create rate-limited Alpha Vantage fetch branch**
   1. Add six **Wait** nodes before API calls:
      - 15s before Balance Sheet
      - 20s before Profile
      - 30s before Income Statement
      - 25s before Cash Flow
      - 20s before Current Price
      - 20s before Current Price Second
   2. Connect the true output of `Is Cache Valid?` to all six wait nodes.

7. **Create Alpha Vantage request nodes**
   1. Add HTTP Request nodes:
      - `alphavantage - Balance Sheet` → `function=BALANCE_SHEET`
      - `alphavantage - Profile` → `function=OVERVIEW`
      - `alphavantage - Income Statement` → `function=INCOME_STATEMENT`
      - `alphavantage - CashFlow` → `function=CASH_FLOW`
      - `alphavantage - Current Price` → `function=GLOBAL_QUOTE`
      - `alphavantage - Current Price Second` → `function=GLOBAL_QUOTE`
   2. Each URL should use `{{ $('loop_over_tickers').item.json.stock }}`.
   3. Response format should be JSON except news XML later.

8. **Normalize financial API outputs**
   1. Add code nodes:
      - `Clean balance sheet`
      - `Clean Profile`
      - `Clean Income statement`
      - `Clean Ccashflow`
      - `Clean Current Price`
      - `Clean Current Price1`
   2. Each should standardize a `stock` field.
   3. `Clean Current Price` nodes should extract `["Global Quote"]["05. price"]`.

9. **Merge refreshed financial sources**
   1. Add `Merge3` to combine profile + income on `stock`.
   2. Add `Merge2` to combine balance + `Merge3`.
   3. Add `Merge5` to combine cash flow + current price.
   4. Add `Merge4` to combine `Merge2` + `Merge5`.

10. **Compute normalized financial dataset**
    1. Add `Clean Read Financial` as a **Code** node.
    2. Implement:
       - numeric parsing helper
       - quarterly revenue/net income extraction
       - gross margins
       - debt/cash calculations
       - `revenue_growth_yoy`
       - `fcf_ttm`
       - Piotroski `f_score`
       - `graham_number`
       - `dcf_anchor`
       - `sector_median_pe`
    3. Output a single normalized JSON object per stock.

11. **Upsert refreshed cache**
    1. Add an **If** node `If2` checking whether a cache row already exists.
    2. Add `Update row in sheet` Google Sheets node matching by `stock`.
    3. Add `Insert Row` Google Sheets append node for missing stocks.
    4. Map all normalized financial fields into the cache sheet.

12. **Build cached-financial reuse path**
    1. Connect `Clean Current Price1` → `Update row in sheet1` to refresh only current price.
    2. Then add `Get row(s) in sheet` to fetch the cache row.
    3. Add `Clean Old Financial` code node to cast strings to numbers/booleans and mark source as cache.

13. **Unify refreshed and cached financial branches**
    1. Add `Merge New Financial` as a code node returning all incoming items unchanged.
    2. Connect `Update row in sheet`, `Insert Row`, and `Clean Old Financial` into it.

14. **Build Seeking Alpha news path**
    1. Add HTTP Request node `Seekingalpha Articles`.
    2. URL: `https://seekingalpha.com/api/sa/combined/{{ $json.stock }}.xml`
    3. Set response format to **text**, output property `sa_xml`.

15. **Normalize and validate news**
    1. Add `Clean the news` code node to:
       - preserve `stock`
       - locate XML in several possible properties
       - decode binary if needed
    2. Add `If1` checking that `sa_xml` is not empty.

16. **Parse XML and shape news**
    1. Add `XML` node with `dataPropertyName = sa_xml`.
    2. Add `Edit Fields2` set node to output:
       - `stock`
       - `rss`
    3. Add `return the news` code node as fallback for empty XML.

17. **Create final news compactor**
    1. Add `Final version of news` code node.
    2. Implement:
       - 96-hour recent window
       - fallback latest 3 items if none recent
       - dedupe by link
       - cleaned snippets
       - `seekingAlphaNewsText`
       - `seekingAlphaCount`
       - `seekingAlphaLatestDate`

18. **Merge financials with news**
    1. Add `Merge1` matching on `stock`.
    2. Connect `Merge New Financial` and `Final version of news`.

19. **Configure OpenAI**
    1. Add OpenAI credentials in n8n.
    2. Add `First round ChatGPT` using the LangChain OpenAI node.
    3. Select model `gpt-4o`.
    4. Force JSON object output.
    5. Paste the prompt logic from the workflow:
       - strict use of provided fields only
       - valuation framework
       - bear/base/bull targets
       - confidence and rationale

20. **Configure Gemini**
    1. Add Google Gemini credentials.
    2. Add `First round Gemini` using Gemini 2.5 Pro.
    3. Enable JSON output.
    4. Use the mirrored institutional valuation prompt.

21. **Parse first-round model outputs**
    1. Add `Clean Up ChatGPT` code node to parse OpenAI response text JSON.
    2. Add `Clean up Gemini` code node to parse Gemini output/fenced JSON.
    3. Reattach original financial/news fields in both nodes.

22. **Merge first-round outputs**
    1. Add `Merge` node matching on `stock`.
    2. Connect both cleanup nodes to it.

23. **Evaluate disagreement**
    1. Add `Clean Up Results from First Round` code node.
    2. Compute:
       - `verdict_chatgpt`
       - `verdict_gemini`
       - `gap_pct`
       - `needs_tiebreaker`
    3. Note: the implemented code uses `gap_pct > 25`, not 20.

24. **Create tiebreaker router**
    1. Add `Needs Tiebreaker?` If node testing `needs_tiebreaker == true`.

25. **Configure bull/bear tiebreaker LLMs**
    1. Add `Tide Breaker - Bull` OpenAI node with prompt forcing optimistic but bounded valuation.
    2. Add `Tide Breaker - Bear` Gemini node with prompt forcing conservative downside framing.
    3. Keep JSON-only output for both.

26. **Parse tiebreaker outputs**
    1. Add `Clean up Chatgpt 2` code node to mark output model as `CHATGPT_BULL`.
    2. Add `Clean up Gemini 2` code node to mark output model as `GEMINI_BEAR`.

27. **Merge and synthesize tiebreaker**
    1. Add `Merge6` to combine the two tiebreaker outputs.
    2. Add `Code in JavaScript3` to produce a single final `TIEBREAKER` row:
       - avg base target
       - bull target from bull model
       - bear target from bear model
       - avg confidence
       - verdict from upside/downside bands

28. **Build consensus formatter**
    1. Add `Code in JavaScript` to validate and flatten first-round results if no tiebreaker is needed.
    2. Emit rows with:
       - `resolution_method`
       - `conviction`
       - `gap_pct`
       - `verdict_chatgpt`
       - `verdict_gemini`
       - `skip_row`

29. **Unify both resolution paths**
    1. Add `Merge7`.
    2. Connect `Code in JavaScript3` and `Code in JavaScript`.

30. **Filter invalid rows**
    1. Add `If` node checking `skip_row == false`.
    2. False branch should return to `loop_over_tickers` so failures do not stop the batch.

31. **Add historical verdict lookup**
    1. Create the results spreadsheet with columns used by the final append node.
    2. Add `Get Previous Verdict` Google Sheets node filtering by `stock`.

32. **Detect thesis reversals**
    1. Add `Thesis Reversal Enricher` code node.
    2. Compare current verdict to most recent previous verdict.
    3. Output:
       - `previous_verdict`
       - `previous_date`
       - `thesis_reversal`
       - `thesis_reversal_msg`

33. **Create alert filter**
    1. Add `Alert Filter` code node.
    2. Parse verdict from rationale.
    3. Set `should_alert = (BUY or SELL) and confidence >= 50`.
    4. Build Telegram-ready `alert_message`.

34. **Add actionable/non-actionable routing**
    1. Add `If Alert?` node on `should_alert`.
    2. False branch goes directly to `write_sentiment_to_sheets`.

35. **Add technical filter enrichment**
    1. Add Alpha Vantage HTTP Request nodes:
       - `Get 50MA`
       - `Get 200MA`
       - `Get RSI`
       - `Get Earnings`
    2. Chain them in that order via `MA Check` and `Entry Signal Enricher`.
    3. `MA Check` should compute:
       - `ma_50`
       - `ma_200`
       - `ma_check_passed`
       - `ma_signal`
       - `trend_tier`
    4. `Entry Signal Enricher` should compute:
       - `rsi_14`
       - `rsi_signal`
       - `next_earnings_date`
       - `days_to_earnings`
       - `earnings_warning`

36. **Route strong vs watchlist alerts**
    1. Add `Above 50MA?` If node testing `ma_check_passed`.
    2. True → `Telegram Alert`
    3. False → `Watchlist Telegram`

37. **Configure Telegram alerts**
    1. Add Telegram credentials.
    2. Add `Telegram Alert` sending `{{$json.alert_message}}` with Markdown parse mode.
    3. Add `Watchlist Telegram` with the custom multiline watchlist message.
    4. Use your actual Telegram chat ID or channel/group ID.

38. **Add thesis reversal notification**
    1. Add `Is Thesis Reversal?` If node.
    2. True → `Thesis Reversal Alert`
    3. False → `write_sentiment_to_sheets`
    4. `Thesis Reversal Alert` sends `thesis_reversal_msg`.

39. **Create result storage**
    1. Add `write_sentiment_to_sheets` Google Sheets append node.
    2. Create results columns including:
       - `stock`, `date`, `model`
       - `current_price`
       - `pt_bear`, `pt_base`, `pt_bull`
       - `f_score`, `confidence`
       - `gap_pct`, `resolution_method`, `conviction`
       - `rationale`
       - `verdict_chatgpt`, `verdict_gemini`
       - `ma_50`, `ma_200`, `ma_signal`
       - `rsi_14`, `rsi_signal`
       - `trend_tier`
       - `next_earnings_date`
       - `previous_verdict`
       - `thesis_reversal`
    3. Connect all final branches into this append node.

40. **Return to loop**
    1. Add `Wait` node after sheet append.
    2. Connect it back to `loop_over_tickers` to continue processing remaining symbols.

41. **Create end-of-batch summary**
    1. Connect the done output of `loop_over_tickers` to `Build Summary`.
    2. Use a code node to aggregate all processed items into one report.
    3. Add `Send a text message` Telegram node to send the summary.

42. **Testing checklist**
    1. Test with 1 ticker first.
    2. Validate each sheet schema exactly matches mapped fields.
    3. Check Alpha Vantage note/rate-limit payloads.
    4. Confirm both LLM nodes return valid JSON.
    5. Validate `stock` field exists in every branch before merges.
    6. Confirm Telegram Markdown renders correctly.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow overview note: retrieves stock tickers from Google Sheets, fetches news, uses two LLMs, writes sentiment/rationale back to Sheets. | Internal sticky note |
| Accuracy Tracker idea: create a separate monthly workflow to score prediction accuracy after 30 days and log to a new `Accuracy_Tracking` sheet. | Sentiments sheet ID noted in workflow: `1fbptcVE0mBjaIZJHkJzFBTdoVJVLgQOWmJXpCx40YyE` |
| The workflow title in JSON is `AI Institutional Stock Valuation Engine with Risk Scoring & Scenario Targets V10`, while the provided title/description emphasize institutional valuation, dual-model consensus, tiebreaker logic, and alerts. | Project naming/context |
| The sticky notes still use older “sentiment analysis” wording in places, but the actual implementation is broader: valuation, targets, consensus, technical filters, and alerting. | Implementation note |
| There are no sub-workflow nodes in this workflow. | Architecture note |

### Important implementation observations
| Note Content | Context or Link |
|---|---|
| The cache validation logic does not implement a true TTL freshness test despite the description claiming freshness-window logic. It mainly distinguishes cache miss vs manual refresh. | `Check the cache` code |
| The description says tiebreaker triggers above a 20% base-target gap, but the actual code uses `gap_pct > 25`. | `Clean Up Results from First Round` |
| Alpha Vantage API keys are hardcoded in URLs. Move them to secure credentials or environment variables before production use. | Security/maintenance |
| `alphavantage - Current Price Second` appears redundant but is used to refresh current price before reading cached fundamentals. | Optimization note |
| The workflow contains several sticky notes and one proposed “accuracy tracker” concept that is not implemented in the actual execution graph. | Scope note |

If you want, I can also produce:
1. a **node-by-node execution map in Mermaid**, or  
2. a **cleaned developer specification of all sheet schemas and expected columns**.