Generate Stock Trading Signals with Gemini 2.5 Pro & TwelveData via Telegram Bot

https://n8nworkflows.xyz/workflows/generate-stock-trading-signals-with-gemini-2-5-pro---twelvedata-via-telegram-bot-9015


# Generate Stock Trading Signals with Gemini 2.5 Pro & TwelveData via Telegram Bot

### 1. Workflow Overview

This workflow automates the generation of stock trading signals by integrating real-time and historical market data with advanced AI analysis, delivering actionable insights via a Telegram bot. It is designed for traders and analysts who want automated, AI-enhanced technical analysis using Gemini 2.5 Pro and TwelveData financial data services.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user stock ticker requests through a Telegram bot.
- **1.2 Data Acquisition:** Fetches multiple technical indicators (EMA 50, EMA 200, RSI, ATR), real-time prices, and OHLC data from TwelveData API.
- **1.3 Data Processing:** Calculates support and resistance levels from OHLC data and organizes all indicator data into a structured JSON.
- **1.4 AI Analysis:** Uses Google Gemini Chat model with LangChain to analyze the technical data and generate trading signals.
- **1.5 Output Delivery:** Sends the final analysis and a 1-day stock chart image back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives user input via Telegram bot, extracts the stock ticker symbol, and initiates the data retrieval process.

**Nodes Involved:**  
- Telegram Trigger  
- Set Stock Ticker  
- Add Twelve Data API Key  

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* Trigger node  
  - *Role:* Listens for incoming Telegram messages to start the workflow.  
  - *Configuration:* Default Telegram Trigger with webhook set up.  
  - *Input:* Incoming Telegram messages.  
  - *Output:* Passes user message data to next node.  
  - *Failures:* Possible webhook misconfiguration, Telegram API downtime.  

- **Set Stock Ticker**  
  - *Type:* Set node  
  - *Role:* Extracts and sets the stock ticker symbol from the Telegram message.  
  - *Configuration:* Uses expressions to parse incoming message text into a standardized ticker variable.  
  - *Input:* Telegram Trigger output.  
  - *Output:* Passes ticker to API key node and chart URL request.  
  - *Failures:* Incorrect message formats or missing ticker symbols can cause empty or invalid outputs.  

- **Add Twelve Data API Key**  
  - *Type:* Set node  
  - *Role:* Injects TwelveData API key into the workflow for subsequent API requests.  
  - *Configuration:* API key stored securely in credentials or environment variables, added to request headers or query parameters.  
  - *Input:* Stock ticker data.  
  - *Output:* Triggers multiple HTTP requests for various indicators and price data.  
  - *Failures:* Invalid or missing API key leads to authorization errors from TwelveData API.

---

#### 1.2 Data Acquisition

**Overview:**  
Fetches multiple technical indicators and price data required for analysis, using the TwelveData API. Includes wait nodes to manage API rate limits and retries.

**Nodes Involved:**  
- OHLC -1Day  
- EMA 50 - 1D  
- EMA 200 -1D  
- RSI -1D  
- ATR -1D  
- Real Time Price  
- Wait -65s  
- Wait1 -65s  

**Node Details:**  

- **OHLC -1Day / EMA 50 - 1D / EMA 200 -1D / RSI -1D / ATR -1D / Real Time Price**  
  - *Type:* HTTP Request nodes  
  - *Role:* Query TwelveData API endpoints for respective financial indicators and price data.  
  - *Configuration:* Each node configured with endpoint URL, query parameters including ticker symbol, interval (daily), and API key.  
  - *Input:* From "Add Twelve Data API Key" node.  
  - *Output:* Each outputs JSON with requested data.  
  - *Failures:* API rate limits, invalid ticker, network failures, or incorrect API key will cause failures or empty data.  
  - *Retry:* Some nodes have retry enabled with 5000 ms wait between tries to handle transient errors.  

- **Wait -65s / Wait1 -65s**  
  - *Type:* Wait nodes  
  - *Role:* Introduce delays between API calls to respect TwelveData’s rate limits.  
  - *Configuration:* Pause of 65 seconds before triggering next HTTP request node.  
  - *Input:* Triggered after initial API calls.  
  - *Output:* Proceed to next API request nodes or merge.  
  - *Failures:* Timeout or workflow stuck if wait node misconfigured.

---

#### 1.3 Data Processing

**Overview:**  
Combines and processes all technical indicator data, calculates support/resistance levels, and formats the data for AI analysis.

**Nodes Involved:**  
- Merge1  
- Calculate Support Resistance  
- Organizing Data  
- Warp JSON  

**Node Details:**  

- **Merge1**  
  - *Type:* Merge node  
  - *Role:* Merges outputs from multiple API requests into a single data stream for processing.  
  - *Configuration:* Default mode (probably 'Merge by Index' or 'Merge by Key').  
  - *Input:* Outputs from ATR, RSI, EMA 200, EMA 50, Real Time Price, and Calculate Support Resistance nodes.  
  - *Output:* Consolidated data array for further processing.  
  - *Failures:* Mismatched data arrays can cause merge errors or missing data.  

- **Calculate Support Resistance**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Computes support and resistance levels using OHLC data from TwelveData.  
  - *Configuration:* Custom JS code parsing OHLC JSON, applying technical formulas to determine levels.  
  - *Input:* OHLC -1Day node output.  
  - *Output:* JSON object with support and resistance levels.  
  - *Failures:* Invalid or missing OHLC data can cause calculation errors or empty output.  

- **Organizing Data**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Formats and organizes all merged technical indicator data into a structured JSON suitable for AI input.  
  - *Input:* Output of Merge1 node.  
  - *Output:* Clean, well-structured JSON.  
  - *Failures:* Code errors or unexpected input data shape can cause failures.  

- **Warp JSON**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Further transforms the JSON, possibly wrapping or renaming keys to match AI agent expectations.  
  - *Input:* Output from Organizing Data node.  
  - *Output:* Final JSON payload for AI agent input.  
  - *Failures:* Similar to Organizing Data node.

---

#### 1.4 AI Analysis

**Overview:**  
Leverages Google Gemini Chat model via LangChain to analyze processed technical data and generate trading signals. Maintains conversational memory for context.

**Nodes Involved:**  
- Simple Memory  
- Google Gemini Chat Model  
- Structured Output Parser  
- Technical Analysis Agent  

**Node Details:**  

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window node  
  - *Role:* Stores conversation or analysis context to improve AI reasoning.  
  - *Configuration:* Default memory size window; stores recent interactions or data.  
  - *Input:* Context and AI agent outputs.  
  - *Output:* Feeds back into AI agent for context-aware generation.  
  - *Failures:* Memory overflow or loss of context if misconfigured.  

- **Google Gemini Chat Model**  
  - *Type:* LangChain Chat Language Model node  
  - *Role:* Invokes Google Gemini 2.5 Pro AI model for natural language understanding and generation.  
  - *Configuration:* Uses stored credentials; default parameters for chat completion.  
  - *Input:* Structured technical data and memory context.  
  - *Output:* Raw AI chat response.  
  - *Failures:* API quota limits, auth errors, network failures.  

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser node  
  - *Role:* Parses Gemini model output into a structured format, extracting actionable signal data.  
  - *Input:* AI model raw output.  
  - *Output:* Parsed JSON with signals and insights.  
  - *Failures:* Parsing errors if AI output format changes or unexpected content.  

- **Technical Analysis Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Orchestrates AI model, memory, and output parser to deliver final analysis.  
  - *Configuration:* Connected to Gemini model, memory, and parser; retry enabled for robustness.  
  - *Input:* Wrapped JSON data from Warp JSON node.  
  - *Output:* Final trading signals and commentary.  
  - *Failures:* Composite failures from underlying AI nodes or malformed input.

---

#### 1.5 Output Delivery

**Overview:**  
Sends the final trading analysis and a 1-day chart image back to the Telegram user.

**Nodes Involved:**  
- Send Final Analysis  
- Get Chart URL  
- Download Chart  
- Send 1D Chart  

**Node Details:**  

- **Send Final Analysis**  
  - *Type:* Telegram node  
  - *Role:* Sends the AI-generated trading signal text back to the user via Telegram.  
  - *Configuration:* Uses Telegram credentials and chat ID extracted from trigger.  
  - *Input:* Output from Technical Analysis Agent.  
  - *Output:* Message delivery confirmation.  
  - *Failures:* Telegram API errors, invalid chat IDs.  

- **Get Chart URL**  
  - *Type:* HTTP Request node  
  - *Role:* Requests a URL for a 1-day stock chart image (likely from TwelveData or another charting service).  
  - *Input:* Stock ticker and API key set earlier.  
  - *Output:* URL of the chart image.  
  - *Failures:* API errors, invalid ticker or parameters.  

- **Download Chart**  
  - *Type:* HTTP Request node  
  - *Role:* Downloads the chart image from the URL.  
  - *Input:* Chart URL from previous node.  
  - *Output:* Binary image data.  
  - *Failures:* Network errors, invalid URLs.  

- **Send 1D Chart**  
  - *Type:* Telegram node  
  - *Role:* Sends the downloaded chart image as a photo to the Telegram user.  
  - *Input:* Binary image from Download Chart node.  
  - *Failures:* Telegram API limits, invalid binary data.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                      | Input Node(s)                           | Output Node(s)                          | Sticky Note                    |
|-------------------------|----------------------------------|------------------------------------|---------------------------------------|---------------------------------------|-------------------------------|
| Telegram Trigger        | Telegram Trigger                 | Receives user input via Telegram   | -                                     | Set Stock Ticker                      |                               |
| Set Stock Ticker        | Set                             | Extracts stock ticker from input   | Telegram Trigger                      | Add Twelve Data API Key, Get Chart URL |                               |
| Add Twelve Data API Key | Set                             | Adds API key for TwelveData calls  | Set Stock Ticker                     | OHLC -1Day, Real Time Price, EMA 50 - 1D, Wait -65s, RSI -1D, Wait1 -65s |                               |
| OHLC -1Day              | HTTP Request                    | Fetch OHLC data 1-day interval     | Add Twelve Data API Key               | Calculate Support Resistance           |                               |
| Calculate Support Resistance | Code                        | Calculates support/resistance levels| OHLC -1Day                         | Merge1                               |                               |
| EMA 50 - 1D             | HTTP Request                    | Fetch EMA 50 indicator             | Add Twelve Data API Key               | Merge1                               |                               |
| EMA 200 -1D             | HTTP Request                    | Fetch EMA 200 indicator            | Wait -65s                           | Merge1                               |                               |
| RSI -1D                 | HTTP Request                    | Fetch RSI indicator                | Add Twelve Data API Key               | Merge1                               |                               |
| ATR -1D                 | HTTP Request                    | Fetch ATR indicator                | Wait1 -65s                         | Merge1                               |                               |
| Real Time Price         | HTTP Request                    | Fetch current price                | Add Twelve Data API Key               | Merge1                               |                               |
| Wait -65s               | Wait                            | Delay for API rate limiting        | Add Twelve Data API Key               | EMA 200 -1D                         |                               |
| Wait1 -65s              | Wait                            | Delay for API rate limiting        | Add Twelve Data API Key               | ATR -1D                             |                               |
| Merge1                  | Merge                           | Combine all indicator data         | Calculate Support Resistance, RSI -1D, EMA 200 -1D, EMA 50 - 1D, ATR -1D, Real Time Price | Organizing Data |                               |
| Organizing Data         | Code                            | Format merged data for AI          | Merge1                              | Warp JSON                            |                               |
| Warp JSON               | Code                            | Wrap JSON for AI agent             | Organizing Data                     | Technical Analysis Agent              |                               |
| Simple Memory           | LangChain Memory Buffer Window  | Maintain AI conversation context  | Technical Analysis Agent             | Google Gemini Chat Model              |                               |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | AI language model for analysis     | Simple Memory                       | Structured Output Parser              |                               |
| Structured Output Parser| LangChain Output Parser         | Parse AI output to structured data | Google Gemini Chat Model             | Technical Analysis Agent              |                               |
| Technical Analysis Agent| LangChain Agent                 | Orchestrate AI analysis            | Warp JSON, Google Gemini Chat Model, Structured Output Parser | Send Final Analysis                  |                               |
| Send Final Analysis     | Telegram                        | Send AI trading signal via Telegram| Technical Analysis Agent             | -                                   |                               |
| Get Chart URL           | HTTP Request                   | Get URL for 1-day stock chart image| Set Stock Ticker                   | Download Chart                      |                               |
| Download Chart          | HTTP Request                   | Download chart image from URL      | Get Chart URL                      | Send 1D Chart                      |                               |
| Send 1D Chart           | Telegram                       | Send chart image via Telegram      | Download Chart                    | -                                   |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Set up Telegram bot credentials and webhook URL.  
   - Purpose: To listen for incoming messages with stock ticker requests.  

2. **Create Set Node "Set Stock Ticker":**  
   - Extract stock ticker symbol from Telegram message text using expression (e.g., `{{$json["message"]["text"]}}`).  
   - Output variable: `ticker` or similar.  
   - Connect input from Telegram Trigger.  

3. **Create Set Node "Add Twelve Data API Key":**  
   - Add TwelveData API key as a parameter or header for subsequent HTTP requests.  
   - Connect input from "Set Stock Ticker".  

4. **Create HTTP Request Nodes for Market Data:**  
   - **OHLC -1Day:** Configure to call TwelveData's OHLC endpoint using `ticker` and API key, daily interval.  
   - **EMA 50 - 1D:** Call TwelveData EMA endpoint, 50-period, daily interval.  
   - **EMA 200 -1D:** Same as above but 200-period.  
   - **RSI -1D:** Call RSI endpoint, daily interval.  
   - **ATR -1D:** Call ATR endpoint, daily interval.  
   - **Real Time Price:** Call real-time price endpoint for the ticker.  
   - Connect all from "Add Twelve Data API Key".  

5. **Create Wait Nodes for Rate Limiting:**  
   - Add two Wait nodes with 65 seconds delay between API calls to avoid rate limit errors.  
   - Connect "Wait -65s" between EMA 50 - 1D and EMA 200 -1D.  
   - Connect "Wait1 -65s" before ATR -1D.  

6. **Create Code Node "Calculate Support Resistance":**  
   - Write JavaScript code to compute support/resistance from OHLC data.  
   - Connect input from OHLC -1Day node.  

7. **Create Merge Node "Merge1":**  
   - Merge outputs of Calculate Support Resistance, RSI -1D, EMA 200 -1D, EMA 50 - 1D, ATR -1D, and Real Time Price nodes.  
   - Set mode to merge by index or default.  

8. **Create Code Node "Organizing Data":**  
   - Consolidate and format merged data into structured JSON suitable for AI input.  
   - Connect input from Merge1.  

9. **Create Code Node "Warp JSON":**  
   - Further wrap or rename JSON keys to match AI agent schema.  
   - Connect input from Organizing Data.  

10. **Set up LangChain Nodes:**  
    - **Simple Memory:** Add memory buffer window to hold context.  
    - **Google Gemini Chat Model:** Configure with Google Gemini 2.5 Pro credentials.  
    - **Structured Output Parser:** Configure to parse Gemini output into structured signals.  
    - **Technical Analysis Agent:** Connect Warp JSON as input, link to memory, language model, and output parser. Enable retry on fail.  

11. **Create Telegram Node "Send Final Analysis":**  
    - Configure to send message to user chat ID from Telegram Trigger.  
    - Connect input from Technical Analysis Agent.  

12. **Create HTTP Request Node "Get Chart URL":**  
    - Request chart image URL for the ticker (1-day interval) from the chart provider.  
    - Connect input from "Set Stock Ticker".  

13. **Create HTTP Request Node "Download Chart":**  
    - Download the chart image binary from URL.  
    - Connect input from Get Chart URL.  

14. **Create Telegram Node "Send 1D Chart":**  
    - Send the downloaded chart image as a photo to the user.  
    - Connect input from Download Chart.  

15. **Test the full workflow:**  
    - Ensure credentials for Telegram, TwelveData API, and Google Gemini are correctly configured.  
    - Run end-to-end test by sending a ticker to Telegram bot and verify outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                   |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow uses LangChain nodes for advanced integration with Google Gemini 2.5 Pro AI model.                   | https://docs.n8n.io/integrations/ai/                             |
| TwelveData API key must have sufficient quota and permissions for all requested endpoints.                    | https://twelvedata.com/docs/                                    |
| Telegram bot token and webhook must be correctly configured to receive and send messages.                     | https://core.telegram.org/bots/api                             |
| Use Wait nodes to respect API rate limits and avoid errors from TwelveData.                                  | Best practice for rate-limited APIs                             |
| Support and resistance calculation code must handle missing or malformed OHLC data gracefully.               | Custom JS code robustness recommended                           |
| AI output parser depends on consistent response formatting from Gemini; changes in AI behavior may require updates. | Monitor AI output formats                                        |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automation workflow. All data processed is legal and public. No illegal, offensive, or protected content is involved. The workflow respects applicable content policies.