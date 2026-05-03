Kubernetes Management with Natural Language using GPT-4o and MCP Tools

https://n8nworkflows.xyz/workflows/kubernetes-management-with-natural-language-using-gpt-4o-and-mcp-tools-7236


# Kubernetes Management with Natural Language using GPT-4o and MCP Tools

---

### 1. Workflow Overview

This workflow, titled **"Kubernetes Management with Natural Language using GPT-4o and MCP Tools"**, enables users to interact with Kubernetes clusters through natural language chat queries. It interprets user input, determines the Kubernetes cluster context, generates appropriate `kubectl` read-only commands, and executes them via MCP (Managed Control Plane) tools, returning the results.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Listens for user chat messages containing Kubernetes queries.
- **1.2 Cluster Context Analysis:** Discovers available Kubernetes clusters dynamically, analyzes the user query to identify the target cluster context, and switches the Kubernetes context accordingly.
- **1.3 Kubectl Command Generation:** Uses an AI language model to analyze the user's query and generate a valid read-only `kubectl` command in strict JSON format.
- **1.4 Command Execution:** Executes the generated `kubectl` command on the specified cluster via MCP tools and returns the read-only results.

Two sticky notes provide high-level guidance about cluster context management and kubectl command generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives chat messages from users containing Kubernetes queries. It triggers the workflow upon new chat input.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **Name:** When chat message received  
  - **Type:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  - **Role:** Entry point node that triggers workflow on receiving chat messages.  
  - **Configuration:** Uses default settings; webhookId assigned for external chat integration.  
  - **Expressions:** Provides the user’s chat input as `chatInput` in the JSON payload for downstream nodes.  
  - **Input:** External chat webhook  
  - **Output:** Passes user query JSON to `K8s Cluster Analyzer`.  
  - **Failure modes:** Network/webhook errors, malformed input (unlikely due to chat trigger).  
  - **Notes:** Sticky Note2 suggests sample query for testing.

#### 2.2 Cluster Context Analysis

- **Overview:**  
  This block identifies the Kubernetes cluster the user intends to interact with by dynamically discovering available clusters, analyzing the user query for cluster references, and switching the Kubernetes context accordingly.

- **Nodes Involved:**  
  - `K8s Cluster Analyzer`  
  - `List K8s Clusters`  
  - `Switch K8s Context`

- **Node Details:**  

  - **Node:** List K8s Clusters  
    - **Type:** `n8n-nodes-mcp.mcpClientTool`  
    - **Role:** MCP tool invocation to list all available Kubernetes clusters from the user's kubeconfig or managed control plane.  
    - **Configuration:** Executes `list_clusters` tool with empty parameters `{}`.  
    - **Credentials:** Uses stored `K8s Creds` for MCP API access.  
    - **Input:** Triggered by `K8s Cluster Analyzer`.  
    - **Output:** Cluster list JSON passed to `K8s Cluster Analyzer`.  
    - **Failure modes:** Authentication issues, MCP API failures, connectivity timeouts.

  - **Node:** K8s Cluster Analyzer  
    - **Type:** `@n8n/n8n-nodes-langchain.agent`  
    - **Role:** AI agent that:  
      1. Calls `List K8s Clusters` tool to obtain current clusters.  
      2. Parses cluster names, extracting short names and keywords dynamically.  
      3. Matches user query against cluster names/keywords to identify which cluster to use.  
      4. Outputs JSON with either the exact cluster context or a message requesting clarification.  
    - **Configuration:** Detailed prompt instructs to use list_clusters tool, match clusters dynamically (no hardcoded names), and always output parseable raw JSON only.  
    - **Expressions:** Uses `{{ $json.chatInput }}` from the chat trigger as input query.  
    - **Input:** User query JSON and cluster list.  
    - **Output:** JSON indicating context switch or message for user.  
    - **Failure modes:** AI model errors, no clusters found, ambiguous cluster names, invalid JSON output.  
    - **Version:** Uses Langchain agent node version 2.1.

  - **Node:** Switch K8s Context  
    - **Type:** `n8n-nodes-mcp.mcpClient`  
    - **Role:** MCP tool client node that switches the Kubernetes context to the cluster identified by the analyzer.  
    - **Configuration:** Executes `switch_context` tool with parameters received from `K8s Cluster Analyzer`.  
    - **Credentials:** Uses `K8s Creds` for MCP API.  
    - **Input:** Receives JSON with cluster context or message.  
    - **Output:** Passes execution result to `K8s Query Analyzer`.  
    - **Failure modes:** Failed context switch, invalid cluster context, MCP API errors, timeouts.  

- **Notes:**  
  Sticky Note labeled "Cluster context switch" documents the logic: exit if no cluster is specified, use `list_clusters` tool to get clusters, then switch context.

#### 2.3 Kubectl Command Generation

- **Overview:**  
  This block uses an AI language model to analyze the user's Kubernetes query and generate a valid `kubectl` read-only command as JSON, strictly adhering to formatting and operation constraints.

- **Nodes Involved:**  
  - `K8s Query Analyzer`  
  - `OpenAI K8s Model`

- **Node Details:**  

  - **Node:** OpenAI K8s Model  
    - **Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - **Role:** Provides the language model GPT-4o for generating kubectl commands.  
    - **Configuration:** Model set explicitly to `gpt-4o`. No special options enabled.  
    - **Credentials:** Uses OpenAI API credentials.  
    - **Input:** Receives prompt from `K8s Query Analyzer`.  
    - **Output:** Model text output passed back to `K8s Query Analyzer`.  
    - **Failure modes:** API quota limits, authentication errors, network issues, model latency.  

  - **Node:** K8s Query Analyzer  
    - **Type:** `@n8n/n8n-nodes-langchain.agent`  
    - **Role:** AI agent that:  
      1. Receives chat input and current cluster context.  
      2. Analyzes user query to generate a valid, read-only `kubectl` command in strict JSON format.  
      3. Enforces rules such as no execution, no modifications, outputting only raw JSON without markdown or explanations.  
    - **Configuration:** Complex prompt with detailed instructions on allowed kubectl commands, formatting, error handling, and examples. Uses expression `{{ $('When chat message received').item.json.chatInput }}` for user query.  
    - **Input:** Receives user query and context from previous nodes.  
    - **Output:** JSON containing the `kubectl` command string.  
    - **Failure modes:** Invalid JSON output, command syntax errors, AI misunderstanding, API errors.  
    - **Version:** Langchain agent node version 2.1.

- **Notes:**  
  Sticky Note "kubectl command generate" explains that this block analyzes chat input, generates kubectl command, and passes it to MCP tool.

#### 2.4 Command Execution

- **Overview:**  
  Executes the generated kubectl command in read-only mode using MCP tooling and returns the command output.

- **Nodes Involved:**  
  - `Kubectl MCP Tool`

- **Node Details:**  

  - **Node:** Kubectl MCP Tool  
    - **Type:** `n8n-nodes-mcp.mcpClient`  
    - **Role:** Executes the kubectl command generated by AI as a read-only operation using MCP tooling.  
    - **Configuration:** Tool name `run_kubectl_command_ro` (read-only), operation `executeTool`.  
    - **Parameters:** Takes `toolParameters` from the `output` field of the previous node (`K8s Query Analyzer`).  
    - **Credentials:** Uses `K8s Creds` for MCP API authentication.  
    - **Input:** Receives JSON command from `K8s Query Analyzer`.  
    - **Output:** Passes command result downstream or to user interface (not shown in JSON).  
    - **Retry:** Enabled retry on failure.  
    - **Failure modes:** Command execution errors, MCP API errors, timeout, invalid command parameters.

---

### 3. Summary Table

| Node Name               | Node Type                                      | Functional Role                         | Input Node(s)                | Output Node(s)           | Sticky Note                                                                                  |
|-------------------------|------------------------------------------------|---------------------------------------|-----------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger          | Entry point: receives user chat input | External webhook             | K8s Cluster Analyzer      | "Try this out using Chat: How many pods are showing a Pending status in the `abc` namespace on the production cluster?" |
| K8s Cluster Analyzer     | @n8n/n8n-nodes-langchain.agent                 | Analyze user query for cluster context | When chat message received, List K8s Clusters, OpenAI Chat Model | Switch K8s Context       | "Cluster context switch: Exit if no cluster name; use list_clusters tool; switch context"    |
| List K8s Clusters        | n8n-nodes-mcp.mcpClientTool                     | Retrieve available K8s clusters        | K8s Cluster Analyzer        | K8s Cluster Analyzer      | "Cluster context switch: Exit if no cluster name; use list_clusters tool; switch context"    |
| Switch K8s Context       | n8n-nodes-mcp.mcpClient                         | Switch Kubernetes cluster context      | K8s Cluster Analyzer        | K8s Query Analyzer        | "Cluster context switch: Exit if no cluster name; use list_clusters tool; switch context"    |
| K8s Query Analyzer       | @n8n/n8n-nodes-langchain.agent                  | Generate kubectl command from query    | Switch K8s Context, OpenAI K8s Model | Kubectl MCP Tool        | "kubectl command generate: Analyze chat input, generate kubectl command, pass to run_kubectl_command_ro tool" |
| OpenAI K8s Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi           | Provide GPT-4o model for command gen   | K8s Query Analyzer          | K8s Query Analyzer        | "kubectl command generate: Analyze chat input, generate kubectl command, pass to run_kubectl_command_ro tool" |
| Kubectl MCP Tool         | n8n-nodes-mcp.mcpClient                         | Execute read-only kubectl command      | K8s Query Analyzer          | (Output to user/interface) | "kubectl command generate: Analyze chat input, generate kubectl command, pass to run_kubectl_command_ro tool" |
| Sticky Note              | n8n-nodes-base.stickyNote                       | Documentation note                     | N/A                         | N/A                      | "Cluster context switch: Exit if no cluster name; list_clusters tool; switch_context tool"   |
| Sticky Note1             | n8n-nodes-base.stickyNote                       | Documentation note                     | N/A                         | N/A                      | "kubectl command generate: Analyze chat input, generate kubectl command, pass to run_kubectl_command_ro tool" |
| Sticky Note2             | n8n-nodes-base.stickyNote                       | Documentation note                     | N/A                         | N/A                      | "Try this out using Chat: How many pods are showing a Pending status in the `abc` namespace on the production cluster?" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add node `When chat message received` (type `@n8n/n8n-nodes-langchain.chatTrigger`).  
   - Configure webhook with unique webhook ID for receiving chat messages.  
   - No special parameters needed.

2. **Add MCP Tool Node to List Clusters:**  
   - Add node `List K8s Clusters` of type `n8n-nodes-mcp.mcpClientTool`.  
   - Set `toolName` to `list_clusters`.  
   - Set `operation` to `executeTool`.  
   - Set `toolParameters` to `{}` (empty JSON object).  
   - Assign credentials using your MCP API credentials named `K8s Creds`.

3. **Add AI Agent Node for Cluster Analysis:**  
   - Add node `K8s Cluster Analyzer` of type `@n8n/n8n-nodes-langchain.agent`.  
   - Paste the detailed prompt instructing the agent to use the `List K8s Clusters` tool to dynamically discover clusters, parse user query to find cluster context, and output only raw JSON with context or message.  
   - Set `promptType` to `define`.  
   - Use expression to reference user query input as `{{ $json.chatInput }}` (from trigger).  
   - No additional options needed.  
   - Link credentials for OpenAI API if required (optional depending on agent config).

4. **Connect Nodes for Cluster Context Analysis:**  
   - Connect `When chat message received` main output to `K8s Cluster Analyzer`.  
   - Connect `K8s Cluster Analyzer` AI tool output to `List K8s Clusters`.  
   - Connect `List K8s Clusters` output back to `K8s Cluster Analyzer` AI tool input for cluster list.  
   - Connect `K8s Cluster Analyzer` main output to `Switch K8s Context`.

5. **Add MCP Client Node to Switch Context:**  
   - Add node `Switch K8s Context` of type `n8n-nodes-mcp.mcpClient`.  
   - Set `toolName` to `switch_context`.  
   - Set `operation` to `executeTool`.  
   - Set `toolParameters` expression to `={{ $json.output }}` to receive cluster context JSON from the analyzer.  
   - Use `K8s Creds` for credentials.

6. **Add AI Model Node for Kubectl Command Generation:**  
   - Add node `OpenAI K8s Model` of type `@n8n/n8n-nodes-langchain.lmChatOpenAi`.  
   - Set model to `gpt-4o`.  
   - Use OpenAI API credentials (`OpenAi account`).  
   - No additional options.

7. **Add AI Agent Node for Kubectl Query Analysis:**  
   - Add node `K8s Query Analyzer` of type `@n8n/n8n-nodes-langchain.agent`.  
   - Provide detailed prompt instructing to generate ONLY valid read-only kubectl commands in strict JSON format, with examples and error handling.  
   - Use expression to pull user query as `{{ $('When chat message received').item.json.chatInput }}`.  
   - Ensure `promptType` is `define`.  
   - Connect input from `Switch K8s Context` and `OpenAI K8s Model`.

8. **Connect Context Switching to Kubectl Query Analyzer:**  
   - Connect `Switch K8s Context` main output to `K8s Query Analyzer` main input.  
   - Connect `OpenAI K8s Model` output to `K8s Query Analyzer` AI language model input.

9. **Add MCP Client Node for Kubectl Command Execution:**  
   - Add node `Kubectl MCP Tool` of type `n8n-nodes-mcp.mcpClient`.  
   - Set `toolName` to `run_kubectl_command_ro` indicating read-only command execution.  
   - Set `operation` to `executeTool`.  
   - Set `toolParameters` to `={{ $json.output }}` to receive the JSON command from `K8s Query Analyzer`.  
   - Use `K8s Creds` for credentials.  
   - Enable retry on failure for robustness.

10. **Connect Kubectl Query Analyzer to Kubectl MCP Tool:**  
    - Connect main output of `K8s Query Analyzer` to `Kubectl MCP Tool`.

11. **Add Sticky Notes for Documentation (Optional):**  
    - Add sticky notes describing cluster context switch logic, kubectl command generation, and usage examples.

12. **Validate and Test:**  
    - Deploy workflow, ensure credentials are properly configured.  
    - Test with chat inputs such as:  
      `"How many pods are showing a Pending status in the abc namespace on the production cluster?"`  
    - Verify cluster detection, context switching, command generation, and command execution.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Cluster context switch: exit if no cluster name provided; use `list_clusters` tool; switch context | Sticky Note attached to nodes handling cluster context discovery and switching.                              |
| Kubectl command generation: analyze chat input, generate kubectl command, pass to MCP tool       | Sticky Note attached to AI nodes generating commands and MCP tool executing them.                           |
| Example query for testing chat input: "How many pods are showing a Pending status in the `abc` namespace on the production cluster?" | Sticky Note near chat trigger node with usage suggestion.                                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.

---