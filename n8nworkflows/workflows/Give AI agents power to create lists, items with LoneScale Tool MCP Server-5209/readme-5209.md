Give AI agents power to create lists, items with LoneScale Tool MCP Server

https://n8nworkflows.xyz/workflows/give-ai-agents-power-to-create-lists--items-with-lonescale-tool-mcp-server-5209


# Give AI agents power to create lists, items with LoneScale Tool MCP Server

### 1. Workflow Overview

This n8n workflow implements a **LoneScale Tool MCP Server** designed to empower AI agents with the ability to autonomously create lists and add items to those lists using the LoneScale Tool integration. It is tailored for scenarios where AI-driven automation must dynamically manage categorized data entries, such as contact lists or company directories, without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger**: Listens for incoming AI requests via a webhook, serving as the entry point for all operations.
- **1.2 List Creation Block**: Handles creation of new lists based on parameters provided by AI agents.
- **1.3 Item Creation Block**: Adds new items (e.g., contacts or companies) into existing lists, using AI-supplied details.

Each block corresponds to specific LoneScale Tool operations exposed through the MCP server interface, enabling seamless AI interaction.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger

- **Overview:**  
  This block activates the workflow by receiving API calls from AI agents through a webhook. It acts as the LoneScale Tool MCP server’s main entry point, routing incoming AI tool requests to downstream nodes.

- **Nodes Involved:**  
  - LoneScale Tool MCP Server

- **Node Details:**  
  - **Node Name:** LoneScale Tool MCP Server  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - **Role:** Listens on the webhook path `lonescale-tool-mcp` for AI agent requests. Serves as a multi-purpose entry trigger for MCP operations.  
  - **Configuration:**  
    - Webhook path: `lonescale-tool-mcp`  
  - **Input:** HTTP request from AI agents (containing operation details and parameters)  
  - **Output:** Routes request data to AI tool nodes (“Create a list” and “Create an item”) via `ai_tool` outputs  
  - **Version:** Version 1; no special version constraints  
  - **Potential Failures:**  
    - Webhook not reachable if n8n instance offline or URL misconfigured  
    - Authentication issues if credentials not set correctly in downstream nodes  
    - Timeout if AI agent request payload is malformed or delayed  
  - **Sub-workflow:** None  

---

#### 1.2 List Creation Block

- **Overview:**  
  This block creates new lists in LoneScale based on AI-provided parameters. It is triggered by AI commands specifying list name and type.

- **Nodes Involved:**  
  - Create a list  
  - Sticky Note 1 (label "List")

- **Node Details:**  
  - **Node Name:** Create a list  
  - **Type:** `n8n-nodes-base.loneScaleTool`  
  - **Role:** Creates a new list resource in LoneScale with dynamic parameters  
  - **Configuration:**  
    - Resource: Implicitly “list” via operation context  
    - Parameters populated dynamically via expressions:  
      - `name`: populated by AI input via `$fromAI('Name', '', 'string')`  
      - `type`: populated by AI input via `$fromAI('Type', '', 'string')`  
  - **Input:** Receives AI tool commands from MCP trigger node  
  - **Output:** Sends API response data downstream (if any)  
  - **Version:** Version 1  
  - **Potential Failures:**  
    - Missing or invalid `name` or `type` parameters causing API errors  
    - Credential or authentication failures if LoneScale credentials are not configured  
    - API rate limiting or network errors  
    - Expression evaluation errors if AI input is malformed or missing  
  - **Sub-workflow:** None  

- **Sticky Note 1:**  
  - Content: `## List`  
  - Purpose: Visually groups and labels the "Create a list" node in the canvas for clarity  

---

#### 1.3 Item Creation Block

- **Overview:**  
  This block adds items (e.g., people or companies) to existing lists. It uses AI-provided list names, item types, and detailed fields like first name, last name, and company name.

- **Nodes Involved:**  
  - Create a item  
  - Sticky Note 2 (label "Item")

- **Node Details:**  
  - **Node Name:** Create a item  
  - **Type:** `n8n-nodes-base.loneScaleTool`  
  - **Role:** Adds an item resource under a specified list with detailed attributes  
  - **Configuration:**  
    - Resource: explicitly set to `"item"`  
    - Parameters dynamically populated via AI inputs:  
      - `list`: `$fromAI('List', '', 'string')` — target list where item is added  
      - `type`: `$fromAI('Type', '', 'string')` — item type (e.g., person, company)  
      - `last_name`, `first_name`, `company_name`: respective AI inputs for item details  
      - `peopleAdditionalFields` and `companyAdditionalFields`: empty objects, placeholders for extended attributes  
  - **Input:** Receives AI tool commands from MCP trigger node  
  - **Output:** Returns API response data  
  - **Version:** Version 1  
  - **Potential Failures:**  
    - Missing required parameters (`list`, `type`, or names) leading to API errors  
    - Incorrect mapping of AI input causing expression failures  
    - Authentication or network issues with LoneScale API  
    - Handling of empty or invalid additional fields may cause errors if not supported by API  
  - **Sub-workflow:** None  

- **Sticky Note 2:**  
  - Content: `## Item`  
  - Purpose: Visually groups and labels the "Create a item" node in the canvas for clarity  

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role              | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                                    |
|-------------------------|-------------------------------------|-----------------------------|-------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0     | n8n-nodes-base.stickyNote            | Documentation & Instructions | None                    | None                      | ## 🛠️ LoneScale Tool MCP Server<br>Setup instructions and feature overview with helpful links to docs and Discord.                            |
| LoneScale Tool MCP Server | @n8n/n8n-nodes-langchain.mcpTrigger | MCP Server webhook trigger   | None                    | Create a list, Create a item |                                                                                                                                              |
| Create a list           | n8n-nodes-base.loneScaleTool         | Creates a new list           | LoneScale Tool MCP Server | None                      | ## List                                                                                                                                        |
| Sticky Note 1           | n8n-nodes-base.stickyNote            | Visual label for List block  | None                    | None                      | ## List                                                                                                                                        |
| Create a item           | n8n-nodes-base.loneScaleTool         | Adds an item to a list       | LoneScale Tool MCP Server | None                      | ## Item                                                                                                                                        |
| Sticky Note 2           | n8n-nodes-base.stickyNote            | Visual label for Item block  | None                    | None                      | ## Item                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note:**  
   - Name: `Workflow Overview 0`  
   - Type: `stickyNote`  
   - Content: Paste the full text describing the LoneScale Tool MCP Server, available operations, setup instructions, features, and help links as provided in the original workflow.  
   - Position: Place at top-left area for visibility.  

2. **Create MCP Trigger Node:**  
   - Name: `LoneScale Tool MCP Server`  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Set webhook path to `lonescale-tool-mcp`  
   - Position: To the right of the Overview sticky note  

3. **Create Sticky Note for List Block:**  
   - Name: `Sticky Note 1`  
   - Type: `stickyNote`  
   - Content: `## List`  
   - Color: Optional (color index 4 in original)  
   - Position: Above List creation node  

4. **Create "Create a list" Node:**  
   - Name: `Create a list`  
   - Type: `loneScaleTool`  
   - Parameters:  
     - `name`: Expression `={{ $fromAI('Name', '', 'string') }}`  
     - `type`: Expression `={{ $fromAI('Type', '', 'string') }}`  
   - Credentials: Configure LoneScale Tool credentials here (OAuth2 or API key as required)  
   - Position: Below Sticky Note 1  

5. **Connect MCP Trigger to Create a list:**  
   - Create connection from `LoneScale Tool MCP Server` node’s `ai_tool` output to `Create a list` node input.  

6. **Create Sticky Note for Item Block:**  
   - Name: `Sticky Note 2`  
   - Type: `stickyNote`  
   - Content: `## Item`  
   - Color: Optional (color index 5 in original)  
   - Position: Above Item creation node  

7. **Create "Create a item" Node:**  
   - Name: `Create a item`  
   - Type: `loneScaleTool`  
   - Parameters:  
     - `resource`: Set to `"item"` explicitly  
     - `list`: Expression `={{ $fromAI('List', '', 'string') }}`  
     - `type`: Expression `={{ $fromAI('Type', '', 'string') }}`  
     - `last_name`: Expression `={{ $fromAI('Last_Name', '', 'string') }}`  
     - `first_name`: Expression `={{ $fromAI('First_Name', '', 'string') }}`  
     - `company_name`: Expression `={{ $fromAI('Company_Name', '', 'string') }}`  
     - `peopleAdditionalFields`: leave as empty object `{}`  
     - `companyAdditionalFields`: leave as empty object `{}`  
   - Credentials: Use the same LoneScale Tool credentials as for the list node  
   - Position: Below Sticky Note 2  

8. **Connect MCP Trigger to Create a item:**  
   - Create connection from `LoneScale Tool MCP Server` node’s `ai_tool` output to `Create a item` node input.  

9. **Credentials Setup:**  
   - Add and configure LoneScale Tool credentials in the n8n Credentials section before activating the workflow.  
   - Ensure the credentials are authorized to perform list and item operations on the LoneScale API.  

10. **Activate Workflow:**  
    - Enable the workflow to start listening on the webhook path `/lonescale-tool-mcp`.  
    - Copy the webhook URL from the MCP trigger node for integration with AI agents.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                       | Context or Link                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Check the [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) for detailed info on MCP integration and usage.                        | Official n8n MCP Trigger node documentation                                                                       |
| For direct support and community help on MCP integration and customizations, join the [Discord server](https://discord.me/cfomodz).                                                              | n8n community and developer Discord channel                                                                        |
| The workflow is designed for zero-configuration usage with AI agents automatically populating parameters via `$fromAI()` expressions, simplifying integration efforts in AI-driven automation.     | Feature description                                                                                                |
| This MCP server supports exactly two operations for LoneScale Tool: `List:create` and `Item:add`. Adjustments require modifying or adding nodes accordingly.                                     | Functional scope                                                                                                   |

---

**Disclaimer:**  
The provided text and workflow derive exclusively from an n8n automation workflow. It complies fully with applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.