Compare Lists and Identify Common Items & Differences Using Custom Keys

https://n8nworkflows.xyz/workflows/compare-lists-and-identify-common-items---differences-using-custom-keys-5033


# Compare Lists and Identify Common Items & Differences Using Custom Keys

### 1. Workflow Overview

This workflow is designed to compare two lists of objects and identify their common elements as well as the differences, based on a specified key property. It is ideal for use cases where you want to analyze two datasets (e.g., transactions, records, or inventory items) and find items that are shared, unique to the first list, or unique to the second list. The workflow is structured into three main logical blocks:

- **1.1 Input Reception:** Receives two input lists and the key field to compare by, triggered by execution from another workflow.
- **1.2 Input Validation:** Checks that the key and both input lists are defined and non-empty, routing to validation messages if any input is missing.
- **1.3 Comparison Logic:** Uses a JavaScript Code node to perform the comparison, producing three outputs: common items, items only in the first list, and items only in the second list.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block is the entry point of the workflow. It waits for execution triggered by another workflow and accepts three inputs: two arrays (`input_a` and `input_b`) and a string key to identify which property to compare between items.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **Name:** When Executed by Another Workflow  
  - **Type:** Execute Workflow Trigger  
  - **Role:** Entry point that triggers this workflow and accepts input parameters.  
  - **Configuration:**  
    - Accepts three inputs defined as workflow inputs:  
      - `input_a` (array) — first list of items  
      - `input_b` (array) — second list of items  
      - `key` (string) — property name used as the comparison key  
  - **Expressions/Variables:** Inputs are accessed downstream via `$json.input_a`, `$json.input_b`, and `$json.key`.  
  - **Connections:** Outputs to the "Switch" node.  
  - **Potential Failures:** If the workflow is triggered without the required inputs, subsequent validation handles errors.  
  - **Version:** 1.1

---

#### 1.2 Input Validation

- **Overview:**  
  This block verifies that all necessary inputs are present and non-empty. It uses a Switch node to route the flow depending on whether the key or either input list is missing or empty, sending control to specific validation messages.

- **Nodes Involved:**  
  - Switch  
  - validation_message 1  
  - validation_message 2  
  - validation_message 3

- **Node Details:**  
  - **Name:** Switch  
  - **Type:** Switch  
  - **Role:** Directs the workflow based on input presence and validity.  
  - **Configuration:**  
    - Three rules check for empty values:  
      1. If the `key` string value is empty → output 1  
      2. If `input_a` array is empty → output 2  
      3. If `input_b` array is empty → output 3  
    - If none of the above, falls back to output 4 (valid inputs).  
  - **Expressions:** Uses expressions like `={{ $json.key }}` and checks if empty with strict type validation.  
  - **Connections:**  
    - Output 1 → validation_message 1  
    - Output 2 → validation_message 2  
    - Output 3 → validation_message 3  
    - Output 4 → Code node (comparison logic)  
  - **Potential Failures:** Misconfigured input format could bypass checks; empty arrays or missing keys cause validation messages.  
  - **Version:** 3.2  
  - **Always outputs data to ensure downstream nodes have input.**

  - **Name:** validation_message 1 / 2 / 3  
  - **Type:** Set  
  - **Role:** Sets failure messages for missing inputs to inform the caller.  
  - **Configuration:**  
    - Each sets a string variable `validation_message` with a specific message:  
      - "You must specify a key"  
      - "You must specify input_a"  
      - "You must specify input_b"  
  - **Connections:** None (end of branch).  
  - **Version:** 3.4

---

#### 1.3 Comparison Logic

- **Overview:**  
  This block performs the core comparison between the two input lists using the specified key. It outputs three arrays: items common to both lists (based on the key), items only in the first list, and items only in the second list.

- **Nodes Involved:**  
  - Code

- **Node Details:**  
  - **Name:** Code  
  - **Type:** Code (JavaScript)  
  - **Role:** Compare two lists by mapping items with the specified key and categorizing them accordingly.  
  - **Configuration:**  
    - Reads `input_a`, `input_b`, and `key` from input JSON.  
    - Creates two JavaScript Maps keyed by the specified property value for fast lookup.  
    - Iterates over the first map:  
      - If key exists in second map, push item from first list to `common` array.  
      - Else, push to `onlyInA`.  
    - Iterates over second map:  
      - If key does not exist in first map, push item to `onlyInB`.  
    - Returns a single JSON object with arrays: `common`, `onlyInA`, `onlyInB`.  
  - **Key Code Snippet:**  
    ```js
    const listA = $input.first().json.input_a;
    const listB = $input.first().json.input_b;
    const key = $input.first().json.key;

    const aMap = new Map(listA.map(item => [item[key], item]));
    const bMap = new Map(listB.map(item => [item[key], item]));

    const common = [];
    const onlyInA = [];
    const onlyInB = [];

    for (const [k, itemA] of aMap) {
      if (bMap.has(k)) {
        common.push(itemA);
      } else {
        onlyInA.push(itemA);
      }
    }

    for (const [k, itemB] of bMap) {
      if (!aMap.has(k)) {
        onlyInB.push(itemB);
      }
    }

    return [{
      json: { common, onlyInA, onlyInB }
    }];
    ```
  - **Input / Output:**  
    - Input: JSON containing `input_a`, `input_b`, and `key`.  
    - Output: JSON object with three arrays: `common`, `onlyInA`, `onlyInB`.  
  - **Potential Failures:**  
    - If `key` does not exist on items, results may be inconsistent or empty.  
    - Input data not arrays or malformed JSON could cause runtime errors.  
    - Large lists may impact performance or timeout.  
  - **Version:** 2

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                      | Input Node(s)                 | Output Node(s)               | Sticky Note                                |
|-----------------------------|------------------------|------------------------------------|------------------------------|-----------------------------|--------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point receiving inputs       |                              | Switch                      |                                            |
| Switch                      | Switch                 | Routes based on input validation   | When Executed by Another Workflow | validation_message 1, validation_message 2, validation_message 3, Code |                                            |
| validation_message 1        | Set                    | Outputs "You must specify a key"   | Switch (output 1)             |                             |                                            |
| validation_message 2        | Set                    | Outputs "You must specify input_a" | Switch (output 2)             |                             |                                            |
| validation_message 3        | Set                    | Outputs "You must specify input_b" | Switch (output 3)             |                             |                                            |
| Code                        | Code                   | Compares lists and outputs results | Switch (output 4)             |                             |                                            |
| Sticky Note                 | Sticky Note            | Instruction: "Pass in two list of item that you like to compare, as well as the key" |                              |                             | ## Pass in two list of item that you like to compare, as well as the key |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When Executed by Another Workflow" node:**  
   - Type: Execute Workflow Trigger  
   - Configure inputs:  
     - Add three inputs under "Workflow Inputs":  
       - Name: `input_a`, Type: Array  
       - Name: `input_b`, Type: Array  
       - Name: `key`, Type: String  
   - Position: (0, 20)

2. **Create "Switch" node:**  
   - Type: Switch  
   - Position: (220, -1)  
   - Configure three rules to validate inputs:  
     - Rule 1: Check if `key` is an empty string (strict type validation).  
     - Rule 2: Check if `input_a` is an empty array.  
     - Rule 3: Check if `input_b` is an empty array.  
   - Set fallback output for valid inputs.  
   - Connect output of "When Executed by Another Workflow" to input of "Switch".

3. **Create three "Set" nodes for validation messages:**  
   - Name: `validation_message 1`  
     - Position: (440, -280)  
     - Set variable `validation_message` to "You must specify a key".  
     - Connect from Switch output 1.  
   - Name: `validation_message 2`  
     - Position: (440, -80)  
     - Set variable `validation_message` to "You must specify input_a".  
     - Connect from Switch output 2.  
   - Name: `validation_message 3`  
     - Position: (440, 120)  
     - Set variable `validation_message` to "You must specify input_b".  
     - Connect from Switch output 3.

4. **Create "Code" node for the list comparison:**  
   - Type: Code  
   - Position: (440, 320)  
   - Use JavaScript code as provided in section 2.3, which:  
     - Reads `input_a`, `input_b`, and `key` from the input JSON.  
     - Creates Maps keyed by the `key` value.  
     - Determines common items, items only in A, and items only in B.  
     - Returns a JSON object with these arrays.  
   - Connect from Switch fallback output (output 4).

5. **Create a "Sticky Note" node:**  
   - Type: Sticky Note  
   - Position: (-140, -160)  
   - Content: "## Pass in two list of item that you like to compare, as well as the key"

6. **Final connections and validation:**  
   - Ensure the flow is linear from "When Executed by Another Workflow" → "Switch" → validation messages or "Code".  
   - No credentials are required for this workflow as it only processes input data internally.  
   - Test by triggering the workflow with example inputs for `input_a`, `input_b`, and `key`.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                            |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| This workflow is inactive by default (`active: false`) and designed to be triggered by other workflows. | Usage context                                               |
| The workflow requires inputs to be passed programmatically from a parent workflow or via manual trigger with parameters. | Input method                                                |
| The key used for comparison must exist in each item of the lists to ensure accurate comparison. | Important usage tip                                         |
| No external integrations or credentials are necessary, simplifying deployment and usage.          | Operational note                                           |
| Sticky Note: "Pass in two list of item that you like to compare, as well as the key"              | User instruction                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.