Extract Google Trends Keywords & Summarize Articles in Google Sheets

https://n8nworkflows.xyz/workflows/extract-google-trends-keywords---summarize-articles-in-google-sheets-3132


# Extract Google Trends Keywords & Summarize Articles in Google Sheets

### 1. Workflow Overview

This workflow automates the process of extracting trending keywords from Google Trends RSS feeds, scraping related news article content, summarizing the content using Jina AI, and storing structured insights in a Google Sheet. It is designed for content strategists, SEO professionals, and news publishers to streamline editorial planning based on real-time trending topics.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Configuration**: Initiates the workflow on schedule or manual trigger and sets configuration parameters.
- **1.2 Google Trends Data Retrieval & Parsing**: Fetches the Google Trends RSS feed and converts it from XML to JSON.
- **1.3 Existing Data Fetch & Filtering**: Retrieves existing keywords from Google Sheets and filters new trending keywords based on traffic and duplication.
- **1.4 Content Scraping & Summarization**: Scrapes article content from URLs related to each trending keyword using Jina AI and generates combined summaries.
- **1.5 Data Mapping & Conditional Saving**: Maps all data fields to the Google Sheets schema and conditionally saves the record if sufficient summary content is available.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration

**Overview:**  
This block starts the workflow either manually or on a scheduled cron trigger and sets key configuration parameters such as minimum traffic threshold, maximum results to process, and the Jina AI API key.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Start every hour past 11 minutes (Schedule Trigger)  
- CONFIG (Set node)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing or on-demand runs.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Triggers CONFIG node.  
  - Edge cases: None significant.

- **Start every hour past 11 minutes**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every hour at 11 minutes past the hour.  
  - Configuration: Cron expression `11 */1 * * *`  
  - Inputs: None  
  - Outputs: Triggers CONFIG node.  
  - Edge cases: Cron misconfiguration could cause missed runs.

- **CONFIG**  
  - Type: Set  
  - Role: Defines workflow parameters:  
    - `min_traffic` (number): Minimum approximate traffic threshold (default 500)  
    - `max_results` (number): Maximum number of keywords to process (default 3)  
    - `jina_key` (string): API key for Jina AI (empty by default)  
  - Inputs: Trigger nodes  
  - Outputs: Triggers "Get saved keywords" node.  
  - Edge cases: Missing or invalid Jina AI key will cause scraping failures.

---

#### 1.2 Google Trends Data Retrieval & Parsing

**Overview:**  
This block fetches the Google Trends RSS feed for Italy, parses the XML response into JSON, preparing it for further processing.

**Nodes Involved:**  
- Get saved keywords (Google Sheets read)  
- GoogleTrends (HTTP Request)  
- XML (XML to JSON conversion)

**Node Details:**

- **Get saved keywords**  
  - Type: Google Sheets (Read)  
  - Role: Retrieves existing keywords stored in the Google Sheet to avoid duplicates.  
  - Configuration: Reads from a configured Google Sheet document and sheet (gid=0). Uses OAuth2 credentials.  
  - Inputs: CONFIG node  
  - Outputs: Triggers GoogleTrends node.  
  - Edge cases: Authentication errors, empty or missing sheet, API quota limits.

- **GoogleTrends**  
  - Type: HTTP Request  
  - Role: Fetches the Google Trends RSS feed from `https://trends.google.it/trending/rss?geo=IT`.  
  - Configuration: Default GET request, retries on failure enabled.  
  - Inputs: Get saved keywords node  
  - Outputs: Triggers XML node.  
  - Edge cases: Network errors, RSS feed unavailability, rate limiting.

- **XML**  
  - Type: XML  
  - Role: Converts the RSS XML feed into JSON format for easier manipulation.  
  - Configuration: Normalization off, explicitArray false to simplify structure.  
  - Inputs: GoogleTrends node  
  - Outputs: Triggers New keywords node.  
  - Edge cases: Malformed XML, parsing errors.

---

#### 1.3 Existing Data Fetch & Filtering

**Overview:**  
This block filters the parsed RSS items to exclude keywords already present in Google Sheets and those below the minimum traffic threshold. It limits the number of processed keywords to the configured maximum.

**Nodes Involved:**  
- New keywords (Code)

**Node Details:**

- **New keywords**  
  - Type: Code (JavaScript)  
  - Role:  
    - Extracts up to 3 news items per trending keyword.  
    - Parses approximate traffic values, filters out keywords below `min_traffic`.  
    - Filters out keywords already saved in Google Sheets.  
    - Sorts by traffic descending and limits results to `max_results`.  
  - Key expressions:  
    - Accesses `CONFIG` node for parameters.  
    - Accesses `Get saved keywords` node to get existing keywords.  
    - Processes `XML` node JSON data.  
  - Inputs: XML node  
  - Outputs: Triggers Loop Over Items node.  
  - Edge cases: Empty RSS feed, missing traffic data, malformed news items, empty saved keywords list.

---

#### 1.4 Content Scraping & Summarization

**Overview:**  
For each filtered trending keyword, this block scrapes the content of up to three related news article URLs using Jina AI’s API, cleaning HTML and extracting text content.

**Nodes Involved:**  
- Loop Over Items (Split in Batches)  
- content1 (HTTP Request)  
- content2 (HTTP Request)  
- content3 (HTTP Request)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each trending keyword item individually to handle scraping sequentially.  
  - Inputs: New keywords node  
  - Outputs: Triggers Mapping and content1 nodes in parallel.  
  - Edge cases: Large batch sizes may cause timeouts; default batch size not explicitly set.

- **content1, content2, content3**  
  - Type: HTTP Request  
  - Role: Each node calls Jina AI API to scrape and clean the content of one of the three news article URLs associated with the trending keyword.  
  - Configuration:  
    - URL dynamically set to `https://r.jina.ai/{news_item_urlX}` where X=1,2,3.  
    - Headers include Authorization Bearer token from `CONFIG.jina_key`.  
    - Additional headers instruct Jina AI to remove certain HTML selectors (links, scripts, images, etc.) and return plain text.  
  - Inputs:  
    - content1 triggered by Loop Over Items  
    - content2 triggered by content1  
    - content3 triggered by content2  
  - Outputs: content3 triggers Loop Over Items again to continue processing.  
  - Edge cases:  
    - Missing or invalid Jina AI key causes auth errors.  
    - URLs that are unreachable or return errors.  
    - Jina AI API rate limits or downtime.  
    - Scraped content may be empty or too short.

---

#### 1.5 Data Mapping & Conditional Saving

**Overview:**  
This block consolidates the scraped content into a single summary, maps all relevant fields for Google Sheets, checks if the summary is sufficiently long, and conditionally saves the record to Google Sheets.

**Nodes Involved:**  
- Mapping (Set)  
- If we have scraped min 1 url -> Save (If)  
- Google Sheets (Append)  
- All scraping node failed. Don't save record without summary (NoOp)

**Node Details:**

- **Mapping**  
  - Type: Set  
  - Role:  
    - Combines the text from the three scraped contents into one `summary` string separated by "---".  
    - Maps all fields from the "New keywords" node to the Google Sheets schema, including keyword, traffic, publication date, URLs, titles, images, sources, and status ("idea").  
  - Key expressions: Uses expressions like `$('content1').item.json.data.text.replaceAll('\n', ' ').trim()` to clean text.  
  - Inputs: Loop Over Items node  
  - Outputs: Triggers If node.  
  - Edge cases: If any content node failed, summary parts may be empty.

- **If we have scraped min 1 url -> Save**  
  - Type: If  
  - Role: Checks if the combined summary length is greater than 100 characters to decide whether to save the record.  
  - Condition: `{{$json.summary.length > 100}}`  
  - Inputs: Mapping node  
  - Outputs:  
    - True: triggers Google Sheets append node.  
    - False: triggers NoOp node.  
  - Edge cases: Summary length check may exclude valid but short summaries.

- **Google Sheets**  
  - Type: Google Sheets (Append)  
  - Role: Appends the mapped data as a new row in the configured Google Sheet.  
  - Configuration:  
    - Uses OAuth2 credentials.  
    - Maps all fields explicitly to columns as per schema.  
    - Limits abstract field to 49,999 characters.  
  - Inputs: If node (True branch)  
  - Outputs: None  
  - Edge cases: Authentication errors, API quota limits, sheet access issues.

- **All scraping node failed. Don't save record without summary**  
  - Type: NoOp  
  - Role: Placeholder node for the False branch of the If node, effectively discarding records without sufficient summary.  
  - Inputs: If node (False branch)  
  - Outputs: None

---

### 3. Summary Table

| Node Name                                | Node Type          | Functional Role                                  | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                              |
|-----------------------------------------|--------------------|-------------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’            | Manual Trigger     | Manual start of workflow                         | None                             | CONFIG                           |                                                                                                        |
| Start every hour past 11 minutes         | Schedule Trigger   | Scheduled hourly trigger at 11 minutes past hour| None                             | CONFIG                           | ## Cron trigger Google Trends update the RSS feed every 10 minutes. This will start wordflow 1 minute after. |
| CONFIG                                  | Set                | Sets workflow parameters (min_traffic, max_results, Jina AI key) | When clicking ‘Test workflow’, Start every hour past 11 minutes | Get saved keywords               | ## CONFIGURATION min_traffic is a numeric value. Google Trend RSS has approx traffic values 100, 200, 500, 1000 etc. max_result limits max rss to scrape. jina_key is the jina.ai API key |
| Get saved keywords                      | Google Sheets      | Reads existing keywords from Google Sheet       | CONFIG                           | GoogleTrends                    | ## Google Sheet Database This is main sheet where all your Editorial plan will be saved. The column status value could be a trigger for other automations |
| GoogleTrends                           | HTTP Request       | Fetches Google Trends RSS feed                   | Get saved keywords               | XML                            | ## Google Trends request We get last keyword in trend. Every item has a main keyword and 3 URL. We will use those url to scrape content and generate a combined summary |
| XML                                    | XML                | Converts RSS XML to JSON                          | GoogleTrends                    | New keywords                   | ## Simple conversion Converts XML RSS into a more readable json object                                  |
| New keywords                           | Code (JavaScript)  | Filters new keywords by traffic and duplicates  | XML                            | Loop Over Items                | ## Building dataset Here we limits results, filter by min traffic and flatten RSS structure to adapt to Google Sheet, fields renamed. Compares RSS and Google Sheet to find new keywords. |
| Loop Over Items                        | SplitInBatches     | Processes each keyword item individually         | New keywords                   | Mapping, content1              | ## Data mapping Here you have all fields needed in Google Sheet. While done, the content of 3 website linked in Google Trends RSS will be merged in a single Summary field |
| content1                              | HTTP Request       | Scrapes content from first news article URL     | Loop Over Items                | content2                      | ## Scraping Here jina.ai will get text content from 3 Google Trends URLs                              |
| content2                              | HTTP Request       | Scrapes content from second news article URL    | content1                      | content3                      | ## Scraping Here jina.ai will get text content from 3 Google Trends URLs                              |
| content3                              | HTTP Request       | Scrapes content from third news article URL     | content2                      | Loop Over Items               | ## Scraping Here jina.ai will get text content from 3 Google Trends URLs                              |
| Mapping                              | Set                | Combines scraped content and maps fields for Google Sheets | Loop Over Items                | If we have scraped min 1 url -> Save | ## Data mapping Here you have all fields needed in Google Sheet. While done, the content of 3 website linked in Google Trends RSS will be merged in a single Summary field |
| If we have scraped min 1 url -> Save  | If                 | Checks if summary length > 100 to decide saving | Mapping                       | Google Sheets (True), NoOp (False) | ## Data check Sometimes scraping HTML content fails (for some reasons), that's normal but this should avoid saving zero content if all 3 scraping nodes fail |
| Google Sheets                        | Google Sheets      | Appends new keyword and summary to Google Sheet | If we have scraped min 1 url -> Save (True) | None                         | ## Saving output                                                                                         |
| All scraping node failed. Don't save record without summary | NoOp               | Discards records without sufficient summary     | If we have scraped min 1 url -> Save (False) | None                         | ## Data check Sometimes scraping HTML content fails (for some reasons), that's normal but this should avoid saving zero content if all 3 scraping nodes fail |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No parameters.  
   - Add a **Schedule Trigger** node named `Start every hour past 11 minutes`. Set cron expression to `11 */1 * * *` to run hourly at 11 minutes past.

2. **Create Configuration Node**  
   - Add a **Set** node named `CONFIG`.  
   - Add three fields:  
     - `min_traffic` (Number): default 500  
     - `max_results` (Number): default 3  
     - `jina_key` (String): leave empty initially, to be filled with your Jina AI API key.  
   - Connect both trigger nodes to this node.

3. **Create Google Sheets Read Node**  
   - Add a **Google Sheets** node named `Get saved keywords`.  
   - Set operation to "Read Rows".  
   - Configure with your Google Sheets OAuth2 credentials.  
   - Set the document ID and sheet name (gid=0) to your prepared Google Sheet.  
   - Connect `CONFIG` node output to this node.

4. **Create HTTP Request Node for Google Trends**  
   - Add an **HTTP Request** node named `GoogleTrends`.  
   - Set method to GET.  
   - URL: `https://trends.google.it/trending/rss?geo=IT` (modify geo param as needed).  
   - Enable "Retry on Fail".  
   - Connect `Get saved keywords` node output to this node.

5. **Create XML to JSON Conversion Node**  
   - Add an **XML** node named `XML`.  
   - Set "Normalize" to false, "Explicit Array" to false.  
   - Connect `GoogleTrends` node output to this node.

6. **Create Filtering Code Node**  
   - Add a **Code** node named `New keywords`.  
   - Paste the provided JavaScript code that:  
     - Reads `min_traffic`, `max_results` from `CONFIG`.  
     - Reads existing keywords from `Get saved keywords`.  
     - Parses RSS items from `XML`.  
     - Filters by traffic and duplicates.  
     - Limits results to `max_results`.  
   - Connect `XML` node output to this node.

7. **Create SplitInBatches Node**  
   - Add a **SplitInBatches** node named `Loop Over Items`.  
   - Connect `New keywords` node output to this node.

8. **Create HTTP Request Nodes for Content Scraping**  
   - Add three **HTTP Request** nodes named `content1`, `content2`, `content3`.  
   - For each:  
     - URL: `https://r.jina.ai/{{ $json.news_item_urlX }}` where X=1,2,3 respectively.  
     - Method: GET.  
     - Add headers:  
       - `Authorization`: `Bearer {{ $json.jina_key }}` (from `CONFIG`)  
       - `Accept`: `application/json`  
       - `X-Remove-Selector`: `a, link, script, footer, img, svg`  
       - `X-Retain-Images`: `none`  
       - `X-Return-Format`: `text`  
   - Connect `Loop Over Items` to `content1`, then chain `content1` → `content2` → `content3`.  
   - Connect `content3` output back to `Loop Over Items` (to continue batch processing).

9. **Create Mapping Node**  
   - Add a **Set** node named `Mapping`.  
   - Map all fields required by Google Sheets, including:  
     - `summary`: concatenate `content1`, `content2`, `content3` text with "---" separators, removing newlines and trimming.  
     - Copy all keyword and news item fields from `New keywords` node.  
     - Set `status` to `"idea"`.  
   - Connect `Loop Over Items` node output to this node.

10. **Create Conditional Save Node**  
    - Add an **If** node named `If we have scraped min 1 url -> Save`.  
    - Condition: Expression `{{$json.summary.length > 100}}` (summary length greater than 100).  
    - Connect `Mapping` node output to this node.

11. **Create Google Sheets Append Node**  
    - Add a **Google Sheets** node named `Google Sheets`.  
    - Operation: Append Row.  
    - Configure with your Google Sheets OAuth2 credentials and target document/sheet.  
    - Map all fields explicitly as per the schema, limiting `abstract` (summary) to 49,999 characters.  
    - Connect the True output of the If node to this node.

12. **Create NoOp Node**  
    - Add a **NoOp** node named `All scraping node failed. Don't save record without summary`.  
    - Connect the False output of the If node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Google Trends RSS updates every 10 minutes; workflow scheduled 1 minute after to capture updates | Sticky Note on "Start every hour past 11 minutes" node                                         |
| `min_traffic` filters keywords by approximate traffic (e.g., 100, 200, 500, 1000)                | Sticky Note on CONFIG node                                                                      |
| `max_results` limits number of keywords processed per run                                        | Sticky Note on CONFIG node                                                                      |
| Jina AI API key required for content scraping                                                   | Sticky Note on CONFIG node                                                                      |
| Google Sheet stores editorial plan; `status` column can trigger other automations                | Sticky Note on "Get saved keywords" node                                                       |
| RSS items contain main keyword and 3 URLs; these URLs are scraped and summarized                  | Sticky Note on "GoogleTrends" node                                                             |
| XML node converts RSS feed to JSON for easier processing                                        | Sticky Note on "XML" node                                                                       |
| Filtering code flattens RSS structure and compares with saved keywords to find new entries       | Sticky Note on "New keywords" node                                                             |
| Summary combines text from 3 scraped articles separated by "---"                                | Sticky Note on "Mapping" node                                                                  |
| Records with insufficient summary content are not saved                                         | Sticky Note on "If we have scraped min 1 url -> Save" node                                     |
| Jina AI scraping removes unwanted HTML elements for clean text extraction                        | Sticky Notes on `content1`, `content2`, `content3` nodes                                       |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the "Extract Google Trends Keywords & Summarize Articles in Google Sheets" workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and automation agents to work effectively with this workflow.