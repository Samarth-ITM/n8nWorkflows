Generate SEO Content Outlines from SERP Analysis with OpenAI and Google Docs

https://n8nworkflows.xyz/workflows/generate-seo-content-outlines-from-serp-analysis-with-openai-and-google-docs-11809


# Generate SEO Content Outlines from SERP Analysis with OpenAI and Google Docs

### 1. Workflow Overview

This workflow automates the generation of SEO content outlines by analyzing the top-ranking Search Engine Results Pages (SERP) for a target keyword. It scrapes competitor articles’ headings and metadata, then leverages AI (OpenAI) to create a structured, optimized content plan. The final output is saved as a Google Document, while form inputs and requests are logged to Google Sheets for tracking.

Logical blocks in the workflow:

- **1.1 Input Reception:** Collects the target keyword and optional target audience via an n8n form.
- **1.2 Configuration Setup:** Defines parameters like maximum results and country code.
- **1.3 SERP Scraping:** Uses Apify’s Google Search Scraper to fetch top search results.
- **1.4 Filtering:** Removes non-article URLs such as PDFs, Amazon, and YouTube links.
- **1.5 Article Headings Scraping:** Visits competitor URLs to extract titles, descriptions, and heading tags (H1-H3).
- **1.6 Data Aggregation and AI Processing:** Aggregates scraped data and sends it to OpenAI to generate a content outline.
- **1.7 Output and Logging:** Creates a Google Doc with the AI-generated outline and logs the form responses to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures the user input keyword and optional target audience through a form trigger node.
- **Nodes Involved:** `SEO Keyword Input Form`, `Store Form Responses`
  
**Node Details:**

- **SEO Keyword Input Form**
  - Type: Form Trigger (Webhook-based form input)
  - Configuration:  
    - Form title: "SEO SERP Analyzer - 競合分析ツール"  
    - Fields:  
      - "対策キーワード" (Target Keyword) — required  
      - "ターゲット読者層（任意）" (Target Audience, optional)  
    - Button label: "分析を開始" (Start Analysis)  
    - Description: Explains that top 10 articles will be analyzed to generate an optimal content structure.
  - Inputs: External HTTP form submission
  - Outputs: JSON with user input fields (target_keyword, target audience)
  - Edge cases:  
    - Missing required keyword field — form validation handles this.  
    - Malformed input or unicode characters handled by n8n form node.

- **Store Form Responses**
  - Type: Google Sheets node (append or update)
  - Configuration:  
    - Operation: Append or update  
    - Sheet name: "Sheet1"  
    - Mapping: Auto-maps incoming data, matching on "target_keyword" column  
    - Document ID: Placeholder for Google Sheets document ID (must be configured)
  - Inputs: Output of `SEO Keyword Input Form`
  - Outputs: Confirmation of data appended/updated
  - Credentials: Requires Google Sheets OAuth2 credentials
  - Edge cases:  
    - Authentication failure  
    - Missing or incorrect Google Sheets document ID  
    - Network issues

---

#### 1.2 Configuration Setup

- **Overview:** Sets workflow parameters like the maximum number of SERP results and the country code for localized search.
- **Nodes Involved:** `Workflow Configuration`

**Node Details:**

- **Workflow Configuration**
  - Type: Set node
  - Configuration:  
    - maxResults: 10 (number) — limits SERP fetch to top 10 results  
    - countryCode: "jp" (string) — configures search localization to Japan  
  - Inputs: Output from `SEO Keyword Input Form`
  - Outputs: JSON including these parameters passed downstream
  - Edge cases:  
    - Misconfiguration of parameters causes unexpected results  
    - Country code unsupported by Apify (rare)

---

#### 1.3 SERP Scraping

- **Overview:** Uses Apify’s Google Search Scraper actor to retrieve the top search results for the input keyword.
- **Nodes Involved:** `Google Search SERP`

**Node Details:**

- **Google Search SERP**
  - Type: Apify node (calls external Apify actor)
  - Configuration:  
    - Actor ID: "apify/google-search-scraper"  
    - Input body:  
      - queries: bound dynamically to user input keyword  
      - maxItems: bound to `maxResults` (10)  
      - countryCode: bound to `countryCode` ("jp")  
      - resultsType: "search" (standard search results)
  - Inputs: Output from `Workflow Configuration`
  - Outputs: JSON array of SERP results with URLs, titles, snippets, etc.
  - Edge cases:  
    - API quota or rate limits on Apify  
    - Network timeouts  
    - No results or zero results returned for obscure keywords  
  - Version: Requires Apify integration available in n8n instance

---

#### 1.4 Filtering

- **Overview:** Filters out URLs that are unlikely to be content articles (e.g., PDFs, Amazon product pages, YouTube videos).
- **Nodes Involved:** `Filter Non-Article URLs`

**Node Details:**

- **Filter Non-Article URLs**
  - Type: Filter node
  - Configuration:  
    - Conditions: URL does NOT contain any of: ".pdf", "amazon.co.jp", "youtube.com", "youtu.be"  
  - Inputs: Output from `Google Search SERP`
  - Outputs: Filter-passed URLs forwarded; others discarded
  - Edge cases:  
    - URLs with query parameters or redirects may evade filter  
    - Legitimate articles on filtered domains (e.g., YouTube transcripts) excluded

---

#### 1.5 Article Headings Scraping

- **Overview:** Scrapes each competitor article URL to extract page metadata and headings (title, meta description, H1, H2, H3).
- **Nodes Involved:** `Scrape Article Headings`

**Node Details:**

- **Scrape Article Headings**
  - Type: Apify node (Cheerio Scraper actor)
  - Configuration:  
    - Actor ID: "apify/cheerio-scraper"  
    - Start URLs: dynamically set to the filtered competitor URLs  
    - Page function (JavaScript): extracts:  
      - title tag text  
      - meta description content  
      - first H1 text  
      - all H2 texts as array  
      - all H3 texts as array  
      - URL of the page  
  - Inputs: Output from `Filter Non-Article URLs`
  - Outputs: JSON objects with structured heading data per URL
  - Edge cases:  
    - Pages blocking scraping or JavaScript rendering issues  
    - Missing or malformed heading tags  
    - Pages with multiple H1s or none  
  - Version: Requires Apify integration

---

#### 1.6 Data Aggregation and AI Processing

- **Overview:** Aggregates all scraped competitor data into one array and sends it to OpenAI to generate an optimized content outline.
- **Nodes Involved:** `Aggregate Competitor Data`, `AI Content Structure Analysis`

**Node Details:**

- **Aggregate Competitor Data**
  - Type: Aggregate node
  - Configuration: Aggregate all incoming items into a single JSON array
  - Inputs: Multiple outputs from `Scrape Article Headings`
  - Outputs: Single aggregated JSON array of competitor headings and metadata
  - Edge cases:  
    - Large volumes of data could exceed memory limits  
    - Empty input if no articles passed filtering

- **AI Content Structure Analysis**
  - Type: OpenAI node (via n8n Langchain integration)
  - Configuration:  
    - Operation: Message (chat completion or prompt)  
    - Input: Aggregated competitor data (passed as prompt or context)  
    - Model & credentials: Configured on n8n instance (user must set OpenAI credentials)  
  - Inputs: Aggregated competitor data JSON  
  - Outputs: AI-generated content structure outline as text or JSON  
  - Edge cases:  
    - API rate limits or quota exceeded  
    - Prompt formatting errors  
    - Model response delays or failures  
  - Version: Requires OpenAI credentials configured in n8n

---

#### 1.7 Output and Logging

- **Overview:** Creates a Google Document with the AI-generated SEO content outline and logs the request details in Google Sheets.
- **Nodes Involved:** `Create Google Doc`

**Node Details:**

- **Create Google Doc**
  - Type: Google Docs node
  - Configuration:  
    - Title: dynamically set as "構成案: " + target keyword (e.g. "構成案: SEO対策")  
    - Content: populated with AI-generated outline (implicit from upstream connection)  
  - Inputs: Output from `AI Content Structure Analysis`  
  - Outputs: Confirmation with Google Doc URL or ID  
  - Credentials: Requires Google Docs OAuth2 credentials  
  - Edge cases:  
    - Authentication failure  
    - Insufficient permissions on Google Drive  
    - API quota exceeded

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                            | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                               |
|---------------------------|-------------------------|--------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------|
| SEO Keyword Input Form     | Form Trigger            | Collects user input keyword and audience  |                             | Workflow Configuration, Store Form Responses | 1.  **Input:** Takes a target keyword and target audience via a built-in **n8n Form**.                    |
| Workflow Configuration     | Set                     | Sets max results and country code          | SEO Keyword Input Form       | Google Search SERP           |                                                                                                           |
| Store Form Responses       | Google Sheets           | Logs form inputs for tracking               | SEO Keyword Input Form       |                             | 6.  **Output:** Logs the request in **Google Sheets** for tracking.                                      |
| Google Search SERP         | Apify (Google Search Scraper) | Scrapes top SERP results for keyword       | Workflow Configuration       | Filter Non-Article URLs      | 2.  **SERP Analysis:** Uses **Apify** (Google Search Scraper) to find the top 10 results for that keyword. |
| Filter Non-Article URLs    | Filter                  | Filters out non-article URLs                | Google Search SERP           | Scrape Article Headings     | 3.  **Filtering:** Removes PDFs, Amazon, YouTube URLs.                                                  |
| Scrape Article Headings    | Apify (Cheerio Scraper) | Scrapes competitor articles for headings   | Filter Non-Article URLs      | Aggregate Competitor Data    | 4.  **Deep Scraping:** Visits competitor URLs to extract titles, descriptions, and headings.             |
| Aggregate Competitor Data  | Aggregate               | Aggregates scraped competitor data          | Scrape Article Headings      | AI Content Structure Analysis | 5.  **AI Generation:** Aggregates data and sends it to **OpenAI** for generating an article outline.      |
| AI Content Structure Analysis | OpenAI (Langchain)      | Generates optimized article outline using AI | Aggregate Competitor Data    | Create Google Doc            |                                                                                                           |
| Create Google Doc          | Google Docs             | Creates a Google Doc with the content outline | AI Content Structure Analysis |                             | 6.  **Output:** Creates a new **Google Doc** with the generated outline.                                 |
| Sticky Note                | Sticky Note             | Overview and description                    |                             |                             | Overview note covering entire workflow purpose and audience                                              |
| Sticky Note1               | Sticky Note             | Input form explanation                       |                             |                             | Describes the Input block                                                                                  |
| Sticky Note2               | Sticky Note             | SERP scraping explanation                    |                             |                             | Describes the SERP scraping block                                                                          |
| Sticky Note3               | Sticky Note             | Filtering explanation                        |                             |                             | Describes filtering logic                                                                                   |
| Sticky Note4               | Sticky Note             | Deep scraping explanation                    |                             |                             | Describes scraping headings                                                                                |
| Sticky Note5               | Sticky Note             | AI generation explanation                    |                             |                             | Describes AI generation step                                                                                |
| Sticky Note6               | Sticky Note             | Output and logging explanation               |                             |                             | Describes Google Docs and Sheets output & logging                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node - "SEO Keyword Input Form"**
   - Type: Form Trigger  
   - Configure webhook with a unique webhook ID  
   - Form title: "SEO SERP Analyzer - 競合分析ツール"  
   - Fields:  
     - "対策キーワード" (Required)  
     - "ターゲット読者層（任意）" (Optional)  
   - Button label: "分析を開始"  
   - Description: "検索キーワードを入力すると、上位10記事を分析して最適な記事構成案を生成します"

2. **Create Set Node - "Workflow Configuration"**
   - Type: Set  
   - Add fields:  
     - maxResults (Number): 10  
     - countryCode (String): "jp"  
   - Connect input from "SEO Keyword Input Form"

3. **Create Apify Node - "Google Search SERP"**
   - Type: Apify  
   - Actor ID: "apify/google-search-scraper"  
   - Set custom body (JSON):  
     ```json
     {
       "queries": {{ $json.target_keyword }},
       "maxItems": {{ $('Workflow Configuration').first().json.maxResults }},
       "countryCode": {{ $('Workflow Configuration').first().json.countryCode }},
       "resultsType": "search"
     }
     ```
   - Connect input from "Workflow Configuration"

4. **Create Filter Node - "Filter Non-Article URLs"**
   - Type: Filter  
   - Conditions: URL does NOT contain: ".pdf", "amazon.co.jp", "youtube.com", "youtu.be"  
   - Connect input from "Google Search SERP"

5. **Create Apify Node - "Scrape Article Headings"**
   - Type: Apify  
   - Actor ID: "apify/cheerio-scraper"  
   - Set custom body (JSON) with:  
     - startUrls: map each incoming item's `url` field as an array of objects `{ url: <url> }`  
     - pageFunction: JavaScript function to extract title, meta description, H1, H2, H3, and URL.  
   - Connect input from "Filter Non-Article URLs"

6. **Create Aggregate Node - "Aggregate Competitor Data"**
   - Type: Aggregate  
   - Operation: Aggregate all incoming items into an array  
   - Connect input from "Scrape Article Headings"

7. **Create OpenAI Node - "AI Content Structure Analysis"**
   - Type: OpenAI (Langchain)  
   - Operation: Message (chat completion)  
   - Input: Pass aggregated competitor data as prompt for AI to generate content outline  
   - Credentials: Set up OpenAI API key in n8n credentials  
   - Connect input from "Aggregate Competitor Data"

8. **Create Google Docs Node - "Create Google Doc"**
   - Type: Google Docs  
   - Title: Expression: `'構成案: ' + $('SEO Keyword Input Form').first().json.target_keyword`  
   - Content: Use AI generated content from previous node as body  
   - Credentials: Set up Google Docs OAuth2 credentials  
   - Connect input from "AI Content Structure Analysis"

9. **Create Google Sheets Node - "Store Form Responses"**
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Document ID: Provide your Google Sheets document ID  
   - Sheet Name: "Sheet1"  
   - Mapping Mode: Auto map input data based on "target_keyword" column  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect input from "SEO Keyword Input Form"

10. **Optionally, add Sticky Notes describing each block for clarity**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow drastically reduces SEO research time by automating SERP analysis and AI content outline generation.                          | Overview sticky note in workflow                                                                                |
| Uses Apify integrations for scraping Google Search results and competitor article headings.                                                  | Apify actors: "apify/google-search-scraper", "apify/cheerio-scraper"                                            |
| Requires valid OpenAI API credentials configured in n8n for AI generation.                                                                  | OpenAI integration                                                                                              |
| Requires Google OAuth2 credentials for both Google Docs and Google Sheets nodes for output and logging.                                      | Google Docs & Sheets integrations                                                                                |
| Input form and button labels are in Japanese, reflecting the target audience and country (Japan).                                           | Localization detail                                                                                              |
| The workflow filters out common non-content URLs (PDFs, Amazon, YouTube) to focus on competitor articles.                                   | Filtering node logic                                                                                            |
| For advanced modification, consider error handling for API rate limits, network failures, and empty results.                                | Suggested improvements                                                                                           |

---

**Disclaimer:** The provided description and analysis are based solely on the n8n workflow JSON shared. All data processed is legal and public. No illegal or protected content is involved.