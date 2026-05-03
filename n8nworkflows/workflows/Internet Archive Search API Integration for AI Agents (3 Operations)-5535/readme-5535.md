Internet Archive Search API Integration for AI Agents (3 Operations)

https://n8nworkflows.xyz/workflows/internet-archive-search-api-integration-for-ai-agents--3-operations--5535


# Internet Archive Search API Integration for AI Agents (3 Operations)

---

### 1. Workflow Overview

This workflow, named **"Search Services MCP Server"**, serves as an integration layer between AI agents and the Internet Archive's Search API. It exposes three distinct API endpoints as tools accessible via a Modular Chat Platform (MCP) trigger, allowing AI agents to perform search-related operations on Internet Archive data.

The workflow is logically divided into the following blocks:

- **1.1 Setup & Documentation**: Provides usage instructions, configuration notes, and an overview for users and administrators.
- **1.2 MCP Trigger Input Reception**: Handles incoming requests from AI agents through a dedicated MCP webhook endpoint.
- **1.3 API Request Processing**: Implements three HTTP request nodes, each corresponding to a different Internet Archive Search API endpoint:
  - Fields endpoint: fetches searchable fields metadata.
  - Organic search endpoint: returns relevance-based search results.
  - Scrape endpoint: enables scrolling cursor-based scraping of search results.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation

**Overview:**  
This block contains sticky notes that provide comprehensive setup instructions, workflow description, usage notes, and customization tips for end users and developers.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Search (Sticky Note)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Documentation for importing, activating, and connecting the workflow; outlines no authentication needed; highlights auto-populated AI parameters; customization options; and help resources including Discord and official docs.  
  - Input/Output: None (informational only)  
  - Edge Cases: None (static content)  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Describes the workflow’s purpose and architecture, the MCP trigger usage, API endpoints exposed, and operational summary.  
  - Input/Output: None  
  - Edge Cases: None  

- **Search**  
  - Type: Sticky Note  
  - Role: Acts as a simple header label for the following search-related nodes.  
  - Input/Output: None  
  - Edge Cases: None  

---

#### 1.2 MCP Trigger Input Reception

**Overview:**  
This block hosts the MCP Trigger node that acts as the entry point for AI agent requests, exposing a webhook endpoint. It allows the workflow to receive and process incoming tool requests dynamically.

**Nodes Involved:**  
- Search Services MCP Server (MCP Trigger)

**Node Details:**

- **Search Services MCP Server**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Entry point that exposes a webhook at path `/search-services-mcp` to receive AI agent requests formatted as MCP tool calls.  
  - Configuration:  
    - Webhook path set explicitly to `search-services-mcp`  
  - Input: Incoming HTTP requests from AI agents invoking any of the three API operations.  
  - Output: Routes requests to connected HTTP Request nodes via the ai_tool interface.  
  - Version Requirements: Requires n8n version supporting MCP trigger nodes (Langchain integration).  
  - Edge Cases:  
    - Webhook URL misconfiguration may prevent AI agents from reaching the workflow.  
    - No authentication is set, so external exposure should be controlled via environment or network policies.  
  - Sub-workflow: None  

---

#### 1.3 API Request Processing

**Overview:**  
This block contains three HTTP Request Tool nodes, each configured to invoke a specific Internet Archive Search API endpoint. These nodes serve as the operational tools accessible via the MCP trigger.

**Nodes Involved:**  
- Fields that can be requested  
- Return relevance-based results from search queries  
- Scrape search results from Internet Archive, allowing a scro

**Node Details:**

- **Fields that can be requested**  
  - Type: `n8n-nodes-base.httpRequestTool`  
  - Role: Queries `https://api.archive.org/search/v1/fields` to retrieve metadata about searchable fields.  
  - Configuration:  
    - URL: `https://api.archive.org/search/v1/fields` (evaluated as expression but static)  
    - Authentication: HTTP Header Authentication with preset credentials (Test Header Auth Cred).  
    - No additional query parameters or body; uses GET method implicitly.  
  - Input: Receives MCP trigger calls requesting this endpoint.  
  - Output: Returns API JSON response maintaining original structure.  
  - Edge Cases:  
    - HTTP errors (e.g., 401 Unauthorized if credentials expire)  
    - Network timeouts or API downtime  
    - Invalid or missing header auth credential setup  
  - Version Requirements: HTTP Request Tool node version 4.2 or higher.  

- **Return relevance-based results from search queries**  
  - Type: `n8n-nodes-base.httpRequestTool`  
  - Role: Invokes `https://api.archive.org/search/v1/organic` to return relevance-ranked search results based on query parameters supplied by AI.  
  - Configuration:  
    - URL: `https://api.archive.org/search/v1/organic`  
    - Authentication: HTTP Header Authentication using same credential as above.  
    - Parameters are auto-populated via `$fromAI()` expressions, enabling dynamic query input from AI agent.  
  - Input: Triggered by MCP node when organic search operation is requested.  
  - Output: Forwards original API response back to AI agent.  
  - Edge Cases:  
    - Invalid query parameters may return API errors or empty results.  
    - Authentication or networking failures.  
  - Version Requirements: HTTP Request Tool node version 4.2+.  

- **Scrape search results from Internet Archive, allowing a scro**  
  - Type: `n8n-nodes-base.httpRequestTool`  
  - Role: Calls `https://api.archive.org/search/v1/scrape` to scrape search results, supporting cursor-based pagination (scrolling).  
  - Configuration:  
    - URL: `https://api.archive.org/search/v1/scrape`  
    - Authentication: HTTP Header Authentication (same credential).  
    - Accepts dynamic parameters from AI via `$fromAI()` expressions for cursor control and filtering.  
  - Input: Invoked by MCP trigger when scrape operation is requested.  
  - Output: Returns raw API response to AI agent.  
  - Edge Cases:  
    - Pagination cursor mismanagement leading to incomplete results or infinite loops.  
    - API rate limiting or errors.  
  - Version Requirements: HTTP Request Tool node version 4.2+.  

---

### 3. Summary Table

| Node Name                                             | Node Type                                 | Functional Role                          | Input Node(s)                | Output Node(s)                                      | Sticky Note                                                                                                                          |
|-------------------------------------------------------|-------------------------------------------|----------------------------------------|-----------------------------|----------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                                    | Sticky Note                              | Setup and usage documentation          | None                        | None                                               | Contains detailed setup and usage instructions; Discord help link; n8n docs link                                                      |
| Workflow Overview                                     | Sticky Note                              | Workflow purpose and API overview      | None                        | None                                               | Describes MCP trigger and available API operations                                                                                   |
| Search                                               | Sticky Note                              | Section header for search nodes        | None                        | None                                               |                                                                                                                                    |
| Search Services MCP Server                           | MCP Trigger                              | Entry point webhook for AI agent calls| None                        | Fields that can be requested, Return relevance-based results..., Scrape search results... |                                                                                                                                    |
| Fields that can be requested                         | HTTP Request Tool                        | Fetch searchable fields metadata       | Search Services MCP Server   | Back to MCP Trigger (ai_tool output)               | Uses HTTP Header Auth credential; calls /search/v1/fields endpoint                                                                    |
| Return relevance-based results from search queries   | HTTP Request Tool                        | Return relevance-ranked search results | Search Services MCP Server   | Back to MCP Trigger (ai_tool output)               | Uses HTTP Header Auth credential; calls /search/v1/organic endpoint                                                                   |
| Scrape search results from Internet Archive, allowing a scro | HTTP Request Tool                        | Scrape search results with cursoring   | Search Services MCP Server   | Back to MCP Trigger (ai_tool output)               | Uses HTTP Header Auth credential; calls /search/v1/scrape endpoint                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Add a Sticky Note node.  
   - Set content to detailed setup instructions including import steps, no authentication, enabling workflow, MCP URL retrieval, AI parameter auto-population via `$fromAI()`, customization tips, and help resources with Discord and n8n docs links.  
   - Set color to orange (color index 4).  
   - Position appropriately (e.g., left side).  

2. **Create Sticky Note: Workflow Overview**  
   - Add a Sticky Note node.  
   - Set content describing the workflow’s purpose as an MCP server for Internet Archive Search API.  
   - Include summary of MCP trigger usage, endpoint URLs, and operations available (fields, return, scrape).  
   - Adjust size to width 420 and height 920.  

3. **Create Sticky Note: Search**  
   - Add a Sticky Note node.  
   - Content: “## Search” (as a section header).  
   - Set color to green (color index 2).  

4. **Add MCP Trigger Node: Search Services MCP Server**  
   - Node type: `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Configure webhook path as `search-services-mcp`.  
   - No authentication needed.  
   - Position centrally for routing requests.  

5. **Add HTTP Request Tool Node: Fields that can be requested**  
   - Node type: `HTTP Request Tool`.  
   - Set URL to `https://api.archive.org/search/v1/fields`.  
   - Use HTTP Header Authentication credentials: create or select existing with necessary API key or token under “Test Header Auth Cred”.  
   - Leave method as GET.  
   - Provide description: “Fields that can be requested”.  

6. **Add HTTP Request Tool Node: Return relevance-based results from search queries**  
   - Node type: `HTTP Request Tool`.  
   - URL: `https://api.archive.org/search/v1/organic`.  
   - Use same HTTP Header Authentication credentials.  
   - Allow parameters to be dynamically set via `$fromAI()` expressions (this is handled automatically by the MCP interface).  
   - Description: “Return relevance-based results from search queries”.  

7. **Add HTTP Request Tool Node: Scrape search results from Internet Archive, allowing a scro**  
   - Node type: `HTTP Request Tool`.  
   - URL: `https://api.archive.org/search/v1/scrape`.  
   - Same HTTP Header Authentication credentials.  
   - Enable dynamic parameter input from AI agents (via `$fromAI()`).  
   - Description: “Scrape search results from Internet Archive, allowing a scrolling cursor”.  

8. **Connect Nodes**  
   - Connect the MCP Trigger node’s `ai_tool` output to each of the three HTTP Request Tool nodes’ inputs.  
   - This enables the MCP to route incoming tool requests to the appropriate HTTP node.  

9. **Credential Setup**  
   - Create HTTP Header Auth credentials containing required API key or token for Internet Archive API.  
   - Assign these credentials to each HTTP Request Tool node under authentication.  

10. **Activate Workflow**  
    - Enable the workflow.  
    - Copy the webhook URL for the MCP trigger for use in AI agent configuration.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| The workflow leverages n8n’s MCP trigger to expose Internet Archive Search APIs as AI-compatible tools.                                            | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                        |
| No authentication is implemented on the webhook endpoint; secure exposure recommended via network or firewall settings.                           | Setup Instructions sticky note                                                                                      |
| For integration support or custom automation requests, contact the author on Discord at https://discord.me/cfomodz                                | Setup Instructions sticky note                                                                                      |
| Parameters for HTTP Request nodes are automatically populated by AI agent using `$fromAI()` expressions, enabling flexible dynamic input.          | Workflow Overview sticky note                                                                                        |
| The workflow returns raw JSON responses from Internet Archive API without transformation, preserving API contract and response structure.           | Workflow Overview sticky note                                                                                        |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*