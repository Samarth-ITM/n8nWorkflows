Generate Research-Backed SEO Content Briefs with OpenAI and Firecrawl to Google Docs

https://n8nworkflows.xyz/workflows/generate-research-backed-seo-content-briefs-with-openai-and-firecrawl-to-google-docs-8289


# Generate Research-Backed SEO Content Briefs with OpenAI and Firecrawl to Google Docs

---

## 1. Workflow Overview

This workflow automates the generation of detailed, research-backed SEO content briefs from a single keyword or topic input. It targets content strategists, SEO teams, and copywriters who seek to streamline their briefing process by combining web scraping of top-ranking pages with advanced AI analysis. The output is a fully formatted Google Docs document containing a comprehensive SEO content brief ready for use.

### Logical Blocks

- **1.1 Input Reception:** Collect user keyword/topic input via a web form and initiate the workflow.
- **1.2 Web Search & Scraping:** Query Firecrawl API to fetch and scrape the top 5 relevant web pages for the keyword.
- **1.3 AI Analysis & Brief Generation:** Use an AI Agent with a Think tool and OpenAI GPT-4 to analyze scraped content and generate a unique SEO content brief in Markdown format.
- **1.4 Markdown to Google Docs Conversion:** Convert the Markdown brief into Google Docs batchUpdate requests for proper formatting.
- **1.5 Google Docs Creation & Population:** Create a new Google Doc named after the keyword and update it with the formatted content.
- **1.6 User Redirection:** Redirect the user to the Google Drive folder containing the generated document after workflow completion.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
Receives keyword or topic input from the user via a web form and triggers the workflow execution.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - *Type/Role:* Form Trigger — starts workflow on form submit.  
  - *Configuration:*  
    - Form titled "VertoDigital Keyword to SEO Content Brief".  
    - One required input field labeled "Keyword/Topic".  
    - HTML description explaining the workflow steps and redirect instructions.  
    - Responds by redirecting users to the specified Google Drive folder URL (`={{google_drive_folder_url}}`).  
  - *Expressions:*  
    - Uses `{{google_drive_folder_url}}` to redirect after processing.  
  - *Connections:* Outputs to "FireCrawl Search & Scrape".  
  - *Potential Failures:*  
    - Form submission errors, invalid/missing keyword input.  
    - Redirect URL misconfiguration.  
  - *Version:* 2.2  

---

### 2.2 Web Search & Scraping

**Overview:**  
Searches the web for the input keyword using Firecrawl API and scrapes the main content of the top 5 relevant pages.

**Nodes Involved:**  
- FireCrawl Search & Scrape

**Node Details:**  

- **FireCrawl Search & Scrape**  
  - *Type/Role:* HTTP Request — calls Firecrawl API to perform search and scrape.  
  - *Configuration:*  
    - POST to `https://api.firecrawl.dev/v2/search`.  
    - JSON body includes:  
      - `query` dynamically set to form input keyword.  
      - `sources`: ["web"] (web search).  
      - `limit`: 5 results.  
      - `location`: "United States".  
      - `scrapeOptions`: only main content, max age 2 days (172800000 ms), parsers include PDFs, formats requested: markdown and links.  
    - Authorization header with `Bearer {{API_KEY}}`.  
  - *Expressions:*  
    - `{{ $('On form submission').item.json['Keyword/Topic'] }}` for query.  
  - *Connections:* Outputs to "AI Agent".  
  - *Potential Failures:*  
    - API key invalid or rate-limited.  
    - Network timeouts or malformed requests.  
    - No search results or empty content.  
  - *Version:* 4.2  

---

### 2.3 AI Analysis & Brief Generation

**Overview:**  
Analyzes the scraped content to generate a unique, SEO-optimized content brief in Markdown format, using a multi-step AI agent with integrated thinking capability.

**Nodes Involved:**  
- AI Agent  
- Think  
- OpenAI Chat Model

**Node Details:**  

- **AI Agent**  
  - *Type/Role:* Langchain Agent — orchestrates analysis and content brief generation.  
  - *Configuration:*  
    - Input prompt dynamically includes:  
      - The keyword/topic.  
      - Up to 5 scraped sources with their markdown content and URLs.  
      - Instruction to analyze each source (key points, unique angles, gaps), compare sources, brainstorm new ideas, identify SEO strategies, and produce a detailed markdown content brief.  
    - Uses a "Think" tool for strategic analysis before content generation.  
    - Final output is a Markdown-formatted SEO content brief.  
  - *Expressions:*  
    - References Firecrawl data via `{{ $json.data.web[i].markdown }}` and URLs.  
    - Injects form keyword dynamically.  
  - *Connections:* Outputs to "Markdown to JSON".  
  - *Potential Failures:*  
    - AI model errors or rate limits.  
    - Incorrect or incomplete input data causing analysis failure.  
    - Think tool invocation failure.  
  - *Version:* 1.7  

- **Think**  
  - *Type/Role:* AI Thought Process Tool — invoked by AI Agent to do strategic analysis first.  
  - *Configuration:* No parameters; acts as middleware for AI strategic thinking.  
  - *Connections:* Output piped into AI Agent's next step.  
  - *Potential Failures:* AI latency or timeout.  
  - *Version:* 1.1  

- **OpenAI Chat Model**  
  - *Type/Role:* Language Model — GPT-4.1-mini for generating text.  
  - *Configuration:* GPT-4.1-mini model selected for balance of capability and performance.  
  - *Credentials:* Uses OpenAI API key credential.  
  - *Connections:* Provides language model backend for AI Agent.  
  - *Potential Failures:* API quota or authentication errors.  
  - *Version:* 1  

---

### 2.4 Markdown to Google Docs Conversion

**Overview:**  
Transforms the AI-generated Markdown content brief into a structured JSON payload compatible with Google Docs API batchUpdate requests.

**Nodes Involved:**  
- Markdown to JSON

**Node Details:**  

- **Markdown to JSON**  
  - *Type/Role:* Code node — custom JavaScript to parse Markdown and generate Google Docs API requests.  
  - *Configuration:*  
    - Parses input Markdown to detect headings (H1-H6), paragraphs, blockquotes, horizontal rules, numbered and bulleted lists.  
    - Normalizes spacing, trims blank lines, and converts inline markdown styles (bold, italic, links) into Google Docs text styles.  
    - Builds safe text ranges for insertion and styling to prevent API errors.  
    - Generates batchUpdate requests to insert text, apply font styles (Sofia Sans, 12.5pt), paragraph styles, bullets, and inline styles.  
  - *Expressions:*  
    - Reads `items[0].json.output` containing Markdown from AI Agent.  
  - *Connections:* Outputs structured JSON to "Create a document".  
  - *Potential Failures:*  
    - Parsing errors due to unexpected Markdown syntax.  
    - Google Docs API limits with complex documents.  
  - *Version:* 2  

---

### 2.5 Google Docs Creation & Population

**Overview:**  
Creates a new Google Docs document in a specified Drive folder and updates it with the formatted SEO content brief.

**Nodes Involved:**  
- Create a document  
- Update a document

**Node Details:**  

- **Create a document**  
  - *Type/Role:* Google Docs node — creates a new blank Google Doc.  
  - *Configuration:*  
    - Title: `SEO Brief for <Keyword>` dynamically set using form input.  
    - Drive: "sharedWithMe" (shared drives or user’s own).  
    - Folder: specified by `={{google_drive_folder_id}}`.  
  - *Credentials:* Google Docs OAuth2 credential attached.  
  - *Connections:* Output document ID to "Update a document".  
  - *Potential Failures:*  
    - Missing or insufficient Drive permissions.  
    - Incorrect folder ID.  
  - *Version:* 2  

- **Update a document**  
  - *Type/Role:* HTTP Request — calls Google Docs API batchUpdate endpoint to apply formatting and insert content.  
  - *Configuration:*  
    - POST to `https://docs.googleapis.com/v1/documents/{{documentId}}:batchUpdate` where documentId comes from the created doc.  
    - Body: JSON payload from "Markdown to JSON" node.  
    - Auth: Google Docs OAuth2 credential.  
    - On error: continue workflow without failing (to avoid blocking).  
  - *Connections:* Terminal node; no further outputs.  
  - *Potential Failures:*  
    - Authorization errors or token expiry.  
    - API quota exceeded.  
    - Malformed request JSON.  
  - *Version:* 4.2  

---

### 2.6 User Redirection

**Overview:**  
Redirects the user to the Google Drive folder containing the newly created SEO content brief after form submission.

**Nodes Involved:**  
- Implemented as response mode on the "On form submission" node.

**Node Details:**  

- *Type/Role:* Part of form trigger response.  
- *Configuration:*  
  - Redirect URL set to `={{google_drive_folder_url}}`.  
- *Potential Failures:*  
  - Incorrect or inaccessible URL leads to failed redirect or user confusion.  

---

## 3. Summary Table

| Node Name                | Node Type                        | Functional Role                       | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                  |
|--------------------------|---------------------------------|-------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                    | Input reception of keyword/topic    | -                      | FireCrawl Search & Scrape | Overview and setup instructions about form, redirect, and credentials                       |
| FireCrawl Search & Scrape | HTTP Request                   | Search & scrape top 5 web pages     | On form submission     | AI Agent               | Setup checklist: API key in header, test with keyword, connection to AI                     |
| AI Agent                 | Langchain Agent                 | Analyze scraped content & generate brief | FireCrawl Search & Scrape | Markdown to JSON       | Detailed prompt instructs content analysis, SEO strategy, Markdown output                   |
| Think                    | Langchain Tool (Think)          | AI strategic analysis step           | AI Agent (ai_tool)     | AI Agent               | Part of AI Agent's internal thinking process                                                |
| OpenAI Chat Model        | Language Model (OpenAI GPT-4)   | Backend language generation          | -                      | AI Agent (ai_languageModel) | Credential setup required for OpenAI                                                       |
| Markdown to JSON         | Code                           | Convert Markdown to Google Docs requests | AI Agent               | Create a document       | Complex parsing for headings, lists, inline styles, safe ranges                             |
| Create a document        | Google Docs                    | Create blank Google Doc               | Markdown to JSON       | Update a document       | Set document title/folder, OAuth2 credentials needed                                      |
| Update a document        | HTTP Request (Google Docs API) | Insert formatted content              | Create a document      | -                      | Uses batchUpdate, OAuth2 authentication, continue on error                                 |
| Sticky Note              | Sticky Note                    | Documentation and setup guidance     | -                      | -                      | Covers overview, setup checklist, and resource links                                      |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: "On form submission"  
   - Configure form title: "VertoDigital Keyword to SEO Content Brief"  
   - Add a required text input field labeled "Keyword/Topic" with placeholder "Write your keyword/topic here".  
   - Add an HTML description field explaining the workflow steps and redirect.  
   - Set response mode: last node with redirect URL expression `={{google_drive_folder_url}}`.  
   - Save webhook URL for testing.

2. **Add an HTTP Request node for Firecrawl**  
   - Name: "FireCrawl Search & Scrape"  
   - Method: POST  
   - URL: `https://api.firecrawl.dev/v2/search`  
   - Authentication: none (API key via header)  
   - Headers: `Authorization: Bearer {{API_KEY}}` (replace `{{API_KEY}}` with your Firecrawl API key)  
   - Body type: JSON  
   - Body content (use expression to inject keyword):  
   ```json
   {
     "query": "{{ $('On form submission').item.json['Keyword/Topic'] }}",
     "sources": ["web"],
     "limit": 5,
     "location": "United States",
     "scrapeOptions": {
       "onlyMainContent": true,
       "maxAge": 172800000,
       "parsers": ["pdf"],
       "formats": ["markdown", "links"]
     }
   }
   ```  
   - Connect "On form submission" main output to this node.

3. **Add an AI Agent node**  
   - Name: "AI Agent"  
   - Select Langchain Agent type.  
   - Configure prompt text with dynamic insertion of keyword and Firecrawl scraped data (markdown and URLs for 5 results).  
   - Include detailed instructions for analysis, think tool usage, and markdown brief generation.  
   - Connect "FireCrawl Search & Scrape" main output to this node.

4. **Add a Think tool node**  
   - Name: "Think"  
   - Place it before the AI Agent’s main operation as an AI tool.  
   - Connect as ai_tool input from "Think" to "AI Agent".

5. **Add an OpenAI Chat Model node**  
   - Name: "OpenAI Chat Model"  
   - Model: GPT-4.1-mini  
   - Attach OpenAI credentials.  
   - Connect as ai_languageModel input to "AI Agent".

6. **Add a Code node for Markdown to JSON conversion**  
   - Name: "Markdown to JSON"  
   - Paste the provided JavaScript code that parses Markdown and creates Google Docs batchUpdate requests.  
   - Connect "AI Agent" main output to this node.

7. **Add a Google Docs node to create a document**  
   - Name: "Create a document"  
   - Set title: `SEO Brief for {{ $('On form submission').item.json['Keyword/Topic'] }}`  
   - Drive ID: "sharedWithMe"  
   - Folder ID: `={{google_drive_folder_id}}` (replace with your target Google Drive folder ID)  
   - Attach Google Docs OAuth2 credentials.  
   - Connect "Markdown to JSON" main output to this node.

8. **Add an HTTP Request node to update the document**  
   - Name: "Update a document"  
   - Method: POST  
   - URL: `https://docs.googleapis.com/v1/documents/{{ $('Create a document').item.json.id }}:batchUpdate`  
   - Authentication: Use Google Docs OAuth2 credentials.  
   - Body type: JSON  
   - Body content: `={{ JSON.stringify($('Markdown to JSON').item.json) }}`  
   - Set "On Error" behavior: continue.  
   - Connect "Create a document" main output to this node.

9. **Finalize connections**  
   - Ensure flow: On form submission → FireCrawl Search & Scrape → AI Agent (with Think and OpenAI Chat Model inputs) → Markdown to JSON → Create a document → Update a document.

10. **Configure credentials**  
    - Firecrawl API key for HTTP Request header.  
    - OpenAI API key for OpenAI Chat Model.  
    - Google Docs OAuth2 credentials for both Docs nodes.

11. **Test the workflow**  
    - Publish workflow.  
    - Open the form URL.  
    - Submit a keyword (e.g., "GEO strategy").  
    - Verify the Google Doc creation in the target Drive folder and user redirection.

---

## 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow enables fast, consistent, research-driven SEO content briefs without manual SERP analysis.                                  | Sticky Note overview in workflow mentions this as the primary benefit for content/SEO teams.        |
| Firecrawl API key is required for web search and scraping: https://take.ms/lGcUp                                                           | Firecrawl API key documentation referenced in Sticky Note.                                          |
| OpenAI API key is required for AI analysis: https://docs.n8n.io/integrations/builtin/credentials/openai/                                   | OpenAI credential setup guide in n8n docs.                                                         |
| Google OAuth2 credentials needed for Docs API access: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/           | Google Docs OAuth2 setup instructions referenced.                                                   |
| Test with example keyword "GEO strategy" to verify full end-to-end functionality.                                                           | Setup checklist Sticky Note.                                                                         |
| The Markdown to JSON code node provides advanced parsing that ensures proper Google Docs formatting including headings, lists, links, etc. | Critical for correct document styling and presentation.                                            |
| Redirect URL after form submission must be a valid and accessible Google Drive folder URL to avoid user confusion.                          | Configured in form trigger node response options.                                                  |

---

**Disclaimer:**  
The provided text and workflow are exclusively derived from an automated process created with n8n, a tool for integration and automation. This process strictly complies with content policies and contains no illegal or protected content. All data processed is legal and publicly available.

---