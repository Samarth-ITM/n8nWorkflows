Competitive Price Monitoring & Alerts with Bright Data, Sheets & Slack

https://n8nworkflows.xyz/workflows/competitive-price-monitoring---alerts-with-bright-data--sheets---slack-9903


# Competitive Price Monitoring & Alerts with Bright Data, Sheets & Slack

### 1. Workflow Overview

This workflow automates competitive price monitoring by scraping competitor product prices daily using Bright Data's web scraping API. It compares competitor prices with your own, logs all data into Google Sheets for historical tracking, and sends alerts via Slack and email if competitors undercut your price beyond a defined threshold. Additionally, it generates and distributes a comprehensive daily summary report of all competitor price data.

**Target Use Cases:**  
- E-commerce businesses seeking real-time competitor price intelligence  
- Pricing teams needing automated alerts for underpricing events  
- Market analysts requiring historical price tracking and daily summaries  

**Logical Blocks:**  
- **1.1 Scheduled Trigger & Input Setup:** Initiates the workflow on a schedule and loads competitor URLs, product names, and pricing parameters.  
- **1.2 Competitor Processing Loop:** Iterates through each competitor URL to individually scrape pricing data.  
- **1.3 Data Scraping & Retrieval:** Triggers the Bright Data scraper, waits for completion, and fetches scraped product data.  
- **1.4 Data Parsing & Analysis:** Extracts prices from scraped data, calculates price differences, and determines if competitors are underpriced.  
- **1.5 Logging & Alerting:** Records all competitor pricing data to Google Sheets; sends Slack and email alerts if significant underpricing is detected.  
- **1.6 Daily Summary Reporting:** Aggregates all competitor data, computes statistics, and sends a daily summary report via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Input Setup

- **Overview:** Starts the workflow automatically on a defined schedule and configures competitor URLs, product names, current price, and alert threshold parameters.  
- **Nodes Involved:**  
  - Schedule Trigger  
  - Load Competitor URLs  

- **Node Details:**  

  1. **Schedule Trigger**  
     - Type: Schedule Trigger  
     - Role: Initiates workflow execution automatically on a schedule (e.g., daily or hourly).  
     - Configuration: Uses default interval (customizable as needed).  
     - Inputs: None (trigger node).  
     - Outputs: Connects to "Load Competitor URLs".  
     - Edge Cases: Misconfigured schedule might cause no or excessive triggering.  

  2. **Load Competitor URLs**  
     - Type: Set  
     - Role: Defines competitor URLs array, product names, your current price, and alert threshold for price difference percentage.  
     - Configuration:  
       - `competitors` is an array of objects with `name`, `url`, and `productName`.  
       - `ourPrice` is a numeric value (e.g., 149.99).  
       - `alertThreshold` is a numeric value representing percentage difference (e.g., 10%).  
     - Key Variables: `competitors`, `ourPrice`, `alertThreshold`.  
     - Inputs: From Schedule Trigger.  
     - Outputs: To "Loop Through Competitors".  
     - Edge Cases: Incorrect URL format or missing data in competitors array could cause scraping failures.

---

#### 1.2 Competitor Processing Loop

- **Overview:** Processes each competitor URL individually by splitting the competitors array into batches of one, enabling sequential scraping and processing per competitor.  
- **Nodes Involved:**  
  - Loop Through Competitors  

- **Node Details:**  

  1. **Loop Through Competitors**  
     - Type: Split In Batches  
     - Role: Iterates over each competitor entry one at a time to control scraping workflow per competitor.  
     - Configuration: Defaults (batch size of 1).  
     - Inputs: From "Load Competitor URLs".  
     - Outputs: To "Scrape with Bright Data" and to "Aggregate All Results" (final aggregation).  
     - Edge Cases: Large competitor lists could cause long execution times; batch size can be adjusted.

---

#### 1.3 Data Scraping & Retrieval

- **Overview:** Triggers Bright Data web scraper for each competitor URL, waits for scraping to complete, and then retrieves the scraped data.  
- **Nodes Involved:**  
  - Scrape with Bright Data  
  - Wait for Scraping  
  - Fetch Scraped Data  

- **Node Details:**  

  1. **Scrape with Bright Data**  
     - Type: HTTP Request  
     - Role: Sends a POST request to Bright Data API to trigger scraping for the current competitor URL.  
     - Configuration:  
       - URL: Bright Data API endpoint for dataset triggering.  
       - JSON Body includes dataset ID, endpoint, competitor URL, and discovery flag.  
       - Authentication: Header-based with generic HTTP header credentials (requires Bright Data API key).  
       - Headers: Content-Type set to application/json.  
       - Dynamic URL extracted from current competitor item.  
     - Inputs: From "Loop Through Competitors".  
     - Outputs: To "Wait for Scraping".  
     - Edge Cases: API authentication errors, invalid URLs, Bright Data service downtime, request timeouts.

  2. **Wait for Scraping**  
     - Type: Wait  
     - Role: Pauses workflow for 10 seconds to allow scraping to complete asynchronously.  
     - Configuration: Fixed wait time of 10 seconds.  
     - Inputs: From "Scrape with Bright Data".  
     - Outputs: To "Fetch Scraped Data".  
     - Edge Cases: Scrape might not be complete after 10 seconds if data is large or slow; could cause premature data fetching.

  3. **Fetch Scraped Data**  
     - Type: HTTP Request  
     - Role: Retrieves the scraping results from Bright Data using the snapshot ID returned from the trigger node.  
     - Configuration:  
       - URL dynamically constructed based on presence of `snapshot_id`.  
       - Authentication: Same as scraping trigger node (generic HTTP header auth).  
     - Inputs: From "Wait for Scraping".  
     - Outputs: To "Parse Price Data".  
     - Edge Cases: Snapshot ID missing or expired, API failures, malformed response.

---

#### 1.4 Data Parsing & Analysis

- **Overview:** Parses the returned scraped data to extract competitor prices, compares them against your own price, calculates percentage differences, and identifies if competitors are underpriced.  
- **Nodes Involved:**  
  - Parse Price Data  

- **Node Details:**  

  1. **Parse Price Data**  
     - Type: Code (JavaScript)  
     - Role: Processes raw scraped JSON to extract price values, cleans data, computes price difference percentage, and flags if competitor is underpriced.  
     - Key Logic:  
       - Extracts price from various fields (`price`, `data[0].price`, `data[0].final_price`) with fallback logic.  
       - Cleans currency symbols and formatting for parsing to float.  
       - Calculates price difference as ((competitorPrice - ourPrice) / ourPrice) * 100.  
       - Determines if competitor price is less than your price (underpriced).  
       - Outputs an array of enriched JSON objects with competitor info, prices, diffs, timestamps, and raw data.  
     - Inputs: From "Fetch Scraped Data" and references "Loop Through Competitors" & "Load Competitor URLs" for context values via expressions.  
     - Outputs: To "Log to Google Sheets", "Check If Alert Needed", and loops back to "Loop Through Competitors" for next item.  
     - Edge Cases: Missing or inconsistent price data in scraped payload, parseFloat failures, expression evaluation errors.

---

#### 1.5 Logging & Alerting

- **Overview:** Stores all competitor price checks into Google Sheets for historical tracking and sends alerts via Slack and email if a competitor's price undercuts your price beyond the alert threshold.  
- **Nodes Involved:**  
  - Log to Google Sheets  
  - Check If Alert Needed  
  - Send Slack Alert  
  - Send Email Alert  

- **Node Details:**  

  1. **Log to Google Sheets**  
     - Type: Google Sheets  
     - Role: Appends or updates rows in a Google Sheet to log competitor price data with columns for URL, date, product, prices, underpricing flag, and difference percentages.  
     - Configuration:  
       - Document ID and sheet name must be set to target spreadsheet.  
       - Columns mapped explicitly to JSON fields (e.g., `competitorUrl`, `scrapedAt`, `productName`, etc.).  
       - Operation: appendOrUpdate.  
     - Inputs: From "Parse Price Data".  
     - Outputs: To "Check If Alert Needed".  
     - Edge Cases: Google Sheets API quota limits, authentication errors, invalid spreadsheet ID.

  2. **Check If Alert Needed**  
     - Type: If  
     - Role: Condition node that checks if competitor is underpriced and the percentage difference exceeds the alert threshold to decide alerting actions.  
     - Configuration:  
       - Two conditions combined with AND:  
         - `isUnderpriced` is true  
         - `percentageDiff` > `alertThreshold` (from Load Competitor URLs node)  
     - Inputs: From "Log to Google Sheets".  
     - Outputs: On true: "Send Slack Alert" and "Send Email Alert"; on false: no output.  
     - Edge Cases: Missing fields or incorrect data types could cause condition failure.

  3. **Send Slack Alert**  
     - Type: Slack  
     - Role: Sends formatted alert message to Slack channel notifying that a competitor is undercutting pricing significantly.  
     - Configuration:  
       - Text includes competitor name, product, prices, difference, and URL with Slack markdown formatting.  
       - Requires Slack webhook or OAuth credentials configured.  
     - Inputs: From "Check If Alert Needed" (true branch).  
     - Outputs: To "Aggregate All Results".  
     - Edge Cases: Slack API rate limits, credential errors.

  4. **Send Email Alert**  
     - Type: Email Send  
     - Role: Sends email alert to pricing team with details of competitor underpricing event.  
     - Configuration:  
       - Subject includes competitor and product names.  
       - To and From emails specified.  
       - Requires SMTP or email credentials configured.  
     - Inputs: From "Check If Alert Needed" (true branch).  
     - Outputs: To "Aggregate All Results".  
     - Edge Cases: SMTP authentication failures, invalid email addresses.

---

#### 1.6 Daily Summary Reporting

- **Overview:** Collects all competitor price data for the current workflow run, computes aggregate statistics such as total competitors, number underpriced, average price difference, and lowest/highest competitor prices, then sends a summary report to Slack.  
- **Nodes Involved:**  
  - Aggregate All Results  
  - Create Daily Summary  
  - Send Daily Report to Slack  

- **Node Details:**  

  1. **Aggregate All Results**  
     - Type: Aggregate  
     - Role: Combines all items processed from competitor loop into a single collection to enable summary computations.  
     - Configuration: Default aggregation over all inputs.  
     - Inputs: From "Send Slack Alert" and "Send Email Alert" nodes (fan-in).  
     - Outputs: To "Create Daily Summary".  
     - Edge Cases: No items aggregated if no competitors processed or no alerts sent; careful handling needed downstream.

  2. **Create Daily Summary**  
     - Type: Code (JavaScript)  
     - Role: Generates a detailed summary report including total competitors monitored, count underpriced, average price difference, lowest and highest competitor prices, and your own price. Also formats competitor details for reporting.  
     - Key Logic:  
       - Checks if any competitor data exists, otherwise sets default summary.  
       - Uses array operations to compute aggregates and find min/max prices.  
       - Returns a single JSON object with all summary data.  
     - Inputs: From "Aggregate All Results".  
     - Outputs: To "Send Daily Report to Slack".  
     - Edge Cases: Empty input array, malformed data, division by zero.

  3. **Send Daily Report to Slack**  
     - Type: Slack  
     - Role: Posts the daily price monitoring summary to Slack channel with formatted message including summary statistics and price ranges.  
     - Configuration:  
       - Message uses Slack markdown with dynamic data fields from summary JSON.  
       - Requires Slack webhook or OAuth credentials configured.  
     - Inputs: From "Create Daily Summary".  
     - Outputs: None (end of workflow).  
     - Edge Cases: Slack API issues or credential errors.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                              | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                                                          |
|-------------------------|---------------------|----------------------------------------------|-------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Initiates workflow on schedule               | None                          | Load Competitor URLs               | Runs workflow automatically on schedule (daily/hourly)                                            |
| Load Competitor URLs    | Set                 | Defines competitor URLs, prices, thresholds | Schedule Trigger              | Loop Through Competitors            | Configure competitor URLs, product names, and alert thresholds                                     |
| Loop Through Competitors | Split In Batches     | Iterates through competitors one-by-one     | Load Competitor URLs          | Scrape with Bright Data, Aggregate All Results | Process each competitor URL one at a time                                                         |
| Scrape with Bright Data | HTTP Request        | Triggers Bright Data scraper for URL         | Loop Through Competitors      | Wait for Scraping                  | Trigger web scraper to extract competitor pricing data                                            |
| Wait for Scraping       | Wait                | Pauses to allow scraper to complete          | Scrape with Bright Data       | Fetch Scraped Data                 | Pause 10 seconds while scraper collects data                                                      |
| Fetch Scraped Data      | HTTP Request        | Retrieves scraped data from Bright Data       | Wait for Scraping             | Parse Price Data                  | Retrieve completed scraping results from Bright Data API                                         |
| Parse Price Data        | Code                | Extracts prices, computes differences        | Fetch Scraped Data            | Log to Google Sheets, Check If Alert Needed, Loop Through Competitors | Extract prices and calculate percentage differences vs yours                                      |
| Log to Google Sheets    | Google Sheets       | Records all price checks for tracking         | Parse Price Data              | Check If Alert Needed              | Record all price checks to spreadsheet for tracking                                              |
| Check If Alert Needed   | If                  | Determines if alert should be sent             | Log to Google Sheets          | Send Slack Alert, Send Email Alert | Determine if competitor price difference exceeds threshold                                       |
| Send Slack Alert        | Slack               | Sends alert message to Slack channel           | Check If Alert Needed         | Aggregate All Results             | Notify team via Slack when competitor undercuts pricing                                         |
| Send Email Alert        | Email Send          | Sends alert email to pricing team               | Check If Alert Needed         | Aggregate All Results             | Email team when significant competitor price drops detected                                      |
| Aggregate All Results   | Aggregate           | Combines all competitor data for summary      | Send Slack Alert, Send Email Alert | Create Daily Summary             | Collect all competitor data for summary report generation                                        |
| Create Daily Summary    | Code                | Generates detailed daily summary report        | Aggregate All Results         | Send Daily Report to Slack        | Calculate statistics: lowest, highest, average competitor prices                                 |
| Send Daily Report to Slack | Slack            | Posts daily summary report to Slack channel    | Create Daily Summary          | None                            | Deliver comprehensive daily summary to Slack channel                                             |
| Sticky Note             | Sticky Note         | Workflow branding and overview                   | None                         | None                            | # 📊 Competitive Price Monitoring & Alert System ... [LinkedIn link]                              |
| Sticky Note1            | Sticky Note         | Note on schedule trigger                         | None                         | None                            | Runs workflow automatically on schedule (daily/hourly)                                            |
| Sticky Note2            | Sticky Note         | Note on competitor configuration                 | None                         | None                            | Configure competitor URLs, product names, and alert thresholds                                    |
| Sticky Note3            | Sticky Note         | Note on batching competitors                      | None                         | None                            | Process each competitor URL one at a time                                                        |
| Sticky Note4            | Sticky Note         | Note on scraper trigger                           | None                         | None                            | Trigger web scraper to extract competitor pricing data                                           |
| Sticky Note5            | Sticky Note         | Note on wait time after trigger                    | None                         | None                            | Pause 10 seconds while scraper collects data                                                     |
| Sticky Note6            | Sticky Note         | Note on data retrieval                             | None                         | None                            | Retrieve completed scraping results from Bright Data API                                        |
| Sticky Note7            | Sticky Note         | Note on price parsing and calculation              | None                         | None                            | Extract prices and calculate percentage differences vs yours                                     |
| Sticky Note8            | Sticky Note         | Note on logging to spreadsheet                      | None                         | None                            | Record all price checks to spreadsheet for tracking                                             |
| Sticky Note9            | Sticky Note         | Note on alert condition checking                     | None                         | None                            | Determine if competitor price difference exceeds threshold                                      |
| Sticky Note10           | Sticky Note         | Note on Slack alerts                                 | None                         | None                            | Notify team via Slack when competitor undercuts pricing                                         |
| Sticky Note11           | Sticky Note         | Note on email alerts                                 | None                         | None                            | Email team when significant competitor price drops detected                                     |
| Sticky Note12           | Sticky Note         | Note on data aggregation                             | None                         | None                            | Collect all competitor data for summary report generation                                       |
| Sticky Note13           | Sticky Note         | Note on daily summary statistics                      | None                         | None                            | Calculate statistics: lowest, highest, average competitor prices                                |
| Sticky Note14           | Sticky Note         | Note on sending daily summary report                   | None                         | None                            | Deliver comprehensive daily summary to Slack channel                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set schedule interval as needed (e.g., daily at specific time).  
   - No credentials required.  

2. **Create Set node named "Load Competitor URLs"**  
   - Type: Set  
   - Add variables:  
     - `competitors` (Array) containing objects with keys: `name`, `url`, `productName` (configure your competitor URLs and product names).  
     - `ourPrice` (Number) e.g., 149.99.  
     - `alertThreshold` (Number) e.g., 10 (percentage).  
   - No credentials required.  
   - Connect Schedule Trigger → Load Competitor URLs.  

3. **Create Split In Batches node named "Loop Through Competitors"**  
   - Type: Split In Batches  
   - Set batch size to 1 (default).  
   - Connect Load Competitor URLs → Loop Through Competitors.  

4. **Create HTTP Request node named "Scrape with Bright Data"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Authentication: HTTP Header Authentication using Bright Data API key credential (create credential with your API key).  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "dataset_id": "gd_l7q7dkf244hwjntr0",
       "endpoint": "https://api.brightdata.com/datasets/v3/snapshot/gd_l7q7dkf244hwjntr0?format=json",
       "url": "={{ $json.competitors[$itemIndex].url }}",
       "discover_new_sites": false
     }
     ```  
   - Connect Loop Through Competitors → Scrape with Bright Data.  

5. **Create Wait node named "Wait for Scraping"**  
   - Type: Wait  
   - Set wait time to 10 seconds.  
   - Connect Scrape with Bright Data → Wait for Scraping.  

6. **Create HTTP Request node named "Fetch Scraped Data"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL:  
     ```javascript
     ={{ $json.snapshot_id ? 'https://api.brightdata.com/datasets/v3/snapshot/' + $json.snapshot_id + '?format=json' : 'https://api.brightdata.com/datasets/v3/progress/' + $json.snapshot_id }}
     ```  
   - Authentication: Use same Bright Data HTTP Header Auth credentials.  
   - Connect Wait for Scraping → Fetch Scraped Data.  

7. **Create Code node named "Parse Price Data"**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code that:  
     - Extracts competitor price from multiple possible fields.  
     - Cleans currency symbols and parses to float.  
     - Calculates price difference and underpriced flag.  
     - Outputs structured JSON objects with competitor data.  
   - Connect Fetch Scraped Data → Parse Price Data.  

8. **Create Google Sheets node named "Log to Google Sheets"**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Set Document ID to your Google Sheet containing price history.  
   - Set Sheet Name (gid=0 or your appropriate sheet).  
   - Map columns: URL, Date, Product, Our Price, Competitor, Their Price, Underpriced, Difference %.  
   - Connect Parse Price Data → Log to Google Sheets.  
   - Configure Google Sheets OAuth2 credentials.  

9. **Create If node named "Check If Alert Needed"**  
   - Type: If  
   - Conditions: AND combination of  
     - `isUnderpriced` is true  
     - `percentageDiff` > `alertThreshold` (from Load Competitor URLs)  
   - Connect Log to Google Sheets → Check If Alert Needed.  

10. **Create Slack node named "Send Slack Alert"**  
    - Type: Slack  
    - Configure Slack credentials with webhook or OAuth2.  
    - Message text with competitor and pricing details formatted using expressions.  
    - Connect Check If Alert Needed (true) → Send Slack Alert.  

11. **Create Email Send node named "Send Email Alert"**  
    - Type: Email Send  
    - Configure SMTP/email credentials.  
    - Set "To": pricing-team@yourcompany.com (replace as needed).  
    - Set "From": alerts@yourcompany.com (replace as needed).  
    - Subject includes competitor and product info.  
    - Connect Check If Alert Needed (true) → Send Email Alert.  

12. **Create Aggregate node named "Aggregate All Results"**  
    - Type: Aggregate  
    - Connect Send Slack Alert → Aggregate All Results.  
    - Connect Send Email Alert → Aggregate All Results.  

13. **Create Code node named "Create Daily Summary"**  
    - Type: Code (JavaScript)  
    - Paste JavaScript code that:  
      - Calculates total competitors, underpriced count, average difference.  
      - Finds lowest and highest competitor prices.  
      - Returns a JSON summary.  
    - Connect Aggregate All Results → Create Daily Summary.  

14. **Create Slack node named "Send Daily Report to Slack"**  
    - Type: Slack  
    - Configure Slack credentials.  
    - Message text formatted with summary statistics from previous node.  
    - Connect Create Daily Summary → Send Daily Report to Slack.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Automates daily competitor price checks using Bright Data Web Scraper API. Compares prices, logs data to Google Sheets, alerts via Slack/email. | Workflow description and use case overview.                                                        |
| Built by Daniel Shashko. Connect on LinkedIn: https://www.linkedin.com/in/daniel-shashko/                                                        | Author and professional contact.                                                                    |
| Setup requires Bright Data API key, Google Sheets OAuth2 credentials, Slack webhook or OAuth, and SMTP/email credentials.                        | Credential configuration instructions.                                                             |
| Customize URLs, alert thresholds, schedule, and notification channels as needed.                                                                  | Flexibility guidance.                                                                                |
| Ensure handling of edge cases such as missing data, API errors, and scraping delays by adjusting wait times or adding error workflows if needed. | Operational advice for robustness.                                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.