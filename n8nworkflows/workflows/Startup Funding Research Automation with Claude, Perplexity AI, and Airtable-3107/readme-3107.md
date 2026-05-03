Startup Funding Research Automation with Claude, Perplexity AI, and Airtable

https://n8nworkflows.xyz/workflows/startup-funding-research-automation-with-claude--perplexity-ai--and-airtable-3107


# Startup Funding Research Automation with Claude, Perplexity AI, and Airtable

### 1. Workflow Overview

This workflow automates the discovery, extraction, and deep research of recently funded startups by monitoring news sources, extracting structured funding data, performing AI-driven company research, and storing results in Airtable. It is designed for venture capitalists, startup analysts, and market researchers who want to streamline funding round tracking and company intelligence gathering.

The workflow is logically divided into these blocks:

- **1.1 News Feed Retrieval & Parsing:** Fetches and parses news sitemap XML feeds from TechCrunch and VentureBeat.
- **1.2 Article Filtering & Content Extraction:** Splits feeds into individual articles, filters for funding announcements, and scrapes article HTML content.
- **1.3 Structured Data Extraction:** Uses AI to extract key funding details (company name, funding amount, investors, etc.) from article text.
- **1.4 Company Website Research:** Uses AI to find the official company website URL.
- **1.5 Data Aggregation & Airtable Storage:** Collects extracted data and writes it to Airtable Funding Round Base.
- **1.6 Deep Company Research (Subworkflow):** Executes a subworkflow to perform detailed company research using Perplexity AI or Jina Deep Search, extracting comprehensive company profiles.
- **1.7 Deep Research Data Parsing & Storage:** Parses deep research AI output into structured JSON and writes detailed company data to Airtable Company Deep Research Base.

---

### 2. Block-by-Block Analysis

#### 1.1 News Feed Retrieval & Parsing

**Overview:**  
This block fetches the latest news sitemap XML feeds from TechCrunch and VentureBeat, then parses the XML into JSON for further processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Techcrunch (TC) (HTTP Request)  
- Venturebeat (VB) (HTTP Request)  
- Parse TC XML (XML)  
- Parse VB XML (XML)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Config: No parameters  
  - Outputs: Triggers HTTP requests to news sitemaps  
  - Edge cases: None (manual trigger)  

- **Techcrunch (TC)**  
  - Type: HTTP Request  
  - Role: Fetch TechCrunch news sitemap XML  
  - Config: URL = https://techcrunch.com/news-sitemap.xml, GET method  
  - Outputs: Raw XML data  
  - Edge cases: HTTP errors, network timeouts  

- **Venturebeat (VB)**  
  - Type: HTTP Request  
  - Role: Fetch VentureBeat news sitemap XML  
  - Config: URL = https://venturebeat.com/news-sitemap.xml, GET method  
  - Outputs: Raw XML data  
  - Edge cases: HTTP errors, network timeouts  

- **Parse TC XML**  
  - Type: XML  
  - Role: Converts TechCrunch XML to JSON  
  - Config: Default options  
  - Inputs: Raw XML from TechCrunch HTTP node  
  - Outputs: JSON object with article metadata  
  - Edge cases: Malformed XML, parsing errors  

- **Parse VB XML**  
  - Type: XML  
  - Role: Converts VentureBeat XML to JSON  
  - Config: Default options  
  - Inputs: Raw XML from VentureBeat HTTP node  
  - Outputs: JSON object with article metadata  
  - Edge cases: Malformed XML, parsing errors  

---

#### 1.2 Article Filtering & Content Extraction

**Overview:**  
Splits the parsed JSON feeds into individual articles, filters articles mentioning funding announcements, fetches full article HTML, and extracts clean text for analysis.

**Nodes Involved:**  
- Split TC Articles (Split Out)  
- Split VB Articles (Split Out)  
- Filter (Filter)  
- Filter1 (Filter)  
- Get Funding Article HTML for scraping (TC) (HTTP Request)  
- Get Funding Article HTML for scraping (VB) (HTTP Request)  
- TC HTML Parser (HTML)  
- VB HTML Parser (HTML)  
- Merge Extracted Data (Merge)

**Node Details:**

- **Split TC Articles**  
  - Type: Split Out  
  - Role: Splits TechCrunch feed JSON array into individual article items  
  - Config: Field to split out = urlset.url  
  - Inputs: JSON from Parse TC XML  
  - Outputs: Individual article JSON objects  

- **Split VB Articles**  
  - Type: Split Out  
  - Role: Splits VentureBeat feed JSON array into individual article items  
  - Config: Field to split out = urlset.url  
  - Inputs: JSON from Parse VB XML  
  - Outputs: Individual article JSON objects  

- **Filter**  
  - Type: Filter  
  - Role: Filters TechCrunch articles containing the word "raise" in the news title  
  - Config: Condition: article title contains "raise" (case sensitive)  
  - Inputs: Individual TechCrunch articles  
  - Outputs: Articles likely about funding announcements  
  - Edge cases: Case sensitivity may miss some articles  

- **Filter1**  
  - Type: Filter  
  - Role: Filters VentureBeat articles containing "raise" in the URL field  
  - Config: Condition: loc contains "raise"  
  - Inputs: Individual VentureBeat articles  
  - Outputs: Articles likely about funding announcements  

- **Get Funding Article HTML for scraping (TC)**  
  - Type: HTTP Request  
  - Role: Fetches full HTML content of filtered TechCrunch articles  
  - Config: URL from article loc field  
  - Outputs: Raw HTML content  

- **Get Funding Article HTML for scraping (VB)**  
  - Type: HTTP Request  
  - Role: Fetches full HTML content of filtered VentureBeat articles  
  - Config: URL from article loc field  
  - Outputs: Raw HTML content  

- **TC HTML Parser**  
  - Type: HTML  
  - Role: Extracts article title and text content from TechCrunch HTML  
  - Config: Extract title from `.wp-block-post-title`, text from `.wp-block-post-content`  
  - Inputs: Raw HTML from TechCrunch article fetch  
  - Outputs: Clean text and title for analysis  

- **VB HTML Parser**  
  - Type: HTML  
  - Role: Extracts article title and text content from VentureBeat HTML  
  - Config: Extract title from `.article-title`, text from `#content`  
  - Inputs: Raw HTML from VentureBeat article fetch  
  - Outputs: Clean text and title for analysis  

- **Merge Extracted Data**  
  - Type: Merge  
  - Role: Combines parsed articles from both sources into one dataset  
  - Config: Default merge (append)  
  - Inputs: Outputs from TC HTML Parser and VB HTML Parser  
  - Outputs: Unified article dataset  

---

#### 1.3 Structured Data Extraction

**Overview:**  
Uses AI to extract structured funding data from article text, including company name, funding round, amount, investors, market, press release URL, and valuation.

**Nodes Involved:**  
- Extract Structured Data (Information Extractor)  
- Research URL (Chain LLM)  
- Perplexity (LM Chat OpenRouter)  
- Extract URL (Chain LLM)  
- Collect Data (Set)  
- Airtable (Funding Round Base)

**Node Details:**

- **Extract Structured Data**  
  - Type: Information Extractor (LangChain)  
  - Role: Extracts key funding attributes from article title and text  
  - Config: Attributes defined: company_name, funding_round, funding_amount (number), lead_investor, market, participating_investors, press_release_url, evaluation (number)  
  - Inputs: Merged article text and title  
  - Outputs: Structured JSON with funding details  
  - Edge cases: AI extraction errors, missing data  

- **Research URL**  
  - Type: Chain LLM (LangChain)  
  - Role: Finds official company website URL based on extracted company name  
  - Config: Prompt: "Find the website for this company: {{ company_name }}"  
  - Inputs: Output from Extract Structured Data or deep research  
  - Outputs: Text containing website URL  

- **Perplexity**  
  - Type: LM Chat OpenRouter  
  - Role: AI model used optionally for research or URL extraction  
  - Config: Model: perplexity/llama-3.1-sonar-small-128k-online  
  - Inputs: Text prompts  
  - Outputs: AI-generated text responses  

- **Extract URL**  
  - Type: Chain LLM (LangChain)  
  - Role: Parses the URL from AI-generated text (two-step approach for reliability)  
  - Config: Uses output parser for structured extraction  
  - Inputs: Text from Research URL or Claude 3.5 Haiku  
  - Outputs: Extracted website URL  

- **Collect Data**  
  - Type: Set  
  - Role: Aggregates all extracted fields into a single JSON object for Airtable  
  - Config: Maps fields from Extract Structured Data and Extract URL outputs  
  - Outputs: Unified data object  

- **Airtable (Funding Round Base)**  
  - Type: Airtable node  
  - Role: Creates a new record in the Funding Round Base with collected data  
  - Config: Base and table IDs set; auto-mapping fields like company_name, funding_amount, etc.  
  - Inputs: Data from Collect Data node  
  - Edge cases: Airtable API errors, rate limits, data validation errors  

---

#### 1.4 Deep Company Research (Subworkflow)

**Overview:**  
Executes a subworkflow to perform detailed company research using AI models (Perplexity or Jina Deep Search) based on the funding data, producing a comprehensive company profile.

**Nodes Involved:**  
- Route to Deep Research (Execute Workflow)  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Prompts (Set)  
- Deep Research (HTTP Request to Perplexity API)  
- JINA Deep Search (HTTP Request)  
- Pick data (Perplexity) (Set)  
- Pick data (jina) (Set)  
- Extract Structured Data (Chain LLM)  
- Extract Structured JSON (Output Parser Structured)  
- Auto-fixing Output Parser (Output Parser Autofixing)  
- Claude 3.5 Sonnet (LM Chat Anthropic)  
- Claude 3.5 Haiku (LM Chat Anthropic)  
- Write Results to Airtable (Company Deep Research Base)

**Node Details:**

- **Route to Deep Research**  
  - Type: Execute Workflow  
  - Role: Calls the subworkflow for deep company research  
  - Config: Passes company_name, funding_round, funding_amount as inputs  
  - Outputs: Receives detailed research data  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for subworkflow execution  
  - Config: Accepts inputs company_name, funding_amount, funding_round  

- **Prompts**  
  - Type: Set  
  - Role: Defines system and user prompts for deep research AI queries  
  - Config: User prompt requests comprehensive company info including background, business analysis, market position, growth, and additional context  
  - Outputs: Prompt texts for AI  

- **Deep Research**  
  - Type: HTTP Request  
  - Role: Calls Perplexity AI deep research API with prompts  
  - Config: POST to https://api.perplexity.ai/chat/completions with sonar-deep-research model, includes system and user prompts in JSON body, uses HTTP header auth  
  - Inputs: Prompts from Set node  
  - Outputs: AI research report and citations  
  - Edge cases: API errors, auth failures, rate limits  

- **JINA Deep Search**  
  - Type: HTTP Request  
  - Role: Alternative deep research via Jina API  
  - Config: POST to https://deepsearch.jina.ai/v1/chat/completions, sends system and user prompts, uses HTTP header auth  
  - Outputs: AI research report and visited URLs  

- **Pick data (Perplexity)**  
  - Type: Set  
  - Role: Extracts report text and citation links from Perplexity response  
  - Outputs: report (string), links (array)  

- **Pick data (jina)**  
  - Type: Set  
  - Role: Extracts report text and visited URLs from Jina response  
  - Outputs: report (string), links (array)  

- **Extract Structured Data**  
  - Type: Chain LLM  
  - Role: Parses AI research report and links into structured company data  
  - Config: Prompt instructs to only extract verifiable info  
  - Inputs: report and links from Pick data nodes  
  - Outputs: Structured JSON with detailed company info  

- **Extract Structured JSON**  
  - Type: Output Parser Structured  
  - Role: Parses complex JSON schema for company deep research data  
  - Config: Detailed JSON schema covering company profile, funding rounds, valuation, etc.  

- **Auto-fixing Output Parser**  
  - Type: Output Parser Autofixing  
  - Role: Automatically fixes JSON parsing errors in AI output for robustness  

- **Claude 3.5 Sonnet & Claude 3.5 Haiku**  
  - Type: LM Chat Anthropic  
  - Role: Alternative AI models used for structured data extraction and URL extraction  
  - Config: Anthropic API credentials required  

- **Write Results to Airtable (Company Deep Research Base)**  
  - Type: Airtable node  
  - Role: Creates a new record in the Company Deep Research Base with detailed company data  
  - Config: Maps all structured fields including CEO name, industry, valuation, previous rounds, source URLs, original report, etc.  
  - Edge cases: Airtable API errors, data validation issues  

---

### 3. Summary Table

| Node Name                          | Node Type                         | Functional Role                                | Input Node(s)                           | Output Node(s)                          | Sticky Note                                                                                              |
|-----------------------------------|----------------------------------|-----------------------------------------------|---------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                   | Manual start trigger                           | -                                     | Techcrunch (TC), Venturebeat (VB)     |                                                                                                        |
| Techcrunch (TC)                   | HTTP Request                    | Fetch TechCrunch news sitemap XML              | When clicking ‘Test workflow’          | Parse TC XML                          | ## TechCrunch & VentureBeat HTTP GET requests to fetch the latest articles from tech news sitemap feeds. |
| Venturebeat (VB)                  | HTTP Request                    | Fetch VentureBeat news sitemap XML             | When clicking ‘Test workflow’          | Parse VB XML                         | ## TechCrunch & VentureBeat HTTP GET requests to fetch the latest articles from tech news sitemap feeds. |
| Parse TC XML                     | XML                             | Parse TechCrunch XML to JSON                    | Techcrunch (TC)                       | Split TC Articles                    | ## Parse XML Converts XML data to structured JSON for easier processing of article metadata.            |
| Parse VB XML                    | XML                             | Parse VentureBeat XML to JSON                    | Venturebeat (VB)                     | Split VB Articles                   | ## Parse XML Converts XML data to structured JSON for easier processing of article metadata.            |
| Split TC Articles                | Split Out                      | Split TechCrunch feed into individual articles | Parse TC XML                        | Filter                             | ## Split Articles & Filter: Separates individual articles and filters to keep only the most relevant ones based on keywords (raised) |
| Split VB Articles               | Split Out                      | Split VentureBeat feed into individual articles| Parse VB XML                       | Filter1                            | ## Split Articles & Filter: Separates individual articles and filters to keep only the most relevant ones based on keywords (raised) |
| Filter                         | Filter                         | Filter TechCrunch articles for funding mentions| Split TC Articles                  | Get Funding Article HTML for scraping (TC) |                                                                                                        |
| Filter1                        | Filter                         | Filter VentureBeat articles for funding mentions| Split VB Articles                 | Get Funding Article HTML for scraping (VB) |                                                                                                        |
| Get Funding Article HTML for scraping (TC) | HTTP Request                    | Fetch full TechCrunch article HTML              | Filter                              | TC HTML Parser                     | ## Get Article Fetches the full article content for each relevant article in the feed.                  |
| Get Funding Article HTML for scraping (VB) | HTTP Request                    | Fetch full VentureBeat article HTML             | Filter1                             | VB HTML Parser                    | ## Get Article Fetches the full article content for each relevant article in the feed.                  |
| TC HTML Parser                 | HTML                            | Extract clean text and title from TechCrunch HTML| Get Funding Article HTML for scraping (TC) | Merge Extracted Data              | ## HTML Parser Extracts clean text content from the HTML articles for analysis.                         |
| VB HTML Parser                | HTML                            | Extract clean text and title from VentureBeat HTML| Get Funding Article HTML for scraping (VB) | Merge Extracted Data              | ## HTML Parser Extracts clean text content from the HTML articles for analysis.                         |
| Merge Extracted Data           | Merge                           | Combine articles from both sources into one dataset| TC HTML Parser, VB HTML Parser    | Extract Structured Data            | ## Merge Extracted Data Combines articles from multiple sources into a unified dataset for comprehensive analysis. |
| Extract Structured Data        | Information Extractor (LangChain)| Extract structured funding data from article text| Merge Extracted Data              | Research URL, Perplexity           | ## Extract Structured Data Identifies and structures key information from article text such as company names, funding details, and tech trends. |
| Research URL                  | Chain LLM (LangChain)           | Find official company website URL               | Extract Structured Data            | Extract URL                       | ## Research company website Uses perplexity to find the company website with search                    |
| Perplexity                   | LM Chat OpenRouter              | AI model for research and URL extraction        | Extract Structured Data            | Research URL                     |                                                                                                        |
| Extract URL                  | Chain LLM (LangChain)           | Parse website URL from AI text output            | Research URL, Claude 3.5 Haiku     | Collect Data                     | ## Extract URL Since perplexity uses Llama which is not great at structured output - 2 step approach for a more reliable run |
| Collect Data                 | Set                            | Aggregate extracted data for Airtable            | Extract URL, Extract Structured Data | Route to Deep Research, Airtable  | ## Collect data & write to airtable Collecting all data to pass on to detailed company research and store information in airtable |
| Airtable                    | Airtable                       | Store funding round data in Airtable              | Collect Data                     | -                                 | ## Airtable Base https://airtable.com/appYwSYZShjr8TN5r/shryOEdmJmZE5ROce                              |
| Route to Deep Research       | Execute Workflow               | Call subworkflow for detailed company research   | Collect Data                     | -                                 | ## Exectuted as a subworkflow                                                                           |
| When Executed by Another Workflow | Execute Workflow Trigger       | Subworkflow entry point for deep research         | -                               | Prompts                         |                                                                                                        |
| Prompts                     | Set                            | Define system and user prompts for deep research  | When Executed by Another Workflow | Deep Research                   |                                                                                                        |
| Deep Research               | HTTP Request                   | Call Perplexity AI deep research API               | Prompts                         | Pick data (Perplexity)           | ## Company Research Using Perplexity Deep Research  we can find more information about the company.     |
| JINA Deep Search            | HTTP Request                   | Alternative deep research via Jina API             | Prompts                         | Pick data (jina)                | ## Optional: Use jina Deep Search https://jina.ai/news/a-practical-guide-to-implementing-deepsearch-deepresearch |
| Pick data (Perplexity)      | Set                            | Extract report text and citations from Perplexity  | Deep Research                   | Extract Structured Data          |                                                                                                        |
| Pick data (jina)            | Set                            | Extract report text and URLs from Jina             | JINA Deep Search                | Extract Structured Data          |                                                                                                        |
| Extract Structured Data      | Chain LLM (LangChain)           | Parse AI research report into structured company data| Pick data (Perplexity), Pick data (jina) | Claude 3.5 Sonnet, Auto-fixing Output Parser |                                                                                                        |
| Extract Structured JSON      | Output Parser Structured       | Parse complex JSON schema for company deep research| Claude 3.5 Sonnet              | Auto-fixing Output Parser        |                                                                                                        |
| Auto-fixing Output Parser    | Output Parser Autofixing       | Fix JSON parsing errors in AI output                | Extract Structured JSON          | Extract Structured Data          |                                                                                                        |
| Claude 3.5 Sonnet           | LM Chat Anthropic              | AI model for structured data extraction             | Extract Structured Data          | Extract Structured JSON          |                                                                                                        |
| Claude 3.5 Haiku            | LM Chat Anthropic              | AI model for URL extraction                           | Extract Structured Data          | Extract URL                    |                                                                                                        |
| Write Results to Airtable   | Airtable                       | Store detailed company research data in Airtable    | Extract Structured Data          | -                               | ## Airtable Base https://airtable.com/appYwSYZShjr8TN5r/shryOEdmJmZE5ROce                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - Type: Manual Trigger  

2. **Add HTTP Request Nodes for News Sitemaps**  
   - Name: Techcrunch (TC)  
   - URL: https://techcrunch.com/news-sitemap.xml  
   - Method: GET  
   - Connect from Manual Trigger  

   - Name: Venturebeat (VB)  
   - URL: https://venturebeat.com/news-sitemap.xml  
   - Method: GET  
   - Connect from Manual Trigger  

3. **Add XML Parse Nodes**  
   - Name: Parse TC XML  
   - Input: Techcrunch (TC) output  
   - Default options  

   - Name: Parse VB XML  
   - Input: Venturebeat (VB) output  
   - Default options  

4. **Add Split Out Nodes to split articles**  
   - Name: Split TC Articles  
   - Field to split out: urlset.url  
   - Input: Parse TC XML output  

   - Name: Split VB Articles  
   - Field to split out: urlset.url  
   - Input: Parse VB XML output  

5. **Add Filter Nodes to keep articles mentioning funding**  
   - Name: Filter  
   - Condition: news:title contains "raise" (case sensitive)  
   - Input: Split TC Articles output  

   - Name: Filter1  
   - Condition: loc contains "raise"  
   - Input: Split VB Articles output  

6. **Add HTTP Request Nodes to fetch full article HTML**  
   - Name: Get Funding Article HTML for scraping (TC)  
   - URL: from filtered article loc field  
   - Input: Filter output  

   - Name: Get Funding Article HTML for scraping (VB)  
   - URL: from filtered article loc field  
   - Input: Filter1 output  

7. **Add HTML Parser Nodes to extract clean text and title**  
   - Name: TC HTML Parser  
   - Extract title: `.wp-block-post-title`  
   - Extract text: `.wp-block-post-content`  
   - Input: Get Funding Article HTML for scraping (TC) output  

   - Name: VB HTML Parser  
   - Extract title: `.article-title`  
   - Extract text: `#content`  
   - Input: Get Funding Article HTML for scraping (VB) output  

8. **Add Merge Node to combine articles**  
   - Name: Merge Extracted Data  
   - Inputs: TC HTML Parser and VB HTML Parser outputs  

9. **Add Information Extractor Node (LangChain)**  
   - Name: Extract Structured Data  
   - Input: Merge Extracted Data output  
   - Attributes: company_name, funding_round, funding_amount (number), lead_investor, market, participating_investors, press_release_url, evaluation (number)  

10. **Add Chain LLM Node to research company website URL**  
    - Name: Research URL  
    - Prompt: "Find the website for this company: {{ company_name }}"  
    - Input: Extract Structured Data output  

11. **Add Chain LLM Node to extract URL from AI text**  
    - Name: Extract URL  
    - Input: Research URL output  
    - Enable output parser for structured extraction  

12. **Add Set Node to collect all extracted data**  
    - Name: Collect Data  
    - Map fields from Extract Structured Data and Extract URL outputs  

13. **Add Airtable Node to store funding round data**  
    - Name: Airtable  
    - Base: Funding Round Base (appYwSYZShjr8TN5r)  
    - Table: Funding Round Base (tblQTIWUC8FBMF16F)  
    - Map fields accordingly  
    - Input: Collect Data output  
    - Configure Airtable Personal Access Token credentials  

14. **Add Execute Workflow Node to call deep research subworkflow**  
    - Name: Route to Deep Research  
    - Workflow ID: (subworkflow ID for deep research)  
    - Pass inputs: company_name, funding_round, funding_amount from Collect Data  

15. **Create Deep Research Subworkflow**  
    - Entry node: Execute Workflow Trigger  
      - Inputs: company_name (string), funding_amount (number), funding_round (string)  

    - Set Node: Prompts  
      - Define system_prompt and user_prompt with detailed research instructions  

    - HTTP Request Node: Deep Research (Perplexity API)  
      - POST to https://api.perplexity.ai/chat/completions  
      - Body: JSON with model sonar-deep-research and messages from Prompts  
      - Use HTTP Header Auth credentials for Perplexity  

    - HTTP Request Node: JINA Deep Search (optional alternative)  
      - POST to https://deepsearch.jina.ai/v1/chat/completions  
      - Use HTTP Header Auth credentials for Jina  

    - Set Nodes: Pick data (Perplexity) and Pick data (jina)  
      - Extract report text and links from respective responses  

    - Chain LLM Node: Extract Structured Data  
      - Input: report and links from Pick data nodes  
      - Prompt: Extract verifiable info only  

    - Output Parser Nodes: Extract Structured JSON and Auto-fixing Output Parser  
      - Parse complex JSON schema for company deep research data  

    - LM Chat Anthropic Nodes: Claude 3.5 Sonnet and Claude 3.5 Haiku  
      - For structured data extraction and URL extraction  

    - Airtable Node: Write Results to Airtable  
      - Base: Company Deep Research Base (appYwSYZShjr8TN5r)  
      - Table: Company Deep Research Base (tbltUvIthISpEbgUp)  
      - Map all detailed company fields  
      - Use Airtable Personal Access Token credentials  

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Using Perplexity via OpenRouter loses access to source URLs; direct API HTTP node preserves source citations.    | See workflow note on Perplexity API usage and header auth configuration                                |
| Header Auth configuration required for Perplexity and Jina API HTTP requests.                                    | https://docs.n8n.io/integrations/builtin/credentials/httprequest/#using-header-auth                     |
| Airtable Base structure for Funding Round and Company Deep Research provided with field definitions.             | https://airtable.com/appYwSYZShjr8TN5r/shryOEdmJmZE5ROce                                               |
| Optional use of Jina Deep Search for enhanced deep research capabilities.                                         | https://jina.ai/news/a-practical-guide-to-implementing-deepsearch-deepresearch                          |
| Workflow demonstrates advanced use of LangChain nodes for information extraction and output parsing.             |                                                                                                        |
| Demonstrates splitting and filtering large XML feeds, merging multi-source data, and chaining AI models.         |                                                                                                        |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.