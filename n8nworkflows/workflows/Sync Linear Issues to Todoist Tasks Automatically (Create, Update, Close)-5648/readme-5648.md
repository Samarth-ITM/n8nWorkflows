Sync Linear Issues to Todoist Tasks Automatically (Create, Update, Close)

https://n8nworkflows.xyz/workflows/sync-linear-issues-to-todoist-tasks-automatically--create--update--close--5648


# Sync Linear Issues to Todoist Tasks Automatically (Create, Update, Close)

### 1. Workflow Overview

This workflow automates synchronization from Linear issue tracking to Todoist task management. It listens for issue events in Linear and ensures that Todoist tasks are created, updated, or closed accordingly. The key use case is to maintain a one-way sync that reflects Linear issues status changes in Todoist, including handling sub-issues with contextual titles.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Captures new or updated issues from Linear via webhook trigger.
- **1.2 Action Routing:** Routes processing based on issue action type (`create`, `update`, `remove`).
- **1.3 Task Existence Check:** Queries Todoist to find if a corresponding task already exists.
- **1.4 Conditional Processing:** Checks due date presence, assignee identity, and issue completion state.
- **1.5 Sub-Issue Enrichment:** For sub-issues, fetches parent issue details to compose enriched task titles.
- **1.6 Task Operations:** Creates, updates, closes, or deletes Todoist tasks based on the processed data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow by listening for issue-related events from Linear. It triggers on issue creation, update, or removal.
- **Nodes Involved:**  
  - `New issue or updated issue`
- **Node Details:**

  - **New issue or updated issue**
    - Type: `linearTrigger` (Webhook trigger node for Linear)
    - Configuration:
      - Team ID set to a specific Linear team.
      - Resource: `issue` (listens to issue events).
      - Authentication: OAuth2, connected to a Linear account credential.
      - Webhook ID provided for Linear to push events.
    - Inputs: None (trigger node).
    - Outputs: Emits issue event data including the `action` (create, update, remove) and issue details.
    - Edge Cases:
      - OAuth token expiration or invalidation.
      - Webhook registration failure.
      - Missing or malformed event payload.
    - Version-specific: Uses version 1 of the Linear Trigger node.

---

#### 1.2 Action Routing

- **Overview:** Routes execution based on the issue action (`create`, `update`, `remove`). This ensures different operational flows for each case.
- **Nodes Involved:**  
  - `Switch based on action`
- **Node Details:**

  - **Switch based on action**
    - Type: `switch`
    - Configuration:
      - Routes based on `$json.action` with strict case-sensitive matching.
      - Outputs labeled as `remove`, `create`, and `update`.
    - Inputs: From `New issue or updated issue`.
    - Outputs: Branches to different task existence checking nodes and onward logic.
    - Edge Cases:
      - Unknown or unsupported action values.
      - Missing `action` property in payload.
    - Version-specific: Version 3.2 of Switch node.

---

#### 1.3 Task Existence Check

- **Overview:** Determines if a Todoist task corresponding to the Linear issue already exists, to avoid duplication and decide whether to create or update.
- **Nodes Involved:**  
  - `Check if task already exists1`  
  - `Check if task already exists3`
- **Node Details:**

  - **Check if task already exists1** (for remove actions)
    - Type: `todoist`
    - Operation: `getAll` with a custom filter searching by issue ID.
    - Credentials: Uses configured Todoist OAuth credentials.
    - Inputs: From `Switch based on action` (remove branch).
    - Outputs: List of matching tasks or empty array.
    - Edge Cases:
      - API rate limits or downtime.
      - Search filter syntax errors.
      - No matching tasks found.
    - Version: 2.1

  - **Check if task already exists3** (for create and update actions)
    - Same configuration as above, but invoked on create/update branches.
    - Outputs: Matching task(s) or empty.
    - Edge Cases and notes similar to above.

---

#### 1.4 Conditional Processing

- **Overview:** Applies filters and conditional logic to decide next steps based on task existence, due date presence, assignee, and issue completion state.
- **Nodes Involved:**  
  - `if task already exists1`  
  - `if task already exists3`  
  - `If action's due date is not empty and assignee is me`  
  - `If action's due date is not empty and assignee is me1`  
  - `Task finished ?`  
  - `Task finished ?1`  
  - `if task already exists1` and `if task already exists3` connect to deletion or noop nodes.
- **Node Details:**

  - **if task already exists1**
    - Type: `if`
    - Checks if the task found is not empty (meaning task exists).
    - If true: proceed to remove task (for remove action).
    - If false: do nothing.
    - Edge Cases: Empty response arrays, malformed data.

  - **if task already exists3**
    - Similar check for create/update actions.
    - If task exists: proceed to update or close.
    - If not: proceed to creation logic.

  - **If action's due date is not empty and assignee is me**
    - Type: `if`
    - Checks two conditions:
      - The issue has a non-empty `dueDate`.
      - The assignee's email matches a hardcoded value (`youremail@example.com`).
    - If true: proceeds to sub-issue checks or task removal.
    - If false: branches to do nothing node.
    - Edge Cases:
      - Emails mismatched.
      - Missing due date fields.

  - **Task finished ?**
    - Type: `switch`
    - Checks if issue state is `"Done"` or one of other states (`Todo`, `In Progress`, `Backlog`, `In Review`).
    - If `"Done"`: close Todoist task.
    - Else: update task.
    - Edge Cases:
      - Unknown state names.
      - Missing state data.

  - Corresponding nodes with `1` suffix handle similar logic for create/update flows.

---

#### 1.5 Sub-Issue Enrichment

- **Overview:** For sub-issues (issues with a parent), this block fetches the parent issue's title and appends it to the sub-issue title for clarity in Todoist.
- **Nodes Involved:**  
  - `If it's a sub-issue`  
  - `Get parent issue`  
  - `Set title with parent and sub-issue`  
  - `If it's a sub-issue1`  
  - `Get parent issue1`  
  - `Set title with parent and sub-issue1`
- **Node Details:**

  - **If it's a sub-issue**
    - Type: `if`
    - Checks if `parentId` is non-empty.
    - If true: fetch parent issue.
    - If false: continue with normal flow.
    - Edge Cases: Missing `parentId` field.

  - **Get parent issue**
    - Type: `linear`
    - Operation: `get` a single issue by ID (`parentId`).
    - Credentials: Linear OAuth2 credentials.
    - Edge Cases:
      - API errors.
      - Parent issue missing or deleted.

  - **Set title with parent and sub-issue**
    - Type: `set`
    - Combines parent title and current issue title into format: `[Parent] -> Sub-Issue`.
    - Preserves other fields.
    - Edge Cases:
      - Empty titles.

  - The nodes with suffix `1` perform the same logic applied to create/update paths.

---

#### 1.6 Task Operations

- **Overview:** Final step: creates, updates, closes, or deletes Todoist tasks based on processed data.
- **Nodes Involved:**  
  - `Create task2`  
  - `Create task3`  
  - `Update task`  
  - `Update task1`  
  - `Close task`  
  - `Close task1`  
  - `Remove task`  
  - `Remove task1`  
  - `Do nothing1`, `Do nothing2` (noOp nodes for terminal no-op flows)
- **Node Details:**

  - **Create task2 / Create task3**
    - Type: `todoist`
    - Operation: `create`
    - Sets:
      - Content: issue title (or enriched title).
      - Description: includes issue URL, description, and ID.
      - Due date: issue due date.
      - Labels: adds label `linear`.
      - Project: fixed Todoist project ID (Inbox).
    - Credentials: Todoist OAuth.
    - Edge Cases:
      - Invalid project ID.
      - Missing required fields.
      - Rate limiting.

  - **Update task / Update task1**
    - Type: `todoist`
    - Operation: `update`
    - Updates existing task by ID with new content, description, due date.
    - Credentials: Todoist OAuth.
    - Edge Cases:
      - Task ID invalid or deleted.
      - Conflicts with concurrent edits.

  - **Close task / Close task1**
    - Type: `todoist`
    - Operation: `close`
    - Closes the task by ID when corresponding Linear issue is marked done.
    - Edge Cases:
      - Task already closed.
      - Task ID invalid.

  - **Remove task / Remove task1**
    - Type: `todoist`
    - Operation: `delete`
    - Deletes the task from Todoist when issue is removed in Linear.
    - Edge Cases:
      - Task ID invalid or already deleted.

  - **Do nothing1 / Do nothing2**
    - Type: `noOp`
    - Serves to gracefully end flow branches where no action is required.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                                                       |
|-----------------------------------|---------------------|----------------------------------------|--------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| New issue or updated issue         | linearTrigger       | Trigger on Linear issue events          | None                           | Switch based on action                 | See Sticky Note1: Listening for Linear Issue Events [Link](https://docs.n8n.io/integrations/linear/)                              |
| Switch based on action             | switch              | Branches flow by action (create/update/remove) | New issue or updated issue     | Check if task already exists1, Check if task already exists3 | See Sticky Note2: Routing by action [Link](https://docs.n8n.io/nodes/n8n-nodes-base.switch/)                                      |
| Check if task already exists1      | todoist             | Searches for existing Todoist task (remove) | Switch based on action          | if task already exists1                | See Sticky Note3: Checking for existing task [Link](https://docs.n8n.io/integrations/todoist/)                                     |
| if task already exists1            | if                  | Checks if task exists to delete or no-op | Check if task already exists1   | Remove task, Do nothing1               |                                                                                                                                |
| Remove task                       | todoist             | Deletes Todoist task for removed issue | if task already exists1         | None                                  |                                                                                                                                |
| Do nothing1                      | noOp                | Ends flow when no delete needed         | if task already exists1         | None                                  |                                                                                                                                |
| Check if task already exists3      | todoist             | Searches for existing Todoist task (create/update) | Switch based on action          | if task already exists3                |                                                                                                                                |
| if task already exists3            | if                  | Checks if task exists to decide update/create | Check if task already exists3   | If action's due date is not empty and assignee is me, If action's due date is not empty and assignee is me1 |                                                                                                                                |
| If action's due date is not empty and assignee is me | if       | Verifies issue has due date and assignee is me | if task already exists3         | If it's a sub-issue, Remove task1     | See Sticky Note4: Conditional logic for updates and completion [Link](https://docs.n8n.io/workflows/expressions/conditions/)       |
| Remove task1                     | todoist             | Deletes Todoist task when conditions met | If action's due date is not empty and assignee is me | None                                  |                                                                                                                                |
| If it's a sub-issue               | if                  | Checks if issue is a sub-issue          | If action's due date is not empty and assignee is me | Get parent issue, Task finished ?1    | See Sticky Note5: Handling sub-issues with context [Link](https://docs.n8n.io/nodes/n8n-nodes-base.linear/)                        |
| Get parent issue                 | linear              | Fetches parent issue details             | If it's a sub-issue             | Set title with parent and sub-issue   |                                                                                                                                |
| Set title with parent and sub-issue | set                | Combines parent and sub-issue titles    | Get parent issue                | Task finished ?                       |                                                                                                                                |
| Task finished ?                  | switch              | Checks if issue is done or not           | Set title with parent and sub-issue | Close task, Update task               | See Sticky Note6: Create, update, or close Todoist tasks [Link](https://docs.n8n.io/integrations/todoist/)                          |
| Close task                      | todoist             | Closes Todoist task when issue is done  | Task finished ?                 | None                                  |                                                                                                                                |
| Update task                     | todoist             | Updates existing Todoist task            | Task finished ?                 | None                                  |                                                                                                                                |
| If action's due date is not empty and assignee is me1 | if       | Variant check for create/update flows    | if task already exists3         | If it's a sub-issue1, Do nothing2     |                                                                                                                                |
| Do nothing2                     | noOp                | Ends flow when no action needed          | If action's due date is not empty and assignee is me1 | None                                  |                                                                                                                                |
| If it's a sub-issue1             | if                  | Checks sub-issue for create/update flows | If action's due date is not empty and assignee is me1 | Get parent issue1, Create task2       |                                                                                                                                |
| Get parent issue1               | linear              | Fetches parent issue details (create/update) | If it's a sub-issue1            | Set title with parent and sub-issue1  |                                                                                                                                |
| Set title with parent and sub-issue1 | set                | Combines titles for create/update flows  | Get parent issue1               | Create task3                         |                                                                                                                                |
| Create task2                   | todoist             | Creates new task with enriched title     | If it's a sub-issue1            | None                                  |                                                                                                                                |
| Create task3                   | todoist             | Creates new task with enriched title     | Set title with parent and sub-issue1 | None                                  |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Linear Trigger Node**  
   - Type: `linearTrigger`  
   - Configure Team ID to your Linear team.  
   - Set resource to `issue`.  
   - Use OAuth2 credentials for Linear.  
   - Save webhook URL and register it in Linear for issue events.

2. **Add Switch Node to Route Actions**  
   - Type: `switch`  
   - Set expression to `$json.action`.  
   - Define outputs for values: `remove`, `create`, `update`.  

3. **Add Todoist Node to Check Existing Task (Remove branch)**  
   - Type: `todoist`  
   - Operation: `getAll`  
   - Filter: Search by issue ID in task content using syntax `=rechercher : {{ $json.data.id }}`.  
   - Use Todoist OAuth credentials.

4. **Add IF Node to Check if Task Exists (Remove branch)**  
   - Condition: Check if the array returned is not empty.

5. **Add Todoist Node to Remove Task**  
   - Operation: `delete`  
   - Task ID from existing task found.

6. **Add NoOp Node for Do Nothing (Remove branch)**

7. **Add Todoist Node to Check Existing Task (Create/Update branches)**  
   - Same as step 3.

8. **Add IF Node to Check if Task Exists (Create/Update branches)**

9. **Add IF Node to Check Due Date & Assignee**  
   - Conditions:  
     - `dueDate` is not empty.  
     - `assignee.email` equals your Linear email (replace `youremail@example.com`).

10. **Add IF Node to Check if Issue is a Sub-Issue**  
    - Condition: `parentId` is not empty.

11. **Add Linear Node to Get Parent Issue**  
    - Operation: `get`  
    - Use `parentId` from issue data.  

12. **Add Set Node to Compose Title with Parent and Sub-Issue**  
    - Set `title` to `[Parent Title] -> Sub-Issue Title`.

13. **Add Switch Node to Check if Task is Finished**  
    - Condition on issue state name: equals `Done` or one of the active states.

14. **Add Todoist Node to Close Task**  
    - Operation: `close`  
    - Use existing task ID.

15. **Add Todoist Node to Update Task**  
    - Operation: `update`  
    - Update content, description, due date.

16. **Add Todoist Node to Create Task**  
    - Operation: `create`  
    - Content: enriched or regular title.  
    - Description: issue URL, description, ID.  
    - Due date: issue due date.  
    - Labels: add `linear`.  
    - Project: set to your Todoist Inbox or desired project.

17. **Add NoOp Nodes to end branches where no action is needed.**

18. **Connect nodes according to the logical flow:**  
    - Trigger → Switch (action) → check existence → conditional checks → sub-issue handling → task operations.

19. **Set credentials:**  
    - Configure OAuth2 for Linear and Todoist nodes.  
    - Replace placeholder email `youremail@example.com` with your actual Linear account email.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| ✨ This workflow automates syncing your Linear issues to Todoist tasks automatically. It handles creation, updating, and closing tasks based on issue events from Linear. Replace your email in IF nodes to personalize the sync and customize project IDs or labels as needed.                                                        | Main workflow description and usage notes                                                                 |
| Learn more about the Linear Trigger node to understand webhook event options and setup.                                                                                                                                                                                                      | https://docs.n8n.io/integrations/linear/                                                                   |
| See how Switch nodes work to route execution paths based on conditions.                                                                                                                                                                                                                       | https://docs.n8n.io/nodes/n8n-nodes-base.switch/                                                           |
| Explore Todoist node documentation for details on task operations like create, update, delete, and close.                                                                                                                                                                                    | https://docs.n8n.io/integrations/todoist/                                                                  |
| Learn about IF and Switch node condition logic and expressions to customize filtering and branching.                                                                                                                                                                                        | https://docs.n8n.io/workflows/expressions/conditions/                                                      |
| Example of fetching parent issue data and enriching sub-issue titles to improve task clarity in Todoist.                                                                                                                                                                                     | https://docs.n8n.io/nodes/n8n-nodes-base.linear/                                                           |
| This workflow is one-way sync (Linear ➜ Todoist). Bi-directional sync is not supported but could be implemented as an enhancement.                                                                                                                                                         | —                                                                                                          |
| Join the n8n forum or Discord for community help and support.                                                                                                                                                                                                                                | https://community.n8n.io/, https://discord.com/invite/XPKeKXeB7d                                           |
| Default Todoist project is set to Inbox (`2353872970`). Adjust project ID in create/update nodes to target other projects.                                                                                                                                                                  | Project ID in Todoist node parameters                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. All data and operations comply with applicable content policies. No illegal or protected content is included.