Use OpenRouter in n8n versions <1.78

https://n8nworkflows.xyz/workflows/use-openrouter-in-n8n-versions--1-78-2897


# Use OpenRouter in n8n versions <1.78

### 1. Workflow Overview

This workflow demonstrates how to use the OpenRouter service with n8n versions prior to 1.78, which do not have a dedicated OpenRouter node. It leverages the existing OpenAI node by configuring it to use OpenRouter’s API endpoint, enabling dynamic selection and switching of large language models (LLMs) from OpenRouter’s catalog.

The workflow is designed for chat-based interactions, receiving chat messages via a LangChain chat trigger and processing them through an AI agent that uses OpenRouter models dynamically configured at runtime.

**Logical Blocks:**

- **1.1 Input Reception:** Receives chat messages and session information.
- **1.2 Settings Configuration:** Defines the model, prompt, and session ID variables dynamically.
- **1.3 AI Processing:** Uses LangChain nodes to manage chat memory, select the LLM model dynamically, and run the AI agent with OpenRouter.
- **1.4 Documentation & Guidance:** Sticky notes provide user instructions and model examples.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages and triggers the workflow execution. It captures the chat input and session ID for downstream processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger` (Webhook trigger for chat messages)  
    - Configuration: Default options; listens for chat messages via webhook with ID `71f56e44-401f-44ba-b54d-c947e283d034`  
    - Key expressions: None (direct webhook input)  
    - Input: External chat message via webhook  
    - Output: JSON containing `chatInput` and `sessionId` fields  
    - Version: 1.1  
    - Potential failures: Webhook connectivity issues, malformed input, missing required fields  
    - Sub-workflow: None  

#### 1.2 Settings Configuration

- **Overview:**  
  Sets up key variables such as the LLM model to use, the prompt text extracted from the chat input, and the session ID for memory management. This allows dynamic model selection and prompt passing.

- **Nodes Involved:**  
  - Settings  
  - Sticky Note (Settings)

- **Node Details:**  
  - **Settings**  
    - Type: `n8n-nodes-base.set` (Assigns variables)  
    - Configuration:  
      - `model`: Default set to `"deepseek/deepseek-r1-distill-llama-8b"` (can be changed dynamically)  
      - `prompt`: Set dynamically to the incoming chat message (`={{ $json.chatInput }}`)  
      - `sessionId`: Set dynamically to the incoming session ID (`={{ $json.sessionId }}`)  
    - Input: Output from "When chat message received"  
    - Output: JSON with `model`, `prompt`, and `sessionId` fields for downstream use  
    - Version: 3.4  
    - Potential failures: Expression evaluation errors if input fields missing or malformed  
    - Sub-workflow: None  

  - **Sticky Note (Settings)**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: "## Settings\nSpecify the model"  
    - Purpose: User guidance on configuring the model parameter  

#### 1.3 AI Processing

- **Overview:**  
  This block manages chat memory, dynamically selects the LLM model from OpenRouter, and runs the AI agent to generate responses based on the prompt and session context.

- **Nodes Involved:**  
  - Chat Memory  
  - LLM Model  
  - AI Agent  
  - Sticky Note1 (Run LLM)  
  - Sticky Note2 (Model examples)

- **Node Details:**  
  - **Chat Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Manages conversational memory)  
    - Configuration: Uses `sessionId` as a custom key to maintain session-specific memory  
    - Input: Output from "Settings" node (sessionId)  
    - Output: Provides memory context to AI Agent  
    - Version: 1.3  
    - Potential failures: Missing or invalid sessionId causing memory mismanagement  

  - **LLM Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI-compatible LLM node)  
    - Configuration:  
      - `model`: Dynamically set from `model` variable (`={{ $json.model }}`)  
      - `options`: Default empty  
    - Credentials: Uses OpenAI credential configured with OpenRouter API base URL (`https://openrouter.ai/api/v1`)  
    - Input: Receives prompt and memory context  
    - Output: LLM response to AI Agent  
    - Version: 1.1  
    - Potential failures: Authentication errors if credentials misconfigured, network timeouts, invalid model IDs, API rate limits  

  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent` (Orchestrates AI processing)  
    - Configuration:  
      - `text`: Uses prompt from `prompt` variable (`={{ $json.prompt }}`)  
      - `promptType`: "define" (custom prompt type)  
      - `options`: Default empty  
    - Input: Receives outputs from "Settings" (prompt), "LLM Model" (language model), and "Chat Memory" (memory) nodes via specialized input connections  
    - Output: Final AI-generated response  
    - Version: 1.7  
    - Potential failures: Expression errors, missing inputs, runtime errors in agent logic  

  - **Sticky Note1 (Run LLM)**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: "## Run LLM\nUsing OpenRouter to make model fully configurable"  
    - Purpose: Explains the dynamic model usage via OpenRouter  

  - **Sticky Note2 (Model examples)**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content:  
      ```
      ## Model examples

      * openai/o3-mini
      * google/gemini-2.0-flash-001
      * deepseek/deepseek-r1-distill-llama-8b
      * mistralai/mistral-small-24b-instruct-2501:free
      * qwen/qwen-turbo

      For more see https://openrouter.ai/models
      ```  
    - Purpose: Provides example model IDs and a link to the OpenRouter model list for user reference  

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                      | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                      |
|-------------------------|----------------------------------------|------------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Input reception of chat messages   | (Webhook external)         | Settings                 |                                                                                                |
| Settings                | n8n-nodes-base.set                     | Assign variables: model, prompt, sessionId | When chat message received | AI Agent                 | ## Settings\nSpecify the model                                                                 |
| Sticky Note             | n8n-nodes-base.stickyNote              | User guidance on settings           |                           |                          | ## Settings\nSpecify the model                                                                 |
| AI Agent                | @n8n/n8n-nodes-langchain.agent        | Orchestrates AI processing          | Settings, LLM Model, Chat Memory |                          | ## Run LLM\nUsing OpenRouter to make model fully configurable                                  |
| Chat Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Manages conversational memory       | Settings                  | AI Agent                 | ## Run LLM\nUsing OpenRouter to make model fully configurable                                  |
| LLM Model               | @n8n/n8n-nodes-langchain.lmChatOpenAi | Runs OpenRouter LLM model           | Settings                  | AI Agent                 | ## Run LLM\nUsing OpenRouter to make model fully configurable                                  |
| Sticky Note1            | n8n-nodes-base.stickyNote              | User guidance on AI processing      |                           |                          | ## Run LLM\nUsing OpenRouter to make model fully configurable                                  |
| Sticky Note2            | n8n-nodes-base.stickyNote              | Lists example models and link       |                           |                          | ## Model examples\n\n* openai/o3-mini\n* google/gemini-2.0-flash-001\n* deepseek/deepseek-r1-distill-llama-8b\n* mistralai/mistral-small-24b-instruct-2501:free\n* qwen/qwen-turbo\n\nFor more see https://openrouter.ai/models |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Add a `LangChain Chat Trigger` node named "When chat message received".  
   - Configure it to listen on a webhook (auto-generated webhook ID).  
   - This node will receive incoming chat messages with fields like `chatInput` and `sessionId`.

2. **Create the Settings Node:**  
   - Add a `Set` node named "Settings".  
   - Connect the output of "When chat message received" to this node.  
   - Configure three variables:  
     - `model` (string): default value `"deepseek/deepseek-r1-distill-llama-8b"` (can be changed dynamically)  
     - `prompt` (string): expression `={{ $json.chatInput }}` to capture incoming chat text  
     - `sessionId` (string): expression `={{ $json.sessionId }}` to capture session identifier  

3. **Create the Chat Memory Node:**  
   - Add a `LangChain Memory Buffer Window` node named "Chat Memory".  
   - Configure `sessionKey` to `={{ $json.sessionId }}` and set `sessionIdType` to `customKey`.  
   - Connect the output of "Settings" to this node.

4. **Create the LLM Model Node:**  
   - Add a `LangChain OpenAI Chat` node named "LLM Model".  
   - Set the `model` parameter to `={{ $json.model }}` to dynamically select the model.  
   - Leave options empty or default.  
   - Assign credentials: create or select an OpenAI credential configured with:  
     - API Key from OpenRouter.ai  
     - Base URL set to `https://openrouter.ai/api/v1` (this is critical to route requests to OpenRouter)  
   - Connect the output of "Settings" to this node.

5. **Create the AI Agent Node:**  
   - Add a `LangChain Agent` node named "AI Agent".  
   - Set `text` parameter to `={{ $json.prompt }}` to use the prompt from "Settings".  
   - Set `promptType` to `"define"`.  
   - Connect three inputs:  
     - Main input from "Settings" (for prompt text)  
     - `ai_languageModel` input from "LLM Model" node  
     - `ai_memory` input from "Chat Memory" node  

6. **Connect the Workflow:**  
   - Connect "When chat message received" → "Settings" → "AI Agent" (main input).  
   - Connect "Settings" → "Chat Memory" → "AI Agent" (memory input).  
   - Connect "Settings" → "LLM Model" → "AI Agent" (language model input).

7. **Add Sticky Notes for Documentation (Optional):**  
   - Add sticky notes near relevant nodes with the following contents:  
     - Near "Settings": "## Settings\nSpecify the model"  
     - Near AI processing nodes: "## Run LLM\nUsing OpenRouter to make model fully configurable"  
     - Near model selection:  
       ```
       ## Model examples

       * openai/o3-mini
       * google/gemini-2.0-flash-001
       * deepseek/deepseek-r1-distill-llama-8b
       * mistralai/mistral-small-24b-instruct-2501:free
       * qwen/qwen-turbo

       For more see https://openrouter.ai/models
       ```

8. **Credential Setup:**  
   - Create an OpenAI credential in n8n named "OpenRouter".  
   - Use your OpenRouter API key.  
   - Set the Base URL to `https://openrouter.ai/api/v1` to redirect OpenAI node calls to OpenRouter.  
   - Assign this credential to the "LLM Model" node.

9. **Activate and Test:**  
   - Save and activate the workflow.  
   - Send chat messages to the webhook URL generated by the "When chat message received" node.  
   - Observe responses generated by the AI Agent using the dynamically selected OpenRouter model.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                          |
|-----------------------------------------------------------------------------------------------------|----------------------------------------|
| In n8n version 1.78+, a dedicated OpenRouter node is available, simplifying integration.             | Workflow description                    |
| OpenRouter supports many LLM models and providers, allowing dynamic model switching in workflows.   | https://openrouter.ai/models            |
| To use OpenRouter in n8n <1.78, configure OpenAI credentials with Base URL `https://openrouter.ai/api/v1` | Setup instructions                     |
| Example models include `openai/o3-mini`, `google/gemini-2.0-flash-001`, `deepseek/deepseek-r1-distill-llama-8b`, `mistralai/mistral-small-24b-instruct-2501:free`, `qwen/qwen-turbo` | Sticky Note content                     |
| Ensure you have an OpenRouter account with API key and credits before using this workflow.           | Setup instructions                      |

---

This documentation provides a complete understanding of the workflow structure, node configurations, and setup instructions to enable dynamic use of OpenRouter LLM models in n8n versions before 1.78.