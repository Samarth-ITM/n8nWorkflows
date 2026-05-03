Trustpilot Review Sentiment Analysis with OpenAI & Bright Data to Google Sheets

https://n8nworkflows.xyz/workflows/trustpilot-review-sentiment-analysis-with-openai---bright-data-to-google-sheets-5950


# Trustpilot Review Sentiment Analysis with OpenAI & Bright Data to Google Sheets

### 1. Workflow Overview

This workflow automates the process of scraping, analyzing, and storing customer reviews from a Trustpilot company page. It targets users who want to extract actionable sentiment insights from recent customer feedback without manual effort or coding.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Manual trigger and setting the Trustpilot company URL to specify which business page to analyze.
- **1.2 AI Processing:** Using an AI agent combined with Bright Data’s Mobile Proxy Client (MCP) tool to scrape the latest 5 reviews, perform sentiment analysis, extract key information, and summarize themes.
- **1.3 Data Output:** Splitting the AI agent's JSON array output into individual review objects and saving each review as a separate row into a Google Sheets document for further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This initial block receives user input to start the workflow manually and sets the target Trustpilot company URL that the AI agent will scrape.

**Nodes Involved:**  
- Manual Start  
- Set Trustpilot company URL  

**Node Details:**

- **Manual Start**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution of the workflow.  
  - Configuration: No parameters; triggers workflow upon user click.  
  - Connections: Output to "Set Trustpilot company URL".  
  - Edge cases: None typical; if not manually triggered, the workflow does not start.

- **Set Trustpilot company URL**  
  - Type: Set  
  - Role: Stores the Trustpilot company page URL as a string variable named `URL`.  
  - Configuration: URL hardcoded to "https://www.trustpilot.com/review/hubspot.com" (modifiable for other companies).  
  - Expressions: None; static value set.  
  - Input: From Manual Start  
  - Output: To AI Agent node.  
  - Edge cases: If URL is invalid or changed to a non-Trustpilot page, scraping may fail downstream.

---

#### 1.2 AI Processing

**Overview:**  
This core block leverages an AI agent integrated with Bright Data MCP to scrape the latest 5 reviews from the specified Trustpilot URL and analyze each for reviewer data, sentiment, and summarization.

**Nodes Involved:**  
- Agent : Fetch and analyze reviews  
- Chat Model  
- Bright Data MCP Scrapper  
- Structured Output Parser  
- Auto-fixing Output Parser  
- OpenAI Chat Model  

**Node Details:**

- **Agent : Fetch and analyze reviews**  
  - Type: LangChain AI Agent  
  - Role: Orchestrates scraping and AI reasoning.  
  - Configuration:  
    - Prompt instructs use of Bright Data's `scrape_as_markdown` tool to scrape 5 latest reviews from the Trustpilot URL.  
    - Request to return JSON array with fields: reviewer, date, rating, sentiment (positive/neutral/negative), mainIssue, reviewText.  
    - Output parser enabled to ensure valid JSON.  
  - Input: Trustpilot URL from Set Node  
  - Output: JSON array of 5 review objects to "Splits Reviews into separate Objects" node.  
  - Edge cases:  
    - Invalid URL input leads to failed scraping.  
    - Bright Data MCP service unavailability or proxy restrictions can cause timeouts or errors.  
    - AI model response could be malformed JSON; mitigated by output parsers.

- **Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides AI reasoning capabilities for the Agent node.  
  - Configuration: Uses GPT-4.1-mini model.  
  - Credentials: OpenAI API key required.  
  - Input/Output: Connected as AI language model for Agent node.  
  - Edge cases: API rate limits, authentication failures, or response timeouts.

- **Bright Data MCP Scrapper**  
  - Type: MCP Client Tool  
  - Role: Executes the `scrape_as_markdown` tool for web scraping via Bright Data's proxy network.  
  - Configuration: Tool name "scrape_as_markdown", parameters dynamically set by the AI prompt.  
  - Credentials: Bright Data MCP API credentials needed.  
  - Input/Output: Used as AI tool by Agent node.  
  - Edge cases: Proxy failures, network timeouts, or Bright Data service downtime.

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Defines explicit JSON schema for AI output to ensure valid, structured data.  
  - Configuration: JSON schema specifying required review fields and types.  
  - Input: Connected from OpenAI Chat Model.  
  - Output: Piped into Auto-fixing Output Parser.  
  - Edge cases: Schema mismatch or incomplete AI response.

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser with Auto-fix  
  - Role: Attempts to correct malformed AI JSON outputs automatically.  
  - Input: From Structured Output Parser.  
  - Output: To Agent node for final output formatting.  
  - Edge cases: If auto-fix fails, output may be invalid; downstream nodes may fail.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Used in the output parsing chain to assist auto-fixing and structured parsing.  
  - Configuration: Same GPT-4.1-mini model and credentials as Chat Model node.  
  - Input/Output: Part of output parsing chain.  
  - Edge cases: Same as other OpenAI nodes.

---

#### 1.3 Data Output

**Overview:**  
This block processes the AI agent's array output by splitting the array into individual review objects and appends each record to a Google Sheet for storage and analysis.

**Nodes Involved:**  
- Splits Reviews into separate Objects  
- Save Reviews to Google Sheets  

**Node Details:**

- **Splits Reviews into separate Objects**  
  - Type: Code (JavaScript)  
  - Role: Converts the JSON array of 5 review objects into separate n8n items for downstream processing.  
  - Configuration:  
    - Accesses the AI Agent output JSON array.  
    - Maps each review object into its own item with `.json` wrapper.  
  - Input: From "Agent : Fetch and analyze reviews" node.  
  - Output: Individual JSON items each containing one review object.  
  - Edge cases: If AI output is empty or malformed, function may fail or return zero items.

- **Save Reviews to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends each review record as a new row in a specified Google Sheets document.  
  - Configuration:  
    - Operation: Append  
    - Mapping: Maps review fields (date, rating, reviewer, mainIssue, reviewText, sentiment) to corresponding sheet columns.  
    - Sheet details: Uses Spreadsheet ID and Sheet GID for "Sheet1".  
  - Credentials: Google Sheets OAuth2 credentials required.  
  - Input: From "Splits Reviews into separate Objects" node.  
  - Output: None (end node)  
  - Edge cases: Authentication errors, API rate limits, or sheet access permissions can cause failures.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                                | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                                     |
|--------------------------------|--------------------------------------|-----------------------------------------------|---------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Manual Start                   | Manual Trigger                       | Starts workflow manually                       | -                               | Set Trustpilot company URL         | 🔹 SECTION 1: 🚦 Start & Input Company URL - Includes Manual Start, Set Company URL - User triggers workflow manually           |
| Set Trustpilot company URL     | Set                                 | Stores Trustpilot company URL                  | Manual Start                    | Agent : Fetch and analyze reviews | 🔹 SECTION 1: 🚦 Start & Input Company URL - Includes Manual Start, Set Company URL - User triggers workflow manually           |
| Agent : Fetch and analyze reviews | LangChain AI Agent                   | Scrapes and analyzes Trustpilot reviews       | Set Trustpilot company URL       | Splits Reviews into separate Objects | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews - AI agent orchestrates scraping and analysis        |
| Chat Model                    | LangChain OpenAI Chat Model           | Provides AI reasoning for Agent                | Agent : Fetch and analyze reviews (ai_languageModel) | Agent : Fetch and analyze reviews (ai_languageModel) | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews - AI reasoning for scraping and analysis             |
| Bright Data MCP Scrapper       | MCP Client Tool                      | Executes scraping tool via Bright Data proxy  | Agent : Fetch and analyze reviews (ai_tool) | Agent : Fetch and analyze reviews (ai_tool) | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews - Proxy scraping with Bright Data                    |
| Structured Output Parser       | LangChain Output Parser Structured   | Validates AI output JSON structure             | OpenAI Chat Model              | Auto-fixing Output Parser          | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews - Ensures structured AI output                       |
| Auto-fixing Output Parser      | LangChain Output Parser Autofixing   | Auto-corrects AI output JSON errors            | Structured Output Parser        | Agent : Fetch and analyze reviews  | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews - Fixes malformed AI JSON output                      |
| OpenAI Chat Model             | LangChain OpenAI Chat Model           | Supports output parsing chain                   | Structured Output Parser        | Auto-fixing Output Parser          | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews - Used in output parsing chain                       |
| Splits Reviews into separate Objects | Code (JavaScript)                    | Splits AI JSON array into individual review objects | Agent : Fetch and analyze reviews | Save Reviews to Google Sheets       | 📊 SECTION 3: 📦 Split Reviews & 📥 Save to Google Sheets - Separates review array into individual rows for storage             |
| Save Reviews to Google Sheets | Google Sheets                        | Appends each review as a row in Google Sheets  | Splits Reviews into separate Objects | -                                | 📊 SECTION 3: 📦 Split Reviews & 📥 Save to Google Sheets - Stores structured review data for reporting and dashboards          |
| Sticky Note                   | Sticky Note                         | Section 1 notes                                | -                               | -                                | 🔹 SECTION 1: 🚦 Start & Input Company URL                                                                                      |
| Sticky Note1                  | Sticky Note                         | Section 2 notes                                | -                               | -                                | 🤖 SECTION 2: 🧠 AI Agent – Scrape, Analyze & Summarize Trustpilot Reviews                                                      |
| Sticky Note2                  | Sticky Note                         | Section 3 notes                                | -                               | -                                | 📊 SECTION 3: 📦 Split Reviews & 📥 Save to Google Sheets                                                                       |
| Sticky Note3                  | Sticky Note                         | Full workflow overview and benefits            | -                               | -                                | Workflow overview and detailed explanation of all sections                                                                    |
| Sticky Note5                  | Sticky Note                         | Workflow assistance contact info               | -                               | -                                | Contact: Yaron@nofluff.online; YouTube and LinkedIn links                                                                      |
| Sticky Note9                  | Sticky Note                         | Affiliate disclosure for Bright Data           | -                               | -                                | Bright Data affiliate link: https://get.brightdata.com/1tndi4600b25                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Manual Start" Node**  
   - Type: Manual Trigger  
   - No parameters.  
   - Position: Starting point.

2. **Create "Set Trustpilot company URL" Node**  
   - Type: Set  
   - Assign a string variable named `URL` with value `"https://www.trustpilot.com/review/hubspot.com"` (or replace with target company URL).  
   - Connect Manual Start → Set.

3. **Create "Agent : Fetch and analyze reviews" Node**  
   - Type: LangChain AI Agent  
   - Configuration:  
     - Prompt: Instruct agent to use Bright Data MCP's `scrape_as_markdown` tool to scrape latest 5 reviews from `{{ $json.URL }}`.  
     - Request JSON array with fields: reviewer, date, rating, sentiment (positive/neutral/negative), mainIssue, reviewText.  
     - Enable output parser.  
   - Connect Set → Agent node.

4. **Create "Chat Model" Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1-mini`  
   - Credentials: Set OpenAI API credentials.  
   - Connect to Agent node as AI language model input.

5. **Create "Bright Data MCP Scrapper" Node**  
   - Type: MCP Client Tool  
   - Tool Name: `scrape_as_markdown`  
   - Operation: Execute tool with parameters passed dynamically by AI agent prompt.  
   - Credentials: Set Bright Data MCP API credentials.  
   - Connect to Agent node as AI tool input.

6. **Create "Structured Output Parser" Node**  
   - Type: LangChain Output Parser Structured  
   - Define JSON schema for expected review array with required fields and types as per prompt.  
   - Connect from OpenAI Chat Model node.

7. **Create "Auto-fixing Output Parser" Node**  
   - Type: LangChain Output Parser Autofixing  
   - No additional options required.  
   - Connect from Structured Output Parser node and output to Agent node.

8. **Create "OpenAI Chat Model" Node** (for output parsing chain)  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1-mini`  
   - Credentials: Same OpenAI account.  
   - Connect from Structured Output Parser to Auto-fixing Output Parser.

9. **Create "Splits Reviews into separate Objects" Node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const outputArray = $('Agent : Fetch and analyze reviews').first().json.output;
     return outputArray.map(review => ({ json: review }));
     ```  
   - Connect Agent node → this node.

10. **Create "Save Reviews to Google Sheets" Node**  
    - Type: Google Sheets  
    - Operation: Append  
    - Spreadsheet ID: `"1dlaEmLIzSgTxH0whWJnx51lPHyb34bGxMSQig5KI7CQ"`  
    - Sheet Name (GID): `"gid=0"`  
    - Map columns:  
      - `date` → `{{ $json.date }}`  
      - `rating` → `{{ $json.rating }}`  
      - `reviewer` → `{{ $json.reviewer }}`  
      - `mainIssue` → `{{ $json.mainIssue }}`  
      - `reviewText` → `{{ $json.reviewText }}`  
      - `sentiment` → `{{ $json.sentiment }}`  
    - Credentials: Set Google Sheets OAuth2 credentials.  
    - Connect Splits Reviews node → Google Sheets node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates scraping, sentiment analysis, and saving of Trustpilot reviews using n8n, Bright Data MCP proxies, OpenAI GPT-4, and Google Sheets.              | Workflow Overview, Section 3 Sticky Note                                                         |
| Bright Data MCP proxies enable reliable scraping by simulating real mobile IPs, bypassing common anti-scraping measures.                                           | Section 2 Sticky Note, Bright Data MCP Scrapper node                                           |
| OpenAI GPT-4 model (gpt-4.1-mini) used for AI reasoning, sentiment classification, and output parsing with auto-fixing to handle malformed responses.               | Section 2 Sticky Notes, Chat Model nodes                                                        |
| Google Sheets used as a lightweight database for storing structured review data, facilitating reporting and dashboarding.                                         | Section 3 Sticky Note, Save Reviews to Google Sheets node                                        |
| Contact for workflow assistance: Yaron@nofluff.online                                                                                                             | Sticky Note5                                                                                     |
| YouTube channel with more content: https://www.youtube.com/@YaronBeen/videos                                                                                       | Sticky Note5                                                                                     |
| LinkedIn profile for further professional resources: https://www.linkedin.com/in/yaronbeen/                                                                        | Sticky Note5                                                                                     |
| Affiliate link for Bright Data (support the creator): https://get.brightdata.com/1tndi4600b25                                                                       | Sticky Note9                                                                                    |

---

**Disclaimer:**  
This documentation is based solely on the provided n8n workflow JSON. It adheres strictly to content policies and does not contain any illegal or protected data. All data processed by this workflow is publicly available and legally accessible.