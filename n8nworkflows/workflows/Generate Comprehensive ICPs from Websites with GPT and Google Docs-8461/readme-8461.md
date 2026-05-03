Generate Comprehensive ICPs from Websites with GPT and Google Docs

https://n8nworkflows.xyz/workflows/generate-comprehensive-icps-from-websites-with-gpt-and-google-docs-8461


# Generate Comprehensive ICPs from Websites with GPT and Google Docs

### 1. Workflow Overview

This workflow automates the creation of a comprehensive Ideal Customer Profile (ICP) document derived from a company’s own website content. It targets growth, marketing, sales, and founder teams who want a rigorous, data-grounded ICP without manual research. The workflow accepts a company website URL and business name via a form, crawls and scrapes relevant website pages, uses AI (OpenAI GPT-4.1-mini model) to generate a detailed ICP in Markdown format, converts the Markdown into a formatted Google Doc, and saves the document in a specified Google Drive folder. Finally, it redirects the user to that folder.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Collect user inputs (website URL and business name) via a web form.
- **1.2 Website Crawling and Scraping:** Crawl up to 20 pages of the target website (including sitemap), focusing on main content extraction.
- **1.3 AI ICP Generation:** Use Langchain node with OpenAI GPT-4.1-mini to analyze scraped data and produce a detailed ICP in Markdown.
- **1.4 Markdown to Google Docs Conversion:** Parse and format the Markdown output into Google Docs batchUpdate requests.
- **1.5 Google Docs Creation and Update:** Create a new Google Doc titled “ICP for <Business Name>” in a specific Drive folder and update it with the converted content.
- **1.6 User Redirection:** Redirect the form submitter to the Google Drive folder containing the generated ICP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Collects Website URL and Business Name from the user via a form and redirects the user to the Google Drive folder upon completion.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger (webhook-based)  
    - Role: Entry point collecting user inputs (Website URL, Business Name) with validation (required fields).  
    - Configuration:  
      - Form titled “Create your ICP”  
      - HTML instructions explaining the workflow steps and output  
      - Fields: Website URL (required), Business Name (required)  
      - Response: Redirect to the Google Drive folder URL after execution  
    - Inputs: HTTP webhook request from form submitter  
    - Outputs: Data containing form fields to next node (Crawl Website)  
    - Edge cases: User submits invalid URL or no input; form validation prevents empty submission but no URL format validation is explicit. Possible webhook timeouts if downstream nodes are slow.  
    - Sticky Note: Setup instructions mention setting redirect URL to `={{google_drive_folder_url}}`.

#### 2.2 Website Crawling and Scraping

- **Overview:**  
  Crawls up to 20 pages of the submitted website including the sitemap, extracting main content in markdown format, then scrapes individual page content to enrich data.

- **Nodes Involved:**  
  - Crawl Website  
  - Scrape Website Content

- **Node Details:**

  - **Crawl Website**  
    - Type: HTTP Request  
    - Role: Sends POST request to Firecrawl API to crawl the entire website domain with sitemap included.  
    - Configuration:  
      - URL: `https://api.firecrawl.dev/v2/crawl`  
      - Method: POST  
      - JSON Body: Contains URL from form input, sitemap included, crawlEntireDomain=true, limit=20 pages, prompt to crawl entire website, scrape only main content, accept markdown format, max age 48 hours.  
      - Header: Authorization Bearer token using `{{API_KEY}}` (Firecrawl API key).  
    - Input: Website URL from form node  
    - Output: Crawled data JSON, passed to Scrape Website Content  
    - Edge cases: API key missing or invalid, API rate limits, network failures, crawl limit exceeded, malformed URL input.

  - **Scrape Website Content**  
    - Type: HTTP Request  
    - Role: Scrapes content from specific URLs returned by Crawl Website node for detailed data.  
    - Configuration:  
      - URL: dynamically set from `$json.url` (from crawl results)  
      - Method: GET (default)  
      - Header: Authorization Bearer `fc-afdcaee8b15c4f858109766aa71bac2e` (static token in workflow)  
      - Retry on Fail enabled  
    - Input: URLs from Crawl Website  
    - Output: Scraped page content JSON, passed to ICP Creator  
    - Edge cases: Invalid URLs, access denied, token expiration, network issues, inconsistent content format.

#### 2.3 AI ICP Generation

- **Overview:**  
  Uses Langchain’s chainLlm node with OpenAI GPT-4.1-mini to analyze the scraped website data exclusively, generating a detailed ICP document in Markdown format. The prompt enforces strict rules to avoid hallucination, differentiates facts vs inferences, and produces a structured deliverable with multiple predefined sections.

- **Nodes Involved:**  
  - ICP Creator  
  - OpenAI Chat Model

- **Node Details:**

  - **ICP Creator**  
    - Type: Langchain chainLlm node  
    - Role: Constructs detailed ICP markdown text from scraped website data, enforcing strict rules and output format.  
    - Configuration:  
      - Input text includes all scraped website data in JSON stringified form between `<website_data>` tags.  
      - Instructions: No external browsing, quote facts, tag facts vs inferences, confidence scores, multiple ICP variants if needed, predefined output sections A-F.  
      - On error: Continue regular output (avoid halting workflow).  
      - Prompt includes a system message positioning the AI as a senior B2B/B2C go-to-market analyst specializing in ICP and ABM.  
    - Input: Scraped website content JSON  
    - Output: Markdown text with structured ICP content  
    - Edge cases: AI token limits, API errors, incomplete data causing low confidence, hallucination risks if prompt altered.  
    - Sub-workflow: Uses OpenAI Chat Model node as language model backend.

  - **OpenAI Chat Model**  
    - Type: Langchain lmChatOpenAi node  
    - Role: Executes the GPT-4.1-mini model with OpenAI API to generate text responses.  
    - Configuration:  
      - Model: gpt-4.1-mini  
      - Credentials: OpenAI API key attached  
    - Input: Prompt from ICP Creator  
    - Output: AI-generated Markdown ICP text  
    - Edge cases: Rate limits, API key errors, model downtime.

#### 2.4 Markdown to Google Docs Conversion

- **Overview:**  
  Converts the Markdown ICP output into a batchUpdate request format compatible with Google Docs API, including headings, lists, inline styles, and tables for rich formatting.

- **Nodes Involved:**  
  - Markdown to Google Doc

- **Node Details:**

  - **Markdown to Google Doc**  
    - Type: Code node (JavaScript)  
    - Role: Parses Markdown string into Google Docs API `batchUpdate` request objects applying paragraph styles, inline text styles (bold, italic, links), blockquotes, lists, and real tables.  
    - Configuration:  
      - Custom JS code implementing a robust Markdown parser with support for headings, blockquotes, numbered and bulleted lists, inline formatting, and tables converted to native Google Docs tables.  
      - Takes the Markdown text from ICP Creator node output.  
      - Outputs JSON requests array to be sent to Google Docs API.  
    - Input: Markdown text JSON  
    - Output: JSON with Google Docs batchUpdate requests  
    - Edge cases: Complex or malformed Markdown may cause parsing errors; large documents may hit Google Docs API limits; unsupported Markdown features are not handled.

#### 2.5 Google Docs Creation and Update

- **Overview:**  
  Creates a new Google Document titled “ICP for <Business Name>” in a specific Drive folder, then applies the converted Markdown formatting and content via batch update.

- **Nodes Involved:**  
  - Create a document  
  - Update a document

- **Node Details:**

  - **Create a document**  
    - Type: Google Docs node  
    - Role: Creates an empty Google Doc with dynamic title based on Business Name from form input, placed in a Drive folder.  
    - Configuration:  
      - Title: `ICP for {{ Business Name }}`  
      - Drive Folder: uses `{{google_drive_folder_id}}` placeholder to specify target folder  
      - Credential: Google Docs OAuth2 attached  
      - Executes once per workflow run  
    - Input: JSON with Business Name from form  
    - Output: JSON including created document ID for update  
    - Edge cases: Invalid folder ID, insufficient permissions, quota limits.

  - **Update a document**  
    - Type: HTTP Request  
    - Role: Sends batchUpdate request to Google Docs API to apply the formatted ICP content into the newly created document.  
    - Configuration:  
      - URL: `https://docs.googleapis.com/v1/documents/{{documentId}}:batchUpdate` where documentId is from Create a document node  
      - Method: POST  
      - Body: JSON requests from Markdown to Google Doc node  
      - Credential: Google Docs OAuth2 attached  
      - On error: Continue regular output (avoid halting)  
    - Input: BatchUpdate request JSON  
    - Output: Google Docs API response  
    - Edge cases: API errors, permission issues, malformed update requests.

#### 2.6 User Redirection

- **Overview:**  
  After ICP document creation, redirects the user who submitted the form to the Google Drive folder containing their generated ICP.

- **Nodes Involved:**  
  - On form submission (response configured to redirect)

- **Node Details:**  
  - Configured in form trigger node to respond with a redirect URL pointing to `{{google_drive_folder_url}}` (Google Drive folder containing ICP documents).  
  - Edge cases: Incorrect folder URL causes broken redirect; user browser blocking redirects.

---

### 3. Summary Table

| Node Name             | Node Type                            | Functional Role                                   | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                  |
|-----------------------|------------------------------------|-------------------------------------------------|----------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| On form submission    | Form Trigger                       | Collect user inputs (Website URL, Business Name) and redirect user | None                       | Crawl Website           | Setup checklist: set redirect URL to `={{google_drive_folder_url}}`                          |
| Crawl Website         | HTTP Request                      | Crawl website via Firecrawl API for main content pages | On form submission         | Scrape Website Content  | Firecrawl API key placeholder `{{API_KEY}}` to be set in header                             |
| Scrape Website Content | HTTP Request                      | Scrape individual page URLs for detailed content | Crawl Website              | ICP Creator             | Retry enabled; uses static Firecrawl token in header                                        |
| ICP Creator           | Langchain chainLlm                | Generate detailed ICP Markdown from scraped data | Scrape Website Content     | Markdown to Google Doc   | Uses OpenAI GPT-4.1-mini; prompt enforces strict no-hallucination rules                      |
| OpenAI Chat Model     | Langchain lmChatOpenAi            | Backend AI model for ICP Creator                  | ICP Creator (ai_languageModel) | ICP Creator             | OpenAI API key credential required                                                          |
| Markdown to Google Doc | Code                              | Parse Markdown ICP into Google Docs batchUpdate JSON | ICP Creator                | Create a document       | Robust Markdown parser with tables, lists, inline styles                                    |
| Create a document     | Google Docs                       | Create Google Doc in Drive folder with ICP title | Markdown to Google Doc      | Update a document       | Folder ID placeholder `={{google_drive_folder_id}}` to be set                               |
| Update a document     | HTTP Request                      | Apply batchUpdate requests to Google Doc content | Create a document           | None                    | Uses Google Docs OAuth2; continues on error                                                 |
| Sticky Note           | Sticky Note                      | Overview and setup instructions                   | None                       | None                    | Covers entire workflow overview and setup steps                                            |
| Sticky Note1          | Sticky Note                      | Setup checklist for credentials and placeholders | None                       | None                    | Covers form redirect URL, Firecrawl headers, OpenAI and Google Docs credentials setup       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “On form submission” node** (Form Trigger):  
   - Configure a webhook to trigger on form submission.  
   - Add a form titled “Create your ICP” with:  
     - HTML instructions describing workflow steps.  
     - Required fields: “Website URL” (text), “Business Name” (text).  
   - Set response mode to redirect with URL parameter `={{google_drive_folder_url}}` to send users to the ICP Google Drive folder after submission.

2. **Create “Crawl Website” node** (HTTP Request):  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v2/crawl`  
   - Headers: Authorization Bearer `{{API_KEY}}` (Firecrawl API key credential)  
   - Body (JSON):  
     ```json
     {
       "url": "{{ $json['Website URL'] }}",
       "sitemap": "include",
       "crawlEntireDomain": true,
       "limit": 20,
       "prompt": "crawl entire website",
       "scrapeOptions": {
         "onlyMainContent": true,
         "maxAge": 172800000,
         "formats": ["markdown"]
       }
     }
     ```  
   - Connect “On form submission” → “Crawl Website”.

3. **Create “Scrape Website Content” node** (HTTP Request):  
   - Method: GET (default)  
   - URL: `={{ $json.url }}` (dynamic from Crawl Website output)  
   - Headers: Authorization Bearer static token `fc-afdcaee8b15c4f858109766aa71bac2e`  
   - Enable “Retry on Fail.”  
   - Connect “Crawl Website” → “Scrape Website Content”.

4. **Create “ICP Creator” node** (Langchain chainLlm):  
   - Use Langchain’s chainLlm node.  
   - Set prompt text with instructions to analyze only `<website_data>` JSON stringified content (from scraped data), strict no hallucination, produce multi-section ICP in Markdown.  
   - Include a system message defining the AI role as senior ICP analyst.  
   - On error: continue regular output.  
   - Connect “Scrape Website Content” → “ICP Creator”.

5. **Create “OpenAI Chat Model” node** (Langchain lmChatOpenAi):  
   - Select model “gpt-4.1-mini”.  
   - Attach OpenAI API credentials.  
   - Connect “OpenAI Chat Model” as AI language model backend for “ICP Creator”.

6. **Create “Markdown to Google Doc” node** (Code):  
   - Add JavaScript code to parse Markdown text from “ICP Creator” output to Google Docs API batchUpdate requests.  
   - Supports headings, lists, inline styles, blockquotes, and tables.  
   - Connect “ICP Creator” → “Markdown to Google Doc”.

7. **Create “Create a document” node** (Google Docs):  
   - Title: `ICP for {{ $('On form submission').item.json['Business Name'] }}`  
   - Drive Folder ID: set to `={{google_drive_folder_id}}` (your target folder)  
   - Attach Google Docs OAuth2 credentials.  
   - Set to execute once per workflow run.  
   - Connect “Markdown to Google Doc” → “Create a document”.

8. **Create “Update a document” node** (HTTP Request):  
   - Method: POST  
   - URL: `https://docs.googleapis.com/v1/documents/{{ $('Create a document').item.json.id }}:batchUpdate`  
   - Body: `={{ JSON.stringify($('Markdown to Google Doc').item.json) }}`  
   - Attach Google Docs OAuth2 credentials.  
   - On error: continue regular output.  
   - Connect “Create a document” → “Update a document”.

9. **Set up credentials:**  
   - Firecrawl API key for “Crawl Website” node header.  
   - OpenAI API key for “OpenAI Chat Model” node.  
   - Google Docs OAuth2 credentials for “Create a document” and “Update a document” nodes.

10. **Configure placeholders:**  
    - Replace `{{API_KEY}}` with your Firecrawl API key in “Crawl Website”.  
    - Replace `{{google_drive_folder_id}}` with your target Google Drive folder ID in “Create a document”.  
    - Replace `{{google_drive_folder_url}}` with your Google Drive folder URL in “On form submission” node redirect response.

11. **Test workflow:**  
    - Publish webhook URL from “On form submission”.  
    - Submit form with a real website URL (e.g., `vertodigital.com`) and business name.  
    - Verify creation of the Google Doc titled “ICP for <Business Name>” with formatted ICP content.  
    - Confirm redirection to Google Drive folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| The workflow strictly distinguishes facts vs. inferences in the AI output and includes confidence scores per ICP section for decision-ready quality control.   | Workflow prompt instructions inside “ICP Creator” node.                                                             |
| Firecrawl API key is required to crawl and scrape website content with sitemap support and content filtering.                                                  | https://take.ms/lGcUp (Firecrawl API key setup resource)                                                             |
| OpenAI API key is required for GPT-4.1-mini usage as the language model backend.                                                                                 | https://docs.n8n.io/integrations/builtin/credentials/openai/                                                        |
| Google Docs OAuth2 credentials must be configured for document creation and update.                                                                              | https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/                                          |
| Markdown to Google Docs conversion code supports real tables, lists, headings, blockquotes, inline styles (bold, italic, links), ensuring rich formatting.     | Custom JavaScript code inside “Markdown to Google Doc” node.                                                        |
| The user is redirected to the Google Drive folder after workflow completion for easy access to generated ICP documents.                                         | Configured in “On form submission” node response with redirect URL.                                                  |
| Recommended to test with a real site like `vertodigital.com` to verify end-to-end execution and output quality.                                                 | Sticky Note recommends test URL.                                                                                      |
| The workflow’s design avoids hallucinations by enforcing input data boundaries and explicit prompt rules.                                                      | “ICP Creator” prompt instructions and AI role definition.                                                            |

---

**Disclaimer:** The provided text is exclusively sourced from an automated n8n workflow designed for safe and legal use. It adheres to all applicable content policies and contains no illegal or protected content. All data processed is publicly accessible and authorized.