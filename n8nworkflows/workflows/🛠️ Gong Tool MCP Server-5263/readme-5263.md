🛠️ Gong Tool MCP Server

https://n8nworkflows.xyz/workflows/----gong-tool-mcp-server-5263


# 🛠️ Gong Tool MCP Server

### 1. Workflow Overview

The **🛠️ Gong Tool MCP Server** workflow serves as a modular API server that exposes four main operations related to "Call" and "User" resources within the Gong Tool ecosystem. It is designed to be integrated with AI agents via an MCP (Modular Connector Protocol) trigger, allowing dynamic parameter input through AI expressions and seamless error handling.

The workflow is logically divided into the following blocks:

- **1.1 MCP Trigger Input Reception**: Listens to incoming MCP requests via a webhook endpoint.
- **1.2 Call Resource Operations**: Handles retrieval of single and multiple "Call" entities from Gong Tool.
- **1.3 User Resource Operations**: Handles retrieval of single and multiple "User" entities from Gong Tool.
- **1.4 Documentation and Setup Instructions**: Provides embedded operational and setup guidance via sticky notes for users and maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Input Reception

- **Overview:**  
  This block listens for API requests routed through an MCP-compatible webhook. It acts as the entry point to trigger subsequent node operations based on incoming parameters.

- **Nodes Involved:**  
  - *Gong Tool MCP Server* (mcpTrigger)

- **Node Details:**

  - **Node Name:** Gong Tool MCP Server  
    - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - **Role:** Webhook-based trigger node that receives incoming MCP requests on a defined path.  
    - **Configuration:**  
      - Path set to `"gong-tool-mcp"` for webhook URL identification.  
      - Uses a webhook ID internally for routing requests.  
    - **Input:** External HTTP requests via webhook.  
    - **Output:** Triggers downstream nodes with parameters extracted from MCP calls, including AI-injected expressions.  
    - **Version Requirements:** Compatible with n8n versions supporting MCP triggers and webhook nodes.  
    - **Potential Failures:**  
      - Webhook misconfiguration causing request failures.  
      - Authentication or MCP protocol errors if parameters are malformed or missing.  
      - Network or server downtime affecting webhook accessibility.  
    - **Sub-Workflow:** None.

---

#### 2.2 Call Resource Operations

- **Overview:**  
  This block handles operations related to the "Call" resource in Gong Tool, including fetching a single call or multiple calls based on parameters. It dynamically receives parameters via AI expressions.

- **Nodes Involved:**  
  - Get call  
  - Get many calls  
  - Sticky Note 1 (documentation label node)

- **Node Details:**

  - **Node Name:** Get call  
    - **Type:** `n8n-nodes-base.gongTool`  
    - **Role:** Retrieves a single "Call" entity from Gong Tool based on an identifier or criteria.  
    - **Configuration:**  
      - Operation set to `"get"`.  
      - The "call" parameter is dynamically assigned using the expression:  
        `{{ $fromAI('Call', ``, 'string') }}` which extracts the "Call" parameter from the AI input.  
      - No additional options or filters configured.  
    - **Input:** Triggered from MCP node output with AI-provided parameters.  
    - **Output:** Returns the requested "Call" data or error information.  
    - **Potential Failures:**  
      - Invalid or missing "Call" identifier leading to API errors.  
      - Authentication failures with Gong Tool API.  
      - Timeouts or network errors during API call.  
      - Expression evaluation errors if AI input is malformed.  
    - **Sub-Workflow:** None.

  - **Node Name:** Get many calls  
    - **Type:** `n8n-nodes-base.gongTool`  
    - **Role:** Retrieves multiple "Call" entities, supporting limits and return-all options.  
    - **Configuration:**  
      - No explicit operation parameter (default assumed "getAll" or similar).  
      - Parameters set dynamically from AI input:  
        - `limit`: `{{ $fromAI('Limit', ``, 'number') }}`  
        - `returnAll`: `{{ $fromAI('Return_All', ``, 'boolean') }}`  
      - Filters and options are empty objects, meaning no extra filtering by default.  
    - **Input:** Triggered from MCP node with AI parameter injection.  
    - **Output:** Returns an array of "Call" entities.  
    - **Potential Failures:**  
      - Invalid limit values (e.g., negative or excessively large).  
      - API response delays or failures.  
      - Expression parsing errors.  
    - **Sub-Workflow:** None.

  - **Node Name:** Sticky Note 1  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Role:** Provides a visual label and documentation for the "Call" operations block.  
    - **Content:** "## Call" (section header).  
    - **Input/Output:** None, purely informational.

---

#### 2.3 User Resource Operations

- **Overview:**  
  This block manages retrieval of user data from Gong Tool, supporting single user fetch and bulk user queries with flexible parameters.

- **Nodes Involved:**  
  - Get user  
  - Get many users  
  - Sticky Note 2 (documentation label node)

- **Node Details:**

  - **Node Name:** Get user  
    - **Type:** `n8n-nodes-base.gongTool`  
    - **Role:** Fetches a single user by identifier or criteria.  
    - **Configuration:**  
      - Resource set explicitly to `"user"`.  
      - Operation defaults to `"get"`.  
      - `user` parameter dynamically assigned via AI expression:  
        `{{ $fromAI('User', ``, 'string') }}`  
      - No additional options or filters.  
    - **Input:** Triggered by MCP node with AI-injected parameters.  
    - **Output:** Data of the requested user or error messages.  
    - **Potential Failures:**  
      - Missing or invalid "User" ID leading to failed API requests.  
      - Authentication issues.  
      - Expression evaluation errors.  
    - **Sub-Workflow:** None.

  - **Node Name:** Get many users  
    - **Type:** `n8n-nodes-base.gongTool`  
    - **Role:** Retrieves multiple users with support for pagination and full data return.  
    - **Configuration:**  
      - Resource set to `"user"`.  
      - Operation explicitly `"getAll"`.  
      - Parameters dynamically set from AI inputs:  
        - `limit`: `{{ $fromAI('Limit', ``, 'number') }}`  
        - `returnAll`: `{{ $fromAI('Return_All', ``, 'boolean') }}`  
      - Empty filters and options indicate default unfiltered retrieval.  
    - **Input:** Triggered by MCP with AI parameters.  
    - **Output:** Array of user data or error details.  
    - **Potential Failures:**  
      - Invalid or missing limit values.  
      - API response errors.  
      - AI expression parsing issues.  
    - **Sub-Workflow:** None.

  - **Node Name:** Sticky Note 2  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Role:** Provides a visual label and documentation for the "User" operations block.  
    - **Content:** "## User" (section header).  
    - **Input/Output:** None, informational only.

---

#### 2.4 Documentation and Setup Instructions

- **Overview:**  
  This block provides detailed user guidance and setup instructions for importing, configuring, and activating the workflow, including AI integration notes.

- **Nodes Involved:**  
  - Workflow Overview 0 (sticky note)

- **Node Details:**

  - **Node Name:** Workflow Overview 0  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Role:** Comprehensive documentation node embedded inside the workflow canvas.  
    - **Content Includes:**  
      - Summary of available operations: "Call" (get, get all) and "User" (get, get all).  
      - Stepwise setup instructions:  
        1. Import workflow  
        2. Add Gong Tool credentials once in any tool node  
        3. Activate workflow  
        4. Copy webhook URL from MCP trigger  
        5. Use webhook URL in AI agent configurations  
      - Features highlight: zero configuration, AI-driven parameter population via `$fromAI()`, error handling, and default parameter customization.  
      - Support references: links to n8n documentation and Discord channel for help.  
    - **Input/Output:** None.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                                         |
|---------------------|----------------------------------|-------------------------------|------------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0  | Sticky Note                      | Setup & documentation overview| None                   | None                   | ## 🛠️ Gong Tool MCP Server<br>Available Operations: Call & User (get/get all)<br>Setup steps and helpful links included            |
| Gong Tool MCP Server | MCP Trigger                     | Entry webhook trigger for MCP | None                   | Get call, Get many calls, Get user, Get many users |                                                                                                                                     |
| Get call            | Gong Tool Node                  | Fetch single Call resource     | Gong Tool MCP Server   | None                   |                                                                                                                                     |
| Get many calls      | Gong Tool Node                  | Fetch multiple Call resources  | Gong Tool MCP Server   | None                   |                                                                                                                                     |
| Sticky Note 1       | Sticky Note                    | Label for Call operations      | None                   | None                   | ## Call                                                                                                                             |
| Get user            | Gong Tool Node                  | Fetch single User resource     | Gong Tool MCP Server   | None                   |                                                                                                                                     |
| Get many users      | Gong Tool Node                  | Fetch multiple User resources  | Gong Tool MCP Server   | None                   |                                                                                                                                     |
| Sticky Note 2       | Sticky Note                    | Label for User operations      | None                   | None                   | ## User                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an MCP Trigger node:**
   - Name: `Gong Tool MCP Server`
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`
   - Set the webhook path to `"gong-tool-mcp"`.
   - This node will serve as the API entry point.

2. **Create a Gong Tool node for fetching a single Call:**
   - Name: `Get call`
   - Type: `n8n-nodes-base.gongTool`
   - Set operation to `"get"`.
   - Set the parameter `call` to the expression:  
     `{{ $fromAI('Call', ``, 'string') }}`
   - Connect this node’s input to the MCP Trigger node output.
   - Configure Gong Tool API credentials once here (reused by other Gong Tool nodes).

3. **Create a Gong Tool node for fetching multiple Calls:**
   - Name: `Get many calls`
   - Type: `n8n-nodes-base.gongTool`
   - Leave operation default or set to "getAll" if required by the node.
   - Set parameters using expressions:  
     - `limit`: `{{ $fromAI('Limit', ``, 'number') }}`  
     - `returnAll`: `{{ $fromAI('Return_All', ``, 'boolean') }}`
   - Connect input from the MCP Trigger node.
   - Use the same Gong Tool credentials.

4. **Create a Gong Tool node for fetching a single User:**
   - Name: `Get user`
   - Type: `n8n-nodes-base.gongTool`
   - Set resource to `"user"` and operation to `"get"`.
   - Set `user` parameter to:  
     `{{ $fromAI('User', ``, 'string') }}`
   - Connect input from the MCP Trigger node.
   - Use the same Gong Tool credentials.

5. **Create a Gong Tool node for fetching multiple Users:**
   - Name: `Get many users`
   - Type: `n8n-nodes-base.gongTool`
   - Set resource to `"user"` and operation to `"getAll"`.
   - Set parameters using expressions:  
     - `limit`: `{{ $fromAI('Limit', ``, 'number') }}`  
     - `returnAll`: `{{ $fromAI('Return_All', ``, 'boolean') }}`
   - Connect input from the MCP Trigger node.
   - Use the same Gong Tool credentials.

6. **Add sticky notes for documentation and block labeling:**
   - Create a sticky note named `Workflow Overview 0` containing the detailed multi-step setup instructions, operation summaries, and support links.
   - Create a sticky note named `Sticky Note 1` with the text `"## Call"`, placed near Call-related nodes.
   - Create a sticky note named `Sticky Note 2` with the text `"## User"`, placed near User-related nodes.

7. **Credential Setup:**
   - Configure Gong Tool credentials in the first Gong Tool node (e.g., `Get call`).
   - Once configured, open and close other Gong Tool nodes to inherit the credentials.
   - Ensure credentials have required API permissions and are valid.

8. **Activation and Testing:**
   - Activate the workflow.
   - Copy the webhook URL generated by the MCP Trigger node (`/webhook/gong-tool-mcp` endpoint).
   - Use this URL in your AI agent or API client to send MCP calls with parameters such as `Call`, `User`, `Limit`, and `Return_All`.
   - Verify responses for correctness and handle errors as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| For MCP integration guidance or customizations, reach out on Discord at https://discord.me/cfomodz                  | Support channel for n8n MCP and Gong Tool integration                                                              |
| Official n8n documentation on MCP and LangChain tool nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Reference for MCP nodes and usage                                                                                   |
| AI parameter injection is facilitated by `$fromAI()` expressions allowing dynamic population of node parameters    | Enables seamless AI-driven workflow parameterization                                                               |
| Zero-code setup with native n8n error handling ensures robustness and ease of customization                         | Designed for maintainers and integrators to adapt without deep coding                                               |

---

**Disclaimer:** The provided text originates exclusively from a workflow automated using n8n, an integration and automation platform. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.