Line Chatbot Handling AI Responses with Groq and Llama3

https://n8nworkflows.xyz/workflows/line-chatbot-handling-ai-responses-with-groq-and-llama3-2977


# Line Chatbot Handling AI Responses with Groq and Llama3

### 1. Workflow Overview

This workflow enables a LINE chatbot to handle user messages by leveraging Meta’s Llama 3.3 AI model via Groq’s API, ensuring robust and intelligent conversational responses. It is designed to process potentially large and complex AI-generated texts without causing JSON or API errors when replying through the LINE Messaging API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming messages from users via the LINE Messaging API webhook.
- **1.2 Message Extraction:** Extracts and prepares the user’s message and relevant metadata for processing.
- **1.3 AI Processing:** Sends the extracted message to Groq’s API using Meta’s Llama 3.3 model to generate an AI response.
- **1.4 Reply Dispatch:** Sends the AI-generated reply back to the user through the LINE Messaging API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming POST requests from the LINE Messaging API webhook, capturing user messages and event data.

- **Nodes Involved:**  
  - `Line: Messaging API`

- **Node Details:**  
  - **Node:** Line: Messaging API  
    - **Type:** Webhook  
    - **Role:** Entry point for receiving messages from LINE users.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Webhook Path: `befed026-573c-4d3a-9113-046ea8ae5930` (unique identifier for this webhook)  
      - No additional options configured.  
    - **Expressions/Variables:** None at this stage; raw webhook payload is captured.  
    - **Input:** External HTTP POST from LINE platform.  
    - **Output:** JSON payload containing LINE event data, including user message, user ID, reply token, etc.  
    - **Version Requirements:** Requires n8n version 1.79.0 or later for compatibility.  
    - **Potential Failures:**  
      - Webhook not reachable or misconfigured URL.  
      - Invalid or missing LINE webhook signature (not handled explicitly here).  
      - Unexpected payload structure from LINE API.  

#### 2.2 Message Extraction

- **Overview:**  
  Extracts the user’s message text, message ID, and user ID from the webhook payload to prepare for AI processing.

- **Nodes Involved:**  
  - `Get Messages`

- **Node Details:**  
  - **Node:** Get Messages  
    - **Type:** Set  
    - **Role:** Extracts and assigns specific fields from the incoming webhook JSON for downstream use.  
    - **Configuration:**  
      - Assigns three fields:  
        - `body.events[0].message.text` → user’s message text  
        - `body.events[0].message.id` → message ID  
        - `body.events[0].source.userId` → user ID  
      - Uses expressions to map these fields from the webhook JSON.  
    - **Expressions:**  
      - `={{ $json.body.events[0].message.text }}`  
      - `={{ $json.body.events[0].message.id }}`  
      - `={{ $json.body.events[0].source.userId }}`  
    - **Input:** JSON from `Line: Messaging API` node.  
    - **Output:** JSON with only the extracted fields for clarity and downstream processing.  
    - **Version Requirements:** Uses n8n version 3.4 for Set node features.  
    - **Potential Failures:**  
      - If the webhook payload structure changes or missing fields, expressions may fail or return undefined.  
      - No error handling for missing or malformed message data.  

#### 2.3 AI Processing

- **Overview:**  
  Sends the extracted user message to Groq’s API, which uses Meta’s Llama 3.3 model to generate a conversational AI response.

- **Nodes Involved:**  
  - `Groq AI Assistant`

- **Node Details:**  
  - **Node:** Groq AI Assistant  
    - **Type:** HTTP Request  
    - **Role:** Calls Groq’s OpenAI-compatible chat completions endpoint to get AI-generated replies.  
    - **Configuration:**  
      - URL: `https://api.groq.com/openai/v1/chat/completions`  
      - Method: POST  
      - Body (JSON):  
        ```json
        {
          "messages": [
            {
              "role": "user",
              "content": "{{ $json.body.events[0].message.text }}"
            }
          ],
          "model": "llama-3.3-70b-versatile",
          "temperature": 1,
          "max_completion_tokens": 2500,
          "top_p": 1,
          "stream": null,
          "stop": null
        }
        ```  
      - Authentication: HTTP Header Auth using Groq API key credential.  
    - **Expressions:**  
      - Injects user message text dynamically into the request body.  
    - **Input:** JSON from `Get Messages` node.  
    - **Output:** JSON response from Groq API containing AI-generated message(s).  
    - **Version Requirements:** HTTP Request node version 4.2 or later recommended.  
    - **Potential Failures:**  
      - Authentication errors if API key is invalid or expired.  
      - Network timeouts or API rate limits.  
      - Payload size exceeding limits (max_completion_tokens capped at 2500 to avoid LINE limits).  
      - Unexpected API response structure changes.  

#### 2.4 Reply Dispatch

- **Overview:**  
  Sends the AI-generated response back to the user via the LINE Messaging API reply endpoint, using the reply token from the original event.

- **Nodes Involved:**  
  - `Line: Reply Message`

- **Node Details:**  
  - **Node:** Line: Reply Message  
    - **Type:** HTTP Request  
    - **Role:** Posts the AI response as a reply message to the user on LINE.  
    - **Configuration:**  
      - URL: `https://api.line.me/v2/bot/message/reply`  
      - Method: POST  
      - Body (JSON):  
        ```json
        {
          "replyToken": "{{ $('Line: Messaging API').item.json.body.events[0].replyToken }}",
          "messages": [
            {
              "type": "text",
              "text": {{ JSON.stringify($('Groq AI Assistant').item.json.choices[0].message.content) }}
            }
          ]
        }
        ```  
      - Authentication: HTTP Header Auth with LINE channel access token credential.  
    - **Expressions:**  
      - Uses replyToken from the original webhook event to ensure correct message threading.  
      - Extracts AI response text from Groq API response dynamically.  
    - **Input:** JSON from `Groq AI Assistant` and `Line: Messaging API` nodes.  
    - **Output:** HTTP response from LINE API confirming message delivery.  
    - **Version Requirements:** HTTP Request node version 4.2 or later recommended.  
    - **Potential Failures:**  
      - Invalid or expired LINE channel access token.  
      - Incorrect replyToken usage (expired or already used tokens).  
      - Message size limits exceeded (handled by limiting tokens in AI response).  
      - JSON formatting errors if AI response contains unexpected characters.  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                        | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                              |
|---------------------|---------------------|-------------------------------------|-----------------------|-----------------------|--------------------------------------------------------------------------------------------------------|
| Sticky Note         | Sticky Note         | Informational note on workflow purpose | None                  | None                  | ## Line AI Chatbot with Groq \nThis workflow automates the process of handling messages from Line Messaging API by sending message to Groq as your AI assistant and reply back to you. In this workflow, you can see that there is no JSON error when sending long and complex message. |
| Line: Messaging API | Webhook             | Receives incoming LINE messages      | None                  | Get Messages          | ## LINE Messaging API \nGet the access token from Line Business https://manager.line.biz/              |
| Get Messages        | Set                 | Extracts user message and metadata   | Line: Messaging API    | Groq AI Assistant     | ## Get Message\nGet message from Line account.                                                        |
| Groq AI Assistant   | HTTP Request        | Sends user message to Groq AI model  | Get Messages          | Line: Reply Message   | ## Groq API Key\nApply Groq account and get API key then you should set ```max_completion_tokens``` less than 5000 because of Line message limitation |
| Line: Reply Message | HTTP Request        | Sends AI response back to LINE user  | Groq AI Assistant     | None                  | ## Reply message\nUse replyToken from Line messaging API and use ```choices[].message.content``` to reply to you. |
| Sticky Note1        | Sticky Note         | Note about LINE Messaging API token  | None                  | None                  | ## LINE Messaging API \nGet the access token from Line Business https://manager.line.biz/              |
| Sticky Note2        | Sticky Note         | Note about message extraction        | None                  | None                  | ## Get Message\nGet message from Line account.                                                        |
| Sticky Note3        | Sticky Note         | Note about Groq API key and token limits | None                  | None                  | ## Groq API Key\nApply Groq account and get API key then you should set ```max_completion_tokens``` less than 5000 because of Line message limitation |
| Sticky Note4        | Sticky Note         | Note about replying with LINE API    | None                  | None                  | ## Reply message\nUse replyToken from Line messaging API and use ```choices[].message.content``` to reply to you. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node named `Line: Messaging API`.  
   - Set HTTP Method to `POST`.  
   - Set Webhook Path to a unique string (e.g., `befed026-573c-4d3a-9113-046ea8ae5930`).  
   - Save and activate the webhook to receive LINE messages.  

2. **Create Set Node for Message Extraction**  
   - Add a **Set** node named `Get Messages`.  
   - Connect `Line: Messaging API` → `Get Messages`.  
   - Add three fields with expressions:  
     - `body.events[0].message.text` = `={{ $json.body.events[0].message.text }}`  
     - `body.events[0].message.id` = `={{ $json.body.events[0].message.id }}`  
     - `body.events[0].source.userId` = `={{ $json.body.events[0].source.userId }}`  

3. **Create HTTP Request Node for Groq AI**  
   - Add an **HTTP Request** node named `Groq AI Assistant`.  
   - Connect `Get Messages` → `Groq AI Assistant`.  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.groq.com/openai/v1/chat/completions`  
     - Authentication: HTTP Header Auth (create credential with Groq API key).  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "messages": [
           {
             "role": "user",
             "content": "{{ $json.body.events[0].message.text }}"
           }
         ],
         "model": "llama-3.3-70b-versatile",
         "temperature": 1,
         "max_completion_tokens": 2500,
         "top_p": 1,
         "stream": null,
         "stop": null
       }
       ```  
     - Ensure `sendBody` is enabled.  

4. **Create HTTP Request Node for LINE Reply**  
   - Add an **HTTP Request** node named `Line: Reply Message`.  
   - Connect `Groq AI Assistant` → `Line: Reply Message`.  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.line.me/v2/bot/message/reply`  
     - Authentication: HTTP Header Auth (create credential with LINE channel access token).  
     - Body Content Type: JSON  
     - Body:  
       ```json
       {
         "replyToken": "{{ $('Line: Messaging API').item.json.body.events[0].replyToken }}",
         "messages": [
           {
             "type": "text",
             "text": {{ JSON.stringify($('Groq AI Assistant').item.json.choices[0].message.content) }}
           }
         ]
       }
       ```  
     - Ensure `sendBody` is enabled.  

5. **Add Sticky Notes (Optional for Documentation)**  
   - Add sticky notes near each block to document purpose and configuration tips, e.g.:  
     - LINE Messaging API token source: https://manager.line.biz/  
     - Groq API key setup and token limits (max_completion_tokens < 5000)  
     - Reply token usage and AI response extraction  

6. **Activate Workflow**  
   - Save and activate the workflow.  
   - Configure LINE Messaging API webhook URL to point to your n8n webhook URL with the specified path.  
   - Test by sending messages to your LINE chatbot and verify AI responses.  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                               |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------|
| To create a LINE Business account and get access tokens, visit: https://manager.line.biz/          | LINE Business portal for access token setup   |
| Groq API key required for AI calls; sign up at https://groq.com                                    | Groq AI platform for API key and model usage  |
| Limit `max_completion_tokens` to less than 5000 to avoid LINE message size limitations             | Important for preventing message truncation   |
| Workflow requires n8n version 1.79.0 or later for compatibility                                    | n8n version requirement                         |
| This workflow demonstrates handling large and complex AI-generated texts without JSON errors       | Key feature of this implementation             |

---

This documentation fully describes the workflow’s structure, node configurations, and operational logic, enabling users or AI agents to understand, reproduce, and maintain the workflow effectively.