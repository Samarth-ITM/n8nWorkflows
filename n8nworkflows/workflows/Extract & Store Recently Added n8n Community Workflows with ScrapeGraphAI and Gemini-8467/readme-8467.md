Extract & Store Recently Added n8n Community Workflows with ScrapeGraphAI and Gemini

https://n8nworkflows.xyz/workflows/extract---store-recently-added-n8n-community-workflows-with-scrapegraphai-and-gemini-8467


# Extract & Store Recently Added n8n Community Workflows with ScrapeGraphAI and Gemini

---

### 1. Workflow Overview

This workflow automates the extraction, processing, and storage of recently added community workflows from the n8n website. It leverages advanced AI models and web scraping to transform unstructured markdown content into structured, enriched data records stored in Google Sheets.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initial Scraping:** Starts execution manually or via schedule, scrapes the n8n workflows main page to retrieve raw markdown content.
- **1.2 Data Extraction from Main Page:** Uses AI to parse the "Recently Added" section from the scraped markdown, producing a JSON array of workflow titles and URLs.
- **1.3 Iterative Detailed Scraping and Information Extraction:** Loops over each workflow URL to scrape detailed markdown, then uses AI to extract structured attributes (author, categories, price, title, ID) and clean the markdown content.
- **1.4 Content Summarization:** Summarizes each workflow’s purpose and tools used via AI, producing a concise Italian summary.
- **1.5 Data Aggregation and Storage:** Merges extracted data and summaries, and appends each record as a row to a Google Sheets spreadsheet for centralized storage.

This design demonstrates a multi-step pipeline that integrates web scraping, AI natural language processing, and data storage in an automated, repeatable fashion.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initial Scraping

- **Overview:** Initiates the workflow execution and scrapes the main n8n workflows web page to obtain raw markdown content.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (manual trigger)  
  - Scrape main page (ScrapeGraphAI node)

- **Node Details:**

  - *When clicking ‘Execute workflow’*  
    - Type: Manual Trigger  
    - Role: Starts workflow on user command  
    - Configuration: Default manual trigger with no parameters  
    - Inputs: None  
    - Outputs: Connected to "Scrape main page"  
    - Failure Modes: None expected; user must manually trigger

  - *Scrape main page*  
    - Type: ScrapeGraphAI (markdownify resource)  
    - Role: Scrapes https://n8n.io/workflows/ with heavy JS rendering enabled to extract markdown content  
    - Configuration: Website URL set to "https://n8n.io/workflows/", renderHeavyJs enabled for dynamic content  
    - Inputs: Trigger node output  
    - Outputs: Raw markdown content, passed to "Google Gemini Chat Model"  
    - Credentials: ScrapeGraphAI API key required  
    - Potential Failures: Network errors, API key invalid/expired, site structure changes affecting scraping  

#### 2.2 Data Extraction from Main Page

- **Overview:** Uses Google Gemini AI to parse the scraped markdown, extracting the workflows listed in the "Recently Added" section as JSON objects containing title and URL.
- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - Extract "Recently added" (Chain LLM node)  
  - Set array

- **Node Details:**

  - *Google Gemini Chat Model*  
    - Type: Language Model Chat node (Google Gemini)  
    - Role: Processes raw markdown text to extract structured data  
    - Configuration: No additional options set  
    - Credentials: Google Palm API (Gemini) credentials required  
    - Input: Markdown content from "Scrape main page"  
    - Output: Passed to "Structured Output Parser"  
    - Failure Cases: API quota limits, malformed input, model response errors  

  - *Structured Output Parser*  
    - Type: Langchain Structured Output Parser  
    - Role: Validates and structures the AI output into a JSON schema with "workflows" array containing "title" and "url" string fields  
    - Configuration: Auto-fix enabled, manual schema defining object with workflows array  
    - Input: From Google Gemini Chat Model  
    - Output: To "Extract 'Recently added'" node  

  - *Extract "Recently added"*  
    - Type: Chain LLM node (Langchain)  
    - Role: Executes a prompt instructing expert extraction of "title" and "url" from the "Recently Added" section in markdown, outputs JSON array named "workflows"  
    - Configuration: Custom prompt in Italian directing the extraction task, output parser active  
    - Input: Raw markdown from Gemini model output  
    - Output: To "Set array" node  

  - *Set array*  
    - Type: Set node  
    - Role: Assigns extracted "workflows" array from AI output into workflow data for iteration  
    - Configuration: Stores JSON path `$json.output.workflows` into variable `workflows`  
    - Input: From "Extract 'Recently added'"  
    - Output: To "Split Out" node  

#### 2.3 Iterative Detailed Scraping and Information Extraction

- **Overview:** Loops through each workflow URL extracted previously, scrapes each workflow page markdown, extracts detailed structured attributes using AI, and cleans the main content.
- **Nodes Involved:**  
  - Split Out  
  - Loop Over Items (SplitInBatches)  
  - Scrape single Workflow (ScrapeGraphAI)  
  - Information Extractor (Langchain Information Extractor)  
  - Google Gemini Chat Model1  
  - Main content (Chain LLM)  
  - Google Gemini Chat Model2  
  - Set content

- **Node Details:**

  - *Split Out*  
    - Type: Split Out  
    - Role: Splits the "workflows" array into individual items for batch processing  
    - Configuration: Splits field "workflows"  
    - Input: From "Set array"  
    - Output: To "Loop Over Items"  

  - *Loop Over Items*  
    - Type: SplitInBatches  
    - Role: Processes items individually or in batches (default batch size)  
    - Configuration: Default batch options  
    - Input: From "Split Out"  
    - Outputs: Main output to "Scrape single Workflow"  

  - *Scrape single Workflow*  
    - Type: ScrapeGraphAI (markdownify resource)  
    - Role: Scrapes each workflow's individual URL to get detailed markdown content  
    - Configuration: URL dynamically set from current item’s `url` field, heavy JS rendering enabled  
    - Credentials: ScrapeGraphAI API key required  
    - Input: From "Loop Over Items"  
    - Output: To "Information Extractor" and "Main content" nodes  

  - *Information Extractor*  
    - Type: Langchain Information Extractor  
    - Role: Extracts specific attributes - categories, author, price, title (translated to Italian), URL, and numeric ID from the markdown text  
    - Configuration: Attributes explicitly defined with descriptive prompts in Italian, text input set from scraped markdown result  
    - Input: From "Scrape single Workflow"  
    - Output: To "Merge" node (second input) via "Merge" node  

  - *Google Gemini Chat Model1*  
    - Type: Google Gemini Chat Model (Langchain)  
    - Role: Used internally by Information Extractor node for attribute extraction AI calls  
    - Credentials: Google Palm API  
    - Input: Text from scraped workflow markdown  

  - *Main content*  
    - Type: Chain LLM node (Langchain)  
    - Role: Cleans scraped markdown by removing superfluous content, validating markdown integrity  
    - Configuration: Prompt instructs expert scraping and content cleaning, no invention of data, output valid markdown only  
    - Input: From "Scrape single Workflow"  
    - Output: To "Set content" node  

  - *Google Gemini Chat Model2*  
    - Type: Google Gemini Chat Model (Langchain)  
    - Role: Used internally by "Main content" node to perform content cleaning via AI  
    - Credentials: Google Palm API  

  - *Set content*  
    - Type: Set node  
    - Role: Stores cleaned markdown text in a string field `content` for downstream summarization  
    - Input: From "Main content"  
    - Output: To "Summarization content"  

#### 2.4 Content Summarization

- **Overview:** Summarizes the purpose and tools of each workflow in Italian, producing concise plain text descriptions.
- **Nodes Involved:**  
  - Summarization content  
  - Google Gemini Chat Model (implicit inside Summarization content)

- **Node Details:**

  - *Summarization content*  
    - Type: Google Gemini Chat Model (Langchain)  
    - Role: Generates concise Italian summary describing workflow purpose and tools used  
    - Configuration: Uses "models/gemini-2.5-flash" model, system message instructs clear, concise summary in plain text without preamble  
    - Credentials: Google Palm API  
    - Input: Uses the cleaned markdown content (`content` field)  
    - Output: To "Merge" node (first input) for aggregation  

#### 2.5 Data Aggregation and Storage

- **Overview:** Combines structured extracted attributes and summaries, then appends each workflow record as a new row in Google Sheets.
- **Nodes Involved:**  
  - Merge  
  - Add row (Google Sheets)

- **Node Details:**

  - *Merge*  
    - Type: Merge node  
    - Role: Combines outputs from Information Extractor and Summarization content into a single merged item  
    - Configuration: Mode "combine" with "combineAll" option to merge all properties  
    - Inputs: From Information Extractor (index 1), Summarization content (index 0)  
    - Output: To "Add row"  

  - *Add row*  
    - Type: Google Sheets node  
    - Role: Appends the combined workflow data as a new row to a specified Google Sheet  
    - Configuration:  
      - Operation: Append  
      - Document ID: Google Sheet ID provided  
      - Sheet Name: "gid=0" (default first sheet)  
      - Columns mapped: ID, URL, PRICE, TITLE, AUTHOR, SUMMARY, CATEGORIES with data from merged JSON fields  
    - Credentials: Google Sheets OAuth2 account needed  
    - Failure Cases: Authentication errors, quota limits, sheet permission issues  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                                  | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                                             |
|----------------------------|----------------------------------|-------------------------------------------------|------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Initiates workflow execution manually            | -                            | Scrape main page                |                                                                                                                                         |
| Scrape main page            | ScrapeGraphAI (markdownify)      | Scrapes main workflows page markdown             | When clicking ‘Execute workflow’ | Google Gemini Chat Model        |                                                                                                                                         |
| Google Gemini Chat Model    | Language Model Chat (Google Gemini) | Parses scraped markdown to extract "Recently Added" workflows | Scrape main page             | Structured Output Parser        |                                                                                                                                         |
| Structured Output Parser    | Output Parser (Langchain)         | Validates AI output into structured JSON         | Google Gemini Chat Model      | Extract "Recently added"        |                                                                                                                                         |
| Extract "Recently added"    | Chain LLM (Langchain)             | Extracts JSON array of workflow titles and URLs  | Structured Output Parser      | Set array                      |                                                                                                                                         |
| Set array                  | Set                              | Assigns extracted workflows array for iteration  | Extract "Recently added"      | Split Out                     |                                                                                                                                         |
| Split Out                 | Split Out                        | Splits workflows array into individual items     | Set array                    | Loop Over Items               |                                                                                                                                         |
| Loop Over Items            | SplitInBatches                   | Iterates over workflow items in batches          | Split Out                   | Scrape single Workflow         |                                                                                                                                         |
| Scrape single Workflow      | ScrapeGraphAI (markdownify)      | Scrapes individual workflow pages for details    | Loop Over Items              | Information Extractor, Main content |                                                                                                                                         |
| Information Extractor       | Langchain Information Extractor  | Extracts structured attributes from markdown     | Scrape single Workflow       | Merge                        |                                                                                                                                         |
| Google Gemini Chat Model1   | Language Model Chat (Google Gemini) | AI model used within Information Extractor       | Scrape single Workflow       | Information Extractor          |                                                                                                                                         |
| Main content               | Chain LLM (Langchain)             | Cleans and validates markdown content             | Scrape single Workflow       | Set content                  |                                                                                                                                         |
| Google Gemini Chat Model2   | Language Model Chat (Google Gemini) | AI model used within Main content node            | Scrape single Workflow       | Main content                  |                                                                                                                                         |
| Set content               | Set                              | Stores cleaned markdown text for summarization   | Main content                 | Summarization content         |                                                                                                                                         |
| Summarization content       | Google Gemini Chat Model          | Summarizes workflow purpose and tools in Italian | Set content                 | Merge                        |                                                                                                                                         |
| Merge                      | Merge                            | Combines extracted attributes and summaries      | Information Extractor, Summarization content | Add row                      |                                                                                                                                         |
| Add row                    | Google Sheets                    | Appends combined workflow data as a new row      | Merge                       | Loop Over Items               |                                                                                                                                         |
| Schedule Trigger            | Schedule Trigger                 | (Not connected, inactive) potential alternative trigger | -                          | -                             |                                                                                                                                         |
| Sticky Note                | Sticky Note                     | Notes and instructions                            | -                          | -                             | ## Extract Recently added WF on n8n Community with ScrapeGraph AI - This is an example of advanced automated data extraction and enrichment pipeline with [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia). Its primary purpose is to systematically scrape the n8n community workflows website, extract detailed information about recently added workflows, process that data using multiple AI models, and store the structured results in a Google Sheets spreadsheet. This workflow demonstrates a sophisticated use of n8n to move beyond simple API calls and into the realm of intelligent, AI-driven web scraping and data processing, turning unstructured website content into valuable, structured business intelligence. |
| Sticky Note1               | Sticky Note                     | Notes and instructions                            | -                          | -                             | ## STEPS - Register for FREE to [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia) and get the API Key - Install the ScrapeGraphAI community node in your n8n instance - Clone this [Google Sheet](https://docs.google.com/spreadsheets/d/1CnTq5kkHkdv8GPfGrjRkeK66R0XHOys25BF_Me2OX6I/edit?usp=sharing) and add it in "Add row" node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - No parameters needed  

2. **Add Scrape Main Page Node**  
   - Node Type: ScrapeGraphAI (markdownify resource)  
   - Parameters:  
     - Website URL: `https://n8n.io/workflows/`  
     - Render Heavy JS: Enabled  
   - Credentials: Configure with ScrapeGraphAI API key  
   - Connect Manual Trigger → Scrape Main Page  

3. **Add Google Gemini Chat Model Node**  
   - Node Type: Langchain LLM Chat (Google Gemini)  
   - Parameters: Default; no special options  
   - Credentials: Google Palm API (Gemini) configured  
   - Connect Scrape Main Page → Google Gemini Chat Model  

4. **Add Structured Output Parser Node**  
   - Node Type: Langchain Output Parser Structured  
   - Parameters:  
     - Auto Fix: Enabled  
     - Schema Type: Manual  
     - Input Schema: Object with property "workflows" as array of objects with properties `title` (string) and `url` (string)  
   - Connect Google Gemini Chat Model → Structured Output Parser  

5. **Add Extract "Recently added" Node**  
   - Node Type: Langchain Chain LLM  
   - Parameters:  
     - Prompt: Italian prompt instructing extraction of "title" and "url" for workflows in "Recently Added" section, output JSON array named "workflows"  
     - Output parser enabled  
   - Connect Structured Output Parser → Extract "Recently added"  

6. **Add Set Array Node**  
   - Node Type: Set  
   - Parameters: Assign variable `workflows` = `{{$json.output.workflows}}`  
   - Connect Extract "Recently added" → Set Array  

7. **Add Split Out Node**  
   - Node Type: Split Out  
   - Parameters: Split field `workflows` into individual items  
   - Connect Set Array → Split Out  

8. **Add Loop Over Items Node**  
   - Node Type: SplitInBatches  
   - Parameters: Default batch size (1 or as suited)  
   - Connect Split Out → Loop Over Items  

9. **Add Scrape Single Workflow Node**  
   - Node Type: ScrapeGraphAI (markdownify)  
   - Parameters:  
     - Website URL: `={{ $json.url }}` (dynamic from current item)  
     - Render Heavy JS: Enabled  
   - Credentials: ScrapeGraphAI  
   - Connect Loop Over Items → Scrape Single Workflow  

10. **Add Information Extractor Node**  
    - Node Type: Langchain Information Extractor  
    - Parameters:  
      - Text: `={{ $json.result }}` (markdown from scrape)  
      - Attributes: categories, author, price, title (translate to Italian), url, id — all required with descriptive prompts in Italian  
    - Connect Scrape Single Workflow → Information Extractor  

11. **Add Main Content Node**  
    - Node Type: Langchain Chain LLM  
    - Parameters:  
      - Text: `={{ $json.result }}`  
      - Prompt: Italian instructions to clean markdown, remove superfluous content, validate markdown  
    - Connect Scrape Single Workflow → Main Content  

12. **Add Set Content Node**  
    - Node Type: Set  
    - Parameters: Store cleaned markdown text from Main Content output field `text` into variable `content`  
    - Connect Main Content → Set Content  

13. **Add Summarization Content Node**  
    - Node Type: Google Gemini Chat Model (Langchain)  
    - Parameters:  
      - Model ID: `models/gemini-2.5-flash`  
      - System Message: Italian instruction to summarize workflow purpose and tools concisely in plain text  
      - Input: `={{ $json.content }}` (cleaned markdown)  
    - Credentials: Google Palm API  
    - Connect Set Content → Summarization Content  

14. **Add Merge Node**  
    - Node Type: Merge  
    - Parameters:  
      - Mode: Combine  
      - Combine By: combineAll  
    - Connect Information Extractor (output) → Merge (input 2)  
    - Connect Summarization Content → Merge (input 1)  

15. **Add Google Sheets "Add row" Node**  
    - Node Type: Google Sheets  
    - Parameters:  
      - Operation: Append  
      - Document ID: Use your Google Sheets document ID  
      - Sheet Name: Use the first sheet (e.g. `gid=0`)  
      - Columns Mapping: Map from merged JSON fields:  
        - ID ← output.id  
        - URL ← output.url  
        - PRICE ← output.price  
        - TITLE ← output.title  
        - AUTHOR ← output.author  
        - SUMMARY ← summarization content text (path: `$('Summarization content').item.json.content.parts[0].text`)  
        - CATEGORIES ← output.categories  
    - Credentials: Google Sheets OAuth2 account configured  
    - Connect Merge → Add row  

16. **Loop Back**  
    - Connect Add row output back to Loop Over Items to continue processing batch  

17. **(Optional) Add Schedule Trigger Node**  
    - Node Type: Schedule Trigger  
    - Parameters: Set desired interval (e.g., daily)  
    - Connect Schedule Trigger → Scrape Main Page if automation desired without manual trigger  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates advanced automated data extraction and enrichment pipeline using [ScrapeGraphAI](https://dashboard.scrapegraphai.com/?via=n3witalia). It moves beyond simple API calls into AI-driven scraping and processing, producing structured business intelligence for n8n community workflows. | Sticky Note covering overview and purpose of the workflow                                                           |
| Steps to set up include registering free at ScrapeGraphAI for API key, installing ScrapeGraphAI community node in n8n, and cloning a provided Google Sheet to store results: https://docs.google.com/spreadsheets/d/1CnTq5kkHkdv8GPfGrjRkeK66R0XHOys25BF_Me2OX6I/edit?usp=sharing                                                  | Sticky Note1 covering setup instructions and useful links                                                           |

---

**Disclaimer:** The text and workflow provided are exclusively derived from an automated n8n workflow. The process respects all applicable policies and handles only legal, publicly available data.

---