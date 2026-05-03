Automated Product Price Tracking with ScrapeGraphAI, Slack Alerts and Jira Tickets

https://n8nworkflows.xyz/workflows/automated-product-price-tracking-with-scrapegraphai--slack-alerts-and-jira-tickets-11608


# Automated Product Price Tracking with ScrapeGraphAI, Slack Alerts and Jira Tickets

### 1. Workflow Overview

This workflow automates product price monitoring by ingesting a list of product URLs and expected prices via a webhook POST request. It scrapes live product data using ScrapeGraphAI, calculates price changes, and triggers alerts or logs issues based on price drop thresholds. The workflow targets e-commerce analysts or operations teams aiming to track price fluctuations efficiently with automated Slack notifications and Jira ticket logging.

Logical blocks:

- **1.1 Input Reception and Validation**: Receiving incoming data via webhook, validating and transforming the product list.
- **1.2 Scraping and Data Enrichment**: Throttled scraping of product pages to extract live product details, merging scraped data with expected prices, and computing price change percentages.
- **1.3 Decision Logic**: Determining if price drops exceed a threshold for alerting.
- **1.4 Outputs and Notifications**: Sending Slack alerts for significant drops and creating Jira issues for routine price changes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block starts the workflow by accepting HTTP POST requests containing product URLs and expected prices. It validates the payload, ensuring it contains a non-empty array of products, then transforms each product into a separate item for parallel processing downstream.

**Nodes Involved:**  
- Incoming Product List (Webhook)  
- Validate & Prepare Data (Code)  
- Iterate Product List (SplitInBatches)  

**Node Details:**

- **Incoming Product List**  
  - Type: Webhook  
  - Role: Entry point accepting POST requests at `/product-price-monitor` path.  
  - Config: Accepts POST only; no additional options.  
  - Inputs: External HTTP POST request with JSON body `{ products: [ { url, expectedPrice } ] }`.  
  - Outputs: Passes raw request data to next node.  
  - Failure cases: Missing or malformed requests; webhook errors.  
  - Notes: Webhook URL must be exposed and reachable by caller.

- **Validate & Prepare Data**  
  - Type: Code (JavaScript)  
  - Role: Validates input structure and flattens products array into individual items.  
  - Config: Checks for presence and non-empty `products` array; throws error if invalid. Maps each product to `{ url, expectedPrice }` fields.  
  - Input: Raw webhook JSON.  
  - Output: Array of items, each with product fields.  
  - Expressions: None complex; straightforward JS logic.  
  - Failure cases: Missing or empty products array triggers error; malformed product objects.  
  - Notes: Acts as validation perimeter and data normalizer for downstream consistency.

- **Iterate Product List**  
  - Type: SplitInBatches  
  - Role: Controls rate of processing per batch to avoid overload on scraping API and target sites.  
  - Config: Default batch size and options (not explicitly set).  
  - Input: List of product items from code node.  
  - Output: Items flow one batch at a time downstream.  
  - Failure cases: Batch processing errors or API rate limits downstream.

---

#### 1.2 Scraping and Data Enrichment

**Overview:**  
This block scrapes live product data from URLs using ScrapeGraphAI, merges scraped data with original expected prices, and calculates the percentage price change.

**Nodes Involved:**  
- Scrape Product Page (ScrapeGraphAI)  
- Merge Scraped & Expected (Merge)  
- Compute Price Change (Code)  

**Node Details:**

- **Scrape Product Page**  
  - Type: ScrapeGraphAI node  
  - Role: Renders product page, extracts structured data such as name, current price, currency, availability, and seasonal tags.  
  - Config: Uses a prompt instructing AI to extract fields in JSON format; dynamically uses `url` from input item.  
  - Input: Single product item with `url`.  
  - Output: JSON containing scraped product info.  
  - Failure cases: API errors, invalid URLs, timeouts, incomplete data extraction.  
  - Notes: Requires configured ScrapeGraphAI API credentials.

- **Merge Scraped & Expected**  
  - Type: Merge  
  - Role: Combines scraped product data with original expected price item to preserve context.  
  - Config: Mode set to "combine" to join data from both inputs without losing fields.  
  - Input: Two inputs — main from ScrapeGraphAI, second from original product item.  
  - Output: Single enriched item with combined fields.  
  - Failure cases: Merge conflicts or missing inputs.

- **Compute Price Change**  
  - Type: Code (JavaScript)  
  - Role: Converts prices to float, calculates percentage price change relative to expected price, appends results as new fields.  
  - Config: Parses `price` and `expectedPrice`; calculates `priceChangePercent = ((price - expected) / expected) * 100`.  
  - Input: Merged product data.  
  - Output: Item enriched with `price`, `expectedPrice`, and `priceChangePercent`.  
  - Failure cases: Non-numeric or zero expected prices leading to invalid calculations; NaN results handled by null change.

---

#### 1.3 Decision Logic

**Overview:**  
This block decides if a product's price drop is significant enough to trigger an alert or just to log the event quietly.

**Nodes Involved:**  
- Significant Price Drop? (IF)  

**Node Details:**

- **Significant Price Drop?**  
  - Type: IF  
  - Role: Compares `priceChangePercent` to threshold (-10%).  
  - Config: Condition checks if `priceChangePercent < -10` (i.e., price is at least 10% lower than expected).  
  - Input: Enriched product data with price change.  
  - Outputs: Two branches — true for significant drop, false for minor/no drop.  
  - Failure cases: Missing or invalid `priceChangePercent` fields.

---

#### 1.4 Outputs and Notifications

**Overview:**  
This final block has two parallel branches: one posts immediate Slack alerts for significant price drops; the other logs all changes as Jira issues for later analysis.

**Nodes Involved:**  
- Format Slack Alert (Set)  
- Send Slack Message (Slack)  
- Prepare Jira Issue (Set)  
- Create an issue (Jira)  

**Node Details:**

- **Format Slack Alert**  
  - Type: Set  
  - Role: Constructs a concise Slack message string summarizing product name, live price, price delta percentage, and product URL as clickable link.  
  - Config: Uses dot notation to build message text fields.  
  - Input: Items where price drop condition is true.  
  - Output: JSON with formatted Slack message.  
  - Failure cases: Missing product fields causing incomplete messages.

- **Send Slack Message**  
  - Type: Slack  
  - Role: Posts message to configured Slack channel.  
  - Config: Uses `postMessage` operation; Slack credentials and channel selection required.  
  - Input: Formatted Slack message from Set node.  
  - Output: Slack API response.  
  - Failure cases: Slack auth errors, invalid channel, rate limits.

- **Prepare Jira Issue**  
  - Type: Set  
  - Role: Maps product data into Jira issue fields such as `summary` and `description` for logging.  
  - Config: Uses dot notation to build issue fields; expects project key and issue type configured in Jira node.  
  - Input: Items where price drop condition is false (minor changes).  
  - Output: JSON structured for Jira issue creation.  
  - Failure cases: Missing required fields for Jira issue creation.

- **Create an issue**  
  - Type: Jira  
  - Role: Creates a new Jira issue in specified project with given details.  
  - Config: Requires Jira cloud credentials; project key and issue type selected from list.  
  - Input: Prepared issue data from Set node.  
  - Output: Jira API response with created issue info.  
  - Failure cases: Jira authentication errors, invalid project or issue type, API rate limits.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                          | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                   |
|------------------------|-----------------------|----------------------------------------|-------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview       | Sticky Note           | Explains workflow purpose and setup    |                         |                           | ## How it works ... (full content describing webhook-driven price monitoring pipeline and setup steps)                       |
| Section – Intake        | Sticky Note           | Describes webhook and payload validation|                         |                           | ## Webhook & Preparation ... (details on input validation and data flattening)                                                |
| Section – Scraping & Data| Sticky Note          | Details scraping and enrichment steps  |                         |                           | ## Scraping & Enrichment ... (explains scraping, merging, and enrichment logic)                                                |
| Section – Logic         | Sticky Note           | Explains decision logic for alerts     |                         |                           | ## Decision Logic ... (describes IF node for price drop threshold)                                                            |
| Section – Outputs       | Sticky Note           | Describes alerting and storage outputs |                         |                           | ## Alerts & Storage Outputs ... (Slack alerts and Jira issue creation explained)                                              |
| Incoming Product List   | Webhook               | Entry point receiving product list     |                         | Validate & Prepare Data    |                                                                                                                               |
| Validate & Prepare Data | Code                  | Validates and flattens input data      | Incoming Product List    | Iterate Product List       |                                                                                                                               |
| Iterate Product List    | SplitInBatches        | Controls batch processing rate         | Validate & Prepare Data  | Scrape Product Page, Merge |                                                                                                                               |
| Scrape Product Page     | ScrapeGraphAI         | Scrapes live product data               | Iterate Product List     | Merge Scraped & Expected   |                                                                                                                               |
| Merge Scraped & Expected| Merge                 | Combines scraped and expected data     | Scrape Product Page, Iterate Product List | Compute Price Change |                                                                                                                               |
| Compute Price Change    | Code                  | Calculates price change percentage      | Merge Scraped & Expected | Significant Price Drop?    |                                                                                                                               |
| Significant Price Drop? | IF                    | Checks if price drop exceeds threshold | Compute Price Change     | Format Slack Alert, Prepare Jira Issue |                                                                                                                     |
| Format Slack Alert      | Set                   | Formats message for Slack alert         | Significant Price Drop? (true) | Send Slack Message   |                                                                                                                               |
| Send Slack Message      | Slack                 | Sends alert to Slack channel            | Format Slack Alert       |                           |                                                                                                                               |
| Prepare Jira Issue      | Set                   | Prepares Jira issue data                 | Significant Price Drop? (false) | Create an issue     |                                                                                                                               |
| Create an issue         | Jira                  | Creates Jira task with product data     | Prepare Jira Issue       |                           |                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Incoming Product List"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `product-price-monitor`  
   - No authentication required  
   - Purpose: Entry point for receiving JSON with product URLs and expected prices.

2. **Create Code Node: "Validate & Prepare Data"**  
   - Type: Code (JavaScript)  
   - Link input from "Incoming Product List"  
   - Code snippet:
     ```js
     const body = $json;
     if (!body.products || !Array.isArray(body.products) || body.products.length === 0) {
       throw new Error('Request body must contain a products array with at least one element');
     }
     return body.products.map(p => ({ json: { url: p.url, expectedPrice: p.expectedPrice } }));
     ```
   - Purpose: Validate payload and convert product array into individual items.

3. **Create SplitInBatches Node: "Iterate Product List"**  
   - Type: SplitInBatches  
   - Connect input from "Validate & Prepare Data"  
   - Default batch size (can be adjusted to throttle API calls)  
   - Purpose: Control processing rate for scraping.

4. **Create ScrapeGraphAI Node: "Scrape Product Page"**  
   - Type: ScrapeGraphAI  
   - Connect input from "Iterate Product List"  
   - Configuration:  
     - Website URL: `={{ $json.url }}`  
     - User Prompt:  
       ```
       Extract the product name, current price, currency, availability status, category, and any seasonal tags (e.g., summer, winter) from this product page. Respond as JSON: {"name": "string", "price": "number", "currency": "string", "availability": "string", "season": "string"}
       ```
   - Credentials: Add ScrapeGraphAI API credentials.  
   - Purpose: Extract live product data.

5. **Create Merge Node: "Merge Scraped & Expected"**  
   - Type: Merge  
   - Mode: Combine  
   - Connect first input to output of "Scrape Product Page"  
   - Connect second input to output of "Iterate Product List"  
   - Purpose: Combine live scraped data with original expected price.

6. **Create Code Node: "Compute Price Change"**  
   - Type: Code (JavaScript)  
   - Connect input from "Merge Scraped & Expected"  
   - Code snippet:
     ```js
     const data = $json;
     const price = parseFloat(data.price);
     const expected = parseFloat(data.expectedPrice);
     let change = null;
     if (!isNaN(price) && !isNaN(expected) && expected !== 0) {
       change = ((price - expected) / expected) * 100;
     }
     return [{ json: { ...data, price: price, expectedPrice: expected, priceChangePercent: change } }];
     ```
   - Purpose: Calculate percentage price change.

7. **Create IF Node: "Significant Price Drop?"**  
   - Type: IF  
   - Connect input from "Compute Price Change"  
   - Condition:  
     - Numeric condition  
     - Value1: `={{ $json.priceChangePercent }}`  
     - Operation: Smaller than  
     - Value2: `-10`  
   - Purpose: Determine if price drop exceeds 10%.

8. **Create Set Node: "Format Slack Alert"**  
   - Type: Set  
   - Connect input from IF node’s "true" output  
   - Dot notation enabled  
   - Set fields to build message string, e.g.:  
     - `message`: `Product "{{ $json.name }}" price dropped to {{ $json.price }} ({{ $json.priceChangePercent.toFixed(2) }}%) - <{{ $json.url }}|View Product>`  
   - Purpose: Prepare Slack notification content.

9. **Create Slack Node: "Send Slack Message"**  
   - Type: Slack  
   - Connect input from "Format Slack Alert"  
   - Operation: Post message  
   - Credentials: Configure Slack OAuth2 credentials with required scopes  
   - Channel: Select alert channel  
   - Message: Use `message` field from previous node  
   - Purpose: Send immediate alerts.

10. **Create Set Node: "Prepare Jira Issue"**  
    - Type: Set  
    - Connect input from IF node’s "false" output  
    - Dot notation enabled  
    - Map fields for Jira issue, e.g.:  
      - `summary`: `Price update for {{ $json.name }}`  
      - `description`: Detailed info including expected price, live price, price change, and URL  
    - Purpose: Prepare data for Jira logging.

11. **Create Jira Node: "Create an issue"**  
    - Type: Jira  
    - Connect input from "Prepare Jira Issue"  
    - Operation: Create issue  
    - Credentials: Configure Jira cloud credentials  
    - Set fields:  
      - Project key: Select your Jira project  
      - Issue type: Select issue type (e.g., Task)  
    - Purpose: Log price changes for analysis.

12. **Activate Workflow** and test by sending POST requests with product data to webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow is designed to be modular: validation, scraping, logic, and output are separate blocks allowing easy customization and extension (e.g., adding more alert channels or enriching data further).                    | Workflow design principle                                                                          |
| ScrapeGraphAI node requires valid API credentials; rate limits and scraping reliability depend on the target website’s structure and bot protection mechanisms.                                                             | ScrapeGraphAI API documentation                                                                   |
| Slack node needs OAuth2 credentials with `chat:write` scope and the channel must be accessible by the bot user.                                                                                                              | Slack API documentation                                                                            |
| Jira node requires cloud credentials with permission to create issues in the specified project.                                                                                                                             | Jira Cloud API documentation                                                                       |
| To adjust price drop sensitivity, modify the numeric threshold in the IF node (default is -10%).                                                                                                                            | Workflow configuration                                                                             |
| For more complex decision rules, add additional conditions or replace the IF node with a Switch node without affecting upstream/downstream logic.                                                                           | N8N IF and Switch node documentation                                                              |
| This workflow exemplifies webhook-driven automation integrating AI scraping and enterprise collaboration tools for efficient monitoring and alerts.                                                                         | Project summary                                                                                   |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.