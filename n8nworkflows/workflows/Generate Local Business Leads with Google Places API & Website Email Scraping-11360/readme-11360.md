Generate Local Business Leads with Google Places API & Website Email Scraping

https://n8nworkflows.xyz/workflows/generate-local-business-leads-with-google-places-api---website-email-scraping-11360


# Generate Local Business Leads with Google Places API & Website Email Scraping

### 1. Workflow Overview

This workflow automates the generation of local business leads by leveraging the Google Places API combined with website email scraping. It is designed to take user input specifying search criteria, query Google Places for matching businesses, retrieve detailed business information, and then attempt to scrape email addresses from the businesses’ websites if available. The final output compiles all gathered data into a CSV file for easy export and further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preparation:** Receives user input via a form, parses it, and creates multiple search queries.
- **1.2 Google Places API Interaction:** Performs batched Google Places searches and extracts basic place information.
- **1.3 Business Detail Enrichment:** For each place, fetches detailed business data from Google Places and merges it.
- **1.4 Website Email Extraction:** Checks for the presence of a website, scrapes it for email addresses, cleans and merges this data.
- **1.5 Fallback Handling:** Provides handling when no website is present.
- **1.6 Final Data Compilation:** Merges all collected data and converts it to CSV format for output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:**  
This block starts the workflow by capturing user input through a form, parsing the submitted data, and generating multiple search combinations for querying Google Places.

- **Nodes Involved:**  
  - Form Trigger  
  - Parse Form Data  
  - Create Search Combinations  
  - Split Into Searches

- **Node Details:**

  - **Form Trigger**  
    - Type: Form Trigger (Webhook-based)  
    - Role: Listens for form submissions containing search parameters.  
    - Config: Uses a webhook ID to receive data. No additional parameters set.  
    - Inputs: External HTTP form submission  
    - Outputs: Raw form data to the next node  
    - Failures: Network/webhook issues; invalid form data can cause errors.  
    - Notes: Entry point of the workflow.

  - **Parse Form Data**  
    - Type: Set  
    - Role: Processes and structures the raw form data into usable variables for search queries.  
    - Config: Typically sets variables based on form inputs; no explicit parameters shown.  
    - Inputs: From Form Trigger  
    - Outputs: Structured search parameters  
    - Failures: Expression errors if expected form fields are missing or malformed.

  - **Create Search Combinations**  
    - Type: Set  
    - Role: Constructs multiple search query combinations (e.g., location + category) to cover various search scenarios.  
    - Config: Defines arrays or combinations for search terms.  
    - Inputs: Parsed form data  
    - Outputs: Multiple search combinations  
    - Failures: Logic errors if input data arrays are empty or improperly formatted.

  - **Split Into Searches**  
    - Type: Split Out  
    - Role: Splits the search combinations into individual items to process each separately in parallel or sequence.  
    - Config: Default splitting behavior  
    - Inputs: Search combinations array  
    - Outputs: Single search query per execution path  
    - Failures: Empty input arrays cause no execution downstream.

---

#### 2.2 Google Places API Interaction

- **Overview:**  
This block executes Google Places API calls to search for businesses based on the prepared queries and extracts relevant place details.

- **Nodes Involved:**  
  - Google Places Search1  
  - Loop Over Items  
  - Extract Place Info  
  - Wait (Rate Limit)

- **Node Details:**

  - **Google Places Search1**  
    - Type: HTTP Request  
    - Role: Calls Google Places API to search for places matching the current query.  
    - Config: Uses HTTP GET with parameters like query, location, radius, and API key.  
    - Inputs: Individual search query from Split Into Searches  
    - Outputs: JSON response with place results  
    - Failures: API key errors, quota exceeded, network timeout.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes each place result individually to avoid rate limits and manage API calls.  
    - Config: Batch size set for rate limiting  
    - Inputs: Places array from Google Places Search1  
    - Outputs: Single place per execution  
    - Failures: Empty input causes no iteration.

  - **Extract Place Info**  
    - Type: Code  
    - Role: Extracts and structures relevant information from raw Google Places search results (e.g., place_id, name).  
    - Config: Custom JavaScript code node  
    - Inputs: Place data from Loop Over Items  
    - Outputs: Simplified place info object  
    - Failures: Parsing errors if response structure changes.

  - **Wait (Rate Limit)**  
    - Type: Wait  
    - Role: Introduces delay between API calls to comply with Google Places rate limits.  
    - Config: Delay duration configured (seconds/milliseconds)  
    - Inputs: After processing each place  
    - Outputs: Passes data after waiting  
    - Failures: None, unless workflow is stopped during wait.

---

#### 2.3 Business Detail Enrichment

- **Overview:**  
For each identified place, this block queries Google Places Details API to obtain detailed business information and prepares the data for downstream processing.

- **Nodes Involved:**  
  - Get Business Details  
  - Merge Details  
  - Has Website?

- **Node Details:**

  - **Get Business Details**  
    - Type: HTTP Request  
    - Role: Requests detailed information for each place using place_id.  
    - Config: HTTP GET with place_id and API key parameters  
    - Inputs: Output from Wait (Rate Limit) node  
    - Outputs: Detailed business JSON data  
    - Failures: API errors, invalid place_id, rate limits.

  - **Merge Details**  
    - Type: Set  
    - Role: Merges detailed business info with previously extracted place info to form a comprehensive data object.  
    - Config: Maps and combines fields  
    - Inputs: Get Business Details output and previous extracted place info  
    - Outputs: Enriched business data  
    - Failures: Missing fields can cause incomplete merges.

  - **Has Website?**  
    - Type: If  
    - Role: Conditional check to determine if the business has a website URL available.  
    - Config: Checks if website field is present and non-empty  
    - Inputs: Merged business details  
    - Outputs: Routes with website to Loop Over Items1, those without to No Website Fallback  
    - Failures: Expression evaluation errors if field missing.

---

#### 2.4 Website Email Extraction

- **Overview:**  
This block scrapes the websites of businesses (when available) to extract email addresses and cleans the extracted data for accuracy.

- **Nodes Involved:**  
  - Loop Over Items1  
  - Scrape Website1  
  - Wait  
  - Extract Emails & Clean

- **Node Details:**

  - **Loop Over Items1**  
    - Type: Split In Batches  
    - Role: Processes each business website URL one by one to manage load and rate limits.  
    - Config: Batch size configured to limit concurrent requests  
    - Inputs: Businesses with website URLs  
    - Outputs: Single website URL per execution  
    - Failures: Empty input arrays halt execution.

  - **Scrape Website1**  
    - Type: HTTP Request  
    - Role: Requests the HTML content of the business website to scrape for emails.  
    - Config: HTTP GET, retries enabled on failure, continues on fail to avoid workflow stop  
    - Inputs: Website URL for each business  
    - Outputs: Raw HTML page content  
    - Failures: Network errors, website blocking, timeouts.

  - **Wait**  
    - Type: Wait  
    - Role: Delays execution between website scrapes to avoid server overload or IP blocking.  
    - Config: Defined wait time (seconds)  
    - Inputs: After Scrape Website1  
    - Outputs: Passes data downstream  
    - Failures: None except manual interruption.

  - **Extract Emails & Clean**  
    - Type: Code  
    - Role: Parses HTML content to extract email addresses, removes duplicates and invalid formats.  
    - Config: Custom JavaScript to find email patterns and sanitize list  
    - Inputs: Scraped website HTML  
    - Outputs: Cleaned email list attached to business data  
    - Failures: Parsing errors, no emails found.

---

#### 2.5 Fallback Handling

- **Overview:**  
If no website is available for a business, this block provides a fallback path to continue integrating the partial data into the final dataset.

- **Nodes Involved:**  
  - No Website Fallback  
  - Merge

- **Node Details:**

  - **No Website Fallback**  
    - Type: Set  
    - Role: Sets empty email or placeholder data when no website is present.  
    - Config: Adds default values for missing email data  
    - Inputs: Businesses without website from Has Website? node  
    - Outputs: Data forwarded to Merge node  
    - Failures: None expected.

  - **Merge**  
    - Type: Merge  
    - Role: Combines results from businesses with scraped emails and those with fallback data into a unified dataset.  
    - Config: Merge by key or index  
    - Inputs: Email extracted data and fallback data  
    - Outputs: Unified data object for final processing  
    - Failures: Mismatched data sets cause incomplete merges.

---

#### 2.6 Final Data Compilation

- **Overview:**  
This block compiles the unified business lead data and converts it into a CSV file for download or further processing.

- **Nodes Involved:**  
  - Final Data  
  - Convert to CSV

- **Node Details:**

  - **Final Data**  
    - Type: Set  
    - Role: Final formatting and selection of fields for export (e.g., business name, address, phone, emails).  
    - Config: Selects and orders columns for CSV  
    - Inputs: Merged leads data  
    - Outputs: Structured data ready for file conversion  
    - Failures: Missing fields cause incomplete rows.

  - **Convert to CSV**  
    - Type: Convert To File  
    - Role: Transforms JSON structured data into a downloadable CSV file.  
    - Config: CSV format selected, separator, encoding defaults  
    - Inputs: Final Data node output  
    - Outputs: CSV file data for export or further use  
    - Failures: Large datasets may cause timeout or memory issues.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                       | Input Node(s)           | Output Node(s)           | Sticky Note                            |
|------------------------|---------------------|------------------------------------|------------------------|--------------------------|--------------------------------------|
| Form Trigger           | Form Trigger        | Start workflow, receive user input | -                      | Parse Form Data           |                                      |
| Parse Form Data        | Set                 | Parse and structure input data      | Form Trigger            | Create Search Combinations |                                      |
| Create Search Combinations | Set              | Generate multiple search queries    | Parse Form Data          | Split Into Searches       |                                      |
| Split Into Searches    | Split Out           | Split combinations into single queries | Create Search Combinations | Google Places Search1  |                                      |
| Google Places Search1  | HTTP Request        | Query Google Places API for places  | Split Into Searches      | Loop Over Items           |                                      |
| Loop Over Items        | Split In Batches    | Process each place result individually | Google Places Search1  | Wait (Rate Limit), Extract Place Info |                      |
| Extract Place Info     | Code                | Extract relevant place info          | Loop Over Items          | Loop Over Items           |                                      |
| Wait (Rate Limit)      | Wait                | Delay requests for rate limiting    | Loop Over Items          | Get Business Details      |                                      |
| Get Business Details   | HTTP Request        | Retrieve detailed business info     | Wait (Rate Limit)        | Merge Details             |                                      |
| Merge Details          | Set                 | Merge place info with details        | Get Business Details     | Has Website?              |                                      |
| Has Website?           | If                  | Check if business has a website     | Merge Details            | Loop Over Items1 (Yes), No Website Fallback (No) |                 |
| Loop Over Items1       | Split In Batches    | Process each website URL             | Has Website?             | Scrape Website1, Extract Emails & Clean |                      |
| Scrape Website1        | HTTP Request        | Scrape website HTML                  | Loop Over Items1         | Wait                     |                                      |
| Wait                   | Wait                | Delay between scrapes                | Scrape Website1          | Loop Over Items1          |                                      |
| Extract Emails & Clean | Code                | Extract and clean emails from HTML  | Loop Over Items1         | Merge                     |                                      |
| No Website Fallback    | Set                 | Provide default data when no website | Has Website?            | Merge                     |                                      |
| Merge                  | Merge               | Combine with and without website data | Extract Emails & Clean, No Website Fallback | Final Data    |                                      |
| Final Data             | Set                 | Format final data for export        | Merge                    | Convert to CSV            |                                      |
| Convert to CSV         | Convert To File     | Convert data to CSV file             | Final Data               | -                        |                                      |
| Sticky Note            | Sticky Note         | -                                  | -                        | -                        |                                      |
| Sticky Note1           | Sticky Note         | -                                  | -                        | -                        |                                      |
| Sticky Note2           | Sticky Note         | -                                  | -                        | -                        |                                      |
| Sticky Note3           | Sticky Note         | -                                  | -                        | -                        |                                      |
| Sticky Note4           | Sticky Note         | -                                  | -                        | -                        |                                      |
| Sticky Note5           | Sticky Note         | -                                  | -                        | -                        |                                      |
| Sticky Note7           | Sticky Note         | -                                  | -                        | -                        |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: Form Trigger (Webhook)  
   - Configure: Create a webhook to receive form submissions with search parameters (e.g., location, type).  
   - Leave default parameters if no custom form UI is defined.

2. **Add a Set node named “Parse Form Data”**  
   - Use expressions to extract and store form fields into variables (e.g., city, category).  
   - Connect output from Form Trigger.

3. **Add a Set node named “Create Search Combinations”**  
   - Use JavaScript or expressions to build an array of search queries combining inputs (e.g., “Plumber in City A”, “Electrician in City B”).  
   - Connect from Parse Form Data.

4. **Add a Split Out node named “Split Into Searches”**  
   - Configure to split the array of search combinations into individual items.  
   - Connect from Create Search Combinations.

5. **Add an HTTP Request node named “Google Places Search1”**  
   - Method: GET  
   - URL: Google Places API endpoint for searching places.  
   - Parameters: query, location, radius, API key (stored as credential).  
   - Connect from Split Into Searches.  
   - Enable “Execute Once” to ensure single execution per input.

6. **Add a Split In Batches node named “Loop Over Items”**  
   - Batch size: set according to rate limits (e.g., 1-5).  
   - Connect from Google Places Search1.

7. **Add a Code node named “Extract Place Info”**  
   - Write JavaScript code to extract fields like place_id, name from the Places API response.  
   - Connect from Loop Over Items.

8. **Connect Extract Place Info back to Loop Over Items**  
   - This allows looping through all place results.

9. **Add a Wait node named “Wait (Rate Limit)”**  
   - Set delay (e.g., 1 second) to avoid API rate limits.  
   - Connect from Loop Over Items.

10. **Add an HTTP Request node named “Get Business Details”**  
    - Method: GET  
    - URL: Google Places Details API endpoint.  
    - Parameters: place_id, API key.  
    - Connect from Wait (Rate Limit).

11. **Add a Set node named “Merge Details”**  
    - Merge data from Get Business Details with previously extracted place info.  
    - Connect from Get Business Details.

12. **Add an If node named “Has Website?”**  
    - Condition: Check if “website” field exists and is not empty.  
    - Connect from Merge Details.

13. **Add a Split In Batches node named “Loop Over Items1”**  
    - Batch size: suitable for website scraping load (e.g., 1-3).  
    - Connect from the “true” output of Has Website?.

14. **Add an HTTP Request node named “Scrape Website1”**  
    - Method: GET  
    - URL: taken from the website field of business data.  
    - Configure to retry on fail and continue on error.  
    - Connect from Loop Over Items1.

15. **Add a Wait node named “Wait”**  
    - Delay between scrapes (e.g., 1-2 seconds).  
    - Connect from Scrape Website1.

16. **Connect Wait back to Loop Over Items1**  
    - Enables processing next batch.

17. **Add a Code node named “Extract Emails & Clean”**  
    - JavaScript code to extract emails from HTML content, remove duplicates and invalid emails.  
    - Connect from Loop Over Items1.

18. **Add a Set node named “No Website Fallback”**  
    - Set default or empty email fields for businesses without a website.  
    - Connect from the “false” output of Has Website?.

19. **Add a Merge node named “Merge”**  
    - Merge data from Extract Emails & Clean and No Website Fallback nodes.  
    - Configure to merge by index or key.  
    - Connect respective outputs to Merge.

20. **Add a Set node named “Final Data”**  
    - Select and format fields for output (name, address, phone, emails).  
    - Connect from Merge.

21. **Add a Convert To File node named “Convert to CSV”**  
    - Format: CSV  
    - Connect from Final Data.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                 |
|------------------------------------------------------------------------------|------------------------------------------------|
| The workflow uses Google Places API; ensure to have a valid API key with proper billing enabled to avoid quota issues. | Google Places API documentation: https://developers.google.com/maps/documentation/places/web-service/overview |
| Website scraping may fail due to website protections like CAPTCHAs or robots.txt; consider adding user-agent headers or proxy rotation. | General scraping best practices: https://www.scrapingbee.com/blog/web-scraping-best-practices/ |
| The workflow respects Google API rate limits by using Wait nodes; adjust delays based on your API quota and limits. | Google API usage limits: https://developers.google.com/maps/documentation/places/web-service/usage-and-billing |
| Emails extracted may contain duplicates or invalid addresses; the cleaning code filters these out for better lead quality. | Email validation regex example: https://emailregex.com/ |
| This workflow was designed with modularity allowing easy expansion for more data enrichment or alternative lead sources. | n8n documentation: https://docs.n8n.io/ |

---

This documentation provides a detailed, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting in n8n environments.