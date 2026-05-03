Real-Time Oil Price Crisis Detection with Qwq-32b AI and Multi-Channel Alerts

https://n8nworkflows.xyz/workflows/real-time-oil-price-crisis-detection-with-qwq-32b-ai-and-multi-channel-alerts-10763


# Real-Time Oil Price Crisis Detection with Qwq-32b AI and Multi-Channel Alerts

### 1. Workflow Overview

This workflow performs **Real-Time Oil Price Crisis Detection and Smart Alerting**, designed for energy market analysts, traders, and operational risk managers. Its purpose is to continuously monitor multiple data streams related to oil markets, detect early signs of crises through statistical and AI-driven analysis, and send multi-channel alerts to stakeholders.

The workflow is logically structured into the following blocks:

- **1.1 Scheduled Data Collection:** Periodic triggering of data fetches from various oil market sources.
- **1.2 Data Aggregation and Cleaning:** Merging and preprocessing heterogeneous data inputs into a unified format.
- **1.3 Statistical Anomaly Detection:** Quantitative analysis identifying unusual market behaviors.
- **1.4 AI-Driven Geopolitical Risk Analysis:** Using a large language model to analyze geopolitical and supply risks.
- **1.5 Crisis Risk Scoring:** Combining statistical and AI results into a composite crisis risk score.
- **1.6 Alert Routing and Formatting:** Classifying alerts by severity and formatting messages for different channels.
- **1.7 Multi-Channel Alert Delivery:** Sending notifications via email, Slack, and dashboard API.
- **1.8 Alert Storage:** Persisting alert history in a PostgreSQL database.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Collection

- **Overview:**  
Triggers the workflow every 5 minutes to initiate data retrieval from multiple oil market-related APIs.

- **Nodes Involved:**  
  - Schedule Every 5 Minutes  
  - Workflow Configuration  
  - Fetch Oil Price Data (API)  
  - Fetch OPEC Reports  
  - Fetch Shipping Data  
  - Fetch News Feed

- **Node Details:**

  - **Schedule Every 5 Minutes**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow on a 5-minute interval.  
    - Config: Interval set to 5 minutes.  
    - Inputs: None (trigger node)  
    - Outputs: Workflow Configuration node.

  - **Workflow Configuration**  
    - Type: Set  
    - Role: Contains all configuration parameters and API endpoint URLs as variables.  
    - Config: Contains placeholders for API URLs, alert recipients, slack channel, and threshold values for alert levels.  
    - Key Variables:  
      - oilPriceApiUrl, opecReportsUrl, shippingDataUrl, newsApiUrl, dashboardApiUrl  
      - alertEmailRecipients, slackChannel  
      - infoThreshold (30), warningThreshold (60), criticalThreshold (85)  
    - Inputs: Trigger node  
    - Outputs: Four HTTP Request nodes fetching data.

  - **Fetch Oil Price Data (API)**  
    - Type: HTTP Request  
    - Role: Retrieves latest oil price data from configured API endpoint.  
    - Config: GET request; Accept header set to application/json; URL dynamically from config.  
    - Inputs: Workflow Configuration  
    - Outputs: Merge All Data Sources node.

  - **Fetch OPEC Reports**  
    - Type: HTTP Request  
    - Role: Retrieves official OPEC production reports.  
    - Config: GET request; Accept: application/json; URL from config.  
    - Inputs: Workflow Configuration  
    - Outputs: Merge All Data Sources node.

  - **Fetch Shipping Data**  
    - Type: HTTP Request  
    - Role: Retrieves global shipping and supply chain data.  
    - Config: GET request; Accept: application/json; URL from config.  
    - Inputs: Workflow Configuration  
    - Outputs: Merge All Data Sources node.

  - **Fetch News Feed**  
    - Type: HTTP Request  
    - Role: Fetches relevant news articles for sentiment analysis.  
    - Config: GET request; Accept: application/json; URL from config.  
    - Inputs: Workflow Configuration  
    - Outputs: Merge All Data Sources node.

- **Potential Failures:**  
  - API endpoint downtime or incorrect URLs  
  - Network timeouts  
  - Authentication failures if APIs require credentials (currently placeholders)  
  - Data format or schema changes in upstream APIs  

---

#### 1.2 Data Aggregation and Cleaning

- **Overview:**  
Combines data from all sources into a single dataset and standardizes fields such as timestamps, oil prices, volumes, and news sentiment for downstream analysis.

- **Nodes Involved:**  
  - Merge All Data Sources  
  - Data Cleaning & Preprocessing

- **Node Details:**

  - **Merge All Data Sources**  
    - Type: Merge  
    - Role: Combines data streams from oil prices, OPEC reports, shipping, and news into a unified dataset.  
    - Config: Combine mode "combineAll" to aggregate all incoming items.  
    - Inputs: All four fetch nodes  
    - Outputs: Data Cleaning & Preprocessing node

  - **Data Cleaning & Preprocessing**  
    - Type: Code (JavaScript)  
    - Role: Parses and normalizes raw data:  
      - Standardizes timestamps to ISO format  
      - Extracts oil prices (supports multiple price keys)  
      - Extracts volume and shipping data  
      - Performs simple keyword-based sentiment analysis on news text  
      - Extracts geopolitical indicators (region, country)  
      - Adds data quality flags and completeness score  
    - Key Expressions:  
      - Uses JavaScript loops and conditionals to handle missing fields and parsing errors  
      - Sentiment keywords for positive and negative bias  
    - Inputs: Merged data  
    - Outputs: Statistical Anomaly Detection and AI Trend & Geopolitical Analyzer nodes

- **Potential Failures:**  
  - Unexpected or missing data fields causing parsing errors  
  - Incorrect or malformed timestamps  
  - Sentiment analysis might miss nuances due to keyword matching limitations  
  - Very incomplete data may reduce anomaly detection accuracy

---

#### 1.3 Statistical Anomaly Detection

- **Overview:**  
Computes statistical metrics and detects anomalies such as sudden price changes, volume spikes, and high volatility using Z-score, moving averages, and volatility measures.

- **Nodes Involved:**  
  - Statistical Anomaly Detection

- **Node Details:**

  - **Statistical Anomaly Detection**  
    - Type: Code (JavaScript)  
    - Role: Performs calculations:  
      - Mean, standard deviation, Z-scores for price and volume  
      - Moving average (window size 5) for price  
      - Price volatility as coefficient of variation  
      - Flags anomalies based on thresholds (Z-score > 2, >5% price change, volatility > 10%)  
      - Composes composite anomaly score (0-100) weighted by different factors  
      - Determines if anomaly conditions are met  
    - Inputs: Cleaned data array  
    - Outputs: Calculate Crisis Risk Score node

- **Potential Failures:**  
  - Insufficient data points for statistical calculations (less than window size)  
  - Division by zero if standard deviation or mean is zero (handled in code)  
  - Outlier data skewing results—no explicit outlier filtering implemented

---

#### 1.4 AI-Driven Geopolitical Risk Analysis

- **Overview:**  
Uses a large language model to interpret combined market data for geopolitical risks, supply disruptions, and crisis indicators, producing a risk assessment score and key risk factors.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - AI Trend & Geopolitical Analyzer

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: LangChain LM Chat OpenRouter  
    - Role: Calls the Qwen/Qwq-32b model via OpenRouter API.  
    - Config: Model set to "qwen/qwq-32b", no additional options.  
    - Credentials: OpenRouter API key configured.  
    - Inputs: None directly; invoked by AI Trend & Geopolitical Analyzer.  
    - Outputs: AI Trend & Geopolitical Analyzer

  - **AI Trend & Geopolitical Analyzer**  
    - Type: LangChain Agent  
    - Role: Sends prompt to LLM to analyze oil market data with focus on:  
      - News sentiment, geopolitical events  
      - OPEC production changes  
      - Shipping disruptions  
      - Historical patterns  
    - Output includes risk assessment score (0-100) and key risk factors extracted by LLM.  
    - Inputs: Cleaned data from Data Cleaning node, passed as JSON string to prompt.  
    - Outputs: Calculate Crisis Risk Score node

- **Potential Failures:**  
  - API limits or timeouts from OpenRouter or LangChain nodes  
  - Model misinterpretation or hallucination risks due to ambiguous data  
  - Network issues affecting LLM calls  
  - Need for valid OpenRouter credentials

---

#### 1.5 Crisis Risk Scoring

- **Overview:**  
Combines statistical anomaly scores and AI-derived geopolitical risk scores into a weighted composite crisis risk score and assigns alert levels.

- **Nodes Involved:**  
  - Calculate Crisis Risk Score

- **Node Details:**

  - **Calculate Crisis Risk Score**  
    - Type: Code (JavaScript)  
    - Role:  
      - Extracts anomaly score, geopolitical risk score, price volatility, and volume change  
      - Normalizes scores to 0-100  
      - Applies weighting: 40% statistical anomaly, 40% geopolitical, 10% volatility, 10% volume  
      - Computes final crisis risk score (0-100)  
      - Determines alert level: info (<40), warning (40-69), critical (≥70)  
      - Prepares detailed score breakdown and timestamp  
    - Inputs: Outputs from Statistical Anomaly Detection and AI Trend & Geopolitical Analyzer nodes  
    - Outputs: Route by Alert Level node

- **Potential Failures:**  
  - Missing or null input scores causing NaN results (handled by min/max normalization)  
  - Misalignment if inputs are not properly merged or synchronized

---

#### 1.6 Alert Routing and Formatting

- **Overview:**  
Routes alerts based on risk score to different severity channels and formats messages accordingly for downstream delivery.

- **Nodes Involved:**  
  - Route by Alert Level  
  - Format Info Alert  
  - Format Warning Alert  
  - Format Critical Alert

- **Node Details:**

  - **Route by Alert Level**  
    - Type: Switch  
    - Role: Routes data based on riskScore compared to thresholds from config node.  
    - Config:  
      - Info: riskScore < warningThreshold (default 60)  
      - Warning: riskScore ≥ warningThreshold and < criticalThreshold (default 85)  
      - Critical: riskScore ≥ criticalThreshold  
    - Inputs: Calculate Crisis Risk Score  
    - Outputs: To respective Format Alert nodes

  - **Format Info Alert**  
    - Type: Set  
    - Role: Prepares alert message for informational alerts.  
    - Fields set: alertLevel = "INFO", alertTitle, alertMessage, timestamp, color (#17a2b8)  
    - Inputs: Route by Alert Level (Info branch)  
    - Outputs: Send to Dashboard API

  - **Format Warning Alert**  
    - Type: Set  
    - Role: Prepares warning-level alert with detailed message including risk factors.  
    - Fields set: alertLevel = "WARNING", alertTitle with warning emoji, alertMessage, timestamp, color (#ffc107)  
    - Inputs: Route by Alert Level (Warning branch)  
    - Outputs: Send Email Alert, Send Slack Alert, Send to Dashboard API

  - **Format Critical Alert**  
    - Type: Set  
    - Role: Prepares critical alert message, emphasizing urgency and factors.  
    - Fields set: alertLevel = "CRITICAL", alertTitle with siren emoji, alertMessage, timestamp, color (#dc3545)  
    - Inputs: Route by Alert Level (Critical branch)  
    - Outputs: Send Email Alert, Send Slack Alert, Send to Dashboard API

- **Potential Failures:**  
  - Incorrect riskScore property naming causing routing errors  
  - Missing alert recipient or channel configs causing delivery failures

---

#### 1.7 Multi-Channel Alert Delivery

- **Overview:**  
Sends formatted alerts via email, Slack, and updates a dashboard through API calls.

- **Nodes Involved:**  
  - Send Email Alert  
  - Send Slack Alert  
  - Send to Dashboard API

- **Node Details:**

  - **Send Email Alert**  
    - Type: Gmail  
    - Role: Sends alert emails to configured recipients.  
    - Config: Uses Gmail OAuth2 credential; recipients from config node; HTML formatted messages including alert title, message, timestamp, and full JSON details.  
    - Inputs: Format Warning Alert, Format Critical Alert  
    - Outputs: None

  - **Send Slack Alert**  
    - Type: Slack  
    - Role: Posts alert messages to configured Slack channel with colored attachments.  
    - Config: Channel ID from config; OAuth2 Slack credential; attachment includes alert title, message, timestamp, color coding.  
    - Inputs: Format Warning Alert, Format Critical Alert  
    - Outputs: None

  - **Send to Dashboard API**  
    - Type: HTTP Request  
    - Role: Sends alert data to an external dashboard API for visualization and tracking.  
    - Config: POST request; JSON body; Content-Type: application/json; URL from config node.  
    - Inputs: All three Format Alert nodes (Info, Warning, Critical)  
    - Outputs: Store Alert History

- **Potential Failures:**  
  - Credential expiry or misconfiguration for Gmail or Slack  
  - Network or API errors posting to Slack or dashboard  
  - Incorrect email addresses or Slack channel IDs  
  - Message formatting issues causing delivery failures

---

#### 1.8 Alert Storage

- **Overview:**  
Stores alert history into a PostgreSQL database table for auditing and historical analysis.

- **Nodes Involved:**  
  - Store Alert History

- **Node Details:**

  - **Store Alert History**  
    - Type: PostgreSQL  
    - Role: Inserts alert records into "alert_history" table in "public" schema.  
    - Config: Auto-maps input JSON fields to table columns.  
    - Inputs: Send to Dashboard API node  
    - Outputs: None

- **Potential Failures:**  
  - Database connection or credential errors  
  - Schema or table missing or mismatched columns  
  - Input data fields not matching table columns (autoMapInputData mitigates this)

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                            | Input Node(s)                          | Output Node(s)                       | Sticky Note                                                                                                   |
|----------------------------|---------------------------------|--------------------------------------------|--------------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Every 5 Minutes    | Schedule Trigger                | Periodic trigger every 5 minutes            | None                                 | Workflow Configuration             | ## Schedule Data Collection<br>Triggers workflow every 5 minutes to fetch oil markets, OPEC reports, shipping data, news feeds. |
| Workflow Configuration      | Set                            | Sets API URLs, alert recipients, thresholds | Schedule Every 5 Minutes             | Fetch Oil Price Data (API), Fetch OPEC Reports, Fetch Shipping Data, Fetch News Feed | ## Setup Steps<br>1. Connect data sources<br>2. Configure AI<br>3. Define alerts<br>4. Setup storage             |
| Fetch Oil Price Data (API)  | HTTP Request                   | Retrieves oil price data                     | Workflow Configuration               | Merge All Data Sources             | ## Fetch Oil Price Data<br>Retrieves spot prices, futures, contract rates from multiple exchanges              |
| Fetch OPEC Reports          | HTTP Request                   | Retrieves OPEC production reports            | Workflow Configuration               | Merge All Data Sources             |                                                                                                               |
| Fetch Shipping Data         | HTTP Request                   | Retrieves shipping and supply chain data     | Workflow Configuration               | Merge All Data Sources             | ## Aggregate Shipping Data<br>Collects vessel tracking, port delays, freight costs                              |
| Fetch News Feed             | HTTP Request                   | Retrieves news articles                       | Workflow Configuration               | Merge All Data Sources             |                                                                                                               |
| Merge All Data Sources      | Merge                          | Combines all fetched data streams            | Fetch Oil Price Data (API), Fetch OPEC Reports, Fetch Shipping Data, Fetch News Feed | Data Cleaning & Preprocessing      | ## Clean Multi-Source Data<br>Standardizes prices, shipping metrics, dates                                      |
| Data Cleaning & Preprocessing | Code (JavaScript)              | Standardizes and enriches data                | Merge All Data Sources               | Statistical Anomaly Detection, AI Trend & Geopolitical Analyzer |                                                                                                               |
| Statistical Anomaly Detection | Code (JavaScript)              | Detects quantitative anomalies in data       | Data Cleaning & Preprocessing        | Calculate Crisis Risk Score        | ## Statistical Anomaly Detection<br>Identifies unusual shifts and volatility spikes                             |
| OpenRouter Chat Model       | LangChain LM Chat OpenRouter   | Calls LLM for geopolitical risk analysis     | - (invoked by AI Trend & Geopolitical Analyzer) | AI Trend & Geopolitical Analyzer  |                                                                                                               |
| AI Trend & Geopolitical Analyzer | LangChain Agent              | Analyzes geopolitical risks and crises       | Data Cleaning & Preprocessing, OpenRouter Chat Model | Calculate Crisis Risk Score        | ## AI Crisis Risk Model<br>Combines signals into coherent risk narrative                                       |
| Calculate Crisis Risk Score | Code (JavaScript)              | Composite scoring and alert level assignment | Statistical Anomaly Detection, AI Trend & Geopolitical Analyzer | Route by Alert Level               | ## Score Emerging Threats<br>Prioritizes highest-threat scenarios                                              |
| Route by Alert Level        | Switch                        | Routes alerts by severity thresholds          | Calculate Crisis Risk Score          | Format Info Alert, Format Warning Alert, Format Critical Alert | ## Route Critical Alerts to Traders<br>Sends high-risk signals immediately via Slack and dashboard              |
| Format Info Alert           | Set                           | Formats informational alert message           | Route by Alert Level (Info branch)  | Send to Dashboard API              |                                                                                                               |
| Format Warning Alert        | Set                           | Formats warning alert message                  | Route by Alert Level (Warning branch) | Send Email Alert, Send Slack Alert, Send to Dashboard API |                                                                                                               |
| Format Critical Alert       | Set                           | Formats critical alert message                 | Route by Alert Level (Critical branch) | Send Email Alert, Send Slack Alert, Send to Dashboard API |                                                                                                               |
| Send Email Alert            | Gmail                         | Sends alert emails                             | Format Warning Alert, Format Critical Alert | None                             | ## Alert to Email & Slack<br>Logs alerts with timestamps, reasoning, sources                                  |
| Send Slack Alert            | Slack                         | Sends alert messages to Slack channel          | Format Warning Alert, Format Critical Alert | None                             |                                                                                                               |
| Send to Dashboard API       | HTTP Request                  | Updates external dashboard with alert data    | Format Info Alert, Format Warning Alert, Format Critical Alert | Store Alert History               | ## Alerts to Dashboard & Storage<br>Posts lower-severity signals to dashboard                                  |
| Store Alert History         | PostgreSQL                    | Saves alert history for auditing               | Send to Dashboard API               | None                             |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set interval to every 5 minutes.

2. **Create Set Node for Workflow Configuration:**  
   - Type: Set  
   - Define string parameters for: oilPriceApiUrl, opecReportsUrl, shippingDataUrl, newsApiUrl, dashboardApiUrl, alertEmailRecipients, slackChannel.  
   - Define number parameters for infoThreshold (30), warningThreshold (60), criticalThreshold (85).

3. **Create HTTP Request Nodes for Data Fetching:**  
   - Four nodes: "Fetch Oil Price Data (API)", "Fetch OPEC Reports", "Fetch Shipping Data", "Fetch News Feed".  
   - Configure each as GET requests with Accept: application/json header.  
   - URLs set dynamically from Workflow Configuration node variables.

4. **Create Merge Node:**  
   - Type: Merge  
   - Mode: combineAll  
   - Connect all four fetch nodes as inputs.

5. **Create Code Node for Data Cleaning & Preprocessing:**  
   - Paste provided JavaScript code that standardizes and enriches incoming data.  
   - Connect Merge node output to this node.

6. **Create Code Node for Statistical Anomaly Detection:**  
   - Paste JavaScript code for computing statistical metrics and anomaly scores.  
   - Connect Data Cleaning & Preprocessing node output.

7. **Create LangChain LM Chat OpenRouter Node:**  
   - Select model "qwen/qwq-32b"  
   - Authenticate with valid OpenRouter API credentials.

8. **Create LangChain Agent Node "AI Trend & Geopolitical Analyzer":**  
   - Use prompt that analyzes combined oil market data for geopolitical risks.  
   - Connect output of Data Cleaning & Preprocessing as input data for prompt.  
   - Connect LM Chat node as AI language model input.

9. **Connect AI Trend & Geopolitical Analyzer and Statistical Anomaly Detection to Code Node "Calculate Crisis Risk Score":**  
   - Paste JavaScript code that calculates weighted risk score and assigns alert levels based on thresholds.

10. **Create Switch Node "Route by Alert Level":**  
    - Configure three outputs: Info, Warning, Critical.  
    - Set conditions based on riskScore compared to thresholds from Workflow Configuration node.

11. **Create three Set Nodes for formatting alerts:**  
    - Format Info Alert: set alertLevel "INFO", alertTitle "Oil Market Update", alertMessage with riskScore and summary, color #17a2b8.  
    - Format Warning Alert: alertLevel "WARNING", alertTitle with warning emoji, more detailed alertMessage, color #ffc107.  
    - Format Critical Alert: alertLevel "CRITICAL", alertTitle with siren emoji, urgent alertMessage, color #dc3545.

12. **Create Gmail Node "Send Email Alert":**  
    - Configure recipients from Workflow Configuration alertEmailRecipients.  
    - Use OAuth2 Gmail credentials.  
    - Use HTML message formatting including alert details.

13. **Create Slack Node "Send Slack Alert":**  
    - Configure channel from Workflow Configuration slackChannel.  
    - Authenticate with Slack OAuth2 credentials.  
    - Send message as attachment with alert title, message, timestamp, and color coding.

14. **Create HTTP Request Node "Send to Dashboard API":**  
    - POST request to dashboardApiUrl from config.  
    - Send JSON body with alert data.

15. **Create PostgreSQL Node "Store Alert History":**  
    - Connect to your PostgreSQL database.  
    - Configure to insert into "alert_history" table in "public" schema with auto-mapping of input fields.

16. **Connect Nodes:**  
    - Schedule Trigger → Workflow Configuration  
    - Workflow Configuration → All Fetch nodes  
    - All Fetch nodes → Merge node  
    - Merge → Data Cleaning & Preprocessing  
    - Data Cleaning & Preprocessing → Statistical Anomaly Detection and AI Trend & Geopolitical Analyzer  
    - AI Trend & Geopolitical Analyzer → Calculate Crisis Risk Score  
    - Statistical Anomaly Detection → Calculate Crisis Risk Score  
    - Calculate Crisis Risk Score → Route by Alert Level  
    - Route by Alert Level → Format Info/Warning/Critical Alert nodes  
    - Format Info Alert → Send to Dashboard API  
    - Format Warning Alert → Send Email Alert, Send Slack Alert, Send to Dashboard API  
    - Format Critical Alert → Send Email Alert, Send Slack Alert, Send to Dashboard API  
    - Send to Dashboard API → Store Alert History

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Scheduled runs collect data from oil markets, shipping, news, and official reports every 5 minutes for rapid early warning.                                                   | Sticky Note near Schedule Every 5 Minutes node                                                        |
| Data sources include oil price APIs, OPEC reports, shipping databases, and news feeds.                                                                                          | Setup Steps sticky note                                                                                |
| AI model used is Qwen/Qwq-32b via OpenRouter for geopolitical risk and crisis analysis.                                                                                        | AI Crisis Risk Model sticky note                                                                       |
| Alerts are routed by severity thresholds into Email, Slack, and Dashboard API channels for comprehensive coverage.                                                             | Alerts to Dashboard & Storage, Alert to Email & Slack sticky notes                                     |
| Statistical anomaly detection uses Z-scores, moving averages, and volatility measures to flag unusual market activity objectively.                                            | Statistical Anomaly Detection sticky note                                                             |
| Simple keyword-based sentiment analysis is applied on news text for preliminary sentiment scoring.                                                                             | Data Cleaning & Preprocessing sticky note                                                             |
| The workflow supports customization of risk thresholds and addition of new data sources or alert routing rules to fit different organizational needs.                         | Customization and Benefits sticky note                                                                |
| Alert history is stored in PostgreSQL for audit trails and historical trend analysis.                                                                                          | Alerts to Dashboard & Storage sticky note                                                             |
| Project use cases include supply chain risk monitoring, energy market crisis detection, geopolitical threat assessment, trader decision support, and operational risk management. | Prerequisites and Use Cases sticky note                                                               |

---

This document provides a detailed, stepwise understanding of the "Real-Time Oil Price Crisis Detection and Smart Alert System via Qwq-32b" workflow for both human analysts and automation agents to maintain, reproduce, or extend the system reliably.