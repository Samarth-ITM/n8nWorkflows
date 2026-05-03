Enhance Chat Responses with Real-Time Search Data via Bright Data & Gemini AI

https://n8nworkflows.xyz/workflows/enhance-chat-responses-with-real-time-search-data-via-bright-data---gemini-ai-3779


# Enhance Chat Responses with Real-Time Search Data via Bright Data & Gemini AI

### 1. Workflow Overview

This workflow, titled **"Enhance Chat Responses with Real-Time Search Data via Bright Data & Gemini AI"**, is designed to enable conversational AI agents to provide dynamically informed responses by integrating real-time web search results from Bright Data MCP Search Engines (Google, Bing, Yandex) with Google Gemini’s advanced language model capabilities. It addresses the common limitation of static, outdated knowledge in chatbots by fusing live search data with AI reasoning to generate accurate, contextually rich conversational replies.

**Target use cases** include data analysis, market and competitor research, product management insights, AI application development, and growth hacking where up-to-date information is critical.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception**: Captures user chat inputs via webhook triggers or manual test triggers.
- **1.2 MCP Tools Initialization & Search Query Preparation**: Retrieves available Bright Data MCP tools and prepares search queries.
- **1.3 Real-Time Search Execution**: Executes search engine queries (Google, Bing, Yandex) using MCP client tools.
- **1.4 AI Processing & Memory Management**: Uses Google Gemini as the chat language model and LangChain AI Agent to reason over search results, maintaining conversation context with memory buffers.
- **1.5 Webhook Notification & Output Delivery**: Sends AI-generated chat responses to external endpoints via HTTP webhook calls.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving user chat messages either through a live chat trigger or manual testing.

**Nodes Involved:**  
- When chat message received  
- When clicking ‘Test workflow’  

**Node Details:**  

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Listens for incoming chat messages via webhook trigger.  
  - *Configuration:* Default options; webhook ID predefined for external integration.  
  - *Input:* External HTTP webhook calls.  
  - *Output:* Passes chat input to the AI Agent node.  
  - *Edge cases:* Webhook downtime, malformed requests, or missing payload fields may cause trigger failures.

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual initiation of the workflow for testing purposes.  
  - *Configuration:* No parameters; triggers downstream MCP tool listing.  
  - *Input:* Manual user action.  
  - *Output:* Starts MCP Client list all tools node.  
  - *Edge cases:* Manual trigger only; no external input validation needed.

---

#### 1.2 MCP Tools Initialization & Search Query Preparation

**Overview:**  
This block discovers the available MCP client tools from Bright Data MCP Server and prepares the search query payload for further processing.

**Nodes Involved:**  
- MCP Client list all tools for Bright Data  
- MCP Client List all tools  
- Set search query  

**Node Details:**  

- **MCP Client list all tools for Bright Data**  
  - *Type:* MCP Client (STDIO) node  
  - *Role:* Lists all MCP tools available via the Bright Data MCP Server connection.  
  - *Configuration:* No parameters; uses configured MCP Client (STDIO) credentials.  
  - *Input:* Manual Trigger node output.  
  - *Output:* Outputs JSON including available tools metadata.  
  - *Edge cases:* Authentication failure, MCP service offline, or network issues.

- **MCP Client List all tools**  
  - *Type:* MCP Client Tool node  
  - *Role:* Redundant listing of MCP tools for AI Agent’s tool injection.  
  - *Configuration:* Uses MCP Client credentials; no parameters set.  
  - *Input:* None explicitly triggered except AI Agent calls.  
  - *Output:* Supplies tools info to AI Agent node.  
  - *Edge cases:* Same as above.

- **Set search query**  
  - *Type:* Set node  
  - *Role:* Initializes a static search query parameter `search_query` with value "Bright Data" as a placeholder or test value.  
  - *Configuration:* Assigns string `"Bright Data"` to `search_query`.  
  - *Input:* Output from MCP Client list all tools for Bright Data node.  
  - *Output:* Passes the search query to the MCP Client Bright Data Search Tool node.  
  - *Edge cases:* Hardcoded value limits dynamic queries unless replaced.

---

#### 1.3 Real-Time Search Execution

**Overview:**  
This block executes live search queries using Bright Data MCP Search Engine tools for Google, Bing, and Yandex, returning search engine results pages (SERPs) data.

**Nodes Involved:**  
- MCP Client Bright Data Search Tool  
- Google Search Engine for Bright Data  
- Bing Search Engine for Bright Data  
- Yandex Search Engine for Bright Data  

**Node Details:**  

- **MCP Client Bright Data Search Tool**  
  - *Type:* MCP Client node  
  - *Role:* Executes a search query tool dynamically selected from the available MCP tools list, using the first tool name returned.  
  - *Configuration:* Tool name set via expression referencing the first tool in tools array from MCP Client list all tools output; query parameter bound to `search_query` variable; search engine fixed to Google.  
  - *Input:* Output from Set search query node.  
  - *Output:* Raw search results in markdown format (URL, title, description).  
  - *Edge cases:* Tool name index out of range, invalid query, MCP client communication errors.

- **Google Search Engine for Bright Data**  
  - *Type:* MCP Client Tool node  
  - *Role:* Executes Google search engine queries via MCP.  
  - *Configuration:* ToolName: `search_engine` (specific tool for search), engine parameter: `"google"`, query parameter dynamically set from incoming chat input (`$json.chatInput`).  
  - *Input:* Routed from AI Agent as an AI tool execution.  
  - *Output:* Search results in markdown SERP format.  
  - *Edge cases:* API quota limits, malformed queries, network failures.

- **Bing Search Engine for Bright Data**  
  - *Type:* MCP Client Tool node  
  - *Role:* Executes Bing search engine queries similarly to Google node.  
  - *Configuration:* Same as Google but engine parameter: `"bing"`.  
  - *Input:* Routed from AI Agent.  
  - *Output:* Bing SERP results in markdown.  
  - *Edge cases:* As above for Bing-specific restrictions.

- **Yandex Search Engine for Bright Data**  
  - *Type:* MCP Client Tool node  
  - *Role:* Executes Yandex search engine queries similarly.  
  - *Configuration:* engine parameter: `"yandex"`.  
  - *Input:* Routed from AI Agent.  
  - *Output:* Yandex SERP markdown results.  
  - *Edge cases:* Regional restrictions, query formatting issues.

---

#### 1.4 AI Processing & Memory Management

**Overview:**  
This block manages the AI conversation flow by interpreting the user input, incorporating live search results, and maintaining context memory, ultimately generating human-like responses.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Simple Memory  

**Node Details:**  

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Orchestrates AI logic by selecting and using the MCP search tools in order, processing user input, and generating responses. Handles both tool selection and invoking the language model.  
  - *Configuration:* System message instructs the agent to use Bright Data MCP Search Engine assistant tools for Google, Bing, or Yandex and to return responses both to chat and via webhook notifications.  
  - *Input:* User chat messages from the chat trigger node.  
  - *Output:* Passes queries to MCP tools and routes final responses to the Google Gemini Chat Model and webhook notification.  
  - *Edge cases:* Tool execution failures, language model timeouts, improper tool parameterization, or expression errors.  
  - *Sub-workflow:* Coordinates other nodes as internal tools and language model components.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini LM Chat node  
  - *Role:* Provides advanced language modeling, interpreting user input and search results to generate contextually rich text responses.  
  - *Configuration:* Model set to `models/gemini-2.0-flash-exp`; credentials configured for Google Gemini (PaLM) API.  
  - *Input:* Invoked by AI Agent as the language model component.  
  - *Output:* Returns chat responses to AI Agent for further processing.  
  - *Edge cases:* API authentication errors, rate limits, model unavailability.

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains recent conversational context for the AI Agent to enable continuity in multi-turn conversations.  
  - *Configuration:* Default buffer window, no custom parameters set.  
  - *Input:* Receives conversation data from AI Agent.  
  - *Output:* Supplies context back to AI Agent for reasoning.  
  - *Edge cases:* Memory overflow or loss if node malfunctions.

---

#### 1.5 Webhook Notification & Output Delivery

**Overview:**  
This block sends the AI-generated chat responses to an external HTTP endpoint via webhook, enabling integration with chat applications or notification systems.

**Nodes Involved:**  
- HTTP Request for Webhook Notification  

**Node Details:**  

- **HTTP Request for Webhook Notification**  
  - *Type:* LangChain Tool HTTP Request node  
  - *Role:* Sends a POST request containing the AI chat response to a configurable external webhook URL.  
  - *Configuration:*  
    - URL set to https://webhook.site/daf9d591-a130-4010-b1d3-0c66f8fcf467 (placeholder endpoint).  
    - Method: POST.  
    - Body includes parameter `chat_response` with the AI agent’s generated text.  
  - *Input:* Receives chat response from AI Agent’s tool output.  
  - *Output:* No further downstream nodes; acts as endpoint notification.  
  - *Edge cases:* Webhook endpoint unavailability, HTTP timeouts, invalid response payload.

---

### 3. Summary Table

| Node Name                          | Node Type                             | Functional Role                                    | Input Node(s)                          | Output Node(s)                      | Sticky Note                                                                                                                           |
|-----------------------------------|-------------------------------------|--------------------------------------------------|--------------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received         | LangChain Chat Trigger               | Receive user chat messages                        | External webhook                     | AI Agent                          |                                                                                                                                       |
| AI Agent                          | LangChain Agent                     | Orchestrate AI logic, tool usage & generate reply| When chat message received           | Google Gemini Chat Model, HTTP Request for Webhook Notification | Sticky Note3: Use Bright Data MCP Search Engine assistant tools. AI Agent returns response to chat and webhook. Source: https://github.com/luminati-io/brightdata-mcp |
| Google Gemini Chat Model          | LangChain LM Chat Google Gemini     | Generate AI chat responses using Google Gemini  | AI Agent (as language model)         | AI Agent                          | Sticky Note4: Google Gemini used to interpret queries and initiate MCP client calls.                                                  |
| Simple Memory                    | LangChain Memory Buffer Window      | Maintain conversational context                   | AI Agent                            | AI Agent                          |                                                                                                                                       |
| When clicking ‘Test workflow’     | Manual Trigger                      | Manual workflow start for testing                 | Manual user action                   | MCP Client list all tools for Bright Data | Sticky Note1: Bright Data Google Search                                                                                              |
| MCP Client list all tools for Bright Data | MCP Client (STDIO) node             | List MCP tools from Bright Data MCP server        | When clicking ‘Test workflow’        | Set search query                  | Sticky Note1: Bright Data Google Search                                                                                              |
| MCP Client Bright Data Search Tool | MCP Client                        | Execute search tool dynamically selected          | Set search query                    | AI Agent                         | Sticky Note1: Bright Data Google Search                                                                                              |
| Set search query                  | Set node                           | Initialize static search query                      | MCP Client list all tools for Bright Data | MCP Client Bright Data Search Tool | Sticky Note1: Bright Data Google Search                                                                                              |
| MCP Client List all tools         | MCP Client Tool                   | Provide tools info to AI Agent                      | None                              | AI Agent                         | Sticky Note3: Use Bright Data MCP Search Engine assistant tools. AI Agent returns response to chat and webhook. Source: https://github.com/luminati-io/brightdata-mcp |
| Google Search Engine for Bright Data | MCP Client Tool                   | Execute Google search engine queries                | AI Agent                          | AI Agent                         | Sticky Note: ## Bright Data Search Engines                                                                                          |
| Bing Search Engine for Bright Data | MCP Client Tool                   | Execute Bing search engine queries                  | AI Agent                          | AI Agent                         | Sticky Note: ## Bright Data Search Engines                                                                                          |
| Yandex Search Engine for Bright Data | MCP Client Tool                   | Execute Yandex search engine queries                | AI Agent                          | AI Agent                         | Sticky Note: ## Bright Data Search Engines                                                                                          |
| HTTP Request for Webhook Notification | LangChain Tool HTTP Request     | Send chat responses to external webhook            | AI Agent                          | None                            |                                                                                                                                       |
| Sticky Note                      | Sticky Note                       | Documentation and contextual notes                  | None                              | None                            | ## Bright Data Search Engines                                                                                                        |
| Sticky Note1                     | Sticky Note                       | Documentation                                      | None                              | None                            | ## Bright Data Google Search                                                                                                        |
| Sticky Note2                     | Sticky Note                       | Disclaimer                                        | None                              | None                            | ## Disclaimer This template is only available on n8n self-hosted as it's making use of the community node for MCP Client.             |
| Sticky Note3                     | Sticky Note                       | Documentation                                      | None                              | None                            | ## Note Use Bright Data MCP Search Engine assistant tools... Source - https://github.com/luminati-io/brightdata-mcp                 |
| Sticky Note4                     | Sticky Note                       | Documentation                                      | None                              | None                            | ## LLM Usage Google Gemini is employed by the AI agent to understand and interpret user queries...                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Add **LangChain Chat Trigger** node.  
   - Configure webhook (auto-generated or custom).  
   - No special parameters. This node listens to incoming chat messages.

2. **Create AI Agent Node**  
   - Add **LangChain Agent** node.  
   - Set parameter:  
     - System message:  
       ```
       You are a helpful assistant.

       Use MCP Search Engine assistant tools for Bright Data for Google, Bing or Yandex Search. 

       Important: Return the response to Chat and also perform the webhook notification of responses.

       Use the relevant tool in the order of execution.
       ```  
   - Connect input from Chat Trigger node.  
   - No credentials needed here, but the node will call MCP tools and LM node.

3. **Create Google Gemini Chat Model Node**  
   - Add **LangChain LM Chat Google Gemini** node.  
   - Set modelName to `models/gemini-2.0-flash-exp`.  
   - Configure credentials with your Google Gemini (PaLM) API key.  
   - Connect as AI language model input to AI Agent node.

4. **Create Simple Memory Node**  
   - Add **LangChain Memory Buffer Window** node.  
   - Default parameters suffice.  
   - Connect memory output to AI Agent (ai_memory).

5. **Create MCP Client List All Tools Node**  
   - Add **MCP Client (STDIO)** node.  
   - Select MCP Client API credentials linked to Bright Data MCP Server.  
   - Operation: list all tools (default, no parameters).  
   - This node provides available MCP tools dynamically.

6. **Create Set Search Query Node**  
   - Add **Set** node.  
   - Create a field `search_query` of type string with default value `"Bright Data"` (can be changed later to dynamic queries).  
   - Connect input from MCP Client list all tools node.

7. **Create MCP Client Bright Data Search Tool Node**  
   - Add **MCP Client** node.  
   - Configure:  
     - Operation: execute tool.  
     - ToolName: expression: `={{ $('MCP Client list all tools for Bright Data').item.json.tools[0].name }}` (uses first tool dynamically).  
     - ToolParameters: JSON with keys:  
       ```json
       {
         "query": "={{ $json.search_query }}",
         "engine": "google"
       }
       ```  
   - Use same MCP Client API credentials.  
   - Connect input from Set search query node.

8. **Create MCP Client Tool Nodes for Search Engines**  
   - Add three **MCP Client Tool** nodes for Google, Bing, and Yandex search engines respectively.  
   - For each:  
     - ToolName: `search_engine`.  
     - Operation: `executeTool`.  
     - ToolParameters: JSON with keys:  
       ```json
       {
         "query": "={{ $json.chatInput }}",
         "engine": "google" // or "bing", "yandex"
       }
       ```  
     - Credentials: MCP Client API as above.  
   - Connect these nodes as AI tools to the AI Agent node.

9. **Create HTTP Request for Webhook Notification Node**  
   - Add **LangChain Tool HTTP Request** node.  
   - Configure:  
     - URL: set to your webhook endpoint (e.g., https://webhook.site/your-id).  
     - Method: POST.  
     - Send Body: true.  
     - Parameters Body: add a parameter named `chat_response` with the AI response content.  
   - Connect as AI tool output to AI Agent node to send notifications.

10. **Create Manual Trigger Node for Testing**  
    - Add **Manual Trigger** node.  
    - Connect output to MCP Client list all tools for Bright Data node.  
    - Useful for manual execution and validation.

11. **Add Sticky Notes**  
    - Add notes for documentation and clarity at appropriate positions describing Bright Data search engines, disclaimer about MCP Client community nodes and self-hosted usage, and AI agent logic.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This template is only available on n8n self-hosted as it uses the community node for MCP Client.                                                                           | Disclaimer section in workflow description.                                                                 |
| Knowledge of Model Context Protocol (MCP) is essential; see https://www.anthropic.com/news/model-context-protocol for details.                                           | Workflow pre-condition.                                                                                       |
| Setup instructions for MCP Server and n8n-nodes-mcp available at https://www.youtube.com/watch?v=NUb73ErUCsA and https://www.npmjs.com/package/@brightdata/mcp           | Setup section for local environment.                                                                         |
| Bright Data MCP Search Engine assistant tools source code and usage details: https://github.com/luminati-io/brightdata-mcp                                                 | Sticky Note3 source link.                                                                                     |
| Google Gemini AI Studio API key required; visit https://aistudio.google.com/                                                                                            | Credential setup instructions.                                                                                |
| Use Bright Data Web Unlocker API for proxy scraping; ensure API tokens are correctly added in MCP Client credentials in n8n.                                              | Authentication details for MCP Client connection.                                                            |
| You can extend outputs to Slack, Discord, WhatsApp, or CRM systems and log conversations in databases like PostgreSQL or MongoDB for audit or training purposes.            | Customization suggestions.                                                                                     |

---

This documentation provides a detailed, stepwise, and comprehensive understanding of the workflow, enabling both human operators and AI automation agents to reproduce, troubleshoot, and extend the solution effectively.