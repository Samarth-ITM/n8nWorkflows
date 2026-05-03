Create a Domains-Index API Server with Full Operation Access for AI Agents

https://n8nworkflows.xyz/workflows/create-a-domains-index-api-server-with-full-operation-access-for-ai-agents-5495


# Create a Domains-Index API Server with Full Operation Access for AI Agents

### 1. Workflow Overview

This workflow implements a **Domains-Index MCP Server** that exposes 14 fully operational API endpoints for AI agents to interact with a Domains-Index database. It acts as a middleware server converting the Domains-Index API into an MCP-compatible interface, enabling AI agents to query domain-related data and info statistics programmatically.

**Target Use Cases:**  
- AI agents requesting domain data and statistics via a unified MCP trigger endpoint  
- Automated domain research and monitoring using multiple API endpoints  
- Integration of domain indexing data into AI workflows without manual parameter entry  

**Logical Blocks:**  
- **1.1 MCP Server Trigger:** Single entry point exposing the webhook URL for AI agents.  
- **1.2 Domains Operations (9 endpoints):** HTTP request nodes that handle various domain-related queries such as search, TLD records, updates, and downloads.  
- **1.3 Info Operations (5 endpoints):** HTTP request nodes providing API info and statistical summaries at global and zone-specific levels.  
- **1.4 Documentation & Notes:** Sticky notes providing instructions, overview, and usage guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger

- **Overview:**  
  Acts as the webhook entry point for AI agents to send requests. It listens on a specific path and routes incoming MCP requests to appropriate nodes internally.

- **Nodes Involved:**  
  - Domains-Index MCP Server

- **Node Details:**  
  - **Domains-Index MCP Server**  
    - Type: `mcpTrigger` (LangChain MCP Node)  
    - Configuration: Webhook path set to `domains-index-mcp`  
    - Role: Serves as the API server endpoint; receives requests and triggers connected AI tool nodes  
    - Inputs: HTTP requests from AI agents  
    - Outputs: Routed to 14 downstream HTTP request nodes covering all operations  
    - Version: v1 MCP node, requires n8n environment supporting LangChain MCP nodes  
    - Edge cases: Webhook auth is disabled (no authentication), so public access; must secure network if sensitive  
    - Notes: This node centralizes all API calls and uses AI expressions to auto-populate parameters  

#### 1.2 Domains Operations (9 endpoints)

- **Overview:**  
  Contains HTTP Request Tool nodes that call specific `/v1/domains` API endpoints. These nodes handle domain searches, TLD record retrievals, dataset downloads, and incremental domain updates.

- **Nodes Involved:**  
  - Domains Database Search  
  - Get TLD records  
  - Download Whole Dataset for TLD  
  - Domains Search for TLD  
  - Get added domains, latest if date not specified  
  - Download added domains, latest if date not specifi  
  - Get deleted domains, latest if date not specified  
  - Download deleted domains, latest if date not speci  
  - List of updates  

- **Node Details:**  

  - **Domains Database Search**  
    - HTTP Request node with GET method to `/v1/domains/search`  
    - Query params include `api_key`, `date`, `page`, `limit`, and domain filters such as `domain`, `zone`, `country`, `isDead`, DNS record types (A, NS, CNAME, MX, TXT)  
    - Uses `$fromAI()` expressions to auto-fill parameters from AI input  
    - Authentication: HTTP Header Authorization requiring API key  
    - Edge cases: Missing or invalid API key, malformed parameters, network timeouts  

  - **Get TLD records**  
    - GET `/v1/domains/tld/{{zone_id}}`  
    - Requires `zone_id` path parameter  
    - Query parameters similar to above, including pagination and filters  
    - Auth: same HTTP header API key  
    - Edge cases: Invalid or missing `zone_id`, parameter errors  

  - **Download Whole Dataset for TLD**  
    - GET `/v1/domains/tld/{{zone_id}}/download`  
    - Downloads entire dataset for given zone  
    - Query params: `api_key`, `date` (optional)  
    - Edge cases: Large payloads may cause timeouts; ensure API key valid  

  - **Domains Search for TLD**  
    - GET `/v1/domains/tld/{{zone_id}}/search`  
    - Supports pagination, filters as above  
    - Edge cases: Same as Get TLD records  

  - **Get added domains, latest if date not specified**  
    - GET `/v1/domains/updates/added`  
    - Query params: `api_key`, `date`, pagination  
    - Provides domains added since a date or latest if none specified  
    - Edge cases: Date format errors, pagination issues  

  - **Download added domains, latest if date not specifi**  
    - GET `/v1/domains/updates/added/download`  
    - Downloads dataset of added domains  
    - Query params: `api_key`, `date`  
    - Edge cases: Large downloads, invalid date  

  - **Get deleted domains, latest if date not specified**  
    - GET `/v1/domains/updates/deleted`  
    - Query params: `api_key`, `date`, pagination  
    - Edge cases: Same as added domains node  

  - **Download deleted domains, latest if date not speci**  
    - GET `/v1/domains/updates/deleted/download`  
    - Query params: `api_key`, `date`  
    - Edge cases: Large payloads, invalid date  

  - **List of updates**  
    - GET `/v1/domains/updates/list`  
    - Query params: `api_key` only  
    - Edge cases: API key validation  

#### 1.3 Info Operations (5 endpoints)

- **Overview:**  
  Provides API meta-information and statistical data at overall and zone-specific levels.

- **Nodes Involved:**  
  - Get info  
  - Returns overall stagtistics  
  - Returns statistics for specific zone  
  - Returns overall Tld info  
  - Returns statistics for specific zone 1  

- **Node Details:**  

  - **Get info**  
    - GET `/v1/info/api`  
    - Query param: `api_key` optional  
    - Returns API information  
    - Edge cases: Missing API key may limit info returned  

  - **Returns overall stagtistics**  
    - GET `/v1/info/stat/`  
    - Query params: `page`, `limit` for pagination  
    - Returns overall statistical summaries  
    - Edge cases: Pagination parameters invalid or missing  

  - **Returns statistics for specific zone**  
    - GET `/v1/info/stat/{{zone}}`  
    - Requires `zone` path param and pagination params  
    - Edge cases: Invalid zone, pagination errors  

  - **Returns overall Tld info**  
    - GET `/v1/info/tld/`  
    - No query params needed  
    - Returns top-level domain info  
    - Edge cases: Network issues  

  - **Returns statistics for specific zone 1**  
    - GET `/v1/info/tld/{{zone}}`  
    - Requires `zone` path param, pagination params  
    - Edge cases: Invalid zone name, pagination errors  

#### 1.4 Documentation & Notes

- **Overview:**  
  Provides detailed sticky notes outlining workflow purpose, setup instructions, usage notes, and operation descriptions.

- **Nodes Involved:**  
  - Workflow Overview (large sticky note)  
  - Sticky Note (Domains header)  
  - Sticky Note2 (Info header)

- **Node Details:**  
  - Sticky notes contain markdown-formatted instructions and metadata  
  - Positioned for clear visual grouping of domains and info blocks  
  - No direct input/output connections  

---

### 3. Summary Table

| Node Name                                      | Node Type                  | Functional Role                     | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                                              |
|-----------------------------------------------|----------------------------|-----------------------------------|---------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview                             | stickyNote                 | Documentation overview             |                           |                            | ## 🛠️ Domains-Index MCP Server ✅ 14 operations ... [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)  |
| Domains-Index MCP Server                      | mcpTrigger                 | MCP webhook server entry point    |                           | All HTTP Request Tool nodes |                                                                                                                                          |
| Sticky Note                                  | stickyNote                 | Domains section header             |                           |                            | ## Domains                                                                                                                               |
| Domains Database Search                       | httpRequestTool            | Domains search API call            | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Get TLD records                              | httpRequestTool            | Retrieve TLD domain records        | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Download Whole Dataset for TLD               | httpRequestTool            | Download full dataset for TLD      | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Domains Search for TLD                       | httpRequestTool            | Search domains under TLD           | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Get added domains, latest if date not specified | httpRequestTool            | List newly added domains           | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Download added domains, latest if date not specifi | httpRequestTool            | Download added domains dataset     | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Get deleted domains, latest if date not specified | httpRequestTool            | List deleted domains               | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Download deleted domains, latest if date not speci | httpRequestTool            | Download deleted domains dataset   | Domains-Index MCP Server  |                            |                                                                                                                                          |
| List of updates                             | httpRequestTool            | List domain update types           | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Sticky Note2                                 | stickyNote                 | Info section header                |                           |                            | ## Info                                                                                                                                  |
| Get info                                    | httpRequestTool            | Retrieve API info                  | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Returns overall stagtistics                  | httpRequestTool            | Get overall statistics             | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Returns statistics for specific zone         | httpRequestTool            | Get zone-specific statistics       | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Returns overall Tld info                     | httpRequestTool            | Get overall TLD info               | Domains-Index MCP Server  |                            |                                                                                                                                          |
| Returns statistics for specific zone 1       | httpRequestTool            | Get TLD zone-specific statistics   | Domains-Index MCP Server  |                            |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node:**  
   - Add **Domains-Index MCP Server** node of type `mcpTrigger` (LangChain)  
   - Configure webhook path as `domains-index-mcp`  
   - No authentication required  
   - This node serves as single entry point for all AI agent API requests  

2. **Add Sticky Note for Workflow Overview:**  
   - Create a `stickyNote` node named `Workflow Overview`  
   - Paste workflow description and setup instructions as markdown content  

3. **Create Domains Section Sticky Note:**  
   - Add a `stickyNote` named `Sticky Note` with content `## Domains`  

4. **Add Domains HTTP Request Tool Nodes:**  
   For each domain-related API endpoint, create an `httpRequestTool` node with the following settings:  
   - **Authentication:** HTTP Header with an `api_key` parameter populated from AI input (`$fromAI('api_key', 'API key', 'string')`)  
   - **Method:** GET (default)  
   - **URL and Parameters:** Use the path and query parameters as per each node below, with `$fromAI()` expressions for dynamic AI input:  

   a. **Domains Database Search:**  
      - URL: `/v1/domains/search`  
      - Query params include `date`, `page`, `limit`, domain filters, DNS record filters  

   b. **Get TLD records:**  
      - URL: `/v1/domains/tld/{{ $fromAI('zone_id', 'Zone Id', 'string') }}`  
      - Query params: same as above minus zone filter  

   c. **Download Whole Dataset for TLD:**  
      - URL: `/v1/domains/tld/{{ $fromAI('zone_id', 'Zone Id', 'string') }}/download`  
      - Query params: `api_key`, `date`  

   d. **Domains Search for TLD:**  
      - URL: `/v1/domains/tld/{{ $fromAI('zone_id', 'Zone Id', 'string') }}/search`  
      - Query params: domain filters and pagination  

   e. **Get added domains:**  
      - URL: `/v1/domains/updates/added`  
      - Query params: `api_key`, `date`, `page`, `limit`  

   f. **Download added domains:**  
      - URL: `/v1/domains/updates/added/download`  
      - Query params: `api_key`, `date`  

   g. **Get deleted domains:**  
      - URL: `/v1/domains/updates/deleted`  
      - Query params: `api_key`, `date`, `page`, `limit`  

   h. **Download deleted domains:**  
      - URL: `/v1/domains/updates/deleted/download`  
      - Query params: `api_key`, `date`  

   i. **List of updates:**  
      - URL: `/v1/domains/updates/list`  
      - Query param: `api_key`  

5. **Connect all Domain HTTP Request nodes as AI tools to MCP Trigger node:**  
   - Connect each node’s `ai_tool` input to the MCP Trigger node’s output  
   - This allows the MCP Trigger to route requests to the correct operation  

6. **Create Info Section Sticky Note:**  
   - Add a `stickyNote` named `Sticky Note2` with content `## Info`  

7. **Add Info HTTP Request Tool Nodes:**  
   Similarly, create HTTP Request Tool nodes for info endpoints with these settings:  

   a. **Get info:**  
      - URL: `/v1/info/api`  
      - Query param: `api_key`  

   b. **Returns overall stagtistics:**  
      - URL: `/v1/info/stat/`  
      - Query params: `page`, `limit`  

   c. **Returns statistics for specific zone:**  
      - URL: `/v1/info/stat/{{ $fromAI('zone', 'Zone', 'string') }}`  
      - Query params: `page`, `limit`  

   d. **Returns overall Tld info:**  
      - URL: `/v1/info/tld/`  
      - No params  

   e. **Returns statistics for specific zone 1:**  
      - URL: `/v1/info/tld/{{ $fromAI('zone', 'Zone', 'string') }}`  
      - Query params: `page`, `limit`  

8. **Connect all Info HTTP Request nodes as AI tools to MCP Trigger node:**  
   - Link their `ai_tool` inputs to the MCP Trigger output  

9. **Set Credentials:**  
   - Create a generic HTTP header credential with API key header for all HTTP Request nodes  
   - Ensure the API key is passed dynamically from AI input using expressions  

10. **Activate the workflow:**  
    - Save and activate to expose the webhook  
    - Copy the webhook URL from the MCP Trigger node for AI agent integration  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow converts the Domains-Index API into an MCP-compatible interface for AI agents, supporting 14 operations related to domains and info data. No authentication is required at the MCP trigger level.                   | Workflow Overview Sticky Note                                                                                                 |
| Parameters for API calls are auto-populated by AI using `$fromAI()` placeholders, enabling dynamic input without manual configuration.                                                                                          | Workflow Overview Sticky Note                                                                                                 |
| For MCP integration details and customization, see [n8n documentation on MCP nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) and contact on [Discord](https://discord.me/cfomodz). | Workflow Overview Sticky Note                                                                                                 |
| HTTP Request Tool nodes use generic HTTP header authentication with an API key that must be provided by the AI agent at runtime.                                                                                                | Implied credential setup for HTTP Request nodes                                                                               |
| Large dataset downloads may cause timeouts; consider adding timeout and error handling nodes if needed.                                                                                                                        | General recommendation                                                                                                        |
| The workflow has no built-in authentication on the webhook endpoint; secure network access accordingly.                                                                                                                       | General security note                                                                                                         |

---

**Disclaimer:**  
The text provided derives exclusively from an automated n8n workflow. It adheres strictly to applicable content policies and contains no illegal or protected elements. All data handled is lawful and public.