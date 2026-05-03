Reconnect migrated workflows and datatables between n8n instances

https://n8nworkflows.xyz/workflows/reconnect-migrated-workflows-and-datatables-between-n8n-instances-14218


# Reconnect migrated workflows and datatables between n8n instances

# 1. Workflow Overview

This workflow is a migration repair utility for n8n. Its purpose is to reconnect references to **sub-workflows** and **data tables** after those assets have been migrated from one n8n instance to another, where internal IDs no longer match.

The workflow compares assets from an **original n8n instance** and a **new n8n instance**, builds a name-based mapping of old IDs to new IDs, scans the migrated workflows in the new instance, replaces outdated IDs inside relevant node parameters, and then updates those workflows back into the new instance through the n8n API.

## 1.1 Input and Configuration

The workflow starts manually and uses a Code node to define the two base URLs:
- `orig_url`: original/source n8n instance
- `new_url`: destination/target n8n instance

These values are reused by subsequent HTTP Request nodes.

## 1.2 Fetching Source Instance Metadata

The workflow retrieves:
- all data tables from the original instance
- all workflows from the original instance

This data is used to identify the original IDs.

## 1.3 Fetching Destination Instance Metadata

The workflow then retrieves:
- all data tables from the new instance
- all workflows from the new instance

This data is used both to identify replacement IDs and to obtain the workflow JSON that will be rewritten and pushed back.

## 1.4 Entity Mapping

A Code node creates a unified map keyed by asset name. For each matching name across old and new instances, it stores:
- `orig_id`
- `new_id`

This map is built for both data tables and workflows.

## 1.5 Workflow JSON Rewriting

Another Code node walks through the workflows from the new instance and looks for nodes of type:
- `n8n-nodes-base.executeWorkflow`
- `n8n-nodes-base.dataTable`

It recursively scans their parameters and replaces any values matching old IDs with the corresponding new IDs.

It also sanitizes each workflow object before update by removing or resetting fields the update API may reject.

## 1.6 Writing Updated Workflows Back

Finally, the workflow updates each affected workflow in the new instance using the n8n node with operation `update`.

---

# 2. Block-by-Block Analysis

## Block 1: Manual Start and Environment Configuration

### Overview
This block initializes execution and defines the source and target instance URLs. It is the only entry point in the workflow and must be configured before any API call can succeed.

### Nodes Involved
- `When clicking ‘Execute workflow’`
- `config`

### Node Details

#### 1. `When clicking ‘Execute workflow’`
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point used to start the workflow from the editor.
- **Configuration choices:** No parameters are configured.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none
  - Output: `config`
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None at runtime beyond normal manual execution requirements.
- **Sub-workflow reference:** None.

#### 2. `config`
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces a single JSON object containing the two instance URLs.
- **Configuration choices:** The JavaScript defines:
  - `orig_url = ""`
  - `new_url = ""`
  and returns them under `json`.
- **Key expressions or variables used:**
  - `orig_url`
  - `new_url`
- **Input and output connections:**  
  - Input: `When clicking ‘Execute workflow’`
  - Output: `orig datatables`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Empty URL values will break all downstream HTTP calls.
  - Invalid URL formatting can produce request errors.
  - Missing protocol (`https://`) will typically fail.
- **Sub-workflow reference:** None.

---

## Block 2: Fetching Original Instance Assets

### Overview
This block queries the original n8n instance through its API to retrieve all data tables and workflows. These results provide the old IDs that must later be replaced.

### Nodes Involved
- `orig datatables`
- `orig workflows`

### Node Details

#### 3. `orig datatables`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs an authenticated GET request to the original instance’s data tables endpoint.
- **Configuration choices:**
  - URL: `={{ $("config").item.json.orig_url }}/api/v1/data-tables`
  - Authentication: predefined credential
  - Credential type: `n8nApi`
- **Key expressions or variables used:**
  - `$("config").item.json.orig_url`
- **Input and output connections:**  
  - Input: `config`
  - Output: `orig workflows`
- **Version-specific requirements:** `typeVersion: 4.4`
- **Edge cases or potential failure types:**
  - Invalid or missing API credential
  - 401/403 authorization failures
  - Network/DNS/TLS issues
  - Endpoint availability may differ by n8n version or edition
  - If the instance has no data tables, downstream code must still tolerate an empty `data` array
- **Sub-workflow reference:** None.

#### 4. `orig workflows`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Performs an authenticated GET request to retrieve workflows from the original instance.
- **Configuration choices:**
  - URL: `={{ $("config").item.json.orig_url }}/api/v1/workflows`
  - Authentication: predefined credential
  - Credential type: `n8nApi`
- **Key expressions or variables used:**
  - `$("config").item.json.orig_url`
- **Input and output connections:**  
  - Input: `orig datatables`
  - Output: `new datatables`
- **Version-specific requirements:** `typeVersion: 4.4`
- **Edge cases or potential failure types:**
  - Same auth/connectivity issues as above
  - Pagination may be relevant on large instances if the API does not return everything in one call
  - If the API response shape differs from expected `{ data: [...] }`, downstream Code nodes will fail
- **Sub-workflow reference:** None.

---

## Block 3: Fetching New Instance Assets

### Overview
This block retrieves the corresponding data tables and workflows from the destination n8n instance. The workflow later uses this data to build replacement mappings and to update the destination workflows in place.

### Nodes Involved
- `new datatables`
- `new workflows`

### Node Details

#### 5. `new datatables`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Authenticated GET request to the new instance’s data tables endpoint.
- **Configuration choices:**
  - URL: `={{ $("config").item.json.new_url }}/api/v1/data-tables`
  - Authentication: predefined credential
  - Credential type: `n8nApi`
- **Key expressions or variables used:**
  - `$("config").item.json.new_url`
- **Input and output connections:**  
  - Input: `orig workflows`
  - Output: `new workflows`
- **Version-specific requirements:** `typeVersion: 4.4`
- **Edge cases or potential failure types:**
  - Wrong credential accidentally pointing to the original instance
  - Missing feature or unavailable endpoint
  - Empty result sets leading to incomplete ID mapping
- **Sub-workflow reference:** None.

#### 6. `new workflows`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Authenticated GET request to retrieve workflows from the new instance.
- **Configuration choices:**
  - URL: `={{ $("config").item.json.new_url }}/api/v1/workflows`
  - Authentication: predefined credential
  - Credential type: `n8nApi`
- **Key expressions or variables used:**
  - `$("config").item.json.new_url`
- **Input and output connections:**  
  - Input: `new datatables`
  - Output: `map`
- **Version-specific requirements:** `typeVersion: 4.4`
- **Edge cases or potential failure types:**
  - Same auth and connectivity concerns
  - If migrated workflows were renamed, name-based matching in the next block will not work
  - If the API returns partial workflow records rather than full structures, later update logic may not have everything it needs
- **Sub-workflow reference:** None.

---

## Block 4: Build Cross-Instance Entity Map

### Overview
This block creates a single dictionary that maps asset names to their original and new IDs. It merges data tables and workflows into one structure to support later replacement operations.

### Nodes Involved
- `map`

### Node Details

#### 7. `map`
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds a name-keyed entity map combining original/new data table IDs and original/new workflow IDs.
- **Configuration choices:**  
  The code:
  - reads `data` arrays from four previous nodes
  - initializes `existingMap = {}`
  - loops through original data tables and stores `orig_id`
  - loops through new data tables and stores `new_id`
  - loops through original workflows and stores `orig_id`
  - loops through new workflows and stores `new_id`
  - returns `{ entity_map: existingMap }`
- **Key expressions or variables used:**
  - `$('orig datatables').first().json.data`
  - `$('new datatables').first().json.data`
  - `$('orig workflows').first().json.data`
  - `$('new workflows').first().json.data`
  - `entityName = item.name`
  - `orig_id`
  - `new_id`
- **Input and output connections:**  
  - Input: `new workflows`
  - Output: `update flow nodes`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - **Name collisions:** data tables and workflows share the same map. If a workflow and a data table have the same name, one may overwrite the other.
  - **Duplicate names within a category:** if multiple workflows or data tables share a name, the map becomes unreliable.
  - **Renamed assets:** if migrated assets changed names between instances, they will not be matched.
  - **Missing `data` arrays:** code will fail if API responses differ.
  - **Partial map coverage:** entities that exist in only one instance will have only one ID field.
- **Sub-workflow reference:** None.

---

## Block 5: Rewrite IDs in Destination Workflow Definitions

### Overview
This block is the core transformation logic. It converts the name-based entity map into an old-ID → new-ID lookup, scans nodes inside each workflow from the new instance, replaces outdated IDs recursively, and prepares sanitized workflow payloads for update.

### Nodes Involved
- `update flow nodes`

### Node Details

#### 8. `update flow nodes`
- **Type and technical role:** `n8n-nodes-base.code`  
  Iterates over workflows from the new instance, updates nested parameter values for target node types, removes unsupported fields, and emits one item per workflow for the update step.
- **Configuration choices:**  
  The code performs these operations:
  1. Reads:
     - `entityMap` from `map`
     - `newFlows` from `new workflows`
  2. Builds `reverseIdMap` using entities that have both `orig_id` and `new_id`
  3. Targets only nodes with type:
     - `n8n-nodes-base.executeWorkflow`
     - `n8n-nodes-base.dataTable`
  4. Uses recursive function `replaceOldIds(parameters, idMap)` to traverse nested parameter objects
  5. For each workflow:
     - updates targeted node parameters
     - copies the workflow object
     - resets `settings = {}`
     - removes:
       - `id`
       - `createdAt`
       - `updatedAt`
       - `versionId`
       - `active`
     - outputs:
       - `id` = original workflow ID from the new instance
       - `workflow` = stringified updated workflow JSON
- **Key expressions or variables used:**
  - `$("map").first().json.entity_map`
  - `$('new workflows').first().json.data`
  - `reverseIdMap[ids.orig_id.toString()] = ids.new_id.toString()`
  - `targetNodeTypes`
  - `replaceOldIds(parameters, idMap)`
- **Input and output connections:**  
  - Input: `map`
  - Output: `update new workflows`
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**
  - **Over-replacement risk:** recursive replacement checks every string/number parameter in targeted nodes. If some unrelated parameter coincidentally equals an old ID, it may also be replaced.
  - **Type coverage limitation:** only `executeWorkflow` and `dataTable` nodes are scanned. Other node types containing migrated IDs will not be fixed.
  - **Array traversal behavior:** arrays are technically objects in JavaScript, so they will be traversed, which is useful, but still worth noting.
  - **Expression-based IDs:** if IDs are embedded inside longer strings or expressions rather than standalone values, they will not be replaced.
  - **API schema changes:** resetting `settings` to `{}` may remove useful workflow settings if newer versions require preserving them.
  - **Workflow serialization:** `workflow` is sent as a JSON string; this assumes the downstream n8n node accepts stringified workflow objects in this mode.
  - **No filtering of unchanged workflows:** all workflows are emitted for update whether modified or not.
- **Sub-workflow reference:** It updates references to `n8n-nodes-base.executeWorkflow` nodes, which are sub-workflow invocation nodes inside migrated workflows, but this node itself does not invoke a sub-workflow.

---

## Block 6: Update Workflows on the New Instance

### Overview
This block writes the sanitized and rewritten workflow definitions back into the new n8n instance. It updates workflows by their current destination-instance IDs.

### Nodes Involved
- `update new workflows`

### Node Details

#### 9. `update new workflows`
- **Type and technical role:** `n8n-nodes-base.n8n`  
  Uses the n8n node to update existing workflows in the destination instance.
- **Configuration choices:**
  - Operation: `update`
  - Workflow ID: expression mode with `={{ $json.id }}`
  - Workflow object: `={{ $json.workflow }}`
  - Request options: empty/default
- **Key expressions or variables used:**
  - `$json.id`
  - `$json.workflow`
- **Input and output connections:**  
  - Input: `update flow nodes`
  - Output: none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Wrong credential can update the wrong n8n instance
  - Invalid workflow payload can trigger schema/API validation errors
  - Permission issues on workflow update
  - Large batches may hit rate limits or execution time constraints
  - If the n8n node expects an object and not a string in a particular version/configuration, update may fail
- **Sub-workflow reference:** None.

---

## Block 7: Documentation and Operator Guidance

### Overview
These Sticky Note nodes provide operator instructions, warnings, and context. They do not participate in execution but are important for understanding intended use and setup.

### Nodes Involved
- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`
- `Sticky Note5`

### Node Details

#### 10. `Sticky Note`
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the configuration block.
- **Configuration choices:** Contains instructions for setting original and new URLs.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 11. `Sticky Note1`
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for original-instance fetch nodes.
- **Configuration choices:** Explains that original-instance n8n API credentials must be used.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 12. `Sticky Note2`
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for new-instance fetch nodes.
- **Configuration choices:** Explains that new-instance n8n API credentials must be used.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 13. `Sticky Note3`
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the mapping and rewrite logic.
- **Configuration choices:** Summarizes the two core Code nodes.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 14. `Sticky Note4`
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Warning note for the final update step.
- **Configuration choices:** Warns to use the credential for the new instance and be careful.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 15. `Sticky Note5`
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level purpose and operator instructions for the entire workflow.
- **Configuration choices:** Describes recursive ID swapping and setup expectations.
- **Input and output connections:** none
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual start of the migration repair process |  | config | ## Migration Helper - Automated Reconnection for Workflows + Datatables<br>**Purpose**<br>This workflow is designed to automate the re-wiring of all your subworkflows and datatable actions after migrating them to a new n8n instance.<br><br>**What it does**<br>- **Recursive ID Swapping:**<br>Takes a master dictionary of your old IDs -> new IDs, loops through your workflow JSONs, and recursively hunts down and replaces the node parameters. (Saves you from having to know if the ID is nested in an options object or at the root).<br><br>---<br><br>**Instructions**<br>Basically you just need to define the relevant URL’s and setup your n8n API credentials - then VERY CAREFULLY select the right credentials in the right nodes. Full instructions can be found in the sticky notes. |
| config | code | Stores source and destination instance base URLs | When clicking ‘Execute workflow’ | orig datatables | ## Config<br>This node just stores the urls as variables so you only have to update them once.<br><br>**Instructions**<br>- Set the orig_url to be the path to your original n8n install<br>- Set the new_url to be the path to your new n8n install |
| orig datatables | httpRequest | Fetches data tables from the original instance | config | orig workflows | ## Get Originals<br>These nodes grab all datatables and workflows from the original n8n install via the n8n API.<br><br>**Instructions**<br>- You'll need to create an n8n credential for your **original** n8n instance and select that in these 2 nodes. |
| orig workflows | httpRequest | Fetches workflows from the original instance | orig datatables | new datatables | ## Get Originals<br>These nodes grab all datatables and workflows from the original n8n install via the n8n API.<br><br>**Instructions**<br>- You'll need to create an n8n credential for your **original** n8n instance and select that in these 2 nodes. |
| new datatables | httpRequest | Fetches data tables from the destination instance | orig workflows | new workflows | ## Get New<br>These nodes grab all datatables and workflows from the new n8n install via the n8n API.<br><br>**Instructions**<br>- You'll need to create an n8n credential for your **new** n8n instance and select that in these 2 nodes. |
| new workflows | httpRequest | Fetches workflows from the destination instance | new datatables | map | ## Get New<br>These nodes grab all datatables and workflows from the new n8n install via the n8n API.<br><br>**Instructions**<br>- You'll need to create an n8n credential for your **new** n8n instance and select that in these 2 nodes. |
| map | code | Builds a name-based old/new ID mapping across workflows and data tables | new workflows | update flow nodes | ## The Magic<br>1. First node creates an entity map of all the datatables and workflows from both instances and stores the original and new ID.<br>2. Second node then goes through all your new workflows looking for any datatable or sub-workflow executions and then rewrites the original ID to be the new ID. |
| update flow nodes | code | Rewrites outdated IDs inside migrated workflow node parameters and sanitizes workflow payloads | map | update new workflows | ## The Magic<br>1. First node creates an entity map of all the datatables and workflows from both instances and stores the original and new ID.<br>2. Second node then goes through all your new workflows looking for any datatable or sub-workflow executions and then rewrites the original ID to be the new ID. |
| update new workflows | n8n | Updates workflows in the destination instance with rewritten JSON | update flow nodes |  | ## Update<br>This node will now update each of the new workflows with the updated JSON.<br><br>**Instructions**<br>- You'll need to select the n8n credential for you **NEW** n8n instance.  ***BE CAREFUL!!!*** |
| Sticky Note | stickyNote | Visual documentation for configuration |  |  |  |
| Sticky Note1 | stickyNote | Visual documentation for source-instance fetch |  |  |  |
| Sticky Note2 | stickyNote | Visual documentation for destination-instance fetch |  |  |  |
| Sticky Note3 | stickyNote | Visual documentation for mapping/update logic |  |  |  |
| Sticky Note4 | stickyNote | Visual documentation for final update warning |  |  |  |
| Sticky Note5 | stickyNote | High-level workflow description |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `util.UpdateMigratedWorkflows`.
   - Keep it inactive while building and testing.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name it: `When clicking ‘Execute workflow’`

3. **Add a Code node for configuration**
   - Node type: `Code`
   - Name it: `config`
   - Connect `When clicking ‘Execute workflow’` → `config`
   - Paste code equivalent to:
     - define `orig_url`
     - define `new_url`
     - return them in `json`
   - Example structure:
     - `orig_url`: source/original n8n base URL
     - `new_url`: destination/new n8n base URL
   - Use full base URLs such as `https://your-instance.example.com`

4. **Create credentials for the original n8n instance**
   - Credential type: `n8n API`
   - Configure it with the original instance’s API access details.
   - Confirm it can access:
     - `/api/v1/workflows`
     - `/api/v1/data-tables`

5. **Create credentials for the new n8n instance**
   - Credential type: `n8n API`
   - Configure it with the destination instance’s API access details.
   - This credential will also be used for updating workflows.
   - Be careful not to swap source and destination credentials.

6. **Add the first HTTP Request node**
   - Node type: `HTTP Request`
   - Name it: `orig datatables`
   - Connect `config` → `orig datatables`
   - Configure:
     - Method: `GET`
     - URL: `={{ $("config").item.json.orig_url }}/api/v1/data-tables`
     - Authentication: `Predefined Credential Type`
     - Credential type: `n8nApi`
     - Select the credential for the **original** instance

7. **Add the second HTTP Request node**
   - Node type: `HTTP Request`
   - Name it: `orig workflows`
   - Connect `orig datatables` → `orig workflows`
   - Configure:
     - Method: `GET`
     - URL: `={{ $("config").item.json.orig_url }}/api/v1/workflows`
     - Authentication: `Predefined Credential Type`
     - Credential type: `n8nApi`
     - Select the credential for the **original** instance

8. **Add the third HTTP Request node**
   - Node type: `HTTP Request`
   - Name it: `new datatables`
   - Connect `orig workflows` → `new datatables`
   - Configure:
     - Method: `GET`
     - URL: `={{ $("config").item.json.new_url }}/api/v1/data-tables`
     - Authentication: `Predefined Credential Type`
     - Credential type: `n8nApi`
     - Select the credential for the **new** instance

9. **Add the fourth HTTP Request node**
   - Node type: `HTTP Request`
   - Name it: `new workflows`
   - Connect `new datatables` → `new workflows`
   - Configure:
     - Method: `GET`
     - URL: `={{ $("config").item.json.new_url }}/api/v1/workflows`
     - Authentication: `Predefined Credential Type`
     - Credential type: `n8nApi`
     - Select the credential for the **new** instance

10. **Add the mapping Code node**
    - Node type: `Code`
    - Name it: `map`
    - Connect `new workflows` → `map`
    - Configure it to:
      - read `.json.data` from:
        - `orig datatables`
        - `new datatables`
        - `orig workflows`
        - `new workflows`
      - build a single object keyed by `name`
      - store `orig_id` and `new_id` when available
      - return `{ entity_map: existingMap }`
    - Important behavior:
      - matching is by asset name, not by old/new ID
      - both workflows and data tables are merged into one map

11. **Add the workflow rewrite Code node**
    - Node type: `Code`
    - Name it: `update flow nodes`
    - Connect `map` → `update flow nodes`
    - Configure it to:
      - read the `entity_map`
      - build a reverse lookup from old ID to new ID
      - read workflows from `new workflows`
      - target these node types:
        - `n8n-nodes-base.executeWorkflow`
        - `n8n-nodes-base.dataTable`
      - recursively scan each target node’s parameters
      - replace any standalone string or numeric parameter equal to an old ID
      - sanitize each workflow payload before update by:
        - setting `settings = {}`
        - deleting `id`
        - deleting `createdAt`
        - deleting `updatedAt`
        - deleting `versionId`
        - deleting `active`
      - emit items shaped like:
        - `id`: destination workflow ID to update
        - `workflow`: stringified updated workflow JSON

12. **Add the n8n node for workflow update**
    - Node type: `n8n`
    - Name it: `update new workflows`
    - Connect `update flow nodes` → `update new workflows`
    - Configure:
      - Operation: `Update`
      - Workflow ID: expression `={{ $json.id }}`
      - Workflow object: expression `={{ $json.workflow }}`
      - Select the credential for the **new** instance
    - This step updates each workflow already present in the destination instance.

13. **Add sticky notes for operator guidance**
    - Add optional `Sticky Note` nodes matching the workflow layout:
      - one for config instructions
      - one for original-instance credential guidance
      - one for destination-instance credential guidance
      - one for mapping/update explanation
      - one warning note before the final update node
      - one high-level note explaining the workflow purpose

14. **Validate response shapes before full execution**
    - Run up to `new workflows`
    - Confirm each HTTP node returns an object with a `data` array
    - If your n8n version/API differs, adapt the Code nodes accordingly

15. **Test the mapping logic safely**
    - Execute up to `map`
    - Inspect `entity_map`
    - Verify names align between original and destination instances
    - Check for missing `orig_id` or `new_id`
    - Watch for duplicate names or collisions between workflows and data tables

16. **Test the rewrite output before enabling updates**
    - Execute up to `update flow nodes`
    - Inspect emitted items
    - Confirm:
      - workflow IDs correspond to destination workflows
      - JSON contains the expected replacements
      - no unrelated values were changed

17. **Run the final update step cautiously**
    - Ensure `update new workflows` uses the destination credential
    - Execute the full workflow
    - Review update results item by item

18. **Recommended hardening improvements if rebuilding for production**
    - Separate workflow and data table maps to avoid cross-type name collisions
    - Add filtering so only changed workflows are updated
    - Add pagination support if API results are paginated
    - Add error handling for missing assets and unmatched names
    - Export a backup of destination workflows before update
    - Log which IDs were replaced in each workflow

## Credential Configuration Notes

### Original instance credential
- Type: `n8n API`
- Used in:
  - `orig datatables`
  - `orig workflows`

### New instance credential
- Type: `n8n API`
- Used in:
  - `new datatables`
  - `new workflows`
  - `update new workflows`

## Sub-workflow Setup / Expectations

This workflow does **not** call another workflow directly as part of its own execution.

However, it specifically rewrites references inside migrated workflows for nodes of type:
- `Execute Workflow` (`n8n-nodes-base.executeWorkflow`)
- `Data Table` (`n8n-nodes-base.dataTable`)

Expected assumptions for those migrated workflows:
- the target sub-workflows already exist in the new instance
- the target data tables already exist in the new instance
- names are preserved between old and new instances so ID mapping can succeed

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Migration Helper - Automated Reconnection for Workflows + Datatables | High-level purpose of the workflow |
| The workflow automates re-wiring of subworkflows and datatable actions after migration to a new n8n instance. | Overall behavior |
| Recursive ID swapping is used to find and replace old IDs even when nested inside parameter objects. | Core implementation detail |
| Operators must define the correct original and new instance URLs in the `config` node. | Configuration |
| Operators must create and assign separate n8n API credentials for the original and new instances. | Authentication |
| Extra caution is required on the final update node because it writes workflow JSON back to the destination instance. | Operational warning |

## Additional Technical Notes

- The mapping strategy depends entirely on **matching names** across instances.
- The workflow assumes the n8n API endpoints `/api/v1/workflows` and `/api/v1/data-tables` are available and return a `data` array.
- The current map merges workflows and data tables into one namespace. If both asset types can share names in your environment, consider splitting them.
- The final update step resets workflow settings to an empty object, which improves compatibility with strict API validation but may remove legitimate settings.
- The workflow is inactive by default and should remain so until tested carefully on non-critical data.