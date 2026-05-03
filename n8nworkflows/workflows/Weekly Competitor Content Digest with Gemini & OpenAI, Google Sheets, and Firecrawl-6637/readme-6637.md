Weekly Competitor Content Digest with Gemini & OpenAI, Google Sheets, and Firecrawl

https://n8nworkflows.xyz/workflows/weekly-competitor-content-digest-with-gemini---openai--google-sheets--and-firecrawl-6637


# Weekly Competitor Content Digest with Gemini & OpenAI, Google Sheets, and Firecrawl

### 1. Workflow Overview

This workflow, titled **"AI Content Scout - Weekly Competitor Blogs and News Digest"**, automates the weekly aggregation, analysis, and digesting of competitor blog posts and news articles. It is designed primarily to:

- Extract candidate URLs from a Google Sheets input.
- Crawl and scrape these URLs to extract links and markdown content.
- Filter and identify relevant URLs for content analysis using AI agents.
- Summarize each article’s key information (title, author, summary, publication date).
- Store structured summaries back in a Google Sheet.
- Compile a daily HTML digest grouped by company.
- Email the digest weekly.

The workflow logically breaks down into these main blocks:

- **1.1 Input Reception and Scheduling:** Fetch input URLs weekly from Google Sheets, triggered by a schedule.
- **1.2 URL Extraction and Filtering:** Use Firecrawl to scrape links from input URLs, filter and identify new and relevant URLs with AI.
- **1.3 Content Scraping and Summarization:** Scrape article markdown content, then summarize key details using AI models (Google Gemini and OpenAI).
- **1.4 Data Storage and Aggregation:** Append structured data into a Google Sheet and generate an HTML digest grouped by company.
- **1.5 Email Notification:** Send the compiled HTML digest via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

**Overview:**  
This block initiates the workflow weekly every Sunday at 5 AM and fetches a list of competitor URLs from a Google Sheet.

**Nodes Involved:**  
- Schedule Trigger: Every Sun 5AM  
- Input Links (Google Sheets)

**Node Details:**  

- **Schedule Trigger: Every Sun 5AM**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow weekly on Sunday at 5 AM using a cron expression.  
  - Configuration: Cron set to `0 0 5 * * SUN`.  
  - Inputs: None  
  - Outputs: Connects to "Input Links" node.  
  - Failure Modes: Cron misconfiguration or n8n service downtime can prevent trigger.

- **Input Links**  
  - Type: Google Sheets  
  - Role: Reads competitor URLs and metadata from a specific sheet named "Input Links" in a Google Sheets document.  
  - Configuration: Connected to a Google Sheets OAuth credential; reads a sheet with ID `288631821` in the document `1R01ELC8ZTWvyw2ULRPnMogrtqkoVV40ANa4qqVi3F-I`.  
  - Outputs: Provides URL list to "Firecrawl_Links".  
  - Failure Modes: Google Sheets API authentication errors, sheet not found, or permission issues.

---

#### 2.2 URL Extraction and Filtering

**Overview:**  
This block scrapes each input URL for links with Firecrawl, then filters and identifies relevant URLs for crawling and summarizing using an AI agent and Google Sheets for existing URLs reference.

**Nodes Involved:**  
- Firecrawl_Links (HTTP Request)  
- Agent: Identify URLs to Crawl (LangChain Agent)  
- Read_Url_Summary_Tool (Google Sheets Tool)  
- Split Out (SplitOut)  
- Structured Output Parser (LangChain Output Parser)  
- Sticky Notes (explanatory)

**Node Details:**  

- **Firecrawl_Links**  
  - Type: HTTP Request  
  - Role: Posts to Firecrawl API to scrape links from each input URL.  
  - Configuration: POST to `https://api.firecrawl.dev/v1/scrape` with payload requesting "links" format and main content only.  
  - Credentials: Uses Firecrawl HTTP header auth.  
  - Batching: Batch size 1, interval 7000ms to respect Firecrawl free tier limits.  
  - Outputs: Raw scraped links to "Agent: Identify URLs to Crawl".  
  - Failure Modes: API rate limits (free tier 10 req/min), network errors, malformed URLs.

- **Agent: Identify URLs to Crawl**  
  - Type: LangChain Agent  
  - Role: Filters candidate URLs from Firecrawl against existing URLs in Google Sheet and applies topical inclusion/exclusion rules.  
  - Configuration: Uses a system prompt defining rigorous URL filtering criteria, including normalization and exclusion of utility pages.  
  - Inputs: Links from Firecrawl_Links, existing URLs from Read_Url_Summary_Tool (Google Sheets) via ai_tool connection.  
  - Outputs: Filtered URLs to "Split Out".  
  - Failure Modes: AI model errors, parsing issues, or tool communication failures.

- **Read_Url_Summary_Tool**  
  - Type: Google Sheets Tool  
  - Role: Reads existing URLs from Google Sheet column G in "Summary" sheet to avoid duplicate processing.  
  - Credentials: Google Sheets OAuth.  
  - Outputs: Provides existing URLs as ai_tool input to Agent: Identify URLs to Crawl.  
  - Failure Modes: Authentication errors, sheet access issues.

- **Split Out**  
  - Type: SplitOut  
  - Role: Splits the filtered array of URLs into individual items for parallel processing downstream.  
  - Outputs: Single URL items to "Firecrawl_Markdown".  
  - Failure Modes: Empty input arrays or invalid JSON could cause issues.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI agent output JSON arrays into usable data structures.  
  - Configuration: Example schema of `[{"url": "https://example.com/topic-1"}]`.  
  - Inputs: Output from Agent: Identify URLs to Crawl.  
  - Outputs: Structured URL objects to Split Out node.

- **Sticky Notes**  
  - Provide context on Firecrawl API rate limits (1 request per 6 seconds), URL filtering logic, and candidate URL evaluation rules.  
  - These notes emphasize operational constraints and filtering rationale.

---

#### 2.3 Content Scraping and Summarization

**Overview:**  
This block scrapes markdown content from each filtered URL, then leverages AI agents (Google Gemini and OpenAI) to extract structured article information such as title, author, summary, and published date.

**Nodes Involved:**  
- Firecrawl_Markdown (HTTP Request)  
- Agent: Summarize Each URL (LangChain Agent)  
- Gemini 2.5 Flash: Temp 0 (Google Gemini LM)  
- OpenAI o4-mini (OpenAI LM)  
- Structured Output Parser2 (LangChain Output Parser)  
- Sticky Note1 (explanatory)

**Node Details:**  

- **Firecrawl_Markdown**  
  - Type: HTTP Request  
  - Role: Scrapes markdown content from each filtered URL.  
  - Configuration: POST to Firecrawl API with `formats:["markdown"]` and main content only.  
  - Credentials: Firecrawl HTTP header auth.  
  - Batching: Batch size 1, interval 7000ms to respect rate limits.  
  - Outputs: Markdown content and metadata to "Agent: Summarize Each URL".  
  - Failure Modes: API limits, network errors, malformed URLs.

- **Agent: Summarize Each URL**  
  - Type: LangChain Agent  
  - Role: Extracts article fields (title, author, URL, summary, published_date) from markdown content using a strict extraction prompt and outputs JSON array.  
  - Configuration: Uses Google Gemini 2.5 Flash (temperature 0) or OpenAI o4-mini as language model. The prompt enforces exact JSON output without extra text.  
  - Inputs: Markdown content and URL from Firecrawl_Markdown.  
  - Outputs: Structured article data to "Append row in sheet".  
  - Failure Modes: AI parsing failures, incomplete content, or JSON formatting errors.

- **Gemini 2.5 Flash: Temp 0**  
  - Type: LangChain Google Gemini Chat LM  
  - Role: Primary AI model for summarization with zero randomness (temperature 0) for consistent output.  
  - Credentials: Google PaLM API.  
  - Connected as ai_languageModel for "Agent: Summarize Each URL".  
  - Failure Modes: API quota, authentication errors.

- **OpenAI o4-mini**  
  - Type: LangChain OpenAI Chat LM  
  - Role: Alternative AI model used in the URL identification agent block.  
  - Credentials: OpenAI API key.  
  - Failure Modes: API quota, rate limiting.

- **Structured Output Parser2**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses JSON array output from summarization agent.  
  - Configuration: Example schema includes title, author, url, summary, published_date.  
  - Inputs: Output from "Agent: Summarize Each URL".  
  - Outputs: Parsed structured data for sheet append.

- **Sticky Note1**  
  - Clarifies the fields to be extracted from the content: title, author, url, summary (3-4 sentences), and published_date.

---

#### 2.4 Data Storage and Aggregation

**Overview:**  
This block appends structured article summaries to a Google Sheet and then aggregates data crawled today into an HTML email digest grouped by company.

**Nodes Involved:**  
- Append row in sheet (Google Sheets)  
- Get row(s) in sheet (Google Sheets)  
- Code (JavaScript code node)  
- Sticky Note4 (explanatory)

**Node Details:**  

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Appends each summarized article as a new row in the "Summary" sheet with fields such as URL, Title, Author, Company, Summary, Page Type, Crawled Date, Published Date.  
  - Configuration: Uses defined column mapping. Crawled Date is set to current ISO date.  
  - Credentials: Google Sheets OAuth.  
  - Outputs: Triggers "Get row(s) in sheet".  
  - Failure Modes: Sheet write permission errors, API rate limits.

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves all rows from "Summary" sheet to prepare for digest generation.  
  - Configuration: Reads entire "Summary" sheet.  
  - Credentials: Google Sheets OAuth.  
  - Outputs: Passes data to Code node for processing.  
  - Failure Modes: Authentication, sheet availability issues.

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Filters rows where Crawled Date equals today, groups rows by Company, and generates an HTML table digest with detailed article info.  
  - Outputs: JSON object with `html` property containing the full digest.  
  - Failure Modes: Date parsing errors, empty data sets.

- **Sticky Note4**  
  - Explains the HTML email creation rules: filter rows by today's date and group articles by company.

---

#### 2.5 Email Notification

**Overview:**  
Sends the compiled HTML digest via Gmail to predefined recipients.

**Nodes Involved:**  
- Send a message (Gmail)

**Node Details:**  

- **Send a message**  
  - Type: Gmail  
  - Role: Sends an email with the generated HTML digest as the message body.  
  - Configuration: Sends to `mwtyang@gmail.com` and `michael.w.yang@elastic.co` with subject "URL Summary". No attribution appended.  
  - Credentials: Gmail OAuth2.  
  - Inputs: HTML content from Code node.  
  - Failure Modes: Email authentication failures, quota limits, invalid email addresses.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                              | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|--------------------------------------|----------------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger: Every Sun 5AM | Schedule Trigger                     | Initiate workflow weekly                      | None                          | Input Links                 |                                                                                              |
| Input Links               | Google Sheets                        | Fetch competitor URLs from sheet              | Schedule Trigger              | Firecrawl_Links             |                                                                                              |
| Firecrawl_Links           | HTTP Request                        | Scrape links from input URLs                  | Input Links                   | Agent: Identify URLs to Crawl | Firecrawl Free Tier only allows 10 requests per min (1 request per 6s)                        |
| Agent: Identify URLs to Crawl | LangChain Agent                    | Filter and identify relevant URLs             | Firecrawl_Links, Read_Url_Summary_Tool | Split Out                  |                                                                                              |
| Read_Url_Summary_Tool     | Google Sheets Tool                  | Read existing URLs to exclude duplicates      | None                          | Agent: Identify URLs to Crawl |                                                                                              |
| Split Out                 | SplitOut                           | Split array of URLs into individual items     | Agent: Identify URLs to Crawl | Firecrawl_Markdown          |                                                                                              |
| Firecrawl_Markdown        | HTTP Request                      | Scrape markdown content from URLs             | Split Out                    | Agent: Summarize Each URL   | Firecrawl Free Tier only allows 10 requests per min (1 request per 6s)                        |
| Agent: Summarize Each URL | LangChain Agent                    | Extract article info (title, author, summary) | Firecrawl_Markdown            | Append row in sheet         | From the provided link, extract: title, author, url, summary: 3–4 sentences, published_date |
| Gemini 2.5 Flash: Temp 0  | LangChain Google Gemini LM         | Primary AI model for content summarization    | Agent: Summarize Each URL     | Agent: Summarize Each URL   |                                                                                              |
| OpenAI o4-mini            | LangChain OpenAI LM               | Alternative AI model for URL identification   | Agent: Identify URLs to Crawl | Agent: Identify URLs to Crawl |                                                                                              |
| Structured Output Parser  | LangChain Output Parser           | Parse filtered URLs JSON array                 | Agent: Identify URLs to Crawl | Agent: Identify URLs to Crawl | Filter all columns except for URL                                                            |
| Structured Output Parser2 | LangChain Output Parser           | Parse article summary JSON output              | Agent: Summarize Each URL     | Append row in sheet         |                                                                                              |
| Append row in sheet       | Google Sheets                     | Append article data to summary sheet           | Agent: Summarize Each URL     | Get row(s) in sheet         |                                                                                              |
| Get row(s) in sheet       | Google Sheets                     | Fetch all summarized rows for today's digest  | Append row in sheet           | Code                        |                                                                                              |
| Code                      | Code (JavaScript)                 | Generate HTML digest grouped by company         | Get row(s) in sheet           | Send a message              | Create HTML email with following conditions: 1) Crawled Date == today, 2) Group by Company   |
| Send a message            | Gmail                            | Send the weekly HTML email digest               | Code                         | None                       |                                                                                              |
| Sticky Note1              | Sticky Note                      | Explain summarization fields                     | None                          | None                       | From the provided link, extract the following: title, author, url, summary: 3–4 sentences, published_date |
| Sticky Note2              | Sticky Note                      | Explain Firecrawl API rate limits                | None                          | None                       | Firecrawl Free Tier only allows 10 requests per min (1 request per 6s)                       |
| Sticky Note3              | Sticky Note                      | Duplicate of Sticky Note2 related to rate limits | None                          | None                       | Firecrawl Free Tier only allows 10 requests per min (1 request per 6s)                       |
| Sticky Note4              | Sticky Note                      | Explain HTML email creation rules                 | None                          | None                       | Create HTML email with following conditions: 1) Crawled Date == today 2) Group by Company   |
| Sticky Note5              | Sticky Note                      | Explains filtering of columns except URL          | None                          | None                       | Filter all columns except for URL                                                           |
| Sticky Note               | Sticky Note                      | URL candidate filtering and evaluation notes     | None                          | None                       | Each link contains candidate URL: Compare Candidate URL to Existing URLs...                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configure cron expression: `0 0 5 * * SUN` to trigger every Sunday at 5 AM.  

2. **Add Google Sheets Node "Input Links"**  
   - Operation: Read rows  
   - Document ID and Sheet Name: Use your Google Sheet containing competitor URLs (e.g., document ID `1R01ELC8ZTWvyw2ULRPnMogrtqkoVV40ANa4qqVi3F-I`, sheet ID `288631821`).  
   - Credentials: Connect your Google Sheets OAuth2 credentials.  
   - Connect this node to the Schedule Trigger.

3. **Add HTTP Request Node "Firecrawl_Links"**  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v1/scrape`  
   - Body type: JSON  
   - JSON Body:
     ```json
     {
       "url": "{{$json.Link}}",
       "formats": ["links"],
       "onlyMainContent": true,
       "timeout": 90000
     }
     ```
   - Authentication: HTTP Header Auth with your Firecrawl API key.  
   - Enable batching: batch size 1, interval 7000 ms (to respect API limits).  
   - Connect the "Input Links" node output to this node.

4. **Add Google Sheets Tool "Read_Url_Summary_Tool"**  
   - Operation: Read a single column (column G) from your "Summary" sheet in the same Google Sheet document.  
   - Credentials: Same Google Sheets OAuth2 credentials.  
   - This node will be used as an AI tool input for URL filtering.

5. **Add LangChain Agent Node "Agent: Identify URLs to Crawl"**  
   - Use OpenAI o4-mini or preferred LM for AI processing.  
   - Set the system prompt to filter candidate URLs based on rules provided in the original prompt (exclude duplicates, utility pages, invalid URLs, etc.).  
   - Input: links from "Firecrawl_Links" and existing URLs from "Read_Url_Summary_Tool" (set as ai_tool input).  
   - Output: filtered URLs as JSON array.  

6. **Add Structured Output Parser Node**  
   - Define example JSON schema for URLs: `[{"url": "https://example.com/topic-1"}]`  
   - Connect output from Agent node to this parser.

7. **Add SplitOut Node "Split Out"**  
   - Field to split: `output` (the array of filtered URLs from previous parser).  
   - Splits each URL into separate items for downstream processing.

8. **Add HTTP Request Node "Firecrawl_Markdown"**  
   - Method: POST to `https://api.firecrawl.dev/v1/scrape`  
   - JSON Body:
     ```json
     {
       "url": "{{ $json.url }}",
       "formats": ["markdown"],
       "onlyMainContent": true,
       "timeout": 90000
     }
     ```
   - Authentication: Firecrawl HTTP header auth.  
   - Batching: batch size 1, interval 7000 ms.  
   - Connect "Split Out" node output to this node.

9. **Add LangChain Agent Node "Agent: Summarize Each URL"**  
   - Use Google Gemini 2.5 Flash (temperature 0) or OpenAI model.  
   - Prompt: Strictly extract title, author, url, summary (3-4 sentences), and published_date from markdown content.  
   - Output parser enabled with a JSON schema matching the extraction fields.  
   - Input: Markdown content and URL from "Firecrawl_Markdown".  
   - Output: JSON array with extracted fields.

10. **Add Structured Output Parser Node (Structured Output Parser2)**  
    - Define JSON schema for article details including title, author, url, summary, published_date.  
    - Connect output of Agent Summarize node.

11. **Add Google Sheets Node "Append row in sheet"**  
    - Operation: Append row to "Summary" sheet in the same Google Sheet document.  
    - Map columns: URL, Title, Author, Company (from Input Links), Summary, Page Type (from Input Links), Crawled Date (set to today’s date), Published Date.  
    - Credentials: Google Sheets OAuth2.  
    - Connect output from Structured Output Parser2.

12. **Add Google Sheets Node "Get row(s) in sheet"**  
    - Operation: Read entire "Summary" sheet to fetch all summarized articles.  
    - Credentials: Google Sheets OAuth2.  
    - Connect output of "Append row in sheet".

13. **Add Code Node**  
    - JavaScript code to:  
      - Filter rows where Crawled Date equals today  
      - Group rows by Company  
      - Generate an HTML table digest with article info  
    - Output: JSON with `html` property containing the digest.  
    - Connect output of "Get row(s) in sheet".

14. **Add Gmail Node "Send a message"**  
    - Send email with:  
      - To: `mwtyang@gmail.com, michael.w.yang@elastic.co`  
      - Subject: "URL Summary"  
      - Message: `={{ $json.html }}` (HTML digest)  
    - Credentials: Gmail OAuth2.  
    - Connect output of Code node.

15. **Add Sticky Notes** at relevant points to document:  
    - Firecrawl API rate limits (one request every 6 seconds, max 10/min).  
    - URL filtering logic and criteria.  
    - Extraction fields to summarize.  
    - HTML email creation conditions.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Firecrawl Free Tier only allows 10 requests per min (1 request per 6 seconds).                       | Important API rate limit for scraping nodes.                                                    |
| Each link contains candidate URL: Compare Candidate URL to Existing URLs in Read_Url_Summary_Tool; Filter out matches; Identify proper URLs (no top-level pages, navigation, bad URLs). | URL filtering rationale and operational instructions.                                          |
| From the provided link, extract the following: title, author, url, summary (3–4 sentences), published_date. | Summarization extraction rules for AI agent.                                                   |
| Create HTML email with following conditions: 1) Crawled Date == today, 2) Group by Company.          | Instructions for the code node generating the email digest.                                    |
| Filter all columns except for URL                                                                     | Relevant to Structured Output Parser filtering.                                                 |
| Google Sheet document used: [Google Sheet - URL Summary & Input Links](https://docs.google.com/spreadsheets/d/1R01ELC8ZTWvyw2ULRPnMogrtqkoVV40ANa4qqVi3F-I) | Source document for URLs and summaries.                                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.