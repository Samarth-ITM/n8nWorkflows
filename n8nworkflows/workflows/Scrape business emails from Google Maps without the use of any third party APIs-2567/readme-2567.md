Scrape business emails from Google Maps without the use of any third party APIs

https://n8nworkflows.xyz/workflows/scrape-business-emails-from-google-maps-without-the-use-of-any-third-party-apis-2567


# Scrape business emails from Google Maps without the use of any third party APIs

### 1. Workflow Overview

This workflow is designed to scrape business email addresses from Google Maps search results without using any third-party APIs or paid services. It targets sales, marketing, and business development professionals who want a cost-effective lead generation method. The workflow accepts a list of search queries, iterates through each query, scrapes Google Maps search results to extract business URLs, fetches the HTML content of each business page, extracts email addresses via regular expressions, filters duplicates and irrelevant entries, and finally saves the clean email list to a Google Sheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Query Iteration:** Receives user-defined queries and processes them one by one.
- **1.2 Google Maps Scraping and URL Extraction:** Sends HTTP requests to Google Maps search URLs for each query and parses the response to extract business URLs.
- **1.3 URL Filtering and Deduplication:** Removes irrelevant and duplicate URLs to optimize subsequent processing.
- **1.4 Web Page Content Retrieval:** Fetches the HTML content of each business URL.
- **1.5 Email Extraction from HTML:** Runs regex-based extraction of email addresses from the page content.
- **1.6 Email Aggregation, Filtering, and Deduplication:** Aggregates all emails, removes duplicates and irrelevant emails.
- **1.7 Data Persistence:** Saves the final list of emails to a configured Google Sheets document.
- **1.8 Execution Control and Workflow Management:** Manages workflow execution timing and sub-workflow triggering for each query.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Query Iteration

- **Overview:**  
  This block allows the user to input a list of search queries (business keywords combined with location) and iterates through them sequentially, triggering the scraping process for each.

- **Nodes Involved:**  
  - Run workflow (Manual Trigger)  
  - Loop over queries (SplitInBatches)  
  - Execute scraper for query (ExecuteWorkflow)  
  - Wait between executions (Wait)  

- **Node Details:**

  - **Run workflow**  
    - Type: Manual Trigger  
    - Role: Entry point for user to input queries as JSON objects with a `query` field (e.g., "hollywood+dentist").  
    - Config: User provides an array of queries.  
    - Input/Output: Outputs the list of queries to "Loop over queries".  
    - Edge Cases: Empty or malformed query input may cause no results or errors downstream.

  - **Loop over queries**  
    - Type: SplitInBatches  
    - Role: Processes each query individually in a batch of size 1 (default).  
    - Config: Default batch options; processes queries sequentially.  
    - Input: Receives array of queries from "Run workflow".  
    - Output: For each query, triggers "Execute scraper for query" sub-workflow.  
    - Edge Cases: Large query lists may cause long execution times; consider batch size tuning.

  - **Execute scraper for query**  
    - Type: ExecuteWorkflow  
    - Role: Triggers the same workflow as a sub-workflow for each individual query.  
    - Config: Mode set to "each"; workflow ID set to current workflow (self). Does not wait for sub-workflow completion.  
    - Input: Receives one query per execution.  
    - Output: Triggers "Wait between executions" node.  
    - Edge Cases: Recursive calls can cause concurrency issues if not managed properly.  
    - Version: Requires n8n version supporting ExecuteWorkflow node (v0.120+).

  - **Wait between executions**  
    - Type: Wait  
    - Role: Introduces a pause (default 2 seconds) between processing each query to prevent rate-limiting or server overload.  
    - Config: Wait time configurable; default is 2 seconds.  
    - Input: Triggered after each query sub-workflow execution.  
    - Output: Loops back to "Loop over queries" for next query.  
    - Edge Cases: Too short wait time can trigger Google anti-bot measures; too long increases total runtime.

---

#### 2.2 Google Maps Scraping and URL Extraction

- **Overview:**  
  For each query, this block sends an HTTP GET request to Google Maps search, retrieves the HTML or response data, and extracts URLs of business listings using regex.

- **Nodes Involved:**  
  - Starts scraper workflow (ExecuteWorkflowTrigger)  
  - Search Google Maps with query (HTTP Request)  
  - Scrape URLs from results (Code)  
  - Filter irrelevant URLs (Filter)  
  - Remove Duplicate URLs (RemoveDuplicates)

- **Node Details:**

  - **Starts scraper workflow**  
    - Type: ExecuteWorkflowTrigger  
    - Role: Invokes the scraping workflow for each query.  
    - Config: No special parameters; triggers "Search Google Maps with query" node.  
    - Edge Cases: Requires workflow to be active for triggering.

  - **Search Google Maps with query**  
    - Type: HTTP Request  
    - Role: Makes a GET request to Google Maps search using the query string.  
    - Config: URL set dynamically as `https://www.google.com/maps/search/{{ $json.query }}`; no authentication.  
    - Input: Receives query from trigger.  
    - Output: Raw response data (HTML/text) sent to "Scrape URLs from results".  
    - Edge Cases:  
      - Google may block automated requests with CAPTCHA or rate limits.  
      - Network timeouts or errors.  
      - Changes in Google Maps HTML structure may break regex extraction.

  - **Scrape URLs from results**  
    - Type: Code  
    - Role: Parses the HTTP response data using regex to extract all URLs.  
    - Config: Uses regex `/https?:\/\/[^\/]+/g` to match URLs at domain level.  
    - Input: Receives HTTP response data as `data`.  
    - Output: Array of URL objects `{url: <extracted_url>}`.  
    - Edge Cases:  
      - Regex may capture irrelevant URLs or incomplete URLs.  
      - Empty or malformed HTML may yield no URLs.

  - **Filter irrelevant URLs**  
    - Type: Filter  
    - Role: Removes URLs containing unwanted domains or patterns (e.g., google, gstatic, schema.org).  
    - Config: Negative regex filter on URL field; excludes URLs matching `(google|gstatic|ggpht|schema\.org|example\.com|sentry-next\.wixpress\.com|imli\.com|sentry\.wixpress\.com|ingest\.sentry\.io)`.  
    - Input: URLs from previous node.  
    - Output: Only relevant business URLs passed forward.  
    - Edge Cases: Overly broad regex may exclude relevant URLs; too narrow may allow noise.

  - **Remove Duplicate URLs**  
    - Type: RemoveDuplicates  
    - Role: Eliminates duplicate URLs to avoid redundant processing.  
    - Config: Default deduplication on entire URL string.  
    - Input: Filtered URLs.  
    - Output: Unique URLs forwarded for web page requests.  
    - Edge Cases: Minor URL variations may not be caught as duplicates.

---

#### 2.3 URL Filtering and Deduplication

- **Overview:**  
  This block ensures only unique and relevant business URLs are processed further to optimize resource usage.

- **Nodes Involved:**  
  - Remove Duplicate URLs (already described above)  
  - Loop over URLs (SplitInBatches)

- **Node Details:**

  - **Loop over URLs**  
    - Type: SplitInBatches  
    - Role: Processes each URL individually or in small batches.  
    - Config: Default batch size; does not reset between batches.  
    - Input: Unique URLs from "Remove Duplicate URLs".  
    - Output: Sequentially sends URLs to "Request web page for URL".  
    - Edge Cases: Large URL lists may cause slow processing.

---

#### 2.4 Web Page Content Retrieval

- **Overview:**  
  Fetches the HTML content of each business listing URL for email extraction.

- **Nodes Involved:**  
  - Request web page for URL (HTTP Request)  
  - Loop over pages (SplitInBatches)

- **Node Details:**

  - **Request web page for URL**  
    - Type: HTTP Request  
    - Role: Downloads the HTML content of each business URL.  
    - Config: URL set dynamically from input JSON `url` field; no authentication.  
    - Input: Individual URLs from "Loop over URLs".  
    - Output: HTML content sent to "Loop over pages" and ultimately to email extraction.  
    - Error Handling: On error, continues regular output to avoid breaking the workflow.  
    - Edge Cases:  
      - 404 or 5xx errors from server.  
      - Anti-bot or CAPTCHA may block requests.  
      - Network timeouts.

  - **Loop over pages**  
    - Type: SplitInBatches  
    - Role: Processes each fetched page content individually for email scraping.  
    - Config: Default batch options.  
    - Input: HTML content from "Request web page for URL".  
    - Output: Forwards data to "Scrape emails from page" and "Aggregate arrays of emails".  
    - Edge Cases: Large page sets may slow processing.

---

#### 2.5 Email Extraction from HTML

- **Overview:**  
  Extracts email addresses from the HTML content using a JavaScript regex.

- **Nodes Involved:**  
  - Scrape emails from page (Code)  
  - Aggregate arrays of emails (Aggregate)  
  - Split out into default data structure (SplitOut)  

- **Node Details:**

  - **Scrape emails from page**  
    - Type: Code  
    - Role: Executes custom JavaScript to extract email addresses from HTML data.  
    - Config: Runs once per item, regex used is `/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!png|jpg|gif|jpeg)[a-zA-Z]{2,}/g` to exclude image file extensions.  
    - Input: Receives `data` field containing HTML content.  
    - Output: `{emails: [...array of matched emails...]}`  
    - Error Handling: Continues on error to avoid workflow failure.  
    - Edge Cases:  
      - False positives or missed emails due to regex limitations.  
      - HTML encoding could affect matching.

  - **Aggregate arrays of emails**  
    - Type: Aggregate  
    - Role: Merges arrays of emails from multiple pages into a single list.  
    - Config: Merges lists from the `emails` field.  
    - Input: Multiple email arrays from "Scrape emails from page".  
    - Output: Single aggregated array of emails.  
    - Edge Cases: Large datasets may impact performance.

  - **Split out into default data structure**  
    - Type: SplitOut  
    - Role: Converts the aggregated emails array into individual items for further processing.  
    - Config: Splits by the `emails` field.  
    - Input: Aggregated email list.  
    - Output: Individual email entries.  
    - Edge Cases: Empty arrays result in no output items.

---

#### 2.6 Email Aggregation, Filtering, and Deduplication

- **Overview:**  
  This block cleans the extracted emails by removing duplicates and filtering out irrelevant or invalid emails before saving.

- **Nodes Involved:**  
  - Remove duplicate emails (RemoveDuplicates)  
  - Filter irrelevant emails (Filter)

- **Node Details:**

  - **Remove duplicate emails**  
    - Type: RemoveDuplicates  
    - Role: Removes duplicate email entries based on the email address.  
    - Config: Compares on `emails` field (email string).  
    - Input: Individual email items from "Split out into default data structure".  
    - Output: Unique email items.  
    - Edge Cases: Case sensitivity or whitespace variations might cause duplicates to remain.

  - **Filter irrelevant emails**  
    - Type: Filter  
    - Role: Excludes emails matching known irrelevant domains or patterns, e.g., Google-related or tracking domains.  
    - Config: Negative regex filter on `emails` field excluding `(google|gstatic|ggpht|schema\.org|example\.com|sentry\.wixpress\.com|sentry-next\.wixpress\.com|ingest\.sentry\.io|sentry\.io|imli\.com)`.  
    - Input: Deduplicated emails.  
    - Output: Clean email list for saving.  
    - Edge Cases: Overly broad regex may remove legitimate emails.

---

#### 2.7 Data Persistence

- **Overview:**  
  Saves the finalized email list to a user-specified Google Sheets document for easy access and further use.

- **Nodes Involved:**  
  - Save emails to Google Sheet (Google Sheets)

- **Node Details:**

  - **Save emails to Google Sheet**  
    - Type: Google Sheets  
    - Role: Appends extracted emails to a selected Google Sheets document and sheet.  
    - Config:  
      - Operation: Append.  
      - Columns: Maps `emails` field to "Emails" column in sheet.  
      - Matching columns used to avoid duplicates in sheet.  
    - Input: Filtered emails from previous node.  
    - Output: Success or failure status.  
    - Credential: Requires Google Sheets OAuth2 credentials configured.  
    - Edge Cases:  
      - Credential expiration or permission issues.  
      - API rate limits.  
      - Sheet or document not found.

---

#### 2.8 Execution Control and Workflow Management

- **Overview:**  
  Manages the overall execution flow, including initial manual triggering and sub-workflow orchestration for each query.

- **Nodes Involved:**  
  - Run workflow (Manual Trigger)  
  - Execute scraper for query (ExecuteWorkflow)  
  - Wait between executions (Wait)  
  - Starts scraper workflow (ExecuteWorkflowTrigger)

- **Node Details:**  
  Covered in sections 2.1 and 2.2.

---

### 3. Summary Table

| Node Name                 | Node Type                | Functional Role                         | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                  |
|---------------------------|--------------------------|---------------------------------------|----------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| Run workflow              | Manual Trigger           | Entry point for inputting queries     |                            | Loop over queries              | ## 🛠 Setup 1. Setup your list of queries in the "Run workflow" manual trigger node. Watch this [video](https://youtu.be/HaiO-UeiKBA) on how to generate the queries with ChatGPT. 3. Choose a sheet to populate with data in the **Google Sheets node** 4. Run the workflow and start getting leads into your Google Sheets document |
| Loop over queries         | SplitInBatches           | Iterate over queries one by one        | Run workflow               | Execute scraper for query, (empty) | **Optional** 👇 Set wait time between each query workflow execution. Default is 2 seconds.   |
| Execute scraper for query | ExecuteWorkflow          | Runs scraping sub-workflow per query  | Loop over queries          | Wait between executions        | ### Scraper 👇 This workflow will be executed in the background for each query. Click the **All executions** tab in the left sidebar to see the executions running. |
| Wait between executions   | Wait                     | Pauses between query executions       | Execute scraper for query  | Loop over queries              | **Optional** 👇 Set wait time between each query workflow execution. Default is 2 seconds.   |
| Starts scraper workflow   | ExecuteWorkflowTrigger   | Triggers scraping workflow             |                            | Search Google Maps with query  | ### Scraper 👇 This workflow will be executed in the background for each query.              |
| Search Google Maps with query | HTTP Request            | Requests Google Maps search page       | Starts scraper workflow    | Scrape URLs from results       |                                                                                            |
| Scrape URLs from results  | Code                     | Extracts URLs from HTML response       | Search Google Maps with query | Filter irrelevant URLs         |                                                                                            |
| Filter irrelevant URLs    | Filter                   | Filters out irrelevant or blocked URLs| Scrape URLs from results   | Remove Duplicate URLs          | **Optional** 👆 Add or change the regex for filtering irrelevant URLs.                      |
| Remove Duplicate URLs     | RemoveDuplicates         | Removes duplicate URLs                  | Filter irrelevant URLs     | Loop over URLs                 |                                                                                            |
| Loop over URLs            | SplitInBatches           | Iterates over each URL                  | Remove Duplicate URLs      | Request web page for URL, Loop over pages |                                                                                            |
| Request web page for URL  | HTTP Request             | Fetches business HTML pages             | Loop over URLs             | Loop over pages                |                                                                                            |
| Loop over pages           | SplitInBatches           | Iterates over pages for email scraping | Loop over URLs, Request web page for URL | Aggregate arrays of emails, Scrape emails from page |                                                                                            |
| Scrape emails from page   | Code                     | Extracts emails from HTML content       | Loop over pages            | Loop over pages               | **Optional** 👆 Add or change the regex for filtering irrelevant/incorrect email addresses. |
| Aggregate arrays of emails| Aggregate                | Merges email arrays into one list       | Loop over pages            | Split out into default data structure |                                                                                            |
| Split out into default data structure | SplitOut                 | Converts array of emails into items     | Aggregate arrays of emails | Remove duplicate emails        |                                                                                            |
| Remove duplicate emails   | RemoveDuplicates         | Removes duplicate email addresses        | Split out into default data structure | Filter irrelevant emails      |                                                                                            |
| Filter irrelevant emails  | Filter                   | Filters out unwanted or invalid emails  | Remove duplicate emails    | Save emails to Google Sheet    |                                                                                            |
| Save emails to Google Sheet | Google Sheets           | Appends emails to Google Sheets document| Filter irrelevant emails   |                              | 👆 1. Setup your **credentials**. Here's a [video tutorial](https://youtu.be/O5RnWDM27M8) on how to do that. 2. Choose which document and sheet to save the scraped emails to. |
| Sticky Note               | StickyNote               | Provides setup and usage guidance       |                            |                                | ## 🛠 Setup 1. Setup your list of queries in the "Run workflow" manual trigger node. Watch  this [video](https://youtu.be/HaiO-UeiKBA) on how to generate the queries with ChatGPT. 3. Choose a sheet to populate with data in the **Google Sheets node** 4. Run the workflow and start getting leads into your Google Sheets document |
| Sticky Note2              | StickyNote               | Notes on optional wait time setting     |                            |                                | **Optional** 👇 Set wait time between each query workflow execution. Default is 2 seconds.   |
| Sticky Note3              | StickyNote               | Explains sub-workflow executions        |                            |                                | ### Scraper 👇 This workflow will be executed in the background for each query. Click the **All executions** tab in the left sidebar to see the executions running. |
| Sticky Note4              | StickyNote               | Setup instructions for credentials      |                            |                                | 👆 1. Setup your **credentials**. Here's a [video tutorial](https://youtu.be/O5RnWDM27M8) on how to do that. 2. Choose which document and sheet to save the scraped emails to. |
| Sticky Note5              | StickyNote               | External video resource                  |                            |                                | ## ⚠️ Note A [video tutorial](https://youtu.be/HaiO-UeiKBA) for this workflow guide is available on my [Youtube channel](https://www.youtube.com/channel/UCn8xmUBunez1SsDVRfZDUGA) |
| Sticky Note6              | StickyNote               | Workflow purpose summary                 |                            |                                | ## Google Maps Automatic Email Scraper This workflow automatically scrapes emails from businesses on Google Maps based on a list of queries that you provide. |
| Sticky Note7              | StickyNote               | Optional regex customization note for URL filtering |                            |                                | **Optional** 👆 Add or change the regex for filtering irrelevant URLs.                      |
| Sticky Note8              | StickyNote               | Optional regex customization note for email filtering |                            |                                | **Optional** 👆 Add or change the regex for filtering irrelevant/incorrect email addresses. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: Run workflow  
   - Role: Input list of queries  
   - Configuration: Add JSON input with field `query` for each search term (e.g., `{"query":"hollywood+dentist"}`).  

2. **Add SplitInBatches Node**  
   - Name: Loop over queries  
   - Role: Iterate queries one by one  
   - Configuration: Default batch size (1). Connect input from "Run workflow".

3. **Add ExecuteWorkflow Node**  
   - Name: Execute scraper for query  
   - Role: Trigger sub-workflow for each query  
   - Configuration: Set mode to "each". Set workflow ID to current workflow (self-reference). Connect output of "Loop over queries" to this node.

4. **Add Wait Node**  
   - Name: Wait between executions  
   - Role: Wait 2 seconds between queries (configurable)  
   - Configuration: Set amount to 2 seconds. Connect output of "Execute scraper for query" to this node.

5. **Connect Wait Node back to Loop over queries**  
   - Forms a loop that iterates over all queries with delay.

6. **Add ExecuteWorkflowTrigger Node**  
   - Name: Starts scraper workflow  
   - Role: Initiates scraping process per query  
   - Configuration: Connect output to "Search Google Maps with query".

7. **Add HTTP Request Node**  
   - Name: Search Google Maps with query  
   - Role: Query Google Maps search page  
   - Configuration: URL: `https://www.google.com/maps/search/{{ $json.query }}`  
   - Connect input from "Starts scraper workflow".

8. **Add Code Node**  
   - Name: Scrape URLs from results  
   - Role: Extract URLs from page source  
   - Configuration: JavaScript code:  
     ```js
     const data = $input.first().json.data;
     const regex = /https?:\/\/[^\/]+/g;
     const urls = data.match(regex);
     return urls.map(url => ({json: {url: url}}));
     ```  
   - Connect input from "Search Google Maps with query".

9. **Add Filter Node**  
   - Name: Filter irrelevant URLs  
   - Role: Remove known irrelevant URLs  
   - Configuration: Filter to exclude URLs matching regex `(google|gstatic|ggpht|schema\.org|example\.com|sentry-next\.wixpress\.com|imli\.com|sentry\.wixpress\.com|ingest\.sentry\.io)`  
   - Connect input from "Scrape URLs from results".

10. **Add RemoveDuplicates Node**  
    - Name: Remove Duplicate URLs  
    - Role: Deduplicate URLs  
    - Configuration: Default, based on full URL string  
    - Connect input from "Filter irrelevant URLs".

11. **Add SplitInBatches Node**  
    - Name: Loop over URLs  
    - Role: Process URLs individually  
    - Configuration: Default batch size  
    - Connect input from "Remove Duplicate URLs".

12. **Add HTTP Request Node**  
    - Name: Request web page for URL  
    - Role: Fetch HTML content of each URL  
    - Configuration: URL: `={{ $json.url }}`  
    - Error Handling: Continue on error  
    - Connect input from "Loop over URLs".

13. **Add SplitInBatches Node**  
    - Name: Loop over pages  
    - Role: Process each fetched page for emails  
    - Configuration: Default batch size  
    - Connect input from "Request web page for URL" and also from "Loop over URLs" (parallel).

14. **Add Code Node**  
    - Name: Scrape emails from page  
    - Role: Extract emails via regex  
    - Configuration: JavaScript code:  
      ```js
      const data = $json.data;
      const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.(?!png|jpg|gif|jpeg)[a-zA-Z]{2,}/g;
      const emails = data.match(emailRegex);
      return {json: {emails: emails}};
      ```  
    - Error Handling: Continue on error  
    - Connect input from "Loop over pages".

15. **Add Aggregate Node**  
    - Name: Aggregate arrays of emails  
    - Role: Merge email arrays into one list  
    - Configuration: Merge lists on field "emails"  
    - Connect input from "Loop over pages".

16. **Add SplitOut Node**  
    - Name: Split out into default data structure  
    - Role: Convert array of emails into individual items  
    - Configuration: Split on field "emails"  
    - Connect input from "Aggregate arrays of emails".

17. **Add RemoveDuplicates Node**  
    - Name: Remove duplicate emails  
    - Role: Remove duplicate email addresses  
    - Configuration: Compare on field "emails"  
    - Connect input from "Split out into default data structure".

18. **Add Filter Node**  
    - Name: Filter irrelevant emails  
    - Role: Remove emails from irrelevant domains  
    - Configuration: Filter excludes emails matching `(google|gstatic|ggpht|schema\.org|example\.com|sentry\.wixpress\.com|sentry-next\.wixpress\.com|ingest\.sentry\.io|sentry\.io|imli\.com)`  
    - Connect input from "Remove duplicate emails".

19. **Add Google Sheets Node**  
    - Name: Save emails to Google Sheet  
    - Role: Append emails to a Google Sheets document  
    - Configuration:  
      - Operation: Append  
      - Map field "emails" to column "Emails"  
      - Matching column: "Emails"  
    - Credential: Configure Google Sheets OAuth2 credentials with edit access  
    - Connect input from "Filter irrelevant emails".

20. **Add Sticky Notes** (Optional)  
    - Add informational sticky notes as per original workflow for guidance.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Setup your list of queries in the "Run workflow" manual trigger node. Watch this [video](https://youtu.be/HaiO-UeiKBA) on how to generate the queries with ChatGPT. | Setup instructions, video tutorial |
| Here's a [video tutorial](https://youtu.be/O5RnWDM27M8) on how to setup Google Sheets credentials in n8n. | Credential configuration |
| A [video tutorial](https://youtu.be/HaiO-UeiKBA) for this workflow guide is available on my [Youtube channel](https://www.youtube.com/channel/UCn8xmUBunez1SsDVRfZDUGA). | General workflow tutorial |
| Optional: Customize regex expressions in "Scrape emails from page" and "Filter irrelevant URLs" nodes to refine scraping accuracy. | Regex customization |
| Use the **All executions** tab in the n8n sidebar to monitor sub-workflow executions for each query. | Execution monitoring |

---

This document provides a detailed, structured reference of the "Google Maps Email Scraper Template" workflow, enabling users and automation agents to understand, reproduce, and modify it confidently.