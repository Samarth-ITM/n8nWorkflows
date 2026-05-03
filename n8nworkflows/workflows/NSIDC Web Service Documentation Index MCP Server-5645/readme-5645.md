NSIDC Web Service Documentation Index MCP Server

https://n8nworkflows.xyz/workflows/nsidc-web-service-documentation-index-mcp-server-5645


# NSIDC Web Service Documentation Index MCP Server

### 1. Workflow Overview

This workflow, titled **NSIDC Web Service Documentation Index MCP Server**, serves as a bridge between AI agents and the National Snow and Ice Data Center (NSIDC) Web Service Documentation Index API. It exposes four distinct API endpoints through a Managed Connector Protocol (MCP) trigger, enabling AI agents to query and retrieve data about snow and ice datasets and metadata programmatically.

**Target Use Cases:**
- AI agents requiring dynamic access to NSIDC dataset search facets, document search results, search engine metadata, and search term suggestions.
- Developers integrating NSIDC data services into applications with minimal manual parameter input, leveraging AI-driven auto-populated parameters.
- Providing a single MCP-compatible endpoint that handles routing and query forwarding to NSIDC’s API with native response formatting.

**Logical Blocks:**

- **1.1 Setup & Documentation**: Contains sticky notes with setup instructions and workflow overview to guide users.
- **1.2 MCP Trigger Block**: The entry point that listens for incoming AI agent requests.
- **1.3 API Operation Blocks**: Four HTTP Request Tool nodes, each representing one NSIDC API endpoint:
  - Retrieve Search Facets
  - Search Documents
  - Get Search Engine Description
  - Suggest Search Terms

Each API node receives parameters automatically from AI input expressions and returns the original API responses to the AI agent.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation Block

**Overview:**  
Provides users with setup instructions, usage notes, customization tips, and a workflow overview describing purpose, architecture, and available API operations.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Swagger Docs (Sticky Note)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Configuration: Detailed multi-section instructions covering import, authentication (none required), activation, MCP URL usage, AI parameter auto-population, customization, and support links (Discord, n8n docs).  
  - Input/Output: None (informational only)  
  - Edge Cases: None applicable  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Configuration: Summarizes workflow purpose, explains the MCP trigger and HTTP Request nodes, describes AI expression usage, and lists four available API operations.  
  - Input/Output: None  
  - Edge Cases: None  

- **Swagger Docs**  
  - Type: Sticky Note  
  - Configuration: Simple label for the API operation nodes section.  
  - Input/Output: None  
  - Edge Cases: None  

#### 1.2 MCP Trigger Block

**Overview:**  
Serves as the single webhook endpoint for AI agents to send requests. MCP trigger routes requests to appropriate HTTP Request Tool nodes based on requested API operation.

**Nodes Involved:**  
- NSIDC Web Service Documentation Index MCP Server (MCP Trigger)

**Node Details:**

- **NSIDC Web Service Documentation Index MCP Server**  
  - Type: MCP Trigger (part of Langchain n8n nodes)  
  - Configuration:  
    - Path: `nsidc-web-service-documentation-index-mcp`  
    - Webhook ID fixed for uniqueness  
  - Input: Incoming HTTP requests from AI agents  
  - Output: Connected to all 4 HTTP Request Tool nodes as AI tools  
  - Version: Requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger` node type  
  - Edge Cases:  
    - Webhook URL must be correctly set and publicly accessible.  
    - Requests missing required parameters may cause downstream nodes to fail.  
    - MCP Trigger node errors on malformed requests or invalid paths.  
  - Sub-workflow: None  

#### 1.3 API Operation Blocks

Each block corresponds to one NSIDC API endpoint implemented via an HTTP Request Tool node. They receive parameters via `$fromAI()` expressions (auto-populated by AI input) and forward requests to NSIDC's HTTP API.

---

##### 1.3.1 Retrieve Search Facets

**Overview:**  
Fetches facet information related to search queries, supporting faceted search filters for dataset exploration.

**Nodes Involved:**  
- Retrieve Search Facets (HTTP Request Tool)

**Node Details:**

- Type: HTTP Request Tool  
- URL: `http://nsidc.org/api/dataset/2/Facets`  
- Method: GET (default for HTTP Request Tool)  
- Query Parameters:  
  - `searchTerms` (optional string, URL-encoded keywords)  
  - `count` (optional number, default 25)  
  - `startIndex` (optional number, default 1)  
  - `spatial` (optional string, default full world bounding box)  
  - `sortKeys` (optional string, default `score,,desc`)  
  - `startDate`, `endDate` (optional date strings)  
  - `facetFilters` (optional JSON string, URL-encoded)  
  - `source` (optional string, default `NSIDC`)  
- Parameters are dynamically populated by AI via `$fromAI()` expressions with descriptions for AI context.  
- Input: Connected from MCP Trigger as AI tool  
- Output: Returns raw API JSON response to MCP trigger for forwarding to AI agent  
- Edge Cases:  
  - Invalid or malformed parameters may cause API errors.  
  - Network or timeout issues reaching NSIDC server.  
  - AI expression failures if expected parameters missing or incorrectly typed.  
- Version: Compatible with n8n HTTP Request Tool v4.2+  

---

##### 1.3.2 Search Documents

**Overview:**  
Performs document search using OpenSearch 1.1 specification, returning relevant dataset metadata matching search criteria.

**Nodes Involved:**  
- Search Documents (HTTP Request Tool)

**Node Details:**

- Type: HTTP Request Tool  
- URL: `http://nsidc.org/api/dataset/2/OpenSearch`  
- Query Parameters: Same as Retrieve Search Facets, mapped via `$fromAI()` expressions, with defaults and descriptions.  
- Input/Output: Same as previous node, connected from MCP Trigger  
- Edge Cases and Version: Same as Retrieve Search Facets  

---

##### 1.3.3 Get Search Engine Description

**Overview:**  
Retrieves metadata describing NSIDC’s search engine interface.

**Nodes Involved:**  
- Get Search Engine Description (HTTP Request Tool)

**Node Details:**

- Type: HTTP Request Tool  
- URL: `http://nsidc.org/api/dataset/2/OpenSearchDescription`  
- No query parameters (static endpoint)  
- Input: MCP Trigger  
- Output: Raw API response forwarded to AI agent  
- Edge Cases:  
  - Network connectivity issues  
  - Unexpected API response formats  
- Version: HTTP Request Tool v4.2+  

---

##### 1.3.4 Suggest Search Terms

**Overview:**  
Provides search term suggestions based on partial queries to aid user input completion.

**Nodes Involved:**  
- Suggest Search Terms (HTTP Request Tool)

**Node Details:**

- Type: HTTP Request Tool  
- URL: `http://nsidc.org/api/dataset/2/suggest`  
- Query Parameters:  
  - `q` (required string, partial search term, minimum 2 characters)  
  - `source` (required string, default `NSIDC`)  
- Parameters populated by `$fromAI()` expressions  
- Input/Output: From MCP Trigger, returns suggestion list to AI agent  
- Edge Cases:  
  - Missing or too short `q` parameter may cause API errors  
  - Network failures or malformed requests  
- Version: HTTP Request Tool v4.2+  

---

### 3. Summary Table

| Node Name                                | Node Type                     | Functional Role                                | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                                                                                                                                                                                              |
|-----------------------------------------|-------------------------------|------------------------------------------------|--------------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                      | Sticky Note                   | Setup & usage instructions                      | None                                 | None                                 | ⚙️ Setup Instructions: Import workflow, no auth required, activate workflow, get MCP URL, connect AI agent, auto-populated parameters, 4 API endpoints, response structure preserved, customization & error handling tips, Discord & n8n docs links.                         |
| Workflow Overview                      | Sticky Note                   | Describes workflow purpose and API operations | None                                 | None                                 | 🛠️ NSIDC Web Service Documentation Index MCP Server overview with 4 operations, AI auto-parameterization, MCP trigger explanation, and API endpoints listed.                                                                                                              |
| Swagger Docs                          | Sticky Note                   | Section label for API nodes                     | None                                 | None                                 | ## Swagger Docs                                                                                                                                                                                                                                                           |
| NSIDC Web Service Documentation Index MCP Server | MCP Trigger (Langchain)       | Entry webhook for AI agent requests             | None                                 | Retrieve Search Facets, Search Documents, Get Search Engine Description, Suggest Search Terms |                                                                                                                                                                                                                                                                           |
| Retrieve Search Facets                 | HTTP Request Tool             | Fetches facet information for search queries   | NSIDC MCP Trigger                    | MCP Trigger (response to AI agent)  | View facet information; parameters auto-populated by AI with defaults; supports OpenSearch 1.1 and OpenSearch-Geo 1.0; returns original API response.                                                                                                                    |
| Search Documents                      | HTTP Request Tool             | Performs OpenSearch document searches           | NSIDC MCP Trigger                    | MCP Trigger (response to AI agent)  | Search documents with OpenSearch 1.1 spec; AI populates parameters with context; returns raw search results.                                                                                                                                                              |
| Get Search Engine Description          | HTTP Request Tool             | Retrieves search engine metadata                 | NSIDC MCP Trigger                    | MCP Trigger (response to AI agent)  | Describes NSIDC's data search engine web interface; no parameters needed; returns API description JSON.                                                                                                                                                                    |
| Suggest Search Terms                   | HTTP Request Tool             | Suggests search terms based on partial query    | NSIDC MCP Trigger                    | MCP Trigger (response to AI agent)  | Suggests search terms; requires partial query; AI auto-populates parameters; returns suggestion list; source parameter defaults to NSIDC.                                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note Node: "Setup Instructions"**  
   - Add a Sticky Note node.  
   - Paste the setup instructions content describing import, auth, activation, MCP URL, AI parameter usage, endpoint count, response format, customization, and help links.  
   - Position roughly at [-1380, -240].

2. **Create Sticky Note Node: "Workflow Overview"**  
   - Add another Sticky Note node.  
   - Paste workflow overview content summarizing purpose, MCP trigger role, AI expressions, and API operations.  
   - Position at [-1120, -240].

3. **Create Sticky Note Node: "Swagger Docs"**  
   - Add Sticky Note node with content "## Swagger Docs".  
   - Position at [-660, -120].

4. **Add MCP Trigger Node: "NSIDC Web Service Documentation Index MCP Server"**  
   - Select node type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Set the `path` parameter to `nsidc-web-service-documentation-index-mcp`.  
   - Position at [-620, -240].  
   - No authentication credentials required.  
   - Confirm webhook URL is active and publicly accessible.

5. **Add HTTP Request Tool Node: "Retrieve Search Facets"**  
   - Set URL to `http://nsidc.org/api/dataset/2/Facets`.  
   - Method: GET.  
   - Enable "Send Query" parameters.  
   - Add query parameters with names and values using `$fromAI()` expressions:  
     - `searchTerms` (string, optional)  
     - `count` (number, optional, default 25)  
     - `startIndex` (number, optional, default 1)  
     - `spatial` (string, optional, default `-180.0,-90.0,180.0,90.0`)  
     - `sortKeys` (string, optional, default `score,,desc`)  
     - `startDate`, `endDate` (string, optional)  
     - `facetFilters` (string, optional)  
     - `source` (string, optional, default `NSIDC`)  
   - Add tool description as in the original node.  
   - Position at [-520, -60].  
   - Connect MCP Trigger node’s `ai_tool` output to this node.

6. **Add HTTP Request Tool Node: "Search Documents"**  
   - URL: `http://nsidc.org/api/dataset/2/OpenSearch`.  
   - Method: GET.  
   - Same query parameters with `$fromAI()` expressions and defaults as "Retrieve Search Facets".  
   - Position at [-320, -60].  
   - Connect MCP Trigger node’s `ai_tool` output to this node.

7. **Add HTTP Request Tool Node: "Get Search Engine Description"**  
   - URL: `http://nsidc.org/api/dataset/2/OpenSearchDescription`.  
   - Method: GET.  
   - No query parameters.  
   - Position at [-120, -60].  
   - Connect MCP Trigger node’s `ai_tool` output to this node.

8. **Add HTTP Request Tool Node: "Suggest Search Terms"**  
   - URL: `http://nsidc.org/api/dataset/2/suggest`.  
   - Method: GET.  
   - Query parameters:  
     - `q` (string, required) using `$fromAI('q', ...)` expression.  
     - `source` (string, optional, default `NSIDC`)  
   - Position at [80, -60].  
   - Connect MCP Trigger node’s `ai_tool` output to this node.

9. **Verify Connections:**  
   - Ensure MCP Trigger node outputs connect to all four HTTP Request Tool nodes as AI tools.

10. **Activate the Workflow:**  
    - Save and activate the workflow.  
    - Copy the webhook URL from the MCP Trigger node.  
    - Use this URL to configure your AI agent to send requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automation requests, contact via Discord: https://discord.me/cfomodz                                                                                                                                      | Support and community help                                                                                                                                                       |
| Official n8n documentation for MCP Nodes and Langchain integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                                                             | Reference documentation for MCP Trigger setup                                                                                                                                    |
| Parameters are auto-populated by AI agents using `$fromAI()` expressions that include detailed descriptions and default values, facilitating seamless AI-driven parameter injection.                                                            | AI integration detail                                                                                                                                                            |
| The workflow does not require any authentication credentials as the NSIDC API endpoints are publicly accessible.                                                                                                                               | Simplifies deployment and usage                                                                                                                                                   |
| Responses maintain the original structure of the NSIDC API, allowing clients to parse and handle data natively without transformation.                                                                                                          | Preserves data integrity                                                                                                                                                          |
| The four HTTP Request nodes correspond directly to four API operations defined in the NSIDC Swagger documentation, ensuring coverage of key dataset search functionalities.                                                                      | Ensures comprehensive API coverage                                                                                                                                               |

---

**Disclaimer:** The text provided derives solely from an automated workflow created with n8n, an integration and automation tool. It strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.