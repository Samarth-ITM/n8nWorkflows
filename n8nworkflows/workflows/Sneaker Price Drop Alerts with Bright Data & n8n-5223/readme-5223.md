Sneaker Price Drop Alerts with Bright Data & n8n

https://n8nworkflows.xyz/workflows/sneaker-price-drop-alerts-with-bright-data---n8n-5223


# Sneaker Price Drop Alerts with Bright Data & n8n

### 1. Workflow Overview

This workflow automates the monitoring of sneaker prices on resale websites (specifically StockX) using Bright Data’s proxy service to bypass anti-bot protections. It runs on a daily schedule, scrapes the sneaker listings’ HTML, extracts sneaker names and prices, and sends an email alert if any sneaker’s price falls below a specified threshold.

The logic is organized into three main blocks:

- **1.1 Scheduled Price Scraping via Bright Data**  
  Automatically triggers the workflow every day and fetches sneaker webpage HTML using Bright Data’s Web Unlocker proxy service.

- **1.2 Extract Sneaker Info from HTML**  
  Parses the raw HTML content to extract sneaker titles and prices, then processes and converts price strings into numeric values.

- **1.3 Check for Deals & Send Alerts**  
  Compares extracted prices against a user-defined threshold and sends email alerts for price drops under that threshold; otherwise, it ends the workflow quietly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Price Scraping via Bright Data

- **Overview:**  
  This block sets a daily schedule to automatically run the price monitoring and uses Bright Data’s API to scrape the sneaker listings page without IP blocks or captchas.

- **Nodes Involved:**  
  - Schedule Trigger — “Run Price Monitor Everyday”  
  - HTTP Request — “HTTP RequestTrigger Bright Data Scraper”

- **Node Details:**

  - **Schedule Trigger — “Run Price Monitor Everyday”**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow daily at 09:00 AM.  
    - Configuration: Set to trigger once every day at hour 9 (9 AM).  
    - Inputs: None  
    - Outputs: Connects to HTTP Request node.  
    - Edge Cases: Ensure n8n server time matches expected timezone to avoid misfires.  
    - Version: 1.2

  - **HTTP Request — “HTTP RequestTrigger Bright Data Scraper”**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Bright Data API to fetch sneaker listings HTML.  
    - Configuration:  
      - URL: https://api.brightdata.com/request  
      - Method: POST  
      - Body Parameters: zone=n8n_unblocker, url=https://stockx.com/category/sneakers, country=us, format=raw  
      - Headers: Authorization Bearer token (requires user’s Bright Data API Key)  
    - Inputs: Receives trigger from Schedule Trigger.  
    - Outputs: Passes raw HTML response to HTML Extractor node.  
    - Edge Cases:  
      - API key misconfiguration or expired token could cause authorization errors.  
      - Network timeouts or rate limiting from Bright Data API.  
      - Changes in the target URL or page structure may return unexpected HTML.  
    - Version: 4.2

---

#### 1.2 Extract Sneaker Info from HTML

- **Overview:**  
  Converts raw HTML into structured data, extracting sneaker titles and prices with CSS selectors, then cleans and transforms extracted price strings into numeric values.

- **Nodes Involved:**  
  - HTML Extractor — “HTML Extractor”  
  - Function — “Extract Title & Price”

- **Node Details:**

  - **HTML Extractor — “HTML Extractor”**  
    - Type: HTML Node  
    - Role: Parses raw HTML content using CSS selectors to extract arrays of sneaker titles and prices.  
    - Configuration:  
      - Extraction operation: extractHtmlContent  
      - Extraction Values:  
        - Title: CSS selector `#product-results > div > div > a > div > div.css-cp13gg > div.css-0 > p` (returns array)  
        - Price: CSS selector `#product-results > div > div > a > div > div.css-cp13gg > div.css-1b6n4o1 > div > p` (returns array)  
    - Inputs: Receives raw HTML from HTTP Request node.  
    - Outputs: JSON object with arrays of Titles and Prices for function node.  
    - Edge Cases:  
      - CSS selectors may break if StockX changes page structure.  
      - Empty or malformed HTML could result in empty arrays.  
    - Version: 1.2

  - **Function — “Extract Title & Price”**  
    - Type: Code (JavaScript)  
    - Role: Iterates over extracted arrays, cleans price strings (removes currency symbols), converts to float, and outputs structured JSON objects with `title` and `price` properties.  
    - Configuration:  
      - Script parses `items[0].json.Title` & `items[0].json.Price` arrays.  
      - Uses regex to strip non-numeric characters from price strings.  
    - Inputs: Receives JSON with Title and Price arrays from HTML Extractor.  
    - Outputs: Array of objects, each with `title` (string) and `price` (number).  
    - Edge Cases:  
      - Price strings not matching expected format might cause NaN results.  
      - Arrays of different lengths could cause index errors (assumes equal lengths).  
    - Version: 2

---

#### 1.3 Check for Deals & Send Alerts

- **Overview:**  
  Evaluates if the sneaker price is below a set threshold and triggers an email alert with details; otherwise, does nothing.

- **Nodes Involved:**  
  - IF Node — “Check Price < Threshold1”  
  - Gmail Node — “Send Price Drop Email Alert1”  
  - No Operation — “No Operation, do nothing”

- **Node Details:**

  - **IF Node — “Check Price < Threshold1”**  
    - Type: IF Node  
    - Role: Compares sneaker price against a threshold value (default $200).  
    - Configuration:  
      - Condition: `$json.price < 200`  
      - Comparison: Numeric less than  
      - Case sensitive: true  
      - Type validation: strict  
    - Inputs: Array of sneaker objects from Function node.  
    - Outputs:  
      - True branch: sneakers priced below threshold.  
      - False branch: sneakers priced above or equal threshold.  
    - Edge Cases:  
      - Missing `price` field or non-numeric values may cause failure.  
    - Version: 2.2

  - **Gmail Node — “Send Price Drop Email Alert1”**  
    - Type: Gmail  
    - Role: Sends email notifications containing sneaker name, price, and StockX link if price is low enough.  
    - Configuration:  
      - Recipient: shahkar.genai@gmail.com (changeable)  
      - Subject: “The sneaker {{ $json.title }} has dropped below your threshold price.”  
      - Message Body: multi-line text with sneaker name, price, and a StockX category link  
      - Email Type: plain text  
      - Credentials: Gmail OAuth2 (user must configure their Gmail account OAuth2 credentials)  
    - Inputs: True output from IF node.  
    - Outputs: None (terminal node).  
    - Edge Cases:  
      - Gmail API quota limits or auth errors.  
      - Invalid recipient email address.  
      - Network issues.  
    - Version: 2.1

  - **No Operation — “No Operation, do nothing”**  
    - Type: No Operation (NoOp)  
    - Role: Terminates workflow without action if price not below threshold.  
    - Configuration: None  
    - Inputs: False output from IF node.  
    - Outputs: None  
    - Edge Cases: None  
    - Version: 1

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                              | Input Node(s)                    | Output Node(s)                    | Sticky Note                                                                                                                  |
|----------------------------------|---------------------|----------------------------------------------|---------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Schedule TriggerRun Price Monitor Everyday | Schedule Trigger    | Triggers workflow daily at 9 AM               | None                            | HTTP RequestTrigger Bright Data Scraper | Section 1: Scheduled Price Scraping via Bright Data                                                                          |
| HTTP RequestTrigger Bright Data Scraper | HTTP Request       | Sends POST to Bright Data API to scrape HTML  | Schedule TriggerRun Price Monitor Everyday | HTML Extractor                   | Section 1: Scheduled Price Scraping via Bright Data                                                                          |
| HTML Extractor                   | HTML Extractor       | Extracts sneaker titles and prices from HTML  | HTTP RequestTrigger Bright Data Scraper | Extract Title & Price             | Section 2: Extract Sneaker Info from HTML                                                                                    |
| Extract Title & Price            | Function (Code)      | Cleans and converts extracted titles and prices | HTML Extractor                 | Check Price < Threshold1          | Section 2: Extract Sneaker Info from HTML                                                                                    |
| Check Price < Threshold1         | IF Node              | Checks if price is below threshold (200 USD)  | Extract Title & Price           | Send Price Drop Email Alert1, No Operation  | Section 3: Check for Deals & Send Alerts                                                                                     |
| Send Price Drop Email Alert1     | Gmail                | Sends email alert for sneaker price drop       | Check Price < Threshold1 (true) | None                           | Section 3: Check for Deals & Send Alerts                                                                                     |
| No Operation, do nothing         | No Operation (NoOp)  | Ends workflow silently if price not low enough | Check Price < Threshold1 (false)| None                           | Section 3: Check for Deals & Send Alerts                                                                                     |
| Sticky Note9                    | Sticky Note          | Workflow assistance and contact info           | None                            | None                           | Contains contact info and tutorial links                                                                                     |
| Sticky Note4                    | Sticky Note          | Full workflow overview and detailed instructions | None                            | None                           | Contains detailed multi-section workflow explanation                                                                        |
| Sticky Note3                    | Sticky Note          | Section 1 summary                               | None                            | None                           | Section 1: Scheduled Price Scraping via Bright Data                                                                          |
| Sticky Note                     | Sticky Note          | Section 2 summary                               | None                            | None                           | Section 2: Extract Sneaker Info from HTML                                                                                    |
| Sticky Note1                   | Sticky Note          | Section 3 summary                               | None                            | None                           | Section 3: Check for Deals & Send Alerts                                                                                     |
| Sticky Note5                   | Sticky Note          | Affiliate link for Bright Data                  | None                            | None                           | Contains affiliate link: https://get.brightdata.com/1tndi4600b25                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Schedule TriggerRun Price Monitor Everyday`  
   - Type: Schedule Trigger  
   - Set to trigger daily at 09:00 (9 AM)  
   - Position: Leftmost in canvas

2. **Create an HTTP Request node**  
   - Name: `HTTP RequestTrigger Bright Data Scraper`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Body Parameters (as JSON or form-data):  
     - `zone`: `n8n_unblocker`  
     - `url`: `https://stockx.com/category/sneakers`  
     - `country`: `us`  
     - `format`: `raw`  
   - Header Parameters:  
     - `Authorization`: `Bearer <YOUR_BRIGHT_DATA_API_KEY>` (replace with your API key)  
   - Connect Schedule Trigger's output to this node’s input

3. **Create an HTML Extractor node**  
   - Name: `HTML Extractor`  
   - Type: HTML  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `Title`  
       - CSS Selector: `#product-results > div > div > a > div > div.css-cp13gg > div.css-0 > p`  
       - Return as array: true  
     - Key: `Price`  
       - CSS Selector: `#product-results > div > div > a > div > div.css-cp13gg > div.css-1b6n4o1 > div > p`  
       - Return as array: true  
   - Connect HTTP Request node’s output to this node’s input

4. **Create a Function node**  
   - Name: `Extract Title & Price`  
   - Type: Function (JavaScript)  
   - Paste this code:  
     ```javascript
     const titles = items[0].json.Title;
     const prices = items[0].json.Price;

     const output = [];

     for (let i = 0; i < titles.length; i++) {
       const title = titles[i];
       const priceStr = prices[i];
       const price = parseFloat(priceStr.replace(/[^0-9.]/g, ''));

       output.push({
         json: {
           title,
           price,
         }
       });
     }

     return output;
     ```  
   - Connect HTML Extractor output to this node’s input

5. **Create an IF node**  
   - Name: `Check Price < Threshold1`  
   - Type: IF  
   - Condition: Numeric comparison  
     - Expression: `{{$json.price}}` less than `200` (you can adjust threshold)  
   - Connect Function node output to IF node input

6. **Create a Gmail node**  
   - Name: `Send Price Drop Email Alert1`  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials for your account  
   - Send to: your email address (example: `shahkar.genai@gmail.com`)  
   - Subject: `The sneaker {{ $json.title }} has dropped below your threshold price.`  
   - Message (plain text):  
     ```
     Sneaker name: {{ $json.title }}
     Current Price: ${{ $json.price }}
     Link: https://stockx.com/category/sneakers
     ```  
   - Connect IF node’s `true` output to Gmail node input

7. **Create a No Operation node**  
   - Name: `No Operation, do nothing`  
   - Type: NoOp  
   - Connect IF node’s `false` output to No Operation node input

8. **Test the workflow**  
   - Activate and run manually or wait for scheduled trigger  
   - Verify emails are received only for sneakers priced below threshold

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Workflow assistance and contact for questions: Yaron@nofluff.online                                                  | Contact email                                           |
| YouTube tutorials and tips by Yaron Been: https://www.youtube.com/@YaronBeen/videos                                   | YouTube channel                                        |
| LinkedIn profile of workflow author: https://www.linkedin.com/in/yaronbeen/                                           | LinkedIn                                                |
| Affiliate link for Bright Data proxy service: https://get.brightdata.com/1tndi4600b25                                  | Supports workflow author                                |
| This workflow leverages Bright Data’s Web Unlocker to bypass scraping blocks on StockX and GOAT                       | Critical integration detail                             |
| Adjust the CSS selectors in the HTML Extractor node if StockX updates their website layout                             | Maintenance tip                                        |
| Gmail OAuth2 credentials must be correctly configured to avoid authentication errors                                  | Credential setup requirement                            |
| Price threshold is configurable in the IF node; change to suit your budget and alert preferences                      | Customization tip                                      |
| The workflow uses plain text emails for broad compatibility; consider HTML formatting if preferred                    | Enhancement suggestion                                 |

---

**Disclaimer:** The provided text is generated solely from an automated n8n workflow export. It fully complies with content policies and contains no illegal, offensive, or protected materials. All handled data is legal and publicly accessible.