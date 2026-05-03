Build your own Github MCP server

https://n8nworkflows.xyz/workflows/build-your-own-github-mcp-server-3635


# Build your own Github MCP server

### 1. Workflow Overview

This workflow implements a custom GitHub Model Context Protocol (MCP) server using n8n. Its primary purpose is to enable MCP-compatible clients to interact with GitHub repositories by viewing and commenting on issues. Unlike the official GitHub MCP server, this implementation offers tailored access control and simplified functionality focused on issue management, making it suitable for organizational customization and security.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger Setup:** Listens for incoming MCP client requests.
- **1.2 Operation Routing:** Routes requests based on the requested operation (e.g., get latest issues, get issue comments, add issue comment).
- **1.3 Get Latest Issues Tool:** Retrieves and simplifies the latest issues from a specified GitHub repository.
- **1.4 Get Issue Comments Tool:** Retrieves a specific issue and its comments, then simplifies and aggregates the data.
- **1.5 Add Issue Comment Tool:** Adds a comment to a specific GitHub issue and confirms the action.

Each tool is implemented as a custom workflow tool node that internally uses standard GitHub API nodes preconfigured with credentials and repository information.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger Setup

- **Overview:**  
  This block initializes the MCP server trigger node that listens for incoming MCP client requests. It acts as the entry point for all MCP interactions.

- **Nodes Involved:**  
  - Github MCP Server  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Github MCP Server**  
    - *Type:* MCP Trigger Node  
    - *Role:* Listens on a specific webhook path for MCP client requests.  
    - *Configuration:* Webhook path set to a unique identifier.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Emits data to connected nodes upon receiving MCP requests.  
    - *Edge Cases:* Requires authentication before production use to prevent unauthorized access.  
    - *Sticky Note:* Provides a link to official MCP Server Trigger documentation.

---

#### 2.2 Operation Routing

- **Overview:**  
  Routes incoming MCP requests to the appropriate tool workflow based on the "operation" parameter in the request JSON.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Operation (Switch node)

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Allows this workflow to be triggered by other workflows, passing parameters such as operation, repo, issueNumber, and text.  
    - *Inputs:* Triggered externally with workflow inputs.  
    - *Outputs:* Passes data to the Operation switch node.  
    - *Edge Cases:* Input validation needed to ensure required parameters are present.  
  - **Operation**  
    - *Type:* Switch Node  
    - *Role:* Routes data based on the "operation" field in the input JSON.  
    - *Configuration:* Three outputs named "getLatestIssues", "getIssueComments", and "addIssueComment" mapped to corresponding string values in the "operation" field.  
    - *Inputs:* Receives JSON with operation details.  
    - *Outputs:* Routes to one of three custom tool workflows.  
    - *Edge Cases:* If operation is missing or unrecognized, no output is triggered; consider adding a default or error handling.

---

#### 2.3 Get Latest Issues Tool

- **Overview:**  
  Retrieves the latest issues from a specified GitHub repository, simplifies the issue data, and aggregates it for response.

- **Nodes Involved:**  
  - Get Latest Issues (Tool Workflow)  
  - Get Many Issues (GitHub node)  
  - Simplify Issues (Set node)  
  - Aggregate Results (Aggregate node)

- **Node Details:**  
  - **Get Latest Issues**  
    - *Type:* Tool Workflow Node  
    - *Role:* Encapsulates the logic to fetch latest issues.  
    - *Configuration:* Passes parameters like repo and operation to the internal workflow.  
    - *Inputs:* Receives operation and repo parameters.  
    - *Outputs:* Returns aggregated simplified issue data.  
  - **Get Many Issues**  
    - *Type:* GitHub Node  
    - *Role:* Calls GitHub API to get up to 10 latest issues sorted by creation date.  
    - *Configuration:* Owner and repository extracted from repo string (format "owner/repo").  
    - *Credentials:* Uses preconfigured GitHub API credentials.  
    - *Edge Cases:* API rate limits, repository access permissions, empty issue list.  
  - **Simplify Issues**  
    - *Type:* Set Node  
    - *Role:* Extracts and renames relevant fields from raw GitHub issue data (e.g., issue_number, title, url, reporter, state, timestamps, body).  
    - *Edge Cases:* Missing fields in GitHub response.  
  - **Aggregate Results**  
    - *Type:* Aggregate Node  
    - *Role:* Aggregates all simplified issue items into a single response field.  
    - *Edge Cases:* Empty input data.

---

#### 2.4 Get Issue Comments Tool

- **Overview:**  
  Retrieves a specific GitHub issue and its associated comments, simplifies the data, and aggregates it for response.

- **Nodes Involved:**  
  - Get Issue Comments (Tool Workflow)  
  - Get Single Issue (GitHub node)  
  - Get Comments (HTTP Request node)  
  - Simplify Comments (Set node)  
  - Aggregate Comments (Aggregate node)

- **Node Details:**  
  - **Get Issue Comments**  
    - *Type:* Tool Workflow Node  
    - *Role:* Encapsulates logic to fetch issue details and comments.  
    - *Inputs:* Requires repo and issueNumber parameters.  
  - **Get Single Issue**  
    - *Type:* GitHub Node  
    - *Role:* Retrieves detailed information about a single issue by issue number.  
    - *Configuration:* Owner and repo parsed from repo string; issueNumber passed dynamically.  
    - *Credentials:* Uses GitHub API credentials.  
    - *Edge Cases:* Issue not found, permission errors.  
  - **Get Comments**  
    - *Type:* HTTP Request Node  
    - *Role:* Fetches comments from the issue's comments_url field.  
    - *Configuration:* URL dynamically set from the issue data; uses GitHub API credentials for authentication.  
    - *Edge Cases:* API rate limits, empty comments.  
  - **Simplify Comments**  
    - *Type:* Set Node  
    - *Role:* Extracts and renames relevant fields from raw comment data (id, issue_url, user, author_association, body, timestamps).  
    - *Edge Cases:* Missing fields or empty comments.  
  - **Aggregate Comments**  
    - *Type:* Aggregate Node  
    - *Role:* Aggregates all simplified comment items into a single response field.  
    - *Edge Cases:* Empty input data.

---

#### 2.5 Add Issue Comment Tool

- **Overview:**  
  Adds a comment to a specified GitHub issue and returns a confirmation response.

- **Nodes Involved:**  
  - Add Issue Comment (Tool Workflow)  
  - Create Comment (GitHub node)  
  - Get Response (Set node)

- **Node Details:**  
  - **Add Issue Comment**  
    - *Type:* Tool Workflow Node  
    - *Role:* Encapsulates logic to add a comment to an issue.  
    - *Inputs:* Requires repo, issueNumber, and text parameters.  
  - **Create Comment**  
    - *Type:* GitHub Node  
    - *Role:* Calls GitHub API to create a comment on the specified issue.  
    - *Configuration:* Owner and repo parsed from repo string; issueNumber and comment body passed dynamically.  
    - *Credentials:* Uses GitHub API credentials.  
    - *Edge Cases:* Permission denied, invalid issue number, empty comment text.  
  - **Get Response**  
    - *Type:* Set Node  
    - *Role:* Sets a simple confirmation response ("ok") after comment creation.  
    - *Edge Cases:* None significant.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                                  | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                                                                    |
|----------------------------|----------------------------------|-------------------------------------------------|-------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Github MCP Server           | MCP Trigger                      | Entry point for MCP client requests             | None                          | When Executed by Another Workflow | ## 1. Set up an MCP Server Trigger [Read more about the MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| When Executed by Another Workflow | Execute Workflow Trigger         | Receives external workflow triggers             | Github MCP Server             | Operation                    |                                                                                                                                                |
| Operation                  | Switch                          | Routes requests based on operation parameter    | When Executed by Another Workflow | Get Many Issues, Get Single Issue, Create Comment |                                                                                                                                               |
| Get Latest Issues          | Tool Workflow                   | Retrieves latest issues                          | Operation (getLatestIssues)   | Github MCP Server            | ## 2. Build Simple Support Tools with Github Node [Read more about the Github Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.github) |
| Get Many Issues            | GitHub                          | Calls GitHub API to get latest issues            | Operation                    | Simplify Issues              |                                                                                                                                               |
| Simplify Issues            | Set                             | Simplifies issue data                            | Get Many Issues              | Aggregate Results            |                                                                                                                                               |
| Aggregate Results          | Aggregate                      | Aggregates simplified issues                     | Simplify Issues              |                              |                                                                                                                                               |
| Get Issue Comments         | Tool Workflow                   | Retrieves issue and comments                     | Operation (getIssueComments) | Github MCP Server            |                                                                                                                                               |
| Get Single Issue           | GitHub                          | Gets single issue details                        | Operation                    | Get Comments                |                                                                                                                                               |
| Get Comments               | HTTP Request                   | Fetches issue comments                           | Get Single Issue             | Simplify Comments            |                                                                                                                                               |
| Simplify Comments          | Set                             | Simplifies comment data                          | Get Comments                 | Aggregate Comments           |                                                                                                                                               |
| Aggregate Comments         | Aggregate                      | Aggregates simplified comments                   | Simplify Comments            |                              |                                                                                                                                               |
| Add Issue Comment          | Tool Workflow                   | Adds comment to issue                            | Operation (addIssueComment)  | Github MCP Server            |                                                                                                                                               |
| Create Comment             | GitHub                          | Calls GitHub API to create comment               | Operation                    | Get Response                |                                                                                                                                               |
| Get Response               | Set                             | Sets confirmation response                       | Create Comment               |                              |                                                                                                                                               |
| Sticky Note                | Sticky Note                    | Instructional note about MCP Server Trigger     | None                         | None                        | ## 1. Set up an MCP Server Trigger [Read more about the MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| Sticky Note1               | Sticky Note                    | Instructional note about Github Node usage      | None                         | None                        | ## 2. Build Simple Support Tools with Github Node [Read more about the Github Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.github) |
| Sticky Note3               | Sticky Note                    | Security reminder about authenticating MCP server | None                         | None                        | ### Always Authenticate Your Server! Before going to production, it's always advised to enable authentication on your MCP server trigger.       |
| Sticky Note2               | Sticky Note                    | Full workflow description and usage instructions | None                         | None                        | See full workflow description and usage instructions including links to official MCP server reference and client integration guides.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Add an **MCP Trigger** node named "Github MCP Server".  
   - Set the webhook path to a unique identifier (e.g., "61848df7-3619-4ccf-831b-d6408e0d6519").  
   - This node will listen for MCP client requests.

2. **Add Execute Workflow Trigger Node**  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
   - Configure workflow inputs: operation (string), repo (string), issueNumber (string), text (string).  
   - Connect "Github MCP Server" output to this node.

3. **Add Switch Node for Operation Routing**  
   - Add a **Switch** node named "Operation".  
   - Add three outputs with conditions on `{{$json.operation}}`:  
     - "getLatestIssues" equals "getLatestIssues"  
     - "getIssueComments" equals "getIssueComments"  
     - "addIssueComment" equals "addIssueComment"  
   - Connect "When Executed by Another Workflow" output to "Operation".

4. **Create Tool Workflow: Get Latest Issues**  
   - Add a **Tool Workflow** node named "Get Latest Issues".  
   - Set the workflow ID to the current workflow's ID (self-reference).  
   - Define workflow inputs: operation="getLatestIssues", repo (string), issueNumber=null, text=null.  
   - Connect "Operation" output "getLatestIssues" to this node.

5. **Inside Get Latest Issues Tool Workflow:**  
   - Add a **GitHub** node named "Get Many Issues".  
     - Operation: List issues for repository.  
     - Owner: Extract from repo parameter (split by "/").  
     - Repository: Extract from repo parameter.  
     - Limit: 10 issues sorted by creation date.  
     - Use GitHub API credentials.  
   - Add a **Set** node named "Simplify Issues".  
     - Map fields: number → issue_number, title, url, user.login → reported_by, state, created_at, updated_at, body.  
   - Add an **Aggregate** node named "Aggregate Results".  
     - Aggregate all items into a single field "response".  
   - Connect nodes in order: Get Many Issues → Simplify Issues → Aggregate Results.  
   - Return aggregated data as output of the tool workflow.

6. **Create Tool Workflow: Get Issue Comments**  
   - Add a **Tool Workflow** node named "Get Issue Comments".  
   - Set workflow inputs: operation="getIssueComments", repo (string), issueNumber (string), text=null.  
   - Connect "Operation" output "getIssueComments" to this node.

7. **Inside Get Issue Comments Tool Workflow:**  
   - Add a **GitHub** node named "Get Single Issue".  
     - Operation: Get issue by number.  
     - Owner and repository parsed from repo input.  
     - Issue number from input.  
     - Use GitHub API credentials.  
   - Add an **HTTP Request** node named "Get Comments".  
     - URL: Use `{{$json.comments_url}}` from "Get Single Issue" output.  
     - Authentication: Use GitHub API credentials.  
   - Add a **Set** node named "Simplify Comments".  
     - Map fields: id, issue_url, user.login → user, author_association, body, created_at, updated_at.  
   - Add an **Aggregate** node named "Aggregate Comments".  
     - Aggregate all items into a single field "response".  
   - Connect nodes: Get Single Issue → Get Comments → Simplify Comments → Aggregate Comments.  
   - Return aggregated data as output of the tool workflow.

8. **Create Tool Workflow: Add Issue Comment**  
   - Add a **Tool Workflow** node named "Add Issue Comment".  
   - Set workflow inputs: operation="addIssueComment", repo (string), issueNumber (string), text (string).  
   - Connect "Operation" output "addIssueComment" to this node.

9. **Inside Add Issue Comment Tool Workflow:**  
   - Add a **GitHub** node named "Create Comment".  
     - Operation: Create comment on issue.  
     - Owner and repository parsed from repo input.  
     - Issue number and comment body from inputs.  
     - Use GitHub API credentials.  
   - Add a **Set** node named "Get Response".  
     - Set field "response" to string "ok".  
   - Connect nodes: Create Comment → Get Response.  
   - Return confirmation response as output of the tool workflow.

10. **Credential Setup:**  
    - Configure GitHub API credentials with appropriate OAuth2 or personal access token having repository read/write permissions.  
    - Assign credentials to all GitHub and HTTP Request nodes requiring authentication.

11. **Security Recommendations:**  
    - Enable authentication on the MCP Server Trigger node before production use to restrict access.  
    - Validate inputs in the "When Executed by Another Workflow" node or add error handling for missing/invalid parameters.

12. **Testing:**  
    - Use an MCP-compatible client such as Claude Desktop.  
    - Connect client to the MCP server webhook URL.  
    - Test queries like:  
      - "Can you get me the latest issues about MCP?"  
      - "What is the current progress on Issue 12345?"  
      - "Please can you add a comment to Issue 12345 that they should try installing the latest version and see if that works?"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow is based on the official MCP reference implementation for GitHub servers.                                                                                                                                       | https://github.com/modelcontextprotocol/servers/tree/main/src/github                                                     |
| MCP Server Trigger documentation and integration guide for Claude Desktop client.                                                                                                                                               | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop       |
| GitHub Node documentation for API operations used in this workflow.                                                                                                                                                             | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.github                                                  |
| Security reminder: Always enable authentication on your MCP server trigger before going to production to prevent unauthorized access.                                                                                          | Sticky Note in workflow                                                                                                   |
| MCP clients require GitHub account access and appropriate repository permissions to interact with this MCP server.                                                                                                            | Workflow description                                                                                                      |
| MCP clients supported include Claude Desktop (https://claude.ai/download).                                                                                                                                                      | Workflow description                                                                                                      |
| Extending this workflow to support pull requests, workflows, or metrics reporting is recommended for organizational use cases.                                                                                                | Workflow description                                                                                                      |

---

This document provides a comprehensive understanding of the "Build your own Github MCP server" workflow, enabling advanced users and AI agents to analyze, reproduce, and customize the workflow effectively.