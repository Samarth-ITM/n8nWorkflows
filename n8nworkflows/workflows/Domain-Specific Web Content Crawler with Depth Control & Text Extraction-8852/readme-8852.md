Domain-Specific Web Content Crawler with Depth Control & Text Extraction

https://n8nworkflows.xyz/workflows/domain-specific-web-content-crawler-with-depth-control---text-extraction-8852


# Domain-Specific Web Content Crawler with Depth Control & Text Extraction

### 1. Workflow Overview

This n8n workflow implements a **domain-specific web content crawler** that recursively fetches web pages starting from a given root URL, extracts textual content and links from each page, and limits crawling depth to prevent infinite loops. It focuses strictly on crawling pages within the same domain (including apex and www variants), avoids non-HTML content like PDF or DOCX files, and deduplicates URLs to ensure efficiency. The output is a consolidated, chunked JSON response containing the collected page data, triggered and returned via a webhook.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Initialization:** Receives the root URL through a webhook, initializes crawl parameters and global static data for tracking visited URLs, queued links, and collected pages.
- **1.2 Crawl Loop Core:** Fetches HTML content for URLs, extracts links and body text, filters and deduplicates URLs, and manages crawl depth and queue state.
- **1.3 Loop Control & Recursion:** Iterates over the queued URLs one at a time, feeding them back into the crawl core to continue crawling up to the max depth.
- **1.4 Page Data Collection & Output:** Stores crawled page data, aggregates all pages when crawling completes, chunks the data for manageable output size, and responds via the webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

**Overview:**  
This block handles receiving the initial crawl request via webhook, extracts the root URL, sets crawl parameters including maximum depth, and initializes global static data structures to track crawl progress.

**Nodes Involved:**  
- Webhook  
- Init Crawl Params (Set Node)  
- Init Globals (Code Node)  
- Seed Root Crawl Item (Merge Node)  

**Node Details:**

- **Webhook**  
  - Type: Webhook (Entry point)  
  - Role: Receives POST requests with payload containing a `url` key to start crawling.  
  - Config: Path set to unique webhook ID, HTTP method POST, response mode set to respond node.  
  - Inputs: External HTTP POST request  
  - Outputs: JSON containing `body.url`  
  - Failure cases: Invalid or missing URL in payload, malformed requests.

- **Init Crawl Params (Set Node)**  
  - Type: Set  
  - Role: Extracts and sets initial crawl parameters: root URL, domain, maxDepth=3, initial depth=0.  
  - Config: Sets `url` and `domain` from webhook `body.url`, `maxDepth=3`, `depth=0`. Keeps only these fields.  
  - Inputs: Webhook output  
  - Outputs: JSON with `{url, domain, maxDepth, depth}`  
  - Edge cases: Malformed URLs may propagate incorrect domain; no validation here.

- **Init Globals (Code Node)**  
  - Type: Code  
  - Role: Initializes static global data for crawl tracking:  
    - `pending` (count of URLs to crawl) = 1 (starts with root)  
    - `visited` (array of crawled URLs) = []  
    - `queued` (dictionary of URLs queued but not crawled) = {}  
    - `pages` (array of collected page data) = []  
  - Also normalizes domain from URL, with fallback for malformed URLs.  
  - Inputs: Output from Init Crawl Params  
  - Outputs: JSON with normalized domain and initialized fields  
  - Edge cases: If URL is malformed, attempts fallback string parsing to extract domain.

- **Seed Root Crawl Item (Merge Node)**  
  - Type: Merge  
  - Role: Combines the crawl params and global static data into one item for the crawl loop.  
  - Config: Combine mode by position, prefer last on clashes, includes unpaired data.  
  - Inputs: Outputs from Init Globals and Init Crawl Params  
  - Outputs: Unified JSON object with crawl parameters and globals  
  - Edge cases: Merge conflicts resolved by preferring last input.

---

#### 2.2 Crawl Loop Core

**Overview:**  
This block fetches the HTML content of the current URL, extracts body text and links, normalizes and deduplicates links, manages the crawl queue and depth, and prepares items for either further crawling or page storage.

**Nodes Involved:**  
- Fetch HTML Page (HTTP Request)  
- Attach URL/Depth to HTML (Code)  
- Extract Body & Links (HTML)  
- Queue & Dedup Links (Code)  
- IF Crawl Depth OK? (If)  
- Requeue Link Item (Code)  
- Store Page Data (Set)  

**Node Details:**

- **Fetch HTML Page (HTTP Request)**  
  - Type: HTTP Request  
  - Role: Performs GET request to fetch the HTML content of the URL.  
  - Config: URL set dynamically from JSON field `url`, timeout 5s, onError set to continue (skip failed URLs).  
  - Inputs: Seed Root Crawl Item output with URL  
  - Outputs: Raw HTML content in response body  
  - Edge cases: HTTP errors, timeouts, non-HTML responses. Continues on error without stopping workflow.

- **Attach URL/Depth to HTML (Code)**  
  - Type: Code  
  - Role: Attaches the original URL and crawl depth metadata to the HTML response item for downstream processing.  
  - Config: Runs once per item, reads URL and depth from Seed Root Crawl Item, merges with HTML response JSON.  
  - Inputs: Fetch HTML Page response  
  - Outputs: JSON containing `{url, depth, ...htmlResponse}`  
  - Edge cases: Missing or malformed fields from upstream.

- **Extract Body & Links (HTML Node)**  
  - Type: HTML Extract  
  - Role: Parses HTML content to extract the body text and all anchor tag href attributes.  
  - Config: Operation `extractHtmlContent`; extracts `content` from `<body>`, trims and cleans text; extracts `links` as array of href attributes from `a[href]`.  
  - Inputs: Output from Attach URL/Depth to HTML  
  - Outputs: JSON with `content` (string) and `links` (array)  
  - Edge cases: Malformed HTML, missing body tags, empty links array.

- **Queue & Dedup Links (Code)**  
  - Type: Code  
  - Role: Core logic to normalize, filter, deduplicate, and queue new links for crawling. Also marks current URL as visited and updates pending counter.  
  - Config:  
    - Normalizes URLs (absolute URLs, strips trailing slashes).  
    - Filters out links with protocols like mailto, tel, javascript, anchors (#), and file extensions (pdf, docx, etc).  
    - Only queues links within same domain (apex or www variants).  
    - Ensures no duplicates in visited or queued.  
    - Updates staticData: visited[], queued{}, pending count.  
    - Outputs new link items for crawling (type='link') and current page item (type='page').  
  - Inputs: Extract Body & Links output  
  - Outputs: Array of new link items and current page item  
  - Edge cases: Handling relative URLs, malformed URLs, external domains, duplicates. Runs with onError continue to avoid blocking.

- **IF Crawl Depth OK? (If Node)**  
  - Type: If  
  - Role: Checks if the current item is a link and if its depth is within maxDepth.  
  - Config: Conditions: `type === 'link'` AND `depth <= maxDepth`  
  - Inputs: Output from Queue & Dedup Links  
  - Outputs:  
    - True: Items to be requeued for crawling  
    - False: Items to be stored as crawled pages  
  - Edge cases: Possible missing or malformed `type` or `depth` fields.

- **Requeue Link Item (Code)**  
  - Type: Code  
  - Role: Cleans up the link item by removing the internal `type` field before re-queuing for crawl.  
  - Config: Runs once per item, deletes `$json.type`.  
  - Inputs: True branch from IF node  
  - Outputs: Cleaned item to be fed back into the crawl loop.  
  - Edge cases: Missing `type` field; safe to delete anyway.

- **Store Page Data (Set Node)**  
  - Type: Set  
  - Role: Captures and keeps essential page data: URL, content, and depth for storage/export.  
  - Config: Sets fields `url`, `content`, `depth` from incoming item, keeps only set fields.  
  - Inputs: False branch from IF node  
  - Outputs: Clean page data item  
  - Edge cases: Missing content or depth fields.

---

#### 2.3 Loop Control & Recursion

**Overview:**  
This block iterates through the queued links one by one, feeding each back into the crawl core for processing, thus enabling recursive crawling up to the max depth.

**Nodes Involved:**  
- Loop Links (Batches) (SplitInBatches)  
- Seed Root Crawl Item (Merge)  
- Fetch HTML Page (HTTP Request) (reused)  

**Node Details:**

- **Loop Links (Batches) (SplitInBatches)**  
  - Type: SplitInBatches  
  - Role: Iterates through the queue of new link items one at a time to control crawl pacing and resource usage.  
  - Config: Batch size = 1, no reset after batch completion.  
  - Inputs: Cleaned link items from Requeue Link Item  
  - Outputs: Single link item per iteration fed back into Seed Root Crawl Item  
  - Edge cases: Empty queues halt iteration; no reset prevents infinite loops.

- **Seed Root Crawl Item (Merge Node) [Reused]**  
  - Role: Merges current crawl item (URL, depth) with globals for next fetch.  
  - Inputs: Output from Loop Links (Batches) and Init Globals  
  - Outputs: Next crawl item for Fetch HTML Page  
  - Edge cases: Merge conflicts handled by config.

- **Fetch HTML Page (HTTP Request) [Reused]**  
  - As above, reuses fetch node for recursive crawl.

---

#### 2.4 Page Data Collection & Output

**Overview:**  
This block collects the crawled page data, determines when all crawling tasks complete, merges all pages into batches, chunks large data for efficient processing, and sends the response back via webhook.

**Nodes Involved:**  
- Store Page Data (Set) [Reused]  
- Collect Pages & Emit When Done (Code)  
- Merge Web Pages (Merge)  
- Combine & Chunk (Code)  
- Respond to Webhook (RespondToWebhook)  

**Node Details:**

- **Collect Pages & Emit When Done (Code)**  
  - Type: Code  
  - Role: Appends each page item to global `pages` array; checks if `pending` <= 0 to know crawl is finished; if done, concatenates all page data into a single string output.  
  - Inputs: Stored page data from Store Page Data  
  - Outputs: Combined content JSON when done, else empty output to continue crawling  
  - Edge cases: Pending counter mismatch may cause premature or delayed emission.

- **Merge Web Pages (Merge Node)**  
  - Type: Merge  
  - Role: Combines page collection outputs with globals or other inputs, prepares for final chunking.  
  - Inputs: Output from Collect Pages & Emit When Done and Init Globals  
  - Outputs: Combined pages for chunking  
  - Edge cases: Merge conflicts resolved by default.

- **Combine & Chunk (Code Node)**  
  - Type: Code  
  - Role:  
    - Normalizes and merges all stored pages and any extra JSON data.  
    - Builds a large combined text content string with URL/depth/content per page.  
    - Chunks pages into groups by character count (max 12,000 chars per batch) and batch size (max 5 pages per chunk).  
    - Outputs batches indexed for downstream processing or API calls.  
  - Inputs: Merged pages from Merge Web Pages  
  - Outputs: Array of batch items with chunked page data and combined content string  
  - Edge cases: Large pages exceeding chunk size; ensures output is manageable.

- **Respond to Webhook (RespondToWebhook)**  
  - Type: RespondToWebhook  
  - Role: Sends the chunked crawl result batches as the HTTP response to the initial webhook call.  
  - Inputs: Output from Combine & Chunk  
  - Outputs: HTTP JSON response  
  - Edge cases: Large payloads; response timeouts if crawl is too long or data too big.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                                      | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                           |
|----------------------------|----------------------|-----------------------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                    | Webhook              | Entry point, receives crawl request                 | -                              | Init Crawl Params              |                                                                                                                                        |
| Init Crawl Params           | Set                  | Sets initial URL, domain, maxDepth, depth           | Webhook                        | Init Globals                  |                                                                                                                                        |
| Init Globals               | Code                 | Initializes static global crawl data                 | Init Crawl Params              | Seed Root Crawl Item, Merge Web Pages | "Initializes the pending count in static data for crawl completion tracking."                                                        |
| Seed Root Crawl Item       | Merge                | Combines crawl parameters and globals                | Init Globals, Init Crawl Params | Fetch HTML Page               |                                                                                                                                        |
| Fetch HTML Page            | HTTP Request         | Fetches HTML content for URL                          | Seed Root Crawl Item            | Attach URL/Depth to HTML      | "Makes HTTP request to fetch the content of the current URL."                                                                         |
| Attach URL/Depth to HTML   | Code                 | Attaches url and depth metadata to HTML response     | Fetch HTML Page                | Extract Body & Links          |                                                                                                                                        |
| Extract Body & Links       | HTML Extract         | Extracts body text and anchor href links             | Attach URL/Depth to HTML       | Queue & Dedup Links           | "Parses HTML content and extracts body text and anchor href links."                                                                   |
| Queue & Dedup Links        | Code                 | Normalizes, filters, deduplicates, and queues links  | Extract Body & Links           | IF Crawl Depth OK?            | "Cleans and deduplicates links. Tracks visited URLs. Prepares next crawl queue."                                                      |
| IF Crawl Depth OK?         | If                   | Checks if depth <= maxDepth and type=='link'         | Queue & Dedup Links            | Requeue Link Item (true), Store Page Data (false) | "Validates whether the current depth is below the maximum depth allowed."                                                            |
| Requeue Link Item          | Code                 | Removes internal 'type' field and requeues link      | IF Crawl Depth OK? (true)      | Loop Links (Batches)          | "Removes internal 'type' field and re-enqueues the link for next crawl."                                                              |
| Loop Links (Batches)       | SplitInBatches       | Iterates queued links one at a time                   | Requeue Link Item              | Seed Root Crawl Item          | "Iterates through the queue of links to be crawled one at a time."                                                                     |
| Store Page Data            | Set                  | Stores url, content, and depth of crawled page       | IF Crawl Depth OK? (false)     | Collect Pages & Emit When Done| "Captures the URL, page content, and depth for storage or export."                                                                     |
| Collect Pages & Emit When Done | Code              | Aggregates pages, emits combined content when done   | Store Page Data                | Merge Web Pages               | "Initializes the pending count in static data for crawl completion tracking." (same note applies, implied)                             |
| Merge Web Pages            | Merge                | Merges collected pages and globals                    | Collect Pages & Emit When Done, Init Globals | Combine & Chunk             |                                                                                                                                        |
| Combine & Chunk            | Code                 | Combines pages, chunks large content for output      | Merge Web Pages               | Respond to Webhook            | "Combines static pages + extra JSON, then chunk pages for model calls."                                                                |
| Respond to Webhook         | RespondToWebhook     | Sends final chunked crawl data as webhook response   | Combine & Chunk               | -                             |                                                                                                                                        |
| Sticky Note                | StickyNote           | Documentation and explanation                         | -                              | -                             | "# n8n Workflow Explanation: Web Crawler ... Depth-Limited Crawling ... Loop Mechanism ... Usage: Trigger via webhook with {\"url\": \"https://example.com\"}." |
| Sticky Note1               | StickyNote           | Step-by-step detailed breakdown of workflow          | -                              | -                             | "Step-by-step detailed breakdown ... Additional Notes ... Usage: Trigger via webhook with {\"url\": \"https://example.com\"}."          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Parameters:  
     - HTTP Method: POST  
     - Path: Unique identifier (e.g., "603a09ed-516c-4c7d-bad3-b05b030503a2")  
     - Response Mode: `responseNode`  
   - Purpose: Receives initial crawl request `{ "url": "https://example.com" }`.

2. **Create Set Node "Init Crawl Params"**  
   - Type: Set  
   - Parameters:  
     - Set fields:  
       - `url` = `{{$json.body.url}}`  
       - `domain` = `{{$json.body.url}}` (initial raw domain)  
       - `maxDepth` = `3` (number)  
       - `depth` = `0` (number)  
     - Keep only these fields  
   - Connect output from Webhook.

3. **Create Code Node "Init Globals"**  
   - Type: Code  
   - Parameters:  
     - JavaScript code to initialize global static data:  
       - `pending = 1`  
       - `visited = []`  
       - `queued = {}`  
       - `pages = []`  
     - Normalize domain from input URL with fallback parsing.  
   - Connect output from Init Crawl Params.

4. **Create Merge Node "Seed Root Crawl Item"**  
   - Type: Merge  
   - Parameters:  
     - Mode: Combine  
     - Combine By: Position  
     - Clash Handling: Prefer last, override empty  
     - Include Unpaired: true  
   - Connect outputs from Init Globals and Init Crawl Params as inputs 1 and 2.

5. **Create HTTP Request Node "Fetch HTML Page"**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `{{$json.url}}` (dynamic)  
     - Options: Timeout 5000ms  
     - On Error: Continue (skip errors)  
   - Connect output from Seed Root Crawl Item.

6. **Create Code Node "Attach URL/Depth to HTML"**  
   - Type: Code  
   - Parameters:  
     - JavaScript to attach `url` and `depth` from Seed Root Crawl Item to HTML response item.  
     - Run once per item.  
   - Connect output from Fetch HTML Page.

7. **Create HTML Extract Node "Extract Body & Links"**  
   - Type: HTML  
   - Parameters:  
     - Operation: extractHtmlContent  
     - Extraction Values:  
       - `content` from `body` selector, trimmed and cleaned  
       - `links` (array) from `a[href]` attribute `href`  
   - Connect output from Attach URL/Depth to HTML.

8. **Create Code Node "Queue & Dedup Links"**  
   - Type: Code  
   - Parameters:  
     - JavaScript code to:  
       - Normalize URLs to absolute, strip trailing slashes  
       - Filter links by protocol, anchor, file type  
       - Deduplicate against staticData.visited and staticData.queued  
       - Only queue links within same domain (apex/www variants)  
       - Update staticData.pending, visited, queued accordingly  
       - Output new link items (type='link') and current page item (type='page')  
     - On Error: Continue  
   - Connect output from Extract Body & Links.

9. **Create If Node "IF Crawl Depth OK?"**  
   - Type: If  
   - Parameters:  
     - Check `type === 'link'` AND `depth <= maxDepth`  
   - Connect output from Queue & Dedup Links.

10. **Create Code Node "Requeue Link Item"**  
    - Type: Code  
    - Parameters:  
      - Delete internal `type` field from JSON  
    - Connect True branch from IF Crawl Depth OK?.

11. **Create SplitInBatches Node "Loop Links (Batches)"**  
    - Type: SplitInBatches  
    - Parameters:  
      - Batch Size: 1  
      - Reset: false  
    - Connect output from Requeue Link Item.

12. **Reconnect Loop Back to "Seed Root Crawl Item"**  
    - Connect output from Loop Links (Batches) to second input of Seed Root Crawl Item node to enable recursion.

13. **Create Set Node "Store Page Data"**  
    - Type: Set  
    - Parameters:  
      - Fields:  
        - `url` = `{{$json.url}}`  
        - `content` = `{{$json.content}}`  
        - `depth` = `{{$json.depth || 0}}`  
      - Keep only these fields  
    - Connect False branch from IF Crawl Depth OK?.

14. **Create Code Node "Collect Pages & Emit When Done"**  
    - Type: Code  
    - Parameters:  
      - Append page to `staticData.pages`  
      - If `staticData.pending <= 0` then concatenate all pages into a string and output combined content  
      - Else output empty array to continue crawling  
    - Connect output from Store Page Data.

15. **Create Merge Node "Merge Web Pages"**  
    - Type: Merge  
    - Parameters: Defaults  
    - Connect output from Collect Pages & Emit When Done and Init Globals as inputs.

16. **Create Code Node "Combine & Chunk"**  
    - Type: Code  
    - Parameters:  
      - Normalize and merge stored pages with any extra JSON data  
      - Combine all page contents into one string  
      - Chunk pages by max 12000 characters per batch and groups of 5 pages each  
      - Output batches with batchIndex, pages, combinedContent, accId  
    - Connect output from Merge Web Pages.

17. **Create RespondToWebhook Node "Respond to Webhook"**  
    - Type: RespondToWebhook  
    - Parameters: Defaults  
    - Connect output from Combine & Chunk to respond with final data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses n8n's **static global data** to track crawl state (pending URLs, visited list, queued dictionary, and collected pages) across asynchronous executions to enable depth-limited recursive crawling with proper termination.                                                                                                                                                                                                                                                                                                                                                         | n8n Static Data Documentation: https://docs.n8n.io/code-examples/static-data/                                                        |
| For URL normalization and domain matching, the workflow treats apex and www subdomains as equivalent to stay strictly on-site. It also excludes common non-HTML file extensions and special schemes like mailto, tel, and javascript.                                                                                                                                                                                                                                                                                                                                                          | URL and domain handling logic is in the "Queue & Dedup Links" code node.                                                              |
| The crawl depth limit (default 3) prevents infinite loops and excessive resource use. The SplitInBatches node controls processing by crawling one URL at a time, allowing manageable throughput and easier debugging.                                                                                                                                                                                                                                                                                                                                                                           | Batch size = 1 in "Loop Links (Batches)" node.                                                                                        |
| The workflow continues on HTTP request errors to avoid blocking on inaccessible pages. It logs and skips such pages gracefully.                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Configured in "Fetch HTML Page" node (onError=continue).                                                                              |
| Output is chunked to 12,000 characters max and batches of 5 pages to handle large crawls efficiently and avoid exceeding payload size limits in downstream systems or APIs.                                                                                                                                                                                                                                                                                                                                                                                                                   | Logic in "Combine & Chunk" code node.                                                                                                |
| Trigger example: POST JSON `{ "url": "https://example.com" }` to the webhook URL. The response will be JSON containing batched crawled page content.                                                                                                                                                                                                                                                                                                                                                                                                                                          | See Sticky Notes for detailed workflow explanation and usage notes.                                                                  |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.