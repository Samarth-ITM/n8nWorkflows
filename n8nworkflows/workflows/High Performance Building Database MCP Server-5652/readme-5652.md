High Performance Building Database MCP Server

https://n8nworkflows.xyz/workflows/high-performance-building-database-mcp-server-5652


# High Performance Building Database MCP Server

### 1. Workflow Overview

This workflow, titled **High Performance Building Database MCP Server**, serves as an MCP-compatible server interface for the High Performance Building Database API. It is designed to facilitate AI agents in querying detailed data about green and high-performance building projects collected by the U.S. Department of Energy and the National Renewable Energy Laboratory (NREL).

The workflow is divided into the following logical blocks:

- **1.1 Setup & Documentation**: Provides setup instructions, usage notes, and project overview information for users and developers.
- **1.2 MCP Server Trigger**: Listens for incoming MCP (Modular Cognition Protocol) requests from AI agents as the server endpoint.
- **1.3 API Operations**: Handles two main API operations corresponding to endpoints of the High Performance Building Database:
  - **List Projects**: Returns a filtered list of building projects.
  - **Get Project Details**: Returns detailed information about a specific building project.

These blocks interact sequentially where the MCP trigger receives requests, routes them to the appropriate HTTP request nodes to fetch data from the external API, and returns the response back to the AI agent.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Documentation

- **Overview:**  
  This block provides comprehensive instructions and contextual information about the workflow, including setup steps, usage notes, and a project overview. It aids users in understanding and operating the workflow effectively.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  
  - Description - project.json (Sticky Note)  
  - Sticky Note labeled "Projects"

- **Node Details:**  

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Provides detailed setup and usage guidance including authentication (none required), activation steps, how to get the MCP URL, and customization tips.  
    - Configuration: Text content formatted with markdown, including links to Discord support and official n8n documentation.  
    - Input/Output: None (informational only)  
    - Edge Cases: None (informational node)

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Describes the project background, workflow purpose, and key operations supported.  
    - Configuration: Markdown text explaining the High Performance Building Database, MCP integration, and the two API operations exposed.  
    - Input/Output: None (informational)  
    - Edge Cases: None

  - **Description - project.json**  
    - Type: Sticky Note  
    - Role: Summarizes the API endpoint category related to project data.  
    - Configuration: Short markdown note titled “Project.Json”.  
    - Input/Output: None

  - **Sticky Note ("Projects")**  
    - Type: Sticky Note  
    - Role: Section header label for the subsequent nodes handling project-related API calls.  
    - Configuration: Short markdown heading “Projects”.  
    - Input/Output: None

#### 2.2 MCP Server Trigger

- **Overview:**  
  This node acts as the entry point for AI agent requests via the MCP protocol, exposing a webhook endpoint that listens for incoming API calls routed through the MCP interface.

- **Nodes Involved:**  
  - High Performance Building Database MCP Server (MCP Trigger)

- **Node Details:**  

  - **High Performance Building Database MCP Server**  
    - Type: MCP Trigger (specialized n8n node for Modular Cognition Protocol)  
    - Role: Starts the workflow upon receiving MCP requests at path `/high-performance-building-database-mcp`.  
    - Configuration:  
      - Webhook path set to `high-performance-building-database-mcp`.  
      - No authentication required.  
    - Input: External MCP requests.  
    - Output: Routes requests to downstream nodes for processing.  
    - Version: Requires n8n version supporting MCP nodes (commonly 1.95+).  
    - Failure Types: Network issues, invalid requests, or MCP protocol errors.

#### 2.3 API Operations

- **Overview:**  
  This block processes two main API calls to the High Performance Building Database: one to list projects with filtering options, and another to retrieve detailed information about a specific project. Both nodes dynamically populate parameters from AI agent inputs using `$fromAI()` expressions.

- **Nodes Involved:**  
  - List Projects (HTTP Request Tool)  
  - Get Project Details (HTTP Request Tool)

- **Node Details:**  

  - **List Projects**  
    - Type: HTTP Request Tool (configured as API tool)  
    - Role: Calls the `/project.{output_format}` API endpoint to fetch a filtered list of building projects.  
    - Configuration:  
      - URL path: `/project.{{ $fromAI('output_format', 'Response Format', 'string', 'xml') }}` (output format dynamically injected, defaulting to XML).  
      - Query parameters include:  
        - `api_key` (required, default `DEMO_KEY`)  
        - `search` (optional filter text)  
        - `portal` (optional portal ID)  
        - `page` (optional pagination)  
        - `city`, `province`, `region` (optional geographic/climate filters)  
      - Parameters auto-filled via `$fromAI()` expressions for dynamic AI-driven input.  
    - Input: MCP trigger passes parameters from AI agent.  
    - Output: Returns API response in original structure to AI agent.  
    - Failure Types: API key invalid/expired, network timeout, invalid parameter values, expression evaluation failure.

  - **Get Project Details**  
    - Type: HTTP Request Tool (configured as API tool)  
    - Role: Calls the `/project/{project_id}.{output_format}` API endpoint to fetch detailed data for a specific project.  
    - Configuration:  
      - URL path uses dynamic project ID and output format: `/project/{{ $fromAI('project_id', 'Project ID', 'number') }}.{{ $fromAI('output_format', 'Response Format', 'string', 'json') }}`  
      - Query parameter: `api_key` (required, default `DEMO_KEY`)  
      - Parameters auto-filled from AI input via `$fromAI()`.  
    - Input: MCP trigger passes parameters from AI agent.  
    - Output: API response sent back to AI agent, preserving original structure.  
    - Failure Types: Missing or invalid project ID, API key issues, network errors, expression failures.

---

### 3. Summary Table

| Node Name                            | Node Type                   | Functional Role                                  | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                          |
|------------------------------------|-----------------------------|-------------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| Setup Instructions                 | Sticky Note                  | Setup and usage instructions                     | None                             | None                             | Contains detailed setup steps, usage notes, customization tips, and support links                  |
| Workflow Overview                 | Sticky Note                  | Workflow and project overview                     | None                             | None                             | Describes the database, workflow purpose, and API operations                                      |
| Description - project.json        | Sticky Note                  | API endpoint category information                 | None                             | None                             | Brief note about project-related API endpoints                                                     |
| Sticky Note ("Projects")          | Sticky Note                  | Section header for project API nodes              | None                             | None                             | Section label for project-related nodes                                                           |
| High Performance Building Database MCP Server | MCP Trigger                | Receives MCP requests, acts as server endpoint   | None                             | List Projects, Get Project Details |                                                                                                    |
| List Projects                    | HTTP Request Tool            | Fetches filtered list of building projects        | High Performance Building Database MCP Server | None                             | Tool to query projects with multiple optional filters; uses dynamic AI-driven parameters           |
| Get Project Details              | HTTP Request Tool            | Fetches detailed data for a specific project      | High Performance Building Database MCP Server | None                             | Tool to retrieve project details by project ID; parameters auto-populated from AI input           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Setup & Documentation**  
   - Add a Sticky Note named **"Setup Instructions"** with the provided markdown content describing setup, usage, and customization.  
   - Add a Sticky Note named **"Workflow Overview"** containing the project background, workflow purpose, and operations.  
   - Add a Sticky Note named **"Description - project.json"** summarizing the project API endpoint category.  
   - Add a Sticky Note named **"Projects"** as a section header for project API nodes.

2. **Add MCP Trigger Node**  
   - Create a node of type **MCP Trigger**.  
   - Set the webhook path to `high-performance-building-database-mcp`.  
   - No authentication is needed.  
   - This node will listen for incoming MCP requests from AI agents.

3. **Add HTTP Request Tool Node for "List Projects"**  
   - Create an **HTTP Request Tool** node named **"List Projects"**.  
   - Set the URL to `/project.{{ $fromAI('output_format', 'Response Format', 'string', 'xml') }}` to dynamically specify response format (default XML).  
   - Enable sending query parameters.  
   - Add query parameters with the following keys and dynamic values using `$fromAI()` expressions:  
     - `api_key` with default `"DEMO_KEY"`  
     - `search` (optional)  
     - `portal` (optional)  
     - `page` (optional)  
     - `city` (optional)  
     - `province` (optional)  
     - `region` (optional) with the provided climate region integer mapping as guidance.  
   - Leave options as default (no extra headers or authentication).  
   - This node interfaces with the list projects API endpoint.

4. **Add HTTP Request Tool Node for "Get Project Details"**  
   - Create an **HTTP Request Tool** node named **"Get Project Details"**.  
   - Set the URL to `/project/{{ $fromAI('project_id', 'Project ID', 'number') }}.{{ $fromAI('output_format', 'Response Format', 'string', 'json') }}` to dynamically specify project ID and response format (default JSON).  
   - Enable sending query parameters.  
   - Add a query parameter for `api_key` with a default of `"DEMO_KEY"`.  
   - This node interfaces with the project details API endpoint.

5. **Connect Nodes**  
   - Connect the **MCP Trigger** node outputs to both **List Projects** and **Get Project Details** nodes.  
   - This allows the MCP server to route incoming requests to the appropriate API operation based on input parameters.

6. **Configure Credentials**  
   - No explicit credentials are required unless a real API key is used.  
   - If needed, replace `"DEMO_KEY"` in `$fromAI()` expressions with a real API key value.  
   - No OAuth or other authentication configured in the nodes.

7. **Activate Workflow**  
   - Save and activate the workflow in your n8n instance.  
   - Copy the webhook URL from the MCP Trigger node for AI agent integration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Parameters in HTTP request nodes are dynamically populated using `$fromAI()` expressions, enabling seamless AI agent interaction with the API endpoints.            | Usage Notes in Setup Instructions sticky note                                                                |
| The workflow exposes two main API endpoints as tools for AI agents: List Projects and Get Project Details. Responses maintain the original API response structure. | Workflow Overview sticky note                                                                                 |
| No authentication is required to use the workflow with the demo API key, but the API key parameter can be customized for production use.                           | Setup Instructions sticky note                                                                                 |
| For integration assistance or custom automation guidance, reach out on Discord: https://discord.me/cfomodz                                                          | Setup Instructions sticky note                                                                                 |
| Official n8n documentation for MCP nodes and integration tools is available at https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions sticky note                                                                                 |

---

This structured reference provides a comprehensive understanding of all nodes, their roles, configurations, and how to reconstruct the entire workflow manually. It anticipates common failure modes and integration points to facilitate robust customization and maintenance.