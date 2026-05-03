Monitor Supplier Financial Health with ScrapeGraphAI & Multi-Channel Risk Alerts

https://n8nworkflows.xyz/workflows/monitor-supplier-financial-health-with-scrapegraphai---multi-channel-risk-alerts-7129


# Monitor Supplier Financial Health with ScrapeGraphAI & Multi-Channel Risk Alerts

---

### 1. Workflow Overview

This workflow, titled **"Monitor Supplier Financial Health with ScrapeGraphAI & Multi-Channel Risk Alerts,"** automates the process of assessing the financial health and associated risks of suppliers within a company’s procurement ecosystem. It targets procurement managers, risk analysts, and supply chain professionals who need to monitor supplier stability, detect early warning signs, and react proactively to potential supplier risks.

The workflow is logically structured into the following functional blocks:

- **1.1 Scheduled Trigger & Supplier Data Loading:** Automatically initiates weekly supplier risk assessments and loads supplier profiles filtered by criticality and last assessment date.
- **1.2 Data Scraping:** Gathers financial health indicators and news sentiment by scraping company websites and financial news sources via ScrapeGraphAI.
- **1.3 Financial Health Analysis:** Aggregates and interprets scraped data to generate a comprehensive risk assessment, including scoring and recommended actions.
- **1.4 Advanced Risk Scoring:** Applies industry-specific and economic multipliers to refine risk scores and predict failure probabilities.
- **1.5 Risk Threshold Evaluation & Supplier Alternatives:** Determines if risk exceeds a threshold and, if so, finds ranked alternative suppliers with transition complexity and mitigation strategies.
- **1.6 Multi-Channel Alerting & Procurement System Integration:** Sends formatted alerts through email and Slack, updates internal procurement systems with assessment data.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Trigger & Supplier Data Loading

- **Overview:**  
This block triggers the workflow weekly and loads a list of suppliers from an internal database, filtering them based on criticality and last assessment date to determine which suppliers require monitoring.

- **Nodes Involved:**  
  - 📅 Weekly Health Check (Schedule Trigger)  
  - 🏪 Supplier Database Loader (Code)  
  - 🔍 Has Suppliers to Monitor? (If)

- **Node Details:**

  - **📅 Weekly Health Check**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates workflow execution every week.  
    - *Configuration:* Interval set to 1 week, no specific time.  
    - *Edges:* Output to Supplier Database Loader.  
    - *Potential Failures:* Scheduling misconfigurations, workflow not enabled.

  - **🏪 Supplier Database Loader**  
    - *Type:* Code Node  
    - *Role:* Loads a hard-coded supplier list, filters suppliers for monitoring based on their criticality and last assessment date.  
    - *Key Logic:*  
      - Defines suppliers with metadata: ID, name, website, category, contract value, criticality, last assessment, rating.  
      - Filters suppliers: criticality-based check intervals (7, 14, 30 days).  
      - Produces an array of suppliers to monitor with added session metadata.  
    - *Output:* Multiple items, each corresponding to a supplier to monitor or a single item indicating no suppliers to monitor.  
    - *Edges:* Output to Has Suppliers to Monitor?  
    - *Potential Failures:* Logic errors in filtering, date parsing issues.

  - **🔍 Has Suppliers to Monitor?**  
    - *Type:* If Node  
    - *Role:* Checks if there are suppliers flagged for monitoring.  
    - *Condition:* Passes if `no_suppliers_to_monitor` is not true.  
    - *Edges:*  
      - True branch leads to Company Website Scraper and Financial News Scraper.  
      - False branch terminates or no further action.  
    - *Potential Failures:* Expression evaluation errors, empty input handling.

---

#### 1.2 Data Scraping

- **Overview:**  
For each supplier to monitor, this block scrapes financial data and recent news about the supplier using ScrapeGraphAI’s smart scraper API.

- **Nodes Involved:**  
  - 🕷️ Company Website Scraper (HTTP Request)  
  - 📰 Financial News Scraper (HTTP Request)

- **Node Details:**

  - **🕷️ Company Website Scraper**  
    - *Type:* HTTP Request  
    - *Role:* Scrapes supplier website for financial health indicators, executive contacts, and risk signals.  
    - *Configuration:*  
      - POST to ScrapeGraphAI API endpoint.  
      - Authenticated via header with API key.  
      - Payload includes website URL, user prompt specifying financial indicators to extract, and output schema (JSON).  
      - Timeout set to 60 seconds.  
    - *Edges:* Output to Financial Health Analyzer.  
    - *Potential Failures:* API key missing/invalid, timeout, malformed response, network issues.

  - **📰 Financial News Scraper**  
    - *Type:* HTTP Request  
    - *Role:* Scrapes financial news related to the supplier from Google Search results filtered by keywords indicating financial troubles.  
    - *Configuration:*  
      - POST request to ScrapeGraphAI with prompt for recent financial news and risk flags, output schema for news articles and sentiment.  
      - Uses same authentication as above.  
      - Timeout 60 seconds.  
    - *Edges:* Output to Financial Health Analyzer.  
    - *Potential Failures:* Same as above, plus risk of incomplete or misleading news data.

---

#### 1.3 Financial Health Analysis

- **Overview:**  
This block merges and analyzes scraped data to evaluate financial health and risk factors, calculates a risk score, and generates a health assessment with recommendations.

- **Nodes Involved:**  
  - 🔬 Financial Health Analyzer (Code)

- **Node Details:**

  - **🔬 Financial Health Analyzer**  
    - *Type:* Code Node  
    - *Role:* Parses website and news data, identifies risk factors across financial, operational, market, and reputational categories.  
    - *Key Logic:*  
      - Parses JSON responses safely with error handling.  
      - Extracts financial highlights, risk indicators, negative news headlines.  
      - Calculates base risk score weighted by number of risk factors, supplier criticality, contract value, and data confidence levels.  
      - Assigns risk level thresholds (low, medium, high, critical).  
      - Creates comprehensive health assessment object with risk details, key findings, recommended actions, next review date.  
    - *Edges:* Output to Advanced Risk Scorer.  
    - *Potential Failures:* JSON parse errors, missing data, division by zero, logic mistakes in scoring.

---

#### 1.4 Advanced Risk Scoring

- **Overview:**  
Applies advanced scoring algorithms incorporating industry-specific risk multipliers and macroeconomic factors to refine the supplier risk score and estimate failure probabilities.

- **Nodes Involved:**  
  - 📊 Advanced Risk Scorer (Code)

- **Node Details:**

  - **📊 Advanced Risk Scorer**  
    - *Type:* Code Node  
    - *Role:* Enhances risk score by applying multipliers for supplier industry category and simulated economic risk factors (inflation, interest rates, etc.).  
    - *Key Logic:*  
      - Industry multipliers defined per category (e.g., Technology 1.1x, Energy 1.2x).  
      - Multiplies base risk by economic factors cumulatively.  
      - Caps score at 100.  
      - Recalculates adjusted risk level with higher thresholds.  
      - Calculates failure probability as percentage.  
      - Breaks down risk into financial, operational, market, reputational components.  
      - Adds trend analysis (direction, magnitude, confidence).  
      - Generates enhanced recommendations based on adjusted risk score.  
    - *Edges:* Output to Risk Above Threshold?  
    - *Potential Failures:* Missing or invalid data, math errors, category not found mapping defaults.

---

#### 1.5 Risk Threshold Evaluation & Supplier Alternatives

- **Overview:**  
Evaluates if the supplier’s adjusted risk exceeds a threshold; if yes, identifies and ranks alternative suppliers with transition planning and risk mitigation strategies.

- **Nodes Involved:**  
  - ⚠️ Risk Above Threshold? (If)  
  - 🔄 Alternative Supplier Finder (Code)

- **Node Details:**

  - **⚠️ Risk Above Threshold?**  
    - *Type:* If Node  
    - *Role:* Checks whether adjusted risk score is greater than or equal to 40.  
    - *Edges:*  
      - True branch proceeds to Alternative Supplier Finder.  
      - False branch terminates or no further action.  
    - *Potential Failures:* Expression evaluation issues.

  - **🔄 Alternative Supplier Finder**  
    - *Type:* Code Node  
    - *Role:* Matches the supplier’s category against a pre-defined alternative supplier database, scoring and ranking alternatives by rating, capacity, risk, and lead time.  
    - *Key Logic:*  
      - Extracts supplier category heuristically from name.  
      - Filters alternatives with capacity >= current contract value.  
      - Scores alternatives on rating (40 pts), capacity (25 pts), risk (25 pts), and lead time (10 pts).  
      - Assesses transition complexity based on lead time, contract size, criticality, and geographic coverage.  
      - Generates transition strategy recommendations and timeline based on risk level.  
      - Calculates budget impact estimate and risk mitigation metrics.  
      - Outputs top 3 suitable alternatives with detailed analysis.  
    - *Edges:* Output to Alert Formatter.  
    - *Potential Failures:* Logic errors in scoring, missing data, fallback defaults, category extraction inaccuracies.

---

#### 1.6 Multi-Channel Alerting & Procurement System Integration

- **Overview:**  
Formats the risk assessment into alerts for email and Slack, sends notifications to appropriate recipients based on risk priority, and updates the internal procurement system via API.

- **Nodes Involved:**  
  - 📧 Alert Formatter (Set)  
  - 📨 Email Alert Sender (Gmail)  
  - 💬 Slack Alert (HTTP Request)  
  - 🏢 Procurement System Update (HTTP Request)

- **Node Details:**

  - **📧 Alert Formatter**  
    - *Type:* Set Node  
    - *Role:* Constructs alert content with markdown formatting including risk summary, key factors, recommendations, alternative suppliers, and next steps.  
    - *Parameters:*  
      - Dynamic expressions for email content using liquid templating syntax.  
      - Sets alert priority and recipients based on risk score thresholds.  
    - *Edges:* Output to Email Alert Sender, Slack Alert, Procurement System Update.  
    - *Potential Failures:* Expression syntax errors, missing data.

  - **📨 Email Alert Sender**  
    - *Type:* Gmail Node  
    - *Role:* Sends risk alert emails to recipients determined in the formatter node.  
    - *Configuration:*  
      - Uses Gmail OAuth2 credentials (must be configured in n8n).  
      - Subject includes supplier name and alert priority.  
      - Email body is plain text from formatted content.  
    - *Potential Failures:* Credential issues, SMTP errors, quota limits.

  - **💬 Slack Alert**  
    - *Type:* HTTP Request  
    - *Role:* Posts a formatted Slack message to a configured Slack incoming webhook URL.  
    - *Payload:* JSON Slack Block Kit format including header, sections, buttons linking to an external dashboard.  
    - *Potential Failures:* Invalid URL, network failure, Slack API rate limits.

  - **🏢 Procurement System Update**  
    - *Type:* HTTP Request  
    - *Role:* Sends a webhook POST to the company’s internal procurement system API with risk assessment and metadata.  
    - *Configuration:*  
      - Requires API token credential.  
      - Sends JSON including event type, supplier ID, risk scores, action flags, full assessment data, and timestamp.  
    - *Potential Failures:* API authentication, network errors, payload size limits.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                               | Input Node(s)                         | Output Node(s)                              | Sticky Note                                                                                  |
|----------------------------|------------------------|----------------------------------------------|-------------------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------|
| 📅 Weekly Health Check      | Schedule Trigger       | Initiates weekly workflow run                 | —                                   | 🏪 Supplier Database Loader                  |                                                                                              |
| 🏪 Supplier Database Loader | Code                   | Loads and filters suppliers to monitor        | 📅 Weekly Health Check               | 🔍 Has Suppliers to Monitor?                 |                                                                                              |
| 🔍 Has Suppliers to Monitor?| If                     | Checks if any suppliers need monitoring       | 🏪 Supplier Database Loader          | 🕷️ Company Website Scraper, 📰 Financial News Scraper |                                                                                              |
| 🕷️ Company Website Scraper   | HTTP Request           | Scrapes supplier website financial data       | 🔍 Has Suppliers to Monitor?         | 🔬 Financial Health Analyzer                  |                                                                                              |
| 📰 Financial News Scraper    | HTTP Request           | Scrapes recent financial news about supplier  | 🔍 Has Suppliers to Monitor?         | 🔬 Financial Health Analyzer                  |                                                                                              |
| 🔬 Financial Health Analyzer | Code                   | Aggregates and analyzes data to assess risk   | 🕷️ Company Website Scraper, 📰 Financial News Scraper, 🏪 Supplier Database Loader | 📊 Advanced Risk Scorer                      | # 🔬 Financial Health Analyzer: Purpose, components, risk categories, output description      |
| 📊 Advanced Risk Scorer      | Code                   | Applies advanced scoring with industry context| 🔬 Financial Health Analyzer         | ⚠️ Risk Above Threshold?                     | # 📊 Advanced Risk Scorer: Purpose, scoring factors, industry multipliers, output            |
| ⚠️ Risk Above Threshold?     | If                     | Checks if risk score exceeds threshold        | 📊 Advanced Risk Scorer              | 🔄 Alternative Supplier Finder                |                                                                                              |
| 🔄 Alternative Supplier Finder| Code                   | Finds and ranks alternative suppliers          | ⚠️ Risk Above Threshold?              | 📧 Alert Formatter                           | # 🔄 Alternative Supplier Finder: Purpose, scoring algorithm, transition complexity, output  |
| 📧 Alert Formatter           | Set                    | Formats alert content and assigns recipients  | 🔄 Alternative Supplier Finder       | 📨 Email Alert Sender, 💬 Slack Alert, 🏢 Procurement System Update | # 📧 Multi-Channel Alert System: alert channels, prioritization, and content overview        |
| 📨 Email Alert Sender        | Gmail                  | Sends email alerts                            | 📧 Alert Formatter                  | —                                           |                                                                                              |
| 💬 Slack Alert               | HTTP Request           | Sends Slack notifications                      | 📧 Alert Formatter                  | —                                           |                                                                                              |
| 🏢 Procurement System Update | HTTP Request           | Updates internal procurement system            | 📧 Alert Formatter                  | —                                           |                                                                                              |
| 📋 Workflow Overview        | Sticky Note            | High-level workflow summary                    | —                                   | —                                           | Contains overall workflow features, monitoring sources, and risk indicators                  |
| Health Analyzer Info        | Sticky Note            | Explains Financial Health Analyzer node       | —                                   | —                                           | # 🔬 Financial Health Analyzer description                                                  |
| Risk Scorer Info            | Sticky Note            | Explains Advanced Risk Scorer node             | —                                   | —                                           | # 📊 Advanced Risk Scorer description                                                       |
| Alternative Finder Info     | Sticky Note            | Explains Alternative Supplier Finder node     | —                                   | —                                           | # 🔄 Alternative Supplier Finder description                                                |
| Alert System Info           | Sticky Note            | Explains alerting system nodes                 | —                                   | —                                           | # 📧 Multi-Channel Alert System description                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node: "📅 Weekly Health Check"**  
   - Type: Schedule Trigger  
   - Configure interval to trigger every 1 week.

2. **Create Code Node: "🏪 Supplier Database Loader"**  
   - Type: Code  
   - Paste the JavaScript code that defines supplier array, filtering logic by criticality and last assessment date, and outputs suppliers to monitor as individual items.  
   - Connect output of "📅 Weekly Health Check" to this node.

3. **Create If Node: "🔍 Has Suppliers to Monitor?"**  
   - Type: If  
   - Set condition to check that `$json.no_suppliers_to_monitor !== true`.  
   - Connect "🏪 Supplier Database Loader" output to this node.

4. **Create HTTP Request Node: "🕷️ Company Website Scraper"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.scrapegraph.ai/v1/smartscraper`  
   - Authentication: Header Auth with key `Authorization: Bearer YOUR_SCRAPEGRAPH_API_KEY`  
   - Body: JSON with parameters:  
     - `website_url`: `{{$json.website}}`  
     - `user_prompt`: Custom prompt to extract financial indicators  
     - `output_schema`: JSON schema for expected data  
   - Timeout: 60000 ms  
   - Connect "🔍 Has Suppliers to Monitor?" true branch to this node.

5. **Create HTTP Request Node: "📰 Financial News Scraper"**  
   - Type: HTTP Request  
   - Same API endpoint and authentication as above.  
   - Body parameters:  
     - `website_url`: Google search URL with query including supplier name and financial risk keywords.  
     - `user_prompt`: Prompt to extract recent financial news and risk flags.  
     - `output_schema`: JSON schema for news data.  
   - Timeout: 60000 ms  
   - Connect "🔍 Has Suppliers to Monitor?" true branch to this node in parallel with the website scraper.

6. **Create Code Node: "🔬 Financial Health Analyzer"**  
   - Type: Code  
   - Paste JavaScript code to parse website and news data, calculate base risk score, determine risk level, generate health assessment object with findings and recommended actions.  
   - Connect outputs of both scrapers to this node.

7. **Create Code Node: "📊 Advanced Risk Scorer"**  
   - Type: Code  
   - Paste JavaScript to apply industry multipliers, economic factors, calculate adjusted risk score, failure probability, risk breakdown, trend analysis, and enhanced recommendations.  
   - Connect "🔬 Financial Health Analyzer" output to this node.

8. **Create If Node: "⚠️ Risk Above Threshold?"**  
   - Type: If  
   - Condition: Check if `{{$json.risk_scoring.adjusted_score}} >= 40`  
   - Connect "📊 Advanced Risk Scorer" output to this node.

9. **Create Code Node: "🔄 Alternative Supplier Finder"**  
   - Type: Code  
   - Paste detailed JavaScript code that:  
     - Extracts supplier category heuristically  
     - Filters, scores, ranks alternative suppliers by rating, capacity, risk, lead time  
     - Assesses transition complexity and budget impact  
     - Generates transition recommendations and risk mitigation metrics  
   - Connect "⚠️ Risk Above Threshold?" true branch to this node.

10. **Create Set Node: "📧 Alert Formatter"**  
    - Type: Set  
    - Define fields for email content using liquid templating with risk assessment details, key factors, alternative suppliers, and next steps.  
    - Define alert priority and recipients dynamically based on risk score thresholds.  
    - Connect output of "🔄 Alternative Supplier Finder" to this node.

11. **Create Gmail Node: "📨 Email Alert Sender"**  
    - Type: Gmail  
    - Configure with OAuth2 Gmail credentials.  
    - Subject: Include supplier name and alert priority.  
    - Message: Use `email_content` from previous node.  
    - Connect "📧 Alert Formatter" to this node.

12. **Create HTTP Request Node: "💬 Slack Alert"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Slack Incoming Webhook URL (replace with your actual webhook)  
    - Body: JSON Slack Block Kit formatted message referencing supplier and risk details.  
    - Connect "📧 Alert Formatter" to this node.

13. **Create HTTP Request Node: "🏢 Procurement System Update"**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Your internal procurement system webhook URL  
    - Headers: Authorization Bearer token and Content-Type application/json  
    - Body: Include event type, supplier ID, risk scores, action required flag, full assessment JSON, timestamp.  
    - Connect "📧 Alert Formatter" to this node.

14. **Create Sticky Notes** at appropriate workflow positions to document:  
    - Workflow Overview (features, sources, risk indicators)  
    - Financial Health Analyzer explanation  
    - Advanced Risk Scorer explanation  
    - Alternative Supplier Finder explanation  
    - Multi-Channel Alert System explanation

15. **Verify connections:**  
    - Schedule Trigger → Supplier Database Loader → Has Suppliers to Monitor?  
    - Has Suppliers to Monitor? → (true) → Company Website Scraper & Financial News Scraper (parallel) → Financial Health Analyzer → Advanced Risk Scorer → Risk Above Threshold? → Alternative Supplier Finder → Alert Formatter → Email, Slack, Procurement update  
    - Has Suppliers to Monitor? → (false) → end workflow.

16. **Configure all API keys, OAuth2 credentials, and URLs as environment variables or credentials in n8n.**

17. **Test workflow with sample suppliers and validate outputs and alerts.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The workflow leverages ScrapeGraphAI for intelligent web scraping of financial data and news articles.                                             | ScrapeGraphAI API: https://scrapegraph.ai                                                                        |
| Risk scoring incorporates domain-specific multipliers and simulated macroeconomic factors to enhance predictive accuracy.                         | Industry risk multipliers and economic factors are hardcoded but can be extended with live data feeds.             |
| Alternative supplier recommendations use a structured scoring system considering rating, capacity, risk, and lead time for best-fit selection.    | Supplier transition complexity and budget impact are estimated within code for realistic procurement planning.     |
| Alerting includes multi-channel outputs: email with Gmail OAuth2, Slack notifications with webhook, and integration with procurement system APIs. | Ensure proper credential and webhook security and rate limits are respected in production environments.           |
| Sticky notes provide useful documentation and summary for each major functional block to improve maintainability and onboarding.                   | Use the sticky notes as in-workflow documentation for clarity.                                                    |
| This workflow is not active by default; enable and schedule it in n8n for regular operation.                                                       |                                                                                                                   |

---

**Disclaimer:** The text above is generated from an automated n8n workflow designed for legal and public supplier financial monitoring. All data references are simulated or sourced from public domains. No illegal or offensive content is included.