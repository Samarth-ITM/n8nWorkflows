Create a Multi-Model AI Chatbot with Telegram, AIMLAPI, and Google Sheets

https://n8nworkflows.xyz/workflows/create-a-multi-model-ai-chatbot-with-telegram--aimlapi--and-google-sheets-7045


# Create a Multi-Model AI Chatbot with Telegram, AIMLAPI, and Google Sheets

### 1. Workflow Overview

This workflow implements a multi-model AI chatbot on Telegram, integrating AIMLAPI and Google Sheets for AI response generation, usage logging, and daily usage limits. It enables Telegram users to interact dynamically with various AI models by specifying model IDs in their messages or requesting a list of available models. The workflow enforces per-user daily generation limits tracked in Google Sheets, sends AI-generated responses back to Telegram, and logs all interactions for monitoring.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives and routes incoming Telegram messages.
- **1.2 Command Routing:** Distinguishes between model list requests and AI prompt processing.
- **1.3 Model List Retrieval & Formatting:** Fetches and formats available AI models from AIMLAPI.
- **1.4 Usage Logging and Limit Enforcement:** Retrieves user usage data, counts daily requests, sets limits, and checks if limits are exceeded.
- **1.5 AI Prompt Processing:** Parses user messages, selects the AI model, sends prompts to AIMLAPI, and handles responses.
- **1.6 Response Delivery and Logging:** Sends AI responses back to Telegram and logs successful generations in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Handles inbound messages from Telegram users by triggering on new messages.

**Nodes Involved:**  
- 📩 Receive Telegram Message  
- Sticky Note (Incoming Message)

**Node Details:**

- **📩 Receive Telegram Message**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for new Telegram messages (updates of type "message").  
  - Configuration: Uses Telegram API credentials; no additional filters.  
  - Inputs: None (trigger node).  
  - Outputs: Message JSON containing chat and user info.  
  - Potential Failures: Telegram API errors, webhook misconfiguration, network issues.

- **Sticky Note (Incoming Message)**  
  - Role: Documentation; explains this block handles incoming Telegram messages.

---

#### 2.2 Command Routing

**Overview:**  
Routes messages either to model list retrieval if the user requests `/models`, or to AI prompt processing if a message contains text.

**Nodes Involved:**  
- Get Models Or Process Message? (Switch)

**Node Details:**

- **Get Models Or Process Message?**  
  - Type: Switch  
  - Role: Checks if message text contains `/models`; routes accordingly.  
  - Configuration:  
    - Output "Get Models" if message text contains "/models".  
    - Output "Process Message" if message text is non-empty and does not contain "/models".  
  - Inputs: Message JSON from Receive Telegram Message.  
  - Outputs: Two possible branches: Get Models List or Fetch Usage Logs.  
  - Edge Cases: Empty messages, messages without text, or unexpected formats.

---

#### 2.3 Model List Retrieval & Formatting

**Overview:**  
Fetches available AI models from AIMLAPI, groups them by provider, and sends formatted lists back to Telegram.

**Nodes Involved:**  
- Get Models List (HTTP Request)  
- Code (Python)  
- Send a text message (Telegram)  
- Sticky Note (Usage Limit Check - related positioning)  

**Node Details:**

- **Get Models List**  
  - Type: HTTP Request  
  - Role: Calls AIMLAPI endpoint to retrieve available models (URL: https://api.aimlapi.com/v1/models).  
  - Inputs: Triggered when user requests `/models`.  
  - Outputs: Raw JSON with model metadata.  
  - Edge Cases: API downtime, authentication issues, malformed responses.

- **Code (Python)**  
  - Type: Code (Python)  
  - Role: Processes raw model data; filters for "chat-completion" type models, groups by developer/provider, and formats output messages.  
  - Key Logic:  
    - Iterates model data, excludes non-chat models.  
    - Groups models under their developers.  
    - Formats output lines with developer name and model IDs/descriptions.  
  - Inputs: Output from Get Models List.  
  - Outputs: Array of JSON messages with formatted content strings.  
  - Failure Modes: Parsing errors, missing fields in API data.

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends formatted model list messages to the requesting Telegram chat.  
  - Inputs: Formatted content from Code node.  
  - Configuration: Uses Telegram API credentials; message text populated dynamically.  
  - Outputs: Message sent confirmation.  
  - Edge Cases: Telegram API rate limits, message length limits.

---

#### 2.4 Usage Logging and Limit Enforcement

**Overview:**  
Checks how many AI generations a user has performed today, enforces a daily generation limit, and notifies if exceeded.

**Nodes Involved:**  
- 📊 Fetch Usage Logs (Google Sheets)  
- 📈 Count Today’s Requests (Aggregate)  
- 🔢 Set Daily Limit (Set)  
- 🚦 Check Limit Exceeded? (If)  
- 🚫 Notify: Limit Exceeded (Telegram)  
- Sticky Note1, Sticky Note6 (Usage Limit Check)  

**Node Details:**

- **📊 Fetch Usage Logs**  
  - Type: Google Sheets  
  - Role: Retrieves usage log entries filtered by user_id and current date (ISO date string).  
  - Inputs: Message JSON (user id), current date.  
  - Outputs: List of usage records for the user today.  
  - Credentials: OAuth2 Google Sheets account.  
  - Failure Modes: Google API auth errors, spreadsheet structure changes, network errors.

- **📈 Count Today’s Requests**  
  - Type: Aggregate  
  - Role: Counts number of usage log entries retrieved.  
  - Inputs: Output from Fetch Usage Logs.  
  - Outputs: Numerical count of today’s requests.  
  - Edge Cases: Empty data sets, aggregation failures.

- **🔢 Set Daily Limit**  
  - Type: Set  
  - Role: Defines the daily generation limit (default 5).  
  - Parameters: Static number assignment `daily_limit = 5`.  
  - Inputs: Count from previous node.  
  - Outputs: Object containing daily_limit value.

- **🚦 Check Limit Exceeded?**  
  - Type: If  
  - Role: Compares today's request count with the daily limit.  
  - Condition: Checks if `count < daily_limit` (allowing further requests if true).  
  - Outputs: Two branches: allowed or exceeded.  
  - Edge Cases: Numeric comparison failures, missing data.

- **🚫 Notify: Limit Exceeded**  
  - Type: Telegram  
  - Role: Sends a message to the user notifying that the daily limit is exceeded.  
  - Inputs: Chat ID from Telegram message, daily_limit from Set node.  
  - Messages: "Sorry! Your *daily limit of 5 generations* is exceeded!" (Markdown formatted).  
  - Credentials: Telegram API.  
  - Edge Cases: Telegram API errors, message delivery failures.

---

#### 2.5 AI Prompt Processing

**Overview:**  
Parses user messages to extract model ID and prompt text, sends prompt to the specified AI model via AIMLAPI, and obtains AI-generated responses.

**Nodes Involved:**  
- Set Custom Model? (If)  
- Group Models By Providers (Code - JavaScript)  
- 🧠 Generate Msg (AI/ML API | Custom Model)  
- 🧠 Generate Msg (AI/ML API | GPT-4o)  
- Send a chat action (Telegram)  
- Sticky Note9 (Customization Options)  

**Node Details:**

- **Set Custom Model?**  
  - Type: If  
  - Role: Checks if message text starts with `#` indicating a custom model usage.  
  - Outputs:  
    - True: Process as custom model (parse model ID and prompt).  
    - False: Use default GPT-4o model.  
  - Inputs: Telegram message text.  
  - Edge Cases: Messages missing `#`, malformed input.

- **Group Models By Providers (JavaScript Code)**  
  - Type: Code (JS)  
  - Role: Parses message text starting with `#model_id prompt_text`. Extracts the model ID (without `#`) and the prompt text.  
  - Logic:  
    - Splits text by space into two parts.  
    - Throws errors if format invalid (no `#`, no prompt).  
    - Returns JSON with `model_id` and `message`.  
  - Inputs: Telegram message text.  
  - Outputs: Structured JSON for AI API.  
  - Failure Modes: Parsing errors, missing or malformed input.

- **🧠 Generate Msg (AI/ML API | Custom Model)**  
  - Type: AIMLAPI Node  
  - Role: Sends prompt and model ID parsed from user message to AIMLAPI for response generation.  
  - Parameters:  
    - Model: Dynamic from parsed `model_id`.  
    - Prompt: Parsed prompt text.  
  - Credentials: AIMLAPI API key.  
  - Inputs: Output from Group Models By Providers.  
  - Outputs: AI-generated text result.  
  - Edge Cases: Invalid model IDs, API quota limits, timeouts.

- **🧠 Generate Msg (AI/ML API | GPT-4o)**  
  - Type: AIMLAPI Node  
  - Role: Default AI generation with fixed model `openai/gpt-4o` if no custom model specified.  
  - Parameters: Prompt is full user message text.  
  - Credentials: AIMLAPI API key.  
  - Inputs: Telegram message JSON.  
  - Outputs: AI-generated text response.  
  - Edge Cases: API failures, invalid input text.

- **Send a chat action**  
  - Type: Telegram  
  - Role: Sends "typing" or similar chat action to Telegram to indicate processing.  
  - Inputs: Chat ID from Telegram message.  
  - Outputs: Confirmation of chat action sent.  
  - Edge Cases: Telegram API errors.

---

#### 2.6 Response Delivery and Logging

**Overview:**  
Delivers the AI-generated text back to the Telegram chat and logs successful interactions to Google Sheets.

**Nodes Involved:**  
- Send a text message1 (Telegram)  
- 📝 Log Successful Generation1 (Google Sheets)  
- Sticky Note5, Sticky Note11 (Logging and Delivery Notes)  

**Node Details:**

- **Send a text message1**  
  - Type: Telegram  
  - Role: Sends the AI-generated response text to the Telegram chat.  
  - Inputs: AI response JSON with `content` or `result.text`.  
  - Configuration: No attribution appended; uses Telegram API credentials.  
  - Edge Cases: Message length limits, Telegram API errors.

- **📝 Log Successful Generation1**  
  - Type: Google Sheets  
  - Role: Appends a new row to a Google Sheet logging user_id, date, original query, and AI result text.  
  - Inputs:  
    - user_id: Extracted from Telegram message.  
    - date: Current date (ISO string slice).  
    - query: User message text.  
    - result: AI response text.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Google API auth errors, sheet structure changes, network issues.

---

### 3. Summary Table

| Node Name                           | Node Type                   | Functional Role                      | Input Node(s)                   | Output Node(s)                     | Sticky Note                                              |
|-----------------------------------|-----------------------------|------------------------------------|--------------------------------|----------------------------------|----------------------------------------------------------|
| 📩 Receive Telegram Message        | Telegram Trigger            | Entry point, receives Telegram messages | None                          | Get Models Or Process Message?     | Incoming Message: Handle incoming user messages from Telegram. |
| Get Models Or Process Message?     | Switch                     | Routes message to model list or AI processing | 📩 Receive Telegram Message    | Get Models List, 📊 Fetch Usage Logs |                                                          |
| Get Models List                   | HTTP Request               | Fetch AI models list from AIMLAPI  | Get Models Or Process Message? (Get Models branch) | Code                             |                                                          |
| Code                             | Python Code                | Group models by developer & format | Get Models List                | Send a text message               |                                                          |
| Send a text message              | Telegram                   | Send formatted model list to user  | Code                          | None                            |                                                          |
| 📊 Fetch Usage Logs               | Google Sheets              | Retrieve user's usage logs for today | Get Models Or Process Message? (Process Message branch) | 📈 Count Today’s Requests          | Usage Limit Check: Check how many times the user has generated today and enforce the daily limit. |
| 📈 Count Today’s Requests          | Aggregate                  | Count number of user's requests today | 📊 Fetch Usage Logs            | 🔢 Set Daily Limit                | Usage Limit Check: Check how many times the user has generated today and enforce the daily limit. |
| 🔢 Set Daily Limit                | Set                        | Define daily generation limit (5)  | 📈 Count Today’s Requests       | 🚦 Check Limit Exceeded?           | Usage Limit Check: Check how many times the user has generated today and enforce the daily limit. |
| 🚦 Check Limit Exceeded?          | If                         | Check if user exceeded daily limit | 🔢 Set Daily Limit             | Send a chat action, 🚫 Notify: Limit Exceeded | Usage Limit Check: Check how many times the user has generated today and enforce the daily limit. |
| Send a chat action               | Telegram                   | Send "typing" or chat action to Telegram | 🚦 Check Limit Exceeded? (allowed branch) | Set Custom Model?                |                                                          |
| 🚫 Notify: Limit Exceeded         | Telegram                   | Notify user daily limit exceeded    | 🚦 Check Limit Exceeded? (exceeded branch) | None                            |                                                          |
| Set Custom Model?                | If                         | Check if message uses custom model | Send a chat action             | Group Models By Providers, 🧠 Generate Msg (GPT-4o) |                                                          |
| Group Models By Providers         | JavaScript Code            | Parse model ID and prompt from message | Set Custom Model?              | 🧠 Generate Msg (Custom Model)    |                                                          |
| 🧠 Generate Msg (AI/ML API | Custom Model) | AIMLAPI                    | Generate AI response with custom model | Group Models By Providers       | Send a text message1                |                                                          |
| 🧠 Generate Msg (AI/ML API | GPT-4o) | AIMLAPI                    | Generate AI response with default GPT-4o model | Set Custom Model? (false branch) | Send a text message1                |                                                          |
| Send a text message1             | Telegram                   | Send AI-generated response text    | 🧠 Generate Msg (Custom Model), 🧠 Generate Msg (GPT-4o) | 📝 Log Successful Generation1    | Delivery & Logging: Send the final image and caption to the user, and log the result to Google Sheets. |
| 📝 Log Successful Generation1     | Google Sheets              | Log successful generation details   | Send a text message1           | None                            | Data Logged to Google Sheets: Logs user_id, date, query, result. |
| Sticky Note                      | Sticky Note                | Documentation                      | None                          | None                            | Incoming Message: Handle incoming user messages from Telegram. |
| Sticky Note1                     | Sticky Note                | Documentation                      | None                          | None                            | Usage Limit Check: Check how many times the user has generated today and enforce the daily limit. |
| Sticky Note5                     | Sticky Note                | Documentation                      | None                          | None                            | Delivery & Logging: Send the final image and caption to the user, and log the result to Google Sheets. |
| Sticky Note6                     | Sticky Note                | Documentation                      | None                          | None                            | Usage Limit Check: Check how many times the user has generated today and enforce the daily limit. |
| Sticky Note8                     | Sticky Note                | Documentation                      | None                          | None                            | AI Chat & Model Selector Bot overview and user instructions. |
| Sticky Note9                     | Sticky Note                | Documentation                      | None                          | None                            | Customization options and example user flow. |
| Sticky Note10                    | Sticky Note                | Documentation                      | None                          | None                            | Setup guide with Telegram, Google Sheets, and AIMLAPI instructions. |
| Sticky Note11                    | Sticky Note                | Documentation                      | None                          | None                            | Data logging and testing/debugging tips. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node: "📩 Receive Telegram Message"**  
   - Type: Telegram Trigger  
   - Set updates to listen for "message".  
   - Connect Telegram API credentials created with your bot token.  
   - No extra filters required.

2. **Add Switch Node: "Get Models Or Process Message?"**  
   - Type: Switch  
   - Rule 1 (Output "Get Models"): If message.text contains "/models".  
   - Rule 2 (Output "Process Message"): If message.text is not empty.  
   - Connect input from "📩 Receive Telegram Message".

3. **Create HTTP Request Node: "Get Models List"**  
   - Type: HTTP Request  
   - URL: https://api.aimlapi.com/v1/models  
   - No authentication if public, else configure AIMLAPI API key if needed.  
   - Connect input from "Get Models Or Process Message?" (Get Models branch).

4. **Add Python Code Node: "Code"**  
   - Type: Code (Python)  
   - Paste the provided Python code to group models by developer and format messages.  
   - Input from "Get Models List".  
   - Output is an array of JSON messages with content strings.

5. **Add Telegram Node: "Send a text message"**  
   - Type: Telegram  
   - Text: `={{ $json.content }}`  
   - Chat ID: `={{ $('📩 Receive Telegram Message').item.json.message.chat.id }}`  
   - Connect input from "Code" node.  
   - Use Telegram credentials.

6. **Add Google Sheets Node: "📊 Fetch Usage Logs"**  
   - Type: Google Sheets  
   - Operation: Lookup rows  
   - Filters: user_id = `={{ $json.message.from.id }}`, date = `={{ new Date().toISOString().slice(0,10) }}`  
   - Sheet Name and Document ID: point to your usage log sheet with headers (user_id, date, query, result).  
   - Credentials: Google Sheets OAuth2.  
   - Connect input from "Get Models Or Process Message?" (Process Message branch).

7. **Add Aggregate Node: "📈 Count Today’s Requests"**  
   - Type: Aggregate  
   - Operation: Count all items  
   - Input from "📊 Fetch Usage Logs".

8. **Add Set Node: "🔢 Set Daily Limit"**  
   - Type: Set  
   - Assign variable `daily_limit` to 5 (default daily limit).  
   - Input from "📈 Count Today’s Requests".

9. **Add If Node: "🚦 Check Limit Exceeded?"**  
   - Type: If  
   - Condition: Number of requests (`={{$('📈 Count Today’s Requests').item.json.data.length}}`) < daily_limit (`={{ $json.daily_limit }}`).  
   - Input from "🔢 Set Daily Limit".  
   - True branch: proceed to send chat action and AI processing.  
   - False branch: notify user limit exceeded.

10. **Add Telegram Node: "🚫 Notify: Limit Exceeded"**  
    - Type: Telegram  
    - Text: `"=Sorry! Your *daily limit of {{ $('🔢 Set Daily Limit').item.json.daily_limit }} generations* is exceeded!"`  
    - Chat ID: `={{ $('📩 Receive Telegram Message').item.json.message.chat.id }}`  
    - Input from false branch of "🚦 Check Limit Exceeded?" node.  
    - Use Telegram credentials.

11. **Add Telegram Node: "Send a chat action"**  
    - Type: Telegram  
    - Operation: sendChatAction  
    - Chat ID from Telegram message.  
    - Input from true branch of "🚦 Check Limit Exceeded?".

12. **Add If Node: "Set Custom Model?"**  
    - Type: If  
    - Condition: message.text starts with `#`.  
    - Input from "Send a chat action".  
    - True branch: custom model processing; false branch: default GPT-4o.

13. **Add JavaScript Code Node: "Group Models By Providers"**  
    - Type: Code (JavaScript)  
    - Parse message text to extract model ID and prompt.  
    - Throw errors if format invalid.  
    - Input from true branch of "Set Custom Model?".

14. **Add AIMLAPI Node: "🧠 Generate Msg (AI/ML API | Custom Model)"**  
    - Type: AIMLAPI  
    - Model: `={{ $json.model_id }}` from parsed data  
    - Prompt: `={{ $json.message }}`  
    - Credentials: AIMLAPI API key  
    - Input from "Group Models By Providers".

15. **Add AIMLAPI Node: "🧠 Generate Msg (AI/ML API | GPT-4o)"**  
    - Type: AIMLAPI  
    - Model: `openai/gpt-4o` (fixed)  
    - Prompt: full message text from Telegram  
    - Credentials: AIMLAPI API key  
    - Input from false branch of "Set Custom Model?".

16. **Add Telegram Node: "Send a text message1"**  
    - Type: Telegram  
    - Text: AI generated response text (`={{ $json.content || $json.result.text }}`)  
    - Chat ID from Telegram message  
    - Inputs from both AIMLAPI nodes.  
    - Use Telegram credentials.

17. **Add Google Sheets Node: "📝 Log Successful Generation1"**  
    - Type: Google Sheets  
    - Operation: Append row  
    - Columns: user_id, date, query, result  
    - Values mapped from Telegram message and AI response.  
    - Credentials: Google Sheets OAuth2.  
    - Input from "Send a text message1".

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Multi-model AI chatbot with model selection by hashtag, daily limits, usage logging, and Telegram integration.                         | Workflow overview and functional description.                                                                |
| Setup guide includes Telegram BotFather creation, credential configuration, Google Sheets setup, and AIMLAPI API key configuration.   | Sticky Note10; Telegram and Google Sheets setup instructions.                                                |
| Daily usage limits can be customized; additional features like NSFW filtering, aliases, and extra commands can be added.               | Sticky Note9; customization and extension ideas.                                                             |
| Testing tips include using separate Telegram chats, inspecting payloads, and edge case validation (missing model tags, invalid IDs). | Sticky Note11; debugging and testing best practices.                                                         |
| Official AIMLAPI and Telegram API documentation recommended for credential setup and API limits understanding.                        | External resource recommendation for integration details.                                                    |
| Usage data logged includes user_id, date, query, and AI-generated result to help monitor usage and behavior.                          | Sticky Note11; data logging schema and purpose.                                                               |
| Model list fetched dynamically from AIMLAPI allows up-to-date model selection by users.                                                | Ensures no hardcoding of model IDs, allowing extensibility.                                                  |
| Telegram's "sendChatAction" used to improve UX by showing typing indication during AI processing.                                      | Enhances responsiveness perception in chat interactions.                                                     |

---

**Disclaimer:** The provided text originates solely from an automated n8n workflow. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.