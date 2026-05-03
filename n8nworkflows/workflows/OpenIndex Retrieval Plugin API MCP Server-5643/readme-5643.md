OpenIndex Retrieval Plugin API MCP Server

https://n8nworkflows.xyz/workflows/openindex-retrieval-plugin-api-mcp-server-5643


# OpenIndex Retrieval Plugin API MCP Server

---
### 1. Workflow Overview

This workflow, titled **OpenIndex Retrieval Plugin API MCP Server**, functions as a server endpoint for AI agents to perform natural language-based document retrieval and filtering via the OpenIndex Retrieval Plugin API. It exposes a single operation allowing AI agents to submit queries and receive responses directly, preserving the original API structure.

The workflow logic is divided into the following blocks:

- **1.1 Setup and Documentation**: Contains detailed setup instructions and overview documentation for users to understand and deploy the workflow.
- **1.2 MCP Server Trigger**: Listens for incoming AI agent requests on a webhook URL, acting as the server entry point.
- **1.3 API Request Handling**: Processes incoming queries by forwarding them to the OpenIndex Retrieval Plugin API endpoint and returns the API responses to the AI agent.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Documentation

- **Overview:**  
  This block provides users with comprehensive setup instructions, usage notes, customization tips, and support information. It ensures users can correctly import, configure, and operate the workflow.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)

- **Node Details:**  

  - **Setup Instructions**  
    - Type & Role: Sticky Note node used for documentation.  
    - Configuration: Contains a multi-section markdown text with setup steps, usage notes, customization advice, and support links.  
    - Key Content: Instructions to import the workflow, activate it, retrieve the MCP URL, connect AI agents, and general customization guidelines. Includes a Discord support link and n8n docs for MCP nodes.  
    - Inputs/Outputs: None (informational only).  
    - Edge Cases: None, as it is purely descriptive.  

  - **Workflow Overview**  
    - Type & Role: Sticky Note node summarizing workflow purpose and operation.  
    - Configuration: Markdown content describing the workflow’s function as a retrieval API, operation details, the MCP trigger role, HTTP request handling, and parameter automation via AI expressions.  
    - Inputs/Outputs: None (informational only).  
    - Edge Cases: None.

#### 1.2 MCP Server Trigger

- **Overview:**  
  Acts as the webhook endpoint that listens for incoming requests from AI agents. This node initiates the workflow upon receiving a request.

- **Nodes Involved:**  
  - OpenIndex Retrieval Plugin MCP Server (MCP Trigger)

- **Node Details:**  

  - **OpenIndex Retrieval Plugin MCP Server**  
    - Type & Role: MCP Trigger node, specialized for handling AI agent requests via a webhook.  
    - Configuration:  
      - Webhook Path: `openindex-retrieval-plugin-mcp` (defines the endpoint URL).  
      - No authentication required (open endpoint).  
    - Key Expressions: Parameters are dynamically populated from AI request context using `$fromAI()` expressions internally (not explicitly shown).  
    - Input: Incoming webhook request from AI agent.  
    - Output: Forwards data to the next node for processing.  
    - Version-specific: Requires n8n version supporting MCP Trigger nodes.  
    - Edge Cases: Possible failures include webhook misconfiguration, network errors, or malformed AI requests.

#### 1.3 API Request Handling

- **Overview:**  
  Handles the actual API interaction by submitting queries received from the MCP trigger to the OpenIndex Retrieval Plugin API and returning the raw API response to the AI agent.

- **Nodes Involved:**  
  - Sticky Note ("Query")  
  - Submit Query (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note ("Query")**  
    - Type & Role: Sticky Note node labeling this block for clarity.  
    - Configuration: Simple markdown header "## Query".  
    - Inputs/Outputs: None.  

  - **Submit Query**  
    - Type & Role: HTTP Request Tool node, performs the POST request to the API endpoint.  
    - Configuration:  
      - URL: `https://retriever.openindex.ai/sub/query`  
      - Method: POST  
      - Options: Default (no special headers or authentication).  
      - Tool Description: "Query" (indicating the node’s functional role).  
      - Parameters: Automatically populated from AI agent request via MCP context (through `$fromAI()` expressions handled internally by MCP trigger).  
    - Input: Receives query data from MCP trigger node.  
    - Output: Returns the API response directly back to the MCP node, which sends it to the AI agent.  
    - Edge Cases:  
      - Network timeouts or unreachable API endpoint.  
      - Invalid or malformed query data causing API errors.  
      - API rate limiting or service unavailability.  
      - No authentication is required by the API, but if API changes, this might need update.

---

### 3. Summary Table

| Node Name                        | Node Type                 | Functional Role                          | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                          |
|---------------------------------|---------------------------|----------------------------------------|----------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Setup Instructions              | Sticky Note               | Provides setup and usage instructions  | None                             | None                          | ### ⚙️ Setup Instructions 1. Import Workflow; 2. No auth needed; 3. Activate workflow; 4. Get MCP URL; 5. Connect AI agent. Usage notes and customization tips included. Discord support link: https://discord.me/cfomodz and n8n docs link provided. |
| Workflow Overview               | Sticky Note               | Summarizes workflow purpose and usage  | None                             | None                          | ## 🛠️ OpenIndex Retrieval Plugin MCP Server: retrieval API, MCP trigger, HTTP request nodes, AI expressions, one operation supported. |
| OpenIndex Retrieval Plugin MCP Server | MCP Trigger             | Entry point webhook for AI agent requests | None                             | Submit Query                  |                                                                                                    |
| Sticky Note ("Query")           | Sticky Note               | Labels the query processing block      | None                             | None                          | ## Query                                                                                           |
| Submit Query                   | HTTP Request Tool          | Sends query to OpenIndex API and returns response | OpenIndex Retrieval Plugin MCP Server | None                          |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Add a Sticky Note node.  
   - Paste detailed setup instructions: import steps, no auth needed, activation, MCP URL retrieval, AI agent connection, usage notes, customization tips, Discord link (https://discord.me/cfomodz), and n8n docs link (https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/).  
   - Set color to teal (color 4), height to approx. 1060.

2. **Create Sticky Note: Workflow Overview**  
   - Add another Sticky Note node.  
   - Enter markdown summarizing the workflow’s purpose as an OpenIndex Retrieval Plugin MCP server, describing the MCP trigger, HTTP requests, AI expressions, and supported operation.  
   - Set width to 420, height to 740.

3. **Add MCP Trigger Node: OpenIndex Retrieval Plugin MCP Server**  
   - Select the MCP Trigger node type (requires n8n version with Langchain MCP nodes).  
   - Set the webhook path to `openindex-retrieval-plugin-mcp`.  
   - No authentication required.  
   - This node will listen for incoming AI agent requests.

4. **Add Sticky Note: Query Label**  
   - Add a Sticky Note node with content "## Query".  
   - Set color to green (color 2), width 300, height 200.

5. **Add HTTP Request Tool Node: Submit Query**  
   - Add HTTP Request Tool node.  
   - Set Method to POST.  
   - Set URL to `https://retriever.openindex.ai/sub/query`.  
   - Leave options as default (no auth or headers).  
   - Enter tool description as "Query".  
   - Parameters will be auto-populated via MCP context from AI agent requests (`$fromAI()` expressions are managed internally by the MCP node).  

6. **Connect Nodes**  
   - Connect MCP Trigger node output to Submit Query node input (using the ai_tool output).  
   - Connect Sticky Note "Query" nearby for visual grouping only.

7. **Activate Workflow**  
   - Enable workflow in n8n to start listening on the defined webhook URL.

8. **Usage**  
   - Provide the webhook URL (base n8n URL + `/webhook/openindex-retrieval-plugin-mcp`) to your AI agent.  
   - The AI agent sends queries, which are forwarded to the OpenIndex Retrieval Plugin API, with responses returned transparently.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| No authentication is required for this workflow’s MCP endpoint or the OpenIndex API.                           | Workflow Setup Instructions                                                                              |
| Parameters in HTTP requests are auto-populated by AI agent context using `$fromAI()` expressions internally.    | Workflow Overview and Setup Instructions                                                                |
| Discord support channel available for integration help: https://discord.me/cfomodz                             | Setup Instructions                                                                                      |
| Official n8n documentation for MCP and Langchain nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions                                                                                      |
| The workflow exposes a single operation endpoint: Submit Query.                                               | Workflow Overview                                                                                       |

---

Disclaimer: The provided content originates solely from an automated n8n workflow. All data handled is legal and public, adhering to content policies.