Build a Telegram Chatbot with Gemini Pro AI and Google Docs Integration

https://n8nworkflows.xyz/workflows/build-a-telegram-chatbot-with-gemini-pro-ai-and-google-docs-integration-8615


# Build a Telegram Chatbot with Gemini Pro AI and Google Docs Integration

### 1. Workflow Overview

This n8n workflow builds an AI-powered Telegram chatbot integrating Google Gemini Pro (Google PaLM API) and Google Docs. It listens for incoming Telegram messages, processes them through Google Gemini AI models to generate user-friendly replies, and sends back responses via Telegram. Additionally, it optionally fetches content from Google Docs and performs external HTTP requests. The workflow incorporates batching and delay mechanisms to handle multiple messages efficiently and respect rate limits.

**Logical Blocks:**

- **1.1 Input Reception**: Listens for Telegram messages using the Telegram Trigger node.
- **1.2 Message Batching**: Processes incoming messages in batches to handle multiple messages smoothly.
- **1.3 AI Processing**: Sends messages to Google Gemini AI models for generating human-friendly replies.
- **1.4 Response Delivery**: Sends AI-generated responses back to Telegram users.
- **1.5 Flow Control**: Implements a wait/delay node to pace the message processing.
- **1.6 Optional External Data Fetching**: Makes HTTP requests to external APIs.
- **1.7 Optional Document Retrieval**: Fetches content from Google Docs to enrich AI responses or provide static content.
- **1.8 Alternative AI Model Usage**: Uses an alternate Google Gemini model for testing or faster responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview**: This block initiates the workflow by triggering on new messages sent to the Telegram bot.
- **Nodes Involved**: `Telegram Trigger`
  
**Node Details:**

- **Telegram Trigger**
  - Type: Telegram Trigger node
  - Role: Listens for incoming Telegram messages to start the workflow.
  - Configuration: Listens for update type "message".
  - Expressions/Variables: Captures user message payload and sender info.
  - Input: External webhook triggered by Telegram.
  - Output: Emits incoming message data as JSON.
  - Edge cases: Possible failure if webhook is not properly registered or credentials invalid.
  - No sub-workflow.

#### 2.2 Message Batching

- **Overview**: Processes incoming messages in batches to handle multiple messages effectively and enable pacing.
- **Nodes Involved**: `Loop Over Items1`
  
**Node Details:**

- **Loop Over Items1**
  - Type: SplitInBatches node
  - Role: Splits incoming batch of messages into individual items for sequential processing.
  - Configuration: Default batching (no custom batch size specified).
  - Input: Takes the output from Telegram Trigger.
  - Output: Emits individual messages one by one.
  - Edge cases: Large message volumes might cause delays; batch size can be tuned.
  - No sub-workflow.

#### 2.3 AI Processing

- **Overview**: Sends each user message to a Google Gemini AI model to generate a conversational, user-friendly reply.
- **Nodes Involved**: `Message a model`, `Message a model in Google Gemini`
  
**Node Details:**

- **Message a model**
  - Type: Langchain Google Gemini node
  - Role: Sends prompt to the Google Gemini Pro AI (model "models/gemini-2.5-pro") to generate a reply.
  - Configuration: Uses modelId "models/gemini-2.5-pro"; sends a test message "Replay test write user-friendly." (likely a placeholder).
  - Input: Accepts message payload forwarded through the batch and optionally from other nodes.
  - Output: Produces AI-generated content in a structured JSON format.
  - Edge cases: API authentication errors, request timeouts, malformed inputs, model unavailability.
  - No sub-workflow.

- **Message a model in Google Gemini**
  - Type: Google Gemini Tool node (alternate AI model)
  - Role: Sends prompt to an alternate Gemini AI model "models/gemini-2.0-flash-exp" for potentially faster or experimental responses.
  - Configuration: Model ID set to "models/gemini-2.0-flash-exp"; similar test message used.
  - Input: Accepts AI tool input connections, can be chained.
  - Output: Generates AI response data.
  - Edge cases: Same as above, plus potential differences in response quality.
  - No sub-workflow.

#### 2.4 Response Delivery

- **Overview**: Sends the AI-generated reply back to the Telegram user.
- **Nodes Involved**: `Send a text message`
  
**Node Details:**

- **Send a text message**
  - Type: Telegram node
  - Role: Sends text messages to users on Telegram.
  - Configuration: 
    - Text set via expression: `={{ $json.content.parts[0].text }}`, extracting the AI response text.
    - Chat ID dynamically extracted from the Telegram Trigger node: `={{ $('Telegram Trigger').item.json.message.from.id }}`
    - Reply markup set to "replyKeyboard" (keyboard options empty by default).
  - Input: Receives AI response JSON from `Message a model`.
  - Output: Confirms message sent, can trigger next node.
  - Edge cases: Telegram API rate limits, invalid chat ID, message formatting errors.
  - Credential: Requires Telegram Bot API credentials configured in n8n.

#### 2.5 Flow Control

- **Overview**: Adds a short delay between sending messages to avoid hitting rate limits and create natural interaction pacing.
- **Nodes Involved**: `Wait 1s`
  
**Node Details:**

- **Wait 1s**
  - Type: Wait node
  - Role: Pauses workflow execution for 1 second.
  - Configuration: Amount set to 1 second.
  - Input: Triggered after sending each Telegram message.
  - Output: Loops back to `Loop Over Items1` to process next message.
  - Edge cases: Excessive waiting may slow down throughput; insufficient waiting may cause API rate limits.

#### 2.6 Optional External Data Fetching

- **Overview**: Makes an HTTP request to an external URL, potentially for fetching additional context or data.
- **Nodes Involved**: `HTTP Request`
  
**Node Details:**

- **HTTP Request**
  - Type: HTTP Request node
  - Role: Sends a GET request to `https://devcodejourney.com/` (placeholder).
  - Configuration: Default GET method; no headers or authentication; tool description says "Replay test write user-friendly."
  - Input: Not connected in the main message flow, but outputs to AI processing node.
  - Output: HTTP response data.
  - Edge cases: Network errors, invalid URL, unexpected response formats.
  - No credentials required by default.

#### 2.7 Optional Document Retrieval

- **Overview**: Retrieves content from a specified Google Docs document to use as static text or context.
- **Nodes Involved**: `Get a document in Google Docs`
  
**Node Details:**

- **Get a document in Google Docs**
  - Type: Google Docs Tool node
  - Role: Fetches complete content from a Google Docs document via URL.
  - Configuration: Operation set to "get"; document URL provided.
  - Input: Not connected directly into the main trigger flow; outputs to AI processing node.
  - Output: Document content.
  - Edge cases: OAuth token expiration, document access permissions, invalid URL.
  - Credential: Requires Google Docs OAuth2 credentials.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                            | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                      |
|-------------------------------|--------------------------------|--------------------------------------------|-----------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger               | telegramTrigger                | Listens for incoming Telegram messages     | —                     | Loop Over Items1          | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| Loop Over Items1               | splitInBatches                 | Splits message batch into individual items | Telegram Trigger       | Message a model (main)    | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| Message a model                | langchain.googleGemini         | Sends prompt to Google Gemini Pro AI       | Loop Over Items1 (main) | Send a text message       | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| Send a text message            | telegram                      | Sends AI response back to Telegram user    | Message a model        | Wait 1s                   | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| Wait 1s                       | wait                          | Adds delay between responses                 | Send a text message    | Loop Over Items1          | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| HTTP Request                  | httpRequestTool               | Makes external HTTP request                  | — (not connected)      | Message a model (ai_tool) | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| Get a document in Google Docs | googleDocsTool                | Fetches content from Google Docs document   | — (not connected)      | Message a model (ai_tool) | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |
| Message a model in Google Gemini | googleGeminiTool           | Alternate Gemini AI model for responses     | — (ai_tool input)      | Message a model (ai_tool) | # 🤖 Telegram + Gemini + Google Docs Automation in n8n ... full overview and use case details  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Configure with your Telegram Bot credentials.
   - Set "Updates" to listen for "message".
   - This node starts the workflow upon receiving Telegram messages.

2. **Add SplitInBatches Node (Loop Over Items1)**
   - Type: SplitInBatches
   - Connect Telegram Trigger’s main output to this node.
   - Use default batch options to process messages individually.
   - This enables sequential processing and pacing.

3. **Add Google Gemini Node (Message a model)**
   - Type: Langchain Google Gemini
   - Connect SplitInBatches node’s main output to this node.
   - Set Model ID to `models/gemini-2.5-pro`.
   - Configure messages to send the user prompt (replace the placeholder test text with expression referencing input).
   - Configure Google Gemini API credentials.
   - This node sends user messages to AI for response generation.

4. **Add Telegram Node (Send a text message)**
   - Type: Telegram
   - Connect output from Google Gemini node to this node.
   - Set "Text" to expression: `={{ $json.content.parts[0].text }}` (to extract AI reply).
   - Set "Chat ID" to expression: `={{ $('Telegram Trigger').item.json.message.from.id }}`.
   - Use proper Telegram Bot credentials.
   - This node sends the AI-generated reply back to the user.

5. **Add Wait Node (Wait 1s)**
   - Type: Wait
   - Connect “Send a text message” main output to this node.
   - Set wait duration to 1 second.
   - Connect Wait node output back to the SplitInBatches node to continue loop processing.

6. **Optional: Add HTTP Request Node**
   - Type: HTTP Request
   - Configure URL to `https://devcodejourney.com/`.
   - No credentials required unless your API needs them.
   - Connect this node's output to the AI processing node’s "ai_tool" input if used.
   - This node can be used to fetch external data to enrich AI replies.

7. **Optional: Add Google Docs Tool Node**
   - Type: Google Docs Tool
   - Configure operation to "get".
   - Provide Google Docs document URL.
   - Configure Google Docs OAuth2 credentials.
   - Connect this node’s output to the AI processing node’s "ai_tool" input.
   - This allows the AI to access document content dynamically.

8. **Optional: Add Alternate Google Gemini Node**
   - Type: Google Gemini Tool
   - Set Model ID to `models/gemini-2.0-flash-exp`.
   - Connect its output to the main Google Gemini node’s “ai_tool” input.
   - This node enables testing different AI model versions.

9. **Ensure Credentials Are Set**
   - Telegram Bot API credentials
   - Google Gemini (PaLM) API credentials
   - Google Docs OAuth2 credentials

10. **Test Workflow**
    - Send messages via Telegram bot.
    - Monitor AI responses and delays.
    - Adjust batching size or wait time as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow integrates Telegram Bot API, Google Gemini (PaLM API), Google Docs API, and n8n for chatbot automation. | Workflow overview sticky note inside n8n describing main purpose and use cases.                              |
| Ensure all API credentials are properly configured in n8n before running the workflow to avoid authentication errors. | Credential configuration hint from sticky note and node setup.                                              |
| The delay node helps avoid Telegram rate limits and creates more natural-paced conversations.                 | Explanation found in workflow overview sticky note.                                                         |
| External HTTP Request and Google Docs nodes are optional and can be used to enrich AI responses.               | Optional usage explained in workflow overview sticky note.                                                  |
| The workflow can be extended with error handling and conditional logic for robustness and scalability.         | Suggested in sticky note for future improvements.                                                           |
| For more information on Google Gemini and PaLM API usage, consult Google Cloud official documentation.         | External documentation recommended for API details (not included in workflow).                               |
| The Telegram bot must have webhook set up correctly to receive messages in real-time.                          | Telegram API setup requirement implied by Telegram Trigger node configuration.                               |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.