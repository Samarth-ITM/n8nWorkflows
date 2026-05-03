🛠️ Microsoft To Do Tool MCP Server 💪 all 15 operations

https://n8nworkflows.xyz/workflows/----microsoft-to-do-tool-mcp-server----all-15-operations-5177


# 🛠️ Microsoft To Do Tool MCP Server 💪 all 15 operations

### 1. Workflow Overview

This workflow serves as a comprehensive Microsoft To Do Tool MCP Server implementation, designed to expose all 15 core operations of the Microsoft To Do API via n8n. It enables creation, retrieval, update, and deletion of To Do tasks, lists, and linked resources through an MCP (Microsoft Communication Protocol) webhook trigger.

The workflow logically divides into three main functional blocks, each handling one entity type within Microsoft To Do:

- **1.1 Linked Resources Management:** Operations on linked resources (create, get one, get many, update, delete)
- **1.2 Lists Management:** Operations on task lists (create, get one, get many, update, delete)
- **1.3 Tasks Management:** Operations on tasks (create, get one, get many, update, delete)

Each operation node is linked to a central MCP trigger node that receives incoming requests and routes to the corresponding Microsoft To Do Tool node for execution.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Linked Resources Management

**Overview:**  
This block handles all operations related to linked resources, which are external resources linked to tasks or lists in Microsoft To Do. It supports creating, retrieving (single and multiple), updating, and deleting linked resources.

**Nodes Involved:**  
- Microsoft To Do Tool MCP Server (trigger)  
- Create a linked resource  
- Get a linked resource  
- Get many linked resources  
- Update a linked resource  
- Delete a linked resource  
- Sticky Note 1 (empty content)

**Node Details:**

- **Microsoft To Do Tool MCP Server**  
  - Type: MCP Trigger (langchain.mcpTrigger)  
  - Role: Receives incoming webhook requests and triggers workflow execution.  
  - Configuration: Uses a fixed webhook ID for MCP requests.  
  - Inputs: External HTTP requests.  
  - Outputs: Routes to all Microsoft To Do Tool nodes via ai_tool connections.  
  - Edge Cases: Webhook failures, malformed requests, authentication errors with Microsoft API.  
  - Sub-workflow: None.

- **Create a linked resource**  
  - Type: Microsoft To Do Tool  
  - Role: Creates a new linked resource in Microsoft To Do.  
  - Configuration: Uses Microsoft To Do Tool node with "Create a linked resource" operation selected.  
  - Inputs: Trigger node outputs with data specifying resource details.  
  - Outputs: Created resource data.  
  - Edge Cases: API auth failure, invalid resource data, network timeout.

- **Get a linked resource**  
  - Type: Microsoft To Do Tool  
  - Role: Retrieves details of a single linked resource by ID.  
  - Configuration: Operation set to "Get a linked resource".  
  - Inputs: Trigger node data containing resource ID.  
  - Outputs: Resource detail JSON.  
  - Edge Cases: Resource not found, permission errors.

- **Get many linked resources**  
  - Type: Microsoft To Do Tool  
  - Role: Retrieves multiple linked resources, possibly with pagination or filters.  
  - Configuration: Operation "Get many linked resources".  
  - Inputs: Trigger with query parameters.  
  - Outputs: Array of linked resources.  
  - Edge Cases: Large data sets, API rate limits.

- **Update a linked resource**  
  - Type: Microsoft To Do Tool  
  - Role: Updates properties of an existing linked resource.  
  - Configuration: Operation "Update a linked resource".  
  - Inputs: Trigger with resource ID and updated fields.  
  - Outputs: Updated resource data.  
  - Edge Cases: Conflicts, invalid update data.

- **Delete a linked resource**  
  - Type: Microsoft To Do Tool  
  - Role: Deletes a linked resource by ID.  
  - Configuration: Operation "Delete a linked resource".  
  - Inputs: Trigger with resource ID.  
  - Outputs: Success/failure response.  
  - Edge Cases: Resource not found, permission denied.

- **Sticky Note 1**  
  - Type: Sticky Note  
  - Role: Placeholder, no content.

---

#### 2.2 Lists Management

**Overview:**  
Manages task lists within Microsoft To Do. Supports CRUD operations on lists including creation, retrieval (single and multiple), updating, and deletion.

**Nodes Involved:**  
- Microsoft To Do Tool MCP Server (trigger)  
- Create a list  
- Get a list  
- Get many lists  
- Update a list  
- Delete a list  
- Sticky Note 2 (empty content)

**Node Details:**

- **Create a list**  
  - Type: Microsoft To Do Tool  
  - Role: Creates a new task list.  
  - Configuration: Operation "Create a list".  
  - Inputs: Data specifying new list details.  
  - Outputs: Created list data.

- **Get a list**  
  - Type: Microsoft To Do Tool  
  - Role: Retrieves a specific list by ID.  
  - Configuration: Operation "Get a list".  
  - Inputs: List ID from trigger.  
  - Outputs: List details.

- **Get many lists**  
  - Type: Microsoft To Do Tool  
  - Role: Retrieves multiple lists.  
  - Configuration: Operation "Get many lists".  
  - Inputs: Possibly pagination or filter parameters.  
  - Outputs: Array of lists.

- **Update a list**  
  - Type: Microsoft To Do Tool  
  - Role: Updates an existing list's properties.  
  - Configuration: Operation "Update a list".  
  - Inputs: List ID and updated fields.  
  - Outputs: Updated list data.

- **Delete a list**  
  - Type: Microsoft To Do Tool  
  - Role: Deletes a list by ID.  
  - Configuration: Operation "Delete a list".  
  - Inputs: List ID.  
  - Outputs: Success/failure confirmation.

- **Sticky Note 2**  
  - Type: Sticky Note  
  - Role: Placeholder, no content.

---

#### 2.3 Tasks Management

**Overview:**  
Covers all task-specific operations in Microsoft To Do including creating, retrieving (single and multiple), updating, and deleting tasks.

**Nodes Involved:**  
- Microsoft To Do Tool MCP Server (trigger)  
- Create a task  
- Get a task  
- Get many tasks  
- Update a task  
- Delete a task  
- Sticky Note 3 (empty content)

**Node Details:**

- **Create a task**  
  - Type: Microsoft To Do Tool  
  - Role: Creates a new task in a specified list.  
  - Configuration: Operation "Create a task".  
  - Inputs: Task details including list ID.  
  - Outputs: Created task data.

- **Get a task**  
  - Type: Microsoft To Do Tool  
  - Role: Retrieves task details by task ID.  
  - Configuration: Operation "Get a task".  
  - Inputs: Task ID.  
  - Outputs: Task data.

- **Get many tasks**  
  - Type: Microsoft To Do Tool  
  - Role: Retrieves multiple tasks from a list.  
  - Configuration: Operation "Get many tasks".  
  - Inputs: List ID, optional filters.  
  - Outputs: Array of tasks.

- **Update a task**  
  - Type: Microsoft To Do Tool  
  - Role: Updates task properties.  
  - Configuration: Operation "Update a task".  
  - Inputs: Task ID and updated properties.  
  - Outputs: Updated task data.

- **Delete a task**  
  - Type: Microsoft To Do Tool  
  - Role: Deletes a task by ID.  
  - Configuration: Operation "Delete a task".  
  - Inputs: Task ID.  
  - Outputs: Confirmation of deletion.

- **Sticky Note 3**  
  - Type: Sticky Note  
  - Role: Placeholder, no content.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                      | Input Node(s)              | Output Node(s)               | Sticky Note |
|-----------------------------|-----------------------------|------------------------------------|----------------------------|-----------------------------|-------------|
| Workflow Overview 0          | Sticky Note                 | Workflow title/overview placeholder|                            |                             |             |
| Microsoft To Do Tool MCP Server | MCP Trigger (langchain.mcpTrigger) | Central webhook trigger for MCP requests | External HTTP requests      | All Microsoft To Do Tool nodes |             |
| Create a linked resource     | Microsoft To Do Tool        | Create linked resource              | MCP Trigger                |                             |             |
| Delete a linked resource     | Microsoft To Do Tool        | Delete linked resource              | MCP Trigger                |                             |             |
| Get a linked resource        | Microsoft To Do Tool        | Get single linked resource          | MCP Trigger                |                             |             |
| Get many linked resources    | Microsoft To Do Tool        | Get multiple linked resources       | MCP Trigger                |                             |             |
| Update a linked resource     | Microsoft To Do Tool        | Update linked resource              | MCP Trigger                |                             |             |
| Sticky Note 1               | Sticky Note                 | Placeholder for linked resources    |                            |                             |             |
| Create a list               | Microsoft To Do Tool        | Create task list                    | MCP Trigger                |                             |             |
| Delete a list               | Microsoft To Do Tool        | Delete task list                    | MCP Trigger                |                             |             |
| Get a list                 | Microsoft To Do Tool        | Get single task list                | MCP Trigger                |                             |             |
| Get many lists             | Microsoft To Do Tool        | Get multiple task lists             | MCP Trigger                |                             |             |
| Update a list               | Microsoft To Do Tool        | Update task list                    | MCP Trigger                |                             |             |
| Sticky Note 2               | Sticky Note                 | Placeholder for lists               |                            |                             |             |
| Create a task              | Microsoft To Do Tool        | Create task                        | MCP Trigger                |                             |             |
| Delete a task              | Microsoft To Do Tool        | Delete task                        | MCP Trigger                |                             |             |
| Get a task                | Microsoft To Do Tool        | Get single task                    | MCP Trigger                |                             |             |
| Get many tasks            | Microsoft To Do Tool        | Get multiple tasks                 | MCP Trigger                |                             |             |
| Update a task              | Microsoft To Do Tool        | Update task                       | MCP Trigger                |                             |             |
| Sticky Note 3               | Sticky Note                 | Placeholder for tasks              |                            |                             |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**  
   - Add the node **Microsoft To Do Tool MCP Server** of type `langchain.mcpTrigger`.  
   - Configure the webhook ID or leave default to generate one. This node listens for incoming MCP requests.  
   - No parameters needed beyond webhook setup. Credential configuration is not required here.

2. **Set up Linked Resources Nodes**  
   - Add five **Microsoft To Do Tool** nodes with the following operations respectively:  
     - "Create a linked resource"  
     - "Get a linked resource"  
     - "Get many linked resources"  
     - "Update a linked resource"  
     - "Delete a linked resource"  
   - For each node:  
     - Select Microsoft To Do Tool node type.  
     - Under Operations, pick the corresponding linked resource operation.  
     - Configure any necessary input parameters (resource ID, data) via expressions or input data.  
     - Connect each node’s input from the MCP Trigger via the ai_tool output connection.  
     - Configure Microsoft To Do credentials (OAuth2) with proper permissions for linked resource operations.

3. **Set up Lists Nodes**  
   - Add five **Microsoft To Do Tool** nodes for list operations:  
     - "Create a list"  
     - "Get a list"  
     - "Get many lists"  
     - "Update a list"  
     - "Delete a list"  
   - Configure each node similarly to step 2, selecting the appropriate list operation.  
   - Connect inputs from MCP Trigger node.  
   - Ensure Microsoft To Do credentials cover list management scopes.

4. **Set up Tasks Nodes**  
   - Add five **Microsoft To Do Tool** nodes for task operations:  
     - "Create a task"  
     - "Get a task"  
     - "Get many tasks"  
     - "Update a task"  
     - "Delete a task"  
   - Configure each node with the respective operation.  
   - Connect from MCP Trigger node.  
   - Use Microsoft To Do credentials with task management permissions.

5. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes as visual placeholders for each block:  
     - One near linked resources nodes  
     - One near lists nodes  
     - One near tasks nodes  
   - Content can be empty or descriptive as needed.

6. **Credential Setup**  
   - Create Microsoft To Do OAuth2 credentials in n8n:  
     - Client ID and Secret from Azure Portal.  
     - Scopes including `Tasks.ReadWrite`, `Tasks.ReadWrite.Shared`, etc.  
   - Assign these credentials to all Microsoft To Do Tool nodes.

7. **Testing and Validation**  
   - Deploy the workflow.  
   - Use the MCP webhook URL to trigger operations.  
   - Validate input data formats and response correctness.  
   - Handle error cases: invalid IDs, permissions, network issues.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                            |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This workflow implements all 15 Microsoft To Do operations available in the n8n Microsoft To Do Tool node. | General project info                                        |
| Microsoft To Do Tool detailed docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.microsoftToDoTool/ | Official n8n node documentation                             |
| MCP Trigger usage details: https://docs.n8n.io/integrations/trigger-nodes/mcpTrigger/           | MCP webhook trigger documentation                           |
| Ensure Microsoft Azure app registration includes correct API permissions for Tasks.ReadWrite scopes. | Microsoft Azure portal                                      |
| Use OAuth2 credentials with refresh token enabled to avoid authorization failures over time.    | Authentication best practices                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.