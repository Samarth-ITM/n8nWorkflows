Create AI-Powered Website Chatbot with Langflow Backend and Custom Branding

https://n8nworkflows.xyz/workflows/create-ai-powered-website-chatbot-with-langflow-backend-and-custom-branding-4645


# Create AI-Powered Website Chatbot with Langflow Backend and Custom Branding

### 1. Workflow Overview

This workflow implements an AI-powered website chatbot leveraging a Langflow backend for AI processing and custom branding via frontend code snippets. It is designed to receive chat messages from website visitors, forward them to a Langflow flow running externally, and return the processed AI-generated response to the website frontend. The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Receives incoming chat messages triggered by a webhook designed to be publicly accessible and handle cross-origin requests.

- **1.2 AI Processing via Langflow:** Sends the received chat input to a Langflow API endpoint, which executes a predefined AI flow to generate a chatbot response.

- **1.3 Response Formatting and Frontend Integration:** Extracts the AI response from the Langflow output and formats it for returning to the frontend client. The workflow also includes several sticky notes providing guidance on embedding and customizing the chatbot frontend, as well as setup instructions for Langflow API endpoint configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from the website chatbot frontend via a webhook. It is configured to accept requests publicly with CORS enabled, enabling broad frontend integration.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
      Specialized trigger node optimized for chat-like interactions.  
    - Configuration:  
      - Mode: `webhook`  
      - Public access: `true` (no authentication required)  
      - Allowed origins: `*` (CORS enabled for all origins)  
      - WebhookId assigned internally  
    - Inputs: None (trigger node)  
    - Outputs: Main output connected to the Langflow HTTP Request node  
    - Edge cases / failure types:  
      - Webhook misconfiguration or network failure  
      - CORS preflight rejection if frontend is not properly configured  
      - Malformed incoming payload  
    - Version: 1.1

#### 2.2 AI Processing via Langflow

- **Overview:**  
  Sends the chat input text to a Langflow API endpoint for AI processing. The Langflow flow is expected to handle the input and return a chat response. This interaction uses HTTP POST with JSON body and header authentication.

- **Nodes Involved:**  
  - Langflow

- **Node Details:**

  - **Langflow**  
    - Type: HTTP Request  
      Performs a POST request to Langflow API.  
    - Configuration:  
      - URL: `https://LANGFLOW_URL/api/v1/run/FLOW_ID?stream=false` (placeholder values to be replaced)  
      - Method: POST  
      - Authentication: HTTP Header Auth using a credential named "Langflow API"  
      - Headers: Content-Type: application/json  
      - Body: JSON containing:  
        ```json
        {
          "input_value": "{{ $json.chatInput }}",
          "output_type": "chat",
          "input_type": "chat"
        }
        ```  
      - Sends both body and headers  
    - Inputs: Receives JSON from "When chat message received" node, which should have a property `chatInput` containing the user message.  
    - Outputs: Returns Langflow flow output for next node processing.  
    - Edge cases / failure types:  
      - Network errors or timeouts communicating with Langflow API  
      - Authentication failure due to invalid API key  
      - Invalid or missing input causing Langflow to error  
      - Incorrect URL or FLOW_ID leading to 404 or API errors  
    - Version: 4.2  
    - Credential requirements: HTTP Header Auth credential "Langflow API" configured with a valid API key/token.

#### 2.3 Response Formatting and Frontend Integration

- **Overview:**  
  Extracts the AI-generated text response from the Langflow output and formats it under a field named `output`. Includes multiple sticky notes that document how to embed and style the chatbot frontend using n8n's CDN, customize the chatbot UI, and instructions for setting Langflow URL and flow ID.

- **Nodes Involved:**  
  - Edit Fields  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**

  - **Edit Fields**  
    - Type: Set  
      Used to assign or transform data fields in the workflow.  
    - Configuration:  
      - Creates a new string field `output` by extracting the AI response text from the nested Langflow output JSON path:  
        `{{$json.outputs[0].outputs[0].results.message.data.text}}`  
      - This expression safely navigates Langflow's nested response to get the chatbot text.  
    - Inputs: Connected from "Langflow" node output.  
    - Outputs: Passes formatted data downstream (not connected further in this workflow).  
    - Edge cases / failure types:  
      - Expression failures if Langflow response structure changes or is missing expected fields.  
      - Null or undefined values in JSON path causing empty output.  
    - Version: 3.4

  - **Sticky Note**  
    - Type: Sticky Note (UI/Documentation aid)  
    - Content: Provides instructions for enabling n8n chatbot CDN on the website, including HTML `<link>` and JavaScript snippet to import and initialize the chat widget.  
    - Position: Top-left area of the canvas for visibility.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Describes Langflow as a low-code visual AI application builder, with a screenshot example of a Langflow flow. Aims to inform users about the backend technology powering the chatbot.  

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Content: Provides a JavaScript code snippet illustrating how to customize the n8n chatbot UI, including parameters for webhook URL, language, UI messages, and other options. Useful for frontend developers integrating the chatbot.  

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Content: Short reminder to set actual `LANGFLOW_URL` and `FLOW_ID` values in the HTTP Request node, critical for correct API integration.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                           |
|--------------------------|----------------------------------|---------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------|
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger | Receives chat messages via webhook    | -                           | Langflow                   |                                                                                                     |
| Langflow                 | HTTP Request                     | Sends chat input to Langflow API      | When chat message received  | Edit Fields                |                                                                                                     |
| Edit Fields              | Set                             | Extracts AI text from Langflow output | Langflow                    | -                          |                                                                                                     |
| Sticky Note              | Sticky Note                     | n8n CDN embedding instructions        | -                           | -                          | Contains HTML/JS snippet to enable n8n CDN chatbot on website                                       |
| Sticky Note1             | Sticky Note                     | Langflow overview and example image   | -                           | -                          | Explains Langflow platform and shows example flow screenshot                                        |
| Sticky Note2             | Sticky Note                     | Chatbot UI customization example      | -                           | -                          | Provides JavaScript code example for customizing chatbot UI                                         |
| Sticky Note3             | Sticky Note                     | Reminder for Langflow URL/Flow ID setup | -                          | -                          | "Set LANGFLOW_URL and FLOW_ID"                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Set mode to `webhook`  
   - Enable `public` access (no auth)  
   - Under options, set `allowedOrigins` to `*` (enable CORS for all origins)  
   - This node will receive chat messages from your website frontend.

2. **Add HTTP Request Node:**  
   - Add node: `HTTP Request`  
   - Name: `Langflow`  
   - Set method to `POST`  
   - Set URL to `https://LANGFLOW_URL/api/v1/run/FLOW_ID?stream=false` (replace placeholders with actual Langflow API URL and flow ID)  
   - Under Authentication, select HTTP Header Auth and configure credentials with your Langflow API key (create credential named "Langflow API")  
   - Under Headers, add: `Content-Type: application/json`  
   - Set Body Content Type to JSON  
   - Set JSON Body to:  
     ```json
     {
       "input_value": "{{ $json.chatInput }}",
       "output_type": "chat",
       "input_type": "chat"
     }
     ```  
   - Connect output of `When chat message received` node to input of this node.

3. **Add Set Node:**  
   - Add node: `Set`  
   - Name: `Edit Fields`  
   - Add new field:  
     - Name: `output`  
     - Type: `string`  
     - Value (expression): `{{$json.outputs[0].outputs[0].results.message.data.text}}`  
   - Connect output of `Langflow` node to input of this node.

4. **Add Sticky Notes for Documentation (optional but recommended):**  
   - Add Sticky Note with content:  
     ```html
     ### Enable n8n CDN on your website
     <link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
     <script type="module">
       import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';

       createChat({
         webhookUrl: 'YOUR_PRODUCTION_WEBHOOK_URL'
       });
     </script>
     ```  
   - Add Sticky Note explaining Langflow platform with example image link.  
   - Add Sticky Note with JavaScript customization example for the chatbot UI.  
   - Add Sticky Note reminding to set `LANGFLOW_URL` and `FLOW_ID` in HTTP Request node.

5. **Configure Credentials:**  
   - Create HTTP Header Auth credential named "Langflow API" with the API key/token used for Langflow API access.

6. **Finalize and Test:**  
   - Activate the workflow.  
   - Deploy your website frontend embedding the n8n chatbot CDN snippet with the webhook URL pointing to the `When chat message received` webhook URL generated by n8n.  
   - Test sending chat messages and verify responses.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Langflow is a Python-based low-code visual AI workflow platform, suitable for building chatbots and AI applications with easy drag-and-drop interface. | https://github.com/langflow/langflow |
| Use n8n’s official chatbot CDN for easy frontend integration: CSS and JS provided via jsdelivr CDN. | https://cdn.jsdelivr.net/npm/@n8n/chat/ |
| Customize chatbot frontend using the provided JavaScript code snippet (Sticky Note2) to adjust UI text, language, and behavior. | See Sticky Note2 content in workflow |
| Ensure to replace placeholders `LANGFLOW_URL` and `FLOW_ID` with actual values from your Langflow deployment to avoid API call errors. | See Sticky Note3 content in workflow |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.