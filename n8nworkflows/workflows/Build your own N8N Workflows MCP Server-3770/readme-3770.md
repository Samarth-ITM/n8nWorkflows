Build your own N8N Workflows MCP Server

https://n8nworkflows.xyz/workflows/build-your-own-n8n-workflows-mcp-server-3770


# Build your own N8N Workflows MCP Server

### 1. Workflow Overview

This workflow implements an MCP (Multi-Channel Processing) server using n8n, designed to enable AI agents or MCP clients (e.g., Claude Desktop) to discover, manage, and execute existing n8n workflows as powerful end-to-end tools rather than simple utilities. It facilitates controlled exposure of selected workflows to the agent, allowing dynamic management of an "available" workflow pool and execution of these workflows with parameter passing.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger & Agent Interface:** Handles incoming chat messages from MCP clients, routes them to the AI agent, and manages the agent’s memory and interaction.
- **1.2 Workflow Discovery & Management Tools:** Implements tools for searching workflows by tag, adding/removing workflows from the available pool, and listing currently available workflows.
- **1.3 Available Workflows Memory Management:** Uses Redis to store and maintain the list of workflows currently available to the agent.
- **1.4 Workflow Execution:** Executes workflows from the available pool using Subworkflow triggers with passthrough parameters.
- **1.5 Workflow Simplification & Metadata Extraction:** Processes workflow JSON to extract simplified metadata and input schema for agent consumption.
- **1.6 Control Logic & Error Handling:** Switches between operations requested by the agent and handles edge cases such as missing workflows or execution errors.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Trigger & Agent Interface

**Overview:**  
This block receives chat messages from MCP clients, processes them with an AI agent node, and maintains conversation context using a memory buffer.

**Nodes Involved:**  
- N8N Workflows MCP Server (MCP Trigger)  
- When chat message received (Chat Trigger)  
- AI Agent (LangChain Agent)  
- MCP Client (MCP Client Tool)  
- Simple Memory (Memory Buffer Window)  
- OpenAI Chat Model (Language Model)

**Node Details:**

- **N8N Workflows MCP Server**  
  - *Type:* MCP Trigger  
  - *Role:* Entry point for MCP client requests; exposes a webhook URL for MCP clients to connect.  
  - *Config:* Webhook path set; no input schema defined here.  
  - *Connections:* Feeds into AI Agent indirectly via MCP Client node.  
  - *Edge Cases:* Webhook availability depends on n8n production mode; ensure correct URL usage.

- **When chat message received**  
  - *Type:* Chat Trigger (LangChain)  
  - *Role:* Receives chat messages from MCP clients.  
  - *Config:* Webhook ID set; no additional parameters.  
  - *Connections:* Output to AI Agent node.  
  - *Edge Cases:* Webhook must be publicly accessible; message format must be compatible.

- **AI Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Core AI logic that interprets user input, decides which workflows/tools to use, and manages workflow execution commands.  
  - *Config:* System message instructs the agent to only use workflows, not raw commands; restricts to available workflows; instructs on workflow discovery and execution logic.  
  - *Connections:* Input from chat trigger and MCP Client; output to MCP Client.  
  - *Edge Cases:* Requires correct system message for expected behavior; failure to find suitable workflows results in user notification.

- **MCP Client**  
  - *Type:* MCP Client Tool (LangChain)  
  - *Role:* Connects the AI agent to the MCP server endpoint for streaming events.  
  - *Config:* SSE endpoint set to the production URL of the MCP server webhook.  
  - *Connections:* Feeds AI Agent node.  
  - *Edge Cases:* Endpoint URL must be accurate; connection failures will disrupt communication.

- **Simple Memory**  
  - *Type:* Memory Buffer Window (LangChain)  
  - *Role:* Maintains conversation context with a window length of 30 messages.  
  - *Config:* Context window length = 30.  
  - *Connections:* Connected to AI Agent node as memory.  
  - *Edge Cases:* Memory overflow or reset may cause loss of context.

- **OpenAI Chat Model**  
  - *Type:* Language Model (LangChain)  
  - *Role:* Provides GPT-4.1-mini model for AI responses.  
  - *Config:* Model set to "gpt-4.1-mini"; uses OpenAI credentials.  
  - *Connections:* Connected as language model for AI Agent.  
  - *Edge Cases:* API key limits, rate limits, or network issues may cause failures.

---

#### 1.2 Workflow Discovery & Management Tools

**Overview:**  
Provides tools for the AI agent to search for workflows tagged "mcp", add or remove workflows from the available pool, and list currently available workflows.

**Nodes Involved:**  
- SearchWorkflows (ToolWorkflow)  
- Add Workflow (ToolWorkflow)  
- RemoveWorkflow (ToolWorkflow)  
- List Workflows (ToolWorkflow)  
- Get MCP-tagged Workflows (n8n Node)  
- Get MCP-tagged Workflows1 (n8n Node)  
- Simplify Workflows (Set)  
- Simplify Workflows1 (Set)  
- Filter Matching Ids (Filter)  
- Filter Matching IDs (Filter)  
- Split Out (SplitOut)  
- Convert to JSON (Set)  
- Convert to JSON1 (Set)  
- listTools Success (Set)  
- listTools Success1 (Aggregate)

**Node Details:**

- **SearchWorkflows**  
  - *Type:* ToolWorkflow  
  - *Role:* Retrieves all workflows tagged "mcp" from the n8n instance to present as candidates for adding.  
  - *Config:* Calls the current workflow itself with operation "searchWorkflows".  
  - *Connections:* Input from Operations switch node.  
  - *Edge Cases:* Requires valid n8n API credentials; tag filtering must match.

- **Add Workflow**  
  - *Type:* ToolWorkflow  
  - *Role:* Adds one or more workflows by ID to the available pool stored in Redis.  
  - *Config:* Accepts workflowIds as string input; operation "addWorkflow".  
  - *Connections:* Input from Operations switch node; output to MCP Server trigger.  
  - *Edge Cases:* If no matching workflows found, triggers error node.

- **RemoveWorkflow**  
  - *Type:* ToolWorkflow  
  - *Role:* Removes workflows by ID from the available pool.  
  - *Config:* Accepts workflowIds as string input; operation "removeWorkflow".  
  - *Connections:* Input from Operations switch node.  
  - *Edge Cases:* Removing non-existent workflows handled gracefully.

- **List Workflows**  
  - *Type:* ToolWorkflow  
  - *Role:* Lists all workflows currently in the available pool.  
  - *Config:* Operation "listWorkflows".  
  - *Connections:* Input from Operations switch node.  
  - *Edge Cases:* Empty list returns empty array.

- **Get MCP-tagged Workflows / Get MCP-tagged Workflows1**  
  - *Type:* n8n Node (n8n API)  
  - *Role:* Queries n8n API for workflows tagged "mcp".  
  - *Config:* Uses n8n API credentials; filters by tag "mcp".  
  - *Connections:* Input from Operations switch node or SearchWorkflows tool.  
  - *Edge Cases:* API key must have sufficient permissions.

- **Simplify Workflows / Simplify Workflows1**  
  - *Type:* Set Node  
  - *Role:* Extracts simplified metadata from workflows: id, name, description (from sticky notes containing "try it out"), and input schema from Subworkflow trigger nodes.  
  - *Config:* Uses JavaScript expressions to parse workflow JSON.  
  - *Connections:* After Get MCP-tagged Workflows nodes.  
  - *Edge Cases:* Workflows without Subworkflow triggers result in empty input schema.

- **Filter Matching Ids / Filter Matching IDs**  
  - *Type:* Filter Node  
  - *Role:* Filters workflows based on whether their IDs are included in the requested workflowIds list.  
  - *Config:* Uses expressions to check inclusion in comma-separated workflowIds string.  
  - *Connections:* After Get MCP-tagged Workflows nodes.  
  - *Edge Cases:* Empty or malformed workflowIds cause no matches.

- **Split Out**  
  - *Type:* SplitOut Node  
  - *Role:* Splits array of workflows into individual items for filtering.  
  - *Connections:* After Convert to JSON node.  
  - *Edge Cases:* Empty arrays handled gracefully.

- **Convert to JSON / Convert to JSON1**  
  - *Type:* Set Node  
  - *Role:* Parses JSON string data from Redis or API responses into arrays for processing.  
  - *Connections:* After Get Memory or Get MCP-tagged Workflows nodes.  
  - *Edge Cases:* Malformed JSON strings cause errors.

- **listTools Success / listTools Success1**  
  - *Type:* Set / Aggregate Node  
  - *Role:* Formats the response for listing workflows, returning the parsed array of available workflows.  
  - *Connections:* After Simplify Workflows nodes.  
  - *Edge Cases:* Empty lists return empty arrays.

---

#### 1.3 Available Workflows Memory Management

**Overview:**  
Manages the "available" workflows pool stored in Redis, allowing addition, removal, retrieval, and clearing of workflows.

**Nodes Involved:**  
- Get Memory (Redis)  
- Store In Memory (Redis)  
- Store In Memory1 (Redis)  
- Delete Key (Redis)  
- Is Empty Array? (If)  
- AddTool Success (Set)  
- Remove Tool Success (Set)  
- AddTool Error (Set)

**Node Details:**

- **Get Memory**  
  - *Type:* Redis Node  
  - *Role:* Retrieves the current list of available workflows from Redis key "mcp_n8n_tools".  
  - *Config:* Operation "get", propertyName "data".  
  - *Connections:* Input from When Executed by Another Workflow node.  
  - *Edge Cases:* Key may not exist initially.

- **Store In Memory / Store In Memory1**  
  - *Type:* Redis Node  
  - *Role:* Stores updated workflows list back into Redis under key "mcp_n8n_tools".  
  - *Config:* Operation "set", value is JSON string of workflows array.  
  - *Connections:* After Simplify Workflows or filtering nodes.  
  - *Edge Cases:* Redis connection failures.

- **Delete Key**  
  - *Type:* Redis Node  
  - *Role:* Deletes the Redis key "mcp_n8n_tools" to clear available workflows.  
  - *Config:* Operation "delete".  
  - *Connections:* Triggered when no workflows are found to add.  
  - *Edge Cases:* Key may not exist.

- **Is Empty Array?**  
  - *Type:* If Node  
  - *Role:* Checks if the workflows array to add is empty; routes accordingly.  
  - *Config:* Checks if the flattened array of workflows is empty.  
  - *Connections:* Routes to Delete Key or Store In Memory1.  
  - *Edge Cases:* Empty input array triggers deletion.

- **AddTool Success / Remove Tool Success / AddTool Error**  
  - *Type:* Set Node  
  - *Role:* Sets response messages indicating success or failure of add/remove operations.  
  - *Config:* Uses expressions to count workflows affected.  
  - *Connections:* After Store In Memory or Delete Key nodes.  
  - *Edge Cases:* Provides user-friendly feedback.

---

#### 1.4 Workflow Execution

**Overview:**  
Executes workflows from the available pool using Subworkflow triggers, passing parameters via passthrough.

**Nodes Involved:**  
- When Executed by Another Workflow (ExecuteWorkflowTrigger)  
- Operations (Switch)  
- Has Workflow Available? (If)  
- Get Parameters (Set)  
- Execute Workflow with PassThrough Variables (ExecuteWorkflow)  
- executeTool Result (Aggregate)  
- ExecuteTool Error (Set)  
- Workflow Exists? (If)

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* ExecuteWorkflowTrigger  
  - *Role:* Triggered when this workflow is called as a subworkflow to perform operations like add, remove, list, search, or execute.  
  - *Config:* Defines workflowInputs: operation, workflowIds, parameters.  
  - *Connections:* Feeds into Operations switch node.  
  - *Edge Cases:* Input parameters must be well-formed.

- **Operations**  
  - *Type:* Switch Node  
  - *Role:* Routes execution based on the "operation" parameter (add, remove, list, search, execute).  
  - *Config:* Matches operation string from input JSON.  
  - *Connections:* Routes to appropriate nodes for each operation.  
  - *Edge Cases:* Unknown operations are not handled explicitly.

- **Has Workflow Available?**  
  - *Type:* If Node  
  - *Role:* Checks if the requested workflow to execute is in the available pool.  
  - *Config:* Uses expression to find workflow in Redis data matching requested workflowIds.  
  - *Connections:* Routes to Get Parameters or ExecuteTool Error.  
  - *Edge Cases:* Execution blocked if workflow not available.

- **Get Parameters**  
  - *Type:* Set Node  
  - *Role:* Extracts parameters from input for execution.  
  - *Config:* Outputs raw JSON of parameters.  
  - *Connections:* Feeds Execute Workflow with PassThrough Variables node.  
  - *Edge Cases:* Parameters must match workflow input schema.

- **Execute Workflow with PassThrough Variables**  
  - *Type:* ExecuteWorkflow Node  
  - *Role:* Executes the specified workflow by ID, passing parameters via passthrough (empty workflowInputs).  
  - *Config:* Waits for subworkflow completion; workflowId from input; empty workflowInputs to enable passthrough.  
  - *Connections:* Output to executeTool Result node.  
  - *Edge Cases:* Workflow execution errors, timeouts, or invalid parameters.

- **executeTool Result**  
  - *Type:* Aggregate Node  
  - *Role:* Aggregates all output data from executed workflow into a single response.  
  - *Connections:* Final output of execution branch.  
  - *Edge Cases:* Empty or malformed output.

- **ExecuteTool Error**  
  - *Type:* Set Node  
  - *Role:* Returns error message if workflow is not available for execution.  
  - *Connections:* Output if Has Workflow Available? is false.  
  - *Edge Cases:* Provides clear feedback to agent.

- **Workflow Exists?**  
  - *Type:* If Node  
  - *Role:* Checks if filtered workflows exist when adding/removing workflows.  
  - *Connections:* Routes to Simplify Workflows or AddTool Error.  
  - *Edge Cases:* Prevents adding/removing non-existent workflows.

---

#### 1.5 Workflow Simplification & Metadata Extraction

**Overview:**  
Extracts essential metadata and input schema from workflows to provide the agent with concise workflow descriptions and parameter requirements.

**Nodes Involved:**  
- Simplify Workflows  
- Simplify Workflows1

**Node Details:**

- **Simplify Workflows / Simplify Workflows1**  
  - *Type:* Set Node  
  - *Role:* For each workflow JSON, extracts:  
    - `id` and `name` directly  
    - `description` from sticky notes containing "try it out" (first 255 chars)  
    - `parameters` by parsing the first Subworkflow trigger node’s input schema, converting it to JSON schema format with required fields and types.  
  - *Config:* Uses JavaScript expressions to filter nodes and extract data.  
  - *Connections:* Used after fetching workflows from n8n API.  
  - *Edge Cases:* Workflows without sticky notes or subworkflow triggers yield empty or minimal descriptions.

---

#### 1.6 Control Logic & Error Handling

**Overview:**  
Manages routing of operations, checks for empty arrays, and handles error responses.

**Nodes Involved:**  
- Operations (Switch)  
- Is Empty Array? (If)  
- Workflow Exists? (If)  
- ExecuteTool Error (Set)  
- AddTool Error (Set)  
- Delete Key (Redis)  
- Filter Matching Ids / Filter Matching IDs (Filter)

**Node Details:**

- **Operations**  
  - *Type:* Switch Node  
  - *Role:* Routes based on operation parameter.  
  - *Edge Cases:* Unrecognized operations not explicitly handled.

- **Is Empty Array?**  
  - *Type:* If Node  
  - *Role:* Checks if workflows array to add is empty; triggers deletion or storage accordingly.  
  - *Edge Cases:* Prevents storing empty workflow lists.

- **Workflow Exists?**  
  - *Type:* If Node  
  - *Role:* Validates existence of workflows before adding/removing.  
  - *Edge Cases:* Prevents invalid operations.

- **ExecuteTool Error / AddTool Error**  
  - *Type:* Set Node  
  - *Role:* Provides user-friendly error messages when operations fail due to missing workflows or invalid requests.

- **Delete Key**  
  - *Type:* Redis Node  
  - *Role:* Clears Redis key when no workflows are found to add.

---

### 3. Summary Table

| Node Name                         | Node Type                     | Functional Role                                   | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                                                    |
|----------------------------------|-------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| N8N Workflows MCP Server          | MCP Trigger                   | Entry point for MCP client requests              | -                                | -                                | [Read more about the MCP server trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/)                   |
| When chat message received        | Chat Trigger                  | Receives chat messages from MCP clients          | -                                | AI Agent                         |                                                                                                                                                |
| AI Agent                         | LangChain Agent               | Processes user input, manages workflow usage     | When chat message received, MCP Client, Simple Memory, OpenAI Chat Model | MCP Client                      |                                                                                                                                                |
| MCP Client                      | MCP Client Tool               | Connects AI agent to MCP server SSE endpoint     | -                                | AI Agent                         |                                                                                                                                                |
| Simple Memory                   | Memory Buffer Window          | Maintains conversation context                    | -                                | AI Agent                         |                                                                                                                                                |
| OpenAI Chat Model               | Language Model                | Provides GPT-4.1-mini model for AI responses     | -                                | AI Agent                         |                                                                                                                                                |
| SearchWorkflows                 | ToolWorkflow                 | Retrieves workflows tagged "mcp"                  | Operations                      | -                                |                                                                                                                                                |
| Add Workflow                   | ToolWorkflow                 | Adds workflows to available pool                   | Operations                      | N8N Workflows MCP Server         |                                                                                                                                                |
| RemoveWorkflow                 | ToolWorkflow                 | Removes workflows from available pool             | Operations                      | N8N Workflows MCP Server         |                                                                                                                                                |
| List Workflows                 | ToolWorkflow                 | Lists workflows in available pool                  | Operations                      | N8N Workflows MCP Server         |                                                                                                                                                |
| Get MCP-tagged Workflows       | n8n Node (n8n API)           | Fetches workflows tagged "mcp"                     | Operations                      | Filter Matching Ids              | [Learn more about the n8n node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n)                                          |
| Get MCP-tagged Workflows1      | n8n Node (n8n API)           | Fetches workflows tagged "mcp" (for search)        | Operations                      | Simplify Workflows1              |                                                                                                                                                |
| Simplify Workflows             | Set                          | Extracts workflow metadata and input schema       | Workflow Exists?                | Store In Memory                  |                                                                                                                                                |
| Simplify Workflows1            | Set                          | Extracts workflow metadata and input schema       | Get MCP-tagged Workflows1       | listTools Success1               |                                                                                                                                                |
| Filter Matching Ids            | Filter                       | Filters workflows by requested IDs                 | Get MCP-tagged Workflows        | Workflow Exists?                 |                                                                                                                                                |
| Filter Matching IDs            | Filter                       | Filters workflows by requested IDs                 | Split Out                      | Is Empty Array?                  |                                                                                                                                                |
| Split Out                     | SplitOut                     | Splits workflows array into individual items       | Convert to JSON                 | Filter Matching IDs              |                                                                                                                                                |
| Convert to JSON               | Set                          | Parses JSON string to array                         | Get MCP-tagged Workflows        | Split Out                      |                                                                                                                                                |
| Convert to JSON1              | Set                          | Parses JSON string to array                         | Get MCP-tagged Workflows1       | Has Workflow Available?          |                                                                                                                                                |
| listTools Success             | Set                          | Formats response for list workflows                 | Simplify Workflows              | Store In Memory                  |                                                                                                                                                |
| listTools Success1            | Aggregate                    | Aggregates response for list workflows              | Simplify Workflows1             | -                              |                                                                                                                                                |
| Get Memory                   | Redis                        | Retrieves available workflows from Redis           | When Executed by Another Workflow | Operations                      |                                                                                                                                                |
| Store In Memory              | Redis                        | Stores updated workflows list in Redis             | Simplify Workflows              | AddTool Success                 |                                                                                                                                                |
| Store In Memory1             | Redis                        | Stores updated workflows list in Redis             | Is Empty Array? (false branch) | Remove Tool Success             |                                                                                                                                                |
| Delete Key                  | Redis                        | Deletes Redis key to clear workflows                | Is Empty Array? (true branch)  | Remove Tool Success             |                                                                                                                                                |
| Is Empty Array?             | If                           | Checks if workflows array is empty                   | Filter Matching IDs             | Delete Key / Store In Memory1   |                                                                                                                                                |
| AddTool Success             | Set                          | Success message after adding workflows               | Store In Memory                 | -                              |                                                                                                                                                |
| Remove Tool Success         | Set                          | Success message after removing workflows             | Store In Memory1 / Delete Key  | -                              |                                                                                                                                                |
| AddTool Error               | Set                          | Error message when no matching workflows found      | Workflow Exists? (false branch) | -                              |                                                                                                                                                |
| When Executed by Another Workflow | ExecuteWorkflowTrigger       | Triggered when called as subworkflow for operations | -                              | Get Memory                     |                                                                                                                                                |
| Operations                 | Switch                       | Routes based on operation parameter                   | When Executed by Another Workflow | Get MCP-tagged Workflows, Convert to JSON, listTools Success, etc. |                                                                                                                                                |
| Has Workflow Available?    | If                           | Checks if requested workflow is in available pool    | Convert to JSON1               | Get Parameters / ExecuteTool Error |                                                                                                                                                |
| Get Parameters             | Set                          | Extracts parameters for workflow execution            | Has Workflow Available? (true) | Execute Workflow with PassThrough Variables |                                                                                                                                                |
| Execute Workflow with PassThrough Variables | ExecuteWorkflow              | Executes workflow with passthrough parameters         | Get Parameters                 | executeTool Result             | [🚨 Ensure this node does not set the input schema!](#sticky-note-7)                                                                            |
| executeTool Result         | Aggregate                    | Aggregates execution results                           | Execute Workflow with PassThrough Variables | -                              |                                                                                                                                                |
| ExecuteTool Error          | Set                          | Error message when workflow not available for execution | Has Workflow Available? (false) | -                              |                                                                                                                                                |
| Workflow Exists?           | If                           | Checks if workflows exist before add/remove operations | Filter Matching Ids            | Simplify Workflows / AddTool Error |                                                                                                                                                |
| Delete Key                 | Redis                        | Deletes Redis key to clear workflows                   | Is Empty Array? (true branch)  | Remove Tool Success             |                                                                                                                                                |
| Sticky Note                | StickyNote                   | Documentation and instructions                         | -                              | -                              | [Read more about the MCP server trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/)                   |
| Sticky Note1               | StickyNote                   | Explains managing available workflows with Redis      | -                              | -                              | [Learn more about the n8n node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n)                                          |
| Sticky Note2               | StickyNote                   | Explains executing workflows with Execute Workflow node | -                              | -                              | [Learn more about the Execute Workflow node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/)               |
| Sticky Note3               | StickyNote                   | Notes on connecting agents with MCP clients            | -                              | -                              |                                                                                                                                                |
| Sticky Note4               | StickyNote                   | Describes the custom tools available to the agent      | -                              | -                              |                                                                                                                                                |
| Sticky Note5               | StickyNote                   | Full workflow description and usage instructions       | -                              | -                              |                                                                                                                                                |
| Sticky Note6               | StickyNote                   | Notes on workflow filtering by tag                      | -                              | -                              |                                                                                                                                                |
| Sticky Note7               | StickyNote                   | Warning about Execute Workflow node input schema       | -                              | -                              |                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Add an MCP Trigger node named "N8N Workflows MCP Server".  
   - Set webhook path (auto-generated or custom).  
   - No input schema defined here.

2. **Create Chat Trigger Node**  
   - Add a LangChain Chat Trigger node named "When chat message received".  
   - Configure webhook ID (auto-generated).  
   - Connect output to AI Agent node.

3. **Create AI Agent Node**  
   - Add LangChain Agent node named "AI Agent".  
   - Set system message to instruct the agent to only use workflows, manage available workflows, and execute workflows with parameters.  
   - Connect input from Chat Trigger and MCP Client nodes.  
   - Connect output to MCP Client node.

4. **Create MCP Client Node**  
   - Add MCP Client Tool node named "MCP Client".  
   - Set SSE endpoint to the production URL of the MCP server webhook.  
   - Connect output to AI Agent node.

5. **Create Simple Memory Node**  
   - Add Memory Buffer Window node named "Simple Memory".  
   - Set context window length to 30.  
   - Connect output to AI Agent node as memory.

6. **Create OpenAI Chat Model Node**  
   - Add LangChain OpenAI Chat Model node named "OpenAI Chat Model".  
   - Select model "gpt-4.1-mini".  
   - Set OpenAI credentials.  
   - Connect output to AI Agent node as language model.

7. **Create ExecuteWorkflowTrigger Node**  
   - Add ExecuteWorkflowTrigger node named "When Executed by Another Workflow".  
   - Define workflow inputs: operation (string), workflowIds (string), parameters (object).  
   - Connect output to Operations switch node.

8. **Create Operations Switch Node**  
   - Add Switch node named "Operations".  
   - Define outputs for operations: Add, remove, list, search, execute.  
   - Use expression to route based on input.operation.

9. **Create Workflow Discovery Nodes**  
   - Add n8n API node named "Get MCP-tagged Workflows" filtering by tag "mcp".  
   - Connect output to "Filter Matching Ids" node.  
   - Add "Filter Matching Ids" node to filter workflows by requested IDs.  
   - Connect output to "Workflow Exists?" node.

10. **Create Workflow Exists? If Node**  
    - Add If node named "Workflow Exists?" to check if filtered workflows exist.  
    - True branch connects to "Simplify Workflows" node; false branch to "AddTool Error" node.

11. **Create Simplify Workflows Node**  
    - Add Set node named "Simplify Workflows".  
    - Extract id, name, description (from sticky notes containing "try it out"), and parameters (input schema from Subworkflow trigger).  
    - Connect output to "Store In Memory" Redis node.

12. **Create Redis Nodes for Memory Management**  
    - Add Redis node "Get Memory" to retrieve "mcp_n8n_tools" key.  
    - Add Redis node "Store In Memory" to save updated workflows list.  
    - Add Redis node "Delete Key" to clear workflows list when needed.

13. **Create Add, Remove, List, Search ToolWorkflow Nodes**  
    - For each operation, add a ToolWorkflow node:  
      - "Add Workflow" with operation "addWorkflow".  
      - "RemoveWorkflow" with operation "removeWorkflow".  
      - "List Workflows" with operation "listWorkflows".  
      - "SearchWorkflows" with operation "searchWorkflows".  
    - Configure each with the current workflow ID and appropriate input schema.  
    - Connect each to the MCP Server trigger node.

14. **Create Execution Nodes**  
    - Add If node "Has Workflow Available?" to check if requested workflow is in Redis data.  
    - Add Set node "Get Parameters" to extract parameters for execution.  
    - Add ExecuteWorkflow node "Execute Workflow with PassThrough Variables" with:  
      - Workflow ID from input.  
      - Empty workflowInputs to enable passthrough.  
      - Wait for subworkflow completion enabled.  
    - Add Aggregate node "executeTool Result" to collect execution output.  
    - Add Set node "ExecuteTool Error" for error messages if workflow unavailable.

15. **Create Supporting Nodes for Filtering and JSON Parsing**  
    - Add Set nodes "Convert to JSON" and "Convert to JSON1" to parse JSON strings from Redis or API.  
    - Add SplitOut node "Split Out" to split arrays for filtering.  
    - Add Filter nodes "Filter Matching Ids" and "Filter Matching IDs" to filter workflows by IDs.  
    - Add If node "Is Empty Array?" to check for empty arrays before storing.

16. **Create Success and Error Message Nodes**  
    - Add Set nodes "AddTool Success", "Remove Tool Success", and "AddTool Error" to provide user feedback.

17. **Add Sticky Notes for Documentation**  
    - Add sticky notes with provided content to document workflow sections and instructions.

18. **Connect Nodes According to Logic**  
    - Follow the connections as described in the workflow JSON to ensure correct data flow and branching.

19. **Configure Credentials**  
    - Set up n8n API credentials with sufficient permissions to list workflows.  
    - Set up Redis credentials pointing to your Redis instance.  
    - Set up OpenAI API credentials for GPT model access.

20. **Tag Target Workflows**  
    - Ensure workflows intended for use have the tag "mcp".  
    - Ensure these workflows have Subworkflow triggers with input schemas defined.

21. **Activate Workflow and Test**  
    - Activate the MCP server workflow in production mode.  
    - Use the production webhook URL in your MCP client (e.g., Claude Desktop).  
    - Test adding, listing, searching, and executing workflows via the agent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates building an MCP server that connects AI agents to existing n8n workflows, enabling outcome-focused automation rather than utility-focused tools.                                                                                                                                                                                      | Workflow description and design philosophy.                                                                                          |
| The MCP server uses Redis as memory to track "available" workflows, limiting the agent’s access to a curated subset to avoid clashes or non-production workflows.                                                                                                                                                                                               | Workflow management strategy.                                                                                                        |
| Execution of workflows uses Subworkflow triggers with input schemas extracted from workflow JSON, enabling parameter passthrough without exposing raw input fields.                                                                                                                                                                                             | Execution design detail.                                                                                                             |
| For MCP client integration, see instructions for Claude Desktop at: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop                                                                                                                                                                         | MCP client integration guide.                                                                                                       |
| If targeted workflows lack Subworkflow triggers, the executeTool node can be customized to use HTTP requests to webhook URLs instead.                                                                                                                                                                                                                           | Customization tip.                                                                                                                   |
| Managing available workflows helps avoid duplication and confusion when many workflows exist; if not needed, the concept can be removed to allow the agent to discover all workflows.                                                                                                                                                                          | Customization advice.                                                                                                                |
| Sticky notes within the workflow provide detailed explanations and links to relevant n8n documentation for nodes such as MCP Trigger, n8n API node, and Execute Workflow node.                                                                                                                                                                                  | In-workflow documentation.                                                                                                          |
| Redis must be accessible and properly configured for the workflow to maintain state.                                                                                                                                                                                                                                                                             | Infrastructure requirement.                                                                                                         |
| OpenAI API key with access to GPT-4.1-mini or equivalent is required for the AI agent to function.                                                                                                                                                                                                                                                              | Credential requirement.                                                                                                             |
| The workflow expects workflows tagged "mcp" with Subworkflow triggers and input schemas to be present in the n8n instance for discovery and execution.                                                                                                                                                                                                         | Workflow prerequisites.                                                                                                             |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the "Build your own N8N Workflows MCP Server" workflow, enabling advanced users and AI agents to work effectively with this powerful MCP server implementation.