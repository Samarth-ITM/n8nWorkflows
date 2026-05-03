Automatically Optimize AI Prompts with OpenAI Using OPRO & DSPy Methodology

https://n8nworkflows.xyz/workflows/automatically-optimize-ai-prompts-with-openai-using-opro---dspy-methodology-11495


# Automatically Optimize AI Prompts with OpenAI Using OPRO & DSPy Methodology

---

### 1. Workflow Overview

This workflow automates the optimization of AI prompts using advanced methodologies inspired by Google DeepMind's OPRO and Stanford's DSPy. Its primary goal is to iteratively refine a system prompt so that an AI model can generate outputs that match a predefined ground truth with high accuracy, minimizing manual trial-and-error in prompt engineering.

The workflow is logically divided into the following blocks:

- **1.1 Initialization:** Define the initial prompt, test input, and expected ground truth.
- **1.2 Loop & State Management:** Maintain and update the current prompt, loop count, and stop conditions.
- **1.3 AI Response Generation:** Use the current prompt to generate a response to the test input.
- **1.4 Evaluation:** Evaluate the AI-generated response against the ground truth, scoring accuracy and format.
- **1.5 Optimization:** Analyze evaluation results and rewrite the prompt to improve future responses.
- **1.6 Control Flow:** Use conditional logic to loop through generation, evaluation, and optimization until a stopping criterion is met (score ≥ 95 or max loops reached).

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization

**Overview:**  
Sets the baseline data for the workflow: the initial prompt, the input text for AI processing, and the expected correct answer (ground truth). This prepares the system for the optimization loop.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Define Initial Prompt & Test Data  
- Sticky Note Init  
- Sticky Note Main  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; triggers subsequent nodes.  
  - Connections: Outputs to "Define Initial Prompt & Test Data".  
  - Edge Cases: None specific; user must trigger manually.

- **Define Initial Prompt & Test Data**  
  - Type: Set Node  
  - Role: Defines three key variables:  
    - `initial_prompt`: "Extract invoice data from the text and output as JSON."  
    - `test_input`: A sample invoice text.  
    - `ground_truth`: The expected JSON output representing the invoice data.  
  - Configuration: Variables are hardcoded but editable for customization.  
  - Connections: Outputs to "Manage Loop & State".  
  - Edge Cases: Ensure JSON in `ground_truth` is correctly formatted; invalid JSON would cause downstream evaluation issues.

- **Sticky Note Init & Sticky Note Main**  
  - Type: Sticky Note  
  - Role: Documentation and instructions embedded in the workflow UI.  
  - Configuration: Text content explaining the purpose and setup of the workflow.  
  - Connections: None (informational only).

---

#### 2.2 Loop & State Management

**Overview:**  
Manages the current state of the optimization loop including the prompt to use, loop iteration count, current evaluation score, and determines when to stop looping.

**Nodes Involved:**  
- Manage Loop & State  
- Check Loop Condition  

**Node Details:**

- **Manage Loop & State**  
  - Type: Code Node (JavaScript)  
  - Role:  
    - Loads static data from initialization node.  
    - Retrieves loop count from previous iteration or AI suggestion.  
    - Maintains or updates the current prompt depending on improvement availability.  
    - Tracks current score from evaluation.  
    - Sets a `stop_loop` flag if the loop count exceeds 5 or score ≥ 95.  
  - Key Expressions:  
    - Reads `next_loop_count`, `loop_count`, `improved_prompt` from incoming data.  
    - Outputs updated state JSON with keys like `current_prompt`, `loop_count`, `stop_loop`.  
  - Input: Data from "Update Prompt & Loop Count" or "Define Initial Prompt & Test Data" on first run.  
  - Output: To "Check Loop Condition".  
  - Edge Cases:  
    - Missing or malformed input data may cause fallback to defaults.  
    - Loop count capping prevents infinite loops.  

- **Check Loop Condition**  
  - Type: If Node  
  - Role: Checks `stop_loop` boolean to decide whether to end the workflow or continue.  
  - Configuration: Condition tests if `stop_loop` is true.  
  - Inputs: From "Manage Loop & State".  
  - Outputs:  
    - True branch to "End Loop" node.  
    - False branch to "AI Response Generator".  
  - Edge Cases: Ensures workflow terminates after max iterations or target score.

---

#### 2.3 AI Response Generation

**Overview:**  
Generates a response from the AI model using the current prompt and test input.

**Nodes Involved:**  
- AI Response Generator  
- OpenAI Chat Model  

**Node Details:**

- **AI Response Generator**  
  - Type: LangChain Agent Node (AI model interface)  
  - Role: Sends prompt and input data to the language model to generate an output.  
  - Configuration:  
    - `text` input: `"Respond to the following input according to the instructions.\n\nInput:\n{{ $json.test_input }}"`  
    - System message: Uses current prompt from state (`current_prompt`).  
  - Input: From "Check Loop Condition" (false branch) and "Manage Loop & State".  
  - Output: AI-generated response JSON.  
  - Connected to: "AI Response Evaluator".  
  - Edge Cases: Handles AI model timeouts or errors; fallback continues workflow.

- **OpenAI Chat Model**  
  - Type: Language Model Node (OpenAI GPT-4o-mini)  
  - Role: Executes the actual call to OpenAI's GPT model.  
  - Configuration: Uses GPT-4o-mini model with credentials named "OpenAi account".  
  - Inputs: Serves AI Prompt Optimizer, AI Response Evaluator, AI Response Generator nodes as language model backend.  
  - Edge Cases: Requires valid OpenAI API credentials; rate limits or connectivity issues possible.

---

#### 2.4 Evaluation

**Overview:**  
Evaluates the AI-generated response by scoring its accuracy and format against the ground truth.

**Nodes Involved:**  
- AI Response Evaluator  
- Evaluator Output Parser  

**Node Details:**

- **AI Response Evaluator**  
  - Type: LangChain Agent Node  
  - Role: Compares the AI-generated answer with expected ground truth and assigns a score (0-100) plus a reason for deductions.  
  - Configuration:  
    - Input data: test input, ground truth, generated answer.  
    - System message: Strict evaluator instructions focusing on accuracy, format, and tone.  
  - Input: Output from "AI Response Generator" and state data from "Manage Loop & State".  
  - Output: Evaluation JSON with `score` and `reason`.  
  - Connections: Outputs to "AI Prompt Optimizer".  
  - Edge Cases: May fail if output format is invalid or key data missing.

- **Evaluator Output Parser**  
  - Type: Output Parser Node  
  - Role: Parses the evaluator's AI response into structured JSON with fields `score` (number) and `reason` (string).  
  - Configuration: Uses JSON schema example for validation.  
  - Input: AI output from "AI Response Evaluator".  
  - Output: Parsed data passed to "AI Response Evaluator" node for further processing.  

---

#### 2.5 Optimization

**Overview:**  
Improves the prompt based on evaluation feedback to increase the score in subsequent iterations.

**Nodes Involved:**  
- AI Prompt Optimizer  
- Optimizer Output Parser  
- Update Prompt & Loop Count  

**Node Details:**

- **AI Prompt Optimizer**  
  - Type: LangChain Agent Node  
  - Role: Acts as a world-class prompt engineer to rewrite the prompt based on evaluation results and AI-generated answer failures.  
  - Configuration:  
    - Input includes previous prompt, generated result, evaluation score/reason, and ground truth.  
    - System message guides optimization strategy including clarity, constraints, guiding thinking, and few-shot examples.  
    - Outputs a JSON with improved prompt and change log.  
  - Input: From "AI Response Evaluator".  
  - Output: To "Update Prompt & Loop Count".  
  - Edge Cases: May produce invalid JSON or insufficient optimizations; error handling set to continue workflow.

- **Optimizer Output Parser**  
  - Type: Output Parser Node  
  - Role: Parses the optimizer's output JSON with `improved_prompt` and `change_log`.  
  - Input: AI output from "AI Prompt Optimizer".  
  - Output: Structured data passed to "Update Prompt & Loop Count".  

- **Update Prompt & Loop Count**  
  - Type: Set Node  
  - Role: Updates workflow variables for the next iteration: increments loop count, applies improved prompt, and records change log.  
  - Configuration:  
    - `next_loop_count` = current loop count + 1  
    - `improved_prompt` from optimizer output  
    - `change_log` from optimizer output  
  - Input: From "AI Prompt Optimizer".  
  - Output: To "Manage Loop & State" for next iteration input.  
  - Edge Cases: Ensures loop count increments properly; improper prompt updates may cause stagnation.

---

#### 2.6 Control Flow and Termination

**Overview:**  
Determines when to stop the optimization loop and gracefully end the workflow.

**Nodes Involved:**  
- Check Loop Condition (described above)  
- End Loop  

**Node Details:**

- **End Loop**  
  - Type: No Operation (NoOp) Node  
  - Role: Acts as an endpoint when stopping criteria are met.  
  - Input: True branch from "Check Loop Condition".  
  - Output: None, terminates workflow execution.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                  |
|----------------------------|-----------------------------------|-------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow start trigger         | -                                 | Define Initial Prompt & Test Data |                                                                                              |
| Define Initial Prompt & Test Data | Set                             | Define initial prompt/test data| When clicking ‘Execute workflow’  | Manage Loop & State                | ## 1. Initialization Define the starting prompt and the test case. Example: Invoice Data Extraction |
| Manage Loop & State         | Code                              | Manage loop count and prompt   | Define Initial Prompt & Test Data, Update Prompt & Loop Count | Check Loop Condition              |                                                                                              |
| Check Loop Condition        | If                                | Control loop continuation      | Manage Loop & State                | End Loop (true), AI Response Generator (false) |                                                                                              |
| End Loop                   | NoOp                              | Workflow termination           | Check Loop Condition (true)        | -                                 |                                                                                              |
| AI Response Generator       | LangChain Agent                   | Generate AI response           | Check Loop Condition (false)       | AI Response Evaluator             | ## 2. Generation & Evaluation Generate response and score it against ground truth.           |
| OpenAI Chat Model           | Language Model (OpenAI)           | Backend for AI nodes           | All LangChain Agent nodes          | All LangChain Agent nodes          |                                                                                              |
| AI Response Evaluator       | LangChain Agent                   | Evaluate AI output             | AI Response Generator              | AI Prompt Optimizer               |                                                                                              |
| Evaluator Output Parser     | Output Parser                    | Parse evaluation results       | AI Response Evaluator              | AI Response Evaluator             |                                                                                              |
| AI Prompt Optimizer         | LangChain Agent                   | Optimize and rewrite prompt    | AI Response Evaluator              | Update Prompt & Loop Count        | ## 3. Optimization Loop Analyze results and improve the prompt if needed.                    |
| Optimizer Output Parser     | Output Parser                    | Parse optimizer output         | AI Prompt Optimizer                | AI Prompt Optimizer               |                                                                                              |
| Update Prompt & Loop Count  | Set                              | Update prompt and iteration    | AI Prompt Optimizer                | Manage Loop & State               |                                                                                              |
| Sticky Note Main            | Sticky Note                      | Documentation                  | -                                 | -                                 | ## Auto-Optimize Prompts (OPRO/DSPy Implementation) Explains overall workflow concept.        |
| Sticky Note Init            | Sticky Note                      | Documentation                  | -                                 | -                                 | ## 1. Initialization Define the starting prompt and the test case. Example: Invoice Data Extraction |
| Sticky Note Gen Eval        | Sticky Note                      | Documentation                  | -                                 | -                                 | ## 2. Generation & Evaluation Generate response and score it against ground truth.           |
| Sticky Note Opt             | Sticky Note                      | Documentation                  | -                                 | -                                 | ## 3. Optimization Loop Analyze results and improve the prompt if needed.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.  

2. **Create Set Node (Define Initial Prompt & Test Data):**
   - Assign variables:  
     - `initial_prompt`: `"Extract invoice data from the text and output as JSON."`  
     - `test_input`: `"Invoice #INV-9982. Date: 12th of Oct, 2023. Grand Total: $1,100 (includes 10% VAT). Items: 2x Widget A, 1x Widget B."`  
     - `ground_truth`: JSON string representing expected invoice data.  
   - Connect Manual Trigger → this node.

3. **Create Code Node (Manage Loop & State):**
   - JavaScript code to:  
     - Retrieve initial prompt and test data.  
     - Track/update loop count (start at 1).  
     - Maintain or update prompt based on improved prompt input.  
     - Capture current evaluation score.  
     - Set `stop_loop` if score ≥ 95 or loop count > 5.  
   - Connect Set Node → this node.

4. **Create If Node (Check Loop Condition):**
   - Condition: `stop_loop == true`  
   - True branch connects to NoOp node "End Loop" (terminates workflow).  
   - False branch connects to AI Response Generator node.  
   - Connect Code Node → If Node.

5. **Create LangChain Agent Node (AI Response Generator):**
   - Prompt text: `"Respond to the following input according to the instructions.\n\nInput:\n{{ $json.test_input }}"`  
   - System message: use `current_prompt` from state.  
   - Connect If Node (false branch) → this node.

6. **Create LangChain Agent Node (AI Response Evaluator):**
   - Input: test input, ground truth, AI-generated answer.  
   - System message: strict evaluator instructions focusing on accuracy, format, and tone.  
   - Connect AI Response Generator → this node.

7. **Create Output Parser Node (Evaluator Output Parser):**
   - JSON schema: `{ "score": number, "reason": string }`  
   - Connect AI Response Evaluator → this node.

8. **Create LangChain Agent Node (AI Prompt Optimizer):**
   - Input: previous prompt, generated result, evaluation score & reason, ground truth.  
   - System message: instructions to improve prompt clarity, constraints, step-by-step guidance, and few-shot examples.  
   - Output: JSON with improved prompt and change log.  
   - Connect AI Response Evaluator → this node.

9. **Create Output Parser Node (Optimizer Output Parser):**
   - JSON schema: `{ "improved_prompt": string, "change_log": string }`  
   - Connect AI Prompt Optimizer → this node.

10. **Create Set Node (Update Prompt & Loop Count):**
    - Assignments:  
      - `next_loop_count`: increment previous loop count by 1.  
      - `improved_prompt`: from optimizer output.  
      - `change_log`: from optimizer output.  
    - Connect AI Prompt Optimizer → this node.

11. **Connect Update Prompt & Loop Count → Manage Loop & State** to complete the loop.

12. **Create NoOp Node (End Loop):**
    - Connect If Node (true branch) → this node.

13. **Create OpenAI Chat Model Node:**
    - Model: `gpt-4o-mini` (or latest available).  
    - Credentials: Configure with valid OpenAI API account.  
    - Connect all LangChain Agent nodes to this model node as their backend.

14. **Add Sticky Notes (Optional):**
    - Add notes to explain each major block for documentation and ease of use.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow implements an iterative AI prompt optimization loop inspired by DeepMind's OPRO and Stanford's DSPy methodologies to enhance prompt engineering systematically.                                                                                                                                                                                              | Conceptual background of the workflow.                                                                           |
| The OpenAI credential node must be configured with an API key that has access to GPT-4o-mini or a compatible advanced GPT model for best results.                                                                                                                                                                                                                          | Credential setup requirement.                                                                                     |
| The workflow limits iterations to a maximum of 5 loops or until the evaluation score reaches 95 or higher to prevent infinite loops.                                                                                                                                                                                                                                       | Safeguard against infinite execution.                                                                             |
| The workflow includes detailed sticky notes inside n8n for user guidance on setup, usage, and methodology.                                                                                                                                                                                                                                                                  | Embedded documentation within workflow.                                                                           |
| For additional reference on prompt optimization techniques, consult resources on OPRO (Open Prompt Optimization) and DSPy (Deep Structured Prompting) methodologies from Google DeepMind and Stanford University respectively.                                                                                                                                             | External research papers and blogs (not linked here).                                                            |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created using n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.