MCP Server Github Agent Tool

https://n8nworkflows.xyz/workflows/mcp-server-github-agent-tool-4475


# MCP Server Github Agent Tool

### 1. Workflow Overview

This workflow, titled **MCP Server Github Agent Tool**, implements a specialized GitHub agent designed to process natural language requests related to GitHub repository and issue management. It acts as an interface between user commands expressed in plain English and the GitHub API, leveraging a multi-component agent pattern to handle complex GitHub operations efficiently.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Triggering:** Receives natural language GitHub operation requests either via an MCP Server Trigger webhook or from another workflow execution.
- **1.2 User Context Initialization:** Sets the GitHub username context necessary for API interactions.
- **1.3 AI Processing and Memory Management:** Uses OpenAI’s GPT-4.1 model combined with a memory buffer to interpret commands, maintain conversation context, and drive the agent logic.
- **1.4 Specialized GitHub Agent Execution:** Runs a dedicated GitHub agent workflow that interprets commands and orchestrates GitHub API calls.
- **1.5 GitHub API Integration:** Connects to GitHub API tools through an MCP client node to execute the actual repository and issue management operations.

This modular design enables scalability, reduces language model context overhead by delegating tasks, and simplifies complex GitHub API interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:**  
  This block receives incoming GitHub operation requests as natural language strings. It supports two entry points: an MCP Server webhook trigger and an execution trigger when called from another workflow.

- **Nodes Involved:**  
  - MCP Server Trigger  
  - When Executed by Another Workflow

- **Node Details:**

  - **MCP Server Trigger**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger` (Webhook trigger)  
    - Role: Listens for HTTP webhook requests containing the “request” parameter, representing the user’s natural language GitHub command.  
    - Configuration: Path set to a unique webhook ID (`5b0e6a8d-f7eb-49ba-ac89-2eb4fc359967`).  
    - Input/Output: Outputs the request JSON to downstream nodes.  
    - Edge Cases: Webhook call failures, malformed requests without "request" parameter.

  - **When Executed by Another Workflow**  
    - Type: `n8n-nodes-base.executeWorkflowTrigger`  
    - Role: Enables this workflow to be triggered programmatically from other workflows with the "request" input parameter.  
    - Configuration: Defines expected workflow input named "request".  
    - Input/Output: Accepts workflow input JSON, outputs it downstream.  
    - Edge Cases: Missing or invalid "request" parameter; failures in upstream workflows.

#### 2.2 User Context Initialization

- **Overview:**  
  This block sets the GitHub username context used for API authentication and to personalize API calls.

- **Nodes Involved:**  
  - Set Github Username

- **Node Details:**

  - **Set Github Username**  
    - Type: `n8n-nodes-base.set`  
    - Role: Assigns a static GitHub username value to the workflow context under `githubUsername`.  
    - Configuration: Hardcoded username `"wayum999"` (should be replaced with actual user info).  
    - Input/Output: Receives input from the "When Executed by Another Workflow" node; outputs JSON with `githubUsername`.  
    - Edge Cases: Username must be valid GitHub username; static assignment limits multi-user support.

#### 2.3 AI Processing and Memory Management

- **Overview:**  
  This block interprets natural language commands using OpenAI’s GPT-4.1 chat model and maintains conversation context using a memory buffer, enabling multi-step interactions and reducing redundant context passing.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides natural language understanding powered by GPT-4.1 to interpret user requests.  
    - Configuration: Model set to `"gpt-4.1"` with OpenAI API credentials (`OpenAi - WILLBOT`).  
    - Input/Output: Receives prompt from upstream; outputs interpreted result to the agent.  
    - Edge Cases: API key invalid or rate-limited, network timeouts, unexpected model output.

  - **Simple Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Stores conversation context keyed by `"github"` and a custom session ID, enabling contextual continuity across requests.  
    - Configuration: Session key `"github"`; sessionIdType `"customKey"`.  
    - Input/Output: Receives input from triggers; feeds memory context to the GitHub AI Agent.  
    - Edge Cases: Memory overflow if conversation too long, session key collisions.

#### 2.4 Specialized GitHub Agent Execution

- **Overview:**  
  This block uses an agent node configured to act as a GitHub API and repo management specialist. It interprets the processed command and coordinates the execution of GitHub operations.

- **Nodes Involved:**  
  - Github AI Agent  
  - Github Agent (toolWorkflow)

- **Node Details:**

  - **Github AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Acts as the main AI agent, consuming the interpreted request text and user context to drive GitHub operations.  
    - Configuration:  
      - Input text dynamically set from `When Executed by Another Workflow` node's `"request"` field.  
      - System message defines agent role as "GitHub API and repo management specialist" with user info interpolation (`githubUsername`).  
      - Prompt type: `"define"`.  
    - Input/Output: Receives input from memory and OpenAI Chat Model; outputs commands to the GitHub Agent tool workflow.  
    - Edge Cases: Incorrect prompt formatting, missing user context, AI misinterpretation.

  - **Github Agent**  
    - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
    - Role: Dedicated workflow tool to handle direct GitHub API interactions based on commands from the AI agent.  
    - Configuration:  
      - References an external workflow (ID `"UyrFAXDtcVf0TqlJ"`) specialized for GitHub operations.  
      - Accepts a `"request"` parameter as plain language command input.  
      - Provides example requests in description for usage guidance.  
    - Input/Output: Receives `"request"` from `Github AI Agent`; outputs results or API responses.  
    - Edge Cases: Workflow failure, misrouting of commands, input validation errors.

#### 2.5 GitHub API Integration

- **Overview:**  
  This block connects to actual GitHub API endpoints through an MCP client node that handles a wide range of GitHub operations such as issue creation, branch management, pull requests, and repository searches.

- **Nodes Involved:**  
  - Github API

- **Node Details:**

  - **Github API**  
    - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
    - Role: Acts as the GitHub API client interface, exposing multiple GitHub tools for repository and issue management.  
    - Configuration:  
      - Connects to MCP server SSE endpoint at `http://localhost:8000/sse`.  
      - Includes a large selection of GitHub API tools such as `create_issue`, `list_issues`, `create_branch`, `push_files`, `create_pull_request`, etc.  
    - Input/Output: Receives tool execution commands from the `Github AI Agent`; returns API responses.  
    - Edge Cases: Network or server errors, API rate limits, invalid authentication, partial operation failures.

---

### 3. Summary Table

| Node Name                    | Node Type                                    | Functional Role                      | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                              |
|------------------------------|----------------------------------------------|------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| MCP Server Trigger            | @n8n/n8n-nodes-langchain.mcpTrigger         | Input reception via webhook        |                              | Github Agent                 | ## 🖥️  MCP Server Trigger<br>🔧 MCP Server Trigger<br>- Receives GitHub operation requests<br>- Single parameter: "request" (natural language)<br>- Triggers specialized GitHub agent workflow |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger     | Input reception from workflows     |                              | Set Github Username          |                                                                                                                                          |
| Set Github Username           | n8n-nodes-base.set                           | User context initialization        | When Executed by Another Workflow | Github AI Agent            | ## Set GitHub Username Node<br>⚙️ Configuration Required<br>- Replace "your-github-username" with actual username<br>- Used for GitHub API authentication context<br>- Essential for proper API calls |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi       | AI language understanding          |                              | Github AI Agent              | ## OpenAI Chat Model Node<br>🧠 AI Processing Engine<br>- Requires OpenAI API key configuration<br>- Powers intelligent GitHub operation understanding<br>- Optimized for GitHub-specific tasks   |
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Conversation context maintenance   |                              | Github AI Agent              | ## Simple Memory Node<br>💾 Conversation Context<br>- Maintains operation history<br>- Enables multi-step GitHub workflows<br>- Reduces repetitive context processing                            |
| Github AI Agent              | @n8n/n8n-nodes-langchain.agent               | Specialized AI agent execution     | Set Github Username, Simple Memory, OpenAI Chat Model | Github Agent              | ## GitHub AI Agent (Tools Agent)<br>🤖 Specialized GitHub Agent<br>- Direct connection to GitHub MCP Server<br>- Processes natural language GitHub requests<br>- Reduces LLM context overhead<br>- Handles all GitHub API complexity |
| Github Agent                 | @n8n/n8n-nodes-langchain.toolWorkflow        | GitHub API command execution tool  | MCP Server Trigger, Github AI Agent | MCP Server Trigger (loopback) |                                                                                                                                          |
| Github API                  | @n8n/n8n-nodes-langchain.mcpClientTool       | GitHub API integration             | Github AI Agent              |                              | ## MCP Client Node<br>💾 Github Tools<br>- Handles calls to the Github API<br>- Manages repos and issues<br>- One node connected to an MCP server |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an MCP Server Trigger node:**  
   - Type: `mcpTrigger` (Langchain MCP Server Trigger)  
   - Set webhook path to a unique identifier (e.g., `5b0e6a8d-f7eb-49ba-ac89-2eb4fc359967`)  
   - This node accepts incoming natural language requests under the parameter `"request"`.  

2. **Create an Execute Workflow Trigger node:**  
   - Type: `executeWorkflowTrigger`  
   - Configure it to accept workflow inputs, defining a single input parameter `"request"`.  
   - This allows the workflow to be triggered programmatically with a natural language request from other workflows.  

3. **Create a Set node for GitHub username:**  
   - Type: `set`  
   - Assign a new field `githubUsername` with your actual GitHub username as a string value (e.g., `"wayum999"`).  
   - Connect the Execute Workflow Trigger node’s output to this Set node.  

4. **Create an OpenAI Chat Model node:**  
   - Type: `lmChatOpenAi` (Langchain OpenAI Chat Model)  
   - Select model `"gpt-4.1"`.  
   - Configure OpenAI API credentials with a valid API key (e.g., credential named `"OpenAi - WILLBOT"`).  
   - No additional options needed unless specific request parameters required.  

5. **Create a Simple Memory node:**  
   - Type: `memoryBufferWindow` (Langchain memory buffer node)  
   - Set `sessionKey` to `"github"`.  
   - Use `sessionIdType` as `"customKey"` to maintain session context across requests.  

6. **Create a Github AI Agent node:**  
   - Type: `agent` (Langchain Agent node)  
   - Set the input text parameter to: `{{$node["When Executed by Another Workflow"].item.json.request}}` for dynamic input.  
   - Configure system message:  
     ```
     # Overview
     You are a Github API and repo management specialist. You use your many tools to achieve the goals set forth by the requests you receive.

     ## User Information
     Github username: {{ $json.githubUsername }}
     ```  
   - Set prompt type as `"define"`.  

7. **Create a Github Agent node (Tool Workflow):**  
   - Type: `toolWorkflow` (Langchain Tool Workflow node)  
   - Set the workflow ID to the dedicated GitHub agent workflow (you must create or import this workflow separately).  
   - Define input parameter `"request"` as string with a default or placeholder.  
   - Configure the node description with usage examples for clarity.  

8. **Create a Github API node (MCP Client Tool):**  
   - Type: `mcpClientTool` (Langchain MCP Client Tool)  
   - Set `sseEndpoint` to your MCP server SSE URL, e.g., `http://localhost:8000/sse`.  
   - Enable relevant GitHub API tools such as `create_issue`, `list_issues`, `create_branch`, `push_files`, etc., to cover your use cases.  

9. **Connect the nodes as follows:**  
   - `When Executed by Another Workflow` → `Set Github Username` → `Github AI Agent`  
   - `Simple Memory` → `Github AI Agent`  
   - `OpenAI Chat Model` → `Github AI Agent`  
   - `Github AI Agent` → `Github Agent` (toolWorkflow)  
   - `Github Agent` → `MCP Server Trigger` (for loopback or further chaining)  
   - `Github AI Agent` (ai_tool) → `Github API` node  

10. **Credential Setup:**  
    - OpenAI Chat Model node requires valid OpenAI API key credentials.  
    - MCP Client Tool node assumes connectivity to the MCP Server managing GitHub API authentication and sessions. Configure accordingly.  

11. **Sub-workflow Setup:**  
    - The `Github Agent` node references a specialized external workflow for GitHub command processing.  
    - Ensure this sub-workflow accepts `"request"` input and returns responses properly.  
    - The sub-workflow should implement detailed GitHub API calls using nodes similar to the MCP Client Tool or other HTTP request nodes.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow implements the MCP Agent pattern, focusing on scalable and context-reducing GitHub automation.                                                                                                                 | Sticky Note4 content                                                                                 |
| Replace the `"githubUsername"` in the Set node with your actual GitHub username for correct API context.                                                                                                                     | Sticky Note content for "Set GitHub Username" node                                                  |
| The OpenAI Chat Model node requires proper OpenAI API key credentials to function and supports GPT-4.1 model usage optimized for GitHub-related tasks.                                                                        | Sticky Note2 content                                                                                 |
| The Simple Memory node maintains conversation history to enable multi-step GitHub workflows, avoiding repeated context loading.                                                                                               | Sticky Note5 content                                                                                 |
| The MCP Client Tool node centralizes GitHub API calls and should be connected to a running MCP Server instance exposing SSE endpoint for smooth API communication.                                                             | Sticky Note6 content                                                                                 |
| For workflow extensibility, add more GitHub API tools to the MCP Client Tool node as needed, mapping complex GitHub operations to plain language commands via the AI agent.                                                    | Node description and MCP client configuration                                                      |
| Example natural language requests for GitHub operations include creating issues, closing issues with comments, branch management, and pull request operations.                                                                | Github Agent node description examples                                                             |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.