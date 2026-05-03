Generate & Optimize Brand Stories with Ollama LLMs and Google Sheets

https://n8nworkflows.xyz/workflows/generate---optimize-brand-stories-with-ollama-llms-and-google-sheets-4765


# Generate & Optimize Brand Stories with Ollama LLMs and Google Sheets

### 1. Workflow Overview

This workflow automates the generation and optimization of brand stories using large language models (LLMs) from Ollama, with integration into Google Sheets for storage. It targets marketing teams, brand managers, and content creators who want to create, evaluate, and refine brand narratives efficiently through AI-assisted processes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures brand storytelling requests via a chat trigger.
- **1.2 Brand Story Generation:** Uses an Ollama-based LLM via the Brand Storytelling Agent to create initial brand story content.
- **1.3 Brand Variable Setting:** Prepares and sets variables related to the brand story for downstream processing.
- **1.4 Story Evaluation:** Employs an Evaluator Agent to assess the generated brand story quality.
- **1.5 Decision Branching:** Determines whether the story passes evaluation or requires optimization.
- **1.6 Story Optimization:** Uses an Optimizer Agent with Ollama to refine the brand story if needed.
- **1.7 Persistence:** Saves the final approved brand story to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives the initial input request to start the brand story generation flow via a chat interface.

- **Nodes Involved:**  
  - `Chat Input: Start Story Flow`

- **Node Details:**  
  - **`Chat Input: Start Story Flow`**  
    - Type: Langchain Chat Trigger  
    - Role: Entry webhook that triggers the workflow when a user submits a prompt or request.  
    - Configuration: Default parameters with a unique webhook ID for triggering externally.  
    - Inputs: External HTTP request from chat clients.  
    - Outputs: Passes the chat input to the Brand Storytelling Agent node.  
    - Edge cases: Possible webhook timeout or malformed input. Should handle unexpected or empty inputs gracefully.

#### 1.2 Brand Story Generation

- **Overview:**  
  Generates an initial brand story based on the user's input using Ollama's LLM integrated via a Langchain agent.

- **Nodes Involved:**  
  - `Brand Storytelling Agent`  
  - `Ollama Chat Model`

- **Node Details:**  
  - **`Brand Storytelling Agent`**  
    - Type: Langchain Agent  
    - Role: Orchestrates the generation of brand story text using a connected language model.  
    - Configuration: Uses `Ollama Chat Model` as its AI language model.  
    - Inputs: Receives chat input from the trigger node.  
    - Outputs: Passes generated story to the `Set Brand variable` node.  
    - Edge cases: Agent may fail if AI model is unavailable or if input is ambiguous.  
  - **`Ollama Chat Model`**  
    - Type: Langchain Ollama LM Chat Node  
    - Role: Provides the underlying LLM response for the Brand Storytelling Agent.  
    - Configuration: Ollama chat model parameters (not explicitly detailed).  
    - Inputs: Text prompt from the agent.  
    - Outputs: Generated text for storytelling agent.  
    - Edge cases: Ollama API connection failures, rate limits, or malformed prompts.

#### 1.3 Brand Variable Setting

- **Overview:**  
  Sets or formats variables related to the brand story to prepare for evaluation and optimization.

- **Nodes Involved:**  
  - `Set Brand variable`

- **Node Details:**  
  - **`Set Brand variable`**  
    - Type: Set Node  
    - Role: Defines key variables or metadata for the brand story, probably formatting or extracting fields.  
    - Configuration: Custom expressions or static values to prepare data for evaluation and optimization.  
    - Inputs: Receives generated story from `Brand Storytelling Agent` or optimization output.  
    - Outputs: Feeds into `Evaluator Agent`.  
    - Edge cases: Expression errors if expected fields are missing or malformed.

#### 1.4 Story Evaluation

- **Overview:**  
  Evaluates the quality or compliance of the generated brand story using an AI evaluation agent.

- **Nodes Involved:**  
  - `Evaluator Agent`

- **Node Details:**  
  - **`Evaluator Agent`**  
    - Type: Langchain Agent  
    - Role: Uses an LLM to assess the brand story against evaluation criteria (quality, tone, etc.).  
    - Configuration: Uses `Ollama Chat Model` as AI language model input.  
    - Inputs: Receives brand story variables from `Set Brand variable`.  
    - Outputs: Passes evaluation results to `Evaluation Check` node.  
    - Edge cases: AI evaluation may be subjective; network or API errors possible.

#### 1.5 Decision Branching

- **Overview:**  
  Branches workflow depending on the evaluation result: if the story is acceptable, it proceeds to saving; if not, it triggers optimization.

- **Nodes Involved:**  
  - `Evaluation Check`

- **Node Details:**  
  - **`Evaluation Check`**  
    - Type: If node  
    - Role: Conditional branching based on evaluation output (pass/fail).  
    - Configuration: Checks a specific evaluation field or score to decide path.  
    - Inputs: From `Evaluator Agent`.  
    - Outputs:  
      - Main output 1 (True): To `Save Brand Story to Sheets`.  
      - Main output 2 (False): To `Optimizer Agent`.  
    - Edge cases: Missing or malformed evaluation data may cause false negatives.

#### 1.6 Story Optimization

- **Overview:**  
  Refines or improves the brand story using a separate optimization AI agent.

- **Nodes Involved:**  
  - `Optimizer Agent`  
  - `Ollama Chat Model1`

- **Node Details:**  
  - **`Optimizer Agent`**  
    - Type: Langchain Agent  
    - Role: Applies AI-driven optimization to the brand story, improving language, tone, or structure.  
    - Configuration: Uses `Ollama Chat Model1` as its AI language model.  
    - Inputs: Receives story from `Evaluation Check` false branch.  
    - Outputs: Sends optimized story to `Set Brand variable` for re-evaluation.  
    - Edge cases: Similar network or model availability issues as other agents.  
  - **`Ollama Chat Model1`**  
    - Type: Langchain Ollama LM Chat Node  
    - Role: LLM backend for the Optimizer Agent.  
    - Inputs: Text prompts from optimizer agent.  
    - Outputs: Optimized brand story text.  
    - Edge cases: Same as `Ollama Chat Model`.

#### 1.7 Persistence

- **Overview:**  
  Saves the final approved brand story to a Google Sheets document for record-keeping and further use.

- **Nodes Involved:**  
  - `Save Brand Story to Sheets`

- **Node Details:**  
  - **`Save Brand Story to Sheets`**  
    - Type: Google Sheets Node  
    - Role: Inserts or updates a row in a Google Sheets spreadsheet with the brand story data.  
    - Configuration: Requires Google Sheets credentials with appropriate scopes; target spreadsheet and sheet specified.  
    - Inputs: Receives the final evaluated brand story from the `Evaluation Check` true branch.  
    - Outputs: None (terminal node).  
    - Edge cases: Authentication failures, rate limits, or incorrect sheet configuration may cause errors.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                 | Input Node(s)                | Output Node(s)                  | Sticky Note                 |
|----------------------------|----------------------------------|--------------------------------|------------------------------|---------------------------------|-----------------------------|
| Sticky Note                | Sticky Note                      | Comment / Visual Aid            |                              |                                 |                             |
| Chat Input: Start Story Flow| Langchain Chat Trigger           | Entry point for brand story input|                              | Brand Storytelling Agent        |                             |
| Brand Storytelling Agent   | Langchain Agent                  | Generates initial brand story  | Chat Input: Start Story Flow  | Set Brand variable              |                             |
| Ollama Chat Model          | Langchain Ollama LM Chat         | LLM backend for storytelling   | Brand Storytelling Agent      | Brand Storytelling Agent        |                             |
| Set Brand variable         | Set Node                        | Sets variables for story data  | Brand Storytelling Agent, Optimizer Agent | Evaluator Agent               |                             |
| Evaluator Agent            | Langchain Agent                  | Evaluates brand story quality  | Set Brand variable            | Evaluation Check               |                             |
| Evaluation Check           | If Node                         | Decides accept or optimize     | Evaluator Agent               | Save Brand Story to Sheets, Optimizer Agent |                             |
| Optimizer Agent            | Langchain Agent                  | Optimizes story if needed      | Evaluation Check (false branch) | Set Brand variable              |                             |
| Ollama Chat Model1         | Langchain Ollama LM Chat         | LLM backend for optimizer      | Optimizer Agent               | Optimizer Agent                |                             |
| Save Brand Story to Sheets | Google Sheets                   | Saves final brand story        | Evaluation Check (true branch) |                                 |                             |
| Sticky Note1               | Sticky Note                      | Comment / Visual Aid            |                              |                                 |                             |
| Sticky Note2               | Sticky Note                      | Comment / Visual Aid            |                              |                                 |                             |
| Sticky Note3               | Sticky Note                      | Comment / Visual Aid            |                              |                                 |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **Langchain Chat Trigger** node named `Chat Input: Start Story Flow`.  
   - Configure the webhook (auto-generate or set your own webhook ID).  
   - This node will receive user inputs to start brand story generation.

2. **Create the Brand Storytelling Agent**  
   - Add a **Langchain Agent** node named `Brand Storytelling Agent`.  
   - Connect its input to `Chat Input: Start Story Flow`.  
   - Configure it to use the Ollama Chat Model as its AI language model.

3. **Add the Ollama Chat Model for Storytelling**  
   - Add a **Langchain Ollama LM Chat** node named `Ollama Chat Model`.  
   - Connect it as the AI language model input for `Brand Storytelling Agent`.  
   - Configure Ollama credentials and model parameters (e.g., model name, temperature).

4. **Add the Set Node for Brand Variables**  
   - Add a **Set** node named `Set Brand variable`.  
   - Connect its input from `Brand Storytelling Agent` output.  
   - Configure fields to extract or define brand story variables needed for evaluation, such as `storyText`, `brandName`, or metadata.

5. **Create the Evaluator Agent**  
   - Add a **Langchain Agent** node named `Evaluator Agent`.  
   - Connect input from `Set Brand variable`.  
   - Configure it to use the same `Ollama Chat Model` or a similar LLM for evaluation purposes.

6. **Add the Evaluation Check (If Node)**  
   - Add an **If** node named `Evaluation Check`.  
   - Connect input from `Evaluator Agent`.  
   - Configure the condition to check if the evaluation result meets a threshold (e.g., a score or pass/fail flag).

7. **Add the Save to Google Sheets Node**  
   - Add a **Google Sheets** node named `Save Brand Story to Sheets`.  
   - Connect input from the `true` output of `Evaluation Check`.  
   - Configure credentials for Google Sheets OAuth2 with proper scopes.  
   - Set the target spreadsheet ID and sheet name.  
   - Map fields from the brand story data to columns.

8. **Add the Optimizer Agent**  
   - Add a **Langchain Agent** node named `Optimizer Agent`.  
   - Connect input from the `false` output of `Evaluation Check`.  
   - Configure it to use a separate Ollama Chat Model for optimization (`Ollama Chat Model1`).

9. **Add the Ollama Chat Model for Optimization**  
   - Add a **Langchain Ollama LM Chat** node named `Ollama Chat Model1`.  
   - Connect as the AI language model for `Optimizer Agent`.  
   - Configure credentials and parameters similarly to the first Ollama Chat Model.

10. **Loop Back from Optimizer to Set Brand Variable**  
    - Connect the output of `Optimizer Agent` to the input of `Set Brand variable` for re-evaluation.

11. **Add Sticky Notes**  
    - Add **Sticky Note** nodes as needed for visual comments or instructions on the canvas (optional for workflow clarity).

12. **Validate and Test**  
    - Test the webhook trigger with sample brand story prompts.  
    - Confirm generation, evaluation, optimization, and Google Sheets saving work correctly.  
    - Handle errors such as authentication, API failures, or invalid inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                         |
|-----------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow uses Ollama LLMs via Langchain integration, requiring valid Ollama API credentials.| Ollama LLM API documentation          |
| Google Sheets node requires OAuth2 credentials with read/write access to target spreadsheets. | https://developers.google.com/sheets/api |
| Brand story evaluation criteria should be clearly defined in the `Evaluator Agent` prompts for consistent results.| Internal documentation or prompt design guidelines |
| Consider rate limits and API error handling for Ollama and Google Sheets in production usage. | n8n error handling best practices     |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.