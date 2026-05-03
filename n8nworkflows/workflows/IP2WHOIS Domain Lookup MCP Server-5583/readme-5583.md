IP2WHOIS Domain Lookup MCP Server

https://n8nworkflows.xyz/workflows/ip2whois-domain-lookup-mcp-server-5583


# IP2WHOIS Domain Lookup MCP Server

---

### 1. Workflow Overview

This workflow, titled **IP2WHOIS Domain Lookup MCP Server**, is designed to provide an MCP (Modular Conversational Platform) compatible API endpoint that allows AI agents to query WHOIS information for domain names using the IP2WHOIS API. It simplifies domain ownership and registrar data retrieval by exposing a single operation through an n8n MCP trigger node.

The workflow is logically grouped into the following blocks:

- **1.1 Setup & Documentation**: Contains sticky notes to guide users on setup, usage, and customization.
- **1.2 MCP Trigger Endpoint**: The entry point that listens for AI agent requests via a webhook.
- **1.3 WHOIS Data Lookup Operation**: Executes the HTTP request to IP2WHOIS API using parameters automatically populated by AI expressions.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Documentation

- **Overview:** Provides detailed setup instructions, usage notes, customization tips, and workflow overview to assist users in deploying and understanding the workflow.
- **Nodes Involved:**  
  - `Setup Instructions` (Sticky Note)  
  - `Workflow Overview` (Sticky Note)  
  - `Sticky Note` (Empty/placeholder)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Documentation and user guidance  
    - Configuration: Contains stepwise setup instructions covering import, authentication (API key), activation, MCP URL retrieval, AI agent connection, usage notes about parameter auto-population with `$fromAI()`, and customization ideas such as error handling and logging. Includes contact and documentation links.  
    - Connections: None (standalone)  
    - Edge Cases: None (documentation only)

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: High-level explanation of workflow purpose and available operations  
    - Configuration: Describes the IP2WHOIS service, how the workflow converts the API into an MCP interface, describes the MCP trigger node, HTTP request usage, AI expressions for parameters, and response forwarding. Lists the single available operation: "Lookup WHOIS Data".  
    - Connections: None (standalone)  
    - Edge Cases: None (documentation only)

  - **Sticky Note** (Unnamed)  
    - Type: Sticky Note  
    - Role: Placeholder or empty note with minimal content  
    - Configuration: Empty content with no instructions or comments  
    - Connections: None  
    - Edge Cases: None

---

#### 2.2 MCP Trigger Endpoint

- **Overview:** Serves as the webhook entry point that receives requests from AI agents in MCP format. This node exposes the server URL that AI agents use to send WHOIS lookup queries.
- **Nodes Involved:**  
  - `IP2WHOIS Domain Lookup MCP Server` (MCP Trigger)

- **Node Details:**

  - **IP2WHOIS Domain Lookup MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Listens for incoming MCP requests and triggers workflow execution  
    - Configuration:  
      - Webhook path set to `"ip2whois-domain-lookup-mcp"`  
      - Serves as the unique public endpoint for AI agent integration  
    - Inputs: None (trigger node)  
    - Outputs: Connected to the HTTP request node performing the WHOIS lookup  
    - Version Specifics: Requires n8n with Langchain MCP node support (version compatible with MCP trigger)  
    - Edge Cases:  
      - Webhook URL must be publicly accessible  
      - Authentication is handled downstream; this node does not validate API keys itself  
      - Potential timeout or connectivity issues if AI agents send malformed or slow requests

---

#### 2.3 WHOIS Data Lookup Operation

- **Overview:** Performs the actual API call to IP2WHOIS to fetch domain WHOIS data. Parameters are dynamically populated from the MCP request via AI expressions.
- **Nodes Involved:**  
  - `Lookup WHOIS Data` (HTTP Request Tool)

- **Node Details:**

  - **Lookup WHOIS Data**  
    - Type: `n8n-nodes-base.httpRequestTool`  
    - Role: Sends HTTP GET requests to IP2WHOIS API endpoint to retrieve domain WHOIS information  
    - Configuration:  
      - Base URL: `https://api.ip2whois.com/v2/`  
      - Query parameters:  
        - `domain` (required): Populated with `$fromAI('domain', 'Domain name for lookup purpose.', 'string')`  
        - `key` (required): Populated with `$fromAI('key', 'API key.', 'string')`  
        - `format` (optional): Populated with `$fromAI('format', 'Returns the API response in json (default) or xml format.', 'string')`  
      - Method: GET (default for query parameters)  
      - Tool description clarifies the parameter roles  
    - Inputs: From MCP trigger node (`IP2WHOIS Domain Lookup MCP Server`)  
    - Outputs: Returns the raw API response directly to the AI agent via MCP response mechanism  
    - Version Specifics: Uses HTTP Request Tool node version 4.2  
    - Edge Cases:  
      - Missing or invalid API key leads to authentication errors from IP2WHOIS  
      - Invalid or malformed domain names result in API errors or empty data  
      - Unsupported format values may cause format errors  
      - Network issues or API downtime can cause request failures or timeouts

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                      | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                                   |
|--------------------------------|----------------------------------|------------------------------------|--------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| Setup Instructions             | Sticky Note                      | Setup instructions and documentation | None                           | None                         | Provides setup steps, authentication notes, usage, customization tips, and support links                       |
| Workflow Overview             | Sticky Note                      | Workflow purpose and operation overview | None                           | None                         | Explains IP2WHOIS service, MCP trigger role, API integration, and available operations                        |
| Sticky Note                   | Sticky Note                      | Placeholder / empty note            | None                           | None                         |                                                                                                               |
| IP2WHOIS Domain Lookup MCP Server | MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger) | Entry point webhook for AI agent MCP requests | None                           | Lookup WHOIS Data            |                                                                                                               |
| Lookup WHOIS Data             | HTTP Request Tool (n8n-nodes-base.httpRequestTool) | Performs WHOIS API call to IP2WHOIS | IP2WHOIS Domain Lookup MCP Server | None (returns response to MCP) | Lookup WHOIS information with parameters populated from AI expressions                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note for Setup Instructions**  
   - Add a Sticky Note node named `Setup Instructions`  
   - Paste the detailed setup instructions including import, authentication, activation, MCP URL retrieval, AI agent integration, usage notes, customization tips, and support links.  
   - Position it visually separate for user guidance.

2. **Create Sticky Note for Workflow Overview**  
   - Add a Sticky Note node named `Workflow Overview`  
   - Add content describing the IP2WHOIS service, the workflow’s MCP conversion, the single operation supported, and usage notes about AI expressions and response handling.  
   - Position near `Setup Instructions`.

3. **Create Empty Sticky Note**  
   - Add a Sticky Note node named `Sticky Note` with empty or placeholder content.  
   - Optional for spacing or future notes.

4. **Add MCP Trigger Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.mcpTrigger` and name it `IP2WHOIS Domain Lookup MCP Server`  
   - Set the `path` parameter to `"ip2whois-domain-lookup-mcp"` to define the webhook endpoint.  
   - No credentials are required here; the node acts as a webhook listener.

5. **Add HTTP Request Tool Node**  
   - Add a node of type `HTTP Request Tool (n8n-nodes-base.httpRequestTool)` and name it `Lookup WHOIS Data`  
   - Configure the URL as `https://api.ip2whois.com/v2/`  
   - Method: GET (default)  
   - Under query parameters, add:  
     - `domain`: Set value to `={{ $fromAI('domain', 'Domain name for lookup purpose.', 'string') }}`  
     - `key`: Set value to `={{ $fromAI('key', 'API key.', 'string') }}`  
     - `format`: Set value to `={{ $fromAI('format', 'Returns the API response in json (default) or xml format.', 'string') }}`  
   - Tool description: Include the API parameter explanation for clarity.

6. **Connect Nodes**  
   - Connect the output of `IP2WHOIS Domain Lookup MCP Server` (MCP trigger) to the input of `Lookup WHOIS Data` node.

7. **Workflow Activation**  
   - Save and activate the workflow in n8n.  
   - Ensure the n8n instance is publicly accessible or properly tunneled to expose the webhook URL.

8. **Credential Setup**  
   - No explicit credential node is required because the API key is passed dynamically from the AI agent request via `$fromAI()` expression.  
   - Ensure AI agent includes a valid IP2WHOIS API key in requests.

9. **Testing**  
   - Retrieve the webhook URL from the MCP trigger node (e.g., `https://your-n8n-instance/webhook/ip2whois-domain-lookup-mcp`)  
   - Use this URL as the MCP endpoint in your AI agent setup.  
   - Send test requests with domain and key parameters to verify WHOIS data retrieval.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| For integration help or custom automations, contact on Discord at https://discord.me/cfomodz                                                           | Support channel mentioned in Setup Instructions sticky note                                                             |
| Refer to official n8n documentation on MCP Trigger node and HTTP Request Tool for advanced usage and parameter handling                              | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                            |
| Workflow handles one operation: domain WHOIS lookup via IP2WHOIS API, returning raw API data to AI agents without transformation                      | Workflow Overview sticky note                                                                                            |
| Parameters are auto-populated using `$fromAI()` expressions ensuring seamless AI agent parameter injection                                              | Setup Instructions and Workflow Overview notes                                                                           |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. All processing complies with applicable content policies and handles only legal and public data.

---