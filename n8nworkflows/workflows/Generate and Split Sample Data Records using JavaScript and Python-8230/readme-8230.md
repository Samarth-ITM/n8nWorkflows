Generate and Split Sample Data Records using JavaScript and Python

https://n8nworkflows.xyz/workflows/generate-and-split-sample-data-records-using-javascript-and-python-8230


# Generate and Split Sample Data Records using JavaScript and Python

### 1. Workflow Overview

This workflow is designed to generate sample data records programmatically within n8n using two different scripting languages: JavaScript and Python. After generating a fixed set of sample records, it splits the array of records into individual items for further processing or testing. This approach is ideal for scenarios requiring mock data generation without relying on external data sources, useful for prototyping, testing, or developing list-based logic such as pagination or mapping.

The workflow is logically organized into these blocks:

- **1.1 Workflow Trigger:** Manual initiation of the workflow.
- **1.2 Data Generation:** Two parallel blocks generating sample data arrays, one using JavaScript and the other Python.
- **1.3 Data Splitting:** Splitting the generated arrays into individual records for downstream consumption.
- **1.4 Documentation:** Sticky notes providing context and instructions.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Workflow Trigger

- **Overview:**  
  Initiates the workflow manually to allow user-controlled execution and testing.

- **Nodes Involved:**  
  - Start Workflow

- **Node Details:**  
  - **Start Workflow**  
    - Type: Manual Trigger  
    - Configuration: No parameters; triggers workflow on user action.  
    - Input: None  
    - Output: Triggers both the JavaScript and Python data generation nodes simultaneously.  
    - Edge Cases: None typical; failure unlikely unless n8n internal error occurs.

---

#### 1.2 Data Generation

- **Overview:**  
  Creates an array of 20 sample records with fields `index`, `num`, and `test`. Two parallel implementations demonstrate how to generate the same data using JavaScript and Python code nodes.

- **Nodes Involved:**  
  - Generate Data Javascript  
  - Generate Data Python

- **Node Details:**

  - **Generate Data Javascript**  
    - Type: Code Node (JavaScript)  
    - Configuration:  
      - Runs JavaScript code to generate an array of 20 objects.  
      - Each object contains:  
        - `index`: loop index (0 to 19)  
        - `num`: index + 1 (1 to 20)  
        - `test`: static string `"asdasd"`  
      - The array is assigned to `item.json.barr`.  
      - Returns the modified item.  
    - Input: Trigger from Start Workflow, no prior data required.  
    - Output: Single item with `barr` array of sample records.  
    - Edge Cases:  
      - Expression failure unlikely unless syntax error introduced.  
      - Large arrays may affect performance but not relevant here.  
      - Ensures returning array of items with correct structure.  

  - **Generate Data Python**  
    - Type: Code Node (Python)  
    - Configuration:  
      - Executes Python code generating a list of dictionaries with the same structure and data as the JavaScript node.  
      - Returns a list containing one item with the key `json.barr` holding the array.  
    - Input: Trigger from Start Workflow.  
    - Output: Single item with `barr` array.  
    - Requirements: Python environment configured in n8n instance.  
    - Edge Cases:  
      - Python runtime errors if environment not set up properly.  
      - Same data consistency considerations as JavaScript node.

---

#### 1.3 Data Splitting

- **Overview:**  
  Splits the generated array of sample records (`barr`) into individual items, fanning out one record per item. This enables downstream nodes to process items individually.

- **Nodes Involved:**  
  - Split Out Javascript  
  - Split Out Python

- **Node Details:**

  - **Split Out Javascript**  
    - Type: Split Out Node  
    - Configuration:  
      - Field to split out: `barr` (array of records from JavaScript code node)  
      - No special options configured.  
    - Input: Single item with array from Generate Data Javascript node.  
    - Output: 20 individual items, one per record in `barr`.  
    - Edge Cases:  
      - If `barr` is missing or empty, output will be empty or cause errors downstream.  
      - Should validate existence of field before splitting in complex workflows.  

  - **Split Out Python**  
    - Type: Split Out Node  
    - Configuration:  
      - Field to split out: `barr` (from Python code node)  
    - Input: Single item with array from Generate Data Python node.  
    - Output: 20 individual items.  
    - Edge Cases: Same as JavaScript split out.

---

#### 1.4 Documentation (Sticky Notes)

- **Overview:**  
  Provides users with explanation, purpose, and credits related to the workflow.

- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note10

- **Node Details:**

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content:  
      Explains the workflow purpose: generating sample data with code nodes (JavaScript and Python) and splitting the data into individual items. Emphasizes plug-and-play simplicity and use cases like prototyping or testing.  
    - Positioning: Visually groups the workflow context on the canvas.

  - **Sticky Note10**  
    - Type: Sticky Note  
    - Content:  
      Details what the template does: generates 20 sample records with specified fields, writes to `item.json.barr`, splits the array into individual items, and provides both JavaScript and Python versions.  
      Also includes author contact and professional links:  
      - Email: rbreen@ynteractive.com  
      - LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/  
      - Website: https://ynteractive.com  
    - Positioning: Near the workflow start for immediate user reference.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role             | Input Node(s)        | Output Node(s)            | Sticky Note                                                                                          |
|-----------------------|---------------------|----------------------------|----------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Start Workflow        | Manual Trigger      | Initiates the workflow      | None                 | Generate Data Javascript, Generate Data Python |                                                                                                    |
| Generate Data Javascript | Code (JavaScript)  | Generates sample data array | Start Workflow       | Split Out Javascript       |                                                                                                    |
| Generate Data Python  | Code (Python)        | Generates sample data array | Start Workflow       | Split Out Python           |                                                                                                    |
| Split Out Javascript  | Split Out            | Splits JS array into items  | Generate Data Javascript | None                    |                                                                                                    |
| Split Out Python      | Split Out            | Splits Python array into items | Generate Data Python | None                      |                                                                                                    |
| Sticky Note1          | Sticky Note          | Workflow purpose and use cases | None              | None                      | Create sample records with Code and split to items in n8n (Code + Split Out). Minimal, plug-and-play workflow. |
| Sticky Note10         | Sticky Note          | Template description, author info | None              | None                      | Generates 20 sample records with fields `index`, `num`, `test`. Includes JavaScript and Python versions. Contact: rbreen@ynteractive.com. https://www.linkedin.com/in/robert-breen-29429625/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `Start Workflow`  
   - Type: Manual Trigger  
   - No parameters needed. This node will serve as the starting point.

2. **Create JavaScript Code Node:**  
   - Name: `Generate Data Javascript`  
   - Type: Code  
   - Language: JavaScript (default)  
   - Code:  
     ```js
     // Generate 20 sample records with fields index, num, test
     return items.map(item => {
       const arr = [];
       for (let i = 0; i < 20; i++) {
         arr.push({
           index: i,
           num: i + 1,
           test: "asdasd"
         });
       }
       item.json.barr = arr;
       return item;
     });
     ```
   - Connect `Start Workflow` output to this node input.

3. **Create Python Code Node:**  
   - Name: `Generate Data Python`  
   - Type: Code  
   - Language: Python  
   - Code:  
     ```python
     arr = [{"index": i, "num": i + 1, "test": "asdasd"} for i in range(20)]
     return [{"json": {"barr": arr}}]
     ```
   - Connect `Start Workflow` output to this node input as well (parallel).

4. **Create Split Out Node for JavaScript:**  
   - Name: `Split Out Javascript`  
   - Type: Split Out  
   - Set `Field to Split Out` to `barr`  
   - Connect output of `Generate Data Javascript` to this node.

5. **Create Split Out Node for Python:**  
   - Name: `Split Out Python`  
   - Type: Split Out  
   - Set `Field to Split Out` to `barr`  
   - Connect output of `Generate Data Python` to this node.

6. **Add Sticky Notes (optional but recommended):**  
   - Add a sticky note near the workflow start explaining the purpose and use case, e.g.:  
     "Generate sample data with JavaScript and Python code nodes, then split into individual items for testing and prototyping."  
   - Add another sticky note citing author info and summary of the workflow.

7. **Save Credentials:**  
   - No external credentials required for this workflow.  
   - Ensure Python environment is properly configured in n8n instance for Python Code node to execute.

8. **Execute:**  
   - Trigger the workflow manually via `Start Workflow`.  
   - Verify the outputs of both Split Out nodes produce 20 individual items each with correct fields.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                       |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow demonstrates a minimal and modular approach to generating and splitting sample data items without external dependencies. | Workflow description and use case.                                  |
| Author and contact: Robert Breen, rbreen@ynteractive.com                                        | Contact for support or questions.                                   |
| LinkedIn profile: https://www.linkedin.com/in/robert-breen-29429625/                            | Professional networking and further info.                           |
| Website: https://ynteractive.com                                                                | Company website for additional resources.                           |
| Python Code node requires Python runtime configured in n8n environment                          | n8n server or desktop must have Python environment for Python nodes.|

---

**Disclaimer:** This text is exclusively generated from an n8n automated workflow and complies strictly with current content policies. It contains no illegal, offensive, or protected elements. All manipulated data is legal and public.