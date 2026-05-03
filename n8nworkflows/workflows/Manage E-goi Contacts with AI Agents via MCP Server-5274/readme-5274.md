Manage E-goi Contacts with AI Agents via MCP Server

https://n8nworkflows.xyz/workflows/manage-e-goi-contacts-with-ai-agents-via-mcp-server-5274


# Manage E-goi Contacts with AI Agents via MCP Server

---
### 1. Workflow Overview

This workflow, titled **"Manage E-goi Contacts with AI Agents via MCP Server"**, serves as a backend automation server that facilitates managing contacts on the E-goi platform through AI-driven interactions. It is designed as a Modular Control Protocol (MCP) server that exposes operations over a webhook to AI agents, enabling them to create, retrieve, update, and list contacts without manual configuration.

**Target Use Cases:**

- Automate contact management in E-goi via AI agents or external systems.
- Allow AI agents to dynamically specify operation parameters using `$fromAI()` expressions.
- Serve as a reusable MCP server endpoint for seamless E-goi integration in AI workflows.

**Logical Blocks:**

- **1.1 Workflow Metadata and Setup Instructions**: Provides user guidance and general info through sticky notes.
- **1.2 MCP Server Trigger**: Listens for AI agent calls via a webhook.
- **1.3 Contact Operations**: Implements four core E-goi contact management operations—Create, Get (single), Get All, and Update.
- **1.4 AI Parameter Injection**: Uses dynamic `$fromAI()` expressions in operation nodes to receive parameters from AI agents.

---

### 2. Block-by-Block Analysis

#### 1.1 Workflow Metadata and Setup Instructions

- **Overview:**  
  This block contains sticky notes that provide an overview of the workflow, setup instructions, and references for users. It does not perform automation but guides users and developers.

- **Nodes Involved:**  
  - Workflow Overview 0 (Sticky Note)  
  - Sticky Note 1 (Sticky Note)

- **Node Details:**

  - **Workflow Overview 0**  
    - *Type & Role:* Sticky note node; provides detailed workflow description, usage instructions, and helpful links.  
    - *Configuration:* Contains markdown-formatted content including operation list, steps to set up credentials, activate the workflow, and use it with AI agents; includes links to official n8n documentation and Discord support.  
    - *Input/Output:* None (informational only).  
    - *Edge Cases:* None; purely informational.

  - **Sticky Note 1**  
    - *Type & Role:* Sticky note node; briefly labels the "Contact" section.  
    - *Configuration:* Simple heading "## Contact".  
    - *Input/Output:* None.  
    - *Edge Cases:* None.

---

#### 1.2 MCP Server Trigger

- **Overview:**  
  This node acts as the entry point for all AI agent requests. It exposes a webhook endpoint that listens for incoming MCP calls, initiating the workflow.

- **Nodes Involved:**  
  - E-goi Tool MCP Server (MCP Trigger)

- **Node Details:**

  - **E-goi Tool MCP Server**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.mcpTrigger` node; listens for MCP protocol calls from AI agents.  
    - *Configuration:*  
      - Webhook path set to `/e-goi-tool-mcp`.  
      - No additional parameters specified.  
    - *Input Connections:* None; this is a trigger node.  
    - *Output Connections:* Feeds into all four contact operation nodes (`Create a member`, `Get a member`, `Get many members`, `Update a member`) via the `ai_tool` output.  
    - *Edge Cases / Failures:*  
      - Webhook URL must be publicly accessible for AI agents to call.  
      - Network issues or invalid MCP requests may cause failures.  
      - MCP version compatibility must match AI agent expectations.

---

#### 1.3 Contact Operations

- **Overview:**  
  This block implements the four main contact operations on E-goi, each mapped to a dedicated node. Parameters for these operations are dynamically populated via AI inputs using `$fromAI()` expressions, enabling flexible and dynamic usage.

- **Nodes Involved:**  
  - Create a member  
  - Get a member  
  - Get many members  
  - Update a member

- **Node Details:**

  - **Create a member**  
    - *Type & Role:* `egoiTool` node; performs contact creation on E-goi.  
    - *Configuration:*  
      - Operation: `create` (default implied by node name).  
      - No additional fields pre-populated; all parameters expected from AI or defaults.  
      - Credentials: E-goi API credential placeholder requiring user setup.  
    - *Inputs:* Receives AI parameters from MCP Trigger node.  
    - *Outputs:* Returns created member data.  
    - *Expressions:* No explicit `$fromAI()` in parameters, indicating default or AI-populated inputs handled internally.  
    - *Edge Cases:*  
      - Missing or invalid credential leads to auth errors.  
      - Required contact fields must be provided or creation may fail.  
      - Network or API errors possible.

  - **Get a member**  
    - *Type & Role:* `egoiTool` node; retrieves a single contact using specified criteria.  
    - *Configuration:*  
      - Operation set explicitly to `get`.  
      - Parameters use `$fromAI()` expressions for dynamic input:  
        - `by`: method to identify contact (e.g., email, ID)  
        - `email`: contact email string  
        - `simple`: boolean toggle for simplified response  
        - `contactId`: ID string of contact  
      - Credentials: E-goi API credential (same placeholder).  
    - *Inputs:* Connected from MCP trigger.  
    - *Outputs:* Returns requested contact data.  
    - *Edge Cases:*  
      - Invalid or missing identification parameters cause errors.  
      - Credential or API errors possible.  
      - If contact not found, API returns empty or error response.

  - **Get many members**  
    - *Type & Role:* `egoiTool` node; lists multiple contacts with optional limits.  
    - *Configuration:*  
      - Operation set to `getAll`.  
      - Parameters use `$fromAI()` expressions:  
        - `limit`: number of contacts to retrieve  
        - `simple`: boolean for simplified output  
        - `returnAll`: boolean to ignore limit and return all contacts  
      - Credentials: E-goi API credential.  
    - *Inputs:* From MCP trigger node.  
    - *Outputs:* Returns list/array of contacts.  
    - *Edge Cases:*  
      - Large data sets may cause timeouts.  
      - Credential/API errors.  
      - If `returnAll` is true, API load may increase.

  - **Update a member**  
    - *Type & Role:* `egoiTool` node; updates contact details.  
    - *Configuration:*  
      - Operation set to `update`.  
      - Parameters populated dynamically via `$fromAI()`:  
        - `contactId`: identifies which contact to update  
        - `updateFields`: object containing fields to update (empty by default, expecting AI input)  
      - Credentials: E-goi API credential.  
    - *Inputs:* From MCP trigger node.  
    - *Outputs:* Returns updated contact data.  
    - *Edge Cases:*  
      - Missing `contactId` or invalid fields cause failure.  
      - Credential/API errors.  
      - Partial updates may not reflect as expected if fields are empty.

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                   | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                                                     |
|---------------------|--------------------------------|---------------------------------|------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0  | Sticky Note                    | Workflow metadata and instructions | None                   | None                      | ## 🛠️ E-goi Tool MCP Server - Setup instructions and operation list, with helpful links to n8n docs and discord support.       |
| Sticky Note 1       | Sticky Note                    | Section label for Contact block | None                   | None                      | ## Contact                                                                                                                     |
| E-goi Tool MCP Server| MCP Trigger (`mcpTrigger`)    | Entry point for AI agent requests | None                   | Create a member, Get a member, Get many members, Update a member |                                                                                                                                |
| Create a member      | E-goi Tool (`egoiTool`)       | Create new contact on E-goi     | E-goi Tool MCP Server  | None                      |                                                                                                                                |
| Get a member         | E-goi Tool (`egoiTool`)       | Retrieve a single contact       | E-goi Tool MCP Server  | None                      |                                                                                                                                |
| Get many members     | E-goi Tool (`egoiTool`)       | Retrieve multiple contacts      | E-goi Tool MCP Server  | None                      |                                                                                                                                |
| Update a member      | E-goi Tool (`egoiTool`)       | Update contact information      | E-goi Tool MCP Server  | None                      |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note "Workflow Overview 0"**  
   - Type: Sticky Note  
   - Content: Markdown text describing the workflow purpose, available operations (create, get, get all, update), setup instructions (import workflow, add credentials, activate, get URL, connect AI), and helpful links:  
     - n8n docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/  
     - Discord: https://discord.me/cfomodz  
   - Position: Approximate coordinates [-1460, -280]  
   - Size: Width 420, Height 760

2. **Create Sticky Note "Sticky Note 1"**  
   - Type: Sticky Note  
   - Content: `## Contact`  
   - Position: [-1000, 120]  
   - Size: Width 1060, Height 180  
   - Color: Optional (color code 4 used in original)

3. **Add MCP Trigger Node "E-goi Tool MCP Server"**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `e-goi-tool-mcp`  
   - Position: [-420, -240]  
   - No credentials needed.  
   - This node will receive incoming MCP calls from AI agents.

4. **Add E-goi Tool Node "Create a member"**  
   - Type: `egoiTool`  
   - Parameters:  
     - Operation: Create (default)  
     - Additional Fields: Empty (defaults)  
   - Credentials: Set up and link valid E-goi API credentials.  
   - Position: [-800, 140]  
   - Connect input from MCP trigger node's `ai_tool` output.

5. **Add E-goi Tool Node "Get a member"**  
   - Type: `egoiTool`  
   - Parameters:  
     - Operation: `get`  
     - By: `={{ $fromAI('By', '', 'string') }}`  
     - Email: `={{ $fromAI('Email', '', 'string') }}`  
     - Simple: `={{ $fromAI('Simple', '', 'boolean') }}`  
     - ContactId: `={{ $fromAI('Contact_Id', '', 'string') }}`  
   - Credentials: Same E-goi API credentials as above.  
   - Position: [-580, 140]  
   - Connect input from MCP trigger node's `ai_tool` output.

6. **Add E-goi Tool Node "Get many members"**  
   - Type: `egoiTool`  
   - Parameters:  
     - Operation: `getAll`  
     - Limit: `={{ $fromAI('Limit', '', 'number') }}`  
     - Simple: `={{ $fromAI('Simple', '', 'boolean') }}`  
     - ReturnAll: `={{ $fromAI('Return_All', '', 'boolean') }}`  
   - Credentials: E-goi API credentials.  
   - Position: [-360, 140]  
   - Connect input from MCP trigger node's `ai_tool` output.

7. **Add E-goi Tool Node "Update a member"**  
   - Type: `egoiTool`  
   - Parameters:  
     - Operation: `update`  
     - ContactId: `={{ $fromAI('Contact_Id', '', 'string') }}`  
     - UpdateFields: Empty object `{}` (to be filled dynamically)  
   - Credentials: E-goi API credentials.  
   - Position: [-140, 140]  
   - Connect input from MCP trigger node's `ai_tool` output.

8. **Set Credentials**  
   - Create or import valid E-goi API credentials in n8n.  
   - Attach these credentials to each `egoiTool` node.

9. **Activate Workflow**  
   - Enable the workflow.  
   - Copy the webhook URL from the MCP trigger node (displayed in n8n).  
   - Configure AI agents to send MCP requests to this URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| The workflow uses `$fromAI()` expressions to dynamically inject parameters from AI agents, enabling zero manual config.| n8n expression feature; enables AI-driven parameter passing.                                                            |
| For detailed MCP node usage and configuration, refer to the official n8n MCP documentation.                           | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                            |
| Support and customization assistance are available on the n8n Discord server.                                         | https://discord.me/cfomodz                                                                                              |
| To use this workflow, ensure the E-goi API credentials are correctly configured with permissions for contact management.| E-goi platform API documentation and credential setup required.                                                         |

---

**Disclaimer:**  
This document is generated from an automated n8n workflow. It complies with current content policies, contains no illegal or protected content, and processes only legal public data.