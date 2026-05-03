Automated Multi-Agent Trading Analysis with GPT-5, Telegram, Coinbase & Notion

https://n8nworkflows.xyz/workflows/automated-multi-agent-trading-analysis-with-gpt-5--telegram--coinbase---notion-8569


# Automated Multi-Agent Trading Analysis with GPT-5, Telegram, Coinbase & Notion

### 1. Workflow Overview

This workflow is designed as an **Automated Multi-Agent Trading Analysis System** integrating advanced AI models (GPT-5) with real-time financial data, trading execution, and logging tools. It targets hedge fund-style multi-agent investment analysis and decision-making, synthesizing diverse expert perspectives into actionable trading signals.

**Use Cases:**
- Automated equity or asset evaluation via multiple expert AI personas.
- Dynamic risk management and portfolio decision support.
- Execution of trades on Coinbase and logging outcomes to Notion.
- Real-time interactive triggers and notifications via Telegram.

**Logical Blocks:**

- **1.1 Input Reception & Market Data Fetching:** Starts with Telegram trigger, fetches market data from Coinbase API.
- **1.2 Multi-Agent AI Analysis:** Several nodes invoke GPT-5 with specialized prompts representing investment legends and analytical roles (e.g., Buffett, Graham, Sentiment, Valuation agents).
- **1.3 Risk Management:** Aggregates agent signals and market data, evaluates risk, sets position sizing and stop-loss limits.
- **1.4 Portfolio Management & Decision Making:** Synthesizes all agent inputs and risk assessments to produce final buy/sell/hold decisions.
- **1.5 Execution & Reporting:** Executes orders on Coinbase, logs trades to Notion, and sends summary notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Market Data Fetching

**Overview:**  
Captures user input via Telegram, triggers the workflow, and retrieves current exchange rate data for the requested asset via Coinbase API.

**Nodes Involved:**  
- Telegram Trigger  
- Coinbase Market Data

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* telegramTrigger  
  - *Role:* Entry point for user commands/messages via Telegram, listens to callback queries, channel posts, and messages.  
  - *Configuration:* Uses a Telegram API credential; listens to multiple update types to capture interactive and direct messages.  
  - *Input/Output:* No input; outputs message data including asset ticker symbol, chat ID, etc.  
  - *Failure Modes:* Telegram API connectivity/auth issues; invalid or unsupported message formats.  
  - *Version:* v1  

- **Coinbase Market Data**  
  - *Type:* httpRequest  
  - *Role:* Fetch exchange rate data for the ticker symbol from Coinbase API.  
  - *Configuration:* GET request to `https://api.coinbase.com/v2/exchange-rates?currency={{$json.ticker}}` with header authentication.  
  - *Input/Output:* Input from Telegram Trigger; outputs live market data including exchange rates.  
  - *Failure Modes:* API rate limits, invalid ticker symbols, network errors, authentication failures.  
  - *Credential:* HTTP Header Auth  

---

#### 2.2 Multi-Agent AI Analysis

**Overview:**  
Runs multiple AI agents, each simulating a renowned investor or analyst with a specialized investment philosophy or analytical role. Each agent receives relevant input data and returns structured JSON signals or qualitative assessments.

**Nodes Involved:**  
- Warren Buffett Agent  
- Ben Graham Agent  
- Aswath Damodaran Agent  
- Michael Burry Agent  
- Charlie Munger Agent  
- Bill Ackman Agent  
- Stanley Druckenmiller Agent  
- Rakesh Jhunjhunwala Agent  
- Phil Fisher Agent  
- Peter Lynch Agent  
- Cathie Wood Agent  
- Mohnish Pabrai Agent  
- Valuation Agent  
- Sentiment Agent  
- Fundamentals Agent  
- Technicals Agent

**Node Details:**  

- *Type:* openAi  
- *Role:* Each node acts as an AI "agent" that analyzes the asset using its unique prompt reflecting the investment style or analytical focus of a famous investor or domain expertise.  
- *Configuration:* All use GPT-5 model with carefully crafted role-based prompts and context variables extracted from Coinbase data and Telegram inputs.  
- *Inputs:* JSON data enriched with ticker, price, financials, market sentiment, macro data, social media, fundamentals, technicals, etc.  
- *Outputs:* Structured JSON signals covering buy/hold/sell recommendations, confidence scores, risk/reward assessments, reasoning, and other agent-specific fields.  
- *Failure Modes:* OpenAI API rate limits, prompt misformatting, missing data in inputs causing incomplete analysis, timeout or service unavailability.  
- *Credentials:* OpenAI API with valid credentials.

---

#### 2.3 Risk Management

**Overview:**  
Aggregates signals from all AI agents plus market data to evaluate portfolio risk, position sizing, stop-loss levels, and approve or reject trading actions.

**Nodes Involved:**  
- Risk Manager

**Node Details:**  

- *Type:* openAi  
- *Role:* Acts as the Chief Risk Officer evaluating aggregated signals and market volatility, calculating position limits, and generating risk scores and notes.  
- *Configuration:* Uses GPT-5 with a prompt defining risk management rules, including max 5% position size, diversification, dynamic stop-loss, and confidence-based adjustments.  
- *Inputs:* Portfolio data, agent signals, volatility index (VIX), correlation matrix, etc.  
- *Outputs:* JSON with max position size, stop loss, risk score, portfolio impact level, risk notes, and approval boolean.  
- *Failure Modes:* Missing or inconsistent input data, API latency, or errors in JSON parsing.  
- *Credential:* OpenAI API.

- *Connections:* Receives output from all AI agent nodes; outputs to Portfolio Manager.

---

#### 2.4 Portfolio Management & Decision Making

**Overview:**  
Synthesizes all agent analyses and risk assessment to make final investment decisions including position sizing and trade signals.

**Nodes Involved:**  
- Portfolio Manager

**Node Details:**  

- *Type:* openAi  
- *Role:* Chief Portfolio Manager node aggregates agent signals weighted by historical accuracy, applies risk limits, and outputs final buy/sell/hold decisions with allocation percentages.  
- *Configuration:* GPT-5 prompt outlines decision framework and rules emphasizing majority agent consensus, risk approval, and confidence thresholds.  
- *Inputs:* Asset info, current portfolio, available cash, all agent signals, and risk assessment data.  
- *Outputs:* Final decision, allocation percentage, and summary signals.  
- *Failure Modes:* Aggregation logic errors, inconsistent or missing input signals, API errors.  
- *Credential:* OpenAI API.

- *Connections:* Outputs connected to execution and reporting nodes.

---

#### 2.5 Execution & Reporting

**Overview:**  
Executes the approved trades on Coinbase, logs the trade to Notion database, and sends Telegram notifications summarizing the analysis and actions.

**Nodes Involved:**  
- Execute Order  
- Log to Notion  
- Send Analysis Result

**Node Details:**  

- **Execute Order**  
  - *Type:* httpRequest  
  - *Role:* Sends POST request to Coinbase API to execute trade transactions based on Portfolio Manager's decisions.  
  - *Configuration:* URL uses dynamic account ID from JSON input; authenticated via header.  
  - *Inputs:* Final order details from Portfolio Manager (asset, amount, side).  
  - *Outputs:* Coinbase transaction response.  
  - *Failure Modes:* Order rejection, insufficient funds, API limits, authentication failure.  
  - *Credential:* HTTP Header Auth.

- **Log to Notion**  
  - *Type:* notion  
  - *Role:* Logs trade details and analysis summary into a Notion database for record-keeping.  
  - *Configuration:* Uses Notion API credentials and targets a specific database ID.  
  - *Inputs:* Trade and analysis data from Portfolio Manager.  
  - *Outputs:* Notion page creation response.  
  - *Failure Modes:* API permission errors, invalid database ID, rate limits.  
  - *Credential:* Notion API.

- **Send Analysis Result**  
  - *Type:* telegram  
  - *Role:* Sends a formatted summary message to the Telegram user/chat who triggered the workflow.  
  - *Configuration:* Uses Telegram API credentials; message includes asset, price, agent signals, risk notes, final decision, allocation, and logs confirmation.  
  - *Inputs:* Final output from Portfolio Manager and risk manager nodes, including chat ID.  
  - *Outputs:* Telegram message delivery confirmation.  
  - *Failure Modes:* Chat ID missing or invalid, Telegram API errors.  
  - *Credential:* Telegram API.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                  | Input Node(s)               | Output Node(s)                  | Sticky Note                                         |
|----------------------------|---------------------|-------------------------------------------------|-----------------------------|--------------------------------|-----------------------------------------------------|
| Telegram Trigger           | telegramTrigger      | Receives user input via Telegram                 |                             | Coinbase Market Data            |                                                     |
| Coinbase Market Data       | httpRequest         | Fetches live exchange rates from Coinbase       | Telegram Trigger             | AI Agent nodes (all)            |                                                     |
| Warren Buffett Agent       | openAi              | Value investing analysis per Buffett philosophy | Coinbase Market Data         | Risk Manager                   |                                                     |
| Ben Graham Agent           | openAi              | Value investing with margin of safety focus     | Coinbase Market Data         | Risk Manager                   |                                                     |
| Aswath Damodaran Agent     | openAi              | Valuation expert analysis                        | Coinbase Market Data         | Risk Manager                   |                                                     |
| Michael Burry Agent        | openAi              | Contrarian deep value and bubble detection      | Coinbase Market Data         | Risk Manager                   |                                                     |
| Charlie Munger Agent       | openAi              | Quality and moat analysis                         | Coinbase Market Data         | Risk Manager                   |                                                     |
| Bill Ackman Agent          | openAi              | Activist investing and value creation signals   | Coinbase Market Data         | Risk Manager                   |                                                     |
| Stanley Druckenmiller Agent| openAi              | Macro and asymmetric trade analysis              | Coinbase Market Data         | Risk Manager                   |                                                     |
| Rakesh Jhunjhunwala Agent  | openAi              | Emerging markets and big bull perspective       | Coinbase Market Data         | Risk Manager                   |                                                     |
| Phil Fisher Agent          | openAi              | Growth and management quality analysis          | Coinbase Market Data         | Risk Manager                   |                                                     |
| Peter Lynch Agent          | openAi              | Identification of simple growth businesses      | Coinbase Market Data         | Risk Manager                   |                                                     |
| Cathie Wood Agent          | openAi              | Disruptive innovation and long-term growth      | Coinbase Market Data         | Risk Manager                   |                                                     |
| Mohnish Pabrai Agent       | openAi              | Low-risk, high-upside asymmetric bets           | Coinbase Market Data         | Risk Manager                   |                                                     |
| Valuation Agent            | openAi              | Multi-method valuation analysis                   | Coinbase Market Data         | Risk Manager                   |                                                     |
| Sentiment Agent            | openAi              | Market sentiment and social media analysis      | Coinbase Market Data         | Risk Manager                   |                                                     |
| Fundamentals Agent         | openAi              | Financial statement and ratio analysis           | Coinbase Market Data         | Risk Manager                   |                                                     |
| Technicals Agent           | openAi              | Technical chart and momentum analysis            | Coinbase Market Data         | Risk Manager                   |                                                     |
| Risk Manager              | openAi              | Portfolio risk evaluation and sizing             | All AI Agents                | Portfolio Manager              |                                                     |
| Portfolio Manager          | openAi              | Final decision aggregation and allocation        | Risk Manager                 | Execute Order, Log to Notion, Send Analysis Result |                                                     |
| Execute Order              | httpRequest         | Sends trade orders to Coinbase                    | Portfolio Manager            |                                |                                                     |
| Log to Notion             | notion              | Logs trade and analysis data                       | Portfolio Manager            |                                |                                                     |
| Send Analysis Result      | telegram            | Sends summary message to Telegram user            | Portfolio Manager            |                                |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: telegramTrigger  
   - Configure with your Telegram API credentials.  
   - Listen for updates: "callback_query", "channel_post", "message".  
   - Position it as the workflow's entry point.

2. **Create Coinbase Market Data Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.coinbase.com/v2/exchange-rates?currency={{$json.ticker}}`  
   - Authentication: HTTP Header Auth with Coinbase API key.  
   - Connect Telegram Trigger output to this node.

3. **Create all AI Agent Nodes (16 total):**  
   - Type: OpenAI  
   - Model: GPT-5  
   - Provide each node with its unique prompt text reflecting the assigned investment legend or analytic role.  
   - Inputs: Use the JSON data from Coinbase Market Data enriched with additional context where applicable.  
   - Credentials: Use OpenAI API key.  
   - Connect Coinbase Market Data output to each AI agent node.

4. **Create Risk Manager Node:**  
   - Type: OpenAI  
   - Model: GPT-5  
   - Prompt includes rules for risk limits, portfolio impact, stop-loss, approval boolean, etc.  
   - Inputs: Aggregate outputs from all AI agents.  
   - Connect outputs from all AI Agents into Risk Manager.

5. **Create Portfolio Manager Node:**  
   - Type: OpenAI  
   - Model: GPT-5  
   - Prompt to aggregate all agent signals and risk assessment, apply decision rules and position sizing.  
   - Input: Risk Manager output plus agent signals summary.  
   - Connect Risk Manager output to Portfolio Manager.

6. **Create Execute Order Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.coinbase.com/v2/accounts/{{$json.account_id}}/transactions`  
   - Authentication: HTTP Header Auth (Coinbase API key).  
   - Connect Portfolio Manager output to Execute Order.

7. **Create Log to Notion Node:**  
   - Type: Notion  
   - Resource: databasePage  
   - Database ID: Set to your Notion trading log database.  
   - Credentials: Notion API.  
   - Connect Portfolio Manager output to Log to Notion.

8. **Create Send Analysis Result Node:**  
   - Type: Telegram  
   - Use Telegram API credentials.  
   - Configure message template with asset, price, agent summary, risk notes, final decision, allocation, and confirmation.  
   - Use chat ID from Telegram Trigger to send message back to user.  
   - Connect Portfolio Manager output to this node.

9. **Validate All Connections:**  
   - Telegram Trigger → Coinbase Market Data  
   - Coinbase Market Data → All AI Agent Nodes  
   - All AI Agent Nodes → Risk Manager  
   - Risk Manager → Portfolio Manager  
   - Portfolio Manager → Execute Order, Log to Notion, Send Analysis Result

10. **Set Credentials:**  
    - Telegram API (for both Trigger and Send nodes)  
    - Coinbase API (HTTP Header Auth for Market Data and Execute Order)  
    - OpenAI API for all GPT-5 nodes  
    - Notion API for logging

11. **Test Workflow:**  
    - Trigger via Telegram message with asset ticker.  
    - Monitor logs and outputs stepwise for errors or missing data.  
    - Confirm order execution, Notion logging, and Telegram notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow integrates multiple AI personas simulating legendary investors and analysts for diversified perspectives.  | Enhances robustness and reduces single-model bias |
| Uses GPT-5 model requiring valid OpenAI API access with appropriate usage limits and cost considerations.            | OpenAI API documentation: https://openai.com/docs |
| Coinbase API requires header authentication and permission for market data and trading execution endpoints.          | Coinbase API docs: https://docs.cloud.coinbase.com |
| Notion API integration requires database setup with a schema to store trade logs and analysis data.                  | Notion API docs: https://developers.notion.com    |
| Telegram bot must have appropriate permissions and webhook configured for triggers and messaging.                    | Telegram Bot API: https://core.telegram.org/bots/api |
| The workflow assumes input JSON fields are correctly parsed and enriched before AI nodes; missing data may cause errors. | Preprocessing or validation recommended           |
| Consider API rate limits and error handling (retry, fallback) especially for OpenAI and Coinbase endpoints.          | Implement error workflows or alerts as enhancements |
| This system is designed for equities or crypto assets where Coinbase supports trading; adjust accordingly for other assets. | Customization may be needed for asset types        |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, respecting all relevant content policies. It contains no illegal, offensive, or protected content. All data handled is legal and public.