Create LinkedIn Posts with AI Agents using MCP Server

https://n8nworkflows.xyz/workflows/create-linkedin-posts-with-ai-agents-using-mcp-server-5210


# Create LinkedIn Posts with AI Agents using MCP Server

### 1. Workflow Overview

This workflow automates the creation of LinkedIn posts using AI agents via an MCP (Model Cluster Proxy) Server. It is designed for seamless integration between AI-driven content generation and LinkedIn publishing through n8n’s LinkedIn Tool node. The workflow is structured into two main logical blocks:

- **1.1 MCP Server Input Reception:** Handles incoming requests from AI agents through an MCP trigger node acting as a webhook server.
- **1.2 LinkedIn Post Creation:** Uses the LinkedIn Tool node to create posts on LinkedIn based on AI-generated parameters received from the MCP trigger.

This setup enables automatic, parameter-driven LinkedIn post creation with minimal manual configuration, leveraging AI to generate post content and metadata dynamically.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Input Reception

- **Overview:**  
  This block receives incoming HTTP requests from AI agents using the MCP trigger node. It acts as the entry point for all LinkedIn post creation requests, providing a secure and standardized interface.

- **Nodes Involved:**  
  - LinkedIn Tool MCP Server

- **Node Details:**  
  - **LinkedIn Tool MCP Server**  
    - *Type:* MCP Trigger (n8n-nodes-langchain.mcpTrigger)  
    - *Role:* Acts as a webhook endpoint to receive AI agent requests for LinkedIn operations.  
    - *Configuration:*  
      - Path set to `"linkedin-tool-mcp"`, exposing this route as the webhook URL.  
      - Webhook ID assigned for internal instance management.  
    - *Input:* External HTTP requests from AI agents via MCP protocol.  
    - *Output:* Passes received parameters to downstream nodes for processing.  
    - *Version Requirements:* Requires n8n version supporting MCP trigger nodes and the @n8n/n8n-nodes-langchain package.  
    - *Edge Cases / Failure Modes:*  
      - Webhook connectivity issues (e.g., server offline).  
      - Malformed or incomplete AI request payloads causing expression failures downstream.  
      - Authentication or authorization issues if MCP server security is enabled.  
    - *Sub-workflow:* None.

#### 2.2 LinkedIn Post Creation

- **Overview:**  
  This block processes the input parameters received from the MCP server and creates a LinkedIn post using the LinkedIn Tool node. The parameters (post text, target person, media category, etc.) are dynamically filled using AI expressions.

- **Nodes Involved:**  
  - Create a post  
  - Sticky Note 1 (for documentation)

- **Node Details:**  
  - **Create a post**  
    - *Type:* LinkedIn Tool (n8n-nodes-base.linkedInTool)  
    - *Role:* Executes the LinkedIn API call to create a post with parameters sourced from AI agent inputs.  
    - *Configuration:*  
      - `text`: Populated dynamically using the expression `$fromAI('Text', '', 'string')`, extracting the post content from AI input.  
      - `person`: Extracts the LinkedIn person ID or identifier via `$fromAI('Person', '', 'string')`.  
      - `binaryPropertyName`: Optional binary data property name from AI input.  
      - `shareMediaCategory`: Optional media category for the post, also from AI input.  
      - `additionalFields`: Empty by default but available for further customization.  
    - *Input:* Receives parameters from MCP trigger node.  
    - *Output:* LinkedIn API response, including post creation confirmation or errors.  
    - *Credentials:* Requires configured LinkedIn Tool credentials (OAuth2 recommended).  
    - *Version Requirements:* Compatible with LinkedIn Tool node versions supporting these parameters.  
    - *Edge Cases / Failure Modes:*  
      - API authentication errors due to expired or missing credentials.  
      - Invalid or missing required parameters causing API errors.  
      - Network timeouts or LinkedIn API rate limiting.  
      - Expression evaluation failures if AI input is malformed.  
    - *Sub-workflow:* None.

  - **Sticky Note 1**  
    - *Type:* Sticky Note  
    - *Role:* Provides a simple label “Post” for organizational clarity near the Create a post node.  
    - *Configuration:* Text content “## Post”, colored with color index 4 (blue).  
    - *Input / Output:* None (annotation only).  
    - *Edge Cases:* None.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                           | Input Node(s)           | Output Node(s)       | Sticky Note                                                                                                                           |
|-------------------------|--------------------------------|-----------------------------------------|-------------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0     | Sticky Note                    | Documentation and setup instructions    |                         |                      | ## 🛠️ LinkedIn Tool MCP Server<br>### 📋 Available Operations (1 total)<br>**Post**: create<br>### ⚙️ Setup Instructions<br>1. Import Workflow into n8n<br>2. Configure LinkedIn Tool credentials<br>3. Activate workflow<br>4. Use webhook URL for MCP connection<br>### Features: Zero config, AI param filling, error handling.<br>### Help: [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/), [Discord](https://discord.me/cfomodz) |
| LinkedIn Tool MCP Server | MCP Trigger                   | Receives AI requests via MCP webhook    |                         | Create a post (ai_tool) |                                                                                                                                         |
| Create a post           | LinkedIn Tool                 | Creates LinkedIn post using AI inputs   | LinkedIn Tool MCP Server |                      |                                                                                                                                         |
| Sticky Note 1           | Sticky Note                   | Annotation for the Create a post node   |                         |                      | ## Post                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note “Workflow Overview 0”**  
   - Type: Sticky Note  
   - Content: Markdown text describing workflow purpose, setup instructions, features, and help links (see Section 3 Sticky Note content).  
   - Position: Near top-left area for visibility.  
   - Size: Width 420, Height 780.

2. **Add MCP Trigger Node “LinkedIn Tool MCP Server”**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `linkedin-tool-mcp` (this sets the webhook URL path).  
   - Position: Center-left area.  
   - No credentials required.  
   - Save to generate webhook URL.

3. **Add LinkedIn Tool Node “Create a post”**  
   - Type: LinkedIn Tool (`n8n-nodes-base.linkedInTool`)  
   - Parameters:  
     - Text: set expression `={{ $fromAI('Text', '', 'string') }}`  
     - Person: set expression `={{ $fromAI('Person', '', 'string') }}`  
     - Binary Property Name: `={{ $fromAI('Binary_Property_Name', '', 'string') }}` (optional, default empty)  
     - Share Media Category: `={{ $fromAI('Share_Media_Category', '', 'string') }}` (optional, default empty)  
     - Additional Fields: leave empty or customize as needed.  
   - Credentials: Configure LinkedIn OAuth2 credentials with appropriate API access.  
   - Position: Below MCP Trigger node.

4. **Add Sticky Note “Post” near the LinkedIn Tool node**  
   - Type: Sticky Note  
   - Content: “## Post”  
   - Color: Blue (color index 4)  
   - Position: Adjacent to the “Create a post” node.

5. **Connect nodes**  
   - Link the MCP Trigger node’s `ai_tool` output to the LinkedIn Tool node’s input (establish the flow of AI parameters).  
   - The Sticky Notes are not connected but placed for clarity.

6. **Activate the workflow**  
   - Ensure LinkedIn credentials are valid and authorized.  
   - Enable the workflow.  
   - Copy the webhook URL from the MCP Trigger node to configure AI agents to send post creation requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses `$fromAI()` expressions to dynamically populate LinkedIn post parameters from AI agent requests.                                                        | n8n’s MCP Server integration with AI agents                                                                                             |
| For full LinkedIn Tool capabilities, refer to n8n LinkedIn Tool documentation and ensure OAuth2 credentials have necessary permissions to post on behalf of users.         | [LinkedIn Tool Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.linkedin/)                                        |
| MCP Server setup requires the @n8n/n8n-nodes-langchain package and n8n version supporting MCP trigger nodes.                                                              | See [n8n MCP Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)                        |
| For community help and customizations, contact the maintainer on Discord: https://discord.me/cfomodz                                                                        | Discord community for MCP integration support                                                                                           |
| The sticky note “Workflow Overview 0” provides ready-to-use instructions and operation overview for users importing this workflow.                                         | Embedded in the workflow for user guidance                                                                                              |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, respecting all valid content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.