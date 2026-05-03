Expose Chomp Food Database to AI Agents with MCP Server (4 food operations)

https://n8nworkflows.xyz/workflows/expose-chomp-food-database-to-ai-agents-with-mcp-server--4-food-operations--5551


# Expose Chomp Food Database to AI Agents with MCP Server (4 food operations)

### 1. Workflow Overview

This workflow, titled **"Chomp Food Database API Documentation MCP Server"**, serves as an MCP (Machine Chat Protocol) server to expose the Chomp Food Database API to AI agents. It enables AI-driven queries to access four distinct food-related operations via HTTP requests, automatically handling parameter population with AI expressions and returning API responses in their native structure.

**Target Use Cases:**  
- AI agents requiring food data through an accessible MCP server interface  
- Automated querying of branded food items by barcode or name  
- Searching branded food items with multiple filters  
- Searching generic food ingredients  

**Logical Blocks:**

- **1.1 Setup and Documentation**: Provides user instructions and workflow overview in sticky notes.
- **1.2 MCP Server Endpoint**: The MCP trigger node listens for AI agent requests.
- **1.3 Food Operations**: Four HTTP request nodes implement the API calls corresponding to supported food operations:
  - Get Branded Food by Barcode
  - Get Branded Food by Name
  - Search Branded Food Items with filters
  - Search Food Ingredients

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Documentation

**Overview:**  
This block provides comprehensive setup instructions, usage notes, and an overview of the workflow, helping users understand and deploy the MCP server effectively.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)

**Node Details:**  

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Document setup steps, usage notes, customization tips, and help contact.  
  - Content Highlights:  
    - Import and activate the workflow  
    - Configure API key credentials (query parameter named `api_key`)  
    - Copy the webhook URL from MCP trigger to connect AI agents  
    - Parameters are auto-populated via `$fromAI()` expressions  
    - Contains links to Discord support and n8n documentation  
  - Input/Output: None (documentation node)  
  - Edge cases: None  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Explain the workflow’s purpose, API requirements, and available operations.  
  - Content Highlights:  
    - Requires API key subscription from https://chompthis.com/api  
    - MCP Trigger serves as the AI endpoint  
    - 4 food operations exposed as tools  
    - Response structure preserved  
  - Input/Output: None (documentation node)  
  - Edge cases: None  

---

#### 1.2 MCP Server Endpoint

**Overview:**  
This node acts as the main entry point for AI agent requests, exposing a webhook path to receive and route requests to the respective food operation nodes.

**Nodes Involved:**  
- Chomp Food Database API Documentation MCP Server (MCP Trigger)

**Node Details:**  

- **Chomp Food Database API Documentation MCP Server**  
  - Type: MCP Trigger node (from n8n-nodes-langchain)  
  - Role: Accept AI agent requests at webhook path `chomp-food-database-api-documentation-mcp` and route to tools  
  - Configuration:  
    - Webhook path: `chomp-food-database-api-documentation-mcp`  
    - MCP trigger automatically parses AI requests and passes tool calls downstream  
  - Input: External AI agent requests over HTTP  
  - Output: Connects to each of the 4 HTTP Request Tool nodes via `ai_tool` connection  
  - Edge cases:  
    - Webhook unreachable or misconfigured path results in failed requests  
    - Authentication errors if API key is missing or invalid (handled downstream)  
    - Timeout or network errors in subsequent API calls  
  - Version-specific: Requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger` node (Langchain MCP integration)  

---

#### 1.3 Food Operations

**Overview:**  
This block contains four HTTP request nodes, each wrapping a specific Chomp Food Database API endpoint. Each node receives parameters auto-populated by AI expressions (`$fromAI()`), sends a query-authenticated request, and returns the exact API response to the MCP trigger.

**Nodes Involved:**  
- Get Branded Food by Barcode  
- Get Branded Food by Name  
- Search Branded Food Items  
- Search Food Ingredients

**Node Details:**  

- **Get Branded Food by Barcode**  
  - Type: HTTP Request Tool  
  - Role: Retrieves a branded food item by UPC/EAN barcode  
  - URL: `https://chompthis.com/api/v2/food/branded/barcode.php`  
  - Authentication: API key passed as query parameter `api_key` (configured in credentials)  
  - Query Parameter:  
    - `code` (string) - barcode value populated by AI expression `$fromAI('code', ...)`  
  - Input: MCP Trigger (AI agent request)  
  - Output: Response data to MCP trigger  
  - Edge cases:  
    - Missing or invalid `code` parameter  
    - Barcode not found (empty or error response)  
    - API key authentication failure  
    - Network or timeout errors  

- **Get Branded Food by Name**  
  - Type: HTTP Request Tool  
  - Role: Retrieves branded food items by a general name keyword with pagination  
  - URL: `https://chompthis.com/api/v2/food/branded/name.php`  
  - Authentication: API key in query parameter  
  - Query Parameters:  
    - `name` (string, required) - general food name keyword  
    - `limit` (number, optional) - max records returned, default 10  
    - `page` (number, optional) - pagination page, default 1  
  - Input: MCP Trigger  
  - Output: Response data to MCP trigger  
  - Edge cases:  
    - Missing `name` parameter  
    - Pagination parameters invalid or out of range  
    - API key errors  
    - No results found  

- **Search Branded Food Items**  
  - Type: HTTP Request Tool  
  - Role: Searches branded food items with multiple optional filters  
  - URL: `https://chompthis.com/api/v2/food/branded/search.php`  
  - Authentication: API key in query parameter  
  - Query Parameters (many optional filters, examples):  
    - `allergen`, `brand`, `category`, `country`, `diet`, `ingredient`, `keyword`, `mineral`, `nutrient`, `palm_oil`, `trace`, `vitamin`  
    - `limit` (number, default 10)  
    - `page` (number, default 1)  
  - Important: Some parameters cannot be used alone and require at least one other filter  
  - Input: MCP Trigger  
  - Output: Response data to MCP trigger  
  - Edge cases:  
    - Using restricted parameters without required companion parameters  
    - Invalid values or empty filters  
    - API rate limit or key errors  

- **Search Food Ingredients**  
  - Type: HTTP Request Tool  
  - Role: Searches raw/generic food ingredient items by single or comma-separated list  
  - URL: `https://chompthis.com/api/v2/food/ingredient/search.php`  
  - Authentication: API key in query parameter  
  - Query Parameters:  
    - `find` (string, required) - ingredient(s) to find; max 10 ingredients if comma-separated  
    - `limit` (number, default 1)  
  - Input: MCP Trigger  
  - Output: Response data to MCP trigger  
  - Edge cases:  
    - Exceeding max 10 ingredients per request (requires multiple calls)  
    - Missing `find` parameter  
    - API key or network errors  

---

### 3. Summary Table

| Node Name                           | Node Type                      | Functional Role                               | Input Node(s)                           | Output Node(s)                        | Sticky Note                                                                                                                               |
|-----------------------------------|--------------------------------|-----------------------------------------------|----------------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                | Sticky Note                   | Provides setup, usage, and customization info | None                                   | None                                | ⚙️ Setup Instructions: Import workflow, configure API key, activate workflow, copy MCP URL, connect AI agent, usage notes, customization tips, support links |
| Workflow Overview                | Sticky Note                   | Explains workflow purpose and API operations  | None                                   | None                                | 🛠️ Chomp Food Database API Documentation MCP Server: API key required, MCP trigger as endpoint, 4 operations available                    |
| Chomp Food Database API Documentation MCP Server | MCP Trigger                   | AI agent webhook entrypoint                     | External AI agent requests             | Get Branded Food by Barcode, Get Branded Food by Name, Search Branded Food Items, Search Food Ingredients |                                                                                                                                            |
| Get Branded Food by Barcode       | HTTP Request Tool             | Get branded food item by barcode                | MCP Trigger                           | MCP Trigger                        |                                                                                                                                            |
| Get Branded Food by Name          | HTTP Request Tool             | Get branded food items by name with pagination | MCP Trigger                           | MCP Trigger                        |                                                                                                                                            |
| Search Branded Food Items         | HTTP Request Tool             | Search branded food items with filters          | MCP Trigger                           | MCP Trigger                        |                                                                                                                                            |
| Search Food Ingredients           | HTTP Request Tool             | Search raw/generic food ingredients             | MCP Trigger                           | MCP Trigger                        |                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Type: Sticky Note  
   - Content: Include detailed setup steps: import workflow, configure API key (query param named `api_key`), activate workflow, copy webhook URL, connect AI agent, usage notes, customization, and support links.  
   - Position appropriately (e.g., left side).

2. **Create Sticky Note: Workflow Overview**  
   - Type: Sticky Note  
   - Content: Describe workflow purpose, API subscription link (https://chompthis.com/api), MCP trigger role, 4 available food operations, and response handling.  
   - Position near Setup Instructions.

3. **Create MCP Trigger Node**  
   - Name: `Chomp Food Database API Documentation MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Webhook Path: `chomp-food-database-api-documentation-mcp`  
   - Position: center-left  
   - Purpose: Listen for AI agent requests and route to tools.

4. **Create HTTP Request Tool Node: Get Branded Food by Barcode**  
   - Name: `Get Branded Food by Barcode`  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://chompthis.com/api/v2/food/branded/barcode.php`  
     - Send Query: true  
     - Authentication: Generic Credential (API key as query parameter named `api_key`)  
     - Query Parameters:  
       - `code` set to expression: `{{$fromAI('code', '#### UPC/EAN barcode **Example** > ```&code=0842234000988```', 'string')}}`  
   - Position: below MCP Trigger node

5. **Create HTTP Request Tool Node: Get Branded Food by Name**  
   - Name: `Get Branded Food by Name`  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://chompthis.com/api/v2/food/branded/name.php`  
     - Send Query: true  
     - Authentication: Generic Credential with API key  
     - Query Parameters:  
       - `name`: `{{$fromAI('name', '#### Search for branded food items using ...', 'string')}}`  
       - `limit`: `{{$fromAI('limit', '#### Set max records ... default 10', 'number')}}`  
       - `page`: `{{$fromAI('page', '#### Pagination page ... default 1', 'number')}}`  
   - Position: to the right of previous node

6. **Create HTTP Request Tool Node: Search Branded Food Items**  
   - Name: `Search Branded Food Items`  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://chompthis.com/api/v2/food/branded/search.php`  
     - Send Query: true  
     - Authentication: Generic Credential with API key  
     - Query Parameters: Include all optional filters (`allergen`, `brand`, `category`, `country`, `diet`, `ingredient`, `keyword`, `mineral`, `nutrient`, `palm_oil`, `trace`, `vitamin`, `limit`, `page`), each set with appropriate `$fromAI()` expressions as in original  
   - Position: right of previous node

7. **Create HTTP Request Tool Node: Search Food Ingredients**  
   - Name: `Search Food Ingredients`  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://chompthis.com/api/v2/food/ingredient/search.php`  
     - Send Query: true  
     - Authentication: Generic Credential with API key  
     - Query Parameters:  
       - `find`: `{{$fromAI('find', 'Search database for ingredients ...', 'string')}}`  
       - `limit`: `{{$fromAI('limit', 'Set max results ... default 1', 'number')}}`  
   - Position: right of previous node

8. **Connect Nodes**  
   - From MCP Trigger output `ai_tool` to each HTTP Request Tool node input.

9. **Configure Credentials**  
   - Create or configure an API Key credential in n8n:  
     - Type: API Key  
     - Location: Query Parameter  
     - Parameter name: `api_key`  
     - Value: Your Chomp API key

10. **Activate Workflow**  
    - Save and activate workflow for live AI agent integration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Discord support for integration guidance and custom automations                                                                                                                                                              | https://discord.me/cfomodz                                                                              |
| Official n8n documentation on MCP nodes and Langchain integration                                                                                                                                                            | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/             |
| API key subscription and documentation for Chomp Food Database API                                                                                                                                                           | https://chompthis.com/api                                                                              |
| Parameters are auto-populated by AI expressions using `$fromAI()` syntax, enabling dynamic requests                                                                                                                       | —                                                                                                       |
| Some search parameters (e.g., allergen, country, diet, keyword, nutrient, trace) require at least one accompanying filter parameter to function correctly.                                                                  | Important for API usage constraints                                                                     |
| Comma-separated ingredient search parameter limited to 10 ingredients per API call; multiple calls required for larger sets.                                                                                               | Limits on Search Food Ingredients node                                                                  |

---

**Disclaimer:**  
The provided content is exclusively generated from an automated n8n workflow. All data handled is legal and publicly accessible. The workflow adheres strictly to content policies and contains no illegal or protected elements.