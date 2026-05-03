Query and Monitor Shopify Orders via Telegram Bot Commands

https://n8nworkflows.xyz/workflows/query-and-monitor-shopify-orders-via-telegram-bot-commands-4714


# Query and Monitor Shopify Orders via Telegram Bot Commands

---

### 1. Workflow Overview

This n8n workflow enables users to query and monitor Shopify store orders interactively through Telegram bot commands. It listens for Telegram messages or interactions, interprets commands to either list all open orders or fetch details of a specific order, retrieves data from Shopify accordingly, and sends formatted responses back to the Telegram chat.

The workflow is logically grouped into these blocks:

- **1.1 Telegram Input Reception:** Trigger on Telegram messages or callback queries to start processing user commands.
- **1.2 Command Routing:** Determine if the user wants to list all open orders or get details of a specific order.
- **1.3 Shopify Data Retrieval:** Fetch all open orders or a single order's details from Shopify via API.
- **1.4 Data Formatting:** Prepare user-friendly messages listing orders or detailed order line items.
- **1.5 Telegram Response Dispatch:** Send or edit messages in Telegram to present retrieved order information.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Telegram Input Reception

- **Overview:**  
  Listens for incoming Telegram messages or callback queries (button presses) which trigger further processing.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - **Type & Role:** Telegram Trigger node; receives Telegram updates (messages and callback queries).  
    - **Configuration:**  
      - Listens for all message updates.  
      - Uses Telegram API credentials with bot access token.  
    - **Key Expressions:** None (direct trigger).  
    - **Inputs:** Incoming Telegram updates.  
    - **Outputs:** Raw message or callback query JSON to downstream nodes.  
    - **Failure Modes:** Possible webhook issues, invalid bot token, or Telegram API connectivity problems.  
    - **Sub-workflow:** None.

---

#### 2.2 Command Routing

- **Overview:**  
  Decides whether to fetch all open orders or get details for a specific order based on the Telegram input text or callback query data.

- **Nodes Involved:**  
  - Get All Orders/Get An Order Detail (Switch)

- **Node Details:**

  - **Get All Orders/Get An Order Detail**  
    - **Type & Role:** Switch node; routes workflow path based on user command.  
    - **Configuration:**  
      - Checks if the message text equals "/orders" → route to "Get Order Detail" output.  
      - Checks if callback query data starts with "/order_" → route to "Get An Order" output.  
      - Case-insensitive matching enabled.  
    - **Key Expressions:**  
      - `{{$json.message.text}}` for message text.  
      - `{{$json.callback_query.data}}` for callback data.  
    - **Inputs:** Telegram Trigger output.  
    - **Outputs:** Two outputs—one for listing all orders, one for individual order detail retrieval.  
    - **Failure Modes:** If input format is unexpected or missing fields, routing may fail or misroute.  
    - **Sub-workflow:** None.

---

#### 2.3 Shopify Data Retrieval

- **Overview:**  
  Fetches order data from Shopify according to the command—either all open orders or one specific order by ID.

- **Nodes Involved:**  
  - get Orders  
  - Get Order ID (Code)  
  - get Order

- **Node Details:**

  - **get Orders**  
    - **Type & Role:** Shopify node; fetches all open orders.  
    - **Configuration:**  
      - Operation: getAll  
      - Filter: orders with status "open"  
      - Return all matched orders without pagination limit.  
      - Auth via Shopify Access Token credential.  
    - **Inputs:** From Switch node "Get All Orders/Get An Order Detail" ("Get Order Detail" output).  
    - **Outputs:** List of open orders.  
    - **Failure Modes:** Auth errors, API rate limits, network timeout.  
    - **Sub-workflow:** None.

  - **Get Order ID**  
    - **Type & Role:** Code node; extracts order ID and chat context from callback query data.  
    - **Configuration:**  
      - Parses `callback_query.data` using regex `/order_(\d+)/` to get order ID.  
      - Extracts `chat_id` and `message_id` from callback query message.  
    - **Inputs:** Switch node output "Get An Order".  
    - **Outputs:** JSON with `orderId`, `chat_id`, and `message_id`.  
    - **Failure Modes:** If callback data format changes or regex fails, orderId extraction fails.  
    - **Sub-workflow:** None.

  - **get Order**  
    - **Type & Role:** Shopify node; fetches details for a single order by ID.  
    - **Configuration:**  
      - Operation: get  
      - Order ID taken from previous node's JSON property `orderId`.  
      - Auth via Shopify Access Token credential.  
    - **Inputs:** From "Get Order ID" node.  
    - **Outputs:** Order detail JSON.  
    - **Failure Modes:** Invalid order ID, Shopify API errors, auth failure.  
    - **Sub-workflow:** None.

---

#### 2.4 Data Formatting

- **Overview:**  
  Formats the fetched data into Telegram-friendly message structures for orders listing or order detail display.

- **Nodes Involved:**  
  - Check If There's Any Order (If)  
  - Orders (Code)  
  - No Order (Code)  
  - Clean Up Order (Code)

- **Node Details:**

  - **Check If There's Any Order**  
    - **Type & Role:** If node; checks if Shopify order list is not empty.  
    - **Configuration:**  
      - Condition: Checks if JSON data is not empty (non-empty object/array).  
    - **Inputs:** From "get Orders".  
    - **Outputs:** Two outputs—true (orders exist), false (no orders).  
    - **Failure Modes:** Data structure mismatch could cause false negatives.  
    - **Sub-workflow:** None.

  - **Orders**  
    - **Type & Role:** Code node; formats a list of open orders into Telegram inline keyboard buttons.  
    - **Configuration:**  
      - Retrieves chat ID from Telegram Trigger message.  
      - Builds message with text "All Open Orders".  
      - Creates `inline_keyboard` array with buttons for each order: text is order name and total price, callback data triggers `/order_<order_id>`.  
    - **Inputs:** True output of "Check If There's Any Order".  
    - **Outputs:** JSON formatted for Telegram sendMessage API.  
    - **Failure Modes:** If input orders data is malformed or missing properties, map operation may fail.  
    - **Sub-workflow:** None.

  - **No Order**  
    - **Type & Role:** Code node; sends a simple "No Order" text message if no orders are found.  
    - **Configuration:**  
      - Extracts chat ID from Telegram Trigger message.  
      - Prepares minimal text response.  
    - **Inputs:** False output of "Check If There's Any Order".  
    - **Outputs:** JSON formatted for Telegram sendMessage API.  
    - **Failure Modes:** Similar to above; if chat ID missing, message sending fails.  
    - **Sub-workflow:** None.

  - **Clean Up Order**  
    - **Type & Role:** Code node; formats detailed line items of a single order into markdown text for Telegram.  
    - **Configuration:**  
      - Reads `line_items` array from order JSON.  
      - Constructs a markdown message with order name and each item with title, variant title if any, quantity, and price.  
      - Includes Telegram chat and message IDs for editing message.  
      - Sets `parse_mode` to "Markdown".  
      - Empties `inline_keyboard` to remove buttons.  
    - **Inputs:** From "get Order".  
    - **Outputs:** JSON formatted for Telegram editMessageText API.  
    - **Failure Modes:** Missing line items or malformed data may cause errors in message construction.  
    - **Sub-workflow:** None.

---

#### 2.5 Telegram Response Dispatch

- **Overview:**  
  Sends the prepared messages back to Telegram either as new messages or edits existing messages.

- **Nodes Involved:**  
  - Send Orders to Telegram (HTTP Request)  
  - Send Order Details (Telegram node)

- **Node Details:**

  - **Send Orders to Telegram**  
    - **Type & Role:** HTTP Request node; sends new messages to Telegram using Telegram Bot API's `sendMessage` method.  
    - **Configuration:**  
      - URL uses bot token placeholder.  
      - Sends chat ID, text, and reply markup (inline keyboards) in body parameters.  
      - Called for listing all orders or no order message.  
    - **Inputs:** Outputs of "Orders" and "No Order" code nodes.  
    - **Outputs:** Telegram API response.  
    - **Failure Modes:** Invalid token, network failure, malformed message data.  
    - **Sub-workflow:** None.

  - **Send Order Details**  
    - **Type & Role:** Telegram node; edits existing Telegram message to show details of a single order.  
    - **Configuration:**  
      - Uses `editMessageText` operation.  
      - Inputs chat ID, message ID, text with markdown formatting.  
      - Uses Telegram API credentials.  
    - **Inputs:** Output of "Clean Up Order".  
    - **Outputs:** Telegram API response.  
    - **Failure Modes:** Message ID invalid or missing, Telegram API errors.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                         | Input Node(s)                  | Output Node(s)                    | Sticky Note                       |
|-----------------------------------|---------------------|---------------------------------------|-------------------------------|----------------------------------|---------------------------------|
| Telegram Trigger                  | Telegram Trigger    | Receives Telegram messages/callbacks | —                             | Get All Orders/Get An Order Detail |                                 |
| Get All Orders/Get An Order Detail| Switch              | Routes commands to fetching all orders or one order detail | Telegram Trigger              | get Orders, Get Order ID           |                                 |
| get Orders                       | Shopify             | Fetches all open orders from Shopify  | Get All Orders/Get An Order Detail ("Get Order Detail") | Check If There's Any Order         |                                 |
| Check If There's Any Order        | If                  | Checks if orders list is empty or not | get Orders                    | Orders (true), No Order (false)   |                                 |
| Orders                          | Code                | Formats list of open orders for Telegram | Check If There's Any Order (true) | Send Orders to Telegram          |                                 |
| No Order                         | Code                | Sends "No Order" text if no orders    | Check If There's Any Order (false) | Send Orders to Telegram          |                                 |
| Send Orders to Telegram           | HTTP Request        | Sends messages with orders list or no order | Orders, No Order              | —                                |                                 |
| Get Order ID                    | Code                | Extracts order ID & chat info from callback query | Get All Orders/Get An Order Detail ("Get An Order") | get Order                         |                                 |
| get Order                       | Shopify             | Fetches single order details by ID    | Get Order ID                  | Clean Up Order                   |                                 |
| Clean Up Order                  | Code                | Formats order details for Telegram edit message | get Order                    | Send Order Details              |                                 |
| Send Order Details               | Telegram            | Edits Telegram message to show order details | Clean Up Order               | —                                |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for updates → select "message" and "*" (all)  
   - Credentials: Configure Telegram API credential with your bot token  
   - Position: Left top area

2. **Add Switch Node (Get All Orders/Get An Order Detail)**  
   - Type: Switch  
   - Add two outputs with these conditions:  
     - Output 1 ("Get Order Detail"): If `{{$json.message.text}}` equals "/orders" (case-insensitive)  
     - Output 2 ("Get An Order"): If `{{$json.callback_query.data}}` starts with "/order_" (case-insensitive)  
   - Connect Telegram Trigger main output to this Switch node

3. **Add Shopify Node to Get All Orders (get Orders)**  
   - Type: Shopify  
   - Operation: getAll  
   - Options: Set status filter to "open"  
   - Authentication: Use Shopify Access Token credential  
   - Return all results  
   - Connect Switch node output "Get Order Detail" to this node

4. **Add If Node (Check If There's Any Order)**  
   - Type: If  
   - Condition: Check if input JSON data is not empty (operator: notEmpty on `$json`)  
   - Connect get Orders main output to this node

5. **Add Code Node (Orders)**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const chatId = $('Telegram Trigger').first().json.message.chat.id;
     return [{
       json: {
         chat_id: chatId,
         text: "All Open Orders",
         reply_markup: {
           inline_keyboard: items.map(order => [{
             text: `${order.json.name} ($${order.json.total_price})`,
             callback_data: `/order_${order.json.id}`
           }])
         }
       }
     }];
     ```  
   - Connect "Check If There's Any Order" true output to this node

6. **Add Code Node (No Order)**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const chatId = $('Telegram Trigger').first().json.message.chat.id;
     return [{
       json: {
         chat_id: chatId,
         text: "No Order"
       }
     }];
     ```  
   - Connect "Check If There's Any Order" false output to this node

7. **Add HTTP Request Node (Send Orders to Telegram)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.telegram.org/bot<Your Telegram Bot Token>/sendMessage` (replace `<Your Telegram Bot Token>`)  
   - Body parameters:  
     - chat_id = `={{ $json.chat_id }}`  
     - text = `={{ $json.text }}`  
     - reply_markup = `={{ JSON.stringify($json.reply_markup) }}`  
   - Connect "Orders" and "No Order" nodes main outputs to this node

8. **Add Code Node (Get Order ID)**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const callbackData = $json.callback_query.data;
     const id = callbackData.match(/^\/order_(\d+)/)?.[1];
     return [{
       json: {
         orderId: id,
         chat_id: $json.callback_query.message.chat.id,
         message_id: $json.callback_query.message.message_id
       }
     }];
     ```  
   - Connect Switch node output "Get An Order" to this node

9. **Add Shopify Node (get Order)**  
   - Type: Shopify  
   - Operation: get  
   - Order ID: `={{ $json.orderId }}`  
   - Auth: Shopify Access Token credential  
   - Connect "Get Order ID" node main output to this node

10. **Add Code Node (Clean Up Order)**  
    - Type: Code  
    - JavaScript code:  
      ```js
      const items = $json.line_items || [];
      let message = '🧾 *'+ $input.first().json.name+'*\n\n';
      items.forEach(item => {
        const variant = item.variant_title ? ` ${item.variant_title}` : '';
        message += `• ${item.title}${variant} x ${item.quantity} — $${item.price}\n`;
      });
      return [{
        json: {
          text: message,
          chat_id: $('Get Order ID').first().json.chat_id,
          message_id: $('Get Order ID').first().json.message_id,
          parse_mode: "Markdown",
          reply_markup: {
            inline_keyboard: []
          }
        }
      }];
      ```  
    - Connect "get Order" node main output to this node

11. **Add Telegram Node (Send Order Details)**  
    - Type: Telegram  
    - Operation: editMessageText  
    - Parameters:  
      - Text: `={{ $json.text }}`  
      - Chat ID: `={{ $json.chat_id }}`  
      - Message ID: `={{ $json.message_id }}`  
      - Parse Mode: Markdown  
    - Credentials: Telegram API with bot token  
    - Connect "Clean Up Order" main output to this node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Replace `<Your Telegram Bot Token>` in HTTP Request URL with your actual Telegram bot token to enable API calls.                                                       | Telegram Bot API documentation: https://core.telegram.org/bots/api#sendmessage                      |
| Shopify Access Token credentials must have permissions to read orders. Use a private app or custom app with appropriate scopes in Shopify Admin.                      | Shopify API docs: https://shopify.dev/api                                                          |
| Telegram Trigger webhook must be properly configured and reachable by Telegram servers for updates to arrive in n8n.                                                  | n8n docs on Telegram Trigger node: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.telegramTrigger/ |
| Inline keyboards in Telegram allow users to interact with bot messages via buttons. This workflow uses them for order selection.                                      | Telegram Inline Keyboard: https://core.telegram.org/bots/api#inlinekeyboard                         |
| Markdown formatting is used in order detail messages to improve readability. Telegram supports a subset of Markdown in messages.                                      | Telegram Markdown: https://core.telegram.org/bots/api#markdown-style                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.