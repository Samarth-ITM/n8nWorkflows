Search and Retrieve eBay Product Data with Catalog API for AI Agents

https://n8nworkflows.xyz/workflows/search-and-retrieve-ebay-product-data-with-catalog-api-for-ai-agents-5566


# Search and Retrieve eBay Product Data with Catalog API for AI Agents

### 1. Workflow Overview

This workflow serves as a custom MCP (Multi-Channel Platform) server interface for AI agents to interact with eBay's Catalog API. It enables AI-driven queries to search for eBay product summaries and retrieve detailed product information using two main API endpoints. The workflow is designed to simplify integration between AI agents and eBay's Catalog API by automatically populating parameters via AI expressions and returning raw API responses.

Logical blocks within the workflow:

- **1.1 Setup and Documentation**: Contains instructions and overview notes for users to understand and configure the workflow.
- **1.2 MCP Server Trigger**: The entry point that listens for incoming AI agent requests and routes them accordingly.
- **1.3 Product Details Retrieval**: Handles requests to retrieve detailed information about a specific eBay catalog product using its ePID.
- **1.4 Product Summary Search**: Handles requests to search product summaries based on multiple criteria and filters.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Documentation

- **Overview**: This block provides users with setup instructions, workflow purpose, usage notes, and customization tips via sticky notes. It aids understanding and configuration.
- **Nodes Involved**:
  - Setup Instructions (Sticky Note)
  - Workflow Overview (Sticky Note)

- **Node Details**:

  - **Setup Instructions**
    - Type: Sticky Note (Documentation)
    - Role: Provides step-by-step setup guidance, explains parameter auto-population, and customization options.
    - Configuration: Contains markdown-formatted content with setup steps, usage notes, customization advice, and support contact info.
    - Inputs: None
    - Outputs: None
    - Edge Cases: None (informational only)

  - **Workflow Overview**
    - Type: Sticky Note (Documentation)
    - Role: Describes the Catalog API purpose, workflow architecture, and available operations.
    - Configuration: Markdown content explaining purpose, MCP trigger role, AI parameter usage, and listing of two API endpoints.
    - Inputs: None
    - Outputs: None
    - Edge Cases: None (informational only)

#### 1.2 MCP Server Trigger

- **Overview**: Acts as the server endpoint that AI agents call to invoke either product search or product details retrieval operations.
- **Nodes Involved**:
  - Catalog MCP Server (MCP Trigger node)

- **Node Details**:

  - **Catalog MCP Server**
    - Type: MCP Trigger (Langchain MCP Trigger)
    - Role: Listens on a webhook path (`catalog-mcp`) for incoming AI agent requests; triggers appropriate downstream nodes.
    - Configuration:
      - Webhook path: `catalog-mcp`
      - Webhook ID: Unique identifier (internal)
    - Inputs: External HTTP requests from AI agents
    - Outputs: Routes to two HTTP Request Tool nodes (Retrieve Product Details, Search Product Summaries)
    - Version Requirements: Requires n8n version supporting Langchain MCP Trigger node.
    - Edge Cases: Webhook connectivity issues, malformed requests, authentication failures on downstream nodes.
    - Sub-Workflow: None

#### 1.3 Product Details Retrieval

- **Overview**: This block fetches detailed catalog product information using the product’s ePID value.
- **Nodes Involved**:
  - Sticky Note (Product)
  - Retrieve Product Details (HTTP Request Tool)

- **Node Details**:

  - **Sticky Note (Product)**
    - Type: Sticky Note
    - Role: Labels and visually separates the product detail retrieval block.
    - Configuration: Simple title "Product"
    - Inputs: None
    - Outputs: None

  - **Retrieve Product Details**
    - Type: HTTP Request Tool
    - Role: Makes authenticated GET requests to eBay Catalog API endpoint `/product/{epid}` to retrieve product details.
    - Configuration:
      - URL constructed dynamically using basePath and ePID from AI input:  
        `https://api.ebay.com{basePath}/product/{{ $fromAI('epid', ...) }}`
      - Authentication: Generic HTTP Header Auth with OAuth2 credentials (configured outside workflow)
      - Header `X-EBAY-C-MARKETPLACE-ID` populated from AI input or defaults to identify marketplace (US, AU, CA, GB)
      - Sends headers, no body payload (GET request)
      - Tool description provides detailed parameter info for users
    - Expressions/Variables:
      - `$fromAI('epid', ...)` to dynamically obtain ePID for targeted product
      - `$fromAI('X-EBAY-C-MARKETPLACE-ID', ...)` for marketplace identification header
    - Inputs: Triggered by MCP Server node
    - Outputs: Sends raw product detail response back to the AI agent
    - Version Requirements: HTTP Request Tool version supporting dynamic expressions and OAuth2 header auth
    - Edge Cases:
      - Invalid or missing ePID parameter leads to API errors
      - Unauthorized errors if OAuth2 token invalid or expired
      - Marketplace ID header missing or invalid may cause API rejection
      - Network or timeout errors
    - Sub-Workflow: None

#### 1.4 Product Summary Search

- **Overview**: This block performs searches for product summaries using various filters like keywords, category IDs, aspects, GTIN, MPN, limits, etc.
- **Nodes Involved**:
  - Sticky Note2 (Product Summary)
  - Search Product Summaries (HTTP Request Tool)

- **Node Details**:

  - **Sticky Note2 (Product Summary)**
    - Type: Sticky Note
    - Role: Visually marks the product summary search block.
    - Configuration: Simple title "Product Summary"
    - Inputs: None
    - Outputs: None

  - **Search Product Summaries**
    - Type: HTTP Request Tool
    - Role: Sends authenticated GET requests to `/product_summary/search` endpoint of eBay Catalog API.
    - Configuration:
      - URL: `https://api.ebay.com{basePath}/product_summary/search`
      - Query parameters dynamically set with `$fromAI()` expressions for:
        - `aspect_filter`
        - `category_ids`
        - `fieldgroups`
        - `gtin`
        - `limit`
        - `mpn`
        - `offset`
        - `q` (keywords)
      - Header `X-EBAY-C-MARKETPLACE-ID` set similarly to product details node
      - Sends headers and query parameters in GET request
      - Tool description extensively documents each query parameter’s meaning and usage, including examples.
    - Expressions/Variables:
      - Each query parameter is populated via `$fromAI()` calls which allow AI agents to specify search criteria dynamically.
    - Inputs: Triggered by MCP Server node
    - Outputs: Returns raw product summary search results to the AI agent
    - Version Requirements: HTTP Request Tool supporting dynamic query parameters with expressions and OAuth2 header auth
    - Edge Cases:
      - Missing required combination of parameters (`q` or `category_ids` or `gtin` or `mpn`) results in API errors.
      - Malformed filter strings or invalid category IDs cause API rejections.
      - OAuth2 token issues or marketplace header errors.
      - Network or timeout failures.
    - Sub-Workflow: None

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                          | Input Node(s)       | Output Node(s)                 | Sticky Note                                                                                          |
|-------------------------|-----------------------|----------------------------------------|---------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Setup Instructions      | Sticky Note           | Setup and configuration instructions   | None                | None                          | Provides detailed setup steps, usage notes, customization tips, and support links.                  |
| Workflow Overview       | Sticky Note           | Documentation of workflow purpose       | None                | None                          | Describes Catalog API purpose, MCP server role, and available operations.                           |
| Catalog MCP Server      | MCP Trigger           | Entry point for AI agent requests       | External HTTP       | Retrieve Product Details, Search Product Summaries |                                                                                                    |
| Sticky Note             | Sticky Note           | Label for product details retrieval block | Catalog MCP Server  | Retrieve Product Details       | "Product"                                                                                          |
| Retrieve Product Details| HTTP Request Tool     | Fetch detailed product info by ePID     | Catalog MCP Server  | Returns API response to MCP    |                                                                                                    |
| Sticky Note2            | Sticky Note           | Label for product summary search block  | Catalog MCP Server  | Search Product Summaries       | "Product Summary"                                                                                   |
| Search Product Summaries| HTTP Request Tool     | Search for product summaries with filters | Catalog MCP Server  | Returns API response to MCP    |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note - Setup Instructions**
   - Type: Sticky Note
   - Position: (e.g., x: -1380, y: -240)
   - Content: Add detailed setup instructions as markdown:
     - Import workflow into n8n
     - Configure OAuth2 credentials
     - Activate workflow
     - Copy webhook URL from MCP trigger
     - Connect AI agents using the webhook URL
     - Usage notes on auto-populated parameters and endpoints
     - Customization and support links (discord, n8n docs)

2. **Create Sticky Note - Workflow Overview**
   - Type: Sticky Note
   - Position: (e.g., x: -1120, y: -240)
   - Content: Markdown text explaining:
     - Purpose of Catalog API and benefits of catalog-based listings
     - Workflow role as MCP server for AI agents
     - List of two API operations supported (product details, product summary search)

3. **Create MCP Trigger Node - Catalog MCP Server**
   - Type: MCP Trigger (Langchain MCP Trigger)
   - Position: (e.g., x: -620, y: -240)
   - Parameters:
     - Set webhook path to `catalog-mcp`
   - Save to generate webhook URL (used later in AI agent configurations)

4. **Create Sticky Note - Product**
   - Type: Sticky Note
   - Position: (e.g., x: -660, y: -100)
   - Content: "Product" (to label product details block)

5. **Create HTTP Request Tool Node - Retrieve Product Details**
   - Type: HTTP Request Tool
   - Position: (e.g., x: -520, y: -60)
   - Parameters:
     - HTTP Method: GET
     - URL: `https://api.ebay.com{basePath}/product/{{ $fromAI('epid', 'The ePID of the product...', 'string') }}`
     - Authentication: Generic HTTP Header Auth (using OAuth2 credentials configured separately)
     - Headers:
       - `X-EBAY-C-MARKETPLACE-ID`: `{{ $fromAI('X-EBAY-C-MARKETPLACE-ID', 'Marketplace ID...', 'string') }}`
     - Send Headers: Enabled
     - Send Query: Disabled
     - Tool Description: Add detailed description of endpoint and parameters for user reference

6. **Create Sticky Note - Product Summary**
   - Type: Sticky Note
   - Position: (e.g., x: -660, y: 140)
   - Content: "Product Summary"

7. **Create HTTP Request Tool Node - Search Product Summaries**
   - Type: HTTP Request Tool
   - Position: (e.g., x: -520, y: 180)
   - Parameters:
     - HTTP Method: GET
     - URL: `https://api.ebay.com{basePath}/product_summary/search`
     - Authentication: Generic HTTP Header Auth (OAuth2 credentials)
     - Headers:
       - `X-EBAY-C-MARKETPLACE-ID`: `{{ $fromAI('X-EBAY-C-MARKETPLACE-ID', 'Marketplace ID...', 'string') }}`
     - Query Parameters (all optional, dynamically populated via AI input):
       - `aspect_filter`: `{{ $fromAI('aspect_filter', 'Aspect filter details...', 'string') }}`
       - `category_ids`: `{{ $fromAI('category_ids', 'Category IDs...', 'string') }}`
       - `fieldgroups`: `{{ $fromAI('fieldgroups', 'Field groups...', 'string') }}`
       - `gtin`: `{{ $fromAI('gtin', 'GTIN...', 'string') }}`
       - `limit`: `{{ $fromAI('limit', 'Limit...', 'string') }}`
       - `mpn`: `{{ $fromAI('mpn', 'MPN...', 'string') }}`
       - `offset`: `{{ $fromAI('offset', 'Offset...', 'string') }}`
       - `q`: `{{ $fromAI('q', 'Keywords...', 'string') }}`
     - Send Headers: Enabled
     - Send Query: Enabled
     - Tool Description: Detailed description of endpoint, parameters, usage, and examples.

8. **Connect Nodes**
   - Connect the MCP Trigger node output to both HTTP Request Tool nodes (`Retrieve Product Details` and `Search Product Summaries`).
   - This allows the MCP server to route incoming requests to the proper API call based on AI agent input.

9. **Credential Configuration**
   - Create or import OAuth2 credentials for eBay API with appropriate scopes.
   - Assign these credentials to the HTTP Request Tool nodes under "Generic Auth" with HTTP Header Authentication.

10. **Activate Workflow**
    - Save and activate the workflow.
    - Copy the webhook URL generated by the MCP Trigger node.
    - Provide the webhook URL to AI agents for integration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automations, join the Discord community: https://discord.me/cfomodz                                                                                                                                                                   | Support Discord                                                                                             |
| Official n8n documentation for MCP nodes and Langchain integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                                                                                        | n8n Documentation                                                                                           |
| The workflow uses AI expressions `$fromAI()` to dynamically populate parameters from the AI agent's request payloads, enabling flexible and automated API calls without manual input.                                                                                      | AI Expression Usage                                                                                         |
| The workflow currently supports two main eBay Catalog API operations: retrieving detailed product info by ePID, and searching product summaries with multiple filters. Additional operations can be integrated by extending the workflow.                                   | Functional Scope                                                                                           |
| OAuth2 authentication must be configured properly for the eBay API, including token refresh handling. Failure to maintain valid tokens will cause authorization errors in API calls.                                                                                       | Authentication Reminder                                                                                     |
| Be aware of API rate limits and error handling strategies; consider adding error handling or logging nodes for production use to improve robustness.                                                                                                                     | Best Practices                                                                                             |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.