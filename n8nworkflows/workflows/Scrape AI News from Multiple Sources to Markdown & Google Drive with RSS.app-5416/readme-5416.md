Scrape AI News from Multiple Sources to Markdown & Google Drive with RSS.app

https://n8nworkflows.xyz/workflows/scrape-ai-news-from-multiple-sources-to-markdown---google-drive-with-rss-app-5416


# Scrape AI News from Multiple Sources to Markdown & Google Drive with RSS.app

### 1. Workflow Overview

This workflow titled **"The Recap AI - News Scraping Pipeline"** automates the process of fetching AI-related news articles and blog posts from multiple RSS feeds, extracting their main content via a scraping API, converting the content to markdown files, and uploading those files to a specified Google Drive folder. It is designed for use cases involving AI news aggregation, content archiving, and automated report or newsletter generation.

The workflow is logically structured into the following blocks:

- **1.1 Scheduled RSS Feed Fetching:** Periodically triggers HTTP requests to multiple AI news RSS feeds via RSS.app.
- **1.2 Items Splitting:** Splits feed responses into individual news/blog items for processing.
- **1.3 Article Content Scraping:** Uses the Firecrawl API to extract clean, markdown-formatted main content from article URLs.
- **1.4 Markdown File Creation:** Converts the scraped markdown content into files for upload.
- **1.5 File Upload to Google Drive:** Uploads the markdown files to a user-specified folder in Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled RSS Feed Fetching

- **Overview:**  
  This block uses scheduled triggers to periodically initiate HTTP requests to multiple RSS feed URLs from RSS.app that provide AI news and blog content.

- **Nodes Involved:**  
  - `google_news_trigger` (Schedule Trigger)  
  - `fetch_google_news_feed` (HTTP Request)  
  - `blog_open_ai_trigger` (Schedule Trigger)  
  - `fetch_blog_open_ai_feed` (HTTP Request)  
  - `Schedule Trigger` (Schedule Trigger)  
  - `HTTP Request` (HTTP Request)

- **Node Details:**

  - **google_news_trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every 3 hours for Google News AI feed.  
    - Configuration: Interval set to every 3 hours.  
    - Inputs: None (trigger)  
    - Outputs: Connected to `fetch_google_news_feed`  
    - Potential Failures: Scheduler not firing due to n8n downtime.

  - **fetch_google_news_feed**  
    - Type: HTTP Request  
    - Role: Fetches Google News AI RSS feed in JSON format from RSS.app.  
    - Configuration: URL set to `https://rss.app/feeds/v1.1/AkOariu1C7YyUUMv.json`.  
    - Inputs: Trigger from `google_news_trigger`  
    - Outputs: Connected to `split_google_news_items`  
    - Potential Failures: Network errors, invalid feed URL, API rate limits.

  - **blog_open_ai_trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow every 4 hours for OpenAI blog feed.  
    - Configuration: Interval set to every 4 hours.  
    - Inputs: None (trigger)  
    - Outputs: Connected to `fetch_blog_open_ai_feed`

  - **fetch_blog_open_ai_feed**  
    - Type: HTTP Request  
    - Role: Fetches OpenAI blog RSS feed JSON from RSS.app.  
    - Configuration: URL `https://rss.app/feeds/v1.1/xNVg2hbY14Z7Gpva.json`.  
    - Inputs: Trigger from `blog_open_ai_trigger`  
    - Outputs: Connected to `split_blog_open_ai_items`  
    - Potential Failures: Same as `fetch_google_news_feed`.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Triggers every 3 hours for a third RSS feed.  
    - Configuration: Interval every 3 hours.  
    - Inputs: None  
    - Outputs: Connected to `HTTP Request`

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetches an additional RSS feed JSON from RSS.app.  
    - Configuration: URL `https://rss.app/feeds/v1.1/sgHcE2ehHQMTWhrL.json`.  
    - Inputs: Trigger from above `Schedule Trigger`  
    - Outputs: Connected to `Split Out`  
    - Potential Failures: Same as previous HTTP requests.

---

#### 2.2 Items Splitting

- **Overview:**  
  This block splits the JSON feed responses into individual items (news/blog articles) to allow processing each article separately.

- **Nodes Involved:**  
  - `split_google_news_items` (SplitOut)  
  - `split_blog_open_ai_items` (SplitOut)  
  - `Split Out` (SplitOut)

- **Node Details:**

  - **split_google_news_items**  
    - Type: SplitOut  
    - Role: Splits the `items` array from the Google News feed JSON into individual items.  
    - Configuration: Field to split out is `items`.  
    - Inputs: Output of `fetch_google_news_feed`  
    - Outputs: Connected to `scrape_url`  
    - Edge Cases: Empty `items` array leads to no output.

  - **split_blog_open_ai_items**  
    - Type: SplitOut  
    - Role: Splits the OpenAI blog feed JSON `items` into single entries.  
    - Configuration: Field `items`.  
    - Inputs: Output of `fetch_blog_open_ai_feed`  
    - Outputs: Connected to `scrape_url`  
    - Edge Cases: Same as above.

  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits items from the third RSS feed JSON.  
    - Configuration: Field `items`.  
    - Inputs: Output of `HTTP Request` (third feed)  
    - Outputs: Connected to `scrape_url`  
    - Edge Cases: Same as above.

---

#### 2.3 Article Content Scraping

- **Overview:**  
  This block sends each article URL to the Firecrawl API to extract the main content, excluding irrelevant sections, returning markdown-formatted text.

- **Nodes Involved:**  
  - `scrape_url` (HTTP Request)

- **Node Details:**

  - **scrape_url**  
    - Type: HTTP Request  
    - Role: Calls Firecrawl API to scrape main content from each article URL.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.firecrawl.dev/v1/scrape`  
      - JSON body includes dynamic URL from item (`{{ $json.url }}`), requests multiple formats but focuses on markdown output.  
      - Excludes tags like `iframe`, `nav`, `header`, `footer`.  
      - Uses prompt to ensure extraction of exact main content, formatted as markdown without ads or navigation.  
      - Authentication: Generic HTTP Header Credential named "Instantly API" with header `Content-Type: application/json`.  
      - Retries: 3 attempts with 5-second wait between tries.  
      - On error: Continue workflow without stopping (error handling).  
    - Inputs: Individual article items from split nodes, each with a `url` field.  
    - Outputs: Connects to `create_markdown_file`.  
    - Potential Failures:  
      - API auth errors if credential missing or invalid.  
      - Network timeouts or API rate limits.  
      - Malformed URLs.  
      - Firecrawl API returning incomplete or no content.  
      - Expression failures if `url` field missing.

---

#### 2.4 Markdown File Creation

- **Overview:**  
  Converts the scraped markdown content string into a file format that can be uploaded to Google Drive.

- **Nodes Involved:**  
  - `create_markdown_file` (ConvertToFile)

- **Node Details:**

  - **create_markdown_file**  
    - Type: ConvertToFile  
    - Role: Converts scraped markdown text (`data.markdown`) into a `.md` file binary format.  
    - Configuration:  
      - Operation: toText  
      - Source property: `data.markdown` (extracted from API JSON response)  
      - Filename pattern: `news_story_{{ $itemIndex + 1 }}.md` (dynamic filename per item)  
    - Inputs: Output of `scrape_url`  
    - Outputs: Connected to `upload_markdown`  
    - Potential Failures:  
      - Missing or malformed markdown content  
      - Expression failures with filename generation

---

#### 2.5 File Upload to Google Drive

- **Overview:**  
  Uploads the generated markdown files to a designated folder in Google Drive using OAuth2 authentication.

- **Nodes Involved:**  
  - `upload_markdown` (Google Drive)

- **Node Details:**

  - **upload_markdown**  
    - Type: Google Drive  
    - Role: Uploads markdown files to a Google Drive folder.  
    - Configuration:  
      - Upload filename: Derived from binary file name property.  
      - Drive: "My Drive" (default user drive)  
      - Folder ID: Specified folder ID "13_W8MvFeaIdGNdkX8lSNV-zVFraoG6j6" (user-configured)  
      - OAuth2 credentials: Requires Google Drive OAuth2 account setup.  
    - Inputs: Files from `create_markdown_file`  
    - Outputs: None (end node)  
    - On error: Continue (does not stop workflow on failure)  
    - Potential Failures:  
      - OAuth token expiration or invalid credentials  
      - Insufficient permissions for folder  
      - API quota exceeded

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                           | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                      |
|------------------------|---------------------|-----------------------------------------|---------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| google_news_trigger     | Schedule Trigger    | Triggers Google News feed fetch every 3 hours | None                      | fetch_google_news_feed      |                                                                                                |
| fetch_google_news_feed  | HTTP Request       | Fetches Google News AI RSS JSON feed    | google_news_trigger       | split_google_news_items     |                                                                                                |
| split_google_news_items | SplitOut           | Splits Google News feed into individual items | fetch_google_news_feed    | scrape_url                 |                                                                                                |
| blog_open_ai_trigger    | Schedule Trigger    | Triggers OpenAI blog feed fetch every 4 hours | None                      | fetch_blog_open_ai_feed     |                                                                                                |
| fetch_blog_open_ai_feed | HTTP Request       | Fetches OpenAI blog RSS JSON feed        | blog_open_ai_trigger      | split_blog_open_ai_items    |                                                                                                |
| split_blog_open_ai_items| SplitOut           | Splits OpenAI blog feed into individual items | fetch_blog_open_ai_feed   | scrape_url                 |                                                                                                |
| Schedule Trigger        | Schedule Trigger    | Triggers third RSS feed fetch every 3 hours | None                      | HTTP Request               |                                                                                                |
| HTTP Request           | HTTP Request       | Fetches third RSS JSON feed               | Schedule Trigger          | Split Out                  |                                                                                                |
| Split Out              | SplitOut           | Splits third RSS feed into individual items | HTTP Request              | scrape_url                 |                                                                                                |
| scrape_url             | HTTP Request       | Calls Firecrawl API to scrape article content | split_google_news_items, split_blog_open_ai_items, Split Out | create_markdown_file       | ## 2. Scrape Urls Using Firecrawl API\n\nCreate and use your Generic Header Credential here in order to authenticate to the Firecrawl API. |
| create_markdown_file   | ConvertToFile      | Converts markdown text to .md file        | scrape_url                | upload_markdown            |                                                                                                |
| upload_markdown        | Google Drive       | Uploads markdown files to Google Drive folder | create_markdown_file      | None                      | ## 3. Upload Markdown File To Google Drive\n\nCreate and specify your Google Drive Credentials and then specify which folder you want to upload the markdown file to. |
| Sticky Note            | Sticky Note        | Section label for block 1                  | None                      | None                      | ## 1. Fetch News Articles from RSS.app Feeds                                                   |
| Sticky Note1           | Sticky Note        | Section label for block 2                  | None                      | None                      | ## 2. Scrape Urls Using Firecrawl API\n\nCreate and use your Generic Header Credential here in order to authenticate to the Firecrawl API. |
| Sticky Note2           | Sticky Note        | Section label for block 3                  | None                      | None                      | ## 3. Upload Markdown File To Google Drive\n\nCreate and specify your Google Drive Credentials and then specify which folder you want to upload the markdown file to. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Nodes for Each RSS Feed:**
   - Create a Schedule Trigger node named `google_news_trigger`.
     - Set interval to every 3 hours.
   - Create a Schedule Trigger node named `blog_open_ai_trigger`.
     - Set interval to every 4 hours.
   - Create a Schedule Trigger node named `Schedule Trigger` (third feed).
     - Set interval to every 3 hours.

2. **Create HTTP Request Nodes to Fetch RSS Feeds:**
   - Create `fetch_google_news_feed` node (HTTP Request).
     - URL: `https://rss.app/feeds/v1.1/AkOariu1C7YyUUMv.json`
     - Method: GET (default)
     - Connect `google_news_trigger` → `fetch_google_news_feed`.
   - Create `fetch_blog_open_ai_feed` node (HTTP Request).
     - URL: `https://rss.app/feeds/v1.1/xNVg2hbY14Z7Gpva.json`
     - Connect `blog_open_ai_trigger` → `fetch_blog_open_ai_feed`.
   - Create `HTTP Request` node (HTTP Request).
     - URL: `https://rss.app/feeds/v1.1/sgHcE2ehHQMTWhrL.json`
     - Connect `Schedule Trigger` → `HTTP Request`.

3. **Create SplitOut Nodes to Split Feed Items:**
   - Create `split_google_news_items` node (SplitOut).
     - Field to split: `items`.
     - Connect `fetch_google_news_feed` → `split_google_news_items`.
   - Create `split_blog_open_ai_items` node (SplitOut).
     - Field to split: `items`.
     - Connect `fetch_blog_open_ai_feed` → `split_blog_open_ai_items`.
   - Create `Split Out` node (SplitOut).
     - Field to split: `items`.
     - Connect `HTTP Request` → `Split Out`.

4. **Create HTTP Request Node to Scrape Article Content:**
   - Create `scrape_url` node (HTTP Request).
     - Method: POST  
     - URL: `https://api.firecrawl.dev/v1/scrape`  
     - Authentication: Generic HTTP Header authentication using a credential named (e.g.) "Instantly API"  
     - Headers: `Content-Type: application/json`  
     - Body (JSON, Raw):  
       ```json
       {
         "url": "{{ $json.url }}",
         "formats": ["json", "markdown", "rawHtml", "links"],
         "excludeTags": ["iframe", "nav", "header", "footer"],
         "onlyMainContent": true,
         "jsonOptions": {
           "prompt": "Identify the main content of the text (i.e., the article or newsletter body). Provide the exact text for that main content verbatim, without summarizing or rewriting any part of it. Exclude all non-essential elements such as banners, headers, footers, calls to action, ads, or purely navigational text. Format this output as markdown using appropriate '#' characters as heading levels. Exclude any promotional or sponsored content on your output.",
           "schema": {
             "type": "string",
             "description": "The exact verbatim main text content of the web page in markdown format."
           }
         }
       }
       ```
     - Enable retry on failure: 3 tries with 5000ms wait between.
     - On error: Continue workflow execution.
     - Connect all three SplitOut nodes (`split_google_news_items`, `split_blog_open_ai_items`, `Split Out`) to `scrape_url`.

5. **Create ConvertToFile Node for Markdown:**
   - Create `create_markdown_file` node (ConvertToFile).
     - Operation: toText  
     - Source Property: `data.markdown`  
     - Filename: `news_story_{{ $itemIndex + 1 }}.md`  
     - Connect `scrape_url` → `create_markdown_file`.
     - On error: Continue.

6. **Create Google Drive Node to Upload Files:**
   - Create `upload_markdown` node (Google Drive).
     - Drive: "My Drive" (default or user-selected drive)  
     - Folder ID: Enter your target folder ID (e.g., `13_W8MvFeaIdGNdkX8lSNV-zVFraoG6j6`).  
     - Name: `={{ $binary.data.fileName }}` (dynamically use file name)  
     - Credentials: Google Drive OAuth2 with appropriate access permissions.  
     - On error: Continue workflow.  
     - Connect `create_markdown_file` → `upload_markdown`.

7. **Create Credentials:**
   - Create Generic HTTP Header authentication credential for Firecrawl API with necessary API key.
   - Create Google Drive OAuth2 credentials with permissions to upload files to your target folder.

8. **Optional - Add Sticky Notes (for documentation clarity):**
   - Sticky Note near fetch nodes: "## 1. Fetch News Articles from RSS.app Feeds"
   - Sticky Note near scrape node: "## 2. Scrape Urls Using Firecrawl API\nCreate and use your Generic Header Credential here in order to authenticate to the Firecrawl API."
   - Sticky Note near upload node: "## 3. Upload Markdown File To Google Drive\nCreate and specify your Google Drive Credentials and then specify which folder you want to upload the markdown file to."

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The Firecrawl API is used to extract and format the main article content verbatim in markdown, excluding non-essential elements. | API Documentation: https://firecrawl.dev (assumed, verify with your Firecrawl API provider)                    |
| RSS.app provides the AI news and blog feeds in JSON format, enabling easy parsing and splitting.                      | RSS.app: https://rss.app                                                                                         |
| Google Drive OAuth2 credentials must have permission to upload files to the specified folder.                         | Google Drive API Docs: https://developers.google.com/drive/api/v3/about-sdk                                    |
| This workflow handles errors gracefully by continuing on failures in scraping and upload steps to avoid pipeline interruption. | Best practice for resilient automation workflows                                                                |

---

**Disclaimer:**  
The text provided here is strictly derived from an n8n automated workflow export. It complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.