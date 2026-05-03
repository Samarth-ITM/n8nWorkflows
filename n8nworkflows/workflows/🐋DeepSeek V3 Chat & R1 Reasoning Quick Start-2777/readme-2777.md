🐋DeepSeek V3 Chat & R1 Reasoning Quick Start

https://n8nworkflows.xyz/workflows/--deepseek-v3-chat---r1-reasoning-quick-start-2777


# 🐋DeepSeek V3 Chat & R1 Reasoning Quick Start

### 1. Workflow Overview

This workflow, titled **🐋DeepSeek V3 Chat & R1 Reasoning Quick Start**, demonstrates multiple integration methods to leverage DeepSeek’s AI models within an n8n automation pipeline. It targets users who want to experiment with or deploy DeepSeek’s conversational and reasoning AI models, either via local deployment or API calls, including advanced conversational agents with memory.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages to trigger the workflow.
- **1.2 Basic Language Model Chain:** Provides a simple initial response using a basic LLM chain.
- **1.3 DeepSeek Reasoner via Ollama Local Model:** Uses the Ollama local deployment of DeepSeek-R1 for advanced reasoning.
- **1.4 DeepSeek Reasoner via HTTP Raw Request:** Sends raw JSON HTTP requests to DeepSeek’s API for reasoning tasks.
- **1.5 DeepSeek Chat V3 via HTTP JSON Request:** Sends structured JSON HTTP requests to DeepSeek’s Chat V3 API.
- **1.6 Conversational AI Agent with Memory Buffer:** Implements a conversational agent with persistent context using a memory buffer and DeepSeek Reasoner model.
- **1.7 Notes and Documentation:** Sticky notes providing contextual information, setup instructions, and references.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages to trigger the workflow and initiate AI processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point webhook node that triggers the workflow upon receiving a chat message.  
    - Configuration: Default options, no custom filters.  
    - Inputs: External chat message via webhook.  
    - Outputs: Passes chat input to the next node (Basic LLM Chain2).  
    - Edge Cases: Webhook connectivity issues, malformed input data.  
    - Version: 1.1

#### 1.2 Basic Language Model Chain

- **Overview:**  
  Provides a simple, initial AI response using a basic language model chain with a fixed system message.

- **Nodes Involved:**  
  - Basic LLM Chain2

- **Node Details:**  
  - **Basic LLM Chain2**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Role: Processes input with a basic LLM chain using a static system message "You are a helpful assistant."  
    - Configuration: Single message with fixed content; no dynamic variables.  
    - Inputs: Receives chat input from "When chat message received".  
    - Outputs: Feeds output to "Ollama DeepSeek" node.  
    - Edge Cases: LLM API failures, timeout, or invalid input.  
    - Version: 1.5

#### 1.3 DeepSeek Reasoner via Ollama Local Model

- **Overview:**  
  Uses the Ollama local deployment of DeepSeek-R1 model for advanced reasoning tasks.

- **Nodes Involved:**  
  - Ollama DeepSeek

- **Node Details:**  
  - **Ollama DeepSeek**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOllama`  
    - Role: Sends chat input to a locally deployed DeepSeek-R1 model via Ollama API.  
    - Configuration:  
      - Model: `deepseek-r1:14b`  
      - Options: Default format, context window size 16384 tokens, temperature 0.6 for creativity.  
    - Credentials: Uses Ollama API credentials configured locally (e.g., localhost).  
    - Inputs: Receives output from "Basic LLM Chain2".  
    - Outputs: Passes results to "Basic LLM Chain2" (feedback loop not explicitly connected in JSON, but logically next).  
    - Edge Cases: Local Ollama service down, model not installed, network issues, invalid credentials.  
    - Version: 1

#### 1.4 DeepSeek Reasoner via HTTP Raw Request

- **Overview:**  
  Sends raw JSON HTTP POST requests to DeepSeek’s API for reasoning using the DeepSeek-R1 model.

- **Nodes Involved:**  
  - DeepSeek Raw Body

- **Node Details:**  
  - **DeepSeek Raw Body**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Makes HTTP POST requests with raw JSON body to DeepSeek API endpoint `/chat/completions`.  
    - Configuration:  
      - URL: `https://api.deepseek.com/chat/completions`  
      - Method: POST  
      - Body: Raw JSON specifying model `deepseek-reasoner` and user message from input (`{{ $json.chatInput.trim() }}`)  
      - Content-Type: `application/json`  
      - Authentication: HTTP header with API key (generic HTTP header auth).  
    - Inputs: Receives chat input from upstream nodes (not explicitly connected in JSON but part of workflow).  
    - Outputs: Returns API response for further processing or output.  
    - Edge Cases: API key invalid or expired, network timeout, malformed JSON, API rate limits.  
    - Version: 4.2

#### 1.5 DeepSeek Chat V3 via HTTP JSON Request

- **Overview:**  
  Sends structured JSON HTTP POST requests to DeepSeek Chat V3 API for conversational completions.

- **Nodes Involved:**  
  - DeepSeek JSON Body

- **Node Details:**  
  - **DeepSeek JSON Body**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Sends JSON body requests to DeepSeek Chat V3 API endpoint `/chat/completions`.  
    - Configuration:  
      - URL: `https://api.deepseek.com/chat/completions`  
      - Method: POST  
      - JSON Body:  
        - Model: `deepseek-chat`  
        - Messages: System message dynamically set from input (`{{ $json.chatInput }}`) and a fixed user message "Hello!"  
        - Stream: false  
      - Authentication: HTTP header with API key.  
    - Inputs: Receives chat input from upstream nodes (not explicitly connected in JSON but part of workflow).  
    - Outputs: Returns API response for further use.  
    - Edge Cases: API key issues, malformed JSON, network errors, unexpected API responses.  
    - Version: 4.2

#### 1.6 Conversational AI Agent with Memory Buffer

- **Overview:**  
  Implements a conversational agent with persistent context using a window buffer memory and DeepSeek Reasoner model for advanced reasoning.

- **Nodes Involved:**  
  - AI Agent  
  - Window Buffer Memory  
  - DeepSeek

- **Node Details:**  
  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Conversational agent node that manages dialogue state and reasoning.  
    - Configuration:  
      - Agent type: `conversationalAgent`  
      - System message: "You are a helpful assistant."  
      - Error handling: Continues on error, retries on failure enabled.  
      - Always outputs data regardless of errors.  
    - Inputs: Receives memory from "Window Buffer Memory" and language model from "DeepSeek".  
    - Outputs: Final conversational response.  
    - Edge Cases: Model API failures, memory overflow, retry limits exceeded.  
    - Version: 1.7

  - **Window Buffer Memory**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Maintains a sliding window of conversation history to provide context to the AI agent.  
    - Configuration: Default parameters (window size unspecified, uses defaults).  
    - Inputs: Receives conversation data from upstream nodes.  
    - Outputs: Provides memory context to "AI Agent".  
    - Edge Cases: Memory overflow, data corruption.  
    - Version: 1.3

  - **DeepSeek**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Language model node configured to use DeepSeek Reasoner (`deepseek-reasoner`) via OpenAI-compatible API.  
    - Configuration: Model set to `deepseek-reasoner`.  
    - Credentials: Uses DeepSeek API key configured as OpenAI API credential type.  
    - Inputs: Provides language model completions to "AI Agent".  
    - Outputs: Sends completions to "AI Agent".  
    - Edge Cases: API key invalid, network issues, API rate limits.  
    - Version: 1.1

#### 1.7 Notes and Documentation

- **Overview:**  
  Provides contextual information, setup instructions, API references, and links to external resources via sticky notes.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note6

- **Node Details:**  
  - Sticky notes contain detailed instructions on API usage, local setup with Ollama, model options, and links to DeepSeek and Ollama documentation.  
  - They serve as inline documentation and quick references for users configuring or extending the workflow.  
  - No inputs or outputs; purely informational.  
  - Edge Cases: None (non-executable).

---

### 3. Summary Table

| Node Name               | Node Type                                | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                           |
|-------------------------|----------------------------------------|----------------------------------------|-----------------------------|---------------------------|-----------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger  | Entry point webhook for chat input     | -                           | Basic LLM Chain2           |                                                                                                     |
| Basic LLM Chain2         | @n8n/n8n-nodes-langchain.chainLlm      | Initial simple LLM response             | When chat message received  | Ollama DeepSeek            |                                                                                                     |
| Ollama DeepSeek          | @n8n/n8n-nodes-langchain.lmChatOllama  | Local DeepSeek-R1 reasoning model       | Basic LLM Chain2            | Basic LLM Chain2 (logical) | Sticky Note1: "## DeepSeek with Ollama Local Model"                                                 |
| DeepSeek Raw Body        | n8n-nodes-base.httpRequest              | HTTP raw JSON request to DeepSeek-R1   | - (external or manual)       | -                         | Sticky Note: "## DeepSeek using HTTP Request\n### DeepSeek Reasoner R1\nhttps://api-docs.deepseek.com/\nRaw Body" |
| DeepSeek JSON Body       | n8n-nodes-base.httpRequest              | HTTP JSON request to DeepSeek Chat V3  | - (external or manual)       | -                         | Sticky Note3: "## DeepSeek using HTTP Request\n### DeepSeek Chat V3\nhttps://api-docs.deepseek.com/\nJSON Body" |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent          | Conversational agent with memory       | DeepSeek, Window Buffer Memory | -                       | Sticky Note2: "## DeepSeek Conversational Agent w/Memory"                                           |
| Window Buffer Memory     | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context         | -                           | AI Agent                  |                                                                                                     |
| DeepSeek                 | @n8n/n8n-nodes-langchain.lmChatOpenAi  | DeepSeek Reasoner model via API         | -                           | AI Agent                  | Sticky Note4: "# Your First DeepSeek API Call\n\nThe DeepSeek API uses an API format compatible with OpenAI..." |
| Sticky Note              | n8n-nodes-base.stickyNote               | Documentation and instructions          | -                           | -                         | "# Your First DeepSeek API Call\n\nThe DeepSeek API uses an API format compatible with OpenAI..."    |
| Sticky Note1             | n8n-nodes-base.stickyNote               | Documentation for Ollama local model    | -                           | -                         | "## DeepSeek with Ollama Local Model"                                                               |
| Sticky Note2             | n8n-nodes-base.stickyNote               | Documentation for conversational agent  | -                           | -                         | "## DeepSeek Conversational Agent w/Memory"                                                         |
| Sticky Note3             | n8n-nodes-base.stickyNote               | Documentation for HTTP JSON requests    | -                           | -                         | "## DeepSeek using HTTP Request\n### DeepSeek Chat V3\nhttps://api-docs.deepseek.com/\nJSON Body"    |
| Sticky Note4             | n8n-nodes-base.stickyNote               | API call instructions and parameters    | -                           | -                         | "# Your First DeepSeek API Call\n\nThe DeepSeek API uses an API format compatible with OpenAI..."    |
| Sticky Note5             | n8n-nodes-base.stickyNote               | Additional examples and API key links   | -                           | -                         | "## Four Examples for Connecting to DeepSeek\nhttps://api-docs.deepseek.com/\nhttps://platform.deepseek.com/api_keys" |
| Sticky Note6             | n8n-nodes-base.stickyNote               | Ollama local deployment references      | -                           | -                         | "### Ollama Local\nhttps://ollama.com/\nhttps://ollama.com/library/deepseek-r1"                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `@n8n/n8n-nodes-langchain.chatTrigger` node named "When chat message received".  
   - Use default settings to listen for incoming chat messages.

2. **Create Basic LLM Chain:**  
   - Add `@n8n/n8n-nodes-langchain.chainLlm` node named "Basic LLM Chain2".  
   - Configure with a single system message: "You are a helpful assistant."  
   - Connect output of "When chat message received" to input of "Basic LLM Chain2".

3. **Set Up Ollama Local Model Node:**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOllama` node named "Ollama DeepSeek".  
   - Set model to `deepseek-r1:14b`.  
   - Configure options: format = default, numCtx = 16384, temperature = 0.6.  
   - Set credentials with your local Ollama API (e.g., `http://127.0.0.1`).  
   - Connect output of "Basic LLM Chain2" to input of "Ollama DeepSeek".

4. **Create DeepSeek Reasoner HTTP Raw Request Node:**  
   - Add `n8n-nodes-base.httpRequest` node named "DeepSeek Raw Body".  
   - Set URL to `https://api.deepseek.com/chat/completions`.  
   - Method: POST.  
   - Content-Type: raw JSON (`application/json`).  
   - Body:  
     ```json
     {
       "model": "deepseek-reasoner",
       "messages": [{"role": "user", "content": "{{ $json.chatInput.trim() }}"}],
       "stream": false
     }
     ```  
   - Enable sending body, set authentication to HTTP header with your DeepSeek API key.  
   - Connect as needed depending on your input source.

5. **Create DeepSeek Chat V3 HTTP JSON Request Node:**  
   - Add `n8n-nodes-base.httpRequest` node named "DeepSeek JSON Body".  
   - Set URL to `https://api.deepseek.com/chat/completions`.  
   - Method: POST.  
   - Specify JSON body:  
     ```json
     {
       "model": "deepseek-chat",
       "messages": [
         {"role": "system", "content": "{{ $json.chatInput }}"},
         {"role": "user", "content": "Hello!"}
       ],
       "stream": false
     }
     ```  
   - Enable sending JSON body, set authentication to HTTP header with your DeepSeek API key.

6. **Create Conversational AI Agent:**  
   - Add `@n8n/n8n-nodes-langchain.memoryBufferWindow` node named "Window Buffer Memory".  
   - Use default parameters for conversation window size.  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named "DeepSeek".  
     - Set model to `deepseek-reasoner`.  
     - Use OpenAI API credential type with your DeepSeek API key.  
   - Add `@n8n/n8n-nodes-langchain.agent` node named "AI Agent".  
     - Set agent type to `conversationalAgent`.  
     - Set system message to "You are a helpful assistant."  
     - Enable retry on failure and continue on error.  
     - Connect "Window Buffer Memory" output to "AI Agent" memory input.  
     - Connect "DeepSeek" output to "AI Agent" language model input.

7. **Add Sticky Notes for Documentation:**  
   - Add multiple `n8n-nodes-base.stickyNote` nodes with content from the original workflow for user guidance, API references, and setup instructions.

8. **Configure Credentials:**  
   - Create credentials for DeepSeek API key using HTTP header auth for HTTP Request nodes and OpenAI API credential type for LangChain nodes.  
   - Create Ollama API credentials pointing to your local Ollama server.

9. **Connect Nodes Appropriately:**  
   - Ensure "When chat message received" triggers "Basic LLM Chain2".  
   - "Basic LLM Chain2" output connects to "Ollama DeepSeek".  
   - "Window Buffer Memory" and "DeepSeek" feed into "AI Agent".  
   - HTTP Request nodes can be triggered independently or integrated as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| DeepSeek API uses OpenAI-compatible API format; base URL: https://api.deepseek.com; API keys at https://platform.deepseek.com/api_keys               | https://api-docs.deepseek.com/                          |
| DeepSeek Chat V3 model invoked with `model='deepseek-chat'`; DeepSeek-R1 reasoning model invoked with `model='deepseek-reasoner'`                      | https://api-docs.deepseek.com/                          |
| Ollama local deployment instructions and DeepSeek-R1 model available at https://ollama.com/ and https://ollama.com/library/deepseek-r1                | https://ollama.com/                                     |
| Conversational agent uses window buffer memory for context persistence and includes built-in error handling with retries                              | n8n LangChain agent node documentation                  |
| Workflow demonstrates multiple integration methods: local Ollama, HTTP raw requests, HTTP JSON requests, and LangChain nodes with memory and retries  | Workflow overview and sticky notes                       |
| For API key management and security, use n8n’s credential management and avoid hardcoding keys in nodes                                                | n8n best practices                                      |

---

This document provides a comprehensive understanding of the **🐋DeepSeek V3 Chat & R1 Reasoning Quick Start** workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.