Split In Batches node noItemsLeft example

https://n8nworkflows.xyz/workflows/split-in-batches-node-noitemsleft-example-995


# Split In Batches node noItemsLeft example

### 1. Workflow Overview

This workflow demonstrates the usage of the **SplitInBatches** node in n8n, specifically illustrating how to utilize the `noItemsLeft` context property to detect when all items have been processed. It is particularly useful for scenarios where data needs to be processed in small chunks or batches, and subsequent logic depends on whether all data has been consumed.

The workflow consists of the following logical blocks:

- **1.1 Input Data Generation:** Generates an array of mock data items to be processed.
- **1.2 Batch Splitting:** Splits the incoming data into batches of size 1.
- **1.3 Batch Completion Check:** Uses an IF node to check if there are any remaining items to process based on the `noItemsLeft` flag.
- **1.4 Post-Processing Notification:** Outputs a message when all batches have been processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Data Generation

- **Overview:**  
  This block generates sample data consisting of 10 items, each with an index property `i`. It serves as a placeholder for any real data source node you might use in a different scenario.

- **Nodes Involved:**  
  - `On clicking 'execute'`  
  - `Function`

- **Node Details:**  
  - **On clicking 'execute'**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow execution manually.  
    - *Configuration:* No parameters; triggers workflow run on button click.  
    - *Input:* None (trigger node).  
    - *Output:* Triggers the `Function` node.  
    - *Edge Cases:* None; manual trigger is stable.  
  - **Function**  
    - *Type:* Function (JavaScript code execution)  
    - *Role:* Generates an array of 10 objects, each with a numeric property `i` from 0 to 9.  
    - *Configuration:* Custom code snippet:
      ```js
      const newItems = [];
      for (let i=0; i<10; i++) {
        newItems.push({json:{i}});
      }
      return newItems;
      ```
    - *Input:* Triggered by Manual Trigger node.  
    - *Output:* Emits 10 JSON objects as separate items.  
    - *Edge Cases:* If the loop count is changed, batch processing logic must be adjusted accordingly. No error expected here unless code is modified improperly.  

#### 2.2 Batch Splitting

- **Overview:**  
  Splits incoming data into batches of size 1 to process items sequentially.

- **Nodes Involved:**  
  - `SplitInBatches`

- **Node Details:**  
  - **SplitInBatches**  
    - *Type:* SplitInBatches  
    - *Role:* Processes input items in batches, emitting one batch at a time downstream.  
    - *Configuration:*  
      - Batch Size: 1 (process one item per batch).  
      - No additional options configured.  
    - *Input:* Receives 10 items from the `Function` node.  
    - *Output:* Sends one item per batch to the `IF` node.  
    - *Context:* Maintains `noItemsLeft` boolean flag indicating whether all batches have been processed.  
    - *Edge Cases:*  
      - If batch size is changed, downstream logic must handle accordingly.  
      - If input is empty, `noItemsLeft` will be true immediately.  
      - Potential timeout if downstream nodes are slow or have errors, causing batch processing delays.  

#### 2.3 Batch Completion Check

- **Overview:**  
  Evaluates if all batches are processed using the `noItemsLeft` context variable from the `SplitInBatches` node.

- **Nodes Involved:**  
  - `IF`

- **Node Details:**  
  - **IF**  
    - *Type:* If  
    - *Role:* Conditional branching based on the batch processing status.  
    - *Configuration:*  
      - Condition Type: Boolean comparison.  
      - Expression: `{{$node["SplitInBatches"].context["noItemsLeft"]}}`  
      - Logic: Compares `true` with the `noItemsLeft` flag from the SplitInBatches node.  
    - *Input:* Receives batch items one by one from `SplitInBatches`.  
    - *Output:*  
      - True Output: Executes when all batches are processed (`noItemsLeft` is true).  
      - False Output: Executes when there are still batches to process (`noItemsLeft` is false).  
    - *Edge Cases:*  
      - Expression failure if the context property is unavailable (e.g., node renamed).  
      - Logic inverted if expression syntax is modified incorrectly.  
      - Workflow may loop infinitely if connections are misconfigured.  

#### 2.4 Post-Processing Notification

- **Overview:**  
  Outputs a message indicating that no items remain to be processed.

- **Nodes Involved:**  
  - `Set`

- **Node Details:**  
  - **Set**  
    - *Type:* Set  
    - *Role:* Creates a static message output confirming completion of batch processing.  
    - *Configuration:*  
      - Sets a single string field: `Message` with value `"No Items Left"`.  
      - Keeps only set fields; all other data discarded.  
    - *Input:* True output branch of the `IF` node (i.e., when all items processed).  
    - *Output:* Outputs message object downstream (if connected).  
    - *Edge Cases:* None significant. Can be replaced with any node to trigger subsequent processing after batches complete.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                 | Input Node(s)          | Output Node(s)       | Sticky Note                                |
|------------------------|--------------------|--------------------------------|-----------------------|----------------------|--------------------------------------------|
| On clicking 'execute'   | Manual Trigger     | Initiates workflow execution    | None                  | Function             |                                            |
| Function               | Function           | Generates mock data items       | On clicking 'execute' | SplitInBatches       | This node generates mock data for the workflow. Replace it with your data source. |
| SplitInBatches         | SplitInBatches     | Splits data into batches of 1   | Function              | IF                   | Batch size set to 1; adjust as needed.     |
| IF                     | If                 | Checks if all batches processed | SplitInBatches        | Set (true), SplitInBatches (false) | Uses expression `{{$node["SplitInBatches"].context["noItemsLeft"]}}` to check if batches remain. |
| Set                    | Set                | Outputs "No Items Left" message | IF (true output)      | None                 | Prints message when no items left.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node of type **Manual Trigger**.  
   - No configuration needed. This node starts the workflow on demand.

2. **Create Function Node**  
   - Add node of type **Function**, connect its input to the Manual Trigger node.  
   - Set the function code to:
     ```js
     const newItems = [];
     for (let i = 0; i < 10; i++) {
       newItems.push({ json: { i } });
     }
     return newItems;
     ```
   - This generates 10 items with an index property.

3. **Create SplitInBatches Node**  
   - Add node of type **SplitInBatches**, connect its input to the Function node.  
   - Set **Batch Size** parameter to `1` (process one item per batch).  
   - No additional options need to be configured.

4. **Create IF Node**  
   - Add node of type **If**, connect its input to the SplitInBatches node.  
   - Configure a single Boolean condition:  
     - Value 1: `true`  
     - Value 2: Expression: `{{$node["SplitInBatches"].context["noItemsLeft"]}}`  
   - This condition evaluates whether all batches have been processed.

5. **Create Set Node**  
   - Add node of type **Set**, connect its input to the **true** output of the IF node.  
   - Configure the Set node to:  
     - Keep only set fields  
     - Set a string field named `Message` with value `"No Items Left"`.

6. **Connect the false output of the IF node back to the SplitInBatches node**  
   - This loops the workflow back to continue processing next batch until no items are left.

7. **Save and execute the workflow**  
   - Click **Execute Workflow** button on the Manual Trigger node to start processing.  
   - Observe batch processing and the final message output when all items are processed.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The expression `{{$node["SplitInBatches"].context["noItemsLeft"]}}` is essential to detect batch completion status. | Official n8n documentation on SplitInBatches node: https://docs.n8n.io/nodes/n8n-nodes-base.splitInBatches/ |
| This workflow is a minimal example; in production, replace the Function node with your actual data source. |                                                                                                          |
| Batch size can be adjusted as needed; remember to update downstream logic accordingly.                     |                                                                                                          |
| The loop back from IF's false output to SplitInBatches allows iterative batch processing until completion. |                                                                                                          |

---

This structured analysis and reproduction guide should enable users and automation agents to understand, modify, and recreate the workflow effectively.