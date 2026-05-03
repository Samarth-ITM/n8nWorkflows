Control Discord Bot with Natural Language via MCP Server

https://n8nworkflows.xyz/workflows/control-discord-bot-with-natural-language-via-mcp-server-3946


# Control Discord Bot with Natural Language via MCP Server

### 1. Workflow Overview

This workflow, titled **"Control Discord Bot with Natural Language via MCP Server"**, provides a straightforward MCP (Multi-Client Platform) server interface for controlling a Discord bot through natural language commands. Its main purpose is to enable users or other n8n workflows to command the Discord bot to perform actions such as sending messages, managing roles, and retrieving server information — all via conversational input.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception (MCP Trigger)**: Captures natural language commands sent via MCP clients.
- **1.2 Server and User Data Retrieval**: Retrieves Discord server IDs, channels, and members relevant for commands.
- **1.3 Message Sending and Interaction**: Sends messages or direct messages (DMs) to channels or users, optionally waiting for replies.
- **1.4 Role Management**: Adds or removes roles from server members programmatically.
- **1.5 Informational Sticky Notes**: Provides context and usage notes embedded inside the workflow for maintainers and users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception (MCP Trigger)

- **Overview:**  
  This block serves as the entry point for the workflow, receiving natural language commands via an MCP client interface. It triggers downstream nodes to process and execute commands on Discord.

- **Nodes Involved:**  
  - Discord MCP Server Trigger

- **Node Details:**  
  - **Discord MCP Server Trigger**  
    - *Type:* MCP Trigger (Langchain MCP Trigger)  
    - *Role:* Listens for incoming MCP client requests containing natural language instructions to control the Discord bot.  
    - *Configuration:* Uses a unique webhook path to receive requests (`404f083e-f3f4-4358-83ef-9804099ee253`).  
    - *Input:* External MCP client requests (natural language commands).  
    - *Output:* Passes parsed command data (e.g., server ID, user ID, message content) to connected Discord action nodes.  
    - *Version Requirements:* Requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger`.  
    - *Edge Cases:*  
      - Failure to receive or parse MCP input correctly can halt workflow execution.  
      - Invalid or incomplete command parameters may cause downstream API errors.

---

#### 1.2 Server and User Data Retrieval

- **Overview:**  
  Retrieves data about Discord servers, channels, and members accessible to the bot. This data is essential for contextualizing commands (e.g., sending messages to the right channel or managing roles for valid users).

- **Nodes Involved:**  
  - Get Discord Server IDs  
  - Get channels of server by server ID  
  - Get members of server by server ID

- **Node Details:**  
  - **Get Discord Server IDs**  
    - *Type:* HTTP Request (Discord API call)  
    - *Role:* Retrieves a list of all Discord servers (guilds) the bot is currently a member of by calling Discord’s `/users/@me/guilds` endpoint directly.  
    - *Configuration:* Uses bot’s Discord credentials for authentication via OAuth2.  
    - *Input:* Triggered via MCP Server Trigger node outputs.  
    - *Output:* List of server IDs for use in subsequent nodes.  
    - *Notes:* This is a custom API call, not a built-in Discord Tool node, allowing extended functionality beyond n8n’s presets.  
    - *Edge Cases:*  
      - Discord API rate limits or authentication failures.  
      - Bot not being in any servers returns empty results.

  - **Get channels of server by server ID**  
    - *Type:* Discord Tool Node  
    - *Role:* Fetches all channels from a specified Discord server using its ID.  
    - *Configuration:* Takes `guildId` dynamically from MCP input. Supports returning all channels or a subset controlled by a boolean flag.  
    - *Input:* Receives server ID from MCP trigger or preceding nodes.  
    - *Output:* List of channels for message targeting or validation.  
    - *Edge Cases:*  
      - Access denied if bot lacks permissions.  
      - Empty channel list if server has none or API issues.

  - **Get members of server by server ID**  
    - *Type:* Discord Tool Node  
    - *Role:* Retrieves members of a specified Discord server.  
    - *Configuration:* Accepts parameters for pagination (`after`) and return-all flag from MCP input.  
    - *Input:* Server ID from MCP trigger node.  
    - *Output:* List of member user IDs and details.  
    - *Edge Cases:*  
      - Pagination or large member count may cause delays or timeouts.  
      - Permission restrictions may limit visibility.

---

#### 1.3 Message Sending and Interaction

- **Overview:**  
  This block handles sending messages or DMs to channels or users, optionally waiting for human replies to enable HITL (Human-In-The-Loop) interactions.

- **Nodes Involved:**  
  - Send Discord Message to Channel  
  - Send to Channel and Wait for Reply  
  - Send DM to User  
  - Send DM and Wait for reply

- **Node Details:**  
  - **Send Discord Message to Channel**  
    - *Type:* Discord Tool Node  
    - *Role:* Sends a simple message to a specified Discord channel.  
    - *Configuration:* Uses dynamic expressions to set message content, guild ID, and channel ID from MCP input.  
    - *Input:* Triggered by MCP commands.  
    - *Output:* Confirmation of message sent.  
    - *Edge Cases:*  
      - Channel ID invalid or bot lacks send permissions.  
      - Message content empty or malformed.

  - **Send to Channel and Wait for Reply**  
    - *Type:* Discord Tool Node  
    - *Role:* Sends a message to a channel and waits for a free-text reply from any user.  
    - *Configuration:* Similar dynamic inputs as above, with operation set to `sendAndWait` and response type `freeText`.  
    - *Input:* MCP trigger output.  
    - *Output:* Captures user reply for further processing if needed.  
    - *Edge Cases:*  
      - User fails to reply within timeout.  
      - Multiple replies causing ambiguity.

  - **Send DM to User**  
    - *Type:* Discord Tool Node  
    - *Role:* Sends a direct message to a specific user.  
    - *Configuration:* Uses user ID, server ID, and message content dynamically from MCP input.  
    - *Input:* MCP trigger output.  
    - *Output:* Confirmation of DM sent.  
    - *Edge Cases:*  
      - User privacy settings block DMs.  
      - Invalid user ID.

  - **Send DM and Wait for reply**  
    - *Type:* Discord Tool Node  
    - *Role:* Sends a DM and waits for a free-text reply from the user.  
    - *Configuration:* Similar dynamic inputs, operation `sendAndWait`.  
    - *Input:* MCP trigger output.  
    - *Output:* Captures user reply for use in workflow.  
    - *Edge Cases:*  
      - No reply received, leading to timeout.  
      - User blocks bot DMs.

---

#### 1.4 Role Management

- **Overview:**  
  Enables programmatic addition or removal of roles for users in the Discord server, allowing automation of moderation or permission changes.

- **Nodes Involved:**  
  - Add Role To Member  
  - Remove Role from member

- **Node Details:**  
  - **Add Role To Member**  
    - *Type:* Discord Tool Node  
    - *Role:* Adds a specified role to a user in a given server.  
    - *Configuration:* Dynamically sets `role`, `userId`, and `guildId` from MCP input.  
    - *Input:* Triggered by MCP commands requesting role addition.  
    - *Output:* Confirmation of role added.  
    - *Edge Cases:*  
      - Role or user ID invalid.  
      - Bot lacks permissions to manage roles.  
      - Role hierarchy restrictions.

  - **Remove Role from member**  
    - *Type:* Discord Tool Node  
    - *Role:* Removes a specified role from a user in a given server.  
    - *Configuration:* Same dynamic input parameters as Add Role node.  
    - *Input:* MCP trigger commands requesting role removal.  
    - *Output:* Confirmation of role removal.  
    - *Edge Cases:*  
      - Same as Add Role node, plus handling if user does not have the role.

---

#### 1.5 Informational Sticky Notes

- **Overview:**  
  These sticky notes provide explanations, usage tips, and context for maintainers or users reviewing the workflow.

- **Nodes Involved:**  
  - Sticky Note (near Get Discord Server IDs)  
  - Sticky Note1 (near Send Discord Message to Channel)  
  - Sticky Note2 (near Send DM to User)  
  - Sticky Note3 (near Role Management nodes)  
  - Sticky Note4 (near Get members of server)  
  - Sticky Note5 (near Get channels of server)

- **Node Details:**  
  - *Role:* Non-executable, purely documentation nodes.  
  - *Content:* Detailed notes clarifying how nodes operate, why custom API calls are used, and potential usage scenarios.  
  - *Use:* Helpful for workflow maintainers to understand rationale and best practices.

---

### 3. Summary Table

| Node Name                       | Node Type                      | Functional Role                      | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                                  |
|--------------------------------|--------------------------------|------------------------------------|-----------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Discord MCP Server Trigger      | MCP Trigger (Langchain)         | Entry point: receives natural language commands | External MCP clients         | All Discord action nodes       |                                                                                                                              |
| Get Discord Server IDs          | HTTP Request                   | Retrieves all Discord servers the bot is in    | Discord MCP Server Trigger   | Get channels, Get members      | This gets all of the servers that your discord bot is currently in. If you have a bot in more than one server... (full note) |
| Get channels of server by server ID | Discord Tool Node             | Retrieves all channels of a given server        | Discord MCP Server Trigger   | Send Discord Message to Channel, Send to Channel and Wait for Reply | Once we have the server ID of the server we want to interact with, we can grab all channels the bot can see.                 |
| Get members of server by server ID | Discord Tool Node             | Retrieves members of a given server              | Discord MCP Server Trigger   | Add Role To Member, Remove Role from member, Send DM nodes | Once we have the server ID of the server we want to interact with, we can grab all members of the server the bot can see.    |
| Send Discord Message to Channel | Discord Tool Node              | Sends a basic message to a channel                | Discord MCP Server Trigger, Get channels node |                               | These nodes either send a basic message from your bot to a channel or sends a message and waits for a response from a human (HITL). |
| Send to Channel and Wait for Reply | Discord Tool Node              | Sends a message to a channel and waits for reply | Discord MCP Server Trigger, Get channels node |                               | These nodes either send a basic message from your bot to a channel or sends a message and waits for a response from a human (HITL). |
| Send DM to User                | Discord Tool Node              | Sends a direct message to a user                   | Discord MCP Server Trigger, Get members node |                               | These nodes either send a basic message from your bot to a user via DM or sends a DM and waits for a response from the human (HITL). |
| Send DM and Wait for reply     | Discord Tool Node              | Sends a DM and waits for reply from user          | Discord MCP Server Trigger, Get members node |                               | These nodes either send a basic message from your bot to a user via DM or sends a DM and waits for a response from the human (HITL). |
| Add Role To Member             | Discord Tool Node              | Adds a role to a user                               | Discord MCP Server Trigger, Get members node |                               | Programmatic role addition and removal are just two examples of the many, many tools/API calls that you can make to Discord...  |
| Remove Role from member        | Discord Tool Node              | Removes a role from a user                          | Discord MCP Server Trigger, Get members node |                               | Programmatic role addition and removal are just two examples of the many, many tools/API calls that you can make to Discord...  |
| Sticky Note                   | Sticky Note                   | Workflow documentation                            |                             |                               | This gets all of the servers that your discord bot is currently in...                                                        |
| Sticky Note1                  | Sticky Note                   | Workflow documentation                            |                             |                               | These nodes either send a basic message from your bot to a channel or sends a message and waits for a response from a human (HITL). |
| Sticky Note2                  | Sticky Note                   | Workflow documentation                            |                             |                               | These nodes either send a basic message from your bot to a user via DM or sends a DM and waits for a response from the human (HITL). |
| Sticky Note3                  | Sticky Note                   | Workflow documentation                            |                             |                               | Programmatic role addition and removal are just two examples of the many, many tools/API calls that you can make to Discord...  |
| Sticky Note4                  | Sticky Note                   | Workflow documentation                            |                             |                               | Once we have the server ID of the server we want to interact with, we can grab all members of the server...                   |
| Sticky Note5                  | Sticky Note                   | Workflow documentation                            |                             |                               | Once we have the server ID of the server we want to interact with, we can grab all channels the bot can see.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add a **Langchain MCP Trigger** node.  
   - Configure the webhook path (e.g., `404f083e-f3f4-4358-83ef-9804099ee253`).  
   - This node receives natural language commands from MCP clients.

2. **Create the "Get Discord Server IDs" Node**  
   - Add an **HTTP Request** node.  
   - Set method to `GET`.  
   - URL: `https://discord.com/api/v10/users/@me/guilds`  
   - Authentication: Use Discord Bot OAuth2 credentials (create or select existing).  
   - Purpose: Retrieve all servers the bot is a member of.  
   - Connect input from MCP Trigger node.

3. **Create the "Get channels of server by server ID" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `channel`  
   - Operation: `getAll`  
   - Guild ID: Use expression to get from MCP input or previous node output (e.g., `{{$json["Server"]}}`).  
   - Return All: Use boolean parameter from MCP input or set to `true` for all channels.  
   - Connect input from MCP Trigger node.

4. **Create the "Get members of server by server ID" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `member`  
   - Operation: `getAll` (or appropriate)  
   - Guild ID: Use expression from MCP input.  
   - After: Use expression or leave blank.  
   - Return All: Boolean from MCP input or set to `true`.  
   - Connect input from MCP Trigger node.

5. **Create the "Send Discord Message to Channel" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `message`  
   - Operation: `create` (send message)  
   - Channel ID, Guild ID, Content: Set dynamically using MCP input expressions (e.g., `{{$json["Channel"]}}`, `{{$json["Server"]}}`, `{{$json["Message"]}}`).  
   - Connect input from MCP Trigger node.

6. **Create the "Send to Channel and Wait for Reply" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `message`  
   - Operation: `sendAndWait`  
   - Response Type: `freeText`  
   - Channel ID, Guild ID, Message: Set dynamically from MCP input.  
   - Connect input from MCP Trigger node.

7. **Create the "Send DM to User" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `message`  
   - Operation: `create` (send DM)  
   - Send To: `user`  
   - User ID, Guild ID, Content: Use MCP input expressions.  
   - Connect input from MCP Trigger node.

8. **Create the "Send DM and Wait for reply" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `message`  
   - Operation: `sendAndWait`  
   - Response Type: `freeText`  
   - Send To: `user`  
   - User ID, Guild ID, Message: Use MCP input expressions.  
   - Connect input from MCP Trigger node.

9. **Create the "Add Role To Member" Node**  
   - Add a **Discord Tool** node.  
   - Resource: `member`  
   - Operation: `roleAdd`  
   - Role ID, User ID, Guild ID: Use MCP input expressions.  
   - Connect input from MCP Trigger node.

10. **Create the "Remove Role from member" Node**  
    - Add a **Discord Tool** node.  
    - Resource: `member`  
    - Operation: `roleRemove`  
    - Role ID, User ID, Guild ID: Use MCP input expressions.  
    - Connect input from MCP Trigger node.

11. **Configure Credentials**  
    - For all Discord Tool nodes and HTTP Request node, assign the same **Discord Bot API** credentials (OAuth2 with the bot token).  
    - Ensure the bot token has required permissions: reading servers, channels, members, sending messages, managing roles.

12. **Add Sticky Notes** (optional but recommended)  
    - Add sticky notes near nodes to document functionality and usage tips as per the original workflow.

13. **Set Execution Order**  
    - The MCP Trigger node is the root trigger. All other nodes receive input via the MCP trigger’s AI tool output connections.  
    - No sequential connections are strictly required beyond this as nodes are invoked contextually based on MCP input values.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                        | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow uses a custom API call for fetching Discord servers the bot is in, showcasing n8n’s flexibility beyond built-in nodes.                                                                                                                                                              | Highlighted in "Get Discord Server IDs" node sticky note.                                                     |
| You can control your Discord bot via any MCP client, including other n8n workflows, enabling powerful integrations and automation.                                                                                                                                                              | Workflow description and usage notes.                                                                         |
| Programmatic role management can be extended to moderation tasks such as spam control or abuse monitoring by automating role permissions.                                                                                                                                                        | Sticky Note3 content near role management nodes.                                                              |
| The "sendAndWait" operations enable Human-In-The-Loop interactions, allowing the bot to await user responses in channels or DMs, enhancing conversational workflows.                                                                                                                             | Sticky Note1 and Sticky Note2 content near message nodes.                                                     |
| For comprehensive examples and quickstarts, refer to the author’s simple Discord MCP Server example using 4o to send messages.                                                                                                                                                                   | Referenced in workflow description (no direct link provided).                                                 |
| Ensure the Discord bot OAuth2 credentials have appropriate scopes and permissions for all intended operations to avoid failures due to insufficient access rights.                                                                                                                                | General best practice for Discord Bot API usage in n8n.                                                       |
| MCP trigger node requires n8n version compatible with `@n8n/n8n-nodes-langchain.mcpTrigger` package.                                                                                                                                                                                              | Version-specific note for MCP Trigger node.                                                                   |

---

This reference document covers the entire structure, configuration, and operational logic of the provided workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.