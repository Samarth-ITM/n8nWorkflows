Build Resilient AI Workflows with Automatic GPT and Gemini Failover Chain

https://n8nworkflows.xyz/workflows/build-resilient-ai-workflows-with-automatic-gpt-and-gemini-failover-chain-5160


# Build Resilient AI Workflows with Automatic GPT and Gemini Failover Chain

---

## 1. Workflow Overview

This workflow implements a **resilient AI agent with automatic failover across multiple language models (LLMs)**, specifically designed to try fallback AI models sequentially if the primary model fails. It targets use cases where robustness and high availability in AI responses are critical, such as customer support, automated content generation, or multi-model experimentation.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Control Loop:** Receives external triggers and manages retry count for failover attempts.
- **1.2 Failover Model Selector:** Dynamically selects which AI model to use based on the current retry count.
- **1.3 AI Agent Execution:** Runs the AI agent with the selected model and prompt, handling errors to trigger failovers.
- **1.4 AI Model Nodes:** The actual OpenAI and Google Gemini model nodes representing available AI providers.
- **1.5 Configuration Notes:** Informative sticky notes guiding setup and usage.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Control Loop

**Overview:**  
This block initializes and increments the failure count (`fail_count`) each time the AI agent encounters an error, effectively controlling which AI model to try next.

**Nodes Involved:**  
- Manual Trigger  
- Agent Variables

**Node Details:**  

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Role:* Entry point to start the workflow manually.  
  - *Configuration:* Default, no parameters.  
  - *Connections:* Outputs to `Agent Variables`.  
  - *Edge cases:* None specific; manual trigger requires human or external initiation.

- **Agent Variables**  
  - *Type:* Set node  
  - *Role:* Maintains and increments the `fail_count` variable, and holds the list of models to try.  
  - *Configuration:*  
    - Defines two variables:  
      - `models`: an array of model identifiers (initialized with `["gemini-2.5-flash"]` in the example, but meant to be customized).  
      - `fail_count`: increments by 1 if the node was previously executed; otherwise, initializes at 0.  
  - *Expressions:*  
    - `fail_count` uses an expression to check if it has been executed before to increment count:  
      `={{ $('Agent Variables')?.isExecuted ? $('Agent Variables').last()?.json?.fail_count + 1 : 0 }}`  
  - *Connections:* Outputs to `AI Agent`.  
  - *Edge cases:*  
    - Expression failures if prior execution data is missing or corrupted.  
    - The `models` array must be populated correctly or fallback logic may fail.

---

### 2.2 Failover Model Selector

**Overview:**  
This block takes the retry count and the connected AI models to select the appropriate model for the current attempt, implementing the failover chain logic.

**Nodes Involved:**  
- Fallback Models

**Node Details:**  

- **Fallback Models**  
  - *Type:* LangChain Code node  
  - *Role:* Selects the AI language model node to use based on the `fail_count` index.  
  - *Configuration:*  
    - Receives an array of AI language model nodes via the `ai_languageModel` input connection.  
    - Reverses the input array to match UI connection order.  
    - Uses `fail_count` as the index to select the model from reversed list.  
  - *Code Logic:*  
    - Throws errors if `fail_count` is not a valid integer or if no model exists at that index.  
    - Returns the selected model node for the AI Agent to use.  
  - *Connections:*  
    - Input: AI language models from all model nodes (`First Model`, `Falback Model`).  
    - Output: AI language model to `AI Agent`.  
  - *Edge cases:*  
    - Index out of range if `fail_count` exceeds available models.  
    - Invalid or missing input connections cause errors.  
  - *Version:* LangChain code node version 1.

---

### 2.3 AI Agent Execution

**Overview:**  
Runs the AI agent using the prompt and the selected AI model. Uses error handling to continue execution on failure, enabling retries with fallback models.

**Nodes Involved:**  
- AI Agent

**Node Details:**  

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Executes the AI prompt task with the selected AI model.  
  - *Configuration:*  
    - Prompt text statically set to `Only output "test".` (meant to be customized).  
    - Prompt type: `define`.  
    - Error handling: `continueErrorOutput` to allow workflow continuation on errors.  
  - *Connections:*  
    - Input: AI language model from `Fallback Models`.  
    - Output on error: sends message to `Agent Variables` (to increment fail count, triggering retry).  
  - *Edge cases:*  
    - Model API errors, timeouts, or rate limits cause failover trigger.  
    - Prompt syntax errors or unexpected response content may affect downstream processing.  
  - *Version:* LangChain agent node version 2.

---

### 2.4 AI Model Nodes

**Overview:**  
These nodes represent the available AI models that the workflow can fallback between.

**Nodes Involved:**  
- First Model (OpenAI GPT-4o)  
- Falback Model (Google Gemini)

**Node Details:**  

- **First Model**  
  - *Type:* LangChain OpenAI Chat model  
  - *Role:* Primary AI model (here configured as `gpt-4o`).  
  - *Configuration:*  
    - Model: `gpt-4o` (custom identifier, could be any valid OpenAI model).  
    - No special options set.  
  - *Connections:* Outputs to `Fallback Models`.  
  - *Edge cases:* API credentials must be configured; API rate limits or quota issues possible.  
  - *Version:* 1.2.

- **Falback Model**  
  - *Type:* LangChain Google Gemini Chat model  
  - *Role:* Secondary fallback AI model.  
  - *Configuration:*  
    - Model Name: `models/gemma-3-27b-it` (example Gemini model).  
    - No extra options set.  
  - *Connections:* Outputs to `Fallback Models`.  
  - *Edge cases:*  
    - Requires Google Gemini API credentials.  
    - Model name must be correctly configured.  
  - *Version:* 1.

---

### 2.5 Configuration Notes

**Overview:**  
Sticky notes provide critical guidance and explanation to users setting up or modifying the workflow.

**Nodes Involved:**  
- Sticky Note (Fallback Chain Config)  
- Sticky Note1 (Prompt Definition)  
- Sticky Note2 (Loop Controller Explanation)

**Node Details:**  

- **Sticky Note (Configuring Fallback Chain)**  
  - Explains how to configure the failover chain by connecting AI models to the `Fallback Models` node.  
  - Emphasizes the importance of connection order as the try order.  

- **Sticky Note1 (Defining Prompt)**  
  - Describes where to define the AI agent prompt (in the `AI Agent` node).  
  - Mentions that the workflow retries the prompt with different models on failure.  

- **Sticky Note2 (Loop Controller)**  
  - Explains the role of `Agent Variables` node in managing the `fail_count` to control retries.  
  - States no user configuration needed here.

---

## 3. Summary Table

| Node Name       | Node Type                        | Functional Role                      | Input Node(s)       | Output Node(s)   | Sticky Note                                                                                         |
|-----------------|---------------------------------|------------------------------------|---------------------|------------------|---------------------------------------------------------------------------------------------------|
| Manual Trigger  | Manual Trigger                  | Entry point to start workflow      |                     | Agent Variables   |                                                                                                   |
| Agent Variables | Set                            | Manages fail_count and models list | Manual Trigger      | AI Agent          | #### 🔁 Loop Controller\nThis node manages the retry loop.\nIt initializes and increments a `fail_count` variable each time the `AI Agent` fails, which tells the `Fallback Models` node to try the next model in the list.\nNo configuration is needed here. |
| AI Agent        | LangChain Agent                | Runs prompt with selected AI model | Agent Variables     | Agent Variables (on error) | #### 📝 DEFINE YOUR PROMPT HERE\nEnter the prompt or task for the AI agent in this node.\nIt will dynamically use the models provided one-by-one from the `Fallback Models` node. If it fails, it will automatically retry with the next model in your chain. |
| First Model     | LangChain OpenAI LM Chat       | Primary AI model (OpenAI GPT-4o)   |                     | Fallback Models   | ### ⚙️ CONFIGURE YOUR FALLBACK CHAIN HERE ⚙️\nThis node selects which AI model to use based on the number of previous failures.\n**To set up your models:**\n1. Add your desired AI model nodes to the canvas (OpenAI, Gemini, Anthropic, etc.).\n2. Connect them to **THIS** node's `ai_languageModel` input.\n**IMPORTANT:** The **order** you connect them in is the order they will be tried. |
| Falback Model   | LangChain Google Gemini LM Chat | Secondary fallback AI model         |                     | Fallback Models   | (Same as First Model)                                                                             |
| Fallback Models | LangChain Code                 | Selects AI model based on fail_count | First Model, Falback Model | AI Agent          | (Same as First Model)                                                                             |
| Sticky Note     | Sticky Note                   | Configuration instructions         |                     |                  | ### ⚙️ CONFIGURE YOUR FALLBACK CHAIN HERE ⚙️\nThis node selects which AI model to use based on the number of previous failures.\n**To set up your models:**\n1. Add your desired AI model nodes to the canvas (OpenAI, Gemini, Anthropic, etc.).\n2. Connect them to **THIS** node's `ai_languageModel` input.\n**IMPORTANT:** The **order** you connect them in is the order they will be tried. |
| Sticky Note1    | Sticky Note                   | Prompt input instructions          |                     |                  | #### 📝 DEFINE YOUR PROMPT HERE\nEnter the prompt or task for the AI agent in this node.\nIt will dynamically use the models provided one-by-one from the `Fallback Models` node. If it fails, it will automatically retry with the next model in your chain. |
| Sticky Note2    | Sticky Note                   | Loop controller explanation        |                     |                  | #### 🔁 Loop Controller\nThis node manages the retry loop.\nIt initializes and increments a `fail_count` variable each time the `AI Agent` fails, which tells the `Fallback Models` node to try the next model in the list.\nNo configuration is needed here. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a `Manual Trigger` node as the workflow entry point. No configuration needed.

2. **Create Agent Variables Node**  
   - Add a `Set` node named `Agent Variables`.  
   - Define two variables:  
     - `models` (type: Array) — initialize with your AI model identifiers, e.g., `["gemini-2.5-flash"]`.  
     - `fail_count` (type: Number) — set the value expression to:  
       `={{ $('Agent Variables')?.isExecuted ? $('Agent Variables').last()?.json?.fail_count + 1 : 0 }}`  
       This expression increments `fail_count` each retry, starting at 0.  
   - Connect `Manual Trigger` output to `Agent Variables` input.

3. **Create AI Agent Node**  
   - Add a LangChain Agent node named `AI Agent`.  
   - Set prompt type to `define`.  
   - Enter your prompt text, for example: `Only output "test".`  
   - Set error handling mode to **Continue on error output** (`continueErrorOutput`) to allow retries.  
   - Connect `Agent Variables` output to `AI Agent` input.

4. **Add AI Model Nodes**  
   - Create nodes for each AI model you want to try in order:  
     - For OpenAI GPT: Add a LangChain OpenAI LM Chat node named `First Model`. Set model to your preferred OpenAI model, e.g., `gpt-4o`.  
     - For Google Gemini: Add a LangChain Google Gemini LM Chat node named `Falback Model`. Set model name, e.g., `models/gemma-3-27b-it`.  
   - Configure API credentials for each model node accordingly (OpenAI API key, Google OAuth2 credentials, etc.).

5. **Create Fallback Models Node**  
   - Add a LangChain Code node named `Fallback Models`.  
   - Use the following code to select the model based on `fail_count`:  
     ```js
     let llms = await this.getInputConnectionData('ai_languageModel', 0);
     llms.reverse(); // reverse array, so the order matches the UI elements

     const llm_index = $input.item.json.fail_count;

     if (!Number.isInteger(llm_index)) {
       throw new Error("'llm_index' is undefined or not a valid integer");
     }

     if(typeof llms[llm_index] === 'undefined') {
       throw new Error(`No LLM found with index ${llm_index}`);
     }

     return llms[llm_index];
     ```  
   - Connect both AI model nodes (`First Model` and `Falback Model`) into the `ai_languageModel` input of this node.  
   - Connect `Fallback Models` output to `AI Agent`'s `ai_languageModel` input.

6. **Connect AI Agent Error Output to Agent Variables**  
   - Connect the `AI Agent` node’s error output back to the `Agent Variables` node to trigger increment of `fail_count` and retry the AI Agent with the next model.

7. **Add Sticky Notes (Optional but Recommended)**  
   - Add sticky notes to guide users on configuration:  
     - One near the AI model nodes and `Fallback Models` to explain the failover chain setup.  
     - One near the `AI Agent` to explain prompt definition and retry logic.  
     - One near the `Agent Variables` node to explain the retry counter.

---

## 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                        |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The order of AI model connections to the `Fallback Models` node defines the retry order of models.   | Sticky Note near `Fallback Models` and model nodes                    |
| To add more fallback models, create additional AI model nodes and connect them to `Fallback Models`. | Workflow configuration instructions                                  |
| The prompt in `AI Agent` is static here but can be adapted dynamically for different tasks.         | Sticky Note near `AI Agent`                                           |
| API credentials must be configured for each AI model node to ensure connectivity and avoid auth errors.| OpenAI and Google Gemini credential setup                            |
| Failover logic depends on `fail_count` being correctly incremented and reset externally if needed.   | Loop Controller explanation in sticky note                           |

---

**Disclaimer:**  
This documentation is generated from an automated n8n workflow export and respects all relevant usage policies. All data and nodes comply with n8n content regulations.

---