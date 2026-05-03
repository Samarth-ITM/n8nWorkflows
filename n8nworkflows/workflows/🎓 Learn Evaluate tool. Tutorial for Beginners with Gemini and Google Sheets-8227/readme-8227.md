🎓 Learn Evaluate tool. Tutorial for Beginners with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/---learn-evaluate-tool--tutorial-for-beginners-with-gemini-and-google-sheets-8227


# 🎓 Learn Evaluate tool. Tutorial for Beginners with Gemini and Google Sheets

### 1. Workflow Overview

This workflow is a beginner-level tutorial demonstrating how to use the **Evaluation tool** in n8n to automatically assess the correctness of AI-generated outputs by comparing them with known correct answers stored in a Google Sheet. It combines AI interaction via Google Gemini language models and a calculator tool to generate responses, then evaluates these responses against a ground truth to score factual correctness on a scale.

The workflow’s logic is organized into the following blocks:

- **1.1 Input Reception**: Receives input either manually or from a Google Sheet dataset row.
- **1.2 AI Processing**: Sends the input text to an AI agent powered by Google Gemini language models and a calculator tool to generate an output.
- **1.3 Evaluation Decision & Extraction**: Checks if evaluation should proceed, then extracts the AI-generated output.
- **1.4 Evaluation & Scoring**: Compares AI output with expected output from the dataset, calculates correctness metrics, and writes results back to the Google Sheet.
- **1.5 Output Handling**: Handles the evaluated results and optionally returns the AI response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives input data either from a manual trigger where the user provides a question or from a Google Sheet dataset row that holds input and expected output pairs for evaluation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- When fetching a dataset row (Evaluation Trigger)  
- Get manual input (Set)  
- Get input (Set)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual trigger  
  - *Role:* Starts the workflow manually for testing or demonstration.  
  - *Configuration:* No parameters; triggers downstream node.  
  - *Connections:* Outputs to “Get manual input”.  
  - *Failure Modes:* None typical; manual start.

- **When fetching a dataset row**  
  - *Type:* Evaluation Trigger (specialized Google Sheets trigger)  
  - *Role:* Watches a specific Google Sheet (ID: 1y6Gyjws8se12QX0uG4eS9il5-v29IpsgdFx3VqIYzWQ, sheet gid=0) for new or updated rows to trigger workflow execution.  
  - *Configuration:* Linked to Google Sheets OAuth2 credentials; monitors “Foglio1” sheet.  
  - *Connections:* Outputs to “Get input”.  
  - *Failure Modes:* OAuth token expiration, sheet access permission errors, sheet not found.  
  - *Version:* 4.6

- **Get manual input**  
  - *Type:* Set node  
  - *Role:* Defines static input text for manual runs (`How much is 8 * 3?`).  
  - *Configuration:* Sets a field `text` with the question string.  
  - *Connections:* Outputs to “AI Agent”.  
  - *Failure Modes:* Expression errors if field names are incorrect.  
  - *Version:* 3.4

- **Get input**  
  - *Type:* Set node  
  - *Role:* Extracts the `input` field from the dataset row into `text` for AI processing.  
  - *Configuration:* Sets `text` from `$json.input`.  
  - *Connections:* Outputs to “AI Agent”.  
  - *Failure Modes:* Missing or malformed input values.  
  - *Version:* 3.4

---

#### 1.2 AI Processing

**Overview:**  
The input text is passed to an AI Agent node that uses Google Gemini language models along with an integrated calculator tool to generate an answer.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Calculator

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Processes the input text using AI language models and tools, returning an answer.  
  - *Configuration:*  
    - Text input from `=$json.text`  
    - System message instructs to be helpful, use calculator for math, reply only with the result  
    - Returns intermediate steps for transparency  
  - *Connections:*  
    - Input from “Get input” or “Get manual input”  
    - AI language model: connected to “Google Gemini Chat Model”  
    - AI tool: connected to “Calculator”  
    - Output connected to “Evaluation?”  
  - *Failure Modes:* API authentication errors, rate limits, tool invocation failures, malformed input text.  
  - *Version:* 2.2

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini chat model node  
  - *Role:* Provides AI language model backend for the agent.  
  - *Configuration:* Uses Google Palm API credentials (configured with OAuth2).  
  - *Connections:* Connected as AI language model for “AI Agent”.  
  - *Failure Modes:* API key expiration, quota limits, network issues.  
  - *Version:* 1

- **Calculator**  
  - *Type:* LangChain calculator tool node  
  - *Role:* Performs math calculations requested by the AI agent.  
  - *Configuration:* No extra parameters.  
  - *Connections:* Connected as AI tool for “AI Agent”.  
  - *Failure Modes:* Calculation errors, invalid expressions, timeout.  
  - *Version:* 1

---

#### 1.3 Evaluation Decision & Extraction

**Overview:**  
Determines if the output should be evaluated, then extracts the AI-generated output for further processing.

**Nodes Involved:**  
- Evaluation? (Evaluation node)  
- Get output (Set)  
- Return chat response (NoOp)

**Node Details:**

- **Evaluation?**  
  - *Type:* Evaluation node  
  - *Role:* Checks if the current run is for evaluation (e.g., triggered by dataset row) or not.  
  - *Configuration:* Operation set to `checkIfEvaluating`.  
  - *Connections:*  
    - Input from “AI Agent”  
    - Outputs two branches:  
      - Main output to “Get output” (for evaluation path)  
      - Secondary output to “Return chat response” (for non-evaluation path)  
  - *Failure Modes:* Misconfigured evaluation flags, unexpected data format.  
  - *Version:* 4.7

- **Get output**  
  - *Type:* Set node  
  - *Role:* Extracts AI-generated answer from `$json.output` into a new field named `output`.  
  - *Configuration:* Sets `output` from AI Agent’s result.  
  - *Connections:* Outputs to “Set output Evaluation”.  
  - *Failure Modes:* Missing or null output fields.  
  - *Version:* 3.4

- **Return chat response**  
  - *Type:* NoOp node  
  - *Role:* Placeholder to return AI response directly when not evaluating.  
  - *Configuration:* None.  
  - *Connections:* Ends workflow branch.  
  - *Version:* 1

---

#### 1.4 Evaluation & Scoring

**Overview:**  
Writes AI output back to the Google Sheet, compares output with expected answers from the dataset, calculates a correctness metric, and writes this metric back to the sheet.

**Nodes Involved:**  
- Set output Evaluation (Evaluation node)  
- Corectness (Evaluation node)  
- Set correctness (Evaluation node)  
- Google Gemini Chat Model1

**Node Details:**

- **Set output Evaluation**  
  - *Type:* Evaluation node  
  - *Role:* Writes the AI output back to the Google Sheet as `actual_output` for the current row.  
  - *Configuration:*  
    - Outputs: `actual_output` = `{{$json.output}}`  
    - Same Google Sheet and sheet name as input trigger.  
    - Uses Google Sheets OAuth2 credentials.  
  - *Connections:* Outputs to “Corectness”.  
  - *Failure Modes:* Google Sheets write permission errors, network timeouts.  
  - *Version:* 4.7

- **Corectness**  
  - *Type:* Evaluation node  
  - *Role:* Calculates correctness metric by comparing AI output with expected answer.  
  - *Configuration:*  
    - Operation: `setMetrics`  
    - Metric name: `Correctness`  
    - Actual answer: from “Get output” node’s `output` field  
    - Expected answer: from “When fetching a dataset row” node’s `expected_output` field  
  - *Connections:*  
    - Input from “Set output Evaluation”  
    - AI language model: uses “Google Gemini Chat Model1” for evaluation logic  
    - Output to “Set correctness”  
  - *Failure Modes:* Metric calculation errors, mismatched fields, API errors.  
  - *Version:* 4.7

- **Set correctness**  
  - *Type:* Evaluation node  
  - *Role:* Writes the calculated correctness score back to the Google Sheet under the field `correctness`.  
  - *Configuration:*  
    - Output: `correctness` = `{{$json.Correctness}}`  
    - Same Google Sheet setup as above.  
    - Uses Google Sheets OAuth2 credentials.  
  - *Connections:* Ends workflow branch.  
  - *Failure Modes:* Same as “Set output Evaluation”.  
  - *Version:* 4.7

- **Google Gemini Chat Model1**  
  - *Type:* LangChain Google Gemini chat model node  
  - *Role:* Provides AI model for the correctness metric evaluation.  
  - *Configuration:* Uses same Google Palm API credentials as the first Gemini node.  
  - *Connections:* Connected as AI language model for the “Corectness” node.  
  - *Failure Modes:* Same as first Gemini model node.  
  - *Version:* 1

---

#### 1.5 Output Handling

**Overview:**  
Final nodes that handle output routing or workflow termination.

**Nodes Involved:**  
- Return chat response (NoOp) (already covered in 1.3)

---

### 3. Summary Table

| Node Name                   | Node Type                              | Functional Role                             | Input Node(s)                     | Output Node(s)               | Sticky Note                                                                                                       |
|-----------------------------|--------------------------------------|---------------------------------------------|----------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------|
| When fetching a dataset row  | Evaluation Trigger (Google Sheets)   | Triggers workflow on Google Sheet row data | None                             | Get input                    | Clone this [sheet](https://docs.google.com/spreadsheets/d/1y6Gyjws8se12QX0uG4eS9il5-v29IpsgdFx3VqIYzWQ/edit?usp=sharing) |
| When clicking ‘Execute workflow’ | Manual Trigger                      | Manual workflow start                        | None                             | Get manual input             |                                                                                                                  |
| Get manual input             | Set                                  | Sets static input text for manual runs      | When clicking ‘Execute workflow’ | AI Agent                    |                                                                                                                  |
| Get input                   | Set                                  | Extracts input from dataset row              | When fetching a dataset row      | AI Agent                    |                                                                                                                  |
| AI Agent                    | LangChain Agent                      | Runs AI processing with language model + calculator | Get input, Get manual input    | Evaluation?                 |                                                                                                                  |
| Google Gemini Chat Model     | LangChain LM Google Gemini           | AI language model for AI Agent               | None                             | AI Agent (ai_languageModel) |                                                                                                                  |
| Calculator                  | LangChain tool Calculator             | Calculator tool for math operations          | None                             | AI Agent (ai_tool)          |                                                                                                                  |
| Evaluation?                 | Evaluation                           | Determines evaluation path                    | AI Agent                        | Get output, Return chat response |                                                                                                                  |
| Get output                  | Set                                  | Extracts AI output for evaluation             | Evaluation?                    | Set output Evaluation        |                                                                                                                  |
| Return chat response        | NoOp                                 | Returns AI response if not evaluating         | Evaluation?                    | None                        |                                                                                                                  |
| Set output Evaluation       | Evaluation                           | Writes AI output back to Google Sheet         | Get output                     | Corectness                  |                                                                                                                  |
| Corectness                  | Evaluation                           | Calculates correctness metric                  | Set output Evaluation           | Set correctness             |                                                                                                                  |
| Set correctness             | Evaluation                           | Writes correctness metric back to Google Sheet | Corectness                    | None                        |                                                                                                                  |
| Google Gemini Chat Model1    | LangChain LM Google Gemini           | AI language model for correctness evaluation  | None                           | Corectness (ai_languageModel) |                                                                                                                  |
| Sticky Note                | Sticky Note                         | Documentation note                            | None                             | None                        | ## Learn Evaluate tool. Tutorial for Beginners\n\nThis workflow is a beginner-friendly tutorial demonstrating how to use the **Evaluation tool** to automatically score the AI’s output against a known correct answer (“ground truth”) stored in a Google Sheet.\n\nEvaluate the factual correctness of a given output compared to the provided ground truth on a scale from 1 to 5.\n\n\n- Clone this [sheet](https://docs.google.com/spreadsheets/d/1y6Gyjws8se12QX0uG4eS9il5-v29IpsgdFx3VqIYzWQ/edit?usp=sharing) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   a. Add a **Manual Trigger** node named `When clicking ‘Execute workflow’` (no parameters).

   b. Add an **Evaluation Trigger** node named `When fetching a dataset row` configured to monitor a Google Sheet:

   - Set Google Sheet document ID to `1y6Gyjws8se12QX0uG4eS9il5-v29IpsgdFx3VqIYzWQ`.
   - Set sheet name or gid to `gid=0`.
   - Set credentials to a configured Google Sheets OAuth2 credential with access permissions.

2. **Create Input Preparation Nodes:**

   a. Add a **Set** node named `Get manual input`:

   - Define an assignment: `text` = `"How much is 8 * 3?"`.
   - Connect the output of `When clicking ‘Execute workflow’` to this node.

   b. Add a **Set** node named `Get input`:

   - Define an assignment: `text` = `{{$json.input}}` (to extract input from fetched dataset row).
   - Connect the output of `When fetching a dataset row` to this node.

3. **Create AI Processing Nodes:**

   a. Add a **LangChain Google Gemini Chat Model** node named `Google Gemini Chat Model`:

   - Set Google Palm API credentials.
   - Leave options default.

   b. Add a **LangChain calculator tool** node named `Calculator`:

   - No parameters needed.

   c. Add a **LangChain Agent** node named `AI Agent`:

   - Set `text` input to `{{$json.text}}`.
   - Under options, add the system message:

     ```
     You are a helpful assistant.

     Use the calculator for math operations or tasks. 

     Reply only with the result
     ```

   - Enable return of intermediate steps.
   - Set AI language model to `Google Gemini Chat Model`.
   - Set AI tool to `Calculator`.
   - Connect outputs of `Get manual input` and `Get input` to `AI Agent`.

4. **Create Evaluation Decision Nodes:**

   a. Add an **Evaluation** node named `Evaluation?`:

   - Set operation to `checkIfEvaluating`.
   - Connect output of `AI Agent` to this node.

   - The node has two outputs:

     - Main output: connect to `Get output` node.
     - Secondary output: connect to `Return chat response`.

5. **Create Output Extraction and Handling Nodes:**

   a. Add a **Set** node named `Get output`:

   - Assign `output` = `{{$json.output}}`.
   - Connect main output of `Evaluation?` to this node.

   b. Add a **NoOp** node named `Return chat response`:

   - Connect secondary output of `Evaluation?` to this node.

6. **Create Evaluation & Scoring Nodes:**

   a. Add an **Evaluation** node named `Set output Evaluation`:

   - Configure to write an output field named `actual_output` = `{{$json.output}}` back to the Google Sheet.
   - Use the same Google Sheet document ID and sheet (gid=0) as earlier.
   - Use the same Google Sheets OAuth2 credentials.
   - Connect output of `Get output` to this node.

   b. Add a **LangChain Google Gemini Chat Model** node named `Google Gemini Chat Model1`:

   - Use the same Google Palm API credentials.
   - Connect as AI language model to the next node.

   c. Add an **Evaluation** node named `Corectness`:

   - Set operation to `setMetrics`.
   - Metric name: `Correctness`.
   - Actual answer: `{{$node["Get output"].json.output}}`.
   - Expected answer: `{{$node["When fetching a dataset row"].json.expected_output}}`.
   - Set AI language model to `Google Gemini Chat Model1`.
   - Connect output of `Set output Evaluation` to this node.

   d. Add an **Evaluation** node named `Set correctness`:

   - Configure to write output field `correctness` = `{{$json.Correctness}}` back to the Google Sheet.
   - Use same Google Sheet and credentials as above.
   - Connect output of `Corectness` to this node.

7. **Finalize Connections:**

   - Ensure all nodes are connected as described in the Summary Table above.
   - Validate credentials for Google Sheets and Google Palm API OAuth2 are correctly set up and authorized.
   - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                         | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is a beginner-friendly tutorial demonstrating how to use the **Evaluation tool** to automatically score the AI’s output against a known correct answer (“ground truth”) stored in a Google Sheet. Evaluate the factual correctness of a given output compared to the provided ground truth on a scale from 1 to 5. | Sticky Note in workflow; see also [Google Sheet clone link](https://docs.google.com/spreadsheets/d/1y6Gyjws8se12QX0uG4eS9il5-v29IpsgdFx3VqIYzWQ/edit?usp=sharing) |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.