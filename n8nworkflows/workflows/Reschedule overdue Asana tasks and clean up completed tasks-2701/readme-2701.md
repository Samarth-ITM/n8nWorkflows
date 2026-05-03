Reschedule overdue Asana tasks and clean up completed tasks

https://n8nworkflows.xyz/workflows/reschedule-overdue-asana-tasks-and-clean-up-completed-tasks-2701


# Reschedule overdue Asana tasks and clean up completed tasks

### 1. Workflow Overview

This n8n workflow automates the management of Asana tasks by focusing on two key operations: rescheduling overdue tasks and cleaning up completed tasks that are overdue. It is designed to run on a schedule (e.g., daily at 7 AM) and performs the following logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow automatically at a specified time.
- **1.2 Task Retrieval**: Fetches all tasks assigned to a specific user within a selected Asana workspace that have been completed since today.
- **1.3 Task Details Fetching**: Retrieves detailed information for each task obtained.
- **1.4 Task Status Evaluation**: Determines if a task is still open (not completed).
- **1.5 Overdue Check and Rescheduling**: For open tasks, checks if the due date is in the past and, if so, updates the due date to the current day.
- **1.6 Cleanup of Completed Overdue Tasks**: For completed tasks, deletes those that have an overdue date to keep the workspace clean.

This structure ensures that no overdue open tasks are forgotten by rescheduling them to today, while completed overdue tasks are removed to reduce clutter.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow automatically every day at 7 AM to ensure timely task management.

- **Nodes Involved:**  
  - Everyday at 7am

- **Node Details:**  
  - **Everyday at 7am**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger daily at 7:00 AM (hour 7)  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Get user tasks" node  
    - Edge Cases: If n8n server is down or paused at trigger time, the workflow will not run; no retry mechanism here.  
    - Version: 1.2  

#### 2.2 Task Retrieval

- **Overview:**  
  Retrieves all tasks assigned to a specific user in a chosen workspace that have been completed since today, effectively fetching active and recently completed tasks.

- **Nodes Involved:**  
  - Get user tasks

- **Node Details:**  
  - **Get user tasks**  
    - Type: Asana node (operation: getAll)  
    - Configuration:  
      - Filters:  
        - Assignee: User ID (e.g., "1201727447190193")  
        - Workspace: Workspace ID (e.g., "1201727656813934")  
        - completed_since: Current date (formatted as yyyy-MM-dd) via expression `={{ DateTime.now().format('yyyy-MM-dd') }}`  
      - Return all tasks: true  
    - Inputs: From "Everyday at 7am"  
    - Outputs: To "Get task infos"  
    - Credentials: Uses configured Asana API credentials  
    - Edge Cases:  
      - API rate limits or authentication errors may cause failure.  
      - If no tasks match, downstream nodes receive empty input.  
    - Version: 1  

#### 2.3 Task Details Fetching

- **Overview:**  
  For each task retrieved, fetches detailed information including completion status and due date.

- **Nodes Involved:**  
  - Get task infos

- **Node Details:**  
  - **Get task infos**  
    - Type: Asana node (operation: get)  
    - Configuration:  
      - Task ID: Dynamic expression `={{ $json.gid }}` to fetch details for each task from previous node  
    - Inputs: From "Get user tasks"  
    - Outputs: To "Task is open?"  
    - Credentials: Same Asana API credentials  
    - Edge Cases:  
      - If a task is deleted or inaccessible, may cause errors.  
      - Network or API errors possible.  
    - Version: 1  

#### 2.4 Task Status Evaluation

- **Overview:**  
  Determines whether each task is still open (not completed) to decide the next action path.

- **Nodes Involved:**  
  - Task is open?

- **Node Details:**  
  - **Task is open?**  
    - Type: If node  
    - Configuration:  
      - Condition: Checks if the `completed` field is false (`={{ $json.completed }}` is false)  
      - Uses strict boolean validation  
    - Inputs: From "Get task infos"  
    - Outputs:  
      - True branch: Tasks that are open (not completed) → to "Due date in the past?"  
      - False branch: Tasks that are completed → to "Clean up task"  
    - Edge Cases:  
      - If `completed` field is missing or malformed, condition may fail or misroute.  
    - Version: 2.2  

#### 2.5 Overdue Check and Rescheduling

- **Overview:**  
  For open tasks, checks if the due date is before today; if yes, updates the due date to today to reschedule the task.

- **Nodes Involved:**  
  - Due date in the past?  
  - Set due date to Today

- **Node Details:**  
  - **Due date in the past?**  
    - Type: If node  
    - Configuration:  
      - Condition: Compares due date (formatted as `yyyyMMdd` by removing dashes) to current date (formatted `yyyyMMdd`)  
      - Checks if due date < today  
      - Uses loose type validation to handle string-to-number comparison  
    - Inputs: From "Task is open?" (true branch)  
    - Outputs:  
      - True branch: Due date is in the past → to "Set due date to Today"  
      - False branch: No action (no connection)  
    - Edge Cases:  
      - Tasks without a due date may cause expression errors or be skipped.  
      - Date format inconsistencies could cause false negatives.  
    - Version: 2.2  

  - **Set due date to Today**  
    - Type: Asana node (operation: update)  
    - Configuration:  
      - Task ID: `={{ $json.gid }}`  
      - Update property: `due_on` set to current date `={{ DateTime.now().format('yyyy-MM-dd') }}`  
    - Inputs: From "Due date in the past?" (true branch)  
    - Outputs: None (end of branch)  
    - Credentials: Asana API credentials  
    - Edge Cases:  
      - API errors or permission issues may prevent update.  
      - Tasks with locked or restricted fields may fail.  
    - Version: 1  

#### 2.6 Cleanup of Completed Overdue Tasks

- **Overview:**  
  Deletes tasks that are completed but still have an overdue date, to keep the workspace clean and focused on active tasks.

- **Nodes Involved:**  
  - Clean up task

- **Node Details:**  
  - **Clean up task**  
    - Type: Asana node (operation: delete)  
    - Configuration:  
      - Task ID: `={{ $json.gid }}`  
    - Inputs: From "Task is open?" (false branch)  
    - Outputs: None (end of branch)  
    - Credentials: Asana API credentials  
    - Edge Cases:  
      - Deletion failures due to permissions or API errors.  
      - Risk of deleting tasks unintentionally if filtering is not precise.  
    - Version: 1  

---

### 3. Summary Table

| Node Name          | Node Type                 | Functional Role                         | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                  |
|--------------------|---------------------------|---------------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Everyday at 7am    | Schedule Trigger          | Initiates workflow daily at 7 AM      | None                  | Get user tasks           | 👆 Update the **Scheduler** here                                                             |
| Get user tasks     | Asana (getAll)            | Retrieves all user tasks since today  | Everyday at 7am       | Get task infos           | 👆 Select your **Workspace Name** & **Assignee Name** here                                   |
| Get task infos     | Asana (get)               | Fetches detailed info for each task   | Get user tasks        | Task is open?            |                                                                                              |
| Task is open?      | If                        | Checks if task is open (not completed)| Get task infos        | Due date in the past?, Clean up task |                                                                                              |
| Due date in the past? | If                      | Checks if due date is before today    | Task is open? (true)  | Set due date to Today    |                                                                                              |
| Set due date to Today | Asana (update)           | Updates task due date to today         | Due date in the past? | None                    |                                                                                              |
| Clean up task      | Asana (delete)            | Deletes completed overdue tasks        | Task is open? (false) | None                    |                                                                                              |
| Sticky Note        | Sticky Note               | Setup instructions                    | None                  | None                    | ### ⚙️ Set Up \n\n1. Add your **Asana** credentials\n2. Schedule the workflow to run at desired intervals (e.g., daily or weekly).\n3. Select your **Workspace Name** and your **Assignee Name** (user) in the **Get user tasks** node\n4. *(Optional) Tailor filtering conditions to match your preferred due-date rules and removal criteria.*\n5. **Activate the workflow** and watch your Asana workspace stay up to date and clutter-free. |
| Sticky Note1       | Sticky Note               | Scheduler update reminder             | None                  | None                    | 👆 \nUpdate the **Scheduler** here                                                           |
| Sticky Note2       | Sticky Note               | Workspace and assignee selection note | None                  | None                    | 👆 \nSelect your **Workspace Name** & **Assignee Name** here                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: "Everyday at 7am"  
   - Type: Schedule Trigger  
   - Set to trigger daily at 7:00 AM (hour 7)  
   - No credentials needed  

2. **Create an Asana node to get all user tasks**  
   - Name: "Get user tasks"  
   - Type: Asana  
   - Operation: getAll  
   - Filters:  
     - Assignee: Enter the Asana user ID (e.g., "1201727447190193")  
     - Workspace: Enter the Asana workspace ID (e.g., "1201727656813934")  
     - completed_since: Use expression `={{ DateTime.now().format('yyyy-MM-dd') }}` to get tasks completed since today  
   - Return All: true  
   - Credentials: Configure with your Asana API credentials  
   - Connect input from "Everyday at 7am"  

3. **Create an Asana node to get task details**  
   - Name: "Get task infos"  
   - Type: Asana  
   - Operation: get  
   - Task ID: Use expression `={{ $json.gid }}` to get each task's ID from previous node  
   - Credentials: Same Asana credentials  
   - Connect input from "Get user tasks"  

4. **Create an If node to check if task is open**  
   - Name: "Task is open?"  
   - Type: If  
   - Condition: Check if `completed` field is false (`={{ $json.completed }}` is false)  
   - Connect input from "Get task infos"  

5. **Create an If node to check if due date is in the past**  
   - Name: "Due date in the past?"  
   - Type: If  
   - Condition: Compare due date to today  
     - Left value: `={{ $json.due_on.replaceAll("-", "") }}` (due date without dashes)  
     - Operator: less than  
     - Right value: `={{ DateTime.now().format('yyyyMMdd') }}` (today's date)  
   - Connect input from "Task is open?" true branch  

6. **Create an Asana node to update the due date**  
   - Name: "Set due date to Today"  
   - Type: Asana  
   - Operation: update  
   - Task ID: `={{ $json.gid }}`  
   - Other Properties: Set `due_on` to `={{ DateTime.now().format('yyyy-MM-dd') }}`  
   - Credentials: Same Asana credentials  
   - Connect input from "Due date in the past?" true branch  

7. **Create an Asana node to delete completed overdue tasks**  
   - Name: "Clean up task"  
   - Type: Asana  
   - Operation: delete  
   - Task ID: `={{ $json.gid }}`  
   - Credentials: Same Asana credentials  
   - Connect input from "Task is open?" false branch  

8. **(Optional) Add Sticky Note nodes for documentation**  
   - Add notes with setup instructions and reminders for scheduler and workspace/assignee selection  

9. **Configure workflow settings**  
   - Set timezone to "Europe/Paris" or your preferred timezone  
   - Set execution order to "v1" (default)  
   - Activate the workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Boost your productivity and keep your Asana workspace clutter-free by automating overdue task rescheduling and cleanup. | Workflow purpose and benefits summary                                                          |
| Scheduler can be updated to run at different times or intervals as needed.                                     | Sticky Note near "Everyday at 7am" node                                                        |
| Select your Workspace Name and Assignee Name carefully in the "Get user tasks" node to target correct tasks.  | Sticky Note near "Get user tasks" node                                                         |
| Ensure Asana API credentials are correctly configured with appropriate permissions for reading, updating, and deleting tasks. | Credential setup requirement                                                                   |
| Be cautious with task deletion to avoid accidental loss of important completed tasks.                          | Potential risk noted in cleanup block                                                          |
| For more information on Asana API and n8n Asana node operations, refer to official documentation:             | https://developers.asana.com/docs/ and https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.asana/ |

---

This document provides a complete and detailed reference for understanding, reproducing, and maintaining the "Reschedule overdue Asana tasks and clean up completed tasks" workflow in n8n.