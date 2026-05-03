Count the items returned by a node

https://n8nworkflows.xyz/workflows/count-the-items-returned-by-a-node-1913


# Count the items returned by a node

### 1. Workflow Overview

This workflow counts the number of items returned by a preceding node in an n8n automation. It is designed as a simple demonstration or utility to measure the size of a dataset fetched from a data source.

**Use Case:**  
- Quickly determining how many records are retrieved from a data source node, useful for validation, reporting, or conditional logic in larger workflows.

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger to start workflow execution.  
- **1.2 Data Retrieval:** Fetch all customer records from a training datastore node.  
- **1.3 Item Counting:** Use a Set node with the "Execute Once" option to count all items received from the previous node and output the count as a single item.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Starts the workflow manually via user interaction in the n8n editor or UI.

- **Nodes Involved:**  
  - When clicking "Execute Workflow"

- **Node Details:**  
  - **Node Name:** When clicking "Execute Workflow"  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters required; triggers execution when user clicks "Execute Workflow".  
  - **Expressions/Variables:** None  
  - **Input Connections:** None (entry node)  
  - **Output Connections:** Connects to "Customer Datastore (n8n training)" node  
  - **Version Requirements:** Standard node, no special version requirements  
  - **Potential Failures:** None generally; user must manually trigger.  
  - **Sub-workflow:** None

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves all customer records from a predefined training datastore node. This node simulates or accesses a dataset for demonstration.

- **Nodes Involved:**  
  - Customer Datastore (n8n training)

- **Node Details:**  
  - **Node Name:** Customer Datastore (n8n training)  
  - **Type:** n8n Training Customer Datastore Node (custom or built-in n8n training node)  
  - **Configuration:** Operation set to "getAllPeople" to fetch all records  
  - **Expressions/Variables:** None used in parameters; static operation  
  - **Input Connections:** From manual trigger node  
  - **Output Connections:** To the Set node  
  - **Version Requirements:** Requires n8n version supporting this training node (generally latest stable versions)  
  - **Potential Failures:**  
    - Data retrieval errors (e.g., no connection to data source if external)  
    - Empty dataset returns zero items (which is handled gracefully)  
  - **Sub-workflow:** None

#### 1.3 Item Counting

- **Overview:**  
  Counts all items received from the previous node by aggregating them into one execution pass and outputs a single record containing the count.

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Node Name:** Set  
  - **Type:** Set Node  
  - **Configuration:**  
    - **Execute Once:** Enabled to ensure the node executes only one time regardless of input item count.  
    - **Values:** Defines a single field named `itemCount` with the value set by the expression `{{$input.all().length}}`. This expression collects all input items as an array and returns its length, effectively counting the items.  
    - **Keep Only Set:** Enabled to output only the newly defined field, not passing through any original data.  
  - **Expressions/Variables:**  
    - `${{ $input.all().length }}` — returns count of all incoming items at once.  
  - **Input Connections:** From "Customer Datastore (n8n training)" node  
  - **Output Connections:** None (terminal node)  
  - **Version Requirements:** No special requirements; expression feature standard in recent n8n versions  
  - **Potential Failures:**  
    - Expression evaluation failure if input is malformed or empty (though `$input.all()` safely returns empty array)  
    - Misconfiguration of Execute Once option could lead to multiple executions or incorrect count  
  - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                     | Node Type                                | Functional Role          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                            |
|-------------------------------|-----------------------------------------|-------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                          | Start workflow execution | None                          | Customer Datastore (n8n training) |                                                                                                                        |
| Customer Datastore (n8n training) | n8n Training Customer Datastore Node   | Fetch all customer records | When clicking "Execute Workflow" | Set                            |                                                                                                                        |
| Set                           | Set Node                                | Count items received     | Customer Datastore (n8n training) | None                           | Uses Execute Once option and expression `$input.all().length` to count items. [Documentation](https://docs.n8n.io/code-examples/methods-variables-reference/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Add a new node of type "Manual Trigger".  
   - No configuration needed. This is the workflow entry point.

2. **Add the Customer Datastore (n8n training) node:**  
   - Add a node of type "n8n Training Customer Datastore" (available in n8n training nodes).  
   - Set the operation parameter to "getAllPeople" to retrieve all customer records.  
   - Connect the Manual Trigger node’s output to this node’s input.

3. **Add a Set node:**  
   - Add a node of type "Set".  
   - In the node parameters:  
     - Enable "Execute Once" option (this makes the node process all incoming items as a batch).  
     - Under "Values," add a new field:  
       - Name: `itemCount`  
       - Type: Number (or leave as default)  
       - Value: Use the expression editor and enter `{{$input.all().length}}`. This counts all the incoming items.  
     - Enable "Keep Only Set" to output just the count field.  
   - Connect the Customer Datastore node’s output to this Set node’s input.

4. **Save the workflow and execute manually:**  
   - Run the workflow by clicking "Execute Workflow".  
   - The Set node will output a single item with the field `itemCount` containing the count of all retrieved items.

**Credentials:**  
- No credentials are required for these nodes as the Customer Datastore node is part of n8n training and does not require authentication.

**Defaults:**  
- The workflow uses default settings for the Manual Trigger and Customer Datastore nodes.  
- The Set node’s key setting is "Execute Once" enabled, which is essential.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Expression `$input.all()` is documented in the n8n docs for accessing all incoming items at once.                        | https://docs.n8n.io/code-examples/methods-variables-reference/                                                              |
| The `.length` property is standard JavaScript to get array length; used here to count items.                              | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/length                                 |
| The "Execute Once" option in the Set node is critical to run the counting logic once on all items rather than per item.   | n8n Set node documentation and UI                                                                                            |