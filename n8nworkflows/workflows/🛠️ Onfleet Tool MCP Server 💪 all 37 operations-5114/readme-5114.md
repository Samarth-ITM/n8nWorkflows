🛠️ Onfleet Tool MCP Server 💪 all 37 operations

https://n8nworkflows.xyz/workflows/----onfleet-tool-mcp-server----all-37-operations-5114


# 🛠️ Onfleet Tool MCP Server 💪 all 37 operations

### 1. Workflow Overview

This workflow serves as a comprehensive MCP (Multi-Channel Platform) server for Onfleet, enabling 37 different Onfleet API operations through a single entry point. It is designed to facilitate a wide range of administrative and operational tasks for Onfleet users within an automated environment. The core logic is organized into the following functional blocks:

- **1.1 Trigger Reception:** Captures incoming requests via a specialized MCP trigger node.
- **1.2 Admin Management:** Supports creation, retrieval, update, and deletion of admin users.
- **1.3 Task Operations:** Covers task creation, updating, cloning, completing, deleting, and bulk operations.
- **1.4 Destination Management:** Handles creation and retrieval of destinations.
- **1.5 Hub Management:** Facilitates creation, updating, and listing of hubs.
- **1.6 Organization & Delegatee Details:** Fetches organizational info and delegatee details.
- **1.7 Recipient Management:** Manages recipients (create, get, update).
- **1.8 Team Management:** Includes team creation, retrieval, update, deletion, auto-dispatch, and time estimates.
- **1.9 Worker Management:** Manages workers including creation, retrieval, update, deletion, and schedule retrieval.
- **1.10 Container Management:** Retrieves container information.

Each block is composed of one or more Onfleet Tool nodes that perform specific API operations, all triggered through a single MCP trigger node.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger Reception

- **Overview:**  
  This block contains the single entry point for the workflow. It listens for incoming HTTP requests or events routed via the MCP trigger node. All subsequent Onfleet Tool nodes receive input from this node.

- **Nodes Involved:**  
  - Onfleet Tool MCP Server

- **Node Details:**  
  - **Node Name:** Onfleet Tool MCP Server  
  - **Type:** MCP Trigger (from Langchain n8n nodes)  
  - **Configuration:** No additional parameters; it acts as a webhook listener with a unique webhook ID.  
  - **Expressions/Variables:** None directly; triggers subsequent nodes through the `ai_tool` output.  
  - **Input:** External webhook/event  
  - **Output:** Routes to all other Onfleet Tool nodes based on requested operation.  
  - **Version Requirements:** Requires n8n version supporting MCP trigger node and Langchain integration.  
  - **Potential Failures:**  
    - Webhook ID conflicts or invalid webhook setup  
    - Network timeouts or connection errors  
    - Incoming request format errors (malformed or missing data)  
  - **Sub-workflow:** None

---

#### 1.2 Admin Management

- **Overview:**  
  Handles all admin-related operations: creation, deletion, retrieval (multiple), and update of admin users in Onfleet.

- **Nodes Involved:**  
  - Create an admin  
  - Delete an admin  
  - Get many admins  
  - Update an admin  

- **Node Details:**  
  Each node is an Onfleet Tool node configured for a specific admin API endpoint.

  - **Create an admin**  
    - Type: Onfleet Tool (API operation)  
    - Configuration: Set to "Create Admin" operation, expects payload with admin details.  
    - Input: Triggered from MCP Server node.  
    - Output: Admin creation response.  
    - Failure Modes: Validation errors, auth errors, API limits.

  - **Delete an admin**  
    - Type: Onfleet Tool  
    - Configuration: "Delete Admin" operation, requires admin ID parameter.  
    - Failure Modes: Nonexistent admin ID, permission errors.

  - **Get many admins**  
    - Type: Onfleet Tool  
    - Configuration: "Get Many Admins" operation, retrieves list of admins.  
    - Failure Modes: Network issues, empty results.

  - **Update an admin**  
    - Type: Onfleet Tool  
    - Configuration: "Update Admin" operation, requires admin ID and update fields.  
    - Failure Modes: Invalid fields, permission denied.

---

#### 1.3 Task Operations

- **Overview:**  
  Supports full task lifecycle management: add tasks, get task(s), update, clone, complete, and delete tasks.

- **Nodes Involved:**  
  - Add tasks  
  - Get a task  
  - Get many tasks  
  - Update tasks  
  - Create a task  
  - Clone a task  
  - Complete a task  
  - Delete a task  
  - Update a task  

- **Node Details:**  
  Each node corresponds to an Onfleet API task endpoint.

  - Example:  
    - **Add tasks**  
      - Adds multiple tasks in batch.  
      - Input expects array of task objects.  
      - Failure: payload format errors, API throttling.

    - **Clone a task**  
      - Clones an existing task by ID.  
      - Input: task ID.  
      - Failure: invalid task ID, cloning conflicts.

    - **Complete a task**  
      - Marks a task as complete.  
      - Input: task ID and optional completion details.  
      - Failure: task already completed or invalid state.

---

#### 1.4 Destination Management

- **Overview:**  
  Manages destinations by creating new ones or retrieving existing destinations by ID.

- **Nodes Involved:**  
  - Create a destination  
  - Get a destination  

- **Node Details:**  
  - **Create a destination:** Expects destination details such as address, notes.  
  - **Get a destination:** Requires destination ID to retrieve details.  
  - Failure modes include invalid address data, missing destination ID.

---

#### 1.5 Hub Management

- **Overview:**  
  Enables creation, retrieval (multiple), and update of hubs used for organizing delivery operations.

- **Nodes Involved:**  
  - Create a hub  
  - Get many hubs  
  - Update a hub  

- **Node Details:**  
  - Input parameters include hub name, location, and other metadata.  
  - Failure modes: invalid hub ID, permission errors, API limitations.

---

#### 1.6 Organization & Delegatee Details

- **Overview:**  
  Retrieves details about the current organization and delegatees.

- **Nodes Involved:**  
  - Get my organization  
  - Get a delegatee's details  

- **Node Details:**  
  - No input parameters required for organization details.  
  - Delegatee details require delegatee ID.  
  - Failures may include permission or invalid ID.

---

#### 1.7 Recipient Management

- **Overview:**  
  Manages recipients: creating new recipients, retrieving recipient details, and updating recipient information.

- **Nodes Involved:**  
  - Create a recipient  
  - Get a recipient  
  - Update a recipient  

- **Node Details:**  
  - Inputs include recipient name, contact info, and address.  
  - Failures: invalid recipient ID, missing mandatory fields.

---

#### 1.8 Team Management

- **Overview:**  
  Handles team-related operations including creation, retrieval (single and multiple), update, deletion, auto-dispatch, and time estimates.

- **Nodes Involved:**  
  - Auto-dispatch a team  
  - Create a team  
  - Delete a team  
  - Get a team  
  - Get many teams  
  - Get time estimates for a team  
  - Update a team  

- **Node Details:**  
  - Operations require team IDs or team data structures.  
  - Auto-dispatch triggers automatic task assignments.  
  - Time estimates provide expected delivery times for teams.  
  - Failures include invalid team ID, dispatch conflicts, authorization errors.

---

#### 1.9 Worker Management

- **Overview:**  
  Comprehensive worker management: create, delete, get, list, update workers, and get their schedules.

- **Nodes Involved:**  
  - Create a worker  
  - Delete a worker  
  - Get a worker  
  - Get many workers  
  - Get the schedule for a worker  
  - Update a worker  

- **Node Details:**  
  - Inputs include worker personal and schedule data.  
  - Schedule retrieval requires worker ID and optionally date ranges.  
  - Failures: invalid worker ID, malformed schedule queries, permission errors.

---

#### 1.10 Container Management

- **Overview:**  
  Retrieves container details from Onfleet.

- **Nodes Involved:**  
  - Get a container  

- **Node Details:**  
  - Requires container ID.  
  - Failures: invalid ID, API errors.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                 | Input Node(s)           | Output Node(s)                 | Sticky Note                          |
|----------------------------|----------------------------|--------------------------------|------------------------|-------------------------------|------------------------------------|
| Workflow Overview 0        | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Onfleet Tool MCP Server    | MCP Trigger                | Main entry point and trigger   | External webhook       | All Onfleet Tool nodes         |                                    |
| Create an admin            | Onfleet Tool               | Create admin user              | Onfleet Tool MCP Server|                               |                                    |
| Delete an admin            | Onfleet Tool               | Delete admin user              | Onfleet Tool MCP Server|                               |                                    |
| Get many admins            | Onfleet Tool               | Retrieve multiple admins       | Onfleet Tool MCP Server|                               |                                    |
| Update an admin            | Onfleet Tool               | Update admin details           | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 1              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Add tasks                  | Onfleet Tool               | Add multiple tasks             | Onfleet Tool MCP Server|                               |                                    |
| Get a container            | Onfleet Tool               | Retrieve container             | Onfleet Tool MCP Server|                               |                                    |
| Update tasks               | Onfleet Tool               | Update multiple tasks          | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 2              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Create a destination       | Onfleet Tool               | Create destination             | Onfleet Tool MCP Server|                               |                                    |
| Get a destination          | Onfleet Tool               | Retrieve destination           | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 3              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Create a hub               | Onfleet Tool               | Create hub                    | Onfleet Tool MCP Server|                               |                                    |
| Get many hubs              | Onfleet Tool               | Retrieve multiple hubs         | Onfleet Tool MCP Server|                               |                                    |
| Update a hub               | Onfleet Tool               | Update hub details             | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 4              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Get my organization        | Onfleet Tool               | Get organization info          | Onfleet Tool MCP Server|                               |                                    |
| Get a delegatee's details  | Onfleet Tool               | Delegatee information          | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 5              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Create a recipient         | Onfleet Tool               | Create recipient               | Onfleet Tool MCP Server|                               |                                    |
| Get a recipient            | Onfleet Tool               | Retrieve recipient             | Onfleet Tool MCP Server|                               |                                    |
| Update a recipient         | Onfleet Tool               | Update recipient details       | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 6              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Clone a task               | Onfleet Tool               | Clone existing task            | Onfleet Tool MCP Server|                               |                                    |
| Complete a task            | Onfleet Tool               | Mark task complete             | Onfleet Tool MCP Server|                               |                                    |
| Create a task              | Onfleet Tool               | Create new task                | Onfleet Tool MCP Server|                               |                                    |
| Delete a task              | Onfleet Tool               | Delete task                   | Onfleet Tool MCP Server|                               |                                    |
| Get a task                 | Onfleet Tool               | Retrieve single task           | Onfleet Tool MCP Server|                               |                                    |
| Get many tasks             | Onfleet Tool               | Retrieve multiple tasks        | Onfleet Tool MCP Server|                               |                                    |
| Update a task              | Onfleet Tool               | Update single task             | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 7              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Auto-dispatch a team       | Onfleet Tool               | Trigger automatic dispatch     | Onfleet Tool MCP Server|                               |                                    |
| Create a team              | Onfleet Tool               | Create new team                | Onfleet Tool MCP Server|                               |                                    |
| Delete a team              | Onfleet Tool               | Delete team                   | Onfleet Tool MCP Server|                               |                                    |
| Get a team                 | Onfleet Tool               | Retrieve single team           | Onfleet Tool MCP Server|                               |                                    |
| Get many teams             | Onfleet Tool               | Retrieve multiple teams        | Onfleet Tool MCP Server|                               |                                    |
| Get time estimates for a team | Onfleet Tool           | Get delivery time estimates    | Onfleet Tool MCP Server|                               |                                    |
| Update a team              | Onfleet Tool               | Update team details            | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 8              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |
| Create a worker            | Onfleet Tool               | Create worker                 | Onfleet Tool MCP Server|                               |                                    |
| Delete a worker            | Onfleet Tool               | Delete worker                 | Onfleet Tool MCP Server|                               |                                    |
| Get a worker               | Onfleet Tool               | Retrieve worker               | Onfleet Tool MCP Server|                               |                                    |
| Get many workers           | Onfleet Tool               | Retrieve multiple workers      | Onfleet Tool MCP Server|                               |                                    |
| Get the schedule for a worker | Onfleet Tool           | Retrieve worker schedule       | Onfleet Tool MCP Server|                               |                                    |
| Update a worker            | Onfleet Tool               | Update worker details          | Onfleet Tool MCP Server|                               |                                    |
| Sticky Note 9              | Sticky Note                | Documentation placeholder      |                        |                               |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Configure the webhook ID or leave default for auto-generation.  
   - This node will act as the main entry point receiving external triggers.

2. **Add Onfleet Tool Nodes for Each Operation**  
   For each API operation below, add an `Onfleet Tool` node configured appropriately:

   - **Admin Management:**  
     - Create an admin: Configure for "Create Admin" operation.  
     - Delete an admin: Configure for "Delete Admin" with admin ID parameter.  
     - Get many admins: Configure for "Get Many Admins".  
     - Update an admin: Configure for "Update Admin" with admin ID and update fields.

   - **Task Operations:**  
     - Add tasks: Configure for "Add Tasks" batch operation.  
     - Get a task: Configure for "Get Task" with task ID.  
     - Get many tasks: Configure for "Get Many Tasks".  
     - Update tasks: Configure for "Update Tasks".  
     - Create a task: Configure for "Create Task".  
     - Clone a task: Configure for "Clone Task" with source task ID.  
     - Complete a task: Configure for "Complete Task".  
     - Delete a task: Configure for "Delete Task".  
     - Update a task: Configure for "Update Task".

   - **Destination Management:**  
     - Create a destination: Configure for "Create Destination".  
     - Get a destination: Configure for "Get Destination".

   - **Hub Management:**  
     - Create a hub: Configure for "Create Hub".  
     - Get many hubs: Configure for "Get Many Hubs".  
     - Update a hub: Configure for "Update Hub".

   - **Organization & Delegatee:**  
     - Get my organization: Configure for "Get Organization".  
     - Get a delegatee's details: Configure for "Get Delegatee Details".

   - **Recipient Management:**  
     - Create a recipient: Configure for "Create Recipient".  
     - Get a recipient: Configure for "Get Recipient".  
     - Update a recipient: Configure for "Update Recipient".

   - **Team Management:**  
     - Auto-dispatch a team: Configure for "Auto-dispatch Team".  
     - Create a team: Configure for "Create Team".  
     - Delete a team: Configure for "Delete Team".  
     - Get a team: Configure for "Get Team".  
     - Get many teams: Configure for "Get Many Teams".  
     - Get time estimates for a team: Configure for "Get Time Estimates".  
     - Update a team: Configure for "Update Team".

   - **Worker Management:**  
     - Create a worker: Configure for "Create Worker".  
     - Delete a worker: Configure for "Delete Worker".  
     - Get a worker: Configure for "Get Worker".  
     - Get many workers: Configure for "Get Many Workers".  
     - Get the schedule for a worker: Configure for "Get Worker Schedule".  
     - Update a worker: Configure for "Update Worker".

   - **Container Management:**  
     - Get a container: Configure for "Get Container".

3. **Connect Nodes**  
   - From the MCP Trigger node, connect its `ai_tool` output to each Onfleet Tool node's input.  
   - No direct chaining between Onfleet Tool nodes is necessary, as the MCP Trigger dispatches to all.

4. **Credentials Setup**  
   - Configure Onfleet API credentials in n8n credentials manager.  
   - Assign these credentials to each Onfleet Tool node.

5. **Parameter Setup**  
   - For each Onfleet Tool node, configure mandatory and optional parameters per the Onfleet API requirements (e.g., IDs for update/delete, data payloads for create/update).  
   - Use expressions or input parameters to dynamically populate these fields based on incoming request data.

6. **Sticky Notes (Optional)**  
   - Add sticky notes above node groups to document the logic or usage.

7. **Testing and Validation**  
   - Test the MCP webhook with various API operations to verify correct routing and execution.  
   - Handle error responses by adding error workflow branches or HTTP response nodes if necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow supports all 37 Onfleet operations via a single MCP trigger node for flexible API routing.   | Project design overview                             |
| Requires n8n with Langchain MCP trigger node installed and Onfleet API credentials properly configured.   | n8n environment requirements                         |
| Onfleet API documentation for reference: https://docs.onfleet.com/docs/api-reference                     | Official Onfleet API docs                            |
| Use of MCP trigger node enables multi-operation handling through one webhook endpoint, simplifying webhook management. | n8n Langchain MCP trigger functionality             |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow built with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.