Conversational Kubernetes Management with GPT-4o and MCP Integration

https://n8nworkflows.xyz/workflows/conversational-kubernetes-management-with-gpt-4o-and-mcp-integration-4023


# Conversational Kubernetes Management with GPT-4o and MCP Integration

### 1. Workflow Overview

This workflow enables conversational management of Kubernetes clusters via GPT-4o integrated with an MCP (Model Context Protocol) server. It is designed to allow users to interact with Kubernetes resources using natural language, with the AI assistant facilitating monitoring, inspection, and troubleshooting tasks.

The workflow is structured into these logical blocks:

- **1.1 Input Reception:** Captures chat messages from users as triggers to start the workflow.
- **1.2 AI Processing and Memory:** Uses GPT-4o as an AI agent to interpret and respond to queries, with conversational context maintained via a memory buffer.
- **1.3 Kubernetes MCP Tool Integration:** Implements multiple Kubernetes management tools using the MCP client node to execute specific cluster operations like retrieving resources, logs, metrics, and creating or updating resources.
- **1.4 AI Agent Tool Orchestration:** The AI agent node acts as the central orchestrator, invoking relevant MCP tools based on the user’s request and the AI’s interpretation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages to start the workflow interaction.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Webhook node that triggers the workflow upon receiving a chat message.  
    - *Configuration:* Uses a specific webhook ID to receive messages from the chat interface. No additional options configured.  
    - *Input/Output:* No inputs; outputs data to the AI Agent node.  
    - *Version:* 1.1  
    - *Potential Failures:* Webhook connectivity issues, message format errors, or webhook ID misconfiguration.

#### 2.2 AI Processing and Memory

- **Overview:**  
  Handles natural language understanding and response generation, maintaining conversational context using memory.

- **Nodes Involved:**  
  - AI Agent  
  - Simple Memory  
  - OpenAI Chat Model

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Central AI node interpreting user input, deciding which Kubernetes tools to invoke, and generating responses.  
    - *Configuration:*  
      - System message sets the assistant role as a Kubernetes assistant connected to MCP.  
      - Provides a detailed list of available Kubernetes tool commands.  
      - Enforces usage rules such as validating input arguments, asking for missing info, avoiding destructive operations unless authorized, and replying concisely.  
      - Uses expressions for dynamic time zone setting (`{{ $now.setZone('Asia/Tehran') }}`).  
    - *Input:* Receives chat messages from the "When chat message received" node and memory context from the Simple Memory node.  
    - *Output:* Calls relevant Kubernetes MCP tool nodes or returns text responses.  
    - *Version:* 1.8  
    - *Edge Cases:* Incorrect or incomplete user inputs, AI misinterpretations, or failure in tool invocation.  
  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Maintains short-term conversational context for the AI agent.  
    - *Configuration:* Default settings; no custom parameters.  
    - *Input/Output:* Receives data from AI Agent and provides memory context back to AI Agent.  
    - *Version:* 1.3  
    - *Potential Failures:* Memory overflow or loss of context if conversation is too long.  
  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Provides language model capabilities (GPT-4o-mini) for the AI Agent.  
    - *Configuration:*  
      - Model set to "gpt-4o-mini" with unlimited max tokens (-1).  
      - Uses configured OpenAI API credentials.  
    - *Input/Output:* Connected as the language model for the AI Agent.  
    - *Version:* 1.2  
    - *Potential Failures:* API key issues, rate limits, network errors.

#### 2.3 Kubernetes MCP Tool Integration

- **Overview:**  
  Executes Kubernetes operations through MCP client tool nodes, each corresponding to specific cluster management commands exposed by the MCP server.

- **Nodes Involved:**  
  - MCP Client  
  - getAPIResources  
  - listResources  
  - describeResource  
  - getPodMetrics  
  - getNodeMetrics  
  - createorUpdateResource  
  - getResource  
  - getPodsLogs  
  - getEvents

- **Node Details:**  
  Each node is an instance of the MCP client tool configured to call an MCP method via JSON-RPC over SSE connection. Credentials for all are set to the same Kubernetes MCP server.

  - **MCP Client**  
    - *Type:* MCP Client Tool  
    - *Role:* Base client node with SSE connection to Kubernetes MCP server.  
    - *Parameters:* Connection type set to SSE.  
    - *Credentials:* Kubernetes MCP SSE API credentials named "k8s".  
    - *Version:* 1  
    - *Input/Output:* Outputs sent to AI Agent as responses.  
    - *Potential Failures:* Connection drops, auth failures.

  For the following nodes, the "toolName" corresponds to a Kubernetes operation; parameters are dynamically populated using expressions from AI Agent variable outputs:

  - **getAPIResources**  
    - Calls `getAPIResources` method to list all supported resource kinds (both namespace- and cluster-scoped).  
    - Parameters include JSON-RPC structure with arguments to include both scopes.  
    - Position connected as AI tool output to AI Agent.  
  - **listResources**  
    - Lists all resources of a specified kind and namespace.  
    - Parameters use dynamic expressions to get "Kind" and "namespace" from AI Agent outputs.  
  - **describeResource**  
    - Provides detailed description of a resource given kind, name, and namespace.  
  - **getPodMetrics**  
    - Retrieves CPU and memory metrics for a specified pod in a namespace.  
  - **getNodeMetrics**  
    - Retrieves CPU and memory metrics for a specified node by name.  
  - **createorUpdateResource**  
    - Creates or updates a Kubernetes resource using a YAML manifest and namespace.  
    - Only node that modifies cluster state; others are read-only.  
  - **getResource**  
    - Fetches details for a specific resource kind, name, and namespace.  
  - **getPodsLogs**  
    - Retrieves logs from a specified pod in a namespace.  
  - **getEvents**  
    - Fetches cluster events within a specified namespace.

- **Common Configurations:**  
  - All nodes use the same MCP SSE API credentials.  
  - Tool parameters use the `$fromAI()` expression to dynamically pull parameters from AI agent context, ensuring flexibility based on user inputs.  
  - Connection type is Server-Sent Events (SSE) for real-time streaming.  
  - Version: 1 for all MCP client nodes.

- **Edge Cases and Failures:**  
  - Invalid or missing parameters from AI agent context causing tool call failure.  
  - Kubernetes API errors such as permission denied, resource not found, or timeout.  
  - SSE connection interruptions.  
  - Malformed YAML manifests in createorUpdateResource causing creation failures.

#### 2.4 AI Agent Tool Orchestration

- **Overview:**  
  The AI Agent node centrally orchestrates the invocation of all Kubernetes MCP tool nodes based on the user's conversational input and AI decision logic.

- **Nodes Involved:**  
  - AI Agent  
  - All MCP client tool nodes  
  - Simple Memory  
  - OpenAI Chat Model

- **Node Details:**  
  - The AI Agent receives user input and context from Simple Memory.  
  - It uses the OpenAI Chat Model for language understanding.  
  - Based on parsed intent and validated inputs, AI Agent triggers corresponding MCP client tool nodes to execute Kubernetes queries or commands.  
  - Results from MCP tools are fed back into AI Agent for summarization and user response.  
  - The AI Agent ensures conversational flow and enforces usage rules embedded in the system prompt.

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                             | Input Node(s)           | Output Node(s)            | Sticky Note                                                  |
|-------------------------|----------------------------------------|---------------------------------------------|-------------------------|---------------------------|--------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger                 | Trigger workflow on user chat message       | -                       | AI Agent                  |                                                              |
| AI Agent                | LangChain Agent                        | Parse user input, orchestrate tool calls    | When chat message received, Simple Memory, MCP clients | MCP client tool nodes, Simple Memory | System message defines Kubernetes assistant and tools list. |
| Simple Memory           | LangChain Memory Buffer Window          | Maintain conversational context              | AI Agent                 | AI Agent                  |                                                              |
| OpenAI Chat Model       | LangChain OpenAI Chat Model             | Language model for AI agent                   | -                        | AI Agent                  |                                                              |
| MCP Client             | MCP Client Tool                         | Base MCP connection node                       | -                        | AI Agent                  |                                                              |
| getAPIResources         | MCP Client Tool                         | List all supported Kubernetes resource kinds | AI Agent                 | AI Agent                  |                                                              |
| listResources           | MCP Client Tool                         | List resources of a specific kind & namespace | AI Agent                 | AI Agent                  |                                                              |
| describeResource        | MCP Client Tool                         | Get detailed description of a resource        | AI Agent                 | AI Agent                  |                                                              |
| getPodMetrics           | MCP Client Tool                         | Get CPU/memory metrics for a pod              | AI Agent                 | AI Agent                  |                                                              |
| getNodeMetrics          | MCP Client Tool                         | Get CPU/memory metrics for a node             | AI Agent                 | AI Agent                  |                                                              |
| createorUpdateResource  | MCP Client Tool                         | Create or update Kubernetes resource          | AI Agent                 | AI Agent                  |                                                              |
| getResource             | MCP Client Tool                         | Retrieve details of a specific resource       | AI Agent                 | AI Agent                  |                                                              |
| getPodsLogs             | MCP Client Tool                         | Retrieve logs from a pod                       | AI Agent                 | AI Agent                  |                                                              |
| getEvents               | MCP Client Tool                         | Fetch cluster events in a namespace           | AI Agent                 | AI Agent                  |                                                              |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Trigger Node:**  
   - Add node: **When chat message received** (LangChain Chat Trigger)  
   - Configure webhook ID as required by your environment to receive chat messages. Leave options default.

2. **Set Up AI Language Model:**  
   - Add node: **OpenAI Chat Model** (LangChain OpenAI Chat Model)  
   - Select model "gpt-4o-mini"  
   - Set max tokens to -1 (unlimited)  
   - Configure with OpenAI API credentials (OAuth or API key).

3. **Add Simple Memory Node:**  
   - Add node: **Simple Memory** (LangChain Memory Buffer Window)  
   - Use default configuration (no parameters). This node will maintain conversation context.

4. **Configure AI Agent Node:**  
   - Add node: **AI Agent** (LangChain Agent)  
   - In parameters, set the **system message** exactly as:  
     ```
     You are a Kubernetes assistant connected to an MCP (Model Context Protocol) Server. Your role is to help users monitor, inspect, and troubleshoot Kubernetes resources using the following tools.
     current time is {{ $now.setZone('Asia/Tehran') }}

     Available Tools:
     1. getEvents(namespace) – Fetch events in the given namespace.
     2. getPodsLogs(namespace, podName) – Retrieve logs for a specific pod.
     3. getResource(namespace, resourceKind, resourceName) – Get details of a specific resource.
     4. createOrUpdateResource(namespace, resourceYaml) – Create or update a resource using full YAML.
     5. getNodeMetrics(nodeName) – Retrieve CPU and memory usage of a node.
     6. getPodMetrics(namespace, podName) – Retrieve CPU and memory usage of a pod.
     7. describeResource(namespace, resourceKind, resourceName) – Detailed description of a resource.
     8. listResources(namespace, resourceKind) – List all resources of a given kind in a namespace. for example{"Kind": "Pod","namespace": "x"}
     9. getAPIResources() – List all supported resource kinds in the cluster.

     Usage Rules:
     ***keep your answers short and meaningful***
     - Always validate input arguments before calling any tool.
     - Ask the user for any missing information (e.g., namespace, podName, resourceKind).
     - Do not assume defaults; always confirm with the user.
     - Avoid destructive operations unless explicitly instructed (only createOrUpdateResource modifies the cluster).
     - Use getAPIResources() to help the user explore available resource kinds.
     - Respond with concise, plain-language explanations of the results when helpful.
     ```  
   - Connect inputs from **When chat message received** (main) and **Simple Memory** (ai_memory).  
   - Connect the **OpenAI Chat Model** node as the AI language model input.

5. **Configure MCP Client Base Node:**  
   - Add node: **MCP Client** (MCP Client Tool)  
   - Set connection type to `sse` (Server-Sent Events).  
   - Configure credentials with your MCP SSE API credentials (e.g., "k8s").

6. **Add Kubernetes MCP Tool Nodes:**  
   For each Kubernetes operation below, create an MCP Client Tool node with these settings:  
   - Connection Type: `sse`  
   - Credentials: MCP SSE API credentials (same as MCP Client)  
   - Tool Name and Operation: `executeTool`  
   - Tool Parameters: JSON or expression-based parameters as specified:

   a. **getAPIResources**  
      - Tool Name: `getAPIResources`  
      - Parameters (JSON string):  
        ```json
        {
          "jsonrpc": "2.0",
          "id": 1,
          "method": "getAPIResources",
          "params": {
            "arguments": {
              "includeNamespaceScoped": true,
              "includeClusterScoped": true
            }
          }
        }
        ```
   
   b. **listResources**  
      - Tool Name: `listResources`  
      - Parameters (expression):  
        ```json
        {
          "Kind": "{{ $fromAI('Kind', 'Select the kind given by ai agent', 'string') }}",
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```
   
   c. **describeResource**  
      - Tool Name: `describeResource`  
      - Parameters (expression):  
        ```json
        {
          "Kind": "{{ $fromAI('Kind', 'Select the kind given by ai agent', 'string') }}",
          "name": "{{ $fromAI('name', 'Select the name given by ai agent', 'string') }}",
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```
   
   d. **getPodMetrics**  
      - Tool Name: `getPodMetrics`  
      - Parameters (expression):  
        ```json
        {
          "podName": "{{ $fromAI('podName', 'Select the pod name given by ai agent', 'string') }}",
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```
   
   e. **getNodeMetrics**  
      - Tool Name: `getNodeMetrics`  
      - Parameters (expression):  
        ```json
        {
          "Name": "{{ $fromAI('Name', 'Select the name given by ai agent', 'string') }}"
        }
        ```
   
   f. **createorUpdateResource**  
      - Tool Name: `createResource`  
      - Parameters (expression):  
        ```json
        {
          "manifest": "{{ $fromAI('manifest', 'Select the manifest given by ai agent', 'string').trim() }}",
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```
   
   g. **getResource**  
      - Tool Name: `getResource`  
      - Parameters (expression):  
        ```json
        {
          "kind": "{{ $fromAI('kind', 'Select the kind given by ai agent', 'string') }}",
          "name": "{{ $fromAI('name', 'Select the name given by ai agent', 'string') }}",
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```
   
   h. **getPodsLogs**  
      - Tool Name: `getPodsLogs`  
      - Parameters (expression):  
        ```json
        {
          "Name": "{{ $fromAI('Name', 'Select the name given by ai agent', 'string') }}",
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```
   
   i. **getEvents**  
      - Tool Name: `getEvents`  
      - Parameters (expression):  
        ```json
        {
          "namespace": "{{ $fromAI('namespace', 'Select the namespace given by ai agent', 'string') }}"
        }
        ```

7. **Connect MCP Tool Nodes to AI Agent:**  
   - For each MCP Client Tool node, connect its output to the AI Agent node's ai_tool input, allowing the AI Agent to invoke these tools dynamically.

8. **Connect Simple Memory to AI Agent:**  
   - Connect Simple Memory output (ai_memory) to AI Agent input (ai_memory) to enable conversation context.

9. **Connect When chat message received to AI Agent:**  
   - Connect the main output of the chat trigger node to AI Agent’s main input.

10. **Test the workflow:**  
    - Deploy webhook and test messaging interface.  
    - Verify AI agent responds, calls Kubernetes MCP tools as expected, and maintains context.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The AI assistant uses the Asia/Tehran timezone for current time references in responses.                                              | System message in AI Agent node                                                                              |
| All Kubernetes commands are executed via MCP protocol using SSE (Server-Sent Events) for real-time streaming.                        | MCP Client nodes configuration                                                                                |
| For safe operation, the AI assistant avoids destructive commands unless explicitly instructed, limiting cluster modifications.       | Usage rules in AI Agent system message                                                                       |
| The workflow leverages LangChain nodes within n8n for chat triggers, memory, agent orchestration, and OpenAI model integration.      | Nodes: When chat message received, Simple Memory, AI Agent, OpenAI Chat Model                                |
| MCP Client Tool node requires valid MCP SSE API credentials linked to your Kubernetes cluster environment for proper function.       | Credentials required: MCP SSE API (named "k8s" in the example)                                               |
| The use of `$fromAI()` expressions allows dynamic retrieval of parameters from the AI agent's variable storage, enabling flexible inputs. | Expressions used in MCP Client Tool nodes                                                                    |

---

This documentation provides a comprehensive understanding of the workflow’s design, node configurations, interconnections, and stepwise reproduction instructions suitable for advanced users or AI agents.