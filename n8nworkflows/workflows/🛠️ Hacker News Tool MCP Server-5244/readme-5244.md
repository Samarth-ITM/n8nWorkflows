🛠️ Hacker News Tool MCP Server

https://n8nworkflows.xyz/workflows/----hacker-news-tool-mcp-server-5244


# 🛠️ Hacker News Tool MCP Server

### 1. Workflow Overview

The **🛠️ Hacker News Tool MCP Server** workflow is designed to serve as a ready-to-use API server for interacting with the Hacker News data via n8n’s native Hacker News Tool nodes. Its target use cases include AI agents or external systems that require fetching Hacker News information such as lists of items, individual articles, or user profiles without requiring authentication. The workflow exposes three operations accessible through a single MCP (Multi-Channel Platform) trigger endpoint.

The workflow’s logic is structured into these logical blocks:

- **1.1 Input Reception:** Accepts incoming HTTP requests via the MCP trigger node.
- **1.2 Data Retrieval Operations:** Executes one of three Hacker News Tool operations based on parameters:
  - "All" — fetch multiple Hacker News items.
  - "Article" — fetch a specific article by ID.
  - "User" — fetch user profile data by username.
- **1.3 Response Handling and Output:** (Implicitly handled by MCP node) returns data back to the requester with built-in error handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block is the entry point of the workflow. It listens for incoming HTTP requests on a webhook endpoint dedicated to the Hacker News Tool MCP Server. It triggers the workflow whenever a client (e.g., an AI agent) calls the HTTP endpoint with the required parameters.

- **Nodes Involved:**  
  - `Hacker News Tool MCP Server` (MCP Trigger Node)

- **Node Details:**  
  - **Node Name:** Hacker News Tool MCP Server  
  - **Type:** `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - **Technical Role:** HTTP webhook trigger specialized for MCP (Multi-Channel Platform) server operations.  
  - **Configuration:**  
    - Path set to `"hacker-news-tool-mcp"`, exposing the webhook at `/webhook/hacker-news-tool-mcp`.  
    - No authentication required (open endpoint).  
  - **Key Expressions:** None directly on this node; parameters are populated downstream using `$fromAI()` expressions.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connects to the three separate operation nodes (`Get many items`, `Get an article`, `Get a user`) via the MCP “ai_tool” output connection.  
  - **Version Requirements:** Requires n8n version supporting MCP trigger nodes (version >= 1.95.3 recommended).  
  - **Potential Failures:**  
    - Webhook endpoint unavailable if workflow is inactive.  
    - Improper or missing parameters passed by clients may result in operation failures downstream.  
  - **Sub-workflow:** Not applicable.

---

#### 2.2 Data Retrieval Operations

This block contains three parallel branches, each corresponding to one Hacker News data operation. The MCP trigger node routes requests to these operations depending on the parameters passed by AI agents.

---

##### 2.2.1 "All" Operation — Fetch Multiple Hacker News Items

- **Overview:**  
  Retrieves multiple Hacker News items with optional parameter controls for limit and whether to return all items.

- **Nodes Involved:**  
  - `Get many items` (Hacker News Tool node)  
  - `Sticky Note 1` (annotation node)

- **Node Details:**  

  - **Node Name:** Get many items  
  - **Type:** `n8n-nodes-base.hackerNewsTool`  
  - **Technical Role:** Fetches multiple Hacker News items from the “all” resource.  
  - **Configuration:**  
    - `resource` set to `"all"` to fetch all item types.  
    - `limit` dynamically set via expression: `{{$fromAI('Limit', ``, 'number')}}` — fetches a numeric limit parameter provided by AI agent context.  
    - `returnAll` dynamically set via expression: `{{$fromAI('Return_All', ``, 'boolean')}}` — allows the client to request either all items or a limited subset.  
    - `additionalFields`: Empty, no extra options configured.  
  - **Input Connections:** Receives requests from the MCP trigger node.  
  - **Output Connections:** Returns data back to the MCP system to be forwarded to the requester.  
  - **Key Expressions:** Use of `$fromAI()` expressions enables dynamic parameter population based on AI agent input.  
  - **Version Requirements:** Requires n8n version with Hacker News Tool node support.  
  - **Potential Failures:**  
    - Invalid `limit` parameter (non-numeric or out of range).  
    - Network issues or Hacker News API downtime.  
    - Misuse of `returnAll` flag causing large data sets and possible timeouts.  
  - **Sticky Note:** Labelled “## All” explaining this node’s role.

---

##### 2.2.2 "Article" Operation — Fetch a Single Article by ID

- **Overview:**  
  Fetches details of a specific Hacker News article identified by an article ID.

- **Nodes Involved:**  
  - `Get an article` (Hacker News Tool node)  
  - `Sticky Note 2` (annotation node)

- **Node Details:**  

  - **Node Name:** Get an article  
  - **Type:** `n8n-nodes-base.hackerNewsTool`  
  - **Technical Role:** Retrieves a single article detail by ID.  
  - **Configuration:**  
    - `articleId` set dynamically via expression: `{{$fromAI('Article_Id', ``, 'string')}}` — expects the AI agent or client to provide the article ID as a string.  
    - `additionalFields`: Empty, no extra options configured.  
  - **Input Connections:** Receives requests from the MCP trigger node.  
  - **Output Connections:** Returns the article data to MCP response.  
  - **Key Expressions:** Dynamic article ID parameter via `$fromAI()`.  
  - **Version Requirements:** Requires Hacker News Tool node support in n8n.  
  - **Potential Failures:**  
    - Missing or invalid `articleId`.  
    - Article not found or deleted on Hacker News.  
    - API rate limits or connectivity issues.  
  - **Sticky Note:** Labelled “## Article” clarifying node purpose.

---

##### 2.2.3 "User" Operation — Fetch a Hacker News User Profile

- **Overview:**  
  Retrieves profile information for a specified Hacker News user by username.

- **Nodes Involved:**  
  - `Get a user` (Hacker News Tool node)  
  - `Sticky Note 3` (annotation node)

- **Node Details:**  

  - **Node Name:** Get a user  
  - **Type:** `n8n-nodes-base.hackerNewsTool`  
  - **Technical Role:** Fetches user profile data.  
  - **Configuration:**  
    - `resource` set to `"user"` to target user data.  
    - `username` dynamically set via expression: `{{$fromAI('Username', ``, 'string')}}` — AI agent provides username string.  
  - **Input Connections:** Connected from MCP trigger node.  
  - **Output Connections:** Returns user details to MCP response.  
  - **Key Expressions:** Uses `$fromAI()` to dynamically receive username.  
  - **Version Requirements:** Requires Hacker News Tool node availability.  
  - **Potential Failures:**  
    - Missing or invalid username parameter.  
    - Username does not exist on Hacker News.  
    - API or network errors.  
  - **Sticky Note:** Labelled “## User” for clarity.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role               | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                                              |
|---------------------------|--------------------------------------|------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0       | Sticky Note                          | Documentation and overview   | None                          | None                          | ## 🛠️ Hacker News Tool MCP Server ... [includes setup instructions, available operations, and help links]                              |
| Hacker News Tool MCP Server | MCP Trigger                         | Entry point, HTTP webhook    | None                          | Get many items, Get an article, Get a user |                                                                                                                                          |
| Get many items            | Hacker News Tool                     | Fetch multiple Hacker News items | Hacker News Tool MCP Server   | MCP response                  | ## All                                                                                                                                   |
| Sticky Note 1             | Sticky Note                          | Annotation for "All" operation | None                         | None                          | ## All                                                                                                                                   |
| Get an article            | Hacker News Tool                     | Fetch single article by ID   | Hacker News Tool MCP Server   | MCP response                  | ## Article                                                                                                                               |
| Sticky Note 2             | Sticky Note                          | Annotation for "Article" operation | None                         | None                          | ## Article                                                                                                                               |
| Get a user                | Hacker News Tool                     | Fetch user profile by username | Hacker News Tool MCP Server   | MCP response                  | ## User                                                                                                                                  |
| Sticky Note 3             | Sticky Note                          | Annotation for "User" operation | None                         | None                          | ## User                                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add node: **MCP Trigger** (`@n8n/n8n-nodes-langchain.mcpTrigger`).  
   - Set the webhook path to: `hacker-news-tool-mcp`.  
   - No authentication required.  
   - This node will serve as the HTTP entry point.

2. **Add the "Get many items" Node**  
   - Add node: **Hacker News Tool** node.  
   - Set `Resource` to `all`.  
   - Set `Limit` parameter to expression: `{{$fromAI('Limit', ``, 'number')}}`.  
   - Set `Return All` parameter to expression: `{{$fromAI('Return_All', ``, 'boolean')}}`.  
   - Leave additional fields empty.  
   - Connect input from MCP Trigger node.

3. **Add the "Get an article" Node**  
   - Add node: **Hacker News Tool** node.  
   - Set `Article Id` parameter to expression: `{{$fromAI('Article_Id', ``, 'string')}}`.  
   - Leave additional fields empty.  
   - Connect input from MCP Trigger node.

4. **Add the "Get a user" Node**  
   - Add node: **Hacker News Tool** node.  
   - Set `Resource` parameter to `user`.  
   - Set `Username` parameter to expression: `{{$fromAI('Username', ``, 'string')}}`.  
   - Connect input from MCP Trigger node.

5. **Add Sticky Notes (Optional for clarity)**  
   - Add sticky notes near each operation node with titles: “All”, “Article”, and “User” respectively.  
   - Add a large sticky note near the MCP trigger with setup instructions and overview content as in the original workflow.

6. **Activate the Workflow**  
   - Save and activate the workflow to expose the webhook endpoint.  
   - Use the webhook URL (`/webhook/hacker-news-tool-mcp`) in AI agents or clients.

7. **Credential Setup**  
   - No credentials are required as the Hacker News Tool nodes and MCP Trigger are configured for open access.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow requires no authentication, enabling immediate use and integration with AI agents.                                                                | Setup instructions in sticky note.                                                                              |
| AI agents automatically populate parameters via `$fromAI()` expressions, enabling dynamic queries without manual input.                                    | See parameter usage in Hacker News Tool nodes.                                                                  |
| For detailed MCP node usage and advanced configuration, refer to the official n8n documentation on MCP nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Official n8n docs.                                                                                               |
| For community support and custom integrations, join the Discord at https://discord.me/cfomodz                                                              | Discord community link.                                                                                          |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created within n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.