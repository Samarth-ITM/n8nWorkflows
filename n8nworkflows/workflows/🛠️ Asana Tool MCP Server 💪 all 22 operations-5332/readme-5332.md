🛠️ Asana Tool MCP Server 💪 all 22 operations

https://n8nworkflows.xyz/workflows/----asana-tool-mcp-server----all-22-operations-5332


# 🛠️ Asana Tool MCP Server 💪 all 22 operations

### 1. Workflow Overview

This workflow, titled **"🛠️ Asana Tool MCP Server 💪 all 22 operations"**, serves as a comprehensive backend server handling all 22 supported operations of the Asana Tool node in n8n. It is designed to provide a unified API endpoint (via the MCP Trigger node) to execute any Asana operation dynamically based on incoming requests. The workflow encompasses all major Asana entities and their core CRUD and management operations, including projects, tasks, subtasks, comments, tags, and users.

**Target Use Cases:**  
- Centralized automation server to manage Asana workspace resources programmatically.  
- Backend flow to expose Asana operations through a single webhook/API for external systems or UI clients.  
- Automate and orchestrate complex Asana workflows by invoking any supported operation as needed.

**Logical Blocks:**  
- **1.1 Trigger Reception:** Entry point receiving requests and routing to operations.  
- **1.2 Project Management:** Nodes handling create, read, update, delete, and list projects.  
- **1.3 Subtask Management:** Nodes for creating and listing subtasks.  
- **1.4 Task Management:** Full set of task operations including create, delete, update, search, move, and get multiple tasks.  
- **1.5 Task Comment Management:** Adding and removing comments on tasks.  
- **1.6 Task Project and Tag Management:** Adding/removing tags and projects to/from tasks.  
- **1.7 User Management:** Retrieving single or multiple users from Asana.  
- **1.8 Documentation and Notes:** Sticky notes scattered to aid understanding and organization.

Each operation node is directly connected to the **"Asana Tool MCP Server"** MCP Trigger node, which acts as a smart router invoking the correct operation node based on the input request.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Reception

- **Overview:**  
  This block contains a single MCP Trigger node that serves as the workflow's entry point. It listens for incoming requests to trigger one of the 22 Asana operations.

- **Nodes Involved:**  
  - Asana Tool MCP Server (MCP Trigger)

- **Node Details:**  
  - Type: MCP Trigger (from `@n8n/n8n-nodes-langchain` package)  
  - Role: Listens for incoming webhook/API calls and routes data to appropriate nodes.  
  - Configuration: Uses a specific webhook ID to receive external requests. No parameters set inside the node itself since it dynamically handles multiple operations.  
  - Connections: Outputs to all Asana operation nodes through the connection named "ai_tool."  
  - Version-specific: Requires n8n version supporting MCP Trigger node (usually 1.95+).  
  - Edge Cases:  
    - Webhook authentication or security misconfiguration could allow unauthorized access.  
    - Network or timeout errors on webhook reception.  
    - Incoming payloads missing required parameters could cause routing failures.  
  - No sub-workflows invoked here.

---

#### 1.2 Project Management

- **Overview:**  
  Handles all CRUD and list operations related to Asana projects.

- **Nodes Involved:**  
  - Create a project  
  - Delete a project  
  - Get a project  
  - Get many projects  
  - Update a project  
  - Sticky Note 1 (Empty, presumably for documentation or grouping)

- **Node Details:**  

  **Create a project**  
  - Type: Asana Tool node  
  - Role: Creates a new project in Asana workspace.  
  - Configuration: Default empty parameters, expects input from MCP Trigger to supply project details.  
  - Connections: Input from MCP Trigger, no further outputs.  
  - Edge Cases: API permission errors, missing mandatory fields (e.g., workspace ID), API rate limits.

  **Delete a project**  
  - Type: Asana Tool node  
  - Role: Deletes a specified Asana project by ID.  
  - Edge Cases: Nonexistent project ID, insufficient permissions, API limits.

  **Get a project**  
  - Type: Asana Tool node  
  - Role: Retrieves details of a specific project by ID.  
  - Edge Cases: Invalid or missing ID, API errors.

  **Get many projects**  
  - Type: Asana Tool node  
  - Role: Lists multiple projects, possibly with filters.  
  - Edge Cases: Large result sets causing timeouts, pagination handling.

  **Update a project**  
  - Type: Asana Tool node  
  - Role: Updates project attributes such as name or notes.  
  - Edge Cases: Missing ID or update fields, permission errors.

  **Sticky Note 1**  
  - Empty content, positioned near project nodes, likely for future notes or grouping.

---

#### 1.3 Subtask Management

- **Overview:**  
  Nodes to create subtasks under existing tasks and to list subtasks.

- **Nodes Involved:**  
  - Create a subtask  
  - Get many subtasks  
  - Sticky Note 2 (Empty)

- **Node Details:**  

  **Create a subtask**  
  - Type: Asana Tool node  
  - Role: Creates a subtask under a specified parent task.  
  - Edge Cases: Invalid parent task ID, missing mandatory fields.

  **Get many subtasks**  
  - Type: Asana Tool node  
  - Role: Retrieves multiple subtasks under a task or project.  
  - Edge Cases: Pagination, large data sets.

  **Sticky Note 2**  
  - Empty content for grouping.

---

#### 1.4 Task Management

- **Overview:**  
  Comprehensive operations for tasks including create, delete, get single/multiple tasks, move, search, and update.

- **Nodes Involved:**  
  - Create a task  
  - Delete a task  
  - Get a task  
  - Get many tasks  
  - Move a task  
  - Search a task  
  - Update a task  
  - Sticky Note 3 (Empty)

- **Node Details:**  

  **Create a task**  
  - Type: Asana Tool node  
  - Role: Creates a new task in a project or workspace.  
  - Edge Cases: Missing required fields like project ID or workspace, permission issues.

  **Delete a task**  
  - Type: Asana Tool node  
  - Role: Deletes a specified task by ID.  
  - Edge Cases: Task not found, permission denied.

  **Get a task**  
  - Type: Asana Tool node  
  - Role: Retrieves task details by ID.  
  - Edge Cases: Invalid ID, API errors.

  **Get many tasks**  
  - Type: Asana Tool node  
  - Role: Lists tasks with optional filters.  
  - Edge Cases: Large data sets, pagination.

  **Move a task**  
  - Type: Asana Tool node  
  - Role: Moves a task between projects or sections.  
  - Edge Cases: Invalid target project or section, permission issues.

  **Search a task**  
  - Type: Asana Tool node  
  - Role: Searches tasks by criteria.  
  - Edge Cases: Complex queries causing timeouts.

  **Update a task**  
  - Type: Asana Tool node  
  - Role: Updates task fields.  
  - Edge Cases: Missing ID, invalid update fields.

  **Sticky Note 3**  
  - Empty content, positioned near task nodes.

---

#### 1.5 Task Comment Management

- **Overview:**  
  Nodes to add or remove comments on tasks.

- **Nodes Involved:**  
  - Add a task comment  
  - Remove a task comment  
  - Sticky Note 4 (Empty)

- **Node Details:**  

  **Add a task comment**  
  - Type: Asana Tool node  
  - Role: Adds a comment or story to a task.  
  - Edge Cases: Missing task ID or comment text, permission errors.

  **Remove a task comment**  
  - Type: Asana Tool node  
  - Role: Deletes a comment/story from a task.  
  - Edge Cases: Invalid comment ID, permission denied.

  **Sticky Note 4**  
  - Empty content.

---

#### 1.6 Task Project and Tag Management

- **Overview:**  
  Manage the association of projects and tags with tasks, enabling adding or removing.

- **Nodes Involved:**  
  - Add a task project  
  - Remove a task project  
  - Add a task tag  
  - Remove a task tag  
  - Sticky Note 5 (Empty near project management nodes)  
  - Sticky Note 6 (Empty near tag management nodes)

- **Node Details:**  

  **Add a task project**  
  - Type: Asana Tool node  
  - Role: Links a task to a project.  
  - Edge Cases: Invalid task or project ID, permission failures.

  **Remove a task project**  
  - Type: Asana Tool node  
  - Role: Removes task from a project.  
  - Edge Cases: Task not linked to project, invalid IDs.

  **Add a task tag**  
  - Type: Asana Tool node  
  - Role: Adds a tag to a task.  
  - Edge Cases: Invalid tag or task ID.

  **Remove a task tag**  
  - Type: Asana Tool node  
  - Role: Removes a tag from a task.  
  - Edge Cases: Tag not present, invalid IDs.

  **Sticky Note 5 and 6**  
  - Empty notes for grouping.

---

#### 1.7 User Management

- **Overview:**  
  Retrieve user details or list multiple users in the Asana workspace.

- **Nodes Involved:**  
  - Get a user  
  - Get many users  
  - Sticky Note 7 (Empty)

- **Node Details:**  

  **Get a user**  
  - Type: Asana Tool node  
  - Role: Retrieves information of a single user by ID.  
  - Edge Cases: Invalid user ID, permissions.

  **Get many users**  
  - Type: Asana Tool node  
  - Role: Lists multiple users in the workspace.  
  - Edge Cases: Large user lists, pagination.

  **Sticky Note 7**  
  - Empty.

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                        | Input Node(s)         | Output Node(s)       | Sticky Note |
|---------------------|-------------------------------|-------------------------------------|-----------------------|----------------------|-------------|
| Workflow Overview 0 | Sticky Note                   | Empty note                          |                       |                      |             |
| Asana Tool MCP Server | MCP Trigger                  | Entry point and router for all ops |                       | All Asana Tool nodes |             |
| Create a project    | Asana Tool                   | Create new project                   | Asana Tool MCP Server |                      |             |
| Delete a project    | Asana Tool                   | Delete project by ID                 | Asana Tool MCP Server |                      |             |
| Get a project       | Asana Tool                   | Retrieve project details             | Asana Tool MCP Server |                      |             |
| Get many projects   | Asana Tool                   | List multiple projects               | Asana Tool MCP Server |                      |             |
| Update a project    | Asana Tool                   | Update project attributes            | Asana Tool MCP Server |                      |             |
| Sticky Note 1       | Sticky Note                  | Empty note near project nodes       |                       |                      |             |
| Create a subtask    | Asana Tool                   | Create a subtask                    | Asana Tool MCP Server |                      |             |
| Get many subtasks   | Asana Tool                   | List subtasks                      | Asana Tool MCP Server |                      |             |
| Sticky Note 2       | Sticky Note                  | Empty note near subtask nodes      |                       |                      |             |
| Create a task       | Asana Tool                   | Create a new task                   | Asana Tool MCP Server |                      |             |
| Delete a task       | Asana Tool                   | Delete task by ID                   | Asana Tool MCP Server |                      |             |
| Get a task          | Asana Tool                   | Retrieve task details               | Asana Tool MCP Server |                      |             |
| Get many tasks      | Asana Tool                   | List multiple tasks                 | Asana Tool MCP Server |                      |             |
| Move a task         | Asana Tool                   | Move task between projects/sections| Asana Tool MCP Server |                      |             |
| Search a task       | Asana Tool                   | Search tasks by criteria            | Asana Tool MCP Server |                      |             |
| Update a task       | Asana Tool                   | Update task attributes             | Asana Tool MCP Server |                      |             |
| Sticky Note 3       | Sticky Note                  | Empty note near task nodes          |                       |                      |             |
| Add a task comment  | Asana Tool                   | Add comment to task                 | Asana Tool MCP Server |                      |             |
| Remove a task comment| Asana Tool                  | Remove comment from task            | Asana Tool MCP Server |                      |             |
| Sticky Note 4       | Sticky Note                  | Empty note near comment nodes       |                       |                      |             |
| Add a task project  | Asana Tool                   | Add project to task                 | Asana Tool MCP Server |                      |             |
| Remove a task project| Asana Tool                  | Remove project from task            | Asana Tool MCP Server |                      |             |
| Sticky Note 5       | Sticky Note                  | Empty note near project assoc nodes |                       |                      |             |
| Add a task tag      | Asana Tool                   | Add tag to task                    | Asana Tool MCP Server |                      |             |
| Remove a task tag   | Asana Tool                   | Remove tag from task                | Asana Tool MCP Server |                      |             |
| Sticky Note 6       | Sticky Note                  | Empty note near tag nodes           |                       |                      |             |
| Get a user          | Asana Tool                   | Get user details                   | Asana Tool MCP Server |                      |             |
| Get many users      | Asana Tool                   | List multiple users                | Asana Tool MCP Server |                      |             |
| Sticky Note 7       | Sticky Note                  | Empty note near user nodes          |                       |                      |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger node:**  
   - Name: `Asana Tool MCP Server`  
   - Type: MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
   - Configure webhook ID (auto-generated or custom) to receive API calls.  
   - No parameters needed. This node is the entry point that will dispatch requests to downstream nodes.

2. **Create all Asana Tool operation nodes:**  
   For each operation below, create an **Asana Tool** node and configure it as follows:

   - **Credentials:** Use a valid Asana API credential with appropriate workspace access.  
   - **Operation:** Select the exact operation matching the node name (e.g., "Create a project", "Delete a task", etc.).  
   - **Parameters:** Leave empty or configure to accept incoming data dynamically from the MCP Trigger node.  
   - **Name:** Use the exact node name as in the original workflow for clarity.

   The nodes to create:  
   - Create a project  
   - Delete a project  
   - Get a project  
   - Get many projects  
   - Update a project  
   - Create a subtask  
   - Get many subtasks  
   - Create a task  
   - Delete a task  
   - Get a task  
   - Get many tasks  
   - Move a task  
   - Search a task  
   - Update a task  
   - Add a task comment  
   - Remove a task comment  
   - Add a task project  
   - Remove a task project  
   - Add a task tag  
   - Remove a task tag  
   - Get a user  
   - Get many users

3. **Connect the MCP Trigger node's output ("ai_tool") to all Asana Tool nodes' inputs:**  
   - This allows the MCP Trigger to route incoming requests to the correct Asana operation node based on the request context.

4. **(Optional) Add Sticky Notes for organization:**  
   - Place sticky notes near grouped nodes for visual clarity: projects, subtasks, tasks, comments, tags, users.  
   - Content can be added for documentation or left empty as placeholders.

5. **Set workflow timezone to America/New_York:**  
   - This matches the original workflow's timezone setting.

6. **Save and activate the workflow:**  
   - Ensure all nodes have valid Asana credentials.  
   - Test by sending requests to the MCP Trigger webhook URL specifying the desired operation and parameters.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link              |
|------------------------------------------------------------------------------|-----------------------------|
| Workflow serves as a full backend API server for all Asana Tool operations.  | Workflow purpose description |
| MCP Trigger node requires n8n 1.95+ and proper credential setup for Asana API. | n8n version requirement     |
| Ensure Asana API credentials have necessary scopes for all operations (projects, tasks, comments, users). | Credential setup guide      |
| For optimal performance, handle pagination in "Get many" operations externally or via workflow enhancements. | API pagination considerations |
| Empty sticky notes are placeholders to organize nodes visually; consider adding descriptive content for maintainability. | Workflow documentation tip |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow and strictly adheres to content policies. It contains no illegal, offensive, or protected elements. All manipulated data is legal and public.