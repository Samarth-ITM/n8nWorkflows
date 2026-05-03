Smart Facebook Messenger Chatbot with Gemini AI, Message Batching and History

https://n8nworkflows.xyz/workflows/smart-facebook-messenger-chatbot-with-gemini-ai--message-batching-and-history-9192


# Smart Facebook Messenger Chatbot with Gemini AI, Message Batching and History

### 1. Workflow Overview

This workflow implements a **Smart Facebook Messenger Chatbot** integrated with Google Gemini AI for natural language processing. It is designed for efficient handling of user messages with **message batching**, **history aggregation**, and **context-aware AI responses**. The chatbot supports multi-turn conversations, ensures human-like interaction via typing and seen indicators, and maintains detailed message logs using n8n’s Data Table feature.

The workflow is structured into four main logical blocks:

- **1.1 Facebook Webhook & Initial Message Preprocessing**  
  Listens to Facebook Messenger webhook events, validates incoming messages, filters out echoes, and extracts key contextual information about the conversation.

- **1.2 Message Batching & History Aggregation**  
  Collects multiple incoming messages from the same user in short intervals, merges them into a single consolidated message, and constructs a chat history for context-aware AI processing.

- **1.3 AI Processing & Enhanced Interaction**  
  Sends the merged user message and conversation history to the AI Agent using Google Gemini Chat Model, which generates concise, persona-driven replies formatted for Facebook Messenger.

- **1.4 Deliver Response & Update Batching Status**  
  Sends the AI-generated reply back to the user on Messenger, updates the Data Table with the bot’s response, and marks processed user messages to prevent duplicate handling.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Facebook Webhook & Initial Message Preprocessing

**Overview:**  
This block handles the Facebook webhook connection, message verification, and initial filtering. It ensures only relevant user messages (non-echo, text messages) proceed downstream, and extracts necessary context for processing.

**Nodes Involved:**  
- Facebook Webhook  
- Confirm Webhook  
- Is message?  
- Is from page?  
- Set Context  
- Seen

**Node Details:**

- **Facebook Webhook**  
  - *Type:* Webhook Listener  
  - *Role:* Receives all incoming events from the Facebook Page Messenger webhook.  
  - *Config:* Path is set to `/nguyenthieutoan-facebook-page`. Supports multiple HTTP methods; response mode is set to respond via a node.  
  - *Input/Output:* Input is external HTTP requests; outputs raw webhook payload.  
  - *Edge Cases:* Missing or malformed webhook requests; Facebook verification requests handled separately.  
  - *Sticky Note:* Explains webhook setup and message preprocessing.

- **Confirm Webhook**  
  - *Type:* Respond to Webhook (HTTP response node)  
  - *Role:* Responds to Facebook’s webhook verification challenge by echoing the `hub.challenge` query parameter.  
  - *Config:* Responds with plain text containing `hub.challenge` value.  
  - *Connections:* Triggered directly by Facebook Webhook node's GET requests.  
  - *Edge Cases:* Missing or invalid verification token; incorrect response breaks Facebook webhook setup.

- **Is message?**  
  - *Type:* If Node (Condition Filter)  
  - *Role:* Checks if the incoming webhook payload contains a user message with text content (exists and non-empty).  
  - *Config:* Condition tests existence of `$json.body.entry[0].messaging[0].message.text`.  
  - *Output:* Passes only relevant text message events downstream.  
  - *Edge Cases:* Events without messages or non-text messages are filtered out.

- **Is from page?**  
  - *Type:* If Node  
  - *Role:* Filters out messages sent from the page itself (echoes) to avoid processing bot replies.  
  - *Config:* Checks if `$json.body.entry[0].messaging[0].message.is_echo` is true.  
  - *Output:* Only messages not from page (user messages) continue.  
  - *Edge Cases:* Misidentification could cause message loops.

- **Set Context**  
  - *Type:* Set Node  
  - *Role:* Extracts and structures key context data (user ID, user text, page ID, page token placeholder) into a JSON object for downstream use.  
  - *Config:*  
    - `user_id` from `$json.body.entry[0].messaging[0].sender.id`  
    - `user_text` from `$json.body.entry[0].messaging[0].message.text`  
    - `page_id` from `$json.body.entry[0].id`  
    - `page_token` is a placeholder string `<YOUR_TOKEN>` to be replaced by the actual page access token.  
  - *Output:* Structured context JSON.  
  - *Edge Cases:* Missing fields in webhook data cause failures.

- **Seen**  
  - *Type:* HTTP Request  
  - *Role:* Sends a "mark_seen" sender action to Facebook Messenger indicating the message is seen.  
  - *Config:* POST request to Facebook Graph API `https://graph.facebook.com/v23.0/{{ $json.page_id }}/messages` with JSON body specifying recipient and sender_action `"mark_seen"`. Uses `page_token` for authentication.  
  - *Error Handling:* Continues on error to avoid blocking workflow.  
  - *Edge Cases:* Token expiration or API errors.

---

#### 2.2 Message Batching & History Aggregation

**Overview:**  
This block batches multiple incoming messages from the same user, merges their content to create a single context-rich message, and constructs chat history for AI context. It uses n8n Data Table `Batch_messages` to store and manage message states.

**Nodes Involved:**  
- Insert To Process  
- Wait 3s  
- Get unprocessed message  
- Get Max ID + Merged Mess  
- Is Max ID?  
- Get history message  
- Merge History

**Node Details:**

- **Insert To Process**  
  - *Type:* Data Table (Insert operation)  
  - *Role:* Stores each incoming user message into the `Batch_messages` Data Table with fields: `user_id`, `user_text`, `processed` (false), `bot_rep` (empty).  
  - *Config:* Uses context JSON for `user_id` and `user_text`.  
  - *Edge Cases:* Data Table unavailability or write errors.

- **Wait 3s**  
  - *Type:* Wait Node  
  - *Role:* Pauses workflow for 3 seconds to allow batching of rapid subsequent messages.  
  - *Config:* Default 3-second wait.  
  - *Edge Cases:* Delays increase response time but improve batching efficiency.

- **Get unprocessed message**  
  - *Type:* Data Table (Get operation)  
  - *Role:* Retrieves up to 10 unprocessed messages (`processed = false`) from the same user from `Batch_messages`.  
  - *Config:* Filter by `user_id` and `processed=false`.  
  - *Edge Cases:* No unprocessed messages means empty batch.

- **Get Max ID + Merged Mess**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Sorts retrieved messages by `id`, merges all user texts into a single string `merged_message`, and outputs the item with the highest `id` enhanced with this merged message.  
  - *Script Logic:*  
    - Sort items ascending by `id`  
    - Concatenate `user_text` fields with spaces  
    - Add `merged_message` to max ID item  
  - *Edge Cases:* Empty input returns empty output.

- **Is Max ID?**  
  - *Type:* If Node  
  - *Role:* Checks if the current message’s ID equals the ID of the max batch item, to ensure only the latest merged message triggers AI processing.  
  - *Config:* Compares `$json.id` with `$('Insert To Process').first().json.id`.  
  - *Output:* True branch proceeds to AI; false branch stops.

- **Get history message**  
  - *Type:* Data Table (Get operation)  
  - *Role:* Retrieves up to 10 processed messages (`processed = true`) for the user from `Batch_messages` to build chat history for context.  
  - *Config:* Filters on `user_id` and `processed=true`.  
  - *Edge Cases:* Empty history handled gracefully.

- **Merge History**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Sorts messages ascending by `id` and creates a chat history array alternating user and bot messages with roles `user` and `bot`. Flattens the array and outputs as `history`.  
  - *Script Logic:*  
    - Map each item to an array of `{role: 'user', text: user_text}` and `{role: 'bot', text: bot_rep}`  
    - Flatten array  
    - Return as `history` JSON key  
  - *Edge Cases:* Missing bot replies or user texts handled by conditional check.

---

#### 2.3 AI Processing & Enhanced Interaction

**Overview:**  
This block sends the merged user message and chat history to Google Gemini AI via an AI Agent node, which applies a persona and system prompt to generate a concise, Vietnamese-language, Facebook Markdown formatted reply, simulating a personal AI assistant.

**Nodes Involved:**  
- Send Typing  
- Process Merged Message  
- Google Gemini Chat Model  
- Groq Chat Model (available but not connected in this workflow)

**Node Details:**

- **Send Typing**  
  - *Type:* HTTP Request  
  - *Role:* Sends a `sender_action: typing_on` event to Facebook Messenger API to simulate typing indicator for better UX.  
  - *Config:* POST request to Facebook Graph API `https://graph.facebook.com/v23.0/{{ page_id }}/messages` with recipient ID and `typing_on`. Uses `page_token` for auth.  
  - *Error Handling:* Continues on error to not block workflow.  
  - *Edge Cases:* Token issues or API limitations.

- **Process Merged Message**  
  - *Type:* AI Agent (Langchain Agent Node)  
  - *Role:* Uses Google Gemini Chat Model (or optionally Groq model) to process the merged message with a rich system prompt defining the persona "Jenix" — a personal AI assistant for Nguyễn Thiệu Toàn.  
  - *Config:*  
    - Input text is merged user message from previous block  
    - System message defines the persona, conversation flow rules, tone, output format (Facebook Markdown), and language (Vietnamese)  
    - Uses conversation history to decide whether to introduce itself or reply directly  
    - Retry enabled on failure with 100 ms wait  
  - *Edge Cases:* API call failures, prompt parsing errors, or missing context.

- **Google Gemini Chat Model**  
  - *Type:* Langchain LLM Chat Node  
  - *Role:* Connected as AI language model to the AI Agent node, provides actual LLM responses using Google Gemini.  
  - *Config:* No special options set beyond defaults.  
  - *Edge Cases:* API rate limits, credentials required.

- **Groq Chat Model**  
  - *Type:* Langchain LLM Chat Node  
  - *Role:* Alternative AI language model (Groq/Compound) available but not actively connected in this workflow.  
  - *Edge Cases:* Not used in current logic.

---

#### 2.4 Deliver Response & Update Batching Status

**Overview:**  
This final block formats the AI response for Facebook Messenger, sends the reply message, updates the Data Table with the bot’s reply, and marks all batched user messages as processed to avoid duplicate replies.

**Nodes Involved:**  
- Format for Facebook Output  
- Send Text  
- Update Page Rep  
- Update FALSE to TRUE

**Node Details:**

- **Format for Facebook Output**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Converts AI output text into Facebook Messenger compatible markdown format and splits long replies into chunks of max 2000 characters to comply with Facebook limits.  
  - *Logic:*  
    - Cleans escape characters and converts Markdown and HTML to Facebook Markdown syntax  
    - Removes unsupported formatting (underline, strike)  
    - Converts lists and headers to Facebook bullet list and bold styles  
    - Splits large texts on line breaks smartly to avoid breaking sentences abruptly  
  - *Output:* Array of JSON items each with a `text` field for sending.  
  - *Edge Cases:* Empty or malformed AI output; long messages.

- **Send Text**  
  - *Type:* HTTP Request  
  - *Role:* Sends the formatted text replies to Facebook Messenger via the Graph API. Only sends for the latest merged message to prevent duplicates.  
  - *Config:* POST request to `https://graph.facebook.com/v23.0/{{ page_id }}/messages` with recipient ID, message type "RESPONSE", and message text. Authenticated with `page_token`.  
  - *Edge Cases:* API errors, token expiry.

- **Update Page Rep**  
  - *Type:* Data Table (Update operation)  
  - *Role:* Stores the bot’s reply text in the `bot_rep` field of the corresponding message record in `Batch_messages` for traceability and historical context.  
  - *Config:* Matches on `user_id` and message `id` (from max batch message) and updates `bot_rep`.  
  - *Edge Cases:* Data update failures.

- **Update FALSE to TRUE**  
  - *Type:* Data Table (Update operation)  
  - *Role:* Marks all unprocessed batched messages for the user as processed (`processed = true`) to close the batch and prevent reprocessing.  
  - *Config:* Filters on `user_id` and `processed=false`.  
  - *Edge Cases:* Partial updates or data consistency issues.

---

### 3. Summary Table

| Node Name              | Node Type                           | Functional Role                                   | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                     |
|------------------------|-----------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------|
| Facebook Webhook       | Webhook                           | Receives Facebook webhook events                 | External HTTP                   | Confirm Webhook, Is message?      | Explains webhook setup and message preprocessing                                                               |
| Confirm Webhook        | RespondToWebhook                  | Responds to Facebook webhook verification         | Facebook Webhook                | None                            | Explains webhook setup and message preprocessing                                                               |
| Is message?            | If                               | Checks if incoming event contains user text      | Facebook Webhook                | Is from page?                    | Explains webhook setup and message preprocessing                                                               |
| Is from page?          | If                               | Filters out page's own echo messages              | Is message?                    | Set Context (if false)           | Explains webhook setup and message preprocessing                                                               |
| Set Context            | Set                              | Extracts user and page context                     | Is from page?                   | Insert To Process, Seen           | Explains webhook setup and message preprocessing                                                               |
| Seen                   | HTTP Request                     | Sends "mark_seen" action to Facebook              | Set Context                    | None                            | Explains webhook setup and message preprocessing                                                               |
| Insert To Process      | Data Table Insert                | Adds incoming user messages to batch table        | Set Context                    | Wait 3s                         | Describes message batching and history aggregation                                                             |
| Wait 3s                | Wait                             | Delays to allow message batching                   | Insert To Process              | Get unprocessed message           | Describes message batching and history aggregation                                                             |
| Get unprocessed message| Data Table Get                  | Retrieves unprocessed messages for user            | Wait 3s                       | Get Max ID + Merged Mess          | Describes message batching and history aggregation                                                             |
| Get Max ID + Merged Mess| Code                            | Merges batch messages and finds max ID message    | Get unprocessed message        | Is Max ID?                      | Describes message batching and history aggregation                                                             |
| Is Max ID?             | If                               | Checks if current message is the latest in batch  | Get Max ID + Merged Mess       | Send Typing, Get history message  | Describes message batching and history aggregation                                                             |
| Get history message    | Data Table Get                  | Retrieves processed messages for building history | Is Max ID?                    | Merge History                   | Describes message batching and history aggregation                                                             |
| Merge History          | Code                            | Creates chat history array for AI context          | Get history message            | Process Merged Message            | Describes message batching and history aggregation                                                             |
| Send Typing            | HTTP Request                     | Sends "typing_on" indicator to Facebook            | Is Max ID?                    | None                           | AI Processing & Enhanced Interaction                                                                            |
| Process Merged Message | AI Agent (Langchain Agent)       | Processes merged message with Google Gemini AI     | Merge History                 | Format for Facebook Output        | AI Processing & Enhanced Interaction                                                                            |
| Google Gemini Chat Model| Langchain LLM Chat               | Provides AI language model for agent node          | Process Merged Message (ai_languageModel) | None                       | AI Processing & Enhanced Interaction                                                                            |
| Groq Chat Model        | Langchain LLM Chat               | Alternate AI model (not connected in this workflow)| None                         | None                           | AI Processing & Enhanced Interaction                                                                            |
| Format for Facebook Output| Code                          | Formats AI output for Facebook Messenger           | Process Merged Message         | Send Text                      | Deliver Response & Update Batching Status                                                                       |
| Send Text              | HTTP Request                     | Sends AI reply to Facebook Messenger                | Format for Facebook Output     | Update Page Rep, Update FALSE to TRUE | Deliver Response & Update Batching Status                                                                       |
| Update Page Rep        | Data Table Update               | Saves bot reply in Data Table                        | Send Text                     | None                           | Deliver Response & Update Batching Status                                                                       |
| Update FALSE to TRUE   | Data Table Update               | Marks processed messages as processed (true)       | Send Text                     | None                           | Deliver Response & Update Batching Status                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Facebook Webhook Listener Node**  
   - Type: Webhook  
   - Path: `nguyenthieutoan-facebook-page`  
   - Methods: Support GET and POST  
   - Response Mode: Response Node

2. **Add Confirm Webhook Node**  
   - Type: Respond To Webhook  
   - Respond With: Text  
   - Response Body: `={{ $json.query["hub.challenge"] }}`  
   - Connect Facebook Webhook GET output to this node.

3. **Add "Is message?" If Node**  
   - Condition: Check if `$json.body.entry[0].messaging[0].message.text` exists and is non-empty.  
   - Connect Facebook Webhook POST output to this node.

4. **Add "Is from page?" If Node**  
   - Condition: Check if `$json.body.entry[0].messaging[0].message.is_echo` is true.  
   - Filter out echo messages.  
   - Connect true branch to stop; false branch continues.

5. **Add Set Context Node**  
   - Mode: Raw  
   - JSON Output:  
     ```json
     {
       "user_id": "{{ $json.body.entry[0].messaging[0].sender.id }}",
       "user_text": "{{ $json.body.entry[0].messaging[0].message.text }}",
       "page_id": "{{ $json.body.entry[0].id }}",
       "page_token": "<YOUR_TOKEN>"
     }
     ```  
   - Connect from false branch of "Is from page?".

6. **Add "Seen" HTTP Request Node**  
   - Method: POST  
   - URL: `https://graph.facebook.com/v23.0/{{ $json.page_id }}/messages`  
   - JSON Body:  
     ```json
     {
       "recipient": { "id": "{{ $json.user_id }}" },
       "sender_action": "mark_seen"
     }
     ```  
   - Query Parameter: `access_token = {{ $json.page_token }}`  
   - Connect from Set Context node.

7. **Add "Insert To Process" Data Table Node**  
   - Operation: Insert  
   - Data Table: `Batch_messages`  
   - Columns:  
     - user_id: `={{ $json.user_id }}`  
     - user_text: `={{ $json.user_text }}`  
     - processed: false (default)  
     - bot_rep: empty  
   - Connect from Set Context node.

8. **Add "Wait 3s" Node**  
   - Default delay 3 seconds  
   - Connect from "Insert To Process".

9. **Add "Get unprocessed message" Data Table Node**  
   - Operation: Get  
   - Data Table: `Batch_messages`  
   - Filters: `user_id` equals current user_id, `processed` is false  
   - Limit: 10  
   - Connect from "Wait 3s".

10. **Add "Get Max ID + Merged Mess" Code Node**  
    - Copy provided JavaScript code for sorting and merging messages.  
    - Connect from "Get unprocessed message".

11. **Add "Is Max ID?" If Node**  
    - Condition: Check if `$json.id` equals `$('Insert To Process').first().json.id` (or use a proper reference to max ID)  
    - Connect from "Get Max ID + Merged Mess".

12. **Add "Send Typing" HTTP Request Node**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/{{ $('Set Context').item.json.page_id }}/messages`  
    - JSON Body:  
      ```json
      {
        "recipient": { "id": "{{ $('Set Context').item.json.user_id }}" },
        "sender_action": "typing_on"
      }
      ```  
    - Query Parameter: `access_token = {{ $('Set Context').first().json.page_token }}`  
    - Connect true branch from "Is Max ID?".

13. **Add "Get history message" Data Table Node**  
    - Operation: Get  
    - Data Table: `Batch_messages`  
    - Filters: `user_id` equals current user_id, `processed` is true  
    - Limit: 10  
    - Connect true branch from "Is Max ID?" (parallel to "Send Typing").

14. **Add "Merge History" Code Node**  
    - Copy provided JS code to create chat history array from previous messages.  
    - Connect from "Get history message".

15. **Add "Process Merged Message" AI Agent Node**  
    - Text input: merged message from max batch item  
    - System prompt: Use provided persona and conversation flow text (Vietnamese AI assistant "Jenix")  
    - AI Language Model: Connect Google Gemini Chat Model node  
    - Retry on failure enabled with delay  
    - Connect from "Merge History".

16. **Add "Google Gemini Chat Model" Node**  
    - Langchain Google Gemini LLM Chat node  
    - Default configuration  
    - Connect as AI language model input to AI Agent node.

17. **Add "Format for Facebook Output" Code Node**  
    - Copy provided JS code for converting AI output to Facebook Markdown and splitting long messages.  
    - Connect from "Process Merged Message".

18. **Add "Send Text" HTTP Request Node**  
    - Method: POST  
    - URL: `https://graph.facebook.com/v23.0/{{ $('Set Context').first().json.page_id }}/messages`  
    - JSON Body: formatted message text chunks  
    - Query Parameter: `access_token = {{ $('Set Context').first().json.page_token }}`  
    - Connect from "Format for Facebook Output".

19. **Add "Update Page Rep" Data Table Node**  
    - Operation: Update  
    - Data Table: `Batch_messages`  
    - Filters: `user_id` equals current user_id, `id` equals max message id  
    - Update: `bot_rep` with AI response text  
    - Connect from "Send Text".

20. **Add "Update FALSE to TRUE" Data Table Node**  
    - Operation: Update  
    - Data Table: `Batch_messages`  
    - Filters: `user_id` equals current user_id, `processed` is false  
    - Update: `processed` field set to true  
    - Connect from "Send Text".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| **Facebook App & Messenger Webhook Setup Guide:** Steps to create a Facebook App, configure Messenger product, set webhook URL, verify token, subscribe page events, and generate page token. | https://developers.facebook.com/ |
| **Data Table Setup:** Create a Data Table named `Batch_messages` with columns `id` (auto-increment primary key), `user_id` (string), `user_text` (string), `bot_rep` (string), `processed` (boolean), plus automatic timestamps `createdAt` and `updatedAt`. | n8n UI -> Data tables section |
| **Workflow Description and Detailed Guide:** Comprehensive workflow description, setup instructions, and troubleshooting notes by Nguyen Thieu Toan. | https://nguyenthieutoan.com/share-n8n-workflow-facebook-messenger-smart-chatbot-use-message-batching-table-data |
| **Author:** Nguyen Thieu Toan is an AI automation and chatbot expert with deep n8n knowledge, specializing in business process optimization and conversational AI. | Personal profile and blog |

---

This documentation enables advanced users and AI agents to fully understand, reproduce, and maintain the Smart Facebook Messenger Chatbot with Gemini AI, message batching, and history aggregation in n8n. It anticipates authentication issues, API rate limits, data synchronization challenges, and provides a solid foundation for further customization or extension.