Watchdog: Update All Workflows With Default Error Workflow

https://n8nworkflows.xyz/workflows/watchdog--update-all-workflows-with-default-error-workflow-2169


# Watchdog: Update All Workflows With Default Error Workflow

### 1. Workflow Overview

This workflow automates the management of default error workflows across all n8n workflows in an instance. Its main purpose is to prevent users from forgetting to assign a default error workflow when creating or updating workflows, which is essential for robust error handling and avoiding silent failures.

**Target Use Cases:**
- Organizations or users who manage multiple workflows and want consistent error handling.
- Ensuring no workflow is left without a default error workflow unless explicitly excluded.
- Avoiding recursive error loops by excluding certain workflows tagged to prevent default error workflow assignment.

**Logical Blocks:**

- **1.1 Trigger Block:** Initiates the workflow either on a schedule (every 4 hours) or manually for testing.
- **1.2 Variable Initialization Block:** Sets critical variables such as the default error workflow ID and the exclusion tag.
- **1.3 Workflow Retrieval Block:** Fetches all workflows from the n8n instance via API.
- **1.4 Filtering Block:** Excludes workflows that have the exclusion tag or already have the correct default error workflow set.
- **1.5 Update Block:** Updates the remaining workflows by setting their default error workflow to the specified one in the variables.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** This block starts the workflow execution either automatically on a schedule (every 4 hours) or manually via a trigger node for testing purposes.
- **Nodes Involved:** 
  - Schedule Trigger
  - When clicking "Test workflow"

- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Automatically triggers the workflow every 4 hours.
    - Configuration: Interval set to 4 hours.
    - Input/Output: No input; triggers output to "Set Vars" node.
    - Edge Cases: Workflow runs even if no updates required; no direct failure expected.

  - **When clicking "Test workflow"**
    - Type: Manual Trigger
    - Role: Allows manual execution for testing.
    - Configuration: No parameters.
    - Input/Output: No input; triggers output to "Set Vars" node.
    - Edge Cases: Useful for debugging or manual runs; no failure expected.

#### 2.2 Variable Initialization Block

- **Overview:** Defines and assigns key variables used throughout the workflow, including the ID of the default error workflow and the tag used to exclude workflows from updates.
- **Nodes Involved:** 
  - Set Vars

- **Node Details:**

  - **Set Vars**
    - Type: Set
    - Role: Holds configurable values for the default error workflow ID and exclusion tag.
    - Configuration:
      - `default_error_workflow_id`: String representing the target workflow ID to set as the default error workflow.
      - `default_error_exclusion_tag`: String representing the tag name used to exclude workflows from having their default error workflow set.
    - Key Expressions: Variables are referenced dynamically throughout the workflow via expressions like `$('Set Vars').item.json.default_error_workflow_id`.
    - Input/Output: Receives trigger from either trigger node; outputs to "Get All Workflows" node.
    - Edge Cases: Incorrect values here will cause wrong updates or no updates; user must customize this node after import.

#### 2.3 Workflow Retrieval Block

- **Overview:** Fetches all workflows present in the n8n instance to analyze and determine which need updating.
- **Nodes Involved:** 
  - Get All Workflows

- **Node Details:**

  - **Get All Workflows**
    - Type: n8n Node (n8n API)
    - Role: Retrieves a list of all workflows via n8n API.
    - Configuration: No filters; fetches all workflows.
    - Credentials: Requires valid n8n API credential configured.
    - Input/Output: Input from "Set Vars"; output to filtering node.
    - Edge Cases: API authentication failure, network issues, or permission errors could prevent proper retrieval.

#### 2.4 Filtering Block

- **Overview:** Filters out workflows that are either tagged with the exclusion tag or already have the default error workflow set correctly, to avoid redundant or recursive updates.
- **Nodes Involved:** 
  - Exclude default_error:false Tagged Workflows (Filter node)

- **Node Details:**

  - **Exclude default_error:false Tagged Workflows**
    - Type: Filter
    - Role: Removes workflows from the list that:
      - Have the exclusion tag (`default_error:false` by default).
      - Already have the default error workflow set to the desired workflow.
    - Configuration:
      - Conditions:
        - Checks if `tags` array includes the exclusion tag using an expression.
        - Checks if `settings.errorWorkflow` is not equal to the default error workflow ID.
    - Input/Output: Input from "Get All Workflows"; output to "Set Default Error Workflow".
    - Edge Cases:
      - Workflows without `tags` or with missing `settings.errorWorkflow` fields.
      - Expression evaluation errors if data structures differ.
      - Case sensitivity in tag matching is enabled.

#### 2.5 Update Block

- **Overview:** Updates each filtered workflow’s settings to assign the default error workflow ID, writing changes back into the PostgreSQL database.
- **Nodes Involved:** 
  - Set Default Error Workflow

- **Node Details:**

  - **Set Default Error Workflow**
    - Type: PostgreSQL
    - Role: Performs an update operation on the `workflow_entity` table to set the `errorWorkflow` property in the `settings` JSON for each workflow.
    - Configuration:
      - Updates the `settings` column by merging the current settings with `errorWorkflow` set to the default error workflow ID.
      - Matches rows by workflow `id`.
    - Credentials: Requires PostgreSQL credentials with access to the n8n database.
    - Input/Output: Receives filtered workflows from the filter node; outputs updated workflow data.
    - Edge Cases:
      - Database connection or authentication errors.
      - JSON parsing or stringifying errors.
      - Potential race conditions if workflows are updated concurrently elsewhere.
      - Must exclude the default error workflow itself from updates (achieved by tagging it with `default_error:false`).

---

### 3. Summary Table

| Node Name                               | Node Type          | Functional Role                        | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                              |
|----------------------------------------|--------------------|-------------------------------------|----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger                       | Schedule Trigger   | Periodic start every 4 hours         |                                  | Set Vars                          |                                                                                                         |
| When clicking "Test workflow"          | Manual Trigger     | Manual start for testing              |                                  | Set Vars                          |                                                                                                         |
| Set Vars                              | Set                | Define default error workflow ID and exclusion tag | Schedule Trigger, When clicking "Test workflow" | Get All Workflows                 | Edit this node after import to set your default_error_workflow_id and exclusion tag (default_error:false)|
| Get All Workflows                     | n8n API Node       | Retrieve all workflows                | Set Vars                         | Exclude default_error:false Tagged Workflows | Requires n8n API credentials                                                                             |
| Exclude default_error:false Tagged Workflows | Filter             | Filter workflows to exclude those with exclusion tag or already set error workflow | Get All Workflows                | Set Default Error Workflow        |                                                                                                         |
| Set Default Error Workflow             | PostgreSQL         | Update workflows in DB with default error workflow | Exclude default_error:false Tagged Workflows |                                   | Requires PostgreSQL credentials to access n8n DB; update `settings.errorWorkflow` field                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node:
     - Set interval to every 4 hours.
   - Add a **Manual Trigger** node for testing purposes.

2. **Create Variable Initialization Node:**
   - Add a **Set** node named "Set Vars".
   - Add two variables:
     - `default_error_workflow_id` (String): Set this to the workflow ID you want as the default error workflow.
     - `default_error_exclusion_tag` (String): Set this to `"default_error:false"` (or your preferred exclusion tag).
   - Connect both trigger nodes to this "Set Vars" node.

3. **Create Workflow Retrieval Node:**
   - Add an **n8n API** node named "Get All Workflows".
   - Select operation to list all workflows (no filters).
   - Configure credentials with a valid n8n API credential.
   - Connect "Set Vars" output to this node.

4. **Create Filtering Node:**
   - Add a **Filter** node named "Exclude default_error:false Tagged Workflows".
   - Setup conditions:
     - Condition 1 (Boolean): Check if the workflow’s `tags` array contains the exclusion tag. Use the expression:
       ```
       {{$json["tags"]?.some(item => item.name === $node["Set Vars"].json["default_error_exclusion_tag"])}}
       ```
       This should be `false` to pass.
     - Condition 2 (String): Check if `settings.errorWorkflow` is not equal to `default_error_workflow_id`.
       Use expression:
       ```
       {{$json["settings"]?.errorWorkflow || ""}} !== $node["Set Vars"].json["default_error_workflow_id"]
       ```
   - Connect "Get All Workflows" output to this filter node.

5. **Create Update Node:**
   - Add a **PostgreSQL** node named "Set Default Error Workflow".
   - Set operation to "Update".
   - Configure to update the table `workflow_entity` in schema `public`.
   - Match rows by workflow `id`.
   - Update the `settings` column by merging existing JSON with `errorWorkflow` set to the variable:
     - Use an expression to stringify the merged settings:
       ```
       JSON.stringify({ ...$json.settings, errorWorkflow: $node["Set Vars"].json["default_error_workflow_id"] })
       ```
   - Configure PostgreSQL credentials with access to your n8n database.
   - Connect output of filter node to this node.

6. **Connections Recap:**
   - Schedule Trigger → Set Vars
   - Manual Trigger → Set Vars
   - Set Vars → Get All Workflows
   - Get All Workflows → Exclude default_error:false Tagged Workflows
   - Exclude default_error:false Tagged Workflows → Set Default Error Workflow

7. **Post-Setup Notes:**
   - Edit the "Set Vars" node to your actual default error workflow ID and exclusion tag.
   - Ensure the exclusion tag is applied to the default error workflow itself to avoid recursive updates.
   - Verify PostgreSQL credentials have sufficient permissions.
   - Test manually using the manual trigger before relying on the schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Make sure your default error workflow has the tag `default_error:false` (or your chosen exclusion tag) set to prevent recursive loops. | Important to avoid infinite update recursion during error handling.                                     |
| Workflow scans all workflows every 4 hours to keep error handling consistent across your n8n instance.           | Operational detail for maintenance frequency.                                                          |
| The workflow requires PostgreSQL access to the n8n internal database to update workflow settings directly.       | Ensure database credentials are securely configured.                                                   |
| For API access, configure an n8n API credential with sufficient permissions to fetch all workflows.               | Related to the "Get All Workflows" node.                                                               |