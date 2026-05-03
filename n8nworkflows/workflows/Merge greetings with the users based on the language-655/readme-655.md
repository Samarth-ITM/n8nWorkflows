Merge greetings with the users based on the language

https://n8nworkflows.xyz/workflows/merge-greetings-with-the-users-based-on-the-language-655


# Merge greetings with the users based on the language

---

### 1. Workflow Overview

This workflow demonstrates how to merge two datasets based on a common field in n8n. It is designed for users interested in learning how to combine items programmatically within n8n workflows. The workflow consists of the following logical blocks:

- **1.1 Input Trigger**: A manual trigger to start the workflow.
- **1.2 Sample Data Generation**: Two separate code nodes generating sample datasets — one with user names and their languages, another with greetings and their languages.
- **1.3 Data Merging**: A merge node that combines the two datasets on the shared "language" field, resulting in unified records containing name, language, and greeting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block initiates the workflow manually when the user clicks the ‘Test workflow’ button in n8n. It starts the entire data generation and merging process.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Test workflow’  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters are set; it simply waits for manual activation.  
  - **Expressions/Variables:** None  
  - **Input Connections:** None (start node)  
  - **Output Connections:** Outputs to both sample data code nodes simultaneously.  
  - **Version-Specific Requirements:** None  
  - **Potential Failures:** Unlikely to fail; manual trigger depends on user interaction.  
  - **Sub-Workflow Reference:** None

#### 1.2 Sample Data Generation

- **Overview:**  
  Generates two sets of sample JSON data using JavaScript code nodes. One dataset contains user names paired with their language codes; the other contains greetings with corresponding language codes.

- **Nodes Involved:**  
  - Sample data (name + language)  
  - Sample data (greeting + language)

- **Node Details:**  

  1. **Node Name:** Sample data (name + language)  
     - **Type:** Code Node (JavaScript)  
     - **Configuration:** Returns an array of JSON objects, each with “name” and “language” properties.  
     - **Key Expressions:**  
       ```js
       return [
         { json: { name: 'Stefan', language: 'de' } },
         { json: { name: 'Jim', language: 'en' } },
         { json: { name: 'Hans', language: 'de' } }
       ];
       ```  
     - **Input Connections:** From manual trigger  
     - **Output Connections:** To Merge node (input 0)  
     - **Version-Specific Requirements:** Code node version 2 to support async/await (not used here, but recommended)  
     - **Failure Cases:** Code syntax errors if modified incorrectly; empty output if return statement missing  
     - **Sub-Workflow Reference:** None

  2. **Node Name:** Sample data (greeting + language)  
     - **Type:** Code Node (JavaScript)  
     - **Configuration:** Returns an array of JSON objects with “greeting” and “language” properties.  
     - **Key Expressions:**  
       ```js
       return [
         { json: { greeting: 'Hello', language: 'en' } },
         { json: { greeting: 'Hallo', language: 'de' } }
       ];
       ```  
     - **Input Connections:** From manual trigger  
     - **Output Connections:** To Merge node (input 1)  
     - **Version-Specific Requirements:** Code node version 2  
     - **Failure Cases:** Same as above  
     - **Sub-Workflow Reference:** None

#### 1.3 Data Merging

- **Overview:**  
  This block merges the two datasets based on the "language" field. The merge mode is set to combine matching entries from both inputs, creating a single output item per language with combined fields.

- **Nodes Involved:**  
  - Merge (name + language + greeting)

- **Node Details:**  
  - **Node Name:** Merge (name + language + greeting)  
  - **Type:** Merge Node  
  - **Configuration:**  
    - Mode: Combine  
    - Fields to Match: `language`  
    - No additional options set  
  - **Key Expressions:** Matching is done on the string field `language`.  
  - **Input Connections:**  
    - Input 0 from "Sample data (name + language)"  
    - Input 1 from "Sample data (greeting + language)"  
  - **Output Connections:** None (end node)  
  - **Version-Specific Requirements:** Merge node version 3 supports mode ‘combine’ and field matching by string.  
  - **Failure Cases:**  
    - Mismatched or missing “language” fields could cause incomplete merges.  
    - If inputs are empty, output will be empty.  
    - Data type mismatches on “language” field (e.g., string vs number) could cause unexpected results.  
  - **Sub-Workflow Reference:** None

---

### 3. Summary Table

| Node Name                      | Node Type      | Functional Role             | Input Node(s)                   | Output Node(s)                  | Sticky Note                          |
|-------------------------------|----------------|----------------------------|--------------------------------|--------------------------------|------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger | Start workflow             | None                           | Sample data (name + language), Sample data (greeting + language) |                                    |
| Sample data (name + language)  | Code           | Generate sample names/language data | When clicking ‘Test workflow’ | Merge (name + language + greeting) |                                    |
| Sample data (greeting + language) | Code        | Generate sample greetings/language data | When clicking ‘Test workflow’ | Merge (name + language + greeting) |                                    |
| Merge (name + language + greeting) | Merge       | Merge datasets on language | Sample data (name + language), Sample data (greeting + language) | None                           | Merges two datasets by the “language” field |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name it: `When clicking ‘Test workflow’`  
   - No additional configuration needed.

2. **Create a Code node for names and languages**  
   - Name it: `Sample data (name + language)`  
   - Set the code language to JavaScript.  
   - Paste the following code in the code editor:
     ```js
     return [
       { json: { name: 'Stefan', language: 'de' } },
       { json: { name: 'Jim', language: 'en' } },
       { json: { name: 'Hans', language: 'de' } }
     ];
     ```  
   - Connect `When clicking ‘Test workflow’` node's output to this node's input.

3. **Create a Code node for greetings and languages**  
   - Name it: `Sample data (greeting + language)`  
   - Set the code language to JavaScript.  
   - Paste the following code:
     ```js
     return [
       { json: { greeting: 'Hello', language: 'en' } },
       { json: { greeting: 'Hallo', language: 'de' } }
     ];
     ```  
   - Connect `When clicking ‘Test workflow’` node's output to this node's input (parallel to previous code node).

4. **Create a Merge node**  
   - Name it: `Merge (name + language + greeting)`  
   - Set the mode to `Combine`.  
   - In “Fields to Match String”, enter `language`.  
   - Connect output of `Sample data (name + language)` node to input 0 of the Merge node.  
   - Connect output of `Sample data (greeting + language)` node to input 1 of the Merge node.

5. **No credentials setup is required** for this workflow as it uses only built-in nodes without external API calls.

6. **Save and test the workflow** by clicking the “Execute Workflow” or “Test Workflow” button.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                 |
|------------------------------------------------------------------------------|----------------------------------------------------------------|
| This workflow is a clear example of merging datasets on a shared key field, useful for learning n8n’s Merge node capabilities. | --                                                             |
| For more about combining data in n8n, see the official docs: https://docs.n8n.io/nodes/n8n-nodes-base.merge/ | Official n8n Merge Node Documentation                           |

---