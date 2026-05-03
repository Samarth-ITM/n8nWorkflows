Automated Oncrawl Sitemap URL Submission to Google Indexing API & IndexNow

https://n8nworkflows.xyz/workflows/automated-oncrawl-sitemap-url-submission-to-google-indexing-api---indexnow-11837


# Automated Oncrawl Sitemap URL Submission to Google Indexing API & IndexNow

### 1. Workflow Overview

This workflow automates the submission of URLs from Oncrawl sitemaps to the Google Indexing API and the IndexNow protocol. It targets SEO professionals and technical teams aiming to keep search engines promptly updated about new or changed URLs for improved indexing. The workflow is structured into the following logical blocks:

- **1.1 Input Reception & Configuration:** Receives webhook notifications from Oncrawl about crawl completion and sets configuration parameters.
- **1.2 Sitemap Retrieval and Parsing:** Retrieves sitemaps declared in robots.txt or crawl configurations, parses them, and extracts URLs with metadata.
- **1.3 URL Filtering & Preparation:** Filters URLs based on last modification dates or crawl comparisons and prepares them for submission.
- **1.4 Google Indexing API Submission:** Checks URL indexing status via Google Search Console API, decides if submission is needed, and submits updated URLs.
- **1.5 IndexNow Submission:** Splits URLs into batches, builds payloads, and submits URLs to the IndexNow API.
- **1.6 Orphan Pages Handling (Variation A & B):** Retrieves and merges orphan pages data for indexing, integrating with the main sitemap URL flow.
- **1.7 Rate Limiting & Timing Controls:** Implements wait nodes to avoid API rate limits and randomizes delays between submissions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration

**Overview:**  
This block starts the workflow via an HTTP webhook notified when an Oncrawl crawl finishes. It sets essential configuration parameters used throughout the workflow such as URLs, API keys, batch sizes, and feature toggles.

**Nodes Involved:**  
- Webhook  
- Post - Get Sitemaps  
- Config  
- Sticky Note (explanatory)  

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook  
  - Role: Entry point receiving POST requests from Oncrawl crawl completion notifications.  
  - Configuration: Listens for POST, response mode set to responseNode.  
  - Input: External Oncrawl webhook call.  
  - Output: JSON payload with crawl and project details.

- **Post - Get Sitemaps**  
  - Type: HTTP Request (POST)  
  - Role: Calls Oncrawl Discover_sitemaps API to fetch sitemap URLs for the project/crawl.  
  - Configuration: Dynamically builds JSON body with crawl configuration data from webhook input.  
  - Headers: Content-Type: application/json.  
  - Output: API response with sitemaps.

- **Config**  
  - Type: Set  
  - Role: Defines and stores key parameters for the workflow, including crawl ID, site URL, sitemap URL, batch size, feature toggles (USE_GOOGLE, USE_INDEXNOW), and IndexNow API keys.  
  - Configuration: Assigns variables using expressions reading from previous node outputs or static values.  
  - Output: Configuration data for downstream nodes.

- **Sticky Note**  
  - Purpose: Documents usage of the Discover_sitemaps endpoint.

**Edge Cases / Failures:**  
- Webhook: Missing or malformed payloads from Oncrawl.  
- Post - Get Sitemaps: API errors, authentication failures, or invalid crawl data could prevent sitemap retrieval.  
- Config: Misconfiguration or missing environment variables could halt further processing.

---

#### 2.2 Sitemap Retrieval and Parsing

**Overview:**  
This block downloads the sitemap.xml file(s), converts XML to JSON, and extracts URL entries for further processing.

**Nodes Involved:**  
- Get sitemap.xml  
- Convert sitemap to JSON  
- Get content-specific sitemaps  
- Get content of each sitemap  
- convert page data to JSON  
- Force urlset.url to array  
- Split Out  
- Sort  
- Assign mandatory sitemap fields  
- Filter: lastmod within DAYS_BACK  
- Sticky Notes (documentation)  

**Node Details:**

- **Get sitemap.xml**  
  - Type: HTTP Request (GET)  
  - Role: Fetches the sitemap XML from the configured SITEMAP_URL.  
  - Input: Config node output for URL.  
  - Output: Raw XML sitemap.

- **Convert sitemap to JSON**  
  - Type: XML  
  - Role: Converts raw XML sitemap to JSON for easier data manipulation.

- **Get content-specific sitemaps**  
  - Type: SplitOut  
  - Role: Extracts nested sitemaps from sitemapindex.sitemap field for recursive processing.

- **Get content of each sitemap**  
  - Type: HTTP Request  
  - Role: Fetches each nested sitemap XML.

- **convert page data to JSON**  
  - Type: XML  
  - Role: Converts nested sitemap XML to JSON.

- **Force urlset.url to array**  
  - Type: Set  
  - Role: Ensures the urlset.url field is always an array to avoid processing errors.

- **Split Out**  
  - Type: SplitOut  
  - Role: Extracts individual URL entries from the sitemap JSON array.

- **Sort**  
  - Type: Sort  
  - Role: Sorts URL entries by lastmod date descending to prioritize recent updates.

- **Assign mandatory sitemap fields**  
  - Type: Set  
  - Role: Ensures that each URL item has loc and lastmod fields for further filtering.

- **Filter: lastmod within DAYS_BACK**  
  - Type: Code  
  - Role: Filters URLs modified within the configured number of days (DAYS_BACK).  
  - Logic: Converts lastmod to Date, compares with cutoff date = now - DAYS_BACK.

- **Sticky Notes**  
  - Document the sitemap parsing logic and lastmod filtering.

**Edge Cases / Failures:**  
- HTTP Request: Sitemap URL unreachable or invalid XML format.  
- XML Parsing: Malformed XML can cause failures.  
- Empty or missing lastmod fields may cause filtering issues.  
- Large sitemaps might cause memory/time constraints.

---

#### 2.3 URL Filtering & Preparation

**Overview:**  
After filtering URLs by lastmod, this block gates the process based on feature toggles and prepares payloads for IndexNow submissions. It also handles batching.

**Nodes Involved:**  
- Gate: IndexNow  
- Gate: Google  
- Split In Batches (IndexNow ≤500)1  
- Build IndexNow payload  
- IndexNow Submit  
- Wait (IndexNow jitter)1  
- Sticky Note (IndexNow autosubmitting explanation)  

**Node Details:**

- **Gate: IndexNow**  
  - Type: Code  
  - Role: Filters items if USE_INDEXNOW is false in Config, effectively disabling IndexNow submission.  
  - Logic: Checks boolean flag, passes items only if true.

- **Gate: Google**  
  - Type: Code  
  - Role: Similar gating logic for Google Indexing API.

- **Split In Batches (IndexNow ≤500)1**  
  - Type: SplitInBatches  
  - Role: Splits URLs into batches of max 500 (or configured BATCH_SIZE capped at 500) for IndexNow API limits.

- **Build IndexNow payload**  
  - Type: Code  
  - Role: Builds JSON payload with host, API key, key location, and list of URLs filtered by host and protocol.  
  - Logic:  
    - Sanitizes URLs (trim, absolute http/https only).  
    - Matches URLs with configured SITE_URL host.  
    - Deduplicates URLs.  
    - Returns empty if no valid URLs or host.

- **IndexNow Submit**  
  - Type: HTTP Request (POST)  
  - Role: Submits the payload to IndexNow API endpoint (https://api.indexnow.org/indexnow).  
  - Config: Sends full JSON body from previous node.

- **Wait (IndexNow jitter)1**  
  - Type: Wait  
  - Role: Adds randomized small delay (0.25 to 1 second) between batches to reduce API rate limiting risks.

- **Sticky Note8**  
  - Documents the IndexNow submission logic.

**Edge Cases / Failures:**  
- Gate nodes: Misconfigured flags could skip entire submission.  
- Payload building: Missing or incorrect SITE_URL or INDEXNOW_KEY disables submission.  
- HTTP Request: API downtime, invalid key, or malformed payload cause submission failure.  
- Batch splitting: Empty batches or uneven splits.

---

#### 2.4 Google Indexing API Submission

**Overview:**  
This block checks the indexing status of each URL via Google Search Console and submits only new or updated URLs to the Google Indexing API.

**Nodes Involved:**  
- Gate: Google  
- Loop Over Items (batches)  
- Check status  
- Switch  
- is new? (If node)  
- URL Updated  
- Wait2  
- Sticky Note12 (Google indexing instructions and authentication details)  

**Node Details:**

- **Gate: Google**  
  - Same as in 2.3, gating Google-related processing.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes URLs one at a time or in small batches for Google API calls.

- **Check status**  
  - Type: HTTP Request (POST)  
  - Role: Calls Google Search Console URL Inspection API to check coverageState and last crawl time of each URL.  
  - Config: Sends form-urlencoded body with inspectionUrl and siteUrl.  
  - Authentication: Uses Google Service Account credentials configured for indexing API access.

- **Switch**  
  - Type: Switch  
  - Role: Routes URLs based on coverageState in API response.  
  - Cases:  
    - "Submitted and indexed" → URL is indexed, check if newly updated.  
    - "Crawled - currently not indexed" → URL needs submission.

- **is new?**  
  - Type: If  
  - Role: Compares lastmod of URL with last crawl time to decide if URL is new or updated.  
  - Logic: lastmod > lastCrawlTime → true (submit), else false (skip).

- **URL Updated**  
  - Type: HTTP Request (POST)  
  - Role: Submits URL update notification to Google Indexing API endpoint (https://indexing.googleapis.com/v3/urlNotifications:publish).  
  - Payload: url and type=URL_UPDATED.

- **Wait2**  
  - Type: Wait  
  - Role: Introduces randomized delay (0.3 to 1.5 seconds) between Google API calls to avoid rate-limiting.

- **Sticky Note12**  
  - Detailed instructions for setting up Google Service Account credentials and necessary permissions.

**Edge Cases / Failures:**  
- Google API authentication errors (invalid key, insufficient permissions).  
- Rate limiting by Google API if delays are insufficient.  
- API response parsing errors if response schema changes.  
- Network timeout or errors on HTTP requests.

---

#### 2.5 Orphan Pages Handling (Variation A & B)

**Overview:**  
Handles indexing of orphan pages — URLs discovered but not linked normally — either from Oncrawl logs or crawl comparisons, enriching them with lastmod data and merging with sitemap URLs.

**Nodes Involved:**  
- Config - Orphan  
- Get Orphan Pages  
- Split Out - Orphan1  
- Split Out - Orphan2  
- Merge  
- Get Crawl over crawl  
- Split Out - Coc1  
- Merge1  
- Sticky Notes10, 11  

**Node Details:**

- **Config - Orphan**  
  - Type: Set  
  - Role: Similar to Config node but specifically for orphan pages.  
  - Sets IndexNow keys, daysBack, batch size, and URLs.

- **Get Orphan Pages**  
  - Type: HTTP Request (POST)  
  - Role: Calls Oncrawl Data API with OQL query to retrieve orphan pages from logs or sitemaps with depth null and sources filters.

- **Split Out - Orphan1 / Split Out - Orphan2**  
  - Type: SplitOut  
  - Role: Extracts URLs array and then individual URLs from orphan pages API response.

- **Merge**  
  - Type: Merge  
  - Role: Combines orphan URLs with previously assigned sitemap fields by matching loc and url fields to recover lastmod dates.

- **Get Crawl over crawl**  
  - Type: HTTP Request (POST)  
  - Role: Retrieves pages newly added between two crawl runs with specific indexability and status code conditions.

- **Split Out - Coc1**  
  - Type: SplitOut  
  - Role: Extracts URLs from crawl over crawl response.

- **Merge1**  
  - Type: Merge  
  - Role: Combines crawl-over-crawl URLs with assigned sitemap fields to enrich data.

- **Sticky Notes10 & 11**  
  - Document the orphan pages logic and crawl-over-crawl variations, including API docs and usage notes.

**Edge Cases / Failures:**  
- API rate limits or failures in Oncrawl Data API calls.  
- Mismatched or missing lastmod data causing incorrect filtering or merges.  
- Large result sets causing performance issues.  
- Configuration errors in OQL queries.

---

#### 2.6 Rate Limiting & Timing Controls

**Overview:**  
This block involves wait nodes that add jittered delays between API calls and batches to prevent rate limiting or throttling by Google and IndexNow.

**Nodes Involved:**  
- Wait (IndexNow jitter)1  
- Wait2  

**Node Details:**

- **Wait (IndexNow jitter)1**  
  - Adds 0.25 to 1 second random delay between IndexNow batch submissions.

- **Wait2**  
  - Adds 0.3 to 1.5 seconds random delay between Google URL status checks and submissions.

**Edge Cases / Failures:**  
- Insufficient wait durations might cause API rate limiting.  
- Excessive delays can slow down the workflow unnecessarily.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                    | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                                  |
|--------------------------------|------------------------|---------------------------------------------------|--------------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Webhook                        | HTTP Webhook           | Entry point receiving crawl completion webhook    | External                       | Post - Get Sitemaps                    | Documents webhook usage and Oncrawl notification endpoint.                                                   |
| Post - Get Sitemaps            | HTTP Request (POST)    | Fetches sitemaps from Oncrawl API                  | Webhook                       | Config                               |                                                                                                              |
| Config                         | Set                    | Sets global parameters and flags                   | Post - Get Sitemaps            | Get sitemap.xml                       | Explains Config variables to update for site, keys, toggles.                                                |
| Get sitemap.xml                | HTTP Request (GET)     | Downloads main sitemap XML                          | Config                        | Convert sitemap to JSON               |                                                                                                              |
| Convert sitemap to JSON        | XML                    | Parses sitemap XML to JSON                          | Get sitemap.xml               | Get content-specific sitemaps        |                                                                                                              |
| Get content-specific sitemaps  | SplitOut               | Extract nested sitemaps                             | Convert sitemap to JSON       | Get content of each sitemap           |                                                                                                              |
| Get content of each sitemap    | HTTP Request (GET)     | Downloads each nested sitemap                       | Get content-specific sitemaps | convert page data to JSON             |                                                                                                              |
| convert page data to JSON      | XML                    | Parses nested sitemap XML                           | Get content of each sitemap    | Force urlset.url to array             |                                                                                                              |
| Force urlset.url to array      | Set                    | Ensures URL list is always an array                 | convert page data to JSON      | Split Out                           |                                                                                                              |
| Split Out                     | SplitOut               | Splits URL entries for processing                   | Force urlset.url to array      | Sort                                |                                                                                                              |
| Sort                          | Sort                   | Sorts URLs by last modification date descending    | Split Out                    | Assign mandatory sitemap fields       |                                                                                                              |
| Assign mandatory sitemap fields| Set                    | Assigns loc and lastmod fields to URLs              | Sort                        | Filter: lastmod within DAYS_BACK      |                                                                                                              |
| Filter: lastmod within DAYS_BACK| Code                   | Filters URLs modified within DAYS_BACK days         | Assign mandatory sitemap fields | Gate: IndexNow, Gate: Google          | Explains filtering to prevent API rate limiting.                                                             |
| Gate: IndexNow                | Code                   | Conditional gate for IndexNow usage                  | Filter: lastmod within DAYS_BACK | Split In Batches (IndexNow ≤500)1     | Details IndexNow usage toggle.                                                                               |
| Split In Batches (IndexNow ≤500)1| SplitInBatches         | Splits URLs into batches of max 500 for IndexNow    | Gate: IndexNow               | Build IndexNow payload                |                                                                                                              |
| Build IndexNow payload         | Code                   | Builds JSON payload for IndexNow API                 | Split In Batches (IndexNow ≤500)1 | IndexNow Submit                    |                                                                                                              |
| IndexNow Submit               | HTTP Request (POST)    | Submits batch payload to IndexNow API                | Build IndexNow payload        | Wait (IndexNow jitter)1              |                                                                                                              |
| Wait (IndexNow jitter)1       | Wait                   | Adds randomized delay between IndexNow batch calls  | IndexNow Submit              | Split In Batches (IndexNow ≤500)1     | Prevents rate limiting by adding jitter.                                                                     |
| Gate: Google                  | Code                   | Conditional gate for Google Indexing API usage       | Filter: lastmod within DAYS_BACK | Loop Over Items                    | Details Google Indexing API toggle.                                                                          |
| Loop Over Items               | SplitInBatches         | Processes URLs for Google API checking               | Gate: Google                 | Check status, (empty)                |                                                                                                              |
| Check status                 | HTTP Request (POST)    | Calls Google Search Console URL Inspection API       | Loop Over Items              | Switch                             |                                                                                                              |
| Switch                       | Switch                 | Routes URLs based on indexing status                  | Check status                 | is new?, URL Updated                |                                                                                                              |
| is new?                      | If                     | Checks if URL lastmod is newer than last crawl        | Switch                      | URL Updated, Wait2                  |                                                                                                              |
| URL Updated                  | HTTP Request (POST)    | Submits URL update to Google Indexing API             | is new?                     | Wait2                             |                                                                                                              |
| Wait2                        | Wait                   | Randomized wait between Google API calls              | is new?, URL Updated         | Loop Over Items                   | Adds delay to avoid Google API rate limits.                                                                   |
| Config - Orphan              | Set                    | Sets config for orphan pages processing               |                              | Get Orphan Pages                  | Explains variables for orphan pages processing.                                                              |
| Get Orphan Pages             | HTTP Request (POST)    | Retrieves orphan pages from Oncrawl Data API          | Config - Orphan              | Split Out - Orphan1               |                                                                                                              |
| Split Out - Orphan1          | SplitOut               | Extracts orphan URLs array                             | Get Orphan Pages             | Split Out - Orphan2              |                                                                                                              |
| Split Out - Orphan2          | SplitOut               | Extracts each orphan URL                               | Split Out - Orphan1          | Merge                           |                                                                                                              |
| Merge                       | Merge                  | Combines orphan URLs with lastmod from sitemaps       | Split Out - Orphan2, Assign mandatory sitemap fields |                              |                                                                                                              |
| Get Crawl over crawl        | HTTP Request (POST)    | Retrieves newly added pages between two crawl runs    |                              | Split Out - Coc1                |                                                                                                              |
| Split Out - Coc1            | SplitOut               | Extracts URLs from crawl-over-crawl response          | Get Crawl over crawl         | Merge1                         |                                                                                                              |
| Merge1                      | Merge                  | Combines crawl-over-crawl URLs with lastmod data      | Split Out - Coc1, Assign mandatory sitemap fields |                              |                                                                                                              |
| Sticky Notes (various)      | Sticky Note            | Documentation and guidance                             |                              |                                | Provide detailed explanations and external links for setup and usage.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook (POST)  
   - Purpose: Receive Oncrawl crawl finished notifications.  
   - Parameters: POST method, responseMode "responseNode".  
   - Save webhook URL externally to configure Oncrawl callback.

2. **Create HTTP Request Node "Post - Get Sitemaps"**  
   - Connect from Webhook.  
   - Method: POST  
   - URL: `https://app.oncrawl.com/api/v2/projects/{{ $json.body.project.id }}/crawl_configs/discover_sitemaps`  
   - Headers: Content-Type: application/json  
   - Body (JSON): Use crawl and project details from webhook payload as in original.  
   - Output: Retrieves sitemap URLs.

3. **Create Set Node "Config"**  
   - Connect from "Post - Get Sitemaps".  
   - Assign variables:  
     - `body.crawl.id` = webhook body crawl id  
     - `SITE_URL` = your site URL (string)  
     - `SITEMAP_URL` = first sitemap URL from previous node results  
     - `DAYS_BACK` = 7 (number)  
     - `BATCH_SIZE` = 500 (number)  
     - `USE_GOOGLE`, `USE_INDEXNOW` = true (boolean)  
     - `INDEXNOW_KEY` and `INDEXNOW_KEY_URL` = your IndexNow key and URL  
   - Save config for use downstream.

4. **Create HTTP Request Node "Get sitemap.xml"**  
   - Connect from Config.  
   - Method: GET  
   - URL: Use `{{$json.SITEMAP_URL}}` from Config.

5. **Create XML Node "Convert sitemap to JSON"**  
   - Connect from "Get sitemap.xml".  
   - Convert XML to JSON.

6. **Create SplitOut Node "Get content-specific sitemaps"**  
   - Connect from "Convert sitemap to JSON".  
   - Field to split out: `sitemapindex.sitemap`.

7. **Create HTTP Request Node "Get content of each sitemap"**  
   - Connect from "Get content-specific sitemaps".  
   - Method: GET  
   - URL: Use `{{$json.loc}}`.

8. **Create XML Node "convert page data to JSON"**  
   - Connect from "Get content of each sitemap".  
   - Convert XML to JSON with option `explicitArray: false`.

9. **Create Set Node "Force urlset.url to array"**  
   - Connect from "convert page data to JSON".  
   - Set `urlset.url` to array:  
     ```javascript
     {{$json.urlset && $json.urlset.url ? ($json.urlset.url[0] ? $json.urlset.url : [$json.urlset.url]) : []}}
     ```

10. **Create SplitOut Node "Split Out"**  
    - Connect from "Force urlset.url to array".  
    - Field to split out: `urlset.url`.

11. **Create Sort Node "Sort"**  
    - Connect from "Split Out".  
    - Sort by `lastmod` descending.

12. **Create Set Node "Assign mandatory sitemap fields"**  
    - Connect from "Sort".  
    - Assign `loc` and `lastmod` fields from input JSON.

13. **Create Code Node "Filter: lastmod within DAYS_BACK"**  
    - Connect from "Assign mandatory sitemap fields".  
    - JavaScript code to filter URLs where lastmod is within DAYS_BACK days from now.

14. **Create Code Node "Gate: IndexNow"**  
    - Connect from "Filter: lastmod within DAYS_BACK".  
    - Pass items only if `USE_INDEXNOW` is true in Config.

15. **Create SplitInBatches Node "Split In Batches (IndexNow ≤500)1"**  
    - Connect from "Gate: IndexNow".  
    - Batch size: minimum of Config BATCH_SIZE and 500.

16. **Create Code Node "Build IndexNow payload"**  
    - Connect from "Split In Batches (IndexNow ≤500)1".  
    - Build payload filtering URLs matching SITE_URL host, deduplicated.

17. **Create HTTP Request Node "IndexNow Submit"**  
    - Connect from "Build IndexNow payload".  
    - POST to `https://api.indexnow.org/indexnow` with JSON body from previous node.

18. **Create Wait Node "Wait (IndexNow jitter)1"**  
    - Connect from "IndexNow Submit".  
    - Wait random 0.25 to 1 second.

19. **Create Code Node "Gate: Google"**  
    - Connect from "Filter: lastmod within DAYS_BACK".  
    - Pass items only if `USE_GOOGLE` is true in Config.

20. **Create SplitInBatches Node "Loop Over Items"**  
    - Connect from "Gate: Google".

21. **Create HTTP Request Node "Check status"**  
    - Connect from second output of "Loop Over Items" (for items processed).  
    - POST to `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`.  
    - Body parameters: inspectionUrl = current URL loc, siteUrl from Config.  
    - Authentication: Google Service Account with indexing API scope.

22. **Create Switch Node "Switch"**  
    - Connect from "Check status".  
    - Cases:  
      - `coverageState = "Submitted and indexed"`  
      - `coverageState = "Crawled - currently not indexed"`

23. **Create If Node "is new?"**  
    - Connect from first output of "Switch".  
    - Condition: Check if sitemap lastmod > last crawl time from Google API.

24. **Create HTTP Request Node "URL Updated"**  
    - Connect from true output of "is new?" and second output of "Switch".  
    - POST to `https://indexing.googleapis.com/v3/urlNotifications:publish`.  
    - Body: url = current URL, type = "URL_UPDATED".  
    - Authentication: same Google Service Account credentials.

25. **Create Wait Node "Wait2"**  
    - Connect from "URL Updated" and false output of "is new?".  
    - Wait random 0.3 to 1.5 seconds.

26. **Connect "Wait2" output back to "Loop Over Items"**  
    - To process next batch.

27. **Optionally implement Orphan Pages handling:**  
    - Create nodes: Config - Orphan, Get Orphan Pages, Split Out - Orphan1/2, Merge, Get Crawl over crawl, Split Out - Coc1, Merge1.  
    - Connect and configure similarly to main flow, integrating orphan URLs with sitemap URLs for indexing.

28. **Add Sticky Notes at relevant points for documentation and instructions.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Oncrawl API documentation for data and notification endpoints: https://developer.oncrawl.com/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Oncrawl API reference                                                                                                                 |
| Google Indexing API setup and credentials instructions including service account creation, permissions, and Google Search Console user management.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | https://developers.google.com/search/apis/indexing-api/v3/using-api                                                                   |
| IndexNow key creation and usage instructions available at Bing Webmaster tools: https://www.bing.com/indexnow/getstarted                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | IndexNow official documentation                                                                                                      |
| YouTube tutorials on Google Indexing API integration (example): https://www.youtube.com/watch?v=HT56wExnN5k and https://www.youtube.com/watch?v=FBGtpWMTppw                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Video resources for practical understanding                                                                                           |
| Rate limiting precautions: randomized wait nodes with jitter to reduce risk of API throttling for both Google and IndexNow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow design best practice                                                                                                        |
| The workflow assumes correct and up-to-date API credentials and permissions; failures in authentication or API structure changes may require updates.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Operational considerations                                                                                                           |

---

**Disclaimer:** This documentation is generated based exclusively on an n8n workflow export. It respects all content policies and includes no illegal or offensive content. The data processed is public and legal.