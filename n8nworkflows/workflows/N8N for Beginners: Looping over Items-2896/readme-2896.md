N8N for Beginners: Looping over Items

https://n8nworkflows.xyz/workflows/n8n-for-beginners--looping-over-items-2896


# N8N for Beginners: Looping over Items

### 1. Workflow Overview

This workflow, titled **"N8N for Beginners: Looping over Items"**, is designed as an educational example to demonstrate how n8n handles iteration (looping) over multiple items. It targets beginners who want to understand the difference between:

- **Built-in looping:** where nodes automatically process each item in an input array individually.
- **Explicit looping:** where the **Loop Over Items** node (SplitInBatches) is used to control iteration, batch size, and execution flow.

The workflow processes an array of URLs in two parallel paths:

- **Path 1 (Unsplit Array Path):** Processes the entire array as a single item, demonstrating that without splitting, nodes treat the array as one object.
- **Path 2 (Split Array Path):** Splits the array into individual URL objects, then processes each item separately, demonstrating explicit iteration with controlled delays and data enrichment.

Logical blocks:

- **1.1 Input Reception:** Manual Trigger node with pinned test data (array of URLs).
- **1.2 Array Splitting:** Split Out node to convert the array into individual items.
- **1.3 Looping Demonstrations:** Two Loop Over Items nodes showing unsplit vs. split processing.
- **1.4 Data Enrichment:** Code nodes adding a constant parameter (`param1`) to each item.
- **1.5 Controlled Execution:** Wait node introducing a 1-second delay per item in the split path.
- **1.6 Output Inspection:** NoOp nodes displaying outputs at various stages.
- **1.7 Sticky Notes:** Visual annotations explaining key points.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow manually and provides the initial JSON data containing an array of URLs.

- **Nodes Involved:**  
  - Paste JSON into this node (Manual Trigger)

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Entry point; triggers workflow execution manually with pinned test data.  
  - **Configuration:**  
    - Test data pinned with JSON containing `"urls"` array of 5 URLs.  
  - **Expressions/Variables:** None (static test data).  
  - **Connections:** Outputs to:  
    - Split Array of Strings into Array of Objects (Split Out)  
    - Add param1 to output5 (Code)  
    - Loop over Items 1 (SplitInBatches)  
  - **Version:** 1  
  - **Potential Failures:** None expected; manual trigger with static data.  
  - **Sub-workflow:** None.

---

#### 2.2 Array Splitting

- **Overview:**  
  This block extracts the `urls` array from the input JSON and splits it into separate JSON objects, each containing one URL, enabling individual item processing.

- **Nodes Involved:**  
  - Split Array of Strings into Array of Objects (Split Out)

- **Node Details:**  
  - **Type:** Split Out  
  - **Role:** Converts an array field (`urls`) into multiple individual items with a new field `url`.  
  - **Configuration:**  
    - Field to split out: `urls`  
    - Destination field name: `url`  
  - **Expressions/Variables:** None dynamic; operates on input JSON field.  
  - **Connections:** Outputs to:  
    - Loop over Items 2 (SplitInBatches)  
    - Add param1 to output4 (Code)  
    - Add param1 to output3 (Code)  
  - **Version:** 1  
  - **Potential Failures:**  
    - If input JSON lacks `urls` field or it is not an array, node will fail or produce empty output.  
  - **Sub-workflow:** None.

---

#### 2.3 Looping Demonstrations

- **Overview:**  
  This block contains two Loop Over Items nodes demonstrating different looping behaviors:

  - **Loop over Items 1:** Processes the entire unsplit array as a single item.
  - **Loop over Items 2:** Processes each split item individually with controlled batch processing.

- **Nodes Involved:**  
  - Loop over Items 1 (SplitInBatches)  
  - Loop over Items 2 (SplitInBatches)

- **Node Details:**  

  **Loop over Items 1:**  
  - **Type:** SplitInBatches  
  - **Role:** Processes input as batches; here it processes the entire array as one batch (single item).  
  - **Configuration:**  
    - Default batch size (1)  
    - Reset option disabled (`reset: false`) to maintain state if rerun.  
  - **Expressions/Variables:** None.  
  - **Connections:**  
    - Input from:  
      - Manual Trigger (direct)  
      - Add param1 to output1 (Code)  
    - Outputs to:  
      - Result1 (NoOp)  
      - Add param1 to output1 (Code) (loopback)  
  - **Version:** 3  
  - **Potential Failures:**  
    - Timeout if batch processing takes too long.  
    - Misconfiguration of batch size could affect iteration.  
  - **Sub-workflow:** None.

  **Loop over Items 2:**  
  - **Type:** SplitInBatches  
  - **Role:** Processes each split item individually, enabling controlled iteration over multiple items.  
  - **Configuration:**  
    - Default batch size (1)  
  - **Expressions/Variables:** None.  
  - **Connections:**  
    - Input from: Split Array of Strings into Array of Objects (Split Out)  
    - Outputs to:  
      - Result2 (NoOp)  
      - Wait one second(just for show) (Wait)  
      - Add param1 to output4 (Code)  
      - Add param1 to output3 (Code)  
  - **Version:** 3  
  - **Potential Failures:**  
    - Timeout or delay issues if Wait node is slow.  
    - Batch size misconfiguration.  
  - **Sub-workflow:** None.

---

#### 2.4 Data Enrichment

- **Overview:**  
  Multiple Code nodes add a constant parameter (`param1`) to each item, demonstrating how to enrich data during iteration.

- **Nodes Involved:**  
  - Add param1 to output1 (Code)  
  - Add param1 to output2 (Code)  
  - Add param1 to output3 (Code)  
  - Add param1 to output4 (Code)  
  - Add param1 to output5 (Code)

- **Node Details:**  

  Each Code node:

  - **Type:** Code  
  - **Role:** Adds a field `param1` with value `"add_me_to_all_items_and_name_me_param1"` to each JSON item.  
  - **Configuration:**  
    - JavaScript code snippet:  
      ```js
      $json.param1 = "add_me_to_all_items_and_name_me_param1";
      return $json;
      ```  
    - Execution mode:  
      - Most nodes run **once per item** (`mode: runOnceForEachItem`) except `Add param1 to output5` which runs once for all items.  
  - **Expressions/Variables:** Uses `$json` to access and modify current item.  
  - **Connections:**  
    - `Add param1 to output1` connects back to `Loop over Items 1` (loopback).  
    - `Add param1 to output2` connects to `Loop over Items 2`.  
    - `Add param1 to output3` connects to `Result3` (NoOp).  
    - `Add param1 to output4` connects to `Result4` (NoOp).  
    - `Add param1 to output5` connects to `Result5` (NoOp).  
  - **Version:** 2  
  - **Potential Failures:**  
    - Expression errors if `$json` is undefined or malformed.  
  - **Sub-workflow:** None.

---

#### 2.5 Controlled Execution

- **Overview:**  
  The Wait node introduces a 1-second delay per item in the split array path, demonstrating how to control execution speed and sequencing.

- **Nodes Involved:**  
  - Wait one second(just for show) (Wait)

- **Node Details:**  
  - **Type:** Wait  
  - **Role:** Pauses workflow execution for 1 second per item to simulate delay.  
  - **Configuration:**  
    - Amount: 1 second  
  - **Expressions/Variables:** None.  
  - **Connections:**  
    - Input from: Loop over Items 2  
    - Output to: Add param1 to output2 (Code)  
  - **Version:** 1.1  
  - **Potential Failures:**  
    - Delays may cause workflow to exceed execution time limits.  
  - **Sub-workflow:** None.

---

#### 2.6 Output Inspection

- **Overview:**  
  NoOp nodes serve as result display points for inspecting the output data at various stages.

- **Nodes Involved:**  
  - Result1  
  - Result2  
  - Result3  
  - Result4  
  - Result5

- **Node Details:**  
  - **Type:** NoOp  
  - **Role:** Display output data for inspection without modifying it.  
  - **Configuration:** None.  
  - **Expressions/Variables:** None.  
  - **Connections:**  
    - Result1: from Loop over Items 1  
    - Result2: from Loop over Items 2  
    - Result3: from Add param1 to output3 (Code)  
    - Result4: from Add param1 to output4 (Code)  
    - Result5: from Add param1 to output5 (Code)  
  - **Version:** 1  
  - **Potential Failures:** None.  
  - **Sub-workflow:** None.

---

#### 2.7 Sticky Notes

- **Overview:**  
  Sticky notes provide visual explanations and clarifications for key nodes and outputs.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note2 (large note with full workflow description)

- **Node Details:**  
  - **Type:** Sticky Note  
  - **Role:** Annotate the workflow canvas with explanations.  
  - **Content Highlights:**  
    - Result1 shows array treated as one item by Loop1.  
    - Result2 shows Loop2 sees 5 items after splitting.  
    - Result3 shows built-in looping in n8n nodes.  
    - Result4 shows disabling looping by setting "Run Once For All Items".  
    - Result5 shows array treated as one item by Code5, same as Loop1.  
    - Large note contains full workflow description and setup instructions.  
  - **Version:** 1  
  - **Potential Failures:** None.

---

### 3. Summary Table

| Node Name                          | Node Type       | Functional Role                                  | Input Node(s)                            | Output Node(s)                                      | Sticky Note                                                                                      |
|-----------------------------------|-----------------|-------------------------------------------------|----------------------------------------|----------------------------------------------------|------------------------------------------------------------------------------------------------|
| Paste JSON into this node          | Manual Trigger  | Entry point; triggers workflow with test data   | None                                   | Split Array of Strings into Array of Objects, Add param1 to output5, Loop over Items 1 |                                                                                                |
| Split Array of Strings into Array of Objects | Split Out      | Splits `urls` array into individual URL objects | Paste JSON into this node              | Loop over Items 2, Add param1 to output4, Add param1 to output3 |                                                                                                |
| Loop over Items 1                 | SplitInBatches  | Processes entire array as single batch (unsplit) | Paste JSON into this node, Add param1 to output1 | Result1, Add param1 to output1                      | Sticky Note: Result1 shows array treated as one item by Loop1                                  |
| Loop over Items 2                 | SplitInBatches  | Processes each split item individually           | Split Array of Strings into Array of Objects | Result2, Wait one second(just for show), Add param1 to output4, Add param1 to output3 | Sticky Note1: Result2 shows Loop2 sees 5 items after splitting                                 |
| Wait one second(just for show)    | Wait            | Adds 1-second delay per item                      | Loop over Items 2                      | Add param1 to output2                               |                                                                                                |
| Add param1 to output1             | Code            | Adds constant `param1` to each item (unsplit path) | Loop over Items 1                     | Loop over Items 1                                  |                                                                                                |
| Add param1 to output2             | Code            | Adds constant `param1` to each item (after Wait) | Wait one second(just for show)         | Loop over Items 2                                  |                                                                                                |
| Add param1 to output3             | Code            | Adds constant `param1` to each item (split path) | Split Array of Strings into Array of Objects | Result3                                           | Sticky Note4: Result3 shows built-in looping in n8n nodes                                     |
| Add param1 to output4             | Code            | Adds constant `param1` to each item (split path) | Split Array of Strings into Array of Objects | Result4                                           | Sticky Note3: Result4 shows disabling looping by setting "Run Once For All Items"              |
| Add param1 to output5             | Code            | Adds constant `param1` to entire array (unsplit) | Paste JSON into this node              | Result5                                           | Sticky Note5: Result5 shows array treated as one item by Code5, same as Loop1                 |
| Result1                          | NoOp            | Displays output after Loop over Items 1          | Loop over Items 1                      | None                                               | Sticky Note: Result1 shows array treated as one item by Loop1                                  |
| Result2                          | NoOp            | Displays output after Loop over Items 2          | Loop over Items 2                      | None                                               | Sticky Note1: Result2 shows Loop2 sees 5 items after splitting                                 |
| Result3                          | NoOp            | Displays output after Add param1 to output3      | Add param1 to output3                  | None                                               | Sticky Note4: Result3 shows built-in looping in n8n nodes                                     |
| Result4                          | NoOp            | Displays output after Add param1 to output4      | Add param1 to output4                  | None                                               | Sticky Note3: Result4 shows disabling looping by setting "Run Once For All Items"              |
| Result5                          | NoOp            | Displays output after Add param1 to output5      | Add param1 to output5                  | None                                               | Sticky Note5: Result5 shows array treated as one item by Code5, same as Loop1                 |
| Sticky Note                      | Sticky Note     | Annotation for Result1                            | None                                   | None                                               | Result1 shows that the array of strings is seen as one item by Loop1                          |
| Sticky Note1                     | Sticky Note     | Annotation for Result2                            | None                                   | None                                               | Result2 shows that the Loop2 sees 5 items after the array of strings is split into separate objects |
| Sticky Note2                     | Sticky Note     | Full workflow description and instructions       | None                                   | None                                               |                                                                                                |
| Sticky Note3                     | Sticky Note     | Annotation for Result4                            | None                                   | None                                               | Result4 shows that we can turn off looping by setting "Run Once For All Items"                |
| Sticky Note4                     | Sticky Note     | Annotation for Result3                            | None                                   | None                                               | Result3 shows that looping over items is built in to n8n nodes                               |
| Sticky Note5                     | Sticky Note     | Annotation for Result5                            | None                                   | None                                               | Result5 shows that the array of strings is seen as one item by Code5. So behavior same as Loop1 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `Paste JSON into this node`  
   - No credentials needed.  
   - Edit Output: Paste the following JSON to pin test data:  
     ```json
     {
       "urls": [
         "https://www.reddit.com",
         "https://www.n8n.io/",
         "https://n8n.io/",
         "https://supabase.com/",
         "https://duckduckgo.com/"
       ]
     }
     ```  
   - Save and close.

2. **Create Split Out node**  
   - Name: `Split Array of Strings into Array of Objects`  
   - Field to split out: `urls`  
   - Destination field name: `url`  
   - Connect input from `Paste JSON into this node`.

3. **Create Loop Over Items 1 node (SplitInBatches)**  
   - Name: `Loop over Items 1`  
   - Set `Reset` option to `false` (to maintain state).  
   - Connect inputs from:  
     - `Paste JSON into this node` (direct)  
     - `Add param1 to output1` (loopback)  
   - Outputs to:  
     - `Result1`  
     - `Add param1 to output1`

4. **Create Loop Over Items 2 node (SplitInBatches)**  
   - Name: `Loop over Items 2`  
   - Default batch size (1)  
   - Connect input from `Split Array of Strings into Array of Objects`.  
   - Outputs to:  
     - `Result2`  
     - `Wait one second(just for show)`  
     - `Add param1 to output4`  
     - `Add param1 to output3`

5. **Create Wait node**  
   - Name: `Wait one second(just for show)`  
   - Amount: 1 second  
   - Connect input from `Loop over Items 2`.  
   - Output to `Add param1 to output2`.

6. **Create Code nodes to add `param1`**  
   - `Add param1 to output1`  
     - Mode: Run Once For Each Item  
     - Code:  
       ```js
       $json.param1 = "add_me_to_all_items_and_name_me_param1";
       return $json;
       ```  
     - Input from `Loop over Items 1`.  
     - Output back to `Loop over Items 1` (loopback).

   - `Add param1 to output2`  
     - Mode: Run Once For Each Item  
     - Same code as above.  
     - Input from `Wait one second(just for show)`.  
     - Output to `Loop over Items 2`.

   - `Add param1 to output3`  
     - Mode: Run Once For Each Item  
     - Same code as above.  
     - Input from `Split Array of Strings into Array of Objects`.  
     - Output to `Result3`.

   - `Add param1 to output4`  
     - Mode: Run Once For Each Item  
     - Same code as above.  
     - Input from `Split Array of Strings into Array of Objects`.  
     - Output to `Result4`.

   - `Add param1 to output5`  
     - Mode: Default (runs once for all items)  
     - Same code as above.  
     - Input from `Paste JSON into this node`.  
     - Output to `Result5`.

7. **Create NoOp nodes for output inspection**  
   - `Result1`  
     - Input from `Loop over Items 1`.  
   - `Result2`  
     - Input from `Loop over Items 2`.  
   - `Result3`  
     - Input from `Add param1 to output3`.  
   - `Result4`  
     - Input from `Add param1 to output4`.  
   - `Result5`  
     - Input from `Add param1 to output5`.

8. **Add Sticky Notes for clarity** (optional but recommended)  
   - Add notes near `Result1` explaining that the array is treated as one item by Loop1.  
   - Add notes near `Result2` explaining that Loop2 sees 5 items after splitting.  
   - Add notes near `Result3` and `Result4` explaining built-in looping and disabling looping behavior.  
   - Add notes near `Result5` explaining similarity to Loop1 behavior.  
   - Add a large note with workflow description and setup instructions.

9. **Connect all nodes as per the connections described in the Summary Table.**

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is designed for beginners to understand looping in n8n, demonstrating built-in vs explicit looping. | Workflow description and setup instructions embedded in Sticky Note2.                              |
| The Wait node is optional and can be removed to speed up execution.                                  | Wait node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/  |
| Code nodes add a constant parameter `param1` to each item to demonstrate data enrichment during iteration. | Code node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/  |
| Split Out node converts arrays into individual items for processing.                                 | Split Out node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitout/ |
| Loop Over Items node (SplitInBatches) controls batch processing and iteration.                       | Loop Over Items node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitinbatches/ |
| Manual Trigger node allows manual execution with pinned test data.                                   | Manual Trigger documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.manualtrigger/ |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and modify the workflow, anticipate potential errors, and optimize integration scenarios.