🛠️ Pushbullet Tool MCP Server 💪 all 4 operations

https://n8nworkflows.xyz/workflows/----pushbullet-tool-mcp-server----all-4-operations-5091


# 🛠️ Pushbullet Tool MCP Server 💪 all 4 operations

### 1. Workflow Overview

This workflow implements a **Pushbullet Tool MCP Server** that exposes all four primary Pushbullet operations—create, delete, retrieve (get all), and update pushes—via a single MCP (Multi-Channel Provider) webhook interface. It is designed to be used by AI agents or external systems that require dynamic interaction with the Pushbullet API through standardized AI-driven parameter injection.

The workflow is logically organized into these blocks:

- **1.1 MCP Trigger Reception:** Listens for incoming requests from AI agents via the MCP webhook.
- **1.2 Pushbullet Operations:** Four parallel nodes each handling one Pushbullet operation (Create, Delete, Get All, Update), with parameters dynamically populated from AI via `$fromAI()` expressions.
- **1.3 Documentation & Guidance:** Sticky notes containing overview, setup instructions, and operation grouping for user reference.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Reception

- **Overview:**  
  This block acts as the entry point, receiving HTTP requests on a webhook URL configured as an MCP trigger. It serves as the centralized interface for AI agents to invoke any of the Pushbullet operations by routing input parameters to the appropriate tool nodes.

- **Nodes Involved:**  
  - `Pushbullet Tool MCP Server`

- **Node Details:**  
  - **Node Name:** Pushbullet Tool MCP Server  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - **Technical Role:** MCP Trigger node that exposes a webhook endpoint to receive requests and forward them internally as AI-driven tool invocations.  
  - **Configuration:**  
    - `path`: `"pushbullet-tool-mcp"` — defines the webhook URL suffix (`/webhook/pushbullet-tool-mcp`).  
  - **Input Connections:** None (start node)  
  - **Output Connections:** Connected internally as AI tool input to all Pushbullet operation nodes.  
  - **Version Requirements:** Requires n8n version supporting MCP nodes and webhook triggers.  
  - **Edge Cases / Failures:**  
    - Webhook URL misconfiguration or conflicts.  
    - Authentication or permission issues if webhook is not secured.  
    - Payload format errors from AI agent requests.  
  - **Sub-workflow:** None  

#### 2.2 Pushbullet Operations

This block contains four distinct nodes, each representing one Pushbullet API operation. All parameters are dynamically resolved using `$fromAI()` expressions, enabling AI agents to specify operation parameters flexibly.

- **Nodes Involved:**  
  - `Create a push`  
  - `Delete a push`  
  - `Get many pushes`  
  - `Update a push`

---

##### 2.2.1 Create a push

- **Overview:**  
  Creates a new Pushbullet push (note, link, file, etc.) with parameters provided dynamically.

- **Node Details:**  
  - **Node Name:** Create a push  
  - **Type:** `n8n-nodes-base.pushbulletTool`  
  - **Technical Role:** Executes the "push" operation on the Pushbullet API to create a new push.  
  - **Configuration:**  
    - Parameters use `$fromAI()` expressions to populate:  
      - `url`, `body`, `type`, `title`, `value`, `target`, `binaryPropertyName`  
    - No static default values to maximize flexibility.  
  - **Input Connections:** AI tool input from the MCP trigger node.  
  - **Output Connections:** None specified; output flows back to MCP infrastructure.  
  - **Version Requirements:** Pushbullet tool node compatible with n8n version.  
  - **Edge Cases / Failures:**  
    - Missing required parameters (e.g., `type` or `title`) leading to API errors.  
    - Invalid URLs or binary properties.  
    - Authentication failure if Pushbullet credentials are not configured.  
  - **Sub-workflow:** None  

---

##### 2.2.2 Delete a push

- **Overview:**  
  Deletes an existing push identified by its Push ID.

- **Node Details:**  
  - **Node Name:** Delete a push  
  - **Type:** `n8n-nodes-base.pushbulletTool`  
  - **Technical Role:** Executes the delete operation to remove a push from Pushbullet.  
  - **Configuration:**  
    - `operation`: hardcoded to `"delete"`  
    - `pushId`: dynamically from `$fromAI('Push_Id')`  
  - **Input Connections:** AI tool input from MCP trigger.  
  - **Output Connections:** None specified; response flows back via MCP.  
  - **Version Requirements:** Compatible Pushbullet tool node.  
  - **Edge Cases / Failures:**  
    - Invalid or missing `Push_Id` causing failure.  
    - Push already deleted or inaccessible.  
    - Authentication errors.  
  - **Sub-workflow:** None  

---

##### 2.2.3 Get many pushes

- **Overview:**  
  Retrieves multiple pushes with optional limit and returnAll flags.

- **Node Details:**  
  - **Node Name:** Get many pushes  
  - **Type:** `n8n-nodes-base.pushbulletTool`  
  - **Technical Role:** Executes the getAll operation to fetch pushes from Pushbullet.  
  - **Configuration:**  
    - `operation`: `"getAll"`  
    - `limit`: dynamically from `$fromAI('Limit')` (number)  
    - `returnAll`: dynamically from `$fromAI('Return_All')` (boolean)  
    - `filters`: empty object (no filtering configured)  
  - **Input Connections:** AI tool input from MCP trigger.  
  - **Output Connections:** None specified; returns data via MCP response.  
  - **Version Requirements:** Compatible Pushbullet tool node.  
  - **Edge Cases / Failures:**  
    - Invalid parameter values (e.g., negative limit).  
    - API rate limiting or timeout.  
    - Authentication errors.  
  - **Sub-workflow:** None  

---

##### 2.2.4 Update a push

- **Overview:**  
  Updates an existing push's properties, e.g., dismissed status.

- **Node Details:**  
  - **Node Name:** Update a push  
  - **Type:** `n8n-nodes-base.pushbulletTool`  
  - **Technical Role:** Executes the update operation on Pushbullet pushes.  
  - **Configuration:**  
    - `operation`: `"update"`  
    - `pushId`: from `$fromAI('Push_Id')`  
    - `dismissed`: from `$fromAI('Dismissed')` (boolean)  
  - **Input Connections:** AI tool input from MCP trigger.  
  - **Output Connections:** None specified; output returns via MCP.  
  - **Version Requirements:** Compatible Pushbullet tool node.  
  - **Edge Cases / Failures:**  
    - Missing or invalid push ID.  
    - Invalid dismissed flag.  
    - Push not found or already updated.  
    - Authentication issues.  
  - **Sub-workflow:** None  

---

#### 2.3 Documentation & Guidance

- **Overview:**  
  Sticky notes provide user-facing documentation and workflow structure grouping.

- **Nodes Involved:**  
  - `Workflow Overview 0` (large note with full setup and usage instructions)  
  - `Sticky Note 1` (labeling the "Push" operations block)

- **Node Details:**  
  - Sticky notes have no inputs or outputs, purely informational.  
  - Content includes setup steps, feature highlights, and support links:  
    - [n8n Pushbullet MCP Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)  
    - Discord link for support: https://discord.me/cfomodz  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                 | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                                               |
|-------------------------|----------------------------------|--------------------------------|-----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0     | Sticky Note                      | Documentation & Setup Guide     | None                        | None                      | ## 🛠️ Pushbullet Tool MCP Server ... See detailed setup instructions and support links.                                                  |
| Pushbullet Tool MCP Server | MCP Trigger                     | Entry webhook for MCP requests | None                        | Create a push, Delete a push, Get many pushes, Update a push (via AI tool) |                                                                                                                                           |
| Create a push           | Pushbullet Tool                  | Create new push operation       | Pushbullet Tool MCP Server  | None                      |                                                                                                                                           |
| Delete a push           | Pushbullet Tool                  | Delete existing push            | Pushbullet Tool MCP Server  | None                      |                                                                                                                                           |
| Get many pushes         | Pushbullet Tool                  | Retrieve multiple pushes        | Pushbullet Tool MCP Server  | None                      |                                                                                                                                           |
| Update a push           | Pushbullet Tool                  | Update existing push            | Pushbullet Tool MCP Server  | None                      |                                                                                                                                           |
| Sticky Note 1           | Sticky Note                     | Label "Push" operations block   | None                        | None                      | ## Push                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note "Workflow Overview 0":**  
   - Type: Sticky Note  
   - Position: Top-left area for visibility  
   - Content: Paste the full workflow overview and setup instructions (see 2.3)  
   - Size: Width 420, Height 800  

2. **Add MCP Trigger Node "Pushbullet Tool MCP Server":**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set parameter `path` to `"pushbullet-tool-mcp"`  
   - Position: Center left (-420, -100)  
   - No credentials needed for trigger itself.

3. **Add Pushbullet Tool Node "Create a push":**  
   - Type: `pushbulletTool`  
   - Position: (-800, 140)  
   - Parameters:  
     - `url`: `={{ $fromAI('Url', ``, 'string') }}`  
     - `body`: `={{ $fromAI('Body', ``, 'string') }}`  
     - `type`: `={{ $fromAI('Type', ``, 'string') }}`  
     - `title`: `={{ $fromAI('Title', ``, 'string') }}`  
     - `value`: `={{ $fromAI('Value', ``, 'string') }}`  
     - `target`: `={{ $fromAI('Target', ``, 'string') }}`  
     - `binaryPropertyName`: `={{ $fromAI('Binary_Property_Name', ``, 'string') }}`  
   - Connect AI tool input from MCP Trigger node.  
   - Configure Pushbullet credentials in this node (OAuth2 or API key).

4. **Add Pushbullet Tool Node "Delete a push":**  
   - Type: `pushbulletTool`  
   - Position: (-580, 140)  
   - Parameters:  
     - `operation`: `"delete"` (static)  
     - `pushId`: `={{ $fromAI('Push_Id', ``, 'string') }}`  
   - Connect AI tool input from MCP Trigger node.  
   - Use the same Pushbullet credentials.

5. **Add Pushbullet Tool Node "Get many pushes":**  
   - Type: `pushbulletTool`  
   - Position: (-360, 140)  
   - Parameters:  
     - `operation`: `"getAll"` (static)  
     - `limit`: `={{ $fromAI('Limit', ``, 'number') }}`  
     - `returnAll`: `={{ $fromAI('Return_All', ``, 'boolean') }}`  
     - `filters`: `{}` (empty object)  
   - Connect AI tool input from MCP Trigger node.  
   - Use shared Pushbullet credentials.

6. **Add Pushbullet Tool Node "Update a push":**  
   - Type: `pushbulletTool`  
   - Position: (-140, 140)  
   - Parameters:  
     - `operation`: `"update"` (static)  
     - `pushId`: `={{ $fromAI('Push_Id', ``, 'string') }}`  
     - `dismissed`: `={{ $fromAI('Dismissed', ``, 'boolean') }}`  
   - Connect AI tool input from MCP Trigger node.  
   - Use same Pushbullet credentials.

7. **Add Sticky Note "Sticky Note 1":**  
   - Type: Sticky Note  
   - Position: (-1000, 120)  
   - Content: `## Push`  
   - Color: Blue (color code 4)  
   - Width: 1060  

8. **Credentials Setup:**  
   - Add Pushbullet API credentials (token or OAuth2) in the n8n Credentials section.  
   - Link these credentials to all Pushbullet Tool nodes.  

9. **Finalization:**  
   - Activate the workflow.  
   - Obtain the webhook URL from the MCP Trigger node (e.g., `https://<n8n-url>/webhook/pushbullet-tool-mcp`).  
   - Use this URL in AI agent configurations to call any of the four operations with parameters as AI input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Check official n8n documentation for MCP nodes and Pushbullet Tool integration for detailed usage and troubleshooting.                                                  | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                                         |
| For real-time support or customization requests, join the Discord community at https://discord.me/cfomodz                                                               | https://discord.me/cfomodz                                                                                                           |
| This workflow uses `$fromAI()` expressions to dynamically bind AI-generated parameters to Pushbullet node inputs, enabling flexible AI-driven automation.              | n8n expression syntax documentation                                                                                                  |
| Ensure Pushbullet API credentials have the necessary scopes for all operations (create, delete, read, update pushes) to avoid authorization failures.                   | Pushbullet API documentation                                                                                                         |
| The MCP server design allows easy extension by adding more Pushbullet operations or other tool nodes following the same AI tool input pattern.                         | Architecture note                                                                                                                    |

---

**Disclaimer:** The provided description and analysis are exclusively derived from an automated n8n workflow export. All data processed is legal and public, adhering strictly to content policies.