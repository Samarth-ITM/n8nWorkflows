Monitor AI Chat Interactions with Gemini 2.5 and Langfuse Tracing

https://n8nworkflows.xyz/workflows/monitor-ai-chat-interactions-with-gemini-2-5-and-langfuse-tracing-4972


# Monitor AI Chat Interactions with Gemini 2.5 and Langfuse Tracing

### 1. Workflow Overview

This workflow monitors AI chat interactions using the Gemini 2.5 language model integrated with Langfuse tracing for advanced observability. It is designed to receive chat messages, process them through an AI agent leveraging Gemini 2.5, and enable detailed tracing of language model calls via Langfuse. The workflow is suitable for applications requiring real-time chat processing with robust monitoring and debugging capabilities.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Triggers the workflow upon receiving a chat message.
- **1.2 AI Language Model Initialization with Langfuse Tracing:** Wraps the Gemini 2.5 model with Langfuse callback handlers to enable tracing.
- **1.3 AI Agent Processing:** Uses the traced AI language model and message context to generate responses.
- **1.4 Memory Buffer Management:** Maintains conversational context to enhance chat continuity.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages and triggers the workflow to process them.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type:* Langchain Chat Trigger  
    - *Role:* Webhook-based trigger node designed to start the workflow when a chat message arrives.  
    - *Configuration:* Uses default options; linked to a unique webhook ID to receive chat inputs.  
    - *Expressions/Variables:* Receives chat message data payload, including session identifiers.  
    - *Input:* External webhook call (no input nodes).  
    - *Output:* Emits chat message data to downstream nodes.  
    - *Version Requirements:* Version 1.1 of the node type.  
    - *Failure Modes:* Network or webhook registration failures; malformed input payloads.  
    - *Sub-workflow:* None.

#### 1.2 AI Language Model Initialization with Langfuse Tracing

- **Overview:**  
  This block initializes the Gemini 2.5 language model and wraps it with Langfuse's CallbackHandler to enable detailed tracing of AI interactions.

- **Nodes Involved:**  
  - gemini-2.5  
  - Langfuse LLM

- **Node Details:**

  - **gemini-2.5**  
    - *Type:* Langchain Google Gemini Chat Model node  
    - *Role:* Provides access to Google Gemini 2.5 language model to generate chat completions.  
    - *Configuration:*  
      - Model: `models/gemini-2.5-flash-preview-05-20`  
      - Temperature: 0.4 (moderate creativity)  
    - *Credentials:* Uses stored Google Palm API credentials named "gemini-personal".  
    - *Input:* None directly connected; provides AI language model object to Langfuse LLM node.  
    - *Output:* AI language model instance for consumption.  
    - *Version Requirements:* Version 1 of this node type.  
    - *Failure Modes:* Authentication errors with Google Palm API, rate limiting, network issues.

  - **Langfuse LLM**  
    - *Type:* Langchain Code node  
    - *Role:* Custom code node to initialize the Langfuse CallbackHandler and wrap the Gemini model for tracing.  
    - *Configuration:*  
      - Contains JavaScript code that:  
        - Imports Langfuse CallbackHandler and Langchain chat model init functions.  
        - Retrieves the AI language model input from the previous node (`gemini-2.5`).  
        - Extracts model provider and model name dynamically.  
        - Instantiates Langfuse CallbackHandler with a sessionId from the input JSON.  
        - Wraps the Gemini model with Langfuse callbacks to enable tracing.  
      - Inputs: Requires one AI language model connection.  
      - Outputs: Provides an AI language model instance enhanced with Langfuse tracing.  
    - *Expressions/Variables:* Uses `$input.item.json.sessionId` to identify Langfuse session.  
    - *Input:* AI language model object from "gemini-2.5" node.  
    - *Output:* Traced AI language model to be used by the AI Agent node.  
    - *Version Requirements:* Version 1 of this node type.  
    - *Failure Modes:* Code execution errors, missing `sessionId`, misconfigured inputs, or dependencies missing in execution environment.

#### 1.3 AI Agent Processing

- **Overview:**  
  This block receives the incoming chat message and processes it through the AI agent, which uses the Langfuse-wrapped Gemini 2.5 model to generate a response.

- **Nodes Involved:**  
  - AI Agent

- **Node Details:**

  - **AI Agent**  
    - *Type:* Langchain AI Agent node  
    - *Role:* Orchestrates AI language model interactions based on the chat input and memory context.  
    - *Configuration:* Default options; consumes traced AI language model and chat input.  
    - *Expressions/Variables:* Receives AI language model from Langfuse LLM node and chat message from trigger node.  
    - *Input:*  
      - Main input from "When chat message received" (chat messages).  
      - AI language model input from "Langfuse LLM" node.  
      - AI memory input from "mem" node.  
    - *Output:* AI-generated chat response (no further downstream nodes in this workflow).  
    - *Version Requirements:* Version 2 of this node type.  
    - *Failure Modes:* AI model errors, timeout, unexpected input data structure, or memory context issues.

#### 1.4 Memory Buffer Management

- **Overview:**  
  Maintains a windowed buffer of recent conversation messages to provide context continuity for the AI agent.

- **Nodes Involved:**  
  - mem

- **Node Details:**

  - **mem**  
    - *Type:* Langchain Memory Buffer Window  
    - *Role:* Stores recent conversation snippets to maintain context within a defined window size.  
    - *Configuration:*  
      - Context window length: 100 (units unspecified but typically tokens or messages).  
    - *Input:* Receives AI memory context output from "AI Agent" node.  
    - *Output:* Sends memory context back to "AI Agent" node to augment processing.  
    - *Version Requirements:* Version 1.3 of this node type.  
    - *Failure Modes:* Context overflow, memory corruption, or misalignment with AI agent expectations.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)               | Output Node(s)           | Sticky Note                                  |
|-------------------------|----------------------------------|----------------------------------------|-----------------------------|--------------------------|----------------------------------------------|
| When chat message received | Langchain Chat Trigger            | Entry point; triggers workflow on chat message | External webhook             | AI Agent                 |                                              |
| gemini-2.5              | Langchain Google Gemini Chat Model | Provides Gemini 2.5 AI language model | None                        | Langfuse LLM             |                                              |
| Langfuse LLM            | Langchain Code                   | Wraps Gemini model with Langfuse tracing | gemini-2.5                  | AI Agent                 |                                              |
| AI Agent                | Langchain AI Agent               | Processes chat input using traced AI model | When chat message received, Langfuse LLM, mem | mem                      |                                              |
| mem                     | Langchain Memory Buffer Window   | Maintains conversational context      | AI Agent                    | AI Agent                 |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook to receive chat messages. Use default options.  
   - Position it as the start node.

2. **Create "gemini-2.5" node**  
   - Type: Langchain Google Gemini Chat Model  
   - Set modelName to `models/gemini-2.5-flash-preview-05-20`.  
   - Set temperature to 0.4.  
   - Assign Google Palm API credentials named "gemini-personal".  
   - No inputs connected.

3. **Create "Langfuse LLM" node**  
   - Type: Langchain Code  
   - Add JavaScript code as follows:  
     - Import `CallbackHandler` from `langfuse-langchain`.  
     - Import `initChatModel` from `langchain/chat_models/universal`.  
     - Retrieve the AI language model from input connection.  
     - Extract provider and model name.  
     - Initialize `CallbackHandler` with sessionId from input JSON (`$input.item.json.sessionId`).  
     - Initialize chat model with Langfuse callback.  
     - Return the wrapped LLM instance.  
   - Input: Connect AI language model output from "gemini-2.5".  
   - Output: Provide traced AI language model.

4. **Create "AI Agent" node**  
   - Type: Langchain AI Agent  
   - Use default options.  
   - Inputs:  
     - Connect main input from "When chat message received".  
     - Connect AI language model input from "Langfuse LLM".  
     - Connect AI memory input from "mem" node (created next).  
   - Output: No downstream nodes required (or connect as needed).

5. **Create "mem" node**  
   - Type: Langchain Memory Buffer Window  
   - Set context window length to 100.  
   - Inputs: Connect AI memory input from "AI Agent".  
   - Outputs: Connect back to AI memory input of "AI Agent" (forming a loop for context).

6. **Validate Connections:**  
   - "When chat message received" → main input → "AI Agent"  
   - "gemini-2.5" → AI language model → "Langfuse LLM"  
   - "Langfuse LLM" → AI language model → "AI Agent"  
   - "AI Agent" → AI memory → "mem"  
   - "mem" → AI memory → "AI Agent"

7. **Credentials Setup:**  
   - Ensure Google Palm API credentials are configured with valid API keys for Gemini 2.5.  
   - No additional credentials needed for Langfuse as it is embedded in code node callbacks.

8. **Testing:**  
   - Deploy webhook for the chat trigger.  
   - Send test chat messages with `sessionId` included in payload JSON to ensure Langfuse tracing works.  
   - Monitor Langfuse dashboard for trace data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                |
|--------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Langfuse CallbackHandler enables comprehensive tracing and monitoring of AI model interactions, enriching debugging and analytics. | https://docs.langfuse.com/integrations/langchain             |
| Gemini 2.5 model access requires Google Palm API credentials; ensure access rights and quota are properly configured.           | https://developers.generativeai.google/api/gemini             |
| SessionId in incoming chat messages is essential for Langfuse to correlate traces with user sessions.                          | Workflow code references `$input.item.json.sessionId`         |
| The memory buffer length (100) can be adjusted based on conversation length and token limits of the AI model.                   | Adjust in "mem" node parameter `contextWindowLength`          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.