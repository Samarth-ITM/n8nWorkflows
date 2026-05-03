Sync tasks automatically from Todoist to Notion

https://n8nworkflows.xyz/workflows/sync-tasks-automatically-from-todoist-to-notion-1778


# Sync tasks automatically from Todoist to Notion

### 1. Workflow Overview

This workflow automates the synchronization of tasks from Todoist to Notion based on a specific label. Its primary use case is to monitor Todoist for tasks tagged with a designated label (e.g., “send-to-notion”) and then create corresponding pages in a Notion database to reflect these tasks. After successful creation in Notion, the Todoist task is updated to reflect the synchronization status.

The workflow is logically divided into the following blocks:

- **1.1 Triggering and Input Retrieval:** Periodically triggers the workflow and fetches all Todoist tasks with the specific label.
- **1.2 Notion Page Creation:** For each fetched task, creates a new page in a Notion database.
- **1.3 Post-Processing Update:** Updates the original Todoist task label and description to indicate it has been synced and adds a Notion link.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering and Input Retrieval

- **Overview:** This block periodically triggers the workflow every second and retrieves all Todoist tasks labeled “send-to-notion".
- **Nodes Involved:**  
  - On schedule  
  - Get all tasks with specific label

- **Node Details:**

  - **On schedule**  
    - **Type:** Schedule Trigger  
    - **Role:** Initiates the workflow execution every second to check for new tasks.  
    - **Configuration:** Interval set to trigger every 1 second (field: seconds).  
    - **Inputs:** None (trigger node).  
    - **Outputs:** Connects to “Get all tasks with specific label”.  
    - **Edge Cases:** Potential high API call frequency may cause rate limiting or performance issues; consider increasing interval for production.  
    - **Version:** 1.1

  - **Get all tasks with specific label**  
    - **Type:** Todoist node (operation: getAll)  
    - **Role:** Queries Todoist for all tasks that have the label “send-to-notion”.  
    - **Configuration:**  
      - Filter: labelId set to “send-to-notion” (the exact label name).  
      - Authentication: OAuth2 with Todoist account credentials.  
    - **Key Expressions:** Uses static filter labelId.  
    - **Inputs:** Receives trigger from “On schedule”.  
    - **Outputs:** Feeds the matching tasks to the “Add to Notion database” node.  
    - **Edge Cases:**  
      - If no tasks have the label, output will be empty, potentially skipping subsequent nodes.  
      - OAuth token expiration or invalid credentials can cause auth errors.  
    - **Version:** 2

#### 2.2 Notion Page Creation

- **Overview:** Creates a new page in a specified Notion database for each Todoist task fetched. The new page title mirrors the task content, and a numeric property stores the Todoist task ID.
- **Nodes Involved:**  
  - Add to Notion database

- **Node Details:**

  - **Add to Notion database**  
    - **Type:** Notion node (resource: databasePage, operation: create)  
    - **Role:** Creates a new database page in Notion for each task from Todoist.  
    - **Configuration:**  
      - Title field set dynamically using `{{$json.content}}` (the task content/title from Todoist).  
      - Database ID statically configured to target a specific Notion database (“My Todoist Tasks”).  
      - Properties: Sets a numeric property “Todoist ID” equal to the parsed integer form of the Todoist task ID (`={{ parseInt($json.id) }}`).  
    - **Inputs:** Receives each Todoist task item from “Get all tasks with specific label”.  
    - **Outputs:** Passes the newly created Notion page data to “Replace label on task”.  
    - **Edge Cases:**  
      - Failure if databaseId is invalid or credentials expired.  
      - Data type mismatch if Todoist ID parsing fails.  
      - Rate limits or API downtime.  
    - **Version:** 2.1  
    - **Credentials:** Requires configured Notion API credentials with access to the database.

#### 2.3 Post-Processing Update

- **Overview:** After creating the Notion page, updates the original Todoist task by replacing its label with “sent” and appending the Notion page link to the task description.
- **Nodes Involved:**  
  - Replace label on task

- **Node Details:**

  - **Replace label on task**  
    - **Type:** Todoist node (operation: update)  
    - **Role:** Modifies the original Todoist task to indicate successful synchronization.  
    - **Configuration:**  
      - Task ID dynamically obtained from the original task (`={{ $('Get all tasks with specific label').item.json.id }}`).  
      - Updates labels to contain only “sent”.  
      - Updates description by prepending the Notion page URL (`Notion Link: {{ $json.url }}`) followed by the original task description.  
      - Authentication: OAuth2 with Todoist credentials.  
    - **Inputs:** Receives Notion page data from “Add to Notion database”.  
    - **Outputs:** None (end of chain).  
    - **Edge Cases:**  
      - If task ID is invalid or task was deleted in the meantime, update will fail.  
      - If Notion URL is missing or malformed, description update may be incorrect.  
      - OAuth token expiration can cause auth errors.  
    - **Version:** 2

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                         | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|-------------------------------|----------------------------|---------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| On schedule                   | Schedule Trigger           | Periodic trigger every second          | None                         | Get all tasks with specific label |                                                                                              |
| Get all tasks with specific label | Todoist                   | Fetch all tasks labeled “send-to-notion” | On schedule                  | Add to Notion database       |                                                                                              |
| Add to Notion database        | Notion                     | Create a new Notion database page for each task | Get all tasks with specific label | Replace label on task        |                                                                                              |
| Replace label on task         | Todoist                    | Update Todoist task label and description after sync | Add to Notion database       | None                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **Schedule Trigger** node named “On schedule”.  
   - Configure interval: set to trigger every 1 second.  
   - No credentials required.

2. **Add the Todoist GetAll Node**  
   - Add a **Todoist** node named “Get all tasks with specific label”.  
   - Set operation to **getAll**.  
   - Under filters, set **labelId** to “send-to-notion” (ensure this label exists in your Todoist).  
   - Select authentication method as **OAuth2** and link your Todoist credentials.  
   - Connect output of “On schedule” to this node’s input.

3. **Add the Notion Database Page Creation Node**  
   - Add a **Notion** node named “Add to Notion database”.  
   - Set resource to **databasePage** and operation to **create**.  
   - Select the target Notion database by its ID (must exist beforehand).  
   - For the **title**, use an expression to set it to the task content: `{{$json.content}}`.  
   - Add a number property (e.g., “Todoist ID”) and set its value to the parsed integer of the Todoist task ID via expression: `={{ parseInt($json.id) }}`.  
   - Configure Notion API credentials with access to the selected database.  
   - Connect output of “Get all tasks with specific label” to this node’s input.

4. **Add the Todoist Task Update Node**  
   - Add a **Todoist** node named “Replace label on task”.  
   - Set operation to **update**.  
   - For **taskId**, use expression to dynamically get the original task ID: `={{ $('Get all tasks with specific label').item.json.id }}`.  
   - Under updateFields:  
     - Set **labels** to `["sent"]`.  
     - Set **description** to an expression concatenating the Notion page URL and original description: `=Notion Link:  {{ $json.url }}\n\n{{ $('Get all tasks with specific label').item.json.description }}`.  
   - Use OAuth2 credentials for Todoist.  
   - Connect output of “Add to Notion database” to this node’s input.

5. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Test by creating a task in Todoist with the label “send-to-notion”. After up to one second, verify a new page appears in the Notion database and the Todoist task updates its label and description.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Todoist credentials and setup guide are available at: https://docs.n8n.io/integrations/builtin/credentials/todoist/ | Official n8n documentation for Todoist OAuth2 credential configuration                         |
| Notion credentials and setup guide are here: https://docs.n8n.io/integrations/builtin/credentials/notion/ | Official n8n documentation for Notion API credential configuration                             |
| Label naming consistency between Todoist and the workflow is critical for proper triggering.   | Ensure the label “send-to-notion” exists on your Todoist account and matches node configuration |
| The workflow runs every second; consider modifying the schedule interval for rate limiting.    | High frequency triggers may cause API throttling or performance issues                          |
| Notion database ID must be correctly set and accessible by the API credential used.            | Verify Notion database permissions and ID before using                                        |

---

This comprehensive documentation enables both human users and automation agents to understand, reproduce, and modify the “Sync tasks automatically from Todoist to Notion” workflow efficiently while anticipating common issues.