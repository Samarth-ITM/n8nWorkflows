Search the Web with MCP-based Brave Search Engine on Telegram

https://n8nworkflows.xyz/workflows/search-the-web-with-mcp-based-brave-search-engine-on-telegram-3231


# Search the Web with MCP-based Brave Search Engine on Telegram

### 1. Workflow Overview

This workflow enables Telegram users to perform web searches using the Brave search engine via MCP Client integration. By sending a message starting with the command `/brave` followed by a search query, the workflow processes the input, executes a Brave search tool, and returns the search results directly in the Telegram chat.

**Target Use Cases:**  
- Quick, private web searches from within Telegram without switching apps.  
- Automating Brave search interactions for users who prefer chat-based commands.  
- Developers or power users wanting to integrate Brave search capabilities into Telegram bots.

**Logical Blocks:**

- **1.1 Input Reception and Filtering:**  
  Captures Telegram messages and filters those starting with `/brave`.

- **1.2 Query Extraction and Cleaning:**  
  Extracts the raw search query by removing the `/brave` command prefix.

- **1.3 Brave Tool Discovery:**  
  Retrieves the list of available Brave search tools via MCP Client.

- **1.4 Brave Tool Execution:**  
  Executes the first Brave search tool with the cleaned query.

- **1.5 Result Delivery:**  
  Sends the search results back to the Telegram user as a message.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Filtering

**Overview:**  
This block listens for incoming Telegram messages and filters only those that start with the `/brave` command, ensuring the workflow processes only relevant queries.

**Nodes Involved:**  
- Get Message (Telegram Trigger)  
- Search with Brave? (If node)  
- Get Text (Set node)

**Node Details:**

- **Get Message**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (updates of type "message").  
  - Configuration: Uses Telegram credentials; triggers on all incoming messages.  
  - Inputs: None (trigger node)  
  - Outputs: Emits message data including text and sender info.  
  - Edge Cases: Telegram API downtime, webhook misconfiguration, unauthorized bot access.  
  - Sticky Note: None directly, but related to the note about preliminary setup.

- **Search with Brave?**  
  - Type: If  
  - Role: Filters messages to only those starting with `/brave `.  
  - Configuration: Checks if `message.text` starts with `/brave ` (case-sensitive, strict validation).  
  - Inputs: From Get Message  
  - Outputs: True branch proceeds; False branch stops workflow.  
  - Edge Cases: Messages without text, commands with different casing, partial matches.  
  - Sticky Note: "the search only occurs when the command \"/brave\" is present in the message"

- **Get Text**  
  - Type: Set  
  - Role: Extracts the `text` field from the Telegram message for further processing.  
  - Configuration: Assigns `text` = `{{$json.message.text}}`.  
  - Inputs: True branch of If node  
  - Outputs: Passes the extracted text forward.  
  - Edge Cases: Missing or empty text fields.  
  - Sticky Note: None

---

#### 1.2 Query Extraction and Cleaning

**Overview:**  
Removes the `/brave` command prefix from the message text, isolating the user's search query.

**Nodes Involved:**  
- Clean query (Code node)

**Node Details:**

- **Clean query**  
  - Type: Code (JavaScript)  
  - Role: Processes each input item to strip `/brave ` from the start of the message text and stores the remainder as `query`.  
  - Configuration:  
    ```js
    for (const item of $input.all()) {
      const originalText = item.json.text;
      const query = originalText.replace("/brave ", "");
      item.json.query = query;
    }
    return $input.all();
    ```  
  - Inputs: From Get Text  
  - Outputs: Items enriched with `query` field containing the cleaned search query.  
  - Edge Cases: Messages that only contain `/brave ` with no query, multiple spaces, or malformed input.  
  - Sticky Note: "I clean the message by removing the \"/brave\" command"

---

#### 1.3 Brave Tool Discovery

**Overview:**  
Retrieves the list of available Brave search tools from the MCP Client to determine which tool to execute.

**Nodes Involved:**  
- List Brave Tools (MCP Client node)

**Node Details:**

- **List Brave Tools**  
  - Type: MCP Client  
  - Role: Calls MCP Client API to list all available Brave tools.  
  - Configuration: Uses MCP Client credentials configured with Brave API key. No additional parameters set.  
  - Inputs: From Clean query  
  - Outputs: JSON containing an array of tools under `tools`.  
  - Edge Cases: API key invalid or missing, network errors, empty tool list.  
  - Sticky Note: "Get all available Brave search tools"  
  - Credential Setup: Requires MCP Client credential with environment variable `BRAVE_API_KEY=your-api-key` as per sticky note instructions.

---

#### 1.4 Brave Tool Execution

**Overview:**  
Executes the first Brave tool from the list using the cleaned query as input, retrieving search results.

**Nodes Involved:**  
- Exec Brave tool (MCP Client node)

**Node Details:**

- **Exec Brave tool**  
  - Type: MCP Client  
  - Role: Executes the Brave tool with the user's query.  
  - Configuration:  
    - `toolName` dynamically set to the first tool name from the previous node: `={{ $json.tools[0].name }}`  
    - `operation` set to `executeTool`  
    - `toolParameters` JSON includes the query:  
      ```json
      {
        "query": "{{ $('Clean query').item.json.query }}"
      }
      ```  
  - Inputs: From List Brave Tools  
  - Outputs: Search results JSON under `result.content[0].text`.  
  - Edge Cases: Tool execution failure, invalid query, API rate limits, empty results.  
  - Sticky Note: "I get the search results"  
  - Credential: Same MCP Client credential as List Brave Tools.

---

#### 1.5 Result Delivery

**Overview:**  
Sends the search results back to the Telegram user as a formatted message.

**Nodes Involved:**  
- Send message (Telegram node)

**Node Details:**

- **Send message**  
  - Type: Telegram  
  - Role: Sends a Telegram message containing the search results.  
  - Configuration:  
    - `text` set to the first result text: `={{ $json.result.content[0].text }}`  
    - `chatId` dynamically set to the Telegram user ID from the original message: `={{ $('Get Message').item.json.message.from.id }}`  
    - `parse_mode` set to `HTML` for rich text formatting.  
  - Inputs: From Exec Brave tool  
  - Outputs: None (terminal node)  
  - Edge Cases: Telegram API errors, invalid chat ID, message length limits.  
  - Sticky Note: None

---

### 3. Summary Table

| Node Name         | Node Type            | Functional Role                          | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                      |
|-------------------|----------------------|----------------------------------------|---------------------|----------------------|------------------------------------------------------------------------------------------------|
| Get Message       | Telegram Trigger     | Receive Telegram messages               | None                | Search with Brave?    |                                                                                                |
| Search with Brave? | If                   | Filter messages starting with `/brave` | Get Message         | Get Text (true branch) | "the search only occurs when the command \"/brave\" is present in the message"                 |
| Get Text          | Set                   | Extract message text                    | Search with Brave?   | Clean query           |                                                                                                |
| Clean query       | Code                  | Remove `/brave` prefix from message    | Get Text            | List Brave Tools      | "I clean the message by removing the \"/brave\" command"                                       |
| List Brave Tools  | MCP Client             | Retrieve available Brave tools         | Clean query         | Exec Brave tool       | "Get all available Brave search tools"                                                         |
| Exec Brave tool   | MCP Client             | Execute Brave tool with query          | List Brave Tools     | Send message          | "I get the search results"                                                                      |
| Send message      | Telegram               | Send search results back to Telegram   | Exec Brave tool     | None                 |                                                                                                |
| Sticky Note       | Sticky Note            | Preliminary steps and setup instructions | None                | None                 | "Access to an n8n self-hosted instance and install the Community node \"n8n-nodes-mcp\". Please see this [easy guide](https://github.com/nerding-io/n8n-nodes-mcp)" |
| Sticky Note1      | Sticky Note            | MCP Brave tool credential setup guide  | None                | None                 | "In \"List Brave Tools\" create new credential as shown in this image ![image](https://github.com/nerding-io/n8n-nodes-mcp/raw/main/assets/credentials-envs.png) In Environment field set this value: BRAVE_API_KEY=your-api-key" |
| Sticky Note2      | Sticky Note            | Command usage note                      | None                | None                 | "the search only occurs when the command \"/brave\" is present in the message"                 |
| Sticky Note3      | Sticky Note            | Query cleaning explanation              | None                | None                 | "I clean the message by removing the \"/brave\" command"                                       |
| Sticky Note4      | Sticky Note            | Brave tools retrieval explanation       | None                | None                 | "Get all available Brave search tools"                                                         |
| Sticky Note5      | Sticky Note            | Brave tool execution explanation        | None                | None                 | "I get the search results"                                                                      |
| Sticky Note6      | Sticky Note            | Workflow purpose summary                 | None                | None                 | "This workflow is a powerful tool for automating interactions with Brave tools through Telegram, providing users with quick and easy access to information directly in their chat. This n8n workflow enables users to perform web searches directly from Telegram using the Brave search engine. By simply sending the command /brave followed by a query, the workflow retrieves search results from Brave and returns them as a Telegram message." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node ("Get Message")**  
   - Type: Telegram Trigger  
   - Configure credentials with your Telegram Bot API token.  
   - Set updates to listen for `message` events.  
   - Position: Start of workflow.

2. **Add If Node ("Search with Brave?")**  
   - Type: If  
   - Connect input from Telegram Trigger node.  
   - Condition: Check if `{{$json.message.text}}` starts with `/brave ` (case-sensitive).  
   - True branch proceeds; False branch ends workflow.

3. **Add Set Node ("Get Text")**  
   - Type: Set  
   - Connect input from True branch of If node.  
   - Add assignment: `text` = `{{$json.message.text}}`.  
   - This extracts the message text for processing.

4. **Add Code Node ("Clean query")**  
   - Type: Code (JavaScript)  
   - Connect input from Set node.  
   - Paste the following code to remove `/brave ` prefix:  
     ```js
     for (const item of $input.all()) {
       const originalText = item.json.text;
       const query = originalText.replace("/brave ", "");
       item.json.query = query;
     }
     return $input.all();
     ```
   - This node outputs the cleaned query.

5. **Add MCP Client Node ("List Brave Tools")**  
   - Type: MCP Client  
   - Connect input from Code node.  
   - Configure MCP Client credentials:  
     - Create a credential with environment variable `BRAVE_API_KEY=your-api-key`.  
     - Use the MCP Client credential configured for Brave.  
   - No additional parameters needed; this node lists available Brave tools.

6. **Add MCP Client Node ("Exec Brave tool")**  
   - Type: MCP Client  
   - Connect input from "List Brave Tools" node.  
   - Set parameters:  
     - `toolName`: `={{ $json.tools[0].name }}` (dynamically use the first tool).  
     - `operation`: `executeTool`  
     - `toolParameters`:  
       ```json
       {
         "query": "{{ $('Clean query').item.json.query }}"
       }
       ```  
   - Use the same MCP Client credential as previous node.

7. **Add Telegram Node ("Send message")**  
   - Type: Telegram  
   - Connect input from "Exec Brave tool" node.  
   - Configure credentials with your Telegram Bot API token.  
   - Set parameters:  
     - `text`: `={{ $json.result.content[0].text }}` (first search result text).  
     - `chatId`: `={{ $('Get Message').item.json.message.from.id }}` (send back to original user).  
     - `parse_mode`: `HTML` (to enable rich text formatting).  

8. **Connect all nodes in order:**  
   - Get Message → Search with Brave? (If) → True → Get Text → Clean query → List Brave Tools → Exec Brave tool → Send message

9. **Test the workflow:**  
   - Send a Telegram message starting with `/brave` followed by your search query.  
   - Verify the bot replies with Brave search results.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Access to an n8n self-hosted instance and install the Community node "n8n-nodes-mcp". Please see this easy guide.        | https://github.com/nerding-io/n8n-nodes-mcp                                                           |
| Get your Brave Search API Key here: https://brave.com/search/api/                                                        | Brave Search API documentation                                                                         |
| Telegram Bot Access Token is required for Telegram Trigger and Send Message nodes.                                       | Telegram BotFather                                                                                      |
| MCP Client Brave credential setup requires setting environment variable: `BRAVE_API_KEY=your-api-key`                    | Credential setup image: https://github.com/nerding-io/n8n-nodes-mcp/raw/main/assets/credentials-envs.png |
| The workflow only triggers on messages starting with `/brave` command to avoid unnecessary executions.                   | Sticky note in workflow                                                                                 |
| This workflow provides a fast, private, and integrated way to search the web from Telegram chats using Brave search.     | Sticky note summary                                                                                     |

---

This documentation fully describes the workflow structure, node configurations, and setup instructions to enable reproduction, modification, and troubleshooting by advanced users or automation agents.