Build an AI agent for WhatsApp with WOZTELL and Google Gemini

https://n8nworkflows.xyz/workflows/build-an-ai-agent-for-whatsapp-with-woztell-and-google-gemini-13641


# Build an AI agent for WhatsApp with WOZTELL and Google Gemini

# Workflow Reference: Build an AI Agent for WhatsApp with WOZTELL and Google Gemini

## 1. Workflow Overview
This workflow automates WhatsApp customer service by integrating the WOZTELL messaging gateway with Google Gemini's AI capabilities. When a customer sends a message, the system retrieves the previous conversation context, generates a professional and contextually relevant response using an AI agent, and delivers it back to the customer via WhatsApp.

The logic is organized into four main functional blocks:
1.  **Input Reception & Validation:** Captures incoming messages and filters for relevant text-based inquiries.
2.  **Context Retrieval & Formatting:** Fetches previous chat history to provide context to the AI.
3.  **AI Intelligence:** Processes the conversation history and generates a reply.
4.  **Response Delivery:** Sends the finalized message back to the customer.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Validation
**Overview:** This block acts as the entry point, receiving webhooks from WOZTELL and ensuring that only specific "Inbound" "Text" messages are processed, while ignoring active live chat sessions.

*   **Webhook (n8n-nodes-base.webhook):**
    *   **Role:** Listens for POST requests at the `/inbound/message` path.
    *   **Configuration:** HTTP Method set to POST.
    *   **Failure Modes:** Incorrect Production URL in WOZTELL settings; timeout if high volume without scaling.
*   **Filter (n8n-nodes-base.filter):**
    *   **Role:** Logic gate to ensure the AI only handles automated text queries.
    *   **Configuration:**
        *   `eventType` must equal `inbound`.
        *   `type` must equal `text`.
        *   `memberExtra.liveChat` must be `false` (prevents AI from interfering with human agents).
    *   **Output:** Only continues if all conditions are met.

### 2.2 Context Retrieval & Preparation
**Overview:** Retrieves the last 100 messages from the specific user and formats them into a single string that an AI model can understand.

*   **Get conversation history by id (woztell-sanuker.woztell):**
    *   **Role:** Fetches the interaction history for the specific WhatsApp user.
    *   **Configuration:** Uses the `from` field from the Webhook as the `externalId`.
    *   **Requirements:** Requires WOZTELL credentials with `member:getConversations` permissions.
*   **Edit Fields (n8n-nodes-base.set):**
    *   **Role:** Data transformation.
    *   **Key Expressions:** 
        *   `messages`: A JavaScript snippet that maps the WOZTELL history into a readable format: `Customer: [Text] \n Agent: [Text]`.
        *   `member`: Extracts the `memberId`.
    *   **Output:** A structured string containing the conversation transcript.

### 2.3 AI Intelligence
**Overview:** The "brain" of the workflow, which uses the LLM to decide what to say based on the formatted context.

*   **AI Agent (@n8n/n8n-nodes-langchain.agent):**
    *   **Role:** Executes the prompt using the provided model.
    *   **Configuration:** 
        *   **Prompt:** Uses the "messages" variable from the previous node.
        *   **System Message:** Instructs the AI to be a professional analyst, generate a single concise reply, and output **only** plain text.
*   **Google Gemini Chat Model (@n8n/n8n-nodes-langchain.lmChatGoogleGemini):**
    *   **Role:** The Language Model (LLM) provider.
    *   **Configuration:** `maxOutputTokens` set to 1000.
    *   **Requirements:** Valid Google Gemini (PaLM) API Key.

### 2.4 Response Delivery
**Overview:** Transmits the AI-generated text back to the customer's WhatsApp device.

*   **Send responses (woztell-sanuker.woztell):**
    *   **Role:** Final action node.
    *   **Configuration:** 
        *   **Recipient ID:** Maps from the original Webhook `from` field.
        *   **Response Text:** Formats the AI output into a JSON object: `{"type": "TEXT", "text": $json.output}`.
    *   **Edge Cases:** Message delivery failure if the recipient's phone is offline or the WhatsApp session has expired (>24 hours for standard messages).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | n8n-nodes-base.webhook | Trigger | None | Filter | Receives incoming WhatsApp messages from WOZTELL. Check how to configure the webhook [here](https://support.woztell.com/portal/en/kb/articles/web#Create_Webhooks). |
| Filter | n8n-nodes-base.filter | Validation | Webhook | Get conversation... | Checks whether the incoming event meets all conditions: Event type is inbound, Message type is text, Live chat is not active. |
| Get conversation history by id | woztell-sanuker.woztell | Context Retrieval | Filter | Edit Fields | Retrieves previous conversation history. Reads up to 100 messages for the account to provide context to the AI. |
| Edit Fields | n8n-nodes-base.set | Formatting | Get conversation... | AI Agent | Formats the conversation for AI processing. Labels messages as Customer or Agent and combines them into a structured conversation block. |
| AI Agent | langchain.agent | AI Reasoning | Edit Fields | Send responses | Generates an AI reply. The system prompt defines tone and behavior. |
| Google Gemini Chat Model | langchain.lmChatGoogleGemini | LLM Provider | None | AI Agent | Google Gemini Chat Model provides the AI language model used to generate replies. [Documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.lmchatgooglegemini/). |
| Send responses | woztell-sanuker.woztell | Transmission | AI Agent | None | Sends the AI generated reply back to the customer via WhatsApp. The recipient ID is automatically taken from the inbound message. |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1.  **WOZTELL:** An account with a WhatsApp channel connected. Generate an Access Token with `channel:list`, `channel:getBasicInfo`, `member:getConversations`, and `bot:sendResponses` permissions.
2.  **Google Gemini:** An API Key from Google AI Studio.

### Step-by-Step Setup
1.  **Create Webhook:** Add a **Webhook** node. Set HTTP Method to `POST` and path to `inbound/message`.
2.  **Add Filter:** Connect a **Filter** node to the Webhook.
    *   Condition 1: `{{ $json.body.eventType }}` Equals `inbound`.
    *   Condition 2: `{{ $json.body.type }}` Equals `text`.
    *   Condition 3: `{{ $json.body.memberExtra.liveChat }}` Equals `false` (Boolean).
3.  **Retrieve History:** Add a **WOZTELL** node (Operation: `getConversationHistory`).
    *   Set `externalId` to `{{ $json.body.from }}`.
    *   Select your specific Channel in the dropdown.
4.  **Format Context:** Add an **Edit Fields** node.
    *   Create a field `messages` (String) with the expression:
        `={{ $json.edges.filter(item => item?.node?.messageEvent?.type === 'TEXT').map(item => ( [[item?.node?.from === 'MEMBER' ? 'Customer' : 'Agent'], item?.node?.messageEvent?.data?.text].join(':') )).join('\n') }}`
    *   Create a field `member` (String) with `={{ $json.edges[0].node.memberId }}`.
5.  **Configure AI:** Add an **AI Agent** node.
    *   Prompt: `={{ $json.messages }}`.
    *   System Message: "You are a Professional Conversation Analyst and Reply Generator Agent... Output ONLY the plain text of the reply."
6.  **Connect Model:** Attach a **Google Gemini Chat Model** node to the AI Agent. Provide your API credentials.
7.  **Send Message:** Add a final **WOZTELL** node (Operation: `sendResponses`).
    *   Recipient ID: `{{ $('Webhook').item.json.body.from }}`.
    *   Response Template (JSON):
        ```json
        {
          "type": "TEXT",
          "text": {{ JSON.stringify($json.output) }}
        }
        ```
8.  **Activation:** Copy the **Production URL** from the Webhook node and paste it into the WOZTELL platform under Inbound Webhook settings.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| WOZTELL Signup | [Sign up here](https://platform.woztell.com/signup?lang=en&utm_campaign=plugin-n8n&utm_medium=plugin-n8n&utm_source=N8N) |
| WhatsApp API Setup Guide | [WABA Setup Procedures](https://doc.woztell.com/docs/procedures/basic-whatsapp-chatbot-setup/standard-procedures-wa-connect-waba/) |
| Access Token Generation | [Token Tutorial](https://support.woztell.com/portal/en/kb/articles/access-token#Access_Token_Generation) |
| Channel Specific Tokens | [Channel Token Guide](https://support.woztell.com/portal/en/kb/articles/access-token-channels-in-woztell#How_to_Create_an_Access_Token_for_a_Channel_in_WOZTELL) |