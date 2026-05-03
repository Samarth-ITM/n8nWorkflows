Automated Execution Cleanup System with n8n API and Custom Retention Rules

https://n8nworkflows.xyz/workflows/automated-execution-cleanup-system-with-n8n-api-and-custom-retention-rules-6833


# Automated Execution Cleanup System with n8n API and Custom Retention Rules

### 1. Workflow Overview

This workflow automates the cleanup of old workflow executions in an n8n instance by retaining only the most recent executions per workflow. Its primary purpose is to optimize n8n instance performance and reduce storage/database bloat by deleting obsolete execution records while preserving a configurable number of recent runs.

The workflow is structured into the following logical blocks:

- **1.1 Schedule Trigger:** Defines the frequency at which the cleanup process runs automatically.
- **1.2 Fetch Executions:** Retrieves recent workflow executions from the n8n instance via API.
- **1.3 Define Retention Count:** Sets how many recent executions per workflow should be kept.
- **1.4 Processing & Filtering:** Groups executions by workflow, sorts them by recency, and selects executions eligible for deletion.
- **1.5 Deletion of Old Executions:** Deletes the filtered old executions via the n8n API.

Each block uses official n8n nodes only, making the workflow compatible with both self-hosted and n8n Cloud environments.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block schedules the workflow to run automatically once per day at 2 AM, ensuring regular cleanup without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: `n8n-nodes-base.scheduleTrigger`  
    - Configuration: Triggers once daily at 2:00 AM (triggerAtHour: 2).  
    - Inputs: None (entry point).  
    - Outputs: Connected to "Get many executions".  
    - Version: 1.2  
    - Edge Cases: Scheduling misconfiguration could cause no runs or excessive runs; relies on n8n’s scheduler accuracy.  
    - Notes: Configurable to different intervals as needed.

#### 2.2 Fetch Executions

- **Overview:**  
  Retrieves up to 250 of the most recent executions from the n8n instance via the internal API, including metadata like workflow ID and execution timestamps.

- **Nodes Involved:**  
  - Get many executions

- **Node Details:**  

  - **Get many executions**  
    - Type: `n8n-nodes-base.n8n` (API node)  
    - Configuration:  
      - Resource: `execution`  
      - Operation: `get many executions`  
      - Limit: 250 executions (maximum allowed by n8n API)  
      - Filters: None (fetches all recent executions regardless of status)  
      - Options: `activeWorkflows` set to false (does not filter to only active workflows)  
    - Credentials: Uses an n8n API credential with a personal API key and base URL configured for the instance.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Passes data to "Set Executions to Keep".  
    - Version: 1  
    - Edge Cases: API key issues (auth error), network timeouts, API limits.

#### 2.3 Define Retention Count

- **Overview:**  
  Defines the number of recent executions per workflow to retain. This number is configurable and can be set to zero to delete all past executions.

- **Nodes Involved:**  
  - Set Executions to Keep

- **Node Details:**  

  - **Set Executions to Keep**  
    - Type: `n8n-nodes-base.set`  
    - Configuration:  
      - Sets variable `executionsToKeep` to 10 by default.  
      - Includes other fields passed from previous node (pass-through).  
    - Inputs: Receives executions list from "Get many executions".  
    - Outputs: Passes data and variable to "Code" node.  
    - Version: 3.4  
    - Edge Cases: If `executionsToKeep` is invalid (e.g., negative), may cause unexpected behavior downstream.

#### 2.4 Processing & Filtering

- **Overview:**  
  Processes the fetched executions: filters only valid executions (those with workflowId and startedAt), groups them by workflow, sorts each group by start time descending, and determines which executions exceed the retention count to delete.

- **Nodes Involved:**  
  - Code

- **Node Details:**  

  - **Code**  
    - Type: `n8n-nodes-base.code`  
    - Configuration:  
      - Custom JavaScript code that:  
        - Reads `executionsToKeep` from input or defaults to 10.  
        - Filters executions to those with valid `workflowId` and `startedAt`.  
        - Groups executions by `workflowId`.  
        - Sorts each group by descending `startedAt` date (most recent first).  
        - Selects executions beyond the `executionsToKeep` threshold for deletion.  
      - Returns items representing executions to delete.  
    - Inputs: Receives executions and `executionsToKeep` variable.  
    - Outputs: Passes executions to delete to "Delete an execution" node.  
    - Version: 2  
    - Edge Cases:  
      - Executions missing workflowId or startedAt are ignored.  
      - Date parsing errors if timestamps are malformed.  
      - Large data sets beyond 250 executions are not handled (API limit).  
      - If `executionsToKeep` is 0, all executions will be marked for deletion.  
    - Notes: Running executions are not explicitly excluded but will be older and likely preserved due to sorting logic.

#### 2.5 Deletion of Old Executions

- **Overview:**  
  Deletes each execution identified by the Code node individually using the n8n API.

- **Nodes Involved:**  
  - Delete an execution

- **Node Details:**  

  - **Delete an execution**  
    - Type: `n8n-nodes-base.n8n` (API node)  
    - Configuration:  
      - Resource: `execution`  
      - Operation: `delete`  
      - Execution ID: Dynamically set via expression `{{$json.id}}` from input.  
    - Credentials: Same n8n API credential as "Get many executions".  
    - Inputs: Receives executions to delete from the Code node.  
    - Outputs: None (terminal node).  
    - Version: 1  
    - Edge Cases:  
      - Deleted execution may not exist (already deleted), causing API errors.  
      - API rate limits or timeouts may interrupt deletion.  
      - Missing or invalid API credentials cause auth failures.  

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                       | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                                               |
|---------------------|-------------------------------|------------------------------------|-----------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | n8n-nodes-base.scheduleTrigger| Triggers workflow execution daily  | None                  | Get many executions       | # 🕒 Schedule Trigger - Defines frequency (daily 2 AM). [Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger) |
| Get many executions  | n8n-nodes-base.n8n            | Fetches recent executions via API  | Schedule Trigger      | Set Executions to Keep    | # 📥 Get Many Executions - Fetch recent executions (limit 250). [Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n#get-many-executions) |
| Set Executions to Keep| n8n-nodes-base.set            | Defines how many executions to keep| Get many executions   | Code                     | # 🛠️ Set Executions to Keep - Configures retention count (default 10). [Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set) |
| Code                | n8n-nodes-base.code            | Filters executions to delete       | Set Executions to Keep| Delete an execution       | # 🧠 Code Node - Groups, sorts, filters executions for deletion. [Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code) |
| Delete an execution  | n8n-nodes-base.n8n            | Deletes executions via API          | Code                  | None                     | # 🗑️ Delete Many Executions - Deletes outdated executions. [Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n#delete-execution) |

Additional Sticky Notes applying to the entire workflow:  
- # 🧹 Automatically Clean Up Old Executions in n8n (Keep Only the Most Recent Per Workflow) - Detailed description, setup instructions, and warnings.  
- # ❗ This workflow deletes data. Always test in staging before enabling in production.

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node:**  
   - Add node: `Schedule Trigger`  
   - Set trigger interval to daily at 2 AM (triggerAtHour: 2)  
   - Position as entry node.

2. **Add the Get Many Executions Node:**  
   - Add node: `n8n` node (API node)  
   - Configure:  
     - Resource: `execution`  
     - Operation: `get many executions`  
     - Limit: 250  
     - Filters: none  
     - Options: `activeWorkflows` = false  
   - Connect input from `Schedule Trigger`.  
   - Set credentials: Create new n8n API credentials:  
     - Name: e.g., “Internal API Access”  
     - API Key: Personal API Key generated in n8n instance (Settings → API Keys)  
     - Base URL: Full n8n instance API URL (e.g., https://your-n8n-instance.com/api/v1)  
   - Assign this credential to this node.

3. **Create the Set Executions to Keep Node:**  
   - Add node: `Set` node  
   - Add assignment:  
     - Variable: `executionsToKeep` (type Number)  
     - Value: 10 (default, adjust as needed)  
   - Enable "Include Other Fields" to pass through executions data.  
   - Connect input from `Get many executions`.

4. **Add the Code Node:**  
   - Add node: `Code` node  
   - Paste the following JavaScript code:

   ```javascript
   const executionsToKeep = $input.first().json.executionsToKeep ?? 10;

   // Filter valid executions with workflowId and startedAt
   const validExecutions = items.filter(item =>
     item.json.workflowId && item.json.startedAt
   );

   // Group executions by workflowId
   const grouped = {};
   for (const item of validExecutions) {
     const wfId = item.json.workflowId;
     if (!grouped[wfId]) grouped[wfId] = [];
     grouped[wfId].push(item);
   }

   // Sort each group by startedAt descending (most recent first)
   for (const wfId in grouped) {
     grouped[wfId].sort((a, b) =>
       new Date(b.json.startedAt) - new Date(a.json.startedAt)
     );
   }

   // Select executions to delete (exceeding executionsToKeep)
   let toDelete = [];
   for (const wfId in grouped) {
     const executions = grouped[wfId];
     if (executions.length > executionsToKeep) {
       toDelete = toDelete.concat(executions.slice(executionsToKeep));
     }
   }

   return toDelete;
   ```
   - Connect input from `Set Executions to Keep`.

5. **Add the Delete an Execution Node:**  
   - Add node: `n8n` node (API node)  
   - Configure:  
     - Resource: `execution`  
     - Operation: `delete`  
     - Execution ID: Use expression `{{$json.id}}` to delete each execution received  
   - Connect input from `Code` node.  
   - Use the same n8n API credentials as the "Get many executions" node.

6. **Final Steps:**  
   - Save the workflow.  
   - Test in a staging environment with a small number of executions or adjust `executionsToKeep` to a high number to avoid accidental deletions.  
   - Enable the workflow for automatic runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                     | Context or Link                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow deletes execution data. Always test in a non-production or staging environment before enabling it in production to avoid accidental data loss.                                                                                                  | Warning sticky note in workflow                                                                                             |
| To create the API credential, generate a Personal API Key in n8n under Settings → API Keys, then configure the n8n API credential with this key and the API base URL.                                                                                          | Setup instructions in sticky note                                                                                           |
| The workflow uses only official n8n nodes and requires no external databases or custom integrations. Compatible with both n8n Cloud and self-hosted instances.                                                                                                  | Workflow overview and sticky notes                                                                                          |
| Documentation links for nodes used:  
- Schedule Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger  
- Get Many Executions & Delete Execution (n8n API node): https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n  
- Set Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set  
- Code Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code | Provided in sticky notes for user reference                                                               |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.