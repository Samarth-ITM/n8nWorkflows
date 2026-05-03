Tip #1: Testing n8n sub-workflow

https://n8nworkflows.xyz/workflows/tip--1--testing-n8n-sub-workflow-5032


# Tip #1: Testing n8n sub-workflow

### 1. Workflow Overview

This workflow demonstrates a testing setup for an n8n sub-workflow, designed to illustrate how to structure inputs and conditional logic to facilitate testing both manual execution and invocation by another workflow. It is primarily a template and example showing how to organize input handling and decision-making based on input data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Handles two modes of triggering the workflow — manual execution and execution triggered by another workflow, each providing input data.
- **1.2 Input Preparation**: Normalizes and combines inputs into a unified format for further processing.
- **1.3 Conditional Logic**: Evaluates input conditions to determine workflow branching.
- **1.4 Documentation**: Provides an embedded sticky note with usage tips and external resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview**  
This block configures two entry points for the workflow: one triggered manually by the user, and one triggered by an external workflow execution. It ensures the workflow can be tested independently or invoked programmatically with inputs.

**Nodes Involved**  
- When clicking ‘Execute workflow’  
- When Executed by Another Workflow

**Node Details**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing, allowing the workflow to start on demand.  
  - Configuration: No parameters; simply triggers execution on user click.  
  - Inputs: None  
  - Outputs: Connected to "Test Input" node.  
  - Edge Cases: None significant; user must manually trigger.  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point when called by another workflow, receiving inputs dynamically.  
  - Configuration: Expects input parameter named `color`.  
  - Inputs: None (triggered externally)  
  - Outputs: Connected to "Combine Input" node.  
  - Edge Cases: Failure if input `color` is missing or of wrong type; authentication or permission errors possible depending on n8n setup.

---

#### 2.2 Input Preparation

**Overview**  
This block standardizes input data regardless of trigger source, preparing it for conditional evaluation.

**Nodes Involved**  
- Test Input  
- Combine Input

**Node Details**

- **Test Input**  
  - Type: Set  
  - Role: Supplies a static test input when manually triggered.  
  - Configuration: Sets a single key `color` with value `"blue"`.  
  - Inputs: Connected from "When clicking ‘Execute workflow’"  
  - Outputs: Connected to "Combine Input"  
  - Edge Cases: Static data; no failures expected unless node misconfigured.

- **Combine Input**  
  - Type: Set  
  - Role: Combines data inputs from both triggers to unify the workflow input format.  
  - Configuration:  
    - Includes other fields from previous nodes to preserve incoming data.  
    - No additional fields assigned explicitly, acts as a pass-through and normalizer.  
  - Inputs: Connected from both "Test Input" and "When Executed by Another Workflow" (merging different paths)  
  - Outputs: Connected to "If" node  
  - Edge Cases: Potential issues if inputs are empty or mismatched; data type consistency important.

---

#### 2.3 Conditional Logic

**Overview**  
Evaluates the prepared input to conditionally branch workflow execution based on the value of `color`.

**Nodes Involved**  
- If

**Node Details**

- **If**  
  - Type: If  
  - Role: Implements conditional branching based on input data.  
  - Configuration:  
    - Checks if the field `color` from "Combine Input" equals exactly `"blue"`.  
    - Case sensitive and strict type validation enabled.  
  - Inputs: Connected from "Combine Input"  
  - Outputs: Two branches — `true` if condition met, `false` otherwise (though not connected further here).  
  - Key Expression: `={{ $('Combine Input').item.json.color }}`  
  - Edge Cases:  
    - Condition fails if `color` is missing or not a string.  
    - Expression failure if referenced node output is unavailable.  
    - Case sensitivity may cause unexpected false negatives.

---

#### 2.4 Documentation

**Overview**  
Provides embedded documentation and usage tips directly within the workflow canvas for ease of reference.

**Nodes Involved**  
- Sticky Note

**Node Details**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Contains explanatory text and a link to an external blog for more details.  
  - Configuration:  
    - Width: 400px, Height: 120px  
    - Content:  
      ```
      ## Tip #1: Testing n8n sub-workflow
      This is the template with example of how you can organize your sub-workflow to be able to test it.
      More details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-1-testing-n8n-sub-workflow.html)
      ```
  - Inputs: None  
  - Outputs: None  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                         | Input Node(s)                      | Output Node(s)            | Sticky Note                                                                                                   |
|-----------------------------|----------------------------|---------------------------------------|----------------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Manual workflow start trigger          | None                             | Test Input                |                                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger   | Triggered when called by another workflow | None                             | Combine Input             |                                                                                                              |
| Test Input                  | Set                        | Supplies static test input data         | When clicking ‘Execute workflow’ | Combine Input             |                                                                                                              |
| Combine Input               | Set                        | Combines inputs from different triggers | Test Input, When Executed by Another Workflow | If                        |                                                                                                              |
| If                         | If                         | Conditional branching on input value    | Combine Input                   | None                      |                                                                                                              |
| Sticky Note                | Sticky Note                | Provides workflow documentation          | None                             | None                      | ## Tip #1: Testing n8n sub-workflow This is the template with example of how you can organize your sub-workflow to be able to test it. More details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-1-testing-n8n-sub-workflow.html) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger node**  
   - Add a node of type **Manual Trigger**  
   - Name it `When clicking ‘Execute workflow’`  
   - Leave parameters empty.  

2. **Create the Execute Workflow Trigger node**  
   - Add a node of type **Execute Workflow Trigger**  
   - Name it `When Executed by Another Workflow`  
   - Configure its input parameters: add one parameter named `color`  
   - Use version 1.1 or later to support workflowInputs.  

3. **Create the Test Input node**  
   - Add a node of type **Set**  
   - Name it `Test Input`  
   - Under "Values to Set", add a string field named `color` with value `"blue"`  
   - No other options needed.  

4. **Create the Combine Input node**  
   - Add a node of type **Set**  
   - Name it `Combine Input`  
   - Enable **Include Other Fields** option to true (to pass through all incoming data)  
   - Do not add new fields—this node acts as a data normalizer.  

5. **Create the If node**  
   - Add a node of type **If**  
   - Name it `If`  
   - Configure the condition as follows:  
     - Set condition operator to `equals`  
     - Left value expression: `={{ $('Combine Input').item.json.color }}`  
     - Right value: `blue`  
     - Enable case sensitivity and strict type validation.  

6. **Create the Sticky Note**  
   - Add a node of type **Sticky Note**  
   - Name it `Sticky Note`  
   - Set width: 400, height: 120  
   - Paste the following content:  
     ```
     ## Tip #1: Testing n8n sub-workflow
     This is the template with example of how you can organize your sub-workflow to be able to test it.
     More details in my [N8n Tips blog](https://n8n-tips.blogspot.com/2025/06/tip-1-testing-n8n-sub-workflow.html)
     ```  

7. **Connect the nodes**  
   - Connect `When clicking ‘Execute workflow’` output to `Test Input` input  
   - Connect `Test Input` output to `Combine Input` input  
   - Connect `When Executed by Another Workflow` output to `Combine Input` input (parallel connection)  
   - Connect `Combine Input` output to `If` input  

8. **Activate and save the workflow**  
   - Ensure all nodes are properly configured with no errors  
   - Activate the workflow to test manual execution or external workflow calls.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| This workflow is an example template to organize and test sub-workflows in n8n.                                                    | Embedded Sticky Note in the workflow                                                                                          |
| More details and explanation are available at the N8n Tips blog: [https://n8n-tips.blogspot.com/2025/06/tip-1-testing-n8n-sub-workflow.html](https://n8n-tips.blogspot.com/2025/06/tip-1-testing-n8n-sub-workflow.html) | External blog post for extended learning                                                                                        |

---

**Disclaimer:** The text is derived exclusively from an automated n8n workflow and adheres strictly to content policies. It contains no illegal, offensive, or protected material. All manipulated data are legal and public.