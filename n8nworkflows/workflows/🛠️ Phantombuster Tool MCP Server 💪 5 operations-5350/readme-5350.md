🛠️ Phantombuster Tool MCP Server 💪 5 operations

https://n8nworkflows.xyz/workflows/----phantombuster-tool-mcp-server----5-operations-5350


# 🛠️ Phantombuster Tool MCP Server 💪 5 operations

### 1. Workflow Overview

This workflow, titled **"Phantombuster Tool MCP Server"**, serves as a centralized controller for interacting with the Phantombuster API through multiple operations. It is designed to manage agents on the Phantombuster platform, enabling automated execution of key agent lifecycle operations such as retrieval, deletion, queue management, and output fetching.

**Target Use Cases:**  
- Automating the management of Phantombuster agents programmatically via webhook-triggered events  
- Integrating Phantombuster agent controls within broader automation pipelines  
- Facilitating operations like listing agents, deleting agents, getting agent details, launching agents, and fetching their outputs in an automated, server-like manner

**Logical Blocks:**

- **1.1 Input Reception:**  
  The workflow starts with a webhook trigger node designed to receive incoming requests that specify which Phantombuster operation to perform.

- **1.2 Phantombuster Operations Execution:**  
  Based on the webhook input, one of five Phantombuster Tool nodes executes the requested operation:  
  - Delete an agent  
  - Get an agent  
  - Get many agents (list)  
  - Get the output of an agent  
  - Add an agent to the launch queue

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures incoming external requests to trigger the workflow. It acts as the entry point and dispatcher for subsequent operations by receiving and interpreting webhook calls.

- **Nodes Involved:**  
  - Phantombuster Tool MCP Server (MCP Trigger node)

- **Node Details:**

  **Phantombuster Tool MCP Server**  
  - Type & Role: `@n8n/n8n-nodes-langchain.mcpTrigger` — A multi-command processor (MCP) trigger node that listens for incoming webhook requests and routes them to the appropriate downstream operation nodes.  
  - Configuration: No explicit parameters set; uses an auto-generated webhook ID (`e5b9ceab-c81e-4681-9d74-056967d540d3`).  
  - Key Expressions/Variables: None shown explicitly; implicitly routes requests based on incoming payload or command type.  
  - Inputs: None (trigger node).  
  - Outputs: Connected to all Phantombuster operation nodes via the `ai_tool` output connection.  
  - Version: Requires n8n version supporting `@n8n/n8n-nodes-langchain` MCP Trigger functionality.  
  - Edge Cases/Potential Failures:  
    - Webhook misconfiguration or absence of input payload leads to no action.  
    - Authentication or permission errors if connected downstream nodes require credentials.  
  - Sub-workflow: None.

---

#### 1.2 Phantombuster Operations Execution

- **Overview:**  
  This block contains nodes that perform specific Phantombuster API operations. Each node is dedicated to one operation and receives input routed from the MCP trigger.

- **Nodes Involved:**  
  - Delete an agent  
  - Get an agent  
  - Get many agents  
  - Get the output of an agent  
  - Add an agent to the launch queue

- **Node Details:**

  **Delete an agent**  
  - Type & Role: `n8n-nodes-base.phantombusterTool` — Performs the API call to delete a specified Phantombuster agent.  
  - Configuration: Parameters unspecified in the JSON; expected to require agent ID for deletion.  
  - Key Expressions/Variables: Likely expects input data from MCP trigger specifying agent ID.  
  - Input: Connected from MCP trigger node via `ai_tool`.  
  - Output: Not explicitly connected to further nodes.  
  - Version: Standard Phantombuster Tool node supported in recent n8n versions.  
  - Edge Cases:  
    - Agent ID missing or invalid leads to API failure.  
    - Permission or authentication errors.  
    - Network or timeout issues.  

  **Get an agent**  
  - Type & Role: `n8n-nodes-base.phantombusterTool` — Retrieves details of a specific Phantombuster agent.  
  - Configuration: Requires agent ID supplied from MCP trigger input.  
  - Input: Connected from MCP trigger node via `ai_tool`.  
  - Output: Not connected downstream.  
  - Edge Cases:  
    - Nonexistent agent ID returns empty or error.  
    - Authentication failures.  

  **Get many agents**  
  - Type & Role: `n8n-nodes-base.phantombusterTool` — Retrieves a list of multiple Phantombuster agents.  
  - Configuration: Possibly supports pagination or filters (not specified here).  
  - Input: Connected from MCP trigger node via `ai_tool`.  
  - Output: Unconnected downstream.  
  - Edge Cases:  
    - Large agent lists causing timeouts.  
    - API rate limits.  

  **Get the output of an agent**  
  - Type & Role: `n8n-nodes-base.phantombusterTool` — Fetches the output data/results from a specific agent run.  
  - Configuration: Requires agent ID and possibly run/session ID from trigger input.  
  - Input: Connected from MCP trigger node via `ai_tool`.  
  - Edge Cases:  
    - Output not available yet or agent run failed.  
    - Access restrictions or invalid IDs.  

  **Add an agent to the launch queue**  
  - Type & Role: `n8n-nodes-base.phantombusterTool` — Queues an agent to be executed/launched.  
  - Configuration: Requires agent ID and possibly parameters for launch.  
  - Input: Connected from MCP trigger node via `ai_tool`.  
  - Edge Cases:  
    - Agent already running or queued.  
    - API limits or launch errors.

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                             | Input Node(s)              | Output Node(s)            | Sticky Note                      |
|----------------------------|-------------------------------------|---------------------------------------------|----------------------------|---------------------------|---------------------------------|
| Workflow Overview 0        | Sticky Note                         | Visual placeholder (no operational role)    |                            |                           |                                 |
| Phantombuster Tool MCP Server | MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`) | Webhook input and command dispatcher        |                            | Delete an agent, Get an agent, Get many agents, Get the output of an agent, Add an agent to the launch queue |                                 |
| Delete an agent            | Phantombuster Tool                  | Deletes a specific Phantombuster agent      | Phantombuster Tool MCP Server |                           |                                 |
| Get an agent               | Phantombuster Tool                  | Retrieves details of a specific agent       | Phantombuster Tool MCP Server |                           |                                 |
| Get many agents            | Phantombuster Tool                  | Lists multiple Phantombuster agents         | Phantombuster Tool MCP Server |                           |                                 |
| Get the output of an agent | Phantombuster Tool                  | Fetches output/results of a specific agent  | Phantombuster Tool MCP Server |                           |                                 |
| Add an agent to the launch queue | Phantombuster Tool                  | Queues an agent for execution                | Phantombuster Tool MCP Server |                           |                                 |
| Sticky Note 1              | Sticky Note                         | Visual placeholder (no operational role)    |                            |                           |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add a new node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Name it `Phantombuster Tool MCP Server`.  
   - Enable webhook, copy the webhook URL for external calls.  
   - No additional parameters needed. This node will receive HTTP requests specifying the command (operation) to perform.

2. **Create the Phantombuster Tool Nodes (Five Nodes)**  
   For each operation below, add a `Phantombuster Tool` node (`n8n-nodes-base.phantombusterTool`):

   - **Delete an agent**  
     - Name the node `Delete an agent`.  
     - Configure the node to use the Phantombuster API delete agent operation.  
     - Set parameters to accept an agent ID from the incoming MCP trigger data.

   - **Get an agent**  
     - Name: `Get an agent`.  
     - Configure to retrieve agent details by ID.  
     - Set input parameter to accept agent ID.

   - **Get many agents**  
     - Name: `Get many agents`.  
     - Configure to list multiple agents. Optionally support pagination parameters.

   - **Get the output of an agent**  
     - Name: `Get the output of an agent`.  
     - Configure to fetch output for a given agent/run ID.

   - **Add an agent to the launch queue**  
     - Name: `Add an agent to the launch queue`.  
     - Configure to queue an agent launch, using agent ID and any launch parameters as input.

3. **Connect Nodes**  
   - Connect the output of the `Phantombuster Tool MCP Server` (MCP trigger) node to the input of all five Phantombuster Tool nodes via the `ai_tool` output. This routing enables the MCP trigger to dispatch commands to the appropriate node based on the incoming request.

4. **Configure Credentials**  
   - For each `Phantombuster Tool` node, configure the Phantombuster API credentials. This usually involves setting an API key or OAuth token in the node credentials section.

5. **Testing Setup**  
   - Test the webhook by sending HTTP requests specifying operations and required IDs or parameters in the request body or query parameters.  
   - Validate that the correct node executes and responds appropriately.

6. **Optional Sticky Notes**  
   - Add sticky notes named `Workflow Overview 0` and `Sticky Note 1` as placeholders for documentation or comments, positioned as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                               |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------|
| The workflow uses the MCP Trigger node to handle multiple commands via a single webhook endpoint. | n8n MCP Trigger documentation                |
| Phantombuster API operations require valid API keys and proper permissions to succeed.             | https://phantombuster.com/api-documentation   |
| Ensure n8n instance has stable internet connectivity to avoid API request timeouts or failures.    | Networking best practices for API-integrations|
| No explicit error handling nodes are present; consider adding error workflows or catch nodes.      | n8n error handling guides                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.