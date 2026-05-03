Smart Chat Routing Between Gemini and GPT Models Based on Query Complexity

https://n8nworkflows.xyz/workflows/smart-chat-routing-between-gemini-and-gpt-models-based-on-query-complexity-9247


# Smart Chat Routing Between Gemini and GPT Models Based on Query Complexity

### 1. Workflow Overview

This workflow implements an intelligent chat routing system that classifies user queries by complexity and dynamically selects the most appropriate large language model (LLM) to generate responses. It optimizes cost and performance by routing complex queries to Google Gemini 2.5 Pro and simpler queries to OpenAI GPT-4.1 Nano. The workflow includes these logical blocks:

- **1.1 Input Reception:** Captures incoming user chat messages in real-time.
- **1.2 Query Complexity Classification:** Uses a LangChain agent to classify the query as either complex or simple.
- **1.3 Model Selection Logic:** Applies rules to route queries based on classification output.
- **1.4 Response Generation:** Invokes the selected LLM model to generate the chat response.
- **1.5 Notes and Documentation:** Provides embedded documentation and usage instructions via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new incoming chat messages via a webhook-enabled chat trigger node, initiating the workflow each time a user sends a chat input.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Captures real-time chat inputs through webhook.  
    - Configuration: Default options, webhook ID set for external integrations.  
    - Expressions/Variables: Outputs JSON with field `chatInput` containing the user's message.  
    - Inputs: None (trigger node)  
    - Outputs: Passes chat input to `Model Selector` node.  
    - Version: 1.3  
    - Edge Cases: Webhook misconfiguration, network failures, or missing input field could cause failures.  
    - Sticky Note: "## 📥 When chat message received\n\n**Purpose:** For real-time chat."

---

#### 2.2 Query Complexity Classification

- **Overview:**  
  This block analyzes the user's chat input to classify the complexity of the query into two categories: complex (1) or simple (2). The classification guides which language model will be used.

- **Nodes Involved:**  
  - `Model Selector`

- **Node Details:**  
  - **Model Selector**  
    - Type: `@n8n/n8n-nodes-langchain.agent` (LangChain Agent)  
    - Role: AI agent designed to classify query complexity.  
    - Configuration:  
      - Input text bound to the incoming chat input: `={{ $json.chatInput }}`  
      - Prompt:  
        - System message instructs the agent to output ONLY the number "1" or "2" based on complexity without any extra text or formatting.  
        - Classification rules:  
          - Output 1 for complex questions requiring reasoning, research, coding, etc.  
          - Output 2 for simple, straightforward questions.  
    - Input: From `When chat message received`.  
    - Output: JSON with field `output` containing classification number as text.  
    - Version: 1.9  
    - Edge Cases:  
      - Misclassification due to ambiguous prompts  
      - Agent outputting unexpected text breaking downstream logic  
      - Timeout or API errors from LangChain/OpenAI services  
    - Sticky Note: "## 🤖Model Selector\n\n**Purpose:** Classifies query complexity.\n\n**Note:** Outputs 1 (complex) or 2 (simple)."

---

#### 2.3 Model Selection Logic

- **Overview:**  
  This block applies conditional routing rules that interpret the classification output from the previous block and select the appropriate LLM model accordingly.

- **Nodes Involved:**  
  - `Model selector`

- **Node Details:**  
  - **Model selector**  
    - Type: `@n8n/n8n-nodes-langchain.modelSelector`  
    - Role: Implements rule-based selection of AI language model based on classification.  
    - Configuration:  
      - Contains two rules matching the classification number:  
        - If classification output equals 1 (complex), route to the first AI model output (Gemini 2.5 Pro).  
        - If classification output equals 2 (simple), route to the second AI model output (GPT-4.1 Nano).  
      - Values are dynamically obtained from the `"Model Selector"` node output, converted to number.  
    - Inputs: From `Model Selector` node (classification output).  
    - Outputs: Routes the flow to one of the two LLM nodes (`2.5 pro` or `4.1 nano`).  
    - Version: 1  
    - Edge Cases:  
      - Unexpected output values outside 1 or 2 may cause no routing or errors.  
      - Expression failures if input is missing or malformed.  
      - Rules must be kept in sync with classification logic.  
    - Sticky Note: "## 📥 When chat message received\n\n**Purpose:** For real-time chat." (Note covers input reception and selection logic nodes.)

---

#### 2.4 Response Generation

- **Overview:**  
  This block consists of the actual LLM nodes that generate chat responses based on the selected model. The Gemini 2.5 Pro model handles complex queries, while GPT-4.1 Nano handles simple queries. The output is then passed to the main agent for generating the final response.

- **Nodes Involved:**  
  - `2.5 pro` (Google Gemini 2.5 Pro)  
  - `4.1 nano` (OpenAI GPT-4.1 Nano)  
  - `Main Agent`

- **Node Details:**  
  - **2.5 pro**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
    - Role: AI language model node using Google Gemini 2.5 Pro.  
    - Configuration:  
      - Model name set to `models/gemini-2.5-pro`.  
      - Default options.  
    - Credentials: Google PaLM API credential named `v9`.  
    - Input: Routed from `Model selector` if classification = 1.  
    - Output: Sent to `Main Agent`.  
    - Version: 1  
    - Edge Cases: API quota limits, authentication errors, network timeouts.  

  - **4.1 nano**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: AI language model node using OpenAI GPT-4.1 Nano.  
    - Configuration:  
      - Model set to `gpt-4.1-nano`.  
      - Default options.  
    - Credentials: OpenAI API credential.  
    - Input: Routed from `Model selector` if classification = 2.  
    - Output: Sent to `Main Agent`.  
    - Version: 1.2  
    - Edge Cases: API key restrictions, rate limits, network issues.  

  - **Main Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Final response generator agent that formats or enhances the LLM output.  
    - Configuration:  
      - Uses the original chat input from `When chat message received` node.  
      - System prompt injects the current date/time dynamically (`={{ $now }}`).  
    - Input: Receives output from `Model Selector` node (which in turn outputs from the selected LLM).  
    - Output: Final chat response to downstream systems or UI.  
    - Version: 2.2  
    - Edge Cases: Expression evaluation failures for date/time, API errors.  
    - Sticky Note: "## 💬 Node: Main Agent\n\n**Purpose:** Generates response with selected model.\n\n**Note:** Includes current date in prompt."

---

#### 2.5 Notes and Documentation

- **Overview:**  
  Provides embedded documentation, setup instructions, and troubleshooting tips as sticky notes within the workflow canvas for user reference.

- **Nodes Involved:**  
  - `Overview Note`  
  - `Note: Trigger`  
  - `Note: Classifier`  
  - `Note: Main Agent`

- **Node Details:**  
  - **Overview Note**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: Comprehensive overview of workflow purpose, prerequisites, credentials, configuration steps, use cases, and troubleshooting.  
    - Positioned prominently to guide users importing or maintaining the workflow.  
    - No input/output connections.  

  - **Note: Trigger**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: Brief purpose note about the chat trigger node.  

  - **Note: Classifier**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: Explains the model selector node’s role in classification and outputs.  

  - **Note: Main Agent**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: Describes the final response generator including the dynamic date prompt injection.

---

### 3. Summary Table

| Node Name               | Node Type                                 | Functional Role                               | Input Node(s)                 | Output Node(s)          | Sticky Note                                                                                                              |
|-------------------------|------------------------------------------|-----------------------------------------------|------------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger    | Captures incoming chat messages via webhook  | None                         | Model Selector           | ## 📥 When chat message received  **Purpose:** For real-time chat.                                                        |
| Model Selector          | @n8n/n8n-nodes-langchain.agent           | Classifies query complexity (1=complex, 2=simple) | When chat message received    | Model selector           | ## 🤖Model Selector  **Purpose:** Classifies query complexity.  **Note:** Outputs 1 (complex) or 2 (simple).              |
| Model selector          | @n8n/n8n-nodes-langchain.modelSelector   | Routes query to appropriate LLM based on classification | Model Selector               | 2.5 pro, 4.1 nano        |                                                                                                                          |
| 2.5 pro                 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Handles complex queries with Gemini 2.5 Pro   | Model selector               | Main Agent               |                                                                                                                          |
| 4.1 nano                | @n8n/n8n-nodes-langchain.lmChatOpenAi    | Handles simple queries with GPT-4.1 Nano      | Model selector               | Main Agent               |                                                                                                                          |
| Main Agent              | @n8n/n8n-nodes-langchain.agent           | Generates final chat response, includes date | Model Selector (via modelSelector output) | None                    | ## 💬 Node: Main Agent  **Purpose:** Generates response with selected model.  **Note:** Includes current date in prompt. |
| Overview Note           | n8n-nodes-base.stickyNote                 | Documentation and setup instructions          | None                         | None                    | # Adaptive LLM Router for Optimized AI Chat Responses ... (full content as in workflow)                                  |
| Note: Trigger           | n8n-nodes-base.stickyNote                 | Notes on chat trigger node                      | None                         | None                    | ## 📥 When chat message received  **Purpose:** For real-time chat.                                                        |
| Note: Classifier        | n8n-nodes-base.stickyNote                 | Notes on classification logic                   | None                         | None                    | ## 🤖Model Selector  **Purpose:** Classifies query complexity.  **Note:** Outputs 1 (complex) or 2 (simple).              |
| Note: Main Agent        | n8n-nodes-base.stickyNote                 | Notes on final response generation              | None                         | None                    | ## 💬 Node: Main Agent  **Purpose:** Generates response with selected model.  **Note:** Includes current date in prompt. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add node: `When chat message received` (`@n8n/n8n-nodes-langchain.chatTrigger`)  
   - Configure webhook with unique ID for chat inputs. Leave options default.  
   - Position it as the start of the workflow.

2. **Create Query Complexity Classifier Node**  
   - Add node: `Model Selector` (`@n8n/n8n-nodes-langchain.agent`)  
   - Set parameter `text` to `={{ $json.chatInput }}` to receive user input from the trigger.  
   - Set `systemMessage` with instructions to classify query complexity, output only 1 or 2 (see overview for exact prompt content).  
   - Choose prompt type `define`.  
   - Connect output of `When chat message received` to input of this node.

3. **Create Model Routing Node**  
   - Add node: `Model selector` (`@n8n/n8n-nodes-langchain.modelSelector`)  
   - Configure two rules:  
     - Rule 1: If output from `Model Selector` equals number 1, route to first AI language model output.  
     - Rule 2: If output equals number 2, route to second AI language model output.  
   - Use expression: `={{ $('Model Selector').item.json.output.toNumber() }}` for condition value.  
   - Connect output of `Model Selector` node to this node.

4. **Add LLM Nodes for Each Model**  
   - Add `2.5 pro` node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)  
     - Set `modelName` to `"models/gemini-2.5-pro"`.  
     - Assign Google PaLM API credentials (create beforehand in n8n).  
     - Connect first output of `Model selector` node to this node.  

   - Add `4.1 nano` node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)  
     - Set `model` to `"gpt-4.1-nano"`.  
     - Assign OpenAI API credentials (create beforehand in n8n).  
     - Connect second output of `Model selector` node to this node.

5. **Add Final Response Generator Node**  
   - Add node: `Main Agent` (`@n8n/n8n-nodes-langchain.agent`)  
   - Set parameter `text` to `={{ $('When chat message received').item.json.chatInput }}`  
   - Set system message to include current date/time: `=Here is the current date/time: {{ $now }}`  
   - Connect outputs of both `2.5 pro` and `4.1 nano` nodes to this node.

6. **Add Sticky Notes for Documentation** (optional but recommended)  
   - Add sticky notes with content as per the overview note and individual node notes for clarity and maintenance.

7. **Activate Workflow**  
   - Ensure all credentials (OpenAI and Google PaLM) are properly set up and assigned.  
   - Activate the workflow.  
   - Test by sending chat inputs through the webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                 | Context or Link                                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow optimizes AI chat responses by adaptively routing queries based on complexity to balance cost and performance between Gemini and GPT models.                                                                                   | Overview Note sticky content                                                                                                  |
| OpenAI API key setup instructions: create keys at https://platform.openai.com/api-keys and add them to n8n credentials.                                                                                                                       | Overview Note sticky content                                                                                                  |
| Google Gemini API key setup: generate API key via https://makersuite.google.com/app/apikey and add as Google PaLM API credential in n8n.                                                                                                     | Overview Note sticky content                                                                                                  |
| Troubleshooting tips include verifying prompt outputs, API quotas, webhook activation, and expression correctness.                                                                                                                           | Overview Note sticky content                                                                                                  |
| The classification prompt is strict to avoid output parsing errors — only “1” or “2” with no extra formatting permitted.                                                                                                                      | Model Selector node description                                                                                               |
| The main agent node enriches responses with the current timestamp dynamically injected into the prompt.                                                                                                                                      | Main Agent sticky note                                                                                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.