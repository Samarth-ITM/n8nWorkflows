Real-Time Sales Pipeline Analytics with Bright Data, OpenAI, and Google Sheets

https://n8nworkflows.xyz/workflows/real-time-sales-pipeline-analytics-with-bright-data--openai--and-google-sheets-5974


# Real-Time Sales Pipeline Analytics with Bright Data, OpenAI, and Google Sheets

### 1. Workflow Overview

This workflow automates real-time monitoring and analysis of sales pipeline metrics by scraping CRM-like data from a configurable source, processing it with AI tools, and storing the structured insights into Google Sheets for reporting and visualization. It is designed for sales operations, marketing analysts, and business stakeholders who seek automated, code-free pipeline analytics with scalable data scraping and AI-powered analysis.

The workflow is structured into three main logical blocks:

- **1.1 Input Reception & Setup**  
  Manual workflow trigger and configuration of the CRM data source URL.

- **1.2 AI-Powered Data Scraping and Analysis**  
  An AI agent generates scraping instructions, invokes Bright Data’s MCP tool to scrape data, parses and cleans the output into structured JSON metrics.

- **1.3 Data Formatting & Storage**  
  Splits the structured metrics into individual records and appends them as rows into a Google Sheet for continuous tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Input Reception & Setup

**Overview:**  
This block initializes the workflow manually and sets the target CRM data source URL to be scraped.

**Nodes Involved:**  
- ⚡ Trigger: Start CRM Scraper  
- 🔗 Set Source URL (CRM/JSONPlaceholder)

**Node Details:**

- **⚡ Trigger: Start CRM Scraper**  
  - *Type:* Manual Trigger  
  - *Role:* Initiates the workflow execution manually by the user.  
  - *Configuration:* No parameters; user clicks execute to start.  
  - *Connections:* Outputs to "🔗 Set Source URL".  
  - *Edge Cases:* User must manually trigger; no automation on schedule.  

- **🔗 Set Source URL (CRM/JSONPlaceholder)**  
  - *Type:* Set Node  
  - *Role:* Defines a workflow variable `URL` containing the source API endpoint for CRM data.  
  - *Configuration:* Sets `"URL"` to `"https://jsonplaceholder.typicode.com/users"` (mock API simulating CRM data).  
  - *Key Expressions:* None dynamic here; static URL assignment.  
  - *Connections:* Input from Trigger; outputs to CRM AI Agent node.  
  - *Edge Cases:* URL must be valid and accessible; if changed to real CRM, ensure correct API and authentication if needed.

---

#### 2.2 Block 2: AI-Powered Data Scraping and Analysis

**Overview:**  
This core block uses OpenAI to generate scraping instructions, leverages Bright Data MCP tool for simulated scraping, and parses the scraped data into structured JSON, representing sales pipeline metrics.

**Nodes Involved:**  
- 🤖 Monitor Sales Pipeline (CRM AI Agent)  
- 🧠 AI Brain  
- 🌐 Bright Data MCP  
- Auto-fixing Output Parser  
- OpenAI Chat Model  
- 🧾 Clean JSON Parser

**Node Details:**

- **🤖 Monitor Sales Pipeline (CRM AI Agent)**  
  - *Type:* LangChain Agent Node  
  - *Role:* Coordinates AI-driven scraping and analysis workflow.  
  - *Configuration:*  
    - Prompt instructs AI to scrape CRM-like sales data from the URL.  
    - Defines a JSON schema simulating leads with `leadId`, `rep`, `stage`, `value`, `status`.  
    - Calculates summary metrics: total leads, pipeline value, stage counts, status counts, top reps.  
  - *Connections:* Receives URL input from Set Source URL; outputs metrics JSON to Split Metrics node.  
  - *Edge Cases:* AI prompt failures, unexpected API data format, malformed output. Requires robust output parsing.

- **🧠 AI Brain**  
  - *Type:* LangChain OpenAI Chat  
  - *Role:* Generates scraping instructions for Bright Data MCP tool.  
  - *Configuration:* Uses OpenAI GPT-4.1-mini model with OpenAI credential.  
  - *Connections:* Feeds instructions to CRM AI Agent node as AI language model.  
  - *Edge Cases:* API authentication failures, rate limits, model unavailability.

- **🌐 Bright Data MCP**  
  - *Type:* MCP Client Tool  
  - *Role:* Executes the scraping on target URL using Bright Data’s mobile proxy network with the `scrape_as_markdown` tool.  
  - *Configuration:* Tool parameters are dynamically injected by AI output.  
  - *Credentials:* Connected to MCP Client account.  
  - *Connections:* Used by CRM AI Agent node as AI tool.  
  - *Edge Cases:* Proxy failures, CAPTCHA blocks, network timeouts, invalid tool parameters.

- **Auto-fixing Output Parser**  
  - *Type:* LangChain Output Parser Autofixing  
  - *Role:* Attempts to correct and parse AI output for consistency and JSON structure before passing on.  
  - *Connections:* Receives output from OpenAI Chat Model and Clean JSON Parser, feeds corrected output back into CRM AI Agent.  
  - *Edge Cases:* Parsing failures, malformed JSON, incomplete data.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat  
  - *Role:* Provides conversational AI model interface for output parsing and instruction generation.  
  - *Configuration:* GPT-4.1-mini model with OpenAI credentials.  
  - *Connections:* Outputs to Auto-fixing Output Parser.  
  - *Edge Cases:* API rate limits, authentication errors.

- **🧾 Clean JSON Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Converts AI-generated raw text output into structured JSON matching the example schema for sales pipeline metrics (totals, breakdowns, top reps).  
  - *Configuration:* Uses a JSON schema example to validate and guide parsing.  
  - *Connections:* Feeds cleaned JSON to Auto-fixing Output Parser.  
  - *Edge Cases:* Schema mismatches, incomplete parsing, unexpected fields.

---

#### 2.3 Block 3: Data Formatting & Storage

**Overview:**  
This block formats the structured JSON metrics into individual records suitable for Google Sheets insertion, then appends them as rows into a configured spreadsheet.

**Nodes Involved:**  
- 🧩 Split Metrics for Sheet Rows  
- 📊 Store CRM Insights (Google Sheet)

**Node Details:**

- **🧩 Split Metrics for Sheet Rows**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Transforms the aggregated JSON metrics object into an array of individual metric entries, each with `Metric Type`, `Label`, and `Value` fields.  
  - *Configuration:* Custom JS code extracts total leads, total pipeline value, status breakdown, stage breakdown, and top reps, pushing each as a separate JSON item.  
  - *Connections:* Input from CRM AI Agent output; output to Google Sheets node.  
  - *Edge Cases:* Input data missing expected fields, JS runtime errors.

- **📊 Store CRM Insights (Google Sheet)**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends each metric record as a new row into a Google Sheet for persistent storage and reporting.  
  - *Configuration:*  
    - Appends rows to sheet with ID `1D8VmSbOUEf1jQWQ-_BaHVV_HOdb_d5DIyXzRSMlU5xs`, sheet name "gid=0" (Sheet1).  
    - Maps columns: Metric Type, Label, Value from incoming JSON fields.  
  - *Credentials:* Google Sheets OAuth2 account configured.  
  - *Connections:* Receives input from Split Metrics node.  
  - *Edge Cases:* Authentication failures, sheet access permissions, API rate limits.

---

### 3. Summary Table

| Node Name                            | Node Type                          | Functional Role                       | Input Node(s)                        | Output Node(s)                        | Sticky Note                                                                                              |
|------------------------------------|----------------------------------|------------------------------------|------------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| ⚡ Trigger: Start CRM Scraper       | Manual Trigger                   | Manually starts the workflow        | -                                  | 🔗 Set Source URL                   | 🔷 SECTION 1: Start & Set Target: Manual start, URL setup for CRM data source.                          |
| 🔗 Set Source URL (CRM/JSONPlaceholder) | Set Node                        | Defines source URL for scraping     | ⚡ Trigger                         | 🤖 Monitor Sales Pipeline (CRM AI Agent) | 🔷 SECTION 1: Start & Set Target: URL configuration node.                                              |
| 🤖 Monitor Sales Pipeline (CRM AI Agent) | LangChain Agent                 | Coordinates scraping & analysis     | 🔗 Set Source URL, AI Brain, MCP, Output Parsers | 🧩 Split Metrics for Sheet Rows      | 🤖 SECTION 2: AI Agent — Scrape + Analyze: Core AI analysis and scraping coordination.                 |
| 🧠 AI Brain                        | LangChain OpenAI Chat             | Generates scraping instructions     | -                                  | 🤖 Monitor Sales Pipeline (AI language model) | 🤖 SECTION 2: AI Agent — Scrape + Analyze: OpenAI logic for generating scrape instructions.            |
| 🌐 Bright Data MCP                 | MCP Client Tool                  | Executes scraping using mobile proxy | -                                  | 🤖 Monitor Sales Pipeline (AI tool) | 🤖 SECTION 2: AI Agent — Scrape + Analyze: Bright Data MCP scraping tool integration.                   |
| Auto-fixing Output Parser          | LangChain Output Parser Autofixing | Parses and corrects AI output       | OpenAI Chat Model, Clean JSON Parser | 🤖 Monitor Sales Pipeline (AI output parser) | 🤖 SECTION 2: AI Agent — Scrape + Analyze: Ensures output is valid JSON.                              |
| OpenAI Chat Model                  | LangChain OpenAI Chat             | Chat model for output processing    | -                                  | Auto-fixing Output Parser          | 🤖 SECTION 2: AI Agent — Scrape + Analyze: OpenAI model for parsing output.                           |
| 🧾 Clean JSON Parser               | LangChain Structured Output Parser | Parses structured JSON metrics      | -                                  | Auto-fixing Output Parser          | 🤖 SECTION 2: AI Agent — Scrape + Analyze: Converts scraped text to structured JSON.                  |
| 🧩 Split Metrics for Sheet Rows    | Code Node                        | Splits aggregated metrics into rows | 🤖 Monitor Sales Pipeline (CRM AI Agent) | 📊 Store CRM Insights (Google Sheet) | 🟨 SECTION 3: Format & Store to Google Sheets: Prepares data for Sheets insertion.                    |
| 📊 Store CRM Insights (Google Sheet) | Google Sheets Node              | Appends metrics rows to Google Sheet | 🧩 Split Metrics for Sheet Rows    | -                                  | 🟨 SECTION 3: Format & Store to Google Sheets: Saves data automatically to Sheet.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add a Set Node to Define Source URL**  
   - Name: "Set Source URL (CRM/JSONPlaceholder)"  
   - Parameters: Set a string field `URL` to `"https://jsonplaceholder.typicode.com/users"` (mock CRM API).  
   - Connect Manual Trigger output to this node.

3. **Add a LangChain Agent Node**  
   - Name: "Monitor Sales Pipeline (CRM AI Agent)"  
   - Purpose: Orchestrates scraping and processing.  
   - Parameters:  
     - Text prompt instructing to scrape CRM sales data from `{{ $json.URL }}`.  
     - Simulate fields: `leadId`, `rep`, `stage`, `value`, `status`.  
     - Compute summary metrics: total leads, pipeline value, stage counts, status counts, top reps.  
   - Connect Set Source URL node output to this node.

4. **Add LangChain OpenAI Chat Node (AI Brain)**  
   - Name: "AI Brain"  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API credentials configured.  
   - Connect output as AI language model input to "Monitor Sales Pipeline" node.

5. **Add Bright Data MCP Client Tool Node**  
   - Name: "Bright Data MCP"  
   - Operation: `scrape_as_markdown` tool execution  
   - Credentials: Bright Data MCP Client account configured.  
   - Configure to receive tool parameters dynamically from AI output.  
   - Connect as an AI tool input to "Monitor Sales Pipeline" node.

6. **Add LangChain Output Parsers:**  
   - "Auto-fixing Output Parser" node to fix and parse AI outputs.  
   - "OpenAI Chat Model" node with `gpt-4.1-mini` for output processing.  
   - "Clean JSON Parser" node configured with example JSON schema representing sales pipeline metrics.  
   - Connect these nodes in the following chain:  
     OpenAI Chat Model → Auto-fixing Output Parser → Monitor Sales Pipeline (AI output parser)  
     Clean JSON Parser → Auto-fixing Output Parser

7. **Add a Code Node to Split Metrics**  
   - Name: "Split Metrics for Sheet Rows"  
   - Paste the provided JavaScript code that converts the aggregated JSON metrics into individual rows with fields `Metric Type`, `Label`, and `Value`.  
   - Connect output of "Monitor Sales Pipeline" node to this node.

8. **Add a Google Sheets Node**  
   - Name: "Store CRM Insights (Google Sheet)"  
   - Operation: Append rows.  
   - Document ID: `1D8VmSbOUEf1jQWQ-_BaHVV_HOdb_d5DIyXzRSMlU5xs` (replace with your own Sheet ID).  
   - Sheet Name: `gid=0` (first sheet).  
   - Map columns:  
     - "Metric Type" ← `{{ $json['Metric Type'] }}`  
     - "Label" ← `{{ $json.Label }}`  
     - "Value" ← `{{ $json.Value }}`  
   - Credentials: Configure Google Sheets OAuth2 account.  
   - Connect output of "Split Metrics for Sheet Rows" node to this node.

9. **Verify all connections:**  
   Manual Trigger → Set Source URL → Monitor Sales Pipeline → Split Metrics → Google Sheets  
   AI Brain, Bright Data MCP, OpenAI Chat Model, Auto-fixing Output Parser, Clean JSON Parser nodes connected as AI languageModel, AI tool, and output parsers within the Monitor Sales Pipeline node.

10. **Test the workflow manually:**  
    - Click the manual trigger node.  
    - Check logs for successful scraping, parsing, and Google Sheets appending.  
    - Adjust URLs and credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow enables non-coders to scrape and analyze sales pipeline data from CRM dashboards using mobile proxies and AI, storing results in Google Sheets automatically.                                                                   | Sticky Note #1 and #3: Workflow description and use cases.                                              |
| Replace mock URL with real CRM dashboard URL accessible via Bright Data’s mobile proxy network for real data scraping.                                                                                                                  | Section 1 overview.                                                                                      |
| Bright Data MCP tool handles bot-protection and login barriers by scraping as a mobile device browser invisibly.                                                                                                                         | Section 2 overview.                                                                                      |
| Generated insights include total leads, pipeline value, lead stage and status breakdowns, and top sales representatives by deal value.                                                                                                | Output format example in Clean JSON Parser node.                                                       |
| Use this workflow to feed data into BI tools like Google Data Studio, Looker, or Excel for live sales pipeline monitoring and reporting.                                                                                                | Section 3 overview.                                                                                      |
| For assistance or questions, contact Yaron at Yaron@nofluff.online; explore more content at YouTube (https://www.youtube.com/@YaronBeen/videos) and LinkedIn (https://www.linkedin.com/in/yaronbeen/).                                   | Sticky Note #9: Workflow assistance contact info.                                                      |
| Affiliate link for Bright Data to support content creation: https://get.brightdata.com/1tndi4600b25                                                                                                                                      | Sticky Note #5: Affiliate note.                                                                         |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects all content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.