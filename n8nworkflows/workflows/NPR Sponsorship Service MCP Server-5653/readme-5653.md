NPR Sponsorship Service MCP Server

https://n8nworkflows.xyz/workflows/npr-sponsorship-service-mcp-server-5653


# NPR Sponsorship Service MCP Server

### 1. Workflow Overview

The **NPR Sponsorship Service MCP Server** workflow serves as an integration layer that exposes the NPR Sponsorship Service API through an MCP (Model-Controller-Processor) compatible interface, enabling AI agents to interact with sponsorship functionalities designed for non-NPR One client applications. The workflow is structured to handle two primary operations:

- Fetching VAST sponsorship units
- Tracking VAST sponsorship data

Logical blocks within this workflow include:

- **1.1 Setup and Documentation:** Contains sticky notes providing setup instructions and a workflow overview for users.
- **1.2 MCP Trigger:** Initiates the workflow by receiving requests from AI agents using the MCP protocol.
- **1.3 Sponsorship Operations:** Contains HTTP Request nodes that interface with the NPR Sponsorship API endpoints to fetch sponsorship data or track sponsorship events.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Setup and Documentation

- **Overview:** This block provides detailed instructions and explanations for users to import, configure, and operate the workflow effectively. It also includes contextual information about the workflow purpose and usage.
- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  
  - Sponsorship (Sticky Note)

##### Node: Setup Instructions

- **Type:** Sticky Note  
- **Role:** Document setup steps, usage notes, and customization tips for the workflow user.  
- **Key Content Highlights:**  
  - How to import and activate the workflow  
  - Setting up OAuth2 credentials (generic authentication)  
  - Retrieving the MCP webhook URL for AI agent integration  
  - Usage notes on parameter auto-population with `$fromAI()` expressions  
  - Recommendations for error handling, logging, and customization  
  - Support links: Discord community and official n8n documentation  
- **Connections:** None (informational only)  
- **Potential Issues:** None (documentation node)

##### Node: Workflow Overview

- **Type:** Sticky Note  
- **Role:** Provides a concise description of the workflow’s purpose, its operations, and how it functions internally.  
- **Key Content Highlights:**  
  - Sponsorship service for non-NPR One client apps  
  - MCP Trigger as the server endpoint  
  - HTTP Request nodes for API interaction  
  - AI expressions for dynamic parameter handling  
  - Two operations supported: Fetch VAST Sponsorships, Track VAST Sponsorship Data  
- **Connections:** None (informational only)  
- **Potential Issues:** None

##### Node: Sponsorship (Sticky Note)

- **Type:** Sticky Note  
- **Role:** Labels the Sponsorship operations block  
- **Connections:** None  
- **Potential Issues:** None

---

#### Block 1.2: MCP Trigger

- **Overview:** This node acts as the entry point for AI agents to invoke the workflow. It exposes an HTTP endpoint that accepts MCP-formatted requests and routes them through the workflow.
- **Node Involved:**  
  - NPR Sponsorship Service MCP Server (MCP Trigger)
  
##### Node: NPR Sponsorship Service MCP Server

- **Type:** MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
- **Role:** Starts the workflow by receiving MCP requests at a specified path (`npr-sponsorship-service-mcp`).  
- **Configuration:**  
  - Webhook Path: `npr-sponsorship-service-mcp`  
  - Webhook ID: Unique identifier for the webhook  
- **Input:** HTTP requests from AI agents formatted as MCP calls  
- **Output:** Routes requests to subsequent nodes depending on the operation invoked  
- **Version Requirements:** Compatible with n8n version supporting MCP nodes (`@n8n/n8n-nodes-langchain` module)  
- **Edge Cases / Failures:**  
  - Webhook not reachable if the workflow is inactive or misconfigured  
  - Authentication or authorization errors if external security is enforced outside MCP  
  - Malformed MCP requests may cause parsing errors  
- **Sub-workflow:** N/A

---

#### Block 1.3: Sponsorship Operations

- **Overview:** This block performs the core functional operations by interfacing with the NPR Sponsorship API. It includes two HTTP Request nodes that respectively fetch sponsorship data and track sponsorship events.
- **Nodes Involved:**  
  - Fetch VAST Sponsorships (HTTP Request Tool)  
  - Track VAST Sponsorship Data (HTTP Request Tool)  

##### Node: Fetch VAST Sponsorships

- **Type:** HTTP Request Tool (`n8n-nodes-base.httpRequestTool`)  
- **Role:** Sends a GET request to `https://sponsorship.api.npr.org/v2/ads` to retrieve VAST sponsorship units.  
- **Configuration:**  
  - URL: Dynamic, set to `https://sponsorship.api.npr.org/v2/ads`  
  - Method: GET (default)  
  - Authentication: Generic HTTP Header authentication (configured externally)  
  - Query Parameters:  
    - `forceResult`: Boolean, optionally forces synchronous call to external provider  
    - `adCount`: Number, specifies how many sponsorship units to request  
  - Parameters are dynamically populated using `$fromAI()` expressions, allowing AI agents to specify values at runtime.  
- **Input:** MCP Trigger node output based on the operation requested  
- **Output:** Returns sponsorship data in the original API response format to the AI agent  
- **Version Requirements:** HTTP Request Tool node version 4.2 or higher recommended  
- **Edge Cases / Failures:**  
  - Authentication failures if credentials are misconfigured  
  - Network timeouts or API server errors  
  - Invalid parameter types could cause API errors  
  - Empty or malformed API responses  
- **Sub-workflow:** N/A

##### Node: Track VAST Sponsorship Data

- **Type:** HTTP Request Tool (`n8n-nodes-base.httpRequestTool`)  
- **Role:** Sends a POST request to `https://sponsorship.api.npr.org/v2/ads` to record tracking data for sponsorship units.  
- **Configuration:**  
  - URL: `https://sponsorship.api.npr.org/v2/ads`  
  - Method: POST  
  - Authentication: Generic HTTP Header authentication (configured externally)  
  - Request Body: Expected to be provided by the MCP request or AI parameters (not explicitly defined in node, relies on input data)  
- **Input:** MCP Trigger node output based on the operation requested  
- **Output:** Returns API response confirming tracking data recording  
- **Version Requirements:** HTTP Request Tool node version 4.2 or higher  
- **Edge Cases / Failures:**  
  - Missing or invalid tracking data in request body  
  - Authentication failures  
  - API endpoint errors or rejection of malformed data  
- **Sub-workflow:** N/A

---

### 3. Summary Table

| Node Name                       | Node Type                  | Functional Role                        | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                                                                  |
|--------------------------------|----------------------------|-------------------------------------|---------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Setup Instructions             | Sticky Note                | Setup and usage documentation        | None                            | None                          | Contains detailed setup steps, usage notes, customization advice, and support links (Discord, n8n docs)      |
| Workflow Overview             | Sticky Note                | Workflow purpose and operation summary | None                            | None                          | Describes workflow features, supported operations, and API integration approach                              |
| Sponsorship                  | Sticky Note                | Labels Sponsorship operation block    | None                            | None                          |                                                                                                              |
| NPR Sponsorship Service MCP Server | MCP Trigger               | Entry point for AI agent requests    | None                            | Fetch VAST Sponsorships, Track VAST Sponsorship Data |                                                                                                              |
| Fetch VAST Sponsorships         | HTTP Request Tool          | Fetch sponsorship units from API     | NPR Sponsorship Service MCP Server | Returns API response to MCP   | Parameters auto-populated from AI using `$fromAI()` expressions                                             |
| Track VAST Sponsorship Data     | HTTP Request Tool          | Submit tracking data for sponsorship | NPR Sponsorship Service MCP Server | Returns API response to MCP   |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Setup Instructions**  
   - Node Type: Sticky Note  
   - Content: Include detailed setup steps, usage notes, customization tips, and support links as per the original.  
   - Position: Set to coordinate similar to original (e.g., left side).  

2. **Create Sticky Note: Workflow Overview**  
   - Node Type: Sticky Note  
   - Content: Describe workflow purpose, MCP trigger role, API endpoints, operations supported, and AI parameter handling.  
   - Position: Adjacent to Setup Instructions.  

3. **Create Sticky Note: Sponsorship**  
   - Node Type: Sticky Note  
   - Content: Title label for Sponsorship operations block.  
   - Position: Near Sponsorship HTTP nodes.  

4. **Add MCP Trigger Node: NPR Sponsorship Service MCP Server**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `npr-sponsorship-service-mcp`  
     - Ensure webhook is active and accessible  
   - Position: Center-left of workspace.  

5. **Add HTTP Request Tool Node: Fetch VAST Sponsorships**  
   - Node Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://sponsorship.api.npr.org/v2/ads`  
     - Method: GET (default)  
     - Authentication: Generic HTTP Header Auth (use OAuth2 or API key as per NPR Sponsorship API requirements)  
     - Query Parameters:  
       - `forceResult`: Set expression to `{{$fromAI('forceResult', 'Whether to force a synchronous call to our external sponsorship provider; the default behavior is asynchronous.', 'boolean')}}`  
       - `adCount`: Set expression to `{{$fromAI('adCount', 'How many sponsorship units to request in one call; if left unspecified, the default behavior is to return only 1.', 'number')}}`  
   - Position: Right of MCP Trigger node.  

6. **Add HTTP Request Tool Node: Track VAST Sponsorship Data**  
   - Node Type: HTTP Request Tool  
   - Parameters:  
     - URL: `https://sponsorship.api.npr.org/v2/ads`  
     - Method: POST  
     - Authentication: Same as above  
     - Body: Expect to receive tracking data from MCP request (configure to accept raw or JSON body forwarded by MCP trigger).  
   - Position: Below "Fetch VAST Sponsorships" node.  

7. **Connect Nodes:**  
   - Connect MCP Trigger node’s output to both "Fetch VAST Sponsorships" and "Track VAST Sponsorship Data" nodes via the `ai_tool` output (or equivalent), enabling routing based on operation requested by the AI agent.  

8. **Credentials Setup:**  
   - Configure Generic HTTP Header Authentication Credentials for NPR Sponsorship API calls. This usually involves OAuth2 or API Key authentication setup within n8n credentials manager.  
   - Assign credentials to both HTTP Request Tool nodes.  

9. **Activate Workflow:**  
   - Save and activate the workflow.  
   - Confirm the MCP Trigger webhook URL is accessible and note it for AI agent integration.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Use `$fromAI()` expressions in HTTP Request nodes to dynamically populate parameters based on AI agent input.                                                | n8n MCP integration with AI agents                                                                    |
| For integration help or custom automations, join the Discord community: https://discord.me/cfomodz                                                             | Community support                                                                                      |
| Official n8n documentation for MCP and Langchain integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/   | n8n documentation                                                                                      |
| Consider adding data transformation, error handling, and logging nodes to extend functionality and robustness.                                                | Workflow customization suggestions                                                                    |
| The workflow is designed to maintain the original API response structure to preserve compatibility with existing AI agent expectations.                     | API response integrity                                                                                 |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and publicly accessible.