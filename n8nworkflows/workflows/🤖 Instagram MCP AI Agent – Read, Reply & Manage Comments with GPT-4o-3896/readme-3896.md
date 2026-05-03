🤖 Instagram MCP AI Agent – Read, Reply & Manage Comments with GPT-4o

https://n8nworkflows.xyz/workflows/---instagram-mcp-ai-agent---read--reply---manage-comments-with-gpt-4o-3896


# 🤖 Instagram MCP AI Agent – Read, Reply & Manage Comments with GPT-4o

### 1. Workflow Overview

This workflow, titled **"🤖 Instagram MCP AI Agent – Read, Reply & Manage Comments with GPT-4o"**, is designed to automate intelligent Instagram engagement by leveraging AI and Meta’s Graph API. It targets social media teams, brands, agencies, and developers who want to automate comment management, reply generation, and direct messaging on Instagram Business accounts.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Mapping**: Receives chat messages via LangChain Chat Trigger, maps incoming data for AI processing.
- **1.2 AI Processing and Memory Management**: Uses GPT-4o via LangChain Agent and Chat Model with session memory to generate intelligent responses.
- **1.3 Instagram API Interaction via MCP Server**: Uses MCP Server nodes and HTTP Request tools to interact with Instagram’s Graph API for media search, comment fetching, replying, and sending direct messages.
- **1.4 Output and Communication**: Sends responses back through MCP Server SSE connection to the client.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Mapping

- **Overview:**  
  This block handles the reception of chat messages from users and prepares the data for AI processing by mapping inputs into a structured format.

- **Nodes Involved:**  
  - When chat message received  
  - Instagram Mapping

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point webhook that triggers the workflow upon receiving a chat message.  
    - Configuration: Default, listens for incoming chat messages via webhook.  
    - Inputs: External chat message via webhook.  
    - Outputs: Passes data to Instagram Mapping node.  
    - Edge Cases: Webhook failures, malformed input, missing required fields.  
    - Version: 1.1

  - **Instagram Mapping**  
    - Type: Set  
    - Role: Maps and structures incoming chat message data into variables usable by the AI Agent.  
    - Configuration: Likely sets variables such as Instagram ID, Access Token, user input text, session info.  
    - Inputs: Output from chat trigger node.  
    - Outputs: Passes structured data to AI Agent node.  
    - Edge Cases: Missing or invalid input data causing mapping errors.  
    - Version: 3.4

---

#### 2.2 AI Processing and Memory Management

- **Overview:**  
  This block processes the user input using GPT-4o via LangChain Agent, maintaining conversational context with a memory buffer, and generating AI-driven responses or instructions.

- **Nodes Involved:**  
  - AI Agent  
  - Chat Model  
  - Simple Memory

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Core AI logic node that interprets user input, decides which tools to invoke, and generates responses.  
    - Configuration: Uses Chat Model (GPT-4o) and Simple Memory for context.  
    - Inputs: Structured input from Instagram Mapping, memory from Simple Memory, tool outputs.  
    - Outputs: Commands to MCP Instagram tool, or responses back to client.  
    - Edge Cases: Model API errors, timeout, invalid tool invocation, memory overflow.  
    - Version: 1.9

  - **Chat Model**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides GPT-4o language model capabilities for the AI Agent.  
    - Configuration: OpenAI API key configured externally; model set to GPT-4o.  
    - Inputs: Prompts from AI Agent.  
    - Outputs: Text completions back to AI Agent.  
    - Edge Cases: API quota exceeded, authentication errors, network issues.  
    - Version: 1.2

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context window for the AI Agent to preserve session memory.  
    - Configuration: Default buffer window size; stores recent conversation history.  
    - Inputs: Conversation data from AI Agent.  
    - Outputs: Memory context to AI Agent.  
    - Edge Cases: Memory overflow, data corruption.  
    - Version: 1.3

---

#### 2.3 Instagram API Interaction via MCP Server

- **Overview:**  
  This block handles all Instagram API interactions through MCP Server, including searching media, fetching media details, retrieving comments, replying to comments, and sending direct messages.

- **Nodes Involved:**  
  - 🗂️ MCP Instagram (MCP Trigger)  
  - MCP Instagram (MCP Tool)  
  - Search Media (HTTP Request)  
  - Media Details (HTTP Request)  
  - Search Comment (HTTP Request)  
  - Reply Comment (HTTP Request)  
  - Send Direct Message (HTTP Request)

- **Node Details:**

  - **🗂️ MCP Instagram**  
    - Type: LangChain MCP Trigger  
    - Role: Listens for MCP Server events or triggers related to Instagram SSE communication.  
    - Configuration: Configured with webhook URL for MCP Server SSE endpoint.  
    - Inputs: External MCP Server SSE events.  
    - Outputs: Passes events to MCP Instagram tool node.  
    - Edge Cases: SSE connection drops, authentication failures.  
    - Version: 1

  - **MCP Instagram**  
    - Type: LangChain MCP Client Tool  
    - Role: Acts as a client tool for AI Agent to invoke Instagram API actions via MCP Server.  
    - Configuration: MCP Server URL configured (e.g., `https://yourdomain.com/mcp/server/instagram/sse`).  
    - Inputs: Commands from AI Agent or HTTP Request nodes.  
    - Outputs: Instagram API responses back to AI Agent.  
    - Edge Cases: Network errors, invalid MCP Server URL, API rate limits.  
    - Version: 1

  - **Search Media**  
    - Type: LangChain Tool HTTP Request  
    - Role: Searches recent media posts for the Instagram Business account using Instagram Graph API.  
    - Configuration: Uses Instagram ID and Access Token from mapped variables; HTTP GET request to `/media` endpoint.  
    - Inputs: Triggered by AI Agent via MCP Instagram tool.  
    - Outputs: Media list JSON to Media Details node.  
    - Edge Cases: API errors, expired tokens, empty media results.  
    - Version: 1.1

  - **Media Details**  
    - Type: LangChain Tool HTTP Request  
    - Role: Fetches detailed information about a specific media post (caption, media URL).  
    - Configuration: HTTP GET request to `/media/{media-id}` endpoint with access token.  
    - Inputs: Media ID from Search Media node.  
    - Outputs: Media details JSON to Search Comment node.  
    - Edge Cases: Invalid media ID, API errors.  
    - Version: 1.1

  - **Search Comment**  
    - Type: LangChain Tool HTTP Request  
    - Role: Retrieves comments and replies for a given media post.  
    - Configuration: HTTP GET request to `/comments` endpoint with pagination support.  
    - Inputs: Media ID from Media Details node.  
    - Outputs: Comments JSON to Reply Comment node or AI Agent.  
    - Edge Cases: Large comment volumes, API rate limits.  
    - Version: 1.1

  - **Reply Comment**  
    - Type: LangChain Tool HTTP Request  
    - Role: Posts AI-generated replies to specific comments on Instagram posts.  
    - Configuration: HTTP POST request to `/comments` endpoint with reply text and parent comment ID.  
    - Inputs: Reply text from AI Agent, comment ID from Search Comment node.  
    - Outputs: Confirmation of reply posted.  
    - Edge Cases: Permission errors, invalid comment IDs, API errors.  
    - Version: 1.1

  - **Send Direct Message**  
    - Type: LangChain Tool HTTP Request  
    - Role: Sends direct messages to followers who commented, using Instagram Messaging API.  
    - Configuration: HTTP POST request to `/messages` endpoint with recipient ID and message content.  
    - Inputs: User ID and message from AI Agent.  
    - Outputs: Confirmation of DM sent.  
    - Edge Cases: Messaging permissions, API limits, invalid user IDs.  
    - Version: 1.1

---

#### 2.4 Output and Communication

- **Overview:**  
  This block manages the communication back to the user via MCP Server SSE connection, delivering AI-generated responses and updates.

- **Nodes Involved:**  
  - MCP Instagram (MCP Trigger)  
  - MCP Instagram (MCP Tool)

- **Node Details:**

  - These nodes are shared with the Instagram API Interaction block, serving dual roles for input and output communication via MCP Server SSE.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)                | Output Node(s)              | Sticky Note                      |
|-------------------------|----------------------------------|----------------------------------------|-----------------------------|-----------------------------|---------------------------------|
| When chat message received | LangChain Chat Trigger           | Entry webhook for chat message input   | External webhook             | Instagram Mapping           |                                 |
| Instagram Mapping       | Set                              | Maps input data for AI processing      | When chat message received   | AI Agent                   |                                 |
| AI Agent                | LangChain Agent                  | Core AI logic and tool orchestration   | Instagram Mapping, Simple Memory, MCP Instagram tool | MCP Instagram tool, Simple Memory |                                 |
| Chat Model              | LangChain LM Chat OpenAI         | GPT-4o language model for AI Agent     | AI Agent                    | AI Agent                   |                                 |
| Simple Memory           | LangChain Memory Buffer Window   | Maintains session conversational memory| AI Agent                    | AI Agent                   |                                 |
| 🗂️ MCP Instagram        | LangChain MCP Trigger            | MCP Server SSE event listener           | External MCP SSE            | MCP Instagram tool         |                                 |
| MCP Instagram           | LangChain MCP Client Tool        | MCP Server client tool for Instagram API| AI Agent, HTTP Request nodes| AI Agent                   |                                 |
| Search Media            | LangChain Tool HTTP Request      | Searches recent Instagram media posts  | MCP Instagram tool          | Media Details              |                                 |
| Media Details           | LangChain Tool HTTP Request      | Fetches detailed media info             | Search Media                | Search Comment             |                                 |
| Search Comment          | LangChain Tool HTTP Request      | Retrieves comments and replies          | Media Details               | Reply Comment              |                                 |
| Reply Comment           | LangChain Tool HTTP Request      | Posts replies to Instagram comments     | Search Comment              | MCP Instagram tool         |                                 |
| Send Direct Message     | LangChain Tool HTTP Request      | Sends direct messages to followers      | AI Agent                   | MCP Instagram tool         |                                 |
| Sticky Note             | Sticky Note                     | Visual notes                           |                             |                             |                                 |
| Sticky Note1            | Sticky Note                     | Visual notes                           |                             |                             |                                 |
| Sticky Note2            | Sticky Note                     | Visual notes                           |                             |                             |                                 |
| Sticky Note3            | Sticky Note                     | Visual notes                           |                             |                             |                                 |
| Sticky Note4            | Sticky Note                     | Visual notes                           |                             |                             |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Configure it to listen for incoming chat messages via webhook (default settings).  
   - This node will serve as the entry point.

2. **Add a Set Node for Input Mapping**  
   - Add a **Set** node named `Instagram Mapping`.  
   - Connect `When chat message received` → `Instagram Mapping`.  
   - Configure to map incoming webhook data into variables such as:  
     - `instagramId` (Instagram Business Account ID)  
     - `accessToken` (Meta Graph API token)  
     - `userInput` (text message from user)  
     - `sessionId` or other session context variables.

3. **Add LangChain Chat Model Node**  
   - Add **LangChain LM Chat OpenAI** node named `Chat Model`.  
   - Configure with your OpenAI API credentials.  
   - Set model to GPT-4o.  
   - No special prompt needed here; it will be used by the AI Agent.

4. **Add LangChain Memory Buffer Node**  
   - Add **LangChain Memory Buffer Window** node named `Simple Memory`.  
   - Use default buffer window size to maintain conversation context.

5. **Add LangChain Agent Node**  
   - Add **LangChain Agent** node named `AI Agent`.  
   - Connect `Instagram Mapping` → `AI Agent` (main input).  
   - Connect `Chat Model` → `AI Agent` (ai_languageModel input).  
   - Connect `Simple Memory` → `AI Agent` (ai_memory input).  
   - Configure the agent to use the Chat Model and Memory Buffer.  
   - This node will orchestrate tool calls and generate responses.

6. **Add MCP Server Trigger Node**  
   - Add **LangChain MCP Trigger** node named `🗂️ MCP Instagram`.  
   - Configure webhook URL to your MCP Server SSE endpoint (e.g., `https://yourdomain.com/mcp/server/instagram/sse`).  
   - This node listens for MCP Server events.

7. **Add MCP Client Tool Node**  
   - Add **LangChain MCP Client Tool** node named `MCP Instagram`.  
   - Configure with the MCP Server URL matching the trigger node.  
   - Connect `AI Agent` → `MCP Instagram` (ai_tool input).  
   - Connect `MCP Instagram` → `AI Agent` (ai_tool output).

8. **Add HTTP Request Nodes for Instagram API Calls**  
   - Add **LangChain Tool HTTP Request** nodes for each Instagram API interaction:  
     - `Search Media`: GET `/media` endpoint using Instagram ID and Access Token.  
     - `Media Details`: GET `/media/{media-id}` for captions and media URLs.  
     - `Search Comment`: GET `/comments` for comments and replies.  
     - `Reply Comment`: POST to `/comments` to reply to comments.  
     - `Send Direct Message`: POST to `/messages` to send DMs.  
   - Connect each HTTP Request node’s output to the next logical node as per the data flow:  
     - `Search Media` → `Media Details` → `Search Comment` → `Reply Comment` → `MCP Instagram`.  
   - Connect `MCP Instagram` → `Search Media` (ai_tool input) to allow AI Agent to trigger these requests.

9. **Connect Memory Node**  
   - Connect `AI Agent` → `Simple Memory` (to store conversation context).  
   - Connect `Simple Memory` → `AI Agent` (to provide memory context).

10. **Set Credentials**  
    - Configure OpenAI API credentials in `Chat Model`.  
    - Configure MCP Server credentials or authentication if required in MCP nodes.  
    - Ensure Instagram Access Token and Instagram Business ID are set in `Instagram Mapping` node or securely stored credentials.

11. **Test the Workflow**  
    - Trigger the workflow by sending a chat message to the webhook URL of `When chat message received`.  
    - Verify that the AI Agent processes the input, calls Instagram API endpoints via MCP Server, and returns responses.  
    - Check that comments are fetched, replies are posted, and direct messages are sent as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow designed by Amanda, expert in intelligent automations with n8n and Make.               | Branding and author info                             |
| Buy workflows and support: [https://iloveflows.com](https://iloveflows.com)                     | Purchase link                                        |
| Try n8n Cloud platform: [https://n8n.partnerlinks.io/amanda](https://n8n.partnerlinks.io/amanda) | Cloud hosting and trial link                         |
| Requires Meta Graph API access and Instagram Business account for full functionality.            | Setup requirement                                    |
| MCP Server SSE endpoint must be configured and accessible for real-time communication.           | Technical prerequisite                               |
| OpenAI API key needed for GPT-4o model usage.                                                    | API credential requirement                           |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the Instagram MCP AI Agent workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with this automation.