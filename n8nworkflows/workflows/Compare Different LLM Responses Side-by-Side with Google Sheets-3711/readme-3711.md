Compare Different LLM Responses Side-by-Side with Google Sheets

https://n8nworkflows.xyz/workflows/compare-different-llm-responses-side-by-side-with-google-sheets-3711


# Compare Different LLM Responses Side-by-Side with Google Sheets

### 1. Workflow Overview

This workflow enables side-by-side comparison of outputs from two different large language models (LLMs) using a chat interface and Google Sheets for logging and evaluation. It is designed for AI developers and teams who want to assess which LLM performs better for their specific use case by comparing responses to the same input prompt.

**Logical Blocks:**

- **1.1 Input Reception and Model Definition**  
  Receives user chat input, defines the list of LLM models to compare.

- **1.2 Model Iteration and Context Setup**  
  Splits the model list, prepares session-specific variables for each model, and manages isolated memory contexts.

- **1.3 AI Processing per Model**  
  Sends the input to each model independently via an AI Agent node, which dynamically selects the model and uses isolated memory.

- **1.4 Data Preparation and Aggregation**  
  Formats each model’s output for chat display, collects context and inputs, and aggregates results for evaluation.

- **1.5 Output Concatenation and Logging**  
  Concatenates model outputs for chat UI display and logs all relevant data into a Google Sheet for manual or automated evaluation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Model Definition

- **Overview:**  
  This block triggers on receiving a chat message and defines the array of models to compare.

- **Nodes Involved:**  
  - When chat message received  
  - Define Models to Compare  
  - Split Models into Items  
  - Sticky Note (explanatory)

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point webhook node that listens for incoming chat messages.  
    - Configuration: Default options, no specific filters.  
    - Inputs: External chat message trigger.  
    - Outputs: Passes chat input and session ID downstream.  
    - Edge cases: Missing or malformed chat input may cause failures.

  - **Define Models to Compare**  
    - Type: Set  
    - Role: Defines an array variable `models` containing two model IDs to compare.  
    - Configuration: Sets `models` to `["openai/gpt-4.1", "mistralai/mistral-large"]` by default.  
    - Inputs: From chat trigger node.  
    - Outputs: Passes the models array downstream.  
    - Edge cases: If model IDs are invalid or unsupported by the API, downstream calls will fail.

  - **Split Models into Items**  
    - Type: Split Out  
    - Role: Splits the `models` array into individual items for iteration.  
    - Configuration: Splits on field `models`.  
    - Inputs: From Define Models to Compare.  
    - Outputs: Each model ID as a separate item for processing.  
    - Edge cases: Empty model array will result in no iterations.

  - **Sticky Note (Define Models to Compare)**  
    - Provides detailed instructions on modifying the model list and notes that the template supports two models by default.

#### 2.2 Model Iteration and Context Setup

- **Overview:**  
  Prepares per-model variables including session keys and input duplication, enabling isolated memory per model.

- **Nodes Involved:**  
  - Set model, sessionId, chatInput, sessionIdBase  
  - Loop Over Items (Split in Batches)  
  - Simple Memory  
  - Chat Memory Manager  
  - Sticky Notes (Set model variables, Chat Memory Manager explanations)

- **Node Details:**

  - **Set model, sessionId, chatInput, sessionIdBase**  
    - Type: Set  
    - Role: Assigns variables for each model iteration:  
      - `model`: current model ID  
      - `sessionId`: unique session key combining original session ID and model ID (for memory isolation)  
      - `chatInput`: user input message  
      - `sessionIdBase`: original session ID without suffix (for grouping in Sheets)  
    - Configuration: Uses expressions referencing upstream nodes for session and input.  
    - Inputs: From Split Models into Items.  
    - Outputs: Passes enriched item downstream.  
    - Edge cases: Missing session ID or chat input will cause errors in memory or logging.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Iterates over each model item, allowing parallel or sequential processing.  
    - Configuration: Default batch size (process all items).  
    - Inputs: From Set model, sessionId, chatInput, sessionIdBase.  
    - Outputs: Two branches: one to AI Agent for processing, one to aggregation nodes.  
    - Edge cases: Batch reset misconfiguration could cause repeated processing.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Stores chat history per model session key to maintain context.  
    - Configuration: Uses `sessionId` from Set node as key.  
    - Inputs: From Loop Over Items (via AI Agent).  
    - Outputs: Provides memory context to Chat Memory Manager and AI Agent.  
    - Edge cases: Memory backend failures or session key collisions.

  - **Chat Memory Manager**  
    - Type: LangChain Memory Manager  
    - Role: Retrieves and manages prior conversation context for qualitative evaluation and logging.  
    - Configuration: Default options, shares memory with Simple Memory.  
    - Inputs: From AI Agent output.  
    - Outputs: Passes context-enriched data downstream.  
    - Edge cases: Memory retrieval failures, context truncation issues.

  - **Sticky Notes**  
    - Explain the purpose of memory isolation per model and the use of session keys.

#### 2.3 AI Processing per Model

- **Overview:**  
  Sends the user input to each LLM independently, using the AI Agent node configured dynamically per model.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  
  - Sticky Note (AI Agent explanation)

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Executes the prompt against the selected LLM model with isolated memory.  
    - Configuration:  
      - Model selected dynamically via `{{$json.model}}` variable.  
      - No system prompt or tools configured by default (user must customize).  
      - Returns final output only (no intermediate steps).  
    - Inputs: From Loop Over Items (with memory context).  
    - Outputs: Model response text.  
    - Edge cases: API authentication errors, model unavailability, rate limits, missing system prompt may reduce output quality.

  - **OpenRouter Chat Model**  
    - Type: LangChain OpenRouter LLM Chat  
    - Role: Provides the actual LLM interface to OpenRouter API.  
    - Configuration: Model ID set dynamically from input JSON.  
    - Credentials: Requires OpenRouter API key.  
    - Inputs: From AI Agent node as language model.  
    - Outputs: Model-generated chat completions.  
    - Edge cases: API key invalid, network timeouts, model not found.

  - **Sticky Note (AI Agent)**  
    - Notes that the AI Agent uses OpenRouter models dynamically and currently lacks system prompts or tools.

#### 2.4 Data Preparation and Aggregation

- **Overview:**  
  Formats each model’s output for chat display, prepares data fields for Google Sheets, and aggregates model outputs for evaluation.

- **Nodes Involved:**  
  - Prepare Data for Chat and Google Sheets  
  - Group Model Outputs for Evaluation  
  - Concatenate Chat Answers  
  - Set Output for Chat UI  
  - Sticky Notes (Data preparation, Google Sheets explanation)

- **Node Details:**

  - **Prepare Data for Chat and Google Sheets**  
    - Type: Set  
    - Role: Creates fields for:  
      - `output`: formatted chat output with model name and separator for UI  
      - `chatInput`: original user input  
      - `model_answer`: raw model response  
      - `model`: model ID  
      - `context`: prior conversation history (excluding latest input), or placeholder if none  
      - `sessionId` and `sessionIdBase`: session keys for grouping  
    - Configuration: Uses JavaScript expression to format context and output.  
    - Inputs: From Chat Memory Manager.  
    - Outputs: Passes enriched data downstream.  
    - Edge cases: Missing or malformed message history may cause context formatting errors.

  - **Group Model Outputs for Evaluation**  
    - Type: Aggregate  
    - Role: Aggregates fields (`model_answer`, `context`, `chatInput`, `sessionIdBase`, `model`) from all model iterations into arrays for combined processing.  
    - Inputs: From Loop Over Items (after Prepare Data).  
    - Outputs: Aggregated data for Google Sheets and UI.  
    - Edge cases: Mismatched array lengths if some models fail.

  - **Concatenate Chat Answers**  
    - Type: Summarize (concatenate)  
    - Role: Concatenates the `output` fields from each model with line breaks for side-by-side chat display.  
    - Inputs: From Loop Over Items.  
    - Outputs: Single concatenated string.  
    - Edge cases: Very long outputs may affect UI readability.

  - **Set Output for Chat UI**  
    - Type: Set  
    - Role: Sets the final `output` field with concatenated model responses for chat interface display.  
    - Inputs: From Concatenate Chat Answers.  
    - Outputs: Final chat message output.  
    - Edge cases: None significant.

  - **Sticky Notes**  
    - Explain the data fields prepared for Sheets and chat, and the Google Sheets logging setup including evaluation dropdowns.

#### 2.5 Output Concatenation and Logging

- **Overview:**  
  Logs the user input, model responses, and context into a Google Sheet for manual or automated evaluation.

- **Nodes Involved:**  
  - Add Model Results to Google Sheet  
  - Sticky Note (Google Sheets logging explanation)

- **Node Details:**

  - **Add Model Results to Google Sheet**  
    - Type: Google Sheets (Append)  
    - Role: Appends a new row to a specified Google Sheet with:  
      - `sessionId` (base)  
      - `model_1_id`, `model_2_id`  
      - `user_input`  
      - `model_1_answer`, `model_2_answer`  
      - `context_model_1`, `context_model_2`  
      - Evaluation fields (`model_1_eval`, `model_2_eval`) for manual rating  
    - Configuration:  
      - Uses service account authentication.  
      - Targets a specific Google Sheet template (linked in notes).  
      - Mapping is explicitly defined for each column.  
      - On error: continues workflow without stopping.  
    - Inputs: From Group Model Outputs for Evaluation.  
    - Outputs: None (terminal node).  
    - Edge cases: Authentication failures, API quota limits, sheet access permissions, row size limits.

  - **Sticky Note (Google Sheets Logging)**  
    - Advises adjusting row height/column width for long responses and customizing evaluation criteria.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                                | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                      |
|----------------------------------|----------------------------------|-----------------------------------------------|----------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------|
| When chat message received        | LangChain Chat Trigger            | Entry point for chat input                     | External webhook                 | Define Models to Compare              |                                                                                                                  |
| Define Models to Compare          | Set                              | Defines array of model IDs to compare          | When chat message received       | Split Models into Items               | Explains model definition and usage; supports two models by default                                              |
| Split Models into Items           | Split Out                       | Splits model array into individual items       | Define Models to Compare          | Set model, sessionId, chatInput...   |                                                                                                                  |
| Set model, sessionId, chatInput, sessionIdBase | Set                  | Prepares per-model variables for iteration     | Split Models into Items           | Loop Over Items                      | Explains variable setup for model, sessionId, input, and base session                                            |
| Loop Over Items                  | Split In Batches                 | Iterates over each model for processing        | Set model, sessionId, chatInput... | AI Agent, Concatenate Chat Answers, Group Model Outputs for Evaluation |                                                                                                                  |
| Simple Memory                   | LangChain Memory Buffer Window   | Stores chat history per model session          | Loop Over Items (via AI Agent)   | Chat Memory Manager, AI Agent        |                                                                                                                  |
| Chat Memory Manager             | LangChain Memory Manager         | Retrieves prior conversation context           | AI Agent                        | Prepare Data for Chat and Google Sheets | Explains memory sharing and context retrieval                                                                    |
| OpenRouter Chat Model           | LangChain OpenRouter LLM Chat    | Connects to OpenRouter API for LLM calls       | AI Agent (as language model)     | AI Agent                           |                                                                                                                  |
| AI Agent                       | LangChain Agent                  | Runs prompt on selected model with memory      | Loop Over Items                  | Chat Memory Manager                 | Notes dynamic model selection and lack of system prompt/tools                                                   |
| Prepare Data for Chat and Google Sheets | Set                      | Formats model output and context for UI and Sheets | Chat Memory Manager             | Loop Over Items                    | Explains data fields prepared for chat display and Google Sheets                                                |
| Group Model Outputs for Evaluation | Aggregate                    | Aggregates model outputs and context arrays    | Loop Over Items                  | Add Model Results to Google Sheet    |                                                                                                                  |
| Concatenate Chat Answers        | Summarize (Concatenate)          | Concatenates model outputs for chat UI display | Loop Over Items                  | Set Output for Chat UI               |                                                                                                                  |
| Set Output for Chat UI           | Set                              | Sets final chat output with concatenated responses | Concatenate Chat Answers         | (Chat UI output)                    |                                                                                                                  |
| Add Model Results to Google Sheet | Google Sheets (Append)          | Logs user input, model responses, and context  | Group Model Outputs for Evaluation | (Terminal)                        | Advises adjusting sheet layout and customizing evaluation fields                                                |
| Sticky Note                     | Sticky Note                     | Explanatory notes for various workflow parts   | -                              | -                                  | Multiple notes explaining model setup, memory, AI Agent, data prep, and Google Sheets logging                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Configure with default options to receive chat messages and session IDs.

2. **Define Models Array:**  
   - Add a **Set** node named `Define Models to Compare`.  
   - Create a variable `models` as an array with two model IDs, e.g., `["openai/gpt-4.1", "mistralai/mistral-large"]`.

3. **Split Models into Items:**  
   - Add a **Split Out** node named `Split Models into Items`.  
   - Configure to split the `models` array into individual items.

4. **Set Per-Model Variables:**  
   - Add a **Set** node named `Set model, sessionId, chatInput, sessionIdBase`.  
   - Assign:  
     - `model` = current model item (`{{$json.models}}`)  
     - `sessionId` = concatenate session ID from trigger + model ID (`{{$('When chat message received').item.json.sessionId}}{{$json.models}}`)  
     - `chatInput` = user input from trigger (`{{$('When chat message received').item.json.chatInput}}`)  
     - `sessionIdBase` = original session ID (`{{$('When chat message received').item.json.sessionId}}`)

5. **Loop Over Models:**  
   - Add a **Split In Batches** node named `Loop Over Items`.  
   - Connect output of Set node to this node to iterate over each model.

6. **Configure Memory:**  
   - Add a **LangChain Memory Buffer Window** node named `Simple Memory`.  
   - Set `sessionKey` to `{{$json.sessionId}}` to isolate memory per model.

7. **Add Chat Memory Manager:**  
   - Add a **LangChain Memory Manager** node named `Chat Memory Manager`.  
   - Connect `Simple Memory` and `AI Agent` nodes to share memory context.

8. **Add OpenRouter Chat Model:**  
   - Add a **LangChain OpenRouter LLM Chat** node named `OpenRouter Chat Model`.  
   - Set model dynamically to `{{$json.model}}`.  
   - Configure with OpenRouter API credentials.

9. **Add AI Agent:**  
   - Add a **LangChain Agent** node named `AI Agent`.  
   - Configure to use `OpenRouter Chat Model` as language model.  
   - Set model dynamically via `{{$json.model}}`.  
   - Disable intermediate steps.  
   - Connect memory nodes for context.  
   - Note: Add system prompt and tools as needed for your use case.

10. **Prepare Data for Chat and Sheets:**  
    - Add a **Set** node named `Prepare Data for Chat and Google Sheets`.  
    - Assign fields:  
      - `output`: formatted string with model name and response, plus separator  
      - `chatInput`: user input  
      - `model_answer`: raw model output  
      - `model`: model ID  
      - `context`: prior conversation history excluding latest input (use JS expression)  
      - `sessionId` and `sessionIdBase`: session keys

11. **Loop Output Branches:**  
    - From `Loop Over Items`, connect one branch to `AI Agent` for processing.  
    - Connect the other branch to aggregation nodes.

12. **Aggregate Model Outputs:**  
    - Add an **Aggregate** node named `Group Model Outputs for Evaluation`.  
    - Aggregate fields: `model_answer`, `context`, `chatInput`, `sessionIdBase`, `model`.

13. **Concatenate Outputs for Chat UI:**  
    - Add a **Summarize** node named `Concatenate Chat Answers`.  
    - Concatenate the `output` fields with newline separators.

14. **Set Final Chat Output:**  
    - Add a **Set** node named `Set Output for Chat UI`.  
    - Assign `output` field to the concatenated string from previous node.

15. **Add Google Sheets Logging:**  
    - Add a **Google Sheets** node named `Add Model Results to Google Sheet`.  
    - Configure to append rows to your copied Google Sheets template.  
    - Map columns:  
      - `sessionId` = `{{$json.sessionIdBase[0]}}`  
      - `model_1_id` = `{{$json.model[0]}}`  
      - `model_2_id` = `{{$json.model[1]}}`  
      - `user_input` = `{{$json.chatInput[0]}}`  
      - `model_1_answer` = `{{$json.model_answer[0]}}`  
      - `model_2_answer` = `{{$json.model_answer[1]}}`  
      - `context_model_1` = `{{$json.context[0]}}`  
      - `context_model_2` = `{{$json.context[1]}}`  
      - Include evaluation fields `model_1_eval` and `model_2_eval` (optional).  
    - Use service account credentials with access to the Google Sheet.

16. **Connect Nodes:**  
    - Connect `When chat message received` → `Define Models to Compare` → `Split Models into Items` → `Set model, sessionId, chatInput, sessionIdBase` → `Loop Over Items`.  
    - From `Loop Over Items`:  
      - Branch 1 → `AI Agent` → `Chat Memory Manager` → `Prepare Data for Chat and Google Sheets` → back to `Loop Over Items` (for aggregation).  
      - Branch 2 → `Concatenate Chat Answers` → `Set Output for Chat UI`.  
      - Also from `Loop Over Items` → `Group Model Outputs for Evaluation` → `Add Model Results to Google Sheet`.

17. **Test and Validate:**  
    - Ensure OpenRouter API credentials are valid.  
    - Copy and configure the Google Sheets template as per instructions.  
    - Start chat messages to trigger the workflow and verify side-by-side outputs and logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Copy this [Google Sheets template](https://docs.google.com/spreadsheets/d/1grO5jxm05kJ7if9wBIOozjkqW27i8tRedrheLRrpxf4/) (File > Make a Copy) to use with this workflow.                                                                                                                                                                                        | Google Sheets template for logging and evaluation                                                       |
| This workflow is designed for two models by default. To compare more, extend the logic and update the Google Sheet accordingly.                                                                                                                                                                                                                              | Usage note                                                                                              |
| You can use OpenRouter or Vertex AI to test models across providers. For provider-specific nodes (e.g., OpenAI), compare different models from the same provider.                                                                                                                                                                                               | Model provider flexibility                                                                               |
| Advanced users can automate evaluation in Google Sheets using a more capable model (like OpenAI’s `o3`), but this increases token usage and cost.                                                                                                                                                                                                             | Evaluation automation note                                                                               |
| Each input is processed by two models, so token consumption and costs will be roughly doubled. Monitor usage carefully, especially with long prompts or frequent evaluations.                                                                                                                                                                                  | Token usage and cost consideration                                                                       |
| The AI Agent node currently has no system prompt or tools configured by default. Customize these to reflect your use case for better results.                                                                                                                                                                                                                 | AI Agent customization advice                                                                            |
| Memory is isolated per model using a unique session key combining session ID and model name, ensuring independent context windows.                                                                                                                                                                                                                            | Memory management explanation                                                                            |
| The Google Sheets node uses a service account for authentication; ensure the service account has edit access to the target spreadsheet.                                                                                                                                                                                                                        | Google Sheets credential setup                                                                           |
| Adjust Google Sheets row height and column width to accommodate long model responses for better readability.                                                                                                                                                                                                                                                  | Google Sheets UI tip                                                                                      |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the workflow to compare LLM outputs side-by-side with Google Sheets logging.