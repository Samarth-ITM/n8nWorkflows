Track and Monitor AI Token Usage Metrics for OpenAI and Gemini Models

https://n8nworkflows.xyz/workflows/track-and-monitor-ai-token-usage-metrics-for-openai-and-gemini-models-7265


# Track and Monitor AI Token Usage Metrics for OpenAI and Gemini Models

### 1. Workflow Overview

This workflow is designed to **track and monitor AI token usage metrics** specifically for OpenAI and Google Gemini (PaLM) language models within n8n. It targets use cases where detailed insights into token consumption across multiple AI nodes are required, such as cost management, performance monitoring, or audit logging.

The workflow comprises the following logical blocks:

- **1.1 Input Reception and Triggering**: Captures chat messages to initiate AI processing.
- **1.2 AI Processing Block**: Routes input through both Gemini and OpenAI language models via a multi-agent setup, including memory handling.
- **1.3 Execution Data Retrieval and Mapping**: Retrieves execution data by execution ID, parses AI nodes’ token usage, aggregates totals, and extracts detailed usage metrics.
- **1.4 Sub-Workflow Invocation for Metrics Extraction**: Calls a dedicated sub-workflow that processes execution data to produce token usage summaries.
- **1.5 Documentation and Operational Notes**: Provides user instructions and operational constraints in sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview**:  
  This block listens for incoming chat messages to start the workflow and accepts external input parameters (`execution_id` and `model_names` array) for token usage processing.

- **Nodes Involved**:  
  - When chat message received  
  - Start

- **Node Details**:

  - **When chat message received**  
    - *Type*: Langchain Chat Trigger  
    - *Role*: Entry point for chat-based AI interactions. Triggers the workflow on receiving a chat message.  
    - *Configuration*: Uses a webhook ID to expose a webhook endpoint; no additional options enabled.  
    - *Connections*: Outputs to `AI Agent` node.  
    - *Potential Failures*: Network issues, webhook misconfiguration, malformed chat input.

  - **Start**  
    - *Type*: Execute Workflow Trigger  
    - *Role*: Allows the workflow to be executed programmatically with inputs.  
    - *Configuration*: Defines two input parameters: `excecution_id` (string) and `model_names` (array).  
    - *Connections*: Outputs to `Get an execution` node.  
    - *Potential Failures*: Missing or invalid input parameters.

---

#### 2.2 AI Processing Block

- **Overview**:  
  Processes the chat input through two AI language models (Google Gemini and OpenAI), managing context with a simple memory buffer, and orchestrates the AI agent logic.

- **Nodes Involved**:  
  - gemini  
  - openai  
  - Simple Memory  
  - AI Agent

- **Node Details**:

  - **gemini**  
    - *Type*: Langchain Google Gemini (PaLM) Chat Node  
    - *Role*: Provides AI responses using Google Gemini model.  
    - *Configuration*: Uses Google Palm API credentials; no additional options configured.  
    - *Connections*: Outputs to `AI Agent` as one of the language models.  
    - *Potential Failures*: API authentication errors, quota exceeded, network timeouts.

  - **openai**  
    - *Type*: Langchain OpenAI Chat Node  
    - *Role*: Provides AI responses using OpenAI GPT-4.1-mini model.  
    - *Configuration*: Model set to `gpt-4.1-mini`; uses OpenAI API credentials.  
    - *Connections*: Outputs to `AI Agent` as a second language model.  
    - *Potential Failures*: API key expiry, rate limits, network issues.

  - **Simple Memory**  
    - *Type*: Langchain Memory Buffer Window  
    - *Role*: Maintains conversational context for the AI agent.  
    - *Configuration*: Default windowed memory buffer without special parameters.  
    - *Connections*: Provides memory input to `AI Agent`.  
    - *Potential Failures*: Memory overflow or improper context management can affect AI responses.

  - **AI Agent**  
    - *Type*: Langchain Agent Node  
    - *Role*: Orchestrates AI model responses and fallback logic.  
    - *Configuration*: Fallback enabled; integrates two language models (gemini and openai) and memory.  
    - *Connections*: Outputs to `Execute Workflow` node.  
    - *Potential Failures*: Misconfiguration of fallback, AI model failures, runtime errors.

---

#### 2.3 Execution Data Retrieval and Mapping

- **Overview**:  
  Retrieves detailed execution data by execution ID, then parses the JSON execution data to aggregate token usage metrics for specified AI model nodes.

- **Nodes Involved**:  
  - Get an execution  
  - map json

- **Node Details**:

  - **Get an execution**  
    - *Type*: n8n API Node (Execution Resource)  
    - *Role*: Fetches workflow execution data using the provided execution ID.  
    - *Configuration*: Fetches active workflow execution by dynamic `executionId` derived from `Start` node input. Uses n8n API credentials.  
    - *Connections*: Outputs execution data to `map json` node.  
    - *Potential Failures*: Invalid execution ID, API errors, permission issues.

  - **map json**  
    - *Type*: Code (JavaScript) Node  
    - *Role*: Parses the execution data JSON to recursively locate token usage metrics (`completionTokens`, `promptTokens`, `totalTokens`) for the specified model nodes (`model_names`). Aggregates totals and detailed usage by model.  
    - *Configuration*: Custom JS code that:
      - Iterates over the execution data nodes.
      - Uses recursive search to find token usage objects.
      - Accumulates metrics and records models found.
      - Returns an object with:
        - `totals` (aggregated token counts),
        - `models` (unique models encountered),
        - `detailedUsages` (array of usage per model).
    - *Inputs*: Execution data JSON and model names array from `Start`.  
    - *Outputs*: Token usage summary object.  
    - *Potential Failures*: Unexpected JSON structure, missing fields, runtime exceptions in JS code.

---

#### 2.4 Sub-Workflow Invocation for Metrics Extraction

- **Overview**:  
  Invokes a sub-workflow asynchronously to process AI agent execution results and extract token usage metrics for the selected models.

- **Nodes Involved**:  
  - Execute Workflow

- **Node Details**:

  - **Execute Workflow**  
    - *Type*: Execute Workflow Node  
    - *Role*: Calls a separate workflow (ID: `6NVerEoYAOMrSF7k`) that further processes token usage metrics.  
    - *Configuration*:
      - Does not wait for sub-workflow completion (`waitForSubWorkflow = false`).
      - Passes input parameters:
        - `excecution_id`: current execution id.
        - `model_names`: array with values `["gemini", "openai"]`.
    - *Connections*: Triggered by `AI Agent` output.  
    - *Potential Failures*: Sub-workflow ID invalid or inaccessible, parameter mismatches, runtime errors in sub-workflow.

---

#### 2.5 Documentation and Operational Notes

- **Overview**:  
  Provides critical operational instructions and warnings to users, emphasizing execution order, parameter requirements, and usage intent.

- **Nodes Involved**:  
  - Sticky Note

- **Node Details**:

  - **Sticky Note**  
    - *Type*: Sticky Note  
    - *Role*: Displays bilingual notes (English and Spanish) for users.  
    - *Content Highlights*:
      - Must run this workflow **at the end** of the main workflow.
      - Disable “Wait until finished” option on sub-workflow execution.
      - Requires inputs: `execution_id` and `model_names` (array of AI node names).
      - Purpose: Extract token usage metrics and list models used.  
    - *Connections*: None; informational only.

---

### 3. Summary Table

| Node Name              | Node Type                                 | Functional Role                               | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                  |
|------------------------|-------------------------------------------|-----------------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger                    | Entry trigger on chat message                   |                            | AI Agent                   |                                                                                                              |
| Start                  | Execute Workflow Trigger                   | Entry trigger for parameterized workflow start |                            | Get an execution           |                                                                                                              |
| Get an execution       | n8n API Node (Execution Resource)          | Retrieves workflow execution data by ID        | Start                      | map json                   |                                                                                                              |
| map json              | Code (JavaScript) Node                      | Parses execution data, extracts token usage    | Get an execution           |                            |                                                                                                              |
| gemini                | Langchain Google Gemini Chat Node           | AI language model (Google Gemini)               |                            | AI Agent                   |                                                                                                              |
| openai                | Langchain OpenAI Chat Node                   | AI language model (OpenAI GPT-4.1-mini)         |                            | AI Agent                   |                                                                                                              |
| Simple Memory         | Langchain Memory Buffer Window               | Maintains conversational context                |                            | AI Agent                   |                                                                                                              |
| AI Agent              | Langchain Agent Node                         | Orchestrates AI language models and fallback   | gemini, openai, Simple Memory, When chat message received | Execute Workflow           |                                                                                                              |
| Execute Workflow      | Execute Workflow Node                        | Invokes sub-workflow to process token metrics  | AI Agent                   |                            |                                                                                                              |
| Sticky Note           | Sticky Note                                 | Provides operational notes and instructions     |                            |                            | ⚠️ Always run at the end of the workflow; ⚙️ Disable “Wait until finished”; 🆔 Requires `execution_id` and `model_names`; 📊 Extracts token metrics |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**:
   - Add **When chat message received** node:
     - Type: `@n8n/n8n-nodes-langchain.chatTrigger`
     - Configure webhook with a unique ID.
     - No special options enabled.
   - Add **Start** node:
     - Type: `n8n-nodes-base.executeWorkflowTrigger`
     - Define two workflowInputs:
       - `excecution_id` (string)
       - `model_names` (array)

2. **Add AI Language Model Nodes**:
   - Add **gemini** node:
     - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`
     - Set credentials using Google Palm API account.
     - Use default settings.
   - Add **openai** node:
     - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
     - Set model to `gpt-4.1-mini`.
     - Set credentials with OpenAI API account.
     - Use default options.

3. **Add Memory Node**:
   - Add **Simple Memory** node:
     - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
     - Use default parameters.

4. **Add AI Agent Node**:
   - Add **AI Agent** node:
     - Type: `@n8n/n8n-nodes-langchain.agent`
     - Configure to use two language models: `gemini` and `openai`.
     - Configure to use `Simple Memory` as memory.
     - Enable fallback option.
     - Connect inputs:
       - From `When chat message received` (main input)
       - From `gemini` (ai_languageModel input)
       - From `openai` (ai_languageModel input)
       - From `Simple Memory` (ai_memory input)

5. **Add Execution Data Retrieval Nodes**:
   - Add **Get an execution** node:
     - Type: `n8n-nodes-base.n8n` (API node)
     - Resource: Execution
     - Operation: Get
     - Execution ID: Set via expression to `{{$node["Start"].item.json.excecution_id}}`
     - Credentials: Use `n8n API` credentials.
   - Add **map json** node:
     - Type: Code (JavaScript)
     - Paste the provided JS code that recursively extracts token usage from execution JSON for specified model nodes.
     - Connect `Get an execution` output to this node’s input.

6. **Add Sub-Workflow Execution Node**:
   - Add **Execute Workflow** node:
     - Type: `n8n-nodes-base.executeWorkflow`
     - Workflow ID: Set to the sub-workflow that processes token usage metrics (ID: `6NVerEoYAOMrSF7k` or appropriate).
     - Disable "Wait until finished" option.
     - Inputs:
       - `excecution_id` set to current execution ID (`{{$execution.id}}`).
       - `model_names` set to `["gemini", "openai"]`.
     - Connect output of `AI Agent` node to this node.

7. **Add Sticky Note**:
   - Add **Sticky Note** node:
     - Type: `n8n-nodes-base.stickyNote`
     - Add bilingual notes describing:
       - Run this workflow at the end.
       - Disable wait option.
       - Required inputs.
       - Purpose of extracting metrics.
     - Position for user visibility; no connections.

8. **Connect nodes as follows**:
   - `When chat message received` → `AI Agent`
   - `Start` → `Get an execution` → `map json`
   - `gemini` → `AI Agent` (ai_languageModel 0)
   - `openai` → `AI Agent` (ai_languageModel 1)
   - `Simple Memory` → `AI Agent` (ai_memory)
   - `AI Agent` → `Execute Workflow`

9. **Credentials Setup**:
   - Configure Google Palm API credentials for `gemini` node.
   - Configure OpenAI API credentials for `openai` node.
   - Configure n8n API credentials for `Get an execution` node.

10. **Testing and Validation**:
    - Trigger the workflow with a chat message or via the `Start` node inputs.
    - Ensure that an `execution_id` and relevant `model_names` are passed.
    - Confirm that token usage metrics are extracted and the sub-workflow is triggered without waiting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                            | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| ⚠️ This workflow must be executed at the end of the main AI processing workflows to ensure complete execution data is available.                                                                        | Sticky Note content                                                                                       |
| ⚙️ Ensure the “Wait until finished” option is disabled on the sub-workflow execution node to avoid blocking.                                                                                             | Sticky Note content                                                                                       |
| 🆔 The workflow depends on `execution_id` and `model_names` input parameters to identify which execution and AI nodes to analyze.                                                                         | Sticky Note content                                                                                       |
| 📊 The recursive token usage extraction handles nested JSON structures and aggregates usage metrics for multiple AI nodes, supporting scalability.                                                       | Code node description                                                                                     |
| Google Gemini (PaLM) API credentials must be correctly configured in n8n for the Gemini language model node.                                                                                            | Gemini node                                                                                              |
| OpenAI API credentials must be valid and have sufficient quota for the OpenAI language model node.                                                                                                       | OpenAI node                                                                                              |
| Sub-workflow ID `6NVerEoYAOMrSF7k` must be accessible and properly configured to handle inputs and outputs for token usage processing.                                                                   | Execute Workflow node                                                                                     |
| For additional information on n8n execution API: https://docs.n8n.io/integrations/builtin/core-nodes/n8n/                                                                                               | Official n8n documentation                                                                                |
| For Langchain nodes and AI agent details, visit: https://docs.n8n.io/integrations/builtin/nodes/langchain/                                                                                              | Official n8n documentation                                                                                |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.