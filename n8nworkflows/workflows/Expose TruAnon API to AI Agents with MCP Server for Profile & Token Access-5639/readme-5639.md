Expose TruAnon API to AI Agents with MCP Server for Profile & Token Access

https://n8nworkflows.xyz/workflows/expose-truanon-api-to-ai-agents-with-mcp-server-for-profile---token-access-5639


# Expose TruAnon API to AI Agents with MCP Server for Profile & Token Access

### 1. Workflow Overview

This workflow, titled **"Expose TruAnon API to AI Agents with MCP Server for Profile & Token Access"**, serves as a bridge between AI agents and the TruAnon Private API. Its main goal is to expose two key TruAnon API operations—retrieving user profiles and requesting access tokens—via an MCP (Modular Chat Platform) server interface.

The workflow logic is divided into three main blocks:

- **1.1 Setup and Documentation**: Provides instructions and overview for users to understand, deploy, and customize the workflow.
- **1.2 MCP Server Trigger**: Hosts the MCP server endpoint that listens for AI agent requests and routes them accordingly.
- **1.3 API Operations**: Contains two HTTP request tool nodes that interact with the TruAnon Private API endpoints:
  - Fetch User Profile
  - Generate Access Token

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation

- **Overview:**  
  This block contains sticky notes that provide comprehensive setup instructions, usage notes, workflow overview, and operation details. It is informational and aids users in deploying and understanding the workflow without additional documentation.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)

- **Node Details:**  

  - **Setup Instructions (Sticky Note)**
    - Type: Sticky Note  
    - Purpose: Guides users through importing, activating, and using the workflow. Includes notes on parameter auto-population via `$fromAI()` expressions, customization tips, and support contact links.
    - Configuration: Static content with detailed steps and links:
      - No authentication required.
      - Explains how to get the MCP webhook URL.
      - Mentions the two available API endpoints.
      - Provides a Discord link for help and official n8n documentation link.
    - Inputs/Outputs: None (informational only).
    - Edge cases: None.
  
  - **Workflow Overview (Sticky Note)**
    - Type: Sticky Note  
    - Purpose: Describes the TruAnon API requirements and the workflow’s functional structure.
    - Configuration: Static content summarizing:
      - API arguments required (TruAnon Service Identifier and Member Name).
      - No dependencies or caching needed.
      - Workflow converts TruAnon API into MCP interface.
      - Lists the two operations exposed.
    - Inputs/Outputs: None (informational only).
    - Edge cases: None.

#### 2.2 MCP Server Trigger

- **Overview:**  
  This block hosts the MCP trigger node that acts as the entry point for AI agents. It receives requests, initiates the workflow, and routes the requests to the appropriate API operation nodes.

- **Nodes Involved:**  
  - TruAnon Private MCP Server (MCP Trigger)

- **Node Details:**  

  - **TruAnon Private MCP Server (MCP Trigger)**
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: MCP server endpoint that listens for AI agent requests on the path `/truanon-private-mcp`.
    - Configuration:
      - Path: `truanon-private-mcp`
      - Webhook ID set internally for persistent endpoint.
    - Input connections: None (trigger node).
    - Output connections:  
      - Connected to both "Fetch User Profile" and "Generate Access Token" nodes as AI tools.
    - Version: Requires n8n version supporting MCP trigger nodes.
    - Edge cases:
      - Failure to receive valid requests from AI agents.
      - Timeout or network issues.
      - Incorrect path or webhook misconfiguration.
    - Sub-workflow: None.

#### 2.3 API Operations

- **Overview:**  
  This block contains two HTTP request tool nodes that implement the two TruAnon API endpoints exposed via the MCP server. Each node uses AI-populated parameters to dynamically query the API and return responses.

- **Nodes Involved:**  
  - Sticky Note (Get Profile)  
  - Fetch User Profile (HTTP Request Tool)  
  - Sticky Note2 (Request Token)  
  - Generate Access Token (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note (Get Profile)**
    - Type: Sticky Note  
    - Content: Title "Get Profile" to visually separate this operation.
    - Inputs/Outputs: None.

  - **Fetch User Profile (HTTP Request Tool)**
    - Type: `httpRequestTool`  
    - Role: Calls the TruAnon API endpoint to fetch user profile data.
    - Configuration:
      - URL: `https://staging.truanon.com/api/get_profile`
      - Method: GET (inferred from query parameters usage)
      - Query Parameters:
        - `id`: Populated dynamically via `$fromAI('id', 'This is your unique username or member ID', 'string')`
        - `service`: Populated dynamically via `$fromAI('service', 'The service name given to you by TruAnon', 'string')`
      - Tool Description: Explains parameters and API purpose.
    - Input Connections: From MCP Trigger as AI tool.
    - Output Connections: None (response returned to MCP automatically).
    - Version: Node version 4.2.
    - Edge cases:
      - Missing or invalid `id` or `service` parameters.
      - API endpoint unavailability or errors.
      - Network timeouts.
      - Response format changes.
    - Sub-workflow: None.

  - **Sticky Note2 (Request Token)**
    - Type: Sticky Note  
    - Content: Title "Request Token" to designate the token generation operation.
    - Inputs/Outputs: None.

  - **Generate Access Token (HTTP Request Tool)**
    - Type: `httpRequestTool`  
    - Role: Requests an access token from the TruAnon API.
    - Configuration:
      - URL: `https://staging.truanon.com/api/request_token`
      - Method: GET (inferred)
      - Query Parameters:
        - `id`: `$fromAI('id', 'This is your unique username or member ID', 'string')`
        - `service`: `$fromAI('service', 'The service name given to you by TruAnon', 'string')`
      - Tool Description: Explains parameters and API purpose.
    - Input Connections: From MCP Trigger as AI tool.
    - Output Connections: None.
    - Version: Node version 4.2.
    - Edge cases:
      - Invalid or missing parameters.
      - API errors or downtime.
      - Token generation failures.
    - Sub-workflow: None.

### 3. Summary Table

| Node Name                | Node Type                     | Functional Role                          | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                                  |
|--------------------------|-------------------------------|----------------------------------------|------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Setup Instructions       | Sticky Note                   | Setup and usage instructions           | None                   | None                            | Includes setup steps, usage notes, customization tips, and support links ([discord](https://discord.me/cfomodz), [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)) |
| Workflow Overview        | Sticky Note                   | Overview of workflow and API operations| None                   | None                            | Describes TruAnon API requirements and MCP server logic                                                      |
| TruAnon Private MCP Server| MCP Trigger                   | Entry point for AI agent requests      | None                   | Fetch User Profile, Generate Access Token |                                                                                                              |
| Sticky Note              | Sticky Note                   | Label for Get Profile operation        | None                   | None                            |                                                                                                              |
| Fetch User Profile       | HTTP Request Tool             | Calls TruAnon API to fetch user profile| TruAnon Private MCP Server | None                            |                                                                                                              |
| Sticky Note2             | Sticky Note                   | Label for Request Token operation      | None                   | None                            |                                                                                                              |
| Generate Access Token    | HTTP Request Tool             | Calls TruAnon API to generate token    | TruAnon Private MCP Server | None                            |                                                                                                              |

### 4. Reproducing the Workflow from Scratch

1. **Create the Setup Instructions Sticky Note:**
   - Add a Sticky Note node.
   - Paste the detailed setup instructions content including steps to import, activate, get MCP URL, connect AI agent, usage notes, customization tips, and support links.
   - Set color to blue (#4) and height ~1060px for readability.

2. **Create the Workflow Overview Sticky Note:**
   - Add another Sticky Note node.
   - Paste the workflow and API overview describing TruAnon API requirements, the MCP server role, and the two exposed operations.
   - Set width to 420px and height to 920px.

3. **Create the MCP Trigger Node:**
   - Add an MCP Trigger node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.
   - Configure the webhook path as `truanon-private-mcp`.
   - No authentication is required.
   - This node acts as the entry point for AI agents.

4. **Create the “Get Profile” Sticky Note:**
   - Add a Sticky Note node.
   - Set its content to "## Get Profile" to label the first API operation.
   - Set color to yellow (#2), width to 300px, height to 200px.

5. **Create the Fetch User Profile HTTP Request Tool Node:**
   - Add an HTTP Request Tool node.
   - Set URL to `https://staging.truanon.com/api/get_profile`.
   - Method: GET.
   - Enable sending query parameters.
   - Add two query parameters:
     - `id` with value expression: `{{$fromAI('id', 'This is your unique username or member ID', 'string')}}`
     - `service` with value expression: `{{$fromAI('service', 'The service name given to you by TruAnon', 'string')}}`
   - Add a tool description explaining this fetches the user profile with optional `id` and `service`.
   - Connect this node’s AI tool input to the MCP Trigger node.

6. **Create the “Request Token” Sticky Note:**
   - Add a Sticky Note node.
   - Content: "## Request Token".
   - Set color to green (#3), width 300px, height 200px.

7. **Create the Generate Access Token HTTP Request Tool Node:**
   - Add an HTTP Request Tool node.
   - URL: `https://staging.truanon.com/api/request_token`.
   - Method: GET.
   - Query parameters:
     - `id`: `{{$fromAI('id', 'This is your unique username or member ID', 'string')}}`
     - `service`: `{{$fromAI('service', 'The service name given to you by TruAnon', 'string')}}`
   - Add tool description about token retrieval.
   - Connect this node’s AI tool input to the MCP Trigger node.

8. **Workflow Activation:**
   - Save and activate the workflow.
   - Copy the webhook URL from the MCP Trigger node for use in AI agent configurations.

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow requires no authentication and parameters are auto-populated by AI via `$fromAI()` expressions. | Usage detail                                                                                                    |
| For integration guidance and custom automations, contact via Discord: https://discord.me/cfomodz    | Support contact                                                                                                 |
| Official n8n documentation on MCP nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Reference documentation                                                                                          |
| TruAnon API requires two arguments on all endpoints: service identifier and member name; no caching needed due to continuous data updates. | API integration note                                                                                             |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. The processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.