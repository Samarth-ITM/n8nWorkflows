E-commerce Price Tracker with ScrapeGraphAI, MongoDB, and Mailgun Alerts

https://n8nworkflows.xyz/workflows/e-commerce-price-tracker-with-scrapegraphai--mongodb--and-mailgun-alerts-11585


# E-commerce Price Tracker with ScrapeGraphAI, MongoDB, and Mailgun Alerts

### 1. Workflow Overview

This workflow is designed for automated monitoring of product prices from multiple e-commerce URLs. It accepts a list of product URLs via a webhook, performs parallel scraping of product data using ScrapeGraphAI, analyzes price changes against defined alert thresholds, stores all observations in MongoDB, and sends email alerts via Mailgun if significant price drops are detected.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception and URL Preparation:** Captures incoming POST requests containing product URLs and prepares them for processing.
- **1.2 Parallel Scraping:** Splits the URL list into individual items and scrapes each product page in parallel using ScrapeGraphAI.
- **1.3 Aggregation and Price Analysis:** Aggregates scraped data and calculates price change percentages relative to alert thresholds.
- **1.4 Storage and Alerting:** Stores observations in MongoDB and sends email alerts via Mailgun if significant price drops occur, then responds to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and URL Preparation

- **Overview:**  
  This block handles the initial reception of product URLs via a webhook POST request. It parses the payload or falls back to a predefined list of URLs with alert thresholds. The URLs are transformed into individual items for downstream processing.

- **Nodes Involved:**  
  - Incoming Monitor Request (Webhook)  
  - Define Product Sources (Code)

- **Node Details:**

  - **Incoming Monitor Request**  
    - Type: Webhook Trigger  
    - Role: Receives incoming POST requests at `/product-price-monitor` endpoint.  
    - Configuration: Sets HTTP method to POST and response mode to wait for explicit response node.  
    - Input/Output: No input; outputs raw JSON body to next node.  
    - Edge Cases: Malformed JSON in POST body or missing URLs array; handled downstream.  
    - Version: 1

  - **Define Product Sources**  
    - Type: Code (JavaScript)  
    - Role: Parses webhook payload to extract product URLs and alert thresholds, or uses fallback default array if none provided.  
    - Key Logic: Checks if `urls` array exists and is non-empty; otherwise, returns hard-coded URLs with alert percentages. Converts each URL object into an individual item.  
    - Input/Output: Input is webhook JSON, output is array of single-item JSON objects with `url` and `alertPercent` keys.  
    - Edge Cases: Empty or invalid payload handled by fallback.  
    - Version: 2

#### 2.2 Parallel Scraping

- **Overview:**  
  This block splits the array of product URLs into single-item batches to enable parallel scraping. Each product URL is sent to ScrapeGraphAI, which extracts key product details using a natural language prompt.

- **Nodes Involved:**  
  - Split URLs (SplitInBatches)  
  - Scrape Product Page (ScrapeGraphAI)

- **Node Details:**

  - **Split URLs**  
    - Type: SplitInBatches  
    - Role: Splits input array into batches of size one to enable parallel processing of each URL.  
    - Configuration: Batch size implicitly set to 1 (default) for maximum parallelism.  
    - Input/Output: Input is array of product items; outputs single-item batches to scraper.  
    - Edge Cases: Large input arrays may increase parallel load; rate limits on ScrapeGraphAI may apply.  
    - Version: 3

  - **Scrape Product Page**  
    - Type: ScrapeGraphAI node  
    - Role: Scrapes product page content using an LLM-based AI prompt, extracting product name, current price (number), currency code, and availability flag.  
    - Configuration:  
      - `userPrompt`: Fixed natural-language prompt requesting JSON output with keys: name, price, currency, availability.  
      - `websiteUrl`: Expression referencing current item's `url` property.  
    - Input/Output: Input is single product URL; output is scraped product data JSON.  
    - Edge Cases: Network failures, unexpected page layouts, or AI interpretation errors may cause extraction failures or incomplete data.  
    - Version: 1

#### 2.3 Aggregation and Price Analysis

- **Overview:**  
  Aggregates all individual scraped results into a single array to ensure all scraping completes before analysis. Then, calculates the percentage price movement relative to alert thresholds and flags significant price drops.

- **Nodes Involved:**  
  - Combine Scraped Data (Merge)  
  - Analyze Price Movement (Code)

- **Node Details:**

  - **Combine Scraped Data**  
    - Type: Merge (Aggregate mode)  
    - Role: Aggregates all scraped product data items into a single array, ensuring downstream nodes wait for all scraping to finish.  
    - Configuration: Mode set to aggregate.  
    - Input/Output: Inputs multiple single scraped items; outputs one aggregated array.  
    - Edge Cases: Missing data from some scraper nodes could cause incomplete aggregation.  
    - Version: 2

  - **Analyze Price Movement**  
    - Type: Code (JavaScript)  
    - Role: Processes each product's scraped data, converts price strings to numbers, calculates percentage price drop, compares against alert threshold, and flags significant drops.  
    - Key Logic:  
      - Uses `alertPercent` field or defaults to 10%.  
      - Dummy `previousPrice` equal to current price (placeholder for real historical querying).  
      - Computes `diffPercent = ((previousPrice - price) / previousPrice) * 100`.  
      - Flags `significantDrop` as true if `diffPercent >= alertPercent`.  
      - Appends fields: `previousPrice`, `diffPercent` (rounded), `significantDrop`.  
    - Input/Output: Input is aggregated scraped data array; output is array with analysis fields added.  
    - Edge Cases: No real historical price lookup implemented here; all drops flagged relative to dummy previous price (equal to current price), so no alerts triggered until real data integration added.  
    - Version: 2

#### 2.4 Storage and Alerting

- **Overview:**  
  Stores all price observations in MongoDB for historical tracking. Checks for any significant price drops and, if found, prepares and sends an alert email via Mailgun. Finally, sends a JSON response back to the webhook caller.

- **Nodes Involved:**  
  - Store to MongoDB (MongoDB)  
  - Significant Price Change? (If)  
  - Prepare Alert Email (Set)  
  - Send Mailgun Alert (Mailgun)  
  - Send Webhook Response (Respond to Webhook)

- **Node Details:**

  - **Store to MongoDB**  
    - Type: MongoDB node  
    - Role: Inserts a timestamped document for each product price observation into the `product_prices` collection.  
    - Configuration:  
      - Operation: Insert  
      - Fields: url, name, price, currency, timestamp (current ISO string), diffPercent, availability  
    - Input/Output: Input is array with analysis fields; outputs inserted documents.  
    - Edge Cases: MongoDB connectivity issues, schema mismatches, or network latency could cause failures.  
    - Version: 1

  - **Significant Price Change?**  
    - Type: If node  
    - Role: Checks if any product item has `significantDrop` flagged true.  
    - Configuration: Boolean condition testing `$json.significantDrop === true`.  
    - Input/Output: Branches workflow to alert preparation or skips if no significant changes.  
    - Edge Cases: If no items flagged, alert branch not executed.  
    - Version: 2

  - **Prepare Alert Email**  
    - Type: Set node  
    - Role: Constructs email content (subject, recipients, HTML body) summarizing products with significant price drops.  
    - Configuration: Custom fields set here (not detailed in JSON), expected to include variables like product names, old/new prices, % savings in HTML format.  
    - Input/Output: Input is filtered items with significant drops; output prepares fields for Mailgun.  
    - Edge Cases: Misconfiguration could lead to empty or malformed email content.  
    - Version: 3.4

  - **Send Mailgun Alert**  
    - Type: Mailgun node  
    - Role: Sends the alert email to defined recipients from a verified sender address.  
    - Configuration:  
      - Subject, To, From email addresses set via expressions or literals.  
      - Requires Mailgun credentials with verified sending domain.  
    - Input/Output: Input is email content from previous node; output is Mailgun API response.  
    - Edge Cases: Authentication failures, quota limits, or invalid recipient addresses may cause errors.  
    - Version: 1

  - **Send Webhook Response**  
    - Type: Respond to Webhook  
    - Role: Sends an immediate JSON confirmation back to the external caller regardless of alert status.  
    - Configuration: Default response options.  
    - Input/Output: Receives flow completion signal; outputs HTTP response.  
    - Edge Cases: If this node fails, caller may hang or receive no confirmation.  
    - Version: 1

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                     | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                                                      |
|-------------------------|------------------------|-----------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview       | StickyNote             | Describes overall workflow purpose| -                      | -                         | Explains workflow logic, setup steps, and how URLs trigger scraping to alerts.                                                  |
| Section – Trigger & URL Setup | StickyNote        | Describes initial input reception | -                      | -                         | Explains webhook trigger and product URL array preparation with fallback.                                                       |
| Section – Parallel Scraping | StickyNote           | Explains URL splitting and scraping| -                      | -                         | Details batch splitting and ScrapeGraphAI usage for robust multi-site scraping.                                                 |
| Section – Aggregation & Analysis | StickyNote       | Describes aggregation and price analysis | -                      | -                         | Covers data merging and price movement calculations with flags for significant drops.                                          |
| Section – Storage & Alerts | StickyNote            | Describes storage and alerting    | -                      | -                         | Explains MongoDB storage, alert condition checking, email preparation, and sending alerts, plus webhook response.               |
| Incoming Monitor Request | Webhook                | Receives POST requests with URLs  | -                      | Define Product Sources     |                                                                                                                                |
| Define Product Sources   | Code                   | Parses input or uses fallback URLs| Incoming Monitor Request| Split URLs                 |                                                                                                                                |
| Split URLs              | SplitInBatches         | Splits URL array for parallel scraping| Define Product Sources | Scrape Product Page        |                                                                                                                                |
| Scrape Product Page      | ScrapeGraphAI          | Scrapes product info via AI prompt| Split URLs             | Combine Scraped Data       |                                                                                                                                |
| Combine Scraped Data     | Merge                  | Aggregates individual scraped data| Scrape Product Page     | Analyze Price Movement     |                                                                                                                                |
| Analyze Price Movement   | Code                   | Calculates price change and flags | Combine Scraped Data    | Store to MongoDB, Significant Price Change? |                                                                                                                                |
| Store to MongoDB         | MongoDB                | Stores observations with timestamp| Analyze Price Movement  | Send Webhook Response      |                                                                                                                                |
| Significant Price Change?| If                     | Checks for any significant drop   | Analyze Price Movement  | Prepare Alert Email        |                                                                                                                                |
| Prepare Alert Email      | Set                    | Constructs alert email content    | Significant Price Change? | Send Mailgun Alert         |                                                                                                                                |
| Send Mailgun Alert       | Mailgun                | Sends alert email                 | Prepare Alert Email     | Send Webhook Response      |                                                                                                                                |
| Send Webhook Response    | Respond to Webhook     | Sends HTTP response to caller     | Store to MongoDB, Send Mailgun Alert | -                   |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Name: Incoming Monitor Request  
   - HTTP Method: POST  
   - Path: `product-price-monitor`  
   - Response Mode: Response Node (waits for explicit response)  

2. **Create Code Node for URL Preparation**  
   - Type: Code  
   - Name: Define Product Sources  
   - Code:  
     ```javascript
     const body = $json;
     let urls = body.urls;
     if (!Array.isArray(urls) || urls.length === 0) {
       urls = [
         { url: 'https://example-ecom.com/productA', alertPercent: 10 },
         { url: 'https://example-ecom.com/productB', alertPercent: 12 }
       ];
     }
     return urls.map(obj => ({ json: obj }));
     ```  
   - Connect Incoming Monitor Request → Define Product Sources

3. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Name: Split URLs  
   - Batch Size: 1 (default)  
   - Connect Define Product Sources → Split URLs

4. **Create ScrapeGraphAI Node**  
   - Type: ScrapeGraphAI (requires ScrapeGraphAI credentials setup)  
   - Name: Scrape Product Page  
   - Parameters:  
     - User Prompt:  
       ```
       Extract the product name, current price (number), currency code, and availability (boolean). Respond strictly as JSON with keys: name, price, currency, availability.
       ```  
     - Website URL: Expression `={{ $json.url }}`  
   - Connect Split URLs → Scrape Product Page

5. **Create Merge Node**  
   - Type: Merge  
   - Name: Combine Scraped Data  
   - Mode: Aggregate  
   - Connect Scrape Product Page → Combine Scraped Data

6. **Create Code Node for Price Analysis**  
   - Type: Code  
   - Name: Analyze Price Movement  
   - Code:  
     ```javascript
     const THRESHOLD_KEY = 'alertPercent';
     return $input.all().map(item => {
       const data = item.json;
       const price = Number(data.price);
       const threshold = data[THRESHOLD_KEY] || 10;
       // Dummy previous price; in real setup, query last price from DB here
       const previousPrice = data.previousPrice || price;
       const diffPercent = previousPrice > 0 ? ((previousPrice - price) / previousPrice) * 100 : 0;
       const significantDrop = diffPercent >= threshold;
       return {
         json: {
           ...data,
           previousPrice,
           diffPercent: Number(diffPercent.toFixed(2)),
           significantDrop
         }
       };
     });
     ```  
   - Connect Combine Scraped Data → Analyze Price Movement

7. **Create MongoDB Node**  
   - Type: MongoDB (requires MongoDB credentials setup)  
   - Name: Store to MongoDB  
   - Operation: Insert  
   - Collection: `product_prices`  
   - Fields to insert (use expressions):  
     - url: `={{ $json.url }}`  
     - name: `={{ $json.name }}`  
     - price: `={{ $json.price }}`  
     - currency: `={{ $json.currency }}`  
     - timestamp: `={{ new Date().toISOString() }}`  
     - diffPercent: `={{ $json.diffPercent }}`  
     - availability: `={{ $json.availability }}`  
   - Connect Analyze Price Movement → Store to MongoDB

8. **Create If Node to Check Alerts**  
   - Type: If  
   - Name: Significant Price Change?  
   - Condition: Boolean expression where `$json.significantDrop === true`  
   - Connect Analyze Price Movement → Significant Price Change?

9. **Create Set Node to Prepare Email Content**  
   - Type: Set  
   - Name: Prepare Alert Email  
   - Configure to assemble email subject, recipients, and HTML body summarizing significant price drops (customize as needed).  
   - Connect Significant Price Change? (true branch) → Prepare Alert Email

10. **Create Mailgun Node to Send Email**  
    - Type: Mailgun (requires Mailgun credentials setup)  
    - Name: Send Mailgun Alert  
    - From Email: e.g., `pricebot@yourdomain.com` (verified in Mailgun)  
    - To Email: Expression or static list (configure recipients)  
    - Subject: Expression from previous node (e.g., `={{ $json.subject }}`)  
    - Connect Prepare Alert Email → Send Mailgun Alert

11. **Create Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Name: Send Webhook Response  
    - Configure default response options (e.g., JSON confirmation)  
    - Connect Store to MongoDB → Send Webhook Response  
    - Connect Send Mailgun Alert → Send Webhook Response  

12. **Activate the workflow** and test by sending POST requests with JSON body containing `urls` array.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow uses ScrapeGraphAI for resilient scraping via AI prompt instead of CSS selectors.| ScrapeGraphAI official documentation recommended for credential setup and usage details.                    |
| To send emails, Mailgun credentials must be configured with verified sending domain and API key.| Mailgun official site: https://www.mailgun.com/                                                             |
| MongoDB node requires connection to a MongoDB instance with the configured database and collection.| MongoDB Atlas or self-hosted MongoDB can be used.                                                           |
| The webhook is designed to be publicly accessible and suitable for scheduled or app-triggered calls.| Ensure secure exposure of webhook endpoint, consider authentication or IP whitelisting if needed.           |
| The price analysis currently uses dummy previous prices; for real alerts, integrate a DB query to fetch last stored prices.| This will require adding a MongoDB Find node or similar to fetch previous prices before analysis.            |
| Parallel processing via SplitInBatches optimizes runtime but may be subject to API rate limits.| Monitor API quotas and adjust batch concurrency accordingly.                                                |
| Setup instructions and workflow logic are summarized in sticky notes within the workflow UI.  | Review sticky notes in n8n editor for contextual help.                                                       |

---

**Disclaimer:**  
The content provided is derived solely from an automated n8n workflow configuration and adheres to applicable content policies. No illegal, offensive, or protected data is included. All handled data is lawful and publicly accessible.