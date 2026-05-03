Create, update and get a task in Microsoft To Do

https://n8nworkflows.xyz/workflows/create--update-and-get-a-task-in-microsoft-to-do-1114


# Create, update and get a task in Microsoft To Do

---

### 1. Workflow Overview

This workflow automates task management within Microsoft To Do by enabling users to create a new task, update its status, and then retrieve the task details sequentially. It is designed primarily for users who want to programmatically manage tasks on Microsoft To Do via n8n, such as in personal productivity automation or team task tracking.

The workflow is structured into three main logical blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow to start the task management process.
- **1.2 Task Creation:** Creating a new task in a specified Microsoft To Do list with a high importance level.
- **1.3 Task Update and Retrieval:** Updating the newly created task’s status to "In Progress" and then fetching the task’s latest details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block initiates the workflow manually. It acts as the entry point allowing the user to start the process on demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Starts workflow execution manually without any input data required.  
  - **Configuration Choices:** No parameters configured; it simply waits for manual trigger.  
  - **Key Expressions/Variables:** None used here.  
  - **Input/Output:** No input; outputs trigger the next node.  
  - **Version Requirements:** Compatible with all n8n versions supporting manual triggers.  
  - **Potential Failures:** None expected; manual node only fails if user does not click execute.

#### 1.2 Task Creation

- **Overview:**  
  Creates a new task in a specified Microsoft To Do task list, setting its importance level to "High" by default. This serves as the fundamental creation step to generate a task that subsequent nodes will modify and retrieve.

- **Nodes Involved:**  
  - Microsoft To Do

- **Node Details:**  
  - **Node Name:** Microsoft To Do  
  - **Type:** Microsoft To Do node (API integration)  
  - **Technical Role:** Executes the "create" operation on Microsoft To Do API to add a new task.  
  - **Configuration Choices:**  
    - Operation: Create  
    - Task List ID: Predefined static ID corresponding to a specific Microsoft To Do list.  
    - Additional Fields: Importance set to "High" (can be customized).  
    - Title: "Document Microsoft To Do node" (static task title).  
  - **Key Expressions/Variables:**  
    - Task List ID is static (hardcoded string).  
  - **Input/Output Connections:**  
    - Input: Manual trigger node’s output.  
    - Output: Passes newly created task data (including task ID) to the next node.  
  - **Version Requirements:** Requires OAuth2 credentials configured for Microsoft To Do API access.  
  - **Potential Failures:**  
    - Authentication errors if OAuth token expires or is invalid.  
    - Network or API rate limits causing timeouts or errors.  
    - Invalid task list ID or permissions issues on the Microsoft To Do side.  
  - **Credential:** Uses OAuth2 credentials labeled "Microsoft OAuth Credentials".  
  - **Sub-workflow:** None.

#### 1.3 Task Update and Retrieval

- **Overview:**  
  Updates the status of the task created in the previous step to "In Progress" and then retrieves the task details to confirm the update or use the task data further.

- **Nodes Involved:**  
  - Microsoft To Do1  
  - Microsoft To Do2

- **Node Details:**  

  - **Node Name:** Microsoft To Do1  
    - **Type:** Microsoft To Do node  
    - **Technical Role:** Updates an existing task’s status in Microsoft To Do.  
    - **Configuration Choices:**  
      - Operation: Update  
      - Task ID: Dynamically set from the output of the previous node (`{{$json["id"]}}`).  
      - Task List ID: Referenced from the "Microsoft To Do" node’s parameter for consistency.  
      - Update Fields: Status set to "inProgress".  
    - **Key Expressions/Variables:**  
      - Task ID expression: `{{$json["id"]}}` (inherits task ID from create node output).  
      - Task List ID expression: `{{$node["Microsoft To Do"].parameter["taskListId"]}}`.  
    - **Input/Output Connections:**  
      - Input: Receives data from "Microsoft To Do" node.  
      - Output: Passes updated task data to the next node.  
    - **Version Requirements:** OAuth2 credentials as above.  
    - **Potential Failures:**  
      - Fail if task ID is missing or invalid.  
      - Auth or permission errors.  
      - API failures during update.  
    - **Credential:** "Microsoft OAuth Credentials".  
    - **Sub-workflow:** None.

  - **Node Name:** Microsoft To Do2  
    - **Type:** Microsoft To Do node  
    - **Technical Role:** Retrieves the task details using the task ID to verify or use the latest task information.  
    - **Configuration Choices:**  
      - Operation: Get (default when taskId is specified)  
      - Task ID: Same dynamic expression as above (`{{$json["id"]}}`).  
      - Task List ID: Same dynamic expression as "Microsoft To Do" node.  
    - **Key Expressions/Variables:** Same as Microsoft To Do1 for task and list IDs.  
    - **Input/Output Connections:**  
      - Input: From Microsoft To Do1 output.  
      - Output: Final output of the workflow with task details.  
    - **Version Requirements:** OAuth2 credentials as above.  
    - **Potential Failures:**  
      - Missing or invalid task ID.  
      - API or network errors.  
      - Auth errors.  
    - **Credential:** "Microsoft OAuth Credentials".  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                     | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                 |
|---------------------|---------------------------|-----------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------|
| On clicking 'execute'| Manual Trigger            | Starts the workflow manually      |                        | Microsoft To Do          |                                                                                             |
| Microsoft To Do     | Microsoft To Do           | Creates a new task in MS To Do    | On clicking 'execute'   | Microsoft To Do1         | This node will create a task with the importance `High` in the Tasks list. You can select a different list as well as the importance level. |
| Microsoft To Do1    | Microsoft To Do           | Updates the status of the task    | Microsoft To Do        | Microsoft To Do2         | This node will update the status of the task that we created in the previous node.          |
| Microsoft To Do2    | Microsoft To Do           | Retrieves the created task        | Microsoft To Do1       |                         | This node will get the task that we created earlier.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Add a **Manual Trigger** node.
   - Leave parameters empty.
   - This node will start the workflow when manually executed.

2. **Add Microsoft To Do Node for Task Creation:**
   - Add a **Microsoft To Do** node.
   - Set **Operation** to `create`.
   - Set **Task List ID** to the desired list's ID (must be obtained from your Microsoft To Do account; example ID used in the workflow: `AQMkADAwATNiZmYAZC0zOTkAMy02ZWZjLTAwAi0wMAoALgAAA3i1fHMTrftIhQBzhywL64UBAFB0wRiJW1FJmmlvlAkVFQA-AAACARIAAAA=`).
   - Set **Title** to `"Document Microsoft To Do node"` or any task title desired.
   - Under **Additional Fields**, set **Importance** to `High`.
   - Configure credentials for Microsoft To Do using OAuth2:
     - Use or create credentials labeled "Microsoft OAuth Credentials" linked to your Microsoft account with the necessary Microsoft Graph API permissions for To Do.
   - Connect the output of the **Manual Trigger** node to this node.

3. **Add Microsoft To Do Node for Updating Task:**
   - Add another **Microsoft To Do** node (name it e.g., "Microsoft To Do1").
   - Set **Operation** to `update`.
   - Set **Task ID** parameter using expression: `{{$json["id"]}}` — this dynamically uses the ID from the previous node’s output.
   - Set **Task List ID** parameter using expression: `{{$node["Microsoft To Do"].parameter["taskListId"]}}` — this references the same task list as the creation node.
   - Under **Update Fields**, set **Status** to `inProgress`.
   - Use the same Microsoft OAuth2 credentials as before.
   - Connect the output of the **Microsoft To Do** creation node to this update node.

4. **Add Microsoft To Do Node for Getting Task Details:**
   - Add a third **Microsoft To Do** node (name it e.g., "Microsoft To Do2").
   - Set **Task ID** using expression: `{{$json["id"]}}` — again referencing the task ID inherited from the update node.
   - Set **Task List ID** similarly to: `{{$node["Microsoft To Do"].parameter["taskListId"]}}`.
   - This node implicitly performs a "get" operation by specifying the task ID.
   - Use the same Microsoft OAuth2 credentials.
   - Connect the output of the update node to this node.

5. **Save and Execute:**
   - Save the workflow.
   - Manually trigger the workflow by clicking the execute button on the manual trigger node.
   - Verify that the task is created, updated, and fetched successfully.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Microsoft To Do node supports multiple task lists and importance levels. Modify the taskListId and importance parameters as needed for different use cases. | Useful for adapting workflow to various task management needs.                                 |
| Ensure your Microsoft OAuth Credentials have the required Microsoft Graph API permissions (`Tasks.ReadWrite`) to create, update, and read tasks. | Microsoft Graph API documentation: https://docs.microsoft.com/en-us/graph/api/resources/todo-overview |
| For the Task List ID, you can retrieve available task lists via the Microsoft To Do node by setting operation to "getAll" to dynamically fetch lists. | Helps when setting up workflows with dynamic task list selection.                              |
| This workflow assumes sequential node execution and that the task ID is always present after creation. Validate outputs if adapting for error handling or retries. | Consider adding error workflows or conditional logic for production robustness.                |

---

This document fully describes the workflow structure, logic, and configuration needed to understand, reproduce, and maintain the Microsoft To Do task management automation implemented in n8n.