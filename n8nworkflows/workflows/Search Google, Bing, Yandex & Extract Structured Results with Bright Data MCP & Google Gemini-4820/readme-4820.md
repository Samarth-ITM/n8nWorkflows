Search Google, Bing, Yandex & Extract Structured Results with Bright Data MCP & Google Gemini

https://n8nworkflows.xyz/workflows/search-google--bing--yandex---extract-structured-results-with-bright-data-mcp---google-gemini-4820


# Search Google, Bing, Yandex & Extract Structured Results with Bright Data MCP & Google Gemini

### 1. Workflow Overview

This workflow automates performing web searches on multiple search engines—Google, Bing, and Yandex—using Bright Data’s MCP Client tools, then processes and extracts structured, human-readable results leveraging Google Gemini (PaLM) LLM models. It is designed for scenarios where a user needs to programmatically query these search engines, collect and clean the returned data, and optionally notify a webhook endpoint with the cleaned results.

The logic is divided into the following blocks:

- **1.1 Input Reception and Initialization**: Triggering the workflow manually and preparing the initial input data such as search query, action type, and webhook URL.
- **1.2 Search Execution via Bright Data MCP Tools**: Using Bright Data MCP Client tools to perform searches on Google, Bing, and Yandex according to the input query.
- **1.3 AI Agent Processing with Google Gemini**: An AI agent interprets the search action and orchestrates the use of the search tools.
- **1.4 Data Extraction and Cleanup**: Using a Google Gemini-powered LangChain information extractor to clean and structure the raw search results.
- **1.5 Output Handling**: Writing the cleaned results to disk and sending them to a configured webhook for downstream consumption.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview**: This block sets up the workflow trigger and initializes input parameters such as the search query, the type of search engine to use, and the webhook notification URL.
- **Nodes Involved**:
  - When clicking ‘Test workflow’
  - Bright Data MCP Client List Tools
  - Set the Input Fields

##### Node Details

- **When clicking ‘Test workflow’**
  - Type: Manual Trigger
  - Role: Entry point for manual execution of the workflow.
  - Config: No parameters; triggers the workflow on demand.
  - Connections: Output → Bright Data MCP Client List Tools
  - Edge cases: No input, manual only; no auth required.
  
- **Bright Data MCP Client List Tools**
  - Type: MCP Client API node
  - Role: Connects to Bright Data MCP to list available client tools (preparatory).
  - Config: Uses MCP Client API credentials.
  - Connections: Output → Set the Input Fields
  - Edge cases: API auth failure, timeout, MCP service unavailability.
  
- **Set the Input Fields**
  - Type: Set node
  - Role: Defines static input fields for the workflow: 
    - `query` = "Bright Data"
    - `action` = "Perform Bing search"
    - `webhook_notification_url` = example webhook URL for notifications
  - Connections: Output → Bright Data Search AI Agent
  - Edge cases: Misconfigured or missing fields can cause downstream errors.

---

#### 2.2 Search Execution via Bright Data MCP Tools

- **Overview**: Executes the search on three different engines (Google, Bing, Yandex) using Bright Data MCP Client tools. Each search tool is called as an AI tool by the search AI agent.
- **Nodes Involved**:
  - MCP Client for Google Search
  - MCP Client for Bing Search
  - MCP Client for Yandex Search

##### Node Details

- **MCP Client for Google Search**
  - Type: MCP Client Tool Node
  - Role: Performs Google search via Bright Data MCP.
  - Config: Search engine = "google", query from input JSON (`{{ $json.query }}`).
  - Credentials: MCP Client API credentials.
  - Connections: AI tool input for Bright Data Search AI Agent.
  - Edge cases: MCP API limits, bad query input, network issues.
  
- **MCP Client for Bing Search**
  - Type: MCP Client Tool Node
  - Role: Performs Bing search via Bright Data MCP.
  - Config: Search engine = "bing", query from input JSON.
  - Credentials: MCP Client API credentials.
  - Connections: AI tool input for Bright Data Search AI Agent.
  - Edge cases: Same as Google.
  
- **MCP Client for Yandex Search**
  - Type: MCP Client Tool Node
  - Role: Performs Yandex search via Bright Data MCP.
  - Config: Search engine = "yandex", query from input JSON.
  - Credentials: MCP Client API credentials.
  - Connections: AI tool input for Bright Data Search AI Agent.
  - Edge cases: Same as above.

---

#### 2.3 AI Agent Processing with Google Gemini

- **Overview**: The Bright Data Search AI Agent node orchestrates the search operation based on the input action and utilizes Google Gemini to interpret and process search results.
- **Nodes Involved**:
  - Bright Data Search AI Agent
  - Google Gemini Chat Model for Search Agent

##### Node Details

- **Bright Data Search AI Agent**
  - Type: LangChain Agent Node
  - Role: Controls the search flow and manages interaction with MCP search tools.
  - Config: Receives action text (e.g., "Perform Bing search") and ensures output is the raw tool response.
  - Connections:
    - Input: Set the Input Fields (main), MCP Client search tools (ai_tool)
    - Output: Human Readable Data Extractor
  - Edge cases: Incorrect action text leads to no tool execution; output parsing failures.
  
- **Google Gemini Chat Model for Search Agent**
  - Type: LangChain Google Gemini LLM Node
  - Role: Provides language model capabilities to the AI agent.
  - Config: Uses model "models/gemini-2.0-flash-exp".
  - Credentials: Google Gemini (PaLM) API credentials.
  - Connections: Output → Bright Data Search AI Agent (ai_languageModel).
  - Edge cases: API quota exceeded, model unavailability, latency.

---

#### 2.4 Data Extraction and Cleanup

- **Overview**: This block cleans and extracts human-readable data from the raw search responses using an AI-powered information extractor.
- **Nodes Involved**:
  - Human Readable Data Extractor
  - Google Gemini Chat Model for Human Readable Data Extractor
  - Create a binary data

##### Node Details

- **Human Readable Data Extractor**
  - Type: LangChain Information Extractor Node
  - Role: AI-based cleaning of the search result text, removing HTML, ads, irrelevant text.
  - Config: Prompt instructs to produce clean, human-readable output without extra commentary.
  - Input: Output from Bright Data Search AI Agent.
  - Output: Cleaned text.
  - Connections: 
    - Output → Create a binary data
    - Output → Webhook for clean data extractor
  - Edge cases: LLM misinterpretation, incomplete cleaning, malformed input.
  
- **Google Gemini Chat Model for Human Readable Data Extractor**
  - Type: LangChain Google Gemini LLM Node
  - Role: Provides LLM support to the Human Readable Data Extractor.
  - Config: Uses same model as search agent.
  - Credentials: Google Gemini (PaLM) API credentials.
  - Connections: Output → Human Readable Data Extractor (ai_languageModel).
  - Edge cases: Same as above.
  
- **Create a binary data**
  - Type: Function Node
  - Role: Converts the JSON cleaned search result into base64-encoded binary data for file writing.
  - Config: Encodes the first item’s JSON into base64 string.
  - Connections: Output → Write the search result to disk.
  - Edge cases: Encoding errors with unexpected input.

---

#### 2.5 Output Handling

- **Overview**: Writes cleaned search results to disk and sends a notification containing the results to a configured webhook URL.
- **Nodes Involved**:
  - Write the search result to disk
  - Webhook for clean data extractor

##### Node Details

- **Write the search result to disk**
  - Type: Read/Write File Node
  - Role: Saves cleaned search results as a JSON file on disk.
  - Config: Writes to file path `d:\Scraped-Search-Results.json`.
  - Connections: Input from Create a binary data.
  - Edge cases: File write permissions, disk full, invalid path.
  
- **Webhook for clean data extractor**
  - Type: HTTP Request Node
  - Role: Sends an HTTP POST request with the cleaned search results to the configured webhook URL.
  - Config: URL dynamically taken from `Set the Input Fields.node.json.webhook_notification_url`.
  - Body Parameter: `response` with cleaned search results.
  - Connections: Input from Human Readable Data Extractor.
  - Edge cases: Webhook URL down, invalid URL, request timeout, network failures.

---

#### 2.6 Sticky Notes (Contextual Comments)

- **Sticky Note2**: Disclaimer that this template requires n8n self-hosted due to the community MCP Client node usage.
- **Sticky Note3**: Notes on workflow’s purpose—to handle Google, Bing, and Yandex search via Bright Data MCP Client; instructions for setting input fields and testing with webhook.site.
- **Sticky Note4**: Notes that Google Gemini LLM is used for structured data extraction.
- **Sticky Note5**: Displays Bright Data logo.

---

### 3. Summary Table

| Node Name                          | Node Type                        | Functional Role                                | Input Node(s)                    | Output Node(s)                             | Sticky Note                                                                                             |
|-----------------------------------|---------------------------------|-----------------------------------------------|---------------------------------|--------------------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                  | Workflow entry trigger                         | -                               | Bright Data MCP Client List Tools           |                                                                                                       |
| Bright Data MCP Client List Tools  | MCP Client API Node             | Lists MCP client tools                          | When clicking ‘Test workflow’   | Set the Input Fields                        |                                                                                                       |
| Set the Input Fields               | Set Node                       | Defines initial inputs: query, action, webhook | Bright Data MCP Client List Tools | Bright Data Search AI Agent                 | See Sticky Note3: Instructions on input fields and testing with webhook.site                           |
| MCP Client for Google Search       | MCP Client Tool Node            | Performs Google search via MCP                  | -                               | Bright Data Search AI Agent (ai_tool)       |                                                                                                       |
| MCP Client for Bing Search         | MCP Client Tool Node            | Performs Bing search via MCP                    | -                               | Bright Data Search AI Agent (ai_tool)       |                                                                                                       |
| MCP Client for Yandex Search       | MCP Client Tool Node            | Performs Yandex search via MCP                  | -                               | Bright Data Search AI Agent (ai_tool)       |                                                                                                       |
| Bright Data Search AI Agent        | LangChain Agent Node            | Orchestrates search based on input action      | Set the Input Fields; MCP Client Tool nodes | Human Readable Data Extractor               | See Sticky Note4: Uses Google Gemini LLM for structured data extraction                                |
| Google Gemini Chat Model for Search Agent | LangChain Google Gemini LLM Node | Provides LLM capabilities for search agent     | -                               | Bright Data Search AI Agent (ai_languageModel) |                                                                                                       |
| Human Readable Data Extractor      | LangChain Information Extractor | Cleans raw search results into human-readable | Bright Data Search AI Agent     | Create a binary data; Webhook for clean data extractor |                                                                                                       |
| Google Gemini Chat Model for Human Readable Data Extractor | LangChain Google Gemini LLM Node | Provides LLM capabilities for data extractor   | -                               | Human Readable Data Extractor (ai_languageModel) |                                                                                                       |
| Create a binary data               | Function Node                  | Encodes cleaned JSON data to base64 binary     | Human Readable Data Extractor   | Write the search result to disk             |                                                                                                       |
| Write the search result to disk    | Read/Write File Node           | Saves cleaned results as JSON file              | Create a binary data            | -                                          |                                                                                                       |
| Webhook for clean data extractor  | HTTP Request Node              | Sends cleaned results to configured webhook    | Human Readable Data Extractor   | -                                          |                                                                                                       |
| Sticky Note2                      | Sticky Note                   | Disclaimer about self-hosting requirement       | -                               | -                                          | This template is only available on n8n self-hosted due to MCP Client community node use                |
| Sticky Note3                      | Sticky Note                   | Workflow purpose and input setup instructions   | -                               | -                                          | Deals with Google, Bing, Yandex search via Bright Data MCP; set query, action, webhook URL; test with webhook.site |
| Sticky Note4                      | Sticky Note                   | LLM usage note                                  | -                               | -                                          | Google Gemini LLM is used for structured data extraction                                              |
| Sticky Note5                      | Sticky Note                   | Branding (Bright Data logo)                      | -                               | -                                          | ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking ‘Test workflow’`
   - No inputs needed.

2. **Add MCP Client List Tools Node**
   - Type: MCP Client (Community Node)
   - Name: `Bright Data MCP Client List Tools`
   - Credentials: Assign MCP Client API credentials (e.g., "MCP Client (STDIO) account").
   - Connect output of Manual Trigger → MCP Client List Tools (main).

3. **Add Set Node for Input Fields**
   - Type: Set
   - Name: `Set the Input Fields`
   - Add fields:
     - `query` (string): "Bright Data"
     - `action` (string): "Perform Bing search"
     - `webhook_notification_url` (string): "https://webhook.site/c9118da2-1c54-460f-a83a-e5131b7098db"
   - Connect MCP Client List Tools (main) → Set the Input Fields (main).

4. **Add MCP Client Tool Nodes for Search Engines**
   - For Google Search:
     - Type: MCP Client Tool
     - Name: `MCP Client for Google Search`
     - Tool Name: "search_engine"
     - Operation: "executeTool"
     - Tool Parameters (expression):
       ```
       {
         "query": "={{ $json.query }}",
         "engine": "google"
       }
       ```
     - Credentials: MCP Client API credentials.
   - For Bing Search:
     - Same as above but engine = "bing".
   - For Yandex Search:
     - Same as above but engine = "yandex".

5. **Add Google Gemini Chat Model Node for Search Agent**
   - Type: LangChain Google Gemini LLM Node
   - Name: `Google Gemini Chat Model for Search Agent`
   - Model Name: "models/gemini-2.0-flash-exp"
   - Credentials: Google Gemini (PaLM) API credentials.

6. **Add Bright Data Search AI Agent Node**
   - Type: LangChain Agent Node
   - Name: `Bright Data Search AI Agent`
   - Parameters:
     - Text: Expression `={{ $json.action }}\n\nMake sure to output the response as returned by th specific tool.`
     - Prompt Type: Define
     - Has Output Parser: Checked/true
   - Connect inputs:
     - Main input: From `Set the Input Fields`
     - AI Language Model input: From `Google Gemini Chat Model for Search Agent`
     - AI Tool inputs: From all MCP Client Tool nodes (Google, Bing, Yandex)
   - Output main → Human Readable Data Extractor

7. **Add Google Gemini Chat Model Node for Data Extraction**
   - Type: LangChain Google Gemini LLM Node
   - Name: `Google Gemini Chat Model for Human Readable Data Extractor`
   - Model Name: "models/gemini-2.0-flash-exp"
   - Credentials: Google Gemini (PaLM) API credentials.

8. **Add Human Readable Data Extractor Node**
   - Type: LangChain Information Extractor Node
   - Name: `Human Readable Data Extractor`
   - Text parameter (expression):
     ```
     =You are a helpful AI assistant. Given the following search result, return a clean, human-readable information.

     Remove any HTML tags, Ignore irrelevant links, ads, navigation text, or footers.

     Here's the content -  {{ $json.output }}

     Important - Do not output your own thoughts or suggestions.
     ```
   - Attributes: `search_response` (description: "Search Response")
   - Connect inputs:
     - Main from `Bright Data Search AI Agent`
     - AI Language Model input from `Google Gemini Chat Model for Human Readable Data Extractor`
   - Outputs:
     - Main → Create a binary data
     - Main → Webhook for clean data extractor

9. **Add Function Node to Create Binary Data**
   - Type: Function
   - Name: `Create a binary data`
   - Code:
     ```javascript
     items[0].binary = {
       data: {
         data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
       }
     };
     return items;
     ```
   - Connect input from `Human Readable Data Extractor`
   - Output main → Write the search result to disk

10. **Add Write File Node**
    - Type: Read/Write File
    - Name: `Write the search result to disk`
    - Operation: Write
    - File Name: `d:\Scraped-Search-Results.json`
    - Connect input from `Create a binary data`

11. **Add HTTP Request Node for Webhook Notification**
    - Type: HTTP Request
    - Name: `Webhook for clean data extractor`
    - HTTP Method: POST
    - URL: Expression from `Set the Input Fields` node: `={{ $('Set the Input Fields').item.json.webhook_notification_url }}`
    - Body Parameters:
      - Name: `response`
      - Value: Expression `={{ $json.output.search_response }}`
    - Send Body: true
    - Connect input from `Human Readable Data Extractor`

12. **Add Sticky Notes (Optional for documentation)**
    - Add notes about self-hosting requirements, LLM usage, input fields, testing instructions, and branding as per the original.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow requires n8n self-hosted because it uses the community MCP Client node.                | Sticky Note2                                                                                                           |
| Use https://webhook.site/ to test webhook notifications by setting the webhook_notification_url input. | Sticky Note3                                                                                                           |
| Google Gemini LLM ("models/gemini-2.0-flash-exp") is used for both search agent orchestration and data extraction. | Sticky Note4                                                                                                           |
| Bright Data logo included for branding: ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) | Sticky Note5                                                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is public and lawful.