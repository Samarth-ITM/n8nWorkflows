Track & Analyze Sales Performance with AI Insights and Google Sheets

https://n8nworkflows.xyz/workflows/track---analyze-sales-performance-with-ai-insights-and-google-sheets-5975


# Track & Analyze Sales Performance with AI Insights and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Track & Analyze Sales Performance with AI Insights and Google Sheets,"** is designed to scrape, analyze, and store sales representatives’ performance data automatically. It integrates AI-driven web scraping, data analysis, and Google Sheets storage to provide real-time insights and performance tracking without manual spreadsheet work.

**Target Use Cases:**  
- Sales managers monitoring team task completion rates  
- Automated performance ranking and coaching recommendations  
- Data collection from web dashboards or APIs protected by anti-bot measures  
- Centralized performance reporting and KPI tracking in Google Sheets

**Logical Blocks:**

- **1.1 Input Reception & Trigger:**  
  Manual start of the workflow and setting the source URL for scraping.

- **1.2 AI-Driven Scraping & Data Analysis:**  
  Uses OpenAI language models combined with Bright Data’s MCP tool to scrape the URL, parse the data, and structure it into JSON with sales rep performance metrics.

- **1.3 Data Processing & Storage:**  
  Splits the structured JSON array into individual records and appends these records into a Google Sheets document for reporting and further analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Trigger

**Overview:**  
This block initiates the workflow manually and defines the URL source to scrape, making it easy to customize and control when and where to pull data from.

**Nodes Involved:**  
- ⚡ Start Scraping (Manual Trigger)  
- 🔗 Set MCP Source URL

**Node Details:**

- **⚡ Start Scraping (Manual Trigger)**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on-demand  
  - Configuration: No parameters; simply triggers the flow manually  
  - Inputs: None  
  - Outputs: Connects to "Set MCP Source URL" node  
  - Edge cases: User must manually trigger; no automatic scheduling unless added externally

- **🔗 Set MCP Source URL**  
  - Type: Set  
  - Role: Defines the URL to scrape (source of sales data)  
  - Configuration: Contains a string parameter `activityURL` set to `https://jsonplaceholder.typicode.com/todos` as a placeholder  
  - Inputs: From manual trigger  
  - Outputs: Provides `activityURL` to the AI analysis node  
  - Edge cases: URL must be valid and reachable; placeholder must be replaced in production

---

#### 1.2 AI-Driven Scraping & Data Analysis

**Overview:**  
This core block uses AI to scrape and analyze the sales performance data from the given URL using Bright Data’s MCP tool and OpenAI’s GPT models, outputting structured JSON with key metrics and coaching recommendations.

**Nodes Involved:**  
- 🤖 Analyze Sales Rep Performance (AI Agent)  
- 🧠 AI Brain (OpenAI)  
- 🌐 Bright Data MCP Tool  
- Auto-fixing Output Parser  
- 🧾 Structured Response Parser1  
- OpenAI Chat Model

**Node Details:**

- **🤖 Analyze Sales Rep Performance (AI Agent)**  
  - Type: LangChain Agent  
  - Role: Manages the AI workflow combining scraping and analysis  
  - Configuration:  
    - Prompt instructs to scrape the URL (`{{ $json.activityURL }}`), group tasks by `userId` (rep ID), calculate total, completed, incomplete tasks, completion rate, rank reps, and recommend coaching if completion rate < 70%.  
    - Output is structured JSON.  
  - Inputs: Receives `activityURL` from Set node, AI Brain, MCP tool, and Output Parser nodes  
  - Outputs: Provides processed rep performance JSON to the splitter node  
  - Edge cases: Prompt misinterpretation, API rate limits, failed scraping, or malformed data could cause errors

- **🧠 AI Brain (OpenAI)**  
  - Type: LangChain OpenAI Chat Node  
  - Role: Processes the natural language instructions for scraping and analysis  
  - Configuration: Uses GPT-4.1-mini model, authenticated via OpenAI API credentials  
  - Inputs: Receives instructions from AI Agent  
  - Outputs: To AI Agent  
  - Edge cases: API key issues, rate limiting, model downtime

- **🌐 Bright Data MCP Tool**  
  - Type: MCP Client Tool  
  - Role: Executes the web scraping using Bright Data’s Mobile Proxy network  
  - Configuration: Uses `scrape_as_markdown` operation with parameters dynamically injected from AI Agent  
  - Inputs: From AI Agent (tool parameters)  
  - Outputs: Scraped markdown content to AI Agent  
  - Edge cases: Network issues, proxy limits, authentication errors on MCP, anti-bot bypass failures

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser Autofixing  
  - Role: Repairs and cleans up AI output to ensure valid JSON  
  - Configuration: Default autofix options  
  - Inputs: From OpenAI Chat Model  
  - Outputs: To AI Agent  
  - Edge cases: If output is too malformed, fixes may fail

- **🧾 Structured Response Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI output into structured JSON according to schema example  
  - Configuration: JSON schema example provided to map rep performance fields  
  - Inputs: From Auto-fixing Output Parser  
  - Outputs: To Auto-fixing Output Parser  
  - Edge cases: Schema mismatch, unexpected output format

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Node  
  - Role: Provides conversational AI capabilities for output parsing  
  - Configuration: Same GPT-4.1-mini model with API credentials  
  - Inputs: From Auto-fixing Output Parser  
  - Outputs: To Auto-fixing Output Parser  
  - Edge cases: Same as AI Brain node

---

#### 1.3 Data Processing & Storage

**Overview:**  
This block converts the structured JSON array into individual records and appends each record as a row in a Google Sheets document for easy reporting and visualization.

**Nodes Involved:**  
- 🧩 Split JSON to Individual Records (Function Node)  
- 📊 Store Rep Performance (Google Sheets)

**Node Details:**

- **🧩 Split JSON to Individual Records**  
  - Type: Code (JavaScript)  
  - Role: Splits the array of rep performance objects into separate workflow items  
  - Configuration:  
    - JavaScript maps over `items[0].json.output` (array of reps) and returns individual JSON objects with keys: repId, totalTasks, completedTasks, incompleteTasks, completionRate, coachingRecommended  
  - Inputs: From AI Agent output JSON array  
  - Outputs: Multiple items, one per rep, to Google Sheets node  
  - Edge cases: If input JSON is not an array or malformed, will fail or return empty

- **📊 Store Rep Performance (Google Sheets)**  
  - Type: Google Sheets (Append operation)  
  - Role: Saves the processed sales rep data to a specific Google Sheet document  
  - Configuration:  
    - Document ID: `1ybPVtArYMojjWfT5EJyiC8I666ZhfQAWAeQ0A8Qidjg`  
    - Sheet Name (gid): `gid=0` (first sheet)  
    - Columns mapped explicitly: repId, totalTasks, completedTasks, incompleteTasks (note: key has typo "incompletedTasks" in mapping), completionRate, coachingRecommended (typo "coachingRecomended" in mapping too)  
    - Append mode (adds new rows)  
    - Authenticated with Google Sheets OAuth2 credentials  
  - Inputs: From splitter node  
  - Outputs: None (end of chain)  
  - Edge cases: Credential expiry, sheet permission issues, column name typos causing data loss or misalignments

---

### 3. Summary Table

| Node Name                         | Node Type                            | Functional Role                            | Input Node(s)                   | Output Node(s)                        | Sticky Note                                                                                                                             |
|----------------------------------|------------------------------------|--------------------------------------------|--------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| ⚡ Start Scraping (Manual Trigger) | Manual Trigger                    | Manual workflow start                       | None                           | 🔗 Set MCP Source URL                | 🔹 SECTION 1: Input & Trigger — manually starts workflow and sets data source URL                                                        |
| 🔗 Set MCP Source URL             | Set                                | Defines scraping source URL                 | ⚡ Start Scraping               | 🤖 Analyze Sales Rep Performance    | 🔹 SECTION 1: Input & Trigger — defines URL to scrape, placeholder example provided                                                     |
| 🤖 Analyze Sales Rep Performance  | LangChain Agent                    | Controls AI scraping, data analysis        | 🔗 Set MCP Source URL, AI Brain, MCP Tool, Parsers | 🧩 Split JSON to Individual Records | 🤖 SECTION 2: AI Scraper & Analyzer — orchestrates AI and MCP scraping, returns structured JSON                                          |
| 🧠 AI Brain (OpenAI)              | LangChain OpenAI Chat              | Processes scraping and analysis instructions | 🤖 Analyze Sales Rep Performance | 🤖 Analyze Sales Rep Performance    | 🤖 SECTION 2: AI Scraper & Analyzer — GPT-4.1-mini model interprets scraping instructions                                                |
| 🌐 Bright Data MCP Tool           | MCP Client Tool                   | Scrapes web data via Bright Data MCP tool  | 🤖 Analyze Sales Rep Performance | 🤖 Analyze Sales Rep Performance    | 🤖 SECTION 2: AI Scraper & Analyzer — scrapes using mobile proxies and markdown                                                        |
| Auto-fixing Output Parser         | LangChain Output Parser Autofixing | Cleans AI output for valid JSON             | OpenAI Chat Model               | 🤖 Analyze Sales Rep Performance    | 🤖 SECTION 2: AI Scraper & Analyzer — ensures structured JSON output                                                                     |
| 🧾 Structured Response Parser1     | LangChain Structured Output Parser | Parses AI output to structured JSON         | Auto-fixing Output Parser       | Auto-fixing Output Parser           | 🤖 SECTION 2: AI Scraper & Analyzer — formats AI output according to JSON schema                                                        |
| OpenAI Chat Model                 | LangChain OpenAI Chat              | Provides conversational AI for parsing      | Auto-fixing Output Parser       | Auto-fixing Output Parser           | 🤖 SECTION 2: AI Scraper & Analyzer — GPT-4.1-mini model for output parsing                                                             |
| 🧩 Split JSON to Individual Records | Code (JavaScript)                 | Splits JSON array into individual records   | 🤖 Analyze Sales Rep Performance | 📊 Store Rep Performance            | 📊 SECTION 3: Process & Store in Google Sheets — splits reps into rows for storage                                                     |
| 📊 Store Rep Performance (Google Sheets) | Google Sheets                   | Appends rep performance data to Google Sheets | 🧩 Split JSON to Individual Records | None                              | 📊 SECTION 3: Process & Store in Google Sheets — stores structured data for reporting                                                   |
| Sticky Note                      | Sticky Note                       | Documentation and explanation notes         | None                           | None                                | Covers multiple nodes; detailed explanations for each section, including workflow overview                                              |
| Sticky Note1                     | Sticky Note                       | Documentation notes for AI scraping section | None                           | None                                | Covers AI Scraper & Analyzer block                                                                                                      |
| Sticky Note2                     | Sticky Note                       | Documentation notes for Google Sheets section | None                           | None                                | Covers Process & Store in Google Sheets block                                                                                           |
| Sticky Note3                     | Sticky Note                       | Full workflow overview and breakdown         | None                           | None                                | Comprehensive workflow explanation covering all sections                                                                                |
| Sticky Note5                     | Sticky Note                       | Affiliate note regarding Bright Data         | None                           | None                                | Bright Data affiliate link and appreciation                                                                                             |
| Sticky Note9                     | Sticky Note                       | Workflow assistance and support contact info | None                           | None                                | Contact info and social media links for workflow support                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "⚡ Start Scraping (Manual Trigger)"  
   - No parameters needed  
   - Position: Left side (start of workflow)

2. **Create a Set Node**  
   - Type: Set  
   - Name: "🔗 Set MCP Source URL"  
   - Add field: `activityURL` (String)  
   - Set value to your target scraping URL (example placeholder: `https://jsonplaceholder.typicode.com/todos`)  
   - Connect output of Manual Trigger node to this Set node

3. **Create LangChain OpenAI Chat Node for AI Brain**  
   - Type: LangChain OpenAI Chat  
   - Name: "🧠 AI Brain (OpenAI)"  
   - Model: GPT-4.1-mini  
   - Credentials: Link your OpenAI API credentials  
   - Position: near AI Agent node for clarity

4. **Create MCP Client Tool Node**  
   - Type: MCP Client Tool  
   - Name: "🌐 Bright Data MCP Tool"  
   - Operation: `scrape_as_markdown`  
   - Credentials: Link Bright Data MCP API credentials  
   - Position near AI Agent node

5. **Create LangChain Output Parser Autofixing Node**  
   - Type: Output Parser Autofixing  
   - Name: "Auto-fixing Output Parser"  
   - Default config is sufficient

6. **Create LangChain Structured Output Parser Node**  
   - Type: Structured Output Parser  
   - Name: "🧾 Structured Response Parser1"  
   - Provide JSON schema example for sales rep data, including fields: repId, totalTasks, completedTasks, incompleteTasks, completionRate, coachingRecommended  
   - Position downstream of output parser autofixing node

7. **Create LangChain OpenAI Chat Node for Output Parsing**  
   - Type: LangChain OpenAI Chat  
   - Name: "OpenAI Chat Model"  
   - Model: GPT-4.1-mini  
   - Credentials: same OpenAI API credentials

8. **Create LangChain Agent Node for AI Scraper & Analyzer**  
   - Type: LangChain Agent  
   - Name: "🤖 Analyze Sales Rep Performance"  
   - Prompt:  
     ```
     You are a web scraper and data analyzer connected to Bright Data’s MCP tool.

     Scrape the following URL:
     {{ $json.activityURL }}

     Each item in the data represents a sales rep activity. The "userId" is the rep ID, "title" is the task name, and "completed" is task status.

     Step 1: Group tasks by each userId (rep).
     Step 2: For each rep, calculate:

     totalTasks

     completedTasks

     incompleteTasks

     completionRate (as percentage)
     Step 3: Rank reps from highest to lowest completion rate.
     Step 4: Add a field coachingRecommended: true if completionRate is less than 70%, false otherwise.
     ```
   - Connect inputs:  
     - From Set MCP Source URL node  
     - AI Brain node as `ai_languageModel`  
     - MCP Tool node as `ai_tool`  
     - Auto-fixing Output Parser node as `ai_outputParser`  
   - Output to next processing nodes

9. **Create Code Node for Splitting JSON**  
   - Type: Code (JavaScript)  
   - Name: "🧩 Split JSON to Individual Records"  
   - Code:  
     ```javascript
     return items[0].json.output.map(rep => {
       return {
         json: {
           repId: rep.repId,
           totalTasks: rep.totalTasks,
           completedTasks: rep.completedTasks,
           incompleteTasks: rep.incompleteTasks,
           completionRate: rep.completionRate,
           coachingRecommended: rep.coachingRecommended
         }
       };
     });
     ```
   - Connect output of AI Agent node to this node

10. **Create Google Sheets Node for Storing Data**  
    - Type: Google Sheets  
    - Name: "📊 Store Rep Performance (Google Sheets)"  
    - Operation: Append  
    - Document ID: Your Google Sheets document ID (example: `1ybPVtArYMojjWfT5EJyiC8I666ZhfQAWAeQ0A8Qidjg`)  
    - Sheet Name: `gid=0` or your target sheet name  
    - Map columns explicitly as follows:  
      - repId → repId  
      - totalTasks → totalTasks  
      - completedTasks → completedTasks  
      - incompleteTasks → incompleteTasks (note: fix typo from original)  
      - completionRate → completionRate  
      - coachingRecommended → coachingRecommended (fix typo)  
    - Credentials: Google Sheets OAuth2  
    - Connect output of Split JSON node to this node

11. **Connect Workflow**  
    - Manual Trigger → Set MCP Source URL → AI Agent  
    - AI Agent → Split JSON → Google Sheets  
    - AI Agent internally uses AI Brain, MCP Tool, Output Parsers, and OpenAI Chat Model as per LangChain agent configuration

12. **Test and Validate**  
    - Manually trigger the workflow  
    - Check logs and Google Sheet for data correctness  
    - Adjust URL and prompt as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Bright Data affiliate link: "I’ll receive a tiny commission if you join Bright Data through this link—thanks for fueling more free content!" | https://get.brightdata.com/1tndi4600b25                          |
| Workflow assistance and support contact: Yaron@nofluff.online, YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ | Contact and resource links for workflow help and community tips |
| Workflow uses placeholder URL `https://jsonplaceholder.typicode.com/todos` for demo; replace with real MCP URL for production use.     | Important for customization and real-world deployment            |
| Column mapping in Google Sheets node contains typos ("incompletedTasks" and "coachingRecomended") — correct these to avoid data errors. | Prevent data misalignment in storage                              |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a no-code integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.