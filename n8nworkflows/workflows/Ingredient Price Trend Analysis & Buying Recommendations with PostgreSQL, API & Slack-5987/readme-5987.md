Ingredient Price Trend Analysis & Buying Recommendations with PostgreSQL, API & Slack

https://n8nworkflows.xyz/workflows/ingredient-price-trend-analysis---buying-recommendations-with-postgresql--api---slack-5987


# Ingredient Price Trend Analysis & Buying Recommendations with PostgreSQL, API & Slack

### 1. Workflow Overview

This workflow automates the monitoring of ingredient price fluctuations, performs trend analysis, generates smart buying recommendations, and disseminates insights via email and Slack. It suits food industry professionals or supply chain managers who want to optimize purchasing decisions based on price trends.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Data Ingestion**  
  Scheduled daily trigger initiates API calls to fetch current ingredient prices.

- **1.2 Database Setup & Data Storage**  
  Ensures database tables exist and stores fetched price data into PostgreSQL.

- **1.3 Trend Calculation & Analysis**  
  Runs SQL queries to calculate price trends over the last 30 days.

- **1.4 Recommendation Generation & Storage**  
  Applies custom JavaScript logic to generate buying recommendations, then stores these in the database.

- **1.5 Dashboard Preparation & Delivery**  
  Queries the stored recommendations, generates an HTML dashboard, and sends it via email and Slack alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Ingestion

- **Overview:**  
  Initiates the workflow daily and fetches the latest ingredient prices from an external API.

- **Nodes Involved:**  
  - Daily Price Check  
  - Fetch API Prices

- **Node Details:**

  - **Daily Price Check**  
    - Type: Cron Trigger  
    - Configured to trigger the workflow once daily (default cron settings implied).  
    - Outputs trigger signal to "Fetch API Prices" and "Setup Database".  
    - Potential Failures: Cron misconfiguration (e.g., timezone issues).  
    - No credentials required.

  - **Fetch API Prices**  
    - Type: HTTP Request  
    - Configured to GET data from `https://api.example-food-prices.com/ingredients`.  
    - No additional query or body parameters; expects JSON response with ingredient pricing.  
    - Outputs data to "Store Price Data".  
    - Potential Failures: Network errors, API downtime, unexpected response format, rate limiting.  
    - No authentication indicated but may require in real scenarios.

#### 2.2 Database Setup & Data Storage

- **Overview:**  
  Ensures necessary database tables exist, then inserts the fetched prices into the PostgreSQL database.

- **Nodes Involved:**  
  - Setup Database  
  - Store Price Data

- **Node Details:**

  - **Setup Database**  
    - Type: PostgreSQL (Execute Query)  
    - Runs SQL commands to create two tables if they do not exist:  
      - `price_history` (stores ingredient prices with timestamp)  
      - `buying_recommendations` (stores generated recommendations with metadata)  
    - Also creates indexes for query optimization on ingredient & timestamp and generated_at columns.  
    - Outputs trigger to "Store Price Data".  
    - Requires PostgreSQL credentials (named "Postgres-test").  
    - Potential Failures: DB connection errors, permission issues.

  - **Store Price Data**  
    - Type: PostgreSQL (Insert)  
    - Inserts API response data into `price_history` table using automatic mapping of fields.  
    - Columns expected: ingredient, price, unit, supplier, timestamp, created_at.  
    - Receives input from both "Fetch API Prices" and "Setup Database" (parallel triggers).  
    - Outputs to "Calculate Trends".  
    - Potential Failures: Data type mismatches, missing fields, DB write errors.

#### 2.3 Trend Calculation & Analysis

- **Overview:**  
  Calculates price changes and trends over the previous 30 days for each ingredient using a SQL query.

- **Nodes Involved:**  
  - Calculate Trends

- **Node Details:**

  - **Calculate Trends**  
    - Type: PostgreSQL (Execute Query)  
    - Runs a window function query:  
      - Compares current and previous prices per ingredient ordered by timestamp.  
      - Calculates percentage price change and assigns trend labels: INCREASING, DECREASING, STABLE.  
    - Filters only entries with valid previous price data.  
    - Outputs results to "Generate Recommendations".  
    - Requires PostgreSQL credentials.  
    - Potential Failures: SQL syntax errors, empty data, DB timeouts.

#### 2.4 Recommendation Generation & Storage

- **Overview:**  
  Processes price trends to generate actionable buying recommendations and stores them in the database.

- **Nodes Involved:**  
  - Generate Recommendations  
  - Store Recommendations

- **Node Details:**

  - **Generate Recommendations**  
    - Type: Code (JavaScript)  
    - Input: Rows from "Calculate Trends".  
    - Logic:  
      - For each ingredient, assesses price_change_percent and assigns recommendation, urgency, and reason.  
      - Rules include:  
        - Price drop >10% → "BUY NOW" (HIGH urgency)  
        - Price drop >5% → "CONSIDER BUYING" (MEDIUM urgency)  
        - Price rise >15% → "AVOID BUYING" (HIGH urgency)  
        - Price rise >5% → "WAIT" (MEDIUM urgency)  
        - Otherwise "MONITOR" (LOW urgency)  
    - Returns an array of recommendation objects.  
    - Outputs to "Store Recommendations".  
    - Potential Failures: JavaScript runtime errors, empty input.

  - **Store Recommendations**  
    - Type: PostgreSQL (Insert)  
    - Inserts generated recommendations into `buying_recommendations` table.  
    - Auto-maps fields including ingredient, current_price, price_change_percent, trend, recommendation, urgency, reason, generated_at.  
    - Outputs to "Get Dashboard Data".  
    - Requires PostgreSQL credentials.  
    - Potential Failures: DB write errors, data type mismatch.

#### 2.5 Dashboard Preparation & Delivery

- **Overview:**  
  Retrieves the latest recommendations, generates an HTML dashboard, and distributes reports via email and Slack.

- **Nodes Involved:**  
  - Get Dashboard Data  
  - Generate Dashboard HTML  
  - Send Email Report  
  - Send Slack Alert

- **Node Details:**

  - **Get Dashboard Data**  
    - Type: PostgreSQL (Execute Query)  
    - Query fetches recommendations generated today, ordered by urgency and price change.  
    - Fields retrieved: ingredient, current_price, price_change_percent, trend, recommendation, urgency, reason, generated_at.  
    - Outputs to "Generate Dashboard HTML".  
    - Requires PostgreSQL credentials.  
    - Potential Failures: Empty data, query errors.

  - **Generate Dashboard HTML**  
    - Type: Code (JavaScript)  
    - Input: Current day's recommendations.  
    - Generates a styled HTML page visualizing:  
      - Summary stats (counts by urgency, buy now opportunities, price trends)  
      - Cards for each ingredient with color-coded urgency and recommendation info.  
      - Timestamp of report generation.  
    - Outputs HTML string in JSON to next nodes.  
    - Potential Failures: Rendering errors, empty data.

  - **Send Email Report**  
    - Type: Email Send  
    - Sends an email with the generated HTML dashboard as an attachment.  
    - Subject includes current date.  
    - Sender and recipient emails configured ("xyz@gmail.com" and "abc@gmail.com").  
    - Uses SMTP credentials.  
    - Potential Failures: SMTP authentication, email sending failures.

  - **Send Slack Alert**  
    - Type: HTTP Request (Webhook)  
    - Posts a Slack message summarizing high urgency items count and details.  
    - Uses Slack Incoming Webhook URL.  
    - Message text includes count of high priority items.  
    - Attachments detail each high urgency item with color coding based on recommendation.  
    - Potential Failures: Slack webhook invalid or unreachable, payload formatting errors.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                        | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                               |
|-----------------------|---------------------|-------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------------|
| Daily Price Check      | Cron Trigger        | Starts workflow daily                | None                  | Fetch API Prices, Setup Database | -                                                                                                        |
| Fetch API Prices       | HTTP Request        | Retrieve ingredient prices from API | Daily Price Check     | Store Price Data           | -                                                                                                        |
| Setup Database        | PostgreSQL          | Create required DB tables if missing| Daily Price Check     | Store Price Data           | -                                                                                                        |
| Store Price Data       | PostgreSQL Insert   | Insert retrieved prices into DB     | Fetch API Prices, Setup Database | Calculate Trends           | -                                                                                                        |
| Calculate Trends       | PostgreSQL Query    | Compute price trends and changes    | Store Price Data      | Generate Recommendations   | -                                                                                                        |
| Generate Recommendations | Code (JavaScript) | Create buying recommendations       | Calculate Trends      | Store Recommendations      | -                                                                                                        |
| Store Recommendations  | PostgreSQL Insert   | Save recommendations to DB          | Generate Recommendations | Get Dashboard Data         | -                                                                                                        |
| Get Dashboard Data     | PostgreSQL Query    | Fetch latest recommendations        | Store Recommendations | Generate Dashboard HTML    | -                                                                                                        |
| Generate Dashboard HTML| Code (JavaScript)   | Build HTML dashboard for reporting  | Get Dashboard Data    | Send Email Report, Send Slack Alert | -                                                                                                        |
| Send Email Report      | Email Send          | Email dashboard report to users     | Generate Dashboard HTML| None                      | -                                                                                                        |
| Send Slack Alert       | HTTP Request        | Post alerts for high urgency items  | Generate Dashboard HTML| None                      | -                                                                                                        |
| Sticky Note            | Sticky Note         | Workflow overview summary           | None                  | None                      | ## 📌 Workflow Overview: Price Fluctuation Dashboard… (full content as in node)                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node named "Daily Price Check"**  
   - Set to run once daily at the desired time (default cron setting).  
   - No credentials needed.

2. **Create an HTTP Request node named "Fetch API Prices"**  
   - Set Method: GET  
   - URL: `https://api.example-food-prices.com/ingredients`  
   - No authentication configured but configure if needed by API.  
   - Connect "Daily Price Check" → "Fetch API Prices".

3. **Create a PostgreSQL node named "Setup Database"**  
   - Operation: Execute Query  
   - Query:  
     ```
     CREATE TABLE IF NOT EXISTS price_history (
       id SERIAL PRIMARY KEY,
       ingredient VARCHAR(100) NOT NULL,
       price DECIMAL(10,2) NOT NULL,
       unit VARCHAR(50) NOT NULL,
       supplier VARCHAR(100),
       timestamp TIMESTAMP NOT NULL,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
     );

     CREATE TABLE IF NOT EXISTS buying_recommendations (
       id SERIAL PRIMARY KEY,
       ingredient VARCHAR(100) NOT NULL,
       current_price DECIMAL(10,2) NOT NULL,
       price_change_percent DECIMAL(5,2),
       trend VARCHAR(20),
       recommendation VARCHAR(50),
       urgency VARCHAR(20),
       reason TEXT,
       generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
     );

     CREATE INDEX IF NOT EXISTS idx_price_history_ingredient_timestamp ON price_history(ingredient, timestamp);
     CREATE INDEX IF NOT EXISTS idx_recommendations_generated_at ON buying_recommendations(generated_at);
     ```  
   - Credentials: PostgreSQL credentials configured (e.g., "Postgres-test")  
   - Connect "Daily Price Check" → "Setup Database".

4. **Create a PostgreSQL Insert node named "Store Price Data"**  
   - Table: `price_history`  
   - Schema: `public`  
   - Map fields automatically from API response: ingredient, price, unit, supplier, timestamp, created_at (if available).  
   - Credentials: PostgreSQL credentials.  
   - Connect both "Fetch API Prices" → "Store Price Data" and "Setup Database" → "Store Price Data" (parallel trigger).  

5. **Create a PostgreSQL Query node named "Calculate Trends"**  
   - Operation: Execute Query  
   - Query:  
     ```
     WITH price_trends AS (
       SELECT 
         ingredient,
         price,
         timestamp,
         LAG(price) OVER (PARTITION BY ingredient ORDER BY timestamp) as prev_price,
         LAG(timestamp) OVER (PARTITION BY ingredient ORDER BY timestamp) as prev_timestamp
       FROM price_history
       WHERE timestamp >= NOW() - INTERVAL '30 days'
     )
     SELECT 
       ingredient,
       price as current_price,
       prev_price,
       CASE 
         WHEN prev_price IS NULL THEN 0
         ELSE ((price - prev_price) / prev_price) * 100
       END as price_change_percent,
       timestamp,
       CASE 
         WHEN price < prev_price THEN 'DECREASING'
         WHEN price > prev_price THEN 'INCREASING'
         ELSE 'STABLE'
       END as trend
     FROM price_trends
     WHERE prev_price IS NOT NULL
     ORDER BY ingredient, timestamp DESC;
     ```  
   - Credentials: PostgreSQL credentials.  
   - Connect "Store Price Data" → "Calculate Trends".

6. **Create a Code node named "Generate Recommendations"**  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const items = $input.all();
     const recommendations = [];

     for (const item of items) {
       const data = item.json;
       let recommendation = {
         ingredient: data.ingredient,
         current_price: data.current_price,
         price_change_percent: data.price_change_percent,
         trend: data.trend,
         recommendation: '',
         urgency: '',
         reason: ''
       };
       
       if (data.price_change_percent < -10) {
         recommendation.recommendation = 'BUY NOW';
         recommendation.urgency = 'HIGH';
         recommendation.reason = `Price dropped by ${Math.abs(data.price_change_percent).toFixed(1)}% - excellent buying opportunity`;
       } else if (data.price_change_percent < -5) {
         recommendation.recommendation = 'CONSIDER BUYING';
         recommendation.urgency = 'MEDIUM';
         recommendation.reason = `Price decreased by ${Math.abs(data.price_change_percent).toFixed(1)}% - good time to stock up`;
       } else if (data.price_change_percent > 15) {
         recommendation.recommendation = 'AVOID BUYING';
         recommendation.urgency = 'HIGH';
         recommendation.reason = `Price increased by ${data.price_change_percent.toFixed(1)}% - wait for better prices`;
       } else if (data.price_change_percent > 5) {
         recommendation.recommendation = 'WAIT';
         recommendation.urgency = 'MEDIUM';
         recommendation.reason = `Price increased by ${data.price_change_percent.toFixed(1)}% - consider delaying purchase`;
       } else {
         recommendation.recommendation = 'MONITOR';
         recommendation.urgency = 'LOW';
         recommendation.reason = 'Price stable - normal purchasing timing';
       }
       
       recommendations.push(recommendation);
     }

     return recommendations.map(rec => ({ json: rec }));
     ```  
   - Connect "Calculate Trends" → "Generate Recommendations".

7. **Create a PostgreSQL Insert node named "Store Recommendations"**  
   - Table: `buying_recommendations`  
   - Schema: `public`  
   - Auto-map fields: ingredient, current_price, price_change_percent, trend, recommendation, urgency, reason, generated_at.  
   - Credentials: PostgreSQL credentials.  
   - Connect "Generate Recommendations" → "Store Recommendations".

8. **Create a PostgreSQL Query node named "Get Dashboard Data"**  
   - Operation: Execute Query  
   - Query:  
     ```
     SELECT 
       ingredient,
       current_price,
       price_change_percent,
       trend,
       recommendation,
       urgency,
       reason,
       generated_at
     FROM buying_recommendations 
     WHERE generated_at >= CURRENT_DATE
     ORDER BY 
       CASE urgency 
         WHEN 'HIGH' THEN 1 
         WHEN 'MEDIUM' THEN 2 
         WHEN 'LOW' THEN 3 
       END,
       price_change_percent ASC;
     ```  
   - Credentials: PostgreSQL credentials.  
   - Connect "Store Recommendations" → "Get Dashboard Data".

9. **Create a Code node named "Generate Dashboard HTML"**  
   - Language: JavaScript  
   - Paste the provided HTML generation code (as detailed in workflow), which builds a styled dashboard with summary stats and cards per ingredient.  
   - Connect "Get Dashboard Data" → "Generate Dashboard HTML".

10. **Create an Email Send node named "Send Email Report"**  
    - Subject: `Daily Price Fluctuation Report - {{ $now.format('YYYY-MM-DD') }}`  
    - To Email: "abc@gmail.com" (replace with real recipient)  
    - From Email: "xyz@gmail.com" (replace with authorized sender)  
    - Attachments: Use expression to send dashboard HTML as base64 encoded attachment:  
      `data:text/html;base64,{{ $json.html | base64 }}`  
    - Credentials: SMTP credentials configured.  
    - Connect "Generate Dashboard HTML" → "Send Email Report".

11. **Create an HTTP Request node named "Send Slack Alert"**  
    - Method: POST  
    - URL: Slack Incoming Webhook URL (replace `https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK` with actual webhook)  
    - Body Type: JSON  
    - Body Parameters:  
      - text: `"🍽️ Daily Price Update: {{ $('Get Dashboard Data').all().filter(item => item.json.urgency === 'HIGH').length }} high priority items need attention!"`  
      - attachments: JSON array listing high urgency items with color coding and details, built from "Get Dashboard Data" output.  
    - Connect "Generate Dashboard HTML" → "Send Slack Alert".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow includes a sticky note summarizing the workflow architecture and main blocks for rapid understanding.                                                                                                                                                                                                                                                                                     | Sticky Note node "Workflow Overview"            |
| The PostgreSQL database is central for historical data tracking and recommendations persistence. Ensure credentials have appropriate permissions.                                                                                                                                                                                                                                                 | PostgreSQL credential "Postgres-test"            |
| Slack alerts are designed to highlight high urgency items only; customize webhook URL and message formatting as needed.                                                                                                                                                                                                                                                                             | Slack Incoming Webhook integration                |
| Email report uses SMTP credentials; ensure sender email is authorized to avoid spam filters.                                                                                                                                                                                                                                                                                                        | SMTP credential "SMTP account"                    |
| API endpoint `https://api.example-food-prices.com/ingredients` is a placeholder; replace with actual data source.                                                                                                                                                                                                                                                                                   | API HTTP Request node                             |
| The dashboard's HTML generation node uses inline CSS for styling; adjust as needed for branding or accessibility.                                                                                                                                                                                                                                                                                   | Code node "Generate Dashboard HTML"              |
| Timezone considerations for cron scheduling and database timestamps should be verified to align with business hours.                                                                                                                                                                                                                                                                               | Cron Trigger node "Daily Price Check"             |

---

This structured documentation enables thorough understanding, replication, and modification of the Ingredient Price Trend Analysis workflow using n8n. It highlights potential failure points and integration details for smooth operation.