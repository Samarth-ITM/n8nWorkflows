Rename a key in n8n

https://n8nworkflows.xyz/workflows/rename-a-key-in-n8n-582


# Rename a key in n8n

### 1. Workflow Overview

This workflow demonstrates how to rename a key in a data object within n8n using the built-in **Rename Keys** node. It serves as a companion example for the Rename Keys node documentation, showing a simple, linear transformation from an initial key to a renamed key. The workflow is designed primarily for instructional and testing purposes.

Logical blocks included:
- **1.1 Input Trigger:** Manual trigger to start the workflow.
- **1.2 Data Setup:** Setting an initial data object with a single key-value pair.
- **1.3 Key Renaming:** Transforming the object by renaming the specified key.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block initiates the workflow manually, allowing the user to execute the workflow on demand for testing or demonstration.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters required; simply triggers execution.  
    - Key expressions/variables: None.  
    - Input connections: None (start node).  
    - Output connections: Connects to the Set node.  
    - Version-specific requirements: Standard manual trigger node, available in all recent n8n versions.  
    - Potential failures: None expected; this node is straightforward.  
    - Sub-workflow: None.

#### 1.2 Data Setup

- **Overview:**  
  This block prepares the initial data object with a key-value pair to be transformed later.

- **Nodes Involved:**  
  - Set

- **Node Details:**

  - **Set**  
    - Type: Set  
    - Role: Defines the initial data object with a key `"key"` assigned the string value `"somevalue"`.  
    - Configuration: Sets one string value pair: `{ key: "somevalue" }`. No additional options or expressions used.  
    - Key expressions/variables: None; static value used.  
    - Input connections: Receives input from the manual trigger node.  
    - Output connections: Connects to the Rename Keys node.  
    - Version-specific requirements: Compatible with n8n versions supporting Set node (standard).  
    - Potential failures: None expected; static data set.  
    - Sub-workflow: None.

#### 1.3 Key Renaming

- **Overview:**  
  This block renames the key `"key"` in the input object to `"newkey"`, demonstrating the Rename Keys node usage.

- **Nodes Involved:**  
  - Rename Keys

- **Node Details:**

  - **Rename Keys**  
    - Type: Rename Keys  
    - Role: Transforms the input object by renaming the existing key `"key"` to `"newkey"`.  
    - Configuration: Defines a single rename rule: currentKey=`key` → newKey=`newkey`.  
    - Key expressions/variables: None; static mapping used.  
    - Input connections: Receives data from the Set node.  
    - Output connections: None (end node).  
    - Version-specific requirements: Requires n8n version supporting Rename Keys node (standard in recent versions).  
    - Potential failures:  
      - If `"key"` does not exist in the input data, the output will be unchanged (no error, but no rename done).  
      - No authentication needed.  
      - If the input data structure is not an object, node may produce unexpected results.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role          | Input Node(s)          | Output Node(s)      | Sticky Note                           |
|---------------------|---------------------|-------------------------|-----------------------|---------------------|-------------------------------------|
| On clicking 'execute'| Manual Trigger      | Start workflow manually | None                  | Set                 |                                     |
| Set                 | Set                 | Define initial data     | On clicking 'execute' | Rename Keys         |                                     |
| Rename Keys         | Rename Keys          | Rename key "key" to "newkey" | Set                   | None                |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger node:**  
   - Add a new node of type **Manual Trigger**.  
   - No configuration needed. This node will start the workflow on manual execution.

2. **Add a Set node:**  
   - Connect the Manual Trigger node’s output to this Set node’s input.  
   - Configure the Set node:  
     - Under "Values to Set," add a new field:  
       - Name: `key`  
       - Type: String  
       - Value: `somevalue`  
     - Leave other options as default.

3. **Add a Rename Keys node:**  
   - Connect the Set node’s output to this Rename Keys node’s input.  
   - Configure the Rename Keys node:  
     - Go to "Keys" parameter and add a mapping rule:  
       - Current Key: `key`  
       - New Key: `newkey`  
     - No other settings necessary.

4. **Save and Execute:**  
   - Save the workflow.  
   - Click the Execute button on the Manual Trigger node to run the workflow.  
   - Observe the output in the Rename Keys node’s output panel, confirming the key rename.

---

### 5. General Notes & Resources

| Note Content                                                | Context or Link                                 |
|-------------------------------------------------------------|------------------------------------------------|
| Companion workflow for Rename Keys node documentation       | This workflow is intended to illustrate the Rename Keys node usage in n8n. |
| Screenshot available (not included here) shows node layout  | File ID: 171 (internal reference)              |