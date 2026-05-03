Generate B2B Lead Opportunities from Websites with Brightdata & OpenRouter AI

https://n8nworkflows.xyz/workflows/generate-b2b-lead-opportunities-from-websites-with-brightdata---openrouter-ai-8076


# Generate B2B Lead Opportunities from Websites with Brightdata & OpenRouter AI

---

### 1. Workflow Overview

This workflow automates **B2B lead opportunity generation** by analyzing company websites through AI-powered content extraction and processing. It is designed for sales and business development teams seeking actionable insights on potential client companies by scanning their public web pages.

The logical flow is divided into these blocks:

- **1.1 Input Reception:** Receives a company URL input via a chat interface.
- **1.2 Initial URL Scraping:** Uses Bright Data’s Web Unblocker to scrape the homepage and extract all linked URLs.
- **1.3 Relevant Page Identification:** Filters extracted URLs, selecting those relevant to business information (e.g., About Us, Team).
- **1.4 Secondary Scraping:** Scrapes each relevant page to extract clean HTML content.
- **1.5 Opportunity Detection:** Applies OpenRouter AI models to analyze page content and identify potential business opportunities.
- **1.6 Data Aggregation & Deduplication:** Merges and consolidates AI-generated insights, removing redundancies to produce a final summarized report.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

- **Overview:**  
  Captures the initial input URL from a user via a chat message, setting it as a parameter for downstream processing.

- **Nodes Involved:**  
  - When chat message received  
  - parameters

- **Node Details:**

  - **When chat message received**  
    - Type: Chat Trigger (LangChain Chat Trigger)  
    - Role: Receives user input via a public webhook chat interface.  
    - Configuration:  
      - Public webhook enabled  
      - Initial greeting message explaining workflow purpose and input expectation (URL of company website).  
      - Response mode: returns output from the last node executed.  
    - Inputs: External chat message webhook  
    - Outputs: User message payload  
    - Failure types: Webhook connectivity or trigger misconfiguration.

  - **parameters**  
    - Type: Set node  
    - Role: Extracts the chat input URL and assigns it to a variable "url". Also sets a "sitemap" parameter with default value "sitemap.xml" (unused downstream).  
    - Configuration:  
      - Assigns "url" from the chat input (JSON path: `$json.chatInput`).  
      - Sets "sitemap" to a static string "sitemap.xml".  
    - Inputs: Message from chat trigger node  
    - Outputs: JSON object with `url` and `sitemap`.  
    - Failure types: Expression errors if input JSON path is missing or malformed.

---

#### 1.2 Initial URL Scraping

- **Overview:**  
  Scrapes the homepage URL using Bright Data’s Web Unblocker to retrieve raw HTML content and extract all hyperlinks.

- **Nodes Involved:**  
  - scrap urls  
  - extract url  
  - Find best pages  
  - clean list

- **Node Details:**

  - **scrap urls**  
    - Type: BrightData Web Scraper node  
    - Role: Scrapes the homepage URL to obtain raw page content.  
    - Configuration:  
      - URL set dynamically from `parameters.url`.  
      - Zone: "web_unlocker1" (Bright Data’s unblocker zone).  
      - Country: "us" (geotargeting).  
      - Format: JSON output.  
    - Credentials: BrightData API key required.  
    - Inputs: URL parameter from previous node  
    - Outputs: JSON including raw HTML body.  
    - Failure types: API key failures, blocking by target site, network timeouts.

  - **extract url**  
    - Type: Code node (JavaScript)  
    - Role: Parses the scraped HTML body to extract all href URLs from anchor (`<a>`) tags.  
    - Configuration:  
      - Uses regex to find all `href` attributes.  
      - Returns an array of items, each with extracted URL.  
    - Inputs: JSON with `body` property containing HTML string.  
    - Outputs: List of URLs extracted from homepage.  
    - Failure types: HTML content missing or malformed, regex failure.

  - **Find best pages**  
    - Type: LangChain Agent node  
    - Role: Filters extracted URLs to identify relevant pages for prospecting (e.g., "about us", "team", "contact").  
    - Configuration:  
      - System message instructs to output only absolute URLs in JSON format, ignoring relative or anchor links.  
      - Input: JSON list of URLs from previous node.  
      - Output parser enabled to handle structured JSON responses.  
    - Inputs: List of extracted URLs  
    - Outputs: Filtered list of relevant URLs.  
    - Failure types: AI model response errors, parsing errors.

  - **clean list**  
    - Type: Code node (JavaScript)  
    - Role: Cleans the filtered URL list by removing empty or invalid entries and maintaining item pairing for tracking.  
    - Configuration:  
      - Extracts URLs from AI output, filters out empty values, returns new list.  
    - Inputs: AI filtered URLs  
    - Outputs: Clean list of valid URLs for further scraping.  
    - Failure types: Missing or malformed AI output.

---

#### 1.3 Secondary Scraping

- **Overview:**  
  Scrapes each relevant company page identified, extracts clean HTML content focused on the `<body>` tag for AI analysis.

- **Nodes Involved:**  
  - scrap urls1  
  - HTML cleaner

- **Node Details:**

  - **scrap urls1**  
    - Type: BrightData Web Scraper node  
    - Role: Scrapes each relevant URL to get raw page content.  
    - Configuration:  
      - URL set dynamically from clean list output.  
      - Same Bright Data zone and country settings as initial scraper.  
      - Format: JSON output.  
    - Credentials: BrightData API key.  
    - Inputs: List of relevant URLs.  
    - Outputs: JSON including raw HTML body per page.  
    - Failure types: Same as initial scraper.

  - **HTML cleaner**  
    - Type: HTML Extract node  
    - Role: Extracts only the `<body>` content from the scraped HTML for cleaner input to AI.  
    - Configuration:  
      - CSS selector: `body`  
      - Extracted content stored in property `body`.  
    - Inputs: Raw HTML from scraper.  
    - Outputs: Cleaned HTML body content per page.  
    - Failure types: If HTML is missing or malformed.

---

#### 1.4 Opportunity Detection

- **Overview:**  
  Applies OpenRouter AI models to analyze the cleaned HTML content of each relevant page to identify potential business opportunities.

- **Nodes Involved:**  
  - Identify business opportunities

- **Node Details:**

  - **Identify business opportunities**  
    - Type: LangChain Agent node  
    - Role: Uses AI to analyze page content and detect possible B2B lead opportunities.  
    - Configuration:  
      - System message sets context as B2B lead generation expert for AI-powered process automation agency.  
      - Input includes the cleaned HTML body and URL context.  
      - Output is a JSON summary of identified opportunities.  
      - Output parser enabled for structured response.  
    - Inputs: Cleaned HTML body content  
    - Outputs: AI-generated opportunity summaries per page.  
    - Failure types: AI API errors, parsing issues.

---

#### 1.5 Data Aggregation & Deduplication

- **Overview:**  
  Merges all identified opportunity summaries from multiple pages into a single object and deduplicates redundant points to produce a concise report.

- **Nodes Involved:**  
  - merge pages  
  - de-dupe

- **Node Details:**

  - **merge pages**  
    - Type: Code node (JavaScript)  
    - Role: Combines multiple AI outputs into one consolidated JSON object.  
    - Configuration:  
      - Iterates over all items and merges their `output` objects into one.  
      - Returns a single item with merged results.  
    - Inputs: List of AI opportunity outputs  
    - Outputs: Single merged JSON object of all opportunities.  
    - Failure types: Missing or malformed AI output.

  - **de-dupe**  
    - Type: LangChain Agent node  
    - Role: Final AI pass to synthesize and eliminate redundant opportunities, formatting output for Google Docs.  
    - Configuration:  
      - System message instructs to produce a clean, non-redundant summary with headings, without commentary.  
      - Input: Merged opportunity summary.  
    - Inputs: Merged JSON from previous node  
    - Outputs: Final synthesized lead opportunity report.  
    - Failure types: AI response or formatting errors.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                                 | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                                             |
|-------------------------|------------------------------------|------------------------------------------------|----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger             | Receives user's company URL input               | -                          | parameters                 | Welcome message explains workflow purpose, input URL expected                                                                           |
| parameters              | Set                                | Sets input parameters (URL, sitemap)            | When chat message received | scrap urls                 |                                                                                                                                         |
| scrap urls              | BrightData Web Scraper              | Scrapes homepage URL for raw HTML                | parameters                 | extract url                |                                                                                                                                         |
| extract url             | Code (JS)                         | Extracts all URLs from homepage HTML             | scrap urls                 | Find best pages            |                                                                                                                                         |
| Find best pages         | LangChain Agent                    | Filters URLs to relevant business pages          | extract url                | clean list, OpenRouter Chat Model1 (AI) |                                                                                                                                         |
| clean list              | Code (JS)                         | Cleans and filters relevant URLs list             | Find best pages            | scrap urls1                |                                                                                                                                         |
| scrap urls1             | BrightData Web Scraper              | Scrapes each relevant page for raw HTML          | clean list                 | HTML cleaner               |                                                                                                                                         |
| HTML cleaner            | HTML Extract                      | Extracts `<body>` content from page HTML          | scrap urls1                | Identify business opportunities |                                                                                                                                         |
| Identify business opportunities | LangChain Agent               | AI analyzes page content to identify opportunities | HTML cleaner               | merge pages                |                                                                                                                                         |
| merge pages             | Code (JS)                         | Merges all AI opportunity outputs into one object | Identify business opportunities | de-dupe                   |                                                                                                                                         |
| de-dupe                 | LangChain Agent                    | Deduplicates and synthesizes final opportunity report | merge pages                | -                          |                                                                                                                                         |
| OpenRouter Chat Model1  | LangChain LM Chat (OpenRouter)    | AI model used in "Find best pages" filtering    | -                          | Structured Output Parser1, Find best pages (AI output) |                                                                                                                                         |
| Structured Output Parser1 | LangChain Output Parser Structured | Parses AI response into structured JSON          | OpenRouter Chat Model1     | Find best pages            |                                                                                                                                         |
| OpenRouter Chat Model2  | LangChain LM Chat (OpenRouter)    | AI model used in "Identify business opportunities" | -                          | Structured Output Parser2  |                                                                                                                                         |
| Structured Output Parser2 | LangChain Output Parser Structured | Parses AI response with opportunity summaries    | OpenRouter Chat Model2     | Identify business opportunities |                                                                                                                                         |
| OpenRouter Chat Model3  | LangChain LM Chat (OpenRouter)    | AI model used in final deduplication & synthesis | -                          | de-dupe                    |                                                                                                                                         |
| Sticky Note             | Sticky Note                       | Workflow info and external resource links        | -                          | -                          | # Lead Opportunity Finder: overview and Bright Data API link: https://get.brightdata.com/scrap                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Type: LangChain Chat Trigger  
   - Configure:  
     - Public webhook enabled  
     - Initial message: "Welcome, I am your business opportunity detection agent. Enter the URL of the company to be analyzed..."  
     - Response mode: lastNode  
   - No credentials needed.

2. **Create Set Node (parameters):**  
   - Type: Set  
   - Assign variables:  
     - `url` = Expression: `{{$json.chatInput}}` (captures user input)  
     - `sitemap` = static string `"sitemap.xml"` (not actively used)  
   - Connect output of Chat Trigger node.

3. **Create BrightData Web Scraper Node (scrap urls):**  
   - Type: BrightData  
   - URL: Expression from `parameters.url`  
   - Zone: Select or enter "web_unlocker1" (Bright Data Web Unblocker zone)  
   - Country: US  
   - Format: JSON  
   - Credentials: Set with your Bright Data API key  
   - Connect from parameters node.

4. **Create Code Node (extract url):**  
   - Type: Code (JavaScript)  
   - Paste code to extract all href URLs from HTML body:  
   ```js
   const htmlBody = items[0].json.body;
   if (typeof htmlBody !== 'string') return [];
   const regex = /<a\s+(?:[^>]*?\s+)?href="([^"]+)"/gi;
   const matches = [...htmlBody.matchAll(regex)];
   return matches.map(match => ({ json: { url: match[1] } }));
   ```  
   - Connect from scrap urls node.

5. **Create LangChain Agent Node (Find best pages):**  
   - Type: LangChain Agent  
   - Parameters:  
     - Input text: `={{ $json }}` (passes the extracted URLs)  
     - System message: "You are a B2B data extraction expert. Identify URLs relevant for prospecting..."  
     - Output parser: enabled, manual schema to expect JSON objects with `url` key.  
     - Prompt type: define  
   - Connect from extract url node.

6. **Create Code Node (clean list):**  
   - Type: Code (JavaScript)  
   - Code to filter and flatten valid URLs:  
   ```js
   return items.flatMap((item, index) => {
     const url = item.json.output?.[0]?.url;
     return url ? [{ json: { url }, pairedItem: index }] : [];
   });
   ```  
   - Connect from Find best pages node.

7. **Create BrightData Web Scraper Node (scrap urls1):**  
   - Same config as scrap urls node except URL is dynamic from clean list output.  
   - Credentials: same Bright Data API key.  
   - Connect from clean list node.

8. **Create HTML Extract Node (HTML cleaner):**  
   - Operation: Extract HTML content  
   - CSS selector: `body`  
   - Store extracted content in property `body`  
   - Connect from scrap urls1 node.

9. **Create LangChain Agent Node (Identify business opportunities):**  
   - Role: Analyze page HTML content for opportunities.  
   - System message: "You are a B2B lead generation expert... Look for business opportunities..."  
   - Input text: `={{ $json.body }}`  
   - Output parser: enabled with manual JSON schema expecting a summary string.  
   - Connect from HTML cleaner node.

10. **Create Code Node (merge pages):**  
    - Purpose: Merge all AI opportunity outputs into a single object.  
    - Code:  
    ```js
    const mergedData = items.reduce((acc, item) => {
      if (item.json && item.json.output) return { ...acc, ...item.json.output };
      return acc;
    }, {});
    return [{ json: { mergedOutput: mergedData }, pairedItem: 0 }];
    ```  
    - Connect from Identify business opportunities node.

11. **Create LangChain Agent Node (de-dupe):**  
    - Role: Deduplicate and synthesize final report.  
    - System message: "You are a B2B lead generation expert... Eliminate redundancies... Format for Google Doc..."  
    - Input text: `={{ $json.mergedOutput }}`  
    - Connect from merge pages node.

12. **Credential Setup:**  
    - BrightData API credentials configured with valid API key.  
    - OpenRouter API credentials configured for OpenAI models used in LangChain Agent nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                       |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Workflow automates B2B lead generation by analyzing company websites with AI and Bright Data.   | Sticky Note in workflow overview                      |
| Bright Data API key needed for web scraping: [https://get.brightdata.com/scrap](https://get.brightdata.com/scrap) | Sticky Note external resource link                    |
| AI models used via OpenRouter with OpenAI GPT-5 and smaller variant ("o4-mini")                  | Node configurations                                   |
| Final output is a synthesized, non-redundant report formatted for Google Docs, aiding sales teams | Workflow description and AI agent system messages    |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and public.

---