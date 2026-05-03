Chat with GPT-5 on Telegram: AI-Powered Conversational Bot

https://n8nworkflows.xyz/workflows/chat-with-gpt-5-on-telegram--ai-powered-conversational-bot-7331


# Chat with GPT-5 on Telegram: AI-Powered Conversational Bot

### 1. Workflow Overview

This n8n workflow enables a real-time AI-powered chat bot on Telegram using GPT-5 via the AIMLAPI service. It listens to Telegram messages sent to a bot, processes the text with GPT-5 to generate conversational responses, and sends replies back to the user within Telegram. Optionally, it logs each interaction to a Google Sheets document for tracking and analytics.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception:** Receiving and capturing incoming Telegram messages from users.
- **1.2 User Experience Enhancement:** Sending a "typing…" chat action to create a natural conversational feel.
- **1.3 AI Processing:** Forwarding the received text to the GPT-5 model via AIMLAPI and obtaining the generated response.
- **1.4 Output Delivery:** Sending the AI-generated reply back to the same Telegram chat.
- **1.5 Interaction Logging (Optional):** Recording chat details (user ID, date, query, and response) in Google Sheets for monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new messages sent to the Telegram bot, extracting key information such as chat IDs and message text for downstream processing.

- **Nodes Involved:**  
  - 📩 Receive Telegram Message

- **Node Details:**  

  **📩 Receive Telegram Message**  
  - Type: Telegram Trigger node  
  - Configuration: Listens for the "message" update type; no command filtering applied, accepts all messages.  
  - Expressions: Outputs full Telegram message JSON, capturing `message.text` and `message.chat.id`.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to the "Simulate Typing in Telegram" node.  
  - Edge Cases:  
    - Telegram API connectivity or auth errors.  
    - Messages without text (e.g., stickers, images) might require filtering if unsupported.  
  - Credentials: Telegram API credentials linked for bot authentication.

#### 2.2 User Experience Enhancement

- **Overview:**  
  This block sends a "typing…" chat action in Telegram, notifying the user that the bot is processing their input, enhancing UX especially for longer response times.

- **Nodes Involved:**  
  - 💬 Simulate Typing in Telegram  

- **Node Details:**  

  **💬 Simulate Typing in Telegram**  
  - Type: Telegram node (send chat action)  
  - Configuration: Sends `sendChatAction` with action type "typing" to the chat ID from the incoming message.  
  - Expressions: `chatId` set dynamically via `{{$json.message.chat.id}}` from the trigger node.  
  - Inputs: Receives data from "📩 Receive Telegram Message"  
  - Outputs: Connects to the "🧠 Process with GPT‑5" node.  
  - Edge Cases:  
    - Sending chat action may fail if invalid chat ID or if Telegram API is down.  
    - May require throttling if high message volume occurs.  
  - Credentials: Uses same Telegram API credentials as the trigger.

#### 2.3 AI Processing

- **Overview:**  
  Sends the user's message text to the GPT-5 model through AIMLAPI to generate a contextual AI response.

- **Nodes Involved:**  
  - 🧠 Process with GPT‑5

- **Node Details:**  

  **🧠 Process with GPT‑5**  
  - Type: AIMLAPI node (AI/ML API integration)  
  - Configuration:  
    - Model set to `"openai/gpt-5-chat-latest"`  
    - Prompt dynamically bound to the original Telegram message text (`{{$('📩 Receive Telegram Message').item.json.message.text}}`)  
    - No additional options or system instructions configured (but can be customized).  
  - Inputs: Receives from "💬 Simulate Typing in Telegram"  
  - Outputs: Connects to "📤 Send Response to Telegram"  
  - Edge Cases:  
    - API key or network failures may cause timeouts or errors.  
    - Model rate limits or quota exhaustion.  
    - Invalid or empty prompt text could cause API errors.  
  - Credentials: Uses AIMLAPI API key credentials.

#### 2.4 Output Delivery

- **Overview:**  
  Sends the generated AI response text back to the same Telegram chat where the message originated, optionally replying to the original message.

- **Nodes Involved:**  
  - 📤 Send Response to Telegram

- **Node Details:**  

  **📤 Send Response to Telegram**  
  - Type: Telegram node (send message)  
  - Configuration:  
    - Sends message text from GPT-5 output (`{{$json.content}}`).  
    - Chat ID dynamically set from original message chat ID (`{{$('📩 Receive Telegram Message').item.json.message.chat.id}}`).  
    - Replies to the original message ID for threading (`reply_to_message_id` set).  
    - Attribution disabled for cleaner message appearance.  
  - Inputs: Receives from "🧠 Process with GPT‑5"  
  - Outputs: Connects to "📝 Log Successful Generation" node.  
  - Edge Cases:  
    - Telegram API errors (chat ID invalid, message too long).  
    - Network or credential issues.  
  - Credentials: Telegram API credentials.

#### 2.5 Interaction Logging (Optional)

- **Overview:**  
  Logs each chat interaction to a Google Sheets document, capturing user ID, date, original query, and AI response for analytics and monitoring.

- **Nodes Involved:**  
  - 📝 Log Successful Generation

- **Node Details:**  

  **📝 Log Successful Generation**  
  - Type: Google Sheets node (append row)  
  - Configuration:  
    - Sheet: Specific Google Sheet identified by document ID and sheet name "Sheet1" (with link provided).  
    - Columns mapped:  
      - `date` — current date in ISO format (YYYY-MM-DD)  
      - `query` — user’s original message text from Telegram  
      - `result` — GPT-5 generated response text  
      - `user_id` — Telegram user ID extracted from the message sender.  
  - Inputs: Receives from "📤 Send Response to Telegram"  
  - Outputs: None (terminal node)  
  - Edge Cases:  
    - Google Sheets API auth or permission issues.  
    - Rate limits on Google Sheets API.  
    - Data format mismatches (e.g., empty fields).  
  - Credentials: Google OAuth2 credentials for Sheets API.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                 |
|------------------------------|----------------------------------|---------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------|
| 📩 Receive Telegram Message   | Telegram Trigger                 | Input Reception (Telegram)       | None                         | 💬 Simulate Typing in Telegram | # 📥 Step 1 — Receive Telegram Message: Listens for new messages, captures chat.id and message.text          |
| 💬 Simulate Typing in Telegram| Telegram                        | UX Enhancement ("typing…" action)| 📩 Receive Telegram Message  | 🧠 Process with GPT‑5          | # 💬 Step 1.5 — Simulate Typing in Telegram: Sends "typing…" chat action to simulate natural conversation    |
| 🧠 Process with GPT‑5          | AIMLAPI (AI/ML API)             | AI Processing of Message         | 💬 Simulate Typing in Telegram| 📤 Send Response to Telegram   | # 🧠 Step 2 — Process with GPT‑5: Sends message text to GPT-5 model via AIMLAPI for response generation      |
| 📤 Send Response to Telegram  | Telegram                        | Output Delivery to Telegram      | 🧠 Process with GPT‑5          | 📝 Log Successful Generation   | # 📤 Step 3 — Send Response to Telegram: Sends AI response back to user chat with reply threading             |
| 📝 Log Successful Generation  | Google Sheets                   | Optional Interaction Logging     | 📤 Send Response to Telegram  | None                          | # 📝 Step 4 — Log Interaction (Optional): Append chat data to Google Sheets for analytics and monitoring      |
| Sticky Note — Overview        | Sticky Note                    | Documentation / Explanation      | None                         | None                          | # 💬 AI Chat Bot — Telegram + AIMLAPI (via n8n): Features and capabilities overview                           |
| Sticky Note — Customization   | Sticky Note                    | Documentation / Explanation      | None                         | None                          | ## ⚙️ Customization: Model change, logging, NSFW filtering, command handling, example user flow              |
| Sticky Note — Setup Guide     | Sticky Note                    | Documentation / Explanation      | None                         | None                          | # 🛠 Setup Guide: Telegram bot creation, credentials setup for Telegram and AIMLAPI                          |
| Sticky Note — Logging & Testing| Sticky Note                   | Documentation / Explanation      | None                         | None                          | ## 📁 Optional Data Logging & 🧪 Testing Tips: Logging schema and test recommendations                         |
| Sticky Note — Receive Message | Sticky Note                    | Documentation / Explanation      | None                         | None                          | # 📥 Step 1 — Receive Telegram Message: Detailed explanation of message capture                              |
| Sticky Note — GPT‑5 Processing| Sticky Note                    | Documentation / Explanation      | None                         | None                          | # 🧠 Step 2 — Process with GPT‑5: Details on sending messages to AI model                                   |
| Sticky Note — Send Response   | Sticky Note                    | Documentation / Explanation      | None                         | None                          | # 📤 Step 3 — Send Response to Telegram: Details on response delivery                                        |
| Sticky Note — Logging         | Sticky Note                    | Documentation / Explanation      | None                         | None                          | # 📝 Step 4 — Log Interaction (Optional): Explanation of logging purpose and data fields                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot & Credentials**  
   - Use Telegram’s [@BotFather](https://t.me/BotFather) to create a new bot. Save the API token.  
   - In n8n, go to Credentials → Create new Telegram API credentials using the saved token.

2. **Create AIMLAPI Credentials**  
   - Obtain your AIMLAPI API key from [docs.aimlapi.com](https://docs.aimlapi.com).  
   - In n8n, create new credentials for AI/ML API with your API key and base URL `https://api.aimlapi.com/v1`.

3. **Create Google Sheets OAuth2 Credentials (Optional Logging)**  
   - Create and configure OAuth2 credentials in n8n for Google Sheets API with proper access scopes.

4. **Add "📩 Receive Telegram Message" Node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: Select "message"  
     - No filters (accept all messages)  
   - Credentials: Select Telegram API credentials  
   - This node triggers the workflow on incoming Telegram messages.

5. **Add "💬 Simulate Typing in Telegram" Node**  
   - Type: Telegram  
   - Operation: `sendChatAction` with action type "typing"  
   - Parameters:  
     - chatId: `{{$json.message.chat.id}}` (from trigger node)  
   - Credentials: Telegram API credentials  
   - Connect output of "📩 Receive Telegram Message" to this node.

6. **Add "🧠 Process with GPT‑5" Node**  
   - Type: AIMLAPI  
   - Parameters:  
     - Model: `"openai/gpt-5-chat-latest"`  
     - Prompt: `={{ $('📩 Receive Telegram Message').item.json.message.text }}`  
     - Leave other options default or add system instructions as needed  
   - Credentials: AIMLAPI API key  
   - Connect output of "💬 Simulate Typing in Telegram" to this node.

7. **Add "📤 Send Response to Telegram" Node**  
   - Type: Telegram  
   - Operation: Send Message  
   - Parameters:  
     - chatId: `={{ $('📩 Receive Telegram Message').item.json.message.chat.id }}`  
     - Text: `={{ $json.content }}` (output from GPT-5 node)  
     - Reply To Message ID: `={{ $('📩 Receive Telegram Message').item.json.message.message_id }}`  
     - Disable attribution for cleaner reply  
   - Credentials: Telegram API credentials  
   - Connect output of "🧠 Process with GPT‑5" to this node.

8. **Add "📝 Log Successful Generation" Node (Optional)**  
   - Type: Google Sheets  
   - Operation: Append Row  
   - Parameters:  
     - Document ID: Your Google Sheet ID  
     - Sheet Name: e.g., "Sheet1"  
     - Columns mapping:  
       - `date`: `={{ new Date().toISOString().split('T')[0] }}`  
       - `query`: `={{ $('📩 Receive Telegram Message').item.json.message.text }}`  
       - `result`: `={{ $('🧠 Process with GPT‑5').item.json.content }}`  
       - `user_id`: `={{ $json.result.from.id }}`  
   - Credentials: Google Sheets OAuth2  
   - Connect output of "📤 Send Response to Telegram" to this node.

9. **Test the Workflow**  
   - Execute the workflow by sending a message to your Telegram bot.  
   - Verify the bot replies correctly and logs data if enabled.

10. **(Optional) Add Enhancements**  
    - Add filters for commands like `/help` or `/reset`.  
    - Add NSFW or profanity filtering nodes.  
    - Extend logging to databases or add analytics dashboards.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                        |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| This workflow enables GPT-5 conversational AI in Telegram using AIMLAPI integration.             | Overview description in Sticky Note — Overview                         |
| Setup requires creating a Telegram bot and configuring API credentials in n8n for Telegram and AIMLAPI. | Setup Guide sticky note; Telegram BotFather: https://t.me/BotFather    |
| AIMLAPI docs and API keys available at https://docs.aimlapi.com                                 | Setup Guide sticky note                                                |
| Optional logging to Google Sheets enables tracking user interactions and analytics.              | Logging sticky note                                                    |
| For smoother user experience, the bot simulates typing in Telegram before sending responses.    | Sticky Note — Receive Message1 and UX block                            |
| Example user flow: user asks a question, GPT-5 responds contextually, maintaining chat flow.     | Sticky Note — Customization                                            |
| Testing tip: run workflow via actual Telegram messages, not just "Execute Node" in n8n UI.       | Logging & Testing sticky note                                          |

---

**Disclaimer:**  
The provided content is exclusively generated from an n8n automation workflow and complies with all applicable content policies. No illegal, offensive, or protected material is included. All data handled is lawful and public.