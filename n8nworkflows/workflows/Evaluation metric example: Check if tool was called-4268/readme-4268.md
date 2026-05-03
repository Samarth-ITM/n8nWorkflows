Evaluation metric example: Check if tool was called

https://n8nworkflows.xyz/workflows/evaluation-metric-example--check-if-tool-was-called-4268


# Evaluation metric example: Check if tool was called

### 1. Workflow Overview

This workflow demonstrates how to evaluate whether a specific tool was called by an AI agent during a chat interaction. It is designed primarily to assess the agent’s behavior against a test dataset of questions and expected tool usage. The main use case is for quality assurance and metric-based evaluation of AI workflows that incorporate tool execution.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Formatting:** Receives chat messages or dataset rows and prepares the input for the AI agent.
- **1.2 AI Agent Processing:** Runs the AI Agent using an OpenAI GPT-4o-mini model and configured tools (Calculator and Webpage Fetcher).
- **1.3 Evaluation Metric Calculation:** Checks if the expected tool was called by inspecting the agent’s intermediate steps.
- **1.4 Metrics Reporting:** Sends the evaluation result to an evaluation node for recording.
- **1.5 Control Flow and Output:** Handles the decision of whether to evaluate or return a chat response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Formatting

- **Overview:**  
  This block receives inputs either from a chat trigger or a dataset row (from a Google Sheet), then normalizes the input format to a consistent string property for the AI agent.

- **Nodes Involved:**  
  - When chat message received  
  - When fetching a dataset row  
  - Match chat format

- **Node Details:**  

  - **When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Entry point for live chat messages to initiate AI processing  
    - Configuration: Webhook ID set, no special options  
    - Inputs: External chat messages  
    - Outputs: Chat message data forwarded to Match chat format  
    - Edge cases: Webhook misconfiguration, missing chat payloads, or network issues

  - **When fetching a dataset row**  
    - Type: Evaluation Trigger (Google Sheets integration)  
    - Role: Triggers workflow based on rows from a Google Sheets test dataset containing questions and expected tool calls  
    - Configuration: Sheet URL and document ID set to a public Google Sheet; uses OAuth2 credentials  
    - Inputs: Dataset rows from the spreadsheet  
    - Outputs: Dataset row forwarded to Match chat format  
    - Edge cases: Google Sheets API rate limiting, OAuth token expiration, invalid sheet URL

  - **Match chat format**  
    - Type: Set Node  
    - Role: Normalizes input by extracting the question text into a single property `chatInput`  
    - Configuration: Sets `chatInput` to the value of `$json.question` (dataset) or `$json.question` from chat input  
    - Inputs: Raw input from either chat or dataset row nodes  
    - Outputs: Passes formatted input to AI Agent  
    - Edge cases: Missing `question` property, malformed JSON input

---

#### 2.2 AI Agent Processing

- **Overview:**  
  This block runs the AI Agent configured with an OpenAI GPT-4o-mini chat model and two tools: a Calculator and a Webpage Fetcher. The agent returns intermediate execution steps to track tool usage.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Calculator  
  - Fetch a webpage  

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Core AI processing node that orchestrates language model and tool execution  
    - Configuration: Option `returnIntermediateSteps` enabled to capture tool calls  
    - Inputs: Receives formatted chat input from Match chat format  
    - Outputs: Sends results downstream to evaluation and response nodes  
    - Edge cases: API key issues, timeouts, malformed inputs, model limitations

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides AI language model responses (GPT-4o-mini) for the Agent  
    - Configuration: Model set to `gpt-4o-mini`, uses OpenAI API credentials  
    - Inputs: Called internally by AI Agent as language model backend  
    - Outputs: Language model completions for AI Agent  
    - Edge cases: API quota exceeded, network errors, invalid credentials

  - **Calculator**  
    - Type: LangChain Calculator Tool  
    - Role: Provides calculation capabilities as a tool for the AI Agent  
    - Configuration: Default (no parameters)  
    - Inputs: Invoked by AI Agent as needed  
    - Outputs: Calculation results returned to AI Agent  
    - Edge cases: Invalid math expressions, zero division

  - **Fetch a webpage**  
    - Type: HTTP Request Tool for LangChain  
    - Role: Tool that fetches webpage content when called by AI Agent  
    - Configuration: URL dynamically set from AI input (overridable via AI)  
    - Inputs: Invoked by AI Agent  
    - Outputs: Webpage content returned to AI Agent  
    - Edge cases: Invalid URL, network timeouts, HTTP errors (404, 500), SSL issues

---

#### 2.3 Evaluation Metric Calculation

- **Overview:**  
  This block inspects the AI Agent’s intermediate steps to determine if the tool expected for the current question was actually called.

- **Nodes Involved:**  
  - Evaluating?  
  - Check if tool called  

- **Node Details:**  

  - **Evaluating?**  
    - Type: Evaluation Node (checkIfEvaluating operation)  
    - Role: Decides whether the workflow is currently running an evaluation iteration  
    - Inputs: AI Agent output  
    - Outputs: Routes either to metric calculation or to chat response  
    - Edge cases: Misrouted data if evaluation state is ambiguous

  - **Check if tool called**  
    - Type: Set Node  
    - Role: Creates a boolean `tool_called` indicating if the expected tool was called  
    - Configuration:  
      - Expression filters `intermediateSteps` for any action where `tool` matches `tool_to_call` from dataset row  
      - Sets `tool_called` to true if at least one matching tool call exists  
    - Inputs: AI Agent output filtered by Evaluating? node  
    - Outputs: Passes metric to Evaluation node  
    - Edge cases: Missing intermediateSteps, undefined tool names, expression evaluation failures

---

#### 2.4 Metrics Reporting

- **Overview:**  
  This block records the evaluation metric indicating tool usage into the evaluation system.

- **Nodes Involved:**  
  - Evaluation  

- **Node Details:**  

  - **Evaluation**  
    - Type: Evaluation Node (setMetrics operation)  
    - Role: Records numeric metrics for evaluation reporting  
    - Configuration: Sets metric `tool_called` to numeric equivalent of boolean from previous node  
    - Inputs: Receives boolean metric from Check if tool called  
    - Outputs: Finalizes evaluation data  
    - Edge cases: Metric misconfiguration, incompatible data types

---

#### 2.5 Control Flow and Output

- **Overview:**  
  This block controls whether to proceed with evaluation or return a chat response directly.

- **Nodes Involved:**  
  - Return chat response

- **Node Details:**  

  - **Return chat response**  
    - Type: No Operation (NoOp) Node  
    - Role: Placeholder for returning chat response when not evaluating  
    - Inputs: From Evaluating? node when no evaluation is performed  
    - Outputs: Ends workflow or sends response downstream  
    - Edge cases: None; acts as a pass-through

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                         | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                              |
|-------------------------|----------------------------------|---------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger           | Entry point for chat messages          |                            | Match chat format            |                                                                                                        |
| When fetching a dataset row | Evaluation Trigger (Google Sheets) | Entry point for dataset test rows      |                            | Match chat format            |                                                                                                        |
| Match chat format         | Set Node                         | Normalize input question format        | When chat message received, When fetching a dataset row | AI Agent                    |                                                                                                        |
| AI Agent                 | LangChain Agent                  | Runs AI with tools and model            | Match chat format           | Evaluating?, Return chat response | "Make sure to enable 'Return intermediate steps' in the agent, to get the list of executed tools"      |
| OpenAI Chat Model         | LangChain Language Model         | Provides GPT-4o-mini completions       | AI Agent (ai_languageModel) | AI Agent                    |                                                                                                        |
| Calculator               | LangChain Tool                   | Calculator tool for AI Agent            | AI Agent (ai_tool)          | AI Agent                    |                                                                                                        |
| Fetch a webpage           | HTTP Request Tool                | Webpage fetching tool for AI Agent     | AI Agent (ai_tool)          | AI Agent                    |                                                                                                        |
| Evaluating?              | Evaluation Node                  | Checks if workflow is in evaluation mode | AI Agent                   | Check if tool called, Return chat response |                                                                                                        |
| Check if tool called      | Set Node                         | Determines if expected tool was called | Evaluating?                 | Evaluation                  | "Check whether the list of executed tools contains the target tool"                                   |
| Evaluation               | Evaluation Node                  | Records evaluation metric               | Check if tool called        |                             |                                                                                                        |
| Return chat response      | No Operation Node                | Returns chat response if not evaluating | Evaluating?                 |                             |                                                                                                        |
| Sticky Note              | Sticky Note                     | Instructional note                      |                            |                             | "Make sure to enable 'Return intermediate steps' in the agent, to get the list of executed tools"      |
| Sticky Note1             | Sticky Note                     | Instructional note                      |                            |                             | "Check whether the list of executed tools contains the target tool"                                   |
| Sticky Note3             | Sticky Note                     | Workflow explanation                    |                            |                             | "## How it works\nThis template shows how to calculate a workflow evaluation metric: **whether a specific tool was called** by an agent.\n\nYou can find more information on workflow evaluation [here](https://docs.n8n.io/advanced-ai/evaluations/overview), and other metric examples [here](https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics)." |
| Sticky Note4             | Sticky Note                     | Dataset link                           |                            |                             | "Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=969651976#gid=969651976) of questions and the tools that should be called when answering them" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**

   - Add **When chat message received** node  
     - Type: LangChain Chat Trigger  
     - Configure webhook ID (auto-generated)  
     - No extra parameters needed

   - Add **When fetching a dataset row** node  
     - Type: Evaluation Trigger (Google Sheets)  
     - Configure with Google Sheets OAuth2 credentials  
     - Set Document ID and Sheet Name to the test dataset URL:  
       `https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=969651976#gid=969651976`

2. **Normalize Input Format**

   - Add **Match chat format** node  
     - Type: Set Node  
     - Create a string field `chatInput`  
     - Set value with expression: `={{ $json.question }}`  
     - Connect output of both triggers to this node

3. **Configure AI Agent**

   - Add **OpenAI Chat Model** node  
     - Type: LangChain OpenAI Chat Model  
     - Select model: `gpt-4o-mini`  
     - Configure OpenAI API credentials (OpenAI API key)  

   - Add **Calculator** node  
     - Type: LangChain Calculator Tool  
     - Default settings

   - Add **Fetch a webpage** node  
     - Type: HTTP Request Tool  
     - URL parameter: set with expression allowing override by AI input, e.g. `={{ $fromAI('URL', '', 'string') }}`  
     - Default HTTP options

   - Add **AI Agent** node  
     - Type: LangChain Agent  
     - Enable option: `Return intermediate steps` to `true`  
     - Connect `chatInput` from Match chat format into AI Agent main input  
     - Connect OpenAI Chat Model node as `ai_languageModel` input  
     - Connect Calculator and Fetch a webpage nodes as `ai_tool` inputs

4. **Set up Evaluation Control**

   - Add **Evaluating?** node  
     - Type: Evaluation Node, operation `checkIfEvaluating`  
     - Connect AI Agent main output to this node

5. **Check Tool Called**

   - Add **Check if tool called** node  
     - Type: Set Node  
     - Add boolean field `tool_called`  
     - Set value with expression:  
       ```
       ={{ $json.intermediateSteps.filter(x => x.action.tool == $('When fetching a dataset row').item.json.tool_to_call).length > 0 }}
       ```  
     - Connect `Evaluating?` node first output (true) to this node

6. **Record Evaluation Metric**

   - Add **Evaluation** node  
     - Type: Evaluation Node, operation `setMetrics`  
     - Add metric assignment:  
       - Name: `tool_called`  
       - Type: number  
       - Value: `={{ $json.tool_called.toNumber() }}`  
     - Connect output of Check if tool called node to Evaluation node

7. **Handle Non-Evaluation Response**

   - Add **Return chat response** node  
     - Type: No Operation (NoOp)  
     - Connect second output of Evaluating? node (false) to this node

8. **Finalize Connections**

   - Connect When chat message received → Match chat format  
   - Connect When fetching a dataset row → Match chat format  
   - Connect Match chat format → AI Agent  
   - Connect AI Agent → Evaluating?  
   - Connect Evaluating? → Check if tool called (main 0) and Return chat response (main 1)  
   - Connect Check if tool called → Evaluation

9. **Add Sticky Notes and Documentation**

   - Add sticky notes with the following content near relevant nodes:  
     - Near AI Agent: "Make sure to enable 'Return intermediate steps' in the agent, to get the list of executed tools"  
     - Near Check if tool called: "Check whether the list of executed tools contains the target tool"  
     - Near workflow start:  
       ```
       ## How it works  
       This template shows how to calculate a workflow evaluation metric: **whether a specific tool was called** by an agent.
       
       You can find more information on workflow evaluation [here](https://docs.n8n.io/advanced-ai/evaluations/overview), and other metric examples [here](https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics).
       ```  
     - Near dataset trigger:  
       ```
       Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=969651976#gid=969651976) of questions and the tools that should be called when answering them
       ```

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                       | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow relies on enabling the "Return intermediate steps" option in the AI Agent to track which tools were executed during processing.                                                                                                                                                    | Sticky Note near AI Agent node                                                                   |
| The test dataset used for evaluation is publicly accessible and contains a list of questions and the corresponding tools that should be called to answer them, facilitating metric calculation.                                                                                                   | https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit          |
| More information on workflow evaluation concepts and additional metric examples is available in the official n8n documentation, useful for extending or adapting this workflow to other evaluation metrics.                                                                                        | https://docs.n8n.io/advanced-ai/evaluations/overview and https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics |

---

**Disclaimer:**  
The text provided stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.