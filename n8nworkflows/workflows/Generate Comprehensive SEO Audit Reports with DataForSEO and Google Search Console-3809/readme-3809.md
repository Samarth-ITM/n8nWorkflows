Generate Comprehensive SEO Audit Reports with DataForSEO and Google Search Console

https://n8nworkflows.xyz/workflows/generate-comprehensive-seo-audit-reports-with-dataforseo-and-google-search-console-3809


# Generate Comprehensive SEO Audit Reports with DataForSEO and Google Search Console

### 1. Workflow Overview

This workflow automates the generation of comprehensive SEO audit reports by integrating DataForSEO’s website crawling capabilities with Google Search Console’s search performance data. It targets SEO professionals and agencies who require consistent, scalable, and branded content audits for client websites, processing up to 1,000 pages per run.

The workflow is logically divided into the following blocks:

- **1.1 Initial Configuration:** Setting audit parameters and branding details.
- **1.2 Crawl Task Management:** Creating and monitoring a crawl task on DataForSEO.
- **1.3 Raw Data Retrieval and URL Extraction:** Fetching crawl results and extracting URLs for further analysis.
- **1.4 Google Search Console Data Enrichment:** Querying GSC for clicks and impressions per URL and merging this data with crawl results.
- **1.5 Identification of Redirects and 404 Pages:** Extracting problematic URLs and their source links.
- **1.6 Data Analysis and Issue Detection:** Applying SEO heuristics to identify issues across multiple categories.
- **1.7 Report Generation:** Creating a branded, interactive HTML report summarizing findings and recommendations.
- **1.8 Report Delivery Preparation:** Converting the report into a downloadable HTML file.

---

### 2. Block-by-Block Analysis

#### 1.1 Initial Configuration

- **Overview:** Defines all user-configurable parameters such as target domain, crawl limits, branding, and GSC property type.
- **Nodes Involved:**  
  - `When clicking ‘Start’`  
  - `Set Fields`

- **Node Details:**

  - **When clicking ‘Start’**  
    - Type: Manual Trigger  
    - Role: Workflow entry point; initiates the process on user command.  
    - Inputs: None  
    - Outputs: Triggers `Set Fields` node.  
    - Edge Cases: None.

  - **Set Fields**  
    - Type: Set  
    - Role: Stores configurable parameters as workflow variables.  
    - Configuration:  
      - `dfs_domain`: Target website domain (default: "yourclientdomain.com")  
      - `dfs_max_crawl_pages`: Max pages to crawl (default: "1000")  
      - `dfs_enable_javascript`: Enable JS rendering (default: "false")  
      - Company branding fields: name, website, logo URL, primary and secondary colors  
      - `gsc_property_type`: "domain" or "url" (default: "domain")  
    - Inputs: Trigger from manual start  
    - Outputs: Passes parameters to `Create Task` node  
    - Edge Cases: Incorrect or missing domain or color codes may cause errors downstream.

#### 1.2 Crawl Task Management

- **Overview:** Creates a crawl task on DataForSEO and polls its status until completion.
- **Nodes Involved:**  
  - `Create Task`  
  - `Check Task Status`  
  - `If`  
  - `Wait`

- **Node Details:**

  - **Create Task**  
    - Type: HTTP Request  
    - Role: Initiates a DataForSEO crawl task with parameters from `Set Fields`.  
    - Configuration: POST to `https://api.dataforseo.com/v3/on_page/task_post` with JSON body including domain, max pages, JS rendering flag, and a random tag.  
    - Authentication: Basic Auth (DataForSEO credentials)  
    - Inputs: Parameters from `Set Fields`  
    - Outputs: Task ID to `Check Task Status`  
    - Edge Cases: Auth failure, invalid domain, API rate limits.

  - **Check Task Status**  
    - Type: HTTP Request  
    - Role: Polls DataForSEO API for crawl progress using task ID.  
    - Configuration: GET `https://api.dataforseo.com/v3/on_page/summary/{{task_id}}` with Basic Auth.  
    - Inputs: Task ID from `Create Task` or `Wait`  
    - Outputs: Passes to `If` node  
    - Edge Cases: Network timeouts, API errors.

  - **If**  
    - Type: If  
    - Role: Checks if crawl progress equals "finished".  
    - Inputs: Crawl status JSON from `Check Task Status`  
    - Outputs:  
      - True: Proceeds to `Get RAW Audit Data`  
      - False: Proceeds to `Wait` (delay before next status check)  
    - Edge Cases: Unexpected status values.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 1 minute before rechecking crawl status.  
    - Inputs: From `If` node (False branch)  
    - Outputs: Loops back to `Check Task Status`  
    - Edge Cases: Workflow execution time limits.

#### 1.3 Raw Data Retrieval and URL Extraction

- **Overview:** Retrieves the full crawl data and extracts URLs with HTTP 200 status for further analysis.
- **Nodes Involved:**  
  - `Get RAW Audit Data`  
  - `Extract URLs`  
  - `Loop Over Items`

- **Node Details:**

  - **Get RAW Audit Data**  
    - Type: HTTP Request  
    - Role: Fetches crawl page data from DataForSEO using task ID.  
    - Configuration: POST to `https://api.dataforseo.com/v3/on_page/pages` with task ID and limit=1000.  
    - Authentication: Basic Auth  
    - Inputs: Task ID from `If` node (True branch)  
    - Outputs: Raw crawl data JSON to `Extract URLs`  
    - Edge Cases: Large response size, API errors.

  - **Extract URLs**  
    - Type: Code  
    - Role: Parses raw audit data, extracts URLs with status code 200.  
    - Key Logic: Iterates over tasks → results → items; filters pages where `status_code === 200`.  
    - Inputs: Raw audit data from `Get RAW Audit Data`  
    - Outputs: List of URLs as individual items to `Loop Over Items`  
    - Edge Cases: Missing or malformed data structures.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes URLs in batches of 100 to avoid API rate limits.  
    - Inputs: URLs from `Extract URLs`  
    - Outputs: Each batch triggers `Query GSC API` and `Merge GSC Data with RAW Data`  
    - Edge Cases: Batch size too large may cause API throttling.

#### 1.4 Google Search Console Data Enrichment

- **Overview:** Queries Google Search Console API for clicks and impressions per URL and merges this data with crawl results.
- **Nodes Involved:**  
  - `Query GSC API`  
  - `Wait1`  
  - `Map GSC Data to URL`  
  - `Merge GSC Data with RAW Data`

- **Node Details:**

  - **Query GSC API**  
    - Type: HTTP Request  
    - Role: Queries GSC Search Analytics API for each URL’s performance data over last 90 days.  
    - Configuration: POST with JSON body specifying date range, dimension filter for page URL, aggregation type, and row limit.  
    - URL: Dynamically constructed based on `gsc_property_type` ("domain" or "url") and URL domain extraction.  
    - Authentication: Google OAuth2 API credentials  
    - Inputs: URLs from `Loop Over Items`  
    - Outputs: GSC data to `Wait1`  
    - Error Handling: Retries up to 5 times, continues on error to avoid full workflow failure.  
    - Edge Cases: API quota limits, invalid URLs, auth token expiry.

  - **Wait1**  
    - Type: Wait  
    - Role: 1-minute delay to pace API requests and avoid throttling.  
    - Inputs: GSC API response  
    - Outputs: `Map GSC Data to URL`  
    - Edge Cases: None.

  - **Map GSC Data to URL**  
    - Type: Set  
    - Role: Extracts URL, clicks, and impressions from GSC response and formats for merging.  
    - Inputs: GSC API data  
    - Outputs: Structured data to `Loop Over Items` (for merging)  
    - Edge Cases: Missing rows or empty data.

  - **Merge GSC Data with RAW Data**  
    - Type: Code  
    - Role: Enriches raw crawl data by attaching GSC clicks and impressions to each page object.  
    - Key Logic: Builds a lookup map from GSC data keyed by URL, then annotates each page in raw audit data.  
    - Inputs: Raw audit data and GSC data items  
    - Outputs: Single enriched audit JSON object to `Extract 404 & 301`  
    - Edge Cases: URLs missing in GSC data get null metrics.

#### 1.5 Identification of Redirects and 404 Pages

- **Overview:** Extracts URLs with 404 and 301 status codes and retrieves their source links for detailed reporting.
- **Nodes Involved:**  
  - `Extract 404 & 301`  
  - `Loop Over Items1`  
  - `Get Source URLs Data`  
  - `Map URLs Data`

- **Node Details:**

  - **Extract 404 & 301**  
    - Type: Code  
    - Role: Filters pages from raw audit data to those with status 404 or 301.  
    - Inputs: Enriched audit data from `Merge GSC Data with RAW Data`  
    - Outputs: List of 404/301 URLs to `Loop Over Items1`  
    - Edge Cases: No 404/301 pages results in empty output.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes each 404/301 URL individually to fetch source link data.  
    - Inputs: URLs from `Extract 404 & 301`  
    - Outputs: Triggers `Get Source URLs Data` and `Build Report Structure`  
    - Edge Cases: Large number of redirects may increase runtime.

  - **Get Source URLs Data**  
    - Type: HTTP Request  
    - Role: Queries DataForSEO API for internal links pointing to each 404/301 URL.  
    - Configuration: POST to `https://api.dataforseo.com/v3/on_page/links` with task ID and target URL.  
    - Authentication: Basic Auth  
    - Inputs: URL from `Loop Over Items1`  
    - Outputs: Link source data to `Map URLs Data`  
    - Edge Cases: API errors, no source links found.

  - **Map URLs Data**  
    - Type: Code  
    - Role: Formats source link data into a structured object with URL, status code, and sources array.  
    - Inputs: Raw link data from `Get Source URLs Data`  
    - Outputs: Structured source link data back to `Loop Over Items1` (feeding into report build)  
    - Edge Cases: Empty or malformed link data.

#### 1.6 Data Analysis and Issue Detection

- **Overview:** Applies SEO heuristics to the enriched data to detect issues across status, content quality, metadata SEO, internal linking, and performance categories.
- **Nodes Involved:**  
  - `Build Report Structure`

- **Node Details:**

  - **Build Report Structure**  
    - Type: Code  
    - Role: Core analysis node that:  
      - Parses enriched audit data and source link data  
      - Detects issues such as 404 errors, redirects, canonicalization problems, outdated content, thin content, readability, meta tag issues, orphan pages, low internal links, and underperforming pages based on GSC data  
      - Aggregates issues into categorized buckets  
      - Calculates summary counts and flags per page  
      - Prepares a structured JSON object with generated timestamp, summary, detailed issues, and page flags  
    - Inputs:  
      - Enriched audit data with GSC metrics  
      - Source link data for 404/301 pages  
    - Outputs: Structured report data to `Generate HTML Report`  
    - Edge Cases: Missing fields, unexpected data formats, date parsing errors.

#### 1.7 Report Generation

- **Overview:** Generates a fully branded, interactive HTML report summarizing audit findings, issue breakdowns, and prioritized recommendations.
- **Nodes Involved:**  
  - `Generate HTML Report`

- **Node Details:**

  - **Generate HTML Report**  
    - Type: Code  
    - Role:  
      - Reads structured report data and branding parameters  
      - Calculates a health score weighted by issue gravity  
      - Generates executive summary, detailed issue tables with pagination and toggleable source links  
      - Includes recommendations prioritized by issue severity  
      - Styles the report with CSS using brand colors  
      - Embeds JavaScript for interactivity (show/hide source links, pagination toggles)  
      - Produces a complete standalone HTML document as output  
    - Inputs: Report structure from `Build Report Structure` and branding from `Set Fields`  
    - Outputs: HTML content to `Download Report`  
    - Edge Cases: Large reports may impact browser rendering; color codes must be valid hex.

#### 1.8 Report Delivery Preparation

- **Overview:** Converts the HTML report content into a downloadable file.
- **Nodes Involved:**  
  - `Download Report`

- **Node Details:**

  - **Download Report**  
    - Type: ConvertToFile  
    - Role: Converts the HTML text from `Generate HTML Report` into a downloadable file with a dynamic filename based on domain and date.  
    - Configuration:  
      - Filename pattern: `{dfs_domain}-content-audit-{Month-Year}.html`  
      - Source property: `html`  
      - Output binary property: `content audit report`  
    - Inputs: HTML content  
    - Outputs: File ready for download or further distribution  
    - Edge Cases: Filename must be valid; large files may affect performance.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                      | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                          |
|-------------------------|---------------------|-----------------------------------------------------|----------------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Start’    | Manual Trigger      | Workflow entry point                                | -                          | Set Fields                | See setup instructions in sticky note for credentials and configuration.                           |
| Set Fields              | Set                 | Defines audit parameters and branding               | When clicking ‘Start’       | Create Task               | See setup instructions in sticky note for credentials and configuration.                           |
| Create Task             | HTTP Request        | Creates DataForSEO crawl task                        | Set Fields                 | Check Task Status         | Requires DataForSEO Basic Auth credential.                                                         |
| Check Task Status       | HTTP Request        | Polls crawl task status                              | Create Task, Wait          | If                        | Requires DataForSEO Basic Auth credential.                                                         |
| If                      | If                  | Checks if crawl is finished                          | Check Task Status          | Get RAW Audit Data, Wait  | -                                                                                                  |
| Wait                    | Wait                | Waits 1 minute before rechecking crawl status       | If                        | Check Task Status         | -                                                                                                  |
| Get RAW Audit Data       | HTTP Request        | Retrieves full crawl data                            | If                        | Extract URLs              | Requires DataForSEO Basic Auth credential.                                                         |
| Extract URLs            | Code                | Extracts URLs with status 200                        | Get RAW Audit Data         | Loop Over Items           | -                                                                                                  |
| Loop Over Items         | SplitInBatches      | Batches URLs for GSC API querying                    | Extract URLs               | Query GSC API, Merge GSC Data with RAW Data | -                                                                                                  |
| Query GSC API           | HTTP Request        | Queries Google Search Console for clicks/impressions| Loop Over Items            | Wait1                     | Requires Google OAuth2 API credential. Retries on failure.                                        |
| Wait1                   | Wait                | Waits 1 minute between GSC API requests              | Query GSC API              | Map GSC Data to URL       | -                                                                                                  |
| Map GSC Data to URL     | Set                 | Maps GSC clicks and impressions to URLs             | Wait1                      | Loop Over Items           | -                                                                                                  |
| Merge GSC Data with RAW Data | Code            | Enriches crawl data with GSC metrics                 | Loop Over Items            | Extract 404 & 301         | -                                                                                                  |
| Extract 404 & 301       | Code                | Extracts 404 and 301 URLs                            | Merge GSC Data with RAW Data | Loop Over Items1         | -                                                                                                  |
| Loop Over Items1        | SplitInBatches      | Processes 404/301 URLs for source link retrieval    | Extract 404 & 301          | Get Source URLs Data, Build Report Structure | -                                                                                                  |
| Get Source URLs Data    | HTTP Request        | Retrieves internal links pointing to 404/301 URLs   | Loop Over Items1           | Map URLs Data             | Requires DataForSEO Basic Auth credential.                                                         |
| Map URLs Data           | Code                | Formats source link data                             | Get Source URLs Data       | Loop Over Items1          | -                                                                                                  |
| Build Report Structure  | Code                | Analyzes data, detects SEO issues, builds report JSON| Loop Over Items1           | Generate HTML Report      | -                                                                                                  |
| Generate HTML Report    | Code                | Generates branded, interactive HTML report           | Build Report Structure, Set Fields | Download Report      | -                                                                                                  |
| Download Report         | ConvertToFile       | Converts HTML to downloadable file                   | Generate HTML Report       | -                         | -                                                                                                  |
| Sticky Note             | Sticky Note         | Setup instructions and workflow overview             | -                          | -                         | Contains detailed setup instructions and useful links for credentials and customization.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Start’` to serve as the workflow entry point.

2. **Add a Set node** named `Set Fields` connected from the manual trigger. Configure it with the following string fields:  
   - `dfs_domain`: e.g., "yourclientdomain.com"  
   - `dfs_max_crawl_pages`: "1000"  
   - `dfs_enable_javascript`: "false"  
   - `company_name`: Your company name  
   - `company_website`: Your company website URL  
   - `company_logo_url`: URL to your logo image  
   - `brand_primary_color`: Hex code (e.g., "#252946")  
   - `brand_secondary_color`: Hex code (e.g., "#0fd393")  
   - `gsc_property_type`: "domain" or "url"

3. **Add an HTTP Request node** named `Create Task` connected from `Set Fields`. Configure:  
   - Method: POST  
   - URL: `https://api.dataforseo.com/v3/on_page/task_post`  
   - Authentication: Basic Auth (DataForSEO credentials)  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     [
       {
         "target": "={{ $json.dfs_domain }}",
         "max_crawl_pages": {{ $json.dfs_max_crawl_pages }},
         "load_resources": false,
         "enable_javascript": {{ $json.dfs_enable_javascript }},
         "custom_js": "meta = {}; meta.url = document.URL; meta;",
         "tag": "={{ $json.dfs_domain + Math.floor(10000 + Math.random() * 90000) }}"
       }
     ]
     ```

4. **Add an HTTP Request node** named `Check Task Status` connected from `Create Task` and later from `Wait`. Configure:  
   - Method: GET  
   - URL: `https://api.dataforseo.com/v3/on_page/summary/{{ $json.tasks[0].id }}`  
   - Authentication: Basic Auth (DataForSEO credentials)  
   - Headers: Content-Type: application/json

5. **Add an If node** named `If` connected from `Check Task Status`. Configure condition:  
   - Expression: `{{$json.tasks[0].result[0].crawl_progress}}` equals `"finished"`  
   - True branch: connect to `Get RAW Audit Data`  
   - False branch: connect to `Wait`

6. **Add a Wait node** named `Wait` connected from `If` (False branch). Configure to wait 1 minute, then connect back to `Check Task Status`.

7. **Add an HTTP Request node** named `Get RAW Audit Data` connected from `If` (True branch). Configure:  
   - Method: POST  
   - URL: `https://api.dataforseo.com/v3/on_page/pages`  
   - Authentication: Basic Auth (DataForSEO credentials)  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     [
       {
         "id": "{{ $json.tasks[0].id }}",
         "limit": "1000"
       }
     ]
     ```

8. **Add a Code node** named `Extract URLs` connected from `Get RAW Audit Data`. Paste the JavaScript code that extracts URLs with status 200 from the raw audit data.

9. **Add a SplitInBatches node** named `Loop Over Items` connected from `Extract URLs`. Set batch size to 100.

10. **Add an HTTP Request node** named `Query GSC API` connected from `Loop Over Items`. Configure:  
    - Method: POST  
    - URL: Use an expression to build the GSC API endpoint based on `gsc_property_type` and URL domain.  
    - Authentication: Google OAuth2 API credentials  
    - Headers: Content-Type: application/json  
    - Body (raw JSON): Query last 90 days clicks and impressions filtered by page URL.

11. **Add a Wait node** named `Wait1` connected from `Query GSC API`. Configure 1 minute delay.

12. **Add a Set node** named `Map GSC Data to URL` connected from `Wait1`. Map URL, clicks, and impressions from GSC response.

13. **Connect `Map GSC Data to URL` back to `Loop Over Items`** to continue batch processing.

14. **Add a Code node** named `Merge GSC Data with RAW Data` connected from `Loop Over Items`. Paste the code that enriches raw audit data with GSC metrics.

15. **Add a Code node** named `Extract 404 & 301` connected from `Merge GSC Data with RAW Data`. Paste the code that filters pages with status 404 or 301.

16. **Add a SplitInBatches node** named `Loop Over Items1` connected from `Extract 404 & 301`.

17. **Add an HTTP Request node** named `Get Source URLs Data` connected from `Loop Over Items1`. Configure:  
    - Method: POST  
    - URL: `https://api.dataforseo.com/v3/on_page/links`  
    - Authentication: Basic Auth (DataForSEO credentials)  
    - Headers: Content-Type: application/json  
    - Body (JSON):  
      ```json
      [
        {
          "id": "{{ $('Get RAW Audit Data').first().json.tasks[0].id }}",
          "page_to": "{{ $json.url }}"
        }
      ]
      ```

18. **Add a Code node** named `Map URLs Data` connected from `Get Source URLs Data`. Paste code that formats source link data.

19. **Connect `Map URLs Data` back to `Loop Over Items1`** to continue batch processing.

20. **Connect `Loop Over Items1` also to a Code node** named `Build Report Structure`. Paste the comprehensive analysis code that detects SEO issues and builds the report JSON.

21. **Add a Code node** named `Generate HTML Report` connected from `Build Report Structure`. Paste the full HTML report generation code, including styling and interactivity.

22. **Add a ConvertToFile node** named `Download Report` connected from `Generate HTML Report`. Configure:  
    - Operation: toText  
    - Source property: `html`  
    - File name: `={{ $json.dfs_domain }}-content-audit-{{ new Date().toLocaleString('en-US', { month: 'long' }) + '-' + new Date().getFullYear() }}.html`  
    - Binary property name: `content audit report`

23. **Set up credentials:**  
    - Create Basic Auth credential with DataForSEO API keys and assign to all DataForSEO HTTP Request nodes.  
    - Create Google OAuth2 API credential with access to Google Search Console and assign to `Query GSC API`.

24. **Test the workflow:** Start manually, monitor progress, and download the generated report upon completion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow is powered by DataForSEO and Google Search Console APIs to generate a fully branded SEO content audit report for up to 1,000 pages. It automates a process that typically takes hours into about 20 minutes.                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow description and purpose                                                                         |
| Setup requires creating Basic Auth credentials for DataForSEO and Google OAuth2 credentials for Search Console. DataForSEO offers a free $1 credit for testing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                | https://app.dataforseo.com/api-access, https://docs.n8n.io/integrations/builtin/credentials/httprequest/  |
| Google OAuth2 API credentials must have access to the target Search Console property.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/                               |
| The report is fully customizable by editing the code nodes `Build Report Structure` and `Generate HTML Report` to adjust thresholds, styling, and recommendations.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Customization instructions in workflow description                                                      |
| For extended functionality, users can add nodes to send the report via email, upload to cloud storage, or convert to PDF after the `Download Report` node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Suggested workflow extensions                                                                            |
| Workflow runtime scales with number of pages; expect ~20 minutes for 500 pages. Batch sizes and wait times are configured to avoid API rate limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Performance considerations                                                                                |
| Contact Custom Workflows AI for professional customization or support.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | https://customworkflows.ai/work-with-us                                                                  |

---

This document provides a detailed, structured reference to understand, reproduce, and customize the "Automated Content SEO Audit Report" workflow integrating DataForSEO and Google Search Console APIs in n8n.