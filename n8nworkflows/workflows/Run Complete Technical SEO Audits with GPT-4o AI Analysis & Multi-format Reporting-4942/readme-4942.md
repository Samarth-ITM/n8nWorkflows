Run Complete Technical SEO Audits with GPT-4o AI Analysis & Multi-format Reporting

https://n8nworkflows.xyz/workflows/run-complete-technical-seo-audits-with-gpt-4o-ai-analysis---multi-format-reporting-4942


# Run Complete Technical SEO Audits with GPT-4o AI Analysis & Multi-format Reporting

### 1. Workflow Overview

This workflow automates comprehensive technical SEO audits by leveraging GPT-4o AI for analysis and generating multi-format reports. It is designed to process website URLs or sitemaps, crawl and analyze pages, and output structured SEO insights enriched by AI interpretation, suitable for SEO professionals and digital marketers.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles external trigger via webhook or manual start, captures the target website URL, and sets language and user-agent parameters.
- **1.2 Sitemap Retrieval & Parsing:** Fetches the robots.txt and sitemap.xml files, extracts sitemap URLs, downloads sitemap XML content, and parses it into URLs.
- **1.3 URL Filtering & Deduplication:** Applies filters to exclude unwanted URLs, removes duplicates, and prepares a clean list for further analysis.
- **1.4 Page Content Retrieval:** Iterates over filtered URLs, fetches HTML content for each page.
- **1.5 AI Analysis & Processing:** Sends page content to OpenAI GPT-4o for SEO audit analysis, processes AI responses.
- **1.6 Report Generation & Distribution:** Converts AI output to HTML and JSON, stores or sends reports via Gmail, and responds back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the input URL either from a webhook or manual trigger, sets the language context, and prepares a rotating User-Agent string for requests.

- **Nodes Involved:**  
  - Webhook  
  - MANUAL (Manual Trigger)  
  - URL WEB (Set Node)  
  - LANGUAGE (Set Node)  
  - UA Rotativo1 (Code Node)  
  - Method detect (Code Node)  

- **Node Details:**  

  - **Webhook**  
    - *Type:* Webhook  
    - *Role:* Entry point for external HTTP requests containing URL parameter (`pag`) for the target website.  
    - *Configuration:* Listens for incoming HTTP requests; expects parameter `pag` with the target domain.  
    - *Connections:* Outputs to LANGUAGE.  
    - *Edge Cases:* Missing or malformed `pag` parameter; webhook downtime.

  - **MANUAL**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual execution for testing or manual input.  
    - *Connections:* Outputs to URL WEB.

  - **URL WEB**  
    - *Type:* Set  
    - *Role:* Stores the input URL parameter from webhook or manual trigger for downstream use.  
    - *Connections:* Outputs to LANGUAGE.

  - **LANGUAGE**  
    - *Type:* Set  
    - *Role:* Sets the language context for the audit, possibly influencing AI prompts or request headers.  
    - *Connections:* Outputs to UA Rotativo1.

  - **UA Rotativo1**  
    - *Type:* Code (JavaScript)  
    - *Role:* Generates a rotating User-Agent string to mimic different browsers/devices for HTTP requests.  
    - *Connections:* Outputs to Method detect.

  - **Method detect**  
    - *Type:* Code  
    - *Role:* Determines the appropriate method or request strategy for fetching robots.txt and sitemaps based on inputs or conditions.  
    - *Connections:* Outputs to Req robots.

  - **Edge Cases:** Invalid or missing URL input; language setting mismatch; User-Agent generation errors.

---

#### 2.2 Sitemap Retrieval & Parsing

- **Overview:**  
  Fetches the robots.txt file to locate sitemap URLs, downloads sitemap XML content, and parses XML to extract URLs for crawling.

- **Nodes Involved:**  
  - Req robots (HTTP Request)  
  - extract sitemap url (Code)  
  - Maping Sitemap (HTTP Request)  
  - XML1 (XML Parse)  
  - Sitemap Error (Stop and Error)  

- **Node Details:**  

  - **Req robots**  
    - *Type:* HTTP Request  
    - *Role:* Requests the robots.txt file from the target website.  
    - *Configuration:* Uses the rotating User-Agent.  
    - *Connections:* On success, outputs to extract sitemap url; on error, outputs to Req Error node.  
    - *Edge Cases:* 404 or inaccessible robots.txt; network timeouts.

  - **extract sitemap url**  
    - *Type:* Code  
    - *Role:* Parses robots.txt content to extract one or multiple sitemap URLs.  
    - *Connections:* Outputs to Maping Sitemap.

  - **Maping Sitemap**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the sitemap XML content from extracted sitemap URLs.  
    - *Connections:* On success, outputs to XML1; on failure, outputs to Sitemap Error node.  
    - *Edge Cases:* Sitemap URL invalid or unreachable; malformed XML content.

  - **XML1**  
    - *Type:* XML Parse  
    - *Role:* Parses sitemap XML to extract URL list nodes.  
    - *Connections:* Outputs to Split Out2 node.

  - **Sitemap Error**  
    - *Type:* Stop and Error  
    - *Role:* Stops the workflow with an error message if sitemap retrieval or parsing fails.

---

#### 2.3 URL Filtering & Deduplication

- **Overview:**  
  Processes the list of URLs from the sitemap to exclude unwanted URLs (e.g., external, irrelevant), removes duplicates, and prepares a refined URL list for crawling.

- **Nodes Involved:**  
  - Split Out2 (SplitOut)  
  - Filter URL (Filter)  
  - Filter URL Intern (Filter)  
  - Eliminar Webs Duplicadas (Remove Duplicates)  
  - Iterar Páginas (SplitInBatches)  

- **Node Details:**  

  - **Split Out2**  
    - *Type:* SplitOut  
    - *Role:* Splits the list of sitemap URLs for parallel processing.  
    - *Connections:* Outputs to Filter URL and Req Error1.

  - **Filter URL**  
    - *Type:* Filter  
    - *Role:* Applies filtering rules to exclude URLs based on criteria (e.g., exclude non-HTTP(s), external domains).  
    - *Connections:* Outputs to Filter URL Intern.

  - **Filter URL Intern**  
    - *Type:* Filter  
    - *Role:* Further filters URLs, specifically to exclude or include based on sitemap.xml content or internal URL rules.  
    - *Connections:* Outputs to Eliminar Webs Duplicadas.

  - **Eliminar Webs Duplicadas**  
    - *Type:* Remove Duplicates  
    - *Role:* Removes duplicate URLs to prevent redundant processing.  
    - *Connections:* Outputs to Iterar Páginas.

  - **Iterar Páginas**  
    - *Type:* SplitInBatches  
    - *Role:* Splits URLs into manageable batches for sequential processing.  
    - *Connections:* Outputs to Split Out8 and Edit Fields1.

  - **Edge Cases:** Duplicate URLs present; unexpected URL formats; filtering logic excluding all URLs.

---

#### 2.4 Page Content Retrieval

- **Overview:**  
  Iterates over each filtered URL batch, retrieves the HTML content of each webpage, preparing data for AI analysis.

- **Nodes Involved:**  
  - Split Out8 (SplitOut)  
  - To html (Code)  
  - Edit Fields1 (Set)  
  - Obtener Contenido Web (HTTP Request)  

- **Node Details:**  

  - **Split Out8**  
    - *Type:* SplitOut  
    - *Role:* Splits batches further for processing or parallelism.  
    - *Connections:* Outputs to To html.

  - **To html**  
    - *Type:* Code  
    - *Role:* Processes or transforms data into HTML format if needed for downstream nodes.  
    - *Connections:* Outputs to Send Results, Html to JSON, HTML format viewer.

  - **Edit Fields1**  
    - *Type:* Set  
    - *Role:* Prepares or adjusts fields in data, possibly setting headers or parameters before HTTP request.  
    - *Connections:* Outputs to Obtener Contenido Web.

  - **Obtener Contenido Web**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the actual HTML content of the target URLs using rotating User-Agent.  
    - *Connections:* Outputs to OpenAI1.  
    - *Edge Cases:* Page not found (404), server errors (500), timeouts.

---

#### 2.5 AI Analysis & Processing

- **Overview:**  
  Sends the retrieved page content to OpenAI GPT-4o for an SEO audit analysis, then processes the AI response for report generation.

- **Nodes Involved:**  
  - OpenAI1 (OpenAI)  
  - Code  

- **Node Details:**  

  - **OpenAI1**  
    - *Type:* OpenAI (LangChain integration)  
    - *Role:* Invokes GPT-4o model with SEO audit prompt, passing page content for AI analysis.  
    - *Connections:* Outputs to Code node.  
    - *Configuration:* Requires OpenAI credentials; configured with GPT-4o model; prompt templates likely embedded in parameters.  
    - *Edge Cases:* API rate limits, authentication errors, timeout or malformed responses.

  - **Code**  
    - *Type:* Code  
    - *Role:* Processes AI response, formats or extracts relevant data for reporting.  
    - *Connections:* Outputs to Iterar Páginas for next batch or final processing.

---

#### 2.6 Report Generation & Distribution

- **Overview:**  
  Converts AI analysis results into HTML and JSON formats, sends email reports, and responds to the webhook request with the final data.

- **Nodes Involved:**  
  - To html (Code)  
  - Html to JSON (Code)  
  - Split Out10 (SplitOut)  
  - Google Sheets1 (Google Sheets)  
  - Send Results (Gmail)  
  - HTML format viewer (HTML)  
  - Respond to Webhook (RespondToWebhook)  

- **Node Details:**  

  - **To html**  
    - *Type:* Code  
    - *Role:* Converts or formats AI analysis into HTML report format.  
    - *Connections:* Outputs to Send Results, Html to JSON, HTML format viewer.

  - **Html to JSON**  
    - *Type:* Code  
    - *Role:* Parses or converts HTML report to JSON structure for storage or API consumption.  
    - *Connections:* Outputs to Split Out10.

  - **Split Out10**  
    - *Type:* SplitOut  
    - *Role:* Splits JSON report data for feeding into Google Sheets or other destinations.  
    - *Connections:* Outputs to Google Sheets1.

  - **Google Sheets1**  
    - *Type:* Google Sheets  
    - *Role:* Stores SEO audit results in a Google Sheets spreadsheet for record keeping or further analysis.  
    - *Connections:* No further outputs.

  - **Send Results**  
    - *Type:* Gmail  
    - *Role:* Sends the final SEO audit report via email to configured recipients.  
    - *Connections:* No further outputs.

  - **HTML format viewer**  
    - *Type:* HTML  
    - *Role:* Provides a visual, interactive HTML report preview within n8n UI or for webhook response.  
    - *Connections:* Outputs to Respond to Webhook.

  - **Respond to Webhook**  
    - *Type:* RespondToWebhook  
    - *Role:* Returns the final report data to the webhook caller, completing the request cycle.  
    - *Connections:* Terminal node.

  - **Edge Cases:** Email sending failures; Google Sheets API quota limits or permission errors; malformed HTML or JSON report; webhook response timeout.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                          | Input Node(s)           | Output Node(s)                                | Sticky Note                                  |
|-------------------------|----------------------------|----------------------------------------|-------------------------|-----------------------------------------------|----------------------------------------------|
| Webhook                 | Webhook                    | Entry point, receives input URL        |                         | LANGUAGE                                      | Using: pag=example.com; example URL provided |
| MANUAL                  | Manual Trigger             | Manual start                          |                         | URL WEB                                       |                                              |
| URL WEB                 | Set                        | Stores input URL                       | MANUAL                  | LANGUAGE                                      |                                              |
| LANGUAGE                | Set                        | Sets language context                  | Webhook, URL WEB         | UA Rotativo1                                  |                                              |
| UA Rotativo1            | Code                       | Generates rotating User-Agent           | LANGUAGE                 | Method detect                                 |                                              |
| Method detect           | Code                       | Determines request method               | UA Rotativo1             | Req robots                                    |                                              |
| Req robots              | HTTP Request               | Fetches robots.txt                     | Method detect            | extract sitemap url, Req Error                |                                              |
| extract sitemap url     | Code                       | Extracts sitemap URLs from robots.txt  | Req robots               | Maping Sitemap                                |                                              |
| Maping Sitemap          | HTTP Request               | Downloads sitemap XML                  | extract sitemap url      | XML1, Sitemap Error                           |                                              |
| XML1                    | XML Parse                  | Parses sitemap XML to URLs             | Maping Sitemap           | Split Out2                                    |                                              |
| Split Out2              | SplitOut                   | Splits URLs list                      | XML1                     | Filter URL, Req Error1                         |                                              |
| Filter URL              | Filter                     | Filters URLs by criteria               | Split Out2               | Filter URL Intern                             |                                              |
| Filter URL Intern       | Filter                     | Further URL filtering                  | Filter URL               | Eliminar Webs Duplicadas                       | Filter sitemap.xml, use this to exclude URLs |
| Eliminar Webs Duplicadas| Remove Duplicates          | Removes duplicate URLs                 | Filter URL Intern        | Iterar Páginas                                | In case of duplicates or malformed sitemap   |
| Iterar Páginas          | SplitInBatches             | Batch iteration of URLs                | Eliminar Webs Duplicadas | Split Out8, Edit Fields1                       |                                              |
| Split Out8              | SplitOut                   | Further splitting for processing      | Iterar Páginas           | To html                                       |                                              |
| Edit Fields1            | Set                        | Prepares fields for HTTP Request      | Iterar Páginas           | Obtener Contenido Web                          |                                              |
| Obtener Contenido Web   | HTTP Request               | Fetches page HTML content              | Edit Fields1             | OpenAI1                                       |                                              |
| OpenAI1                 | OpenAI (LangChain)         | Sends content for AI SEO analysis     | Obtener Contenido Web    | Code                                          |                                              |
| Code                    | Code                       | Processes AI response                  | OpenAI1                  | Iterar Páginas                                |                                              |
| To html                 | Code                       | Converts AI output to HTML             | Split Out8               | Send Results, Html to JSON, HTML format viewer|                                              |
| Html to JSON            | Code                       | Converts HTML report to JSON           | To html                  | Split Out10                                   |                                              |
| Split Out10             | SplitOut                   | Splits JSON for storage                | Html to JSON             | Google Sheets1                                |                                              |
| Google Sheets1          | Google Sheets              | Stores audit data                      | Split Out10              |                                               |                                              |
| Send Results            | Gmail                      | Sends email report                    | To html                  |                                               |                                              |
| HTML format viewer      | HTML                       | Provides HTML report preview           | To html                  | Respond to Webhook                            |                                              |
| Respond to Webhook      | RespondToWebhook           | Returns final report to caller         | HTML format viewer       |                                               |                                              |
| Req Error               | Stop and Error             | Stops workflow on robots.txt error     | Req robots (error path)  |                                               |                                              |
| Req Error1              | Stop and Error             | Stops workflow on URL filter error     | Split Out2 (error path)  |                                               |                                              |
| Sitemap Error           | Stop and Error             | Stops workflow on sitemap retrieval error | Maping Sitemap (error path) |                                               |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**
   - Type: Webhook  
   - Configure to receive HTTP GET or POST requests with query parameter `pag` for target URL.

2. **Create Manual Trigger Node**
   - Type: Manual Trigger  
   - For manual runs/testing.

3. **Create Set Node "URL WEB"**
   - Set the URL parameter from Manual Trigger output or other input sources.

4. **Create Set Node "LANGUAGE"**
   - Define the language context for the audit (e.g., "en", "es").

5. **Create Code Node "UA Rotativo1"**
   - JavaScript code to generate a rotating User-Agent string (e.g., randomly select from a list of common User-Agent headers).

6. **Create Code Node "Method detect"**
   - Logic to determine request strategy based on input or other criteria.

7. **Create HTTP Request Node "Req robots"**
   - Request robots.txt from the base URL, use User-Agent from previous node.  
   - On success: connect to "extract sitemap url".  
   - On error: connect to Stop and Error node "Req Error".

8. **Create Code Node "extract sitemap url"**
   - Parse robots.txt response to extract sitemap URLs.

9. **Create HTTP Request Node "Maping Sitemap"**
   - Fetch sitemap XML from extracted URLs.  
   - On success: connect to "XML1".  
   - On error: connect to "Sitemap Error".

10. **Create XML Node "XML1"**
    - Parse sitemap XML to extract URL list.

11. **Create SplitOut Node "Split Out2"**
    - Split parsed URL list into individual URLs.

12. **Create Filter Nodes "Filter URL" and "Filter URL Intern"**
    - Apply filtering criteria (e.g., domain matching, exclude certain paths).

13. **Create Remove Duplicates Node "Eliminar Webs Duplicadas"**
    - Remove any duplicate URLs from the filtered list.

14. **Create SplitInBatches Node "Iterar Páginas"**
    - Batch process URLs in manageable chunks.

15. **Create SplitOut Node "Split Out8"**
    - Further split batches for per-URL processing.

16. **Create Set Node "Edit Fields1"**
    - Prepare HTTP request parameters, headers for page content fetching.

17. **Create HTTP Request Node "Obtener Contenido Web"**
    - Fetch HTML content of each URL.

18. **Create OpenAI Node "OpenAI1"**
    - Configure with OpenAI credentials, use GPT-4o model, set prompt for SEO analysis.  
    - Input: HTML page content.

19. **Create Code Node "Code"**
    - Process AI response to extract and format SEO audit data.

20. **Create Code Node "To html"**
    - Convert AI output into HTML report format.

21. **Create Code Node "Html to JSON"**
    - Convert HTML report to JSON structure.

22. **Create SplitOut Node "Split Out10"**
    - Split JSON data for storage.

23. **Create Google Sheets Node "Google Sheets1"**
    - Store audit data in a spreadsheet. Configure with Google Sheets OAuth2 credentials.

24. **Create Gmail Node "Send Results"**
    - Configure with Gmail OAuth2 credentials.  
    - Send final report via email.

25. **Create HTML Node "HTML format viewer"**
    - Render HTML report for preview.

26. **Create RespondToWebhook Node "Respond to Webhook"**
    - Return final data to webhook caller.

27. **Create Stop and Error Nodes "Req Error", "Req Error1", "Sitemap Error"**
    - To handle and stop workflow on critical errors.

28. **Connect Nodes**  
    - Follow connections as per the workflow diagram:  
      Webhook/MANUAL → URL WEB → LANGUAGE → UA Rotativo1 → Method detect → Req robots → extract sitemap url → Maping Sitemap → XML1 → Split Out2 → Filter URL → Filter URL Intern → Eliminar Webs Duplicadas → Iterar Páginas → Split Out8 → To html + Edit Fields1 → Obtener Contenido Web → OpenAI1 → Code → Iterar Páginas (loop)  
      To html → Send Results + Html to JSON → Split Out10 → Google Sheets1  
      To html → HTML format viewer → Respond to Webhook  

29. **Set default values and constraints**  
    - Batch sizes for SplitInBatches.  
    - Timeout and retry settings for HTTP Requests.  
    - Rate limits and error handling for OpenAI node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                    |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow enables multi-format SEO auditing using GPT-4o, integrating crawling, AI analysis, and reporting.| Project goal.                                                    |
| Webhook example usage: `https://yourspace.app.n8n.cloud/webhook-test/bbdf9cca-e5f4-4bae-afb1-a893ffb51b18?pag=onlineseoscan.com` | Usage note in Webhook node.                                       |
| Filter URL Intern node uses sitemap.xml filtering logic to exclude URLs.                                  | Important for correct URL selection.                             |
| "Eliminar Webs Duplicadas" node removes duplicate URLs to prevent redundant processing.                   | Critical for scalability and accuracy.                          |
| OpenAI node configured for GPT-4o integration via LangChain plugin in n8n v1.8+.                          | Requires valid OpenAI API key and LangChain node support.       |
| Gmail node requires OAuth2 credential setup for sending email reports.                                    | Credential configuration required.                              |
| Google Sheets node requires OAuth2 credentials and appropriate spreadsheet permissions.                   | Credential configuration and sheet setup required.              |

---

**Disclaimer:** The provided content derives exclusively from an automated n8n workflow respecting content policies and handling only legal and public data.