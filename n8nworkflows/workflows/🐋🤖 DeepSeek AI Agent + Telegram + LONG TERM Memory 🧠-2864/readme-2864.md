🐋🤖 DeepSeek AI Agent + Telegram + LONG TERM Memory 🧠

https://n8nworkflows.xyz/workflows/-----deepseek-ai-agent---telegram---long-term-memory----2864


# 🐋🤖 DeepSeek AI Agent + Telegram + LONG TERM Memory 🧠

### 1. Workflow Overview

This workflow, titled **"🐋🤖 DeepSeek AI Agent + Telegram + LONG TERM Memory 🧠"**, integrates a DeepSeek AI conversational agent with Telegram messaging, enhanced by long-term memory capabilities stored in Google Docs. It is designed to provide personalized, context-aware AI responses to Telegram users by validating user identity, routing messages by type, invoking AI models for processing, and managing memory for continuity across sessions.

**Target Use Cases:**  
- Conversational AI bots on Telegram that remember user preferences and past interactions.  
- Personalized AI assistants that adapt responses based on stored long-term memories.  
- Handling multiple message types (text, audio, images) with appropriate routing and fallback.  

**Logical Blocks:**  
- **1.1 Input Reception and Validation:** Receive Telegram messages via webhook, validate user identity to authorize processing.  
- **1.2 Message Routing:** Classify incoming messages by type (text, audio, image) for tailored handling.  
- **1.3 Memory Retrieval:** Fetch stored long-term memories from Google Docs to provide context.  
- **1.4 AI Agent Processing:** Use DeepSeek AI models (DeepSeek-V3 Chat and DeepSeek-R1 Reasoning) with system prompts and memory context to generate responses.  
- **1.5 Memory Saving:** Store new relevant memories back to Google Docs for future personalization.  
- **1.6 Response Delivery and Error Handling:** Send AI-generated responses or error messages back to Telegram users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block receives incoming Telegram messages through a webhook and validates the user’s identity by comparing the incoming message’s user data with predefined authorized user data.

**Nodes Involved:**  
- Listen for Telegram Events (Webhook)  
- Validation (Set)  
- Check User & Chat ID (If)  
- Error message (Telegram)  

**Node Details:**  

- **Listen for Telegram Events**  
  - Type: Webhook  
  - Role: Entry point receiving Telegram updates via POST requests at path `/wbot`.  
  - Configuration: HTTP POST, binary data property named `data`.  
  - Inputs: External Telegram webhook calls.  
  - Outputs: JSON containing Telegram message payload.  
  - Edge Cases: Webhook misconfiguration, invalid payloads, network issues.

- **Validation**  
  - Type: Set  
  - Role: Defines authorized user data (first name, last name, ID) for validation.  
  - Configuration: Sets static values for `first_name`, `last_name`, and `id` (e.g., "FirstName", "LastName", 12345667891).  
  - Inputs: Output from webhook.  
  - Outputs: User data for comparison.  
  - Edge Cases: Hardcoded values require manual update for different users.

- **Check User & Chat ID**  
  - Type: If  
  - Role: Compares incoming Telegram user info with authorized user data.  
  - Configuration: Checks equality of first name, last name, and user ID between webhook data and Validation node data.  
  - Inputs: From Validation node.  
  - Outputs:  
    - True: Proceed to message routing.  
    - False: Trigger error message node.  
  - Edge Cases: Case sensitivity and strict type validation ensure exact matching; mismatches block processing.

- **Error message**  
  - Type: Telegram  
  - Role: Sends an error message "Unable to process your message." to the Telegram chat if validation fails.  
  - Configuration: Uses Telegram API credentials; sends message to chat ID from incoming message.  
  - Inputs: False branch from Check User & Chat ID.  
  - Outputs: None.  
  - Edge Cases: Telegram API errors, invalid chat ID.

---

#### 1.2 Message Routing

**Overview:**  
Routes incoming messages based on their content type (audio, text, image) to ensure appropriate processing paths.

**Nodes Involved:**  
- Message Router (Switch)  
- Merge (Combine)  
- Error message (Telegram)  

**Node Details:**  

- **Message Router**  
  - Type: Switch  
  - Role: Detects message type by checking existence of `voice`, `text`, or `photo` fields in Telegram message JSON.  
  - Configuration:  
    - Outputs: `audio` (if voice exists), `text` (if text exists), `image` (if photo exists), `extra` (fallback).  
  - Inputs: True branch from Check User & Chat ID.  
  - Outputs: Routes to different nodes based on message type.  
  - Edge Cases: Messages without recognized types go to fallback output; missing fields cause routing to fallback.

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from multiple nodes (e.g., AI Agent and memory retrieval) into a single stream for further processing.  
  - Configuration: Combine mode, combines all inputs.  
  - Inputs: From Message Router’s text output and Retrieve Long Term Memories node.  
  - Outputs: To AI Agent node.  
  - Edge Cases: If one input is missing, merge still proceeds; ensure data consistency.

- **Error message** (reused)  
  - Role: Sends error message if message type is unrecognized (fallback output).  
  - Inputs: Fallback output from Message Router.  
  - Edge Cases: Same as above.

---

#### 1.3 Memory Retrieval

**Overview:**  
Fetches long-term user memories stored in a Google Docs document to provide context for AI responses.

**Nodes Involved:**  
- Retrieve Long Term Memories (Google Docs)  
- Merge (Combine)  

**Node Details:**  

- **Retrieve Long Term Memories**  
  - Type: Google Docs  
  - Role: Retrieves content from a specified Google Docs document containing stored memories.  
  - Configuration: Operation set to "get" with a document URL (Google Doc ID placeholder).  
  - Credentials: Google Docs OAuth2.  
  - Inputs: From Message Router (text message path).  
  - Outputs: Document content as JSON field `content`.  
  - Edge Cases: Google Docs API errors, invalid document ID, permission issues.

- **Merge** (as above)  
  - Combines retrieved memories with message data for AI Agent input.

---

#### 1.4 AI Agent Processing

**Overview:**  
Processes the user’s text message using DeepSeek AI models with system prompts incorporating user identity and memory context. Supports two models: DeepSeek-V3 Chat for general conversation and DeepSeek-R1 Reasoning for advanced queries.

**Nodes Involved:**  
- AI Agent (Langchain Agent)  
- DeepSeek-V3 Chat (LM Chat OpenAI)  
- DeepSeek-R1 Reasoning (LM Chat OpenAI)  
- Window Buffer Memory (Memory Buffer Window)  

**Node Details:**  

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Central AI processing node that receives combined inputs (user message + memories) and generates responses.  
  - Configuration:  
    - Text input: User’s Telegram text message.  
    - System prompt: Defines AI role, rules for memory management, context awareness, user-centric responses, privacy, fallback responses, and tools usage.  
    - Memory tool: Connected to Save Long Term Memories node.  
    - Memory context: Injects recent memories from Google Docs.  
  - Inputs: From Merge node combining message and memories.  
  - Outputs:  
    - Main output: AI-generated response text.  
    - Error output: Continues workflow with error message node if AI processing fails.  
  - Edge Cases: API errors, prompt formatting issues, memory context overload.

- **DeepSeek-V3 Chat**  
  - Type: LM Chat OpenAI  
  - Role: Language model node configured to use DeepSeek’s chat model (`deepseek-chat`).  
  - Configuration: Model name set to `deepseek-chat`.  
  - Credentials: DeepSeek API key via OpenAI-compatible credential.  
  - Inputs: AI Agent’s language model input.  
  - Outputs: AI Agent receives model output.  
  - Edge Cases: API key invalid, rate limits, network errors.

- **DeepSeek-R1 Reasoning**  
  - Type: LM Chat OpenAI  
  - Role: Language model node configured for DeepSeek’s reasoning model (`deepseek-reasoner`).  
  - Configuration: Model name set to `deepseek-reasoner`.  
  - Credentials: Same as above.  
  - Inputs: AI Agent’s language model input (optional advanced reasoning).  
  - Outputs: AI Agent receives model output.  
  - Edge Cases: Same as above.

- **Window Buffer Memory**  
  - Type: Memory Buffer Window (Langchain)  
  - Role: Maintains session context with a sliding window of 50 messages, keyed by user ID.  
  - Configuration: Session key set to user ID from JSON, context window length 50.  
  - Inputs: Connected as AI memory input to AI Agent.  
  - Outputs: Provides memory context for AI Agent.  
  - Edge Cases: Memory overflow, session key missing.

---

#### 1.5 Memory Saving

**Overview:**  
Stores new summarized memories extracted by the AI Agent into the Google Docs document to maintain long-term memory.

**Nodes Involved:**  
- Save Long Term Memories (Google Docs Tool)  

**Node Details:**  

- **Save Long Term Memories**  
  - Type: Google Docs Tool  
  - Role: Inserts new memory entries into the Google Docs document.  
  - Configuration:  
    - Action: Insert text formatted as `Memory: {{ $fromAI('memory') }} - Date: {{ $now }}`.  
    - Document URL: Same Google Docs document as retrieval node.  
  - Credentials: Google Docs OAuth2.  
  - Inputs: AI Agent’s tool output for memory saving.  
  - Outputs: None (side-effect node).  
  - Edge Cases: API permission errors, document locking, formatting issues.

---

#### 1.6 Response Delivery and Error Handling

**Overview:**  
Sends the AI-generated response back to the Telegram user or an error message if processing fails.

**Nodes Involved:**  
- Telegram Response (Telegram)  
- Response Error message (Telegram)  

**Node Details:**  

- **Telegram Response**  
  - Type: Telegram  
  - Role: Sends AI-generated text response to the Telegram chat.  
  - Configuration:  
    - Text: AI Agent output field `output`.  
    - Chat ID: Extracted from incoming Telegram message.  
    - Parse mode: HTML enabled for rich text.  
    - Attribution: Disabled.  
  - Credentials: Telegram API.  
  - Inputs: AI Agent main output.  
  - Outputs: None.  
  - Edge Cases: Telegram API errors, invalid chat ID, message formatting issues.

- **Response Error message**  
  - Type: Telegram  
  - Role: Sends error message "Unable to process your message." if AI Agent fails.  
  - Configuration: Similar to Error message node.  
  - Inputs: AI Agent error output.  
  - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                         | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                   |
|---------------------------|--------------------------------|---------------------------------------|-----------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Listen for Telegram Events | Webhook                        | Receive Telegram messages              | External webhook calls       | Validation                    | # Receive Telegram Message with Webhook                                                     |
| Validation                | Set                            | Define authorized user data            | Listen for Telegram Events   | Check User & Chat ID          | ## Validate Telegram User                                                                   |
| Check User & Chat ID      | If                             | Validate user identity                  | Validation                  | Message Router, Error message | ## Validate Telegram User                                                                   |
| Error message            | Telegram                       | Send error if validation fails          | Check User & Chat ID (false) | None                         | ## Validate Telegram User                                                                   |
| Message Router           | Switch                        | Route messages by type                  | Check User & Chat ID (true) | Merge, Retrieve Long Term Memories, Error message | # Process Text Message                                                                       |
| Retrieve Long Term Memories | Google Docs                   | Fetch stored long-term memories         | Message Router (text output) | Merge                        | ## Retrieve Long Term Memories Google Docs                                                  |
| Merge                    | Merge                         | Combine message and memory data         | Message Router, Retrieve Long Term Memories | AI Agent                    |                                                                                              |
| AI Agent                 | Langchain Agent               | Process message with AI and memory      | Merge                       | Telegram Response, Response Error message, Save Long Term Memories |                                                                                              |
| DeepSeek-V3 Chat         | LM Chat OpenAI                | DeepSeek chat model for conversation    | AI Agent (lmChat input)      | AI Agent                     | # DeepSeek API Call                                                                         |
| DeepSeek-R1 Reasoning    | LM Chat OpenAI                | DeepSeek reasoning model for complex queries | AI Agent (lmChat input)      | AI Agent                     | # DeepSeek API Call                                                                         |
| Window Buffer Memory     | Memory Buffer Window          | Maintain session context window         | AI Agent (memory input)      | AI Agent                     | ## Save To Current Chat Memory (Optional)                                                  |
| Save Long Term Memories  | Google Docs Tool              | Save new memories to Google Docs         | AI Agent (tool output)       | AI Agent                     | ## Save Long Term Memories Google Docs                                                     |
| Telegram Response        | Telegram                      | Send AI response to Telegram chat        | AI Agent                    | None                        |                                                                                              |
| Response Error message   | Telegram                      | Send error message if AI processing fails | AI Agent (error output)      | None                        |                                                                                              |
| Sticky Note              | Sticky Note                   | Instructional notes                      | None                       | None                        | # Receive Telegram Message with Webhook                                                     |
| Sticky Note1             | Sticky Note                   | Telegram webhook setup instructions      | None                       | None                        | # How to set up a Telegram Bot WebHook                                                     |
| Sticky Note3             | Sticky Note                   | Notes on retrieving long-term memories   | None                       | None                        | ## Retrieve Long Term Memories Google Docs                                                  |
| Sticky Note4             | Sticky Note                   | Notes on user validation                  | None                       | None                        | ## Validate Telegram User                                                                   |
| Sticky Note5             | Sticky Note                   | Notes on saving current chat memory       | None                       | None                        | ## Save To Current Chat Memory (Optional)                                                  |
| Sticky Note6             | Sticky Note                   | Notes on saving long-term memories        | None                       | None                        | ## Save Long Term Memories Google Docs                                                     |
| Sticky Note10            | Sticky Note                   | Notes on processing text messages         | None                       | None                        | # Process Text Message                                                                       |
| Sticky Note12            | Sticky Note                   | DeepSeek API call and configuration notes | None                       | None                        | # DeepSeek API Call                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Listen for Telegram Events`  
   - HTTP Method: POST  
   - Path: `wbot`  
   - Binary Property Name: `data`  
   - Purpose: Receive Telegram updates via webhook.

2. **Create Set Node for Validation**  
   - Type: Set  
   - Name: `Validation`  
   - Assign static values:  
     - `first_name`: "FirstName"  
     - `last_name`: "LastName"  
     - `id`: 12345667891 (number)  
   - Pass through other fields.

3. **Create If Node for User Validation**  
   - Type: If  
   - Name: `Check User & Chat ID`  
   - Conditions (all must be true):  
     - Incoming message first_name equals Validation first_name  
     - Incoming message last_name equals Validation last_name  
     - Incoming message id equals Validation id  
   - Case sensitive, strict type validation.

4. **Create Telegram Node for Validation Failure**  
   - Type: Telegram  
   - Name: `Error message`  
   - Text: "Unable to process your message."  
   - Chat ID: Extract from incoming message JSON path `body.message.chat.id`  
   - Credentials: Telegram API OAuth2.

5. **Connect Nodes:**  
   - `Listen for Telegram Events` → `Validation` → `Check User & Chat ID`  
   - `Check User & Chat ID` True → Next block (Message Router)  
   - `Check User & Chat ID` False → `Error message`

6. **Create Switch Node for Message Routing**  
   - Type: Switch  
   - Name: `Message Router`  
   - Rules:  
     - Output `audio`: if `body.message.voice` exists  
     - Output `text`: if `body.message.text` exists  
     - Output `image`: if `body.message.photo` exists  
     - Fallback output: `extra`

7. **Create Google Docs Node to Retrieve Memories**  
   - Type: Google Docs  
   - Name: `Retrieve Long Term Memories`  
   - Operation: Get document content  
   - Document URL: Insert your Google Docs document ID URL  
   - Credentials: Google Docs OAuth2

8. **Create Merge Node**  
   - Type: Merge  
   - Name: `Merge`  
   - Mode: Combine all inputs  
   - Inputs: Connect from `Message Router` (text output) and `Retrieve Long Term Memories`

9. **Create Langchain Agent Node**  
   - Type: Langchain Agent  
   - Name: `AI Agent`  
   - Text input: `={{ $('Message Router').item.json.body.message.text }}`  
   - System Message: Use the detailed system prompt provided in the original workflow, including role, rules, memory management, and tools usage.  
   - Configure to use `Save Long Term Memories` as a tool.  
   - Connect `Merge` output to `AI Agent` input.

10. **Create LM Chat OpenAI Nodes for DeepSeek Models**  
    - Name: `DeepSeek-V3 Chat`  
      - Model: `deepseek-chat`  
      - Credentials: DeepSeek API key (OpenAI-compatible)  
      - Connect as language model input to `AI Agent`  
    - Name: `DeepSeek-R1 Reasoning`  
      - Model: `deepseek-reasoner`  
      - Credentials: Same as above  
      - Connect as language model input to `AI Agent`

11. **Create Memory Buffer Window Node**  
    - Type: Memory Buffer Window  
    - Name: `Window Buffer Memory`  
    - Session Key: `={{ $json.id }}` (user ID)  
    - Context Window Length: 50  
    - Connect as memory input to `AI Agent`

12. **Create Google Docs Tool Node to Save Memories**  
    - Type: Google Docs Tool  
    - Name: `Save Long Term Memories`  
    - Action: Insert text  
    - Text: `Memory: {{ $fromAI('memory') }} - Date: {{ $now }}`  
    - Document URL: Same as retrieval node  
    - Credentials: Google Docs OAuth2  
    - Connect as AI tool output from `AI Agent`

13. **Create Telegram Node to Send Responses**  
    - Type: Telegram  
    - Name: `Telegram Response`  
    - Text: `={{ $json.output }}` (AI Agent output)  
    - Chat ID: `={{ $('Listen for Telegram Events').item.json.body.message.chat.id }}`  
    - Parse Mode: HTML  
    - Credentials: Telegram API OAuth2  
    - Connect from `AI Agent` main output

14. **Create Telegram Node for AI Errors**  
    - Type: Telegram  
    - Name: `Response Error message`  
    - Text: "Unable to process your message."  
    - Chat ID: Same as above  
    - Credentials: Telegram API OAuth2  
    - Connect from `AI Agent` error output

15. **Connect Message Router Outputs:**  
    - `text` output → `Merge` and `Retrieve Long Term Memories`  
    - `audio` and `image` outputs: (Not connected in this workflow, can be extended)  
    - `extra` output → `Error message`

16. **Set Credentials:**  
    - Telegram API: OAuth2 credentials for bot token  
    - Google Docs OAuth2: Access to Google Docs document  
    - OpenAI-compatible API: DeepSeek API key configured as OpenAI API credentials

17. **Test the Workflow:**  
    - Deploy webhook URL to Telegram Bot API webhook setup  
    - Send messages from authorized Telegram user  
    - Verify AI responses and memory saving in Google Docs

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To set up Telegram webhook, use: `https://api.telegram.org/bot{my_bot_token}/setWebhook?url={url_to_send_updates_to}` and verify with `getWebhookInfo`.                                                                                                                                                                                                       | Sticky Note1 content in workflow                                                                |
| DeepSeek API is OpenAI-compatible; base URL: `https://api.deepseek.com`. Models: `deepseek-chat` (DeepSeek-V3 Chat), `deepseek-reasoner` (DeepSeek-R1 Reasoning).                                                                                                                                                                                              | Sticky Note12 content in workflow                                                               |
| Google Docs is used as long-term memory storage; ensure OAuth2 credentials have edit access to the document.                                                                                                                                                                                                                                                  | Memory retrieval and saving nodes                                                                |
| The AI Agent system prompt enforces privacy, memory management, and user-centric conversational rules to maintain personalized and sensitive interactions.                                                                                                                                                                                                   | AI Agent node system message                                                                     |
| The workflow currently processes text messages; audio and image message handling can be extended as needed.                                                                                                                                                                                                                                                   | Message Router node fallback outputs                                                            |
| For advanced usage, consider expanding memory management to include session expiration or multi-user support.                                                                                                                                                                                                                                                | Window Buffer Memory node                                                                         |

---

This documentation provides a complete, detailed reference to understand, reproduce, and maintain the "🐋🤖 DeepSeek AI Agent + Telegram + LONG TERM Memory 🧠" workflow, ensuring robust integration between Telegram messaging, DeepSeek AI, and persistent memory storage.