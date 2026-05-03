Summarize stock market signals with Alpaca, xAI Grok, Telegram and WhatsApp

https://n8nworkflows.xyz/workflows/summarize-stock-market-signals-with-alpaca--xai-grok--telegram-and-whatsapp-14079


# Summarize stock market signals with Alpaca, xAI Grok, Telegram and WhatsApp

# 1. Workflow Overview

This workflow automates a **weekly stock market analysis and notification process**. Its main purpose is to check whether the US market is open on Friday, collect stock data for predefined ticker symbols, calculate technical indicators, summarize the results with **xAI Grok**, and send the final report to Telegram recipients. It also includes a **Telegram-triggered entry point** that appears intended to manually request or gate the Friday summary flow, plus fallback alert notifications when conditions are not met.

## 1.1 Entry Points

The workflow has **two entry points**:

1. **Schedule Trigger (Every Friday)**  
   Starts the automated weekly execution.

2. **Telegram Trigger**  
   Starts a manual/chat-driven execution path.

## 1.2 Market Status Validation

Both entry paths are designed to validate whether the workflow should proceed:
- The scheduled path calls a market clock API and formats the returned date/time.
- The Telegram path first checks whether the request is valid for a Friday summary, then branches either to another market-clock request or to an immediate Telegram alert.

## 1.3 Technical Market Analysis

If the market is open, the workflow:
- defines target stock ticker symbols,
- fetches stock data for those symbols,
- calculates RSI and MACD,
- generates Buy/Sell/Hold-style signals.

## 1.4 AI Summarization

The indicator output is passed into a **Basic LLM Chain** powered by **xAI Grok Chat Model** to create a human-readable market summary.

## 1.5 Notification Delivery

The generated summary is delivered to:
- an admin via Telegram,
- a Telegram channel.

If the market is not open, the workflow instead sends alert notifications via:
- WhatsApp through Rapiwa,
- Telegram.

---

# 2. Block-by-Block Analysis

## 2.1 Block A — Scheduled Weekly Entry

### Overview
This block launches the workflow automatically on a weekly schedule. It is the primary automation entry point for the Friday stock-analysis run.

### Nodes Involved
- Schedule Trigger (Every Friday)

### Node Details

#### Schedule Trigger (Every Friday)
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger that starts the workflow automatically.
- **Configuration choices:**  
  The node name indicates it is configured to run **every Friday**. The exact hour/timezone is not visible in the provided JSON, so these must be checked in the live workflow.
- **Key expressions or variables used:**  
  None visible in the JSON excerpt.
- **Input and output connections:**  
  - Input: none
  - Output: `HTTP (Get Market Clock Response)`
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Misconfigured timezone can cause execution on the wrong day/time.
  - Disabled workflow or inactive trigger prevents execution.
  - If intended to run near market open/close, daylight-saving shifts may affect timing.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block B — Telegram Manual Entry and Friday Gating

### Overview
This block receives incoming Telegram messages and appears to determine whether the user is requesting the Friday market summary. If the condition fails, it sends a Telegram alert instead of proceeding.

### Nodes Involved
- Telegram Trigger
- If (Check Friday market summary)
- Send Telegram Again Aleart Message
- HTTP (Get Market Clock Response)1

### Node Details

#### Telegram Trigger
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Webhook-based trigger for incoming Telegram bot events.
- **Configuration choices:**  
  Uses a Telegram bot webhook. The exact update types, command filters, or chat restrictions are not visible in the JSON.
- **Key expressions or variables used:**  
  Likely depends on incoming Telegram message fields such as message text, chat ID, username, or date, though the exact conditions are defined downstream.
- **Input and output connections:**  
  - Input: none
  - Output: `If (Check Friday market summary)`
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Telegram bot credentials missing or invalid.
  - Webhook registration conflict with another environment.
  - Unsupported update payload shape if the downstream IF node expects specific fields.
- **Sub-workflow reference:**  
  None.

#### If (Check Friday market summary)
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branching node used to determine whether to proceed with the Friday summary flow.
- **Configuration choices:**  
  The exact rule is not shown. Based on the name and routing:
  - **True branch** -> `HTTP (Get Market Clock Response)1`
  - **False branch** -> `Send Telegram Again Aleart Message`
- **Key expressions or variables used:**  
  Likely a condition based on message content, weekday, or command semantics.
- **Input and output connections:**  
  - Input: `Telegram Trigger`
  - Outputs:
    - True: `HTTP (Get Market Clock Response)1`
    - False: `Send Telegram Again Aleart Message`
- **Version-specific requirements:**  
  Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Expression errors if expected Telegram fields are absent.
  - Logic mismatch if command formatting varies by user.
  - If weekday validation depends on server locale/timezone, results may be inconsistent.
- **Sub-workflow reference:**  
  None.

#### Send Telegram Again Aleart Message
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message back to the user or target chat.
- **Configuration choices:**  
  Intended as an alert/fallback message when the Friday-summary condition is not met.
- **Key expressions or variables used:**  
  Likely references incoming chat ID or fixed destination chat.
- **Input and output connections:**  
  - Input: `If (Check Friday market summary)` false branch
  - Output: none
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Invalid chat ID or bot not authorized in the chat.
  - Rate limiting by Telegram.
  - Expression failure if chat ID is dynamically referenced and missing.
- **Sub-workflow reference:**  
  None.

#### HTTP (Get Market Clock Response)1
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Intended to fetch market clock status for the Telegram-driven path.
- **Configuration choices:**  
  The exact endpoint is not shown, but based on naming it likely targets a brokerage/market-status API such as Alpaca’s clock endpoint.
- **Key expressions or variables used:**  
  None visible.
- **Input and output connections:**  
  - Input: `If (Check Friday market summary)` true branch
  - Output: none in the provided connections
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Authentication/header misconfiguration.
  - Endpoint/network failure.
  - This node currently has **no downstream connection**, so the Telegram manual path appears incomplete or disconnected.
- **Sub-workflow reference:**  
  None.

**Important design note:**  
The manual Telegram path currently ends at `HTTP (Get Market Clock Response)1` with no continuation. Unless this is intentional, the manual summary flow is **incomplete** in the exported JSON.

---

## 2.3 Block C — Market Clock Retrieval and Time Normalization

### Overview
This block checks the current market clock status and normalizes date/time information before making the open/closed decision. It is the validation gate before any stock analysis begins.

### Nodes Involved
- HTTP (Get Market Clock Response)
- Format Date & Time
- If (Check Market is Open)

### Node Details

#### HTTP (Get Market Clock Response)
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves market clock information from an external API.
- **Configuration choices:**  
  Likely configured to call a market-status endpoint such as Alpaca’s `/v2/clock`. The JSON does not expose method, URL, headers, or authentication details.
- **Key expressions or variables used:**  
  May use credentials or fixed headers such as API key/secret.
- **Input and output connections:**  
  - Input: `Schedule Trigger (Every Friday)`
  - Output: `Format Date & Time`
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - API authentication failure.
  - Non-200 response not handled.
  - Rate limiting.
  - Unexpected schema causing downstream date formatting to fail.
- **Sub-workflow reference:**  
  None.

#### Format Date & Time
- **Type and technical role:** `n8n-nodes-base.dateTime`  
  Reformats or converts market-clock timestamps for downstream conditional logic.
- **Configuration choices:**  
  Specific formatting patterns are not shown. This node likely parses the clock response and standardizes date/time fields such as current timestamp, opening time, or weekday.
- **Key expressions or variables used:**  
  Probably references fields from the market clock API response, such as `timestamp`, `next_open`, `next_close`, or a market-open boolean.
- **Input and output connections:**  
  - Input: `HTTP (Get Market Clock Response)`
  - Output: `If (Check Market is Open)`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Invalid date string.
  - Missing source field in API response.
  - Timezone conversion errors.
- **Sub-workflow reference:**  
  None.

#### If (Check Market is Open)
- **Type and technical role:** `n8n-nodes-base.if`  
  Decides whether to proceed with stock analysis or send alerts.
- **Configuration choices:**  
  Exact rule not shown. Based on routing:
  - **True branch** -> `Stock ticker symbols`
  - **False branch** -> `Rapiwa (Send Whatsapp Aleart Message)` and `Send Telegram Aleart Message`
- **Key expressions or variables used:**  
  Likely checks a boolean like `is_open` or equivalent from the market clock response.
- **Input and output connections:**  
  - Input: `Format Date & Time`
  - Outputs:
    - True: `Stock ticker symbols`
    - False: `Rapiwa (Send Whatsapp Aleart Message)`, `Send Telegram Aleart Message`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If API returns market closed for holidays or half-days, the workflow correctly diverts but may surprise users.
  - Expression failure if field names change.
  - Parallel false-branch notifications may both fail independently.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block D — Alerting When Market Is Closed

### Overview
If the market is closed, this block sends warning/alert notifications instead of generating analysis. It uses both WhatsApp and Telegram for redundancy.

### Nodes Involved
- Rapiwa (Send Whatsapp Aleart Message)
- Send Telegram Aleart Message

### Node Details

#### Rapiwa (Send Whatsapp Aleart Message)
- **Type and technical role:** `n8n-nodes-rapiwa.rapiwa`  
  Third-party WhatsApp messaging node used to send a market-closed alert.
- **Configuration choices:**  
  Exact operation and message payload are not visible. It is presumably configured to send a WhatsApp text message to a predefined number or contact.
- **Key expressions or variables used:**  
  Likely includes dynamic text based on market state or execution context.
- **Input and output connections:**  
  - Input: `If (Check Market is Open)` false branch
  - Output: none
- **Version-specific requirements:**  
  Type version `1`.  
  Requires the Rapiwa community node/package to be installed in the n8n instance.
- **Edge cases or potential failure types:**  
  - Community node missing in another environment.
  - WhatsApp provider auth/token errors.
  - Invalid recipient number format.
  - Message template restrictions depending on provider policy.
- **Sub-workflow reference:**  
  None.

#### Send Telegram Aleart Message
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message when market conditions prevent the analysis run.
- **Configuration choices:**  
  Likely posts a simple alert to a fixed chat or admin account.
- **Key expressions or variables used:**  
  Could include market status text from prior nodes.
- **Input and output connections:**  
  - Input: `If (Check Market is Open)` false branch
  - Output: none
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Invalid bot credentials or chat permissions.
  - Rate limiting or blocked bot.
  - Dynamic text rendering errors if expressions reference missing fields.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Block E — Stock Universe Definition and Market Data Retrieval

### Overview
This block defines which stock symbols to analyze and then fetches market data for them. It is the data acquisition stage for the technical analysis pipeline.

### Nodes Involved
- Stock ticker symbols
- Fetch Stock Data (base on symble)

### Node Details

#### Stock ticker symbols
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates or overwrites fields containing the list of ticker symbols to analyze.
- **Configuration choices:**  
  The exact symbols are not shown in the JSON. This node likely creates one or more fields such as a symbol array, comma-separated list, or single symbol records.
- **Key expressions or variables used:**  
  Could use static values only.
- **Input and output connections:**  
  - Input: `If (Check Market is Open)` true branch
  - Output: `Fetch Stock Data (base on symble)`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Ticker formatting mismatch with the target API.
  - If multiple symbols are packed into one field, the downstream HTTP request must handle iteration or batching correctly.
- **Sub-workflow reference:**  
  None.

#### Fetch Stock Data (base on symble)
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches stock price/history data for the selected symbols.
- **Configuration choices:**  
  The name strongly suggests dynamic API calls per symbol. The exact endpoint, authentication, query parameters, and pagination handling are not visible.
- **Key expressions or variables used:**  
  Likely references the symbol(s) from `Stock ticker symbols`.
- **Input and output connections:**  
  - Input: `Stock ticker symbols`
  - Output: `Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid or delisted ticker.
  - API pagination not handled for long lookback periods.
  - Rate limits if fetching many symbols individually.
  - Missing historical candle data causing indicator calculation errors.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Block F — Indicator Calculation and Signal Generation

### Overview
This block computes technical indicators from the fetched stock data and transforms raw market data into actionable signal labels such as Buy, Sell, or Hold.

### Nodes Involved
- Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)

### Node Details

#### Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript execution node used to calculate RSI, MACD, and generate trading signals.
- **Configuration choices:**  
  Exact code is not visible in the export, but the title indicates:
  - RSI calculation
  - MACD calculation
  - signal derivation logic
- **Key expressions or variables used:**  
  Likely reads price arrays/candle series from the HTTP response. It may output structured summary objects per symbol.
- **Input and output connections:**  
  - Input: `Fetch Stock Data (base on symble)`
  - Output: `Basic LLM Chain`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Insufficient historical data to compute RSI/MACD.
  - Division-by-zero or NaN values.
  - Unexpected API payload shape.
  - Multi-item handling errors if several symbols arrive as separate items.
- **Sub-workflow reference:**  
  None.

**Implementation concern:**  
This is a critical logic node. Reproducing the workflow accurately requires access to the JavaScript formula implementation and output schema, which are not present in the provided JSON excerpt.

---

## 2.7 Block G — AI-Based Market Summary

### Overview
This block converts the raw technical outputs into a concise narrative summary using an LLM chain backed by xAI Grok. It is responsible for transforming indicator data into readable commentary.

### Nodes Involved
- Basic LLM Chain
- xAI Grok Chat Model

### Node Details

#### Basic LLM Chain
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  LangChain-based orchestration node that sends prompt/context to a connected language model and returns generated text.
- **Configuration choices:**  
  No parameters are visible in the JSON, which may mean:
  - defaults are used,
  - or prompt fields were left empty in the export,
  - or hidden node internals are omitted from this summary.
- **Key expressions or variables used:**  
  Likely uses the output of the Code node as prompt context.  
  It also receives a model connection from `xAI Grok Chat Model`.
- **Input and output connections:**  
  - Main input: `Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)`
  - AI model input: `xAI Grok Chat Model`
  - Main output: `Send Admin (Telegram market summary)`, `Send Channel (Telegram market summary)`
- **Version-specific requirements:**  
  Type version `1.8`.  
  Requires n8n LangChain-compatible AI nodes to be available.
- **Edge cases or potential failure types:**  
  - Missing prompt instructions can produce poor summaries.
  - LLM token/context limits if too many symbols or too much historical data are passed.
  - Output variability may break downstream formatting if structured output is expected.
- **Sub-workflow reference:**  
  None.

#### xAI Grok Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatXAiGrok`  
  Provides the LLM backend used by the chain.
- **Configuration choices:**  
  The exact Grok model, temperature, and API credential selection are not visible.
- **Key expressions or variables used:**  
  None visible directly; this node serves as the connected language model.
- **Input and output connections:**  
  - Output (AI language model): `Basic LLM Chain`
- **Version-specific requirements:**  
  Type version `1`.  
  Requires valid xAI credentials and a compatible n8n version with this node installed.
- **Edge cases or potential failure types:**  
  - xAI API authentication issues.
  - Region/account access limitations.
  - Request failures due to quota or unavailable model.
- **Sub-workflow reference:**  
  None.

---

## 2.8 Block H — Summary Distribution

### Overview
This final block sends the AI-generated market summary to Telegram recipients. One message goes to an admin destination and one to a broader Telegram channel.

### Nodes Involved
- Send Admin (Telegram market summary)
- Send Channel (Telegram market summary)

### Node Details

#### Send Admin (Telegram market summary)
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the final market summary to an admin chat.
- **Configuration choices:**  
  Presumably configured with a fixed admin chat ID and a message body based on the LLM output.
- **Key expressions or variables used:**  
  Likely references the generated text from `Basic LLM Chain`.
- **Input and output connections:**  
  - Input: `Basic LLM Chain`
  - Output: none
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Incorrect admin chat ID.
  - Telegram bot lacks permission to message the user.
  - If LLM output is too long, Telegram message length limits may require truncation.
- **Sub-workflow reference:**  
  None.

#### Send Channel (Telegram market summary)
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the same or similar summary to a Telegram channel.
- **Configuration choices:**  
  Presumably configured with a Telegram channel ID or username and message text sourced from the LLM output.
- **Key expressions or variables used:**  
  Likely references the generated text from `Basic LLM Chain`.
- **Input and output connections:**  
  - Input: `Basic LLM Chain`
  - Output: none
- **Version-specific requirements:**  
  Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Bot must be an admin in the Telegram channel.
  - Markdown/HTML formatting may break if message parsing mode is enabled.
  - Channel posts can fail if the wrong identifier format is used.
- **Sub-workflow reference:**  
  None.

---

## 2.9 Block I — Sticky Notes / Canvas Annotation Nodes

### Overview
These nodes are purely visual annotations in the n8n canvas. In this export, all sticky notes appear to contain empty content.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7

### Node Details

All sticky-note nodes:
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Each note has empty `content`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives Telegram bot events and starts the manual path |  | If (Check Friday market summary) |  |
| If (Check Friday market summary) | n8n-nodes-base.if | Checks whether the Telegram-driven request should proceed to Friday summary logic | Telegram Trigger | HTTP (Get Market Clock Response)1; Send Telegram Again Aleart Message |  |
| Send Telegram Again Aleart Message | n8n-nodes-base.telegram | Sends fallback Telegram alert when Friday-summary condition fails | If (Check Friday market summary) |  |  |
| Schedule Trigger (Every Friday) | n8n-nodes-base.scheduleTrigger | Starts the automated weekly Friday execution |  | HTTP (Get Market Clock Response) |  |
| HTTP (Get Market Clock Response) | n8n-nodes-base.httpRequest | Fetches market clock status for the scheduled path | Schedule Trigger (Every Friday) | Format Date & Time |  |
| HTTP (Get Market Clock Response)1 | n8n-nodes-base.httpRequest | Fetches market clock status for the Telegram/manual path | If (Check Friday market summary) |  |  |
| Format Date & Time | n8n-nodes-base.dateTime | Normalizes market clock date/time fields | HTTP (Get Market Clock Response) | If (Check Market is Open) |  |
| If (Check Market is Open) | n8n-nodes-base.if | Routes execution based on whether the market is open | Format Date & Time | Stock ticker symbols; Rapiwa (Send Whatsapp Aleart Message); Send Telegram Aleart Message |  |
| Rapiwa (Send Whatsapp Aleart Message) | n8n-nodes-rapiwa.rapiwa | Sends WhatsApp alert when the market is closed | If (Check Market is Open) |  |  |
| Send Telegram Aleart Message | n8n-nodes-base.telegram | Sends Telegram alert when the market is closed | If (Check Market is Open) |  |  |
| Stock ticker symbols | n8n-nodes-base.set | Defines the stock symbols to analyze | If (Check Market is Open) | Fetch Stock Data (base on symble) |  |
| Fetch Stock Data (base on symble) | n8n-nodes-base.httpRequest | Retrieves stock market data for the selected symbols | Stock ticker symbols | Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals) |  |
| Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals) | n8n-nodes-base.code | Calculates RSI, MACD, and trading signals | Fetch Stock Data (base on symble) | Basic LLM Chain |  |
| xAI Grok Chat Model | @n8n/n8n-nodes-langchain.lmChatXAiGrok | Provides the LLM backend for summary generation |  | Basic LLM Chain |  |
| Basic LLM Chain | @n8n/n8n-nodes-langchain.chainLlm | Converts technical outputs into a narrative market summary | Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals); xAI Grok Chat Model | Send Admin (Telegram market summary); Send Channel (Telegram market summary) |  |
| Send Admin (Telegram market summary) | n8n-nodes-base.telegram | Sends final summary to admin via Telegram | Basic LLM Chain |  |  |
| Send Channel (Telegram market summary) | n8n-nodes-base.telegram | Sends final summary to Telegram channel | Basic LLM Chain |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual annotation node |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence based on the available JSON. Because many node parameters are empty in the export, some settings must be inferred and completed manually.

## 4.1 Prepare credentials and prerequisites

1. **Create Telegram Bot credentials**
   - In n8n, configure Telegram credentials for:
     - `Telegram Trigger`
     - `Send Telegram Again Aleart Message`
     - `Send Telegram Aleart Message`
     - `Send Admin (Telegram market summary)`
     - `Send Channel (Telegram market summary)`
   - You will need:
     - bot token
     - target chat IDs or channel identifiers

2. **Create xAI credentials**
   - Add credentials for `xAI Grok Chat Model`.
   - Confirm the target model is available to your xAI account.

3. **Create stock-market API credentials**
   - The title mentions Alpaca, so the HTTP nodes are likely intended to use Alpaca API credentials.
   - Prepare:
     - API key
     - secret key
     - base URL for market clock and historical stock data endpoints

4. **Install the Rapiwa node if needed**
   - `n8n-nodes-rapiwa.rapiwa` is a community node.
   - Ensure your n8n environment supports and installs community nodes.
   - Add the required Rapiwa credentials for WhatsApp sending.

5. **Ensure AI nodes are supported**
   - The workflow uses LangChain nodes:
     - `Basic LLM Chain`
     - `xAI Grok Chat Model`
   - Use an n8n version that supports these nodes.

---

## 4.2 Build the scheduled execution path

6. **Add a Schedule Trigger**
   - Node type: `Schedule Trigger`
   - Name it: `Schedule Trigger (Every Friday)`
   - Configure it to run every Friday at your desired time.
   - Choose the correct timezone.

7. **Add an HTTP Request node for market clock**
   - Node type: `HTTP Request`
   - Name it: `HTTP (Get Market Clock Response)`
   - Configure:
     - Method: typically `GET`
     - URL: your market clock endpoint, e.g. Alpaca clock endpoint
     - Authentication: add required headers or credentials
   - Connect:
     - `Schedule Trigger (Every Friday)` -> `HTTP (Get Market Clock Response)`

8. **Add a Date & Time node**
   - Node type: `Date & Time`
   - Name it: `Format Date & Time`
   - Configure it to parse or reformat the timestamp returned by the market clock API.
   - Typical goal:
     - normalize current market timestamp
     - extract weekday or open/close-related fields if needed
   - Connect:
     - `HTTP (Get Market Clock Response)` -> `Format Date & Time`

9. **Add an IF node for market-open check**
   - Node type: `IF`
   - Name it: `If (Check Market is Open)`
   - Configure condition using the market clock response:
     - for example, check whether a field like `is_open` equals `true`
   - Connect:
     - `Format Date & Time` -> `If (Check Market is Open)`

---

## 4.3 Build the closed-market notification path

10. **Add a Rapiwa node**
    - Node type: `Rapiwa`
    - Name it: `Rapiwa (Send Whatsapp Aleart Message)`
    - Configure it to send a WhatsApp message to your target number.
    - Suggested message:
      - “Market is closed. Weekly stock summary was not generated.”
    - Connect from the **false** output of `If (Check Market is Open)`.

11. **Add a Telegram node for alerting**
    - Node type: `Telegram`
    - Name it: `Send Telegram Aleart Message`
    - Configure operation: send message
    - Set the target chat ID
    - Suggested message:
      - “Market is closed. Weekly stock summary was skipped.”
    - Connect from the **false** output of `If (Check Market is Open)`.

---

## 4.4 Build the stock-analysis path

12. **Add a Set node for tickers**
    - Node type: `Set`
    - Name it: `Stock ticker symbols`
    - Add fields representing the ticker list.
    - Example approaches:
      - a single field `symbols` = `AAPL,TSLA,NVDA,MSFT`
      - or one item per symbol
    - Match the structure expected by the next HTTP node.
    - Connect from the **true** output of `If (Check Market is Open)`.

13. **Add an HTTP Request node for stock data**
    - Node type: `HTTP Request`
    - Name it: `Fetch Stock Data (base on symble)`
    - Configure:
      - Method: usually `GET`
      - URL: historical bars or market-data endpoint
      - Query params: symbol, timeframe, lookback window, limit
      - Authentication: API headers/credentials
    - Use expressions to reference the symbol field from `Stock ticker symbols`.
    - Connect:
      - `Stock ticker symbols` -> `Fetch Stock Data (base on symble)`

14. **Add a Code node**
    - Node type: `Code`
    - Name it: `Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)`
    - Implement JavaScript logic to:
      - extract closing prices
      - calculate RSI
      - calculate MACD
      - assign signal labels such as Buy/Sell/Hold
    - Output a concise structured payload per symbol, for example:
      - symbol
      - latest price
      - RSI
      - MACD
      - signal
      - short interpretation
    - Connect:
      - `Fetch Stock Data (base on symble)` -> `Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)`

**Important:**  
Because the original code is not included, you must recreate the formulas yourself. A robust implementation should:
- validate minimum bar count,
- handle missing candle data,
- avoid NaN output,
- return clean JSON objects.

---

## 4.5 Build the AI summarization layer

15. **Add an xAI Grok Chat Model node**
    - Node type: `xAI Grok Chat Model`
    - Name it: `xAI Grok Chat Model`
    - Select your xAI credentials.
    - Choose the desired Grok model.
    - Optional:
      - set temperature low for consistent summaries.

16. **Add a Basic LLM Chain node**
    - Node type: `Basic LLM Chain`
    - Name it: `Basic LLM Chain`
    - Configure the prompt to summarize the technical indicators.
    - Example prompt logic:
      - instruct the model to summarize RSI/MACD results,
      - mention strongest bullish and bearish signals,
      - keep wording concise for Telegram,
      - avoid financial advice language unless intended.
    - Feed the Code node output into the prompt.
    - Connect:
      - `Code (Calculate RSI & MACD and Generate Buy/Sell/Hold Signals)` -> `Basic LLM Chain`
      - `xAI Grok Chat Model` -> `Basic LLM Chain` via AI model connection

17. **Test prompt formatting**
    - Ensure the prompt references the incoming JSON correctly.
    - If multiple symbols are present, either aggregate them before the chain or instruct the chain to iterate through the provided list.

---

## 4.6 Build Telegram summary delivery

18. **Add a Telegram node for admin delivery**
    - Node type: `Telegram`
    - Name it: `Send Admin (Telegram market summary)`
    - Configure:
      - operation: send message
      - chat ID: admin/user destination
      - message body: use the LLM output text
    - Connect:
      - `Basic LLM Chain` -> `Send Admin (Telegram market summary)`

19. **Add a Telegram node for channel delivery**
    - Node type: `Telegram`
    - Name it: `Send Channel (Telegram market summary)`
    - Configure:
      - operation: send message
      - channel ID or username
      - message body: use the same or adapted LLM output
    - Connect:
      - `Basic LLM Chain` -> `Send Channel (Telegram market summary)`

---

## 4.7 Build the Telegram-triggered path

20. **Add a Telegram Trigger node**
    - Node type: `Telegram Trigger`
    - Name it: `Telegram Trigger`
    - Configure your Telegram bot credentials.
    - Optionally filter to a command such as `/summary`.

21. **Add an IF node**
    - Node type: `IF`
    - Name it: `If (Check Friday market summary)`
    - Configure a condition that determines whether the incoming Telegram request should proceed.
    - Possible logic:
      - message text equals `/friday_summary`
      - current day is Friday
      - sender is authorized
    - Connect:
      - `Telegram Trigger` -> `If (Check Friday market summary)`

22. **Add a Telegram fallback node**
    - Node type: `Telegram`
    - Name it: `Send Telegram Again Aleart Message`
    - Configure a response for invalid requests.
    - Example:
      - “This summary is only available on Friday.”
    - Connect from the **false** output of `If (Check Friday market summary)`.

23. **Add another market-clock HTTP node**
    - Node type: `HTTP Request`
    - Name it: `HTTP (Get Market Clock Response)1`
    - Configure similarly to the scheduled-path market-clock node.
    - Connect from the **true** output of `If (Check Friday market summary)`.

24. **Complete the manual path**
    - In the provided JSON, this node is not connected further.
    - To make the manual path functional, connect `HTTP (Get Market Clock Response)1` into:
      - either `Format Date & Time`
      - or a duplicated version of the downstream market-open logic
    - The cleanest rebuild is:
      - `HTTP (Get Market Clock Response)1` -> `Format Date & Time`
    - If you do this, ensure the node can accept input from both market-clock nodes.

---

## 4.8 Optional sticky notes

25. **Add sticky notes if desired**
    - The export contains 8 sticky-note nodes, all empty.
    - These are optional and have no effect on execution.

---

## 4.9 Recommended validation tests

26. **Test the market-closed branch**
    - Temporarily simulate `is_open = false`
    - Confirm both WhatsApp and Telegram alerts are sent.

27. **Test the market-open branch**
    - Simulate or run during open hours
    - Confirm:
      - stock data fetch succeeds,
      - code node returns valid indicators,
      - Grok returns summary text,
      - both Telegram destinations receive it.

28. **Test Telegram manual trigger**
    - Send both valid and invalid commands
    - Confirm the IF node behaves as expected.

29. **Check message size and formatting**
    - Telegram has length and formatting constraints.
    - If needed, shorten the LLM output or split long summaries.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow title in export: “Automated Stock Market Analysis and Notification Report” | Internal workflow name |
| User-provided title: “Summarize stock market signals with Alpaca, xAI Grok, Telegram and WhatsApp” | External title / catalog label |
| The provided JSON does not expose actual HTTP endpoints, prompts, ticker values, message templates, or code-node formulas | Important limitation when reproducing |
| The Telegram-triggered path appears incomplete because `HTTP (Get Market Clock Response)1` has no downstream connection | Design issue to correct before production use |
| The workflow uses a community node for WhatsApp (`n8n-nodes-rapiwa.rapiwa`) | Requires installation support in the target n8n instance |
| All sticky notes in the export are empty | No annotation content to preserve |