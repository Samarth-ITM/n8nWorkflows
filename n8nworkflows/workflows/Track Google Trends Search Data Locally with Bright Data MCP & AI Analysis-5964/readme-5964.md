Track Google Trends Search Data Locally with Bright Data MCP & AI Analysis

https://n8nworkflows.xyz/workflows/track-google-trends-search-data-locally-with-bright-data-mcp---ai-analysis-5964


# Track Google Trends Search Data Locally with Bright Data MCP & AI Analysis

### 1. Workflow Overview

This workflow automates tracking of Google Trends local search data using Bright Data's MCP (Massive Crawling Platform) combined with AI-powered parsing and analysis. It is designed for marketing, SEO professionals, or researchers who want to automatically gather trending keywords and their scores from Google Trends for a specified region, then store the data locally in a Google Sheet for further use.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Input Setup:** Manual initiation with setting the target Google Trends URL (localized region/topic can be customized here).
- **1.2 AI-Powered Scraping & Parsing:** Uses an AI Agent integrated with Bright Data MCP to scrape Google Trends, then parses the scraped data into structured JSON.
- **1.3 Processing & Storage:** Breaks down the scraped trends into individual items and appends them into a Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1 — Trigger & Input Setup

**Overview:**  
This block starts the workflow manually and sets the target Google Trends URL for scraping.

**Nodes Involved:**  
- 🔌 Trigger: Manual Start  
- 📝 Set google trends URL

**Node Details:**

- **🔌 Trigger: Manual Start**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow on user command (button press)  
  - Configuration: Default — no parameters needed  
  - Inputs: None (start node)  
  - Outputs: Passes flow to Set URL node  
  - Potential Failures: None (manual trigger)  
  - Notes: Used for manual control instead of automatic scheduling.

- **📝 Set google trends URL**  
  - Type: Set  
  - Role: Defines the Google Trends URL to scrape  
  - Configuration: Assigns a hardcoded string URL `"https://trends.google.com/trending?geo=US"` to variable `url`  
  - Key Expressions: `url = "https://trends.google.com/trending?geo=US"`  
  - Inputs: From Manual Trigger  
  - Outputs: Passes URL data to the Agent node  
  - Potential Failures: None typical; user can modify URL here for different regions/topics  
  - Notes: This node can be extended to set dynamic URLs or input parameters.

---

#### 2.2 Block 2 — AI-Powered Scraping & Parsing

**Overview:**  
This block uses an AI Agent to instruct Bright Data MCP to scrape the Google Trends URL, then processes the raw scraped data into structured JSON.

**Nodes Involved:**  
- 🤖 Scrape Trends with MCP (LangChain Agent)  
- 🧠 OpenAI Model  
- 🌐 Bright Data MCP  
- Auto-fixing Output Parser  
- Structured Output Parser

**Node Details:**

- **🤖 Scrape Trends with MCP**  
  - Type: LangChain Agent (AI Agent node)  
  - Role: Orchestrates interaction between AI model and scraping tool  
  - Configuration:  
    - Prompt: Requests Bright Data MCP to scrape the given Google Trends URL and return keywords, score, and date  
    - Uses `{{ $json.url }}` for dynamic URL input  
    - Output parser enabled for structured data  
  - Inputs: Receives URL from Set URL node  
  - Outputs: Provides structured trend data to Code node  
  - Potential Failures: API limits, parsing errors, scraping failures from MCP  
  - Notes: This node integrates AI logic with the scraping execution.

- **🧠 OpenAI Model**  
  - Type: LangChain Chat OpenAI  
  - Role: Provides the AI language model (gpt-4o-mini) for agent prompt processing  
  - Configuration: Model set to "gpt-4o-mini"  
  - Inputs: Feeds AI instructions to Agent node  
  - Outputs: AI-generated instructions for scraping and parsing  
  - Credentials: Requires OpenAI API key  
  - Potential Failures: API quota, connectivity, invalid prompts

- **🌐 Bright Data MCP**  
  - Type: MCP Client Tool  
  - Role: Performs the actual web scraping of Google Trends page as instructed by Agent  
  - Configuration: Uses `scrape_as_markdown` tool with parameters dynamically set by AI  
  - Inputs: Receives scrape commands from Agent node  
  - Outputs: Raw scraped content to Output Parsers  
  - Credentials: Requires Bright Data MCP API credentials  
  - Potential Failures: Scraping blocked by Google, API limits, network issues

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser Autofixing  
  - Role: Attempts to correct and fix malformed JSON output from scraping before final parsing  
  - Inputs: Raw data from Bright Data MCP and AI Agent  
  - Outputs: Cleaner JSON to Structured Output Parser  
  - Potential Failures: Unable to fix severe JSON errors, causing downstream failure

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI/fixed output into defined JSON schema with trending keywords  
  - Configuration: JSON schema example provided, expecting array of `trending_keywords` with fields: `keyword`, `score`, `date`  
  - Inputs: Fixed JSON from Auto-fixing Output Parser  
  - Outputs: Structured JSON passed to next block  
  - Potential Failures: Schema mismatch, unexpected data format

---

#### 2.3 Block 3 — Processing & Storage

**Overview:**  
This block splits the list of keywords into individual items and appends each as a row in a Google Sheet.

**Nodes Involved:**  
- 🧩 Split Trends (One per Item)  
- 📄 Save to Google Sheets

**Node Details:**

- **🧩 Split Trends (One per Item)**  
  - Type: Code (Function)  
  - Role: Transforms the array of trending keywords into single items for iteration  
  - Configuration: JavaScript function extracts `trending_keywords` array, outputs one item per keyword with `keyword`, `score`, and `date`  
  - Key Expressions: Maps each keyword object to separate item  
  - Inputs: Structured JSON from Agent node output  
  - Outputs: Individual items for Google Sheets node  
  - Potential Failures: If input JSON is empty or malformed, function errors may occur

- **📄 Save to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends each individual keyword item as a new row in a specified Google Sheet  
  - Configuration:  
    - Operation: Append  
    - Sheet: Sheet1 (gid=0) in document with ID `1U3JIgUjCjjOUssjcwDcV5v5dFI_ssjzMHb9DzjtH4nU`  
    - Columns mapped: Date → Date, Score → Score, Keyword → Keyword  
  - Inputs: Single keyword items from Code node  
  - Outputs: None (end node)  
  - Credentials: Uses OAuth2 credential for Google Sheets  
  - Potential Failures: Authentication error, rate limits, sheet access permission errors

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                      | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                         |
|------------------------|---------------------------------|------------------------------------|---------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| 🔌 Trigger: Manual Start | Manual Trigger                  | Start workflow manually             | None                      | 📝 Set google trends URL    | Section 1 — Trigger & Input: “Set the Mission”                                                     |
| 📝 Set google trends URL | Set                            | Define Google Trends URL to scrape | 🔌 Trigger: Manual Start  | 🤖 Scrape Trends with MCP   | Section 1 — Trigger & Input: “Set the Mission”                                                     |
| 🤖 Scrape Trends with MCP | LangChain Agent                | AI orchestration of scraping        | 📝 Set google trends URL  | 🧩 Split Trends (One per Item) | Section 2 — Scrape Trends with Bright Data MCP                                                    |
| 🧠 OpenAI Model          | LangChain LM Chat OpenAI       | AI Language model for Agent          | None                      | 🤖 Scrape Trends with MCP   | Section 2 — Scrape Trends with Bright Data MCP                                                    |
| 🌐 Bright Data MCP       | MCP Client Tool                | Execute web scraping                 | 🤖 Scrape Trends with MCP  | Auto-fixing Output Parser   | Section 2 — Scrape Trends with Bright Data MCP                                                    |
| Auto-fixing Output Parser| LangChain Output Parser Autofixing | Fix JSON output from scraping     | 🌐 Bright Data MCP, OpenAI Model | 🤖 Scrape Trends with MCP | Section 2 — Scrape Trends with Bright Data MCP                                                    |
| Structured Output Parser | LangChain Structured Output Parser | Parse fixed output JSON schema     | Auto-fixing Output Parser | 🤖 Scrape Trends with MCP   | Section 2 — Scrape Trends with Bright Data MCP                                                    |
| 🧩 Split Trends (One per Item) | Code (Function)             | Split trends array into single items | 🤖 Scrape Trends with MCP  | 📄 Save to Google Sheets    | Section 3 — Process & Save: “Put it to Work”                                                      |
| 📄 Save to Google Sheets | Google Sheets                  | Append individual keyword rows      | 🧩 Split Trends (One per Item) | None                    | Section 3 — Process & Save: “Put it to Work”                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually  
   - No special configuration needed  
   - Position it as the first node.

2. **Create Set Node to Define Google Trends URL**  
   - Type: Set  
   - Name: "Set google trends URL"  
   - Add a string field named `url` with value `"https://trends.google.com/trending?geo=US"` (modifiable for other regions)  
   - Connect output of Manual Trigger to this node.

3. **Create LangChain Agent Node to Scrape Trends**  
   - Type: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)  
   - Name: "Scrape Trends with MCP"  
   - Parameters:  
     - Prompt: "Use Bright Data MCP to scrape the following Google Trends URL and return with keywords, score, and date.\n\nURL: {{ $json.url }}"  
     - Enable output parser  
   - Connect output of Set URL node to this node.

4. **Add LangChain OpenAI Chat Model Node**  
   - Type: LangChain LM Chat OpenAI (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)  
   - Name: "OpenAI Model"  
   - Model: Select "gpt-4o-mini"  
   - Connect this node as the AI language model input for the Agent node.

5. **Add Bright Data MCP Client Tool Node**  
   - Type: MCP Client Tool (`n8n-nodes-mcp.mcpClientTool`)  
   - Name: "Bright Data MCP"  
   - Tool Name: "scrape_as_markdown"  
   - Operation: "executeTool"  
   - Tool Parameters: leave dynamic, passed from AI Agent  
   - Connect this node as the AI tool input for the Agent node.

6. **Add Auto-fixing Output Parser Node**  
   - Type: LangChain Output Parser Autofixing (`@n8n/n8n-nodes-langchain.outputParserAutofixing`)  
   - Name: "Auto-fixing Output Parser"  
   - Connect input from Bright Data MCP and OpenAI Model outputs as required by the Agent flow.  
   - Connect output to Structured Output Parser.

7. **Add Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`)  
   - Name: "Structured Output Parser"  
   - Configure JSON schema example matching expected data structure:  
     ```json
     {
       "trending_keywords": [
         {
           "keyword": "example",
           "score": "string",
           "date": "string"
         }
       ]
     }
     ```  
   - Connect input from Auto-fixing Output Parser.  
   - Connect output to next block (Code node).

8. **Add Code Node to Split Trends One per Item**  
   - Type: Code (Function)  
   - Name: "Split Trends (One per Item)"  
   - JavaScript code:  
     ```javascript
     const input = items[0].json;
     const keywords = input.output.trending_keywords;
     return keywords.map(keyword => {
       return {
         json: {
           keyword: keyword.keyword,
           score: keyword.score,
           date: keyword.date
         }
       };
     });
     ```  
   - Connect input from Structured Output Parser.  
   - Connect output to Google Sheets node.

9. **Add Google Sheets Node to Save Data**  
   - Type: Google Sheets  
   - Name: "Save to Google Sheets"  
   - Operation: Append  
   - Document ID: Set your Google Sheet document ID (e.g., `1U3JIgUjCjjOUssjcwDcV5v5dFI_ssjzMHb9DzjtH4nU`)  
   - Sheet Name: Use `"gid=0"` or actual sheet name  
   - Columns to map:  
     - Keyword → Keyword  
     - Score → Score  
     - Date → Date  
   - Set Google Sheets OAuth2 credentials for authentication.  
   - Connect input from Code node.

10. **Credential Configuration**  
    - Configure OpenAI API credentials with valid API key for OpenAI nodes.  
    - Configure Bright Data MCP API credentials in MCP Client Tool node.  
    - Configure Google Sheets OAuth2 credentials for Google Sheets node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                 |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| I’ll receive a tiny commission if you join Bright Data through this link—thanks for fueling more free content! | https://get.brightdata.com/1tndi4600b25                        |
| Workflow assistance and support contact: Yaron@nofluff.online                                                 |                                                                |
| More tips and tutorials available at:                                                                         | - YouTube: https://www.youtube.com/@YaronBeen/videos           |
|                                                                                                               | - LinkedIn: https://www.linkedin.com/in/yaronbeen/             |
| Visual Summary:                                                                                               | 📌 SECTION 1: 🟢 Trigger & Input → Manual Start → Set URL       |
|                                                                                                               | 📌 SECTION 2: 🟡 Scrape & Structure → Agent → OpenAI → MCP → Parser |
|                                                                                                               | 📌 SECTION 3: 🟣 Process & Save → Split Keywords → Save to Sheets |
| Use Case Example:                                                                                             | Local bakery in Los Angeles tracks trending search phrases for SEO optimization |

---

**Disclaimer:** The text above is extracted and analyzed from an automated n8n workflow, fully compliant with content policies, handling only legal and public data.