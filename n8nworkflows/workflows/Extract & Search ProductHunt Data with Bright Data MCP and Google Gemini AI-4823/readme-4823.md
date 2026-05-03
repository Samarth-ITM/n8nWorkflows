Extract & Search ProductHunt Data with Bright Data MCP and Google Gemini AI

https://n8nworkflows.xyz/workflows/extract---search-producthunt-data-with-bright-data-mcp-and-google-gemini-ai-4823


# Extract & Search ProductHunt Data with Bright Data MCP and Google Gemini AI

### 1. Workflow Overview

This workflow automates the extraction and search of Product Hunt data by leveraging Bright Data's MCP (Massive Cloud Proxy) services alongside Google Gemini AI language models. It is designed to:

- Initiate data extraction from Product Hunt categories using Bright Data's scraping tools.
- Perform Google search queries to complement the data extraction.
- Use Google Gemini AI to process, clean, and structure the extracted data.
- Store and notify about the structured results via Google Sheets and webhooks.

The workflow is logically divided into these blocks:

- **1.1 Input Initialization:** Manual trigger and setup of base URLs, search queries, and configuration variables.
- **1.2 Bright Data MCP Tool Usage:** Listing available MCP tools and executing scraping/search operations.
- **1.3 AI Agent Processing:** Using Google Gemini AI to analyze, clean, and format the extracted data.
- **1.4 Structured Data Extraction & Parsing:** Further AI-driven processing to extract structured fields from AI outputs.
- **1.5 Data Storage & Notification:** Writing structured data to disk, updating Google Sheets, and sending webhook notifications.
- **1.6 Logging & Documentation Notes:** Sticky notes with disclaimers, usage notes, and branding.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:** This block sets up the initial input parameters and triggers the workflow manually.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
  - List all tools for Bright Data
  - Set the Input Fields
  - Set the Agent Operation

- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - *Type:* Manual Trigger
    - *Role:* Entry point, allows manual start of the workflow.
    - *Config:* No parameters.
    - *Input/Output:* Output connected to "List all tools for Bright Data".
    - *Edge Cases:* N/A (manual trigger).

  - **List all tools for Bright Data**
    - *Type:* MCP Client (Bright Data SDK integration)
    - *Role:* Fetches a list of available tools from Bright Data's MCP API.
    - *Config:* Uses MCP Client API credentials.
    - *Input:* Trigger from manual.
    - *Output:* Connected to "Set the Input Fields".
    - *Failures:* API auth errors, network timeout.

  - **Set the Input Fields**
    - *Type:* Set Node
    - *Role:* Defines key input variables such as:
      - `base_url`: Product Hunt main URL (`https://www.producthunt.com`)
      - `category`: Scraping category (default "resumes")
      - `search`: Google search query ("The best resume tools in 2025")
      - `engine`: Search engine identifier ("google")
      - `webhook_url`: URL for webhook notifications
    - *Input:* From MCP Client node.
    - *Output:* Connected to "Set the Agent Operation".
    - *Edge Cases:* Input values must be valid URLs and strings.

  - **Set the Agent Operation**
    - *Type:* Set Node
    - *Role:* Defines the AI Agent operation instruction (`Perform a Product Hunt data extract`).
    - *Input:* From "Set the Input Fields".
    - *Output:* Connected to "AI Agent".
    - *Edge Cases:* Operation string must be meaningful to the AI agent.

#### 1.2 Bright Data MCP Tool Usage

- **Overview:** Executes Bright Data MCP tools for scraping and searching Product Hunt data, feeding results into the AI Agent.
- **Nodes Involved:**
  - MCP Client for Google Search
  - MCP Client for Markdown Data Extract

- **Node Details:**

  - **MCP Client for Google Search**
    - *Type:* MCP Client Tool (Bright Data)
    - *Role:* Executes Google search queries using Bright Data’s search engine tool.
    - *Config:* Parameters:
      - `query` from "Set the Input Fields" search field.
      - `engine` from "Set the Input Fields" engine field.
    - *Input:* Invoked as AI tool by AI Agent node.
    - *Output:* Feeds search results to AI Agent.
    - *Failures:* API limits, invalid query, network errors.

  - **MCP Client for Markdown Data Extract**
    - *Type:* MCP Client Tool (Bright Data)
    - *Role:* Scrapes Product Hunt category pages, extracting data in markdown format.
    - *Config:* Parameter:
      - `url` constructed as `https://www.producthunt.com/categories/<category>` from input fields.
    - *Input:* Invoked as AI tool by AI Agent node.
    - *Output:* Feeds scraped markdown data to AI Agent.
    - *Failures:* Website structure changes, scraping failures, rate limits.

#### 1.3 AI Agent Processing

- **Overview:** Processes the extracted and searched data using Google Gemini AI, producing clean, human-readable output.
- **Nodes Involved:**
  - AI Agent
  - Google Gemini Chat Model
  - Update Google Sheets for AI Agent
  - Create a binary data for Structured Data Extract
  - Initiate a Webhook Notification for Structured Data
  - Write the structured content to disk

- **Node Details:**

  - **AI Agent**
    - *Type:* LangChain AI Agent Node
    - *Role:* Orchestrates AI processing using Google Gemini and MCP tools.
    - *Config:* Uses `agent_operation` text as prompt, outputs tool responses.
    - *Input:* From "Set the Agent Operation" and MCP Client tools.
    - *Output:* Feeds to binary data creation, webhook, Google Sheets, and Structured Data Extractor.
    - *Failure:* AI model errors, prompt failures, API quota limits.

  - **Google Gemini Chat Model**
    - *Type:* LangChain Google Gemini Chat LLM
    - *Role:* Provides language model backend for AI Agent.
    - *Config:* Model set to "models/gemini-2.0-flash-exp".
    - *Credentials:* Google Palm API.
    - *Input:* Connected as ai_languageModel for AI Agent.
    - *Failures:* Authentication, model availability.

  - **Update Google Sheets for AI Agent**
    - *Type:* Google Sheets Node
    - *Role:* Appends or updates AI Agent output in a Google Sheet ("Sheet1").
    - *Config:* Maps `output` JSON string to the sheet.
    - *Credentials:* Google Sheets OAuth2.
    - *Input:* From AI Agent.
    - *Failures:* Auth errors, quota exceeded, sheet access denied.

  - **Create a binary data for Structured Data Extract**
    - *Type:* Function Node
    - *Role:* Converts AI Agent JSON output into base64-encoded binary data for downstream use.
    - *Input:* From AI Agent.
    - *Output:* Connected to webhook notification and write file nodes.
    - *Failures:* Serialization errors, buffer issues.

  - **Initiate a Webhook Notification for Structured Data**
    - *Type:* HTTP Request Node
    - *Role:* Sends structured data as POST payload to configured webhook URL.
    - *Config:* URL from input fields; body parameter `product_info` with AI output.
    - *Input:* From binary data function node.
    - *Failures:* Network errors, webhook URL invalid.

  - **Write the structured content to disk**
    - *Type:* Read/Write File Node
    - *Role:* Saves extracted structured data as JSON file locally (`d:\ProductData.json`).
    - *Input:* From binary data function node.
    - *Failures:* File permission errors, disk space.

#### 1.4 Structured Data Extraction & Parsing

- **Overview:** Further processes AI Agent output to extract structured fields like links and descriptions using Google Gemini AI and output parser.
- **Nodes Involved:**
  - Structured Data Extractor (LangChain Chain LLM)
  - Google Gemini Chat Model Structured Data Extract
  - Structured Output Parser
  - Update Google Sheets for Structured Data

- **Node Details:**

  - **Structured Data Extractor**
    - *Type:* LangChain Chain LLM Node
    - *Role:* Extracts structured data (links, keywords, descriptions) from AI Agent output using a defined prompt.
    - *Config:* Uses prompt with input from AI Agent output and base URL for link construction.
    - *Input:* From AI Agent output.
    - *Output:* Parsed structured data sent to Google Sheets.
    - *Failure:* Parsing errors, AI model response issues.

  - **Google Gemini Chat Model Structured Data Extract**
    - *Type:* LangChain Google Gemini Chat LLM
    - *Role:* Provides language model backend for the structured data extraction chain.
    - *Config:* Same model as AI Agent.
    - *Credentials:* Google Palm API.
    - *Input:* Connected as ai_languageModel to Structured Data Extractor.
    - *Failure:* Same as Gemini Chat Model.

  - **Structured Output Parser**
    - *Type:* LangChain Output Parser Structured
    - *Role:* Parses the structured JSON output into an array schema with `link` and `desc` fields.
    - *Config:* Manual JSON schema defined.
    - *Input:* From Structured Data Extractor.
    - *Output:* Parsed data to Google Sheets.
    - *Failure:* Schema mismatch, parse failures.

  - **Update Google Sheets for Structured Data**
    - *Type:* Google Sheets Node
    - *Role:* Appends or updates structured data results in a dedicated Google Sheet tab ("Sheet2").
    - *Config:* Stores JSON stringified structured data.
    - *Credentials:* Google Sheets OAuth2.
    - *Input:* From Structured Output Parser.
    - *Failures:* Same as Google Sheets node above.

#### 1.5 Data Storage & Notification

- Covered in previous blocks:
  - Writing JSON to disk
  - Sending webhook notifications
  - Updating Google Sheets

#### 1.6 Logging & Documentation Notes

- **Overview:** Sticky notes provide contextual information, disclaimers, branding, and usage instructions.
- **Nodes Involved:** All Sticky Note nodes
- **Content Highlights:**
  - Bright Data MCP Client community node only works on n8n self-hosted.
  - Google Gemini LLM is used for AI agent handling.
  - Instructions to customize input fields and agent operation.
  - Workflow logo for Bright Data branding.

---

### 3. Summary Table

| Node Name                                  | Node Type                      | Functional Role                                           | Input Node(s)                       | Output Node(s)                                        | Sticky Note                                                                                   |
|--------------------------------------------|-------------------------------|-----------------------------------------------------------|-----------------------------------|------------------------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’           | Manual Trigger                | Workflow entry trigger                                    | -                                 | List all tools for Bright Data                        |                                                                                              |
| List all tools for Bright Data             | MCP Client                    | Fetches MCP tools list                                    | When clicking ‘Execute workflow’  | Set the Input Fields                                 |                                                                                              |
| Set the Input Fields                       | Set                           | Defines base URL, category, search query, engine, webhook | List all tools for Bright Data    | Set the Agent Operation                              | "Deals with the ProductHunt data extraction by utilizing the Bright Data MCP and Google Gemini LLM.\n\n**Please make sure to set the input fields node and the agent operation node to fulfill your needs**" |
| Set the Agent Operation                    | Set                           | Defines AI Agent operation instruction                    | Set the Input Fields              | AI Agent                                            | "## Agent Operation\n\n1. Perform a Product Hunt data extract\n2. Google search and extract data" |
| AI Agent                                  | LangChain AI Agent            | Orchestrates AI processing and tool invocation            | Set the Agent Operation, MCP tools | Create a binary data for Structured Data Extract, Initiate a Webhook Notification for Structured Data, Update Google Sheets for AI Agent, Structured Data Extractor |                                                                                              |
| Google Gemini Chat Model                   | LangChain LLM Google Gemini  | LLM backend for AI Agent                                 | - (connected as ai_languageModel to AI Agent) | AI Agent                                            | "## LLM Usages\n\nGoogle Gemini LLM is being utilized for the AI Agent handling"              |
| MCP Client for Google Search               | MCP Client Tool               | Executes search engine queries via Bright Data           | AI Agent                         | AI Agent                                            |                                                                                              |
| MCP Client for Markdown Data Extract       | MCP Client Tool               | Scrapes Product Hunt category in markdown                 | AI Agent                         | AI Agent                                            |                                                                                              |
| Create a binary data for Structured Data Extract | Function                     | Converts AI output JSON to base64 binary                  | AI Agent                         | Write the structured content to disk, Initiate a Webhook Notification for Structured Data |                                                                                              |
| Write the structured content to disk       | Read/Write File               | Saves JSON data locally                                   | Create a binary data for Structured Data Extract | -                                                    |                                                                                              |
| Initiate a Webhook Notification for Structured Data | HTTP Request                 | Sends structured data to configured webhook URL          | Create a binary data for Structured Data Extract | -                                                    |                                                                                              |
| Structured Data Extractor                   | LangChain Chain LLM           | Extracts links, keywords, descriptions from AI output    | AI Agent                        | Update Google Sheets for Structured Data             |                                                                                              |
| Google Gemini Chat Model Structured Data Extract | LangChain LLM Google Gemini  | LLM backend for Structured Data Extractor                 | - (connected as ai_languageModel to Structured Data Extractor) | Structured Data Extractor                           |                                                                                              |
| Structured Output Parser                    | LangChain Output Parser       | Parses structured JSON output                             | Structured Data Extractor        | Update Google Sheets for Structured Data             |                                                                                              |
| Update Google Sheets for Structured Data   | Google Sheets                 | Stores structured data into Google Sheets                 | Structured Output Parser         | -                                                    |                                                                                              |
| Update Google Sheets for AI Agent           | Google Sheets                 | Stores AI Agent output into Google Sheets                 | AI Agent                        | -                                                    |                                                                                              |
| Sticky Note                                | Sticky Note                   | Notes on Agent Operation                                  | -                               | -                                                    | "## Agent Operation\n\n1. Perform a Product Hunt data extract\n2. Google search and extract data" |
| Sticky Note2                               | Sticky Note                   | Disclaimer about MCP Client usage                          | -                               | -                                                    | "## Disclaimer\nThis template is only available on n8n self-hosted as it's making use of the community node for MCP Client." |
| Sticky Note3                               | Sticky Note                   | General note on usage and customization                   | -                               | -                                                    | "## Note\n\nDeals with the ProductHunt data extraction by utilizing the Bright Data MCP and Google Gemini LLM.\n\n**Please make sure to set the input fields node and the agent operation node to fulfill your needs**" |
| Sticky Note4                               | Sticky Note                   | LLM usage information                                     | -                               | -                                                    | "## LLM Usages\n\nGoogle Gemini LLM is being utilized for the AI Agent handling"              |
| Sticky Note5                               | Sticky Note                   | Bright Data branding logo                                  | -                               | -                                                    | "## Logo\n\n\n![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `When clicking ‘Execute workflow’`
   - Type: Manual Trigger
   - No parameters.

2. **Add MCP Client Node (List Tools)**
   - Name: `List all tools for Bright Data`
   - Type: MCP Client
   - Credentials: MCP Client API (Bright Data account)
   - No extra parameters.
   - Connect output from manual trigger node.

3. **Add Set Node for Input Fields**
   - Name: `Set the Input Fields`
   - Type: Set
   - Assign string variables:
     - `base_url` = `https://www.producthunt.com`
     - `category` = `resumes`
     - `search` = `The best resume tools in 2025`
     - `engine` = `google`
     - `webhook_url` = `<your webhook URL>`
   - Connect from MCP Client List Tools node.

4. **Add Set Node for Agent Operation**
   - Name: `Set the Agent Operation`
   - Type: Set
   - Assign string variable:
     - `agent_operation` = `Perform a Product Hunt data extract`
   - Connect from `Set the Input Fields`.

5. **Add LangChain AI Agent Node**
   - Name: `AI Agent`
   - Type: LangChain AI Agent
   - Parameters:
     - Text: `={{ $json.agent_operation }}\n\nOutput the data in a clean and human readable format. Output only the tool response.`
     - Prompt Type: Define
     - Enable retry on fail.
   - Connect input from `Set the Agent Operation`.
   - Configure AI Language Model input connection from:
     - `Google Gemini Chat Model` node (see step 6).
   - Configure AI Tool inputs from:
     - `MCP Client for Google Search`
     - `MCP Client for Markdown Data Extract`

6. **Add Google Gemini Chat Model Node (For AI Agent)**
   - Name: `Google Gemini Chat Model`
   - Type: LangChain LLM Google Gemini Chat
   - Model Name: `models/gemini-2.0-flash-exp`
   - Credentials: Google Gemini (PaLM) API
   - Connect as ai_languageModel input to AI Agent.

7. **Add MCP Client Tool Node for Google Search**
   - Name: `MCP Client for Google Search`
   - Type: MCP Client Tool
   - Tool Name: `search_engine`
   - Operation: `executeTool`
   - Tool Parameters:
     ```
     {
       "query": "{{ $('Set the Input Fields').item.json.search }}",
       "engine": "{{ $('Set the Input Fields').item.json.engine }}"
     }
     ```
   - Description (manual): `Perform a search as per the specified search engine : {{ $('Set the Input Fields').item.json.engine }}`
   - Credentials: MCP Client API
   - Connect as ai_tool input to AI Agent.

8. **Add MCP Client Tool Node for Markdown Data Extract**
   - Name: `MCP Client for Markdown Data Extract`
   - Type: MCP Client Tool
   - Tool Name: `scrape_as_markdown`
   - Operation: `executeTool`
   - Tool Parameters:
     ```
     {
       "url": "{{ $('Set the Input Fields').item.json.base_url }}/categories/{{ encodeURI($('Set the Input Fields').item.json.category) }}"
     }
     ```
   - Description (manual): `Perform Product Hunt data scrapping in markdown format`
   - Credentials: MCP Client API
   - Connect as ai_tool input to AI Agent.

9. **Add Function Node to Create Binary Data**
   - Name: `Create a binary data for Structured Data Extract`
   - Type: Function
   - Code:
     ```javascript
     items[0].binary = {
       data: {
         data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
       }
     };
     return items;
     ```
   - Connect main input from AI Agent output.

10. **Add HTTP Request Node for Webhook Notification**
    - Name: `Initiate a Webhook Notification for Structured Data`
    - Type: HTTP Request
    - HTTP Method: POST
    - URL: `={{ $('Set the Input Fields').item.json.webhook_url }}`
    - Send Body: true
    - Body Parameters:
      - Name: `product_info`
      - Value: `={{ $json.output }}`
    - Connect from Function node (binary data).

11. **Add Read/Write File Node**
    - Name: `Write the structured content to disk`
    - Type: Read/Write File
    - Operation: Write
    - File Name: `d:\ProductData.json`
    - Connect from Function node.

12. **Add LangChain Chain LLM Node for Structured Data Extraction**
    - Name: `Structured Data Extractor`
    - Type: LangChain Chain LLM
    - Prompt Text:
      ```
      Extract the links, keywords, description from  {{ $('AI Agent').item.json.output }}

      Construct the links with the base url as {{ $('Set the Input Fields').item.json.base_url }}
      ```
    - Output Parser: Enabled (hasOutputParser = true)
    - Retry on fail: Enabled
    - Connect main input from AI Agent output.
    - Connect ai_languageModel input from `Google Gemini Chat Model Structured Data Extract` (next step).

13. **Add Google Gemini Chat Model Node for Structured Data Extract**
    - Name: `Google Gemini Chat Model Structured Data Extract`
    - Type: LangChain LLM Google Gemini Chat
    - Model Name: `models/gemini-2.0-flash-exp`
    - Credentials: Google Gemini (PaLM) API
    - Connect as ai_languageModel input to Structured Data Extractor.

14. **Add LangChain Output Parser Structured Node**
    - Name: `Structured Output Parser`
    - Type: LangChain Output Parser Structured
    - Schema Type: Manual
    - Input Schema:
      ```json
      {
        "type": "array",
        "properties": {
          "link": { "type": "string" },
          "desc": { "type": "string" }
        }
      }
      ```
    - Connect ai_outputParser input from Structured Data Extractor.

15. **Add Google Sheets Node for Structured Data**
    - Name: `Update Google Sheets for Structured Data`
    - Type: Google Sheets
    - Operation: Append or Update
    - Document ID: `1cmJkB_DuSUbHoZ-LthySa7utEZFIvzeLinGcHjMyvzI` (replace with your own)
    - Sheet Name: `Sheet2` (ID: 1785677350)
    - Columns Mapping:
      - `structured_data` = `={{ $json.output.toJsonString() }}`
    - Credentials: Google Sheets OAuth2
    - Connect main input from Structured Output Parser.

16. **Add Google Sheets Node for AI Agent Output**
    - Name: `Update Google Sheets for AI Agent`
    - Type: Google Sheets
    - Operation: Append or Update
    - Document ID: Same as above
    - Sheet Name: `Sheet1` (gid=0)
    - Columns Mapping:
      - `output` = `={{ $json.output.toJsonString() }}`
    - Credentials: Google Sheets OAuth2
    - Connect main input from AI Agent output.

17. **Add Sticky Notes**
    - Add sticky notes with content:
      - Disclaimer about MCP Client usage (self-hosted only).
      - LLM usage info (Google Gemini).
      - General notes on customization.
      - Agent operation summary.
      - Bright Data branding logo.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This template is only available on n8n self-hosted as it's making use of the community node for MCP Client. | Disclaimer on MCP Client node usage.                                                                         |
| Google Gemini LLM is being utilized for the AI Agent handling.                                           | Technology stack detail.                                                                                      |
| Deals with the ProductHunt data extraction by utilizing the Bright Data MCP and Google Gemini LLM.       | General workflow purpose and customization note.                                                             |
| Please make sure to set the input fields node and the agent operation node to fulfill your needs.        | Important customization instruction.                                                                         |
| Bright Data Logo ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) | Bright Data branding.                                                                                          |

---

**Disclaimer:** The provided text originates exclusively from an n8n automated workflow and strictly complies with current content policies without any illegal or offensive material. All processed data is legal and public.