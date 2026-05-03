Evaluation metric example: Correctness (judged by AI)

https://n8nworkflows.xyz/workflows/evaluation-metric-example--correctness--judged-by-ai--4271


# Evaluation metric example: Correctness (judged by AI)

### 1. Workflow Overview

This n8n workflow implements an evaluation metric to assess the **correctness** of AI-generated answers by comparing them with expected reference answers. It is designed primarily for workflows that generate factual answers to questions (here: causes of historical events) and need to automatically score their accuracy on a defined scale.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives questions and triggers evaluation processing by reading dataset rows or chat messages.
- **1.2 Formatting and AI Agent Processing:** Prepares inputs and calls an AI agent to produce concise answers.
- **1.3 Evaluation Mode Check:** Determines if the workflow is in evaluation mode to decide whether to calculate metrics.
- **1.4 Correctness Metric Calculation:** Uses a detailed OpenAI prompt to compare the AI-generated output with the ground truth and score similarity on a 1-5 scale.
- **1.5 Metric Setting and Output:** Sets the similarity metric in the evaluation system and returns responses accordingly.

The workflow also includes sticky notes providing user guidance and links to documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block listens for incoming data, either from a dataset row fetched from a Google Sheet or from chat messages, to initiate the evaluation process.
- **Nodes Involved:**  
  - `When fetching a dataset row`  
  - `When chat message received`
  
- **Node Details:**

  - **When fetching a dataset row**  
    - Type: `evaluationTrigger` (n8n base node specialized for evaluation workflows)  
    - Role: Triggers the workflow when a new row is fetched from the specified Google Sheets document containing questions and reference answers.  
    - Configuration: Linked to a specific Google Sheet URL and sheet name. Uses OAuth2 credentials for Google Sheets access.  
    - Inputs: External trigger via Google Sheets row fetch  
    - Outputs: Emits row data including question and reference answer fields  
    - Edge cases: OAuth token expiry, network issues, invalid or missing rows
    
  - **When chat message received**  
    - Type: `chatTrigger` (from LangChain n8n nodes)  
    - Role: Triggers when a chat message is received via webhook; supports conversational input  
    - Configuration: Webhook ID provided; no special options configured  
    - Inputs: Incoming chat messages via webhook  
    - Outputs: Passes chat message content for further processing  
    - Edge cases: Webhook downtime, missing message content

#### 2.2 Formatting and AI Agent Processing

- **Overview:** This block formats the input question to a uniform variable and sends it to an AI agent for a concise answer.
- **Nodes Involved:**  
  - `Match chat format`  
  - `AI Agent`  
  - `OpenAI Chat Model`
  
- **Node Details:**

  - **Match chat format**  
    - Type: `set` node  
    - Role: Extracts the question field from input JSON and assigns it to a new variable `chatInput` to standardize input format  
    - Configuration: Sets `chatInput = {{$json.question}}`  
    - Inputs: Data from dataset row trigger  
    - Outputs: Data with standardized `chatInput` field  
    - Edge cases: Missing `question` field in input
    
  - **AI Agent**  
    - Type: `langchain.agent`  
    - Role: Processes the input question through an AI agent to generate an answer  
    - Configuration:  
      - Input text: `={{ $json.chatInput }}`  
      - System message: "You are a helpful assistant. Answer the user's questions, but be very concise (max one sentence)"  
      - Prompt type: define  
    - Inputs: Standardized question text  
    - Outputs: AI-generated answer in JSON format under `output` field  
    - Edge cases: API rate limits, response timeouts, malformed output
    
  - **OpenAI Chat Model**  
    - Type: `lmChatOpenAi` (LangChain OpenAI chat model)  
    - Role: Provides the underlying GPT-4o-mini model used by the AI agent for chat completions  
    - Configuration: Model set to `"gpt-4o-mini"`  
    - Credentials: OpenAI API key configured  
    - Inputs: Chat prompts from AI Agent  
    - Outputs: Model responses  
    - Edge cases: API key invalidation, quota exceeded

#### 2.3 Evaluation Mode Check

- **Overview:** This block verifies if the workflow is currently in evaluation mode to decide if metric calculation should proceed, reducing unnecessary costs.
- **Nodes Involved:**  
  - `AI Agent` (output connection)  
  - `Evaluating?`  
  - `Calculate correctness metric`  
  - `Return chat response`  
  - `Sticky Note` (explaining purpose)
  
- **Node Details:**

  - **Evaluating?**  
    - Type: `evaluation` node  
    - Role: Checks if the workflow is actively evaluating, i.e., if metrics need to be calculated  
    - Configuration: Operation set to "checkIfEvaluating"  
    - Inputs: AI Agent output  
    - Outputs: Two branches: one to calculate metrics if evaluating, one to return response without metric calculation  
    - Edge cases: Misconfiguration causing false positives/negatives
    
  - **Calculate correctness metric**  
    - Type: `openAi` (LangChain OpenAI node)  
    - Role: Uses OpenAI GPT-4o-mini to evaluate factual correctness of AI output against a ground truth answer on a 1-5 similarity scale  
    - Configuration:  
      - Model: GPT-4o-mini  
      - Messages: Detailed system prompt specifying evaluation steps, scoring criteria, output JSON format, and examples  
      - Input includes AI output and ground truth answer from dataset row  
      - Output: JSON with fields `extended_reasoning`, `reasoning_summary`, and `score`  
    - Credentials: OpenAI API key configured  
    - Inputs: AI Agent output and reference answer from dataset row (via expression referencing `When fetching a dataset row`)  
    - Outputs: JSON-formatted correctness metric  
    - Edge cases: Prompt misinterpretation, API errors, malformed inputs
    
  - **Return chat response**  
    - Type: `noOp` (no operation)  
    - Role: Passes the flow along without metric calculation when not evaluating  
    - Inputs: Evaluation check negative branch  
    - Outputs: Final pass-through  
    - Edge cases: None
    
  - **Sticky Note**  
    - Content: "Only calculate metrics if we're evaluating, to reduce costs"  
    - Provides important user guidance on cost-saving behavior

#### 2.4 Metric Setting and Output

- **Overview:** This block records the computed similarity score metric back into the evaluation system for reporting and further use.
- **Nodes Involved:**  
  - `Set metrics`
  
- **Node Details:**

  - **Set metrics**  
    - Type: `evaluation` node  
    - Role: Sets the calculated similarity score as a numeric evaluation metric named "similarity"  
    - Configuration:  
      - Operation: "setMetrics"  
      - Metric assignment: `similarity = {{$json.message.content.score}}` (score extracted from previous OpenAI node output)  
    - Inputs: Correctness metric JSON output  
    - Outputs: Final metrics set for evaluation  
    - Edge cases: Missing or malformed score value

#### 2.5 Sticky Notes and Documentation

- **Nodes Involved:**  
  - `Sticky Note1`  
  - `Sticky Note3`  
  - `Sticky Note4`
  
- **Details:**

  - **Sticky Note1:**  
    - Content: "Check whether the answer has the same meaning as the expected answer"  
    - Context: Clarifies the core evaluation goal
    
  - **Sticky Note3:**  
    - Content:  
      ```
      ## How it works
      This template shows how to calculate a workflow evaluation metric: **whether an output matches an expected output** (i.e. has the same meaning).

      The workflow takes questions about the causes of historical events and compares them with the reference answers in the dataset.

      You can find more information on workflow evaluation [here](https://docs.n8n.io/advanced-ai/evaluations/overview), and other metric examples [here](https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics).
      ```
    - Context: High-level documentation and external links
    
  - **Sticky Note4:**  
    - Content:  
      ```
      Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=662663849#gid=662663849) of questions
      ```
    - Context: Reference to dataset source

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                                   | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                        |
|----------------------------|-------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| When fetching a dataset row | evaluationTrigger              | Trigger workflow on new dataset row              | (external)                  | Match chat format            | Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=662663849#gid=662663849) of questions |
| Match chat format           | set                           | Standardize question input field                  | When fetching a dataset row  | AI Agent                    |                                                                                                                    |
| AI Agent                   | langchain.agent                | Generate concise answer from question             | When chat message received, Match chat format | Evaluating?                 |                                                                                                                    |
| When chat message received  | langchain.chatTrigger          | Trigger workflow on incoming chat message         | (webhook)                   | AI Agent                    |                                                                                                                    |
| Evaluating?                | evaluation                    | Check if workflow is in evaluation mode           | AI Agent                    | Calculate correctness metric, Return chat response | Only calculate metrics if we're evaluating, to reduce costs                                                       |
| Calculate correctness metric| openAi                        | Compare output with ground truth and score similarity | Evaluating?                 | Set metrics                 | Check whether the answer has the same meaning as the expected answer                                               |
| Set metrics                | evaluation                    | Set similarity score metric                         | Calculate correctness metric|                             |                                                                                                                    |
| Return chat response       | noOp                          | Pass through response if not evaluating            | Evaluating?                 |                             |                                                                                                                    |
| OpenAI Chat Model          | lmChatOpenAi                  | Provides GPT-4o-mini model for AI Agent            | AI Agent                    | AI Agent                    |                                                                                                                    |
| Sticky Note1               | stickyNote                    | Guidance: check answer meaning                      |                             |                             | Check whether the answer has the same meaning as the expected answer                                               |
| Sticky Note3               | stickyNote                    | Documentation and links                             |                             |                             | [Workflow evaluation overview and metric examples](https://docs.n8n.io/advanced-ai/evaluations/overview)           |
| Sticky Note4               | stickyNote                    | Dataset link                                       |                             |                             | Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=662663849#gid=662663849) of questions |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "When fetching a dataset row"**  
   - Type: `evaluationTrigger`  
   - Configure with the Google Sheets document URL and sheet name containing the test dataset of questions and reference answers.  
   - Use OAuth2 credentials for Google Sheets access.

2. **Create Trigger Node: "When chat message received"**  
   - Type: `langchain.chatTrigger`  
   - Configure webhook ID (auto-generated or custom) to receive chat messages.

3. **Create "Match chat format" Node**  
   - Type: `set`  
   - Add assignment: `chatInput = {{$json.question}}`  
   - Connect input from "When fetching a dataset row" node.

4. **Create "AI Agent" Node**  
   - Type: `langchain.agent`  
   - Set input text to `={{ $json.chatInput }}`  
   - Configure system message:  
     "You are a helpful assistant. Answer the user's questions, but be very concise (max one sentence)"  
   - Set prompt type to `define`.  
   - Connect inputs from "When chat message received" and "Match chat format" nodes.

5. **Create "OpenAI Chat Model" Node**  
   - Type: `lmChatOpenAi`  
   - Set model to `"gpt-4o-mini"`  
   - Configure with OpenAI API credentials.  
   - Connect as AI language model for "AI Agent" node.

6. **Create "Evaluating?" Node**  
   - Type: `evaluation`  
   - Set operation to `checkIfEvaluating`  
   - Connect input from "AI Agent" output.

7. **Create "Calculate correctness metric" Node**  
   - Type: `openAi`  
   - Configure with model `gpt-4o-mini` and OpenAI credentials.  
   - Set system prompt per supplied detailed prompt that:  
     - Explains scoring criteria (1-5 scale), evaluation steps, JSON output format.  
     - Provides examples of scoring.  
     - Inputs: AI Agent output (as "Output") and ground truth answer from `When fetching a dataset row` node (`reference_answer` field).  
   - Enable JSON output parsing.  
   - Connect this node to the "true" branch of "Evaluating?" node.

8. **Create "Set metrics" Node**  
   - Type: `evaluation`  
   - Set operation to `setMetrics`  
   - Assign metric `"similarity"` with value from `{{$json.message.content.score}}` (score from previous node).  
   - Connect input from "Calculate correctness metric".

9. **Create "Return chat response" Node**  
   - Type: `noOp`  
   - Connect input from "false" branch of "Evaluating?" node (when not evaluating).

10. **Add Sticky Notes** (Optional for documentation and user guidance)  
    - "Check whether the answer has the same meaning as the expected answer" near "Calculate correctness metric" node.  
    - "Only calculate metrics if we're evaluating, to reduce costs" near "Evaluating?" node.  
    - Overview and dataset link notes as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This template shows how to calculate a workflow evaluation metric: whether an output matches an expected output (meaning).  | https://docs.n8n.io/advanced-ai/evaluations/overview                                                           |
| Other metric examples available here: metric-based evaluations documentation                                                 | https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics                        |
| Test dataset of questions used for evaluation                                                                              | https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=662663849#gid=662663849 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.