Automatically document n8n workflows directly in Notion database

https://n8nworkflows.xyz/workflows/automatically-document-n8n-workflows-directly-in-notion-database-2151


# Automatically document n8n workflows directly in Notion database

### 1. Workflow Overview

This workflow automates the synchronization of n8n workflows tagged with `sync-to-notion` into a dedicated Notion database for documentation and tracking. It periodically fetches all workflows with the specified tag, processes their metadata, queries Notion to check if a corresponding page exists, then either creates a new page or updates the existing one accordingly.

**Target use cases:**  
- Automated documentation and ownership tracking of n8n workflows  
- Centralizing workflow metadata in Notion for easy access and maintenance  
- Ensuring that any workflow tagged for sync is reflected in the Notion database without manual intervention

**Logical blocks:**

- **1.1 Scheduling & Workflow Retrieval:** Periodically triggers the workflow and fetches all workflows tagged `sync-to-notion`.  
- **1.2 Data Preparation:** Extracts and formats relevant workflow metadata, including status, URLs, and timestamps.  
- **1.3 Notion Query:** Checks if the workflow is already documented in the Notion database by querying with a unique environment ID.  
- **1.4 Decision Making:** Determines whether to create a new Notion page (for newly added workflows) or update an existing one.  
- **1.5 Notion Update:** Executes the creation or update operation in the Notion database.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Workflow Retrieval

- **Overview:** This block triggers every 15 minutes and fetches all n8n workflows tagged `sync-to-notion` via the n8n API.  
- **Nodes Involved:**  
  - Every 15 minutes  
  - Get all workflows with tag  

##### Node: Every 15 minutes  
- Type: Schedule Trigger  
- Role: Initiates the workflow execution every 15 minutes automatically.  
- Configuration: Interval set to 15 minutes.  
- Input: None (trigger).  
- Output: Triggers the next node.  
- Edge cases: Workflow may not trigger if n8n is down or scheduling misconfigured.

##### Node: Get all workflows with tag  
- Type: n8n API node (n8n-nodes-base.n8n)  
- Role: Calls n8n internal API to list workflows filtered by the tag `sync-to-notion`.  
- Configuration: Filter set to tag = `sync-to-notion`.  
- Credentials: Uses `n8n admin account` credentials for API access.  
- Input: Trigger from schedule node.  
- Output: List of workflows matching the tag.  
- Edge cases: API auth failures, no workflows found, or network issues.

---

#### 1.2 Data Preparation

- **Overview:** Prepares and formats workflows’ metadata for Notion insertion, including setting URLs, status flags, and timestamps.  
- **Nodes Involved:**  
  - Set fields  

##### Node: Set fields  
- Type: Set node  
- Role: Extracts and assigns fields from the workflow JSON to a simplified structure for Notion.  
- Configuration:  
  - `active` boolean from workflow’s active status.  
  - `url` constructed using environment variable `instance_url` plus workflow path.  
  - `errorWorkflow` boolean indicating if the workflow is an error workflow.  
  - `name`, `updatedAt`, `createdAt` copied as strings.  
  - `envId` constructed as `internal-<workflow_id>` (unique identifier).  
- Expressions: Uses `$json` to access workflow data and `$vars.instance_url` environment variable.  
- Input: Workflow objects from previous node.  
- Output: Prepared workflow metadata objects for Notion query.  
- Edge cases: Missing environment variable `instance_url`, malformed workflow data, expression evaluation errors.

---

#### 1.3 Notion Query

- **Overview:** Queries the Notion database to check if a page already exists for each workflow by filtering on the unique `env id` field.  
- **Nodes Involved:**  
  - Get notion page with workflow id  
  - Map fields  

##### Node: Get notion page with workflow id  
- Type: HTTP Request  
- Role: Sends a POST request to Notion API to query the database filtering on `env id` containing the workflow’s `envId`.  
- Configuration:  
  - URL set to Notion database query endpoint.  
  - Body contains filter object with `env id` property filter, dynamically set using expression `{{ $json.envId }}`.  
  - Predefined credential type set to `notionApi`.  
  - Header sets `Notion-Version` to `2022-06-28`.  
- Credentials: Uses configured Notion API credentials.  
- Input: Workflow metadata including `envId`.  
- Output: Query result with pages matching the `envId`.  
- Edge cases: Notion API rate limits, invalid or expired credentials, incorrect database ID, malformed query.

##### Node: Map fields  
- Type: Set node  
- Role: Wraps the previously set fields into an `input` object for downstream usage, preserving other fields.  
- Configuration: Assigns `input` to the JSON from `Set fields`.  
- Input: Response from Notion query.  
- Output: Object containing both Notion query results and original prepared workflow metadata.  
- Edge cases: Expression evaluation errors; missing input data.

---

#### 1.4 Decision Making

- **Overview:** Determines whether the workflow is newly added (no existing Notion page) or already documented, branching the workflow accordingly.  
- **Nodes Involved:**  
  - if newly added workflow  

##### Node: if newly added workflow  
- Type: If node  
- Role: Checks if the Notion query results array is empty, indicating the workflow is not yet documented.  
- Configuration: Condition tests if `{{$json.results}}` array is empty.  
- Input: Output from `Map fields`.  
- Output: Two branches:  
  - True (empty results) → create new Notion page  
  - False (existing page) → update Notion page  
- Edge cases: Unexpected query response structure, false negatives if Notion data lags.

---

#### 1.5 Notion Update

- **Overview:** Based on the decision, either adds a new page to the Notion database or updates an existing page with the latest workflow metadata.  
- **Nodes Involved:**  
  - Add to Notion  
  - Update in Notion  

##### Node: Add to Notion  
- Type: Notion node  
- Role: Creates a new page in the Notion database with workflow metadata.  
- Configuration:  
  - Database ID set to the configured Notion database.  
  - Properties mapped for: `env id`, `URL (dev)`, `isActive (dev)`, `Workflow created at`, `Workflow updated at`, and `Error workflow setup`.  
  - Title set to workflow name.  
- Credentials: Notion API credentials.  
- Input: Workflow data from `if newly added workflow` true branch.  
- Output: Newly created page info.  
- Edge cases: API errors, permission issues, incorrect property mappings.

##### Node: Update in Notion  
- Type: Notion node  
- Role: Updates the existing Notion page with refreshed workflow metadata.  
- Configuration:  
  - Page ID extracted from the first result’s URL of the Notion query.  
  - Properties updated similarly to the add node, except `Error workflow setup` is set to false explicitly.  
- Credentials: Notion API credentials.  
- Input: Workflow data from `if newly added workflow` false branch.  
- Output: Updated page info.  
- Edge cases: Page ID extraction errors, API errors, stale data overwriting.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                          | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                             |
|-----------------------------|--------------------------------|----------------------------------------|----------------------------|--------------------------------|-------------------------------------------------------------------------------------------------------|
| Every 15 minutes            | Schedule Trigger               | Triggers workflow every 15 minutes     | -                          | Get all workflows with tag      |                                                                                                       |
| Get all workflows with tag  | n8n API node                   | Fetches workflows with tag `sync-to-notion` | Every 15 minutes           | Set fields                     |                                                                                                       |
| Set fields                 | Set                           | Prepares workflow metadata for Notion  | Get all workflows with tag  | Get notion page with workflow id | ### 👆 Set instance url instead of `{{ $vars.instance_url }}` (or set the env variable if you have that feature) |
| Get notion page with workflow id | HTTP Request               | Queries Notion database for existing page | Set fields                 | Map fields                    |                                                                                                       |
| Map fields                 | Set                           | Wraps prepared data for decision making | Get notion page with workflow id | if newly added workflow        |                                                                                                       |
| if newly added workflow    | If                            | Branches logic to add or update Notion page | Map fields                 | Add to Notion, Update in Notion |                                                                                                       |
| Add to Notion              | Notion                        | Creates new page in Notion database    | if newly added workflow (true) | -                            |                                                                                                       |
| Update in Notion           | Notion                        | Updates existing Notion page            | if newly added workflow (false) | -                            |                                                                                                       |
| Sticky Note3               | Sticky Note                   | Setup instructions                      | -                          | -                              | ### 👨‍🎤 Setup 1. Add your n8n api creds 2. Add your notion creds 3. create notion database with fields `env id` as `text`, `isActive (dev)` as `boolean`, `URL (dev)` as `url`, `Workflow created at` as `date`, `Workflow updated at` as `date`, `Error workflow setup` as `boolean` 4. Add tag `sync-to-notion` to some workflows |
| Sticky Note2               | Sticky Note                   | Instruction on instance URL setting     | -                          | -                              | ### 👆 Set instance url instead of `{{ $vars.instance_url }}` (or set the env variable if you have that feature) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to every 15 minutes.  
   - No credentials needed.

2. **Create an n8n API Node to Fetch Workflows:**  
   - Type: n8n (API) node  
   - Credentials: Use an n8n admin account credential with API access.  
   - Parameters: Set filter to fetch workflows tagged `sync-to-notion`.  
   - Connect output of Schedule Trigger node to this node.

3. **Create a Set Node to Prepare Workflow Fields:**  
   - Type: Set  
   - Parameters:  
     - Assign `active` as boolean from `$json.active`.  
     - Assign `url` as string: concatenate your instance URL environment variable (e.g., `$vars.instance_url`) with `workflow/` and the workflow ID (`$json.id`).  
     - Assign `errorWorkflow` as boolean from `$json.settings?.errorWorkflow`.  
     - Assign `name` as string from `$json.name`.  
     - Assign `updatedAt` and `createdAt` as strings from respective workflow fields.  
     - Assign `envId` as string constructed as `internal-` + workflow ID.  
   - Connect output of n8n API node to this node.

4. **Create HTTP Request Node to Query Notion:**  
   - Type: HTTP Request  
   - Credentials: Use Notion API credentials.  
   - Parameters:  
     - Method: POST  
     - URL: `https://api.notion.com/v1/databases/<YourDatabaseID>/query` (replace with your Notion database ID).  
     - Headers: Include `Notion-Version: 2022-06-28`.  
     - Body (JSON): Filter on property `env id` with rich_text contains the expression `{{ $json.envId }}`.  
     - Authentication: Predefined credential type `notionApi`.  
   - Connect output of Set fields node to this node.

5. **Create a Set Node to Map Fields:**  
   - Type: Set  
   - Parameters: Assign a new field `input` containing the JSON from the previous Set fields node (e.g., `={{ $('Set fields').item.json }}`) to pass along.  
   - Connect output of HTTP Request node to this node.

6. **Create an If Node to Check if Workflow is New:**  
   - Type: If  
   - Condition: Check if the Notion query results array `$json.results` is empty. Use the "is empty" array condition on `{{$json.results}}`.  
   - Connect output of Map fields node to this node.

7. **Create a Notion Node to Add New Page:**  
   - Type: Notion  
   - Credentials: Use Notion API credentials.  
   - Parameters:  
     - Resource: databasePage  
     - Database ID: Your Notion database ID.  
     - Title: Set to `{{$json.input.name}}`.  
     - Properties: Map the following fields to Notion properties:  
       - `env id` (rich_text) ← `{{$json.input.envId}}`  
       - `URL (dev)` (url) ← `{{$json.input.url}}`  
       - `isActive (dev)` (checkbox) ← `{{$json.input.active}}`  
       - `Workflow created at` (date) ← `{{$json.input.createdAt}}`  
       - `Workflow updated at` (date) ← `{{$json.input.updatedAt}}`  
       - `Error workflow setup` (checkbox) ← `{{$json.input.errorWorkflow}}`  
   - Connect the true output of the If node to this node.

8. **Create a Notion Node to Update Existing Page:**  
   - Type: Notion  
   - Credentials: Notion API credentials.  
   - Parameters:  
     - Resource: databasePage  
     - Operation: update  
     - Page ID: Extract from the first result URL in Notion query results: `{{$json.results[0].url}}` (ensure to convert URL properly to page ID if needed).  
     - Properties to update:  
       - `Name` (title) ← `{{$json.input.name}}`  
       - `isActive (dev)` (checkbox) ← `{{$json.input.active}}`  
       - `URL (dev)` (url) ← `{{$json.input.url}}`  
       - `Workflow updated at` (date) ← `{{$json.input.updatedAt}}`  
       - `Error workflow setup` (checkbox) ← `false` (hardcoded)  
   - Connect the false output of the If node to this node.

9. **(Optional) Add Sticky Note Nodes:**  
   - Add sticky notes for setup instructions and environment variable usage to improve maintainability and onboarding.

10. **Set Workflow Variables and Credentials:**  
    - Define environment variable `instance_url` with your n8n instance URL (e.g., `https://your-n8n-instance.com/`).  
    - Configure credentials for:  
      - n8n API with admin rights  
      - Notion API with access to the target database

11. **Create Notion Database:**  
    - Create a Notion database with the following properties:  
      - `env id` (Text)  
      - `isActive (dev)` (Checkbox)  
      - `URL (dev)` (URL)  
      - `Workflow created at` (Date)  
      - `Workflow updated at` (Date)  
      - `Error workflow setup` (Checkbox)  
    - Ensure the integration has access permissions to the database.

12. **Tag Your Workflows:**  
    - Add the tag `sync-to-notion` to workflows you want to be documented in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Use environment variable `instance_url` to set your n8n instance URL instead of hardcoding URLs in the workflow.                        | Sticky Note2 in workflow                                                                            |
| Setup instructions include adding appropriate API credentials and creating the Notion database with required fields and permissions.    | Sticky Note3 in workflow                                                                            |
| Tags are used to filter workflows that should be synced; only workflows tagged `sync-to-notion` will be processed.                      | Workflow description section                                                                       |
| Notion API version used is `2022-06-28`. Ensure your credentials and database support this version to avoid compatibility issues.       | HTTP Request node headers                                                                          |
| Official n8n documentation and Notion API docs are good resources for credential setup and API usage details.                         | General recommendation                                                                             |

---

This detailed structured reference enables advanced users and automation agents to understand, reproduce, and extend the workflow while anticipating integration challenges and edge cases.