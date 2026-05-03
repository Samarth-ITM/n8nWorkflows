Generate SEO Meta Tags with Gemini AI & Competitor Analysis using Google Sheets

https://n8nworkflows.xyz/workflows/generate-seo-meta-tags-with-gemini-ai---competitor-analysis-using-google-sheets-9750


# Generate SEO Meta Tags with Gemini AI & Competitor Analysis using Google Sheets

---

### 1. Workflow Overview

This workflow automates the generation of SEO meta titles and descriptions using Google Sheets as the data source and sink, combined with Google Gemini AI models and live competitor analysis through Google SERP data. It targets SEO professionals or content strategists who want to optimize web pages systematically by leveraging AI insights and competitor intelligence.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Filtering**: Detects new URLs in a Google Sheet marked with status "New" and loads them for processing.
- **1.2 URL Processing Loop**: Iterates over each URL individually, marking their status as "Generating..." to ensure exclusive processing.
- **1.3 Own Page Analysis**: Scrapes the current webpage, extracts meta data, and uses AI to analyze the page's SEO-relevant content attributes.
- **1.4 Competitor Intelligence Gathering**: Queries Google search results for the primary keyword, filters competitors’ data intelligently, and runs competitor pattern analysis AI.
- **1.5 SEO Meta Generation**: Combines own page data and competitor patterns to generate optimized meta titles and descriptions under strict character limits.
- **1.6 Result Write-Back and Loop Continuation**: Writes the newly generated meta data and status back to the Google Sheet and continues processing the next URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:**  
  This block triggers the workflow when a new row is added to a specific Google Sheet and retrieves all URLs marked with the status "New" for processing.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Get row(s) in sheet1  
  - Loop Over Items (entry point for the loop)

- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger node for Google Sheets row addition  
    - Configuration: Triggers on new rows added every minute in the "Control Panel" sheet of the specified Google Sheet document  
    - Input: None (trigger)  
    - Output: Newly added row(s) data  
    - Failure Modes: OAuth issues, API quota limits, connectivity issues

  - **Get row(s) in sheet1**  
    - Type: Google Sheets node (Read)  
    - Configuration: Reads rows from "Control Panel" sheet filtering rows where `Status` equals "New"  
    - Input: Trigger data (Google Sheets Trigger)  
    - Output: Array of rows matching the filter  
    - Failure Modes: Sheet access permission errors, incorrect filter column names, API quota

  - **Loop Over Items**  
    - Type: Split In Batches node  
    - Configuration: Splits the array of rows into single-item batches for sequential processing  
    - Input: Rows from Get row(s) in sheet1  
    - Output: Single row per batch  
    - Failure Modes: Large batch size causing delays, improper input format

#### 2.2 URL Processing Loop

- **Overview:**  
  Processes each URL one at a time, immediately updates its status to prevent duplicate processing, and prepares for analysis.

- **Nodes Involved:**  
  - Update row in sheet1 (mark status "Generating - wait for a few minutes")

- **Node Details:**

  - **Update row in sheet1**  
    - Type: Google Sheets node (Update)  
    - Configuration: Updates the current row's `Status` to "Generating - wait for a few minutes" based on matching `URL` column  
    - Input: Current batch item (single URL row)  
    - Output: Confirmation of update  
    - Failure Modes: Row not found for update, permission errors

#### 2.3 Own Page Analysis

- **Overview:**  
  Scrapes the target URL's webpage, extracts current meta title and description, then uses Google Gemini AI to analyze the page content for SEO insights.

- **Nodes Involved:**  
  - Scrape Website (HTTP Request)  
  - HTML (HTML Extract)  
  - Edit Fields (Set node)  
  - Google Gemini Chat Model (AI LLM)  
  - Structured Output Parser  
  - AI Analyzer (LangChain LLM Chain)

- **Node Details:**

  - **Scrape Website**  
    - Type: HTTP Request  
    - Configuration: Calls ScrapingDog API with dynamic rendering enabled to retrieve webpage HTML of current URL  
    - Input: URL from current batch item  
    - Output: Raw HTML content in `data` key  
    - Failure Modes: API key invalid, page unreachable, timeout, CAPTCHA blocking, dynamic content load failure

  - **HTML**  
    - Type: HTML Extract  
    - Configuration: Extracts `<title>` tag content and meta description (`meta[name="description"]`) content attribute from the scraped HTML  
    - Input: Scraped HTML (`data`)  
    - Output: `current_title` and `description` as arrays of strings  
    - Failure Modes: Missing tags, malformed HTML

  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Converts extracted arrays into plain strings for further processing by selecting first elements  
    - Input: Extracted HTML fields  
    - Output: Cleaned `current_title` and `description` strings  
    - Failure Modes: Empty arrays, null values

  - **Google Gemini Chat Model**  
    - Type: AI Language Model (Google Gemini LLM)  
    - Configuration: Uses a prompt to analyze webpage text and output a structured JSON with keys like `primary_keyword`, `semantic_keyword_cluster`, `search_intent`, `target_audience`, `content_angle`, `content_summary`  
    - Input: Webpage text from Scrape Website node  
    - Output: Raw AI response text (JSON string)  
    - Credentials: Google Palm API with valid authentication  
    - Failure Modes: API quota, prompt errors, malformed AI output, connectivity

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Configuration: Strict JSON schema defined to parse AI output into structured fields, enforcing required keys and types  
    - Input: Raw AI output from Google Gemini Chat Model  
    - Output: Parsed JSON object with SEO analysis  
    - Failure Modes: Parsing errors if AI output deviates from schema

  - **AI Analyzer (Chain LLM)**  
    - Type: LangChain Chain LLM  
    - Configuration: Receives parsed output, forwards primary keyword to next block  
    - Input: Parsed SEO analysis  
    - Output: Structured data for competitor search  
    - Failure Modes: Internal errors, chaining issues

#### 2.4 Competitor Intelligence Gathering

- **Overview:**  
  Searches Google for the primary keyword using SerpApi, filters real competitors intelligently using semantic keywords, then analyzes competitor patterns via AI.

- **Nodes Involved:**  
  - Googl SERP (HTTP Request)  
  - Code1 (JavaScript filtering)  
  - Google Gemini Chat Model1 (AI LLM)  
  - Competitor Analysis (LangChain Chain LLM)  
  - Code (JavaScript JSON extractor)

- **Node Details:**

  - **Googl SERP**  
    - Type: HTTP Request  
    - Configuration: Calls SerpApi Google search engine with query set to the `primary_keyword` extracted earlier  
    - Input: `primary_keyword` from AI Analyzer  
    - Output: Raw text response of Google SERP JSON data  
    - Failure Modes: API key invalid, rate limits, network issues

  - **Code1**  
    - Type: Code (JavaScript)  
    - Configuration: Parses raw SERP JSON string, extracts organic results, filters entries by matching any of the AI-generated semantic keywords (after removing stop words), limits to top 7 relevant competitors  
    - Input: Raw SERP data and AI Analyzer output  
    - Output: Cleaned list of competitor data (title, description, link)  
    - Failure Modes: JSON parsing errors, empty data, missing fields

  - **Google Gemini Chat Model1**  
    - Type: AI Language Model (Google Gemini LLM)  
    - Configuration: Analyzes competitor titles and descriptions to detect common SEO patterns and tones, outputting a strict JSON object with a summary string of patterns  
    - Input: Filtered competitor data from Code1  
    - Output: Raw AI JSON string describing competitor patterns  
    - Credentials: Google Palm API  
    - Failure Modes: Parsing errors, API issues

  - **Competitor Analysis**  
    - Type: LangChain Chain LLM  
    - Configuration: Parses and validates competitor patterns AI output  
    - Input: Raw AI output from Google Gemini Chat Model1  
    - Output: Structured competitor pattern insights  
    - Failure Modes: Parsing failures, invalid JSON output

  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration: Extracts and parses JSON from the messy AI output text from Competitor Analysis, handling errors and fallback messages  
    - Input: Raw AI text output  
    - Output: Clean JSON object with competitor patterns or error notices  
    - Failure Modes: Missing JSON, parsing errors

#### 2.5 SEO Meta Generation

- **Overview:**  
  Uses the combined intelligence from own page analysis and competitor patterns to generate a length-constrained optimized meta title and description using AI.

- **Nodes Involved:**  
  - Google Gemini Chat Model2 (AI LLM)  
  - Master Generator (LangChain Chain LLM)  
  - Code2 (JavaScript JSON extractor)

- **Node Details:**

  - **Google Gemini Chat Model2**  
    - Type: AI Language Model (Google Gemini LLM)  
    - Configuration: Receives all prior insights and is prompted to generate SEO-optimized meta tags strictly under 60 and 160 characters respectively  
    - Input: Own page info and competitor patterns  
    - Output: Raw JSON text output of generated meta tags  
    - Credentials: Google Palm API  
    - Failure Modes: API quota, malformed output, prompt failure

  - **Master Generator**  
    - Type: LangChain Chain LLM  
    - Configuration: Wraps the generation prompt and passes output to parsing node  
    - Input: AI Gemini Chat Model2 output  
    - Output: Raw generated meta tags text  
    - Failure Modes: Internal AI errors

  - **Code2**  
    - Type: Code (JavaScript)  
    - Configuration: Parses JSON string from AI output, handles parse errors gracefully by returning error placeholders in meta fields  
    - Input: Raw AI output text  
    - Output: Clean JSON object with `optimized_title` and `optimized_meta` fields  
    - Failure Modes: Parsing errors, no JSON found

#### 2.6 Result Write-Back and Loop Continuation

- **Overview:**  
  Updates the Google Sheet row with generated SEO meta tags, current meta data, competitor insights, and marks the status as "Generated". Then continues to the next URL.

- **Nodes Involved:**  
  - Update row in sheet (final update)  
  - Loop Over Items (continuation)

- **Node Details:**

  - **Update row in sheet**  
    - Type: Google Sheets node (Update)  
    - Configuration: Writes back generated meta title & description, current meta, competitor pattern summary, and updates status to "Generated" based on matching URL  
    - Input: Final generated SEO data and current meta info  
    - Output: Confirmation of update  
    - Failure Modes: Row not found, permission errors, write conflicts

  - **Loop Over Items** (second output)  
    - Type: Split In Batches node (loop continuation)  
    - Configuration: Automatically proceeds to next batch item  
    - Input: Confirmation from update node  
    - Output: Next URL to process or ends if none left  
    - Failure Modes: Infinite loop if misconfigured, batch size mismanagement

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                             | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                                                  |
|-------------------------|--------------------------------|---------------------------------------------|---------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger    | Trigger                        | Workflow start on new Google Sheets row     | None                      | Get row(s) in sheet1       | # 1. START HERE: Get the To-Do List - triggers on new rows with status "New"                                                   |
| Get row(s) in sheet1    | Google Sheets (Read)           | Fetch rows with Status = "New"               | Google Sheets Trigger      | Loop Over Items            | # 1. START HERE: Get the To-Do List - filters URLs with status "New"                                                          |
| Loop Over Items          | Split In Batches               | Process each URL one-by-one                   | Get row(s) in sheet1       | Update row in sheet1, (loop continuation) | # 2. Process URLs One-by-One - loops over URLs and marks status                                                               |
| Update row in sheet1     | Google Sheets (Update)         | Mark URL status as "Generating - wait..."   | Loop Over Items            | Scrape Website             | # 2. Process URLs One-by-One - updates status to prevent duplicate processing                                                  |
| Scrape Website           | HTTP Request                  | Scrape the target URL via ScrapingDog API    | Update row in sheet1       | HTML                      | # 3. PHASE 1: Analyze Our Own Page - scrape website content                                                                   |
| HTML                    | HTML Extract                  | Extract current meta title and description   | Scrape Website             | Edit Fields                | # 3. PHASE 1: Analyze Our Own Page - extract meta tags                                                                         |
| Edit Fields              | Set                           | Clean extracted meta fields                   | HTML                       | AI Analyzer                | # 3. PHASE 1: Analyze Our Own Page - prepare fields for AI                                                                     |
| Google Gemini Chat Model | AI Language Model (LLM)        | Analyze page content for SEO insights         | Edit Fields                | Structured Output Parser   | # 3. PHASE 1: Analyze Our Own Page - AI deep dive analysis                                                                     |
| Structured Output Parser | LangChain Output Parser        | Parse AI output JSON to structured data       | Google Gemini Chat Model   | AI Analyzer                | # 3. PHASE 1: Analyze Our Own Page - structured parsing                                                                         |
| AI Analyzer              | LangChain Chain LLM            | Provide structured SEO analysis data           | Structured Output Parser   | Googl SERP                 | # 3. PHASE 1: Analyze Our Own Page - passes primary keyword forward                                                            |
| Googl SERP               | HTTP Request                  | Fetch Google SERP results for primary keyword | AI Analyzer                | Code1                     | # 4. PHASE 2: Spy on Competitors - get Google search results                                                                  |
| Code1                   | Code (JavaScript)              | Filter and clean competitor results           | Googl SERP, AI Analyzer    | Competitor Analysis        | # 4. PHASE 2: Spy on Competitors - dynamic filtering based on semantic keywords                                                |
| Google Gemini Chat Model1| AI Language Model (LLM)        | Analyze competitor data for patterns           | Code1                      | Competitor Analysis        | # 5. Find the Competitor Strategies - AI pattern detection                                                                     |
| Competitor Analysis      | LangChain Chain LLM            | Parse competitor pattern AI output              | Google Gemini Chat Model1  | Code                      | # 5. Find the Competitor Strategies - structured competitor insights                                                            |
| Code                     | Code (JavaScript)              | Extract and parse competitor AI JSON output    | Competitor Analysis        | Master Generator           | # 5. Find the Competitor Strategies - JSON extraction and error handling                                                       |
| Google Gemini Chat Model2| AI Language Model (LLM)        | Generate optimized meta title & description    | Code                       | Master Generator           | # 6. PHASE 3: The Master Generator - final SEO meta generation                                                                 |
| Master Generator         | LangChain Chain LLM            | Wrap generation prompt and pass output         | Google Gemini Chat Model2  | Code2                     | # 6. PHASE 3: The Master Generator - synthesizes all intel                                                                     |
| Code2                    | Code (JavaScript)              | Parse final AI JSON output with error handling | Master Generator           | Update row in sheet        | # 6. PHASE 3: The Master Generator - clean JSON parsing                                                                         |
| Update row in sheet      | Google Sheets (Update)         | Write generated meta data back to Google Sheet | Code2                      | Loop Over Items            | # 7. The Final Write-Back - update sheet, set status to "Generated" and continue loop                                          |
| Loop Over Items (2nd out)| Split In Batches               | Continue processing next URL in batch           | Update row in sheet        | Update row in sheet1       | # 2. Process URLs One-by-One - continue looping                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Trigger on event: Row Added  
   - Document: Select your Google Sheet by ID  
   - Sheet Name: "Control Panel" (or your equivalent)  
   - Poll interval: Every minute  
   - Credentials: Configure Google Sheets OAuth2

2. **Create Google Sheets Node (Get row(s) in sheet1)**  
   - Operation: Read Rows  
   - Document and Sheet: same as trigger  
   - Filter: Status column equals "New"  
   - Credentials: same Google Sheets OAuth2 account  
   - Connect trigger’s output to this node

3. **Create Split In Batches Node (Loop Over Items)**  
   - Operation: Split input into batches of size 1 (default)  
   - Connect from Get row(s) in sheet1 node

4. **Create Google Sheets Node (Update row in sheet1)**  
   - Operation: Update Row  
   - Document and Sheet: same as above  
   - Matching column: URL  
   - Columns to update: Status = "Generating - wait for a few minutes"  
   - Credentials: Google Sheets OAuth2  
   - Connect from Loop Over Items node (first output)

5. **Create HTTP Request Node (Scrape Website)**  
   - URL: `https://api.scrapingdog.com/scrape`  
   - Query parameters:  
     - api_key: Your ScrapingDog API key  
     - url: `={{ $json.URL }}` (current URL)  
     - dynamic: true  
   - Method: GET  
   - Connect from Update row in sheet1

6. **Create HTML Extract Node (HTML)**  
   - Operation: Extract Content  
   - Extraction rules:  
     - Key: current_title, CSS Selector: `title`, return as array  
     - Key: description, CSS Selector: `meta[name="description"]`, attribute: content, return as array  
   - Connect from Scrape Website

7. **Create Set Node (Edit Fields)**  
   - Set variables:  
     - current_title = first element of `current_title` array from HTML node  
     - description = first element of `description` array from HTML node  
   - Connect from HTML

8. **Create Google Gemini Chat Model Node (AI Analyzer)**  
   - Credentials: Google Palm API with valid key  
   - Prompt: Insert the provided prompt that extracts SEO insights from webpage text  
   - Input: Pass `data` from Scrape Website node  
   - Output parser: Use Structured Output Parser node next  
   - Connect from Edit Fields

9. **Create Structured Output Parser Node**  
   - Schema: Copy the provided JSON schema for SEO analysis keys  
   - Connect from Google Gemini Chat Model (AI Analyzer)

10. **Create HTTP Request Node (Googl SERP)**  
    - URL: `https://serpapi.com/search`  
    - Query parameters:  
      - api_key: Your SerpApi key  
      - engine: "google"  
      - q: `={{ $json.output.primary_keyword }}` (from AI Analyzer)  
      - google_domain: "google.com"  
      - hl: "en"  
    - Method: GET  
    - Connect from AI Analyzer

11. **Create Code Node (Code1)**  
    - JavaScript: Implement code to parse SERP JSON, filter organic results by semantic keywords, remove stop words, and limit to top 7 competitors  
    - Connect from Googl SERP

12. **Create Google Gemini Chat Model Node (Competitor Analysis)**  
    - Credentials: Google Palm API  
    - Prompt: Analyze competitor titles and descriptions for SEO patterns  
    - Input: Use filtered competitor data from Code1  
    - Connect from Code1

13. **Create LangChain Chain LLM Node (Competitor Analysis)**  
    - Input: Direct from Google Gemini Chat Model1  
    - Connect to a Code Node (Code) for JSON extraction

14. **Create Code Node (Code)**  
    - JavaScript: Extract JSON from AI text output, handle parse errors  
    - Connect from Competitor Analysis

15. **Create Google Gemini Chat Model Node (Master Generator)**  
    - Credentials: Google Palm API  
    - Prompt: Generate optimized meta title & description under strict length limits, using all collected data  
    - Connect from Code

16. **Create LangChain Chain LLM Node (Master Generator)**  
    - Connect from Google Gemini Chat Model2  
    - Connect to Code Node (Code2)

17. **Create Code Node (Code2)**  
    - JavaScript: Parse final AI output JSON, handle errors  
    - Connect from Master Generator

18. **Create Google Sheets Node (Update row in sheet)**  
    - Operation: Update Row  
    - Matching column: URL  
    - Columns to update:  
      - Status = "Generated"  
      - Current Meta Title, Current Meta Description, Generated Meta Title, Generated Meta Description, Ranking Factor (from competitor insights)  
    - Credentials: Google Sheets OAuth2  
    - Connect from Code2

19. **Connect Update row in sheet back to Loop Over Items**  
    - Connect the first output of Update row in sheet back to the second input of Loop Over Items (to continue processing next URL)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The workflow uses a combination of ScrapingDog and SerpApi for web scraping and SERP data, respectively; valid API keys are required for both services.             | ScrapingDog: https://scrapingdog.com, SerpApi: https://serpapi.com                                                |
| Google Gemini AI is used for all NLP tasks, including content analysis, competitor analysis, and meta tag generation. Proper Google Palm API credentials are needed. | Google Palm API documentation: https://developers.generativeai.google                                            |
| The workflow enforces strict character limits on meta title (under 60 chars) and meta description (under 160 chars) to comply with SEO best practices.               | SEO guidelines: https://moz.com/learn/seo/title-tag, https://moz.com/learn/seo/meta-description                   |
| The dynamic competitor filtering leverages a "bag of words" built from semantic keyword clusters to exclude irrelevant SERP results, improving relevance.             | Custom JavaScript code for dynamic filtering is critical for precise competitor analysis.                          |
| Error handling is implemented in code nodes parsing AI responses to avoid workflow crashes due to malformed or missing JSON output from AI nodes.                    | Standard practice for robust AI integration in n8n workflows                                                      |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---