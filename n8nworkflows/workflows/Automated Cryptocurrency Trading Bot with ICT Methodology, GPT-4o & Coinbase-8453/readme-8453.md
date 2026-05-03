Automated Cryptocurrency Trading Bot with ICT Methodology, GPT-4o & Coinbase

https://n8nworkflows.xyz/workflows/automated-cryptocurrency-trading-bot-with-ict-methodology--gpt-4o---coinbase-8453


# Automated Cryptocurrency Trading Bot with ICT Methodology, GPT-4o & Coinbase

---

### 1. Workflow Overview

This workflow is an **Automated Cryptocurrency Trading Bot** that leverages advanced **ICT (Inner Circle Trader) 2025 Smart Money Concepts**, GPT-4o AI analysis, and real-time Coinbase market data to make informed trading decisions. It processes incoming trading signals (primarily via Telegram), validates and enriches them with market and session data, analyzes the signals using AI trained on ICT methodology, filters signals based on quality and session alignment, executes trades on Coinbase, records trades and rejections in Notion databases, and sends formatted trading alerts back to Telegram for human monitoring.

**Target Use Cases:**  
- Automated crypto trading with ICT methodology and smart money concepts.  
- Real-time signal processing and trade execution via Coinbase Advanced API.  
- AI-powered signal quality and risk assessment using GPT-4o.  
- Session-based trade filtering aligned with ICT Kill Zones (market time zones with different volatility and trading characteristics).  
- Logging and auditing trades and rejected signals in Notion databases.  
- Sending professional Telegram notifications for executed trades.

**Logical Blocks:**  
- **1.1 Input Reception and Signal Extraction:** Receives Telegram trading signals, extracts and formats key ICT signal data.  
- **1.2 Session Validation and Market Data Retrieval:** Validates the current trading session (Kill Zone) and fetches Coinbase market data for the symbol.  
- **1.3 AI Signal Analysis:** Uses GPT-4o to analyze the enriched signal data according to ICT 2025 methodology.  
- **1.4 AI Analysis Parsing and Enrichment:** Parses AI JSON response, applies fallback logic, and enriches with session and market metadata.  
- **1.5 Signal Quality Filtering:** Filters signals for quality, confidence, session alignment, and ICT structure alignment.  
- **1.6 Trade Execution:** Executes the trade on Coinbase if the signal passes filters.  
- **1.7 Trade and Rejection Logging:** Logs executed trades or rejected signals to Notion databases.  
- **1.8 Notification and Reporting:** Generates and sends formatted Telegram alerts, and responds to webhook calls.  
- **1.9 Miscellaneous:** Auxiliary nodes for HTTP requests and Telegram chat retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Signal Extraction

**Overview:**  
Receives incoming trading signals from Telegram, extracts key ICT trading parameters such as RSI, MACD, volume, symbol, action (buy/sell/hold), price, quantity, and session time, defaulting values where missing.

**Nodes Involved:**  
- ICT Telegram Signal Trigger  
- Extract ICT Signal Data

**Node Details:**

- **ICT Telegram Signal Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for Telegram messages containing trading signals.  
  - Configuration: Triggers on any new message update; webhook tied to Telegram channel/group.  
  - Inputs: External Telegram messages.  
  - Outputs: JSON with raw message data.  
  - Edge Cases: Missing or malformed Telegram messages; delayed webhook delivery.

- **Extract ICT Signal Data**  
  - Type: Set Node  
  - Role: Extracts and formats trading signal data from raw input JSON.  
  - Configuration:  
    - Extracts `rsi`, `macd`, `volume` or null if missing.  
    - Extracts `symbol` from JSON or matches pattern `<3-6 uppercase letters>-<3-6 uppercase letters>` in text; defaults to "BTC-USD".  
    - Infers `action` from text keywords or defaults to "HOLD".  
    - Extracts `price`, defaults to "0".  
    - Sets current timestamp and session time.  
    - Determines `trading_session` based on current hour to identify ICT kill zones.  
  - Expressions: Uses JavaScript expressions with fallback logic.  
  - Inputs: From Telegram trigger node.  
  - Outputs: Cleaned and structured signal JSON.  
  - Edge Cases: Unrecognized symbols, missing fields, incorrect text parsing.

---

#### 1.2 Session Validation and Market Data Retrieval

**Overview:**  
Validates which ICT Kill Zone (trading session) is currently active based on UTC time, annotates the signal with session strength and trading permission. Fetches live market product data from Coinbase for the target symbol.

**Nodes Involved:**  
- ICT Session Validator  
- Get Coinbase Market Data

**Node Details:**

- **ICT Session Validator**  
  - Type: Code Node (JavaScript)  
  - Role: Determines current ICT Kill Zone (Asian, London, New York, London Close, or Off Hours) with enhanced priority and characteristics.  
  - Configuration:  
    - Uses UTC time to compare against kill zone time ranges.  
    - Assigns priority (HIGH/MEDIUM/LOW) and session strength scores.  
    - Outputs enriched session validation data appended to input signal JSON.  
  - Inputs: Cleaned signal JSON.  
  - Outputs: Signal JSON enriched with `session_validation` object.  
  - Edge Cases: Timezone mismatches, server time skew, undefined session intervals.

- **Get Coinbase Market Data**  
  - Type: HTTP Request  
  - Role: Retrieves current product details from Coinbase API for the symbol.  
  - Configuration:  
    - URL dynamically set to `https://api.coinbase.com/api/v3/brokerage/products/{symbol}`.  
    - Uses Coinbase Advanced API credentials.  
    - 10-second timeout.  
  - Inputs: Enriched signal with session data.  
  - Outputs: Coinbase product JSON appended.  
  - Edge Cases: Network timeouts, API rate limits, invalid symbol errors, authentication failures.

---

#### 1.3 AI Signal Analysis

**Overview:**  
Sends the enriched signal data (including session validation and coinbase market data) to a GPT-4o AI model that applies ICT 2025 methodology and Smart Money Concepts to generate a detailed trading signal analysis in a strict JSON format.

**Nodes Involved:**  
- ICT AI Analysis

**Node Details:**

- **ICT AI Analysis**  
  - Type: OpenAI (Langchain)  
  - Role: Uses GPT-4o to analyze the trading signal in the context of ICT 2025 rules.  
  - Configuration:  
    - Model: GPT-4o  
    - Max tokens: 1000  
    - Temperature: 0.3 (low randomness for consistent output)  
    - User prompt includes: Input data JSON, instructions to return structured JSON with fields like signal_quality, confidence_score, risk_level, recommendation, reasoning, stop_loss, take_profit, and detailed ICT analysis subfields.  
  - Inputs: Signal JSON enriched with session and market data.  
  - Outputs: AI response JSON with ICT analysis.  
  - Edge Cases: AI timeout, malformed response, rate limits, prompt failures.

---

#### 1.4 AI Analysis Parsing and Enrichment

**Overview:**  
Parses the AI response to extract the JSON analysis, applies robust fallback logic if parsing fails, enriches the final signal with AI insights, market data, and metadata.

**Nodes Involved:**  
- Parse ICT AI Analysis

**Node Details:**

- **Parse ICT AI Analysis**  
  - Type: Code Node (JavaScript)  
  - Role: Parses AI JSON response robustly; handles malformed input gracefully with fallback ICT-informed default analysis.  
  - Configuration:  
    - Attempts multiple ways to extract AI text (message.content, content, text).  
    - Parses JSON directly or from markdown code blocks.  
    - On failure, applies fallback analysis using session data and input signal defaults.  
    - Enriches AI analysis with current kill zone, zone characteristics, and GMT time.  
    - Attaches Coinbase data and metadata (processing time, workflow version, AI model).  
  - Inputs: AI analysis node output and Coinbase market data.  
  - Outputs: Fully enriched signal JSON with `ai_analysis` and metadata.  
  - Edge Cases: Parsing errors, missing AI output, unexpected AI formats, runtime JS errors.

---

#### 1.5 Signal Quality Filtering

**Overview:**  
Applies filtering criteria to decide if the signal quality, confidence, session alignment, and ICT session structure justify trade execution or rejection.

**Nodes Involved:**  
- ICT Quality & Session Filter

**Node Details:**

- **ICT Quality & Session Filter**  
  - Type: If Node  
  - Role: Evaluates multiple conditions combined with AND logic:  
    - Signal quality not "LOW"  
    - Confidence score >= 60  
    - Session trading allowed (kill zone active)  
    - ICT session alignment true  
  - Inputs: Enriched signal JSON with AI analysis.  
  - Outputs:  
    - True branch: Passes filter (trade execution).  
    - False branch: Fails filter (signal rejection).  
  - Edge Cases: Missing AI analysis fields, edge confidence values, ambiguous session flags.

---

#### 1.6 Trade Execution

**Overview:**  
If the signal passes filtering, places a market IOC (Immediate or Cancel) order on Coinbase with specified symbol, action, and quantity.

**Nodes Involved:**  
- Execute ICT Trade

**Node Details:**

- **Execute ICT Trade**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Coinbase brokerage orders endpoint to place the trade.  
  - Configuration:  
    - URL: https://api.coinbase.com/api/v3/brokerage/orders  
    - Method: POST  
    - Body includes product_id (symbol), side (action lowercased), order configuration as market IOC with quote size quantity.  
    - Headers: Content-Type application/json  
    - Authentication: Coinbase Advanced API credentials  
    - Timeout: 30 seconds  
  - Inputs: Filter-passed signal JSON.  
  - Outputs: Coinbase trade execution response.  
  - Edge Cases: API errors, insufficient funds, invalid order parameters, network issues.

---

#### 1.7 Trade and Rejection Logging

**Overview:**  
Logs executed trades or rejected signals into separate Notion databases for audit and tracking.

**Nodes Involved:**  
- Create ICT Trading Record  
- Log ICT Rejected Signal

**Node Details:**

- **Create ICT Trading Record**  
  - Type: Notion Node  
  - Role: Creates a new page in a Notion database for executed trades.  
  - Configuration:  
    - Database ID from workflow variable `NOTION_TRADING_DB_ID`  
    - Maps signal fields: Symbol (title), Action (select), Confidence (number), Kill Zone (rich text), Price (number), Timestamp (date), Session Strength (number), Risk Level (select)  
  - Inputs: Coinbase trade execution success output.  
  - Outputs: Notion page creation response.  
  - Edge Cases: Notion API rate limits, permission errors, missing fields.

- **Log ICT Rejected Signal**  
  - Type: Notion Node  
  - Role: Creates a new page in a separate Notion database for rejected signals.  
  - Configuration:  
    - Database ID from workflow variable `NOTION_REJECTED_DB_ID`  
    - Maps fields: Symbol (title), Rejection Reason (rich text), Confidence Score (number), Session (rich text), Timestamp (date)  
  - Inputs: Filter-failed signals.  
  - Outputs: Notion page creation response.  
  - Edge Cases: Same as above.

---

#### 1.8 Notification and Reporting

**Overview:**  
Generates professional Telegram notification messages based on ICT analysis results and sends them to a designated Telegram chat. Also responds to the webhook caller with JSON summary.

**Nodes Involved:**  
- Generate ICT Notification  
- Send ICT Telegram Alert  
- ICT Webhook Response

**Node Details:**

- **Generate ICT Notification**  
  - Type: OpenAI (Langchain)  
  - Role: Creates a concise, formatted Telegram message with emojis, key ICT points, confidence, session info, and risk management details.  
  - Configuration:  
    - Model: GPT-4o  
    - Max tokens: 500  
    - Temperature: 0.7 (more creative formatting)  
    - Prompt includes full signal JSON for context.  
  - Inputs: Executed trade enriched JSON.  
  - Outputs: Text message content.  
  - Edge Cases: AI formatting errors, output truncation.

- **Send ICT Telegram Alert**  
  - Type: Telegram Node  
  - Role: Sends generated message to Telegram chat using Markdown format.  
  - Configuration:  
    - Chat ID from variable `TELEGRAM_CHAT_ID`  
    - Parse mode: Markdown  
    - Disables web page preview  
    - Fallback message if AI notification missing.  
  - Inputs: Generated notification text.  
  - Outputs: Telegram API response.  
  - Edge Cases: Chat ID misconfiguration, Telegram API rate limits.

- **ICT Webhook Response**  
  - Type: Respond to Webhook  
  - Role: Sends a JSON response back to the webhook caller with processed signal summary including status, symbol, action, kill zone, confidence, session strength, and ICT factors.  
  - Inputs: Final enriched signal JSON after processing.  
  - Outputs: HTTP JSON response.  
  - Edge Cases: Response formatting failures.

---

#### 1.9 Miscellaneous

**Overview:**  
Auxiliary nodes to support HTTP communication and Telegram chat metadata retrieval.

**Nodes Involved:**  
- HTTP Request  
- Get a chat

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Sends POST requests with signal data and workflow metadata to an external webhook URL (configurable via variable).  
  - Configuration:  
    - URL from variable `WEBHOOK_URL` or fallback webhook.site URL  
    - Sends signal data JSON, timestamp, and workflow ID.  
    - Content-Type: application/json  
  - Inputs: Output from ICT Webhook Response node.  
  - Outputs: HTTP response.  
  - Edge Cases: Network errors, webhook downtime.

- **Get a chat**  
  - Type: Telegram Node  
  - Role: Retrieves Telegram chat metadata for the configured chat ID.  
  - Inputs: Output from HTTP Request node.  
  - Outputs: Telegram chat metadata JSON.  
  - Edge Cases: Invalid chat ID, Telegram API errors.

---

### 3. Summary Table

| Node Name                  | Node Type                       | Functional Role                      | Input Node(s)                | Output Node(s)                  | Sticky Note                                             |
|----------------------------|--------------------------------|------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------|
| ICT Telegram Signal Trigger| Telegram Trigger               | Receive Telegram trading signals   | External (Telegram)          | Extract ICT Signal Data         |                                                         |
| Extract ICT Signal Data     | Set                            | Extract and format signal data     | ICT Telegram Signal Trigger | ICT Session Validator           |                                                         |
| ICT Session Validator      | Code                           | Validate current ICT Kill Zone     | Extract ICT Signal Data      | Get Coinbase Market Data        |                                                         |
| Get Coinbase Market Data   | HTTP Request                   | Fetch Coinbase product data        | ICT Session Validator       | ICT AI Analysis                |                                                         |
| ICT AI Analysis            | OpenAI (Langchain)             | AI analysis of trading signal      | Get Coinbase Market Data    | Parse ICT AI Analysis           |                                                         |
| Parse ICT AI Analysis      | Code                           | Parse and enrich AI JSON response  | ICT AI Analysis             | ICT Quality & Session Filter    |                                                         |
| ICT Quality & Session Filter| If                            | Filter signals by quality and session | Parse ICT AI Analysis    | Execute ICT Trade (true branch) <br> Log ICT Rejected Signal (false branch) |                                                         |
| Execute ICT Trade          | HTTP Request                   | Place trade order on Coinbase      | ICT Quality & Session Filter| Create ICT Trading Record       |                                                         |
| Create ICT Trading Record  | Notion                         | Log executed trade                 | Execute ICT Trade           | Generate ICT Notification       |                                                         |
| Log ICT Rejected Signal    | Notion                         | Log rejected signals                | ICT Quality & Session Filter| ICT Webhook Response            |                                                         |
| Generate ICT Notification  | OpenAI (Langchain)             | Create Telegram notification text | Create ICT Trading Record   | Send ICT Telegram Alert         |                                                         |
| Send ICT Telegram Alert    | Telegram                       | Send notification message          | Generate ICT Notification   | ICT Webhook Response            |                                                         |
| ICT Webhook Response       | Respond to Webhook             | Respond to webhook caller           | Send ICT Telegram Alert <br> Log ICT Rejected Signal | HTTP Request                   |                                                         |
| HTTP Request              | HTTP Request                   | Send signal data to external webhook | ICT Webhook Response       | Get a chat                     |                                                         |
| Get a chat                | Telegram                       | Retrieve Telegram chat metadata     | HTTP Request                | —                              |                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook with Telegram bot token and subscribe to "message" updates.  
   - Position: Leftmost node, entry point.

2. **Add Set Node "Extract ICT Signal Data"**  
   - Extract RSI, MACD, volume from incoming data or set to null.  
   - Extract or infer symbol from JSON or text with regex pattern `<3-6 uppercase letters>-<3-6 uppercase letters>`, default "BTC-USD".  
   - Infer action from text containing "buy", "sell", or default "HOLD".  
   - Set price default to "0".  
   - Set current timestamp and session time (`HH:mm`).  
   - Determine `trading_session` based on current hour using conditions for Asian, London, NY, London Close Kill Zones or Off Hours.  
   - Connect Telegram Trigger output to this node input.

3. **Add Code Node "ICT Session Validator"**  
   - Implement JavaScript to detect current ICT Kill Zone based on UTC time and assign priority, characteristics, session strength, and trading permission flags.  
   - Input: Output of "Extract ICT Signal Data".  
   - Output: Enriched JSON with `session_validation` object.

4. **Add HTTP Request Node "Get Coinbase Market Data"**  
   - Method: GET  
   - URL: `https://api.coinbase.com/api/v3/brokerage/products/{{ $json.symbol }}`  
   - Set timeout to 10000 ms.  
   - Use Coinbase Advanced API credentials.  
   - Input: Output of "ICT Session Validator".  
   - Output: Coinbase product data appended.

5. **Add OpenAI Node "ICT AI Analysis"**  
   - Use GPT-4o model with maxTokens 1000 and temperature 0.3.  
   - Provide detailed prompt instructing ICT 2025 methodology JSON output including session kill zone analysis, market structure, smart money concepts, and risk management.  
   - Input: Output of "Get Coinbase Market Data".  
   - Output: AI JSON response.

6. **Add Code Node "Parse ICT AI Analysis"**  
   - Parse AI response content robustly with fallback to default ICT analysis if parsing fails.  
   - Enrich AI analysis with current kill zone, session characteristics, GMT time, and append Coinbase data and workflow metadata.  
   - Input: Output of "ICT AI Analysis".  
   - Output: Fully enriched signal JSON.

7. **Add If Node "ICT Quality & Session Filter"**  
   - Conditions (all must be true):  
     - Signal quality != "LOW"  
     - Confidence score >= 60  
     - Session trading allowed == true  
     - ICT session alignment == true  
   - Input: Output of "Parse ICT AI Analysis".  
   - True branch: Proceed to trade execution.  
   - False branch: Proceed to rejection logging.

8. **Add HTTP Request Node "Execute ICT Trade"**  
   - Method: POST  
   - URL: `https://api.coinbase.com/api/v3/brokerage/orders`  
   - Body (JSON):  
     ```json
     {
       "product_id": "{{ $json.symbol }}",
       "side": "{{ $json.action.toLowerCase() }}",
       "order_configuration": {
         "market_market_ioc": { "quote_size": "{{ $json.quantity || '10' }}" }
       }
     }
     ```  
   - Headers: Content-Type application/json  
   - Use Coinbase Advanced API credentials.  
   - Timeout: 30000 ms.  
   - Input: True branch output of filter node.  
   - Output: Coinbase trade response.

9. **Add Notion Node "Create ICT Trading Record"**  
   - Resource: databasePage  
   - Database ID: Use environment variable `NOTION_TRADING_DB_ID`.  
   - Map properties: Symbol (title), Action (select), Confidence (number), Kill Zone (rich_text), Price (number), Timestamp (date), Session Strength (number), Risk Level (select).  
   - Input: Output of "Execute ICT Trade".  
   - Output: Confirmation of page creation.

10. **Add OpenAI Node "Generate ICT Notification"**  
    - Model: GPT-4o, maxTokens 500, temperature 0.7.  
    - Prompt: Generate Telegram formatted notification with emojis, key ICT points, action, confidence, session info, and risk details.  
    - Input: Output of Notion trading record node.  
    - Output: Notification text message.

11. **Add Telegram Node "Send ICT Telegram Alert"**  
    - Chat ID: Use variable `TELEGRAM_CHAT_ID`.  
    - Text: Use generated notification message or fallback.  
    - Parse mode: Markdown.  
    - Disable web page preview.  
    - Input: Output of "Generate ICT Notification".  
    - Output: Telegram message confirmation.

12. **Add Notion Node "Log ICT Rejected Signal"**  
    - Resource: databasePage  
    - Database ID: Use environment variable `NOTION_REJECTED_DB_ID`.  
    - Map fields: Symbol (title), Rejection Reason (rich_text), Confidence Score (number), Session (rich_text), Timestamp (date).  
    - Input: False branch output of filter node.  
    - Output: Confirmation of rejection log.

13. **Add Respond to Webhook Node "ICT Webhook Response"**  
    - Respond with JSON summarizing processing result: status, symbol, action, kill zone, confidence, session strength, ICT factors (structure break, liquidity grab, fair value gap, order block).  
    - Input: Both outputs of "Send ICT Telegram Alert" and "Log ICT Rejected Signal".  
    - Output: HTTP JSON response.

14. **Add HTTP Request Node "HTTP Request"**  
    - Method: POST  
    - URL: From environment variable `WEBHOOK_URL` or fallback webhook.site URL.  
    - Body: JSON with signal_data, timestamp, workflow_id.  
    - Content-Type: application/json  
    - Input: Output of "ICT Webhook Response".  
    - Output: External webhook response.

15. **Add Telegram Node "Get a chat"**  
    - Chat ID: Use variable `TELEGRAM_CHAT_ID` or default.  
    - Resource: chat  
    - Input: Output of HTTP Request node.  
    - Output: Telegram chat metadata.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow follows ICT 2025 Smart Money Concepts and Kill Zones methodology to align trades with market sessions. | Tags: ICT Trading 2025, Kill Zones, Smart Money Concepts                                        |
| Coinbase Advanced API is used for market data and trade execution requiring proper API credentials and permissions. | https://docs.cloud.coinbase.com/sign-in-with-coinbase/docs/api-brokerage-overview               |
| GPT-4o model is used for advanced AI signal analysis with custom prompt for ICT methodology adherence.                | OpenAI GPT-4o documentation and Langchain integration for n8n                                   |
| Notion databases are used for logging executed and rejected trades; environment variables must be set for DB IDs.    | Notion API docs: https://developers.notion.com/docs/getting-started                              |
| Telegram bot credentials and chat IDs must be configured as environment variables for triggers and notifications.    | Telegram Bot API: https://core.telegram.org/bots/api                                            |
| Timezone handling uses UTC to standardize kill zone detection; server time sync is critical for accurate session validation. |                                                                                                 |
| Error handling in AI parsing includes fallback to ensure no critical failure blocks workflow execution.              |                                                                                                 |
| The workflow's execution order is sequential following signal processing, analysis, filtering, execution, logging, and notification. |                                                                                                 |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.

---