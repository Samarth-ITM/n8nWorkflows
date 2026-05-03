XOR Encryption and Decryption with Base64 Encoding for Workflow Data

https://n8nworkflows.xyz/workflows/xor-encryption-and-decryption-with-base64-encoding-for-workflow-data-5025


# XOR Encryption and Decryption with Base64 Encoding for Workflow Data

### 1. Workflow Overview

This workflow provides a simple XOR-based encryption and decryption mechanism with Base64 encoding/decoding for data transmitted between workflows in n8n. It is designed to securely encode and decode string data using a symmetric key, supporting two main actions: "encrypt" and "decrypt". The workflow is triggered externally by another workflow, receiving action type, key, and data as inputs.

**Logical Blocks:**

- **1.1 Input Reception:** Receives parameters from an external workflow to determine the action type ("encrypt" or "decrypt"), the key, and the data to process.
- **1.2 Conditional Routing:** Routes the execution flow based on the action type.
- **1.3 Encryption Processing:** Performs XOR encryption of the input data with the key and encodes the result in Base64.
- **1.4 Decryption Processing:** Decodes Base64 input data and performs XOR decryption with the key to recover the original string.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for an execution call from another workflow, accepting three input parameters: "action-type", "key", and "data". This node acts as the entry point for external triggers.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger (n8n-nodes-base.executeWorkflowTrigger)  
    - *Role:* Entry point triggered remotely by another workflow execution.  
    - *Configuration:*  
      - Workflow inputs defined: "action-type", "key", and "data".  
    - *Key Expressions / Variables:*  
      - Receives `action-type`, `key`, and `data` from external workflow input.  
    - *Input/Output:*  
      - No input connections (start node).  
      - Outputs to the "If" node for conditional routing.  
    - *Edge Cases / Failure Types:*  
      - Missing or malformed input parameters may cause downstream logic failures.  
      - Execution may fail if called improperly or if parameters are absent.  
    - *Sub-workflow:*  
      - This node is designed to be called by another workflow, not standalone.

---

#### 1.2 Conditional Routing

- **Overview:**  
  Evaluates the input parameter "action-type" to decide whether to perform encryption or decryption.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - *Type:* Conditional node (n8n-nodes-base.if)  
    - *Role:* Routes workflow execution based on the string equality check of "action-type".  
    - *Configuration:*  
      - Condition checks if `{{$json["action-type"]}}` equals `"encrypt"`.  
      - Uses strict type validation and case sensitivity.  
    - *Key Expressions / Variables:*  
      - `{{$json["action-type"]}}`  
    - *Input/Output:*  
      - Input from "When Executed by Another Workflow".  
      - If true (action-type = "encrypt"), routes to "encrypt" node.  
      - If false (any other action-type), routes to "decrypt" node.  
    - *Edge Cases / Failure Types:*  
      - If "action-type" is missing or empty, defaults to false branch (decrypt).  
      - Case sensitivity means "Encrypt" or "ENCRYPT" will not match.  
    - *Sub-workflow:*  
      - None.

---

#### 1.3 Encryption Processing

- **Overview:**  
  Performs XOR encryption on the input text with the provided key, then encodes the encrypted result in Base64.

- **Nodes Involved:**  
  - encrypt

- **Node Details:**

  - **encrypt**  
    - *Type:* Code node (n8n-nodes-base.code)  
    - *Role:* Runs custom JavaScript to XOR encrypt the input data and encode it as Base64.  
    - *Configuration:*  
      - Reads `data` and `key` from input JSON.  
      - Defines a function `xorEncrypt` that:  
        - Iterates over each character of the input text, XORs its char code with the key's corresponding char code (mod key length).  
        - Converts the resulting encrypted string to Base64 using `Buffer`.  
      - Returns JSON containing the encrypted Base64 string under `encrypted` key.  
    - *Key Expressions / Variables:*  
      - `const message = $input.first().json.data || "No Data";`  
      - `const key = $input.first().json.key;`  
    - *Input/Output:*  
      - Input: JSON with keys `data` and `key`.  
      - Output: JSON with key `encrypted`.  
    - *Edge Cases / Failure Types:*  
      - If `key` is missing or empty, encryption logic will error or yield incorrect results.  
      - If `data` is missing, defaults to "No Data".  
      - Encoding assumes UTF-8 text; special characters may cause unexpected results.  
      - Buffer usage requires Node.js environment compatibility (n8n standard).  
    - *Sub-workflow:*  
      - None.

---

#### 1.4 Decryption Processing

- **Overview:**  
  Accepts Base64-encoded XOR encrypted input, decodes Base64, and XOR decrypts using the same key to recover the original message.

- **Nodes Involved:**  
  - decrypt

- **Node Details:**

  - **decrypt**  
    - *Type:* Code node (n8n-nodes-base.code)  
    - *Role:* Runs JavaScript to decode Base64 input and XOR decrypt using the key.  
    - *Configuration:*  
      - Reads `data` and `key` from input JSON.  
      - Defines a function `xorDecrypt` that:  
        - Decodes the Base64 string to a text string.  
        - XORs each character with the key's corresponding char code.  
        - Returns the decrypted string.  
      - Returns JSON containing the decrypted text under `decrypted` key.  
    - *Key Expressions / Variables:*  
      - `const encrypted = $input.first().json.data;`  
      - `const key = $input.first().json.key;`  
    - *Input/Output:*  
      - Input: JSON with keys `data` (Base64 string) and `key`.  
      - Output: JSON with key `decrypted`.  
    - *Edge Cases / Failure Types:*  
      - Fails or returns garbage if `key` does not match the encryption key.  
      - If `data` is not valid Base64, decoding will throw an error.  
      - Empty or missing inputs may cause failure.  
      - Unicode or multibyte characters may cause unexpected results due to char code operations.  
    - *Sub-workflow:*  
      - None.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                         | Input Node(s)                  | Output Node(s)          | Sticky Note                                            |
|------------------------------|----------------------------------|---------------------------------------|-------------------------------|------------------------|--------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point receiving inputs           | None                          | If                     |                                                        |
| If                           | Conditional                      | Routes based on "action-type"          | When Executed by Another Workflow | encrypt (true branch), decrypt (false branch) |                                                        |
| encrypt                      | Code                             | XOR encrypts data with key and encodes | If (true branch)               | None                   |                                                        |
| decrypt                      | Code                             | Decodes Base64 and XOR decrypts data   | If (false branch)              | None                   |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node: When Executed by Another Workflow**  
   - Type: Execute Workflow Trigger  
   - Configure workflow inputs:  
     - `action-type` (string)  
     - `key` (string)  
     - `data` (string)  
   - Position the node as the starting point.  

2. **Create Node: If**  
   - Type: If (Conditional)  
   - Connect the output of "When Executed by Another Workflow" to this node.  
   - Set condition:  
     - Left value: Expression `{{$json["action-type"]}}`  
     - Operator: Equals  
     - Right value: `"encrypt"` (string literal)  
     - Use case sensitive and strict type validation.  

3. **Create Node: encrypt (Code)**  
   - Type: Code  
   - Connect the **true** output of the "If" node to this node.  
   - Paste the following JavaScript code:
     ```javascript
     const message = $input.first().json.data || "No Data";
     const key = $input.first().json.key;

     function xorEncrypt(text, key) {
       let encrypted = "";
       for (let i = 0; i < text.length; i++) {
         const charCode = text.charCodeAt(i) ^ key.charCodeAt(i % key.length);
         encrypted += String.fromCharCode(charCode);
       }
       return Buffer.from(encrypted).toString('base64'); 
     }

     return [
       {
         json: {
           encrypted: xorEncrypt(message, key)
         }
       }
     ];
     ```
   - No special credentials needed.  

4. **Create Node: decrypt (Code)**  
   - Type: Code  
   - Connect the **false** output of the "If" node to this node.  
   - Paste the following JavaScript code:
     ```javascript
     const encrypted = $input.first().json.data;
     const key = $input.first().json.key; // Must be the same key used to encrypt

     function xorDecrypt(base64Text, key) {
       const text = Buffer.from(base64Text, 'base64').toString();
       let decrypted = "";
       for (let i = 0; i < text.length; i++) {
         const charCode = text.charCodeAt(i) ^ key.charCodeAt(i % key.length);
         decrypted += String.fromCharCode(charCode);
       }
       return decrypted;
     }

     return [
       {
         json: {
           decrypted: xorDecrypt(encrypted, key)
         }
       }
     ];
     ```
   - No special credentials needed.  

5. **Set Workflow Settings:**  
   - Execution order: default (v1).  
   - Save and activate workflow as needed.  

6. **Usage:**  
   - Trigger this workflow from another workflow using the Execute Workflow node.  
   - Provide inputs as JSON with keys: `action-type` ("encrypt" or "decrypt"), `key` (string), and `data` (string).

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                            |
|----------------------------------------------------------------------------------------------|------------------------------------------------------------|
| XOR encryption is a simple symmetric cipher but not secure for sensitive data in production. | Use only for basic obfuscation or learning purposes.       |
| The code uses Node.js Buffer for Base64 encoding/decoding, standard in n8n environment.      | n8n documentation: https://docs.n8n.io/nodes/expressions/  |
| When using this workflow, ensure the key is consistent between encryption and decryption.    | Otherwise, decryption will fail or yield incorrect data.   |

---

**Disclaimer:** The provided description and workflow are exclusively derived from an automated n8n workflow, compliant with all applicable content policies. No illegal, offensive, or protected content is included.