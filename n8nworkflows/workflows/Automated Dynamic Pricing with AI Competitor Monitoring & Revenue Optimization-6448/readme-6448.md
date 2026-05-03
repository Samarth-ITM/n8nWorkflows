Automated Dynamic Pricing with AI Competitor Monitoring & Revenue Optimization

https://n8nworkflows.xyz/workflows/automated-dynamic-pricing-with-ai-competitor-monitoring---revenue-optimization-6448


# Automated Dynamic Pricing with AI Competitor Monitoring & Revenue Optimization

### 1. Workflow Overview

**Purpose:**  
This workflow automates dynamic pricing for e-commerce products by continuously monitoring competitor prices, analyzing market demand and customer sentiment, and optimizing product prices in real-time to maximize revenue and maintain margins.

**Target Use Cases:**  
- Online retailers managing multiple products with frequent price changes  
- Businesses needing real-time competitor price tracking across multiple platforms  
- Revenue management teams aiming to optimize profitability with AI-driven insights  
- Automated workflows integrating pricing decisions with e-commerce platforms, analytics, and notifications

**Logical Blocks:**

- **1.1 Input Reception and Triggering:**  
  Hourly execution trigger initiating the pricing process.

- **1.2 Pricing Configuration and Task Generation:**  
  Processes product catalog, competitor info, and market context; generates individual monitoring tasks.

- **1.3 Task Distribution and Preparation:**  
  Splits monitoring tasks into batches and prepares detailed, context-rich queries.

- **1.4 AI Data Collection:**  
  Parallel scraping nodes that collect competitor prices, demand analytics, and customer sentiment data.

- **1.5 Data Integration:**  
  Merges collected data into a unified dataset for pricing analysis.

- **1.6 Pricing Optimization Engine:**  
  Calculates optimal prices based on multi-factor AI analysis including competitor pricing, demand, sentiment, inventory, and margin.

- **1.7 Decision Filtering:**  
  Filters price changes based on significance and confidence before applying.

- **1.8 Execution and Logging:**  
  Updates e-commerce platform prices, logs pricing and revenue data into Google Sheets, and sends alerts and reports via Slack and email.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Triggering

- **Overview:**  
  Initiates the workflow every hour to ensure timely price updates.

- **Nodes Involved:**  
  - Hourly Pricing Monitor Trigger

- **Node Details:**  
  - *Hourly Pricing Monitor Trigger*  
    - Type: Schedule Trigger  
    - Role: Automates hourly initiation of pricing workflow  
    - Configuration: Fires every 1 hour (configurable interval)  
    - Inputs: None (trigger node)  
    - Outputs: Triggers "Pricing Configuration Processor"  
    - Edge Cases: Potential scheduling errors or system downtime can delay execution  
    - Notes: Balances monitoring frequency with API limits and cost considerations

#### 1.2 Pricing Configuration and Task Generation

- **Overview:**  
  Processes the entire product catalog, competitor platform configurations, and current market timing to generate a comprehensive set of monitoring tasks.

- **Nodes Involved:**  
  - Pricing Configuration Processor

- **Node Details:**  
  - *Pricing Configuration Processor*  
    - Type: Code Node (JavaScript)  
    - Role: Defines product profiles, competitor platforms, pricing strategies, and market demand factors; generates search URLs and monitoring tasks per product-competitor-keyword combination  
    - Key Configurations:  
      - Product catalog with pricing rules and inventory data  
      - Competitor websites with URL templates and price CSS selectors  
      - Pricing strategies (aggressive, competitive, premium) based on inventory levels  
      - Market context: time of day, day of week, quarter seasonality multipliers  
      - Generates session ID and timestamp for tracking  
    - Inputs: Trigger node  
    - Outputs: JSON containing session info, market context, config, and a large array of monitoring tasks  
    - Edge Cases: Incorrect time zone handling could misclassify demand factors; product data inaccuracies affect downstream logic  
    - Version: n8n code node v2  

#### 1.3 Task Distribution and Preparation

- **Overview:**  
  The large monitoring task list is split into batches for parallel processing, and each task is enriched with contextual information for accurate scraping.

- **Nodes Involved:**  
  - Split Monitoring Tasks  
  - Task Processor

- **Node Details:**  
  - *Split Monitoring Tasks*  
    - Type: Split in Batches  
    - Role: Divides the total monitoring tasks array into manageable batches for parallel scraping  
    - Configuration: Default batch size (not explicitly set)  
    - Inputs: Output of Pricing Configuration Processor  
    - Outputs: Batches of monitoring tasks  
    - Edge Cases: Large batch sizes could lead to API rate limits or timeouts  

  - *Task Processor*  
    - Type: Code Node  
    - Role: Adds market context and config data to each monitoring task for scraping nodes  
    - Key Expressions: Accesses batch index and total batches for session tracking  
    - Inputs: Batch item from Split Monitoring Tasks and full configuration data  
    - Outputs: Enhanced monitoring task with URLs and metadata for scrapers  
    - Edge Cases: Misalignment between batch index and task array could cause incorrect data assignment  
    - Version: n8n code node v2  

#### 1.4 AI Data Collection

- **Overview:**  
  Three AI scraping nodes run in parallel to collect comprehensive market data from multiple sources, including competitor prices, demand trends, and customer sentiment.

- **Nodes Involved:**  
  - AI Competitor Price Scraper  
  - AI Demand Analysis Scraper  
  - AI Customer Sentiment Scraper

- **Node Details:**  
  Each node uses ScrapeGraph AI to extract structured data using custom prompts and URLs generated per task.

  - *AI Competitor Price Scraper*  
    - Type: scrapegraphAi node  
    - Role: Extracts competitor pricing, discounts, ratings, and availability from e-commerce sites  
    - Configuration: Uses user prompt with detailed schema for product matching and pricing stats; dynamic URL from task data  
    - Inputs: Task Processor output  
    - Outputs: Competitor pricing data JSON  
    - Edge Cases: Website layout changes may break selectors; API quota or scraping limits; noisy data from irrelevant products  

  - *AI Demand Analysis Scraper*  
    - Type: scrapegraphAi node  
    - Role: Analyzes Google Trends data for search volume and market demand trends  
    - Configuration: Custom prompt extracting search volume trends, related queries, and regional interest; URL built from encoded search keyword  
    - Inputs: Task Processor output  
    - Outputs: Demand trend data JSON  
    - Edge Cases: Google Trends data latency or format changes; low search volume keywords may yield sparse data  

  - *AI Customer Sentiment Scraper*  
    - Type: scrapegraphAi node  
    - Role: Analyzes customer reviews and sentiment on product categories to understand price sensitivity  
    - Configuration: Custom prompt for sentiment analysis and price elasticity indicators; URL targets Amazon reviews sorted by rank  
    - Inputs: Task Processor output  
    - Outputs: Sentiment data JSON  
    - Edge Cases: Review data sparsity or bias; changes in review site structure; sentiment misclassification  

#### 1.5 Data Integration

- **Overview:**  
  Combines outputs from all AI scraping nodes into a single cohesive dataset to feed into the pricing optimization algorithm.

- **Nodes Involved:**  
  - Merge Pricing Data

- **Node Details:**  
  - *Merge Pricing Data*  
    - Type: Merge (Combine mode)  
    - Role: Aligns and merges competitor, demand, and sentiment data outputs by their execution position  
    - Inputs: Outputs from the three AI scraping nodes  
    - Outputs: Single merged JSON object per monitoring task  
    - Edge Cases: Missing data from any scraper handled gracefully; data misalignment if nodes run asynchronously with different speeds  
    - Version: n8n merge node v2.1  

#### 1.6 Pricing Optimization Engine

- **Overview:**  
  Runs an advanced multi-factor AI algorithm that calculates recommended prices based on competitor analysis, demand trends, customer sentiment, inventory levels, and margin protection.

- **Nodes Involved:**  
  - Pricing Optimization Engine

- **Node Details:**  
  - *Pricing Optimization Engine*  
    - Type: Code Node  
    - Role: Implements weighted scoring model combining competitor pricing position, demand scores, sentiment, inventory status, and margin analysis to output optimal price and strategy  
    - Key Logic:  
      - Weighted factors with specified influence percentages (e.g., competitor 35%, demand 25%)  
      - Conversion of optimization score to constrained price adjustment (max 15% change, min 2% threshold)  
      - Margin and price boundary enforcement  
      - Confidence calculation based on data completeness and volume  
      - Strategy recommendation (aggressive, competitive, premium) based on score and inventory  
    - Inputs: Merged pricing data, original task data  
    - Outputs: Comprehensive pricing analysis JSON including recommended price, change metrics, confidence, strategy, and revenue projections  
    - Edge Cases: Missing data handled with defaults; extreme market conditions might skew scores; rounding applied to prices  
    - Version: n8n code node v2  

#### 1.7 Decision Filtering

- **Overview:**  
  Filters pricing recommendations to apply only significant and confident price changes, reducing noise and protecting against risky updates.

- **Nodes Involved:**  
  - Price Change Filter

- **Node Details:**  
  - *Price Change Filter*  
    - Type: If Node  
    - Role: Allows price updates only if absolute price change percentage > 2% AND confidence level ≥ 0.7  
    - Inputs: Pricing Optimization Engine output  
    - Outputs:  
      - True branch: Proceed with price update and notifications  
      - False branch: Skip price update  
    - Edge Cases: Thresholds can be tuned; false negatives may delay profitable updates; false positives minimized by confidence gating  
    - Version: n8n if node v2  

#### 1.8 Execution and Logging

- **Overview:**  
  Applies approved price changes to the e-commerce system, logs detailed pricing and revenue data to Google Sheets, and sends alerts and reports to Slack and email.

- **Nodes Involved:**  
  - Price Update API Call  
  - Pricing History Logger  
  - Revenue Analytics Logger  
  - Pricing Alert Sender  
  - Pricing Report Sender

- **Node Details:**  
  - *Price Update API Call*  
    - Type: HTTP Request  
    - Role: Sends PATCH/POST request to e-commerce API to update product price  
    - Configurations:  
      - URL dynamic per product ID  
      - Body includes new price, previous price, reason, confidence, effective date  
      - Authorization header with Bearer token (replace YOUR_API_TOKEN)  
      - Content-Type application/json  
    - Inputs: Price Change Filter (true branch)  
    - Outputs: API response (optional)  
    - Edge Cases: API failures, token expiration, rate limits, network errors  

  - *Pricing History Logger*  
    - Type: Google Sheets  
    - Role: Appends pricing change records with detailed metadata to "Pricing_History" sheet  
    - Configurations: Spreadsheet ID and sheet name provided  
    - Logged fields: Strategy, prices, timestamps, confidence, competitor, session ID, revenue impact, sentiment, market context  
    - Inputs: Pricing Optimization Engine output (via Price Change Filter)  
    - Edge Cases: Sheet quota limits, connectivity issues  

  - *Revenue Analytics Logger*  
    - Type: Google Sheets  
    - Role: Logs revenue impact and related analytics to "Revenue_Analytics" sheet for performance tracking  
    - Logged fields: Date, hour, product ID, revenue impact, inventory, strategy, demand multiplier, optimization score, competitor average price, etc.  
    - Inputs: Pricing Optimization Engine output (via Price Change Filter)  
    - Edge Cases: Same as Pricing History Logger  

  - *Pricing Alert Sender*  
    - Type: Slack  
    - Role: Sends formatted alert messages to Slack channel "pricing-alerts" with detailed pricing change info and analysis  
    - Configuration: Uses message template with emojis and dynamic fields for clarity  
    - Inputs: Price Change Filter (true branch)  
    - Edge Cases: Slack API rate limits, channel misconfiguration  

  - *Pricing Report Sender*  
    - Type: Email Send  
    - Role: Sends summarized pricing update emails with subject referencing product name  
    - Inputs: Price Change Filter (true branch)  
    - Edge Cases: SMTP issues, email delivery failures  

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                        | Input Node(s)                     | Output Node(s)                             | Sticky Note                                                                                                                     |
|-----------------------------|--------------------------|-------------------------------------|----------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Hourly Pricing Monitor Trigger | Schedule Trigger         | Initiates workflow hourly             | None                             | Pricing Configuration Processor             | Triggers every hour for real-time price optimization                                                                           |
| Pricing Configuration Processor | Code Node               | Processes config, products, tasks    | Hourly Pricing Monitor Trigger   | Split Monitoring Tasks                      | Processes product catalog, competitor settings, and market context for pricing optimization                                     |
| Split Monitoring Tasks         | Split in Batches          | Splits monitoring tasks for parallel | Pricing Configuration Processor  | Task Processor                              | Breaks down monitoring into task batches for parallel processing                                                               |
| Task Processor                | Code Node                | Enhances tasks with market context   | Split Monitoring Tasks, Pricing Configuration Processor | AI Competitor Price Scraper, AI Demand Analysis Scraper, AI Customer Sentiment Scraper | Prepares individual monitoring tasks with enhanced context data for AI scrapers                                                |
| AI Competitor Price Scraper   | scrapegraphAi            | Scrapes competitor pricing data      | Task Processor                   | Merge Pricing Data                          | Multi-source AI scraper for competitor prices (Amazon, Best Buy, Walmart, Target)                                              |
| AI Demand Analysis Scraper    | scrapegraphAi            | Gathers market demand trends         | Task Processor                   | Merge Pricing Data                          | AI scraper analyzing Google Trends and demand indicators                                                                       |
| AI Customer Sentiment Scraper | scrapegraphAi            | Extracts customer sentiment          | Task Processor                   | Merge Pricing Data                          | Scrapes customer reviews and sentiment data                                                                                   |
| Merge Pricing Data            | Merge                    | Combines AI scraper outputs          | AI Competitor Price Scraper, AI Demand Analysis Scraper, AI Customer Sentiment Scraper | Pricing Optimization Engine                  | Combines competitor, demand, and sentiment data for pricing analysis                                                          |
| Pricing Optimization Engine   | Code Node                | Calculates optimal prices             | Merge Pricing Data               | Price Change Filter, Pricing History Logger, Revenue Analytics Logger | Advanced AI pricing optimization engine combining multi-factor analysis                                                        |
| Price Change Filter           | If Node                  | Filters significant/confident price changes | Pricing Optimization Engine      | Price Update API Call, Pricing Alert Sender, Pricing Report Sender | Filters price changes based on >2% change and >70% confidence                                                                   |
| Price Update API Call         | HTTP Request             | Applies price updates to e-commerce  | Price Change Filter (true branch) | Pricing History Logger, Revenue Analytics Logger, Pricing Alert Sender | Integrates with e-commerce platform API to update prices                                                                        |
| Pricing History Logger        | Google Sheets            | Logs pricing changes history         | Pricing Optimization Engine      | None                                       | Logs every price change with full context                                                                                     |
| Revenue Analytics Logger      | Google Sheets            | Logs revenue and performance metrics | Pricing Optimization Engine      | None                                       | Tracks revenue impact and pricing effectiveness                                                                                |
| Pricing Alert Sender          | Slack                    | Sends price change alerts            | Price Change Filter (true branch) | None                                       | Slack notifications with rich pricing and market analysis                                                                      |
| Pricing Report Sender         | Email Send               | Sends pricing update emails          | Price Change Filter (true branch) | None                                       | Email reports summarizing pricing changes                                                                                      |
| Workflow Overview             | Sticky Note              | Documentation overview                | None                             | None                                       | # 💰 AI-Powered Dynamic Pricing Optimizer ...                                                                                  |
| Step 1: Hourly Trigger Guide | Sticky Note              | Documentation for hourly trigger      | None                             | None                                       | # ⏱️ Step 1: Hourly Pricing Monitor Trigger ...                                                                                |
| Step 2: Configuration Processor | Sticky Note              | Documentation for config processor    | None                             | None                                       | # 🔧 Step 2: Pricing Configuration Processor ...                                                                                |
| Step 3: Task Splitting       | Sticky Note              | Documentation for task splitting      | None                             | None                                       | # 🔄 Step 3: Split Monitoring Tasks ...                                                                                         |
| Step 4: Task Processing      | Sticky Note              | Documentation for task processor      | None                             | None                                       | # 🎛️ Step 4: Task Processor ...                                                                                                |
| Steps 5-7: AI Scraping Network | Sticky Note              | Documentation for AI scrapers         | None                             | None                                       | # 🤖 Steps 5-7: Multi-Source AI Scraping Network ...                                                                            |
| Step 8: Data Merging         | Sticky Note              | Documentation for data merging        | None                             | None                                       | # 🔗 Step 8: Merge Pricing Data ...                                                                                             |
| Step 9: Optimization Engine  | Sticky Note              | Documentation for optimization engine | None                             | None                                       | # 🧠 Step 9: Pricing Optimization Engine ...                                                                                    |
| Step 10: Price Change Filter | Sticky Note              | Documentation for price change filter | None                             | None                                       | # 🎯 Step 10: Price Change Filter ...                                                                                           |
| Steps 11-15: Analytics & Communication | Sticky Note              | Documentation for analytics & communication | None                             | None                                       | # 📊 Steps 11-15: Analytics & Communication ...                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Name: "Hourly Pricing Monitor Trigger"  
   - Type: Schedule Trigger  
   - Settings: Interval every 1 hour  

2. **Add Code Node for Configuration Processor:**  
   - Name: "Pricing Configuration Processor"  
   - Paste JavaScript defining product catalog, competitors, strategies, demand factors, and generating monitoring tasks  
   - Connect from "Hourly Pricing Monitor Trigger"  

3. **Add Split in Batches Node:**  
   - Name: "Split Monitoring Tasks"  
   - Connect from "Pricing Configuration Processor"  
   - Leave batch size default or configure as needed  

4. **Add Code Node for Task Processor:**  
   - Name: "Task Processor"  
   - JavaScript to enhance each batch task with market context and config data  
   - Connect from "Split Monitoring Tasks"  

5. **Add Three Parallel ScrapeGraph AI Nodes:**  
   - Names: "AI Competitor Price Scraper", "AI Demand Analysis Scraper", "AI Customer Sentiment Scraper"  
   - Configure each with appropriate userPrompt JSON schemas and dynamic websiteUrl expressions from task data  
   - Connect all three nodes from "Task Processor" in parallel  

6. **Add Merge Node:**  
   - Name: "Merge Pricing Data"  
   - Mode: Combine  
   - Connect each AI scraper output to this merge node  

7. **Add Code Node for Pricing Optimization Engine:**  
   - Name: "Pricing Optimization Engine"  
   - Paste JavaScript implementing multi-factor pricing algorithm and revenue projections  
   - Connect from "Merge Pricing Data"  

8. **Add If Node for Price Change Filtering:**  
   - Name: "Price Change Filter"  
   - Condition:  
     - Absolute value of price change percentage > 2%  
     - Confidence level ≥ 0.7  
   - Connect from "Pricing Optimization Engine"  

9. **Add HTTP Request Node for Price Update API:**  
   - Name: "Price Update API Call"  
   - Method: POST or PATCH depending on API  
   - URL: `https://your-ecommerce-api.com/products/{{ $json.product_info.id }}/price`  
   - Headers: Authorization Bearer token, Content-Type application/json  
   - Body: JSON with new price, previous price, reason, confidence, effective date  
   - Connect from "Price Change Filter" (true branch)  

10. **Add Google Sheets Node for Pricing History Logging:**  
    - Name: "Pricing History Logger"  
    - Operation: Append  
    - Spreadsheet ID and sheet name "Pricing_History"  
    - Map relevant fields (strategy, prices, confidence, revenue impact, etc.)  
    - Connect from "Pricing Optimization Engine" (true branch of Price Change Filter)  

11. **Add Google Sheets Node for Revenue Analytics Logging:**  
    - Name: "Revenue Analytics Logger"  
    - Operation: Append  
    - Spreadsheet ID and sheet name "Revenue_Analytics"  
    - Map relevant revenue and optimization metrics  
    - Connect from "Pricing Optimization Engine" (true branch of Price Change Filter)  

12. **Add Slack Node for Alerts:**  
    - Name: "Pricing Alert Sender"  
    - Channel: "pricing-alerts"  
    - Message: Formatted with product, price change, strategy, confidence, market analysis, revenue impact  
    - Connect from "Price Change Filter" (true branch)  

13. **Add Email Send Node for Reports:**  
    - Name: "Pricing Report Sender"  
    - Subject: "Dynamic Pricing Update - {{ $json.product_info.name }}"  
    - Connect from "Price Change Filter" (true branch)  

14. **Add Sticky Notes for Documentation:**  
    - Names and contents as per workflow documentation nodes for overview and step guides  

15. **Set Credentials:**  
    - Configure API token for your e-commerce platform in HTTP Request node  
    - Configure Google credentials in Google Sheets nodes  
    - Configure Slack credentials for Slack node  
    - Configure SMTP or email credentials for Email Send node  

16. **Test Workflow:**  
    - Run manually or wait for the scheduled trigger  
    - Monitor logs and outputs for correctness  
    - Adjust batch sizes, thresholds, and API parameters as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow created July 25, 2025, for AI-powered dynamic pricing automation with competitor monitoring      | Workflow overview sticky note                                                                        |
| Hourly trigger frequency balances real-time responsiveness with API rate limits and cost considerations  | Step 1 sticky note                                                                                   |
| Product catalog includes detailed pricing, margin, inventory, and competitor keyword mappings            | Step 2 sticky note                                                                                   |
| Task splitting enables scalable, parallel processing for hundreds of product-competitor-keyword combos   | Step 3 sticky note                                                                                   |
| AI scraping nodes use ScrapeGraph AI with custom prompts for structured data extraction                   | Step 5-7 sticky note                                                                                 |
| Pricing optimization uses weighted multi-factor scoring with constraints to avoid market shocks          | Step 9 sticky note                                                                                   |
| Price change filter enforces significance and confidence thresholds for safe automated updates           | Step 10 sticky note                                                                                  |
| Analytics and communication include Google Sheets logging, Slack alerts, and email reports for transparency | Steps 11-15 sticky note                                                                              |
| Replace placeholder API tokens and spreadsheet IDs with real credentials before production deployment     | Critical for API connectivity and logging integrity                                                 |
| Consider API quotas, error handling, and fallback mechanisms for scraper and API failures                  | General best practice for robust production workflows                                               |
| Documentation sticky notes provide detailed explanations for each step for maintainability and onboarding | Useful for new users and team members managing this workflow                                        |

---

**Disclaimer:** The text above is derived exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.