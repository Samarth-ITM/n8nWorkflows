🔐🦙Private & Local Ollama Self-Hosted + Dynamic LLM Router

https://n8nworkflows.xyz/workflows/----private---local-ollama-self-hosted---dynamic-llm-router-3139


# 🔐🦙Private & Local Ollama Self-Hosted + Dynamic LLM Router

### 1. Workflow Overview

This workflow, titled **"🔐🦙🤖 Private & Local Ollama Self-Hosted + Dynamic LLM Router"**, is designed for AI enthusiasts, developers, and privacy-conscious users who want to leverage multiple local Ollama large language models (LLMs) without sending data externally. It intelligently routes user prompts to the most suitable local Ollama model based on the nature of the request, enabling optimized performance and privacy.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users.
- **1.2 LLM Routing Decision:** Analyzes the user prompt to select the best-suited Ollama model dynamically.
- **1.3 Router Conversation Memory:** Maintains context and conversation history for the routing decision process.
- **1.4 Dynamic Ollama Model Invocation:** Calls the selected Ollama LLM model to generate a response.
- **1.5 Agent Conversation Memory:** Maintains conversation context for the final AI agent responding to the user.
- **1.6 Output Delivery:** Returns the AI-generated response to the user.

This structure ensures seamless, privacy-preserving orchestration of multiple specialized local LLMs, including text-only, code-specific, and vision-capable models.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users, triggering the workflow to start processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point for user chat input  
    - Configuration: Default options, webhook enabled for receiving chat messages  
    - Inputs: External user chat messages via webhook  
    - Outputs: Passes chat input to the LLM Router node  
    - Edge Cases: Webhook connectivity issues, malformed chat input  
    - Version: 1.1

---

#### 2.2 LLM Routing Decision

- **Overview:**  
  This block analyzes the user prompt and selects the most appropriate Ollama LLM model based on a detailed decision framework considering task complexity, modality (text or image), language needs, and coding requirements.

- **Nodes Involved:**  
  - LLM Router  
  - Router Chat Memory  
  - Ollama phi4 (fallback or example model)

- **Node Details:**  
  - **LLM Router**  
    - Type: LangChain Agent  
    - Role: Core decision-making agent that classifies user prompts and selects the optimal LLM model  
    - Configuration:  
      - System message defines the role as an expert LLM router  
      - Detailed classification rules listing available models and their specialties  
      - Decision tree logic to route based on prompt content (image presence, complexity, multilingual needs, coding)  
      - Output format: JSON object with selected model name and reasoning  
    - Key Expressions: Uses `{{$json.chatInput}}` to access user prompt  
    - Inputs: Receives chat input from "When chat message received" node  
    - Outputs: JSON with selected LLM model and reason, passed to "AI Agent with Dynamic LLM"  
    - Edge Cases: Ambiguous or harmful requests trigger fallback model or ethical refusal message  
    - Version: 1.7

  - **Router Chat Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversation context for the routing decision to improve consistency  
    - Configuration: Default session management  
    - Inputs: Connected to LLM Router node as memory source  
    - Outputs: Provides memory context for LLM Router  
    - Edge Cases: Memory overflow or session key mismatches  
    - Version: 1.3

  - **Ollama phi4**  
    - Type: LangChain Ollama Chat Model  
    - Role: Example or fallback lightweight Ollama model for simple tasks  
    - Configuration: Model set to "phi4:latest", output format JSON  
    - Credentials: Uses local Ollama API at 127.0.0.1:11434  
    - Inputs: Connected as an AI language model for LLM Router (not directly used in routing decision)  
    - Outputs: Provides model responses if invoked  
    - Edge Cases: Ollama API connection failure, model not pulled locally  
    - Version: 1

---

#### 2.3 Dynamic Ollama Model Invocation

- **Overview:**  
  This block invokes the Ollama LLM dynamically selected by the router to generate the AI agent's response to the user prompt.

- **Nodes Involved:**  
  - Ollama Dynamic LLM  
  - AI Agent with Dynamic LLM  
  - Agent Chat Memory

- **Node Details:**  
  - **Ollama Dynamic LLM**  
    - Type: LangChain Ollama Chat Model  
    - Role: Executes the selected Ollama model dynamically based on router output  
    - Configuration: Model name is dynamically set via expression from LLM Router output JSON (`={{ $('LLM Router').item.json.output.parseJson().llm }}`)  
    - Credentials: Local Ollama API credentials (127.0.0.1:11434)  
    - Inputs: Connected as AI language model for "AI Agent with Dynamic LLM"  
    - Outputs: Provides AI-generated response text  
    - Edge Cases: Model name invalid or not pulled locally, Ollama API downtime  
    - Version: 1

  - **AI Agent with Dynamic LLM**  
    - Type: LangChain Agent  
    - Role: Final AI agent that uses the dynamically selected Ollama model to answer the user's prompt  
    - Configuration:  
      - Text input is the original user chat input from "When chat message received" node  
      - No additional system message configured (empty)  
    - Inputs: Receives chat input and AI language model from "Ollama Dynamic LLM"  
    - Outputs: Final AI response to be sent back to user  
    - Edge Cases: Empty or malformed user input, AI model errors  
    - Version: 1.7

  - **Agent Chat Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversation memory for the AI agent to ensure coherent multi-turn interactions  
    - Configuration: Uses custom session key from user session ID (`={{ $('When chat message received').item.json.sessionId }}`)  
    - Inputs: Connected as memory for "AI Agent with Dynamic LLM"  
    - Outputs: Provides conversation context for AI agent  
    - Edge Cases: Session ID missing or inconsistent, memory overflow  
    - Version: 1.3

---

#### 2.4 Output Delivery

- **Overview:**  
  The final AI-generated response is returned to the user through the chat interface (implicit in the chat trigger node’s webhook response).

- **Nodes Involved:**  
  - (No explicit output node; response flows back through the chat trigger mechanism)

- **Node Details:**  
  - The "AI Agent with Dynamic LLM" node’s output is implicitly sent back as the chat response via the LangChain chat trigger webhook.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                              | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                     |
|-------------------------|----------------------------------|----------------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger           | Entry point for user chat input               | (external webhook)            | LLM Router                   |                                                                                                |
| LLM Router              | LangChain Agent                   | Analyzes prompt and selects optimal LLM      | When chat message received, Router Chat Memory | AI Agent with Dynamic LLM     | ## Ollama LLM Router Based on User Prompt: This agent chooses the Ollama LLM dynamically based on user prompt |
| Router Chat Memory      | LangChain Memory Buffer Window   | Maintains conversation memory for router      | LLM Router                   | LLM Router                   | ## Router Chat Memory                                                                          |
| Ollama phi4             | LangChain Ollama Chat Model      | Example/fallback lightweight Ollama model     | (none, linked as AI model)   | LLM Router                   | ## Ollama LLM                                                                                  |
| Ollama Dynamic LLM      | LangChain Ollama Chat Model      | Dynamically invokes selected Ollama model     | (none, linked as AI model)   | AI Agent with Dynamic LLM     | ## Dynamic Ollama LLM                                                                         |
| AI Agent with Dynamic LLM | LangChain Agent                 | Final AI agent responding using selected LLM | LLM Router, Ollama Dynamic LLM, Agent Chat Memory | (implicit output to user)     | ## AI Agent using Dynamic Local Ollama LLM: This agent uses the Ollama LLM based on router choice to answer user prompt |
| Agent Chat Memory       | LangChain Memory Buffer Window   | Maintains conversation memory for AI agent    | AI Agent with Dynamic LLM     | AI Agent with Dynamic LLM     | ## Agent Chat Memory                                                                          |
| Sticky Note             | Sticky Note                      | Informational / branding                       | (none)                      | (none)                      | # 🔐🦙🤖 Private & Local Ollama Self-Hosted + Dynamic LLM Router                                |
| Sticky Note1            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## Ollama LLM                                                                                |
| Sticky Note2            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## 👍Try Me!                                                                                |
| Sticky Note3            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## Ollama LLM Router Based on User Prompt                                                    |
| Sticky Note4            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## Router Chat Memory                                                                        |
| Sticky Note5            | Sticky Note                      | Informational / detailed workflow description | (none)                      | (none)                      | ## Who is this for? This workflow template is designed for AI enthusiasts, developers, and privacy-conscious users... (full description) |
| Sticky Note7            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## AI Agent using Dynamic Local Ollama LLM                                                  |
| Sticky Note8            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## Dynamic Ollama LLM                                                                       |
| Sticky Note9            | Sticky Note                      | Informational                                 | (none)                      | (none)                      | ## Agent Chat Memory                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **LangChain Chat Trigger** node named **"When chat message received"**.  
   - Configure with default options and enable webhook to receive chat messages.

2. **Add Router Chat Memory:**  
   - Add a **LangChain Memory Buffer Window** node named **"Router Chat Memory"**.  
   - Use default settings to maintain routing conversation context.

3. **Add the LLM Router Agent:**  
   - Add a **LangChain Agent** node named **"LLM Router"**.  
   - Set **Prompt Type** to "define".  
   - Configure the **text** parameter with the expression:  
     ```
     =Choose the most appropriate LLM model for the following user request. Analyze the task requirements carefully and select the model that will provide optimal performance.  Only choose from the provided list.

     <user_input>
     {{ $json.chatInput }}
     </user_input>
     ```
   - In **Options > System Message**, paste the detailed system prompt defining:  
     - Role as expert LLM router  
     - Classification rules listing models: qwq, llama3.2, phi4, qwen2.5-coder:14b, granite3.2-vision, llama3.2-vision  
     - Decision tree logic for model selection  
     - Examples of user inputs and model selections  
     - Error handling instructions  
     - Output format as JSON with keys "llm" and "reason"  
   - Enable **Output Parser** to parse JSON output.  
   - Connect **"When chat message received"** node main output to **LLM Router** main input.  
   - Connect **Router Chat Memory** node as AI memory input to **LLM Router**.

4. **Add Ollama phi4 Model Node (Optional):**  
   - Add a **LangChain Ollama Chat Model** node named **"Ollama phi4"**.  
   - Set model to `"phi4:latest"`.  
   - Set output format to JSON.  
   - Configure credentials with local Ollama API (default: http://127.0.0.1:11434).  
   - Connect as AI language model input to **LLM Router** (optional, for fallback or testing).

5. **Add Dynamic Ollama Model Node:**  
   - Add a **LangChain Ollama Chat Model** node named **"Ollama Dynamic LLM"**.  
   - Set model name dynamically using expression:  
     ```
     ={{ $('LLM Router').item.json.output.parseJson().llm }}
     ```  
   - Use default options.  
   - Configure credentials with local Ollama API.  
   - Connect as AI language model input to **AI Agent with Dynamic LLM**.

6. **Add Agent Chat Memory:**  
   - Add a **LangChain Memory Buffer Window** node named **"Agent Chat Memory"**.  
   - Set **Session Key** to:  
     ```
     ={{ $('When chat message received').item.json.sessionId }}
     ```  
   - Set **Session ID Type** to "customKey".  
   - Connect as AI memory input to **AI Agent with Dynamic LLM**.

7. **Add AI Agent with Dynamic LLM:**  
   - Add a **LangChain Agent** node named **"AI Agent with Dynamic LLM"**.  
   - Set **Prompt Type** to "define".  
   - Set **text** parameter to:  
     ```
     ={{ $('When chat message received').item.json.chatInput }}
     ```  
   - Leave system message empty.  
   - Connect main input from **LLM Router** output.  
   - Connect AI language model input from **Ollama Dynamic LLM**.  
   - Connect AI memory input from **Agent Chat Memory**.

8. **Connect Workflow:**  
   - Connect **"When chat message received"** main output to **LLM Router** main input.  
   - Connect **LLM Router** main output to **AI Agent with Dynamic LLM** main input.  
   - Connect **Router Chat Memory** AI memory output to **LLM Router** AI memory input.  
   - Connect **Ollama phi4** AI language model output to **LLM Router** AI language model input (optional).  
   - Connect **Ollama Dynamic LLM** AI language model output to **AI Agent with Dynamic LLM** AI language model input.  
   - Connect **Agent Chat Memory** AI memory output to **AI Agent with Dynamic LLM** AI memory input.

9. **Credential Setup:**  
   - Create and configure **Ollama API credentials** in n8n pointing to your local Ollama instance (default URL: http://127.0.0.1:11434).  
   - Assign these credentials to all Ollama model nodes.

10. **Testing and Activation:**  
    - Pull required Ollama models locally using Ollama CLI (e.g., `ollama pull phi4`).  
    - Activate the workflow.  
    - Test by sending chat messages to the webhook endpoint; observe dynamic routing and responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                               |
|---------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow demonstrates how n8n can orchestrate multiple local LLMs for privacy-preserving AI applications with dynamic routing. | Workflow purpose and design                    |
| Ensure Ollama is installed and running locally; pull required models before use.                                                     | https://ollama.ai/                             |
| Customize the routing logic by editing the system prompt in the LLM Router node to fit your specific model collection and use cases. | Customization instructions                      |
| The workflow maintains separate conversation memories for routing decisions and final AI agent responses to ensure context consistency. | Design note                                    |
| The Ollama API default endpoint is `http://127.0.0.1:11434`; ensure this matches your local setup and credentials in n8n.             | Credential configuration                        |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and customizing the **Private & Local Ollama Self-Hosted + Dynamic LLM Router** workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and automation agents to work effectively with this privacy-focused AI orchestration system.