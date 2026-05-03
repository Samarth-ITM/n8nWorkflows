Get, Create, Upadte Profiles 🛠️ Humantic AI Tool MCP Server

https://n8nworkflows.xyz/workflows/get--create--upadte-profiles-----humantic-ai-tool-mcp-server-5240


# Get, Create, Upadte Profiles 🛠️ Humantic AI Tool MCP Server

---

### 1. Workflow Overview

This workflow, titled **"Humantic AI Tool MCP Server"**, is designed to serve as a minimal MCP (Model Control Plane) server that integrates with the Humantic AI Tool to manage user profiles. It exposes three primary operations — **Create**, **Get**, and **Update** a profile — that can be triggered via an MCP webhook. The workflow is intended for use cases where AI agents require programmatic access to Humantic AI profile management, such as automating recruitment, sales, or personalized communication workflows.

The workflow logic is structured into the following logical blocks:

- **1.1 MCP Server Initialization:** The workflow listens for incoming MCP requests via a webhook trigger node. This node acts as the entry point for all profile-related operations.
- **1.2 Profile Operations:** Depending on the AI agent’s request, one of three Humantic AI Tool nodes is executed to create, get, or update a user profile.
- **1.3 Workflow Metadata and Guidance:** Sticky notes provide operational instructions and contextual explanations for users configuring or maintaining this workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Initialization

- **Overview:**  
  This block handles incoming MCP requests by exposing a webhook endpoint. It serves as the gateway for AI agents to invoke profile operations.

- **Nodes Involved:**  
  - Humantic AI Tool MCP Server (MCP Trigger)

- **Node Details:**

  - **Node Name:** Humantic AI Tool MCP Server  
    - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP Trigger)  
    - **Technical Role:** Acts as the webhook trigger for MCP server requests, listening on a specified path.  
    - **Configuration:**  
      - Webhook path set to `"humantic-ai-tool-mcp"`  
      - Automatically generates the webhook URL used by AI agents.  
    - **Key Expressions/Variables:** None directly, but it passes received data downstream.  
    - **Input:** No input nodes; this is a trigger node.  
    - **Output:** Connects to all three profile operation nodes through the `ai_tool` output.  
    - **Version Requirements:** Requires n8n version supporting MCP triggers (generally version 1.95+).  
    - **Potential Failures:**  
      - Webhook URL misconfiguration or conflicts  
      - Network or authentication issues if secured externally  
    - **Sub-workflow Reference:** None  

#### 1.2 Profile Operations

- **Overview:**  
  Contains three Humantic AI Tool nodes that perform profile management operations — Create, Get, and Update — based on parameters supplied by AI agents via MCP.

- **Nodes Involved:**  
  - Create a profile  
  - Get a profile  
  - Update a profile  

- **Node Details:**

  - **Node Name:** Create a profile  
    - **Type:** `n8n-nodes-base.humanticAiTool`  
    - **Technical Role:** Calls Humantic AI API to create a new user profile.  
    - **Configuration:**  
      - `userId` sourced dynamically from MCP input via `$fromAI('User_Id', '', 'string')`.  
      - `sendResume` boolean parameter from AI input.  
      - `binaryPropertyName` string parameter from AI input.  
      - Operation defaults to create (implicit).  
    - **Input:** Data received from MCP trigger node.  
    - **Output:** Profile creation response sent back to MCP client.  
    - **Credentials:** Requires Humantic AI API credentials configured once and referenced here.  
    - **Potential Failures:**  
      - Invalid or missing `userId`  
      - API authentication failure  
      - Network timeouts or rate limits  
      - Malformed AI input expressions causing evaluation errors  
    - **Sub-workflow Reference:** None  

  - **Node Name:** Get a profile  
    - **Type:** `n8n-nodes-base.humanticAiTool`  
    - **Technical Role:** Retrieves an existing user profile from Humantic AI by `userId`.  
    - **Configuration:**  
      - `userId` dynamically from `$fromAI('User_Id', '', 'string')`.  
      - Operation explicitly set to `"get"`.  
      - Options object left empty (default).  
    - **Input:** Receives data from MCP trigger node.  
    - **Output:** Returns profile data or error to MCP client.  
    - **Credentials:** Uses same Humantic AI API credentials.  
    - **Potential Failures:**  
      - User ID not found or invalid  
      - API authentication or permission errors  
      - Expression evaluation failures on `userId`  
    - **Sub-workflow Reference:** None  

  - **Node Name:** Update a profile  
    - **Type:** `n8n-nodes-base.humanticAiTool`  
    - **Technical Role:** Updates an existing user profile with new data.  
    - **Configuration:**  
      - `userId` from `$fromAI('User_Id', '', 'string')`.  
      - `text` input from AI via `$fromAI('Text', '', 'string')` specifying update content.  
      - `sendResume` and `binaryPropertyName` similarly from AI input.  
      - Operation explicitly set to `"update"`.  
    - **Input:** Receives AI-driven parameters from MCP trigger.  
    - **Output:** Sends update response back to MCP client.  
    - **Credentials:** Same Humantic AI API credentials applied.  
    - **Potential Failures:**  
      - Missing or invalid `userId` or update `text`  
      - API request rejected due to invalid data or permissions  
      - Expression evaluation errors for input parameters  
    - **Sub-workflow Reference:** None  

#### 1.3 Workflow Metadata and Guidance

- **Overview:**  
  Provides detailed instructions and operational guidance for users deploying or maintaining the workflow.

- **Nodes Involved:**  
  - Workflow Overview 0 (Sticky Note)  
  - Sticky Note 1

- **Node Details:**

  - **Node Name:** Workflow Overview 0  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Technical Role:** Documentation node containing setup instructions, feature highlights, and support links.  
    - **Content Highlights:**  
      - Lists available operations (create, get, update)  
      - Stepwise setup instructions including credential configuration and webhook URL usage  
      - Notes on zero-configuration features and AI parameter population using `$fromAI()`  
      - Links to n8n documentation and Discord support for MCP integration  
    - **Position:** Placed at the top-left for visibility.  
    - **Potential Issues:** None (informational only).  

  - **Node Name:** Sticky Note 1  
    - **Type:** `n8n-nodes-base.stickyNote`  
    - **Technical Role:** A simple label "Profile" grouping the profile operation nodes visually.  
    - **Content:** "## Profile"  
    - **Potential Issues:** None  

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                        | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                                           |
|--------------------------|----------------------------------|--------------------------------------|----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0      | stickyNote                       | Workflow metadata and setup guide    | None                       | None                      | ## 🛠️ Humantic AI Tool MCP Server; Setup instructions; Available operations; Support links: [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/), [Discord](https://discord.me/cfomodz) |
| Humantic AI Tool MCP Server | @n8n/n8n-nodes-langchain.mcpTrigger | MCP webhook trigger for AI requests  | None                       | Create a profile, Get a profile, Update a profile (via ai_tool output) |                                                                                                                                     |
| Create a profile         | humanticAiTool                   | Creates a user profile                | Humantic AI Tool MCP Server | None                      |                                                                                                                                     |
| Get a profile            | humanticAiTool                   | Retrieves a user profile              | Humantic AI Tool MCP Server | None                      |                                                                                                                                     |
| Update a profile         | humanticAiTool                   | Updates a user profile                | Humantic AI Tool MCP Server | None                      |                                                                                                                                     |
| Sticky Note 1            | stickyNote                       | Label grouping profile nodes         | None                       | None                      | ## Profile                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note (Workflow Overview):**  
   - Add a Sticky Note node.  
   - Set width to ~420, height to ~760.  
   - Paste the following content (adjust formatting as needed):  
     ```
     ## 🛠️ Humantic AI Tool MCP Server

     ### 📋 Available Operations (3 total)

     Profile: create, get, update

     ### ⚙️ Setup Instructions

     1. Import Workflow into n8n
     2. Add Humantic AI API credentials in one tool node, then open/close others to sync credentials
     3. Activate the workflow
     4. Copy webhook URL from MCP trigger node
     5. Use webhook URL in your AI agent MCP configuration

     ### ✨ Ready-to-Use Features

     • Zero configuration - all 3 operations pre-built
     • AI agents populate parameters via $fromAI() expressions
     • Native n8n error handling and response formatting
     • Modify defaults in tool nodes as needed

     ### 💬 Need Help?

     See n8n docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/
     or Discord: https://discord.me/cfomodz
     ```
   - Position this node top-left.

2. **Create MCP Trigger Node:**  
   - Add `@n8n/n8n-nodes-langchain.mcpTrigger` node.  
   - Set the webhook path parameter to `"humantic-ai-tool-mcp"`.  
   - This node will listen for incoming MCP requests.  
   - Position roughly center-left.

3. **Create "Create a profile" Node:**  
   - Add `humanticAiTool` node.  
   - Configure parameters:  
     - `userId`: expression `$fromAI('User_Id', '', 'string')`  
     - `sendResume`: expression `$fromAI('Send_Resume', '', 'boolean')`  
     - `binaryPropertyName`: expression `$fromAI('Binary_Property_Name', '', 'string')`  
   - Set credentials for Humantic AI API (create and assign your Humantic AI API credential).  
   - Position left of MCP trigger node.

4. **Create "Get a profile" Node:**  
   - Add `humanticAiTool` node.  
   - Set operation to `"get"`.  
   - Configure:  
     - `userId`: expression `$fromAI('User_Id', '', 'string')`  
     - Leave options empty/default.  
   - Set same Humantic AI API credentials as above.  
   - Position right of "Create a profile" node.

5. **Create "Update a profile" Node:**  
   - Add `humanticAiTool` node.  
   - Set operation to `"update"`.  
   - Configure parameters:  
     - `userId`: `$fromAI('User_Id', '', 'string')`  
     - `text`: `$fromAI('Text', '', 'string')`  
     - `sendResume`: `$fromAI('Send_Resume', '', 'boolean')`  
     - `binaryPropertyName`: `$fromAI('Binary_Property_Name', '', 'string')`  
   - Assign Humantic AI API credentials.  
   - Position rightmost among profile nodes.

6. **Create Sticky Note (Profile Label):**  
   - Add a Sticky Note node near the three profile operation nodes.  
   - Content: `## Profile`  
   - Set color to a visible shade (e.g., color index 4).  
   - Adjust width to about 840 and height about 180 to cover the three nodes visually.

7. **Connect Nodes:**  
   - Connect the MCP Trigger node’s `ai_tool` output to each of the three profile operation nodes' inputs (`Create a profile`, `Get a profile`, `Update a profile`).  
   - No further connections necessary; MCP trigger handles branching internally.

8. **Credential Setup:**  
   - Create Humantic AI API credentials in n8n credentials manager.  
   - Assign these credentials to all three Humantic AI Tool nodes.

9. **Activate Workflow:**  
   - Save and activate the workflow.  
   - Copy the webhook URL from the MCP trigger node; this URL will be used by AI agents to invoke profile operations.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| AI agents populate parameters via `$fromAI()` expressions, enabling dynamic input from MCP requests | n8n MCP integration feature                                                                         |
| Humantic AI Tool API credentials must be configured once and assigned to all relevant nodes         | Credential management in n8n                                                                       |
| Workflow provides native error handling and response formatting for MCP requests                     | Improves robustness and ease of integration                                                        |
| Support and documentation available at n8n docs and Discord                                          | [n8n MCP docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) <br> [Discord](https://discord.me/cfomodz) |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.