🛠️ Philips Hue Tool MCP Server 💪 all 4 operations

https://n8nworkflows.xyz/workflows/----philips-hue-tool-mcp-server----all-4-operations-5095


# 🛠️ Philips Hue Tool MCP Server 💪 all 4 operations

### 1. Workflow Overview

This workflow implements a **Philips Hue Tool MCP (Machine Control Protocol) Server** enabling remote control of Philips Hue lighting devices through four primary operations on the light resource: **delete**, **get (single)**, **get all**, and **update**. It is designed to serve as a backend MCP server endpoint for AI agents or other clients to interact with Philips Hue lights via standardized API calls.

The workflow is logically organized into the following blocks:

- **1.1 MCP Server Trigger**: Listens for incoming MCP requests on a webhook endpoint.
- **1.2 Light Operations**: Handles the four Philips Hue light operations (delete, get single, get all, update), each implemented as a separate Philips Hue Tool node.
- **1.3 Workflow Metadata and Documentation**: Provides detailed setup instructions, operational overview, and references via sticky notes.

This structure allows AI agents to send commands to the MCP server, which routes and executes the appropriate Philips Hue API operation with parameters dynamically provided from AI inputs.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger

- **Overview:**  
  This block serves as the entry point, waiting for incoming MCP requests over HTTP via a webhook. It triggers the workflow execution and routes input parameters to the corresponding Philips Hue Tool nodes.

- **Nodes Involved:**  
  - Philips Hue Tool MCP Server

- **Node Details:**  
  - **Philips Hue Tool MCP Server**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.mcpTrigger` — triggers the workflow through an MCP webhook, enabling external AI agents to call the workflow operations.  
    - *Configuration:*  
      - Webhook path set to `"philips-hue-tool-mcp"`.  
      - No additional parameters configured, operates with default MCP trigger settings.  
    - *Input/Output:*  
      - Receives HTTP requests containing AI-structured data.  
      - Outputs AI tool data routed to connected Philips Hue Tool nodes.  
    - *Version & Requirements:* Compatible with n8n versions supporting MCP triggers and the Philips Hue Tool integration.  
    - *Edge Cases & Failures:*  
      - Webhook URL misconfiguration or network issues could block triggering.  
      - Authentication failures if Philips Hue credentials are not set or invalid.  
      - Timeout or malformed input could cause errors in downstream nodes.  
    - *Sub-workflow:* None.

#### 2.2 Light Operations

- **Overview:**  
  This block contains four individual Philips Hue Tool nodes, each implementing one CRUD operation on Philips Hue lights. They receive parameters dynamically from AI via the MCP trigger and perform the API calls accordingly.

- **Nodes Involved:**  
  - Delete a light  
  - Get a light  
  - Get many lights  
  - Update a light

- **Node Details:**  

  - **Delete a light**  
    - *Type & Role:* `n8n-nodes-base.philipsHueTool` — deletes a specified light.  
    - *Configuration:*  
      - Operation set to `"delete"`.  
      - Parameter `lightId` dynamically injected via expression `$fromAI('Light_Id', '', 'string')`.  
    - *Input/Output:*  
      - Input: Parameters from MCP trigger.  
      - Output: API response confirming deletion.  
    - *Edge Cases:*  
      - Invalid or missing `lightId` causes failure.  
      - Unauthorized or network errors from Philips Hue API.  
    - *Credentials:* Required Philips Hue credentials (configured once).  

  - **Get a light**  
    - *Type & Role:* `n8n-nodes-base.philipsHueTool` — gets details for one light.  
    - *Configuration:*  
      - Operation set to `"get"`.  
      - Parameter `lightId` via `$fromAI('Light_Id', '', 'string')`.  
    - *Edge Cases:*  
      - Same as Delete: invalid ID, auth failures.  

  - **Get many lights**  
    - *Type & Role:* `n8n-nodes-base.philipsHueTool` — retrieves multiple lights.  
    - *Configuration:*  
      - Operation `"getAll"`.  
      - Parameters: `limit` as number, `returnAll` as boolean, both from AI inputs.  
    - *Edge Cases:*  
      - Invalid parameter types or values may cause errors.  

  - **Update a light**  
    - *Type & Role:* `n8n-nodes-base.philipsHueTool` — updates light properties.  
    - *Configuration:*  
      - Operation `"update"`.  
      - Parameters:  
        - `lightId` from AI input.  
        - `on` (boolean) from AI input controlling ON/OFF state.  
        - `additionalFields` empty by default, can be extended for other light properties.  
    - *Edge Cases:*  
      - Invalid light ID or parameters.  
      - Unsupported update fields could cause API errors.  

- *Connections:*  
  All four Philips Hue Tool nodes receive input connections from the MCP Server node’s AI tool output. This setup enables dynamic routing of AI-supplied parameters to each operation node.

- *Credentials:*  
  Each Philips Hue Tool node requires Philips Hue API credentials. The instruction is to configure credentials once in one tool node and then open and close the others to ensure shared credential use.

#### 2.3 Workflow Metadata and Documentation

- **Overview:**  
  This block provides detailed instructions and workflow context via sticky notes, aiding users in setup, operation, and troubleshooting.

- **Nodes Involved:**  
  - Workflow Overview 0 (main sticky note)  
  - Sticky Note 1 ("Light")

- **Node Details:**  

  - **Workflow Overview 0**  
    - *Type & Role:* `n8n-nodes-base.stickyNote` — contains formatted markdown instructions and overview text.  
    - *Content Highlights:*  
      - Lists the four available operations on lights.  
      - Stepwise setup instructions including importing, credential configuration, activation, and webhook URL retrieval.  
      - Notes on zero-configuration, AI parameter population via `$fromAI()`, and error handling.  
      - References to official n8n documentation and Discord support.  
    - *Position:* Left side for easy reference.

  - **Sticky Note 1**  
    - *Type & Role:* Simple labeled note with text "Light" to group the operation nodes visually.

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                  | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                       |
|--------------------------|----------------------------------|--------------------------------|---------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Workflow Overview 0      | Sticky Note                      | Documentation & instructions   | —                         | —                       | ## 🛠️ Philips Hue Tool MCP Server... See detailed setup & support info at https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ and https://discord.me/cfomodz |
| Philips Hue Tool MCP Server | MCP Trigger (@n8n/n8n-nodes-langchain.mcpTrigger) | Entry point for MCP requests   | —                         | Delete a light, Get a light, Get many lights, Update a light |                                                                                                 |
| Delete a light           | Philips Hue Tool                 | Delete a specified light       | Philips Hue Tool MCP Server| —                       |                                                                                                 |
| Get a light              | Philips Hue Tool                 | Retrieve details of one light  | Philips Hue Tool MCP Server| —                       |                                                                                                 |
| Get many lights          | Philips Hue Tool                 | Retrieve multiple lights       | Philips Hue Tool MCP Server| —                       |                                                                                                 |
| Update a light           | Philips Hue Tool                 | Update properties of a light   | Philips Hue Tool MCP Server| —                       |                                                                                                 |
| Sticky Note 1            | Sticky Note                     | Visual group label for lights  | —                         | —                       | Light                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Sticky Note node:**  
   - Name: `Workflow Overview 0`  
   - Content: Paste the detailed markdown overview and instructions from the original sticky note, including links to docs and Discord support.  
   - Position it on the left side for reference.

3. **Add an MCP Trigger node:**  
   - Name: `Philips Hue Tool MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Set the webhook path to `philips-hue-tool-mcp`.  
   - Position roughly center-left.  
   - This node will receive all incoming MCP requests.

4. **Add four Philips Hue Tool nodes for each operation:**  

   a. **Delete a light**  
      - Name: `Delete a light`  
      - Type: `n8n-nodes-base.philipsHueTool`  
      - Parameters:  
        - Operation: `delete`  
        - `lightId`: Expression — `={{ $fromAI('Light_Id', '', 'string') }}`  
      - Connect input from `Philips Hue Tool MCP Server` AI Tool output.

   b. **Get a light**  
      - Name: `Get a light`  
      - Type: `n8n-nodes-base.philipsHueTool`  
      - Parameters:  
        - Operation: `get`  
        - `lightId`: Expression — `={{ $fromAI('Light_Id', '', 'string') }}`  
      - Connect input from `Philips Hue Tool MCP Server` AI Tool output.

   c. **Get many lights**  
      - Name: `Get many lights`  
      - Type: `n8n-nodes-base.philipsHueTool`  
      - Parameters:  
        - Operation: `getAll`  
        - `limit`: Expression — `={{ $fromAI('Limit', '', 'number') }}`  
        - `returnAll`: Expression — `={{ $fromAI('Return_All', '', 'boolean') }}`  
      - Connect input from `Philips Hue Tool MCP Server` AI Tool output.

   d. **Update a light**  
      - Name: `Update a light`  
      - Type: `n8n-nodes-base.philipsHueTool`  
      - Parameters:  
        - Operation: `update`  
        - `lightId`: Expression — `={{ $fromAI('Light_Id', '', 'string') }}`  
        - `on`: Expression — `={{ $fromAI('On', '', 'boolean') }}`  
        - `additionalFields`: leave empty or extend as needed.  
      - Connect input from `Philips Hue Tool MCP Server` AI Tool output.

5. **Add a small Sticky Note node:**  
   - Name: `Sticky Note 1`  
   - Content: `Light`  
   - Use to visually group the four Philips Hue Tool nodes.

6. **Configure Philips Hue API credentials:**  
   - Add Philips Hue Tool credentials in n8n Credentials manager.  
   - Assign the credentials to the first Philips Hue Tool node (e.g., Delete a light).  
   - Open and close other Philips Hue Tool nodes to inherit credentials automatically.

7. **Activate the workflow.**

8. **Use the webhook URL from the MCP Trigger node:**  
   - Copy the webhook URL from the MCP Trigger node details.  
   - Configure your AI agent or client to send MCP requests to this URL, passing parameters expected by `$fromAI()` expressions (e.g., Light_Id, On, Limit, Return_All).

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Full setup instructions and MCP integration guidance are available in the official n8n documentation for the Philips Hue Tool MCP node. | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/           |
| Community support and customization help can be found on Discord.                                                                          | https://discord.me/cfomodz                                                                               |
| The workflow uses `$fromAI()` expressions to dynamically populate parameters from AI agent inputs, simplifying integration.              | n8n expression function for AI-driven parameter injection                                               |
| Native n8n error handling captures API failures and invalid inputs, ensuring robust operation.                                              | Error handling is built into Philips Hue Tool nodes and MCP trigger                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.