Create Structured Company Profiles with Mistral AI, Firecrawl & Competitor Analysis

https://n8nworkflows.xyz/workflows/create-structured-company-profiles-with-mistral-ai--firecrawl---competitor-analysis-10097


# Create Structured Company Profiles with Mistral AI, Firecrawl & Competitor Analysis

### 1. Workflow Overview

This workflow automates the creation of detailed, structured company profiles by scraping and analyzing website data using AI models and specialized tools. It supports two scraping modes: **basic web scraping** for rapid extraction and **deep AI-based scraping** for comprehensive data gathering. The extracted data is then structured into rich company profiles, enriched with competitor information discovered through AI-powered web search, and stored reliably in both Supabase and Google Drive for easy access and further analysis.

The workflow is logically divided into these main blocks:

- **1.1 User Input Reception and Scraping Mode Selection:** Captures user input via a form, including the target website URL and scraping preferences, then routes the flow based on the selected scraping mode.

- **1.2 Data Extraction via Scraping Agents:** Executes either a basic web scraper or a deep AI-powered scraper (Firecrawl) to gather raw company data from the website.

- **1.3 AI-Powered Data Structuring and Extraction:** Processes the raw scraped data with Mistral AI language models and structured output parsers to generate well-defined company profiles.

- **1.4 Data Storage and Archiving:** Saves structured company data and related information into Supabase tables and archives JSON files in Google Drive folders.

- **1.5 Competitor Discovery and Enrichment:** Uses the Tavily AI agent to find competitors and enrich the company profile with competitor data stored in Supabase.

- **1.6 Auxiliary and Control Nodes:** Includes switches, no-op nodes, and sticky notes for flow control, documentation, and user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception and Scraping Mode Selection

**Overview:**  
This initial block gathers user input via a web form and directs the processing flow depending on the requested scraping type (basic or deep).

**Nodes Involved:**  
- On form submission  
- Switch

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; captures user inputs: Website URL, Scraping type (basic/deep), Competitor analysis option, Social media scraping option.  
  - Configuration: Form titled "profiler" with fields for URL and dropdowns for scraping type and options.  
  - Connections: Outputs to the Switch node.  
  - Edge Cases: Invalid or missing website URLs, unexpected form values may cause downstream failures.

- **Switch**  
  - Type: Switch Router  
  - Role: Routes workflow based on the "Scrapping type" field from the form.  
  - Configuration: Two outputs — "for crawlee" for basic scraping if "Scrapping type" equals "basic", and "crawlai" for deep scraping if "deep scraping" is selected.  
  - Connections: "crawlai" output connects to "MCP based scraper FIRECRAWLER" node; "for crawlee" output is unused in this workflow JSON (implies disabled or future extension).  
  - Edge Cases: Unrecognized scraping type leads to no route and halts processing.

---

#### 2.2 Data Extraction via Scraping Agents

**Overview:**  
Executes the appropriate scraping method based on user choice: a simple scraping agent or a comprehensive AI-based scraper leveraging Firecrawl MCP.

**Nodes Involved:**  
- MCP based scraper FIRECRAWLER  
- list tools (Firecrawl)  
- execute tools (Firecrawl)  
- Firecrawl tools1  
- Firecrawl list1  
- Mistral Cloud Chat Model2  
- Structured Output Parser1  
- basic web scraper (potentially dormant in this flow)  
- Mistral Cloud Chat Model5  
- Information Extractor1  
- Mistral Cloud Chat Model6

**Node Details:**

- **MCP based scraper FIRECRAWLER**  
  - Type: LangChain Agent Node  
  - Role: AI-driven scraper that uses Firecrawl tools to extract detailed company data from the website URL.  
  - Configuration: System message guides the AI to list and use Firecrawl tools iteratively, extract comprehensive company profile data including mission, services, team, social media links, blogs, CTAs, SEO keywords, etc. Output is structured JSON.  
  - Inputs: Website URL from form submission.  
  - Outputs: Feeds into Supabase storage and JSON file conversion nodes.  
  - Edge Cases: Firecrawl API rate limits, incomplete or noisy website data, network timeouts.

- **list tools, execute tools, Firecrawl tools1, Firecrawl list1**  
  - Type: MCP Client Tool Nodes  
  - Role: Interface nodes for listing and executing Firecrawl tools leveraged by the "MCP based scraper FIRECRAWLER" agent.  
  - Configuration: No parameters; rely on credentials for Firecrawl API access.  
  - Edge Cases: Authentication failures, API connectivity issues.

- **Mistral Cloud Chat Model2**  
  - Type: Language Model (Mistral AI)  
  - Role: Supports Firecrawl scraper agent by providing AI language processing capabilities.  
  - Configuration: Uses "mistral-small-latest" model.  
  - Edge Cases: Model API errors or latency.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the JSON-formatted company profile output from the Firecrawl AI scraper to ensure adherence to a predefined schema.  
  - Configuration: Uses a detailed JSON schema example defining company attributes like name, about summary, services, social links, testimonials, location, contact details, etc.  
  - Edge Cases: Parsing errors from malformed JSON or schema mismatches.

- **basic web scraper** (dormant in current flow)  
  - Type: LangChain Agent Node  
  - Role: Intended for simpler, faster web scraping mode; analyzes raw HTML/text to produce a company profile JSON.  
  - Configuration: Configured with a system message instructing brand analysis.  
  - Connections: Outputs to "Information Extractor1".  
  - Edge Cases: Limited extraction depth; may miss detailed info.

- **Mistral Cloud Chat Model5 & Mistral Cloud Chat Model6**  
  - Type: Language Model (Mistral AI)  
  - Role: Provide AI language processing for the basic scraper and information extraction steps.  
  - Edge Cases: API availability and response quality.

- **Information Extractor1**  
  - Type: LangChain Information Extractor  
  - Role: Extracts structured information from raw JSON output of the basic web scraper to map fields like company summary, mission, products, target audience, etc.  
  - Configuration: Uses a JSON schema example to guide extraction.  
  - Outputs: Sends structured data to Supabase storage and file conversion nodes.  
  - Edge Cases: Extraction errors on incomplete or ambiguous input data.

---

#### 2.3 Data Storage and Archiving

**Overview:**  
Saves extracted and structured company profiles and related data into Supabase tables and archives JSON files in Google Drive.

**Nodes Involved:**  
- Supabase  
- Supabase4  
- social media db  
- keywords  
- save to Supabase  
- Convert to File for storage  
- Convert to File for storage1  
- Google Drive2  
- save to Google Drive1  

**Node Details:**

- **Supabase**  
  - Type: Supabase Node  
  - Role: Inserts or updates the main company profile data into the "companies" table.  
  - Configuration: Maps structured output fields like company_name, mission, vision, values, services, social links, contact details, target audience, testimonials, blogs, SEO keywords, notable features, location, tone of voice, calls to action, slug, and brand story.  
  - Inputs: Output from "MCP based scraper FIRECRAWLER".  
  - Outputs: Connects to competitor search, social media db, and keywords nodes.  
  - Edge Cases: Database connectivity, duplicate entries, schema mismatches.

- **Supabase4**  
  - Type: Supabase Node  
  - Role: Inserts competitor data into the "competitors" table, linking to the parent company by company_id.  
  - Configuration: Fields include industry, location, website, notes, market leader flag, and competitor name.  
  - Inputs: Output from "find competitor" node.  
  - Edge Cases: Referential integrity errors if company_id missing.

- **social media db**  
  - Type: Supabase Node  
  - Role: Stores social media links for the company in the "social_links" table.  
  - Configuration: References company_id from Supabase main profile node.  
  - Edge Cases: Missing or malformed social media data.

- **keywords**  
  - Type: Supabase Node  
  - Role: Stores SEO keywords associated with the company in the "seo_keywords" table.  
  - Configuration: Uses company_id and keywords extracted from profile.  
  - Edge Cases: Empty or duplicate keywords.

- **save to Supabase**  
  - Type: Supabase Node  
  - Role: Stores basic company profile information into "company_basicprofiles" table using data extracted by the basic scraper.  
  - Configuration: Fields include name, summary, mission, vision, target audience, founding story, digital presence, visual identity, core values, products/services, and unique selling points.  
  - Edge Cases: Data mismatch or incomplete profiles.

- **Convert to File for storage & Convert to File for storage1**  
  - Type: Convert to File Node  
  - Role: Converts extracted JSON profiles into JSON files for archival.  
  - Configuration: Filename uses company name or extracted NAME field.  
  - Outputs: Connects to Google Drive nodes for uploading.  
  - Edge Cases: Large JSON objects may cause timeouts.

- **Google Drive2 & save to Google Drive1**  
  - Type: Google Drive Node  
  - Role: Uploads JSON files to designated folders in Google Drive ("Companies_profiles").  
  - Configuration: Uses folder IDs and drive selection "My Drive".  
  - Edge Cases: API quota exceeded, permission errors.

---

#### 2.4 Competitor Discovery and Enrichment

**Overview:**  
Discovers competitors of the profiled company using the Tavily AI agent, enriches the profile with competitor information, and stores it in Supabase.

**Nodes Involved:**  
- find competitor  
- Structured Output Parser  
- Mistral Cloud Chat Model3  
- Supabase4  
- Web Search tool

**Node Details:**

- **find competitor**  
  - Type: LangChain Agent Node  
  - Role: Uses Tavily search and Crunchbase data to identify competitors for the company based on its name and headquarters location.  
  - Configuration: System message instructs the agent to list tools, select appropriate ones, perform a web search, and return competitor data in structured format.  
  - Inputs: Company name and location from Supabase profile node.  
  - Outputs: Passes structured competitor info to Supabase4.  
  - Edge Cases: Search API failures, incomplete competitor data, ambiguous matches.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses competitor data into a JSON schema with fields like company name, address, industry, website, competitor notes, market leader flag, and additional info.  
  - Edge Cases: Parser errors on poorly formatted data.

- **Mistral Cloud Chat Model3**  
  - Type: Language Model (Mistral AI)  
  - Role: Supports the competitor finding agent with natural language understanding and response generation.  
  - Edge Cases: Response latency or API limits.

- **Web Search tool**  
  - Type: LangChain HTTP Request Tool Node  
  - Role: Performs API calls to Tavily search engine for acquiring competitor data.  
  - Configuration: POST request with query and parameters; requires Bearer token for authorization.  
  - Edge Cases: API key validity, network errors.

---

#### 2.5 Auxiliary and Control Nodes

**Overview:**  
Supporting nodes for flow control, no-operation placeholders, and workflow documentation.

**Nodes Involved:**  
- No Operation, do nothing  
- Sticky Notes (multiple)

**Node Details:**

- **No Operation, do nothing**  
  - Type: NoOp Node  
  - Role: Acts as a terminal or placeholder node for branches that do not proceed further in the workflow.  
  - Edge Cases: None; safe to ignore.

- **Sticky Notes**  
  - Provide important documentation, instructions, and links:  
    - Basic web scraper installation guide (crawl and scrape node).  
    - Information about Tavily MCP tool for competitor finding.  
    - Firecrawl MCP deep scraping guide with external link: https://www.firecrawl.dev/  
    - User instructions on form usage and scraper switching.  
    - Workflow overview and benefits summary.  
  - Role: Facilitate understanding and maintenance by human operators.

---

### 3. Summary Table

| Node Name                  | Node Type                                          | Functional Role                                          | Input Node(s)                | Output Node(s)                             | Sticky Note                                                                                         |
|----------------------------|---------------------------------------------------|----------------------------------------------------------|------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                                      | Captures user input form for URL and scraping options    |                              | Switch                                     | ## USER FILL FORM ENTER WEBSITE URL SELECT TYPE OF SCRAPING 1 SIMPLE 2 DEEP                         |
| Switch                     | Switch                                            | Routes flow based on scraping type                        | On form submission            | MCP based scraper FIRECRAWLER               | ## SWITCH BETWEEN SCRAPPER BASED ON USER SELECTION                                                 |
| MCP based scraper FIRECRAWLER | LangChain Agent Node                              | Deep AI scraping via Firecrawl tools                      | Switch (crawlai output)       | Supabase, Convert to File for storage      | FIRECRAWL MCP (DEEP AI BASED SCRAPING) [firecrawl guide ](https://www.firecrawl.dev/)               |
| list tools                 | MCP Client Tool                                   | Lists available Firecrawl tools                           |                              |                                              | TAVILY MCP (COMPETITORS FINDING THROUGH SPECIALIZED SEARCH                                         |
| execute tools              | MCP Client Tool                                   | Executes Firecrawl tools                                  |                              |                                              | TAVILY MCP (COMPETITORS FINDING THROUGH SPECIALIZED SEARCH                                         |
| Firecrawl tools1           | MCP Client Tool                                   | Firecrawl support tool                                    |                              |                                              | FIRECRAWL MCP (DEEP AI BASED SCRAPING) [firecrawl guide ](https://www.firecrawl.dev/)               |
| Firecrawl list1            | MCP Client Tool                                   | Firecrawl support tool                                    |                              |                                              | FIRECRAWL MCP (DEEP AI BASED SCRAPING) [firecrawl guide ](https://www.firecrawl.dev/)               |
| Mistral Cloud Chat Model2  | LangChain LM Chat Model                           | AI language model used by Firecrawl scraper               |                              | MCP based scraper FIRECRAWLER               | FIRECRAWL MCP (DEEP AI BASED SCRAPING) [firecrawl guide ](https://www.firecrawl.dev/)               |
| Structured Output Parser1  | LangChain Structured Output Parser                | Parses Firecrawl scraper output into structured JSON      | MCP based scraper FIRECRAWLER  | MCP based scraper FIRECRAWLER               | FIRECRAWL MCP (DEEP AI BASED SCRAPING) [firecrawl guide ](https://www.firecrawl.dev/)               |
| Supabase                   | Supabase Node                                     | Stores main company profile data                          | MCP based scraper FIRECRAWLER  | find competitor, social media db, keywords |                                                                                                   |
| social media db            | Supabase Node                                     | Stores social media links                                 | Supabase                      | No Operation                               |                                                                                                   |
| keywords                   | Supabase Node                                     | Stores SEO keywords                                      | Supabase                      | No Operation                               |                                                                                                   |
| find competitor            | LangChain Agent Node                              | Finds company competitors using Tavily & Crunchbase      | Supabase                     | Supabase4                                  | TAVILY MCP (COMPETITORS FINDING THROUGH SPECIALIZED SEARCH                                         |
| Structured Output Parser   | LangChain Structured Output Parser                | Parses competitor data                                    | find competitor              | find competitor                            | TAVILY MCP (COMPETITORS FINDING THROUGH SPECIALIZED SEARCH                                         |
| Mistral Cloud Chat Model3  | LangChain LM Chat Model                           | AI model supporting competitor finder                     |                              | find competitor                            | TAVILY MCP (COMPETITORS FINDING THROUGH SPECIALIZED SEARCH                                         |
| Web Search tool            | LangChain HTTP Request Tool                       | Performs Tavily web searches                              | find competitor (ai_tool)     | MCP based scraper FIRECRAWLER              | TAVILY MCP (COMPETITORS FINDING THROUGH SPECIALIZED SEARCH                                         |
| Supabase4                  | Supabase Node                                     | Stores competitor data                                   | find competitor              | No Operation                               |                                                                                                   |
| Convert to File for storage| Convert to File                                   | Converts Firecrawl JSON output to file                   | MCP based scraper FIRECRAWLER  | Google Drive2                              |                                                                                                   |
| Google Drive2              | Google Drive                                     | Uploads JSON file to Google Drive folder                  | Convert to File for storage   |                                            |                                                                                                   |
| basic web scraper          | LangChain Agent Node                              | Basic web scraping agent (currently unused)               |                              | Information Extractor1                      | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| Mistral Cloud Chat Model5  | LangChain LM Chat Model                           | AI model for basic web scraper                            |                              | basic web scraper                          | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| Information Extractor1     | LangChain Information Extractor                   | Extracts structured info from basic scraper output        | basic web scraper             | save to Supabase, Convert to File for storage1 | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| Mistral Cloud Chat Model6  | LangChain LM Chat Model                           | Supports Information Extractor1                           |                              | Information Extractor1                     | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| save to Supabase           | Supabase Node                                     | Stores basic profile data from basic scraper             | Information Extractor1        | No Operation                               | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| Convert to File for storage1| Convert to File                                  | Converts basic scraper JSON to file                       | Information Extractor1        | save to Google Drive1                      | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| save to Google Drive1      | Google Drive                                     | Uploads basic profile JSON file to Drive                  | Convert to File for storage1  | No Operation                               | BASIC WEB SCRAPER [Installation Guide crawl and scrape node](https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape) |
| No Operation, do nothing   | No Operation                                     | Placeholder terminal node                                 | keywords, Supabase4, save to Supabase, save to Google Drive1 | |                                                                                                   |
| Sticky Note                | Sticky Note                                      | Documentation and user guidance                           |                              |                                            |                                                                                                   |
| Sticky Note1               | Sticky Note                                      | Documentation and user guidance                           |                              |                                            |                                                                                                   |
| Sticky Note2               | Sticky Note                                      | Documentation and user guidance                           |                              |                                            |                                                                                                   |
| Sticky Note3               | Sticky Note                                      | Documentation and user guidance                           |                              |                                            |                                                                                                   |
| Sticky Note4               | Sticky Note                                      | Documentation and user guidance                           |                              |                                            |                                                                                                   |
| Sticky Note5               | Sticky Note                                      | Documentation and user guidance                           |                              |                                            |                                                                                                   |
| Sticky Note7               | Sticky Note                                      | Workflow overview and benefits summary                    |                              |                                            |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission")**  
   - Set form title: "profiler"  
   - Add fields:  
     - "Website url" (text)  
     - "Scrapping type" (dropdown: options "basic", "deep scraping")  
     - "Competitor anaylsis" (dropdown: "YES", "NO")  
     - "Social Media scraping" (dropdown: "YES", "NO")  
   - Configure webhook and enable.

2. **Add a Switch Node ("Switch")**  
   - Route based on `{{$json["Scrapping type "]}}` field from form:  
     - If equals "basic" → output "for crawlee"  
     - If equals "deep scraping" → output "crawlai"  
   - Connect "On form submission" main output to "Switch" input.

3. **Set up Deep Scraper Branch:**  
   - Add MCP Client Tool nodes: "list tools", "execute tools", "Firecrawl tools1", "Firecrawl list1"  
   - Add Mistral Cloud Chat Model node ("Mistral Cloud Chat Model2") with "mistral-small-latest" model.  
   - Add LangChain Agent node named "MCP based scraper FIRECRAWLER":  
     - Input: Website URL from form (`{{$json["Website url"]}}`)  
     - System prompt instructs to use Firecrawl tools iteratively to extract detailed company profile info, returning structured JSON.  
     - Set max iterations to 30.  
     - Connect "Switch" crawlai output to this node.  
     - Link LangChain LM node and MCP Client Tool nodes as required.

4. **Parse Deep Scraper Output:**  
   - Add a Structured Output Parser node ("Structured Output Parser1")  
   - Provide detailed JSON schema for company profile fields.  
   - Connect output from "MCP based scraper FIRECRAWLER" to this parser.

5. **Store Deep Scraper Data:**  
   - Add Supabase node ("Supabase") configured to insert into "companies" table with fields mapped from parsed output.  
   - Credentials: Configure Supabase credentials with appropriate project access and table permissions.  
   - Connect Structured Output Parser output to this node.

6. **Store Social Media Links and SEO Keywords:**  
   - Add Supabase nodes "social media db" (for "social_links" table) and "keywords" (for "seo_keywords" table), referencing company_id from Supabase insertion.  
   - Connect "Supabase" node output to these nodes.

7. **Archive Deep Scraper Data:**  
   - Add "Convert to File for storage" node to convert JSON output from deep scraper to JSON file named after company.  
   - Add Google Drive node ("Google Drive2") to upload this JSON file to a designated folder (e.g., "Companies_profiles").  
   - Connect conversion node output to Google Drive upload.

8. **Set up Competitor Discovery:**  
   - Add LangChain Agent node ("find competitor")  
     - Input: company name and headquarters location from Supabase.  
     - System prompt instructs to use Tavily tools and Crunchbase data to find comparable competitors.  
   - Use Mistral Cloud Chat Model3 as LM support.  
   - Add Structured Output Parser node ("Structured Output Parser") for competitor data with its JSON schema.  
   - Add Supabase node ("Supabase4") to insert competitor info into "competitors" table, linking by company_id from main profile.  
   - Add LangChain HTTP Request node ("Web Search tool") configured for Tavily API with required headers and authorization token.  
   - Connect nodes accordingly: "Supabase" → "find competitor" → "Structured Output Parser" → "Supabase4".

9. **Basic Scraper Branch (Optional / Future Use):**  
   - Add LangChain Agent node ("basic web scraper") configured for brand analysis of raw website text.  
   - Add Mistral Cloud Chat Model5 as LM support.  
   - Add Information Extractor node ("Information Extractor1") with appropriate JSON schema to extract detailed structured data.  
   - Use Mistral Cloud Chat Model6 as LM for extractor.  
   - Store results in Supabase table "company_basicprofiles" using "save to Supabase" node.  
   - Convert output to JSON file using "Convert to File for storage1".  
   - Upload to Google Drive using "save to Google Drive1".  
   - Connect from "Switch" node "for crawlee" output when implementing.

10. **Add No Operation Nodes:**  
    - Place NoOp nodes after terminal outputs where no further action is required to properly terminate branches.

11. **Add Sticky Notes:**  
    - Insert sticky notes at strategic positions with user guidance, installation links, and workflow overview for maintainers.

12. **Credential Setup:**  
    - Configure Supabase credentials with access to required tables ("companies", "competitors", "social_links", "seo_keywords", "company_basicprofiles").  
    - Configure Google Drive credentials with access to the target Drive and folder.  
    - Configure Firecrawl MCP credentials for API access.  
    - Configure Tavily API key in HTTP Request node headers.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| BASIC WEB SCRAPER Installation Guide for crawl and scrape node                                     | https://www.npmjs.com/package/n8n-nodes-crawl-and-scrape                                         |
| FIRECRAWL MCP (Deep AI Based Scraping) guide                                                      | https://www.firecrawl.dev/                                                                        |
| TAVILY MCP for Competitor Finding through specialized search                                      | (Embedded in sticky notes; also requires API authorization in "Web Search tool" node)            |
| Workflow Overview: AI Website Scraping & Company Intelligence Workflow with dual scraping modes    | Sticky Note7 in workflow provides detailed textual overview and core flow explanation            |

---

This documentation fully covers the workflow’s structure, logic, node configurations, and reproduction steps, enabling advanced users or AI agents to understand, reproduce, and maintain it effectively.