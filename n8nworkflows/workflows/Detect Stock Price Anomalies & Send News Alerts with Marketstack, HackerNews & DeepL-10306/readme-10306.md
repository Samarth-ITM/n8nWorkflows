Detect Stock Price Anomalies & Send News Alerts with Marketstack, HackerNews & DeepL

https://n8nworkflows.xyz/workflows/detect-stock-price-anomalies---send-news-alerts-with-marketstack--hackernews---deepl-10306


# Detect Stock Price Anomalies & Send News Alerts with Marketstack, HackerNews & DeepL

### 1. Workflow Overview

This workflow automates the monitoring of stock price anomalies for a specified stock ticker, using Marketstack's end-of-day (EOD) data. It performs a statistical analysis based on a 20-day moving average and ±2 standard deviations (σ) to detect significant deviations (anomalies) in the closing price. Upon detecting an anomaly, it fetches related news headlines from Hacker News, translates them from English to Japanese with DeepL, and sends a bilingual alert message to a Slack channel. If no anomaly is detected, it sends a concise normal status report to Slack.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates daily execution at 09:00 JST.
- **1.2 Stock Data Retrieval:** Pulls recent stock price data from Marketstack.
- **1.3 Anomaly Detection:** Calculates statistical deviation to classify the latest closing price.
- **1.4 Conditional Branching:** Determines if the stock price anomaly condition is met.
- **1.5 Anomaly Notification Path:** If anomaly detected, fetch related news, format, translate, merge, and send alert.
- **1.6 Normal Status Reporting:** If no anomaly, send a normal status message to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
Starts the workflow every day at 09:00 JST to ensure timely stock monitoring.

- **Nodes Involved:**  
  - Daily Check

- **Node Details:**  
  - **Daily Check**  
    - Type: Schedule Trigger  
    - Config: Triggers daily at 9:00 AM (JST). The timezone is implicit based on user setup; adjust if needed.  
    - Inputs: None (trigger node)  
    - Outputs: Triggers "Get Stock Data" node  
    - Failure cases: Trigger misconfiguration, timezone mismatch could cause off-hours runs.

#### 1.2 Stock Data Retrieval

- **Overview:**  
Fetches End-of-Day stock price data for the configured ticker symbol from Marketstack over a specified date range.

- **Nodes Involved:**  
  - Get Stock Data

- **Node Details:**  
  - **Get Stock Data**  
    - Type: Marketstack node  
    - Config:  
      - Symbol: "AMZN" (modifiable)  
      - Date range: from 2025-09-01 to 2025-09-30 (future date placeholders, should be updated to current range)  
      - Latest flag disabled (fetches historical data)  
    - Credentials: Marketstack API key (stored securely in n8n credentials)  
    - Input: Trigger from "Daily Check"  
    - Output: Passes raw stock data JSON array to "Calculate Deviation"  
    - Failure cases: API key authorization errors, rate limits, incorrect date range, symbol typos.

#### 1.3 Anomaly Detection

- **Overview:**  
Processes the stock data to compute a 20-day moving average and standard deviation, then classifies the latest closing price as normal, high anomaly, or low anomaly based on ±2σ thresholds.

- **Nodes Involved:**  
  - Calculate Deviation

- **Node Details:**  
  - **Calculate Deviation**  
    - Type: Code node (JavaScript)  
    - Config:  
      - Uses a fixed window size N=20 for moving average  
      - Sigma multiplier k=2 for ±2σ bounds  
      - Extracts closing prices from Marketstack data  
      - Handles cases where data may be missing or malformed  
      - Computes mean, variance, standard deviation (sigma), and classifies latest close status  
      - **Note:** The code adds 100 to the latest close price (`closes[0] + 100`), which appears as a test or placeholder and should be corrected to `closes[0]` for real use.  
    - Inputs: Marketstack output array  
    - Outputs: JSON object with mean, sigma, upper/lower bounds, latest close, status ("normal", "high (above +2σ)", "low (below -2σ)"), count  
    - Failure cases: No valid closing price data, code errors, unexpected input structure.

#### 1.4 Conditional Branching (Anomaly Check)

- **Overview:**  
Determines whether the latest stock price status is an anomaly (not "normal") and branches the workflow accordingly.

- **Nodes Involved:**  
  - Is Anomaly? (status != "normal")

- **Node Details:**  
  - **Is Anomaly? (status != "normal")**  
    - Type: If node  
    - Condition: Checks if `status` field from previous node is not equal to "normal" (case-sensitive, strict)  
    - True output: Proceeds to anomaly notification path  
    - False output: Proceeds to send a normal report to Slack  
    - Inputs: Output of "Calculate Deviation"  
    - Outputs: Two branches, anomaly and normal  
    - Failure cases: Missing or malformed `status` field, expression evaluation errors.

#### 1.5 Anomaly Notification Path

- **Overview:**  
When an anomaly is detected, fetch related news articles from Hacker News using the stock/company keyword, format the news, translate it to Japanese using DeepL, merge original and translated texts, and send a bilingual alert message to Slack.

- **Nodes Involved:**  
  - Add Symbol Field  
  - Build News Keyword  
  - Get Related News  
  - Compose Slack Message  
  - Translate News  
  - Merge Original + Translated  
  - Send Alert to Slack

- **Node Details:**  

  - **Add Symbol Field**  
    - Type: Set node  
    - Config: Extracts the `symbol` from "Get Stock Data" output and assigns it for downstream use  
    - Input: True branch from "Is Anomaly?"  
    - Output: Passes `symbol` to "Build News Keyword"  
    - Failure cases: Missing symbol, empty input.

  - **Build News Keyword**  
    - Type: Code node (JavaScript)  
    - Config:  
      - Maps stock ticker symbols to company names (e.g., AMZN → Amazon)  
      - Outputs `keyword` and `symbol` fields for news search  
      - If symbol is missing, returns an error JSON  
    - Input: Output from "Add Symbol Field"  
    - Output: Passes keyword to "Get Related News"  
    - Failure cases: Unmapped symbol, missing symbol.

  - **Get Related News**  
    - Type: Hacker News node  
    - Config: Searches Hacker News for articles using the keyword from previous node  
    - Input: From "Build News Keyword"  
    - Output: List of news items to "Compose Slack Message"  
    - Failure cases: API limits, no results, network errors.

  - **Compose Slack Message**  
    - Type: Code node (JavaScript)  
    - Config:  
      - Extracts top 3 news items from Hacker News results  
      - Cleans HTML tags from titles and summaries  
      - Formats a message combining title, summary, and URL for each item  
      - Outputs combined `message` string for translation and Slack  
    - Input: "Get Related News" output  
    - Output: Passes formatted message to "Translate News" and also directly to "Merge Original + Translated"  
    - Failure cases: Missing fields, malformed data.

  - **Translate News**  
    - Type: DeepL node  
    - Config:  
      - Translates the `message` from English to Japanese (`translateTo: "JA"`)  
      - Requires valid DeepL API credentials  
    - Input: From "Compose Slack Message"  
    - Output: Translated text to "Merge Original + Translated"  
    - Failure cases: API key issues, rate limits, translation errors.

  - **Merge Original + Translated**  
    - Type: Merge node  
    - Config: Merges original English and translated Japanese texts into a single payload for Slack  
    - Input: Receives original message and translated message via two inputs  
    - Output: Passes combined message to "Send Alert to Slack"  
    - Failure cases: Mismatched input counts, merge conflicts.

  - **Send Alert to Slack**  
    - Type: Slack node  
    - Config:  
      - Posts a bilingual alert message including original and translated texts, plus statistical details  
      - Uses OAuth2 credentials for Slack with bot permission  
      - Posts to configured Slack channel (channel ID "CKUCBTG0H")  
    - Input: From "Merge Original + Translated"  
    - Output: None  
    - Failure cases: Auth failure, channel permission issues, rate limits.

#### 1.6 Normal Status Reporting

- **Overview:**  
If no anomaly is detected, sends a concise normal status report message with current price statistics to Slack.

- **Nodes Involved:**  
  - Send Normal Report to Slack

- **Node Details:**  
  - **Send Normal Report to Slack**  
    - Type: Slack node  
    - Config:  
      - Sends a text message indicating no anomaly and shows latest close with mean and sigma values  
      - Uses OAuth2 Slack credentials (same as alert)  
      - Posts to Slack channel "CKUCBTG0H"  
    - Input: False branch of "Is Anomaly?"  
    - Output: None  
    - Failure cases: Slack API errors, permission issues.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                          | Input Node(s)               | Output Node(s)                         | Sticky Note                                                                                                         |
|----------------------------|-------------------------|----------------------------------------|-----------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Daily Check                | Schedule Trigger        | Start workflow daily at 09:00 JST       | None                        | Get Stock Data                       | ## Daily Check (09:00 JST) Starts the workflow every morning at 09:00 JST. Adjust schedule/timezone as needed.      |
| Get Stock Data             | Marketstack             | Fetch EOD stock price data               | Daily Check                 | Calculate Deviation                  | ## Get Stock Data (Marketstack) Retrieves latest EOD prices; edit symbol and date range. Keep limit ≥ 20.            |
| Calculate Deviation        | Code                    | Compute 20-day mean, std dev, detect anomaly | Get Stock Data              | Is Anomaly? (status != "normal")    | ## Calculate Deviation Computes 20-day mean and std dev; classifies latest close as normal, high, or low anomaly.    |
| Is Anomaly? (status != "normal") | If                  | Branch based on anomaly detection        | Calculate Deviation         | Add Symbol Field (true), Send Normal Report to Slack (false) | ## Is Anomaly? Branches to news if anomaly detected (status != "normal").                                            |
| Add Symbol Field           | Set                     | Extract symbol for news keyword building | Is Anomaly? (true)          | Build News Keyword                  |                                                                                                                     |
| Build News Keyword         | Code                    | Map ticker to company name for news search | Add Symbol Field            | Get Related News                   |                                                                                                                     |
| Get Related News           | Hacker News             | Fetch news articles matching keyword    | Build News Keyword          | Compose Slack Message              | ## Get Related News Searches articles by ticker/company keyword. Can switch to NewsAPI/Google RSS.                   |
| Compose Slack Message      | Code                    | Format top news items for translation and Slack | Get Related News            | Translate News, Merge Original + Translated | ## Format News Cleans HTML and formats top news items.                                                               |
| Translate News             | DeepL                   | Translate news message from EN to JA    | Compose Slack Message       | Merge Original + Translated        | ## Translate News DeepL translation EN → JA. Change target_lang if needed.                                           |
| Merge Original + Translated| Merge                   | Combine original English and translated Japanese texts | Translate News, Compose Slack Message | Send Alert to Slack                | ## Merge Original + Translated Combines original and translated messages.                                            |
| Send Alert to Slack        | Slack                   | Send bilingual anomaly alert to Slack   | Merge Original + Translated | None                             | ## Send Alert to Slack Sends bilingual alert with stats and news.                                                    |
| Send Normal Report to Slack| Slack                   | Send normal status report to Slack       | Is Anomaly? (false)         | None                             | ## Send Normal Report to Slack Sends concise “no anomaly” message with stats.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node named "Daily Check"**  
   - Set to trigger daily at 09:00 (adjust timezone as needed).

2. **Create Marketstack node named "Get Stock Data"**  
   - Configure credentials with your Marketstack API key.  
   - Set symbol to your target ticker (default "AMZN").  
   - Set date range for historical data (e.g., last 1 month).  
   - Disable "latest" flag to fetch multiple days.  
   - Connect "Daily Check" → "Get Stock Data".

3. **Create Code node named "Calculate Deviation"**  
   - Copy provided JavaScript code to calculate 20-day mean and std dev, classify latest close status.  
   - Ensure to correct or review the line adding 100 to latest close price (likely a placeholder).  
   - Connect "Get Stock Data" → "Calculate Deviation".

4. **Create If node named "Is Anomaly? (status != \"normal\")"**  
   - Condition: `{{$json["status"]}}` != `"normal"` (case-sensitive, strict).  
   - Connect "Calculate Deviation" → "Is Anomaly?".

5. **Create Set node named "Add Symbol Field"**  
   - Assign a new field "symbol" from `"{{$item(0).$node["Get Stock Data"].json["symbol"]}}"`.  
   - Connect "Is Anomaly?" true output → "Add Symbol Field".

6. **Create Code node named "Build News Keyword"**  
   - Use JavaScript to map ticker symbols to company names (e.g., AMZN → Amazon).  
   - Outputs object with fields `keyword` and `symbol`.  
   - Connect "Add Symbol Field" → "Build News Keyword".

7. **Create Hacker News node named "Get Related News"**  
   - Set resource to "all".  
   - Set keyword to expression: `{{$json["keyword"]}}` from previous node.  
   - Connect "Build News Keyword" → "Get Related News".

8. **Create Code node named "Compose Slack Message"**  
   - Format top 3 news items with cleaned titles, summaries, and URLs.  
   - Output combined message string in `message` field.  
   - Connect "Get Related News" → "Compose Slack Message".

9. **Create DeepL node named "Translate News"**  
   - Provide DeepL API credentials.  
   - Set text to translate: `{{$json["message"]}}` from "Compose Slack Message".  
   - Set target language to Japanese ("JA").  
   - Connect "Compose Slack Message" → "Translate News".

10. **Create Merge node named "Merge Original + Translated"**  
    - Configure to merge inputs by index (binary merge).  
    - Connect "Compose Slack Message" (original message) to first input.  
    - Connect "Translate News" (translated message) to second input.

11. **Create Slack node named "Send Alert to Slack"**  
    - Configure Slack OAuth2 credentials with bot token and channel permissions.  
    - Set channel to desired Slack channel (e.g., "general" channel ID).  
    - Message text:  
      ```
      🌐 *Original (English)*  
      {{$json["message"]}}

      ---

      🇯🇵 *Translated (Japanese)*  
      {{$json["text"]}}
      ```  
    - Connect "Merge Original + Translated" → "Send Alert to Slack".

12. **Create Slack node named "Send Normal Report to Slack"**  
    - Use same Slack credentials as above.  
    - Channel: same as alert node.  
    - Message text:  
      ```
      ✅ 異常なし この銘柄の終値は安定しています。 現在値：{{ $('Calculate Deviation').item.json.latest }}（平均 {{ $('Calculate Deviation').item.json.mean }} ± {{ $('Calculate Deviation').item.json.sigma }})
      ```  
    - Connect "Is Anomaly?" false output → "Send Normal Report to Slack".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow includes customization tips such as monitoring multiple tickers by duplicating stock data nodes, swapping Hacker News with NewsAPI or Google News RSS, and using alternative messaging platforms like Telegram or Discord. | Sticky Note 2 content in workflow |
| To optimize DeepL translation costs, consider adding an IF node before translation to skip translation if the news is already in Japanese. | Sticky Note 2 content in workflow |
| For user convenience, add a Set node to centralize configurable variables such as `symbol`, `days` (N), `sigmaK` (k), `newsQuery`, and `slackChannel`. | Sticky Note 2 content in workflow |
| Credentials for Marketstack, DeepL, and Slack must be securely configured in n8n’s credential manager and referenced in respective nodes. | Sticky Note 5 content in workflow |
| The workflow is designed for daily monitoring and alerting but can be adapted for other schedules or multiple stocks with minor modifications. | Sticky Note 6 content in workflow |
| Slack OAuth2 credentials require bot token with permissions to post messages to the selected channel. | Slack node configuration notes |
| Marketstack API requires a valid API key with access to historical stock data. | Marketstack node configuration notes |
| DeepL API key is required for translation; usage costs may apply. | DeepL node configuration notes |
| To extend the news source beyond Hacker News, n8n supports NewsAPI, Google RSS, or custom HTTP requests. | Sticky Note 9 content in workflow |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal or protected elements. All data handled are legal and publicly available.