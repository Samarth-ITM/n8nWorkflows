Evaluate AI Agent Response Relevance using OpenAI and Cosine Similarity

https://n8nworkflows.xyz/workflows/evaluate-ai-agent-response-relevance-using-openai-and-cosine-similarity-4425


# Evaluate AI Agent Response Relevance using OpenAI and Cosine Similarity

### 1. Workflow Overview

This workflow evaluates the relevance of an AI agent’s response to a user question by leveraging OpenAI models and cosine similarity of embeddings. The key use case is to assess how well the AI-generated answer aligns with the original question, which is particularly valuable for Q&A agents or conversational AI systems seeking quality control and continuous improvement.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Preprocessing:** Triggered by either a chat message or dataset row fetch, it normalizes and prepares input data for evaluation.
- **1.2 AI Agent Response Generation and Evaluation:** Invokes an AI agent to answer the question and initiates an evaluation process that involves generating a question from the answer, parsing output, and scoring.
- **1.3 Relevance Scoring and Metrics Update:** Computes embeddings for the original and generated questions, calculates cosine similarity as a relevance score, and updates Google Sheets with output and metrics.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

**Overview:**  
This block receives input either from a Google Sheets dataset or a chat message webhook. It remaps and structures the input for the AI agent component.

**Nodes Involved:**  
- When fetching a dataset row  
- Remap Input  
- When chat message received

**Node Details:**

- **When fetching a dataset row**  
  - *Type:* Evaluation Trigger (Google Sheets based)  
  - *Role:* Triggers workflow execution when a new row is fetched from a specified Google Sheet and sheet tab named “Relevance”.  
  - *Configuration:* Connected to a Google Sheets document with OAuth2 credentials; sheet and document IDs are linked to a public sample sheet.  
  - *Inputs:* None (trigger)  
  - *Outputs:* JSON containing row data including `input` field.  
  - *Potential Failures:* Authentication errors, Google Sheets API rate limits or unavailability.

- **Remap Input**  
  - *Type:* Set  
  - *Role:* Extracts and maps the `input` field from the fetched row into a new field `chatInput` for consistent downstream usage.  
  - *Key Expression:* `={{ $json.input }}`  
  - *Inputs:* From “When fetching a dataset row”  
  - *Outputs:* JSON with `chatInput` field set.  
  - *Potential Failures:* Expression evaluation failure if `input` is missing or malformed.

- **When chat message received**  
  - *Type:* LangChain Chat Trigger (Webhook)  
  - *Role:* Starts workflow when a chat message is received on a webhook endpoint (id: ba1fadeb-b566-469a-97b3-3159a99f1805).  
  - *Inputs:* Incoming webhook data representing user chat message.  
  - *Outputs:* JSON containing chat message content.  
  - *Potential Failures:* Webhook connectivity issues, malformed payloads.

---

#### 1.2 AI Agent Response Generation and Evaluation

**Overview:**  
This block runs the AI agent to generate an answer to the input question, then evaluates the answer’s relevance by generating a question from the answer and parsing the results.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model1  
- Evaluation  
- Set Input Fields  
- Answer Relevance  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Executes AI agent workflow, generating an answer to the input question.  
  - *Configuration:* Uses “OpenAI Chat Model1” as its language model.  
  - *Inputs:* `chatInput` from “Remap Input” or chat message trigger.  
  - *Outputs:* Generated response under `output` field.  
  - *Potential Failures:* API rate limits, model availability, malformed inputs.

- **OpenAI Chat Model1**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides GPT-4.1-mini model for the AI Agent node.  
  - *Credentials:* OpenAI API key configured.  
  - *Inputs:* From AI Agent node (as language model).  
  - *Outputs:* AI-generated response text.  
  - *Potential Failures:* API authentication, network timeouts.

- **Evaluation**  
  - *Type:* Evaluation node (built-in)  
  - *Role:* Checks if the current execution context is for evaluation and routes flow accordingly.  
  - *Inputs:* From AI Agent output.  
  - *Outputs:* Two paths: one to “Set Input Fields” for evaluation, one to “No Operation” (skip).  
  - *Potential Failures:* Misconfiguration of evaluation flags.

- **Set Input Fields**  
  - *Type:* Set  
  - *Role:* Sets two key fields — `question` (from original input) and `answer` (from AI response) for further evaluation.  
  - *Key Expressions:*  
    - `question = {{ $('When fetching a dataset row').first().json.input }}`  
    - `answer = {{ $json.output }}`  
  - *Inputs:* From Evaluation node (evaluation path).  
  - *Outputs:* JSON with `question` and `answer`.  
  - *Potential Failures:* Missing or undefined JSON fields.

- **Answer Relevance**  
  - *Type:* LangChain Chain LLM  
  - *Role:* Given the AI agent’s answer, generates a question that corresponds to this answer and determines if the answer is noncommittal (vague or evasive).  
  - *Prompt:* Instructs to generate a question from answer and label noncommittal answers as 1, committal as 0.  
  - *Inputs:* `answer` field from “Set Input Fields”  
  - *Outputs:* JSON with generated `question` and `noncommittal` indicator.  
  - *Potential Failures:* Model misinterpretation, prompt engineering issues.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides GPT-4.1-mini model for “Answer Relevance” node.  
  - *Credentials:* OpenAI API key configured.  
  - *Inputs:* From “Answer Relevance” as language model.  
  - *Outputs:* Structured JSON response.  
  - *Potential Failures:* API key limits, malformed output.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses the JSON output from “Answer Relevance” to ensure clean, structured data extraction for next steps.  
  - *Configuration:* Example schema with fields: `question` (string), `noncommittal` (number).  
  - *Inputs:* From “Answer Relevance” output.  
  - *Outputs:* Parsed JSON data.  
  - *Potential Failures:* Parsing errors if output deviates from schema.

---

#### 1.3 Relevance Scoring and Metrics Update

**Overview:**  
This block calculates the cosine similarity between embeddings of the original question and the AI-generated question and updates the evaluation results and metrics in the connected Google Sheets.

**Nodes Involved:**  
- Questions to Items  
- Get Embeddings  
- Calculate Similarity Score  
- Calculate Relevance Score  
- Update Output  
- Update Metrics  
- No Operation, do nothing

**Node Details:**

- **Questions to Items**  
  - *Type:* Code (JavaScript)  
  - *Role:* Prepares two items with text fields for embedding calculation: original question and generated question from the answer.  
  - *Code:*  
    ```js
    return [
      { json: { data: $('Set Input Fields').first().json.question } },
      { json: { data: $input.first().json.output.question } }
    ]
    ```  
  - *Inputs:* From “Answer Relevance” node.  
  - *Outputs:* Array of two JSON items with keys `data`.  
  - *Potential Failures:* Undefined input paths or missing fields.

- **Get Embeddings**  
  - *Type:* HTTP Request  
  - *Role:* Calls OpenAI embeddings API (`text-embedding-3-small`) to get vector embeddings for both questions.  
  - *Configuration:* POST to `https://api.openai.com/v1/embeddings`, input from “Questions to Items”.  
  - *Credentials:* OpenAI API key.  
  - *Inputs:* Two JSON items from “Questions to Items”.  
  - *Outputs:* Embedding arrays.  
  - *Potential Failures:* API rate limits, authentication failures, malformed requests.

- **Calculate Similarity Score**  
  - *Type:* Code (JavaScript)  
  - *Role:* Calculates cosine similarity between two embedding vectors to quantify semantic similarity.  
  - *Code Summary:*  
    - Extracts embeddings from input JSON.  
    - Computes dot product and norms, returns normalized cosine similarity score.  
  - *Inputs:* Embeddings from “Get Embeddings”.  
  - *Outputs:* JSON with `similarityScore`.  
  - *Potential Failures:* Missing embeddings, vector length mismatch.

- **Calculate Relevance Score**  
  - *Type:* Set  
  - *Role:* Multiplies cosine similarity by `(1 - noncommittal)` indicator to weigh score down if the answer is noncommittal.  
  - *Expression:*  
    ``` 
    score = similarityScore * Math.abs(!$('Answer Relevance').first().json.output.noncommittal)
    ```  
  - *Inputs:* From "Calculate Similarity Score" and "Answer Relevance".  
  - *Outputs:* JSON with final `score` numeric field.  
  - *Potential Failures:* Expression evaluation errors if fields missing.

- **Update Output**  
  - *Type:* Evaluation (Google Sheets update)  
  - *Role:* Writes back the `answer` and computed `score` to the Google Sheet row used as input for evaluation.  
  - *Configuration:* Points to same Google Sheet and sheet tab as input trigger.  
  - *Inputs:* From “Calculate Relevance Score”.  
  - *Outputs:* Confirmation of update.  
  - *Potential Failures:* Google Sheets API errors, permission issues.

- **Update Metrics**  
  - *Type:* Evaluation (Set Metrics)  
  - *Role:* Updates evaluation metrics, setting the `score` metric for reporting or analytics.  
  - *Inputs:* From “Update Output”.  
  - *Outputs:* Metrics updated context.  
  - *Potential Failures:* Misconfigured metric fields.

- **No Operation, do nothing**  
  - *Type:* NoOp  
  - *Role:* Placeholder node for alternative evaluation branch doing nothing.  
  - *Inputs:* From “Evaluation” (non-evaluating path).  
  - *Outputs:* None.  
  - *Potential Failures:* None.

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                                        | Input Node(s)                  | Output Node(s)                       | Sticky Note                                                                                              |
|-----------------------------|-------------------------------------|-------------------------------------------------------|-------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| When fetching a dataset row  | n8n-nodes-base.evaluationTrigger    | Trigger on new dataset row from Google Sheets          | -                             | Remap Input                        | Setup Your AI Workflow to Use Evaluations - [Learn more](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Remap Input                 | n8n-nodes-base.set                   | Map input field to `chatInput`                         | When fetching a dataset row    | AI Agent                          |                                                                                                        |
| When chat message received  | @n8n/n8n-nodes-langchain.chatTrigger| Trigger on chat webhook message                        | -                             | AI Agent                          |                                                                                                        |
| AI Agent                   | @n8n/n8n-nodes-langchain.agent       | Generate AI response to input question                  | Remap Input, When chat message | Evaluation                        |                                                                                                        |
| OpenAI Chat Model1          | @n8n/n8n-nodes-langchain.lmChatOpenAi| Language model for AI Agent (GPT-4.1-mini)             | AI Agent (lmChatOpenAi)        | AI Agent                         |                                                                                                        |
| Evaluation                 | n8n-nodes-base.evaluation             | Branch evaluation flow                                 | AI Agent                      | Set Input Fields, No Operation    |                                                                                                        |
| Set Input Fields           | n8n-nodes-base.set                    | Set `question` and `answer` fields for evaluation      | Evaluation                    | Answer Relevance                 |                                                                                                        |
| Answer Relevance           | @n8n/n8n-nodes-langchain.chainLlm    | Generate question from answer + identify noncommittal | Set Input Fields, OpenAI Chat Model, Structured Output Parser | Questions to Items        | Answer Relevance: How relevant is the agent response? - [Learn more](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi| Language model for Answer Relevance (GPT-4.1-mini)     | Answer Relevance (ai_languageModel) | Answer Relevance             |                                                                                                        |
| Structured Output Parser   | @n8n/n8n-nodes-langchain.outputParserStructured | Parses structured JSON output from Answer Relevance     | Answer Relevance (ai_outputParser) | Answer Relevance             |                                                                                                        |
| Questions to Items         | n8n-nodes-base.code                   | Prepare original and generated questions for embedding| Answer Relevance              | Get Embeddings                  |                                                                                                        |
| Get Embeddings             | n8n-nodes-base.httpRequest            | Call OpenAI embeddings API                             | Questions to Items            | Calculate Similarity Score       |                                                                                                        |
| Calculate Similarity Score | n8n-nodes-base.code                   | Compute cosine similarity between embeddings          | Get Embeddings                | Calculate Relevance Score        |                                                                                                        |
| Calculate Relevance Score  | n8n-nodes-base.set                    | Compute final relevance score weighted by noncommittal| Calculate Similarity Score    | Update Output                   |                                                                                                        |
| Update Output              | n8n-nodes-base.evaluation             | Update answer and score back to Google Sheet           | Calculate Relevance Score     | Update Metrics                  |                                                                                                        |
| Update Metrics             | n8n-nodes-base.evaluation             | Update evaluation metrics                               | Update Output                | -                              |                                                                                                        |
| No Operation, do nothing   | n8n-nodes-base.noOp                   | Placeholder for non-evaluation branch                   | Evaluation                   | -                              |                                                                                                        |
| Sticky Note1               | n8n-nodes-base.stickyNote             | Setup instructions for Evaluations Trigger             | -                             | -                              | Setup Your AI Workflow to Use Evaluations - See Section 1                                              |
| Sticky Note                | n8n-nodes-base.stickyNote             | Explanation of Answer Relevance concept                  | -                             | -                              | Answer Relevance: How relevant is the agent response? - See Section 1                                  |
| Sticky Note3               | n8n-nodes-base.stickyNote             | Full template explanation, requirements, and help links| -                             | -                              | Try It Out! n8n template for Answer Relevance scoring - See Section 1                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Google Sheets Evaluation Trigger** node named "When fetching a dataset row". Configure it with your Google Sheets OAuth2 credentials, and specify the Sheet ID and tab (“Relevance”) where evaluation data is stored.
   - Add a **LangChain Chat Trigger** node named "When chat message received" with webhook enabled to accept incoming chat messages.

2. **Map Input Fields:**
   - Add a **Set** node named "Remap Input".  
   - Set one assignment field:  
     - Name: `chatInput`  
     - Value: `={{ $json.input }}`  
   - Connect "When fetching a dataset row" → "Remap Input".

3. **Add AI Agent:**
   - Add a **LangChain Agent** node named "AI Agent".
   - Add a **OpenAI Chat Model** node named "OpenAI Chat Model1".  
     - Set model to "gpt-4.1-mini".  
     - Set credentials to your OpenAI API key.  
   - Connect "OpenAI Chat Model1" → "AI Agent" (as language model).  
   - Connect "Remap Input" → "AI Agent".  
   - Connect "When chat message received" → "AI Agent".

4. **Add Evaluation Branching:**
   - Add an **Evaluation** node named "Evaluation" with operation set to "checkIfEvaluating".
   - Connect "AI Agent" → "Evaluation".
   - Add a **Set** node named "Set Input Fields".  
     - Assign two fields:  
       - `question` = `={{ $('When fetching a dataset row').first().json.input }}`  
       - `answer` = `={{ $json.output }}`  
   - Connect "Evaluation" (evaluation path) → "Set Input Fields".
   - Add a **No Operation** node named "No Operation, do nothing".  
   - Connect "Evaluation" (non-evaluation path) → "No Operation, do nothing".

5. **Answer Relevance Chain:**
   - Add a **Chain LLM** node named "Answer Relevance".  
     - Set prompt text:  
       `"Generate a question for the given answer and Identify if answer is noncommittal. Give noncommittal as 1 if the answer is noncommittal and 0 if the answer is committal. A noncommittal answer is one that is evasive, vague, or ambiguous. For example, 'I don't know' or 'I'm not sure' are noncommittal answers."`  
     - Enable output parser.  
   - Add an **OpenAI Chat Model** node named "OpenAI Chat Model".  
     - Model: "gpt-4.1-mini" with OpenAI credentials.  
   - Add a **Structured Output Parser** node named "Structured Output Parser".  
     - Provide JSON schema example:  
       ```json
       {
         "question": "Where was Albert Einstein born?",
         "noncommittal": 0
       }
       ```  
   - Connect "Set Input Fields" → "Answer Relevance".  
   - Connect "OpenAI Chat Model" → "Answer Relevance" (language model).  
   - Connect "Structured Output Parser" → "Answer Relevance" (output parser).

6. **Prepare Questions for Embedding:**
   - Add a **Code** node named "Questions to Items".  
     - JavaScript code:  
       ```js
       return [
         { json: { data: $('Set Input Fields').first().json.question } },
         { json: { data: $input.first().json.output.question } }
       ];
       ```  
   - Connect "Answer Relevance" → "Questions to Items".

7. **Call OpenAI Embeddings API:**
   - Add an **HTTP Request** node named "Get Embeddings".  
     - Method: POST  
     - URL: `https://api.openai.com/v1/embeddings`  
     - Body Parameters:  
       - `input`: `={{ $json.data }}`  
       - `model`: `text-embedding-3-small`  
       - `encoding_format`: `float`  
     - Authentication: Use OpenAI API credentials.  
   - Connect "Questions to Items" → "Get Embeddings".

8. **Calculate Cosine Similarity:**
   - Add a **Code** node named "Calculate Similarity Score".  
     - JavaScript code:  
       ```js
       const [vectorsA, vectorsB] = $input.all().map(item => item.json.data[0].embedding);
       const score = cosineSimilarity(vectorsA, vectorsB);
       return { json: { similarityScore: score } };
       function cosineSimilarity(a, b) {  
         let dotProduct = normA = normB = 0;
         for (let i = 0; i < a.length; i++) {
           dotProduct += a[i] * b[i];
           normA += a[i] ** 2;
           normB += b[i] ** 2;
         }
         return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
       }
       ```  
   - Connect "Get Embeddings" → "Calculate Similarity Score".

9. **Calculate Final Relevance Score:**
   - Add a **Set** node named "Calculate Relevance Score".  
   - Assign field `score` with expression:  
     ``` 
     ={{ $json.similarityScore * Math.abs(!$('Answer Relevance').first().json.output.noncommittal) }}
     ```  
   - Connect "Calculate Similarity Score" → "Calculate Relevance Score".

10. **Update Google Sheets with Results:**
    - Add an **Evaluation** node named "Update Output".  
      - Configure to update the Google Sheet row with:  
        - `output` = `={{ $('Set Input Fields').first().json.answer }}`  
        - `score` = `={{ $json.score }}`  
      - Use same Sheet and document IDs as trigger.  
      - Credentials: Google Sheets OAuth2.  
    - Connect "Calculate Relevance Score" → "Update Output".

11. **Update Metrics:**
    - Add an **Evaluation** node named "Update Metrics".  
      - Set metric `score` with value: `={{ $json.score }}`  
      - Operation: setMetrics  
    - Connect "Update Output" → "Update Metrics".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Your AI Workflow to Use Evaluations. The Evaluations Trigger runs separately and pulls datasets from Google Sheets.                                                                                                   | [Evaluations Trigger Documentation](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Answer Relevance evaluation involves generating a question from the AI answer and measuring similarity to the original question, akin to the game show Jeopardy!.                                                           | Same as above link                                                                                                                            |
| The scoring approach adapts from a research metric repository for answer relevance: https://github.com/explodinggradients/ragas/blob/main/ragas/src/ragas/metrics/_answer_relevance.py                                         | GitHub repo                                                                                                                                    |
| Sample Google Sheet data can be found here for testing or adaptation: https://docs.google.com/spreadsheets/d/1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y/edit?usp=sharing                                                      | Sample Data Sheet                                                                                                                             |
| For community support and discussion, join the official n8n Discord or Forum.                                                                                                                                                 | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                                                           |

---

This document enables a technical user or AI agent to fully understand, reproduce, and extend the AI evaluation workflow with considerations for potential failure points, credential setup, and logical flow.