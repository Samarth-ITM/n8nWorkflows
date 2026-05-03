Evaluate AI Agent Response Correctness with OpenAI and RAGAS Methodology

https://n8nworkflows.xyz/workflows/evaluate-ai-agent-response-correctness-with-openai-and-ragas-methodology-4424


# Evaluate AI Agent Response Correctness with OpenAI and RAGAS Methodology

### 1. Workflow Overview

This workflow evaluates the correctness of AI agent responses by comparing them against ground truth data using the RAGAS methodology. It integrates OpenAI’s GPT-4.1-mini model for semantic analysis and classification, and computes evaluation metrics focusing on answer correctness. The workflow is designed to:

- Receive input data from a Google Sheet dataset or chat messages.
- Use an AI agent to generate answers.
- Classify answer statements into True Positive, False Positive, and False Negative categories.
- Calculate semantic similarity scores using embeddings.
- Compute a combined correctness score weighted by classification and similarity.
- Update evaluation metrics back into the data source for monitoring.

The workflow’s logic is grouped into the following blocks:

- **1.1 Input Reception:** Triggers and input preprocessing from Google Sheets or chat.
- **1.2 AI Agent Response Generation:** Generates answers using the LangChain agent and OpenAI models.
- **1.3 Correctness Classification:** Uses a custom LangChain chain to classify statements comparing agent answers to ground truth.
- **1.4 Embeddings and Similarity Calculation:** Creates embeddings for answer and ground truth; calculates cosine similarity.
- **1.5 Scoring and Metrics Update:** Calculates F1 and combined correctness scores and updates evaluation metrics.
- **1.6 Utility and Control:** Includes no-operation nodes and merge nodes for flow control.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block triggers the workflow either from a Google Sheets evaluation trigger or from a chat message webhook. It prepares the input data for processing.
- **Nodes Involved:**  
  - When fetching a dataset row  
  - Remap Input  
  - When chat message received  

- **Node Details:**

  - **When fetching a dataset row**  
    - Type: Evaluation Trigger  
    - Role: Initiates workflow execution when a new row is fetched from a specified Google Sheet.  
    - Config: Reads from Google Sheet ID "1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y", sheet "gid=0".  
    - Input/Output: Outputs JSON with fields including `input`, `ground truth`.  
    - Potential Failures: Google Sheets OAuth failures, network timeouts, missing data.  
    - Notes: Manual trigger, isolated from production workflow.

  - **Remap Input**  
    - Type: Set Node  
    - Role: Maps the raw input from the trigger into a field named `chatInput` for downstream consumption.  
    - Config: Sets `chatInput` = `{{$json.input}}`.  
    - Input: Output of "When fetching a dataset row".  
    - Output: JSON with `chatInput` string.  
    - Notes: Simple reassignment, no complex logic.

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Alternative trigger that activates the workflow upon receiving a chat message.  
    - Config: Webhook enabled with ID `ba1fadeb-b566-469a-97b3-3159a99f1805`.  
    - Input/Output: Receives chat input, outputs to "AI Agent" node.  
    - Potential Failures: Webhook connectivity issues, malformed messages.

#### 1.2 AI Agent Response Generation

- **Overview:** This block uses the LangChain AI Agent integrated with OpenAI chat models to generate answers based on the input question.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model1  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Orchestrates the AI response generation using the OpenAI model.  
    - Config: Default options; uses OpenAI API credentials.  
    - Input: Receives `chatInput` from "Remap Input" or chat trigger.  
    - Output: Produces an `output` field with the agent’s answer text.  
    - Potential Failures: Model API rate limits, invalid credentials, timeout.  
    - Sub-workflow: None.

  - **OpenAI Chat Model1**  
    - Type: LangChain Chat Model (OpenAI)  
    - Role: Provides the underlying GPT-4.1-mini model for the AI Agent.  
    - Config: Model set to "gpt-4.1-mini" using OpenAI credentials.  
    - Input: Called internally by "AI Agent".  
    - Output: Returns chat completions.  
    - Dependencies: Requires valid OpenAI API key.  
    - Potential Failures: API key invalid, network errors.

#### 1.3 Correctness Classification

- **Overview:** This block classifies each statement in the AI agent’s answer against ground truth statements into True Positive (TP), False Positive (FP), and False Negative (FN) categories using a LangChain chain with OpenAI.
- **Nodes Involved:**  
  - Set Input Fields  
  - Correctness Classifier  
  - OpenAI Chat Model  
  - Examples1  
  - Calculate F1 Score  

- **Node Details:**

  - **Set Input Fields**  
    - Type: Set Node  
    - Role: Extracts and prepares `question`, `answer`, and splits `groundTruth` into an array from the fetched dataset row.  
    - Config:  
      - `question` = input question from dataset  
      - `answer` = output from AI Agent  
      - `groundTruth` = dataset’s ground truth split by newlines  
    - Inputs: Output of "When fetching a dataset row" and "AI Agent".  
    - Outputs: JSON fields for classification.  

  - **Correctness Classifier**  
    - Type: LangChain Chain LLM Node  
    - Role: Runs a prompt to classify statements as TP, FP, FN with reasons.  
    - Config:  
      - Prompt includes question, answers (list of statements), and ground truth statements.  
      - Instructions specify classification rules for each statement.  
      - Output parser enabled for structured JSON (TP, FP, FN arrays).  
    - Input: From "Set Input Fields".  
    - Output: JSON with classified statements and explanations.  
    - Potential Failures: Prompt parsing errors, unexpected output formats, API errors.

  - **OpenAI Chat Model**  
    - Type: LangChain Chat Model (OpenAI)  
    - Role: Provides GPT-4.1-mini model for the classifier chain.  
    - Config: Similar to the AI Agent’s model node.  
    - Input: Used internally by "Correctness Classifier".  
    - Output: Classification results.

  - **Examples1**  
    - Type: LangChain Output Parser Structured  
    - Role: Defines the expected JSON schema example for the classifier’s output to ensure correct parsing.  
    - Config: Includes example TP, FP, FN statements with reasons.  
    - Input: Connected to AI output parser of "Correctness Classifier".  
    - Output: Structured parsed output for further processing.

  - **Calculate F1 Score**  
    - Type: Code Node  
    - Role: Computes the F1 score based on counts of TP, FP, FN from classification.  
    - Config: JavaScript code implementing weighted F1 calculation with beta=1.  
    - Input: JSON output from "Correctness Classifier".  
    - Output: `{f1Score: number}`.  
    - Edge Cases: Handles zero division and empty categories.

#### 1.4 Embeddings and Similarity Calculation

- **Overview:** This block calculates semantic similarity between the AI agent’s answer and ground truth statements using OpenAI embeddings and cosine similarity.
- **Nodes Involved:**  
  - GroundTruth to Items  
  - Get Embeddings  
  - Remap Embeddings1  
  - Aggregate  
  - Get Embeddings1  
  - Remap Embeddings  
  - Create Embeddings Result  
  - Calculate Similarity Score  

- **Node Details:**

  - **GroundTruth to Items**  
    - Type: Split Out  
    - Role: Splits the ground truth array into individual items for embedding.  
    - Input: `groundTruth` array from "Set Input Fields".  
    - Output: One item per ground truth statement.

  - **Get Embeddings**  
    - Type: HTTP Request  
    - Role: Calls OpenAI embeddings API for ground truth statements.  
    - Config:  
      - URL: `https://api.openai.com/v1/embeddings`  
      - Model: `text-embedding-3-small`  
      - Input: Each ground truth statement  
      - Authentication: OpenAI API credentials  
    - Output: Embedding vectors for each ground truth statement.  
    - Potential Failures: API rate limits, network errors.

  - **Remap Embeddings1**  
    - Type: Set Node  
    - Role: Extracts embedding vector from API response and assigns to `data`.  
    - Input: Output from "Get Embeddings".  
    - Output: JSON with `data` field holding embedding array.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates all individual ground truth embedding items into a single array under `groundTruth`.  
    - Input: Multiple `data` embedding items from "Remap Embeddings1".  
    - Output: JSON with `groundTruth` array of embedding vectors.

  - **Get Embeddings1**  
    - Type: HTTP Request  
    - Role: Calls OpenAI embeddings API for the AI agent’s answer text.  
    - Config: Same as "Get Embeddings" but input is the full answer string.  
    - Output: Embedding vector for the answer.

  - **Remap Embeddings**  
    - Type: Set Node  
    - Role: Extracts embedding vector from API response and assigns to `answer`.  
    - Input: Output from "Get Embeddings1".  
    - Output: JSON with `answer` field holding embedding array.

  - **Create Embeddings Result**  
    - Type: Merge  
    - Role: Combines `answer` embedding and `groundTruth` embeddings into one JSON object.  
    - Input: From "Remap Embeddings" (answer) and "Aggregate" (groundTruth).  
    - Output: Merged JSON object with embeddings for similarity calculation.

  - **Calculate Similarity Score**  
    - Type: Code  
    - Role: Computes average cosine similarity score between answer embedding and each ground truth embedding.  
    - Config: JavaScript function implementing cosine similarity and averaging.  
    - Input: Combined embeddings JSON from "Create Embeddings Result".  
    - Output: `{similarityScore: number}` representing semantic similarity.  
    - Edge Cases: Empty arrays, zero vectors.

#### 1.5 Scoring and Metrics Update

- **Overview:** This block calculates a final weighted correctness score, updates outputs, and sets evaluation metrics in the dataset.
- **Nodes Involved:**  
  - Merge  
  - Correctness Score  
  - Update Outputs  
  - Update Metrics  

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from similarity score and F1 score calculations by position.  
    - Input: From "Calculate Similarity Score" and "Calculate F1 Score" (index 1).  
    - Output: Merged JSON with both scores.

  - **Correctness Score**  
    - Type: Code  
    - Role: Calculates final weighted correctness score using weights [0.75 for F1, 0.25 for similarity].  
    - Config: JavaScript weighted average function.  
    - Input: Merged JSON with `f1Score` and `similarityScore`.  
    - Output: `{score: number}` representing final correctness metric.

  - **Update Outputs**  
    - Type: Evaluation Node  
    - Role: Updates the evaluation outputs in the Google Sheet row with the agent’s answer and final score.  
    - Config:  
      - Outputs: `output` = agent’s answer, `score` = computed score  
      - Google Sheet and document IDs same as trigger node.  
    - Input: From "Correctness Score".  
    - Potential Failures: Google Sheets write permission errors, network issues.

  - **Update Metrics**  
    - Type: Evaluation Node  
    - Role: Sets the `score` metric in the evaluation system for monitoring.  
    - Input: From "Update Outputs".  
    - Output: None.  
    - Notes: Enables tracking metrics over time.

#### 1.6 Utility and Control

- **Overview:** Nodes used for flow control, no-ops, and documentation.
- **Nodes Involved:**  
  - No Operation, do nothing  
  - Sticky Notes (Sticky Note, Sticky Note1, Sticky Note3)  

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp Node  
    - Role: Placeholder node that performs no action.  
    - Input: From "Evaluation" node’s second output path.  
    - Use: For graceful handling or future extension.

  - **Sticky Notes**  
    - Type: Sticky Note Nodes  
    - Role: Provide detailed documentation and instructions visible inside the workflow editor.  
    - Content:  
      - Setup instructions for evaluation trigger.  
      - Explanation of answer correctness evaluation methodology.  
      - Template usage guide with links to relevant resources.  
    - Position: Placed near relevant nodes for contextual help.

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                                | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                                                                                                                                                  |
|----------------------------|-------------------------------------|------------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When fetching a dataset row | n8n-nodes-base.evaluationTrigger    | Workflow trigger from Google Sheets dataset   | None                             | Remap Input                     | Setup Your AI Workflow to Use Evaluations: [Learn more about the Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Remap Input                | n8n-nodes-base.set                  | Maps input field for AI agent input            | When fetching a dataset row      | AI Agent                       |                                                                                                                                                                                                                                              |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Alternative trigger via chat webhook            | None                             | AI Agent                       |                                                                                                                                                                                                                                              |
| AI Agent                   | @n8n/n8n-nodes-langchain.agent      | Generates AI agent answer                       | Remap Input, When chat message received | Evaluation                    |                                                                                                                                                                                                                                              |
| OpenAI Chat Model1         | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI GPT-4.1-mini model for AI Agent         | AI Agent (internal)              | AI Agent (internal)            |                                                                                                                                                                                                                                              |
| Evaluation                 | n8n-nodes-base.evaluation           | Checks if evaluation is running                 | AI Agent                        | Set Input Fields, No Operation  | Answer Correctness: Is the agent getting its facts correct? [Learn more about the Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Set Input Fields           | n8n-nodes-base.set                  | Prepares question, answer, and ground truth    | Evaluation                     | Correctness Classifier, Get Embeddings1, GroundTruth to Items |                                                                                                                                                                                                                                              |
| Correctness Classifier     | @n8n/n8n-nodes-langchain.chainLlm   | Classifies answer statements as TP, FP, FN     | Set Input Fields                | Calculate F1 Score             |                                                                                                                                                                                                                                              |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI GPT-4.1-mini model for classifier       | Correctness Classifier (internal) | Correctness Classifier (internal) |                                                                                                                                                                                                                                              |
| Examples1                  | @n8n/n8n-nodes-langchain.outputParserStructured | Defines JSON schema example for output parser | Correctness Classifier (ai_outputParser) | Correctness Classifier         |                                                                                                                                                                                                                                              |
| Calculate F1 Score         | n8n-nodes-base.code                 | Calculates F1 score from TP, FP, FN counts     | Correctness Classifier          | Merge                         |                                                                                                                                                                                                                                              |
| GroundTruth to Items       | n8n-nodes-base.splitOut             | Splits ground truth array into individual items | Set Input Fields               | Get Embeddings                |                                                                                                                                                                                                                                              |
| Get Embeddings             | n8n-nodes-base.httpRequest          | Gets embeddings for each ground truth statement | GroundTruth to Items           | Remap Embeddings1             |                                                                                                                                                                                                                                              |
| Remap Embeddings1          | n8n-nodes-base.set                  | Extracts embedding data from API response      | Get Embeddings                 | Aggregate                     |                                                                                                                                                                                                                                              |
| Aggregate                  | n8n-nodes-base.aggregate            | Aggregates all ground truth embeddings         | Remap Embeddings1              | Create Embeddings Result      |                                                                                                                                                                                                                                              |
| Get Embeddings1            | n8n-nodes-base.httpRequest          | Gets embedding for the AI agent’s answer       | Set Input Fields               | Remap Embeddings              |                                                                                                                                                                                                                                              |
| Remap Embeddings           | n8n-nodes-base.set                  | Extracts embedding data from API response      | Get Embeddings1                | Create Embeddings Result      |                                                                                                                                                                                                                                              |
| Create Embeddings Result   | n8n-nodes-base.merge                | Combines answer and ground truth embeddings    | Aggregate, Remap Embeddings    | Calculate Similarity Score    |                                                                                                                                                                                                                                              |
| Calculate Similarity Score | n8n-nodes-base.code                 | Computes cosine similarity between embeddings  | Create Embeddings Result       | Merge                        |                                                                                                                                                                                                                                              |
| Merge                     | n8n-nodes-base.merge                | Combines similarity and F1 scores               | Calculate Similarity Score, Calculate F1 Score | Correctness Score            |                                                                                                                                                                                                                                              |
| Correctness Score          | n8n-nodes-base.code                 | Calculates weighted correctness score          | Merge                         | Update Outputs               |                                                                                                                                                                                                                                              |
| Update Outputs             | n8n-nodes-base.evaluation           | Updates output answer and score in Google Sheet | Correctness Score             | Update Metrics               |                                                                                                                                                                                                                                              |
| Update Metrics             | n8n-nodes-base.evaluation           | Sets evaluation metric score                    | Update Outputs                | None                        |                                                                                                                                                                                                                                              |
| No Operation, do nothing   | n8n-nodes-base.noOp                 | Placeholder node for unused output path        | Evaluation                   | None                        |                                                                                                                                                                                                                                              |
| Sticky Note1               | n8n-nodes-base.stickyNote           | Documentation: Setup AI Workflow & Evaluations | None                         | None                        | See note in block 1.1                                                                                                                                                                                                                        |
| Sticky Note                | n8n-nodes-base.stickyNote           | Documentation: Answer correctness explanation  | None                         | None                        | See note in block 1.3                                                                                                                                                                                                                        |
| Sticky Note3               | n8n-nodes-base.stickyNote           | Documentation: Template usage and requirements | None                         | None                        | See note in overall workflow                                                                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Evaluation Trigger** node named "When fetching a dataset row".  
   - Configure Google Sheets OAuth credentials.  
   - Set Document ID: `1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y`.  
   - Set Sheet name: `gid=0`.  
   - This node triggers the workflow on new or updated rows.

2. **Add Input Remapping:**  
   - Add a **Set** node called "Remap Input".  
   - Create a string field `chatInput` with value `={{ $json.input }}`.  
   - Connect "When fetching a dataset row" → "Remap Input".

3. **Add Chat Webhook Trigger (optional):**  
   - Add **LangChain Chat Trigger** node "When chat message received".  
   - Enable webhook and note webhook URL.  
   - This node enables chat-driven testing.  

4. **Add AI Agent Node:**  
   - Add **LangChain Agent** node named "AI Agent".  
   - Connect "Remap Input" and "When chat message received" to "AI Agent".  
   - Assign OpenAI API credentials to an **OpenAI Chat Model** node "OpenAI Chat Model1" with model `gpt-4.1-mini`.  
   - Connect this model node as the language model inside "AI Agent".

5. **Add Evaluation Check Node:**  
   - Add **Evaluation** node "Evaluation" with operation "checkIfEvaluating".  
   - Connect "AI Agent" → "Evaluation".

6. **Set Input Fields:**  
   - Add **Set** node "Set Input Fields".  
   - Assign fields:  
     - `question` = `={{ $('When fetching a dataset row').first().json.input }}`  
     - `answer` = `={{ $json.output }}` (from AI Agent)  
     - `groundTruth` = `={{ $('When fetching a dataset row').first().json['ground truth'].split('\n') }}`  
   - Connect "Evaluation" → "Set Input Fields".

7. **Correctness Classification Chain:**  
   - Add **LangChain Chain LLM** node "Correctness Classifier".  
   - Configure prompt to classify statements into TP, FP, FN with reasons as per the original prompt.  
   - Enable output parser.  
   - Assign OpenAI API credentials to **OpenAI Chat Model** node "OpenAI Chat Model" with model `gpt-4.1-mini`.  
   - Connect this model to the classifier node.  
   - Connect "Set Input Fields" → "Correctness Classifier".

8. **Add Output Parser Example:**  
   - Add **LangChain Output Parser Structured** node "Examples1".  
   - Provide JSON schema example of TP, FP, FN classification with reasons.  
   - Connect "Correctness Classifier" → "Examples1" on the AI output parser path.

9. **Calculate F1 Score:**  
   - Add **Code** node "Calculate F1 Score".  
   - Paste JavaScript code to calculate F1 score from TP, FP, FN counts.  
   - Connect "Correctness Classifier" → "Calculate F1 Score".

10. **Process Ground Truth Embeddings:**  
    - Add **Split Out** node "GroundTruth to Items" to split `groundTruth` array.  
    - Connect "Set Input Fields" → "GroundTruth to Items".  

11. **Get Embeddings for Ground Truth:**  
    - Add **HTTP Request** node "Get Embeddings".  
    - Configure POST to `https://api.openai.com/v1/embeddings` with body parameters:  
      - `input` = current ground truth statement  
      - `model` = `text-embedding-3-small`  
      - `encoding_format` = `float`  
    - Set OpenAI credentials.  
    - Connect "GroundTruth to Items" → "Get Embeddings".

12. **Remap Ground Truth Embeddings:**  
    - Add **Set** node "Remap Embeddings1".  
    - Extract embedding array from response as `data`.  
    - Connect "Get Embeddings" → "Remap Embeddings1".

13. **Aggregate Ground Truth Embeddings:**  
    - Add **Aggregate** node "Aggregate".  
    - Aggregate all items under `groundTruth` array.  
    - Connect "Remap Embeddings1" → "Aggregate".

14. **Get Embeddings for Answer:**  
    - Add **HTTP Request** node "Get Embeddings1".  
    - Similar config as for ground truth but input is `answer` string.  
    - Set OpenAI credentials.  
    - Connect "Set Input Fields" → "Get Embeddings1".

15. **Remap Answer Embeddings:**  
    - Add **Set** node "Remap Embeddings".  
    - Extract embedding array from response as `answer`.  
    - Connect "Get Embeddings1" → "Remap Embeddings".

16. **Merge Embeddings:**  
    - Add **Merge** node "Create Embeddings Result".  
    - Mode: Combine by position.  
    - Inputs: `Aggregate` (groundTruth embeddings) and `Remap Embeddings` (answer embedding).  
    - Connect "Aggregate" → input 2, "Remap Embeddings" → input 1.

17. **Calculate Similarity:**  
    - Add **Code** node "Calculate Similarity Score".  
    - Implement cosine similarity between the answer embedding and each ground truth embedding, averaging scores.  
    - Connect "Create Embeddings Result" → "Calculate Similarity Score".

18. **Merge Scores:**  
    - Add **Merge** node "Merge".  
    - Combine outputs of "Calculate Similarity Score" and "Calculate F1 Score".  
    - Connect "Calculate Similarity Score" → input 1, "Calculate F1 Score" → input 2.

19. **Calculate Final Correctness Score:**  
    - Add **Code** node "Correctness Score".  
    - Calculate weighted average: 0.75 * F1 + 0.25 * similarity.  
    - Connect "Merge" → "Correctness Score".

20. **Update Outputs:**  
    - Add **Evaluation** node "Update Outputs".  
    - Configure to update Google Sheet outputs:  
      - Output: agent’s answer  
      - Score: final correctness score  
    - Use same Google Sheets credentials and IDs as trigger.  
    - Connect "Correctness Score" → "Update Outputs".

21. **Update Metrics:**  
    - Add **Evaluation** node "Update Metrics".  
    - Set metric `score` with value from final score.  
    - Connect "Update Outputs" → "Update Metrics".

22. **Add No-Op Node:**  
    - Add **No Operation** node "No Operation, do nothing".  
    - Connect second output of "Evaluation" node (if used for control flow) → "No Operation".

23. **Add Sticky Notes:**  
    - Add sticky notes near relevant nodes with the content from the original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                                                                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Your AI Workflow to Use Evaluations. The Evaluations Trigger runs independently and pulls datasets from Google Sheets automatically.                                                                                                                                                                                                                     | [Evaluations Trigger Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger)                                                                 |
| Answer Correctness measures the agent’s factuality and semantic similarity against ground truth. Best results are achieved with verbose, conversational agent responses.                                                                                                                                                                                         | [Evaluations Trigger Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger)                                                                 |
| This evaluation template demonstrates the "Correctness" metric, classifying answers into TP/FP/FN based on comparison with ground truths. The scoring approach is adapted from the RAGAS project on GitHub.                                                                                                                                                        | [RAGAS code reference](https://github.com/explodinggradients/ragas/blob/main/ragas/src/ragas/metrics/_answer_correctness.py)                                                                                                                                      |
| Sample Google Sheet data used: https://docs.google.com/spreadsheets/d/1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y/edit?usp=sharing                                                                                                                                                                                                                            | Sample dataset for testing                                                                                                                                                                                                                                    |
| For assistance, join the n8n Discord or community forum.                                                                                                                                                                                                                                                                                                          | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                                                                                                                                                                                 |

---

**Disclaimer:** The provided text and workflow come exclusively from an automated process created in n8n, respecting all applicable content policies and handling only legal, public data.