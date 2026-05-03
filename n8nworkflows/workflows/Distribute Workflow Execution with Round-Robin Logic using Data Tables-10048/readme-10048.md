Distribute Workflow Execution with Round-Robin Logic using Data Tables

https://n8nworkflows.xyz/workflows/distribute-workflow-execution-with-round-robin-logic-using-data-tables-10048


# Distribute Workflow Execution with Round-Robin Logic using Data Tables

### 1. Workflow Overview

This workflow implements a **round-robin load balancer** logic to distribute workflow executions evenly across multiple routes or resources. It targets use cases where a workflow is triggered frequently, and the workload should be balanced among several duplicated processing nodes or sub-workflows to avoid overload on a single instance.

The core logic uses a **data table** to store and track the last used route index (`Last_Used`). Each execution increments this index cyclically from 0 to 3, determining which route to select next. The workflow then routes the execution to one of three no-operation nodes, placeholders meant to be replaced with actual sub-workflow calls or processing resources.

The workflow’s logic can be grouped into the following blocks:

- **1.1 Input Reception and Initialization:** Trigger and initial data retrieval from the data table.
- **1.2 Round-Robin Calculation:** JavaScript code logic to calculate the next route index.
- **1.3 Data Table Update:** Update the data table with the new `Last_Used` value.
- **1.4 Routing:** Switch node directs workflow execution to one of the three routes based on the updated `Last_Used` value.
- **1.5 Placeholder Routes:** NoOp nodes representing actual downstream workflows or resources.
- **1.6 Data Merging:** Merge trigger input data with updated data for passing to downstream routes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
This block starts the workflow upon manual trigger and retrieves the current route index (`Last_Used`) from the data table to determine the next route.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Calculate the next route to use (Data Table - Get operation)  
  - Merge trigger data to pass to subworkflow if needed (Merge)

- **Node Details:**

1. **When clicking ‘Execute workflow’**  
   - **Type:** Manual Trigger  
   - **Role:** Starts the workflow on demand.  
   - **Config:** No parameters; always outputs data even if empty.  
   - **Input:** None  
   - **Output:** Emits a trigger event, optionally passing input data.  
   - **Edge Cases:** Manual trigger can only be activated by a user; no auth issues.  
   - **Sub-workflow:** None

2. **Calculate the next route to use**  
   - **Type:** Data Table (Get operation)  
   - **Role:** Retrieves the current `Last_Used` index from a specific data table entry (`Load Balancer` table with id `l3AW3b12ncdHyI35`).  
   - **Config:** Filters with condition keyValue "1" to fetch the relevant record.  
   - **Input:** Trigger from manual node  
   - **Output:** Outputs the current `Last_Used` value and record id.  
   - **Edge Cases:** Potential errors if data table is inaccessible, empty, or filter condition fails to find a record.  
   - **Sub-workflow:** None

3. **Merge trigger data to pass to subworkflow if needed**  
   - **Type:** Merge node  
   - **Role:** Combines the manual trigger input data and the updated data from the data table for downstream use.  
   - **Config:** Default merge mode (likely merge by index).  
   - **Input:** Two inputs: one from manual trigger and one from data table update node.  
   - **Output:** Combined data for routing.  
   - **Edge Cases:** Mismatched data lengths could cause incomplete merges; careful configuration recommended.  
   - **Sub-workflow:** None

---

#### 1.2 Round-Robin Calculation

- **Overview:**  
Calculates the next route index by incrementing the current `Last_Used` value cyclically from 0 to 3.

- **Nodes Involved:**  
  - Code in JavaScript

- **Node Details:**

1. **Code in JavaScript**  
   - **Type:** Code node (JavaScript)  
   - **Role:** Processes the retrieved data table record to compute the next `Last_Used` value.  
   - **Config Details:**  
     - Reads all input items (should be only one in typical use).  
     - Checks current `Last_Used` value; if 3, resets to 0, else increments by 1.  
     - Returns updated JSON with new `Last_Used` and record `id` only.  
   - **Key Expressions:**  
     ```js
     const currentValue = item.json.Last_Used;
     const newValue = currentValue === 3 ? 0 : currentValue + 1;
     ```
   - **Input:** Output from data table get node  
   - **Output:** Updated `Last_Used` and `id` for update operation  
   - **Edge Cases:**  
     - Input data missing or malformed JSON.  
     - Multiple input items might cause unexpected behavior (should be one).  
     - JavaScript runtime errors if fields missing.  
   - **Sub-workflow:** None

---

#### 1.3 Data Table Update

- **Overview:**  
Updates the `Last_Used` value in the data table with the newly calculated index.

- **Nodes Involved:**  
  - Update last_used in the datatable

- **Node Details:**

1. **Update last_used in the datatable**  
   - **Type:** Data Table (Update operation)  
   - **Role:** Persists the new `Last_Used` index back into the data table record using the record id.  
   - **Config:**  
     - Updates the `Last_Used` column with new value from the code node.  
     - Uses the same filter keyValue "1" to identify record to update.  
     - Mapping mode defines column manually with schema for `Last_Used` as number.  
   - **Input:** Output from JavaScript code node  
   - **Output:** Updated record data, passed to merge node for downstream use  
   - **Edge Cases:**  
     - Data table write failures (permission, network).  
     - Concurrent update conflicts if multiple executions happen simultaneously.  
   - **Sub-workflow:** None

---

#### 1.4 Routing

- **Overview:**  
Routes the workflow execution to one of three predefined routes based on the updated `Last_Used` value.

- **Nodes Involved:**  
  - Round Robin Router (Switch)

- **Node Details:**

1. **Round Robin Router**  
   - **Type:** Switch node  
   - **Role:** Branches execution flow into three separate outputs corresponding to values 1, 2, and 3 of `Last_Used`.  
   - **Config:**  
     - Three conditions checking if `Last_Used` equals 1, 2, or 3 respectively.  
     - Values 0 or others fall back to default output 0 (which routes nowhere in this workflow).  
   - **Key Expression:**  
     - `{{ $json.Last_Used }}` used for condition comparisons.  
   - **Input:** Merged data from trigger and update nodes  
   - **Output:** Three outputs connecting to Route 1, Route 2, and Route 3 nodes.  
   - **Edge Cases:**  
     - Unexpected `Last_Used` values (e.g. 0) go to fallback output 0 (no connected node).  
     - Data type mismatches may cause condition failures.  
   - **Sub-workflow:** None

---

#### 1.5 Placeholder Routes

- **Overview:**  
NoOp nodes serve as placeholders where actual processing workflows or sub-workflows should be linked. They represent the distributed resources or duplicated workflows.

- **Nodes Involved:**  
  - Route 1 (NoOp)  
  - Route 2 (NoOp)  
  - Route 3 (NoOp)

- **Node Details:**

1. **Route 1 / Route 2 / Route 3**  
   - **Type:** No Operation (NoOp) node  
   - **Role:** Placeholder nodes to be replaced by sub-workflow nodes or actual processing logic.  
   - **Config:** No parameters, simply passes data through.  
   - **Input:** From switch node output matching the route index.  
   - **Output:** None further in this workflow.  
   - **Edge Cases:** None inherent; must be replaced or linked for productive use.  
   - **Sub-workflow:** Intended to link to external workflows or duplicates.

---

#### 1.6 Data Merging

- **Overview:**  
Merges input trigger data with updated route data to ensure downstream nodes receive all necessary context.

- **Nodes Involved:**  
  - Merge trigger data to pass to subworkflow if needed (Merge)

- **Node Details:**

1. **Merge trigger data to pass to subworkflow if needed**  
   - **Type:** Merge node  
   - **Role:** Combines initial input and updated data for routing and downstream use.  
   - **Config:** Default merge mode (likely merge by index).  
   - **Input:**  
     - Input 1: Data from manual trigger node  
     - Input 2: Data from update node  
   - **Output:** Combined data passed to routing switch node  
   - **Edge Cases:**  
     - Data length mismatch may cause incomplete merges.  
   - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                              | Node Type           | Functional Role                                  | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                      |
|--------------------------------------|---------------------|-------------------------------------------------|-----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’     | Manual Trigger      | Initiates workflow on manual execution           | None                              | Calculate the next route to use, Merge trigger data to pass to subworkflow if needed | “## Round Robin Load Balancer\nIf you have got a workflow that is getting called frequently. A common way to scale the service is to duplicate resources and have a load balancer that distributes the work between the resources. A simple form is to use a round robin approach that cycles round the available resources...\nIn reality I understand N8N can spawn multiple workers in the backend so can parallel process the same workflow multiple times so this is likely an over the top approach from most use cases.” |
| Calculate the next route to use      | Data Table (Get)    | Fetches current last used route index             | When clicking ‘Execute workflow’  | Code in JavaScript                |                                                                                                |
| Code in JavaScript                   | Code                | Calculates next route index with round-robin logic | Calculate the next route to use    | Update last_used in the datatable |                                                                                                |
| Update last_used in the datatable    | Data Table (Update) | Persists updated last used route index            | Code in JavaScript                | Merge trigger data to pass to subworkflow if needed |                                                                                                |
| Merge trigger data to pass to subworkflow if needed | Merge               | Combines trigger input and updated data          | When clicking ‘Execute workflow’, Update last_used in the datatable | Round Robin Router               | “## Merge Input \nIf your using a trigger that passes some data to process you can merge it.”    |
| Round Robin Router                   | Switch              | Routes execution to one of three routes based on last used index | Merge trigger data to pass to subworkflow if needed | Route 1, Route 2, Route 3        | “## Link to sub-workflows\nReplace these do nothing nodes with links to your duplicates of the workflows to call” |
| Route 1                             | No Operation (NoOp) | Placeholder for first route processing             | Round Robin Router               | None                             |                                                                                                |
| Route 2                             | No Operation (NoOp) | Placeholder for second route processing            | Round Robin Router               | None                             |                                                                                                |
| Route 3                             | No Operation (NoOp) | Placeholder for third route processing             | Round Robin Router               | None                             |                                                                                                |
| Sticky Note                         | Sticky Note         | Note about merging trigger input                   | None                              | None                             | “## Merge Input \nIf your using a trigger that passes some data to process you can merge it.”    |
| Sticky Note1                       | Sticky Note         | Note about replacing NoOp nodes with sub-workflow links | None                              | None                             | “## Link to sub-workflows\nReplace these do nothing nodes with links to your duplicates of the workflows to call” |
| Sticky Note2                       | Sticky Note         | Explanation of purpose and logic of round robin load balancer | None                              | None                             | “## Round Robin Load Balancer\nIf you have got a workflow that is getting called frequently. A common way to scale the service is to duplicate resources and have a load balancer that distributes the work between the resources...” |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow named "Load Balancer".**

2. **Add a Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.  
   - This node initiates the workflow manually.

3. **Add a Data Table node (Get operation):**  
   - Name: `Calculate the next route to use`  
   - Operation: Get  
   - Data Table: Select your "Load Balancer" data table by ID or name.  
   - Filters: Add condition where key equals "1" (to fetch the current route index).  
   - Connect this node to the Manual Trigger node.

4. **Add a Code node (JavaScript):**  
   - Name: `Code in JavaScript`  
   - Paste the following code:  
     ```js
     const items = $input.all();
     return items.map(item => {
       const currentValue = item.json.Last_Used;
       const newValue = currentValue === 3 ? 0 : currentValue + 1;
       return {
         json: {
           Last_Used: newValue,
           id: item.json.id,
         }
       };
     });
     ```  
   - Connect it to the Data Table Get node.

5. **Add a Data Table node (Update operation):**  
   - Name: `Update last_used in the datatable`  
   - Operation: Update  
   - Data Table: Same as above ("Load Balancer").  
   - Filters: Same key filter "1" to find the record to update.  
   - Columns: Define `Last_Used` column (number type) to update with the new value from the code node.  
   - Connect it to the Code node.

6. **Add a Merge node:**  
   - Name: `Merge trigger data to pass to subworkflow if needed`  
   - Merge Mode: Default (merge by index).  
   - Connect input 1 from the Manual Trigger node.  
   - Connect input 2 from the Data Table Update node.

7. **Add a Switch node:**  
   - Name: `Round Robin Router`  
   - Add three rules:  
     - If `Last_Used` equals 1, output 1.  
     - If `Last_Used` equals 2, output 2.  
     - If `Last_Used` equals 3, output 3.  
   - Use expression `{{ $json.Last_Used }}` for condition values.  
   - Connect this node to the Merge node.

8. **Add three No Operation (NoOp) nodes:**  
   - Names: `Route 1`, `Route 2`, `Route 3`  
   - Connect outputs 1, 2, and 3 of the Switch node to these nodes respectively.  
   - These nodes are placeholders for your actual workflows or sub-workflows.

9. **(Optional) Replace NoOp nodes with Sub-Workflow nodes:**  
   - For production, replace each NoOp node with a Sub-Workflow node pointing to the actual workflow that should process that route.  
   - Configure Sub-Workflow nodes with appropriate input/output mappings and credentials.

10. **Add Sticky Notes (optional):**  
    - Add notes for documentation inside the workflow explaining the merge input, round-robin logic, and placeholders.

11. **Save and activate the workflow once tested.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates a simple round-robin load balancer using n8n’s data table feature to track the last used route index. It is suitable for scenarios requiring manual or programmatic distribution of workload across duplicated resources.                                                                                                           | Core workflow description                                                                                                                  |
| For production use, replace NoOp nodes with actual sub-workflows or processing nodes that represent your duplicated resources or services.                                                                                                                                                                                                                  | Sticky Note1 in the workflow                                                                                                               |
| N8N inherently supports parallel processing with multiple workers, so this round-robin approach may be overkill for many scenarios but is useful for explicit workload control.                                                                                                                                                                               | Sticky Note2 in the workflow                                                                                                               |
| Merge node usage allows combining external trigger data with internal routing data, ensuring downstream workflows receive complete context.                                                                                                                                                                                                                | Sticky Note for merge node                                                                                                                 |
| Data table ID used in this workflow: `l3AW3b12ncdHyI35` ("Load Balancer") – ensure you create a data table with this ID or adjust the workflow configuration accordingly.                                                                                                                                                                                     | Data table configuration                                                                                                                   |
| For more information on n8n Data Tables and Switch nodes, refer to n8n documentation: https://docs.n8n.io/nodes/n8n-nodes-base.dataTable/ and https://docs.n8n.io/nodes/n8n-nodes-base.switch/                                                                                                                                                                 | n8n official documentation links                                                                                                          |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly follows content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.