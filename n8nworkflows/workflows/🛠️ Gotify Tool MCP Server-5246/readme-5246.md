🛠️ Gotify Tool MCP Server

https://n8nworkflows.xyz/workflows/----gotify-tool-mcp-server-5246


# 🛠️ Gotify Tool MCP Server

---

### 1. Workflow Overview

The **🛠️ Gotify Tool MCP Server** workflow is designed to expose Gotify messaging operations via an MCP (Multi-Channel Platform) server interface, enabling AI agents or other clients to perform message management tasks programmatically. It supports three primary operations on Gotify messages: creating a message, deleting a message, and retrieving multiple messages.

The workflow logic is grouped into the following logical blocks:

- **1.1 MCP Trigger Reception:** Listens for incoming MCP requests at a webhook endpoint, initiating workflow execution.
- **1.2 Gotify Message Operations:** Contains three parallel nodes that implement the core Gotify operations:
  - Creating a message
  - Deleting a message
  - Getting many messages
- **1.3 Documentation & Instructions:** Sticky notes providing usage instructions, setup guidance, and operation summaries.

This setup allows zero-configuration usage with AI agents automatically providing parameters via `$fromAI()` expressions, facilitating seamless integration into AI-driven automation environments.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Trigger Reception

- **Overview:**  
  This block receives incoming MCP requests targeting the Gotify messaging operations through a dedicated webhook. It acts as the entry point of the workflow, routing requests to the appropriate Gotify operation nodes.

- **Nodes Involved:**  
  - `Gotify Tool MCP Server`

- **Node Details:**

  | Node Name               | Details                                                                                                              |
  |-------------------------|----------------------------------------------------------------------------------------------------------------------|
  | Gotify Tool MCP Server  | - **Type:** MCP Trigger (LangChain MCP Trigger) <br> - **Role:** Receives MCP requests on path `/gotify-tool-mcp` <br> - **Configuration:** Webhook path set to `"gotify-tool-mcp"`; webhook ID is system-generated <br> - **Input/Output:** No input nodes; outputs to all three Gotify operation nodes via `ai_tool` connection <br> - **Expressions:** Uses no expressions itself; serves as trigger <br> - **Failure modes:** Webhook access errors, malformed requests, or unsupported MCP calls <br> - **Version:** Requires n8n version supporting LangChain MCP Trigger nodes (post v1.95) <br> - **Sub-workflow:** None |

#### 2.2 Gotify Message Operations

- **Overview:**  
  This block implements the core Gotify API interactions. Each node corresponds to a specific operation and utilizes AI-driven dynamic parameter injection to handle incoming requests flexibly.

- **Nodes Involved:**  
  - `Create a message`  
  - `Delete a message`  
  - `Get many messages`

- **Node Details:**

  | Node Name         | Details                                                                                                                                                                                                                       |
  |-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Create a message  | - **Type:** Gotify Tool Node <br> - **Role:** Sends a request to create a Gotify message <br> - **Configuration:** Uses `$fromAI('Message', '', 'string')` expression to dynamically obtain the message content from AI input <br> - **Credentials:** Requires a Gotify API credential configured under `gotifyApi` <br> - **Input:** Connected from `Gotify Tool MCP Server` <br> - **Output:** Returns Created message data <br> - **Failure modes:** Invalid credentials, API errors, empty or malformed message content <br> - **Version:** Requires the Gotify Tool node available in n8n <br> - **Sub-workflow:** None |
  | Delete a message  | - **Type:** Gotify Tool Node <br> - **Role:** Deletes a Gotify message by ID <br> - **Configuration:** Operation set to `"delete"`; message ID dynamically obtained via `$fromAI('Message_Id', '', 'string')` <br> - **Credentials:** Same Gotify API credential as above <br> - **Input:** Connected from `Gotify Tool MCP Server` <br> - **Output:** Returns deletion confirmation or error <br> - **Failure modes:** Invalid message ID, authorization failure, API errors <br> - **Version:** As above <br> - **Sub-workflow:** None |
  | Get many messages | - **Type:** Gotify Tool Node <br> - **Role:** Retrieves multiple Gotify messages <br> - **Configuration:** Operation set to `"getAll"`; parameters `limit` and `returnAll` dynamically injected via `$fromAI('Limit', '', 'number')` and `$fromAI('Return_All', '', 'boolean')` respectively <br> - **Credentials:** Same Gotify API credential <br> - **Input:** Connected from `Gotify Tool MCP Server` <br> - **Output:** Returns list of messages as per parameters <br> - **Failure modes:** Invalid limit values, API timeout, malformed parameters <br> - **Version:** As above <br> - **Sub-workflow:** None |

#### 2.3 Documentation & Instructions

- **Overview:**  
  This block contains sticky notes documenting the workflow's purpose, available operations, setup instructions, and help resources. It is informational only and does not affect workflow execution.

- **Nodes Involved:**  
  - `Workflow Overview 0`  
  - `Sticky Note 1`

- **Node Details:**

  | Node Name          | Details                                                                                                                                                                                                                          |
  |--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | Workflow Overview 0 | - **Type:** Sticky Note <br> - **Role:** Provides detailed overview, setup instructions, features list, and help links for the workflow <br> - **Content Highlights:** <br> • Lists 3 operations: create, delete, get all messages <br> • Stepwise setup guide <br> • Usage tips and AI integration notes <br> • Links to official n8n docs and Discord for help <br> - **Impact:** None on runtime <br> - **Position:** Top-left for visibility |
  | Sticky Note 1       | - **Type:** Sticky Note <br> - **Role:** Labels the grouping of message-related nodes with the header “Message” <br> - **Content:** Simple title with no dynamic content <br> - **Impact:** None on runtime |

---

### 3. Summary Table

| Node Name            | Node Type                 | Functional Role                      | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                  |
|----------------------|---------------------------|------------------------------------|------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0   | Sticky Note               | Documentation & instructions       | None                   | None                       | ## 🛠️ Gotify Tool MCP Server Setup and usage instructions with links to n8n docs and Discord                 |
| Gotify Tool MCP Server| MCP Trigger (LangChain)   | Entry trigger receiving MCP calls  | None                   | Create a message, Delete a message, Get many messages |                                                                                                              |
| Create a message      | Gotify Tool Node          | Creates Gotify messages             | Gotify Tool MCP Server | None                       |                                                                                                              |
| Delete a message      | Gotify Tool Node          | Deletes Gotify messages by ID       | Gotify Tool MCP Server | None                       |                                                                                                              |
| Get many messages     | Gotify Tool Node          | Retrieves multiple Gotify messages  | Gotify Tool MCP Server | None                       |                                                                                                              |
| Sticky Note 1         | Sticky Note               | Section label “Message”             | None                   | None                       |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note for Workflow Overview:**  
   - Add a Sticky Note node  
   - Set content to:  
     ```
     ## 🛠️ Gotify Tool MCP Server

     ### 📋 Available Operations (3 total)

     **Message**: create, delete, get all

     ### ⚙️ Setup Instructions

     1. **Import Workflow**: Load this workflow into your n8n instance
     2. **🔑 Add Credentials**: Configure Gotify Tool authentication in one tool node then open and close all others.
     3. **🚀 Activate**: Enable this workflow to start your MCP server
     4. **🔗 Get URL**: Copy webhook URL from MCP trigger (right side)
     5. **🤖 Connect**: Use MCP URL in your AI agent configurations

     ### ✨ Ready-to-Use Features

     • Zero configuration - all 3 operations pre-built
     • AI agents automatically populate parameters via `$fromAI()` expressions
     • Every resource and operation combination available
     • Native n8n error handling and response formatting
     • Modify parameter defaults in any tool node as needed

     ### 💬 Need Help?
     Check the [n8n documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) or ping me on [discord](https://discord.me/cfomodz) for MCP integration guidance or customizations.
     ```
   - Position near top-left of canvas.

2. **Create MCP Trigger Node:**  
   - Add a node of type **LangChain MCP Trigger**  
   - Set parameter `path` to `"gotify-tool-mcp"`  
   - No credentials needed  
   - This will generate a webhook URL to be used externally  
   - Position center-left

3. **Create Gotify Tool Node for "Create a message":**  
   - Add a **Gotify Tool** node  
   - Set operation to default (create message)  
   - Configure `message` parameter with expression:  
     ```
     {{$fromAI('Message', '', 'string')}}
     ```  
   - Assign Gotify API credentials (create or select your Gotify API credential)  
   - Connect MCP Trigger node output to this node input  
   - Position left-bottom

4. **Create Gotify Tool Node for "Delete a message":**  
   - Add another **Gotify Tool** node  
   - Set operation to `"delete"`  
   - Set `messageId` parameter with expression:  
     ```
     {{$fromAI('Message_Id', '', 'string')}}
     ```  
   - Use the same Gotify API credentials as above  
   - Connect MCP Trigger node output to this node input  
   - Position center-bottom

5. **Create Gotify Tool Node for "Get many messages":**  
   - Add a third **Gotify Tool** node  
   - Set operation to `"getAll"`  
   - Set `limit` parameter with expression:  
     ```
     {{$fromAI('Limit', '', 'number')}}
     ```  
   - Set `returnAll` parameter with expression:  
     ```
     {{$fromAI('Return_All', '', 'boolean')}}
     ```  
   - Use the same Gotify API credentials  
   - Connect MCP Trigger node output to this node input  
   - Position right-bottom

6. **Create Sticky Note for Section Label:**  
   - Add Sticky Note node with content:  
     ```
     ## Message
     ```  
   - Position above the three Gotify Tool nodes

7. **Final Steps:**  
   - Save the workflow  
   - Activate the workflow to enable the webhook  
   - Copy the webhook URL from the MCP Trigger node for external AI agent integration  
   - Test each operation by sending appropriate MCP requests with parameters matching the AI input keys (`Message`, `Message_Id`, `Limit`, `Return_All`)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| AI agents automatically populate parameters using the `$fromAI()` expression, simplifying integration and parameter passing. | Workflow feature highlight; enables zero-configuration calls from AI-driven environments.                            |
| Official n8n MCP Trigger documentation and Gotify Tool integration info available at: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Primary documentation source for MCP and Gotify Tool nodes.                                                         |
| Discord community available for MCP integration help: https://discord.me/cfomodz                                              | Direct support channel for customization and troubleshooting.                                                      |
| Ensure Gotify API credentials are correctly configured once; other Gotify Tool nodes will reuse these credentials automatically. | Prevents authentication errors; follow setup instructions carefully.                                                |
| The workflow uses expression-based dynamic input, so invalid or missing AI parameters may cause operation failures.            | Important for error handling and testing; validate AI inputs before production use.                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.

---