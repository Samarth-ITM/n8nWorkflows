Check dependencies before completing Awork tasks (Workaround)

https://n8nworkflows.xyz/workflows/check-dependencies-before-completing-awork-tasks--workaround--4350


# Check dependencies before completing Awork tasks (Workaround)

### 1. Workflow Overview

This workflow provides a workaround for the Awork task management system to enforce dependency and subtask completion rules before allowing a task to be marked as completed. Since Awork does not natively prevent tasks from being marked done when subtasks or prerequisite tasks remain open, this automation intercepts task status changes and enforces these constraints.

The workflow listens to task status changes via a webhook and processes the following logical blocks:

- **1.1 Input Reception:** Receives webhook calls from Awork about task status changes.
- **1.2 Configuration Setup:** Loads user-defined configuration settings controlling workflow behavior.
- **1.3 Status Change Extraction and Verification:** Extracts task status changes and filters only those marking tasks as done.
- **1.4 Dependency Check:** If enabled, loads and verifies if all prerequisite tasks are completed before allowing the current task to be marked done.
- **1.5 Tree Structure (Subtasks) Check:** If enabled, verifies that all subtasks are completed before allowing the parent task to be marked done.
- **1.6 Rollback and Commenting:** If any prerequisite or subtask is still open, rolls back the task status to its previous state and optionally adds a comment explaining why.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives webhook calls from Awork when task status changes occur.

- **Nodes Involved:**  
  - Webhook call by Awork

- **Node Details:**

  | Node Name             | Details                                                                                                   |
  |-----------------------|-----------------------------------------------------------------------------------------------------------|
  | Webhook call by Awork  | Type: Webhook (HTTP POST) <br> Role: Entry point receiving Awork webhook payload <br> Configuration: Receives POST requests at path `57b88b94-6228-4e49-897d-64bf8c7d32db`. No authentication on webhook itself; expects JSON payload containing task change info. <br> Input: External webhook call <br> Output: JSON payload with task entity and changes <br> Edge cases: Webhook unreachable, malformed payload, missing expected fields |

---

#### 2.2 Configuration Setup

- **Overview:**  
  Loads and sets configuration parameters that control workflow behavior such as which statuses are considered "done", whether to check dependencies or subtasks, and comment texts.

- **Nodes Involved:**  
  - Workflow config  
  - Fetch task changes  
  - Split out task changes  
  - Filter task status change

- **Node Details:**

  | Node Name             | Details                                                                                                   |
  |-----------------------|-----------------------------------------------------------------------------------------------------------|
  | Workflow config       | Type: Set <br> Role: Defines configuration variables <br> Key config:  
    - `finishedStateLabels`: array of status labels considered "done" (e.g., ["Fertig","Total fertig"])  
    - `useDependencies`: boolean, whether to check dependencies (default false)  
    - `useTreeStructure`: boolean, whether to check subtasks (default true)  
    - `addComments`: boolean, whether to add comments when rolling back status (default true)  
    - `commentTextDependencies` and `commentTextTreeStructure`: strings for rollback comments <br> Input: webhook payload <br> Output: JSON with config variables <br> Edge cases: Missing or malformed config values could lead to misbehavior |
  | Fetch task changes    | Type: Set <br> Role: Extracts the `changes` array from webhook payload into a separate field for processing <br> Uses expression: `={{ $('Webhook call by Awork').item.json.body.changes }}` <br> Input: config node output <br> Output: array of changes |
  | Split out task changes | Type: SplitOut <br> Role: Splits the array of changes into individual items for separate processing <br> Configured to split on field `changes` <br> Input: changes array <br> Output: individual change objects |
  | Filter task status change | Type: Filter <br> Role: Filters changes to only pass those where the property changed is `TaskStatusId` <br> Condition: `$json.property == "TaskStatusId"` <br> Input: individual changes <br> Output: changes related to task status only |

---

#### 2.3 Status Done Verification

- **Overview:**  
  Checks if the new task status corresponds to a "done" state based on configured labels and type.

- **Nodes Involved:**  
  - Check if task done

- **Node Details:**

  | Node Name             | Details                                                                                                   |
  |-----------------------|-----------------------------------------------------------------------------------------------------------|
  | Check if task done    | Type: If <br> Role: Determines if task is being marked as done <br> Conditions:  
    - Task status name contained in `finishedStateLabels` array  
    - Task status type equals `"done"` <br> Uses expressions referencing webhook data and config node <br> Input: filtered task status changes <br> Output: true branch continues workflow, false branch exits (no further processing) <br> Edge cases: Unrecognized status names or types could skip enforcement |

---

#### 2.4 Dependency Check (Optional)

- **Overview:**  
  If enabled, loads task dependencies and checks if all prerequisite tasks are completed before allowing completion.

- **Nodes Involved:**  
  - Check if dependencies enabled  
  - Load dependencies for task  
  - Aggregate task dependencies  
  - Build filter params  
  - Load all linked tasks  
  - Filter only open tasks  
  - Aggregate open task IDs  
  - Check if open tasks exist  
  - Roll back task status from done to previous state  
  - Check if comments are enabled  
  - Add comment to tasks

- **Node Details:**

  | Node Name                    | Details                                                                                                   |
  |------------------------------|-----------------------------------------------------------------------------------------------------------|
  | Check if dependencies enabled | Type: If <br> Role: Branches workflow based on `useDependencies` config boolean <br> Condition: `useDependencies == true` <br> Input: from "Check if task done" <br> Output: true branch for dependencies processing; false branch skips to tree structure check |
  | Load dependencies for task    | Type: HTTP Request <br> Role: Calls Awork API to retrieve dependencies of the current task <br> URL template: `https://api.awork.com/api/v1/tasks/{{ taskId }}/taskdependencies` <br> Auth: HTTP header auth with Awork API token <br> Input: task ID from webhook <br> Output: list of dependency objects <br> Retry enabled for transient errors <br> Edge cases: API errors, invalid task ID, network failures |
  | Aggregate task dependencies   | Type: Aggregate <br> Role: Extracts and merges all predecessor task IDs from dependencies into a single array `taskDependencies` <br> Input: dependencies list <br> Output: aggregated list of dependency IDs |
  | Build filter params           | Type: Code <br> Role: Transforms each dependency ID into a filter query segment `"id eq guid'<id>'"` for Awork API <br> Input: aggregated dependency IDs <br> Output: query filter array |
  | Load all linked tasks         | Type: HTTP Request <br> Role: Calls Awork API to load all project tasks filtered by dependency IDs <br> URL: `https://api.awork.com/api/v1/me/projecttasks` with query param `filterby` set to OR-combined dependency filters <br> Auth: Awork API token <br> Output: tasks data with statuses <br> Edge cases: large number of dependencies may cause API pagination issues |
  | Filter only open tasks        | Type: Filter <br> Role: Filters returned tasks to only those not marked as done (`taskStatus.type != "done"`) <br> Input: list of linked tasks <br> Output: open (unfinished) dependencies |
  | Aggregate open task IDs       | Type: Aggregate <br> Role: Aggregates IDs of open dependency tasks into an array <br> Input: filtered open tasks <br> Output: array of open dependency IDs |
  | Check if open tasks exist     | Type: If <br> Role: Checks if any open dependencies remain <br> Condition: aggregated open task IDs array is not empty <br> True branch triggers rollback; false branch proceeds to tree check |
  | Roll back task status from done to previous state | Type: HTTP Request <br> Role: Calls Awork API to revert task status to old status ID <br> URL: `https://api.awork.com/api/v1/tasks/changestatuses` <br> Method: POST <br> Body: JSON with taskId and old statusId from webhook data <br> Auth: Awork API token <br> Retry enabled <br> Edge cases: API failure to change status |
  | Check if comments are enabled | Type: If <br> Role: Determines if comment should be added based on config `addComments` <br> False branch exits workflow without comment |
  | Add comment to tasks          | Type: HTTP Request <br> Role: Posts a comment to the task explaining the rollback due to uncompleted dependencies <br> URL: `https://api.awork.com/api/v1/tasks/{{taskId}}/comments` <br> Body: JSON with `message` set to `commentTextDependencies` from config <br> Auth: Awork API token <br> Retry enabled |

---

#### 2.5 Tree Structure (Subtasks) Check (Optional)

- **Overview:**  
  If enabled, verifies whether all subtasks of the current task are completed before allowing the parent task to be marked done.

- **Nodes Involved:**  
  - Check if dependencies enabled (false branch)  
  - Check if tree structure enabled  
  - Check subtasks  
  - Load all open sub tasks  
  - Aggregate task IDs  
  - Check open subtasks  
  - Roll back task status from done to previous state1  
  - Check if comments are enabled1  
  - Add comment to tasks1

- **Node Details:**

  | Node Name                    | Details                                                                                                   |
  |------------------------------|-----------------------------------------------------------------------------------------------------------|
  | Check if dependencies enabled (false branch) | Skips dependency check, proceeds to tree structure check |
  | Check if tree structure enabled | Type: If <br> Role: Branches workflow based on `useTreeStructure` config boolean <br> Condition: `useTreeStructure == true` <br> Input: from dependency check or previous step <br> Output: true proceeds to subtask check; false exits workflow |
  | Check subtasks               | Type: If <br> Role: Checks if task has subtasks (`numberOfSubtasks > 0`) <br> Input: webhook entity <br> Output: true continues, false exits workflow |
  | Load all open sub tasks      | Type: HTTP Request <br> Role: Calls Awork API to load subtasks of current task with status not done <br> URL: `https://api.awork.com/api/v1/me/projecttasks` with filter: `parentid eq guid'<taskId>' and taskstatus/type ne 'done'` <br> Auth: Awork API token <br> Output: list of open subtasks |
  | Aggregate task IDs           | Type: Aggregate <br> Role: Aggregates IDs of open subtasks into array <br> Input: open subtasks list <br> Output: array of open subtask IDs |
  | Check open subtasks          | Type: If <br> Role: Checks if any subtasks remain open (array length > 0) <br> True branch triggers rollback; false branch exits workflow |
  | Roll back task status from done to previous state1 | Same as rollback node in dependency check but separate instance for subtasks branch |
  | Check if comments are enabled1 | Same as comments check but for subtasks branch |
  | Add comment to tasks1        | Posts comment to task with `commentTextTreeStructure` message from config |

---

#### 2.6 Sticky Notes and Documentation

- **Overview:**  
  Multiple sticky note nodes provide documentation and explanations within the workflow for easier maintenance and understanding.

- **Nodes Involved:**  
  All sticky note nodes, notably:  
  - Initial large sticky note explaining the overall purpose and usage  
  - Notes marking logical groups (green) and routing/exit points (red)  
  - Notes describing each logical block and decision points

---

### 3. Summary Table

| Node Name                          | Node Type         | Functional Role                                   | Input Node(s)                    | Output Node(s)                             | Sticky Note                                                                                                                       |
|----------------------------------|-------------------|-------------------------------------------------|---------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Webhook call by Awork             | Webhook           | Receives webhook from Awork on task changes     | -                               | Workflow config                             |                                                                                                                                  |
| Workflow config                  | Set               | Loads workflow configuration parameters          | Webhook call by Awork            | Fetch task changes                          | Sticky Note1: "Workflow config - Configure basic settings"                                                                       |
| Fetch task changes               | Set               | Extracts changes array from webhook payload      | Workflow config                 | Split out task changes                      | Sticky Note2: "Extract status changes from webhook call payload"                                                                 |
| Split out task changes           | SplitOut          | Splits changes array to individual items         | Fetch task changes              | Filter task status change                   |                                                                                                                                  |
| Filter task status change        | Filter            | Filters to only status changes                    | Split out task changes          | Check if task done                          |                                                                                                                                  |
| Check if task done               | If                | Checks if new status is "done"                    | Filter task status change       | Check if dependencies enabled               | Sticky Note3: "Check if task status is currently done; proceed or exit"                                                          |
| Check if dependencies enabled    | If                | Branches based on `useDependencies` config       | Check if task done              | Load dependencies for task, Check if tree structure enabled | Sticky Note4: "Check if dependencies are enabled; process dependencies or check tree structure"                                  |
| Load dependencies for task       | HTTP Request      | Loads task dependencies from Awork API           | Check if dependencies enabled  | Aggregate task dependencies                 | Sticky Note5: "Load task dependencies and filter open tasks"                                                                      |
| Aggregate task dependencies      | Aggregate         | Aggregates dependency task IDs                    | Load dependencies for task      | Build filter params                         |                                                                                                                                  |
| Build filter params              | Code              | Builds filter query for Awork API                 | Aggregate task dependencies     | Load all linked tasks                       |                                                                                                                                  |
| Load all linked tasks            | HTTP Request      | Loads tasks filtered by dependency IDs            | Build filter params             | Filter only open tasks                      |                                                                                                                                  |
| Filter only open tasks           | Filter            | Filters linked tasks to open (not done)           | Load all linked tasks           | Aggregate open task IDs                     | Sticky Note6: "Check if dependency tasks are open; rollback if yes, else check tree"                                              |
| Aggregate open task IDs          | Aggregate         | Aggregates open dependency task IDs               | Filter only open tasks          | Check if open tasks exist                   |                                                                                                                                  |
| Check if open tasks exist        | If                | Checks if any open dependencies exist             | Aggregate open task IDs         | Roll back task status from done to previous state, Check if tree structure enabled | Sticky Note7: "Roll back task status if open dependencies exist"                                                                   |
| Roll back task status from done to previous state | HTTP Request | Reverts task status to previous if dependencies open | Check if open tasks exist       | Check if comments are enabled               | Sticky Note8: "Check if comment should be added after rollback"                                                                   |
| Check if comments are enabled    | If                | Checks if comments should be added                 | Roll back task status          | Add comment to tasks, exit                  | Sticky Note9: "Add comment to task regarding rollback and then exit workflow"                                                     |
| Add comment to tasks            | HTTP Request      | Adds comment about rollback due to dependencies   | Check if comments are enabled  | End                                         |                                                                                                                                  |
| Check if tree structure enabled | If                | Branches based on `useTreeStructure` config       | Check if open tasks exist or dependencies disabled | Check subtasks                              | Sticky Note11: "Check if tree structure check is enabled; exit if not"                                                            |
| Check subtasks                 | If                | Checks if task has subtasks                         | Check if tree structure enabled | Load all open sub tasks                      | Sticky Note12: "Check if task has subtasks; exit if none"                                                                          |
| Load all open sub tasks         | HTTP Request      | Loads open subtasks (not done) from Awork API     | Check subtasks                 | Aggregate task IDs                          | Sticky Note10: "Load subtasks not marked as done"                                                                                 |
| Aggregate task IDs              | Aggregate         | Aggregates open subtasks IDs                        | Load all open sub tasks        | Check open subtasks                         |                                                                                                                                  |
| Check open subtasks             | If                | Checks if any subtasks remain open                  | Aggregate task IDs             | Roll back task status from done to previous state1, exit | Sticky Note14: "Check if open subtasks exist; exit if none"                                                                        |
| Roll back task status from done to previous state1 | HTTP Request | Reverts task status for subtasks branch            | Check open subtasks            | Check if comments are enabled1              | Sticky Note15: "Roll back task status if open subtasks exist"                                                                     |
| Check if comments are enabled1  | If                | Checks if comments should be added for subtasks    | Roll back task status from done to previous state1 | Add comment to tasks1                        | Sticky Note16: "Check if comment should be added for subtasks rollback"                                                           |
| Add comment to tasks1          | HTTP Request      | Adds comment about rollback due to subtasks        | Check if comments are enabled1 | End                                         | Sticky Note17: "Add comment to task regarding rollback for subtasks and exit workflow"                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node** named `Webhook call by Awork`:  
   - Set HTTP Method to POST.  
   - Set path to a unique string (e.g., `57b88b94-6228-4e49-897d-64bf8c7d32db`).  
   - This node receives incoming webhook calls from Awork on task changes.

2. **Add a Set node** named `Workflow config`:  
   - Create variables:  
     - `finishedStateLabels` (Array): e.g., `["Fertig","Total fertig"]`  
     - `useDependencies` (Boolean): false by default  
     - `useTreeStructure` (Boolean): true by default  
     - `addComments` (Boolean): true by default  
     - `commentTextDependencies` (String): e.g., `"Some dependencies not yet finished! Status rolled back!"`  
     - `commentTextTreeStructure` (String): e.g., `"Some child tasks not yet finished! Status rolled back!"`  
   - Connect input from webhook node.

3. **Add a Set node** named `Fetch task changes`:  
   - Set a field `changes` to expression: `={{ $('Webhook call by Awork').item.json.body.changes }}`  
   - Connect input from `Workflow config`.

4. **Add a SplitOut node** named `Split out task changes`:  
   - Set field to split on: `changes`  
   - Connect input from `Fetch task changes`.

5. **Add a Filter node** named `Filter task status change`:  
   - Filter condition: `$json.property == "TaskStatusId"`  
   - Connect input from `Split out task changes`.

6. **Add an If node** named `Check if task done`:  
   - Conditions:  
     - Task status name is in `finishedStateLabels` array from `Workflow config`.  
     - Task status type equals `"done"`.  
   - Connect input from `Filter task status change`.  
   - True branch continues workflow; False branch terminates workflow.

7. **Add an If node** named `Check if dependencies enabled`:  
   - Condition: `useDependencies` from `Workflow config` equals true.  
   - Connect input from `Check if task done`.

8. **On true branch of dependencies:**

   a. **HTTP Request node** named `Load dependencies for task`:  
      - URL: `https://api.awork.com/api/v1/tasks/{{ $json.body.entity.id }}/taskdependencies`  
      - Authentication: HTTP Header Auth with Awork API key (bearer token)  
      - Method: GET  
      - Retry on fail enabled with 2 sec wait.  
      - Connect input from `Check if dependencies enabled`.  

   b. **Aggregate node** named `Aggregate task dependencies`:  
      - Aggregate field: `predecessorId` renamed to `taskDependencies`.  
      - Merge lists enabled.  
      - Connect input from `Load dependencies for task`.

   c. **Code node** named `Build filter params`:  
      - JavaScript to transform each dependency ID into filter query string:  
        ```js
        for (const item of $input.all()) {
          for (var key in item.json.taskDependencies) {
            item.json.taskDependencies[key] = "id eq guid'" + item.json.taskDependencies[key] + "'";
          }
        }
        return $input.all();
        ```  
      - Connect input from `Aggregate task dependencies`.

   d. **HTTP Request node** named `Load all linked tasks`:  
      - URL: `https://api.awork.com/api/v1/me/projecttasks`  
      - Method: GET  
      - Query Parameter: `filterby` set to the joined `taskDependencies` strings with " or "  
      - Authentication: Awork API key  
      - Retry enabled  
      - Connect input from `Build filter params`.

   e. **Filter node** named `Filter only open tasks`:  
      - Condition: `taskStatus.type != "done"`  
      - Connect input from `Load all linked tasks`.

   f. **Aggregate node** named `Aggregate open task IDs`:  
      - Aggregate field: `id`  
      - Connect input from `Filter only open tasks`.

   g. **If node** named `Check if open tasks exist`:  
      - Condition: aggregated `id` array is not empty  
      - True branch triggers rollback; False branch continues to tree structure check  
      - Connect input from `Aggregate open task IDs`.

   h. **HTTP Request node** named `Roll back task status from done to previous state`:  
      - URL: `https://api.awork.com/api/v1/tasks/changestatuses`  
      - Method: POST  
      - Body JSON:  
        ```json
        [
          {
            "taskId": "{{ $json.body.entity.id }}",
            "statusId": "{{ $json.old }}"
          }
        ]
        ```  
      - Authentication: Awork API key  
      - Retry enabled  
      - Connect input from `Check if open tasks exist` (true branch).

   i. **If node** named `Check if comments are enabled`:  
      - Condition: `addComments` from `Workflow config` is true  
      - Connect input from rollback node.

   j. **HTTP Request node** named `Add comment to tasks`:  
      - URL: `https://api.awork.com/api/v1/tasks/{{ $json.body.entity.id }}/comments`  
      - Method: POST  
      - Body JSON:  
        ```json
        {
          "message": "{{ $json.commentTextDependencies }}"
        }
        ```  
      - Authentication: Awork API key  
      - Retry enabled  
      - Connect input from `Check if comments are enabled` (true branch).

9. **On false branch of dependencies (or after dependency check):**

   a. **If node** named `Check if tree structure enabled`:  
      - Condition: `useTreeStructure` from `Workflow config` equals true  
      - Connect input from dependency check false branch or from `Check if open tasks exist` (false branch).

   b. **If node** named `Check subtasks`:  
      - Condition: `numberOfSubtasks > 0` from webhook entity  
      - Connect input from `Check if tree structure enabled`.

   c. **HTTP Request node** named `Load all open sub tasks`:  
      - URL: `https://api.awork.com/api/v1/me/projecttasks`  
      - Method: GET  
      - Query: `parentid eq guid'{{ $json.body.entity.id }}' and taskstatus/type ne 'done'`  
      - Authentication: Awork API key  
      - Connect input from `Check subtasks`.

   d. **Aggregate node** named `Aggregate task IDs`:  
      - Aggregate field: `id`  
      - Connect input from `Load all open sub tasks`.

   e. **If node** named `Check open subtasks`:  
      - Condition: aggregated `id` array length > 0  
      - True branch triggers rollback; false branch ends workflow  
      - Connect input from `Aggregate task IDs`.

   f. **HTTP Request node** named `Roll back task status from done to previous state1`:  
      - Same settings as rollback node for dependencies  
      - Connect input from `Check open subtasks` (true branch).

   g. **If node** named `Check if comments are enabled1`:  
      - Condition: `addComments` is true  
      - Connect input from rollback node.

   h. **HTTP Request node** named `Add comment to tasks1`:  
      - Posts comment with `commentTextTreeStructure` message  
      - Connect input from `Check if comments are enabled1` (true branch).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow is a workaround for Awork's current limitation where tasks can be marked done even if dependent tasks or subtasks are incomplete. It intercepts task status changes via webhook and enforces completion constraints, rolling back status and adding comments if necessary.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note initial large documentation node                                                                         |
| The workflow does not use the Awork community nodes package due to incomplete API support. Instead, it uses HTTP Request nodes with direct API calls. The official documentation for Awork's API and webhooks can be found at https://support.awork.com/en/articles/5415462-webhooks and https://support.awork.com/en/articles/5415664-client-applications-and-api-keys.                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note initial documentation, links inside                                                                        |
| To use the workflow, create a webhook in Awork configured for task status changes, enable "Only events from first level properties" to reduce calls, and configure n8n credentials with an Awork API key using HTTP header authorization with bearer token.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note initial documentation                                                                                     |
| Green sticky notes in the workflow visually group logical node blocks, while red sticky notes mark routing decisions or workflow exit points.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Notes nodes labeled Sticky Note10, Sticky Note18, and Sticky Note1, Sticky Note2, etc.                        |
| The workflow uses retry on fail and wait between tries (2 seconds) on HTTP requests to handle transient API or network errors gracefully.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | HTTP Request nodes parameters                                                                                        |
| The workflow assumes the task's previous status ID is included in the webhook payload's `old` field for rollback purposes. If this is missing, rollback will fail.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Implied by usage of `old` status ID from webhook data                                                                 |
| Comments added upon rollback serve as user hints to explain why the task status was reverted, improving transparency and user experience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Comment text variables and Add comment nodes                                                                          |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.