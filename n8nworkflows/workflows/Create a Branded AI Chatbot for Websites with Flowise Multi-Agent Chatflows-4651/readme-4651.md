Create a Branded AI Chatbot for Websites with Flowise Multi-Agent Chatflows

https://n8nworkflows.xyz/workflows/create-a-branded-ai-chatbot-for-websites-with-flowise-multi-agent-chatflows-4651


# Create a Branded AI Chatbot for Websites with Flowise Multi-Agent Chatflows

### 1. Workflow Overview

This workflow implements a branded AI chatbot for websites using Flowise’s Multi-Agent Chatflows integrated into n8n. It is designed to receive chat messages from website visitors, forward the user input to a Flowise AI agent for processing, and return AI-generated responses in real time.

The workflow includes the following logical blocks:

- **1.1 Input Reception:** Captures incoming chat messages from website visitors through a webhook trigger.
- **1.2 AI Processing:** Forwards the user query to a Flowise API endpoint that hosts a multi-agent chatflow to generate a AI-powered response.
- **1.3 Response Preparation:** Extracts and formats the AI response for sending back to the chatbot interface.
- **1.4 Documentation & Branding:** Provides important instructions and customizable code snippets for embedding and branding the chatbot on a website.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from the website’s chatbot widget. It uses a LangChain chatTrigger node configured as a public webhook to receive messages.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **Name:** When chat message received  
  - **Type:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  - **Role:** Webhook trigger node designed to receive chat inputs via HTTP POST requests.  
  - **Configuration:**  
    - Mode: Webhook  
    - Public: true (accepts requests from any origin)  
    - Allowed Origins: "*" (no CORS restrictions)  
    - Webhook ID: auto-generated unique ID  
  - **Key Expressions:** None, purely input trigger. The chat message payload is expected under `$json.text` (implied by downstream usage).  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Connected to the Flowise node for AI processing.  
  - **Potential Failures:**  
    - Webhook connectivity issues (network errors)  
    - Malformed or missing chat input payloads  
  - **Version Requirements:** Version 1.1 or higher (supports webhook mode)  
  - **Sub-workflows:** None

---

#### 1.2 AI Processing

- **Overview:**  
  This block sends the received user question to the Flowise multi-agent chatflow API endpoint and retrieves the AI-generated response.

- **Nodes Involved:**  
  - Flowise

- **Node Details:**  
  - **Name:** Flowise  
  - **Type:** HTTP Request  
  - **Role:** Makes a POST request to the Flowise API to get AI chat responses.  
  - **Configuration:**  
    - URL: `https://FLOWISEURL/api/v1/prediction/FLOWISE_ID` (placeholders to be replaced with actual Flowise instance URL and flow ID)  
    - Method: POST  
    - Body (JSON): `{ "question": "{{ $json.chatInput }}" }` — sends the chat input as the question parameter  
    - Headers: Content-Type: application/json and Authorization via HTTP Header Authentication (Bearer token)  
    - Credentials: HTTP Header Auth with a stored Flowise API Bearer token  
  - **Key Expressions:** Uses expression `{{ $json.chatInput }}` to dynamically pass user message  
  - **Input Connections:** Connected from "When chat message received" node output  
  - **Output Connections:** Connected to "Edit Fields" node for formatting response  
  - **Potential Failures:**  
    - Authentication errors (invalid or expired API token)  
    - Network timeouts or unreachable Flowise server  
    - Incorrect URL or flow ID causing 404/400 errors  
    - Malformed JSON response or unexpected API output  
  - **Version Requirements:** Version 4.2 or higher to support advanced HTTP request features  
  - **Sub-workflows:** None

---

#### 1.3 Response Preparation

- **Overview:**  
  Extracts the relevant text from the Flowise API response and formats it as the chatbot output.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**  
  - **Name:** Edit Fields  
  - **Type:** Set Node  
  - **Role:** Assigns the AI response text to a specific field named `output` for downstream use by the chatbot interface.  
  - **Configuration:**  
    - Assignments: Set the field `output` to the value `{{$json.text}}` (assumes the Flowise response places the answer in the `text` property)  
  - **Key Expressions:** Uses expression `={{ $json.text }}` to extract text from API response  
  - **Input Connections:** Connected from "Flowise" node output  
  - **Output Connections:** None (end of flow for response)  
  - **Potential Failures:**  
    - If `$json.text` is missing or undefined, output will be empty or cause errors downstream  
  - **Version Requirements:** Version 3.4 or higher  
  - **Sub-workflows:** None

---

#### 1.4 Documentation & Branding

- **Overview:**  
  A set of sticky notes provide essential instructions and customizable code snippets for embedding and branding the chatbot widget on a website, and background information on Flowise.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**  

  1. **Sticky Note**  
     - Content:  
       - HTML snippet to enable n8n CDN for chatbot styling and script  
       - JavaScript snippet to instantiate the chat widget with a placeholder webhook URL  
     - Purpose: Guide users to embed the chatbot widget on their website with proper styling and initialization.  

  2. **Sticky Note1**  
     - Content:  
       - Overview of Flowise platform with an embedded image  
       - Instructions to set HTTP header authorization with "Authorization: Bearer FLOWSIE_API"  
     - Purpose: Introduces Flowise and highlights authentication setup for API calls.  

  3. **Sticky Note2**  
     - Content:  
       - Example JavaScript code showing customization options for the n8n chatbot widget, including UI text, session keys, and initial messages.  
     - Purpose: Helps users customize chatbot appearance and behavior on the frontend.  

  4. **Sticky Note3**  
     - Content:  
       - Reminds users to set environment variables `FLOWISE_URL` and `FLOW_ID`  
     - Purpose: Clarifies that users must configure these critical parameters before running the workflow.  

- **Potential Issues:**  
  - Users must replace placeholders and environment variables correctly to avoid runtime errors.  

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                 | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                   |
|-------------------------|-------------------------------------|--------------------------------|--------------------------|--------------------------|-----------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Receive chat input via webhook | None                     | Flowise                  | Sticky Note1 (Flowise platform info and auth instructions)                                  |
| Flowise                 | HTTP Request                        | Call Flowise API for AI response | When chat message received | Edit Fields              | Sticky Note3 (Set FLOWISE_URL and FLOW_ID)                                                   |
| Edit Fields             | Set                                | Format AI response output        | Flowise                  | None                     |                                                                                               |
| Sticky Note             | Sticky Note                        | Branding and embedding instructions | None                     | None                     | Contains CDN enabling HTML/JS snippet                                                         |
| Sticky Note1            | Sticky Note                        | Flowise platform overview & auth | None                     | None                     | Describes Flowise, API Header Auth setup, includes image                                     |
| Sticky Note2            | Sticky Note                        | Chatbot UI customization sample | None                     | None                     | Provides example JS code for chatbot customization                                            |
| Sticky Note3            | Sticky Note                        | Environment variable reminder    | None                     | None                     | Reminds to set FLOWISE_URL and FLOW_ID                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**
   - Add node: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Name it: "When chat message received"
   - Set mode to `webhook`
   - Make it `public` and allow all origins (`*`)
   - Save webhook URL for embedding in website chatbot
   - This node will receive chat messages containing a `text` property.

2. **Create the HTTP Request Node to Call Flowise:**
   - Add node: HTTP Request
   - Name it: "Flowise"
   - Set method to `POST`
   - Set URL to `https://FLOWISEURL/api/v1/prediction/FLOWISE_ID` (replace placeholders)
   - Set authentication type: HTTP Header Auth
     - Add header: `Authorization: Bearer <your Flowise API token>`
   - Add header: `Content-Type: application/json`
   - Set request body type to JSON
   - Use expression in body: `{ "question": "{{ $json.chatInput }}" }`  
     (ensure the incoming chat message text is mapped to `chatInput`)
   - Connect "When chat message received" node output to this node input.

3. **Create the Set Node to Format Response:**
   - Add node: Set
   - Name it: "Edit Fields"
   - Add assignment: Set field `output` to `{{$json.text}}` (extract answer from Flowise response)
   - Connect "Flowise" node output to this node input.

4. **Establish Connections:**
   - Connect "When chat message received" → "Flowise"
   - Connect "Flowise" → "Edit Fields"

5. **Add Sticky Notes for Documentation and Branding (Optional but Recommended):**
   - Add Sticky Note with CDN and embed code snippet for chatbot widget:
     ```html
     <link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
     <script type="module">
         import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';

         createChat({
             webhookUrl: 'YOUR_PRODUCTION_WEBHOOK_URL'
         });
     </script>
     ```
   - Add Sticky Note describing Flowise platform and API auth header setup.
   - Add Sticky Note with example JS code for customizing chatbot frontend UI.
   - Add Sticky Note reminding to set environment variables `FLOWISE_URL` and `FLOW_ID`.

6. **Credential Setup:**
   - Configure HTTP Header Auth credentials in n8n:
     - Create credential type HTTP Header Auth
     - Store your Flowise API Bearer token securely

7. **Test the Workflow:**
   - Trigger the webhook with a sample chat message payload: `{ "text": "Hello" }`
   - Verify the Flowise API is called and response text is extracted correctly.
   - Confirm output field contains the AI-generated reply.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                      | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Flowise is an open-source, low-code/no-code platform to build AI agents with visual flow design.                                                                                                                                                 | https://flowiseai.com/                                                                           |
| To embed the chatbot on your website, enable n8n CDN styles and scripts as specified in the sticky note to ensure proper UI rendering and functionality.                                                                                       | n8n chat widget CDN: https://cdn.jsdelivr.net/npm/@n8n/chat/dist/                                |
| Customize the chatbot appearance and behavior using the provided JavaScript snippet in Sticky Note2. Adjust welcome messages, language, and UI elements as needed.                                                                              | Inline JavaScript customization snippet                                                         |
| Ensure environment variables `FLOWISE_URL` and `FLOW_ID` are correctly set before running to avoid API call failures.                                                                                                                           | Workflow requirement                                                                             |
| For detailed API authentication, set HTTP header `Authorization` to `Bearer <Flowise API token>` in the HTTP Request node credentials.                                                                                                         | Sticky Note1 instruction                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.