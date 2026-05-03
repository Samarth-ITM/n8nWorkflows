Connect Color Name API with AI Agents via MCP Server

https://n8nworkflows.xyz/workflows/connect-color-name-api-with-ai-agents-via-mcp-server-5489


# Connect Color Name API with AI Agents via MCP Server

### 1. Workflow Overview

This workflow, titled **"Connect Color Name API with AI Agents via MCP Server"**, is designed to expose the Color Name API (https://api.color.pizza) through an MCP (Model-Controller-Proxy) server interface. It allows AI agents to query color data in various ways by interacting with a dedicated webhook endpoint. The workflow transforms the Color Name API into a set of callable tools within an AI agent ecosystem, enabling automated and AI-driven color information retrieval.

**Target Use Cases:**  
- AI agents querying color names, lists, swatches, or color data by hex or name.  
- Systems requiring color information integration without direct API management.  
- Developers looking to extend or customize color data handling via n8n workflows.

**Logical Blocks:**  
- **1.1 Workflow Introduction:** Sticky note overview explaining the workflow purpose and setup.  
- **1.2 MCP Server Trigger:** Entry point for AI agent requests via a webhook.  
- **1.3 Color Name API Endpoints:** Four HTTP Request nodes, each implementing one API operation:  
  - Get all colors of a list (with optional filters)  
  - Get all available lists  
  - Get colors by name  
  - Generate a color swatch  

Each HTTP request node is configured to dynamically receive parameters from AI agent input using `$fromAI()` expressions and return raw API responses.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Workflow Introduction

**Overview:**  
This block contains a comprehensive sticky note summarizing the workflow’s purpose, operations, usage, and setup instructions. It serves as an internal documentation node for users and maintainers.

**Nodes Involved:**  
- Workflow Overview (Sticky Note)

**Node Details:**  
- **Type:** Sticky Note  
- **Role:** Documentation and guidance for users importing or modifying the workflow.  
- **Configuration:** Contains markdown-formatted content describing the API, MCP integration, endpoints, setup steps, usage notes, and customization tips.  
- **Input/Output:** No data input or output connections; purely informational.  
- **Edge Cases:** None.  
- **Version Requirements:** None.

---

#### Block 1.2: MCP Server Trigger

**Overview:**  
Acts as the webhook trigger that opens this workflow to external AI agent calls via MCP protocol. It listens for incoming requests targeting the path "color-name-api-mcp".

**Nodes Involved:**  
- Color Name API MCP Server (MCP Trigger)

**Node Details:**  
- **Type:** MCP Trigger (from LangChain package for n8n)  
- **Role:** Entry point for AI agents to invoke workflow operations.  
- **Configuration:**  
  - Webhook path: `color-name-api-mcp`  
  - No authentication required (default MCP trigger setup)  
- **Input:** HTTP requests from AI agents routed through MCP.  
- **Output:** Passes data to connected HTTP Request nodes that perform API calls.  
- **Version Requirements:** Requires n8n version supporting MCP nodes and LangChain integration (n8n v1.95.3 or later recommended).  
- **Edge Cases:**  
  - Invalid or malformed MCP requests could cause failure.  
  - Network or webhook availability issues impact accessibility.  
  - No direct authentication; ensure secure MCP endpoint exposure.

---

#### Block 1.3: Color Name API Endpoints

This block contains four HTTP request nodes exposing separate API functions. Each node uses `$fromAI()` expressions to dynamically fill query parameters from the AI agent’s input.

---

##### Node: Get all colors of the default color name list

- **Type:** HTTP Request Tool  
- **Role:** Retrieves colors from the specified color name list with optional filters.  
- **Configuration:**  
  - URL: `https://api.color.pizza/v1//` (note the double slash is intentional for endpoint root)  
  - Query Parameters (all optional):  
    - `list`: Name of the color list to use (string)  
    - `values`: Hex values of colors to retrieve, without '#' (string)  
    - `noduplicates`: Boolean to allow or disallow duplicate names  
  - Authentication: Generic HTTP Header authentication configured (likely empty as no auth required by API)  
  - Parameters dynamically populated using `$fromAI()` placeholders for seamless AI integration.  
- **Input:** Receives data from MCP Trigger node.  
- **Output:** Returns the API JSON response to the MCP client.  
- **Edge Cases:**  
  - Missing or malformed hex values or list names could cause API errors.  
  - Boolean parsing for `noduplicates` must be validated.  
  - API downtime or network issues may cause failures.

---

##### Node: Get all colors of the default color name list 1 (Lists)

- **Type:** HTTP Request Tool  
- **Role:** Retrieves all available color name lists from the API.  
- **Configuration:**  
  - URL: `https://api.color.pizza/v1//lists/`  
  - No query parameters.  
  - Authentication: Generic HTTP Header auth (likely unused).  
- **Input:** Connected to MCP Trigger.  
- **Output:** Returns list of color name lists as JSON.  
- **Edge Cases:**  
  - API endpoint availability.  
  - No user parameters required, so minimal error risk.

---

##### Node: Get all colors of the default color name list 2 (Names)

- **Type:** HTTP Request Tool  
- **Role:** Retrieves colors based on color name and optional list.  
- **Configuration:**  
  - URL: `https://api.color.pizza/v1//names/`  
  - Query Parameters:  
    - `name`: Color name to search (string, minimum 3 characters)  
    - `list`: Optional color list name (string)  
  - Parameters sourced from `$fromAI()` expressions.  
  - Authentication: Generic HTTP Header auth.  
- **Input:** From the MCP Trigger node.  
- **Output:** Returns matched color information JSON.  
- **Edge Cases:**  
  - Name parameter must meet minimum length to avoid API errors.  
  - Optional list name must be valid if provided.  
  - API or network failures possible.

---

##### Node: Generate a color swatch for any color

- **Type:** HTTP Request Tool  
- **Role:** Generates a color swatch image or data for a given color hex and optional name.  
- **Configuration:**  
  - URL: `https://api.color.pizza/v1//swatch/`  
  - Query Parameters:  
    - `color`: Hex value of the color (string, without '#')  
    - `name`: Optional color name (string)  
  - Parameters populated with `$fromAI()` expressions.  
  - Authentication: Generic HTTP Header auth.  
- **Input:** Connected to MCP Trigger node.  
- **Output:** Returns swatch data as provided by the API.  
- **Edge Cases:**  
  - Color hex must be valid and correctly formatted.  
  - Optional name is flexible.  
  - API or network errors possible.  

---

### 3. Summary Table

| Node Name                                  | Node Type                  | Functional Role                        | Input Node(s)             | Output Node(s)    | Sticky Note                                                                                                                        |
|--------------------------------------------|----------------------------|--------------------------------------|---------------------------|-------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview                          | Sticky Note                | Documentation and setup instructions | None                      | None              | ## 🛠️ Color Name API MCP Server ✅ 4 operations ... (full markdown documentation)                                                  |
| Color Name API MCP Server                  | MCP Trigger                | Entry point webhook for AI agent requests | None                    | Get all colors..., Get all colors... 1, Get all colors... 2, Generate a color swatch | See Workflow Overview for MCP trigger information                                                                                  |
| Sticky Note                               | Sticky Note                | (Empty placeholder, no content)      | None                      | None              | (Empty)                                                                                                                           |
| Get all colors of the default color name list | HTTP Request Tool          | Retrieves colors with filters from a list | Color Name API MCP Server | None              | Parameters auto-filled by AI via $fromAI()                                                                                         |
| Sticky Note2                              | Sticky Note                | Section label for "Lists"             | None                      | None              | ## Lists                                                                                                                          |
| Get all colors of the default color name list 1 | HTTP Request Tool          | Retrieves all available color lists   | Color Name API MCP Server | None              | No parameters needed                                                                                                              |
| Sticky Note3                              | Sticky Note                | Section label for "Names"             | None                      | None              | ## Names                                                                                                                          |
| Get all colors of the default color name list 2 | HTTP Request Tool          | Retrieves colors by name               | Color Name API MCP Server | None              | Requires name parameter ≥ 3 characters                                                                                            |
| Sticky Note4                              | Sticky Note                | Section label for "Swatch"            | None                      | None              | ## Swatch                                                                                                                         |
| Generate a color swatch for any color      | HTTP Request Tool          | Generates swatch data for given color | Color Name API MCP Server | None              | Requires valid hex color (without '#'), name optional                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Sticky Note node ("Workflow Overview"):**  
   - Paste the provided markdown content describing the workflow, usage, and setup.  
   - Position it prominently for reference.

3. **Add an MCP Trigger node ("Color Name API MCP Server"):**  
   - Select the MCP Trigger node type from LangChain nodes.  
   - Set the webhook path to `color-name-api-mcp`.  
   - No authentication configured.  
   - This node will serve as the entry point for AI agent requests.

4. **Create 4 HTTP Request Tool nodes for API endpoints:**

   - **Node 1: "Get all colors of the default color name list"**  
     - Set HTTP Method: GET  
     - URL: `https://api.color.pizza/v1//`  
     - Enable Query Parameters:  
       - `list` → Expression: `$fromAI('list', 'The name of the color name list to use', 'string')`  
       - `values` → Expression: `$fromAI('values', 'The hex values of the colors to retrieve without \'#\'', 'string')`  
       - `noduplicates` → Expression: `$fromAI('noduplicates', 'Allow duplicate names or not', 'boolean')`  
     - Authentication: Generic HTTP Header Auth (empty or default, as no auth needed)  

   - **Node 2: "Get all colors of the default color name list 1" (Lists)**  
     - HTTP Method: GET  
     - URL: `https://api.color.pizza/v1//lists/`  
     - No query parameters.  
     - Authentication: Generic HTTP Header Auth.

   - **Node 3: "Get all colors of the default color name list 2" (Names)**  
     - HTTP Method: GET  
     - URL: `https://api.color.pizza/v1//names/`  
     - Query Parameters:  
       - `name` → Expression: `$fromAI('name', 'The name of the color to retrieve (min 3 characters)', 'string')`  
       - `list` → Expression: `$fromAI('list', 'The name of the color name list to use', 'string')`  
     - Authentication: Generic HTTP Header Auth.

   - **Node 4: "Generate a color swatch for any color"**  
     - HTTP Method: GET  
     - URL: `https://api.color.pizza/v1//swatch/`  
     - Query Parameters:  
       - `color` → Expression: `$fromAI('color', 'The hex value of the color to retrieve without \'#\'', 'string')`  
       - `name` → Expression: `$fromAI('name', 'The name of the color', 'string')`  
     - Authentication: Generic HTTP Header Auth.

5. **Connect each HTTP Request node’s input to the MCP Trigger node's output:**  
   - This setup allows all API calls to be triggered by a single MCP webhook endpoint.

6. **Add Sticky Notes as section labels for clarity:**  
   - For example, add Sticky Notes titled "Lists", "Names", "Swatch" positioned near corresponding HTTP Request nodes.

7. **Credential Setup:**  
   - Create a generic HTTP Header authentication credential in n8n (even if empty) and assign to each HTTP Request node.  
   - No token or username/password is required for the Color Name API.

8. **Activate the workflow:**  
   - Ensure the workflow is enabled to start accepting MCP requests.

9. **Testing:**  
   - Use the MCP webhook URL exposed by the MCP Trigger node to send test requests from your AI agent or HTTP client.  
   - Confirm parameters are correctly passed and responses match API expectations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow converts the Color Name API into an MCP-compatible interface for AI agents, enabling seamless AI integration without authentication.                                                                                                                                           | Workflow Overview Sticky Note                                                                                                  |
| Parameters for API calls are dynamically populated from AI agent requests using the `$fromAI()` expression placeholders.                                                                                                                                                                      | Workflow Overview Sticky Note                                                                                                  |
| For detailed information on MCP nodes and AI integrations, refer to the official n8n MCP documentation.                                                                                                                                                                                     | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                  |
| For MCP integration guidance and community support, join the n8n Discord at https://discord.me/cfomodz                                                                                                                                                                                      | Workflow Overview Sticky Note                                                                                                  |
| The Color Name API requires no authentication but expects parameters such as color hex values without the '#' symbol and minimum length constraints for name parameters.                                                                                                                     | Node parameter descriptions                                                                                                    |
| The workflow can be customized with additional error handling, logging, or data transformation nodes as needed to suit use cases.                                                                                                                                                            | Workflow Overview Sticky Note                                                                                                  |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.