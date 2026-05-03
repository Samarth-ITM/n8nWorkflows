Clone n8n Workflows between Instances using n8n API

https://n8nworkflows.xyz/workflows/clone-n8n-workflows-between-instances-using-n8n-api-3048


# Clone n8n Workflows between Instances using n8n API

### 1. Workflow Overview

This workflow, titled **"Clone n8n Workflows between Instances using n8n API"**, automates the process of copying, syncing, and migrating workflows between different n8n instances or projects. It is designed for users managing multiple environments (e.g., development, staging, production) or teams needing to share workflows seamlessly. The workflow detects workflows present in the source instance but missing in the destination instance and copies only those, ensuring efficient synchronization without duplication.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Source Workflows Retrieval:** Fetch all workflows from the source n8n instance.
- **1.3 Destination Workflows Retrieval:** Fetch all workflows from the destination n8n instance.
- **1.4 Workflow Comparison:** Compare source and destination workflows to identify missing workflows.
- **1.5 Workflow Creation:** Create missing workflows in the destination instance.
- **1.6 Project Assignment:** Assign newly created workflows to a specific project in the destination instance.
- **1.7 Batch Processing:** Manage processing of workflows in batches to handle multiple workflows efficiently.
- **1.8 Utility and Control Nodes:** Includes splitting, merging, filtering, and setting data nodes to support the main logic.
- **1.9 Sticky Notes:** Provide user guidance on changing source/destination credentials and project filters.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually when the user clicks "Test Workflow".
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`
- **Node Details:**  
  - Type: Manual Trigger  
  - Configuration: Default manual trigger, no parameters.  
  - Input: None  
  - Output: Triggers the next nodes to fetch workflows from source and destination.  
  - Edge Cases: None (manual trigger).  
  - Notes: Entry point for the workflow.

#### 2.2 Source Workflows Retrieval

- **Overview:** Fetches all workflows from the source n8n instance using the n8n API.
- **Nodes Involved:**  
  - `GET - Workflows`  
  - `Split Out Workflows`  
- **Node Details:**  
  - `GET - Workflows`  
    - Type: n8n API node  
    - Role: Retrieves workflows from the source instance.  
    - Configuration: Uses API credentials named "AK n8n original account". No filters applied, default request options.  
    - Input: Trigger from manual node.  
    - Output: JSON array of workflows.  
    - Edge Cases: API authentication failure, network timeout, empty workflow list.  
    - Sticky Note: "Change the Source n8n Instance by changing the Credential".  
  - `Split Out Workflows`  
    - Type: Split Out  
    - Role: Splits the workflows array into individual workflow items for processing.  
    - Configuration: Splits on the "id" field, includes all other fields.  
    - Input: Output from `GET - Workflows`.  
    - Output: Individual workflow objects.  
    - Edge Cases: Empty input array.

#### 2.3 Destination Workflows Retrieval

- **Overview:** Fetches workflows from the destination n8n instance to compare against source workflows.
- **Nodes Involved:**  
  - `GET - Destination Workflows`  
  - `Split Out Workflows1`  
- **Node Details:**  
  - `GET - Destination Workflows`  
    - Type: n8n API node  
    - Role: Retrieves workflows from the destination instance.  
    - Configuration: Uses API credentials named "AlexK1919 n8n ent account". Limit set to 200 workflows, batching enabled.  
    - Input: Trigger from manual node.  
    - Output: JSON array of workflows.  
    - Edge Cases: API authentication failure, network timeout, pagination issues if workflows exceed limit.  
    - Sticky Note: "Change the Destination n8n Instance by changing the Credential".  
  - `Split Out Workflows1`  
    - Type: Split Out  
    - Role: Splits workflows array into individual workflow items.  
    - Configuration: Splits on "id" field, includes all other fields.  
    - Input: Output from `GET - Destination Workflows`.  
    - Output: Individual workflow objects.  
    - Edge Cases: Empty input array.

#### 2.4 Workflow Comparison

- **Overview:** Compares source workflows against destination workflows to identify workflows missing in the destination.
- **Nodes Involved:**  
  - `Merge Workflows`  
  - `Code` (disabled)  
- **Node Details:**  
  - `Merge Workflows`  
    - Type: Merge  
    - Role: Performs a SQL-like left join between source and destination workflows on the workflow name to find missing workflows.  
    - Configuration: Mode "combineBySql" with query selecting workflows from source where no matching name exists in destination.  
    - Input: Two inputs - source workflows (input1) and destination workflows (input2).  
    - Output: Workflows present in source but missing in destination.  
    - Edge Cases: Workflows with identical names but different content, name collisions.  
  - `Code`  
    - Type: Code (JavaScript)  
    - Role: Intended for debugging or logging merged output.  
    - Configuration: Disabled in this workflow.  
    - Input: Output from `Merge Workflows`.  
    - Output: Pass-through data.  
    - Edge Cases: None (disabled).

#### 2.5 Workflow Creation

- **Overview:** Creates missing workflows in the destination instance using the n8n API.
- **Nodes Involved:**  
  - `Loop Over Items`  
  - `CREATE - Workflow`  
- **Node Details:**  
  - `Loop Over Items`  
    - Type: Split In Batches  
    - Role: Processes workflows in batches of 5 to avoid API rate limits or overload.  
    - Configuration: Batch size set to 5.  
    - Input: Workflows missing in destination.  
    - Output: Batches of workflows for creation.  
    - Edge Cases: Large number of workflows causing long processing time.  
  - `CREATE - Workflow`  
    - Type: n8n API node  
    - Role: Creates a workflow in the destination instance.  
    - Configuration: Operation "create", constructs workflow object with name, nodes, connections from input JSON.  
    - Input: Individual workflow data from batch.  
    - Output: Created workflow metadata including new workflow ID.  
    - Credentials: Uses "AlexK1919 n8n ent account".  
    - Edge Cases: API authentication failure, invalid workflow data, name conflicts, API rate limits.  
    - Sticky Note: "Change the Destination n8n Instance by changing the Credential".

#### 2.6 Project Assignment

- **Overview:** Assigns the newly created workflows to a specific project in the destination instance.
- **Nodes Involved:**  
  - `n8n - GET - Projects`  
  - `Split Out Projects`  
  - `Filter Project`  
  - `SET Project ID`  
  - `PUT - Workflow in Project`  
- **Node Details:**  
  - `n8n - GET - Projects`  
    - Type: HTTP Request (n8n API)  
    - Role: Retrieves all projects from the destination instance.  
    - Configuration: GET request to `/api/v1/projects` endpoint, authenticated with destination credentials.  
    - Input: Output from `CREATE - Workflow`.  
    - Output: List of projects.  
    - Edge Cases: API failure, empty project list.  
  - `Split Out Projects`  
    - Type: Split Out  
    - Role: Splits projects array into individual project objects.  
    - Input: Output from `n8n - GET - Projects`.  
    - Output: Individual project data.  
  - `Filter Project`  
    - Type: Filter  
    - Role: Filters projects to find the one named "z Original n8n Workflows from AlexK1919" (default).  
    - Configuration: Case-sensitive strict equality on project name.  
    - Input: Individual projects.  
    - Output: Project matching the filter.  
    - Sticky Note: "Change the Destination Project by changing the Project Name".  
    - Edge Cases: Project name not found, multiple projects with same name.  
  - `SET Project ID`  
    - Type: Set  
    - Role: Extracts and assigns the project ID from filtered project data to a JSON field `data.id`.  
    - Input: Filtered project.  
    - Output: JSON with project ID.  
  - `PUT - Workflow in Project`  
    - Type: HTTP Request (n8n API)  
    - Role: Transfers the created workflow to the filtered project by PUT request to `/api/v1/workflows/{workflowId}/transfer`.  
    - Configuration: Sends `destinationProjectId` in body parameters.  
    - Input: Workflow ID from creation and project ID from `SET Project ID`.  
    - Output: Confirmation of transfer.  
    - Edge Cases: API failure, invalid project ID, workflow ID mismatch, permission errors.

#### 2.7 Batch Processing

- **Overview:** Manages the batch processing loop to handle multiple workflows efficiently.
- **Nodes Involved:**  
  - `Loop Over Items` (also part of workflow creation)  
  - `PUT - Workflow in Project` (loops back to `Loop Over Items`)  
- **Node Details:**  
  - The batch node processes workflows in groups of 5, sending each to creation and project assignment sequentially.  
  - Ensures API rate limits and performance stability.  
  - Edge Cases: Long processing times, partial failures in batch.

#### 2.8 Utility and Control Nodes

- **Overview:** Support nodes for data manipulation and flow control.
- **Nodes Involved:**  
  - `Split Out Workflows` and `Split Out Workflows1` (splitting arrays)  
  - `Merge Workflows` (joining data)  
  - `Split Out Projects` (splitting projects array)  
  - `Filter Project` (filtering projects)  
  - `SET Project ID` (setting project ID)  
- **Node Details:**  
  - These nodes handle JSON data transformations, filtering, and preparation for API calls.  
  - They ensure the workflow logic is modular and maintainable.

#### 2.9 Sticky Notes

- **Overview:** Provide user guidance and instructions within the workflow canvas.
- **Nodes Involved:**  
  - `Sticky Note` (near source credentials)  
  - `Sticky Note2` (near destination credentials)  
  - `Sticky Note1` (near project filter)  
- **Content:**  
  - "Change the Source n8n Instance by changing the Credential"  
  - "Change the Destination n8n Instance by changing the Credential"  
  - "Change the Destination Project by changing the Project Name"  
- **Role:** Assist users in customizing the workflow for their environments.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                         | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                  |
|---------------------------|---------------------------|---------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger            | Workflow start trigger                 | None                             | GET - Workflows, GET - Destination Workflows |                                                                                              |
| GET - Workflows           | n8n API Node              | Fetch workflows from source instance  | When clicking ‘Test workflow’    | Split Out Workflows             | Change the Source n8n Instance by changing the Credential                                    |
| Split Out Workflows       | Split Out                 | Split source workflows array           | GET - Workflows                  | Merge Workflows                 |                                                                                              |
| GET - Destination Workflows | n8n API Node              | Fetch workflows from destination       | When clicking ‘Test workflow’    | Split Out Workflows1            | Change the Destination n8n Instance by changing the Credential                               |
| Split Out Workflows1      | Split Out                 | Split destination workflows array      | GET - Destination Workflows      | Merge Workflows                 |                                                                                              |
| Merge Workflows           | Merge                     | Compare source and destination workflows | Split Out Workflows, Split Out Workflows1 | Code                          |                                                                                              |
| Code                     | Code (disabled)           | Debug merged workflows output          | Merge Workflows                  | Loop Over Items                 |                                                                                              |
| Loop Over Items           | Split In Batches          | Batch processing of workflows          | Code                           | CREATE - Workflow (on second output), (empty first output) |                                                                                              |
| CREATE - Workflow         | n8n API Node              | Create missing workflows in destination | Loop Over Items                 | n8n - GET - Projects            | Change the Destination n8n Instance by changing the Credential                               |
| n8n - GET - Projects      | HTTP Request (n8n API)    | Retrieve projects from destination     | CREATE - Workflow               | Split Out Projects             |                                                                                              |
| Split Out Projects        | Split Out                 | Split projects array                    | n8n - GET - Projects            | Filter Project                 |                                                                                              |
| Filter Project            | Filter                    | Filter for specific project by name    | Split Out Projects              | SET Project ID                 | Change the Destination Project by changing the Project Name                                 |
| SET Project ID            | Set                       | Assign project ID for workflow transfer | Filter Project                 | PUT - Workflow in Project       |                                                                                              |
| PUT - Workflow in Project | HTTP Request (n8n API)    | Assign workflow to project              | SET Project ID                  | Loop Over Items                |                                                                                              |
| Sticky Note              | Sticky Note               | Guidance on source credentials          | None                           | None                          | Change the Source n8n Instance by changing the Credential                                    |
| Sticky Note2             | Sticky Note               | Guidance on destination credentials     | None                           | None                          | Change the Destination n8n Instance by changing the Credential                               |
| Sticky Note1             | Sticky Note               | Guidance on changing destination project | None                           | None                          | Change the Destination Project by changing the Project Name                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Source Workflows Fetch Node**  
   - Name: `GET - Workflows`  
   - Type: n8n API Node  
   - Operation: List workflows (default)  
   - Credentials: Configure with source n8n API credentials (read access required)  
   - No filters applied.

3. **Create Split Out Node for Source Workflows**  
   - Name: `Split Out Workflows`  
   - Type: Split Out  
   - Field to split: `id`  
   - Include all other fields.

4. **Create Destination Workflows Fetch Node**  
   - Name: `GET - Destination Workflows`  
   - Type: n8n API Node  
   - Operation: List workflows  
   - Limit: 200 (or adjust as needed)  
   - Credentials: Configure with destination n8n API credentials (read access required)  
   - Enable batching if needed.

5. **Create Split Out Node for Destination Workflows**  
   - Name: `Split Out Workflows1`  
   - Type: Split Out  
   - Field to split: `id`  
   - Include all other fields.

6. **Create Merge Node to Compare Workflows**  
   - Name: `Merge Workflows`  
   - Type: Merge  
   - Mode: combineBySql  
   - SQL Query:  
     ```
     SELECT input1.name, input1.createdAt, input1.updatedAt, input1.active, input1.nodes, input1.settings, input1.connections, input1.pinData, input1.tags, input1.id
     FROM input1
     LEFT JOIN input2 
     ON input1.name = input2.name
     WHERE input2.name IS NULL
     ```
   - Input 1: Source workflows  
   - Input 2: Destination workflows

7. **(Optional) Create Code Node for Debugging**  
   - Name: `Code`  
   - Type: Code (JavaScript)  
   - Disabled by default.  
   - Code: `console.log("Merged Output:", $json); return [$json];`

8. **Create Batch Processing Node**  
   - Name: `Loop Over Items`  
   - Type: Split In Batches  
   - Batch Size: 5

9. **Create Workflow Creation Node**  
   - Name: `CREATE - Workflow`  
   - Type: n8n API Node  
   - Operation: Create workflow  
   - Workflow Object:  
     ```json
     {
       "name": "{{ $json.name }}",
       "nodes": {{ JSON.stringify($json["nodes"]) }},
       "connections": {{ JSON.stringify($json["connections"] || {}) }}
     }
     ```  
   - Credentials: Destination n8n API credentials (write access required)

10. **Create Projects Fetch Node**  
    - Name: `n8n - GET - Projects`  
    - Type: HTTP Request (n8n API)  
    - Method: GET  
    - URL: `https://<destination-instance-url>/api/v1/projects`  
    - Authentication: Use destination n8n API credentials

11. **Create Split Out Node for Projects**  
    - Name: `Split Out Projects`  
    - Type: Split Out  
    - Field to split: `data`  
    - Include all other fields.

12. **Create Filter Node for Project Selection**  
    - Name: `Filter Project`  
    - Type: Filter  
    - Condition:  
      - Field: `{{$json.data.name}}`  
      - Operator: Equals  
      - Value: `"z Original n8n Workflows from AlexK1919"` (change as needed)

13. **Create Set Node to Extract Project ID**  
    - Name: `SET Project ID`  
    - Type: Set  
    - Assign: `data.id` = `{{$json.data.id}}`

14. **Create Workflow Transfer Node**  
    - Name: `PUT - Workflow in Project`  
    - Type: HTTP Request (n8n API)  
    - Method: PUT  
    - URL: `https://<destination-instance-url>/api/v1/workflows/{{ $('CREATE - Workflow').item.json.id }}/transfer`  
    - Body Parameters:  
      - `destinationProjectId`: `{{$json.data.id}}`  
    - Authentication: Destination n8n API credentials

15. **Connect Nodes in Order:**  
    - `When clicking ‘Test workflow’` → `GET - Workflows` and `GET - Destination Workflows` (parallel)  
    - `GET - Workflows` → `Split Out Workflows`  
    - `GET - Destination Workflows` → `Split Out Workflows1`  
    - `Split Out Workflows` and `Split Out Workflows1` → `Merge Workflows`  
    - `Merge Workflows` → `Code` (optional) → `Loop Over Items`  
    - `Loop Over Items` (second output) → `CREATE - Workflow`  
    - `CREATE - Workflow` → `n8n - GET - Projects`  
    - `n8n - GET - Projects` → `Split Out Projects` → `Filter Project` → `SET Project ID` → `PUT - Workflow in Project`  
    - `PUT - Workflow in Project` → loops back to `Loop Over Items` for next batch

16. **Add Sticky Notes for User Guidance:**  
    - Near source credentials: "Change the Source n8n Instance by changing the Credential"  
    - Near destination credentials: "Change the Destination n8n Instance by changing the Credential"  
    - Near project filter: "Change the Destination Project by changing the Project Name"

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow automates workflow migration, syncing, and backup between n8n instances.             | Workflow purpose and benefits as described in the overview.                                        |
| Ensure API credentials have both read and write permissions on source and destination instances.   | Critical for successful workflow fetching and creation.                                            |
| Default destination project is "z Original n8n Workflows from AlexK1919" but can be changed easily.| Modify the Filter Project node condition to target a different project.                             |
| Batch processing size is set to 5 to balance performance and API rate limits.                       | Can be adjusted in the Loop Over Items node parameters.                                            |
| For large numbers of workflows, consider increasing pagination or batch sizes carefully.           | To avoid timeouts or API throttling.                                                               |
| Workflow uses n8n API endpoints and requires n8n version supporting API v1 endpoints (Enterprise). | Version 0.154.0+ recommended for full API support.                                                 |
| Sticky notes in the workflow provide inline guidance for customization.                            | Visible in the workflow editor canvas.                                                             |

---

This documentation provides a complete and detailed reference to understand, reproduce, and modify the "Clone n8n Workflows between Instances using n8n API" workflow, including its structure, node configurations, and operational logic.