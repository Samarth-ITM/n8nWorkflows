Chat with Uncensored Dolphin Mixtral 8x22B using Novita AI

https://n8nworkflows.xyz/workflows/chat-with-uncensored-dolphin-mixtral-8x22b-using-novita-ai-6746


# Chat with Uncensored Dolphin Mixtral 8x22B using Novita AI

---

### 1. Workflow Overview

This workflow enables conversational interaction with the "Uncensored Dolphin Mixtral 8x22B" AI model via Novita AI’s chat completion API. It is designed to receive chat messages, process them through the specified AI model hosted by Novita, and return the generated AI response.

The workflow’s logic is grouped into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages as triggers.
- **1.2 Parameter Preparation:** Sets necessary API parameters including authentication, system prompt, and token limits.
- **1.3 AI Processing:** Calls Novita’s chat completion API with the prepared parameters.
- **1.4 Response Formatting:** Processes and formats the API response to be returned to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages to trigger the workflow execution.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chatTrigger`; serves as the workflow’s webhook trigger for incoming chat inputs.  
    - *Configuration:* Uses default options with a webhook ID for message reception.  
    - *Key Expressions:* None within the node; raw chat input is captured as `chatInput` in JSON.  
    - *Input/Output:* No input nodes; outputs to "Set fields".  
    - *Version Requirements:* Type version 1.1; requires n8n version supporting Langchain chat trigger nodes.  
    - *Potential Failures:* Webhook not reachable, malformed incoming data, or network issues.  
    - *Sub-workflow:* None.

#### 2.2 Parameter Preparation

- **Overview:**  
  Sets the required parameters for the Novita API call, including API key, system prompt, and token limits.

- **Nodes Involved:**  
  - Set fields

- **Node Details:**  
  - **Set fields**  
    - *Type & Role:* `Set` node; prepares and attaches necessary parameters to the workflow data.  
    - *Configuration:* Assigns three fields:  
      - `NovitaKey`: API key placeholder ("yourNovitakey") to be replaced with a valid key.  
      - `MaxTokens`: Integer set to 500, controlling maximum tokens in the AI response.  
      - `systemPrompt`: String “you are a professional AI helper.” defining the system role in the chat completion.  
    - *Key Expressions:* Values are static, except `NovitaKey` must be replaced with a valid credential.  
    - *Input/Output:* Input from "When chat message received", output to "Generate Chat Completion".  
    - *Version Requirements:* Type version 3.4; standard Set node.  
    - *Potential Failures:* Missing or invalid API key; incorrect data types for parameters.  
    - *Sub-workflow:* None.

#### 2.3 AI Processing

- **Overview:**  
  Sends a POST request to Novita’s chat completion API with the user’s input and system prompt, retrieving the AI-generated chat response.

- **Nodes Involved:**  
  - Generate Chat Completion

- **Node Details:**  
  - **Generate Chat Completion**  
    - *Type & Role:* `HTTP Request` node; executes a POST request to Novita AI’s chat completions endpoint.  
    - *Configuration:*  
      - URL: `https://api.novita.ai/v3/openai/chat/completions`  
      - Method: POST  
      - Body (JSON): Includes the model `"cognitivecomputations/dolphin-mixtral-8x22b"`, system message (from `$json.systemPrompt`), user message (from the incoming chat input `$('When chat message received').item.json.chatInput`), and `max_tokens` (from `$json.MaxTokens`).  
      - Headers: Authorization header with Bearer token from `$json.NovitaKey`.  
    - *Key Expressions:* Dynamic insertion of system prompt, user chat input, max tokens, and API key.  
    - *Input/Output:* Input from "Set fields", output to "Return output".  
    - *Version Requirements:* Type version 4.2; supports advanced HTTP request features.  
    - *Potential Failures:*  
      - Authentication errors due to invalid or missing API key.  
      - Network timeouts or API endpoint failures.  
      - Malformed request body if expressions fail.  
      - Rate limiting or quota exceeded errors from Novita API.  
    - *Sub-workflow:* None.

#### 2.4 Response Formatting

- **Overview:**  
  Extracts and cleans the AI-generated response text to prepare the final output.

- **Nodes Involved:**  
  - Return output

- **Node Details:**  
  - **Return output**  
    - *Type & Role:* `Set` node; formats the API response by extracting the first message content and trimming whitespace.  
    - *Configuration:* Sets the field `output` to the trimmed text from the first choice’s message content (`{{$json.choices[0].message.content.trim()}}`).  
    - *Key Expressions:* Accesses nested JSON response from the API.  
    - *Input/Output:* Input from "Generate Chat Completion", outputs final data for response.  
    - *Version Requirements:* Type version 3.4.  
    - *Potential Failures:*  
      - Missing or unexpected response structure (e.g., no choices array).  
      - Null or empty message content.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role              | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                        |
|---------------------------|-----------------------------------|-----------------------------|----------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Trigger for incoming chat messages | (none)                    | Set fields               | ## Requirements \n- Create a [Novita](https://novita.ai/?ref=mze5m2e&utm_source=affiliate) account\n- Get your Novita api key\n\n## How it works\n- Set your fields and add more parameters as needed into the endpoint\n- Review the [novita docs](https://novita.ai/docs/api-reference/model-apis-llm-create-chat-completion) for chat completions |
| Set fields                | Set                               | Prepare API parameters       | When chat message received | Generate Chat Completion | ## Config \nSet fields to pass into the chat completion api                                                                        |
| Generate Chat Completion  | HTTP Request                      | Call Novita chat completion API | Set fields                 | Return output            | See "When chat message received" sticky note                                                                                      |
| Return output             | Set                               | Format and trim API response | Generate Chat Completion   | (none)                   | See "When chat message received" sticky note                                                                                      |
| Sticky Note               | Sticky Note                       | Documentation and instructions| (none)                     | (none)                   | ## Requirements \n- Create a [Novita](https://novita.ai/?ref=mze5m2e&utm_source=affiliate) account\n- Get your Novita api key\n\n## How it works\n- Set your fields and add more parameters as needed into the endpoint\n- Review the [novita docs](https://novita.ai/docs/api-reference/model-apis-llm-create-chat-completion) for chat completions |
| Sticky Note1              | Sticky Note                       | Configuration note           | (none)                     | (none)                   | ## Config \nSet fields to pass into the chat completion api                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it e.g., "dolphin".**

2. **Add node: When chat message received**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configuration: Use default webhook settings; ensure webhook is active to receive chat inputs.

3. **Add node: Set fields**  
   - Type: `Set`  
   - Configuration:  
     - Add field `NovitaKey` (string), default value: `"yourNovitakey"` (replace with your actual Novita API key).  
     - Add field `MaxTokens` (number), value: `500`.  
     - Add field `systemPrompt` (string), value: `"you are a professional AI helper."`.

4. **Connect "When chat message received" output to "Set fields" input.**

5. **Add node: Generate Chat Completion**  
   - Type: `HTTP Request`  
   - Configuration:  
     - URL: `https://api.novita.ai/v3/openai/chat/completions`  
     - Method: POST  
     - Send Body: JSON  
     - Body content (set as JSON with expressions):  
       ```json
       {
         "model": "cognitivecomputations/dolphin-mixtral-8x22b",
         "messages": [
           {
             "role": "system",
             "content": "{{ $json.systemPrompt }}"
           },
           {
             "role": "user",
             "content": "{{ $('When chat message received').item.json.chatInput }}"
           }
         ],
         "max_tokens": {{ $json.MaxTokens }}
       }
       ```  
     - Headers: Add header  
       - Name: `Authorization`  
       - Value: `Bearer {{ $json.NovitaKey }}`

6. **Connect "Set fields" output to "Generate Chat Completion" input.**

7. **Add node: Return output**  
   - Type: `Set`  
   - Configuration:  
     - Add field `output` (string) with value: `={{ $json.choices[0].message.content.trim() }}`

8. **Connect "Generate Chat Completion" output to "Return output" input.**

9. **Activate the workflow and test by sending chat messages to the webhook URL generated by "When chat message received".**

10. **Optional: Add sticky notes to document requirements and configuration**  
    - Note with content:  
      ```
      ## Requirements 
      - Create a [Novita](https://novita.ai/?ref=mze5m2e&utm_source=affiliate) account
      - Get your Novita API key

      ## How it works
      - Set your fields and add more parameters as needed into the endpoint
      - Review the [novita docs](https://novita.ai/docs/api-reference/model-apis-llm-create-chat-completion) for chat completions
      ```
    - Note with content:  
      ```
      ## Config 
      Set fields to pass into the chat completion api
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Create a [Novita](https://novita.ai/?ref=mze5m2e&utm_source=affiliate) account and obtain your API key to use this workflow | Novita AI official website                                                                                        |
| Review the [Novita API documentation](https://novita.ai/docs/api-reference/model-apis-llm-create-chat-completion) for details on parameters and usage | Novita API docs page                                                                                              |
| The system prompt is currently set to "you are a professional AI helper." Customize it to tailor AI behavior                | Workflow parameter customization                                                                                  |
| Ensure your n8n instance supports Langchain chat trigger node (version >= 1.1)                                              | Node version dependency                                                                                            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---