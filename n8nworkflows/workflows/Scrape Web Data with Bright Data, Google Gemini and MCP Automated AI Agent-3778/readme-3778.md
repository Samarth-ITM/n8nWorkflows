Scrape Web Data with Bright Data, Google Gemini and MCP Automated AI Agent

https://n8nworkflows.xyz/workflows/scrape-web-data-with-bright-data--google-gemini-and-mcp-automated-ai-agent-3778


# Scrape Web Data with Bright Data, Google Gemini and MCP Automated AI Agent

### 1. Workflow Overview

This workflow automates large-scale web data extraction by integrating Bright Data’s MCP Server tools with Google Gemini AI for intelligent scraping and data processing. It is designed for professionals needing structured, enriched web data for analysis, marketing research, product insights, AI training, or growth hacking.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initialization**: Manual trigger to start the workflow and initial setup of URLs and webhook endpoints.
- **1.2 Bright Data MCP Tool Discovery and URL Setting**: Listing available MCP scraping tools and setting target URLs.
- **1.3 AI Agent Processing with Google Gemini**: Using Google Gemini to interpret scraping requests and select appropriate MCP tools.
- **1.4 Web Scraping Execution via MCP Client**: Executing the scraping task using MCP tools to retrieve web content in Markdown or HTML.
- **1.5 Output Handling and Notification**: Sending scraped data to a webhook endpoint and writing results to disk.
- **1.6 Memory and Context Management**: Maintaining session memory for AI agent context.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block starts the workflow manually and initializes the URLs and webhook endpoints used throughout the scraping process.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - MCP Client list all tools for Bright Data  
  - Set the URLs  
  - Set the URL with the Webhook URL and data format

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Configuration: No parameters, simply a manual start.  
    - Connections: Outputs to "MCP Client list all tools for Bright Data" and "Set the URL with the Webhook URL and data format".  
    - Edge Cases: None typical, but workflow won’t run unless manually triggered.

  - **MCP Client list all tools for Bright Data**  
    - Type: MCP Client (mcpClient)  
    - Role: Queries Bright Data MCP Server to list all available scraping tools.  
    - Configuration: Uses MCP Client (STDIO) credentials for authentication.  
    - Input: Triggered by manual start.  
    - Output: Passes data to "Set the URLs".  
    - Edge Cases: Authentication failure, MCP server unavailability, or empty tool list.

  - **Set the URLs**  
    - Type: Set  
    - Role: Defines static URLs and webhook URLs for scraping and notifications.  
    - Configuration: Sets two string variables:  
      - `url`: "https://about.google/"  
      - `webhook_url`: "https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7"  
    - Input: From "MCP Client list all tools for Bright Data".  
    - Output: Passes URL data to "MCP Client Bright Data Web Scraper".  
    - Edge Cases: Hardcoded URLs require manual update for different targets.

  - **Set the URL with the Webhook URL and data format**  
    - Type: Set  
    - Role: Prepares URL, webhook endpoint, and scraping format for AI Agent.  
    - Configuration: Sets three variables:  
      - `url`: "https://about.google/"  
      - `webhook_url`: "https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467"  
      - `format`: "scrape_as_markdown"  
    - Input: From "MCP Client list all tools for Bright Data".  
    - Output: Feeds into "AI Agent".  
    - Edge Cases: Hardcoded values require manual changes for different scraping targets or formats.

---

#### 2.2 Bright Data MCP Tool Discovery and URL Setting

- **Overview:**  
  This block discovers available MCP scraping tools and sets the URLs for scraping, preparing the workflow for the AI agent and scraping execution.

- **Nodes Involved:**  
  - MCP Client List all tools (mcpClientTool)  
  - MCP Client to Scrape as Markdown (mcpClientTool)  
  - MCP Client to Scrape as HTML (mcpClientTool)  
  - Sticky Note (Bright Data Web Scraper Tools)

- **Node Details:**

  - **MCP Client List all tools**  
    - Type: MCP Client Tool  
    - Role: Lists all MCP tools available on Bright Data MCP Server.  
    - Configuration: Uses MCP Client (STDIO) credentials.  
    - Input: None (standalone or triggered by AI Agent).  
    - Output: Feeds into AI Agent as available tools.  
    - Edge Cases: Possible connection/authentication errors.

  - **MCP Client to Scrape as Markdown**  
    - Type: MCP Client Tool  
    - Role: Executes scraping of a single webpage URL, returning content in Markdown.  
    - Configuration:  
      - Tool name: "scrape_as_markdown"  
      - Operation: executeTool  
      - Tool parameters: JSON with `"url": "{{ $json.url }}"`  
      - Credentials: MCP Client (STDIO)  
    - Input: AI Agent (as ai_tool)  
    - Output: AI Agent (ai_tool)  
    - Edge Cases: Invalid URL, scraping failure, MCP server errors.

  - **MCP Client to Scrape as HTML**  
    - Type: MCP Client Tool  
    - Role: Executes scraping of a single webpage URL, returning content in HTML.  
    - Configuration:  
      - Tool name: "scrape_as_html"  
      - Operation: executeTool  
      - Tool parameters: JSON with `"url": "{{ $json.url }}"`  
      - Credentials: MCP Client (STDIO)  
    - Input: AI Agent (as ai_tool)  
    - Output: AI Agent (ai_tool)  
    - Edge Cases: Same as Markdown scraping node.

  - **Sticky Note (Bright Data Web Scraper Tools)**  
    - Type: Sticky Note  
    - Role: Provides a descriptive label for the block.  
    - Content: "## Bright Data Web Scraper Tools"  
    - Position: Near MCP Client tool nodes.

---

#### 2.3 AI Agent Processing with Google Gemini

- **Overview:**  
  This block uses Google Gemini AI to interpret the scraping request, select the appropriate MCP tool, and orchestrate the scraping process intelligently.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent)  
  - Google Gemini Chat Model for AI Agent (lmChatGoogleGemini)  
  - Simple Memory (memoryBufferWindow)  
  - MCP Client List all tools (mcpClient)  
  - Sticky Note3 (AI agent explanation)

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Acts as the intelligent orchestrator that receives the URL and format, uses Google Gemini to understand the request, and calls the appropriate MCP scraping tool.  
    - Configuration:  
      - Text prompt: "Scrape the web data as per the provided URL:  {{ $json.url }} using the format as {{ $json.format }}"  
      - System message: "You are a helpful assistant."  
      - Prompt type: define  
    - Inputs:  
      - Receives AI language model output from Google Gemini Chat Model  
      - Receives AI tools from MCP Client List all tools, MCP Client to Scrape as Markdown, MCP Client to Scrape as HTML  
      - Receives AI memory from Simple Memory  
    - Outputs:  
      - Sends main output to "Webhook for Web Scraper AI Agent" and "Create a binary data"  
    - Edge Cases: Expression evaluation errors, AI model API errors, tool invocation failures.

  - **Google Gemini Chat Model for AI Agent**  
    - Type: Langchain Google Gemini Chat Model  
    - Role: Provides natural language understanding and generation capabilities to the AI Agent.  
    - Configuration:  
      - Model name: "models/gemini-2.0-flash-exp"  
      - Credentials: Google Gemini (PaLM) API key  
    - Input: AI Agent (ai_languageModel)  
    - Output: AI Agent (ai_languageModel)  
    - Edge Cases: API key invalid, rate limits, network errors.

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Maintains session context for the AI Agent to remember previous interactions or instructions.  
    - Configuration:  
      - Session key: Custom key with text "Perform the web scraping for the below URL\n\n{{ $json.url }}"  
      - Context window length: 10  
    - Input: AI Agent (ai_memory)  
    - Output: AI Agent (ai_memory)  
    - Edge Cases: Memory overflow or session key misconfiguration.

  - **MCP Client List all tools**  
    - Type: MCP Client (mcpClient)  
    - Role: Provides the AI Agent with a list of available MCP tools to choose from.  
    - Configuration: Uses MCP Client (STDIO) credentials.  
    - Input: AI Agent (ai_tool)  
    - Output: AI Agent (ai_tool)  
    - Edge Cases: Same as previous MCP Client list node.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Content:  
      ```
      ## Note
      The AI agent utilizes Bright Data's MCP tools to perform web scraping based on user requests. It intelligently selects the most suitable web scraping tool to fulfill the user's query.

      Once the web scraping is complete, the AI agent's response is:

      1. Used to trigger a webhook call.

      2. Persisted to disk for future reference.

      Google Gemini is employed by the AI agent to understand and interpret user queries. Based on this interpretation, the agent initiates a call to the appropriate MCP client to perform the required web scraping task.

      Source - https://github.com/luminati-io/brightdata-mcp
      ```
    - Position: Near AI Agent node.

---

#### 2.4 Web Scraping Execution via MCP Client

- **Overview:**  
  This block performs the actual web scraping using the MCP Client tool "scrape_as_markdown" and sends the scraped content to a webhook.

- **Nodes Involved:**  
  - MCP Client Bright Data Web Scraper (mcpClient)  
  - Webhook for web scraper (httpRequest)  
  - Sticky Note1 (Bright Data Web Scraper)

- **Node Details:**

  - **MCP Client Bright Data Web Scraper**  
    - Type: MCP Client  
    - Role: Executes the "scrape_as_markdown" tool on the MCP Server to scrape the specified URL.  
    - Configuration:  
      - Tool name: "scrape_as_markdown"  
      - Operation: executeTool  
      - Tool parameters: JSON with `"url": "{{ $json.url }}"`  
      - Credentials: MCP Client (STDIO)  
    - Input: From "Set the URLs" node.  
    - Output: Passes scraped results to "Webhook for web scraper".  
    - Edge Cases: Invalid URL, MCP server errors, network issues.

  - **Webhook for web scraper**  
    - Type: HTTP Request  
    - Role: Sends the scraped content to a specified webhook endpoint.  
    - Configuration:  
      - URL: Hardcoded to "https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467" (can be parameterized)  
      - Method: POST (default)  
      - Body parameters: Sends the scraped content text from `{{$json.result.content[0].text}}` as "response".  
    - Input: From MCP Client Bright Data Web Scraper.  
    - Output: None (terminal node).  
    - Edge Cases: Webhook endpoint unavailability, network errors.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: "## Bright Data Web Scraper"  
    - Position: Near MCP Client Bright Data Web Scraper node.

---

#### 2.5 Output Handling and Notification

- **Overview:**  
  This block processes the AI Agent’s output by sending it to a webhook and saving the scraped data to disk in JSON format.

- **Nodes Involved:**  
  - Webhook for Web Scraper AI Agent (httpRequest)  
  - Create a binary data (Function)  
  - Write the scraped content to disk (readWriteFile)

- **Node Details:**

  - **Webhook for Web Scraper AI Agent**  
    - Type: HTTP Request  
    - Role: Sends the AI Agent’s output to a configurable webhook endpoint.  
    - Configuration:  
      - URL: Dynamically set from `{{ $('Set the URL with the Webhook URL and data format').item.json.webhook_url }}`  
      - Method: POST  
      - Body parameters: Sends `{{$json.output}}` as "response".  
    - Input: From AI Agent node.  
    - Output: None (terminal).  
    - Edge Cases: Webhook endpoint errors, network failures.

  - **Create a binary data**  
    - Type: Function  
    - Role: Converts the JSON scraped data into a base64-encoded binary format for file writing.  
    - Configuration:  
      - JavaScript code creates a binary property `data` with base64-encoded JSON string of the first item.  
    - Input: From AI Agent node.  
    - Output: Passes binary data to "Write the scraped content to disk".  
    - Edge Cases: Buffer encoding errors, malformed JSON.

  - **Write the scraped content to disk**  
    - Type: Read/Write File  
    - Role: Writes the binary scraped content to a local file.  
    - Configuration:  
      - Operation: write  
      - File name: "d:\Scraped-Content.json" (Windows path)  
    - Input: From "Create a binary data".  
    - Output: None (terminal).  
    - Edge Cases: File system permission errors, invalid path.

---

#### 2.6 Memory and Context Management

- **Overview:**  
  Maintains conversational or session context for the AI Agent to improve understanding and continuity.

- **Nodes Involved:**  
  - Simple Memory (memoryBufferWindow)

- **Node Details:**

  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window  
    - Role: Stores recent context for AI Agent sessions keyed by a custom session key including the URL.  
    - Configuration:  
      - Session key: Custom text including URL  
      - Context window length: 10 messages  
    - Input: AI Agent (ai_memory)  
    - Output: AI Agent (ai_memory)  
    - Edge Cases: Memory size limits, session key collisions.

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                              | Input Node(s)                              | Output Node(s)                              | Sticky Note                                                                                          |
|-----------------------------------|-------------------------------|----------------------------------------------|--------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                | Manual start trigger                          | -                                          | MCP Client list all tools for Bright Data, Set the URL with the Webhook URL and data format         |                                                                                                    |
| MCP Client list all tools for Bright Data | MCP Client (mcpClient)        | Lists available MCP scraping tools           | When clicking ‘Test workflow’               | Set the URLs, Set the URL with the Webhook URL and data format                                     |                                                                                                    |
| Set the URLs                      | Set                           | Defines static URLs and webhook URLs          | MCP Client list all tools for Bright Data  | MCP Client Bright Data Web Scraper                                                                |                                                                                                    |
| Set the URL with the Webhook URL and data format | Set                           | Prepares URL, webhook, and format for AI Agent | MCP Client list all tools for Bright Data  | AI Agent                                                                                          |                                                                                                    |
| AI Agent                         | Langchain Agent               | Orchestrates scraping task using AI and MCP tools | Set the URL with the Webhook URL and data format, MCP Client List all tools, MCP Client to Scrape as Markdown, MCP Client to Scrape as HTML, Simple Memory, Google Gemini Chat Model for AI Agent | Webhook for Web Scraper AI Agent, Create a binary data                                            | Bright Data Web Scraping Agent                                                                     |
| Google Gemini Chat Model for AI Agent | Langchain Google Gemini Chat Model | Provides AI language understanding            | AI Agent (ai_languageModel)                  | AI Agent (ai_languageModel)                  |                                                                                                    |
| Simple Memory                    | Langchain Memory Buffer Window | Maintains AI session context                   | AI Agent (ai_memory)                         | AI Agent (ai_memory)                         |                                                                                                    |
| MCP Client List all tools        | MCP Client Tool               | Provides AI Agent with MCP tools list          | AI Agent (ai_tool)                           | AI Agent (ai_tool)                           |                                                                                                    |
| MCP Client to Scrape as Markdown | MCP Client Tool               | Scrapes webpage returning Markdown content    | AI Agent (ai_tool)                           | AI Agent (ai_tool)                           | Scrape a single webpage URL with advanced options and get results in MarkDown language.            |
| MCP Client to Scrape as HTML     | MCP Client Tool               | Scrapes webpage returning HTML content        | AI Agent (ai_tool)                           | AI Agent (ai_tool)                           | Scrape a single webpage URL with advanced options and get results in HTML.                         |
| MCP Client Bright Data Web Scraper | MCP Client (mcpClient)        | Executes scraping as Markdown                   | Set the URLs                                 | Webhook for web scraper                      | Scrape a single webpage URL with advanced options and get results in MarkDown language.            |
| Webhook for web scraper          | HTTP Request                 | Sends scraped content to webhook endpoint      | MCP Client Bright Data Web Scraper           | -                                            |                                                                                                    |
| Webhook for Web Scraper AI Agent | HTTP Request                 | Sends AI Agent output to webhook endpoint      | AI Agent                                     | -                                            |                                                                                                    |
| Create a binary data             | Function                     | Converts JSON scraped data to base64 binary    | AI Agent                                     | Write the scraped content to disk            |                                                                                                    |
| Write the scraped content to disk | Read/Write File              | Saves scraped data to local disk                | Create a binary data                          | -                                            |                                                                                                    |
| Sticky Note1                    | Sticky Note                  | Label for Bright Data Web Scraper block         | -                                            | -                                            | ## Bright Data Web Scraper                                                                         |
| Sticky Note                     | Sticky Note                  | Label for Bright Data Web Scraper Tools block   | -                                            | -                                            | ## Bright Data Web Scraper Tools                                                                   |
| Sticky Note2                    | Sticky Note                  | Disclaimer about self-hosted MCP Client node    | -                                            | -                                            | ## Disclaimer This template is only available on n8n self-hosted as it's making use of the community node for MCP Client. |
| Sticky Note3                    | Sticky Note                  | Explanation of AI Agent role and workflow logic | -                                            | -                                            | ## Note The AI agent utilizes Bright Data's MCP tools to perform web scraping based on user requests. It intelligently selects the most suitable web scraping tool to fulfill the user's query. Once the web scraping is complete, the AI agent's response is: 1. Used to trigger a webhook call. 2. Persisted to disk for future reference. Google Gemini is employed by the AI agent to understand and interpret user queries. Based on this interpretation, the agent initiates a call to the appropriate MCP client to perform the required web scraping task. Source - https://github.com/luminati-io/brightdata-mcp |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters needed.

2. **Add MCP Client node to list all tools**  
   - Name: "MCP Client list all tools for Bright Data"  
   - Credentials: Configure MCP Client (STDIO) with Bright Data MCP Server API token.  
   - No parameters needed.

3. **Connect Manual Trigger → MCP Client list all tools for Bright Data**

4. **Add a Set node to define URLs and webhook URLs**  
   - Name: "Set the URLs"  
   - Assign variables:  
     - `url` = "https://about.google/"  
     - `webhook_url` = "https://webhook.site/ce41e056-c097-48c8-a096-9b876d3abbf7"

5. **Connect MCP Client list all tools for Bright Data → Set the URLs**

6. **Add MCP Client node to scrape webpage as Markdown**  
   - Name: "MCP Client Bright Data Web Scraper"  
   - Credentials: MCP Client (STDIO)  
   - Parameters:  
     - Tool name: "scrape_as_markdown"  
     - Operation: executeTool  
     - Tool parameters: JSON `{ "url": "{{ $json.url }}" }`

7. **Connect Set the URLs → MCP Client Bright Data Web Scraper**

8. **Add HTTP Request node to send scraped content to webhook**  
   - Name: "Webhook for web scraper"  
   - URL: "https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467" (or your webhook endpoint)  
   - Method: POST  
   - Body parameters:  
     - Name: "response"  
     - Value: `{{$json.result.content[0].text}}`

9. **Connect MCP Client Bright Data Web Scraper → Webhook for web scraper**

10. **Add another Set node to prepare AI Agent input**  
    - Name: "Set the URL with the Webhook URL and data format"  
    - Assign variables:  
      - `url` = "https://about.google/"  
      - `webhook_url` = "https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467"  
      - `format` = "scrape_as_markdown"

11. **Connect MCP Client list all tools for Bright Data → Set the URL with the Webhook URL and data format**

12. **Add Langchain Google Gemini Chat Model node**  
    - Name: "Google Gemini Chat Model for AI Agent"  
    - Model name: "models/gemini-2.0-flash-exp"  
    - Credentials: Google Gemini (PaLM) API key

13. **Add Langchain Memory Buffer Window node**  
    - Name: "Simple Memory"  
    - Session key: Custom key with text including URL  
    - Context window length: 10

14. **Add Langchain Agent node**  
    - Name: "AI Agent"  
    - Text prompt: "Scrape the web data as per the provided URL:  {{ $json.url }} using the format as {{ $json.format }}"  
    - System message: "You are a helpful assistant."  
    - Prompt type: define  
    - Connect inputs:  
      - AI language model input from "Google Gemini Chat Model for AI Agent"  
      - AI memory input from "Simple Memory"  
      - AI tools input from:  
        - MCP Client List all tools (mcpClient)  
        - MCP Client to Scrape as Markdown (mcpClientTool)  
        - MCP Client to Scrape as HTML (mcpClientTool)  
    - Credentials for MCP Client nodes: MCP Client (STDIO)

15. **Connect Set the URL with the Webhook URL and data format → AI Agent**

16. **Add MCP Client List all tools node (mcpClientTool)**  
    - Name: "MCP Client List all tools"  
    - Credentials: MCP Client (STDIO)

17. **Add MCP Client to Scrape as Markdown node**  
    - Name: "MCP Client to Scrape as Markdown"  
    - Tool name: "scrape_as_markdown"  
    - Operation: executeTool  
    - Tool parameters: `{ "url": "{{ $json.url }}" }`  
    - Credentials: MCP Client (STDIO)

18. **Add MCP Client to Scrape as HTML node**  
    - Name: "MCP Client to Scrape as HTML"  
    - Tool name: "scrape_as_html"  
    - Operation: executeTool  
    - Tool parameters: `{ "url": "{{ $json.url }}" }`  
    - Credentials: MCP Client (STDIO)

19. **Connect MCP Client List all tools, MCP Client to Scrape as Markdown, MCP Client to Scrape as HTML → AI Agent (ai_tool inputs)**

20. **Connect Google Gemini Chat Model for AI Agent → AI Agent (ai_languageModel input)**

21. **Connect Simple Memory → AI Agent (ai_memory input)**

22. **Add HTTP Request node to send AI Agent output to webhook**  
    - Name: "Webhook for Web Scraper AI Agent"  
    - URL: `{{ $('Set the URL with the Webhook URL and data format').item.json.webhook_url }}`  
    - Method: POST  
    - Body parameters:  
      - Name: "response"  
      - Value: `{{$json.output}}`

23. **Connect AI Agent → Webhook for Web Scraper AI Agent**

24. **Add Function node to create binary data for file writing**  
    - Name: "Create a binary data"  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```

25. **Connect AI Agent → Create a binary data**

26. **Add Read/Write File node to save scraped content**  
    - Name: "Write the scraped content to disk"  
    - Operation: write  
    - File name: "d:\\Scraped-Content.json" (adjust path as needed)

27. **Connect Create a binary data → Write the scraped content to disk**

28. **Add Sticky Notes as per workflow for documentation and clarity**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires n8n self-hosted due to the use of the community MCP Client node.                                                                                                                                                                                                                                                                              | Disclaimer in workflow description                                                                  |
| Setup instructions for MCP Server and Bright Data MCP Client are available at https://www.youtube.com/watch?v=NUb73ErUCsA and https://www.npmjs.com/package/@brightdata/mcp                                                                                                                                                                                           | Setup prerequisites                                                                                  |
| Bright Data Web Unlocker proxy zone "mcp_unlocker" must be created in Bright Data control panel for scraping.                                                                                                                                                                                                                                                       | Bright Data configuration                                                                           |
| Google Gemini API key or access via Vertex AI or proxy must be configured in n8n credentials.                                                                                                                                                                                                                                                                         | Google Gemini API setup                                                                             |
| MCP Client (STDIO) credentials require the Bright Data API_TOKEN to be set in environment variables.                                                                                                                                                                                                                                                                 | MCP Client credential setup                                                                         |
| The AI Agent intelligently selects the appropriate MCP scraping tool based on the user’s URL and format request using Google Gemini natural language understanding.                                                                                                                                                                                                 | Sticky Note3 content                                                                                |
| Webhook endpoints used in the workflow can be customized to integrate with Slack, Airtable, Notion, CRM systems, or other automation targets.                                                                                                                                                                                                                        | Customization suggestions                                                                           |
| Source code and examples for Bright Data MCP tools are available at https://github.com/luminati-io/brightdata-mcp                                                                                                                                                                                                                                                   | External resource                                                                                   |

---

This comprehensive reference document enables advanced users and AI agents to understand, reproduce, and customize the "Scrape Web Data with Bright Data, Google Gemini and MCP Automated AI Agent" workflow effectively.