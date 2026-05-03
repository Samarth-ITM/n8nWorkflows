Sync Notion to Clockify including Clients Projects and Tasks

https://n8nworkflows.xyz/workflows/sync-notion-to-clockify-including-clients-projects-and-tasks-3177


# Sync Notion to Clockify including Clients Projects and Tasks

### 1. Workflow Overview

This workflow synchronizes three hierarchical entities—Clients, Projects, and Tasks—from Notion databases to the Clockify time-tracking platform. Its primary purpose is to ensure that Clockify accurately reflects the current state of clients, projects, and tasks managed within Notion, enabling time tracking to be linked effectively.

The workflow is functional for:
- Regular daily synchronization of Clients, Projects, and Tasks.
- On-demand sync via webhook trigger (e.g., using a Notion button).

It processes data sequentially due to hierarchical dependencies:
- Block 1: Initialization (Triggers and Global Settings).
- Block 2: Synchronize Clients.
- Block 3: Synchronize Projects.
- Block 4: Synchronize Tasks.

Each block first retrieves active entities from both Notion and Clockify, maps and compares data sets, handles creates, updates, and deletions, and finally updates Notion with Clockify IDs for reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization: Triggers and Globals

**Overview:**  
Prepares the workflow execution context by setting up the workspace identification and triggers the synchronization process either on schedule or via webhook.

**Nodes Involved:**  
- Webhook  
- Schedule Trigger  
- Get first workspace ID  
- Globals  
- No Operation (No Operation node simply passes data along)

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Accepts external HTTP POST calls to start the workflow manually.  
  - Config: Path set for a webhook ID; POST method.  
  - Outputs to "Get first workspace ID".  
  - Edge Cases: Webhook unreachable if URL invalid; authorization not required by default.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Runs the workflow automatically daily at 4 AM.  
  - Config: Interval trigger set at 4:00 o’clock.  
  - Outputs to "Get first workspace ID".

- **Get first workspace ID**  
  - Type: Clockify node  
  - Role: Fetches the first available workspace to set as the context.  
  - Config: resource=workspace, limit=1, uses Clockify API credentials.  
  - Output flows into "Globals".  
  - Failure: API timeout, auth errors.

- **Globals**  
  - Type: Set node  
  - Role: Configures the "workspace_id" global variable dynamically with the first workspace ID.  
  - Config: Assigns workspace_id from the previous node’s output JSON field "id".  
  - Outputs to "No Operation".

- **No Operation (No Operation)**  
  - Type: NoOp node (does nothing)  
  - Role: Placeholder to continue workflow flow to client synchronization.  
  - No parameters.

---

#### 2.2 Synchronize Clients

**Overview:**  
Fetches active clients from both Notion and Clockify, compares their data sets by Clockify client ID, and applies updates such as creating new clients, updating existing clients, or removing archived ones. Notion pages are updated with Clockify IDs as references.

**Nodes Involved:**  
- Get active Clients from Notion  
- Get active Clients from Clockify  
- Map values (for Notion clients)  
- Map values1 (for Clockify clients)  
- Compare Datasets  
- If unmapped in Notion  
- Create Client in Clockify  
- Store Clockify ID in Notion  
- Update Client in Clockify  
- Get archived Client from Notion  
- Set new values (for archived clients)  
- Remove Client from Clockify  
- Stop and Error  
- Structure output  
- Merge  
- Limit  
- No Operation, do nothing  
- No Operation

**Node Details:**

- **Get active Clients from Notion**  
  - Type: Notion node  
  - Role: Retrieves all clients not archived (with archive checkbox false).  
  - Config: Database ID set to Clients database; filter: Archive checkbox = false.  
  - Retries on failure, waits 5s between tries.

- **Get active Clients from Clockify**  
  - Type: Clockify node  
  - Role: Gets all non-archived clients in the workspace.  
  - Output mapped for comparison.

- **Map values** & **Map values1**  
  - Type: Set nodes  
  - Role: Normalize and map relevant client fields: name, archived status, and Clockify client ID.  
  - Notion source uses 'property_clockify_client_id', Clockify uses 'id'.

- **Compare Datasets**  
  - Type: Compare Datasets node  
  - Role: Compares Notion and Clockify client records by Clockify client ID to find created, updated, or deleted clients.

- **If unmapped in Notion**  
  - Type: If node  
  - Condition: Checks for clients that exist in Clockify but have no Clockify ID in Notion (i.e., new in Clockify).  
  - Flow: New clients in Clockify → create in Notion or update accordingly.

- **Create Client in Clockify**  
  - Type: Clockify node  
  - Role: Creates a new client in Clockify with the mapped name.

- **Store Clockify ID in Notion**  
  - Type: Notion node  
  - Role: Updates the Notion client page with Clockify ID returned after creation, ensuring ID reference is stored.

- **Update Client in Clockify**  
  - Type: Clockify node  
  - Role: Updates client attributes, specifically archival status and name.

- **Get archived Client from Notion**  
  - Type: Notion node  
  - Role: Checks if a client in Clockify was archived in Notion.

- **Set new values**  
  - Type: Set node  
  - Sets "archived" to true for clients archived in Notion.

- **Remove Client from Clockify**  
  - Type: Clockify node  
  - Role: Deletes client entry from Clockify if client is removed.

- **Stop and Error**  
  - Type: Stop and Error node  
  - Raises error when Client deletion fails or conflicts happen (e.g., client deleted again in Clockify).

- **Structure output**, **Merge**, **Limit**, **No Operation** nodes  
  - Nodes to control data flow, merge datasets, and avoid execution flooding.

**Edge Cases:**  
- ID mismatches causing create/update conflicts.  
- Archived client synchronization lag.  
- API limits or failures on delete or update operations.  
- Notion page update conflicts or permission errors.

---

#### 2.3 Synchronize Projects

**Overview:**  
Identical logic to Clients synchronization but applies it to "Projects." Projects are linked to Clients via Clockify client IDs. Closed or obsolete projects in Notion are archived in Clockify.

**Nodes Involved:**  
- Get active Projects from Notion  
- Get active Projects from Clockify  
- Map values2, Map values3  
- Compare Datasets1  
- If unmapped in Notion1  
- Create Project in Clockify (HTTP Request node)  
- Store Clockify ID in Notion1  
- Update Project in Clockify (HTTP Request node)  
- Get completed Project from Notion  
- Set new values1  
- Remove Project from Clockify  
- Stop and Error1  
- Structure output1  
- Merge1  
- Limit1  
- No Operation1, No Operation, do nothing2

**Node Details:**

- **Get active Projects from Notion**  
  - Filters exclude projects with "Done" or "Obsolete" status and require non-empty client relation.  
  - Authorization with Notion API.

- **Get active Projects from Clockify**  
  - Retrieves non-archived projects in the workspace.

- **Map values2** & **Map values3**  
  - Normalize project properties including name, archived status, Clockify project ID, and client linkage.

- **Compare Datasets1**  
  - Matches projects by Clockify project ID, detecting changes.

- **If unmapped in Notion1**  
  - Detects projects missing Clockify IDs, indicating new projects needing creation.

- **Create Project in Clockify** (HTTP Request node)  
  - Direct HTTP call to Clockify REST API for project creation including name, clientId, and archived state.

- **Store Clockify ID in Notion1**  
  - Updates Notion project with new Clockify project ID.

- **Update Project in Clockify** (HTTP Request node)  
  - PUT request updating archived flag, clientId, and name fields.

- **Remove Project from Clockify**  
  - Deletes project if archived or removed.

- **Stop and Error1**  
  - Errors out on failed deletion or update conflict.

**Edge Cases:**  
- Required Status options in Notion must include "Done" and "Obsolete" for filtering.  
- Race conditions when updating or removing projects already changed externally.  
- HTTP request failures, API rate limits, or permission errors.

---

#### 2.4 Synchronize Tasks

**Overview:**  
Manages synchronization of active tasks between Notion and Clockify. Tasks are associated with Clockify projects. Completed or obsolete tasks are archived accordingly.

**Nodes Involved:**  
- Get active Tasks from Notion  
- Get active Tasks from Clockify  
- Map values4, Map values5  
- Compare Datasets2  
- If unmapped in Notion2  
- Create Task in Clockify  
- Store Clockify ID in Notion2  
- Update Task in Clockify  
- Get completed Task from Notion  
- Set new values2  
- Remove Task from Clockify  
- Stop and Error2  
- Structure output2  
- Merge (main merge for all entities)  
- Limit2  
- No Operation2

**Node Details:**

- **Get active Tasks from Notion**  
  - Retrieves tasks excluding those with status "Done" or "Obsolete".  
  - Client rollup filtering ensures parent data relevance.

- **Get active Tasks from Clockify**  
  - Retrieves active tasks linked to projects in workspace.

- **Map values4**, **Map values5**  
  - Map task name, archived status, and clockify identifiers for tasks and projects.

- **Compare Datasets2**  
  - Detects differences between Notion and Clockify tasks by Clockify task ID.

- **If unmapped in Notion2**  
  - Detects tasks to create in Clockify.

- **Create Task in Clockify**  
  - Creates new task in Clockify under appropriate project.

- **Store Clockify ID in Notion2**  
  - Writes new Clockify task ID back to Notion.

- **Update Task in Clockify**  
  - Updates task name and status ("DONE" or "ACTIVE").

- **Remove Task from Clockify**  
  - Deletes task when removed or archived.

- **Stop and Error2**  
  - Raises error on failed deletion or update conflicts.

**Edge Cases:**  
- Projects must be synchronized in order for tasks referencing them to be valid.  
- Status field mapping from Notion status to Clockify task status must be aligned.  
- API limits, permission errors, and data consistency on concurrent edits.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                                  | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                         |
|-----------------------------|-------------------------|-------------------------------------------------|--------------------------------------|--------------------------------------|---------------------------------------------------------------------|
| Webhook                     | Webhook                 | Starts workflow manually                         | -                                    | Get first workspace ID                | Run manually via webhook for instant sync                           |
| Schedule Trigger            | Schedule Trigger         | Triggers daily sync at 4AM                       | -                                    | Get first workspace ID                | Runs automatically once daily                                       |
| Get first workspace ID      | Clockify                 | Gets first Clockify workspace ID                 | Webhook, Schedule Trigger              | Globals                              |                                                                     |
| Globals                    | Set                     | Stores workspace id as global parameter          | Get first workspace ID                | No Operation                        | Override default workspace ID if necessary                         |
| No Operation               | NoOp                    | Pass-through for workflow continuation           | Globals                              | Get active Clients from Notion, Get active Clients from Clockify |                                                                     |
| Get active Clients from Notion | Notion               | Fetches active clients filtered by archive false | No Operation                        | Map values                          | Select clients database                                              |
| Get active Clients from Clockify | Clockify           | Fetches active clients, non-archived             | No Operation                        | Map values1                         |                                                                     |
| Map values                 | Set                     | Normalize Notion client data                      | Get active Clients from Notion        | Compare Datasets                    |                                                                     |
| Map values1                | Set                     | Normalize Clockify client data                    | Get active Clients from Clockify      | Compare Datasets                    |                                                                     |
| Compare Datasets           | Compare Datasets         | Diff and merge clients by Clockify Client ID     | Map values, Map values1               | If unmapped in Notion, Structure output, Update Client in Clockify, Get archived Client from Notion |                                                                     |
| If unmapped in Notion      | If                      | Checks unmapped clients in Notion                 | Compare Datasets                     | Create Client in Clockify, Update Client in Clockify |                                                                     |
| Create Client in Clockify  | Clockify                 | Creates new client in Clockify                     | If unmapped in Notion                | Store Clockify ID in Notion          |                                                                     |
| Store Clockify ID in Notion | Notion                  | Updates Notion with Clockify client ID            | Create Client in Clockify, Update Client in Clockify | No Operation, Remove Client from Clockify |                                                                     |
| Update Client in Clockify  | Clockify                 | Updates existing Clockify client                  | Compare Datasets, If unmapped in Notion | Merge                              |                                                                     |
| Get archived Client from Notion | Notion               | Fetches archived clients to sync archive state    | Compare Datasets                    | Set new values                       |                                                                     |
| Set new values             | Set                     | Marks client as archived                           | Get archived Client from Notion      | Update Client in Clockify           |                                                                     |
| Remove Client from Clockify | Clockify                | Deletes client in Clockify                         | Store Clockify ID in Notion          | Stop and Error                      |                                                                     |
| Stop and Error             | Stop and Error           | Stops workflow on failures                         | Remove Client from Clockify          | -                                  |                                                                     |
| Structure output           | NoOp                    | Data flow management                               | Compare Datasets                    | Merge                              |                                                                     |
| Merge                     | Merge                   | Merges processed data streams                      | Structure output, other branches    | Limit                              | Synchronizes sequential block flow                                 |
| Limit                     | Limit                   | Controls execution concurrency                      | Merge                              | No Operation1                     |                                                                     |
| No Operation1              | NoOp                    | Pass-through for projects block                    | Limit                              | Get active Projects from Notion, Get active Projects from Clockify |                                                                     |
| Get active Projects from Notion | Notion              | Fetches active projects by status filter           | No Operation1                       | Map values2                       | Select projects database                                           |
| Get active Projects from Clockify | Clockify           | Fetches active projects in Clockify                | No Operation1                       | Map values3                       |                                                                     |
| Map values2                | Set                     | Normalize Notion project data                       | Get active Projects from Notion     | Compare Datasets1                 |                                                                     |
| Map values3                | Set                     | Normalize Clockify project data                     | Get active Projects from Clockify   | Compare Datasets1                 |                                                                     |
| Compare Datasets1          | Compare Datasets         | Diff projects by Clockify project ID               | Map values2, Map values3             | If unmapped in Notion1, Structure output1, Update Project in Clockify, Get completed Project from Notion |                                                                     |
| If unmapped in Notion1     | If                      | Checks projects unmapped in Notion                  | Compare Datasets1                   | Create Project in Clockify, Update Project in Clockify |                                                                     |
| Create Project in Clockify | HTTP Request             | Creates new project in Clockify                      | If unmapped in Notion1              | Store Clockify ID in Notion1       |                                                                     |
| Store Clockify ID in Notion1 | Notion                 | Updates Notion with new Project Clockify ID         | Create Project in Clockify          | No Operation, do nothing2          |                                                                     |
| Update Project in Clockify | HTTP Request             | Updates project info in Clockify                     | Compare Datasets1, If unmapped in Notion1 | Merge1                         |                                                                     |
| Get completed Project from Notion | Notion              | Gets completed projects to mark archive              | Compare Datasets1                   | Set new values1                   |                                                                     |
| Set new values1            | Set                     | Sets archived flag true for projects                 | Get completed Project from Notion   | Update Project in Clockify        |                                                                     |
| Remove Project from Clockify| Clockify                | Removes project from Clockify                         | Store Clockify ID in Notion1       | Stop and Error1                  |                                                                     |
| Stop and Error1            | Stop and Error           | Stops workflow when project update fails             | Remove Project from Clockify        | -                                |                                                                     |
| Structure output1          | NoOp                    | Manages project data flow                             | Compare Datasets1                 | Merge1                          |                                                                     |
| Merge1                    | Merge                   | Merge projects data streams                           | Structure output1, other branches | Limit1                         |                                                                     |
| Limit1                    | Limit                   | Controls execution for project block                  | Merge1                          | No Operation2                 |                                                                     |
| No Operation2              | NoOp                    | Pass-through for tasks block                           | Limit1                          | Get active Tasks from Notion, Get active Tasks from Clockify |                                                                     |
| Get active Tasks from Notion | Notion                | Gets active tasks filtered by status and clients      | No Operation2                   | Map values4                     | Select tasks database                                              |
| Get active Tasks from Clockify | Clockify             | Gets active tasks in workspace projects                | Get active Projects from Clockify1 | Map values5                    |                                                                     |
| Map values4                | Set                     | Normalize Notion task data                              | Get active Tasks from Notion     | Compare Datasets2               |                                                                     |
| Map values5                | Set                     | Normalize Clockify task data                            | Get active Tasks from Clockify  | Compare Datasets2               |                                                                     |
| Compare Datasets2          | Compare Datasets         | Compares tasks by Clockify task ID                      | Map values4, Map values5          | If unmapped in Notion2, Structure output2, Update Task in Clockify, Get completed Task from Notion |                                                                     |
| If unmapped in Notion2     | If                      | Detects tasks missing from Notion                        | Compare Datasets2                | Create Task in Clockify, Update Task in Clockify |                                                                     |
| Create Task in Clockify    | Clockify                 | Creates new task linked to project                       | If unmapped in Notion2           | Store Clockify ID in Notion2     |                                                                     |
| Store Clockify ID in Notion2| Notion                  | Updates Notion task page with Clockify task ID          | Create Task in Clockify          | Remove Task from Clockify (conditional) |                                                                     |
| Update Task in Clockify    | Clockify                 | Updates task name and status                             | Compare Datasets2, If unmapped in Notion2 | Merge                      |                                                                     |
| Remove Task from Clockify  | Clockify                 | Deletes task from Clockify                               | Store Clockify ID in Notion2     | Stop and Error2                 |                                                                     |
| Stop and Error2            | Stop and Error           | Stops execution on task deletion failure                 | Remove Task from Clockify        | -                              |                                                                     |
| Structure output2          | NoOp                    | Flow management for task-level data                      | Compare Datasets2               | Merge                          |                                                                     |
| Merge                     | Merge                   | Master merge node joining clients, projects, tasks       | Structure output, Structure output1, Structure output2 | Limit                         | Coordinates end of sync process                                    |
| Limit                     | Limit                   | Final execution limiter to regulate run                   | Merge                          | No Operation1                 |                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Webhook** node: set HTTP method "POST" and path (e.g., unique ID).  
   - Add a **Schedule Trigger** node: set daily interval at 4:00 AM.  
   - Connect both to the next node.

2. **Get Workspace ID:**  
   - Add a **Clockify** node: resource = "workspace", operation = "list", limit = 1.  
   - Use your Clockify API credentials.

3. **Set Global Workspace ID:**  
   - Add a **Set** node: assign variable `workspace_id` to `{{$json["id"]}}` from previous workspace node.

4. **Create No Operation placeholder node.**

5. **Clients Sync Block:**  
   - Fetch Notion clients: Notion node, operation "getAll", database set to Clients database, filter by archive checkbox = false.  
   - Fetch Clockify clients: Clockify node, resource "client", getAll, filter archived = false, workspaceId = global workspace_id.  
   - Map necessary client fields in two Set nodes for Notion and Clockify data respectively (map name, archived status, Clockify client ID).  
   - Add a Compare Datasets node to merge by "clockify_client_id".  
   - Branch "If unmapped in Notion" (filter where Clockify client ID is empty):  
     - Create Client in Clockify node for new clients.  
   - Store Clockify ID in Notion (update Notion client page with new Clockify ID).  
   - Update Client in Clockify for changes on existing clients.  
   - Add Notion node to check if a client is archived; then a Set node to mark archive true; update Client in Clockify accordingly.  
   - Remove Client from Clockify node: delete operation for clients removed in Notion but still existing in Clockify.  
   - Use Stop and Error node to handle failed removal.

6. **Projects Sync Block:**  
   - Similar pattern to clients: use Notion (Buckets database) to get active projects (filtered by status not equal to "Done" or "Obsolete").  
   - Get Clockify projects similarly with archived false.  
   - Map relevant fields including project-client relationships.  
   - Compare datasets by "clockify_project_id".  
   - Handle unmapped projects in Notion: create in Clockify (use HTTP Request node with proper API call).  
   - Save Clockify project ID back to Notion project page.  
   - Update projects in Clockify with relevant fields (archived, clientId, name).  
   - Remove projects from Clockify when archived or gone in Notion, with error handling.

7. **Tasks Sync Block:**  
   - Get active tasks from Notion with filters by status (exclude "Done" and "Obsolete") and with client relation nonempty.  
   - Fetch active tasks from Clockify per project.  
   - Map task fields including Clockify IDs and project references.  
   - Compare by "clockify_task_id".  
   - Create tasks in Clockify if unmapped; store IDs in Notion.  
   - Update tasks in Clockify (name and status).  
   - Remove tasks from Clockify as necessary with error catch.

8. **Merging and Limits:**  
   - Use Merge nodes to join outputs from the three entity blocks sequentially.  
   - Add Limit nodes to control concurrency and avoid flooding Notion or Clockify APIs.  
   - NoOp nodes to manage and organize data flow.

9. **Credentials Setup:**  
   - Configure Clockify API credentials with OAuth2 or API key for all Clockify nodes and HTTP Request nodes (ensure uniform use).  
   - Configure Notion API credentials with integration token allowing database read/write as configured.

10. **Sticky Notes and Documentation:**  
    - Add Sticky Notes describing setup instructions, required database properties, and usage tips as per original workflow.

11. **Activation:**  
    - Test webhook trigger by POST request or via embedded Notion button.  
    - Enable scheduled trigger to run daily at designated time.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|----------------|
| Workflow demo & explanation video available: ![Watch](https://youtu.be/qr0cvtAAMrc) | https://youtu.be/qr0cvtAAMrc |
| Prerequisites include defined Notion database schema for Clients, Projects, and Tasks. | See large yellow sticky note in the workflow detailing Required Properties |
| Notion Template to start with: [Notion Template Link](https://steadfast-banjo-d1f.notion.site/1ae82b476c84808e9409c08baf382c45) | |
| Related workflows: Backup Clockify reports to GitHub, Prevent simultaneous workflow executions with Redis | https://n8n.io/workflows/3147-backup-clockify-to-github-based-on-monthly-reports/ <br> https://n8n.io/workflows/2270-prevent-simultaneous-workflow-executions-with-redis/ |
| Optional: Add "Clockify Report" formula field in Notion projects for report links. | Formula code present in sticky note for convenient reporting links. |
| Optional: Add webhook-based sync button in Notion requires paid plan | Instructions detailed in sticky notes |

---

This structured documentation provides a clear technical reference for developers or AI agents to understand the full workflow architecture, node-level configurations, critical decision points, and recreation steps necessary for modification or deployment.