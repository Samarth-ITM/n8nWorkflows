Build Your Own Counseling Chatbot on LINE to Support Mental Health Conversations

https://n8nworkflows.xyz/workflows/build-your-own-counseling-chatbot-on-line-to-support-mental-health-conversations-2975


# Build Your Own Counseling Chatbot on LINE to Support Mental Health Conversations

### 1. Workflow Overview

This workflow implements a counseling chatbot on the LINE messaging platform designed to provide emotional support and mental health guidance using AI language models, specifically GPT-4 via Azure OpenAI. It targets developers, counselors, businesses, and nonprofits aiming to offer accessible, empathetic, and professional conversational therapy through LINE.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Webhook Listener)**: Receives incoming messages from LINE via webhook.
- **1.2 User Feedback (Loading Animation)**: Sends a loading animation to reassure users that their message is being processed.
- **1.3 Message Type Validation**: Checks if the incoming message is text; non-text inputs are rejected with a polite notification.
- **1.4 AI Processing (Chat Model & Agent)**: Sends the validated text input to an AI agent configured with a CBT-based system prompt, leveraging Azure OpenAI GPT-4.
- **1.5 Response Formatting**: Cleans and formats the AI-generated response to ensure compatibility with LINE’s messaging API.
- **1.6 Reply to User**: Sends the formatted response back to the user on LINE using the reply token.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception (Webhook Listener)

- **Overview:**  
  This block listens for incoming POST requests from LINE’s webhook, capturing user messages to trigger the workflow.

- **Nodes Involved:**  
  - `Line Chatbot`

- **Node Details:**  
  - **Type:** Webhook  
  - **Role:** Entry point for incoming LINE messages.  
  - **Configuration:**  
    - HTTP Method: POST  
    - Path: `AIChatbot` (used as webhook endpoint)  
    - No additional options enabled.  
  - **Input/Output:**  
    - Input: HTTP POST from LINE platform.  
    - Output: JSON payload containing message event data.  
  - **Version Requirements:** n8n webhook node v2.  
  - **Edge Cases:**  
    - Invalid webhook URL or misconfiguration in LINE Developer Console will prevent message reception.  
    - Payload format changes from LINE API could break parsing.  
  - **Sticky Note:**  
    - Instructions to set up webhook URL in LINE Developer Console, including removing 'test' suffix for production.  
    - Reference: https://developers.line.biz/en/docs/messaging-api/receiving-messages/

#### 2.2 User Feedback (Loading Animation)

- **Overview:**  
  Sends a loading animation to the user on LINE to indicate that the chatbot is processing their input, improving user experience by reducing perceived latency.

- **Nodes Involved:**  
  - `Loading Animation`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Calls LINE Messaging API to start a loading animation for the user’s chat session.  
  - **Configuration:**  
    - URL: `https://api.line.me/v2/bot/chat/loading/start`  
    - Method: POST  
    - Body: JSON containing `chatId` extracted dynamically from the webhook payload (`userId`) and `loadingSeconds` set to 60 seconds.  
    - Authentication: Header Authorization via stored credential named `Talking Therapy Line@`.  
  - **Input/Output:**  
    - Input: JSON from `Line Chatbot` node.  
    - Output: HTTP response from LINE API (not further processed).  
  - **Edge Cases:**  
    - Authorization failure if token is invalid or expired.  
    - Network timeout or LINE API downtime.  
  - **Sticky Note:**  
    - Explains the importance of loading animation for user reassurance.  
    - Authorization instructions and link: https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/

#### 2.3 Message Type Validation

- **Overview:**  
  Determines if the incoming message is text. If yes, proceeds to AI processing; if not, sends a polite reply indicating unsupported input type.

- **Nodes Involved:**  
  - `Check Message Type IsText?` (If node)  
  - `ReplyMessage - Not supported`

- **Node Details:**  
  - **Check Message Type IsText?**  
    - Type: If  
    - Role: Conditional branching based on message type.  
    - Configuration: Checks if `body.events[0].message.type` equals `"text"`.  
    - Input: Output from `Loading Animation`.  
    - Output:  
      - True branch: proceeds to `AI Agent`.  
      - False branch: proceeds to `ReplyMessage - Not supported`.  
    - Edge Cases:  
      - Missing or malformed message type field could cause expression failure.  
  - **ReplyMessage - Not supported**  
    - Type: HTTP Request  
    - Role: Sends a reply message to LINE user informing that non-text inputs are unsupported.  
    - Configuration:  
      - URL: `https://api.line.me/v2/bot/message/reply`  
      - Method: POST  
      - Body: JSON with reply token and a fixed text message.  
      - Authentication: Header Authorization with Bearer token (hardcoded in header parameters).  
    - Input: Reply token extracted from `Line Chatbot` node.  
    - Output: HTTP response from LINE API.  
    - Edge Cases:  
      - Authorization failure or invalid reply token.  
      - User sends unsupported media types (images, stickers, etc.).  
    - Sticky Note:  
      - Clarifies that non-text inputs are not processed and users are notified accordingly.

#### 2.4 AI Processing (Chat Model & Agent)

- **Overview:**  
  Processes the user’s text input through an AI agent configured as a CBT therapist, using Azure OpenAI GPT-4 to generate empathetic and therapeutic responses.

- **Nodes Involved:**  
  - `AI Agent`  
  - `Azure OpenAI Chat Model`

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Acts as the conversational AI agent, managing prompt construction and interaction with the language model.  
    - Configuration:  
      - Input text: Extracted from webhook message text field.  
      - System message: Detailed CBT therapist persona prompt guiding the AI’s behavior and tone, including references and therapy principles.  
      - Character limit: Response constrained under 500 characters.  
      - Prompt type: Defined prompt.  
    - Input: Text from `Line Chatbot` node (via `Check Message Type IsText?` true branch).  
    - Output: Raw AI-generated text response.  
    - Edge Cases:  
      - Model timeout or API errors.  
      - System prompt misconfiguration leading to irrelevant responses.  
      - Character limit enforcement issues.  
    - Sticky Note:  
      - Explains how to configure system prompt and model parameters.  
      - Reference: https://davoy.tech/how-to-use-azure-openai-2/  
  - **Azure OpenAI Chat Model**  
    - Type: Langchain Azure OpenAI Chat Model Node  
    - Role: Provides GPT-4 model inference for the AI Agent.  
    - Configuration:  
      - Model: `4o` (GPT-4o)  
      - Credentials: Azure OpenAI API credentials stored securely.  
    - Input: Connected as language model to `AI Agent`.  
    - Output: AI-generated text passed back to `AI Agent`.  
    - Edge Cases:  
      - API key invalid or quota exceeded.  
      - Network or service downtime.

#### 2.5 Response Formatting

- **Overview:**  
  Cleans and formats the AI-generated response to ensure it is valid JSON and suitable for sending back through LINE’s messaging API.

- **Nodes Involved:**  
  - `Format Reply`

- **Node Details:**  
  - **Type:** Set Node  
  - **Role:** Processes the AI Agent’s output string to remove newlines, markdown, HTML tags, and quotes that could break JSON formatting.  
  - **Configuration:**  
    - Assigns a new field `output` with the cleaned string.  
    - Uses chained string operations: replace newlines with escaped newlines, remove markdown and tags, remove quotes.  
  - **Input:** Raw AI response from `AI Agent`.  
  - **Output:** Cleaned text string for reply.  
  - **Edge Cases:**  
    - Unexpected characters or formatting in AI output could cause incomplete cleaning.  
  - **Sticky Note:**  
    - Notes that AI output is not JSON-ready and requires formatting before reply.

#### 2.6 Reply to User

- **Overview:**  
  Sends the final formatted text response back to the user on LINE using the reply token, completing the conversational loop.

- **Nodes Involved:**  
  - `ReplyMessage - Line`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Calls LINE Messaging API to reply to the user’s message.  
  - **Configuration:**  
    - URL: `https://api.line.me/v2/bot/message/reply`  
    - Method: POST  
    - Body: JSON containing reply token and the cleaned AI response text.  
    - Authentication: Header Authorization via stored credential `Talking Therapy Line@`.  
  - **Input:** Reply token from `Line Chatbot` node and formatted text from `Format Reply`.  
  - **Output:** HTTP response from LINE API.  
  - **Edge Cases:**  
    - Expired or invalid reply token.  
    - Authorization failure.  
  - **Sticky Note:**  
    - Explains usage of reply token to avoid broadcast quota consumption.  
    - Instructions for setting up header authorization.  
    - Reference: https://developers.line.biz/en/docs/messaging-api/sending-messages/

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                         |
|---------------------------|----------------------------------|----------------------------------------|-------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Line Chatbot              | Webhook                          | Receives incoming LINE messages        | (Webhook HTTP POST)      | Loading Animation        | Set webhook URL in LINE Developer Console; remove 'test' for production; see LINE docs link       |
| Loading Animation         | HTTP Request                    | Sends loading animation to user        | Line Chatbot            | Check Message Type IsText? | Loading animation reassures user; requires header auth; see LINE docs link                        |
| Check Message Type IsText?| If                              | Checks if message is text               | Loading Animation       | AI Agent (true), ReplyMessage - Not supported (false) | Non-text inputs not processed; polite notification sent                                         |
| ReplyMessage - Not supported | HTTP Request                  | Replies to user for unsupported input  | Check Message Type IsText? (false) |                          | Non-text inputs not supported message                                                            |
| AI Agent                 | Langchain Agent                  | Processes text input with CBT persona  | Check Message Type IsText? (true), Azure OpenAI Chat Model | Format Reply             | Configure system prompt for CBT therapist; see Azure OpenAI usage notes                           |
| Azure OpenAI Chat Model  | Langchain Azure OpenAI Chat Model | Provides GPT-4 inference                | AI Agent (language model) | AI Agent                 | Requires Azure OpenAI credentials                                                                |
| Format Reply             | Set                             | Cleans AI output for JSON compatibility| AI Agent                | ReplyMessage - Line      | AI output needs formatting before reply                                                          |
| ReplyMessage - Line      | HTTP Request                    | Sends reply message back to user       | Format Reply            |                          | Use reply token to avoid broadcast quota; header auth setup instructions; see LINE docs link     |
| Sticky Note              | Sticky Note                     | Documentation and instructions         |                         |                          | Various notes linked to nodes as detailed above                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: `Line Chatbot`**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `AIChatbot`  
   - Save and copy the generated webhook URL.  
   - Configure this URL in LINE Developer Console as the webhook endpoint. Remove any 'test' suffix when moving to production.

2. **Create HTTP Request Node: `Loading Animation`**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/chat/loading/start`  
   - Body (Raw JSON):  
     ```json
     {
       "chatId": "{{ $json.body.events[0].source.userId }}",
       "loadingSeconds": 60
     }
     ```  
   - Content-Type: application/json  
   - Authentication: Generic Credential with HTTP Header Auth  
     - Create credential named e.g. `Talking Therapy Line@`  
     - Header: `Authorization: Bearer <Your LINE Channel Access Token>`  
   - Connect output of `Line Chatbot` to input of `Loading Animation`.

3. **Create If Node: `Check Message Type IsText?`**  
   - Type: If  
   - Condition:  
     - Expression: `{{$json.body.events[0].message.type}}`  
     - Operator: Equals  
     - Value: `text`  
   - Connect output of `Loading Animation` to input of this node.

4. **Create HTTP Request Node: `ReplyMessage - Not supported`**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/reply`  
   - Body (Raw JSON):  
     ```json
     {
       "replyToken": "{{ $('Line Chatbot').item.json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "Currently, the input of image or other type are not supported."
         }
       ]
     }
     ```  
   - Content-Type: application/json  
   - Headers:  
     - Authorization: Bearer `<Your LINE Channel Access Token>` (can be hardcoded or via credential)  
   - Connect the false output of `Check Message Type IsText?` to this node.

5. **Create Langchain Azure OpenAI Chat Model Node: `Azure OpenAI Chat Model`**  
   - Type: Langchain Azure OpenAI Chat Model  
   - Model: `4o` (GPT-4o)  
   - Credentials: Create and assign Azure OpenAI API credentials with your API key and endpoint.

6. **Create Langchain Agent Node: `AI Agent`**  
   - Type: Langchain Agent  
   - Text Input: `={{ $('Line Chatbot').item.json.body.events[0].message.text }}`  
   - System Message: Paste the detailed CBT therapist prompt provided in the original workflow (guiding the AI to act as a CBT therapist).  
   - Prompt Type: Define  
   - Connect the `Azure OpenAI Chat Model` node as the language model input to this node.  
   - Connect the true output of `Check Message Type IsText?` to this node.

7. **Create Set Node: `Format Reply`**  
   - Type: Set  
   - Add a new field `output` (string) with expression:  
     ```javascript
     {{$json.output.replaceAll("\n","\\n").replaceAll("\n","").removeMarkdown().removeTags().replaceAll('"',"")}}
     ```  
   - Connect output of `AI Agent` to input of `Format Reply`.

8. **Create HTTP Request Node: `ReplyMessage - Line`**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/reply`  
   - Body (JSON):  
     ```json
     {
       "replyToken": "{{ $('Line Chatbot').item.json.body.events[0].replyToken }}",
       "messages": [
         {
           "type": "text",
           "text": "{{ $json.output }}"
         }
       ]
     }
     ```  
   - Content-Type: application/json  
   - Authentication: Generic Credential with HTTP Header Auth  
     - Use the same credential as `Loading Animation` node (`Talking Therapy Line@`) or create a new one with your LINE Channel Access Token.  
   - Connect output of `Format Reply` to input of this node.

9. **Verify Connections:**  
   - `Line Chatbot` → `Loading Animation` → `Check Message Type IsText?`  
   - `Check Message Type IsText?` true → `AI Agent` → `Format Reply` → `ReplyMessage - Line`  
   - `Check Message Type IsText?` false → `ReplyMessage - Not supported`

10. **Activate Workflow and Test:**  
    - Deploy the workflow.  
    - Send text messages via LINE to your bot and verify responses.  
    - Send non-text messages to confirm the unsupported message reply.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook setup is critical: copy the webhook URL from the `Line Chatbot` node and configure it in the LINE Developer Console. Remove any 'test' suffix before production deployment.                                                                                                                                                                                        | https://developers.line.biz/en/docs/messaging-api/receiving-messages/                                            |
| Loading animation improves user experience by signaling that the bot is processing the request. Requires header authorization with a valid LINE Channel Access Token.                                                                                                                                                                                                    | https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/                                         |
| The AI Agent is configured with a detailed Cognitive Behavioral Therapy (CBT) system prompt to simulate a CBT therapist persona, providing empathetic and structured mental health support.                                                                                                                                                                               | https://www.rcpsych.ac.uk/mental-health/treatments-and-wellbeing/cognitive-behavioural-therapy-(cbt)             |
| Azure OpenAI GPT-4o model is used for AI inference; ensure you have valid Azure OpenAI credentials and quota.                                                                                                                                                                                                                                                           | https://davoy.tech/how-to-use-azure-openai-2/                                                                    |
| Reply messages use the reply token to avoid consuming broadcast quota and require header authorization with a Bearer token.                                                                                                                                                                                                                                             | https://developers.line.biz/en/docs/messaging-api/sending-messages/                                              |
| Non-text inputs (images, stickers, etc.) are currently not supported and users receive a polite notification message.                                                                                                                                                                                                                                                    |                                                                                                                  |

---

This documentation provides a detailed, stepwise understanding of the counseling chatbot workflow on LINE, enabling developers and AI agents to reproduce, modify, and troubleshoot the system effectively.