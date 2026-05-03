Track E-commerce Price Changes with ScrapeGraphAI, Baserow & Pushover Alerts

https://n8nworkflows.xyz/workflows/track-e-commerce-price-changes-with-scrapegraphai--baserow---pushover-alerts-11803


# Track E-commerce Price Changes with ScrapeGraphAI, Baserow & Pushover Alerts

### 1. Workflow Overview

This workflow tracks e-commerce product price changes by scraping live data, enriching it with historical pricing, storing results, and sending notifications for significant price drops or errors. It targets online retailers’ product pages, enabling users to monitor price trends and receive alerts on mobile devices.

Logical blocks:

- **1.1 Trigger & Configuration**: Receives webhook calls and loads configured product list and thresholds.
- **1.2 Scraping**: Iterates over products, scrapes live price and availability using AI-driven ScrapeGraphAI.
- **1.3 Enrichment**: Fetches historical pricing data from an external API and combines it with scraped data.
- **1.4 Storage & Validation**: Cleans and enriches data, calculates price drop percentages, timestamps, and stores records in a PostgreSQL table.
- **1.5 Alerts & Error Handling**: Sends notifications via Pushover for significant price drops or scraping errors with different priorities.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration

**Overview:**  
This block initiates the workflow via a webhook and centralizes product data and thresholds in one coded list, making management straightforward and stateless.

**Nodes Involved:**  
- Price Monitor Webhook  
- Product Configuration  
- Loop Products  

**Node Details:**

- **Price Monitor Webhook**  
  - Type: Webhook Trigger  
  - Role: Exposes public POST endpoint (`/product-price-monitor`) to trigger the workflow.  
  - Config: HTTP POST, no special options.  
  - Inputs: External HTTP calls  
  - Outputs: Triggers Product Configuration node  
  - Edge cases: Missing or malformed POST requests; webhook downtime or URL changes.

- **Product Configuration**  
  - Type: Code (JavaScript)  
  - Role: Holds static product list with URLs and threshold percentages.  
  - Config: Returns array of JSON objects, each representing a product SKU with name, URL, and threshold.  
  - Key code:  
    ```js
    const products = [
      { name: 'Winter Jacket', url: 'https://example.com/winter-jacket', thresholdPercentage: 10 },
      { name: 'Snow Boots', url: 'https://example.com/snow-boots', thresholdPercentage: 10 }
    ];
    return products.map(p => ({ json: p }));
    ```  
  - Inputs: Trigger from webhook  
  - Outputs: List of products to Loop Products node  
  - Edge cases: Empty product list, invalid URLs, or missing threshold.

- **Loop Products**  
  - Type: Split In Batches  
  - Role: Iterates over each product individually to avoid rate limits and enable scaling.  
  - Config: Default batch size (usually 1)  
  - Inputs: Product Configuration output array  
  - Outputs: Single product JSON per loop iteration to Scrape Product Data  
  - Edge cases: Large product lists causing long execution times; ensure batch size properly configured.

---

#### 1.2 Scraping

**Overview:**  
Scrapes each product page using ScrapeGraphAI’s natural language prompt to reliably extract price, name, currency, and availability, avoiding brittle CSS selectors.

**Nodes Involved:**  
- Scrape Product Data  
- Valid Price Check  

**Node Details:**

- **Scrape Product Data**  
  - Type: ScrapeGraphAI node  
  - Role: Uses AI prompt to extract specific product data fields from product URL.  
  - Config:  
    - User prompt: `"Extract the product title as \"name\", numeric price as \"price\", currency as \"currency\", and availability text as \"availability\". Respond as JSON."`  
    - Website URL dynamically set from current product JSON (`={{ $json.url }}`).  
  - Inputs: Single product JSON from Loop Products  
  - Outputs: Scraped data JSON to Valid Price Check  
  - Edge cases: Page load failures, AI misinterpretation, missing or malformed price data.

- **Valid Price Check**  
  - Type: If (Conditional)  
  - Role: Checks that scraped price is a valid number > 0 before continuing.  
  - Config: Condition: `$json.price > 0`  
  - Inputs: Scraped data JSON  
  - Outputs:  
    - True: Fetch Historical Data (enrichment)  
    - False: Prepare Error Message (error handling)  
  - Edge cases: Zero or missing price, parsing errors from scrape.

---

#### 1.3 Enrichment

**Overview:**  
Fetches 90-day average price data for each product from an external API endpoint and merges it with live scrape data for richer analysis.

**Nodes Involved:**  
- Fetch Historical Data  
- Combine Scrape & History  

**Node Details:**

- **Fetch Historical Data**  
  - Type: HTTP Request  
  - Role: Calls external API with product name query parameter to get historical average price.  
  - Config:  
    - URL: `'https://api.pricingexample.com/history?name=' + encodeURIComponent($json.name)`  
    - Method: GET (default)  
  - Inputs: Valid Price Check (true branch) output JSON  
  - Outputs: Historical price data JSON to Combine Scrape & History (second input)  
  - Edge cases: HTTP errors, API downtime, API rate limits, malformed response.

- **Combine Scrape & History**  
  - Type: Merge  
  - Role: Joins live scrape data and historical data arrays by index to create a unified JSON object.  
  - Config: Mode: Merge by Index  
  - Inputs:  
    - First: From Fetch Historical Data (historical data)  
    - Second: From Valid Price Check (scraped data)  
  - Outputs: Merged JSON to Transform & Enrich Data  
  - Edge cases: Mismatched array lengths, missing data from one source.

---

#### 1.4 Storage & Validation

**Overview:**  
Standardizes and enriches data fields, calculates price change percentages, timestamps records, and persists every record into a PostgreSQL table for auditing and reporting.

**Nodes Involved:**  
- Transform & Enrich Data  
- Insert rows in a table  

**Node Details:**

- **Transform & Enrich Data**  
  - Type: Code  
  - Role: Converts strings to numbers, calculates price drop %, adds timestamp and threshold, standardizes field names.  
  - Key logic:  
    ```js
    const items = $input.all();
    return items.map(item => {
      const current = Number(item.json.price);
      const average = Number(item.json.averagePrice || item.json.average_price || 0);
      const drop = average > 0 ? ((average - current) / average) * 100 : 0;
      return {
        json: {
          ...item.json,
          averagePrice: average,
          changePercent: Number(drop.toFixed(2)),
          timestamp: new Date().toISOString(),
          thresholdPercentage: item.json.thresholdPercentage || 10
        }
      };
    });
    ```  
  - Inputs: Combined scrape and history JSON  
  - Outputs: Enriched JSON to Insert rows in a table  
  - Edge cases: Non-numeric prices, missing averagePrice, date formatting issues.

- **Insert rows in a table**  
  - Type: PostgreSQL node  
  - Role: Inserts enriched records into a configured PostgreSQL table for storage.  
  - Config:  
    - Database schema: `public`  
    - Table: unspecified (must be configured by user)  
  - Inputs: Enriched JSON  
  - Outputs: Success confirmation to Price Drop? node  
  - Edge cases: DB connectivity, schema mismatch, duplicate or invalid data.

---

#### 1.5 Alerts & Error Handling

**Overview:**  
Sends two types of Pushover notifications: price drop alerts for items exceeding threshold, and high-priority error alerts for scraping failures.

**Nodes Involved:**  
- Price Drop?  
- Prepare Alert Message  
- Send Pushover Alert  
- Prepare Error Message  
- Send Error Alert  

**Node Details:**

- **Price Drop?**  
  - Type: If  
  - Role: Checks if calculated price drop % exceeds threshold percentage.  
  - Config: Condition: `$json.changePercent > $json.thresholdPercentage`  
  - Inputs: Insert rows in a table output  
  - Outputs:  
    - True: Prepare Alert Message  
    - False: Ends here (no alert)  
  - Edge cases: Threshold missing or zero, floating point precision.

- **Prepare Alert Message**  
  - Type: Set  
  - Role: Formats the alert message string for Pushover notification on price drop.  
  - Config: Creates a message string in `$json.message` (content details not shown in JSON, assumed formatted)  
  - Inputs: Price Drop? (true branch)  
  - Outputs: Send Pushover Alert  
  - Edge cases: Message formatting errors, missing fields.

- **Send Pushover Alert**  
  - Type: Pushover  
  - Role: Sends normal priority notification to user’s device with alert message.  
  - Config:  
    - Message: from previous node’s `$json.message`  
    - Priority: 0 (normal)  
    - Credentials: Pushover user and app tokens (configured externally)  
  - Inputs: Alert message JSON  
  - Outputs: End of alert path  
  - Edge cases: Pushover API errors, invalid tokens, network issues.

- **Prepare Error Message**  
  - Type: Set  
  - Role: Formats error messages for failed scraping or invalid price data.  
  - Config: Sets `$json.message` for error alert (content not detailed)  
  - Inputs: Valid Price Check (false branch)  
  - Outputs: Send Error Alert  
  - Edge cases: Missing error details, incomplete data.

- **Send Error Alert**  
  - Type: Pushover  
  - Role: Sends high priority notification for errors in scraping or validation.  
  - Config:  
    - Message: from `$json.message`  
    - Priority: 1 (high)  
    - Credentials: Same as normal alerts  
  - Inputs: Prepared error message  
  - Outputs: End of error path  
  - Edge cases: Same as Send Pushover Alert.

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role                     | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                         |
|-----------------------|--------------------------|-----------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| Workflow Overview     | Sticky Note              | Overview of entire workflow       |                           |                            | ## How it works… (full workflow explanation and setup steps)                                      |
| Section – Trigger & Config | Sticky Note           | Grouping for trigger & config     |                           |                            | ## Trigger & Configuration… (explains webhook and product config)                                 |
| Section – Scraping    | Sticky Note              | Grouping for scraping logic       |                           |                            | ## Scraping… (explains AI scraping and batch looping)                                             |
| Section – Enrichment  | Sticky Note              | Grouping for enrichment logic     |                           |                            | ## Enrichment… (explains historical data enrichment)                                              |
| Section – Storage & Validation | Sticky Note        | Grouping for data storage & validation |                       |                            | ## Storage & Validation… (explains data transformation and storage in Baserow/Postgres)           |
| Section – Notifications | Sticky Note             | Grouping for alerts & error handling |                         |                            | ## Alerts & Error Handling… (explains Pushover notifications for drops and errors)                |
| Price Monitor Webhook | Webhook                  | Entry point HTTP trigger          |                           | Product Configuration       |                                                                                                   |
| Product Configuration | Code                     | Defines products and thresholds   | Price Monitor Webhook      | Loop Products              |                                                                                                   |
| Loop Products         | Split In Batches         | Iterates products one by one      | Product Configuration      | Scrape Product Data         |                                                                                                   |
| Scrape Product Data   | ScrapeGraphAI            | AI-driven product data scrape     | Loop Products              | Valid Price Check           |                                                                                                   |
| Valid Price Check     | If                       | Validates price > 0               | Scrape Product Data        | Fetch Historical Data / Prepare Error Message |                                                                                   |
| Fetch Historical Data | HTTP Request             | Gets 90-day average price         | Valid Price Check          | Combine Scrape & History    |                                                                                                   |
| Combine Scrape & History | Merge                  | Combines scrape & historical data | Fetch Historical Data / Valid Price Check | Transform & Enrich Data |                                                                                                   |
| Transform & Enrich Data | Code                    | Calculates drop %, timestamps data | Combine Scrape & History  | Insert rows in a table      |                                                                                                   |
| Insert rows in a table | PostgreSQL               | Stores data records               | Transform & Enrich Data    | Price Drop?                |                                                                                                   |
| Price Drop?           | If                       | Checks if price drop > threshold  | Insert rows in a table     | Prepare Alert Message       |                                                                                                   |
| Prepare Alert Message | Set                      | Formats alert message             | Price Drop?                | Send Pushover Alert         |                                                                                                   |
| Send Pushover Alert   | Pushover                 | Sends price drop notification     | Prepare Alert Message      |                            |                                                                                                   |
| Prepare Error Message | Set                      | Formats error alert message       | Valid Price Check          | Send Error Alert            |                                                                                                   |
| Send Error Alert      | Pushover                 | Sends high priority error alert   | Prepare Error Message      |                            |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Price Monitor Webhook`  
   - HTTP Method: POST  
   - Path: `product-price-monitor`  

2. **Create Code Node for Product Configuration**  
   - Name: `Product Configuration`  
   - Code (JavaScript):  
     ```js
     const products = [
       { name: 'Winter Jacket', url: 'https://example.com/winter-jacket', thresholdPercentage: 10 },
       { name: 'Snow Boots', url: 'https://example.com/snow-boots', thresholdPercentage: 10 }
     ];
     return products.map(p => ({ json: p }));
     ```  
   - Connect Webhook output to this node input.

3. **Create Split In Batches Node**  
   - Name: `Loop Products`  
   - Connect Product Configuration output to this node input.  
   - Default batch size (1) to process one product at a time.

4. **Create ScrapeGraphAI Node**  
   - Name: `Scrape Product Data`  
   - Configure:  
     - User Prompt:  
       ```
       Extract the product title as "name", numeric price as "price", currency as "currency", and availability text as "availability". Respond as JSON.
       ```  
     - Website URL: Set expression to `{{$json.url}}` from input item.  
   - Connect Loop Products output to this node.

5. **Create If Node for Price Validation**  
   - Name: `Valid Price Check`  
   - Condition: Number `price > 0`  
   - Connect Scrape Product Data output to this node.

6. **Create HTTP Request Node for Historical Data**  
   - Name: `Fetch Historical Data`  
   - Method: GET  
   - URL: Expression:  
     ```
     https://api.pricingexample.com/history?name={{ encodeURIComponent($json.name) }}
     ```  
   - Connect Valid Price Check (true) output to this node.

7. **Create Merge Node**  
   - Name: `Combine Scrape & History`  
   - Mode: Merge by Index  
   - Connect Valid Price Check (true) node output to second input.  
   - Connect Fetch Historical Data output to first input.

8. **Create Code Node for Data Transformation**  
   - Name: `Transform & Enrich Data`  
   - Code:  
     ```js
     const items = $input.all();
     return items.map(item => {
       const current = Number(item.json.price);
       const average = Number(item.json.averagePrice || item.json.average_price || 0);
       const drop = average > 0 ? ((average - current) / average) * 100 : 0;
       return {
         json: {
           ...item.json,
           averagePrice: average,
           changePercent: Number(drop.toFixed(2)),
           timestamp: new Date().toISOString(),
           thresholdPercentage: item.json.thresholdPercentage || 10
         }
       };
     });
     ```  
   - Connect Merge output to this node.

9. **Create PostgreSQL Node for Storage**  
   - Name: `Insert rows in a table`  
   - Configure PostgreSQL credentials.  
   - Select schema: `public`  
   - Select target table (must exist and match data fields).  
   - Connect Transform & Enrich Data output to this node.

10. **Create If Node for Price Drop Check**  
    - Name: `Price Drop?`  
    - Condition: Number `changePercent > thresholdPercentage`  
    - Connect PostgreSQL node output to this node.

11. **Create Set Node for Alert Message**  
    - Name: `Prepare Alert Message`  
    - Configure to set `$json.message` with a concise alert string using fields such as product name, current price, change percentage, etc.  
    - Connect Price Drop? (true) output here.

12. **Create Pushover Node for Price Drop Alerts**  
    - Name: `Send Pushover Alert`  
    - Configure Pushover credentials (user key and app token).  
    - Message: Set to expression `{{$json.message}}`  
    - Priority: 0 (normal)  
    - Connect Prepare Alert Message output here.

13. **Create Set Node for Error Messages**  
    - Name: `Prepare Error Message`  
    - Configure to set `$json.message` describing the error (e.g., invalid or missing price).  
    - Connect Valid Price Check (false) output here.

14. **Create Pushover Node for Error Alerts**  
    - Name: `Send Error Alert`  
    - Configure Pushover credentials (same as above).  
    - Message: Expression `{{$json.message}}`  
    - Priority: 1 (high)  
    - Connect Prepare Error Message output here.

15. **Connect all nodes as per logical flow:**  
    - Webhook → Product Configuration → Loop Products → Scrape Product Data → Valid Price Check  
    - Valid Price Check (true) → Fetch Historical Data → Combine Scrape & History → Transform & Enrich Data → Insert rows in a table → Price Drop?  
    - Price Drop? (true) → Prepare Alert Message → Send Pushover Alert  
    - Valid Price Check (false) → Prepare Error Message → Send Error Alert  

16. **Test workflow:**  
    - Trigger webhook with POST (empty or any payload).  
    - Verify logs, data stored in PostgreSQL, and notifications received.

17. **Optional:**  
    - Modify Product Configuration node for your own products and thresholds.  
    - Change HTTP Request URL in Fetch Historical Data to your preferred API.  
    - Adjust PostgreSQL table and schema as needed.  
    - Set Pushover credentials via n8n Credentials manager.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses ScrapeGraphAI’s natural language prompt instead of CSS selectors for robust scraping across site changes.                                                  | See node "Scrape Product Data" description                                                     |
| Centralized product configuration simplifies maintenance and scaling to many SKUs without changing multiple places.                                                      | Section – Trigger & Configuration sticky note                                                  |
| Storing every scrape in PostgreSQL ensures auditable history and aids in debugging and trend analysis.                                                                     | Section – Storage & Validation sticky note                                                    |
| Pushover sends two levels of notifications: normal priority for price drops and high priority for errors, reducing alert fatigue.                                        | Section – Notifications sticky note                                                           |
| Setup requires API credentials for ScrapeGraphAI, Baserow or PostgreSQL, and Pushover.                                                                                     | See Workflow Overview and Setup steps in sticky note                                          |
| Pushover documentation and API details: https://pushover.net/api                                                                                                         | For credential setup and troubleshooting                                                     |
| ScrapeGraphAI details can be found at https://scrapegraph.com/                                                                                                           | For advanced prompt customization                                                             |
| This workflow is designed to be stateless and scalable, processing one product per batch to comply with rate limits and avoid anti-bot blocks.                            | Section – Scraping sticky note                                                                |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.