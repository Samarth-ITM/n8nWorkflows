Create Nested Data Processing Loops Using n8n Sub-workflows

https://n8nworkflows.xyz/workflows/create-nested-data-processing-loops-using-n8n-sub-workflows-8556


# Create Nested Data Processing Loops Using n8n Sub-workflows

### 1. Workflow Overview

This n8n workflow demonstrates how to implement **nested loops using sub-workflows**, overcoming the limitation that n8n’s native loop nodes cannot reliably nest. It is designed for scenarios where each item in one list must be processed in combination with every item of another list, a common requirement when handling related data sets.

The workflow consists of two main logical blocks:

- **1.1 Main Workflow: Outer Loop Over Colors**  
  This block initiates the workflow manually, generates a list of colors, and iterates over them one by one. For each color, it triggers a sub-workflow responsible for the inner loop.

- **1.2 Sub-Workflow: Inner Loop Over Integers**  
  Triggered per color, this sub-workflow receives the color as input, generates a list of integers, loops through each integer, and combines it with the color to produce a concatenated result string.

This modular structure ensures clean, maintainable workflows that can be extended or adapted for other nested iteration use cases.

---

### 2. Block-by-Block Analysis

#### 2.1 Main Workflow: Outer Loop Over Colors

**Overview:**  
Starts with manual triggering, generates an array of colors, loops over each color, and invokes the sub-workflow for each color item.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Colors (Code node)  
- Loop Over Colors (SplitInBatches node)  
- Execute Sub-workflow (Execute Workflow node)  
- Sticky Note (Documentation only)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution  
  - *Config:* No parameters, simply triggers the workflow on demand  
  - *Connections:* Outputs to Colors node  
  - *Edge Cases:* None expected; user must manually trigger

- **Colors**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates a static list of color objects: [{color: "yellow"}, {color: "blue"}, {color: "green"}]  
  - *Config:* Inline JS code returning an array of JSON objects with a "color" property  
  - *Connections:* Receives input from manual trigger, outputs to Loop Over Colors  
  - *Edge Cases:* If JS fails due to syntax, workflow halts; data is static so low risk

- **Loop Over Colors**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over each color one at a time (batch size defaults to 1)  
  - *Config:* Default options, no additional batching parameters set  
  - *Connections:* Input from Colors, outputs to Execute Sub-workflow (second output) and main output unconnected  
  - *Edge Cases:* Large lists may slow execution; no batch size set explicitly; empty input halts downstream nodes

- **Execute Sub-workflow**  
  - *Type:* Execute Workflow  
  - *Role:* Calls the sub-workflow for each color item, passing the color as input  
  - *Config:*  
    - Workflow linked to the same workflow ID (self-referential) but triggered via trigger node  
    - Input mapping defines parameter "color" passed as string from current color JSON value  
  - *Connections:* Input from Loop Over Colors, output loops back to Loop Over Colors for batch continuation  
  - *Version Requirements:* 1.2 minimum for input mapping features  
  - *Edge Cases:*  
    - Sub-workflow must exist and be accessible; misconfiguration causes errors  
    - Passing incorrect or missing parameters causes failures in sub-workflow  
    - Execution errors inside sub-workflow bubble up here

- **Sticky Note**  
  - *Type:* Sticky Note (Documentation)  
  - *Role:* Explains the template’s purpose, usage instructions, and links to a blog post  
  - *Content:*  
    - Explains challenge of nested loops in n8n  
    - Describes how this template uses sub-workflows as workaround  
    - Setup instructions for copying and linking sub-workflows  
    - Link: https://n8nplaybook.com/post/2025/07/how-to-handle-nested-loops-in-n8n-with-sub-workflows/  
  - *Connections:* None; purely informational

#### 2.2 Sub-Workflow: Inner Loop Over Integers

**Overview:**  
Triggered by the main workflow per color, this sub-workflow generates integers, loops over them, and combines each integer with the color input to produce a result string.

**Nodes Involved:**  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Integers (Code node)  
- Loop Over Integers (SplitInBatches node)  
- Edit Fields (Set node)

**Node Details:**  

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry node for sub-workflow, receives "color" as input parameter  
  - *Config:* Defines input schema with a single parameter "color" (string)  
  - *Connections:* Outputs to Integers node  
  - *Edge Cases:*  
    - Must be triggered by Execute Workflow node from main workflow  
    - Missing or malformed input "color" causes failures downstream

- **Integers**  
  - *Type:* Code (JavaScript)  
  - *Role:* Returns a static list of integers [{number:1}, {number:2}, {number:3}]  
  - *Config:* Inline JavaScript returning the array  
  - *Connections:* Input from trigger node, output to Loop Over Integers  
  - *Edge Cases:* Static data, low risk; code errors cause failure

- **Loop Over Integers**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over each integer one at a time  
  - *Config:* Default batch size (1), no advanced options  
  - *Connections:* Input from Integers node, outputs to Edit Fields (second output) and main output unconnected  
  - *Edge Cases:* Empty input array will halt downstream nodes

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Combines the input color and current integer into a single "result" string  
  - *Config:*  
    - Creates a new string field "result" using expression: `{{ $('When Executed by Another Workflow').item.json.color }} - {{ $('Loop Over Integers').item.json.number }}`  
  - *Connections:* Input from Loop Over Integers, output loops back to Loop Over Integers for batch continuation  
  - *Edge Cases:* Expressions depend on correct input data; missing inputs cause empty or error results

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                            | Input Node(s)                     | Output Node(s)               | Sticky Note                                                                                       |
|-----------------------------|-------------------------|--------------------------------------------|----------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Entry point to start the main workflow     | —                                | Colors                      |                                                                                                 |
| Colors                      | Code                    | Generates list of colors                    | When clicking ‘Execute workflow’ | Loop Over Colors             |                                                                                                 |
| Loop Over Colors             | SplitInBatches          | Loops over each color                       | Colors                           | Execute Sub-workflow         |                                                                                                 |
| Execute Sub-workflow         | Execute Workflow        | Calls sub-workflow per color, passes input | Loop Over Colors                 | Loop Over Colors             | Explains nested loops pattern and setup instructions with blog link (https://n8nplaybook.com/post/2025/07/how-to-handle-nested-loops-in-n8n-with-sub-workflows/) |
| Sticky Note                 | Sticky Note             | Documentation and instructions             | —                                | —                           | Explains the template’s purpose and usage instructions with blog link (https://n8nplaybook.com/post/2025/07/how-to-handle-nested-loops-in-n8n-with-sub-workflows/) |
| When Executed by Another Workflow | Execute Workflow Trigger | Entry point of sub-workflow, receives color | Execute Sub-workflow             | Integers                    |                                                                                                 |
| Integers                    | Code                    | Generates list of integers                  | When Executed by Another Workflow | Loop Over Integers           |                                                                                                 |
| Loop Over Integers           | SplitInBatches          | Loops over each integer                     | Integers                        | Edit Fields                 |                                                                                                 |
| Edit Fields                 | Set                     | Combines color and integer into result string | Loop Over Integers             | Loop Over Integers           |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** (this will be the main workflow).

2. **Add a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

3. **Add a Code node**  
   - Name: `Colors`  
   - Paste this JavaScript code to return colors:  
     ```js
     return [
       { "color": "yellow" },
       { "color": "blue" },
       { "color": "green" }
     ];
     ```  
   - Connect Manual Trigger node output to this Code node input.

4. **Add a SplitInBatches node**  
   - Name: `Loop Over Colors`  
   - No special parameters; default batch size 1 (processes one color at a time).  
   - Connect Colors node output to this node input.

5. **Create the sub-workflow** in a new, separate workflow tab or canvas:  

   - **Add an Execute Workflow Trigger node**  
     - Name: `When Executed by Another Workflow`  
     - Define workflow input parameter:  
       - Name: `color`  
       - Type: String  
     - No default value necessary.

   - **Add a Code node**  
     - Name: `Integers`  
     - Paste this JavaScript code to return integers:  
       ```js
       return [
         { "number": 1 },
         { "number": 2 },
         { "number": 3 }
       ];
       ```  
     - Connect Execute Workflow Trigger node output to this node input.

   - **Add a SplitInBatches node**  
     - Name: `Loop Over Integers`  
     - Default batch size (1).  
     - Connect Integers node output to this node input.

   - **Add a Set node**  
     - Name: `Edit Fields`  
     - Add a string field assignment:  
       - Field Name: `result`  
       - Value (expression mode):  
         ```
         {{$json["color"]}} - {{$json["number"]}}
         ```  
       - Use expression editor or reference nodes:  
         `{{ $('When Executed by Another Workflow').item.json.color }} - {{ $('Loop Over Integers').item.json.number }}`  
     - Connect SplitInBatches node second output (batch continuation) to this Set node input.  
     - Connect Set node output back to SplitInBatches node input for batch continuation.

6. **Back in the main workflow**, add an Execute Workflow node:  
   - Name: `Execute Sub-workflow`  
   - Set to execute the sub-workflow created in step 5.  
   - Configure input mapping:  
     - Pass current color: Map parameter `color` to expression `{{$json.color}}` from the current item.  
   - Connect Loop Over Colors node second output (batch continuation) to this Execute Workflow node input.  
   - Connect Execute Workflow node output back to Loop Over Colors node input for batch continuation.

7. **(Optional) Add a Sticky Note** on the main workflow canvas:  
   - Add description explaining the purpose of the template and instructions, including the blog link:  
     https://n8nplaybook.com/post/2025/07/how-to-handle-nested-loops-in-n8n-with-sub-workflows/

8. **Activate and run** the main workflow by clicking "Execute workflow" on the Manual Trigger node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| This template solves the nested loop problem in n8n by using sub-workflows to perform inner loops reliably. It demonstrates how to modularize complex iterations cleanly.                                                                               | Template purpose and design rationale                                                                           |
| For a detailed explanation and step-by-step walkthrough, visit the blog post: [How to Handle Nested Loops in n8n with Sub-workflows](https://n8nplaybook.com/post/2025/07/how-to-handle-nested-loops-in-n8n-with-sub-workflows/)                         | https://n8nplaybook.com/post/2025/07/how-to-handle-nested-loops-in-n8n-with-sub-workflows/                       |
| When reproducing this workflow, ensure the sub-workflow is saved and accessible, and inputs/outputs are correctly mapped to avoid runtime errors.                                                                                                   | Workflow setup requirement                                                                                       |
| The SplitInBatches node batch size defaults to 1 here to simulate looping; this can be adjusted depending on performance needs and data size.                                                                                                        | Performance consideration                                                                                        |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.