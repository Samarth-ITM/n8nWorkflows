Insurance News Aggregation Keyword Analysis & Database Storage via Supabase

https://n8nworkflows.xyz/workflows/insurance-news-aggregation-keyword-analysis---database-storage-via-supabase-9507


# Insurance News Aggregation Keyword Analysis & Database Storage via Supabase

### 1. Workflow Overview

This workflow automates the aggregation of insurance-related news from multiple sources, analyzes the content for relevant keywords, and stores the processed data into two separate Supabase databases: a Content Library and a Knowledge Base. It is designed for periodic execution every six hours to maintain an up-to-date repository of insurance news with enriched metadata for further use, such as analytics or content curation.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Source Identification**: Periodic initiation and determination of news sources with distinction between RSS feeds and other web content.

- **1.2 Content Retrieval**: Fetching news articles from RSS feeds, Google News RSS, or through direct web scraping depending on the source type.

- **1.3 Article Processing**: Parsing articles, optionally fetching full article content, and enforcing rate limits when necessary.

- **1.4 Content Extraction and Keyword Analysis**: Extracting meaningful content and keywords from articles using code logic.

- **1.5 Relevance Filtering and Storage**: Filtering relevant articles and storing them into two Supabase databases; includes error handling for non-relevant content or failures.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Source Identification

- **Overview:**  
This block triggers the workflow every six hours and retrieves the list of news sources. It then decides whether each source is an RSS feed or not, steering subsequent processing accordingly.

- **Nodes Involved:**  
  - Schedule Every 6 Hours  
  - Get News Sources  
  - Is RSS Feed?

- **Node Details:**

  - **Schedule Every 6 Hours**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow on a fixed 6-hour interval.  
    - Configuration: Default schedule trigger with interval set to 6 hours.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get News Sources".  
    - Edge Cases: Workflow will not run if n8n is offline or schedule misconfigured.

  - **Get News Sources**  
    - Type: Function  
    - Role: Provides a list or array of news sources to process.  
    - Configuration: Custom JavaScript function; likely returns URLs and metadata indicating source type.  
    - Inputs: Trigger from Schedule node.  
    - Outputs: Connects to "Is RSS Feed?" node.  
    - Edge Cases: Empty list or malformed source data could cause downstream failures.

  - **Is RSS Feed?**  
    - Type: If (Conditional)  
    - Role: Determines if the current news source is an RSS feed.  
    - Configuration: Expression-based condition evaluating source metadata.  
    - Inputs: Receives news source data from "Get News Sources".  
    - Outputs:  
      - True branch: fetch RSS feed (via "Fetch RSS Feed")  
      - False branch: fetch Google News RSS or scrape website content.  
    - Edge Cases: Misclassification can cause inappropriate node execution paths.

#### 1.2 Content Retrieval

- **Overview:**  
This block fetches articles from the identified sources using RSS feed readers or web scraping HTTP requests. It manages multiple fetching strategies depending on the source classification.

- **Nodes Involved:**  
  - Fetch RSS Feed  
  - Fetch Google News RSS  
  - Scrape Web Page

- **Node Details:**

  - **Fetch RSS Feed**  
    - Type: RSS Feed Read  
    - Role: Reads articles from an RSS feed URL.  
    - Configuration: URL parameter dynamically set from the input source.  
    - Inputs: True output of "Is RSS Feed?"  
    - Outputs: Connects to "Parse Articles".  
    - Edge Cases: RSS feed unavailability, malformed RSS, HTTP errors handled by continueOnFail.

  - **Fetch Google News RSS**  
    - Type: RSS Feed Read  
    - Role: Fetches Google News RSS feeds when source is not a standard RSS feed.  
    - Configuration: URL parameter dynamically set based on source.  
    - Inputs: False output branch of "Is RSS Feed?"  
    - Outputs: Connects to "Parse Articles".  
    - Edge Cases: Google RSS feed format changes or restrictions, HTTP errors.

  - **Scrape Web Page**  
    - Type: HTTP Request  
    - Role: Performs direct web scraping for news sources without RSS feeds.  
    - Configuration: HTTP GET request to source URL with appropriate headers.  
    - Inputs: False branch of "Is RSS Feed?" (parallel with Fetch Google News RSS)  
    - Outputs: Connects to "Parse Articles".  
    - Edge Cases: Site layout changes, anti-scraping protections, timeouts; errors continue gracefully.

#### 1.3 Article Processing

- **Overview:**  
This block parses the fetched articles from various sources, optionally fetches full article content when only summaries are available, and enforces rate limiting to avoid API or server overload.

- **Nodes Involved:**  
  - Parse Articles  
  - Fetch Full Article  
  - Rate Limit Wait

- **Node Details:**

  - **Parse Articles**  
    - Type: Function  
    - Role: Extracts article metadata such as title, link, publication date from raw RSS or scraped content.  
    - Configuration: Custom JavaScript for parsing different content formats.  
    - Inputs: Data from RSS/HTTP fetch nodes.  
    - Outputs: Connects to "Fetch Full Article".  
    - Edge Cases: Unexpected data formats, missing fields handled by continueOnFail.

  - **Fetch Full Article**  
    - Type: HTTP Request  
    - Role: Retrieves the full text of the article if only partial content was initially available.  
    - Configuration: HTTP GET request dynamically targeted at article URLs.  
    - Inputs: Output from "Parse Articles".  
    - Outputs: Connects to both "Extract Content & Keywords" and "Rate Limit Wait".  
    - Edge Cases: 404 errors, timeouts, paywalls; continueOnFail ensures flow continuity.

  - **Rate Limit Wait**  
    - Type: Wait  
    - Role: Introduces a delay to respect rate limits or avoid server overload.  
    - Configuration: Wait time parameter, likely a few seconds to minutes.  
    - Inputs: Parallel output from "Fetch Full Article".  
    - Outputs: None (terminal for rate-limiting branch).  
    - Edge Cases: None significant; prevents excessive requests.

#### 1.4 Content Extraction and Keyword Analysis

- **Overview:**  
Processes the full article content to extract meaningful text and analyze keywords for relevance to insurance topics.

- **Nodes Involved:**  
  - Extract Content & Keywords

- **Node Details:**

  - **Extract Content & Keywords**  
    - Type: Code  
    - Role: Custom code node that extracts article content and identifies keywords, possibly using natural language processing techniques or keyword matching.  
    - Configuration: Contains JavaScript or other code to parse the article body and generate keywords.  
    - Inputs: Article content from "Fetch Full Article".  
    - Outputs: Connects to "Is Relevant?" to decide storage.  
    - Edge Cases: Parsing failures, keyword extraction inaccuracies; continueOnFail used to prevent workflow halt.

#### 1.5 Relevance Filtering and Storage

- **Overview:**  
Filters articles based on relevance criteria and stores them into two Supabase databases: one for content aggregation, one for knowledge base enrichment. Includes error handling for non-relevant or failed cases.

- **Nodes Involved:**  
  - Is Relevant?  
  - Store in Content Library  
  - Store in Knowledge Base  
  - Error Handler

- **Node Details:**

  - **Is Relevant?**  
    - Type: If (Conditional)  
    - Role: Evaluates if the article meets relevance conditions based on extracted keywords or other rules.  
    - Configuration: Expression logic comparing keywords or scores against thresholds.  
    - Inputs: From "Extract Content & Keywords".  
    - Outputs:  
      - True branch: proceeds to store in databases  
      - False branch: triggers "Error Handler".  
    - Edge Cases: Incorrect filtering could exclude relevant content or include noise.

  - **Store in Content Library**  
    - Type: HTTP Request  
    - Role: Sends article data to Supabase Content Library database via REST API.  
    - Configuration: HTTP POST with authentication credentials for Supabase, payload includes article and metadata.  
    - Inputs: True branch of "Is Relevant?".  
    - Outputs: None (terminal).  
    - Edge Cases: API auth failures, network issues, data schema mismatches; continueOnFail used.

  - **Store in Knowledge Base**  
    - Type: HTTP Request  
    - Role: Sends processed data to Supabase Knowledge Base database, possibly for deeper analytics or AI training use.  
    - Configuration: Similar to Content Library node with tailored endpoint and payload.  
    - Inputs: True branch of "Is Relevant?".  
    - Outputs: None (terminal).  
    - Edge Cases: Same as Content Library node.

  - **Error Handler**  
    - Type: Function  
    - Role: Handles articles deemed not relevant or any errors from previous steps, potentially logging or discarding them.  
    - Configuration: Custom JavaScript to log or manage errors.  
    - Inputs: False branch of "Is Relevant?".  
    - Outputs: None (terminal).  
    - Edge Cases: Logic errors here could lead to lost error info.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                      | Input Node(s)       | Output Node(s)                     | Sticky Note |
|-------------------------|---------------------|-----------------------------------------------------|---------------------|----------------------------------|-------------|
| Sticky Note - Overview  | Sticky Note         | Visual note for overview                             |                     |                                  |             |
| Schedule Every 6 Hours  | Schedule Trigger    | Periodic trigger every 6 hours                       |                     | Get News Sources                 |             |
| Get News Sources        | Function            | Provides list of news sources                        | Schedule Every 6 Hours | Is RSS Feed?                   |             |
| Is RSS Feed?            | If                  | Determines if source is RSS feed                     | Get News Sources      | Fetch RSS Feed, Fetch Google News RSS, Scrape Web Page |             |
| Fetch RSS Feed          | RSS Feed Read       | Reads articles from RSS feed                         | Is RSS Feed? (true)   | Parse Articles                  |             |
| Fetch Google News RSS   | RSS Feed Read       | Reads Google News RSS feed                           | Is RSS Feed? (false)  | Parse Articles                  |             |
| Scrape Web Page         | HTTP Request        | Scrapes non-RSS news websites                        | Is RSS Feed? (false)  | Parse Articles                  |             |
| Parse Articles          | Function            | Parses article metadata                              | Fetch RSS Feed, Fetch Google News RSS, Scrape Web Page | Fetch Full Article          |             |
| Fetch Full Article      | HTTP Request        | Retrieves full article content                       | Parse Articles        | Extract Content & Keywords, Rate Limit Wait |             |
| Rate Limit Wait         | Wait                | Enforces rate limiting delay                         | Fetch Full Article    |                                  |             |
| Extract Content & Keywords | Code             | Extracts content and keywords from article          | Fetch Full Article    | Is Relevant?                   |             |
| Is Relevant?            | If                  | Filters articles based on relevance                  | Extract Content & Keywords | Store in Content Library, Store in Knowledge Base, Error Handler |             |
| Store in Content Library | HTTP Request       | Stores article in Supabase Content Library          | Is Relevant? (true)   |                                  |             |
| Store in Knowledge Base | HTTP Request        | Stores article in Supabase Knowledge Base           | Is Relevant? (true)   |                                  |             |
| Error Handler           | Function            | Handles non-relevant articles or errors             | Is Relevant? (false)  |                                  |             |
| Sticky Note - Setup     | Sticky Note         | Visual note for setup instructions                   |                     |                                  |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add **Schedule Trigger** node named "Schedule Every 6 Hours".  
   - Set trigger interval to every 6 hours.

2. **Add Source Retrieval Node**  
   - Add **Function** node named "Get News Sources".  
   - Configure JavaScript function to return an array of news source objects, each with URL and a property to distinguish RSS feeds.

3. **Add Conditional Node to Detect RSS Sources**  
   - Add **If** node named "Is RSS Feed?".  
   - Configure condition to check if source is an RSS feed (e.g., `{{$json["isRSS"] === true}}`).

4. **Add RSS Feed Reading Nodes**  
   - Add **RSS Feed Read** node named "Fetch RSS Feed".  
   - Set URL parameter to source URL from input.  
   - Connect from "Is RSS Feed?" true output.  
   - Add **RSS Feed Read** node named "Fetch Google News RSS".  
   - Set URL parameter to the appropriate Google News RSS feed URL derived from source.  
   - Connect from "Is RSS Feed?" false output.

5. **Add Web Scraping Node**  
   - Add **HTTP Request** node named "Scrape Web Page".  
   - Configure as GET request with source URL.  
   - Connect also from "Is RSS Feed?" false output, parallel to "Fetch Google News RSS".

6. **Add Article Parsing Node**  
   - Add **Function** node named "Parse Articles".  
   - Write JavaScript to parse article metadata from RSS or scraped content.  
   - Connect outputs of "Fetch RSS Feed", "Fetch Google News RSS", and "Scrape Web Page" to this node.

7. **Add Full Article Fetch Node**  
   - Add **HTTP Request** node named "Fetch Full Article".  
   - Configure GET requests to article URLs parsed previously.  
   - Connect from "Parse Articles".

8. **Add Rate Limiting Wait**  
   - Add **Wait** node named "Rate Limit Wait".  
   - Set wait time according to API or server limits (e.g., 5 seconds).  
   - Connect parallel output from "Fetch Full Article".

9. **Add Content Extraction and Keyword Analysis Node**  
   - Add **Code** node named "Extract Content & Keywords".  
   - Implement logic to extract main content and keywords from article text.  
   - Connect from "Fetch Full Article".

10. **Add Relevance Filtering Node**  
    - Add **If** node named "Is Relevant?".  
    - Define condition based on keyword presence or relevance score.  
    - Connect from "Extract Content & Keywords".

11. **Add Supabase Storage Nodes**  
    - Add **HTTP Request** node named "Store in Content Library".  
    - Configure with POST method, set URL to Supabase Content Library API endpoint.  
    - Set authentication credentials (Supabase API key).  
    - Connect from "Is Relevant?" true output.

    - Add **HTTP Request** node named "Store in Knowledge Base".  
    - Configure similarly to above but target Knowledge Base API endpoint.  
    - Connect from "Is Relevant?" true output, after "Store in Content Library" or parallel.

12. **Add Error Handling Node**  
    - Add **Function** node named "Error Handler".  
    - Implement logging or alternative handling for irrelevant or failed articles.  
    - Connect from "Is Relevant?" false output.

13. **Add Sticky Notes for Documentation**  
    - Add **Sticky Note** nodes for Overview and Setup instructions at appropriate positions.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is designed for insurance news aggregation, leveraging Supabase for data storage. | Workflow purpose                                 |
| Ensure Supabase API keys and endpoints are securely stored in n8n credentials.                   | Credential configuration note                    |
| Rate limiting is crucial to avoid API bans or server overload, adjust wait time as needed.      | Rate Limit Wait node explanation                  |
| ContinueOnFail is enabled on most HTTP and feed nodes to ensure workflow robustness.             | Workflow resilience strategy                      |
| For more on n8n RSS feed reading: https://docs.n8n.io/nodes/n8n-nodes-base.rssFeedRead/         | Official n8n RSS Feed Read node documentation     |
| Supabase REST API docs: https://supabase.com/docs/reference/rest-api                            | Supabase integration details                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.