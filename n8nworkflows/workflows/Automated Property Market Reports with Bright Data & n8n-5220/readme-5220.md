Automated Property Market Reports with Bright Data & n8n

https://n8nworkflows.xyz/workflows/automated-property-market-reports-with-bright-data---n8n-5220


# Automated Property Market Reports with Bright Data & n8n

---

### 1. Workflow Overview

This workflow automates the weekly scraping and reporting of commercial real estate property listings for a specified location using Bright Data’s Web Unlocker API integrated into n8n. It is designed to reliably extract property details from websites protected by anti-bot measures and store the structured data in Google Sheets for further analysis or reporting.

**Target Use Cases:**

- Real estate market analysts tracking commercial properties weekly
- Agencies or investors needing automated data collection without manual intervention
- Teams wanting clean, structured property data for dashboards or client reports

**Logical Blocks:**

- **1.1 Automation Trigger:** Schedules the workflow to run automatically every Monday at 9 AM.
- **1.2 Data Acquisition via Bright Data:** Sends a POST request to Bright Data’s API to fetch raw HTML of commercial property listings.
- **1.3 Data Extraction:** Parses the received HTML to extract key property details using CSS selectors.
- **1.4 Data Storage:** Appends the extracted structured data directly into a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Automation Trigger

- **Overview:**  
  Initiates the workflow on a fixed weekly schedule, ensuring automated execution without manual input.

- **Nodes Involved:**  
  - `Weekly Market Trigger`

- **Node Details:**

  - **Node Name:** Weekly Market Trigger  
  - **Type:** Schedule Trigger  
  - **Role:** Triggers the workflow every week on Monday at 9:00 AM.  
  - **Configuration:**  
    - Interval set to weeks  
    - Trigger day: Monday (day 1)  
    - Trigger hour: 9 AM  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connects to `Fetch Listings via Bright Data` node  
  - **Version-specific requirements:** Version 1.2 or above to support weekly triggers with day and hour settings  
  - **Potential Failures / Edge Cases:**  
    - Incorrect time zone handling may cause unexpected trigger times  
    - Workflow disabled or credentials missing could prevent execution  
  - **Sub-workflow:** None

#### 1.2 Data Acquisition via Bright Data

- **Overview:**  
  Sends a POST request to Bright Data’s Web Unlocker API to retrieve the fully rendered HTML content of a commercial real estate listings page, bypassing bot detection.

- **Nodes Involved:**  
  - `Fetch Listings via Bright Data`

- **Node Details:**

  - **Node Name:** Fetch Listings via Bright Data  
  - **Type:** HTTP Request  
  - **Role:** Requests HTML content from a target property listing URL via Bright Data’s API.  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://api.brightdata.com/request`  
    - Body Parameters:  
      - `zone`: `"n8n_unblocker"` (Bright Data zone for web unlocking)  
      - `url`: `"https://www.crexi.com/properties?geo=san-francisco-ca"` (target commercial listings page)  
      - `country`: `"us"` (geolocation setting)  
      - `format`: `"raw"` (request raw HTML)  
    - Headers: Authorization with Bearer token (`API_KEY` placeholder)  
    - Sends body and headers with the request  
  - **Inputs:** Connected from `Weekly Market Trigger`  
  - **Outputs:** Connects to `Extract Listing Details (HTML)`  
  - **Version-specific requirements:** HTTP Request node Version 4.2 or above recommended for robust POST handling  
  - **Potential Failures / Edge Cases:**  
    - Invalid or expired API key causing authorization errors  
    - Bright Data zone misconfiguration resulting in failed unlocking  
    - Network timeouts or API rate limits  
    - Changes in target website structure or URL could yield invalid HTML  
  - **Sub-workflow:** None

#### 1.3 Data Extraction

- **Overview:**  
  Parses the HTML response from Bright Data to extract key commercial property details using CSS selectors, converting unstructured HTML into structured data.

- **Nodes Involved:**  
  - `Extract Listing Details (HTML)`

- **Node Details:**

  - **Node Name:** Extract Listing Details (HTML)  
  - **Type:** HTML Extract  
  - **Role:** Extracts property information fields such as price from the HTML content using configured CSS selectors.  
  - **Configuration:**  
    - Operation: extractHtmlContent  
    - Extraction Values:  
      - Key: `"Price"`  
      - CSS Selector: `"<span _ngcontent-ng-c2107907900=\"\" data-cy=\"propertyPrice\" class=\"ng-star-inserted\">$3,995,000</span>"` (note: this appears to be a direct HTML snippet rather than a selector string, which may be an error or placeholder)  
      - Return Array: true (extract multiple entries)  
  - **Inputs:** Connected from `Fetch Listings via Bright Data`  
  - **Outputs:** Connects to `Save to Google Sheets`  
  - **Version-specific requirements:** Node version 1.2 or above to support HTML content extraction operation  
  - **Potential Failures / Edge Cases:**  
    - CSS selector syntax errors or invalid selectors causing extraction failure  
    - Changes in target website HTML structure leading to missing data  
    - Empty or malformed HTML input from previous node  
  - **Sub-workflow:** None

#### 1.4 Data Storage

- **Overview:**  
  Appends the extracted property listing data into a specified Google Sheets document for easy access and further analysis.

- **Nodes Involved:**  
  - `Save to Google Sheets`

- **Node Details:**

  - **Node Name:** Save to Google Sheets  
  - **Type:** Google Sheets  
  - **Role:** Appends rows of extracted data into a Google Sheet document.  
  - **Configuration:**  
    - Operation: append  
    - Sheet Name: dynamically configured (parameters mode "list", actual sheet name value not specified in the JSON)  
    - Document ID: dynamically configured (parameters mode "list", actual document ID value not specified)  
    - Credentials: Google Sheets OAuth2 (configured with the credential named `Google Sheets account`)  
  - **Inputs:** Connected from `Extract Listing Details (HTML)`  
  - **Outputs:** None (terminal node)  
  - **Version-specific requirements:** Node version 4.6 or above to support append operation with OAuth2 credentials  
  - **Potential Failures / Edge Cases:**  
    - Missing or invalid Google Sheets credentials causing authorization errors  
    - Incorrect document ID or sheet name causing append failures  
    - API rate limits or Google Sheets API outages  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                         | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                                   |
|----------------------------|--------------------|---------------------------------------|----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| Weekly Market Trigger       | Schedule Trigger   | Initiates workflow weekly on Monday 9 AM | -                          | Fetch Listings via Bright Data | See Sticky Note3: Explains weekly trigger automation benefits                                                 |
| Fetch Listings via Bright Data | HTTP Request      | Fetches HTML data via Bright Data API| Weekly Market Trigger       | Extract Listing Details (HTML) | See Sticky Note3: Details on Bright Data unlocking and fetching listings                                      |
| Extract Listing Details (HTML) | HTML Extract      | Extracts property details from HTML   | Fetch Listings via Bright Data | Save to Google Sheets         | See Sticky Note2: Details on extraction of key fields like price, address, etc.                              |
| Save to Google Sheets       | Google Sheets      | Appends extracted data to Google Sheet| Extract Listing Details (HTML) | -                            | See Sticky Note2: Explains appending data to Google Sheets for reporting                                      |
| Sticky Note9                | Sticky Note        | Workflow assistance contact and links | -                          | -                            | Provides contact info and links for support and further tutorials                                            |
| Sticky Note4                | Sticky Note        | Workflow full description and benefits| -                          | -                            | Contains detailed workflow documentation and user benefits                                                   |
| Sticky Note3                | Sticky Note        | Section 1 notes for automation and scraping | -                          | -                            | Explains Section 1 nodes and their benefits                                                                  |
| Sticky Note                 | Sticky Note        | Section 2 notes on extraction and saving | -                          | -                            | Explains Section 2 nodes and their benefits                                                                  |
| Sticky Note5                | Sticky Note        | Bright Data affiliate link             | -                          | -                            | Affiliate link with disclaimer                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `Weekly Market Trigger`  
   - Type: Schedule Trigger  
   - Set interval to weekly, trigger day to Monday (1), trigger hour to 9 AM  
   - No credentials needed  
   - Connect this as the start node

2. **Add an HTTP Request Node**  
   - Name: `Fetch Listings via Bright Data`  
   - Type: HTTP Request  
   - Set method to POST  
   - URL: `https://api.brightdata.com/request`  
   - Under Body Parameters, add:  
     - `zone` = `n8n_unblocker`  
     - `url` = `https://www.crexi.com/properties?geo=san-francisco-ca`  
     - `country` = `us`  
     - `format` = `raw`  
   - Under Headers, add:  
     - `Authorization` = `Bearer API_KEY` (replace `API_KEY` with your actual Bright Data API token)  
   - Enable sending body and headers  
   - Connect output of `Weekly Market Trigger` to this node

3. **Add an HTML Extract Node**  
   - Name: `Extract Listing Details (HTML)`  
   - Type: HTML Extract  
   - Set operation to `extractHtmlContent`  
   - Configure extraction values:  
     - Key: `Price`  
     - CSS Selector: Use appropriate selector for price, e.g. `.property-price` or the actual selector matching your target page (note: adjust from placeholder)  
     - Set `Return Array` to true to capture multiple listings  
   - Connect output of `Fetch Listings via Bright Data` to this node

4. **Add a Google Sheets Node**  
   - Name: `Save to Google Sheets`  
   - Type: Google Sheets  
   - Set operation to `append`  
   - Enter your Google Sheets `Document ID` and `Sheet Name` where data should be appended  
   - Select or create OAuth2 credentials for Google Sheets access  
   - Connect output of `Extract Listing Details (HTML)` to this node

5. **Workflow Connections:**  
   - Connect `Weekly Market Trigger` → `Fetch Listings via Bright Data` → `Extract Listing Details (HTML)` → `Save to Google Sheets`

6. **Credential Setup:**  
   - Configure Bright Data API key as bearer token in HTTP Request node headers  
   - Configure Google Sheets OAuth2 credentials in Google Sheets node

7. **Testing & Validation:**  
   - Run the workflow manually once to verify the HTML fetch and extraction  
   - Inspect Google Sheets to confirm data appending  
   - Adjust CSS selectors in the HTML Extract node as needed to match actual site structure

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow assistance contact: Yaron@nofluff.online                                             | Contact email provided in Sticky Note9 for support                                                  |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos             | Video resources for n8n and workflow assistance                                                     |
| LinkedIn profile for workflow author: https://www.linkedin.com/in/yaronbeen/                   | Professional profile link for further contact or info                                               |
| Bright Data affiliate link with commission disclaimer: https://get.brightdata.com/1tndi4600b25 | Included in Sticky Note5 for users interested in Bright Data services                                |
| Workflow automates commercial real estate weekly report generation from scraping to Google Sheets | Described in Sticky Note4 with user benefits and suggestions for workflow improvements               |

---

**Disclaimer:**  
The provided text and workflow are generated solely from an n8n automation workflow and comply fully with content policies. All data processed is legal and public. No illegal or offensive content is included.

---