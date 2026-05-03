Generate Recipes from Ingredients with Ollama AI Chef Agent

https://n8nworkflows.xyz/workflows/generate-recipes-from-ingredients-with-ollama-ai-chef-agent-4007


# Generate Recipes from Ingredients with Ollama AI Chef Agent

### 1. Workflow Overview

This n8n workflow, titled **"Generate Recipes from Ingredients with Ollama AI Chef Agent,"** is designed as an AI-powered kitchen assistant. It transforms a list of available ingredients into creative recipe suggestions using an AI language model (Ollama). The workflow is ideal for users seeking meal inspiration based on leftover or on-hand ingredients, with a focus on fun, customization, and simplicity.

The workflow’s logic is grouped into these functional blocks:

- **1.1 Input Reception:** Accepts ingredient data via webhook or chat message.
- **1.2 AI Processing:** Passes input to an AI Agent that uses Ollama Chat Model and memory to generate recipe suggestions and missing ingredients.
- **1.3 Response Delivery:** Returns the AI-generated suggestions back to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives the user's list of ingredients either through an HTTP webhook call or via a chat message trigger, enabling flexible input methods.

**Nodes Involved:**  
- Webhook  
- When chat message received  

**Node Details:**

- **Webhook**  
  - *Type:* n8n-nodes-base.webhook  
  - *Role:* Listens for HTTP POST requests at the path `/lets-cook` by default.  
  - *Configuration:* Path set to `lets-cook`, listens publicly by default.  
  - *Inputs:* External HTTP requests with JSON body containing `"ingredients"`.  
  - *Outputs:* Passes received data to the Chef AI Agent node.  
  - *Edge Cases:* Missing or malformed JSON body, unauthorized requests if not secured.  
  - *Credentials:* None required.  

- **When chat message received**  
  - *Type:* @n8n/n8n-nodes-langchain.chatTrigger  
  - *Role:* Receives chat messages as an alternative input method (e.g., from chat platforms integrated via LangChain).  
  - *Configuration:* Publicly accessible webhook with a replaceable webhook ID.  
  - *Inputs:* Chat message events containing ingredient lists.  
  - *Outputs:* Forwards message content to Chef AI Agent for processing.  
  - *Edge Cases:* Connectivity issues with chat platform, malformed messages.  
  - *Credentials:* None specified; depends on LangChain chat integration setup.  

---

#### 2.2 AI Processing

**Overview:**  
This block orchestrates the AI-driven logic: it uses the Ollama Chat Model and a memory buffer to generate recipe suggestions and missing ingredient recommendations, coordinated by the Chef AI Agent. It also includes a "Think" tool node representing internal AI reasoning.

**Nodes Involved:**  
- Chef AI Agent  
- Ollama Chat Model  
- Simple Memory  
- Think  

**Node Details:**

- **Chef AI Agent**  
  - *Type:* @n8n/n8n-nodes-langchain.agent  
  - *Role:* Central AI agent coordinating the recipe generation logic.  
  - *Configuration:* System message instructs to generate 5 recipe ideas based on provided ingredients and suggest up to 3 missing ingredients to improve dishes.  
  - *Inputs:* Receives triggers from Webhook or chat message nodes.  
  - *Outputs:* Sends response to Respond to Webhook node.  
  - *AI Dependencies:* Uses Ollama Chat Model as language model, Simple Memory as context memory, and Think tool for reasoning.  
  - *Edge Cases:* AI model unavailability, response timeouts, unexpected AI output formats.  
  - *Credentials:* Requires valid Ollama API credentials.  

- **Ollama Chat Model**  
  - *Type:* @n8n/n8n-nodes-langchain.lmChatOllama  
  - *Role:* Provides AI language model capabilities via Ollama API calls.  
  - *Configuration:* Default parameters, connected to Ollama API credentials for authentication.  
  - *Inputs:* Requests from Chef AI Agent node.  
  - *Outputs:* Returns AI-generated text to Chef AI Agent.  
  - *Edge Cases:* Ollama service offline, API errors, rate limits.  
  - *Credentials:* Ollama API credentials must be configured.  

- **Simple Memory**  
  - *Type:* @n8n/n8n-nodes-langchain.memoryBufferWindow  
  - *Role:* Maintains a context window of prior dialogue or AI interactions to provide memory for the agent.  
  - *Configuration:* Context window length set to 5000 tokens/characters.  
  - *Inputs:* Connected as AI memory input to Chef AI Agent.  
  - *Outputs:* Provides stored context back to agent to maintain conversational continuity.  
  - *Edge Cases:* Memory overflow, loss of context if node fails.  

- **Think**  
  - *Type:* @n8n/n8n-nodes-langchain.toolThink  
  - *Role:* AI tool node that simulates internal reasoning or complex thought processes to aid the agent.  
  - *Configuration:* Default, no parameters set.  
  - *Inputs:* Connected as AI tool input for Chef AI Agent.  
  - *Outputs:* Results feed back into Chef AI Agent processing.  
  - *Edge Cases:* Processing delays or failures.  

---

#### 2.3 Response Delivery

**Overview:**  
This block returns the AI-generated recipe and ingredient suggestions as an HTTP response to the original webhook call.

**Nodes Involved:**  
- Respond to Webhook  

**Node Details:**

- **Respond to Webhook**  
  - *Type:* n8n-nodes-base.respondToWebhook  
  - *Role:* Sends back the final response payload to the client that triggered the webhook.  
  - *Configuration:* Default settings; automatically responds with data from previous node (Chef AI Agent).  
  - *Inputs:* Receives AI-generated suggestions from Chef AI Agent.  
  - *Outputs:* HTTP response to the webhook caller.  
  - *Edge Cases:* Client disconnects before response, malformed response data.  

---

### 3. Summary Table

| Node Name                 | Node Type                                    | Functional Role                     | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                         |
|---------------------------|----------------------------------------------|-----------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger         | Receives chat message input       | —                            | Chef AI Agent               |                                                                                                                     |
| Ollama Chat Model          | @n8n/n8n-nodes-langchain.lmChatOllama        | AI language model interface       | Chef AI Agent (AI LanguageModel) | Chef AI Agent               | Requires Ollama API credentials configured for local or remote Ollama instance                                      |
| Simple Memory              | @n8n/n8n-nodes-langchain.memoryBufferWindow  | Maintains AI conversational memory| —                            | Chef AI Agent (AI Memory)   |                                                                                                                     |
| Think                      | @n8n/n8n-nodes-langchain.toolThink            | Provides AI reasoning tool        | —                            | Chef AI Agent (AI Tool)     |                                                                                                                     |
| Chef AI Agent              | @n8n/n8n-nodes-langchain.agent                | Core AI logic for recipe generation | Webhook, When chat message received, Ollama Chat Model, Simple Memory, Think | Respond to Webhook          | System message config: limit 5 recipe ideas, suggest up to 3 missing ingredients                                     |
| Webhook                   | n8n-nodes-base.webhook                        | Receives HTTP POST ingredient input | —                            | Chef AI Agent               | Default webhook path `/lets-cook`; accepts JSON with `"ingredients"` key                                           |
| Respond to Webhook        | n8n-nodes-base.respondToWebhook               | Sends HTTP response with AI output| Chef AI Agent                | —                           | Responds with generated recipes or fallback message                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - Parameters: Set path to `lets-cook` (or desired endpoint).  
   - Purpose: To receive POST requests containing JSON with `ingredients` field.  

2. **Create “When chat message received” Node**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Parameters: Leave public, set webhook ID after creation.  
   - Purpose: Alternative input via chat messages.  

3. **Create Ollama Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOllama`  
   - Parameters: Default options.  
   - Credentials: Set up Ollama API credentials pointing to your local or remote Ollama AI instance.  

4. **Create Simple Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Parameters: Context window length set to `5000`.  

5. **Create Think Node**  
   - Type: `@n8n/n8n-nodes-langchain.toolThink`  
   - Parameters: Default settings.  

6. **Create Chef AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - System Message:  
       ```
       You are a helpful cooking assistant. You will receive items that are available to make food with. Provide a list of 5 items to make. If there are recipes that include Ingredients that are not provided in the beginning, advise the user that these are ingredients that would be great to add in. Limit to 3 missing ingredients.
       ```  
   - Connect the following as inputs:  
     - AI Language Model input → Ollama Chat Model  
     - AI Memory input → Simple Memory  
     - AI Tool input → Think  
     - Main input → Webhook and When chat message received nodes  
   - Output: Connect main output to Respond to Webhook node.  

7. **Create Respond to Webhook Node**  
   - Type: `Respond to Webhook`  
   - Parameters: Default.  
   - Input: Connect from Chef AI Agent node’s main output.  
   - Purpose: To send the AI-generated recipes and missing ingredients back to the HTTP client.  

8. **Connect Nodes**  
   - Webhook → Chef AI Agent (main input)  
   - When chat message received → Chef AI Agent (main input)  
   - Ollama Chat Model → Chef AI Agent (ai_languageModel input)  
   - Simple Memory → Chef AI Agent (ai_memory input)  
   - Think → Chef AI Agent (ai_tool input)  
   - Chef AI Agent → Respond to Webhook (main output)  

9. **Set Credentials**  
   - Configure Ollama API credentials with valid endpoint and authentication details.  

10. **Activate Workflow**  
    - Enable the workflow.  
    - Test by sending a POST request to your webhook with JSON payload, e.g.:  
      ```json
      {
        "ingredients": "eggs, rice, spinach"
      }
      ```  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| “Powered by Clown Mutiny’s taste-bud liberation division.”                                  | Branding flair included in workflow description                                                       |
| Workflow is a great introduction to combining AI and automation using n8n and Ollama AI.    | Useful for beginners aiming to integrate LLMs with workflow automation                                |
| Ollama AI must be running locally or accessible remotely for the Ollama Chat Model node.    | https://ollama.com/                                                                                     |
| Webhook path defaults to `/lets-cook`, but can be customized as needed.                     | Adjust in Webhook node parameters                                                                      |
| To test via HTTP, use tools like Postman or curl to POST JSON payload to webhook URL.        | Example payload: `{"ingredients":"eggs, rice, spinach"}`                                              |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.