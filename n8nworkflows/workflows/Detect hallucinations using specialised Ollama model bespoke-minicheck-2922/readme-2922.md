Detect hallucinations using specialised Ollama model bespoke-minicheck

https://n8nworkflows.xyz/workflows/detect-hallucinations-using-specialised-ollama-model-bespoke-minicheck-2922


# Detect hallucinations using specialised Ollama model bespoke-minicheck

### 1. Workflow Overview

This workflow automates the fact-checking of a given text against a set of verified facts using AI models. It is designed to identify potential hallucinations or inaccuracies in the text by splitting it into sentences, verifying each sentence with a specialized Ollama model, filtering out sentences flagged as incorrect, and then summarizing the findings with a larger language model.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts inputs either manually or from another workflow, including the text to check and the list of verified facts.
- **1.2 Text Preparation:** Splits the input text into individual sentences while handling date formats and list elements carefully.
- **1.3 Fact Checking:** Each sentence is compared against the facts using the "bespoke-minicheck" Ollama model, which outputs a "Yes" or "No" indicating factual correctness.
- **1.4 Filtering and Aggregation:** Filters out sentences marked "No" (incorrect) and aggregates these results.
- **1.5 Summary Generation:** Uses a larger language model (Qwen2.5) to generate a structured summary of the fact-checking results, including counts and assessments.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block provides two entry points to start the workflow: manual trigger for testing and execution trigger for integration with other workflows. It also sets or receives the required inputs: `facts` and `text`.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Edit Fields  
  - When Executed by Another Workflow  

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for testing purposes.  
    - Configuration: No parameters; triggers downstream nodes on manual activation.  
    - Inputs: None  
    - Outputs: Triggers "Edit Fields" node.  
    - Edge Cases: None specific; manual trigger requires user action.

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Defines static input data for testing, setting `facts` and `text` fields with example content.  
    - Configuration: Assigns two string fields:  
      - `facts`: A multi-paragraph verified fact text about Sara Beery and salmon conservation.  
      - `text`: The article text to be checked.  
    - Inputs: Trigger from manual node  
    - Outputs: Sends data to "Code" and "Merge1" nodes.  
    - Edge Cases: If fields are empty or malformed, downstream nodes may fail.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows this workflow to be called from other workflows, passing `facts` and `text` as inputs.  
    - Configuration: Defines expected input parameters `facts` and `text`.  
    - Inputs: External workflow call  
    - Outputs: Triggers "Code" and "Merge1" nodes.  
    - Edge Cases: Missing inputs or incorrect data types may cause errors.

---

#### 2.2 Text Preparation

- **Overview:** Splits the input text into sentences, carefully handling date formats and list markers to avoid incorrect splits.
- **Nodes Involved:**  
  - Code  
  - Sticky Note1 (documentation)

- **Node Details:**

  - **Code**  
    - Type: Code Node (JavaScript)  
    - Role: Processes the input text to split it into sentences.  
    - Configuration:  
      - Runs once per input item.  
      - Implements a custom regex-based function that:  
        - Recognizes German month names and date formats to avoid splitting inside dates.  
        - Avoids splitting on list markers like hyphens or bullet points.  
      - Throws an error if input text is empty.  
      - Outputs an object with a `sentences` array.  
    - Inputs: Receives JSON with `text` field.  
    - Outputs: JSON with `sentences` array.  
    - Edge Cases:  
      - Empty or null input text triggers an error.  
      - Complex sentence structures or unexpected date formats might cause imperfect splits.

---

#### 2.3 Fact Checking

- **Overview:** Each sentence is individually checked against the verified facts using a specialized Ollama model ("bespoke-minicheck") that returns "Yes" or "No" indicating factual correctness.
- **Nodes Involved:**  
  - Merge1  
  - Split Out1  
  - Basic LLM Chain4  
  - Ollama Chat Model  
  - Sticky Note2 (documentation)

- **Node Details:**

  - **Merge1**  
    - Type: Merge Node  
    - Role: Combines the outputs of the "Code" node (sentences) and the input facts into paired items for processing.  
    - Configuration: Combines by position, pairing each sentence with the facts.  
    - Inputs:  
      - From "Code" (sentences)  
      - From "Edit Fields" or "When Executed by Another Workflow" (facts)  
    - Outputs: To "Split Out1".  
    - Edge Cases: Mismatched array lengths could cause pairing issues.

  - **Split Out1**  
    - Type: Split Out Node  
    - Role: Splits the combined array into individual items, each containing one sentence (`claim`) and the facts.  
    - Configuration: Splits on the `sentences` field renamed as `claim`.  
    - Inputs: From "Merge1".  
    - Outputs: To "Merge" and "Basic LLM Chain4".  
    - Edge Cases: Empty arrays or missing fields may cause errors.

  - **Basic LLM Chain4**  
    - Type: Langchain Chain LLM Node  
    - Role: Prepares the prompt text combining facts and the current claim for the Ollama model.  
    - Configuration:  
      - Prompt template:  
        ```
        Document: {{ $('Merge1').item.json.facts }}
        Claim: {{ $json.claim }}
        ```  
      - Prompt type: define  
    - Inputs: From "Split Out1".  
    - Outputs: To "Merge" node.  
    - Edge Cases: Missing facts or claim fields could cause prompt errors.

  - **Ollama Chat Model**  
    - Type: Langchain Ollama Chat Model Node  
    - Role: Runs the "bespoke-minicheck" Ollama model to verify each claim against the facts.  
    - Configuration:  
      - Model: `bespoke-minicheck:latest` (must be pre-installed via `ollama pull bespoke-minicheck`)  
      - No additional options set.  
    - Credentials: Ollama API credentials required.  
    - Inputs: From "Basic LLM Chain4" (prompt).  
    - Outputs: To "Basic LLM Chain4" (feeding back model response).  
    - Edge Cases:  
      - Model not installed or Ollama API authentication failure.  
      - Timeout or network issues.  
      - Unexpected model output format.

---

#### 2.4 Filtering and Aggregation

- **Overview:** Filters out sentences marked as factually incorrect ("No") and aggregates these filtered results for summary.
- **Nodes Involved:**  
  - Merge  
  - Filter  
  - Aggregate  
  - Sticky Note (build summary note)

- **Node Details:**

  - **Merge**  
    - Type: Merge Node  
    - Role: Combines the outputs from "Split Out1" (claims) and "Basic LLM Chain4" (model responses) by position to associate each claim with its verification result.  
    - Configuration: Combine by position.  
    - Inputs:  
      - From "Split Out1" (claims)  
      - From "Basic LLM Chain4" (model responses)  
    - Outputs: To "Filter".  
    - Edge Cases: Mismatched input lengths could cause pairing errors.

  - **Filter**  
    - Type: Filter Node  
    - Role: Filters the combined items to keep only those where the model response is "No" (indicating a hallucination or incorrect fact).  
    - Configuration:  
      - Condition: `$json.text === "No"` (case-sensitive, strict validation)  
    - Inputs: From "Merge".  
    - Outputs: To "Aggregate".  
    - Edge Cases:  
      - Model output variations (e.g., lowercase "no", extra whitespace) could cause misses.  
      - Empty input results in no output.

  - **Aggregate**  
    - Type: Aggregate Node  
    - Role: Aggregates all filtered items into a single data structure for summarization.  
    - Configuration: Aggregates all item data into one combined output.  
    - Inputs: From "Filter".  
    - Outputs: To "Basic LLM Chain".  
    - Edge Cases: Empty input results in empty aggregation.

---

#### 2.5 Summary Generation

- **Overview:** Generates a structured summary of the fact-checking results using a larger language model (Qwen2.5), including counts of errors, listing incorrect statements, and an overall assessment.
- **Nodes Involved:**  
  - Basic LLM Chain  
  - Ollama Model  
  - Sticky Note (build summary note)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: Langchain Chain LLM Node  
    - Role: Prepares the summarization prompt with aggregated data for the larger model.  
    - Configuration:  
      - Text input: `={{ $json.data }}` (aggregated filtered claims)  
      - Prompt message instructs the model to:  
        - Identify incorrect factual statements marked "no"  
        - Ignore chit-chat or non-factual statements  
        - Count errors  
        - Provide a structured summary with problem count, list, and final assessment  
      - Tone: Neutral, factual  
      - Output format: Markdown with sections for problem summary, list, and assessment  
    - Inputs: From "Aggregate".  
    - Outputs: To "Ollama Model".  
    - Edge Cases: Empty input data may produce empty or default summaries.

  - **Ollama Model**  
    - Type: Langchain Ollama Model Node  
    - Role: Runs the Qwen2.5 Ollama model to generate the final summary text.  
    - Configuration:  
      - Model: `qwen2.5:1.5b`  
      - No additional options set.  
    - Credentials: Ollama API credentials required.  
    - Inputs: From "Basic LLM Chain".  
    - Outputs: Final output of the workflow.  
    - Edge Cases: Model or API failures, timeouts.

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                      | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                  |
|-----------------------------|----------------------------------|------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                   | Manual start for testing            | None                           | Edit Fields                   | ## Test workflow                                                                                             |
| Edit Fields                 | Set Node                         | Defines test input facts and text  | When clicking ‘Test workflow’  | Code, Merge1                  |                                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger     | Entry point for external calls     | None                           | Code, Merge1                  | ## Entrypoint to use in other workflows                                                                      |
| Code                        | Code Node                       | Splits text into sentences         | Edit Fields, When Executed by Another Workflow | Merge1                       | ## Split into sentences                                                                                       |
| Merge1                      | Merge Node                      | Combines facts and sentences       | Code, Edit Fields/When Executed | Split Out1                   |                                                                                                              |
| Split Out1                  | Split Out Node                  | Splits combined data into claims   | Merge1                        | Merge, Basic LLM Chain4       | ## Fact checking  This use a small ollama model that is specialized on that task: https://ollama.com/library/bespoke-minicheck You have to install it before use with `ollama pull bespoke-minicheck`. |
| Basic LLM Chain4            | Langchain Chain LLM             | Prepares prompt for fact-checking  | Split Out1                    | Ollama Chat Model, Merge      |                                                                                                              |
| Ollama Chat Model           | Langchain Ollama Chat Model     | Runs bespoke-minicheck model        | Basic LLM Chain4              | Basic LLM Chain4              |                                                                                                              |
| Merge                       | Merge Node                      | Combines claims and model responses| Split Out1, Basic LLM Chain4  | Filter                       | ## Build a summary  This is useful to run it in an agentic workflow. You may remove the summary part and return the raw array with the found issues. |
| Filter                      | Filter Node                    | Filters sentences marked "No"      | Merge                        | Aggregate                    |                                                                                                              |
| Aggregate                   | Aggregate Node                 | Aggregates filtered incorrect claims| Filter                      | Basic LLM Chain              |                                                                                                              |
| Basic LLM Chain             | Langchain Chain LLM             | Prepares summarization prompt      | Aggregate                    | Ollama Model                 |                                                                                                              |
| Ollama Model                | Langchain Ollama Model          | Generates final summary             | Basic LLM Chain              | None                        |                                                                                                              |
| Sticky Note                 | Sticky Note                    | Documentation                      | None                         | None                        | ## Build a summary  This is useful to run it in an agentic workflow. You may remove the summary part and return the raw array with the found issues. |
| Sticky Note1                | Sticky Note                    | Documentation                      | None                         | None                        | ## Split into sentences                                                                                       |
| Sticky Note2                | Sticky Note                    | Documentation                      | None                         | None                        | ## Fact checking  This use a small ollama model that is specialized on that task: https://ollama.com/library/bespoke-minicheck You have to install it before use with `ollama pull bespoke-minicheck`. |
| Sticky Note3                | Sticky Note                    | Documentation                      | None                         | None                        | ## Test workflow                                                                                             |
| Sticky Note4                | Sticky Note                    | Documentation                      | None                         | None                        | ## Entrypoint to use in other workflows                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Set Node for Test Inputs**  
   - Name: `Edit Fields`  
   - Type: Set  
   - Add two string fields:  
     - `facts`: Paste the verified facts text about Sara Beery and salmon conservation.  
     - `text`: Paste the article text to be fact-checked.  
   - Connect output of `When clicking ‘Test workflow’` to this node.

3. **Create Execute Workflow Trigger Node**  
   - Name: `When Executed by Another Workflow`  
   - Type: Execute Workflow Trigger  
   - Define inputs: `facts` (string), `text` (string).  
   - No outputs connected yet.

4. **Create Code Node for Sentence Splitting**  
   - Name: `Code`  
   - Type: Code (JavaScript)  
   - Set mode to "Run Once For Each Item".  
   - Paste the provided JavaScript code that:  
     - Validates non-empty input text.  
     - Splits text into sentences using regex that respects German month names, dates, and list markers.  
   - Connect outputs of `Edit Fields` and `When Executed by Another Workflow` to this node.

5. **Create Merge Node to Combine Facts and Sentences**  
   - Name: `Merge1`  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Connect output of `Code` to input 2, and output of `Edit Fields` or `When Executed by Another Workflow` (facts) to input 1.

6. **Create Split Out Node**  
   - Name: `Split Out1`  
   - Type: Split Out  
   - Field to Split Out: `sentences` (from merged data)  
   - Destination Field Name: `claim`  
   - Connect output of `Merge1` to this node.

7. **Create Langchain Chain LLM Node for Prompt Preparation**  
   - Name: `Basic LLM Chain4`  
   - Type: Langchain Chain LLM  
   - Prompt Text:  
     ```
     Document: {{ $('Merge1').item.json.facts }}
     Claim: {{ $json.claim }}
     ```  
   - Prompt Type: Define  
   - Connect output of `Split Out1` to this node.

8. **Create Ollama Chat Model Node**  
   - Name: `Ollama Chat Model`  
   - Type: Langchain Ollama Chat Model  
   - Model: `bespoke-minicheck:latest` (ensure installed via `ollama pull bespoke-minicheck`)  
   - Credentials: Configure Ollama API credentials.  
   - Connect output of `Basic LLM Chain4` to this node.

9. **Connect Ollama Chat Model output back to Basic LLM Chain4**  
   - Connect output of `Ollama Chat Model` to input of `Basic LLM Chain4` (for chaining).

10. **Create Merge Node to Combine Claims and Model Responses**  
    - Name: `Merge`  
    - Type: Merge  
    - Mode: Combine  
    - Combine By: Position  
    - Connect outputs of `Split Out1` and `Basic LLM Chain4` to this node.

11. **Create Filter Node**  
    - Name: `Filter`  
    - Type: Filter  
    - Condition: `$json.text === "No"` (case-sensitive, strict)  
    - Connect output of `Merge` to this node.

12. **Create Aggregate Node**  
    - Name: `Aggregate`  
    - Type: Aggregate  
    - Aggregate: Aggregate All Item Data  
    - Connect output of `Filter` to this node.

13. **Create Langchain Chain LLM Node for Summary Preparation**  
    - Name: `Basic LLM Chain`  
    - Type: Langchain Chain LLM  
    - Text Input: `={{ $json.data }}` (aggregated filtered claims)  
    - Prompt Message: Provide detailed instructions to:  
      - Identify incorrect factual statements marked "no"  
      - Ignore chit-chat  
      - Count errors  
      - Provide structured summary with problem count, list, and final assessment  
    - Connect output of `Aggregate` to this node.

14. **Create Ollama Model Node for Final Summary**  
    - Name: `Ollama Model`  
    - Type: Langchain Ollama Model  
    - Model: `qwen2.5:1.5b`  
    - Credentials: Configure Ollama API credentials.  
    - Connect output of `Basic LLM Chain` to this node.

15. **Final Output**  
    - The output of `Ollama Model` is the final summarized fact-checking result.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The "bespoke-minicheck" Ollama model is specialized for fact-checking and must be installed before use.       | https://ollama.com/library/bespoke-minicheck        |
| To install the model: run `ollama pull bespoke-minicheck` in your environment before executing the workflow.  | Command line instruction                             |
| The workflow ignores chit-chat and focuses on verifiable factual statements only.                              | Workflow design note                                 |
| The summarization step can be removed or customized to return raw data instead of a summary.                   | Customization option                                 |
| Ollama API credentials are required for both the fact-checking and summarization models.                       | Credential setup required                            |
| The workflow can be integrated into larger editorial or agentic workflows via the "When Executed by Another Workflow" node. | Integration note                                     |

---

This documentation provides a detailed and structured reference to understand, reproduce, and modify the fact-checking workflow using n8n and Ollama models. It anticipates potential errors such as empty inputs, model installation issues, and data mismatches, enabling robust deployment and integration.