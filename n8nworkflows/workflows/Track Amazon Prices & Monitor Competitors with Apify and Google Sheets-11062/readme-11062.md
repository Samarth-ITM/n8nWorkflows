Track Amazon Prices & Monitor Competitors with Apify and Google Sheets

https://n8nworkflows.xyz/workflows/track-amazon-prices---monitor-competitors-with-apify-and-google-sheets-11062


# Track Amazon Prices & Monitor Competitors with Apify and Google Sheets

### 1. Workflow Overview

This workflow automates daily tracking of Amazon product prices and competitor prices by using Apify's Amazon Product Scraper and Google Sheets for data storage and update. It is designed primarily for Amazon sellers, product managers, and growth teams who require automatic price monitoring and competitor comparison without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Auto-Start & Scheduling:** Daily automatic trigger to start the workflow at a fixed time.
- **1.2 Fetch Product Data:** Extract product and competitor URLs from a Google Sheet.
- **1.3 Loop Over Each Product Row:** Process each product entry one by one for scalability.
- **1.4 Scraping via Apify Actor:** Call Apify’s Amazon Product Scraper with both product and competitor URLs.
- **1.5 Retrieve Scraper Output:** Fetch the scraping results from Apify dataset.
- **1.6 Extract & Format Price Data:** Use JavaScript to parse and format price data from Apify output.
- **1.7 Update Google Sheet:** Append or update the product row with the latest prices in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Auto-Start & Scheduling

**Overview:**  
This block triggers the entire workflow automatically every day at 8 AM India Standard Time (Asia/Kolkata timezone), ensuring consistent daily updates without manual input.

**Nodes Involved:**  
- Schedule Trigger  
- Sticky Note (Auto-start workflow daily)

**Node Details:**  
- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Fires daily at 8:00 AM (hour-based trigger)  
  - Input: None (trigger node)  
  - Output: Triggers the next node to fetch data  
  - Edge Cases: Misconfiguration of timezone or trigger hour can cause missed runs.  
  - Version: 1.2  

- **Sticky Note**  
  - Content: "Auto-start workflow daily"  
  - Purpose: Documentation only  

---

#### 1.2 Fetch Product Data from Google Sheets

**Overview:**  
Fetch all rows containing product data and competitor URLs from a specific Google Sheet document. This data acts as input for subsequent scraping and processing steps.

**Nodes Involved:**  
- Get row(s) in sheet  
- Sticky Note (Fetch product row data)

**Node Details:**  
- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Configuration:  
    - Document ID: Spreadsheet ID for product data  
    - Sheet Name: "Sheet1" (gid=0)  
    - Operation: Get rows (default)  
  - Credentials: OAuth2 for Google Sheets with authorized user  
  - Input: Trigger from Schedule Trigger  
  - Output: All rows from the sheet, each representing a product with URLs  
  - Edge Cases:  
    - Sheet permission errors  
    - Empty sheet or invalid sheet ID  
    - API limits or OAuth token expiry  
  - Version: 4.7  

- **Sticky Note**  
  - Content: "Fetch product row data"  

---

#### 1.3 Loop Over Each Product Row

**Overview:**  
Process each row from the Google Sheet individually to handle multiple products and competitors in a controlled manner.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Sticky Note (Process each row one-by-one)

**Node Details:**  
- **Loop Over Items**  
  - Type: SplitInBatches  
  - Configuration: Default batch size (1 by default), processes one row at a time  
  - Input: Rows from Google Sheets node  
  - Output: Individual product rows to be processed further  
  - Edge Cases: Large sheets may cause delays; consider batch size tuning  
  - Version: 3  

- **Sticky Note**  
  - Content: "Process each row (one-by-one)"  

---

#### 1.4 Scraping via Apify Actor

**Overview:**  
For each product row, invoke Apify's Amazon Product Scraper actor with both the main product URL and competitor URLs to retrieve up-to-date pricing data.

**Nodes Involved:**  
- Run an Actor (Apify)  
- Sticky Note (Scrape Amazon data)

**Node Details:**  
- **Run an Actor**  
  - Type: Apify node (custom n8n node for Apify integration)  
  - Configuration:  
    - Actor ID: "BG3WDrGdteHgZgbPK" (Amazon Product Scraper)  
    - Input JSON: Constructs a "categoryOrProductUrls" array with two URLs from the current row’s fields: "Amazon S3P (Revlon)" and "Amazon P3P (RK)"  
  - Credentials: Apify API token (requires paid plan with Amazon Scraper)  
  - Input: One product row from Loop Over Items  
  - Output: Apify actor run result, including dataset ID for later fetching  
  - On Error: Continue workflow even if this node fails (important for robustness)  
  - Edge Cases:  
    - API rate limits or token expiry  
    - Invalid URLs or missing URLs in input  
    - Actor failures or timeouts  
  - Version: 1  

- **Sticky Note**  
  - Content: "Scrape Amazon data"  

---

#### 1.5 Retrieve Scraper Output

**Overview:**  
Once Apify actor completes, fetch the scraped data from the actor’s dataset via an HTTP Request node.

**Nodes Involved:**  
- HTTP Request1  
- Sticky Note (Fetch Apify output)

**Node Details:**  
- **HTTP Request1**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: Dynamically constructed using the dataset ID from Apify output and user’s Apify API token  
    - Method: GET  
  - Input: Output of the "Run an Actor" node (dataset ID)  
  - Output: JSON array of scraped items, including price data  
  - On Error: Continue workflow on failure to prevent breakage  
  - Edge Cases:  
    - Invalid token or expired token  
    - Dataset not ready yet or empty dataset  
    - Network timeouts or errors  
  - Version: 4.2  

- **Sticky Note**  
  - Content: "Fetch Apify output"  

---

#### 1.6 Extract & Format Price Data

**Overview:**  
Parse the scraped JSON data to extract relevant prices for the product and competitor, format them, and prepare data for updating the sheet.

**Nodes Involved:**  
- Code in JavaScript  
- Sticky Note (Extract price data)

**Node Details:**  
- **Code in JavaScript**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Script extracts prices from first two scraped items (main product and competitor)  
    - Uses fallback fields if price is missing  
    - Retrieves original product row data for context  
    - Formats output JSON including price values, category, material name, and timestamp (India timezone)  
  - Input: Array of scraped items from HTTP Request  
  - Output: JSON object with extracted prices and metadata  
  - On Error: Continue workflow to avoid stopping on parsing issues  
  - Edge Cases:  
    - Missing or malformed price fields  
    - Missing original row data if Loop Over Items context fails  
  - Version: 2  

- **Sticky Note**  
  - Content: "Extract price data"  

---

#### 1.7 Update Google Sheet

**Overview:**  
Append or update the corresponding product row in Google Sheets with the newly scraped and formatted price data.

**Nodes Involved:**  
- Append or update row in sheet (Google Sheets)  
- Sticky Note (Update data to sheet)

**Node Details:**  
- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Configuration:  
    - Operation: Append or update row based on matching "Material Name" column  
    - Column mapping: Updates "RK Price", "Revlon Price", and "Material Name" with computed values  
    - Document ID and Sheet Name same as input sheet  
  - Credentials: Same Google Sheets OAuth2 credentials as read node  
  - Input: Parsed JSON from JavaScript node  
  - Output: Confirmation of update operation  
  - Edge Cases:  
    - Matching row not found or multiple matches  
    - Permission or quota errors  
  - Version: 4.7  

- **Sticky Note**  
  - Content: "Update data to sheet"  

---

### 3. Summary Table

| Node Name                | Node Type                  | Functional Role                         | Input Node(s)         | Output Node(s)             | Sticky Note                        |
|--------------------------|----------------------------|---------------------------------------|-----------------------|----------------------------|----------------------------------|
| Schedule Trigger         | Schedule Trigger            | Triggers workflow daily at 8 AM       | None                  | Get row(s) in sheet         | Auto-start workflow daily         |
| Get row(s) in sheet      | Google Sheets              | Fetch product and competitor URLs     | Schedule Trigger       | Loop Over Items             | Fetch product row data            |
| Loop Over Items          | SplitInBatches             | Process each product row individually | Get row(s) in sheet    | Run an Actor, Loop Over Items | Process each row (one-by-one)    |
| Run an Actor             | Apify                      | Run Amazon Product Scraper on URLs    | Loop Over Items        | HTTP Request1               | Scrape Amazon data                |
| HTTP Request1            | HTTP Request               | Fetch scraped data from Apify dataset | Run an Actor           | Code in JavaScript          | Fetch Apify output                |
| Code in JavaScript       | Code                       | Extract and format prices              | HTTP Request1          | Append or update row in sheet | Extract price data               |
| Append or update row in sheet | Google Sheets          | Update sheet rows with new prices     | Code in JavaScript     | Loop Over Items             | Update data to sheet              |
| Sticky Note              | Sticky Note                | Documentation                         | None                  | None                       | Auto-start workflow daily         |
| Sticky Note1             | Sticky Note                | Documentation                         | None                  | None                       | Fetch product row data            |
| Sticky Note2             | Sticky Note                | Documentation                         | None                  | None                       | Process each row (one-by-one)     |
| Sticky Note3             | Sticky Note                | Documentation                         | None                  | None                       | Scrape Amazon data                |
| Sticky Note4             | Sticky Note                | Documentation                         | None                  | None                       | Fetch Apify output                |
| Sticky Note5             | Sticky Note                | Documentation                         | None                  | None                       | Extract price data                |
| Sticky Note6             | Sticky Note                | Documentation                         | None                  | None                       | Update data to sheet              |
| Sticky Note7             | Sticky Note                | Documentation                         | None                  | None                       | Amazon Price Tracker & Competitor Monitor (see details) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to trigger daily at 8:00 AM (hour-based trigger).  
   - Set timezone to Asia/Kolkata.

3. **Add a Google Sheets node (“Get row(s) in sheet”):**  
   - Operation: Get rows.  
   - Document ID: Enter your Google Sheet ID where product and competitor URLs are stored.  
   - Sheet Name: Use the sheet tab name or gid (e.g., "Sheet1" or gid=0).  
   - Connect this node to the Schedule Trigger.  
   - Set up Google Sheets OAuth2 credentials with access to the spreadsheet.

4. **Add a SplitInBatches node (“Loop Over Items”):**  
   - Default batch size (1) to process rows one-by-one.  
   - Connect to the Google Sheets node output.

5. **Add an Apify node (“Run an Actor”):**  
   - Select “Run Actor” operation.  
   - Actor ID: Use Apify’s Amazon Product Scraper actor ID (BG3WDrGdteHgZgbPK).  
   - Input Parameters:  
     ```json
     {
       "categoryOrProductUrls": [
         { "url": "{{ $json['Amazon S3P (Revlon)'] }}" },
         { "url": "{{ $json['Amazon P3P (RK)'] }}" }
       ]
     }
     ```  
   - Connect input to the SplitInBatches node output.  
   - Provide Apify API token credentials (requires paid plan).  
   - Set “On Error” to continue to prevent workflow stop if actor fails.

6. **Add HTTP Request node (“HTTP Request1”):**  
   - Method: GET  
   - URL:  
     ```
     https://api.apify.com/v2/datasets/{{ $json.defaultDatasetId }}/items?token={your_apify_api_YOUR_TOKEN_HERE}
     ```  
   - Replace `{your_apify_api_YOUR_TOKEN_HERE}` with your Apify API token.  
   - Connect input from Apify node output.  
   - Set “On Error” to continue for resilience.

7. **Add Code node (“Code in JavaScript”):**  
   - Paste the following JavaScript code to extract prices and format data:
     ```javascript
     // Get both product results
     const items = $input.all();

     // Extract prices from both products
     const price1 = items[0]?.json?.price?.value || items[0]?.json?.listPrice?.value || 'N/A';
     const price2 = items[1]?.json?.price?.value || items[1]?.json?.listPrice?.value || 'N/A';

     // Get the original row data from Loop Over Items
     const originalRow = $('Loop Over Items').item.json;

     return [{
       json: {
         rowNumber: originalRow.row_number,
         MyListingPrice: price1,
         OtherSellerPrice: price2,
         categoryName: originalRow['Category Name'],
         materialName: originalRow['Material Name'],
         lastUpdated: new Date().toLocaleString('en-IN', { timeZone: 'Asia/Kolkata' })
       }
     }];
     ```
   - Connect input from HTTP Request node output.  
   - Set “On Error” to continue.

8. **Add Google Sheets node (“Append or update row in sheet”):**  
   - Operation: Append or Update row.  
   - Document ID and Sheet Name: Same as the first Google Sheets node.  
   - Mapping Mode: Define columns below.  
   - Matching Columns: Use “Material Name” as key to match rows.  
   - Columns to update:  
     - RK Price ← `={{ $json.rkPrice }}`  
     - Revlon Price ← `={{ $json.revlonPrice }}`  
     - Material Name ← `={{ $json.materialName }}`  
   - Connect input from Code node output.  
   - Use same Google Sheets OAuth2 credentials.

9. **Connect the output of the Google Sheets update node back to the SplitInBatches node to continue processing the next row.**

10. **Add Sticky Notes for documentation:**  
    - Place them near appropriate nodes with descriptive comments such as:  
      - "Auto-start workflow daily" near Schedule Trigger.  
      - "Fetch product row data" near Google Sheets get node.  
      - "Process each row (one-by-one)" near SplitInBatches node.  
      - "Scrape Amazon data" near Apify node.  
      - "Fetch Apify output" near HTTP Request node.  
      - "Extract price data" near Code node.  
      - "Update data to sheet" near Google Sheets update node.  
      - Add a large sticky note summarizing workflow purpose and usage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow tracks Amazon product prices and competitor prices daily using Apify's Amazon Scraper and updates a Google Sheet, designed for sellers and growth teams. Supports unlimited competitors by adding more URL columns and extending the flow.                                                                                                                                  | Workflow Purpose                                                                                                                                                         |
| Requires Apify API token with paid plan ($40/month) to use Amazon Product Scraper.                                                                                                                                                                                                                                                                                              | https://apify.com/store/actors/junglee/amazon-crawler                                                                                                                  |
| Google Sheets OAuth2 credentials set with read/write access to the product data spreadsheet.                                                                                                                                                                                                                                                                                     | Credential Setup                                                                                                                                                         |
| To extend with more competitors, add extra columns in the Google Sheet, modify Apify input JSON, adjust JavaScript code to extract more price fields, and map those fields in the Google Sheets update node.                                                                                                                                                                       | Customization Instructions                                                                                                                                                |
| Consider adding alerts for price drops, Buy Box owner detection, or Slack/email notifications for enhanced monitoring.                                                                                                                                                                                                                                                           | Suggested Enhancements                                                                                                                                                   |
| Timezone for scheduling and timestamps is Asia/Kolkata (India Standard Time). Adjust if deploying in other regions.                                                                                                                                                                                                                                                              | Timezone Info                                                                                                                                                            |
| Workflow is resilient to some errors: Apify failures or HTTP request errors continue workflow to avoid total failure.                                                                                                                                                                                                                                                             | Error Handling                                                                                                                                                           |
| Contact for help: imarunavadas@gmail.com                                                                                                                                                                                                                                                                                                                                         | Support Contact                                                                                                                                                          |
| Detailed documentation sticky note inside workflow contains full project overview and setup instructions.                                                                                                                                                                                                                                                                          | Included inside the workflow as Sticky Note7                                                                                                                            |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. It complies fully with content policies and handles only legal, public data.