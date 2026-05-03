GitHub Automation Hub: Complete API Controls for AI Agents

https://n8nworkflows.xyz/workflows/github-automation-hub--complete-api-controls-for-ai-agents-4629


# GitHub Automation Hub: Complete API Controls for AI Agents

### 1. Workflow Overview

This workflow, titled **"GitHub Automation Hub: Complete API Controls for AI Agents"**, provides a comprehensive integration layer between AI agents and GitHub's API, enabling automated, precise control over a wide range of GitHub resources and operations. Its primary use case is to serve as a centralized API control hub where AI agents can programmatically manage repositories, files, issues, pull requests, releases, workflows, and organizational settings on GitHub via n8n.

The workflow is logically divided into the following blocks:

- **1.1 GitHub API Operations Block**: Implements nodes that handle core GitHub API operations such as file management (create, get, edit, delete), issue management (create, get, edit, comment, lock), release management (create, get, update, delete), pull request reviews, repository listing, and organization invitations.

- **1.2 AI Agent Trigger Block**: A single specialized trigger node that serves as the entry point, receiving AI agent commands and routing them to the relevant GitHub API operation nodes.

- **1.3 Workflow and Usage Management Block**: Nodes focused on managing n8n workflows within GitHub, including listing workflows, getting workflows by ID or name, enabling/disabling workflows, dispatching workflows, and retrieving usage statistics.

- **1.4 Disabled Custom HTTP Request Nodes**: A set of disabled HTTP Request nodes configured for custom GitHub API requests via POST, PATCH, GET, PUT, and DELETE methods, likely placeholders for advanced or custom API calls beyond standard nodes.

- **1.5 Sticky Notes for Documentation**: Multiple sticky notes placed throughout the canvas for organizational or instructional purposes, though content is empty in the provided JSON.

---

### 2. Block-by-Block Analysis

#### 2.1 GitHub API Operations Block

**Overview:**  
This block contains the main GitHub API integration nodes, each implementing a specific GitHub action, such as file management, issue tracking, releases, pull request reviews, repository queries, and user invitations. These nodes receive requests triggered by the AI agent and perform the corresponding GitHub API calls.

**Nodes Involved:**  
- Create File  
- Delete File  
- Edit File  
- Get File  
- List Files  
- Create Issue  
- Edit Issue  
- Comment on Existing Issue  
- Lock Issue by number  
- Get Issue  
- Get Issues  
- Create Release  
- Delete Release  
- Get Release  
- Get Many Releases  
- Update Releases  
- Get Pull Requests  
- Create PR Review  
- Update PR Review  
- Get a PR Review  
- Get All Reviews by PR Number  
- Get Repo  
- Get Organization's Repositories  
- Get User Repos by URL  
- Get User Repos by Name  
- Invite User to Organization  
- List Popular Paths  
- List Referrers  
- Get License  
- Get Profile  

**Node Details (common pattern):**  
- **Type & Role:** All are `n8n-nodes-base.githubTool` nodes, representing predefined integrations with GitHub's API for specific endpoints and operations.  
- **Configuration:** Parameters are empty in JSON, indicating runtime dynamic input likely provided by the AI agent trigger or via expressions. Each node is configured for a particular GitHub API action (e.g., "Create File" for creating repository files).  
- **Input Connections:** Each node receives input from the AI agent trigger node "Github MCP Server" via the custom `ai_tool` connection type, which implies input data is structured commands or parameters from the AI agent.  
- **Output Connections:** No downstream nodes, indicating these are terminal operation endpoints responding back to the trigger or external caller.  
- **Version:** All nodes use typeVersion 1.1, compatible with current GitHub integration standards in n8n.  
- **Edge Cases & Failures:**  
  - GitHub API rate limits and authentication failures (invalid or expired tokens).  
  - Resource not found errors (e.g., file, issue, or repo does not exist).  
  - Permission errors due to insufficient OAuth scopes or user rights.  
  - Network timeouts or API outages.  
  - Invalid input data causing API rejection.  
- **Sub-Workflow:** None, directly connected to the AI trigger.

---

#### 2.2 AI Agent Trigger Block

**Overview:**  
This block contains the entry point node that listens for AI agent commands and dispatches these commands to the appropriate GitHub API operation nodes.

**Nodes Involved:**  
- Github MCP Server

**Node Details:**  
- **Type & Role:** `@n8n/n8n-nodes-langchain.mcpTrigger` node, a specialized trigger designed to interface with AI agents (likely LangChain MCP - Multi-Channel Plugin).  
- **Configuration:** No explicit parameters set in JSON, implying default listening on configured webhook or API endpoint.  
- **Input Connections:** None, it is a trigger node.  
- **Output Connections:** Has multiple outputs connected to all GitHub operation nodes via the `ai_tool` connection, routing input commands to specific nodes.  
- **Version:** typeVersion 1.  
- **Edge Cases & Failures:**  
  - Invalid or malformed AI commands.  
  - Authorization failure for incoming requests.  
  - Overload or concurrency limits for incoming agent requests.  
- **Sub-Workflow:** None

---

#### 2.3 Workflow and Usage Management Block

**Overview:**  
Manages GitHub workflows within the repository or organization, including retrieving workflow definitions, enabling/disabling workflows, dispatching workflows, and usage retrieval, enabling AI agents to control CI/CD pipelines programmatically.

**Nodes Involved:**  
- List workflows  
- Get Workflow by ID  
- Get Workflow by Name  
- Enable Workflow by ID  
- Enable Workflow by Name  
- Disable Workflow by ID  
- Disable Workflow by Name  
- Dispatch Worthflow by ID  
- Dispatch Worthflow by Name  
- Get Usage by ID  
- Get Usage by Name  

**Node Details:**  
- **Type & Role:** `n8n-nodes-base.githubTool` nodes specialized for GitHub Actions workflow management API endpoints.  
- **Configuration:** Parameters unset, implying dynamic input from the AI trigger node.  
- **Input Connections:** All connected from "Github MCP Server" for direct AI control.  
- **Output Connections:** None downstream; terminal interaction nodes.  
- **Version:** typeVersion 1.1.  
- **Edge Cases & Failures:**  
  - Workflow or usage ID/name not found errors.  
  - Permission issues managing workflows.  
  - API rate limiting or transient failures during workflow dispatch.  
  - Misspelled "Dispatch Worthflow" node names (likely a typo for "Dispatch Workflow") — potential source of confusion or errors.  
- **Sub-Workflow:** None

---

#### 2.4 Disabled Custom HTTP Request Nodes

**Overview:**  
These nodes are configured for custom HTTP requests to GitHub API endpoints using various methods (POST, PATCH, GET, PUT, DELETE) but are disabled by default. They seem to be placeholders for advanced or custom API operations not covered by the standard GitHub integration nodes.

**Nodes Involved:**  
- Custom POST Github Request  
- Custom PATCH Github Request  
- Custom GET Github Request  
- Custom PUT Github Request  
- Custom DELETE Github Request  

**Node Details:**  
- **Type & Role:** `n8n-nodes-base.httpRequestTool`, generic HTTP request nodes.  
- **Configuration:** Disabled, no parameters set.  
- **Input Connections:** No active connections.  
- **Output Connections:** None.  
- **Version:** typeVersion 4.2.  
- **Edge Cases & Failures:** These nodes are disabled; if enabled, they require manual configuration of headers (authentication), URLs, and payloads, which can cause errors if misconfigured.

---

#### 2.5 Sticky Notes for Documentation

**Overview:**  
Multiple sticky notes are scattered around the workflow canvas to organize or annotate sections of the workflow. All have empty content in the provided JSON.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6  
- Sticky Note7  
- Sticky Note8  
- Sticky Note9  

**Node Details:**  
- **Type & Role:** `n8n-nodes-base.stickyNote`, used for annotations.  
- **Configuration:** Content fields all empty.  
- **Input/Output Connections:** None.  
- **Version:** typeVersion 1.  
- **Edge Cases:** No effect on workflow logic.

---

### 3. Summary Table

| Node Name                     | Node Type                    | Functional Role                          | Input Node(s)   | Output Node(s) | Sticky Note |
|-------------------------------|------------------------------|----------------------------------------|-----------------|----------------|-------------|
| Create File                   | githubTool                   | Create a file in GitHub repo            | Github MCP Server |                |             |
| Delete File                   | githubTool                   | Delete a file from GitHub repo          | Github MCP Server |                |             |
| Edit File                     | githubTool                   | Edit a file in GitHub repo              | Github MCP Server |                |             |
| Get File                     | githubTool                   | Retrieve a file contents from repo      | Github MCP Server |                |             |
| List Files                   | githubTool                   | List files in a repository              | Github MCP Server |                |             |
| Create Issue                 | githubTool                   | Create a new GitHub issue               | Github MCP Server |                |             |
| Edit Issue                   | githubTool                   | Edit an existing GitHub issue           | Github MCP Server |                |             |
| Comment on Existing Issue     | githubTool                   | Add a comment to an existing issue      | Github MCP Server |                |             |
| Lock Issue by number          | githubTool                   | Lock a GitHub issue by its number       | Github MCP Server |                |             |
| Get Issue                    | githubTool                   | Retrieve details of a specific issue    | Github MCP Server |                |             |
| Get Issues                   | githubTool                   | List issues in a repository              | Github MCP Server |                |             |
| Create Release               | githubTool                   | Create a release in GitHub repo          | Github MCP Server |                |             |
| Delete Release               | githubTool                   | Delete a release from GitHub repo        | Github MCP Server |                |             |
| Get Release                  | githubTool                   | Get release details                      | Github MCP Server |                |             |
| Get Many Releases            | githubTool                   | List multiple releases                   | Github MCP Server |                |             |
| Update Releases              | githubTool                   | Update release details                   | Github MCP Server |                |             |
| Get Pull Requests            | githubTool                   | List pull requests                       | Github MCP Server |                |             |
| Create PR Review             | githubTool                   | Create a pull request review             | Github MCP Server |                |             |
| Update PR Review             | githubTool                   | Update a pull request review             | Github MCP Server |                |             |
| Get a PR Review              | githubTool                   | Get details of a specific PR review      | Github MCP Server |                |             |
| Get All Reviews by PR Number | githubTool                   | List all PR reviews by PR number         | Github MCP Server |                |             |
| Get Repo                    | githubTool                   | Retrieve repository details              | Github MCP Server |                |             |
| Get Organization's Repositories | githubTool                | List organization repositories           | Github MCP Server |                |             |
| Get User Repos by URL        | githubTool                   | List user repositories by URL            | Github MCP Server |                |             |
| Get User Repos by Name       | githubTool                   | List user repositories by name           | Github MCP Server |                |             |
| Invite User to Organization  | githubTool                   | Invite a user to a GitHub organization   | Github MCP Server |                |             |
| List Popular Paths           | githubTool                   | List popular repository paths            | Github MCP Server |                |             |
| List Referrers               | githubTool                   | List referrer URLs to a repository       | Github MCP Server |                |             |
| Get License                  | githubTool                   | Get repository license information       | Github MCP Server |                |             |
| Get Profile                  | githubTool                   | Get user profile information             | Github MCP Server |                |             |
| Get Pull Requests            | githubTool                   | Get pull requests for repo                | Github MCP Server |                |             |
| Github MCP Server            | mcpTrigger                   | AI agent command trigger node             |                 | Multiple GitHub nodes |             |
| List workflows              | githubTool                   | List GitHub workflows                     | Github MCP Server |                |             |
| Get Workflow by ID          | githubTool                   | Get workflow details by ID                 | Github MCP Server |                |             |
| Get Workflow by Name        | githubTool                   | Get workflow details by name               | Github MCP Server |                |             |
| Enable Workflow by ID       | githubTool                   | Enable a workflow by ID                    | Github MCP Server |                |             |
| Enable Workflow by Name     | githubTool                   | Enable a workflow by name                  | Github MCP Server |                |             |
| Disable Workflow by ID      | githubTool                   | Disable a workflow by ID                   | Github MCP Server |                |             |
| Disable Workflow by Name    | githubTool                   | Disable a workflow by name                 | Github MCP Server |                |             |
| Dispatch Worthflow by ID    | githubTool                   | Dispatch a workflow run by ID              | Github MCP Server |                |             |
| Dispatch Worthflow by Name  | githubTool                   | Dispatch a workflow run by name            | Github MCP Server |                |             |
| Get Usage by ID             | githubTool                   | Get usage stats by ID                      | Github MCP Server |                |             |
| Get Usage by Name           | githubTool                   | Get usage stats by name                    | Github MCP Server |                |             |
| Custom POST Github Request  | httpRequestTool (disabled)   | Custom POST API call to GitHub             | None            | None           |             |
| Custom PATCH Github Request | httpRequestTool (disabled)   | Custom PATCH API call to GitHub            | None            | None           |             |
| Custom GET Github Request   | httpRequestTool (disabled)   | Custom GET API call to GitHub              | None            | None           |             |
| Custom PUT Github Request   | httpRequestTool (disabled)   | Custom PUT API call to GitHub              | None            | None           |             |
| Custom DELETE Github Request| httpRequestTool (disabled)   | Custom DELETE API call to GitHub           | None            | None           |             |
| Sticky Note / Sticky Note1-9| stickyNote                  | Annotations (empty content)                | None            | None           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create AI Agent Trigger Node**  
   - Add node: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name it: `Github MCP Server`  
   - No additional parameters required.  
   - Set up webhook credentials as needed for AI agent access.

2. **Add GitHub API Operation Nodes**  
   For each operation below:  
   - Add node type: `GitHub` (n8n built-in `githubTool`)  
   - Name nodes exactly as in the summary table (e.g., "Create File", "Delete File", "Edit File", etc.)  
   - In credentials tab: select or create GitHub OAuth2 credentials with appropriate scopes (repo, admin:org, workflow, etc.).  
   - Leave parameters blank to allow dynamic input from the trigger.  
   - Connect each node’s input to the `Github MCP Server` node’s output via the `ai_tool` connection (special connection type in n8n).  
   
3. **Configure Nodes for Workflow and Usage Management**  
   - Add GitHub nodes handling workflows: "List workflows", "Get Workflow by ID", "Enable Workflow by Name", etc.  
   - Configure credentials as above.  
   - Connect inputs to `Github MCP Server`.

4. **Add Pull Request Review Nodes**  
   - Add nodes: "Create PR Review", "Get a PR Review", "Update PR Review", etc.  
   - Configure credentials and connect input from `Github MCP Server`.

5. **Add Disabled Custom HTTP Request Nodes (Optional)**  
   - Add `HTTP Request` nodes for POST, PATCH, GET, PUT, DELETE.  
   - Set method accordingly.  
   - Keep nodes disabled by default.  
   - Configure authentication headers for GitHub API if enabling.

6. **Add Sticky Notes for Documentation**  
   - Add `Sticky Note` nodes at logical places to annotate workflow sections.  
   - Fill content as needed.

7. **Verify Connections**  
   - Ensure all GitHub nodes receive input only from the `Github MCP Server` node.  
   - No outputs from GitHub nodes are chained further; these are terminal operation nodes.

8. **Credentials Setup**  
   - Create GitHub OAuth2 credentials with required scopes: `repo`, `workflow`, `admin:org`, `read:user`, etc.  
   - Assign credentials to all GitHub nodes.

9. **Test Workflow**  
   - Trigger `Github MCP Server` with valid AI agent commands specifying the API action and parameters.  
   - Verify correct routing and successful GitHub API responses.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------|
| The node named "Dispatch Worthflow by ID" and "Dispatch Worthflow by Name" likely have a typo.     | Should be "Dispatch Workflow" for clarity.    |
| This workflow enables AI agents to fully control GitHub resources programmatically via n8n.        | Useful for automating repository management.  |
| GitHub OAuth2 credentials require appropriate scopes for each API operation to avoid permission errors. | See GitHub OAuth documentation for scopes.    |
| The `@n8n/n8n-nodes-langchain.mcpTrigger` node is designed for AI agent integration, enabling flexible command routing. | LangChain MCP integration docs.                |
| Disabled custom HTTP request nodes are placeholders for advanced/custom API calls beyond standard integrations. | Can be enabled for custom GitHub API endpoints. |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or copyrighted material. All manipulated data are legal and public.