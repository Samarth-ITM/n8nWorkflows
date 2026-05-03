Extract Business Email Addresses Using Serper.dev and ScrapingBee from Google Sheets

https://n8nworkflows.xyz/workflows/extract-business-email-addresses-using-serper-dev-and-scrapingbee-from-google-sheets-7071


# Extract Business Email Addresses Using Serper.dev and ScrapingBee from Google Sheets

### 1. Workflow Overview

This workflow automates the enrichment and email extraction process for business leads stored in a Google Sheet. It listens for row updates where the user activates a lead to trigger the process. Using Serper.dev, it performs a location and business-type-based Google search to find relevant companies and their websites. It then generates potential contact page URLs for these companies and uses ScrapingBee to scrape these pages for email addresses. Extracted emails are consolidated and written back to the Google Sheet, with status updates marking each step's progress.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Validation:** Triggering on Google Sheet row update and verifying required fields.
- **1.2 Search Query Preparation and Execution:** Setting search parameters and querying Serper.dev for companies.
- **1.3 Search Results Filtering and Storage:** Filtering out unwanted results and appending valid company data to the sheet.
- **1.4 URL Variant Generation and Validation:** Generating and testing potential contact/support page URLs for each company.
- **1.5 Web Page Scraping and Email Extraction:** Scraping valid URLs using ScrapingBee and extracting emails from HTML.
- **1.6 Email Consolidation and Sheet Update:** Merging found emails with existing data and updating the Google Sheet.
- **1.7 Status Updates:** Marking each lead as running, missing info, or finished based on workflow progress.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:** Listens for changes in the Google Sheet and validates that mandatory fields are present before processing.
- **Nodes Involved:**  
  - Google Sheets Trigger  
  - If3 (Validation)  
  - Update Running Status  
  - Update Missing Information Status

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type:* Trigger node for Google Sheets row updates  
    - *Config:* Watches the `Activate` column for changes, polling every minute  
    - *Input:* Google Sheet row data  
    - *Output:* Passes row data downstream  
    - *Edge Cases:* API auth errors, rate limits, network issues

  - **If3 (Validation Check)**  
    - *Type:* Conditional node  
    - *Config:* Checks if at least one of `Client`, `City`, or `State` is not empty  
    - *Input:* Output from Google Sheets Trigger  
    - *Output:* Routes to either Update Running Status or Update Missing Information Status  
    - *Edge Cases:* Expression evaluation errors, empty fields

  - **Update Running Status**  
    - *Type:* Google Sheets update node  
    - *Config:* Updates the `Status` column to "Running" for the given `Client` row  
    - *Input:* Validated input data from If3  
    - *Output:* Triggers the next block  
    - *Edge Cases:* API write failures, mismatched row update keys

  - **Update Missing Information Status**  
    - *Type:* Google Sheets update node  
    - *Config:* Updates the `Status` column to "Missing data" if validation fails  
    - *Input:* Invalid input data from If3  
    - *Output:* Ends processing for that row  
    - *Edge Cases:* Same as above

---

#### 2.2 Search Query Preparation and Execution

- **Overview:** Sets up search parameters and queries Serper.dev to find companies matching business type and location.
- **Nodes Involved:**  
  - Set Information  
  - Search Companies (Serper.dev)

- **Node Details:**

  - **Set Information**  
    - *Type:* Set node  
    - *Config:* Defines search parameters such as `result_count`, `state`, `city`, `client`, `business_type`, `country`, `country_code`, and `language`  
    - *Input:* Data from Update Running Status  
    - *Output:* Parameters for the search query  
    - *Edge Cases:* Missing or malformed input fields

  - **Search Companies (Serper.dev)**  
    - *Type:* HTTP Request (POST)  
    - *Config:* Sends POST request to Serper.dev API with search query built from node inputs; uses API key in HTTP header authentication  
    - *Input:* Parameters from Set Information  
    - *Output:* JSON response with organic search results  
    - *Edge Cases:* API key invalid or expired, request throttling, network errors

---

#### 2.3 Search Results Filtering and Storage

- **Overview:** Filters out undesirable companies based on blacklist terms and appends filtered results to the Google Sheet.
- **Nodes Involved:**  
  - Extract Company & Website (Code)  
  - Add research Results (Google Sheets Append)

- **Node Details:**

  - **Extract Company & Website**  
    - *Type:* Code node (JavaScript)  
    - *Config:* Filters search results using a blacklist of unwanted company names and links, extracting company name, website, and contextual info from Set Information  
    - *Input:* Search results JSON from Serper.dev  
    - *Output:* Filtered array of company and website records  
    - *Edge Cases:* Unexpected API response structure, empty results

  - **Add research Results**  
    - *Type:* Google Sheets append node  
    - *Config:* Appends filtered companies and their websites with location and client info to a target sheet tab named “Data”  
    - *Input:* Filtered companies from the code node  
    - *Output:* Triggers Website Options node  
    - *Edge Cases:* API write errors, data schema mismatch

---

#### 2.4 URL Variant Generation and Validation

- **Overview:** For each company website, generates multiple URL variants targeting contact/support pages and tests their validity.
- **Nodes Involved:**  
  - Website Options (Code)  
  - Loop Over Items (Split in Batches)  
  - Test pages (HTTP Request)  
  - If (Conditional: Check for error message)

- **Node Details:**

  - **Website Options**  
    - *Type:* Code node (JavaScript)  
    - *Config:* Extracts base URL from website and generates variants by appending common contact/support page paths  
    - *Input:* Appended companies from the previous node  
    - *Output:* Expanded list of URL variants per company  
    - *Edge Cases:* Invalid or malformed URLs, missing website field

  - **Loop Over Items**  
    - *Type:* SplitInBatches node  
    - *Config:* Processes URL variants in batches (default batch size) to avoid overwhelming downstream nodes  
    - *Input:* URL variants array  
    - *Output:* One batch at a time to Test pages node  
    - *Edge Cases:* Large batch size causing timeouts

  - **Test pages**  
    - *Type:* HTTP Request node  
    - *Config:* Sends GET request to each URL variant to check if the page exists  
    - *Input:* Single URL from Loop Over Items batch  
    - *Output:* Passes response or error downstream  
    - *Error Handling:* On error, continues regular output to avoid stopping the flow  
    - *Edge Cases:* 404 or 500 errors, network timeouts

  - **If (Check for error message)**  
    - *Type:* Conditional node  
    - *Config:* Checks if the response contains an error message to filter valid pages  
    - *Input:* Output from Test pages  
    - *Output:* Routes valid URLs forward, invalid ones back into Loop Over Items for retry or discard  
    - *Edge Cases:* Unexpected error message format

---

#### 2.5 Web Page Scraping and Email Extraction

- **Overview:** Scrapes valid company pages using ScrapingBee and extracts email addresses from the HTML content.
- **Nodes Involved:**  
  - Scraping Bee (HTTP Request)  
  - If1 (Conditional: Check for error)  
  - Email Extractor (Code)  
  - If2 (Conditional: Check if emails found)

- **Node Details:**

  - **Scraping Bee**  
    - *Type:* HTTP Request node  
    - *Config:* Uses ScrapingBee API with API key to scrape the full rendered HTML of each valid URL  
    - *Input:* Valid URLs from If node  
    - *Output:* Scraped HTML content or error message  
    - *Error Handling:* On error, continues regular output to maintain flow  
    - *Edge Cases:* API rate limits, invalid key, page load failure

  - **If1 (Check for error message)**  
    - *Type:* Conditional node  
    - *Config:* Filters out results with error messages in the scraping response  
    - *Input:* Scraping Bee output  
    - *Output:* Routes successful scrapes to Email Extractor, errors back to Loop Over Items  
    - *Edge Cases:* False positives in error detection

  - **Email Extractor**  
    - *Type:* Code node (JavaScript)  
    - *Config:* Uses regex to extract all unique email addresses from scraped HTML, combines with client, company, and website info  
    - *Input:* Scraping Bee successful responses  
    - *Output:* JSON with emails and related info  
    - *Edge Cases:* HTML content missing or malformed, regex misses uncommon email formats

  - **If2 (Check if emails found)**  
    - *Type:* Conditional node  
    - *Config:* Checks if the extracted `email` field is non-empty  
    - *Input:* Output of Email Extractor  
    - *Output:* Routes to Get Emails node if emails found; otherwise loops back to Loop Over Items to check other URLs  
    - *Edge Cases:* Empty or malformed email strings

---

#### 2.6 Email Consolidation and Sheet Update

- **Overview:** Retrieves any existing emails for companies, merges newly found emails, and updates the Google Sheet.
- **Nodes Involved:**  
  - Get Emails (Google Sheets)  
  - Add Emails (Google Sheets Update)  
  - Wait (Wait node for rate limiting or pacing)

- **Node Details:**

  - **Get Emails**  
    - *Type:* Google Sheets node  
    - *Config:* Searches the "Data" sheet for existing emails matching the extracted company name  
    - *Input:* Company name from Email Extractor  
    - *Output:* Existing emails if found  
    - *Edge Cases:* No existing record found, read errors

  - **Add Emails**  
    - *Type:* Google Sheets update node  
    - *Config:* Updates the emails column by appending new emails to existing ones (comma-separated) for the company row  
    - *Input:* Existing emails from Get Emails and new emails from Email Extractor  
    - *Output:* Triggers Wait node  
    - *Edge Cases:* Concurrent updates, data overwrite risk

  - **Wait**  
    - *Type:* Wait node  
    - *Config:* Pauses the workflow for 1 minute to avoid API rate limits or server overload  
    - *Input:* Completion of Add Emails update  
    - *Output:* Loops back to Loop Over Items for next URL variant processing  
    - *Edge Cases:* Delay too short causing rate limiting, delay too long impacting throughput

---

#### 2.7 Status Updates

- **Overview:** Marks each lead as finished once all URL variants and email extraction are completed.
- **Nodes Involved:**  
  - Update Finished Status (Google Sheets Update)

- **Node Details:**

  - **Update Finished Status**  
    - *Type:* Google Sheets update node  
    - *Config:* Updates the `Status` column to "Finished" for the processed client row  
    - *Input:* Completion signal from Loop Over Items node (all URLs processed)  
    - *Output:* Ends the workflow for that lead  
    - *Edge Cases:* API write errors, row matching errors

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                               | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                         |
|---------------------------|-------------------------|-----------------------------------------------|-------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger      | googleSheetsTrigger     | Trigger on row update in Google Sheet         | -                       | If3                       | # Sheet Trigger and Set Information: instructions on sheet setup and trigger activation                                            |
| If3                       | if                      | Validate required fields (Client, City, State)| Google Sheets Trigger    | Update Running Status, Update Missing Information Status |                                                                                                                             |
| Update Running Status      | googleSheets            | Mark row as Running                            | If3                     | Set Information            |                                                                                                                                    |
| Update Missing Information Status | googleSheets      | Mark row as Missing data                      | If3                     | -                         |                                                                                                                                    |
| Set Information           | set                     | Prepare search parameters                      | Update Running Status    | Search Companies (Serper.dev) |                                                                                                                                    |
| Search Companies (Serper.dev) | httpRequest          | Perform company search via Serper.dev API     | Set Information          | Extract Company & Website  | # Serper.dev Integration: API key instructions, search filtering, update sheet, generate possible email pages                      |
| Extract Company & Website | code                    | Filter search results and extract company info| Search Companies (Serper.dev) | Add research Results      |                                                                                                                                    |
| Add research Results      | googleSheets            | Append filtered companies and websites        | Extract Company & Website| Website Options            |                                                                                                                                    |
| Website Options           | code                    | Generate URL variants for contact/support pages| Add research Results     | Loop Over Items            | # Check Generated URLs: test URLs, keep valid, avoid unnecessary calls                                                             |
| Loop Over Items           | splitInBatches          | Process URL variants in batches                | Website Options, If, If1, If2, Wait, Add Emails | Test pages, Update Finished Status |                                                                                                                                    |
| Test pages                | httpRequest             | Test each generated URL for existence          | Loop Over Items          | If                        |                                                                                                                                    |
| If                       | if                      | Check for error in page test response          | Test pages               | Scraping Bee, Loop Over Items |                                                                                                                                    |
| Scraping Bee              | httpRequest             | Scrape valid pages using ScrapingBee API       | If                       | If1                       | # ScrapingBee Integration: API key, scraping info, filter results, extract emails, update sheet                                     |
| If1                      | if                      | Check for error in scraping result              | Scraping Bee             | Email Extractor, Loop Over Items |                                                                                                                                    |
| Email Extractor           | code                    | Extract emails from scraped HTML                | If1                      | If2                       |                                                                                                                                    |
| If2                      | if                      | Check if emails were found                       | Email Extractor           | Get Emails, Loop Over Items |                                                                                                                                    |
| Get Emails               | googleSheets            | Retrieve existing emails for company            | If2                      | Add Emails                 |                                                                                                                                    |
| Add Emails               | googleSheets            | Append new emails to existing ones in sheet     | Get Emails                | Wait                      |                                                                                                                                    |
| Wait                     | wait                    | Pause for 1 minute to avoid rate limits         | Add Emails                | Loop Over Items            |                                                                                                                                    |
| Update Finished Status    | googleSheets            | Mark lead as Finished after processing          | Loop Over Items           | -                         | # Update Status to Finished: marks row as fully processed                                                                           |
| Sticky Note               | stickyNote              | Documentation note about sheet and trigger setup| -                        | -                         | # Sheet Trigger and Set Information: detailed instructions and example sheet link                                                   |
| Sticky Note1              | stickyNote              | Documentation note about Serper.dev integration | -                        | -                         | # Serper.dev Integration: API key, search, filtering, update, generate URLs                                                         |
| Sticky Note2              | stickyNote              | Documentation note about URL checking           | -                        | -                         | # Check Generated URLs: testing and validation of company contact URLs                                                              |
| Sticky Note3              | stickyNote              | Documentation note about ScrapingBee integration| -                        | -                         | # ScrapingBee Integration: API key, scraping, filter, extract emails, update sheet                                                  |
| Sticky Note4              | stickyNote              | Documentation note about finishing status update| -                        | -                         | # Update Status to Finished: marks row as finished                                                                                   |
| Sticky Note5              | stickyNote              | General workflow summary and prerequisites      | -                        | -                         | # Template Summary: overview, prerequisites, inputs, outputs, configuration, customization tips                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger node:**
   - Configure to watch your Google Sheet document and sheet tab.
   - Set event to `rowUpdate`.
   - Watch the `Activate` column.
   - Set polling frequency (e.g., every minute).
   - Attach Google Sheets OAuth2 credentials.

2. **Add an If node ("If3") to validate required columns:**
   - Condition: Check if `Client` OR `City` OR `State` is not empty.
   - Use expression mode with OR combinator.

3. **Add two Google Sheets nodes for status update:**
   - "Update Running Status": update matching row (by `Client`), set `Status` = "Running".
   - "Update Missing Information Status": update matching row, set `Status` = "Missing data".
   - Connect validation If node outputs to these nodes accordingly.

4. **Create a Set node ("Set Information"):**
   - Set parameters for search: `result_count`, `state`, `city`, `client`, `business_type`, `country`, `country_code`, `language`.
   - Use expression referencing values from Google Sheets Trigger node where possible.
   - Example: country = "Argentina", country_code = "AR", language = "es-419", result_count = 10.

5. **Add an HTTP Request node ("Search Companies (Serper.dev)"):**
   - URL: https://google.serper.dev/search
   - Method: POST
   - Authentication: HTTP Header Auth with Serper.dev API key
   - Request Body (JSON):  
     ```
     {
       "q": "{{ $json.business_type }} in {{ $json.city }}, {{ $json.state }}, {{ $json.country }}",
       "num": {{ $json.result_count }},
       "gl": "{{ $json.country_code }}",
       "hl": "{{ $json.language }}"
     }
     ```
   - Set credentials accordingly.

6. **Add a Code node ("Extract Company & Website"):**
   - JavaScript to filter out unwanted companies by blacklist terms.
   - Extract company title and link.
   - Add contextual info from Set Information node.
   - Output array of filtered results.

7. **Add a Google Sheets Append node ("Add research Results"):**
   - Append filtered company data (`Company`, `website`, `Client`, `City`, `State`) to a "Data" tab.
   - Map columns explicitly.
   - Configure sheet and document IDs.

8. **Add a Code node ("Website Options"):**
   - Extract base URL from each company website.
   - Generate URL variants by appending contact/support paths (e.g., `/contact`, `/support`, `/help-center`).
   - Return array of JSON objects with company info and variant URLs.

9. **Add a SplitInBatches node ("Loop Over Items"):**
   - Process URL variants in manageable batches (default batch size).
   - Connect output to Test pages node.

10. **Add an HTTP Request node ("Test pages"):**
    - Send GET requests to each URL variant.
    - No authentication.
    - Set error handling to continue on error.
    - Evaluate if URL exists based on response.

11. **Add an If node ("If"):**
    - Check if response contains an error message.
    - Route valid URLs to Scraping Bee node; invalid ones back to Loop Over Items or discard.

12. **Add an HTTP Request node ("Scraping Bee"):**
    - Scrape valid URLs using ScrapingBee API.
    - URL parameter from tested URL.
    - API key in query string or header.
    - Enable JavaScript rendering.
    - On error, continue regular output.

13. **Add an If node ("If1"):**
    - Check scraping response for errors.
    - Route successful scrapes to Email Extractor node.
    - Route errors back to Loop Over Items.

14. **Add a Code node ("Email Extractor"):**
    - Parse scraped HTML.
    - Extract unique emails using regex.
    - Return client, company, website, and email string.

15. **Add an If node ("If2"):**
    - Check if emails field is not empty.
    - Route non-empty emails to Get Emails node.
    - Route empty results back to Loop Over Items.

16. **Add a Google Sheets node ("Get Emails"):**
    - Search "Data" tab for existing row matching `Company`.
    - Retrieve existing emails if any.

17. **Add a Google Sheets Update node ("Add Emails"):**
    - Update the emails column by appending new emails (comma separated).
    - Match rows by `Company`.

18. **Add a Wait node ("Wait"):**
    - Pause workflow for 1 minute before continuing.
    - Connect output back to Loop Over Items for next batch processing.

19. **Connect Loop Over Items node's completion output to a Google Sheets Update node ("Update Finished Status"):**
    - Update the original lead row `Status` to "Finished".

20. **Add Sticky Notes for documentation:**
    - Add on-screen instructions about API keys, sheet setup, and workflow purpose.

21. **Configure all Google Sheets and HTTP Request nodes with proper credentials.**

22. **Test workflow end-to-end with a sample Google Sheet row.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Sheet Template for input data with required columns: business type, city, state, activate                                               | https://docs.google.com/spreadsheets/d/1222TvBxE2UBb1MK2xDMoQSd5WHQ7mA5Ew-W6vBgfCJs/edit?usp=sharing       |
| Serper.dev API signup and usage instructions                                                                                           | https://serper.dev/                                                                                       |
| ScrapingBee API signup and usage instructions                                                                                          | https://www.scrapingbee.com/                                                                              |
| Workflow summary: automates lead enrichment and email extraction from Google Sheets using Serper.dev and ScrapingBee                   | See Sticky Note5 content                                                                                   |
| Blacklist terms in code node to exclude irrelevant or unwanted companies                                                                | Customizable in "Extract Company & Website" code node                                                    |
| Wait node added to avoid API rate limits and pacing between batch processing                                                           | Can be adjusted for different rate limits or throughput needs                                            |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting current content policies with no illegal or protected elements. All data handled is legal and public.