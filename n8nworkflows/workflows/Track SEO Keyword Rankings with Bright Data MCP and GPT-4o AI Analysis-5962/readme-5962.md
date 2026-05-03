Track SEO Keyword Rankings with Bright Data MCP and GPT-4o AI Analysis

https://n8nworkflows.xyz/workflows/track-seo-keyword-rankings-with-bright-data-mcp-and-gpt-4o-ai-analysis-5962


# Track SEO Keyword Rankings with Bright Data MCP and GPT-4o AI Analysis

### 1. Workflow Overview

This workflow automates the monitoring of SEO keyword rankings on Google using a combination of AI and proxy-based scraping technologies. It is designed to run on a scheduled basis (daily or weekly) to track the top 5 search results for specified keywords, log the rankings, and provide structured historical data for analysis or reporting.

**Target Use Cases:**  
- SEO professionals or digital marketers tracking keyword performance over time  
- Agencies managing multiple client SEO campaigns  
- Automated reporting and alerting on ranking changes  
- Historical data collection for SEO audits and competitor analysis

**Logical Blocks:**

- **1.1 Input & Trigger Configuration**  
  Sets the workflow schedule and defines the keywords to monitor.

- **1.2 Smart SERP Scraping with AI Agent**  
  Uses Bright Data MCP proxy via an AI agent to fetch and parse Google SERP results accurately and anonymously.

- **1.3 Rank Logging and Reporting**  
  Processes the scraped data into individual records and logs them into Google Sheets for persistent storage and analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Trigger Configuration

**Overview:**  
This block sets the workflow execution schedule and defines the keyword(s) for which Google rankings will be monitored. It enables automated, recurring keyword tracking without manual intervention.

**Nodes Involved:**  
- 🕒 Trigger: Run Daily/Weekly  
- 📝 Input: Keyword & Domain  
- Sticky Note (Section 1 description)

**Node Details:**  

- **🕒 Trigger: Run Daily/Weekly**  
  - Type: Schedule Trigger  
  - Role: Starts workflow at a fixed time daily (9 AM)  
  - Configuration: Interval set to trigger daily at hour 9  
  - Connections: Output → Input of "Input: Keyword & Domain" node  
  - Edge cases: Workflow won't run if n8n instance is down at trigger time; time zone considerations apply  
  - No special version requirements

- **📝 Input: Keyword & Domain**  
  - Type: Set node  
  - Role: Defines the keyword(s) and optionally domain(s) to track  
  - Configuration: Contains a string field named `keyword` with value `"best running shoes"` (modifiable)  
  - Connections: Output → Input of "🤖 SERP Scraper Agent (MCP)" node  
  - Edge cases: Empty or invalid keywords will cause downstream nodes to fail or return empty results  

- **Sticky Note (Section 1: Input & Trigger Configuration)**  
  - Provides detailed explanation about the purpose and usage of these nodes for users  
  - No technical function but helpful for documentation and maintenance

---

#### 1.2 Smart SERP Scraping with AI Agent

**Overview:**  
This block performs the core task of scraping Google Search Results Pages (SERP) for the specified keyword using an AI-driven approach combined with Bright Data MCP proxy technology. It ensures accurate, undetectable scraping of the top 5 results, which are parsed into structured data.

**Nodes Involved:**  
- 🤖 SERP Scraper Agent (MCP) — AI Agent node  
- OpenAI Chat Model — AI language model for prompt crafting  
- MCP Client — Proxy client for real Google SERP fetching  
- Auto-fixing Output Parser — AI output parser that corrects structured data inconsistencies  
- Structured Output Parser1 — Defines expected JSON schema for SERP results  
- Sticky Note (Section 2 description)

**Node Details:**  

- **🤖 SERP Scraper Agent (MCP)**  
  - Type: LangChain AI Agent node  
  - Role: Orchestrates AI prompt creation, MCP search execution, and result parsing  
  - Configuration: Uses a prompt that instructs to provide ranking of first 5 websites based on input keyword  
  - Inputs: Receives keyword from "Input: Keyword & Domain"  
  - Outputs: Produces a structured list of top 5 SERP results (each with rank, title, url, description)  
  - Internal connections (via LangChain config):  
    - AI Language Model: "OpenAI Chat Model" (for understanding and prompt crafting)  
    - AI Tool: "MCP Client" (executes the real web scraping)  
    - AI Output Parser: "Auto-fixing Output Parser" (cleans and fixes output format)  
  - Edge cases:  
    - Failures in proxy connection (MCP Client) such as network issues, authentication errors, or rate limiting  
    - AI model errors or timeouts  
    - Unexpected SERP page structure changes causing misparsing  
  - Version: 2

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Generates natural language prompts for the scraping agent  
  - Configuration: Uses GPT-4o-mini model variant  
  - Inputs/Outputs: Connected as AI language model to "SERP Scraper Agent (MCP)"  
  - Edge cases: API key limits, connectivity issues, or model unavailability  
  - Version: 1.2

- **MCP Client**  
  - Type: n8n MCP Client Tool  
  - Role: Executes actual Google search requests through Bright Data MCP proxy service  
  - Configuration: Executes "search_engine" tool with parameters dynamically generated by the AI agent  
  - Credentials: Requires MCP Client API credentials with appropriate permissions  
  - Edge cases: Proxy downtime, quota limits, invalid credentials, CAPTCHAs not bypassed  
  - Version: 1

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser (Auto-fixing)  
  - Role: Ensures the AI-generated output conforms to the expected structured format; attempts to fix errors automatically  
  - Configuration: Default options (automatic error correction)  
  - Inputs: Connected to "OpenAI Chat Model1" output parser chain  
  - Outputs: Parsed and corrected structured data for downstream consumption  
  - Edge cases: Severe format deviations might still cause failure  
  - Version: 1

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Defines the expected JSON schema for the SERP results (array of objects with rank, title, url, description)  
  - Configuration: Contains an example JSON schema to guide parsing  
  - Inputs: Feeds into "Auto-fixing Output Parser"  
  - Edge cases: Schema mismatches or changes in SERP data structure  
  - Version: 1.2

- **Sticky Note (Section 2: Smart SERP Scraping with AI Agent)**  
  - Explains the combination of AI and MCP proxy for reliable, undetectable scraping  
  - Highlights the three sub-nodes roles inside the agent  

---

#### 1.3 Rank Logging and Reporting

**Overview:**  
This block processes the structured SERP data by splitting the list into individual entries and logging each entry as a separate row into a Google Sheets spreadsheet for historical tracking and analysis.

**Nodes Involved:**  
- 🧠 Format SERP Results (Code node)  
- 📊 Log to Google Sheets  
- Sticky Note (Section 3 description)

**Node Details:**  

- **🧠 Format SERP Results**  
  - Type: Code node (JavaScript)  
  - Role: Takes the list of SERP results and emits each item as an individual workflow item for separate logging  
  - Configuration: Custom JS code that maps the array of results into separate JSON items  
  - Inputs: Receives structured SERP results from "🤖 SERP Scraper Agent (MCP)"  
  - Outputs: Multiple items each containing one search result object with rank, title, url, description  
  - Edge cases: Empty or malformed input array leading to zero output or errors  
  - Version: 2

- **📊 Log to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends or updates rows in a Google Sheets document with the parsed SERP data  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Spreadsheet ID: `1p64unH_JjzG978cAxPZC4kSZmoXgvYTA-Q7qdfnxr8Y`  
    - Sheet: `gid=0` (Sheet1)  
    - Columns mapped: URL, Rank, Title, Description from JSON fields  
  - Credentials: Google Sheets OAuth2 account required  
  - Edge cases: Remote spreadsheet access errors, API limits, data type mismatches  
  - Version: 4.6

- **Sticky Note (Section 3: Rank Logging and Reporting)**  
  - Describes the purpose of splitting and logging results  
  - Emphasizes the benefit of structured historical tracking for SEO audits and alerts  

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                            | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                         |
|--------------------------|----------------------------------|--------------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| 🕒 Trigger: Run Daily/Weekly | Schedule Trigger                 | Triggers workflow on daily schedule (9 AM) | —                           | 📝 Input: Keyword & Domain      | See Section 1 sticky note for explanation                                                                           |
| 📝 Input: Keyword & Domain  | Set                              | Defines keywords to monitor                  | 🕒 Trigger: Run Daily/Weekly | 🤖 SERP Scraper Agent (MCP)    | See Section 1 sticky note for explanation                                                                           |
| 🤖 SERP Scraper Agent (MCP) | LangChain AI Agent               | Orchestrates AI prompt, MCP proxy search, and parsing | 📝 Input: Keyword & Domain  | 🧠 Format SERP Results          | See Section 2 sticky note for explanation                                                                           |
| OpenAI Chat Model          | LangChain OpenAI Chat Model      | Generates natural language prompts           | — (AI language model input to agent) | 🤖 SERP Scraper Agent (MCP)    | See Section 2 sticky note for explanation                                                                           |
| MCP Client                 | MCP Client Tool                  | Executes Google search via Bright Data MCP  | 🤖 SERP Scraper Agent (MCP) (as AI tool) | 🤖 SERP Scraper Agent (MCP)    | See Section 2 sticky note for explanation                                                                           |
| Auto-fixing Output Parser  | LangChain Output Parser (Auto-fixing) | Cleans and fixes AI output format             | OpenAI Chat Model1, Structured Output Parser1 | 🤖 SERP Scraper Agent (MCP)    | See Section 2 sticky note for explanation                                                                           |
| Structured Output Parser1  | LangChain Structured Output Parser | Defines expected SERP JSON schema              | OpenAI Chat Model1           | Auto-fixing Output Parser       | See Section 2 sticky note for explanation                                                                           |
| 🧠 Format SERP Results      | Code Node                       | Splits SERP list into individual records     | 🤖 SERP Scraper Agent (MCP) | 📊 Log to Google Sheets          | See Section 3 sticky note for explanation                                                                           |
| 📊 Log to Google Sheets     | Google Sheets Node              | Logs individual SERP results into spreadsheet | 🧠 Format SERP Results       | —                             | See Section 3 sticky note for explanation                                                                           |
| Sticky Note                | Sticky Note                     | Documentation and explanation only            | —                           | —                             | Multiple sticky notes cover sections 1, 2, 3, and workflow assistance contact info (distributed as noted above)      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger node**  
   - Type: Schedule Trigger  
   - Name: `Trigger: Run Daily/Weekly`  
   - Configure to trigger daily at 9:00 AM (adjust as needed)  
   - No credentials needed

2. **Create the Set node for keyword input**  
   - Type: Set  
   - Name: `Input: Keyword & Domain`  
   - Add string field `keyword` with a default value (e.g., `"best running shoes"`)  
   - Connect output of Trigger node to this node's input

3. **Create the LangChain OpenAI Chat Model node**  
   - Type: LangChain LM Chat OpenAI  
   - Name: `OpenAI Chat Model`  
   - Choose model: `gpt-4o-mini`  
   - Provide OpenAI API credentials  
   - No direct input from workflow, but will be connected internally later

4. **Create the MCP Client node**  
   - Type: MCP Client Tool  
   - Name: `MCP Client`  
   - Set toolName to `search_engine`, operation to `executeTool`  
   - Tool parameters will be dynamically set by AI agent  
   - Provide MCP Client API credentials

5. **Create the Structured Output Parser1 node**  
   - Type: LangChain Output Parser Structured  
   - Name: `Structured Output Parser1`  
   - Paste the example JSON schema defining expected SERP result fields (rank, title, url, description)

6. **Create the OpenAI Chat Model1 node** (used for output parsing chain)  
   - Type: LangChain LM Chat OpenAI  
   - Name: `OpenAI Chat Model1`  
   - Use model `gpt-4o-mini` with same OpenAI credentials as step 3

7. **Create the Auto-fixing Output Parser node**  
   - Type: LangChain Output Parser Autofixing  
   - Name: `Auto-fixing Output Parser`  
   - Default options for automatic corrections

8. **Create the SERP Scraper Agent (MCP) node**  
   - Type: LangChain AI Agent  
   - Name: `🤖 SERP Scraper Agent (MCP)`  
   - Configure prompt: `"Based on the following keyword, provide me ranking of the 1st 5 website.\nkeyword: {{ $json.keyword }}"`  
   - Set AI language model node to `OpenAI Chat Model` (step 3)  
   - Assign AI tool node to `MCP Client` (step 4)  
   - Set AI output parser to `Auto-fixing Output Parser` (step 7)  
   - Connect input from `Input: Keyword & Domain` node (step 2)  

9. **Link the output parsers internally:**  
   - Connect `Structured Output Parser1` output to `Auto-fixing Output Parser` input  
   - Connect `OpenAI Chat Model1` output to `Structured Output Parser1` input  
   - Connect `OpenAI Chat Model1` input from `Auto-fixing Output Parser` as per LangChain sequence (used in agent)

10. **Create the Code node to format SERP results**  
    - Type: Code (JavaScript)  
    - Name: `🧠 Format SERP Results`  
    - Paste the code:  
      ```javascript
      const serpList = items[0].json.output;
      return serpList.map(result => ({ json: result }));
      ```  
    - Connect input from `🤖 SERP Scraper Agent (MCP)`

11. **Create the Google Sheets node for logging**  
    - Type: Google Sheets  
    - Name: `📊 Log to Google Sheets`  
    - Operation: appendOrUpdate  
    - Provide Google Sheets OAuth2 credentials  
    - Set Document ID: `1p64unH_JjzG978cAxPZC4kSZmoXgvYTA-Q7qdfnxr8Y` (or your own)  
    - Set Sheet Name: `gid=0` (Sheet1)  
    - Map columns:  
      - URL → `{{$json.url}}`  
      - Rank → `{{$json.rank}}`  
      - Title → `{{$json.title}}`  
      - Description → `{{$json.description}}`  
    - Connect input from `🧠 Format SERP Results`

12. **Wire connections:**  
    - `Trigger: Run Daily/Weekly` → `Input: Keyword & Domain`  
    - `Input: Keyword & Domain` → `🤖 SERP Scraper Agent (MCP)`  
    - `🤖 SERP Scraper Agent (MCP)` → `🧠 Format SERP Results`  
    - `🧠 Format SERP Results` → `📊 Log to Google Sheets`  

13. **Test the workflow:**  
    - Run manually or wait for scheduled trigger  
    - Verify Google Sheets receives populated rows for keyword rankings  

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| I’ll receive a tiny commission if you join Bright Data through this link—thanks for fueling more free content!                        | https://get.brightdata.com/1tndi4600b25                                                             |
| Workflow support contact and tutorial links                                                                                          | Contact: Yaron@nofluff.online <br> YouTube: https://www.youtube.com/@YaronBeen/videos <br> LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| This workflow combines AI and proxy scraping for robust SEO rank tracking that adapts to SERP changes and bypasses scraping blocks.  | See detailed sticky notes inside the workflow on sections 1–3                                        |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies fully with content policies and contains no illegal or offensive content. All data handled is legal and publicly accessible.