Compare GPT-4, Claude & Gemini Responses with Contextual AI's LMUnit Evaluation

https://n8nworkflows.xyz/workflows/compare-gpt-4--claude---gemini-responses-with-contextual-ai-s-lmunit-evaluation-11618


# Compare GPT-4, Claude & Gemini Responses with Contextual AI's LMUnit Evaluation

### 1. Workflow Overview

This workflow automates the evaluation and comparison of responses from multiple large language models (LLMs)—specifically OpenAI GPT-4.1, Anthropic Claude 4.5 Sonnet, and Google Gemini 2.5 Flash—using Contextual AI’s LMUnit, a natural language unit testing framework. It is designed to provide systematic, fine-grained quality feedback on LLM outputs, focusing on clarity and conciseness criteria.

The logical structure is organized into these main blocks:

- **1.1 Input Reception**: Captures user chat input and simultaneously sends it to three LLMs for response generation.
- **1.2 Response Preprocessing and Combination**: Normalizes and merges the different model responses into a unified data structure.
- **1.3 Unit Test Association**: Attaches predefined natural language unit tests to each model response.
- **1.4 Unit Test Iteration and Evaluation**: Iterates through each test for every response, invoking Contextual AI’s LMUnit to score responses.
- **1.5 Score Association and Grouping**: Associates scores with responses, groups results by provider and test, and aggregates scores.
- **1.6 Final Result Formatting and Output**: Formats the aggregated results into a human-readable summary and sends it as a chat response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives chat messages from the user and triggers response generation by three different LLMs in parallel.

**Nodes Involved:**  
- When chat message received  
- OpenAI GPT 4.1  
- Gemini 2.5 Flash  
- Claude 4.5 Sonnet  

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point; listens for incoming chat messages.  
  - Config: Webhook-based trigger, configured to output response nodes mode.  
  - Outputs to: OpenAI GPT 4.1, Gemini 2.5 Flash, Claude 4.5 Sonnet.  
  - Edge cases: Webhook failures, malformed input, connection issues.  

- **OpenAI GPT 4.1**  
  - Type: LangChain OpenAI node  
  - Role: Generates response from OpenAI GPT-4.1 model using user input.  
  - Config: Model set to “gpt-4.1”; input message passed as chatInput expression from trigger.  
  - Credentials: OpenAI API key required.  
  - Output: Raw OpenAI response JSON.  
  - Edge cases: API rate limits, authentication errors, invalid prompt errors, timeout.  

- **Gemini 2.5 Flash**  
  - Type: LangChain Google Gemini node  
  - Role: Generates response from Google Gemini 2.5 Flash model.  
  - Config: Model set to “models/gemini-2.5-flash”; input message from chatInput expression.  
  - Credentials: Google Palm API key required.  
  - Output: Raw Gemini response JSON.  
  - Edge cases: API key invalid, quota exceeded, network errors.  

- **Claude 4.5 Sonnet**  
  - Type: LangChain Anthropic node  
  - Role: Generates Anthropic Claude 4.5 Sonnet response.  
  - Config: Model set to “claude-sonnet-4-5-20250929”; input message from chatInput.  
  - Credentials: Anthropic API key required.  
  - Output: Raw Anthropic response JSON.  
  - Edge cases: API errors, rate limiting, invalid input format.  

---

#### 1.2 Response Preprocessing and Combination

**Overview:**  
Extracts and normalizes the text content from each model’s raw response, labels the provider, then merges all responses into one data set for unified processing.

**Nodes Involved:**  
- Preprocess OpenAI Response  
- Preprocess Gemini Response  
- Preprocess Anthropic Response  
- Combine responses  

**Node Details:**

- **Preprocess OpenAI Response**  
  - Type: Set node  
  - Role: Extracts OpenAI’s message content and assigns “OpenAI” as provider.  
  - Config: Sets `provider` = "OpenAI"; extracts `response` = message.content from OpenAI output JSON.  
  - Input: Output from OpenAI GPT 4.1.  
  - Output: Simplified JSON with provider and response fields.  
  - Edge cases: Missing message content, unexpected JSON structure.  

- **Preprocess Gemini Response**  
  - Type: Set node  
  - Role: Extracts Gemini’s textual response and assigns provider label.  
  - Config: Sets `provider` = "Gemini"; extracts `response` = content.parts[0].text.  
  - Input: Gemini 2.5 Flash output.  
  - Output: Normalized JSON with provider and response.  
  - Edge cases: Missing content array or unexpected array length.  

- **Preprocess Anthropic Response**  
  - Type: Set node  
  - Role: Extracts Anthropic Claude response text and labels provider.  
  - Config: Sets `provider` = "Anthropic"; extracts `response` = content[0].text.  
  - Input: Claude 4.5 Sonnet output.  
  - Output: JSON with provider and response fields.  
  - Edge cases: Missing content array or malformed response.  

- **Combine responses**  
  - Type: Merge node  
  - Role: Merges the three preprocessed response streams into a single unified stream.  
  - Config: Set to merge 3 inputs.  
  - Inputs: From Preprocess OpenAI Response, Preprocess Gemini Response, Preprocess Anthropic Response.  
  - Output: Combined stream of all provider responses.  
  - Edge cases: Mismatch in input counts, data loss in merging.  

---

#### 1.3 Unit Test Association

**Overview:**  
Expands each response to produce multiple items, each associated with a different unit test (quality criterion) for evaluation.

**Nodes Involved:**  
- Add unit tests to responses  

**Node Details:**

- **Add unit tests to responses**  
  - Type: Code node (JavaScript)  
  - Role: For each response, creates copies with different unit tests attached.  
  - Config: Uses predefined unit tests:  
    1. “Is the response clear and easy to understand?” (Clarity)  
    2. “Does the response use precise and accurate information?” (Conciseness)  
  - Input: Combined responses stream.  
  - Output: Expanded list where each original response is duplicated for each test, containing fields: provider, response, unit_test.  
  - Edge cases: Empty input, ensuring no data duplication errors, preserving original response integrity.  

---

#### 1.4 Unit Test Iteration and Evaluation

**Overview:**  
Processes each (response + unit test) pair by invoking Contextual AI's LMUnit API to evaluate quality and score the response on the given test. Includes a wait node to respect API rate limits or pacing.

**Nodes Involved:**  
- Iterate over each unit tests  
- Wait for 3 sec  
- Run LMUnit  

**Node Details:**

- **Iterate over each unit tests**  
  - Type: SplitInBatches node  
  - Role: Processes each unit test item individually or in small batches to control flow.  
  - Config: Default batching (likely batch size 1).  
  - Input: Expanded response + unit test items.  
  - Outputs: Two outputs—one proceeds to formatting, the other to waiting + LMUnit evaluation.  
  - Edge cases: Handling large number of batches, potential timeouts.  

- **Wait for 3 sec**  
  - Type: Wait node  
  - Role: Pauses execution for 3 seconds to manage API call pacing.  
  - Config: Fixed 3 seconds delay.  
  - Input: From Iterate over each unit tests (second output).  
  - Output: Connects to Run LMUnit.  
  - Edge cases: Possible workflow delays, concurrency effects.  

- **Run LMUnit**  
  - Type: Contextual AI node  
  - Role: Calls Contextual AI LMUnit service to evaluate response against the unit test.  
  - Config: Takes parameters:  
    - `query` = user input message  
    - `response` = model response string  
    - `unitTest` = unit test question/criteria string  
  - Credentials: Requires valid Contextual AI API key.  
  - Input: Response + unit test pairs post-wait.  
  - Output: Evaluation scores (1-5) returned.  
  - Edge cases: API errors, invalid input formats, auth issues, rate limits.  

---

#### 1.5 Score Association and Grouping

**Overview:**  
Associates the LMUnit evaluation scores back with the original response data, groups results by provider and unit test, and aggregates the scores for summary generation.

**Nodes Involved:**  
- Associate scores with Responses  
- Group Results Together  

**Node Details:**

- **Associate scores with Responses**  
  - Type: Set node  
  - Role: Combines the original response data (provider, response, unit_test) with the LMUnit score returned.  
  - Config: Sets `provider`, `response`, `unit_test` from original; adds `score` from LMUnit output JSON.  
  - Input: From Run LMUnit output.  
  - Output: JSON records with full evaluation data per test per provider.  
  - Edge cases: Missing scores, data misalignment.  

- **Group Results Together**  
  - Type: Code node (JavaScript)  
  - Role: Groups all evaluation entries by provider and unit test; constructs nested object structure.  
  - Logic:  
    - For each record, indexes by `provider` and `unit_test`, storing response and numeric score.  
  - Output: Single JSON object with grouped results for all providers and tests.  
  - Edge cases: Missing fields, empty inputs.  

---

#### 1.6 Final Result Formatting and Output

**Overview:**  
Formats the grouped evaluation results into a readable text summary message and sends it back to the chat interface.

**Nodes Involved:**  
- Format Final Result  
- Final Response  

**Node Details:**

- **Format Final Result**  
  - Type: Code node (JavaScript)  
  - Role: Iterates over grouped results and constructs a multi-line text summary including:  
    - Provider name  
    - Model response text (from first unit test)  
    - Scores for each evaluation criterion  
  - Output: JSON with single field `message` containing formatted string.  
  - Edge cases: Empty grouped results, string formatting issues.  

- **Final Response**  
  - Type: LangChain Chat node  
  - Role: Sends the formatted evaluation summary back to the user in chat.  
  - Config: Message content set from `Format Final Result` output.  
  - Edge cases: Failures in sending response, chat interface errors.  

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                           | Input Node(s)                        | Output Node(s)                    | Sticky Note                                                                                                      |
|-------------------------------|----------------------------------|-----------------------------------------|------------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------|
| When chat message received     | LangChain Chat Trigger            | Entry point for user input               | -                                  | OpenAI GPT 4.1, Gemini 2.5 Flash, Claude 4.5 Sonnet | ### 1: User Message Submission<br>Captures input, sends to 3 LLMs simultaneously                               |
| OpenAI GPT 4.1                | LangChain OpenAI                  | Generates GPT-4.1 response               | When chat message received          | Preprocess OpenAI Response        |                                                                                                                  |
| Gemini 2.5 Flash              | LangChain Google Gemini           | Generates Gemini 2.5 Flash response      | When chat message received          | Preprocess Gemini Response        |                                                                                                                  |
| Claude 4.5 Sonnet            | LangChain Anthropic               | Generates Claude 4.5 Sonnet response     | When chat message received          | Preprocess Anthropic Response     |                                                                                                                  |
| Preprocess OpenAI Response    | Set                              | Extracts and labels OpenAI response      | OpenAI GPT 4.1                     | Combine responses                |                                                                                                                  |
| Preprocess Gemini Response    | Set                              | Extracts and labels Gemini response      | Gemini 2.5 Flash                   | Combine responses                |                                                                                                                  |
| Preprocess Anthropic Response | Set                              | Extracts and labels Anthropic response   | Claude 4.5 Sonnet                 | Combine responses                |                                                                                                                  |
| Combine responses             | Merge                            | Merges all preprocessed responses        | Preprocess OpenAI, Gemini, Anthropic | Add unit tests to responses       |                                                                                                                  |
| Add unit tests to responses   | Code                             | Attaches predefined unit tests to responses | Combine responses                  | Iterate over each unit tests       | ### 2: Associate unit tests with language model responses<br>Assigns clarity and conciseness tests              |
| Iterate over each unit tests  | SplitInBatches                   | Processes each (response + unit test) individually | Add unit tests to responses        | Format Final Result, Wait for 3 sec |                                                                                                                  |
| Wait for 3 sec               | Wait                             | Delays flow to manage API rate limits    | Iterate over each unit tests         | Run LMUnit                       |                                                                                                                  |
| Run LMUnit                   | Contextual AI                    | Evaluates responses against unit tests   | Wait for 3 sec                    | Associate scores with Responses   | ### 3: Use Contextual AI's LMUnit to evaluate responses<br>Generates evaluation scores and aggregates results   |
| Associate scores with Responses | Set                            | Combines evaluation scores with responses | Run LMUnit                       | Group Results Together            |                                                                                                                  |
| Group Results Together        | Code                             | Groups evaluation data by provider and test | Associate scores with Responses    | Iterate over each unit tests       |                                                                                                                  |
| Format Final Result           | Code                             | Formats grouped results into summary text | Iterate over each unit tests       | Final Response                   |                                                                                                                  |
| Final Response               | LangChain Chat                   | Sends formatted evaluation summary to user | Format Final Result               | -                                |                                                                                                                  |
| Sticky Note                  | Sticky Note                      | Project overview and detailed explanation | -                                | -                                | Multi-Model Response Evaluation using Contextual AI’s LMUnit; detailed workflow explanation and setup notes     |
| Sticky Note1                 | Sticky Note                      | Describes input reception block          | -                                | -                                | User message submission and parallel LLM response generation                                                   |
| Sticky Note2                 | Sticky Note                      | Describes unit test association block    | -                                | -                                | Associating unit tests (clarity, conciseness) with model responses                                              |
| Sticky Note3                 | Sticky Note                      | Describes LMUnit evaluation block        | -                                | -                                | Using Contextual AI LMUnit for scoring and final result aggregation                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a LangChain Chat Trigger node**  
   - Name: "When chat message received"  
   - Set as webhook trigger for chat messages.  
   - Configure to output response nodes.  

2. **Add OpenAI GPT 4.1 node**  
   - Name: "OpenAI GPT 4.1"  
   - Set model to "gpt-4.1".  
   - Map input message content from trigger’s chatInput.  
   - Assign OpenAI API credentials.  
   - Connect input from "When chat message received".  

3. **Add Google Gemini 2.5 Flash node**  
   - Name: "Gemini 2.5 Flash"  
   - Set model to "models/gemini-2.5-flash".  
   - Map input message from chatInput.  
   - Assign Google Palm API credentials.  
   - Connect input from "When chat message received".  

4. **Add Anthropic Claude 4.5 Sonnet node**  
   - Name: "Claude 4.5 Sonnet"  
   - Set model to "claude-sonnet-4-5-20250929".  
   - Map input message from chatInput.  
   - Assign Anthropic API credentials.  
   - Connect input from "When chat message received".  

5. **Create Preprocess nodes for each provider**:

   - **Preprocess OpenAI Response** (Set node)  
     - Assign `provider` = "OpenAI"  
     - Extract `response` = `message.content` from OpenAI output JSON  
     - Connect input from "OpenAI GPT 4.1"  

   - **Preprocess Gemini Response** (Set node)  
     - Assign `provider` = "Gemini"  
     - Extract `response` = `content.parts[0].text` from Gemini output  
     - Connect input from "Gemini 2.5 Flash"  

   - **Preprocess Anthropic Response** (Set node)  
     - Assign `provider` = "Anthropic"  
     - Extract `response` = `content[0].text` from Anthropic output  
     - Connect input from "Claude 4.5 Sonnet"  

6. **Add a Merge node ("Combine responses")**  
   - Set number of inputs to 3.  
   - Connect each Preprocess node’s output to one input of this Merge node.  

7. **Create a Code node ("Add unit tests to responses")**  
   - JavaScript code to:  
     - Define unit tests:  
       - "Is the response clear and easy to understand?"  
       - "Does the response use precise and accurate information?"  
     - For each input item, duplicate it for each unit test with added `unit_test` field.  
   - Connect input from "Combine responses".  

8. **Add SplitInBatches node ("Iterate over each unit tests")**  
   - Default batch size (1).  
   - Connect input from "Add unit tests to responses".  
   - Create two outputs:  
     - Output 1: Connect to "Format Final Result" (step 12)  
     - Output 2: Connect to "Wait for 3 sec" (step 9)  

9. **Add Wait node ("Wait for 3 sec")**  
   - Set delay to 3 seconds.  
   - Connect input from second output of "Iterate over each unit tests".  
   - Connect output to "Run LMUnit".  

10. **Add Contextual AI node ("Run LMUnit")**  
    - Resource: LMUnit  
    - Parameters:  
      - `query` = user chat input from trigger (expression)  
      - `response` = current response string  
      - `unitTest` = current unit test string  
    - Assign Contextual AI API credentials.  
    - Connect input from "Wait for 3 sec".  
    - Connect output to next node.  

11. **Add Set node ("Associate scores with Responses")**  
    - Assign fields:  
      - `provider`, `response`, `unit_test` from original unit test item  
      - `score` from LMUnit evaluation result  
    - Connect input from "Run LMUnit".  
    - Connect output to "Group Results Together".  

12. **Add Code node ("Group Results Together")**  
    - JavaScript code to group all results by provider and unit test, creating nested JSON structure.  
    - Connect input from "Associate scores with Responses".  
    - Connect output to first output of "Iterate over each unit tests" (completing loop).  

13. **Add Code node ("Format Final Result")**  
    - JavaScript code to format grouped results into readable multi-line string summarizing each provider’s responses and scores.  
    - Connect input from first output of "Iterate over each unit tests".  
    - Connect output to "Final Response".  

14. **Add LangChain Chat node ("Final Response")**  
    - Set message to formatted summary from previous node.  
    - Connect input from "Format Final Result".  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Multi-Model Response Evaluation using Contextual AI’s LMUnit provides systematic LLM output quality evaluation beyond traditional metrics, focusing on clarity and conciseness. The workflow enables consistent, interpretable scoring of multiple LLM outputs for the same input prompt.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Workflow overview sticky note with detailed explanation.                                                                  |
| To set up, create free Contextual AI account and obtain API key: https://app.contextual.ai/. Add API keys for OpenAI, Anthropic, and Gemini API providers as credentials in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Credential setup instructions with official API key links: OpenAI (https://platform.openai.com/account/api-keys), Anthropic (https://console.anthropic.com/settings/keys), Gemini (https://ai.google.dev/gemini-api/docs/api-key) |
| To customize, add more unit test criteria in the code node or include additional LLM providers by duplicating response generation and preprocessing nodes. Adjust aggregation and formatting as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Customization suggestions and reference to LMUnit API docs: https://docs.contextual.ai/api-reference/lmunit/lmunit         |
| For support or feedback, contact feedback@contextual.ai                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Contact email for project support                                                                                         |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.