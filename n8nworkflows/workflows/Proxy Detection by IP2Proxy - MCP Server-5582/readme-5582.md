Proxy Detection by IP2Proxy - MCP Server

https://n8nworkflows.xyz/workflows/proxy-detection-by-ip2proxy---mcp-server-5582


# Proxy Detection by IP2Proxy - MCP Server

### 1. Workflow Overview

This workflow, titled **"IP2Proxy Proxy Detection MCP Server"**, serves as a dedicated MCP (Modular Component Processor) server endpoint to detect proxy usage and threat levels for given IP addresses using the IP2Proxy API. It is designed for integration with AI agents, providing real-time proxy detection via a single API operation.  

The workflow is logically divided into the following blocks:

- **1.1 Setup & Documentation**: Contains detailed setup instructions and a comprehensive workflow overview to guide users in deployment and usage.

- **1.2 MCP Server Trigger**: The core entry point node acting as the MCP endpoint, receiving AI agent requests.

- **1.3 Proxy Detection API Call**: Executes the HTTP request to the IP2Proxy API with parameters dynamically populated from AI agent input.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Documentation

**Overview:**  
This block provides users with setup instructions, usage notes, customization tips, and a general overview of the workflow’s purpose and functionality.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- Sticky Note (Empty, placeholder)

**Node Details:**  

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Documentation for installation and configuration  
  - Configuration: Detailed markdown instructions including import steps, authentication notes (none required), activation instructions, MCP URL retrieval, AI agent connection, parameter usage, and support contact links (Discord and n8n docs).  
  - Inputs: None  
  - Outputs: None  
  - Edge Cases: None (informational node)  

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Provides a summary of IP2Proxy service capabilities, workflow logic, and available operations (single endpoint)  
  - Configuration: Markdown content explaining IP2Proxy features, MCP trigger role, HTTP request usage, AI expressions, and response handling.  
  - Inputs: None  
  - Outputs: None  
  - Edge Cases: None (informational node)  

- **Sticky Note (Empty)**  
  - Type: Sticky Note  
  - Role: Placeholder or formatting node with no content  
  - Inputs: None  
  - Outputs: None  

---

#### 1.2 MCP Server Trigger

**Overview:**  
This node serves as the workflow’s external interface, exposing a webhook endpoint that AI agents call to initiate proxy detection requests.

**Nodes Involved:**  
- IP2Proxy Proxy Detection MCP Server (MCP Trigger)

**Node Details:**  

- **IP2Proxy Proxy Detection MCP Server**  
  - Type: MCP Trigger (from Langchain package)  
  - Role: Listens for incoming MCP requests at a defined path, acting as an API endpoint for AI agents  
  - Configuration:  
    - Webhook path set to `ip2proxy-proxy-detection-mcp`  
    - No authentication required  
  - Key Expressions / Variables: None internally; parameters extracted and passed to downstream nodes via AI expressions  
  - Input Connections: None (trigger node)  
  - Output Connections: Connects to "Check Proxy IP" node via `ai_tool` connection  
  - Version Requirements: Requires n8n version supporting MCP nodes (Langchain integration)  
  - Edge Cases:  
    - Network errors or webhook unavailability  
    - Invalid or malformed incoming requests could cause downstream failures  
    - MCP-specific version mismatches or deprecated features  
  - Sub-workflow: None  

---

#### 1.3 Proxy Detection API Call

**Overview:**  
This block performs the actual detection logic by calling the IP2Proxy API endpoint with parameters supplied by the AI agent via MCP trigger.

**Nodes Involved:**  
- Check Proxy IP (HTTP Request Tool)

**Node Details:**  

- **Check Proxy IP**  
  - Type: HTTP Request Tool (specialized HTTP node integrated with AI tools)  
  - Role: Executes HTTP GET request to IP2Proxy API to check if an IP is a proxy and retrieve threat details  
  - Configuration:  
    - URL: `https://api.ip2proxy.com/`  
    - Query Parameters dynamically set via `$fromAI()` expressions to extract from incoming MCP request:  
      - `package`: Optional, string, e.g., PX1 to PX11  
      - `ip`: Required, IPv4 address to check  
      - `format`: Optional, default is JSON  
      - `key`: Required, API key from IP2Location signup  
    - HTTP method: GET  
    - Sends query parameters in URL  
    - Tool description included for user clarity within n8n  
  - Key Expressions / Variables:  
    - `$fromAI('package', ...)`  
    - `$fromAI('ip', ...)`  
    - `$fromAI('format', ...)`  
    - `$fromAI('key', ...)`  
  - Input Connections: Receives input from MCP trigger node as `ai_tool`  
  - Output Connections: None (response returns directly to AI agent via MCP mechanism)  
  - Version Requirements: Compatible with n8n versions supporting HTTP Request Tool node version 4.2+  
  - Edge Cases / Failure Types:  
    - Missing or invalid API key causing authentication errors  
    - Invalid IP format causing API errors  
    - API rate limiting or network timeouts  
    - Missing required parameters leads to incomplete or failed requests  
    - Response parsing errors if `format` parameter is unsupported or malformed  
  - Sub-workflow: None  

---

### 3. Summary Table

| Node Name                          | Node Type                   | Functional Role                            | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                                          |
|----------------------------------|-----------------------------|--------------------------------------------|----------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Setup Instructions               | Sticky Note                 | Setup and usage instructions                | None                             | None                          | ### ⚙️ Setup Instructions 1. Import Workflow 2. Authentication: None 3. Activate Workflow 4. Get MCP URL 5. Connect AI Agent ● Parameters auto-populated via $fromAI() ● Responses keep API structure ● Customize error handling, logging, transforms ● Contact via Discord https://discord.me/cfomodz ● Docs https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ |
| Workflow Overview               | Sticky Note                 | Overview of workflow purpose and operations | None                             | None                          | ## 🛠️ IP2Proxy Proxy Detection MCP Server ✅ 1 operations About IP2Proxy service and MCP server usage Details of API and AI expression usage |
| Sticky Note                    | Sticky Note                 | Empty placeholder                           | None                             | None                          |                                                                                                                      |
| IP2Proxy Proxy Detection MCP Server | MCP Trigger (Langchain)     | Entry point MCP webhook trigger             | None                             | Check Proxy IP                |                                                                                                                      |
| Check Proxy IP                 | HTTP Request Tool           | Calls IP2Proxy API with AI parameters       | IP2Proxy Proxy Detection MCP Server | None                        | Check if IP is proxy via https://api.ip2proxy.com/ Parameters: package, ip, format, key                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note:**  
   - Name: `Setup Instructions`  
   - Paste detailed setup instructions as markdown (use content from the original sticky note)  
   - Position appropriately for documentation  

2. **Create Sticky Note:**  
   - Name: `Workflow Overview`  
   - Paste the workflow overview markdown describing IP2Proxy features and workflow logic  

3. **Create Sticky Note:**  
   - Name: `Sticky Note` (optional placeholder)  
   - Leave content empty or use as visual spacer  

4. **Add MCP Trigger Node:**  
   - Name: `IP2Proxy Proxy Detection MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Set `path` to `ip2proxy-proxy-detection-mcp`  
   - No authentication needed  
   - This node will listen for incoming MCP requests at URL: `https://<your-n8n-instance>/webhook/ip2proxy-proxy-detection-mcp`  

5. **Add HTTP Request Tool Node:**  
   - Name: `Check Proxy IP`  
   - Type: `HTTP Request Tool` (ensure version 4.2 or higher)  
   - Parameters:  
     - URL: `https://api.ip2proxy.com/`  
     - Method: GET  
     - Enable sending query parameters  
     - Add query parameters with values using expressions:  
       - `package`: `{{$fromAI('package', 'Package name from PX1 to PX11. If not present, the web service will assume the PX1 package query.', 'string')}}`  
       - `ip`: `{{$fromAI('ip', 'IP address (IPv4) for lookup purpose. If not present, the server IP address will be used for the lookup.', 'string')}}`  
       - `format`: `{{$fromAI('format', 'If not present, json format will be returned as the search result.', 'string')}}`  
       - `key`: `{{$fromAI('key', 'API key. Please sign up free trial license key at ip2location.com', 'string')}}`  
     - Include the tool description as a comment for clarity  

6. **Connect Nodes:**  
   - Connect the output of `IP2Proxy Proxy Detection MCP Server` node to the `Check Proxy IP` node using the connection type `ai_tool`  

7. **Activate Workflow:**  
   - Save and activate the workflow to enable the MCP server endpoint  

8. **Usage:**  
   - Copy the webhook URL from the MCP trigger node  
   - Configure your AI agent to send requests to this URL including parameters `package`, `ip`, `format`, and `key` as needed  
   - The workflow will process requests and return proxy detection responses transparently  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                              |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Use free trial API keys by signing up at https://www.ip2location.com/web-service/ip2proxy           | IP2Proxy API key registration                                                                                                 |
| MCP servers in n8n provide seamless AI agent integrations by exposing webhook endpoints             | n8n Langchain MCP Trigger documentation                                                                                       |
| For setup or customization help, contact via Discord: https://discord.me/cfomodz                    | Community support and integration guidance                                                                                    |
| Official n8n documentation on MCP nodes and Langchain tools: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | n8n documentation for MCP workflows                                                                                           |

---

**Disclaimer:**  
This documentation is generated exclusively from an automated n8n workflow export. All content complies with applicable content policies and contains no illegal or protected data.