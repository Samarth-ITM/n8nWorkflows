Find step-by-step recipes from API Ninjas Recipe API in an n8n AI agent

https://n8nworkflows.xyz/workflows/find-step-by-step-recipes-from-api-ninjas-recipe-api-in-an-n8n-ai-agent-7837


# Find step-by-step recipes from API Ninjas Recipe API in an n8n AI agent

### 1. Workflow Overview

This workflow implements an AI-powered chat agent designed to find and provide step-by-step cooking recipes by querying the API Ninjas Recipe API. It targets use cases such as conversational recipe assistance, enabling users to request recipes in natural language and receive clear, structured ingredient lists and instructions.

The workflow’s logic is organized into the following functional blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users.
- **1.2 Context Memory Management:** Maintains recent conversation context to provide continuity.
- **1.3 AI Decision & Routing:** Uses an AI agent to interpret user requests and decide when to invoke tools.
- **1.4 Recipe Retrieval Tool:** Queries the external Recipe API to fetch recipe data.
- **1.5 AI Response Generation:** Generates a natural language response to send back to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming chat messages and triggers the workflow for each new message.

- **Nodes Involved:**  
  - Chat Trigger - Receive Message

- **Node Details:**

  - **Chat Trigger - Receive Message**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Entry point webhook node that receives chat messages and starts the workflow.  
    - Configuration: Default webhook ID is set to receive messages; no additional options configured.  
    - Inputs: External HTTP webhook calls containing user messages.  
    - Outputs: Forwards the incoming message data to the next node (AI Agent).  
    - Version: 1.3  
    - Potential Failures: Webhook unavailability, malformed requests, or network issues.

---

#### 1.2 Context Memory Management

- **Overview:**  
  Maintains a short-term memory buffer of recent conversation messages to provide context for the AI agent's reasoning.

- **Nodes Involved:**  
  - Memory - Recent Messages (Window)

- **Node Details:**

  - **Memory - Recent Messages (Window)**  
    - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
    - Role: Provides a sliding window memory buffer to keep recent messages in context for the AI agent.  
    - Configuration: Default settings used; no custom parameters specified.  
    - Inputs: Receives current message from the Chat Trigger node.  
    - Outputs: Provides memory context to the AI Agent node.  
    - Version: 1.3  
    - Edge Cases: Excessive message lengths may cause context truncation; memory window size is fixed and may not cover very long conversations.

---

#### 1.3 AI Decision & Routing

- **Overview:**  
  The AI agent interprets the user's message, decides if a recipe tool should be used based on the system message hint, and routes the request accordingly.

- **Nodes Involved:**  
  - AI Agent - Route to Tools

- **Node Details:**

  - **AI Agent - Route to Tools**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Acts as an orchestrator AI agent that decides which tool to invoke based on user input and system instructions.  
    - Configuration:  
      - System Message: "Always use the recipe tool if I ask you for a recipe" — this directs the agent to call the recipe tool on relevant queries.  
    - Inputs:  
      - Receives chat message (main input)  
      - Receives memory context (ai_memory input)  
      - Receives output from the LLM node (ai_languageModel input)  
      - Receives output from the Recipe Tool node (ai_tool input)  
    - Outputs: Forwards calls to the OpenAI LLM or Recipe Tool as needed, and returns the final message reply.  
    - Version: 2.2  
    - Edge Cases:  
      - Misinterpretation of user intent could lead to incorrect tool usage.  
      - Timeout or API limits on called tools may cause failures.  
      - Expression errors if unexpected data formats are returned.  

---

#### 1.4 Recipe Retrieval Tool

- **Overview:**  
  This block performs the actual recipe data retrieval by querying the API Ninjas Recipe API with the user’s food query.

- **Nodes Involved:**  
  - Recipe Tool - Fetch from API Ninjas

- **Node Details:**

  - **Recipe Tool - Fetch from API Ninjas**  
    - Type: `n8n-nodes-base.httpRequestTool`  
    - Role: Makes an authenticated HTTP request to the API Ninjas recipe endpoint to fetch recipe data.  
    - Configuration:  
      - URL: `https://api.api-ninjas.com/v1/recipe`  
      - Method: GET (default for HTTP request tool)  
      - Authentication: HTTP Header Auth using API Ninjas API key credential.  
      - Query Parameters:  
        - `query` parameter dynamically set from AI agent’s parameter override expression, extracting the food item requested.  
      - Tool Description: "Use the query parameter to specify the food, and it will return a recipe"  
    - Inputs: Receives tool call from AI Agent node.  
    - Outputs: Returns recipe data (ingredients and instructions) back to AI Agent.  
    - Version: 4.2  
    - Edge Cases:  
      - Authentication failures if API key is invalid or expired.  
      - Empty or no results if query is ambiguous or unsupported food.  
      - API rate limits or downtime.  

---

#### 1.5 AI Response Generation

- **Overview:**  
  Uses OpenAI GPT-5-mini model to generate natural language responses based on the AI agent’s instructions and API data.

- **Nodes Involved:**  
  - LLM - OpenAI Chat

- **Node Details:**

  - **LLM - OpenAI Chat**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Calls the OpenAI GPT-5-mini model to process messages and generate replies.  
    - Configuration:  
      - Model: GPT-5-mini (selected from a list, cached for efficiency)  
      - No additional options specified.  
      - Credentials: Uses OpenAI API key configured in `OpenAi account`.  
    - Inputs: Receives prompts and context from AI Agent node.  
    - Outputs: Returns language model responses to AI Agent.  
    - Version: 1.2  
    - Edge Cases:  
      - API quota exhaustion or invalid key.  
      - Model response latency or timeouts.  
      - Unexpected output format or hallucinations.

---

### 3. Summary Table

| Node Name                       | Node Type                                  | Functional Role            | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                              |
|--------------------------------|--------------------------------------------|----------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Chat Trigger - Receive Message  | @n8n/n8n-nodes-langchain.chatTrigger       | Input Reception            | External webhook              | AI Agent - Route to Tools    |                                                                                                        |
| Memory - Recent Messages (Window) | @n8n/n8n-nodes-langchain.memoryBufferWindow | Context Memory Management  | Chat Trigger - Receive Message | AI Agent - Route to Tools    |                                                                                                        |
| AI Agent - Route to Tools       | @n8n/n8n-nodes-langchain.agent              | AI Decision & Routing      | Chat Trigger - Receive Message, Memory - Recent Messages, LLM - OpenAI Chat, Recipe Tool | LLM - OpenAI Chat, Recipe Tool - Fetch from API Ninjas |                                                                                                        |
| LLM - OpenAI Chat               | @n8n/n8n-nodes-langchain.lmChatOpenAi       | AI Response Generation     | AI Agent - Route to Tools     | AI Agent - Route to Tools    |                                                                                                        |
| Recipe Tool - Fetch from API Ninjas | n8n-nodes-base.httpRequestTool               | Recipe Retrieval Tool      | AI Agent - Route to Tools     | AI Agent - Route to Tools    |                                                                                                        |
| Workflow description            | n8n-nodes-base.stickyNote                    | Documentation Note         |                              |                             | # Workflow description  A small AI agent that answers chat messages and calls a recipe tool when you ask for a recipe. Setup instructions included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Chat Trigger node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `Chat Trigger - Receive Message`  
   - Configuration: Use default webhook settings; enable webhook to receive chat messages.

3. **Add a Memory Buffer Window node:**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Name: `Memory - Recent Messages (Window)`  
   - Configuration: Use default parameters to maintain recent conversation context.

4. **Add an AI Agent node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `AI Agent - Route to Tools`  
   - Configuration:  
     - Set the system message to: "Always use the recipe tool if I ask you for a recipe"  
     - This guides the agent to invoke the recipe tool on recipe-related queries.

5. **Add an OpenAI Chat node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `LLM - OpenAI Chat`  
   - Configuration:  
     - Select the GPT-5-mini model from the list.  
     - Credentials: configure OpenAI API key credentials (named `OpenAi account`).  
     - Leave other options default.

6. **Add an HTTP Request Tool node:**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - Name: `Recipe Tool - Fetch from API Ninjas`  
   - Configuration:  
     - Set URL to: `https://api.api-ninjas.com/v1/recipe`  
     - Method: GET (default)  
     - Authentication: HTTP Header Auth, use API Ninjas API key credentials configured in n8n (named `API Ninjas Credential`).  
     - Query Parameters: Add `query` parameter with value:  
       ```  
       ={{ $fromAI('parameters0_Value', ``, 'string') }}  
       ```  
       This expression dynamically extracts the recipe query from the AI agent parameters.  
     - Tool Description: "Use the query parameter to specify the food, and it will return a recipe"

7. **Connect the nodes:**  
   - Connect `Chat Trigger - Receive Message` main output to `AI Agent - Route to Tools` main input.  
   - Connect `Memory - Recent Messages (Window)` output to `AI Agent - Route to Tools` ai_memory input.  
   - Connect `LLM - OpenAI Chat` ai_languageModel output to `AI Agent - Route to Tools` ai_languageModel input.  
   - Connect `Recipe Tool - Fetch from API Ninjas` ai_tool output to `AI Agent - Route to Tools` ai_tool input.  
   - Connect `AI Agent - Route to Tools` outputs to `LLM - OpenAI Chat` and `Recipe Tool - Fetch from API Ninjas` as needed (managed internally by the agent node).

8. **Credentials setup:**  
   - Add valid OpenAI API key credential with name `OpenAi account`.  
   - Add valid API Ninjas API key credential with name `API Ninjas Credential`.

9. **Testing the workflow:**  
   - Start the workflow.  
   - Send a chat message via the webhook with a recipe request, e.g., "Find me a pasta recipe".  
   - The AI Agent should interpret the request, call the recipe tool, and return a clean, formatted list of ingredients and cooking steps.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| A small AI agent that answers chat messages and calls a recipe tool when you ask for a recipe. Setup involves adding OpenAI and API Ninjas API keys to respective nodes. | Workflow description sticky note within the workflow itself                                                                                 |
| Try example user input: "find me a pasta recipe". The agent should respond with ingredients and steps fetched from API Ninjas.                                            | Usage instruction included in workflow description sticky note                                                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.