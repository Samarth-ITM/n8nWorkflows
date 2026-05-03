Scrape any web page into structured JSON data with ScrapeNinja and AI

https://n8nworkflows.xyz/workflows/scrape-any-web-page-into-structured-json-data-with-scrapeninja-and-ai-2812


# Scrape any web page into structured JSON data with ScrapeNinja and AI

### 1. Workflow Overview

This workflow automates the extraction of structured JSON data from any web page by combining web scraping with AI-generated custom extraction code. It is designed to address the common issue of web scrapers breaking due to changes in page layouts by dynamically generating JavaScript extraction functions using a Large Language Model (Google Gemini).

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the trigger to start scraping a predefined URL.
- **1.2 Web Scraping:** Uses the ScrapeNinja community node to fetch raw HTML content from the target webpage.
- **1.3 HTML Cleanup:** Cleans the scraped HTML to prepare it for extraction.
- **1.4 AI Processing:** Sends the cleaned HTML to Google Gemini to generate a custom JavaScript function that extracts relevant data using Cheerio.js.
- **1.5 Data Extraction:** Executes the AI-generated JavaScript extractor function in a sandboxed environment against the cleaned HTML to produce structured JSON data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, serving as the entry point to trigger the scraping process.

- **Nodes Involved:**  
  - Generate custom web scraper (Manual Trigger)

- **Node Details:**  
  - **Generate custom web scraper**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on demand.  
    - Configuration: No parameters; simply triggers downstream nodes.  
    - Inputs: None  
    - Outputs: Connects to the ScrapeNinja node to start scraping.  
    - Edge Cases: None, but manual trigger requires user interaction.  

#### 2.2 Web Scraping

- **Overview:**  
  This block scrapes the raw HTML content of the specified webpage URL using the ScrapeNinja node.

- **Nodes Involved:**  
  - ScrapeNinja

- **Node Details:**  
  - **ScrapeNinja**  
    - Type: Custom community node (CUSTOM.scrapeNinja)  
    - Role: Fetches the full HTML content of the target URL.  
    - Configuration:  
      - URL set to `https://news.ycombinator.com/` (hardcoded).  
      - Uses ScrapeNinja API credentials.  
    - Inputs: Triggered by the Manual Trigger node.  
    - Outputs: Provides raw HTML in the `body` property of the JSON output.  
    - Version Requirements: Requires ScrapeNinja node version >= 0.3.0.  
    - Edge Cases:  
      - Network errors or invalid URL may cause failure.  
      - API quota or authentication errors with ScrapeNinja credentials.  

#### 2.3 HTML Cleanup

- **Overview:**  
  Cleans the raw HTML to remove unnecessary elements or formatting, improving the reliability of subsequent extraction.

- **Nodes Involved:**  
  - Cleanup HTML

- **Node Details:**  
  - **Cleanup HTML**  
    - Type: Custom community node (CUSTOM.scrapeNinja)  
    - Role: Performs HTML cleanup operation on the scraped HTML.  
    - Configuration:  
      - Operation set to `cleanup-html`.  
      - Input HTML is dynamically set to the `body` property from the ScrapeNinja node output (`={{ $json.body }}`).  
    - Inputs: Receives raw HTML from ScrapeNinja node.  
    - Outputs: Cleaned HTML in the `html` property of the JSON output.  
    - Edge Cases:  
      - If input HTML is empty or malformed, cleanup may fail or produce unexpected output.  

#### 2.4 AI Processing

- **Overview:**  
  Uses Google Gemini (PaLM) Large Language Model to generate a custom JavaScript extraction function based on the cleaned HTML.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Generate JS eval code via LLM

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: Provides AI language model capabilities to generate code.  
    - Configuration:  
      - Model name: `models/gemini-exp-1206`.  
      - Credentials: Google Palm API credentials required.  
    - Inputs: Connected to the `ai_languageModel` input of the next node.  
    - Outputs: AI-generated text response containing JavaScript code.  
    - Edge Cases:  
      - API authentication or quota errors.  
      - Model response latency or timeouts.  
      - Unexpected or malformed AI output.  

  - **Generate JS eval code via LLM**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Role: Sends prompt and cleaned HTML to AI to generate a JS extraction function.  
    - Configuration:  
      - Prompt instructs the AI to write a Cheerio.js function named `extract` that returns an array of news items with fields like `url`, `title`, `score`, and `comments`.  
      - The prompt explicitly forbids use of `new URL()` to ensure compatibility with sandbox environment.  
      - Input text includes the cleaned HTML (`{{ $json.html }}`).  
      - Prompt type set to `define`.  
    - Inputs: Receives cleaned HTML from Cleanup HTML node and AI model from Google Gemini Chat Model node.  
    - Outputs: JavaScript code as text in the `text` property.  
    - Edge Cases:  
      - AI may generate invalid or incomplete JavaScript code.  
      - Expression evaluation errors if input HTML is missing or malformed.  

#### 2.5 Data Extraction

- **Overview:**  
  Executes the AI-generated JavaScript extraction function safely in a sandbox against the cleaned HTML to produce structured JSON data.

- **Nodes Involved:**  
  - Eval generated code to extract data

- **Node Details:**  
  - **Eval generated code to extract data**  
    - Type: Custom community node (CUSTOM.scrapeNinja)  
    - Role: Runs the custom JS extractor function on the cleaned HTML to extract structured data.  
    - Configuration:  
      - Operation set to `extract-custom`.  
      - HTML input dynamically set to the cleaned HTML output from `Cleanup HTML` node (`={{ $('Cleanup HTML').item.json.html }}`).  
      - Extraction function code dynamically set to the AI-generated JS code from the previous node (`={{ $json.text }}`).  
    - Inputs: Receives cleaned HTML and JS code.  
    - Outputs: Extracted structured JSON data representing the news items.  
    - Edge Cases:  
      - JavaScript runtime errors if AI-generated code is invalid.  
      - Sandbox execution limits or timeouts.  
      - Empty or malformed HTML input causing extraction failure.  

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                          | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                       |
|-----------------------------|---------------------------------------------|----------------------------------------|--------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Generate custom web scraper  | Manual Trigger                              | Workflow entry trigger                  |                          | ScrapeNinja                |                                                                                                 |
| ScrapeNinja                 | CUSTOM.scrapeNinja                          | Scrape raw HTML from target URL        | Generate custom web scraper | Cleanup HTML               | Requires ScrapeNinja community node v0.3.0 or higher.                                           |
| Cleanup HTML                | CUSTOM.scrapeNinja                          | Clean raw HTML for better parsing      | ScrapeNinja              | Generate JS eval code via LLM |                                                                                                 |
| Google Gemini Chat Model    | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI model to generate JS extraction code |                          | Generate JS eval code via LLM (ai_languageModel input) |                                                                                                 |
| Generate JS eval code via LLM | @n8n/n8n-nodes-langchain.chainLlm          | Generate JS extractor function via AI  | Cleanup HTML, Google Gemini Chat Model | Eval generated code to extract data |                                                                                                 |
| Eval generated code to extract data | CUSTOM.scrapeNinja                          | Execute AI-generated JS to extract data | Generate JS eval code via LLM |                            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Generate custom web scraper`  
   - Type: Manual Trigger  
   - No parameters needed. This node will start the workflow.

2. **Add ScrapeNinja Node**  
   - Name: `ScrapeNinja`  
   - Type: CUSTOM.scrapeNinja (community node)  
   - Parameters:  
     - URL: `https://news.ycombinator.com/` (or any target URL)  
   - Credentials: Select or create ScrapeNinja API credentials.  
   - Connect output of `Generate custom web scraper` to input of this node.

3. **Add Cleanup HTML Node**  
   - Name: `Cleanup HTML`  
   - Type: CUSTOM.scrapeNinja  
   - Parameters:  
     - Operation: `cleanup-html`  
     - HTML: Set expression to `={{ $json.body }}` to use the raw HTML from `ScrapeNinja` node output.  
   - Connect output of `ScrapeNinja` node to input of this node.

4. **Add Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Parameters:  
     - Model Name: `models/gemini-exp-1206`  
     - Options: Leave default or empty.  
   - Credentials: Configure Google Palm API credentials.  
   - This node will be connected as an AI language model input to the next node.

5. **Add Generate JS eval code via LLM Node**  
   - Name: `Generate JS eval code via LLM`  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Parameters:  
     - Prompt Type: `define`  
     - Text: Use the following prompt with embedded cleaned HTML:  
       ```
       write cheerio.js function to news items. your response MUST start with:

       function extract(html, cheerioInstance) {
       // use cheerio to load html...

       return [{ url: "item1", "title": "xxx", "score": "5", comments: 6 }, ... ]
       }
       do not use new URL() because this wont work in my env.
       html:
       {{ $json.html }}
       ```
   - Connect the output of `Cleanup HTML` node to the main input of this node.  
   - Connect the output of `Google Gemini Chat Model` node to the `ai_languageModel` input of this node.

6. **Add Eval generated code to extract data Node**  
   - Name: `Eval generated code to extract data`  
   - Type: CUSTOM.scrapeNinja  
   - Parameters:  
     - Operation: `extract-custom`  
     - HTML: Set expression to `={{ $('Cleanup HTML').item.json.html }}` to use cleaned HTML.  
     - Extraction Function: Set expression to `={{ $json.text }}` to use AI-generated JS code.  
   - Connect output of `Generate JS eval code via LLM` node to input of this node.

7. **Save and Activate Workflow**  
   - Ensure all credentials are properly configured:  
     - ScrapeNinja API credentials for scraping and cleanup nodes.  
     - Google Palm API credentials for AI model nodes.  
   - Test the workflow by manually triggering `Generate custom web scraper`.  
   - Monitor outputs for structured JSON data extracted from the target webpage.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow only works on self-hosted n8n instances because it uses the ScrapeNinja community node. | Installation: Settings → Community nodes → Search "n8n-nodes-scrapeninja" → Install (v0.3.0 or higher). |
| See this workflow in action on LinkedIn: https://www.linkedin.com/feed/update/urn:li:activity:7289659870935490560/ | Demonstration video and usage example.                                                                  |