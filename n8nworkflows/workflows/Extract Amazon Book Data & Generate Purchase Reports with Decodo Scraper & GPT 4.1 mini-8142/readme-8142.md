Extract Amazon Book Data & Generate Purchase Reports with Decodo Scraper & GPT 4.1 mini

https://n8nworkflows.xyz/workflows/extract-amazon-book-data---generate-purchase-reports-with-decodo-scraper---gpt-4-1-mini-8142


# Extract Amazon Book Data & Generate Purchase Reports with Decodo Scraper & GPT 4.1 mini

### 1. Workflow Overview

This n8n workflow automates the extraction of Amazon book data from a bestseller listing page and generates a detailed purchase report. It leverages the Decodo Scraper API to crawl and render the dynamic web page, cleans and parses the HTML content, uses GPT-4.1-mini to extract structured book information, and compiles a comprehensive Markdown report. The report is then saved as a Google Document, converted to PDF, and automatically uploaded to a Slack channel for team distribution.

The workflow is organized into the following logical blocks:

- **1.1 Trigger and Input Setup:** Manual trigger initiation and configuration of key input parameters such as target URL and authentication token.
- **1.2 Web Scraping:** Calls Decodo Scraper API to fetch the Amazon bestseller page HTML content with headless browser emulation.
- **1.3 HTML Processing:** Cleans the raw HTML to extract meaningful text content suitable for AI analysis.
- **1.4 AI Extraction:** Uses an LLM agent powered by GPT-4.1-mini and a structured output parser to extract normalized book data in JSON format.
- **1.5 Report Generation:** Converts extracted data into a rich Markdown report featuring KPIs, top books, category breakdowns, and recommendations.
- **1.6 Google Drive Document Handling:** Creates a Google Doc from the Markdown, then exports it as a PDF.
- **1.7 Slack Upload:** Uploads the generated PDF report to a configured Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Setup

**Overview:**  
Starts the workflow on manual execution and sets required input fields including the target Amazon URL and authentication token for Decodo API.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Edit Fields

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually.  
  - Configuration: No parameters; triggers workflow execution on demand.  
  - Inputs: None  
  - Outputs: Connected to Edit Fields node  
  - Edge Cases: None  
  
- **Edit Fields**  
  - Type: Set node  
  - Role: Defines input variables such as:  
    - `url`: Amazon bestsellers URL (default: https://www.amazon.com/Best-Sellers-Books/zgbs/books)  
    - `Authenticate_Token`: Decodo API key (placeholder text instructs user to get token from Decodo dashboard)  
  - Configuration: Hardcoded string values for demo; user must replace with actual token.  
  - Inputs: Manual Trigger node output  
  - Outputs: Connects to Scraper API Request  
  - Edge Cases: Missing or invalid token will cause authentication failures downstream.

---

#### 1.2 Web Scraping

**Overview:**  
Sends a POST request to Decodo Scraper API to fetch the HTML content of the Amazon bestseller page with device emulation and headless JavaScript rendering.

**Nodes Involved:**  
- Scraper API Request

**Node Details:**

- **Scraper API Request**  
  - Type: HTTP Request (POST)  
  - Role: Calls Decodo Scraper API endpoint to scrape the target URL.  
  - Configuration:  
    - URL: https://scraper-api.decodo.com/v2/scrape  
    - Method: POST  
    - Headers: Accept: application/json, Authorization: Bearer token from input  
    - Body Parameters: `url` (from input), `device_type`: desktop (hardcoded)  
  - Inputs: Edit Fields output  
  - Outputs: Connects to HTML Response Parser  
  - Edge Cases:  
    - Authentication errors (invalid/missing token)  
    - API rate limits or quota exceeded  
    - Network timeouts  
    - Empty or malformed responses

---

#### 1.3 HTML Processing

**Overview:**  
Cleans the raw HTML returned from Decodo API by removing scripts, styles, and extraneous tags, producing normalized plain text for AI analysis.

**Nodes Involved:**  
- HTML Response Parser

**Node Details:**

- **HTML Response Parser**  
  - Type: Code node (JavaScript)  
  - Role: Processes `results[0].content` from Decodo response JSON, strips out scripts, styles, head, comments, SVG, canvas, and decodes common HTML entities, collapses whitespace and block tags to newlines.  
  - Key Expression: Custom `stripAll` function defined in JS code.  
  - Inputs: Scraper API Request output  
  - Outputs: Produces clean text and character count  
  - Edge Cases:  
    - Missing or empty HTML content results in error message output  
    - Unexpected HTML structure might cause incomplete cleaning

---

#### 1.4 AI Extraction

**Overview:**  
Uses GPT-4.1-mini to analyze cleaned HTML text and extract a structured array of books with fields like ASIN, title, author, rank, category, rating, price, publisher, language, etc. A structured output parser enforces JSON schema validity.

**Nodes Involved:**  
- OpenAI Chat Model  
- Structured Output Parser  
- Product Analyzer Agent

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides GPT-4.1-mini LLM for text processing.  
  - Configuration: Model set to "gpt-4.1-mini".  
  - Credentials: OpenAI API key credential.  
  - Inputs: Connected as AI language model input from Product Analyzer Agent node.  
  - Outputs: Passes data to Product Analyzer Agent.  
  - Edge Cases: API key limits, rate limiting, downtimes.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Validates and parses AI output to conform to a strict JSON schema example for book objects.  
  - Configuration: JSON schema example defines all expected fields and types for each book.  
  - Inputs: AI model output.  
  - Outputs: Structured JSON array for downstream processing.  
  - Edge Cases: Schema mismatches or malformed JSON might cause parsing errors.

- **Product Analyzer Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates the AI prompt with system message instructing to parse HTML, uses the OpenAI Chat Model and Structured Output Parser to extract book data.  
  - Configuration: Prompt includes the cleaned HTML text as input, system message defines assistant role to parse HTML and output structured JSON.  
  - Inputs: HTML Response Parser output (clean text)  
  - Outputs: Structured book data JSON array, connected to report builder.  
  - Edge Cases: Large input size exceeding token limits, incomplete extraction, malformed output.

---

#### 1.5 Report Generation

**Overview:**  
Transforms the structured book data JSON into a comprehensive Markdown report including executive KPIs, tables of popular and highly rated books, category breakdowns, price histograms, highlights, and recommendations.

**Nodes Involved:**  
- Build 📚 Book Purchase Report

**Node Details:**

- **Build 📚 Book Purchase Report**  
  - Type: Code node (JavaScript)  
  - Role: Processes array of book objects to compute metrics like total books, average/median price, rating averages, counts by category/subcategory/format, top books by ratings and rating score, price distribution histogram, and generates a Markdown report string.  
  - Key Expressions:  
    - Functions for median, mean, sum, currency formatting, bar graphs  
    - Tables built in Markdown syntax  
    - Highlight and recommendation logic based on data completeness and quality  
  - Inputs: Product Analyzer Agent output (structured book JSON)  
  - Outputs: Object containing Markdown report and structured stats for optional reuse  
  - Edge Cases: Empty or incomplete data arrays, missing price or rating fields.

---

#### 1.6 Google Drive Document Handling

**Overview:**  
Creates a Google Document from the Markdown report, then exports it as a PDF for easy sharing.

**Nodes Involved:**  
- Configure Google Drive Folder  
- Create document file  
- Convert document to PDF

**Node Details:**

- **Configure Google Drive Folder**  
  - Type: Set node  
  - Role: Sets `Drive Folder ID` (hardcoded folder ID) and timestamp string `Today` used for naming the document.  
  - Inputs: Build Book Purchase Report output  
  - Outputs: Create document file node  
  - Edge Cases: Invalid folder ID will cause Google API errors.

- **Create document file**  
  - Type: HTTP Request (Google Drive API)  
  - Role: Uploads a new Google Docs file with the Markdown content as text/markdown within a multipart request.  
  - Configuration:  
    - Endpoint: Google Drive API `files?uploadType=multipart`  
    - MIME type: Google Docs document  
    - Parent folder: from `Drive Folder ID`  
    - Content: Markdown report from previous node  
  - Credentials: Google Drive OAuth2  
  - Inputs: Configure Google Drive Folder output  
  - Outputs: File metadata including new file ID  
  - Edge Cases: Authentication failures, invalid folder permissions, malformed content.

- **Convert document to PDF**  
  - Type: Google Drive node  
  - Role: Downloads the created Google Doc as a PDF file by specifying conversion.  
  - Configuration: File ID from created Google Doc, conversion to application/pdf  
  - Credentials: Google Drive OAuth2  
  - Inputs: Create document file output  
  - Outputs: PDF binary file for next upload step  
  - Edge Cases: API quota limits, conversion errors.

---

#### 1.7 Slack Upload

**Overview:**  
Uploads the generated PDF report to a Slack channel with a comment summarizing the content.

**Nodes Involved:**  
- Upload report to Slack

**Node Details:**

- **Upload report to Slack**  
  - Type: Slack node  
  - Role: Uploads the PDF file to a designated Slack channel, with a friendly initial comment.  
  - Configuration:  
    - File name: Dynamic, e.g., "Book Purchase Report YYYY-MM-DD"  
    - Channel ID: Preconfigured Slack channel  
    - Initial comment: "📚 Book Purchase Report"  
    - Authentication: OAuth2 with Slack bot/user token  
  - Inputs: Convert document to PDF output (file binary)  
  - Outputs: Slack upload response (not connected further)  
  - Edge Cases: Slack token permissions issues, channel ID invalid, file size limits.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                                  | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                     |
|----------------------------|------------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                     | Manual start trigger for workflow                | None                          | Edit Fields                     | ### 1. Trigger Workflow Execution: The workflow starts manually by clicking **Execute workflow**.             |
| Edit Fields                | Set                                | Sets input parameters: target URL and API token | When clicking ‘Execute workflow’ | Scraper API Request             | ### 2. Edit Input Fields: Set required inputs like `targetUrl` and `Authenticate_Token`.                        |
| Scraper API Request        | HTTP Request (POST)                 | Calls Decodo Scraper API to fetch Amazon page   | Edit Fields                   | HTML Response Parser            | ### 3. Send Scraper API Request (Decodo): Calls API with headless JS and device emulation.                     |
| HTML Response Parser       | Code (JS)                          | Cleans and normalizes raw HTML content           | Scraper API Request           | Product Analyzer Agent          | ### 4. Parse HTML Response: Removes scripts/styles, leaving clean text for AI analysis.                        |
| OpenAI Chat Model          | Langchain OpenAI Chat Model        | Provides GPT-4.1-mini LLM for AI processing     | Product Analyzer Agent (AI LM) | Product Analyzer Agent (AI Parser) |                                                                                                               |
| Structured Output Parser   | Langchain Output Parser Structured | Enforces JSON schema on AI output                | OpenAI Chat Model (AI Output) | Product Analyzer Agent (main)  |                                                                                                               |
| Product Analyzer Agent     | Langchain Agent                    | Extracts structured book data from cleaned HTML | HTML Response Parser          | Build 📚 Book Purchase Report   | ### 5. Product Analyzer Agent (LLM): Extracts structured book data using AI and schema validation.             |
| Build 📚 Book Purchase Report | Code (JS)                          | Generates Markdown report from structured data   | Product Analyzer Agent        | Configure Google Drive Folder   | ### 6. Build Book Purchase Report: Converts JSON data to detailed Markdown report with KPIs and tables.        |
| Configure Google Drive Folder  | Set                                | Sets Google Drive folder ID and timestamp        | Build 📚 Book Purchase Report | Create document file           | ### Create report Book Purchase Report PDF - Configure Google Drive Folder, Create Google Document, Convert PDF |
| Create document file       | HTTP Request (Google Drive API)    | Creates Google Docs file from Markdown report    | Configure Google Drive Folder | Convert document to PDF         |                                                                                                               |
| Convert document to PDF    | Google Drive                      | Converts Google Doc to PDF                        | Create document file          | Upload report to Slack          |                                                                                                               |
| Upload report to Slack     | Slack                             | Uploads PDF report to Slack channel               | Convert document to PDF       | None                          | ### 10. Upload Report to Slack: Sends PDF report to Slack channel for instant distribution.                    |
| Sticky Note                | Sticky Note                       | Documentation and usage guidance                  | None                          | None                          | # Decodo Scraper API Workflow Template (full detailed workflow explanation with images and instructions)       |
| Sticky Note1               | Sticky Note                       | Documentation snippet for trigger step            | None                          | None                          | ### 1. Trigger Workflow Execution: The workflow starts manually by clicking **Execute workflow**.             |
| Sticky Note2               | Sticky Note                       | Documentation snippet for input fields             | None                          | None                          | ### 2. Edit Input Fields: Set required inputs like `targetUrl` and `deviceType`.                              |
| Sticky Note3               | Sticky Note                       | Documentation snippet for scraper API request      | None                          | None                          | ### 3. Send Scraper API Request (Decodo): Calls API with headless JS and device emulation.                     |
| Sticky Note4               | Sticky Note                       | Documentation snippet for HTML parser              | None                          | None                          | ### 4. Parse HTML Response: Cleans and normalizes raw HTML content.                                           |
| Sticky Note5               | Sticky Note                       | Documentation snippet for AI product analyzer      | None                          | None                          | ### 5. Product Analyzer Agent (LLM): Extracts structured book data using AI.                                  |
| Sticky Note6               | Sticky Note                       | Documentation snippet for report building          | None                          | None                          | ### 6. Build Book Purchase Report: Generates human-readable purchase report.                                 |
| Sticky Note7               | Sticky Note                       | Documentation snippet for Google Drive handling    | None                          | None                          | ### Create report Book Purchase Report PDF - Configure Google Drive Folder, Create Document, Convert to PDF.  |
| Sticky Note8               | Sticky Note                       | Documentation snippet for Slack upload              | None                          | None                          | ### 10. Upload Report to Slack: Uploads PDF report to Slack channel.                                         |
| Sticky Note9               | Sticky Note                       | Sample output report link                            | None                          | None                          | ## Sample output report from crawl data: https://s3.ap-southeast-1.amazonaws.com/automatewith.me/Book+Purchase+Report+2025-09-02 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No configuration needed  
   - Purpose: Start workflow manually

2. **Add Set Node “Edit Fields”**  
   - Type: Set  
   - Add fields:  
     - `url` (string): `"https://www.amazon.com/Best-Sellers-Books/zgbs/books"` (default target Amazon URL)  
     - `Authenticate_Token` (string): `"Get token from your Decodo dashboard (https://dashboard.decodo.com/web-scraping-api/scraper)"` (replace with actual token)  
   - Connect Manual Trigger output to this node

3. **Add HTTP Request Node “Scraper API Request”**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://scraper-api.decodo.com/v2/scrape`  
   - Headers:  
     - `Accept`: `application/json`  
     - `Authorization`: set to `{{$json.Authenticate_Token}}` (from input)  
   - Body (JSON parameters):  
     - `url`: `{{$json.url}}`  
     - `device_type`: `"desktop"` (can be changed to `mobile` or `tablet`)  
   - Connect Edit Fields output to this node

4. **Add Code Node “HTML Response Parser”**  
   - Type: Code (JavaScript)  
   - Paste the provided `stripAll` JavaScript code that cleans HTML from the response at `json.results[0].content`  
   - Outputs clean plain text (`text`) and character count (`chars`)  
   - Connect Scraper API Request output to this node

5. **Add Langchain Agent Node “Product Analyzer Agent”**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text prompt: `"Get top 10 best selling book from the below web content:\n{{ $json.text }}"`  
     - System message: `"You are a helpful assistant to parse the HTML content and output as well-structure JSON"`  
     - Enable structured output parser  
   - Connect HTML Response Parser output to this node

6. **Add Langchain OpenAI Chat Model Node “OpenAI Chat Model”**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4.1-mini`  
   - Authenticate with your OpenAI API key credential  
   - Connect as AI language model node input of Product Analyzer Agent

7. **Add Langchain Structured Output Parser Node “Structured Output Parser”**  
   - Type: Langchain Structured Output Parser  
   - Paste JSON schema example for expected book objects (fields like asin, title, author, rank, category, price, etc.)  
   - Connect as AI output parser node input of Product Analyzer Agent

8. **Add Code Node “Build 📚 Book Purchase Report”**  
   - Type: Code (JavaScript)  
   - Paste the JavaScript code that:  
     - Computes summary statistics (total books, avg price, ratings)  
     - Builds Markdown report with tables and highlights  
   - Connect Product Analyzer Agent output to this node

9. **Add Set Node “Configure Google Drive Folder”**  
   - Type: Set  
   - Fields:  
     - `Drive Folder ID`: `"1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI"` (replace with your Drive folder ID)  
     - `Today`: Current timestamp formatted as `ddMMyyyyhhmmss` using expression `{{$now.format("ddMMyyyyhhmmss")}}`  
   - Connect Build Book Purchase Report output to this node

10. **Add HTTP Request Node “Create document file”**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`  
    - Authentication: Google Drive OAuth2 credentials with Docs and Drive scopes  
    - Body (raw multipart/related content):  
      - Metadata JSON: name = `{{$json.Today}}`, mimeType = Google Docs, parents = Drive Folder ID  
      - Document content: Markdown from previous node under `'markdown'` property  
    - Headers: `Content-Type: multipart/related; boundary=foo_bar_baz`  
    - Connect Configure Google Drive Folder output to this node

11. **Add Google Drive Node “Convert document to PDF”**  
    - Type: Google Drive  
    - Operation: Download  
    - File ID: `={{$json.id}}` (from Create document file output)  
    - Conversion: Docs to PDF (`application/pdf`)  
    - Credentials: Google Drive OAuth2  
    - Connect Create document file output to this node

12. **Add Slack Node “Upload report to Slack”**  
    - Type: Slack  
    - Resource: File  
    - Operation: Upload  
    - Channel ID: Select your Slack channel ID  
    - File Name: `Book Purchase Report {{ $today.format('yyyy-MM-dd') }}`  
    - Initial Comment: `"📚 Book Purchase Report"`  
    - Authentication: Slack OAuth2 credentials with `files:write` and `chat:write` scopes  
    - Connect Convert document to PDF output to this node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                     | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates the combined use of Decodo Scraper API (headless JS scraping with device emulation) and GPT-4.1-mini for structured data extraction from dynamic web pages like Amazon bestseller lists. | Main workflow purpose and demonstration.                                                                         |
| Decodo Scraper API dashboard for token management: https://dashboard.decodo.com/web-scraping-api/scraper                                                                                                           | Required for API key setup.                                                                                       |
| Sample output report from crawl data available at: https://s3.ap-southeast-1.amazonaws.com/automatewith.me/Book+Purchase+Report+2025-09-02                                                                       | Example of generated book purchase report.                                                                       |
| Required credentials: Decodo API key, OpenAI API key, Google Drive/Docs OAuth2 with write scopes, Slack OAuth2 token with files:write and chat:write permissions.                                                   | Credential setup prerequisites.                                                                                   |
| Customization tips include changing target URL, device emulation, adjusting scraping wait conditions, modifying AI prompt and schema, and extending report formatting with logos or additional destinations (email, Notion). | For adapting the workflow to other sources or richer reports.                                                    |
| Workflow supports manual execution but can be extended with Cron triggers for scheduled automated reporting.                                                                                                       | Scheduling and automation options.                                                                                |
| Node version requirements: Use n8n version supporting Langchain nodes (v1.2+), Google Drive API with OAuth2 v4+, and Slack API v2.3+.                                                                              | Ensures compatibility with nodes used.                                                                            |

---

**Disclaimer:** The provided text is exclusively generated from an n8n workflow. All data processed are legal and public. No illegal or offensive content is included.