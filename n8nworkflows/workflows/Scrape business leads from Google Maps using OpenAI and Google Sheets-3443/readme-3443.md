Scrape business leads from Google Maps using OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/scrape-business-leads-from-google-maps-using-openai-and-google-sheets-3443


# Scrape business leads from Google Maps using OpenAI and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of detailed business leads from Google Maps, enriches the data by crawling associated business websites, and stores the structured results in Google Sheets. It is designed primarily for sales teams, marketers, entrepreneurs, and researchers who need efficient, accurate, and scalable lead generation and market analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and AI Processing:** Receives user input (via chat or subworkflow trigger), processes it with an AI agent that orchestrates the lead extraction logic.
- **1.2 Google Maps Data Extraction:** Uses a dedicated subworkflow and API to scrape business listings from Google Maps based on search parameters.
- **1.3 Website Content Crawling:** Extracts detailed content from business websites linked in the Google Maps data to enrich lead information.
- **1.4 Data Aggregation and Storage:** Aggregates and stores the extracted business and website data into Google Sheets for easy access and further analysis.
- **1.5 Fallback and Enrichment:** Uses Google Search API as a fallback to enrich or complete missing data when scraping results are incomplete.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI Processing

**Overview:**  
This block handles user input either via chat messages or direct subworkflow triggers. It uses an AI agent to interpret the input, manage the scraping logic, and coordinate calls to scraping tools and data storage.

**Nodes Involved:**  
- Trigger - When User Sends Message  
- Trigger - On Subworkflow Start  
- AI Agent - Lead Collection  
- GPT-4o - Generate & Process Requests  
- Memory - Track Recent Context  

**Node Details:**

- **Trigger - When User Sends Message**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point for user chat input to start lead extraction  
  - Configuration: Webhook-based trigger listening for incoming chat messages  
  - Inputs: External user messages  
  - Outputs: Connects to AI Agent - Lead Collection  
  - Edge Cases: Webhook failures, malformed messages  

- **Trigger - On Subworkflow Start**  
  - Type: Execute Workflow Trigger  
  - Role: Allows external systems or manual triggers to start the workflow with JSON input parameters (search term, city, state/county, country code)  
  - Configuration: Predefined JSON example input for testing  
  - Inputs: JSON data with search parameters  
  - Outputs: Connects to Scrape Google Maps (via Apify) node  
  - Edge Cases: Invalid or incomplete JSON input  

- **AI Agent - Lead Collection**  
  - Type: LangChain Agent  
  - Role: Central orchestrator interpreting user intent, managing scraping calls, and handling data retrieval/storage logic  
  - Configuration: Complex system prompt defining tasks, constraints, ethical guidelines, and interaction rules for lead extraction  
  - Key Expressions: Uses AI memory and tool integrations to decide when to scrape, when to query Google Sheets, and when to fallback to internet search  
  - Inputs: User messages or AI-generated prompts  
  - Outputs: Calls to GPT-4o, scraping tools, and fallback nodes  
  - Edge Cases: AI prompt failures, incomplete data handling, API rate limits  

- **GPT-4o - Generate & Process Requests**  
  - Type: OpenAI Chat Completion (gpt-4o-mini)  
  - Role: Generates natural language responses and processes AI agent requests  
  - Configuration: Uses OpenAI API credentials, model set to gpt-4o-mini  
  - Inputs: Prompts from AI Agent  
  - Outputs: Responses back to AI Agent  
  - Edge Cases: API quota exceeded, network timeouts  

- **Memory - Track Recent Context**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains recent conversational context (up to 50 messages) for AI agent continuity  
  - Configuration: Context window length set to 50  
  - Inputs: AI Agent outputs  
  - Outputs: AI Agent inputs  
  - Edge Cases: Memory overflow, context loss  

---

#### 2.2 Google Maps Data Extraction

**Overview:**  
This block performs the actual scraping of business listings from Google Maps using an Apify actor via HTTP requests. It extracts business names, addresses, phones, websites, emails, and other contact details based on user-defined search parameters.

**Nodes Involved:**  
- Scrape Google Maps (via Apify)  
- Save Extracted Data to Google Sheets  
- Aggregate Business Listings  
- Tool - Scrape Google Maps Business Data (LangChain Tool Workflow)  
- Sticky Note1 (Documentation)  

**Node Details:**

- **Scrape Google Maps (via Apify)**  
  - Type: HTTP Request  
  - Role: Calls Apify Google Maps Scraper API synchronously to get business data  
  - Configuration: POST request with JSON body containing city, countryCode, locationQuery, max places per search (5), search strings array, and skipClosedPlaces flag  
  - Headers: Content-Type application/json, Authorization Bearer token (needs user API key)  
  - Inputs: JSON parameters from trigger or AI agent  
  - Outputs: Raw scraped business data to Save Extracted Data to Google Sheets  
  - Edge Cases: API authentication errors, rate limits, empty results, malformed parameters  

- **Save Extracted Data to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Appends scraped business data to a specified Google Sheet document and sheet name  
  - Configuration: Append operation, documentId and sheetName set via expressions or credentials  
  - Inputs: Data from Scrape Google Maps node  
  - Outputs: Connects to Aggregate Business Listings  
  - Edge Cases: Google Sheets API quota exceeded, permission errors, invalid sheet IDs  

- **Aggregate Business Listings**  
  - Type: Aggregate Node  
  - Role: Aggregates all appended business listings into a single dataset for further processing or reporting  
  - Configuration: Aggregate all item data  
  - Inputs: Data from Save Extracted Data to Google Sheets  
  - Outputs: None (end of this sub-block)  
  - Edge Cases: Empty input data  

- **Tool - Scrape Google Maps Business Data**  
  - Type: LangChain Tool Workflow  
  - Role: AI agent interface to invoke the Google Maps scraping subworkflow with parameters (city, search, countryCode, state/county)  
  - Configuration: Points to subworkflow ID "9rD7iD6sbXqDX44S" with defined inputs mapped from AI variables  
  - Inputs: AI agent calls with parameters  
  - Outputs: Data back to AI agent  
  - Edge Cases: Subworkflow failures, parameter mismatches  

- **Sticky Note1**  
  - Purpose: Documentation explaining the Google Maps Extractor subworkflow, its purpose, and input parameters  
  - Content: Describes automation of business lead collection by search term, city, region, and country code  

---

#### 2.3 Website Content Crawling

**Overview:**  
This block enriches business leads by crawling their websites to extract readable, structured content. It uses an Apify Website Content Crawler actor to fetch and clean website data, which is then stored in Google Sheets.

**Nodes Involved:**  
- Scrape Website Content (via Apify)  
- Save Website Data to Google Sheets  
- Aggregate Website Content  
- Tool - Crawl Business Website (LangChain Tool Workflow)  
- Sticky Note2 (Documentation)  

**Node Details:**

- **Scrape Website Content (via Apify)**  
  - Type: HTTP Request  
  - Role: Calls Apify Website Content Crawler API synchronously to extract text and markdown content from business URLs  
  - Configuration: POST request with JSON body specifying crawler options such as CSS selectors to remove, iframe expansion, proxy usage, and start URLs dynamically set from input query  
  - Headers: Content-Type application/json, Authorization Bearer token (user API key)  
  - Inputs: URLs passed from AI agent or previous scraping results  
  - Outputs: Raw website content data to Save Website Data to Google Sheets  
  - Edge Cases: Website blocking, proxy failures, invalid URLs, timeouts  

- **Save Website Data to Google Sheets**  
  - Type: Google Sheets Node  
  - Role: Appends extracted website content data (URL, crawl metadata, markdown, text) to a dedicated Google Sheet tab  
  - Configuration: Append operation, specific documentId and sheetName set to "MYWEBBASE" tab  
  - Inputs: Data from Scrape Website Content node  
  - Outputs: Connects to Aggregate Website Content  
  - Edge Cases: API errors, permission issues  

- **Aggregate Website Content**  
  - Type: Aggregate Node  
  - Role: Aggregates all website content entries into a single dataset for reporting or further AI processing  
  - Configuration: Aggregate all item data  
  - Inputs: Data from Save Website Data to Google Sheets  
  - Outputs: None (end of this sub-block)  
  - Edge Cases: Empty input data  

- **Tool - Crawl Business Website**  
  - Type: LangChain Tool Workflow  
  - Role: AI agent interface to invoke the Website Content Crawler subworkflow  
  - Configuration: Points to subworkflow ID "I7KceT8Mg1lW7BW4" with no explicit inputs defined (dynamic URL input expected)  
  - Inputs: URLs from AI agent  
  - Outputs: Data back to AI agent  
  - Edge Cases: Subworkflow failures, missing URL input  

- **Sticky Note2**  
  - Purpose: Documentation describing the Website Content Crawler subworkflow and its role in enriching lead data with detailed website content  

---

#### 2.4 Data Aggregation and Storage

**Overview:**  
This block consolidates and stores all extracted data into Google Sheets, ensuring organized and accessible lead information for users.

**Nodes Involved:**  
- Save Extracted Data to Google Sheets  
- Aggregate Business Listings  
- Save Website Data to Google Sheets  
- Aggregate Website Content  

**Node Details:**  
(Details covered above in respective blocks 2.2 and 2.3)

---

#### 2.5 Fallback and Enrichment

**Overview:**  
When Google Maps scraping returns incomplete or missing data, this block uses the SerpAPI Google Search tool to enrich or complete the lead information.

**Nodes Involved:**  
- Fallback - Enrich with Google Search  

**Node Details:**

- **Fallback - Enrich with Google Search**  
  - Type: LangChain SerpAPI Tool  
  - Role: Performs internet search queries to supplement missing business data  
  - Configuration: Uses SerpAPI credentials, no additional parameters specified (dynamic queries expected)  
  - Inputs: AI agent calls when scraping is incomplete or fails  
  - Outputs: Returns enriched data to AI Agent - Lead Collection  
  - Edge Cases: API limits, inaccurate search results, network issues  

---

### 3. Summary Table

| Node Name                        | Node Type                         | Functional Role                                  | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                      |
|---------------------------------|----------------------------------|-------------------------------------------------|------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Trigger - When User Sends Message| LangChain Chat Trigger            | Entry point for user chat input                  | External webhook                   | AI Agent - Lead Collection         |                                                                                                |
| AI Agent - Lead Collection       | LangChain Agent                  | Orchestrates lead extraction and data handling  | Trigger - When User Sends Message  | GPT-4o, Tool - Scrape Google Maps Business Data, Tool - Crawl Business Website, Fallback - Enrich with Google Search |                                                                                                |
| GPT-4o - Generate & Process Requests | OpenAI Chat Completion (gpt-4o-mini) | Generates AI responses and processes requests    | AI Agent - Lead Collection         | AI Agent - Lead Collection         |                                                                                                |
| Memory - Track Recent Context    | LangChain Memory Buffer Window   | Maintains conversational context                 | AI Agent - Lead Collection         | AI Agent - Lead Collection         |                                                                                                |
| Tool - Scrape Google Maps Business Data | LangChain Tool Workflow          | Invokes Google Maps scraping subworkflow         | AI Agent - Lead Collection         | AI Agent - Lead Collection         |                                                                                                |
| Scrape Google Maps (via Apify)  | HTTP Request                    | Calls Apify Google Maps Scraper API               | Trigger - On Subworkflow Start     | Save Extracted Data to Google Sheets |                                                                                                |
| Save Extracted Data to Google Sheets | Google Sheets                   | Stores scraped Google Maps business data          | Scrape Google Maps (via Apify)     | Aggregate Business Listings        |                                                                                                |
| Aggregate Business Listings      | Aggregate Node                  | Aggregates business listings data                  | Save Extracted Data to Google Sheets | None                             |                                                                                                |
| Tool - Crawl Business Website    | LangChain Tool Workflow          | Invokes Website Content Crawler subworkflow       | AI Agent - Lead Collection         | AI Agent - Lead Collection         |                                                                                                |
| Scrape Website Content (via Apify) | HTTP Request                    | Calls Apify Website Content Crawler API            | AI Agent - Lead Collection (via Tool) | Save Website Data to Google Sheets |                                                                                                |
| Save Website Data to Google Sheets | Google Sheets                   | Stores extracted website content                    | Scrape Website Content (via Apify) | Aggregate Website Content          |                                                                                                |
| Aggregate Website Content        | Aggregate Node                  | Aggregates website content data                     | Save Website Data to Google Sheets | None                             |                                                                                                |
| Fallback - Enrich with Google Search | LangChain SerpAPI Tool           | Performs internet search to enrich incomplete data | AI Agent - Lead Collection         | AI Agent - Lead Collection         |                                                                                                |
| Trigger - On Subworkflow Start   | Execute Workflow Trigger         | Allows external JSON input to start scraping       | External JSON input                | Scrape Google Maps (via Apify)     |                                                                                                |
| Sticky Note                     | Sticky Note                     | Workflow overview and dependencies documentation  | None                             | None                             | # AI-Powered Lead Generation Workflow\n\nDependencies: OpenAI API, Google Sheets API, Apify Actors, SerpAPI\nGuide: https://automatisation.notion.site/GOOGLE-MAPS-SCRAPER-1cc3d6550fd98005a99cea02986e7b05 |
| Sticky Note1                    | Sticky Note                     | Google Maps Extractor subworkflow documentation    | None                             | None                             | # 📍 Google Maps Extractor Subworkflow\nPurpose: Automates business lead collection by search term, city, region, country code |
| Sticky Note2                    | Sticky Note                     | Website Content Crawler subworkflow documentation  | None                             | None                             | # 🌐 Website Content Crawler Subworkflow\nPurpose: Extracts detailed website content to enrich leads |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **LangChain Chat Trigger** node named "Trigger - When User Sends Message" to receive chat inputs.
   - Add an **Execute Workflow Trigger** node named "Trigger - On Subworkflow Start" with a JSON example input containing keys: `search`, `city`, `state/county`, `countryCode`.

2. **Add AI Agent Node:**
   - Add a **LangChain Agent** node named "AI Agent - Lead Collection".
   - Configure with a detailed system prompt that defines the lead extraction task, ethical guidelines, constraints, and interaction rules.
   - Connect "Trigger - When User Sends Message" to this node.
   - Connect "Memory - Track Recent Context" node to maintain conversation context.
   - Connect "GPT-4o - Generate & Process Requests" node for language model processing.
   - Configure OpenAI credentials for GPT-4o node with model "gpt-4o-mini".

3. **Set up Memory Node:**
   - Add **LangChain Memory Buffer Window** node named "Memory - Track Recent Context".
   - Set context window length to 50.
   - Connect it bi-directionally with the AI Agent node.

4. **Configure GPT-4o Node:**
   - Add **LangChain LM Chat OpenAI** node named "GPT-4o - Generate & Process Requests".
   - Set model to "gpt-4o-mini".
   - Provide OpenAI API credentials.
   - Connect input from AI Agent and output back to AI Agent.

5. **Add Google Maps Scraper Subworkflow:**
   - Create or import a subworkflow for Google Maps scraping (ID: "9rD7iD6sbXqDX44S").
   - Add a **LangChain Tool Workflow** node named "Tool - Scrape Google Maps Business Data".
   - Configure inputs: `city`, `search`, `countryCode`, `state/county` mapped from AI agent variables.
   - Connect AI Agent node output to this tool node.
   - Inside the subworkflow:
     - Add an **HTTP Request** node to call Apify Google Maps Scraper API.
     - Configure POST request with JSON body including city, countryCode, locationQuery, maxCrawledPlacesPerSearch (default 5), searchStringsArray, skipClosedPlaces false.
     - Add **Google Sheets** node to append scraped data to a specified sheet.
     - Add **Aggregate** node to consolidate data.
     - Ensure API keys and Google Sheets OAuth2 credentials are set.

6. **Add Website Content Crawler Subworkflow:**
   - Create or import a subworkflow for website crawling (ID: "I7KceT8Mg1lW7BW4").
   - Add a **LangChain Tool Workflow** node named "Tool - Crawl Business Website".
   - Connect AI Agent node output to this tool node.
   - Inside the subworkflow:
     - Add an **HTTP Request** node to call Apify Website Content Crawler API.
     - Configure POST request with JSON body specifying crawler options and dynamic start URL.
     - Add **Google Sheets** node to append website content data to a designated sheet/tab.
     - Add **Aggregate** node to consolidate website content.
     - Set API keys and Google Sheets OAuth2 credentials.

7. **Add Fallback Enrichment Node:**
   - Add a **LangChain SerpAPI Tool** node named "Fallback - Enrich with Google Search".
   - Configure with SerpAPI credentials.
   - Connect AI Agent node to this fallback node for use when scraping is incomplete.

8. **Connect Data Flows:**
   - Connect "Trigger - On Subworkflow Start" to "Scrape Google Maps (via Apify)" HTTP request node.
   - Connect "Scrape Google Maps (via Apify)" to "Save Extracted Data to Google Sheets".
   - Connect "Save Extracted Data to Google Sheets" to "Aggregate Business Listings".
   - Connect "Scrape Website Content (via Apify)" to "Save Website Data to Google Sheets".
   - Connect "Save Website Data to Google Sheets" to "Aggregate Website Content".

9. **Set Credentials:**
   - Configure OpenAI API credentials for GPT nodes.
   - Configure Google Sheets OAuth2 credentials with access to target spreadsheets.
   - Configure Apify API tokens for Google Maps Scraper and Website Content Crawler HTTP requests.
   - Configure SerpAPI credentials for fallback enrichment.

10. **Add Sticky Notes:**
    - Add descriptive sticky notes for workflow overview, Google Maps Extractor subworkflow, and Website Content Crawler subworkflow for documentation purposes.

11. **Test Workflow:**
    - Test chat trigger with sample queries.
    - Test subworkflow trigger with JSON input.
    - Verify data is scraped, enriched, and stored correctly.
    - Monitor for errors such as API limits, invalid inputs, or network issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| AI-Powered Lead Generation Workflow dependencies include OpenAI API, Google Sheets API, Apify Actors, and SerpAPI | Workflow Sticky Note with overview and dependencies                                                             |
| External Setup Guide available on Notion                                                                      | https://automatisation.notion.site/GOOGLE-MAPS-SCRAPER-1cc3d6550fd98005a99cea02986e7b05?pvs=4                    |
| Demo Video for workflow usage                                                                                  | https://www.youtube.com/watch?v=DoBRufiwElU                                                                      |
| Ethical guidelines enforced: only public professional data extracted, no personal/sensitive data stored         | Defined in AI Agent system prompt                                                                                |
| ISO 3166 Alpha-2 country codes must be lowercase when passed to Google Maps Scraper                            | Defined in AI Agent system prompt                                                                                |
| URLs passed to Website Scraper must be exact, without quotation marks or formatting                            | Defined in AI Agent system prompt                                                                                |
| Google Sheets output format can be customized to fit user needs                                                | Mentioned in workflow description                                                                                |
| Workflow designed for sales, marketing, and research professionals needing efficient lead generation          | Workflow description and sticky notes                                                                            |

---

This documentation provides a complete, structured understanding of the Google Maps FULL n8n workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.