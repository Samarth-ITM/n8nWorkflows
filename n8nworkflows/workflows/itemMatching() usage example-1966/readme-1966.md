itemMatching() usage example

https://n8nworkflows.xyz/workflows/itemmatching---usage-example-1966


# itemMatching() usage example

### 1. Workflow Overview

This workflow demonstrates a practical example of using the `itemMatching(itemIndex: Number)` method within a Code node in n8n. The primary purpose is to show how to retrieve linked items from earlier nodes in the workflow, enabling data restoration or augmentation based on previously processed data.

**Target Use Cases:**  
- Learning how to link and access data items across nodes for complex data transformations.  
- Manipulating dataset fields selectively, such as reducing data to a subset and then restoring original fields.  

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger to start workflow execution.  
- **1.2 Data Retrieval:** Fetch all customer records from a training datastore node.  
- **1.3 Data Reduction:** Simplify the dataset by retaining only the "name" field.  
- **1.4 Data Restoration:** Use a Python Code node leveraging `itemMatching` to restore the email address fields from the original dataset.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually. It allows a user to start the process on demand.

- **Nodes Involved:**  
  - When clicking "Execute Workflow"

- **Node Details:**

  - **Node Name:** When clicking "Execute Workflow"  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters; acts as a simple trigger to start the workflow.  
  - **Expressions/Variables:** None.  
  - **Input Connections:** None (start node).  
  - **Output Connections:** Connects to "Customer Datastore (n8n training)".  
  - **Version Requirements:** Standard node available in all current n8n versions.  
  - **Failure Modes:** None expected; manual trigger unless interrupted by user or system.  

---

#### 2.2 Data Retrieval

- **Overview:**  
  This block retrieves all customer data from a pre-configured training datastore. It outputs a full dataset with all fields.

- **Nodes Involved:**  
  - Customer Datastore (n8n training)

- **Node Details:**

  - **Node Name:** Customer Datastore (n8n training)  
  - **Type:** n8n Training Customer Datastore (Custom data source node)  
  - **Configuration:**  
    - Operation: `getAllPeople` — fetches all customer records.  
    - Return all: `true` — no pagination, returns all data at once.  
  - **Expressions/Variables:** None.  
  - **Input Connections:** From "When clicking \"Execute Workflow\"".  
  - **Output Connections:** To "Edit Fields".  
  - **Version Requirements:** Custom node for training, available in n8n training environments.  
  - **Failure Modes:**  
    - Data source unavailability or incorrect configuration.  
    - Network or permission errors if accessing a remote datastore.  

---

#### 2.3 Data Reduction

- **Overview:**  
  Simplifies the dataset by removing all fields except the "name" field, effectively reducing data volume and complexity.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**

  - **Node Name:** Edit Fields  
  - **Type:** Set (Field editing/selection)  
  - **Configuration:**  
    - Fields: Only "name" is retained, set via expression `={{ $json.name }}`.  
    - Include Mode: `none` (exclude all fields not explicitly listed).  
  - **Expressions/Variables:**  
    - Uses direct JSON path reference to retain "name".  
  - **Input Connections:** From "Customer Datastore (n8n training)".  
  - **Output Connections:** To "Code".  
  - **Version Requirements:** Set node v3.2 (version 3.2 or higher recommended).  
  - **Failure Modes:**  
    - If input data is empty or missing "name", output will be empty or missing field.  

---

#### 2.4 Data Restoration

- **Overview:**  
  Uses a Python Code node to restore the "email" field from the original full dataset by matching the current item index with the corresponding item in the prior node’s output using `itemMatching`.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Node Name:** Code  
  - **Type:** Code Node (Python)  
  - **Configuration:**  
    - Language: Python  
    - Code logic:  
      Iterates over all input items; for each index `i`, retrieves the linked item from the node named "Customer Datastore (n8n training)" using `itemMatching(i)`. It then restores the email address field into the current item JSON under `restoreEmail`.  
  - **Key expressions:**  
    ```python
    for i,item in enumerate(_input.all()):
        _input.all()[i].json.restoreEmail = _('Customer Datastore (n8n training)').itemMatching(i).json.email
    return _input.all();
    ```  
  - **Input Connections:** From "Edit Fields".  
  - **Output Connections:** None (end of workflow).  
  - **Version Requirements:** Code node v2 or higher supports Python and `itemMatching`.  
  - **Failure Modes:**  
    - If item indexes do not align or data missing in the referenced node, may cause index errors or null references.  
    - If "Customer Datastore (n8n training)" output changes structure, `email` field may be missing.  
    - Python environment errors or syntax errors if code is modified incorrectly.  

---

#### 2.5 Sticky Notes

Sticky notes are used to document the workflow steps visually in the editor but do not affect execution.

- **Sticky Note (Generate example data):** Positioned near the trigger and datastore node, indicating these nodes generate example data.  
- **Sticky Note1 (Reduce the data):** Positioned near "Edit Fields," describes the purpose to reduce data to names only.  
- **Sticky Note2 (Restore):** Positioned near the "Code" node, explaining that the email addresses are restored here.  
- **Sticky Note3 (About this workflow):** Describes the overall workflow purpose and references the official documentation link for `itemMatching`. Provides a link to [Retrieve linked items from earlier in the workflow](https://docs.n8n.io/code/cookbook/builtin/itemmatching/).

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                      | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                          |
|-------------------------------|--------------------------------------|------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                      | Workflow initiation trigger        | None                        | Customer Datastore (n8n training) |                                                                                                |
| Customer Datastore (n8n training) | n8n Training Customer Datastore    | Retrieve all customer data          | When clicking "Execute Workflow" | Edit Fields                 | ## Generate example data                                                                           |
| Edit Fields                   | Set                                  | Reduce dataset to only "name" field | Customer Datastore (n8n training) | Code                        | ## Reduce the data<br>Remove all data except the names                                            |
| Code                         | Code (Python)                        | Restore email addresses using itemMatching | Edit Fields                 | None                         | ## Restore<br>Restore the email address data                                                      |
| Sticky Note                   | Sticky Note                         | Documentation, no execution impact | None                        | None                         | ## Generate example data                                                                           |
| Sticky Note1                  | Sticky Note                         | Documentation, no execution impact | None                        | None                         | ## Reduce the data<br>Remove all data except the names                                            |
| Sticky Note2                  | Sticky Note                         | Documentation, no execution impact | None                        | None                         | ## Restore<br>Restore the email address data                                                      |
| Sticky Note3                  | Sticky Note                         | Documentation, no execution impact | None                        | None                         | ## About this workflow<br>This workflow provides a simple example of how to use `itemMatching(itemIndex: Number)` in the Code node to retrieve linked items from earlier in the workflow.<br><br>This example uses JavaScript. Refer to [Retrieve linked items from earlier in the workflow](https://docs.n8n.io/code/cookbook/builtin/itemmatching/) for the Python code. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: "When clicking \"Execute Workflow\""  
   - No parameters required.  
   - Position accordingly (e.g., starting point).  

2. **Add Customer Datastore Node**  
   - Node Type: n8n Training Customer Datastore (or equivalent data source)  
   - Name: "Customer Datastore (n8n training)"  
   - Parameters:  
     - Operation: `getAllPeople`  
     - Return All: `true`  
   - Connect input from "When clicking \"Execute Workflow\"" node.  

3. **Add Set Node to Edit Fields**  
   - Node Type: Set  
   - Name: "Edit Fields"  
   - Parameters:  
     - Include Mode: `none` (exclude all fields by default)  
     - Add a field named "name" with value expression: `={{ $json.name }}`  
   - Connect input from "Customer Datastore (n8n training)" node.  

4. **Add Code Node for Data Restoration**  
   - Node Type: Code  
   - Name: "Code"  
   - Parameters:  
     - Language: Python  
     - Code:  
       ```python
       for i,item in enumerate(_input.all()):
           _input.all()[i].json.restoreEmail = _('Customer Datastore (n8n training)').itemMatching(i).json.email
       return _input.all();
       ```  
   - Connect input from "Edit Fields" node.  

5. **Add Sticky Notes (Optional for Documentation)**  
   - Add a sticky note near nodes 1-2 titled "Generate example data".  
   - Add a sticky note near node 3 titled "Reduce the data\nRemove all data except the names".  
   - Add a sticky note near node 4 titled "Restore\nRestore the email address data".  
   - Add a sticky note summarizing the workflow’s purpose and providing the link:  
     https://docs.n8n.io/code/cookbook/builtin/itemmatching/  

6. **Save and Activate the Workflow**  
   - Use default credentials or configure any required credentials if the datastore or other nodes require authentication (not applicable in this example).  
   - Execute manually by clicking the trigger node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow uses the `itemMatching(itemIndex: Number)` method in the Code node to access linked items by index.   | Official n8n documentation: [Retrieve linked items from earlier in the workflow](https://docs.n8n.io/code/cookbook/builtin/itemmatching/)   |
| The example is provided with Python code in the Code node. A JavaScript alternative is mentioned in the sticky note. | n8n supports both JavaScript and Python in Code nodes; adjust accordingly based on user preference.            |
| The "Customer Datastore (n8n training)" node is a custom training data source used for example purposes.             | This node might not be available in all n8n installations; replace with equivalent data source if needed.       |

---

This completes the detailed reference for the `itemMatching() example` workflow, enabling full understanding, reproduction, and adaptation.