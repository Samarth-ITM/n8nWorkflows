Web Research Summarizer with Decodo Scraper, Google Gemini AI & Sheets Integration

https://n8nworkflows.xyz/workflows/web-research-summarizer-with-decodo-scraper--google-gemini-ai---sheets-integration-11389


# Web Research Summarizer with Decodo Scraper, Google Gemini AI & Sheets Integration

### 1. Workflow Overview

This workflow, titled **AI Research Scraper & Summary Generator**, automates the process of extracting, analyzing, and summarizing web research articles or web pages. It is designed for researchers, content creators, and knowledge managers who need to convert a list of URLs into structured, AI-generated summaries with insights, stored conveniently in Google Sheets.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Fetches a list of URLs to process from a Google Sheets document.
- **1.2 Scraping & Cleaning:** Uses Decodo to scrape webpage content and a JavaScript function to clean and extract relevant data (title, domain, text).
- **1.3 AI Processing:** Sends cleaned text and metadata to a Google Gemini-based AI agent, which returns structured JSON summaries and insights.
- **1.4 Data Parsing & Formatting:** Parses the AI JSON output and ensures all fields are correctly extracted and formatted.
- **1.5 Output Storage:** Appends the structured summary data into another Google Sheets document for later review or use.
- **1.6 Workflow Trigger & Control:** Manual trigger to start the workflow and batch processing logic to handle multiple URLs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Retrieves all URLs listed in a designated Google Sheets tab (`input` sheet) to prepare them for sequential processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet  
  - Loop Over Items

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual trigger  
    - Role: Initiates execution manually  
    - Config: No special parameters  
    - Inputs: None (start node)  
    - Outputs: Connects to “Get row(s) in sheet”  
    - Edge cases: User must manually trigger; no automation or webhook trigger available.

  - **Get row(s) in sheet**  
    - Type: Google Sheets (Read rows)  
    - Role: Reads all URLs from the `input` sheet (sheet GID=0) in the specified Google Sheets document  
    - Config: Document ID and sheet name set to the input sheet  
    - Inputs: Triggered by manual trigger  
    - Outputs: Array of rows with URLs to process  
    - Edge cases:  
      - Empty or missing URL column leads to no processing  
      - Google Sheets API limits may apply  
      - Credential errors if not connected or authorized properly

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes URLs one-by-one or in small batches for better control and API rate management  
    - Config: Default batch size (not explicitly set, defaults to 1)  
    - Inputs: List of URLs from Google Sheets  
    - Outputs: Sends each URL individually to the scraper node  
    - Edge cases:  
      - Large datasets may increase total runtime  
      - Batch size tuning may be needed for performance or API limits

---

#### 2.2 Scraping & Cleaning

- **Overview:**  
  For each URL, the workflow scrapes the entire webpage content using Decodo, then cleans and extracts key information such as title, source domain, article text, and date.

- **Nodes Involved:**  
  - Decodo  
  - Code in JavaScript (cleaning and extraction)

- **Node Details:**

  - **Decodo**  
    - Type: Decodo node (third-party scraper)  
    - Role: Performs universal scraping of the URL content, extracting raw HTML and metadata  
    - Config: Operation set to "universal" (general scraping mode)  
    - Inputs: Single URL from Loop Over Items  
    - Outputs: Raw scraped HTML content with metadata  
    - Credentials: Requires Decodo API key (must be added in n8n credentials)  
    - Edge cases:  
      - API key missing or invalid leads to auth errors  
      - Target page structure may vary causing incomplete scraping  
      - Network or timeout failures

  - **Code in JavaScript**  
    - Type: Function node running custom JavaScript  
    - Role: Parses Decodo raw HTML result, extracts:  
      - Article title from `<title>` tag  
      - Plain text from HTML (removes scripts, styles, tags, decodes entities)  
      - Source domain extracted from URL  
      - Adds current date as saved date  
    - Key expressions: Uses regex to extract title and clean HTML  
    - Inputs: Raw Decodo output  
    - Outputs: JSON objects with fields: `url`, `fuente` (source), `titulo` (title), `article_text`, `fecha_guardado` (saved date)  
    - Edge cases:  
      - Missing or malformed HTML results in empty strings  
      - URL parsing errors handled by try-catch, returns empty string on failure

---

#### 2.3 AI Processing

- **Overview:**  
  The cleaned article data is sent to an AI agent powered by Google Gemini that analyzes the text and returns a fully structured JSON summary including metadata, key insights, and content ideas.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent node)  
  - Google Gemini Chat Model (AI language model node)  
  - Code in JavaScript1 (parses AI JSON output)

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Sends prompt and cleaned article text to AI for structured JSON response  
    - Configuration:  
      - Prompt instructs AI to return JSON with exact keys (url, title, source, published_date, saved_date, resource_type, main_topic, level, three_key_insights, short_summary, content_idea, language)  
      - Uses passed-in data fields via expressions to fill prompt template  
      - System message guides AI behavior to produce structured JSON only  
    - Inputs: Cleaned article JSON from JavaScript node  
    - Outputs: AI-generated JSON string under field `output`  
    - Edge cases:  
      - AI response may be malformed JSON  
      - API or quota errors with Google Gemini  
      - Latency or timeout on AI calls

  - **Google Gemini Chat Model**  
    - Type: Langchain LM Chat Google Gemini node  
    - Role: Backend AI model for the Langchain agent  
    - Inputs: Receives prompt and data from AI Agent node  
    - Outputs: AI-generated response text  
    - Edge cases: Same as AI Agent node (API limits, errors)

  - **Code in JavaScript1**  
    - Type: Function node  
    - Role: Parses AI response JSON string, extracts all expected fields into structured JSON item  
    - Key expressions: JSON.parse on AI output string, fallback error capture  
    - Inputs: AI Agent output with raw JSON string  
    - Outputs: Parsed JSON with keys: url, title, source, published_date, saved_date, resource_type, main_topic, three_key_insights, short_summary, content_idea, language  
    - Edge cases:  
      - Parsing failure returns error field with original output  
      - Missing fields handled by default empty strings

---

#### 2.4 Output Storage

- **Overview:**  
  Appends the final structured summary data to an output Google Sheets tab for further use or export.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets (Append row)  
    - Role: Adds one row per summarized article into the `output` sheet of the specified spreadsheet  
    - Config: Maps each JSON field to a corresponding column (url, title, topic, source, summary, key_ideas, text_type, main_topic, published_date)  
    - Inputs: Parsed structured data from AI JSON parsing node  
    - Outputs: None (end node)  
    - Edge cases:  
      - Sheet access permissions errors  
      - Column schema mismatches or missing columns  
      - API rate limits or quota exceeded

---

#### 2.5 Workflow Trigger & Control

- **Overview:**  
  Allows manual starting of the workflow and controls iteration over multiple URLs.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Loop Over Items

- **Node Details:**

  - Already covered in 2.1 Input Reception block.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                         | Input Node(s)                   | Output Node(s)              | Sticky Note                                                                                   |
|-------------------------|------------------------------------|---------------------------------------|--------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                     | Starts workflow manually               | None                           | Get row(s) in sheet          |                                                                                               |
| Get row(s) in sheet     | Google Sheets (Read rows)           | Reads URLs list from input sheet      | When clicking ‘Execute workflow’ | Loop Over Items              | ###  **Get Links**  Reads all the URLs from the `input` sheet and prepares them one by one.   |
| Loop Over Items         | SplitInBatches                     | Processes URLs in batches              | Get row(s) in sheet            | Decodo                      |                                                                                               |
| Decodo                  | Decodo Scraper Node                | Scrapes webpage content                | Loop Over Items                | Code in JavaScript           | ### **Scrape Content** Uses [Decodo](https://visit.decodo.com/raqXGD) to scrape webpage text. |
| Code in JavaScript      | Function (JavaScript)              | Cleans and extracts article info      | Decodo                        | AI Agent                    | ###  **Clean Text**  Removes HTML noise, extracts title, domain, and text for AI analysis.    |
| AI Agent                | Langchain Agent                   | Sends cleaned text to AI for summary  | Code in JavaScript             | Code in JavaScript1          | ### **Generate Summary (AI)** Sends clean text to AI to create JSON summary and insights.     |
| Google Gemini Chat Model| Langchain LM Chat (Google Gemini) | AI language model backend              | AI Agent (ai_languageModel)    | AI Agent                    |                                                                                               |
| Code in JavaScript1     | Function (JavaScript)              | Parses AI JSON output into structured data | AI Agent                     | Append row in sheet          | ### **Parse & Format Data** Parses AI JSON output, ensures all fields are properly set.       |
| Append row in sheet     | Google Sheets (Append row)          | Saves summarized data to output sheet | Code in JavaScript1            | Loop Over Items (loop back) | ###  **Save Results**  Saves final info to the `output` Google Sheet for review.              |
| Sticky Note             | Sticky Note                       | Documentation and instructions         | None                         | None                        | ###  **AI Research Scraper & Summary Generator** How it works, inputs, outputs, and setup.    |
| Sticky Note1            | Sticky Note                       | Documentation                         | None                         | None                        | ###  **Get Links**  Reads URLs from input sheet.                                              |
| Sticky Note2            | Sticky Note                       | Documentation                         | None                         | None                        | ### **Scrape Content**  Uses Decodo to extract webpage content.                               |
| Sticky Note3            | Sticky Note                       | Documentation                         | None                         | None                        | ###  **Save Results** Saves data into output Google Sheet.                                   |
| Sticky Note4            | Sticky Note                       | Documentation                         | None                         | None                        | ### **Parse & Format Data** Ensures AI output is clean and structured.                        |
| Sticky Note5            | Sticky Note                       | Documentation                         | None                         | None                        | ### **Generate Summary (AI)** Sends clean text to AI for summary generation.                  |
| Sticky Note6            | Sticky Note                       | Documentation                         | None                         | None                        | ###  **Clean Text** Cleans HTML to extract useful article data for AI.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually  
   - Connect to next node.

2. **Add Google Sheets 'Get Rows' Node**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Configure with your Google Sheets credentials.  
   - Set Document ID to your spreadsheet containing URLs.  
   - Set Sheet Name to your input tab (e.g., `input`, GID=0).  
   - Connect manual trigger to this node.

3. **Add SplitInBatches Node (Loop Over Items)**  
   - Type: SplitInBatches  
   - Purpose: Process URLs one by one (default batch size 1)  
   - Connect "Get Rows" output to this node.

4. **Add Decodo Node**  
   - Type: Decodo Scraper Node (from Decodo's n8n integration)  
   - Operation: Universal (general scraping)  
   - Insert your Decodo API credentials in n8n credentials manager.  
   - Connect SplitInBatches output to Decodo input.

5. **Add Function Node 'Code in JavaScript' to Clean and Extract Data**  
   - Paste the provided JavaScript code that extracts the `<title>`, cleans the HTML to text, extracts domain from URL, and adds current date.  
   - Connect Decodo output to this node.

6. **Add Langchain Agent Node 'AI Agent'**  
   - Set the prompt exactly as provided, including instruction to output strictly a JSON object with specified keys and structure, using expressions to inject the fields from the previous node (url, source, title, saved date, article text).  
   - Connect the Function Node output to this node.

7. **Add Google Gemini Chat Model Node**  
   - This node acts as the AI language model backend.  
   - Connect this node to the AI Agent’s ai_languageModel input.  
   - Make sure you have proper API credentials and access for Google Gemini in n8n.

8. **Add Function Node 'Code in JavaScript1' to Parse AI JSON Output**  
   - Paste the JavaScript code that parses the AI Agent’s JSON string output, extracts all fields, or returns an error if parsing fails.  
   - Connect AI Agent output to this node.

9. **Add Google Sheets 'Append Row' Node**  
   - Type: Google Sheets  
   - Operation: Append Row  
   - Configure with your Google Sheets credentials.  
   - Set Document ID to your spreadsheet for output.  
   - Set Sheet Name to your output tab (e.g., `output`, GID=60764768).  
   - Map fields from the parsed JSON to the sheet columns: url, title, topic, source, summary, key_ideas, text_type, main_topic, published_date.  
   - Connect Function Node (parsing) output to this node.

10. **Connect Append Row Output Back to SplitInBatches**  
    - This looping connection allows processing the next batch until all URLs are processed.

11. **Add Sticky Notes (Optional for Documentation)**  
    - Add descriptive sticky notes near relevant nodes for clarity, including links to Decodo and explanations of each block.

12. **Credentials Setup**  
    - Google Sheets: Connect OAuth2 credentials with access to both input and output sheets.  
    - Decodo: Add API key credential for Decodo node.  
    - Google Gemini: Setup API credentials for Google Gemini AI model access.

13. **Run the Workflow**  
    - Add URLs to the `input` Google Sheet under the `url` column.  
    - Execute manually using the trigger node.  
    - Monitor progress and review output in the `output` Google Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow uses [Decodo](https://visit.decodo.com/raqXGD) for web scraping, which specializes in extracting main article text and metadata from webpages.                                                                                             | Decodo official site: https://visit.decodo.com/raqXGD                                                                    |
| The AI summarization leverages Google Gemini via n8n Langchain integration, providing structured JSON summaries ideal for research and content generation workflows.                                                                                      | Gemini AI model integration in n8n                                                                                        |
| The workflow is designed for batch processing to handle multiple URLs efficiently, with error handling in parsing AI outputs to avoid crashes due to malformed responses.                                                                                 |                                                                                                                          |
| The output Google Sheet schema includes fields like URL, title, main topic, source, summary, key ideas, text type, and published date for easy filtering and indexing.                                                                                     |                                                                                                                          |
| The included sticky notes in the workflow provide helpful instructions and references for setup and usage.                                                                                                                                                | Sticky notes visible inside the n8n workflow editor                                                                       |
| When deploying, ensure API quotas and limits are respected for Google Sheets, Decodo, and Google Gemini to avoid interruptions.                                                                                                                           |                                                                                                                          |
| For better results, maintain clean and up-to-date URLs in the input sheet and validate API keys before running the workflow.                                                                                                                             |                                                                                                                          |

---

This detailed documentation enables understanding, reproduction, and modification of the AI Research Scraper & Summary Generator workflow, making it accessible for both advanced users and automated agents.