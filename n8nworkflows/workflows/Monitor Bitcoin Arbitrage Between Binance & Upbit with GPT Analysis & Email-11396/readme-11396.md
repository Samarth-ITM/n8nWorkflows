Monitor Bitcoin Arbitrage Between Binance & Upbit with GPT Analysis & Email

https://n8nworkflows.xyz/workflows/monitor-bitcoin-arbitrage-between-binance---upbit-with-gpt-analysis---email-11396


# Monitor Bitcoin Arbitrage Between Binance & Upbit with GPT Analysis & Email

### 1. Workflow Overview

This workflow automates the monitoring and analysis of Bitcoin price arbitrage opportunities between two cryptocurrency exchanges: Binance (a global market) and Upbit (a Korean market). It specifically tracks the "Kimchi Premium," the price gap between BTC prices on these exchanges, adjusted by currency exchange rates between USD and KRW. The workflow fetches live market prices and Forex rates at scheduled intervals, normalizes and merges this data, then leverages an AI agent to analyze the arbitrage spreads and generate a strategic report sent via email to traders.

Logical blocks:

- **1.1 Data Collection:** Scheduled trigger fetching live BTC prices from Binance and Upbit, plus USD/KRW Forex rates.
- **1.2 Data Normalization & Merging:** Formatting raw API data into a common schema and merging into a unified dataset.
- **1.3 AI Analysis & Reporting:** Using LangChain-based OpenAI models to calculate spreads, generate a readable arbitrage report, parse structured output, and send it as an email.
- **1.4 Notification:** Sending the AI-generated report to traders via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Collection

**Overview:**  
This block triggers the workflow every 30 minutes and collects real-time price data from Binance and Upbit, plus the current USD/KRW Forex exchange rate.

**Nodes Involved:**  
- Schedule Trigger  
- Get price from Binance (HTTP Request)  
- Get price from Upbit (HTTP Request)  
- Get price from Forex (HTTP Request)

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every 30 minutes automatically.  
  - Configuration: Interval set to every 30 minutes.  
  - Inputs: None  
  - Outputs: Triggers three HTTP request nodes simultaneously.  
  - Edge cases: Misconfiguration can cause no triggers or too frequent triggers, which may hit API rate limits.

- **Get price from Binance**  
  - Type: HTTP Request  
  - Role: Fetch latest BTC price from Binance API.  
  - Configuration:  
    - URL: `https://api.binance.com/api/v3/trades`  
    - Query Parameters: `symbol=BTCUSDT`, `limit=1` (fetch last trade)  
    - Method: GET  
  - Input: Trigger from Schedule Trigger  
  - Output: Raw JSON with trade price data.  
  - Edge cases: API downtime, rate limiting, response format changes.

- **Get price from Upbit**  
  - Type: HTTP Request  
  - Role: Fetch latest BTC prices in USDT-BTC, KRW-BTC, and KRW-USDT markets from Upbit API.  
  - Configuration:  
    - URL: `https://api.upbit.com/v1/ticker`  
    - Query Parameters: `markets=USDT-BTC,KRW-BTC,KRW-USDT`  
    - Headers: `accept: application/json`  
  - Input: Trigger from Schedule Trigger  
  - Output: JSON array with prices for multiple markets.  
  - Edge cases: API changes, connection errors, missing markets.

- **Get price from Forex**  
  - Type: HTTP Request  
  - Role: Fetch USD/KRW exchange rate from public Forex API.  
  - Configuration:  
    - URL: `https://api.manana.kr/exchange/rate/KRW/USD.json`  
    - Method: GET  
  - Input: Trigger from Schedule Trigger  
  - Output: JSON with exchange rate data.  
  - Edge cases: API unavailability, rate format changes.

---

#### 1.2 Data Normalization & Merging

**Overview:**  
This block standardizes the diverse data formats returned from the APIs into a unified schema and merges the results into one dataset for AI processing.

**Nodes Involved:**  
- Edit Fields For Binance Data (Set Node)  
- Edit Fields For Upbit Data (Set Node)  
- Edit Fields For Forex Data (Set Node)  
- Merge Data (Merge Node)

**Node Details:**  

- **Edit Fields For Binance Data**  
  - Type: Set Node  
  - Role: Assigns consistent field names and adds metadata to Binance price data.  
  - Configuration:  
    - Sets `Exchange` = "Binance" (string)  
    - Sets `Markets` = "USDT-BTC" (string)  
    - Converts Binance price JSON field `price` to `Price` (number)  
  - Input: Output from Binance HTTP Request  
  - Output: One normalized JSON object per trade.  
  - Edge cases: Missing price data or unexpected data types.

- **Edit Fields For Upbit Data**  
  - Type: Set Node  
  - Role: Normalizes Upbit data fields similarly.  
  - Configuration:  
    - Sets `Exchange` = "Upbit"  
    - Maps Upbit `market` field to `Markets`  
    - Maps Upbit `trade_price` field to `Price` (number)  
  - Input: Output from Upbit HTTP Request  
  - Output: Array of normalized price data for multiple markets.  
  - Edge cases: API data missing expected fields.

- **Edit Fields For Forex Data**  
  - Type: Set Node  
  - Role: Normalizes Forex exchange rate data.  
  - Configuration:  
    - Sets `Exchange` = "Forex"  
    - Sets `Markets` = "USDKRW=X" (symbol for USD to KRW exchange)  
    - Maps `rate` field to `Price` (number)  
  - Input: Output from Forex HTTP Request  
  - Output: Single normalized object with exchange rate.  
  - Edge cases: Unexpected JSON structure.

- **Merge Data**  
  - Type: Merge Node  
  - Role: Combines the three normalized datasets from Binance, Upbit, and Forex into one aggregated dataset.  
  - Configuration:  
    - Number of inputs: 3 (Binance, Upbit, Forex)  
    - Mode: Append (combine all items into one list)  
  - Input: Outputs from above three Set nodes  
  - Output: Unified dataset for analysis.  
  - Edge cases: Missing inputs can cause incomplete data sets.

---

#### 1.3 AI Analysis & Reporting

**Overview:**  
This block feeds the merged data to an AI agent to calculate arbitrage spreads, generate a detailed report, parse structured output, and format the message for email delivery.

**Nodes Involved:**  
- Analyzer (LangChain Agent Node)  
- OpenAI Chat Model for Analyzer (OpenAI Chat Model Node)  
- OpenAI Chat Model for Format (OpenAI Chat Model Node)  
- Structured Output Parser (LangChain Output Parser Node)

**Node Details:**  

- **Analyzer**  
  - Type: LangChain Agent Node  
  - Role: Uses AI prompt to analyze price data, calculate spreads, and compose a strategic arbitrage report.  
  - Configuration:  
    - Prompt includes timestamp and all merged data with exchange, market, and price.  
    - System message instructs AI to focus on price gaps, arbitrage profitability, and format output with HTML-friendly breaks (`<br>`).  
    - Output parser enabled to produce structured JSON with `subject` and `message` fields.  
  - Input: Merged dataset from Merge Data node  
  - Output: AI-generated structured analysis and report.  
  - Edge cases: API rate limits, malformed input data causing AI prompt failures, network errors.

- **OpenAI Chat Model for Analyzer**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Underlying model providing language generation for Analyzer node.  
  - Configuration:  
    - Model: `gpt-4o-mini` (optimized GPT-4 variant)  
    - Code interpreter enabled for enhanced reasoning  
  - Input: Receives prompts from Analyzer node  
  - Output: AI textual completion  
  - Credentials: Requires valid OpenAI API key configured in n8n credentials  
  - Edge cases: API key invalid or quota exceeded, model not available

- **OpenAI Chat Model for Format**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Formats the AI-generated report message for email presentation.  
  - Configuration:  
    - Same model as analyzer (`gpt-4o-mini`) without code interpreter  
  - Input: Output from Structured Output Parser  
  - Output: Clean formatted email content  
  - Edge cases: Model errors, formatting inconsistencies

- **Structured Output Parser**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Parses AI output text into JSON structure with `subject` and `message` fields for email.  
  - Configuration:  
    - Auto-fix enabled to handle minor JSON formatting issues  
    - Example schema provided to guide parsing  
  - Input: Output from OpenAI Chat Model for Format  
  - Output: Structured JSON for email node  
  - Edge cases: Parsing errors if AI output deviates from expected schema

---

#### 1.4 Notification

**Overview:**  
Final block sends the AI-generated arbitrage report email to the trader.

**Nodes Involved:**  
- Send a Message for Traders (Gmail Node)

**Node Details:**  

- **Send a Message for Traders**  
  - Type: Gmail Node  
  - Role: Sends email with arbitrage report to designated trader email address.  
  - Configuration:  
    - `To` email: `trader@example.com` (user should update to real address)  
    - Subject and Message consumed from structured AI output (`subject` and `message` fields)  
    - Uses connected Google/Gmail OAuth2 credentials  
  - Input: Output from Analyzer node (structured email content)  
  - Output: Email sent event  
  - Edge cases: Credential expiration, email send failure, invalid email addresses

---

### 3. Summary Table

| Node Name                      | Node Type                                   | Functional Role                          | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                          |
|--------------------------------|---------------------------------------------|----------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                           | Initiates workflow every 30 minutes    | None                             | Get price from Binance, Upbit, Forex | ## 1. Data Collection<br>Triggers on a schedule to fetch live BTC prices from Binance and Upbit, along with the real-time USD/KRW exchange rate. |
| Get price from Binance         | HTTP Request                              | Fetch BTC price from Binance API       | Schedule Trigger                 | Edit Fields For Binance Data       |                                                                                                                      |
| Get price from Upbit           | HTTP Request                              | Fetch BTC prices from Upbit API        | Schedule Trigger                 | Edit Fields For Upbit Data         |                                                                                                                      |
| Get price from Forex           | HTTP Request                              | Fetch USD/KRW Forex rate               | Schedule Trigger                 | Edit Fields For Forex Data         |                                                                                                                      |
| Edit Fields For Binance Data   | Set Node                                  | Normalize Binance price data fields    | Get price from Binance           | Merge Data                       | ## 2. Normalization<br>Standardizes data formats from the different exchanges and merges them into a single dataset for the AI to process. |
| Edit Fields For Upbit Data     | Set Node                                  | Normalize Upbit price data fields      | Get price from Upbit             | Merge Data                       |                                                                                                                      |
| Edit Fields For Forex Data     | Set Node                                  | Normalize Forex exchange rate data     | Get price from Forex             | Merge Data                       |                                                                                                                      |
| Merge Data                    | Merge Node                                | Combine normalized data into one set   | Edit Fields For Binance, Upbit, Forex | Analyzer                       |                                                                                                                      |
| Analyzer                      | LangChain Agent                           | AI analysis of price data and report generation | Merge Data                      | Send a Message for Traders         | ## 3. Analysis & Report<br>An AI Agent calculates the price spread, determines profitability, formats a report, and sends it via Gmail. |
| OpenAI Chat Model for Analyzer | LangChain OpenAI Chat Model               | Provides language model for analysis   | Analyzer (ai_languageModel input) | Analyzer (ai_languageModel output) |                                                                                                                      |
| OpenAI Chat Model for Format   | LangChain OpenAI Chat Model               | Formats AI-generated report content    | Structured Output Parser         | Structured Output Parser          |                                                                                                                      |
| Structured Output Parser       | LangChain Output Parser (Structured)     | Parses AI output into structured JSON  | OpenAI Chat Model for Format     | Analyzer                        |                                                                                                                      |
| Send a Message for Traders     | Gmail Node                               | Sends arbitrage report email           | Analyzer                        | None                            | ## 🚀 Crypto Arbitrage Analyzer<br>### How it works<br>This workflow monitors the "Kimchi Premium" (price gap) between Bitcoin prices on Binance (Global) and Upbit (Korea). It fetches real-time data, converts currencies using live Forex rates, and uses an AI Agent to generate and email a strategic arbitrage report.<br><br>### Setup steps<br>1. **OpenAI**: Add your API Key to the **OpenAI Chat Model** nodes.<br>2. **Gmail**: Connect your Google account in the **Send a Message** node and update the `To` email address.<br>3. **Schedule**: Adjust the **Schedule Trigger** node to change how often the analysis runs (currently every 30 mins). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Set to trigger every 30 minutes (minutes interval = 30).

2. **Create HTTP Request Node “Get price from Binance”**  
   - URL: `https://api.binance.com/api/v3/trades`  
   - Method: GET  
   - Query Parameters: `symbol=BTCUSDT`, `limit=1`  
   - Connect input from Schedule Trigger output.

3. **Create HTTP Request Node “Get price from Upbit”**  
   - URL: `https://api.upbit.com/v1/ticker`  
   - Method: GET  
   - Query Parameters: `markets=USDT-BTC,KRW-BTC,KRW-USDT`  
   - Header: `accept: application/json`  
   - Connect input from Schedule Trigger output.

4. **Create HTTP Request Node “Get price from Forex”**  
   - URL: `https://api.manana.kr/exchange/rate/KRW/USD.json`  
   - Method: GET  
   - Connect input from Schedule Trigger output.

5. **Create Set Node “Edit Fields For Binance Data”**  
   - Add fields:  
     - `Exchange` = “Binance” (string)  
     - `Markets` = “USDT-BTC” (string)  
     - `Price` = Expression: `{{$json["price"]}}` (number)  
   - Connect input from “Get price from Binance”.

6. **Create Set Node “Edit Fields For Upbit Data”**  
   - Add fields:  
     - `Exchange` = “Upbit” (string)  
     - `Markets` = Expression: `{{$json["market"]}}` (string)  
     - `Price` = Expression: `{{$json["trade_price"]}}` (number)  
   - Connect input from “Get price from Upbit”.

7. **Create Set Node “Edit Fields For Forex Data”**  
   - Add fields:  
     - `Exchange` = “Forex” (string)  
     - `Markets` = “USDKRW=X” (string)  
     - `Price` = Expression: `{{$json["rate"]}}` (number)  
   - Connect input from “Get price from Forex”.

8. **Create Merge Node “Merge Data”**  
   - Set Number of Inputs: 3  
   - Mode: Append  
   - Connect inputs from the three Set nodes above.  
   - Output will consolidate all normalized data.

9. **Create LangChain Agent Node “Analyzer”**  
   - Text prompt:  
     ```
     Write a report with this price data and send a message to a trader.

     Timestamp:{{$json["timestamp"]}}

     {{$input.all().map((item) => `Exchange: ${item.json.Exchange}\nMarkets: ${item.json.Markets}\nPrice: ${item.json.Price}`).join('\n')}}
     ```  
   - System Message:  
     ```
     You are a helpful analyst that provides information about the price gap of Bitcoin, KRW, USD between two markets(World based market Binance and Korea based market Upbit) and Forex.
     Use <br> and symbols to optimize readability for mail format and do not use \n when split line.
     Don't write any unnecessary messages other than body of mail. Just write a subject and body of the mail in 3~4 paragraphs.
     ```  
   - Output parser enabled (structured)  
   - Connect input from Merge Data node output.

10. **Create LangChain OpenAI Chat Model Node “OpenAI Chat Model for Analyzer”**  
    - Model: `gpt-4o-mini` (or equivalent GPT-4 variant)  
    - Enable code interpreter tool  
    - Connect input from Analyzer node’s `ai_languageModel` input.

11. **Create LangChain OpenAI Chat Model Node “OpenAI Chat Model for Format”**  
    - Model: `gpt-4o-mini`  
    - No code interpreter  
    - Connect input from Structured Output Parser node’s `ai_languageModel` input.

12. **Create LangChain Output Parser Node “Structured Output Parser”**  
    - Enable auto-fix mode  
    - Provide example JSON schema for subject and message fields  
    - Connect input from OpenAI Chat Model for Format output  
    - Connect output to Analyzer node’s `ai_outputParser` input.

13. **Create Gmail Node “Send a Message for Traders”**  
    - Credential: Connect your Gmail OAuth2 credentials  
    - To: set trader email address (e.g., `trader@example.com`)  
    - Subject: Expression `{{$json["output"]["subject"]}}` (from AI output)  
    - Message: Expression `{{$json["output"]["message"]}}` (HTML format)  
    - Connect input from Analyzer node output.

14. **Verify Connections:**  
    - Schedule Trigger → Get price from Binance, Upbit, Forex  
    - Each HTTP Request → respective Set node for normalization  
    - Set nodes → Merge Data  
    - Merge Data → Analyzer (LangChain Agent)  
    - Analyzer ↔ OpenAI Chat Model for Analyzer (language model)  
    - Analyzer → Send a Message for Traders  
    - OpenAI Chat Model for Format → Structured Output Parser  
    - Structured Output Parser → Analyzer (output parser)

15. **Configure Credentials:**  
    - Add OpenAI API key credential for LangChain OpenAI Chat Model nodes  
    - Connect Google Gmail OAuth2 credential to Gmail node

16. **Test Workflow:**  
    - Run manually or wait for scheduled trigger  
    - Verify data fetching, AI analysis, and email delivery

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                | Context or Link                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow monitors the "Kimchi Premium," a well-known arbitrage opportunity between Korean and global BTC markets by comparing Binance and Upbit prices adjusted for USD-KRW exchange rate.                                                                                                                | Crypto arbitrage concept                                                                                      |
| Uses LangChain integration with OpenAI GPT-4 variant (`gpt-4o-mini`) for advanced AI-driven market analysis and report writing, including a structured output parser for reliable email content extraction.                                                                                                   | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/                                           |
| Gmail node requires OAuth2 credentials; ensure account permissions allow sending emails programmatically. Update recipient email to actual trader address.                                                                                                                                                 | Gmail OAuth2 setup in n8n                                                                                    |
| Schedule Trigger interval can be adjusted for more or less frequent arbitrage monitoring (currently 30 minutes). Consider API rate limits for Binance, Upbit, and Forex providers when changing frequency.                                                                                                  | n8n scheduling                                                                                               |
| AI-generated report is formatted with HTML breaks (`<br>`) for email readability. The prompt instructs the AI to avoid unnecessary text except for subject and message content.                                                                                                                             | Prompt engineering best practices                                                                             |
| The Forex API endpoint `https://api.manana.kr/exchange/rate/KRW/USD.json` is a public, free service. Monitor its availability and consider fallback or caching to avoid failures.                                                                                                                          | Forex API reliability                                                                                         |
| Potential risks include API downtime, rate limits, malformed data, and email delivery failures. Implement error handling or notifications for production use.                                                                                                                                            | Error handling recommendations                                                                                 |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.