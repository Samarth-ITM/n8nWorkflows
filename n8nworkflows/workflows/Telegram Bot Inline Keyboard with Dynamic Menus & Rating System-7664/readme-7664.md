Telegram Bot Inline Keyboard with Dynamic Menus & Rating System

https://n8nworkflows.xyz/workflows/telegram-bot-inline-keyboard-with-dynamic-menus---rating-system-7664


# Telegram Bot Inline Keyboard with Dynamic Menus & Rating System

### 1. Workflow Overview

This workflow implements a sophisticated Telegram Bot that uses inline keyboards with dynamic menus and a star-based rating system. It is designed to interactively engage users by presenting multi-level menus, personalized greetings, feedback collection, and feature ratings within Telegram chats. The workflow handles both regular messages and callback queries (button clicks) to update messages dynamically or send new messages accordingly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Listens to incoming Telegram updates including regular messages and callback queries.
- **1.2 Response Preparation:** Processes incoming data to extract user details and dynamically builds the appropriate menu response (text + inline keyboard) based on user input or button clicks.
- **1.3 Bot Token Management:** Stores and supplies the Telegram Bot API token securely for API calls.
- **1.4 Message Routing:** Determines whether the incoming update is a callback query or a new message to decide the next step.
- **1.5 Telegram API Communication:** Sends or edits messages on Telegram via HTTP requests to Telegram’s Bot API, supporting dynamic inline keyboards.
- **1.6 Callback Query Acknowledgement:** Sends a callback query answer to Telegram to remove the loading animation after button presses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives Telegram updates, including messages and callback queries, triggering the workflow to start processing.

- **Nodes Involved:**  
  - TG Trigger node

- **Node Details:**  
  - **TG Trigger node**  
    - Type: Telegram Trigger  
    - Role: Listens for Telegram updates of type "message" and "callback_query".  
    - Configuration: Uses the Telegram credentials set up for the bot.  
    - Inputs: None (trigger node)  
    - Outputs: Emits JSON data representing Telegram update received.  
    - Edge cases/failures: Authentication failure if Telegram credentials are invalid; webhook not set or network issues may prevent receiving updates.

#### 2.2 Response Preparation

- **Overview:**  
Processes incoming Telegram update data and constructs a dynamic response message and inline keyboard tailored to the user’s input or button clicks.

- **Nodes Involved:**  
  - Prepare Magic Response (Function node)

- **Node Details:**  
  - **Prepare Magic Response**  
    - Type: Function  
    - Role: Analyzes incoming data (message or callback query), extracts user names, determines the appropriate response text and inline keyboard layout, and prepares parameters for Telegram API calls (sendMessage or editMessageText).  
    - Configuration: Contains JavaScript code implementing a switch-case logic on callback data or message text to build multi-level menus and a 1-5 star rating system.  
    - Key expressions: Uses Telegram user’s first_name and last_name to personalize responses; detects commands or callback data such as 'feature1', 'rate_1'...'rate_5', 'main', 'settings', etc.  
    - Input: JSON from TG Trigger node (Telegram update)  
    - Output: JSON containing API method ('sendMessage' or 'editMessageText'), request body for Telegram API, callback query ID if applicable, and flag isCallback.  
    - Edge cases: Missing user names fallback to "User"; unknown callback data defaults to main menu; potential failures if Telegram message structure changes or unexpected data formats appear.

#### 2.3 Bot Token Management

- **Overview:**  
Stores the Telegram Bot API token for use in subsequent HTTP API requests.

- **Nodes Involved:**  
  - Set Bot Token API KEY

- **Node Details:**  
  - **Set Bot Token API KEY**  
    - Type: Set node  
    - Role: Holds the bot token string as a workflow variable `botToken`.  
    - Configuration: The user must replace the placeholder string with their actual Telegram Bot API token.  
    - Input: JSON from Prepare Magic Response node  
    - Output: JSON augmented with `botToken` for use by HTTP Request nodes.  
    - Edge cases: Workflow will fail to authenticate with Telegram API if token is incorrect or missing. Security note: never share the token publicly.  

#### 2.4 Message Routing

- **Overview:**  
Determines if the incoming update is a callback query (button press) or a new message to decide the appropriate Telegram API method and subsequent nodes.

- **Nodes Involved:**  
  - Is CALLBACK? (If node)

- **Node Details:**  
  - **Is CALLBACK?**  
    - Type: If  
    - Role: Checks the boolean flag `isCallback` from the previous node to branch the flow.  
    - Configuration: Condition is `$json.isCallback == true`  
    - Input: JSON from Set Bot Token API KEY node  
    - Output: Two branches: TRUE (callback query path), FALSE (new message path)  
    - Edge cases: If flag missing or malformed, default branch may be incorrect.  

#### 2.5 Telegram API Communication

- **Overview:**  
Sends or edits messages on Telegram using direct HTTP requests, enabling dynamic inline keyboards not supported by the native Telegram node.

- **Nodes Involved:**  
  - Send to Telegram API (HTTP Request node)

- **Node Details:**  
  - **Send to Telegram API**  
    - Type: HTTP Request  
    - Role: Performs POST requests to Telegram Bot API endpoints `sendMessage` or `editMessageText`, using the bot token and request body prepared earlier.  
    - Configuration: URL dynamically built with bot token and API method from JSON input; sends JSON request body with message text, parse mode HTML, and inline keyboard.  
    - Input: JSON from both Is CALLBACK? branches  
    - Output: Telegram API response  
    - Edge cases: HTTP errors (network issues, invalid token, malformed request body), rate limits, Telegram API downtime.  
    - Note: Native Telegram node is not used due to lack of dynamic keyboard support.

#### 2.6 Callback Query Acknowledgement

- **Overview:**  
Sends an acknowledgment to Telegram to remove the loading spinner after a callback button press, improving user experience.

- **Nodes Involved:**  
  - Answer Callback (HTTP Request node)

- **Node Details:**  
  - **Answer Callback**  
    - Type: HTTP Request  
    - Role: Calls Telegram API method `answerCallbackQuery` with the callback_query_id and a simple confirmation text to remove the loading animation.  
    - Configuration: URL built with bot token from Set Bot Token API KEY node; JSON body includes callback_query_id from Prepare Magic Response and a checkmark text "✅".  
    - Input: JSON from Send to Telegram API node (TRUE branch of Is CALLBACK?)  
    - Output: Telegram API response  
    - Edge cases: Failure to acknowledge callback may cause loading spinner to persist; token or network issues can cause errors.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                         | Input Node(s)              | Output Node(s)                    | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|----------------------|---------------------|---------------------------------------|----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| TG Trigger node       | Telegram Trigger    | Receives Telegram messages & callbacks | None                       | Prepare Magic Response           |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Prepare Magic Response| Function            | Builds dynamic responses & keyboards  | TG Trigger node             | Set Bot Token API KEY            | ## 🎯 Dynamic Menu Builder  This node processes incoming messages and builds dynamic menus  Features: Extracts real user names, creates dynamic keyboards, rating system, multiple menu levels. Callback data format explained.                                                                                                                                                                                                                                                                                                                                                                               |
| Set Bot Token API KEY | Set                 | Stores Telegram bot API token         | Prepare Magic Response      | Is CALLBACK?                    | ## 🔑 Bot Token Configuration  IMPORTANT: Replace with your bot token! How to get token instructions. Security note: Never share your bot token publicly!                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Is CALLBACK?          | If                  | Routes based on message or callback   | Set Bot Token API KEY       | Send to Telegram API, Answer Callback (TRUE branch); Send to Telegram API (FALSE branch) | ## 🚦 Message Router  Checks if incoming data is callback query or message; routes accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Send to Telegram API  | HTTP Request        | Sends or edits Telegram messages      | Is CALLBACK?                | Answer Callback (TRUE branch); none (FALSE branch) | ## 📡 Telegram API Sender  Sends messages with inline keyboards using Telegram API methods sendMessage or editMessageText. HTTP Request node used because n8n Telegram node lacks dynamic keyboard support.                                                                                                                                                                                                                                                                                                                                                                                            |
| Answer Callback       | HTTP Request        | Acknowledges callback queries         | Send to Telegram API (TRUE) | None                           | ## ✅ Callback Answer  Removes loading animation after button click. Sends confirmation to Telegram callback query.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note1          | Sticky Note         | Documentation                         | None                       | None                           | ## 🎯 Dynamic Menu Builder  This node processes incoming messages and builds dynamic menus (same content as Prepare Magic Response explanation).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Sticky Note           | Sticky Note         | Documentation                         | None                       | None                           | ## 🔑 Bot Token Configuration  Instructions on how to get and set Telegram bot token securely.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note 6         | Sticky Note         | Documentation                         | None                       | None                           | ## 🚦 Message Router  Explains routing logic for callback vs. message updates.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Sticky Note 7         | Sticky Note         | Documentation                         | None                       | None                           | ## 📡 Telegram API Sender  Explains why HTTP Request is used to send messages with inline keyboards.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| Sticky Note 8         | Sticky Note         | Documentation                         | None                       | None                           | ## ✅ Callback Answer  Explains the purpose of sending callback query acknowledgments to remove loading animation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| Sticky Note Main1     | Sticky Note         | Documentation                         | None                       | None                           | # 🤖 Telegram Bot Inline Keyboard with Dynamic Menus & Rating System  Quick start and features. Troubleshooting tips included.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Set updates to listen for `message` and `callback_query`  
   - Credentials: Set your Telegram bot credentials (OAuth2 with your bot token)  
   - Position: Start of workflow

2. **Add a Function node named "Prepare Magic Response"**  
   - Connect from TG Trigger node  
   - Paste the provided JavaScript code that:  
     - Extracts user first and last names  
     - Determines if input is message or callback query  
     - Builds dynamic response texts and inline keyboards based on input text or callback data  
     - Sets API method (sendMessage or editMessageText) accordingly  
     - Outputs JSON with `apiMethod`, `requestBody`, `isCallback`, and `callbackQueryId`  

3. **Add a Set node named "Set Bot Token API KEY"**  
   - Connect from Prepare Magic Response  
   - Assign a string variable `botToken` with your Telegram Bot API token (replace placeholder)  
   - Ensure the token is kept secret and updated here if changed

4. **Add an If node named "Is CALLBACK?"**  
   - Connect from Set Bot Token API KEY  
   - Condition: Boolean check if `{{$json.isCallback}} == true`  
   - This node will split the flow into two branches: TRUE for callback queries, FALSE for new messages

5. **Add an HTTP Request node named "Send to Telegram API"**  
   - Connect TRUE and FALSE outputs of "Is CALLBACK?" to this node  
   - Configure:  
     - Method: POST  
     - URL: `https://api.telegram.org/bot{{ $json.botToken }}/{{ $json.apiMethod }}` (dynamic)  
     - Body content type: JSON  
     - Body: use `{{ JSON.stringify($json.requestBody) }}` expression to send dynamic request body  
   - This node sends or edits messages with dynamic inline keyboards

6. **Add an HTTP Request node named "Answer Callback"**  
   - Connect from "Send to Telegram API" node only on TRUE branch of "Is CALLBACK?" (callback queries)  
   - Configure:  
     - Method: POST  
     - URL: `https://api.telegram.org/bot{{ $('Set Bot Token API KEY').item.json.botToken }}/answerCallbackQuery`  
     - Body JSON:  
       ```json
       {
         "callback_query_id": "{{ $('Prepare Magic Response').item.json.callbackQueryId }}",
         "text": "✅"
       }
       ```  
   - This node acknowledges callback queries to remove loading animation

7. **Optional: Add Sticky Note nodes for documentation at relevant points**  
   - Place notes explaining dynamic menu logic, bot token setup, message routing, Telegram API usage, and callback acknowledgments  
   - Use colors and sizes for clarity

8. **Activate the workflow**  
   - Ensure Telegram webhook is set up and bot token is valid  
   - Test by sending messages or clicking inline keyboard buttons in your Telegram bot

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                | Context or Link                                            |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Replace the placeholder `PLACE YOUR TELEGRAM API KEY HERE` in the Set node with your actual Telegram Bot token obtained from @BotFather. Never share this token publicly.                                                                                                                                                   | Bot Token Configuration Sticky Note                         |
| Since n8n's native Telegram node does not support dynamic inline keyboards, the workflow uses direct Telegram Bot API HTTP requests (`sendMessage` and `editMessageText`) to implement complex interactive menus.                                                                                                         | Telegram API Sender Sticky Note                             |
| When users click inline keyboard buttons, Telegram shows a loading spinner. To remove it, the workflow must respond with `answerCallbackQuery`. This workflow acknowledges callbacks with a checkmark "✅".                                                                                                               | Callback Answer Sticky Note                                 |
| The workflow implements a multi-level menu system with commands like "Help", "Settings", "Statistics", and a 1-5 star rating system with feedback options. It dynamically personalizes messages using the user's real name extracted from the Telegram update payload.                                                      | Dynamic Menu Builder Sticky Note                            |
| Common issues include missing keyboards (check token), loading spinners not disappearing (ensure callback answer node runs), and unclickable buttons (ensure workflow is active).                                                                                                                                           | Main Sticky Note with troubleshooting tips                 |
| Telegram Bot API documentation: https://core.telegram.org/bots/api                                                                                                                                                                                                                                                        | Official Telegram Bot API                                 |

---

This completes the detailed reference documentation of the "Telegram Bot Inline Keyboard with Dynamic Menus & Rating System" n8n workflow. It covers the entire structure, logic, and instructions to reproduce or modify the workflow securely and effectively.