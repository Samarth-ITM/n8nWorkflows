Build Custom AI Agent with LangChain & Gemini (Self-Hosted)

https://n8nworkflows.xyz/workflows/build-custom-ai-agent-with-langchain---gemini--self-hosted--3326


# Build Custom AI Agent with LangChain & Gemini (Self-Hosted)

### 1. Workflow Overview

This workflow implements a fully customizable conversational AI agent using LangChain’s Code node integrated with Google Gemini (PaLM) language model. It is designed for users who want granular control over the agent’s personality, prompt engineering, and conversation memory, while minimizing token usage by avoiding built-in tool-calling overhead. The workflow is self-hosted and leverages n8n’s LangChain nodes for chat triggering, memory management, and AI model interaction.

**Target Use Cases:**  
- Developers or AI practitioners building tailored conversational agents with specific personalities and response styles.  
- Scenarios requiring conversation history management and prompt customization beyond standard n8n Conversation Agent capabilities.  
- Deployments using Google Gemini or alternative AI providers with LangChain integration.

**Logical Blocks:**  
- **1.1 Input Reception:** Captures user chat messages and manages session initiation.  
- **1.2 Memory Management:** Stores and retrieves conversation history with windowed memory buffer.  
- **1.3 AI Processing:** Constructs a custom prompt, invokes the language model, and generates responses.  
- **1.4 Configuration & Documentation:** Sticky notes provide setup instructions, customization tips, and operational guidance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming chat messages via a webhook, manages session state, and triggers the workflow execution.

**Nodes Involved:**  
- When chat message received

**Node Details:**  

- **When chat message received**  
  - *Type:* `@n8n/n8n-nodes-langchain.chatTrigger`  
  - *Role:* Entry point for chat messages; exposes a public webhook URL for chat clients.  
  - *Configuration:*  
    - Public access enabled (`public: true`)  
    - Response mode set to return output from the last node (`responseMode: lastNode`)  
    - Allowed origins set to all (`allowedOrigins: "*"`)  
    - Loads previous session data from memory (`loadPreviousSession: memory`)  
    - Initial messages empty (no preloaded conversation)  
  - *Input Connections:* None (trigger node)  
  - *Output Connections:* Main output connected to `Construct & Execute LLM Prompt` node  
  - *Version:* 1.1  
  - *Potential Failures:*  
    - Webhook connectivity issues or unauthorized access if public flag is misconfigured  
    - Session loading errors if memory node malfunctions  
  - *Notes:*  
    - Chat UI elements such as title can be configured here (see Sticky Note2)  
    - Provides URL for external chat interface access  

---

#### 1.2 Memory Management

**Overview:**  
Manages conversation history using a windowed buffer memory to maintain context without excessive token usage.

**Nodes Involved:**  
- Store conversation history

**Node Details:**  

- **Store conversation history**  
  - *Type:* `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - *Role:* Maintains a sliding window of recent conversation messages to provide context for the AI model.  
  - *Configuration:* Default parameters (no custom window size specified here, but adjustable)  
  - *Input Connections:* None explicitly shown; connected as AI memory input to both `Construct & Execute LLM Prompt` and `When chat message received` nodes  
  - *Output Connections:* AI memory output connected to `Construct & Execute LLM Prompt` and `When chat message received` nodes  
  - *Version:* 1.3  
  - *Potential Failures:*  
    - Memory overflow or loss if window size is too small or improperly configured  
    - Data corruption or session mismatch  
  - *Notes:*  
    - Memory length can be adjusted to control conversation history size (see Sticky Note4)  

---

#### 1.3 AI Processing

**Overview:**  
Constructs a custom prompt template defining the agent’s personality and conversation rules, executes the prompt using LangChain’s ConversationChain with the selected language model and memory, and returns the generated response.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Construct & Execute LLM Prompt

**Node Details:**  

- **Google Gemini Chat Model**  
  - *Type:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  - *Role:* Provides access to Google Gemini language model for chat completions.  
  - *Configuration:*  
    - Model: `models/gemini-2.0-flash-exp`  
    - Temperature: 0.7 (balanced creativity)  
    - Safety settings configured to allow all content except blocking sexually explicit content is disabled (`BLOCK_NONE`)  
  - *Credentials:* Uses Google Palm API credentials configured externally  
  - *Input Connections:* None (model node)  
  - *Output Connections:* AI language model output connected to `Construct & Execute LLM Prompt` node  
  - *Version:* 1  
  - *Potential Failures:*  
    - API authentication errors if credentials invalid or expired  
    - Rate limiting or quota exceeded errors from Google API  
    - Model unavailability or version deprecation  
  - *Notes:*  
    - Model can be swapped by changing the `modelName` parameter (see Sticky Note3)  

- **Construct & Execute LLM Prompt**  
  - *Type:* `@n8n/n8n-nodes-langchain.code`  
  - *Role:* Implements LangChain ConversationChain with a custom prompt template and memory, executes the prompt, and returns the AI-generated output.  
  - *Configuration:*  
    - Uses LangChain’s `PromptTemplate`, `ConversationChain`, and `BufferMemory` classes  
    - Prompt template defines a persona: user’s girlfriend named “Bunny” with specific personality traits, speaking exclusively in Chinese, no questions, concise responses  
    - Template includes placeholders `{chat_history}` and `{input}` which must be preserved for LangChain to function correctly  
    - Receives inputs:  
      - `main` input: user chat message as `chatInput`  
      - `ai_languageModel`: connected to Google Gemini Chat Model output  
      - `ai_memory`: connected to Store conversation history output  
    - Returns output as the main output of the node  
  - *Input Connections:*  
    - Main input from `When chat message received` node  
    - AI language model input from `Google Gemini Chat Model` node  
    - AI memory input from `Store conversation history` node  
  - *Output Connections:* None (last node in chain)  
  - *Version:* 1  
  - *Potential Failures:*  
    - Code execution errors if LangChain classes or methods change in future versions  
    - Missing or malformed input data causing runtime exceptions  
    - Template errors if placeholders are removed or altered  
  - *Notes:*  
    - This node is the core of prompt engineering and agent personality definition (see Sticky Note)  

---

#### 1.4 Configuration & Documentation

**Overview:**  
Sticky Note nodes provide inline documentation, setup instructions, and customization guidance to users and maintainers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**  

- **Sticky Note** (positioned near prompt node)  
  - Content:  
    - Emphasizes prompt engineering importance  
    - Warns to preserve `{chat_history}` and `{input}` placeholders in the prompt template for LangChain operation  

- **Sticky Note1** (positioned near trigger node)  
  - Content:  
    - Setup instructions for Gemini credentials with link: [Get API key here](https://ai.google.dev/)  
    - Interaction methods: test in editor or via chat URL  

- **Sticky Note2** (near trigger node)  
  - Content:  
    - Interface settings hint: configure chat UI elements like title in `When chat message received` node  

- **Sticky Note3** (near Google Gemini Chat Model node)  
  - Content:  
    - Model selection tip: swap language models via `language model` input in `Construct & Execute LLM Prompt` node  

- **Sticky Note4** (near Store conversation history node)  
  - Content:  
    - Memory control tip: adjust conversation history length in `Store conversation history` node  

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                         | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                  |
|-----------------------------|--------------------------------------------|---------------------------------------|------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received   | @n8n/n8n-nodes-langchain.chatTrigger       | Entry point for chat messages          | None                         | Construct & Execute LLM Prompt  | 👆 **Interface Settings** Configure chat UI elements (e.g., title) in this node                                |
| Google Gemini Chat Model     | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Google Gemini LLM              | None                         | Construct & Execute LLM Prompt  | 👆 **Model Selection** Swap language models via `language model` input in `Construct & Execute LLM Prompt`     |
| Store conversation history   | @n8n/n8n-nodes-langchain.memoryBufferWindow | Manages conversation history memory    | None                         | Construct & Execute LLM Prompt, When chat message received | 👆 **Memory Control** Adjust conversation history length in this node                                         |
| Construct & Execute LLM Prompt | @n8n/n8n-nodes-langchain.code             | Custom prompt construction and AI call | When chat message received, Google Gemini Chat Model, Store conversation history | None                           | 👇 **Prompt Engineering** Define agent personality and conversation structure here; preserve `{chat_history}` and `{input}` placeholders |
| Sticky Note                 | n8n-nodes-base.stickyNote                   | Documentation and guidance             | None                         | None                           | 👇 **Prompt Engineering** Define agent personality and conversation structure in the prompt template          |
| Sticky Note1                | n8n-nodes-base.stickyNote                   | Setup instructions                     | None                         | None                           | Setup Instructions: Configure Gemini credentials ([Get API key here](https://ai.google.dev/)); interaction methods |
| Sticky Note2                | n8n-nodes-base.stickyNote                   | Interface settings hint                | None                         | None                           | 👆 **Interface Settings** Configure chat UI elements (e.g., title) in `When chat message received` node       |
| Sticky Note3                | n8n-nodes-base.stickyNote                   | Model selection hint                   | None                         | None                           | 👆 **Model Selection** Swap language models through the `language model` input field in `Construct & Execute LLM Prompt` |
| Sticky Note4                | n8n-nodes-base.stickyNote                   | Memory control hint                    | None                         | None                           | 👆 **Memory Control** Adjust conversation history length in the `Store Conversation History` node             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a `When chat message received` node (`@n8n/n8n-nodes-langchain.chatTrigger`).  
   - Set `public` to `true` to allow external chat clients.  
   - Set `responseMode` to `lastNode`.  
   - Allow all origins (`allowedOrigins: "*"`) for CORS.  
   - Enable `loadPreviousSession` with `memory` to maintain conversation history.  
   - Leave `initialMessages` empty or customize as needed.  

2. **Create Memory Node:**  
   - Add a `Store conversation history` node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`).  
   - Use default settings or configure window size to control how much conversation history is retained.  

3. **Create Language Model Node:**  
   - Add a `Google Gemini Chat Model` node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`).  
   - Configure `modelName` to `models/gemini-2.0-flash-exp` or another Gemini model.  
   - Set `temperature` to `0.7` for balanced creativity.  
   - Configure `safetySettings` to allow all content except blocking sexually explicit content is disabled (`BLOCK_NONE`).  
   - Attach Google Palm API credentials (OAuth2 or API key) under `credentials`.  

4. **Create LangChain Code Node:**  
   - Add a `Construct & Execute LLM Prompt` node (`@n8n/n8n-nodes-langchain.code`).  
   - Paste the provided JavaScript code implementing LangChain’s `PromptTemplate`, `ConversationChain`, and `BufferMemory`.  
   - Ensure the prompt template includes `{chat_history}` and `{input}` placeholders exactly as in the example.  
   - Configure inputs:  
     - Connect main input to `When chat message received` node output.  
     - Connect AI language model input to `Google Gemini Chat Model` node output.  
     - Connect AI memory input to `Store conversation history` node output.  
   - No outputs need to be connected further (last node).  

5. **Connect Nodes:**  
   - Connect `When chat message received` main output to `Construct & Execute LLM Prompt` main input.  
   - Connect `Google Gemini Chat Model` output to `Construct & Execute LLM Prompt` AI language model input.  
   - Connect `Store conversation history` output to `Construct & Execute LLM Prompt` AI memory input and also to `When chat message received` AI memory input.  

6. **Add Sticky Notes (Optional but Recommended):**  
   - Add sticky notes for setup instructions, prompt engineering tips, interface settings, model selection, and memory control as per the original workflow for documentation clarity.  

7. **Credential Setup:**  
   - Configure Google Palm API credentials in n8n credentials manager with valid API key or OAuth2 token.  
   - Ensure credentials are linked to the `Google Gemini Chat Model` node.  

8. **Testing:**  
   - Use the “Chat” button in the `When chat message received` node to test the agent in the editor.  
   - Alternatively, activate the workflow and use the webhook URL to interact via external chat clients.  

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires self-hosted n8n to use the LangChain Code node.                                     | [LangChain Code node docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.code/) |
| Google Gemini API key is required for the Gemini Chat Model node; obtain it at Google AI platform.         | [Get API key here](https://ai.google.dev/)                                                      |
| Prompt template must preserve `{chat_history}` and `{input}` placeholders for LangChain to operate properly. | Prompt engineering best practices                                                                |
| Customize chat UI elements like title in the `When chat message received` node parameters.                  | Interface customization                                                                          |
| Adjust conversation history length in the `Store conversation history` node to balance context and token usage. | Memory management                                                                               |

---

This documentation provides a complete, structured reference to understand, reproduce, and customize the “Build Custom AI Agent with LangChain & Gemini (Self-Hosted)” workflow in n8n.