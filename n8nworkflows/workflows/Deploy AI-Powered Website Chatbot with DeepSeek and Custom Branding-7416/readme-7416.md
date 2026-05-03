Deploy AI-Powered Website Chatbot with DeepSeek and Custom Branding

https://n8nworkflows.xyz/workflows/deploy-ai-powered-website-chatbot-with-deepseek-and-custom-branding-7416


# Deploy AI-Powered Website Chatbot with DeepSeek and Custom Branding

---

### 1. Workflow Overview

This workflow implements a **brandable, AI-powered chatbot widget for websites** using n8n automation. It targets **business owners, developers, and marketers** who want to embed a customizable chat interface on their websites, supporting session management and AI-driven conversational responses powered by DeepSeek.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receiving and handling incoming chat messages from the web chat widget via an HTTP webhook.
- **1.2 AI Processing:** Processing the message input through an AI agent that uses DeepSeek's language model and maintains conversation memory.
- **1.3 Response Delivery:** Returning the AI-generated response back to the chat widget with appropriate HTTP headers to support CORS and JSON.

Additionally, the workflow includes sticky notes providing branding, usage instructions, and implementation details for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives chat messages posted from the website chat widget through a RESTful webhook endpoint. It serves as the entry point to the workflow, capturing user input and session information.

- **Nodes Involved:**  
  - Webhook (POST)

- **Node Details:**

  - **Node Name:** Webhook (POST)  
    - **Type and Role:** n8n Webhook node; exposes an HTTP POST endpoint to receive incoming chat messages.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Webhook Path: `brand-bot` (i.e., the webhook URL ends with `/webhook/brand-bot`)  
      - Response Mode: `responseNode` (response is sent by a downstream node)  
    - **Key Expressions:** None directly; expects JSON payload with at least `message` and `sessionId` fields in the request body.  
    - **Input/Output:** No input nodes; outputs the received request data (body) to the next node.  
    - **Edge Cases / Potential Failures:**  
      - Invalid or malformed JSON in POST body.  
      - Missing required fields (`message` or `sessionId`) could cause downstream errors.  
      - Unauthorized access is not explicitly restricted here; CORS and headers set downstream.  
    - **Version Requirements:** Compatible with n8n v1 and above.

#### 2.2 AI Processing

- **Overview:**  
  This block orchestrates AI-driven chat response generation. It uses a LangChain AI Agent that integrates a DeepSeek chat language model with simple windowed memory keyed by sessionId to maintain conversation context.

- **Nodes Involved:**  
  - AI Agent  
  - DeepSeek Chat Model  
  - Simple Memory

- **Node Details:**

  - **Node Name:** AI Agent  
    - **Type and Role:** LangChain Agent node; central AI processing node that runs the prompt using linked memory and language model.  
    - **Configuration:**  
      - Input Text: dynamically set to the incoming chat message (`{{$json.body.message}}`).  
      - Prompt Type: `define` (custom prompt definition, likely preconfigured internally).  
      - Options: default / empty.  
    - **Input Connections:**  
      - From Webhook (POST) (main input for text).  
      - From Simple Memory (memory context input).  
      - From DeepSeek Chat Model (language model input).  
    - **Output Connections:**  
      - To Respond to Webhook (returns AI output).  
    - **Edge Cases / Failures:**  
      - If message text is missing or empty, AI may fail or return invalid response.  
      - Memory session key missing or invalid can cause memory context loss.  
      - DeepSeek API errors (rate limits, auth failure) propagate here.  
    - **Version:** v2.1 for LangChain Agent.

  - **Node Name:** DeepSeek Chat Model  
    - **Type and Role:** LangChain language model node using DeepSeek API; generates AI chat completions.  
    - **Configuration:**  
      - Default options, no custom parameters specified.  
      - Credentials: DeepSeek API account credentials required.  
    - **Input:** Connected to AI Agent's `ai_languageModel` input.  
    - **Output:** Sends language model completions back to AI Agent.  
    - **Edge Cases:**  
      - API authentication failure if credentials are invalid.  
      - Network timeouts or API quota limits.  
    - **Version:** 1.

  - **Node Name:** Simple Memory  
    - **Type and Role:** LangChain memory buffer node; maintains a windowed conversation history keyed by session.  
    - **Configuration:**  
      - Session Key: derived dynamically from webhook POST body `sessionId` to track unique user sessions (`={{ $('Webhook (POST)').item.json.body.sessionId }}`).  
      - Session ID Type: `customKey` (uses custom session key rather than default).  
    - **Input:** Connected from Webhook (POST) to AI Agent via `ai_memory` input.  
    - **Output:** Provides memory context to AI Agent.  
    - **Edge Cases:**  
      - Missing or malformed `sessionId` results in inability to maintain session context.  
      - Memory overflow or excessive session length could degrade performance.  
    - **Version:** 1.3.

#### 2.3 Response Delivery

- **Overview:**  
  This block sends the AI-generated chat response back to the web client that initiated the webhook call, including CORS headers to enable browser-based cross-origin requests.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Node Name:** Respond to Webhook  
    - **Type and Role:** n8n node that sends HTTP response to webhook caller.  
    - **Configuration:**  
      - Response Headers set explicitly to:  
        - `Content-Type: application/json`  
        - `Access-Control-Allow-Origin: *` (enable CORS from any origin)  
        - `Access-Control-Allow-Headers: Content-Type, x-api-key`  
        - `Access-Control-Allow-Methods: POST, OPTIONS`  
    - **Input:** Connected from AI Agent main output.  
    - **Output:** Returns the AI-generated message (likely in JSON) to the client.  
    - **Edge Cases:**  
      - If AI Agent output is malformed or empty, response may be invalid.  
      - CORS headers may need adjustment for specific deployment domains.  
    - **Version:** 1.

#### 2.4 Documentation and Branding (Sticky Notes)

- **Overview:**  
  Provides visual documentation and branding information for users deploying or customizing the chatbot.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Content: Detailed markdown with branding, usage instructions, links to implementation details, and feature list of the chat widget.  
    - Position: Top-left area for visibility.  
    - Purpose: Helps users understand the project goals and capabilities.

  - **Sticky Note1**  
    - Content: An image with a hyperlink to the branded chatbox demo or resource page.  
    - Purpose: Visual branding and easy access to more information.

---

### 3. Summary Table

| Node Name         | Node Type                          | Functional Role               | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                                                              |
|-------------------|----------------------------------|------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook (POST)    | n8n-nodes-base.webhook            | Input reception (chat message) | None                  | AI Agent             |                                                                                                                                          |
| AI Agent          | @n8n/n8n-nodes-langchain.agent   | AI chat processing and orchestration | Webhook (POST), Simple Memory, DeepSeek Chat Model | Respond to Webhook    |                                                                                                                                          |
| DeepSeek Chat Model| @n8n/n8n-nodes-langchain.lmChatDeepSeek | Language model providing AI completions | None (credential-based) | AI Agent             |                                                                                                                                          |
| Simple Memory     | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context memory | None                  | AI Agent             |                                                                                                                                          |
| Respond to Webhook| n8n-nodes-base.respondToWebhook   | Sends AI response to client  | AI Agent              | None                 |                                                                                                                                          |
| Sticky Note       | n8n-nodes-base.stickyNote         | Documentation and branding   | None                  | None                 | ## Brandable Custom Chatbox for N8N - Follow along to add a custom branded chat widget to your webiste. [Implementation Details](https://omerfayyaz.com/n8n-brandable-chatbox/index.html) This template is perfect for business owners, developers, and marketers who want to add a professional, branded AI chatbot to their website. Whether you're running an e-commerce site, a SaaS platform, or a corporate website, this template gives you a fully customizable chat widget that integrates seamlessly with your brand. The chat widget itself is a vanilla JavaScript component that you embed on your website. It features: - Customizable colors, branding, and positioning - Light/dark theme support - Mobile-responsive design - Local conversation history - Session management with expiration - WordPress plugin integration |
| Sticky Note1      | n8n-nodes-base.stickyNote         | Branding image and link      | None                  | None                 | [![Brandable Custom Chatbox for N8N](https://omerfayyaz.com/n8n-brandable-chatbox/n8n-brandable-chatbox-copy.jpg)](https://omerfayyaz.com/n8n-brandable-chatbox/index.html) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Type: Webhook (n8n-nodes-base.webhook)  
   - Set HTTP Method to `POST`  
   - Set Path to `brand-bot`  
   - Set Response Mode to `responseNode` (response will be sent from a downstream node)  
   - This node will receive chat messages from the website.

2. **Create a DeepSeek Chat Model node**  
   - Type: LangChain DeepSeek Chat Model (`@n8n/n8n-nodes-langchain.lmChatDeepSeek`)  
   - Assign DeepSeek API credentials (create or select credential with your account details)  
   - Use default options unless customization is needed.

3. **Create a Simple Memory node**  
   - Type: LangChain Memory Buffer Window (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)  
   - Set Session Key to: `={{ $('Webhook (POST)').item.json.body.sessionId }}` (captures the sessionId from incoming webhook payload)  
   - Set Session ID Type to `customKey`  
   - This enables conversation context per user session.

4. **Create an AI Agent node**  
   - Type: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)  
   - Set the Text input expression to: `={{ $json.body.message }}` (extracts user message from webhook body)  
   - Set Prompt Type to `define` (use default or existing prompt definition)  
   - Connect the Simple Memory node to the AI Agent’s `ai_memory` input  
   - Connect the DeepSeek Chat Model node to the AI Agent’s `ai_languageModel` input  
   - Connect the Webhook node’s main output to the AI Agent’s main input

5. **Create a Respond to Webhook node**  
   - Type: Respond to Webhook (n8n-nodes-base.respondToWebhook)  
   - Configure Response Headers:  
     - `Content-Type: application/json`  
     - `Access-Control-Allow-Origin: *`  
     - `Access-Control-Allow-Headers: Content-Type, x-api-key`  
     - `Access-Control-Allow-Methods: POST, OPTIONS`  
   - Connect AI Agent’s main output to this node.  
   - This node sends the AI response back to the chat widget caller.

6. **Connect the workflow**  
   - Webhook (POST) → AI Agent (main input)  
   - Simple Memory → AI Agent (ai_memory input)  
   - DeepSeek Chat Model → AI Agent (ai_languageModel input)  
   - AI Agent → Respond to Webhook

7. **Optional: Add Sticky Notes**  
   - Add sticky notes with branding and documentation content for clarity and user guidance. Use markdown for formatting and images as needed.

8. **Credentials Setup**  
   - Ensure DeepSeek API credentials are configured in n8n credentials manager before running.  
   - No other credentials are required.

9. **Test the workflow**  
   - Deploy the webhook URL to your website chat widget.  
   - Send POST requests with payload:  
     ```json
     {
       "message": "Hello, how can you help me?",
       "sessionId": "unique-session-identifier"
     }
     ```  
   - Verify the AI response is returned with correct headers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| Brandable Custom Chatbox for N8N: Add a custom branded chat widget to your website with a fully customizable vanilla JavaScript component supporting themes, session management, and WordPress integration.                                      | https://omerfayyaz.com/n8n-brandable-chatbox/index.html                |
| Image and demo link to the branded chatbox example for visual reference and design inspiration.                                                                                                                                                 | https://omerfayyaz.com/n8n-brandable-chatbox/n8n-brandable-chatbox-copy.jpg |

---

**Disclaimer:** The provided content is exclusively generated from an n8n automated workflow. It strictly respects all current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---