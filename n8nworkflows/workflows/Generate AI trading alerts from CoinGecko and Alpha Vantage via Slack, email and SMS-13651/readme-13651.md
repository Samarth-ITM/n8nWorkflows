Generate AI trading alerts from CoinGecko and Alpha Vantage via Slack, email and SMS

https://n8nworkflows.xyz/workflows/generate-ai-trading-alerts-from-coingecko-and-alpha-vantage-via-slack--email-and-sms-13651


# Generate AI trading alerts from CoinGecko and Alpha Vantage via Slack, email and SMS

# AI Trading Alert Bot Reference Document

This document provides a technical breakdown of the "AI Trading Alert Bot" n8n workflow. The system automates the lifecycle of market analysis, from data ingestion and technical calculation to AI-driven signal generation and multi-channel notification.

---

### 1. Workflow Overview

The workflow monitors financial markets (Crypto and Stocks) on a recurring schedule. It processes raw price data through a technical analysis engine and an external AI model to generate high-confidence trading signals. These signals are filtered, archived in a database, and broadcasted via Slack, Email, and SMS.

#### Logical Blocks:
*   **1.1 Data Ingestion:** Scheduled trigger and parallel API requests to market data providers.
*   **1.2 Technical Analysis:** JS-based calculation of financial indicators (RSI, MACD, etc.).
*   **1.3 AI Intelligence:** Communication with a machine learning model to predict market movements.
*   **1.4 Signal Processing:** Evaluation of AI confidence levels and risk assessment.
*   **1.5 Data Persistence & Routing:** Saving signals to PostgreSQL and branching logic for alerts.
*   **1.6 Multi-Channel Notification:** Formatting and delivery of alerts via Slack, SMTP, and Twilio.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Collection
**Overview:** This block initiates the process every 15 minutes and fetches raw data from CoinGecko (Crypto) and Alpha Vantage (Stocks).
*   **Nodes Involved:** 
    *   `Market Data Trigger - Every 15 minutes` (Schedule Trigger): Set to cron `*/15 * * * *`.
    *   `Fetch cryptocurrency prices from CoinGecko` (HTTP Request): GET request to CoinGecko API.
    *   `Fetch stock prices from Alpha Vantage` (HTTP Request): GET request to Alpha Vantage query endpoint.
    *   `Combine crypto and stock market data` (Merge): Combines the results from both API calls into a single dataset.
*   **Potential Failures:** API rate limiting (429 errors), invalid API keys, or timeout during market volatility.

#### 2.2 Analysis & AI Generation
**Overview:** Transforms raw price data into technical insights and sends them to an AI model for signal prediction.
*   **Nodes Involved:**
    *   `Calculate technical indicators` (Code): A JavaScript node that calculates SMA (50/200), EMA (12), RSI (14), MACD, and Bollinger Bands. 
    *   `Call AI model for signal prediction` (HTTP Request): A POST request sending the symbol, price, technical indicators, and trend data to `https://api.example.com/ai/predict-signal`.
    *   `Analyze AI signals and confidence` (Code): Evaluates the AI's response. It sets thresholds (e.g., 0.65 for Buy/Sell) and determines the `alertLevel` (Info, Medium, High, Critical) and recommended `positionSize`.
*   **Key Expressions:** Uses `$json.technicalIndicators` and `$json.trend` to populate the AI prompt.

#### 2.3 Validation & Persistence
**Overview:** Filters out low-confidence "Hold" signals and ensures only actionable data is stored and sent forward.
*   **Nodes Involved:**
    *   `Validate signal strength and filter` (Filter): Only passes items where `shouldAlert` is true.
    *   `Store trade signal in PostgreSQL` (Postgres): Inserts the signal details into the `trading_signals` table for historical audit.
*   **Edge Cases:** Database connection timeouts or schema mismatches if the table structure changes.

#### 2.4 Alerting & Logging
**Overview:** Converts technical data into human-readable messages and distributes them across three platforms.
*   **Nodes Involved:**
    *   `Route by signal action` (Switch): Branches logic based on whether the action is "BUY" or "SELL".
    *   `Generate trading alert message` (Code): Formats three distinct message types: Markdown for Slack, HTML for Email, and plain text for SMS.
    *   `Send Slack notification` (HTTP Request): POSTs to a Slack Webhook.
    *   `Send Email alert` (Email Send): Uses SMTP to send structured HTML reports.
    *   `Send SMS alert (Twilio)` (HTTP Request): POSTs to the Twilio Messages API.
    *   `Log trade execution and success` (Code): Updates workflow static data to track cumulative statistics (total signals, average confidence).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Market Data Trigger | Schedule Trigger | Workflow Initiation | None | CoinGecko, Alpha Vantage | 1. Trigger: Automatically runs every 15 minutes |
| Fetch cryptocurrency prices | HTTP Request | Data Ingestion | Market Data Trigger | Combine data | 2. Fetch & Prepare Data |
| Fetch stock prices | HTTP Request | Data Ingestion | Market Data Trigger | Combine data | 2. Fetch & Prepare Data |
| Combine crypto and stock data | Merge | Data Synthesis | CoinGecko, Alpha Vantage | Calculate indicators | 2. Fetch & Prepare Data |
| Calculate technical indicators | Code | Math Processing | Combine data | Call AI model | 3. Analysis & Signal Generation |
| Call AI model | HTTP Request | AI Integration | Calculate indicators | Analyze signals | 3. Analysis & Signal Generation |
| Analyze AI signals | Code | Decision Logic | Call AI model | Validate signal | 3. Analysis & Signal Generation |
| Validate signal strength | Filter | Data Filtering | Analyze AI signals | Store in Postgres | 4. Validate, Alert & Store |
| Store trade signal | Postgres | Persistence | Validate signal | Route by action | 4. Validate, Alert & Store |
| Route by signal action | Switch | Logic Branching | Store trade signal | Generate alert | 4. Validate, Alert & Store |
| Generate trading alert message | Code | Content Formatting | Route by action | Slack, Email, SMS | 4. Validate, Alert & Store |
| Send Slack notification | HTTP Request | Notification | Generate alert | Log execution | 4. Validate, Alert & Store |
| Send Email alert | Email Send | Notification | Generate alert | Log execution | 4. Validate, Alert & Store |
| Send SMS alert (Twilio) | HTTP Request | Notification | Generate alert | Log execution | 4. Validate, Alert & Store |
| Log trade execution | Code | Analytics/Logging | Slack, Email, SMS | None | 4. Validate, Alert & Store |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** node. Set the mode to "Cron Expression" and use `*/15 * * * *`.
2.  **API Data Fetching:** 
    *   Add an **HTTP Request** node for CoinGecko. URL: `https://api.coingecko.com/api/v3/simple/price`.
    *   Add an **HTTP Request** node for Alpha Vantage. URL: `https://www.alphavantage.co/query`.
    *   Connect both to a **Merge** node (Mode: Combine).
3.  **Indicator Logic:** Add a **Code** node. Use JavaScript to calculate RSI and Moving Averages based on the price data provided by the APIs.
4.  **AI Integration:** Add an **HTTP Request** node. Method: `POST`. URL: `[Your AI Provider Endpoint]`. Body: JSON-map the technical indicators from the previous step.
5.  **Confidence Logic:** Add a **Code** node to parse the AI output. Create a boolean variable `shouldAlert` that is `true` only if `confidence > 0.65`.
6.  **Filter & DB:**
    *   Add a **Filter** node. Condition: `shouldAlert` equals `true`.
    *   Add a **PostgreSQL** node. Action: Insert. Map the symbol, action, and price fields.
7.  **Routing:** Add a **Switch** node. Set it to check the `action` field. Route "BUY" and "SELL" to the next step.
8.  **Messaging Engine:** Add a **Code** node to create message strings. Define `alertMessages.slack`, `alertMessages.email.body`, and `alertMessages.sms`.
9.  **Notifications:**
    *   **Slack:** HTTP Request (POST) to your Webhook URL.
    *   **Email:** Email Send node. Configure SMTP credentials and map the HTML body.
    *   **SMS:** HTTP Request (POST) to Twilio. URL: `https://api.twilio.com/2010-04-01/Accounts/[SID]/Messages.json`.
10. **Final Logging:** Add a **Code** node. Use `$getWorkflowStaticData('global')` to store the count of signals generated over time.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Market Data Providers** | [CoinGecko API Documentation](https://www.coingecko.com/en/api) |
| **Stock Data Providers** | [Alpha Vantage API Documentation](https://www.alphavantage.co/documentation/) |
| **Database Schema** | Requires a table named `trading_signals` with columns: `signalId`, `symbol`, `action`, `confidence`, `currentPrice`, and `timestamp`. |
| **AI Model Requirement** | The workflow expects a JSON response containing `signal`, `confidence`, and `reasoning`. |
| **Static Data Usage** | The final node uses n8n's Static Data feature to persist statistics across executions without an external DB. |