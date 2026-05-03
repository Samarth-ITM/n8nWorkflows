🛠️ Jina AI Tool MCP Server 💪 all 3 operations

https://n8nworkflows.xyz/workflows/----jina-ai-tool-mcp-server----all-3-operations-5238


# 🛠️ Jina AI Tool MCP Server 💪 all 3 operations

### 1. Workflow Overview

This workflow, titled **"🛠️ Jina AI Tool MCP Server"**, serves as a multi-operation MCP (Multi-Channel Platform) server endpoint to perform AI-powered content reading, web searching, and deep research using the Jina AI Tool. The workflow exposes a webhook endpoint that AI agents or external systems can call to trigger one of three operations, each corresponding to a different type of information retrieval or processing task.

**Target Use Cases:**  
- Reading and simplifying content from a provided URL.  
- Searching the web based on a query string.  
- Conducting deep research using AI-enhanced queries.

**Logical Blocks:**  
- **1.1 MCP Trigger Input Reception:** Accepts incoming requests at a webhook URL and routes them to appropriate operations.  
- **1.2 Reader Operations:** Handles reading content from URLs and simplified response generation.  
- **1.3 Search Operation:** Executes web search queries and returns results.  
- **1.4 Research Operation:** Performs deep research queries with AI assistance.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger Input Reception

- **Overview:**  
  This block listens for incoming HTTP requests on a webhook URL and triggers the workflow. It acts as the main entry point, enabling the workflow to function as an MCP server.

- **Nodes Involved:**  
  - `Jina AI Tool MCP Server`

- **Node Details:**  
  - **Node Name:** Jina AI Tool MCP Server  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP Trigger node)  
  - **Configuration:**  
    - Webhook path set to `jina-ai-tool-mcp`  
    - This node listens for requests from clients or AI agents.  
  - **Expressions / Variables:** None (direct webhook usage)  
  - **Input:** None (start node)  
  - **Output:** Feeds data to all three AI tool operation nodes (`Read URL content`, `Search web`, `Perform deep research`) via the connection labeled `ai_tool`  
  - **Version-specific Requirements:** Requires n8n version that supports the MCP Trigger node (`@n8n/n8n-nodes-langchain` integration)  
  - **Potential Failures:**  
    - Webhook connection errors (port conflicts, invalid host configuration)  
    - Unauthorized or malformed requests may cause failures if not handled externally  
  - **Sub-workflow:** None

---

#### 1.2 Reader Operations

- **Overview:**  
  Provides two reader-related operations: reading content from a URL and optionally simplifying it, and searching the web. These operations rely on AI-generated inputs to populate parameters.

- **Nodes Involved:**  
  - `Read URL content`  
  - `Search web`  
  - `Sticky Note 1` (labeling block as "Reader")

- **Node Details:**  

  - **Node Name:** Read URL content  
    - Type: `n8n-nodes-base.jinaAiTool`  
    - Role: Reads content from a provided URL and optionally simplifies the output  
    - Configuration:  
      - `url` parameter dynamically populated by AI expression: `$fromAI('Url', '', 'string')`  
      - `simplify` parameter dynamically populated by AI expression: `$fromAI('Simplify', '', 'boolean')`  
      - No additional options or request options configured  
      - Credential: Jina AI API credentials must be set (credential ID placeholder present)  
    - Input: From `Jina AI Tool MCP Server` node via `ai_tool` connection  
    - Output: Response data sent downstream (not connected to any further node inside this workflow)  
    - Edge Cases / Failures:  
      - Invalid URL formats or inaccessible URLs  
      - API authentication failures if credentials are missing or invalid  
      - Timeout or rate limiting issues from Jina AI service

  - **Node Name:** Search web  
    - Type: `n8n-nodes-base.jinaAiTool`  
    - Role: Performs a web search based on a user query with optional simplification of results  
    - Configuration:  
      - `operation` explicitly set to `"search"`  
      - `searchQuery` dynamically populated by AI expression: `$fromAI('Search_Query', '', 'string')`  
      - `simplify` parameter dynamically populated by AI expression: `$fromAI('Simplify', '', 'boolean')`  
      - Credential: Same Jina AI API credentials as above  
    - Input: From `Jina AI Tool MCP Server` node via `ai_tool` connection  
    - Output: Search results returned (no further internal connections)  
    - Edge Cases / Failures:  
      - Empty or malformed search queries  
      - API limits or auth errors  
      - No results found or low relevance

  - **Node Name:** Sticky Note 1  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Visual label indicating this block is the "Reader" operations group  
    - Configuration: Simple text content "## Reader"

---

#### 1.3 Research Operation

- **Overview:**  
  Performs a deep research operation as a separate AI-powered resource action, accepting a research query and returning enriched research results.

- **Nodes Involved:**  
  - `Perform deep research`  
  - `Sticky Note 2` (labeling block as "Research")

- **Node Details:**  

  - **Node Name:** Perform deep research  
    - Type: `n8n-nodes-base.jinaAiTool`  
    - Role: Executes a deep research query using the `research` resource and returns AI-enhanced results  
    - Configuration:  
      - `resource` explicitly set to `"research"`  
      - `researchQuery` dynamically populated by AI expression: `$fromAI('Research_Query', '', 'string')`  
      - `simplify` parameter dynamically populated by AI expression: `$fromAI('Simplify', '', 'boolean')`  
      - Credential: Jina AI API credentials (same as others)  
    - Input: From `Jina AI Tool MCP Server` node via `ai_tool` connection  
    - Output: Research results returned to requester  
    - Edge Cases / Failures:  
      - Empty or invalid research queries  
      - API authentication or quota issues  
      - Response delays for complex research tasks

  - **Node Name:** Sticky Note 2  
    - Type: `n8n-nodes-base.stickyNote`  
    - Role: Visual label indicating this block is the "Research" operation group

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role             | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                                                                                                                             |
|-------------------------|--------------------------------------|-----------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0     | n8n-nodes-base.stickyNote             | Documentation / overview    | None                   | None                    | ## 🛠️ Jina AI Tool MCP Server<br>### 📋 Available Operations (3 total)<br>**Reader**: read, search<br>**Research**: deep research<br>### ⚙️ Setup Instructions<br>1. Import Workflow<br>2. Add Credentials<br>3. Activate<br>4. Get URL<br>5. Connect<br>### ✨ Features<br>• Zero config<br>• AI agents auto-populate parameters<br>• Native error handling<br>• Modify defaults as needed<br>### 💬 Need Help?<br>See https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/<br>Discord: https://discord.me/cfomodz |
| Jina AI Tool MCP Server | @n8n/n8n-nodes-langchain.mcpTrigger  | MCP webhook trigger         | None                   | Read URL content, Search web, Perform deep research |                                                                                                                                                                                                                                        |
| Read URL content        | n8n-nodes-base.jinaAiTool             | Read content from URL       | Jina AI Tool MCP Server | None                    |                                                                                                                                                                                                                                        |
| Search web             | n8n-nodes-base.jinaAiTool             | Perform web search          | Jina AI Tool MCP Server | None                    |                                                                                                                                                                                                                                        |
| Sticky Note 1           | n8n-nodes-base.stickyNote             | Label for Reader operations | None                   | None                    | ## Reader                                                                                                                                                                                                                              |
| Perform deep research   | n8n-nodes-base.jinaAiTool             | Deep research operation     | Jina AI Tool MCP Server | None                    |                                                                                                                                                                                                                                        |
| Sticky Note 2           | n8n-nodes-base.stickyNote             | Label for Research operation | None                   | None                    | ## Research                                                                                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance and name it `Jina AI Tool MCP Server`.

2. **Add MCP Trigger node:**  
   - Add node: Type `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure webhook path as `jina-ai-tool-mcp`  
   - No credentials needed here  
   - This node will be your workflow's start node.

3. **Add 'Read URL content' node:**  
   - Add node: Type `jinaAiTool` (Jina AI Tool)  
   - Set parameters:  
     - `url`: Expression `{{$fromAI('Url', '', 'string')}}`  
     - `simplify`: Expression `{{$fromAI('Simplify', '', 'boolean')}}`  
     - Leave `options` and `requestOptions` empty  
   - Set credentials: Select or create Jina AI API credentials  
   - Connect input from `Jina AI Tool MCP Server` node with connection type `ai_tool`.

4. **Add 'Search web' node:**  
   - Add node: Type `jinaAiTool`  
   - Set parameters:  
     - `operation`: `"search"` (select or type explicitly)  
     - `searchQuery`: Expression `{{$fromAI('Search_Query', '', 'string')}}`  
     - `simplify`: Expression `{{$fromAI('Simplify', '', 'boolean')}}`  
     - Leave `options` and `requestOptions` empty  
   - Use the same Jina AI API credentials as above  
   - Connect input from `Jina AI Tool MCP Server` node with connection type `ai_tool`.

5. **Add 'Perform deep research' node:**  
   - Add node: Type `jinaAiTool`  
   - Set parameters:  
     - `resource`: `"research"` (select or type explicitly)  
     - `researchQuery`: Expression `{{$fromAI('Research_Query', '', 'string')}}`  
     - `simplify`: Expression `{{$fromAI('Simplify', '', 'boolean')}}`  
     - Leave `options` and `requestOptions` empty  
   - Use the same Jina AI API credentials as above  
   - Connect input from `Jina AI Tool MCP Server` node with connection type `ai_tool`.

6. **Add sticky notes for clarity:**  
   - Add a sticky note near `Read URL content` and `Search web` nodes with content: `## Reader`  
   - Add a sticky note near `Perform deep research` node with content: `## Research`  
   - Optionally add a large sticky note at the left/top with the workflow overview content from step 1.

7. **Configure Credentials:**  
   - Create or select existing Jina AI API credentials in n8n (API key or OAuth2 depending on Jina setup)  
   - Assign these credentials to all Jina AI Tool nodes.

8. **Activate the workflow:**  
   - Save and activate the workflow.  
   - Obtain the webhook URL from the MCP trigger node’s webhook tab.  
   - Integrate this URL with your AI agents or external callers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                               | Context or Link                                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow supports zero-configuration use with AI agents that automatically provide input parameters via `$fromAI()` expressions.                                                    | Workflow design detail                                                                                               |
| For further help with MCP integration or customizations, join n8n Discord at https://discord.me/cfomodz.                                                                                   | Support and community                                                                                               |
| Official documentation for the MCP node and Langchain tool integration: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                      | Official n8n docs                                                                                                   |
| Ensure Jina AI API credentials are correctly set once in one node; then open and close other tool nodes to refresh credential usage.                                                      | Setup instruction                                                                                                |
| The workflow is designed to handle all three operations natively with error handling and response formatting managed by MCP trigger node.                                               | Feature description                                                                                                |

---

**Disclaimer:**  
The text provided is extracted solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.