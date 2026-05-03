Stock Market Analysis & Prediction with GPT, Claude & Gemini via Telegram

https://n8nworkflows.xyz/workflows/stock-market-analysis---prediction-with-gpt--claude---gemini-via-telegram-10460


# Stock Market Analysis & Prediction with GPT, Claude & Gemini via Telegram

### 1. Workflow Overview

This workflow is designed to automate daily stock market analysis and prediction by integrating multiple AI models and data sources, delivering actionable insights via Telegram alerts. It targets traders, portfolio managers, and analysts seeking comprehensive, multi-dimensional investment guidance extracted from real-time stock data, news sentiment, analyst ratings, and social media trends.

The workflow is logically divided into these blocks:

- **1.1 Scheduling & Configuration:** Daily trigger and initial configuration of API keys and parameters.
- **1.2 Data Collection:** Fetching stock price data, news sentiment, analyst ratings, and social media sentiment from external APIs.
- **1.3 Technical & Predictive Analysis:** Computing technical indicators and forecasting future stock trends using custom code nodes.
- **1.4 Sentiment & Ratings Analysis:** Processing news sentiment, analyst ratings, and social media sentiment with dedicated code nodes.
- **1.5 Aggregation & Recommendation Generation:** Merging all analyses to generate a comprehensive weighted recommendation.
- **1.6 Multi-AI Validation:** Preparing inputs and using three AI models (OpenAI GPT, Anthropic Claude, Google Gemini) to validate and cross-check the recommendation.
- **1.7 Consensus Evaluation & Reporting:** Evaluating AI validators’ consensus, formatting the final message, and sending Telegram alerts with detailed insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Configuration

- **Overview:** Triggers the workflow at a scheduled time and sets all configuration parameters, including API keys and stock symbols.
- **Nodes Involved:**  
  - Daily Stock Check  
  - Workflow Configuration

- **Node Details:**

  - **Daily Stock Check**  
    - Type: Schedule Trigger  
    - Configuration: Triggers daily at 9:00 AM (hour 9)  
    - Input: None (start node)  
    - Output: Workflow Configuration  
    - Failure Cases: Trigger misconfiguration or timezone issues  
    - Version: 1.2

  - **Workflow Configuration**  
    - Type: Set  
    - Role: Sets static parameters like stock symbols (e.g., AAPL, GOOGL), API keys for stock, news, analyst ratings, social sentiment APIs, and Telegram chat ID.  
    - Inputs: Daily Stock Check  
    - Outputs: Fetch nodes for data collection  
    - Failure Cases: Missing or invalid configuration values resulting in API request failures  
    - Version: 3.4

#### 2.2 Data Collection

- **Overview:** Fetches required external data for analysis: stock market prices, news articles, analyst ratings, and social media sentiment mentions.
- **Nodes Involved:**  
  - Fetch Stock Data  
  - Fetch News Sentiment  
  - Fetch Analyst Ratings  
  - Fetch Social Media Sentiment

- **Node Details:**

  - **Fetch Stock Data**  
    - Type: HTTP Request  
    - Role: Queries the configured stock API using stock symbols and API key, fetches historical price and volume data.  
    - Query Params: symbols, apikey from Workflow Configuration  
    - Outputs: Analyze Stock Trends  
    - Failure Cases: API unavailability, invalid API key, rate limiting, network errors  
    - Version: 4.3

  - **Fetch News Sentiment**  
    - Type: HTTP Request  
    - Role: Retrieves news articles related to the stock symbols from configured news API with API key.  
    - Query Params: q (stock symbols), apiKey  
    - Outputs: Analyze News Sentiment  
    - Failure Cases: Invalid API key, API limit exceeded, no news data found  
    - Version: 4.3

  - **Fetch Analyst Ratings**  
    - Type: HTTP Request  
    - Role: Fetches analyst rating data from the configured API endpoint using stock symbols and API key/token.  
    - Query Params: symbol, token (apikey)  
    - Outputs: Process Analyst Ratings  
    - Failure Cases: API endpoint errors, incorrect token, missing ratings data  
    - Version: 4.3

  - **Fetch Social Media Sentiment**  
    - Type: HTTP Request  
    - Role: Pulls social media data (e.g., Twitter, StockTwits) for stock symbol from sentiment API.  
    - Query Params: symbol  
    - Outputs: Analyze Social Sentiment  
    - Failure Cases: API errors, no social data, timeout  
    - Version: 4.3

#### 2.3 Technical & Predictive Analysis

- **Overview:** Performs technical analysis (moving averages, RSI, support/resistance) and predicts future price trends with linear regression.
- **Nodes Involved:**  
  - Analyze Stock Trends  
  - Predict Future Trends

- **Node Details:**

  - **Analyze Stock Trends**  
    - Type: Code  
    - Role: Calculates SMA (20, 50, 200), RSI, volume trends, support/resistance levels, and overall trend (Bullish, Bearish, Neutral) from fetched stock data.  
    - Input: Fetch Stock Data  
    - Output: Predict Future Trends  
    - Important Expressions: JS functions for SMA, RSI, support/resistance, trend determination  
    - Failure Cases: Insufficient data points, invalid or missing price data, JS execution errors  
    - Version: 2

  - **Predict Future Trends**  
    - Type: Code  
    - Role: Uses simple linear regression on historical prices to forecast next 7 days and generate buy/hold/sell recommendation with confidence.  
    - Input: Analyze Stock Trends  
    - Output: Combine All Analysis  
    - Key Logic: Linear regression slope and intercept, prediction array, recommendation logic based on slope and expected price change  
    - Failure Cases: Insufficient historical data, division by zero in regression, unexpected data format  
    - Version: 2

#### 2.4 Sentiment & Ratings Analysis

- **Overview:** Processes news sentiment, analyst ratings, and social media sentiment data to extract meaningful scores and summaries.
- **Nodes Involved:**  
  - Analyze News Sentiment  
  - Process Analyst Ratings  
  - Analyze Social Sentiment

- **Node Details:**

  - **Analyze News Sentiment**  
    - Type: Code  
    - Role: Performs simple keyword-based sentiment analysis on news headlines/descriptions, calculates overall sentiment score, distribution, and summary.  
    - Input: Fetch News Sentiment  
    - Output: Combine All Analysis  
    - Key Logic: Positive/negative keyword matching, sentiment score calculation, article classification  
    - Failure Cases: No news data, empty text fields, text encoding issues  
    - Version: 2

  - **Process Analyst Ratings**  
    - Type: Code  
    - Role: Aggregates analyst ratings, counts buy/hold/sell, calculates consensus rating, average/min/max price targets, and confidence level.  
    - Input: Fetch Analyst Ratings  
    - Output: Combine All Analysis  
    - Key Logic: String matching on recommendations, average calculations, consensus determination logic  
    - Failure Cases: Missing ratings array, unexpected data structure, parsing errors  
    - Version: 2

  - **Analyze Social Sentiment**  
    - Type: Code  
    - Role: Analyzes social media posts for sentiment score, engagement, trending hashtags, community mood, and top engaging posts.  
    - Input: Fetch Social Media Sentiment  
    - Output: Combine All Analysis  
    - Key Logic: Positive/negative keyword counting, engagement scoring, hashtag extraction, mood categorization  
    - Failure Cases: No social data, missing fields, text parsing errors  
    - Version: 2

#### 2.5 Aggregation & Recommendation Generation

- **Overview:** Merges the four analysis streams and generates a weighted composite investment recommendation with confidence scores and detailed breakdown.
- **Nodes Involved:**  
  - Combine All Analysis  
  - Generate Comprehensive Recommendation

- **Node Details:**

  - **Combine All Analysis**  
    - Type: Merge (Multiple Inputs)  
    - Role: Combines outputs from Predict Future Trends, Analyze News Sentiment, Process Analyst Ratings, Analyze Social Sentiment into a single dataset.  
    - Inputs: Predict Future Trends (technical), Analyze News Sentiment, Process Analyst Ratings, Analyze Social Sentiment  
    - Output: Generate Comprehensive Recommendation  
    - Failure Cases: Mismatched input lengths, missing inputs  
    - Version: 3.2

  - **Generate Comprehensive Recommendation**  
    - Type: Code  
    - Role: Applies weighted scoring to technical, news, analyst, and social sentiment analyses; derives final recommendation and confidence; generates a summary.  
    - Input: Combine All Analysis  
    - Output: Format Telegram Message, Prepare AI Validation Input  
    - Key Expressions: Weighted score calculation, recommendation thresholds, detailed breakdown object, summary text  
    - Failure Cases: Missing data fields, division by zero, unexpected input shapes  
    - Version: 2

#### 2.6 Multi-AI Validation

- **Overview:** Prepares input prompt and sends it to three AI models (OpenAI GPT, Anthropic Claude, Google Gemini) to independently validate the generated recommendation.
- **Nodes Involved:**  
  - Prepare AI Validation Input  
  - AI Validator 1 - OpenAI  
  - AI Validator 2 - Anthropic  
  - AI Validator 3 - Gemini  
  - Combine AI Validations

- **Node Details:**

  - **Prepare AI Validation Input**  
    - Type: Set  
    - Role: Constructs a formatted prompt string including stock symbol, price, recommendation, confidence, technical and sentiment data for AI validation.  
    - Input: Generate Comprehensive Recommendation  
    - Output: AI Validator nodes  
    - Failure Cases: Missing or undefined fields in input data  
    - Version: 3.4

  - **AI Validator 1 - OpenAI**  
    - Type: Langchain Chain LLM  
    - Role: Sends validation prompt to OpenAI GPT-4.1-mini model for assessment.  
    - Input: Prepare AI Validation Input  
    - Output: Combine AI Validations  
    - Credentials: OpenAI API key  
    - Failure Cases: API errors, rate limits, prompt formatting issues  
    - Version: 1.7

  - **AI Validator 2 - Anthropic**  
    - Type: Langchain Chain LLM  
    - Role: Sends prompt to Anthropic Claude Sonnet 4 model for independent validation.  
    - Input: Prepare AI Validation Input  
    - Output: Combine AI Validations  
    - Credentials: Anthropic API key  
    - Failure Cases: API errors, authentication, rate limits  
    - Version: 1.7

  - **AI Validator 3 - Gemini**  
    - Type: Langchain Chain LLM  
    - Role: Sends prompt to Google Gemini model for third-party validation.  
    - Input: Prepare AI Validation Input  
    - Output: Combine AI Validations  
    - Credentials: Google Gemini API key  
    - Failure Cases: API availability, authentication, response delays  
    - Version: 1.7

  - **Combine AI Validations**  
    - Type: Merge (Multiple Inputs)  
    - Role: Aggregates the three AI validators’ responses for consensus evaluation.  
    - Inputs: AI Validator 1, 2, 3  
    - Output: Evaluate AI Consensus  
    - Failure Cases: Missing or partial AI responses  
    - Version: 3.2

#### 2.7 Consensus Evaluation & Reporting

- **Overview:** Evaluates AI validators’ consensus, determines agreement level, formats a detailed Telegram message, and sends alert to the configured chat.
- **Nodes Involved:**  
  - Evaluate AI Consensus  
  - Format Telegram Message  
  - Send Telegram Alert

- **Node Details:**

  - **Evaluate AI Consensus**  
    - Type: Code  
    - Role: Analyzes AI recommendations, computes average score, agreement level (Full, Partial, Divergent), average confidence; outputs consensus summary.  
    - Input: Combine AI Validations  
    - Output: Format Telegram Message  
    - Failure Cases: Empty input, inconsistent AI responses  
    - Version: 2

  - **Format Telegram Message**  
    - Type: Set  
    - Role: Prepares a rich-text formatted message combining all analysis results, predictions, sentiments, ratings, AI consensus, and final recommendation with disclaimers.  
    - Input: Evaluate AI Consensus + other analysis nodes (via expressions)  
    - Output: Send Telegram Alert  
    - Failure Cases: Expression evaluation errors, missing data fields  
    - Version: 3.4

  - **Send Telegram Alert**  
    - Type: Telegram  
    - Role: Sends the formatted message to the specified Telegram chat ID using Telegram Bot API credentials.  
    - Input: Format Telegram Message  
    - Output: None (terminal node)  
    - Credentials: Telegram Bot API token  
    - Failure Cases: Invalid chat ID, bot permissions, network errors  
    - Version: 1.2

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                                  | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                     |
|-----------------------------|-----------------------------|-------------------------------------------------|-------------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Daily Stock Check            | Schedule Trigger            | Initiates daily workflow run                     | None                                | Workflow Configuration             | ## Introduction Automates stock market analysis using multiple AI models...                     |
| Workflow Configuration       | Set                        | Sets API keys, stock symbols, and parameters    | Daily Stock Check                   | Fetch Stock Data, Fetch News Sentiment, Fetch Analyst Ratings, Fetch Social Media Sentiment | ## Prerequisites - Stock data API (Alpha Vantage, Yahoo Finance)...                             |
| Fetch Stock Data             | HTTP Request               | Retrieves stock price and volume data            | Workflow Configuration              | Analyze Stock Trends               |                                                                                                |
| Analyze Stock Trends         | Code                       | Calculates technical indicators and trend       | Fetch Stock Data                   | Predict Future Trends              |                                                                                                |
| Predict Future Trends        | Code                       | Predicts price trends and generates recommendations | Analyze Stock Trends              | Combine All Analysis              |                                                                                                |
| Fetch News Sentiment         | HTTP Request               | Retrieves stock-related news articles            | Workflow Configuration              | Analyze News Sentiment            |                                                                                                |
| Analyze News Sentiment       | Code                       | Performs sentiment analysis on news data         | Fetch News Sentiment                | Combine All Analysis              |                                                                                                |
| Fetch Analyst Ratings        | HTTP Request               | Retrieves analyst ratings                         | Workflow Configuration              | Process Analyst Ratings           |                                                                                                |
| Process Analyst Ratings      | Code                       | Aggregates and analyzes analyst ratings          | Fetch Analyst Ratings               | Combine All Analysis              |                                                                                                |
| Fetch Social Media Sentiment | HTTP Request               | Retrieves social media data                        | Workflow Configuration              | Analyze Social Sentiment          |                                                                                                |
| Analyze Social Sentiment     | Code                       | Analyzes social media sentiment and engagement   | Fetch Social Media Sentiment        | Combine All Analysis              |                                                                                                |
| Combine All Analysis         | Merge                      | Combines technical, news, ratings, and social sentiment data | Predict Future Trends, Analyze News Sentiment, Process Analyst Ratings, Analyze Social Sentiment | Generate Comprehensive Recommendation |                                                                                                |
| Generate Comprehensive Recommendation | Code               | Generates weighted overall recommendation         | Combine All Analysis                | Format Telegram Message, Prepare AI Validation Input |                                                                                                |
| Prepare AI Validation Input  | Set                        | Prepares formatted input prompt for AI validators | Generate Comprehensive Recommendation | AI Validator 1, AI Validator 2, AI Validator 3 |                                                                                                |
| AI Validator 1 - OpenAI      | Langchain Chain LLM        | Validates recommendation using OpenAI GPT        | Prepare AI Validation Input         | Combine AI Validations            |                                                                                                |
| AI Validator 2 - Anthropic   | Langchain Chain LLM        | Validates recommendation using Anthropic Claude  | Prepare AI Validation Input         | Combine AI Validations            |                                                                                                |
| AI Validator 3 - Gemini      | Langchain Chain LLM        | Validates recommendation using Google Gemini     | Prepare AI Validation Input         | Combine AI Validations            |                                                                                                |
| Combine AI Validations       | Merge                      | Aggregates AI validation results                  | AI Validator 1, 2, 3                | Evaluate AI Consensus             |                                                                                                |
| Evaluate AI Consensus        | Code                       | Determines consensus recommendation and confidence | Combine AI Validations              | Format Telegram Message           |                                                                                                |
| Format Telegram Message      | Set                        | Formats full report message for Telegram alert   | Evaluate AI Consensus + others      | Send Telegram Alert               |                                                                                                |
| Send Telegram Alert          | Telegram                   | Sends formatted stock analysis alert via Telegram | Format Telegram Message             | None                            |                                                                                                |
| Sticky Note                 | Sticky Note                | Prerequisites, Use Cases, Benefits, Customization | None                              | None                            | ## Prerequisites - Stock data API (Alpha Vantage, Yahoo Finance)...                             |
| Sticky Note1                | Sticky Note                | Introduction, Workflow Template, How It Works    | None                              | None                            | ## Introduction Automates stock market analysis using multiple AI models...                     |
| Sticky Note2                | Sticky Note                | Workflow Steps, Setup Instructions                | None                              | None                            | ## Workflow Steps 1. Data Collection... 2. AI Analysis... 6. Alert Delivery...                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 9:00 AM (hour 9)  

2. **Create Set Node for Workflow Configuration**  
   - Add string parameters for:  
     - `stockSymbols` (e.g., "AAPL,GOOGL,MSFT")  
     - `apiKey` (stock API key)  
     - `apiUrl` (stock API endpoint)  
     - `telegramChatId` (Telegram chat ID)  
     - `newsApiKey` (news API key)  
     - `newsApiUrl` (news API endpoint)  
     - `analystRatingsApiUrl` (analyst ratings API URL)  
     - `socialSentimentApiUrl` (social sentiment API URL)  

3. **Connect Schedule Trigger → Workflow Configuration**

4. **Create HTTP Request Node to Fetch Stock Data**  
   - URL: `={{ $json.apiUrl }}` from Workflow Configuration  
   - Query parameters:  
     - `symbols` = `={{ $json.stockSymbols }}`  
     - `apikey` = `={{ $json.apiKey }}`  

5. **Create Code Node "Analyze Stock Trends"**  
   - Implement JS code to calculate SMA20, SMA50, SMA200, RSI, volume trends, support/resistance, and overall trend.  
   - Input: Fetch Stock Data output  

6. **Create Code Node "Predict Future Trends"**  
   - Implement linear regression on historical prices, predict next 7 days, and generate buy/sell/hold recommendation and confidence.  
   - Connect input from Analyze Stock Trends output  

7. **Create HTTP Request Node to Fetch News Sentiment**  
   - URL: `={{ $json.newsApiUrl }}`  
   - Query:  
     - `q` = `={{ $json.stockSymbols }}`  
     - `apiKey` = `={{ $json.newsApiKey }}`  

8. **Create Code Node "Analyze News Sentiment"**  
   - Use keyword-based sentiment analysis on headlines and descriptions.  
   - Input: Fetch News Sentiment output  

9. **Create HTTP Request Node "Fetch Analyst Ratings"**  
   - URL: `={{ $json.analystRatingsApiUrl }}`  
   - Query:  
     - `symbol` = `={{ $json.stockSymbols }}`  
     - `token` = `={{ $json.apiKey }}`  

10. **Create Code Node "Process Analyst Ratings"**  
    - Aggregate buy/hold/sell counts, calculate consensus and average price targets.  
    - Input: Fetch Analyst Ratings output  

11. **Create HTTP Request Node "Fetch Social Media Sentiment"**  
    - URL: `={{ $json.socialSentimentApiUrl }}`  
    - Query:  
      - `symbol` = `={{ $json.stockSymbols }}`  

12. **Create Code Node "Analyze Social Sentiment"**  
    - Analyze sentiment scores, engagement, hashtags, community mood from social media data.  
    - Input: Fetch Social Media Sentiment output  

13. **Create Merge Node "Combine All Analysis"**  
    - Number of inputs: 4  
    - Connect inputs from:  
      - Predict Future Trends  
      - Analyze News Sentiment  
      - Process Analyst Ratings  
      - Analyze Social Sentiment  

14. **Create Code Node "Generate Comprehensive Recommendation"**  
    - Apply weighted scoring (technical 35%, news 25%, analyst 25%, social 15%)  
    - Generate final recommendation, confidence score, and summary.  
    - Input: Combine All Analysis output  

15. **Create Set Node "Prepare AI Validation Input"**  
    - Construct a formatted prompt string using fields from Generate Comprehensive Recommendation output.  
    - Input: Generate Comprehensive Recommendation output  

16. **Create Langchain Chain LLM Nodes for AI Validators**  
    - AI Validator 1 - OpenAI GPT-4.1-mini  
    - AI Validator 2 - Anthropic Claude Sonnet 4  
    - AI Validator 3 - Google Gemini  
    - Credentials: Configure API keys for each respective model  
    - Input: Prepare AI Validation Input output  

17. **Create Merge Node "Combine AI Validations"**  
    - Number of inputs: 3  
    - Connect outputs from all three AI Validator nodes  

18. **Create Code Node "Evaluate AI Consensus"**  
    - Calculate average recommendation score, agreement level, average confidence, and consensus summary.  
    - Input: Combine AI Validations output  

19. **Create Set Node "Format Telegram Message"**  
    - Build a formatted message string combining all analysis, predictions, sentiments, AI consensus, and disclaimers.  
    - Input: Evaluate AI Consensus output and expressions referencing other analysis nodes as needed  

20. **Create Telegram Node "Send Telegram Alert"**  
    - Configure Telegram API credentials (bot token)  
    - Chat ID: from Workflow Configuration  
    - Text: `={{ $json.message }}` from Format Telegram Message output  

21. **Connect Format Telegram Message → Send Telegram Alert**

22. **Connect Evaluate AI Consensus → Format Telegram Message**

23. **Connect Combine AI Validations → Evaluate AI Consensus**

24. **Connect AI Validator nodes → Combine AI Validations**

25. **Connect Prepare AI Validation Input → AI Validator nodes**

26. **Connect Generate Comprehensive Recommendation → Prepare AI Validation Input**

27. **Connect Combine All Analysis → Generate Comprehensive Recommendation**

28. **Connect analysis code nodes and fetch nodes as per above data flow**

29. **Validate all credentials and API keys**

30. **Set error handling and retry logic (optional)**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Prerequisites: Stock data API (Alpha Vantage, Yahoo Finance), News API key, Social media API, OpenAI, Anthropic, Google Gemini API keys, Telegram bot token                   | Sticky Note near Workflow Configuration                                                        |
| Use Cases: Day Trading (real-time volatile analysis), Portfolio Management (daily consensus reports)                                                                        | Sticky Note                                                                                     |
| Benefits: Eliminates manual research, reduces AI hallucination via multi-model validation, 24/7 monitoring, combined multiple data sources                                 | Sticky Note                                                                                     |
| Workflow Steps: 1) Data Collection 2) AI Analysis 3) Report Generation 4) Multi-AI Validation 5) Consensus Building 6) Alert Delivery                                        | Sticky Note                                                                                     |
| Customization Ideas: Add more technical indicators (MACD), crypto analysis, portfolio tracking, alternative notifications (email, Slack), sector-specific analysis           | Sticky Note                                                                                     |
| Disclaimer: Automated analysis for informational purposes only; users should perform their own research before investing                                                  | Included in Telegram message                                                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.