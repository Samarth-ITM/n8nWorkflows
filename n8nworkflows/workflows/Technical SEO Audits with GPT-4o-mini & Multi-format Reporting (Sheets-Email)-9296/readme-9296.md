Technical SEO Audits with GPT-4o-mini & Multi-format Reporting (Sheets/Email)

https://n8nworkflows.xyz/workflows/technical-seo-audits-with-gpt-4o-mini---multi-format-reporting--sheets-email--9296


# Technical SEO Audits with GPT-4o-mini & Multi-format Reporting (Sheets/Email)

### 1. Workflow Overview

This n8n workflow automates **Technical SEO Audits** by leveraging GPT-4o-mini for AI analysis and generates multi-format reports delivered via Google Sheets and email. It is designed to intake a website URL, extract and process its sitemap URLs, fetch page content, perform AI-driven SEO analysis, and output structured reports for SEO professionals or clients.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts input either via webhook or manual trigger, setting the target website URL.
- **1.2 Language and User-Agent Setup:** Determines language context and rotates User-Agent strings for HTTP requests.
- **1.3 Sitemap Retrieval and Parsing:** Requests `robots.txt` to extract sitemap URLs, fetches and parses the sitemap XML, handling errors gracefully.
- **1.4 URL Filtering and Deduplication:** Filters URLs internally to exclude unwanted links, removes duplicates, and prepares the final list for processing.
- **1.5 Page Content Retrieval:** Iterates over filtered URLs, fetching HTML content for each page.
- **1.6 AI Analysis with OpenAI GPT-4o-mini:** Sends page content to the OpenAI node for SEO audit analysis via GPT.
- **1.7 Reporting and Output:** Formats results into HTML, Google Sheets, and emails reports back to the requester.
- **1.8 Error Handling:** Includes nodes to stop the workflow and report errors on request or sitemap issues.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block receives the target website URL either via a webhook URL or manually triggered input. It initializes the workflow by setting the URL for subsequent processing.

**Nodes Involved:**  
- Webhook  
- MANUAL  
- URL WEB

**Node Details:**  
- **Webhook**  
  - Type: Webhook (HTTP endpoint)  
  - Role: Receives HTTP GET/POST requests with parameter `pag` containing the target domain (e.g., `onlineseoscan.com`).  
  - Config: Webhook ID fixed; no authentication.  
  - Inputs: External HTTP request.  
  - Outputs: Passes URL parameter to next node.  
  - Edge cases: Missing or malformed `pag` parameter; network latency.  

- **MANUAL**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for testing or batch processing.  
  - Inputs: User-initiated.  
  - Outputs: Passes control to URL WEB node.  

- **URL WEB**  
  - Type: Set node  
  - Role: Stores/sets the target website URL variable extracted from webhook or manual input.  
  - Config: Extracts `pag` query parameter.  
  - Inputs: From webhook or manual.  
  - Outputs: Feeds LANGUAGE node.  

---

#### 1.2 Language and User-Agent Setup

**Overview:**  
Sets the language context and rotates User-Agent headers to mimic different browsers for HTTP requests, reducing blocking risk.

**Nodes Involved:**  
- LANGUAGE  
- UA Rotativo1  
- Method detect

**Node Details:**  
- **LANGUAGE**  
  - Type: Set  
  - Role: Define or detect language parameters for the audit (e.g., "en", "es").  
  - Inputs: From URL WEB.  
  - Outputs: To UA Rotativo1.  
  - Edge cases: Language detection failure or unsupported language.  

- **UA Rotativo1**  
  - Type: Code (JavaScript)  
  - Role: Selects a rotating User-Agent string from a predefined list for HTTP requests.  
  - Inputs: LANGUAGE node output.  
  - Outputs: To Method detect.  
  - Edge cases: Empty User-Agent list or invalid selection logic.  

- **Method detect**  
  - Type: Code  
  - Role: Detects request method or constructs HTTP request parameters based on input.  
  - Inputs: UA Rotativo1 output.  
  - Outputs: To Req robots node.  
  - Edge cases: Logic errors in method detection or incorrect headers.  

---

#### 1.3 Sitemap Retrieval and Parsing

**Overview:**  
Fetches the website's `robots.txt`, extracts sitemap URLs, requests the sitemap XML, and parses it to retrieve URLs for audit.

**Nodes Involved:**  
- Req robots  
- extract sitemap url  
- Maping Sitemap  
- XML1  
- Sitemap Error

**Node Details:**  
- **Req robots**  
  - Type: HTTP Request  
  - Role: Downloads `robots.txt` from the target site.  
  - Config: On error, continues to output error node.  
  - Inputs: From Method detect.  
  - Outputs: To extract sitemap url and Req Error.  
  - Edge cases: Network errors, 404 not found, robots.txt malformed.  

- **extract sitemap url**  
  - Type: Code  
  - Role: Parses `robots.txt` content to extract all sitemap URLs.  
  - Inputs: Req robots output.  
  - Outputs: Maping Sitemap.  
  - Edge cases: No sitemap URLs found, malformed robots.txt.  

- **Maping Sitemap**  
  - Type: HTTP Request  
  - Role: Retrieves sitemap XML file from extracted URL.  
  - Config: On error, continues to Sitemap Error node.  
  - Inputs: extract sitemap url output.  
  - Outputs: XML1 or Sitemap Error.  
  - Edge cases: Sitemap not accessible, timeout, invalid XML.  

- **XML1**  
  - Type: XML node  
  - Role: Parses sitemap XML to extract URLs.  
  - Inputs: Maping Sitemap output.  
  - Outputs: Split Out2.  
  - Edge cases: XML parsing errors, empty sitemap.  

- **Sitemap Error**  
  - Type: Stop and Error  
  - Role: Stops the workflow with an error message when sitemap retrieval fails.  
  - Inputs: Maping Sitemap error output.  

---

#### 1.4 URL Filtering and Deduplication

**Overview:**  
Filters URLs to exclude internal or unwanted links and removes duplicates ensuring clean input for content fetching.

**Nodes Involved:**  
- Split Out2  
- Filter URL  
- Filter URL Intern  
- Eliminar Webs Duplicadas  
- Iterar Páginas

**Node Details:**  
- **Split Out2**  
  - Type: SplitOut  
  - Role: Splits batch of URLs for filtering or error handling.  
  - Inputs: XML1 output.  
  - Outputs: Filter URL or Req Error1.  

- **Filter URL**  
  - Type: Filter  
  - Role: Filters URLs based on criteria (e.g., exclude non-HTML files or external links).  
  - Inputs: Split Out2 output.  
  - Outputs: Filter URL Intern.  

- **Filter URL Intern**  
  - Type: Filter  
  - Role: Further filters URLs, specifically for sitemap exclusions or internal logic.  
  - Inputs: Filter URL output.  
  - Outputs: Eliminar Webs Duplicadas.  
  - Notes: Contains comment "Filter sitemap.xml, usa ese filtro para excluir urls del sitemap".  

- **Eliminar Webs Duplicadas**  
  - Type: Remove Duplicates  
  - Role: Removes duplicate URLs that may appear due to sitemap errors or overlaps.  
  - Inputs: Filter URL Intern output.  
  - Outputs: Iterar Páginas.  
  - Notes: Comment "En caso de haber duplicados o sitemap mal hecho".  

- **Iterar Páginas**  
  - Type: SplitInBatches  
  - Role: Iterates over URLs in batches for processing.  
  - Inputs: Eliminar Webs Duplicadas output.  
  - Outputs: Split Out8 (for content fetch) and Edit Fields1 (for extra processing).  

---

#### 1.5 Page Content Retrieval

**Overview:**  
Fetches HTML content of each URL batch for further analysis.

**Nodes Involved:**  
- Split Out8  
- To html  
- Edit Fields1  
- Obtener Contenido Web

**Node Details:**  
- **Split Out8**  
  - Type: SplitOut  
  - Role: Splits batch for parallel content fetching and formatting.  
  - Inputs: Iterar Páginas output.  
  - Outputs: To html.  

- **To html**  
  - Type: Code  
  - Role: Converts or processes raw content to HTML format as needed.  
  - Inputs: Split Out8 output.  
  - Outputs: Send Results, Html to JSON, and HTML format viewer.  

- **Edit Fields1**  
  - Type: Set  
  - Role: Prepares or modifies fields before HTTP content retrieval.  
  - Inputs: Iterar Páginas output.  
  - Outputs: Obtener Contenido Web.  

- **Obtener Contenido Web**  
  - Type: HTTP Request  
  - Role: Fetches actual HTML content of each URL.  
  - Config: Continues on error without stopping.  
  - Inputs: Edit Fields1 output.  
  - Outputs: OpenAI1.  
  - Edge cases: HTTP timeouts, 404 errors, blocked by site.  

---

#### 1.6 AI Analysis with OpenAI GPT-4o-mini

**Overview:**  
Sends page content to OpenAI GPT-4o-mini for SEO audit analysis.

**Nodes Involved:**  
- OpenAI1  
- Code

**Node Details:**  
- **OpenAI1**  
  - Type: OpenAI node (LangChain integration)  
  - Role: Sends page content to GPT-4o-mini for technical SEO audit comments/suggestions.  
  - Inputs: Obtener Contenido Web output.  
  - Outputs: Code node.  
  - Config: Requires OpenAI credentials; model set to GPT-4o-mini.  
  - Edge cases: API rate limits, authentication errors, prompt failures.  

- **Code**  
  - Type: Code  
  - Role: Post-processes OpenAI output for formatting or aggregation.  
  - Inputs: OpenAI1 output.  
  - Outputs: Iterar Páginas (loop continuation).  

---

#### 1.7 Reporting and Output

**Overview:**  
Formats the audit results into HTML for visualization, Google Sheets for data storage, and emails reports.

**Nodes Involved:**  
- To html  
- Html to JSON  
- Split Out10  
- Google Sheets1  
- Send Results  
- HTML format viewer  
- Respond to Webhook

**Node Details:**  
- **To html**  
  - (Described above) Formats data as HTML for email and viewing.  

- **Html to JSON**  
  - Type: Code  
  - Role: Converts HTML formatted audit data into JSON structure for Google Sheets.  
  - Inputs: To html output.  
  - Outputs: Split Out10.  

- **Split Out10**  
  - Type: SplitOut  
  - Role: Splits JSON data batches for Google Sheets insertion.  
  - Inputs: Html to JSON output.  
  - Outputs: Google Sheets1.  

- **Google Sheets1**  
  - Type: Google Sheets node  
  - Role: Inserts or updates audit data rows into a specified Google Sheet.  
  - Inputs: Split Out10.  
  - Credentials: Requires Google API OAuth2 credentials.  
  - Edge cases: API quota limits, sheet access permissions.  

- **Send Results**  
  - Type: Gmail node  
  - Role: Sends the final HTML audit report via email.  
  - Inputs: To html output.  
  - Credentials: Requires Gmail OAuth2 credentials.  
  - Edge cases: Email sending failures, authentication errors.  

- **HTML format viewer**  
  - Type: HTML node  
  - Role: Provides an HTML preview of the audit report within n8n.  
  - Inputs: To html output.  
  - Outputs: Respond to Webhook.  

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Returns final audit report as HTTP response to the initial webhook call.  
  - Inputs: HTML format viewer output.  

---

#### 1.8 Error Handling

**Overview:**  
Stops workflow execution upon critical errors such as request failures or sitemap retrieval issues.

**Nodes Involved:**  
- Req Error  
- Req Error1  
- Sitemap Error

**Node Details:**  
- **Req Error**  
  - Type: Stop and Error  
  - Role: Stops workflow on HTTP request errors from Req robots node.  

- **Req Error1**  
  - Type: Stop and Error  
  - Role: Stops workflow on HTTP request errors from Split Out2 node.  

- **Sitemap Error**  
  - (Described above) Stops workflow if sitemap cannot be fetched or parsed.  

---

### 3. Summary Table

| Node Name              | Node Type              | Functional Role                              | Input Node(s)             | Output Node(s)                            | Sticky Note                                                                                       |
|------------------------|------------------------|----------------------------------------------|---------------------------|-------------------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                | Webhook                | Receives target website URL via HTTP         | External HTTP request     | URL WEB                                  | using: pag=example.com Real example: https://yourspace.app.n8n.cloud/webhook-test/...            |
| MANUAL                 | Manual Trigger         | Allows manual workflow start                   | User                      | URL WEB                                  |                                                                                                 |
| URL WEB                | Set                    | Sets target URL variable                       | Webhook, MANUAL           | LANGUAGE                                 |                                                                                                 |
| LANGUAGE               | Set                    | Sets language context                          | URL WEB                   | UA Rotativo1                             |                                                                                                 |
| UA Rotativo1           | Code                   | Rotates User-Agent strings                     | LANGUAGE                  | Method detect                            |                                                                                                 |
| Method detect          | Code                   | Detects HTTP method and prepares request      | UA Rotativo1              | Req robots                              |                                                                                                 |
| Req robots             | HTTP Request           | Fetches robots.txt                             | Method detect             | extract sitemap url, Req Error          |                                                                                                 |
| extract sitemap url    | Code                   | Extracts sitemap URLs from robots.txt          | Req robots                | Maping Sitemap                          |                                                                                                 |
| Maping Sitemap         | HTTP Request           | Fetches sitemap XML                            | extract sitemap url       | XML1, Sitemap Error                      |                                                                                                 |
| XML1                   | XML                    | Parses sitemap XML to extract URLs             | Maping Sitemap            | Split Out2                              |                                                                                                 |
| Sitemap Error          | Stop and Error         | Stops on sitemap fetch/parsing errors          | Maping Sitemap (error)    |                                           |                                                                                                 |
| Split Out2             | SplitOut               | Splits URLs for filtering or error handling   | XML1                      | Filter URL, Req Error1                   |                                                                                                 |
| Filter URL             | Filter                 | Filters URLs by criteria                        | Split Out2                | Filter URL Intern                        |                                                                                                 |
| Filter URL Intern      | Filter                 | Further filters URLs to exclude sitemap items | Filter URL                | Eliminar Webs Duplicadas                 | Filter sitemap.xml, usa ese filtro para excluir urls del sitemap                               |
| Eliminar Webs Duplicadas| Remove Duplicates      | Removes duplicate URLs                          | Filter URL Intern         | Iterar Páginas                          | En caso de haber duplicados o sitemap mal hecho                                                |
| Iterar Páginas         | SplitInBatches         | Iterates over URLs in batches                   | Eliminar Webs Duplicadas  | Split Out8, Edit Fields1                 |                                                                                                 |
| Split Out8             | SplitOut               | Splits batch for parallel content fetching     | Iterar Páginas            | To html                                |                                                                                                 |
| To html                | Code                   | Converts content to HTML for reporting          | Split Out8                | Send Results, Html to JSON, HTML format viewer |                                                                                                 |
| Edit Fields1           | Set                    | Prepares fields before content fetch            | Iterar Páginas            | Obtener Contenido Web                   |                                                                                                 |
| Obtener Contenido Web  | HTTP Request           | Fetches HTML content of URLs                    | Edit Fields1              | OpenAI1                                |                                                                                                 |
| OpenAI1                | OpenAI (LangChain)     | Sends content to GPT-4o-mini for SEO audit      | Obtener Contenido Web    | Code                                   |                                                                                                 |
| Code                   | Code                   | Post-processes OpenAI output                     | OpenAI1                   | Iterar Páginas (loop continuation)      |                                                                                                 |
| Html to JSON           | Code                   | Converts HTML report to JSON for Sheets          | To html                   | Split Out10                            |                                                                                                 |
| Split Out10            | SplitOut               | Splits JSON data for Google Sheets               | Html to JSON              | Google Sheets1                         |                                                                                                 |
| Google Sheets1         | Google Sheets          | Inserts audit data into Google Sheets             | Split Out10               |                                           |                                                                                                 |
| Send Results           | Gmail                  | Emails the HTML audit report                       | To html                   |                                           |                                                                                                 |
| HTML format viewer     | HTML                   | Displays HTML report preview in n8n               | To html                   | Respond to Webhook                     |                                                                                                 |
| Respond to Webhook     | Respond to Webhook     | Returns audit report as HTTP response              | HTML format viewer        |                                           |                                                                                                 |
| Req Error              | Stop and Error         | Stops on robots.txt request error                  | Req robots (error)        |                                           |                                                                                                 |
| Req Error1             | Stop and Error         | Stops on URL split or filtering error              | Split Out2 (error)        |                                           |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set HTTP Method (GET or POST)  
   - Add query parameter `pag` to receive target domain  
   - No authentication required  
   - Position as entry point  

2. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Enables manual workflow start  

3. **Create Set Node (URL WEB)**  
   - Type: Set  
   - Extract and store URL from webhook query parameter `pag` or manual input  
   - Output to LANGUAGE node  

4. **Create Set Node (LANGUAGE)**  
   - Type: Set  
   - Define language context variable (e.g., "en", "es")  
   - Output to UA Rotativo1  

5. **Create Code Node (UA Rotativo1)**  
   - Type: Code  
   - Implement logic to select random User-Agent string from predefined list  
   - Output to Method detect node  

6. **Create Code Node (Method detect)**  
   - Type: Code  
   - Prepare HTTP request options (method, headers including User-Agent)  
   - Output to Req robots  

7. **Create HTTP Request Node (Req robots)**  
   - Type: HTTP Request  
   - Request robots.txt from target URL (`https://<target-domain>/robots.txt`)  
   - On error, continue to Req Error node  
   - Output to extract sitemap url  

8. **Create Code Node (extract sitemap url)**  
   - Type: Code  
   - Parse robots.txt content to extract all `Sitemap:` URLs  
   - Output to Maping Sitemap node  

9. **Create HTTP Request Node (Maping Sitemap)**  
   - Type: HTTP Request  
   - Request sitemap XML from extracted sitemap URL  
   - On error, continue to Sitemap Error node  
   - Output to XML1  

10. **Create XML Node (XML1)**  
    - Type: XML  
    - Parse sitemap XML to extract URLs  
    - Output to Split Out2  

11. **Create Stop and Error Node (Sitemap Error)**  
    - Type: Stop and Error  
    - Configure message for sitemap retrieval/parsing failure  

12. **Create SplitOut Node (Split Out2)**  
    - Type: SplitOut  
    - Split batch of URLs from sitemap parsing  
    - Output to Filter URL and Req Error1  

13. **Create Filter Node (Filter URL)**  
    - Type: Filter  
    - Filter URLs with criteria (exclude non-HTML, external links)  
    - Output to Filter URL Intern  

14. **Create Filter Node (Filter URL Intern)**  
    - Type: Filter  
    - Further internal filters for sitemap exclusions  
    - Output to Eliminar Webs Duplicadas  

15. **Create Remove Duplicates Node (Eliminar Webs Duplicadas)**  
    - Type: Remove Duplicates  
    - Remove duplicate URLs  
    - Output to Iterar Páginas  

16. **Create SplitInBatches Node (Iterar Páginas)**  
    - Type: SplitInBatches  
    - Batch process URLs for efficient handling  
    - Output to Split Out8 and Edit Fields1  

17. **Create SplitOut Node (Split Out8)**  
    - Type: SplitOut  
    - Split batch for content fetching and formatting  
    - Output to To html  

18. **Create Code Node (To html)**  
    - Type: Code  
    - Convert or prepare content in HTML format for reporting  
    - Output to Send Results, Html to JSON, HTML format viewer  

19. **Create Set Node (Edit Fields1)**  
    - Type: Set  
    - Prepare request parameters or metadata for content fetch  
    - Output to Obtener Contenido Web  

20. **Create HTTP Request Node (Obtener Contenido Web)**  
    - Type: HTTP Request  
    - Fetch HTML content of URLs  
    - Continue on error  
    - Output to OpenAI1  

21. **Create OpenAI Node (OpenAI1)**  
    - Type: OpenAI (LangChain)  
    - Configure to use GPT-4o-mini model  
    - Send page content for SEO audit analysis  
    - Requires OpenAI API credentials  
    - Output to Code node  

22. **Create Code Node (Code)**  
    - Type: Code  
    - Post-process OpenAI response for further processing or loop control  
    - Output back to Iterar Páginas for next batch  

23. **Create Code Node (Html to JSON)**  
    - Type: Code  
    - Convert HTML report to JSON structured data for Google Sheets  
    - Input from To html  
    - Output to Split Out10  

24. **Create SplitOut Node (Split Out10)**  
    - Type: SplitOut  
    - Split JSON data for Google Sheets insertion  
    - Output to Google Sheets1  

25. **Create Google Sheets Node (Google Sheets1)**  
    - Type: Google Sheets  
    - Insert or update rows with audit data  
    - Requires Google OAuth2 credentials  

26. **Create Gmail Node (Send Results)**  
    - Type: Gmail  
    - Send final HTML audit report by email  
    - Requires Gmail OAuth2 credentials  

27. **Create HTML Node (HTML format viewer)**  
    - Type: HTML  
    - Display HTML report preview inside n8n UI  
    - Output to Respond to Webhook  

28. **Create Respond to Webhook Node**  
    - Type: Respond to Webhook  
    - Sends final HTML report as HTTP response to webhook caller  

29. **Create Stop and Error Nodes (Req Error, Req Error1)**  
    - Type: Stop and Error  
    - Stop workflow on HTTP request failures from robots.txt or URL splitting  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Webhook URL example usage: `https://yourspace.app.n8n.cloud/webhook-test/bbdf9cca-e5f4-4bae-afb1-a893ffb51b18?pag=onlineseoscan.com` | Webhook node description                             |
| Filter URL Intern node uses a filter specifically to exclude unwanted URLs from sitemap processing | Node comment                                        |
| Eliminar Webs Duplicadas node handles duplicates or malformed sitemap URLs                         | Node comment                                        |
| OpenAI node uses GPT-4o-mini model for technical SEO audit analysis                                | Requires OpenAI API credentials                      |
| Google Sheets and Gmail nodes require respective OAuth2 credentials for operation                   | Google API & Gmail API credentials                   |

---

**Disclaimer:**  
The provided text is solely derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.