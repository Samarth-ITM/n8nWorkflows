Monitor & Extract Seed-Funded Startup Data with RSS, GPT-4.1-MINI & BrightData to Excel

https://n8nworkflows.xyz/workflows/monitor---extract-seed-funded-startup-data-with-rss--gpt-4-1-mini---brightdata-to-excel-6775


# Monitor & Extract Seed-Funded Startup Data with RSS, GPT-4.1-MINI & BrightData to Excel

### 1. Workflow Overview

This workflow automates monitoring and extracting data about seed-funded startups from news articles discovered via a Google Alerts RSS feed. It is designed for users interested in tracking recent startup funding events and compiling structured data for analysis or reporting.

The workflow logic is divided into the following main blocks:

- **1.1 Trigger & Article Discovery:**  
  Automatically detects new news articles from a specified RSS feed, extracts and cleans actual article URLs from redirect links.

- **1.2 Content Scraping & Preparation:**  
  Scrapes the full article content using BrightData to bypass paywalls or dynamic content, then converts the raw HTML content into clean markdown format optimized for AI processing.

- **1.3 Data Extraction with AI:**  
  Uses the GPT-4.1-MINI model via LangChain to analyze the markdown article content and extract structured JSON data about seed-funded startups, including company and founder details.

- **1.4 Extract Valid Startup Entries & Deduplication:**  
  Processes the AI output to recursively extract and deduplicate unique startup entries based on normalized company names.

- **1.5 Append Data to Excel Sheet:**  
  Sends the cleaned, deduplicated startup data to a Microsoft Excel online sheet via the Microsoft Graph API for storage and further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Article Discovery

- **Overview:**  
  Detects new articles via RSS feed, extracts actual URLs from redirect links to ensure correct article retrieval.

- **Nodes Involved:**  
  - RSS Feed Trigger  
  - Refactor article link  
  - Get article Page  
  - Add article link  
  - Sticky Note (Trigger & Article Discovery)

- **Node Details:**

  - **RSS Feed Trigger**  
    - Type: RSS Feed Read Trigger  
    - Role: Triggers workflow on new RSS feed items from Google Alerts every 5 minutes.  
    - Configuration: Polling interval set to 5 minutes; feed URL is a Google Alerts RSS feed for startup funding news.  
    - Input: None (trigger)  
    - Output: RSS items with redirect URLs in `link` field.  
    - Edge cases: Possible failure if RSS feed is down or malformed; network timeouts.

  - **Refactor article link**  
    - Type: Code node (JavaScript)  
    - Role: Parses redirect URLs to extract the actual article URL by decoding the `url` query parameter.  
    - Key logic: Uses regex to find `url` parameter, decodes it, replaces original redirect link.  
    - Input: RSS feed items with redirect URLs.  
    - Output: Items with cleaned article URLs.  
    - Edge cases: If URL parameter missing or malformed, it returns original link; script handles gracefully.

  - **Get article Page**  
    - Type: BrightData node  
    - Role: Scrapes the full content of the article URL using BrightData's web scraper to handle paywalls and dynamic content.  
    - Configuration:  
      - URL taken from cleaned article URL field.  
      - Zone: `web_unlocker1` (BrightData configuration).  
      - Country: US.  
      - Format: JSON.  
    - Input: Items with cleaned article URLs.  
    - Output: Scraped article content as JSON.  
    - Edge cases: Request failure or paywall blocking; configured to continue on error.

  - **Add article link**  
    - Type: Code node (JavaScript)  
    - Role: Merges the article URL from the Refactor article link node back into the scraped article content data, ensuring the article URL is preserved downstream.  
    - Input: Refactor article link output and BrightData scrape output.  
    - Output: Scraped article JSON with attached article URL.  
    - Edge cases: Index mismatch between inputs; assumes one-to-one mapping.

  - **Sticky Note (Trigger & Article Discovery)**  
    - Content: Explains that this block triggers on new articles and extracts actual URLs from redirect links for accurate scraping.

---

#### 1.2 Content Scraping & Preparation

- **Overview:**  
  Converts raw scraped HTML article content into cleaned markdown format for better processing by the AI model.

- **Nodes Involved:**  
  - Markdown  
  - Sticky Note (Content Scraping & Preparation)

- **Node Details:**

  - **Markdown**  
    - Type: Markdown node  
    - Role: Converts HTML content from scraped article into markdown format while ignoring certain tags (head, script, img) to clean the text.  
    - Configuration:  
      - Input HTML: `$json.body` (article body from BrightData scrape).  
      - Options: Ignores `<head>`, `<script>`, and `<img>` tags, uses link reference definitions.  
    - Input: Scraped article JSON with article URL.  
    - Output: Markdown formatted article content.  
    - Edge cases: If `body` is missing or malformed, output may be empty or incomplete.

  - **Sticky Note (Content Scraping & Preparation)**  
    - Content: Notes about BrightData scraping full article content including paywalls and markdown formatting for AI.

---

#### 1.3 Data Extraction with AI

- **Overview:**  
  Invokes GPT-4.1-MINI via LangChain OpenAI node to extract structured startup funding data from the markdown article.

- **Nodes Involved:**  
  - Message a model  
  - Edit Fields  
  - Sticky Note (Data Extraction with AI)

- **Node Details:**

  - **Message a model**  
    - Type: LangChain OpenAI node  
    - Role: Sends system prompt and article markdown content to GPT-4.1-MINI to extract JSON structured data for seed-funded startups.  
    - Configuration:  
      - Model ID: gpt-4.1-mini  
      - System prompt: Detailed instructions to extract companyName, companyWebsite, companyLinkedIn, fundingAmount, founderName (array), founderLinkedIn (array), articleUrl.  
      - Input messages: System role + user content including article markdown and article link.  
      - JSON output enabled to parse AI response into JSON automatically.  
    - Input: Markdown article content with article URL.  
    - Output: JSON array of startup funding details.  
    - Edge cases: AI may produce invalid JSON or incomplete data; model limitations on very long articles; network or auth failures.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Extracts the AI response content (which is nested in `choices`) and maps it to an array of startup data objects for downstream processing.  
    - Configuration: Sets a new array field from `{{$json.choices.map(choice => choice.message.content)}}`.  
    - Input: AI JSON response.  
    - Output: Array of startup JSON objects as strings (to be parsed later).  
    - Edge cases: If AI response structure changes, expression may fail.

  - **Sticky Note (Data Extraction with AI)**  
    - Content: Explains the AI extracts structured data including company and founder info from markdown articles.

---

#### 1.4 Extract Valid Startup Entries & Deduplication

- **Overview:**  
  Recursively extracts valid startup entries from AI output, removes duplicates based on normalized company names.

- **Nodes Involved:**  
  - Filter company data  
  - Sticky Note (Extract Valid Startup Entries from Nested Data)

- **Node Details:**

  - **Filter company data**  
    - Type: Code node (JavaScript)  
    - Role: Parses nested AI output JSON, recursively extracts objects with `companyName`, removes duplicates by normalizing companyName to lowercase and trimming whitespace.  
    - Key logic:  
      - Recursion into arrays and objects.  
      - Skips error objects.  
      - Uses a Set to track seen company names.  
    - Input: Array of JSON strings representing extracted startup data (from Edit Fields).  
    - Output: Deduplicated array of startup objects with valid fields.  
    - Edge cases: Malformed JSON strings; missing companyName fields; empty or null data.

  - **Sticky Note (Extract Valid Startup Entries from Nested Data)**  
    - Content: Describes the deduplication and extraction of unique startup entries.

---

#### 1.5 Append Data to Excel Sheet

- **Overview:**  
  Posts the deduplicated startup data to an Excel spreadsheet hosted on OneDrive or SharePoint using Microsoft Graph API.

- **Nodes Involved:**  
  - Add data into excel sheet  
  - Sticky Note (Append data to Excel sheet)

- **Node Details:**

  - **Add data into excel sheet**  
    - Type: HTTP Request node  
    - Role: Sends a POST request to Microsoft Graph API endpoint to insert rows into an Excel table.  
    - Configuration:  
      - URL: `https://graph.microsoft.com/v1.0/drives/{{drive-id}}/items/{{file-id}}/workbook/tables/{{sheet-id}}/rows` (parameters must be replaced by actual IDs).  
      - Method: POST  
      - Authentication: OAuth2 (Microsoft OAuth2 credentials)  
      - Body: JSON with `values` array built from startup fields including timestamp, companyName, companyWebsite, companyLinkedIn, fundingAmount, founderName (joined string), founderLinkedIn (joined string), articleUrl.  
      - Batching: Enabled with batch size 1 and 3-second interval.  
      - On error: Continue error output and retry enabled.  
    - Input: Deduplicated array of startup JSON objects.  
    - Output: HTTP response from Microsoft Graph API.  
    - Edge cases: OAuth token expiry; invalid or missing drive/file/table IDs; API rate limits; malformed data causing API rejection.

  - **Sticky Note (Append data to Excel sheet)**  
    - Content: Notes that this node posts data to Excel sheet using MS Graph API.

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                             | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                       |
|------------------------|----------------------------------|---------------------------------------------|-----------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| RSS Feed Trigger       | RSS Feed Read Trigger             | Triggers workflow on new RSS feed items     | None                  | Refactor article link  | ## Trigger & Article Discovery: Automatically triggers on new articles, extracts actual URLs   |
| Refactor article link  | Code                             | Extracts actual article URLs from redirects | RSS Feed Trigger      | Get article Page       | ## Trigger & Article Discovery: Extracts actual article URLs                                    |
| Get article Page       | BrightData scraper               | Scrapes full article content                 | Refactor article link | Add article link       | ## Trigger & Article Discovery: Scrapes article content including paywalls                      |
| Add article link       | Code                             | Adds article URL to scraped content          | Get article Page, Refactor article link | Markdown               | ## Trigger & Article Discovery: Preserves article URL for downstream processing                 |
| Markdown               | Markdown node                    | Converts HTML article content to markdown   | Add article link       | Message a model        | ## Content Scraping & Preparation: Cleans and formats article content for AI                    |
| Message a model        | LangChain OpenAI node            | Extracts structured startup data from article | Markdown              | Edit Fields            | ## Data Extraction with AI: Extracts startup funding data using GPT-4.1-MINI                    |
| Edit Fields            | Set                             | Extracts AI JSON response content array      | Message a model        | Filter company data    | ## Extract Valid Startup Entries from Nested Data: Prepares AI output for filtering             |
| Filter company data    | Code                            | Extracts and deduplicates startup entries    | Edit Fields            | Add data into excel sheet | ## Extract Valid Startup Entries from Nested Data: Deduplicates and filters unique startups     |
| Add data into excel sheet | HTTP Request                   | Posts startup data to Excel via MS Graph API | Filter company data    | None                  | ## Append data to Excel sheet: Posts data to Excel sheet with MS Graph API                      |
| Sticky Note            | Sticky Note                     | Workflow explanation and documentation       | None                  | None                  | ## Trigger & Article Discovery: Automatically triggers the workflow and extracts URLs          |
| Sticky Note1           | Sticky Note                     | Workflow explanation and documentation       | None                  | None                  | ## Content Scraping & Preparation: BrightData scraping and markdown formatting                  |
| Sticky Note2           | Sticky Note                     | Workflow explanation and documentation       | None                  | None                  | ## Data Extraction with AI: Extracts structured startup seed funding data                       |
| Sticky Note3           | Sticky Note                     | Workflow explanation and documentation       | None                  | None                  | ## Extract Valid Startup Entries from Nested Data: Deduplication and extraction                 |
| Sticky Note4           | Sticky Note                     | Workflow explanation and documentation       | None                  | None                  | ## Append data to Excel sheet: Posts data to excel sheet with MS Graph API                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger Node**  
   - Type: RSS Feed Read Trigger  
   - Parameters:  
     - Feed URL: `https://www.google.co.in/alerts/feeds/02881064610578318478/7170584238880554951`  
     - Poll every 5 minutes  
   - Position: Start of workflow

2. **Create Code Node "Refactor article link"**  
   - Paste JavaScript code to extract actual article URL from redirect (extract `url` parameter from query string, decode it, replace in JSON).  
   - Connect input from RSS Feed Trigger.

3. **Create BrightData Node "Get article Page"**  
   - Set URL parameter to `{{$json.link}}` (cleaned URL from previous node).  
   - Configure Zone: `web_unlocker1`  
   - Country: `us`  
   - Format: JSON  
   - Error handling: Continue on error.  
   - Connect input from "Refactor article link".

4. **Create Code Node "Add article link"**  
   - JavaScript code merges article URL from "Refactor article link" node back into BrightData output.  
   - Connect inputs:  
     - Main input from "Get article Page"  
     - Reference input from "Refactor article link" to access article URLs.

5. **Create Markdown Node "Markdown"**  
   - Input HTML: `{{$json.body}}` (article content from BrightData).  
   - Options: Ignore tags `head, script, img`.  
   - Use link reference definitions.  
   - Connect input from "Add article link".

6. **Create LangChain OpenAI Node "Message a model"**  
   - Model ID: `gpt-4.1-mini`  
   - System prompt: Provide detailed prompt as per workflow, instructing extraction of JSON structured startup funding data from markdown article.  
   - Input messages: include article markdown and article URL dynamically.  
   - Enable JSON output parsing.  
   - Connect input from "Markdown".

7. **Create Set Node "Edit Fields"**  
   - Assign new field with expression: `={{ $json.choices.map(choice => choice.message.content) }}` to extract AI response array.  
   - Connect input from "Message a model".

8. **Create Code Node "Filter company data"**  
   - JavaScript code to recursively parse AI JSON output, extract valid startup entries, deduplicate on normalized company names.  
   - Connect input from "Edit Fields".

9. **Create HTTP Request Node "Add data into excel sheet"**  
   - URL:  
     `https://graph.microsoft.com/v1.0/drives/{{drive-id}}/items/{{file-id}}/workbook/tables/{{sheet-id}}/rows`  
     *(Replace `{{drive-id}}`, `{{file-id}}`, and `{{sheet-id}}` with actual Microsoft Graph IDs)*  
   - Method: POST  
   - Authentication: OAuth2 with Microsoft credentials (Outlook or OneDrive OAuth2).  
   - Body JSON: Construct `values` array with startup data fields, including timestamp, companyName, companyWebsite, companyLinkedIn, fundingAmount, founders' names and LinkedIns joined by commas, articleUrl.  
   - Batching: Enable with batch size 1, interval 3 seconds.  
   - On error: Continue error output.  
   - Retry on fail: true  
   - Connect input from "Filter company data".

10. **Add Sticky Notes** at appropriate positions to document blocks:  
    - Trigger & Article Discovery  
    - Content Scraping & Preparation  
    - Data Extraction with AI  
    - Extract Valid Startup Entries from Nested Data  
    - Append Data to Excel Sheet

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Workflow uses BrightData for scraping behind paywalls and dynamic content.                    | BrightData scraper node configured with `web_unlocker1` zone and US country for article scraping.                        |
| GPT-4.1-MINI model is used via LangChain OpenAI node for specialized extraction tasks.       | OpenAI credentials and LangChain integration required; refer to https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/ |
| Microsoft Graph API integration requires valid OAuth2 credentials and correct drive/file/table IDs. | Setup Microsoft OAuth2 credentials in n8n and configure relevant drive, file, and sheet IDs for Excel integration.        |
| Google Alerts RSS feed URL used for startup funding news discovery.                           | RSS feed URL: https://www.google.co.in/alerts/feeds/02881064610578318478/7170584238880554951                                |

---

**Disclaimer:** The text provided is exclusively generated from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.