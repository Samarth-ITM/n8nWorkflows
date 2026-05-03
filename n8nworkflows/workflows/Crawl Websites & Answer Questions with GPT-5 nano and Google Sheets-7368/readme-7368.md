Crawl Websites & Answer Questions with GPT-5 nano and Google Sheets

https://n8nworkflows.xyz/workflows/crawl-websites---answer-questions-with-gpt-5-nano-and-google-sheets-7368


# Crawl Websites & Answer Questions with GPT-5 nano and Google Sheets

### 1. Workflow Overview

This workflow automates the process of crawling websites starting from a user-provided URL, extracting structured information from the site’s pages, and answering user questions about the site using AI. It integrates Google Sheets as a database for storing extracted site data and leverages OpenAI GPT models (including GPT-5 nano and GPT-4o) via LangChain nodes for URL validation, sitemap processing, content summarization, and interactive Q&A.

**Target Use Cases:**  
- Automatically discover and index website content via sitemap crawling.  
- Extract and store summaries, language, heading hierarchies, and link structures from pages.  
- Enable conversational AI to answer user queries about the site based on stored data and live fetches.  
- Support ongoing updates by marking when data schema is ready and switching between crawl and agent modes.

**Logical Blocks:**

- **1.1 Chat Trigger & Schema Check:** Capture user input and check if website data is already indexed (via Google Sheets flag).  
- **1.2 Branch A - Agent Mode:** If data exists, use an AI Agent to answer user questions using stored data and live HTTP fetches.  
- **1.3 Branch B - URL Validation & Initial Crawling:** Validate input URL, fetch robots.txt, extract sitemap URL, download and parse sitemap index.  
- **1.4 Sitemap Selection & Expansion:** Use GPT-4o to select relevant sitemaps; download and parse the pages sitemap.  
- **1.5 Page-by-Page Processing Loop:** Iterate over URLs, fetch content, convert HTML to Markdown, extract structured data via GPT-5 nano, and save to Google Sheets.  
- **1.6 Data Update and Schema Marking:** Append or update a “ready” flag in the sheet to switch to Agent Mode for future queries.

---

### 2. Block-by-Block Analysis

#### 1.1 Chat Trigger & Schema Check

- **Overview:**  
  Captures chat input via webhook and checks Google Sheets for existing data schema flag to decide workflow branch.

- **Nodes Involved:**  
  - Chat web (trigger)  
  - Get data schema (Google Sheets)  
  - If (conditional branching)  

- **Node Details:**  
  - **Chat web**  
    - Type: LangChain chat trigger node (webhook)  
    - Config: Public webhook with Basic Auth, captures `chatInput`  
    - Inputs: HTTP request from user  
    - Outputs: Passes chat input downstream  
    - Edge cases: Authentication failure, missing input, webhook downtime.

  - **Get data schema**  
    - Type: Google Sheets node (read rows)  
    - Config: Filter rows where “Data schema” column is true (boolean)  
    - Credentials: Google Sheets OAuth2  
    - Input: Triggered by chat input  
    - Output: Data rows indicating if schema exists for site  
    - Edge cases: Sheet access failure, rate limits, empty results.

  - **If**  
    - Type: Conditional node  
    - Config: Checks if data schema exists (boolean true)  
    - Input: Output of Google Sheets  
    - Output: Branch A if true, Branch B if false  
    - Edge cases: Expression evaluation errors, data format inconsistencies.

#### 1.2 Branch A – Agent Mode (Existing Indexed Data)

- **Overview:**  
  Enables conversational AI acting as the website, answering questions using stored data and live HTTP fetches.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - OpenAI Chat Model (gpt-5-nano)  
  - Simple Memory (context window)  
  - Get row(s) in sheet in Google Sheets  
  - HTTP Request2 (allows agent HTTP fetch)  
  - Respond to Chat1  

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent for conversational AI  
    - Config: System message instructs acting as website, accessing Google Sheets and HTTP Request as tools  
    - Uses prompt with user chat input  
    - Inputs: chatInput from webhook node  
    - Outputs: Agent’s answer in `output` field  
    - Linked credentials: OpenAI API, Google Sheets OAuth2, HTTP Request node (as AI tool)  
    - Edge cases: API rate limits, tool invocation failures, memory overflow.  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat model  
    - Model: GPT-5-nano  
    - Credentials: OpenAI API  
    - Role: Powers AI Agent responses  
    - Edge cases: API errors, model unavailability.

  - **Simple Memory**  
    - Type: LangChain memory buffer window  
    - Config: 50-message context window for short-term chat history  
    - Connects to AI Agent as memory  
    - Edge cases: Memory overflow, data loss on restart.

  - **Get row(s) in sheet in Google Sheets**  
    - Type: Google Sheets node (read rows)  
    - Purpose: Provides agent access to stored site data as a tool  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: Sheet read errors, latency.

  - **HTTP Request2**  
    - Type: HTTP Request (Tool)  
    - Purpose: Allows AI Agent to perform live HTTP fetches when needed  
    - Edge cases: Request timeouts, blocked URLs.

  - **Respond to Chat1**  
    - Type: LangChain chat node  
    - Purpose: Sends AI Agent’s answer back to user  
    - Edge cases: Message formatting errors.

#### 1.3 Branch B – URL Validation & Initial Crawling

- **Overview:**  
  Validates the user input URL, fetches `robots.txt`, extracts sitemap URL, downloads sitemap index, and parses it.

- **Nodes Involved:**  
  - AI Agent1 (URL validation agent)  
  - OpenAI Chat Model1 (gpt-5-nano)  
  - Structured Output Parser  
  - If1  
  - Respond to Chat (invalid URL message)  
  - OPTIONS (flags for sitemap selection)  
  - UA Rotativo1 (random User-Agent generator)  
  - Req robots (HTTP Request for robots.txt)  
  - Req Error (stop on bad URL)  
  - extract sitemap url (Code)  
  - Maping Sitemap (HTTP Request for sitemap index)  
  - Sitemap Error (stop if sitemap access fails)  
  - XML1 (XML to JSON conversion)  

- **Node Details:**  
  - **AI Agent1**  
    - Type: LangChain Agent for URL validation  
    - Config: Returns JSON with fields `URL` (string) and `URL_bool` (boolean true if valid)  
    - Uses OpenAI Chat Model1 + Structured Output Parser to enforce JSON  
    - Input: `chatInput` from webhook  
    - Output: JSON parsed output  

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat model  
    - Model: GPT-5-nano  
    - Credentials: OpenAI API  

  - **Structured Output Parser**  
    - Type: LangChain output parser for JSON  
    - Config: Expects schema `{ "URL": string, "URL_bool": boolean }`  

  - **If1**  
    - Checks if `URL_bool` is true  
    - If true: continue to OPTIONS  
    - If false: Responds to user with error message  

  - **Respond to Chat**  
    - Sends message: “You must enter a valid URL example: https://google.es”  

  - **OPTIONS**  
    - Sets boolean flags to choose which sitemaps to process:  
      - scan_pages: true  
      - scan_posts, category, tags: false  

  - **UA Rotativo1**  
    - Code node that randomly selects a User-Agent string (desktop/mobile, various OS/browsers)  
    - Purpose: Reduce blocking from HTTP requests by rotating agents  

  - **Req robots**  
    - HTTP Request node fetching `robots.txt` from validated URL (`{{ AI Agent1.output.URL }}/robots.txt`)  
    - Sends realistic headers including User-Agent from UA Rotativo1  
    - On failure: routes to Req Error node  

  - **Req Error**  
    - Stops workflow with error: “URL mal introducida, debes introducir con el siguiente formato: ejemplo.com”  

  - **extract sitemap url**  
    - Code node parses `robots.txt` content to extract first `Sitemap:` URL line  
    - Outputs `{ sitemapUrl: string|null }`  

  - **Maping Sitemap**  
    - HTTP Request downloads sitemap index at `sitemapUrl`  
    - On failure: routes to Sitemap Error node  

  - **Sitemap Error**  
    - Stops workflow with error: “Sitemap no encontrado o acceso bloqueado”  

  - **XML1**  
    - Converts sitemap XML content to JSON structure  

#### 1.4 Sitemap Selection & Expansion

- **Overview:**  
  Uses GPT-4o to select relevant sitemaps based on OPTIONS flags, downloads selected sitemap, and converts it to JSON.

- **Nodes Involved:**  
  - Message a model (GPT-4o)  
  - Maping Sitemaps (HTTP Request)  
  - XML (XML to JSON)  

- **Node Details:**  
  - **Message a model**  
    - LangChain OpenAI node with GPT-4o  
    - System prompt instructs model to select sitemaps for pages/posts/categories/tags according to OPTIONS flags  
    - Input: first 3 sitemap URLs from sitemapindex JSON  
    - Output: JSON with selected sitemap URLs such as `sitemap_page`  

  - **Maping Sitemaps**  
    - HTTP Request downloads sitemap XML from selected sitemap URL (e.g. `sitemap_page`)  
    - Sends standard headers for language and security  

  - **XML**  
    - Converts downloaded sitemap XML to JSON (`urlset.url` array)  

#### 1.5 Page-by-Page Processing Loop

- **Overview:**  
  Processes each URL from the sitemap, fetches HTML, converts to Markdown, extracts structured data via GPT-5 nano, and appends to Google Sheets.

- **Nodes Involved:**  
  - Merge (Code)  
  - Split URLs (splitOut)  
  - Loop Over Items (splitInBatches)  
  - Req URL (HTTP Request)  
  - HTML to Markdown  
  - Message a model1 (GPT-5 nano)  
  - Append row in sheet in Google Sheets  

- **Node Details:**  
  - **Merge**  
    - Code node transforms JSON array of URLs into an object `{ url 1: "...", url 2: "..." }` for splitting  

  - **Split URLs**  
    - Splits merged URLs object into separate items (one URL per item)  

  - **Loop Over Items**  
    - Splits processing into batches, iterates over URL items for parallel processing  

  - **Req URL**  
    - HTTP Request fetches HTML content of each URL using randomized User-Agent  
    - Edge cases: request failures, timeouts  

  - **HTML to Markdown**  
    - Converts fetched HTML content to Markdown format for easier text processing  

  - **Message a model1**  
    - LangChain OpenAI node (GPT-5 nano)  
    - System prompt instructs model to extract:  
      - Page summary  
      - Language  
      - H1 and heading hierarchy  
      - Internal links (excluding images, CSS, JS)  
      - External links  
    - Uses `$fromAI` to map extracted data to columns for Google Sheets insertion  

  - **Append row in sheet in Google Sheets**  
    - Google Sheets node in append mode  
    - Inserts row with extracted fields: Lang, Page URL, External URLs, Internal URLs, Summary Content, H1 and hierarchy  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: API limits, data mapping errors.

#### 1.6 Data Update and Schema Marking

- **Overview:**  
  Ensures a row with `Data schema = true` exists in the sheet to mark the site as indexed and ready for Agent mode.

- **Nodes Involved:**  
  - Complete (Google Sheets appendOrUpdate)  

- **Node Details:**  
  - **Complete**  
    - Google Sheets node with appendOrUpdate operation  
    - Matches on `Data schema` column to set boolean flag true  
    - Marks the dataset as ready for future queries to use Agent mode  
    - Edge cases: concurrency conflicts, update failures.

---

### 3. Summary Table

| Node Name                     | Node Type                                 | Functional Role                                            | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                      |
|-------------------------------|-------------------------------------------|------------------------------------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Chat web                      | LangChain chatTrigger (webhook)            | Captures user chat input                                   | (Trigger)                    | Get data schema                | Overview: Web chat trigger with Basic Auth, captures `chatInput`.                              |
| Get data schema               | Google Sheets                              | Reads if Data schema=true exists                            | Chat web                     | If                            | Filters rows where Data schema is true.                                                        |
| If                           | Conditional                               | Branches depending on schema existence                      | Get data schema              | AI Agent / AI Agent1           | Branches to Agent mode or URL validation & crawling.                                          |
| AI Agent                     | LangChain Agent                            | Conversational AI answering based on stored data           | If (true branch)             | Respond to Chat1               | Acts as website using Google Sheets & HTTP fetch tools.                                       |
| OpenAI Chat Model             | LangChain OpenAI Chat                      | Powers AI Agent with GPT-5-nano                             | AI Agent                    | AI Agent                     | GPT-5-nano model used.                                                                         |
| Simple Memory                 | LangChain Memory Buffer                    | Stores short-term conversation context                      | -                           | AI Agent                     | 50-message context window.                                                                     |
| Get row(s) in sheet in Google Sheets | Google Sheets                             | Allows Agent to read site data                              | AI Agent                    | AI Agent (as tool)            | Provides database access for Agent.                                                           |
| HTTP Request2                 | HTTP Request Tool                          | Allows Agent to make live HTTP requests                     | AI Agent                    | AI Agent (as tool)            | Agent’s live HTTP fetch tool.                                                                 |
| Respond to Chat1              | LangChain chat                            | Sends Agent response to user                                | AI Agent                    | -                             |                                                                                              |
| AI Agent1                    | LangChain Agent                            | Validates input URL                                         | If (false branch)            | If1                           | Returns JSON `{URL, URL_bool}` to validate URL.                                               |
| OpenAI Chat Model1            | LangChain OpenAI Chat                      | Powers AI Agent1 with GPT-5-nano                            | AI Agent1                   | AI Agent1                    | GPT-5-nano model used.                                                                         |
| Structured Output Parser      | LangChain Output Parser Structured         | Enforces JSON output format                                 | AI Agent1                   | AI Agent1                    | Ensures URL validation output is JSON.                                                       |
| If1                          | Conditional                               | Checks if URL is valid                                      | AI Agent1                   | OPTIONS / Respond to Chat      | Branches based on URL validity.                                                               |
| Respond to Chat               | LangChain chat                            | Sends invalid URL message                                   | If1                        | -                             | “You must enter a valid URL example...”                                                      |
| OPTIONS                      | Set                                       | Sets flags for sitemap scanning                             | If1                        | UA Rotativo1                  | Flags: scan_pages=true, others false.                                                        |
| UA Rotativo1                 | Code                                      | Randomly selects User-Agent string                          | OPTIONS                    | Req robots                   | Reduces blocking by rotating User-Agent strings.                                             |
| Req robots                   | HTTP Request                              | Fetches robots.txt from validated URL                       | UA Rotativo1                | extract sitemap url / Req Error | Fetch robots.txt to get sitemap URL.                                                         |
| Req Error                    | Stop and Error                            | Stops workflow on invalid robots.txt or URL                 | Req robots                  | -                             | “URL mal introducida...”                                                                       |
| extract sitemap url          | Code                                      | Extracts sitemap URL from robots.txt content                | Req robots                  | Maping Sitemap                | Parses robots.txt for Sitemap line.                                                           |
| Maping Sitemap               | HTTP Request                              | Downloads sitemap index XML                                 | extract sitemap url          | XML1 / Sitemap Error           | Downloads sitemap index.                                                                       |
| Sitemap Error                | Stop and Error                            | Stops workflow if sitemap download fails                    | Maping Sitemap              | -                             | “Sitemap no encontrado o acceso bloqueado”                                                    |
| XML1                        | XML to JSON                              | Converts sitemap XML to JSON                                | Maping Sitemap              | Message a model               | XML to JSON conversion of sitemap index.                                                     |
| Message a model              | LangChain OpenAI                          | Selects relevant sitemap URLs based on OPTIONS             | XML1                        | Maping Sitemaps              | Uses GPT-4o to select sitemaps for pages/posts/etc.                                           |
| Maping Sitemaps              | HTTP Request                              | Downloads selected sitemap XML                              | Message a model             | XML                         | Downloads pages sitemap.                                                                       |
| XML                         | XML to JSON                              | Converts sitemap XML to JSON                                | Maping Sitemaps             | Merge                       | Converts pages sitemap XML to JSON.                                                          |
| Merge                       | Code                                      | Converts array of URLs to object for splitting             | XML                         | Split URLs                  | Prepares URLs for batch processing.                                                           |
| Split URLs                  | Split Out                                | Splits URLs object into individual URL items               | Merge                       | Loop Over Items              | Creates one workflow item per URL.                                                            |
| Loop Over Items             | Split In Batches                         | Iterates over URLs in batches                               | Split URLs                  | Complete / Req URL           | Processes each URL in parallel batches.                                                       |
| Req URL                     | HTTP Request                              | Fetches HTML content for each URL                           | Loop Over Items             | HTML to Markdown             | Fetches page HTML with rotating User-Agent.                                                  |
| HTML to Markdown            | Markdown Conversion                       | Converts HTML content to Markdown                           | Req URL                     | Message a model1             | Simplifies content for AI extraction.                                                        |
| Message a model1            | LangChain OpenAI                          | Extracts structured data from page content                 | HTML to Markdown            | Append row in sheet in Google Sheets | GPT-5-nano extracts summary, language, headings, links.                                      |
| Append row in sheet in Google Sheets | Google Sheets                             | Appends extracted data row to sheet                         | Message a model1            | -                           | Stores page data: Lang, H1, External/Internal URLs, Summary.                                 |
| Complete                    | Google Sheets                             | Appends or updates “Data schema = true” row                | Loop Over Items             | -                           | Marks site as indexed and ready for Agent mode.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node (LangChain Chat Trigger):**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configuration: Public webhook, Basic Auth enabled, captures `chatInput`  
   - Position: Input start node.

2. **Add Google Sheets Node to Check Data Schema:**  
   - Type: Google Sheets (read rows)  
   - Credentials: Connect Google Sheets OAuth2 account  
   - Sheet: “Web” (gid=0)  
   - Filter: `Data schema = true`  
   - Connect from Chat Trigger node.

3. **Add If Node to Branch Workflow:**  
   - Condition: Check if rows with `Data schema = true` exist  
   - True branch → Agent Mode (Block 1.2)  
   - False branch → URL Validation & Crawl (Block 1.3)

---

4. **Branch A – Agent Mode Setup:**

   a. Add LangChain Agent Node:  
      - Type: `@n8n/n8n-nodes-langchain.agent`  
      - Prompt system message: “Act as a website, use sheet tool and HTTP requests to answer.”  
      - Input: `chatInput` from Chat Trigger  
      - Connect OpenAI Chat Model, Google Sheets (read rows), HTTP Request as tools.

   b. Add OpenAI Chat Model Node:  
      - Model: GPT-5-nano  
      - Credentials: OpenAI API  
      - Connect output to Agent’s AI language model input.

   c. Add Simple Memory Node:  
      - Context window length: 50  
      - Connect as Agent’s memory input.

   d. Add Google Sheets Node (read rows):  
      - Provides database access to Agent  
      - Credentials: Google Sheets OAuth2

   e. Add HTTP Request Node (HTTP Request2):  
      - Enables Agent live fetches  
      - No fixed URL; used as AI tool.

   f. Add LangChain Chat Node (Respond to Chat1) to send Agent’s output back to user.

---

5. **Branch B – URL Validation & Initial Crawling:**

   a. Add LangChain Agent Node (AI Agent1) to validate URLs:  
      - System message: “Return JSON with URL and boolean URL_bool”  
      - Use OpenAI Chat Model1 (GPT-5-nano)  
      - Add Structured Output Parser with JSON schema `{ URL: string, URL_bool: boolean }`.

   b. Add If Node (If1) to check `URL_bool`:  
      - True → continue  
      - False → send error message with LangChain Chat Node “Respond to Chat”.

   c. Add Set Node (OPTIONS) with boolean flags:  
      - `scan_pages: true`  
      - `scan_posts: false`  
      - `category: false`  
      - `tags: false`

   d. Add Code Node (UA Rotativo1) to randomly select User-Agent string.

   e. Add HTTP Request Node (Req robots) to fetch `robots.txt` from validated URL + `/robots.txt`, with headers including User-Agent from UA Rotativo1.

   f. Add Stop and Error Node (Req Error) if `robots.txt` fetch fails.

   g. Add Code Node (extract sitemap url) to parse `robots.txt` and extract `Sitemap:` URL.

   h. Add HTTP Request Node (Maping Sitemap) to download sitemap index XML.

   i. Add Stop and Error Node (Sitemap Error) if sitemap fetch fails.

   j. Add XML Node (XML1) to convert sitemap index to JSON.

---

6. **Sitemap Selection and Expansion:**

   a. Add LangChain OpenAI Node (Message a model) with GPT-4o:  
      - Prompt to select relevant sitemaps based on OPTIONS flags  
      - Input: first 3 sitemap URLs from sitemap index JSON.

   b. Add HTTP Request Node (Maping Sitemaps) to fetch selected sitemap XML (e.g., pages sitemap).

   c. Add XML Node (XML) to convert sitemap XML to JSON.

---

7. **Page-by-Page Processing Loop:**

   a. Add Code Node (Merge) to convert URL array to key-value object.

   b. Add Split Out Node (Split URLs) to create one item per URL.

   c. Add Split In Batches Node (Loop Over Items) to iterate over URLs in batches.

   d. Add HTTP Request Node (Req URL) to fetch page HTML with rotating User-Agent.

   e. Add Markdown Node (HTML to Markdown) to convert HTML to Markdown.

   f. Add LangChain OpenAI Node (Message a model1) with GPT-5 nano:  
      - Extract summary, language, H1, internal/external links, etc.  
      - Map output fields to Google Sheets columns using `$fromAI()` expressions.

   g. Add Google Sheets Node (Append row in sheet) to append extracted data as new rows.

---

8. **Data Update and Schema Marking:**

   a. Add Google Sheets Node (Complete) with appendOrUpdate operation:  
      - Match on `Data schema` column and set it to true  
      - Marks site as indexed and ready for Agent mode.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is created by OXSR. More info and nodes: https://n8n.io/creators/oxsr11/                         | Node By OXSR sticky note                                                                         |
| GitHub repository with additional nodes and info: https://github.com/oxsr                                      | Node By OXSR sticky note                                                                         |
| Video explanation and detailed project overview can be found at the shared Google Sheets URL:                   | https://docs.google.com/spreadsheets/d/112qqkm4omdSzDT2jI17IQAxYvGjKuGlYxj6XytDA5L8/edit?usp=sharing |
| User experience summary: First runs trigger crawling; subsequent queries use AI Agent with stored data.         | Sticky Note7                                                                                     |
| Models used: GPT-5 nano for URL and content processing, GPT-4o for sitemap selection, Simple Memory for chat.   | Sticky Note3, Sticky Note6                                                                       |
| The “Data schema” boolean flag in Google Sheets controls switching between crawling and Agent Q&A modes.        | Sticky Note2, Sticky Note9                                                                       |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data processed is legal and public.