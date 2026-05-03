SEO On-site Audit

https://n8nworkflows.xyz/workflows/seo-on-site-audit-5106


# SEO On-site Audit

### 1. Workflow Overview

**Purpose:**  
This workflow performs a comprehensive SEO on-site audit for a given webpage URL and keyword. It collects page content, analyzes SEO-relevant data points like titles, descriptions, headings, images, links, keyword usage, technical SEO elements, and page performance metrics. Finally, it generates a detailed HTML SEO audit report and sends it via email.

**Target Use Cases:**  
- SEO professionals who want automated, in-depth on-page SEO audits.  
- Website owners and marketers seeking quick SEO insights and recommendations.  
- Agencies delivering SEO audit reports with actionable advice.  
- Automated integration into larger SEO monitoring or client reporting systems.

**Logical Blocks:**  
- **1.1 Input Reception and Data Gathering:** Receives URL and keyword input, fetches page content and metadata.  
- **1.2 Content Extraction and Basic Analysis:** Extracts HTML elements, analyzes keywords, titles, descriptions, images, and links.  
- **1.3 Technical SEO and Performance Checks:** Checks robots.txt, sitemap.xml, HTTPS, mobile-friendliness, Google Analytics presence, and Google PageSpeed Insights API data.  
- **1.4 AI-Powered SEO Recommendations:** Uses AI models to generate targeted SEO improvement tips based on extracted data.  
- **1.5 Data Aggregation and Report Generation:** Merges all analysis results, formats an HTML report, and sends it by email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Gathering

**Overview:**  
This block receives a webhook request containing the target URL and keyword, then fetches the webpage content and determines the root domain.

**Nodes Involved:**  
- Webhook  
- HTTP Request - Get Page  
- Domain  
- Robots.txt  
- Sitemap  
- Robots Analysis  
- Code  

**Node Details:**  
- **Webhook**  
  - Type: Webhook (HTTP trigger)  
  - Role: Entry point; accepts HTTP requests with URL and keyword query parameters.  
  - Config: Path set uniquely to accept requests.  
  - Outputs to: HTTP Request - Get Page, Domain, PageSpeed API.  
  - Edge cases: Invalid or missing parameters may cause downstream errors.

- **HTTP Request - Get Page**  
  - Type: HTTP Request  
  - Role: Fetches raw HTML content of the target URL from webhook input.  
  - Config: URL set dynamically from webhook query parameter. Response format: string (raw HTML).  
  - Outputs to: HTML Extract, Code Analysis.  
  - Potential failures: Network errors, 404/500 response, timeout.

- **Domain**  
  - Type: Code (JavaScript)  
  - Role: Extracts the root domain from the given URL for use in robots.txt and sitemap requests.  
  - Logic: Handles various URL formats, removing subdomains.  
  - Outputs to: Robots.txt, Sitemap.  
  - Failure modes: Malformed URLs could yield null or incorrect root domains.

- **Robots.txt**  
  - Type: HTTP Request  
  - Role: Downloads the robots.txt file from the root domain.  
  - Output to: Robots Analysis.  
  - Failures: 404 if robots.txt missing, network issues.

- **Robots Analysis**  
  - Type: Code  
  - Role: Parses robots.txt content to check presence of sitemap directive.  
  - Output: Boolean flag "hasSitemapInRobots".  
  - Edge cases: Empty or malformed robots.txt will return "No".

- **Sitemap**  
  - Type: HTTP Request  
  - Role: Downloads sitemap.xml from root domain.  
  - Output: Code node to validate XML format.  
  - Failures: 404 or invalid sitemap.xml content.

- **Code**  
  - Type: Code  
  - Role: Validates sitemap.xml content by checking for XML declaration.  
  - Output: Flag "hasValidSitemapXml".  
  - Edge cases: Non-XML or empty response returns "No".

---

#### 2.2 Content Extraction and Basic Analysis

**Overview:**  
Extracts key SEO elements from the HTML content and performs preliminary analyses on titles, descriptions, images, and links.

**Nodes Involved:**  
- HTML Extract  
- Keyword Density  
- Links  
- Check Image  
- Title  
- Description  

**Node Details:**  
- **HTML Extract**  
  - Type: HTML Extractor  
  - Role: Parses fetched HTML and extracts: title, h1, h2, meta description, links (href), body text, breadcrumbs, schema JSON, image src and alt attributes.  
  - Config: Multiple CSS selectors with attributes and arrays.  
  - Outputs: Extracted data passed to Keyword Density, Links, Check Image, Title, Description, Content Analysis nodes.  
  - Edge cases: Missing elements return empty or null arrays.

- **Keyword Density**  
  - Type: Code  
  - Role: Calculates keyword count, total words, and keyword density percentage in page text.  
  - Logic: Strips HTML tags, normalizes text, counts keyword occurrences.  
  - Output: Keyword statistics including isOptimal flag.  
  - Failures: Missing keyword or empty page content throws error.

- **Links**  
  - Type: Code  
  - Role: Processes all extracted links to categorize as internal, external, or invalid based on hostname matching root domain.  
  - Logic: Robust URL parsing with fallback, filters empty or malformed links.  
  - Output: Arrays of internalLinks, externalLinks, invalidLinks with counts.  
  - Edge cases: Malformed hrefs lead to invalidLinks classification.

- **Check Image**  
  - Type: Code  
  - Role: Combines image src and alt arrays into objects, counts images missing alt attributes.  
  - Output: List of images with alt text info, total count, missing alt count.

- **Title**  
  - Type: Code  
  - Role: Calculates title length from extracted title string.  
  - Output: titleLength number.

- **Description**  
  - Type: Code  
  - Role: Checks existence and length of meta description. Returns zero length and flag if missing.  
  - Output: descriptionLength, descriptionAvailable boolean.

---

#### 2.3 Technical SEO and Performance Checks

**Overview:**  
Examines technical SEO factors via code analysis and external API for performance metrics.

**Nodes Involved:**  
- Code Analysis  
- PageSpeed API  

**Node Details:**  
- **Code Analysis**  
  - Type: Code  
  - Role: Analyzes raw HTML to detect presence of robots.txt, sitemap.xml, HTTPS, viewport meta tag (mobile friendly), friendly URLs (absence of query params), noindex meta tag, Google Analytics, and social media links.  
  - Logic: Uses string includes and HTTP requests to verify files existence.  
  - Output: Boolean flags and social media presence indicators.  
  - Edge cases: Partial HTML or missing tags may cause false negatives.

- **PageSpeed API**  
  - Type: HTTP Request  
  - Role: Calls Google PageSpeed Insights API for mobile strategy on target URL.  
  - Config: URL dynamically built with API key (parameterized as {{APIKEY}}).  
  - Output: Performance audit JSON with scores and metrics.  
  - Failures: API quota exceeded, invalid key, network errors.

---

#### 2.4 AI-Powered SEO Recommendations

**Overview:**  
Uses AI language models to generate targeted SEO improvement suggestions based on extracted content and metrics.

**Nodes Involved:**  
- DeepSeek Chat Model (AI language model orchestrator)  
- Title Analysis  
- Description Analysis  
- Density Analysis  
- Alts Analysis  
- Content Analysis  

**Node Details:**  
- **DeepSeek Chat Model**  
  - Type: AI Chat Model (DeepSeek) node  
  - Role: Orchestrates calls to multiple downstream AI prompt nodes, passing content for analysis.  
  - Credential: Uses DeepSeek API credentials.  
  - Output: Feeds AI-generated recommendation nodes.

- **Title Analysis**  
  - Type: ChainLlm (AI prompt)  
  - Role: Provides 1 precise SEO recommendation for page title optimization relative to target keyword.  
  - Prompt includes title text, keyword, title length.  
  - Output: JSON with field "titlerecommendation".

- **Description Analysis**  
  - Type: ChainLlm  
  - Role: Generates 2 SEO recommendations for meta description improvement.  
  - Input: Description text, keyword, description length.  
  - Output: JSON with fields "descRecommendation1" and "descRecommendation2".

- **Density Analysis**  
  - Type: ChainLlm  
  - Role: Provides 2 keyword usage recommendations based on keyword count, density, and total words.  
  - Output: JSON with "keywordRecommendations" array (2 tips).

- **Alts Analysis**  
  - Type: ChainLlm  
  - Role: Analyzes image ALT attributes and gives up to 2 SEO recommendations to improve accessibility and keyword relevance.  
  - Output: JSON with "altRecommendations" array.

- **Content Analysis**  
  - Type: ChainLlm  
  - Role: Analyzes full page text content for keyword integration and relevance; produces 2 SEO tips.  
  - Output: JSON with "contentRecommendations" array.

---

#### 2.5 Data Aggregation and Report Generation

**Overview:**  
Merges data from all analyses, compiles a detailed HTML report, and sends it by email.

**Nodes Involved:**  
- Merge  
- FUnctions to report (Code)  
- Generate HTML REPORT  
- Send Email  

**Node Details:**  
- **Merge**  
  - Type: Merge (multiple inputs)  
  - Role: Aggregates outputs from all prior nodes (title, description, links, images, AI recommendations, technical SEO, PageSpeed, etc.) into a single JSON object.  
  - Config: Number of inputs set to 10.  
  - Output: Combined data object.

- **FUnctions to report**  
  - Type: Code  
  - Role: Prepares structured page analysis object with all relevant fields, generates HTML table fragments (headings, links, images, social media, technical SEO), and formats performance metrics. Contains various helper functions for safe extraction and HTML generation.  
  - Output: Fully structured JSON report data for HTML generation.

- **Generate HTML REPORT**  
  - Type: HTML node  
  - Role: Takes JSON report data and renders a comprehensive, styled HTML document summarizing the SEO audit. Includes sections for performance, titles, descriptions, headings, images, links, keywords, content, technical SEO, and social media.  
  - Output: HTML string.

- **Send Email**  
  - Type: Email Send  
  - Role: Sends the generated HTML report via SMTP email.  
  - Config: Subject “SEO Audit Report”, recipient and sender emails configured externally in credentials.  
  - Credential: SMTP account configured.  
  - Edge cases: Email sending failure due to SMTP issues.

---

### 3. Summary Table

| Node Name           | Node Type                   | Functional Role                          | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                  |
|---------------------|-----------------------------|----------------------------------------|--------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Webhook             | Webhook                     | Entry point, receives URL and keyword  | -                              | HTTP Request - Get Page, Domain, PageSpeed API |                                                                                              |
| HTTP Request - Get Page | HTTP Request              | Fetches webpage HTML content            | Webhook                        | HTML Extract, Code Analysis    |                                                                                              |
| Domain              | Code                        | Extracts root domain from URL           | Webhook                        | Robots.txt, Sitemap            |                                                                                              |
| Robots.txt           | HTTP Request                | Downloads robots.txt                    | Domain                        | Robots Analysis               |                                                                                              |
| Robots Analysis      | Code                        | Checks sitemap directive in robots.txt | Robots.txt                    | Merge                        |                                                                                              |
| Sitemap              | HTTP Request                | Downloads sitemap.xml                   | Domain                        | Code                         |                                                                                              |
| Code                 | Code                        | Validates sitemap.xml format            | Sitemap                       | Merge                        |                                                                                              |
| HTML Extract         | HTML Extract                | Extracts SEO elements from HTML         | HTTP Request - Get Page        | Keyword Density, Links, Check Image, Title, Description, Content Analysis |                                                                                              |
| Keyword Density      | Code                        | Calculates keyword usage stats          | HTML Extract                  | Density Analysis             |                                                                                              |
| Links                | Code                        | Categorizes links as internal/external | HTML Extract                  | Merge                        |                                                                                              |
| Check Image          | Code                        | Prepares image src and alt info         | HTML Extract                  | Alts Analysis                |                                                                                              |
| Title                | Code                        | Calculates title length                  | HTML Extract                  | Title Analysis               |                                                                                              |
| Description          | Code                        | Calculates description length and presence | HTML Extract              | Description Analysis         |                                                                                              |
| Content Analysis     | ChainLlm (AI)               | Provides SEO tips on content keyword use | HTML Extract                  | Merge                        |                                                                                              |
| Title Analysis       | ChainLlm (AI)               | Provides SEO tip for title optimization | Title                        | Merge                        |                                                                                              |
| Description Analysis | ChainLlm (AI)               | Provides SEO tips for meta description  | Description                  | Merge                        |                                                                                              |
| Density Analysis     | ChainLlm (AI)               | Provides keyword usage SEO tips         | Keyword Density              | Merge                        |                                                                                              |
| Alts Analysis        | ChainLlm (AI)               | Provides SEO tips for image ALT texts   | Check Image                  | Merge                        |                                                                                              |
| DeepSeek Chat Model  | AI Chat Model               | Orchestrates AI prompt calls             | -                            | Title Analysis, Description Analysis, Density Analysis, Alts Analysis, Content Analysis |                                                                                              |
| Code Analysis        | Code                        | Checks technical SEO elements in HTML   | HTTP Request - Get Page        | Merge                        |                                                                                              |
| PageSpeed API        | HTTP Request                | Fetches Lighthouse performance metrics  | Webhook                      | Merge                        |                                                                                              |
| Merge                | Merge                       | Aggregates all analysis outputs          | Multiple upstream nodes       | FUnctions to report           |                                                                                              |
| FUnctions to report  | Code                        | Builds structured report object and HTML fragments | Merge                  | Generate HTML REPORT          |                                                                                              |
| Generate HTML REPORT | HTML                        | Renders final SEO audit HTML report      | FUnctions to report           | Send Email                   |                                                                                              |
| Send Email           | Email Send                  | Sends SEO audit report via email         | Generate HTML REPORT          | -                           |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set path (e.g., unique string) to accept incoming HTTP requests with query parameters `url` and `keyword`.

2. **Create HTTP Request - Get Page**  
   - Type: HTTP Request  
   - Configure URL: `={{ $json.query.url }}` (dynamic from webhook)  
   - Response Format: String (raw HTML)  
   - Connect Webhook → HTTP Request - Get Page.

3. **Create Domain Node (Code)**  
   - Type: Code (JavaScript)  
   - Paste logic to extract root domain from input URL (handle subdomains and localhost).  
   - Input from Webhook.  
   - Output to Robots.txt and Sitemap nodes.

4. **Create Robots.txt Node (HTTP Request)**  
   - Type: HTTP Request  
   - URL: `={{ $json.rootDomain }}/robots.txt` (dynamic from Domain node)  
   - Connect Domain → Robots.txt.

5. **Create Robots Analysis Node (Code)**  
   - Type: Code  
   - Logic: Check presence of "Sitemap: ..." line in robots.txt content.  
   - Input: Robots.txt.  
   - Output to Merge node.

6. **Create Sitemap Node (HTTP Request)**  
   - Type: HTTP Request  
   - URL: `={{ $json.rootDomain }}/sitemap.xml`  
   - Connect Domain → Sitemap.

7. **Create Code Node for Sitemap Validation**  
   - Type: Code  
   - Logic: Check if sitemap.xml starts with `<?xml`.  
   - Input: Sitemap node.  
   - Output to Merge.

8. **Create HTML Extract Node**  
   - Type: HTML Extract  
   - Data Property Name: `data`  
   - Extraction Values: title, h1 (array), h2 (array), meta description (attribute content), links (href, array), body (full text), breadcrumbs, schemaJson, imgSrcs (src, array), imgAlts (alt, array).  
   - Input: HTTP Request - Get Page.  
   - Outputs: Keyword Density, Links, Check Image, Title, Description, Content Analysis.

9. **Create Keyword Density Node (Code)**  
   - Logic: Calculate keyword count, total words, density in cleaned text.  
   - Input: HTML Extract and Webhook (keyword).  
   - Output: Density Analysis.

10. **Create Links Node (Code)**  
    - Logic: Normalize and categorize links into internal, external, invalid based on root domain.  
    - Input: HTML Extract.  
    - Output: Merge.

11. **Create Check Image Node (Code)**  
    - Logic: Combine src and alt arrays, count missing alts.  
    - Input: HTML Extract.  
    - Output: Alts Analysis.

12. **Create Title Node (Code)**  
    - Logic: Calculate title length.  
    - Input: HTML Extract.  
    - Output: Title Analysis.

13. **Create Description Node (Code)**  
    - Logic: Check existence and length of meta description.  
    - Input: HTML Extract.  
    - Output: Description Analysis.

14. **Create Content Analysis Node (ChainLlm AI)**  
    - Prompt: Provide 2 SEO recommendations for content keyword integration.  
    - Input: Full page body text from HTML Extract, keyword from Webhook.

15. **Create Title Analysis Node (ChainLlm AI)**  
    - Prompt: Provide 1 SEO recommendation for title optimization.

16. **Create Description Analysis Node (ChainLlm AI)**  
    - Prompt: Provide 2 SEO recommendations for meta description.

17. **Create Density Analysis Node (ChainLlm AI)**  
    - Prompt: Provide 2 keyword usage recommendations.

18. **Create Alts Analysis Node (ChainLlm AI)**  
    - Prompt: Provide up to 2 SEO recommendations for image ALT texts.

19. **Create DeepSeek Chat Model Node**  
    - Credential: Configure DeepSeek API credentials.  
    - Connect to all AI prompt nodes (Title Analysis, Description Analysis, Density Analysis, Alts Analysis, Content Analysis).

20. **Create Code Analysis Node (Code)**  
    - Logic: Analyze raw HTML for robots.txt and sitemap presence, HTTPS, mobile friendliness, friendly URLs, noindex tag, Google Analytics snippets, social media links presence.  
    - Input: HTTP Request - Get Page.  
    - Output: Merge.

21. **Create PageSpeed API Node (HTTP Request)**  
    - URL: `"https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=" + $('Webhook').item.json.query.url + "&key={{APIKEY}}&strategy=mobile"`  
    - Configure Google API key in credentials.  
    - Input: Webhook.  
    - Output: Merge.

22. **Create Merge Node**  
    - Number of Inputs: 10 (connect all relevant nodes: Title Analysis, Description Analysis, Density Analysis, Alts Analysis, Content Analysis, Links, Code Analysis, Robots Analysis, Code (Sitemap check), PageSpeed API).  
    - Output: Functions to report.

23. **Create Functions to report Node (Code)**  
    - Logic: Assemble all data into a single JSON object, generate HTML fragments (tables for headings, links, images, social media, technical SEO checks), and prepare performance metrics for reporting.  
    - Input: Merge.

24. **Create Generate HTML REPORT Node (HTML)**  
    - Paste full HTML template including CSS styles and dynamic placeholders for all data fields from Functions to report.  
    - Input: Functions to report.

25. **Create Send Email Node**  
    - Configure SMTP credentials and email parameters (To, From, Subject).  
    - Input: Generate HTML REPORT node output.  
    - Sends the final SEO audit report email.

**Connections:**  
- Webhook → HTTP Request - Get Page → HTML Extract → multiple analysis nodes → Merge → Functions to report → Generate HTML REPORT → Send Email.  
- Webhook → Domain → Robots.txt → Robots Analysis → Merge.  
- Domain → Sitemap → Code (sitemap validation) → Merge.  
- Webhook → PageSpeed API → Merge.  
- DeepSeek Chat Model → AI prompt nodes → Merge.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses Google PageSpeed Insights API; ensure you have a valid API key and quota.                                     | https://developers.google.com/speed/pagespeed/insights/                                           |
| AI recommendations are powered by DeepSeek’s language model integration; requires valid DeepSeek API credentials.              | https://www.deepseek.ai/                                                                           |
| The workflow generates and sends a fully styled HTML email report summarizing all SEO audit results.                           |                                                                                                   |
| Robust link parsing handles relative and malformed URLs with fallback logic in the Links code node.                            |                                                                                                   |
| The report includes accessibility checks such as image ALT text analysis and social media link presence detection.             |                                                                                                   |
| The workflow can be triggered externally via webhook by providing `url` and `keyword` query parameters.                       |                                                                                                   |

---

_Disclaimer: The provided content derives exclusively from an automated workflow created with n8n, a workflow automation tool. This processing fully complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and publicly available._