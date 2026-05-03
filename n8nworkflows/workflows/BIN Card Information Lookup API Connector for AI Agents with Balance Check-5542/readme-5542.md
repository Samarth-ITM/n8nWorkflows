BIN Card Information Lookup API Connector for AI Agents with Balance Check

https://n8nworkflows.xyz/workflows/bin-card-information-lookup-api-connector-for-ai-agents-with-balance-check-5542


# BIN Card Information Lookup API Connector for AI Agents with Balance Check

### 1. Workflow Overview

This workflow, titled **BIN Card Information Lookup API Connector for AI Agents with Balance Check**, provides an interface to the BIN Lookup API from bintable.com. It enables AI agents to query card BIN (Bank Identification Number) information and check API balance through a Managed Connector Platform (MCP) server setup inside n8n.

The workflow is structured into the following logical blocks:

- **1.1 Setup and Documentation**: Provides setup instructions and overview notes as sticky notes for user reference.
- **1.2 MCP Server Trigger**: Defines the API endpoint to receive requests from AI agents.
- **1.3 Balance Check Operation**: Queries the balance of the API key usage.
- **1.4 BIN Lookup Operation**: Looks up BIN card information using the provided BIN code.

The design leverages MCP triggers and HTTP Request Tool nodes to expose the external API with AI-driven parameter injection using `$fromAI()` expressions, facilitating seamless integration with AI agents.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation Block

**Overview:**  
This block provides comprehensive user guidance through sticky notes. It explains setup steps, usage instructions, customization options, and how to connect AI agents to the MCP server endpoint.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Documentation and user guidance on setup and usage.  
  - Configuration: Contains setup steps (importing workflow, no authentication required, activation instructions), usage notes (auto-populated parameters via AI expressions), customization tips (error handling, logging), and support contact links including Discord and official n8n docs.  
  - Connections: None (informational only).  
  - Edge Cases: None (non-executable node).

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Describes the workflow purpose, operations available, and general architecture.  
  - Configuration: Explains that the workflow exposes 2 operations (balance check and BIN lookup) provided by bintable.com via MCP trigger and HTTP request nodes. Includes API endpoint URLs and parameter handling notes.  
  - Connections: None (informational only).  
  - Edge Cases: None.

---

#### 2.2 MCP Server Trigger Block

**Overview:**  
This block hosts the MCP trigger node which acts as the HTTP webhook endpoint for AI agents to send BIN lookup or balance check requests.

**Nodes Involved:**  
- BIN Lookup MCP Server (MCP Trigger)

**Node Details:**

- **BIN Lookup MCP Server**  
  - Type: MCP Trigger (n8n-nodes-langchain.mcpTrigger)  
  - Role: Exposes a webhook path `bin-lookup-mcp` to listen for incoming AI agent requests and route them to the appropriate operation.  
  - Configuration:  
    - Path: `bin-lookup-mcp` (custom webhook path).  
    - No authentication configured (public endpoint).  
  - Inputs: None (trigger node).  
  - Outputs: Connected downstream to the two HTTP Request Tool nodes via MCP tool connections.  
  - Version-specific: Requires n8n version supporting MCP trigger nodes.  
  - Edge Cases:  
    - Webhook connectivity issues if server is offline.  
    - Potential denial-of-service if exposed publicly without authentication.  
    - Input validation depends on downstream nodes.  
  - Sub-workflow: None.

---

#### 2.3 Balance Check Operation Block

**Overview:**  
This block allows checking the API balance for the provided API key by querying the bintable.com balance endpoint.

**Nodes Involved:**  
- Sticky Note (Balance)  
- Check Balance (HTTP Request Tool)

**Node Details:**

- **Sticky Note (Balance)**  
  - Type: Sticky Note  
  - Role: Marks and documents the balance check section.  
  - Configuration: Simple title "Balance".  
  - Connections: None.

- **Check Balance**  
  - Type: HTTP Request Tool (n8n-nodes-base.httpRequestTool)  
  - Role: Calls the balance endpoint of bintable.com API to retrieve balance information for the user's API key.  
  - Configuration:  
    - URL: `https://api.bintable.com/v1/balance`  
    - Method: GET (default)  
    - Query Parameters: `api_key` dynamically injected by AI expression: `{{$fromAI('api_key', 'The API key, which you can get from bintable.com website.', 'string')}}`  
    - No body or headers configured.  
    - Tool Description: Provides parameter documentation for the API key.  
  - Inputs: Receives from MCP trigger node as AI tool.  
  - Outputs: Sends response back to MCP trigger output (response to AI agent).  
  - Edge Cases:  
    - Invalid or missing API key leading to authentication errors.  
    - Network timeouts or API service outages.  
    - Rate limiting from bintable.com.  
    - Expression failure if AI does not supply `api_key`.  
  - Version-specific: Requires HTTP Request Tool version 4.2 or higher.

---

#### 2.4 BIN Lookup Operation Block

**Overview:**  
This block performs the BIN lookup operation by querying the BIN code information from the bintable.com API.

**Nodes Involved:**  
- Sticky Note (Lookup)  
- Lookup for bin (HTTP Request Tool)

**Node Details:**

- **Sticky Note (Lookup)**  
  - Type: Sticky Note  
  - Role: Marks and documents the BIN lookup section.  
  - Configuration: Simple title "Lookup".  
  - Connections: None.

- **Lookup for bin**  
  - Type: HTTP Request Tool (n8n-nodes-base.httpRequestTool)  
  - Role: Calls the BIN lookup endpoint with the BIN code supplied by the AI agent.  
  - Configuration:  
    - URL Template: `https://api.bintable.com/v1/{{ $fromAI('bin', 'pass the required BIN code', 'string') }}`  
    - Method: GET (default)  
    - Query Parameters: `api_key` dynamically injected by AI expression: `{{$fromAI('api_key', 'The API key, which you can get from bintable.com website.', 'string')}}`  
    - Tool Description: Documents required path parameter `bin` and query parameter `api_key`.  
  - Inputs: Receives from MCP trigger as AI tool.  
  - Outputs: Sends API response back to MCP trigger output (AI agent response).  
  - Edge Cases:  
    - Missing or malformed BIN code causing API errors.  
    - Invalid or missing API key.  
    - Network or service failures.  
    - Expression failure if `bin` or `api_key` are not provided by AI.  
  - Version-specific: HTTP Request Tool version 4.2 or higher.

---

### 3. Summary Table

| Node Name             | Node Type                   | Functional Role                            | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                   |
|-----------------------|-----------------------------|--------------------------------------------|----------------------|-----------------------|--------------------------------------------------------------------------------------------------------------|
| Setup Instructions    | Sticky Note                  | Setup and usage documentation               | None                 | None                  | Contains detailed setup steps, usage notes, customization tips, and support links including Discord and docs. |
| Workflow Overview     | Sticky Note                  | Workflow purpose and operation overview     | None                 | None                  | Explains workflow architecture and available BIN Lookup API operations.                                      |
| BIN Lookup MCP Server | MCP Trigger                  | AI agent webhook endpoint                    | None (trigger node)  | Check Balance, Lookup for bin |                                                                                                              |
| Sticky Note (Balance) | Sticky Note                  | Marks Balance check operation block          | None                 | None                  | Simple title "Balance".                                                                                        |
| Check Balance         | HTTP Request Tool            | Calls balance endpoint with AI-provided API key | BIN Lookup MCP Server | MCP Trigger output     |                                                                                                              |
| Sticky Note (Lookup)  | Sticky Note                  | Marks BIN lookup operation block              | None                 | None                  | Simple title "Lookup".                                                                                        |
| Lookup for bin        | HTTP Request Tool            | Calls BIN lookup endpoint with AI-provided BIN and API key | BIN Lookup MCP Server | MCP Trigger output     |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Add a Sticky Note node.  
   - Paste the setup instructions content explaining importing the workflow, no authentication, activating the workflow, copying MCP URL, connecting AI agent, usage notes, customization, and support links.  
   - Position it on the left side for visibility.

2. **Create Sticky Note: Workflow Overview**  
   - Add another Sticky Note node.  
   - Add content describing the workflow purpose (BIN Lookup API MCP server), how it works with MCP trigger and HTTP nodes, and list the two available operations (balance check and lookup).  
   - Position next to the Setup Instructions note.

3. **Create MCP Trigger Node: BIN Lookup MCP Server**  
   - Add an MCP Trigger node (type: `@n8n/n8n-nodes-langchain.mcpTrigger`).  
   - Set the webhook path to `bin-lookup-mcp`.  
   - No authentication required.  
   - Position this node centrally to connect downstream operations.

4. **Create Sticky Note: Balance**  
   - Add a Sticky Note node with content "## Balance".  
   - Position near the Balance operation nodes.

5. **Create HTTP Request Tool Node: Check Balance**  
   - Add an HTTP Request Tool node.  
   - Set URL to `https://api.bintable.com/v1/balance`.  
   - Set method to GET (default).  
   - Configure Query Parameters:  
     - `api_key` with expression `{{$fromAI('api_key', 'The API key, which you can get from bintable.com website.', 'string')}}`.  
   - Add tool description documenting the `api_key`.  
   - Connect MCP Trigger node output to this node’s input using the AI tool connection.

6. **Create Sticky Note: Lookup**  
   - Add a Sticky Note node with content "## Lookup".  
   - Position near the BIN lookup operation nodes.

7. **Create HTTP Request Tool Node: Lookup for bin**  
   - Add an HTTP Request Tool node.  
   - Set URL to `https://api.bintable.com/v1/{{ $fromAI('bin', 'pass the required BIN code', 'string') }}`.  
   - Set method to GET (default).  
   - Configure Query Parameters:  
     - `api_key` with expression `{{$fromAI('api_key', 'The API key, which you can get from bintable.com website.', 'string')}}`.  
   - Add tool description documenting `bin` path parameter and `api_key` query parameter.  
   - Connect MCP Trigger node output to this node’s input using the AI tool connection.

8. **Workflow Activation**  
   - Save the workflow.  
   - Activate the workflow to expose the MCP webhook endpoint.

9. **Credential Setup**  
   - No credentials needed inside n8n for this workflow because the `api_key` is supplied dynamically via AI expressions.  
   - Ensure your AI agent includes the `api_key` parameter in requests.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automation help, contact via Discord: https://discord.me/cfomodz      | Support channel for workflow and MCP integration questions                                               |
| Official n8n documentation for MCP and LangChain nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Detailed MCP trigger and LangChain node documentation                                                    |
| This workflow uses free API service from bintable.com with an updated BIN database maintained by community | API service source for BIN lookup and balance checking                                                   |
| Parameters are auto-populated with `$fromAI()` expressions to enable AI-driven dynamic input and seamless integration with AI agents | Important note on how parameters are passed dynamically                                                   |

---

*Disclaimer: The provided text is derived exclusively from an automated n8n workflow. All data handled is legal and publicly available. No illegal, offensive, or protected content is included.*