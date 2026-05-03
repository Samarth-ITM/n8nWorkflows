Price Monitoring Dashboard with AI Component and Alerts

https://n8nworkflows.xyz/workflows/price-monitoring-dashboard-with-ai-component-and-alerts-6428


# Price Monitoring Dashboard with AI Component and Alerts

### 1. Workflow Overview

This workflow, **Competitor Price Monitoring**, is designed for e-commerce businesses or analysts to automatically track and analyze competitor pricing for wireless headphones across multiple major retailers. It integrates scheduled and manual triggers to initiate price scraping, uses AI-powered extraction for structured product data, conducts intelligent price and competitive analyses, stores data in Google Sheets, and sends prioritized alerts via Slack for significant price changes.

The workflow is logically divided into five main blocks:

- **1.1 Triggers**: Initiate the workflow either on a daily schedule or on-demand via webhook.
- **1.2 Multi-Platform Scraping**: Fetch raw product pages from Amazon, Best Buy, and Target with custom HTTP requests.
- **1.3 AI Price Data Extraction**: Use ScrapeGraphAI to parse complex web pages and extract structured pricing and product details.
- **1.4 Price Analysis & Intelligence**: Analyze product pricing trends, detect significant changes, and generate competitive intelligence.
- **1.5 Data Storage & Alerts**: Save processed data to Google Sheets and send Slack alerts for meaningful price changes.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggers

**Overview:**  
This block enables both automated daily checks and manual on-demand price monitoring through a webhook endpoint, ensuring flexible and consistent data collection.

**Nodes Involved:**  
- Daily Price Check Trigger  
- Manual Price Check Webhook  
- Sticky Note - Triggers

**Node Details:**  

- **Daily Price Check Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every 24 hours.  
  - Configuration: Set to run every 24 hours (hourly interval of 24).  
  - Inputs: None (start node)  
  - Outputs: Connects to all three scrapers (Amazon, Best Buy, Target).  
  - Edge Cases: Possible issues if n8n is down during scheduled time; no retry configured.  
  - Notes: Recommended to schedule during off-peak hours for minimal rate limiting.

- **Manual Price Check Webhook**  
  - Type: Webhook Trigger  
  - Role: Allows external systems or manual calls to trigger a price check immediately.  
  - Configuration: HTTP GET method; path: `/price-check-webhook`  
  - Inputs: HTTP request  
  - Outputs: Connects to all three scrapers.  
  - Edge Cases: Unauthorized calls not restricted; consider adding authentication if exposed publicly.  
  - Notes: Useful for ad-hoc price checks or integration with other tools.

- **Sticky Note - Triggers**  
  - Type: Sticky Note  
  - Content: Explains dual trigger system benefits and configurations.

---

#### 2.2 Multi-Platform Scraping

**Overview:**  
This block performs parallel HTTP requests to scrape product search pages from Amazon, Best Buy, and Target using custom headers to mimic browser requests.

**Nodes Involved:**  
- Amazon Price Scraper  
- Best Buy Price Scraper  
- Target Price Scraper  
- Sticky Note - Scraping

**Node Details:**  

- **Amazon Price Scraper**  
  - Type: HTTP Request  
  - Role: Fetches Amazon search results for "wireless headphones".  
  - Configuration: URL set to Amazon search, custom User-Agent header to avoid bot detection.  
  - Inputs: Trigger nodes (daily/manual)  
  - Outputs: Passes raw HTML to AI Price Data Extractor.  
  - Edge Cases: Amazon might block IPs or show CAPTCHA; no explicit retry or error handling configured.

- **Best Buy Price Scraper**  
  - Type: HTTP Request  
  - Role: Fetches Best Buy search results for wireless headphones.  
  - Configuration: Similar to Amazon, with User-Agent header.  
  - Inputs: Trigger nodes  
  - Outputs: To AI Price Data Extractor.  
  - Edge Cases: Same as Amazon; dynamic content might affect scraping.

- **Target Price Scraper**  
  - Type: HTTP Request  
  - Role: Fetches Target search results.  
  - Configuration: Similar approach, User-Agent header included.  
  - Inputs: Trigger nodes  
  - Outputs: To AI Price Data Extractor.  
  - Edge Cases: Dynamic loading or anti-bot measures may impact results.

- **Sticky Note - Scraping**  
  - Type: Sticky Note  
  - Content: Describes scraping approach, parallel processing, extensibility, and anti-detection.

---

#### 2.3 AI Price Data Extraction

**Overview:**  
This block uses ScrapeGraphAI to intelligently extract structured product data (pricing, brand, ratings, availability, etc.) from the raw HTML responses, overcoming challenges of dynamic content and inconsistent site layouts.

**Nodes Involved:**  
- AI Price Data Extractor  
- Sticky Note - AI Extraction

**Node Details:**  

- **AI Price Data Extractor**  
  - Type: ScrapeGraphAI Node  
  - Role: Parses raw HTML to extract product pricing and details into a structured JSON schema.  
  - Configuration:  
    - UserPrompt explicitly defines schema for product data extraction including prices, discounts, ratings, availability, URLs, images, and features.  
    - Website URL dynamically taken from previous node’s JSON URL or defaults to Amazon search page.  
  - Inputs: From all three scraper HTTP requests.  
  - Outputs: Structured product data to Price Analysis & Intelligence node.  
  - Credentials: Requires ScrapeGraphAI API credentials.  
  - Edge Cases: Potential API rate limits; extraction inaccuracies if site structure changes; fallback URL set.  
  - Notes: AI enables adaptability to different site layouts with high accuracy.

- **Sticky Note - AI Extraction**  
  - Type: Sticky Note  
  - Content: Explains smart extraction capabilities, data points, and AI advantages.

---

#### 2.4 Price Analysis & Intelligence

**Overview:**  
Processes extracted product data to detect significant price changes, analyze discounts and sales, generate competitive intelligence metrics, and prioritize alerts.

**Nodes Involved:**  
- Price Analysis & Intelligence (Code Node)  
- Price Change Alert Filter  
- Sticky Note - Analysis

**Node Details:**  

- **Price Analysis & Intelligence**  
  - Type: Code Node (JavaScript)  
  - Role:  
    - Analyzes each product for pricing insights (on sale, discount levels, value scores).  
    - Detects significant price changes based on thresholds.  
    - Generates competitive intelligence (market position, brand strength, customer satisfaction).  
    - Filters products to process only target brands or high-rated products.  
  - Configuration:  
    - Configurable thresholds (15% drop, 10% increase, 20% discount).  
    - Competitor weights and priorities defined for Amazon, Best Buy, and Target.  
    - Generates unique product IDs.  
  - Inputs: Structured product JSON from AI extractor.  
  - Outputs: Enriched product data to Google Sheets and alert filter.  
  - Edge Cases: No historical data comparison implemented (simulated logic used); if new products appear frequently, false alerts possible.  
  - Notes: Important to extend for real database integration for historical price tracking.

- **Price Change Alert Filter**  
  - Type: If Node  
  - Role: Filters products to identify those deserving alerts based on:  
    - Alert-worthy flag true, OR  
    - Discount percentage >= 20%, OR  
    - Change significance equals "high"  
  - Inputs: From analysis node.  
  - Outputs: Sends qualifying products to Slack Price Alert.  
  - Edge Cases: Boolean logic uses "any" condition; ensure consistent data types to avoid filtering errors.

- **Sticky Note - Analysis**  
  - Type: Sticky Note  
  - Content: Details price change detection, competitive intelligence, and prioritization scheme.

---

#### 2.5 Data Storage & Alerts

**Overview:**  
Stores the enriched product pricing data in Google Sheets for historical tracking and sends formatted Slack alerts for significant price changes or discounts to notify stakeholders.

**Nodes Involved:**  
- Google Sheets Price Log  
- Slack Price Alert  
- Sticky Note - Storage & Alerts

**Node Details:**  

- **Google Sheets Price Log**  
  - Type: Google Sheets Node  
  - Role: Appends or updates product pricing data and metadata in a Google Sheet.  
  - Configuration:  
    - Uses service account credentials for authentication.  
    - Auto-maps input data to predefined columns including product ID, prices, brand, rating, availability, sale status, market position, priority, timestamps, and URLs.  
    - Key matching column: `product_id` for append or update operations.  
    - Sheet and document IDs are configured with placeholders (replace with actual IDs).  
  - Inputs: From Price Analysis & Intelligence node.  
  - Outputs: None (terminal for data storage).  
  - Edge Cases: Google API quota limits, credential expiration, and data mapping mismatches. Proper error handling recommended.

- **Slack Price Alert**  
  - Type: Slack Node  
  - Role: Sends rich formatted notifications to a Slack channel for prioritized price alerts.  
  - Configuration:  
    - Message template includes product info, pricing, discounts, source, ratings, availability, and market intelligence.  
    - Uses Slack OAuth2 credentials.  
    - Sends to channel `"C1234567890"` (replace with actual channel ID).  
  - Inputs: From Price Change Alert Filter node (only filtered alerts).  
  - Outputs: None (terminal).  
  - Edge Cases: Slack API rate limits; OAuth token expiration; channel permissions.

- **Sticky Note - Storage & Alerts**  
  - Type: Sticky Note  
  - Content: Describes storage capabilities, alert criteria, and analytics features.

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                                    | Input Node(s)                          | Output Node(s)                    | Sticky Note                                                                 |
|---------------------------|-------------------------|---------------------------------------------------|--------------------------------------|----------------------------------|-----------------------------------------------------------------------------|
| Daily Price Check Trigger  | Schedule Trigger        | Initiates daily automated price scraping          | None                                 | Amazon, Best Buy, Target Scrapers | Yes - Sticky Note - Triggers                                                |
| Manual Price Check Webhook | Webhook Trigger         | Enables on-demand manual price scraping            | None                                 | Amazon, Best Buy, Target Scrapers | Yes - Sticky Note - Triggers                                                |
| Amazon Price Scraper       | HTTP Request            | Fetches Amazon product search page                 | Daily Price Check, Manual Webhook    | AI Price Data Extractor           | Yes - Sticky Note - Scraping                                                |
| Best Buy Price Scraper     | HTTP Request            | Fetches Best Buy product search page                | Daily Price Check, Manual Webhook    | AI Price Data Extractor           | Yes - Sticky Note - Scraping                                                |
| Target Price Scraper       | HTTP Request            | Fetches Target product search page                  | Daily Price Check, Manual Webhook    | AI Price Data Extractor           | Yes - Sticky Note - Scraping                                                |
| AI Price Data Extractor    | ScrapeGraphAI Node      | Extracts structured pricing/product data via AI    | Amazon, Best Buy, Target Scrapers    | Price Analysis & Intelligence     | Yes - Sticky Note - AI Extraction                                           |
| Price Analysis & Intelligence | Code Node            | Analyzes pricing, detects changes, generates intel | AI Price Data Extractor               | Google Sheets Price Log, Price Change Alert Filter | Yes - Sticky Note - Analysis                                 |
| Price Change Alert Filter  | If Node                 | Filters products for alert-worthy price changes    | Price Analysis & Intelligence         | Slack Price Alert                |                                                                             |
| Google Sheets Price Log    | Google Sheets Node      | Logs product pricing and metadata to Google Sheets | Price Analysis & Intelligence         | None                            | Yes - Sticky Note - Storage & Alerts                                        |
| Slack Price Alert          | Slack Node              | Sends formatted Slack alerts for significant price changes | Price Change Alert Filter          | None                            | Yes - Sticky Note - Storage & Alerts                                        |
| Sticky Note - Triggers     | Sticky Note             | Explains trigger system                             | None                                 | None                            | Covers Daily Price Check Trigger, Manual Price Check Webhook                |
| Sticky Note - Scraping     | Sticky Note             | Describes scraping approach and features           | None                                 | None                            | Covers Amazon, Best Buy, Target Scrapers                                   |
| Sticky Note - AI Extraction| Sticky Note             | Details AI extraction capabilities                  | None                                 | None                            | Covers AI Price Data Extractor                                             |
| Sticky Note - Analysis     | Sticky Note             | Explains price analysis and intelligence logic     | None                                 | None                            | Covers Price Analysis & Intelligence, Price Change Alert Filter            |
| Sticky Note - Storage & Alerts | Sticky Note         | Describes data storage and alerting mechanisms     | None                                 | None                            | Covers Google Sheets Price Log, Slack Price Alert                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Daily Price Check Trigger” node**  
   - Type: Schedule Trigger  
   - Set interval to every 24 hours (hoursInterval = 24).  
   - No inputs.  

2. **Create “Manual Price Check Webhook” node**  
   - Type: Webhook Trigger  
   - HTTP Method: GET  
   - Path: `price-check-webhook`  
   - No authentication configured (optional to add).  

3. **Create “Amazon Price Scraper” node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.amazon.com/s?k=wireless+headphones`  
   - Headers: `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36`  
   - Connect both triggers to this node.  

4. **Create “Best Buy Price Scraper” node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.bestbuy.com/site/searchpage.jsp?st=wireless+headphones`  
   - Same User-Agent header as Amazon scraper.  
   - Connect both triggers to this node.  

5. **Create “Target Price Scraper” node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.target.com/s?searchTerm=wireless+headphones`  
   - Same User-Agent header.  
   - Connect both triggers to this node.  

6. **Create “AI Price Data Extractor” node**  
   - Type: ScrapeGraphAI  
   - Set “User Prompt” with JSON schema detailing product fields (product name, brand, prices, discount, ratings, availability, shipping, prime eligibility, features, etc.)  
   - Set “Website URL” parameter as expression: `{{$json.url || 'https://www.amazon.com/s?k=wireless+headphones'}}` to dynamically use input URL or default.  
   - Add ScrapeGraphAI API credentials.  
   - Connect all three scraper nodes to this node.  

7. **Create “Price Analysis & Intelligence” node**  
   - Type: Code (JavaScript)  
   - Insert provided JavaScript code that:  
     - Filters to target brands or high-rated products  
     - Calculates pricing insights and discount levels  
     - Detects price changes (simulated logic)  
     - Generates competitive intelligence metrics  
     - Adds metadata and tracking priority  
   - Connect AI Price Data Extractor node to this node.  

8. **Create “Google Sheets Price Log” node**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Configure with service account credentials (Google Sheets OAuth2 or Service Account)  
   - Set Document ID and Sheet Name (gid) to your target spreadsheet  
   - Define columns to map from input JSON, with `product_id` as matching column  
   - Connect Price Analysis & Intelligence node to this node.  

9. **Create “Price Change Alert Filter” node**  
   - Type: If node  
   - Conditions: Any of the following true:  
     - `alert_worthy` boolean equals true  
     - `discount_percentage` number greater or equal 20  
     - `change_significance` string equals "high"  
   - Connect Price Analysis & Intelligence node to this node (parallel to Google Sheets node).  

10. **Create “Slack Price Alert” node**  
    - Type: Slack  
    - Authentication: OAuth2 with Slack app credentials  
    - Set channel to your target Slack channel (e.g., channel ID `C1234567890`)  
    - Message: Use provided markdown template with product details and alert highlights  
    - Connect Price Change Alert Filter’s true output to this node.  

11. **Add Sticky Notes** for documentation:  
    - Step 1: Triggers  
    - Step 2: Multi-Platform Scraping  
    - Step 3: AI Extraction  
    - Step 4: Analysis  
    - Step 5: Storage & Alerts  

12. **Set Execution Order:**  
    - Triggers → Scrapers → AI Extraction → Analysis → [Google Sheets & Alert Filter] → Slack Alert  

13. **Test end-to-end:**  
    - Trigger manual webhook or wait for schedule  
    - Confirm scraping, AI extraction, analysis, data logging, and Slack alert delivery  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow uses ScrapeGraphAI for robust extraction of complex e-commerce pages with dynamic content.                                                                  | Requires ScrapeGraphAI API credentials and subscription.                                          |
| Slack alerts are richly formatted to highlight significant price changes, discounts, and market intelligence for actionable insights.                               | Ensure Slack OAuth2 credentials and proper channel permissions are configured.                     |
| Google Sheets integration uses service account for secure, automated data logging with idempotent append-or-update operation keyed by product ID.                    | Replace placeholder Google Sheet IDs with actual document and sheet IDs.                          |
| Manual webhook trigger enables integration with external systems or manual checks without waiting for scheduled runs.                                               | Endpoint path: `/price-check-webhook`; consider securing with authentication if exposed publicly. |
| The JavaScript code simulates price change detection due to lack of historical database; extending with persistent storage is recommended for accuracy improvement. | Ideal extension: connect to a database or enhanced Google Sheets logic for historical comparison. |

---

_Disclaimer: The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._