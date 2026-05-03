n8n Nodemation basic - getting started on the workflow canvas (1/3)

https://n8nworkflows.xyz/workflows/n8n-nodemation-basic---getting-started-on-the-workflow-canvas--1-3--18


# n8n Nodemation basic - getting started on the workflow canvas (1/3)

### 1. Workflow Overview

This workflow, named "testworkflow," serves as a basic demonstration of how to use the n8n workflow canvas and nodes by building a simple automated process. It is designed for beginners to understand the core mechanics of node connections, data manipulation, and scheduled execution within n8n.

The workflow consists of three logical blocks:

- **1.1 Scheduled Trigger:** Periodically triggers the workflow every 2 hours.
- **1.2 Data Processing:** Runs a JavaScript function on each item, adding custom variables.
- **1.3 Data Formatting:** Sets and formats specific output data extracted from the function node for further use or inspection.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow execution every 2 hours. It controls when the subsequent nodes run by using a timed interval.

- **Nodes Involved:**  
  - `2 hours Interval`

- **Node Details:**

  - **Node Name:** 2 hours Interval  
  - **Type and Technical Role:** Interval Trigger — initiates the workflow on a fixed time schedule.  
  - **Configuration Choices:**  
    - Interval unit set to "hours"  
    - Interval value set to 2 (executes every two hours)  
  - **Key Expressions/Variables:** None  
  - **Input/Output Connections:**  
    - No input (trigger node)  
    - Outputs to the `FunctionItem` node  
  - **Version Requirements:** Standard n8n interval trigger, no special version dependencies.  
  - **Potential Failures:**  
    - Workflow execution delay or drift if system clock changes  
    - Node may not trigger if workflow is inactive or n8n service is down  
  - **Sub-Workflow Reference:** None

#### 1.2 Data Processing

- **Overview:**  
  This block processes each incoming item using a JavaScript function to add custom variables. It demonstrates how to manipulate data dynamically inside the workflow.

- **Nodes Involved:**  
  - `FunctionItem`

- **Node Details:**

  - **Node Name:** FunctionItem  
  - **Type and Technical Role:** Function Item — executes custom JavaScript code on each input item individually.  
  - **Configuration Choices:**  
    - Adds two new properties to the item:  
      - `myVariable` set to integer `1`  
      - `myVariable2` set to string `"this is exciting"`  
    - Returns the modified item for downstream nodes  
  - **Key Expressions/Variables:**  
    - JavaScript code:
      ```javascript
      item.myVariable = 1;
      item.myVariable2 = "this is exciting";
      return item;
      ```  
  - **Input/Output Connections:**  
    - Input from `2 hours Interval`  
    - Output to `Set` node  
  - **Version Requirements:** Compatible with n8n version supporting Function Item node (v0.130+ recommended).  
  - **Potential Failures:**  
    - JavaScript errors if code is modified incorrectly  
    - Empty or missing input data could cause unexpected behavior but current code is safe  
  - **Sub-Workflow Reference:** None

#### 1.3 Data Formatting

- **Overview:**  
  This block extracts data from the previous function node and formats it by setting a new variable `data` based on the earlier computed property `myVariable2`. It demonstrates data mapping and the use of expressions to reference node outputs.

- **Nodes Involved:**  
  - `Set`

- **Node Details:**

  - **Node Name:** Set  
  - **Type and Technical Role:** Set — assigns new values to output data fields, optionally filtering output to only set values.  
  - **Configuration Choices:**  
    - Sets one string field named `data`  
    - Value is dynamically assigned using expression referencing `FunctionItem` node's output:  
      ```n8n
      {{$node["FunctionItem"].data["myVariable2"]}}
      ```  
    - `Keep Only Set` option enabled, so only the `data` field is output downstream  
  - **Input/Output Connections:**  
    - Input from `FunctionItem`  
    - No further output connections (end of workflow)  
  - **Version Requirements:** No special requirements; expression syntax supported in all recent n8n versions.  
  - **Potential Failures:**  
    - Expression failure if `FunctionItem` node output is missing or malformed  
    - If multiple items are processed, ensure expression indexing matches expected data structure  
  - **Sub-Workflow Reference:** None

---

### 3. Summary Table

| Node Name       | Node Type         | Functional Role            | Input Node(s)      | Output Node(s)   | Sticky Note                                      |
|-----------------|-------------------|----------------------------|--------------------|------------------|-------------------------------------------------|
| 2 hours Interval| Interval Trigger  | Scheduled trigger every 2 hours | (none)            | FunctionItem     | It is the 4 hours interval in which this node gets executed |
| FunctionItem    | Function Item     | Adds custom variables to each item | 2 hours Interval  | Set              |                                                 |
| Set             | Set               | Maps and formats output data | FunctionItem       | (none)           |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Interval Trigger Node:**
   - Add a new node of type **Interval**.
   - Set the **Unit** parameter to `hours`.
   - Set the **Interval** parameter to `2`.
   - Optionally, add a note: "It is the 4 hours interval in which this node gets executed" (note states 4 hours but configured as 2 hours in this workflow).

2. **Create a Function Item Node:**
   - Add a new node of type **Function Item**.
   - In the **Function Code** editor, enter:
     ```javascript
     item.myVariable = 1;
     item.myVariable2 = "this is exciting";
     return item;
     ```
   - Connect the output of the **Interval** node to the input of this **Function Item** node.

3. **Create a Set Node:**
   - Add a new node of type **Set**.
   - In the **Values to Set**, add a new string field named `data`.
   - Set the value for `data` using the expression editor:
     ```
     {{$node["FunctionItem"].data["myVariable2"]}}
     ```
   - Enable the option **Keep Only Set** to output only the `data` field.
   - Connect the output of the **Function Item** node to the input of this **Set** node.

4. **Finalize Workflow:**
   - Ensure all nodes are connected in the order: Interval Trigger → Function Item → Set.
   - Save and activate the workflow.
   - Confirm credentials are not required for these nodes.
   - Test the workflow by manual trigger or waiting for the interval execution.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                           |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow demonstrates basic node usage and workflow canvas interactions in n8n.                 | Workflow description                                      |
| Video tutorial explaining this workflow: [YouTube Video](https://youtu.be/JIaxjH2CyFc)               | https://youtu.be/JIaxjH2CyFc                              |
| Thumbnail preview available here: ![Thumbnail](http://img.youtube.com/vi/JIaxjH2CyFc/0.jpg)         | Video thumbnail                                           |

---

This reference fully documents the simple "testworkflow" and enables reproduction, modification, and error anticipation for users ranging from beginners to advanced practitioners.