Find the Cheapest Flights Automatically with Bright Data & n8n

https://n8nworkflows.xyz/workflows/find-the-cheapest-flights-automatically-with-bright-data---n8n-5218


# Find the Cheapest Flights Automatically with Bright Data & n8n

---

### 1. Workflow Overview

This workflow automates the process of finding the cheapest flight prices from Skiplagged.com by leveraging Bright Data’s proxy service to bypass scraping protections, extracting relevant pricing data from the HTML content, and logging the prices into a Google Sheet for tracking and review.

**Target Use Cases:**  
- Travelers seeking real-time cheap flight deals.  
- Travel agencies or price monitoring services wanting automated price updates.  
- Users who want a no-code, repeatable workflow for scraping difficult web data and storing it efficiently.

**Logical Blocks:**

- **1.1 Input Reception:** Manual initiation of the workflow to trigger flight price scraping.  
- **1.2 Data Acquisition via Proxy:** HTTP Request node uses Bright Data’s Web Unlocker API to fetch the raw HTML from Skiplagged, circumventing anti-bot measures.  
- **1.3 Data Extraction:** HTML node parses the fetched HTML to extract flight price elements.  
- **1.4 Data Logging:** Appended extracted flight prices as rows into a configured Google Sheet for persistence and analysis.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block begins the workflow manually, providing a user-controlled start point without schedules or triggers.

- **Nodes Involved:**  
  - Manual Trigger

- **Node Details:**

  - **Manual Trigger**  
    - Type: `Manual Trigger` (n8n core node)  
    - Role: Initiates workflow execution on user command only.  
    - Configuration: Default (no parameters), runs when user clicks "Execute Workflow".  
    - Inputs: None  
    - Outputs: Triggers next node (`Fetch flight details from skiplegged via bright data`).  
    - Edge Cases: None significant; user must manually start workflow.  
    - Version: Compatible with all standard n8n versions.

---

#### Block 1.2: Data Acquisition via Proxy

- **Overview:**  
  Retrieves the raw HTML content of a flight search results page from Skiplagged.com by sending a POST request to Bright Data’s API, which proxies and unlocks the web page to bypass bot detection.

- **Nodes Involved:**  
  - Fetch flight details from skiplegged via bright data (HTTP Request)

- **Node Details:**

  - **Fetch flight details from skiplegged via bright data**  
    - Type: `HTTP Request`  
    - Role: Posts a request to Bright Data API to retrieve raw HTML from a target URL.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.brightdata.com/request`  
      - Body (JSON): Contains parameters:  
        - `zone`: `"n8n_unblocker"` (Bright Data zone to use)  
        - `url`: Target Skiplagged flight search URL (e.g., `https://skiplagged.com/flights/NYC/LAX/2024-07-01`)  
        - `country`: `"ie"` (Ireland, proxy exit location)  
        - `format`: `"raw"` (returns the raw HTML)  
        - `headers`: User-Agent and Accept headers mimicking a real browser.  
      - Authentication: Bearer token in headers (`Authorization: Bearer API_KEY`) requiring Bright Data API key credentials.  
    - Inputs: Triggered by Manual Trigger node.  
    - Outputs: Raw HTML content of the Skiplagged search results page.  
    - Edge Cases / Failure Modes:  
      - Invalid or expired Bright Data API key causing 401 Unauthorized.  
      - Rate limiting from Bright Data if quota exceeded.  
      - Network timeouts or proxy errors.  
      - Skiplagged webpage structure changes (may not affect here but downstream parsing impacted).  
      - Incorrect zone or country parameters causing unexpected results.  
    - Version: Requires n8n version supporting HTTP Request node with JSON body and header configuration (v0.130+ recommended).  

---

#### Block 1.3: Data Extraction

- **Overview:**  
  Parses the raw HTML content to extract all flight price values by CSS selector targeting the flight price elements.

- **Nodes Involved:**  
  - HTML

- **Node Details:**

  - **HTML**  
    - Type: `HTML Extract` (HTML node)  
    - Role: Extracts desired data fields from the raw HTML using CSS selectors.  
    - Configuration:  
      - Operation: `extractHtmlContent`  
      - Extraction Values:  
        - Key: `"Title"` with CSS selector targeting flight airline name span (example given but likely static).  
        - Key: `"Price"` with CSS selector targeting `<div class="flights-landing__flight-price">` elements (main target).  
      - Multiple matches enabled to extract all flight prices on the page.  
    - Inputs: Raw HTML from Bright Data HTTP Request node.  
    - Outputs: JSON array with flight prices as string values (e.g., `["$104", "$117", "$99", "$88"]`).  
    - Edge Cases / Failure Modes:  
      - Skiplagged changing HTML structure or CSS classes will break extraction.  
      - Empty or malformed HTML input from failed HTTP request.  
      - Selector may return no matches if page content changes or is blocked.  
    - Version: Requires n8n version supporting HTML node with multi-value extraction (v0.150+ recommended).

---

#### Block 1.4: Data Logging

- **Overview:**  
  Appends each extracted flight price as a new row into a Google Sheets document for easy tracking and historical record.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**

  - **Google Sheets**  
    - Type: `Google Sheets` (Append operation)  
    - Role: Logs each extracted flight price into a designated Google Sheet.  
    - Configuration:  
      - Operation: `append`  
      - Document ID and Sheet Name: Configured to point to the target Google Sheet and worksheet.  
      - Mapping: Maps extracted `Price` values into the corresponding column.  
      - Credentials: Google Sheets OAuth2 credential configured for access.  
    - Inputs: JSON array of prices from HTML node.  
    - Outputs: Confirmation of appended rows.  
    - Edge Cases / Failure Modes:  
      - Missing or invalid Google Sheets credentials causing auth failure.  
      - Incorrect Document ID or Sheet name causing write failure.  
      - Field mapping mismatch resulting in empty or misplaced data.  
      - API rate limiting from Google Sheets.  
    - Version: Requires n8n version supporting Google Sheets OAuth2 integration (v0.140+ recommended).

---

### 3. Summary Table

| Node Name                                   | Node Type            | Functional Role                      | Input Node(s)                             | Output Node(s)                           | Sticky Note                                                                                           |
|---------------------------------------------|----------------------|------------------------------------|------------------------------------------|-----------------------------------------|-----------------------------------------------------------------------------------------------------|
| Manual Trigger                              | Manual Trigger       | Starts workflow manually           | None                                     | Fetch flight details from skiplegged via bright data | Section 1 explanation: manual start, no schedule needed                                              |
| Fetch flight details from skiplegged via bright data | HTTP Request         | Fetches raw HTML via Bright Data proxy | Manual Trigger                          | HTML                                    | Section 1 explanation: uses Bright Data to bypass bot protection and fetch Skiplagged HTML           |
| HTML                                        | HTML Extract         | Extracts flight prices from HTML   | Fetch flight details from skiplegged via bright data | Google Sheets                           | Section 2 explanation: extracts prices using CSS selectors                                          |
| Google Sheets                               | Google Sheets Append | Appends extracted prices to Sheet | HTML                                     | None                                    | Section 2 explanation: logs prices to Google Sheet, watch for credentials & mapping errors           |
| Sticky Note9                                | Sticky Note          | Workflow assistance info           | None                                     | None                                    | Contact info and tutorial links: YouTube and LinkedIn channels                                      |
| Sticky Note4                                | Sticky Note          | Workflow description and overview  | None                                     | None                                    | Detailed workflow description and tips                                                             |
| Sticky Note5                                | Sticky Note          | Section 1 detailed explanation     | None                                     | None                                    | Description of manual trigger and HTTP Request node                                                |
| Sticky Note6                                | Sticky Note          | Section 2 detailed explanation     | None                                     | None                                    | Explanation of HTML extraction and Google Sheets append                                            |
| Sticky Note                                 | Sticky Note          | Bright Data affiliate link         | None                                     | None                                    | Affiliate link and thank you note                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: `Manual Trigger`  
   - No parameters required. This node will start the workflow manually.

2. **Create HTTP Request Node (Fetch flight details from skiplegged via bright data)**  
   - Node Type: `HTTP Request`  
   - Set Method to `POST`.  
   - Set URL to `https://api.brightdata.com/request`.  
   - Under "Body Parameters", specify JSON body:  
     ```json
     {
       "zone": "n8n_unblocker",
       "url": "https://skiplagged.com/flights/NYC/LAX/2024-07-01",
       "country": "ie",
       "format": "raw",
       "headers": {
         "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/113.0.0.0 Safari/537.36",
         "Accept-Language": "en-US,en;q=0.9",
         "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8"
       }
     }
     ```  
   - Enable "Send Body" and "Send Headers".  
   - Under Headers, add an `Authorization` header with value `Bearer API_KEY` (replace `API_KEY` with your Bright Data API key).  
   - Connect Manual Trigger's output to this HTTP Request node input.

3. **Create HTML Extract Node**  
   - Node Type: `HTML`  
   - Operation: `extractHtmlContent`  
   - Configure extraction values as:  
     - Key: `Price`  
     - CSS Selector: `.flights-landing__flight-price`  
   - Enable "Return All Matches" to get all prices.  
   - Connect the output of HTTP Request node to this HTML node.

4. **Create Google Sheets Append Node**  
   - Node Type: `Google Sheets`  
   - Operation: `append`  
   - Configure:  
     - Document ID: ID of your target Google Sheet (copy from Google Sheets URL).  
     - Sheet Name: Name of the worksheet/tab where prices will be appended.  
     - Map the incoming `Price` value to the correct column (ensure your sheet has a column header like "Price").  
   - Set up Google Sheets OAuth2 credentials in n8n beforehand and select the credential here.  
   - Connect the output of the HTML node to Google Sheets node.

5. **Final Connections and Testing**  
   - Ensure the nodes are connected in order:  
     Manual Trigger → HTTP Request → HTML Extract → Google Sheets Append.  
   - Save the workflow.  
   - Execute manually via the Manual Trigger node.  
   - Confirm that flight prices are appended to your Google Sheet.

6. **Optional Enhancements**  
   - Replace static URLs and dates in HTTP Request body with dynamic parameters or Set nodes to make origin, destination, and date configurable.  
   - Add error handling via IF nodes or error workflow to catch failed HTTP requests or extraction issues.  
   - Schedule workflow using Cron node to automate recurring checks.  
   - Add notification nodes (Telegram, Email) to alert on price drops.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                             |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| For any workflow questions or support, contact: Yaron@nofluff.online. YouTube tutorials at https://www.youtube.com/@YaronBeen/videos | Workflow assistance and extended learning resources         |
| Workflow uses Bright Data for proxy scraping, affiliate link: https://get.brightdata.com/1tndi4600b25                              | Support Bright Data usage and get commission                 |
| Workflow is designed for spontaneous travelers to find last-minute deals cheaply with minimal setup                                | Use case motivation                                          |
| Consider adding scheduling (Cron) and alerts (Telegram/Email) for enhanced automation                                               | Suggested next steps                                         |

---

**Disclaimer:**  
This document is an analysis based exclusively on an n8n automation workflow. All content adheres to legal and ethical use policies. No protected or offensive data is processed. The workflow operates on publicly accessible data and authorized API usage.

---