Google Search Console and Analytics Analysis with AI Optimizations

https://n8nworkflows.xyz/workflows/google-search-console-and-analytics-analysis-with-ai-optimizations-5303


# Google Search Console and Analytics Analysis with AI Optimizations

### 1. Workflow Overview

This workflow automates the process of analyzing website performance data by integrating Google Search Console (GSC), Google Analytics (GA), and AI-powered SEO analysis. Its primary purpose is to collect sitemap URLs, fetch performance data for each page, analyze this data with AI to generate specific SEO improvement recommendations, and send a detailed report via email. The workflow is designed for webmasters, SEO professionals, and digital marketers seeking automated, data-driven insights to optimize web pages.

The workflow is logically organized into the following blocks:

- **1.1 Initialization and Configuration**  
  Set global parameters including sitemap URL, Search Console selector, date ranges, Analytics property ID, and report recipient email.

- **1.2 Sitemap Retrieval and URL Extraction**  
  Download sitemap XML, convert it to JSON, extract all URLs, remove duplicates, and prepare the list for processing.

- **1.3 Per-Page Data Collection**  
  For each URL:
  - Fetch GSC URL inspection status.
  - Fetch GSC search analytics stats.
  - Fetch Google Analytics metrics.

- **1.4 Data Merging and Ordering**  
  Combine GSC status and stats with GA data, enforce ordering, and clean the dataset for reporting.

- **1.5 AI-Driven SEO Analysis**  
  Use the Langchain SEO analyst agent powered by OpenAI to analyze combined data for each page and generate specific SEO improvement recommendations.

- **1.6 Report Formatting and Delivery**  
  Convert AI output to Markdown, then HTML, merge with input data, and send a comprehensive email report.

- **1.7 Execution Control and Testing Utilities**  
  Includes batching, waiting to comply with API limits, sorting, and limiting data for testing purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration

**Overview:**  
Defines global parameters used throughout the workflow such as sitemap URL, Google Search Console domain selector, analysis date range, Google Analytics property ID, and report recipient email. This block serves as the single source of truth for configurable inputs.

**Nodes Involved:**  
- Globals - CHANGE ME!

**Node Details:**  
- **Type:** Set  
- **Role:** Define workflow-wide variables that are referenced by expressions in other nodes.  
- **Key Parameters:**  
  - `sitemap_url`: URL of the sitemap XML.  
  - `search_console_selector`: GSC site selector (e.g., "sc-domain:sailingbyte.com").  
  - `analysis_start_date` and `analysis_end_date`: Date range for analytics, defaults to last 30 days.  
  - `analytics_selector_id`: Numeric ID for Google Analytics property.  
  - `report_receiver`: Email address for receiving report.  
- **Input:** Manual trigger node output  
- **Output:** Feeds the sitemap retrieval node and others via expressions.  
- **Edge Cases:** Misconfiguration here (e.g., wrong sitemap URL or selector) will cause downstream errors or empty data.  
- **Version Requirements:** Uses version 3.4 of Set node for advanced expression support.

---

#### 2.2 Sitemap Retrieval and URL Extraction

**Overview:**  
Fetches the sitemap XML, converts it to JSON, extracts all page URLs, removes duplicates, sorts them randomly (for testing), limits the number for test runs, and prepares a clean, ordered list of URLs.

**Nodes Involved:**  
- Get sitemap XML  
- Convert to JSON  
- Split Out to Table  
- Remove Duplicates  
- Sort For Testing Purposes  
- Limit for Testing Purposes  
- Enforce Order - Positions Last  
- Sticky Notes (contextual guidance)

**Node Details:**

- **Get sitemap XML**  
  - Type: HTTP Request  
  - Role: Fetch sitemap.xml content from the configured URL.  
  - Configuration: Uses URL from Globals node via expression.  
  - Output: Raw XML data.  
  - Edge Cases: Network error, invalid URL, or sitemap not accessible.

- **Convert to JSON**  
  - Type: XML  
  - Role: Parses the XML sitemap into JSON format.  
  - Input: Raw XML from previous node.  
  - Output: JSON object with sitemap URLs.  
  - Edge Cases: Malformed XML causing parse failures.

- **Split Out to Table**  
  - Type: Split Out  
  - Role: Extracts the array of URLs from JSON (`urlset.url`).  
  - Output: Each URL becomes an individual item.  
  - Edge Cases: Empty or missing URL entries.

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Ensures unique URLs by comparing the 'loc' field.  
  - Edge Cases: Duplicates in sitemap could cause redundant processing.

- **Sort For Testing Purposes**  
  - Type: Sort  
  - Role: Randomizes URL order to facilitate test runs.  
  - Edge Cases: None critical.

- **Limit for Testing Purposes**  
  - Type: Limit  
  - Role: Limits number of URLs processed to 5 (configurable).  
  - Edge Cases: Limits full data processing; remove for production.

- **Enforce Order - Positions Last**  
  - Type: Merge (combineByPosition)  
  - Role: Maintains or re-orders data structure for consistency downstream.

---

#### 2.3 Per-Page Data Collection

**Overview:**  
Processes each URL batch to retrieve detailed data from Google Search Console and Google Analytics APIs for SEO analysis.

**Nodes Involved:**  
- Loop Over Items  
- Get Page GSC Status  
- Get Page GSC Stats  
- Get Google Analytics Data  
- Merge Status and Stats  
- Wait to Comply with API Limits  
- Sticky Note (contextual)

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes URLs in manageable batches to avoid API overload or limits.

- **Get Page GSC Status**  
  - Type: HTTP Request  
  - Role: Calls GSC URL Inspection API to retrieve indexing status of each URL.  
  - Configuration: POST request with `inspectionUrl` and `siteUrl` parameters from current URL and global selector.  
  - Authentication: Google OAuth2.  
  - Error Handling: Continues on error to avoid workflow halt.  
  - Edge Cases: API quota limits, network issues, invalid URLs.

- **Get Page GSC Stats**  
  - Type: HTTP Request  
  - Role: Queries GSC Search Analytics API for page-specific performance metrics within date range.  
  - Configuration: POST with JSON body filtering by page URL.  
  - Authentication: Google OAuth2.  
  - Error Handling: Continues on error.  
  - Edge Cases: API quota, missing data for URL.

- **Get Google Analytics Data**  
  - Type: Google Analytics Node  
  - Role: Retrieves GA4 metrics (screenPageViews, userEngagementDuration) filtered by page URL.  
  - Authentication: Google Analytics OAuth2.  
  - Edge Cases: Permissions errors, incorrect property ID.

- **Merge Status and Stats**  
  - Type: Merge  
  - Role: Combines outputs of GSC status, GSC stats, and GA data by position ensuring all data for a single page is unified.

- **Wait to Comply with API Limits**  
  - Type: Wait  
  - Role: Adds delay (1 second) to throttle API calls and avoid quota exhaustion.

---

#### 2.4 Data Merging and Ordering

**Overview:**  
Processes merged GSC and GA data by cleaning and selecting only relevant fields, converting to HTML tables, and preparing data for AI analysis and email reporting.

**Nodes Involved:**  
- Keep Only Valuable Columns  
- Change to HTML table for sending  
- Enforce Order - Positions Last  
- Merge AI Output With AI Input

**Node Details:**

- **Keep Only Valuable Columns**  
  - Type: Set  
  - Role: Reduces data to only useful fields: URL (`loc`), GSC coverage state, AI output text.  
  - Edge Cases: Missing fields may lead to incomplete reports.

- **Change to HTML table for sending**  
  - Type: HTML  
  - Role: Converts JSON data table into an HTML table format for email embedding.

- **Enforce Order - Positions Last**  
  - Role: Ensures correct ordering of data items before sending.

- **Merge AI Output With AI Input**  
  - Type: Merge  
  - Role: Combines AI-generated recommendations with original data for comprehensive reporting.

---

#### 2.5 AI-Driven SEO Analysis

**Overview:**  
The core analytical block uses Langchain agent with OpenAI models to interpret combined GSC and GA data per page and generate detailed, page-specific SEO improvement recommendations.

**Nodes Involved:**  
- SEO analyst (Langchain agent)  
- OpenAI Chat Model  
- Markdown to HTML  
- HTTP Request (for additional page data)  

**Node Details:**

- **SEO analyst**  
  - Type: Langchain Agent Node  
  - Role: Receives combined data and prompts AI to act as a professional SEO analyst.  
  - Prompt: Detailed instructions to analyze indexing status, analytics data, and page content for SEO improvements including technical, content, and keyword strategy evaluations.  
  - AI Model: Uses "o4-mini" OpenAI chat model via Langchain.  
  - Input: Merged data for one page.  
  - Output: SEO recommendations in Markdown format.

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Language model backend powering the SEO analyst.  
  - Credentials: OpenAI API key.  
  - Edge Cases: API rate limits, model errors.

- **Markdown to HTML**  
  - Type: Markdown  
  - Role: Converts AI Markdown output to HTML for email compatibility.

- **HTTP Request (HTTP Request Tool)**  
  - Role: Optional node to fetch additional page data for SEO analysis.  
  - Configured dynamically via AI override URL input.  
  - Edge Cases: Network errors, invalid URLs.

---

#### 2.6 Report Formatting and Delivery

**Overview:**  
Prepares the final report in HTML and sends it via email to the configured recipient.

**Nodes Involved:**  
- Send Email with Full report

**Node Details:**

- **Send Email with Full report**  
  - Type: Email Send  
  - Role: Sends an HTML email with the SEO analysis results table.  
  - Configuration:  
    - Subject: "Your Analysis Is Complete"  
    - To and From: Email from Globals configuration.  
  - Credentials: SMTP configured with user credentials.  
  - Edge Cases: SMTP authentication failures, email delivery issues.

---

#### 2.7 Execution Control and Testing Utilities

**Overview:**  
Includes manual trigger, batching, sorting, limiting, and waiting nodes to control workflow execution, manage API quotas, and facilitate testing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Loop Over Items (Split In Batches)  
- Wait to Comply with API Limits  
- Sort For Testing Purposes  
- Limit for Testing Purposes  
- Sticky Notes (setup instructions and explanations)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution.  

- **Loop Over Items**  
  - Role: Processes URLs in batches, enabling scalability and rate limiting.

- **Wait to Comply with API Limits**  
  - Role: Adds delay to avoid API throttling.

- **Sort For Testing Purposes and Limit for Testing Purposes**  
  - Role: Control dataset size and order for manageable test runs.

---

### 3. Summary Table

| Node Name                  | Node Type                       | Functional Role                         | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                                  |
|----------------------------|--------------------------------|---------------------------------------|----------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Workflow start trigger                 | -                                | Globals - CHANGE ME!                |                                                                                                              |
| Globals - CHANGE ME!        | Set                            | Global configuration and parameters   | When clicking ‘Test workflow’    | Get sitemap XML                    | - sitemap_url: sitemap XML URL<br>- search_console_selector: GSC site selector<br>- analysis_start_date and analysis_end_date: date range<br>- analytics_selector_id: GA property ID<br>- report_receiver: email for report |
| Get sitemap XML             | HTTP Request                   | Fetch sitemap XML                     | Globals - CHANGE ME!             | Convert to JSON                   | ## Get All Pages From Sitemap                                                                                 |
| Convert to JSON            | XML                            | Parse XML to JSON                     | Get sitemap XML                  | Split Out to Table                |                                                                                                              |
| Split Out to Table          | Split Out                      | Extract URLs array                    | Convert to JSON                 | Remove Duplicates                 |                                                                                                              |
| Remove Duplicates           | Remove Duplicates              | Remove duplicate URLs                 | Split Out to Table              | Sort For Testing Purposes         |                                                                                                              |
| Sort For Testing Purposes   | Sort                          | Randomize URL order (testing only)   | Remove Duplicates               | Limit for Testing Purposes        | ## Limit for Testing                                                                                           |
| Limit for Testing Purposes  | Limit                         | Limit number of URLs for tests        | Sort For Testing Purposes       | Enforce Order - Positions Last, Loop Over Items |                                                                                                              |
| Enforce Order - Positions Last | Merge                       | Enforce data order                    | Limit for Testing Purposes, Loop Over Items | Keep Only Valuable Columns      | ## Get Google Data and Push It Through AI                                                                     |
| Loop Over Items             | Split In Batches              | Batch process URLs                    | Limit for Testing Purposes, Merge AI Output With AI Input | Get Page GSC Status, Get Page GSC Stats, Get Google Analytics Data |                                                                                                              |
| Get Page GSC Status         | HTTP Request                  | Fetch GSC URL inspection status      | Loop Over Items                 | Merge Status and Stats           |                                                                                                              |
| Get Page GSC Stats          | HTTP Request                  | Fetch GSC search analytics stats     | Loop Over Items                 | Merge Status and Stats           |                                                                                                              |
| Get Google Analytics Data   | Google Analytics              | Fetch GA metrics per URL              | Loop Over Items                 | Merge Status and Stats           |                                                                                                              |
| Merge Status and Stats      | Merge                        | Combine GSC status, stats, and GA data | Get Page GSC Status, Get Page GSC Stats, Get Google Analytics Data | SEO analyst, Wait to Comply with API Limits |                                                                                                              |
| Wait to Comply with API Limits | Wait                       | Throttle API calls                   | Merge Status and Stats          | Merge AI Output With AI Input     |                                                                                                              |
| Merge AI Output With AI Input | Merge                      | Combine AI output with input data    | Markdown to HTML, Wait to Comply with API Limits | Loop Over Items                  |                                                                                                              |
| SEO analyst                 | Langchain Agent              | AI SEO analysis and recommendations | Merge Status and Stats          | Markdown to HTML                 |                                                                                                              |
| OpenAI Chat Model           | Langchain LM Chat OpenAI     | AI language model backend             | SEO analyst                    | SEO analyst                     |                                                                                                              |
| Markdown to HTML            | Markdown                     | Convert AI Markdown to HTML           | SEO analyst                    | Merge AI Output With AI Input     |                                                                                                              |
| Keep Only Valuable Columns  | Set                          | Select relevant fields for report    | Enforce Order - Positions Last  | Change to HTML table for sending  |                                                                                                              |
| Change to HTML table for sending | HTML                    | Convert data to HTML table            | Keep Only Valuable Columns      | Send Email with Full report       |                                                                                                              |
| Send Email with Full report | Email Send                   | Send final SEO report via email      | Change to HTML table for sending | -                              |                                                                                                              |
| HTTP Request               | HTTP Request Tool              | Optional: fetch page data for SEO analysis | SEO analyst                    | SEO analyst                     | Make HTTP request to get page data so you can further analyze page for SEO optimizations                       |
| Sticky Note                | Sticky Note                   | Setup instructions and section labels | Various                       | -                              | Multiple sticky notes provide guidance on setup, testing limits, and data flow blocks                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for Globals:**  
   - Name: "Globals - CHANGE ME!"  
   - Define variables:  
     - `sitemap_url` (string): e.g., "https://sailingbyte.com/sitemap.xml"  
     - `search_console_selector` (string): e.g., "sc-domain:example.com"  
     - `analysis_start_date` (string, expression): `{{$now.minus(30,'days').format('yyyy-MM-dd')}}`  
     - `analysis_end_date` (string, expression): `{{$now.format('yyyy-MM-dd')}}`  
     - `analytics_selector_id` (number): Your GA property ID  
     - `report_receiver` (string): Email for report delivery  
   - Connect "When clicking ‘Test workflow’" → "Globals - CHANGE ME!"

3. **Create HTTP Request Node to Get Sitemap XML:**  
   - Name: "Get sitemap XML"  
   - Method: GET  
   - URL: `={{ $('Globals - CHANGE ME!').item.json.sitemap_url }}`  
   - Connect "Globals - CHANGE ME!" → "Get sitemap XML"

4. **Create XML Node to Convert Sitemap:**  
   - Name: "Convert to JSON"  
   - Operation: XML to JSON  
   - Connect "Get sitemap XML" → "Convert to JSON"

5. **Create Split Out Node:**  
   - Name: "Split Out to Table"  
   - Field to split out: `urlset.url`  
   - Connect "Convert to JSON" → "Split Out to Table"

6. **Create Remove Duplicates Node:**  
   - Name: "Remove Duplicates"  
   - Compare by selected fields: `loc`  
   - Connect "Split Out to Table" → "Remove Duplicates"

7. **Create Sort Node:**  
   - Name: "Sort For Testing Purposes"  
   - Sort type: Random  
   - Connect "Remove Duplicates" → "Sort For Testing Purposes"

8. **Create Limit Node:**  
   - Name: "Limit for Testing Purposes"  
   - Max items: 5 (adjust or remove for full runs)  
   - Connect "Sort For Testing Purposes" → "Limit for Testing Purposes"

9. **Create Merge Node to Enforce Order:**  
   - Name: "Enforce Order - Positions Last"  
   - Mode: Combine by Position  
   - Connect "Limit for Testing Purposes" → "Enforce Order - Positions Last"

10. **Create Split In Batches Node:**  
    - Name: "Loop Over Items"  
    - Configure batch size as needed (default or small for API limits)  
    - Connect "Limit for Testing Purposes" and "Merge AI Output With AI Input" to "Loop Over Items"

11. **Create HTTP Request Node for GSC URL Inspection:**  
    - Name: "Get Page GSC Status"  
    - Method: POST  
    - URL: `https://searchconsole.googleapis.com/v1/urlInspection/index:inspect`  
    - Query parameters:  
      - `inspectionUrl`: `={{ $json.loc }}`  
      - `siteUrl`: `={{ $('Globals - CHANGE ME!').first().json.search_console_selector }}`  
    - Authentication: Google OAuth2 (set credentials)  
    - Error behavior: Continue on fail  
    - Connect "Loop Over Items" → "Get Page GSC Status"

12. **Create HTTP Request Node for GSC Search Analytics:**  
    - Name: "Get Page GSC Stats"  
    - Method: POST  
    - URL: `https://www.googleapis.com/webmasters/v3/sites/{{ $('Globals - CHANGE ME!').first().json.search_console_selector }}/searchAnalytics/query`  
    - Body (JSON):  
      ```json
      {
        "startDate": "{{ $('Globals - CHANGE ME!').first().json.analysis_start_date }}",
        "endDate": "{{ $('Globals - CHANGE ME!').first().json.analysis_end_date }}",
        "type": "web",
        "dimensionFilterGroups": [{
          "groupType": "AND",
          "filters": [{
            "dimension": "page",
            "operator": "contains",
            "expression": "{{ $json.loc }}"
          }]
        }]
      }
      ```  
    - Authentication: Google OAuth2  
    - Error behavior: Continue on fail  
    - Connect "Loop Over Items" → "Get Page GSC Stats"

13. **Create Google Analytics Node:**  
    - Name: "Get Google Analytics Data"  
    - Set date range start and end from Globals  
    - Metrics: screenPageViews, userEngagementDuration  
    - Dimensions: pageLocation  
    - Filters: pageLocation contains current URL  
    - Property ID: from Globals  
    - Credentials: Google Analytics OAuth2  
    - Connect "Loop Over Items" → "Get Google Analytics Data"

14. **Create Merge Node for GSC and GA Data:**  
    - Name: "Merge Status and Stats"  
    - Mode: Combine by Position, 3 inputs  
    - Connect outputs of "Get Page GSC Status", "Get Page GSC Stats", and "Get Google Analytics Data" → "Merge Status and Stats"

15. **Create Langchain Agent Node for SEO Analysis:**  
    - Name: "SEO analyst"  
    - Prompt: Detailed SEO analyst instructions with placeholders for page URL, GA data, GSC data, and indexing status (see original prompt text)  
    - AI Model: Connect to OpenAI Chat Model node  
    - Connect "Merge Status and Stats" → "SEO analyst"

16. **Create OpenAI Chat Model Node:**  
    - Name: "OpenAI Chat Model"  
    - Model: "o4-mini" or preferred model  
    - Credentials: OpenAI API key  
    - Connect "SEO analyst" (AI language model input) → "OpenAI Chat Model"

17. **Create Markdown Node:**  
    - Name: "Markdown to HTML"  
    - Mode: Markdown to HTML  
    - Source: AI output field (`$json.output`)  
    - Connect "SEO analyst" → "Markdown to HTML"

18. **Create Merge Node for AI Output and Input Data:**  
    - Name: "Merge AI Output With AI Input"  
    - Connect "Markdown to HTML" → "Merge AI Output With AI Input" (main input 2)  
    - Connect "Wait to Comply with API Limits" → "Merge AI Output With AI Input" (main input 1)

19. **Create Wait Node:**  
    - Name: "Wait to Comply with API Limits"  
    - Duration: 1 second  
    - Connect "Merge Status and Stats" → "Wait to Comply with API Limits"

20. **Create Set Node to Keep Valuable Columns:**  
    - Name: "Keep Only Valuable Columns"  
    - Keep fields: `loc`, `inspectionResult.indexStatusResult.coverageState`, `output` (AI result)  
    - Connect "Enforce Order - Positions Last" → "Keep Only Valuable Columns"

21. **Create HTML Node:**  
    - Name: "Change to HTML table for sending"  
    - Operation: Convert JSON to HTML table  
    - Connect "Keep Only Valuable Columns" → "Change to HTML table for sending"

22. **Create Email Send Node:**  
    - Name: "Send Email with Full report"  
    - To and From: `report_receiver` from Globals  
    - Subject: "Your Analysis Is Complete"  
    - Email body: Include HTML table from previous node  
    - Credentials: SMTP account  
    - Connect "Change to HTML table for sending" → "Send Email with Full report"

23. **Final Connections:**  
    - Connect "When clicking ‘Test workflow’" → "Globals - CHANGE ME!"  
    - Connect "Limit for Testing Purposes" → "Enforce Order - Positions Last" and "Loop Over Items"  
    - Connect "Merge AI Output With AI Input" → "Loop Over Items" for iterative processing.

24. **Add Sticky Notes:**  
    - Add notes near relevant nodes explaining configuration, setup, and testing as per original.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The `search_console_selector` can be either domain-scoped like "sc-domain:sailingbyte.com" or a URL, depending on your Search Console setup.                                                                                      | Globals - CHANGE ME! variables explanation                                                                      |
| Google Analytics property ID can be found in the URL of your GA dashboard preceded by "p", for example: https://analytics.google.com/analytics/web/#/pXXXXXXXXX/reports/intelligenthome                                             | Globals - CHANGE ME! variables explanation                                                                      |
| To avoid API quota and rate limit issues, the workflow uses batching, waiting between API requests, and limits for testing. Remove or adjust limits for production use.                                                             | Execution control block                                                                                        |
| SEO analyst Langchain prompt includes advanced readability metrics like Flesch-Kincaid and Gunning-Fog to assess content clarity and SEO quality.                                                                                   | SEO analyst node prompt                                                                                         |
| OpenAI model "o4-mini" is used for cost-efficient analysis; you can replace it with other models as needed.                                                                                                                        | OpenAI Chat Model node                                                                                          |
| SMTP credentials must be properly configured for email delivery; test with your email provider settings.                                                                                                                            | Email Send node                                                                                                |
| Google API credentials (OAuth2) with appropriate scopes are required for Search Console and Analytics nodes. Ensure tokens are valid and have sufficient permissions.                                                                | Get Page GSC Status, Get Page GSC Stats, Get Google Analytics Data nodes                                       |
| More info on setting up Google API credentials: https://developers.google.com/identity/protocols/oauth2                                                                                                                             | External documentation                                                                                         |
| n8n documentation on Langchain integration: https://docs.n8n.io/integrations/builtin/nodes/langchain/                                                                                                                             | n8n official docs                                                                                              |

---

**Disclaimer:** The provided content is exclusively generated from an automated n8n workflow. It fully complies with current content policies and does not contain illegal, offensive, or protected elements. All data handled is legal and publicly accessible.