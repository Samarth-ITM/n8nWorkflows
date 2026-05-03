Build a Telegram Subscription Gate with Lead Magnet and Upsell Sequence

https://n8nworkflows.xyz/workflows/build-a-telegram-subscription-gate-with-lead-magnet-and-upsell-sequence-10598


# Build a Telegram Subscription Gate with Lead Magnet and Upsell Sequence

---

### 1. Workflow Overview

This workflow implements a **Telegram subscription gate with a lead magnet delivery and upsell/cross-sell sequence** tailored for Telegram channel marketing. It is designed for channel owners who want to verify if users are subscribed to a private Telegram channel before granting access to exclusive bonuses (lead magnet). The workflow also includes an upsell sequence to promote additional offers after subscription confirmation.

**Primary use cases:**
- Lead generation by gating content behind Telegram channel subscription
- Delivering bonuses or coupons upon subscription confirmation
- Engaging users with upsell or cross-sell marketing messages
- Handling subscription check retries and errors gracefully
- Supporting bilingual messaging (Ukrainian & English)

**Logical blocks:**

1.1 **Telegram Input Reception**  
- Listening for Telegram messages and callback queries that trigger subscription checks or upsell commands.

1.2 **User Data Extraction & Callback Acknowledgement**  
- Parsing incoming Telegram updates to extract user IDs and callback info, and acknowledging button presses for user experience.

1.3 **Subscription Verification via Telegram API**  
- Querying Telegram’s `getChatMember` API to check if the user is subscribed to the specified private channel.

1.4 **Subscription Status Validation and Routing**  
- Evaluating API response to determine if the user is a valid subscriber and routing accordingly:
  - If subscribed → Send lead magnet bonus message
  - If not subscribed → Send instructions to subscribe and retry button

1.5 **Upsell/Cross-sell Marketing Sequence**  
- Triggered by a specific user command (`/more`), sends a series of upsell messages with delays and typing indicators.

1.6 **Bot Configuration & Documentation**  
- Sticky notes with detailed instructions on Telegram bot setup, permissions, channel ID format, and links to documentation and tutorial videos.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Telegram Input Reception

**Overview:**  
This block listens for user interactions via Telegram messages or inline button presses that initiate subscription checks or upsell commands.

**Nodes Involved:**  
- 🔔 Telegram Message Trigger | Тригер повідомлень Телеграм2  
- ❓ Is Subscription Check Request? | Запит на перевірку підписки?2  
- ❓ Is /ok Command? | Команда /more?2  

**Node Details:**

- **🔔 Telegram Message Trigger | Тригер повідомлень Телеграм2**  
  - Type: Telegram Trigger node  
  - Role: Listens for incoming Telegram updates: messages and callback queries  
  - Configuration: Watches for updates of types "message" and "callback_query"  
  - Credentials: Uses Telegram bot credentials for authorized webhook  
  - Input: External Telegram updates via webhook  
  - Output: Passes update JSON downstream  
  - Failure modes: Network issues, webhook misconfiguration, invalid credentials  

- **❓ Is Subscription Check Request? | Запит на перевірку підписки?2**  
  - Type: If node  
  - Role: Checks if incoming callback query data equals "check_subscription"  
  - Configuration: Checks `{{ $json.callback_query?.data }}` equals "check_subscription" string  
  - Input: From Telegram trigger  
  - Output: Two branches - true (subscription check requested), false (not requested)  
  - Edge cases: Missing callback data, malformed JSON  

- **❓ Is /ok Command? | Команда /more?2**  
  - Type: If node  
  - Role: Checks if incoming message text equals "/more" command (upsell trigger)  
  - Configuration: Checks `{{ $json.message?.text || '' }}` equals "/more" string  
  - Input: From Telegram trigger (false branch of subscription check)  
  - Output: True triggers upsell sequence, false triggers subscription gate  
  - Edge cases: Case sensitivity, command variations  

---

#### 1.2 User Data Extraction & Callback Acknowledgement

**Overview:**  
Extracts critical user identifiers and callback query IDs from Telegram updates and acknowledges button presses to improve UX.

**Nodes Involved:**  
- 🔧 Extract User & Callback Data | Витяг даних користувача2  
- ✅ Acknowledge Button Click | Підтвердження натискання кнопки2  

**Node Details:**

- **🔧 Extract User & Callback Data | Витяг даних користувача2**  
  - Type: Code node (JavaScript)  
  - Role: Parses incoming Telegram JSON to extract user ID, chat ID (channel ID), and callback query ID  
  - Configuration:  
    - Extracts user ID from message, callback_query, or result fields  
    - Sets chat_id to channel numeric ID `-1001234567890` (placeholder to replace)  
    - Extracts callback_query_id if available  
    - Throws error if user ID not found  
  - Input: Telegram update JSON  
  - Output: JSON object with user_id, chat_id, callback_query_id fields  
  - Edge cases: Missing or malformed Telegram update data, invalid user or channel ID  

- **✅ Acknowledge Button Click | Підтвердження натискання кнопки2**  
  - Type: HTTP Request node  
  - Role: Calls Telegram API `answerCallbackQuery` to acknowledge inline button press with a "Checking subscription..." message  
  - Configuration:  
    - URL constructed with bot token  
    - Query parameters: callback_query_id and confirmation text  
  - Input: From subscription check request detection  
  - Output: Telegram API response (usually ignored)  
  - Edge cases: Telegram API rate limits, invalid callback_query_id, network failures  

---

#### 1.3 Subscription Verification via Telegram API

**Overview:**  
Queries Telegram API to verify if the extracted user is a member of the specified private Telegram channel.

**Nodes Involved:**  
- Closed Channel (HTTP Request)  
- 🌐 Check Telegram Subscription | Перевірка підписки в Телеграм2 (disabled)  

**Node Details:**

- **Closed Channel**  
  - Type: HTTP Request node  
  - Role: Calls Telegram Bot API `getChatMember` with channel ID and user ID to check membership status  
  - Configuration:  
    - URL: `https://api.telegram.org/bot{TOKEN}/getChatMember`  
    - Query parameters: `chat_id` (private channel numeric ID), `user_id` (extracted user)  
  - Input: From user data extraction node  
  - Output: Telegram API JSON response containing membership status (`member`, `administrator`, `creator`, `restricted`, etc.)  
  - Edge cases: Invalid token, channel ID, user ID; user not found; bot lacking permissions  
  - Note: The alternative node "🌐 Check Telegram Subscription" is disabled, presumably replaced by this node  

---

#### 1.4 Subscription Status Validation and Routing

**Overview:**  
Validates the Telegram API membership status and routes the workflow to send either the lead magnet or subscription failure instructions.

**Nodes Involved:**  
- ✅ Is User Subscribed? | Користувач підписаний?2  
- ✅ Subscription Confirmed - Send Lead Magnet | Підписка підтверджена2  
- ❌ Subscription Check Failed | Перевірка підписки не вдалася2  

**Node Details:**

- **✅ Is User Subscribed? | Користувач підписаний?2**  
  - Type: If node  
  - Role: Checks if membership status is one of accepted values: "member", "administrator", "creator", or "restricted"  
  - Configuration: Logical OR condition over membership status field from `getChatMember` response  
  - Input: From Telegram API response node  
  - Output:  
    - True branch → user is subscribed  
    - False branch → user is not subscribed  
  - Edge cases: Unexpected membership status, API errors  

- **✅ Subscription Confirmed - Send Lead Magnet | Підписка підтверджена2**  
  - Type: Telegram node (send message)  
  - Role: Sends the lead magnet bonus message with instructions and clickable link for the bonus coupon (800 UAH SendPulse)  
  - Configuration:  
    - Message text with detailed step-by-step instructions in Ukrainian and English  
    - Dynamic chat ID from Telegram API result user ID  
    - Force reply disabled, no attribution appended  
  - Input: True branch from subscription validation  
  - Output: Sends Telegram message to user  
  - Edge cases: User blocked bot, chat ID invalid, message too long  

- **❌ Subscription Check Failed | Перевірка підписки не вдалася2**  
  - Type: Telegram node (send message)  
  - Role: Sends message informing user is not subscribed with instructions to subscribe and retry button  
  - Configuration:  
    - Message text in Ukrainian explaining how to join the channel and retry  
    - Inline keyboard with a button labeled "Try Again" that triggers subscription check callback  
    - Dynamic chat ID from Telegram API result user ID  
  - Input: False branch from subscription validation  
  - Output: Sends Telegram message with retry button  
  - Edge cases: User ignores instructions, button callback fails  

---

#### 1.5 Upsell/Cross-sell Marketing Sequence

**Overview:**  
Handles upsell/cross-sell messaging triggered by the `/more` command, sending a sequence of promotional messages with typing indicators and delays.

**Nodes Involved:**  
- 📈 Upsell/CrossSell/DownSell System | Система допродажів  
- 📈 Upsell/CrossSell/DownSell System | Система допродажів2  
- 🎁 Premium Offer Message | Повідомлення преміум пропозиції2  
- Wait, Wait1  
- HTTP typing..., HTTP typing...1  

**Node Details:**

- **❓ Is /ok Command? | Команда /more?2**  
  - (Already described in 1.1) Routes to upsell if true.

- **📈 Upsell/CrossSell/DownSell System | Система допродажів**  
  - Type: Telegram node  
  - Role: Sends initial upsell message describing channel benefits and inviting to join  
  - Configuration: Message text promoting channel content and mission  
  - Input: From `/more` command conditional node  
  - Output: Passes to Wait and HTTP typing nodes  

- **Wait, Wait1**  
  - Type: Wait node  
  - Role: Introduces delays (20-25 seconds) between upsell messages to simulate conversational pacing  
  - Input: From upsell message nodes  
  - Output: Passes to next upsell message or typing indicator  

- **HTTP typing..., HTTP typing...1**  
  - Type: HTTP Request node  
  - Role: Sends Telegram `sendChatAction` API calls with action "typing" to simulate typing indicator to user  
  - Configuration: POST to `sendChatAction` with chat ID and action "typing"  
  - Input: After wait nodes  
  - Output: Passes to next upsell message or wait node  

- **📈 Upsell/CrossSell/DownSell System | Система допродажів2**  
  - Type: Telegram node  
  - Role: Sends secondary upsell message continuing the marketing pitch  
  - Configuration: Text with additional promotional content, links to channel  
  - Input: From `/more` command conditional node or after first upsell message sequence  
  - Output: Passes to typing and wait nodes to continue sequence  

- **🎁 Premium Offer Message | Повідомлення преміум пропозиції2**  
  - Type: Telegram node  
  - Role: Final upsell premium offer message with link to setup template bot and instructions  
  - Configuration: Message text with link to the lead magnet bot template and encouragement to act  
  - Input: After Wait node in upsell sequence  
  - Output: Ends upsell sequence  
  - Edge cases: User ignores upsell, link broken  

---

#### 1.6 Bot Configuration & Documentation (Sticky Notes)

**Overview:**  
Contains multiple sticky notes with configuration instructions, workflow documentation, logic explanation, and YouTube tutorial placeholders.

**Nodes Involved:**  
- 🔐 Bot Configuration | Налаштування бота1 and 2 (two nodes)  
- 📚 Workflow Documentation | Документація воркфлоу2  
- 🧠 Logic Explanation | Пояснення логіки2  
- 📹 YouTube Integration | Інтеграція YouTube4  

**Node Details:**

- **🔐 Bot Configuration | Налаштування бота1 and 2**  
  - Type: Sticky Note  
  - Role: Detailed instructions on how to setup Telegram bot permissions for private channels, including:  
    - Adding bot as channel admin with required permissions (Read messages, View members, Post messages)  
    - Proper channel ID format (`-1001234567890`) for private channels  
    - How to get private channel ID using @getidsbot  
    - Security tips for bot token storage  
    - Troubleshooting tips for subscription check failures  

- **📚 Workflow Documentation | Документація воркфлоу2**  
  - Type: Sticky Note  
  - Role: High-level workflow overview, use cases, logic steps, and configuration notes including placeholders to replace channel ID and tokens  

- **🧠 Logic Explanation | Пояснення логіки2**  
  - Type: Sticky Note  
  - Role: Explains node logic and workflow flow in brief sentences to help users understand the implementation and bilingual support  

- **📹 YouTube Integration | Інтеграція YouTube4**  
  - Type: Sticky Note  
  - Role: Placeholder for embedding a tutorial video with description and thumbnail (link needs to be updated)  

---

### 3. Summary Table

| Node Name                                           | Node Type            | Functional Role                                      | Input Node(s)                          | Output Node(s)                                     | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                              |
|----------------------------------------------------|----------------------|-----------------------------------------------------|--------------------------------------|---------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🔔 Telegram Message Trigger | Тригер повідомлень Телеграм2 | Telegram Trigger listening for messages and callbacks | -                                    | ❓ Is Subscription Check Request?, ❓ Is /ok Command?  |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ❓ Is Subscription Check Request? | Запит на перевірку підписки?2 | Checks if callback data equals "check_subscription"  | 🔔 Telegram Message Trigger           | ✅ Acknowledge Button Click, 🔧 Extract User & Callback Data, ❓ Is /ok Command? |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ✅ Acknowledge Button Click | Підтвердження натискання кнопки2 | Acknowledges button press via Telegram API           | ❓ Is Subscription Check Request?     | 🔧 Extract User & Callback Data                      |                                                                                                                                                                                                                                                                                                                                                                                                          |
| 🔧 Extract User & Callback Data | Витяг даних користувача2       | Extracts user ID, chat ID, and callback query ID     | ✅ Acknowledge Button Click            | Closed Channel                                     |                                                                                                                                                                                                                                                                                                                                                                                                          |
| Closed Channel                                     | HTTP Request          | Calls Telegram API to get user subscription status   | 🔧 Extract User & Callback Data        | ✅ Is User Subscribed?                              |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ✅ Is User Subscribed? | Користувач підписаний?2           | Validates if user is subscribed based on status      | Closed Channel                       | ✅ Subscription Confirmed - Send Lead Magnet, ❌ Subscription Check Failed |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ✅ Subscription Confirmed - Send Lead Magnet | Підписка підтверджена2      | Sends lead magnet bonus message to subscribed users  | ✅ Is User Subscribed? (true)          | -                                                 |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ❌ Subscription Check Failed | Перевірка підписки не вдалася2   | Sends subscription failure message with retry button | ✅ Is User Subscribed? (false)         | -                                                 |                                                                                                                                                                                                                                                                                                                                                                                                          |
| ❓ Is /ok Command? | Команда /more?2                  | Checks if user sent "/more" command for upsell       | ❓ Is Subscription Check Request? (false) | 📈 Upsell/CrossSell/DownSell System, 🔒 Lock Permissions |                                                                                                                                                                                                                                                                                                                                                                                                          |
| 📈 Upsell/CrossSell/DownSell System | Система допродажів         | Sends initial upsell promotional message             | ❓ Is /ok Command? (true)              | Wait1, HTTP typing...1                             |                                                                                                                                                                                                                                                                                                                                                                                                          |
| Wait1                                              | Wait                  | Wait delay before next upsell message                 | 📈 Upsell/CrossSell/DownSell System    | 📈 Upsell/CrossSell/DownSell System 2              |                                                                                                                                                                                                                                                                                                                                                                                                          |
| 📈 Upsell/CrossSell/DownSell System | Система допродажів2        | Sends secondary upsell promotional message           | Wait1                                | HTTP typing...1, Wait1                              |                                                                                                                                                                                                                                                                                                                                                                                                          |
| Wait                                               | Wait                  | Wait delay before premium offer message               | 📈 Upsell/CrossSell/DownSell System  | 🎁 Premium Offer Message                            |                                                                                                                                                                                                                                                                                                                                                                                                          |
| HTTP typing...                                     | HTTP Request          | Sends "typing" chat action to Telegram                | Wait                                | -                                                 |                                                                                                                                                                                                                                                                                                                                                                                                          |
| HTTP typing...1                                    | HTTP Request          | Sends "typing" chat action to Telegram                | 📈 Upsell/CrossSell/DownSell System 2 | Wait1                                              |                                                                                                                                                                                                                                                                                                                                                                                                          |
| 🎁 Premium Offer Message | Повідомлення преміум пропозиції2  | Sends final upsell premium offer message              | Wait                                | -                                                 |                                                                                                                                                                                                                                                                                                                                                                                                          |
| 🔒 Lock Permissions | Блокування дозволів2             | Sends message prompting access request                | ❓ Is /ok Command? (false branch)      | -                                                 |                                                                                                                                                                                                                                                                                                                                                                                                          |
| 🔐 Bot Configuration | Налаштування бота1              | Sticky note with bot setup instructions                | -                                    | -                                                 | Configuration details for Telegram bot permissions, channel ID format, token security, and troubleshooting.                                                                                                                                                                                                                                                                                              |
| 🔐 Bot Configuration | Налаштування бота2              | Sticky note with bot setup instructions (Ukrainian)   | -                                    | -                                                 | Ukrainian version of bot configuration instructions.                                                                                                                                                                                                                                                                                                                                                     |
| 📚 Workflow Documentation | Документація воркфлоу2         | Sticky note with overall workflow description          | -                                    | -                                                 | High-level documentation, use cases, and setup notes.                                                                                                                                                                                                                                                                                                                                                   |
| 🧠 Logic Explanation | Пояснення логіки2               | Sticky note explaining node logic                       | -                                    | -                                                 | Node-by-node explanation of workflow logic and bilingual support.                                                                                                                                                                                                                                                                                                                                      |
| 📹 YouTube Integration | Інтеграція YouTube4            | Sticky note with tutorial video placeholder            | -                                    | -                                                 | Placeholder for tutorial video link and description; requires updating link.                                                                                                                                                                                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Credentials in n8n:**  
   - Obtain your Telegram bot token from BotFather.  
   - Store credentials securely in n8n (e.g., named "SENDPUSLE 800 UAH EXTRA BONUX").  

2. **Add Telegram Message Trigger Node:**  
   - Node Type: Telegram Trigger  
   - Parameters: Listen for updates types `message` and `callback_query`.  
   - Credentials: Use Telegram bot credentials.  
   - Position: Place near workflow start.  

3. **Add If Node for Subscription Check Request:**  
   - Node Type: If  
   - Condition: Check if `{{ $json.callback_query?.data }}` equals `"check_subscription"`.  
   - Connect from Telegram Trigger node (main output).  

4. **Add If Node for Upsell Command:**  
   - Node Type: If  
   - Condition: Check if `{{ $json.message?.text || '' }}` equals `"/more"`.  
   - Connect from false output of subscription check If node.  

5. **Add HTTP Request Node to Acknowledge Button Click:**  
   - Node Type: HTTP Request  
   - URL: `https://api.telegram.org/bot{{ $credentials.telegramApi.token }}/answerCallbackQuery`  
   - Query Parameters:  
     - `callback_query_id` = `{{ $json.callback_query.id }}`  
     - `text` = "Перевіряємо підписку… (Checking subscription...)"  
   - Connect from true output of subscription check If node.  

6. **Add Code Node to Extract User and Callback Data:**  
   - Node Type: Code (JavaScript)  
   - Script: Extract user ID from various Telegram update fields, channel ID (replace with your private channel numeric ID, e.g., `-1001234567890`), and callback query ID. Throw error if user ID missing.  
   - Connect from HTTP Acknowledge node.  

7. **Add HTTP Request Node for Telegram API getChatMember:**  
   - Node Type: HTTP Request  
   - URL: `https://api.telegram.org/bot{{ $credentials.telegramApi.token }}/getChatMember`  
   - Query Parameters:  
     - `chat_id` = `{{ $json.chat_id }}` (your private channel ID)  
     - `user_id` = `{{ $json.user_id }}`  
   - Connect from Code node.  

8. **Add If Node to Validate Subscription Status:**  
   - Node Type: If  
   - Condition: Check if `{{ $json.result.status }}` is one of ["member", "administrator", "creator", "restricted"].  
   - Connect from Telegram API getChatMember node.  

9. **Create Telegram Node to Send Lead Magnet Message:**  
   - Node Type: Telegram  
   - Text: Message with bonus link and steps to redeem coupon, including call to action for `/more` command.  
   - Chat ID: `{{ $json.result.user.id }}`  
   - Connect from true branch of subscription status If node.  

10. **Create Telegram Node to Send Subscription Failure Message:**  
    - Node Type: Telegram  
    - Text: Instructions to subscribe to the Telegram channel with inline keyboard button "Try Again" having callback data "check_subscription".  
    - Chat ID: `{{ $json.result.user.id }}`  
    - Connect from false branch of subscription status If node.  

11. **Create Telegram Node for Upsell Initial Message:**  
    - Node Type: Telegram  
    - Text: Upsell/cross-sell marketing message with channel benefits and mission statement.  
    - Chat ID: `{{ $json.result.chat.id }}`  
    - Connect from true output of `/more` command If node.  

12. **Add Wait Node (20 seconds):**  
    - Node Type: Wait  
    - Amount: 20 seconds  
    - Connect from initial upsell message node.  

13. **Create Telegram Node for Upsell Secondary Message:**  
    - Node Type: Telegram  
    - Text: Additional upsell message with channel link and motivation.  
    - Chat ID: `{{ $json.result.chat.id }}`  
    - Connect from Wait node.  

14. **Add HTTP Request Node to Send Typing Action:**  
    - Node Type: HTTP Request  
    - URL: `https://api.telegram.org/bot{{ $credentials.telegramApi.token }}/sendChatAction`  
    - Method: POST  
    - Body Parameters: chat_id and action = "typing"  
    - Connect from upsell secondary message node.  

15. **Add Wait Node (25 seconds):**  
    - Node Type: Wait  
    - Amount: 25 seconds  
    - Connect from typing action node.  

16. **Create Telegram Node for Premium Offer Message:**  
    - Node Type: Telegram  
    - Text: Final upsell message with link to premium offer and encouragement to act.  
    - Chat ID: `{{ $json.result.chat.id }}`  
    - Connect from Wait node.  

17. **Create Telegram Node to Lock Permissions Message:**  
    - Node Type: Telegram  
    - Text: Message asking user to request access to bonus with inline button linking to subscription check.  
    - Chat ID: `{{ $json.message.from.id }}`  
    - Connect from false output of `/more` command If node.  

18. **Add Sticky Notes for Documentation and Bot Configuration:**  
    - Add nodes with detailed bot setup instructions (admin permissions, channel ID format, token handling, troubleshooting).  
    - Add notes with workflow overview, logic explanation, and YouTube tutorial placeholder.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Telegram bot must be administrator in private channel with permissions: Read messages, View members, and optionally Post messages. Use numeric channel ID starting with -100 for private channels. Obtain channel ID using @getidsbot.                                                                                                                                                | Sticky Notes 🔐 Bot Configuration                                                                                 |
| Store bot token securely in n8n credentials; do not hardcode tokens in nodes. Use credential references like `{{ $credentials.telegramApi.token }}`.                                                                                                                                                                                                                               | Sticky Notes 🔐 Bot Configuration                                                                                 |
| Subscription check failures often caused by incorrect channel ID format, missing bot admin rights, or token problems. Test API manually with curl for debugging.                                                                                                                                                                                                                  | Sticky Notes 🔐 Bot Configuration                                                                                 |
| Workflow is bilingual with Ukrainian and English messages to maximize audience reach.                                                                                                                                                                                                                                                                                             | All message nodes and sticky notes                                                                                 |
| Upsell sequence triggered by `/more` command sends multiple promotional messages with controlled delays and typing indicators for better user engagement.                                                                                                                                                                                                                         | Upsell block explanation                                                                                           |
| Tutorial video placeholder included - update with actual YouTube link and thumbnail for end user training.                                                                                                                                                                                                                                                                          | Sticky Note 📹 YouTube Integration                                                                                 |
| For private channel users, ensure your bot is listed as an administrator and your channel ID is numeric with -100 prefix; public channels use @channelname format but may lack some subscription verification features.                                                                                                                                                              | Sticky Notes 🔐 Bot Configuration                                                                                 |
| Lead magnet delivery message includes coupon link to SendPulse with instructions on how to redeem bonus. Adjust URLs and coupon codes as needed.                                                                                                                                                                                                                                  | Lead Magnet Message Node                                                                                           |
| Retry mechanism implemented via inline keyboard button "Try Again" to prompt users to resubscribe before rechecking status.                                                                                                                                                                                                                                                        | Subscription failure message node                                                                                  |
| Workflow uses Telegram API methods: getChatMember (check subscription), answerCallbackQuery (button ack), sendChatAction (typing indicator), and sendMessage (sending all messages).                                                                                                                                                                                                | Node technical details                                                                                            |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow, strictly adhering to content policies. No illegal, offensive, or protected content is included. All data handled is legal and publicly available.

---