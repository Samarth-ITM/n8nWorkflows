Analyze stocks in Telegram with Twelve Data, GPT-4o and o3-mini

https://n8nworkflows.xyz/workflows/analyze-stocks-in-telegram-with-twelve-data--gpt-4o-and-o3-mini-13981


# Analyze stocks in Telegram with Twelve Data, GPT-4o and o3-mini

# 1. Workflow Overview

This workflow implements a Telegram-based stock analysis assistant named **Ade**. It receives Telegram messages, filters unsupported inputs, routes valid text into an AI agent powered by **OpenRouter o3-mini**, and allows that agent to call three sub-workflows plus one risk-analysis sub-workflow backed by **Twelve Data** and **OpenAI GPT-4o** for deeper single-stock analysis.

Its main use cases are:

- Answering single-stock analysis questions
- Comparing multiple stocks
- Analyzing sector performance
- Calculating stock risk metrics
- Responding conversationally in Telegram with memory

## 1.1 Telegram Input Reception and Preprocessing

This block captures Telegram messages, normalizes the payload, extracts the message text, user name, and chat ID, and flags unsupported non-text messages.

## 1.2 Message Validation and Response Routing

This block branches based on whether the incoming Telegram event contains usable text. Unsupported content is answered directly; valid text is sent to the AI agent.

## 1.3 Conversational AI Orchestration

This block contains the main agent, its memory, its language model, and its callable workflow tools. It interprets user intent and decides whether to answer directly or invoke one of the sub-workflows.

## 1.4 Telegram Output Delivery

This block sends the AI agent response back to Telegram. It also contains the unsupported-message response path.

## 1.5 Sub-Workflow: Risk Calculator Tool

This callable sub-workflow receives a ticker and optional position size, fetches historical and fundamental data from Twelve Data, computes risk metrics, and returns both structured JSON and a formatted risk report.

## 1.6 Sub-Workflow: Stock Comparison Tool

This callable sub-workflow extracts one or more stock tickers, retrieves quote/profile/RSI data from Twelve Data, compares them, and returns a formatted comparison report plus structured summary fields.

## 1.7 Sub-Workflow: Sector Analysis Tool

This callable sub-workflow receives a sector name, seeds representative stocks for that sector, enriches them with Twelve Data quote/statistics data, computes sector-level metrics, and returns a formatted sector report.

## 1.8 Sub-Workflow: Single Stock Analysis Tool

This callable sub-workflow receives one stock ticker, gathers seven parallel market/fundamental/technical streams from Twelve Data, formats them for AI, sends them to GPT-4o for institutional-style analysis, and returns the generated report.

---

# 2. Block-by-Block Analysis

## 2.1 Telegram Input Reception and Preprocessing

**Overview:**  
This block starts the main Telegram bot flow. It receives incoming Telegram messages and reshapes the event into a simpler structure that downstream nodes can handle reliably.

**Nodes Involved:**
- Telegram Trigger1
- PreProcessing1
- Filter Message Type

### Telegram Trigger1
- **Type and technical role:** `n8n-nodes-base.telegramTrigger` — entry point for Telegram updates.
- **Configuration choices:** Configured to listen only for `message` updates.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `PreProcessing1`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Telegram credential misconfiguration
  - Webhook registration issues
  - Telegram bot permissions or webhook conflicts
- **Sub-workflow reference:** None.

### PreProcessing1
- **Type and technical role:** `n8n-nodes-base.set` — normalizes the Telegram payload.
- **Configuration choices:** Uses dot notation to create:
  - `message.text`
  - `user.name`
  - `chat.id`
- **Key expressions or variables used:**
  - `{{ $json?.message?.text || '' }}`
  - `{{ $json?.message?.from?.first_name || 'User' }}`
  - `{{ $json?.message?.chat?.id }}`
- **Input and output connections:** Input from `Telegram Trigger1`; output to `Filter Message Type`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Missing Telegram `message` object
  - Empty or undefined chat ID
- **Sub-workflow reference:** None.

### Filter Message Type
- **Type and technical role:** `n8n-nodes-base.code` — validates the incoming payload and prepares routing fields.
- **Configuration choices:** Custom JavaScript:
  - Reads `message.text`
  - Resolves chat ID from multiple possible locations
  - Resolves user name from multiple possible locations
  - Handles emoji-only names
  - Returns either a `skip: true` fallback response or a `skip: false` normalized text input
- **Key expressions or variables used:**
  - `input.message`
  - `input.chat?.id || input.chat_id || msg.chat?.id`
  - `input.user?.name`, `msg.from?.first_name`, etc.
  - Output fields: `skip`, `text`, `chat_id`, `user_name`, `display_name`, `fallback_message`
- **Input and output connections:** Input from `PreProcessing1`; output to `Is Valid Message?`
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Non-text Telegram messages
  - Missing name fields
  - Missing chat ID causing later Telegram send failures
  - Empty text being treated as unsupported
- **Sub-workflow reference:** None.

---

## 2.2 Message Validation and Response Routing

**Overview:**  
This block decides whether the workflow should answer with a fallback message or continue to the AI agent. It is the guardrail between input validation and AI processing.

**Nodes Involved:**
- Is Valid Message?
- Send Unsupported Message

### Is Valid Message?
- **Type and technical role:** `n8n-nodes-base.if` — branches based on the `skip` flag.
- **Configuration choices:** Checks whether `{{ $json.skip }}` equals `true`.
- **Key expressions or variables used:**
  - `{{ $json.skip }}`
- **Input and output connections:** Input from `Filter Message Type`; true branch to `Send Unsupported Message`; false branch to `AI Agent1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Boolean type mismatches if upstream output changes
  - Misinterpretation of branch semantics by maintainers
- **Sub-workflow reference:** None.

### Send Unsupported Message
- **Type and technical role:** `n8n-nodes-base.telegram` — sends a fallback message to Telegram.
- **Configuration choices:**
  - Sends `{{ $json.fallback_message }}`
  - Uses `{{ $json.chat_id }}`
  - Markdown parse mode enabled
  - Attribution disabled
- **Key expressions or variables used:**
  - `{{ $json.fallback_message }}`
  - `{{ $json.chat_id }}`
- **Input and output connections:** Input from `Is Valid Message?` true branch; no downstream node.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - Invalid or empty `chat_id`
  - Telegram markdown formatting issues
  - Credential/auth problems
- **Sub-workflow reference:** None.

---

## 2.3 Conversational AI Orchestration

**Overview:**  
This is the main intelligence layer. The agent receives user text, uses memory, and can call one of four tool workflows depending on whether the request concerns a single stock, stock comparison, sector analysis, or risk analysis.

**Nodes Involved:**
- AI Agent1
- Conversation Memory1
- Analytics_Model1
- Stock Analysis Tool1
- Stock Comparison Tool1
- Sector Analysis Tool1
- Risk Calculator Tool1

### AI Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — central orchestrator for conversational reasoning and tool calling.
- **Configuration choices:**
  - Prompt input comes from `{{ $json.text }}`
  - Uses a long system prompt defining persona, operating procedure, output style, and required tool usage
  - Explicitly instructs first interaction behavior, tool selection rules, error handling, and final verdict formatting
- **Key expressions or variables used:**
  - `{{ $json.text }}`
- **Input and output connections:**
  - Main input from `Is Valid Message?` false branch
  - Language model input from `Analytics_Model1`
  - Memory input from `Conversation Memory1`
  - Tool inputs from all four tool workflow nodes
  - Main output to `Send Response1`
- **Version-specific requirements:** Type version `1.7`.
- **Edge cases or potential failure types:**
  - Tool misselection if prompt interpretation is poor
  - Hallucinated responses if tools fail and prompt constraints are ignored
  - Long outputs potentially exceeding Telegram limits
  - Missing chat-specific context because memory session key is not dynamic
- **Sub-workflow reference:** Calls:
  - `Stock Analysis Tool1`
  - `Stock Comparison Tool1`
  - `Sector Analysis Tool1`
  - `Risk Calculator Tool1`

### Conversation Memory1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` — stores chat memory for the agent.
- **Configuration choices:**
  - Uses a custom session key
  - `sessionKey` is hardcoded as `7107208579`
- **Key expressions or variables used:** Hardcoded `=7107208579`
- **Input and output connections:** Connected as AI memory to `AI Agent1`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Memory is not actually per-user despite sticky-note claims
  - Cross-user context leakage in multi-user deployments
- **Sub-workflow reference:** None.

### Analytics_Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — primary LLM for the agent.
- **Configuration choices:**
  - Model: `openai/o3-mini`
  - `maxTokens: 4000`
  - No temperature override
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected as language model to `AI Agent1`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - OpenRouter auth or rate limit issues
  - Model availability changes
  - Long tool outputs causing token exhaustion
- **Sub-workflow reference:** None.

### Stock Analysis Tool1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — exposes a child workflow as the `AnalyzeStock` tool.
- **Configuration choices:**
  - Tool name: `AnalyzeStock`
  - Calls workflow `TwelveData_Pro_Helper`
  - Passes `query = {{ $json.query }}`
- **Key expressions or variables used:**
  - `{{ $json.query }}`
- **Input and output connections:** Connected as AI tool to `AI Agent1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Sub-workflow ID mismatch after import
  - Agent passes malformed ticker text instead of pure symbol
  - Child workflow failures from Twelve Data or OpenAI
- **Sub-workflow reference:** Invokes workflow `TwelveData_Pro_Helper`.

### Stock Comparison Tool1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — exposes stock comparison as `CompareStocks`.
- **Configuration choices:**
  - Tool name: `CompareStocks`
  - Calls workflow `Stock_Comparison_Tool`
  - Passes `tickers = {{ $json.tickers }}`
- **Key expressions or variables used:**
  - `{{ $json.tickers }}`
- **Input and output connections:** Connected as AI tool to `AI Agent1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Invalid ticker parsing in child workflow
  - Empty ticker list
  - Rate limiting from multiple quote requests
- **Sub-workflow reference:** Invokes workflow `Stock_Comparison_Tool`.

### Sector Analysis Tool1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — exposes sector analysis as `SectorAnalysis`.
- **Configuration choices:**
  - Tool name: `SectorAnalysis`
  - Calls workflow `Sector_Analysis_Tool_Enhanced`
  - Passes `sector = {{ $json.sector }}`
- **Key expressions or variables used:**
  - `{{ $json.sector }}`
- **Input and output connections:** Connected as AI tool to `AI Agent1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Agent passing an unsupported sector label
  - Child workflow falling back unexpectedly to Technology
- **Sub-workflow reference:** Invokes workflow `Sector_Analysis_Tool_Enhanced`.

### Risk Calculator Tool1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWorkflow` — exposes risk analysis as `CalculateRisk`.
- **Configuration choices:**
  - Tool name: `CalculateRisk`
  - Calls workflow `Risk_Calculator_Tool`
  - Passes:
    - `ticker = {{ $json.ticker }}`
    - `positionSize = {{ $json.positionSize || 10000 }}`
- **Key expressions or variables used:**
  - `{{ $json.ticker }}`
  - `{{ $json.positionSize || 10000 }}`
- **Input and output connections:** Connected as AI tool to `AI Agent1`.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Missing ticker
  - Numeric coercion issues on `positionSize`
  - Downstream divide-by-zero cases if no market data is returned
- **Sub-workflow reference:** Invokes workflow `Risk_Calculator_Tool`.

---

## 2.4 Telegram Output Delivery

**Overview:**  
This block sends the final AI-generated answer back to Telegram. It is the final user-facing output path for valid messages.

**Nodes Involved:**
- Send Response1

### Send Response1
- **Type and technical role:** `n8n-nodes-base.telegram` — sends the main AI response to Telegram.
- **Configuration choices:**
  - Sends `{{ $json.output }}`
  - Markdown enabled
  - Attribution disabled
  - `chatId` is configured as `=`, which is effectively incomplete
- **Key expressions or variables used:**
  - `{{ $json.output }}`
- **Input and output connections:** Input from `AI Agent1`.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - This node is likely misconfigured because `chatId` is blank
  - Telegram markdown formatting can break on AI-generated tables/content
  - Telegram message length limits
- **Sub-workflow reference:** None.

---

## 2.5 Sub-Workflow: Risk Calculator Tool

**Overview:**  
This sub-workflow calculates quantitative risk metrics for a stock position using Twelve Data historical and statistical fundamentals. It returns a formatted report and structured numeric outputs suitable for downstream AI interpretation.

**Nodes Involved:**
- Trigger
- Get Historical Data
- Get Statistics
- Merge
- Calculate Risk Metrics
- Output

### Trigger
- **Type and technical role:** `n8n-nodes-base.executeWorkflowTrigger` — entry point for child workflow execution.
- **Configuration choices:** `inputSource: passthrough`
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Get Historical Data` and `Get Statistics`.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Missing expected input fields such as `ticker`
- **Sub-workflow reference:** Entry point for the risk tool workflow.

### Get Historical Data
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches time series.
- **Configuration choices:** Operation `getTimeSeries`
- **Key expressions or variables used:**
  - `{{ $json.ticker }}{{ $json.chatInput }}`
- **Input and output connections:** Input from `Trigger`; output to `Merge`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:**
  - Concatenation of `ticker` and `chatInput` may create invalid symbols
  - Missing or invalid ticker
  - Twelve Data auth, quota, or symbol errors
- **Sub-workflow reference:** Risk tool.

### Get Statistics
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches fundamentals statistics.
- **Configuration choices:** Resource `fundamentals`, operation `getStatistics`
- **Key expressions or variables used:**
  - `{{ $json.ticker }}{{ $json.chatInput }}`
- **Input and output connections:** Input from `Trigger`; output to `Merge`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:**
  - Same malformed symbol risk as above
  - Incomplete statistics payloads
- **Sub-workflow reference:** Risk tool.

### Merge
- **Type and technical role:** `n8n-nodes-base.merge` — combines historical and statistics outputs by position.
- **Configuration choices:**
  - Mode `combine`
  - `combineByPosition`
  - `includeUnpaired: true`
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from `Get Historical Data` and `Get Statistics`; output to `Calculate Risk Metrics`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Misaligned item pairing
  - One API branch failing and creating partially merged objects
- **Sub-workflow reference:** Risk tool.

### Calculate Risk Metrics
- **Type and technical role:** `n8n-nodes-base.code` — computes volatility, drawdown, VaR, leverage, valuation risk, and formatted report text.
- **Configuration choices:** Custom JavaScript calculates:
  - Daily and annualized volatility
  - Maximum drawdown and dates
  - 95% 1-day VaR
  - 52-week positioning
  - MA50/MA200 signals
  - Short interest metrics
  - Margin/ROE/ROA/cash/debt metrics
  - Composite risk score from five weighted components
  - Suggested stop-loss and portfolio sizing
- **Key expressions or variables used:**
  - `$input.first().json.positionSize || 10000`
  - nested Twelve Data fields such as `statistics.valuations_metrics`, `financials`, `stock_price_summary`
- **Input and output connections:** Input from `Merge`; output to `Output`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Empty `values` array leading to invalid calculations
  - Division by zero in return/position/range calculations
  - Missing nested fundamentals producing `NaN`
  - Assumes percentages stored as decimals in some fields
- **Sub-workflow reference:** Risk tool.

### Output
- **Type and technical role:** `n8n-nodes-base.set` — shapes final child workflow response.
- **Configuration choices:** Returns:
  - `ticker`
  - `companyName`
  - `response = {{ $json.formatted }}`
- **Key expressions or variables used:**
  - `{{ $json.ticker }}`
  - `{{ $json.companyName }}`
  - `{{ $json.formatted }}`
- **Input and output connections:** Input from `Calculate Risk Metrics`; no downstream node shown.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Missing formatted report if prior code fails
- **Sub-workflow reference:** Final output of the risk tool.

---

## 2.6 Sub-Workflow: Stock Comparison Tool

**Overview:**  
This sub-workflow parses one or more ticker symbols from free text, retrieves quote, RSI, and company profile data for each ticker, then builds a comparative report and summary statistics.

**Nodes Involved:**
- Split Tickers1
- Get Quotes1
- Get RSI1
- Get Profile1
- Merge Stock Data1
- Format Comparison1
- Output1

### Split Tickers1
- **Type and technical role:** `n8n-nodes-base.code` — extracts ticker symbols from input text.
- **Configuration choices:** Accepts values from `chatInput`, `tickers`, `message`, or `text`; uppercases; extracts 1–5 letter tokens; removes common stop words; deduplicates.
- **Key expressions or variables used:**
  - `input.chatInput || input.tickers || input.message || input.text || ''`
  - regex `\b[A-Z]{1,5}\b`
- **Input and output connections:** No explicit incoming connection shown in this JSON, so this appears to be part of a separate callable workflow definition; outputs to `Get Quotes1`, `Get RSI1`, and `Get Profile1`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - No ticker found throws an error
  - False positives for valid-looking English words
  - Does not support symbols with dots or dashes
- **Sub-workflow reference:** Stock comparison tool entry logic.

### Get Quotes1
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches current quote.
- **Configuration choices:** Default quote operation.
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Split Tickers1`; output to `Merge Stock Data1`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:**
  - Invalid symbol
  - Rate limits when many tickers are processed
- **Sub-workflow reference:** Stock comparison tool.

### Get RSI1
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches RSI indicator.
- **Configuration choices:** Resource `technicalIndicators`, operation `rsi`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Split Tickers1`; output to `Merge Stock Data1`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:**
  - Missing values array
  - Indicator API errors
- **Sub-workflow reference:** Stock comparison tool.

### Get Profile1
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches company profile/fundamentals base info.
- **Configuration choices:** Resource `fundamentals`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Split Tickers1`; output to `Merge Stock Data1`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:**
  - Sparse profile data
- **Sub-workflow reference:** Stock comparison tool.

### Merge Stock Data1
- **Type and technical role:** `n8n-nodes-base.merge` — combines quote, RSI, and profile per ticker by position.
- **Configuration choices:**
  - `combine`
  - `combineByPosition`
  - `numberInputs: 3`
  - `includeUnpaired: true`
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from `Get Quotes1`, `Get RSI1`, `Get Profile1`; output to `Format Comparison1`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Positional mismatch if one request errors or returns fewer items
- **Sub-workflow reference:** Stock comparison tool.

### Format Comparison1
- **Type and technical role:** `n8n-nodes-base.code` — creates structured stock comparison objects and a formatted report.
- **Configuration choices:** Calculates:
  - ranking by daily % change
  - best/worst performer
  - performance gap
  - average RSI
  - overbought/oversold lists
  - sectors covered
  - quick verdict per stock
- **Key expressions or variables used:** Reads merged fields such as `close`, `percent_change`, `values[0].rsi`, `sector`, `industry`, `fifty_two_week`
- **Input and output connections:** Input from `Merge Stock Data1`; output to `Output1`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - If all items error, resulting comparison may be empty
  - Mixed quote/profile field naming assumptions
  - `toFixed` on invalid numbers can fail if not sanitized upstream
- **Sub-workflow reference:** Stock comparison tool.

### Output1
- **Type and technical role:** `n8n-nodes-base.set` — final output payload for the comparison tool.
- **Configuration choices:** Sets `response = {{ $json.response }}`
- **Key expressions or variables used:** `{{ $json.response }}`
- **Input and output connections:** Input from `Format Comparison1`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Missing response if prior code fails
- **Sub-workflow reference:** Final output of stock comparison tool.

---

## 2.7 Sub-Workflow: Sector Analysis Tool

**Overview:**  
This sub-workflow analyzes a predefined sector using seeded representative stocks. It enriches those stocks with live quote and statistics data, calculates sector-wide metrics, and produces an investment-oriented sector report.

**Nodes Involved:**
- Set Sector
- Get Sector Stocks
- Get Quote
- Get Statistics1
- Merge2
- Merge & Enrich Data
- Analyze Sector
- Output2

### Set Sector
- **Type and technical role:** `n8n-nodes-base.set` — normalizes the incoming sector query.
- **Configuration choices:** Creates uppercase `sector` from `query`, `chatInput`, or `ticker`.
- **Key expressions or variables used:**
  - `{{ ($json.query || $json.chatInput || $json.ticker || '').toString().trim().toUpperCase() }}`
- **Input and output connections:** No incoming connection shown here; outputs to `Get Sector Stocks`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Converts sector names to uppercase while lookup table uses title case, causing fallback to Technology unless exact upstream handling compensates
- **Sub-workflow reference:** Sector analysis tool entry logic.

### Get Sector Stocks
- **Type and technical role:** `n8n-nodes-base.code` — seeds 5 representative stocks for each sector using hardcoded fallback metadata.
- **Configuration choices:** Lookup table for:
  - Technology
  - Healthcare
  - Finance
  - Energy
  - Consumer
  - Real Estate
  - Utilities
  - Industrials
- **Key expressions or variables used:** `const sector = $input.first().json.sector || 'Technology';`
- **Input and output connections:** Input from `Set Sector`; output to `Get Quote` and `Get Statistics1`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - If sector key mismatch occurs, defaults to Technology
  - Several `knownMarketCap` values are placeholder arithmetic, not realistic production-grade values
- **Sub-workflow reference:** Sector analysis tool.

### Get Quote
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches live quote data.
- **Configuration choices:** Default quote fetch.
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Get Sector Stocks`; output to `Merge2`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Invalid symbol, API quota, partial data.
- **Sub-workflow reference:** Sector analysis tool.

### Get Statistics1
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches stock statistics/fundamentals.
- **Configuration choices:** Resource `fundamentals`, operation `getStatistics`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Get Sector Stocks`; output to `Merge2`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Missing statistics sections for some symbols.
- **Sub-workflow reference:** Sector analysis tool.

### Merge2
- **Type and technical role:** `n8n-nodes-base.merge` — combines quote and statistics by position.
- **Configuration choices:**
  - `combine`
  - `combineByPosition`
  - `includeUnpaired: true`
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from `Get Quote` and `Get Statistics1`; output to `Merge & Enrich Data`.
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:** Positional mismatches after API failures.
- **Sub-workflow reference:** Sector analysis tool.

### Merge & Enrich Data
- **Type and technical role:** `n8n-nodes-base.code` — merges live and fallback data into unified stock records.
- **Configuration choices:** Uses live data first, then hardcoded fallback values for market cap, PE, beta, industry, CEO, employees.
- **Key expressions or variables used:**
  - `$('Set Sector').first().json.sector`
  - `$('Get Sector Stocks').all()`
- **Input and output connections:** Input from `Merge2`; output to `Analyze Sector`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Depends on node-name references remaining unchanged
  - Missing quote/statistics fields can create zeros that skew averages
- **Sub-workflow reference:** Sector analysis tool.

### Analyze Sector
- **Type and technical role:** `n8n-nodes-base.code` — computes sector aggregates and formatted report.
- **Configuration choices:** Calculates:
  - average % change
  - total market cap/revenue/employees
  - average PE/beta/profit margin/revenue growth
  - gainers/losers
  - industry breakdown
  - sector strength
  - recommendation and top pick/avoid
- **Key expressions or variables used:** Uses all incoming stock records from `$input.all()`
- **Input and output connections:** Input from `Merge & Enrich Data`; output to `Output2`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Empty stock list causes divide-by-zero or undefined top/worst records
  - Uses percentages stored as decimals in some places
- **Sub-workflow reference:** Sector analysis tool.

### Output2
- **Type and technical role:** `n8n-nodes-base.set` — final child workflow response for sector analysis.
- **Configuration choices:** Sets `response = {{ $json.response }}`
- **Key expressions or variables used:** `{{ $json.response }}`
- **Input and output connections:** Input from `Analyze Sector`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Missing response if prior step fails.
- **Sub-workflow reference:** Final output of sector analysis tool.

---

## 2.8 Sub-Workflow: Single Stock Analysis Tool

**Overview:**  
This sub-workflow performs the deepest analysis for one stock. It collects seven data streams, computes derived indicators and context, then asks GPT-4o to produce an institutional-style analyst report.

**Nodes Involved:**
- Set Stock Ticker
- Get Real-Time Quote
- Get Time Series (90 days)
- RSI Indicator
- MACD Indicator
- Bollinger Bands
- Get Company Profile
- Get Income Statement
- Merge All Data
- Format Data for AI
- AI Analysis
- Response Output

### Set Stock Ticker
- **Type and technical role:** `n8n-nodes-base.set` — normalizes the incoming ticker symbol.
- **Configuration choices:** Creates uppercase `ticker` from `query`, `chatInput`, or `ticker`.
- **Key expressions or variables used:**
  - `{{ ($json.query || $json.chatInput || $json.ticker || '').toString().trim().toUpperCase() }}`
- **Input and output connections:** No explicit incoming connection shown here; outputs to all seven Twelve Data fetch nodes.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Does not validate ticker format
- **Sub-workflow reference:** Stock analysis tool entry logic.

### Get Real-Time Quote
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches live quote.
- **Configuration choices:** Default quote operation.
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Invalid ticker, missing fields, quota errors.
- **Sub-workflow reference:** Stock analysis tool.

### Get Time Series (90 days)
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches time series.
- **Configuration choices:** Operation `getTimeSeries`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Too-short history for 10-day/30-day analysis assumptions.
- **Sub-workflow reference:** Stock analysis tool.

### RSI Indicator
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches RSI.
- **Configuration choices:** Resource `technicalIndicators`, operation `rsi`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Empty `values`.
- **Sub-workflow reference:** Stock analysis tool.

### MACD Indicator
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches MACD.
- **Configuration choices:** Resource `technicalIndicators`, operation `macd`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Missing histogram/signal values.
- **Sub-workflow reference:** Stock analysis tool.

### Bollinger Bands
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches Bollinger Bands.
- **Configuration choices:** Resource `technicalIndicators`, operation `bbands`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Missing band values.
- **Sub-workflow reference:** Stock analysis tool.

### Get Company Profile
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches company fundamentals/profile.
- **Configuration choices:** Resource `fundamentals`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Sparse profile data.
- **Sub-workflow reference:** Stock analysis tool.

### Get Income Statement
- **Type and technical role:** `n8n-nodes-twelve-data.twelveData` — fetches income statement.
- **Configuration choices:** Resource `fundamentals`, operation `getIncomeStatement`
- **Key expressions or variables used:** `{{ $json.ticker }}`
- **Input and output connections:** Input from `Set Stock Ticker`; output to `Merge All Data`.
- **Version-specific requirements:** Twelve Data node version `1`.
- **Edge cases or potential failure types:** Less than two fiscal periods causing YoY growth calculations to fail.
- **Sub-workflow reference:** Stock analysis tool.

### Merge All Data
- **Type and technical role:** `n8n-nodes-base.merge` — combines seven parallel inputs by position.
- **Configuration choices:**
  - `combine`
  - `combineByPosition`
  - `numberInputs: 7`
  - `includeUnpaired: true`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from all seven data nodes; output to `Format Data for AI`.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Data stream mismatch
  - Partial responses creating incomplete merged object
- **Sub-workflow reference:** Stock analysis tool.

### Format Data for AI
- **Type and technical role:** `n8n-nodes-base.code` — builds structured market/fundamental context and a formatted prompt.
- **Configuration choices:** Computes:
  - quote object
  - company profile object
  - latest technical indicators
  - RSI/MACD/Bollinger interpretations
  - 5-day momentum
  - 5-day volume trend
  - fundamentals and YoY revenue/net income growth
  - price history table
  - a long text prompt for GPT-4o
- **Key expressions or variables used:** Relies on merged fields such as `data.values[0]`, `data.income_statement[0]`, `data.income_statement[1]`
- **Input and output connections:** Input from `Merge All Data`; output to `AI Analysis`.
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Assumes all indicators share the same `values` structure after merge
  - Fails if `income_statement` has fewer than 2 rows
  - Divides by zero if prior averages are zero
- **Sub-workflow reference:** Stock analysis tool.

### AI Analysis
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi` — generates final single-stock report using GPT-4o.
- **Configuration choices:**
  - Model: `gpt-4o`
  - `maxTokens: 4000`
  - `temperature: 0.3`
  - Uses a highly structured system prompt with mandatory output format
  - User content is `{{ $json.formattedData }}`
- **Key expressions or variables used:** `{{ $json.formattedData }}`
- **Input and output connections:** Input from `Format Data for AI`; output to `Response Output`.
- **Version-specific requirements:** Type version `1.8`.
- **Edge cases or potential failure types:**
  - OpenAI credential/auth issues
  - Token limit reached on long prompts
  - Model may still reference unavailable metrics unless prompt is strictly followed
- **Sub-workflow reference:** Stock analysis tool.

### Response Output
- **Type and technical role:** `n8n-nodes-base.set` — shapes the final child workflow response.
- **Configuration choices:** Returns:
  - `response = {{ $json.message.content }}`
  - `ticker = {{ $('Set Stock Ticker').item.json.ticker }}`
- **Key expressions or variables used:**
  - `{{ $json.message.content }}`
  - `{{ $('Set Stock Ticker').item.json.ticker }}`
- **Input and output connections:** Input from `AI Analysis`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Depends on OpenAI node output schema containing `message.content`
  - Node-name reference breaks if renamed
- **Sub-workflow reference:** Final output of stock analysis tool.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Documentation/comment block |  |  | ## TwelveData Pro Analyst<br>### Elite Financial Analysis with Real-Time Data<br><br>Features:<br>- 70+ Technical Indicators<br>- Fundamental Analysis<br>- Multi-Stock Comparison<br>- Sector Analysis<br>- Portfolio Risk Assessment<br>- AI-Powered Insights |
| Telegram Trigger1 | telegramTrigger | Main Telegram entry point |  | PreProcessing1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| PreProcessing1 | set | Normalizes Telegram payload | Telegram Trigger1 | Filter Message Type | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Filter Message Type | code | Validates text input and builds fallback payload | PreProcessing1 | Is Valid Message? | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Is Valid Message? | if | Branches valid vs unsupported messages | Filter Message Type | Send Unsupported Message, AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Send Unsupported Message | telegram | Replies to non-text/unsupported Telegram messages | Is Valid Message? |  | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| AI Agent1 | langchain.agent | Main conversational stock-analysis agent | Is Valid Message? | Send Response1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Conversation Memory1 | memoryBufferWindow | Conversation memory for agent |  | AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Analytics_Model1 | lmChatOpenRouter | Primary reasoning model for agent |  | AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Stock Analysis Tool1 | toolWorkflow | Exposes single-stock analysis child workflow |  | AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Stock Comparison Tool1 | toolWorkflow | Exposes stock-comparison child workflow |  | AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Sector Analysis Tool1 | toolWorkflow | Exposes sector-analysis child workflow |  | AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Risk Calculator Tool1 | toolWorkflow | Exposes risk-calculation child workflow |  | AI Agent1 | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Send Response1 | telegram | Sends AI response to Telegram | AI Agent1 |  | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Trigger | executeWorkflowTrigger | Entry point for risk tool sub-workflow |  | Get Historical Data, Get Statistics | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Get Historical Data | twelveData | Fetches historical price series for risk analysis | Trigger | Merge | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Get Statistics | twelveData | Fetches fundamentals statistics for risk analysis | Trigger | Merge | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Merge | merge | Combines time series and statistics for risk computation | Get Historical Data, Get Statistics | Calculate Risk Metrics | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Calculate Risk Metrics | code | Computes quantitative risk metrics and report | Merge | Output | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Output | set | Final output of risk tool workflow | Calculate Risk Metrics |  | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Split Tickers1 | code | Extracts multiple ticker symbols from text |  | Get Quotes1, Get RSI1, Get Profile1 | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Get Quotes1 | twelveData | Fetches current quote for comparison | Split Tickers1 | Merge Stock Data1 | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Get RSI1 | twelveData | Fetches RSI for comparison | Split Tickers1 | Merge Stock Data1 | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Get Profile1 | twelveData | Fetches company profile for comparison | Split Tickers1 | Merge Stock Data1 | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Merge Stock Data1 | merge | Combines quote, RSI, and profile per ticker | Get Quotes1, Get RSI1, Get Profile1 | Format Comparison1 | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Format Comparison1 | code | Builds stock comparison report | Merge Stock Data1 | Output1 | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Output1 | set | Final output of stock comparison tool | Format Comparison1 |  | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Set Sector | set | Normalizes sector input |  | Get Sector Stocks | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Get Sector Stocks | code | Seeds representative stocks for a sector | Set Sector | Get Quote, Get Statistics1 | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Get Quote | twelveData | Fetches quote for each sector stock | Get Sector Stocks | Merge2 | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Get Statistics1 | twelveData | Fetches statistics for each sector stock | Get Sector Stocks | Merge2 | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Merge2 | merge | Combines quote and statistics for sector stocks | Get Quote, Get Statistics1 | Merge & Enrich Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Merge & Enrich Data | code | Enriches live sector data with fallbacks | Merge2 | Analyze Sector | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Analyze Sector | code | Computes sector-wide metrics and recommendation | Merge & Enrich Data | Output2 | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Output2 | set | Final output of sector analysis tool | Analyze Sector |  | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Set Stock Ticker | set | Normalizes ticker input for single-stock analysis |  | Get Real-Time Quote, Get Time Series (90 days), RSI Indicator, MACD Indicator, Bollinger Bands, Get Company Profile, Get Income Statement | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Get Real-Time Quote | twelveData | Fetches live quote for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Get Time Series (90 days) | twelveData | Fetches time series for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| RSI Indicator | twelveData | Fetches RSI for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| MACD Indicator | twelveData | Fetches MACD for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Bollinger Bands | twelveData | Fetches Bollinger Bands for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Get Company Profile | twelveData | Fetches company profile for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Get Income Statement | twelveData | Fetches income statement for single-stock analysis | Set Stock Ticker | Merge All Data | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Merge All Data | merge | Combines seven data streams for single-stock analysis | Get Real-Time Quote, Get Time Series (90 days), RSI Indicator, MACD Indicator, Bollinger Bands, Get Company Profile, Get Income Statement | Format Data for AI | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Format Data for AI | code | Converts merged stock data into AI-ready prompt | Merge All Data | AI Analysis | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| AI Analysis | langchain.openAi | Uses GPT-4o to generate final single-stock report | Format Data for AI | Response Output | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Response Output | set | Final output of single-stock analysis tool | AI Analysis |  | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Sticky Note1 | stickyNote | Documentation/comment block |  |  | ## TwelveData Pro Analyst v2<br>### Elite Financial Analysis — Ade AI<br><br>**Fixes Applied:**<br>- ✅ Dynamic chat.id (multi-user)<br>- ✅ Dynamic memory per user<br>- ✅ o3-mini model (upgraded from GPT-4o)<br>- ✅ Non-text message filter<br>- ✅ All 8 sectors in tool descriptions<br>- ✅ Improved tool descriptions<br>- ✅ Error handling in system prompt<br>- ✅ Temperature removed (o3-mini)<br>- ✅ Max tokens raised to 4000<br><br>**Tools Wired:**<br>- 📊 AnalyzeStock → Helper<br>- 📈 CompareStocks → Comparison Tool<br>- 🏢 SectorAnalysis → Sector Tool<br>- ⚠️ CalculateRisk → Risk Calculator |
| Sticky Note2 | stickyNote | Documentation/comment block |  |  | ## Risk Calculator Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a **stock ticker** and optional **position size** (default $10,000)<br>- Fetches **historical price data** and **fundamental statistics** from Twelve Data in parallel<br>- Calculates **annualised volatility**, **maximum drawdown** (with dates), and **Value at Risk** (95% confidence, 1-day)<br>- Scores **valuation risk** using P/E, PEG, Price-to-Book and EV/EBITDA<br>- Assesses **financial health** — margins, ROE, free cash flow, debt vs cash<br>- Checks **sentiment** — beta, short interest, institutional ownership, dividend sustainability<br>- Produces a **composite risk score out of 10** (volatility + drawdown + beta + valuation + leverage)<br>- Recommends a **stop-loss price** and **position sizing** based on the risk score<br>- Returns a formatted plain-text report plus structured JSON back to the calling workflow |
| Sticky Note3 | stickyNote | Documentation/comment block |  |  | ## Stock Comparison Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing ticker symbols as a comma-separated string, natural language (e.g. "compare AAPL and TSLA"), or space-separated list<br>- **Smart ticker extraction** strips common English words (AND, THE, VS, etc.) to isolate valid stock symbols, then deduplicates<br>- Fetches **Quote**, **RSI (14)**, and **Company Profile** from Twelve Data in parallel for every ticker<br>- Merges all three data sources and builds a structured comparison object per stock<br>- Calculates **summary metrics** — best/worst performer, performance gap, average RSI, overbought/oversold stocks, sectors covered<br>- Produces a **momentum ranking** sorted by daily % change<br>- Generates a **Quick Verdict per stock** combining RSI signal, daily move strength, and 52-week range positioning<br>- Returns a formatted plain-text comparison report plus structured JSON back to the calling workflow |
| Sticky Note4 | stickyNote | Documentation/comment block |  |  | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>- Triggered by a parent workflow passing a sector name (Technology, Healthcare, Finance, Energy, Consumer, Real Estate, Utilities, or Industrials)<br>- **Seeds 5 representative stocks** per sector from a built-in lookup table with known fallback data (CEO, industry, beta, P/E, market cap, employees)<br>- Fetches live **Quote** and **Statistics** from Twelve Data in parallel for each stock, then merges and enriches with fallback values where live data is unavailable<br>- Sorts stocks by market cap and keeps the top 5, building a fully structured object per stock covering price, valuation, margins, cash flow, dividends, moving averages and share stats<br>- Calculates **sector-wide metrics** — average % change, combined market cap & revenue, total employees, average P/E, average beta, average profit margin and revenue growth<br>- Rates **sector strength** on a 5-tier scale: Very Strong → Strong → Neutral → Weak → Very Weak<br>- Identifies **top 3 performers** and **bottom 2 performers** with full fundamental detail<br>- Breaks down the sector by **industry sub-group** with average performance and combined market cap per industry<br>- Generates a **Bullish / Mixed / Bearish investment recommendation** with top pick and stock to avoid<br>- Returns a formatted plain-text sector report plus structured JSON back to the calling workflow |
| Sticky Note5 | stickyNote | Documentation/comment block |  |  | ## Sector Analysis Tool<br>### Elite Financial Analysis — Ade AI<br><br>## TwelveData_Pro_Helper (Stock Analysis Tool)<br><br>- Triggered by a parent workflow passing a single stock ticker symbol<br>- Fetches **7 data streams in parallel** from Twelve Data: real-time quote, 90-day price time series, RSI, MACD, Bollinger Bands, company profile, and income statement<br>- Merges all 7 streams into a single object, then calculates:<br>  - **5-day price momentum** vs the prior 5-day average (Strong Bullish → Strong Bearish)<br>  - **Volume trend** — recent 5-day average vs prior 5-day average<br>  - **RSI signal** (Overbought / Oversold / Neutral)<br>  - **MACD direction** (Bullish / Bearish based on histogram)<br>  - **Bollinger Band position** (Above Upper / Below Lower / Inside Bands)<br>  - **Revenue & net income YoY growth** from the two most recent fiscal years<br>- Formats all data into a structured plain-text prompt including company overview, live market data, technical readings, fundamentals, and a 10-day price history table<br>- Passes the formatted prompt to **GPT-4o** with a detailed institutional analyst system prompt instructing it to perform a pre-analysis scan before writing,<br> then produce a structured report covering: Technical Analysis, Fundamental Assessment, Risk Assessment (with support/resistance/stop-loss), <br>and a Final Verdict table with BUY/HOLD/SELL, price target, timeframe, conviction level and 3 data-backed reasons<br>- Returns the AI-generated analysis plus the ticker back to the calling workflow |
| Sticky Note6 | stickyNote | Documentation/comment block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is the practical rebuild sequence. Because the JSON contains **one main Telegram workflow plus several callable child workflows embedded in the same export**, you should recreate them as **four separate callable workflows** and **one parent Telegram workflow**.

## 4.1 Create required credentials first

1. Create a **Telegram Bot API** credential in n8n.
   - Use the bot token from BotFather.
   - Name it something like `stockanalyst_twelvedata`.

2. Create a **Twelve Data API** credential.
   - Use your Twelve Data API key.
   - Name it something like `Twelve Data account`.

3. Create an **OpenRouter API** credential.
   - Use your OpenRouter key.
   - Name it something like `hgray_openrouter`.

4. Create an **OpenAI API** credential.
   - Use your OpenAI API key.
   - Name it something like `OpenAi account`.

---

## 4.2 Build the parent Telegram workflow

1. Add a **Telegram Trigger** node.
   - Name: `Telegram Trigger1`
   - Listen for `message` updates only.

2. Add a **Set** node after it.
   - Name: `PreProcessing1`
   - Enable dot notation.
   - Add fields:
     - `message.text = {{ $json?.message?.text || '' }}`
     - `user.name = {{ $json?.message?.from?.first_name || 'User' }}`
     - `chat.id = {{ $json?.message?.chat?.id }}`

3. Add a **Code** node.
   - Name: `Filter Message Type`
   - Paste the JavaScript logic that:
     - extracts text
     - normalizes `chat_id`
     - normalizes user name
     - sets `skip: true` for non-text input
     - returns a fallback message when needed

4. Add an **IF** node.
   - Name: `Is Valid Message?`
   - Condition:
     - Boolean
     - Left value: `{{ $json.skip }}`
     - Equals `true`

5. Add a **Telegram** node on the true branch.
   - Name: `Send Unsupported Message`
   - Operation: send message
   - Text: `{{ $json.fallback_message }}`
   - Chat ID: `{{ $json.chat_id }}`
   - Parse mode: Markdown
   - Disable attribution

6. Add a **LangChain Agent** node on the false branch.
   - Name: `AI Agent1`
   - Prompt text: `{{ $json.text }}`
   - Set prompt mode to define prompt directly
   - Paste the full system message from the JSON, including:
     - persona “Ade”
     - tool rules
     - final verdict rules
     - first interaction script
     - error handling requirements

7. Add an **OpenRouter Chat Model** node.
   - Name: `Analytics_Model1`
   - Model: `openai/o3-mini`
   - Max tokens: `4000`
   - Connect it to the agent as the language model.

8. Add a **Memory Buffer Window** node.
   - Name: `Conversation Memory1`
   - Session ID type: custom key
   - The JSON uses `7107208579`
   - Better implementation: use dynamic chat-specific memory such as `{{ $json.chat_id }}`
   - Connect it to the agent as memory.

9. Create four **Tool Workflow** nodes and connect each to the agent:
   - `Stock Analysis Tool1`
   - `Stock Comparison Tool1`
   - `Sector Analysis Tool1`
   - `Risk Calculator Tool1`

10. Configure **Stock Analysis Tool1**:
    - Tool name: `AnalyzeStock`
    - Description: same functional description as in JSON
    - Select the child workflow for single-stock analysis
    - Define workflow input:
      - `query = {{ $json.query }}`

11. Configure **Stock Comparison Tool1**:
    - Tool name: `CompareStocks`
    - Select the stock comparison child workflow
    - Input:
      - `tickers = {{ $json.tickers }}`

12. Configure **Sector Analysis Tool1**:
    - Tool name: `SectorAnalysis`
    - Select the sector analysis child workflow
    - Input:
      - `sector = {{ $json.sector }}`

13. Configure **Risk Calculator Tool1**:
    - Tool name: `CalculateRisk`
    - Select the risk calculator child workflow
    - Inputs:
      - `ticker = {{ $json.ticker }}`
      - `positionSize = {{ $json.positionSize || 10000 }}`

14. Add a final **Telegram** node after the agent.
   - Name: `Send Response1`
   - Text: use the agent output field, typically `{{ $json.output }}`
   - Chat ID: this must be dynamic, for example `{{ $('Filter Message Type').item.json.chat_id }}`
   - Parse mode: Markdown
   - Disable attribution

15. Activate the Telegram webhook by saving and enabling the workflow.

**Important correction:**  
In the provided JSON, `Send Response1` has an effectively blank `chatId`. You must fix this or the workflow will not send valid responses back to Telegram.

---

## 4.3 Build the Risk Calculator child workflow

1. Create a new workflow named similar to `Risk_Calculator_Tool`.

2. Add an **Execute Workflow Trigger** node.
   - Name: `Trigger`
   - Input source: passthrough

3. Add two **Twelve Data** nodes from the trigger in parallel:
   - `Get Historical Data`
     - Symbol: `{{ $json.ticker }}`
     - Operation: `getTimeSeries`
   - `Get Statistics`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `fundamentals`
     - Operation: `getStatistics`

4. Add a **Merge** node.
   - Name: `Merge`
   - Mode: `combine`
   - Combine by: position
   - Include unpaired: true

5. Add a **Code** node.
   - Name: `Calculate Risk Metrics`
   - Paste the JavaScript from the JSON.
   - It expects:
     - historical `values`
     - `statistics`
     - optional `positionSize`

6. Add a **Set** node.
   - Name: `Output`
   - Return:
     - `ticker = {{ $json.ticker }}`
     - `companyName = {{ $json.companyName }}`
     - `response = {{ $json.formatted }}`

7. Save the workflow and note its workflow ID for the parent tool node.

**Recommended fix:** remove `{{ $json.chatInput }}` concatenation from the symbol fields in both Twelve Data nodes unless you intentionally support raw chat text inside this child workflow.

---

## 4.4 Build the Stock Comparison child workflow

1. Create a new workflow named similar to `Stock_Comparison_Tool`.

2. Add an **Execute Workflow Trigger** node if you want a complete callable entry point.
   - The JSON excerpt does not show one connected here, but in production this child workflow should have one.
   - Pass through the parent payload.

3. Add a **Code** node.
   - Name: `Split Tickers1`
   - Paste the ticker extraction JavaScript.
   - It reads:
     - `chatInput`
     - `tickers`
     - `message`
     - `text`

4. Add three **Twelve Data** nodes in parallel:
   - `Get Quotes1`
     - Symbol: `{{ $json.ticker }}`
   - `Get RSI1`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `technicalIndicators`
     - Operation: `rsi`
   - `Get Profile1`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `fundamentals`

5. Add a **Merge** node.
   - Name: `Merge Stock Data1`
   - Mode: `combine`
   - Combine by position
   - Number of inputs: `3`
   - Include unpaired: true

6. Add a **Code** node.
   - Name: `Format Comparison1`
   - Paste the formatting and summarization JavaScript from the JSON.

7. Add a **Set** node.
   - Name: `Output1`
   - Field:
     - `response = {{ $json.response }}`

8. Save the workflow and attach it to the parent `CompareStocks` tool node.

**Input expectation:**  
The parent should pass a string like `AAPL,MSFT,GOOGL` or natural language containing tickers.

**Output expectation:**  
At minimum, return `response` as plain text; the code also generates structured comparison fields.

---

## 4.5 Build the Sector Analysis child workflow

1. Create a new workflow named similar to `Sector_Analysis_Tool_Enhanced`.

2. Add an **Execute Workflow Trigger** node if building it as a proper callable workflow.

3. Add a **Set** node.
   - Name: `Set Sector`
   - Field:
     - `sector = {{ ($json.query || $json.chatInput || $json.ticker || '').toString().trim() }}`
   - Prefer title case normalization rather than uppercase, because the lookup table keys are title-cased.

4. Add a **Code** node.
   - Name: `Get Sector Stocks`
   - Paste the hardcoded sector stock lookup from the JSON.

5. Add two **Twelve Data** nodes in parallel:
   - `Get Quote`
     - Symbol: `{{ $json.ticker }}`
   - `Get Statistics1`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `fundamentals`
     - Operation: `getStatistics`

6. Add a **Merge** node.
   - Name: `Merge2`
   - Mode: `combine`
   - Combine by position
   - Include unpaired: true

7. Add a **Code** node.
   - Name: `Merge & Enrich Data`
   - Paste the merge/enrichment logic from the JSON.

8. Add another **Code** node.
   - Name: `Analyze Sector`
   - Paste the sector-report logic from the JSON.

9. Add a **Set** node.
   - Name: `Output2`
   - Field:
     - `response = {{ $json.response }}`

10. Save the workflow and link it to the parent `SectorAnalysis` tool.

**Important correction:**  
Do not uppercase the sector if your lookup table keys remain `Technology`, `Healthcare`, etc. Otherwise unsupported values will silently fall back to Technology.

---

## 4.6 Build the Single Stock Analysis child workflow

1. Create a new workflow named similar to `TwelveData_Pro_Helper`.

2. Add an **Execute Workflow Trigger** node if building it as a proper callable workflow.

3. Add a **Set** node.
   - Name: `Set Stock Ticker`
   - Field:
     - `ticker = {{ ($json.query || $json.chatInput || $json.ticker || '').toString().trim().toUpperCase() }}`

4. Add seven **Twelve Data** nodes in parallel:
   - `Get Real-Time Quote`
     - Symbol: `{{ $json.ticker }}`
   - `Get Time Series (90 days)`
     - Symbol: `{{ $json.ticker }}`
     - Operation: `getTimeSeries`
   - `RSI Indicator`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `technicalIndicators`
     - Operation: `rsi`
   - `MACD Indicator`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `technicalIndicators`
     - Operation: `macd`
   - `Bollinger Bands`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `technicalIndicators`
     - Operation: `bbands`
   - `Get Company Profile`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `fundamentals`
   - `Get Income Statement`
     - Symbol: `{{ $json.ticker }}`
     - Resource: `fundamentals`
     - Operation: `getIncomeStatement`

5. Add a **Merge** node.
   - Name: `Merge All Data`
   - Mode: `combine`
   - Combine by position
   - Number inputs: `7`
   - Include unpaired: true
   - Enable always output data

6. Add a **Code** node.
   - Name: `Format Data for AI`
   - Paste the code from the JSON that:
     - computes derived technicals
     - computes 5-day price momentum and volume trend
     - computes YoY revenue/net income growth
     - prepares a large text prompt for AI

7. Add an **OpenAI** node.
   - Name: `AI Analysis`
   - Model: `gpt-4o`
   - Temperature: `0.3`
   - Max tokens: `4000`
   - Add:
     - one large system message from the JSON
     - one user message with `{{ $json.formattedData }}`

8. Add a **Set** node.
   - Name: `Response Output`
   - Fields:
     - `response = {{ $json.message.content }}`
     - `ticker = {{ $('Set Stock Ticker').item.json.ticker }}`

9. Save this workflow and attach it to the parent `AnalyzeStock` tool.

**Input expectation:**  
One ticker symbol, ideally already cleaned.

**Output expectation:**  
A `response` field containing the generated report, plus `ticker`.

---

## 4.7 Recommended hardening before production

1. Fix the parent Telegram response node `chatId`.
2. Make memory session dynamic using `chat_id`.
3. Add explicit **Execute Workflow Trigger** nodes to all child workflows if they are to be callable.
4. Add validation before every Twelve Data request:
   - reject empty ticker
   - reject invalid sector labels
5. Add fallback handling when:
   - time series is empty
   - fewer than 10 candles are returned
   - income statement has fewer than 2 periods
6. Consider disabling Markdown for AI outputs or switching to MarkdownV2-safe escaping because table-like AI responses often break Telegram formatting.
7. Add truncation logic for very long responses to stay within Telegram limits.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| TwelveData Pro Analyst | Main branding in the workflow |
| Elite Financial Analysis with Real-Time Data | Main workflow positioning |
| TwelveData Pro Analyst v2 — Ade AI | Main bot versioning note |
| Fixes Applied: Dynamic chat.id (multi-user), Dynamic memory per user, o3-mini model, non-text filter, improved tool descriptions, all 8 sectors, error handling, max tokens 4000 | Main parent workflow notes |
| Risk Calculator Tool — returns formatted plain-text report plus structured JSON | Risk sub-workflow note |
| Stock Comparison Tool — smart ticker extraction and momentum ranking | Comparison sub-workflow note |
| Sector Analysis Tool — top/bottom performers and sector recommendation | Sector sub-workflow note |
| TwelveData_Pro_Helper (Stock Analysis Tool) — 7 parallel data streams, GPT-4o synthesis | Single-stock sub-workflow note |