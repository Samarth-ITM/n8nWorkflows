E-commerce Product Price Tracker with ScrapeGraphAI, Baserow and Slack Alerts

https://n8nworkflows.xyz/workflows/e-commerce-product-price-tracker-with-scrapegraphai--baserow-and-slack-alerts-11609


# E-commerce Product Price Tracker with ScrapeGraphAI, Baserow and Slack Alerts

---

## 1. Workflow Overview

This workflow automates weekly monitoring of product prices from e-commerce websites using AI-powered scraping, data normalization, storage, and alerting. It targets retail analysts, merchandisers, and pricing teams who want timely price intelligence without manual tracking.

The workflow is logically divided into five main blocks:

- **1.1 Data Collection:** Triggered weekly, it defines product URLs, iterates over each product, and uses ScrapeGraphAI to extract structured product data (name, price, currency, availability) resiliently despite website layout changes.

- **1.2 Data Processing:** Cleans and enriches scraped data by normalizing prices, adding timestamps, and filtering out invalid or missing price data to maintain data quality.

- **1.3 Data Storage:** Persistently stores validated price records into a Baserow database table for historical analytics and reporting.

- **1.4 Alerting:** Compares current prices against a configurable threshold and sends Slack alerts when prices fall below this threshold to notify relevant stakeholders immediately.

- **1.5 Error Handling:** Manages and logs scraping errors and missing price data separately to maintain workflow robustness and transparency.

---

## 2. Block-by-Block Analysis

### 2.1 Data Collection

**Overview:**  
This block triggers the workflow on a weekly schedule, defines the list of product URLs to monitor, and invokes the AI scraper to extract product information. It processes products one at a time to avoid rate-limiting.

**Nodes Involved:**  
- Weekly Schedule  
- Define Product Sources  
- Iterate Products  
- Scrape Product Page  

**Node Details:**

- **Weekly Schedule**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow weekly (every 7 days) automatically.  
  - *Configuration:* Interval set to 1 week.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Passes control to Define Product Sources.  
  - *Edge Cases:* Missed triggers if n8n instance offline; interval accuracy depends on server uptime.

- **Define Product Sources**  
  - *Type:* Code  
  - *Role:* Contains hardcoded list of products with names and URLs to be scraped.  
  - *Configuration:* JavaScript array returning objects with `productName` and `url`. Can be edited to add/remove products.  
  - *Key Expressions:* Returns JSON array of products; no external inputs.  
  - *Inputs:* Trigger from Weekly Schedule.  
  - *Outputs:* Array of products passed to Iterate Products.  
  - *Edge Cases:* Incorrect URLs or typos may cause scraping failures.

- **Iterate Products**  
  - *Type:* SplitInBatches  
  - *Role:* Processes one product at a time to control scraping rate and avoid IP blocking or rate-limiting.  
  - *Configuration:* Default batch size (1).  
  - *Inputs:* List of products from Define Product Sources.  
  - *Outputs:* Single product object per iteration to Scrape Product Page.  
  - *Edge Cases:* Large product lists increase total run time linearly.

- **Scrape Product Page**  
  - *Type:* ScrapeGraphAI  
  - *Role:* Uses AI to scrape product page and extract structured JSON data (name, price, currency, availability).  
  - *Configuration:* User prompt instructs AI to extract JSON with fields: name (string), currentPrice (string), currency (string), availability (string). URL dynamically set from current product’s URL.  
  - *Inputs:* Single product URL from Iterate Products.  
  - *Outputs:* Scraped JSON data to Clean & Enrich Data; errors routed to Error Handler.  
  - *Edge Cases:* Possible scraping failures due to CAPTCHAs, site changes, or network errors; handled downstream.  
  - *Version Requirements:* ScrapeGraphAI node with proper API credentials configured.  

---

### 2.2 Data Processing

**Overview:**  
Normalizes and enriches the scraped data, converting textual price to numeric form, adding timestamps, and validating presence of price data to filter out incomplete records.

**Nodes Involved:**  
- Clean & Enrich Data  
- Has Price Data?  
- Log Missing Price  

**Node Details:**

- **Clean & Enrich Data**  
  - *Type:* Code  
  - *Role:* Processes raw scraped data to extract numeric price, standardizes fields, and adds scrape timestamp.  
  - *Configuration:* JavaScript that parses price strings removing currency symbols and thousand separators, converts to float, sets fallback currency "USD" if missing, and includes scrapedAt timestamp. Also sets productName fallback.  
  - *Inputs:* Scraped JSON product data from Scrape Product Page.  
  - *Outputs:* Cleaned, enriched product records to Has Price Data?  
  - *Edge Cases:* Malformed or missing price strings lead to null price; handled downstream.

- **Has Price Data?**  
  - *Type:* If  
  - *Role:* Quality gate that checks if the numeric price is greater than zero.  
  - *Configuration:* Condition: price > 0.  
  - *Inputs:* Cleaned product data.  
  - *Outputs:*  
    - True: Valid price, forward to Store Price Record.  
    - False: Invalid/missing price, forward to Log Missing Price.  
  - *Edge Cases:* Zero or negative prices filtered out; avoids storing bad data.

- **Log Missing Price**  
  - *Type:* Code  
  - *Role:* Logs warnings for products missing price data without stopping workflow.  
  - *Configuration:* Console warning with product name.  
  - *Inputs:* Items failing price check.  
  - *Outputs:* Passes items unchanged for possible further processing or logging.  
  - *Edge Cases:* Logging only; no failure expected.

---

### 2.3 Data Storage

**Overview:**  
Stores validated and enriched product pricing data into a Baserow database table for historical tracking and analytics.

**Nodes Involved:**  
- Store Price Record  

**Node Details:**

- **Store Price Record**  
  - *Type:* Baserow  
  - *Role:* Creates new row in specified Baserow table with product pricing data.  
  - *Configuration:*  
    - Table ID linked to environment variable `BASEROW_TABLE_ID` or default `1`.  
    - Maps fields: productName, price, currency, availability, url, scrapedAt.  
  - *Inputs:* Validated product data from Has Price Data? (True branch).  
  - *Outputs:* Passes record to Is Price Below Threshold? node.  
  - *Edge Cases:* API token misconfiguration or network errors cause failures; requires correct Baserow credentials.

---

### 2.4 Alerting

**Overview:**  
Checks if the price is below a configurable threshold and sends an alert message via Slack with product details.

**Nodes Involved:**  
- Is Price Below Threshold?  
- Prepare Mailchimp Content (used here for Slack message preparation)  
- Send a message (Slack)  

**Node Details:**

- **Is Price Below Threshold?**  
  - *Type:* If  
  - *Role:* Compares current price against alert threshold.  
  - *Configuration:* Condition: price < `PRICE_ALERT_THRESHOLD` environment variable or default 50.  
  - *Inputs:* Stored price record from Store Price Record.  
  - *Outputs:* True branch triggers alert preparation.  
  - *Edge Cases:* Threshold misconfiguration may cause false alerts or misses.

- **Prepare Mailchimp Content**  
  - *Type:* Set  
  - *Role:* Prepares message content payload for alert. Despite the node name, it is used here to format Slack message content.  
  - *Configuration:* Sets fields for alert message (e.g., productName, price, URL). Exact fields not detailed but implied.  
  - *Inputs:* Price data from Is Price Below Threshold?  
  - *Outputs:* Passes formatted message to Send a message node.  
  - *Edge Cases:* Misformatted messages may cause Slack API errors.

- **Send a message**  
  - *Type:* Slack  
  - *Role:* Sends alert message to Slack channel via webhook.  
  - *Configuration:* Uses Slack webhook ID `f1347800-ebb4-4086-b72a-55cf6219d4e8`. No extra options set.  
  - *Inputs:* Message content from Prepare Mailchimp Content.  
  - *Outputs:* None (terminal node).  
  - *Edge Cases:* Slack API rate limits, webhook misconfiguration, or network issues may prevent alert delivery.  
  - *Version Requirements:* Slack node v2.3 or compatible with webhook usage.

---

### 2.5 Error Handling

**Overview:**  
Captures and logs errors from the scraping process and missing price data to maintain workflow stability and provide operational visibility.

**Nodes Involved:**  
- Error Handler  
- Log Missing Price (also part of Data Processing block)  

**Node Details:**

- **Error Handler**  
  - *Type:* Code  
  - *Role:* Catches scraping errors routed from Scrape Product Page and logs them to the n8n console.  
  - *Configuration:* JavaScript code logs error details for diagnostics.  
  - *Inputs:* Error objects from Scrape Product Page node’s error output.  
  - *Outputs:* Passes items unchanged to prevent workflow halt.  
  - *Edge Cases:* Does not escalate errors; consider integration with monitoring tools for production.

---

## 3. Summary Table

| Node Name            | Node Type               | Functional Role                      | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                      |
|----------------------|-------------------------|------------------------------------|------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------|
| Weekly Schedule      | Schedule Trigger        | Triggers workflow weekly           | —                      | Define Product Sources    | ## Data Collection - Triggers workflow every 7 days to kick off scraping process.                                |
| Define Product Sources | Code                    | Defines product URLs                | Weekly Schedule         | Iterate Products         | ## Data Collection - Contains list of products with URLs to scrape.                                             |
| Iterate Products     | SplitInBatches          | Processes products one by one      | Define Product Sources  | Scrape Product Page      | ## Data Collection - Sends one product at a time to avoid rate limits.                                          |
| Scrape Product Page  | ScrapeGraphAI           | AI-powered product data extraction | Iterate Products        | Clean & Enrich Data, Error Handler | ## Data Collection - Uses AI to extract structured product data resiliently from pages.                    |
| Clean & Enrich Data  | Code                    | Normalizes and enriches data       | Scrape Product Page     | Has Price Data?          | ## Data Processing - Converts price to number, adds timestamp, and validates data.                              |
| Has Price Data?      | If                      | Validates presence of price        | Clean & Enrich Data     | Store Price Record, Log Missing Price | ## Data Processing - Filters out invalid price records.                                                  |
| Log Missing Price    | Code                    | Logs missing price data            | Has Price Data? (False) | —                        | ## Error Handling - Warns about missing price data without stopping flow.                                       |
| Store Price Record   | Baserow                 | Stores price data historically     | Has Price Data? (True)  | Is Price Below Threshold? | ## Data Storage - Inserts clean data into Baserow for analytics.                                               |
| Is Price Below Threshold? | If                      | Checks if price is below threshold | Store Price Record      | Prepare Mailchimp Content | ## Alerting - Triggers alert if price drops below configured threshold.                                         |
| Prepare Mailchimp Content | Set                     | Prepares content for alert message | Is Price Below Threshold? | Send a message           | ## Alerting - Formats alert message for Slack notification.                                                    |
| Send a message       | Slack                   | Sends alert to Slack channel       | Prepare Mailchimp Content | —                        | ## Alerting - Pushes notification to Slack using webhook.                                                      |
| Error Handler        | Code                    | Logs scraping errors               | Scrape Product Page     | —                        | ## Error Handling - Captures and logs scraping errors to console.                                              |
| Workflow Overview    | Sticky Note             | Describes overall workflow         | —                      | —                        | ## Workflow Overview - Explains workflow purpose and setup instructions.                                       |
| Section – Data Collection | Sticky Note             | Describes data collection block    | —                      | —                        | ## Data Collection - Summarizes scraping and scheduling process.                                               |
| Section – Data Processing | Sticky Note             | Describes data processing block    | —                      | —                        | ## Data Processing - Explains data normalization and validation.                                               |
| Section – Data Storage | Sticky Note             | Describes data storage block       | —                      | —                        | ## Data Storage - Details storage into Baserow for analytics.                                                  |
| Section – Alerting   | Sticky Note             | Describes alerting block           | —                      | —                        | ## Alerting - Details alert flow and Slack notification.                                                       |
| Section – Error Handling | Sticky Note             | Describes error handling block     | —                      | —                        | ## Error Handling - Details error logging and monitoring approach.                                            |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger every 1 week (interval: 1 week).  
   - Name the node "Weekly Schedule".

3. **Add a Code node for product definitions:**  
   - Name: "Define Product Sources"  
   - Paste JavaScript code defining an array of product objects with fields `productName` and `url`. Example:  
     ```js
     return [
       { json: { productName: 'Winter Jacket A', url: 'https://example-store.com/winter-jacket-a' } },
       { json: { productName: 'Snow Boots B', url: 'https://example-store.com/snow-boots-b' } }
     ];
     ```
   - Connect output of "Weekly Schedule" to this node.

4. **Add a SplitInBatches node:**  
   - Name: "Iterate Products"  
   - Default batch size (1) to process one product at a time.  
   - Connect output of "Define Product Sources" to this node.

5. **Add ScrapeGraphAI node:**  
   - Name: "Scrape Product Page"  
   - Configure credentials for ScrapeGraphAI API.  
   - Set parameter `websiteUrl` to: `={{ $json.url }}` to dynamically scrape current product URL.  
   - Set `userPrompt` to:  
     ```
     Extract the following as JSON: {"name": "string", "currentPrice": "string", "currency": "string", "availability": "string"}. Make sure numbers include currency symbols if present.
     ```  
   - Connect output of "Iterate Products" to this node.

6. **Add a Code node to clean and enrich data:**  
   - Name: "Clean & Enrich Data"  
   - Paste JavaScript code to:  
     - Parse `currentPrice` string to numeric price.  
     - Add fallback currency `"USD"` if missing.  
     - Add `scrapedAt` timestamp with current ISO datetime.  
     - Use fallback product name if scraped name missing.  
   - Connect output of "Scrape Product Page" to this node.

7. **Add an If node to check price validity:**  
   - Name: "Has Price Data?"  
   - Condition: Check if `price` > 0.  
   - Connect output of "Clean & Enrich Data" to this node.

8. **Add Code node for logging missing prices:**  
   - Name: "Log Missing Price"  
   - JavaScript:  
     ```js
     console.warn('No price found for', $json.productName);
     return items;
     ```  
   - Connect False output of "Has Price Data?" to this node.

9. **Add Baserow node to store price record:**  
   - Name: "Store Price Record"  
   - Configure Baserow credentials and set Table ID (use env var `BASEROW_TABLE_ID` or default 1).  
   - Map fields: productName, price, currency, availability, url, scrapedAt.  
   - Connect True output of "Has Price Data?" to this node.

10. **Add an If node to check price threshold:**  
    - Name: "Is Price Below Threshold?"  
    - Condition: price < environment variable `PRICE_ALERT_THRESHOLD` or default 50.  
    - Connect output of "Store Price Record" to this node.

11. **Add a Set node to prepare alert content:**  
    - Name: "Prepare Mailchimp Content" (name retained but used for Slack content)  
    - Configure fields to format alert message including product name, price, and URL.  
    - Connect True output of "Is Price Below Threshold?" to this node.

12. **Add Slack node to send message:**  
    - Name: "Send a message"  
    - Configure Slack webhook credentials using webhook ID `f1347800-ebb4-4086-b72a-55cf6219d4e8`.  
    - Connect output of "Prepare Mailchimp Content" to this node.

13. **Add a Code node for error handling:**  
    - Name: "Error Handler"  
    - JavaScript:  
      ```js
      const err = $input.item.json;
      console.error('Scrape error', err);
      return items;
      ```  
    - Connect error output of "Scrape Product Page" node to this node.

14. **Add Sticky Notes:**  
    - Create sticky notes for each logical block describing functionality and setup instructions as per the workflow’s annotations. This helps maintain clarity and documentation within the workflow.

15. **Configure environment variables:**  
    - `BASEROW_TABLE_ID`: Baserow table ID to store price records.  
    - `PRICE_ALERT_THRESHOLD`: Numeric price threshold to trigger alerts.  
    - Set up credentials for ScrapeGraphAI, Baserow API token, and Slack Webhook.

16. **Test and activate the workflow.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| The workflow extracts pricing data using ScrapeGraphAI, an AI-powered tool that adapts to webpage layout changes without fragile selectors. | Official ScrapeGraphAI documentation (not linked in workflow)                                                    |
| Storing data in Baserow allows building historical price trend dashboards without extra tooling.                                            | Baserow: https://baserow.io                                                                                       |
| Slack alerts provide immediate notifications with open-rate and click tracking via Mailchimp integration (name reused for convenience).    | Slack Webhook setup: https://api.slack.com/messaging/webhooks                                                    |
| Error handling nodes log both technical scraping errors and missing business data separately to maintain transparency.                      | Suggested to integrate with monitoring tools like Sentry or Slack for real-time alerts                            |
| Setup instructions are detailed in the "Workflow Overview" sticky note node for easy administrator reference.                               | Sticky note content inside workflow JSON                                                                          |

---

**Disclaimer:**  
The content provided originates exclusively from an automated workflow authored in n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---