🛠️ Mattermost Tool MCP Server 💪 all 19 operations

https://n8nworkflows.xyz/workflows/----mattermost-tool-mcp-server----all-19-operations-5186


# 🛠️ Mattermost Tool MCP Server 💪 all 19 operations

### 1. Workflow Overview

This workflow titled **🛠️ Mattermost Tool MCP Server 💪 all 19 operations** is designed to serve as a comprehensive interface for interacting with a Mattermost server via 19 distinct operations. It is triggered by an MCP (Mattermost Command Protocol) event and can handle a wide range of user and channel management tasks within Mattermost.

The workflow logically divides into these key blocks:

- **1.1 Trigger Reception:** Initiates workflow execution upon receiving a command/event from Mattermost via the MCP trigger node.
- **1.2 Channel Management Operations:** Nodes to create, delete, search, restore, and manipulate channels and their memberships.
- **1.3 Message Management Operations:** Nodes to post, delete, and manage messages (including ephemeral messages and reactions).
- **1.4 User Management Operations:** Nodes to create, deactivate, invite users, and retrieve user information.
- **1.5 Reaction Management Operations:** Nodes to create, delete, and fetch reactions on messages.

Each operation node is directly connected to the MCP trigger node, indicating that the workflow routes commands received from Mattermost to the corresponding Mattermost API operation node.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Reception

- **Overview:**  
  This block receives and triggers the workflow execution upon an MCP command from Mattermost.

- **Nodes Involved:**  
  - Mattermost Tool MCP Server

- **Node Details:**

  - **Mattermost Tool MCP Server**  
    - **Type:** MCP Trigger (part of the langchain MCP integration nodes)  
    - **Technical Role:** Entry point for the workflow, listens for Mattermost MCP commands via webhook  
    - **Configuration:** Uses a unique webhook ID to receive requests from Mattermost  
    - **Input/Output:** No input connections; outputs to all operation nodes via the `ai_tool` connection  
    - **Version Requirements:** Requires n8n version supporting MCP Trigger nodes  
    - **Failure Types:** Webhook registration failure, network issues, invalid MCP commands  
    - **Notes:** Central to routing commands to various Mattermost operations  

#### 2.2 Channel Management Operations

- **Overview:**  
  Nodes in this block handle channels: creating, deleting, restoring, searching channels, and managing channel membership.

- **Nodes Involved:**  
  - Create a channel  
  - Delete a channel  
  - Restore a soft-deleted channel  
  - Search for a channel  
  - Get a page of members for a channel  
  - Add a user to a channel  
  - Get statistics for a channel

- **Node Details:**

  - **Create a channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Creates a new channel in Mattermost  
    - **Configuration:** Parameters are dynamically set based on MCP command inputs  
    - **Connections:** Input from MCP Trigger, no further outputs  
    - **Failures:** API errors due to invalid channel data, permission errors  

  - **Delete a channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Deletes a specified channel  
    - **Configuration:** Channel ID or name passed dynamically  
    - **Failures:** Channel not found, insufficient permissions  

  - **Restore a soft-deleted channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Restores a previously soft-deleted channel  
    - **Failures:** Channel not in soft-deleted state, permission issues  

  - **Search for a channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Searches a channel by criteria (e.g., name)  
    - **Failures:** No matching channel found  

  - **Get a page of members for a channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Retrieves members in a channel, supports pagination  
    - **Failures:** Channel not found, pagination errors  

  - **Add a user to a channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Adds a user to a channel specified by user and channel IDs  
    - **Failures:** User already a member, invalid IDs  

  - **Get statistics for a channel**  
    - **Type:** Mattermost Tool  
    - **Role:** Retrieves statistics such as post counts or member counts for a channel  
    - **Failures:** Invalid channel ID  

#### 2.3 Message Management Operations

- **Overview:**  
  This block manages messages in Mattermost channels, including posting, deleting, and sending ephemeral messages.

- **Nodes Involved:**  
  - Post a message  
  - Delete a message  
  - Post an ephemeral message

- **Node Details:**

  - **Post a message**  
    - **Type:** Mattermost Tool  
    - **Role:** Posts a message to a channel  
    - **Configuration:** Message content and channel ID passed dynamically  
    - **Failures:** Invalid channel, message content errors  

  - **Delete a message**  
    - **Type:** Mattermost Tool  
    - **Role:** Deletes a specified message by ID  
    - **Failures:** Message not found, permission errors  

  - **Post an ephemeral message**  
    - **Type:** Mattermost Tool  
    - **Role:** Posts a temporary message visible only to a specific user  
    - **Failures:** Invalid user or channel references  

#### 2.4 Reaction Management Operations

- **Overview:**  
  Nodes in this block handle reactions on messages, including creation, deletion, and retrieval of multiple reactions.

- **Nodes Involved:**  
  - Create a reaction  
  - Delete a reaction  
  - Get many reactions

- **Node Details:**

  - **Create a reaction**  
    - **Type:** Mattermost Tool  
    - **Role:** Adds a reaction emoji to a message  
    - **Failures:** Invalid emoji, message not found  

  - **Delete a reaction**  
    - **Type:** Mattermost Tool  
    - **Role:** Removes a reaction from a message  
    - **Failures:** Reaction not found  

  - **Get many reactions**  
    - **Type:** Mattermost Tool  
    - **Role:** Retrieves reactions on a message, possibly paginated  
    - **Failures:** Message not found  

#### 2.5 User Management Operations

- **Overview:**  
  This block manages user accounts, including creating, deactivating, inviting users, and fetching user data.

- **Nodes Involved:**  
  - Create a user  
  - Deactivate a user  
  - Invite a user  
  - Get a user by email  
  - Get a user by ID  
  - Get many users

- **Node Details:**

  - **Create a user**  
    - **Type:** Mattermost Tool  
    - **Role:** Creates a new user in Mattermost with specified details  
    - **Failures:** Validation errors, email conflicts  

  - **Deactivate a user**  
    - **Type:** Mattermost Tool  
    - **Role:** Deactivates a user account by ID  
    - **Failures:** User not found, permission issues  

  - **Invite a user**  
    - **Type:** Mattermost Tool  
    - **Role:** Sends an invite to a user via email  
    - **Failures:** Email invalid, user already invited  

  - **Get a user by email**  
    - **Type:** Mattermost Tool  
    - **Role:** Retrieves user info by email address  
    - **Failures:** User not found  

  - **Get a user by ID**  
    - **Type:** Mattermost Tool  
    - **Role:** Retrieves user info by user ID  
    - **Failures:** User not found  

  - **Get many users**  
    - **Type:** Mattermost Tool  
    - **Role:** Retrieves multiple users, supports pagination  
    - **Failures:** Pagination or permission errors  

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                 | Input Node(s)           | Output Node(s)          | Sticky Note                      |
|----------------------------------|----------------------------------|--------------------------------|------------------------|-------------------------|---------------------------------|
| Workflow Overview 0              | Sticky Note                      | Documentation note             | —                      | —                       |                                 |
| Mattermost Tool MCP Server       | MCP Trigger                     | Workflow trigger from Mattermost MCP commands | —                      | All Mattermost Tool nodes |                                 |
| Add a user to a channel          | Mattermost Tool                 | Adds user to channel           | Mattermost Tool MCP Server | —                       |                                 |
| Create a channel                 | Mattermost Tool                 | Creates a channel              | Mattermost Tool MCP Server | —                       |                                 |
| Delete a channel                 | Mattermost Tool                 | Deletes a channel              | Mattermost Tool MCP Server | —                       |                                 |
| Get a page of members for a channel | Mattermost Tool             | Lists members of a channel     | Mattermost Tool MCP Server | —                       |                                 |
| Restore a soft-deleted channel   | Mattermost Tool                 | Restores soft-deleted channel  | Mattermost Tool MCP Server | —                       |                                 |
| Search for a channel             | Mattermost Tool                 | Searches channels              | Mattermost Tool MCP Server | —                       |                                 |
| Get statistics for a channel     | Mattermost Tool                 | Retrieves channel statistics   | Mattermost Tool MCP Server | —                       |                                 |
| Sticky Note 1                   | Sticky Note                      | Documentation note             | —                      | —                       |                                 |
| Delete a message                | Mattermost Tool                 | Deletes a message              | Mattermost Tool MCP Server | —                       |                                 |
| Post a message                 | Mattermost Tool                 | Posts a message                | Mattermost Tool MCP Server | —                       |                                 |
| Post an ephemeral message       | Mattermost Tool                 | Posts ephemeral user message   | Mattermost Tool MCP Server | —                       |                                 |
| Sticky Note 2                   | Sticky Note                      | Documentation note             | —                      | —                       |                                 |
| Create a reaction               | Mattermost Tool                 | Creates a reaction on message  | Mattermost Tool MCP Server | —                       |                                 |
| Delete a reaction               | Mattermost Tool                 | Deletes a reaction             | Mattermost Tool MCP Server | —                       |                                 |
| Get many reactions              | Mattermost Tool                 | Retrieves multiple reactions   | Mattermost Tool MCP Server | —                       |                                 |
| Sticky Note 3                   | Sticky Note                      | Documentation note             | —                      | —                       |                                 |
| Create a user                  | Mattermost Tool                 | Creates a user account         | Mattermost Tool MCP Server | —                       |                                 |
| Deactivate a user              | Mattermost Tool                 | Deactivates a user             | Mattermost Tool MCP Server | —                       |                                 |
| Get a user by email            | Mattermost Tool                 | Retrieves user by email        | Mattermost Tool MCP Server | —                       |                                 |
| Get a user by ID               | Mattermost Tool                 | Retrieves user by ID           | Mattermost Tool MCP Server | —                       |                                 |
| Get many users                 | Mattermost Tool                 | Retrieves multiple users       | Mattermost Tool MCP Server | —                       |                                 |
| Invite a user                 | Mattermost Tool                 | Invites a user via email       | Mattermost Tool MCP Server | —                       |                                 |
| Sticky Note 4                   | Sticky Note                      | Documentation note             | —                      | —                       |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Node Type: `Mattermost Tool MCP Server` (MCP Trigger)  
   - Configure webhook with a unique webhook ID registered with Mattermost MCP integration.  
   - No parameters required.

2. **Create Mattermost Tool Nodes for Each Operation**  
   - For each Mattermost API operation, create a `Mattermost Tool` node with the following names and roles:  
     - Create a user  
     - Invite a user  
     - Get many users  
     - Post a message  
     - Create a channel  
     - Delete a channel  
     - Delete a message  
     - Get a user by ID  
     - Create a reaction  
     - Deactivate a user  
     - Delete a reaction  
     - Get many reactions  
     - Get a user by email  
     - Search for a channel  
     - Add a user to a channel  
     - Post an ephemeral message  
     - Get statistics for a channel  
     - Restore a soft-deleted channel  
     - Get a page of members for a channel  

3. **Connect the MCP Trigger Node to All Mattermost Tool Nodes**  
   - Each Mattermost Tool node receives input from the `Mattermost Tool MCP Server` node using the connection named `ai_tool`.  
   - No further downstream connections are needed as each node handles an individual operation.

4. **Configure Each Mattermost Tool Node**  
   - Set the operation type inside the node matching its name (e.g., "Create a user" node sets operation to Create User).  
   - Parameters such as user details, channel IDs, message contents, reaction emoji, etc., should be set dynamically using expressions that parse the incoming MCP command data.  
   - Credentials: Assign valid Mattermost API credentials for all Mattermost Tool nodes. This usually requires a Personal Access Token or OAuth2 access token with permissions for all user, channel, message, and reaction management scopes.

5. **Add Sticky Notes**  
   - Add sticky notes to document the workflow sections and nodes for clarity. These notes do not affect execution.

6. **Save and Activate the Workflow**  
   - Ensure webhook URL is correctly registered with Mattermost MCP integration.  
   - Test each operation via Mattermost commands routed through the MCP integration.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                |
|------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow covers all 19 core Mattermost operations exposed via n8n Mattermost Tool. | Workflow description                           |
| MCP (Mattermost Command Protocol) Trigger integration requires webhook setup in Mattermost. | Mattermost Docs: https://docs.mattermost.com/ |
| Ensure the Mattermost access token used has all required scopes for user, channel, message, and reaction management. | Best practice for API credential setup        |
| For detailed Mattermost API docs refer to: https://api.mattermost.com/       | Official Mattermost API documentation          |
| This workflow can be extended by adding error handling nodes such as Error Trigger or Function nodes for custom error responses. | Recommended improvement                        |

---

**Disclaimer:** The provided text exclusively originates from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.