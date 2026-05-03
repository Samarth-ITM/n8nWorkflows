Dynamically switch between LLMs for AI Agents using LangChain Code

https://n8nworkflows.xyz/workflows/dynamically-switch-between-llms-for-ai-agents-using-langchain-code-3820


# Dynamically switch between LLMs for AI Agents using LangChain Code

### 1. Workflow Overview

This workflow demonstrates a dynamic approach to switch between multiple Large Language Models (LLMs) within a single AI agent powered by LangChain in n8n. It handles customer complaints by generating polite, empathetic responses and validating their quality before delivering the final output. If the response is unsatisfactory, the workflow iteratively retries with the next available LLM until either a valid response is produced or all models have been exhausted.

The workflow logic is grouped into these blocks:

- **1.1 Input Reception:** Receives chat messages (customer complaints) via the LangChain Chat Trigger.
- **1.2 LLM Index Initialization:** Sets or increments the LLM selection index to control which LLM to use.
- **1.3 Dynamic LLM Switching:** Programmatically selects an LLM based on the current index.
- **1.4 Response Generation:** Generates a polite AI response using the selected LLM.
- **1.5 Response Validation:** Validates the response sentiment and quality.
- **1.6 Loop Control and Error Handling:** Handles looping on failure, error evaluation, and termination conditions.
- **1.7 Result Return or Workflow Termination:** Returns the final output or a fallback message when no satisfactory result is found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures incoming chat messages as customer complaints to be processed.

**Nodes Involved:**  
- When chat message received

**Node Details:**  
- **Type:** LangChain Chat Trigger Node  
- **Role:** Serves as the webhook entry point for customer messages delivered to the workflow.  
- **Configuration:** Default webhook settings with no additional options.  
- **Key Data:** Receives chatInput text as input (`$json.chatInput`).  
- **Input:** External webhook call from chat interface  
- **Output:** Passes message JSON to "Set LLM index" node.  
- **Edge Cases:** Webhook failures, malformed JSON input, missing `chatInput` field.  
- **Sub-workflow:** None

---

#### 1.2 LLM Index Initialization

**Overview:**  
Determines which LLM to use initially or after each failed attempt by tracking an index (`llm_index`).

**Nodes Involved:**  
- Set LLM index  
- Increase LLM index

**Node Details:**  

- *Set LLM index*  
  - **Type:** Set Node  
  - **Role:** Initializes or resets `llm_index` to 0 on first run, or uses existing index if present.  
  - **Configuration:** Assigns `llm_index` to either existing value (`$json.llm_index`) or `0` by default.  
  - **Connection:** Input from "When chat message received" and from "No Operation" node (loop back).  
  - **Output:** Passes data with `llm_index` to "Generate response".  
  - **Edge Cases:** Missing or invalid index values; ensures initial value always integer 0 or more.

- *Increase LLM index*  
  - **Type:** Set Node  
  - **Role:** Increments `llm_index` by one to try the next model.  
  - **Configuration:** Adds 1 to the value from "Set LLM index" node (`$('Set LLM index').item.json.llm_index + 1`).  
  - **Connection:** Input from "Validate response" fail output; output to "No Operation".  
  - **Edge Cases:** Integer overflow unlikely but logically possible; out-of-range index errors handled downstream.

---

#### 1.3 Dynamic LLM Switching

**Overview:**  
Selects the LLM to be used dynamically by consuming all LLM nodes connected and choosing one via `llm_index`.

**Nodes Involved:**  
- Switch Model

**Node Details:**  
- **Type:** LangChain Code Node  
- **Role:** Dynamically selects the specific LLM node corresponding to the current `llm_index`. This allows a single AI Chain node to use multiple underlying LLMs seamlessly in sequence.  
- **Configuration:** Custom JavaScript code:  
  - Reads all connected LLM nodes (`llms`) in reverse order to align with UI order.  
  - Uses `llm_index` from input JSON to pick the matching LLM.  
  - Throws an error if index is invalid or no LLM found.  
- **Key Expressions:**  
  ```js
  let llms = await this.getInputConnectionData('ai_languageModel', 0);
  llms.reverse();
  const llm_index = $input.item.json.llm_index;
  if (!Number.isInteger(llm_index)) throw Error("'llm_index' invalid");
  if (typeof llms[llm_index] === 'undefined') throw Error(`No LLM found with index ${llm_index}`);
  return llms[llm_index];
  ```  
- **Input:** Receives all connected LLM nodes and `llm_index`.  
- **Output:** Passes selected LLM node downstream as the active model to "Generate response".  
- **Edge Cases:**  
  - Invalid or missing `llm_index`, no matching LLM node, or connection issues cause workflow errors caught downstream.  
  - Errors reported specifically as `"Error in sub-node Switch Model"` for conditional checks.  
- **Notes:** This node centralizes switching logic; all OpenAI LLM nodes connect as inputs to it.

---

#### 1.4 Response Generation

**Overview:**  
Uses the dynamically selected LLM to generate a short, polite response to the customer complaint.

**Nodes Involved:**  
- Generate response

**Node Details:**  
- **Type:** LangChain LLM Chain Node  
- **Role:** Generates customer support reply with a prompt instructing empathy, politeness, and help offer, based on the incoming complaint text.  
- **Configuration:**  
  - Text input: `={{ $('When chat message received').item.json.chatInput }}` — the raw complaint text.  
  - Prompt: Fixed system message defining assistant role and tone for reply.  
- **Input:** Receives the currently selected LLM from "Switch Model" and text input.  
- **Output:** Produces AI-generated text reply.  
- **Error Handling:** Continues on errors without stopping workflow (`onError: continueErrorOutput`), allowing fallback logic.  
- **Edge Cases:** LLM API timeouts, invalid prompt, or failure to generate text.  
- **Notes:** Connects output to sentiment validation and error checking nodes.

---

#### 1.5 Response Validation

**Overview:**  
Assesses the quality of the generated response to ensure it satisfies conditions for adequacy.

**Nodes Involved:**  
- Validate response

**Node Details:**  
- **Type:** LangChain Sentiment Analysis Node  
- **Role:** Analyzes the generated text reply’s sentiment according to custom criteria to decide if response "passes" or "fails".  
- **Configuration:**  
  - Categories: `pass, fail`  
  - System prompt template includes criteria:  
    1. Acknowledges both broken keyboard and late delivery  
    2. Uses polite and empathetic tone  
    3. Offers clear resolution or next steps  
  - Input text mapped from AI reply: `={{ $json.text }}`  
- **Input:** Receives generated response text.  
- **Output:** JSON object with field `quality` equal to "pass" or "fail".  
- **Edge Cases:** Possible misclassification, malformed output, or API failure.  
- **Notes:** The decision here controls the loop flow.

---

#### 1.6 Loop Control and Error Handling

**Overview:**  
Decides whether to return the answer or retry with the next LLM or handle errors.

**Nodes Involved:**  
- Check for expected error  
- Loop finished without results  
- Unexpected error  
- No Operation, do nothing

**Node Details:**  

- *Check for expected error*  
  - **Type:** If Node  
  - **Role:** Checks if the "Switch Model" node returned a known error indicating no more models available.  
  - **Condition:** Evaluates if `$json.error === "Error in sub-node Switch Model"`.  
  - **Outputs:**  
    - True: Loop finished without results  
    - False: Unexpected error  
  - **Edge Cases:** Unexpected error message formats not caught.

- *Loop finished without results*  
  - **Type:** Set Node  
  - **Role:** Sets output message signaling no satisfying result after all LLMs exhausted.  
  - **Parameters:** Assigns static string: "The loop finished without a satisfying result".  
  - **Connection:** True branch of error check.

- *Unexpected error*  
  - **Type:** Set Node  
  - **Role:** Sets output for unexpected failure cases not covered by known errors.  
  - **Parameters:** Assigns static string: "An unexpected error happened".  
  - **Connection:** False branch of error check.

- *No Operation, do nothing*  
  - **Type:** NoOp Node  
  - **Role:** Placeholder to facilitate connection loops without modifying data.  
  - **Connection:** Passes data from incremented `llm_index` back to "Set LLM index" to restart response generation with next LLM.  
  - **Edge Cases:** None; serves structural role.

---

#### 1.7 Result Return or Workflow Termination

**Overview:**  
Returns the final response text back to the invoking interface or continues loop when necessary.

**Nodes Involved:**  
- Return result

**Node Details:**  
- **Type:** Set Node  
- **Role:** Sets the final output text to be returned by the workflow.  
- **Parameters:**  
  - Assigns field `output`: either the validated reply text or fallback output string.  
  - Expression: `={{ $json.text || $json.output }}` to cover both LLM responses and error/failure strings.  
- **Connection:** Receives from the pass path of "Validate response".  
- **Edge Cases:** Missing text fallback handled by default.  
- **Notes:** Final output for the workflow execution.

---

#### Supporting LLM Nodes

**Overview:**  
Multiple OpenAI LLM Chat nodes provide the language model API connections.

**Nodes Involved:**  
- OpenAI 4o-mini  
- OpenAI 4o  
- OpenAI o1  
- OpenAI Chat Model

**Node Details:**  
- **Type:** LangChain OpenAI Chat LLM Nodes  
- **Role:** These represent different OpenAI model variants associated to the workflow.  
- **Configuration:** Each node specifies a distinct model name (e.g. `gpt-4o-mini`, `gpt-4o`, `o1`).  
- **Credentials:** Use shared OpenAI API key credential named "OpenAi (octionicsolutions)".  
- **Connection:** All connect as inputs to "Switch Model" node, which picks one per iteration.  
- **Edge Cases:** API authentication failure, quota limits, latency.  
- **Notes:** Order of nodes defines the LLM consumption order via index; change order to affect selection order.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                               | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                         |
|----------------------------|-----------------------------------|-----------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger            | Entry point to receive chat messages          | (Webhook external)             | Set LLM index               |                                                                                                   |
| Set LLM index              | Set Node                         | Initialize or reset LLM index                  | When chat message received, No Operation | Generate response           | Defines the LLM node by index which should be used.                                               |
| OpenAI 4o-mini             | LangChain OpenAI Chat LLM        | One of the connected LLM models                |                               | Switch Model (ai_languageModel) |                                                                                                   |
| OpenAI 4o                  | LangChain OpenAI Chat LLM        | One of the connected LLM models                |                               | Switch Model (ai_languageModel) |                                                                                                   |
| OpenAI o1                  | LangChain OpenAI Chat LLM        | One of the connected LLM models                |                               | Switch Model (ai_languageModel) |                                                                                                   |
| Switch Model               | LangChain Code Node              | Selects LLM dynamically based on index         | OpenAI nodes (multiple), Set LLM index | Generate response           | Dynamically connects the LLM by the index provided in the previous node.                          |
| Generate response          | LangChain Chain LLM              | Generates polite AI reply from complaint       | Switch Model, Set LLM index    | Validate response, Check for expected error | Generates a polite answer based on the customers complaint.                                        |
| Validate response          | LangChain Sentiment Analysis     | Validates if AI response meets quality criteria | Generate response            | Return result, Increase LLM index | Analyses the generated answer by certain criteria                                                 |
| Return result              | Set Node                         | Returns final output response                   | Validate response             |                             |                                                                                                   |
| Increase LLM index         | Set Node                         | Increments `llm_index` for next LLM retry      | Validate response             | No Operation                | Increases the index to choose the next available LLM on the next run                              |
| No Operation, do nothing   | NoOp Node                       | Loops updated index back to LLM index setter   | Increase LLM index            | Set LLM index               |                                                                                                   |
| Check for expected error   | If Node                         | Detects known “no LLM found” error             | Generate response             | Loop finished without results, Unexpected error | Check if LangChain Code Node ran into error. _Currently only supports error output from main Node_ |
| Loop finished without results | Set Node                     | Sets output when all LLMs exhausted             | Check for expected error      |                             |                                                                                                   |
| Unexpected error           | Set Node                         | Sets output for unexpected error                | Check for expected error      |                             |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the LangChain Chat Trigger Node:**  
   - Add node: `When chat message received` (type: `@n8n/n8n-nodes-langchain.chatTrigger`).  
   - Use default settings.  
   - This will serve as the webhook entry point receiving chat messages from customers.

2. **Add a Set Node to Initialize LLM Index:**  
   - Create `Set LLM index` node (type: Set).  
   - Configure: Add assignment `"llm_index"` as a number type, value expression: `={{ $json.llm_index || 0 }}` to default index to 0.  
   - Connect `When chat message received` → `Set LLM index`.

3. **Add LangChain OpenAI LLM Nodes:**  
   - Create multiple LLM nodes for each model variant you want (e.g. `OpenAI 4o-mini`, `OpenAI 4o`, `OpenAI o1`, `OpenAI Chat Model`).  
   - Set each node’s model property accordingly (e.g., `"gpt-4o-mini"`, `"gpt-4o"`, `"o1"`).  
   - Assign valid OpenAI API credentials to each LLM node.  
   - No connections yet.

4. **Create LangChain Code Node 'Switch Model':**  
   - Add node `Switch Model` (type: LangChain Code).  
   - Set code with the following logic:  
     ```js
     let llms = await this.getInputConnectionData('ai_languageModel', 0);
     llms.reverse();
     const llm_index = $input.item.json.llm_index;
     if (!Number.isInteger(llm_index)) throw new Error("'llm_index' is undefined or invalid.");
     if(typeof llms[llm_index] === 'undefined') throw new Error(`No LLM found with index ${llm_index}`);
     return llms[llm_index];
     ```  
   - Inputs: Connect all OpenAI LLM nodes as `ai_languageModel` input to this node.  
   - Input from `Set LLM index` node as main input.  
   - Output: will pass selected LLM downstream.

5. **Add LangChain LLM Chain Node 'Generate response':**  
   - Create `Generate response` node (LangChain LLM Chain).  
   - Set input text to: `={{ $('When chat message received').item.json.chatInput }}`  
   - Define prompt message:  
     `"You’re an AI assistant replying to a customer who is upset about a faulty product and late delivery. The customer uses sarcasm and is vague. Write a short, polite response, offering help."`  
   - Connect `Switch Model` output to this node’s LLM input.  
   - Connect `Set LLM index` output as main input for data.

6. **Create a Sentiment Analysis Node 'Validate response':**  
   - Add `Validate response` node (LangChain Sentiment Analysis).  
   - Configure categories: `pass, fail`  
   - System Prompt Template: Include instructions to output JSON with `"quality": "pass"` or `"fail"` based on:  
     1. Acknowledgement of issues  
     2. Politeness and empathy  
     3. Offers resolution or next next step  
   - Input: Map analysis text from `Generate response` output JSON field `text`.  
   - Connect `Generate response` output to `Validate response`.

7. **Add a Set Node 'Return result':**  
   - Create `Return result` node (Set).  
   - Assign `"output"` field with expression: `={{ $json.text || $json.output }}` to return the generated or fallback text.  
   - Connect `Validate response` success output ("pass" branch) to this node.

8. **Add Set Node 'Increase LLM index':**  
   - Create `Increase LLM index` node (Set).  
   - Assign `"llm_index"` field: `={{ $('Set LLM index').item.json.llm_index + 1 }}` to increment index by 1.  
   - Connect from `Validate response` fail output.

9. **Add No Operation (NoOp) Node:**  
   - Add `No Operation, do nothing` node (NoOp).  
   - Connect `Increase LLM index` output to this node.  
   - Connect `No Operation` output back to `Set LLM index` to loop with updated index.

10. **Add If Node 'Check for expected error':**  
    - Create an If Node to evaluate if an error from "Switch Model" occurred.  
    - Condition: `$json.error` equals `"Error in sub-node Switch Model"`.  
    - Connect `Generate response` error output to this node.

11. **Add Set Node 'Loop finished without results':**  
    - Create node with static string: `"The loop finished without a satisfying result"` assigned to field `output`.  
    - Connect If node true output (error detected) here.

12. **Add Set Node 'Unexpected error':**  
    - Assign `"An unexpected error happened"` string to `"output"`.  
    - Connect If node false output here.

13. **Connect all outputs:**  
    - Set `Loop finished without results` and `Unexpected error` nodes to the end or return as workflow termination outputs.

14. **Final connections summary:**  
    - `When chat message received` → `Set LLM index` → `Switch Model` (via all OpenAI LLM nodes) → `Generate response` → `Validate response`  
    - `Validate response` pass → `Return result`  
    - `Validate response` fail → `Increase LLM index` → `No Operation` → loops back to `Set LLM index`  
    - `Generate response` error → `Check for expected error` → either termination (loop finished / unexpected error) or else (not caught)  
    - Confirm inputs/outputs match described flow.

15. **Credential Setup:**  
    - Provide OpenAI API credentials named identically for all LLM model nodes.  
    - No additional credentials needed unless swapping providers.

16. **Optional:** Add sticky notes for documentation at each major logical block for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Remember the LLM execution order depends on the order of LLM nodes connected to the LangChain Code Node, not their canvas position.               | Workflow Use / Design                                           |
| This workflow requires a self-hosted n8n instance because it relies on the LangChain Code Node, which is not available on the cloud version.     | Workflow Prerequisite                                          |
| The example complaint message for testing is: <br> "I really *love* waiting two weeks just to get a keyboard that doesn’t even work. Great job..."| Included in Sticky Note at workflow start                      |
| Project supports easy substitution of LLM providers, e.g., Anthropic, by replacing or adding alternative LLM nodes on the "Switch Model" input. | Workflow extensibility                                         |
| Sentiment Analysis output is strictly JSON with field `quality` expected to be `"pass"` or `"fail"`; incorrect outputs may break the control flow.| Validation robustness note                                     |

---

This completes the comprehensive analysis, reproduction guidelines, and structured reference for the provided dynamic LLM switching workflow using LangChain Code in n8n.