Real-Time Bitcoin Price Alerts with Bright Data & n8n

https://n8nworkflows.xyz/workflows/real-time-bitcoin-price-alerts-with-bright-data---n8n-5217


# Real-Time Bitcoin Price Alerts with Bright Data & n8n

---

## 1. Workflow Overview

This workflow is designed to monitor the real-time price of Bitcoin and send an email alert if the price drops by 5% or more compared to the last recorded price stored in a Google Sheet. It targets cryptocurrency traders, analysts, and enthusiasts who want automated notifications about significant price drops without manual monitoring.

The workflow is structured into three logical blocks:

- **1.1 Fetch Current Bitcoin Price**: Initiates the workflow and retrieves the live Bitcoin price from a public website using the Bright Data scraping API.
- **1.2 Calculate and Compare with Historical Price**: Converts and compares the current price with the last recorded price from Google Sheets to calculate the drop percentage.
- **1.3 Alert Decision and Notification**: Evaluates if the drop exceeds the 5% threshold and sends an email alert if necessary, or ends silently otherwise.

---

## 2. Block-by-Block Analysis

### 1.1 Fetch Current Bitcoin Price

**Overview:**  
This block manually triggers the workflow and fetches the current Bitcoin price by scraping a website through the Bright Data API, then extracts the price from the HTML content.

**Nodes Involved:**  
- Manual Trigger  
- Fetch Bitcoin Price  
- Extract Price

**Node Details:**

- **Manual Trigger**  
  - *Type & Role:* Trigger node to start the workflow manually.  
  - *Configuration:* No parameters; acts as a button to initiate the process.  
  - *Connections:* Outputs to "Fetch Bitcoin Price".  
  - *Edge Cases:* None; manual trigger expected to always succeed.  
  - *Version:* 1

- **Fetch Bitcoin Price**  
  - *Type & Role:* HTTP Request node to POST to Bright Data API to scrape the Bitcoin price page.  
  - *Configuration:*  
    - URL: `https://api.brightdata.com/request`  
    - Method: POST  
    - Body parameters:  
      - zone: `n8n_unblocker` (proxy zone for scraping)  
      - url: `https://coinmarketcap.com/currencies/bitcoin/` (target page)  
      - country: `us` (geolocation for scraping)  
      - format: `raw` (return raw HTML)  
    - Header: Authorization Bearer token (API_KEY placeholder) for Bright Data authentication.  
  - *Connections:* Outputs to "Extract Price".  
  - *Edge Cases:*  
    - HTTP errors (timeouts, 401 Unauthorized if API key invalid)  
    - Changes in target webpage structure may affect scraping success  
  - *Version:* 4.2

- **Extract Price**  
  - *Type & Role:* HTML node to extract Bitcoin price from scraped HTML using CSS selector.  
  - *Configuration:*  
    - Operation: Extract content via CSS selector  
    - Selector used targets a span with class `sc-65e7f566-0 esyGGG base-text` and a data-test attribute `text-cdp-price-display`  
  - *Key Expressions:* Extracted value stored under `price`.  
  - *Connections:* Outputs to "Convert type".  
  - *Edge Cases:*  
    - Selector failure if webpage structure changes  
    - Missing or malformed HTML content  
  - *Version:* 1.2

---

### 1.2 Calculate and Compare with Historical Price

**Overview:**  
Converts the scraped price string to a numeric value, retrieves the last recorded price from Google Sheets, and calculates the percentage drop.

**Nodes Involved:**  
- Convert type  
- Get Last Recorded Price  
- Find drop percentage

**Node Details:**

- **Convert type**  
  - *Type & Role:* Code node to parse price string into a float number for calculations.  
  - *Configuration:*  
    - JavaScript code removes `$` and commas, converts to float, and returns the numeric price.  
  - *Inputs:* JSON with `price` string from "Extract Price".  
  - *Outputs:* JSON with numeric `price`.  
  - *Connections:* Outputs to "Get Last Recorded Price".  
  - *Edge Cases:*  
    - Parsing errors if input format unexpected  
  - *Version:* 2

- **Get Last Recorded Price**  
  - *Type & Role:* Google Sheets node to read the last stored Bitcoin price from a specific Google Sheet.  
  - *Configuration:*  
    - Document ID: specific Google Sheet storing historical prices  
    - Sheet Name: `gid=0` (default first sheet)  
    - Credential: Google Sheets OAuth2 configured  
  - *Inputs:* From "Convert type" (via connection)  
  - *Outputs:* JSON with previous price under property `Price` (case-sensitive)  
  - *Connections:* Outputs to "Find drop percentage".  
  - *Edge Cases:*  
    - Authentication errors (expired OAuth token)  
    - Sheet not found or empty data  
  - *Version:* 4.6

- **Find drop percentage**  
  - *Type & Role:* Code node calculating the percentage drop between previous and current prices.  
  - *Configuration:*  
    - Reads current price from "Convert type"  
    - Reads previous price from "Get Last Recorded Price"  
    - Calculates drop % as `((previous - current)/previous)*100`  
    - Handles division by zero by returning 0 drop  
    - Round to two decimals  
  - *Outputs:* JSON with `currentPrice`, `previousPrice`, and `dropPercent`.  
  - *Connections:* Outputs to "If droppercentage >= 5".  
  - *Edge Cases:*  
    - Missing or zero previous price  
    - Unexpected input formats  
  - *Version:* 2

---

### 1.3 Alert Decision and Notification

**Overview:**  
Checks if the drop percentage is at least 5% and sends an alert email if true, otherwise ends the workflow quietly.

**Nodes Involved:**  
- If droppercentage >= 5  
- Send Alert Email  
- No Alert Needed

**Node Details:**

- **If droppercentage >= 5**  
  - *Type & Role:* If node to conditionally branch workflow based on drop percentage.  
  - *Configuration:*  
    - Condition: numeric check if `dropPercent` ≥ 5  
  - *Inputs:* From "Find drop percentage".  
  - *Outputs:*  
    - True branch: "Send Alert Email"  
    - False branch: "No Alert Needed"  
  - *Edge Cases:*  
    - Missing `dropPercent` value  
  - *Version:* 2.2

- **Send Alert Email**  
  - *Type & Role:* Gmail node to send notification email about the Bitcoin price drop.  
  - *Configuration:*  
    - Send to: `shahkar.genai@gmail.com` (hardcoded recipient)  
    - Subject: dynamic, includes drop percentage  
    - Message body: text with previous and current price details and drop percent  
    - Email type: plain text  
    - Gmail OAuth2 credentials configured for authentication  
  - *Inputs:* From "If droppercentage >= 5" (true)  
  - *Edge Cases:*  
    - Email sending failures (OAuth issues, quota limits)  
  - *Version:* 2.1

- **No Alert Needed**  
  - *Type & Role:* NoOp node to end workflow silently if no alert is required.  
  - *Inputs:* From "If droppercentage >= 5" (false)  
  - *Edge Cases:* None  
  - *Version:* 1

---

## 3. Summary Table

| Node Name               | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                   |
|-------------------------|---------------------|---------------------------------------|------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Manual Trigger          | manualTrigger       | Start workflow manually                | —                      | Fetch Bitcoin Price      | See Sticky Note "Section 1: Fetch the Current Bitcoin Price"                                                 |
| Fetch Bitcoin Price     | httpRequest         | Scrape live Bitcoin price via API     | Manual Trigger          | Extract Price            | See Sticky Note "Section 1: Fetch the Current Bitcoin Price"                                                 |
| Extract Price           | html                | Extract price from HTML                | Fetch Bitcoin Price     | Convert type             | See Sticky Note "Section 1: Fetch the Current Bitcoin Price"                                                 |
| Convert type            | code                | Convert price string to number        | Extract Price           | Get Last Recorded Price  | See Sticky Note "Section 2: Calculate and Compare with Historical Price"                                    |
| Get Last Recorded Price | googleSheets        | Fetch previous price from Google Sheet| Convert type            | Find drop percentage     | See Sticky Note "Section 2: Calculate and Compare with Historical Price"                                    |
| Find drop percentage    | code                | Calculate % drop between prices       | Get Last Recorded Price | If droppercentage >= 5   | See Sticky Note "Section 2: Calculate and Compare with Historical Price"                                    |
| If droppercentage >= 5  | if                  | Check if drop ≥ 5%                    | Find drop percentage    | Send Alert Email / No Alert Needed | See Sticky Note "Section 3: Send Alert if Drop ≥ 5%"                                                        |
| Send Alert Email        | gmail               | Send notification email on drop       | If droppercentage >= 5  | —                        | See Sticky Note "Section 3: Send Alert if Drop ≥ 5%"; Also commission link in Sticky Note                      |
| No Alert Needed         | noOp                | End workflow silently if no alert     | If droppercentage >= 5  | —                        | See Sticky Note "Section 3: Send Alert if Drop ≥ 5%"                                                          |
| Sticky Note9            | stickyNote          | Workflow assistance contact and links | —                      | —                        | Contains contact info and YouTube, LinkedIn links                                                            |
| Sticky Note3            | stickyNote          | Full workflow description and benefits| —                      | —                        | Detailed workflow explanation with sections                                                                 |
| Sticky Note             | stickyNote          | Section 1 description                  | —                      | —                        | Details Section 1 nodes and logic                                                                            |
| Sticky Note1            | stickyNote          | Section 2 description                  | —                      | —                        | Details Section 2 nodes and logic                                                                            |
| Sticky Note2            | stickyNote          | Section 3 description                  | —                      | —                        | Details Section 3 nodes and logic                                                                            |
| Sticky Note4            | stickyNote          | Affiliate commission link for Bright Data | —                  | —                        | Contains affiliate link: https://get.brightdata.com/1tndi4600b25                                              |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - No parameters needed.  
   - This node will start the workflow manually.

2. **Add an HTTP Request node named "Fetch Bitcoin Price"**  
   - Set method to POST.  
   - URL: `https://api.brightdata.com/request`.  
   - Body parameters (send as JSON):  
     - `zone`: `n8n_unblocker`  
     - `url`: `https://coinmarketcap.com/currencies/bitcoin/`  
     - `country`: `us`  
     - `format`: `raw`  
   - Headers:  
     - `Authorization` with value `Bearer API_KEY` (replace `API_KEY` with your Bright Data API token).  
   - Connect output from "Manual Trigger" to "Fetch Bitcoin Price".

3. **Add an HTML Extract node named "Extract Price"**  
   - Operation: Extract HTML content.  
   - Add extraction value:  
     - Key: `price`  
     - CSS Selector: `span.sc-65e7f566-0.esyGGG.base-text[data-test="text-cdp-price-display"]`  
   - Connect output from "Fetch Bitcoin Price" to "Extract Price".

4. **Add a Code node named "Convert type"**  
   - JavaScript code:  
   ```javascript
   return items.map(item => {
     const priceStr = item.json.price;
     const priceNumber = parseFloat(priceStr.replace(/[$,]/g, ''));
     return { json: { price: priceNumber } };
   });
   ```  
   - Connect output from "Extract Price" to "Convert type".

5. **Add a Google Sheets node named "Get Last Recorded Price"**  
   - Authentication: Use Google Sheets OAuth2 credentials with access to your target spreadsheet.  
   - Document ID: Enter your Google Sheet ID that stores previous Bitcoin prices.  
   - Sheet Name: Use `gid=0` or the actual sheet name containing price data.  
   - Configure node to read the last recorded price cell or row, ensure the output JSON contains `Price` key with numeric value.  
   - Connect output from "Convert type" to "Get Last Recorded Price".

6. **Add a Code node named "Find drop percentage"**  
   - JavaScript code:  
   ```javascript
   const currentPrice = $('Convert type').first().json.price;
   const previousPrice = $input.first().json.Price;

   const dropPercent = previousPrice !== 0 
     ? ((previousPrice - currentPrice) / previousPrice) * 100 
     : 0;

   return [{
     json: {
       currentPrice,
       previousPrice,
       dropPercent: parseFloat(dropPercent.toFixed(2))
     }
   }];
   ```  
   - Connect output from "Get Last Recorded Price" to "Find drop percentage".

7. **Add an If node named "If droppercentage >= 5"**  
   - Condition: Numeric check where  
     - Left value: `{{$json.dropPercent}}`  
     - Operation: Greater than or equal to  
     - Right value: `5`  
   - Connect output from "Find drop percentage" to this If node.

8. **Add a Gmail node named "Send Alert Email"**  
   - Authentication: Use Gmail OAuth2 credentials.  
   - To: Enter recipient email, e.g., `shahkar.genai@gmail.com`.  
   - Subject: `Bitcoin Price has dropped {{$json.dropPercent}}%`.  
   - Message (plain text):  
   ```
   Hello,

   The price of Bitcoin has dropped by {{$json.dropPercent}}% since the last check.

   Previous Price: {{$json.previousPrice}}
   Current Price: {{$json.currentPrice}}

   Please review accordingly.

   Best regards,
   Your Crypto Monitor Bot
   ```  
   - Connect the *true* output of the If node to "Send Alert Email".

9. **Add a NoOp node named "No Alert Needed"**  
   - No parameters; used to end workflow silently.  
   - Connect the *false* output of the If node to "No Alert Needed".

10. **Optional: Add Sticky Notes for documentation**  
    - Add notes describing each section and instructions, including contact info and helpful links.  
    - Add an affiliate link sticky note with Bright Data referral URL if desired.

11. **Save and activate the workflow** (consider adding a Cron Trigger for scheduled runs).

---

## 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow assistance contact: Yaron@nofluff.online                                                             | Sticky Note9 in workflow                                                                        |
| YouTube channel for tips and tutorials: https://www.youtube.com/@YaronBeen/videos                              | Sticky Note9                                                                                   |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                      | Sticky Note9                                                                                   |
| Bright Data official website and API for scraping: https://brightdata.com/                                     | Referenced in Section 1 Sticky Note                                                             |
| Affiliate commission link for Bright Data: https://get.brightdata.com/1tndi4600b25                             | Sticky Note4                                                                                   |
| Workflow handles only manual trigger but can be extended with Cron Trigger for periodic checks                 | Recommended enhancement                                                                        |
| Price extraction relies on webpage structure, which may change; selector updates might be needed periodically   | Important maintenance note                                                                     |
| Google Sheets OAuth2 credentials need proper scopes to read the price data                                     | Authentication setup must be verified                                                          |
| Gmail OAuth2 credentials must have send email permission and handle quota limits                               | Possible failure mode during sending alerts                                                    |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---