Discover Article URLs from Any Website with GPT-5-mini and Google Sheets

https://n8nworkflows.xyz/workflows/discover-article-urls-from-any-website-with-gpt-5-mini-and-google-sheets-11211


# Discover Article URLs from Any Website with GPT-5-mini and Google Sheets

### 1. Workflow Overview

This workflow, titled **"AI-Powered Multi-Source URL Discovery Engine"**, is designed to automatically discover article URLs from any given list of seed websites by leveraging AI and Google Sheets integration. It targets users who want to aggregate fresh article links from multiple publishers, blogs, or news sources regularly and efficiently.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Reading seed URLs from Google Sheets to serve as the starting points for discovery.
- **1.2 URL Processing Loop:** Iterating over each seed URL with rate limiting and error tolerance to prevent blocks.
- **1.3 Webpage Fetching & Conversion:** Downloading webpage HTML with spoofed browser headers and converting it to Markdown for cleaner AI input.
- **1.4 AI-Powered URL Extraction:** Using an AI language model to analyze page content and extract valid article URLs according to strict inclusion/exclusion rules.
- **1.5 URL Parsing & Normalization:** Cleaning, validating, deduplicating, and normalizing the AI output URLs.
- **1.6 Output & Storage:** Appending the cleaned URLs back into a Google Sheets spreadsheet with source and status tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Reads the seed URLs that will be processed from a designated Google Sheets document. These URLs represent the entry points for article discovery.

- **Nodes Involved:**  
  - Manual Trigger  
  - Daily Schedule (6 AM)  
  - Read Seed URLs

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger  
    - Role: Manual start of the workflow for on-demand runs.  
    - Configuration: Default with no parameters.  
    - Connections: Outputs to Read Seed URLs node.  
    - Edge Cases: None specific; manual user control.  

  - **Daily Schedule (6 AM)**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at 6 AM.  
    - Configuration: Cron expression `"0 6 * * *"` for daily 6 AM execution.  
    - Connections: Outputs to Read Seed URLs node.  
    - Edge Cases: Cron misconfiguration or timezone mismatch could cause missed runs.  

  - **Read Seed URLs**  
    - Type: Google Sheets  
    - Role: Reads the list of seed URLs from a specified sheet (requires `URL` column).  
    - Configuration:  
      - Document ID and Sheet Name are dynamically set to user’s spreadsheet and "Seed URLs" sheet.  
      - Uses Google Sheets OAuth2 credentials.  
    - Inputs: Triggers from Manual or Schedule nodes.  
    - Outputs: List of URLs passed to Loop Over URLs node.  
    - Edge Cases: Authentication errors, empty or malformed sheet, missing URL column.

#### 2.2 URL Processing Loop

- **Overview:**  
  Iterates over each seed URL one at a time with a wait period to avoid rapid-fire requests that might trigger rate limits or blocks.

- **Nodes Involved:**  
  - Loop Over URLs (SplitInBatches)  
  - Completion Summary (Set)  
  - Rate Limit (Wait)

- **Node Details:**

  - **Loop Over URLs**  
    - Type: SplitInBatches  
    - Role: Splits the input array of URLs into single-item batches (1 at a time).  
    - Configuration: Batch size = 1; no reset on error.  
    - Inputs: From Read Seed URLs.  
    - Outputs: To Completion Summary and Rate Limit.  
    - Edge Cases: Large input arrays may slow processing; failure handling is set to continue.  

  - **Completion Summary**  
    - Type: Set  
    - Role: Sets metadata about the current batch processing (count of URLs, timestamp, status).  
    - Configuration: Sets three variables:  
      - `urlsDiscovered` = length of items in batch  
      - `completedAt` = current ISO timestamp  
      - `status` = "completed"  
    - Inputs: From Loop Over URLs.  
    - Outputs: To Rate Limit node.  
    - Edge Cases: None critical; purely metadata.

  - **Rate Limit (3s)**  
    - Type: Wait  
    - Role: Adds a 3-second delay between processing each URL to avoid rate limiting.  
    - Configuration: Wait for 3 seconds.  
    - Inputs: From Completion Summary.  
    - Outputs: To Fetch Webpage HTML.  
    - Edge Cases: Delay may slow workflow with large inputs; no error expected.

#### 2.3 Webpage Fetching & Conversion

- **Overview:**  
  Downloads the HTML content from each seed URL using HTTP request with browser-like headers to avoid bot detection, then converts HTML to Markdown for easier AI processing.

- **Nodes Involved:**  
  - Fetch Webpage HTML (HTTP Request)  
  - HTML to Markdown

- **Node Details:**

  - **Fetch Webpage HTML**  
    - Type: HTTP Request  
    - Role: Fetches webpage content with spoofed browser headers.  
    - Configuration:  
      - URL dynamically set from current seed URL.  
      - Timeout: 30 seconds.  
      - Custom headers: User-Agent for Chrome on Windows, Accept headers for HTML.  
      - On error: Continue workflow without failing (error handling).  
    - Inputs: From Rate Limit node.  
    - Outputs: To HTML to Markdown node.  
    - Edge Cases: HTTP errors, timeouts, 403 Forbidden due to bot detection, malformed URLs.  

  - **HTML to Markdown**  
    - Type: Markdown Converter  
    - Role: Converts fetched HTML content to Markdown text for cleaner AI input.  
    - Configuration: Converts from field `$json.data` (HTML string).  
    - On error: Continue without failing.  
    - Inputs: From Fetch Webpage HTML.  
    - Outputs: To AI URL Extractor.  
    - Edge Cases: Malformed HTML may cause conversion issues; fallback continues workflow.

#### 2.4 AI-Powered URL Extraction

- **Overview:**  
  Uses a GPT-5-mini AI agent to analyze the Markdown content and extract valid article URLs according to strict inclusion and exclusion criteria.

- **Nodes Involved:**  
  - AI URL Extractor (LangChain Agent)  
  - OpenAI Chat Model (Language Model used by AI URL Extractor)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-5-mini model for language understanding.  
    - Configuration: Model set to "gpt-5-mini", no additional options or tools.  
    - Credentials: OpenAI API key (OAuth or API key-based).  
    - Inputs: Consumed by AI URL Extractor internally.  
    - Outputs: None standalone; feeds AI URL Extractor.  
    - Edge Cases: API quota reached, authentication errors, latency or timeout from OpenAI API.  

  - **AI URL Extractor**  
    - Type: LangChain Agent (AI Agent)  
    - Role: Processes Markdown text, applies extensive rules to identify valid article URLs and exclude navigation, tag pages, PDFs, etc.  
    - Configuration:  
      - Input text: Markdown content.  
      - System prompt includes detailed inclusion/exclusion rules and output format as JSON array.  
      - Output: Strict JSON array of objects with `url` and `source`.  
      - On error: Continue to next node.  
    - Inputs: From HTML to Markdown node.  
    - Outputs: To URL Parser & Normalizer node.  
    - Edge Cases: AI hallucination, malformed JSON output, API errors, empty or irrelevant output.

#### 2.5 URL Parsing & Normalization

- **Overview:**  
  Cleans AI output by parsing JSON, removing markdown formatting, normalizing URLs (stripping tracking parameters, hashes, trailing slashes), and deduplicating results.

- **Nodes Involved:**  
  - URL Parser & Normalizer (Code Node)

- **Node Details:**

  - **URL Parser & Normalizer**  
    - Type: Code (JavaScript)  
    - Role:  
      - Parses AI JSON output or raw URL strings.  
      - Removes markdown code fences.  
      - Normalizes URLs by removing tracking parameters (`utm_*`, `ref`, `fbclid`, etc.), hashes, and trailing slashes.  
      - Deduplicates URLs by exact match.  
      - Returns cleaned array of URL objects with `url` and `source`.  
    - Inputs: From AI URL Extractor.  
    - Outputs: To Save Discovered URLs node.  
    - Edge Cases: Malformed AI output causing parse errors, invalid URLs filtered out, empty results.

#### 2.6 Output & Storage

- **Overview:**  
  Appends the cleaned and normalized article URLs into the designated Google Sheets document, including the source domain and a status field.

- **Nodes Involved:**  
  - Save Discovered URLs

- **Node Details:**

  - **Save Discovered URLs**  
    - Type: Google Sheets  
    - Role: Appends or updates rows with discovered URLs.  
    - Configuration:  
      - Operation: Append or update existing rows by matching URL field (deduplication).  
      - Target sheet name: "Discovered URLs".  
      - Columns: `URL`, `Source`, `Status` ("Pending" for all new entries).  
      - Uses Google Sheets OAuth2 credentials (same as input).  
    - Inputs: From URL Parser & Normalizer.  
    - Outputs: Back to Loop Over URLs node (to continue batch processing).  
    - Edge Cases: Authentication failures, rate limits on Google Sheets API, malformed data rejection.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                      |
|-------------------------|-----------------------------|-------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Sticky Note - Introduction | Sticky Note                 | Overview & introduction        | —                            | —                           | ## 🚀 AI-Powered Multi-Source URL Discovery Engine... (full content as in node)                                  |
| Sticky Note - Input      | Sticky Note                 | Explains input stage           | —                            | —                           | ## 📥 Input Stage... (full content as in node)                                                                   |
| Sticky Note - Loop       | Sticky Note                 | Explains processing loop       | —                            | —                           | ## 🔄 Processing Loop... (full content as in node)                                                               |
| Sticky Note - Fetch      | Sticky Note                 | Explains web fetching          | —                            | —                           | ## 🌐 Web Fetching... (full content as in node)                                                                  |
| Sticky Note - AI         | Sticky Note                 | Explains AI URL extraction     | —                            | —                           | ## 🤖 AI URL Extraction... (full content as in node)                                                             |
| Sticky Note - Parser     | Sticky Note                 | Explains URL parsing           | —                            | —                           | ## 🔧 URL Parser & Normalizer... (full content as in node)                                                       |
| Sticky Note - Output     | Sticky Note                 | Explains output stage          | —                            | —                           | ## 💾 Output Stage... (full content as in node)                                                                  |
| Daily Schedule (6 AM)    | Schedule Trigger            | Automated daily run trigger    | —                            | Read Seed URLs              |                                                                                                                 |
| Manual Trigger           | Manual Trigger             | Manual run trigger             | —                            | Read Seed URLs              |                                                                                                                 |
| Read Seed URLs           | Google Sheets               | Reads seed URLs from sheet     | Manual Trigger, Daily Schedule (6 AM) | Loop Over URLs             |                                                                                                                 |
| Loop Over URLs           | SplitInBatches              | Batch processing of URLs       | Read Seed URLs               | Completion Summary, Rate Limit (3s) |                                                                                                                 |
| Completion Summary       | Set                         | Metadata assignment            | Loop Over URLs               | Rate Limit (3s)             |                                                                                                                 |
| Rate Limit (3s)          | Wait                        | Delay to avoid rate limiting   | Completion Summary           | Fetch Webpage HTML          |                                                                                                                 |
| Fetch Webpage HTML       | HTTP Request                | Fetches HTML with headers      | Rate Limit (3s)              | HTML to Markdown            |                                                                                                                 |
| HTML to Markdown         | Markdown Converter          | Converts HTML to Markdown      | Fetch Webpage HTML           | AI URL Extractor            |                                                                                                                 |
| OpenAI Chat Model        | LangChain LM Chat Model     | GPT-5-mini language model      | (Internal to AI URL Extractor) | AI URL Extractor          |                                                                                                                 |
| AI URL Extractor         | LangChain Agent             | Extracts article URLs via AI   | HTML to Markdown             | URL Parser & Normalizer     |                                                                                                                 |
| URL Parser & Normalizer  | Code                        | Cleans & deduplicates URLs     | AI URL Extractor             | Save Discovered URLs        |                                                                                                                 |
| Save Discovered URLs     | Google Sheets               | Saves discovered URLs to sheet | URL Parser & Normalizer      | Loop Over URLs              |                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node for manual workflow starts.
   - Add a **Schedule Trigger** node configured with cron expression `0 6 * * *` for daily 6 AM automatic runs.

2. **Read Seed URLs:**
   - Add a **Google Sheets** node named "Read Seed URLs".
   - Configure with your Google Sheets OAuth2 credentials.
   - Set Document ID to your spreadsheet.
   - Set Sheet Name to "Seed URLs".
   - Configure node to read rows containing at least a column named `URL`.

3. **Batch Processing Setup:**
   - Add a **SplitInBatches** node named "Loop Over URLs".
   - Configure batch size to 1.
   - Connect "Read Seed URLs" output to "Loop Over URLs" input.

4. **Completion Metadata:**
   - Add a **Set** node named "Completion Summary".
   - Configure static fields:  
     - `urlsDiscovered` = `{{$items().length}}`  
     - `completedAt` = `{{$now.toISO()}}`  
     - `status` = `"completed"`  
   - Connect "Loop Over URLs" output to "Completion Summary".

5. **Rate Limiting:**
   - Add a **Wait** node named "Rate Limit (3s)".
   - Configure to wait 3 seconds.
   - Connect "Completion Summary" output to "Rate Limit (3s)".

6. **Fetch Webpage HTML:**
   - Add an **HTTP Request** node named "Fetch Webpage HTML".
   - Set URL to `={{ $json.URL }}`.
   - Configure a 30-second timeout.
   - Add HTTP headers:  
     - `User-Agent`: `"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36"`  
     - `Accept`: `"text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8"`  
     - `Accept-Language`: `"en-US,en;q=0.5"`  
   - Set "On Error" to continue workflow.
   - Connect "Rate Limit (3s)" output to this node.

7. **Convert HTML to Markdown:**
   - Add a **Markdown** node named "HTML to Markdown".
   - Set input HTML to `={{ $json.data }}`.
   - On error continue.
   - Connect "Fetch Webpage HTML" output to this node.

8. **Configure OpenAI Credentials:**
   - Add your OpenAI API credentials for GPT-5-mini model usage.

9. **Add OpenAI Chat Model Node:**
   - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model".
   - Set model to `"gpt-5-mini"`.
   - Connect internally to AI URL Extractor node later.

10. **AI URL Extraction:**
    - Add a **LangChain Agent** node named "AI URL Extractor".
    - Input text: `={{ $json.data }}` (Markdown content).
    - System prompt: Use detailed prompt defining inclusion/exclusion rules and JSON output format as in the workflow.
    - Connect "HTML to Markdown" output to this node.
    - Set On Error to continue.

11. **URL Parsing & Normalization:**
    - Add a **Code** node named "URL Parser & Normalizer".
    - Paste the provided JavaScript code to parse AI output, normalize URLs by removing tracking params, hashes, trailing slashes, and deduplicate.
    - Connect "AI URL Extractor" output to this node.

12. **Save Discovered URLs:**
    - Add another **Google Sheets** node named "Save Discovered URLs".
    - Configure with same Google Sheets OAuth2 credentials.
    - Set Document ID and Sheet Name to your target sheet (e.g., "Discovered URLs").
    - Operation: Append or update rows, matching by URL.
    - Columns: `URL`, `Source`, `Status` (set to "Pending" for new entries).
    - Connect "URL Parser & Normalizer" output to this node.

13. **Loop Continuation:**
    - Connect "Save Discovered URLs" output back to the "Loop Over URLs" node to continue batch processing.

14. **Workflow Settings:**
    - Ensure workflow execution order is sequential (set to version v1).
    - Set error handling to continue where configured.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow is branded as the "AI-Powered Multi-Source URL Discovery Engine" designed for automated discovery of article URLs from arbitrary websites leveraging GPT-5-mini and Google Sheets for input/output integration.                                                                                                                                      | Introduction Sticky Note content             |
| Setup instructions include connecting Google Sheets and OpenAI API credentials, creating input and output sheets with proper columns, and adding seed URLs to start crawling.                                                                                                                                                                                 | Introduction Sticky Note content             |
| The AI extraction agent uses strict rules to include only content-rich article URLs and exclude navigation, category tags, PDFs, and other non-article pages, ensuring high-quality results.                                                                                                                                                                    | AI Sticky Note content                        |
| Rate limiting with a 3-second wait between URL fetches is implemented to avoid IP blocking and ensure reliable scraping.                                                                                                                                                                                                                                     | Loop Sticky Note content                      |
| The workflow uses a user-agent string that mimics Chrome browser on Windows to bypass simple bot detection mechanisms during HTTP requests.                                                                                                                                                                                                                | Fetch Sticky Note content                     |
| Output deduplication relies on URL normalization that removes tracking parameters, hashes, and trailing slashes to avoid duplicates in Google Sheets.                                                                                                                                                                                                       | Parser Sticky Note content                    |
| Google Sheets is used as both input source for seed URLs and output destination for discovered URLs, with columns carefully defined for URL, source domain, and status.                                                                                                                                                                                       | Output Sticky Note content                    |
| The workflow is designed for extensibility and error tolerance, continuing processing on individual URL failures without stopping the entire batch.                                                                                                                                                                                                        | General workflow design                       |
| For more on n8n Google Sheets integration, see: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googlesheets/                                                                                                                                                                                                                                  | n8n Docs                                     |
| For details on LangChain and n8n AI agent nodes, see: https://docs.n8n.io/integrations/ai/langchain/                                                                                                                                                                                                                                                          | n8n LangChain Docs                            |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.