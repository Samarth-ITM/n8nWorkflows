Build a Multichannel Customer Support AI Assistant with Chatwoot & OpenRouter

https://n8nworkflows.xyz/workflows/build-a-multichannel-customer-support-ai-assistant-with-chatwoot---openrouter-8260


# Build a Multichannel Customer Support AI Assistant with Chatwoot & OpenRouter

---

### 1. Workflow Overview

This workflow implements a **Multichannel Customer Support AI Assistant** integrated with **Chatwoot** and **OpenRouter** (LLM provider). Its main purpose is to automate customer support by receiving new messages from Chatwoot, retrieving conversation history, processing it into a compact format, generating an AI-based response, and sending this response back into the Chatwoot conversation.

**Target Use Cases:**
- Multichannel customer support across platforms like Telegram, Instagram, WhatsApp, Facebook, unified via Chatwoot.
- Automated AI assistance that understands conversation context without storing history locally.
- AI-powered help focused on Chatwoot API-related inquiries.

**Logical Blocks:**

- **1.1 Input Reception and Filtering**: Receiving incoming messages via Chatwoot webhook, extracting relevant data, and filtering for incoming messages only.
- **1.2 Conversation History Retrieval and Processing**: Fetching full conversation history from Chatwoot API and formatting it into a text string suitable for AI consumption.
- **1.3 AI Processing**: Using OpenRouter language model and a LangChain-based assistant node to generate a contextually relevant AI response based on conversation history and a knowledge base.
- **1.4 Sending AI Response**: Sending the generated AI response back to the Chatwoot conversation via API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:**  
This block receives webhook calls from Chatwoot upon message creation events, extracts and structures essential data fields, then filters to process only incoming messages (to avoid self-replies or loops).

- **Nodes Involved:**  
  - Chatwoot Webhook  
  - Squize Webhook Data (Set node)  
  - Check If Incoming Message (IF node)

- **Node Details:**

1. **Chatwoot Webhook**  
   - *Type:* Webhook Trigger  
   - *Role:* Entry point receiving POST requests from Chatwoot webhook on message creation.  
   - *Configuration:*  
     - HTTP Method: POST  
     - Path: Customizable (e.g., "your-path")  
   - *Inputs:* HTTP webhook request  
   - *Outputs:* Raw JSON payload from Chatwoot event  
   - *Edge Cases:* Missing webhook calls, invalid payloads, network issues  
   - *Notes:* Must configure Chatwoot webhook URL with this node URL and select "message created" event.

2. **Squize Webhook Data**  
   - *Type:* Set node  
   - *Role:* Extracts and restructures relevant fields from the webhook payload into a simplified JSON object named `data`.  
   - *Configuration:* Custom JSON using expressions to extract fields such as inbox ID, conversation ID, message content, message type, account info, attachments, labels, and metadata. Also replaces newlines in message content with escaped "\n" for safe downstream processing.  
   - *Inputs:* Raw webhook JSON from previous node  
   - *Outputs:* Simplified, normalized JSON object with key conversation/message fields under `data`.  
   - *Edge Cases:* Missing fields in webhook payload, malformed JSON data.

3. **Check If Incoming Message**  
   - *Type:* IF node  
   - *Role:* Filters messages to proceed only if `message_type` equals `"incoming"`. This prevents the workflow from reacting to outgoing or system messages, avoiding infinite response loops.  
   - *Configuration:* Condition: `{{$json.data.message_type}} === "incoming"`  
   - *Inputs:* Structured data from previous Set node  
   - *Outputs:*  
     - True: Continue processing (incoming message)  
     - False: Stop (non-incoming message)  
   - *Edge Cases:* Unexpected or missing `message_type` field.

---

#### 2.2 Conversation History Retrieval and Processing

- **Overview:**  
This block retrieves the entire message history of the current conversation using Chatwoot’s REST API and processes the messages into a concise text format suitable for input to the AI model.

- **Nodes Involved:**  
  - Load Chatwoot Conversation History (HTTP Request)  
  - Process Loaded History (Code)

- **Node Details:**

1. **Load Chatwoot Conversation History**  
   - *Type:* HTTP Request node  
   - *Role:* Calls Chatwoot API to get all messages for the specified conversation in the specified account.  
   - *Configuration:*  
     - URL constructed dynamically using:  
       `https://yourchatwooturl.com/api/v1/accounts/{{ account_id }}/conversations/{{ conv_id }}/messages`  
     - Headers include:  
       - `api_access_token`: Your Chatwoot API token (replace `"YOUR_ACCESS_TOKEN"`)  
       - `Content-Type`: `application/json`  
     - Method: GET (default)  
   - *Inputs:* Data from `Check If Incoming Message` node (filtered to incoming)  
   - *Outputs:* JSON response containing message array (`payload`)  
   - *Edge Cases:* API authentication errors, network timeouts, invalid account or conversation IDs, empty conversations.

2. **Process Loaded History**  
   - *Type:* Code node (JavaScript)  
   - *Role:* Parses the API response to transform the message array into a formatted conversation history string for AI input.  
   - *Key Logic:*  
     - Reads `messages` array from HTTP response payload.  
     - Maps each message to `{ role: "user" | "assistant", content: string }` based on sender type (`contact` = user, others assistant).  
     - Replaces any message content `/start` with `"Hello!"`.  
     - Filters out empty or whitespace-only messages.  
     - Joins messages into a single string with format: `[ROLE]: content` separated by `, \n`.  
   - *Inputs:* HTTP response from previous node  
   - *Outputs:* JSON object with `conversation_text` property containing the formatted string.  
   - *Edge Cases:* Empty message arrays, malformed API response, runtime errors during transformation (caught and default empty string returned).

---

#### 2.3 AI Processing

- **Overview:**  
Uses OpenRouter API via a LangChain AI assistant node to generate a response based on the processed conversation history and a predefined knowledge base about Chatwoot APIs.

- **Nodes Involved:**  
  - OpenRouter Chat Model (LLM Node)  
  - Chatwoot Assistant (Chain LLM Node)

- **Node Details:**

1. **OpenRouter Chat Model**  
   - *Type:* LangChain OpenRouter LLM node  
   - *Role:* Provides the language model backend for AI responses.  
   - *Configuration:*  
     - Uses stored OpenRouter API credentials.  
     - Default options, no custom prompt here (handled in the Chain LLM node).  
   - *Inputs:* None (used as AI engine for subsequent node)  
   - *Outputs:* LLM completions  
   - *Edge Cases:* API key missing or invalid, rate limits, network errors.

2. **Chatwoot Assistant**  
   - *Type:* LangChain Chain LLM node  
   - *Role:* Defines a structured prompt and conversation context for the AI to answer questions about Chatwoot APIs.  
   - *Configuration:*  
     - Input text: `conversation history` variable (from Process Loaded History node)  
     - Prompt includes:  
       - Role & goal: Friendly AI assistant focused solely on Chatwoot API info.  
       - Knowledge base: Detailed descriptions of Chatwoot Application, Client, and Platform APIs.  
       - Operating rules: Tone, knowledge constraints, fallback to official docs URL if question is outside scope.  
       - Input/Output format: Receives conversation history, outputs only user-facing message.  
     - Uses OpenRouter Chat Model node as LLM backend.  
   - *Inputs:* JSON with `conversation_text` from previous node  
   - *Outputs:* AI-generated response text (string)  
   - *Edge Cases:* LLM failures, API errors, empty input text.

---

#### 2.4 Sending AI Response

- **Overview:**  
Sends the AI-generated response back to the relevant Chatwoot conversation via the Chatwoot API, posting a new outgoing message.

- **Nodes Involved:**  
  - Send Message (HTTP Request)

- **Node Details:**

1. **Send Message**  
   - *Type:* HTTP Request node  
   - *Role:* Posts the AI-generated response as an outgoing message to Chatwoot.  
   - *Configuration:*  
     - URL dynamically constructed as:  
       `https://yourchatwooturl.com/api/v1/accounts/{{ account_id }}/conversations/{{ conv_id }}/messages`  
     - Method: POST  
     - Body parameters:  
       - `content`: AI response text (`$json.text`)  
       - `message_type`: `"outgoing"`  
       - `private`: `"false"`  
       - `content_type`: `"text"`  
       - `content_attributes`: empty JSON object (`{}`)  
     - Headers:  
       - `api_access_token`: Your Chatwoot API token  
   - *Inputs:* AI text output from Chatwoot Assistant node  
   - *Outputs:* API response from Chatwoot (message created confirmation)  
   - *Edge Cases:* API authentication failure, invalid conversation, network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                               | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                                         |
|-------------------------------|-------------------------------|-----------------------------------------------|------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Chatwoot Webhook              | Webhook                       | Receives Chatwoot message webhook             | (Webhook trigger)             | Squize Webhook Data                 | See Sticky Note7: Overview of the entire workflow, use case, and instructions.                                      |
| Squize Webhook Data           | Set                           | Extracts and normalizes webhook payload data  | Chatwoot Webhook             | Check If Incoming Message           | Sticky Note6: Data reception and filtering explanation.                                                             |
| Check If Incoming Message     | IF                            | Filters only incoming messages                 | Squize Webhook Data          | Load Chatwoot Conversation History | Sticky Note6: Explains importance of filtering incoming messages only.                                              |
| Load Chatwoot Conversation History | HTTP Request            | Retrieves conversation history via Chatwoot API | Check If Incoming Message    | Process Loaded History              | Sticky Note: Details about loading and processing conversation history for AI input.                                 |
| Process Loaded History        | Code                          | Transforms API response into AI input string  | Load Chatwoot Conversation History | Chatwoot Assistant             | Sticky Note: Explains processing and formatting logic for conversation history.                                      |
| OpenRouter Chat Model         | LangChain OpenRouter LLM      | Provides LLM backend for AI responses          | (None, backend for Chatwoot Assistant) | Chatwoot Assistant           | Sticky Note1: Describes AI assistant node and how it uses the language model.                                        |
| Chatwoot Assistant            | LangChain Chain LLM           | Generates AI response to user question         | Process Loaded History, OpenRouter Chat Model | Send Message           | Sticky Note1: Also provides knowledge base and prompt structure for AI assistant.                                    |
| Send Message                 | HTTP Request                  | Sends AI-generated response back to Chatwoot  | Chatwoot Assistant          | (End)                             | Sticky Note2: Explains the final message dispatch to Chatwoot.                                                       |
| Sticky Note7                 | Sticky Note                   | Overview & instructions for entire workflow    | (None)                      | (None)                            | See content for full project description, use cases, and setup instructions.                                        |
| Sticky Note6                 | Sticky Note                   | Explanation for receiving and filtering data   | (None)                      | (None)                            | See content about webhook data filtering and importance of incoming message check.                                   |
| Sticky Note                  | Sticky Note                   | Explanation of conversation history loading     | (None)                      | (None)                            | Details on loading and formatting conversation history for AI input.                                                |
| Sticky Note1                 | Sticky Note                   | Explanation of AI assistant node and prompt     | (None)                      | (None)                            | Provides context on prompt design and possible extensions like RAG or advanced prompting.                            |
| Sticky Note2                 | Sticky Note                   | Explanation for sending AI response back        | (None)                      | (None)                            | Details about sending AI response to Chatwoot conversation.                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node: "Chatwoot Webhook"**  
   - Type: Webhook (Trigger)  
   - HTTP Method: POST  
   - Path: Choose a unique path (e.g., "your-path")  
   - Purpose: Receive message created events from Chatwoot.

2. **Create a Set Node: "Squize Webhook Data"**  
   - Purpose: Extract and normalize relevant data from webhook payload.  
   - Mode: Raw JSON  
   - JSON Output (example):  
     ```json
     {
       "data": {
         "inbox_id": "={{ $json.body.inbox.id }}",
         "inbox_title": "={{ $json.body.inbox.name }}",
         "conv_id": "={{ $json.body.conversation.id }}",
         "channel_type": "={{ $json.body.conversation.channel }}",
         "message_id": "={{ $json.body.id }}",
         "message_content": "={{ $json.body.content.replace(/\\r?\\n/g, '\\\\n') }}",
         "message_type": "={{ $json.body.message_type }}",
         "private": "={{ $json.body.private }}",
         "account_id": "={{ $json.body.account.id }}",
         "account_name": "={{ $json.body.account.name }}",
         "content_type": "={{ $json.body.content_type }}",
         "created_at": "={{ $json.body.created_at }}",
         "event": "={{ $json.body.event }}",
         "conversation_labels": "={{ $json.body.conversation.labels || [] }}",
         "attachments": "={{ $json.body.attachments || [] }}",
         "meta": "={{ $json.body.conversation.meta || null }}"
       }
     }
     ```
   - Connect output of "Chatwoot Webhook" to this node.

3. **Create an IF Node: "Check If Incoming Message"**  
   - Condition:  
     - Type: String  
     - Left Value: `={{ $json.data.message_type }}`  
     - Operator: Equals  
     - Right Value: `"incoming"`  
   - Connect output of "Squize Webhook Data" to this node.

4. **Create an HTTP Request Node: "Load Chatwoot Conversation History"**  
   - Method: GET  
   - URL:  
     `https://yourchatwooturl.com/api/v1/accounts/{{ $json.data.account_id }}/conversations/{{ $json.data.conv_id }}/messages`  
   - Headers:  
     - `api_access_token`: Your Chatwoot API Token (replace placeholder)  
     - `Content-Type`: `application/json`  
   - Connect the "true" output of "Check If Incoming Message" to this node.

5. **Create a Code Node: "Process Loaded History"**  
   - Language: JavaScript  
   - Paste the provided JS code that:  
     - Extracts messages array from payload  
     - Maps roles (user/assistant) based on sender type  
     - Replaces `/start` with `"Hello!"`  
     - Filters empty messages  
     - Joins into formatted string `[ROLE]: content` separated by comma and newline  
     - Outputs `{ conversation_text: string }`  
   - Connect output of "Load Chatwoot Conversation History" to this node.

6. **Create an OpenRouter LLM Node: "OpenRouter Chat Model"**  
   - Credentials: Configure with your OpenRouter API key  
   - No additional parameters needed here.  
   - This node serves as the backend LLM for the assistant.

7. **Create a Chain LLM Node: "Chatwoot Assistant"**  
   - Input: Expression referencing `conversation_text` from "Process Loaded History"  
   - Configure prompt messages:  
     - System / user messages setting the assistant role, knowledge base, operating rules, and input/output format as given.  
   - Link AI backend to "OpenRouter Chat Model".  
   - Connect output of "Process Loaded History" to this node as input.  
   - Connect "OpenRouter Chat Model" node as the AI language model backend.

8. **Create an HTTP Request Node: "Send Message"**  
   - Method: POST  
   - URL:  
     `https://yourchatwooturl.com/api/v1/accounts/{{ $json.data.account_id }}/conversations/{{ $json.data.conv_id }}/messages`  
   - Headers:  
     - `api_access_token`: Your Chatwoot API Token (replace placeholder)  
   - Body (JSON):  
     ```json
     {
       "content": "={{ $json.text }}",
       "message_type": "outgoing",
       "private": "false",
       "content_type": "text",
       "content_attributes": "{}"
     }
     ```  
   - Connect output of "Chatwoot Assistant" to this node.

9. **Connect nodes according to the flow:**  
   - Chatwoot Webhook → Squize Webhook Data → Check If Incoming Message (true path) → Load Chatwoot Conversation History → Process Loaded History → Chatwoot Assistant → Send Message  
   - Check If Incoming Message (false path) is left unconnected (stops flow).

10. **Credentials Setup:**  
    - OpenRouter API credentials configured in "OpenRouter Chat Model" node.  
    - Chatwoot API access token configured in HTTP Request nodes ("Load Chatwoot Conversation History" and "Send Message").

11. **Additional Setup:**  
    - Configure Chatwoot webhook to point to the webhook node URL (e.g., `https://your-n8n-domain/webhook/your-path`) and subscribe to "message created" event.  
    - Replace all placeholder URLs and tokens with your actual Chatwoot instance details and API tokens.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Multichannel AI Assistant Demo for Chatwoot enables AI-powered support over multiple messaging channels unified in Chatwoot.                                                                                                        | Sticky Note7: Full project description and instructions.                                       |
| To avoid infinite reply loops, the workflow filters only incoming messages before processing.                                                                                                                                       | Sticky Note6: Explains importance of filtering incoming messages.                              |
| Conversation history is retrieved dynamically via API and processed into a simple text format for efficient and clean AI input.                                                                                                    | Sticky Note (near history processing nodes).                                                  |
| AI assistant prompt is designed with a knowledge base strictly about Chatwoot APIs and strict fallback rules to official docs.                                                                                                     | Sticky Note1: Prompt design and potential advanced extensions.                                |
| Final AI responses are sent back as outgoing messages via Chatwoot API, enabling seamless conversational AI support.                                                                                                               | Sticky Note2: Final message sending explanation.                                              |
| Official Chatwoot API docs for reference: https://developers.chatwoot.com/introduction                                                                                                                                               | Referenced in prompt and fallback instructions.                                               |
| OpenRouter API key or equivalent LLM provider credentials required for AI node to function.                                                                                                                                         | Credential requirement in AI nodes.                                                          |
| Chatwoot API token must have appropriate permissions to read conversations and post messages.                                                                                                                                       | Credential requirement in HTTP Request nodes.                                                |
| Contact for support or further development help: Telegram @ninesfork                                                                                                                                                                | Provided in Sticky Note7.                                                                    |

---

**Disclaimer:**  
The content above derives exclusively from an n8n workflow automation built with publicly available APIs and tools. It contains no illegal or protected material. All data processed is legal and public.

---