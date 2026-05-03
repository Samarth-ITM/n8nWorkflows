Mobility API MCP Server

https://n8nworkflows.xyz/workflows/mobility-api-mcp-server-5651


# Mobility API MCP Server

### 1. Workflow Overview

This workflow, titled **Mobility API MCP Server**, is designed to serve as an MCP-compatible server endpoint for AI agents to query mobility data related to transit movements within the Czech Republic. It exposes two API endpoints of the Mobility API from O2’s developer platform, allowing retrieval of application info and calculation of transit routes between basic residential units.

The workflow is logically divided into the following blocks:

- **1.1 Setup & Documentation:** Provides setup instructions and describes the workflow's purpose and usage notes.
- **1.2 MCP Trigger:** Accepts incoming AI agent requests via a webhook acting as the server endpoint.
- **1.3 Info Retrieval Block:** Handles the API call to retrieve general application info.
- **1.4 Transit Calculation Block:** Handles the API call to calculate transit routes based on dynamic path and query parameters supplied by AI expressions.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation

- **Overview:**  
  This block contains sticky notes that provide setup instructions, workflow overview, usage notes, and customization guidelines. It serves as in-workflow documentation for users to understand, deploy, and extend the workflow.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)

- **Node Details:**  

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Provides detailed, step-wise instructions for importing, activating, and using the workflow, including notes on parameter auto-population via AI expressions, available API endpoints, and customization tips.  
    - Configuration: Static text with markdown formatting, emphasizing no authentication requirement, MCP URL usage, and AI integration.  
    - Connections: None (documentation only)  
    - Edge Cases: None  

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Summarizes the high-level design, explaining the transit API’s purpose, workflow components, and available operations (Info and Transit).  
    - Configuration: Static markdown content outlining the API, AI integration, and workflow architecture.  
    - Connections: None (documentation only)  
    - Edge Cases: None  

---

#### 1.2 MCP Trigger

- **Overview:**  
  The core entry point of the workflow which listens for incoming AI agent requests via a webhook. It acts as an MCP server trigger that routes requests to appropriate API call nodes.

- **Nodes Involved:**  
  - Mobility MCP Server (MCP Trigger node)

- **Node Details:**  

  - **Mobility MCP Server**  
    - Type: MCP Trigger (custom LangChain node)  
    - Role: Exposes a webhook endpoint `/mobility-mcp` to receive AI agent requests; acts as the server-side trigger for processing.  
    - Configuration: Path set to `mobility-mcp`; webhookId assigned by n8n.  
    - Input: Incoming HTTP requests from AI agents.  
    - Output: Routes requests to HTTP Request nodes based on MCP tool integration.  
    - Version-specific: Requires n8n version supporting LangChain MCP nodes.  
    - Edge Cases: Webhook availability, malformed requests, or version incompatibility could cause failures.  
    - Notes: This node is the only trigger node; all subsequent logic depends on it.

---

#### 1.3 Info Retrieval Block

- **Overview:**  
  Handles retrieval of application information from the Mobility API’s `/info` endpoint. This provides metadata about application and data versions.

- **Nodes Involved:**  
  - Sticky Note (Info)  
  - Retrieve Application Info (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note (Info)**  
    - Type: Sticky Note  
    - Role: Labels the block for clarity with the title “Info”.  
    - Configuration: Static text `"## Info"`  
    - Connections: None  
    - Edge Cases: None  

  - **Retrieve Application Info**  
    - Type: HTTP Request Tool  
    - Role: Sends GET request to `https://developer.o2.cz/mobility/sandbox/api/info` to fetch API info.  
    - Configuration:  
      - URL: Hardcoded to the `/info` endpoint.  
      - No additional parameters or authentication required.  
      - Tool description explains the purpose (information about application and data versions).  
    - Input: Triggered by MCP Trigger node via AI tool interface.  
    - Output: Response data passed back to MCP trigger for returning to AI agent.  
    - Edge Cases: Network errors, API downtime, or unexpected response formats.  
    - Notes: Returns raw API response to maintain data fidelity.

---

#### 1.4 Transit Calculation Block

- **Overview:**  
  Performs transit route calculation between two specified basic residential units, using dynamic path and query parameters populated automatically by AI expressions.

- **Nodes Involved:**  
  - Sticky Note2 (Transit)  
  - Calculate Transit Route (HTTP Request Tool)

- **Node Details:**  

  - **Sticky Note2 (Transit)**  
    - Type: Sticky Note  
    - Role: Labels the block with the title “Transit”.  
    - Configuration: Static text `"## Transit"`  
    - Connections: None  
    - Edge Cases: None  

  - **Calculate Transit Route**  
    - Type: HTTP Request Tool  
    - Role: Sends GET request to the `/transit/{from}/{to}` endpoint with query parameters to calculate transit data.  
    - Configuration:  
      - URL: Uses template expressions to dynamically inject `from` and `to` path parameters via `$fromAI()` expressions.  
      - Query Parameters:  
        - `fromType`, `toType`, and `uniques` are also dynamically populated using `$fromAI()` expressions with descriptions guiding AI input.  
      - Tool description details parameter meanings and expected values.  
      - Sends query parameters along with path parameters.  
    - Input: Triggered by MCP Trigger node via AI tool interface.  
    - Output: API response forwarded to MCP trigger to be returned to the AI agent.  
    - Edge Cases:  
      - Missing or invalid `from` or `to` values could cause API errors.  
      - Invalid query parameter values may lead to unexpected results.  
      - Timeout or network issues possible.  
      - AI expression failures if expected parameters are not properly provided.  
    - Notes: Designed for flexible AI-driven parameter input, enabling automated and dynamic transit queries.

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                          | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                                                      |
|-----------------------|----------------------------|----------------------------------------|----------------------|------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions    | Sticky Note                | Setup and usage documentation           | None                 | None                   | Provides detailed setup instructions, usage notes, customization tips, and help resources including Discord and docs links.     |
| Workflow Overview     | Sticky Note                | Workflow high-level overview             | None                 | None                   | Describes the Mobility MCP Server purpose, API info, endpoints, and AI integration overview.                                    |
| Mobility MCP Server   | MCP Trigger                | Entry point webhook for AI agent requests | None                 | Retrieve Application Info, Calculate Transit Route | Serves as the webhook endpoint `/mobility-mcp` for AI agents; triggers API calls.                                                |
| Sticky Note           | Sticky Note                | Label for Info block                     | None                 | None                   | Displays “## Info” label for clarity.                                                                                          |
| Retrieve Application Info | HTTP Request Tool        | Calls Mobility API `/info` endpoint     | Mobility MCP Server   | Mobility MCP Server     | Retrieves application and data version information from the Mobility API.                                                      |
| Sticky Note2          | Sticky Note                | Label for Transit block                  | None                 | None                   | Displays “## Transit” label for clarity.                                                                                       |
| Calculate Transit Route | HTTP Request Tool         | Calls Mobility API `/transit` endpoint  | Mobility MCP Server   | Mobility MCP Server     | Calculates transit routes with dynamic path and query parameters populated by AI expressions.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Setup Instructions Sticky Note:**  
   - Type: Sticky Note  
   - Content: Insert setup instructions text describing import, activation, URL retrieval, AI agent connection, usage notes, customization, and help links.  
   - Position: Left side, for easy visibility.

2. **Create Workflow Overview Sticky Note:**  
   - Type: Sticky Note  
   - Content: Provide a high-level description of the Mobility MCP Server, API purpose, AI integration, and available operations.  
   - Position: Adjacent to Setup Instructions.

3. **Add MCP Trigger Node:**  
   - Type: MCP Trigger (LangChain)  
   - Parameters:  
     - Path: `mobility-mcp`  
   - Position: Center-left.  
   - Purpose: This node acts as the webhook entry point for AI agent requests.

4. **Add Sticky Note Label for Info Block:**  
   - Type: Sticky Note  
   - Content: `## Info`  
   - Position: Near Info HTTP Request node.

5. **Add HTTP Request Tool Node: Retrieve Application Info:**  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://developer.o2.cz/mobility/sandbox/api/info`  
     - HTTP Method: GET  
     - No authentication required.  
     - Add descriptive tool description about its purpose.  
   - Position: Right of MCP Trigger, under Info label.

6. **Add Sticky Note Label for Transit Block:**  
   - Type: Sticky Note  
   - Content: `## Transit`  
   - Position: Near Transit HTTP Request node.

7. **Add HTTP Request Tool Node: Calculate Transit Route:**  
   - Type: HTTP Request Tool  
   - Parameters:  
     - URL Template: `https://developer.o2.cz/mobility/sandbox/api/transit/{{ $fromAI('from', 'source basic residential unit', 'string') }}/{{ $fromAI('to', 'destination basic residential unit', 'string') }}`  
     - HTTP Method: GET  
     - Query Parameters:  
       - `fromType` = `{{ $fromAI('fromType', 'occurence type in the source basic residential unit (1 - transit, 2 - visit, 0 - both)', 'string') }}`  
       - `toType` = `{{ $fromAI('toType', 'occurence type in the destination basic residential unit (1 - transit, 2 - visit, 0 - both)', 'string') }}`  
       - `uniques` = `{{ $fromAI('uniques', 'all or only uniques (0 - all, 1 - uniques)', 'string') }}`  
     - Add descriptive tool description explaining parameters.  
   - Position: Below Info block, right of MCP Trigger.

8. **Connect Nodes:**  
   - Connect MCP Trigger node’s AI tool output to both HTTP Request Tool nodes’ AI tool input.  
   - Ensure responses flow back to MCP Trigger for returning to the AI agent.

9. **Activate Workflow:**  
   - Enable the workflow in n8n to make the webhook live.

10. **Usage:**  
    - Provide the webhook URL (`/mobility-mcp`) to AI agent configurations to enable dynamic requests for info and transit operations.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                                                             |
|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Parameters in HTTP requests are auto-populated dynamically using `$fromAI()` expressions by the AI agent.     | Usage Notes in Setup Instructions Sticky Note                                                                                              |
| No authentication is required to use the Mobility API sandbox endpoints.                                      | Setup Instructions Sticky Note                                                                                                             |
| For integration help or custom automation support, join the Discord community: https://discord.me/cfomodz     | Setup Instructions Sticky Note                                                                                                             |
| Detailed documentation on the LangChain MCP node can be found at: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions Sticky Note                                                                                                             |
| The transit API provides data based on mobile station movement in the O2 mobile network within Czech Republic | Workflow Overview Sticky Note                                                                                                              |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow, fully compliant with applicable content policies. The data handled is legal and public.