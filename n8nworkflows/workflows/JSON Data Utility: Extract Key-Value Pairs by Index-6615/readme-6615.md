JSON Data Utility: Extract Key-Value Pairs by Index

https://n8nworkflows.xyz/workflows/json-data-utility--extract-key-value-pairs-by-index-6615


# JSON Data Utility: Extract Key-Value Pairs by Index

### 1. Workflow Overview

This workflow, titled **"📄 Extract Key Value from JSON - n8n Workflow"**, is designed to extract a specific key-value pair from a JSON object based on a user-provided index. It is particularly useful for scenarios where you need to dynamically select and retrieve one key-value entry from a structured JSON dataset by specifying its position (index) rather than the key name itself.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** The workflow is manually triggered and receives the JSON data along with the index of the key-value pair to extract.
- **1.2 Key-Value Extraction Logic:** A Python code node processes the JSON input, extracts the key-value pair at the specified index, and returns it in a structured format.
- **1.3 Output Structuring:** Two separate nodes isolate and format the extracted key and value for downstream use or display.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and provides the JSON input data along with the index of the key-value pair to extract.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Input JSON Node

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type & Role:* Manual Trigger node; starts workflow execution on demand.  
  - *Configuration:* No parameters; simple manual trigger.  
  - *Input/Output:* No input; outputs trigger event to next node.  
  - *Edge Cases:* None specific; user must manually start workflow.

- **Input JSON Node**  
  - *Type & Role:* Set node; provides static JSON data and index as workflow input.  
  - *Configuration:*  
    - Mode: Raw JSON input  
    - JSON provided:
      ```json
      {
        "myData": {
          "name": "Alice",
          "age": "30",
          "city": "Paris"
        },
        "rowIndex": "1"
      }
      ```  
    - `rowIndex` is the position of the key-value pair to extract (0-based).  
  - *Input/Output:* Receives trigger from manual node; outputs JSON for processing.  
  - *Edge Cases:*  
    - `rowIndex` must be a valid integer string convertible to int.  
    - If index is out of range of keys, extraction node will handle it.

#### 1.2 Key-Value Extraction Logic

**Overview:**  
This block extracts the key-value pair at the specified index from the input JSON using a Python script.

**Nodes Involved:**  
- Find Key-Value Pair

**Node Details:**  

- **Find Key-Value Pair**  
  - *Type & Role:* Code node running Python; performs indexed key-value extraction.  
  - *Configuration:*  
    - Language: Python  
    - Code logic:  
      - Converts `rowIndex` to integer.  
      - Iterates over keys of `myData`.  
      - When the countdown reaches zero, appends the key and its value to a list `ans`.  
      - Returns the first matched key-value pair as a JSON array under `result`.  
  - *Key Expressions/Variables:*  
    - `_input.first().json.rowIndex` — index input  
    - `_input.first().json.myData` — JSON object to extract from  
  - *Input/Output:*  
    - Input: JSON object from "Input JSON Node"  
    - Output: JSON with structure `{ "result": [key, value] }`  
  - *Edge Cases & Failures:*  
    - Non-integer or missing `rowIndex` may cause conversion error.  
    - Index out of bounds returns empty or no result (no explicit error handling).  
    - If `myData` is empty or not an object, may fail or return empty result.  
  - *Version Requirements:* Python environment available (typeVersion 2 for code node).  
  - *Notes:* Returns a single key-value pair as a two-element array.

#### 1.3 Output Structuring

**Overview:**  
Separates the extracted key and value into distinct data fields for further usage or clarity.

**Nodes Involved:**  
- Key  
- Value

**Node Details:**  

- **Key**  
  - *Type & Role:* Set node; extracts the key (first element of `result` array).  
  - *Configuration:*  
    - Assigns `result` field to `{{$json.result[0]}}` (the key string).  
    - Options set to avoid dot notation and ignore conversion errors.  
  - *Input/Output:*  
    - Input: Output from "Find Key-Value Pair"  
    - Output: JSON with `result` key holding the extracted key string.  
  - *Edge Cases:*  
    - If `result` is empty or malformed, `result` field may be undefined.  

- **Value**  
  - *Type & Role:* Set node; extracts the value (second element of `result` array).  
  - *Configuration:*  
    - Assigns `result[1]` to `{{$json.result[1]}}` (the value string).  
    - Options set to avoid dot notation and ignore conversion errors.  
  - *Input/Output:*  
    - Input: Output from "Find Key-Value Pair"  
    - Output: JSON with `result[1]` holding the extracted value string.  
  - *Edge Cases:*  
    - If no value exists at index 1, result may be undefined or empty.  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                     | Input Node(s)                 | Output Node(s)           | Sticky Note |
|---------------------------|---------------------|-----------------------------------|------------------------------|--------------------------|-------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Initiates workflow execution       |                              | Input JSON Node           |             |
| Input JSON Node           | Set                 | Provides JSON data and index input | When clicking ‘Test workflow’ | Find Key-Value Pair       |             |
| Find Key-Value Pair       | Code (Python)       | Extracts key-value pair by index   | Input JSON Node              | Key, Value                |             |
| Key                       | Set                 | Extracts the key string            | Find Key-Value Pair           |                          |             |
| Value                     | Set                 | Extracts the value string          | Find Key-Value Pair           |                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually.

2. **Create a Set Node for Input JSON**  
   - Name: `Input JSON Node`  
   - Mode: Raw JSON  
   - JSON Content:
     ```json
     {
       "myData": {
         "name": "Alice",
         "age": "30",
         "city": "Paris"
       },
       "rowIndex": "1"
     }
     ```  
   - Connect the output of the Manual Trigger node to this node.

3. **Create a Code Node for Extraction Logic**  
   - Name: `Find Key-Value Pair`  
   - Language: Python  
   - Code:  
     ```python
     import json
     ans=[]
     ind = (_input.first().json.rowIndex)
     ind=int(ind)

     for i in _input.first().json.myData:
       if ind==0:
         ans.append(i)
         ans.append(_input.first().json.myData[i])
         break
       ind=ind-1

     return [{"json": {"result": ans}}]
     ```  
   - Connect the output of the Input JSON Node to this node.

4. **Create a Set Node to Extract the Key**  
   - Name: `Key`  
   - Assignments:  
     - Field: `result`  
     - Value: `={{ $json.result[0] }}` (expression)  
   - Options: Disable dot notation, ignore conversion errors.  
   - Connect the output of the Code node to this node.

5. **Create a Set Node to Extract the Value**  
   - Name: `Value`  
   - Assignments:  
     - Field: `result[1]`  
     - Value: `={{ $json.result[1] }}` (expression)  
   - Options: Disable dot notation, ignore conversion errors.  
   - Connect the output of the Code node to this node.

6. **Save and Test the Workflow**  
   - Execute manually to verify the output key and value correspond to the specified index.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                       |
|-------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The Python code node requires a Python environment configured in n8n for execution. | n8n documentation on [Code node usage](https://docs.n8n.io/nodes/n8n-nodes-base.code/)                |
| The workflow assumes zero-based indexing for the key-value pair extraction.    | Important when setting `rowIndex` value in the Input JSON node.                                      |
| This approach is useful when keys are not known in advance or dynamic selection is needed. | Ideal for flexible JSON data processing tasks.                                                       |

---

*Disclaimer: The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*