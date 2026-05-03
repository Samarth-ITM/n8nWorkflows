High-Speed AI Chat with OpenAI's gpt-oss-120B Model via Cerebras Inference

https://n8nworkflows.xyz/workflows/high-speed-ai-chat-with-openai-s-gpt-oss-120b-model-via-cerebras-inference-7651


# High-Speed AI Chat with OpenAI's gpt-oss-120B Model via Cerebras Inference

---

### 1. Workflow Overview

This workflow enables a high-speed AI chat interaction using OpenAI's GPT-OSS-120B model, accessed via Cerebras AI's inference API. It is designed to receive chat messages, forward them to the Cerebras-hosted large language model for processing, and return the model's response to the user. The workflow is organized into three logical blocks:

- **1.1 Input Reception:** Listens for incoming chat messages via an n8n LangChain chat trigger node.
- **1.2 API Key Injection:** Adds the required API key to authenticate requests to the Cerebras API.
- **1.3 AI Processing and Output:** Sends the chat input to the Cerebras inference endpoint, processes the response, and formats the output for return.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow by capturing incoming chat messages from users. It serves as the entry point triggering subsequent processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  

  - **When chat message received**  
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chatTrigger` — Listens for chat input events.  
    - *Configuration:* Uses default options, no additional parameters configured. It waits for user chat input to trigger the workflow.  
    - *Key Variables:* Outputs the user message under `chatInput` in the JSON payload.  
    - *Input/Output Connections:* No input (trigger node). Outputs to the "Set API Key" node.  
    - *Version Requirements:* Uses typeVersion 1.3 of the LangChain chat trigger node.  
    - *Potential Failures:* Webhook connection issues or malformed incoming data could prevent triggering.  
    - *Sub-workflow:* N/A

#### 1.2 API Key Injection

- **Overview:**  
  This block injects the necessary authorization key into the workflow data to authenticate API calls to Cerebras. It separates credential management from API request logic.

- **Nodes Involved:**  
  - Set API Key  
  - Sticky Note (API key guidance)

- **Node Details:**  

  - **Set API Key**  
    - *Type and Technical Role:* `n8n-nodes-base.set` — Assigns static values to workflow data fields.  
    - *Configuration:* Sets a string variable `apiKey` with the placeholder value `your-api-key`. This must be replaced by the user with an actual Cerebras API key.  
    - *Key Variables:* Adds `apiKey` to the JSON data used downstream.  
    - *Input/Output Connections:* Input from chat trigger node; output to the HTTP request node.  
    - *Version Requirements:* Uses typeVersion 3.4.  
    - *Potential Failures:* Missing or invalid API key will cause authentication errors in subsequent HTTP requests.  
    - *Sub-workflow:* N/A  

  - **Sticky Note**  
    - *Content:*  
      ```
      ## Set API key 
      **Create an account and [get your Cerebras key](https://cerebras.ai)**
      ```  
    - *Purpose:* Provides user guidance on acquiring a valid Cerebras API key.  
    - *Nodes Covered:* Set API Key node.  

#### 1.3 AI Processing and Output

- **Overview:**  
  This block sends the user’s chat message to the Cerebras GPT-OSS-120B model endpoint, retrieves the completion response, and formats it for return to the user.

- **Nodes Involved:**  
  - Cerebras endpoint  
  - Return Output  
  - Sticky Note1 (parameter guidance)

- **Node Details:**  

  - **Cerebras endpoint**  
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — Performs the API call to the Cerebras inference endpoint.  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://api.cerebras.ai/v1/chat/completions`  
      - Headers: Authorization bearer token set dynamically from `apiKey` variable  
      - Body (JSON):  
        - Model: `gpt-oss-120b`  
        - Stream: false (no streaming responses)  
        - Messages: array containing one user message with content dynamically injected from the chat input (`{{ $('When chat message received').item.json.chatInput }}`)  
        - Temperature: 0 (deterministic output)  
        - max_completion_tokens: -1 (no token limit)  
        - seed: 0  
        - top_p: 1  
      - Redirect handling enabled  
    - *Key Variables:* Uses expression to extract user message and API key dynamically.  
    - *Input/Output Connections:* Input from Set API Key node; output to Return Output node.  
    - *Version Requirements:* Uses typeVersion 4.2.  
    - *Potential Failures:*  
      - Authentication errors if API key is invalid or missing.  
      - Network or timeout issues contacting Cerebras API.  
      - Malformed responses or unexpected JSON structure.  
      - Rate limiting or quota exceeded errors.  
    - *Sub-workflow:* N/A  

  - **Return Output**  
    - *Type and Technical Role:* `n8n-nodes-base.set` — Extracts and formats the AI response for output.  
    - *Configuration:* Assigns a new `output` string variable from the API response path: `{{$json.choices[0].message.content}}`.  
    - *Key Variables:* Extracts chat completion text from the Cerebras API response JSON.  
    - *Input/Output Connections:* Input from Cerebras endpoint node; output is the final workflow return value.  
    - *Version Requirements:* Uses typeVersion 3.4.  
    - *Potential Failures:* If the response structure changes or no choices are returned, this node may fail or produce empty output.  
    - *Sub-workflow:* N/A  

  - **Sticky Note1**  
    - *Content:*  
      ```
      ## Parameters
      Set your endpoint parameters as needed
      - temperature
      - completion tokens
      - top P
      - reasoning effort

      [Review the endpoint](https://inference-docs.cerebras.ai/api-reference/chat-completions)
      ```  
    - *Purpose:* Provides guidance on configurable parameters for the Cerebras chat completions API.  
    - *Nodes Covered:* Cerebras endpoint node.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                      | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                         |
|---------------------------|----------------------------------|------------------------------------|-----------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Receive chat input trigger          | -                           | Set API Key               |                                                                                                   |
| Set API Key               | n8n-nodes-base.set               | Inject API key for authorization   | When chat message received  | Cerebras endpoint         | ## Set API key \n**Create an account and [get your Cerebras key](https://cerebras.ai)**          |
| Cerebras endpoint         | n8n-nodes-base.httpRequest       | Call Cerebras GPT-OSS-120B API     | Set API Key                 | Return Output             | ## Parameters\nSet your endpoint parameters as needed\n- temperature\n- completion tokens\n- top P\n- reasoning effort\n\n[Review the endpoint](https://inference-docs.cerebras.ai/api-reference/chat-completions) |
| Return Output             | n8n-nodes-base.set               | Extract and format AI response     | Cerebras endpoint           | -                        |                                                                                                   |
| Sticky Note               | n8n-nodes-base.stickyNote        | Guidance on API key acquisition    | -                           | -                        | ## Set API key \n**Create an account and [get your Cerebras key](https://cerebras.ai)**          |
| Sticky Note1              | n8n-nodes-base.stickyNote        | Guidance on endpoint parameters     | -                           | -                        | ## Parameters\nSet your endpoint parameters as needed\n- temperature\n- completion tokens\n- top P\n- reasoning effort\n\n[Review the endpoint](https://inference-docs.cerebras.ai/api-reference/chat-completions) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Add node `@n8n/n8n-nodes-langchain.chatTrigger` named "When chat message received".  
   - Use default settings with no additional parameters. This node will listen for chat messages to trigger the workflow.

2. **Create the Set API Key Node:**  
   - Add node `Set` named "Set API Key".  
   - Configure it to assign a new string variable `apiKey` with value `your-api-key`.  
   - This node injects the Cerebras API key for authentication. Replace `your-api-key` with your actual key.

3. **Connect Trigger to Set API Key:**  
   - Link the output of "When chat message received" to the input of "Set API Key".

4. **Create the HTTP Request Node for Cerebras Endpoint:**  
   - Add node `HTTP Request` named "Cerebras endpoint".  
   - Configure as follows:  
     - HTTP Method: POST  
     - URL: `https://api.cerebras.ai/v1/chat/completions`  
     - Headers: Add header `Authorization` with value expression: `Bearer {{$json.apiKey}}`  
     - Body Content Type: JSON  
     - Body: Use the JSON editor with this content:  
       ```json
       {
         "model": "gpt-oss-120b",
         "stream": false,
         "messages": [
           {
             "content": "{{ $('When chat message received').item.json.chatInput }}",
             "role": "user"
           }
         ],
         "temperature": 0,
         "max_completion_tokens": -1,
         "seed": 0,
         "top_p": 1
       }
       ```  
     - Enable redirect handling.

5. **Connect Set API Key to Cerebras Endpoint:**  
   - Link the output of "Set API Key" to the input of "Cerebras endpoint".

6. **Create the Set Node for Output Formatting:**  
   - Add node `Set` named "Return Output".  
   - Configure to assign a string variable `output` with value: `{{$json.choices[0].message.content}}`.  
   - This extracts the text completion from the API response.

7. **Connect Cerebras Endpoint to Return Output:**  
   - Link the output of "Cerebras endpoint" to the input of "Return Output".

8. **Add Sticky Notes (Optional for User Guidance):**  
   - Add a `Sticky Note` near "Set API Key" with content:  
     ```
     ## Set API key 
     **Create an account and [get your Cerebras key](https://cerebras.ai)**
     ```  
   - Add a `Sticky Note` near "Cerebras endpoint" with content:  
     ```
     ## Parameters
     Set your endpoint parameters as needed
     - temperature
     - completion tokens
     - top P
     - reasoning effort

     [Review the endpoint](https://inference-docs.cerebras.ai/api-reference/chat-completions)
     ```

9. **Credentials Setup:**  
   - Ensure you have a valid Cerebras API key. No special credential node is required since the key is injected via a Set node and passed as a bearer token header.  
   - Make sure your OpenAI or LangChain environment is properly configured to support the chat trigger node.

10. **Activate the Workflow:**  
    - Save and activate the workflow.  
    - Test by sending a chat message via the designated trigger interface.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Create an account and get your Cerebras API key to authenticate requests                                       | https://cerebras.ai                                                                                        |
| Cerebras Chat Completions API documentation for parameter tuning and detailed API reference                    | https://inference-docs.cerebras.ai/api-reference/chat-completions                                         |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---