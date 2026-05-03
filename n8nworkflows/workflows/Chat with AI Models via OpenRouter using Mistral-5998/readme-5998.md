Chat with AI Models via OpenRouter using Mistral

https://n8nworkflows.xyz/workflows/chat-with-ai-models-via-openrouter-using-mistral-5998


# Chat with AI Models via OpenRouter using Mistral

### 1. Workflow Overview

This workflow enables users to interact with AI language models hosted on OpenRouter.ai, specifically using the Mistral model variant. It sends a user-defined prompt to the OpenRouter chat completion API and processes the AI's textual response by summarizing it. The workflow is designed for testing or demonstrating conversational AI capabilities via HTTP API calls within n8n.

Logical blocks:

- **1.1 Input Trigger:** Manual trigger to start the workflow.
- **1.2 Setting Model and Prompt:** Prepare the input data specifying the AI model and the user message.
- **1.3 AI Processing:** Send the prompt to OpenRouter's chat completions API with appropriate authentication.
- **1.4 Summarization:** Extract and summarize the returned AI response message for simplified output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:**  
  This block initiates the workflow execution manually by the user, allowing on-demand interaction.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **Node Name:** When clicking ‘Execute workflow’  
    - **Type:** Manual Trigger  
    - **Configuration:** No parameters; triggers workflow when user executes manually.  
    - **Expressions/Variables:** None  
    - **Inputs:** None  
    - **Outputs:** Passes empty data to the next node to start the workflow  
    - **Version Requirements:** Standard since n8n version 0.92+  
    - **Failures:** None expected, unless workflow is paused or stopped  
    - **Sub-workflow:** None

#### 2.2 Setting Model and Prompt

- **Overview:**  
  This block defines the AI model to use and the user prompt message that will be sent to the AI API.

- **Nodes Involved:**  
  - Set Model & Prompt

- **Node Details:**

  - **Node Name:** Set Model & Prompt  
    - **Type:** Set  
    - **Configuration:**  
      - Assigns two string variables in the workflow’s JSON data for subsequent use:  
        - `Model`: `"mistralai/mistral-small-3.2-24b-instruct:free"` (specifies the Mistral model hosted on OpenRouter)  
        - `Message`: `"What is the meaning of life?"` (the prompt sent to AI)  
      - An unused empty string assignment is present but has no effect.  
    - **Expressions/Variables:** Static string values set here; referenced later in HTTP request body via expressions `{{ $json.Model }}` and `{{ $json.Message }}`.  
    - **Inputs:** Receives trigger from manual start node  
    - **Outputs:** Outputs JSON with assigned variables to the next node  
    - **Version Requirements:** Standard Set node functionality  
    - **Failures:** None expected unless expression syntax is malformed  
    - **Sub-workflow:** None

#### 2.3 AI Processing

- **Overview:**  
  This block sends the chat prompt and model choice to OpenRouter's chat completions API, returning the AI-generated response.

- **Nodes Involved:**  
  - OpenRouter.ai

- **Node Details:**

  - **Node Name:** OpenRouter.ai  
    - **Type:** HTTP Request  
    - **Configuration:**  
      - HTTP Method: POST  
      - URL: `https://openrouter.ai/api/v1/chat/completions`  
      - Body (JSON): Constructs a request payload with:  
        ```json
        {
          "model": "{{ $json.Model }}",
          "messages": [
            {
              "role": "user",
              "content": "{{ $json.Message }}"
            }
          ]
        }
        ```  
        - This dynamically injects the model and user message from previous Set node.  
      - Authentication: Bearer token via generic HTTP credential (requires valid OpenRouter API key stored in n8n credentials).  
      - Sends body as JSON.  
    - **Expressions/Variables:** Uses expressions to pull model and message from incoming JSON.  
    - **Inputs:** Receives JSON with `Model` and `Message` from Set node  
    - **Outputs:** Returns JSON response from OpenRouter API, typically including `choices[0].message.content` with AI reply text.  
    - **Version Requirements:** HTTP Request node version 4+ recommended for latest features  
    - **Failures / Edge Cases:**  
      - Authentication errors if API key is invalid or missing  
      - API rate limiting or quota exceeded errors  
      - Network timeouts or connectivity issues  
      - Malformed response data or unexpected API changes  
    - **Sub-workflow:** None

#### 2.4 Summarization

- **Overview:**  
  This block processes the AI response to extract and summarize the generated content, providing a concise output.

- **Nodes Involved:**  
  - Summarize

- **Node Details:**

  - **Node Name:** Summarize  
    - **Type:** Summarize  
    - **Configuration:**  
      - Summarizes the field `choices[0].message.content` using the aggregation method `min` (which in this context extracts the minimal or first content string, effectively extracting the AI's reply).  
    - **Expressions/Variables:** Field to summarize is specified as `choices[0].message.content`  
    - **Inputs:** Takes JSON output from OpenRouter.ai node  
    - **Outputs:** Outputs summarized text (the AI reply) for further use or display  
    - **Version Requirements:** Summarize node version 1.1 or higher  
    - **Failures / Edge Cases:**  
      - Missing or malformed `choices` array or message content in API response  
      - Empty or null response fields causing summarization to fail  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                 | Node Type        | Functional Role                   | Input Node(s)               | Output Node(s)     | Sticky Note                                  |
|---------------------------|------------------|---------------------------------|-----------------------------|--------------------|----------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger   | Starts the workflow manually     | None                        | Set Model & Prompt  |                                              |
| Set Model & Prompt         | Set              | Defines AI model and user prompt | When clicking ‘Execute workflow’ | OpenRouter.ai      |                                              |
| OpenRouter.ai             | HTTP Request     | Sends prompt to OpenRouter AI API | Set Model & Prompt           | Summarize          |                                              |
| Summarize                 | Summarize        | Extracts and condenses AI response | OpenRouter.ai               | None               |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   - No configuration needed.

2. **Add Set Node to Define Model and Prompt**  
   - Add a **Set** node named `Set Model & Prompt`.  
   - Add two string fields:  
     - `Model` = `mistralai/mistral-small-3.2-24b-instruct:free`  
     - `Message` = `What is the meaning of life?`  
   - Connect output of Manual Trigger to input of this Set node.

3. **Add HTTP Request Node to Call OpenRouter API**  
   - Add an **HTTP Request** node named `OpenRouter.ai`.  
   - Configure as follows:  
     - Method: POST  
     - URL: `https://openrouter.ai/api/v1/chat/completions`  
     - Authentication: Select or create a new **HTTP Bearer Token Credential** with your OpenRouter API key.  
     - Body Content Type: JSON  
     - Body Parameters (raw JSON, use expressions):  
       ```json
       {
         "model": "{{ $json.Model }}",
         "messages": [
           {
             "role": "user",
             "content": "{{ $json.Message }}"
           }
         ]
       }
       ```  
   - Connect output of `Set Model & Prompt` node to input of this HTTP Request node.

4. **Add Summarize Node to Extract AI Response**  
   - Add a **Summarize** node named `Summarize`.  
   - Configure to summarize the field: `choices[0].message.content` using aggregation method `min`.  
   - Connect output of `OpenRouter.ai` node to input of this Summarize node.

5. **Save and Test the Workflow**  
   - Activate or execute the workflow manually via the trigger node.  
   - Verify the summarized AI response output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses the OpenRouter API which requires an API key with Bearer authentication. Ensure your API key is current and valid.       | https://openrouter.ai/docs/api-reference/chat-completions |
| Model used is `mistralai/mistral-small-3.2-24b-instruct:free` – a free instruct-tuned Mistral small model available via OpenRouter.         | Model info from OpenRouter.ai                             |
| Summarize node configured with `min` aggregation effectively extracts the first or minimal value from array fields for simple text output. | n8n Summarize node documentation                          |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, a tool for integration and automation. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.