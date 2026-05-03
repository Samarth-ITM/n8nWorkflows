Create Data-Driven SEO Content Briefs with AI Analysis of SERP Data using Bright Data

https://n8nworkflows.xyz/workflows/create-data-driven-seo-content-briefs-with-ai-analysis-of-serp-data-using-bright-data-8053


# Create Data-Driven SEO Content Briefs with AI Analysis of SERP Data using Bright Data

### 1. Workflow Overview

This workflow automates the generation of data-driven SEO content briefs by analyzing the top 10 Google search results for a user-provided keyword. It combines real-time SERP scraping using Bright Data with AI-powered content analysis and strategic planning via OpenAI models accessed through OpenRouter. The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Initial Interaction:** Receives user keyword input through a chat interface and provides initial status responses.
- **1.2 SERP Data Retrieval:** Queries Google Search via Bright Data to obtain the top 10 organic search results for the keyword.
- **1.3 URL Extraction and Processing Loop:** Extracts individual URLs from the SERP data, iterates through each URL to scrape page contents.
- **1.4 Content Cleaning and Extraction:** Cleans raw HTML content and extracts structured information such as titles, meta descriptions, headings, and word counts.
- **1.5 Content Analysis and Aggregation:** Aggregates extracted data, analyzes the collective insights for search intent, core topics, and differentiation opportunities using AI language models.
- **1.6 Content Brief Generation and Formatting:** Produces a structured strategic content plan and formats it into a human-readable Markdown summary.
- **1.7 User Feedback and Memory Management:** Provides ongoing chat responses to the user and maintains conversational context via memory buffering.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initial Interaction

- **Overview:** Captures the user's target keyword through a chat interface and signals workflow progression with messages.
- **Nodes Involved:** When chat message received, Respond to Chat1, Respond to Chat2, Respond to Chat
- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point; triggers workflow on chat input  
    - Config: Public webhook; title "SEO Content Strategist"; input placeholder prompts for a target keyword; memory enabled to load prior sessions.  
    - Expressions: Uses `$json.chatInput` to capture keyword.  
    - Outputs: Connected to Respond to Chat1.  
    - Edge cases: Missing or malformed input; webhook availability.  

  - **Respond to Chat1**  
    - Type: LangChain Chat  
    - Role: Provides immediate feedback that keyword analysis has started  
    - Config: Message dynamically interpolates the user keyword for clarity.  
    - Inputs: From "When chat message received"  
    - Outputs: Triggers Google SERP node indirectly.  
    - Edge cases: Expression failures if keyword missing.  

  - **Respond to Chat2**  
    - Type: LangChain Chat  
    - Role: Confirms processing of Google results.  
    - Config: Outputs a status message with the keyword.  
    - Inputs: From Merge1 node (later merge of analysis data)  
    - Outputs: Leads to "Loop Over Items" for detailed processing.  

  - **Respond to Chat**  
    - Type: LangChain Chat  
    - Role: Notifies user that content extraction from pages is starting.  
    - Inputs: From extract url code node.  
    - Outputs: Leads to Limit node (currently disabled).  

#### 1.2 SERP Data Retrieval

- **Overview:** Performs a Google search on the user keyword using Bright Data's Google SERP API, retrieving top 10 organic results as JSON.
- **Nodes Involved:** Google SERP
- **Node Details:**

  - **Google SERP**  
    - Type: Bright Data node  
    - Role: Fetches search results JSON from Google via Bright Data proxy  
    - Config: URL dynamically constructed with encoded user keyword; results limited to 10; uses Bright Data's "serp_api1" zone and US country setting; credentials are Bright Data API keys.  
    - Input: Triggered by Respond to Chat1 node.  
    - Output: JSON containing SERP data with an "organic" array.  
    - Edge cases: Proxy failures, API rate limits, network errors, invalid keyword.  

#### 1.3 URL Extraction and Processing Loop

- **Overview:** Extracts each organic result URL from the SERP JSON and processes each URL individually in batches.
- **Nodes Involved:** extract url (Code), Respond to Chat, Limit (disabled), Loop Over Items, Aggregate, Access and extract data from a specific URL, url (Set), Merge1
- **Node Details:**

  - **extract url**  
    - Type: Code node (JavaScript)  
    - Role: Parses the "organic" array from SERP JSON to emit individual items per result  
    - Key code: Uses optional chaining and array mapping to flatten and isolate each URL result.  
    - Input: From Google SERP output  
    - Output: One item per SERP result for downstream processing  
    - Edge cases: Missing or malformed "organic" property, empty results.  

  - **Respond to Chat**  
    - Role: Notifies user that content extraction starts  
    - Input: From extract url  
    - Output: Triggers Limit node (currently disabled)  

  - **Limit**  
    - Disabled node; would limit batch size if enabled.  

  - **Loop Over Items**  
    - Type: splitInBatches  
    - Role: Iterates over each extracted URL result with controlled batch processing  
    - Input: From Respond to Chat2  
    - Outputs: Two branches: one aggregates all data, the other scrapes each URL page content and sets URL context.  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects all batch iteration outputs into a single aggregated data set  
    - Input: From Loop Over Items  
    - Output: Feeds into Respond to Chat3  

  - **Access and extract data from a specific URL**  
    - Type: Bright Data node  
    - Role: Scrapes the actual page content of each URL using Bright Data's web unlocker proxy  
    - Config: URL dynamically set from current item’s "link" field; uses "web_unlocker1" zone and US country; retries on failure enabled  
    - Input: From Loop Over Items batch branch  
    - Output: Raw HTML content for cleaning  
    - Edge cases: Page load failures, anti-bot measures, malformed URLs  

  - **url**  
    - Type: Set node  
    - Role: Sets a simple JSON property "url" with the current processed link for tracking  
    - Input: From Access and extract data from a specific URL  
    - Output: Merges into Merge1  

  - **Merge1**  
    - Type: Merge  
    - Role: Combines data streams from content analysis and URL setting nodes  
    - Inputs: Three inputs – from analyse site, extract html1, and url nodes  
    - Output: Flows into Respond to Chat2 and further downstream  

#### 1.4 Content Cleaning and Extraction

- **Overview:** Cleans HTML content to remove noise and extracts structured elements including titles, meta descriptions, headings, and word counts.
- **Nodes Involved:** clean html (Code), analyse site (Chain LLM), extract html1 (Code), Structured Output Parser
- **Node Details:**

  - **clean html**  
    - Type: Code node  
    - Role: Applies regex cleaning to raw HTML removing scripts, styles, SVGs, navs, ul/li tags, and extraneous attributes  
    - Input: From Access and extract data from a specific URL  
    - Output: Produces "cleanedHtml" property for further processing  
    - Edge cases: Missing or empty HTML content  

  - **analyse site**  
    - Type: LangChain Chain LLM node  
    - Role: Uses AI to summarize the cleaned HTML content from each page into JSON summary  
    - Config: Prompt instructs SEO expert role to provide a JSON summary without comments  
    - Input: From clean html output (cleanedHtml)  
    - Output: Parsed JSON summary of page content  
    - Edge cases: Model timeouts, API errors, malformed input  

  - **extract html1**  
    - Type: Code node  
    - Role: Extracts page title, meta description, all headings (H1-H6), and counts words from cleaned HTML  
    - Input: From clean html  
    - Output: Structured JSON with these fields for analysis  
    - Edge cases: Missing tags, malformed HTML, empty content  

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Parses AI output from analyse site into structured JSON format as defined by schema  
    - Input: From analyse site  
    - Output: Validated JSON data for merging  

#### 1.5 Content Analysis and Aggregation

- **Overview:** Aggregates all extracted data and synthesizes a strategic SEO content brief by analyzing search intent, core topics, differentiation points, and proposing an article outline.
- **Nodes Involved:** Aggregate, Respond to Chat3, Analysis (Chain LLM), Structured Output Parser1, OpenRouter Chat Model2, Respond to Chat5
- **Node Details:**

  - **Aggregate**  
    - Collects all page summaries and extracted data into one set to provide comprehensive input for analysis  

  - **Respond to Chat3**  
    - Alerts the user that data collection completed and synthesis is ongoing  

  - **Analysis**  
    - Type: LangChain Chain LLM node  
    - Role: Performs deep AI analysis on aggregated SERP and page data, guided by a detailed prompt to generate a JSON SEO content brief with fields for search intent, global intent, must-cover topics, differentiation suggestions, and suggested H2 outline  
    - Config: Uses OpenRouter with GPT-4o model; error continuation enabled to avoid workflow stops  
    - Input: Aggregated data and user keyword  
    - Output: Structured JSON content plan  
    - Edge cases: AI model errors, malformed data, timeout  

  - **Structured Output Parser1**  
    - Parses the JSON output of the Analysis node to ensure correct schema adherence  

  - **Respond to Chat5**  
    - Sends the raw JSON analysis back to the user for transparency or further use  

#### 1.6 Content Brief Generation and Formatting

- **Overview:** Formats the AI-generated JSON content brief into a Markdown summary for easier human consumption.
- **Nodes Involved:** Format Output, Respond to Chat4, OpenRouter Chat Model1
- **Node Details:**

  - **Format Output**  
    - Type: LangChain Chain LLM node  
    - Role: Converts the JSON content brief into a structured Markdown report with headings and bullet points for search intent, must-cover topics, differentiation, and H2 outline  
    - Config: Uses GPT-5-nano model on OpenRouter  
    - Input: JSON from Analysis node  
    - Output: Markdown formatted content plan  

  - **Respond to Chat4**  
    - Delivers the formatted Markdown content back to the chat user as the final content brief  

  - **OpenRouter Chat Model1**  
    - Used as the LLM engine powering the formatting step  

#### 1.7 User Feedback and Memory Management

- **Overview:** Maintains conversational memory and provides ongoing interactive user feedback throughout the workflow.
- **Nodes Involved:** Simple Memory, When chat message received (memory connection), various Respond to Chat nodes
- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversation context across chat messages to enable session continuity  
    - Input/Output: Connected bi-directionally to chat trigger node  
    - Edge cases: Memory overflow, context mismatch  

---

### 3. Summary Table

| Node Name                         | Node Type                                  | Functional Role                               | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                                         |
|----------------------------------|--------------------------------------------|-----------------------------------------------|-------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received        | LangChain Chat Trigger                      | Entry point, receives user keyword            | Simple Memory                       | Respond to Chat1                      | # SEO content Planner  \n\n **Google SERP Analysis**  \nThe [n8n](https://n8n.partnerlinks.io/build) workflow retrieves the **top 10 Google search results** for a given keyword using Bright Data.  \n👉 Get your free Bright Data API key here: [https://get.brightdata.com/scrap](https://get.brightdata.com/scrap) |
| Respond to Chat1                 | LangChain Chat                             | Notifies user of keyword analysis start       | When chat message received          | Google SERP                          | See above                                                                                                                           |
| Google SERP                     | Bright Data                                | Retrieves top 10 Google organic results        | Respond to Chat1                   | extract url                         | See above                                                                                                                           |
| extract url                    | Code (JS)                                   | Extracts individual URLs from SERP JSON        | Google SERP                      | Respond to Chat                      | See above                                                                                                                           |
| Respond to Chat                | LangChain Chat                             | Notifies user extraction starts                | extract url                     | Limit (disabled)                    | See above                                                                                                                           |
| Limit                          | Limit (disabled)                           | (Disabled) limit item count                     | Respond to Chat                 | Loop Over Items                    | See above                                                                                                                           |
| Loop Over Items                | splitInBatches                            | Iterates over URLs batch-wise                    | Respond to Chat2                | Aggregate, Access and extract data from a specific URL, url | See above                                                                                                                           |
| Aggregate                     | Aggregate                                  | Aggregates batch results for analysis           | Loop Over Items                | Respond to Chat3                   | See above                                                                                                                           |
| Access and extract data from a specific URL | Bright Data                                | Scrapes page content for each URL               | Loop Over Items                | clean html                       | See above                                                                                                                           |
| url                           | Set                                        | Sets current URL for tracking                    | Access and extract data from a specific URL | Merge1                          | See above                                                                                                                           |
| clean html                    | Code (JS)                                   | Cleans raw HTML content                          | Access and extract data from a specific URL | analyse site, extract html1        | See above                                                                                                                           |
| analyse site                  | LangChain Chain LLM                         | Summarizes page content via AI                   | clean html                     | Merge1                            | See above                                                                                                                           |
| extract html1                 | Code (JS)                                   | Extracts title, description, headings, word count | clean html                     | Merge1                            | See above                                                                                                                           |
| Merge1                       | Merge                                      | Merges analysis and extraction data              | analyse site, extract html1, url | Respond to Chat2                  | See above                                                                                                                           |
| Respond to Chat2              | LangChain Chat                             | Shows processing status of URLs                  | Merge1                        | Loop Over Items                  | See above                                                                                                                           |
| Respond to Chat3              | LangChain Chat                             | Notifies user that data collection is complete  | Aggregate                     | Analysis                       | See above                                                                                                                           |
| Analysis                     | LangChain Chain LLM                         | Synthesizes strategic SEO content brief          | Respond to Chat3              | Format Output, Respond to Chat5  | See above                                                                                                                           |
| Structured Output Parser1    | LangChain Output Parser                     | Parses AI JSON analysis output                    | Analysis                     | Analysis (chain continuation)   | See above                                                                                                                           |
| Respond to Chat5              | LangChain Chat                             | Sends raw JSON analysis to user                   | Analysis                     | -                             | See above                                                                                                                           |
| Format Output                | LangChain Chain LLM                         | Formats JSON brief into Markdown summary          | Analysis                     | Respond to Chat4                | See above                                                                                                                           |
| Respond to Chat4              | LangChain Chat                             | Sends formatted content brief to user             | Format Output                | -                             | See above                                                                                                                           |
| OpenRouter Chat Model         | LangChain LLM via OpenRouter                | Powers content cleaning analysis                   | clean html                   | analyse site                   | See above                                                                                                                           |
| OpenRouter Chat Model1        | LangChain LLM via OpenRouter                | Powers output formatting                            | Format Output                | Respond to Chat4                | See above                                                                                                                           |
| OpenRouter Chat Model2        | LangChain LLM via OpenRouter                | Powers strategic brief analysis                      | Analysis                     | Structured Output Parser1       | See above                                                                                                                           |
| Structured Output Parser      | LangChain Output Parser                     | Parses page summary output                           | analyse site                 | Merge1                        | See above                                                                                                                           |
| Simple Memory                 | LangChain Memory Buffer Window              | Maintains chat conversation context                | -                           | When chat message received      | See above                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a LangChain Chat Trigger node**  
   - Name: When chat message received  
   - Configure as a public webhook with title "SEO Content Strategist" and subtitle "Generate a strategic content brief based on SERP analysis."  
   - Set input placeholder as "Enter your target keyword..."  
   - Enable memory loading (connect to a Memory Buffer node)  
   - Save webhook ID for external chat integration.

2. **Add a LangChain Chat node**  
   - Name: Respond to Chat1  
   - Message: `=Processing: Analyzing top 10 Google results for "{{ $json.chatInput }}".`  
   - Connect input from "When chat message received".

3. **Add a Bright Data node for Google SERP API**  
   - Name: Google SERP  
   - URL: `=https://www.google.com/search?q={{ encodeURIComponent($json.chatInput) }}&num=10&brd_json=1`  
   - Zone: Select or create "serp_api1" zone in Bright Data credentials  
   - Country: US  
   - Connect input from Respond to Chat1.

4. **Add a Code node to extract URLs from SERP data**  
   - Name: extract url  
   - Use provided JavaScript code to flatten the "organic" array into individual items  
   - Connect input from Google SERP output.

5. **Add a LangChain Chat node**  
   - Name: Respond to Chat  
   - Message: "Starting content extraction from top-ranking pages."  
   - Connect input from extract url.

6. **(Optional) Add a Limit node**  
   - Name: Limit  
   - Initially disable this node (for possible future batch size limiting)  
   - Connect input from Respond to Chat.

7. **Add a splitInBatches node**  
   - Name: Loop Over Items  
   - Connect input from Respond to Chat2 (which will be created later)  
   - Configure default batch size or leave default.

8. **Add a Bright Data node for page scraping**  
   - Name: Access and extract data from a specific URL  
   - URL: `={{ $json.link }}` (from current batch item)  
   - Zone: "web_unlocker1"  
   - Country: US  
   - Enable retry on failure  
   - Connect input from Loop Over Items.

9. **Add a Code node to clean HTML**  
   - Name: clean html  
   - Insert given regex cleaning code to remove scripts, styles, SVG, nav, ul/li, and attributes from HTML content  
   - Connect input from Access and extract data from a specific URL.

10. **Add a LangChain Chain LLM node**  
    - Name: analyse site  
    - Prompt: "You are an SEO expert:|Output in JSON without any comments** summary: summarize this page."  
    - Input text: `={{ $json.cleanedHtml }}`  
    - Enable output parser and link to Structured Output Parser node  
    - Connect input from clean html.

11. **Add a Code node to extract structured HTML info**  
    - Name: extract html1  
    - Use provided code to extract title, description, headings, and word count from cleaned HTML  
    - Connect input from clean html.

12. **Add a Set node**  
    - Name: url  
    - Assign a string field "url" with value `={{ $json.link }}` for tracking  
    - Connect input from Access and extract data from a specific URL.

13. **Add a Merge node**  
    - Name: Merge1  
    - Set number of inputs to 3  
    - Connect inputs from analyse site, extract html1, and url nodes respectively.

14. **Add a LangChain Chat node**  
    - Name: Respond to Chat2  
    - Message: `=Processing: Analyzing top 10 Google results for "{{ $json.chatInput }}".`  
    - Connect input from Merge1.

15. **Connect Respond to Chat2 output back to Loop Over Items input**  
    - This closes the loop for each batch item processed.

16. **Add an Aggregate node**  
    - Name: Aggregate  
    - Aggregate all batch outputs into a single dataset  
    - Connect input from Loop Over Items.

17. **Add a LangChain Chat node**  
    - Name: Respond to Chat3  
    - Message: "All data collected. Synthesizing insights and generating your strategic content plan."  
    - Connect input from Aggregate.

18. **Add a LangChain Chain LLM node for analysis**  
    - Name: Analysis  
    - Use provided prompt to analyze aggregated SERP and page data for search intent and content plan  
    - Enable structured output parser (Structured Output Parser1)  
    - Connect input from Respond to Chat3.

19. **Add a LangChain Output Parser node**  
    - Name: Structured Output Parser1  
    - Schema: Use provided JSON schema defining search intent, global intent, must-cover topics, differentiation suggestions, and suggested H2 outline  
    - Connect input from Analysis node.

20. **Add a LangChain Chain LLM node for formatting**  
    - Name: Format Output  
    - Prompt: Convert JSON analysis to Markdown report as specified  
    - Connect input from Analysis node.

21. **Add LangChain Chat node**  
    - Name: Respond to Chat4  
    - Message: Use `={{ $json.text }}` to output formatted Markdown  
    - Connect input from Format Output.

22. **Add LangChain Chat node**  
    - Name: Respond to Chat5  
    - Message: Use `={{ $json.text }}` to output raw JSON analysis  
    - Connect input from Analysis node.

23. **Add LangChain Memory Buffer Window node**  
    - Name: Simple Memory  
    - Connect AI memory input/output to "When chat message received" node to maintain session memory.

24. **Credential Setup**  
    - Bright Data API: Configure with valid API keys for zones "serp_api1" and "web_unlocker1"  
    - OpenRouter API: Configure with valid credentials to access GPT-4o and GPT-5-nano models  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow retrieves the top 10 Google search results and scrapes their content using Bright Data proxies.                                                                | Bright Data: https://get.brightdata.com/scrap                 |
| The AI-driven analysis uses OpenRouter-hosted GPT models (GPT-4o, GPT-5-nano) for summarization, SEO analysis, and content plan generation.                                 | OpenRouter API credentials required                            |
| The final content brief includes search intent classification, must-cover topics, differentiation suggestions, and a suggested H2 outline formatted in Markdown.            | SEO Content Planning best practices                            |
| The workflow uses memory buffering to maintain chat context and enhance user interaction continuity.                                                                        | LangChain memory buffers                                       |
| The cleaning step removes common HTML noise elements such as scripts, styles, SVGs, navigation bars, and list tags to focus on meaningful text content for analysis.         | Custom regex cleaning implemented in code node                |

---

**Disclaimer:** The text above is exclusively derived from an automated workflow constructed with n8n, adhering strictly to content policies and containing no illegal or offensive elements. All processed data is legal and publicly available.