Scrape Yelp Business Data with Scrape.do API & Google Sheets Storage

https://n8nworkflows.xyz/workflows/scrape-yelp-business-data-with-scrape-do-api---google-sheets-storage-11454


# Scrape Yelp Business Data with Scrape.do API & Google Sheets Storage

---
### 1. Workflow Overview

This n8n workflow automates the scraping of Yelp business data by URL, using the Scrape.do API for web scraping and Google Sheets for data storage. It is designed for users who want to collect detailed business information from Yelp pages asynchronously and save it in an organized spreadsheet format.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the Yelp business URL submitted by the user via a web form.
- **1.2 Scraping Job Creation and Polling:** Sends the URL to Scrape.do’s asynchronous scraping API, waits, and polls until the scraping job is complete.
- **1.3 Data Extraction:** Retrieves the raw HTML content of the scraped page and parses it using regex and JSON-LD patterns to extract business details.
- **1.4 Data Storage:** Appends the extracted structured data to a predefined Google Sheet for further use or analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving a Yelp business URL input from the user via a web form node.

**Nodes Involved:**  
- 📥 Form Trigger  
- Sticky Note (📋 Form Input)

**Node Details:**

- **📥 Form Trigger**  
  - Type: Form Trigger node  
  - Role: Starts the workflow when a user submits the Yelp business URL  
  - Configuration: Single required form field labeled "Yelp Business URL" with placeholder "https://www.yelp.com/biz/business-name-city"  
  - Input: User-submitted form data  
  - Output: Passes form data as JSON containing the URL  
  - Version: 2.2  
  - Edge Cases: Users submitting invalid URLs or incomplete URLs could cause failures downstream; no explicit validation beyond required field is configured.

- **Sticky Note (📋 Form Input)**  
  - Type: Sticky Note  
  - Role: Documentation for the Input Reception block; no workflow logic  

---

#### 2.2 Scraping Job Creation and Polling

**Overview:**  
This block creates an asynchronous scraping job through the Scrape.do API to fetch the Yelp business page with JavaScript rendering enabled, then polls the job status until completion.

**Nodes Involved:**  
- 🔍 Create Scrape.do Job  
- ⏳ Wait 15s  
- 📡 Check Job Status  
- 🔁 Is Job Complete?  
- ⏳ Wait 10s Retry  
- Sticky Notes (Scraping Section, Sticky Note1)

**Node Details:**

- **🔍 Create Scrape.do Job**  
  - Type: HTTP Request node  
  - Role: Sends a POST request to Scrape.do API to create a scraping job  
  - Configuration:  
    - URL: https://q.scrape.do/api/v1/jobs  
    - Method: POST  
    - Headers: Includes `X-Token` (replace with actual Scrape.do API token) and `Content-Type: application/json`  
    - JSON Body: Contains the target URL from form input, with parameters enabling JS rendering (`Render` object), US geocode, desktop device, and super scraping mode  
  - Input: JSON with user-submitted URL  
  - Output: JSON with JobID and TaskIDs used for later polling  
  - Version: 4.2  
  - Edge Cases: Invalid or missing API token leads to auth failure; malformed URL input; API rate limits or downtime

- **⏳ Wait 15s**  
  - Type: Wait node  
  - Role: Waits 15 seconds before polling job status to allow Scrape.do to start processing  
  - Input: Job creation output  
  - Output: Passes data forward after delay  
  - Version: 1.1  
  - Edge Cases: Fixed wait time may be insufficient for very slow responses; no dynamic backoff

- **📡 Check Job Status**  
  - Type: HTTP Request node  
  - Role: Polls Scrape.do API for job status using JobID from previous node  
  - Configuration:  
    - URL: https://q.scrape.do/api/v1/jobs/{{ JobID }} (JobID dynamically from previous node)  
    - Method: GET  
    - Headers: Contains `X-Token` with API token  
  - Input: JobID from job creation  
  - Output: JSON with job status (e.g., "success")  
  - Version: 4.2  
  - Edge Cases: Network errors, incorrect JobID, expired tokens

- **🔁 Is Job Complete?**  
  - Type: If node  
  - Role: Checks if job status equals "success"  
  - Configuration: Condition compares `Status` field from job status response to "success"  
  - Input: Job status response  
  - Output:  
    - True branch: Proceed to fetch results  
    - False branch: Wait 10 seconds then retry polling  
  - Version: 2.2  
  - Edge Cases: Status could be other states like "error" or "pending"; no explicit error handling branch

- **⏳ Wait 10s Retry**  
  - Type: Wait node  
  - Role: Waits 10 seconds before retrying job status check  
  - Input: False branch from If node  
  - Output: Loops back to Check Job Status node  
  - Version: 1.1  
  - Edge Cases: May loop indefinitely if job never completes; no max retry limit

- **Sticky Notes (Scraping Section, Sticky Note1)**  
  - Type: Sticky Notes  
  - Role: Describe the scraping loop and job creation logic; no execution logic

---

#### 2.3 Data Extraction

**Overview:**  
After scraping job completion, this block fetches the raw HTML content of the Yelp business page and parses it using JavaScript regex and JSON-LD patterns to extract key business information.

**Nodes Involved:**  
- 📥 Fetch Task Results  
- 🔧 Parse Yelp HTML  
- Sticky Note (Processing Section, Sticky Note - Parse)

**Node Details:**

- **📥 Fetch Task Results**  
  - Type: HTTP Request node  
  - Role: Retrieves the scraped HTML content for the first task of the job  
  - Configuration:  
    - URL: https://q.scrape.do/api/v1/jobs/{{ JobID }}/{{ TaskID }} (JobID and TaskID from job creation node)  
    - Method: GET  
    - Headers: Includes `X-Token` with API token  
  - Input: Job and Task IDs  
  - Output: JSON containing scraped HTML content  
  - Version: 4.2  
  - Edge Cases: TaskID missing or invalid; network issues

- **🔧 Parse Yelp HTML**  
  - Type: Code node (JavaScript)  
  - Role: Parses raw HTML to extract structured business details such as name, rating, reviews count, phone, address, price range, categories, website, hours, images/videos URLs, and timestamp  
  - Configuration:  
    - Runs once per item  
    - Uses regex patterns and JSON-LD matching to extract data fields from HTML string  
    - Decodes HTML entities for clean text  
  - Key Expressions: Regex `.match()` on HTML content for fields like `"name"`, `"aggregateRating"`, `"telephone"`, `"address"`, `"priceRange"`, `"openingHours"` etc.  
  - Input: Raw HTML content and URL  
  - Output: JSON object with all extracted fields, defaulting to 'N/A' if not found  
  - Version: 2  
  - Edge Cases: HTML structure changes may break regex; missing fields; malformed HTML; regex failures

- **Sticky Notes (Processing Section, Sticky Note - Parse)**  
  - Type: Sticky Notes  
  - Role: Document the parsing logic and data extraction details

---

#### 2.4 Data Storage

**Overview:**  
This block appends the extracted business data to a configured Google Sheet for persistent storage.

**Nodes Involved:**  
- 📊 Store to Google Sheet  
- Sticky Note (Sticky Note6)

**Node Details:**

- **📊 Store to Google Sheet**  
  - Type: Google Sheets node  
  - Role: Appends parsed business data as a new row in Google Sheets  
  - Configuration:  
    - Operation: Append  
    - Sheet Name: "Sheet1" (gid=0)  
    - Document ID: User must replace `YOUR_GOOGLE_SHEET_ID` with actual Google Sheet ID  
    - Columns: Defines mapping for fields `name, overall_rating, reviews_count, url, phone, address, price_range, categories, website, hours, images_videos_urls, scraped_at`  
  - Input: JSON with parsed business fields  
  - Output: Confirmation of append operation  
  - Credentials: Requires Google Sheets OAuth2 credentials with write access  
  - Version: 4.6  
  - Edge Cases: Invalid Sheet ID; OAuth token expiration; permission denied; schema mismatch

- **Sticky Note (Sticky Note6)**  
  - Type: Sticky Note  
  - Role: Documentation describing the Google Sheets storage function

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role                           | Input Node(s)            | Output Node(s)         | Sticky Note                                                                                     |
|------------------------|-------------------------|-----------------------------------------|--------------------------|------------------------|------------------------------------------------------------------------------------------------|
| 📥 Form Trigger         | Form Trigger            | Receives Yelp business URL input        |                          | 🔍 Create Scrape.do Job | ## 📋 Form Input Starts the workflow when the user submits a business Yelp URL through the form |
| 🔍 Create Scrape.do Job | HTTP Request            | Creates async scraping job via Scrape.do | 📥 Form Trigger          | ⏳ Wait 15s             | ## 🚀 Scrape.do Job Creates an async scraping job via Scrape.do API with JS rendering enabled   |
| ⏳ Wait 15s             | Wait                    | Waits 15 seconds before polling          | 🔍 Create Scrape.do Job  | 📡 Check Job Status     | ## 🔄 Scraping Loop Submits URL to Scrape.do async API and polls until the job completes        |
| 📡 Check Job Status     | HTTP Request            | Polls Scrape.do for job status           | ⏳ Wait 15s, ⏳ Wait 10s Retry | 🔁 Is Job Complete?    | ## 🔄 Scraping Loop Submits URL to Scrape.do async API and polls until the job completes        |
| 🔁 Is Job Complete?     | If                      | Checks if job status is "success"        | 📡 Check Job Status      | 📥 Fetch Task Results (true), ⏳ Wait 10s Retry (false) | ## 🔄 Scraping Loop Submits URL to Scrape.do async API and polls until the job completes        |
| ⏳ Wait 10s Retry       | Wait                    | Waits 10 seconds before retrying         | 🔁 Is Job Complete?      | 📡 Check Job Status     | ## 🔄 Scraping Loop Submits URL to Scrape.do async API and polls until the job completes        |
| 📥 Fetch Task Results   | HTTP Request            | Retrieves scraped HTML content            | 🔁 Is Job Complete? (true) | 🔧 Parse Yelp HTML      | ## 🔧 Data Extraction Fetches raw HTML and parses business details using regex and JSON-LD      |
| 🔧 Parse Yelp HTML      | Code                    | Parses HTML to extract structured data    | 📥 Fetch Task Results    | 📊 Store to Google Sheet | ## 🔧 Parse HTML Extracts business data (name, rating, reviews, URL, images) from the raw HTML  |
| 📊 Store to Google Sheet| Google Sheets           | Appends extracted data to Google Sheet    | 🔧 Parse Yelp HTML       |                        | ## 📊 Google Sheets Stores business data (name, rating, reviews, phone, address, etc.) to Google Sheet |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form title: "Yelp Business URL Scraper"  
   - Add one required form field:  
     - Label: "Yelp Business URL"  
     - Placeholder: "https://www.yelp.com/biz/business-name-city"  
   - This node will start the workflow on URL submission.

2. **Add an HTTP Request Node to Create Scrape.do Job**  
   - Name: "🔍 Create Scrape.do Job"  
   - Method: POST  
   - URL: `https://q.scrape.do/api/v1/jobs`  
   - Headers:  
     - `X-Token`: Your Scrape.do API token (replace `"YOUR_SCRAPEDO_TOKEN"`)  
     - `Content-Type`: `application/json`  
   - Body: JSON (use expression mode)  
     ```json
     {
       "Targets": [{{$json["Yelp Business URL"]}}],
       "Super": true,
       "GeoCode": "us",
       "Device": "desktop",
       "Render": {
         "BlockResources": false,
         "WaitUntil": "networkidle2",
         "CustomWait": 3000
       }
     }
     ```
   - Enable "Send Body" and "Send Headers" options.

3. **Add a Wait Node (15s)**  
   - Name: "⏳ Wait 15s"  
   - Set wait time to 15 seconds.

4. **Add an HTTP Request Node to Check Job Status**  
   - Name: "📡 Check Job Status"  
   - Method: GET  
   - URL: `https://q.scrape.do/api/v1/jobs/{{ $json["JobID"] }}` (use expression to get JobID from job creation node)  
   - Headers:  
     - `X-Token`: Your Scrape.do API token  
   - Enable "Send Headers".

5. **Add an If Node to Check if Job is Complete**  
   - Name: "🔁 Is Job Complete?"  
   - Condition: Compare `Status` in JSON response equals `"success"`  
   - True branch: proceed to fetch task results  
   - False branch: retry after wait

6. **Add a Wait Node (10s) for Retry**  
   - Name: "⏳ Wait 10s Retry"  
   - Wait time: 10 seconds  
   - Connect False branch of If node here, then loop back to "📡 Check Job Status".

7. **Add an HTTP Request Node to Fetch Task Results**  
   - Name: "📥 Fetch Task Results"  
   - Method: GET  
   - URL: `https://q.scrape.do/api/v1/jobs/{{ $json["JobID"] }}/{{ $json["TaskIDs"][0] }}`  
   - Headers:  
     - `X-Token`: Your Scrape.do API token  
   - Enable "Send Headers".

8. **Add a Code Node to Parse Yelp HTML**  
   - Name: "🔧 Parse Yelp HTML"  
   - Language: JavaScript  
   - Paste the provided JS code that parses the HTML content to extract fields such as name, rating, phone, address, etc.  
   - Input: Raw HTML content from the previous node  
   - Output: JSON object with business data fields and timestamp.

9. **Add a Google Sheets Node to Store Data**  
   - Name: "📊 Store to Google Sheet"  
   - Operation: Append  
   - Google Sheets Document ID: Replace with your actual Google Sheet ID  
   - Sheet Name: Select the sheet (e.g., "Sheet1" or GID 0)  
   - Map columns to output JSON fields:  
     - name  
     - overall_rating  
     - reviews_count  
     - url  
     - phone  
     - address  
     - price_range  
     - categories  
     - website  
     - hours  
     - images_videos_urls  
     - scraped_at  
   - Connect Google Sheets OAuth2 credentials with proper access.

10. **Connect Workflow Nodes in This Order:**  
    📥 Form Trigger → 🔍 Create Scrape.do Job → ⏳ Wait 15s → 📡 Check Job Status → 🔁 Is Job Complete?  
    - True branch → 📥 Fetch Task Results → 🔧 Parse Yelp HTML → 📊 Store to Google Sheet  
    - False branch → ⏳ Wait 10s Retry → loops back to 📡 Check Job Status

11. **Replace Placeholder Tokens and IDs:**  
    - Replace `"YOUR_SCRAPEDO_TOKEN"` with your actual Scrape.do API token in all HTTP Request nodes.  
    - Replace `"YOUR_GOOGLE_SHEET_ID"` with your actual Google Sheet ID in the Google Sheets node configuration.

12. **Test the Workflow:**  
    - Trigger the form with a valid Yelp business URL and ensure the data appears in your Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                    |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Get your API token from https://scrape.do/dashboard                                                        | Scrape.do API Dashboard                           |
| Documentation for Scrape.do API usage: https://scrape.do/documentation                                     | Scrape.do official docs                            |
| Create Google Sheet with columns: name, overall_rating, reviews_count, url, phone, address, price_range, categories, website, hours, images_videos_urls, scraped_at | Required Google Sheets schema                       |
| Connect Google Sheets OAuth2 credentials with write permissions                                           | Google Sheets API Authentication                   |
| Workflow uses asynchronous polling pattern with fixed wait intervals (15s initial, 10s retry)             | Can be optimized with dynamic backoff or max retries |
| Regex parsing in code node sensitive to Yelp HTML structure changes; maintain and update if site changes  | Potential maintenance point                         |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---