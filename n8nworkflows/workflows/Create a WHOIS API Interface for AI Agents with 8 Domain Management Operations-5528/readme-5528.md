Create a WHOIS API Interface for AI Agents with 8 Domain Management Operations

https://n8nworkflows.xyz/workflows/create-a-whois-api-interface-for-ai-agents-with-8-domain-management-operations-5528


# Create a WHOIS API Interface for AI Agents with 8 Domain Management Operations

### 1. Workflow Overview

This workflow, titled **"Create a WHOIS API Interface for AI Agents with 8 Domain Management Operations"**, serves as a comprehensive API server that exposes Bulk WHOIS and domain-related functionalities to AI agents via a Master Control Program (MCP) trigger. It is designed to facilitate domain management tasks including batch operations, database queries, and domain-specific checks by wrapping calls to a local Bulk WHOIS API service.

The workflow is logically divided into the following blocks:

- **1.1 Setup and Information**: Provides setup instructions and a workflow overview for users.
- **1.2 MCP Trigger (Entry Point)**: The main webhook endpoint that AI agents call to access any of the 8 domain management operations.
- **1.3 Batch Operations**: Four endpoints to manage batches of domain-related queries (get, create, delete, and get status).
- **1.4 Database Query**: A single endpoint to query the domain database.
- **1.5 Domain Operations**: Three endpoints to check domain availability, get domain rank, and perform WHOIS queries.

Each block contains HTTP request nodes communicating with a local API server at `http://localhost:5000`, authenticated via API key in the request header.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Information Block

- **Overview:**  
  This block provides essential workflow setup instructions and a concise overview of the workflow purpose, usage, and available operations. It is intended to guide users on importing, configuring, activating, and integrating the workflow.

- **Nodes Involved:**  
  - `Setup Instructions` (Sticky Note)  
  - `Workflow Overview` (Sticky Note)

- **Node Details:**  
  - **Setup Instructions (Sticky Note)**  
    - Type: Sticky Note (documentation)  
    - Contains detailed instructions on importing the workflow, setting up API key authentication (header key `X-API-KEY`), activating the workflow, retrieving the MCP webhook URL, and connecting AI agents.  
    - Notes also provide usage tips, customization ideas, and support links including Discord and official n8n documentation.  
    - No input or output connections; purely informational.  
    - No version-specific requirements or failure modes.  

  - **Workflow Overview (Sticky Note)**  
    - Type: Sticky Note (documentation)  
    - Summarizes the workflow’s architecture, the MCP trigger role, parameter auto-population with `$fromAI()` expressions, and the eight API endpoints grouped by function (Batch, Db, Domains).  
    - No input or output connections; purely informational.  

#### 1.2 MCP Trigger (Entry Point)

- **Overview:**  
  This node acts as the main webhook endpoint for AI agents, serving as the entry point to all domain management operations. It listens for incoming MCP requests and routes them to the appropriate HTTP request nodes.

- **Nodes Involved:**  
  - `Bulk WHOIS MCP Server` (MCP Trigger)

- **Node Details:**  
  - **Bulk WHOIS MCP Server**  
    - Type: MCP Trigger node (specialized webhook for Master Control Program integration)  
    - Configured with path `bulk-whois-mcp` which exposes a webhook URL for AI agents to send requests.  
    - No direct input nodes; triggers all subsequent HTTP request nodes via AI tool connections.  
    - Outputs are routed to each operation node via `ai_tool` connections for dynamic tool invocation based on AI agent input.  
    - Requires n8n version supporting MCP Trigger nodes.  
    - Potential failure modes include webhook connectivity, invalid requests, or authentication errors.  

#### 1.3 Batch Operations Block

- **Overview:**  
  This block implements four HTTP request nodes to manage batch operations on domain data: listing batches, creating a batch, deleting a batch, and retrieving batch status. These operations enable bulk processing of domain-related queries.

- **Nodes Involved:**  
  - `Sticky Note` (Batch header)  
  - `Get your batches` (HTTP Request Tool)  
  - `Create batch. Batch is then being processed until` (HTTP Request Tool)  
  - `Delete batch` (HTTP Request Tool)  
  - `Get batch` (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note (Batch header)**  
    - Type: Sticky Note  
    - Content: "## Batch" — visual separator/header for batch-related nodes.  

  - **Get your batches**  
    - Type: HTTP Request Tool  
    - Role: Retrieve the list of batches from the API endpoint `/batch` via GET request.  
    - Authentication: API key in header (`X-API-KEY`).  
    - No body or query parameters.  
    - Input: Triggered by MCP node.  
    - Output: Returns batch list to AI agent preserving original API response format.  
    - Potential failure: API connection failure, auth failure, or invalid response.  

  - **Create batch. Batch is then being processed until**  
    - Type: HTTP Request Tool  
    - Role: Create a new batch with domain list, operation type, and optional parameters via POST to `/batch`.  
    - Authentication: API key in header.  
    - Body parameters dynamically populated from AI inputs using expressions:  
      - `domains` (JSON array)  
      - `operation` (string)  
      - `options` (JSON object, optional)  
    - Input: Triggered by MCP node.  
    - Output: Returns batch creation confirmation and batch ID.  
    - Possible failure: malformed input, missing required fields, API timeout.  

  - **Delete batch**  
    - Type: HTTP Request Tool  
    - Role: Delete a specific batch by ID via DELETE request to `/batch/{id}`.  
    - Authentication: API key in header.  
    - Path parameter `id` dynamically from AI input.  
    - Input: Triggered by MCP node.  
    - Output: Returns deletion status.  
    - Failure modes: invalid batch ID, batch not found, auth error.  

  - **Get batch**  
    - Type: HTTP Request Tool  
    - Role: Retrieve details/status of a specific batch by ID via GET `/batch/{id}`.  
    - Authentication: API key in header.  
    - Path parameter `id` from AI input.  
    - Input: Triggered by MCP node.  
    - Output: Batch status and results.  
    - Possible failures: invalid ID, batch expired, API errors.  

#### 1.4 Database Query Block

- **Overview:**  
  This block contains a single HTTP request node to query the domain database for records matching criteria such as contact name, DNS, or domain.

- **Nodes Involved:**  
  - `Sticky Note2` (Db header)  
  - `Query domain database` (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note2 (Db header)**  
    - Type: Sticky Note  
    - Content: "## Db" — visual header for the database query block.  

  - **Query domain database**  
    - Type: HTTP Request Tool  
    - Role: Query the domain database via GET request to `/db` with query parameter `query`.  
    - Authentication: API key in header.  
    - Query parameter `query` is dynamically populated from AI input.  
    - Input: Triggered by MCP node.  
    - Output: Returns matching database records.  
    - Edge cases: empty or malformed queries, database timeout, auth failures.  

#### 1.5 Domain Operations Block

- **Overview:**  
  This block provides three HTTP request nodes to perform domain-specific checks: availability, rank (authority), and WHOIS queries.

- **Nodes Involved:**  
  - `Sticky Note3` (Domains header)  
  - `Check domain availability` (HTTP Request Tool)  
  - `Check domain rank (authority).` (HTTP Request Tool)  
  - `WHOIS query for a domain` (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note3 (Domains header)**  
    - Type: Sticky Note  
    - Content: "## Domains" — visual separator/header for domain operation nodes.  

  - **Check domain availability**  
    - Type: HTTP Request Tool  
    - Role: Check if a domain is available by GET to `/domains/{domain}/check`.  
    - Authentication: API key in header.  
    - Path parameter `domain` dynamically from AI input string.  
    - Input: Triggered by MCP node.  
    - Output: Returns availability status.  
    - Edge cases: invalid domain format, API downtime, auth failure.  

  - **Check domain rank (authority).**  
    - Type: HTTP Request Tool  
    - Role: Retrieve domain rank (authority) by GET `/domains/{domain}/rank`.  
    - Authentication: API key in header.  
    - Path parameter `domain` from AI input.  
    - Input: Triggered by MCP node.  
    - Output: Returns domain rank metrics.  
    - Possible failures: invalid domain, no ranking data, API errors.  

  - **WHOIS query for a domain**  
    - Type: HTTP Request Tool  
    - Role: Perform WHOIS lookup via GET `/domains/{domain}/whois` with optional query parameter `format`.  
    - Authentication: API key in header.  
    - Path parameter `domain` from AI input.  
    - Query parameter `format` optionally supplied by AI input (e.g., json, xml).  
    - Input: Triggered by MCP node.  
    - Output: WHOIS data in requested format or default.  
    - Edge cases: unsupported format, domain not found, API timeout.  

---

### 3. Summary Table

| Node Name                                   | Node Type                   | Functional Role                        | Input Node(s)             | Output Node(s)                           | Sticky Note                                                                                                                     |
|---------------------------------------------|-----------------------------|-------------------------------------|---------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions                          | Sticky Note                 | Setup and usage instructions         | None                      | None                                    | Instructions for workflow import, API key setup (X-API-KEY), activation, MCP URL, AI agent connection, customization, and support. |
| Workflow Overview                           | Sticky Note                 | Workflow purpose and operations overview | None                      | None                                    | Summary of workflow structure, MCP trigger role, 8 operations grouped by Batch, Db, Domains.                                   |
| Bulk WHOIS MCP Server                       | MCP Trigger                 | Main webhook entry point for AI agents | None                      | All HTTP request nodes                   |                                                                                                                               |
| Sticky Note (Batch header)                  | Sticky Note                 | Visual header for Batch operations   | None                      | None                                    | "## Batch"                                                                                                                     |
| Get your batches                            | HTTP Request Tool           | Retrieve list of batches             | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| Create batch. Batch is then being processed until | HTTP Request Tool           | Create new batch with domains and options | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| Delete batch                               | HTTP Request Tool           | Delete batch by ID                   | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| Get batch                                  | HTTP Request Tool           | Retrieve batch status by ID          | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| Sticky Note2 (Db header)                    | Sticky Note                 | Visual header for Db query           | None                      | None                                    | "## Db"                                                                                                                       |
| Query domain database                       | HTTP Request Tool           | Query domain database                | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| Sticky Note3 (Domains header)                | Sticky Note                 | Visual header for Domain operations  | None                      | None                                    | "## Domains"                                                                                                                  |
| Check domain availability                   | HTTP Request Tool           | Check if domain is available         | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| Check domain rank (authority).               | HTTP Request Tool           | Check domain authority rank          | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |
| WHOIS query for a domain                    | HTTP Request Tool           | Perform WHOIS query on domain        | Bulk WHOIS MCP Server      | Bulk WHOIS MCP Server                   |                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Add a Sticky Note node.  
   - Enter detailed setup instructions for workflow import, API key header config (`X-API-KEY`), activation, MCP URL retrieval, connecting AI agents, and customization tips.  
   - Set note color to distinguish (e.g., color 4).  

2. **Create Sticky Note: Workflow Overview**  
   - Add another Sticky Note node.  
   - Provide a summary of workflow purpose, MCP trigger role, and list of 8 API endpoints grouped by Batch, Db, Domains.  
   - Adjust width and height for readability.  

3. **Add MCP Trigger Node**  
   - Add an MCP Trigger node.  
   - Set webhook path to `bulk-whois-mcp`.  
   - No credentials needed on this node itself.  

4. **Batch Operations Setup**  
   - Add Sticky Note with "## Batch" as content.  
   - Add HTTP Request Tool node named `Get your batches`.  
     - Method: GET  
     - URL: `http://localhost:5000/batch`  
     - Authentication: Set to API Key Header with key name `X-API-KEY` (create or select credential accordingly).  
   - Add HTTP Request Tool node named `Create batch. Batch is then being processed until`.  
     - Method: POST  
     - URL: `http://localhost:5000/batch`  
     - Authentication: API Key Header.  
     - Body parameters (JSON):  
       - `domains`: use expression `$fromAI('domains', 'Domains', 'json')`  
       - `operation`: `$fromAI('operation', 'Operation', 'string')`  
       - `options`: `$fromAI('options', 'Options', 'json')` (optional)  
   - Add HTTP Request Tool node named `Delete batch`.  
     - Method: DELETE  
     - URL: `http://localhost:5000/batch/{{ $fromAI('id', 'Batch ID', 'string') }}`  
     - Authentication: API Key Header.  
   - Add HTTP Request Tool node named `Get batch`.  
     - Method: GET  
     - URL: `http://localhost:5000/batch/{{ $fromAI('id', 'Batch ID', 'string') }}`  
     - Authentication: API Key Header.  
   - Connect MCP Trigger node output to all four batch operation nodes using `ai_tool` connections or equivalent routing to allow dynamic selection.  

5. **Database Query Setup**  
   - Add Sticky Note with content "## Db".  
   - Add HTTP Request Tool node named `Query domain database`.  
     - Method: GET  
     - URL: `http://localhost:5000/db`  
     - Authentication: API Key Header.  
     - Query parameters:  
       - `query`: expression `$fromAI('query', 'Query (contact name, dns, domain etc)', 'string')`  
   - Connect MCP Trigger node output to this node using `ai_tool` connection.  

6. **Domain Operations Setup**  
   - Add Sticky Note with content "## Domains".  
   - Add HTTP Request Tool node named `Check domain availability`.  
     - Method: GET  
     - URL: `http://localhost:5000/domains/{{ $fromAI('domain', 'Domain', 'string') }}/check`  
     - Authentication: API Key Header.  
   - Add HTTP Request Tool node named `Check domain rank (authority).`  
     - Method: GET  
     - URL: `http://localhost:5000/domains/{{ $fromAI('domain', 'Domain', 'string') }}/rank`  
     - Authentication: API Key Header.  
   - Add HTTP Request Tool node named `WHOIS query for a domain`.  
     - Method: GET  
     - URL: `http://localhost:5000/domains/{{ $fromAI('domain', 'Domain', 'string') }}/whois`  
     - Authentication: API Key Header.  
     - Query parameter:  
       - `format`: expression `$fromAI('format', 'Format', 'string')` (optional)  
   - Connect MCP Trigger node output to each of these domain operation nodes via `ai_tool` connections.  

7. **Create API Key Credential**  
   - In n8n credentials, create a new API Key credential configured to send header `X-API-KEY` with the appropriate API key value (as per your Bulk WHOIS API server setup).  
   - Assign this credential to all HTTP Request Tool nodes.  

8. **Activate Workflow**  
   - Save and activate the workflow.  
   - Obtain the webhook URL from the MCP Trigger node (`https://your-n8n-instance/webhook/bulk-whois-mcp`).  

9. **Integrate with AI Agents**  
   - Configure your AI agent to send requests to the MCP webhook URL with appropriate parameters for the desired operation.  
   - The `$fromAI()` expressions in HTTP Request nodes automatically map AI input parameters to API request fields.  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| For integration help or custom automation requests, contact on Discord: https://discord.me/cfomodz                   | Support and community assistance                                                                                                                                  |
| Official n8n documentation for MCP and tool integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Reference for MCP Trigger node and advanced integration                                                                                                          |
| The workflow preserves the original API response structure to simplify AI agent processing and reduce transformation overhead | Design note ensuring seamless AI-agent integration                                                                                                               |
| Parameters are dynamically populated via `$fromAI()` expressions that map AI input to workflow parameters automatically | Enables flexible, AI-driven parameter passing                                                                                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.